**1. Ryan Roberts（ARM）贡献的 Multi-size THP for anonymous memory**

mm: allow deferred splitting of arbitrary anon large folios

这个 patchset，允许匿名页发生缺页中断的时候，申请多种不同 size 的 PTE-mapped 的 large folios。而内核原先的 THP 主要针对的是 PMD-mapped 的2MiB size，在支持多种 size 后，我们把 multi-size THP 简称为 mTHP。现在 /sys/kernel/mm/transparent_hugepage 目录下面，会有多个 hugepages- 子目录：

```bash
rlk@rlk:~$ cat  /sys/kernel/mm/transparent_hugepage/h
hpage_pmd_size    hugepages-128kB/  hugepages-2048kB/ hugepages-32kB/   hugepages-64kB/
hugepages-1024kB/ hugepages-16kB/   hugepages-256kB/  hugepages-512kB/


rlk@rlk:~$ cat  /sys/kernel/mm/transparent_hugepage/hugepages-16kB/enabled
always inherit madvise [never]
```

这样在发生 PF 的时候，do_anonymous_page () 可以申请 64KiB 的 mTHP，并一次性透过 set_ptes 把 16 个 PTE 全部设置上：

```c
do_anonymous_page
    ->alloc_anon_folio  //新增申请页面接口
        ->thp_vma_allowable_orders//根据VMA->vm_end，和系统允许的大页order，确定要申请的order
        ->thp_vma_suitable_orders
        ->//根据是否已经申请pte表项（pte_range_none），确定order
        ->vma_alloc_folio
```

后续研究是否可以优化下order的计算，提个patch？？？

```c
/**
 * folio_add_new_anon_rmap - Add mapping to a new anonymous folio.
 * @folio:	The folio to add the mapping to.
 * @vma:	the vm area in which the mapping is added
 * @address:	the user virtual address mapped
 *
 * Like folio_add_anon_rmap_*() but must only be called on *new* folios.
 * This means the inc-and-test can be bypassed.
 * The folio does not have to be locked.
 *
 * If the folio is pmd-mappable, it is accounted as a THP.  As the folio
 * is new, it's assumed to be mapped exclusively by a single process.
 */
void folio_add_new_anon_rmap(struct folio *folio, struct vm_area_struct *vma,
		unsigned long address)
{
	int nr = folio_nr_pages(folio);

	VM_WARN_ON_FOLIO(folio_test_hugetlb(folio), folio);
	VM_BUG_ON_VMA(address < vma->vm_start ||
			address + (nr << PAGE_SHIFT) > vma->vm_end, vma);
	__folio_set_swapbacked(folio);
	__folio_set_anon(folio, vma, address, true);

	if (likely(!folio_test_large(folio))) {//folio不是大页，folio是单个的页
		/* increment count (starts at -1) */
		atomic_set(&folio->_mapcount, 0);
		SetPageAnonExclusive(&folio->page);
	} else if (!folio_test_pmd_mappable(folio)) {//folio是大页，但不是PMD_ORDER的大页,代表folio是64KB、128KN这样的普通大页
		int i;

		for (i = 0; i < nr; i++) {
			struct page *page = folio_page(folio, i);

			/* increment count (starts at -1) */
			atomic_set(&page->_mapcount, 0);
			SetPageAnonExclusive(page);
		}

		atomic_set(&folio->_nr_pages_mapped, nr);
	} else {//filio是folio_test_pmd_mappable的大页
		/* increment count (starts at -1) */
		atomic_set(&folio->_entire_mapcount, 0);
		atomic_set(&folio->_nr_pages_mapped, ENTIRELY_MAPPED);
		SetPageAnonExclusive(&folio->page);
		__lruvec_stat_mod_folio(folio, NR_ANON_THPS, nr);
	}

	__lruvec_stat_mod_folio(folio, NR_ANON_MAPPED, nr);
}
```

