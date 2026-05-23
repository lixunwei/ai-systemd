# 25. Timer 单元与时间调度深度分析

## 1. 概述

Timer 单元是 systemd 的**定时任务机制**，替代传统的 cron。它支持两种调度模式：
- **单调时钟**（monotonic）：相对于某个时间点的延迟（OnBootSec=, OnActiveSec= 等）
- **日历时钟**（realtime）：绝对时间表达式（OnCalendar=）

### 核心源文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/core/timer.h` | ~70 | Timer/TimerValue 结构体、TimerBase 枚举 |
| `src/core/timer.c` | ~900 | 状态机、调度逻辑、持久化 |
| `src/shared/calendarspec.c` | ~1400 | 日历表达式解析与求值 |
| `src/shared/calendarspec.h` | ~60 | CalendarSpec 结构体 |

---

## 2. 核心数据结构

### 2.1 TimerBase 枚举（timer.h:9-18）

```c
typedef enum TimerBase {
    TIMER_ACTIVE,           // OnActiveSec=    相对于 Timer 自身激活时间
    TIMER_BOOT,             // OnBootSec=      相对于系统启动时间
    TIMER_STARTUP,          // OnStartupSec=   相对于 systemd 用户空间启动时间
    TIMER_UNIT_ACTIVE,      // OnUnitActiveSec=  相对于触发单元上次激活时间
    TIMER_UNIT_INACTIVE,    // OnUnitInactiveSec= 相对于触发单元上次停用时间
    TIMER_CALENDAR,         // OnCalendar=     绝对日历表达式
    _TIMER_BASE_MAX,
} TimerBase;
```

### 2.2 TimerValue 结构体（timer.h:20-29）

每个 `OnXxxSec=` / `OnCalendar=` 指令对应一个 TimerValue：

```c
typedef struct TimerValue {
    TimerBase base;              // 时间基准类型
    bool disabled;              // 是否禁用
    usec_t value;               // 单调定时器的延迟值（微秒）
    CalendarSpec *calendar_spec; // 日历定时器的日历规格
    usec_t next_elapse;         // 计算出的下次触发时间

    LIST_FIELDS(TimerValue, value); // 链表
} TimerValue;
```

### 2.3 Timer 结构体（timer.h:39-65）

```c
struct Timer {
    Unit meta;                  // 基类

    /* 配置 */
    usec_t accuracy_usec;       // AccuracySec=（默认 1min）
    usec_t random_usec;         // RandomizedDelaySec=

    LIST_HEAD(TimerValue, values); // 所有定时规格的链表

    /* 调度状态 */
    usec_t next_elapse_realtime;                  // 下次日历触发时间
    usec_t next_elapse_monotonic_or_boottime;     // 下次单调触发时间
    dual_timestamp last_trigger;                  // 上次触发时间戳

    /* 状态机 */
    TimerState state, deserialized_state;
    TimerResult result;

    /* sd-event 事件源 */
    sd_event_source *monotonic_event_source;
    sd_event_source *realtime_event_source;

    /* 持久化 */
    char *stamp_path;           // 时间戳文件路径
    bool persistent;            // Persistent= 配置

    /* 功能标志 */
    bool wake_system;           // WakeSystem= 唤醒休眠系统
    bool remain_after_elapse;   // RemainAfterElapse=
    bool on_clock_change;       // OnClockChange=
    bool on_timezone_change;    // OnTimezoneChange=
    bool fixed_random_delay;    // FixedRandomDelay= 确定性随机延迟
};
```

---

## 3. 状态机

### 3.1 状态定义

```c
typedef enum TimerState {
    TIMER_DEAD,      // 未启动
    TIMER_WAITING,   // 等待触发
    TIMER_RUNNING,   // 已触发，关联服务正在启动
    TIMER_ELAPSED,   // 已触发且无需再调度
    TIMER_FAILED,    // 启动失败
    _TIMER_STATE_MAX,
} TimerState;
```

### 3.2 状态转换图

```
                 start()
    DEAD/FAILED ─────────→ WAITING
                              │
                    定时器到期 │
                              ▼
                           RUNNING ──→ 服务启动成功
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
           WAITING         ELAPSED          DEAD
         (重新调度)     (RemainAfterElapse  (stop)
                         =yes, 单次型)
```

