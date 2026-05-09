# 基础库分析 — basic 与 shared

## 一、src/basic/ — 基础工具库

`src/basic/` 提供最底层的 C 工具函数，被所有 systemd 组件依赖。不依赖任何 systemd 特有概念，纯粹是 C 语言基础设施。

### 1.1 字符串与文本处理

| 文件 | 功能 |
|------|------|
| string-util.c | 字符串操作（startswith、endswith、strjoin、truncate 等） |
| strv.c | 字符串数组（NULL-terminated string array）操作 |
| utf8.c | UTF-8 编码验证与转换 |
| escape.c | 字符串转义/反转义（shell、C 风格等） |
| extract-word.c | 从字符串中逐词提取（配置文件解析基础） |
| string-table.c | 字符串 ↔ 枚举值映射表 |
| strbuf.c | 去重字符串缓冲区 |

**设计模式**：`strv` 是 systemd 最核心的数据类型之一，贯穿整个代码库。大量配置选项（如 `Environment=`）解析为 `strv`。

### 1.2 文件与文件系统

| 文件 | 功能 |
|------|------|
| fileio.c | 文件读写（read_one_line_file、write_string_file、read_full_file 等） |
| fs-util.c | 文件系统工具（touch、chmod、unlink、rename 等） |
| path-util.c | 路径处理（path_simplify、path_make_absolute、path_equal 等） |
| path-lookup.c | 单元文件搜索路径 |
| tmpfile-util.c | 临时文件管理 |
| mkdir.c | 递归目录创建 |
| glob-util.c | glob 模式匹配 |
| chase-symlinks.c | 安全的符号链接解析（防止符号链接攻击） |
| stat-util.c | stat 相关工具 |

### 1.3 进程与信号

| 文件 | 功能 |
|------|------|
| process-util.c | 进程操作（get_process_comm、pid_is_alive、wait_for_terminate 等） |
| signal-util.c | 信号处理（block/unblock、signal_to_string 等） |
| fd-util.c | 文件描述符管理（close_many、safe_close、fd_nonblock 等） |
| io-util.c | I/O 工具（loop_read、loop_write） |
| terminal-util.c | 终端操作 |

### 1.4 数据结构

| 文件 | 功能 |
|------|------|
| hashmap.c | 哈希表（systemd 最核心的容器，支持 ordered hashmap） |
| ordered-set.c | 有序集合 |
| prioq.c | 优先队列（用于定时器管理） |
| bitmap.c | 位图 |
| list.h | 侵入式双向链表宏（LIST_HEAD、LIST_FIELDS、LIST_PREPEND 等） |
| sort-util.c | 排序工具 |

**关键设计**：`hashmap.c` 是 systemd 中使用最广泛的数据结构，Manager 中的 `units`、`jobs` 等核心集合都使用它。采用开放寻址法，支持多种变体（Hashmap、OrderedHashmap、Set）。

### 1.5 哈希与随机

| 文件 | 功能 |
|------|------|
| siphash24.c | SipHash 哈希算法（hashmap 的哈希函数） |
| MurmurHash2.c | MurmurHash2 算法 |
| random-util.c | 随机数生成（getrandom、/dev/urandom） |

### 1.6 系统信息

| 文件 | 功能 |
|------|------|
| os-util.c | 操作系统信息（/etc/os-release 解析） |
| procfs-util.c | /proc 文件系统工具 |
| proc-cmdline.c | 内核命令行参数解析 |
| cgroup-util.c | cgroup 操作工具（路径解析、控制器查询等） |
| capability-util.c | Linux capabilities 操作 |
| login-util.c | 登录相关工具 |

### 1.7 时间与解析

| 文件 | 功能 |
|------|------|
| time-util.c | 时间操作（now()、format_timestamp、parse_time 等） |
| parse-util.c | 通用解析（safe_atou、parse_boolean、parse_size 等） |
| format-util.c | 格式化输出 |
| percent-util.c | 百分比解析 |
| ratelimit.c | 速率限制 |

