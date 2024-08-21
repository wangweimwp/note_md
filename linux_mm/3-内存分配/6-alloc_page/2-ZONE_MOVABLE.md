ZONE_MOVABLE虚拟内存区域


# 注意
ZONE_MOVABLE 与CMA内存，两者都是可移动页面，用于分配大块内存，但两者分配优先级不同，对于ZONE_MOVABLE，当有用户申请可移动页面是，内核会优先分配ZONE_MOVABLE的页面，而对于CMA，内核优先从MIGRATE_MOVABLE中分配，当内存不足时才会从CMA分配。因此，当内存紧张时内核若要申请不可移动内存，即使CMA有空闲内存也有可能分配失败。

**前言**
ZONE_MOVABLE是一个虚拟内存域，ZONE_MOVABLE内存区域的范围实际上会覆盖高端内存或者NORMAL内存。

ZONE_MOVABLE有两个作用，其一是解决内存碎片问题，将内存域分为可移动和不可移动的（对于可移动和不可移动概念不清楚的可以先了解一下迁移类型以及已分配内存的类型划分），避免不可移动内存处于可移动内存块之间。其二是用于虚拟内存的热插拔，容器动态的进行内存的缩容和扩容需要一整块内存的重分配，对于内存碎片的问题很敏感。

**ZONE_MOVABLE的开启**
ZONE_MOVABLE的开启与是否指定了kernelcore有关，该参数指定了不可移动的内存数量，计算结果会存储在required_kernelcore。movablecore指定了可移动的内存数量。如果同时指定kernelcore和movablecore此时会按照两种方式计算出较大的required_kernelcore，如果都未指定，则该区域范围为0，相当于未开启。内核中find_zone_movable_pfns_for_nodes就是在做这个事情，该函数会按照上述的参数计算出ZONE_MOVABLE内存域的首个page的PFN（page frame number）。

一直都说`ZONE_MOVABLE`是一个虚拟内存域，是因为`ZONE_MOVABLE`管理了`NORMAL`或者`HIGHMEM`的内存，看到这里之后我就产生了疑问，管理的内存与其他zone是重叠的吗？难道同一个内存页会属于两个内存域吗？答案是`ZONE_MOVABLE`管理的内存与其他内存区域是不重叠的。之前我们计算好了`ZONE_MOVABLE`的区域大小和在每个node上的起始pfn的时候就已经确定了`ZONE_MOVABLE`管理的内存区域，在做其他内存区域的初始化时就会对原本的内存区域进行调整，如果别的内存区域计算出的范围和`ZONE_MOVABLE`内定的范围产生了重叠，`adjust_zone_range_for_zone_movable`这个函数就会对这些区域进行压缩，比如某个情况下的`HIGHMEM`区域的内存都被包含在`ZONE_MOBVABLE`中了，经过调整后高端内存内存区域管理的内存就为0。
```c
void __meminit adjust_zone_range_for_zone_movable(int nid,
     unsigned long zone_type,
     unsigned long node_start_pfn,
     unsigned long node_end_pfn,
     unsigned long *zone_start_pfn,
     unsigned long *zone_end_pfn)
{
 /* Only adjust if ZONE_MOVABLE is on this node */
 if (zone_movable_pfn[nid]) {
  /* Size ZONE_MOVABLE */
  if (zone_type == ZONE_MOVABLE) {
   *zone_start_pfn = zone_movable_pfn[nid];
   *zone_end_pfn = min(node_end_pfn,
    arch_zone_highest_possible_pfn[movable_zone]);

  /* Adjust for ZONE_MOVABLE starting within this range */
  } else if (*zone_start_pfn < zone_movable_pfn[nid] &&
    *zone_end_pfn > zone_movable_pfn[nid]) {
   *zone_end_pfn = zone_movable_pfn[nid];

  /* Check if this whole range is within ZONE_MOVABLE */
  } else if (*zone_start_pfn >= zone_movable_pfn[nid])
   *zone_start_pfn = *zone_end_pfn;
 }
}
```
**访问ZONE_MOVABLE**
我们知道访问内存的入口函数需要指定一些`GFP_MASK`，其中和访问内存区域相关的flag会被gfp_zone转化为`ZONE_TYPE`，仅当同时设置`__GFP_HIGHMEM | __GFP_MOVABLE`时伙伴系统才会对`ZONE_MOVABLE`内存区域进行访问。其内存分配流程和其他内存区域没有区别。
```c
static inline enum zone_type gfp_zone(gfp_t flags)
{
#ifdef CONFIG_ZONE_DMA
    if (flags & __GFP_DMA)
        return ZONE_DMA;
#endif
#ifdef CONFIG_ZONE_DMA32
    if (flags & __GFP_DMA32)
        return ZONE_DMA32;
#endif
    if ((flags & (__GFP_HIGHMEM | __GFP_MOVABLE)) ==
            (__GFP_HIGHMEM | __GFP_MOVABLE))
        return ZONE_MOVABLE;
#ifdef CONFIG_HIGHMEM
    if (flags & __GFP_HIGHMEM)
        return ZONE_HIGHMEM;
#endif
    return ZONE_NORMAL;
}
```



