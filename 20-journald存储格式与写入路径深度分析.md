# 20. journald 存储格式与写入路径深度分析

## 概述

systemd-journald 是 systemd 的日志守护进程，负责收集来自内核、系统服务、
用户进程的日志消息，存储为结构化的二进制日志文件。其核心设计特点：

| 特性 | 实现 |
|------|------|
| **结构化存储** | 键值对格式，不是纯文本行 |
| **二进制文件** | 高效检索，支持多种索引 |
| **哈希去重** | 相同数据对象只存一份 |
| **mmap 访问** | 零拷贝读写，窗口缓存 |
| **压缩** | 支持 XZ/LZ4/ZSTD 三种算法 |
| **密封** | FSPRG + HMAC-SHA256 前向安全签名 |
| **多输入源** | syslog、kmsg、native、stdout、audit |

---

## 一、日志文件二进制格式（journal-def.h）

### 1.1 文件签名与总体布局

```
┌──────────────────────────────────────────────────────────────┐
│  Header (256 字节)                                           │
│  签名: "LPKSHHRH"                                           │
├──────────────────────────────────────────────────────────────┤
│  Arena（对象存储区）                                          │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Object 1 (DataHashTable)                              │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  Object 2 (FieldHashTable)                             │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  Object 3 (Data)                                       │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  Object 4 (Field)                                      │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  Object 5 (Entry)                                      │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  Object 6 (EntryArray)                                 │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  Object 7 (Tag) — 仅密封模式                           │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  ... 更多对象 ...                                      │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘

对齐规则：所有对象起始于 8 字节边界
最小偏移量：对象不能位于 Header 区域内（偏移量 ≥ 256）
```

### 1.2 Header 结构体（256 字节）

```c
struct Header {
    // === 基础标识（0-23）===
    uint8_t  signature[8];          // @0:   "LPKSHHRH"
    le32_t   compatible_flags;      // @8:   兼容标志（SEALED）
    le32_t   incompatible_flags;    // @12:  不兼容标志（压缩/keyed_hash）
    uint8_t  state;                 // @16:  文件状态
    uint8_t  reserved[7];           // @17:  保留

    // === ID 标识（24-87）===
    sd_id128_t file_id;             // @24:  文件唯一 ID（创建时生成）
    sd_id128_t machine_id;          // @40:  机器 ID
    sd_id128_t boot_id;             // @56:  最后写入者的 boot ID
    sd_id128_t seqnum_id;           // @72:  序号域 ID

    // === 文件布局（88-135）===
    le64_t   header_size;           // @88:  Header 大小（256）
    le64_t   arena_size;            // @96:  Arena 区大小
    le64_t   data_hash_table_offset;// @104: 数据哈希表偏移
    le64_t   data_hash_table_size;  // @112: 数据哈希表大小（字节）
    le64_t   field_hash_table_offset;// @120: 字段哈希表偏移
    le64_t   field_hash_table_size; // @128: 字段哈希表大小（字节）
    le64_t   tail_object_offset;    // @136: 最后一个对象的偏移

    // === 计数器（144-175）===
    le64_t   n_objects;             // @144: 对象总数
    le64_t   n_entries;             // @152: 条目总数
    le64_t   tail_entry_seqnum;     // @160: 最大序号
    le64_t   head_entry_seqnum;     // @168: 最小序号
    le64_t   entry_array_offset;    // @176: 全局 EntryArray 链起始偏移

    // === 时间范围（184-207）===
    le64_t   head_entry_realtime;   // @184: 最早条目实时时间
    le64_t   tail_entry_realtime;   // @192: 最新条目实时时间
    le64_t   tail_entry_monotonic;  // @200: 最新条目单调时间

    // === 扩展计数器（v187+，208-247）===
    le64_t   n_data;                // @208: Data 对象总数
    le64_t   n_fields;              // @216: Field 对象总数
    le64_t   n_tags;                // @224: Tag 对象总数
    le64_t   n_entry_arrays;        // @232: EntryArray 对象总数

    // === 哈希链深度（v246+，240-255）===
    le64_t   data_hash_chain_depth; // @240: 数据哈希最大链深
    le64_t   field_hash_chain_depth;// @248: 字段哈希最大链深
};  // sizeof == 256
```

#### 文件状态

| 值 | 常量 | 含义 |
|----|------|------|
| 0 | `STATE_OFFLINE` | 正常关闭 |
| 1 | `STATE_ONLINE` | 正在写入（异常断电 = 不干净关闭） |
| 2 | `STATE_ARCHIVED` | 已归档（只读轮转后的文件） |

#### 不兼容标志

| 标志位 | 含义 |
|--------|------|
| `1 << 0` | XZ 压缩（文件中存在 XZ 压缩对象） |
| `1 << 1` | LZ4 压缩 |
| `1 << 2` | Keyed Hash（使用 SipHash 而非 Jenkins） |
| `1 << 3` | ZSTD 压缩 |

#### 兼容标志

| 标志位 | 含义 |
|--------|------|
| `1 << 0` | SEALED（文件使用 FSPRG 密封） |

### 1.3 Object 类型系统

#### ObjectHeader（16 字节）

