# systemd 关机流程深度分析（shutdown/reboot/poweroff）

## 1. 概述

systemd 的关机流程是启动流程的逆向镜像，但设计上更加复杂：需要有序停止所有服务、卸载文件系统、拆除存储设备、最终调用内核关机系统调用。整个流程分为两大阶段：

```
Phase 1: PID1 事件循环内 —— isolate 到关机 target，有序停止服务
Phase 2: systemd-shutdown 二进制 —— 杀死残余进程，卸载设备，reboot()
```

关键设计原则：
- **有序性**：通过 target 依赖链保证关键服务最后停止
- **超时保护**：`JobTimeoutSec=30min` + `JobTimeoutAction=*-force` 兜底
- **数据安全**：两次 `sync()` + 逐级卸载确保数据落盘
- **容错降级**：强制关机路径作为后备

---

## 2. 关机触发路径

### 2.1 用户发起

```
systemctl poweroff/reboot/halt
  → D-Bus: org.freedesktop.systemd1.Manager.PowerOff/Reboot/Halt()
    → PID1: m->objective = MANAGER_POWEROFF/REBOOT/HALT
      → manager_loop() 退出
```

D-Bus 方法实现: `dbus-manager.c:1537-1614`

### 2.2 logind 发起（推荐路径）

```
systemctl poweroff  (实际走 logind)
  → D-Bus: org.freedesktop.login1.Manager.PowerOff(interactive)
    → 检查 inhibitor locks
    → 检查 PolicyKit 授权
    → 启动 poweroff.target
```

logind 关机方法: `logind-dbus.c:1661-1684`

### 2.3 ManagerObjective 枚举

| 值 | 含义 | 触发来源 |
|----|------|---------|
| `MANAGER_REBOOT` | 重启 | D-Bus / 信号 / FailureAction |
| `MANAGER_POWEROFF` | 关机 | D-Bus / 信号 / FailureAction |
| `MANAGER_HALT` | 停机 | D-Bus / 信号 |
| `MANAGER_KEXEC` | kexec 热重启 | D-Bus / 信号 |

源文件: `manager.h:41-53`

### 2.4 信号触发（紧急路径）

PID1 响应以下实时信号：

| 信号 | 动作 | 源文件 |
|------|------|--------|
| `SIGRTMIN+0` | 进入 `default.target` | `manager.c:2824` |
| `SIGRTMIN+1` | 进入 `rescue.target` | `manager.c:2827` |
| `SIGRTMIN+2` | 进入 `emergency.target` | `manager.c:2830` |
| `SIGRTMIN+3` | 启动 `halt.target` | `manager.c:2833` |
| `SIGRTMIN+4` | 启动 `poweroff.target` | `manager.c:2836` |
| `SIGRTMIN+5` | 启动 `reboot.target` | `manager.c:2839` |
| `SIGRTMIN+6` | 启动 `kexec.target` | `manager.c:2842` |
| `SIGRTMIN+13` | 立即 halt（不走 target） | `manager.c:2848` |
| `SIGRTMIN+14` | 立即 poweroff | `manager.c:2851` |
| `SIGRTMIN+15` | 立即 reboot | `manager.c:2854` |
| `SIGRTMIN+16` | 立即 kexec | `manager.c:2857` |

`SIGRTMIN+13~16` 直接设置 `m->objective`，跳过 target isolate 流程。

---

## 3. 关机 Target 依赖链

### 3.1 Target 层级结构

```
         用户请求: systemctl reboot
                    │
                    ▼
            reboot.target                    ← AllowIsolate=yes
                    │
                    │ Requires= + After=
                    ▼
          systemd-reboot.service             ← 关键服务(无 ExecStart)
                    │
                    │ Requires= + After=
                    ▼
    ┌───────────────┼───────────────┐
    │               │               │
shutdown.target  umount.target  final.target
    │                               │
    │  DefaultDependencies=no       │  After=shutdown+umount
    │  RefuseManualStart=yes        │
    │                               │
```

### 3.2 各 Target 定义

#### reboot.target

```ini
# units/reboot.target
[Unit]
DefaultDependencies=no
Requires=systemd-reboot.service
After=systemd-reboot.service
AllowIsolate=yes
JobTimeoutSec=30min
JobTimeoutAction=reboot-force

[Install]
Alias=ctrl-alt-del.target
```

#### poweroff.target

```ini
# units/poweroff.target
[Unit]
DefaultDependencies=no
Requires=systemd-poweroff.service
After=systemd-poweroff.service
AllowIsolate=yes
JobTimeoutSec=30min
JobTimeoutAction=poweroff-force
```