Linux内存管理子系统把内存划分为不同zone，本文主要来介绍下其中的一个：ZONE_MOVABLE。我在网上看到一些文章经常会把它叫做虚拟内存区（pseudo zone），为什么说它是一个虚拟内存区呢？实际上它是从平台中最高内存区中（比如ZONE_HIHGMEM）划出了一部分内存，作为ZONE_MOVABLE。内核中有如下代码可以作为参考依据：
```c
 static void __init find_usable_zone_for_movable(void)
 {
     int zone_index;
     for (zone_index = MAX_NR_ZONES - 1; zone_index >= 0; zone_index--) {
         if (zone_index == ZONE_MOVABLE)
             continue;

         if (arch_zone_highest_possible_pfn[zone_index] >
                 arch_zone_lowest_possible_pfn[zone_index])
             break;
     }

     VM_BUG_ON(zone_index == -1);
     movable_zone = zone_index;
 }


```
那么引入该内存区的目的是什么？

它存在的意义实际上是为了减少内存的碎片化，想象一下这个场景，当我们需要一块大的连续内存时向伙伴系统申请，虽然对应的内存区剩余内存还很多，但是却发现对应的内存区并无法满足连续的内存申请需求，这就是由于内存碎片化导致的问题。那么此时是可以通过内存迁移来完成连续内存的申请，但是这个过程并不一定能够成功，因为中间有一些页面可能是不允许迁移的。

引入ZONE_MOVABLE就是为了优化内存迁移场景的，主要目的是想要把Non-Movable和Movable的内存区分管理，当我们划分出该区域后，那么只有可迁移的页才能够从该区域申请，这样当我们后面回收内存时，针对该区域就都可以执行迁移，从而保证能够获取到足够大的连续内存。

除了这个用途之外，该区域还有一种使用场景，那就是memory hotplug场景，内存热插拔场景，当我们对内存区执行remove时，必须保证其中的内容都是可以被迁移走的，因此热插拔的内存区必须位于ZONE_MOVABLE区域。

**ZONE_MOVABLE**
内核中定义了如下一些管理区zone：

```c
enum zone_type {
#ifdef CONFIG_ZONE_DMA
    /*
     * ZONE_DMA is used when there are devices that are not able
     * to do DMA to all of addressable memory (ZONE_NORMAL). Then we
     * carve out the portion of memory that is needed for these devices.
     * The range is arch specific.
     *
     * Some examples
     *
     * Architecture     Limit
     * ---------------------------
     * parisc, ia64, sparc  <4G
     * s390         <2G
     * arm          Various
     * alpha        Unlimited or 0-16MB.
     *
     * i386, x86_64 and multiple other arches
     *          <16M.
     */
    ZONE_DMA,
#endif
#ifdef CONFIG_ZONE_DMA32
    /*
     * x86_64 needs two ZONE_DMAs because it supports devices that are
     * only able to do DMA to the lower 16M but also 32 bit devices that
     * can only do DMA areas below 4G.
     */
    ZONE_DMA32,
#endif
    /*
     * Normal addressable memory is in ZONE_NORMAL. DMA operations can be
     * performed on pages in ZONE_NORMAL if the DMA devices support
     * transfers to all addressable memory.
     */
    ZONE_NORMAL,
#ifdef CONFIG_HIGHMEM
    /*
     * A memory area that is only addressable by the kernel through
     * mapping portions into its own address space. This is for example
     * used by i386 to allow the kernel to address the memory beyond
     * 900MB. The kernel will set up special mappings (page
     * table entries on i386) for each page that the kernel needs to
     * access.
     */
    ZONE_HIGHMEM,
#endif
    ZONE_MOVABLE,
    __MAX_NR_ZONES
};


```
**ZONE_DMA**
该管理区是一些设备无法使用DMA访问所有地址的范围，因此特意划分出来的一块内存，专门用于特殊DMA访问分配使用的区域。比如x86架构此区域为0-16M
**ZONE_NORMAL**
NORMAL区域是直接映射区。
**ZONE_HIGHMEM**
高端内存管理区，申请的内存，需要内核进行map后才能访问。对于64bit Arch架构，我们一般不需要高端内存区，因为地址空间足够映射所有的物理内存。
**ZONE_MOVABLE**
可移除内存区，这是一个虚拟内存区，一般是从平台的最高内存区（比如HIGHMEM）中划分出一部分作为MOVABLE ZONE。

**简单来说，可迁移的页面不一定都在ZONE_MOVABLE中，但是ZONE_MOVABLE中的页面必须都是可迁移的。**

我们通过查看 /proc/pagetypeinfo 的返回信息可以看到在Movable Zone中不存在Unmovable类型的页面，只有Movable类型的页面。

