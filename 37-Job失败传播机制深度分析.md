# Job 失败传播机制深度分析

## 1. 概述

当一个单元的启动 Job 失败时，systemd 会将失败向依赖它的单元传播。传播的范围取决于**依赖类型**——只有强依赖（`Requires=`、`Requisite=`、`BindsTo=`）才会传播失败，弱依赖（`Wants=`）不会。

本文分析 Job 失败时的完整传播路径和决策逻辑。

---

## 2. Job 结果类型

```c
// job.h 中定义
typedef enum JobResult {
    JOB_DONE,        // 成功完成
    JOB_CANCELED,    // 被取消（不视为失败）
    JOB_TIMEOUT,     // 超时
    JOB_FAILED,      // 执行失败
    JOB_DEPENDENCY,  // 因依赖失败而级联失败
    JOB_SKIPPED,     // 条件不满足跳过
    JOB_INVALID,     // 无效（如单元已消失）
    JOB_ASSERT,      // 断言不满足
    JOB_UNSUPPORTED, // 操作不支持
    JOB_ONCE,        // RefuseManualStart 等
} JobResult;
```

**关键**：只要 `result != JOB_DONE`，就会触发依赖失败传播（`job.c:970`）。

---

## 3. 核心传播函数

### 3.1 job_finish_and_invalidate()

| 源文件 | `job.c:930-1000` |
|--------|------------------|

```c
int job_finish_and_invalidate(Job *j, JobResult result, bool recursive, bool already) {
    u = j->unit;
    t = j->type;

    // ... 日志、统计 ...

    job_uninstall(j);
    job_free(j);

    /* 失败依赖传播 */
    if (result != JOB_DONE && recursive) {
        if (IN_SET(t, JOB_START, JOB_VERIFY_ACTIVE))
            job_fail_dependencies(u, UNIT_ATOM_PROPAGATE_START_FAILURE);
        else if (t == JOB_STOP)
            job_fail_dependencies(u, UNIT_ATOM_PROPAGATE_STOP_FAILURE);
    }

    /* 特殊：启动成功但单元未真正 active（如 oneshot 条件不满足） */
    if (result == JOB_DONE && recursive &&
        IN_SET(t, JOB_START, JOB_RELOAD) &&
        !UNIT_IS_ACTIVE_OR_RELOADING(unit_active_state(u)))
        job_fail_dependencies(u, UNIT_ATOM_PROPAGATE_INACTIVE_START_AS_FAILURE);
}
```

### 3.2 job_fail_dependencies()

```c
// job.c:913-928
static void job_fail_dependencies(Unit *u, UnitDependencyAtom match_atom) {
    UNIT_FOREACH_DEPENDENCY(other, u, match_atom) {
        Job *j = other->job;
        if (!j) continue;
        if (!IN_SET(j->type, JOB_START, JOB_VERIFY_ACTIVE)) continue;

        job_finish_and_invalidate(j, JOB_DEPENDENCY, true, false);
        //                                           ^^^^
        //                        recursive=true → 继续级联传播！
    }
}
```

**要点**：
- `recursive=true` 意味着级联失败会继续向上传播
- 仅影响正在等待（pending）的 `JOB_START` 或 `JOB_VERIFY_ACTIVE` 类型的 Job
- 不会影响已运行中的服务（没有 pending job）

---

## 4. 依赖原子属性与传播规则

### 4.1 哪些依赖传播失败

| 反向依赖类型 | PROPAGATE_START_FAILURE | PROPAGATE_STOP_FAILURE | 效果 |
|-------------|:----------------------:|:----------------------:|------|
| `REQUIRED_BY`（对应 Requires=） | ✓ | - | 启动失败会级联 |
| `REQUISITE_OF`（对应 Requisite=） | ✓ | - | 启动失败会级联 |
| `BOUND_BY`（对应 BindsTo=） | ✓ | - | 启动失败会级联 |
| `WANTED_BY`（对应 Wants=） | ✗ | - | **不传播** |
| `UPHELD_BY`（对应 Upholds=） | ✗ | - | **不传播** |

