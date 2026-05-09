# D-Bus 通信深度分析 — sd-bus 实现与 PID 1 接口

> sd-bus 是 systemd 对 D-Bus 协议的原生实现，不依赖 libdbus。它直接操作 Unix
> 域套接字，实现 D-Bus 协议的认证、消息序列化、方法分发和信号匹配。PID 1 通过
> sd-bus 暴露 `org.freedesktop.systemd1` 接口，systemctl 等工具通过此接口与
> PID 1 通信。

---

## 一、sd-bus 核心数据结构

### 1.1 sd_bus — 总线连接

`bus-internal.h:183-310`：

```c
struct sd_bus {
    unsigned n_ref;            // 引用计数
    enum bus_state state;      // 连接状态机

    int input_fd, output_fd;   // 套接字 FD（通常同一个）
    int inotify_fd;            // watch_bind 模式的 inotify
    int message_version;       // D-Bus 协议版本
    int message_endian;        // 字节序

    /* ---- 运行标志（位域） ---- */
    bool can_fds:1;            // 支持 FD 传递
    bool bus_client:1;         // 客户端模式（需要 Hello）
    bool ucred_valid:1;        // ucred 已获取
    bool is_server:1;          // 服务端模式
    bool anonymous_auth:1;     // 允许匿名认证
    bool trusted:1;            // 信任的连接（跳过权限检查）
    bool is_system:1;          // 系统总线
    bool is_user:1;            // 用户总线
    bool is_monitor:1;         // 监控模式
    bool watch_bind:1;         // 等待 socket 出现
    bool accept_fd:1;          // 接受 FD 传递
    bool exit_on_disconnect:1; // 断开时退出事件循环

    /* ---- 读写缓冲 ---- */
    void *rbuffer;             // 原始读缓冲区
    size_t rbuffer_size;

    sd_bus_message **rqueue;   // 已解析的接收消息队列
    size_t rqueue_size;

    sd_bus_message **wqueue;   // 待发送消息队列
    size_t wqueue_size;
    size_t windex;             // 当前写位置

    /* ---- 事务与标识 ---- */
    uint64_t cookie;           // 消息序号（递增）
    uint64_t read_counter;     // 接收消息计数
    char *unique_name;         // 分配的唯一名称（:1.xxx）
    uint64_t unique_id;        // 唯一 ID

    /* ---- 回调注册 ---- */
    struct bus_match_node match_callbacks;  // 信号匹配树
    Prioq *reply_callbacks_prioq;          // 异步回复超时队列
    OrderedHashmap *reply_callbacks;       // 异步回复回调（按 cookie）
    LIST_HEAD(struct filter_callback, filter_callbacks);  // 过滤器链

    /* ---- 对象树 ---- */
    Hashmap *nodes;            // 对象路径 → node 映射
    Hashmap *vtable_methods;   // 方法快速查找
    Hashmap *vtable_properties;// 属性快速查找

    /* ---- 认证 ---- */
    enum bus_auth auth;        // EXTERNAL 或 ANONYMOUS
    struct iovec auth_iovec[3];// 认证 IO 向量
    usec_t auth_timeout;       // 认证超时
    struct ucred ucred;        // 对端凭据（pid, uid, gid）
    char *label;               // SELinux 标签
};
```

### 1.2 连接状态机

`bus-internal.h:161-171`：

```
BUS_UNSET ──> BUS_WATCH_BIND ──> BUS_OPENING ──> BUS_AUTHENTICATING
                                      │                │
                                      │         SASL 握手完成
                                      │                │
                                      │                ▼
                                      │          BUS_HELLO
                                      │                │
                                      │        Hello 回复到达
                                      │                │
                                      └────────> BUS_RUNNING ──> BUS_CLOSING ──> BUS_CLOSED
```

