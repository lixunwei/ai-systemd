# 23. systemd-oomd 内存压力管理深度分析

## 1. 概述

`systemd-oomd` 是 systemd 提供的**用户空间 OOM（Out-Of-Memory）杀手**，利用 Linux cgroup v2 的 PSI（Pressure Stall Information）机制实现**主动式**内存压力管理。与内核 OOM killer 的被动触发不同，oomd 在系统陷入严重内存压力之前就介入，选择性地终止消耗资源最多的 cgroup。

### 核心设计原则

- **基于压力而非阈值**：使用 PSI `full` 指标度量实际性能损失
- **cgroup 粒度杀戮**：终止整个 cgroup（服务），而非单个进程
- **与 PID 1 集成**：通过 Varlink 协议获知哪些 cgroup 需要监控
- **防杀死风暴**：Post-action delay 机制防止连续快速杀死

### 源文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `oomd.c` | ~200 | main() 入口、配置解析、启动 |
| `oomd-manager.h` | 74 | Manager 结构体定义、常量 |
| `oomd-manager.c` | ~800 | 定时器处理、PID 1 集成、varlink 通信 |
| `oomd-manager-bus.c` | ~50 | D-Bus 接口（DumpByFileDescriptor） |
| `oomd-util.h` | 137 | OomdCGroupContext 结构体、比较函数、API 声明 |
| `oomd-util.c` | ~450 | 核心算法：压力检测、排序、kill 逻辑、数据采集 |
| `oomctl.c` | ~100 | oomctl 命令行工具 |

---

## 2. 关键常量

```c
// oomd-manager.h
#define SWAP_INTERVAL_USEC             150000          // Swap 轮询间隔：150ms
#define MEM_PRESSURE_INTERVAL_USEC     (1 * USEC_PER_SEC)  // 压力轮询间隔：1秒

#define DEFAULT_MEM_PRESSURE_DURATION_USEC  (30 * USEC_PER_SEC)  // 压力持续阈值：30秒
#define DEFAULT_MEM_PRESSURE_LIMIT_PERCENT  60   // PSI full avg10 > 60% 触发
#define DEFAULT_SWAP_USED_LIMIT_PERCENT     90   // Swap 使用 > 90% 触发

#define THRESHOLD_SWAP_USED_PERCENT   5    // 候选 cgroup 最少使用 5% 总 swap
#define RECLAIM_DURATION_USEC         (30 * USEC_PER_SEC)  // 回收活动检查窗口：30秒
#define POST_ACTION_DELAY_USEC        (15 * USEC_PER_SEC)  // kill 后冷却期：15秒
#define DUMP_ON_KILL_COUNT            10   // kill 时记录 top-10 候选
```

---

## 3. Manager 结构体（oomd-manager.h:32-62）

```c
struct Manager {
    sd_bus *bus;                          // D-Bus 连接
    sd_event *event;                     // sd-event 事件循环

    Hashmap *polkit_registry;            // Polkit 异步认证状态

    /* 配置 */
    bool dry_run;                        // 模拟模式（只记录不杀）
    int swap_used_limit_permyriad;       // Swap 使用阈值（万分比）
    loadavg_t default_mem_pressure_limit; // PSI 压力阈值
    usec_t default_mem_pressure_duration_usec; // 压力持续时间阈值

    /* 监控集合 (key=cgroup路径, value=OomdCGroupContext) */
    Hashmap *monitored_swap_cgroup_contexts;           // Swap 监控目标
    Hashmap *monitored_mem_pressure_cgroup_contexts;   // 压力监控目标（检测用）
    Hashmap *monitored_mem_pressure_cgroup_contexts_candidates; // 压力监控候选（kill 用）

    /* 系统全局内存/Swap 状态 */
    OomdSystemContext system_context;

    /* 防杀死风暴 */
    usec_t mem_pressure_post_action_delay_start; // 上次 kill 的时间戳

    /* 定时器事件源 */
    sd_event_source *swap_context_event_source;          // 150ms 周期
    sd_event_source *mem_pressure_context_event_source;  // 1s 周期

    /* Varlink 通信 */
    Varlink *varlink_client;        // 订阅 PID 1 的 ManagedOOM 变更
    VarlinkServer *varlink_server;  // 接收用户 Manager 上报的 ManagedOOM 变更
};
```