```c
struct ObjectHeader {
    uint8_t  type;        // @0: 对象类型（8种）
    uint8_t  flags;       // @1: 压缩标志
    uint8_t  reserved[6]; // @2: 保留
    le64_t   size;        // @8: 对象总大小（含头部）
    uint8_t  payload[];   // @16: 载荷（变长）
};
```

#### 7 种对象类型

| 类型 | 枚举值 | 用途 |
|------|--------|------|
| `OBJECT_DATA` | 1 | 数据对象（键值对内容） |
| `OBJECT_FIELD` | 2 | 字段对象（字段名，如 "MESSAGE"） |
| `OBJECT_ENTRY` | 3 | 条目对象（一条日志记录） |
| `OBJECT_DATA_HASH_TABLE` | 4 | 数据哈希表 |
| `OBJECT_FIELD_HASH_TABLE` | 5 | 字段哈希表 |
| `OBJECT_ENTRY_ARRAY` | 6 | 条目数组（索引链） |
| `OBJECT_TAG` | 7 | 密封标签（HMAC-SHA256） |

### 1.4 DataObject — 数据对象

存储单个键值对的完整内容（如 `MESSAGE=Hello World`）。

```
┌─────────────────────────────────────────────┐
│ ObjectHeader (16 字节)                      │
│   type = OBJECT_DATA                        │
│   flags = 压缩标志                          │
│   size = 总大小                             │
├─────────────────────────────────────────────┤
│ hash              (le64)  @16               │ ← 内容哈希（SipHash/Jenkins）
│ next_hash_offset  (le64)  @24               │ ← 哈希链下一个
│ next_field_offset (le64)  @32               │ ← 同字段的下一个 Data
│ entry_offset      (le64)  @40               │ ← 第一个引用此数据的 Entry
│ entry_array_offset(le64)  @48               │ ← EntryArray 链（更多引用）
│ n_entries         (le64)  @56               │ ← 引用此数据的 Entry 总数
├─────────────────────────────────────────────┤
│ payload[]                 @64               │ ← 原始数据或压缩数据
└─────────────────────────────────────────────┘
```

**去重机制**：写入前先计算 payload 哈希，在数据哈希表中查找。如果找到相同哈希
且内容完全匹配的 DataObject，直接复用其偏移量，不再重复存储。

**每数据条目索引**：每个 DataObject 维护自己的 Entry 引用列表
（`entry_offset` + `entry_array_offset` 链），支持快速查找包含特定数据的所有条目。

### 1.5 FieldObject — 字段对象

存储字段名（不含值），如 `MESSAGE`、`_PID`、`PRIORITY`。

```
┌─────────────────────────────────────────────┐
│ ObjectHeader (16 字节)                      │
│   type = OBJECT_FIELD                       │
├─────────────────────────────────────────────┤
│ hash              (le64)  @16               │ ← 字段名哈希
│ next_hash_offset  (le64)  @24               │ ← 字段哈希链下一个
│ head_data_offset  (le64)  @32               │ ← 此字段的第一个 Data 对象
├─────────────────────────────────────────────┤
│ payload[]                 @40               │ ← 字段名字节
└─────────────────────────────────────────────┘
```

Field → Data 链：通过 `head_data_offset` → DataObject.`next_field_offset`
链表串联同一字段的所有不同值。

### 1.6 EntryObject — 条目对象

一条完整的日志记录，引用多个 DataObject。

```
┌─────────────────────────────────────────────┐
│ ObjectHeader (16 字节)                      │
│   type = OBJECT_ENTRY                       │
├─────────────────────────────────────────────┤
│ seqnum    (le64)  @16                       │ ← 全局序号
│ realtime  (le64)  @24                       │ ← 实时时钟（μs）
│ monotonic (le64)  @32                       │ ← 单调时钟（μs）
│ boot_id   (128位) @40                       │ ← 写入时的 boot ID
│ xor_hash  (le64)  @56                       │ ← 所有数据哈希的异或
├─────────────────────────────────────────────┤
│ items[0]: { object_offset, hash }  @64      │ ← DataObject 引用 0
│ items[1]: { object_offset, hash }  @80      │ ← DataObject 引用 1
│ items[N]: ...                               │
└─────────────────────────────────────────────┘

EntryItem = { le64 object_offset, le64 hash }  // 16 字节
```

**items 排序**：按 `object_offset` 升序排列并去重（写入时排序）。
**xor_hash**：所有引用的 Data 的 hash 异或值，用于快速条目匹配。

### 1.7 HashTableObject — 哈希表

文件中有两个哈希表：数据哈希表和字段哈希表。

```
┌─────────────────────────────────────────────┐
│ ObjectHeader (16 字节)                      │
├─────────────────────────────────────────────┤
│ items[0]: { head_hash_offset,               │
│             tail_hash_offset }     @16      │ ← 桶 0
│ items[1]: ...                      @32      │ ← 桶 1
│ items[N]: ...                               │
└─────────────────────────────────────────────┘

HashItem = { le64 head_hash_offset, le64 tail_hash_offset }  // 16 字节
```