### 3.3 UnitActiveState 映射

```c
static const UnitActiveState state_translation_table[_TIMER_STATE_MAX] = {
    [TIMER_DEAD]    = UNIT_INACTIVE,
    [TIMER_WAITING] = UNIT_ACTIVE,
    [TIMER_RUNNING] = UNIT_ACTIVE,
    [TIMER_ELAPSED] = UNIT_ACTIVE,
    [TIMER_FAILED]  = UNIT_FAILED,
};
```

---

## 4. 调度核心：timer_enter_waiting()（timer.c:374-581）

这是 Timer 单元最核心的函数，计算下一次触发时间并设置定时器事件源。

### 4.1 算法概要

```
timer_enter_waiting():
  earliest_monotonic = USEC_INFINITY
  earliest_realtime = USEC_INFINITY

  for each TimerValue v in values:
    if (v->disabled) continue

    switch (v->base):
      TIMER_ACTIVE:
        base_time = timer自身的 inactive_exit_timestamp.monotonic
        next = base_time + v->value

      TIMER_BOOT:
        base_time = 0 (系统启动时的单调时钟)
        // 容器环境中使用 userspace 启动时间
        next = base_time + v->value

      TIMER_STARTUP:
        base_time = userspace_start_timestamp.monotonic
        next = base_time + v->value

      TIMER_UNIT_ACTIVE:
        base_time = 触发单元的 inactive_exit_timestamp.monotonic
        // 或 last_trigger.monotonic（取较新的）
        next = base_time + v->value

      TIMER_UNIT_INACTIVE:
        base_time = 触发单元的 inactive_enter_timestamp.monotonic
        next = base_time + v->value

      TIMER_CALENDAR:
        base_time = last_trigger.realtime (若有) 或 当前时间
        next = calendar_spec_next_usec(v->calendar_spec, base_time)

    // 已触发过的单次定时器跳过
    if (monotonic类 && next <= last_trigger) continue

    // 应用随机延迟
    next += add_random(random_usec, fixed_random_delay)

    // 更新最早时间
    earliest = min(earliest, next)

  // 安装 sd-event 定时器
  if (earliest_monotonic != INFINITY):
    sd_event_add_time(event, &monotonic_event_source,
                      wake_system ? CLOCK_BOOTTIME_ALARM : CLOCK_MONOTONIC,
                      earliest_monotonic, accuracy_usec, timer_dispatch, ...)

  if (earliest_realtime != INFINITY):
    sd_event_add_time(event, &realtime_event_source,
                      wake_system ? CLOCK_REALTIME_ALARM : CLOCK_REALTIME,
                      earliest_realtime, accuracy_usec, timer_dispatch, ...)

  state = TIMER_WAITING
```

### 4.2 单调定时器的基准时间

| TimerBase | 基准时间 | 适用场景 |
|-----------|----------|----------|
| TIMER_ACTIVE | Timer 单元激活的时刻 | 周期性自触发 |
| TIMER_BOOT | 系统启动（monotonic=0） | 开机后延迟执行 |
| TIMER_STARTUP | 用户空间启动 | 类似 BOOT 但跳过固件/引导 |
| TIMER_UNIT_ACTIVE | 关联服务上次激活 | 服务完成后再等待 |
| TIMER_UNIT_INACTIVE | 关联服务上次停用 | 服务停止后再等待 |

### 4.3 一次性触发抑制

```c
// timer.c:481-488
// 对于 ACTIVE/BOOT/STARTUP 类型，如果计算出的 next 已经在过去
// 且已经触发过（last_trigger > next），则标记为 disabled 不再触发
if (v->base == TIMER_ACTIVE || v->base == TIMER_BOOT || v->base == TIMER_STARTUP)
    if (next <= last_trigger.monotonic)
        v->disabled = true;
```

---

## 5. 触发执行：timer_enter_running()（timer.c:583-616）

