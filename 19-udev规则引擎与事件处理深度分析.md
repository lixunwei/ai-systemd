# 19. udev 规则引擎与事件处理深度分析

## 概述

udev 是 systemd 的设备管理子系统，负责在内核报告设备事件时动态创建/删除设备节点、
设置权限、创建符号链接、加载内核模块等。其核心由三大部分组成：

| 组件 | 源文件 | 职责 |
|------|--------|------|
| **守护进程** | `udevd.c` | 事件队列、Worker 进程池、信号处理 |
| **事件处理** | `udev-event.c` | 规则执行、设备节点更新、数据库维护 |
| **规则引擎** | `udev-rules.c` | 规则解析、Token 匹配、属性赋值 |
| **设备节点** | `udev-node.c` | /dev 节点创建、符号链接优先级栈 |
| **设备监视** | `udev-watch.c` | inotify 文件变化监视 |
| **控制协议** | `udev-ctrl.c` | udevadm ↔ udevd 通信 |
| **内建命令** | `udev-builtin*.c` | 11 个内建命令（blkid、kmod 等） |

---

## 一、守护进程架构（udevd.c）

### 1.1 核心数据结构

#### Manager 结构体（udevd.c:84-113）

```c
typedef struct Manager {
    sd_event *event;           // sd-event 事件循环
    Hashmap *workers;          // Worker 进程池（pid → Worker*）
    LIST_HEAD(Event, events);  // 事件队列（链表）
    const char *cgroup;        // cgroup 路径
    pid_t pid;                 // 主进程 PID

    int log_level;             // 日志级别
    UdevRules *rules;          // 已解析的规则集
    Hashmap *properties;       // 全局属性

    sd_netlink *rtnl;          // rtnetlink 连接（网络设备重命名）
    sd_device_monitor *monitor;// netlink 设备监视器
    UdevCtrl *ctrl;            // 控制套接字（udevadm 通信）
    int worker_watch[2];       // Worker 通信管道

    int inotify_fd;            // inotify 文件描述符
    sd_event_source *inotify_event;

    sd_event_source *kill_workers_event; // Worker 超时清理定时器
    usec_t last_usec;          // 上次事件时间

    bool stop_exec_queue;      // 暂停事件处理
    bool exit;                 // 退出标志
    pid_t pid_udevadm;         // udevadm control 进程 PID
} Manager;
```

#### Event 结构体（udevd.c:121-136）

```c
typedef enum EventState {
    EVENT_UNDEF,    // 未定义
    EVENT_QUEUED,   // 已入队，等待处理
    EVENT_RUNNING,  // 正在被 Worker 处理
} EventState;

typedef struct Event {
    Manager *manager;
    Worker *worker;           // 处理此事件的 Worker
    EventState state;

    sd_device *dev;           // 设备对象（可被规则修改）
    sd_device *dev_kernel;    // 内核原始设备对象（不可变副本）

    uint64_t seqnum;          // 内核 uevent 序号
    uint64_t blocker_seqnum;  // 阻塞此事件的最早序号

    sd_event_source *timeout_warning_event;  // 超时警告定时器
    sd_event_source *timeout_event;          // 超时杀死定时器

    LIST_FIELDS(Event, event);  // 链表节点
} Event;
```

#### Worker 结构体（udevd.c:138-152）

```c
typedef enum WorkerState {
    WORKER_UNDEF,    // 未定义
    WORKER_RUNNING,  // 正在处理事件
    WORKER_IDLE,     // 空闲，等待新事件
    WORKER_KILLED,   // 已被杀死（超时）
    WORKER_KILLING,  // 正在杀死（发送 SIGTERM）
} WorkerState;

typedef struct Worker {
    Manager *manager;
    pid_t pid;                    // Worker 进程 PID
    sd_device_monitor *monitor;   // 单播接收器
    WorkerState state;
    Event *event;                 // 当前处理的事件
} Worker;
```

### 1.2 启动流程

`run_udevd()`（udevd.c:1914-2016）→ `main_loop()`（udevd.c:1805-1911）：

```
run_udevd()
 ├── 解析配置文件 /etc/udev/udev.conf
 ├── 解析命令行参数
 ├── 检查 root 权限
 ├── 设置 cgroup
 ├── 检查 socket activation（sd_listen_fds）
 ├── manager_new() → 分配 Manager 结构
 ├── 可选 daemonize + fork
 └── main_loop()
      ├── 创建 worker_watch socketpair
      ├── udev_watch_restore() → 恢复 inotify 监视
      ├── sd_event_new() → 创建事件循环
      ├── udev_ctrl_start() → 启动控制套接字
      ├── sd_device_monitor_start() → 启动 netlink 监视
      ├── 注册 SIGINT/SIGTERM/SIGHUP/SIGCHLD 处理器
      ├── 注册 worker_watch IO 事件
      ├── 注册 POST 事件（event_queue_start）
      ├── udev_rules_load() → 加载所有规则文件
      ├── udev_rules_apply_static_dev_perms() → 应用静态权限
      └── sd_event_loop() → 进入主循环
```

### 1.3 事件队列与调度

#### 事件入队（udevd.c:980-1029）

```
kernel uevent
  → sd_device_monitor 接收
  → on_uevent() 回调
  → 克隆设备对象
  → 复制全局属性
  → Event 入队（LIST 尾部）
  → 触发 /run/udev/queue 标记文件
```

#### 事件调度（udevd.c:916-978）

