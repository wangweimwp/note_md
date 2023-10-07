假设进程使用munmap释放虚拟内存区域，而这个虚拟内存区域可能映射到透明巨型页，可能是释放巨型页的一部分，也可能是释放整个巨型页。

函数unmap_single_vma负责删除一个虚拟内存区域，执行流程如图3.79所示。如果虚拟内存区域使用普通页或透明巨型页，把主要工作委托给函数unmap_page_range

`unmap_single_vma->unmap_page_range->zap_p4d_range->zap_pud_range`

```c
__split_huge_pmd_locked
    ->if (!vma_is_anonymous(vma))     //文件页面或共享匿名页面
        ....
    ->if (is_huge_zero_pmd(*pmd))    //全局零大页
        ....
    ->其他页面 
        ....
    ->pgtable_trans_huge_withdraw  //获取一个PTE页表
    ->pmd_populate            //把pte页表地址写到临时的pmd entry中
    ->for (i = 0, addr = haddr; i < HPAGE_PMD_NR; i++, addr += PAGE_SIZE)     
        ->根据上述三种页面的具体情况(可写？独占？young？dirty？等)，制作pte entry
        ->set_pte_at        //将做好的pte entry 写到pte也表中
    ->if (!pmd_migration)
        page_remove_rmap(page, vma, true);    //解除大页映射，把这个大页添加到deferred_split也表中，等待内存shrink时释放部分页
    ->if (freeze)
        put_page(page);
    ->pmd_populate(mm, pmd, pgtable);    //把做好的PTE页表地址写到pmd entry中
```

