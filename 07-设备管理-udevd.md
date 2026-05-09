# 设备管理分析 — systemd-udevd

## 一、概述

`systemd-udevd` 是 Linux 用户空间设备管理器，负责：
- 接收内核设备事件（uevent）
- 执行匹配规则
- 创建/修改设备节点（/dev/）
- 设置设备权限和属性
- 触发依赖的系统操作

源码位于 `src/udev/`。

---

## 二、核心架构

```
┌─────────────┐
│  Linux 内核  │
│  kobject     │
└──────┬──────┘
       │ uevent (netlink)
       ▼
┌──────────────────────────────────────┐
│          systemd-udevd               │
│                                      │
│  Manager                             │
│  ├── Event Queue (事件队列)           │
│  ├── Worker Pool (工作进程池)         │
│  ├── Rules (规则集合)                 │
│  └── Control Socket (控制套接字)      │
│                                      │
│  处理流程:                            │
│  1. 接收 uevent                      │
│  2. 匹配 rules                       │
│  3. fork worker 执行                  │
│  4. 创建/修改 /dev/ 节点              │
└──────────────────────────────────────┘
```

---

## 三、关键源文件

| 文件 | 职责 |
|------|------|
| udevd.c | 守护进程核心（Manager、Event、Worker 管理） |
| udev-rules.c | 规则解析与匹配引擎 |
| udev-event.c | 设备事件执行逻辑 |
| udev-node.c | 设备节点创建与管理 |
| udev-netlink.c | 内核 uevent 接收 |
| udev-ctrl.c | 控制套接字（udevadm 通信） |
| udev-watch.c | inotify 设备节点监控 |

### 内置插件（builtin）

| 文件 | 功能 |
|------|------|
| udev-builtin-hwdb.c | 硬件数据库查询 |
| udev-builtin-net_id.c | 网卡可预测命名（enp0s3 等） |
| udev-builtin-usb_id.c | USB 设备 ID 生成 |
| udev-builtin-blkid.c | 块设备 ID 探测（文件系统类型等） |
| udev-builtin-input_id.c | 输入设备 ID |
| udev-builtin-kmod.c | 内核模块加载 |
| udev-builtin-path_id.c | 设备路径 ID |
| udev-builtin-net_setup_link.c | 网卡链路配置 |

---

## 四、核心数据结构

### 4.1 Manager（udevd.c）

```c
// udevd.c 中的管理器结构（简化）
typedef struct Manager {
    sd_event *event;              // 事件循环
    sd_netlink *rtnl;             // netlink 连接

    // === 规则 ===
    UdevRules *rules;             // 已加载的规则集

    // === 工作进程 ===
    LIST_HEAD(Worker, workers);   // 工作进程列表
    unsigned children_count;       // 当前工作进程数
    unsigned children_max;         // 最大工作进程数

    // === 事件队列 ===
    struct udev_list_node events;  // 待处理事件队列

    // === 控制 ===
    int fd_ctrl;                   // 控制套接字
    int fd_inotify;                // inotify 监控

    // === 配置 ===
    usec_t event_timeout_usec;     // 事件超时
    int exec_delay_usec;           // 执行延迟
    ...
} Manager;
```

### 4.2 Event

```c
typedef struct Event {
    Manager *manager;
    struct udev_device *dev;       // 关联设备
    enum event_state state;        // 事件状态
    Worker *worker;                // 处理此事件的工作进程
    ...
} Event;
```

### 4.3 Worker

```c
typedef struct Worker {
    Manager *manager;
    pid_t pid;                     // 工作进程 PID
    Event *event;                  // 正在处理的事件
    enum worker_state state;       // 工作进程状态
    ...
} Worker;
```

---

## 五、事件处理流程

### 5.1 事件接收

```
内核产生 uevent
  ↓
udev-netlink.c 通过 netlink socket 接收
  ↓
创建 Event 对象，加入事件队列
  ↓
manager_dispatch_event() 分发事件
```

### 5.2 规则匹配