**查找流程**：
```
bucket = hash % (table_size / sizeof(HashItem))
p = items[bucket].head_hash_offset
while p != 0:
    obj = read_object(p)
    if obj.hash == target_hash && compare_payload(obj, target):
        return obj  // 找到
    p = obj.next_hash_offset
return NOT_FOUND
```

**尾指针优化**：`tail_hash_offset` 使得插入操作为 O(1)（追加到链尾）。

### 1.8 EntryArrayObject — 条目数组

存储有序的 Entry 偏移量数组，支持链式扩展。

```
┌─────────────────────────────────────────────┐
│ ObjectHeader (16 字节)                      │
├─────────────────────────────────────────────┤
│ next_entry_array_offset (le64)  @16         │ ← 下一个 EntryArray
├─────────────────────────────────────────────┤
│ items[0]  (le64)                @24         │ ← Entry 偏移量 0
│ items[1]  (le64)                @32         │ ← Entry 偏移量 1
│ items[N]  ...                               │
└─────────────────────────────────────────────┘
```

**链式结构**：当一个 EntryArray 装满时，创建新的 EntryArrayObject 并通过
`next_entry_array_offset` 链接。

**两类 EntryArray 链**：
1. **全局链**：Header.`entry_array_offset` 指向，包含文件中所有 Entry
2. **每数据链**：DataObject.`entry_array_offset` 指向，包含引用该数据的所有 Entry

### 1.9 TagObject — 密封标签

```
┌─────────────────────────────────────────────┐
│ ObjectHeader (16 字节)                      │
├─────────────────────────────────────────────┤
│ seqnum  (le64)   @16                        │ ← Tag 序号
│ epoch   (le64)   @24                        │ ← FSPRG epoch
│ tag[32]          @32                        │ ← HMAC-SHA256 签名
└─────────────────────────────────────────────┘
```

Tag 对象用于前向安全密封（Forward Secure Sealing）。FSPRG 密钥在每个 epoch
后不可逆地演进，使得攻击者即使获得当前密钥也无法伪造过去的日志。

### 1.10 对象关系图

```
                    ┌─────────────────────┐
                    │ Header              │
                    │ entry_array_offset ─┼──────────────────┐
                    │ data_hash_table ───┼──┐                │
                    │ field_hash_table ──┼┐ │                │
                    └────────────────────┘│ │                │
                                          │ │                ▼
  ┌───────────────┐   ┌──────────────────┘ │    ┌────────────────────┐
  │FieldHashTable │   │                    │    │ EntryArray (全局)   │
  │  bucket[0] ───┼───┤                    │    │ items[]: Entry偏移  │
  │  bucket[1]    │   │                    │    │ next → ...          │
  │  ...          │   │                    │    └────────┬───────────┘
  └───────────────┘   │                    │             │
         │            │                    │             ▼
         ▼            │    ┌───────────────┘    ┌──────────────┐
  ┌──────────────┐    │    │                    │ EntryObject   │
  │ FieldObject  │    │    ▼                    │ seqnum, time  │
  │ "MESSAGE"    │    │  ┌──────────────────┐   │ items[]:      │
  │ head_data ───┼────┼→ │ DataHashTable    │   │  ┌─DataObj偏移│
  └──────────────┘    │  │  bucket[N] ──────┼─┐ │  └─DataObj偏移│
                      │  └──────────────────┘ │ └──────────────┘
                      │                       │        ↑
                      │                       ▼        │
                      │              ┌──────────────┐  │
                      │              │ DataObject   │  │
                      │              │ "MESSAGE=Hi" │  │
                      │              │ hash         │  │
                      │              │ entry_offset─┼──┘
                      │              │ entry_array──┼→ EntryArray(每数据)
                      │              │ next_hash ───┼→ 链中下一个 Data
                      └──────────────│ next_field ──┼→ 同字段下一个 Data
                                     └──────────────┘
```

---

## 二、JournalFile 结构体

### 2.1 内存表示（journal-file.h:59-127）

