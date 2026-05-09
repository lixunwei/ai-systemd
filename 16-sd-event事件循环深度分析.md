# sd-event 事件循环深度分析

> sd-event 是 systemd 的核心事件循环库，封装了 epoll、timerfd、signalfd、pidfd、
> inotify 等 Linux 内核机制，提供统一的事件驱动编程模型。PID 1 以及 journald、
> logind、networkd 等所有子系统都基于 sd-event 构建。

---

## 一、核心数据结构

### 1.1 sd_event — 事件循环主体

`sd-event.c:93-157`：

```c
struct sd_event {
    unsigned n_ref;                    // 引用计数

    int epoll_fd;                      // epoll 实例 FD
    int watchdog_fd;                   // 看门狗 timerfd

    Prioq *pending;                    // 待分发事件源优先队列
    Prioq *prepare;                    // prepare 回调优先队列

    /* 5 种时钟各一个 timerfd */
    struct clock_data realtime;        // CLOCK_REALTIME
    struct clock_data boottime;        // CLOCK_BOOTTIME
    struct clock_data monotonic;       // CLOCK_MONOTONIC
    struct clock_data realtime_alarm;  // CLOCK_REALTIME_ALARM
    struct clock_data boottime_alarm;  // CLOCK_BOOTTIME_ALARM

    usec_t perturb;                    // 定时器扰动值（基于 boot ID）

    sd_event_source **signal_sources;  // 信号源数组（按信号编号索引）
    Hashmap *signal_data;              // signalfd（按优先级索引）

    Hashmap *child_sources;            // 子进程源（按 PID 索引）
    unsigned n_online_child_sources;   // 活跃子进程源数

    Set *post_sources;                 // POST 类型事件源集合
    Prioq *exit;                       // EXIT 类型事件源优先队列

    Hashmap *inotify_data;             // inotify 实例（按优先级索引）

    uint64_t iteration;                // 迭代计数器
    triple_timestamp timestamp;        // 当前迭代时间戳
    int state;                         // 循环状态机

    bool exit_requested:1;             // 退出请求标志
    bool need_process_child:1;         // 需要处理子进程
    bool watchdog:1;                   // 看门狗启用

    int exit_code;                     // 退出码
    unsigned n_sources;                // 事件源总数

    struct epoll_event *event_queue;   // epoll_wait 结果缓冲区
};
```

### 1.2 sd_event_source — 事件源

`event-source.h:48-132`：

```c
struct sd_event_source {
    WakeupType wakeup;           // 唤醒类型标识（epoll 分发用）
    unsigned n_ref;              // 引用计数
    sd_event *event;             // 所属事件循环
    void *userdata;              // 用户数据
    sd_event_handler_t prepare;  // prepare 回调

    EventSourceType type;        // 事件源类型（13种）
    signed int enabled:3;        // 启用状态（OFF/ON/ONESHOT）
    bool pending:1;              // 是否待分发
    bool dispatching:1;          // 是否正在分发
    bool floating:1;             // 浮动引用（不持有 event ref）
    bool exit_on_failure:1;      // 回调失败时退出循环
    bool ratelimited:1;          // 是否被频率限制

    int64_t priority;            // 优先级（数值越小越优先）
    unsigned pending_index;      // pending 队列索引
    unsigned prepare_index;      // prepare 队列索引
    uint64_t pending_iteration;  // 变为 pending 时的迭代号
    uint64_t prepare_iteration;  // 上次 prepare 的迭代号

    RateLimit rate_limit;        // 频率限制器

    /* 类型特定数据（联合体） */
    union {
        struct { /* IO */
            sd_event_io_handler_t callback;
            int fd;
            uint32_t events, revents;
            bool registered:1, owned:1;
        } io;

        struct { /* Time */
            sd_event_time_handler_t callback;
            usec_t next, accuracy;     // 触发时间和精度
        } time;

        struct { /* Signal */
            sd_event_signal_handler_t callback;
            struct signalfd_siginfo siginfo;
            int sig;
        } signal;

        struct { /* Child */
            sd_event_child_handler_t callback;
            siginfo_t siginfo;
            pid_t pid;
            int options;
            int pidfd;                 // pidfd（如果支持）
            bool registered:1;         // pidfd 已注册到 epoll
            bool pidfd_owned:1;        // 关闭时释放 pidfd
            bool process_owned:1;      // 关闭时 kill 进程
        } child;

        struct { sd_event_handler_t callback; } defer;
        struct { sd_event_handler_t callback; } post;
        struct { sd_event_handler_t callback; unsigned prioq_index; } exit;
        struct { /* Inotify */
            sd_event_inotify_handler_t callback;
            uint32_t mask;
            struct inode_data *inode_data;
        } inotify;
    };
};
```

