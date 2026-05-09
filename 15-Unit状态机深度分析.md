# Unit 状态机深度分析

> 每种 Unit 类型都有独立的状态机，但都通过 `unit_notify()` 映射到统一的
> `UnitActiveState`（7 种）上。本文档详细分析各类型 Unit 的状态定义、转换规则、
> SIGCHLD 驱动机制和自动重启逻辑。

---

## 一、统一的 UnitActiveState

所有 Unit 类型的内部状态都映射到以下 7 种统一状态（`unit-def.h:42-52`）：

```
UnitActiveState    │ 含义           │ 稳定/瞬态
───────────────────┼────────────────┼──────────
UNIT_ACTIVE        │ 正在运行       │ 稳定
UNIT_RELOADING     │ 正在重载       │ 瞬态
UNIT_INACTIVE      │ 未运行         │ 稳定
UNIT_FAILED        │ 失败           │ 稳定
UNIT_ACTIVATING    │ 正在启动       │ 瞬态
UNIT_DEACTIVATING  │ 正在停止       │ 瞬态
UNIT_MAINTENANCE   │ 维护/清理中    │ 瞬态
```

---

## 二、Service 状态机（最复杂）

### 2.1 状态枚举与映射

Service 有 19 个内部状态（`service.c:48-68`）：

```
ServiceState            │ UnitActiveState │ 含义
────────────────────────┼─────────────────┼──────────────
SERVICE_DEAD            │ INACTIVE        │ 初始/终止
SERVICE_CONDITION       │ ACTIVATING      │ 执行 ExecCondition=
SERVICE_START_PRE       │ ACTIVATING      │ 执行 ExecStartPre=
SERVICE_START           │ ACTIVATING      │ 执行 ExecStart=
SERVICE_START_POST      │ ACTIVATING      │ 执行 ExecStartPost=
SERVICE_RUNNING         │ ACTIVE          │ 主进程运行中
SERVICE_EXITED          │ ACTIVE          │ 主进程已退出(RemainAfterExit)
SERVICE_RELOAD          │ RELOADING       │ 执行 ExecReload=
SERVICE_STOP            │ DEACTIVATING    │ 执行 ExecStop=
SERVICE_STOP_WATCHDOG   │ DEACTIVATING    │ 看门狗超时信号
SERVICE_STOP_SIGTERM    │ DEACTIVATING    │ 发送 SIGTERM
SERVICE_STOP_SIGKILL    │ DEACTIVATING    │ 发送 SIGKILL
SERVICE_STOP_POST       │ DEACTIVATING    │ 执行 ExecStopPost=
SERVICE_FINAL_WATCHDOG  │ DEACTIVATING    │ 最终看门狗信号
SERVICE_FINAL_SIGTERM   │ DEACTIVATING    │ 最终 SIGTERM
SERVICE_FINAL_SIGKILL   │ DEACTIVATING    │ 最终 SIGKILL
SERVICE_FAILED          │ FAILED          │ 失败
SERVICE_AUTO_RESTART    │ ACTIVATING      │ 等待重启计时器
SERVICE_CLEANING        │ MAINTENANCE     │ 清理运行时目录
```

**Type=idle 特殊映射**（`service.c:72-92`）：
idle 类型的 CONDITION/START_PRE/START/START_POST 全部映射为 `UNIT_ACTIVE`（而非 ACTIVATING），
避免阻塞其他 Job 的执行。

### 2.2 完整状态转换图

