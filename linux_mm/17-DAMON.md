# 用户使用案例
## 1. DAMON的madvise用法，动态监测页面访问情况，对符合要求的页面调用madvise系统调用
```text

以下命令应用了一种策略，其内容为：“如果某个大小在[4KiB，8KiB]范围内的内存区域，在[10，20]的聚合时间段内每秒的访问次数处于[0，5]之间，则对该区域进行页面置换操作。在进行页面置换时，每秒的耗时上限为10毫秒，并且每次页面置换的内存大小上限为1GiB。在上述限制条件下，应优先置换较旧的内存区域。此外，每5秒检查一次系统的可用内存率，当可用内存率低于50%时启动监测和页面置换操作，但若可用内存率高于60%或低于30%则停止该操作。”

# cd <sysfs>/kernel/mm/damon/admin
# # populate directories
# echo 1 > kdamonds/nr_kdamonds; echo 1 > kdamonds/0/contexts/nr_contexts;
# echo 1 > kdamonds/0/contexts/0/schemes/nr_schemes
# cd kdamonds/0/contexts/0/schemes/0
# # set the basic access pattern and the action
# echo 4096 > access_pattern/sz/min
# echo 8192 > access_pattern/sz/max
# echo 0 > access_pattern/nr_accesses/min
# echo 5 > access_pattern/nr_accesses/max
# echo 10 > access_pattern/age/min
# echo 20 > access_pattern/age/max
# echo pageout > action
# # set quotas
# echo 10 > quotas/ms
# echo $((1024*1024*1024)) > quotas/bytes
# echo 1000 > quotas/reset_interval_ms
# # set watermark
# echo free_mem_rate > watermarks/metric
# echo 5000000 > watermarks/interval_us
# echo 600 > watermarks/high
# echo 500 > watermarks/mid
# echo 300 > watermarks/low

```

## 2. DAMON的LRU排序用法，对符合要求的页面，调整这些页面在LRU上的位置。
```text
如下是一个运行时的命令示例，使DAMON_LRU_SORT查找访问频率超过50%的区域并对其进行LRU的优先级的提升，同时降低那些超过120秒无人访问的内存区域的优先级。优先级的处理被限制在最多1%的CPU以避免DAMON_LRU_SORT消费过多CPU时间。在系统空闲内存超过50%时DAMON_LRU_SORT停止工作，并在低于40%时重新开始工作。如果DAMON_RECLAIM没有取得进展且空闲内存低于20%，再次让DAMON_LRU_SORT停止工作，以此回退到以LRU链表为基础以页面为单位的内存回收上。 

# cd /sys/modules/damon_lru_sort/parameters
# echo 500 > hot_thres_access_freq
# echo 120000000 > cold_min_age
# echo 10 > quota_ms
# echo 1000 > quota_reset_interval_ms
# echo 500 > wmarks_high
# echo 400 > wmarks_mid
# echo 200 > wmarks_low
# echo Y > enabled


问题1：问频率超过50%怎么理解？
在damon_hot_score函数中
问题2：进行LRU的优先级的提升怎么理解？

//lru_sort会新建一个paddr的检测方案
kdamond_fn
	->kdamond_apply_schemes
		->damon_do_apply_schemes
			->damos_apply_scheme
				->ops.apply_scheme = damon_pa_apply_scheme
					->damon_pa_mark_accessed
						->damon_pa_mark_accessed_or_deactivate
							->if (mark_accessed)
								folio_mark_accessed(folio);
							else
								folio_deactivate(folio);

```

