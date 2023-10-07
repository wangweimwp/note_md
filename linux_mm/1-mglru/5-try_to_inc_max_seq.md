```c
static bool try_to_inc_max_seq(struct lruvec *lruvec, unsigned long max_seq,
                   struct scan_control *sc, bool can_swap, bool force_scan)
{
    bool success;
    struct lru_gen_mm_walk *walk;
    struct mm_struct *mm = NULL;
    struct lru_gen_struct *lrugen = &lruvec->lrugen;
    //在get_nr_to_scan中，max_seq赋值为lruvec->lrugen.max_seq
    VM_WARN_ON_ONCE(max_seq > READ_ONCE(lrugen->max_seq));//lrugen->max_seq初始化时为MIN_NR_GENS + 1

    /* see the comment in iterate_mm_list() lruvec->mm_state.seq初始化时为MIN_NR_GENS*/
    if (max_seq <= READ_ONCE(lruvec->mm_state.seq)) {//若为首次迭代，这里判断不成立
        success = false;
        goto done;
    }

    /*
     * If the hardware doesn't automatically set the accessed bit, fallback
     * to lru_gen_look_around(), which only clears the accessed bit in a
     * handful of PTEs. Spreading the work out over a period of time usually
     * is less efficient, but it avoids bursty page faults.
     * 不是强制扫描，不支持硬件PTE设置访问位，
     */
    if (!force_scan && !(arch_has_hw_pte_young() && get_cap(LRU_GEN_MM_WALK))) {
        success = iterate_mm_list_nowalk(lruvec, max_seq);
        goto done;
    }

    walk = set_mm_walk(NULL)//获取当前进程的current->reclaim_state->mm_walk;
    if (!walk) {//不在kswapd进程里，并且current->reclaim_state->mm_walk为空时，似乎由于内存紧张没有申请到内存
        success = iterate_mm_list_nowalk(lruvec, max_seq);//将lruvec->mm_state结构体清0，不遍历mm_list直接迭代
        goto done;
    }

    walk->lruvec = lruvec;
    walk->max_seq = max_seq;
    walk->can_swap = can_swap;    //lru_gen_shrink_lruvec中和swappiness有关
    walk->force_scan = force_scan;

    do {
        success = iterate_mm_list(lruvec, walk, &mm);//
        if (mm)//找到合适mm_struct
            walk_mm(lruvec, mm, walk);//对mm_struct的vma进行遍历

        cond_resched();
    } while (mm);
done:
    if (!success) {//由iterate_mm_list中可知，该进程不是最后一个退出mm_list上mm_struct页表遍历的进程，但是遍历完上一个mm_struct的页表之后发现其余mm_struct都已经有进程在遍历了，mm返回了NULL，所以退出了do_while循环
        if (sc->priority <= DEF_PRIORITY - 2)//若是强制回收，在最后一个wlaker离开之前，其余walker在这里睡眠，等最后一个walker完成了遍历并且调用inc_max_seq(lruvec, can_swap, force_scan)后；睡眠的进程会被唤醒
            wait_event_killable(lruvec->mm_state.wait,
                        max_seq < READ_ONCE(lrugen->max_seq));    //在lruvec->mm_state.wait上睡眠等待

        return max_seq < READ_ONCE(lrugen->max_seq);
    }

    VM_WARN_ON_ONCE(max_seq != READ_ONCE(lrugen->max_seq));

    inc_max_seq(lruvec, can_swap, force_scan);//遍历完成，增加lrugen->max_seq
    /* either this sees any waiters or they will see updated max_seq */
    if (wq_has_sleeper(&lruvec->mm_state.wait))
        wake_up_all(&lruvec->mm_state.wait);

    return true;
}

static bool iterate_mm_list_nowalk(struct lruvec *lruvec, unsigned long max_seq)
{
    bool success = false;
    struct mem_cgroup *memcg = lruvec_memcg(lruvec);
    struct lru_gen_mm_list *mm_list = get_mm_list(memcg);//初始化一个mm_list
    struct lru_gen_mm_state *mm_state = &lruvec->mm_state;

    spin_lock(&mm_list->lock);

    VM_WARN_ON_ONCE(mm_state->seq + 1 < max_seq);

    if (max_seq > mm_state->seq && !mm_state->nr_walkers) {//max_seq > mm_state->seq并且没有线程查询页表
        VM_WARN_ON_ONCE(mm_state->head && mm_state->head != &mm_list->fifo);

        WRITE_ONCE(mm_state->seq, mm_state->seq + 1);//mm_state->seq + 1
        reset_mm_stats(lruvec, NULL, true);//加1之后，在设置下一个gen的mm_state.stats为0
        success = true;
    }

    spin_unlock(&mm_list->lock);

    return success;
}


static void reset_mm_stats(struct lruvec *lruvec, struct lru_gen_mm_walk *walk, bool last)
{
    int i;
    int hist;

    lockdep_assert_held(&get_mm_list(lruvec_memcg(lruvec))->lock);

    if (walk) {
        hist = lru_hist_from_seq(walk->max_seq);//获得相对gen

        for (i = 0; i < NR_MM_STATS; i++) {
            /**/
            WRITE_ONCE(lruvec->mm_state.stats[hist][i],
                   lruvec->mm_state.stats[hist][i] + walk->mm_stats[i]);
            walk->mm_stats[i] = 0;
        }
    }

    if (NR_HIST_GENS > 1 && last) {
        hist = lru_hist_from_seq(lruvec->mm_state.seq + 1);//获得相对gen（下一个gen）

        for (i = 0; i < NR_MM_STATS; i++)
            WRITE_ONCE(lruvec->mm_state.stats[hist][i], 0);//将下一个gen的state设置为0
    }
}




static bool iterate_mm_list(struct lruvec *lruvec, struct lru_gen_mm_walk *walk,
                struct mm_struct **iter)
{
    bool first = false;    //当前gen的第一个
    bool last = true;    //当前gen的最后一个
    struct mm_struct *mm = NULL;
    struct mem_cgroup *memcg = lruvec_memcg(lruvec);
    struct lru_gen_mm_list *mm_list = get_mm_list(memcg);//初始化一个mm_list
    struct lru_gen_mm_state *mm_state = &lruvec->mm_state;

    /*try_to_inc_max_seq调用本函数时*iter为NULL
     * There are four interesting cases for this page table walker:
     * 1. It tries to start a new iteration of mm_list with a stale max_seq;
     *    there is nothing left to do.
     * 2. It's the first of the current generation, and it needs to reset
     *    the Bloom filter for the next generation.
     * 3. It reaches the end of mm_list, and it needs to increment
     *    mm_state->seq; the iteration is done.
     * 4. It's the last of the current generation, and it needs to reset the
     *    mm stats counters for the next generation.
     */
    spin_lock(&mm_list->lock);

    VM_WARN_ON_ONCE(mm_state->seq + 1 < walk->max_seq);
    VM_WARN_ON_ONCE(*iter && mm_state->seq > walk->max_seq);
    VM_WARN_ON_ONCE(*iter && !mm_state->nr_walkers);

    if (walk->max_seq <= mm_state->seq) {    //代表需要遍历的mm_struct都已经有进程在遍历了
        if (!*iter)//*iter=NULL内核中又有进程开始mglru老化，try_to_inc_max_seq中的do-while第一次循环
            last = false;//有进程开始遍历，但这个mm_list的所有mm_struct都有进程在遍历了，所以什么都不做
        goto done;//try_to_inc_max_seq中的do-while不是第一次循环，遍历完上一个mm_struct的页表之后发现其余mm_struct都已经有进程在遍历了，因此这个进程的遍历可以结束了
    }

    if (!mm_state->nr_walkers) {    //没有线程在遍历页表  对当前gen的第一次迭代
        VM_WARN_ON_ONCE(mm_state->head && mm_state->head != &mm_list->fifo);

        mm_state->head = mm_list->fifo.next;//进行迭代时，当前的列表节点
        first = true;    //第一次进行迭代
    }

    //遍历mm_state->head，找到一个合适的mm_struct
    while (!mm && mm_state->head != &mm_list->fifo) { //mm_state->head不是mm_list->fifo第一个成员
        mm = list_entry(mm_state->head, struct mm_struct, lru_gen.list);//通过mm_state->head拿到mm_struct

        mm_state->head = mm_state->head->next;//下一个mm_struct结构体

        /* force scan for those added after the last iteration */
        if (!mm_state->tail || mm_state->tail == &mm->lru_gen.list) {
            mm_state->tail = mm_state->head;
            walk->force_scan = true;
        }

        if (should_skip_mm(mm, walk))//mm_struct是否满足扫描条件
            mm = NULL;
    }

    if (mm_state->head == &mm_list->fifo)//到达mm_list的末尾（注意，这里不代表mm_list遍历完成，只代表需要遍历的mm_struct都已经有进程在遍历了）
        WRITE_ONCE(mm_state->seq, mm_state->seq + 1);
done:
    if (*iter && !mm)//见上文  try_to_inc_max_seq中的do-while不是第一次循环，遍历完上一个mm_struct的页表之后发现mm_struct都已经有进程在遍历了，因此这个进程的遍历可以结束了
        mm_state->nr_walkers--;
    if (!*iter && mm)
        mm_state->nr_walkers++;//*iter=NULL内核中又有进程开始mglru老化，try_to_inc_max_seq中的do-while第一次执行，并找到了合适的mm_struct，要对这个mm_struct进行页表遍历

    if (mm_state->nr_walkers)
        last = false;

    if (*iter || last)//最后一个进程遍历完成了
        reset_mm_stats(lruvec, walk, last);//将下一个gen的state设置为0

    spin_unlock(&mm_list->lock);

    if (mm && first)//第一次迭代要做的事
        reset_bloom_filter(lruvec, walk->max_seq + 1);//初始化walk->max_seq + 1的mm_stat.filters

    if (*iter)
        mmput_async(*iter);//上个mm_struct遍历完成，所以mm_users减一 

    *iter = mm;//下个要遍历的mm_struct

    return last;
}


static bool should_skip_mm(struct mm_struct *mm, struct lru_gen_mm_walk *walk)
{
    int type;
    unsigned long size = 0;
    struct pglist_data *pgdat = lruvec_pgdat(walk->lruvec);
    int key = pgdat->node_id % BITS_PER_TYPE(mm->lru_gen.bitmap);//拿到内存节点的比特位

    if (!walk->force_scan && !test_bit(key, &mm->lru_gen.bitmap))//不是强制扫描，并且该mm_struct没有被占用
        return true;

    clear_bit(key, &mm->lru_gen.bitmap);

    for (type = !walk->can_swap; type < ANON_AND_FILE; type++) {//获取mm_struct需要扫描的页面数量
        size += type ? get_mm_counter(mm, MM_FILEPAGES) :
                   get_mm_counter(mm, MM_ANONPAGES) +
                   get_mm_counter(mm, MM_SHMEMPAGES);
    }

    if (size < MIN_LRU_BATCH)//页面数量太小，跳过
        return true;

    return !mmget_not_zero(mm);//看是否有人在使用
}


static void inc_max_seq(struct lruvec *lruvec, bool can_swap, bool force_scan)
{
    int prev, next;
    int type, zone;
    struct lru_gen_struct *lrugen = &lruvec->lrugen;

    spin_lock_irq(&lruvec->lru_lock);

    VM_WARN_ON_ONCE(!seq_is_valid(lruvec));

    for (type = ANON_AND_FILE - 1; type >= 0; type--) {
        if (get_nr_gens(lruvec, type) != MAX_NR_GENS)    //min_seq和max_seq的代差 ！=4
            continue;

        VM_WARN_ON_ONCE(!force_scan && (type == LRU_GEN_FILE || can_swap));

        while (!inc_min_seq(lruvec, type, can_swap)) {    //先lrugen->min_seq[type] + 1
            spin_unlock_irq(&lruvec->lru_lock);
            cond_resched();
            spin_lock_irq(&lruvec->lru_lock);
        }
    }

    /*更新lru列表 active和inactive
     * Update the active/inactive LRU sizes for compatibility. Both sides of
     * the current max_seq need to be covered, since max_seq+1 can overlap
     * with min_seq[LRU_GEN_ANON] if swapping is constrained. And if they do
     * overlap, cold/hot inversion happens.
     */
    prev = lru_gen_from_seq(lrugen->max_seq - 1);//inactive
    next = lru_gen_from_seq(lrugen->max_seq + 1);//active

    for (type = 0; type < ANON_AND_FILE; type++) {
        for (zone = 0; zone < MAX_NR_ZONES; zone++) {
            enum lru_list lru = type * LRU_INACTIVE_FILE;
            long delta = lrugen->nr_pages[prev][type][zone] -
                     lrugen->nr_pages[next][type][zone];

            if (!delta)
                continue;

            __update_lru_size(lruvec, lru, zone, delta);
            __update_lru_size(lruvec, lru + LRU_ACTIVE, zone, -delta);
        }
    }

    for (type = 0; type < ANON_AND_FILE; type++)
        reset_ctrl_pos(lruvec, type, false);//将lrugen->refaulted和lrugen->evicted清零

    WRITE_ONCE(lrugen->timestamps[next], jiffies);//更新时间戳
    /* make sure preceding modifications appear */
    smp_store_release(&lrugen->max_seq, lrugen->max_seq + 1);//之前min_seq+1，再lrugen->max_seq + 1

    spin_unlock_irq(&lruvec->lru_lock);
}


static bool inc_min_seq(struct lruvec *lruvec, int type, bool can_swap)
{//
    int zone;
    int remaining = MAX_LRU_BATCH;
    struct lru_gen_struct *lrugen = &lruvec->lrugen;
    int new_gen, old_gen = lru_gen_from_seq(lrugen->min_seq[type]);

    if (type == LRU_GEN_ANON && !can_swap)
        goto done;

    /* prevent cold/hot inversion if force_scan is true */
    for (zone = 0; zone < MAX_NR_ZONES; zone++) {
        struct list_head *head = &lrugen->lists[old_gen][type][zone];

        while (!list_empty(head)) {
            struct folio *folio = lru_to_folio(head);

            VM_WARN_ON_ONCE_FOLIO(folio_test_unevictable(folio), folio);
            VM_WARN_ON_ONCE_FOLIO(folio_test_active(folio), folio);
            VM_WARN_ON_ONCE_FOLIO(folio_is_file_lru(folio) != type, folio);
            VM_WARN_ON_ONCE_FOLIO(folio_zonenum(folio) != zone, folio);

            new_gen = folio_inc_gen(lruvec, folio, false);
            list_move_tail(&folio->lru, &lrugen->lists[new_gen][type][zone]);

            if (!--remaining)
                return false;
        }
    }
done://最小实现只有这两行
    reset_ctrl_pos(lruvec, type, true); //更新lrugen->avg_refaulted和lrugen->avg_total
    WRITE_ONCE(lrugen->min_seq[type], lrugen->min_seq[type] + 1); //min_seq +1

    return true;
}


static bool try_to_inc_min_seq(struct lruvec *lruvec, bool can_swap)
{//与inc_min_seq函数并不相同，这里只对不同zone和type的min_seq做一个整理，不一定会使min_seq增加
    int gen, type, zone;
    bool success = false;
    struct lru_gen_struct *lrugen = &lruvec->lrugen;
    DEFINE_MIN_SEQ(lruvec);

    VM_WARN_ON_ONCE(!seq_is_valid(lruvec));

    /* find the oldest populated generation */
    for (type = !can_swap; type < ANON_AND_FILE; type++) {
        while (min_seq[type] + MIN_NR_GENS <= lrugen->max_seq) {//代差大于等于2代
            gen = lru_gen_from_seq(min_seq[type]);//获取相对gen

            for (zone = 0; zone < MAX_NR_ZONES; zone++) {
                if (!list_empty(&lrugen->lists[gen][type][zone]))
                    goto next;
            }
            /*遍历每个zone对应type的gen列表，若都为空，这将该type的min_seq+1
             *每个zone，每个type的min_seq不同，找每个type最小的那个            
            */
            min_seq[type]++;
        }
next:
        ;
    }

    /* see the comment on lru_gen_struct */
    if (can_swap) {//不同type=的min_seq的计算方法
        min_seq[LRU_GEN_ANON] = min(min_seq[LRU_GEN_ANON], min_seq[LRU_GEN_FILE]);//anon页面最小代取最小
        min_seq[LRU_GEN_FILE] = max(min_seq[LRU_GEN_ANON], lrugen->min_seq[LRU_GEN_FILE]);
    }

    for (type = !can_swap; type < ANON_AND_FILE; type++) {
        if (min_seq[type] == lrugen->min_seq[type])
            continue;

        reset_ctrl_pos(lruvec, type, true);
        WRITE_ONCE(lrugen->min_seq[type], min_seq[type]);//写入lrugen中
        success = true;
    }

    return success;
}


static void walk_mm(struct lruvec *lruvec, struct mm_struct *mm, struct lru_gen_mm_walk *walk)
{
    static const struct mm_walk_ops mm_walk_ops = {
        .test_walk = should_skip_vma,
        .p4d_entry = walk_pud_range,
    };

    int err;
    struct mem_cgroup *memcg = lruvec_memcg(lruvec);

    walk->next_addr = FIRST_USER_ADDRESS;

    do {
        err = -EBUSY;

        /* folio_update_gen() requires stable folio_memcg() */
        if (!mem_cgroup_trylock_pages(memcg))
            break;

        /* the caller might be holding the lock for write */
        if (mmap_read_trylock(mm)) {
            err = walk_page_range(mm, walk->next_addr, ULONG_MAX, &mm_walk_ops, walk);

            mmap_read_unlock(mm);
        }

        mem_cgroup_unlock_pages();

        if (walk->batched) {
            spin_lock_irq(&lruvec->lru_lock);
            reset_batch_size(lruvec, walk);
            spin_unlock_irq(&lruvec->lru_lock);
        }

        cond_resched();
    } while (err == -EAGAIN);
}

static int walk_pud_range(p4d_t *p4d, unsigned long start, unsigned long end,
              struct mm_walk *args)
{
    int i;
    pud_t *pud;
    unsigned long addr;
    unsigned long next;
    struct lru_gen_mm_walk *walk = args->private;

    VM_WARN_ON_ONCE(p4d_leaf(*p4d));

    pud = pud_offset(p4d, start & P4D_MASK);
restart:
    for (i = pud_index(start), addr = start; addr != end; i++, addr = next) {
        pud_t val = READ_ONCE(pud[i]);

        next = pud_addr_end(addr, end);

        if (!pud_present(val) || WARN_ON_ONCE(pud_leaf(val)))
            continue;

        walk_pmd_range(&val, addr, next, args);

        /* a racy check to curtail the waiting time */
        if (wq_has_sleeper(&walk->lruvec->mm_state.wait))
            return 1;

        if (need_resched() || walk->batched >= MAX_LRU_BATCH) {
            end = (addr | ~PUD_MASK) + 1;
            goto done;
        }
    }

    if (i < PTRS_PER_PUD && get_next_vma(P4D_MASK, PUD_SIZE, args, &start, &end))
        goto restart;

    end = round_up(end, P4D_SIZE);
done:
    if (!end || !args->vma)
        return 1;

    walk->next_addr = max(end, args->vma->vm_start);

    return -EAGAIN;
}
```
