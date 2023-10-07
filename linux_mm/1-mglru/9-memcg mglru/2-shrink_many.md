```c
static void shrink_many(struct pglist_data *pgdat, struct scan_control *sc)
{
	int op;
	int gen;
	int bin;
	int first_bin;
	struct lruvec *lruvec;
	struct lru_gen_folio *lrugen;
	struct mem_cgroup *memcg;
	const struct hlist_nulls_node *pos;
	unsigned long nr_to_reclaim = get_nr_to_reclaim(sc);

	bin = first_bin = get_random_u32_below(MEMCG_NR_BINS);//选定一个bin值
restart:
	op = 0;
	memcg = NULL;
	gen = get_memcg_gen(READ_ONCE(pgdat->memcg_lru.seq));

	rcu_read_lock();

	hlist_nulls_for_each_entry_rcu(lrugen, pos, &pgdat->memcg_lru.fifo[gen][bin], list) {//遍历pgdat->memcg_lru.fifo上的节点
		if (op)
			lru_gen_rotate_memcg(lruvec, op);//根据shrink_one的返回值做hlist_nulls_node移动和

		mem_cgroup_put(memcg);

		lruvec = container_of(lrugen, struct lruvec, lrugen);
		memcg = lruvec_memcg(lruvec);

		if (!mem_cgroup_tryget(memcg)) {
			op = 0;
			memcg = NULL;
			continue;
		}

		rcu_read_unlock();

		op = shrink_one(lruvec, sc);//对拿到的lruvec进行内存回收

		rcu_read_lock();

		if (sc->nr_reclaimed >= nr_to_reclaim)
			break;
	}

	rcu_read_unlock();

	if (op)
		lru_gen_rotate_memcg(lruvec, op);

	mem_cgroup_put(memcg);

	if (sc->nr_reclaimed >= nr_to_reclaim)
		return;

	/* restart if raced with lru_gen_rotate_memcg() */
	if (gen != get_nulls_value(pos))
		goto restart;

	/* try the rest of the bins of the current generation */
	bin = get_memcg_bin(bin + 1);//选定下一个bin值
	if (bin != first_bin)
		goto restart;
}
```


