# 日志系统分析 — systemd-journald

## 一、概述

`systemd-journald` 是 systemd 的集中日志守护进程，替代传统的 syslog。它接收来自多种来源的日志，以结构化的二进制格式存储，支持索引查询。

源码位于 `src/journal/`。

---

## 二、核心架构

```
                日志来源
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ native   │ syslog   │  kmsg    │ stream   │  audit   │
│ /dev/log │ /dev/log │ /dev/kmsg│ stdout/  │ audit    │
│ journal  │ compat   │ 内核日志  │ stderr   │ 审计日志  │
└────┬─────┴────┬─────┴────┬─────┴────┬─────┴────┬─────┘
     │          │          │          │          │
     ▼          ▼          ▼          ▼          ▼
┌──────────────────────────────────────────────────────┐
│                   Server（核心）                      │
│  • 消息接收与解析                                      │
│  • 速率限制                                           │
│  • 存储写入                                           │
│  • 空间管理（vacuum）                                  │
│  • 密封与认证                                         │
└───────────────────────┬──────────────────────────────┘
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
     /var/log/     /run/log/     console
     journal/      journal/      控制台输出
     (持久化)      (运行时)
```

---

## 三、关键源文件

| 文件 | 职责 |
|------|------|
| journald.c | 守护进程入口（初始化 Server、进入事件循环） |
| journald-server.c | Server 结构体管理、核心调度逻辑 |
| journald-native.c | 原生协议（/run/systemd/journal/socket）消息处理 |
| journald-syslog.c | syslog 协议兼容（/dev/log）消息处理 |
| journald-kmsg.c | 内核日志（/dev/kmsg）消息处理 |
| journald-stream.c | 标准输出/标准错误流日志处理 |
| journald-audit.c | 审计日志处理 |
| journald-console.c | 控制台输出 |
| journald-context.c | 客户端上下文管理（进程元数据缓存） |
| journald-rate-limit.c | 日志速率限制 |

---

## 四、Server 结构体

`Server` 是 journald 的核心数据结构，包含：

| 字段分类 | 包含内容 |
|---------|---------|
| 事件循环 | sd_event 实例 |
| 存储管理 | 系统/运行时 JournalFile、存储策略、压缩设置 |
| 输入端 | native/syslog/kmsg/audit socket fd |
| 流管理 | stdout/stderr 流连接列表 |
| 速率限制 | 每个客户端的速率限制状态 |
| 空间管理 | max_use、max_file_size、keep_free 等 |
| 转发 | 转发到 syslog、kmsg、console、wall 的开关 |
| 密封 | HMAC 密封密钥和状态 |

---

## 五、日志处理流程

### 5.1 消息接收

```
客户端调用 sd_journal_send() 或写入 /dev/log
  ↓
journald 通过 epoll 收到可读事件
  ↓
根据来源调用对应处理函数：
  • journald-native.c  → server_process_native_message()
  • journald-syslog.c  → server_process_syslog_message()
  • journald-kmsg.c    → server_process_kmsg()
  • journald-audit.c   → server_process_audit_message()
  • journald-stream.c  → stdout_stream_process()
```

### 5.2 消息写入

```
server_dispatch_message()
  ├── 速率限制检查 → journal_rate_limit_test()
  │     如果超限 → 丢弃并记录"suppressed N messages"
  ├── 添加元数据字段
  │     _PID=, _UID=, _GID=, _COMM=, _EXE=, _CMDLINE=
  │     _SYSTEMD_UNIT=, _SYSTEMD_CGROUP=, _BOOT_ID=
  ├── 写入日志文件 → journal_file_append_entry()
  ├── 转发（如果配置）
  │     → server_forward_syslog()   转发到 syslog
  │     → server_forward_kmsg()     转发到 kmsg
  │     → server_forward_console()  转发到控制台
  │     → server_forward_wall()     转发到 wall
  └── 空间检查 → 触发 vacuum（如果需要）
```

### 5.3 空间管理（Vacuum）

```
server_vacuum()
  ├── 检查存储使用量
  ├── 根据策略删除旧日志文件
  │     SystemMaxUse=         系统日志最大总量
  │     RuntimeMaxUse=        运行时日志最大总量
  │     SystemMaxFileSize=    单文件最大大小
  │     SystemKeepFree=       保留空闲空间
  │     MaxRetentionSec=      最大保留时间
  └── 文件轮转（创建新文件，归档旧文件）
```

---

## 六、日志文件格式

日志以**二进制格式**存储，文件扩展名 `.journal`：

```
┌─────────────────────┐
│    File Header      │  文件头（魔数、版本、兼容性标志）
├─────────────────────┤
│    Data Objects     │  数据对象（键值对，如 MESSAGE=Hello）
├─────────────────────┤
│   Field Objects     │  字段对象（字段名索引）
├─────────────────────┤
│   Entry Objects     │  日志条目（时间戳 + 数据对象引用列表）
├─────────────────────┤
│  Hash Table Objects │  数据和字段的哈希索引
├─────────────────────┤
│  Entry Array Objects│  条目的有序数组（支持二分查找）
├─────────────────────┤
│    Tag Objects      │  HMAC 密封标签（可选）
└─────────────────────┘
```

**关键设计特点**：
- **结构化存储**：每个字段独立存储，支持按任意字段过滤
- **去重**：相同的数据对象只存储一次（通过哈希表索引）
- **索引**：内置哈希表和有序数组，查询无需外部索引
- **密封**：支持 HMAC-SHA256 防篡改

---

## 七、存储策略

| 选项 | 值 | 说明 |
|------|-----|------|
| Storage= | volatile | 仅存储到 /run/log/journal/（重启丢失） |
| Storage= | persistent | 存储到 /var/log/journal/（持久化） |
| Storage= | auto | 如果 /var/log/journal/ 存在则持久化 |
| Storage= | none | 不存储（仅转发） |
| Compress= | yes/no | 是否压缩存储（默认 yes，使用 LZ4/XZ） |
| Seal= | yes/no | 是否启用 HMAC 密封 |

---

## 八、配置文件

主配置文件：`/etc/systemd/journald.conf`

```ini
[Journal]
Storage=auto
Compress=yes
Seal=yes
SplitMode=uid
RateLimitIntervalSec=30s
RateLimitBurst=10000
SystemMaxUse=4G
SystemKeepFree=1G
SystemMaxFileSize=128M
RuntimeMaxUse=256M
MaxRetentionSec=1month
ForwardToSyslog=no
ForwardToKMsg=no
ForwardToConsole=no
MaxLevelStore=debug
MaxLevelSyslog=debug
MaxLevelKMsg=notice
MaxLevelConsole=info
```

---

## 九、与其他组件的交互

| 交互方 | 交互方式 | 内容 |
|--------|---------|------|
| PID 1 | stdout/stderr stream | 收集服务输出日志 |
| 所有进程 | native socket | 结构化日志写入 |
| syslog 客户端 | /dev/log | syslog 兼容 |
| 内核 | /dev/kmsg | 内核日志 |
| auditd | audit socket | 审计日志 |
| journalctl | sd-journal API | 日志查询 |
| journal-remote | HTTP/HTTPS | 远程日志收集 |
| journal-upload | HTTP/HTTPS | 日志上传 |
