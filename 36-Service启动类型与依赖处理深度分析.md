# Service 启动类型(Type=)与依赖处理深度分析

## 1. 概述

systemd 的 `Type=` 指令决定了服务何时被认为"已启动就绪"(ready)，这直接影响依赖它的后续单元何时开始启动。

| 类型 | 就绪信号 | 典型用途 |
|------|----------|----------|
| `simple` | fork() 后立即就绪 | 现代守护进程（不 fork） |
| `exec` | exec() 成功后就绪 | 需确认程序确实启动 |
| `forking` | 启动进程退出后就绪 | 传统 fork 两次的守护进程 |
| `oneshot` | 主进程退出(成功)后就绪 | 一次性脚本/命令 |
| `dbus` | 在 D-Bus 上获取指定 bus name 后就绪 | D-Bus 服务 |
| `notify` | 进程发送 READY=1 后就绪 | sd_notify() 协议服务 |
| `idle` | 同 simple，但延迟 exec 到所有 job 派发完成 | 控制台输出类服务 |

---

## 2. 源码定义

```c
// service.h:27-32
typedef enum ServiceType {
    SERVICE_SIMPLE,   // fork and go on right-away
    SERVICE_FORKING,  // forks by itself (traditional daemons)
    SERVICE_ONESHOT,  // fork and wait until program finishes
    SERVICE_DBUS,     // fork and wait for D-Bus name on bus
    SERVICE_NOTIFY,   // fork and wait for sd_notify() ready
    SERVICE_IDLE,     // like simple, but delay exec() until all jobs dispatched
    SERVICE_EXEC,     // fork and wait until exec() succeeds
} ServiceType;
```

---

## 3. 状态机与就绪判定

### 3.1 通用状态流转

```
DEAD → CONDITION → START_PRE → START → START_POST → RUNNING
                                                         ↓
                                              STOP → STOP_SIGTERM → STOP_SIGKILL → STOP_POST → DEAD/FAILED
```

### 3.2 就绪点与 UnitActiveState 映射

普通类型的状态映射（`state_translation_table`，`service.c:48-68`）：

| ServiceState | UnitActiveState | 含义 |
|--------------|-----------------|------|
| `SERVICE_DEAD` | INACTIVE | 未运行 |
| `SERVICE_CONDITION` | **ACTIVATING** | 正在检查条件 |
| `SERVICE_START_PRE` | **ACTIVATING** | 正在执行 ExecStartPre |
| `SERVICE_START` | **ACTIVATING** | 正在启动（等待就绪信号） |
| `SERVICE_START_POST` | **ACTIVATING** | 正在执行 ExecStartPost |
| `SERVICE_RUNNING` | **ACTIVE** | 已就绪运行中 |
| `SERVICE_EXITED` | **ACTIVE** | 进程已退出但视为活跃 |

**关键**：只有当 UnitActiveState 变为 `ACTIVE` 时，依赖该服务的后续单元（通过 `After=`）才会开始启动。

---

## 4. 各类型详细实现

### 4.1 Type=simple

**源码**：`service.c:2240-2245`

```c
if (IN_SET(s->type, SERVICE_SIMPLE, SERVICE_IDLE)) {
    /* For simple services we immediately start the START_POST binaries. */
    service_set_main_pid(s, pid);
    service_enter_start_post(s);    // ← fork() 后立即进入 START_POST
}
```

**流程**：
```
fork() → 记录 main_pid → 立即进入 START_POST → 进入 RUNNING(ACTIVE)
```

**特点**：
- spawn 子进程后**不等待任何信号**
- 如果子进程 exec() 失败（如二进制文件不存在），systemd 不会知道
- 最快的就绪方式，依赖单元几乎立即开始
- 不设置 TimeoutStartSec（使用 USEC_INFINITY）

**适用场景**：程序启动极快、不需要初始化确认的简单守护进程

---

### 4.2 Type=exec

**源码**：`service.c:1408-1431`（pipe 分配）、`service.c:3283-3334`（就绪检测）

**机制**：通过 `O_CLOEXEC` 管道检测 exec() 成功

```c
// 分配 exec_fd pipe
if (s->type == SERVICE_EXEC) {
    service_allocate_exec_fd(s, &exec_fd_source, &exec_params.exec_fd);
    // pipe2(p, O_CLOEXEC|O_NONBLOCK)
    // p[0] = parent 读端 (监控 EOF)
    // p[1] = child 写端 (传给子进程作为 exec_fd)
}
```

