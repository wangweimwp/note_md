**使能memcg后**

调用shrink_many，



pgdat->memcg_lru.seq 表示per-node的memcg的代（每个内存节点上的memcg代）struct lru_gen_folio通过lrugen->list节点将自身挂到pgdat->memcg_lru.folio上通过struct lru_gen_folio反推出struct lruvec，再反推出struct mem_cgroup每个memcg有来自不同内存节点的page，同一个内存节点的page挂载同一个lruvec上，在开启memcg的设备上，内存回收就是针对这个内存点上的memcg里属于这个内存节点上的lruvec

```c
mem_cgroup_css_online->lru_gen_online_memcg->hlist_nulls_add_tail_rcu //将memcg添加添加到pgdat->memcg_lru上
```

通过pgdat->memcg_lru.seq将memcg层分成2代（根据shrink_one返回值进行gen移动），而后将每个节点上属于该memcg的页面又通过struct lru_gen_folio分成4代

```c

/*
 * For each node, memcgs are divided into two generations: the old and the
 * young. For each generation, memcgs are randomly sharded into multiple bins
 * to improve scalability. For each bin, the hlist_nulls is virtually divided      		nulls（零位）
 * into three segments: the head, the tail and the default.			每个列表又分为3段，头，尾，默认
 *
 * An onlining memcg is added to the tail of a random bin in the old generation.		新上线的memcg被添加到old gen的尾部
 * The eviction starts at the head of a random bin in the old generation. The		驱逐从old gen的随机bin的头部开始
 * per-node memcg generation counter, whose reminder (mod MEMCG_NR_GENS) indexes
 * the old generation, is incremented when all its bins become empty.			memcg_lru.seq记录old gen，当old gen的所有bin清空后+1
 *
 * There are four operations:		lruvec->lrugen.seg，memcg的segments信息  	lruvec->lrugen.genmemcg的gen信息
 * 1. MEMCG_LRU_HEAD, which moves an memcg to the head of a random bin in its  
 *    current generation (old or young) and updates its "seg" to "head";
 * 2. MEMCG_LRU_TAIL, which moves an memcg to the tail of a random bin in its
 *    current generation (old or young) and updates its "seg" to "tail";
 * 3. MEMCG_LRU_OLD, which moves an memcg to the head of a random bin in the old
 *    generation, updates its "gen" to "old" and resets its "seg" to "default";
 * 4. MEMCG_LRU_YOUNG, which moves an memcg to the tail of a random bin in the
 *    young generation, updates its "gen" to "young" and resets its "seg" to
 *    "default".
 *
 * The events that trigger the above operations are:列表的移动规则
 * 1. Exceeding the soft limit, which triggers MEMCG_LRU_HEAD;		超过软限制触发MEMCG_LRU_HEAD
 * 2. The first attempt to reclaim an memcg below low, which triggers
 *    MEMCG_LRU_TAIL;
 * 3. The first attempt to reclaim an memcg below reclaimable size threshold,
 *    which triggers MEMCG_LRU_TAIL;
 * 4. The second attempt to reclaim an memcg below reclaimable size threshold,
 *    which triggers MEMCG_LRU_YOUNG;
 * 5. Attempting to reclaim an memcg below min, which triggers MEMCG_LRU_YOUNG;
 * 6. Finishing the aging on the eviction path, which triggers MEMCG_LRU_YOUNG;
 * 7. Offlining an memcg, which triggers MEMCG_LRU_OLD.
 *
 * Note that memcg LRU only applies to global reclaim, and the round-robin
 * incrementing of their max_seq counters ensures the eventual fairness to all
 * eligible memcgs. For memcg reclaim, it still relies on mem_cgroup_iter().
只保留全局老化，memcg局部老化接受各个memcg的max_seq增长速度不同，
balance_pgdat->kswapd_age_node->lru_gen_age_node->mem_cgroup_iter
 */

```



**没有使能memcg**

调用shrink_one

lrugen->seg 表示接下来的动作


