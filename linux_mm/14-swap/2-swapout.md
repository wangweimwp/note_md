shrink_folio_list简化流程

```c
shrink_folio_liststatic unsigned int shrink_folio_list(struct list_head *folio_list,
        struct pglist_data *pgdat, struct scan_control *sc,
        struct reclaim_stat *stat, bool ignore_references)
{
    LIST_HEAD(ret_pages); //初始化ret_page链表, 把此次无法回收的page放到此链表
    LIST_HEAD(free_pages); //初始化free_page链表, 把此次可以回收的page放到此链表
    ....
    while (!list_empty(folio_list)) {
        /* folio是一个标准匿名页，而不是lazyfree页
         * Anonymous process memory has backing store?
         * Try to allocate it some swap space here.
         * Lazyfree folio could be freed directly
         */
        if (folio_test_anon(folio) && folio_test_swapbacked(folio)) {
            if (!folio_test_swapcache(folio)) {
                if (!(sc->gfp_mask & __GFP_IO))
                    goto keep_locked;
                if (folio_maybe_dma_pinned(folio))
                    goto keep_locked;
                if (folio_test_large(folio)) {
                    /* cannot split folio, skip it */
                    if (!can_split_folio(folio, NULL))
                        goto activate_locked;
                    /*
                     * Split folios without a PMD map right
                     * away. Chances are some or all of the
                     * tail pages can be freed without IO.
                     */
                    if (!folio_entire_mapcount(folio) &&
                        split_folio_to_list(folio,
                                folio_list))
                        goto activate_locked;
                }
                 /*为该页面分配swap entry,  并存放到page->private变量中，
                  *把page放入 swap cache，设置page的PG_swapcache和
                  *PG_dirty的flag，并更新swap_info_struct的页槽信息
                  */
                if (!add_to_swap(folio)) {
                    if (!folio_test_large(folio))
                        goto activate_locked_split;
                    /* Fallback to swap normal pages */
                    /*添加到swap失败，如果是大页，分裂后再添加*/
                    if (split_folio_to_list(folio,
                                folio_list))
                        goto activate_locked;
#ifdef CONFIG_TRANSPARENT_HUGEPAGE
                    count_memcg_folio_events(folio, THP_SWPOUT_FALLBACK, 1);
                    count_vm_event(THP_SWPOUT_FALLBACK);
#endif
                    if (!add_to_swap(folio))
                        goto activate_locked_split;
                }
            }
        } else if (folio_test_swapbacked(folio) &&
               folio_test_large(folio)) {
            /* Split shmem folio */
            if (split_folio_to_list(folio, folio_list))
                goto keep_locked;
        }
    }
}
```

```c
add_to_swap
    ->folio_alloc_swap
        ->get_swap_pages//申请swap slot，THP直接获取slot， 若per_cpu缓存用完则再申请填充
    ->add_to_swap_cache
    ->folio_mark_dirty(folio);
```

```c
int get_swap_pages(int n_goal, swp_entry_t swp_entries[], int entry_size)
{
    unsigned long size = swap_entry_size(entry_size);
    struct swap_info_struct *si, *next;
    long avail_pgs;
    int n_ret = 0;
    int node;

    /* Only single cluster request supported */
    WARN_ON_ONCE(n_goal > 1 && size == SWAPFILE_CLUSTER);

    spin_lock(&swap_avail_lock);

    avail_pgs = atomic_long_read(&nr_swap_pages) / size;
    if (avail_pgs <= 0) {
        spin_unlock(&swap_avail_lock);
        goto noswap;
    }

    n_goal = min3((long)n_goal, (long)SWAP_BATCH, avail_pgs);

    atomic_long_sub(n_goal * size, &nr_swap_pages);

start_over:
    node = numa_node_id();
    plist_for_each_entry_safe(si, next, &swap_avail_heads[node], avail_lists[node]) {
        /* requeue si to after same-priority siblings */
        plist_requeue(&si->avail_lists[node], &swap_avail_heads[node]);
        spin_unlock(&swap_avail_lock);
        spin_lock(&si->lock);
        if (!si->highest_bit || !(si->flags & SWP_WRITEOK)) {
        /*swap_info_struct 没有可用的slot，寻找下一个*/
            spin_lock(&swap_avail_lock);
            if (plist_node_empty(&si->avail_lists[node])) {
                spin_unlock(&si->lock);
                goto nextsi;
            }
            WARN(!si->highest_bit,
                 "swap_info %d in list but !highest_bit\n",
                 si->type);
            WARN(!(si->flags & SWP_WRITEOK),
                 "swap_info %d in list but !SWP_WRITEOK\n",
                 si->type);
            __del_from_avail_list(si);
            spin_unlock(&si->lock);
            goto nextsi;
        }
        if (size == SWAPFILE_CLUSTER) {//直接申请一个cluster
            if (si->flags & SWP_BLKDEV)
                n_ret = swap_alloc_cluster(si, swp_entries);
        } else
            n_ret = scan_swap_map_slots(si, SWAP_HAS_CACHE,
                            n_goal, swp_entries);//根据swap_map位图获取slot
        spin_unlock(&si->lock);
        if (n_ret || size == SWAPFILE_CLUSTER)
            goto check_out;
        cond_resched();

        spin_lock(&swap_avail_lock);
nextsi:
        /*
         * if we got here, it's likely that si was almost full before,
         * and since scan_swap_map_slots() can drop the si->lock,
         * multiple callers probably all tried to get a page from the
         * same si and it filled up before we could get one; or, the si
         * filled up between us dropping swap_avail_lock and taking
         * si->lock. Since we dropped the swap_avail_lock, the
         * swap_avail_head list may have been modified; so if next is
         * still in the swap_avail_head list then try it, otherwise
         * start over if we have not gotten any slots.
         */
        if (plist_node_empty(&next->avail_lists[node]))
            goto start_over;
    }

    spin_unlock(&swap_avail_lock);

check_out:
    if (n_ret < n_goal)
        atomic_long_add((long)(n_goal - n_ret) * size,
                &nr_swap_pages);
noswap:
    return n_ret;
}
```