**子进程协议**：
1. 子进程准备 exec() 前写入非零字节 → 设置 `exec_fd_hot = true`
2. exec() 成功 → 内核因 `O_CLOEXEC` 自动关闭管道 → parent 收到 EOF
3. exec() 失败 → 子进程写入零字节 → `exec_fd_hot = false`

```c
// service_dispatch_exec_io() - service.c:3283
if (n == 0) {  // EOF
    if (s->exec_fd_hot) {  // 子进程已表明即将 exec
        // exec() 成功！
        if (s->type == SERVICE_EXEC && s->state == SERVICE_START)
            service_enter_start_post(s);  // → RUNNING(ACTIVE)
    }
}
// 读到字节时: s->exec_fd_hot = x;
```

**流程**：
```
fork() → 子进程写入 1 → exec() → O_CLOEXEC 关闭管道 → parent 收到 EOF → START_POST → RUNNING
```

**对比 simple**：
| | simple | exec |
|--|--------|------|
| 就绪点 | fork() 后 | exec() 成功后 |
| 检测失败 | 不能 | 能（EOF 不来或收到 0） |
| 适用 TimeoutStartSec | 否 | 是 |

---

### 4.3 Type=forking

**源码**：`service.c:2194-2253`

```c
if (s->type == SERVICE_FORKING) {
    s->control_command_id = SERVICE_EXEC_START;
    c = s->control_command = s->exec_command[SERVICE_EXEC_START];
    s->main_command = NULL;  // ← ExecStart 作为 control process

    // spawn 后:
    s->control_pid = pid;            // 记录为 control_pid（非 main_pid）
    service_set_state(s, SERVICE_START);  // 保持 ACTIVATING
}
```

**就绪条件**：control_pid（启动进程）退出成功后

```c
// service_sigchld_event() 中，当 control_pid 退出:
case SERVICE_START:
    if (s->type == SERVICE_FORKING) {
        // 启动进程退出了
        service_load_pid_file(s, ...);  // 从 PIDFile= 读取真正的 main PID
        service_enter_start_post(s);    // → RUNNING(ACTIVE)
    }
```

**流程**：
```
fork() ExecStart → ExecStart fork() 子守护进程 → ExecStart 退出
→ systemd 检测 control_pid 退出 → 读取 PIDFile → START_POST → RUNNING
```

**关键配置**：
- `PIDFile=`：告知 systemd 真正守护进程的 PID
- `GuessMainPID=`：没有 PIDFile 时猜测主进程

---

### 4.4 Type=oneshot

**源码**：`service.c:3540-3581`

```c
// service_sigchld_event() 中，主进程退出时:
case SERVICE_START:
    if (s->type == SERVICE_ONESHOT) {
        if (f == SERVICE_SUCCESS)
            service_enter_start_post(s);  // 成功退出 → 就绪
        else
            service_enter_signal(s, SERVICE_STOP_SIGTERM, f);  // 失败 → 停止
    }
```

**特殊**：支持多命令串行执行：

```c
if (s->main_command &&
    s->main_command->command_next &&
    s->type == SERVICE_ONESHOT &&
    f == SERVICE_SUCCESS) {
    /* There is another command to execute */
    service_run_next_main(s);  // 继续执行下一条 ExecStart=
}
```

**流程**：
```
fork() ExecStart[0] → 成功退出 → ExecStart[1] → ... → 最后一条退出
→ START_POST → RUNNING(实际为 EXITED 状态)
```

**特点**：
- 允许多条 `ExecStart=`，顺序执行
- 无 ExecStart 时也可工作（仅靠 ExecStartPre/Post）
- 退出码判定使用 `EXIT_CLEAN_COMMAND`（只检查退出码为 0）
- 默认 `RemainAfterExit=yes`（进程退出后保持 active 状态）

---

### 4.5 Type=dbus

**源码**：`service.c:4352-4400`

**就绪条件**：配置的 `BusName=` 出现在 D-Bus 总线上

```c
// service_bus_name_owner_change() - service.c:4352
static void service_bus_name_owner_change(Unit *u, const char *new_owner) {
    s->bus_name_good = new_owner;  // 有 owner → true

    if (s->type == SERVICE_DBUS) {
        if (s->state == SERVICE_START && new_owner)
            service_enter_start_post(s);  // Bus name 出现 → 就绪！
        else if (s->state == SERVICE_RUNNING)
            service_enter_running(s, SERVICE_SUCCESS);  // 运行时 bus name 消失 → 重新判断
    }
}
```

**service_good() 检查**：

