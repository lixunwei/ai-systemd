# 26. Condition/Assert 条件判断机制深度分析

## 1. 概述

systemd 提供了**条件检查**（Condition）和**断言检查**（Assert）机制，允许 Unit 在启动前检测系统环境，决定是否执行。这是实现"条件化服务"的基础——同一套 Unit 文件在不同环境中智能适配。

### 行为差异

| 类型 | 失败行为 | 日志级别 | Job 结果 |
|------|----------|----------|----------|
| Condition | **跳过**启动（视为成功） | debug | done（条件跳过） |
| Assert | **失败**退出 | error | failed |

### 核心源文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/shared/condition.h` | ~72 | ConditionType 枚举、Condition 结构体 |
| `src/shared/condition.c` | ~1300 | 所有条件类型的测试实现 |
| `src/core/unit.c` | (相关段) | 条件检查集成点 |
| `src/core/load-fragment.c` | (相关段) | 配置解析 |

---

## 2. ConditionType 枚举（condition.h:10-50）

共 32 种条件类型：

### 2.1 系统环境类

| 枚举值 | 配置指令 | 检查内容 |
|--------|----------|----------|
| `CONDITION_ARCHITECTURE` | ConditionArchitecture= | CPU 架构 (x86-64, arm64...) |
| `CONDITION_FIRMWARE` | ConditionFirmware= | 固件类型 (uefi, device-tree) |
| `CONDITION_VIRTUALIZATION` | ConditionVirtualization= | 虚拟化环境 (kvm, docker, none...) |
| `CONDITION_HOST` | ConditionHost= | 主机名或 machine-id |
| `CONDITION_KERNEL_COMMAND_LINE` | ConditionKernelCommandLine= | 内核启动参数 |
| `CONDITION_KERNEL_VERSION` | ConditionKernelVersion= | 内核版本（支持比较运算符） |
| `CONDITION_SECURITY` | ConditionSecurity= | 安全框架 (selinux, apparmor, tomoyo...) |
| `CONDITION_CAPABILITY` | ConditionCapability= | 当前进程拥有的 capability |
| `CONDITION_AC_POWER` | ConditionACPower= | 是否连接电源 |

### 2.2 硬件资源类

| 枚举值 | 配置指令 | 检查内容 |
|--------|----------|----------|
| `CONDITION_MEMORY` | ConditionMemory= | 物理内存大小（支持比较） |
| `CONDITION_CPUS` | ConditionCPUs= | CPU 核心数（支持比较） |
| `CONDITION_CPU_FEATURE` | ConditionCPUFeature= | CPU 特性 (sse4, avx2...) |

### 2.3 系统压力类

| 枚举值 | 配置指令 | 检查内容 |
|--------|----------|----------|
| `CONDITION_MEMORY_PRESSURE` | ConditionMemoryPressure= | 内存压力 PSI 值 |
| `CONDITION_CPU_PRESSURE` | ConditionCPUPressure= | CPU 压力 PSI 值 |
| `CONDITION_IO_PRESSURE` | ConditionIOPressure= | IO 压力 PSI 值 |

### 2.4 文件系统类

| 枚举值 | 配置指令 | 检查内容 |
|--------|----------|----------|
| `CONDITION_PATH_EXISTS` | ConditionPathExists= | 路径存在 |
| `CONDITION_PATH_EXISTS_GLOB` | ConditionPathExistsGlob= | 通配路径存在 |
| `CONDITION_PATH_IS_DIRECTORY` | ConditionPathIsDirectory= | 是目录 |
| `CONDITION_PATH_IS_SYMBOLIC_LINK` | ConditionPathIsSymbolicLink= | 是符号链接 |
| `CONDITION_PATH_IS_MOUNT_POINT` | ConditionPathIsMountPoint= | 是挂载点 |
| `CONDITION_PATH_IS_READ_WRITE` | ConditionPathIsReadWrite= | 可读写 |
| `CONDITION_PATH_IS_ENCRYPTED` | ConditionPathIsEncrypted= | 是加密卷 |
| `CONDITION_DIRECTORY_NOT_EMPTY` | ConditionDirectoryNotEmpty= | 目录非空 |
| `CONDITION_FILE_NOT_EMPTY` | ConditionFileNotEmpty= | 文件非空 |
| `CONDITION_FILE_IS_EXECUTABLE` | ConditionFileIsExecutable= | 文件可执行 |

### 2.5 系统状态类

| 枚举值 | 配置指令 | 检查内容 |
|--------|----------|----------|
| `CONDITION_NEEDS_UPDATE` | ConditionNeedsUpdate= | /usr 是否比 /etc 新 |
| `CONDITION_FIRST_BOOT` | ConditionFirstBoot= | 是否首次启动 |

