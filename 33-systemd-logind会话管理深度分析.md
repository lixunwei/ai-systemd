# systemd-logind 会话管理深度分析

## 1. 概述

systemd-logind 是 systemd 的登录和会话管理守护进程，负责：

- **会话（Session）管理**：追踪用户登录/注销，为每个会话创建 scope
- **座位（Seat）管理**：管理物理/虚拟终端设备分配
- **用户（User）管理**：启动/停止用户服务实例
- **设备访问控制**：管理 GPU/输入设备的会话级访问权限
- **电源管理**：处理电源键/盖子/休眠事件
- **抑制锁（Inhibitor）**：允许应用程序阻止关机/休眠

核心设计：以 D-Bus 为 IPC，以 cgroup scope 为进程追踪边界，以 FIFO fd 为会话生命周期锚。

---

## 2. 核心数据模型

### 2.1 对象关系图

```
Manager (全局单例)
├── seats[]          ← Hashmap: name → Seat
│   └── seat0 (主座位，拥有 VT)
├── sessions[]       ← Hashmap: id → Session
├── sessions_by_leader[] ← Hashmap: pid → Session
├── users[]          ← Hashmap: uid → User
├── inhibitors[]     ← Hashmap: id → Inhibitor
└── buttons[]        ← Hashmap: name → Button
```

源文件: `logind.h:25-139`

### 2.2 Session 结构体

```c
// logind-session.h:58-122
struct Session {
    Manager *manager;
    const char *id;          // 会话 ID (如 "c1", "2")
    unsigned position;       // 在 seat 上的位置
    SessionType type;        // TTY/X11/Wayland/Web
    SessionClass class;      // USER/GREETER/LOCK_SCREEN/BACKGROUND

    User *user;              // 所属用户
    Seat *seat;              // 所在座位
    unsigned vtnr;           // VT 编号
    int vtfd;                // VT 文件描述符

    char *scope;             // "session-c1.scope"
    pid_t leader;            // 会话领导进程 PID
    uint32_t audit_id;       // 审计会话 ID

    int fifo_fd;             // 生命周期 FIFO
    char *fifo_path;

    char *controller;        // 设备控制器（如 Wayland compositor）
    Hashmap *devices;        // 会话设备列表
    sd_bus_track *track;     // 控制器总线追踪

    bool idle_hint;
    bool locked_hint;
    bool started, stopping;
};
```

### 2.3 Session 状态机

```
SESSION_OPENING ──→ SESSION_ONLINE ──→ SESSION_ACTIVE
       │                   │                  │
       │                   │                  │ (切换到后台)
       │                   │                  ▼
       │                   │           SESSION_ONLINE
       │                   │                  │
       │                   │  (leader 退出)   │
       │                   ▼                  ▼
       └──────────→ SESSION_CLOSING ──→ [销毁]
```

| 状态 | 含义 | 触发条件 |
|------|------|---------|
| `SESSION_OPENING` | scope 正在创建 | CreateSession 调用后 |
| `SESSION_ONLINE` | 已登录，后台 | scope 创建完成 |
| `SESSION_ACTIVE` | 已登录，前台 | 成为 seat 的活动会话 |
| `SESSION_CLOSING` | 已注销，scope 残留 | leader 退出 |

源文件: `logind-session.h:12-19`

### 2.4 SessionType 和 SessionClass

| SessionType | 用途 |
|-------------|------|
| `SESSION_TTY` | 文本终端登录 |
| `SESSION_X11` | X11 图形会话 |
| `SESSION_WAYLAND` | Wayland 图形会话 |
| `SESSION_MIR` | Mir 图形会话 |
| `SESSION_WEB` | Web 远程会话 |
| `SESSION_UNSPECIFIED` | 未指定 |

| SessionClass | 用途 |
|-------------|------|
| `SESSION_USER` | 正常用户会话 |
| `SESSION_GREETER` | 登录管理器 |
| `SESSION_LOCK_SCREEN` | 锁屏会话 |
| `SESSION_BACKGROUND` | 后台会话（如 cron） |

