# 前言

## delay alloc模式

写入延迟‌：应用程序执行write()系统调用时，数据仅被写入‌Page Cache内存页缓存‌，此时文件系统‌不立即分配磁盘块‌‌

‌元数据延迟‌：系统暂不建立逻辑块到物理块的映射关系（buffer_head状态保持“未映射”），仅标记需要后续分配‌

‌分配时机‌：
    实际磁盘块分配延迟到以下时机触发：
    脏页比例达到阈值（由vm.dirty_ratio控制）
    定时刷新机制激活
    主动调用sync()或fsync()

通过合并连续写入请求，结合‌mballoc多块分配器‌一次性分配连续物理块‌

| ‌特性      | ‌	‌延迟分配 (Delay Allocation) | 立即分配 (No Delayed Allocation)     |
| :---        | :---- | :--- |
| ‌分配时机      | 数据写入 Page Cache 后‌暂不分配磁盘块‌；实际分配延迟到脏页回写时（如 sync、fsync 或内存压力触发）‌       | 数据写入 Page Cache 时‌立即分配磁盘块‌，建立逻辑块到物理块的映射   |
| ‌碎片控制  | 通过合并连续写入请求，结合 ‌mballoc 多块分配器‌一次性分配连续物理块，显著减少碎片 ‌  | 每次小写入均触发独立分配，易产生磁盘碎片 ‌ |
| CPU 开销 |减少元数据频繁更新，降低 CPU 负载 ‌ | 每次写入均需分配块并更新元数据，CPU 开销较高|
|写入性能|机械硬盘场景下顺序写吞吐量提升 5-10 倍（尤其大文件连续写入）‌|随机写入性能较差，频繁分配导致 I/O 效率低下|  
|数据一致性风险|崩溃时未回写的延迟分配数据可能丢失，依赖 fsync() 保障安全|分配即落盘（元数据），崩溃风险较低但性能牺牲大|


*No Delayed Allocation 与 direct IO区别在于是否经过page cache这一层*