```c
typedef struct JournalFile {
    // === 文件描述符 ===
    int fd;                          // 文件描述符
    MMapFileDescriptor *cache_fd;    // mmap 缓存关联
    mode_t mode;                     // 文件权限
    int flags;                       // 打开标志

    // === 功能标志 ===
    bool writable:1;                 // 可写
    bool compress_xz:1;             // 启用 XZ 压缩
    bool compress_lz4:1;            // 启用 LZ4 压缩
    bool compress_zstd:1;           // 启用 ZSTD 压缩
    bool seal:1;                    // 启用密封
    bool close_fd:1;                // 关闭时关闭 fd
    bool archive:1;                 // 归档文件
    bool keyed_hash:1;              // 使用 SipHash

    // === 遍历状态 ===
    direction_t last_direction;      // 最后遍历方向
    LocationType location_type;      // 定位类型
    uint64_t last_n_entries;         // 缓存的条目数

    // === 文件信息 ===
    char *path;                      // 文件路径
    struct stat last_stat;           // 缓存的 stat
    usec_t last_stat_usec;           // 上次 stat 时间

    // === 映射指针 ===
    Header *header;                  // mmap 映射的文件头
    HashItem *data_hash_table;       // mmap 映射的数据哈希表
    HashItem *field_hash_table;      // mmap 映射的字段哈希表

    // === 当前游标 ===
    uint64_t current_offset;         // 当前条目偏移
    uint64_t current_seqnum;         // 当前序号
    uint64_t current_realtime;       // 当前实时时间
    uint64_t current_monotonic;      // 当前单调时间
    sd_id128_t current_boot_id;      // 当前 boot ID
    uint64_t current_xor_hash;       // 当前 xor 哈希

    // === 存储配额 ===
    JournalMetrics metrics;          // 文件大小限制

    // === 变更通知 ===
    sd_event_source *post_change_timer;  // 延迟 ftruncate 定时器
    usec_t post_change_timer_period;

    // === 链缓存 ===
    OrderedHashmap *chain_cache;     // EntryArray 链遍历缓存

    // === 离线归档 ===
    pthread_t offline_thread;        // 离线归档线程
    volatile OfflineState offline_state;

    // === 压缩 ===
    uint64_t compress_threshold_bytes; // 压缩阈值
    void *compress_buffer;             // 压缩缓冲区

    // === 密封（FSPRG + HMAC）===
    gcry_md_hd_t hmac;              // HMAC 上下文
    bool hmac_running;
    FSSHeader *fss_file;            // FSS 密钥文件映射
    size_t fss_file_size;
    uint64_t fss_start_usec;        // FSS 起始时间
    uint64_t fss_interval_usec;     // FSS epoch 间隔
    void *fsprg_state;              // FSPRG 当前状态
    size_t fsprg_state_size;
    void *fsprg_seed;               // FSPRG 种子
    size_t fsprg_seed_size;
} JournalFile;
```

### 2.2 文件打开流程（journal-file.c:3314-3574）

```
journal_file_open(fd, path, flags, mode, compress, seal, metrics, ...)
 ├── 1. 校验参数，分配 JournalFile 结构
 ├── 2. 设置压缩/keyed_hash/seal 标志
 ├── 3. 打开文件（如需），关联到 MMapCache
 ├── 4. fstat() 获取文件信息
 ├── 5. 如果文件为空且可写：
 │    ├── 设置创建时间（crtime xattr）
 │    ├── 加载 FSS 密钥（如密封）
 │    ├── 初始化 Header（写签名、标志、ID、初始偏移）
 │    └── fstat() 刷新
 ├── 6. mmap Header 区域（CONTEXT_HEADER）
 ├── 7. 校验 Header：
 │    ├── 检查签名 "LPKSHHRH"
 │    ├── 检查 compatible/incompatible 标志
 │    ├── 检查 state（ONLINE = 不干净关闭）
 │    ├── 检查大小/对齐/机器 ID
 │    └── 检查时间戳合理性
 ├── 8. 加载 FSS 密钥（已有文件）
 ├── 9. 刷新 Header（写 machine_id、boot_id、state=ONLINE）
 ├── 10. 设置 HMAC
 ├── 11. 如果新文件：
 │    ├── 创建 FieldHashTable 对象
 │    ├── 创建 DataHashTable 对象
 │    └── 追加第一个 Tag（如密封）
 └── 12. 安装 post_change_timer → 返回
```

### 2.3 mmap 窗口缓存（mmap-cache.c）

| 参数 | 值 | 说明 |
|------|-----|------|
| 窗口大小 | 8 MiB | 默认 mmap 区域大小 |
| Context 数 | 9 | 每种 Object 类型 + Header + 额外 |
| SIGBUS 处理 | 匿名页替换 | 防止文件截断时崩溃 |

```
mmap_cache_fd_get(cache_fd, context, keep_always, offset, size, &ret)
 ├── 1. 检查当前 context 是否命中（偏移在窗口内）→ 直接返回
 ├── 2. 搜索同 fd 的其他窗口覆盖此范围 → 复用
 ├── 3. 创建新窗口：
 │    ├── 计算 mmap 偏移（页对齐）和大小（至少 8MiB）
 │    ├── mmap(MAP_SHARED)
 │    └── 关联到 context
 └── 4. 返回映射后的指针

SIGBUS 处理：
 ├── 安装 sigbus_handler（进程级）
 ├── 当收到 SIGBUS 时：
 │    ├── 找到故障地址所在窗口
 │    ├── mmap(MAP_ANONYMOUS) 覆盖该窗口
 │    └── 标记 fd 为无效
 └── 后续访问返回零页（优雅降级而非崩溃）
```

### 2.4 链缓存（Chain Cache）

遍历 EntryArray 链时，`chain_cache` 缓存最后访问的位置：

```c
typedef struct ChainCacheItem {
    uint64_t first;     // 链起始偏移
    unsigned array;     // 当前在第几个 EntryArray
    uint64_t begin;     // 当前数组的起始 Entry 索引
    uint64_t total;     // 链中已知的 Entry 总数
    uint64_t last_index;// 上次访问的索引
} ChainCacheItem;
```

**效果**：顺序读取时，避免每次从链头遍历，直接跳到缓存位置附近。

---

## 三、写入路径

### 3.1 追加条目（journal-file.c:1856-2097）