## 3. DAMON的reclaim用法 超过一定时间未被访问，回收这些页面
```text
下面的运行示例命令使DAMON_RECLAIM找到30秒或更长时间没有访问的内存区域并“回收”？为了避免DAMON_RECLAIM在分页操作中消耗过多的CPU时间，回收被限制在每秒1GiB以内。它还要求DAMON_RECLAIM在系统的可用内存率超过50%时不做任何事情，但如果它低于40%时就开始真正的工作。如果DAMON_RECLAIM没有取得进展，因此空闲内存率低于20%，它会要求DAMON_RECLAIM再次什么都不做，这样我们就可以退回到基于LRU列表的页面粒度回收了::

# cd /sys/module/damon_reclaim/parameters
# echo 30000000 > min_age
# echo $((1 * 1024 * 1024 * 1024)) > quota_sz
# echo 1000 > quota_reset_interval_ms
# echo 500 > wmarks_high
# echo 400 > wmarks_mid
# echo 200 > wmarks_low
# echo Y > enabled

reclaim同样采用paddr策略，使用PAGEOUT操作
```

# 代码结构

```c
//对/sys/kernel/mm/damon/admin/kdamons目录下文件操作
//nr_kdamonds文件(可读写)
nr_kdamonds_store
	->damon_sysfs_kdamonds_add_dirs		//输入kdamod进程个数
		->kobject_init_and_add			//分别初始各个kdamod进程 建立state和pid文件节点
		->damon_sysfs_kdamond_add_dirs	//分别初始各个kdamod进程 建立contexts文件夹
		
//state文件(可读写)
state_store
	->damon_sysfs_handle_cmd
		->damon_sysfs_turn_damon_on		//开启kdamod进程
			->damon_sysfs_build_ctx
				->damon_new_ctx
				->damon_sysfs_apply_inputs
					->damon_select_ops
					->damon_sysfs_set_attrs
					->damon_sysfs_add_targets
					->damon_sysfs_add_schemes
			->damon_start
				->使能kdamon内核线程(kdamond_fn)

//vaddr模式				
kdamond_fn
	->ops.prepare_access_checks = damon_va_prepare_access_checks
		->__damon_va_prepare_access_check
			->damon_va_mkold
				->walk_page_range
					->damon_mkold_pmd_entry
						->damon_ptep_mkold
							->damon_ptep_mkold
								->pmdp_clear_young_notify
								->folio_set_young
								->folio_set_idle
	->ops.check_accesses = damon_va_check_accesses
		->__damon_va_check_access
			->damon_va_young
				->damon_young_pmd_entry
					->if (pmd_young(pmde) || !folio_test_idle(folio)
	->ops.reset_aggregated = NULL
	->ops.update = damon_va_update		
	->ops.cleanup = NULL


```

# 代码详解


