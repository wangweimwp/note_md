```c
do_swap_page
    ->entry = pte_to_swp_entry(vmf->orig_pte);
    ->si = get_swap_device(entry);
    ->folio = swap_cache_get_folio(entry, vma, vmf->address);//现在swapcache中找
        ->filemap_get_folio//文件预读机制
    ->page = folio_file_page(folio, swp_offset(entry));//在swapcache中找到了
    ->folio = vma_alloc_folio(GFP_HIGHUSER_MOVABLE, 0,
						vma, vmf->address, false);    
    ->page = swapin_readahead(entry, GFP_HIGHUSER_MOVABLE,
						vmf);//在swapcache没有找到,从swap分区读数据
        
```





```c
/*
 * We enter with non-exclusive mmap_lock (to exclude vma changes,
 * but allow concurrent faults), and pte mapped but not yet locked.
 * We return with pte unmapped and unlocked.
 *
 * We return with the mmap_lock locked or unlocked in the same cases
 * as does filemap_fault().
 */
vm_fault_t do_swap_page(struct vm_fault *vmf)
{
	struct vm_area_struct *vma = vmf->vma;
	struct folio *swapcache, *folio = NULL;
	struct page *page;
	struct swap_info_struct *si = NULL;
	rmap_t rmap_flags = RMAP_NONE;
	bool need_clear_cache = false;
	bool exclusive = false;
	swp_entry_t entry;
	pte_t pte;
	vm_fault_t ret = 0;
	void *shadow = NULL;

	if (!pte_unmap_same(vmf))
		goto out;

	entry = pte_to_swp_entry(vmf->orig_pte);
	.....
    /*处理Unaddressable device memory support情况， 这里不做讨论
    Documentation/mm/hmm.rst*/
    .....
	/* Prevent swapoff from happening to us. */
	si = get_swap_device(entry);
	if (unlikely(!si))
		goto out;

	folio = swap_cache_get_folio(entry, vma, vmf->address);
	if (folio)
		page = folio_file_page(folio, swp_offset(entry));
	swapcache = folio;

	if (!folio) {
		if (data_race(si->flags & SWP_SYNCHRONOUS_IO) &&
		    __swap_count(entry) == 1) {//这是对zram等fastdevice的优化， 先跳过，后面会详细分析
			/*
			 * Prevent parallel swapin from proceeding with
			 * the cache flag. Otherwise, another thread may
			 * finish swapin first, free the entry, and swapout
			 * reusing the same entry. It's undetectable as
			 * pte_same() returns true due to entry reuse.
			 */
			if (swapcache_prepare(entry)) {
				/* Relax a bit to prevent rapid repeated page faults */
				schedule_timeout_uninterruptible(1);
				goto out;
			}
			need_clear_cache = true;

			/* skip swapcache */
			folio = vma_alloc_folio(GFP_HIGHUSER_MOVABLE, 0,
						vma, vmf->address, false);
			page = &folio->page;
			if (folio) {
				__folio_set_locked(folio);
				__folio_set_swapbacked(folio);

				if (mem_cgroup_swapin_charge_folio(folio,
							vma->vm_mm, GFP_KERNEL,
							entry)) {
					ret = VM_FAULT_OOM;
					goto out_page;
				}
				mem_cgroup_swapin_uncharge_swap(entry);

				shadow = get_shadow_from_swap_cache(entry);
				if (shadow)
					workingset_refault(folio, shadow);

				folio_add_lru(folio);

				/* To provide entry to swap_read_folio() */
				folio->swap = entry;
				swap_read_folio(folio, true, NULL);
				folio->private = NULL;
			}
		} else {
			page = swapin_readahead(entry, GFP_HIGHUSER_MOVABLE,
						vmf);
			if (page)
				folio = page_folio(page);
			swapcache = folio;
		}

		if (!folio) {
			/*
			 * Back out if somebody else faulted in this pte
			 * while we released the pte lock.
			 */
			vmf->pte = pte_offset_map_lock(vma->vm_mm, vmf->pmd,
					vmf->address, &vmf->ptl);
			if (likely(vmf->pte &&
				   pte_same(ptep_get(vmf->pte), vmf->orig_pte)))
				ret = VM_FAULT_OOM;
			goto unlock;
		}

		/* Had to read the page from swap area: Major fault */
		ret = VM_FAULT_MAJOR;
		count_vm_event(PGMAJFAULT);
		count_memcg_event_mm(vma->vm_mm, PGMAJFAULT);
	} else if (PageHWPoison(page)) {
		/*
		 * hwpoisoned dirty swapcache pages are kept for killing
		 * owner processes (which may be unknown at hwpoison time)
		 */
		ret = VM_FAULT_HWPOISON;
		goto out_release;
	}

	ret |= folio_lock_or_retry(folio, vmf);
	if (ret & VM_FAULT_RETRY)
		goto out_release;

	if (swapcache) {
		/*
		 * Make sure folio_free_swap() or swapoff did not release the
		 * swapcache from under us.  The page pin, and pte_same test
		 * below, are not enough to exclude that.  Even if it is still
		 * swapcache, we need to check that the page's swap has not
		 * changed.
		 */
		if (unlikely(!folio_test_swapcache(folio) ||
			     page_swap_entry(page).val != entry.val))
			goto out_page;

		/*
		 * KSM sometimes has to copy on read faults, for example, if
		 * page->index of !PageKSM() pages would be nonlinear inside the
		 * anon VMA -- PageKSM() is lost on actual swapout.
		 */
		folio = ksm_might_need_to_copy(folio, vma, vmf->address);
		if (unlikely(!folio)) {
			ret = VM_FAULT_OOM;
			folio = swapcache;
			goto out_page;
		} else if (unlikely(folio == ERR_PTR(-EHWPOISON))) {
			ret = VM_FAULT_HWPOISON;
			folio = swapcache;
			goto out_page;
		}
		if (folio != swapcache)
			page = folio_page(folio, 0);

		/*
		 * If we want to map a page that's in the swapcache writable, we
		 * have to detect via the refcount if we're really the exclusive
		 * owner. Try removing the extra reference from the local LRU
		 * caches if required.
		 */
		if ((vmf->flags & FAULT_FLAG_WRITE) && folio == swapcache &&
		    !folio_test_ksm(folio) && !folio_test_lru(folio))
			lru_add_drain();
	}

	folio_throttle_swaprate(folio, GFP_KERNEL);

	/*
	 * Back out if somebody else already faulted in this pte.
	 */
	vmf->pte = pte_offset_map_lock(vma->vm_mm, vmf->pmd, vmf->address,
			&vmf->ptl);
	if (unlikely(!vmf->pte || !pte_same(ptep_get(vmf->pte), vmf->orig_pte)))
		goto out_nomap;

	if (unlikely(!folio_test_uptodate(folio))) {
		ret = VM_FAULT_SIGBUS;
		goto out_nomap;
	}

	/*
	 * PG_anon_exclusive reuses PG_mappedtodisk for anon pages. A swap pte
	 * must never point at an anonymous page in the swapcache that is
	 * PG_anon_exclusive. Sanity check that this holds and especially, that
	 * no filesystem set PG_mappedtodisk on a page in the swapcache. Sanity
	 * check after taking the PT lock and making sure that nobody
	 * concurrently faulted in this page and set PG_anon_exclusive.
	 */
	BUG_ON(!folio_test_anon(folio) && folio_test_mappedtodisk(folio));
	BUG_ON(folio_test_anon(folio) && PageAnonExclusive(page));

	/*
	 * Check under PT lock (to protect against concurrent fork() sharing
	 * the swap entry concurrently) for certainly exclusive pages.
	 */
	if (!folio_test_ksm(folio)) {
		exclusive = pte_swp_exclusive(vmf->orig_pte);
		if (folio != swapcache) {
			/*
			 * We have a fresh page that is not exposed to the
			 * swapcache -> certainly exclusive.
			 */
			exclusive = true;
		} else if (exclusive && folio_test_writeback(folio) &&
			  data_race(si->flags & SWP_STABLE_WRITES)) {
			/*
			 * This is tricky: not all swap backends support
			 * concurrent page modifications while under writeback.
			 *
			 * So if we stumble over such a page in the swapcache
			 * we must not set the page exclusive, otherwise we can
			 * map it writable without further checks and modify it
			 * while still under writeback.
			 *
			 * For these problematic swap backends, simply drop the
			 * exclusive marker: this is perfectly fine as we start
			 * writeback only if we fully unmapped the page and
			 * there are no unexpected references on the page after
			 * unmapping succeeded. After fully unmapped, no
			 * further GUP references (FOLL_GET and FOLL_PIN) can
			 * appear, so dropping the exclusive marker and mapping
			 * it only R/O is fine.
			 */
			exclusive = false;
		}
	}

	/*
	 * Some architectures may have to restore extra metadata to the page
	 * when reading from swap. This metadata may be indexed by swap entry
	 * so this must be called before swap_free().
	 */
	arch_swap_restore(folio_swap(entry, folio), folio);

	/*
	 * Remove the swap entry and conditionally try to free up the swapcache.
	 * We're already holding a reference on the page but haven't mapped it
	 * yet.
	 */
	swap_free(entry);
	if (should_try_to_free_swap(folio, vma, vmf->flags))
		folio_free_swap(folio);

	inc_mm_counter(vma->vm_mm, MM_ANONPAGES);
	dec_mm_counter(vma->vm_mm, MM_SWAPENTS);
	pte = mk_pte(page, vma->vm_page_prot);

	/*
	 * Same logic as in do_wp_page(); however, optimize for pages that are
	 * certainly not shared either because we just allocated them without
	 * exposing them to the swapcache or because the swap entry indicates
	 * exclusivity.
	 */
	if (!folio_test_ksm(folio) &&
	    (exclusive || folio_ref_count(folio) == 1)) {
		if (vmf->flags & FAULT_FLAG_WRITE) {
			pte = maybe_mkwrite(pte_mkdirty(pte), vma);
			vmf->flags &= ~FAULT_FLAG_WRITE;
		}
		rmap_flags |= RMAP_EXCLUSIVE;
	}
	flush_icache_page(vma, page);
	if (pte_swp_soft_dirty(vmf->orig_pte))
		pte = pte_mksoft_dirty(pte);
	if (pte_swp_uffd_wp(vmf->orig_pte))
		pte = pte_mkuffd_wp(pte);
	vmf->orig_pte = pte;

	/* ksm created a completely new copy */
	if (unlikely(folio != swapcache && swapcache)) {
		folio_add_new_anon_rmap(folio, vma, vmf->address);
		folio_add_lru_vma(folio, vma);
	} else {
		folio_add_anon_rmap_pte(folio, page, vma, vmf->address,
					rmap_flags);
	}

	VM_BUG_ON(!folio_test_anon(folio) ||
			(pte_write(pte) && !PageAnonExclusive(page)));
	set_pte_at(vma->vm_mm, vmf->address, vmf->pte, pte);
	arch_do_swap_page(vma->vm_mm, vma, vmf->address, pte, vmf->orig_pte);

	folio_unlock(folio);
	if (folio != swapcache && swapcache) {
		/*
		 * Hold the lock to avoid the swap entry to be reused
		 * until we take the PT lock for the pte_same() check
		 * (to avoid false positives from pte_same). For
		 * further safety release the lock after the swap_free
		 * so that the swap count won't change under a
		 * parallel locked swapcache.
		 */
		folio_unlock(swapcache);
		folio_put(swapcache);
	}

	if (vmf->flags & FAULT_FLAG_WRITE) {
		ret |= do_wp_page(vmf);
		if (ret & VM_FAULT_ERROR)
			ret &= VM_FAULT_ERROR;
		goto out;
	}

	/* No need to invalidate - it was non-present before */
	update_mmu_cache_range(vmf, vma, vmf->address, vmf->pte, 1);
unlock:
	if (vmf->pte)
		pte_unmap_unlock(vmf->pte, vmf->ptl);
out:
	/* Clear the swap cache pin for direct swapin after PTL unlock */
	if (need_clear_cache)
		swapcache_clear(si, entry);
	if (si)
		put_swap_device(si);
	return ret;
out_nomap:
	if (vmf->pte)
		pte_unmap_unlock(vmf->pte, vmf->ptl);
out_page:
	folio_unlock(folio);
out_release:
	folio_put(folio);
	if (folio != swapcache && swapcache) {
		folio_unlock(swapcache);
		folio_put(swapcache);
	}
	if (need_clear_cache)
		swapcache_clear(si, entry);
	if (si)
		put_swap_device(si);
	return ret;
}
```


