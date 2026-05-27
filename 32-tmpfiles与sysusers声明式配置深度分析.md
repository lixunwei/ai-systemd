# systemd-tmpfiles 与 systemd-sysusers 声明式系统配置深度分析

## 1. 概述

systemd 提供两个声明式系统配置工具，在启动早期批量创建/维护系统必需的文件和用户：

| 工具 | 用途 | 配置目录 | 执行时机 |
|------|------|---------|---------|
| `systemd-tmpfiles` | 创建/清理/删除文件和目录 | `tmpfiles.d/` | 启动时 + 定时器 |
| `systemd-sysusers` | 创建系统用户和组 | `sysusers.d/` | 启动早期(sysinit) |

核心设计理念：**声明式配置取代命令式脚本**，使包安装时只需放置配置文件，无需编写和维护创建脚本。

---

## 2. systemd-tmpfiles

### 2.1 配置文件搜索路径（优先级从高到低）

```
/etc/tmpfiles.d/            ← 管理员配置（最高优先）
/run/tmpfiles.d/            ← 运行时配置
/usr/local/lib/tmpfiles.d/  ← 本地安装
/usr/lib/tmpfiles.d/        ← 发行版供应商
```

同名文件高优先级覆盖低优先级（整文件替换）。

源文件: `tmpfiles.c:3729-3751`，使用 `CONF_PATHS_NULSTR("tmpfiles.d")`

### 2.2 配置行格式

```
Type Path Mode User Group Age Argument
```

示例：
```
d   /run/myapp     0755 root root  -     -
f   /tmp/foo.conf  0644 root root  10d   "default content"
L   /etc/os        -    -    -     -     ../usr/lib/os-release
```

### 2.3 Item 类型完整列表

| 类型 | 枚举名 | 操作 | 接受 glob |
|------|--------|------|-----------|
| `f` | CREATE_FILE | 创建文件（不存在时），可写入内容 | 否 |
| `F` | TRUNCATE_FILE | 创建/截断文件（已弃用，用 `f+`） | 否 |
| `d` | CREATE_DIRECTORY | 创建目录 | 否 |
| `D` | TRUNCATE_DIRECTORY | 创建目录，清理时清空内容 | 否 |
| `v` | CREATE_SUBVOLUME | 创建 btrfs 子卷（回退为普通目录） | 否 |
| `q` | CREATE_SUBVOLUME_INHERIT_QUOTA | 创建子卷，继承父配额组 | 否 |
| `Q` | CREATE_SUBVOLUME_NEW_QUOTA | 创建子卷，建立新配额组 | 否 |
| `p` | CREATE_FIFO | 创建命名管道 | 否 |
| `L` | CREATE_SYMLINK | 创建符号链接 | 否 |
| `c` | CREATE_CHAR_DEVICE | 创建字符设备节点 | 否 |
| `b` | CREATE_BLOCK_DEVICE | 创建块设备节点 | 否 |
| `C` | COPY_FILES | 递归复制文件/目录树 | 否 |
| `w` | WRITE_FILE | 写入内容到已存在的文件 | **是** |
| `e` | EMPTY_DIRECTORY | 确保目录存在并调整属性 | **是** |
| `t` | SET_XATTR | 设置扩展属性 | **是** |
| `T` | RECURSIVE_SET_XATTR | 递归设置扩展属性 | **是** |
| `a` | SET_ACL | 设置 POSIX ACL | **是** |
| `A` | RECURSIVE_SET_ACL | 递归设置 ACL | **是** |
| `h` | SET_ATTRIBUTE | 设置文件属性（chattr flags） | **是** |
| `H` | RECURSIVE_SET_ATTRIBUTE | 递归设置文件属性 | **是** |
| `x` | IGNORE_PATH | 清理时忽略该路径 | **是** |
| `X` | IGNORE_DIRECTORY_PATH | 清理时忽略目录（但可清理内容） | **是** |
| `r` | REMOVE_PATH | 删除文件/空目录 | **是** |
| `R` | RECURSIVE_REMOVE_PATH | 递归删除 | **是** |
| `z` | RELABEL_PATH | 重置所有权/权限/SELinux标签 | **是** |
| `Z` | RECURSIVE_RELABEL_PATH | 递归重置标签 | **是** |
| `m` | ADJUST_MODE | 调整权限（遗留，等同于 `z`） | **是** |

源文件: `tmpfiles.c:81-112`

### 2.4 `+` 后缀和 `!` `~` 修饰符