```
                           service_start()
  DEAD/FAILED ──────────────────────────────────> CONDITION
                                                    │
                                          ExecCondition= 成功
                                                    │
                                                    ▼
                                               START_PRE
                                                    │
                                          ExecStartPre= 成功
                                                    │
                                                    ▼
                                                 START
                                              ╱    │    ╲
                           ┌─────────────────╱     │     ╲─────────────┐
                           │                       │                   │
                    simple/idle            forking/oneshot         dbus/notify/exec
                    立即→START_POST      等待进程退出/就绪     等待就绪信号
                           │                       │                   │
                           └─────────────┬─────────┘                   │
                                         │                             │
                                         ▼                             │
                                    START_POST ◄───────────────────────┘
                                         │
                               ExecStartPost= 成功
                                         │
                                   ┌─────┴─────┐
                                   │           │
                            service_good()  remain_after_exit
                              = true          = true
                                   │           │
                                   ▼           ▼
                               RUNNING      EXITED
                                   │           │
                           ┌───────┤           │
                           │       │           │
                      sd_notify    │           │
                     RELOADING     │           │
                           │       │           │
                           ▼       │           │
                        RELOAD     │           │
                           │       │           │
                    ExecReload=    │           │
                       成功        │           │
                           │       │           │
                           └───┬───┘           │
                               │               │
                          service_stop()        │
                               ├───────────────┘
                               ▼
                             STOP
                               │
                      ExecStop= 完成（或无）
                               │
                               ▼
                        STOP_SIGTERM ──(超时)──> STOP_SIGKILL
                               │                      │
                          进程退出                 进程退出
                               │                      │
                               ▼                      ▼
                          STOP_POST ◄─────────────────┘
                               │
                      ExecStopPost= 完成
                               │
                               ▼
                       FINAL_SIGTERM ──(超时)──> FINAL_SIGKILL
                               │                      │
                          进程退出                 进程退出
                               │                      │
                               └──────────┬───────────┘
                                          │
                                          ▼
                                    DEAD 或 FAILED
                                          │
                                   shall_restart()?
                                     ╱          ╲
                                   是             否
                                   │              │
                                   ▼              ▼
                             AUTO_RESTART       [终态]
                                   │
                            RestartSec 到期
                                   │
                                   ▼
                         service_enter_restart()
                               → JOB_RESTART
                               → 回到 DEAD→CONDITION→...
```

**看门狗路径**（STOP_WATCHDOG / FINAL_WATCHDOG）：
当看门狗超时时，先发送 SIGABRT（生成 coredump），然后等待 `WatchdogSignal=` 配置的超时后进入 SIGKILL。

### 2.3 Service 类型对启动流程的影响

`service_enter_start()`（`service.c:2179-2272`）：

| Type | ExecStart 进程身份 | START 行为 | 就绪条件 |
|------|-------------------|-----------|---------|
| **simple** | 主进程（main_pid） | 立即进入 START_POST | fork() 返回即就绪 |
| **idle** | 主进程 | 同 simple，但映射为 ACTIVE | 同 simple |
| **forking** | 控制进程（control_pid） | 停留在 START | 进程退出 + 发现主 PID |
| **oneshot** | 主进程 | 停留在 START | 所有 ExecStart= 全部退出 |
| **dbus** | 主进程 | 停留在 START | 总线名称出现 |
| **notify** | 主进程 | 停留在 START | 收到 READY=1 通知 |
| **exec** | 主进程 | 停留在 START | exec-fd EOF（exec 成功） |

### 2.4 service_good() — 判断服务是否健康

`service.c:2060-2078`：

```c
static bool service_good(Service *s) {
    // D-Bus 类型：必须总线名称已获取
    if (s->type == SERVICE_DBUS && !s->bus_name_good)
        return false;

    // 主进程存活 → 健康
    int main_pid_ok = main_pid_good(s);
    if (main_pid_ok > 0) return true;
    if (main_pid_ok == 0) return false;

    // 无主进程信息 → 检查 cgroup 是否还有进程
    return cgroup_good(s) != 0;
}
```

### 2.5 service_enter_running() — 进入运行状态的决策

`service.c:2080-2106`：

```
service_enter_running(s, f)
  │
  ├── result ≠ SUCCESS → 失败，进入 STOP_SIGTERM
  │
  ├── service_good() = true
  │     ├── notify_state == RELOADING → 进入 RELOAD
  │     ├── notify_state == STOPPING → 进入 STOP_SIGTERM
  │     └── 正常 → 设置 RUNNING，启动运行超时
  │
  ├── remain_after_exit = true → 设置 EXITED
  │
  └── 其他 → 进入 STOP（进程已退出且不保留）
```

