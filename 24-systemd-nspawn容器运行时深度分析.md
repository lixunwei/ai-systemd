# 24. systemd-nspawn 容器运行时深度分析

## 1. 概述

`systemd-nspawn` 是 systemd 提供的**轻量级容器运行时**，使用 Linux namespace、seccomp 和 capabilities 实现进程隔离。它的设计定位介于 `chroot` 和完整虚拟机之间——比 chroot 提供更强的隔离，比 VM 更轻量。

### 核心特点

- **完整的 namespace 隔离**：PID、Mount、UTS、IPC、Network、User、Cgroup
- **多种根文件系统**：目录树、磁盘镜像（.raw）、btrfs 子卷
- **OCI 兼容**：支持 OCI Runtime Bundle 格式
- **与 systemd 生态集成**：machined 注册、journald 日志、cgroup 资源控制
- **双进程模型**：outer child（mount namespace）→ inner child（完整隔离）

### 源文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `nspawn.c` | 5887 | 主程序：CLI 解析、容器启动流程、父子协调 |
| `nspawn-mount.c` | 1302 | 文件系统组装：bind/overlay/tmpfs、/proc、/sys、/dev、pivot_root |
| `nspawn-network.c` | 750 | 网络设置：veth、macvlan、ipvlan、bridge |
| `nspawn-seccomp.c` | 260 | seccomp 系统调用过滤 |
| `nspawn-settings.h` | 287 | Settings 结构体定义、枚举、配置掩码 |
| `nspawn-settings.c` | 955 | .nspawn 配置文件解析 |
| `nspawn-cgroup.c` | 605 | cgroup 创建/同步/挂载 |
| `nspawn-register.c` | 361 | machined 注册（D-Bus） |
| `nspawn-oci.c` | 2274 | OCI Bundle 解析 |
| `nspawn-expose-ports.c` | 228 | 端口转发（DNAT） |
| `nspawn-patch-uid.c` | 475 | UID/GID 递归修补（用户命名空间） |
| `nspawn-stub-pid1.c` | 200 | 容器内 stub PID 1（--as-pid2 模式） |
| `nspawn-bind-user.c` | 475 | --bind-user 映射逻辑 |
| `nspawn-setuid.c` | 245 | UID/GID 切换辅助 |
| `nspawn-creds.c` | 25 | 凭据传递 |

---

## 2. Settings 结构体（nspawn-settings.h:162-237）

容器的所有配置集中在一个结构体中：

