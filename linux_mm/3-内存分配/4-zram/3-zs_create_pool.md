```c
/**
 * zs_create_pool - Creates an allocation pool to work from.
 * @name: pool name to be created
 *
 * This function must be called before anything when using
 * the zsmalloc allocator.
 *
 * On success, a pointer to the newly created pool is returned,
 * otherwise NULL.
 */
struct zs_pool *zs_create_pool(const char *name)
{
	int i;
	struct zs_pool *pool;
	struct size_class *prev_class = NULL;

	pool = kzalloc(sizeof(*pool), GFP_KERNEL);
	if (!pool)
		return NULL;

	init_deferred_free(pool);
	pool->size_class = kcalloc(zs_size_classes, sizeof(struct size_class *),
			GFP_KERNEL);
	if (!pool->size_class) {
		kfree(pool);
		return NULL;
	}

	pool->name = kstrdup(name, GFP_KERNEL);
	if (!pool->name)
		goto err;

	if (create_cache(pool))//为zs_handle和zspage创建slab内存池
		goto err;

	/*
	 * Iterate reversly, because, size of size_class that we want to use
	 * for merging should be larger or equal to current size.
/*
#define CLASS_BITS	8
#define ZS_SIZE_CLASS_DELTA	(PAGE_SIZE >> CLASS_BITS)  //16

#define ZS_MAX_ALLOC_SIZE	PAGE_SIZE
#define ZS_MAX_ZSPAGE_ORDER 3
#define ZS_MAX_PAGES_PER_ZSPAGE (_AC(1, UL) << ZS_MAX_ZSPAGE_ORDER) //8

#define ZS_MIN_ALLOC_SIZE \
	MAX(32, (ZS_MAX_PAGES_PER_ZSPAGE << PAGE_SHIFT >> OBJ_INDEX_BITS)) //32
#define ZS_SIZE_CLASSES	(DIV_ROUND_UP(ZS_MAX_ALLOC_SIZE - ZS_MIN_ALLOC_SIZE, \
				      ZS_SIZE_CLASS_DELTA) + 1)  // ZS_SIZE_CLASS_DELTA为16，ZS_SIZE_CLASSES为254+1
*/
	 */
	for (i = zs_size_classes - 1; i >= 0; i--) {//zs_size_classes见init_zs_size_classes
		int size;
		int pages_per_zspage;
		int objs_per_zspage;
		struct size_class *class;
		int fullness = 0;
/*初始化各class, 范围为ZS_MIN_ALLOC_SIZE(32)到ZS_MAX_ALLOC_SIZE(4096)， 间隔为_SIZE_CLASS_DELTA(16),对象大小可以是32、48。。。4k，一共255种可能*/
		size = ZS_MIN_ALLOC_SIZE + i * ZS_SIZE_CLASS_DELTA;
		if (size > ZS_MAX_ALLOC_SIZE)
			size = ZS_MAX_ALLOC_SIZE; //ZS_MAX_ALLOC_SIZE为PAGE_SIZE, 最大情况为不压缩
		pages_per_zspage = get_pages_per_zspage(size);//对每种size object分配合适的page数， 使得内存浪费最少。
		objs_per_zspage = pages_per_zspage * PAGE_SIZE / size;

		/*
		 * size_class is used for normal zsmalloc operation such
		 * as alloc/free for that size. Although it is natural that we
		 * have one size_class for each size, there is a chance that we
		 * can get more memory utilization if we use one size_class for
		 * many different sizes whose size_class have same
		 * characteristics. So, we makes size_class point to
		 * previous size_class if possible.
		 */
/*
		判断是否可以使用上一次分配的size_class作为本次的size_class。
		如果本次要分配的页数和上次分配的页数相同，并且两次可保存的最大对象数也相同，则使用上次的size_class。
		如果条件成立，意味着如果为本次分配新的size_class，将产生比上次更大的内存碎片而造成浪费，
		这时不如直接用上次的size_class。这就是为什么for循环是从zs_size_classes - 1开始向下遍历，而不是从0开始。
		如果不能merge，则创建新的size_class并初始化各个成员变量。注意，此时并未为size_class分配内存页
*/
		if (prev_class) {
			if (can_merge(prev_class, pages_per_zspage, objs_per_zspage)) {
				pool->size_class[i] = prev_class;
				continue;
			}
		}

		class = kzalloc(sizeof(struct size_class), GFP_KERNEL);//分配class内存
		if (!class)
			goto err;

		class->size = size;
		class->index = i;
		class->pages_per_zspage = pages_per_zspage;
		class->objs_per_zspage = objs_per_zspage;
		spin_lock_init(&class->lock);
		pool->size_class[i] = class;
		for (fullness = ZS_EMPTY; fullness < NR_ZS_FULLNESS;//初始化	ZS_ALMOST_EMPTY,ZS_ALMOST_FULL,ZS_FULL几种链表
							fullness++)
			INIT_LIST_HEAD(&class->fullness_list[fullness]);

		prev_class = class;
	}

	/* debug only, don't abort if it fails */
	zs_pool_stat_create(pool, name);

	if (zs_register_migration(pool))
		goto err;

	/*
	 * Not critical, we still can use the pool
	 * and user can trigger compaction manually.
	 */
	if (zs_register_shrinker(pool) == 0)//为zs_pool注册shrinker
		pool->shrinker_enabled = true;
	return pool;

err:
	zs_destroy_pool(pool);
	return NULL;
}
EXPORT_SYMBOL_GPL(zs_create_pool);

```

可以看到zs_size_classes的计算和ZS_SIZE_CLASS_DELTA、ZS_MIN_ALLOC_SIZE、ZS_MAX_ALLOC_SIZE是有关系的。

这使得zs_size_classes即为所有可能的个数。

```c
static void __init init_zs_size_classes(void)
{
	int nr;

	nr = (ZS_MAX_ALLOC_SIZE - ZS_MIN_ALLOC_SIZE) / ZS_SIZE_CLASS_DELTA + 1;
	if ((ZS_MAX_ALLOC_SIZE - ZS_MIN_ALLOC_SIZE) % ZS_SIZE_CLASS_DELTA)
		nr += 1;

	zs_size_classes = nr;
}
```