---

## 三、SIGCHLD 驱动的状态转换

### 3.1 SIGCHLD 总体流程

```
内核: 子进程退出
  │
  ▼
sd_event: SIGCHLD 信号
  │
  ▼
manager_dispatch_sigchld()
  │
  ├── waitid() 收割子进程
  ├── 通过 manager->watch_pids 查找所属 Unit
  │
  └── 调用 Unit 类型的 sigchld_event() 回调
        ├── service_sigchld_event()
        ├── socket_sigchld_event()
        ├── mount_sigchld_event()
        └── swap_sigchld_event()
```

### 3.2 service_sigchld_event() 核心逻辑

`service.c:3467-3822`，根据退出的是**主进程**还是**控制进程**分别处理：

#### 主进程退出时的状态转换

```
当前状态           │ 主进程退出后
───────────────────┼─────────────────────────────
START              │ simple/idle: 不应发生（主进程在 START_POST）
                   │ forking: 不应发生（这里是控制进程）
                   │ notify/dbus/exec: 失败 → STOP_SIGTERM
                   │ oneshot: 运行下一个 ExecStart= 或 → START_POST
START_POST         │ 忽略主进程退出（等控制进程）
RUNNING            │ service_enter_running() 重新评估
                   │   → good: 保持 RUNNING
                   │   → 退出: EXITED 或 STOP
STOP               │ 记录退出状态，等控制进程退出
STOP_SIGTERM/KILL  │ 进入 STOP_POST（清理阶段）
STOP_POST          │ 进入 FINAL_SIGTERM
FINAL_*            │ 进入 DEAD
```

#### 控制进程退出时的状态转换

```
当前状态           │ 控制进程退出后
───────────────────┼─────────────────────────────
CONDITION          │ 成功 → START_PRE
                   │ 失败 → STOP_SIGTERM 或 DEAD(skip)
START_PRE          │ 成功 → START
                   │ 失败 → STOP_SIGTERM
START              │ (仅 forking 类型)
                   │ 成功 → START_POST（发现主 PID）
                   │ 失败 → STOP_SIGTERM
START_POST         │ 成功 → RUNNING/EXITED
                   │ 失败 → STOP
RELOAD             │ 成功/失败 → RUNNING
STOP               │ 完成 → STOP_SIGTERM
STOP_POST          │ 完成 → FINAL_SIGTERM 或 DEAD
```

---

## 四、自动重启机制

### 4.1 Restart= 配置与决策矩阵

`service_shall_restart()`（`service.c:1745-1793`）：

```
Restart= 值          │ 什么时候重启
─────────────────────┼────────────────────────────────────
no                   │ 从不（默认）
always               │ 总是（除了 ConditionXxx 跳过）
on-success           │ 仅 result == SUCCESS
on-failure           │ result 不是 SUCCESS 且不是 SKIP
on-abnormal          │ result 不是 SUCCESS/EXIT_CODE/SKIP
on-abort             │ result 是 SIGNAL 或 CORE_DUMP
on-watchdog          │ result 仅是 WATCHDOG
```

**完整决策真值表**：

```
退出原因 ╲ Restart=  │ no │always│success│failure│abnormal│abort│watchdog│
─────────────────────┼────┼──────┼───────┼───────┼────────┼─────┼────────┤
正常退出(0)          │    │  ✓   │   ✓   │       │        │     │        │
非零退出             │    │  ✓   │       │   ✓   │        │     │        │
信号杀死(非ABRT)     │    │  ✓   │       │   ✓   │   ✓    │  ✓  │        │
核心转储(ABRT)       │    │  ✓   │       │   ✓   │   ✓    │  ✓  │        │
看门狗超时           │    │  ✓   │       │   ✓   │   ✓    │     │   ✓    │
资源耗尽             │    │  ✓   │       │   ✓   │   ✓    │     │        │
```