```
journal_file_append_entry(f, ts, boot_id, iovec, n_iovec, seqnum, ret_object, ret_offset)
 │
 ├── 1. 校验时间戳（拒绝未来 > 1年 或过去的时间）
 │
 ├── 2. 可选：追加密封 Tag
 │    └── journal_file_maybe_append_tag(f, ts->realtime)
 │
 ├── 3. 逐个追加 Data 对象：
 │    └── for each iovec[i]:
 │         journal_file_append_data(f, data, size, &items[i])
 │         ├── 计算内容哈希（SipHash 或 Jenkins）
 │         ├── 在数据哈希表中查找：
 │         │    └── journal_file_find_data_object_with_hash()
 │         │         → 找到 → 直接复用（去重）
 │         │         → 未找到 → 继续创建
 │         ├── journal_file_append_object(OBJECT_DATA, size)
 │         │    └── 在文件末尾分配空间（fallocate + 扩展 arena_size）
 │         ├── 可选压缩：
 │         │    ├── 尝试 ZSTD/LZ4/XZ 压缩
 │         │    ├── 如果压缩后更小 → 使用压缩版本，缩小对象
 │         │    └── 设置 flags 中的压缩位
 │         ├── 写入 payload
 │         ├── journal_file_link_data()
 │         │    ├── 计算桶号 = hash % (table_size / sizeof(HashItem))
 │         │    ├── 设置 bucket 的 tail → 新对象
 │         │    ├── 如果桶非空：前一个尾节点.next_hash = 新对象
 │         │    ├── 如果桶为空：bucket.head = 新对象
 │         │    └── 更新 n_data、hash_chain_depth
 │         ├── HMAC 新对象
 │         └── 查找/创建 FieldObject，链接 Data→Field
 │
 ├── 4. 计算 xor_hash = 所有 items[i].hash 的异或
 │
 ├── 5. 按 object_offset 排序 items[] 并去重
 │
 ├── 6. 追加 EntryObject：
 │    ├── journal_file_append_object(OBJECT_ENTRY, 固定头 + items 数组)
 │    ├── 写入 seqnum、realtime、monotonic、boot_id、xor_hash、items[]
 │    └── HMAC 新对象
 │
 ├── 7. 链接 Entry：
 │    └── journal_file_link_entry(f, entry, items, n_items)
 │         ├── 链入全局 EntryArray：
 │         │    └── link_entry_into_array(header->entry_array_offset, ...)
 │         │         ├── 找到链尾 EntryArray
 │         │         ├── 如果有空间 → 直接追加
 │         │         └── 如果满了 → 创建新 EntryArray → 链接
 │         ├── 更新 Header 时间/序号字段
 │         └── 对每个 items[i] 的 DataObject：
 │              └── 链入该 Data 的每数据 EntryArray
 │                   ├── entry_offset（内联第一个）
 │                   └── entry_array_offset 链（更多）
 │
 └── 8. 触发 post_change_timer → 延迟 ftruncate/inotify 通知
```

### 3.2 数据去重流程

```
写入 "MESSAGE=Hello World":
  1. hash = siphash("MESSAGE=Hello World")
  2. bucket = hash % N_BUCKETS
  3. 遍历 bucket 链：
     for p = table[bucket].head; p; p = p->next_hash:
       if p.hash == hash:
         decompress(p.payload) if needed
         if memcmp(p.payload, "MESSAGE=Hello World") == 0:
           return p  // 去重成功！
  4. 未找到 → 新建 DataObject
```

**去重效果**：
- 相同优先级值（`PRIORITY=6`）只存一份
- 相同 systemd unit 名只存一份
- 相同 machine_id 只存一份
- 极大减少重复日志的存储开销

### 3.3 压缩策略

```
append_data() 中的压缩决策：
  if payload_size >= compress_threshold_bytes（默认 512 字节）:
    尝试压缩（优先级：ZSTD > LZ4 > XZ）
    if compressed_size < payload_size:
      使用压缩版本
      设置 object.flags |= OBJECT_COMPRESSED_*
      缩小 object.size
    else:
      保持原始数据
```

---

## 四、journald 守护进程（journald-server.c）

### 4.1 Server 结构体（journald-server.h:63-177）

