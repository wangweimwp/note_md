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
    ->alloc_anon_folio  
        ->thp_vma_allowable_orders//根据VMA->vm_end，和系统允许的大页order，确定要申请的order
        ->thp_vma_suitable_orders
        ->//根据是否已经申请pte表项（pte_range_none），确定order
        ->vma_alloc_folio
```

后续研究是否可以优化下order的计算，提个patch？？？

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


