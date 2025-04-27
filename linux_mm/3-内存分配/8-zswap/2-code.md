```c
//初始化

/*********************************
* module init and exit
**********************************/
static int zswap_setup(void)
{
	struct zswap_pool *pool;
	int ret;
    //创建zswap用的slab描述符
	zswap_entry_cache = KMEM_CACHE(zswap_entry, 0);
	if (!zswap_entry_cache) {
		pr_err("entry cache creation failed\n");
		goto cache_fail;
	}
    //注册CPU热插拔回调函数
	ret = cpuhp_setup_state_multi(CPUHP_MM_ZSWP_POOL_PREPARE,
				      "mm/zswap_pool:prepare",
				      zswap_cpu_comp_prepare,
				      zswap_cpu_comp_dead);
	if (ret)
		goto hp_fail;
    //注册shrinker的工作队列，
	shrink_wq = alloc_workqueue("zswap-shrink",
			WQ_UNBOUND|WQ_MEM_RECLAIM, 1);
	if (!shrink_wq)
		goto shrink_wq_fail;
    //注册shrinker，申请内存慢速路径会掉
	zswap_shrinker = zswap_alloc_shrinker();
	if (!zswap_shrinker)
		goto shrinker_fail;
	if (list_lru_init_memcg(&zswap_list_lru, zswap_shrinker))//每个shrinker有ID，将zswap_shrinker与zswap_list_lru通过ID绑定，同时将zswap_list_lru添加到memcg_list_lrus上
		goto lru_fail;
	shrinker_register(zswap_shrinker);

	INIT_WORK(&zswap_shrink_work, shrink_worker);

	/*
		创建zpool（zbud/z3fold）,调用上边注册的zswap_cpu_comp_prepare函数，
		以及其他一些成员
	*/
	pool = __zswap_pool_create_fallback();
	if (pool) {
		pr_info("loaded using pool %s/%s\n", pool->tfm_name,
			zpool_get_type(pool->zpool));
		list_add(&pool->list, &zswap_pools);
		zswap_has_pool = true;
		static_branch_enable(&zswap_ever_enabled);
	} else {
		pr_err("pool creation failed\n");
		zswap_enabled = false;
	}

	if (zswap_debugfs_init())
		pr_warn("debugfs initialization failed\n");
	zswap_init_state = ZSWAP_INIT_SUCCEED;
	return 0;

lru_fail:
	shrinker_free(zswap_shrinker);
shrinker_fail:
	destroy_workqueue(shrink_wq);
shrink_wq_fail:
	cpuhp_remove_multi_state(CPUHP_MM_ZSWP_POOL_PREPARE);
hp_fail:
	kmem_cache_destroy(zswap_entry_cache);
cache_fail:
	/* if built-in, we aren't unloaded on failure; don't allow use */
	zswap_init_state = ZSWAP_INIT_FAILED;
	zswap_enabled = false;
	return -ENOMEM;
}
```
```c
bool zswap_store(struct folio *folio)
{
	long nr_pages = folio_nr_pages(folio);
	swp_entry_t swp = folio->swap;
	struct obj_cgroup *objcg = NULL;
	struct mem_cgroup *memcg = NULL;
	struct zswap_pool *pool;
	bool ret = false;
	long index;

	VM_WARN_ON_ONCE(!folio_test_locked(folio));
	VM_WARN_ON_ONCE(!folio_test_swapcache(folio));

	if (!zswap_enabled)
		goto check_old;

	objcg = get_obj_cgroup_from_folio(folio);
	if (objcg && !obj_cgroup_may_zswap(objcg)) {
		memcg = get_mem_cgroup_from_objcg(objcg);
		if (shrink_memcg(memcg)) {若memcg的zswap沾满，则将zswap里的数据解压出来，回写的swap磁盘分区
			mem_cgroup_put(memcg);
			goto put_objcg;
		}
		mem_cgroup_put(memcg);
	}

	if (zswap_check_limits())
		goto put_objcg;

	pool = zswap_pool_current_get();
	if (!pool)
		goto put_objcg;

	if (objcg) {
		memcg = get_mem_cgroup_from_objcg(objcg);
		if (memcg_list_lru_alloc(memcg, &zswap_list_lru, GFP_KERNEL)) {
			mem_cgroup_put(memcg);
			goto put_pool;
		}
		mem_cgroup_put(memcg);
	}

	for (index = 0; index < nr_pages; ++index) {
		struct page *page = folio_page(folio, index);

		if (!zswap_store_page(page, objcg, pool))
			goto put_pool;
	}

	if (objcg)
		count_objcg_events(objcg, ZSWPOUT, nr_pages);

	count_vm_events(ZSWPOUT, nr_pages);

	ret = true;

put_pool:
	zswap_pool_put(pool);
put_objcg:
	obj_cgroup_put(objcg);
	if (!ret && zswap_pool_reached_full)
		queue_work(shrink_wq, &zswap_shrink_work);
check_old:
	/*
	 * If the zswap store fails or zswap is disabled, we must invalidate
	 * the possibly stale entries which were previously stored at the
	 * offsets corresponding to each page of the folio. Otherwise,
	 * writeback could overwrite the new data in the swapfile.
	 */
	if (!ret) {
		unsigned type = swp_type(swp);
		pgoff_t offset = swp_offset(swp);
		struct zswap_entry *entry;
		struct xarray *tree;

		for (index = 0; index < nr_pages; ++index) {
			tree = swap_zswap_tree(swp_entry(type, offset + index));
			entry = xa_erase(tree, offset + index);
			if (entry)
				zswap_entry_free(entry);
		}
	}

	return ret;
}
```