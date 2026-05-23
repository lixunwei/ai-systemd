# systemd 启动流程深度分析（PID1 初始化到 default.target）

## 1. 概述

systemd 作为 PID1 的启动流程是 Linux 系统从内核态进入用户态后的第一段关键代码路径。整个过程可概括为：

```
内核 → /sbin/init(systemd) → 早期挂载 → 安全策略 → Manager 创建
→ 生成器运行 → 单元枚举 → 初始事务(start default.target)
→ 事件循环执行作业 → 启动完成
```

本文档分析从 `main()` 入口到 `default.target` 完全激活的完整代码路径。

---

## 2. main() 入口与早期初始化

### 2.1 入口点

| 项目 | 说明 |
|------|------|
| 源文件 | `main.c:2625` |
| 签名 | `int main(int argc, char *argv[])` |
| 关键时间戳 | `kernel_timestamp`(单调时间 0)、`userspace_timestamp`(当前时间) |

### 2.2 PID1 身份判定

```c
// main.c:2679
if (getpid_cached() == 1) {
    arg_system = true;    // 强制系统模式
    umask(0);             // 禁用 umask
    log_set_prohibit_ipc(true);  // 禁止 IPC 日志（journald 未启动）
}
```

三种运行模式判断：

| 条件 | 模式 | 日志目标 |
|------|------|---------|
| `getpid() == 1 && !container` | 物理机 PID1 | `LOG_TARGET_KMSG` |
| `getpid() == 1 && container` | 容器 PID1 | `LOG_TARGET_CONSOLE` |
| `getpid() != 1` | 用户实例 | `LOG_TARGET_AUTO` |

### 2.3 SysV 兼容

```c
// main.c:2649
redirect_telinit(argc, argv);
```

当通过 `/sbin/init` 符号链接调用时，若 PID ≠ 1 则重定向到 `telinit`。

### 2.4 早期设置序列（物理机 PID1）

以下操作仅在 `!skip_setup`（非 reexec）时执行：

```
┌─ mount_setup_early()          ← 挂载 /proc /sys /dev/shm 等
│   main.c:2706
├─ log_open()                   ← 重新打开日志（/dev/kmsg 现在可用）
│   main.c:2715
├─ disable_printk_ratelimit()   ← 禁用内核日志限速
│   main.c:2717
├─ initialize_security()        ← SELinux/AppArmor/IMA/SMACK 策略加载
│   main.c:2719-2725
├─ mac_selinux_init()           ← SELinux 库初始化
│   main.c:2728
├─ initialize_clock()           ← 同步硬件时钟
│   main.c:2734
├─ initialize_coredump()        ← 配置 core dump 处理
│   main.c:2754
├─ fixup_environment()          ← 修复 PID1 环境变量
│   main.c:2756
├─ make_null_stdio()            ← 重定向 stdin/stdout/stderr → /dev/null
│   main.c:2767
├─ kmod_setup()                 ← 加载关键内核模块
│   main.c:2772
├─ mount_setup()                ← 完整挂载 API 文件系统
│   main.c:2776
└─ efi_take_random_seed()       ← 从 EFI 变量读取随机种子
    main.c:2783
```

### 2.5 配置解析

```c
// main.c:2811
parse_configuration(&saved_rlimit_nofile, &saved_rlimit_memlock);
// 读取 /etc/systemd/system.conf 或 /etc/systemd/user.conf

// main.c:2813
parse_argv(argc, argv);
// 解析命令行参数（含内核 cmdline systemd.* 参数）
```

---

## 3. initialize_runtime() — 运行时环境设置

| 源文件 | `main.c:2065-2174` |
|--------|-------------------|

此函数在 Manager 创建前设置运行时环境：

### 系统模式专属操作

```c
install_crash_handler();           // 安装崩溃信号处理器
mount_cgroup_controllers();        // 挂载 cgroup 控制器
status_welcome();                  // 显示欢迎信息到控制台
hostname_setup(true);              // 设置主机名
machine_id_setup(NULL, first_boot, arg_machine_id, NULL);  // 生成/读取 machine-id
loopback_setup();                  // 配置 lo 网卡
bump_unix_max_dgram_qlen();        // 提升 Unix socket 队列长度
bump_file_max_and_nr_open();       // 提升系统文件描述符限制
```

### 通用操作

