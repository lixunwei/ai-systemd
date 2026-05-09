# 22. cgroup 资源管理与 Slice 层次结构深度分析

## 1. 概述

systemd 通过 Linux cgroup（control group）机制实现进程的**资源控制**、**资源审计**和**进程分组**。每个 Unit 在启动时会被放入对应的 cgroup 节点，通过 Slice 层次结构形成树形的资源管控拓扑。

systemd 同时支持 cgroup v1（legacy）、cgroup v2（unified）和混合（hybrid）三种模式，内部通过控制器掩码（`CGroupMask`）统一抽象不同版本的差异。

### 核心源文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `cgroup.h` | 337 | CGroupContext 结构体、所有枚举与函数声明 |
| `cgroup.c` | ~3500 | 核心实现：context_apply、realize、manager_setup、资源审计 |
| `cgroup-util.h` | 310 | CGroupController 枚举、CGroupMask 位掩码、路径工具 |
| `cgroup-util.c` | ~2800 | cg_create/attach/kill、unified 检测、路径操作 |
| `cgroup-setup.c` | ~400 | cg_create、cg_attach、unified/hybrid 模式判断 |
| `slice.c` / `slice.h` | 450/19 | Slice Unit 实现：2状态机、父子层次、冻结器 |
| `scope.c` | ~500 | Scope Unit 的 cgroup 管理（外部进程组） |
| `bpf-devices.c` | ~300 | BPF 设备访问控制（cgroup v2） |
| `bpf-firewall.c` | ~700 | BPF IP 防火墙（IPAddressAllow/Deny） |
| `dbus-cgroup.c` | ~750 | 资源控制属性的 D-Bus 暴露与运行时修改 |

---

## 2. CGroupController 枚举与掩码体系

### 2.1 控制器枚举（cgroup-util.h:20-43）

```c
typedef enum CGroupController {
    /* 真实内核控制器 */
    CGROUP_CONTROLLER_CPU,         // cpu (v1+v2)
    CGROUP_CONTROLLER_CPUACCT,     // cpuacct (v1 only)
    CGROUP_CONTROLLER_CPUSET,      // cpuset (v2 only)
    CGROUP_CONTROLLER_IO,          // io (v2 only)
    CGROUP_CONTROLLER_BLKIO,       // blkio (v1 only)
    CGROUP_CONTROLLER_MEMORY,      // memory (v1+v2)
    CGROUP_CONTROLLER_DEVICES,     // devices (v1 only)
    CGROUP_CONTROLLER_PIDS,        // pids (v1+v2)

    /* BPF 伪控制器 (v2 only) */
    CGROUP_CONTROLLER_BPF_FIREWALL,                  // IP 防火墙
    CGROUP_CONTROLLER_BPF_DEVICES,                   // 设备访问 BPF
    CGROUP_CONTROLLER_BPF_FOREIGN,                   // 外部 BPF 程序
    CGROUP_CONTROLLER_BPF_SOCKET_BIND,               // Socket 绑定限制
    CGROUP_CONTROLLER_BPF_RESTRICT_NETWORK_INTERFACES, // 网络接口限制
} CGroupController;
```

### 2.2 掩码分组

```c
// 位掩码转换公式
#define CGROUP_CONTROLLER_TO_MASK(c) (1U << (c))

// v1 真实控制器集合
CGROUP_MASK_V1 = CPU | CPUACCT | BLKIO | MEMORY | DEVICES | PIDS

// v2 真实控制器集合
CGROUP_MASK_V2 = CPU | CPUSET | IO | MEMORY | PIDS

// BPF 伪控制器集合
CGROUP_MASK_BPF = BPF_FIREWALL | BPF_DEVICES | BPF_FOREIGN
                | BPF_SOCKET_BIND | BPF_RESTRICT_NETWORK_INTERFACES
```

### 2.3 v1/v2 控制器对应关系

| v1 控制器 | v2 控制器 | 说明 |
|-----------|-----------|------|
| `cpu` + `cpuacct` | `cpu` | v1 总是联合挂载（EXTEND_JOINED） |
| `blkio` | `io` | systemd 自动在两者间转换权重 |
| `memory` | `memory` | v2 新增 min/low/high 层级化保护 |
| `devices` | BPF_DEVICES | v2 用 BPF 替代 devices 控制器 |
| `pids` | `pids` | 两版本接口一致 |
| —— | `cpuset` | v2 新增 |

### 2.4 Unified/Legacy/Hybrid 检测

```c
// cgroup-setup.c:82-155
cg_is_unified_wanted()   // 检查 /proc/cmdline: systemd.unified_cgroup_hierarchy=1
cg_is_hybrid_wanted()    // 检查 /proc/cmdline: systemd.legacy_systemd_cgroup_controller

// cgroup-util.c 运行时检测
cg_all_unified()           // 所有控制器是否都在 v2
cg_hybrid_unified()        // 是否混合模式
cg_unified_controller(c)   // 特定控制器是否在 v2
cg_unified_cached()        // 缓存的统一模式结果
```

---

## 3. CGroupContext 结构体（cgroup.h:113-197）

