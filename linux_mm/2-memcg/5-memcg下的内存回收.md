```c
try_to_free_mem_cgroup_pages->do_try_to_free_pages->shrink_zones->shrink_node->lru_gen_shrink_node
```

```c
unsigned long try_to_free_pages(struct zonelist *zonelist, int order,
                gfp_t gfp_mask, nodemask_t *nodemask)
{
    unsigned long nr_reclaimed;
    struct scan_control sc = {
        .nr_to_reclaim = SWAP_CLUSTER_MAX,
        .gfp_mask = current_gfp_context(gfp_mask),
        .reclaim_idx = gfp_zone(gfp_mask),
        .order = order,
        .nodemask = nodemask,
        .priority = DEF_PRIORITY,
        .may_writepage = !laptop_mode,
        .may_unmap = 1,
        .may_swap = 1,
    };

    /*
     * scan_control uses s8 fields for order, priority, and reclaim_idx.
     * Confirm they are large enough for max values.
     */
    BUILD_BUG_ON(MAX_ORDER > S8_MAX);
    BUILD_BUG_ON(DEF_PRIORITY > S8_MAX);
    BUILD_BUG_ON(MAX_NR_ZONES > S8_MAX);

    /*
     * Do not enter reclaim if fatal signal was delivered while throttled.
     * 1 is returned so that the page allocator does not OOM kill at this
     * point.
    /*内存回收进程节流
        *throttle_direct_reclaim会对任务(进程)是否进行直接内存回收请求进行限制:
     *1.return False:
     *    触发条件：
     *        (a).当前进程是内核进程或当前进程收到了kill信号
     *        (b).当前进程对应的最优节点不平衡,然后kswapd进程被唤醒用于平衡最优节点,于此同时当前进程被
     *            加入到最优节点的pfmemalloc_wait等待队列等待节点达到平衡状态。当kswapd将节点调节到平    
     *            衡状态后，唤醒被加入到队列中的进程，进程唤醒后继续执行，并再次检查进程是否收到kill信
     *            号，若未收到kill信号返回False     
     *    产生的结果:内存分配任务继续执行直接内存回收操作

     *2.return True ：
     *    触发条件：当前进程对应的最优节点不平衡，同上kswapd被唤醒，进程被加入到等待对列。在进程被唤醒
     *             后，若检查到进程收到了kill信号，则返回True
     *    产生的结果:内存分配函数跳过直接内存内存回收操作
     */
     */
    if (throttle_direct_reclaim(sc.gfp_mask, zonelist, nodemask))
        return 1;

    set_task_reclaim_state(current, &sc.reclaim_state);
    trace_mm_vmscan_direct_reclaim_begin(order, sc.gfp_mask);

    nr_reclaimed = do_try_to_free_pages(zonelist, &sc);

    trace_mm_vmscan_direct_reclaim_end(nr_reclaimed);
    set_task_reclaim_state(current, NULL);

    return nr_reclaimed;
}


/*
 * This is the main entry point to direct page reclaim.
 *
 * If a full scan of the inactive list fails to free enough memory then we
 * are "out of memory" and something needs to be killed.
 *
 * If the caller is !__GFP_FS then the probability of a failure is reasonably
 * high - the zone may be full of dirty or under-writeback pages, which this
 * caller can't do much about.  We kick the writeback threads and take explicit
 * naps in the hope that some of these pages can be written.  But if the
 * allocating task holds filesystem locks which prevent writeout this might not
 * work, and the allocation attempt will fail.
 *
 * returns:    0, if no pages reclaimed
 *         else, the number of pages reclaimed
 */
static unsigned long do_try_to_free_pages(struct zonelist *zonelist,
                      struct scan_control *sc)
{
    int initial_priority = sc->priority;
    pg_data_t *last_pgdat;
    struct zoneref *z;
    struct zone *zone;
retry:
    delayacct_freepages_start();

    if (!cgroup_reclaim(sc))
        __count_zid_vm_events(ALLOCSTALL, sc->reclaim_idx, 1);

    do {
        //若不是有空户空间调用的主动回收（memory.reclaim），则计算内存压力
        //由于无法回收足够多的页，回收深度的不断加深，vmpressure_prio计算内存压力等级（3个等级）逐级发送内存压力事件
        if (!sc->proactive)
            vmpressure_prio(sc->gfp_mask, sc->target_mem_cgroup,
                    sc->priority);
        sc->nr_scanned = 0;
        shrink_zones(zonelist, sc);

        if (sc->nr_reclaimed >= sc->nr_to_reclaim)//回收的页数>=需要回收的页数
            break;

        if (sc->compaction_ready)//有一个zone已经压缩完成
            break;

        /*
         * If we're getting trouble reclaiming, start doing
         * writepage even in laptop mode.
         */
        if (sc->priority < DEF_PRIORITY - 2)
            sc->may_writepage = 1;//写入批量处理
    } while (--sc->priority >= 0);//加大回收力度

    last_pgdat = NULL;
    for_each_zone_zonelist_nodemask(zone, z, zonelist, sc->reclaim_idx,
                    sc->nodemask) {
        if (zone->zone_pgdat == last_pgdat)
            continue;
        last_pgdat = zone->zone_pgdat;

        snapshot_refaults(sc->target_mem_cgroup, zone->zone_pgdat);//和refault页面有关，后续分析？？？

        if (cgroup_reclaim(sc)) {
            struct lruvec *lruvec;

            lruvec = mem_cgroup_lruvec(sc->target_mem_cgroup,
                           zone->zone_pgdat);
            clear_bit(LRUVEC_CONGESTED, &lruvec->flags);//清楚lruvec的LRUVEC_CONGESTED位
        }
    }

    delayacct_freepages_end();

    if (sc->nr_reclaimed)
        return sc->nr_reclaimed;

    /* Aborted reclaim to try compaction? don't OOM, then */
    if (sc->compaction_ready)
        return 1;

    /*
     * We make inactive:active ratio decisions based on the node's
     * composition of memory, but a restrictive reclaim_idx or a
     * memory.low cgroup setting can exempt large amounts of
     * memory from reclaim. Neither of which are very common, so
     * instead of doing costly eligibility calculations of the
     * entire cgroup subtree up front, we assume the estimates are
     * good, and retry with forcible deactivation if that fails.
     */
    if (sc->skipped_deactivate) {
        sc->priority = initial_priority;
        sc->force_deactivate = 1;
        sc->skipped_deactivate = 0;
        goto retry;
    }

    /* Untapped cgroup reserves?  Don't OOM, retry. */
    if (sc->memcg_low_skipped) {
        sc->priority = initial_priority;
        sc->force_deactivate = 0;
        sc->memcg_low_reclaim = 1;
        sc->memcg_low_skipped = 0;
        goto retry;
    }

    return 0;
}

/*
 * This is the direct reclaim path, for page-allocating processes.  We only
 * try to reclaim pages from zones which will satisfy the caller's allocation
 * request.
 *
 * If a zone is deemed to be full of pinned pages then just give it a light
 * scan then give up on it.
内存申请过程中的直接内存回收， 
 */
static void shrink_zones(struct zonelist *zonelist, struct scan_control *sc)
{
    struct zoneref *z;
    struct zone *zone;
    unsigned long nr_soft_reclaimed;
    unsigned long nr_soft_scanned;
    gfp_t orig_mask;
    pg_data_t *last_pgdat = NULL;
    pg_data_t *first_pgdat = NULL;

    /*
     * If the number of buffer_heads in the machine exceeds the maximum
     * allowed level, force direct reclaim to scan the highmem zone as
     * highmem pages could be pinning lowmem pages storing buffer_heads
     */
    orig_mask = sc->gfp_mask;
    if (buffer_heads_over_limit) {//buffer_heads超过系统允许的最大值，则强制扫描highmem zone
        sc->gfp_mask |= __GFP_HIGHMEM;
        sc->reclaim_idx = gfp_zone(sc->gfp_mask);
    }

    for_each_zone_zonelist_nodemask(zone, z, zonelist,
                    sc->reclaim_idx, sc->nodemask) {
        /*
         * Take care memory controller reclaiming has small influence
         * to global LRU.
         */
        if (!cgroup_reclaim(sc)) {
            if (!cpuset_zone_allowed(zone,    //不允许在这个zone上申请页面，后续分析？？？
                         GFP_KERNEL | __GFP_HARDWALL))
                continue;

            /*
             * If we already have plenty of memory free for
             * compaction in this zone, don't free any more.
             * Even though compaction is invoked for any
             * non-zero order, only frequent costly order
             * reclamation is disruptive enough to become a
             * noticeable problem, like transparent huge
             * page allocations.
             */
            if (IS_ENABLED(CONFIG_COMPACTION) &&
                sc->order > PAGE_ALLOC_COSTLY_ORDER &&
                compaction_ready(zone, sc)) {
                sc->compaction_ready = true;//这个zone回收完成，由zone水位判断
                continue;
            }

            /*
             * Shrink each node in the zonelist once. If the
             * zonelist is ordered by zone (not the default) then a
             * node may be shrunk multiple times but in that case
             * the user prefers lower zones being preserved.
             */
            if (zone->zone_pgdat == last_pgdat)
                continue;

            /*
             * This steals pages from memory cgroups over softlimit
             * and returns the number of reclaimed pages and
             * scanned pages. This works for global memory pressure
             * and balancing, not for a memcg's limit.
             */
            nr_soft_scanned = 0;
            /*从超过软限制的memcg中窃取页，注意if (!cgroup_reclaim(sc))条件
             * 从soft_limit_tree红黑树最右边拿个memcg，最右边意味着超过soft_limit最大
             * 依次调用mem_cgroup_soft_limit_reclaim->mem_cgroup_soft_reclaim->mem_cgroup_shrink_node->shrink_lrvec
             */            
            nr_soft_reclaimed = mem_cgroup_soft_limit_reclaim(zone->zone_pgdat,
                        sc->order, sc->gfp_mask,
                        &nr_soft_scanned);
            sc->nr_reclaimed += nr_soft_reclaimed;
            sc->nr_scanned += nr_soft_scanned;
            /* need some check for avoid more shrink_zone() */
        }

        if (!first_pgdat)
            first_pgdat = zone->zone_pgdat;

        /* See comment about same check for global reclaim above */
        if (zone->zone_pgdat == last_pgdat)
            continue;
        last_pgdat = zone->zone_pgdat;
        shrink_node(zone->zone_pgdat, sc);
    }

    if (first_pgdat)
        consider_reclaim_throttle(first_pgdat, sc);//若回收的不够多，等待，或回收的足够多，唤醒之前等待的进程，见回收节流章节

    /*
     * Restore to original mask to avoid the impact on the caller if we
     * promoted it to __GFP_HIGHMEM.
     */
    sc->gfp_mask = orig_mask;
}
```
