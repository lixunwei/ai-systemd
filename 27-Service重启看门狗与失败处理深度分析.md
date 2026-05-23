# 27. Service 重启、看门狗与失败处理深度分析

## 1. 概述

systemd 的 Service 单元拥有复杂的**失败检测与自动恢复**机制，包括：
- 多模式自动重启（Restart=）
- 硬件级看门狗（WatchdogSec=）
- 启动频率限制（StartLimitBurst=）
- 超时保护（TimeoutStartSec=/TimeoutStopSec=）
- 失败后系统级动作（FailureAction=）

### 核心源文件

| 文件 | 关键行 | 职责 |
|------|--------|------|
| `src/core/service.h` | 68-82, 109-111 | ServiceResult 枚举、状态字段 |
| `src/core/service.c` | 1745-1864, 2345-2388, 3824-4040, 4231-4239 | 重启决策、看门狗、超时 |
| `src/core/unit.c` | 1792-1812 | 启动频率限制 |
| `src/core/emergency-action.c` | 15-25 | FailureAction 执行 |
| `src/basic/ratelimit.h` | 9-14 | RateLimit 结构体 |

---

## 2. ServiceResult 枚举（service.h:68-82）

所有可能的服务退出结果：

```c
typedef enum ServiceResult {
    SERVICE_SUCCESS,                    // 正常退出（exit 0 或配置的成功码）
    SERVICE_FAILURE_RESOURCES,          // 资源不足（fork 失败等）
    SERVICE_FAILURE_PROTOCOL,           // 协议错误（Type=notify 但未 READY）
    SERVICE_FAILURE_TIMEOUT,            // 启动/停止超时
    SERVICE_FAILURE_EXIT_CODE,          // 非零退出码
    SERVICE_FAILURE_SIGNAL,             // 被信号杀死
    SERVICE_FAILURE_CORE_DUMP,          // 产生 core dump
    SERVICE_FAILURE_WATCHDOG,           // 看门狗超时
    SERVICE_FAILURE_START_LIMIT_HIT,    // 启动频率限制
    SERVICE_FAILURE_OOM_KILL,           // 被 OOM killer 杀死
    SERVICE_SKIP_CONDITION,             // 条件不满足（跳过）
    _SERVICE_RESULT_MAX,
} ServiceResult;
```

---

## 3. Restart= 重启模式

### 3.1 模式定义

| 模式 | 触发条件 |
|------|----------|
| `no` | 永不重启 |
| `always` | 任何退出都重启（除 condition-skip） |
| `on-success` | 仅正常退出时重启 |
| `on-failure` | 异常退出时重启 |
| `on-abnormal` | 信号/core dump/超时/看门狗/OOM 时重启 |
| `on-watchdog` | 仅看门狗超时时重启 |
| `on-abort` | 仅被信号杀死或 core dump 时重启 |

### 3.2 决策矩阵

| ServiceResult | always | on-success | on-failure | on-abnormal | on-watchdog | on-abort |
|---------------|--------|------------|------------|-------------|-------------|----------|
| SUCCESS | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ |
| EXIT_CODE | ✓ | ✗ | ✓ | ✗ | ✗ | ✗ |
| SIGNAL | ✓ | ✗ | ✓ | ✓ | ✗ | ✓ |
| CORE_DUMP | ✓ | ✗ | ✓ | ✓ | ✗ | ✓ |
| WATCHDOG | ✓ | ✗ | ✓ | ✓ | ✓ | ✗ |
| TIMEOUT | ✓ | ✗ | ✓ | ✓ | ✗ | ✗ |
| OOM_KILL | ✓ | ✗ | ✓ | ✓ | ✗ | ✗ |
| RESOURCES | ✓ | ✗ | ✓ | ✗ | ✗ | ✗ |
| SKIP_CONDITION | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |

### 3.3 service_shall_restart()（service.c:1745-1793）

```c
static bool service_shall_restart(Service *s):
  // 第一优先级: RestartPreventExitStatus= 阻止重启
  if (exit_status_in(s->restart_prevent_status, main_exit_code, main_exit_status)):
    return false;

  // 第二优先级: RestartForceExitStatus= 强制重启
  if (exit_status_in(s->restart_force_status, main_exit_code, main_exit_status)):
    return true;

  // 第三优先级: 按 Restart= 模式判断
  switch (s->restart):
    case SERVICE_RESTART_ALWAYS:
      return s->result != SERVICE_SKIP_CONDITION;
    case SERVICE_RESTART_ON_SUCCESS:
      return s->result == SERVICE_SUCCESS;
    case SERVICE_RESTART_ON_FAILURE:
      return !IN_SET(s->result, SERVICE_SUCCESS, SERVICE_SKIP_CONDITION);
    case SERVICE_RESTART_ON_ABNORMAL:
      return !IN_SET(s->result, SERVICE_SUCCESS, SERVICE_FAILURE_EXIT_CODE,
                     SERVICE_SKIP_CONDITION);
    case SERVICE_RESTART_ON_WATCHDOG:
      return s->result == SERVICE_FAILURE_WATCHDOG;
    case SERVICE_RESTART_ON_ABORT:
      return IN_SET(s->result, SERVICE_FAILURE_SIGNAL, SERVICE_FAILURE_CORE_DUMP);
    default:
      return false;
```