| 状态 | 含义 |
|------|------|
| `BUS_UNSET` | 初始状态 |
| `BUS_WATCH_BIND` | inotify 等待 socket 文件出现 |
| `BUS_OPENING` | connect() 非阻塞进行中 |
| `BUS_AUTHENTICATING` | SASL 认证阶段 |
| `BUS_HELLO` | 等待 org.freedesktop.DBus.Hello 回复 |
| `BUS_RUNNING` | 正常运行 |
| `BUS_CLOSING` | 正在关闭 |
| `BUS_CLOSED` | 已关闭 |

### 1.3 sd_bus_message — 消息对象

`bus-message.h:50-132`：

```c
struct sd_bus_message {
    unsigned n_ref;            // 引用计数
    unsigned n_queued;         // 在队列中的引用
    sd_bus *bus;               // 所属总线

    /* 消息头部字段 */
    uint8_t header[...];       // 固定头部
    char *path;                // 对象路径
    char *interface;           // 接口名
    char *member;              // 方法/信号名
    char *destination;         // 目标名
    char *sender;              // 发送者名

    sd_bus_error error;        // 错误信息
    sd_bus_creds creds;        // 发送者凭据

    /* 时间戳 */
    usec_t monotonic, realtime;
    uint64_t seqnum;           // 内核分配的序号

    /* 负载 */
    size_t fields_size;        // 头部字段大小
    size_t body_size;          // 消息体大小
    struct bus_body_part *body;// 消息体链表

    int *fds;                  // 附带的文件描述符
    size_t n_fds;

    uint64_t reply_cookie;     // 回复对应的请求 cookie
    usec_t timeout;            // 调用超时
};
```

### 1.4 sd_bus_slot — 回调槽位

`bus-internal.h:128-159`：

```c
struct sd_bus_slot {
    unsigned n_ref;
    sd_bus *bus;
    void *userdata;
    sd_bus_destroy_t destroy_callback;
    enum bus_slot_type type;
    bool floating:1;           // 浮动引用
    bool match_added:1;        // 已向 daemon 注册

    union {
        struct reply_callback reply;   // 异步回复回调
        struct filter_callback filter; // 过滤器回调
        struct match_callback match;   // 信号匹配回调
        struct node_callback node;     // 对象节点回调
        struct node_vtable vtable;     // vtable 注册
        // ... 更多类型
    };
};
```

---

## 二、连接生命周期

### 2.1 创建与连接

```
sd_bus_open_system()
  │
  ├── sd_bus_new()                     [sd-bus.c:231]
  │     → 分配 sd_bus 结构
  │     → state = BUS_UNSET
  │     → accept_fd = true
  │
  ├── 设置地址
  │     → /run/dbus/system_bus_socket  (system bus)
  │     → $XDG_RUNTIME_DIR/bus         (user bus)
  │     → bus_client = true, trusted = false
  │
  └── sd_bus_start()                   [sd-bus.c:1188]
        │
        ├── bus_start_address()
        │     → socket(AF_UNIX) + connect()
        │     → state = BUS_OPENING 或 BUS_AUTHENTICATING
        │
        ├── SASL 认证                   [bus-socket.c:376]
        │     ├── 客户端发送: "\0AUTH EXTERNAL <uid_hex>\r\n"
        │     ├── 服务端发送: "OK <server_id>\r\n"
        │     ├── FD 协商:    "NEGOTIATE_UNIX_FD\r\n"
        │     └── 开始:       "BEGIN\r\n"
        │     → state = BUS_AUTHENTICATING → BUS_HELLO
        │
        └── bus_send_hello()            [sd-bus.c:576]
              → 发送 Hello 方法调用
              → hello_callback() 解析唯一名称
              → state = BUS_RUNNING
```

### 2.2 SASL 认证细节

`bus-socket.c:298-512`：

```
客户端认证流程（EXTERNAL）：
  1. 发送 NUL 字节（Unix 域套接字身份标识）
  2. 发送 "AUTH EXTERNAL <uid_hex>"
  3. 服务端通过 SO_PEERCRED 获取 ucred
  4. verify_external_token() 验证 UID 匹配
  5. 协商 NEGOTIATE_UNIX_FD（FD 传递能力）
  6. 客户端发送 "BEGIN"，进入二进制协议

匿名认证：
  → "AUTH ANONYMOUS" — 不验证身份
  → anonymous_auth 标志控制是否允许
```