---

## 4. OomdCGroupContext 结构体（oomd-util.h:20-40）

每个被监控的 cgroup 一份实例：

```c
struct OomdCGroupContext {
    char *path;                       // cgroup 路径

    ResourcePressure memory_pressure; // PSI 数据 (avg10, avg60, avg300, total)

    uint64_t current_memory_usage;    // memory.current
    uint64_t memory_min;              // memory.min（保护下限）
    uint64_t memory_low;              // memory.low（保护下限）
    uint64_t swap_usage;              // memory.swap.current

    uint64_t last_pgscan;             // 上一次采样的 pgscan 值
    uint64_t pgscan;                  // 当前 pgscan 值（from memory.stat）

    ManagedOOMPreference preference;  // NONE / AVOID / OMIT

    /* 仅用于压力监控模式 */
    loadavg_t mem_pressure_limit;           // 该 cgroup 的压力阈值
    usec_t mem_pressure_limit_hit_start;    // 压力首次超限的时间戳
    usec_t last_had_mem_reclaim;            // 上次有回收活动的时间戳
};
```

### 4.1 数据采集（oomd-util.c:329-410）

`oomd_cgroup_context_acquire(path)` 采集流程：

```
1. 读取 memory.pressure → PSI full 数据 (avg10/avg60/avg300/total)
2. 读取 cgroup owner UID
   - UID=0 时检查 xattr:
     - user.oomd_avoid → MANAGED_OOM_PREFERENCE_AVOID
     - user.oomd_omit  → MANAGED_OOM_PREFERENCE_OMIT
3. if (root cgroup):
     从 /proc/meminfo 获取内存使用量
   else:
     memory.current      → current_memory_usage
     memory.min          → memory_min
     memory.low          → memory_low
     memory.swap.current → swap_usage
     memory.stat:pgscan  → pgscan（页面扫描计数）
```

### 4.2 系统上下文（OomdSystemContext）

```c
struct OomdSystemContext {
    uint64_t mem_total;    // 总物理内存
    uint64_t mem_used;     // 已使用内存
    uint64_t swap_total;   // 总 Swap
    uint64_t swap_used;    // 已使用 Swap
};
```

从 `/proc/meminfo` 读取 MemTotal、MemFree、SwapTotal、SwapFree 计算。

---

## 5. 双通道监控架构

oomd 运行两个独立的定时器，分别处理两种 OOM 场景：

```
┌─────────────────────────────────────────┐
│              sd-event 事件循环            │
├───────────────────┬─────────────────────┤
│ Swap 监控定时器   │ 内存压力监控定时器     │
│ 间隔: 150ms      │ 间隔: 1s             │
│ 触发条件:        │ 触发条件:             │
│  mem_free < 10%  │  PSI avg10 > 60%     │
│  AND             │  持续 > 30s          │
│  swap_free < 10% │  AND 有回收活动       │
│ Kill 策略:       │ Kill 策略:            │
│  最高 swap 使用   │  最高 pgscan 速率     │
└───────────────────┴─────────────────────┘
```

---

## 6. Swap 监控逻辑（oomd-manager.c:348-425）

### 6.1 触发条件

```c
// 两个条件同时满足才触发：
if (oomd_mem_free_below(&system_context, 10000 - swap_used_limit_permyriad) &&
    oomd_swap_free_below(&system_context, 10000 - swap_used_limit_permyriad))
```

默认配置（swap_used_limit = 90%）意味着：
- 内存空闲 < 10% **且** Swap 空闲 < 10% 时触发

### 6.2 Kill 流程

```
monitor_swap_contexts_handler() [每150ms]:
  1. 重置定时器
  2. 检查 varlink 连接（断线重连）
  3. 从 /proc/meminfo 更新系统上下文
  4. 检查 swap 监控集合是否为空（空则跳过）
  5. 检查触发条件（mem_free AND swap_free 都低于阈值）
  6. 获取候选 cgroup（monitored_swap 的子 cgroup）
  7. 计算 swap 门槛 = swap_total × 5%
  8. oomd_kill_by_swap_usage(candidates, threshold)
     → 选择 swap 使用量最高且 > 5% 总 swap 的 cgroup
     → 递归 SIGKILL 该 cgroup 所有进程
  9. 记录 kill 日志
```