```c
typedef struct Settings {
    /* [Exec] 段 */
    StartMode start_mode;           // START_PID1 / START_PID2 / START_BOOT
    int ephemeral;                  // 临时容器（退出后丢弃变更）
    char **parameters;              // 容器内执行的命令
    char **environment;             // 环境变量
    char *user;                     // 容器内用户
    uint64_t capability;            // 保留的 capabilities
    uint64_t drop_capability;       // 丢弃的 capabilities
    uint64_t ambient_capability;    // ambient capabilities
    int kill_signal;                // 停止信号
    unsigned long personality;      // 执行域（x86-64 / i686）
    sd_id128_t machine_id;          // 容器 machine-id
    char *working_directory;        // 工作目录
    char *pivot_root_new/old;       // pivot_root 路径
    UserNamespaceMode userns_mode;  // NO / PICK / FIXED
    uid_t uid_shift, uid_range;     // UID 偏移量和范围（默认 0x10000）
    int notify_ready;               // sd_notify READY=1
    char **syscall_allow_list;      // seccomp 白名单
    char **syscall_deny_list;       // seccomp 黑名单
    struct rlimit *rlimit[_RLIMIT_MAX];
    char *hostname;                 // UTS hostname
    int no_new_privileges;          // PR_SET_NO_NEW_PRIVS
    int oom_score_adjust;           // OOM 评分
    CPUSet cpu_set;                 // CPU 亲和性
    ResolvConfMode resolv_conf;     // resolv.conf 处理方式
    LinkJournal link_journal;       // journal 链接模式
    TimezoneMode timezone;          // 时区处理
    int suppress_sync;              // 抑制 fsync

    /* [Files] 段 */
    int read_only;                  // 只读根文件系统
    VolatileMode volatile_mode;     // VOLATILE_NO / STATE / YES / OVERLAY
    CustomMount *custom_mounts;     // 自定义挂载点
    size_t n_custom_mounts;
    UserNamespaceOwnership userns_ownership;
    char **bind_user;               // 绑定用户

    /* [Network] 段 */
    int private_network;            // 独立网络命名空间
    int network_veth;               // 创建 veth 对
    char *network_bridge;           // 桥接名
    char *network_zone;             // 网络区域
    char **network_interfaces;      // 移入容器的宿主接口
    char **network_macvlan;         // macvlan 接口列表
    char **network_ipvlan;          // ipvlan 接口列表
    char **network_veth_extra;      // 额外 veth 对
    ExposePort *expose_ports;       // 端口转发规则

    /* OCI 专用字段 */
    char *bundle;                   // OCI bundle 路径
    char *root;                     // OCI rootfs 路径
    OciHook *oci_hooks_prestart/poststart/poststop;
    char *slice;                    // systemd slice
    sd_bus_message *properties;     // 资源属性
    unsigned long clone_ns_flags;   // 额外 clone flags
    char *network_namespace_path;   // 外部 netns 路径
    int use_cgns;                   // 使用 cgroup namespace
    char **sysctl;                  // sysctl 设置
    scmp_filter_ctx seccomp;        // OCI seccomp 过滤器
} Settings;
```

---

## 3. 默认 Capabilities（nspawn.c:147-173）

容器默认保留 25 个 capabilities：

```c
arg_caps_retain =
    CAP_AUDIT_CONTROL | CAP_AUDIT_WRITE |
    CAP_CHOWN | CAP_DAC_OVERRIDE | CAP_DAC_READ_SEARCH |
    CAP_FOWNER | CAP_FSETID | CAP_IPC_OWNER |
    CAP_KILL | CAP_LEASE | CAP_LINUX_IMMUTABLE |
    CAP_MKNOD | CAP_NET_BIND_SERVICE | CAP_NET_BROADCAST | CAP_NET_RAW |
    CAP_SETFCAP | CAP_SETGID | CAP_SETPCAP | CAP_SETUID |
    CAP_SYS_ADMIN | CAP_SYS_BOOT | CAP_SYS_CHROOT |
    CAP_SYS_NICE | CAP_SYS_PTRACE | CAP_SYS_RESOURCE | CAP_SYS_TTY_CONFIG;
```

**注意**：包含 `CAP_SYS_ADMIN`，这使得容器内可以执行 mount 等操作。生产环境应按需裁剪。

---

## 4. 容器启动流程

### 4.1 总体架构

```
┌───────────────────────────────────────────────────────┐
│                    nspawn 进程 (父)                     │
│                                                       │
│  run()                                                │
│   ├─ parse_argv()         解析命令行                   │
│   ├─ load settings        加载 .nspawn / OCI bundle   │
│   ├─ prepare rootfs       准备根文件系统                │
│   ├─ PR_SET_CHILD_SUBREAPER                          │
│   └─ loop: run_container()                           │
│        ├─ 创建 8 组 socketpair                        │
│        ├─ 创建 barrier 同步对象                        │
│        ├─ raw_clone(SIGCHLD|CLONE_NEWNS)             │
│        │                                             │
│        ├─ [父] 等待 outer child 信息                   │
│        │   ├─ 接收 UID shift                         │
│        │   ├─ 写入 uid_map/gid_map                   │
│        │   ├─ 设置网络（veth/bridge/macvlan）          │
│        │   ├─ 创建 cgroup（payload/supervisor）        │
│        │   ├─ 注册到 machined                         │
│        │   └─ sd_event_loop（监听 notify/pty）         │
│        │                                             │
│        └─ [子] outer_child()                         │
│             ├─ MS_SLAVE|MS_REC 全局                   │
│             ├─ 挂载镜像/设置 UID shift                 │
│             ├─ raw_clone(...|CLONE_NEWPID|CLONE_NEWUTS|...) │
│             └─ [孙] inner_child()                    │
│                  ├─ mount_all() (proc/sys/dev)       │
│                  ├─ setup_network (loopback)         │
│                  ├─ mount_custom() (bind/overlay)    │
│                  ├─ pivot_root()                     │
│                  ├─ setup_seccomp()                  │
│                  ├─ drop privileges                  │
│                  └─ execv(payload)                   │
└───────────────────────────────────────────────────────┘
```