```c
update_cpu_affinity(skip_setup);   // CPU 亲和性设置
update_numa_policy(skip_setup);    // NUMA 策略
bump_rlimit_nofile();              // 提升 NOFILE 限制
bump_rlimit_memlock();             // 提升 MEMLOCK 限制
```

### 用户模式专属

```c
// 创建 $XDG_RUNTIME_DIR/systemd 并设置 inaccessible 节点
mkdir_p_label(p, 0755);
make_inaccessible_nodes(p, UID_INVALID, GID_INVALID);
// 成为子进程 subreaper
prctl(PR_SET_CHILD_SUBREAPER, 1);
```

---

## 4. Manager 创建与启动

### 4.1 manager_new()

| 源文件 | `manager.c:784-967` |
|--------|---------------------|

```c
// main.c:2891
r = manager_new(arg_system ? UNIT_FILE_SYSTEM : UNIT_FILE_USER,
                arg_action == ACTION_TEST ? MANAGER_TEST_FULL : 0,
                &m);
```

`manager_new()` 分配并初始化 Manager 结构体的核心字段：

| 初始化项 | 说明 |
|---------|------|
| `sd_event_new()` | 创建事件循环 |
| hashmap 系列 | units, jobs, watch_pids, cgroup_map 等 |
| 信号处理 | SIGCHLD/SIGTERM/SIGINT/SIGHUP 事件源 |
| 队列 | load_queue, cleanup_queue, gc_unit_queue, dbus_queue |
| FD 占位 | notify_fd, cgroups_agent_fd, signal_fd 等 |
| 默认值 | default_restart_usec, default_timeout 等 |

### 4.2 时间戳记录

```c
// main.c:2900-2904
m->timestamps[MANAGER_TIMESTAMP_KERNEL] = kernel_timestamp;
m->timestamps[MANAGER_TIMESTAMP_INITRD] = initrd_timestamp;
m->timestamps[MANAGER_TIMESTAMP_USERSPACE] = userspace_timestamp;
m->timestamps[MANAGER_TIMESTAMP_SECURITY_START] = security_start_timestamp;
m->timestamps[MANAGER_TIMESTAMP_SECURITY_FINISH] = security_finish_timestamp;
```

这些时间戳用于 `systemd-analyze` 计算启动耗时。

### 4.3 manager_startup()

| 源文件 | `manager.c:1756-1851` |
|--------|----------------------|

这是启动过程的核心函数，完成从"空 Manager"到"就绪可运行"的全部准备：

```
manager_startup()
├── lookup_paths_init()              ← 初始化单元搜索路径
├── manager_run_environment_generators() ← 环境生成器
├── manager_run_generators()         ← 单元生成器
├── manager_preset_all()             ← 应用 preset 策略（首次启动）
├── manager_enumerate_perpetual()    ← 枚举永久单元（-.slice等）
├── manager_enumerate()              ← 枚举所有单元文件
├── manager_deserialize()            ← 反序列化（reexec 时）
├── manager_distribute_fds()         ← 分发预传入的 FD
├── manager_setup_notify()           ← 设置 notify socket
├── manager_setup_cgroups_agent()    ← 设置 cgroup 通知
├── manager_setup_user_lookup_fd()   ← 用户查找通道
├── manager_setup_bus()              ← 连接 D-Bus
├── manager_varlink_init()           ← 初始化 Varlink
├── manager_coldplug()               ← 冷启动所有已反序列化的单元
├── manager_vacuum()                 ← 清理运行时对象
└── manager_ready()                  ← 标记就绪状态
```

---

## 5. 生成器（Generators）

### 5.1 生成器原理

生成器是外部程序，在单元枚举前运行，动态生成单元文件或符号链接。

| 优先级 | 输出目录 | 在搜索路径中的位置 |
|--------|---------|------------------|
| early | `/run/systemd/generator.early` | 高于 /etc 配置 |
| normal | `/run/systemd/generator` | 高于 /usr 供应商 |
| late | `/run/systemd/generator.late` | 低于 /usr 供应商 |

### 5.2 生成器搜索路径

```
// path-lookup.c:784-828
系统生成器路径:
  /run/systemd/system-generators
  /etc/systemd/system-generators
  /usr/local/lib/systemd/system-generators
  /usr/lib/systemd/system-generators
```

### 5.3 主要引导生成器