---

## 三、消息收发机制

### 3.1 发送消息

```
sd_bus_send(bus, message, cookie)
  │
  ├── bus_seal_message()
  │     → 分配 cookie（递增）
  │     → bus_message_setup_iovec()
  │     → 设置消息字节序/版本
  │
  ├── 如果 wqueue 为空且直接可写：
  │     → bus_write_message() → writev()
  │     → 写完 → 完成
  │     → 部分写 → 放入 wqueue
  │
  └── 否则放入 wqueue 尾部
```

### 3.2 同步调用

`sd_bus_call()`（`sd-bus.c:2343-2455`）：

```
sd_bus_call(bus, msg, timeout, &error, &reply)
  │
  ├── 发送消息（同 sd_bus_send）
  │
  ├── 阻塞等待回复：
  │     loop:
  │       扫描 rqueue 查找 reply_cookie 匹配的回复
  │       如果找到 → 返回
  │       sd_bus_process() → 处理到达的消息
  │       sd_bus_wait() → epoll_wait 等待新数据
  │
  └── 超时 → 返回 -ETIMEDOUT
```

### 3.3 异步调用

```
sd_bus_call_async(bus, &slot, msg, callback, userdata, timeout)
  │
  ├── 创建 reply_callback slot
  │     → cookie → callback 映射
  │     → 放入 reply_callbacks (OrderedHashmap)
  │     → 放入 reply_callbacks_prioq（超时排序）
  │
  ├── 发送消息
  │
  └── 回复到达时：
        process_message() → process_reply()
        → 从 reply_callbacks 取出 callback
        → 调用 callback(msg, userdata, &error)
```

### 3.4 消息分发流程

`process_running()`（`sd-bus.c:2980-3041`）：

```
sd_bus_process(bus, &msg)
  │
  ├── [1] process_timeout()
  │     → 检查 reply_callbacks_prioq 中超时的异步调用
  │     → 超时的调用触发 -ETIMEDOUT 回调
  │
  ├── [2] dispatch_wqueue()
  │     → 尝试发送 wqueue 中积压的消息
  │
  ├── [3] dispatch_track()
  │     → 追踪的对象路径变更处理
  │
  ├── [4] dispatch_rqueue()
  │     → 从 rbuffer 读取并解析新消息到 rqueue
  │
  └── [5] process_message(msg)
        │
        ├── [a] process_reply()
        │     → 匹配 reply_cookie → 调用异步回调
        │
        ├── [b] process_filter()
        │     → 遍历 filter_callbacks 链表
        │     → 每个过滤器都能看到所有消息
        │
        ├── [c] process_match()
        │     → 在 match_callbacks 匹配树中查找
        │     → 匹配信号/广播消息
        │
        ├── [d] process_builtin()
        │     → 处理 D-Bus 标准接口
        │     → Introspectable, Peer, Properties
        │
        └── [e] bus_process_object()
              → 在 nodes 树中查找目标路径
              → 匹配 vtable 方法/属性
              → 调用注册的处理函数
```

### 3.5 读写队列

```
                   应用层
                     │
        ┌────────────┼────────────┐
        │            │            │
   sd_bus_send()  sd_bus_call()  sd_bus_call_async()
        │            │            │
        └────────────┼────────────┘
                     │
              ┌──────┴──────┐
              │   wqueue    │  → 待发送消息队列
              │  (出站方向)  │  → dispatch_wqueue() → writev()
              └──────┬──────┘
                     │ Unix 域套接字
              ┌──────┴──────┐
              │   rqueue    │  → 已解析接收消息队列
              │  (入站方向)  │  ← bus_read_message() ← readv()
              └──────┬──────┘
                     │
              process_message()
```

---

## 四、vtable 机制 — 对象方法注册

### 4.1 注册接口

