# systemd Unit/Job 逻辑关系、依赖图与执行顺序深度分析

## 一、总体概述

systemd 的核心工作就是管理 Unit 之间的**依赖关系**，并据此生成**Job 事务**来执行操作。整个过程分为三个阶段：

```
依赖关系建模（静态）→ 事务构建与优化（动态）→ 执行排序与调度（运行时）
```

---

## 二、依赖关系体系

### 2.1 依赖类型全表

依赖关系定义在 `src/basic/unit-def.h:207-263`，共 **30 种**依赖类型，可分为 6 大类：

#### 第一类：正向拉入依赖（启动时拉入依赖单元）

| 依赖类型 | 反向类型 | 语义 |
|---------|---------|------|
| **Requires** | RequiredBy | 强依赖：启动本单元时**必须**启动依赖单元，依赖失败则本单元也失败 |
| **Requisite** | RequisiteOf | 前提条件：依赖单元**必须已经**处于活跃状态，否则立即失败 |
| **Wants** | WantedBy | 弱依赖：启动本单元时**尝试**启动依赖单元，依赖失败**不影响**本单元 |
| **BindsTo** | BoundBy | 强绑定：比 Requires 更强，依赖单元**停止**则本单元也**被停止** |
| **PartOf** | ConsistsOf | 部分关系：随依赖一起 restart/stop，但不拉入启动 |
| **Upholds** | UpheldBy | 持续维持：只要本单元活跃，就确保依赖也活跃（自动重启） |

#### 第二类：冲突依赖

| 依赖类型 | 反向类型 | 语义 |
|---------|---------|------|
| **Conflicts** | ConflictedBy | 互斥：启动一个会停止另一个 |

#### 第三类：排序依赖（不创建拉入关系，仅控制执行顺序）

| 依赖类型 | 反向类型 | 语义 |
|---------|---------|------|
| **Before** | After | 本单元先于依赖单元启动，晚于依赖单元停止 |
| **After** | Before | 本单元晚于依赖单元启动，先于依赖单元停止 |

#### 第四类：事件触发

| 依赖类型 | 反向类型 | 语义 |
|---------|---------|------|
| **OnSuccess** | OnSuccessOf | 成功后触发 |
| **OnFailure** | OnFailureOf | 失败后触发 |
| **Triggers** | TriggeredBy | 触发关系（socket → service） |
| **PropagatesReloadTo** | ReloadPropagatedFrom | reload 传播 |
| **PropagatesStopTo** | StopPropagatedFrom | stop 传播 |

#### 第五类：命名空间与 Slice

| 依赖类型 | 反向类型 | 语义 |
|---------|---------|------|
| **JoinsNamespaceOf** | — | 共享命名空间 |
| **InSlice** | SliceOf | cgroup slice 关系 |

#### 第六类：GC 引用

| 依赖类型 | 反向类型 | 语义 |
|---------|---------|------|
| **References** | ReferencedBy | 引用关系（防止被 GC 回收） |

### 2.2 依赖原子（Dependency Atom）

systemd v250 引入了**依赖原子**机制（`unit-dependency-atom.h`），将高层依赖类型分解为**36 个原子行为位**。这是依赖系统的核心抽象层。

每个高层依赖类型是若干原子行为的组合：

```
Requires = PULL_IN_START          （启动时拉入）
         + RETROACTIVE_START_FAIL （追溯启动）
         + PROPAGATE_STOP         （传播停止）
         + PROPAGATE_START_FAILURE（传播启动失败）
         + ADD_STOP_WHEN_UNNEEDED_QUEUE（不需要时停止）

Wants    = PULL_IN_START_IGNORED  （启动时拉入，失败忽略）
         + RETROACTIVE_START_FAIL
         + ADD_STOP_WHEN_UNNEEDED_QUEUE

BindsTo  = PULL_IN_START
         + RETROACTIVE_START_FAIL
         + PROPAGATE_STOP
         + PROPAGATE_START_FAILURE
         + CANNOT_BE_ACTIVE_WITHOUT  （绑定：依赖停止则自己也停止）
         + ADD_CANNOT_BE_ACTIVE_WITHOUT_QUEUE
```