| 生成器 | 功能 |
|--------|------|
| `systemd-fstab-generator` | 将 /etc/fstab 转换为 .mount 单元 |
| `systemd-getty-generator` | 为活动 VT/串口生成 getty 服务 |
| `systemd-gpt-auto-generator` | 根据 GPT 分区类型自动挂载 |
| `systemd-cryptsetup-generator` | 处理 /etc/crypttab |
| `systemd-system-update-generator` | 系统更新重定向到 system-update.target |
| `systemd-sysv-generator` | 将 SysV init 脚本转换为单元 |
| `systemd-hibernate-resume-generator` | 休眠恢复 |
| `systemd-debug-generator` | 处理 systemd.mask= 内核参数 |
| `systemd-run-generator` | 处理 systemd.run= 内核参数 |

### 5.4 执行时机

```c
// manager.c:1769-1773
dual_timestamp_get(m->timestamps + MANAGER_TIMESTAMP_GENERATORS_START);
r = manager_run_environment_generators(m);   // 先跑环境生成器
r = manager_run_generators(m);               // 再跑单元生成器
dual_timestamp_get(m->timestamps + MANAGER_TIMESTAMP_GENERATORS_FINISH);
```

---

## 6. 单元搜索路径优先级

`lookup_paths_init()` 建立的搜索路径优先级（从高到低）：

```
// path-lookup.c:592-653
1. /etc/systemd/system.control      ← 控制目录（systemctl edit）
2. /run/systemd/transient           ← 临时单元（systemd-run）
3. /run/systemd/generator.early     ← 早期生成器输出
4. /etc/systemd/system              ← 管理员配置
5. /etc/systemd/system.attached     ← portable service
6. /run/systemd/system              ← 运行时配置
7. /run/systemd/system.attached     ← portable service runtime
8. /run/systemd/generator           ← 普通生成器输出
9. /usr/local/lib/systemd/system    ← 本地安装
10. /usr/lib/systemd/system         ← 发行版供应商
11. /run/systemd/generator.late     ← 晚期生成器输出
```

---

## 7. 初始事务创建

### 7.1 do_queue_default_job()

| 源文件 | `main.c:2176-2239` |
|--------|-------------------|

```c
// 确定默认目标
if (arg_default_unit)
    unit = arg_default_unit;                    // 命令行/配置指定
else if (in_initrd())
    unit = SPECIAL_INITRD_TARGET;               // initrd 中：initrd.target
else
    unit = SPECIAL_DEFAULT_TARGET;              // 正常：default.target

// 加载目标单元
r = manager_load_startable_unit_or_warn(m, unit, NULL, &target);
// 失败回退链: default.target → rescue.target

// 创建初始作业
r = manager_add_job(m, JOB_START, target, JOB_ISOLATE, NULL, &error, &job);
// JOB_ISOLATE: 启动目标并停止所有不需要的单元
// 若 isolate 失败（target 不允许）则回退到 JOB_REPLACE
```

### 7.2 事务引擎工作流

```
manager_add_job()
├── transaction_new()                ← 创建事务对象
├── transaction_add_job_and_dependencies()  ← 递归展开依赖
│   ├── UNIT_ATOM_PULL_IN_START      ← Requires=/BindsTo= 拉入
│   ├── UNIT_ATOM_PULL_IN_START_IGNORED ← Wants=/Upholds= 拉入(忽略失败)
│   ├── UNIT_ATOM_PULL_IN_VERIFY     ← Requisite= 验证
│   ├── UNIT_ATOM_PULL_IN_STOP       ← Conflicts= 拉入停止
│   └── 递归处理 After=/Before= 排序
├── transaction_activate()           ← 激活事务
│   ├── transaction_find_jobs_that_matter() ← 标记重要作业
│   ├── transaction_gc_jobs()        ← 垃圾回收无关作业
│   ├── transaction_minimize_impact() ← 最小化影响
│   ├── transaction_verify_order()   ← 检测/打破循环依赖
│   ├── transaction_merge_jobs()     ← 合并同单元作业
│   ├── transaction_is_destructive() ← 检查是否破坏性
│   └── transaction_apply()          ← 安装作业到 Manager
└── transaction_free()               ← 释放事务对象
```

### 7.3 依赖原子类型

| 原子类型 | 对应依赖 | 行为 |
|---------|---------|------|
| `PULL_IN_START` | Requires, BindsTo, Requisite | 拉入启动，失败导致事务失败 |
| `PULL_IN_START_IGNORED` | Wants, Upholds | 拉入启动，失败被忽略 |
| `PULL_IN_VERIFY` | Requisite | 仅验证已启动，不拉入 |
| `PULL_IN_STOP` | Conflicts, ConflictedBy | 拉入停止作业 |
| `PULL_IN_STOP_IGNORED` | 冲突弱引用 | 拉入停止，失败忽略 |

