内核补丁`commit 34e58cac6d8f ("mm: workingset: let cache workingset challenge anon")`中提到`Reclaim can still decide how to balance pressure among the two LRUsdepending on the IO situation`，寻找如何平衡看补丁`mm: balance LRU lists based on relative thrashing`

在每次进行内存回收时候，会对每个LRU列表该扫描多收页面进行计算（`shrink_lruvec->get_scan_count`），计算原理是-如果页缓存(page cache)颠簸，则缓存页在内存中需要更多的时间，而匿名链表上可能有较冷的页，那么就要更多的扫描匿名页面。同样，如果换出的页出swapin，则表明我们回收匿名页过于激进，应该退回一点。

计算回收成本的数据结构时`strcut lruvec->cost_cost  和  strcut lruvec->file_cost`，

```c
//三个地方记录回收
shrink_inactive_list->lru_note_cost
shrink_active_list->lru_note_cost
workingset_refault->lru_note_cost_refault
```

**记录lru列表上的evictions和activations次数**

对于每个内存节点的LRU链表，维护一个计数器(`lruvec->nonresident_age`),当有page读入时，都有可能增加计数

```c


//页面被加入active列表
move_folios_to_lru
	->if (folio_test_active(folio))
			workingset_age_nonresident();
//页面被eviction时
shrink_folio_list			
  ->__remove_mapping
	    ->workingset_eviction
		    ->workingset_age_nonresident();
		
workingset_activation
	->workingset_age_nonresident

__read_swap_cache_async
	->workingset_refault
		->workingset_age_nonresident();
	
handle_pte_fault
	->do_swap_page
		->workingset_refault
			->workingset_age_nonresident();
	->swapin_readahead
		->swap_vma_readahead
			->__read_swap_cache_async
				->workingset_refault
					->workingset_age_nonresident();
					
filemap_add_folio
	->workingset_refault
		->workingset_age_nonresident();	

folio_mark_accessed
	->workingset_activation
		->workingset_age_nonresident();
```





**页面被eviction时**

```c
static int __remove_mapping(struct address_space *mapping, struct folio *folio,
			    bool reclaimed, struct mem_cgroup *target_memcg)
{
	....
	
	if (folio_test_swapcache(folio)) {//匿名页面已经分配了swapcache
		swp_entry_t swap = folio_swap_entry(folio);

		if (reclaimed && !mapping_exiting(mapping))
			shadow = workingset_eviction(folio, target_memcg);//获取的页面eviction时的lruvec->nonresident_age值
		__delete_from_swap_cache(folio, swap, shadow);//记录到文件xarray树中
		mem_cgroup_swapout(folio, swap);
		xa_unlock_irq(&mapping->i_pages);
		put_swap_folio(folio, swap);
	} else {
		void (*free_folio)(struct folio *);

		free_folio = mapping->a_ops->free_folio;
		/*
		 * Remember a shadow entry for reclaimed file cache in
		 * order to detect refaults, thus thrashing, later on.
		 *
		 * But don't store shadows in an address space that is
		 * already exiting.  This is not just an optimization,
		 * inode reclaim needs to empty out the radix tree or
		 * the nodes are lost.  Don't plant shadows behind its
		 * back.
		 *
		 * We also don't store shadows for DAX mappings because the
		 * only page cache folios found in these are zero pages
		 * covering holes, and because we don't want to mix DAX
		 * exceptional entries and shadow exceptional entries in the
		 * same address_space.
		 */
		if (reclaimed && folio_is_file_lru(folio) &&
		    !mapping_exiting(mapping) && !dax_mapping(mapping))
			shadow = workingset_eviction(folio, target_memcg);
		__filemap_remove_folio(folio, shadow);//与上边类似，文件页面被eviction出去之后，将shadow记录到xarray中，下次系统又访问这个页面时用于计算refault distance
		xa_unlock_irq(&mapping->i_pages);
		if (mapping_shrinkable(mapping))
			inode_add_lru(mapping->host);
		spin_unlock(&mapping->host->i_lock);

		if (free_folio)
			free_folio(folio);
	}

	....
}


void *workingset_eviction(struct folio *folio, struct mem_cgroup *target_memcg)
{
	struct pglist_data *pgdat = folio_pgdat(folio);
	unsigned long eviction;
	struct lruvec *lruvec;
	int memcgid;

	/* Folio is fully exclusive and pins folio's memory cgroup pointer */
	VM_BUG_ON_FOLIO(folio_test_lru(folio), folio);
	VM_BUG_ON_FOLIO(folio_ref_count(folio), folio);
	VM_BUG_ON_FOLIO(!folio_test_locked(folio), folio);

	if (lru_gen_enabled())
		return lru_gen_eviction(folio);

	lruvec = mem_cgroup_lruvec(target_memcg, pgdat);
	/* XXX: target_memcg can be NULL, go through lruvec */
	memcgid = mem_cgroup_id(lruvec_memcg(lruvec));
	eviction = atomic_long_read(&lruvec->nonresident_age);
	eviction >>= bucket_order;
	workingset_age_nonresident(lruvec, folio_nr_pages(folio));	//有页面发生eviction，记录
	return pack_shadow(memcgid, pgdat, eviction,
				folio_test_workingset(folio));	//计算folio的shadow值
}


```



**页面发生refault时**

```c
vm_fault_t do_swap_page(struct vm_fault *vmf)
{
	...
	folio = swap_cache_get_folio(entry, vma, vmf->address);//在swap中寻找该页
	if (folio)
		page = folio_file_page(folio, swp_offset(entry));
	swapcache = folio;

	if (!folio) {//如果swap cache中没有该页，那么说明已经回写到磁盘了
		...
		shadow = get_shadow_from_swap_cache(entry);
		if (shadow)
			workingset_refault(folio, shadow);//有shadow值，书名发生了refault，计算refault distance
		...
	}
	...	
}
```


