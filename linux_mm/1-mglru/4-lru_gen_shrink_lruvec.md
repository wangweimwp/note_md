```c

/*
 * current->reclaim_state points to one of these when a task is running
 * memory reclaim
 */
struct reclaim_state {
	unsigned long reclaimed_slab;
#ifdef CONFIG_LRU_GEN
	/* per-thread mm walk data */
	struct lru_gen_mm_walk *mm_walk;
#endif
};

static bool try_to_shrink_lruvec(struct lruvec *lruvec, struct scan_control *sc)//新版
{
	long nr_to_scan;
	unsigned long scanned = 0;
	unsigned long nr_to_reclaim = get_nr_to_reclaim(sc);
	int swappiness = get_swappiness(lruvec, sc);

	/* clean file folios are more likely to exist */
	if (swappiness && !(sc->gfp_mask & __GFP_IO))
		swappiness = 1;

	while (true) {
		int delta;

		nr_to_scan = get_nr_to_scan(lruvec, sc, swappiness);
		if (nr_to_scan <= 0)
			break;

		delta = evict_folios(lruvec, sc, swappiness);
		if (!delta)
			break;

		scanned += delta;
		if (scanned >= nr_to_scan)
			break;

		if (sc->nr_reclaimed >= nr_to_reclaim)
			break;

		cond_resched();
	}

	/* whether try_to_inc_max_seq() was successful 
		get_nr_to_scan中不跑try_to_inc_max_seq，nr_to_scan必定>=0 返回失败
		get_nr_to_scan中跑try_to_inc_max_seq，max_seq增加成功返回ture，增加失败返回fasle
	*/
	return nr_to_scan < 0;
}

static void lru_gen_shrink_lruvec(struct lruvec *lruvec, struct scan_control *sc)//旧版
{
	struct blk_plug plug;
	bool need_aging = false;
	bool need_swapping = false;
	unsigned long scanned = 0;
	unsigned long reclaimed = sc->nr_reclaimed;
	DEFINE_MAX_SEQ(lruvec);		//unsigned long max_seq = READ_ONCE((lruvec)->lrugen.max_seq)  获取最大gen

	lru_add_drain();

	blk_start_plug(&plug);

	set_mm_walk(lruvec_pgdat(lruvec));	//获取待扫描页面列表 更新当前进程的reclaim_state->mm_walk扫描状态

	while (true) {
		int delta;
		int swappiness;
		unsigned long nr_to_scan;

		if (sc->may_swap)/* Can folios be swapped as part of reclaim? */
			swappiness = get_swappiness(lruvec, sc);//获取扫描活跃度
		else if (!cgroup_reclaim(sc) && get_swappiness(lruvec, sc))
			/*
	 * The memory cgroup that hit its limit and as a result is the
	 * primary target of this reclaim invocation.
	 * 
	 * struct mem_cgroup *target_mem_cgroup;
	 */
			swappiness = 1;
		else
			swappiness = 0;

		nr_to_scan = get_nr_to_scan(lruvec, sc, swappiness, &need_aging);//获取需要扫描的page，并根据需要判断是都进行老化
		if (!nr_to_scan)
			goto done;

		delta = evict_folios(lruvec, sc, swappiness, &need_swapping);
		if (!delta)
			goto done;

		scanned += delta;
		if (scanned >= nr_to_scan)
			break;

		if (should_abort_scan(lruvec, max_seq, sc, need_swapping))
			break;

		cond_resched();
	}

	/* see the comment in lru_gen_age_node() */
	if (sc->nr_reclaimed - reclaimed >= MIN_LRU_BATCH && !need_aging)
		sc->memcgs_need_aging = false;
done:
	clear_mm_walk();

	blk_finish_plug(&plug);
}


static struct lru_gen_mm_walk *set_mm_walk(struct pglist_data *pgdat)
{
	struct lru_gen_mm_walk *walk = current->reclaim_state->mm_walk;	//获取当前进程的reclaim_state->mm_walk

	if (pgdat && current_is_kswapd()) {
		VM_WARN_ON_ONCE(walk);

		walk = &pgdat->mm_walk;	//在kswapd进程里，将pglist_data->mm_walk设置到当前进程
	} else if (!pgdat && !walk) {
		VM_WARN_ON_ONCE(current_is_kswapd());
		//似乎是进程初始化时pgdat和walk都为空
		walk = kzalloc(sizeof(*walk), __GFP_HIGH | __GFP_NOMEMALLOC | __GFP_NOWARN);
	}

	current->reclaim_state->mm_walk = walk;//再设置到当前进程里，

	return walk;
}


/*
 * For future optimizations:
 * 1. Defer try_to_inc_max_seq() to workqueues to reduce latency for memcg
 *    reclaim.
	放到工作队列里减少延时
 */
static unsigned long get_nr_to_scan(struct lruvec *lruvec, struct scan_control *sc,
				    bool can_swap, bool *need_aging)
{
	unsigned long nr_to_scan;
	struct mem_cgroup *memcg = lruvec_memcg(lruvec);
	DEFINE_MAX_SEQ(lruvec);
	DEFINE_MIN_SEQ(lruvec);

	if (mem_cgroup_below_min(sc->target_mem_cgroup, memcg) ||
	    (mem_cgroup_below_low(sc->target_mem_cgroup, memcg) &&
	     !sc->memcg_low_reclaim))
		return 0;

	*need_aging = should_run_aging(lruvec, max_seq, min_seq, sc, can_swap, &nr_to_scan);
	if (!*need_aging)
		return nr_to_scan;//若不需要跑老化，直接返回

	/* skip the aging path at the default priority */
	if (sc->priority == DEF_PRIORITY)
		goto done;

	/* leave the work to lru_gen_age_node() */
	if (current_is_kswapd())
		return 0;

	//最小实现直接调用inc_max_seq(lruvec, max_seq, can_swap);
	if (try_to_inc_max_seq(lruvec, max_seq, sc, can_swap, false))
		return nr_to_scan;
done:
	/**/
	return min_seq[!can_swap] + MIN_NR_GENS <= max_seq ? nr_to_scan : 0;
}


static bool should_run_aging(struct lruvec *lruvec, unsigned long max_seq, unsigned long *min_seq,
			     struct scan_control *sc, bool can_swap, unsigned long *nr_to_scan)
{//对lrugen->nr_pages中young和old页进行统计，计算代差，结合两者，计算出nr_to_scan和是否需要老化
	int gen, type, zone;
	unsigned long old = 0;
	unsigned long young = 0;
	unsigned long total = 0;
	struct lru_gen_struct *lrugen = &lruvec->lrugen;
	struct mem_cgroup *memcg = lruvec_memcg(lruvec);
	
	/*can_swap: lru_gen_shrink_lruvec里的swappiness，swappiness = 0只扫描file页面
	swappiness 非 0，扫描anon和file页面*/
	for (type = !can_swap; type < ANON_AND_FILE; type++) {
		unsigned long seq;

		for (seq = min_seq[type]; seq <= max_seq; seq++) {
			unsigned long size = 0;

			gen = lru_gen_from_seq(seq);//获取相对gen

			for (zone = 0; zone < MAX_NR_ZONES; zone++)//将所有zone的低gen代的type页面加起来
				size += max(READ_ONCE(lrugen->nr_pages[gen][type][zone]), 0L);

			total += size;
			if (seq == max_seq)
				young += size;//最年轻页面数量
			else if (seq + MIN_NR_GENS == max_seq)
				old += size;//
			/*第 0,1,2,3代共4个gen，第0代是要被逐出的，第1代是old的，第4代是young的*/
		}
	}

	/* try to scrape all its memory if this memcg was deleted */
	/* 如果 cgroup正在运行，total/2^priority,若cgroup不在线，这total不变*/
	*nr_to_scan = mem_cgroup_online(memcg) ? (total >> sc->priority) : total;

	/*
	 * The aging tries to be lazy to reduce the overhead, while the eviction
	 * stalls when the number of generations reaches MIN_NR_GENS. Hence, the
	 * ideal number of generations is MIN_NR_GENS+1. 
	 * 计算代差，min_seq和max_seq只相差1代跑老化
	 * 		 min_seq和max_seq只相2代以上不跑老化
	 * 理想代差是2    0 1 2  代
	 */
	if (min_seq[!can_swap] + MIN_NR_GENS > max_seq)  //max_seq-min_seq>MIN_NR_GENS
		return true;
	if (min_seq[!can_swap] + MIN_NR_GENS < max_seq)  //max_seq-min_seq<MIN_NR_GENS
		return false;

	/*
	 * It's also ideal to spread pages out evenly, i.e., 1/(MIN_NR_GENS+1)
	 * of the total number of pages for each generation. A reasonable range
	 * for this average portion is [1/MIN_NR_GENS, 1/(MIN_NR_GENS+2)]. The
	 * aging cares about the upper bound of hot pages, while the eviction
	 * cares about the lower bound of cold pages.
	 * min_seq和max_seq只相差2具体操作
	 */
	if (young * MIN_NR_GENS > total)
		return true;
	if (old * (MIN_NR_GENS + 2) < total)
		return true;

	return false;
}

```