---

## 4. 退出状态映射

### 4.1 SuccessExitStatus=

```ini
# 将额外的退出码视为成功
SuccessExitStatus=0 1 SIGHUP SIGTERM
```

```c
// service_sigchld_event() 中:
if (exit_status_in(s->success_status, code, status))
  result = SERVICE_SUCCESS;  // 视为正常退出
```

### 4.2 RestartForceExitStatus=

```ini
# 这些退出码强制触发重启（无论 Restart= 设置）
RestartForceExitStatus=42 SIGUSR2
```

### 4.3 RestartPreventExitStatus=

```ini
# 这些退出码阻止重启（即使 Restart=always）
RestartPreventExitStatus=1 6 SIGABRT
```

### 4.4 优先级顺序

```
RestartPreventExitStatus > RestartForceExitStatus > Restart= 模式
```

---

## 5. 重启流程

### 5.1 service_enter_restart()（service.c:2345-2388）

```c
service_enter_restart(Service *s):
  1. 检查是否有 stop job 排队（有则不重启）
  2. 向 Manager 请求 JOB_RESTART job
  3. 递增 s->n_restarts 计数
  4. 记录日志: "Scheduled restart job, restart counter is at %u"
  5. 设置状态为 SERVICE_AUTO_RESTART
  6. 若失败 → SERVICE_FAILURE_RESOURCES
```

### 5.2 完整重启时序

```
主进程退出
    │
    ▼
service_sigchld_event()
    ├─ 判定 ServiceResult
    ├─ service_enter_dead(result)
    │       │
    │       ▼
    │   service_shall_restart()?
    │       │
    │   YES │ NO → 进入 DEAD/FAILED
    │       ▼
    │   unit_test_start_limit()
    │       │
    │   PASS│ FAIL → SERVICE_FAILURE_START_LIMIT_HIT
    │       ▼         + emergency_action()
    │   设置定时器: RestartSec= 延迟
    │       │
    │   [等待 RestartSec]
    │       │
    │       ▼
    └── service_enter_restart()
            ├─ JOB_RESTART → stop 当前 → start 新实例
            └─ SERVICE_AUTO_RESTART 状态
```

---

## 6. RestartSec= 延迟

```c
// service_enter_dead() 中:
if (service_shall_restart(s)):
  // 设置定时器
  r = service_arm_timer(s, s->restart_usec);  // 默认 100ms
  // 定时器到期后调用 service_enter_restart()
```

- 默认值：100ms（`DEFAULT_RESTART_USEC`）
- 用途：防止 CPU 密集循环重启
- 与启动限制配合使用

---

## 7. 启动频率限制

### 7.1 RateLimit 结构体（ratelimit.h:9-14）

```c
typedef struct RateLimit {
    usec_t interval;    // 时间窗口
    unsigned burst;     // 窗口内允许的最大次数
    unsigned num;       // 当前计数
    usec_t begin;       // 窗口起始时间
} RateLimit;
```

### 7.2 配置项

| 指令 | 默认值 | 说明 |
|------|--------|------|
| `StartLimitIntervalSec=` | 10s | 计数窗口 |
| `StartLimitBurst=` | 5 | 窗口内最大启动次数 |
| `StartLimitAction=` | none | 超限后的动作 |

### 7.3 检查逻辑（unit.c:1792-1812）

```c
unit_test_start_limit(Unit *u):
  if (!ratelimit_below(&u->start_ratelimit)):
    // 超过频率限制
    log_unit_warning(u, "Start request repeated too quickly.");
    u->start_limit_hit = true;

    // 执行 StartLimitAction
    emergency_action(u->manager, u->start_limit_action, ...
                     "unit failed");

    return -ECANCELED;
  return 0;
```

### 7.4 StartLimitAction= 可选值

