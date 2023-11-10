零页合并解决zram+swap和KSM不能处理的页，

1，系统中存在经常读取，但不写入的页，不会swap出去

2，没有调用`madvise(addr, length, MADV_MERGEABLE)`的页无法合并

3，KSM合并需要时间长，先加入不稳定树，再加入稳定树才允许合并



**注意事项**

KSM只会处理通过madvise系统调用显示处理的用户进程空间内存，因此，用户想用这个功能，必须在分配内存后调用`madvise(addr, length, MADV_MERGEABLE)`,

调试路径`/sys/kernel/mm/ksm/`



**代码分析**

```c
/**
 * ksm_do_scan  - the ksm scanner main worker function.
 * @scan_npages:  number of pages we want to scan before we return.
 */
static void ksm_do_scan(unsigned int scan_npages)
{
	struct ksm_rmap_item *rmap_item;
	struct page *page;

	while (scan_npages-- && likely(!freezing(current))) {
		cond_resched();
		rmap_item = scan_get_next_rmap_item(&page);
		if (!rmap_item)
			return;
		cmp_and_merge_page(page, rmap_item);
		put_page(page);
	}
}


static struct ksm_rmap_item *scan_get_next_rmap_item(struct page **page)
{
	struct mm_struct *mm;
	struct ksm_mm_slot *mm_slot;
	struct mm_slot *slot;
	struct vm_area_struct *vma;
	struct ksm_rmap_item *rmap_item;
	struct vma_iterator vmi;
	int nid;

	if (list_empty(&ksm_mm_head.slot.mm_node))
		return NULL;

	mm_slot = ksm_scan.mm_slot;//当前扫描的struct ksm_mm_slot
	if (mm_slot == &ksm_mm_head) {
		/*
		 * A number of pages can hang around indefinitely on per-cpu
		 * pagevecs, raised page count preventing write_protect_page
		 * from merging them.  Though it doesn't really matter much,
		 * it is puzzling to see some stuck in pages_volatile until
		 * other activity jostles them out, and they also prevented
		 * LTP's KSM test from succeeding deterministically; so drain
		 * them here (here rather than on entry to ksm_do_scan(),
		 * so we don't IPI too often when pages_to_scan is set low).
		 */
		lru_add_drain_all();

		/*
		 * Whereas stale stable_nodes on the stable_tree itself
		 * get pruned in the regular course of stable_tree_search(),
		 * those moved out to the migrate_nodes list can accumulate:
		 * so prune them once before each full scan.
		 */
		if (!ksm_merge_across_nodes) {
			struct ksm_stable_node *stable_node, *next;
			struct page *page;

			list_for_each_entry_safe(stable_node, next,
						 &migrate_nodes, list) {
				page = get_ksm_page(stable_node,
						    GET_KSM_PAGE_NOLOCK);
				if (page)
					put_page(page);
				cond_resched();
			}
		}

		for (nid = 0; nid < ksm_nr_node_ids; nid++)
			root_unstable_tree[nid] = RB_ROOT;

		spin_lock(&ksm_mmlist_lock);
		slot = list_entry(mm_slot->slot.mm_node.next,
				  struct mm_slot, mm_node);//拿到下一个struct ksm_mm_slot里的struct mm_slot 
		mm_slot = mm_slot_entry(slot, struct ksm_mm_slot, slot);//通过struct mm_slot再拿到struct ksm_mm_slot
		ksm_scan.mm_slot = mm_slot;//设置为当前要扫描的
		spin_unlock(&ksm_mmlist_lock);
		/*
		 * Although we tested list_empty() above, a racing __ksm_exit
		 * of the last mm on the list may have removed it since then.
		 */
		if (mm_slot == &ksm_mm_head)//如果当前要扫描任然是ksm_mm_head，则说明列表是空的
			return NULL;
next_mm:
		ksm_scan.address = 0;
		ksm_scan.rmap_list = &mm_slot->rmap_list;
	}

	slot = &mm_slot->slot;
	mm = slot->mm;
	vma_iter_init(&vmi, mm, ksm_scan.address);

	mmap_read_lock(mm);
	if (ksm_test_exit(mm))
		goto no_vmas;

	for_each_vma(vmi, vma) {
		if (!(vma->vm_flags & VM_MERGEABLE))
			continue;
		if (ksm_scan.address < vma->vm_start)
			ksm_scan.address = vma->vm_start;
		if (!vma->anon_vma)
			ksm_scan.address = vma->vm_end;

		while (ksm_scan.address < vma->vm_end) {
			if (ksm_test_exit(mm))
				break;
			*page = follow_page(vma, ksm_scan.address, FOLL_GET);
			if (IS_ERR_OR_NULL(*page)) {
				ksm_scan.address += PAGE_SIZE;
				cond_resched();
				continue;
			}
			if (is_zone_device_page(*page))
				goto next_page;
			if (PageAnon(*page)) {
				flush_anon_page(vma, *page, ksm_scan.address);
				flush_dcache_page(*page);
				rmap_item = get_next_rmap_item(mm_slot,
					ksm_scan.rmap_list, ksm_scan.address);
				if (rmap_item) {
					ksm_scan.rmap_list =
							&rmap_item->rmap_list;
					ksm_scan.address += PAGE_SIZE;
				} else
					put_page(*page);
				mmap_read_unlock(mm);
				return rmap_item;
			}
next_page:
			put_page(*page);
			ksm_scan.address += PAGE_SIZE;
			cond_resched();
		}
	}

	if (ksm_test_exit(mm)) {
no_vmas:
		ksm_scan.address = 0;
		ksm_scan.rmap_list = &mm_slot->rmap_list;
	}
	/*
	 * Nuke all the rmap_items that are above this current rmap:
	 * because there were no VM_MERGEABLE vmas with such addresses.
	 */
	remove_trailing_rmap_items(ksm_scan.rmap_list);

	spin_lock(&ksm_mmlist_lock);
	slot = list_entry(mm_slot->slot.mm_node.next,
			  struct mm_slot, mm_node);
	ksm_scan.mm_slot = mm_slot_entry(slot, struct ksm_mm_slot, slot);
	if (ksm_scan.address == 0) {
		/*
		 * We've completed a full scan of all vmas, holding mmap_lock
		 * throughout, and found no VM_MERGEABLE: so do the same as
		 * __ksm_exit does to remove this mm from all our lists now.
		 * This applies either when cleaning up after __ksm_exit
		 * (but beware: we can reach here even before __ksm_exit),
		 * or when all VM_MERGEABLE areas have been unmapped (and
		 * mmap_lock then protects against race with MADV_MERGEABLE).
		 */
		hash_del(&mm_slot->slot.hash);
		list_del(&mm_slot->slot.mm_node);
		spin_unlock(&ksm_mmlist_lock);

		mm_slot_free(mm_slot_cache, mm_slot);
		clear_bit(MMF_VM_MERGEABLE, &mm->flags);
		mmap_read_unlock(mm);
		mmdrop(mm);
	} else {
		mmap_read_unlock(mm);
		/*
		 * mmap_read_unlock(mm) first because after
		 * spin_unlock(&ksm_mmlist_lock) run, the "mm" may
		 * already have been freed under us by __ksm_exit()
		 * because the "mm_slot" is still hashed and
		 * ksm_scan.mm_slot doesn't point to it anymore.
		 */
		spin_unlock(&ksm_mmlist_lock);
	}

	/* Repeat until we've completed scanning the whole list */
	mm_slot = ksm_scan.mm_slot;
	if (mm_slot != &ksm_mm_head)
		goto next_mm;

	ksm_scan.seqnr++;
	return NULL;
}

```