源文件: `unit-dependency-atom.c:16-80`

---

## 8. Target 依赖链

### 8.1 标准启动链

```
                    default.target
                         │
                    (symlink to)
                         │
              ┌──────────┴──────────┐
              │                     │
     graphical.target      multi-user.target
              │                     │
              │  Requires=          │  Requires=
              │                     │
     multi-user.target       basic.target
              │                     │
              │  Requires=          │  Requires=
              │                     │
       basic.target          sysinit.target
              │                     │
              │  Requires=          │  Wants=
              │                     │
       sysinit.target     ┌────────┴────────┐
              │           │                 │
              │     local-fs.target   swap.target
              │           │
              │     (After=local-fs-pre.target)
              │
         Wants=
    ┌────────┼────────────┐
    │        │            │
sockets  timers.target  paths.target
.target       │            │
    │    slices.target     │
    │                      │
```

### 8.2 各 Target 详细定义

| Target | Requires | Wants | After | 源文件 |
|--------|----------|-------|-------|--------|
| `graphical.target` | multi-user.target | display-manager.service | multi-user.target | `units/graphical.target:13-17` |
| `multi-user.target` | basic.target | — | basic.target | `units/multi-user.target:13-16` |
| `basic.target` | sysinit.target | sockets/timers/paths/slices.target | sysinit.target | `units/basic.target:13-22` |
| `sysinit.target` | — | local-fs.target, swap.target | local-fs/swap.target | `units/sysinit.target:13-16` |
| `local-fs.target` | — | — | local-fs-pre.target | `units/local-fs.target:13-18` |

### 8.3 AllowIsolate 机制

只有设置了 `AllowIsolate=yes` 的 target 才能作为 isolate 目标：

- `graphical.target` ✓
- `multi-user.target` ✓
- `rescue.target` ✓
- `emergency.target` ✓

初始事务使用 `JOB_ISOLATE` 模式，即启动 default.target 的同时停止所有不属于其依赖树的单元。

---

## 9. 事件循环 — manager_loop()

| 源文件 | `manager.c:3000-3057` |
|--------|----------------------|

### 9.1 循环结构

```c
while (m->objective == MANAGER_OK) {
    watchdog_ping();                              // 喂硬件看门狗

    // 速率限制：1秒内超过 50000 次循环则节流
    if (!ratelimit_below(&rl)) { sleep(1); }

    // 按优先级顺序处理各队列
    manager_dispatch_load_queue(m);               // 加载待加载单元
    manager_dispatch_gc_job_queue(m);             // GC 已完成作业
    manager_dispatch_gc_unit_queue(m);            // GC 不再需要的单元
    manager_dispatch_cleanup_queue(m);            // 清理单元
    manager_dispatch_cgroup_realize_queue(m);     // 实现 cgroup
    manager_dispatch_start_when_upheld_queue(m);  // Upholds= 自动启动
    manager_dispatch_stop_when_bound_queue(m);    // BindsTo= 自动停止
    manager_dispatch_stop_when_unneeded_queue(m); // StopWhenUnneeded=
    manager_dispatch_dbus_queue(m);               // D-Bus 属性变更通知

    // 等待事件（使用看门狗超时作为最大等待时间）
    sd_event_run(m->event, watchdog_runtime_wait());
}
```

### 9.2 队列优先级说明

| 优先级 | 队列 | 说明 |
|--------|------|------|
| 1 | load_queue | 新发现的单元需要加载配置 |
| 2 | gc_job_queue | 已完成/取消的作业需要清理 |
| 3 | gc_unit_queue | 无引用的单元需要回收 |
| 4 | cleanup_queue | 单元清理（如重置失败计数） |
| 5 | cgroup_realize_queue | cgroup 目录需要创建/更新 |
| 6 | start_when_upheld | Upholds= 依赖触发启动 |
| 7 | stop_when_bound | BindsTo= 源停止后跟随停止 |
| 8 | stop_when_unneeded | StopWhenUnneeded= 检查 |
| 9 | dbus_queue | 发送属性变更信号 |

### 9.3 作业执行

作业通过 sd-event 的事件源触发：