| 值 | 行为 |
|-----|------|
| `none` | 仅停止重启，服务进入 failed |
| `reboot` | 正常重启系统 |
| `reboot-force` | 强制重启（跳过 shutdown） |
| `reboot-immediate` | 立即重启（直接 reboot(2)） |
| `poweroff` | 正常关机 |
| `poweroff-force` | 强制关机 |
| `poweroff-immediate` | 立即关机 |
| `exit` | 退出用户 Manager |
| `exit-force` | 强制退出 |

---

## 8. WatchdogSec= 看门狗

### 8.1 工作原理

```
┌──────────────┐                    ┌─────────────┐
│   服务进程    │  sd_notify         │  PID 1      │
│              │  "WATCHDOG=1"      │             │
│  主循环:     │ ───────────────→  │ 重置定时器   │
│   work()     │  每 WatchdogSec/2  │             │
│   notify()   │                    │  若超时:     │
│   work()     │                    │  → SIGABRT  │
│   notify()   │                    │  → 进入 stop│
└──────────────┘                    └─────────────┘
```

### 8.2 定时器管理（service.c:199-298）

```c
service_start_watchdog(Service *s):
  watchdog_usec = s->watchdog_usec;  // WatchdogSec= 值
  
  if (watchdog_usec == USEC_INFINITY):
    service_stop_watchdog(s);  // 禁用
    return;
  
  // 创建或更新定时器
  // 到期时间 = 当前时间 + watchdog_usec
  sd_event_add_time(... CLOCK_MONOTONIC ...
                    watchdog_usec, 0,
                    service_dispatch_watchdog, s);

service_reset_watchdog(Service *s):
  // sd_notify("WATCHDOG=1") 调用此函数
  s->watchdog_timestamp = now(CLOCK_MONOTONIC);
  service_start_watchdog(s);  // 重新 arm 定时器
```

### 8.3 超时处理（service.c:4021-4040）

```c
service_dispatch_watchdog(sd_event_source *source, usec_t usec, void *userdata):
  Service *s = userdata;

  log_unit_error(UNIT(s), "Watchdog timeout (limit %s)!",
                 FORMAT_TIMESPAN(s->watchdog_usec));

  // 发送 SIGABRT（产生 core dump 用于调试）
  service_enter_signal(s, SERVICE_STOP_WATCHDOG, SERVICE_FAILURE_WATCHDOG);
```

### 8.4 WATCHDOG=trigger

```c
// service_notify_message() 中:
if (streq(e, "WATCHDOG=trigger")):
  // 服务主动请求看门狗故障（用于自检失败）
  service_force_watchdog(s);  // 立即触发超时
```

### 8.5 TimeoutAbortSec=

看门狗超时后进入 `SERVICE_STOP_WATCHDOG` 状态，等待 `TimeoutAbortSec=` 秒（默认等于 WatchdogSec）让进程处理 SIGABRT 产生 core dump。超过此时间则发送 SIGKILL。

---

## 9. 超时机制

### 9.1 三阶段超时

| 指令 | 默认值 | 应用阶段 |
|------|--------|----------|
| `TimeoutStartSec=` | 90s | ExecStartPre/ExecStart/ExecStartPost |
| `TimeoutStopSec=` | 90s | ExecStop/SIGTERM 等待 |
| `TimeoutAbortSec=` | = TimeoutStopSec | SIGABRT 后等待 core dump |

### 9.2 超时处理流程（service.c:3824-4019）

```
service_dispatch_timer() — 统一超时派发:

启动阶段超时 (SERVICE_CONDITION, START_PRE, START, START_POST):
  1. 发送 SIGTERM (或 KillSignal=)
  2. 设置 TimeoutStopSec 等待
  3. 超时后发送 SIGKILL
  4. 进入 SERVICE_STOP_SIGTERM → SERVICE_STOP_SIGKILL → DEAD

停止阶段超时 (SERVICE_STOP, STOP_SIGTERM):
  1. 升级到 SIGKILL
  2. 进入 SERVICE_STOP_SIGKILL

看门狗阶段 (SERVICE_STOP_WATCHDOG):
  1. TimeoutAbortSec 到期
  2. 升级到 SIGKILL 或直接 SIGTERM

最终阶段 (SERVICE_FINAL_SIGTERM, FINAL_SIGKILL):
  1. 清理残留进程
  2. 进入 DEAD
```

### 9.3 信号升级链

```
正常停止:  SIGTERM → [TimeoutStopSec] → SIGKILL → [5s] → 放弃
看门狗:    SIGABRT → [TimeoutAbortSec] → SIGKILL → [5s] → 放弃
超时启动:  SIGTERM → [TimeoutStopSec] → SIGKILL → [5s] → 放弃
```

---

## 10. OOMPolicy=

### 10.1 三种策略

