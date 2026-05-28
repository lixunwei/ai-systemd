# RequiresMountsFor= 实现原理深度分析

## 1. 功能说明

`RequiresMountsFor=` 指令确保某个路径及其所有父路径对应的挂载点在单元启动前已就绪。例如：

```ini
[Unit]
RequiresMountsFor=/var/lib/docker
```

这会自动为 `/var/lib/docker`、`/var/lib`、`/var`、`/` 这些路径中已知的 `.mount` 单元添加 `Requires=` + `After=` 依赖。

---

## 2. 核心数据结构

### 2.1 单元侧：路径列表

```c
// unit.h:147-149
struct Unit {
    /* 记录该单元依赖哪些挂载路径。Key=路径, Value=UnitDependencyInfo(来源标记) */
    Hashmap *requires_mounts_for;
};
```

### 2.2 Manager 侧：前缀索引表

```c
// Manager 中维护一个全局反向索引:
// 路径前缀 → 依赖该前缀的所有单元集合
Hashmap *units_requiring_mounts_for;  // key=prefix_path, value=Set<Unit*>
```

这是一个**前缀索引**：对于 `/var/lib/docker`，会在表中注册 `/var/lib/docker`、`/var/lib`、`/var`、`/` 这些键，每个键都指向包含该单元的 Set。

---

## 3. 注册流程

### 3.1 配置解析

```c
// load-fragment.c:3201
config_parse_unit_requires_mounts_for()
  → unit_require_mounts_for(u, path, UNIT_DEPENDENCY_FILE)
```

### 3.2 unit_require_mounts_for()

| 源文件 | `unit.c:4640-4711` |
|--------|-------------------|

```c
int unit_require_mounts_for(Unit *u, const char *path, UnitDependencyMask mask) {
    // 1. 验证路径是绝对路径
    if (!path_is_absolute(path)) return -EINVAL;

    // 2. 已注册则跳过
    if (hashmap_contains(u->requires_mounts_for, path)) return 0;

    // 3. 规范化路径
    path = path_simplify(p);

    // 4. 存入单元的 requires_mounts_for 哈希表
    hashmap_ensure_put(&u->requires_mounts_for, &path_hash_ops, p, di.data);

    // 5. 为路径的每个前缀，注册到 Manager 全局索引
    PATH_FOREACH_PREFIX_MORE(prefix, path) {
        Set *x = hashmap_get(manager->units_requiring_mounts_for, prefix);
        if (!x) {
            // 前缀不存在则创建新 Set
            x = set_new(NULL);
            hashmap_put(manager->units_requiring_mounts_for, strdup(prefix), x);
        }
        set_put(x, u);  // 将单元加入该前缀的集合
    }
}
```

### 3.3 PATH_FOREACH_PREFIX_MORE 宏

```c
// path-util.h:123-131
// 遍历路径自身及所有父前缀:
// "/var/lib/docker" → "/var/lib/docker", "/var/lib", "/var", "/"
#define PATH_FOREACH_PREFIX_MORE(prefix, path)
    for (char *_slash = strrchr(prefix, 0);  // 从末尾开始
         _slash && ((*_slash = 0), true);     // 截断到上一个 /
         _slash = strrchr(prefix, '/'))
```

---

## 4. 依赖生成（双向机制）

### 4.1 方向 A：单元加载时 → 查找已有 mount

```c
// unit.c:1504-1552
static int unit_add_mount_dependencies(Unit *u) {
    HASHMAP_FOREACH_KEY(di.data, path, u->requires_mounts_for) {
        PATH_FOREACH_PREFIX_MORE(prefix, path) {
            // 将路径前缀转换为 .mount 单元名
            // 如 "/var/lib" → "var-lib.mount"
            unit_name_from_path(prefix, ".mount", &p);

            m = manager_get_unit(manager, p);
            if (!m) {
                // mount 单元不存在→尝试加载（可能还未枚举到）
                manager_load_unit_prepare(manager, p, NULL, NULL, &m);
                continue;
            }

            // 添加 After= 依赖
            unit_add_dependency(u, UNIT_AFTER, m, true, di.origin_mask);

            // 若 mount 有配置文件（非自动生成），还添加 Requires=
            if (m->fragment_path)
                unit_add_dependency(u, UNIT_REQUIRES, m, true, di.origin_mask);
        }
    }
}
```

**调用时机**: `unit.c:1652`，在 `unit_add_dependencies()` 中（单元加载最后阶段）。

### 4.2 方向 B：mount 单元加载时 → 查找依赖它的单元

```c
// mount.c:325-344
// 当一个 .mount 单元加载时：
s = manager_get_units_requiring_mounts_for(manager, m->where);
SET_FOREACH(other, s) {
    // 为每个需要该挂载点的单元添加依赖
    unit_add_dependency(other, UNIT_AFTER, UNIT(m), true, UNIT_DEPENDENCY_PATH);

    if (UNIT(m)->fragment_path)
        unit_add_dependency(other, UNIT_REQUIRES, UNIT(m), true, UNIT_DEPENDENCY_PATH);
}
```

