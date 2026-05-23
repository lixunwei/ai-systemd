# 28. Path 单元、环境变量与凭据机制深度分析

## 第一部分：Path 单元

### 1.1 概述

Path 单元通过 **inotify** 监视文件系统路径变化，当条件满足时触发关联服务。它是实现"文件驱动激活"的基础。

### 核心源文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/core/path.h` | ~67 | PathSpec/Path 结构体、PathType 枚举 |
| `src/core/path.c` | ~860 | 状态机、inotify 管理、触发逻辑 |

---

### 1.2 PathType 枚举（path.h:9-17）

```c
typedef enum PathType {
    PATH_EXISTS,              // 路径存在即触发
    PATH_EXISTS_GLOB,         // 通配路径存在即触发
    PATH_DIRECTORY_NOT_EMPTY, // 目录非空即触发
    PATH_CHANGED,             // 路径内容变化时触发（首次存在也触发）
    PATH_MODIFIED,            // 路径被修改时触发（比 CHANGED 更严格）
    _PATH_TYPE_MAX,
} PathType;
```

| 类型 | 触发条件 | 持续监控 |
|------|----------|----------|
| PathExists | 路径出现 | 触发后停止监控 |
| PathExistsGlob | 通配匹配出现 | 触发后停止监控 |
| DirectoryNotEmpty | 目录中有文件 | 触发后停止监控 |
| PathChanged | 路径创建/删除/重命名 | 持续监控 |
| PathModified | 路径写入/修改 | 持续监控 |

---

### 1.3 PathSpec 结构体（path.h:20-33）

```c
typedef struct PathSpec {
    Unit *unit;                    // 所属 Unit
    char *path;                    // 监视路径

    sd_event_source *event_source; // inotify 事件源
    int inotify_fd;                // inotify 文件描述符
    int primary_wd;                // 主 watch descriptor

    bool previous_exists;          // 上次检查时路径是否存在
    PathType type;                 // 路径类型

    LIST_FIELDS(PathSpec, spec);   // 链表
} PathSpec;
```

### 1.4 Path 结构体（path.h:54-67）

```c
struct Path {
    Unit meta;

    LIST_HEAD(PathSpec, specs);    // PathSpec 链表
    
    PathState state, deserialized_state;
    PathResult result;

    bool make_directory;           // MakeDirectory=
    mode_t directory_mode;         // DirectoryMode=（默认 0755）

    RateLimit trigger_limit;       // TriggerLimitBurst/IntervalSec
};
```

---

### 1.5 状态机

```c
// 状态: DEAD → WAITING → RUNNING → DEAD/FAILED
static const UnitActiveState state_translation_table[] = {
    [PATH_DEAD]    = UNIT_INACTIVE,
    [PATH_WAITING] = UNIT_ACTIVE,
    [PATH_RUNNING] = UNIT_ACTIVE,
    [PATH_FAILED]  = UNIT_FAILED,
};
```

```
          start()
  DEAD ──────────→ WAITING ←──────────┐
                      │                 │
                      │ 条件满足        │ 服务完成
                      ▼                 │
                   RUNNING ─────────────┘
                      │
                   stop()
                      ▼
                    DEAD
```

---

### 1.6 inotify 监视机制（path.c:37-165）

`path_spec_watch()` 的核心逻辑：

```c
path_spec_watch(PathSpec *s):
  1. inotify_init1(IN_NONBLOCK|IN_CLOEXEC)
  2. sd_event_add_io(event, &s->event_source, inotify_fd, EPOLLIN, path_dispatch_io, s)

  3. 逐级监视路径的每个组件:
     例如 /etc/myapp/config.json:
     - watch "/" (IN_MOVE_SELF|IN_DELETE_SELF|IN_ATTRIB|IN_CREATE|IN_MOVED_TO)
     - watch "/etc" (同上)
     - watch "/etc/myapp" (同上)
     - watch "/etc/myapp/config.json" ← primary_wd

  4. 对符号链接:
     - 同时监视链接本身（IN_DONT_FOLLOW）
     - 和链接目标

  5. PathChanged: IN_CREATE|IN_DELETE|IN_DELETE_SELF|IN_MOVED_FROM|IN_MOVED_TO|IN_MOVE_SELF|IN_ATTRIB
     PathModified: 以上 + IN_MODIFY|IN_CLOSE_WRITE
```

**为什么监视每个路径组件？**

如果只监视 `/etc/myapp/config.json`，当 `/etc/myapp/` 目录不存在时 inotify 会失败。通过监视每一级，可以在父目录被创建时自动重新安装子目录的 watch。

---

### 1.7 path_enter_waiting()（path.c:553-591）

```c
path_enter_waiting(Path *p, bool initial):
  1. 检查触发单元状态（如果正在运行则等待）
  2. path_check_good(p, initial)
     → 遍历所有 PathSpec，检查条件是否已满足
     → PathExists: access(path, F_OK)
     → PathExistsGlob: glob_exists(path)
     → DirectoryNotEmpty: dir_is_empty(path) == 0
     → PathChanged/Modified: access(path, F_OK) && initial
  3. 如果条件已满足 → 直接进入 RUNNING
  4. 否则:
     path_spec_watch(spec)  // 安装 inotify
     state = PATH_WAITING
  5. 安装后再次检查（防止安装期间的竞态）
```