### 4.2 重启优先级规则

```
检查顺序（service_shall_restart 内部）：
  1. forbid_restart = true → 不重启（手动 stop 时设置）
  2. restart_prevent_status 匹配 → 不重启（RestartPreventExitStatus=）
  3. restart_force_status 匹配 → 强制重启（RestartForceExitStatus=）
  4. 按 Restart= 配置判断
```

### 4.3 重启流程

```
service_enter_dead(s, result, allow_restart=true)
  │
  ├── unit_stop_pending() → allow_restart = false
  │
  ├── 记录 result → 决定 end_state = DEAD 或 FAILED
  │
  ├── service_shall_restart(s, &reason)
  │     → 决定是否重启
  │
  ├── 设置 end_state（先短暂进入 DEAD/FAILED）
  │     → 触发 unit_notify() → 更新时间戳、通知依赖
  │
  ├── will_auto_restart = true
  │     → 启动定时器：now() + restart_usec
  │     → 设置状态 SERVICE_AUTO_RESTART
  │
  └── 清理资源
        → exec_runtime 释放
        → 运行时目录删除
        → PID 文件删除
        → TTY 还原

                    ┌──── RestartSec 到期 ────┐
                    │                         │
                    ▼                         │
         service_enter_restart()              │
              │                               │
              ├── 检查是否有 STOP Job → 跳过   │
              │                               │
              ├── manager_add_job(JOB_RESTART) │
              │     → 事务引擎处理             │
              │     → 先 STOP 再 START         │
              │                               │
              ├── n_restarts++                 │
              │                               │
              └── 保持 AUTO_RESTART 状态       │
                    直到 JOB_RESTART 的 STOP   │
                    部分把状态拉回 DEAD         │
```

### 4.4 RestartSec= 和速率限制

```ini
[Service]
RestartSec=100ms      # 重启延迟（默认 100ms）
StartLimitIntervalSec=10s  # 速率限制窗口
StartLimitBurst=5          # 窗口内最大启动次数
StartLimitAction=none      # 超限动作（none/reboot/reboot-force/...）
```

---

## 五、其他 Unit 类型的状态机

### 5.1 Mount — 挂载点

12 个状态（`mount.c:37-50`）：

```
MountState              │ UnitActiveState
────────────────────────┼─────────────────
MOUNT_DEAD              │ INACTIVE
MOUNT_MOUNTING          │ ACTIVATING
MOUNT_MOUNTING_DONE     │ ACTIVATING
MOUNT_MOUNTED           │ ACTIVE
MOUNT_REMOUNTING        │ RELOADING
MOUNT_UNMOUNTING        │ DEACTIVATING
MOUNT_REMOUNTING_SIGTERM│ RELOADING
MOUNT_REMOUNTING_SIGKILL│ RELOADING
MOUNT_UNMOUNTING_SIGTERM│ DEACTIVATING
MOUNT_UNMOUNTING_SIGKILL│ DEACTIVATING
MOUNT_FAILED            │ FAILED
MOUNT_CLEANING          │ MAINTENANCE
```

状态转换：
```
DEAD ──> MOUNTING ──> MOUNTING_DONE ──> MOUNTED
                                           │
                              mount_enter_remounting()
                                           │
                                           ▼
                                      REMOUNTING
                                        │    ╲
                                   成功  │     ╲ 超时
                                        ▼      ▼
                                    MOUNTED  REMOUNTING_SIGTERM → SIGKILL
                                                                    │
                                                                    ▼
                                           ┌──────── MOUNTED ◄──────┘
                                           │
                              mount_enter_unmounting()
                                           │
                                           ▼
                                     UNMOUNTING ── 超时 ──> UNMOUNTING_SIGTERM → SIGKILL
                                        │                                         │
                                   成功  │                                        │
                                        ▼                                        ▼
                                    DEAD/FAILED ◄────────────────────────────────┘
```

