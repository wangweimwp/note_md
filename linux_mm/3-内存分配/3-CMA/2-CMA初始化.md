如果通过内核参数或配置宏配置全局CMA区域 则调用dma_contiguous_reserve初始化cma

设备树方式

```c
setup_arch->arm64_memblock_init->early_init_fdt_scan_reserved_mem->fdt_scan_reserved_mem->__reserved_mem_reserve_reg

->early_init_dt_reserve_memory->memblock_reserve->memblock_add_rang
```

解析设备树二进制文件中的节点“reserved-memory”，把保留内存块添加到memblock的reserved类型

```c
RESERVEDMEM_OF_DECLARE(cma, "shared-dma-pool", rmem_cma_setup);

static int __init rmem_cma_setup(struct reserved_mem *rmem)
{
    unsigned long node = rmem->fdt_node;
    bool default_cma = of_get_flat_dt_prop(node, "linux,cma-default", NULL);
    struct cma *cma;
    int err;

    if (size_cmdline != -1 && default_cma) {
        pr_info("Reserved memory: bypass %s node, using cmdline CMA params instead\n",
            rmem->name);
        return -EBUSY;
    }

    if (!of_get_flat_dt_prop(node, "reusable", NULL) ||
        of_get_flat_dt_prop(node, "no-map", NULL))
        return -EINVAL;

    if (!IS_ALIGNED(rmem->base | rmem->size, CMA_MIN_ALIGNMENT_BYTES)) {
        pr_err("Reserved memory: incorrect alignment of CMA region\n");
        return -EINVAL;
    }

    err = cma_init_reserved_mem(rmem->base, rmem->size, 0, rmem->name, &cma);//初始化struct cma
    if (err) {
        pr_err("Reserved memory: unable to setup CMA region\n");
        return err;
    }
    /* Architecture specific contiguous memory fixup. */
    dma_contiguous_early_fixup(rmem->base, rmem->size);//记录到dma_mmu_remap数组中

    if (default_cma)
        dma_contiguous_default_area = cma;

    rmem->ops = &rmem_cma_ops;//cma操作函数
    rmem->priv = cma;

    pr_info("Reserved memory: created CMA memory pool at %pa, size %ld MiB\n",
        &rmem->base, (unsigned long)rmem->size / SZ_1M);

    return 0;
}
```

memblock是内核初始化的时候使用的内存分配器，内核初始化完以后使用伙伴分配器管理物理页。内核初始化完成的时候，把空闲的内存释放给伙伴分配器，不会把保留的内存释放给伙伴分配器。<u>CMA区域属于保留的内存，但是我们需要把CMA区域的物理页交给伙伴分配器管理</u>

通常情况下，一个pageblock的大小（pageblock_order）是<u>2^(MAX_ORDER-1)个page </u>MIGRATE_UNMOVABLE管理最小单位为一个pageblock

```c
static void __init cma_activate_area(struct cma *cma)
{
    unsigned long base_pfn = cma->base_pfn, pfn;
    struct zone *zone;

    cma->bitmap = bitmap_zalloc(cma_bitmap_maxno(cma), GFP_KERNEL);//为每个page申请一个bit
    if (!cma->bitmap)
        goto out_error;

    /*
     * alloc_contig_range() requires the pfn range specified to be in the
     * same zone. Simplify by forcing the entire CMA resv range to be in the
     * same zone.
     */
    WARN_ON_ONCE(!pfn_valid(base_pfn));
    zone = page_zone(pfn_to_page(base_pfn));
    for (pfn = base_pfn + 1; pfn < base_pfn + cma->count; pfn++) {//这个struct cma内的page必须在一个zone中或struct mem_section中
        WARN_ON_ONCE(!pfn_valid(pfn));
        if (page_zone(pfn_to_page(pfn)) != zone)
            goto not_in_zone;
    }

    for (pfn = base_pfn; pfn < base_pfn + cma->count;
         pfn += pageblock_nr_pages)
        init_cma_reserved_pageblock(pfn_to_page(pfn));

    spin_lock_init(&cma->lock);

#ifdef CONFIG_CMA_DEBUGFS
    INIT_HLIST_HEAD(&cma->mem_head);
    spin_lock_init(&cma->mem_head_lock);
#endif

    return;

not_in_zone:
    bitmap_free(cma->bitmap);
out_error:
    /* Expose all pages to the buddy, they are useless for CMA. */
    if (!cma->reserve_pages_on_error) {
        for (pfn = base_pfn; pfn < base_pfn + cma->count; pfn++)
            free_reserved_page(pfn_to_page(pfn));
    }
    totalcma_pages -= cma->count;
    cma->count = 0;
    pr_err("CMA area %s could not be activated\n", cma->name);
    return;
}

/* Free whole pageblock and set its migration type to MIGRATE_CMA. */
void __init init_cma_reserved_pageblock(struct page *page)
{
    unsigned i = pageblock_nr_pages;
    struct page *p = page;

    do {
        __ClearPageReserved(p);
        set_page_count(p, 0);
    } while (++p, --i);

    set_pageblock_migratetype(page, MIGRATE_CMA);//设置MIGRATE_CMA属性到zone->pageblock_flags里,
    set_page_refcounted(page);//pageblock的第一页的page->__refcount设置为1，防止__free_pages报错
    __free_pages(page, pageblock_order);//将page放到zone DMA的 zone->free_area->free_list[MIGRATE_CMA]管理

    adjust_managed_page_count(page, pageblock_nr_pages);
    page_zone(page)->cma_pages += pageblock_nr_pages;
}
```