**原子行为分类表**：

| 原子 | 含义 | 用于哪些依赖 |
|------|------|------------|
| PULL_IN_START | 拉入 JOB_START（失败则失败） | Requires, BindsTo |
| PULL_IN_START_IGNORED | 拉入 JOB_START（失败忽略） | Wants, Upholds |
| PULL_IN_VERIFY | 拉入 JOB_VERIFY_ACTIVE | Requisite |
| PULL_IN_STOP | 拉入 JOB_STOP（冲突时） | Conflicts |
| PULL_IN_STOP_IGNORED | 拉入 JOB_STOP（忽略失败） | ConflictedBy |
| PROPAGATE_STOP | 停止时传播 JOB_STOP | Requires, BindsTo, PartOf |
| PROPAGATE_RESTART | 重启时传播 JOB_TRY_RESTART | PartOf |
| CANNOT_BE_ACTIVE_WITHOUT | 依赖不活跃则自己也停止 | BindsTo |
| START_STEADILY | 持续确保依赖活跃 | Upholds |
| BEFORE / AFTER | 排序原子 | Before, After |
| TRIGGERS / TRIGGERED_BY | 触发原子 | Triggers |

### 2.3 依赖存储结构

在 `Unit` 结构体中：

```c
struct Unit {
    Hashmap *dependencies;  // UnitDependency → Set<Unit*>
    ...
};
```

依赖以**哈希表套集合**的形式存储：
- Key = `UnitDependency` 枚举值
- Value = `Set` 集合，包含所有具有该依赖关系的 Unit 指针

添加依赖时同时设置正反向：
```c
unit_add_dependency(a, UNIT_REQUIRES, b, ...)
  → a->dependencies[UNIT_REQUIRES].add(b)
  → b->dependencies[UNIT_REQUIRED_BY].add(a)  // 自动设置反向
```

---

## 三、依赖图的生成

### 3.1 依赖来源

依赖关系从以下途径生成：

| 来源 | 方式 | 示例 |
|------|------|------|
| **单元文件** | 解析 [Unit] 段 | Requires=, After=, Wants= |
| **单元文件** | [Install] 段 enable 时 | WantedBy=, RequiredBy= → 创建 .wants/ 符号链接 |
| **隐式依赖** | 单元类型自动添加 | Service → 自动 After=sysinit.target |
| **路径依赖** | 挂载点依赖 | /home/user → Requires=home.mount, After=home.mount |
| **设备依赖** | udev 设备触发 | DEVICE → BindsTo= |
| **默认目标** | DefaultDependencies=yes | After=basic.target, Before=shutdown.target |
| **Slice** | Slice= 配置 | InSlice 依赖 |
| **Socket 激活** | socket → service | Triggers= |

### 3.2 依赖加载流程

```
unit_load()
  ├── load_fragment()
  │     解析 .service/.socket 等配置文件
  │     → config_parse_unit_deps()  解析 Requires=/Wants=/After= 等
  │     → unit_add_dependency_by_name()
  │         → unit_add_dependency()
  │             → 双向设置 dependencies hashmap
  │
  ├── 类型特有加载（如 service_load()）
  │     → service_add_default_dependencies()
  │         添加 Requires=sysinit.target
  │         添加 After=sysinit.target, basic.target
  │         添加 Conflicts=shutdown.target
  │         添加 Before=shutdown.target
  │
  ├── unit_add_mount_dependencies()
  │     扫描 ExecContext 中的所有路径
  │     → 对每个挂载点创建 Requires= + After= 依赖
  │
  └── unit_add_default_target_dependency()
       非 target 单元 → After=target（如果 target 有 DefaultDependencies=yes）
```

### 3.3 可视化依赖图

systemd 提供内置的依赖图生成：