### 1.3 clock_data — 每时钟定时器数据

`event-source.h:134-150`：

```c
struct clock_data {
    WakeupType wakeup;     // WAKEUP_CLOCK_DATA
    int fd;                // timerfd 文件描述符

    Prioq *earliest;       // 最早触发时间优先队列
    Prioq *latest;         // 最晚触发时间优先队列
    usec_t next;           // 当前设定的 timerfd 唤醒时间

    bool needs_rearm:1;    // 是否需要重新设置 timerfd
};
```

---

## 二、事件源类型

`event-source.h:16-32` 定义了 13 种事件源类型：

```
类型                        │ 机制         │ 用途
────────────────────────────┼──────────────┼────────────────────
SOURCE_IO                   │ epoll        │ 文件描述符 I/O 监控
SOURCE_TIME_REALTIME        │ timerfd      │ 墙上时钟定时
SOURCE_TIME_BOOTTIME        │ timerfd      │ 启动后时钟定时
SOURCE_TIME_MONOTONIC       │ timerfd      │ 单调时钟定时
SOURCE_TIME_REALTIME_ALARM  │ timerfd      │ RTC 唤醒定时
SOURCE_TIME_BOOTTIME_ALARM  │ timerfd      │ RTC 唤醒定时
SOURCE_SIGNAL               │ signalfd     │ Unix 信号处理
SOURCE_CHILD                │ pidfd/SIGCHLD│ 子进程状态监控
SOURCE_DEFER                │ 软件触发     │ 延迟执行
SOURCE_POST                 │ 软件触发     │ 每次分发后执行
SOURCE_EXIT                 │ 软件触发     │ 循环退出时执行
SOURCE_WATCHDOG             │ timerfd      │ 看门狗喂狗
SOURCE_INOTIFY              │ inotify      │ 文件系统监控
```

**WakeupType** — epoll 中的分发标识：

```
WAKEUP_EVENT_SOURCE  → IO 源或 pidfd 源的 epoll 事件
WAKEUP_CLOCK_DATA    → timerfd 可读事件
WAKEUP_SIGNAL_DATA   → signalfd 可读事件
WAKEUP_INOTIFY_DATA  → inotify fd 可读事件
```

---

## 三、事件循环状态机

```c
enum {
    SD_EVENT_INITIAL,      // 初始状态
    SD_EVENT_PREPARING,    // 正在执行 prepare 回调
    SD_EVENT_ARMED,        // 已设置好 timerfd/epoll，准备等待
    SD_EVENT_PENDING,      // 有事件待分发
    SD_EVENT_RUNNING,      // 正在分发事件
    SD_EVENT_EXITING,      // 正在执行退出回调
    SD_EVENT_FINISHED,     // 循环结束
};
```

```
INITIAL ──> PREPARING ──> ARMED ──> (epoll_wait) ──> PENDING ──> RUNNING ──> INITIAL
                                                                    │
                                                               exit_requested
                                                                    │
                                                                    ▼
                                                               EXITING ──> FINISHED
```

---

## 四、主循环详解

### 4.1 sd_event_loop() — 外层循环

`sd-event.c:4254-4271`：

```c
int sd_event_loop(sd_event *e) {
    while (e->state != SD_EVENT_FINISHED) {
        r = sd_event_run(e, UINT64_MAX);
        if (r < 0) return r;
    }
    return e->exit_code;
}
```

### 4.2 sd_event_run() — 单次迭代

`sd-event.c:4206-4252`：

