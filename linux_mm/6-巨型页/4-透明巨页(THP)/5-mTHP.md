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