### 2.5 User 结构体

```c
// logind-user.h:22-51
struct User {
    Manager *manager;
    UserRecord *user_record;

    char *slice;                  // "user-1000.slice"
    char *service;                // "user@1000.service"
    char *runtime_dir_service;    // "user-runtime-dir@1000.service"

    Session *display;             // 显示会话（最活跃的图形会话）

    dual_timestamp timestamp;
    usec_t last_session_timestamp;

    bool started, stopping;
    LIST_HEAD(Session, sessions);  // 该用户的所有会话
};
```

### 2.6 User 状态机

| 状态 | 含义 |
|------|------|
| `USER_OFFLINE` | 未登录 |
| `USER_OPENING` | 正在登录 |
| `USER_LINGERING` | 启用了 linger，无活动会话 |
| `USER_ONLINE` | 有会话，但都在后台 |
| `USER_ACTIVE` | 有前台会话 |
| `USER_CLOSING` | 所有会话已关闭，进程残留清理中 |

源文件: `logind-user.h:11-20`

---

## 3. 会话生命周期

### 3.1 创建流程

```
用户登录
  │
  ├─ PAM: pam_systemd.so
  │    └── D-Bus 调用 → org.freedesktop.login1.Manager.CreateSession()
  │
  ▼
logind: method_create_session()    [logind-dbus.c:692-988]
  ├── 验证参数（uid, pid, type, class, seat, vtnr, tty, display）
  ├── 查找/创建 User 对象
  ├── 生成 Session ID
  │    ├── 优先使用 audit session id
  │    └── 否则 "c%lu" (m->session_counter++)
  ├── session_new() → 分配 Session 对象
  ├── session_set_user() → 关联用户
  ├── session_set_leader() → 记录 leader PID
  ├── seat_attach_session() → 关联座位
  ├── session_start()       → 启动会话
  │    ├── user_start()     → 启动用户服务(首次)
  │    ├── session_start_scope() → 创建 session-N.scope
  │    ├── session_save()   → 持久化状态
  │    └── 发送 SessionNew 信号
  └── 返回 session_id, object_path, seat, vtnr, fifo_fd
```

### 3.2 Session ID 生成策略

```c
// logind-dbus.c:864-888
// 1. 若有 audit session ID → 使用审计 ID（字符串形式）
// 2. 否则 → "c" + 递增计数器（如 c1, c2, c3...）
```

### 3.3 Scope 单元创建

```c
// logind-session.c:642-687
session_start_scope()
  → manager_start_scope(
        scope = "session-c1.scope",
        leader_pid,
        user_slice,          // user-1000.slice
        description,
        properties: {After=, Requires=} on user@.service
    );
```

Scope 在 `user-UID.slice` 下创建，确保 cgroup 隔离。

### 3.4 FIFO 生命周期锚

```c
// CreateSession 返回一个 FIFO fd 给 PAM
// PAM 持有该 fd 直到会话结束
// 当 PAM 关闭 fd（或 PAM 进程退出）→ FIFO 被 hang up

// logind-session.c:1047-1057
session_dispatch_fifo()
// FIFO EOF → 触发 session_stop()
```

这是 logind 检测会话结束的主要机制：PAM 模块关闭 FIFO fd 表示会话结束。

### 3.5 销毁流程

```
会话结束触发（以下任一）:
  ├── FIFO EOF (PAM 关闭 fd)
  ├── Leader PID 退出 (SIGCHLD 通知)
  ├── ReleaseSession D-Bus 调用
  └── release_timeout 超时
         │
         ▼
session_stop()                    [logind-session.c:793-829]
  ├── session_set_state(CLOSING)
  ├── 停止 scope 单元 (kill cgroup)
  └── session_save()
         │
         ▼
session_finalize()                [logind-session.c:831-875]
  ├── session_release_controller()  ← 释放设备控制器
  ├── session_device_free_all()     ← 释放所有设备
  ├── seat_remove_session()         ← 从 seat 移除
  ├── user_remove_session()         ← 从 user 移除
  ├── unlink(state_file)
  └── 加入 gc_queue
         │
         ▼
session_free()                    [logind-session.c:87-153]
  ├── 从 Manager hashmaps 移除
  ├── 关闭所有 fd
  └── 释放内存
```