`CGroupContext` 是资源控制的核心数据结构，每个支持 cgroup 的 Unit（Service、Scope、Slice、Socket、Mount、Swap）都内嵌一份。它映射了 `.service`/`.slice` 配置文件中的所有资源控制指令。

### 3.1 完整字段表

#### 审计开关（Accounting）

| 字段 | 配置指令 | 说明 |
|------|----------|------|
| `cpu_accounting` | `CPUAccounting=` | 开启 CPU 使用统计 |
| `io_accounting` | `IOAccounting=` | 开启 IO 使用统计 |
| `blockio_accounting` | `BlockIOAccounting=` | 开启 BlockIO 统计（v1） |
| `memory_accounting` | `MemoryAccounting=` | 开启内存统计 |
| `tasks_accounting` | `TasksAccounting=` | 开启任务数统计 |
| `ip_accounting` | `IPAccounting=` | 开启 IP 流量统计 |

#### CPU 控制

| 字段 | 配置指令 | v2 内核文件 | v1 内核文件 |
|------|----------|-------------|-------------|
| `cpu_weight` | `CPUWeight=` | `cpu.weight` | — |
| `startup_cpu_weight` | `StartupCPUWeight=` | `cpu.weight`（启动阶段） | — |
| `cpu_quota_per_sec_usec` | `CPUQuota=` | `cpu.max` 的配额部分 | `cpu.cfs_quota_us` |
| `cpu_quota_period_usec` | `CPUQuotaPeriodSec=` | `cpu.max` 的周期部分 | `cpu.cfs_period_us` |
| `cpu_shares` | `CPUShares=` | — | `cpu.shares` |
| `startup_cpu_shares` | `StartupCPUShares=` | — | `cpu.shares`（启动阶段） |

#### cpuset 控制（v2 only）

| 字段 | 配置指令 | 内核文件 |
|------|----------|----------|
| `cpuset_cpus` | `AllowedCPUs=` | `cpuset.cpus` |
| `startup_cpuset_cpus` | `StartupAllowedCPUs=` | `cpuset.cpus` |
| `cpuset_mems` | `AllowedMemoryNodes=` | `cpuset.mems` |
| `startup_cpuset_mems` | `StartupAllowedMemoryNodes=` | `cpuset.mems` |

#### IO 控制

| 字段 | 配置指令 | v2 内核文件 | v1 内核文件 |
|------|----------|-------------|-------------|
| `io_weight` | `IOWeight=` | `io.weight` | — |
| `startup_io_weight` | `StartupIOWeight=` | `io.weight` | — |
| `io_device_weights` | `IODeviceWeight=` | `io.weight` per-device | — |
| `io_device_limits` | `IOReadBandwidthMax=` 等 | `io.max` | — |
| `io_device_latencies` | `IODeviceLatencyTargetSec=` | `io.latency` | — |
| `blockio_weight` | `BlockIOWeight=` | — | `blkio.weight` |
| `blockio_device_weights` | `BlockIODeviceWeight=` | — | `blkio.weight_device` |
| `blockio_device_bandwidths` | `BlockIOReadBandwidth=` 等 | — | `blkio.throttle.*` |

#### 内存控制

| 字段 | 配置指令 | v2 内核文件 | v1 内核文件 |
|------|----------|-------------|-------------|
| `memory_min` | `MemoryMin=` | `memory.min` | — |
| `memory_low` | `MemoryLow=` | `memory.low` | — |
| `memory_high` | `MemoryHigh=` | `memory.high` | — |
| `memory_max` | `MemoryMax=` | `memory.max` | — |
| `memory_swap_max` | `MemorySwapMax=` | `memory.swap.max` | — |
| `memory_limit` | `MemoryLimit=` | — | `memory.limit_in_bytes` |
| `memory_oom_group` | `OOMPolicy=` | `memory.oom.group` | — |
| `default_memory_min` | `DefaultMemoryMin=` | 子 cgroup 继承 | — |
| `default_memory_low` | `DefaultMemoryLow=` | 子 cgroup 继承 | — |

#### 任务数控制

| 字段 | 配置指令 | 内核文件 |
|------|----------|----------|
| `tasks_max` | `TasksMax=` | `pids.max`（根 cgroup 映射到 `/proc/sys/kernel/pid_max`） |

#### 设备访问

| 字段 | 配置指令 | 机制 |
|------|----------|------|
| `device_policy` | `DevicePolicy=` | AUTO / CLOSED / STRICT |
| `device_allow` | `DeviceAllow=` | v1: `devices.allow/deny`, v2: BPF |

#### 委托与禁用

| 字段 | 配置指令 | 说明 |
|------|----------|------|
| `delegate` | `Delegate=` | 允许子进程管理自己的 cgroup 子树 |
| `delegate_controllers` | `DelegateControllers=` | 指定委托哪些控制器 |
| `disable_controllers` | `DisableControllers=` | 在子树中禁用指定控制器 |

#### 网络/BPF