### 4.2 入口函数（nspawn.c:5420）

```c
static int run(int argc, char *argv[]) {
    parse_argv(argc, argv);           // 150+ 命令行选项
    
    // 镜像/目录处理
    if (arg_image)
        loop_device + dissect_image   // 挂载磁盘镜像
    else if (arg_directory)
        btrfs_subvol_snapshot()       // ephemeral 模式下快照
    
    // 加载 .nspawn 配置或 OCI bundle
    settings_load() / oci_load()
    
    // 安全设置
    prctl(PR_SET_CHILD_SUBREAPER)     // 成为子进程收割者
    
    // 主循环（支持容器内 reboot）
    for (;;) {
        r = run_container(...)
        if (r == 0 || r == 133)       // 正常退出或 reboot
            continue / break
    }
}
```

### 4.3 run_container()（nspawn.c:4700-5205）

这是容器启动的核心函数：

```
1. 创建通信通道:
   - kmsg_socket_pair      → 容器 kmsg 日志
   - rtnl_socket_pair      → rtnetlink fd 传递
   - pid_socket_pair       → inner child PID 传回
   - uuid_socket_pair      → 容器 UUID 传回
   - notify_socket_pair    → sd_notify 通道
   - master_pty_socket_pair → PTY master fd
   - uid_shift_socket_pair → UID shift 协商
   - unified_cgroup_hierarchy_socket_pair → cgroup 模式

2. 创建 barrier 同步对象（基于 eventfd）

3. raw_clone(SIGCHLD|CLONE_NEWNS) → 创建 outer child
   - outer child 只有独立的 Mount namespace
   - 后续 namespace 在 inner child 中创建

4. 父进程流程:
   ├─ 接收 outer child 发来的 UID shift
   ├─ 若需要用户命名空间：写入 /proc/PID/uid_map, gid_map
   ├─ barrier_place(#2) 通知 outer child uid_map 已就绪
   ├─ 接收 inner PID
   ├─ barrier_place(#3) 通知可以设置网络
   ├─ setup_veth / setup_bridge / setup_macvlan / setup_ipvlan
   ├─ move_network_interfaces() 移动宿主接口到容器 netns
   ├─ 创建 cgroup: payload + supervisor 子组
   ├─ register_machine() 注册到 machined
   ├─ barrier_place(#4) 通知 inner child 可以继续
   ├─ 设置 sd-event 事件循环
   │   ├─ 监听 notify fd → sd_notify READY=1
   │   ├─ 监听 pty master → 终端 I/O
   │   └─ 监听 SIGCHLD → 容器退出
   └─ sd_event_loop() 等待容器退出
```

---

## 5. Namespace 配置

### 5.1 默认 namespace flags

```c
// nspawn.c:209
arg_clone_ns_flags = CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWUTS;
```

加上 outer child 自带的 `CLONE_NEWNS`，默认有 4 个 namespace。

### 5.2 各 namespace 创建时机