---

## 4. 座位（Seat）与 VT 管理

### 4.1 座位检测

通过 udev 标签 `master-of-seat` 识别座位主设备：

```c
// logind.c:181-207
// 枚举所有带 "master-of-seat" 标签的设备
// 对应设备的 seat 属性决定归属哪个 seat
```

`seat0` 是默认主座位，始终存在且拥有 VT 设备。

### 4.2 VT 切换

```c
// logind-seat.c:387-416
// 监控 /sys/class/tty/tty0/active 获知当前活动 VT

// logind-seat.c:351-380
seat_active_vt_changed()
  → 根据 VT 号找到对应 session
  → seat_set_active(session)
```

### 4.3 会话激活

```c
// logind-seat.c:227-267
seat_set_active(seat, session)
  ├── 暂停旧活动会话的设备: session_device_pause_all(old)
  ├── 设置新活动会话
  ├── 恢复新会话的设备: session_device_resume_all(new)
  ├── 发送 PropertyChanged 信号
  └── 更新 seat 状态文件
```

### 4.4 自动 VT 分配

```c
// logind-seat.c:185-208
seat_preallocate_vts()
// 为配置的 n_autovts 个 VT 预分配 getty 服务

// logind-seat.c:536-553
seat_attach_session()
// 新会话加入 seat 时，非 VT seat 自动激活
```

### 4.5 多座位支持

每个座位独立管理其会话列表和活动会话：
- `seat0`：拥有 VT，通常是内置显示器
- `seatN`：USB 外接显示器 + 键鼠，通过 `loginctl attach` 分配

---

## 5. 抑制锁（Inhibitor Locks）

### 5.1 抑制类型

```c
// logind-inhibit.h:6-17
INHIBIT_SHUTDOWN             = 1 << 0,  // 阻止关机
INHIBIT_SLEEP                = 1 << 1,  // 阻止休眠
INHIBIT_IDLE                 = 1 << 2,  // 阻止空闲
INHIBIT_HANDLE_POWER_KEY     = 1 << 3,  // 阻止电源键
INHIBIT_HANDLE_SUSPEND_KEY   = 1 << 4,  // 阻止挂起键
INHIBIT_HANDLE_HIBERNATE_KEY = 1 << 5,  // 阻止休眠键
INHIBIT_HANDLE_LID_SWITCH    = 1 << 6,  // 阻止合盖
INHIBIT_HANDLE_REBOOT_KEY    = 1 << 7,  // 阻止重启键
```

### 5.2 抑制模式

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `INHIBIT_BLOCK` | 完全阻止操作 | 包管理器升级 |
| `INHIBIT_DELAY` | 延迟操作（最长 InhibitDelayMaxSec） | 保存文件 |

### 5.3 Inhibitor 结构体

```c
// logind-inhibit.h:28-50
struct Inhibitor {
    const char *id;
    InhibitWhat what;     // 阻止什么
    char *who;            // 谁在阻止
    char *why;            // 为什么阻止
    InhibitMode mode;     // block / delay
    pid_t pid;            // 持有者 PID
    uid_t uid;            // 持有者 UID
    int fifo_fd;          // 生命周期 FIFO（fd 关闭 = 释放锁）
};
```

### 5.4 基于 FD 的生命周期

```
应用程序:
  fd = Inhibit("shutdown", "Package Manager", "Upgrading...", "block")
  // ... 执行升级 ...
  close(fd)  // 释放抑制锁

logind:
  // 创建 FIFO，返回 fd 给调用者
  // 监控 FIFO：fd 关闭 → 自动释放 Inhibitor
  // 若进程崩溃 → fd 被内核回收 → 锁自动释放
```

优势：进程崩溃时抑制锁自动释放，无需手动清理。

### 5.5 InhibitDelayMaxSec

