```c

static int shrink_one(struct lruvec *lruvec, struct scan_control *sc)
{
	bool success;
	unsigned long scanned = sc->nr_scanned;
	unsigned long reclaimed = sc->nr_reclaimed;
	int seg = lru_gen_memcg_seg(lruvec);
	struct mem_cgroup *memcg = lruvec_memcg(lruvec);
	struct pglist_data *pgdat = lruvec_pgdat(lruvec);

	/* see the comment on MEMCG_NR_GENS */
	if (!lruvec_is_sizable(lruvec, sc))
		return seg != MEMCG_LRU_TAIL ? MEMCG_LRU_TAIL : MEMCG_LRU_YOUNG;

	mem_cgroup_calculate_protection(NULL, memcg);

	if (mem_cgroup_below_min(NULL, memcg))
		return MEMCG_LRU_YOUNG;

	if (mem_cgroup_below_low(NULL, memcg)) {
		/* see the comment on MEMCG_NR_GENS */
		if (seg != MEMCG_LRU_TAIL)
			return MEMCG_LRU_TAIL;

		memcg_memory_event(memcg, MEMCG_LOW);
	}

	success = try_to_shrink_lruvec(lruvec, sc);

	shrink_slab(sc->gfp_mask, pgdat->node_id, memcg, sc->priority);

	if (!sc->proactive)
		vmpressure(sc->gfp_mask, memcg, false, sc->nr_scanned - scanned,
			   sc->nr_reclaimed - reclaimed);

	sc->nr_reclaimed += current->reclaim_state->reclaimed_slab;
	current->reclaim_state->reclaimed_slab = 0;

	return success ? MEMCG_LRU_YOUNG : 0;
}

```