### 2.6 用户/组/控制器类

| 枚举值 | 配置指令 | 检查内容 |
|--------|----------|----------|
| `CONDITION_USER` | ConditionUser= | 当前用户（UID 或名称） |
| `CONDITION_GROUP` | ConditionGroup= | 当前组（GID 或名称） |
| `CONDITION_CONTROL_GROUP_CONTROLLER` | ConditionControlGroupController= | cgroup 控制器可用 |
| `CONDITION_ENVIRONMENT` | ConditionEnvironment= | 环境变量设置 |
| `CONDITION_OS_RELEASE` | ConditionOSRelease= | os-release 字段匹配 |

---

## 3. Condition 结构体（condition.h）

```c
typedef struct Condition {
    ConditionType type;         // 条件类型
    char *parameter;            // 参数（如路径、值）
    bool trigger;               // '|' 前缀 — OR 触发模式
    bool negate;                // '!' 前缀 — 取反

    ConditionResult result;     // 上次测试结果缓存
    LIST_FIELDS(Condition, conditions); // 链表
} Condition;

typedef enum ConditionResult {
    CONDITION_UNTESTED,         // 未测试
    CONDITION_SUCCEEDED,        // 通过
    CONDITION_FAILED,           // 未通过
    CONDITION_ERROR,            // 测试出错
} ConditionResult;
```

---

## 4. 条件检查核心逻辑

### 4.1 condition_test()（condition.c:1113-1165）

中央派发函数，通过函数指针表分派到具体测试：

```c
int condition_test(Condition *c, char **env):
  // 派发表（与 ConditionType 枚举对齐）
  static const condition_test_func_t table[] = {
      [CONDITION_PATH_EXISTS]         = condition_test_path_exists,
      [CONDITION_PATH_IS_DIRECTORY]   = condition_test_path_is_directory,
      [CONDITION_KERNEL_COMMAND_LINE] = condition_test_kernel_command_line,
      [CONDITION_KERNEL_VERSION]      = condition_test_kernel_version,
      [CONDITION_VIRTUALIZATION]      = condition_test_virtualization,
      [CONDITION_SECURITY]            = condition_test_security,
      [CONDITION_CAPABILITY]          = condition_test_capability,
      [CONDITION_AC_POWER]            = condition_test_ac_power,
      [CONDITION_MEMORY]              = condition_test_memory,
      [CONDITION_CPUS]                = condition_test_cpus,
      [CONDITION_USER]                = condition_test_user,
      [CONDITION_GROUP]               = condition_test_group,
      // ... 所有 32 种
  };

  r = table[c->type](c, env);  // 调用具体测试函数

  // 应用取反
  b = (r > 0) == !c->negate;
  c->result = b ? CONDITION_SUCCEEDED : CONDITION_FAILED;
  return b;
```

### 4.2 condition_test_list()（condition.c:1167-1217）

列表级别的 AND/OR 逻辑：

```c
int condition_test_list(Condition *first, char **env,
                        condition_to_string_t to_string,
                        condition_test_logger_t logger):
  // 空列表 = 通过（无条件）
  if (!first) return 1;

  bool has_trigger = false;    // 是否有 '|' 标记的条件
  bool trigger_passed = false; // 至少一个 trigger 通过

  LIST_FOREACH(conditions, c, first):
    r = condition_test(c, env);

    if (c->trigger):
      has_trigger = true;
      if (r > 0):
        trigger_passed = true;  // OR 语义
    else:
      if (r <= 0):
        return 0;   // 非 trigger 条件必须全部通过（AND 语义）

  // trigger 条件: 至少一个通过
  if (has_trigger && !trigger_passed):
    return 0;

  return 1;  // 全部通过
```

**逻辑规则**：
- 普通条件（无 `|`）：**全部必须通过**（AND）
- 触发条件（有 `|`）：**至少一个通过**（OR）
- 两类条件同时存在时：AND 条件全部通过 **且** OR 条件至少一个通过

---

## 5. 启动路径中的条件检查（unit.c:1890-1900）

```c
// unit.c 中 unit_start() 相关逻辑:

r = unit_test_condition(u);    // 测试所有 Condition
if (r == 0):
  log_unit_debug(u, "Starting requested but condition not met. Not starting unit.");
  return -ERFKILL;  // 跳过启动（不是错误）

r = unit_test_assert(u);       // 测试所有 Assert
if (r == 0):
  log_unit_notice(u, "Starting requested but asserts failed.");
  return -EPROTO;   // 断言失败（是错误）
```

### 5.1 unit_test_condition()（unit.c:1731-1753）