| 字段 | 配置指令 |
|------|----------|
| `ip_address_allow` | `IPAddressAllow=` |
| `ip_address_deny` | `IPAddressDeny=` |
| `ip_filters_ingress/egress` | `IPIngressFilterPath=` / `IPEgressFilterPath=` |
| `bpf_foreign_programs` | `BPFProgram=` |
| `socket_bind_allow/deny` | `SocketBindAllow=` / `SocketBindDeny=` |
| `restrict_network_interfaces` | `RestrictNetworkInterfaces=` |

#### OOMD 设置

| 字段 | 配置指令 |
|------|----------|
| `moom_swap` | `ManagedOOMSwap=` |
| `moom_mem_pressure` | `ManagedOOMMemoryPressure=` |
| `moom_mem_pressure_limit` | `ManagedOOMMemoryPressureLimit=` |
| `moom_preference` | `ManagedOOMPreference=` |

### 3.2 默认值初始化（cgroup.c:136-168）

```c
void cgroup_context_init(CGroupContext *c) {
    *c = (CGroupContext) {
        .cpu_weight              = CGROUP_WEIGHT_INVALID,      // 未设置
        .startup_cpu_weight      = CGROUP_WEIGHT_INVALID,
        .cpu_quota_per_sec_usec  = USEC_INFINITY,              // 无限制
        .cpu_quota_period_usec   = USEC_INFINITY,
        .cpu_shares              = CGROUP_CPU_SHARES_INVALID,  // v1 未设置
        .startup_cpu_shares      = CGROUP_CPU_SHARES_INVALID,
        .memory_high             = CGROUP_LIMIT_MAX,           // 无限制
        .memory_max              = CGROUP_LIMIT_MAX,
        .memory_swap_max         = CGROUP_LIMIT_MAX,
        .memory_limit            = CGROUP_LIMIT_MAX,           // v1 无限制
        .io_weight               = CGROUP_WEIGHT_INVALID,
        .startup_io_weight       = CGROUP_WEIGHT_INVALID,
        .blockio_weight          = CGROUP_BLKIO_WEIGHT_INVALID,
        .startup_blockio_weight  = CGROUP_BLKIO_WEIGHT_INVALID,
        .tasks_max               = TASKS_MAX_UNSET,
        .moom_swap               = MANAGED_OOM_AUTO,
        .moom_mem_pressure       = MANAGED_OOM_AUTO,
        .moom_preference         = MANAGED_OOM_PREFERENCE_NONE,
    };
}
```

所有字段使用 **INVALID/INFINITY/MAX** 哨兵值，表示"未配置"，写入内核时转换为对应的 `max` 字符串。

---

## 4. cgroup_context_apply()：控制器写入逻辑

### 4.1 函数签名与调用时机

`cgroup_context_apply()` 在 `cgroup.c:1340-1630` 实现，约 290 行。由 `unit_update_cgroup()` 在 cgroup 实现化（realize）后调用。

### 4.2 各控制器写入细节

#### CPU 控制器（1354-1384）

```
if (unified) {
    cpu.weight  ← cgroup_context_cpu_weight(c, state)
    cpu.max     ← "quota period" 或 "max period"
} else {  // legacy
    // 如果用户设了 CPUWeight=，自动转换为 cpu.shares
    cpu.shares  ← cgroup_cpu_weight_to_shares(weight)
    cpu.cfs_quota_us / cpu.cfs_period_us ← 配额
}
```

**兼容性转换**：`CPUWeight=` 在 v1 上自动转为 `CPUShares=`，转换函数 `cgroup_cpu_weight_to_shares()` 使用对数映射。

#### cpuset 控制器（1386-1389）

```
cpuset.cpus ← cgroup_context_allowed_cpus(c, state)
cpuset.mems ← cgroup_context_allowed_mems(c, state)
```

仅 v2，支持 Startup 变体（系统启动阶段使用 startup_ 版本）。

#### IO/BlockIO 控制器（1394-1524）

双向兼容转换是最复杂的部分：

```
if (v2 IO controller) {
    if (用户设了 IOWeight=) {
        io.weight ← weight
    } else if (用户设了 BlockIOWeight=) {
        io.weight ← cgroup_weight_blkio_to_io(blkio_weight)  // 自动转换
    }
    io.max ← "major:minor rbps=... wbps=... riops=... wiops=..."
    io.latency ← "major:minor target=usec"
} else {  // v1 blkio controller
    if (用户设了 IOWeight=) {
        blkio.weight ← cgroup_weight_io_to_blkio(io_weight)  // 反向转换
    } else if (用户设了 BlockIOWeight=) {
        blkio.weight ← weight
    }
    blkio.throttle.read_bps_device ← 带宽
}
```

**关键设计**：用户只需配置 `IOWeight=`（v2 风格）或 `BlockIOWeight=`（v1 风格），systemd 自动在两个版本间转换，通过 `log_cgroup_compat()` 记录转换日志。

#### Memory 控制器（1530-1570）

```
if (unified) {
    memory.min      ← unit_get_ancestor_memory_min(u)   // 层级继承
    memory.low      ← unit_get_ancestor_memory_low(u)   // 层级继承
    memory.high     ← c->memory_high
    memory.max      ← c->memory_max (或 MemoryLimit= 的兼容值)
    memory.swap.max ← c->memory_swap_max
    memory.oom.group ← c->memory_oom_group (0/1)
} else {
    memory.limit_in_bytes ← c->memory_limit (或 MemoryMax= 的兼容值)
}
```