```c
// sd_bus_add_object_vtable() → bus-objects.c:1990

// vtable 定义示例（PID 1 Manager）：
const sd_bus_vtable bus_manager_vtable[] = {
    SD_BUS_VTABLE_START(0),

    // 只读属性
    SD_BUS_PROPERTY("Version", "s", getter, 0, CONST),

    // 可写属性
    SD_BUS_WRITABLE_PROPERTY("LogLevel", "s", getter, setter, 0, 0),

    // 方法
    SD_BUS_METHOD("StartUnit", "ss", "o", handler, UNPRIVILEGED),

    // 信号
    SD_BUS_SIGNAL("UnitNew", "so", 0),

    SD_BUS_VTABLE_END
};
```

### 4.2 内部存储

```
注册流程：
  sd_bus_add_object_vtable(bus, &slot, path, interface, vtable, userdata)
    │
    ├── 在 bus->nodes 中查找/创建 path 对应的 node
    │
    ├── 遍历 vtable 中的每个条目：
    │     ├── SD_BUS_VTABLE_METHOD → 创建 vtable_member
    │     │     → 放入 bus->vtable_methods (Hashmap)
    │     │     → key = {path, interface, member}
    │     │
    │     └── SD_BUS_VTABLE_PROPERTY → 创建 vtable_member
    │           → 放入 bus->vtable_properties
    │
    └── vtable_member 结构：
          { path, interface, member, vtable_entry, node }
```

### 4.3 方法调用分发

```
收到方法调用消息 (type=METHOD_CALL)
  │
  ├── bus_process_object(bus, msg)
  │     → 在 vtable_methods 中查找
  │       key = {msg->path, msg->interface, msg->member}
  │
  ├── 访问控制检查  [bus-objects.c:302-320]
  │     ├── bus->trusted → 跳过检查
  │     ├── VTABLE_UNPRIVILEGED → 允许非特权调用
  │     └── 否则 → 检查 capability/polkit
  │
  └── 调用 vtable 中注册的 handler(msg, userdata, &error)
        → 处理函数构造回复消息
        → sd_bus_reply_method_return/error()
```

---

## 五、sd-event 集成

### 5.1 附加到事件循环

`sd_bus_attach_event()`（`sd-bus.c:3782-3835`）：

```
sd_bus_attach_event(bus, event, priority)
  │
  ├── 注册 IO 事件源
  │     → sd_event_add_io(event, &source, bus_fd, EPOLLIN, io_callback, bus)
  │     → 当 bus FD 可读时触发
  │
  ├── 注册定时器事件源
  │     → sd_event_add_time(event, &source, CLOCK_MONOTONIC,
  │                         timeout, 0, time_callback, bus)
  │     → 处理异步调用超时
  │
  ├── 注册退出事件源
  │     → sd_event_add_exit(event, &source, quit_callback, bus)
  │
  ├── 注册 prepare 回调
  │     → sd_event_source_set_prepare(io_source, prepare_callback)
  │
  └── 注册 inotify 事件源（watch_bind 模式）
        → bus_attach_inotify_event()
```

### 5.2 prepare/io/time 回调

```
prepare_callback(source, userdata)         [sd-bus.c:3627-3677]
  │
  ├── 检查 bus 是否有数据要发送
  │     → 有 → POLLIN | POLLOUT
  │     → 无 → POLLIN
  │
  └── 更新定时器
        → sd_event_source_set_time(time_source, next_timeout)

io_callback(source, fd, revents, userdata)  [sd-bus.c:3595-3610]
  → sd_bus_process(bus, NULL)
  → 失败 → bus_enter_closing(bus)

time_callback(source, usec, userdata)       [sd-bus.c:3612-3625]
  → sd_bus_process(bus, NULL)
  → 处理超时的异步调用
```

### 5.3 事件驱动模型图

