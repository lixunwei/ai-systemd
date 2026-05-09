# Socket Activation 机制深度分析

> Socket Activation 是 systemd 最重要的设计创新之一：由 PID 1 代管监听套接字，
> 在有连接到来时才按需启动服务进程，实现零延迟的并行启动和按需加载。

---

## 一、核心设计理念

传统模式：
```
服务自行 bind()+listen()
  → 服务启动慢时客户端连接被拒绝
  → 服务间串行依赖（必须等 A 监听后 B 才能连接）
```

Socket Activation 模式：
```
PID 1 提前 bind()+listen() 所有 socket
  → 内核缓冲区暂存连接
  → 服务启动后接管 FD
  → 所有服务可并行启动
  → 未使用的服务零开销
```

---

## 二、数据结构

### 2.1 SocketPort — 单个监听端口

每个 `.socket` 单元可包含多个 `ListenXxx=` 指令，每个对应一个 `SocketPort`（`socket.h:45-58`）：

```c
typedef struct SocketPort {
    Socket *socket;              // 所属 Socket 单元
    SocketType type;             // 类型（见下文）
    int fd;                      // 打开的文件描述符（-1 表示未打开）
    int *auxiliary_fds;          // USB 辅助 FD（USB Function 专用）
    size_t n_auxiliary_fds;
    SocketAddress address;       // 套接字地址（sockaddr 封装）
    char *path;                  // FIFO/Special/MQueue 路径
    sd_event_source *event_source; // epoll 事件源
    LIST_FIELDS(struct SocketPort, port);  // 链表节点
} SocketPort;
```

### 2.2 SocketType — 支持的 IPC 类型

```c
typedef enum SocketType {
    SOCKET_SOCKET,        // 网络/Unix 套接字（ListenStream/Datagram/SequentialPacket）
    SOCKET_FIFO,          // 命名管道（ListenFIFO=）
    SOCKET_SPECIAL,       // 特殊文件（ListenSpecial=）
    SOCKET_MQUEUE,        // POSIX 消息队列（ListenMessageQueue=）
    SOCKET_USB_FUNCTION,  // USB 功能端点（ListenUSBFunction=）
} SocketType;
```

### 2.3 Socket 结构体

Socket 单元的完整结构（`socket.h:68-162`），重要字段：

```c
struct Socket {
    Unit meta;                         // Unit 基类

    LIST_HEAD(SocketPort, ports);      // 所有监听端口链表
    Set *peers_by_address;             // 按地址索引的对端信息

    /* 连接计数 */
    unsigned n_accepted;               // 累计接受连接数
    unsigned n_connections;            // 当前活跃连接数
    unsigned n_refused;                // 累计拒绝连接数
    unsigned max_connections;          // 最大连接数（默认 64）
    unsigned max_connections_per_source; // 单一源最大连接数

    unsigned backlog;                  // listen() backlog（默认 SOMAXCONN）
    usec_t timeout_usec;               // 操作超时

    /* 执行环境 */
    ExecCommand* exec_command[_SOCKET_EXEC_COMMAND_MAX]; // 5 个钩子
    ExecContext exec_context;          // 执行上下文
    KillContext kill_context;          // 信号配置
    CGroupContext cgroup_context;      // cgroup 配置

    /* 关联服务 */
    UnitRef service;                   // Accept=no: 触发的服务
                                       // Accept=yes: 下一个实例化的服务模板

    SocketState state;                 // 当前状态
    bool accept;                       // Accept= 模式（是否每连接一个服务实例）

    /* Socket 选项 */
    bool keep_alive, no_delay, free_bind, transparent;
    bool broadcast, pass_cred, pass_sec, pass_pktinfo;
    bool reuse_port, remove_on_stop, writable, flush_pending;

    char *fdname;                      // FD 名称（LISTEN_FDNAMES）
    RateLimit trigger_limit;           // 触发频率限制
};
```

---

## 三、状态机

### 3.1 状态定义与映射

`socket.c:58-73` 定义了 14 个状态到 UnitActiveState 的映射：