```c
struct Server {
    char *namespace;                // 命名空间（可选）

    // === 输入源文件描述符 ===
    int syslog_fd;                  // /run/systemd/journal/dev-log
    int native_fd;                  // /run/systemd/journal/socket
    int stdout_fd;                  // /run/systemd/journal/stdout
    int dev_kmsg_fd;                // /dev/kmsg
    int audit_fd;                   // 审计套接字
    int hostname_fd;                // 主机名变更监视
    int notify_fd;                  // sd_notify 通知

    // === 事件循环 ===
    sd_event *event;
    sd_event_source *syslog_event_source;
    sd_event_source *native_event_source;
    sd_event_source *stdout_event_source;
    sd_event_source *dev_kmsg_event_source;
    sd_event_source *audit_event_source;
    sd_event_source *sync_event_source;       // 定期 fsync
    sd_event_source *sigusr1_event_source;    // 手动 flush
    sd_event_source *sigusr2_event_source;    // 手动 rotate
    sd_event_source *sigterm_event_source;
    sd_event_source *sigint_event_source;
    sd_event_source *sigrtmin1_event_source;  // 手动 sync
    sd_event_source *hostname_event_source;
    sd_event_source *notify_event_source;
    sd_event_source *watchdog_event_source;
    sd_event_source *idle_event_source;

    // === 日志文件 ===
    ManagedJournalFile *runtime_journal;      // /run/log/journal/ (tmpfs)
    ManagedJournalFile *system_journal;       // /var/log/journal/ (持久)
    OrderedHashmap *user_journals;            // 每用户日志（UID → file）

    uint64_t seqnum;                // 全局序号

    // === 限速 ===
    JournalRateLimit *ratelimit;    // 限速器
    usec_t sync_interval_usec;     // fsync 间隔
    usec_t ratelimit_interval;     // 限速窗口
    unsigned ratelimit_burst;      // 限速突发

    // === 存储配置 ===
    JournalStorage runtime_storage; // 运行时存储配额
    JournalStorage system_storage;  // 持久存储配额

    JournalCompressOptions compress; // 压缩配置
    bool seal;                       // 密封开关
    bool read_kmsg;                  // 读取 kmsg

    // === 转发配置 ===
    bool forward_to_kmsg;           // 转发到 kmsg
    bool forward_to_syslog;         // 转发到 syslog
    bool forward_to_console;        // 转发到控制台
    bool forward_to_wall;           // 转发到 wall

    // === 保留策略 ===
    usec_t max_retention_usec;      // 最长保留时间
    usec_t max_file_usec;           // 单文件最大使用时间
    usec_t oldest_file_usec;        // 最旧文件时间

    // === stdout 流 ===
    LIST_HEAD(StdoutStream, stdout_streams);
    unsigned n_stdout_streams;

    // === 级别限制 ===
    int max_level_store;            // 存储的最高级别
    int max_level_syslog;           // 转发 syslog 的最高级别
    int max_level_kmsg;             // 转发 kmsg 的最高级别
    int max_level_console;          // 转发控制台的最高级别
    int max_level_wall;             // 转发 wall 的最高级别

    Storage storage;                // VOLATILE/PERSISTENT/AUTO/NONE
    SplitMode split_mode;           // UID/NONE（用户分离模式）

    MMapCache *mmap;                // 共享 mmap 缓存

    // === 缓存的元数据字段 ===
    char machine_id_field[...];     // "_MACHINE_ID=..."
    char boot_id_field[...];        // "_BOOT_ID=..."
    char *hostname_field;           // "_HOSTNAME=..."
    char *cgroup_root;              // cgroup 根路径

    // === 客户端上下文缓存 ===
    Hashmap *client_contexts;       // PID → ClientContext
    Prioq *client_contexts_lru;     // LRU 淘汰队列
    ClientContext *my_context;      // journald 自身的上下文
    ClientContext *pid1_context;    // PID 1 的上下文

    VarlinkServer *varlink_server;  // Varlink 接口
};
```

### 4.2 五大输入源

#### 4.2.1 syslog 输入（journald-syslog.c）

```
套接字：/run/systemd/journal/dev-log（AF_UNIX DGRAM）
  → /dev/log 符号链接指向此处

server_process_syslog_message():
  ├── syslog_parse_priority() → 解析 <pri> 前缀
  ├── syslog_skip_date() → 跳过时间戳
  ├── syslog_parse_identifier() → 提取程序名和 PID
  ├── 可选转发：
  │    ├── forward_to_kmsg → 写 /dev/kmsg
  │    ├── forward_to_console → 写 /dev/console
  │    └── forward_to_wall → 广播终端消息
  └── server_dispatch_message() → 写入日志
```

#### 4.2.2 内核日志（journald-kmsg.c）

```
文件：/dev/kmsg（只读）

dev_kmsg_record():
  ├── 解析内核日志格式：pri,seqnum,timestamp,flags;key=value\n message
  ├── 跟踪序号间隙（丢失消息检测）
  ├── 提取元数据字段（_KERNEL_DEVICE, _KERNEL_SUBSYSTEM 等）
  ├── 导入 udev 设备信息
  └── server_dispatch_message() → 写入日志
```

#### 4.2.3 Native 协议（journald-native.c）

```
套接字：/run/systemd/journal/socket（AF_UNIX DGRAM）

协议格式：
  FIELD1=value1\n
  FIELD2=value2\n
  FIELD3\n               ← 二进制值前导
  <8字节 le64 长度>      ← 后跟原始字节
  <原始字节>\n

server_process_native_message():
  ├── 逐行解析字段
  ├── 特殊处理：PRIORITY=, SYSLOG_FACILITY=, MESSAGE=, OBJECT_PID=
  ├── 校验字段名（大写字母+数字+下划线）
  ├── 构建 iovec 数组
  └── server_dispatch_message() → 写入日志
```

**大消息支持**：通过 `SCM_RIGHTS` 传递 memfd 文件描述符，避免 DGRAM 大小限制。

#### 4.2.4 stdout 流（journald-stream.c）

