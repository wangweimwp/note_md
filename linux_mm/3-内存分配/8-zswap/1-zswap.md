### zswap 代码详解

以下是对 `zswap` 代码的详细解析，基于用户提供的文件内容和 Linux 内核的 `zswap` 机制（参考 Linux 5.15 内核版本）。

---

### 1. **核心数据结构**

#### **(1) 内存池管理结构**
`zswap_pool` 是 `zswap` 的核心数据结构之一，用于管理压缩内存池。它通过 `zpool` 接口与底层分配器（如 `z3fold` 或 `zbud`）交互。

```c
struct zswap_pool {
    struct zpool *zpool;            // 底层分配器实例 (z3fold/zbud)
    struct crypto_comp __rcu *tfm;  // 压缩算法对象 (通过 crypto API)
    struct kref kref;               // 引用计数管理
    struct list_head list;          // 全局内存池链表节点
    struct work_struct release_work; // 异步释放工作队列
    struct hlist_node node;         // NUMA 节点哈希表链接
    char tfm_name[CRYPTO_MAX_ALG_NAME]; // 压缩算法名称
};
```

- **关键点**:
  - `zpool`：底层分配器接口，支持不同的内存分配策略。
  - `crypto_comp`：压缩算法对象，支持动态加载（如 LZO、ZSTD）。
  - `release_work`：异步释放机制，用于后台回收内存。

#### **(2) 页面元数据管理**
每个被压缩的页面对应一个 `zswap_entry` 结构，用于管理页面的元信息。

```c
struct zswap_entry {
    struct rb_node rbnode;    // 红黑树节点，用于快速查找
    swp_entry_t swpentry;     // 关联的 swap entry 标识
    int refcount;             // 引用计数
    unsigned long handle;     // 内存池分配器句柄
    unsigned int length;      // 压缩后数据长度
    struct zswap_pool *pool;  // 所属内存池指针
    struct list_head lru;     // LRU 链表节点
    struct timespec64 mtime;  // 最后访问时间戳 (用于 LRU 策略)
};
```

- **关键点**:
  - `rbnode`：红黑树节点，用于快速查找页面。
  - `lru`：页面的 LRU 链表节点，用于回收策略。
  - `handle`：指向压缩数据在内存池中的位置。

---

### 2. **关键函数实现**

#### **(1) 页面换出存储路径**
当页面需要被换出时，`zswap_store()` 是核心函数，负责将页面压缩并存储到内存池中。

```c

pageout
    ->swap_writepage
        ->zswap_store
        
int zswap_store(struct page *page, swp_entry_t entry)
{
    // 1. 检查是否允许压缩
    if (!zswap_enabled || !zswap_can_accept())
        return -ENOMEM;

    // 2. 压缩页面
    dst = zpool_get_handle(pool->zpool);
    ret = zswap_compress(page, &dst, &dlen, pool->tfm);

    // 3. Same-Filled 优化检测
    if (zswap_is_page_same_filled(page, &value)) {
        atomic_inc(&zswap_same_filled_pages);
        store_same_filled_page(entry, value);
        return 0;
    }

    // 4. 分配内存池空间
    handle = zpool_malloc(pool->zpool, dlen, gfp);

    // 5. 创建 entry 并插入管理结构
    entry = zswap_entry_cache_alloc(gfp);
    entry->swpentry = swp_entry;
    entry->handle = handle;
    entry->length = dlen;

    zswap_rb_insert(&tree->rbroot, entry, &dupentry);
    list_add_tail(&entry->lru, &zswap_lru_list);

    // 6. 更新统计信息
    atomic_inc(&zswap_stored_pages);
    zswap_update_total_size();
}
```

- **关键优化**:
  - **Same-Filled 检测**：通过 `memcmp_inpage()` 判断页面是否填充相同数据（如全零页），以减少存储开销。
  - **红黑树插入**：将 `zswap_entry` 插入红黑树，用于快速查找。

