```c
int __mem_cgroup_charge(struct folio *folio, struct mm_struct *mm, gfp_t gfp)
{
    struct mem_cgroup *memcg;
    int ret;

    /**
 * get_mem_cgroup_from_mm: Obtain a reference on given mm_struct's memcg.
 * @mm: mm from which memcg should be extracted. It can be NULL.
 *
 * Obtain a reference on mm->memcg and returns it if successful. If mm
 * is NULL, then the memcg is chosen as follows:如果mm传入NULL
 * 1) The active memcg, if set.
 * 2) current->mm->memcg, if available
 * 3) root memcg
 * If mem_cgroup is disabled, NULL is returned.
 */
    memcg = get_mem_cgroup_from_mm(mm);
    ret = charge_memcg(folio, memcg, gfp);
    css_put(&memcg->css);//css的refcnt-1

    return ret;
}
static int charge_memcg(struct folio *folio, struct mem_cgroup *memcg,
            gfp_t gfp)
{
    long nr_pages = folio_nr_pages(folio);
    int ret;

    ret = try_charge(memcg, gfp, nr_pages);
    if (ret)
        goto out;

    css_get(&memcg->css);//css的refcnt+1
    commit_charge(folio, memcg);

    local_irq_disable();
    mem_cgroup_charge_statistics(memcg, nr_pages);
    memcg_check_events(memcg, folio_nid(folio));
    local_irq_enable();
out:
    return ret;
}

static inline int try_charge(struct mem_cgroup *memcg, gfp_t gfp_mask,
                 unsigned int nr_pages)
{
    if (mem_cgroup_is_root(memcg))
        return 0;

    return try_charge_memcg(memcg, gfp_mask, nr_pages);
}

static int try_charge_memcg(struct mem_cgroup *memcg, gfp_t gfp_mask,
            unsigned int nr_pages)
{    //gfp_mask = GFP_KERNEL 是do_anonymous_page传进来的
    unsigned int batch = max(MEMCG_CHARGE_BATCH, nr_pages);
    int nr_retries = MAX_RECLAIM_RETRIES;
    struct mem_cgroup *mem_over_limit;
    struct page_counter *counter;
    unsigned long nr_reclaimed;
    bool passed_oom = false;
    unsigned int reclaim_options = MEMCG_RECLAIM_MAY_SWAP;
    bool drained = false;
    bool raised_max_event = false;
    unsigned long pflags;

retry:
    if (consume_stock(memcg, nr_pages))  //消耗存货 如果当前cpu的记账缓存从准备记账的内存控制组预留的页数足够多，那么从记账缓存减去准备记账的页数(而无需向内存控制组记账), 并结束记账返回成功
        return 0;
    // 1. 没有开启内存+交换分区记账, 直接进入mem_cgroup->memory记账流程（或（||）之前的条件成立，后边的条件便不再执行）
    // 2. 开启内存+交换分区记账, 则首先进入memsw记账流程, 成功后再进入memory记账流程
    if (!do_memsw_account() ||        //
        page_counter_try_charge(&memcg->memsw, batch, &counter)) {
        if (page_counter_try_charge(&memcg->memory, batch, &counter))
            goto done_restock;//到这说明存货不足，记账成功后补充存货
        if (do_memsw_account())
            page_counter_uncharge(&memcg->memsw, batch);     // 如果mem_cgroup->memory超过限制记账失败,而且开启内存+交换分区记账, 则还需要撤销29行内存+交换分区的记账    
        mem_over_limit = mem_cgroup_from_counter(counter, memory);    //记录内存使用量超过限制的内存控制组
    } else {
        mem_over_limit = mem_cgroup_from_counter(counter, memsw);    //记录内存使用量超过限制的内存控制组
        reclaim_options &= ~MEMCG_RECLAIM_MAY_SWAP;
    }

    if (batch > nr_pages) {//记账失败的情况下，若batch大于实际的页数，用实际的页数重新记账
        batch = nr_pages;
        goto retry;
    }

    /*代码到这意味着记账失败，记账会超过内存用量限制
    若在内存回收操作时触发记账，允许暂时超过限制，期望回收操作使内存回到水位之下
     * Prevent unbounded recursion when reclaim operations need to
     * allocate memory. This might exceed the limits temporarily,
     * but we prefer facilitating memory reclaim and getting back
     * under the limit over triggering OOM kills in these cases.
     */
    if (unlikely(current->flags & PF_MEMALLOC))
        goto force;

    if (unlikely(task_in_memcg_oom(current)))
        goto nomem;

    if (!gfpflags_allow_blocking(gfp_mask))    //调用者不允许被阻塞
        goto nomem;
    // 记录MEMCG_MAX事件到mem_events 和 mem_events_local(父mem_cgroup也要记录)
    memcg_memory_event(mem_over_limit, MEMCG_MAX);
    raised_max_event = true;

    psi_memstall_enter(&pflags);//设置进程标记，进程正在应为申请内存而停顿
    nr_reclaimed = try_to_free_mem_cgroup_pages(mem_over_limit, nr_pages,
                            gfp_mask, reclaim_options);//释放memcg的页，并返回已释放的页
    psi_memstall_leave(&pflags);

    if (mem_cgroup_margin(mem_over_limit) >= nr_pages)//内存释放之后，超过限制的memcg是否满足nr_pages个page的记账
        goto retry;

    if (!drained) {
        drain_all_stock(mem_over_limit);//代表是否有把每处理器记账存货的页数归还给内存控制组
        drained = true;
        goto retry;
    }

    if (gfp_mask & __GFP_NORETRY)
        goto nomem;
    /*
     * Even though the limit is exceeded at this point, reclaim
     * may have been able to free some pages.  Retry the charge
     * before killing the task.
     *
     * Only for regular pages, though: huge pages are rather
     * unlikely to succeed so close to the limit, and we fall back
     * to regular pages anyway in case of failure.
        巨型页在如此接近基线的情况下不太可能成功
     */
    if (nr_reclaimed && nr_pages <= (1 << PAGE_ALLOC_COSTLY_ORDER))
        goto retry;
    /*
     * At task move, charge accounts can be doubly counted. So, it's
     * better to wait until the end of task_move if something is going on.
     */
    if (mem_cgroup_wait_acct_move(mem_over_limit))//超过内存限制的memcg正在进行move charg，等待完成后重试
        goto retry;

    if (nr_retries--)//最多重试16次
        goto retry;

    if (gfp_mask & __GFP_RETRY_MAYFAIL)
        goto nomem;

    /* Avoid endless loop for tasks bypassed by the oom killer */
    if (passed_oom && task_is_dying())
        goto nomem;

    /*
     * keep retrying as long as the memcg oom killer is able to make
     * a forward progress or bypass the charge if the oom killer
     * couldn't make any progress.只有OOM还可以杀死进程，就继续重试
     */
    if (mem_cgroup_oom(mem_over_limit, gfp_mask,
               get_order(nr_pages * PAGE_SIZE))) {
        passed_oom = true;
        nr_retries = MAX_RECLAIM_RETRIES;
        goto retry;
    }
nomem:
    /*
     * Memcg doesn't have a dedicated reserve for atomic
     * allocations. But like the global atomic pool, we need to
     * put the burden of reclaim on regular allocation requests
     * and let these go through as privileged allocations.
     */
    if (!(gfp_mask & (__GFP_NOFAIL | __GFP_HIGH)))//没事设置该优先级内存申请标记
        return -ENOMEM;
force:
    /*
     * If the allocation has to be enforced, don't forget to raise
     * a MEMCG_MAX event.
     */
    if (!raised_max_event)
        memcg_memory_event(mem_over_limit, MEMCG_MAX);

    /*
     * The allocation either can't fail or will lead to more memory
     * being freed very soon.  Allow memory usage go over the limit
     * temporarily by force charging it.允许暂时超出限制
     */
    page_counter_charge(&memcg->memory, nr_pages);
    if (do_memsw_account())
        page_counter_charge(&memcg->memsw, nr_pages);

    return 0;

done_restock:
    if (batch > nr_pages)
        refill_stock(memcg, batch - nr_pages);//memcg记账首先会计MAX_RECLAIM_RETRIES个page，失败后才会计nr_pages
                                    //如果计MAX_RECLAIM_RETRIES个page成功，多出来的page会被当做存货，因此refill_stock不会再调用page_counter_try_charge
                                    //因为MAX_RECLAIM_RETRIES个page已经记账了

    /*
     * If the hierarchy is above the normal consumption range, schedule
     * reclaim on returning to userland.  We can perform reclaim here
     * if __GFP_RECLAIM but let's always punt for simplicity and so that
     * GFP_KERNEL can consistently be used during reclaim.  @memcg is
     * not recorded as it most likely matches current's and won't
     * change in the meantime.  As high limit is checked again before
     * reclaim, the cost of mismatch is negligible.
     */
    do {
        bool mem_high, swap_high;

        mem_high = page_counter_read(&memcg->memory) >
            READ_ONCE(memcg->memory.high);
        swap_high = page_counter_read(&memcg->swap) >
            READ_ONCE(memcg->swap.high);

        /* Don't bother a random interrupted task */
        if (!in_task()) {
            if (mem_high) {//在进程上下文中，若内存使用超过memory.high，将memcg->high_work加入调度队列
                schedule_work(&memcg->high_work);//memcg->high_work在mem_cgroup_alloc中初始化，high_work中会进行内存回收
                break;
            }
            continue;
        }

        if (mem_high || swap_high) {
            /*
             * The allocating tasks in this cgroup will need to do
             * reclaim or be throttled to prevent further growth
             * of the memory or swap footprints.
             *
             * Target some best-effort fairness between the tasks,
             * and distribute reclaim work and delay penalties
             * based on how much each task is actually allocating.
             */
            current->memcg_nr_pages_over_high += batch;
            set_notify_resume(current);//返回用户空间时调用回调函数
            break;
        }
    } while ((memcg = parent_mem_cgroup(memcg)));

    if (current->memcg_nr_pages_over_high > MEMCG_CHARGE_BATCH &&
        !(current->flags & PF_MEMALLOC) &&
        gfpflags_allow_blocking(gfp_mask)) {
        mem_cgroup_handle_over_high();//进行内存回收
    }
    return 0;
}

/**
 * page_counter_try_charge - try to hierarchically charge pages
 * @counter: counter
 * @nr_pages: number of pages to charge
 * @fail: points first counter to hit its limit, if any
 *
 * Returns %true on success, or %false and @fail if the counter or one
 * of its ancestors has hit its configured limit.
 */
bool page_counter_try_charge(struct page_counter *counter,
                 unsigned long nr_pages,
                 struct page_counter **fail)
{
    struct page_counter *c;

    for (c = counter; c; c = c->parent) {//遍历所有父counter，并记账
        long new;
        /*
         * Charge speculatively to avoid an expensive CAS.  If
         * a bigger charge fails, it might falsely lock out a
         * racing smaller charge and send it into reclaim
         * early, but the error is limited to the difference
         * between the two sizes, which is less than 2M/4M in
         * case of a THP locking out a regular page charge.
         *
         * The atomic_long_add_return() implies a full memory
         * barrier between incrementing the count and reading
         * the limit.  When racing with page_counter_set_max(),
         * we either see the new limit or the setter sees the
         * counter has changed and retries.
         */
        new = atomic_long_add_return(nr_pages, &c->usage);
        if (new > c->max) {
            atomic_long_sub(nr_pages, &c->usage);
            /*
             * This is racy, but we can live with some
             * inaccuracy in the failcnt which is only used
             * to report stats.
             */
            data_race(c->failcnt++);
            *fail = c;//记账超过限制，返回第一个超过限制的counter
            goto failed;
        }
        propagate_protected_usage(c, new);//用于跟踪childern counter的low_usage
        /*
         * Just like with failcnt, we can live with some
         * inaccuracy in the watermark.
         */
        if (new > READ_ONCE(c->watermark))
            WRITE_ONCE(c->watermark, new);
    }
    return true;

failed:
    for (c = counter; c != *fail; c = c->parent)
        page_counter_cancel(c, nr_pages);//将失败之前的counter取消记账

    return false;
}
```