#### systemd-reboot.service

```ini
# units/systemd-reboot.service
[Unit]
DefaultDependencies=no
Requires=shutdown.target umount.target final.target
After=shutdown.target umount.target final.target
SuccessAction=reboot-force
```

**关键设计**：该服务没有 `[Service]` 段和 `ExecStart=`，它只是一个依赖锚点。当 shutdown/umount/final 三个 target 都完成后，该服务立即变为 active(exited)，然后 `SuccessAction=reboot-force` 触发强制重启。

#### shutdown.target

```ini
# units/shutdown.target
DefaultDependencies=no
RefuseManualStart=yes
```

用途：作为 `Conflicts=shutdown.target` 的锚点。所有设置了 `DefaultDependencies=yes`（默认值）的单元自动获得 `Conflicts=shutdown.target` + `Before=shutdown.target`，这确保它们在关机时被停止。

### 3.3 DefaultDependencies 的关机效果

当一个 unit 设置 `DefaultDependencies=yes`（默认）时，自动添加：

```
Conflicts=shutdown.target
Before=shutdown.target
```

这意味着：当 `shutdown.target` 启动时，所有普通服务因 Conflicts 被停止，且它们必须在 shutdown.target 之前停止完毕。

---

## 4. 服务停止流程

### 4.1 Stop 状态机

```
RUNNING
  │
  │  unit_stop()
  ▼
SERVICE_STOP                ← 执行 ExecStop= 命令
  │
  │  ExecStop 完成/超时
  ▼
SERVICE_STOP_SIGTERM        ← 发送 SIGTERM 给主进程
  │
  │  等待 TimeoutStopSec
  ▼
SERVICE_STOP_SIGKILL        ← 发送 SIGKILL
  │
  │  收到 SIGCHLD
  ▼
SERVICE_STOP_POST           ← 执行 ExecStopPost= 命令
  │
  ▼
SERVICE_FINAL_SIGTERM       ← SIGTERM 给 ExecStopPost 进程
  │
  ▼
SERVICE_FINAL_SIGKILL       ← SIGKILL 给残留进程
  │
  ▼
SERVICE_DEAD                ← 清理完成
```

源文件: `service.c:2026-2058`（stop 入口）、`service.c:1962-2003`（SIGTERM/SIGKILL 升级）

### 4.2 超时处理

```c
// service.c:3888-3966
// 超时事件处理函数
case SERVICE_STOP:
    // ExecStop= 超时 → 直接进入 STOP_SIGTERM
case SERVICE_STOP_SIGTERM:
    // SIGTERM 等待超时 → 进入 STOP_SIGKILL
    if (s->kill_context.send_sigkill)
        service_enter_signal(s, SERVICE_STOP_SIGKILL, ...);
case SERVICE_STOP_SIGKILL:
    // SIGKILL 后仍未退出 → 直接进入 STOP_POST
```

### 4.3 停止顺序

PID1 根据作业依赖图并行停止服务，但遵循 `Before=/After=` 约束：

1. 用户服务先停止（依赖 `basic.target`）
2. 系统基础服务后停止（`sysinit.target` 的依赖）
3. 日志/D-Bus 等基础设施最后停止
4. `shutdown.target` 激活（标志性服务已停止完毕）
5. `umount.target` 处理卸载相关清理
6. `final.target` 最终完成

### 4.4 JobTimeoutSec 保护

```ini
# reboot.target
JobTimeoutSec=30min
JobTimeoutAction=reboot-force
```

如果关机作业超过 30 分钟未完成，`JobTimeoutAction` 直接设置 `m->objective = MANAGER_REBOOT`（强制路径），绕过正常 target 流程。

---

## 5. PID1 到 systemd-shutdown 的交接

### 5.1 objective 处理

当 `manager_loop()` 因 `m->objective != MANAGER_OK` 退出后：

```c
// main.c:2943-2959
r = invoke_main_loop(m, ...);
// r = MANAGER_REBOOT / MANAGER_POWEROFF / MANAGER_HALT / MANAGER_KEXEC
```

### 5.2 invoke_main_loop 内部

关机 objective 导致 PID1 执行以下操作：
1. 设置 `shutdown_verb` 为 "reboot"/"poweroff"/"halt"/"kexec"
2. 释放 Manager 对象
3. `execve("/usr/lib/systemd/systemd-shutdown", verb, timeout)`

PID1 不会调用 `exit()` — 它通过 `execve()` 将自身替换为 shutdown 二进制。

---

## 6. systemd-shutdown 最终关机