**层级化保护**：`memory.min` 和 `memory.low` 通过 `unit_get_ancestor_memory_min/low()` 实现父 cgroup 的默认值继承（`DefaultMemoryMin=`/`DefaultMemoryLow=`）。

#### PIDs 控制器（1578-1617）

```
if (host_root_cgroup) {
    // 根 cgroup 没有 pids.max，映射到 sysctl
    /proc/sys/kernel/pid_max ← tasks_max_resolve(&c->tasks_max)
} else {
    pids.max ← "数值" 或 "max"
}
```

**特殊处理**：根 cgroup 的 `TasksMax=` 映射到 procfs sysctl，且采用"单向接管"策略 — 一旦 systemd 设置过，后续的 unbounded 也会写入；从未设置过则不干预。reload 可重置此标志。

#### 设备控制器（1574-1576）

```
if (v2 或 host_root) {
    cgroup_apply_devices(u)  → 使用 BPF 程序
} else if (v1) {
    devices.allow / devices.deny ← 规则列表
}
```

#### BPF 伪控制器（1619-1629）

```
CGROUP_MASK_BPF_FIREWALL                  → cgroup_apply_firewall(u)
CGROUP_MASK_BPF_FOREIGN                   → cgroup_apply_bpf_foreign_program(u)
CGROUP_MASK_BPF_SOCKET_BIND               → cgroup_apply_socket_bind(u)
CGROUP_MASK_BPF_RESTRICT_NETWORK_INTERFACES → cgroup_apply_restrict_network_interfaces(u)
```

### 4.3 is_local_root 保护

几乎所有控制器都检查 `is_local_root`（本容器/主机的根 cgroup），避免在根 cgroup 上写入不支持或应由上级管理的属性。这对容器场景特别重要 — 容器的 root cgroup 资源由容器管理器控制。

---

## 5. 控制器掩码计算体系

systemd 使用四层掩码来决定每个 cgroup 节点应启用哪些控制器：

### 5.1 掩码函数

```
unit_get_own_mask(u)           // 本 Unit 自身需要的控制器
  = unit_get_cgroup_mask(u)    //   配置文件声明的
  | unit_get_bpf_mask(u)       //   BPF 需要的
  | unit_get_delegate_mask(u)  //   委托的

unit_get_members_mask(u)       // 所有子 Unit 需要的（递归聚合）
unit_get_siblings_mask(u)      // 兄弟 Unit 需要的（= 父 slice 的 members_mask）

unit_get_target_mask(u)        // 最终目标掩码
  = own_mask | members_mask | siblings_mask
  & manager->cgroup_supported  // 内核支持的
  & ~ancestor_disable_mask     // 祖先禁用的

unit_get_enable_mask(u)        // 为子 cgroup 启用的（v2: cgroup.subtree_control）
  = members_mask
  & cgroup_supported
  & ~ancestor_disable_mask
```

### 5.2 掩码传播方向

```
         -.slice (root)
         /       \
   system.slice   user.slice
    /    \
  a.service  b.service

掩码向上聚合（members_mask）:
  a.service 需要 CPU → system.slice.members_mask |= CPU
  system.slice.members_mask → root.slice.members_mask |= CPU

启用向下传播（enable_mask → cgroup.subtree_control）:
  root.slice 写 cgroup.subtree_control = "+cpu"
  system.slice 写 cgroup.subtree_control = "+cpu"
```

### 5.3 DisableControllers 继承

```c
// cgroup.c:1848-1863
CGroupMask unit_get_ancestor_disable_mask(Unit *u) {
    mask = unit_get_disable_mask(u);       // 本 Unit 的 DisableControllers=
    slice = UNIT_GET_SLICE(u);
    if (slice)
        mask |= unit_get_ancestor_disable_mask(slice);  // 递归向上
    return mask;
}
```

任何祖先 slice 的 `DisableControllers=` 都会向下继承，子孙无法覆盖。

---

## 6. Cgroup 实现化（Realize）流程

### 6.1 unit_realize_cgroup()（cgroup.c:2686-2709）

这是 cgroup 实现化的入口，在 Unit 启动时调用：

```
unit_realize_cgroup(u)
  ├── unit_get_target_mask(u)      // 计算需要哪些控制器
  ├── unit_get_enable_mask(u)      // 计算为子启用哪些
  └── unit_realize_cgroup_now(u, target, enable, state)
        ├── 递归处理父 slice（深度优先上溯）
        ├── unit_realize_cgroup_now_enable(u)   // 广度优先启用
        ├── unit_realize_cgroup_now_disable(u)  // 深度优先禁用
        └── unit_update_cgroup(u, target, enable, state)
```

### 6.2 unit_update_cgroup()（cgroup.c:2128-2217）

10 步核心流程：