```bash
# 文本形式
systemctl list-dependencies sshd.service

# DOT 图形（可用 graphviz 渲染）
systemd-analyze dot sshd.service | dot -Tsvg > deps.svg

# 反向依赖
systemctl list-dependencies --reverse sshd.service
```

---

## 四、事务系统（Transaction）

### 4.1 事务的概念

当用户执行 `systemctl start foo.service` 时，systemd 不是简单地启动一个服务——它需要：
1. 找出所有需要一起启动/停止的单元
2. 检查是否有冲突
3. 确定执行顺序
4. 优化掉多余的操作

这个过程通过**事务（Transaction）**机制完成。事务是一个临时的 Job 集合，经过多步优化后，被安装到 Manager 的运行队列中。

### 4.2 Transaction 结构体

```c
typedef struct Transaction {
    Hashmap *jobs;        // Unit* → Job*，事务中的所有 Job
    Job *anchor_job;      // 锚点 Job（用户直接请求的那个）
    bool irreversible;    // 是否不可逆
} Transaction;
```

### 4.3 事务构建流程（10 步算法）

事务激活的完整流程定义在 `transaction_activate()`（`transaction.c:690-793`）：

```
┌─────────────────────────────────────────────────────────────────┐
│              transaction_activate() 十步算法                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  第 1 步：标记重要 Job                                            │
│  transaction_find_jobs_that_matter_to_anchor()                    │
│  → 从 anchor_job 递归标记所有 matters_to_anchor=true 的 Job       │
│  → 不重要的 Job 可以被后续步骤安全删除                               │
│                                                                  │
│  第 2 步：最小化影响（仅 JOB_FAIL 模式）                           │
│  transaction_minimize_impact()                                    │
│  → 删除会停止运行中服务的非必要 Job                                 │
│  → 删除会改变已有 Job 的非必要 Job                                 │
│                                                                  │
│  第 3 步：删除冗余 Job                                            │
│  transaction_drop_redundant()                                     │
│  → 删除已经满足的 Job（如 start 一个已 active 的服务）              │
│                                                                  │
│  ┌─ 循环开始 ──────────────────────────────────────────┐          │
│  │                                                       │         │
│  │  第 4 步：垃圾回收                                     │         │
│  │  transaction_collect_garbage()                         │         │
│  │  → 删除没有任何其他 Job 依赖的孤立 Job                  │         │
│  │                                                       │         │
│  │  第 5 步：验证排序（检测循环依赖）                       │         │
│  │  transaction_verify_order()                            │         │
│  │  → 遍历排序图，检测循环                                │         │
│  │  → 如果发现循环 → 尝试删除非必要 Job 打破循环            │         │
│  │  → 如果无法打破 → 返回错误                             │         │
│  │  → 成功则跳出循环                                     │         │
│  │                                                       │         │
│  └─ 循环结束（如果有循环被打破则重试）────────────────────┘          │
│                                                                  │
│  ┌─ 循环开始 ──────────────────────────────────────────┐          │
│  │                                                       │         │
│  │  第 6 步：合并 Job                                     │         │
│  │  transaction_merge_jobs()                              │         │
│  │  → 同一 Unit 上可能有多个 Job → 尝试合并               │         │
│  │  → 例如 START + RESTART → RESTART                     │         │
│  │  → 如果无法合并（如 START + STOP）→ 尝试删除非必要的    │         │
│  │                                                       │         │
│  │  第 7 步：垃圾回收删除后的依赖                          │         │
│  │  transaction_collect_garbage()                         │         │
│  │                                                       │         │
│  └─ 循环结束（如果有无法合并的 Job 被删除则重试）──────────┘          │
│                                                                  │
│  第 8 步：再次删除冗余                                            │
│  transaction_drop_redundant()                                     │
│                                                                  │
│  第 9 步：检查破坏性                                              │
│  transaction_is_destructive()                                     │
│  → 在 JOB_FAIL 模式下，检查是否会替换现有 Job                      │
│  → 如果会 → 返回错误                                              │
│                                                                  │
│  第 10 步：应用事务                                               │
│  transaction_apply()                                              │
│  → 将所有 Job 安装到 Manager 的活跃 Job 哈希表                    │
│  → 将每个 Job 加入 run_queue                                      │
│  → 启动超时定时器                                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 依赖递归展开

`transaction_add_job_and_dependencies()`（`transaction.c:899-1098`）是依赖展开的核心。以 JOB_START 为例：

```
请求 START unit A
  │
  ├── 遍历 A 的 PULL_IN_START 原子依赖（Requires/BindsTo）
  │     → 对每个依赖 B: 递归 transaction_add_job_and_dependencies(START, B)
  │     → 失败则整个事务失败
  │
  ├── 遍历 A 的 PULL_IN_START_IGNORED 原子依赖（Wants/Upholds）
  │     → 对每个依赖 B: 递归添加 START
  │     → 失败则忽略，记录日志
  │
  ├── 遍历 A 的 PULL_IN_VERIFY 原子依赖（Requisite）
  │     → 对每个依赖 B: 递归添加 VERIFY_ACTIVE
  │     → 如果 B 不活跃 → 事务失败
  │
  ├── 遍历 A 的 PULL_IN_STOP 原子依赖（Conflicts）
  │     → 对每个冲突单元 C: 递归添加 STOP
  │     → 失败则事务失败
  │
  └── 遍历 A 的 PULL_IN_STOP_IGNORED 原子依赖
       → 对每个单元: 递归添加 STOP
       → 失败则忽略
