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

```