```
1. unit_pick_cgroup_path(u)                    // 确定 cgroup 路径
2. cg_create_everywhere(supported, target, path) // 在所有需要的层级创建目录
3. cg_get_cgroupid()                            // 获取 cgroup ID (v2)
4. unit_watch_cgroup(u)                         // inotify 监视 cgroup.events
5. unit_watch_cgroup_memory(u)                  // inotify 监视 memory.events
6. cg_enable_everywhere(supported, enable, path) // 写 cgroup.subtree_control
7. 记录 realized 状态                            // cgroup_realized = true
8. cg_migrate_v1_controllers()                  // v1: 迁移进程到正确的层级
9. cg_trim_v1_controllers()                     // v1: 清理不需要的 cgroup 目录
10. cgroup_context_apply(u, target, state)       // 写入所有资源控制参数
```

### 6.3 Cgroup 路径规则

```
unit_default_cgroup_path(u):
  系统模式: /<cgroup_root>/<slice_path>/<unit_name>
  用户模式: /<cgroup_root>/<slice_path>/<unit_name>

示例:
  nginx.service → /sys/fs/cgroup/system.slice/nginx.service
  user@1000.service → /sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service
  docker-abc.scope → /sys/fs/cgroup/machine.slice/docker-abc.scope
```

---

## 7. Manager Cgroup 初始化

### 7.1 manager_setup_cgroup()（cgroup.c:3294-3448）

PID 1 启动时执行的 10 步 cgroup 初始化：

```
步骤 1: 确定当前 cgroup 根路径
  cg_pid_get_path(SYSTEMD_CGROUP_CONTROLLER, 0, &m->cgroup_root)
  去除尾部的 /init.scope 或 /system.slice

步骤 2: 检测 unified/legacy/hybrid 模式
  cg_all_unified(), cg_unified_controller()

步骤 3: 创建 cgroup-empty 延迟事件源
  sd_event_add_defer() → on_cgroup_empty_event
  优先级: SD_EVENT_PRIORITY_NORMAL-5（在 SIGCHLD 之后）

步骤 4: 安装通知机制
  if (v2) {
    inotify_init1() → 监视 cgroup.events
    优先级: SD_EVENT_PRIORITY_NORMAL-9（比 empty 事件更早）
  } else if (v1 + system manager) {
    cg_install_release_agent(SYSTEMD_CGROUPS_AGENT_PATH)
  }

步骤 5: 创建 init.scope 并迁移进程
  cg_create_and_attach(controller, "<root>/init.scope", 0)
  cg_migrate(root → init.scope)  // 其他用户空间进程也移入

步骤 6: 固定 cgroupfs 挂载点
  open(path, O_RDONLY) → m->pin_cgroupfs_fd

步骤 7: 启用 legacy memory 层级支持
  memory.use_hierarchy = 1

步骤 8: 探测支持的真实控制器
  cg_mask_supported_subtree(root, &m->cgroup_supported)

步骤 9: 探测支持的 BPF 伪控制器
  cg_bpf_mask_supported(&mask)

步骤 10: 日志记录每个控制器的支持状态
```

### 7.2 cgroup 空通知机制

```
cgroup v2 路径:
  inotify(cgroup.events) → on_cgroup_inotify_event()
    → 读取 "populated 0" → unit_add_to_cgroup_empty_queue(u)
    → on_cgroup_empty_event() → unit_gc_sweep() 触发清理

cgroup v1 路径:
  release_agent → /usr/lib/systemd/systemd-cgroups-agent
    → 通过 D-Bus 通知 PID 1 → manager_notify_cgroup_empty()
```

---

## 8. Slice 层次结构

### 8.1 Slice Unit 结构

```c
// slice.h:8-14
struct Slice {
    Unit meta;                      // 基类
    SliceState state;               // SLICE_DEAD / SLICE_ACTIVE
    SliceState deserialized_state;
    CGroupContext cgroup_context;   // 资源控制上下文
};
```

### 8.2 状态机（2 个状态）

```
SLICE_DEAD ─── slice_start() ──→ SLICE_ACTIVE
     ↑                                │
     └──── slice_stop() ──────────────┘

slice_start():
  1. unit_acquire_invocation_id(u)
  2. unit_realize_cgroup(u)    // 创建 cgroup 目录
  3. unit_reset_accounting(u)  // 清零审计数据
  4. → SLICE_ACTIVE

slice_stop():
  1. → SLICE_DEAD
  2. unit_notify() 会自动修剪 cgroup
```

### 8.3 父子层次构建

Slice 名称自动编码层次关系：

```
名称解析规则 (slice_build_parent_slice()):
  "-.slice"                → 根 slice（无父）
  "system.slice"           → 父是 "-.slice"
  "system-dbus.slice"      → 父是 "system.slice"
  "user-1000.slice"        → 父是 "user.slice"
  "user-1000-app.slice"    → 父是 "user-1000.slice"

依赖关系:
  slice_add_parent_slice(s):
    unit_add_dependency_by_name(u, UNIT_IN_SLICE, parent_name)
```

### 8.4 永久 Slice（Perpetual）

