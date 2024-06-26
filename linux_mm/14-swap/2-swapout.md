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