```
                    sd_event_loop()
                         │
                    sd_event_run()
                         │
              ┌──────────┼──────────┐
              │          │          │
         prepare()    epoll_wait  dispatch
              │          │          │
              ▼          ▼          ▼
        更新 POLLOUT   bus FD    io_callback
        更新超时       可读       │
                                  ▼
                          sd_bus_process()
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
              process_reply   process_match  bus_process_object
              (异步回复)      (信号匹配)     (方法调用)
```

---

## 六、PID 1 的 D-Bus 接口

### 6.1 连接初始化

`src/core/dbus.c:619-847`：

```
bus_init_api(manager)
  │
  ├── sd_bus_open_system() 或 sd_bus_open_user()
  │
  ├── sd_bus_attach_event(bus, manager->event, ...)
  │
  ├── bus_setup_api_vtables(manager, bus)
  │     │
  │     ├── 安装 SELinux 过滤器
  │     │     → sd_bus_add_filter(bus, mac_selinux_filter)
  │     │
  │     └── 注册所有 vtable：
  │           ├── bus_manager_vtable        → /org/freedesktop/systemd1
  │           ├── bus_unit_vtable           → /org/freedesktop/systemd1/unit/*
  │           ├── bus_service_vtable        → (Service 类型扩展)
  │           ├── bus_socket_vtable         → (Socket 类型扩展)
  │           ├── bus_timer_vtable          → (Timer 类型扩展)
  │           ├── bus_mount_vtable          → (Mount 类型扩展)
  │           ├── bus_scope_vtable          → (Scope 类型扩展)
  │           ├── bus_slice_vtable          → (Slice 类型扩展)
  │           ├── bus_path_vtable           → (Path 类型扩展)
  │           ├── bus_swap_vtable           → (Swap 类型扩展)
  │           ├── bus_automount_vtable      → (Automount 类型扩展)
  │           ├── bus_target_vtable         → (Target 类型扩展)
  │           └── bus_job_vtable            → /org/freedesktop/systemd1/job/*
  │
  └── sd_bus_request_name_async("org.freedesktop.systemd1", ...)
```

### 6.2 Manager 接口 — org.freedesktop.systemd1.Manager

**属性**（`dbus-manager.c:2739-2841`）：

| 属性 | 类型 | 说明 |
|------|------|------|
| Version | s | systemd 版本号 |
| Features | s | 编译特性列表 |
| Virtualization | s | 虚拟化类型 |
| Architecture | s | CPU 架构 |
| Tainted | s | 污染标记 |
| *Timestamp | (tt) | 多种启动时间戳（Firmware/Kernel/InitRD/...） |
| LogLevel | s | 日志级别（可写） |
| LogTarget | s | 日志目标（可写） |
| NNames | u | 已加载 unit 数量 |
| NFailedUnits | u | 失败 unit 数量 |
| NJobs | u | 活跃 job 数量 |
| Progress | d | 启动进度（0.0-1.0） |
| Environment | as | 全局环境变量 |
| SystemState | s | 系统状态（initializing/starting/running/...） |
| ControlGroup | s | 根 cgroup 路径 |
| Default* | 各种 | 默认超时/限制/策略等 |

**Unit 生命周期方法**：

| 方法 | 签名 | 说明 |
|------|------|------|
| GetUnit | s→o | 按名称获取 unit 对象路径 |
| GetUnitByPID | u→o | 按 PID 获取 unit |
| GetUnitByInvocationID | ay→o | 按调用 ID 获取 unit |
| LoadUnit | s→o | 加载 unit（如未加载） |
| StartUnit | ss→o | 启动 unit（返回 job） |
| StopUnit | ss→o | 停止 unit |
| RestartUnit | ss→o | 重启 unit |
| ReloadUnit | ss→o | 重载 unit 配置 |
| TryRestartUnit | ss→o | 仅重启正在运行的 unit |
| ReloadOrRestartUnit | ss→o | 重载或重启 |
| KillUnit | ssi→ | 向 unit 发信号 |
| ResetFailedUnit | s→ | 重置失败状态 |
| FreezeUnit | s→ | 冻结 unit（cgroup freezer） |
| ThawUnit | s→ | 解冻 unit |