```c
// service.c:2060-2078
static bool service_good(Service *s) {
    if (s->type == SERVICE_DBUS && !s->bus_name_good)
        return false;  // D-Bus 名消失 → 服务不再 good
    // ... 检查主进程
}
```

**流程**：
```
fork() → 设置 main_pid → 保持 SERVICE_START(ACTIVATING)
→ 进程获取 D-Bus name → Manager 监听到 NameOwnerChanged
→ service_bus_name_owner_change(new_owner=xxx) → START_POST → RUNNING
```

**特点**：
- Manager 自动监控 `BusName=` 的所有权变化
- Bus name 消失时服务状态变为 not-good → 触发停止
- 同时设置 `After=dbus.socket`（隐含依赖）

---

### 4.6 Type=notify

**源码**：`service.c:4086-4179`

**就绪条件**：主进程通过 `sd_notify("READY=1")` 通知 systemd

```c
// service_notify_message() - service.c:4145-4151
if (streq(*i, "READY=1")) {
    s->notify_state = NOTIFY_READY;

    /* Type=notify services inform us about completed initialization */
    if (s->type == SERVICE_NOTIFY && s->state == SERVICE_START)
        service_enter_start_post(s);  // 收到 READY=1 → 就绪！
}
```

**通知协议支持的消息**：

| 消息 | 作用 |
|------|------|
| `READY=1` | 服务就绪 |
| `RELOADING=1` | 正在重载配置 |
| `STOPPING=1` | 正在停止 |
| `STATUS=text` | 状态文本（显示在 systemctl status 中） |
| `MAINPID=pid` | 报告真正的主进程 PID |
| `WATCHDOG=1` | 看门狗心跳 |
| `ERRNO=n` | 报告错误码 |

**流程**：
```
fork() → 设置 main_pid → 保持 SERVICE_START(ACTIVATING)
→ 进程完成初始化 → sd_notify("READY=1")
→ service_notify_message() 处理 → START_POST → RUNNING
```

**错误处理**：
```c
// service_sigchld_event() - 如果主进程在发送 READY=1 前退出:
case SERVICE_START:
    if (s->type == SERVICE_NOTIFY) {
        if (f != SERVICE_SUCCESS)
            service_enter_signal(s, SERVICE_STOP_SIGTERM, f);  // 异常退出
        else if (!s->remain_after_exit || s->notify_access == NOTIFY_MAIN)
            service_enter_signal(s, SERVICE_STOP_SIGTERM, SERVICE_FAILURE_PROTOCOL);
            // ↑ 未发 READY=1 就退出 = 协议违规
    }
```

**NotifyAccess= 权限控制**：
- `none`：禁止通知
- `main`：仅主进程可通知
- `exec`：ExecStart 进程可通知
- `all`：cgroup 内所有进程可通知

---

### 4.7 Type=idle

**源码**：`service.c:1665-1666`、`state_translation_table_idle`

```c
// 传递 idle_pipe 给子进程
if (s->type == SERVICE_IDLE)
    exec_params.idle_pipe = UNIT(s)->manager->idle_pipe;
```

**机制**：
1. Manager 维护一个 `idle_pipe`
2. idle 类型服务的子进程在 exec() 前会等待 idle_pipe 可读
3. Manager 在所有 job 派发完成后关闭 idle_pipe 写端
4. 这触发所有等待的 idle 服务同时开始 exec()

**状态映射的特殊性**：

```c
// state_translation_table_idle - service.c:72-92
// SERVICE_START → UNIT_ACTIVE (不是 ACTIVATING!)
// SERVICE_START_PRE → UNIT_ACTIVE
// SERVICE_CONDITION → UNIT_ACTIVE
```

idle 类型从 CONDITION 阶段开始就报告为 `ACTIVE`！这意味着**它不会阻塞依赖链**。

**流程**：
```
fork() → 子进程等待 idle_pipe → 设置 main_pid → 报告为 ACTIVE
→ 所有 job 派发完成 → idle_pipe 关闭 → 子进程 exec()
```

**适用场景**：`getty@.service` 等需要在控制台输出的服务，避免启动日志与登录提示混杂。

---

## 5. 依赖处理与启动顺序

### 5.1 就绪时机对依赖的影响

假设：`B.service` 配置 `After=A.service Requires=A.service`

| A 的类型 | B 何时开始启动 |
|----------|---------------|
| simple | A fork() 后立即（A 可能还没执行到 main()） |
| exec | A exec() 成功后（确认二进制已加载） |
| forking | A 的启动进程退出后（传统就绪方式） |
| oneshot | A 的所有 ExecStart 命令成功退出后 |
| dbus | A 在 D-Bus 上获取 BusName 后 |
| notify | A 发送 READY=1 后（应用层就绪） |
| idle | A fork() 后立即（idle 不阻塞依赖链） |

