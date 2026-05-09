# 登录管理分析 — systemd-logind

## 一、概述

`systemd-logind` 管理用户登录会话（Session）、座位（Seat）和用户（User），是多用户系统管理的核心。它取代了传统的 ConsoleKit，提供：

- 会话跟踪与管理
- 座位管理（多座位支持）
- 设备访问控制（ACL）
- 电源管理（关机/重启/休眠）抑制锁
- 空闲检测与自动操作

源码位于 `src/login/`，D-Bus 名称 `org.freedesktop.login1`。

---

## 二、核心数据模型

```
Manager
  ├── Seat (座位)
  │     ├── Session (会话)  ←─── 多个会话可属于同一座位
  │     └── Device (设备)   ←─── 分配给座位的设备
  ├── User (用户)
  │     └── Session (会话)  ←─── 用户的所有会话
  ├── Inhibitor (抑制锁)
  └── Button (按钮)         ←─── 电源/休眠按钮
```

### 2.1 关键文件

| 文件 | 职责 |
|------|------|
| logind.c | 守护进程入口 |
| logind.h | Manager 结构体定义 |
| logind-core.c | 核心管理逻辑 |
| logind-session.c/h | Session 对象管理 |
| logind-user.c/h | User 对象管理 |
| logind-seat.c/h | Seat 对象管理 |
| logind-device.c/h | Device 分配管理 |
| logind-inhibit.c/h | 抑制锁管理 |
| logind-button.c/h | 电源按钮事件处理 |
| logind-action.c/h | 电源操作执行 |
| logind-dbus.c | Manager D-Bus 接口 |
| logind-session-dbus.c | Session D-Bus 接口 |
| logind-user-dbus.c | User D-Bus 接口 |
| logind-seat-dbus.c | Seat D-Bus 接口 |
| pam_systemd.c | PAM 模块（创建会话） |
| inhibit.c | systemd-inhibit 命令 |

### 2.2 Manager 结构体

```c
struct Manager {
    sd_event *event;
    sd_bus *bus;

    // === 对象集合 ===
    Hashmap *seats;           // 座位（按名称）
    Hashmap *sessions;        // 会话（按 ID）
    Hashmap *users;           // 用户（按 UID）
    Hashmap *inhibitors;      // 抑制锁（按 ID）
    Hashmap *buttons;         // 按钮设备

    // === 设备监控 ===
    sd_device_monitor *device_monitor;  // udev 设备监控

    // === 会话跟踪 ===
    LIST_HEAD(Session, session_gc_queue);  // 待 GC 会话

    // === 电源管理 ===
    HandleAction handle_power_key;      // 电源键操作
    HandleAction handle_suspend_key;    // 休眠键操作
    HandleAction handle_hibernate_key;  // 冬眠键操作
    HandleAction handle_lid_switch;     // 合盖操作
    HandleAction handle_lid_switch_ep;  // 外部电源下合盖操作

    // === 空闲管理 ===
    usec_t idle_action_usec;            // 空闲超时
    HandleAction idle_action;           // 空闲操作

    // === 状态 ===
    bool session_jobs_pending;
    char **kill_exclude_users;
    char **kill_only_users;
    ...
};
```

---

## 三、会话生命周期

### 3.1 会话创建

```
用户登录（通过 login/gdm/sshd 等）
  ↓
PAM 模块 pam_systemd.so 被调用
  ↓
pam_sm_open_session()
  ├── 收集用户信息（UID、PID、TTY、显示器等）
  ├── D-Bus 调用 → logind Manager.CreateSession()
  │     ├── 创建 Session 对象
  │     ├── 关联到 User 和 Seat
  │     ├── 创建 scope 单元（session-N.scope）
  │     ├── 分配设备访问权限
  │     └── 通知 PID 1
  └── 设置 XDG_SESSION_ID 等环境变量
```

### 3.2 会话状态机

```
        创建
         │
         ▼
      opening ──→ online ──→ active
                    │           │
                    │     ←─────┘ （切换到其他会话）
                    ▼
                 closing
                    │
                    ▼
                  ended
```

- **active**：当前前台会话（拥有 VT 控制权）
- **online**：后台活跃会话
- **closing**：正在关闭
- **ended**：已结束