```
套接字：/run/systemd/journal/stdout（AF_UNIX STREAM）

StdoutStream 协议（8 步握手）：
  1. IDENTIFIER    → 程序标识符（如 "myapp"）
  2. UNIT_ID       → systemd unit 名
  3. PRIORITY      → 默认优先级
  4. LEVEL_PREFIX  → 是否解析 <pri> 前缀
  5. FORWARD_TO_SYSLOG → 是否转发
  6. FORWARD_TO_KMSG   → 是否转发
  7. FORWARD_TO_CONSOLE → 是否转发
  8. RUNNING       → 开始传输日志行

状态机：IDENTIFIER → UNIT_ID → PRIORITY → LEVEL_PREFIX
       → FORWARD_TO_SYSLOG → FORWARD_TO_KMSG → FORWARD_TO_CONSOLE → RUNNING

stdout_stream_process():
  ├── 读取数据
  ├── 按 \n 或 NUL 分割
  ├── 检测 PID 变化（exec 后更新凭证）
  └── stdout_stream_log() → server_dispatch_message()
```

**systemd 集成**：`StandardOutput=journal` 时，systemd 自动建立 stdout stream 连接。

#### 4.2.5 audit 输入

```
套接字：AF_NETLINK NETLINK_AUDIT
  → 接收内核审计消息
  → 转换为 journal 格式存储
```

### 4.3 消息分发（journald-server.c:1080-1122）

```
server_dispatch_message(s, iovec, n, m, client_context, tv, priority, object_pid)
 │
 ├── 1. 级别过滤：priority > max_level_store → 丢弃
 │
 ├── 2. 存储模式检查：storage == NONE → 丢弃
 │
 ├── 3. 限速检查（按 unit 分组）：
 │    ├── journal_ratelimit_test(unit, interval, burst, priority, available)
 │    │    → 返回 0：被限速，丢弃
 │    │    → 返回 1：正常写入
 │    │    → 返回 >1：恢复后，先写一条 "Suppressed N messages" 驱动消息
 │    └── 限速粒度：per-unit + per-priority
 │
 └── 4. dispatch_message_real()
      ├── 添加可信元数据字段：
      │    _PID=, _UID=, _GID=, _COMM=, _EXE=, _CMDLINE=
      │    _SYSTEMD_CGROUP=, _SYSTEMD_UNIT=, _SYSTEMD_SLICE=
      │    _SELINUX_CONTEXT=, _BOOT_ID=, _MACHINE_ID=, _HOSTNAME=
      │    _TRANSPORT=（syslog/journal/stdout/kernel/audit）
      ├── 如果 split_mode == UID：
      │    └── 选择用户日志文件
      └── write_to_journal()
           ├── 如果 system_journal 存在 → 写入
           └── 否则写入 runtime_journal
```

### 4.4 存储架构

```
/run/log/journal/<machine-id>/     ← 运行时存储（tmpfs，重启丢失）
  system.journal                    ← 运行时系统日志
  system@<seqnum-id>-<tail-seqnum>.journal~  ← 轮转中
  system@<...>.journal              ← 轮转完成

/var/log/journal/<machine-id>/     ← 持久存储
  system.journal                    ← 当前系统日志
  system@<...>.journal              ← 已轮转归档
  user-<uid>.journal                ← 当前用户日志
  user-<uid>@<...>.journal          ← 已轮转用户日志
```

**Storage 模式**：

| 模式 | 行为 |
|------|------|
| `volatile` | 仅 /run/log/journal/（重启丢失） |
| `persistent` | /var/log/journal/（自动创建目录） |
| `auto` | /var/log/journal/ 如果已存在，否则 /run/ |
| `none` | 不存储 |

### 4.5 轮转与清理

#### 触发条件

| 条件 | 检查点 |
|------|--------|
| 文件大小超限 | 每次写入前检查 |
| 文件使用时间超限 | `max_file_usec`（默认 1 个月） |
| 存储空间不足 | `determine_space()` |
| 手动信号 | SIGUSR2 触发轮转 |

#### 清理策略（Vacuuming）

```
vacuum 流程：
  1. 按保留时间删除过期文件（max_retention_usec）
  2. 按空间限制删除最旧文件：
     ├── SystemMaxUse / RuntimeMaxUse    ← 总使用量上限
     ├── SystemKeepFree / RuntimeKeepFree ← 保留可用空间
     ├── SystemMaxFileSize               ← 单文件大小上限
     └── SystemMaxFiles                  ← 文件数量上限
  3. 删除顺序：最旧文件优先
```

### 4.6 信号处理

| 信号 | 行为 |
|------|------|
| SIGUSR1 | flush：将 /run 日志刷新到 /var |
| SIGUSR2 | rotate：轮转所有日志文件 |
| SIGRTMIN+1 | sync：强制 fsync 所有文件 |
| SIGTERM/SIGINT | 优雅退出 |

### 4.7 同步策略

```
sync_interval_usec（默认 5 分钟）：
  ├── 定时器触发 → fsync 所有打开的日志文件
  └── 确保崩溃时最多丢失 5 分钟数据

post_change_timer：
  ├── 写入后延迟 1 秒触发
  ├── ftruncate() → 更新文件大小（使 inotify 客户端感知变化）
  └── sd-journal 客户端收到通知 → 重新读取
```

---

## 五、检索与二分查找

### 5.1 全局 EntryArray 二分查找