### 5.2 超时机制

```c
// service_enter_start() - service.c:2225-2230
if (IN_SET(s->type, SERVICE_SIMPLE, SERVICE_IDLE))
    timeout = USEC_INFINITY;  // simple/idle 不设启动超时
else
    timeout = s->timeout_start_usec;  // 其他类型使用 TimeoutStartSec=
```

| 类型 | 默认 TimeoutStartSec |
|------|---------------------|
| simple/idle | 无限（不适用） |
| exec/forking/oneshot/dbus/notify | 90s（DefaultTimeoutStartSec=） |

### 5.3 Job 完成判定

```
Job(JOB_START) for unit u:
  → unit_start(u)
  → 等待 UnitActiveState 变为 ACTIVE 或 FAILED
  → ACTIVE → job_finish(JOB_DONE) → 触发依赖 u 的单元开始
  → FAILED/超时 → job_finish(JOB_FAILED) → 级联失败
```

---

## 6. 选型决策树

```
需要等待服务完全初始化完成？
├── 是 → 服务支持 sd_notify()？
│   ├── 是 → Type=notify（最精确的就绪判定）
│   └── 否 → 服务提供 D-Bus 接口？
│       ├── 是 → Type=dbus
│       └── 否 → 传统 fork-and-exit 守护进程？
│           ├── 是 → Type=forking
│           └── 否 → 一次性命令（执行完即完成）？
│               ├── 是 → Type=oneshot
│               └── 否 → 至少确认 exec 成功？
│                   ├── 是 → Type=exec
│                   └── 否 → Type=simple
└── 否 → 延迟到所有 job 派发完？
    ├── 是 → Type=idle
    └── 否 → Type=simple
```

---

## 7. 实际配置示例

```ini
# Type=simple (默认) - 现代守护进程
[Service]
Type=simple
ExecStart=/usr/bin/mydaemon --foreground

# Type=exec - 确认启动
[Service]
Type=exec
ExecStart=/usr/bin/complex-app
TimeoutStartSec=30

# Type=forking - 传统守护进程
[Service]
Type=forking
PIDFile=/run/legacy.pid
ExecStart=/usr/sbin/legacy-daemon

# Type=oneshot - 系统初始化脚本
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/setup-network.sh
ExecStart=/usr/bin/setup-firewall.sh

# Type=dbus - D-Bus 激活服务
[Service]
Type=dbus
BusName=org.freedesktop.NetworkManager
ExecStart=/usr/sbin/NetworkManager --no-daemon

# Type=notify - 精确就绪通知
[Service]
Type=notify
ExecStart=/usr/lib/systemd/systemd-resolved
WatchdogSec=3min

# Type=idle - 登录提示
[Service]
Type=idle
ExecStart=/sbin/agetty --noclear %I $TERM
```

---

## 8. 关键源码索引

| 功能 | 文件:行 |
|------|---------|
| ServiceType 枚举定义 | `service.h:27-33` |
| state_translation_table | `service.c:48-68` |
| state_translation_table_idle | `service.c:72-92` |
| service_enter_start() 分发逻辑 | `service.c:2179-2266` |
| simple/idle 立即就绪 | `service.c:2240-2245` |
| exec 管道分配 | `service.c:1408-1431` |
| exec EOF 检测 | `service.c:3283-3334` |
| forking control_pid 记录 | `service.c:2247-2253` |
| oneshot 多命令串行 | `service.c:3540-3548` |
| notify READY=1 处理 | `service.c:4145-4151` |
| dbus bus_name_owner_change | `service.c:4352-4400` |
| idle_pipe 传递 | `service.c:1665-1666` |
| service_good() 判定 | `service.c:2060-2078` |
| sigchld 事件处理 | `service.c:3467-3600` |

---

## 9. 设计总结

| 设计原则 | 体现 |
|----------|------|
| **零假设** | simple 不假设进程行为；forking 不假设 PID 归属 |
| **渐进精确** | simple < exec < notify 就绪精度递增 |
| **向后兼容** | forking 支持传统 SysV 守护进程 |
| **协议清晰** | notify 定义完整的 sd_notify 协议（READY/STOPPING/RELOADING） |
| **不阻塞** | idle 通过特殊状态映射确保不阻塞依赖链 |
| **失败检测** | exec 用 O_CLOEXEC 管道；notify 检测未发 READY 就退出 |