**特殊性**：Mount 单元由 `/proc/self/mountinfo` 的 inotify 监控驱动，
PID 1 通过解析 mountinfo 来检测挂载状态变更，而非纯粹通过 SIGCHLD。

### 5.2 Socket — 套接字

14 个状态（详见 [14-Socket-Activation机制深度分析.md](14-Socket-Activation机制深度分析.md)）。

### 5.3 Timer — 定时器

5 个状态（`timer.c:27-33`）：

```
TimerState    │ UnitActiveState │ 含义
──────────────┼─────────────────┼────────────────
TIMER_DEAD    │ INACTIVE        │ 未启动
TIMER_WAITING │ ACTIVE          │ 等待触发时间
TIMER_RUNNING │ ACTIVE          │ 已触发，服务运行中
TIMER_ELAPSED │ ACTIVE          │ 已触发（RemainAfterElapse=）
TIMER_FAILED  │ FAILED          │ 失败
```

转换：
```
DEAD ──> WAITING ──(时间到达)──> RUNNING ──(服务完成)──> WAITING
                                    │                       │
                                    └── 或 ──> ELAPSED ─────┘
                                    (RemainAfterElapse=yes)
```

Timer 单元使用 `sd_event_add_time()` 设置单调/实时定时器，
到期后调用 `timer_enter_running()` 提交 `JOB_START` 启动关联服务。

### 5.4 Path — 路径监控

5 个状态（`path.c:28-33`）：

```
PathState      │ UnitActiveState │ 含义
───────────────┼─────────────────┼────────────────
PATH_DEAD      │ INACTIVE        │ 未启动
PATH_WAITING   │ ACTIVE          │ 等待路径事件
PATH_RUNNING   │ ACTIVE          │ 已触发，服务运行中
PATH_FAILED    │ FAILED          │ 失败
```

Path 使用 inotify 监控文件/目录的存在（PathExists）、
修改（PathChanged）、目录非空（DirectoryNotEmpty）等条件。

### 5.5 Scope — 作用域

6 个状态（`scope.c:22-29`）：

```
ScopeState         │ UnitActiveState │ 含义
───────────────────┼─────────────────┼────────────────
SCOPE_DEAD         │ INACTIVE        │ 未启动
SCOPE_RUNNING      │ ACTIVE          │ 进程组运行中
SCOPE_ABANDONED    │ ACTIVE          │ 控制器断开连接
SCOPE_STOP_SIGTERM │ DEACTIVATING    │ 停止中 SIGTERM
SCOPE_STOP_SIGKILL │ DEACTIVATING    │ 停止中 SIGKILL
SCOPE_FAILED       │ FAILED          │ 失败
```

**Scope 特殊性**：
- 不由 systemd 启动，而是外部进程（如 `systemd-run`）创建并交接进程组
- 没有 ExecStart/ExecStop 钩子
- 通过 D-Bus 控制器管理生命周期
- 典型用途：用户会话 scope、transient 运行

### 5.6 Swap — 交换分区

9 个状态（`swap.c:32-42`）：

```
SwapState                │ UnitActiveState │ 含义
─────────────────────────┼─────────────────┼────────────────
SWAP_DEAD                │ INACTIVE        │ 未激活
SWAP_ACTIVATING          │ ACTIVATING      │ swapon 执行中
SWAP_ACTIVATING_DONE     │ ACTIVE          │ swapon 完成
SWAP_ACTIVE              │ ACTIVE          │ 交换已激活
SWAP_DEACTIVATING        │ DEACTIVATING    │ swapoff 执行中
SWAP_DEACTIVATING_SIGTERM│ DEACTIVATING    │ 停止 SIGTERM
SWAP_DEACTIVATING_SIGKILL│ DEACTIVATING    │ 停止 SIGKILL
SWAP_FAILED              │ FAILED          │ 失败
SWAP_CLEANING            │ MAINTENANCE     │ 清理中
```

Swap 类似 Mount，通过 `/proc/swaps` 的 inotify 监控检测状态。

