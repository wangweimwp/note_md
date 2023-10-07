```c
void lru_gen_online_memcg(struct mem_cgroup *memcg)
{
	int gen;
	int nid;
	int bin = get_random_u32_below(MEMCG_NR_BINS);

	for_each_node(nid) {//遍历每个内存节点
		struct pglist_data *pgdat = NODE_DATA(nid);//拿到内存节点结构体
		struct lruvec *lruvec = get_lruvec(memcg, nid);//拿到当前memcg的nid节点上的lru列表，一个memcg可以跨多个内存节点

		spin_lock(&pgdat->memcg_lru.lock);

		VM_WARN_ON_ONCE(!hlist_nulls_unhashed(&lruvec->lrugen.list));

		gen = get_memcg_gen(pgdat->memcg_lru.seq);//拿到当前内存节点的memcg代

		//将lruvec->lrugen（struct lru_gen_folio mglru结构体，将folio分成四代）添加到nid内存节点的memcg的gen代中		
		hlist_nulls_add_tail_rcu(&lruvec->lrugen.list, &pgdat->memcg_lru.fifo[gen][bin]);
		pgdat->memcg_lru.nr_memcgs[gen]++;

		lruvec->lrugen.gen = gen;

		spin_unlock(&pgdat->memcg_lru.lock);
	}
}
```