```c
int unit_test_condition(Unit *u):
  // 使用 condition_test_list() 测试 u->conditions 链表
  // 结果缓存在 u->condition_result
  // 时间戳记录在 u->condition_timestamp
  r = condition_test_list(u->conditions, ...)
  u->condition_result = r > 0 ? CONDITION_SUCCEEDED : CONDITION_FAILED;
  return r;
```

---

## 6. 配置解析

### 6.1 gperf 映射（load-fragment-gperf.gperf.in:317-377）

```
Unit.ConditionPathExists        → config_parse_unit_condition_path
Unit.ConditionPathIsDirectory   → config_parse_unit_condition_path
Unit.ConditionFileNotEmpty      → config_parse_unit_condition_path
Unit.ConditionKernelCommandLine → config_parse_unit_condition_string
Unit.ConditionVirtualization    → config_parse_unit_condition_string
Unit.ConditionSecurity          → config_parse_unit_condition_string
Unit.ConditionUser              → config_parse_unit_condition_string
Unit.AssertPathExists           → config_parse_unit_condition_path
Unit.AssertUser                 → config_parse_unit_condition_string
// ... 所有 Condition* 和 Assert* 指令
```

### 6.2 前缀解析（load-fragment.c:3122-3128）

```c
config_parse_unit_condition_path(..., rvalue):
  // 处理 '|' 和 '!' 前缀
  trigger = rvalue[0] == '|';
  if (trigger) rvalue++;

  negate = rvalue[0] == '!';
  if (negate) rvalue++;

  // 创建 Condition 并加入链表
  c = condition_new(type, rvalue, trigger, negate);
  LIST_APPEND(conditions, u->conditions, c);  // 或 u->asserts
```

### 6.3 前缀组合示例

```ini
# 普通条件（AND）
ConditionPathExists=/run/my.lock

# 取反
ConditionPathExists=!/run/my.lock      # 路径不存在时通过

# 触发模式（OR）
ConditionPathExists=|/run/option-a
ConditionPathExists=|/run/option-b     # 任一存在即通过

# 取反 + 触发
ConditionPathExists=|!/run/disabled    # 任一不存在即通过
```

---

## 7. 关键条件实现详解

### 7.1 ConditionPathExists（condition.c:859-865）

```c
condition_test_path_exists(Condition *c, char **env):
  return access(c->parameter, F_OK) >= 0;
```

### 7.2 ConditionKernelCommandLine（condition.c:104-144）

```c
condition_test_kernel_command_line(Condition *c, char **env):
  // 读取 /proc/cmdline
  r = proc_cmdline_get_key(c->parameter, ...);
  if (包含 '='):
    // "key=value" 模式: 检查 key 存在且值匹配
  else:
    // "key" 模式: 只检查 key 存在
```

### 7.3 ConditionKernelVersion（condition.c:213-269）

```c
condition_test_kernel_version(Condition *c, char **env):
  uname(&u);  // 获取内核版本字符串

  // 支持比较运算符: >=, <=, !=, <, >, =
  if (parameter 以比较运算符开头):
    return version_compare(u.release, parameter)
  else:
    // fnmatch 通配匹配
    return fnmatch(parameter, u.release, 0) == 0;
```

### 7.4 ConditionVirtualization（condition.c:463-491）

```c
condition_test_virtualization(Condition *c, char **env):
  v = detect_virtualization();  // 检测虚拟化类型

  if (streq(c->parameter, "vm"))
    return VIRTUALIZATION_IS_VM(v);
  if (streq(c->parameter, "container"))
    return VIRTUALIZATION_IS_CONTAINER(v);
  if (streq(c->parameter, "private-users"))
    return running_in_userns();

  // 精确匹配: kvm, qemu, vmware, docker, lxc, ...
  return v == virtualization_from_string(c->parameter);
```

### 7.5 ConditionSecurity（condition.c:652-675）

```c
condition_test_security(Condition *c, char **env):
  if (streq(c->parameter, "selinux"))
    return mac_selinux_use();
  if (streq(c->parameter, "apparmor"))
    return mac_apparmor_use();
  if (streq(c->parameter, "tomoyo"))
    return mac_tomoyo_use();
  if (streq(c->parameter, "smack"))
    return mac_smack_use();
  if (streq(c->parameter, "ima"))
    return use_ima();
  if (streq(c->parameter, "audit"))
    return use_audit();
  if (streq(c->parameter, "uefi-secureboot"))
    return is_efi_secure_boot();
  if (streq(c->parameter, "tpm2"))
    return tpm2_support() > 0;
```

### 7.6 ConditionMemory / ConditionCPUs（condition.c:326-374）

