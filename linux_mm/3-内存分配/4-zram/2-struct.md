```c
 #define ZS_MAX_ZSPAGE_ORDER 2
 #define ZS_MAX_PAGES_PER_ZSPAGE (_AC(1, UL) << ZS_MAX_ZSPAGE_ORDER)

 #define _PFN_BITS       (MAX_PHYSMEM_BITS - PAGE_SHIFT)

 #define OBJ_TAG_BITS 1
 #define OBJ_INDEX_BITS  (BITS_PER_LONG - _PFN_BITS - OBJ_TAG_BITS)

 #define ZS_MIN_ALLOC_SIZE \
     MAX(32, (ZS_MAX_PAGES_PER_ZSPAGE << PAGE_SHIFT >> OBJ_INDEX_BITS))
 #define ZS_MAX_ALLOC_SIZE   PAGE_SIZE
```

- ZS_MAX_PAGES_PER_ZSPAGE指定了单个zspage中最多内存页的个数，即4个。

- MAX_PHYSMEM_BITS指定了系统可访问内存的总位数，该系统支持的最大内存即为2^MAX_PHYSMEM_BITS；

- _PFN_BITS表示页帧号需要占用的bit数。MAX_PHYSMEM_BITS在32位系统上通常为32，如果开启了PAE（物理地址扩展）则为36，

- 在64位系统上为46。我们以不支持PAE的32位系统，内存页大小为4k为例，_PFN_BITS=20，

- 剩下的12位分配给obj index和obj tag（后面介绍），OBJ_TAG_BITS=1，OBJ_INDEX_BITS=11，所以ZS_MIN_ALLOC_SIZE = 32.

- 最大的对象大小为一个页（如前所述，这种对象称为巨型对象）

- ZS_SIZE_CLASS_DELTA指定了相邻两个size_class的对象大小差

`#define ZS_SIZE_CLASS_DELTA (PAGE_SIZE >> 8)`

所以ZS_SIZE_CLASS_DELTA=16