### 6.3 oomd_kill_by_swap_usage()（oomd-util.c:285-327）

```c
int oomd_kill_by_swap_usage(Hashmap *h, uint64_t threshold_usage, bool dry_run, char **ret_selected):
  1. oomd_sort_cgroup_contexts(h, compare_swap_usage, NULL, &sorted)
     - 排除 OMIT 标记的 cgroup
     - 按 swap_usage 从大到小排序
     - AVOID 标记的排在最后
  2. 遍历排序后的列表:
     - 跳过 swap_usage <= threshold（5% 总 swap）
     - oomd_cgroup_kill(path, recurse=true, dry_run)
     - 首次成功 kill 后立即返回
```

---

## 7. 内存压力监控逻辑（oomd-manager.c:433-562）

### 7.1 触发条件

```
PSI full avg10 > mem_pressure_limit（默认 60%）
持续时间 > default_mem_pressure_duration（默认 30s）
且 30s 内有页面回收活动（pgscan 在增长）
```

### 7.2 完整流程

```
monitor_memory_pressure_contexts_handler() [每1s]:
  1. 重置定时器
  2. 检查 varlink 连接
  3. 检查监控集合是否为空
  4. update_monitored_cgroup_contexts() — 刷新所有被监控 cgroup 的数据
  5. 检查 post-action delay（15s 冷却期）
  6. oomd_pressure_above(monitored, duration, &targets) — 找出超限 cgroup
  7. if (有超限 cgroup AND 不在冷却期):
     for each target in targets:
       a. 检查 30s 内是否有回收活动（last_had_mem_reclaim）
          - 无活动则跳过（kill 无益）
       b. 更新候选 cgroup 数据
       c. oomd_kill_by_pgscan_rate(candidates, target->path, ...)
          - 在 target 路径前缀下选择 pgscan 最快的子 cgroup
       d. 设置 post_action_delay_start = now
       e. 首次成功 kill 后立即返回（每轮只杀一个）
  8. else if (有 cgroup 正在接近阈值):
     - 预先更新候选数据（省 CPU）
```

### 7.3 oomd_pressure_above()（oomd-util.c:69-106）

```c
int oomd_pressure_above(Hashmap *h, usec_t duration, Set **ret):
  遍历所有被监控 cgroup:
    if (ctx->memory_pressure.avg10 > ctx->mem_pressure_limit):
      // 首次超限，记录时间戳
      if (mem_pressure_limit_hit_start == 0)
          mem_pressure_limit_hit_start = now()
      
      // 检查持续时间
      diff = now() - mem_pressure_limit_hit_start
      if (diff >= duration):
          加入 targets 集合
    else:
      // 回到正常，重置时间戳
      mem_pressure_limit_hit_start = 0
```

### 7.4 oomd_kill_by_pgscan_rate()（oomd-util.c:243-283）

```c
int oomd_kill_by_pgscan_rate(Hashmap *h, const char *prefix, bool dry_run, char **ret_selected):
  1. oomd_sort_cgroup_contexts(h, compare_pgscan_rate_and_memory_usage, prefix, &sorted)
     - 仅保留 prefix 路径下的 cgroup
     - 排除 OMIT 标记
     - 按 pgscan_rate 从大到小排序
     - 相同 pgscan_rate 时按 current_memory_usage 排序
     - AVOID 标记的排在最后
  2. 遍历:
     - 跳过 pgscan==0 且 memory_usage==0 的（杀了没用）
     - oomd_cgroup_kill(path, recurse=true, dry_run)
     - 首次成功 kill 后返回
  3. 记录 top-10 候选日志
```

### 7.5 pgscan_rate 计算

```c
// oomd-util.c:108-123
uint64_t oomd_pgscan_rate(const OomdCGroupContext *c):
  if (last_pgscan > pgscan):
    // cgroup 被重建，重置为 0
    last_pgscan = 0
  return pgscan - last_pgscan  // 本采样间隔的页面扫描增量
```