```c
timer_enter_running(Timer *t):
  1. 获取触发单元（通常是同名 .service）
  2. manager_add_job(m, JOB_START, trigger_unit, JOB_REPLACE, ...)
     → 向触发单元发起启动 Job
  3. dual_timestamp_get(&t->last_trigger)
     → 记录本次触发时间
  4. if (t->persistent):
     touch_file(t->stamp_path, ... last_trigger.realtime ...)
     → 持久化到磁盘
  5. timer_set_state(t, TIMER_RUNNING)
```

当触发的服务完成后，Timer 收到通知，调用 `timer_trigger_notify()`：
- 如果是 UNIT_ACTIVE/UNIT_INACTIVE 类型 → 重新进入 `timer_enter_waiting()`
- 如果是单次类型且 `remain_after_elapse=yes` → 进入 `TIMER_ELAPSED`

---

## 6. 日历表达式（calendarspec.c）

### 6.1 语法格式

```
[星期] 年-月-日 时:分:秒[.微秒] [时区]

示例:
  Mon *-*-* 08:00:00         每周一 08:00
  *-*-01 00:00:00            每月 1 号零点
  Mon..Fri *-*-* 10:00:00    工作日 10:00
  *-*-* *:00/15:00           每 15 分钟
  2024-06-*                  2024年6月每天
  minutely / hourly / daily / weekly / monthly / quarterly / yearly
```

### 6.2 CalendarSpec 结构体

```c
typedef struct CalendarSpec {
    int weekdays_bits;          // 星期位掩码（0=Sun .. 6=Sat）
    bool end_of_month;          // 月末标志
    bool utc;                   // UTC 时区
    char *timezone;             // 指定时区

    CalendarComponent *year;    // 年份链（链表，支持范围/重复）
    CalendarComponent *month;   // 月份链
    CalendarComponent *day;     // 日期链
    CalendarComponent *hour;    // 小时链
    CalendarComponent *minute;  // 分钟链
    CalendarComponent *microsecond; // 微秒链（含秒）
} CalendarSpec;

typedef struct CalendarComponent {
    int start;                  // 起始值（-1 = 任意）
    int stop;                   // 结束值（范围用）
    int repeat;                 // 重复间隔（0 = 不重复）
    LIST_FIELDS(...);           // 链表（多个值/范围）
} CalendarComponent;
```

### 6.3 解析流程（calendarspec.c:878-1102）

```
calendar_spec_from_string(spec_string):
  1. 检查预定义快捷词:
     "minutely" → *-*-* *:*:00
     "hourly"   → *-*-* *:00:00
     "daily"    → *-*-* 00:00:00
     "weekly"   → Mon *-*-* 00:00:00
     "monthly"  → *-*-01 00:00:00
     "quarterly"→ *-01/3-01 00:00:00
     "yearly"   → *-01-01 00:00:00

  2. parse_weekdays(spec) → weekdays_bits
     支持: Mon, Tue, ..., Sun
     支持范围: Mon..Fri
     支持列表: Mon,Wed,Fri

  3. parse_date(spec) → year/month/day 链
     支持: 2024-06-*, *-*-01, *-01/3-01

  4. parse_calendar_time(spec) → hour/minute/microsecond 链
     支持: 08:00, *:00/15, 08:00:00.000000

  5. 时区解析（末尾 "UTC" 或 IANA 名称）
```

### 6.4 下次触发计算（calendarspec.c:1232-1380）

```c
calendar_spec_next_usec(CalendarSpec *spec, usec_t after, usec_t *next):
  // 从 after 时间开始，找到下一个匹配的时间点
  find_next(spec, after_tm):
    for (iteration = 0; iteration < MAX_CALENDAR_ITERATIONS; iteration++):
      // 逐层匹配: 年 → 月 → 日 → 星期 → 时 → 分 → 秒
      if (!match_year(year_chain, &tm.year)):
          advance to next matching year, reset month/day/...
      if (!match_month(month_chain, &tm.month)):
          advance to next matching month, reset day/...
      if (!match_day(day_chain, &tm.day)):
          advance to next matching day, reset hour/...
      if (!match_weekday(weekdays_bits, &tm)):
          advance to next matching weekday
      if (!match_hour(hour_chain, &tm.hour)):
          advance to next matching hour, reset min/sec
      // ... 直到找到完全匹配的时间点
    return tm → usec_t
```