```c
// slice.c:327-348
slice_enumerate_perpetual(Manager *m):
  slice_make_perpetual(m, "-.slice")      // root.slice - 永远存在
    → 如果管理主机根 cgroup：
      cpu_accounting = true
      tasks_accounting = true
      memory_accounting = true            // 自动启用审计
  
  if (MANAGER_IS_SYSTEM(m)):
    slice_make_perpetual(m, "system.slice") // 系统服务默认 slice
```

### 8.5 默认 Slice 层次拓扑

```
-.slice (root)
├── init.scope                  ← PID 1 自身
├── system.slice                ← 系统服务默认位置
│   ├── sshd.service
│   ├── nginx.service
│   └── dbus.service
├── user.slice                  ← 用户会话
│   ├── user-1000.slice
│   │   └── user@1000.service
│   │       ├── session.slice
│   │       └── app.slice
│   └── user-1001.slice
└── machine.slice               ← 容器/虚拟机
    ├── docker-xxx.scope
    └── machine-qemu-vm1.scope
```

### 8.6 Slice 冻结器

```c
// slice.c:371-415
slice_freezer_action(Unit *s, FreezerAction action):
  1. slice_freezer_action_supported_by_children(s)  // 递归检查
     - 遍历 UNIT_ATOM_SLICE_OF 的所有成员
     - 子 slice 递归检查
     - 检查每个成员 vtable 是否有 freeze 方法
  
  2. UNIT_FOREACH_DEPENDENCY(member, s, UNIT_ATOM_SLICE_OF):
     - FREEZER_FREEZE → member->vtable->freeze(member)
     - FREEZER_THAW   → member->vtable->thaw(member)
  
  3. unit_cgroup_freezer_action(s, action)
     → 写入 cgroup.freeze = 1/0（v2 only）
```

**设计**：冻结是自顶向下递归的，先冻结子成员，再冻结 slice 自身的 cgroup。

---

## 9. BPF 集成

### 9.1 BPF 设备控制（bpf-devices.c）

在 cgroup v2 中，`devices` 控制器被移除，由 BPF 程序替代：

```
cgroup_apply_devices(u):
  if (v2) → bpf-devices.c:
    1. 构建 BPF_PROG_TYPE_CGROUP_DEVICE 程序
    2. 每条 DeviceAllow= 规则生成一条 BPF 指令
       - 匹配 type (block/char)、major、minor、access (r/w/m)
    3. BPF_F_ALLOW_MULTI 方式 attach 到 cgroup
    
    三种策略:
      AUTO   → 允许列表设备 + 内建设备（/dev/null 等）
      CLOSED → 仅允许列表设备 + 内建设备
      STRICT → 严格只允许列表设备
  
  if (v1) → 直接写:
    devices.deny  = "a"     // 先拒绝所有
    devices.allow = "c 1:3 rwm"  // 再逐条允许
```

### 9.2 BPF IP 防火墙（bpf-firewall.c）

```
cgroup_apply_firewall(u):
  1. 构建 BPF_PROG_TYPE_CGROUP_SKB 程序
  2. 两个方向：ingress + egress
  3. 遍历 ip_address_allow / ip_address_deny 集合
     - IPv4: 比较 iphdr->saddr/daddr 与前缀
     - IPv6: 比较 ipv6hdr->saddr/daddr 与前缀
  4. 如果启用 ip_accounting：
     - BPF map 记录 packets/bytes 计数
  5. attach 到 cgroup（BPF_CGROUP_INET_INGRESS/EGRESS）
```

### 9.3 其他 BPF 程序

| BPF 类型 | 配置指令 | 功能 |
|----------|----------|------|
| Foreign | `BPFProgram=` | 加载用户自定义 BPF 程序 |
| Socket Bind | `SocketBindAllow/Deny=` | 限制可绑定的端口范围 |
| Restrict NI | `RestrictNetworkInterfaces=` | 限制可使用的网络接口 |
| Restrict FS | `RestrictFileSystems=` | 限制可访问的文件系统类型（在 exec_child 中应用） |

---

## 10. 委托机制（Delegation）

### 10.1 概念

Delegation 允许 Unit 内的进程自行管理 cgroup 子树，常用于：
- 容器运行时（Docker、Podman）：Scope 单元 + `Delegate=yes`
- systemd 用户实例：`user@.service` + `Delegate=yes`

### 10.2 实现

```c
// cgroup.h:124-126
bool delegate;                    // Delegate= 布尔开关
CGroupMask delegate_controllers;  // DelegateControllers= 指定控制器

// cgroup.c:1771-1792
unit_get_delegate_mask(u):
  if (!delegate) return 0;
  
  // v1: 如果进程会降权，不委托（安全考虑）
  if (v1 && exec_context_drops_privileges(e))
      return 0;
  
  // v2: 安全委托给非特权进程
  return CGROUP_MASK_EXTEND_JOINED(c->delegate_controllers);
```

### 10.3 委托对 realize 的影响

```c
// unit_update_cgroup() 中:
if (created || !realized || !unit_cgroup_delegate(u)) {
    // 非委托单元: 每次都更新 subtree_control
    cg_enable_everywhere(supported, enable_mask, path);
} else {
    // 委托单元: 保留进程自行设置的 subtree_control
    // 不覆盖委托子树的控制器配置
}
```