### 1.8 网络与 Socket

| 文件 | 功能 |
|------|------|
| socket-util.c | Socket 操作（地址解析、选项设置等） |
| in-addr-util.c | IP 地址解析与格式化 |
| ether-addr-util.c | MAC 地址工具 |

---

## 二、src/shared/ — 共享代码库

`src/shared/` 包含依赖 systemd 概念但被多个组件共用的代码。

### 2.1 D-Bus 辅助

| 文件 | 功能 |
|------|------|
| bus-util.c | D-Bus 连接管理、错误处理、通用方法调用 |
| bus-unit-util.c | 单元相关的 D-Bus 操作辅助 |
| bus-message-util.c | D-Bus 消息构造辅助 |
| bus-wait-for-jobs.c | 等待 Job 完成（systemctl 等工具使用） |
| bus-map-properties.c | D-Bus 属性映射到 C 结构体 |
| bus-polkit.c | PolicyKit 鉴权集成 |

### 2.2 安全模块

| 文件 | 功能 |
|------|------|
| seccomp-util.c | seccomp 系统调用过滤封装 |
| selinux-util.c | SELinux 集成 |
| apparmor-util.c | AppArmor 集成 |
| smack-util.c | SMACK 安全模块 |
| tomoyo-util.c | TOMOYO 安全模块 |
| acl-util.c | POSIX ACL 操作 |

### 2.3 文件系统与镜像

| 文件 | 功能 |
|------|------|
| mount-util.c | 挂载操作封装 |
| fstab-util.c | fstab 解析 |
| rm-rf.c | 递归删除目录树 |
| copy.c | 文件/目录复制 |
| dissect-image.c | 磁盘镜像解剖（GPT 分区表解析） |
| discover-image.c | 镜像发现（/var/lib/machines 等） |

### 2.4 网络辅助

| 文件 | 功能 |
|------|------|
| netif-util.c | 网络接口信息查询 |
| netif-naming-scheme.c | 网卡命名方案 |
| bond-util.c | bonding 配置 |
| bridge-util.c | bridge 配置 |
| vlan-util.c | VLAN 配置 |
| macvlan-util.c | MACVLAN 配置 |
| ipvlan-util.c | IPVLAN 配置 |
| geneve-util.c | GENEVE 隧道配置 |

### 2.5 启动与系统管理

| 文件 | 功能 |
|------|------|
| bootspec.c | Boot Loader Spec 解析 |
| switch-root.c | 根文件系统切换（initrd → rootfs） |
| reboot-util.c | 重启工具 |
| boot-timestamps.c | 启动时间戳 |
| machine-id-setup.c | machine-id 初始化 |
| hostname-setup.c | 主机名设置 |

### 2.6 数据格式

| 文件 | 功能 |
|------|------|
| json.c | JSON 解析与生成 |
| varlink.c | Varlink 协议实现（systemd 的轻量级 IPC） |
| conf-parser.c | 通用 INI 风格配置解析 |

### 2.7 其他

| 文件 | 功能 |
|------|------|
| journal-importer.c | 日志导入 |
| journal-util.c | 日志辅助 |
| resolve-util.c | DNS 解析辅助 |
| userdb.c | 用户数据库查询 |
| coredump-util.c | coredump 处理辅助 |
| qrcode-util.c | 二维码生成（用于密钥恢复等） |
| condition.c | 条件评估（ConditionPathExists= 等） |
| killall.c | 批量 kill 进程 |
| exec-util.c | 执行辅助（运行目录中的脚本） |

---

## 三、库层依赖关系

```
src/fundamental/  ←  最底层（EFI 也可用）
       ↑
src/basic/        ←  基础 C 工具（不依赖 systemd 概念）
       ↑
src/shared/       ←  共享代码（依赖 systemd 概念）
       ↑
src/libsystemd/   ←  公共库 API
       ↑
各守护进程 / 工具   ←  最终应用
```

每一层只能依赖下层，不能反向依赖。这种严格的层次结构是 systemd 代码组织的重要设计原则。
