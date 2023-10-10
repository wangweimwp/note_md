最终申请的大小size和align有关系，若align传入0，这会在申请是4字节对齐（32位机）

```c
struct kmem_cache *
kmem_cache_create(const char *name, size_t size, size_t align,
          unsigned long flags, void (*ctor)(void *))
```

zram中使用zs_malloc的例子

```c
    handle = zs_malloc(meta->mem_pool, clen,                
                __GFP_KSWAPD_RECLAIM |
                __GFP_NOWARN |
                __GFP_HIGHMEM |
                __GFP_MOVABLE);        //从zs_pool中申请一个对象

    cmem = zs_map_object(meta->mem_pool, handle, ZS_MM_WO);//将handle所在的zspage用kmap_stomic订住

    memcpy(cmem, src, PAGE_SIZE);//做复制操作

    zs_unmap_object(meta->mem_pool, handle);//kumap_atomic掉zspage
```

zs_map_object中的zs_map_area变量，它保存了映射地址和模式，所以同一时间同一cpu只能有一个handle被映射
