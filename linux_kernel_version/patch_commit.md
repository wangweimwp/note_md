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




