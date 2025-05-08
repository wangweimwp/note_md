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

	if (zswap_check_limits())//zswap占满
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

		if (!zswap_store_page(page, objcg, pool))//压缩page到zswap pool中
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
		queue_work(shrink_wq, &zswap_shrink_work);//zswap占满，启动工作队列进行后台回写，本质上调用shrink_memcg
check_old:
	/*
	 * If the zswap store fails or zswap is disabled, we must invalidate
	 * the possibly stale entries which were previously stored at the
	 * offsets corresponding to each page of the folio. Otherwise,
	 * writeback could overwrite the new data in the swapfile.
	 之前可能有一些页面被存储在 zswap 中，对应的交换缓存条目(swap cache entries)仍然存在。
		这些条目现在可能已经"陈旧"(stale)，因为：
		zswap 存储失败导致实际数据未正确保存
		zswap 被禁用后这些缓存不再有效
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


zswap_store_page
	->zswap_entry_cache_alloc
	->zswap_compress
	->xa_store	//存储到xarray树中
	->zswap_lru_add(&zswap_list_lru, entry); //添加到zswap_list_lru列表上
	
shrink_memcg
	->list_lru_walk_one(&zswap_list_lru, nid, memcg,
			&shrink_memcg_cb, NULL, &nr_to_walk); //遍历zswap_list_lru上的zswap_entry，调用shrink_memcg_cb

/*
动态收缩器的调节机制基于以下因素：

1,引用位二次机会机制
每个zswap条目都设有引用标记位。收缩器首先会清除该标记位（给予条目二次机会）并将其在LRU列表中轮转。若该条目再次被收缩器扫描且引用位仍未被置位，则执行回写操作。这种机制使得回写速率能根据存储池活动动态调整：当池中多为新条目（即近期大量zswap换出）时，这些活跃条目会受到保护，回写速率降低；反之，若池中存在大量陈旧条目，则会立即回收这些条目，实质上提高了回写速率。

2,交换入计数器反馈
当监测到交换入(swapin)操作时，表明当前收缩过度，应降低回收强度。系统维护一个交换入计数器，该计数器会在zswap_shrinker_count()函数中消耗，并相应减少LRU链表中符合回收条件的对象数量。

3,压缩比感知调节
工作负载的压缩效率越高，通过回写获得的收益就越小。系统会根据压缩比例动态缩放可供回收的对象数量——压缩比越好，可回收对象数越少。
*/

static enum lru_status shrink_memcg_cb(struct list_head *item, struct list_lru_one *l,
				       void *arg)
{
	struct zswap_entry *entry = container_of(item, struct zswap_entry, lru);
	bool *encountered_page_in_swapcache = (bool *)arg;
	swp_entry_t swpentry;
	enum lru_status ret = LRU_REMOVED_RETRY;
	int writeback_result;

	/*
	 * Second chance algorithm: if the entry has its referenced bit set, give it
	 * a second chance. Only clear the referenced bit and rotate it in the
	 * zswap's LRU list.
	 */
	if (entry->referenced) {
		entry->referenced = false;
		return LRU_ROTATE;
	}

	/*
	 当释放LRU锁后，条目可能被并发执行的失效操作释放。这意味着：

	1,交换条目提取与验证机制
	我们将swp_entry_t提取到栈内存中，使zswap_writeback_entry()能固定该交换条目，随后通过指针值比较来验证zswap条目与交换条目树的匹配性。只有验证成功后，才能解引用该条目。

	2,LRU轮转保护策略
	常规情况下，对象会从LRU移除进行回收。但此处不可行，因为若回收失败（无论何种原因），我们无法判断条目是否存活以将其重新放回LRU。

	因此需在释放锁前执行轮转操作：

		若条目被成功回写或失效，释放路径会自动解除其链接
		对于失败情况，轮转同样是正确选择
		临时性失败（需立即重试同一条目）在该收缩器中几乎不会发生：
			不采用任何尝试锁机制
			-ENOMEM是最接近的案例，但极为罕见且不会伪触发
			无需特别区分此类情况
	 */
	list_move_tail(item, &l->list);

	/*
		一旦释放了LRU锁，该条目可能会被释放。此时会将交换条目（swpentry）复制到栈上，并且在确认该条目在树中仍然存活之前，不会再次对其进行解引用操作。
	 */
	swpentry = entry->swpentry;

	/*
	 * It's safe to drop the lock here because we return either
	 * LRU_REMOVED_RETRY, LRU_RETRY or LRU_STOP.
	 */
	spin_unlock(&l->lock);

	writeback_result = zswap_writeback_entry(entry, swpentry);

	if (writeback_result) {
		zswap_reject_reclaim_fail++;
		ret = LRU_RETRY;

		/*
		 * Encountering a page already in swap cache is a sign that we are shrinking
		 * into the warmer region. We should terminate shrinking (if we're in the dynamic
		 * shrinker context).
		 */
		if (writeback_result == -EEXIST && encountered_page_in_swapcache) {
			ret = LRU_STOP;
			*encountered_page_in_swapcache = true;
		}
	} else {
		zswap_written_back_pages++;
	}

	return ret;
}
```