delay 模式的最大等待时间（默认 5s）。超时后无论应用是否释放锁，操作继续执行。

---

## 6. 电源管理

### 6.1 按键/盖子事件处理

```c
// logind-button.c:121-183
// 通过 /dev/input 事件监听电源键/挂起键/盖子状态
```

### 6.2 默认动作配置

| 事件 | 配置项 | 默认值 |
|------|--------|--------|
| 电源键 | `HandlePowerKey=` | poweroff |
| 挂起键 | `HandleSuspendKey=` | suspend |
| 休眠键 | `HandleHibernateKey=` | hibernate |
| 重启键 | `HandleRebootKey=` | reboot |
| 合盖 | `HandleLidSwitch=` | suspend |
| 合盖(对接) | `HandleLidSwitchDocked=` | ignore |
| 合盖(外接电源) | `HandleLidSwitchExternalPower=` | (同HandleLidSwitch) |

源文件: `logind-core.c:41-63`

### 6.3 合盖策略判断

```c
// logind-button.c:78-93
if (docked || external_display)
    action = handle_lid_switch_docked;    // 通常 ignore
else if (on_external_power && handle_lid_switch_ep configured)
    action = handle_lid_switch_ep;
else
    action = handle_lid_switch;           // 通常 suspend
```

### 6.4 空闲动作

```c
// logind.c:953-980
// 当所有会话报告 idle 状态超过 IdleActionSec 后
// 执行 IdleAction（默认 ignore，可配为 suspend/poweroff 等）
```

`IdleActionSec` 默认 30min。

---

## 7. 设备访问控制

### 7.1 设备控制器概念

图形 compositor（如 Weston/Mutter）通过 `TakeControl` 成为会话的设备控制器：

```
Compositor:
  TakeControl(session)     → 获得 /dev/dri, /dev/input 访问权
  TakeDevice(major, minor) → 获得指定设备的 fd
  ReleaseDevice(...)       → 归还设备
  ReleaseControl(session)  → 放弃控制权
```

源文件: `logind-session-dbus.c:345-388`

### 7.2 VT 切换时的设备暂停/恢复

```c
// logind-seat.c:227-267
// 切换 VT 时:
session_device_pause_all(old_session);   // 暂停旧会话设备
session_device_resume_all(new_session);  // 恢复新会话设备
```

这使得切换 VT 时，旧 compositor 的 GPU 访问被收回，新 compositor 获得访问权。

### 7.3 设备暂停通知

```c
// logind-session-device.c:28-90
// 向控制器发送 PauseDevice / ResumeDevice D-Bus 信号
// Compositor 收到 PauseDevice 后释放 DRM master
// 收到 ResumeDevice 后重新获取 DRM master
```

---

## 8. 用户服务集成

### 8.1 用户实例启动

```c
// logind-user.c:342-358
user_start_service()
  → 启动 user@UID.service
  // 该服务运行 "systemd --user" 实例
```

### 8.2 关联的系统服务

| 服务 | 用途 |
|------|------|
| `user@UID.service` | 运行用户 systemd 实例 |
| `user-runtime-dir@UID.service` | 创建 /run/user/UID |
| `user-UID.slice` | 用户 cgroup 切片 |

### 8.3 用户启动/停止生命周期

```
首次登录:
  session_start()
    → user_start()                [logind-user.c:446-482]
        → user_start_service()    (启动 user@.service)
        → user_save()
        → 发送 UserNew 信号

最后一个会话关闭:
  user_stop()                     [logind-user.c:501-520+]
    → 停止 user@.service
    → 停止 user-runtime-dir@.service
    → user_save()
    → 发送 UserRemoved 信号
```

### 8.4 Linger 支持

```bash
loginctl enable-linger <user>
# 在 /var/lib/systemd/linger/ 下创建用户名文件
```

效果：即使用户无活动会话，`user@.service` 仍然运行，用户定时器和服务保持活跃。

```c
// logind.c:286-312
// 启动时枚举 /var/lib/systemd/linger/ 目录
// 为 linger 用户创建 User 对象并启动服务
```

---