**系统管理方法**：

| 方法 | 说明 |
|------|------|
| Reload | 重新加载所有 unit 文件 |
| Reexecute | PID 1 重新执行自身 |
| PowerOff | 关机 |
| Reboot | 重启 |
| Halt | 停机 |
| SetEnvironment | 设置全局环境变量 |
| UnsetEnvironment | 删除全局环境变量 |

**Unit 文件管理方法**：

| 方法 | 说明 |
|------|------|
| ListUnitFiles | 列出所有 unit 文件及状态 |
| GetUnitFileState | 获取 unit 文件启用状态 |
| EnableUnitFiles | 启用 unit 文件（创建符号链接） |
| DisableUnitFiles | 禁用 unit 文件 |
| PresetUnitFiles | 按预设策略启用/禁用 |
| MaskUnitFiles | 屏蔽 unit 文件 |
| UnmaskUnitFiles | 取消屏蔽 |

**信号**：

| 信号 | 参数 | 触发时机 |
|------|------|----------|
| UnitNew | so | 新 unit 加载 |
| UnitRemoved | so | unit 卸载 |
| JobNew | uos | 新 job 创建 |
| JobRemoved | uoss | job 完成/取消 |
| StartupFinished | tttttt | 启动完成（含各阶段耗时） |
| UnitFilesChanged | — | unit 文件变更 |
| Reloading | b | 开始/结束重载 |

### 6.3 Unit 接口 — org.freedesktop.systemd1.Unit

`dbus-unit.c:860-1620`：

**关键属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| Id | s | unit 名称 |
| Names | as | 所有名称（含别名） |
| ActiveState | s | 统一状态（active/inactive/...） |
| SubState | s | 子系统特定状态 |
| LoadState | s | 加载状态（loaded/not-found/error） |
| UnitFileState | s | 文件状态（enabled/disabled/...） |
| Description | s | 描述文本 |
| Requires | as | 依赖列表 |
| Wants | as | 弱依赖列表 |
| After | as | 顺序依赖 |
| Before | as | 顺序依赖 |
| CanStart | b | 是否可启动 |
| CanStop | b | 是否可停止 |
| CanReload | b | 是否可重载 |
| Job | (uo) | 当前活跃 job |
| InvocationID | ay | 16字节调用标识 |

**方法**：

```
Start(mode) → job_path
Stop(mode) → job_path
Reload(mode) → job_path
Restart(mode) → job_path
Kill(who, signal)
ResetFailed()
SetProperties(runtime, properties)
Freeze() / Thaw()
```

`mode` 参数：`"replace"`, `"fail"`, `"isolate"`, `"ignore-dependencies"`, `"ignore-requirements"`

---

## 七、systemctl 与 PID 1 的通信流程

### 7.1 启动服务示例

```
systemctl start nginx.service
  │
  ├── sd_bus_open_system()
  │     → 连接到 /run/dbus/system_bus_socket
  │     → SASL EXTERNAL 认证
  │     → Hello 获取唯一名称
  │
  ├── sd_bus_call(bus, "StartUnit",
  │               "nginx.service", "replace")
  │     → 目标: org.freedesktop.systemd1
  │     → 路径: /org/freedesktop/systemd1
  │     → 接口: org.freedesktop.systemd1.Manager
  │     → 方法: StartUnit
  │
  ├── PID 1 处理：
  │     bus_process_object()
  │       → bus_manager_vtable 查找 "StartUnit"
  │       → method_start_unit()
  │         → manager_load_unit()
  │         → manager_add_job(JOB_START, unit, mode, &error, &job)
  │         → 返回 job 对象路径
  │
  ├── systemctl 收到回复：
  │     → job 对象路径 "/org/freedesktop/systemd1/job/42"
  │
  └── 等待 job 完成：
        → 订阅 JobRemoved 信号
        → sd_bus_add_match(bus, "signal,member=JobRemoved")
        → 收到信号 → 检查 job ID 和结果
        → 返回成功/失败给用户
```

