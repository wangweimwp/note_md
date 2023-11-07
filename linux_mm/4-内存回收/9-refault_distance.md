内核补丁`commit 34e58cac6d8f ("mm: workingset: let cache workingset challenge anon")`中提到`Reclaim can still decide how to balance pressure among the two LRUsdepending on the IO situation`，寻找如何平衡看补丁`mm: balance LRU lists based on relative thrashing`

在每次进行内存回收时候，会对每个LRU列表该扫描多收页面进行计算（`shrink_lruvec->get_scan_count`），计算原理是-如果页缓存(page cache)颠簸，则缓存页在内存中需要更多的时间，而匿名链表上可能有较冷的页，那么就要更多的扫描匿名页面。同样，如果换出的页出swapin，则表明我们回收匿名页过于激进，应该退回一点。

计算回收成本的数据结构时`strcut lruvec->cost_cost  和  strcut lruvec->file_cost`，

```c
//三个地方记录回收
shrink_inactive_list->lru_note_cost
shrink_active_list->lru_note_cost
workingset_refault->lru_note_cost_refault
```

**记录lru列表上的evictions和activations次数**

对于每个内存节点的LRU链表，维护一个计数器(`lruvec->nonresident_age`),当有page读入时，都有可能增加计数

```c
//页面被加入active列表
move_folios_to_lru
    ->if (folio_test_active(folio))
            workingset_age_nonresident();
//页面被eviction时
shrink_folio_list            
  ->__remove_mapping
        ->workingset_eviction
            ->workingset_age_nonresident();

workingset_activation
    ->workingset_age_nonresident

__read_swap_cache_async
    ->workingset_refault
        ->workingset_age_nonresident();

handle_pte_fault
    ->do_swap_page
        ->workingset_refault
            ->workingset_age_nonresident();
    ->swapin_readahead
        ->swap_vma_readahead
            ->__read_swap_cache_async
                ->workingset_refault
                    ->workingset_age_nonresident();

filemap_add_folio
    ->workingset_refault
        ->workingset_age_nonresident();    

folio_mark_accessed
    ->workingset_activation
        ->workingset_age_nonresident();
```

**页面被eviction时**

```c
static int __remove_mapping(struct address_space *mapping, struct folio *folio,
                bool reclaimed, struct mem_cgroup *target_memcg)
{
    ....

    if (folio_test_swapcache(folio)) {//匿名页面已经分配了swapcache
        swp_entry_t swap = folio_swap_entry(folio);

        if (reclaimed && !mapping_exiting(mapping))
            shadow = workingset_eviction(folio, target_memcg);//获取的页面eviction时的lruvec->nonresident_age值
        __delete_from_swap_cache(folio, swap, shadow);//记录到文件xarray树中
        mem_cgroup_swapout(folio, swap);
        xa_unlock_irq(&mapping->i_pages);
        put_swap_folio(folio, swap);
    } else {
        void (*free_folio)(struct folio *);

        free_folio = mapping->a_ops->free_folio;
        /*
         * Remember a shadow entry for reclaimed file cache in
         * order to detect refaults, thus thrashing, later on.
         *
         * But don't store shadows in an address space that is
         * already exiting.  This is not just an optimization,
         * inode reclaim needs to empty out the radix tree or
         * the nodes are lost.  Don't plant shadows behind its
         * back.
         *
         * We also don't store shadows for DAX mappings because the
         * only page cache folios found in these are zero pages
         * covering holes, and because we don't want to mix DAX
         * exceptional entries and shadow exceptional entries in the
         * same address_space.
         */
        if (reclaimed && folio_is_file_lru(folio) &&
            !mapping_exiting(mapping) && !dax_mapping(mapping))
            shadow = workingset_eviction(folio, target_memcg);
        __filemap_remove_folio(folio, shadow);//与上边类似，文件页面被eviction出去之后，将shadow记录到xarray中，下次系统又访问这个页面时用于计算refault distance
        xa_unlock_irq(&mapping->i_pages);
        if (mapping_shrinkable(mapping))
            inode_add_lru(mapping->host);
        spin_unlock(&mapping->host->i_lock);

        if (free_folio)
            free_folio(folio);
    }

    ....
}


void *workingset_eviction(struct folio *folio, struct mem_cgroup *target_memcg)
{
    struct pglist_data *pgdat = folio_pgdat(folio);
    unsigned long eviction;
    struct lruvec *lruvec;
    int memcgid;

    /* Folio is fully exclusive and pins folio's memory cgroup pointer */
    VM_BUG_ON_FOLIO(folio_test_lru(folio), folio);
    VM_BUG_ON_FOLIO(folio_ref_count(folio), folio);
    VM_BUG_ON_FOLIO(!folio_test_locked(folio), folio);

    if (lru_gen_enabled())
        return lru_gen_eviction(folio);

    lruvec = mem_cgroup_lruvec(target_memcg, pgdat);
    /* XXX: target_memcg can be NULL, go through lruvec */
    memcgid = mem_cgroup_id(lruvec_memcg(lruvec));
    eviction = atomic_long_read(&lruvec->nonresident_age);
    eviction >>= bucket_order;
    workingset_age_nonresident(lruvec, folio_nr_pages(folio));    //有页面发生eviction，记录
    return pack_shadow(memcgid, pgdat, eviction,
                folio_test_workingset(folio));    //计算folio的shadow值
}
```

