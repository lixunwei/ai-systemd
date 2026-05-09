# 公共库分析 — libsystemd

## 一、概述

`libsystemd` 是 systemd 提供的核心公共库，位于 `src/libsystemd/`，提供以下 API 子系统：

| API | 头文件 | 功能 |
|-----|--------|------|
| sd-bus | sd-bus.h | D-Bus 客户端/服务端通信 |
| sd-event | sd-event.h | 事件循环（epoll 封装） |
| sd-journal | sd-journal.h | 日志读写 |
| sd-device | sd-device.h | 设备模型访问 |
| sd-login | sd-login.h | 登录会话信息 |
| sd-daemon | sd-daemon.h | 守护进程辅助（通知、socket激活） |
| sd-id128 | sd-id128.h | 128-bit ID 操作 |
| sd-netlink | sd-netlink.h | Netlink 通信 |
| sd-hwdb | sd-hwdb.h | 硬件数据库 |
| sd-resolve | sd-resolve.h | 异步 DNS 解析 |
| sd-path | sd-path.h | 标准路径查询 |
| sd-network | sd-network.h | 网络状态查询 |
| sd-dhcp* | sd-dhcp-client.h 等 | DHCP 客户端/服务端 |
| sd-lldp* | sd-lldp.h | LLDP 协议 |

---

## 二、sd-bus — D-Bus 通信

### 2.1 核心实现文件

| 文件 | 职责 |
|------|------|
| sd-bus.c | sd_bus 连接管理（创建、连接、关闭） |
| bus-message.c | D-Bus 消息构造与解析 |
| bus-objects.c | 对象路径注册与方法分发 |
| bus-introspect.c | D-Bus 自省（introspection） |
| bus-socket.c | 基于 Unix socket 的 D-Bus 传输 |
| bus-kernel.c | 基于 kdbus 的传输（已弃用） |
| bus-track.c | 总线名称跟踪（监控客户端生命周期） |
| bus-match.c | 信号匹配规则 |
| bus-creds.c | 对端凭据（PID、UID、SELinux 标签等） |
| bus-slot.c | 回调槽位管理 |
| bus-error.c | 错误处理 |

### 2.2 关键 API

```c
// 连接与通信
sd_bus_open_system()       // 连接系统总线
sd_bus_open_user()         // 连接用户总线
sd_bus_call()              // 同步方法调用
sd_bus_call_async()        // 异步方法调用
sd_bus_send()              // 发送消息
sd_bus_process()           // 处理待处理消息

// 消息构造
sd_bus_message_new_method_call()   // 创建方法调用消息
sd_bus_message_append()            // 追加参数
sd_bus_message_read()              // 读取参数

// 对象注册（服务端）
sd_bus_add_object_vtable()   // 注册对象方法/属性/信号
sd_bus_add_match()           // 注册信号匹配回调

// 事件循环集成
sd_bus_attach_event()        // 将 bus 附加到 sd_event
```

### 2.3 vtable 模式

D-Bus 服务端使用声明式 vtable 导出接口：

```c
static const sd_bus_vtable manager_vtable[] = {
    SD_BUS_VTABLE_START(0),
    SD_BUS_METHOD("StartUnit", "ss", "o", method_start_unit, SD_BUS_VTABLE_UNPRIVILEGED),
    SD_BUS_METHOD("StopUnit", "ss", "o", method_stop_unit, SD_BUS_VTABLE_UNPRIVILEGED),
    SD_BUS_PROPERTY("Version", "s", property_get_version, 0, SD_BUS_VTABLE_PROPERTY_CONST),
    SD_BUS_SIGNAL("UnitNew", "so", 0),
    SD_BUS_VTABLE_END,
};
```

---

## 三、sd-event — 事件循环

### 3.1 实现文件

| 文件 | 职责 |
|------|------|
| sd-event.c | 核心事件循环实现 |
| event-util.c | 事件辅助工具 |

### 3.2 支持的事件源类型

| 事件源 | API | 底层机制 |
|--------|-----|---------|
| I/O | sd_event_add_io() | epoll |
| 定时器 | sd_event_add_time() | timerfd |
| 信号 | sd_event_add_signal() | signalfd |
| 子进程 | sd_event_add_child() | pidfd / SIGCHLD |
| defer | sd_event_add_defer() | 延迟执行 |
| post | sd_event_add_post() | 每轮循环后执行 |
| exit | sd_event_add_exit() | 退出时执行 |
| inotify | sd_event_add_inotify() | inotify |

### 3.3 事件循环生命周期