`event_queue_start()` 作为 sd-event POST 回调，每次事件循环迭代后执行：

```
event_queue_start()
 ├── 检查 stop_exec_queue 标志
 ├── 检查规则/builtins 是否需要重新加载
 ├── 遍历事件队列：
 │    ├── 跳过 EVENT_RUNNING 事件
 │    ├── event_is_blocked() → 检查依赖关系
 │    └── event_run() → 分配 Worker 处理
 └── 返回
```

#### 事件阻塞检测（udevd.c:780-914）

`event_is_blocked()` 是 udev 的关键调度算法，确保**同一设备及其父/子设备的事件串行处理**：

```
event_is_blocked(event)
 ├── 优化：如果 blocker_seqnum == seqnum → 无阻塞（已检查过）
 ├── 优化：如果已知阻塞事件仍存在 → 直接返回阻塞
 ├── 获取自身设备信息：subsystem, devpath, devnum, ifindex, DEVPATH_OLD
 └── 遍历队列中序号更小的事件：
      ├── 匹配 major:minor（同一设备号） → 阻塞
      ├── 匹配 ifindex（同一网络接口） → 阻塞
      ├── 匹配 DEVPATH_OLD（设备改名前路径） → 阻塞
      ├── devpath 完全相同 → 阻塞（同一设备）
      ├── devpath 是前缀 + '/' → 阻塞（父/子设备）
      └── 无匹配 → 不阻塞
```

**阻塞规则总结**：

| 条件 | 含义 | 原因 |
|------|------|------|
| `devnum` 相同 | 同一块/字符设备 | 避免并发修改设备节点 |
| `ifindex` 相同 | 同一网络接口 | 避免并发重命名 |
| `DEVPATH_OLD` 匹配 | 设备 MOVE 事件 | 新旧路径关联 |
| devpath 相同 | 完全同一设备 | 序列化处理 |
| devpath 父子关系 | 父设备/子设备 | 确保父设备先完成 |

### 1.4 Worker 进程模型

#### Worker 生成（udevd.c:683-725）

```
worker_spawn(manager, event)
 ├── device_monitor_new_full() → 创建 Worker 专用监视器
 ├── device_monitor_allow_unicast_sender() → 只允许主进程发送
 ├── device_monitor_enable_receiving() → 启用接收
 ├── safe_fork(FORK_DEATHSIG) → fork 子进程
 │    └── 子进程：
 │         └── worker_main() → Worker 主函数
 └── 父进程：
      ├── worker_new() → 创建 Worker 对象
      └── worker_attach_event() → 绑定事件（设置超时定时器）
```

#### Worker 主循环（udevd.c:584-631）

```
worker_main(manager, monitor, first_device)
 ├── unsetenv("NOTIFY_SOCKET") → 不干扰 sd_notify
 ├── sigprocmask(SIGTERM) → 阻塞 SIGTERM
 ├── set_oom_score_adjust(0) → 重置 OOM 分数
 ├── manager_clear_for_worker() → 清理不需要的资源
 ├── sd_event_new() → 创建新事件循环
 ├── sd_event_add_signal(SIGTERM) → 注册退出处理
 ├── sd_device_monitor_attach_event() → 关联监视器
 ├── sd_device_monitor_start() → 启动监听
 ├── worker_device_monitor_handler() → 处理第一个设备
 └── sd_event_loop() → 等待更多设备（Worker 复用）
```

#### Worker 事件处理流程

```
worker_device_monitor_handler(monitor, dev, manager)
 ├── udev_event_new() → 创建 UdevEvent
 ├── worker_mark_block_device_read_only() → 标记只读（如需）
 ├── udev_event_execute_rules() → 执行规则（核心）
 ├── udev_event_execute_run() → 执行 RUN 命令
 ├── udev_event_process_inotify_watch() → 处理 inotify 监视
 └── 发送 WorkerMessage → 通知主进程完成
```

#### Worker 完成通知（udevd.c:1053-1117）

Worker 通过 socketpair 发送空 `WorkerMessage`，主进程 `on_worker()` 回调：

```
on_worker()
 ├── 读取 WorkerMessage
 ├── 查找对应 Worker
 ├── 标记 Worker 为 IDLE
 ├── 解绑事件
 ├── 释放 Event
 └── 触发 event_queue_start()（POST 回调）
```

### 1.5 超时与信号处理

| 信号 | 处理函数 | 行为 |
|------|----------|------|
| SIGTERM/SIGINT | `on_sigterm()` | 设置 exit=true，清理退出 |
| SIGHUP | `on_sighup()` | 重新加载规则和配置 |
| SIGCHLD | `on_sigchld()` | 回收 Worker 进程 |

**事件超时机制**：
- **警告超时**：`event_timeout / 3` 后打印警告日志
- **杀死超时**：`event_timeout`（默认 180s）后发送 `arg_timeout_signal` 杀死 Worker
- Worker 被杀后标记为 `WORKER_KILLED`

---

## 二、事件处理流水线（udev-event.c）

### 2.1 UdevEvent 结构体（udev-event.h:21-48）