| 源文件 | `shutdown/shutdown.c:333-647` |
|--------|------------------------------|

### 6.1 执行条件

```c
// shutdown.c:359-363
if (getpid_cached() != 1) {
    log_error("Not executed by init (PID 1).");
    // 必须以 PID1 身份运行（由 PID1 execve 过来）
}
```

### 6.2 完整执行序列

```
┌─────────────────────────────────────────────────────────────┐
│  systemd-shutdown main()                                     │
├─────────────────────────────────────────────────────────────┤
│  1. 解析参数（verb = reboot/poweroff/halt/kexec）           │
│  2. 确定 reboot cmd (RB_AUTOBOOT/RB_POWER_OFF/RB_HALT)     │
│  3. 检测是否在容器中                                         │
│  4. init_watchdog() — 接管看门狗                             │
│  5. mlockall(MCL_CURRENT|MCL_FUTURE) — 锁定内存             │
│  6. sync_with_progress() — 第一次同步                        │
│  7. disable_coredumps() + disable_binfmt()                   │
│                                                              │
│  8. broadcast_signal(SIGTERM) — 通知所有残余进程             │
│  9. broadcast_signal(SIGKILL) — 强杀所有残余进程             │
│                                                              │
│  10. 循环卸载（直到全部完成或无进展）:                         │
│      ├── umount_all()          — 卸载文件系统                │
│      ├── swapoff_all()         — 停用 swap                   │
│      ├── loopback_detach_all() — 分离 loop 设备              │
│      ├── md_detach_all()       — 停止 MD RAID               │
│      └── dm_detach_all()       — 分离 Device Mapper          │
│                                                              │
│  11. execute_directories(SYSTEM_SHUTDOWN_PATH) — 执行钩子    │
│                                                              │
│  12. 若有 initramfs shutdown: switch_root + execv(/shutdown) │
│                                                              │
│  13. sync_with_progress() — 第二次同步                        │
│  14. reboot(cmd) — 调用内核关机                              │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 进程清杀

```c
// shutdown.c:411-415
log_info("Sending SIGTERM to remaining processes...");
broadcast_signal(SIGTERM, true, true, arg_timeout);

log_info("Sending SIGKILL to remaining processes...");
broadcast_signal(SIGKILL, true, false, arg_timeout);
```

`broadcast_signal()` 发送信号给所有进程（除 PID1 和内核线程），第三个参数控制是否等待进程退出。

### 6.4 设备卸载循环

关键设计：**循环重试**。因为设备间存在依赖关系（如 LVM 在 LUKS 之上），单次卸载可能失败，需要多轮迭代。

```c
// shutdown.c:424-532
for (;;) {
    bool changed = false;
    watchdog_ping();          // 每轮喂狗
    cg_trim(...);             // 裁剪 cgroup 树

    umount_all(&changed, ...);
    swapoff_all(&changed);
    loopback_detach_all(&changed, ...);
    md_detach_all(&changed, ...);
    dm_detach_all(&changed, ...);

    // 全部完成 → 退出循环
    if (!need_umount && !need_swapoff && !need_loop && !need_dm && !need_md)
        break;

    // 无进展 → 放弃
    if (!changed) break;
}
```

### 6.5 initramfs 回切

如果存在 `/run/initramfs/shutdown`，systemd-shutdown 会切换回 initramfs 执行清理：

```c
// shutdown.c:546-560
can_initrd = !in_container && !in_initrd()
             && access("/run/initramfs/shutdown", X_OK) == 0;

if (can_initrd) {
    switch_root_initramfs();   // pivot_root 到 /run/initramfs
    execv("/shutdown", argv);  // 在 initramfs 中执行 shutdown
}
```

这允许 initramfs 中的 shutdown 脚本完成最终清理（如 LUKS close）。

### 6.6 最终系统调用

```c
// shutdown.c:587-633
switch (cmd) {
case LINUX_REBOOT_CMD_KEXEC:
    execv(KEXEC, {"-e", NULL});  // 尝试 kexec 工具
    reboot(cmd);                  // 失败则直接 syscall
    // fallthrough to RB_AUTOBOOT

case RB_AUTOBOOT:
    reboot_with_parameter(REBOOT_LOG);  // 可能带 reboot=xxx 参数
    break;

case RB_POWER_OFF:
    // 关电
    break;

case RB_HALT_SYSTEM:
    // 停机（不关电）
    break;
}