| 策略 | 行为 |
|------|------|
| `continue` | OOM kill 后不做额外处理，依赖 Restart= |
| `stop` | OOM kill 后主动停止服务（发送 SIGTERM） |
| `kill` | OOM kill 后杀死整个 cgroup（SIGKILL） |

### 10.2 实现（service.c:3415-3464）

```c
// 当收到 cgroup OOM 通知时:
unit_notify_cgroup_oom(Unit *u):
  switch (oom_policy):
    case OOM_CONTINUE:
      // 记录日志，不做处理
      break;
    case OOM_STOP:
      service_enter_signal(s, SERVICE_STOP_SIGTERM, SERVICE_FAILURE_OOM_KILL);
      break;
    case OOM_KILL:
      service_enter_signal(s, SERVICE_STOP_SIGKILL, SERVICE_FAILURE_OOM_KILL);
      break;
```

### 10.3 memory.oom.group 集成

```c
// 当 OOMPolicy=kill 时:
// 写入 cgroup memory.oom.group = 1
// 这使内核 OOM killer 杀死整个 cgroup 而非单个进程
```

---

## 11. FailureAction= / SuccessAction=

### 11.1 配置

```ini
[Service]
FailureAction=reboot       # 服务失败时重启系统
SuccessAction=poweroff     # 服务成功退出时关机
```

### 11.2 触发时机

```c
// unit.c 中:
// 服务进入 FAILED 状态后:
if (u->failure_action != EMERGENCY_ACTION_NONE)
  emergency_action(m, u->failure_action, u->failure_action_exit_status,
                   u->reboot_arg, -1, "unit failed");

// 服务成功退出后:
if (u->success_action != EMERGENCY_ACTION_NONE)
  emergency_action(m, u->success_action, u->success_action_exit_status,
                   u->reboot_arg, -1, "unit succeeded");
```

---

## 12. 完整失败处理流程图

```
┌──────────────────────────────────────────────────────────┐
│                 服务异常退出                               │
│                     │                                    │
│                     ▼                                    │
│           判定 ServiceResult                             │
│           (exit_code/signal/timeout/watchdog/oom)         │
│                     │                                    │
│                     ▼                                    │
│      ┌─── service_shall_restart()? ───┐                 │
│      │                                │                 │
│     YES                              NO                  │
│      │                                │                 │
│      ▼                                ▼                 │
│  unit_test_start_limit()        进入 FAILED 状态         │
│      │                                │                 │
│  ┌───┴───┐                           ▼                 │
│  │PASS   │FAIL              FailureAction= 执行?        │
│  │       │                  (reboot/poweroff/none)       │
│  ▼       ▼                                              │
│ arm     SERVICE_FAILURE_                                 │
│ timer   START_LIMIT_HIT                                  │
│ (RestartSec)  │                                         │
│  │            ▼                                         │
│  │    StartLimitAction= 执行                             │
│  │    (reboot/poweroff/exit/none)                        │
│  │                                                      │
│  ▼                                                      │
│ service_enter_restart()                                  │
│  → JOB_RESTART                                          │
│  → n_restarts++                                         │
│  → 新实例启动                                            │
└──────────────────────────────────────────────────────────┘
```

---

## 13. 配置示例

### 13.1 高可用服务

```ini
[Service]
Type=notify
Restart=on-failure
RestartSec=5s
WatchdogSec=30s

# 10秒内最多重启3次
StartLimitIntervalSec=10s
StartLimitBurst=3
StartLimitAction=reboot-force

# 退出码 42 强制重启
RestartForceExitStatus=42

# 超时配置
TimeoutStartSec=60s
TimeoutStopSec=30s
```

### 13.2 关键基础设施

```ini
[Service]
Type=notify
Restart=always
RestartSec=100ms
WatchdogSec=10s

FailureAction=reboot-force
OOMPolicy=stop

# 永不放弃重启
StartLimitIntervalSec=0
```

---

## 14. 设计要点

### 14.1 分层防护

```
第一层: Restart= 自动重启（毫秒级恢复）
第二层: StartLimitBurst= 频率限制（防止循环崩溃）
第三层: StartLimitAction= 系统级动作（最后手段）
第四层: FailureAction= 系统级动作（服务级别的熔断器）
```

### 14.2 看门狗的双重价值

- **死锁检测**：进程还在但主循环卡住时，内核信号无法发现
- **调试支持**：SIGABRT 默认产生 core dump，便于事后分析

### 14.3 退出状态的细粒度控制

通过 `SuccessExitStatus` + `RestartForceExitStatus` + `RestartPreventExitStatus` 三个配置，应用可以精确控制哪些退出码触发重启、哪些阻止重启、哪些视为成功。