```c
typedef struct UdevEvent {
    sd_device *dev;            // 当前设备（可被规则修改）
    sd_device *dev_parent;     // 父设备（PARENTS 匹配用）
    sd_device *dev_db_clone;   // 数据库中的旧设备记录（克隆）
    char *name;                // 设备节点名称
    char *program_result;      // PROGRAM 命令输出结果

    mode_t mode;               // 设备节点权限
    uid_t uid;                 // 所有者 UID
    gid_t gid;                 // 所有者 GID

    OrderedHashmap *seclabel_list;  // 安全标签列表（SELinux/SMACK）
    OrderedHashmap *run_list;       // RUN 命令列表

    usec_t exec_delay_usec;    // RUN 命令延迟
    usec_t birth_usec;         // 事件创建时间
    sd_netlink *rtnl;          // rtnetlink 连接

    unsigned builtin_run;      // 已执行的 builtin 位掩码
    unsigned builtin_ret;      // builtin 返回状态位掩码

    UdevRuleEscapeType esc:8;  // 字符串转义模式
    bool inotify_watch;        // 是否启用 inotify 监视
    bool inotify_watch_final;  // inotify 设置是否为 final
    bool group_final;          // GROUP 是否为 final（:= 操作符）
    bool owner_final;          // OWNER 是否为 final
    bool mode_final;           // MODE 是否为 final
    bool name_final;           // NAME 是否为 final
    bool devlink_final;        // SYMLINK 是否为 final
    bool run_final;            // RUN 是否为 final
    bool log_level_was_debug;
    int default_log_level;
} UdevEvent;
```

**final 标志说明**：当规则使用 `:=`（ASSIGN_FINAL）操作符赋值时，对应字段的 `*_final`
标志被设置为 true，后续规则不再修改该字段。这实现了规则的"最终赋值"语义。

### 2.2 执行规则主函数（udev-event.c:1045-1118）

```
udev_event_execute_rules(event, inotify_fd, timeout_usec, timeout_signal, properties, rules)
 ├── sd_device_get_action() → 获取设备动作
 │
 ├── 【REMOVE 动作】→ event_execute_rules_on_remove()
 │    ├── device_read_db_internal(dev_db_clone) → 读取数据库
 │    ├── device_tag_index(remove) → 删除标签索引
 │    ├── device_delete_db() → 删除数据库
 │    ├── udev_watch_end() → 停止 inotify 监视
 │    ├── udev_rules_apply_to_event() → 执行规则
 │    └── udev_node_remove() → 删除设备节点和符号链接
 │
 ├── 【其他动作】（ADD/CHANGE/MOVE/BIND/UNBIND）
 │    ├── udev_watch_end() → 临时停止 inotify 监视
 │    ├── device_clone_with_db() → 克隆数据库中旧设备记录
 │    ├── copy_all_tags() → 复制旧标签
 │    ├── 【MOVE 动作】→ udev_event_on_move() → 清除 ID_RENAMING
 │    ├── udev_rules_apply_to_event() → 【核心】执行所有规则
 │    ├── rename_netif() → 重命名网络接口（如需）
 │    ├── update_devnode() → 创建/更新设备节点和符号链接
 │    ├── device_ensure_usec_initialized() → 设置初始化时间戳
 │    ├── device_tag_index(add) → 更新标签索引
 │    ├── device_update_db() → 更新数据库 /run/udev/data/
 │    └── device_set_is_initialized() → 标记设备已初始化
 └── 返回
```

### 2.3 RUN 命令执行（udev-event.c:1121-1150）

```c
void udev_event_execute_run(UdevEvent *event, usec_t timeout_usec, int timeout_signal) {
    ORDERED_HASHMAP_FOREACH_KEY(val, command, event->run_list) {
        UdevBuiltinCommand builtin_cmd = PTR_TO_UDEV_BUILTIN_CMD(val);

        if (builtin_cmd != _UDEV_BUILTIN_INVALID) {
            // 内建命令 → 直接调用
            udev_builtin_run(event->dev, &event->rtnl, builtin_cmd, command, false);
        } else {
            // 外部程序 → fork+exec
            if (event->exec_delay_usec > 0)
                usleep(event->exec_delay_usec);  // 延迟执行
            udev_event_spawn(event, timeout_usec, timeout_signal, ...);
        }
    }
}
```

**关键设计**：
- `run_list` 是 OrderedHashmap，键为命令字符串，值编码了是 builtin 还是外部命令
- 内建命令在 Worker 进程内直接调用，无 fork 开销
- 外部命令通过 `udev_event_spawn()` fork+exec，有超时保护

### 2.4 格式化字符串替换（udev-event.c:236-524）

规则中的 `%` 和 `$` 替换变量：

| 格式 | 短格式 | 含义 | 数据来源 |
|------|--------|------|----------|
| `$kernel` | `%k` | 内核设备名 | `sd_device_get_sysname()` |
| `$number` | `%n` | 内核设备编号 | `sd_device_get_sysnum()` |
| `$devpath` | `%p` | 设备路径 | `sd_device_get_devpath()` |
| `$id` | `%b` | 文件名部分 | 路径最后一段 |
| `$driver` | `%d` | 驱动名 | `sd_device_get_driver()` |
| `$attr{name}` | `%s{name}` | sysfs 属性值 | `sd_device_get_sysattr_value()` |
| `$env{key}` | `%E{key}` | 设备属性值 | `sd_device_get_property_value()` |
| `$major` | `%M` | 主设备号 | `sd_device_get_devnum()` |
| `$minor` | `%m` | 次设备号 | `sd_device_get_devnum()` |
| `$result` | `%c` | PROGRAM 输出 | `event->program_result` |
| `$parent` | `%P` | 父设备节点名 | 父设备的 `DEVNAME` |
| `$name` | `%D` | 当前设备名 | `event->name` |
| `$links` | `%L` | 符号链接列表 | `sd_device_get_devlink_*()` |
| `$root` | `%r` | /dev 根路径 | 固定值 |
| `$sys` | `%S` | /sys 路径 | 固定值 |
| `$devnode` | `%N` | 设备节点路径 | `/dev/` + name |