pgscan 来自 `memory.stat` 中的 `pgscan` 字段，表示内核为该 cgroup 扫描了多少页面试图回收。值越高说明该 cgroup 导致的内存压力越大。

---

## 8. Kill 执行（oomd-util.c:174-217）

```c
int oomd_cgroup_kill(const char *path, bool recurse, bool dry_run):
  if (dry_run):
    log_debug("Would have killed %s")
    return 0

  if (recurse):
    cg_kill_recursive(SYSTEMD_CGROUP_CONTROLLER, path, SIGKILL,
                      CGROUP_IGNORE_SELF, pids_killed, log_kill, NULL)
  else:
    cg_kill(SYSTEMD_CGROUP_CONTROLLER, path, SIGKILL,
            CGROUP_IGNORE_SELF, pids_killed, log_kill, NULL)

  // 记录 kill 计数到 cgroup xattr
  increment_oomd_xattr(path, "user.oomd_kill", set_size(pids_killed))

  return set_size(pids_killed) != 0
```

**关键细节**：
- 使用 `SIGKILL`（不可捕获）
- `CGROUP_IGNORE_SELF`：不杀 oomd 自身
- `recurse=true`：递归杀死所有子 cgroup 中的进程
- `user.oomd_kill` xattr 记录累计被杀进程数（供调试/监控）
- 如果 cgroup 在 kill 过程中消失（-ENOENT/-ENODEV），忽略错误

---

## 9. 偏好优先级机制（ManagedOOMPreference）

### 9.1 三种偏好

| 值 | 含义 | 行为 |
|-----|------|------|
| `NONE` | 默认 | 正常参与候选排序 |
| `AVOID` | 尽量避免 | 排在所有 NONE 之后，只有在没有其他候选时才被杀 |
| `OMIT` | 完全排除 | 从候选列表中移除，永远不被杀 |

### 9.2 实现

排序比较函数中：
```c
// oomd-util.h:81
r = CMP((*c1)->preference, (*c2)->preference);
// NONE=0 < AVOID=1, 所以 AVOID 排在后面

// oomd_sort_cgroup_contexts():
if (item->preference == MANAGED_OOM_PREFERENCE_OMIT)
    continue;  // 完全跳过 OMIT
```

### 9.3 偏好来源

```
Unit 配置:
  [Service]
  ManagedOOMPreference=avoid

→ PID 1 通过 Varlink 告知 oomd
→ oomd 设置 xattr: user.oomd_avoid = 1

运行时读取:
  oomd_cgroup_context_acquire():
    cg_get_xattr_bool(path, "user.oomd_avoid")
    cg_get_xattr_bool(path, "user.oomd_omit")
```

---

## 10. 与 PID 1 的 Varlink 集成

### 10.1 通信架构

```
┌──────────────┐  Varlink (subscribe)  ┌─────────────────┐
│  systemd     │◄──────────────────────│  systemd-oomd   │
│  (PID 1)     │  "哪些 cgroup 需要    │  (oomd client)  │
│              │   OOM 监控？"          │                 │
└──────┬───────┘                       └────────┬────────┘
       │                                        │
       │  io.systemd.ManagedOOM                 │
       │  .SubscribeManagedOOMCGroups            │
       │                                        │
┌──────┴───────┐  Varlink (report)     ┌────────┴────────┐
│  user@1000   │──────────────────────→│  systemd-oomd   │
│  (user mgr)  │  "用户服务也要监控"    │  (oomd server)  │
└──────────────┘                       └─────────────────┘
```

### 10.2 订阅协议

```
oomd → PID 1:
  方法: io.systemd.ManagedOOM.SubscribeManagedOOMCGroups
  响应: JSON 数组 [{
    "mode": "swap" | "mem-pressure",
    "path": "/sys/fs/cgroup/system.slice/nginx.service",
    "property": "ManagedOOMSwap" | "ManagedOOMMemoryPressure",
    "limit": 6000  // permyriad 格式的压力阈值
  }, ...]
```

### 10.3 Unit 配置映射