```
sd_event_run(e, timeout)
  │
  ├── sd_event_prepare(e)
  │     │
  │     ├── e->iteration++
  │     ├── event_prepare(e)     ← 执行所有 prepare 回调
  │     ├── event_arm_timer()×5  ← 设置 5 个时钟的 timerfd
  │     ├── event_close_inode_data_fds()
  │     │
  │     ├── 有 pending 事件?
  │     │     → 是: 直接调用 sd_event_wait(e, 0)
  │     │     → 否: 设置状态 ARMED
  │     │
  │     └── 返回 0（无 pending）或 >0（有 pending）
  │
  ├── r == 0 → sd_event_wait(e, timeout)
  │     → 阻塞等待 epoll 事件
  │     → 返回 >0 表示有 pending 事件
  │
  └── r > 0 → sd_event_dispatch(e)
        → 分发一个最高优先级的 pending 事件源
        → 返回
```

### 4.3 sd_event_wait() — 等待事件

`sd-event.c:4063-4163`：

```
sd_event_wait(e, timeout)
  │
  ├── exit_requested → 直接返回 PENDING
  │
  ├── 【防饥饿循环】 threshold = INT64_MAX
  │     │
  │     ├── process_epoll(e, timeout, threshold)
  │     │     → epoll_wait() 获取内核事件
  │     │     → 仅处理 priority ≤ threshold 的事件
  │     │     → 返回本轮最小 priority
  │     │
  │     ├── process_child(e, threshold)
  │     │     → waitid() 收割子进程
  │     │     → 仅处理 priority ≤ threshold 的子进程事件
  │     │
  │     ├── threshold = min(epoll_min, child_min)
  │     │     → 收紧阈值，重新 epoll_wait(timeout=0)
  │     │     → 确保高优先级事件不被遗漏
  │     │
  │     └── 无新事件 → 跳出循环
  │
  ├── process_watchdog(e)     ← 看门狗喂狗
  ├── process_inotify(e)      ← 处理 inotify 事件
  ├── process_timer() × 5     ← 处理 5 个时钟的到期事件
  │
  └── 有 pending 事件 → 返回 PENDING
```

**防饥饿设计**：`sd_event_wait()` 在处理 epoll 和 child 事件后，会用更严格的
threshold 重新检查，确保在 process_child() 期间到达的高优先级 IO 事件不被遗漏。

### 4.4 sd_event_dispatch() — 分发事件

`sd-event.c:4165-4191`：

```
sd_event_dispatch(e)
  │
  ├── exit_requested → dispatch_exit(e)
  │     → 从 exit 优先队列取最高优先级退出源
  │     → 执行退出回调
  │     → 设置 SD_EVENT_FINISHED
  │
  └── event_next_pending(e)
        → 从 pending 优先队列取最高优先级事件源
        → source_dispatch(p)
```

---

## 五、优先级排序

### 5.1 pending 队列排序规则

`sd-event.c:168-192`，四级排序：

```
1. 启用状态：enabled ≠ OFF 的排前面
2. 频率限制：非 ratelimited 的排前面
3. 优先级值：priority 数值小的排前面
4. 到达顺序：pending_iteration 小的排前面（先到先分发）
```

### 5.2 prepare 队列排序规则

`sd-event.c:194-220`，四级排序：

```
1. 启用状态：enabled ≠ OFF 的排前面
2. 频率限制：非 ratelimited 的排前面
3. 上次执行：prepare_iteration 小的排前面（最久未执行的先来）
4. 优先级值：priority 小的作为平局决胜
```

### 5.3 优先级在 systemd 中的典型使用

```
SD_EVENT_PRIORITY_IMPORTANT    = -100  // 关键事件（SIGCHLD）
SD_EVENT_PRIORITY_NORMAL       = 0     // 普通事件
SD_EVENT_PRIORITY_IDLE         = 100   // 空闲时执行

PID 1 中的典型设置：
  SIGCHLD 处理    → priority = -6  (比 IO 高)
  D-Bus 消息      → priority = 0   (普通)
  cgroup 清理     → priority = 100 (空闲时)
```

---

## 六、定时器实现

### 6.1 架构：每时钟一个 timerfd