| Namespace | 创建位置 | 条件 |
|-----------|----------|------|
| Mount (NEWNS) | outer child clone | 始终创建 |
| PID (NEWPID) | inner child clone | 始终（在 arg_clone_ns_flags 中） |
| UTS (NEWUTS) | inner child clone | 始终 |
| IPC (NEWIPC) | inner child clone | 始终 |
| Network (NEWNET) | inner child unshare | --private-network 或 --network-veth 等 |
| User (NEWUSER) | outer child clone | --private-users |
| Cgroup (NEWCGROUP) | inner child unshare | --use-cgns（默认启用） |

### 5.3 可通过环境变量禁用

```c
// nspawn.c:578-589 parse_share_ns_env()
SYSTEMD_NSPAWN_SHARE_NS_IPC=1    → 清除 CLONE_NEWIPC
SYSTEMD_NSPAWN_SHARE_NS_PID=1    → 清除 CLONE_NEWPID
SYSTEMD_NSPAWN_SHARE_NS_UTS=1    → 清除 CLONE_NEWUTS
SYSTEMD_NSPAWN_SHARE_SYSTEM=1    → 清除全部三个
```

---

## 6. 文件系统组装（nspawn-mount.c）

### 6.1 API 虚拟文件系统挂载

`mount_all()` 函数（nspawn-mount.c:478）按表驱动方式挂载：

| 挂载点 | 类型 | 标志 | 说明 |
|--------|------|------|------|
| `/proc` | proc | MS_NOSUID\|MS_NOEXEC\|MS_NODEV | 进程信息 |
| `/proc/sys` | bind(self) | MS_RDONLY | 防止修改内核参数 |
| `/sys` | sysfs | MS_RDONLY\|MS_NOSUID\|MS_NOEXEC\|MS_NODEV | 设备拓扑 |
| `/sys/fs/selinux` | bind(self) | MS_RDONLY | SELinux（若存在） |
| `/dev` | tmpfs | MS_NOSUID\|MS_STRICTATIME | 设备节点 |
| `/dev/pts` | devpts | newinstance | 伪终端 |
| `/dev/shm` | tmpfs | MS_NOSUID\|MS_NODEV\|MS_STRICTATIME | 共享内存 |
| `/dev/mqueue` | mqueue | — | POSIX 消息队列 |
| `/tmp` | tmpfs | — | 临时目录（可选） |
| `/run` | tmpfs | MS_NOSUID\|MS_NODEV\|MS_STRICTATIME | 运行时状态 |

### 6.2 设备节点创建

在 `/dev` tmpfs 上创建标准设备节点：

```
/dev/null, /dev/zero, /dev/full, /dev/random, /dev/urandom,
/dev/tty, /dev/net/tun, /dev/loop-control
```

非特权容器使用 bind mount 而非 mknod。

### 6.3 pivot_root（nspawn-mount.c:1219）

```c
setup_pivot_root(directory, pivot_root_new, pivot_root_old):
  1. mount(directory, directory, MS_BIND|MS_REC)  // 自绑定
  2. pivot_root(directory, oldroot)               // 切换根
  3. umount2(oldroot, MNT_DETACH)                 // 卸载旧根
  4. 可选：将 pivot_root_old 绑定挂载到指定路径
```

### 6.4 Volatile 模式（nspawn-mount.c:1163）

| 模式 | 行为 |
|------|------|
| `VOLATILE_NO` | 正常使用根文件系统 |
| `VOLATILE_STATE` | `/var` 使用 tmpfs 覆盖 |
| `VOLATILE_YES` | 整个根使用 tmpfs + overlay |
| `VOLATILE_OVERLAY` | overlay 叠加在根之上 |

---

## 7. 网络配置（nspawn-network.c）

### 7.1 veth 对创建（nspawn-network.c:237）

