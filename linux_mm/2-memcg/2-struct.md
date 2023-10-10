```c
/*
 * The memory controller data structure. The memory controller controls both
 * page cache and RSS per cgroup. We would eventually like to provide
 * statistics based on the statistics developed by Rik Van Riel for clock-pro,
 * to help the administrator determine what knobs to tune.
 */
struct mem_cgroup {
    struct cgroup_subsys_state css;//所有资源控制器的基类

    /* Private memcg ID. Used to ID objects that outlive the cgroup */
    struct mem_cgroup_id id;

    /* Accounted resources 资源记账  _MEM类型的内存计数器: 记录内存的限制和当前使用量*/
    struct page_counter memory;        /* Both v1 & v2 */

    union {
        struct page_counter swap;    /* v2 only */
        struct page_counter memsw;    /* v1 only _MEMSWAP类型的内存计数器: 记录内存+交换分区的限制和当前使用量*/
    };

    /* Legacy consumer-oriented counters */
    struct page_counter kmem;        /* v1 only _KMEM类型的内核内存计数器: 记录内核内存的限制和当前使用量*/
    struct page_counter tcpmem;        /* v1 only _TCP类型的tcp缓冲区计数器: 记录tcp缓冲区的限制和当前使用量*/

    /* Range enforcement for interrupt charges */
    struct work_struct high_work;

#if defined(CONFIG_MEMCG_KMEM) && defined(CONFIG_ZSWAP)
    unsigned long zswap_max;
#endif

    unsigned long soft_limit;         // 内存使用软限制

    /* vmpressure notifications */
    struct vmpressure vmpressure;

    /*
     * Should the OOM killer kill all belonging tasks, had it kill one?
     */
    bool oom_group;

    /* protected by memcg_oom_lock */
    bool        oom_lock;
    int        under_oom;

    int    swappiness;
    /* OOM-Killer disable */
    int        oom_kill_disable;

    /* memory.events and memory.events.local */
    struct cgroup_file events_file;
    struct cgroup_file events_local_file;

    /* handle for "memory.swap.events" */
    struct cgroup_file swap_events_file;

    /* protect arrays of thresholds */
    struct mutex thresholds_lock;

    /* thresholds for memory usage. RCU-protected */
    struct mem_cgroup_thresholds thresholds;

    /* thresholds for mem+swap usage. RCU-protected */
    struct mem_cgroup_thresholds memsw_thresholds;

    /* For oom notifier event fd */
    struct list_head oom_notify;

    /*
     * Should we move charges of a task when a task is moved into this
     * mem_cgroup ? And what type of charges should we move ?
     */
    unsigned long move_charge_at_immigrate;
    /* taken only while moving_account > 0 */
    spinlock_t        move_lock;
    unsigned long        move_lock_flags;

    CACHELINE_PADDING(_pad1_);

    /* memory.stat */
    struct memcg_vmstats    *vmstats;

    /* memory.events */
    atomic_long_t        memory_events[MEMCG_NR_MEMORY_EVENTS];
    atomic_long_t        memory_events_local[MEMCG_NR_MEMORY_EVENTS];

    unsigned long        socket_pressure;

    /* Legacy tcp memory accounting */
    bool            tcpmem_active;
    int            tcpmem_pressure;

#ifdef CONFIG_MEMCG_KMEM
    int kmemcg_id;
    struct obj_cgroup __rcu *objcg;
    /* list of inherited objcgs, protected by objcg_lock */
    struct list_head objcg_list;
#endif

    CACHELINE_PADDING(_pad2_);

    /*
     * set > 0 if pages under this cgroup are moving to other cgroup.
     */
    atomic_t        moving_account;
    struct task_struct    *move_lock_task;

    struct memcg_vmstats_percpu __percpu *vmstats_percpu;//每cpu变量: 统计内存控制组状态(包括内存使用量和内存事件)？？？

#ifdef CONFIG_CGROUP_WRITEBACK
    struct list_head cgwb_list;
    struct wb_domain cgwb_domain;
    struct memcg_cgwb_frn cgwb_frn[MEMCG_CGWB_FRN_CNT];
#endif

    /* List of events which userspace want to receive */
    struct list_head event_list;
    spinlock_t event_list_lock;

#ifdef CONFIG_TRANSPARENT_HUGEPAGE
    struct deferred_split deferred_split_queue;
#endif

#ifdef CONFIG_LRU_GEN
    /* per-memcg mm_struct list */
    struct lru_gen_mm_list mm_list;
#endif

    struct mem_cgroup_per_node *nodeinfo[];//每个节点对应一个mem_cgroup_per_node实例
};


struct page_counter {
    /*
     * Make sure 'usage' does not share cacheline with any other field. The
     * memcg->memory.usage is a hot member of struct mem_cgroup.
     */
    atomic_long_t usage;
    CACHELINE_PADDING(_pad1_);

    /* effective memory.min and memory.min usage tracking */
    unsigned long emin;
    atomic_long_t min_usage;
    atomic_long_t children_min_usage;

    /* effective memory.low and memory.low usage tracking */
    unsigned long elow;
    atomic_long_t low_usage;
    atomic_long_t children_low_usage;

    unsigned long watermark;
    unsigned long failcnt;

    /* Keep all the read most fields in a separete cacheline. */
    CACHELINE_PADDING(_pad2_);

    unsigned long min;
    unsigned long low;
    unsigned long high;
    unsigned long max;
    struct page_counter *parent;
} ____cacheline_internodealigned_in_smp;

/*
 * per-node information in memory controller.
 */
struct mem_cgroup_per_node {
    // 内存控制组私有的lru链表
    // 当进程加入内存控制组后, 给进程分配的页面不再加入node的lru链表, 而是加入内存控制组私有的lru链表
    struct lruvec        lruvec;

    struct lruvec_stats_percpu __percpu    *lruvec_stats_percpu;
    struct lruvec_stats            lruvec_stats;

    unsigned long        lru_zone_size[MAX_NR_ZONES][NR_LRU_LISTS];

    struct mem_cgroup_reclaim_iter    iter;

    struct shrinker_info __rcu    *shrinker_info;

    struct rb_node        tree_node;    /* RB tree node */
    unsigned long        usage_in_excess;/* Set to the value by which */
                        /* the soft limit is exceeded*  内存使用量超过软限制的数值 = mem_cgroup.memory.count - mem_cgroup.soft_limit ？？*/
    bool            on_tree;
    struct mem_cgroup    *memcg;        /* Back pointer, we cannot */
                        /* use container_of       */
};


struct memcg_vmstats_percpu {
    /* Local (CPU and cgroup) page state & events */
    long            state[MEMCG_NR_STAT];
    unsigned long        events[NR_MEMCG_EVENTS];

    /* Delta calculation for lockless upward propagation */
    long            state_prev[MEMCG_NR_STAT];
    unsigned long        events_prev[NR_MEMCG_EVENTS];

    /* Cgroup1: threshold notifications & softlimit tree updates */
    unsigned long        nr_page_events;
    unsigned long        targets[MEM_CGROUP_NTARGETS];
};
```