reboot(cmd);  // 最终系统调用
```

---

## 7. logind 抑制锁（Inhibitor Locks）

### 7.1 抑制锁类型

| 类型 | 含义 | 用途举例 |
|------|------|---------|
| `shutdown` | 阻止关机 | 包管理器升级中 |
| `sleep` | 阻止休眠 | 音乐播放中 |
| `idle` | 阻止空闲关机 | 文件传输中 |
| `handle-power-key` | 阻止电源键 | 自定义电源管理 |
| `handle-suspend-key` | 阻止挂起键 | |
| `handle-lid-switch` | 阻止合盖动作 | |

### 7.2 抑制锁检查流程

```c
// logind-dbus.c:1754-1770
// 延迟型抑制锁：等待持有者释放（最长 InhibitDelayMaxSec）
// 阻止型抑制锁：需要用户确认或管理员权限覆盖
```

### 7.3 计划关机与 wall 消息

```c
// logind-dbus.c:1859-1880
setup_wall_message_timer()
// 在计划关机前定时发送 wall 消息通知所有终端用户
```

计划关机 D-Bus 方法: `logind-dbus.c:1958-1986`

---

## 8. Ctrl+Alt+Del 处理

### 8.1 触发机制

内核将 Ctrl+Alt+Del 转换为 SIGINT 发给 PID1：

```c
// manager.c:2703-2713
// 收到 SIGINT 时启动 ctrl-alt-del.target
manager_start_target(m, SPECIAL_CTRL_ALT_DEL_TARGET, JOB_REPLACE_IRREVERSIBLY);
```

### 8.2 速率限制

```c
// manager.c
m->ctrl_alt_del_ratelimit = (RateLimit) {
    .interval = 2 * USEC_PER_SEC,
    .burst = 7
};
```

2 秒内连按 7 次以上会触发速率限制，PID1 会发出警告并忽略后续按键。

### 8.3 默认绑定

`ctrl-alt-del.target` 默认是 `reboot.target` 的别名：

```ini
# units/reboot.target [Install]
Alias=ctrl-alt-del.target
```

可通过 `systemctl mask ctrl-alt-del.target` 禁用。

---

## 9. 紧急关机与看门狗重启

### 9.1 EmergencyAction

```c
// emergency-action.c:35-151
```

| 动作 | 效果 |
|------|------|
| `none` | 无操作 |
| `reboot` | 正常重启（走 target） |
| `reboot-force` | 强制重启（直接设 objective） |
| `reboot-immediate` | 立即重启（直接 reboot() syscall） |
| `poweroff` | 正常关机 |
| `poweroff-force` | 强制关机 |
| `poweroff-immediate` | 立即关机 |
| `exit` | 退出（用户实例） |
| `exit-force` | 强制退出 |

三级升级：
- `*`（无后缀）：启动对应 target，走正常流程
- `*-force`：设置 `m->objective`，跳过 target 直接进入关机
- `*-immediate`：直接调用 `reboot()`/`sync()+reboot()`，不经过 systemd-shutdown

### 9.2 看门狗超时处理

当硬件看门狗超时未被喂狗时，硬件直接重启系统。PID1 中的看门狗配置：

```ini
# system.conf
RuntimeWatchdogSec=0        # 运行时看门狗（0=禁用）
RebootWatchdogSec=10min     # 关机看门狗（关机时最大等待）
KExecWatchdogSec=0          # kexec 看门狗
```

关机阶段，PID1 将看门狗超时设置为 `RebootWatchdogSec`，如果关机过程超时（如卸载挂起），硬件看门狗会强制重启。

### 9.3 软件看门狗（WatchdogSec=）

服务级看门狗超时触发的动作链：

```
服务未及时 sd_notify("WATCHDOG=1")
  → service WatchdogSec 超时
    → SERVICE_FAILURE_WATCHDOG
      → 检查 FailureAction= / StartLimitAction=
        → 可能触发 reboot-force
```

---

## 10. 容器中的关机

### 10.1 检测

```c
// shutdown.c:382
in_container = detect_container() > 0;
```

### 10.2 容器特殊处理

容器中跳过以下操作：
- `sync_with_progress()` — 容器无独立块设备
- `broadcast_signal()` — 容器进程空间隔离
- `umount_all()/swapoff_all()` — 无独立挂载命名空间（通常）
- loop/MD/DM 分离

容器中的"关机"实际是：

```c
// shutdown.c:578-581
if (streq(arg_verb, "exit") && in_container) {
    return arg_exit_code;  // 直接退出，容器管理器处理清理
}
```

如果容器没有 `CAP_SYS_BOOT`：

```c
// shutdown.c:634-639
if (errno == EPERM && in_container) {
    return EXIT_SUCCESS;  // 退出即可，宿主清理
}
```

---

## 11. 关机钩子脚本

```c
// shutdown.c:542
static const char* const dirs[] = {SYSTEM_SHUTDOWN_PATH, NULL};
// SYSTEM_SHUTDOWN_PATH = "/usr/lib/systemd/system-shutdown"