```

JOB_STOP 时的传播：

```
请求 STOP unit A
  │
  ├── 遍历 A 的 PROPAGATE_STOP 原子依赖
  │     → 对每个依赖 B: 递归添加 STOP
  │     （PartOf=A, Requires=A, BindsTo=A 的单元也被停止）
  │
  └── 遍历 A 的 PROPAGATE_RESTART 原子依赖
       → 对每个依赖 B: 递归添加 TRY_RESTART
```

---

## 五、循环依赖检测与处理

### 5.1 检测算法

`transaction_verify_order_one()`（`transaction.c:349-479`）使用**DFS 深度优先搜索**检测排序图中的循环：

```
transaction_verify_order_one(job j, from, generation)
  │
  ├── 如果 j.generation == generation（之前已访问过）
  │     └── 发现循环！
  │           ├── 沿 marker 回溯找到循环路径
  │           ├── 在循环中找到一个不重要的 Job（!matters_to_anchor）
  │           ├── 删除该 Job 打破循环
  │           └── 返回 -EAGAIN → 外层重试
  │
  ├── 设置 j.marker = from（记录路径）
  ├── 设置 j.generation = generation
  │
  ├── 遍历 j 的 BEFORE/AFTER 排序依赖
  │     对每个有 Job 的邻居 o:
  │       ├── 调用 job_compare(j, o, direction)
  │       │     确认 j 确实在 o "之前"
  │       └── 递归 transaction_verify_order_one(o, j, generation)
  │
  └── 清除 j.marker = NULL（回溯）
```

### 5.2 循环处理策略

```
发现循环 A → B → C → A
  │
  ├── 在循环中寻找 !matters_to_anchor 的 Job
  │     （非锚点 Job 的直接/间接依赖）
  │
  ├── 如果找到 → 删除该 Job，返回 -EAGAIN 重试
  │     日志: "Job X/start deleted to break ordering cycle"
  │
  └── 如果找不到 → 循环不可打破
       返回错误: "Transaction order is cyclic"
