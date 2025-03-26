- mm: compaction: avoid GFP_NOFS ABBA deadlock

```context
During stress testing with higher-order allocations, a deadlock scenario
was observed in compaction: One GFP_NOFS allocation was sleeping on
mm/compaction.c::too_many_isolated(), while all CPUs in the system were
busy with compactors spinning on buffer locks held by the sleeping
GFP_NOFS allocation


系统处于内存压力下，带有GFP_NOFS申请内存时，系统在 too_many_isolated函数睡眠，
但其他压缩器在buffer locks上自旋等待，造成死锁
```

- mm/vmalloc: prevent stale TLBs in fully utilized blocks

```context
_vm_unmap_aliases() is used to ensure that no unflushed TLB entries for a
page are left in the system. This is required due to the lazy TLB flush
mechanism in vmalloc.

This is tried to achieve by walking the per CPU free lists, but those do
not contain fully utilized vmap blocks because they are removed from the
free list once the blocks free space became zero.

When the block is not fully unmapped then it is not on the purge list
either.

So neither the per CPU list iteration nor the purge list walk find the
block and if the page was mapped via such a block and the TLB has not yet
been flushed, the guarantee of _vm_unmap_aliases() that there are no stale
TLBs after returning is broken:

x = vb_alloc() // Removes vmap_block from free list because vb->free became 0
vb_free(x)     // Unmaps page and marks in dirty_min/max range
           // Block has still mappings and is not put on purge list

// Page is reused
vm_unmap_aliases() // Can't find vmap block with the dirty space -> FAIL

So instead of walking the per CPU free lists, walk the per CPU xarrays
which hold pointers to _all_ active blocks in the system including those
removed from the free lists.
```

- mm: compaction: skip fast freepages isolation if enough freepages are isolated

```context
I've observed that fast isolation often isolates more pages than
cc->migratepages, and the excess freepages will be released back to the
buddy system.  So skip fast freepages isolation if enough freepages are
isolated to save some CPU cycles.
内存规整时可以减少CPU消耗
```

- vmstat: allow_direct_reclaim should use zone_page_state_snapshot

```context
修复在用nohz_full（cmdline参数）指定的CPU上，空闲页面技术不准确的
```

- Patch series "mm/ksm: Add ksm advisor", v5.

```text
KSM advisor特性允许自动调整KSM系统
```

- mm: add new api to enable ksm per process

```textile
KSM 继承 prctl()新增PR_SET_MEMORY_MERGE 标志可用于在进程的所有兼容 VMAs 
中启用 KSM 这个设置在进程 fork 时也会继承，因此对于任何子进程来说，
KSM 也将在兼容 VMA 上启用
```

- Merge tag 'slab-for-6.8' of git://git.kernel.org/pub/scm/linux/kernel/git/vbabka/slab

```context
内核删除slab机制，只留slub
```

- sched/deadline: Introduce deadline servers

```text
加入deadline servers机制，防止实时进程占用所有CPU时间而普通进程饿死
```

- mm: skip CMA pages when they are not available

```context
在内存回收时调过CMA页面。
This patch fixes unproductive reclaiming of CMA pages by skipping them
when they are not available for current context.  It arises from the below
OOM issue, which was caused by a large proportion of MIGRATE_CMA pages
among free pages.

遍历不可用的CMA页面会增加nr_scan，系统误任务可回收的页面很少，报OOM，
但是有些可以被回收的页面没有被遍历到，这种现象在CMA页面越多的系统中越容易
```

- mm/filemap.c: fix update prev_pos after one read request done

```context
修复文件预读性能下降，这个性能下降是有V6.4版本合入的一个补丁引入的
```

- mm/vmscan: fix root proactive reclaim unthrottling unbalanced node

```textile
When memory.reclaim was introduced, it became the first case where
cgroup_reclaim() is true for the root cgroup.  Johannes concluded [1] that
for most cases this is okay, except for one case.  Historically, kswapd
would throttle reclaim on a node if a lot of pages marked for reclaim are
under writeback (aka the node is congested).  This occurred by setting
LRUVEC_CONGESTED bit in lruvec->flags.  The bit would be cleared when the
node is balanced.

root cgroup和non-root memcgs共用LRUVEC_CONGESTED 比特位，用于标记当前内存节点
有许多赃页在回写，当node达到平衡后清除，这回造成
```

- Multi-gen LRU: fix per-zone reclaim

```textile
MGLRU,回收时只会回收申请zone的最老一代，若申请zone的最老一代没有页面，
但其他zone的最老一代有比较多的cold，这些page将得不到回收，修复这个问题
```

mm: pgtable: try to reclaim empty PTE pages in zap_page_range_single()

