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
```



调用read读文件时

```c
read文件
ksys_read
	->vfs_read
		->new_sync_read
			->file->f_op->read_iter(ext4_file_read_iter)
				->filemap_read
					->filemap_get_pages
						->filemap_get_read_batch
						->filemap_create_folio
							->filemap_alloc_folio
								->folio_alloc
							->filemap_add_folio
								->folio_ref_add    //增加 _ref_count计数
								->folio_add_lru
							->filemap_read_folio
							->folio_batch_add
						->filemap_readahead
						->filemap_update_page
		->inc_syscr
```