源码：`unit-dependency-atom.c:44-70`

```c
[UNIT_REQUIRED_BY] = UNIT_ATOM_PROPAGATE_STOP |
                     UNIT_ATOM_PROPAGATE_RESTART |
                     UNIT_ATOM_PROPAGATE_START_FAILURE |  // ← 传播启动失败
                     ...

[UNIT_WANTED_BY]   = UNIT_ATOM_DEFAULT_TARGET_DEPENDENCIES |
                     UNIT_ATOM_PINS_STOP_WHEN_UNNEEDED,
                     // ← 没有 PROPAGATE_START_FAILURE！
```

### 4.2 传播方向说明

```
                 依赖关系
A ─── Requires= ──→ B
     (A 需要 B)

内部存储为:
  A.dependencies[UNIT_REQUIRES] 包含 B
  B.dependencies[UNIT_REQUIRED_BY] 包含 A

传播方向:
  B 失败 → 遍历 B 的 REQUIRED_BY → 找到 A → A 的 Job 标记为 JOB_DEPENDENCY
```

---

## 5. 完整场景分析

### 5.1 场景：A Requires=B, C Requires=A, B oneshot 失败

```
依赖关系: C ─Requires→ A ─Requires→ B(oneshot)

启动顺序（假设也配了 After）:
  Job Queue: [start B] [start A] [start C]

1. B(oneshot) 的脚本 exit 1
   → is_clean_exit() = false
   → f = SERVICE_FAILURE_EXIT_CODE
   → service_enter_signal(SERVICE_STOP_SIGTERM, f)
   → 最终状态: FAILED

2. B 的 start Job 完成: result = JOB_FAILED
   → job_finish_and_invalidate(B.job, JOB_FAILED, recursive=true)
   → result != JOB_DONE && t == JOB_START
   → job_fail_dependencies(B, UNIT_ATOM_PROPAGATE_START_FAILURE)

3. 遍历 B 的 REQUIRED_BY 依赖 → 找到 A
   → A 有 pending start Job
   → job_finish_and_invalidate(A.job, JOB_DEPENDENCY, recursive=true)
   → A 的 Job 标记为 "dependency" 失败

4. A 的 Job 也 result != JOB_DONE
   → job_fail_dependencies(A, UNIT_ATOM_PROPAGATE_START_FAILURE)
   → 遍历 A 的 REQUIRED_BY → 找到 C
   → C 有 pending start Job
   → job_finish_and_invalidate(C.job, JOB_DEPENDENCY, recursive=true)

结果: B 失败 → A 级联失败 → C 级联失败
      三个服务都不会启动
```

### 5.2 场景：A Requires=B, C Wants=A, B 失败

```
依赖关系: C ─Wants→ A ─Requires→ B(oneshot)

1. B 失败 → A 的 Job 级联失败 (JOB_DEPENDENCY)

2. A 失败后:
   → job_fail_dependencies(A, UNIT_ATOM_PROPAGATE_START_FAILURE)
   → 遍历 A 的 REQUIRED_BY → 空（C 用的是 Wants，对应 WANTED_BY）
   → WANTED_BY 不包含 PROPAGATE_START_FAILURE
   → C 的 Job 不受影响

3. C 的 start Job 正常执行
   （因为有 After=A，会等 A 的 job 结束，然后正常启动）

结果: B 失败 → A 失败 → C 正常启动 ✓
```

### 5.3 场景：C 仅 After=A（无 Requires/Wants）

```
After= 只影响执行顺序，不产生依赖关系。

1. B 失败 → A 失败
2. C 的 Job 等 A 的 Job 完成（无论成功失败）
3. A 的 Job 完成后，C 的 Job 变为可运行
4. C 正常启动

结果: C 正常启动 ✓
注意: 仅有 After= 时，如果 A 的 Job 不存在，C 也不会等待
```