---

## 11. 资源审计（Accounting）

### 11.1 审计数据读取函数

```c
// cgroup.h:294-304
unit_get_cpu_usage(u, &nsec)          // 读取 cpuacct.usage (v1) 或 cpu.stat (v2)
unit_get_memory_current(u, &bytes)    // 读取 memory.current (v2) 或 memory.usage_in_bytes (v1)
unit_get_memory_available(u, &bytes)  // memory.high/max - memory.current
unit_get_tasks_current(u, &count)     // 读取 pids.current
unit_get_io_accounting(u, metric, &val) // 读取 io.stat (v2) 或 blkio.throttle.* (v1)
unit_get_ip_accounting(u, metric, &val) // 从 BPF map 读取 IP 流量统计
```

### 11.2 审计指标枚举

```c
// IP 审计
CGROUP_IP_INGRESS_BYTES, CGROUP_IP_INGRESS_PACKETS,
CGROUP_IP_EGRESS_BYTES,  CGROUP_IP_EGRESS_PACKETS

// IO 审计
CGROUP_IO_READ_BYTES,  CGROUP_IO_WRITE_BYTES,
CGROUP_IO_READ_OPERATIONS, CGROUP_IO_WRITE_OPERATIONS
```

### 11.3 审计重置

```c
unit_reset_accounting(u):
  unit_reset_cpu_accounting(u)   // 记录当前值作为基线
  unit_reset_io_accounting(u)
  unit_reset_ip_accounting(u)
```

---

## 12. D-Bus 接口与运行时修改

### 12.1 属性暴露（dbus-cgroup.c:452-509）

所有 CGroupContext 字段都通过 D-Bus 属性暴露，systemctl show 可查看：

```
CPUAccounting, CPUWeight, StartupCPUWeight, CPUQuotaPerSecUSec,
CPUQuotaPeriodUSec, IOAccounting, IOWeight, StartupIOWeight,
MemoryAccounting, MemoryMin, MemoryLow, MemoryHigh, MemoryMax,
MemorySwapMax, TasksAccounting, TasksMax, DevicePolicy,
Delegate, DelegateControllers, DisableControllers,
IPAddressAllow, IPAddressDeny, ManagedOOMSwap, ...
```

### 12.2 运行时 set-property（dbus-cgroup.c:512-740）

`systemctl set-property` 调用路径：

```
systemctl set-property nginx.service MemoryMax=512M
  → D-Bus: SetProperties(["MemoryMax", 536870912])
  → bus_cgroup_set_transient_property()
    1. 更新内存中的 CGroupContext 字段
    2. unit_write_settingf() 写入 drop-in 文件持久化
    3. unit_invalidate_cgroup() 标记 cgroup 需要重新 apply
    4. 下一轮 realize 时 cgroup_context_apply() 写入新值
```

---

## 13. Scope 的 Cgroup 管理

### 13.1 外部进程分组

Scope 不创建子进程，而是将**已有的外部进程**纳入 cgroup：

```c
// scope.c:356-416
scope_start():
  1. unit_realize_cgroup(u)           // 创建 cgroup 目录
  2. unit_reset_accounting(u)         // 清零审计
  3. unit_attach_pids_to_cgroup(u, u->pids, NULL)  // 将 PID 集合移入
  4. → SCOPE_RUNNING

// scope 是容器和用户会话的关键机制:
// - systemd-logind 为每个会话创建 scope
// - 容器运行时创建 scope 并设 Delegate=yes
```

### 13.2 Scope 放弃（Abandon）

```c
scope_abandon():
  was_abandoned = true
  // 后续 stop 时使用 KILL_TERMINATE_AND_LOG
  // 而非正常的 KILL_TERMINATE
```

---

## 14. cg_create / cg_attach / cg_kill 底层实现

### 14.1 创建 cgroup（cgroup-setup.c:283-320）

```c
cg_create(controller, path):
  cg_get_path(controller, path, NULL, &fs)  // 拼接文件系统路径
  mkdir_parents(fs, 0755)                    // 创建父目录
  mkdir(fs, 0755)                            // 创建 cgroup 目录
```

### 14.2 附加进程（cgroup-setup.c:322-350）

```c
cg_attach(controller, path, pid):
  cg_get_path(controller, path, "cgroup.procs", &fs)  // v2
  // 或 "tasks" (v1)
  write(fs, pid_str)  // 写入 PID
```

### 14.3 杀死 cgroup 中的进程（cgroup-util.c:250-462）

```c
cg_kill(controller, path, sig, flags):
  if (v2 && sig == SIGKILL) {
    // 优先使用原子的 cgroup.kill 接口
    cg_get_path(controller, path, "cgroup.kill", &killfile)
    write(killfile, "1")  // 一次性杀死所有进程
  } else {
    // 遍历 cgroup.procs 逐个发信号
    opendir(cgroup_path)
    while (readdir → pid):
      kill(pid, sig)
  }

cg_kill_recursive():
  // 递归遍历子 cgroup，深度优先杀死
  // CGROUP_REMOVE 标志：杀完后 rmdir
```