```textile
现在，为了追求高性能，应用程序大多使用一些高性能的用户模式内存分配器，
比如jemalloc或tcmalloc。这些内存分配器使用madvise(MADV_DONTNEED或
MADV_FREE)来释放物理内存，但是MADV_DONTNEED和MADV_FREE都不会释放页表内存，
这可能会导致大量页表内存的使用。对于这样一个所有条目都为空的PTE页面，
我们实际上可以将其释放回系统供其他人使用


```

- [v5,3/4] mm: support large folios swapin as a whole for zRAM-like swapfile

```textile
邮件讨论里有swapin相关
```

- [v2,2/2] mm: Compute first_set_pte to eliminate evaluating redundant ranges
```
作者提交的补丁有性能回退，看看是否能在作者基础上优化
```
- mm: compaction: push watermark into compaction_suitable() callers
  
```c
//优化内存规整过程中的水位问题，使内存规整更多的规整出连续页面
https://lore.kernel.org/all/20250313210647.1314586-1-hannes@cmpxchg.org/
```

# 快起方面

- mm: pass nid to reserve_bootmem_region()

```context
early_pfn_to_nid() is called frequently in init_reserved_page(), it
returns the node id of the PFN.  These PFN are probably from the same
memory region, they have the same node id.  It's not necessary to call
early_pfn_to_nid() for each PFN
before:
memmap_init_reserved_pages()  67ms

after:
memmap_init_reserved_pages()  20ms
```
# 社区讨论中

* [RFCv1 0/6] Page Detective
  
```context
page 数据探测，可以做成驱动，用于探测内存数据
https://lore.kernel.org/all/20241116175922.3265872-1-pasha.tatashin@soleen.com/
```

* [PATCHSET v5 0/17] Uncached buffered IO
```context
新增 Uncached特性，用于快速释放只读写一次的pagecache，有效节省内存
```

# 工作清单
探测机制
1，PSI		一个或多个进程因为等待内存申请而造成的时间停顿超过了阈值，将触发通知事件
2，碎片指数	当前内存无法分配的原因是由于低内存还是外部碎片导致


优化机制
1 内存规整
	规整阈值
	主动规整
	预应性规整
	
2 drop_cache
3 利用ZONE_MOVABLE优化内存碎片

策略
1，分配一定内存到ZONE_MOVABLE中
2，设置PSI阈值，有通知时触发预应性主动内存规整
3，检测到有突发的大量文件读写后触发一次drop_cache和主动规整
4，oomd工具
5，统计设备运行各个order的页面数量，对每个order进行加权，计算出一个整体的碎片指数
	当检测到碎片后进行一次预应性规整，会检测个别order是否有空闲页面，若没有进行预应性规整
	若规整后任没有大块页面进行报警并记录日志
6，之前没有设置uncached的页面，后边设置uncached后会在下次读取后swap出去吗
7，uncached特性与refault distance互动，在page cache读取时检测refault distance值，
	若值大于active_num,说明短期内不会再次读取，设置uncached标记，此次读取后清出page cache



如何检测系统page_cache的数量，那些是可以清掉的，那些不可以   
如何将只使用一次的pagecache尽快清除掉		 RWF_UNCACHED特性 https://lore.kernel.org/all/20241108174505.1214230-1-axboe@kernel.dk/

将uncached特性与refault distance特性相节后
1，若发现xarray树中有这个page的shadow，若在下次读取这个页面前，这个页面有可能被清出lru，则设置uncached flag
2，当page cache被访问时，会更新shadow值，因此短暂出现连续读该page，再第二次读时判断refault distance小于active lru，则会加入到active中


static_assert静态断言解决hal_bsp可能引起的错位问题




# 可以提的补丁
```
https://patchwork.kernel.org/project/linux-mm/patch/20250110182149.746551-3-david@redhat.com/

+/**
+ * folio_isolate_hugetlb: try to isolate an allocated hugetlb folio
+ * @folio: the folio to isolate
+ * @list: the list to add the folio to on success
+ *
+ * Isolate an allocated (refcount > 0) hugetlb folio, marking it as
+ * isolated/non-migratable, and moving it from the active list to the

这里改为 LRU list

+ * given list.
+ *
+ * Isolation will fail if @folio is not an allocated hugetlb folio, or if
+ * it is already isolated/non-migratable.
+ *
+ * On success, an additional folio reference is taken that must be dropped
+ * using folio_putback_active_hugetlb() to undo the isolation.
+ *
+ * Return: True if isolation worked, otherwise False.
+ */
+bool folio_isolate_hugetlb(struct folio *folio, struct list_head *list)
```