### 7.2 查询状态示例

```
systemctl status nginx.service
  │
  ├── GetUnit("nginx.service") → unit_path
  │
  ├── 批量读取属性：
  │     GetAll(unit_path, "org.freedesktop.systemd1.Unit")
  │     GetAll(unit_path, "org.freedesktop.systemd1.Service")
  │
  └── 解析并显示：
        ActiveState, SubState, LoadState, MainPID,
        ExecMainStartTimestamp, MemoryCurrent, etc.
```

---

## 八、凭据与访问控制

### 8.1 对端凭据获取

```c
// SO_PEERCRED — 在 Unix 域套接字连接时由内核设置
// 包含: pid, uid, gid

// sd-bus 中获取发送者凭据：
sd_bus_query_sender_creds(call, SD_BUS_CREDS_PID|SD_BUS_CREDS_EUID|
                          SD_BUS_CREDS_EFFECTIVE_CAPS|
                          SD_BUS_CREDS_SELINUX_CONTEXT, &creds);
```

### 8.2 PID 1 的访问控制层次

```
收到方法调用
  │
  ├── [1] SELinux 过滤器          [dbus.c:213-274]
  │     → mac_selinux_filter()
  │     → 检查 SELinux 策略是否允许此 D-Bus 调用
  │
  ├── [2] vtable 级别检查         [bus-objects.c:302-320]
  │     ├── bus->trusted → 跳过（PID 1 内部调用）
  │     ├── SD_BUS_VTABLE_UNPRIVILEGED → 允许（如 GetUnit）
  │     └── 否则 → 需要特权
  │
  └── [3] 方法实现级别检查
        → 具体方法内部验证
        → bus_verify_manage_units_async() — 检查 polkit 授权
        → 如：StartUnit 需要 org.freedesktop.systemd1.manage-units
```

### 8.3 Polkit 集成

```
特权操作 → bus_verify_*_async()
  │
  ├── 调用者是 root (uid==0) → 直接允许
  │
  └── 否则 → 查询 polkit
        → sd_bus_call_async(polkit_bus,
              "org.freedesktop.PolicyKit1.Authority",
              "CheckAuthorization", ...)
        → polkit 检查用户是否有权限
        → 可能弹出图形化认证对话框
```

---

## 九、私有总线（Direct Connection）

PID 1 除了连接系统 D-Bus daemon，还可以通过私有套接字直接通信：

```
/run/systemd/private
  → PID 1 监听此 Unix 域套接字
  → systemctl --no-ask-password 直接连接（不走 dbus-daemon）
  → 更快、更可靠（不依赖 dbus-daemon 运行）
  → 用于早期启动和紧急恢复场景
```

---

## 十、D-Bus 消息线格式

```
┌──────────────────────────────────────────────────┐
│ 固定头部 (16 bytes)                               │
│  byte_order │ type │ flags │ version              │
│  body_length │ serial                             │
├──────────────────────────────────────────────────┤
│ 头部字段（变长，8字节对齐）                         │
│  PATH, INTERFACE, MEMBER, ERROR_NAME,             │
│  REPLY_SERIAL, DESTINATION, SENDER, SIGNATURE     │
│  UNIX_FDS                                         │
├──────────────────────────────────────────────────┤
│ 消息体（按 SIGNATURE 描述的类型序列化）             │
│  基本类型: y(byte) b(bool) n(int16) q(uint16)    │
│           i(int32) u(uint32) x(int64) t(uint64)  │
│           d(double) s(string) o(object_path)      │
│           g(signature) h(unix_fd)                 │
│  容器类型: a(array) (struct) v(variant)            │
│           {dict_entry}                            │
└──────────────────────────────────────────────────┘
```

**sd-bus 优化**：
- **memfd**：大消息体通过 memfd 传递（零拷贝）
- **FD 传递**：通过 `SCM_RIGHTS` 辅助消息传递文件描述符
- **凭据传递**：通过 `SCM_CREDENTIALS` 获取对端 pid/uid/gid
