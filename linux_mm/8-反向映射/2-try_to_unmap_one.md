```c

/*
 * @arg: enum ttu_flags will be passed to this argument
 */
static bool try_to_unmap_one(struct folio *folio, struct vm_area_struct *vma,
		     unsigned long address, void *arg)
{
	struct mm_struct *mm = vma->vm_mm;
	DEFINE_FOLIO_VMA_WALK(pvmw, folio, vma, address, 0);
	pte_t pteval;
	struct page *subpage;
	bool anon_exclusive, ret = true;
	struct mmu_notifier_range range;
	enum ttu_flags flags = (enum ttu_flags)(long)arg;

	/*
	 * When racing against e.g. zap_pte_range() on another cpu,
	 * in between its ptep_get_and_clear_full() and page_remove_rmap(),
	 * try_to_unmap() may return before page_mapped() has become false,
	 * if page table locking is skipped: use TTU_SYNC to wait for that.
	 */
	if (flags & TTU_SYNC)
		pvmw.flags = PVMW_SYNC;

	if (flags & TTU_SPLIT_HUGE_PMD)//若有需要先将THP分裂
		split_huge_pmd_address(vma, address, false, folio);

	/*
	 * For THP, we have to assume the worse case ie pmd for invalidation.
	 * For hugetlb, it could be much worse if we need to do pud
	 * invalidation in the case of pmd sharing.
	 *
	 * Note that the folio can not be freed in this function as call of
	 * try_to_unmap() must hold a reference on the folio.
	 */
	range.end = vma_address_end(&pvmw);
	mmu_notifier_range_init(&range, MMU_NOTIFY_CLEAR, 0, vma->vm_mm,
				address, range.end);
	if (folio_test_hugetlb(folio)) {
		/*
		 * If sharing is possible, start and end will be adjusted
		 * accordingly. 如果是共享hugetlb，则调整start和end地址
		 */
		adjust_range_if_pmd_sharing_possible(vma, &range.start,
						     &range.end);
	}
	mmu_notifier_invalidate_range_start(&range);

	while (page_vma_mapped_walk(&pvmw)) {//检查VMA有没有映射这个page，并获取pte entry，循环迭代filio中的所有page，对每个page解除映射
		/* Unexpected PMD-mapped THP? */
		VM_BUG_ON_FOLIO(!pvmw.pte, folio);

		/*如果vma被malock住，则直接退出循环
		 * If the folio is in an mlock()d vma, we must not swap it out.
		 */
		if (!(flags & TTU_IGNORE_MLOCK) &&
		    (vma->vm_flags & VM_LOCKED)) {
			/* Restore the mlock which got missed */
			mlock_vma_folio(folio, vma, false);
			page_vma_mapped_walk_done(&pvmw);
			ret = false;
			break;
		}

		subpage = folio_page(folio,
					pte_pfn(*pvmw.pte) - folio_pfn(folio));
		address = pvmw.address;
		anon_exclusive = folio_test_anon(folio) &&
				 PageAnonExclusive(subpage);//见page_move_anon_rmap和__page_set_anon_rmap设置PG_anon_exclusive位

		if (folio_test_hugetlb(folio)) {
			bool anon = folio_test_anon(folio);

			/*
			 * The try_to_unmap() is only passed a hugetlb page
			 * in the case where the hugetlb page is poisoned.
			 */
			VM_BUG_ON_PAGE(!PageHWPoison(subpage), subpage);
			/*
			 * huge_pmd_unshare may unmap an entire PMD page.
			 * There is no way of knowing exactly which PMDs may
			 * be cached for this mm, so we must flush them all.
			 * start/end were already adjusted above to cover this
			 * range.
			 */
			flush_cache_range(vma, range.start, range.end);

			/*
			 * To call huge_pmd_unshare, i_mmap_rwsem must be
			 * held in write mode.  Caller needs to explicitly
			 * do this outside rmap routines.
			 *
			 * We also must hold hugetlb vma_lock in write mode.
			 * Lock order dictates acquiring vma_lock BEFORE
			 * i_mmap_rwsem.  We can only try lock here and fail
			 * if unsuccessful.
			 */
			if (!anon) {
				VM_BUG_ON(!(flags & TTU_RMAP_LOCKED));
				if (!hugetlb_vma_trylock_write(vma)) {
					page_vma_mapped_walk_done(&pvmw);
					ret = false;
					break;
				}
				if (huge_pmd_unshare(mm, vma, address, pvmw.pte)) {
					hugetlb_vma_unlock_write(vma);
					flush_tlb_range(vma,
						range.start, range.end);
					mmu_notifier_invalidate_range(mm,
						range.start, range.end);
					/*
					 * The ref count of the PMD page was
					 * dropped which is part of the way map
					 * counting is done for shared PMDs.
					 * Return 'true' here.  When there is
					 * no other sharing, huge_pmd_unshare
					 * returns false and we will unmap the
					 * actual page and drop map count
					 * to zero.
					 */
					page_vma_mapped_walk_done(&pvmw);
					break;
				}
				hugetlb_vma_unlock_write(vma);
			}
			pteval = huge_ptep_clear_flush(vma, address, pvmw.pte);//获取pte entry，保存到pteval，清空pte entry
		} else {//普通页情况
			flush_cache_page(vma, address, pte_pfn(*pvmw.pte));
			/* Nuke the page table entry. */
			if (should_defer_flush(mm, flags)) {//获取pte entry，保存到pteval，清空pte entry
				/*
				 * We clear the PTE but do not flush so potentially
				 * a remote CPU could still be writing to the folio.
				 * If the entry was previously clean then the
				 * architecture must guarantee that a clear->dirty
				 * transition on a cached TLB entry is written through
				 * and traps if the PTE is unmapped.
				 */
				pteval = ptep_get_and_clear(mm, address, pvmw.pte);

				set_tlb_ubc_flush_pending(mm, pte_dirty(pteval));
			} else {
				pteval = ptep_clear_flush(vma, address, pvmw.pte);
			}
		}
/**************上文分别对THP映射进行分裂，而后对hugetlb和普通页进行清空PTE ENTRY，******************************/
/**************下文进行swp entry的制作，并将其写回pte页表里，将页面主要分为  坏页（毒页） 匿名页  文件页 ******************************/		/*
		 * Now the pte is cleared. If this pte was uffd-wp armed,
		 * we may want to replace a none pte with a marker pte if
		 * it's file-backed, so we don't lose the tracking info.
		 */
		pte_install_uffd_wp_if_needed(vma, address, pvmw.pte, pteval);//在pte entry中写入PTE_MARKER_UFFD_WP,而不是none

		/* Set the dirty flag on the folio now the pte is gone. */
		if (pte_dirty(pteval))
			folio_mark_dirty(folio);

		/* Update high watermark before we lower rss */
		update_hiwater_rss(mm);// 更新进程所拥有的最大页框数 
		/* 此页是被标记为"坏页（毒页）"的页，这种页用于内核纠正一些错误，是否用于边界检查? */
		if (PageHWPoison(subpage) && (flags & TTU_HWPOISON)) {
			pteval = swp_entry_to_pte(make_hwpoison_entry(subpage));//制作swp PTE entry
			if (folio_test_hugetlb(folio)) {
				hugetlb_count_sub(folio_nr_pages(folio), mm);
				set_huge_pte_at(mm, address, pvmw.pte, pteval);
			} else {
				dec_mm_counter(mm, mm_counter(&folio->page));
				set_pte_at(mm, address, pvmw.pte, pteval);//向制作的swp PTE entry写入到pte entry中
			}

		} else if (pte_unused(pteval) && !userfaultfd_armed(vma)) {
			/*
			 * The guest indicated that the page content is of no
			 * interest anymore. Simply discard the pte, vmscan
			 * will take care of the rest.
			 * A future reference will then fault in a new zero
			 * page. When userfaultfd is active, we must not drop
			 * this page though, as its main user (postcopy
			 * migration) will not expect userfaults on already
			 * copied pages.
			 */
			dec_mm_counter(mm, mm_counter(&folio->page));
			/* We have to invalidate as we cleared the pte */
			mmu_notifier_invalidate_range(mm, address,
						      address + PAGE_SIZE);
		} else if (folio_test_anon(folio)) {//匿名页面
			/* 获取page->private中保存的内容，调用到try_to_unmap()前会把此页加入到swapcache，然后分配一个以swap页槽偏移量为内容的swp_entry_t */
			swp_entry_t entry = { .val = page_private(subpage) };
			pte_t swp_pte;
			/*
			 * Store the swap location in the pte.
			 * See handle_pte_fault() ...
			 */
			if (unlikely(folio_test_swapbacked(folio) !=
					folio_test_swapcache(folio))) {
				WARN_ON_ONCE(1);
				ret = false;
				/* We have to invalidate as we cleared the pte */
				mmu_notifier_invalidate_range(mm, address,
							address + PAGE_SIZE);
				page_vma_mapped_walk_done(&pvmw);
				break;
			}

			/* MADV_FREE page check */
			if (!folio_test_swapbacked(folio)) {
				int ref_count, map_count;

				/*
				 * Synchronize with gup_pte_range():
				 * - clear PTE; barrier; read refcount
				 * - inc refcount; barrier; read PTE
				 */
				smp_mb();

				ref_count = folio_ref_count(folio);
				map_count = folio_mapcount(folio);

				/*
				 * Order reads for page refcount and dirty flag
				 * (see comments in __remove_mapping()).
				 */
				smp_rmb();

				/*
				 * The only page refs must be one from isolation
				 * plus the rmap(s) (dropped by discard:).
				 */
				if (ref_count == 1 + map_count &&
				    !folio_test_dirty(folio)) {//该filio其他地方没有在使用，这跳转到discard。直接_mapcount-1
					/* Invalidate as we cleared the pte */
					mmu_notifier_invalidate_range(mm,
						address, address + PAGE_SIZE);
					dec_mm_counter(mm, MM_ANONPAGES);
					goto discard;
				}

				/*这个filio被再次dirty了，不能被丢弃，恢复映射
				 * If the folio was redirtied, it cannot be
				 * discarded. Remap the page to page table.
				 */
				set_pte_at(mm, address, pvmw.pte, pteval);
				folio_set_swapbacked(folio);
				ret = false;
				page_vma_mapped_walk_done(&pvmw);
				break;
			}
			/*看这个swp entry是否可用，并且增加entry对应页槽在swap_info_struct的swap_map的数值，
			 *此数值标记此页槽的页有多少个进程引用
			 *对于THP分裂后的页表，并没有设置swp_entry_t，swap_duplicate返回0  */
			if (swap_duplicate(entry) < 0) {
				set_pte_at(mm, address, pvmw.pte, pteval);
				ret = false;
				page_vma_mapped_walk_done(&pvmw);
				break;
			}
			if (arch_unmap_one(mm, vma, address, pteval) < 0) {//体系架构是都支持UNMAP_ONE
				swap_free(entry);
				set_pte_at(mm, address, pvmw.pte, pteval);
				ret = false;
				page_vma_mapped_walk_done(&pvmw);
				break;
			}

			/* See page_try_share_anon_rmap(): clear PTE first. */
			if (anon_exclusive &&
			    page_try_share_anon_rmap(subpage)) {//临时清除PG_anon_exclusive，为什么这样做？？？
				swap_free(entry);
				set_pte_at(mm, address, pvmw.pte, pteval);//清除失败，恢复映射
				ret = false;
				page_vma_mapped_walk_done(&pvmw);
				break;
			}
			/* entry有效，并且swap_map中目标页槽的数值也++了
             	 * 这个if的情况是此vma所属进程的mm没有加入到所有进程的mmlist中(init_mm.mmlist) 
			 */
			if (list_empty(&mm->mmlist)) {
				spin_lock(&mmlist_lock);
				if (list_empty(&mm->mmlist))
					list_add(&mm->mmlist, &init_mm.mmlist);
				spin_unlock(&mmlist_lock);
			}
			dec_mm_counter(mm, MM_ANONPAGES);
			inc_mm_counter(mm, MM_SWAPENTS);
			swp_pte = swp_entry_to_pte(entry);
			if (anon_exclusive)
				swp_pte = pte_swp_mkexclusive(swp_pte);
			if (pte_soft_dirty(pteval))
				swp_pte = pte_swp_mksoft_dirty(swp_pte);
			if (pte_uffd_wp(pteval))
				swp_pte = pte_swp_mkuffd_wp(swp_pte);
			set_pte_at(mm, address, pvmw.pte, swp_pte);//制作swp entry 并写入到pte entry中
			/* Invalidate as we cleared the pte */
			mmu_notifier_invalidate_range(mm, address,
						      address + PAGE_SIZE);
		} else {//文件页或共享页
			/*
			 * This is a locked file-backed folio,
			 * so it cannot be removed from the page
			 * cache and replaced by a new folio before
			 * mmu_notifier_invalidate_range_end, so no
			 * concurrent thread might update its page table
			 * to point at a new folio while a device is
			 * still using this folio.
			 *
			 * See Documentation/mm/mmu_notifier.rst
			 */
			dec_mm_counter(mm, mm_counter_file(&folio->page));
		}
discard:
		/*
		 * No need to call mmu_notifier_invalidate_range() it has be
		 * done above for all cases requiring it to happen under page
		 * table lock before mmu_notifier_invalidate_range_end()
		 *
		 * See Documentation/mm/mmu_notifier.rst
		 */ /* 对此页的页描述符的_mapcount进行--操作，当_mapcount为-1时，表示此页已经没有页表项映射了 */
		page_remove_rmap(subpage, vma, folio_test_hugetlb(folio));
		if (vma->vm_flags & VM_LOCKED)
			mlock_drain_local();
		folio_put(folio);
	}

	mmu_notifier_invalidate_range_end(&range);

	return ret;
}
```