```
sd_event_run()
  → run_queue 事件源触发
    → manager_dispatch_run_queue()
      → job_run_and_invalidate()
        → job_perform_on_unit()
          → unit_start() / unit_stop() / unit_reload()
```

---

## 10. 启动完成检测

### 10.1 manager_check_finished()

```c
// manager.c:3491-3525
// 当所有作业完成（has_jobs == false）且有启动作业时，标记启动完成
```

触发条件：`m->n_running_jobs == 0`（所有初始作业执行完毕）

### 10.2 启动完成消息

```
Startup finished in <kernel>s (kernel) + <initrd>s (initrd) + <userspace>s (userspace) = <total>s.
```

时间计算：
- **kernel**: `MANAGER_TIMESTAMP_KERNEL` → `MANAGER_TIMESTAMP_USERSPACE`
- **initrd**: `MANAGER_TIMESTAMP_INITRD` → `MANAGER_TIMESTAMP_USERSPACE`（仅从 initrd 切换时）
- **userspace**: `MANAGER_TIMESTAMP_USERSPACE` → `MANAGER_TIMESTAMP_FINISH`

### 10.3 systemd-analyze 实现

| 命令 | 实现 | 源文件 |
|------|------|--------|
| `systemd-analyze blame` | 按激活耗时排序所有单元 | `analyze-blame.c:8-65` |
| `systemd-analyze critical-chain` | 递归遍历 After= 依赖找最长路径 | `analyze-critical-chain.c:14-236` |

---

## 11. initrd → 真实根文件系统切换

### 11.1 initrd Target 链

```
initrd.target
├── Requires=basic.target
├── Wants=initrd-root-fs.target
├── Wants=initrd-root-device.target
├── Wants=initrd-fs.target
├── Wants=initrd-usr-fs.target
└── Wants=initrd-parse-etc.service

         ↓ (root mounted)

initrd-switch-root.target
├── Wants=initrd-switch-root.service
├── After=initrd-root-fs.target
├── After=initrd-fs.target
└── After=systemd-journald.service
```

### 11.2 switch_root() 实现

| 源文件 | `switch-root.c:28-127` |
|--------|----------------------|

```c
// 核心步骤:
1. pivot_root(new_root, pivot_dir)  // 或 MS_MOVE
2. 递归移动 /sys /dev /run /proc 到新根
3. chroot(".")                       // 切换到新根
4. rmdir(old_root)                   // 删除旧根挂载点
```

### 11.3 reexec 流程

switch-root 后 PID1 需要重新执行自身：

```c
// main.c:2972-2980
if (IN_SET(r, MANAGER_REEXECUTE, MANAGER_SWITCH_ROOT))
    r = do_reexecute(...);
```

`do_reexecute()` 序列：
1. `manager_serialize()` — 序列化所有状态到 FD
2. `execve()` — 执行新 systemd 二进制（带 `--deserialize` 参数）
3. 新进程 `manager_deserialize()` 恢复状态

---

## 12. 用户实例启动

### 12.1 触发机制

```
用户登录 → logind → user_start()
                      → start user@UID.service
                        → ExecStart=systemd --user
```

源文件: `logind-user.c:342-350, 446-481`

### 12.2 user@.service 定义

```ini
# units/user@.service.in
[Unit]
After=systemd-user-sessions.service user-runtime-dir@%i.service dbus.service
Requires=user-runtime-dir@%i.service

[Service]
ExecStart=systemd --user
Type=notify
```

### 12.3 用户实例 Target 链

```
user default.target
      │
      │ Wants=
      ├── graphical-session.target
      │     └── After=graphical-session-pre.target
      └── basic.target (user)
            └── default.target
```

### 12.4 用户实例特殊处理

```c
// main.c:2788-2801
arg_system = false;
log_set_target(LOG_TARGET_AUTO);
prctl(PR_SET_CHILD_SUBREAPER, 1);  // 成为子进程收割者
```

---

## 13. 紧急/救援模式

### 13.1 emergency.target

```ini
# units/emergency.target
Requires=emergency.service
AllowIsolate=yes
```

进入条件：
- 内核参数 `systemd.unit=emergency.target`
- 关键挂载失败（local-fs.target OnFailure=emergency.target）
- 启动严重错误

### 13.2 rescue.target

```ini
# units/rescue.target
Requires=sysinit.target rescue.service
After=sysinit.target
AllowIsolate=yes
Conflicts=multi-user.target
```