**页面发生refault时**

对于匿名页面，发生缺页中断，若在swap cache中无法找到这个页面，则要从swap分区中读取，并记录refault

```c
vm_fault_t do_swap_page(struct vm_fault *vmf)
{
    ...
    folio = swap_cache_get_folio(entry, vma, vmf->address);//在swap中寻找该页
    if (folio)
        page = folio_file_page(folio, swp_offset(entry));
    swapcache = folio;

    if (!folio) {//如果swap cache中没有该页，那么说明已经回写到磁盘了
        ...
        shadow = get_shadow_from_swap_cache(entry);
        if (shadow)
            workingset_refault(folio, shadow);//有shadow值，书名发生了refault，计算refault distance
        ...
    }
    ...    
}
```

对于文件页面,当文件页面不在page cache中是，需要从磁盘读取，同时记录refault

```c
static struct folio *do_read_cache_folio(struct address_space *mapping,
        pgoff_t index, filler_t filler, struct file *file, gfp_t gfp)
{
    ...
    folio = filemap_get_folio(mapping, index);
    if (!folio) {
        folio = filemap_alloc_folio(gfp, 0);
        if (!folio)
            return ERR_PTR(-ENOMEM);
        err = filemap_add_folio(mapping, folio, index, gfp);
        if (unlikely(err)) {
            folio_put(folio);
            if (err == -EEXIST)
                goto repeat;
            /* Presumably ENOMEM for xarray node */
            return ERR_PTR(err);
        }
    ...
}

int filemap_add_folio(struct address_space *mapping, struct folio *folio,
                pgoff_t index, gfp_t gfp)
{
    void *shadow = NULL;
    int ret;

    __folio_set_locked(folio);
    ret = __filemap_add_folio(mapping, folio, index, gfp, &shadow);
    if (unlikely(ret))
        __folio_clear_locked(folio);
    else {
        /*
         * The folio might have been evicted from cache only
         * recently, in which case it should be activated like
         * any other repeatedly accessed folio.
         * The exception is folios getting rewritten; evicting other
         * data from the working set, only to cache data that will
         * get overwritten with something else, is a waste of memory.
         */
        WARN_ON_ONCE(folio_test_active(folio));
        if (!(gfp & __GFP_WRITE) && shadow)
            workingset_refault(folio, shadow);
        folio_add_lru(folio);
    }
    return ret;
}
```

**workingset_refault代码分析**