### 14.4 路径映射

```c
cg_get_path(controller, path, suffix, &ret):
  v2: /sys/fs/cgroup/<path>/<suffix>
  v1: /sys/fs/cgroup/<controller>/<path>/<suffix>

cg_pid_get_path(controller, pid, &path):
  读取 /proc/<pid>/cgroup
  解析 "层级ID:控制器列表:路径" 格式
```

---

## 15. 完整数据流图

```
┌──────────────────────────────────────────────────────────┐
│                    配置文件层                              │
│  [Service]               [Slice]                         │
│  CPUWeight=200           MemoryMax=2G                    │
│  MemoryMax=512M          TasksMax=1024                   │
│  IOWeight=100                                            │
└────────────────────┬─────────────────────────────────────┘
                     │ load_fragment()
                     ▼
┌──────────────────────────────────────────────────────────┐
│                CGroupContext                              │
│  .cpu_weight = 200                                       │
│  .memory_max = 536870912                                 │
│  .io_weight = 100                                        │
└────────────────────┬─────────────────────────────────────┘
                     │ unit_realize_cgroup()
                     ▼
┌──────────────────────────────────────────────────────────┐
│            掩码计算层                                     │
│  own_mask = CPU | MEMORY | IO                            │
│  members_mask = (子 Unit 聚合)                           │
│  target_mask = own | members | siblings                  │
│               & supported & ~disabled                    │
│  enable_mask = members & supported & ~disabled           │
└────────────────────┬─────────────────────────────────────┘
                     │ unit_update_cgroup()
                     ▼
┌──────────────────────────────────────────────────────────┐
│            内核 cgroup 文件系统                            │
│                                                          │
│  /sys/fs/cgroup/                                         │
│  ├── cgroup.subtree_control = "+cpu +memory +io +pids"   │
│  ├── system.slice/                                       │
│  │   ├── cgroup.subtree_control = "+cpu +memory +io"     │
│  │   └── nginx.service/                                  │
│  │       ├── cpu.weight = 200                            │
│  │       ├── cpu.max = max 100000                        │
│  │       ├── memory.max = 536870912                      │
│  │       ├── memory.high = max                           │
│  │       ├── io.weight = default 100                     │
│  │       ├── pids.max = max                              │
│  │       ├── cgroup.events (inotify 监视)                │
│  │       └── cgroup.procs = {PID1, PID2, ...}           │
│  └── init.scope/                                         │
│      └── cgroup.procs = {1}  (PID 1)                    │
│                                                          │
│  BPF 程序 (attach 到 cgroup fd):                         │
│  ├── BPF_PROG_TYPE_CGROUP_DEVICE → 设备访问控制          │
│  ├── BPF_PROG_TYPE_CGROUP_SKB    → IP 防火墙             │
│  └── BPF_PROG_TYPE_SOCK_ADDR    → Socket 绑定限制        │
└──────────────────────────────────────────────────────────┘
```

---

## 16. 设计要点总结

### 16.1 透明的 v1/v2 兼容

- 用户配置 `CPUWeight=`（v2 语义），systemd 自动转换为 `CPUShares=`（v1）
- 用户配置 `IOWeight=`（v2 语义），自动转换为 `BlockIOWeight=`（v1）
- 反向兼容同样支持（v1 配置在 v2 系统上生效）
- 所有转换通过 `log_cgroup_compat()` 记录日志

### 16.2 四层掩码架构

`own_mask → members_mask → siblings_mask → target_mask` 的层级聚合确保：
- 父 cgroup 总是启用子 cgroup 需要的控制器
- `DisableControllers=` 向下继承，无法被子级覆盖
- v2 的 `cgroup.subtree_control` 自动维护

### 16.3 BPF 扩展机制

cgroup v2 移除了 `devices` 控制器，systemd 使用 BPF 替代，并扩展为通用的安全过滤框架：
- 设备访问控制 → BPF_PROG_TYPE_CGROUP_DEVICE
- IP 防火墙 → BPF_PROG_TYPE_CGROUP_SKB
- Socket 绑定 → BPF_PROG_TYPE_CGROUP_SOCK_ADDR
- 网络接口限制 → BPF LSM
- 文件系统限制 → BPF LSM

### 16.4 Slice 层次与命名约定

- Slice 名称通过 `-` 分隔符编码层次关系
- `system.slice` / `user.slice` / `machine.slice` 三大默认 slice
- Slice 是资源控制的组织单元，本身只有 DEAD/ACTIVE 两个状态
- 冻结操作递归传播到所有子成员

### 16.5 安全委托

- v2 支持安全地委托控制器给非特权进程
- v1 在进程降权时不委托（避免安全问题）
- 委托单元的 `subtree_control` 不被 systemd 覆盖

### 16.6 根 cgroup 特殊处理

- 根 cgroup 的 `TasksMax=` 映射到 procfs sysctl
- 根 cgroup 不写入 CPU/Memory/IO 属性（由内核管理）
- 容器根 cgroup 不写入属性（由容器管理器控制）
- `is_local_root` 和 `is_host_root` 两个标志区分不同场景