```
SocketState             │ UnitActiveState │ 含义
────────────────────────┼─────────────────┼────────────────────
SOCKET_DEAD             │ INACTIVE        │ 初始/终止状态
SOCKET_START_PRE        │ ACTIVATING      │ 执行 ExecStartPre=
SOCKET_START_CHOWN      │ ACTIVATING      │ 更改 socket 所有者
SOCKET_START_POST       │ ACTIVATING      │ 执行 ExecStartPost=
SOCKET_LISTENING        │ ACTIVE          │ 正在监听（核心状态）
SOCKET_RUNNING          │ ACTIVE          │ 服务已启动运行中
SOCKET_STOP_PRE         │ DEACTIVATING    │ 执行 ExecStopPre=
SOCKET_STOP_PRE_SIGTERM │ DEACTIVATING    │ 发送 SIGTERM 给钩子
SOCKET_STOP_PRE_SIGKILL │ DEACTIVATING    │ 发送 SIGKILL 给钩子
SOCKET_STOP_POST        │ DEACTIVATING    │ 执行 ExecStopPost=
SOCKET_FINAL_SIGTERM    │ DEACTIVATING    │ 最终清理 SIGTERM
SOCKET_FINAL_SIGKILL    │ DEACTIVATING    │ 最终清理 SIGKILL
SOCKET_FAILED           │ FAILED          │ 失败状态
SOCKET_CLEANING         │ MAINTENANCE     │ 清理状态
```

### 3.2 状态转换图

```
                    socket_start()
 DEAD ─────────────────────────────────> START_PRE
  ▲                                        │
  │                                        │ ExecStartPre= 完成
  │                                        ▼
  │                                    START_CHOWN ──(有 User=/Group=)──> chown 完成
  │                                        │                                  │
  │                                  (无 User=/Group=)                        │
  │                                        │                                  │
  │                                        ▼ <────────────────────────────────┘
  │                                    START_POST
  │                                        │
  │                                        │ ExecStartPost= 完成
  │                                        ▼
  │                                    LISTENING ◄──────────────┐
  │                                        │                    │
  │                           socket_dispatch_io()              │
  │                           (EPOLLIN 事件到达)                │
  │                                        │                    │
  │                                        ▼                    │
  │                                    RUNNING                  │
  │                                        │                    │
  │                               服务完成，连接关闭             │
  │                                        └────────────────────┘
  │
  │    socket_stop()
  ├────────────────── STOP_PRE
  │                     │
  │                     ▼
  │              STOP_PRE_SIGTERM ──> STOP_PRE_SIGKILL
  │                     │                    │
  │                     ▼                    ▼
  │                 STOP_POST ◄──────────────┘
  │                     │
  │                     ▼
  │              FINAL_SIGTERM ──> FINAL_SIGKILL
  │                     │                │
  └─────────────────────┴────────────────┘

  任何阶段失败 ──> FAILED
```

---

## 四、启动流程详解

### 4.1 socket_start() — 入口

`socket.c:2472-2524`：

```
socket_start(unit)
  │
  ├── 检查状态：正在停止中 → 返回 -EAGAIN（稍后重试）
  ├── 检查状态：正在启动中 → 返回 0（已在进行）
  ├── 检查关联服务：必须已加载且处于 dead/failed/auto_restart
  │
  └── socket_enter_start_pre(s)
```

### 4.2 socket_enter_start_pre() — 执行 ExecStartPre=

`socket.c:2268-2294`：

```
socket_enter_start_pre()
  │
  ├── 有 ExecStartPre= 命令
  │     → socket_spawn() 执行命令
  │     → 设置状态 START_PRE
  │     → 等待 SIGCHLD
  │
  └── 无 ExecStartPre= 命令
        → 直接进入 socket_enter_start_chown()
```

### 4.3 socket_enter_start_chown() — 打开 FD + 更改权限

`socket.c:2235-2266`：