### 5.7 Automount — 自动挂载

6 个状态：

```
AutomountState      │ UnitActiveState │ 含义
────────────────────┼─────────────────┼────────────────
AUTOMOUNT_DEAD      │ INACTIVE        │ 未启动
AUTOMOUNT_WAITING   │ ACTIVE          │ 等待访问
AUTOMOUNT_RUNNING   │ ACTIVE          │ 已触发挂载
AUTOMOUNT_FAILED    │ FAILED          │ 失败
```

Automount 使用 `autofs` 内核模块，在目录被访问时自动触发关联的 Mount 单元。

### 5.8 Target / Device / Slice — 无状态机

这三种 Unit 类型没有独立的状态机：

| 类型 | 状态 | 说明 |
|------|------|------|
| Target | DEAD / ACTIVE | 同步点，无进程 |
| Device | DEAD / PLUGGED / TENTATIVE | 由 udev 事件驱动 |
| Slice | DEAD / ACTIVE | cgroup 层级节点，无进程 |

---

## 六、unit_notify() — 中央状态通知

`unit.c:2631-2782`，**所有 Unit 状态变更都通过此函数**：

```
unit_notify(u, old_state, new_state, flags)
  │
  ├── [1] D-Bus 通知入队
  │     unit_add_to_dbus_queue(u)
  │
  ├── [2] oomd 通知
  │     manager_varlink_send_managed_oom_update()
  │
  ├── [3] 时间戳更新
  │     ├── inactive → active: inactive_exit_timestamp
  │     ├── active → inactive: inactive_enter_timestamp
  │     ├── !active → active: active_enter_timestamp
  │     └── active → !active: active_exit_timestamp
  │
  ├── [4] 失败单元跟踪
  │     manager_update_failed_units()
  │
  ├── [5] cgroup 清理
  │     INACTIVE/FAILED → unit_prune_cgroup() + unlink_state_files()
  │
  ├── [6] Job 处理
  │     unit_process_job(u->job, new_state)
  │     → Job 根据新状态决定完成/继续等待
  │
  ├── [7] 意外状态变更的追溯处理
  │     ├── 意外激活 → retroactively_start_dependencies()
  │     │     启动 Requires/Wants/After 等依赖
  │     └── 意外停止 → retroactively_stop_dependencies()
  │           停止 PropagatesStopTo 依赖
  │
  ├── [8] OnFailure= / OnSuccess= 触发
  │     ├── 进入 FAILED → unit_start_on_failure("OnFailure=")
  │     └── 进入 INACTIVE（非失败/非重启）→ unit_start_on_failure("OnSuccess=")
  │
  ├── [9] 审计事件
  │     ├── 变为 ACTIVE → unit_emit_audit_start()
  │     └── 变为 INACTIVE/FAILED → unit_emit_audit_stop()
  │
  ├── [10] 紧急动作
  │     ├── → FAILED: emergency_action(failure_action)
  │     └── → INACTIVE: emergency_action(success_action)
  │
  └── [11] 依赖队列更新
        ├── INACTIVE/FAILED:
        │     check_unneeded_dependencies()
        │     check_bound_by_dependencies()
        │     submit_to_start_when_upheld_queue()
        │     unit_add_to_gc_queue()
        │
        └── ACTIVE/RELOADING:
              check_uphold_dependencies()
              submit_to_stop_when_unneeded_queue()
              submit_to_stop_when_bound_queue()
```

---

## 七、状态变更的传播效应

### 7.1 retroactively_start_dependencies()

当一个 Unit 意外变为活跃（非 Job 触发），追溯启动它的依赖：

```
Unit A 意外变为 ACTIVE
  │
  ├── 遍历 A 的 RETROACTIVE_START_ON_START 依赖
  │     → 对每个依赖 B：manager_add_job(START, B)
  │     示例：A 被手动启动，A.Wants=B → 也启动 B
  │
  └── 遍历 A 的 RETROACTIVE_STOP_ON_START 依赖
        → 对每个依赖 C：manager_add_job(STOP, C)
        示例：A 启动，A.Conflicts=C → 停止 C
```

