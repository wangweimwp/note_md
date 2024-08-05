```c
static void walk_mm(struct lruvec *lruvec, struct mm_struct *mm, struct lru_gen_mm_walk *walk)
{
	static const struct mm_walk_ops mm_walk_ops = {
		.test_walk = should_skip_vma,
		.p4d_entry = walk_pud_range,
	};

	int err;
	struct mem_cgroup *memcg = lruvec_memcg(lruvec);

	walk->next_addr = FIRST_USER_ADDRESS;

	do {
		err = -EBUSY;

		/* folio_update_gen() requires stable folio_memcg() */
		if (!mem_cgroup_trylock_pages(memcg))
			break;

		/* the caller might be holding the lock for write */
		if (mmap_read_trylock(mm)) {
			err = walk_page_range(mm, walk->next_addr, ULONG_MAX, &mm_walk_ops, walk);

			mmap_read_unlock(mm);
		}

		mem_cgroup_unlock_pages();

		if (walk->batched) {
			spin_lock_irq(&lruvec->lru_lock);
			reset_batch_size(lruvec, walk);
			spin_unlock_irq(&lruvec->lru_lock);
		}

		cond_resched();
	} while (err == -EAGAIN);
}

static int walk_pud_range(p4d_t *p4d, unsigned long start, unsigned long end,
			  struct mm_walk *args)
{
	int i;
	pud_t *pud;
	unsigned long addr;
	unsigned long next;
	struct lru_gen_mm_walk *walk = args->private;

	VM_WARN_ON_ONCE(p4d_leaf(*p4d));

	pud = pud_offset(p4d, start & P4D_MASK);
restart:
	for (i = pud_index(start), addr = start; addr != end; i++, addr = next) {
		pud_t val = READ_ONCE(pud[i]);

		next = pud_addr_end(addr, end);

		if (!pud_present(val) || WARN_ON_ONCE(pud_leaf(val)))
			continue;

		walk_pmd_range(&val, addr, next, args);

		/* a racy check to curtail the waiting time */
		if (wq_has_sleeper(&walk->lruvec->mm_state.wait))
			return 1;

		if (need_resched() || walk->batched >= MAX_LRU_BATCH) {
			end = (addr | ~PUD_MASK) + 1;
			goto done;
		}
	}

	if (i < PTRS_PER_PUD && get_next_vma(P4D_MASK, PUD_SIZE, args, &start, &end))
		goto restart;

	end = round_up(end, P4D_SIZE);
done:
	if (!end || !args->vma)
		return 1;

	walk->next_addr = max(end, args->vma->vm_start);

	return -EAGAIN;
}


static void walk_pmd_range(pud_t *pud, unsigned long start, unsigned long end,
			   struct mm_walk *args)
{
	int i;
	pmd_t *pmd;
	unsigned long next;
	unsigned long addr;
	struct vm_area_struct *vma;
	unsigned long pos = -1;
	struct lru_gen_mm_walk *walk = args->private;
	unsigned long bitmap[BITS_TO_LONGS(MIN_LRU_BATCH)] = {};

	VM_WARN_ON_ONCE(pud_leaf(*pud));

	/*
	 * Finish an entire PMD in two passes: the first only reaches to PTE
	 * tables to avoid taking the PMD lock; the second, if necessary, takes
	 * the PMD lock to clear the accessed bit in PMD entries.
	 */
	pmd = pmd_offset(pud, start & PUD_MASK);
restart:
	/* walk_pte_range() may call get_next_vma() */
	vma = args->vma;
	for (i = pmd_index(start), addr = start; addr != end; i++, addr = next) {
		pmd_t val = pmdp_get_lockless(pmd + i);

		next = pmd_addr_end(addr, end);

		if (!pmd_present(val) || is_huge_zero_pmd(val)) {
			walk->mm_stats[MM_LEAF_TOTAL]++;//pmd页表项为空，叶子页表项+1
			continue;
		}

#ifdef CONFIG_TRANSPARENT_HUGEPAGE
		if (pmd_trans_huge(val)) {
			unsigned long pfn = pmd_pfn(val);
			struct pglist_data *pgdat = lruvec_pgdat(walk->lruvec);

			walk->mm_stats[MM_LEAF_TOTAL]++;

			if (!pmd_young(val)) {
				walk->mm_stats[MM_LEAF_OLD]++;
				continue;
			}

			/* try to avoid unnecessary memory loads */
			if (pfn < pgdat->node_start_pfn || pfn >= pgdat_end_pfn(pgdat))
				continue;

			walk_pmd_range_locked(pud, addr, vma, args, bitmap, &pos);
			continue;
		}
#endif
		walk->mm_stats[MM_NONLEAF_TOTAL]++;//非叶子页表+1

		if (arch_has_hw_nonleaf_pmd_young() &&
		    get_cap(LRU_GEN_NONLEAF_YOUNG)) {
			if (!pmd_young(val))
				continue;

			walk_pmd_range_locked(pud, addr, vma, args, bitmap, &pos);
		}

		if (!walk->force_scan && !test_bloom_filter(walk->lruvec, walk->max_seq, pmd + i))
			continue;//不是强制扫描，并且bloom_filter中没有

		walk->mm_stats[MM_NONLEAF_FOUND]++;//在boom_filter中找到该PMD

		if (!walk_pte_range(&val, addr, next, args))//遍历PTE，清楚PTE的young标记，更新walk->mm_stats计数，若平均每个cache_line的youngPTE的个数>=一个，则返回true
			continue;

		walk->mm_stats[MM_NONLEAF_ADDED]++;	//加入到bloomfilter的叶子页表项+1

		/* carry over to the next generation */
		update_bloom_filter(walk->lruvec, walk->max_seq + 1, pmd + i);//将当前遍历的PMD延续到下一代bloomfilter中
	}

	walk_pmd_range_locked(pud, -1, vma, args, bitmap, &pos);

	if (i < PTRS_PER_PMD && get_next_vma(PUD_MASK, PMD_SIZE, args, &start, &end))
		goto restart;
}



static bool walk_pte_range(pmd_t *pmd, unsigned long start, unsigned long end,
			   struct mm_walk *args)
{
	int i;
	pte_t *pte;
	spinlock_t *ptl;
	unsigned long addr;
	int total = 0;
	int young = 0;
	struct lru_gen_mm_walk *walk = args->private;
	struct mem_cgroup *memcg = lruvec_memcg(walk->lruvec);
	struct pglist_data *pgdat = lruvec_pgdat(walk->lruvec);
	int old_gen, new_gen = lru_gen_from_seq(walk->max_seq);

	VM_WARN_ON_ONCE(pmd_leaf(*pmd));

	ptl = pte_lockptr(args->mm, pmd);
	if (!spin_trylock(ptl))
		return false;

	arch_enter_lazy_mmu_mode();

	pte = pte_offset_map(pmd, start & PMD_MASK);
restart:
	for (i = pte_index(start), addr = start; addr != end; i++, addr += PAGE_SIZE) {
		unsigned long pfn;
		struct folio *folio;

		total++;
		walk->mm_stats[MM_LEAF_TOTAL]++;		//叶子页表项+1

		pfn = get_pte_pfn(pte[i], args->vma, addr);
		if (pfn == -1)
			continue;

		if (!pte_young(pte[i])) {
			walk->mm_stats[MM_LEAF_OLD]++;//该PTE不为young
			continue;
		}

		folio = get_pfn_folio(pfn, memcg, pgdat, walk->can_swap);
		if (!folio)
			continue;

		if (!ptep_test_and_clear_young(args->vma, addr, pte + i))//pte为young，清除young标记
			VM_WARN_ON_ONCE(true);

		young++;
		walk->mm_stats[MM_LEAF_YOUNG]++; //young页表项+1

		if (pte_dirty(pte[i]) && !folio_test_dirty(folio) &&
		    !(folio_test_anon(folio) && folio_test_swapbacked(folio) &&
		      !folio_test_swapcache(folio))) //pte为脏&&对应的folio不为脏&&不为匿名folio && folio有swapbacked && filio没有swapcache
			folio_mark_dirty(folio);

		old_gen = folio_update_gen(folio, new_gen);//给filio升到当前最年轻一代，这里只更新folio的flag，没用将folio放到新的lrugen->list上，后边在evict_folios->isolate_folios->scan_folios->sort_folio中，if (gen != lru_gen_from_seq(lrugen->min_seq[type]))成立，移动到新的lrugen->list上，还有inc_min_seq中也会移动
		if (old_gen >= 0 && old_gen != new_gen)
			update_batch_size(walk, folio, old_gen, new_gen);//更新walk->nr_pages
	}

	if (i < PTRS_PER_PTE && get_next_vma(PMD_MASK, PAGE_SIZE, args, &start, &end))
		goto restart;

	pte_unmap(pte);	//CONFIG_HIGHPTE=y才会去unmap

	arch_leave_lazy_mmu_mode();
	spin_unlock(ptl);

	return suitable_to_scan(total, young);
}
```