```
socket_enter_start_chown()
  │
  ├── socket_open_fds(s)          ← 关键！打开所有监听端口
  │     遍历 s->ports 链表：
  │     ├── SOCKET_SOCKET  → socket_address_listen_in_cgroup()
  │     │     → socket()/bind()/listen()
  │     │     → 应用 socket 选项（keep_alive, no_delay 等）
  │     │     → 创建 symlinks
  │     ├── SOCKET_FIFO    → fifo_address_create()
  │     │     → mkfifo()/open()
  │     ├── SOCKET_SPECIAL → special_address_create()
  │     │     → open() 特殊文件
  │     ├── SOCKET_MQUEUE  → mq_address_create()
  │     │     → mq_open()
  │     └── SOCKET_USB_FUNCTION → usbffs_address_create()
  │
  ├── 有 User=/Group= 配置
  │     → fork() 子进程执行 chown()
  │     → 设置状态 START_CHOWN
  │
  └── 无 User=/Group=
        → 直接进入 socket_enter_start_post()
```

### 4.4 socket_enter_listening() — 激活监听

`socket.c:2188-2208`：

```
socket_enter_listening()
  │
  ├── flush_pending=yes 且 Accept=no
  │     → flush_ports(s)  清空缓冲区中已有数据
  │
  ├── socket_watch_fds(s)
  │     遍历所有端口：
  │     └── sd_event_add_io(event, &p->event_source,
  │                         p->fd, EPOLLIN,
  │                         socket_dispatch_io, p)
  │         ↑ 注册 epoll 监听，回调函数 socket_dispatch_io
  │
  └── socket_set_state(s, SOCKET_LISTENING)
        → Socket 单元变为 ACTIVE
```

---

## 五、连接到来时的触发流程

### 5.1 socket_dispatch_io() — epoll 回调

`socket.c:3005-3044`：

```
socket_dispatch_io(source, fd, revents, userdata)
  │
  ├── 检查状态 ≠ LISTENING → 忽略
  │
  ├── revents ≠ EPOLLIN → 错误处理
  │
  ├── Accept=yes 且类型是 SOCKET_SOCKET
  │     → cfd = socket_accept_in_cgroup(s, p, fd)
  │     → socket_apply_socket_options(s, p, cfd)
  │     → socket_enter_running(s, cfd)  ← 传入已接受的连接 FD
  │
  └── Accept=no 或非 socket 类型
        → socket_enter_running(s, -1)   ← 传入 -1，无连接 FD
```

### 5.2 socket_enter_running() — 启动服务

`socket.c:2311-2442`，分两种模式：

#### 模式 A：Accept=no（单例服务，默认模式）

```
socket_enter_running(s, cfd=-1)
  │
  ├── 检查停止挂起 → 拒绝并刷空端口
  ├── 检查触发频率限制 → 超限则进入 stop_pre
  │
  ├── 检查关联服务是否已运行/启动中
  │     → 已活跃则跳过（避免重复启动）
  │
  ├── manager_add_job(JOB_START, s->service, JOB_REPLACE)
  │     → 向事务引擎提交启动服务的 Job
  │
  └── socket_set_state(s, SOCKET_RUNNING)
```

**FD 传递**：单例模式下，FD 不在触发时传递。服务启动时通过
`service_collect_fds()` 从关联的 Socket 单元收集所有监听 FD。

#### 模式 B：Accept=yes（每连接实例化）

```
socket_enter_running(s, cfd=已接受的连接FD)
  │
  ├── 连接数检查
  │     ├── n_connections >= max_connections → 拒绝
  │     └── 同源连接 > max_connections_per_source → 拒绝
  │
  ├── socket_load_service_unit(s, cfd, &service)
  │     → 从服务模板实例化：foo@<连接标识>.service
  │
  ├── service_set_socket_fd(SERVICE(service), cfd, s, peer)
  │     → 将连接 FD 保存到 Service.socket_fd
  │     → 记录对端地址到 Service.socket_peer
  │     → 设置 Unit 描述为 "服务名 (对端地址)"
  │
  ├── s->n_connections++
  │
  └── manager_add_job(JOB_START, service, JOB_REPLACE)
        → 启动实例化服务
```

