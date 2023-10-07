```c
static void lru_gen_shrink_node(struct pglist_data *pgdat, struct scan_control *sc)
{
	struct blk_plug plug;
	unsigned long reclaimed = sc->nr_reclaimed;

	VM_WARN_ON_ONCE(!global_reclaim(sc));

	/*
	 * Unmapped clean folios are already prioritized. Scanning for more of
	 * them is likely futile and can cause high reclaim latency when there
	 * is a large number of memcgs.
	 */
	if (!sc->may_writepage || !sc->may_unmap)
		goto done;

	lru_add_drain();//将当前cpu的cpu_fbatches里的页面加到对应的lru列表下,这里似乎也会把walk_pte_range中更新过gen的也放到对应列表里

	blk_start_plug(&plug);//告诉block层，接下来可能有大量IO请求

	set_mm_walk(pgdat, sc->proactive);

	set_initial_priority(pgdat, sc);//设置回收优先级

	if (current_is_kswapd())
		sc->nr_reclaimed = 0;

	if (mem_cgroup_disabled())
		shrink_one(&pgdat->__lruvec, sc);
	else
		shrink_many(pgdat, sc);

	if (current_is_kswapd())
		sc->nr_reclaimed += reclaimed;

	clear_mm_walk();

	blk_finish_plug(&plug);
done:
	/* kswapd should never fail */
	pgdat->kswapd_failures = 0;
}

static void set_initial_priority(struct pglist_data *pgdat, struct scan_control *sc)
{
	int priority;
	unsigned long reclaimable;
	struct lruvec *lruvec = mem_cgroup_lruvec(NULL, pgdat);

	if (sc->priority != DEF_PRIORITY || sc->nr_to_reclaim < MIN_LRU_BATCH)
		return;
	/*
	 * Determine the initial priority based on ((total / MEMCG_NR_GENS) >>
	 * priority) * reclaimed_to_scanned_ratio = nr_to_reclaim, where the
	 * estimated reclaimed_to_scanned_ratio = inactive / total.
	 */
	reclaimable = node_page_state(pgdat, NR_INACTIVE_FILE);
	if (get_swappiness(lruvec, sc))
		reclaimable += node_page_state(pgdat, NR_INACTIVE_ANON);

	reclaimable /= MEMCG_NR_GENS;

	/* round down reclaimable and round up sc->nr_to_reclaim */
	priority = fls_long(reclaimable) - 1 - fls_long(sc->nr_to_reclaim - 1);

	sc->priority = clamp(priority, 0, DEF_PRIORITY);
}
```