---

## 三、规则引擎（udev-rules.c）

### 3.1 规则数据结构

#### 四层链表结构（udev-rules.c:146-186）

```
UdevRules
 └── LIST: UdevRuleFile           ← 每个 .rules 文件
      ├── filename                ← 文件路径
      └── LIST: UdevRuleLine      ← 每条规则（一行）
           ├── line_number        ← 行号
           ├── type               ← 行类型标志（LINE_HAS_NAME 等）
           ├── label / goto_label ← LABEL/GOTO 目标
           ├── goto_line          ← GOTO 跳转指针（已解析）
           └── LIST: UdevRuleToken ← 每个键值对
                ├── type:8        ← Token 类型（40+种）
                ├── op:8          ← 操作符（==, !=, =, +=, -=, :=）
                ├── match_type:8  ← 匹配模式
                ├── attr_subst_type:7 ← 属性替换类型
                ├── value         ← 值字符串
                └── data          ← 附加数据（属性名等）
```

#### 操作符类型（udev-rules.c:40-49）

```c
typedef enum {
    OP_MATCH,        // == 匹配
    OP_NOMATCH,      // != 不匹配
    OP_ADD,          // += 追加
    OP_REMOVE,       // -= 删除
    OP_ASSIGN,       // =  赋值
    OP_ASSIGN_FINAL, // := 最终赋值（后续规则不再修改）
} UdevRuleOperatorType;
```

#### 匹配模式（udev-rules.c:51-60）

```c
typedef enum {
    MATCH_TYPE_EMPTY,            // 空字符串匹配
    MATCH_TYPE_PLAIN,            // 纯文本精确匹配
    MATCH_TYPE_PLAIN_WITH_EMPTY, // 纯文本或空（"|foo" 语法）
    MATCH_TYPE_GLOB,             // Shell 通配符匹配（?,*,[]）
    MATCH_TYPE_GLOB_WITH_EMPTY,  // 通配符或空
    MATCH_TYPE_SUBSYSTEM,        // "subsystem" / "bus" / "class" 等价
} UdevRuleMatchType;
```

#### Token 类型（udev-rules.c:70-131）

Token 分为两大类，以 `_TK_M_MAX` / `_TK_A_MIN` 分界：

**匹配类 Token（TK_M_*）**：

| Token | 规则键 | 匹配对象 |
|-------|--------|----------|
| `TK_M_ACTION` | `ACTION` | 设备动作（add/remove/change...） |
| `TK_M_DEVPATH` | `DEVPATH` | sysfs 设备路径 |
| `TK_M_KERNEL` | `KERNEL` | 内核设备名 |
| `TK_M_DEVLINK` | `DEVLINK` | 已有符号链接 |
| `TK_M_NAME` | `NAME` | 网络接口名 |
| `TK_M_ENV{key}` | `ENV{key}` | 设备属性 |
| `TK_M_CONST` | `CONST{key}` | 系统常量 |
| `TK_M_TAG` | `TAG` | 设备标签 |
| `TK_M_SUBSYSTEM` | `SUBSYSTEM` | 子系统名 |
| `TK_M_DRIVER` | `DRIVER` | 驱动名 |
| `TK_M_ATTR{name}` | `ATTR{name}` | sysfs 属性值 |
| `TK_M_SYSCTL{key}` | `SYSCTL{key}` | 内核参数 |
| `TK_M_PARENTS_*` | `ATTRS{...}` 等 | 父设备链匹配 |
| `TK_M_TEST` | `TEST` | 文件存在性测试 |
| `TK_M_PROGRAM` | `PROGRAM` | 执行外部程序 |
| `TK_M_IMPORT_*` | `IMPORT{...}` | 导入属性（6种来源） |
| `TK_M_RESULT` | `RESULT` | PROGRAM 输出结果 |

**赋值类 Token（TK_A_*）**：

| Token | 规则键 | 赋值目标 |
|-------|--------|----------|
| `TK_A_OWNER` / `TK_A_OWNER_ID` | `OWNER` | 设备节点所有者 |
| `TK_A_GROUP` / `TK_A_GROUP_ID` | `GROUP` | 设备节点用户组 |
| `TK_A_MODE` / `TK_A_MODE_ID` | `MODE` | 设备节点权限 |
| `TK_A_TAG` | `TAG` | 添加/删除标签 |
| `TK_A_SECLABEL` | `SECLABEL{type}` | 安全标签 |
| `TK_A_ENV{key}` | `ENV{key}` | 设置属性 |
| `TK_A_NAME` | `NAME` | 设备节点名 / 网络接口名 |
| `TK_A_DEVLINK` | `SYMLINK` | 符号链接 |
| `TK_A_ATTR{name}` | `ATTR{name}` | 写入 sysfs 属性 |
| `TK_A_SYSCTL{key}` | `SYSCTL{key}` | 写入内核参数 |
| `TK_A_RUN_BUILTIN` | `RUN{builtin}` | 执行内建命令 |
| `TK_A_RUN_PROGRAM` | `RUN{program}` | 执行外部程序 |
| `TK_A_OPTIONS_*` | `OPTIONS` | 各种选项 |

### 3.2 规则文件加载

