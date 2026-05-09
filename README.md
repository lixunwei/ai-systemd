# systemd v250 源码深度分析 — 概要与归档说明

> 分析完成日期：2025-07-14
> 源码版本：systemd 250（LGPL v2.1+）
> 文档语言：中文
> 文档总数：23 篇（00-22）
> 文档总量：约 434KB / 11,779 行

---

## 一、项目总览

systemd 是 Linux 系统的核心初始化系统和服务管理器。本分析系列从源码层面深入剖析了 systemd v250 的架构设计、核心算法和实现细节，覆盖了 **1245 个 C 源文件**和 **897 个头文件**中的关键路径。

### 分析方法

- **索引驱动**：使用 zoekt（全文搜索）、Universal Ctags（符号索引）、cscope（调用图）建立源码索引
- **结构化分析**：先广度扫描架构，再深度潜入关键子系统
- **源码引用**：所有结论标注具体源文件和行号

---

## 二、文档分类与层次

### 第一层：架构概览（4 篇，~36KB）

| 编号 | 文档 | 核心内容 |
|------|------|----------|
| 00 | 索引与总览 | 项目指标、索引工具使用、文档目录、核心结构体速查 |
| 01 | 架构总览 | 四层架构（内核→基础库→公共库→应用层）、6大设计模式、构建系统 |
| 02 | 核心层-PID1 | PID 1 初始化流程、Manager 结构体、Unit/Job/Execute 子系统 |
| 11 | 其他子系统 | homed、oomd、coredump、boot、sleep 等辅助子系统 |

### 第二层：子系统分析（8 篇，~72KB）

| 编号 | 文档 | 核心内容 |
|------|------|----------|
| 03 | 公共库 libsystemd | sd-bus、sd-event、sd-journal、sd-device、sd-daemon API |
| 04 | 基础库 basic/shared | 字符串、哈希表、路径、日志等基础工具 |
| 05 | 日志系统 journald | journald 守护进程架构 |
| 06 | 登录管理 logind | Session/Seat/User 模型 |
| 07 | 设备管理 udevd | 事件处理、规则引擎概览 |
| 08 | 系统控制 systemctl | 命令结构、Job 等待机制 |
| 09 | 网络子系统 | networkd、resolved、timesyncd |
| 10 | 容器与虚拟化 | nspawn、machined、portabled |

### 第三层：深度分析（11 篇，~325KB）

| 编号 | 文档 | 核心内容 | 关键发现 |
|------|------|----------|----------|
| 12 | Unit-Job 依赖 | 30种依赖类型、36种原子行为、事务10步算法 | UnitDependencyAtom 是 v2 依赖系统的核心 |
| 13 | 事务引擎 | 循环检测 DFS、Job 合并矩阵、失败传播 | 合并矩阵公式 (a-1)*a/2+b |
| 14 | Socket 激活 | 14状态机、FD 传递、Accept 模式 | FD 从 SD_LISTEN_FDS_START=3 开始编号 |
| 15 | Unit 状态机 | 11种 Unit 类型的状态机、SIGCHLD 驱动 | unit_notify() 11步状态变更处理 |
| 16 | sd-event 事件循环 | 13种事件源、三段式循环、定时器合并 | sleep_between() 4级对齐 |
| 17 | 执行环境 | exec_child() 33步、7种命名空间、seccomp | 1070行的 exec_child() |
| 18 | D-Bus 通信 | sd-bus 结构、8状态连接机、vtable 分发 | 5步消息分发链 |
| 19 | udev 规则引擎 | Worker 进程池、40+种 Token、符号链接优先级 | 四层链表：Rules→File→Line→Token |
| 20 | journald 存储格式 | 256字节 Header、7种 Object、mmap 缓存 | FSPRG 前向安全密封 |
| 21 | networkd 网络配置 | Link 7状态机、37种 NetDev、异步请求队列 | link_check_ready() 20+检查条件 |
| 22 | cgroup 资源管理 | 13种控制器、CGroupContext、v1/v2 兼容、BPF | 四层掩码架构 |

---

## 三、关键技术发现摘要

### 3.1 PID 1 核心机制

- **事务引擎**：10步算法处理 Unit 启动/停止，含循环检测（DFS + generation marker）和 Job 合并（下三角矩阵）
- **依赖系统**：30种 UnitDependency 分解为 36种 UnitDependencyAtom 原子行为，简化了运行时查询
- **状态通知**：`unit_notify()` 是中央 11步状态变更处理器，驱动所有副作用