### 1.8 path_enter_running()（path.c:503-539）

```c
path_enter_running(Path *p):
  1. 速率限制检查:
     if (!ratelimit_below(&p->trigger_limit)):
       log_unit_warning("Trigger limit hit, refusing further activation.")
       path_enter_dead(p, PATH_FAILURE_TRIGGER_LIMIT_HIT)
       return

  2. 请求启动触发单元:
     manager_add_job(m, JOB_START, trigger_unit, JOB_REPLACE, ...)

  3. state = PATH_RUNNING
```

### 1.9 MakeDirectory=

```c
// path.c:593-602 (path_start)
if (p->make_directory):
  LIST_FOREACH(spec, s, p->specs):
    // 跳过 PathExists/PathExistsGlob（不需要创建）
    if (IN_SET(s->type, PATH_EXISTS, PATH_EXISTS_GLOB))
      continue;
    mkdir_p_label(s->path, p->directory_mode);
```

### 1.10 TriggerLimitBurst= / TriggerLimitIntervalSec=

```c
// 默认值 (path.c:269-270):
p->trigger_limit.interval = 2 * USEC_PER_SEC;  // 2秒窗口
p->trigger_limit.burst = 200;                   // 最多200次触发

// 可通过配置覆盖:
TriggerLimitBurst=50
TriggerLimitIntervalSec=10s
```

防止文件系统频繁变化导致关联服务被反复启动。

---

### 1.11 配置示例

```ini
# myapp-config.path
[Path]
PathModified=/etc/myapp/config.json
MakeDirectory=yes
DirectoryMode=0755

# 速率限制
TriggerLimitBurst=5
TriggerLimitIntervalSec=60s

[Install]
WantedBy=multi-user.target
```

---

## 第二部分：环境变量机制

### 2.1 概述

systemd 提供多层环境变量控制，从 Manager 全局到单个 ExecStart 进程。

### 2.2 环境变量来源与优先级

```
最终 environ[] 合成顺序（后者覆盖前者）:

1. Manager 默认环境 (DefaultEnvironment=)
2. PassEnvironment= 从 PID 1 继承的变量
3. /etc/environment (全局)
4. EnvironmentFile= 中加载的变量
5. Environment= 直接指定的变量
6. systemd 内建变量 (MAINPID, LISTEN_FDS, CREDENTIALS_DIRECTORY 等)
7. UnsetEnvironment= 最后移除指定变量
```

### 2.3 配置指令

| 指令 | 说明 |
|------|------|
| `Environment=` | 直接设置键值对 |
| `EnvironmentFile=` | 从文件加载（支持 `-` 前缀表示可选） |
| `PassEnvironment=` | 从 PID 1 环境继承指定变量 |
| `UnsetEnvironment=` | 从最终环境中移除指定变量 |

### 2.4 Environment= 解析

```ini
Environment=VAR1=value1
Environment="VAR2=value with spaces"
Environment=VAR3=a VAR4=b    # 单行多个
```

### 2.5 EnvironmentFile= 处理（execute.c:5521-5590）

```c
exec_context_load_environment(ExecContext *c, ...):
  LIST_FOREACH(env_files):
    // '-' 前缀: 文件不存在时不报错
    if (path[0] == '-') optional = true;

    // 支持 glob 展开
    safe_glob(path, GLOB_NOSORT|GLOB_BRACE, &pglob);

    for each matched file:
      load_env_file(NULL, fn, &p);
      // 格式: KEY=VALUE（每行一个）
      // '#' 开头的行为注释
      // 无效行被忽略并记录日志

    // 合并到已有环境
    strv_env_merge(&result, p);
```

### 2.6 PassEnvironment=（execute.c:2005-2030）

```c
build_pass_environment(ExecContext *c, char ***ret):
  // 从 PID 1 进程的 environ 中选择性继承
  STRV_FOREACH(var, c->pass_environment):
    v = getenv(*var);
    if (v):
      add to result: "VAR=value"
```

### 2.7 最终环境合成（execute.c:4452-4503）

```c
// 按优先级合成（低→高）:
final_env = strv_env_merge(
    params->environment,           // Manager 全局
    our_env,                       // build_environment() 内建变量
    pass_env,                      // PassEnvironment=
    context->environment,          // Environment=
    files_env                      // EnvironmentFile=
);

// 应用 UnsetEnvironment=
strv_env_delete(final_env, context->unset_environment);
```

### 2.8 内建环境变量