```
sd_event
  ├── monotonic.fd  → timerfd(CLOCK_MONOTONIC)  → epoll
  ├── realtime.fd   → timerfd(CLOCK_REALTIME)   → epoll
  ├── boottime.fd   → timerfd(CLOCK_BOOTTIME)   → epoll
  ├── realtime_alarm.fd  → timerfd(CLOCK_REALTIME_ALARM) → epoll
  └── boottime_alarm.fd  → timerfd(CLOCK_BOOTTIME_ALARM) → epoll

每个 clock_data 中：
  earliest 优先队列 → 所有同类定时器中最早触发时间
  latest 优先队列   → 所有同类定时器中最晚必须触发时间
```

### 6.2 定时器合并（Coalescing）

`event_arm_timer()`（`sd-event.c:3057-3113`）：

```
event_arm_timer(e, d)
  │
  ├── a = prioq_peek(d->earliest)  → 最近要触发的事件源
  │     earliest_time = a->time.next
  │
  ├── b = prioq_peek(d->latest)    → 最晚窗口截止
  │     latest_time = b->time.next + b->time.accuracy
  │
  ├── t = sleep_between(e, earliest_time, latest_time)
  │     → 在 [earliest, latest] 窗口中选择一个唤醒点
  │
  └── timerfd_settime(d->fd, TFD_TIMER_ABSTIME, t)
```

### 6.3 sleep_between() — 智能唤醒点选择

`sd-event.c:2977-3055`：

```
sleep_between(e, a, b)
  // 目标：在 [a, b] 区间内选择一个唤醒时间
  // 策略：系统级对齐，减少唤醒次数

  尝试对齐到 1 分钟边界 + boot_id 扰动
    → 成功 → 返回（全系统同时唤醒，节能）

  尝试对齐到 10 秒边界 + 扰动
    → 成功 → 返回

  尝试对齐到 1 秒边界 + 扰动
    → 成功 → 返回

  尝试对齐到 250 毫秒边界 + 扰动
    → 成功 → 返回

  兜底 → 返回 b（最晚时间）
```

**扰动值**（perturb）：从 `/proc/sys/kernel/random/boot_id` 派生，
确保同一台机器上所有 sd-event 实例在相同的偏移点唤醒，
而不同机器唤醒时间不同（避免雷暴效应）。

### 6.4 精度（accuracy）

```c
// sd_event_add_time() 默认精度
if (accuracy == 0)
    accuracy = DEFAULT_ACCURACY_USEC;  // 250ms

// latest = next + accuracy
// 即定时器可在 [next, next+accuracy] 范围内触发
```

---

## 七、信号处理

### 7.1 signalfd 架构

```
每个优先级一个 signalfd：
  signal_data[priority_A] → signalfd(mask_A) → epoll
  signal_data[priority_B] → signalfd(mask_B) → epoll
  ...

sd_event_add_signal(sig=SIGTERM, priority=-100)
  → 将 SIGTERM 加入 priority=-100 的 signalfd 掩码
  → sigprocmask(SIG_BLOCK, SIGTERM)  阻塞传统信号处理
```

### 7.2 信号分发流程

```
epoll_wait() → signalfd 可读
  │
  ├── 识别 WakeupType = WAKEUP_SIGNAL_DATA
  │
  ├── process_signal(e, signal_data)
  │     → read(signalfd) → siginfo
  │     → 查找 signal_sources[signo]
  │     → 填充 s->signal.siginfo
  │     → source_set_pending(s, true)
  │
  └── 如果是 SIGCHLD → e->need_process_child = true
```

### 7.3 SIGCHLD 特殊处理

SIGCHLD 不仅触发 signal 源，还触发 child 源：

```
process_signal() 发现 SIGCHLD
  → 设置 need_process_child = true
  → process_child() 被调用

process_child(e, threshold)
  遍历所有 child_sources：
    如果 child.priority ≤ threshold：
      waitid(P_PID, pid, WNOHANG|WNOWAIT)
      如果有状态变更：
        填充 s->child.siginfo
        source_set_pending(s, true)
```

---

## 八、子进程监控（pidfd 支持）

### 8.1 pidfd vs SIGCHLD 模式