---

## 六、文件描述符传递机制

### 6.1 整体 FD 流转图

```
                PID 1 进程空间                    │    服务进程空间
                                                  │
Socket 单元                                       │
  SocketPort.fd ──┐                               │
  SocketPort.fd ──┤                               │
  SocketPort.fd ──┘                               │
       │                                          │
       │ service_collect_fds()                    │
       ▼                                          │
  int *fds = [fd0, fd1, fd2]                      │
  char **fd_names = ["http","https","mgmt"]       │
  n_socket_fds = 3                                │
       │                                          │
       │ ExecParameters.fds = fds                 │
       │ ExecParameters.n_socket_fds = 3          │
       ▼                                          │
  exec_child_process() ─── fork() ────────────────┤
       │                                          │
       │  在子进程 exec() 前：                     │
       │  1. dup2(fds[i], SD_LISTEN_FDS_START+i)  │
       │  2. 设置环境变量                          │
       │     LISTEN_PID=<子进程PID>                │
       │     LISTEN_FDS=3                          │
       │     LISTEN_FDNAMES=http:https:mgmt        │
       │  3. exec() 目标程序                       │
       │                                          │    ┌─────────────────┐
       │                                          │    │ 服务程序启动      │
                                                  │    │                   │
                                                  │    │ sd_listen_fds(1)  │
                                                  │    │   检查 LISTEN_PID │
                                                  │    │   读取 LISTEN_FDS │
                                                  │    │   → n=3           │
                                                  │    │                   │
                                                  │    │ fd3 = 第一个 socket│
                                                  │    │ fd4 = 第二个 socket│
                                                  │    │ fd5 = 第三个 socket│
                                                  │    └─────────────────┘
```

### 6.2 service_collect_fds() — 服务端收集 FD

`service.c:1260-1374`：

```c
// Accept=no（单例模式）
if (s->socket_fd >= 0) {
    // 每连接模式：只传递该连接的 FD
    rfds = { s->socket_fd };
    rfd_names = { "connection" };
    rn_socket_fds = 1;
} else {
    // 单例模式：收集所有触发此服务的 Socket 单元的 FD
    UNIT_FOREACH_DEPENDENCY(u, UNIT(s), UNIT_ATOM_TRIGGERED_BY) {
        if (u->type != UNIT_SOCKET) continue;
        cn_fds = socket_collect_fds(SOCKET(u), &cfds);
        // 合并到 rfds 数组
        rfds = rfds + cfds;
        rn_socket_fds += cn_fds;
        // FD 名称使用 Socket.FileDescriptorName= 或单元名
        rfd_names += socket_fdname(sock) × cn_fds;
    }
}

// 追加 fd-store 中的 FD（FileDescriptorStoreMax=）
LIST_FOREACH(fd_store, fs, s->fd_store) {
    rfds[n] = fs->fd;
    rfd_names[n] = fs->fdname;
    rn_storage_fds++;
}
```

### 6.3 环境变量设置

`execute.c:1837-1855`，在子进程中设置：

```c
if (n_fds > 0) {
    // 设置 LISTEN_PID=<当前进程PID>
    asprintf(&x, "LISTEN_PID=" PID_FMT, getpid_cached());

    // 设置 LISTEN_FDS=<FD数量>
    asprintf(&x, "LISTEN_FDS=%zu", n_fds);

    // 设置 LISTEN_FDNAMES=<名称1>:<名称2>:...
    joined = strv_join(p->fd_names, ":");
    x = strjoin("LISTEN_FDNAMES=", joined);
}
```

### 6.4 sd_listen_fds() — 服务端 API

`sd-daemon.c:42-90`：