### 3.2 执行沙箱

- **exec_child()**：1070行、33步执行环境构建，涵盖 cgroup、命名空间、挂载、seccomp、MAC 策略
- **7种命名空间**：Mount、User、PID、Network、IPC、UTS、cgroup
- **seccomp 过滤**：14层系统调用过滤，seccomp 作为最后一步应用

### 3.3 事件驱动架构

- **sd-event**：13种事件源类型，prepare/wait/dispatch 三段式循环，优先级阈值防饥饿
- **纯异步设计**：所有 Manager 守护进程共享同一事件循环模型

### 3.4 存储与通信

- **journald**：自定义二进制格式，256字节 Header，哈希去重，mmap 8MiB 窗口缓存，FSPRG 前向安全密封
- **sd-bus**：用户空间 D-Bus 实现，8状态连接机，SASL EXTERNAL 认证，vtable 方法分发

### 3.5 设备与网络

- **udev**：fork 型 Worker 进程池，事件阻塞算法（devnum/ifindex/devpath），四层链表规则引擎
- **networkd**：纯异步 rtnetlink 驱动，Link 7状态机，37种虚拟设备类型，消息计数器模式

### 3.6 资源管理

- **cgroup**：透明 v1/v2 兼容，四层掩码计算，BPF 替代 v2 devices 控制器
- **Slice 层次**：名称编码层次关系，system/user/machine 三大默认 slice

---

## 四、文档统计

| 分类 | 篇数 | 大小 | 占比 |
|------|------|------|------|
| 架构概览 | 4 | ~36KB | 8% |
| 子系统分析 | 8 | ~72KB | 17% |
| 深度分析 | 11 | ~325KB | 75% |
| **总计** | **23** | **~434KB** | **100%** |

---

## 五、文件清单

```
darren/
├── README.md                                          ← 本文件（概要与归档说明）
├── 00-索引与总览.md                                    ← 主索引（7KB）
├── 01-架构总览.md                                      ← 架构层次（9KB）
├── 02-核心层-PID1.md                                   ← PID 1 核心（12KB）
├── 03-公共库-libsystemd.md                             ← 公共 API（7KB）
├── 04-基础库-basic与shared.md                          ← 基础工具库（7KB）
├── 05-日志系统-journald.md                             ← journald 概览（8KB）
├── 06-登录管理-logind.md                               ← logind 概览（8KB）
├── 07-设备管理-udevd.md                                ← udevd 概览（8KB）
├── 08-系统控制-systemctl.md                            ← systemctl 分析（6KB）
├── 09-网络子系统.md                                    ← 网络三组件（11KB）
├── 10-容器与虚拟化.md                                  ← nspawn/machined（8KB）
├── 11-其他子系统.md                                    ← 辅助子系统（9KB）
├── 12-Unit-Job依赖与执行顺序.md                        ← 依赖与事务（26KB）
├── 13-事务引擎内部实现深度分析.md                       ← 事务引擎（20KB）
├── 14-Socket-Activation机制深度分析.md                  ← Socket 激活（26KB）
├── 15-Unit状态机深度分析.md                             ← 状态机（30KB）
├── 16-sd-event事件循环深度分析.md                       ← 事件循环（24KB）
├── 17-执行环境深度分析.md                               ← 执行沙箱（27KB）
├── 18-D-Bus通信深度分析.md                              ← D-Bus 通信（27KB）
├── 19-udev规则引擎与事件处理深度分析.md                  ← udev 深度（41KB）
├── 20-journald存储格式与写入路径深度分析.md              ← journald 深度（44KB）
├── 21-systemd-networkd网络配置与状态机深度分析.md        ← networkd 深度（34KB）
└── 22-cgroup资源管理与slice层次结构深度分析.md           ← cgroup 深度（35KB）
```

---

## 六、使用建议

### 阅读顺序

1. **入门**：00 → 01 → 02（理解整体架构）
2. **子系统**：根据兴趣选读 03-11
3. **深度**：12-13（依赖系统）→ 15（状态机）→ 17（执行环境）→ 22（cgroup）

### 检索方式

- 通过 `00-索引与总览.md` 的目录和速查表快速定位
- 本文件（README.md）提供分类视图和关键发现摘要
- 每篇文档内部有详细的源码行号引用，可直接对照源码阅读

---

*本分析系列使用 AI 辅助工具基于 systemd v250 源码生成，所有结论均标注具体源文件和行号。*