#### 文件搜索路径

规则目录定义（udev-rules.c:38 + def.h:45-58）：

```
/etc/udev/rules.d           ← 最高优先级（管理员配置）
/run/udev/rules.d           ← 运行时覆盖
/usr/local/lib/udev/rules.d ← 本地安装
/usr/lib/udev/rules.d       ← 最低优先级（发行版默认）
```

**覆盖规则**：同名文件（basename 相同），前面目录的优先。
例如 `/etc/udev/rules.d/99-foo.rules` 覆盖 `/usr/lib/udev/rules.d/99-foo.rules`。

#### 文件排序

文件按字典序排列，因此编号前缀决定执行顺序：
- `10-*.rules` 先于 `50-*.rules` 先于 `99-*.rules`

#### 解析流程

```
udev_rules_load()
 └── conf_files_list_strv(RULES_DIRS) → 枚举并排序所有 .rules 文件
      └── 对每个文件：udev_rules_parse_file()（udev-rules.c:1183-1284）
           ├── 逐行读取
           ├── 跳过注释和空行
           ├── 处理续行符 '\'
           └── rule_add_line()（udev-rules.c:1084-1145）
                ├── 创建 UdevRuleLine
                ├── parse_line()（udev-rules.c:1010-1062）
                │    └── 循环调用 parse_token()（udev-rules.c:550-988）
                │         └── 识别键名 → 确定 Token 类型 → 解析值 → 创建 UdevRuleToken
                └── rule_resolve_goto()（udev-rules.c:1148-1181）
                     └── 解析 GOTO → LABEL 跳转指针
```

### 3.3 规则执行引擎

#### 主循环（udev-rules.c:2521-2545）

```c
int udev_rules_apply_to_event(UdevRules *rules, UdevEvent *event, ...) {
    LIST_FOREACH(rule_files, file, rules->rule_files) {      // 遍历每个文件
        LIST_FOREACH_SAFE(rule_lines, line, next_line, file->rule_lines) {  // 遍历每条规则
            udev_rule_apply_line_to_event(rules, event, ..., &next_line);
        }
    }
}
```

#### 单行规则执行

```
udev_rule_apply_line_to_event(rules, event, ...)
 ├── 遍历当前行的所有 Token：
 │    ├── 匹配 Token（TK_M_*）：
 │    │    ├── 匹配成功 → 继续下一个 Token
 │    │    └── 匹配失败 → 跳过整行（goto next）
 │    └── 赋值 Token（TK_A_*）：
 │         └── 执行赋值操作
 └── 处理 GOTO：
      └── 如果行有 goto_line → next_line = goto_line
```

**关键规则**：一行中所有匹配条件必须全部满足（AND 逻辑），才执行赋值操作。

#### 字符串匹配（udev-rules.c:1333-1382）

```c
static bool token_match_string(UdevRuleToken *token, const char *str) {
    switch (token->match_type) {
    case MATCH_TYPE_EMPTY:
        match = isempty(str);
        break;
    case MATCH_TYPE_SUBSYSTEM:
        match = STR_IN_SET(str, "subsystem", "class", "bus");
        break;
    case MATCH_TYPE_PLAIN:
        NULSTR_FOREACH(i, value)    // 多值匹配（用 "|" 分隔的值）
            if (streq(i, str)) { match = true; break; }
        break;
    case MATCH_TYPE_GLOB:
        NULSTR_FOREACH(i, value)
            if (fnmatch(i, str, 0) == 0) { match = true; break; }
        break;
    }
    return token->op == (match ? OP_MATCH : OP_NOMATCH);
}
```

**匹配细节**：
- 多值用 `|` 分隔，存为 NULSTR（以 `\0` 分隔的连续字符串）
- `MATCH_TYPE_PLAIN` 用 `streq()` 精确匹配
- `MATCH_TYPE_GLOB` 用 `fnmatch()` 通配符匹配
- `OP_NOMATCH` 反转匹配结果

#### 属性匹配（udev-rules.c:1384-1434）

```
token_match_attr(rules, token, dev, event)
 ├── 获取属性名（可能含格式替换）
 ├── 根据 attr_subst_type：
 │    ├── SUBST_TYPE_FORMAT → 先做 %/$ 替换，再读 sysfs
 │    ├── SUBST_TYPE_PLAIN → 直接读 sysfs 属性
 │    └── SUBST_TYPE_SUBSYS → [subsystem/kernel]attribute 格式
 ├── 可选去除尾部空白
 └── token_match_string() → 匹配属性值
```

#### GOTO/LABEL 机制

```
规则解析时：
  LABEL="my_label"  → line->label = "my_label"
  GOTO="my_label"   → line->goto_label = "my_label"

rule_resolve_goto()（解析完文件后）：
  遍历后续行，找到 label == goto_label 的行
  设置 goto_line 指针（O(1) 跳转）

执行时：
  如果行有 goto_line → next_line = goto_line → 跳过中间规则
```

#### IMPORT 机制（6种来源）

| 类型 | Token | 行为 |
|------|-------|------|
| `IMPORT{file}` | `TK_M_IMPORT_FILE` | 从文件读取 key=value 对 |
| `IMPORT{program}` | `TK_M_IMPORT_PROGRAM` | 执行程序，解析 stdout |
| `IMPORT{builtin}` | `TK_M_IMPORT_BUILTIN` | 执行内建命令 |
| `IMPORT{db}` | `TK_M_IMPORT_DB` | 从设备数据库导入属性 |
| `IMPORT{cmdline}` | `TK_M_IMPORT_CMDLINE` | 从内核命令行导入 |
| `IMPORT{parent}` | `TK_M_IMPORT_PARENT` | 从父设备导入属性 |