```c
_public_ int sd_listen_fds(int unset_environment) {
    // 1. 读取 LISTEN_PID，检查是否是发给自己的
    e = getenv("LISTEN_PID");
    parse_pid(e, &pid);
    if (getpid_cached() != pid) return 0;  // 不是给我的

    // 2. 读取 LISTEN_FDS，获取 FD 数量
    e = getenv("LISTEN_FDS");
    safe_atoi(e, &n);

    // 3. 设置所有 FD 的 CLOEXEC 标志
    for (fd = SD_LISTEN_FDS_START; fd < SD_LISTEN_FDS_START + n; fd++)
        fd_cloexec(fd, true);

    // 4. 可选清除环境变量（防止子进程继承）
    if (unset_environment) {
        unsetenv("LISTEN_PID");
        unsetenv("LISTEN_FDS");
        unsetenv("LISTEN_FDNAMES");
    }

    return n;  // 返回 FD 数量
}
```

**SD_LISTEN_FDS_START = 3**（stdin=0, stdout=1, stderr=2 之后）。

### 6.5 FD 类型检查 API

```c
// 检查 fd 是否是指定类型的 socket
sd_is_socket(fd, AF_INET, SOCK_STREAM, 1/*listening*/)

// 检查是否是 Unix 域 socket
sd_is_socket_unix(fd, SOCK_STREAM, 1, "/run/foo.sock", 0)

// 检查是否是 INET socket
sd_is_socket_inet(fd, AF_INET6, SOCK_STREAM, 1, 8080)

// 检查是否是 FIFO
sd_is_fifo(fd, "/run/foo.fifo")

// 检查是否是特殊文件
sd_is_special(fd, "/dev/initctl")

// 检查是否是 POSIX 消息队列
sd_is_mq(fd, "/my-queue")
```

---

## 七、两种激活模式对比

### 7.1 Accept=no（单例模式，默认）

```ini
# foo.socket
[Socket]
ListenStream=8080

# foo.service（普通服务，非模板）
[Service]
ExecStart=/usr/bin/foo-daemon
```

```
流程：
  PID 1: bind(:8080) + listen()
  客户端连接 → EPOLLIN
  PID 1: socket_enter_running(s, -1)
       → manager_add_job(START, foo.service)
  foo.service 启动:
       → service_collect_fds() 收集所有 Socket FD
       → fork() + 设置 LISTEN_FDS=1
       → exec foo-daemon
  foo-daemon:
       → n = sd_listen_fds(1)  // n=1
       → accept(3, ...)        // FD 3 是监听套接字
       → 服务自行 accept 和处理连接
```

### 7.2 Accept=yes（每连接模式）

```ini
# foo.socket
[Socket]
ListenStream=8080
Accept=yes

# foo@.service（模板服务）
[Service]
ExecStart=/usr/bin/foo-handler
StandardInput=socket
```

```
流程：
  PID 1: bind(:8080) + listen()
  客户端连接 → EPOLLIN
  PID 1: cfd = accept(fd)    ← PID 1 执行 accept
       → socket_enter_running(s, cfd)
       → 实例化 foo@<对端地址>.service
       → service_set_socket_fd(service, cfd)
       → manager_add_job(START, foo@xxx.service)
  foo@xxx.service 启动:
       → LISTEN_FDS=1, FD 3 = 已连接的 socket
       → exec foo-handler
  foo-handler:
       → 直接读写 FD 3（已是连接状态，无需 accept）
```

### 7.3 对比表

| 特性 | Accept=no | Accept=yes |
|------|-----------|------------|
| 服务类型 | 普通服务 | 模板服务（`@.service`） |
| accept() 执行者 | 服务程序自己 | PID 1 |
| 传递的 FD | 监听 socket | 已连接 socket |
| 服务实例数 | 1 个 | 每连接 1 个 |
| MaxConnections | 不适用 | 生效 |
| MaxConnectionsPerSource | 不适用 | 生效 |
| 适用场景 | 长驻守护进程 | 简单请求处理器 |
| 典型例子 | sshd、nginx | 简单 CGI |

---

## 八、Socket 选项配置

### 8.1 配置指令一览