---

## 7. AccuracySec= 与 RandomizedDelaySec=

### 7.1 AccuracySec=（精度控制）

```c
// 默认: 1 分钟
// 效果: sd-event 定时器的精度参数
sd_event_add_time(event, &source, clock,
                  next_elapse,
                  t->accuracy_usec,  // ← 精度
                  timer_dispatch, t);
```

- sd-event 会将精度范围内的多个定时器**合并唤醒**
- 减少系统唤醒次数，节省功耗
- 设为 `1us` 可获得最精确触发

### 7.2 RandomizedDelaySec=（随机延迟）

```c
// timer.c:353-372
static usec_t add_random(Timer *t):
  if (t->random_usec == 0) return 0;

  if (t->fixed_random_delay):
    // 确定性随机: 基于 machine-id + unit-name 的哈希
    hash = siphash24(machine_id, uid, unit_id)
    return hash % random_usec
  else:
    // 真随机
    return random_u64_range(random_usec)
```

**FixedRandomDelay=yes** 的用途：确保同一台机器上每次重启后延迟一致，但不同机器之间延迟不同。适合分布式场景（如多台服务器错开 cron 任务）。

---

## 8. Persistent= 持久化机制

### 8.1 时间戳文件

```
路径: /var/lib/systemd/timers/stamp-<unit-name>
用户: ~/.local/share/systemd/timers/stamp-<unit-name>
```

### 8.2 工作原理

```
启动时:
  if (stamp_path 存在 && mtime 在过去):
    last_trigger.realtime = mtime
    // 如果错过了触发（如系统关机期间），立即触发

触发时:
  touch_file(stamp_path, last_trigger.realtime)
  // 更新文件 mtime 为触发时间

效果:
  - 系统关机期间错过的定时任务，开机后立即补执行
  - 防止因断电丢失调度状态
```

### 8.3 时钟跳变处理（timer.c:818-825）

```c
timer_time_change():
  // 当系统时钟发生跳变（如 NTP 同步）时:
  if (last_trigger.realtime > now_realtime):
    // 时钟回退了，夹紧到当前时间
    last_trigger.realtime = now_realtime
  timer_enter_waiting()  // 重新计算调度
```

---

## 9. WakeSystem= 唤醒机制

### 9.1 时钟选择

| WakeSystem | 单调时钟 | 实时时钟 |
|------------|----------|----------|
| no (默认) | CLOCK_MONOTONIC | CLOCK_REALTIME |
| yes | CLOCK_BOOTTIME_ALARM | CLOCK_REALTIME_ALARM |

### 9.2 工作原理

- `CLOCK_BOOTTIME_ALARM` / `CLOCK_REALTIME_ALARM` 是 Linux 内核提供的定时器时钟
- 当系统处于 suspend/hibernate 状态时，ALARM 时钟到期会**唤醒系统**
- 底层通过 RTC（Real-Time Clock）硬件实现
- 典型场景：定时备份、计划任务在系统休眠时仍能执行

---

## 10. OnClockChange= / OnTimezoneChange=

```c
// 当系统时钟被手动调整或时区改变时触发
// 实现: 通过 timerfd + TFD_TIMER_CANCEL_ON_SET 检测时钟跳变
// 通过 inotify 监视 /etc/localtime 检测时区变更

timer_time_change():   // 时钟跳变回调
timer_timezone_change(): // 时区变更回调
  → 都会重新调用 timer_enter_waiting() 重新计算
  → 如果配置了 OnClockChange/OnTimezoneChange，额外触发一次
```

---

## 11. 定时器派发（timer.c:744-755）

```c
static int timer_dispatch(sd_event_source *s, usec_t usec, void *userdata):
  Timer *t = userdata;

  if (t->state != TIMER_WAITING)
    return 0;

  log_unit_debug(UNIT(t), "Timer elapsed.");
  timer_enter_running(t);
  return 0;
```

- sd-event 在定时器到期时回调此函数
- 从 WAITING 转入 RUNNING，触发关联服务

---

