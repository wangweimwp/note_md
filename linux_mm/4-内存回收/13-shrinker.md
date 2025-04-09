Linux内核shrinker机制的代码实现围绕内存回收策略展开，以下是关键代码层面的解析：
主要用于对slab的回收
以zswap机制中的应用为例，在```zswap_setup```中首先调用```__kmem_cache_create_args```创建了```zswap_entry_cache```的slab描述符，用于后续的zswap机制的slab申请，然后调用```zswap_alloc_shrinker```和```shrinker_register```注册了shrinker回调函数，用于计算回收量和执行回收操作，shrinker数据结构挂载在```shrinker_list```全局列表上。
当系统申请页面时：
```c
alloc_pages() 
    -> __alloc_pages_slowpath() 
        -> shrink_node() 
            -> shrink_slab() 
                -> do_shrink_slab()
```
会遍历shrinker_list上的所有shrinker回调函数进行内存回收

除此之外，最广泛的应用场景是文件系统缓存回收‌：如inode缓存和dentry缓存通过注册shrinker实现动态收缩‌
# 一、核心数据结构定义
```c
// include/linux/shrinker.h
struct shrinker {
    unsigned long (*count_objects)(struct shrinker *, struct shrink_control *sc);
    unsigned long (*scan_objects)(struct shrinker *, struct shrink_control *sc);
    int seeks;      /* 回收成本权重 */
    long batch;      // 批量处理粒度
    struct list_head list; // 注册链表
    // ... 其他字段（如flags等） 
};
```
count_objects‌: 计算当前可回收对象数量‌
scan_objects‌: 执行实际回收操作的回调函数‌
seeks‌: 影响回收优先级，低值优先执行‌
# 二、注册与注销流程
## 1. 注册shrinker（以动态注册为例）
```c
// 示例代码（参考‌:ml-citation{ref="1" data="citationList"}）
static struct shrinker my_shrinker = {
    .count_objects = my_count,
    .scan_objects = my_scan,
    .seeks = DEFAULT_SEEKS,
    .batch = 0, // 由内核自动计算
};

void init_module(void) {
    register_shrinker(&my_shrinker); // 核心注册函数‌:ml-citation{ref="8" data="citationList"}
}
```
register_shrinker()内部将shrinker加入全局链表shrinker_list‌
## 2. 注销接口
```c
void cleanup_module(void) {
    unregister_shrinker(&my_shrinker);
}
```
# 三、核心执行路径
## 1. 触发入口

内存回收通过shrink_slab()函数触发：

```c
// mm/vmscan.c
void shrink_slab(struct shrink_control *shrinkctl) {
    list_for_each_entry(shrinker, &shrinker_list, list) {
        unsigned long ret;
        // 计算待回收量
        ret = shrinker->count_objects(shrinker, shrinkctl);
        // 执行回收操作
        freed += shrinker->scan_objects(shrinker, shrinkctl);
    }
}
```
遍历所有注册的shrinker，按优先级执行回收‌
## 2. 典型回收逻辑实现（以slab为例）
```c
// mm/slub.c
static unsigned long slab_memcg_scan(struct shrinker *s, struct shrink_control *sc) {
    unsigned long freed = 0;
    // 遍历slab缓存
    list_for_each_entry(slab_cache, &slab_caches, list) {
        // 释放未使用对象
        freed += drain_freelist(slab_cache, sc->nr_to_scan);
        if (freed >= sc->nr_to_scan)
            break;
    }
    return freed;
}
```
# 四、关键机制实现
## 1. 批次处理（Batch Processing）
```c
// mm/vmscan.c
static unsigned long do_shrink_slab(struct shrink_control *shrinkctl,
                    struct shrinker *shrinker) {
    // 计算批次大小（动态调整）
    batch_size = shrinker->batch ? shrinker->batch : SHRINK_BATCH;
    // 分批扫描以避免锁竞争‌:ml-citation{ref="8" data="citationList"}
    for (total_scan = 0; total_scan < max_pass; total_scan += batch_size) {
        // 执行实际回收
        freed += shrinker->scan_objects(shrinker, shrinkctl);
    }
}
```
## 2. NUMA节点感知
```c
struct shrink_control {
    gfp_t gfp_mask;
    unsigned long nr_to_scan;
    int nid;        // NUMA节点ID‌:ml-citation{ref="8" data="citationList"}
    struct mem_cgroup *memcg;
};
```
回收操作按NUMA节点粒度执行，减少跨节点开销‌
# 五、性能优化点
1. 惰性初始化‌：部分shrinker（如文件系统缓存）采用延迟注册策略‌
2. 优先级排序‌：通过seeks字段动态调整执行顺序，低权重shrinker优先执行‌
3. 锁优化‌：采用rcu机制保护shrinker链表遍历‌
# 六、调试与监控
```/sys/kernel/debug/shrinker```‌：查看各shrinker的统计信息‌
Tracepoint‌：```trace_mm_shrink_slab_start/end```跟踪回收事件‌
附录：典型调用栈
```text
alloc_pages() -> __alloc_pages_slowpath() -> shrink_node() -> shrink_slab() -> do_shrink_slab() -> scan_objects()‌:ml-citation{ref="4,8" data="citationList"}
```


