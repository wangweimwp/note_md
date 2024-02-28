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

调用read读文件时

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
                        ->filemap_readahead        //若查找到的page是PG_readahead调用page_cache_async_readahead()进行异步预读
                            ->page_cache_async_ra        //异步读取page cache
                                ->ondemand_readahead    //最终调到aops->readahead，最终发送一些列bio
                        ->filemap_update_page    
                                ->filemap_read_folio    //最终调到aops->read_folio，最终发送一些列bio
        ->inc_syscr
```

**同步预读**：从存储器件读取多个page数据，一些page给当前的read（触发同步预读的read）使用，一些page留给将来的read使用。

**异步预读**：本次读的page纯粹是为将来准备的，目前用不到。