## 12. 完整生命周期

```
┌─────────────────────────────────────────────────────────────┐
│                      Timer 生命周期                           │
│                                                             │
│  systemctl start xxx.timer                                  │
│         │                                                   │
│         ▼                                                   │
│  timer_start() [timer.c:618-664]                           │
│    ├─ 加载 stamp file (Persistent=)                         │
│    └─ timer_enter_waiting()                                 │
│              │                                              │
│              ▼                                              │
│  计算 next_elapse [timer.c:374-581]                        │
│    ├─ 遍历所有 TimerValue                                   │
│    ├─ 计算各基准 + 偏移量                                    │
│    ├─ 加上 RandomizedDelaySec                               │
│    ├─ 选取最早时间                                           │
│    └─ 安装 sd-event 定时器（accuracy_usec 精度）             │
│              │                                              │
│         [等待...]                                            │
│              │                                              │
│              ▼ (定时器到期)                                   │
│  timer_dispatch() → timer_enter_running()                   │
│    ├─ manager_add_job(JOB_START, trigger_unit)              │
│    ├─ 更新 last_trigger                                     │
│    ├─ touch stamp_path (Persistent=)                        │
│    └─ state = RUNNING                                       │
│              │                                              │
│         [服务执行...]                                        │
│              │                                              │
│              ▼ (服务完成通知)                                 │
│  timer_trigger_notify()                                     │
│    ├─ OnUnitActive/Inactive → timer_enter_waiting() (循环)  │
│    └─ 单次型 → timer_enter_elapsed()                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 13. 配置指令速查

| 指令 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `OnActiveSec=` | 单调 | — | 相对于 Timer 激活后的延迟 |
| `OnBootSec=` | 单调 | — | 相对于系统启动后的延迟 |
| `OnStartupSec=` | 单调 | — | 相对于 systemd 启动后的延迟 |
| `OnUnitActiveSec=` | 单调 | — | 相对于触发单元上次激活后的延迟 |
| `OnUnitInactiveSec=` | 单调 | — | 相对于触发单元上次停用后的延迟 |
| `OnCalendar=` | 日历 | — | 日历表达式（绝对时间） |
| `AccuracySec=` | 精度 | 1min | 定时器合并精度 |
| `RandomizedDelaySec=` | 延迟 | 0 | 触发前的随机延迟 |
| `FixedRandomDelay=` | 布尔 | no | 使用确定性随机（基于 machine-id） |
| `Persistent=` | 布尔 | no | 持久化上次触发时间 |
| `WakeSystem=` | 布尔 | no | 从休眠唤醒系统 |
| `RemainAfterElapse=` | 布尔 | yes | 触发后保持 active 状态 |
| `OnClockChange=` | 布尔 | no | 时钟跳变时触发 |
| `OnTimezoneChange=` | 布尔 | no | 时区变更时触发 |
| `Unit=` | 字符串 | 同名.service | 触发的目标单元 |

---

## 14. 设计要点

### 14.1 与 cron 的对比

| 维度 | cron | systemd Timer |
|------|------|---------------|
| 调度精度 | 分钟级 | 微秒级 |
| 错过补执行 | anacron（独立工具） | Persistent= |
| 随机延迟 | RANDOM_DELAY（非标准） | RandomizedDelaySec= |
| 资源控制 | 无 | 完整 cgroup 支持 |
| 依赖管理 | 无 | 完整 systemd 依赖 |
| 日志 | syslog | journald（结构化） |
| 唤醒休眠 | 不支持 | WakeSystem= |
| 事件驱动 | 不支持 | OnClockChange/OnTimezoneChange |

### 14.2 sd-event 定时器合并

通过 `accuracy_usec` 参数，sd-event 引擎将多个相近的定时器合并为一次唤醒，减少 CPU 唤醒频率。这对笔记本电脑电池续航至关重要。

### 14.3 确定性随机的分布式价值

`FixedRandomDelay=yes` 使用 `siphash24(machine_id + unit_name)` 生成确定性延迟。这意味着：
- 同一台机器重启后延迟不变（调度稳定）
- 不同机器自动错开执行（负载均衡）
- 不需要外部协调