| 变量 | 来源 | 说明 |
|------|------|------|
| `MAINPID` | service_set_main_pid() | 主进程 PID |
| `LISTEN_FDS` | socket activation | 传入的 FD 数量 |
| `LISTEN_FDNAMES` | socket activation | FD 名称列表 |
| `LISTEN_PID` | socket activation | 目标进程 PID |
| `NOTIFY_SOCKET` | sd_notify | 通知 socket 路径 |
| `WATCHDOG_USEC` | watchdog | 看门狗超时值 |
| `WATCHDOG_PID` | watchdog | 看门狗目标 PID |
| `CREDENTIALS_DIRECTORY` | credentials | 凭据目录路径 |
| `INVOCATION_ID` | unit start | 本次调用的唯一 ID |
| `JOURNAL_STREAM` | journal | journal 流 fd |

---

## 第三部分：凭据（Credentials）机制

### 3.1 概述

凭据是 systemd v250 引入的**安全秘密传递机制**，将密码、证书等敏感数据安全地传递给服务进程，避免在 Unit 文件中明文存储。

### 3.2 配置指令

| 指令 | 说明 |
|------|------|
| `LoadCredential=name:path` | 从文件加载凭据 |
| `LoadCredentialEncrypted=name:path` | 从加密文件加载（自动解密） |
| `SetCredential=name:value` | 直接设置凭据值 |
| `SetCredentialEncrypted=name:base64` | 设置加密的凭据值 |

### 3.3 凭据存储结构

```c
// execute.h:159-171
typedef struct ExecLoadCredential {
    char *id;           // 凭据名称
    char *path;         // 来源路径
    bool encrypted;     // 是否加密
} ExecLoadCredential;

typedef struct ExecSetCredential {
    char *id;           // 凭据名称
    void *data;         // 凭据数据
    size_t size;        // 数据大小
    bool encrypted;     // 是否加密
} ExecSetCredential;
```

### 3.4 凭据目录结构

```
/run/credentials/<unit-name>/
├── db-password          # LoadCredential=db-password:/etc/secrets/db
├── tls-cert            # LoadCredential=tls-cert:/etc/ssl/cert.pem
└── api-key             # SetCredential=api-key:xxxxx
```

### 3.5 实现流程（execute.c:2893-3163）

```c
setup_credentials(ExecContext *context, ExecParameters *params, ...):

  1. setup_credentials_internal():
     a. 创建凭据工作区:
        - 优先使用 ramfs（不交换到磁盘）
        - 回退到 tmpfs
        - 最后回退到 bind mount
     b. 权限: 0700, 属主 = 服务运行用户

  2. 处理 SetCredential=:
     for each set_credential:
       if (encrypted):
         decrypt_credential_and_warn(data) → plaintext
       write(credential_dir/name, data)
       chmod(0400)  // 只读

  3. 处理 LoadCredential=:
     for each load_credential:
       read(source_path) → data
       if (encrypted):
         decrypt_credential_and_warn(data) → plaintext
       write(credential_dir/name, data)
       chmod(0400)

  4. 设置环境变量:
     CREDENTIALS_DIRECTORY=/run/credentials/<unit>

  5. 挂载为只读:
     mount(credential_dir, MS_RDONLY|MS_REMOUNT)
```

### 3.6 安全特性

| 特性 | 实现方式 |
|------|----------|
| 内存存储 | ramfs（不写入 swap） |
| 最小权限 | 0400，属主为服务用户 |
| 只读挂载 | remount readonly |
| 加密支持 | systemd-creds 工具加密，运行时解密 |
| 隔离 | 每个 Unit 独立目录，namespace 隔离 |
| 短生命周期 | 服务停止后自动清理 |

### 3.7 加密凭据工作流

```bash
# 加密凭据（离线）
echo "my-secret" | systemd-creds encrypt - /etc/credstore/db-password.encrypted

# Unit 文件使用
[Service]
LoadCredentialEncrypted=db-password:/etc/credstore/db-password.encrypted

# 服务内读取
cat $CREDENTIALS_DIRECTORY/db-password  # → "my-secret"
```

加密密钥来源：
- TPM2（硬件绑定）
- `/var/lib/systemd/credential.secret`（主机绑定）
- 两者组合（最安全）

### 3.8 凭据搜索路径

```
系统凭据 (PID 1 启动时):
  /run/credentials/@system/
  内核命令行: systemd.set_credential=name:value

服务凭据:
  LoadCredential= 指定的路径
  /run/credentials/ 下的全局凭据
  /etc/credstore/ 和 /etc/credstore.encrypted/
```

---

## 4. 设计要点总结

### 4.1 Path 单元的多级 inotify

通过监视路径的每一级组件，即使父目录不存在也能正确工作。当父目录被创建时，自动安装子目录的 watch，实现"最终一致"的文件监控。

### 4.2 环境变量的可预测性

严格定义的合成顺序确保环境变量行为可预测。`UnsetEnvironment=` 作为最后一步，确保可以移除不想继承的变量。

### 4.3 凭据的零信任设计

- 不依赖文件系统权限（ramfs + namespace 隔离）
- 不依赖网络（本地加密/解密）
- 支持硬件绑定（TPM2）
- 自动生命周期管理（随服务创建/销毁）