```
generic_array_bisect(f, first_array_offset, n_entries, needle, ...)
 │
 ├── 使用 chain_cache 跳到缓存位置附近
 │
 ├── 遍历 EntryArray 链，找到 needle 所在的数组段
 │
 └── 在目标数组段内二分查找：
      ├── 比较函数根据查找类型不同：
      │    ├── by_seqnum → 比较 Entry.seqnum
      │    ├── by_realtime → 比较 Entry.realtime
      │    └── by_monotonic → 比较 Entry.monotonic
      └── 返回最接近的 Entry 偏移
```

### 5.2 每数据查找

```
"查找所有 PRIORITY=3 的条目"：
  1. 在数据哈希表中查找 DataObject("PRIORITY=3")
  2. 获取该 Data 的 entry_offset + entry_array_offset 链
  3. 在这条链上二分查找（按时间/序号）
  → 不需要全表扫描，O(log n) 复杂度
```

---

## 六、密封机制（Forward Secure Sealing）

### 6.1 原理

```
FSPRG (Forward Secure Pseudo Random Generator):
  ├── 初始密钥 K₀ → 生成 HMAC 密钥 HK₀
  ├── K₀ → 不可逆演进 → K₁ → 生成 HK₁
  ├── K₁ → 不可逆演进 → K₂ → 生成 HK₂
  └── ...
  
每个 epoch：
  ├── 用 HKₙ 计算 HMAC-SHA256（Header 稳定字段 + 所有新 Object 的稳定字段）
  └── 写入 TagObject { seqnum, epoch, tag[32] }

安全性：
  ├── 攻击者获得 Kₙ → 只能伪造 epoch ≥ n 的日志
  ├── 无法回推 Kₙ₋₁ → 无法伪造过去的日志
  └── 验证密钥（seed）应离线保存
```

### 6.2 HMAC 计算范围

```
journal_file_hmac_put_header():
  → 哈希 Header 中的稳定字段（排除 state、实时时间等可变字段）

journal_file_hmac_put_object():
  → 哈希 Object 的类型特定稳定部分（排除链接指针等可变字段）
```

---

## 七、验证算法（journal-verify.c）

```
journal_file_verify():
 │
 ├── 第一遍：顺序扫描所有对象
 │    ├── 验证 ObjectHeader 结构完整性
 │    ├── 验证哈希值正确性
 │    ├── 记录各类对象偏移到临时文件
 │    ├── 检查 Header 计数器一致性
 │    ├── 检查时间戳单调性
 │    ├── 如果密封：在 Tag 之间重算 HMAC 并比对
 │    └── 统计对象数量
 │
 └── 第二遍：验证索引结构
      ├── verify_entry_array()：
      │    ├── 遍历全局 EntryArray 链
      │    ├── 检查条目单调递增（无乱序）
      │    ├── 检查无循环引用
      │    └── 验证每个 Entry 的 items 指向有效 Data
      └── verify_data_hash_table()：
           ├── 遍历每个哈希桶链
           ├── 验证 head/tail 指针一致性
           ├── 验证哈希值落在正确桶中
           ├── 验证链内无循环
           └── 验证每个 Data 的 entry 列表有效
```

---

## 八、设计要点总结

### 8.1 存储效率

| 机制 | 效果 |
|------|------|
| **数据去重** | 相同内容只存一份（哈希查找） |
| **压缩** | 大载荷自动压缩（阈值 512B） |
| **二进制格式** | 无文本解析开销 |
| **mmap 零拷贝** | 避免 read/write 系统调用 |
| **窗口缓存** | 8MiB 窗口复用，减少 mmap 调用 |

### 8.2 检索性能

| 机制 | 效果 |
|------|------|
| **数据哈希表** | O(1) 平均查找特定键值对 |
| **每数据 EntryArray** | 快速过滤包含特定数据的条目 |
| **全局 EntryArray** | 有序遍历所有条目 |
| **二分查找** | O(log n) 按时间/序号定位 |
| **链缓存** | 顺序读优化，避免重复遍历 |
| **xor_hash** | 快速条目指纹比对 |

### 8.3 可靠性

| 机制 | 效果 |
|------|------|
| **追加写入** | 不修改已写数据，崩溃恢复友好 |
| **STATE_ONLINE** | 检测不干净关闭 |
| **FSPRG 密封** | 前向安全，防篡改 |
| **双遍验证** | 检测文件损坏和篡改 |
| **SIGBUS 处理** | 文件截断时优雅降级 |
| **定期 fsync** | 控制崩溃数据丢失窗口 |

### 8.4 与传统 syslog 对比

| 特性 | syslog | journald |
|------|--------|----------|
| 格式 | 纯文本 | 结构化二进制 |
| 检索 | grep O(n) | 哈希+二分 O(log n) |
| 去重 | 无 | 哈希去重 |
| 压缩 | 外部 logrotate | 内建实时压缩 |
| 元数据 | 仅 facility/priority | 完整进程/cgroup/SELinux 上下文 |
| 安全 | 无 | FSPRG 前向安全密封 |
| 轮转 | 外部 logrotate | 内建自动轮转+清理 |
| 多源 | 仅 syslog | syslog+kmsg+native+stdout+audit |