```ini
[Socket]
# 监听地址
ListenStream=            # TCP / Unix SOCK_STREAM
ListenDatagram=          # UDP / Unix SOCK_DGRAM
ListenSequentialPacket=  # SCTP / Unix SOCK_SEQPACKET
ListenFIFO=              # 命名管道
ListenSpecial=           # 特殊文件
ListenMessageQueue=      # POSIX 消息队列
ListenUSBFunction=       # USB Function

# 连接控制
Accept=no                # 是否每连接实例化
MaxConnections=64        # 最大同时连接（Accept=yes）
MaxConnectionsPerSource=0 # 单源最大连接（0=无限制）
Backlog=4096             # listen() backlog
TriggerLimitIntervalSec= # 触发频率限制时间窗口
TriggerLimitBurst=       # 触发频率限制阈值

# Socket 选项
KeepAlive=               # SO_KEEPALIVE
KeepAliveTimeSec=        # TCP_KEEPIDLE
KeepAliveIntervalSec=    # TCP_KEEPINTVL
KeepAliveProbes=         # TCP_KEEPCNT
NoDelay=                 # TCP_NODELAY
FreeBind=                # IP_FREEBIND
Transparent=             # IP_TRANSPARENT
Broadcast=               # SO_BROADCAST
ReusePort=               # SO_REUSEPORT
DeferAcceptSec=          # TCP_DEFER_ACCEPT
ReceiveBuffer=           # SO_RCVBUF
SendBuffer=              # SO_SNDBUF
IPTOS=                   # IP_TOS
IPTTL=                   # IP_TTL
Mark=                    # SO_MARK
PipeSize=                # F_SETPIPE_SZ（FIFO）
Priority=                # SO_PRIORITY

# 安全选项
PassCredentials=         # SO_PASSCRED
PassSecurity=            # SO_PASSSEC
PassPacketInfo=          # IP_PKTINFO
SocketProtocol=          # IPPROTO_SCTP / IPPROTO_UDPLITE
BindIPv6Only=            # IPV6_V6ONLY
BindToDevice=            # SO_BINDTODEVICE

# 权限与路径
SocketUser=              # socket 文件所有者
SocketGroup=             # socket 文件所属组
SocketMode=0666          # socket 文件权限
DirectoryMode=0755       # 路径中目录的权限
RemoveOnStop=            # 停止时删除 socket 文件
Symlinks=                # 创建指向 socket 的符号链接
FlushPending=            # 监听前清空缓冲数据

# FD 命名
FileDescriptorName=      # LISTEN_FDNAMES 中的名称
Service=                 # 指定触发的服务（默认同名）

# 执行钩子
ExecStartPre=            # 打开 socket 前执行
ExecStartPost=           # 打开 socket 后执行
ExecStopPre=             # 关闭 socket 前执行
ExecStopPost=            # 关闭 socket 后执行
```

---

## 九、socket_apply_socket_options() — 选项应用

`socket.c:975+`，在 FD 打开后逐一应用配置的 socket 选项：

```
socket_apply_socket_options(s, p, fd)
  │
  ├── SO_PASSCRED (pass_cred)
  ├── SO_PASSSEC (pass_sec)
  ├── IP_PKTINFO / IPV6_RECVPKTINFO (pass_pktinfo)
  ├── SO_TIMESTAMP / SO_TIMESTAMPNS (timestamping)
  ├── SO_KEEPALIVE (keep_alive)
  │     ├── TCP_KEEPIDLE (keep_alive_time)
  │     ├── TCP_KEEPINTVL (keep_alive_interval)
  │     └── TCP_KEEPCNT (keep_alive_cnt)
  ├── SO_BROADCAST (broadcast)
  ├── TCP_NODELAY (no_delay)
  ├── SO_PRIORITY (priority)
  ├── IP_TOS (ip_tos)
  ├── IP_TTL (ip_ttl)
  ├── SO_MARK (mark)
  ├── SO_RCVBUF (receive_buffer)
  ├── SO_SNDBUF (send_buffer)
  ├── IP_FREEBIND (free_bind)
  ├── IP_TRANSPARENT (transparent)
  ├── SO_REUSEPORT (reuse_port)
  ├── SO_BINDTODEVICE (bind_to_device)
  ├── TCP_CONGESTION (tcp_congestion)
  ├── TCP_DEFER_ACCEPT (defer_accept)
  ├── SCTP options (sctp_*_appdata)
  └── SMACK labels (smack, smack_ip_in, smack_ip_out)
```