```

---

## 六、Job 执行排序

### 6.1 排序规则（job_compare）

`job_compare()`（`job.c:1615-1638`）定义了**两个 Job 的执行先后顺序**。给定 `b.After=a` 的单元排序关系：

| a 的操作 | b 的操作 | 执行顺序 | 原因 |
|---------|---------|---------|------|
| start | start | 先 a，后 b | a 先启动，b 在 a 之后 |
| start | stop | 先 stop b，后 start a | 先把冲突的停掉 |
| stop | start | 先 stop a，后 start b | 先停掉旧的，再启动新的 |
| stop | stop | 先 stop b，后 stop a | 反向停止（依赖的先停） |

**核心规则**：
- **STOP 总是优先于 START**（在有依赖的情况下）
- **启动顺序**：按 After/Before 正向
- **停止顺序**：按 After/Before 反向（先停依赖者，再停被依赖者）

### 6.2 运行队列调度

```c
// manager.c → manager_dispatch_run_queue()
static int manager_dispatch_run_queue(sd_event_source *source, void *userdata) {
    Manager *m = userdata;

    while ((j = prioq_peek(m->run_queue))) {
        // 检查 Job 的所有排序依赖是否满足
        // 如果有 BEFORE 的 Job 还未完成 → 跳过，保持在队列中
        // 否则 → 执行

        job_run_and_invalidate(j);
    }
}
```

### 6.3 完整执行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     Job 执行生命周期                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 事务安装阶段                                                  │
│     transaction_apply()                                          │
│     → job_install(j)          安装到 Unit 上                     │
│     → job_add_to_run_queue(j) 加入运行队列                       │
│     → job_start_timer(j)      启动超时定时器                     │
│                                                                  │
│  2. 等待阶段（WAITING 状态）                                      │
│     Job 在 run_queue 中等待排序依赖满足                           │
│     → 检查 Before 关系中的其他 Job 是否已完成                     │
│     → 如果依赖的 Job 还在 WAITING/RUNNING → 继续等待             │
│                                                                  │
│  3. 执行阶段（RUNNING 状态）                                      │
│     manager_dispatch_run_queue()                                  │
│     → job_run_and_invalidate(j)                                  │
│       → 根据 Job 类型调用 Unit vtable:                           │
│           JOB_START   → unit_start(u)   → service_start()        │
│           JOB_STOP    → unit_stop(u)    → service_stop()         │
│           JOB_RELOAD  → unit_reload(u)  → service_reload()       │
│           JOB_RESTART → 先 stop 再 start                        │
│                                                                  │
│  4. 完成阶段                                                     │
│     unit 状态变更（SIGCHLD、sd_notify 等触发）                    │
│     → job_finish_and_invalidate(j, result)                       │
│       ├── result = JOB_DONE    → 成功                            │
│       ├── result = JOB_FAILED  → 失败                            │
│       ├── result = JOB_TIMEOUT → 超时                            │
│       ├── result = JOB_DEPENDENCY → 依赖失败                     │
│       └── 传播结果给依赖的 Job                                    │
│           → 如果 START 失败且有 PROPAGATE_START_FAILURE           │
│             → 依赖此单元的 Job 也标记失败                         │
│                                                                  │
│  5. 后续触发                                                     │
│     → 已完成的 Job 被移除                                         │
│     → run_queue 中等待的 Job 被重新评估                           │
│     → 排序依赖已满足的 Job 可以开始执行                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 七、Job 模式（JobMode）

事务的行为受 JobMode 影响：

| 模式 | 含义 | 使用场景 |
|------|------|---------|
| **replace** | 替换已有冲突 Job | 默认模式（systemctl start） |
| **fail** | 如果有冲突 Job 则失败 | 保守模式 |
| **replace-irreversibly** | 替换且标记不可逆 | 关机/重启 |
| **isolate** | 仅启动目标及其依赖，停止所有其他 | systemctl isolate |
| **flush** | 清除所有已有 Job 后执行 | 紧急操作 |
| **ignore-dependencies** | 忽略依赖关系 | 强制操作 |
| **ignore-requirements** | 忽略拉入依赖但保留排序 | 半强制操作 |
| **triggering** | 启动触发此单元的所有单元 | 反向触发 |

---

## 八、Job 类型合并规则

当事务中同一 Unit 出现多个 Job 时，需要合并：

```
合并矩阵 (job_type_merge_and_collapse):

           ┌─────────┬─────────┬─────────┬─────────┬─────────┐
           │  START  │  STOP   │ RELOAD  │ RESTART │VERIFY_ACT│
    ───────┼─────────┼─────────┼─────────┼─────────┼──────────┤
    START  │ START   │ 冲突!   │ RELOAD_S│ RESTART │  START   │
    STOP   │ 冲突!   │ STOP    │ 冲突!   │ 冲突!   │  冲突!   │
    RELOAD │ RELOAD_S│ 冲突!   │ RELOAD  │ RESTART │  RELOAD  │
    RESTART│ RESTART │ 冲突!   │ RESTART │ RESTART │  RESTART │
    VERIFY │ START   │ 冲突!   │ RELOAD  │ RESTART │  VERIFY  │
    ───────┴─────────┴─────────┴─────────┴─────────┴──────────┘

    RELOAD_S = RELOAD_OR_START（如果没运行则 START，运行中则 RELOAD）
    冲突! = 无法合并，事务会尝试删除不重要的 Job