| 修饰符 | 含义 |
|--------|------|
| `+` | 追加语义（如 `f+` = 创建并追加写入） |
| `!` | 仅在启动时执行（`--boot` 标志） |
| `~` | 仅在首次 age 时保留顶层目录 |

### 2.5 三种运行模式

```bash
systemd-tmpfiles --create    # 创建文件/目录/设备
systemd-tmpfiles --clean     # 清理超龄文件
systemd-tmpfiles --remove    # 删除文件
```

执行顺序（`run()` 函数）:

```c
// tmpfiles.c:3821-3848
// Phase 1: 处理 remove 和 clean（先删后建）
// Phase 2: 处理 create
```

源文件: `tmpfiles.c:3690-3858`

### 2.6 Age 清理算法

配置格式: `Age` 字段支持标准时间表达式（如 `10d`, `1w`, `30m`）。

```
d /tmp 1777 root root 10d
```

清理判断（`needs_cleanup()`）:

```c
// tmpfiles.c:523-575
// 检查 atime, mtime, ctime, btime (birth time)
// 文件"年龄" = now - max(atime, mtime, ctime, btime)
// 若年龄 > 配置的 Age 值 → 需要清理
```

`age-by:` 扩展语法可精确指定使用哪些时间戳：

```
d /tmp 1777 root root age-by:abcm:10d
```

- `a` = atime, `b` = btime, `c` = ctime, `m` = mtime

递归清理实现（`dir_cleanup()`）:

```c
// tmpfiles.c:577-810
// 递归遍历目录
// 跳过 x/X 标记的路径
// 检查每个文件的年龄
// 删除超龄文件
// 空目录在其 age 过期后也被删除
```

### 2.7 Glob 展开与递归操作

```c
// tmpfiles.c:1962-2018
glob_item()              // 对 glob 模式匹配的每个路径执行操作
glob_item_recursively()  // 递归版本（用于 T/A/H/Z 等递归类型）
```

### 2.8 所有权与权限解析

```c
// tmpfiles.c:2829-2877
find_uid() / find_gid()
// 解析顺序:
// 1. 数字 UID/GID
// 2. NSS 查找（getpwnam/getgrnam）
// 3. 离线 passwd/group 文件解析（用于 --root= 场景）
// 4. "root" 特殊处理
```

ACL 支持:
```c
// tmpfiles.c:1075-1192
parse_acls_from_arg()  // 解析 ACL 字符串
fd_set_acls()          // 应用 ACL
```

xattr 支持:
```c
// tmpfiles.c:1008-1073
parse_xattrs_from_arg()  // 解析 "name=value" 格式
fd_set_xattrs()          // 设置扩展属性
```

### 2.9 路径说明符（Specifiers）

| 说明符 | 含义 |
|--------|------|
| `%a` | 架构 |
| `%b` | boot ID |
| `%B` | OS build ID |
| `%H` | 主机名 |
| `%l` | 短主机名 |
| `%m` | machine ID |
| `%M` | OS image ID |
| `%o` | OS ID |
| `%v` | 内核版本 |
| `%w` | OS version ID |
| `%W` | OS variant ID |
| `%T` | `/tmp` 目录 |
| `%V` | `/var/tmp` 目录 |
| `%C` | 缓存目录 |
| `%L` | 日志目录 |
| `%S` | 状态目录 |
| `%t` | 运行时目录 |
| `%%` | 字面 `%` |

源文件: `tmpfiles.c:207-291`，使用 `COMMON_SYSTEM_SPECIFIERS` 和 `COMMON_TMP_SPECIFIERS`

### 2.10 Btrfs 子卷操作

```c
// tmpfiles.c:1622-1745
create_subvolume()
create_directory_or_subvolume()
// v: 创建子卷，无配额设置
// q: 创建子卷，继承父配额组（qgroup 2/N）
// Q: 创建子卷，创建新配额组
// 若 btrfs 不可用，回退为普通 mkdir()
```

### 2.11 执行时机

| 服务 | 模式 | 时机 |
|------|------|------|
| `systemd-tmpfiles-setup.service` | `--create --remove` | sysinit.target |
| `systemd-tmpfiles-setup-dev.service` | `--create` (仅 /dev) | local-fs-pre.target 前 |
| `systemd-tmpfiles-clean.timer` | `--clean` | 启动 15min 后，每天一次 |

---

## 3. systemd-sysusers

### 3.1 配置文件搜索路径