这个管理区域存放的page都是可迁移的，只能被带有__GFP_HIGHMEM和__GFP_MOVABLE标志的内存申请所使用，比如：
```c
#define GFP_HIGHUSER_MOVABLE    (GFP_HIGHUSER | __GFP_MOVABLE)

#define GFP_USER    (__GFP_WAIT | __GFP_IO | __GFP_FS | __GFP_HARDWALL)
#define GFP_HIGHUSER    (GFP_USER | __GFP_HIGHMEM)
```
主要注意的是不要把分配标志__GFP_MOVABLE和管理区ZONE_MOVABLE混淆，两者并不是对应的关系。

__GFP_MOVABLE表示的是一种分配页面属性，表示页面可迁移，即使不在ZONE_MOVABLE管理区，有些页面也是可以迁移的，比如cache；
ZONE_MOVABLE表示的是管理区，其中的页面必须要可迁移。
**分配标志__GFP_MOVABLE**
```c
#define __GFP_DMA   ((__force gfp_t)___GFP_DMA)
#define __GFP_HIGHMEM   ((__force gfp_t)___GFP_HIGHMEM)
#define __GFP_DMA32 ((__force gfp_t)___GFP_DMA32)
#define __GFP_MOVABLE   ((__force gfp_t)___GFP_MOVABLE)  /* Page is movable */
#define GFP_ZONEMASK    (__GFP_DMA|__GFP_HIGHMEM|__GFP_DMA32|__GFP_MOVABLE)
```
分配标志__GFP_MOVABLE
```c
#define __GFP_DMA   ((__force gfp_t)___GFP_DMA)
#define __GFP_HIGHMEM   ((__force gfp_t)___GFP_HIGHMEM)
#define __GFP_DMA32 ((__force gfp_t)___GFP_DMA32)
#define __GFP_MOVABLE   ((__force gfp_t)___GFP_MOVABLE)  /* Page is movable */
#define GFP_ZONEMASK    (__GFP_DMA|__GFP_HIGHMEM|__GFP_DMA32|__GFP_MOVABLE)
```
这几个分配标志被称为Zone modifiers，他们用来标识优先从哪个zone分配内存。
```c
bit       result
=================
0x0    => NORMAL
0x1    => DMA or NORMAL
0x2    => HIGHMEM or NORMAL
0x3    => BAD (DMA+HIGHMEM)
0x4    => DMA32 or DMA or NORMAL
0x5    => BAD (DMA+DMA32)
0x6    => BAD (HIGHMEM+DMA32)
0x7    => BAD (HIGHMEM+DMA32+DMA)
0x8    => NORMAL (MOVABLE+0)
0x9    => DMA or NORMAL (MOVABLE+DMA)
0xa    => MOVABLE (Movable is valid only if HIGHMEM is set too)
0xb    => BAD (MOVABLE+HIGHMEM+DMA)
0xc    => DMA32 (MOVABLE+DMA32)
0xd    => BAD (MOVABLE+DMA32+DMA)
0xe    => BAD (MOVABLE+DMA32+HIGHMEM)
0xf    => BAD (MOVABLE+DMA32+HIGHMEM+DMA)
```
一共有4个bit用来表示组合类型，其中低3个bit只能选择一个（__GFP_DMA/__GFP_HIGHMEM/__GFP_DMA32），而__GFP_MOVABLE可以和其他三种的任何一个组合使用，因此一共有16中组合，根据各种类型进行一个偏移存放到一个long类型table中。
```c
GFP_ZONE_TABLE：

|BAD|BAD|BAD|DMA32|BAD|MOVABLE|......|NORMAL|
```
这些结果会根据上面的bit组合值做一个偏移，存放到ZONE TABLE中，从而可以根据组合快速定位要使用的ZONE管理区。由上可见，__GFP_MOVABLE代表的是一种分配策略，并不是和ZONE_MOVABLE匹配的，上一节也做了介绍，必须是（__GFP_HIGHMEM和__GFP_MOVABLE）同时置位才会从ZONE_MOVABLE管理区去分配内存。
```c
The zone fallback order is MOVABLE=>HIGHMEM=>NORMAL=>DMA32=>DMA
```
因此我们分配内存时并不一定就会按照传入的FLAG来进行分配，如果对应zone中没有符合要求的内存，那么会依次进行fallback查找符合要求的内存。

如何使能ZONE_MOVABLE
可以通过传入不同的cmdline来设置：

比如kernelcore=YYYY和movablecore=ZZZZ分别表示不同的含义：
```c
1) When kernelcore=YYYY boot option is used,
   Size of memory not for movable pages (not for offline) is YYYY.
   Size of memory for movable pages (for offline) is TOTAL-YYYY.

2) When movablecore=ZZZZ boot option is used,
   Size of memory not for movable pages (not for offline) is TOTAL - ZZZZ.
   Size of memory for movable pages (for offline) is ZZZZ.
```