```c
setup_veth(machine_name, pid, veth_name, bridge):
  1. 生成接口名: "ve-<machine_name>" (host 端)
                 "host0" (容器端)
  2. 通过 netlink RTM_NEWLINK:
     - 创建 veth 对
     - host 端留在宿主 namespace
     - container 端通过 IFLA_NET_NS_PID 移入容器
  3. 启动 host 端接口
  4. 若指定 bridge：将 host 端加入网桥
```

### 7.2 网桥管理（nspawn-network.c:395）

```c
setup_bridge(veth_name, bridge_name, create_missing):
  如果网桥不存在且在 --network-zone 模式下：
    自动创建以 "vz-<zone>" 命名的网桥
  将 veth host 端设为网桥的 slave
```

### 7.3 macvlan / ipvlan（nspawn-network.c:532/620）

```
macvlan: 基于宿主物理接口创建虚拟 MAC 地址接口
  - 接口名: "mv-<host_iface>"
  - 移入容器 namespace
  
ipvlan: 共享 MAC 但使用不同 IP
  - 接口名: "iv-<host_iface>"
  - 移入容器 namespace
```

### 7.4 宿主接口移动（nspawn-network.c:496）

```c
move_network_interfaces(netns_fd, interfaces):
  通过 netlink RTM_NEWLINK + IFLA_NET_NS_FD
  将指定的宿主接口直接移入容器 network namespace
  容器获得该接口的完全控制权
```

### 7.5 端口转发（nspawn-expose-ports.c）

```
--port=tcp:8080:80 的实现:
  1. 解析: 协议=tcp, 宿主端口=8080, 容器端口=80
  2. 监听 rtnetlink 地址变更事件
  3. 当容器获得 IP 地址时:
     安装 iptables/nftables DNAT 规则:
     -A PREROUTING -p tcp --dport 8080 -j DNAT --to <container_ip>:80
  4. 通过 rtnl_socket_pair 将 fd 传给容器
```

---

## 8. 安全隔离

### 8.1 seccomp 过滤（nspawn-seccomp.c:184-251）

```c
setup_seccomp(caps_retain, allow_list, deny_list):
  1. 创建 seccomp filter（默认 ALLOW）
  2. 构建黑名单:
     - 基础黑名单: 37 个危险系统调用
       (open_by_handle_at, bpf, clock_settime, mount, reboot, ...)
     - 条件黑名单（根据 caps 决定）:
       无 CAP_SYS_RAWIO → 禁止 iopl, ioperm
       无 CAP_SYS_TIME → 禁止 clock_settime, settimeofday
       无 CAP_NET_ADMIN → 禁止 sethostname, setdomainname
  3. 禁止创建 NETLINK_AUDIT socket（即使有 CAP_AUDIT_*）
  4. 应用用户指定的 allow/deny list 覆盖
  5. seccomp_load() 安装过滤器
```

### 8.2 Capability 削减流程

```
outer_child:
  prctl(PR_SET_KEEPCAPS, 1)    // 跨 setuid 保留 caps

inner_child:
  1. setgroups/setgid/setuid    // 降权到目标用户
  2. 计算最终 cap 集:
     - 白名单: arg_caps_retain
     - 黑名单: arg_caps_drop
  3. 设置 bounding set（cap_bset）
  4. 设置 ambient capabilities
  5. prctl(PR_SET_NO_NEW_PRIVS) // 若配置
```

### 8.3 用户命名空间（User Namespace）

```
模式:
  USER_NAMESPACE_NO    → 不使用（容器内 root = 宿主 root）
  USER_NAMESPACE_PICK  → 自动从 /etc/subuid 分配 UID 范围
  USER_NAMESPACE_FIXED → 使用指定的 uid_shift

UID 映射:
  /proc/<outer_child_pid>/uid_map: "0 <uid_shift> <uid_range>"
  - 容器内 UID 0 映射到宿主 UID uid_shift
  - 默认范围: 0x10000（65536 个 UID）

文件系统适配:
  - idmapped mounts（内核 5.12+）: 最优方案，零拷贝
  - 递归 chown（nspawn-patch-uid.c）: 后备方案，逐文件修改
```

