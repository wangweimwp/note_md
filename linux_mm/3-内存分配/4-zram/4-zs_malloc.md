```c

/**
 * zs_malloc - Allocate block of given size from pool.
 * @pool: pool to allocate from
 * @size: size of block to allocate
 * @gfp: gfp flags when allocating object
 *
 * On success, handle to the allocated object is returned,
 * otherwise 0.
 * Allocation requests with size > ZS_MAX_ALLOC_SIZE will fail.
 */
unsigned long zs_malloc(struct zs_pool *pool, size_t size, gfp_t gfp)
{
	unsigned long handle, obj;
	struct size_class *class;
	enum fullness_group newfg;
	struct zspage *zspage;

	if (unlikely(!size || size > ZS_MAX_ALLOC_SIZE))
		return 0;

	handle = cache_alloc_handle(pool, gfp);
	if (!handle)
		return 0;

	/* extra space in chunk to keep the handle */
	size += ZS_HANDLE_SIZE;
	class = pool->size_class[get_size_class_index(size)];//拿到对应size大小的 struct size_cleass指针

	spin_lock(&class->lock);
	zspage = find_get_zspage(class);//遍历class->fullness_list每个列表，拿到struct zspage
	if (likely(zspage)) {
		obj = obj_malloc(class, zspage, handle);
		/* Now move the zspage to another fullness group, if required */
		fix_fullness_group(class, zspage);
		record_obj(handle, obj);
		spin_unlock(&class->lock);

		return handle;
	}
	/*如果class中没有可用的zspage，则需要申请内存，新建zspage，完成该工作的是alloc_zspage。
	该函数分配出class->pages_per_zspage个单个的内存页按前述的zspage结构组合成一个zspage。
	并初始化link_free，如前所述，link_free是一个由空闲对象组成的单链表，如果对象已被分配，
	则该位置保存了该对象对应的handle。这就是为什么开始时要将size加上ZS_HANDLE_SIZE，
	它占用了内存块的前ZS_HANDLE_SIZE字节。*/
	spin_unlock(&class->lock);

	zspage = alloc_zspage(pool, class, gfp);//一页一页的申请内存，并初始化struct zspage
	if (!zspage) {
		cache_free_handle(pool, handle);
		return 0;
	}

	spin_lock(&class->lock);
	obj = obj_malloc(class, zspage, handle);//从zspage中申请一个空闲的对象
	newfg = get_fullness_group(class, zspage);看这个zspage是否用完了
	insert_zspage(class, zspage, newfg);//把刚创建的zspage插入到class->fullness_list列表上
	set_zspage_mapping(zspage, class->index, newfg);//设置新创建的zspage部分成员
	record_obj(handle, obj);//handle = obj；
	atomic_long_add(class->pages_per_zspage,
				&pool->pages_allocated);
	zs_stat_inc(class, OBJ_ALLOCATED, class->objs_per_zspage);

	/* We completely set up zspage so mark them as movable */
	SetZsPageMovable(pool, zspage);
	spin_unlock(&class->lock);

	return handle;
}
EXPORT_SYMBOL_GPL(zs_malloc);
```