## 9. 状态持久化

logind 将所有对象状态持久化到 `/run/systemd/sessions/`、`/run/systemd/users/`、`/run/systemd/seats/`：

```
/run/systemd/sessions/c1    ← 会话状态文件
/run/systemd/users/1000     ← 用户状态文件
/run/systemd/seats/seat0    ← 座位状态文件
/run/systemd/inhibit/1      ← 抑制锁状态文件
```

这确保 logind 重启后可以恢复追踪已有会话。

---

## 10. D-Bus 接口概览

### 10.1 Manager 接口 (org.freedesktop.login1.Manager)

| 方法 | 用途 |
|------|------|
| `CreateSession(...)` | 创建新会话 |
| `ReleaseSession(id)` | 释放会话 |
| `ActivateSession(id)` | 激活会话到前台 |
| `LockSession(id)` | 锁定会话 |
| `UnlockSession(id)` | 解锁会话 |
| `PowerOff(interactive)` | 关机 |
| `Reboot(interactive)` | 重启 |
| `Suspend(interactive)` | 挂起 |
| `Hibernate(interactive)` | 休眠 |
| `Inhibit(what,who,why,mode)` | 创建抑制锁 |
| `ListSessions()` | 列出所有会话 |
| `ListUsers()` | 列出所有用户 |
| `ListSeats()` | 列出所有座位 |

### 10.2 Session 接口 (org.freedesktop.login1.Session)

| 方法 | 用途 |
|------|------|
| `Activate()` | 激活到前台 |
| `Lock()` / `Unlock()` | 锁定/解锁 |
| `SetIdleHint(b)` | 设置空闲提示 |
| `TakeControl(force)` | 获取设备控制权 |
| `ReleaseControl()` | 释放设备控制权 |
| `TakeDevice(maj,min)` | 获取设备 fd |
| `ReleaseDevice(maj,min)` | 释放设备 |
| `Kill(who,signal)` | 杀死会话进程 |
| `Terminate()` | 终止会话 |

---

## 11. 与 PAM 的集成

### 11.1 pam_systemd.so 工作流

```
登录时 (pam_open_session):
  1. 收集环境信息（TTY, DISPLAY, seat, vtnr, service）
  2. D-Bus 调用 CreateSession()
  3. 保存返回的 session_id 到 PAM 环境
  4. 保持 FIFO fd 打开
  5. 设置 XDG_SESSION_ID, XDG_RUNTIME_DIR 等环境变量

注销时 (pam_close_session):
  1. 关闭 FIFO fd → logind 检测到 EOF
  2. logind 触发 session_stop()
```

### 11.2 PAM 设置的环境变量

| 变量 | 来源 |
|------|------|
| `XDG_SESSION_ID` | session id |
| `XDG_RUNTIME_DIR` | /run/user/UID |
| `XDG_SESSION_TYPE` | tty/x11/wayland |
| `XDG_SESSION_CLASS` | user/greeter |
| `XDG_SEAT` | seat0 等 |
| `XDG_VTNR` | VT 编号 |

---

## 12. 设计哲学

1. **FD 生命周期绑定**：会话和抑制锁的生命周期绑定到 FIFO fd，利用内核 fd 引用计数自动处理崩溃清理

2. **Scope 进程追踪**：通过 cgroup scope 精确追踪会话所有子进程，无论 fork 多少层都不会逃逸

3. **声明式设备访问**：通过 TakeControl/TakeDevice 机制，compositor 无需 root 权限即可访问 GPU 和输入设备

4. **状态持久化**：所有运行时状态写入 `/run/systemd/`，logind 重启后可完整恢复，会话不中断

5. **多座位原生支持**：对象模型中 Seat 是一等公民，天然支持多用户同时使用不同物理终端

6. **策略与机制分离**：logind 提供 D-Bus 接口（机制），具体策略由 desktop environment 和配置文件决定（如合盖动作、电源键响应）

7. **无特权 compositor**：设备暂停/恢复协议使 Wayland compositor 可以在非 root 下安全运行，通过 logind 中介获得设备访问