#### PARENTS 匹配

`ATTRS{...}`、`KERNELS`、`SUBSYSTEMS`、`DRIVERS` 使用设备树向上遍历：

```
从当前设备开始，沿 parent 链逐级向上：
  对每个祖先设备，尝试匹配所有 PARENTS_* Token
  如果某个祖先匹配所有条件 → 整组匹配成功
  如果到根设备都不匹配 → 整组匹配失败
```

### 3.4 规则执行完整示例

规则文件内容：
```
SUBSYSTEM=="block", KERNEL=="sd[a-z]*", ATTR{removable}=="1", \
    SYMLINK+="removable/%k", MODE="0660", GROUP="disk", \
    RUN+="/usr/local/bin/notify-removable %k"
```

执行流程：
```
Token 1: TK_M_SUBSYSTEM == "block"
  → sd_device_get_subsystem() → "block" → 匹配 ✓

Token 2: TK_M_KERNEL == "sd[a-z]*"
  → sd_device_get_sysname() → "sda" → fnmatch("sd[a-z]*", "sda") → 匹配 ✓

Token 3: TK_M_ATTR{removable} == "1"
  → sd_device_get_sysattr_value("removable") → "1" → 匹配 ✓

Token 4: TK_A_DEVLINK += "removable/%k"
  → 展开格式："%k" → "sda"
  → 添加符号链接 /dev/removable/sda

Token 5: TK_A_MODE_ID = 0660
  → event->mode = 0660

Token 6: TK_A_GROUP = "disk"
  → event->gid = lookup("disk")

Token 7: TK_A_RUN_PROGRAM += "/usr/local/bin/notify-removable %k"
  → 展开格式 → 加入 run_list
```

---

## 四、设备节点管理（udev-node.c）

### 4.1 符号链接优先级栈

udev 允许多个设备竞争同一符号链接名（如 `/dev/disk/by-id/xxx`）。通过
`/run/udev/links/` 目录实现优先级仲裁：

```
/run/udev/links/<escaped_link_name>/
 ├── <devpath1>    → 包含优先级信息
 ├── <devpath2>
 └── ...

link_update() 流程（udev-node.c:415-488）：
 ├── update_stack_directory() → 添加/删除本设备到链接栈
 ├── link_find_prioritized() → 从栈中选择最高优先级设备
 │    └── 比较 devlink_priority（OPTIONS 设置）
 ├── 如果栈为空 → 删除符号链接
 └── node_symlink() → 创建/更新符号链接（原子操作）
```

**原子更新**：`node_symlink()` 先创建临时符号链接，然后 `rename()` 替换，避免竞态。

**并发安全**：`link_update()` 使用 stat-before/stat-after 模式检测竞争修改，
最多重试 `LINK_UPDATE_MAX_RETRIES` 次，重试间添加随机延迟。

### 4.2 设备节点操作

```
udev_node_update()（udev-node.c:509-564）：
 ├── 删除旧符号链接（不在新列表中的）
 │    └── link_update(dev, old_link, add=false)
 ├── 创建/更新新符号链接
 │    └── link_update(dev, new_link, add=true)
 └── 创建 /dev/{block,char}/major:minor 持久链接

udev_node_remove()（udev-node.c:566-590）：
 ├── 删除所有符号链接
 │    └── link_update(dev, link, add=false)
 └── 删除 /dev/{block,char}/major:minor

udev_node_apply_permissions()（udev-node.c:593-718）：
 ├── open(devnode, O_PATH) → 打开设备节点（不实际访问）
 ├── fstat() → 验证设备号和类型
 ├── fchmod() → 设置权限
 ├── fchown() → 设置所有者
 ├── 设置 SELinux 安全标签
 ├── 设置 SMACK 标签
 └── 更新时间戳
```

---

## 五、设备监视（udev-watch.c）

### 5.1 inotify 机制

```
udev_watch_begin()（udev-watch.c:76-137）：
 ├── 获取设备节点路径
 ├── inotify_add_watch(IN_CLOSE_WRITE)
 │    → 监视设备节点的写关闭事件
 ├── device_set_watch_handle(wd)
 │    → 保存 watch 描述符到设备对象
 └── 创建 /run/udev/watch/<wd> → devpath 映射文件

udev_watch_end()（udev-watch.c:139-161）：
 ├── inotify_rm_watch()
 ├── 删除 /run/udev/watch/<wd>
 └── device_set_watch_handle(DEVICE_WATCH_HANDLE_UNSET)

udev_watch_restore()（udev-watch.c:22-74）：
 ├── rename /run/udev/watch → /run/udev/watch.old
 ├── 遍历 .old 目录：
 │    ├── 读取 devpath → 打开设备
 │    └── udev_watch_begin() → 重新建立监视
 └── 删除 .old 目录
```

**用途**：当设备文件被修改时（如固件更新、配置变更），inotify 通知 udevd
重新触发规则处理。规则中通过 `OPTIONS+="watch"` 启用。

---

## 六、控制协议（udev-ctrl.c）

### 6.1 通信协议

```c
typedef struct UdevCtrlMessageWire {
    char version[16];    // 版本字符串 "udev-" VERSION
    unsigned magic;      // 0xdead1dea（魔数）
    enum UdevCtrlMsgType type;
    union UdevCtrlMsgValue value;
} UdevCtrlMessageWire;
```