```c
condition_test_memory(Condition *c, char **env):
  // 解析参数: ">4G", "<=512M", "8G" (等价 >=)
  m = physical_memory();  // 从 sysconf(_SC_PHYS_PAGES) 获取
  return compare(m, order, parameter_value);

condition_test_cpus(Condition *c, char **env):
  n = cpus_in_affinity_mask();  // sched_getaffinity
  return compare(n, order, parameter_value);
```

### 7.7 ConditionACPower（condition.c:612-624）

```c
condition_test_ac_power(Condition *c, char **env):
  // 读取 /sys/class/power_supply/*/online
  r = on_ac_power();
  if (streq(c->parameter, "true"))
    return r > 0;
  else
    return r == 0;
```

### 7.8 ConditionNeedsUpdate（condition.c:720-803）

```c
condition_test_needs_update(Condition *c, char **env):
  // 比较 /usr 和 /etc 的修改时间
  // 如果 /usr 比 /etc 新，说明系统更新后需要重新生成配置
  // 用于 systemd-update-done.service 等
  usr_mtime = stat("/usr").st_mtim;
  etc_mtime = stat(c->parameter).st_mtim;  // 通常是 "/etc" 或 "/var"
  return usr_mtime > etc_mtime;
```

### 7.9 ConditionFirstBoot（condition.c:805-828）

```c
condition_test_first_boot(Condition *c, char **env):
  // 检查 /run/systemd/first-boot 是否存在
  // 首次启动时由 systemd 创建
  r = access("/run/systemd/first-boot", F_OK);
  if (streq(c->parameter, "true"))
    return r >= 0;
  else
    return r < 0;
```

### 7.10 ConditionUser / ConditionGroup（condition.c:376-461）

```c
condition_test_user(Condition *c, char **env):
  if (streq(c->parameter, "@system"))
    return uid_is_system(getuid());
  // 支持 UID 数字或用户名
  uid = parse_uid_or_lookup(c->parameter);
  return getuid() == uid || geteuid() == uid;

condition_test_group(Condition *c, char **env):
  gid = parse_gid_or_lookup(c->parameter);
  // 检查主组和所有附加组
  return getgid() == gid || in_supplementary_groups(gid);
```

---

## 8. 触发模式（|）与普通模式的组合

### 8.1 完整逻辑表

```
假设配置:
  ConditionPathExists=/a          # 普通 (AND)
  ConditionPathExists=/b          # 普通 (AND)
  ConditionPathExists=|/c         # 触发 (OR)
  ConditionPathExists=|/d         # 触发 (OR)

最终条件:
  (/a 存在) AND (/b 存在) AND (/c 存在 OR /d 存在)
```

### 8.2 实际应用

```ini
# 场景: 只在物理机或 KVM 虚拟机上启动（不在容器中）
[Unit]
ConditionVirtualization=|no
ConditionVirtualization=|kvm

# 场景: 网络配置存在时才启动
[Unit]
ConditionPathExists=/etc/myapp/network.conf
ConditionPathIsReadWrite=/run

# 场景: 非首次启动时执行
[Unit]
ConditionFirstBoot=!true
```

---

## 9. 条件结果缓存与展示

### 9.1 结果缓存

```c
// 每次 unit_test_condition() 后:
u->condition_result = CONDITION_SUCCEEDED / CONDITION_FAILED;
u->condition_timestamp = now(CLOCK_MONOTONIC);
```

### 9.2 systemctl 展示

```bash
$ systemctl show myservice -p ConditionResult
ConditionResult=yes

$ systemctl status myservice
  Condition: start condition failed at ...
    ├─ ConditionPathExists=/run/my.lock was not met
    └─ ...
```

---

## 10. Assert 与 Condition 的内部统一

两者共享完全相同的：
- 数据结构（`Condition`）
- 解析逻辑（`config_parse_unit_condition_*`）
- 测试函数（`condition_test_list()`）
- 存储位置不同：`u->conditions` vs `u->asserts`

唯一区别在 `unit_start()` 中：
```c
// Condition 失败: 返回 -ERFKILL → Job 结果为 "done"（跳过）
// Assert 失败: 返回 -EPROTO → Job 结果为 "failed"（错误）
```

---

## 11. 设计要点

### 11.1 零开销设计

- 条件检查只在 Unit 启动时执行一次
- 不进行持续监控（那是 Path 单元的工作）
- 结果被缓存，restart 时重新检查

### 11.2 环境自适应

通过条件系统，同一个 Unit 文件可以：
- 在物理机和容器中运行不同行为
- 在首次启动和常规启动时执行不同任务
- 根据硬件配置自动启用/禁用功能

### 11.3 可组合的布尔逻辑

- `!` 取反 + `|` OR + 默认 AND = 完整的布尔表达能力
- 不需要独立的条件表达式语言
- 每行一个原子条件，可读性好