```c
// sd_event_add_child() 尝试顺序：
1. pidfd_open(pid) → 成功 → 使用 pidfd 模式
   → epoll_ctl(ADD, pidfd, EPOLLIN)
   → 进程退出时 pidfd 变为可读

2. pidfd 不可用 → 使用 SIGCHLD 模式
   → 依赖 signalfd 接收 SIGCHLD
   → process_child() 中 waitid() 轮询
```

### 8.2 pidfd 分发

```
epoll_wait() → pidfd 可读
  │
  ├── 识别 WakeupType = WAKEUP_EVENT_SOURCE
  │     且 type = SOURCE_CHILD
  │
  └── process_pidfd(e, s)
        → waitid(P_PID, pid, WNOHANG|WNOWAIT)
        → source_set_pending(s, true)
```

---

## 九、source_dispatch() — 分发引擎

`sd-event.c:3540-3697`：

```
source_dispatch(s)
  │
  ├── [1] 频率限制检查
  │     如果触发太频繁 → 转入 ratelimited 模式
  │     → 变成等效的 CLOCK_MONOTONIC 定时器
  │     → ratelimit 窗口结束后恢复
  │
  ├── [2] 清除 pending 标志（defer 和 exit 除外）
  │
  ├── [3] POST 源标记
  │     如果当前不是 POST 源 → 将所有 POST 源标记为 pending
  │     → 保证每次非 POST 分发后都执行 POST 回调
  │
  ├── [4] ONESHOT 处理
  │     enabled == SD_EVENT_ONESHOT → 自动禁用
  │
  ├── [5] 类型分发
  │     ├── IO      → callback(s, fd, revents, userdata)
  │     ├── TIME_*  → callback(s, next_time, userdata)
  │     ├── SIGNAL  → callback(s, &siginfo, userdata)
  │     ├── CHILD   → callback(s, &siginfo, userdata)
  │     │             → 如果子进程是 zombie → waitid() 收割
  │     ├── DEFER   → callback(s, userdata)
  │     ├── POST    → callback(s, userdata)
  │     ├── EXIT    → callback(s, userdata)
  │     └── INOTIFY → callback(s, &inotify_event, userdata)
  │
  ├── [6] 错误处理
  │     回调返回 < 0：
  │     ├── exit_on_failure → sd_event_exit()
  │     └── 否则 → 禁用该事件源
  │
  └── [7] 引用计数检查
        n_ref == 0 → source_free()
```

---

## 十、Prepare 回调机制

`event_prepare()`（`sd-event.c:3699-3738`）：

```
event_prepare(e)
  │
  循环取 prepare 队列顶部事件源：
  │
  ├── 已在本迭代执行过 (prepare_iteration == iteration) → 结束
  │
  ├── 标记 prepare_iteration = iteration
  │     → 重排队列（已执行的沉底）
  │
  └── 调用 s->prepare(s, userdata)
        → prepare 回调可以：
          - 修改自身优先级
          - 启用/禁用自身
          - 添加新事件源
          - 设置自身为 pending
```

**用途**：prepare 在每次迭代开始时执行，允许事件源在实际分发前做准备工作。
例如 D-Bus 可以在 prepare 中检查写缓冲区，决定是否需要监听 EPOLLOUT。

---

## 十一、IO 事件处理

```c
// 注册
sd_event_add_io(e, &source, fd, EPOLLIN|EPOLLOUT, callback, userdata)
  → epoll_ctl(EPOLL_CTL_ADD, fd, events)

// epoll 事件到达时
process_io(e, s, revents)
  → 如果已 pending → revents |= 新事件（合并）
  → 否则 → revents = 新事件
  → source_set_pending(s, true)

// EPOLLONESHOT 场景
// IO 事件只触发一次，需要重新 arm
// sd-event 通过合并 revents 来正确处理
```

---

## 十二、Inotify 事件处理

### 12.1 数据结构层次

```
sd_event
  └── inotify_data (per-priority)
        ├── fd (inotify 实例)
        ├── buffer (事件缓冲区)
        └── inode_data (per-watched-inode)
              ├── wd (watch descriptor)
              ├── path
              └── event_sources[] (同一 inode 的多个监控源)
```

### 12.2 共享 inotify 实例