#### 消息类型

| 类型 | udevadm 命令 | 行为 |
|------|-------------|------|
| `SET_LOG_LEVEL` | `udevadm control --log-level=` | 设置日志级别 |
| `STOP_EXEC_QUEUE` | `udevadm control --stop-exec-queue` | 暂停事件处理 |
| `START_EXEC_QUEUE` | `udevadm control --start-exec-queue` | 恢复事件处理 |
| `RELOAD` | `udevadm control --reload` | 重新加载规则 |
| `SET_ENV` | `udevadm control --property=K=V` | 设置全局属性 |
| `SET_CHILDREN_MAX` | `udevadm control --children-max=N` | 设置最大 Worker 数 |
| `PING` | `udevadm control --ping` | 存活检测 |
| `EXIT` | `udevadm control --exit` | 退出守护进程 |

### 6.2 安全控制

```
udev_ctrl_connection_event_handler()：
 ├── recvmsg() + SCM_CREDENTIALS → 获取发送者凭证
 ├── 检查 ucred.uid == 0 → 必须是 root
 ├── 验证 magic == 0xdead1dea
 └── 调用回调处理消息
```

套接字路径：`/run/udev/control`（UNIX domain socket）

---

## 七、内建命令系统

### 7.1 UdevBuiltin 接口（udev-builtin.h:31-39）

```c
typedef struct UdevBuiltin {
    const char *name;     // 命令名
    int (*cmd)(...);      // 执行函数
    const char *help;     // 帮助文本
    int (*init)(void);    // 初始化（守护进程启动时调用）
    void (*exit)(void);   // 清理（守护进程退出时调用）
    bool (*validate)(void); // 检查是否需要重新初始化
    bool run_once;        // 每个事件只运行一次
} UdevBuiltin;
```

### 7.2 内建命令清单

| 枚举值 | 名称 | 源文件 | 功能 |
|--------|------|--------|------|
| `UDEV_BUILTIN_BLKID` | `blkid` | `udev-builtin-blkid.c` | 探测文件系统/分区类型（UUID、LABEL、TYPE） |
| `UDEV_BUILTIN_BTRFS` | `btrfs` | `udev-builtin-btrfs.c` | btrfs 卷管理（检测 ready 状态） |
| `UDEV_BUILTIN_HWDB` | `hwdb` | `udev-builtin-hwdb.c` | 硬件数据库查询（modalias 匹配） |
| `UDEV_BUILTIN_INPUT_ID` | `input_id` | `udev-builtin-input_id.c` | 输入设备分类（键盘/鼠标/触控板等） |
| `UDEV_BUILTIN_KEYBOARD` | `keyboard` | `udev-builtin-keyboard.c` | 键盘扫描码映射（自定义按键） |
| `UDEV_BUILTIN_KMOD` | `kmod` | `udev-builtin-kmod.c` | 内核模块加载（modprobe） |
| `UDEV_BUILTIN_NET_ID` | `net_id` | `udev-builtin-net_id.c` | 网络设备命名（可预测命名） |
| `UDEV_BUILTIN_NET_LINK` | `net_setup_link` | `udev-builtin-net_setup_link.c` | 网络链路配置（.link 文件） |
| `UDEV_BUILTIN_PATH_ID` | `path_id` | `udev-builtin-path_id.c` | 持久化设备路径（PCI/USB/SCSI 拓扑） |
| `UDEV_BUILTIN_USB_ID` | `usb_id` | `udev-builtin-usb_id.c` | USB 设备属性（VID/PID/序列号） |
| `UDEV_BUILTIN_UACCESS` | `uaccess` | `udev-builtin-uaccess.c` | 设备节点 ACL 管理（logind 联动） |

### 7.3 典型规则中的 builtin 使用

```bash
# 自动加载内核模块
ACTION=="add", SUBSYSTEM=="pci", RUN{builtin}+="kmod load $attr{modalias}"

# 文件系统探测
SUBSYSTEM=="block", IMPORT{builtin}="blkid"

# 可预测网络命名
SUBSYSTEM=="net", IMPORT{builtin}="net_id", IMPORT{builtin}="hwdb"

# USB 设备识别
SUBSYSTEM=="usb", IMPORT{builtin}="usb_id", IMPORT{builtin}="hwdb"

# 输入设备分类
SUBSYSTEM=="input", IMPORT{builtin}="input_id"
```

---

## 八、完整事件处理流程

### 8.1 端到端数据流