比 emergency 多一步：完成 sysinit（挂载文件系统等）后进入单用户模式。

### 13.3 回退链

```
default.target 加载失败
  → 回退到 rescue.target
    → rescue.target 加载失败
      → 无法启动，打印 emergency 错误
```

---

## 14. Reexec 与 Reload

### 14.1 Reexec（重新执行）

用途：升级 PID1 二进制而不重启系统。

```
systemctl daemon-reexec
  → D-Bus 调用 Reexecute()
    → m->objective = MANAGER_REEXECUTE
      → manager_loop() 退出
        → prepare_reexecute()
          → manager_serialize() 到 FD
            → execve("/usr/lib/systemd/systemd", "--deserialize", fd_str)
```

序列化内容包括：
- 所有单元状态
- 所有作业
- 所有打开的 FD
- 所有计时器/监控

### 14.2 Reload（重新加载配置）

```
systemctl daemon-reload
  → manager_reload()
    → manager_run_generators()   ← 重新运行生成器
    → manager_enumerate()        ← 重新枚举单元
    → manager_coldplug()         ← 应用变更
```

---

## 15. 关键数据结构

### 15.1 Manager 核心字段

```c
struct Manager {
    UnitFileScope unit_file_scope;    // SYSTEM/USER
    LookupPaths lookup_paths;         // 单元搜索路径
    Hashmap *units;                   // name → Unit*
    Hashmap *jobs;                    // id → Job*
    LIST_HEAD(Job, run_queue);        // 待执行作业队列
    sd_event *event;                  // 事件循环
    ManagerObjective objective;       // 当前目标(OK/EXIT/REEXEC等)
    dual_timestamp timestamps[_MANAGER_TIMESTAMP_MAX];  // 时间戳
    ...
};
```

### 15.2 ManagerObjective 枚举

| 值 | 含义 |
|----|------|
| `MANAGER_OK` | 正常运行 |
| `MANAGER_EXIT` | 退出（用户实例） |
| `MANAGER_RELOAD` | 重新加载配置 |
| `MANAGER_REEXECUTE` | 重新执行 PID1 |
| `MANAGER_REBOOT` | 重启 |
| `MANAGER_POWEROFF` | 关机 |
| `MANAGER_HALT` | 停机 |
| `MANAGER_KEXEC` | kexec 热重启 |
| `MANAGER_SWITCH_ROOT` | 切换根文件系统 |

---

## 16. 完整启动时间线

```
T=0 ─────── 内核启动
             │
T=kernel ──── /sbin/init (systemd) 开始执行
             │
             ├── 早期挂载 (/proc, /sys, /dev)
             ├── SELinux/AppArmor 策略加载
             ├── mount_setup() 完整 API FS
             ├── 解析 system.conf + cmdline
             │
T=gen_start ── manager_run_generators()
             │  ├── fstab-generator
             │  ├── getty-generator
             │  └── ... (20+ 生成器)
T=gen_end ──── 生成器完成
             │
T=load_start ─ manager_enumerate()
             │  └── 扫描所有搜索路径，加载单元文件
T=load_end ─── 单元加载完成
             │
             ├── do_queue_default_job()
             │    └── transaction: start default.target (isolate)
             │         → 展开依赖 → 创建完整作业图
             │
T=loop_start ─ manager_loop() 开始
             │  ├── 执行 run_queue 中的作业
             │  │    ├── start sysinit.target deps
             │  │    ├── start basic.target deps
             │  │    ├── start multi-user.target deps
             │  │    └── start graphical.target deps
             │  │
             │  └── 所有作业完成
             │
T=finish ──── "Startup finished in ..."
             │
             └── 正常事件循环（等待请求）
```

---

## 17. 设计哲学

1. **确定性启动**：通过事务引擎将 "start default.target" 展开为完整的作业图，确保依赖正确且无遗漏

2. **并行最大化**：没有 After/Before 约束的单元同时启动，socket activation 进一步解耦依赖

3. **优雅降级**：default.target → rescue.target → emergency.target 三级回退确保系统总能启动到某种可用状态

4. **状态持久化**：reexec/switch-root 通过序列化/反序列化保持 PID1 状态连续性，实现无感升级

5. **生成器解耦**：外部程序动态生成单元文件，使核心代码不必硬编码对 fstab/crypttab 等的支持

6. **时间戳全程记录**：从内核到各阶段完成都有精确时间戳，支持 `systemd-analyze` 诊断启动性能瓶颈