```

---

## 九、实例分析

### 9.1 启动 sshd.service 的完整过程

假设 sshd.service 的依赖配置：

```ini
[Unit]
Requires=network.target
After=network.target sshd-keygen.target
Wants=sshd-keygen.target
```

**事务构建过程**：

```
Step 1: 用户请求 → transaction_add_job_and_dependencies(START, sshd.service)
  │
  ├── 添加 Job: START sshd.service  (anchor_job)
  │
  ├── PULL_IN_START (Requires):
  │     → START network.target
  │       └── 递归展开 network.target 的依赖...
  │
  ├── PULL_IN_START_IGNORED (Wants):
  │     → START sshd-keygen.target (失败则忽略)
  │
  └── PULL_IN_STOP (Conflicts): 无

Step 2-8: 优化事务
  → 删除已满足的 Job (如 network.target 已 active)
  → 检查循环依赖
  → 合并重复 Job

Step 9: 检查破坏性
Step 10: 安装到 run_queue
```

**执行顺序（基于 After/Before）**：

```
时间线 →
  ┌──────────────────┐
  │ network.target   │  ← 首先（sshd After network）
  │ (如果需要启动)     │
  └────────┬─────────┘
           │
  ┌────────┴─────────┐
  │ sshd-keygen.tgt  │  ← 然后（sshd After sshd-keygen）
  └────────┬─────────┘
           │
  ┌────────┴─────────┐
  │ sshd.service     │  ← 最后
  └──────────────────┘
```

### 9.2 关机过程的依赖分析

```
systemctl poweroff
  │
  └── isolate poweroff.target
       → 启动 poweroff.target 及其依赖
       → 停止所有非 ignore_on_isolate 的单元

执行顺序（逆序停止）：
  1. 停止用户服务 (user@.service)
  2. 停止系统服务 (sshd.service, nginx.service, ...)
  3. 停止网络 (networkd, resolved)
  4. 卸载文件系统 (*.mount)
  5. 停止 swap (*.swap)
  6. 执行 systemd-poweroff.service
  7. 内核关机
```

---

## 十、调试工具

```bash
# 查看单元的依赖树
systemctl list-dependencies --all sshd.service

# 查看反向依赖（谁依赖了这个单元）
systemctl list-dependencies --reverse sshd.service

# 生成 DOT 格式依赖图
systemd-analyze dot sshd.service | dot -Tpng > deps.png

# 查看启动关键路径
systemd-analyze critical-chain

# 查看启动时序
systemd-analyze plot > boot.svg

# 验证单元文件
systemd-analyze verify foo.service

# 查看某个单元的所有属性（包括依赖）
systemctl show sshd.service

# 查看当前所有活跃 Job
systemctl list-jobs

# 模拟启动（不实际执行）
systemd-analyze dot --order sshd.service
```
