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


