## **挂载**

使用`swapon`命令启动swap分区时，内核调用对应系统调用

```c
SYSCALL_DEFINE2(swapon, const char __user *, specialfile, int, swap_flags)
```

`swapon`系统调用调用`init_swap_address_space`，初始化`struct address_space->a_ops = & swap_aops`,后续kswap线程向swap分区回写数据时会调用

## 回写

内存紧张时，`shrink_folio_list->pageout`调用`mapping->a_ops->writepage`方法回写页面，在swap_state.c文件中，（Only kswapd can writeback filesystem pages 留意`page_is_file_cache`）

```c
static const struct address_space_operations swap_aops = {
    .writepage    = swap_writepage,
    .dirty_folio    = noop_dirty_folio,
#ifdef CONFIG_MIGRATION
    .migrate_folio    = migrate_folio,
#endif
};

->__swap_writepage
	->swap_writepage_bdev_async
		->set_page_writeback


```