```c
static int scan_swap_map_slots(struct swap_info_struct *si,
			       unsigned char usage, int nr,
			       swp_entry_t slots[])
{
/*1， si->lowest_bit指向第一个可用的slot
  2， 由于优先找到一个完整的空的cluster扫描，因此scan_base可能大于lowest_bit*/
	struct swap_cluster_info *ci;
	unsigned long offset;
	unsigned long scan_base;
	unsigned long last_in_cluster = 0;
	int latency_ration = LATENCY_LIMIT;
	int n_ret = 0;
	bool scanned_many = false;

	/*
	 * We try to cluster swap pages by allocating them sequentially
	 * in swap.  Once we've allocated SWAPFILE_CLUSTER pages this
	 * way, however, we resort to first-free allocation, starting
	 * a new cluster.  This prevents us from scattering swap pages
	 * all over the entire swap partition, so that we reduce
	 * overall disk seek times between swap pages.  -- sct
	 * But we do now try to find an empty cluster.  -Andrea
	 * And we let swap pages go all over an SSD partition.  Hugh
	 */

	si->flags += SWP_SCANNING;
	/*
	 * Use percpu scan base for SSD to reduce lock contention on
	 * cluster and swap cache.  For HDD, sequential access is more
	 * important.
	 */
	if (si->flags & SWP_SOLIDSTATE)
		scan_base = this_cpu_read(*si->cluster_next_cpu);
	else
		scan_base = si->cluster_next;
	offset = scan_base;

	/* SSD algorithm */
	if (si->cluster_info) {
		if (!scan_swap_map_try_ssd_cluster(si, &offset, &scan_base))
			goto scan;
	} else if (unlikely(!si->cluster_nr--)) {//当前cluster扫描倒计值用完
		if (si->pages - si->inuse_pages < SWAPFILE_CLUSTER) {
			si->cluster_nr = SWAPFILE_CLUSTER - 1;//剩余的页数小于SWAPFILE_CLUSTER
			goto checks;
		}

		spin_unlock(&si->lock);

		/*
		 * If seek is expensive, start searching for new cluster from
		 * start of partition, to minimize the span of allocated swap.
		 * If seek is cheap, that is the SWP_SOLIDSTATE si->cluster_info
		 * case, just handled by scan_swap_map_try_ssd_cluster() above.
		 */
		scan_base = offset = si->lowest_bit;
		last_in_cluster = offset + SWAPFILE_CLUSTER - 1;

		/* Locate the first empty (unaligned) cluster */
        /*当前cluster倒计值已用完
        找到一个完整的空的cluster 可以非SWAPFILE_CLUSTER地址对齐 */
		for (; last_in_cluster <= si->highest_bit; offset++) {
			if (si->swap_map[offset])
				last_in_cluster = offset + SWAPFILE_CLUSTER;//si->lowest_bit位置被占用了，循环边界向后移动一个
			else if (offset == last_in_cluster) {//找到了一个完整的空的cluster
				spin_lock(&si->lock);
				offset -= SWAPFILE_CLUSTER - 1;//空cluster的首地址
				si->cluster_next = offset;
				si->cluster_nr = SWAPFILE_CLUSTER - 1;
				goto checks;
			}
			if (unlikely(--latency_ration < 0)) {
				cond_resched();
				latency_ration = LATENCY_LIMIT;
			}
		}

		offset = scan_base;//尝试多次没找到一个完整的空的cluster
		spin_lock(&si->lock);
		si->cluster_nr = SWAPFILE_CLUSTER - 1;
	}

checks:
	if (si->cluster_info) {
		while (scan_swap_map_ssd_cluster_conflict(si, offset)) {
		/* take a break if we already got some slots */
			if (n_ret)
				goto done;
			if (!scan_swap_map_try_ssd_cluster(si, &offset,
							&scan_base))
				goto scan;
		}
	}
	if (!(si->flags & SWP_WRITEOK))
		goto no_page;
	if (!si->highest_bit)
		goto no_page;
	if (offset > si->highest_bit)
		scan_base = offset = si->lowest_bit;

	ci = lock_cluster(si, offset);
	/* reuse swap entry of cache-only swap if not busy. */
    /*swap消耗过半，看当前slot是否可复用*/
	if (vm_swap_full() && si->swap_map[offset] == SWAP_HAS_CACHE) {
		int swap_was_freed;
		unlock_cluster(ci);
		spin_unlock(&si->lock);
		swap_was_freed = __try_to_reclaim_swap(si, offset, TTRS_ANYWAY);
		spin_lock(&si->lock);
		/* entry was freed successfully, try to use this again */
		if (swap_was_freed)
			goto checks;
		goto scan; /* check next one */
	}

	if (si->swap_map[offset]) {
		unlock_cluster(ci);
		if (!n_ret)
			goto scan;
		else
			goto done;
	}
	WRITE_ONCE(si->swap_map[offset], usage);//将空闲page标记为使用
	inc_cluster_info_page(si, si->cluster_info, offset);//增加
	unlock_cluster(ci);

	swap_range_alloc(si, offset, 1);//更新lowest_bit、hight_bit、inuse_pages
	slots[n_ret++] = swp_entry(si->type, offset);//记录申请到的slot(swap entry)

	/* got enough slots or reach max slots? */
	if ((n_ret == nr) || (offset >= si->highest_bit))
		goto done;

	/* search for next available slot */

	/* time to take a break? 搜索了256次，休息一下*/
	if (unlikely(--latency_ration < 0)) {
		if (n_ret)
			goto done;
		spin_unlock(&si->lock);
		cond_resched();
		spin_lock(&si->lock);
		latency_ration = LATENCY_LIMIT;
	}

	/* try to get more slots in cluster */
	if (si->cluster_info) {
		if (scan_swap_map_try_ssd_cluster(si, &offset, &scan_base))
			goto checks;
	} else if (si->cluster_nr && !si->swap_map[++offset]) {
		/* non-ssd case, still more slots in cluster? */
		--si->cluster_nr;//又找到一个slot
		goto checks;
	}

	/*
	 * Even if there's no free clusters available (fragmented),
	 * try to scan a little more quickly with lock held unless we
	 * have scanned too many slots already.
	 */
	if (!scanned_many) {
		unsigned long scan_limit;

		if (offset < scan_base)
			scan_limit = scan_base;
		else
			scan_limit = si->highest_bit;
		for (; offset <= scan_limit && --latency_ration > 0;
		     offset++) {//没有完整的空的cluster，slot数量还不够，再次扫描
			if (!si->swap_map[offset])
				goto checks;
		}
	}

done:
	set_cluster_next(si, offset + 1);//设置下次扫描的cluster
	si->flags -= SWP_SCANNING;
	return n_ret;

scan:
	spin_unlock(&si->lock);
    /*对scan_base 到highest_bit再扫一遍 */
	while (++offset <= READ_ONCE(si->highest_bit)) {
		if (unlikely(--latency_ration < 0)) {
			cond_resched();
			latency_ration = LATENCY_LIMIT;
			scanned_many = true;
		}
		if (swap_offset_available_and_locked(si, offset))
			goto checks;
	}
    /*对lowest_bit到scan_base再扫一遍 */
	offset = si->lowest_bit;
	while (offset < scan_base) {
		if (unlikely(--latency_ration < 0)) {
			cond_resched();
			latency_ration = LATENCY_LIMIT;
			scanned_many = true;
		}
		if (swap_offset_available_and_locked(si, offset))
			goto checks;
		offset++;
	}
	spin_lock(&si->lock);

no_page:
	si->flags -= SWP_SCANNING;
	return n_ret;
}


```


