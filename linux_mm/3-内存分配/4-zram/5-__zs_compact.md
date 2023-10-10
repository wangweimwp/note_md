```c
static void __zs_compact(struct zs_pool *pool, struct size_class *class)
{
    struct zs_compact_control cc;
    struct zspage *src_zspage;
    struct zspage *dst_zspage = NULL;

    spin_lock(&class->lock);
    while ((src_zspage = isolate_zspage(class, true))) {//从class的ZS_ALMOST_EMPTY链表或ZS_ALMOST_FULL链表中移出一个zspage。

        if (!zs_can_compact(class))//zs_can_compact根据总的对象个数和已使用的对象个数计算有多少对象空闲，以此得到最多可收缩多少个页。
            break;

        cc.obj_idx = 0;
        cc.s_page = get_first_page(src_zspage);

        while ((dst_zspage = isolate_zspage(class, false))) {//接着再次从class中移出一个zspage，尝试将src_page中的使用中的handle迁移到dst_page中
            cc.d_page = get_first_page(dst_zspage);
            /*
             * If there is no more space in dst_page, resched
             * and see if anyone had allocated another zspage.
             */
            if (!migrate_zspage(pool, class, &cc))//尝试将src_page中的使用中的handle迁移到dst_page中
                break;

            putback_zspage(class, dst_zspage);
        }

        /* Stop if we couldn't find slot */
        if (dst_zspage == NULL)
            break;

        putback_zspage(class, dst_zspage);
        if (putback_zspage(class, src_zspage) == ZS_EMPTY) {
            free_zspage(pool, class, src_zspage);
            pool->stats.pages_compacted += class->pages_per_zspage;
        }
        spin_unlock(&class->lock);
        cond_resched();
        spin_lock(&class->lock);
    }

    if (src_zspage)
        putback_zspage(class, src_zspage);

    spin_unlock(&class->lock);
}

unsigned long zs_compact(struct zs_pool *pool)
{
    int i;
    struct size_class *class;

    for (i = zs_size_classes - 1; i >= 0; i--) {//zs_compact遍历pool中所有的size_class，调用__zs_compact进行收缩工作。
        class = pool->size_class[i];
        if (!class)
            continue;
        if (class->index != i)
            continue;
        __zs_compact(pool, class);
    }

    return pool->stats.pages_compacted;
}
EXPORT_SYMBOL_GPL(zs_compact);
```