```c
sd_event *e;
sd_event_new(&e);

// 添加各种事件源
sd_event_add_io(e, NULL, fd, EPOLLIN, callback, userdata);
sd_event_add_time(e, NULL, CLOCK_MONOTONIC, usec, 0, timer_cb, data);
sd_event_add_signal(e, NULL, SIGTERM, signal_cb, data);

// 运行
sd_event_loop(e);     // 或 sd_event_run(e, timeout) 单步

// 清理
sd_event_unref(e);
```

### 3.4 优先级系统

事件源可设置优先级，值越小优先级越高：
- `SD_EVENT_PRIORITY_IMPORTANT` (-100)
- `SD_EVENT_PRIORITY_NORMAL` (0)
- `SD_EVENT_PRIORITY_IDLE` (100)

---

## 四、sd-journal — 日志 API

### 4.1 实现文件

| 文件 | 职责 |
|------|------|
| sd-journal.c | 日志读取 API |
| journal-file.c | 日志文件格式（二进制格式读写） |
| journal-send.c | 日志发送 API |
| journal-verify.c | 日志文件完整性校验 |
| journal-vacuum.c | 日志空间回收 |
| journal-authenticate.c | 日志 HMAC 认证 |

### 4.2 关键 API

```c
// === 写日志 ===
sd_journal_print(LOG_INFO, "Hello %s", "world");
sd_journal_send("MESSAGE=Hello", "PRIORITY=%d", LOG_INFO, NULL);

// === 读日志 ===
sd_journal *j;
sd_journal_open(&j, SD_JOURNAL_LOCAL_ONLY);
SD_JOURNAL_FOREACH(j) {
    const void *data;
    size_t length;
    sd_journal_get_data(j, "MESSAGE", &data, &length);
    printf("%.*s\n", (int)length, (const char*)data);
}
sd_journal_close(j);

// === 过滤 ===
sd_journal_add_match(j, "_SYSTEMD_UNIT=sshd.service", 0);
sd_journal_add_match(j, "PRIORITY=3", 0);  // 仅 ERR 级别

// === 实时跟踪 ===
sd_journal_seek_tail(j);
sd_journal_wait(j, (uint64_t) -1);  // 阻塞等待新条目
```

### 4.3 日志文件格式

日志使用自定义的二进制格式存储：
- **Header** — 文件元数据（版本、兼容性标志、密封标志）
- **Object** — 数据/字段/条目/哈希表/标签对象
- **Hash Table** — 数据对象和字段的哈希索引
- **Entry Array** — 条目的有序数组（支持二分查找）

---

## 五、sd-device — 设备模型

### 5.1 实现文件

| 文件 | 职责 |
|------|------|
| sd-device.c | 设备对象（从 sysfs 读取属性） |
| device-enumerator.c | 设备枚举器 |
| device-monitor.c | 设备事件监控（uevent） |
| device-private.c | 内部辅助函数 |

### 5.2 关键 API

```c
sd_device *dev;
sd_device_new_from_syspath(&dev, "/sys/class/net/eth0");

const char *val;
sd_device_get_property_value(dev, "ID_VENDOR", &val);
sd_device_get_sysattr_value(dev, "address", &val);

sd_device_unref(dev);
```

---

## 六、sd-daemon — 守护进程辅助

提供 socket 激活和服务通知的标准 API：

```c
// Socket 激活：获取传入的文件描述符
int n = sd_listen_fds(0);  // 返回 3+n 个 fd

// 服务通知
sd_notify(0, "READY=1");           // 通知就绪
sd_notify(0, "STATUS=Processing"); // 更新状态
sd_notify(0, "WATCHDOG=1");        // 看门狗心跳
sd_notify(0, "STOPPING=1");        // 通知即将停止
```

---

## 七、其他 API

### sd-netlink
封装 Linux netlink 通信，用于网络配置（路由、地址、链路等）：
- `sd-netlink.c` — 连接管理
- `netlink-message.c` — 消息构造
- `netlink-types.c` — 属性类型定义

### sd-login
查询登录会话信息：
- `sd_pid_get_session()` — 获取进程所属会话
- `sd_session_get_seat()` — 获取会话所属座位
- `sd_seat_get_active()` — 获取活跃会话

### sd-network
查询网络状态：
- `sd_network_get_operational_state()` — 网络总体状态
- `sd_network_link_get_operational_state()` — 链路状态

### sd-hwdb
访问硬件数据库（modalias → 属性映射）：
- `sd_hwdb_new()` / `sd_hwdb_get()` / `sd_hwdb_unref()`