```c
static inline unsigned long zap_pud_range(struct mmu_gather *tlb,
                struct vm_area_struct *vma, p4d_t *p4d,
                unsigned long addr, unsigned long end,
                struct zap_details *details)
{
    pud_t *pud;
    unsigned long next;

    pud = pud_offset(p4d, addr);
    do {
        next = pud_addr_end(addr, end);
        if (pud_trans_huge(*pud) || pud_devmap(*pud)) {//p4d页表项的值是一个大页
            if (next - addr != HPAGE_PUD_SIZE) {//释放的不是一个完整的p4d大页
                mmap_assert_locked(tlb->mm);
                split_huge_pud(vma, pud, addr);//没有实质操作，只上报一些事件
            } else if (zap_huge_pud(tlb, vma, pud, addr))//是一个完整的p4d大页，目前没有实现
                goto next;
            /* fall through */
        }
        if (pud_none_or_clear_bad(pud))
            continue;
        next = zap_pmd_range(tlb, vma, pud, addr, next, details);
next:
        cond_resched();
    } while (pud++, addr = next, addr != end);

    return addr;
}


static inline unsigned long zap_pmd_range(struct mmu_gather *tlb,
                struct vm_area_struct *vma, pud_t *pud,
                unsigned long addr, unsigned long end,
                struct zap_details *details)
{
    pmd_t *pmd;
    unsigned long next;

    pmd = pmd_offset(pud, addr);
    do {
        next = pmd_addr_end(addr, end);
        if (is_swap_pmd(*pmd) || pmd_trans_huge(*pmd) || pmd_devmap(*pmd)) {//pmd-mapped THP
            if (next - addr != HPAGE_PMD_SIZE)//释放一部分
                __split_huge_pmd(vma, pmd, addr, false, NULL);
            else if (zap_huge_pmd(tlb, vma, pmd, addr))//释放整个pmd-mapped THP
                goto next;
            /* fall through */
        } else if (details && details->single_folio &&
               folio_test_pmd_mappable(details->single_folio) &&
               next - addr == HPAGE_PMD_SIZE && pmd_none(*pmd)) {//zap_huge_pmd之后的扫尾工作
            spinlock_t *ptl = pmd_lock(tlb->mm, pmd);
            /*
             * Take and drop THP pmd lock so that we cannot return
             * prematurely, while zap_huge_pmd() has cleared *pmd,
             * but not yet decremented compound_mapcount().
             */
            spin_unlock(ptl);
        }

        /*
         * Here there can be other concurrent MADV_DONTNEED or
         * trans huge page faults running, and if the pmd is
         * none or trans huge it can change under us. This is
         * because MADV_DONTNEED holds the mmap_lock in read
         * mode.
         */
        if (pmd_none_or_trans_huge_or_clear_bad(pmd))
            goto next;
        next = zap_pte_range(tlb, vma, pmd, addr, next, details);
next:
        cond_resched();
    } while (pmd++, addr = next, addr != end);

    return addr;
}

__split_huge_pmd->__split_huge_pmd_locked

static void __split_huge_pmd_locked(struct vm_area_struct *vma, pmd_t *pmd,
        unsigned long haddr, bool freeze)
{
    struct mm_struct *mm = vma->vm_mm;
    struct page *page;
    pgtable_t pgtable;
    pmd_t old_pmd, _pmd;
    bool young, write, soft_dirty, pmd_migration = false, uffd_wp = false;
    bool anon_exclusive = false, dirty = false;
    unsigned long addr;
    int i;

    VM_BUG_ON(haddr & ~HPAGE_PMD_MASK);
    VM_BUG_ON_VMA(vma->vm_start > haddr, vma);
    VM_BUG_ON_VMA(vma->vm_end < haddr + HPAGE_PMD_SIZE, vma);
    VM_BUG_ON(!is_pmd_migration_entry(*pmd) && !pmd_trans_huge(*pmd)
                && !pmd_devmap(*pmd));

    count_vm_event(THP_SPLIT_PMD);

    if (!vma_is_anonymous(vma)) {//文件映射,或共享匿名映射，内核选择了直接解除映射而不是将其分裂, 因为可能有多个进程共享，所以不能把巨型页分裂成普通页。
        old_pmd = pmdp_huge_clear_flush_notify(vma, haddr, pmd);//解除PMD映射
        /*
         * We are going to unmap this huge page. So
         * just go ahead and zap it
         */
        if (arch_needs_pgtable_deposit())
            zap_deposited_table(mm, pmd);
        if (vma_is_special_huge(vma))
            return;
        if (unlikely(is_pmd_migration_entry(old_pmd))) {//见，那里设置pmd entry为MIGRATION
            swp_entry_t entry;

            entry = pmd_to_swp_entry(old_pmd);
            page = pfn_swap_entry_to_page(entry);
        } else {
            page = pmd_page(old_pmd);
            if (!PageDirty(page) && pmd_dirty(old_pmd))
                set_page_dirty(page);
            if (!PageReferenced(page) && pmd_young(old_pmd))
                SetPageReferenced(page);
            page_remove_rmap(page, vma, true);
            put_page(page);
        }
        add_mm_counter(mm, mm_counter_file(page), -HPAGE_PMD_NR);
        return;
    }

    if (is_huge_zero_pmd(*pmd)) {//映射到大0页
        /*
         * FIXME: Do we want to invalidate secondary mmu by calling
         * mmu_notifier_invalidate_range() see comments below inside
         * __split_huge_pmd() ?
         *
         * We are going from a zero huge page write protected to zero
         * small page also write protected so it does not seems useful
         * to invalidate secondary mmu at this time.
         */
        return __split_huge_zero_page_pmd(vma, haddr, pmd);
    }

    /*
     * Up to this point the pmd is present and huge and userland has the
     * whole access to the hugepage during the split (which happens in
     * place). If we overwrite the pmd with the not-huge version pointing
     * to the pte here (which of course we could if all CPUs were bug
     * free), userland could trigger a small page size TLB miss on the
     * small sized TLB while the hugepage TLB entry is still established in
     * the huge TLB. Some CPU doesn't like that.
     * See http://support.amd.com/TechDocs/41322_10h_Rev_Gd.pdf, Erratum
     * 383 on page 105. Intel should be safe but is also warns that it's
     * only safe if the permission and cache attributes of the two entries
     * loaded in the two TLB is identical (which should be the case here).
     * But it is generally safer to never allow small and huge TLB entries
     * for the same virtual address to be loaded simultaneously. So instead
     * of doing "pmd_populate(); flush_pmd_tlb_range();" we first mark the
     * current pmd notpresent (atomically because here the pmd_trans_huge
     * must remain set at all times on the pmd until the split is complete
     * for this pmd), then we flush the SMP TLB and finally we write the
     * non-huge version of the pmd entry with pmd_populate.
     */
    old_pmd = pmdp_invalidate(vma, haddr, pmd);//将PMD页表项设置为不可用

    pmd_migration = is_pmd_migration_entry(old_pmd);
    if (unlikely(pmd_migration)) {
        swp_entry_t entry;

        entry = pmd_to_swp_entry(old_pmd);
        page = pfn_swap_entry_to_page(entry);
        write = is_writable_migration_entry(entry);
        if (PageAnon(page))
            anon_exclusive = is_readable_exclusive_migration_entry(entry);
        young = is_migration_entry_young(entry);
        dirty = is_migration_entry_dirty(entry);
        soft_dirty = pmd_swp_soft_dirty(old_pmd);
        uffd_wp = pmd_swp_uffd_wp(old_pmd);
    } else {
        page = pmd_page(old_pmd);
        if (pmd_dirty(old_pmd)) {
            dirty = true;
            SetPageDirty(page);
        }
        write = pmd_write(old_pmd);
        young = pmd_young(old_pmd);
        soft_dirty = pmd_soft_dirty(old_pmd);
        uffd_wp = pmd_uffd_wp(old_pmd);

        VM_BUG_ON_PAGE(!page_count(page), page);

        /*
         * Without "freeze", we'll simply split the PMD, propagating the
         * PageAnonExclusive() flag for each PTE by setting it for
         * each subpage -- no need to (temporarily) clear.
         *
         * With "freeze" we want to replace mapped pages by
         * migration entries right away. This is only possible if we
         * managed to clear PageAnonExclusive() -- see
         * set_pmd_migration_entry().
         *
         * In case we cannot clear PageAnonExclusive(), split the PMD
         * only and let try_to_migrate_one() fail later.
         *
         * See page_try_share_anon_rmap(): invalidate PMD first.
         */
        anon_exclusive = PageAnon(page) && PageAnonExclusive(page);
        if (freeze && anon_exclusive && page_try_share_anon_rmap(page))
            freeze = false;
        if (!freeze)
            page_ref_add(page, HPAGE_PMD_NR - 1);
    }

    /*
     * Withdraw the table only after we mark the pmd entry invalid.
     * This's critical for some architectures (Power).
     */
    pgtable = pgtable_trans_huge_withdraw(mm, pmd);//从PTE寄存器列表中取出一个来
    pmd_populate(mm, &_pmd, pgtable);//把PTE页表写到PMD entry中

    for (i = 0, addr = haddr; i < HPAGE_PMD_NR; i++, addr += PAGE_SIZE) {//制作每个PAGE的PTE entry
        pte_t entry, *pte;
        /*
         * Note that NUMA hinting access restrictions are not
         * transferred to avoid any possibility of altering
         * permissions across VMAs.
         */
        if (freeze || pmd_migration) {
            swp_entry_t swp_entry;
            if (write)
                swp_entry = make_writable_migration_entry(
                            page_to_pfn(page + i));
            else if (anon_exclusive)
                swp_entry = make_readable_exclusive_migration_entry(
                            page_to_pfn(page + i));
            else
                swp_entry = make_readable_migration_entry(
                            page_to_pfn(page + i));
            if (young)
                swp_entry = make_migration_entry_young(swp_entry);
            if (dirty)
                swp_entry = make_migration_entry_dirty(swp_entry);
            entry = swp_entry_to_pte(swp_entry);
            if (soft_dirty)
                entry = pte_swp_mksoft_dirty(entry);
            if (uffd_wp)
                entry = pte_swp_mkuffd_wp(entry);
        } else {
            entry = mk_pte(page + i, READ_ONCE(vma->vm_page_prot));
            entry = maybe_mkwrite(entry, vma);
            if (anon_exclusive)
                SetPageAnonExclusive(page + i);
            if (!young)
                entry = pte_mkold(entry);
            /* NOTE: this may set soft-dirty too on some archs */
            if (dirty)
                entry = pte_mkdirty(entry);
            /*
             * NOTE: this needs to happen after pte_mkdirty,
             * because some archs (sparc64, loongarch) could
             * set hw write bit when mkdirty.
             */
            if (!write)
                entry = pte_wrprotect(entry);
            if (soft_dirty)
                entry = pte_mksoft_dirty(entry);
            if (uffd_wp)
                entry = pte_mkuffd_wp(entry);
            page_add_anon_rmap(page + i, vma, addr, false);
        }
        pte = pte_offset_map(&_pmd, addr);
        BUG_ON(!pte_none(*pte));
        set_pte_at(mm, addr, pte, entry);
        pte_unmap(pte);
    }

    if (!pmd_migration)
        page_remove_rmap(page, vma, true);//如果不是swapd页面，减少page->_mapcount
    if (freeze)
        put_page(page);//将page释放掉

    smp_wmb(); /* make pte visible before pmd */
    pmd_populate(mm, pmd, pgtable);
}
```

