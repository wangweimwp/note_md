1 低内存时整合碎片

从buddy申请内存页，如果找不到合适的页，则会进行两步调整内存的工作，compact和reclaim。前者是为了整合碎片，以得到更大的连续内存；后者是回收不一定必须占用内存的缓冲内存。这里重点了解comact，整个流程大致如下：

```c
__alloc_pages_nodemask
    -> __alloc_pages_slowpath
        -> __alloc_pages_direct_compact
            -> try_to_compact_pages
                -> compact_zone_order
                    -> compact_zone
                        -> isolate_migratepages
                        -> migrate_pages
                        -> release_freepages
```
并不是所有申请不到内存的场景都会compact，首先要满足order大于0，并且gfp_mask携带__GFP_FS和__GFP_IO；另外，需要zone的剩余内存情况满足一定条件，kernel称之为“碎片指数”（fragmentation index），这个值在0~1000之间，默认碎片指数大于500时才能进行compact，可以通过proc文件extfrag_threshold来调整这个默认值。fragmentation index通过fragmentation_index函数来计算：


```c
/*
  * Index is between 0 and 1000
  *
  * 0 => allocation would fail due to lack of memory
  * 1000 => allocation would fail due to fragmentation
  */
return 1000 - div_u64( (1000+(div_u64(info->free_pages * 1000ULL, requested))), info->free_blocks_total)
```
在整合内存碎片的过程中，碎片页只会在本zone的内部移动，将位于zone低地址的页尽量移到zone的末端。申请新的页面位置通过compaction_alloc函数实现。

移动过程又分为同步和异步，内存申请失败后第一次compact将会使用异步，后续reclaim之后将会使用同步。同步过程只移动当面未被使用的页，异步过程将遍历并等待所有MOVABLE的页使用完成后进行移动。


## 问题一：
在申请页面时进行内存规整（被动规整）和`echo 1 > /proc/sys/vm/compact_memory`（主动规整）的区别？


1. 主动规整会对所有zone进行规整，而被动规整只会规整申请页面时选定的zone
2. 二者结束条件不同，被动回收检测到水位满足特定条件即可结束，或者两个扫描指针（migrate_pfn、free_pfn）相遇。而主动规整需要两个扫描指针相遇才会结束

## 问题二
unmovable的页面是否可以被compact？

slab申请的页不会挂到lru链表上，也不会作为ksm的目标，因而也就是不会被规整（或者说页不会被迁移），所以不用担心slab会乱套；此外，不可移动页确实也是能迁移的，但是有特殊的要求，就是lru链表上的页，而lru链表上的不可移动页，我理解就只有文件页和被mlock住的匿名页了，而文件页并没有做什么映射动作，只是挂在文件address_space的radix树里面，所以，只需要把old page取下来，替换成newpage上去即可

