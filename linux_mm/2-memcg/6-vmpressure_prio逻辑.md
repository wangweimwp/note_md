```c
/*
 * The window size (vmpressure_win) is the number of scanned pages before
 * we try to analyze scanned/reclaimed ratio. So the window is used as a
 * rate-limit tunable for the "low" level notification, and also for
 * averaging the ratio for medium/critical levels. Using small window
 * sizes can cause lot of false positives, but too big window size will
 * delay the notifications.
 *
 * As the vmscan reclaimer logic works with chunks which are multiple of
 * SWAP_CLUSTER_MAX, it makes sense to use it for the window size as well.
 *
 * TODO: Make the window size depend on machine size, as we do for vmstat
 * thresholds. Currently we set it to 512 pages (2MB for 4KB pages).
 */
static const unsigned long vmpressure_win = SWAP_CLUSTER_MAX * 16 = 32 * 16;

/*
 * When there are too little pages left to scan, vmpressure() may miss the
 * critical pressure as number of pages will be less than "window size".
 * However, in that case the vmscan priority will raise fast as the
 * reclaimer will try to scan LRUs more deeply.
 *
 * The vmscan logic considers these special priorities:
 *
 * prio == DEF_PRIORITY (12): reclaimer starts with that value
 * prio <= DEF_PRIORITY - 2 : kswapd becomes somewhat overwhelmed  变的不知所措
 * prio == 0                : close to OOM, kernel scans every page in an lru
 *
 * Any value in this range is acceptable for this tunable (i.e. from 12 to
 * 0). Current value for the vmpressure_level_critical_prio is chosen
 * empirically, but the number, in essence, means that we consider
 * critical level when scanning depth is ~10% of the lru size (vmscan
 * scans 'lru_size >> prio' pages, so it is actually 12.5%, or one
 * eights).
临界压力sc->priority值，期望回收lru_size 的10%，但由于该值是int类型，实际是12.5%
 */
static const unsigned int vmpressure_level_critical_prio = ilog2(100 / 10) = log2(10);


/**
 * vmpressure_prio() - Account memory pressure through reclaimer priority level
 * @gfp:    reclaimer's gfp mask
 * @memcg:    cgroup memory controller handle
 * @prio:    reclaimer's priority
 *
 * This function should be called from the reclaim path every time when
 * the vmscan's reclaiming priority (scanning depth) changes.
 *
 * This function does not return any value.
 */
void vmpressure_prio(gfp_t gfp, struct mem_cgroup *memcg, int prio)
{
    /*
     * We only use prio for accounting critical level. For more info
     * see comment for vmpressure_level_critical_prio variable above.
     */
    if (prio > vmpressure_level_critical_prio)//大于3
        return;

    /*
     * OK, the prio is below the threshold, updating vmpressure
     * information before shrinker dives into long shrinking of long
     * range vmscan. Passing scanned = vmpressure_win, reclaimed = 0
     * to the vmpressure() basically means that we signal 'critical'
     * level.
     */
    vmpressure(gfp, memcg, true, vmpressure_win, 0);
}

/**
 * vmpressure() - Account memory pressure through scanned/reclaimed ratio
 * @gfp:    reclaimer's gfp mask
 * @memcg:    cgroup memory controller handle
 * @tree:    legacy subtree mode
 * @scanned:    number of pages scanned
 * @reclaimed:    number of pages reclaimed
 *
 * This function should be called from the vmscan reclaim path to account
 * "instantaneous" memory pressure (scanned/reclaimed ratio). The raw
 * pressure index is then further refined and averaged over time.
 *
 * If @tree is set, vmpressure is in traditional userspace reporting
 * mode: @memcg is considered the pressure root and userspace is
 * notified of the entire subtree's reclaim efficiency.
 *
 * If @tree is not set, reclaim efficiency is recorded for @memcg, and
 * only in-kernel users are notified.
 *
 * This function does not return any value.
 */
void vmpressure(gfp_t gfp, struct mem_cgroup *memcg, bool tree,
        unsigned long scanned, unsigned long reclaimed)
{    //gfp = GFP_KERNEL 是do_anonymous_page传进来的
    struct vmpressure *vmpr;

    if (mem_cgroup_disabled())
        return;

    vmpr = memcg_to_vmpressure(memcg);

    /*
     * Here we only want to account pressure that userland is able to
     * help us with. For example, suppose that DMA zone is under
     * pressure; if we notify userland about that kind of pressure,
     * then it will be mostly a waste as it will trigger unnecessary
     * freeing of memory by userland (since userland is more likely to
     * have HIGHMEM/MOVABLE pages instead of the DMA fallback). That
     * is why we include only movable, highmem and FS/IO pages.
     * Indirect reclaim (kswapd) sets sc->gfp_mask to GFP_KERNEL, so
     * we account it too.
     */
    if (!(gfp & (__GFP_HIGHMEM | __GFP_MOVABLE | __GFP_IO | __GFP_FS)))
        return;

    /*
     * If we got here with no pages scanned, then that is an indicator
     * that reclaimer was unable to find any shrinkable LRUs at the
     * current scanning depth. But it does not mean that we should
     * report the critical pressure, yet. If the scanning priority
     * (scanning depth) goes too high (deep), we will be notified
     * through vmpressure_prio(). But so far, keep calm.
     */
    if (!scanned)
        return;

    if (tree) {
        spin_lock(&vmpr->sr_lock);
        scanned = vmpr->tree_scanned += scanned;
        vmpr->tree_reclaimed += reclaimed;
        spin_unlock(&vmpr->sr_lock);

        if (scanned < vmpressure_win)
            return;
        schedule_work(&vmpr->work);
    } else {
        enum vmpressure_levels level;

        /* For now, no users for root-level efficiency */
        if (!memcg || mem_cgroup_is_root(memcg))
            return;

        spin_lock(&vmpr->sr_lock);
        scanned = vmpr->scanned += scanned;
        reclaimed = vmpr->reclaimed += reclaimed;
        if (scanned < vmpressure_win) {
            spin_unlock(&vmpr->sr_lock);
            return;
        }
        vmpr->scanned = vmpr->reclaimed = 0;
        spin_unlock(&vmpr->sr_lock);

        level = vmpressure_calc_level(scanned, reclaimed);

        if (level > VMPRESSURE_LOW) {
            /*
             * Let the socket buffer allocator know that
             * we are having trouble reclaiming LRU pages.
             *
             * For hysteresis keep the pressure state
             * asserted for a second in which subsequent
             * pressure events can occur.
             */
            WRITE_ONCE(memcg->socket_pressure, jiffies + HZ);
        }
    }
}
```