```c
/**
 * workingset_refault - Evaluate the refault of a previously evicted folio.
 * @folio: The freshly allocated replacement folio.
 * @shadow: Shadow entry of the evicted folio.
 *
 * Calculates and evaluates the refault distance of the previously
 * evicted folio in the context of the node and the memcg whose memory
 * pressure caused the eviction.
 */
void workingset_refault(struct folio *folio, void *shadow)
{
    bool file = folio_is_file_lru(folio);
    struct mem_cgroup *eviction_memcg;
    struct lruvec *eviction_lruvec;
    unsigned long refault_distance;
    unsigned long workingset_size;
    struct pglist_data *pgdat;
    struct mem_cgroup *memcg;
    unsigned long eviction;
    struct lruvec *lruvec;
    unsigned long refault;
    bool workingset;
    int memcgid;
    long nr;

    if (lru_gen_enabled()) {
        lru_gen_refault(folio, shadow);
        return;
    }

    unpack_shadow(shadow, &memcgid, &pgdat, &eviction, &workingset);
    eviction <<= bucket_order;

    rcu_read_lock();
    /*
     * Look up the memcg associated with the stored ID. It might
     * have been deleted since the folio's eviction.
     *
     * Note that in rare events the ID could have been recycled
     * for a new cgroup that refaults a shared folio. This is
     * impossible to tell from the available data. However, this
     * should be a rare and limited disturbance, and activations
     * are always speculative anyway. Ultimately, it's the aging
     * algorithm's job to shake out the minimum access frequency
     * for the active cache.
     *
     * XXX: On !CONFIG_MEMCG, this will always return NULL; it
     * would be better if the root_mem_cgroup existed in all
     * configurations instead.
     */
    eviction_memcg = mem_cgroup_from_id(memcgid);
    if (!mem_cgroup_disabled() && !eviction_memcg)
        goto out;
    eviction_lruvec = mem_cgroup_lruvec(eviction_memcg, pgdat);
    refault = atomic_long_read(&eviction_lruvec->nonresident_age);

    /*
     * Calculate the refault distance
     *
     * The unsigned subtraction here gives an accurate distance
     * across nonresident_age overflows in most cases. There is a
     * special case: usually, shadow entries have a short lifetime
     * and are either refaulted or reclaimed along with the inode
     * before they get too old.  But it is not impossible for the
     * nonresident_age to lap a shadow entry in the field, which
     * can then result in a false small refault distance, leading
     * to a false activation should this old entry actually
     * refault again.  However, earlier kernels used to deactivate
     * unconditionally with *every* reclaim invocation for the
     * longest time, so the occasional inappropriate activation
     * leading to pressure on the active list is not a problem.
     */
    refault_distance = (refault - eviction) & EVICTION_MASK;//计算refault_distance

    /*
     * The activation decision for this folio is made at the level
     * where the eviction occurred, as that is where the LRU order
     * during folio reclaim is being determined.
     *
     * However, the cgroup that will own the folio is the one that
     * is actually experiencing the refault event.
     * 计算refault distance 要考虑folio属于哪个memcg
     */
    nr = folio_nr_pages(folio);
    memcg = folio_memcg(folio);
    pgdat = folio_pgdat(folio);
    lruvec = mem_cgroup_lruvec(memcg, pgdat);

    mod_lruvec_state(lruvec, WORKINGSET_REFAULT_BASE + file, nr);

    mem_cgroup_flush_stats_delayed();
    /*
     * Compare the distance to the existing workingset size. We
     * don't activate pages that couldn't stay resident even if
     * all the memory was available to the workingset. Whether
     * workingset competition needs to consider anon or not depends
     * on having swap.
     * 是否考虑anon页面取决于是都拥有swap分区
     */
    workingset_size = lruvec_page_state(eviction_lruvec, NR_ACTIVE_FILE);
    if (!file) {
        workingset_size += lruvec_page_state(eviction_lruvec,
                             NR_INACTIVE_FILE);
    }
    if (mem_cgroup_get_nr_swap_pages(eviction_memcg) > 0) {
        workingset_size += lruvec_page_state(eviction_lruvec,
                             NR_ACTIVE_ANON);
        if (file) {
            workingset_size += lruvec_page_state(eviction_lruvec,
                             NR_INACTIVE_ANON);
        }
    }
    if (refault_distance > workingset_size)
        goto out;

    folio_set_active(folio);//refault_distance <= workingset_size,可以放到actice列表
    workingset_age_nonresident(lruvec, nr);
    mod_lruvec_state(lruvec, WORKINGSET_ACTIVATE_BASE + file, nr);

    /* Folio was active prior to eviction */
    if (workingset) {
        folio_set_workingset(folio);
        /*
         * XXX: Move to folio_add_lru() when it supports new vs
         * putback
         */
        lru_note_cost_refault(folio);
        mod_lruvec_state(lruvec, WORKINGSET_RESTORE_BASE + file, nr);
    }
out:
    rcu_read_unlock();
}
```

#### **PG_workingset flags含义和使用**

含义：Page is considered part of active userspace's workingset。如下两处代码有设置：

```c
shrink_active_list
    ->folio_set_workingset//当页面被demotion到inactive list时候设置


workingset_refault
    ->if (workingset) {
            folio_set_workingset(folio)
      }
```

内核需要区分是否是thrashing的情况：我们知道workingset算法同时保护anon和file page之后，这两个page生成时都是加入了inactive list当中，第一种情况：这种page直接被eviction之后，再次读取触发refault，这种情况仅仅称为transition；第二个种情况是说：如果内核判定refault distance加入了active list，之后老化被回收，再次access触发的refault就叫做thrashing。内核怎么区分两种情况呢？第二种情况明显是加入过active list，所以shrink_active_list内核就设置PageWorkingSet区分两种情况，nice!!!!

对PG_workingset的判定后，只要有2种操作

```c
//仅存在MGLRU中
evict_folios
    ->if (folio_test_workingset(folio))
                folio_set_referenced(folio);

//在文件系统中  记录memstall事件？？？
bool workingset = folio_test_workingset(folio);
    if (unlikely(workingset))
        psi_memstall_enter(&pflags)    //标记当前task由于内存不足而被阻塞

    ......

    if (unlikely(workingset))
        psi_memstall_leave(&pflags);
```