```
/etc/sysusers.d/            ← 管理员配置
/run/sysusers.d/            ← 运行时配置
/usr/local/lib/sysusers.d/  ← 本地安装
/usr/lib/sysusers.d/        ← 发行版供应商
```

同名文件覆盖规则与 tmpfiles 相同。

源文件: 使用 `CONF_PATHS_STRV("sysusers.d")`

### 3.2 配置行格式

```
Type Name ID GECOS HomeDir Shell
```

### 3.3 指令类型

| 类型 | 枚举 | 含义 | 示例 |
|------|------|------|------|
| `u` | ADD_USER | 创建系统用户（同时创建同名组） | `u systemd-journal 190 "Journal Daemon"` |
| `g` | ADD_GROUP | 仅创建组 | `g input 104` |
| `m` | ADD_MEMBER | 将用户加入组 | `m _flatpak video` |
| `r` | ADD_RANGE | 声明 UID/GID 分配范围 | `r 100-999` |

源文件: `sysusers.c:40-45`

### 3.4 UID/GID 字段格式

| 格式 | 含义 |
|------|------|
| `500` | 固定 UID=500 |
| `100-999` | 范围（仅用于 `r` 类型） |
| `500:600` | UID=500, GID=600 |
| `-` | 自动分配 |
| `65534` | nfsnobody 等特殊 ID |

### 3.5 UID/GID 分配算法

#### 范围确定

```c
// uid-alloc-range.c:33-87
read_login_defs()
// 从 /etc/login.defs 读取:
//   SYS_UID_MIN (默认 100)
//   SYS_UID_MAX (默认 999)
//   SYS_GID_MIN (默认 100)
//   SYS_GID_MAX (默认 999)
```

#### 分配策略

```c
// sysusers.c:1137-1155 (用户)
// sysusers.c:1308-1327 (组)
uid_range_next_lower()
// 从范围的 最高端向低端 搜索空闲 ID
// 检查 /etc/passwd 和 /etc/group 中不存在该 ID
// 检查 NSS 不返回该 ID
```

搜索方向：**从高到低**（999→100），这样新分配的系统用户 ID 从高端开始，与手动创建的低端 ID 不冲突。

### 3.6 冲突处理

```c
// sysusers.c:939-991
uid_is_ok()
// 检查顺序:
// 1. UID 0 始终被占用
// 2. 检查本地 /etc/passwd
// 3. 检查 NSS (getpwuid)
// 4. 检查是否在已分配列表中

// sysusers.c:1174-1208
gid_is_ok()
// 同理检查 /etc/group + NSS
```

已存在用户的处理：
- 用户已存在且 UID 匹配 → 跳过，不报错
- 用户已存在但 UID 不匹配 → 跳过并发出警告
- 用户名冲突 → 报错中止

源文件: `sysusers.c:1762-1770`

### 3.7 文件修改方式

**直接文件 I/O，不使用 NSS 写入**：

```c
// sysusers.c:399-520   — write_temporary_passwd()
// sysusers.c:522-658   — write_temporary_group()
// sysusers.c:660-844   — write_temporary_shadow() / gshadow

// 流程:
// 1. 读取现有 /etc/passwd 到内存
// 2. 追加新条目到临时文件
// 3. rename() 临时文件覆盖原文件（原子替换）
```

原子替换确保崩溃安全:
```c
// sysusers.c:846-936
// 同时处理 passwd + shadow + group + gshadow 四个文件
// 全部写入临时文件后，批量 rename()
```

### 3.8 Shadow 密码处理

| 场景 | 行为 |
|------|------|
| 默认 | 锁定密码（`!*`） |
| 有 `passwd.hashed-password.$NAME` 凭据 | 使用凭据中的哈希 |
| 有 `passwd.plaintext-password.$NAME` 凭据 | 用 crypt() 哈希后写入 |

```c
// sysusers.c:600-626
// 凭据搜索: 先查 hashed-password，再查 plaintext-password
```

### 3.9 Shell 凭据

```c
// sysusers.c:481-489
// 凭据 "passwd.shell.$NAME" 可覆盖默认 shell
// 默认系统用户 shell = /usr/sbin/nologin
```

### 3.10 路径说明符

sysusers 配置支持 `%` 说明符展开（与 tmpfiles 相同的 `system_and_tmp_specifier_table`）。

```c
// sysusers.c:1545-1609
// 对 Name, ID, GECOS, HomeDir, Shell 字段进行说明符展开
```

### 3.11 执行时机

