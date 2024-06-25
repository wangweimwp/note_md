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
    ->add_to_swap_cache
    ->folio_mark_dirty(folio);
```




