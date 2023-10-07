`migrate_pages->migrate_pages_batch`



```c

/*
 * migrate_pages_batch() first unmaps folios in the from list as many as
 * possible, then move the unmapped folios.
 *
 * We only batch migration if mode == MIGRATE_ASYNC to avoid to wait a
 * lock or bit when we have locked more than one folio.  Which may cause
 * deadlock (e.g., for loop device).  So, if mode != MIGRATE_ASYNC, the
 * length of the from list must be <= 1.
	get_new_page申请新内存页的函数指针
	put_new_page迁移失败时，释放目标内存页的函数指针
	private 传递给get_new_page和put_new_page的参数
 */
static int migrate_pages_batch(struct list_head *from, new_page_t get_new_page,
		free_page_t put_new_page, unsigned long private,
		enum migrate_mode mode, int reason, struct list_head *ret_folios,
		struct list_head *split_folios, struct migrate_pages_stats *stats,
		int nr_pass)
{
	int retry = 1;
	int large_retry = 1;
	int thp_retry = 1;
	int nr_failed = 0;
	int nr_retry_pages = 0;
	int nr_large_failed = 0;
	int pass = 0;
	bool is_large = false;
	bool is_thp = false;
	struct folio *folio, *folio2, *dst = NULL, *dst2;
	int rc, rc_saved = 0, nr_pages;
	LIST_HEAD(unmap_folios);
	LIST_HEAD(dst_folios);
	bool nosplit = (reason == MR_NUMA_MISPLACED);

	VM_WARN_ON_ONCE(mode != MIGRATE_ASYNC &&
			!list_empty(from) && !list_is_singular(from));

	for (pass = 0; pass < nr_pass && (retry || large_retry); pass++) {
		retry = 0;
		large_retry = 0;
		thp_retry = 0;
		nr_retry_pages = 0;

		list_for_each_entry_safe(folio, folio2, from, lru) {
			/*
			 * Large folio statistics is based on the source large
			 * folio. Capture required information that might get
			 * lost during migration.
			 */
			is_large = folio_test_large(folio);
			is_thp = is_large && folio_test_pmd_mappable(folio);
			nr_pages = folio_nr_pages(folio);

			cond_resched();

			/*
			 * Large folio migration might be unsupported or
			 * the allocation might be failed so we should retry
			 * on the same folio with the large folio split
			 * to normal folios.
			 *
			 * Split folios are put in split_folios, and
			 * we will migrate them after the rest of the
			 * list is processed.
			 */
			if (!thp_migration_supported() && is_thp) {//是THP但是不支持THP
				nr_large_failed++;
				stats->nr_thp_failed++;
				if (!try_split_folio(folio, split_folios)) {//将THP分裂成普通页
					stats->nr_thp_split++;
					continue;
				}
				stats->nr_failed_pages += nr_pages;
				list_move_tail(&folio->lru, ret_folios);
				continue;
			}

			rc = migrate_folio_unmap(get_new_page, put_new_page, private,
						 folio, &dst, mode, reason, ret_folios);//解除PTE映射
			/*
			 * The rules are:
			 *	Success: folio will be freed
			 *	Unmap: folio will be put on unmap_folios list,
			 *	       dst folio put on dst_folios list
			 *	-EAGAIN: stay on the from list
			 *	-ENOMEM: stay on the from list
			 *	Other errno: put on ret_folios list
			 */
			switch(rc) {
			case -ENOMEM:
				/*
				 * When memory is low, don't bother to try to migrate
				 * other folios, move unmapped folios, then exit.
				 */
				if (is_large) {
					nr_large_failed++;
					stats->nr_thp_failed += is_thp;
					/* Large folio NUMA faulting doesn't split to retry. */
					if (!nosplit) {
						int ret = try_split_folio(folio, split_folios);

						if (!ret) {
							stats->nr_thp_split += is_thp;
							break;
						} else if (reason == MR_LONGTERM_PIN &&
							   ret == -EAGAIN) {
							/*
							 * Try again to split large folio to
							 * mitigate the failure of longterm pinning.
							 */
							large_retry++;
							thp_retry += is_thp;
							nr_retry_pages += nr_pages;
							break;
						}
					}
				} else {
					nr_failed++;
				}

				stats->nr_failed_pages += nr_pages + nr_retry_pages;
				/* nr_failed isn't updated for not used */
				nr_large_failed += large_retry;
				stats->nr_thp_failed += thp_retry;
				rc_saved = rc;
				if (list_empty(&unmap_folios))
					goto out;
				else
					goto move;
			case -EAGAIN:
				if (is_large) {
					large_retry++;
					thp_retry += is_thp;
				} else {
					retry++;
				}
				nr_retry_pages += nr_pages;
				break;
			case MIGRATEPAGE_SUCCESS://迁移可能会出现重试。有些页面在上次迁移时时已经成功被迁移了
				stats->nr_succeeded += nr_pages;
				stats->nr_thp_succeeded += is_thp;
				break;
			case MIGRATEPAGE_UNMAP://页面被成功解除映射，加入列表中等待迁移
				list_move_tail(&folio->lru, &unmap_folios);
				list_add_tail(&dst->lru, &dst_folios);
				break;
			default:
				/*
				 * Permanent failure (-EBUSY, etc.):
				 * unlike -EAGAIN case, the failed folio is
				 * removed from migration folio list and not
				 * retried in the next outer loop.
				 */
				if (is_large) {
					nr_large_failed++;
					stats->nr_thp_failed += is_thp;
				} else {
					nr_failed++;
				}

				stats->nr_failed_pages += nr_pages;
				break;
			}
		}
	}
	nr_failed += retry;
	nr_large_failed += large_retry;
	stats->nr_thp_failed += thp_retry;
	stats->nr_failed_pages += nr_retry_pages;
move:
	/* Flush TLBs for all unmapped folios */
	try_to_unmap_flush();//解除映射后刷新TLB

	retry = 1;
	for (pass = 0; pass < nr_pass && (retry || large_retry); pass++) {
		retry = 0;
		large_retry = 0;
		thp_retry = 0;
		nr_retry_pages = 0;

		dst = list_first_entry(&dst_folios, struct folio, lru);
		dst2 = list_next_entry(dst, lru);
		list_for_each_entry_safe(folio, folio2, &unmap_folios, lru) {
			is_large = folio_test_large(folio);
			is_thp = is_large && folio_test_pmd_mappable(folio);
			nr_pages = folio_nr_pages(folio);

			cond_resched();

			rc = migrate_folio_move(put_new_page, private,
						folio, dst, mode,
						reason, ret_folios);
			/*
			 * The rules are:
			 *	Success: folio will be freed
			 *	-EAGAIN: stay on the unmap_folios list
			 *	Other errno: put on ret_folios list
			 */
			switch(rc) {
			case -EAGAIN:
				if (is_large) {
					large_retry++;
					thp_retry += is_thp;
				} else {
					retry++;
				}
				nr_retry_pages += nr_pages;
				break;
			case MIGRATEPAGE_SUCCESS:
				stats->nr_succeeded += nr_pages;
				stats->nr_thp_succeeded += is_thp;
				break;
			default:
				if (is_large) {
					nr_large_failed++;
					stats->nr_thp_failed += is_thp;
				} else {
					nr_failed++;
				}

				stats->nr_failed_pages += nr_pages;
				break;
			}
			dst = dst2;
			dst2 = list_next_entry(dst, lru);
		}
	}
	nr_failed += retry;
	nr_large_failed += large_retry;
	stats->nr_thp_failed += thp_retry;
	stats->nr_failed_pages += nr_retry_pages;

	if (rc_saved)
		rc = rc_saved;
	else
		rc = nr_failed + nr_large_failed;
out:
	/* Cleanup remaining folios */
	dst = list_first_entry(&dst_folios, struct folio, lru);
	dst2 = list_next_entry(dst, lru);
	list_for_each_entry_safe(folio, folio2, &unmap_folios, lru) {
		int page_was_mapped = 0;
		struct anon_vma *anon_vma = NULL;

		__migrate_folio_extract(dst, &page_was_mapped, &anon_vma);
		migrate_folio_undo_src(folio, page_was_mapped, anon_vma,
				       true, ret_folios);
		list_del(&dst->lru);
		migrate_folio_undo_dst(dst, true, put_new_page, private);
		dst = dst2;
		dst2 = list_next_entry(dst, lru);
	}

	return rc;
}

migrate_folio_move
	->move_to_new_folio迁移到新的页面
	->remove_migration_ptes迁移PTE映射
	->migrate_folio_undo_src迁移失败后，恢复原有PTE映射
	->migrate_folio_undo_dst迁移失败后，将申请的目标页释放
/*
 * Move a page to a newly allocated page
 * The page is locked and all ptes have been successfully removed.
 *
 * The new page will have replaced the old page if this function
 * is successful.
 *
 * Return value:
 *   < 0 - error code
 *  MIGRATEPAGE_SUCCESS - success
 */
static int move_to_new_folio(struct folio *dst, struct folio *src,
				enum migrate_mode mode)
{
	int rc = -EAGAIN;
	bool is_lru = !__PageMovable(&src->page);

	VM_BUG_ON_FOLIO(!folio_test_locked(src), src);
	VM_BUG_ON_FOLIO(!folio_test_locked(dst), dst);

	if (likely(is_lru)) {
		struct address_space *mapping = folio_mapping(src);

		if (!mapping)//匿名页面或者slab页面
			rc = migrate_folio(mapping, dst, src, mode);
		else if (mapping->a_ops->migrate_folio)//其他一些特殊页面，文件页面或swap cache等
			/*
			 * Most folios have a mapping and most filesystems
			 * provide a migrate_folio callback. Anonymous folios
			 * are part of swap space which also has its own
			 * migrate_folio callback. This is the most common path
			 * for page migration.
			 */
			rc = mapping->a_ops->migrate_folio(mapping, dst, src,
								mode);
		else
			rc = fallback_migrate_folio(mapping, dst, src, mode);
	} else {
		const struct movable_operations *mops;

		/*对于非lru页面，可能会被释放，一般不需要迁移
		 * In case of non-lru page, it could be released after
		 * isolation step. In that case, we shouldn't try migration.
		 */
		VM_BUG_ON_FOLIO(!folio_test_isolated(src), src);
		if (!folio_test_movable(src)) {
			rc = MIGRATEPAGE_SUCCESS;
			folio_clear_isolated(src);
			goto out;
		}

		mops = folio_movable_ops(src);
		rc = mops->migrate_page(&dst->page, &src->page, mode);
		WARN_ON_ONCE(rc == MIGRATEPAGE_SUCCESS &&
				!folio_test_isolated(src));
	}

	/*
	 * When successful, old pagecache src->mapping must be cleared before
	 * src is freed; but stats require that PageAnon be left as PageAnon.
	 */
	if (rc == MIGRATEPAGE_SUCCESS) {
		if (__PageMovable(&src->page)) {
			VM_BUG_ON_FOLIO(!folio_test_isolated(src), src);

			/*
			 * We clear PG_movable under page_lock so any compactor
			 * cannot try to migrate this page.
			 */
			folio_clear_isolated(src);
		}

		/*
		 * Anonymous and movable src->mapping will be cleared by
		 * free_pages_prepare so don't reset it here for keeping
		 * the type to work PageAnon, for example.
		 */
		if (!folio_mapping_flags(src))
			src->mapping = NULL;

		if (likely(!folio_is_zone_device(dst)))
			flush_dcache_folio(dst);
	}
out:
	return rc;
}
```