## vaddr模式
```c

void damon_ptep_mkold(pte_t *pte, struct vm_area_struct *vma, unsigned long addr)
{
	struct folio *folio = damon_get_folio(pte_pfn(ptep_get(pte)));

	if (!folio)
		return;

	if (ptep_clear_young_notify(vma, addr, pte))//清除掉PET的young标记
		folio_set_young(folio);//设置struct folio的young标记

	folio_set_idle(folio);//设置struct folio的idle标记
	folio_put(folio);
}

static int damon_mkold_pmd_entry(pmd_t *pmd, unsigned long addr,
		unsigned long next, struct mm_walk *walk)
{
	pte_t *pte;
	pmd_t pmde;
	spinlock_t *ptl;

	if (pmd_trans_huge(pmdp_get(pmd))) {
		ptl = pmd_lock(walk->mm, pmd);
		pmde = pmdp_get(pmd);

		if (!pmd_present(pmde)) {
			spin_unlock(ptl);
			return 0;
		}

		if (pmd_trans_huge(pmde)) {
			damon_pmdp_mkold(pmd, walk->vma, addr);
			spin_unlock(ptl);
			return 0;
		}
		spin_unlock(ptl);
	}

	pte = pte_offset_map_lock(walk->mm, pmd, addr, &ptl);
	if (!pte) {
		walk->action = ACTION_AGAIN;
		return 0;
	}
	if (!pte_present(ptep_get(pte)))
		goto out;
	damon_ptep_mkold(pte, walk->vma, addr);//标记为old
out:
	pte_unmap_unlock(pte, ptl);
	return 0;
}

#ifdef CONFIG_HUGETLB_PAGE
static void damon_hugetlb_mkold(pte_t *pte, struct mm_struct *mm,
				struct vm_area_struct *vma, unsigned long addr)
{
	bool referenced = false;
	pte_t entry = huge_ptep_get(mm, addr, pte);
	struct folio *folio = pfn_folio(pte_pfn(entry));
	unsigned long psize = huge_page_size(hstate_vma(vma));

	folio_get(folio);

	if (pte_young(entry)) {
		referenced = true;
		entry = pte_mkold(entry);
		set_huge_pte_at(mm, addr, pte, entry, psize);
	}

	if (mmu_notifier_clear_young(mm, addr,
				     addr + huge_page_size(hstate_vma(vma))))
		referenced = true;

	if (referenced)
		folio_set_young(folio);

	folio_set_idle(folio);
	folio_put(folio);
}

static int damon_mkold_hugetlb_entry(pte_t *pte, unsigned long hmask,
				     unsigned long addr, unsigned long end,
				     struct mm_walk *walk)
{
	struct hstate *h = hstate_vma(walk->vma);
	spinlock_t *ptl;
	pte_t entry;

	ptl = huge_pte_lock(h, walk->mm, pte);
	entry = huge_ptep_get(walk->mm, addr, pte);
	if (!pte_present(entry))
		goto out;

	damon_hugetlb_mkold(pte, walk->mm, walk->vma, addr);

out:
	spin_unlock(ptl);
	return 0;
}
#else
#define damon_mkold_hugetlb_entry NULL
#endif /* CONFIG_HUGETLB_PAGE */

static const struct mm_walk_ops damon_mkold_ops = {
	.pmd_entry = damon_mkold_pmd_entry,
	.hugetlb_entry = damon_mkold_hugetlb_entry,
	.walk_lock = PGWALK_RDLOCK,
};

static void damon_va_mkold(struct mm_struct *mm, unsigned long addr)
{
	mmap_read_lock(mm);
	walk_page_range(mm, addr, addr + 1, &damon_mkold_ops, NULL);
	mmap_read_unlock(mm);
}

/*
 * Functions for the access checking of the regions
 */

static void __damon_va_prepare_access_check(struct mm_struct *mm,
					struct damon_region *r)
{
	r->sampling_addr = damon_rand(r->ar.start, r->ar.end);//随机抽取一个地址

	damon_va_mkold(mm, r->sampling_addr);//想页面标记为old
}

	
static void damon_va_prepare_access_checks(struct damon_ctx *ctx)
{
	struct damon_target *t;
	struct mm_struct *mm;
	struct damon_region *r;

	damon_for_each_target(t, ctx) {
		mm = damon_get_mm(t);//拿到target_pid的struct mm_struct
		if (!mm)
			continue;
		damon_for_each_region(r, t)
			__damon_va_prepare_access_check(mm, r);
		mmput(mm);
	}
}
	
```

## paddr
```c
//paddr模式				
kdamond_fn
	->ops.prepare_access_checks = damon_pa_prepare_access_checks
		->__damon_pa_prepare_access_check
			->damon_pa_mkold
				->damon_folio_mkold  //通过物理地址拿到PFN然后拿到struct folio
					->rmap_walk		//通过反向映射，找到映射这个页的所有进程
						->damon_folio_mkold_one
							->ptep_clear_young_notify
							->folio_set_young(folio);
							->folio_set_idle(folio);
	->ops.check_accesses = damon_pa_check_accesses
		->__damon_pa_check_access
			->damon_pa_young
				->damon_folio_young
					->damon_folio_young_one
						->*accessed = pte_young(ptep_get(pvmw.pte)) || !folio_test_idle(folio)
	->ops.reset_aggregated = NULL
	->ops.update = NULL		
	->ops.cleanup = NULL
```


# 社区动态
https://lore.kernel.org/all/20250712195016.151108-1-sj@kernel.org/
用damon_call替代callback，将任务加到列表中，按照先来后到的数执行，这是否会造成任务优先级的问题。