当进程munmap大页的一部分时，并不会马上发生同步的大页分裂，因为在进程munmap的上下文进行大页分裂开销很高，现在是在反向映射时进行感知，通过deferred_split机制进行。

在内存压力紧张时系统回调各驱动注册的shrink接口，就会调用到deferred_split_scan。这个回调会进程大页分裂和普通页释放。

```c
deferred_split_scan
    ->split_folio
        ->split_huge_page_to_list
            ->unmap_folio没有拆分pmd的才去__split_huge_pmd_locked。差分过的不需要
                ->匿名页面调用->try_to_migrate_one->__split_huge_pmd_locked
                ->其他页面调用->try_to_unmap_one->__split_huge_pmd_locked
            ->__split_huge_page    分裂大页，把不用的page释放掉，把还在使用的page通过remove_migration_pte恢复页表映射
```

**问题：哪里设置page为SWP_MIGRATION_READ，后续确认页迁移时会不会设置？？？**

对于THP，split_huge_page_to_list->unmap_folio->try_to_migrate_one->是唯一调用try_to_migrate_one的路径

try_to_migrate_one中将freeza参数设置为true

在__split_huge_pmd_locked中，若freeze为true，则会设置页面为SWP_MIGRATION_READ

```c
static unsigned long deferred_split_scan(struct shrinker *shrink,
        struct shrink_control *sc)
{
    struct pglist_data *pgdata = NODE_DATA(sc->nid);
    struct deferred_split *ds_queue = &pgdata->deferred_split_queue;    //获取到deferred_split_queue链表
    unsigned long flags;
    LIST_HEAD(list);
    struct folio *folio, *next;
    int split = 0;

#ifdef CONFIG_MEMCG
    if (sc->memcg)
        ds_queue = &sc->memcg->deferred_split_queue;
#endif

    spin_lock_irqsave(&ds_queue->split_queue_lock, flags);
    /* Take pin on all head pages to avoid freeing them under us */
    list_for_each_entry_safe(folio, next, &ds_queue->split_queue,    //遍历deferred_split_queue, 对要处理的page放到list中
                            _deferred_list) {
        if (folio_try_get(folio)) {
            list_move(&folio->_deferred_list, &list);
        } else {
            /* We lost race with folio_put() */
            list_del_init(&folio->_deferred_list);
            ds_queue->split_queue_len--;
        }
        if (!--sc->nr_to_scan)
            break;
    }
    spin_unlock_irqrestore(&ds_queue->split_queue_lock, flags);

    list_for_each_entry_safe(folio, next, &list, _deferred_list) { //遍历list链表,获取page调用split_huge_page进行大页分裂
        if (!folio_trylock(folio))
            goto next;
        /* split_huge_page() removes page from list on success */
        if (!split_folio(folio))
            split++;        //分裂页面
        folio_unlock(folio);
next:
        folio_put(folio);//与folio_try_get对应，
    }

    spin_lock_irqsave(&ds_queue->split_queue_lock, flags);
    list_splice_tail(&list, &ds_queue->split_queue);//吧没有分裂的大页再放回到deferred_split_queue
    spin_unlock_irqrestore(&ds_queue->split_queue_lock, flags);

    /*
     * Stop shrinker if we didn't split any page, but the queue is empty.
     * This can happen if pages were freed under us.
     */
    if (!split && list_empty(&ds_queue->split_queue))
        return SHRINK_STOP;
    return split;
}



/*
 * This function splits huge page into normal pages. @page can point to any
 * subpage of huge page to split. Split doesn't change the position of @page.
 *
 * Only caller must hold pin on the @page, otherwise split fails with -EBUSY.
 * The huge page must be locked.
 *
 * If @list is null, tail pages will be added to LRU list, otherwise, to @list.
 *
 * Both head page and tail pages will inherit mapping, flags, and so on from
 * the hugepage.
 *
 * GUP pin and PG_locked transferred to @page. Rest subpages can be freed if
 * they are not mapped.
 *
 * Returns 0 if the hugepage is split successfully.
 * Returns -EBUSY if the page is pinned or if anon_vma disappeared from under
 * us.
 */
int split_huge_page_to_list(struct page *page, struct list_head *list)
{
    struct folio *folio = page_folio(page);
    struct deferred_split *ds_queue = get_deferred_split_queue(folio);//加到队列里的大页说明已经没有映射了，可以释放参考page_remove_rmap
    XA_STATE(xas, &folio->mapping->i_pages, folio->index);
    struct anon_vma *anon_vma = NULL;
    struct address_space *mapping = NULL;
    int extra_pins, ret;
    pgoff_t end;
    bool is_hzp;

    VM_BUG_ON_FOLIO(!folio_test_locked(folio), folio);
    VM_BUG_ON_FOLIO(!folio_test_large(folio), folio);

    is_hzp = is_huge_zero_page(&folio->page);
    VM_WARN_ON_ONCE_FOLIO(is_hzp, folio);
    if (is_hzp)
        return -EBUSY;

    if (folio_test_writeback(folio))
        return -EBUSY;

    if (folio_test_anon(folio)) {//文件页面
        /*
         * The caller does not necessarily hold an mmap_lock that would
         * prevent the anon_vma disappearing so we first we take a
         * reference to it and then lock the anon_vma for write. This
         * is similar to folio_lock_anon_vma_read except the write lock
         * is taken to serialise against parallel split or collapse
         * operations.
         */
        anon_vma = folio_get_anon_vma(folio);
        if (!anon_vma) {
            ret = -EBUSY;
            goto out;
        }
        end = -1;
        mapping = NULL;
        anon_vma_lock_write(anon_vma);
    } else {//文件页面或共享页面
        gfp_t gfp;

        mapping = folio->mapping;

        /* Truncated ? */
        if (!mapping) {
            ret = -EBUSY;
            goto out;
        }

        gfp = current_gfp_context(mapping_gfp_mask(mapping) &
                            GFP_RECLAIM_MASK);

        if (folio_test_private(folio) &&
                !filemap_release_folio(folio, gfp)) {
            ret = -EBUSY;
            goto out;
        }

        xas_split_alloc(&xas, folio, folio_order(folio), gfp);
        if (xas_error(&xas)) {
            ret = xas_error(&xas);
            goto out;
        }

        anon_vma = NULL;
        i_mmap_lock_read(mapping);

        /*
         *__split_huge_page() may need to trim off pages beyond EOF:
         * but on 32-bit, i_size_read() takes an irq-unsafe seqlock,
         * which cannot be nested inside the page tree lock. So note
         * end now: i_size itself may be changed at any moment, but
         * folio lock is good enough to serialize the trimming.
         */
        end = DIV_ROUND_UP(i_size_read(mapping->host), PAGE_SIZE);
        if (shmem_mapping(mapping))
            end = shmem_fallocend(mapping->host, end);
    }

    /*
     * Racy check if we can split the page, before unmap_folio() will
     * split PMDs
     */
    if (!can_split_folio(folio, &extra_pins)) {
        ret = -EAGAIN;
        goto out_unlock;
    }

    unmap_folio(folio);/*解映射，无论文件页面还是匿名页面，如还没有拆分pmd，__split_huge_pmd_locked，拆分pmd页表，如果页表已经拆分过了，就不需要再拆分了。*/

    /* block interrupt reentry in xa_lock and spinlock */
    local_irq_disable();
    if (mapping) {
        /*
         * Check if the folio is present in page cache.
         * We assume all tail are present too, if folio is there.
         */
        xas_lock(&xas);
        xas_reset(&xas);
        if (xas_load(&xas) != folio)
            goto fail;
    }

    /* Prevent deferred_split_scan() touching ->_refcount */
    spin_lock(&ds_queue->split_queue_lock);
    if (folio_ref_freeze(folio, 1 + extra_pins)) {
        if (!list_empty(&folio->_deferred_list)) {
            ds_queue->split_queue_len--;
            list_del(&folio->_deferred_list);
        }
        spin_unlock(&ds_queue->split_queue_lock);
        if (mapping) {
            int nr = folio_nr_pages(folio);

            xas_split(&xas, folio, folio_order(folio));
            if (folio_test_swapbacked(folio)) {
                __lruvec_stat_mod_folio(folio, NR_SHMEM_THPS,
                            -nr);
            } else {
                __lruvec_stat_mod_folio(folio, NR_FILE_THPS,
                            -nr);
                filemap_nr_thps_dec(mapping);
            }
        }

        __split_huge_page(page, list, end);//分裂大页，把不用的page释放掉，把还在使用的page通过remove_migration_pte恢复页表映射，无论pte entry是否设置了MIGRATION，都可以恢复
        ret = 0;
    } else {
        spin_unlock(&ds_queue->split_queue_lock);
fail:
        if (mapping)
            xas_unlock(&xas);
        local_irq_enable();
        remap_page(folio, folio_nr_pages(folio));//解除映射失败，通过remove_migration_pte恢复页表映射，
        ret = -EAGAIN;
    }

out_unlock:
    if (anon_vma) {    //匿名页面
        anon_vma_unlock_write(anon_vma);
        put_anon_vma(anon_vma);
    }
    if (mapping)        //文件页面或共享页面
        i_mmap_unlock_read(mapping);
out:
    xas_destroy(&xas);
    count_vm_event(!ret ? THP_SPLIT_PAGE : THP_SPLIT_PAGE_FAILED);
    return ret;
}

void free_transhuge_page(struct page *page)
{
    struct folio *folio = (struct folio *)page;
    struct deferred_split *ds_queue = get_deferred_split_queue(folio);
    unsigned long flags;

    spin_lock_irqsave(&ds_queue->split_queue_lock, flags);
    if (!list_empty(&folio->_deferred_list)) {
        ds_queue->split_queue_len--;
        list_del(&folio->_deferred_list);
    }
    spin_unlock_irqrestore(&ds_queue->split_queue_lock, flags);
    free_compound_page(page);
}
```
