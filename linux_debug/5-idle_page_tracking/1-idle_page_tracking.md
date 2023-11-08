**原理**

通过`PG_idel`和`PG_young`对页面进行追踪





**代码分析**

```c
static ssize_t page_idle_bitmap_read(struct file *file, struct kobject *kobj,
				     struct bin_attribute *attr, char *buf,
				     loff_t pos, size_t count)
{
	u64 *out = (u64 *)buf;
	struct folio *folio;
	unsigned long pfn, end_pfn;
	int bit;

	if (pos % BITMAP_CHUNK_SIZE || count % BITMAP_CHUNK_SIZE)
		return -EINVAL;

	pfn = pos * BITS_PER_BYTE;
	if (pfn >= max_pfn)
		return 0;

	end_pfn = pfn + count * BITS_PER_BYTE;
	if (end_pfn > max_pfn)
		end_pfn = max_pfn;

	for (; pfn < end_pfn; pfn++) {
		bit = pfn % BITMAP_CHUNK_BITS;
		if (!bit)
			*out = 0ULL;
		folio = page_idle_get_folio(pfn);
		if (folio) {
			if (folio_test_idle(folio)) {//注意64位和32位系统实现不同
				/*
				 * The page might have been referenced via a
				 * pte, in which case it is not idle. Clear
				 * refs and recheck.
                 * 该页被访问后PTE的accessed bit被硬件置1，但idle标记不会被清除
				 */
				page_idle_clear_pte_refs(folio);
				if (folio_test_idle(folio))
					*out |= 1ULL << bit;
			}
			folio_put(folio);
		}
		if (bit == BITMAP_CHUNK_BITS - 1)
			out++;
		cond_resched();
	}
	return (char *)out - buf;
}


static void page_idle_clear_pte_refs(struct folio *folio)
{
	/* 通过方向映射机制遍历每个映射这个页的PTE，看这个页最近是都被访问
	 * Since rwc.try_lock is unused, rwc is effectively immutable, so we
	 * can make it static to save some cycles and stack.
	 */
	static struct rmap_walk_control rwc = {
		.rmap_one = page_idle_clear_pte_refs_one,
		.anon_lock = folio_lock_anon_vma_read,
	};
	bool need_lock;

	if (!folio_mapped(folio) || !folio_raw_mapping(folio))
		return;

	need_lock = !folio_test_anon(folio) || folio_test_ksm(folio);
	if (need_lock && !folio_trylock(folio))
		return;

	rmap_walk(folio, &rwc);

	if (need_lock)
		folio_unlock(folio);
}

static bool page_idle_clear_pte_refs_one(struct folio *folio,
					struct vm_area_struct *vma,
					unsigned long addr, void *arg)
{
	DEFINE_FOLIO_VMA_WALK(pvmw, folio, vma, addr, 0);
	bool referenced = false;

	while (page_vma_mapped_walk(&pvmw)) {
		addr = pvmw.address;
		if (pvmw.pte) {
			/*
			 * For PTE-mapped THP, one sub page is referenced,
			 * the whole THP is referenced.
			 */
			if (ptep_clear_young_notify(vma, addr, pvmw.pte))
				referenced = true;
		} else if (IS_ENABLED(CONFIG_TRANSPARENT_HUGEPAGE)) {
			if (pmdp_clear_young_notify(vma, addr, pvmw.pmd))
				referenced = true;
		} else {
			/* unexpected pmd-mapped page? */
			WARN_ON_ONCE(1);
		}
	}

	if (referenced) {//如果这个页最近有被访问过，则清除idle标记并设置young标记
		folio_clear_idle(folio);
		/*
		 * We cleared the referenced bit in a mapping to this page. To
		 * avoid interference with page reclaim, mark it young so that
		 * folio_referenced() will return > 0.
         * 虽然accessed bit被清除了，但是PG_young被置1表示最近该页被访问过，
         * 所以在LRU扫描时folio_referenced() will return > 0，不影响lru扫描
		 */
		folio_set_young(folio);
	}
	return true;
}



//手动清除页面的idle标记
static ssize_t page_idle_bitmap_write(struct file *file, struct kobject *kobj,
				      struct bin_attribute *attr, char *buf,
				      loff_t pos, size_t count)
{
	const u64 *in = (u64 *)buf;
	struct folio *folio;
	unsigned long pfn, end_pfn;
	int bit;

	if (pos % BITMAP_CHUNK_SIZE || count % BITMAP_CHUNK_SIZE)
		return -EINVAL;

	pfn = pos * BITS_PER_BYTE;
	if (pfn >= max_pfn)
		return -ENXIO;

	end_pfn = pfn + count * BITS_PER_BYTE;
	if (end_pfn > max_pfn)
		end_pfn = max_pfn;

	for (; pfn < end_pfn; pfn++) {
		bit = pfn % BITMAP_CHUNK_BITS;
		if ((*in >> bit) & 1) {
			folio = page_idle_get_folio(pfn);
			if (folio) {
				page_idle_clear_pte_refs(folio);//清除PTE的accessed bit
				folio_set_idle(folio);   //设置idle标记，PG_young保留了accessed bit的信息
				folio_put(folio);
			}
		}
		if (bit == BITMAP_CHUNK_BITS - 1)
			in++;
		cond_resched();
	}
	return (char *)in - buf;
}
```