---

## 9. outer_child ↔ inner_child ↔ 父进程通信

### 9.1 通信通道

```
┌────────┐  uid_shift_socket  ┌─────────────┐
│  父    │◄──────────────────│ outer child │
│  进程  │  pid_socket        │ (CLONE_NEWNS)│
│        │◄──────────────────│             │
│        │  uuid_socket       │             │
│        │◄──────────────────│             │
│        │  notify_socket     │             │
│        │◄──────────────────│             │
│        │  master_pty_socket │             │
│        │◄──────────────────│             │
│        │  rtnl_socket       │             │
│        │────────────────→ │             │
│        │  unified_cgroup    │             │
│        │◄──────────────────│             │
└────────┘                    └──────┬──────┘
                                     │
                                     │ clone(NEWPID|NEWUTS|NEWIPC)
                                     ▼
                              ┌─────────────┐
                              │ inner child │
                              │ (完整隔离)  │
                              │ → 执行载荷  │
                              └─────────────┘
```

### 9.2 Barrier 同步序列

基于 eventfd 的双向同步：

| 阶段 | 触发方 | 等待方 | 用途 |
|------|--------|--------|------|
| #1 | outer child | 父 | outer child 已启动 |
| #2 | 父 | outer child | uid_map/gid_map 已写入 |
| #3 | outer child | 父 | inner child 已创建，可设网络 |
| #4 | 父 | inner child | 网络/cgroup 设置完成，可继续启动 |
| #5 | inner child | 父 | 载荷已就绪（或已 execv） |

---

## 10. cgroup 设置（nspawn-cgroup.c）

### 10.1 双层结构

```
/sys/fs/cgroup/machine.slice/
  └── systemd-nspawn@<machine>.service/
      ├── supervisor/     ← nspawn 进程自身
      └── payload/        ← 容器内进程
          └── (容器内 systemd 在此创建子 cgroup)
```

### 10.2 创建流程（nspawn-cgroup.c:145-195）

```c
create_subcgroup(pid, keep_unit, unified):
  1. 确定容器 cgroup 路径
  2. mkdir "supervisor" 和 "payload"
  3. 启用所需控制器（cpu, memory, io, pids 等）
  4. 将容器 PID 移入 payload/
  5. 将 nspawn 自身移入 supervisor/
```

### 10.3 cgroup 挂载（nspawn-cgroup.c:246-524）

根据 unified/legacy/hybrid 模式：

| 模式 | 挂载方式 |
|------|----------|
| unified (v2) | /sys/fs/cgroup → cgroup2 (payload/ 子树) |
| legacy (v1) | /sys/fs/cgroup/XXX → 各控制器 |
| hybrid | /sys/fs/cgroup → tmpfs + /sys/fs/cgroup/unified → cgroup2 |

容器内只能看到自己的 cgroup 子树（通过 cgroup namespace 或 bind mount 限制）。

---

## 11. Stub PID 1（nspawn-stub-pid1.c:33-200）

当使用 `--as-pid2` 模式时，nspawn 在容器内注入一个最小 PID 1：

```c
int stub_pid1(sd_id128_t uuid):
  状态机: STATE_RUNNING → STATE_REBOOT / STATE_POWEROFF

  1. fork() — 子进程返回执行真正载荷
  2. 父进程成为 stub init:
     a. setsid() 成为会话领导
     b. close_all_fds() 关闭继承的 fd
     c. 重命名进程为 "(sd-stubinit)"
     d. 设置 /proc/1/environ:
        container=systemd-nspawn
        container_uuid=<uuid>
     e. 信号处理循环:
        SIGCHLD      → waitid() 收割僵尸进程
        SIGINT       → STATE_POWEROFF (ctrl-alt-del)
        SIGRTMIN+3   → STATE_POWEROFF (halt)
        SIGRTMIN+4   → STATE_POWEROFF (poweroff)
        SIGRTMIN+5   → STATE_REBOOT (reboot)
     f. 当载荷退出:
        向所有进程发送 SIGTERM → 等待 → SIGKILL
     g. 返回退出码（133 = reboot 请求）
```