```ini
# systemd-sysusers.service
[Unit]
DefaultDependencies=no
Before=sysinit.target systemd-update-done.service
After=systemd-remount-fs.service
ConditionNeedsUpdate=/etc

[Service]
Type=oneshot
ExecStart=systemd-sysusers
```

在文件系统可写之后、其他服务启动之前执行，确保系统用户在需要它们的服务启动前就绑存在。

---

## 4. 对比与协作

### 4.1 执行顺序

```
sysinit.target 启动链:
  systemd-sysusers.service     ← 先创建用户
    ↓
  systemd-tmpfiles-setup.service  ← 再创建文件（可引用用户）
```

### 4.2 设计对比

| 维度 | tmpfiles | sysusers |
|------|---------|---------|
| 配置格式 | 单字母类型 + 路径 + 权限 | 单字母类型 + 名称 + ID |
| 操作对象 | 文件/目录/设备/链接/属性 | 用户/组/成员 |
| 幂等性 | 已存在则跳过（大多数类型） | 已存在则跳过 |
| 修改方式 | 直接文件系统操作 | 原子文件替换 |
| 周期执行 | 是（clean 定时器） | 否（仅启动时） |
| 安全特性 | MAC 标签重置 | shadow 锁定密码 |
| `--root=` | 支持（离线操作） | 支持（用于镜像构建） |

### 4.3 包管理器集成

```
RPM/DEB 包安装时:
  1. 放置 /usr/lib/sysusers.d/myapp.conf
  2. 放置 /usr/lib/tmpfiles.d/myapp.conf
  3. 运行 systemd-sysusers (创建用户)
  4. 运行 systemd-tmpfiles --create (创建运行时目录)
```

无需 %pre/%post 脚本中手动 `useradd`/`mkdir`。

---

## 5. 高级用法

### 5.1 tmpfiles 凭据和条件

```
# 仅在特定架构上执行
C /usr/lib/firmware - - - - /run/credentials/...

# 使用凭据目录作为源
C /etc/ssl/certs - - - - %d/ssl-certs
```

### 5.2 sysusers 与 DynamicUser= 的关系

`systemd-sysusers` 创建**静态**系统用户（写入 /etc/passwd），适用于：
- 需要固定 UID 的服务（如 systemd-journal: 190）
- 需要 home 目录的守护进程
- 需要被多个服务引用的用户

`DynamicUser=yes` 创建**动态**临时用户（通过 NSS 模块），适用于：
- 无状态服务
- 不需要持久化数据的服务
- UID 不需要稳定的场景

两者互补，不冲突。

### 5.3 tmpfiles 子卷与配额

```
# 创建 btrfs 子卷，继承父配额
q /var/lib/machines 0755 root root -

# 创建子卷，建立新配额组
Q /var/lib/portables 0755 root root -
```

回退策略：若文件系统不是 btrfs，自动降级为 `mkdir()`。

---

## 6. 错误处理与安全

### 6.1 tmpfiles 安全考量

- **路径遍历保护**：使用 `O_NOFOLLOW` + `openat()` 避免符号链接攻击
- **权限设置顺序**：先 chown 再 chmod，避免竞态
- **MAC 标签**：创建文件后立即设置 SELinux 上下文

### 6.2 sysusers 安全考量

- **原子替换**：passwd/group 文件修改是原子的（rename）
- **锁定密码**：默认 shadow 条目为 `!*`（不可登录）
- **凭据隔离**：密码凭据仅通过受控的凭据机制传入

### 6.3 幂等性保证

两个工具都设计为可重复执行：
- tmpfiles：文件已存在且属性匹配 → 跳过
- sysusers：用户已存在且 UID 匹配 → 跳过

这使得它们可以安全地在每次启动时运行。

---

## 7. 设计哲学

1. **声明式优于命令式**：描述"期望状态"而非"操作步骤"，系统自行收敛到目标状态

2. **零配置默认值**：包只需提供 `.conf` 文件，systemd 负责正确时机执行

3. **幂等安全**：任何时候重复执行都不会造成破坏，适合启动时无条件运行

4. **离线操作**：`--root=` 参数支持对未启动的系统镜像操作，适配镜像构建流程

5. **优先级覆盖**：`/etc` > `/run` > `/usr` 的层级允许管理员覆盖供应商默认而不修改包文件

6. **最小化依赖**：sysusers 直接操作文件（不依赖 NSS 写入），tmpfiles 使用原生系统调用（不依赖 shell），确保在启动早期可靠运行

7. **原子性**：sysusers 的 rename 替换和 tmpfiles 的 `O_TMPFILE` + `linkat()` 确保崩溃安全