```
                          ┌─────────────────────────────────────────────────┐
                          │                  内核空间                       │
                          │  设备驱动 → kobject_uevent() → netlink 广播    │
                          └───────────────────────┬─────────────────────────┘
                                                  │ AF_NETLINK
                                                  ▼
                          ┌─────────────────────────────────────────────────┐
                          │              udevd 主进程                       │
                          │                                                 │
                          │  sd_device_monitor ──→ on_uevent()             │
                          │       │                                         │
                          │       ▼                                         │
                          │  Event 入队（events 链表）                      │
                          │       │                                         │
                          │       ▼                                         │
                          │  event_queue_start()（POST 回调）              │
                          │       │                                         │
                          │       ├── event_is_blocked() → 检查依赖        │
                          │       │                                         │
                          │       └── event_run()                           │
                          │            ├── 有空闲 Worker → 复用            │
                          │            └── 无空闲 → worker_spawn()         │
                          └──────────────────┬──────────────────────────────┘
                                             │ unicast netlink
                                             ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                            Worker 进程                                       │
│                                                                              │
│  worker_device_monitor_handler()                                            │
│    │                                                                         │
│    ├── 1. udev_event_execute_rules()                                        │
│    │       ├── 获取 ACTION                                                  │
│    │       ├── REMOVE → 删除数据库/标签/节点                                │
│    │       └── ADD/CHANGE/MOVE:                                             │
│    │            ├── 克隆数据库旧记录                                         │
│    │            ├── udev_rules_apply_to_event()                              │
│    │            │    └── 遍历所有规则文件 × 所有行 × 所有 Token             │
│    │            │         ├── TK_M_* → 匹配检查                             │
│    │            │         ├── TK_M_IMPORT_BUILTIN → 执行内建+导入属性        │
│    │            │         ├── TK_M_PROGRAM → fork+exec 外部程序             │
│    │            │         ├── TK_A_ENV{} → 设置属性                         │
│    │            │         ├── TK_A_NAME → 设置设备名                        │
│    │            │         ├── TK_A_DEVLINK → 添加符号链接                    │
│    │            │         ├── TK_A_OWNER/GROUP/MODE → 设置权限              │
│    │            │         └── TK_A_RUN_* → 加入 run_list                    │
│    │            ├── rename_netif() → 重命名网络接口                          │
│    │            ├── update_devnode() → 创建/更新 /dev 节点                   │
│    │            │    ├── udev_node_update() → 符号链接管理                   │
│    │            │    └── udev_node_apply_permissions() → 权限/SELinux        │
│    │            ├── device_tag_index() → 更新 /run/udev/tag/               │
│    │            └── device_update_db() → 更新 /run/udev/data/              │
│    │                                                                         │
│    ├── 2. udev_event_execute_run()                                          │
│    │       └── 遍历 run_list:                                               │
│    │            ├── builtin → udev_builtin_run()（进程内直接调用）           │
│    │            └── program → udev_event_spawn()（fork+exec）               │
│    │                                                                         │
│    ├── 3. udev_event_process_inotify_watch()                                │
│    │       └── 根据规则决定是否启用 inotify 监视                             │
│    │                                                                         │
│    └── 4. 发送 WorkerMessage → 通知主进程完成                               │
└──────────────────────────────────────────────────────────────────────────────┘
                                             │
                                             ▼
                          ┌─────────────────────────────────────────────────┐
                          │  主进程 on_worker()                             │
                          │    ├── Worker → IDLE                           │
                          │    ├── Event → 释放                            │
                          │    └── 触发下一轮 event_queue_start()          │
                          └─────────────────────────────────────────────────┘
```

### 8.2 运行时文件布局

```
/run/udev/
 ├── control              ← udevadm 控制套接字
 ├── queue                ← 队列非空时存在（settle 等待用）
 ├── data/
 │    ├── b8:0            ← 块设备数据库（b=block, major:minor）
 │    ├── c189:0          ← 字符设备数据库（c=char）
 │    └── n3              ← 网络设备数据库（n=net, ifindex）
 ├── tags/
 │    ├── systemd/        ← 标签索引目录
 │    │    ├── b8:0       ← 带此标签的设备
 │    │    └── ...
 │    └── uaccess/
 ├── links/
 │    ├── <escaped_name>/ ← 符号链接优先级栈
 │    │    ├── <devpath1>
 │    │    └── <devpath2>
 │    └── ...
 └── watch/
      ├── <wd1> → devpath ← inotify watch 映射
      └── ...
```

---

## 九、设计要点总结

### 9.1 性能优化

| 策略 | 实现 |
|------|------|
| **Worker 进程池** | 空闲 Worker 复用，避免重复 fork 开销 |
| **事件阻塞优化** | `blocker_seqnum` 缓存，避免重复扫描 |
| **内建命令** | 在进程内执行，无 fork+exec 开销 |
| **Token 编译** | 规则解析为内存结构，执行时无字符串解析 |
| **GOTO 跳转** | 跳过不相关规则段，减少迭代 |
| **run_once 标志** | 每个事件只执行一次 builtin（位掩码跟踪） |
| **匹配类型预判** | 解析时确定 PLAIN/GLOB，运行时选择快速路径 |

### 9.2 安全设计

| 机制 | 说明 |
|------|------|
| **Worker 隔离** | 每个 Worker 是独立进程，崩溃不影响主进程 |
| **超时杀死** | 默认 180s 超时，防止 Worker 挂死 |
| **OOM 保护** | 主进程 OOM score=-1000，Worker 重置为 0 |
| **控制权限** | 控制套接字通过 `SCM_CREDENTIALS` 验证 root |
| **ASSIGN_FINAL** | `:=` 操作符防止后续规则覆盖关键设置 |
| **设备号验证** | 设置权限前验证设备号，防止 TOCTOU 攻击 |

### 9.3 可靠性设计

| 机制 | 说明 |
|------|------|
| **序列号排序** | 内核 uevent seqnum 保证事件顺序 |
| **父子设备串行** | event_is_blocked() 确保父设备先处理完 |
| **原子符号链接** | temp-symlink + rename 保证原子性 |
| **竞争重试** | 符号链接更新有随机延迟重试机制 |
| **watch 恢复** | 守护进程重启后自动恢复 inotify 监视 |
| **数据库持久化** | /run/udev/data/ 存储设备状态，重启可恢复 |