```
udev-rules.c → udev_rules_apply_to_event()
  ├── 遍历已加载的规则
  ├── 匹配条件检查
  │     KERNEL=="sd[a-z]"        匹配内核名
  │     SUBSYSTEM=="block"       匹配子系统
  │     ATTR{size}=="..."        匹配 sysfs 属性
  │     ENV{...}=="..."          匹配环境变量
  │     DRIVERS=="..."           匹配驱动
  │     ACTION=="add"            匹配操作类型
  ├── 执行赋值操作
  │     NAME="..."               设置设备名
  │     SYMLINK+="..."           添加符号链接
  │     OWNER="..."              设置所有者
  │     GROUP="..."              设置组
  │     MODE="..."               设置权限
  │     ENV{...}="..."           设置环境变量
  │     TAG+="..."               添加标签
  │     RUN+="..."               添加后置命令
  └── 执行内置命令
       IMPORT{builtin}="..."    调用内置插件
       IMPORT{program}="..."    调用外部程序
```

### 5.3 工作进程执行

```
manager_dispatch_event()
  ├── 检查设备锁（同一设备的事件串行化）
  ├── 从工作进程池分配 Worker
  │     如果无空闲 Worker 且未达上限 → fork 新 Worker
  ├── Worker 进程中：
  │     ├── udev_event_execute_rules() — 执行规则
  │     ├── udev_node_add() — 创建设备节点
  │     ├── udev_node_update() — 更新符号链接
  │     └── 执行 RUN 命令
  └── Worker 完成 → 报告结果给 Manager
```

---

## 六、设备节点管理

### 6.1 节点创建（udev-node.c）

```
设备 ADD 事件
  ↓
udev_node_add()
  ├── mknod() — 创建设备节点
  ├── chmod/chown — 设置权限
  ├── setfattr — 设置扩展属性
  └── 创建符号链接
       /dev/disk/by-id/...
       /dev/disk/by-uuid/...
       /dev/disk/by-path/...
       /dev/disk/by-label/...
```

### 6.2 符号链接

udev 维护一系列符号链接目录，提供稳定的设备引用：

| 目录 | 用途 |
|------|------|
| /dev/disk/by-id/ | 按硬件 ID |
| /dev/disk/by-uuid/ | 按文件系统 UUID |
| /dev/disk/by-path/ | 按硬件路径 |
| /dev/disk/by-label/ | 按文件系统标签 |
| /dev/input/by-id/ | 输入设备按 ID |
| /dev/input/by-path/ | 输入设备按路径 |

---

## 七、规则文件格式

规则文件位于以下目录（按优先级）：

1. `/etc/udev/rules.d/` — 管理员自定义
2. `/run/udev/rules.d/` — 运行时
3. `/usr/lib/udev/rules.d/` — 系统默认

### 规则语法

```udev
# 条件匹配 + 赋值操作
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", \
    ATTR{type}=="1", KERNEL=="eth*", \
    NAME="net0"

# 内置插件调用
SUBSYSTEM=="usb", IMPORT{builtin}="usb_id"

# 运行外部程序
KERNEL=="sd[a-z]", RUN+="/usr/bin/my-script %k"

# 标签设置（用于 logind 设备管理）
SUBSYSTEM=="drm", TAG+="seat", TAG+="uaccess"
```

---

## 八、与其他组件的交互

| 组件 | 交互方式 | 内容 |
|------|---------|------|
| PID 1 | device 单元 | 设备变更触发 device 单元状态更新 |
| logind | TAG="seat" | 设备分配到座位 |
| logind | TAG="uaccess" | 设备 ACL 权限管理 |
| networkd | net 设备事件 | 网卡出现/消失触发网络配置 |
| homed | block 设备事件 | LUKS 设备触发家目录激活 |
| udevadm | 控制套接字 | 管理命令（trigger、settle、info） |
| hwdb | 硬件数据库 | 设备属性查询（厂商/型号等） |

---

## 九、udevadm 工具

```bash
udevadm info /dev/sda        # 查看设备信息
udevadm trigger              # 重新触发所有设备事件
udevadm settle                # 等待所有事件处理完成
udevadm monitor               # 实时监控设备事件
udevadm test /sys/class/...  # 测试规则匹配
udevadm control --reload     # 重新加载规则
```