---

## 十、SocketPeer — 对端地址跟踪

`socket.c:50-56`，用于 `MaxConnectionsPerSource` 的实现：

```c
struct SocketPeer {
    unsigned n_ref;        // 引用计数（= 来自同一地址的活跃连接数）
    Socket *socket;        // 所属 Socket 单元
    union sockaddr_union peer;  // 对端地址
    socklen_t peer_salen;       // 地址长度
};
```

工作原理：
1. `socket_acquire_peer(s, cfd, &p)` — 用 `getpeername()` 获取对端地址
2. 在 `s->peers_by_address` 集合中查找是否已有记录
3. 已有 → 引用计数 +1
4. 没有 → 创建新的 SocketPeer
5. 检查 `p->n_ref > max_connections_per_source` → 超限则拒绝
6. 服务结束时 `socket_peer_unref()` → 引用计数 -1 → 降为 0 时从集合移除

---

## 十一、触发频率限制

`socket.h:161`：

```c
RateLimit trigger_limit;  // 频率限制器
```

在 `socket_enter_running()` 中检查（`socket.c:2333-2337`）：

```c
if (!ratelimit_below(&s->trigger_limit)) {
    log_unit_warning(UNIT(s),
        "Trigger limit hit, refusing further activation.");
    socket_enter_stop_pre(s, SOCKET_FAILURE_TRIGGER_LIMIT_HIT);
    goto refuse;
}
```

防止恶意客户端疯狂连接导致服务被无限实例化。

配置：
```ini
[Socket]
TriggerLimitIntervalSec=2s
TriggerLimitBurst=200
```

---

## 十二、服务端编程模式

### 12.1 最简 C 语言示例

```c
#include <systemd/sd-daemon.h>

int main(void) {
    int n = sd_listen_fds(1);  // 获取 FD，同时清除环境变量
    if (n < 0) return EXIT_FAILURE;
    if (n == 0) {
        // 非 socket activation 启动，自行 bind/listen
        ...
    }

    int listen_fd = SD_LISTEN_FDS_START;  // = 3

    // 验证 FD 类型
    if (sd_is_socket_inet(listen_fd, AF_INET6, SOCK_STREAM, 1, 8080) <= 0)
        return EXIT_FAILURE;

    // 主循环
    for (;;) {
        int cfd = accept4(listen_fd, NULL, NULL, SOCK_CLOEXEC);
        if (cfd < 0) continue;
        handle_connection(cfd);
        close(cfd);
    }
}
```

### 12.2 对应的 .socket 文件

```ini
[Unit]
Description=My Service Socket

[Socket]
ListenStream=8080
BindIPv6Only=both

[Install]
WantedBy=sockets.target
```

### 12.3 对应的 .service 文件

```ini
[Unit]
Description=My Service

[Service]
ExecStart=/usr/bin/my-service
Type=simple
```

---

## 十三、与依赖系统的集成

Socket activation 利用 systemd 的依赖系统实现自动关联：

```
sshd.socket ──── Triggers ────> sshd.service
             ──── Before ─────>

加载 sshd.socket 时自动添加：
  sshd.socket Before sshd.service  （socket 先于服务）
  sshd.socket Triggers sshd.service（socket 触发服务）

这意味着：
  1. 启动 sshd.socket 时不会启动 sshd.service
  2. 有连接到来时 socket_enter_running() 提交 JOB_START
  3. sshd.service 启动时通过 TRIGGERED_BY 找到 sshd.socket
  4. service_collect_fds() 收集 sshd.socket 的 FD
```

`sockets.target` 是所有 socket 的同步点：
```
sockets.target ─── Wants ──> *.socket
                ─── After ──> *.socket

multi-user.target ─── After ──> sockets.target
```

这确保所有 socket 在服务启动前打开，实现**并行启动零阻塞**。