### 7.2 retroactively_stop_dependencies()

当一个 Unit 意外停止：

```
Unit A 意外变为 INACTIVE
  │
  └── 遍历 A 的 RETROACTIVE_STOP_ON_STOP 依赖
        → 对每个依赖 B：manager_add_job(STOP, B)
        示例：B.BindsTo=A → A 停止 → B 也停止
```

### 7.3 check_unneeded_dependencies()

```
Unit A 变为 INACTIVE
  │
  └── 遍历所有通过 Wants/Requires 等依赖 A 的单元
        → 如果这些单元中没有任何活跃的 → A 的依赖是"不需要的"
        → 提交到 stop_when_unneeded_queue
```

---

## 八、Service 结果码

`ServiceResult` 决定了退出原因的分类（用于 Restart= 决策）：

```
ServiceResult                  │ 含义                    │ 来源
───────────────────────────────┼─────────────────────────┼──────────
SERVICE_SUCCESS                │ 成功退出                │ exit(0)
SERVICE_FAILURE_RESOURCES      │ 资源不足                │ fork/exec 失败
SERVICE_FAILURE_TIMEOUT        │ 超时                    │ TimeoutStartSec/StopSec
SERVICE_FAILURE_EXIT_CODE      │ 非零退出码              │ exit(非0)
SERVICE_FAILURE_SIGNAL         │ 被信号杀死              │ SIGTERM/SIGINT 等
SERVICE_FAILURE_CORE_DUMP      │ 核心转储                │ SIGABRT/SIGSEGV 等
SERVICE_FAILURE_WATCHDOG       │ 看门狗超时              │ WatchdogSec= 到期
SERVICE_FAILURE_START_LIMIT_HIT│ 启动频率超限            │ StartLimitBurst=
SERVICE_FAILURE_OOM_KILL       │ OOM 杀死               │ cgroup OOM
SERVICE_SKIP_CONDITION         │ 条件不满足              │ ExecCondition= 失败
```

---

## 九、状态持久化与恢复

### 9.1 序列化（daemon-reload / reexec）

当 PID 1 重新执行自身时，所有 Unit 状态通过序列化保存：

```c
// service_serialize() 保存：
serialize_item(f, "state", service_state_to_string(s->state));
serialize_item(f, "result", service_result_to_string(s->result));
serialize_item_format(f, "main-pid", PID_FMT, s->main_pid);
serialize_item_format(f, "control-pid", PID_FMT, s->control_pid);
serialize_item_format(f, "n-restarts", "%u", s->n_restarts);
// + FD、exec 状态等
```

### 9.2 反序列化（coldplug）

重新加载后，`service_coldplug()` 恢复状态：
- 重新 watch 主/控制进程的 PID
- 恢复定时器（超时、重启延迟）
- 重新注册 FD 事件源
- 如果状态是 `SERVICE_AUTO_RESTART`，恢复重启计时器

---

## 十、状态复杂度对比

| Unit 类型 | 内部状态数 | 有进程跟踪 | 有 ExecXxx 钩子 | 自动重启 |
|-----------|-----------|-----------|----------------|---------|
| Service | 19 | ✓ 主+控制 | ✓ 6个 | ✓ |
| Socket | 14 | ✓ 控制 | ✓ 5个 | ✗ |
| Mount | 12 | ✓ 控制 | ✗ | ✗ |
| Swap | 9 | ✓ 控制 | ✗ | ✗ |
| Scope | 6 | ✓ cgroup | ✗ | ✗ |
| Timer | 5 | ✗ | ✗ | ✗ |
| Path | 4 | ✗ | ✗ | ✗ |
| Automount | 4 | ✗ | ✗ | ✗ |
| Target | 2 | ✗ | ✗ | ✗ |
| Device | 3 | ✗ | ✗ | ✗ |
| Slice | 2 | ✗ | ✗ | ✗ |