```c

static bool pte_range_none(pte_t *pte, int nr_pages)
{
	int i;

	for (i = 0; i < nr_pages; i++) {
		if (!pte_none(ptep_get_lockless(pte + i)))
			return false;
	}

	return true;
}

static struct folio *alloc_anon_folio(struct vm_fault *vmf)
{
	struct vm_area_struct *vma = vmf->vma;
#ifdef CONFIG_TRANSPARENT_HUGEPAGE
	unsigned long orders;
	struct folio *folio;
	unsigned long addr;
	pte_t *pte;
	gfp_t gfp;
	int order;

	/*
	 * If uffd is active for the vma we need per-page fault fidelity to
	 * maintain the uffd semantics.
	 */
	if (unlikely(userfaultfd_armed(vma)))
		goto fallback;

	/*
	 * Get a list of all the (large) orders below PMD_ORDER that are enabled
	 * for this vma. Then filter out the orders that can't be allocated over
	 * the faulting address and still be fully contained in the vma.
	 */
	orders = thp_vma_allowable_orders(vma, vma->vm_flags, false, true, true,
					  BIT(PMD_ORDER) - 1);//检查vma允许的大页order，返回允许order位图
	orders = thp_vma_suitable_orders(vma, vmf->address, orders);//计算本次申请合适的order

	if (!orders)//没有合适的大页order，申请小页
		goto fallback;

	pte = pte_offset_map(vmf->pmd, vmf->address & PMD_MASK);
	if (!pte)
		return ERR_PTR(-EAGAIN);

	/*
	 * Find the highest order where the aligned range is completely
	 * pte_none(). Note that all remaining orders will be completely
	 * pte_none().
	 */

/*
在所有允许的order中，找到一个最大的order，这连续2^order个PTE表项都为空，
意味着后边可以为申请到的页面做映射
*/
	order = highest_order(orders);
	while (orders) {
		addr = ALIGN_DOWN(vmf->address, PAGE_SIZE << order);
		if (pte_range_none(pte + pte_index(addr), 1 << order))
			break;
		order = next_order(&orders, order);
	}

	pte_unmap(pte);

	/* Try allocating the highest of the remaining orders. */
    /* 尽可能申请最大order的页面. */
	gfp = vma_thp_gfp_mask(vma);
	while (orders) {
		addr = ALIGN_DOWN(vmf->address, PAGE_SIZE << order);
		folio = vma_alloc_folio(gfp, order, vma, addr, true);
		if (folio) {
			if (mem_cgroup_charge(folio, vma->vm_mm, gfp)) {
				folio_put(folio);
				goto next;
			}
			folio_throttle_swaprate(folio, gfp);
			clear_huge_page(&folio->page, vmf->address, 1 << order);//页面内容写0
			return folio;
		}
next:
		order = next_order(&orders, order);
	}

fallback:
#endif
	return folio_prealloc(vma->vm_mm, vma, vmf->address, true);
}


```





**2、 Ryan Roberts（ARM）贡献的 Transparent Contiguous PTEs for Us er Mappings**

这个 patchset 主要让 mTHP 可以自动用上 ARM64 的 CONT-PTE，即 16 个 PTE 对应的 PFN 如果物理连续且自然对界，则设 CONT bit 以便让它们只占用一个 TLB entry。Ryan 的这个 patchset 比较精彩的地方在于，mm 的 core 层其实不必意识到 CONT-PTE 的存在（因为不是啥硬件 ARCH 都有这个优化），保持了 PTE 相关 API 向 mm 的完全兼容，而在 ARM64 arch 的实现层面，自动加上或者去掉 CONT bit。

比如原先 16 个 PTE 满足 CONT 的条件，如果有人 unmap 掉了其中 1 个 PTE 或者 mprotect 改变了 16 个 PTE 中一部分 PTE 的属性导致 CONT 不再能满足，set_ptes() 调用的 contpte_try_unfold() 则可将 CONT bit 自动 unfold 掉

```c
static inline void set_ptes(struct mm_struct *mm, unsigned long addr,
                        pte_t *ptep, pte_t pte, unsigned int nr)
{
    pte = pte_mknoncont(pte);

    if (likely(nr == 1)) {
            contpte_try_unfold(mm, addr, ptep, __ptep_get(ptep));
            __set_ptes(mm, addr, ptep, pte, 1);
    } else {
            contpte_set_ptes(mm, addr, ptep, pte, nr);
    }
}
```

**3、Ryan Roberts（ARM）贡献的 Swap-out mTHP without splitting**

此 patchset 在 vmscan.c 对内存进行回收的时候，不将 mTHP split 为 small folios（除非 large folio 之前已经被加入了 _deferred_list，证明其很可能已经被 partially unmap 了），而是整体申请多个 swap slots 和写入 swapfile。

不过这里有一个问题，在 add_to_swap() 整体申请 nr_pages 个连续的 swap slots 的时候，swapfile 完全可能已经碎片化导致申请不到，这样它仍然需要回退到 split。

相信 swapfile 的反碎片问题，将是后续社区的一个重要话题，这方面 Chris Li（Google）有一些思考 Swap Abstraction "the pony"[5]，更多的讨论可能在 2024 年 5 月盐湖城的 LSF 上进行。

**4、Chuanhua Han（OPPO）、Barry Song（OPPO 顾问）贡献的 mm: support large folios swap-in**

这个 patchset 瞄准让 do_swap_page() 在 swapin 的时候也能直接以 large folio 形式进行，从而减小 do_swap_page() 路径上的 PF。另外，更重要的一点是，如果 swapin 路径上不支持 mTHP 的话，前述 Ryan Roberts 的工作成果可能就因为 mTHP swapout 出去，swapin 回来就再也不是 mTHP了。

针对 Android、嵌入式等场景，swapout 频繁发生，因为 swapout 而一夜之间失去 mTHP 的优势，变成穷光蛋，实在有点说不过去。理论上 swapin 路径上的 mTHP 支持有 3 条可能路径：

a，在 swapcache 中直接命中了一个 large folio

b，在 SWP_SYNCHRONOUS_IO 路径上针对同步设备的 swapin

c，在 swapin_readahead() 路径上针对非同步设备或者 __swap_count(entry) != 1 的 swapin。

目前 patchset 瞄准 a、b 针对手机和嵌入式等采用 zRAM 的场景进行，相信该 patchset 后续会进一步发展到针对路径 c 的支持。近期可能较早能合入的是路径 a 的部分 large folios swap-in: handle refault cases first