**关键**：这是反向方向——当新的 mount 单元出现时（如 fstab-generator 生成后枚举到），它主动查询全局前缀索引表，找出哪些单元需要它，并为它们建立依赖。

---

## 5. 路径到 Mount 单元名的转换

```c
// unit-name.c: unit_name_from_path()
// 规则: "/" → "-"，特殊字符转义
// 示例:
//   "/"           → "-.mount"
//   "/var"        → "var.mount"
//   "/var/lib"    → "var-lib.mount"
//   "/home/user"  → "home-user.mount"
```

---

## 6. Mount 检测机制

### 6.1 mount 单元的来源

| 来源 | 说明 |
|------|------|
| `/proc/self/mountinfo` | 内核挂载表，systemd 实时监控 |
| `fstab-generator` | 将 /etc/fstab 转换为 .mount 单元 |
| `.mount` 单元文件 | 管理员/包管理器手动定义 |

### 6.2 /proc/self/mountinfo 监控

```c
// mount.c 中通过 inotify 或 epoll 监控 /proc/self/mountinfo
// 当挂载表变化时:
// 1. 重新解析 mountinfo
// 2. 对新出现的挂载点创建/更新 .mount 单元
// 3. 对消失的挂载点标记 dead
```

### 6.3 mount 单元状态

| 状态 | 含义 |
|------|------|
| `mounted` | 已挂载（出现在 mountinfo 中） |
| `mounting` | 正在挂载 |
| `unmounting` | 正在卸载 |
| `dead` | 未挂载 |
| `failed` | 挂载失败 |

---

## 7. Requires vs After 的条件

```c
// 两处代码一致的逻辑:
if (m->fragment_path) {
    // mount 有配置文件(手动定义或 fstab 生成) → Requires + After
    unit_add_dependency(other, UNIT_REQUIRES, m, ...);
}
// 始终添加 After（无论是否有配置文件）
unit_add_dependency(other, UNIT_AFTER, m, ...);
```

| 场景 | After | Requires |
|------|-------|----------|
| mount 有 fragment_path（.mount 文件或 fstab 生成） | ✓ | ✓ |
| mount 仅从 mountinfo 发现（已手动挂载） | ✓ | ✗ |

**设计意图**：
- 有配置文件的 mount → systemd 有责任确保它被挂载 → 用 `Requires`
- 仅从 mountinfo 发现的 mount → 它已经挂载了，systemd 只需等待 → 只用 `After`

---

## 8. 完整流程示例

配置：
```ini
# myapp.service
[Unit]
RequiresMountsFor=/data/myapp/storage
```

假设系统中 `/data` 来自 fstab（有 .mount 文件），`/data/myapp/storage` 是 bind mount：

```
1. 解析 myapp.service
   → unit_require_mounts_for(u, "/data/myapp/storage")
   → 注册前缀索引:
       Manager.units_requiring_mounts_for["/data/myapp/storage"] += {myapp}
       Manager.units_requiring_mounts_for["/data/myapp"] += {myapp}
       Manager.units_requiring_mounts_for["/data"] += {myapp}
       Manager.units_requiring_mounts_for["/"] += {myapp}

2. 加载 myapp.service 的依赖阶段
   → unit_add_mount_dependencies(myapp)
   → 遍历 "/data/myapp/storage", "/data/myapp", "/data", "/"
   → 尝试查找 "data-myapp-storage.mount", "data-myapp.mount", "data.mount", "-.mount"
   → 找到 "data.mount"（有 fragment_path）
       → 添加 myapp After= data.mount
       → 添加 myapp Requires= data.mount
   → 找到 "-.mount"（根文件系统）
       → 添加 After=（通常 Requires 也有，取决于是否有 fragment）

3. 当 fstab-generator 生成 "data-myapp-storage.mount" 后
   → mount 加载时查询 units_requiring_mounts_for["/data/myapp/storage"]
   → 找到 {myapp}
   → 添加 myapp After= data-myapp-storage.mount
   → 添加 myapp Requires= data-myapp-storage.mount

结果: myapp.service 获得:
  After=data.mount data-myapp-storage.mount
  Requires=data.mount data-myapp-storage.mount
```

---

## 9. 设计亮点

1. **前缀索引表**：O(1) 查找哪些单元需要某个挂载点，避免遍历所有单元

2. **双向建立依赖**：无论是单元先加载还是 mount 先加载，依赖都能正确建立

3. **自动遍历父路径**：无需用户列出每个中间目录，`PATH_FOREACH_PREFIX_MORE` 自动展开

4. **Requires 有条件**：仅对 systemd 管理的 mount（有配置文件）添加 Requires，避免对已有手动挂载产生不必要的强依赖

5. **惰性加载**：若 mount 单元尚未枚举到，先触发 `manager_load_unit_prepare()`，等它加载后方向 B 会补充依赖
