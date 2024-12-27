**当调用close关闭文件fd时，内核会检测文件的应用数，若为0则会调用文件系统的
release成员关闭文件**

```c
fat_file_relase
    ->fat_flush_inodes
        ->sync_blockdev_nowait
            ->filemap_flush
                ->__filemap_fdatawrite_range
                    ->filemap_fdatawrite_wbc
                        ->do_writepages    //只会回写脏页，如果_refcount减为0则free，对其他pagecache无操作
```

**当drop_cache时**

```c
invalidate_mapping_pages
    ->invalidate_mapping_pagevec
        ->mapping_evict_folio    //判读页面有没有被mapped，若有被mapped则返回
            ->remove_mapping    //到这里说明页面没有mapped，直接释放即可，不需要try_to_unmap
    ->folio_batch_release     //释放页面，复合页和大页分开释放
//echo 1 > drop_cache会遍历文件系统struct adress_space里的page，将没有被mapped页
//read进来的page cache没有映射到用户空间，__map_count为0，所以drop_cache会清除page cache
```
缺页中断文件页面处理流程
```c
do_read_fault
	->__do_fault
		->vma->vm_ops->fault(ext4_filemap_fault)
			->filemap_fault
			->do_async_mmap_readahead or do_sync_mmap_readahead
	->alloc_set_pte//这里会增加__map_count，因此在drop_cache时不会被清掉
```

**drop_cache能drop掉hugetlbfs文件的pagecache吗？**

本分分析基于linux内核4.19.195版本。
我们知道，drop_cache有两个用途，第一个是清除掉page cache以释放出一些内存空间，第二个是调用shrinker的接口，这个接口会释放掉inode、dentry等文件系统相关的cache以及其他组件的一些东西。
今天突然脑袋发热的问了自己一个问题：


drop_cache能drop掉hugetlbfs文件的pagecache吗？

按drop_cache的原理来说是没问题的，drop_cache会去遍历每个super_block下的所有inode，并扫描其address_space中与page_cache相关的结构体（可能是radix_tree或者xarray），然后清理出page_cache占用的内存。
试了一下，发现根本drop不掉，why？
这里先给出答案：



drop不掉的本质原因是hugetlbfs是一个没有[backend](https://so.csdn.net/so/search?q=backend&spm=1001.2101.3001.7020)的文件系统，若这些cache被清除掉了，将无法恢复



那么，内核代码中是在哪里把hugetlbfs的drop_cache流程给拦住的呢？
答案是drop_caches_sysctl_handler()->drop_pagecache_sb()->invalidate_mapping_pages()->invalidate_inode_page()函数中，会判断相关page是否是能够write back的，

```c
/*
 * Safely invalidate one page from its pagecache mapping.
 * It only drops clean, unused pages. The page must be locked.
 *
 * Returns 1 if the page is successfully invalidated, otherwise 0.
 */
int invalidate_inode_page(struct page *page)
{
	struct address_space *mapping = page_mapping(page);
	if (!mapping)
		return 0;
	if (PageDirty(page) || PageWriteback(page)) //hugetlbfs的文件会因为PageWriteback(page)的判断而导致不能drop cache
		return 0;
	if (page_mapped(page))
		return 0;
	return invalidate_complete_page(mapping, page);
}

```







**调用read读文件时**

```c
ksys_read
    ->vfs_read
        ->new_sync_read
            ->file->f_op->read_iter(ext4_file_read_iter)
                ->filemap_read
                    ->filemap_get_pages
                        ->filemap_get_read_batch    //在address_space中逐页查找数据对应的page，若查找不到调用page_cache_sync_ra同步预读
                        ->page_cache_sync_readahead
                            ->page_cache_sync_ra        //同步读取page cache
                                ->ondemand_readahead
                                    ->do_page_cache_ra    //与page_cache_ra_order一样，最终调到aops->readahead，最终发送一些列bio
                                    ->page_cache_ra_order
                        ->filemap_create_folio
                            ->filemap_alloc_folio
                                ->folio_alloc
                            ->filemap_add_folio
                                ->folio_ref_add
                                ->folio_add_lru
                            ->filemap_read_folio
                            ->folio_batch_add
                        ->filemap_readahead        //若查找到的page是PG_readahead,说明用户读到了pagecache的末尾页，则page_cache_async_readahead()进行异步预读更多页进来
                            ->page_cache_async_ra        //异步读取page cache
                                ->ondemand_readahead    //最终调到aops->readahead，最终发送一些列bio
                        ->filemap_update_page    
                                ->filemap_read_folio    //最终调到aops->read_folio，最终发送一些列bio
        ->inc_syscr
```

**同步预读**：从存储器件读取多个page数据，一些page给当前的read（触发同步预读的read）使用，一些page留给将来的read使用。

**异步预读**：本次读的page纯粹是为将来准备的，目前用不到。

**写文件结束后清除page脏标记**
```c
//后台回写进程
wb_workfn
	->wb_do_writeback
		->wb_writeback
			->writeback_sb_inodes
				->__writeback_single_inode
					->do_writepages
						->ext4_writepages
							->write_cache_pages
								->clear_page_dirty_for_i
									->TestClearPageDirty(page)
								->ext4_writepage

//write系统调用
generic_perform_write
	->a_ops->write_begin	(ext4_da_write_begin)
	->iov_iter_copy_from_user_atomic
	->a_ops->write_end		(ext4_da_write_end)
		->ext4_da_write_inline_data_end
			->ext4_write_inline_data_end
				->SetPageUptodate(page);
				->ClearPageDirty(page);
```