# ext4_da_write_begin
```c
static int ext4_da_write_begin(struct file *file, struct address_space *mapping,
			       loff_t pos, unsigned len,
			       struct folio **foliop, void **fsdata)
{
	int ret, retries = 0;
	struct folio *folio;
	pgoff_t index;
	struct inode *inode = mapping->host;

	if (unlikely(ext4_forced_shutdown(inode->i_sb)))//ext4挂载是否异常
		return -EIO;

	index = pos >> PAGE_SHIFT;

	/*两种情况切换到nodelalloc模式。
		1.磁盘剩余的freeblock不足，先触发回写，仍不足则切到nodelalloc模式
		2.ext4挂载时使用了EXT4_STATE_VERITY_IN_PROGRESS特性
	*/
	if (ext4_nonda_switch(inode->i_sb) || ext4_verity_in_progress(inode)) {
		*fsdata = (void *)FALL_BACK_TO_NONDELALLOC;
		return ext4_write_begin(file, mapping, pos,
					len, foliop, fsdata);
	}
	*fsdata = (void *)0;
	trace_ext4_da_write_begin(inode, pos, len);

	/*INLINE_DATA情况，对于小文件，数据直接存储到inode结构体中
	*/
	if (ext4_test_inode_state(inode, EXT4_STATE_MAY_INLINE_DATA)) {
		ret = ext4_da_write_inline_data_begin(mapping, inode, pos, len,
						      foliop, fsdata);
		if (ret < 0)
			return ret;
		if (ret == 1)
			return 0;
	}

retry:
	folio = __filemap_get_folio(mapping, index, FGP_WRITEBEGIN,
			mapping_gfp_mask(mapping));
	if (IS_ERR(folio))
		return PTR_ERR(folio);

	/*获取folio的buffer head，对每个buffer head分别调用ext4_da_get_block_prep*/
	ret = ext4_block_write_begin(NULL, folio, pos, len,
				     ext4_da_get_block_prep);
	if (ret < 0) {
		folio_unlock(folio);
		folio_put(folio);
		/*
		 * block_write_begin may have instantiated a few blocks
		 * outside i_size.  Trim these off again. Don't need
		 * i_size_read because we hold inode lock.
		 */
		if (pos + len > inode->i_size)
			ext4_truncate_failed_write(inode);

		if (ret == -ENOSPC &&
		    ext4_should_retry_alloc(inode->i_sb, &retries))
			goto retry;
		return ret;
	}

	*foliop = folio;
	return ret;
}

int ext4_block_write_begin(handle_t *handle, struct folio *folio,
			   loff_t pos, unsigned len,
			   get_block_t *get_block)
{
	unsigned from = pos & (PAGE_SIZE - 1);
	unsigned to = from + len;
	struct inode *inode = folio->mapping->host;
	unsigned block_start, block_end;
	sector_t block;
	int err = 0;
	unsigned blocksize = inode->i_sb->s_blocksize;
	unsigned bbits;
	struct buffer_head *bh, *head, *wait[2];
	int nr_wait = 0;
	int i;
	bool should_journal_data = ext4_should_journal_data(inode);

	BUG_ON(!folio_test_locked(folio));
	BUG_ON(from > PAGE_SIZE);
	BUG_ON(to > PAGE_SIZE);
	BUG_ON(from > to);

	/*获取对应的buffer head*/
	head = folio_buffers(folio);
	if (!head)
		head = create_empty_buffers(folio, blocksize, 0);
	bbits = ilog2(blocksize);
	block = (sector_t)folio->index << (PAGE_SHIFT - bbits);

	for (bh = head, block_start = 0; bh != head || !block_start;
	    block++, block_start = block_end, bh = bh->b_this_page) {
		block_end = block_start + blocksize;
		if (block_end <= from || block_start >= to) {//当前block全部长度落在[form,to]区间之外
		[	if (folio_test_uptodate(folio)) {
				set_buffer_uptodate(bh);
			}
			continue;
		}
		if (buffer_new(bh))
			clear_buffer_new(bh);
		if (!buffer_mapped(bh)) {//buffer head还没有映射到磁盘
			WARN_ON(bh->b_size != blocksize);
			err = get_block(inode, block, bh, 1);
			if (err)
				break;
			if (buffer_new(bh)) {//buffer head的操盘映射时新创建的
				/*
				 * We may be zeroing partial buffers or all new
				 * buffers in case of failure. Prepare JBD2 for
				 * that.
				 */
				if (should_journal_data)
					do_journal_get_write_access(handle,
								    inode, bh);
				if (folio_test_uptodate(folio)) {
					/*
					 * Unlike __block_write_begin() we leave
					 * dirtying of new uptodate buffers to
					 * ->write_end() time or
					 * folio_zero_new_buffers().
					 */
					set_buffer_uptodate(bh);
					continue;
				}
				if (block_end > to || block_start < from)//当前block落在[form,to]区间之外的部分清零
					folio_zero_segments(folio, to,
							    block_end,
							    block_start, from);
				continue;
			}
		}
		if (folio_test_uptodate(folio)) {
			set_buffer_uptodate(bh);
			continue;
		}
		if (!buffer_uptodate(bh) && !buffer_delay(bh) &&
		    !buffer_unwritten(bh) &&
		    (block_start < from || block_end > to)) { //当前block部分长度落在[form,to]区间之内
			ext4_read_bh_lock(bh, 0, false);
			wait[nr_wait++] = bh;//部分长度落在[form,to]区间之内的block有且只有2个
		}
	}
	/*
	 * If we issued read requests, let them complete.
	 */
	for (i = 0; i < nr_wait; i++) {
		wait_on_buffer(wait[i]);
		if (!buffer_uptodate(wait[i]))
			err = -EIO;
	}
	if (unlikely(err)) {
		if (should_journal_data)
			ext4_journalled_zero_new_buffers(handle, inode, folio,
							 from, to);
		else
			folio_zero_new_buffers(folio, from, to);
	} else if (fscrypt_inode_uses_fs_layer_crypto(inode)) {
		for (i = 0; i < nr_wait; i++) {
			int err2;

			err2 = fscrypt_decrypt_pagecache_blocks(folio,
						blocksize, bh_offset(wait[i]));
			if (err2) {
				clear_buffer_uptodate(wait[i]);
				err = err2;
			}
		}
	}

	return err;
}
```