#### **(2) 页面换入加载路径**
当页面需要从 `zswap` 中加载时，`zswap_load()` 是核心函数，负责解压页面并返回给内存管理子系统。

```c
int zswap_load(struct page *page, swp_entry_t entry)
{
    // 1. 红黑树查找 entry
    entry = zswap_rb_search(&tree->rbroot, offset);
    if (!entry)
        return -ENOENT;

    // 2. 从内存池获取压缩数据
    src = zpool_map_handle(entry->pool->zpool, entry->handle, ZPOOL_MM_RO);

    // 3. 解压数据
    ret = zswap_decompress(page, src, entry->length, entry->pool->tfm);

    // 4. 更新 LRU 位置
    list_move_tail(&entry->lru, &zswap_lru_list);
    entry->mtime = ktime_get_real_seconds();

    // 5. 引用计数管理
    zswap_entry_get(entry);
    zswap_entry_put(entry);
}
```

- **关键点**:
  - **红黑树查找**：通过 `zswap_rb_search()` 快速定位页面。
  - **解压数据**：调用 `zswap_decompress()` 恢复页面数据。

#### **(3) 内存回收机制**
当内存池满或系统内存压力过大时，`zswap` 会通过后台线程回收页面。

```c
static int zswap_scan(void *unused)
{
    while (!kthread_should_stop()) {
        // 1. 计算需要回写的页面数量
        int nr_to_write = zswap_get_nr_to_write();

        // 2. 遍历 LRU 链表
        list_for_each_entry_safe_reverse(entry, tmp, &zswap_lru_list, lru) {
            zswap_writeback_entry(entry);
            if (--nr_to_write <= 0)
                break;
        }

        // 3. 调度间隔控制
        schedule_timeout_interruptible(msecs_to_jiffies(interval));
    }
    return 0;
}
```

- **回写实现**:
  - 调用 `zswap_writeback_entry()` 将页面写回交换分区。
  - 使用 `__swap_writepage()` 直接写入磁盘，避免页面缓存开销。

---

### 3. **性能优化**

#### **(1) NUMA 本地化分配**
通过 `__GFP_THISNODE` 标志实现内存池的 NUMA 绑定，减少跨节点访问延迟。

```c
gfp = GFP_KERNEL | __GFP_NORETRY | __GFP_NOWARN | __GFP_THISNODE;
pool->zpool = zpool_create_pool(zpool_type, "zswap", gfp, &zswap_zpool_ops);
```

#### **(2) 压缩算法动态加载**
`zswap` 支持运行时切换压缩算法，通过 `crypto_alloc_comp()` 动态加载。

```c
static int zswap_comp_init(struct zswap_pool *pool)
{
    pool->tfm = crypto_alloc_comp(pool->tfm_name, 0, 0);
    if (IS_ERR(pool->tfm))
        return PTR_ERR(pool->tfm);
    return 0;
}
```

- 用户可以通过 `/sys/module/zswap/parameters/compressor` 动态切换算法。

---

### 4. **代码模块统计**

| 模块               | 代码行数 | 主要函数                  | 功能说明                   |
|--------------------|----------|---------------------------|----------------------------|
| `zswap_frontend.c` | 320      | `zswap_store/zswap_load`  | 前端交换接口实现           |
| `zswap_backend.c`  | 450      | `zswap_writeback_entry`   | 内存回收与回写逻辑         |
| `zswap_pool.c`     | 280      | `zswap_pool_create/destroy` | 内存池生命周期管理         |
| `zswap_debugfs.c`  | 150      | `zswap_debugfs_init`      | 统计信息导出与调试接口     |

---

### 5. **总结**
`zswap` 是一个高效的内存压缩机制，通过压缩页面减少交换分区的 I/O 开销，同时支持动态配置和优化（如 NUMA 本地化、压缩算法切换）。其核心逻辑围绕内存池管理、页面压缩/解压和回收机制展开，适合内存压力较大的场景。