| Unit 配置 | 效果 |
|-----------|------|
| `ManagedOOMSwap=auto` | 不监控（不加入 swap 集合） |
| `ManagedOOMSwap=kill` | 加入 swap 监控集合 |
| `ManagedOOMMemoryPressure=auto` | 不监控 |
| `ManagedOOMMemoryPressure=kill` | 加入压力监控集合 |
| `ManagedOOMMemoryPressureLimit=80%` | 覆盖默认的 60% 阈值 |
| `ManagedOOMPreference=avoid` | 设置 cgroup xattr |
| `ManagedOOMPreference=omit` | 设置 cgroup xattr |

---

## 11. 启动流程（oomd.c）

```
main():
  1. 解析 /etc/systemd/oomd.conf 配置文件
     - SwapUsedLimit=（默认 90%）
     - DefaultMemoryPressureLimit=（默认 60%）
     - DefaultMemoryPressureDurationSec=（默认 30s）
  
  2. 环境检查:
     - 验证 PSI 支持（/proc/pressure/memory 存在）
     - 验证 cgroup v2 统一模式
     - 验证 memory 控制器可用
  
  3. 信号处理:
     - 阻塞 SIGINT、SIGTERM
  
  4. manager_new() → 创建 Manager
  
  5. manager_start():
     a. 连接系统 D-Bus
     b. 注册对象 /org/freedesktop/oom1
     c. 连接 PID 1 varlink（订阅 ManagedOOM 变更）
     d. 启动 varlink server（接收用户 Manager 上报）
     e. 创建 swap 定时器（150ms 周期）
     f. 创建 memory pressure 定时器（1s 周期）
  
  6. sd_event_loop(m->event)  → 进入事件循环
```

---

## 12. 防杀死风暴机制

### 12.1 问题

PSI 计数器有 ~2秒 的滞后。kill 之后，即使压力已缓解，读到的 PSI 值仍然很高，可能导致连续杀死多个 cgroup。

### 12.2 三重保护

```
保护1: Post-Action Delay（15秒）
  kill 后设置 mem_pressure_post_action_delay_start = now()
  15秒内不再执行任何压力 kill

保护2: 回收活动检查
  要求 30s 内 pgscan 有增长（last_had_mem_reclaim < 30s 前）
  如果无回收活动，说明压力可能是"假象"（页面已回收完毕）

保护3: 单轮单杀
  每个定时器周期最多杀一个 cgroup
  杀完立即返回，等下一轮重新评估
```

### 12.3 Swap 监控的不同策略

Swap 监控**没有** post-action delay，因为：
- Swap 轮询间隔仅 150ms（足够快检测变化）
- Swap 使用量变化比 PSI 更即时
- 但仍遵循"单轮单杀"原则

---

## 13. 候选选择算法对比

### 13.1 内存压力模式

```
排序键: pgscan_rate (降序) → current_memory_usage (降序)
含义: 谁导致了最多的页面扫描（即最大的回收压力）
限定: 只在超限 cgroup 的子树中选择候选
排除: OMIT 标记的 cgroup
延后: AVOID 标记的排最后
过滤: 跳过 pgscan=0 且 memory=0 的
```

### 13.2 Swap 模式

```
排序键: swap_usage (降序)
含义: 谁使用了最多的 Swap
限定: 无前缀限定（全局选择）
排除: OMIT 标记的 cgroup
延后: AVOID 标记的排最后
过滤: 跳过 swap_usage ≤ 5% 总 Swap 的
```

---

## 14. 配置文件

### 14.1 /etc/systemd/oomd.conf

```ini
[OOM]
SwapUsedLimitPercent=90
DefaultMemoryPressureLimitPercent=60
DefaultMemoryPressureDurationSec=30s
```

### 14.2 Unit 文件示例

```ini
[Service]
# 启用 Swap 监控（系统 swap 耗尽时可能被杀）
ManagedOOMSwap=kill

# 启用内存压力监控（PSI > 60% 持续 30s 时可能被杀）
ManagedOOMMemoryPressure=kill

# 自定义压力阈值（覆盖全局 60%）
ManagedOOMMemoryPressureLimit=80%

# 偏好设置
ManagedOOMPreference=avoid   # 尽量不杀
```

---

## 15. 完整数据流图

