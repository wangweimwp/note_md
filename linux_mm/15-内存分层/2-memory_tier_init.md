```c
static int __init memory_tier_init(void)
{
    int ret, node;
    struct memory_tier *memtier;

    ret = subsys_virtual_register(&memory_tier_subsys, NULL);
    if (ret)
        panic("%s() failed to register memory tier subsystem\n", __func__);

#ifdef CONFIG_MIGRATION
    node_demotion = kcalloc(nr_node_ids, sizeof(struct demotion_nodes),
                GFP_KERNEL);
    WARN_ON(!node_demotion);
#endif
    mutex_lock(&memory_tier_lock);
    /*
     * For now we can have 4 faster memory tiers with smaller adistance
     * than default DRAM tier.初始化default_dram_type 结构体，访问距离为MEMTIER_ADISTANCE_DRAM
     */
    default_dram_type = mt_find_alloc_memory_type(MEMTIER_ADISTANCE_DRAM,
                              &default_memory_types);
    if (IS_ERR(default_dram_type))
        panic("%s() failed to allocate default DRAM tier\n", __func__);

    /*
     * Look at all the existing N_MEMORY nodes and add them to
     * default memory tier or to a tier if we already have memory
     * types assigned.为有cpu的内存节点初始化struct memory_tier结构体
     */
    for_each_node_state(node, N_MEMORY) {
        if (!node_state(node, N_CPU))
            /*
             * Defer memory tier initialization on
             * CPUless numa nodes. These will be initialized
             * after firmware and devices are initialized.
             */
            continue;

        memtier = set_node_memory_tier(node);//计算距离
        if (IS_ERR(memtier))
            /*
             * Continue with memtiers we are able to setup
             */
            break;
    }
    establish_demotion_targets();
    mutex_unlock(&memory_tier_lock);

    hotplug_memory_notifier(memtier_hotplug_callback, MEMTIER_HOTPLUG_PRI);
    return 0;
}


static struct memory_tier *set_node_memory_tier(int node)
{
	struct memory_tier *memtier;
	struct memory_dev_type *memtype = default_dram_type;
	int adist = MEMTIER_ADISTANCE_DRAM;
	pg_data_t *pgdat = NODE_DATA(node);


	lockdep_assert_held_once(&memory_tier_lock);

	if (!node_state(node, N_MEMORY))
		return ERR_PTR(-EINVAL);
    /*用注册好的算法计算路径距离
    用register_mt_adistance_algorithm注册
    hmat_adist_nb
    */
	mt_calc_adistance(node, &adist);
	if (!node_memory_types[node].memtype) {
		memtype = mt_find_alloc_memory_type(adist, &default_memory_types);
		if (IS_ERR(memtype)) {
			memtype = default_dram_type;
			pr_info("Failed to allocate a memory type. Fall back.\n");
		}
	}

	__init_node_memory_type(node, memtype);

	memtype = node_memory_types[node].memtype;
	node_set(node, memtype->nodes);
	memtier = find_create_memory_tier(memtype);
	if (!IS_ERR(memtier))
		rcu_assign_pointer(pgdat->memtier, memtier);
	return memtier;
}
```
