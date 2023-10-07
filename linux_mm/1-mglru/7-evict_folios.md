最小实现

```c

static int evict_folios(struct lruvec *lruvec, struct scan_control *sc, int swappiness)
{
	int type;
	int scanned;
	int reclaimed;
	LIST_HEAD(list);
	struct folio *folio;
	enum vm_event_item item;
	struct reclaim_stat stat;
	struct mem_cgroup *memcg = lruvec_memcg(lruvec);
	struct pglist_data *pgdat = lruvec_pgdat(lruvec);

	spin_lock_irq(&lruvec->lru_lock);

	scanned = isolate_folios(lruvec, sc, swappiness, &type, &list);

	scanned += try_to_inc_min_seq(lruvec, swappiness);

	if (get_nr_gens(lruvec, !swappiness) == MIN_NR_GENS)
		scanned = 0;

	spin_unlock_irq(&lruvec->lru_lock);

	if (list_empty(&list))
		return scanned;

	/*与shrink_folio_list作用相同，确认那些页面要请出去，那些页面保留
		list是inactive列表
	 */
	reclaimed = shrink_page_list(&list, pgdat, sc, &stat, false);

	list_for_each_entry(folio, &list, lru) {
		/* restore LRU_REFS_FLAGS cleared by isolate_folio() */
		if (folio_test_workingset(folio))
			folio_set_referenced(folio);

		/* don't add rejected pages to the oldest generation */
		if (folio_test_reclaim(folio) &&
		    (folio_test_dirty(folio) || folio_test_writeback(folio)))
			folio_clear_active(folio);
		else
			folio_set_active(folio);
	}

	spin_lock_irq(&lruvec->lru_lock);

	move_pages_to_lru(lruvec, &list);

	item = current_is_kswapd() ? PGSTEAL_KSWAPD : PGSTEAL_DIRECT;
	if (!cgroup_reclaim(sc))
		__count_vm_events(item, reclaimed);
	__count_memcg_events(memcg, item, reclaimed);
	__count_vm_events(PGSTEAL_ANON + type, reclaimed);

	spin_unlock_irq(&lruvec->lru_lock);

	mem_cgroup_uncharge_list(&list);
	free_unref_page_list(&list);

	sc->nr_reclaimed += reclaimed;

	return scanned;
}

static bool sort_folio(struct lruvec *lruvec, struct folio *folio, int tier_idx)
{
	bool success;
	int gen = folio_lru_gen(folio);
	int type = folio_is_file_lru(folio);
	int zone = folio_zonenum(folio);
	int delta = folio_nr_pages(folio);
	int refs = folio_lru_refs(folio);
	int tier = lru_tier_from_refs(refs);
	struct lru_gen_struct *lrugen = &lruvec->lrugen;

	VM_WARN_ON_ONCE_FOLIO(gen >= MAX_NR_GENS, folio);

	/* unevictable */
	if (!folio_evictable(folio)) {/将filio从原来的folio->lru列表删除，添加到下一gen，3中添加方式见lru_gen_add_dolio
		success = lru_gen_del_folio(lruvec, folio, true);//从原有lrugen->lists删除，更新lrugen->nr_pages和zone->vm_stat，
		VM_WARN_ON_ONCE_FOLIO(!success, folio);
		folio_set_unevictable(folio);
		lruvec_add_folio(lruvec, folio);//将页面（folio->lru）加到lrugen->lists，更新lrugen->nr_pages和zone->vm_stat
		__count_vm_events(UNEVICTABLE_PGCULLED, delta);
		return true;
	}

	/* dirty lazyfree *///干净匿名页面变脏了，需要设置PG_swapbacked，表明可以回写到swap分区
	if (type == LRU_GEN_FILE && folio_test_anon(folio) && folio_test_dirty(folio)) {
		success = lru_gen_del_folio(lruvec, folio, true);
		VM_WARN_ON_ONCE_FOLIO(!success, folio);
		folio_set_swapbacked(folio);
		lruvec_add_folio_tail(lruvec, folio);
		return true;
	}

	/* promoted */
	if (gen != lru_gen_from_seq(lrugen->min_seq[type])) {
		list_move(&folio->lru, &lrugen->lists[gen][type][zone]);
		return true;
	}

	/* protected */
	if (tier > tier_idx) {
		int hist = lru_hist_from_seq(lrugen->min_seq[type]);//获取相对gen

		gen = folio_inc_gen(lruvec, folio, false);//操作folio->flag,gen+1,清楚flag一些位，，更新lrugen->nr_pages和zone->vm_stat
		list_move_tail(&folio->lru, &lrugen->lists[gen][type][zone]);

		WRITE_ONCE(lrugen->protected[hist][type][tier - 1],
			   lrugen->protected[hist][type][tier - 1] + delta);
		__mod_lruvec_state(lruvec, WORKINGSET_ACTIVATE_BASE + type, delta);//对lruvec->pgdat->vm_stat操作
		return true;
	}

	/* waiting for writeback */
	if (folio_test_locked(folio) || folio_test_writeback(folio) ||
	    (type == LRU_GEN_FILE && folio_test_dirty(folio))) {
		gen = folio_inc_gen(lruvec, folio, true);
		list_move(&folio->lru, &lrugen->lists[gen][type][zone]);
		return true;
	}

	return false;
}


static inline void lru_gen_update_size(struct lruvec *lruvec, struct folio *folio,
				       int old_gen, int new_gen)
{
	int type = folio_is_file_lru(folio);
	int zone = folio_zonenum(folio);
	int delta = folio_nr_pages(folio);
	enum lru_list lru = type * LRU_INACTIVE_FILE;
	struct lru_gen_struct *lrugen = &lruvec->lrugen;

	VM_WARN_ON_ONCE(old_gen != -1 && old_gen >= MAX_NR_GENS);
	VM_WARN_ON_ONCE(new_gen != -1 && new_gen >= MAX_NR_GENS);
	VM_WARN_ON_ONCE(old_gen == -1 && new_gen == -1);

	if (old_gen >= 0)//更新lrugen->nr_pages数量
		WRITE_ONCE(lrugen->nr_pages[old_gen][type][zone],
			   lrugen->nr_pages[old_gen][type][zone] - delta);
	if (new_gen >= 0)
		WRITE_ONCE(lrugen->nr_pages[new_gen][type][zone],
			   lrugen->nr_pages[new_gen][type][zone] + delta);

	/* addition */
	if (old_gen < 0) {
		if (lru_gen_is_active(lruvec, new_gen))
			lru += LRU_ACTIVE;
		__update_lru_size(lruvec, lru, zone, delta);
		return;
	}

	/* deletion */
	if (new_gen < 0) {
		if (lru_gen_is_active(lruvec, old_gen))//folio的gen如果是max_seq或者max_seq - 1被认为是active，否者则是inactiove
			lru += LRU_ACTIVE;
		__update_lru_size(lruvec, lru, zone, -delta);//更新zone->vm_stat的计数，作用与zone->per_cpu_pageset相似，用于记录该zone的free_page
		return;
	}

	/* promotion */
	if (!lru_gen_is_active(lruvec, old_gen) && lru_gen_is_active(lruvec, new_gen)) {
		__update_lru_size(lruvec, lru, zone, -delta);
		__update_lru_size(lruvec, lru + LRU_ACTIVE, zone, delta);
	}

	/* demotion requires isolation, e.g., lru_deactivate_fn() */
	VM_WARN_ON_ONCE(lru_gen_is_active(lruvec, old_gen) && !lru_gen_is_active(lruvec, new_gen));
}


static inline bool lru_gen_add_folio(struct lruvec *lruvec, struct folio *folio, bool reclaiming)
{
	unsigned long seq;
	unsigned long flags;
	int gen = folio_lru_gen(folio);
	int type = folio_is_file_lru(folio);
	int zone = folio_zonenum(folio);
	struct lru_gen_struct *lrugen = &lruvec->lrugen;

	VM_WARN_ON_ONCE_FOLIO(gen != -1, folio);

	if (folio_test_unevictable(folio) || !lrugen->enabled)
		return false;
	/*
	 * There are three common cases for this page:
	 * 1. If it's hot, e.g., freshly faulted in or previously hot and
	 *    migrated, add it to the youngest generation.
	 * 2. If it's cold but can't be evicted immediately, i.e., an anon page
	 *    not in swapcache or a dirty page pending writeback, add it to the
	 *    second oldest generation.
	 * 3. Everything else (clean, cold) is added to the oldest generation.
	 */
	if (folio_test_active(folio))
		seq = lrugen->max_seq;
	else if ((type == LRU_GEN_ANON && !folio_test_swapcache(folio)) ||
		 (folio_test_reclaim(folio) &&
		  (folio_test_dirty(folio) || folio_test_writeback(folio))))
		seq = lrugen->min_seq[type] + 1;//这里有+1操作
	else
		seq = lrugen->min_seq[type];

	gen = lru_gen_from_seq(seq);
	flags = (gen + 1UL) << LRU_GEN_PGOFF;//这里又有+1操作
	/* see the comment on MIN_NR_GENS about PG_active */
	set_mask_bits(&folio->flags, LRU_GEN_MASK | BIT(PG_active), flags);

	lru_gen_update_size(lruvec, folio, -1, gen);
	/* for folio_rotate_reclaimable() */
	if (reclaiming)
		list_add_tail(&folio->lru, &lrugen->lists[gen][type][zone]);
	else
		list_add(&folio->lru, &lrugen->lists[gen][type][zone]);

	return true;
}
static bool isolate_folio(struct lruvec *lruvec, struct folio *folio, struct scan_control *sc)
{
	bool success;

	/* swapping inhibited 静止写入交换分区*/
	if (!(sc->gfp_mask & __GFP_IO) &&
	    (folio_test_dirty(folio) ||
	     (folio_test_anon(folio) && !folio_test_swapcache(folio))))
		return false;

	/* raced with release_pages() 若page->_refcount！=0 则+1，若page->_refcount==0说明该页已经释放 */
	if (!folio_try_get(folio))
		return false;

	/* raced with another isolation 清除PG_lru并返回旧值*/
	if (!folio_test_clear_lru(folio)) {
		folio_put(folio);//page->_refcount - 1,减一后若为0则释放
		return false;
	}

	/* see the comment on MAX_NR_TIERS */
	if (!folio_test_referenced(folio))//将folio->flags保存的gen清零
		set_mask_bits(&folio->flags, LRU_REFS_MASK | LRU_REFS_FLAGS, 0);//设置LRU_REFS_FLAGS为0

	/* for shrink_folio_list() */
	folio_clear_reclaim(folio);
	folio_clear_referenced(folio);

	success = lru_gen_del_folio(lruvec, folio, true);//从原有lrugen->lists删除，更新lrugen->nr_pages和zone->vm_stat
	VM_WARN_ON_ONCE_FOLIO(!success, folio);

	return true;
}


static int get_type_to_scan(struct lruvec *lruvec, int swappiness, int *tier_idx)
{
	int type, tier;
	struct ctrl_pos sp, pv;
	int gain[ANON_AND_FILE] = { swappiness, 200 - swappiness };

	/*
	 * Compare the first tier of anon with that of file to determine which
	 * type to scan. Also need to compare other tiers of the selected type
	 * with the first tier of the other type to determine the last tier (of
	 * the selected type) to evict.
	 */
	read_ctrl_pos(lruvec, LRU_GEN_ANON, 0, gain[LRU_GEN_ANON], &sp);
	read_ctrl_pos(lruvec, LRU_GEN_FILE, 0, gain[LRU_GEN_FILE], &pv);
	type = positive_ctrl_err(&sp, &pv);

	read_ctrl_pos(lruvec, !type, 0, gain[!type], &sp);
	for (tier = 1; tier < MAX_NR_TIERS; tier++) {
		read_ctrl_pos(lruvec, type, tier, gain[type], &pv);
		if (!positive_ctrl_err(&sp, &pv))
			break;
	}

	*tier_idx = tier - 1;

	return type;
}

```

```c

```