```
┌─────────────────────────────────────────────────────────────────┐
│                        systemd-oomd                              │
│                                                                 │
│  ┌─────────────────┐        ┌──────────────────────────────┐   │
│  │ Varlink Client  │        │     sd-event 事件循环          │   │
│  │ (订阅 PID 1)    │        │                              │   │
│  │                 │        │  ┌──────────┐ ┌───────────┐  │   │
│  │ → 获取监控列表  │──────→│  │Swap 定时器│ │压力定时器  │  │   │
│  └─────────────────┘        │  │  150ms   │ │   1s      │  │   │
│                             │  └────┬─────┘ └─────┬─────┘  │   │
│  ┌─────────────────┐        │       │             │         │   │
│  │ Varlink Server  │        └───────┼─────────────┼─────────┘   │
│  │ (接收 user mgr) │               │             │              │
│  └─────────────────┘               ▼             ▼              │
│                          ┌──────────────┐  ┌──────────────┐     │
│                          │/proc/meminfo │  │memory.pressure│     │
│                          │mem_free/swap │  │PSI full avg10│     │
│                          └──────┬───────┘  └──────┬───────┘     │
│                                 │                 │              │
│                                 ▼                 ▼              │
│                          ┌──────────────┐  ┌──────────────┐     │
│                          │ 触发条件检查  │  │ 触发条件检查  │     │
│                          │mem<10%       │  │avg10>60%     │     │
│                          │AND swap<10%  │  │持续>30s      │     │
│                          └──────┬───────┘  │AND有回收活动  │     │
│                                 │          └──────┬───────┘     │
│                                 ▼                 ▼              │
│                          ┌──────────────┐  ┌──────────────┐     │
│                          │候选排序      │  │候选排序      │     │
│                          │by swap_usage │  │by pgscan_rate│     │
│                          │排除OMIT      │  │限定子树前缀  │     │
│                          │延后AVOID     │  │排除OMIT      │     │
│                          └──────┬───────┘  └──────┬───────┘     │
│                                 │                 │              │
│                                 ▼                 ▼              │
│                          ┌─────────────────────────────┐        │
│                          │    oomd_cgroup_kill()        │        │
│                          │  cg_kill_recursive(SIGKILL)  │        │
│                          │  + user.oomd_kill xattr      │        │
│                          └──────────────┬──────────────┘        │
│                                         │                        │
│                                         ▼                        │
│                          ┌─────────────────────────────┐        │
│                          │  Post-Action Delay (15s)    │        │
│                          │  防止连续快速杀死            │        │
│                          └─────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 16. 与内核 OOM Killer 的对比

| 维度 | 内核 OOM Killer | systemd-oomd |
|------|----------------|--------------|
| 触发时机 | 极端情况（分配失败） | 主动检测压力上升 |
| 粒度 | 单个进程（oom_score） | 整个 cgroup（服务） |
| 选择依据 | RSS + oom_adj | PSI pgscan_rate / swap_usage |
| 可配置性 | oom_score_adj | 完整的 ManagedOOM* 配置 |
| 副作用 | 可能杀关键进程 | 按服务粒度，OMIT/AVOID 保护 |
| 延迟 | 极低（内核路径） | 1-30秒检测延迟 |
| 依赖 | 无 | cgroup v2 + PSI |

---

## 17. 设计要点总结

### 17.1 PSI 作为核心指标

- 使用 PSI `full` 而非 `some`：`full` 表示**所有任务都被阻塞**的时间比例
- `avg10` 提供 10 秒滑动窗口，平衡响应速度与稳定性
- 60% 阈值 + 30s 持续 = 高置信度判断

### 17.2 pgscan 作为 Kill 选择依据

- pgscan 直接反映"谁导致了内核的回收压力"
- 比简单的 memory.current 更准确（大内存不一定有压力）
- 增量计算（pgscan - last_pgscan）反映当前活跃压力

### 17.3 Varlink 协议解耦

- oomd 不直接读取 Unit 文件或 D-Bus 属性
- 通过 Varlink 订阅/上报模式，PID 1 推送变更
- 用户 Manager 也可上报，支持用户会话级别的 OOM 管理

### 17.4 最小侵入性

- `dry_run` 模式供测试
- OMIT/AVOID 保护关键服务
- 单轮单杀 + 15s 冷却防止过度反应
- kill 前验证回收活动存在（避免杀死无辜）