同一优先级的所有 inotify 事件源共享一个 inotify 实例。
同一 inode 的多个事件源共享一个 watch descriptor。
这大大减少了 FD 使用量。

---

## 十三、看门狗集成

```c
// sd_event 自动看门狗（sd-event.c:3778-3812）
// 在每次 wait 后检查是否需要喂狗

process_watchdog(e)
  如果 distance(watchdog_last, now) > period/4：
    sd_notify(false, "WATCHDOG=1")  → 通知 PID 1
    watchdog_last = now
    arm_watchdog(e)  → 在 [period/2, period*3/4] 范围内设定下次喂狗
```

---

## 十四、频率限制（RateLimit）

任何事件源都可以设置频率限制：

```c
sd_event_source_set_ratelimit(s, interval, burst)
  → 设置 s->rate_limit = { interval, burst }

// 触发频率检查（source_dispatch 入口）
if (!ratelimit_below(&s->rate_limit))
    event_source_enter_ratelimited(s)
      → 将事件源转换为等效的 CLOCK_MONOTONIC 定时器
      → 在 ratelimit 窗口结束时触发 ratelimit_expire_callback
      → 回调中恢复原始事件源类型
```

---

## 十五、完整架构图

```
                         ┌─────────────────────────┐
                         │     sd_event 事件循环     │
                         └────────────┬────────────┘
                                      │
            ┌────── sd_event_run() ────┤
            │                         │
    ┌───────▼───────┐         ┌───────▼───────┐         ┌───────────────┐
    │ sd_event_     │         │ sd_event_     │         │ sd_event_     │
    │ prepare()     │────────>│ wait()        │────────>│ dispatch()    │
    └───────────────┘         └───────────────┘         └───────────────┘
    │                         │                         │
    ├─ prepare 回调           ├─ epoll_wait()           ├─ event_next_pending()
    ├─ arm_timer() ×5         ├─ process_epoll()        │   从 pending 队列取
    └─ 检查 pending           ├─ process_child()        │   最高优先级事件源
                              ├─ process_watchdog()     │
                              ├─ process_inotify()      ├─ source_dispatch(s)
                              └─ process_timer() ×5     │   ├─ 频率限制检查
                                                        │   ├─ 清除 pending
                                                        │   ├─ 标记 POST 源
                                                        │   ├─ ONESHOT 禁用
                                                        │   └─ 类型回调
                                                        │
                                                        └─ exit_requested?
                                                             → dispatch_exit()

    内核机制映射：
    ┌──────────┐     ┌──────────┐     ┌──────────┐
    │  epoll   │     │ timerfd  │     │ signalfd │
    │ (IO/pidfd│     │ (5个时钟)│     │ (per优先 │
    │  /inotify│     │          │     │  级)     │
    │  /signal │     │          │     │          │
    │  /timer) │     │          │     │          │
    └──────────┘     └──────────┘     └──────────┘
         ↑                ↑                ↑
    sd_event_add_io  sd_event_add_time  sd_event_add_signal
    sd_event_add_child(pidfd)           sd_event_add_child(SIGCHLD)
    sd_event_add_inotify
```

---

## 十六、与 PID 1 的集成

PID 1（Manager）使用 sd-event 作为核心事件循环：

```c
// manager.c 中的初始化
r = sd_event_new(&m->event);

// 注册事件源
sd_event_add_signal(m->event, NULL, SIGCHLD, ...);    // 子进程
sd_event_add_signal(m->event, NULL, SIGTERM, ...);    // 关机信号
sd_event_add_signal(m->event, NULL, SIGINT, ...);     // Ctrl+C
sd_event_add_io(m->event, NULL, m->notify_fd, ...);   // sd_notify
sd_event_add_io(m->event, NULL, bus_fd, ...);         // D-Bus

// 运行队列调度
sd_event_add_defer(m->event, &m->run_queue_event_source,
                   manager_dispatch_run_queue, m);
sd_event_source_set_priority(m->run_queue_event_source,
                             SD_EVENT_PRIORITY_IMPORTANT);

// 主循环
manager_loop():
    while (m->objective == MANAGER_OK)
        r = sd_event_run(m->event, timeout);
```
