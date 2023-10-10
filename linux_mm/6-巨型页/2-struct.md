```c
int hugetlb_max_hstate __read_mostly;        //巨型页池的数量
unsigned int default_hstate_idx;            //默认巨型页池的索引
struct hstate hstates[HUGE_MAX_HSTATE];        //巨型页池数组


__setup("hugepages=", hugepages_setup);        //解析cmdline


#define HSTATE_NAME_LEN 32
/* Defines one hugetlb page size */
struct hstate {
    struct mutex resize_lock;
    int next_nid_to_alloc;    //分配永久巨型页并添加到巨型页池中的时候，在允许的内存节点集合中轮流从每个内存节点分配永久巨型页，这个成员用来记录下次从哪个内存节点分配永久巨型页
    int next_nid_to_free;    //从巨型页池释放空闲巨型页的时候，在允许的内存节点集合中轮流从每个内存节点释放巨型页，这个成员用来记录下次从哪个内存节点释放巨型页
    unsigned int order;            //巨型页的长度，页的阶数
    unsigned int demote_order;
    unsigned long mask;            //巨型页页号的掩码，将虚拟地址和掩码按位与，得到巨型页页号
    unsigned long max_huge_pages;    //永久巨型页的最大数量
    unsigned long nr_huge_pages;    //巨型页的数量
    unsigned long free_huge_pages;    //空闲巨型页的数量
    unsigned long resv_huge_pages;    //预留巨型页的数量，它们已经承诺分配但还没有分配
    unsigned long surplus_huge_pages;        //临时巨型页的数量
    unsigned long nr_overcommit_huge_pages;    //临时巨型页的最大数量
    struct list_head hugepage_activelist;                //把已分配出去的巨型页链接起来
    struct list_head hugepage_freelists[MAX_NUMNODES];    //每个内存节点一个空闲巨型页链表
    unsigned int max_huge_pages_node[MAX_NUMNODES];
    unsigned int nr_huge_pages_node[MAX_NUMNODES];    //每个内存节点中巨型页的数量
    unsigned int free_huge_pages_node[MAX_NUMNODES];        //每个内存节点中空闲巨型页的数量
    unsigned int surplus_huge_pages_node[MAX_NUMNODES];    //每个内存节点中临时巨型页的数量
#ifdef CONFIG_CGROUP_HUGETLB
    /* cgroup control files */
    struct cftype cgroup_files_dfl[8];
    struct cftype cgroup_files_legacy[10];
#endif
    char name[HSTATE_NAME_LEN];    //巨型页池的名称，格式是“hugepages-kB”
};

struct huge_bootmem_page {
    struct list_head list;
    struct hstate *hstate;
};


enum scan_result {
    SCAN_FAIL,
    SCAN_SUCCEED,
    SCAN_PMD_NULL,
    SCAN_PMD_NONE,
    SCAN_PMD_MAPPED,
    SCAN_EXCEED_NONE_PTE,    //扫描0页或没有映射PTE的数量超过khugepaged_max_ptes_none
    SCAN_EXCEED_SWAP_PTE,    //累计扫描的页数（swapd分区的页）超出khugepaged_max_ptes_swap
    SCAN_EXCEED_SHARED_PTE,    //共享页面扫描数量超过khugepaged_max_ptes_shared
    SCAN_PTE_NON_PRESENT,    //PTE映射无效或不存在
    SCAN_PTE_UFFD_WP,    //与userfaultfd有关，扫描到userfaultfd页，  只有x86支持
    SCAN_PTE_MAPPED_HUGEPAGE,    //文件映射  是一个PTE-mapped THP
    SCAN_PAGE_RO,        //这个PMD中全是只读页面 这个PMD可以合并要求至少有一个页面可写
    SCAN_LACK_REFERENCED_PAGE,    //这个PMD中全是old页面，或者有swap页面但是young页面少于一半，这个PMD可以合并要求只要有一个young页面，若有swap页面时young页面大于PMD的一半
    SCAN_PAGE_NULL,    //无法获取struct page（无法获取的原因可能是pte映射的页是“特殊”的页面）
    SCAN_SCAN_ABORT,    //第一次扫描当前page所在的节点 并且 在这个PMD扫描过程中，出现过距离太远的page
    SCAN_PAGE_COUNT,    //page 的total_mapcount ！= refcount 表明这个page在进行其他操作
    SCAN_PAGE_LRU,
    SCAN_PAGE_LOCK,    //folio_trylock失败
    SCAN_PAGE_ANON,    //该地址可能会解除映射，然后在khugepage请求mmap_lock之后重新映射到file页面。对于符合条件的文件vma, Hugepage_vma_check可能返回true，再次检查时候发现是文件页面返回错误
    SCAN_PAGE_COMPOUND,    //文件映射，PTE映射的页面只是一个普通复合页，不是PTE-mapped THP
    SCAN_ANY_PROCESS,    //mm_struct->mm_users=0 没有用户再使用这个mm_struct了
    SCAN_VMA_NULL,        //没有找到地址对应的VMA
    SCAN_VMA_CHECK,    //这个VMA不支持巨型页
    SCAN_ADDRESS_RANGE,    //adress合并巨页以后超出了VMA的地址范围
    SCAN_DEL_PAGE_LRU,
    SCAN_ALLOC_HUGE_PAGE_FAIL,    //申请巨型页失败（申请连续PMD_ORDER个页面）
    SCAN_CGROUP_CHARGE_FAIL,        //申请到巨型页后进行memcg内存记账时候失败
    SCAN_TRUNCATED,
    SCAN_PAGE_HAS_PRIVATE,
};
```