---

## 12. OCI 兼容（nspawn-oci.c）

### 12.1 支持的 OCI 字段

| OCI 字段 | nspawn 映射 |
|----------|------------|
| `process.args` | parameters |
| `process.env` | environment |
| `process.cwd` | working_directory |
| `process.capabilities` | capability / drop_capability |
| `process.rlimits` | rlimit[] |
| `process.noNewPrivileges` | no_new_privileges |
| `process.oomScoreAdj` | oom_score_adjust |
| `process.user.uid/gid` | uid/gid |
| `root.path` | root（相对于 bundle） |
| `root.readonly` | read_only |
| `mounts` | custom_mounts（bind 或 tmpfs） |
| `hostname` | hostname |
| `linux.namespaces` | clone_ns_flags / private_network / userns_mode / use_cgns |
| `linux.uidMappings` | uid_shift / uid_range |
| `linux.seccomp` | seccomp filter |
| `linux.resources.memory` | systemd 属性 MemoryMax 等 |
| `linux.resources.cpu` | CPUWeight / CPUQuota |
| `linux.resources.pids` | TasksMax |
| `linux.resources.blockIO` | IOWeight |
| `hooks.prestart/poststart/poststop` | oci_hooks_* |

### 12.2 解析入口（nspawn-oci.c:2200-2274）

```c
oci_load(FILE *f, const char *bundle, Settings **ret):
  顶层字段:
    "ociVersion" → 验证版本
    "process"    → oci_process()
    "root"       → oci_root()
    "hostname"   → settings->hostname
    "mounts"     → oci_mounts()
    "linux"      → oci_linux() (namespaces, seccomp, resources, mappings)
    "hooks"      → oci_hooks()
    "annotations" → (忽略)
```

---

## 13. machined 注册（nspawn-register.c）

### 13.1 注册方式

```c
register_machine(bus, machine_name, pid, ...):
  方式1: CreateMachineWithNetwork (transient scope)
    - 创建 scope unit: machine-<name>.scope
    - 设置属性: Slice, DevicePolicy, DeviceAllow, KillSignal
    
  方式2: RegisterMachineWithNetwork (已有 unit)
    - 当 --keep-unit 时使用
    - 只注册，不创建新 scope
```

### 13.2 传递的属性

```
Slice       = machine.slice（或自定义）
DevicePolicy = strict
DeviceAllow  = [/dev/null, /dev/zero, /dev/full, ...]
KillSignal   = SIGRTMIN+3（或自定义）
KillMode     = mixed
```

注册后容器出现在 `machinectl list` 中，可通过 `machinectl shell/login/status` 管理。

---

## 14. 镜像与根文件系统处理

### 14.1 支持的根类型

| 类型 | 指定方式 | 处理 |
|------|----------|------|
| 目录 | `--directory=/path` | 直接使用或 btrfs snapshot |
| 磁盘镜像 | `--image=file.raw` | loop device + dissect + mount |
| OCI bundle | `--oci-bundle=/path` | 解析 config.json |

### 14.2 磁盘镜像处理流程

```
1. loop_device_make_by_path(image_path)
   → 创建 /dev/loop<N> 关联镜像文件

2. dissect_image_and_warn(loop, verity_settings)
   → 解析分区表（GPT）
   → 识别 root/home/srv/var 等分区
   → 验证 dm-verity（可选）

3. dissected_image_mount(dissected, directory, flags)
   → 逐分区 mount 到目标目录
   → 支持 fsck / growfs
   → 支持 idmapped mounts
```

### 14.3 Verity 支持