execute_directories(dirs, DEFAULT_TIMEOUT_USEC, NULL, NULL, arguments, NULL,
                    EXEC_DIR_PARALLEL | EXEC_DIR_IGNORE_ERRORS);
```

在卸载设备后、最终 reboot() 前，执行 `/usr/lib/systemd/system-shutdown/` 下的所有脚本。脚本接收参数 `argv[1]` = "reboot"/"poweroff"/"halt"/"kexec"。

---

## 12. 完整关机时间线

```
T=0 ─────── 用户执行 systemctl poweroff
             │
             ├── logind 检查抑制锁
             ├── PolicyKit 授权
             │
T=auth ──── PID1 收到 poweroff.target isolate 请求
             │
             ├── 事务引擎创建关机事务
             │    ├── Conflicts=shutdown.target 展开
             │    │    → 所有普通服务获得 stop 作业
             │    ├── After= 约束建立停止顺序
             │    └── 作业安装到 run_queue
             │
T=stop ──── 开始并行停止服务
             │  ├── 用户服务（TimeoutStopSec 后升级 SIGKILL）
             │  ├── 系统服务
             │  └── 基础设施（journald, dbus）
             │
T=shutdown ── shutdown.target 激活
             │
T=umount ──── umount.target 激活
             │
T=final ───── final.target 激活
             │
T=service ─── systemd-poweroff.service 变为 active(exited)
             │  └── SuccessAction=poweroff-force 触发
             │
T=exec ────── PID1 execve("systemd-shutdown", "poweroff")
             │
             ├── sync_with_progress() — 第一次同步
             ├── broadcast_signal(SIGTERM)
             ├── broadcast_signal(SIGKILL)
             │
T=teardown ── 循环卸载
             │  ├── umount_all()
             │  ├── swapoff_all()
             │  ├── loopback_detach_all()
             │  ├── md_detach_all()
             │  └── dm_detach_all()
             │
             ├── execute_directories(/system-shutdown/)
             ├── [可选] switch_root → initramfs shutdown
             ├── sync_with_progress() — 第二次同步
             │
T=reboot ──── reboot(RB_POWER_OFF) — 内核关机
```

---

## 13. 超时保护总结

| 层级 | 超时 | 动作 |
|------|------|------|
| 服务 TimeoutStopSec | 默认 90s | SIGTERM → SIGKILL 升级 |
| 关机 target JobTimeoutSec | 30min | 触发 JobTimeoutAction（*-force） |
| RebootWatchdogSec | 可配（默认 10min） | 硬件看门狗强制重启 |
| broadcast_signal timeout | arg_timeout | 杀进程等待上限 |
| SYNC_TIMEOUT_USEC | 10s | sync 超时则继续 |

---

## 14. SuccessAction 机制

`systemd-reboot.service` 等关机服务使用 `SuccessAction=reboot-force` 而非 `ExecStart=systemctl --force reboot`，这是因为：

1. 该服务没有 `[Service]` 段 — 它是纯 Type=oneshot 默认行为
2. 当其 `Requires=shutdown.target umount.target final.target` 全部完成后，oneshot 服务因无 ExecStart 立即成功
3. `SuccessAction=reboot-force` 在服务成功时设置 `m->objective`

这种设计避免了在关机后期还需要 fork/exec 新进程的问题。

**例外**: `systemd-halt.service` 仍使用旧式 `ExecStart=systemctl --force halt`（历史原因）。

---

## 15. 设计哲学

1. **两阶段设计**：Phase1（PID1 内有序停止）+ Phase2（shutdown 二进制强制清理）确保既尊重服务生命周期，又保证最终一定能关机

2. **超时兜底层叠**：服务级超时 → target 级超时 → 硬件看门狗三层保护，避免"关不掉"

3. **数据安全优先**：两次 sync + 逐级卸载 + initramfs 回切三重保障数据落盘

4. **优雅降级**：normal → force → immediate 三级升级，从有序关机到暴力关机逐步降级

5. **循环重试**：设备卸载循环处理依赖关系，而非硬编码卸载顺序

6. **容器感知**：自动检测容器环境，跳过无意义的底层操作

7. **Conflicts 锚定**：通过 `DefaultDependencies` 自动为所有普通单元添加 `Conflicts=shutdown.target`，无需每个服务手动声明关机行为
