```c
/*
 * By the time we get here, we already hold the mm semaphore
 *
 * The mmap_lock may have been released depending on flags and our
 * return value.  See filemap_fault() and __folio_lock_or_retry().
 */
static vm_fault_t __handle_mm_fault(struct vm_area_struct *vma,
        unsigned long address, unsigned int flags)
{
    struct vm_fault vmf = {
        .vma = vma,
        .address = address & PAGE_MASK,
        .real_address = address,
        .flags = flags,
        .pgoff = linear_page_index(vma, address),
        .gfp_mask = __get_fault_gfp_mask(vma),
    };
    struct mm_struct *mm = vma->vm_mm;
    unsigned long vm_flags = vma->vm_flags;
    pgd_t *pgd;
    p4d_t *p4d;
    vm_fault_t ret;

    pgd = pgd_offset(mm, address);
    p4d = p4d_alloc(mm, pgd, address);
    if (!p4d)
        return VM_FAULT_OOM;

    vmf.pud = pud_alloc(mm, p4d, address);//若是写时复制，页表已经建立，直接返回建立好的页表
    if (!vmf.pud)
        return VM_FAULT_OOM;
retry_pud:
    if (pud_none(*vmf.pud) &&
        hugepage_vma_check(vma, vm_flags, false, true, true)) {
        ret = create_huge_pud(&vmf);//PUD透明大页，暂不支持匿名PUD透明大页，调用vma->vm_ops->huge_fault  X86才支持PUD透明大页，一般挂载了文件系统才会给huge_fualt赋值（并检查映射巨页后是否超出VMA地址范围）
        if (!(ret & VM_FAULT_FALLBACK))//建立PUD大页失败，继续往下走
            return ret;
    } else {
        pud_t orig_pud = *vmf.pud;

        barrier();
        if (pud_trans_huge(orig_pud) || pud_devmap(orig_pud)) {//只有x86支持PUD透明大页

            /*
             * TODO once we support anonymous PUDs: NUMA case and
             * FAULT_FLAG_UNSHARE handling.
             */
            if ((flags & FAULT_FLAG_WRITE) && !pud_write(orig_pud)) {//写缺页中断，但是PUD标记不可写，写时复制情况
                ret = wp_huge_pud(&vmf, orig_pud);
                if (!(ret & VM_FAULT_FALLBACK))
                    return ret;
            } else {
                huge_pud_set_accessed(&vmf, orig_pud);
                return 0;
            }
        }
    }

    vmf.pmd = pmd_alloc(mm, vmf.pud, address);
    if (!vmf.pmd)
        return VM_FAULT_OOM;

    /* Huge pud page fault raced with pmd_alloc? */
    if (pud_trans_unstable(vmf.pud))//进程1建立巨页失败，开始pmd_alloc，若在pmd_alloc返回前，进程2在相同的PMD建立巨页成功，此时pmd_alloc会返回一个无效指针
        goto retry_pud;

    if (pmd_none(*vmf.pmd) &&
        hugepage_vma_check(vma, vm_flags, false, true, true)) {
        ret = create_huge_pmd(&vmf);        //若PMD表项为空，则建立PMD映射
        if (!(ret & VM_FAULT_FALLBACK))
            return ret;
    } else {
        vmf.orig_pmd = *vmf.pmd;

        barrier();
        if (unlikely(is_swap_pmd(vmf.orig_pmd))) {//PMD内容在交换分区
            VM_BUG_ON(thp_migration_supported() &&
                      !is_pmd_migration_entry(vmf.orig_pmd));
            if (is_pmd_migration_entry(vmf.orig_pmd))
                pmd_migration_entry_wait(mm, vmf.pmd);//wait for a migration entry to be remove, 见 那里设置pmd entry为MIGRATION
            return 0;
        }
        if (pmd_trans_huge(vmf.orig_pmd) || pmd_devmap(vmf.orig_pmd)) {
            if (pmd_protnone(vmf.orig_pmd) && vma_is_accessible(vma))
                return do_huge_pmd_numa_page(&vmf);

            if ((flags & (FAULT_FLAG_WRITE|FAULT_FLAG_UNSHARE)) &&    //非共享页面写时复制
                !pmd_write(vmf.orig_pmd)) {
                ret = wp_huge_pmd(&vmf);//写时复制PMD透明巨页
                if (!(ret & VM_FAULT_FALLBACK))
                    return ret;
            } else {
                huge_pmd_set_accessed(&vmf);
                return 0;
            }
        }
    }

    return handle_pte_fault(&vmf);
}



create_huge_pmd->do_huge_pmd_anonymous_page
vm_fault_t do_huge_pmd_anonymous_page(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    gfp_t gfp;
    struct folio *folio;
    unsigned long haddr = vmf->address & HPAGE_PMD_MASK;

    if (!transhuge_vma_suitable(vma, haddr))//检查映射整个PMD巨页是否超出了VMA（文件VMA或匿名VMA）的大小
        return VM_FAULT_FALLBACK;
    if (unlikely(anon_vma_prepare(vma)))//添加到反向映射系统
        return VM_FAULT_OOM;
    khugepaged_enter_vma(vma, vma->vm_flags);//添加到khugepaged系统

    if (!(vmf->flags & FAULT_FLAG_WRITE) &&
            !mm_forbids_zeropage(vma->vm_mm) &&
            transparent_hugepage_use_zero_page()) {//如果不是写操作并允许0页，这无需申请内存，直接映射全局0页
        pgtable_t pgtable;
        struct page *zero_page;
        vm_fault_t ret;
        pgtable = pte_alloc_one(vma->vm_mm);//用于存储 pmd_huge 的页表项
        if (unlikely(!pgtable))
            return VM_FAULT_OOM;
        zero_page = mm_get_huge_zero_page(vma->vm_mm);//拿到PMD全局0巨页（计数+1）
        if (unlikely(!zero_page)) {
            pte_free(vma->vm_mm, pgtable);
            count_vm_event(THP_FAULT_FALLBACK);
            return VM_FAULT_FALLBACK;
        }
        vmf->ptl = pmd_lock(vma->vm_mm, vmf->pmd);
        ret = 0;
        if (pmd_none(*vmf->pmd)) {//如果锁住页表以后发现页中间目录表项不是空表项，说明其他处理器正在竞争，已经分配并且映射到物理页，那么当前处理器放弃操作。
            ret = check_stable_address_space(vma->vm_mm);//检查mm_struct是否可靠，若OOM要杀掉这个mm_struct，会设置MMF_UNSTABLE
            if (ret) {
                spin_unlock(vmf->ptl);
                pte_free(vma->vm_mm, pgtable);
            } else if (userfaultfd_missing(vma)) {//用户处理缺页中断，与虚拟机迁移有关，这是一种后复制方案
                spin_unlock(vmf->ptl);
                pte_free(vma->vm_mm, pgtable);
                ret = handle_userfault(vmf, VM_UFFD_MISSING);
                VM_BUG_ON(ret & VM_FAULT_FALLBACK);
            } else {
                set_huge_zero_page(pgtable, vma->vm_mm, vma,
                           haddr, vmf->pmd, zero_page);//构建PMD巨页，后续和khugepaged系统一起分析？？？
                update_mmu_cache_pmd(vma, vmf->address, vmf->pmd);
                spin_unlock(vmf->ptl);
            }
        } else {
            spin_unlock(vmf->ptl);
            pte_free(vma->vm_mm, pgtable);
        }
        return ret;
    }
    gfp = vma_thp_gfp_mask(vma);
    folio = vma_alloc_folio(gfp, HPAGE_PMD_ORDER, vma, haddr, true);//若是写操作，则申请HPAGE_PMD_ORDER个页面
    if (unlikely(!folio)) {
        count_vm_event(THP_FAULT_FALLBACK);
        return VM_FAULT_FALLBACK;
    }
    return __do_huge_pmd_anonymous_page(vmf, &folio->page, gfp);
}



static vm_fault_t __do_huge_pmd_anonymous_page(struct vm_fault *vmf,
            struct page *page, gfp_t gfp)
{
    struct vm_area_struct *vma = vmf->vma;
    pgtable_t pgtable;
    unsigned long haddr = vmf->address & HPAGE_PMD_MASK;
    vm_fault_t ret = 0;

    VM_BUG_ON_PAGE(!PageCompound(page), page);

    if (mem_cgroup_charge(page_folio(page), vma->vm_mm, gfp)) {//cgroup内存记账
        put_page(page);
        count_vm_event(THP_FAULT_FALLBACK);
        count_vm_event(THP_FAULT_FALLBACK_CHARGE);
        return VM_FAULT_FALLBACK;
    }
    cgroup_throttle_swaprate(page, gfp);

    pgtable = pte_alloc_one(vma->vm_mm);
    if (unlikely(!pgtable)) {
        ret = VM_FAULT_OOM;
        goto release;
    }

    clear_huge_page(page, vmf->address, HPAGE_PMD_NR);//将申请到的folio清零
    /*
     * The memory barrier inside __SetPageUptodate makes sure that
     * clear_huge_page writes become visible before the set_pmd_at()
     * write.
     */
    __SetPageUptodate(page);

    vmf->ptl = pmd_lock(vma->vm_mm, vmf->pmd);
    if (unlikely(!pmd_none(*vmf->pmd))) {
        goto unlock_release;
    } else {
        pmd_t entry;

        ret = check_stable_address_space(vma->vm_mm);
        if (ret)
            goto unlock_release;

        /* Deliver the page fault to userland */
        if (userfaultfd_missing(vma)) {//用户处理缺页中断情况
            spin_unlock(vmf->ptl);
            put_page(page);
            pte_free(vma->vm_mm, pgtable);
            ret = handle_userfault(vmf, VM_UFFD_MISSING);
            VM_BUG_ON(ret & VM_FAULT_FALLBACK);
            return ret;
        }

        entry = mk_huge_pmd(page, vma->vm_page_prot);
        entry = maybe_pmd_mkwrite(pmd_mkdirty(entry), vma);
        page_add_new_anon_rmap(page, vma, haddr);    //加入反向映射系统
        lru_cache_add_inactive_or_unevictable(page, vma);//将folio加入folio
        pgtable_trans_huge_deposit(vma->vm_mm, vmf->pmd, pgtable);
        set_pmd_at(vma->vm_mm, haddr, vmf->pmd, entry);
        update_mmu_cache_pmd(vma, vmf->address, vmf->pmd);
        add_mm_counter(vma->vm_mm, MM_ANONPAGES, HPAGE_PMD_NR);
        mm_inc_nr_ptes(vma->vm_mm);
        spin_unlock(vmf->ptl);
        count_vm_event(THP_FAULT_ALLOC);
        count_memcg_event_mm(vma->vm_mm, THP_FAULT_ALLOC);
    }

    return 0;
unlock_release:
    spin_unlock(vmf->ptl);
release:
    if (pgtable)
        pte_free(vma->vm_mm, pgtable);
    put_page(page);
    return ret;

}


static inline vm_fault_t wp_huge_pmd(struct vm_fault *vmf)
{
    const bool unshare = vmf->flags & FAULT_FLAG_UNSHARE;
    vm_fault_t ret;

    if (vma_is_anonymous(vmf->vma)) {//匿名页面
        if (likely(!unshare) &&
            userfaultfd_huge_pmd_wp(vmf->vma, vmf->orig_pmd))
            return handle_userfault(vmf, VM_UFFD_WP);
        return do_huge_pmd_wp_page(vmf);//写时复制，若原来映射的是全局0页，直接__split_huge_pmd，若不是则判断巨页的_ref_count判断巨页是否可重用，若不行则__split_huge_pmd。总之若透明巨页发生写时复制，一般情况下会分裂成小页
    }

    if (vmf->vma->vm_flags & (VM_SHARED | VM_MAYSHARE)) {//文件页面
        if (vmf->vma->vm_ops->huge_fault) {
            ret = vmf->vma->vm_ops->huge_fault(vmf, PE_SIZE_PMD);
            if (!(ret & VM_FAULT_FALLBACK))
                return ret;
        }
    }

    /* COW or write-notify handled on pte level: split pmd. */
    __split_huge_pmd(vmf->vma, vmf->pmd, vmf->address, false, NULL);

    return VM_FAULT_FALLBACK;
}
```