```
--root-hash=<hex>        指定 root hash
--root-hash-sig=<file>   签名文件
--verity-data=<file>     verity 数据分区
```

dm-verity 确保镜像完整性，任何篡改都会导致 I/O 错误。

---

## 15. 容器重启与退出处理

### 15.1 退出码约定

| 退出码 | 含义 | 父进程行为 |
|--------|------|-----------|
| 0 | 正常退出 | 退出 nspawn |
| 133 | 请求重启 | 重新运行 run_container() |
| 其他 | 错误退出 | 退出 nspawn（传递退出码） |

### 15.2 重启流程

```
容器内:
  systemctl reboot → SIGRTMIN+5 → inner child 退出(133)

nspawn 父:
  检测 exit code = 133
  loop 回到 run_container() 起点
  重新 clone → 重新挂载 → 重新执行
```

---

## 16. 完整启动时序图

```
时间轴 ──────────────────────────────────────────────────────────→

父进程      │ clone ──→ 等待uid ──→ 写uid_map ──→ 等待pid ──→ 网络 ──→ cgroup ──→ event_loop
            │    ↓         ↑            ↓           ↑          ↓          ↓
outer_child │    *─→ 发uid ─→ 等barrier#2 →├ mount image ─→ clone2 ──→ ...
            │                               ↓
inner_child │                               * ──→ 等barrier#4 ──→ mount_all ──→
            │                                                       setup_net ──→
            │                                                       pivot_root ──→
            │                                                       seccomp ──→
            │                                                       drop_privs ──→
            │                                                       execv(payload)

barrier:        #1              #2                    #3         #4         #5
```

---

## 17. 配置文件（.nspawn）

### 17.1 搜索路径

```
/etc/systemd/nspawn/<machine>.nspawn
/run/systemd/nspawn/<machine>.nspawn
~/.config/systemd/nspawn/<machine>.nspawn（用户）
```

### 17.2 示例

```ini
[Exec]
Boot=true
Hostname=mycontainer
Environment=LANG=en_US.UTF-8
Capability=CAP_NET_ADMIN
PrivateUsers=pick

[Files]
ReadOnly=false
Bind=/srv/data:/data
TemporaryFileSystem=/tmp
Volatile=no

[Network]
Private=yes
VirtualEthernet=yes
Bridge=br0
Port=tcp:8080:80
Zone=myzone
```

### 17.3 gperf 配置映射（nspawn-gperf.gperf:22-83）

配置项通过 gperf 哈希表映射到对应的解析函数，支持 60+ 配置键。

---

## 18. 设计要点总结

### 18.1 双 clone 架构的原因

- **outer child**（仅 CLONE_NEWNS）：负责挂载镜像、设置 UID shift。因为 mount namespace 需要在 uid_map 写入之前就就绪。
- **inner child**（NEWPID|NEWUTS|NEWIPC + 可选 NEWNET|NEWUSER|NEWCGROUP）：在干净的 mount 上执行最终载荷。PID namespace 要求 fork，使得容器内进程号从 1 开始。

### 18.2 socketpair 通信的优势

- 不依赖文件系统（无临时文件竞争）
- 天然支持 fd 传递（SCM_RIGHTS）
- 阻塞语义简化同步逻辑

### 18.3 与 Docker/runc 的对比

| 维度 | systemd-nspawn | runc/Docker |
|------|---------------|-------------|
| 定位 | 开发/测试/系统容器 | 生产应用容器 |
| 镜像格式 | 目录/磁盘镜像 | OCI image layers |
| Init 系统 | 支持完整 systemd | 通常单进程 |
| 网络 | 内建 veth/bridge | CNI 插件 |
| 存储 | bind mount / overlay | overlay2 / aufs |
| 编排 | machinectl | Docker/K8s |
| OCI 支持 | 部分（config.json） | 完整 |
| seccomp | 基于 caps 的自适应 | 固定 profile |