### 3.3 会话销毁

```
用户退出
  ↓
所有进程退出 → scope 单元为空
  ↓
logind 检测到 scope 空 → session_finalize()
  ├── 释放设备访问权限
  ├── 取消 VT 分配（如果适用）
  ├── 更新 User 状态
  ├── 发送 SessionRemoved 信号
  └── 清理 Session 对象
```

---

## 四、座位管理

座位（Seat）表示一组输入/输出设备的集合。典型场景：

| 座位 | 含义 |
|------|------|
| seat0 | 默认座位（主显示器+键盘） |
| seatN | 额外座位（多用户共用一台机器） |

### 设备分配

```
设备 uevent
  ↓
udev 标记 ID_SEAT=seatN
  ↓
logind 的 device_monitor 收到事件
  ↓
manager_process_seat_device()
  ├── 创建或更新 Seat
  ├── 设备关联到 Seat
  └── 更新 ACL 权限
```

---

## 五、抑制锁（Inhibitor）

抑制锁允许应用阻止特定电源操作：

| 锁类型 | 含义 |
|--------|------|
| shutdown | 阻止关机 |
| sleep | 阻止休眠 |
| idle | 阻止空闲检测 |
| handle-power-key | 阻止电源键操作 |
| handle-suspend-key | 阻止休眠键 |
| handle-hibernate-key | 阻止冬眠键 |
| handle-lid-switch | 阻止合盖操作 |

### 锁模式

| 模式 | 行为 |
|------|------|
| block | 完全阻止操作 |
| delay | 延迟操作（最多 InhibitDelayMaxSec，默认 5 秒） |

### 使用示例

```bash
# 阻止关机（软件更新期间）
systemd-inhibit --what=shutdown --who="updater" --why="Updating" apt upgrade

# 程序化使用
sd_bus_call_method(bus,
    "org.freedesktop.login1",
    "/org/freedesktop/login1",
    "org.freedesktop.login1.Manager",
    "Inhibit",
    &error, &reply,
    "ssss",
    "shutdown",        // what
    "my-app",          // who
    "Saving data",     // why
    "block");          // mode
```

---

## 六、电源管理

### 6.1 操作类型

| 操作 | 方法 | 效果 |
|------|------|------|
| PowerOff | Manager.PowerOff() | 关机 |
| Reboot | Manager.Reboot() | 重启 |
| Suspend | Manager.Suspend() | 挂起（到内存） |
| Hibernate | Manager.Hibernate() | 冬眠（到磁盘） |
| HybridSleep | Manager.HybridSleep() | 混合睡眠 |
| SuspendThenHibernate | Manager.SuspendThenHibernate() | 先挂起后冬眠 |

### 6.2 操作流程

```
电源键按下 / D-Bus 请求
  ↓
logind-action.c → manager_handle_action()
  ├── 检查抑制锁
  │     有 block 锁 → 拒绝操作
  │     有 delay 锁 → 延迟最多 InhibitDelayMaxSec
  │     无锁 → 继续
  ├── 发送 PrepareForShutdown/PrepareForSleep 信号
  ├── 通过 PID 1 执行操作
  │     D-Bus → systemd1.Manager.StartUnit()
  │     启动 systemd-poweroff.service / systemd-suspend.service 等
  └── 等待操作完成
```

---

## 七、配置文件

主配置：`/etc/systemd/logind.conf`

```ini
[Login]
NAutoVTs=6                          # 自动分配的 VT 数量
ReserveVT=6                         # 保留给 gdm 的 VT
KillUserProcesses=no                # 用户退出时是否 kill 进程
KillOnlyUsers=                      # 仅 kill 这些用户的进程
KillExcludeUsers=root               # 排除 kill 的用户
InhibitDelayMaxSec=5                # 延迟抑制最大时间
UserStopDelaySec=10                 # 用户停止延迟
HandlePowerKey=poweroff             # 电源键操作
HandleSuspendKey=suspend            # 休眠键操作
HandleHibernateKey=hibernate        # 冬眠键操作
HandleLidSwitch=suspend             # 合盖操作
HandleLidSwitchExternalPower=suspend # 外部电源下合盖
IdleAction=ignore                   # 空闲操作
IdleActionSec=30min                 # 空闲超时
```
