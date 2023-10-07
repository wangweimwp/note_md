```c
struct lruvec {
    struct list_head        lists[NR_LRU_LISTS];
    /* per lruvec lru_lock for memcg */
    spinlock_t            lru_lock;
    /*
     * These track the cost of reclaiming one LRU - file or anon -
     * over the other. As the observed cost of reclaiming one LRU
     * increases, the reclaim scan balance tips toward the other.
     */
    unsigned long            anon_cost;
    unsigned long            file_cost;
    /* Non-resident age, driven by LRU movement */
    atomic_long_t            nonresident_age;
    /* Refaults at the time of last reclaim cycle */
    unsigned long            refaults[ANON_AND_FILE];
    /* Various lruvec state flags (enum lruvec_flags */
    unsigned long            flags;
#ifdef CONFIG_LRU_GEN
    /* evictable pages divided into generations */
    struct lru_gen_struct        lrugen;
    /* to concurrently iterate lru_gen_mm_list */
    struct lru_gen_mm_state        mm_state;
#endif
#ifdef CONFIG_MEMCG
    struct pglist_data *pgdat;
#endif
};



/*
 * The youngest generation number is stored in max_seq for both anon and file
 * types as they are aged on an equal footing. The oldest generation numbers are
 * stored in min_seq[] separately for anon and file types as clean file pages
 * can be evicted regardless of swap constraints.
 *
 * Normally anon and file min_seq are in sync. But if swapping is constrained,
 * e.g., out of swap space, file min_seq is allowed to advance and leave anon
 * min_seq behind.
 *
 * The number of pages in each generation is eventually consistent and therefore
 * can be transiently negative when reset_batch_size() is pending.
 */
struct lru_gen_struct {
    /* the aging increments the youngest generation number 对最yuongest页面老化的次数*/
    unsigned long max_seq;
    /* the eviction increments the oldest generation numbers 对oldest generation逐出次数*/
    unsigned long min_seq[ANON_AND_FILE];
    /* the birth time of each generation in jiffies 没个gen产生的时间 （4个gen）*/
    unsigned long timestamps[MAX_NR_GENS];
    /* the multi-gen LRU lists, lazily sorted on eviction */
    struct list_head lists[MAX_NR_GENS][ANON_AND_FILE][MAX_NR_ZONES];
    /* the multi-gen LRU sizes, eventually consistent */
    long nr_pages[MAX_NR_GENS][ANON_AND_FILE][MAX_NR_ZONES];
    /* the exponential moving average of refaulted */
    unsigned long avg_refaulted[ANON_AND_FILE][MAX_NR_TIERS];
    /* the exponential moving average of evicted+protected */
    unsigned long avg_total[ANON_AND_FILE][MAX_NR_TIERS];
    /* the first tier doesn't need protection, hence the minus one */
    unsigned long protected[NR_HIST_GENS][ANON_AND_FILE][MAX_NR_TIERS - 1];
    /* can be modified without holding the LRU lock */
    atomic_long_t evicted[NR_HIST_GENS][ANON_AND_FILE][MAX_NR_TIERS];
    atomic_long_t refaulted[NR_HIST_GENS][ANON_AND_FILE][MAX_NR_TIERS];
    /* whether the multi-gen LRU is enabled */
    bool enabled;
};

/*在每次迭代结束之前，seq总是比max_seq小一，该结构体的主要作用是：                mem_cgroup_per_node
 *1，存放要迭代的mm_struct，                                            ->lruvec
 *2，记录迭代时的睡眠情况，提供等待队列供进程睡眠唤醒                                ->mm_state  记录lru页面的遍历状态
 *3，记录迭代时的状态并用bloom filters做一些优化                                ->memcg
 *4，该结构嵌入到struct lruvec结构体中，记录该结构体上的页面遍历状态                        ->mm_list   挂着memcg的所有mm_struct
*/
struct lru_gen_mm_state {
    /* set to max_seq after each iteration 每次迭代之后，将该值写入max_seq */
    unsigned long seq;
    /* where the current iteration continues (inclusive) */
    struct list_head *head;
    /* where the last iteration ended (exclusive) */
    struct list_head *tail;
    /* to wait for the last page table walker to finish */
    struct wait_queue_head wait;    //等待队列
    /* Bloom filters flip after each iteration */
    unsigned long *filters[NR_BLOOM_FILTERS];
    /* the mm stats for debugging */
    unsigned long stats[NR_HIST_GENS][NR_MM_STATS];
    /* the number of concurrent page table walkers */
    int nr_walkers;
};



//跑老化时候用的结构体，记录多进程遍历mm_struct时的动态信息
struct lru_gen_mm_walk {
    /* the lruvec under reclaim */
    struct lruvec *lruvec;
    /* unstable max_seq from lru_gen_struct */
    unsigned long max_seq;
    /* the next address within an mm to scan */
    unsigned long next_addr;
    /* to batch promoted pages 需要提升gen的页面数量 没有用锁，可能不是特别准*/
    int nr_pages[MAX_NR_GENS][ANON_AND_FILE][MAX_NR_ZONES];
    /* to batch the mm stats */
    int mm_stats[NR_MM_STATS];
    /* total batched items */
    int batched;
    bool can_swap;
    bool force_scan;
};

struct lru_gen_mm_list {
    /* mm_struct list for page table walkers */
    struct list_head fifo;
    /* protects the list above */
    spinlock_t lock;
};


/*tier的PID控制结构体
P = refaulted/(evicted+protected)
I根据P的历史值而定
D未使用
在进行页面驱逐时，该结构体用于计算选择哪个type的folio被驱逐
见get_type_to_scan
*/
struct ctrl_pos {
    unsigned long refaulted;
    unsigned long total;
    int gain;
};



/**
 * struct folio - Represents a contiguous set of bytes.
 * @flags: Identical to the page flags.
 * @lru: Least Recently Used list; tracks how recently this folio was used.
 * @mlock_count: Number of times this folio has been pinned by mlock().
 * @mapping: The file this page belongs to, or refers to the anon_vma for
 *    anonymous memory.
 * @index: Offset within the file, in units of pages.  For anonymous memory,
 *    this is the index from the beginning of the mmap.
 * @private: Filesystem per-folio data (see folio_attach_private()).
 *    Used for swp_entry_t if folio_test_swapcache().
 * @_mapcount: Do not access this member directly.  Use folio_mapcount() to
 *    find out how many times this folio is mapped by userspace.
 * @_refcount: Do not access this member directly.  Use folio_ref_count()
 *    to find how many references there are to this folio.
 * @memcg_data: Memory Control Group data.
 * @_flags_1: For large folios, additional page flags.
 * @_head_1: Points to the folio.  Do not use.
 * @_folio_dtor: Which destructor to use for this folio.
 * @_folio_order: Do not use directly, call folio_order().
 * @_compound_mapcount: Do not use directly, call folio_entire_mapcount().
 * @_subpages_mapcount: Do not use directly, call folio_mapcount().
 * @_pincount: Do not use directly, call folio_maybe_dma_pinned().
 * @_folio_nr_pages: Do not use directly, call folio_nr_pages().
 * @_flags_2: For alignment.  Do not use.
 * @_head_2: Points to the folio.  Do not use.
 * @_hugetlb_subpool: Do not use directly, use accessor in hugetlb.h.
 * @_hugetlb_cgroup: Do not use directly, use accessor in hugetlb_cgroup.h.
 * @_hugetlb_cgroup_rsvd: Do not use directly, use accessor in hugetlb_cgroup.h.
 * @_hugetlb_hwpoison: Do not use directly, call raw_hwp_list_head().
 *
 * A folio is a physically, virtually and logically contiguous set
 * of bytes.  It is a power-of-two in size, and it is aligned to that
 * same power-of-two.  It is at least as large as %PAGE_SIZE.  If it is
 * in the page cache, it is at a file offset which is a multiple of that
 * power-of-two.  It may be mapped into userspace at an address which is
 * at an arbitrary page offset, but its kernel virtual address is aligned
 * to its size.
 */
struct folio {
    /* private: don't document the anon union */
    union {
        struct {
    /* public: */
            unsigned long flags;
            union {
                struct list_head lru;
    /* private: avoid cluttering the output */
                struct {
                    void *__filler;
    /* public: */
                    unsigned int mlock_count;
    /* private: */
                };
    /* public: */
            };
            struct address_space *mapping;
            pgoff_t index;
            void *private;
            atomic_t _mapcount;
            atomic_t _refcount;
#ifdef CONFIG_MEMCG
            unsigned long memcg_data;
#endif
    /* private: the union with struct page is transitional */
        };
        struct page page;
    };
    union {
        struct {
            unsigned long _flags_1;
            unsigned long _head_1;
            unsigned char _folio_dtor;
            unsigned char _folio_order;
            atomic_t _compound_mapcount;
            atomic_t _subpages_mapcount;
            atomic_t _pincount;
#ifdef CONFIG_64BIT
            unsigned int _folio_nr_pages;
#endif
        };
        struct page __page_1;
    };
    union {
        struct {
            unsigned long _flags_2;
            unsigned long _head_2;
            void *_hugetlb_subpool;
            void *_hugetlb_cgroup;
            void *_hugetlb_cgroup_rsvd;
            void *_hugetlb_hwpoison;
        };
        struct page __page_2;
    };
};
```