---

## 6. 特殊情况

### 6.1 ExecStart=-（忽略退出码）

```ini
ExecStart=-/usr/bin/might-fail.sh
```

命令前的 `-` 前缀设置 `EXEC_COMMAND_IGNORE_FAILURE` 标志：

```c
// service_sigchld_event()
if (s->main_command->flags & EXEC_COMMAND_IGNORE_FAILURE)
    f = SERVICE_SUCCESS;  // 强制标记为成功
```

此时 oneshot 会走 `service_enter_start_post()` → 正常就绪，不会传播失败。

### 6.2 SuccessExitStatus=

```ini
SuccessExitStatus=1 2 SIGTERM
```

`is_clean_exit()` 会检查 `success_status` 集合：

```c
bool is_clean_exit(int code, int status, ExitClean clean, const ExitStatusSet *success_status) {
    if (code == CLD_EXITED)
        return status == 0 ||
               (success_status && bitmap_isset(&success_status->status, status));
}
```

将退出码 1、2 视为成功 → 不触发失败传播。

### 6.3 Requisite= vs Requires=

| | Requires= | Requisite= |
|--|-----------|------------|
| B 还未启动 | 拉起 B 启动 | 立即失败（不尝试启动 B） |
| B 启动中失败 | A 失败 | A 失败 |
| 额外原子 | - | `PROPAGATE_INACTIVE_START_AS_FAILURE` |

`Requisite=` 更严格：如果 B 没有处于 active 状态（即使 Job 返回 DONE），也会传播失败。

### 6.4 BindsTo= 的额外行为

`BindsTo=` 除了传播启动失败外，还有：
- B 停止时 → A 也被停止（`RETROACTIVE_STOP_ON_STOP`）
- B 不活跃时 → A 进入 "cannot be active without" 队列

---

## 7. 总结对照表

| 场景 | B 失败后 A 的命运 | A 失败后 C 的命运 |
|------|------------------|------------------|
| A `Requires=B`, C `Requires=A` | ❌ 失败 | ❌ 失败 |
| A `Requires=B`, C `Wants=A` | ❌ 失败 | ✓ 启动 |
| A `Requires=B`, C 仅 `After=A` | ❌ 失败 | ✓ 启动 |
| A `Wants=B`, C `Requires=A` | ✓ 启动 | ✓ 启动 |
| A `BindsTo=B`, C `Requires=A` | ❌ 失败 | ❌ 失败 |
| A `Requisite=B`(B inactive), C `Requires=A` | ❌ 失败 | ❌ 失败 |

---

## 8. 关键源码索引

| 功能 | 文件:行 |
|------|---------|
| job_finish_and_invalidate() | `job.c:930-1000` |
| job_fail_dependencies() | `job.c:913-928` |
| PROPAGATE_START_FAILURE 定义 | `unit-dependency-atom.h:53` |
| REQUIRED_BY 原子组合 | `unit-dependency-atom.c:44-48` |
| WANTED_BY 原子组合 | `unit-dependency-atom.c:57-58` |
| is_clean_exit() 退出码检查 | `exit-status.c:140-155` |
| oneshot 失败处理 | `service.c:3574-3581` |
| EXEC_COMMAND_IGNORE_FAILURE | `service.c:3518-3519` |

---

## 9. 设计哲学

| 原则 | 体现 |
|------|------|
| **强依赖严格** | Requires/Requisite/BindsTo 传播失败，确保系统一致性 |
| **弱依赖宽容** | Wants 不传播失败，最大化系统可用性 |
| **递归传播** | 级联失败可以穿过整条依赖链，无需逐层配置 |
| **仅影响 pending** | 只取消正在等待的 Job，不干扰已运行的服务 |
| **明确错误码** | JOB_DEPENDENCY 明确告知失败原因是依赖，非自身问题 |
