```c
ext4_file_write_iter
	->ext4_dio_write_iter
		->ext4_dio_write_checks
		->iomap_dio_rw
			->__iomap_dio_rw
				->iomap_iter
					->ops->iomap_end	ext4_iomap_end
					->ops->iomap_begin   ext4_iomap_begin
				->iomap_dio_iter
					->iomap_dio_submit_bi
						->submit_bio
			->iomap_dio_complete
				->dops->end_io	ext4_dio_write_end_io	

ext4_iomap_begin
	->ext4_iomap_alloc
		->ext4_map_blocks
or	->ext4_map_blocks
		->ext4_map_query_blocks
		->ext4_map_create_block
			->ext4_ext_map_blocks
			->ext4_issue_zeroout
			->ext4_es_lookup_extent
			->ext4_es_insert_extent
	->ext4_set_iomap
		
//通用
struct iomap {
	u64			addr; /* disk offset of mapping, bytes */
	loff_t			offset;	/* file offset of mapping, bytes */
	u64			length;	/* length of mapping, bytes */
	u16			type;	/* type of mapping */
	u16			flags;	/* flags for mapping */
	struct block_device	*bdev;	/* block device for I/O */
	struct dax_device	*dax_dev; /* dax_dev for dax operations */
	void			*inline_data;
	void			*private; /* filesystem private */
	const struct iomap_folio_ops *folio_ops;
	u64			validity_cookie; /* used with .iomap_valid() */
};


/**
 * struct iomap_iter - Iterate through a range of a file
 * @inode: Set at the start of the iteration and should not change.
 * @pos: The current file position we are operating on.  It is updated by
 *	calls to iomap_iter().  Treat as read-only in the body.
 * @len: The remaining length of the file segment we're operating on.
 *	It is updated at the same time as @pos.
 * @processed: The number of bytes processed by the body in the most recent
 *	iteration, or a negative errno. 0 causes the iteration to stop.
 * @flags: Zero or more of the iomap_begin flags above.
 * @iomap: Map describing the I/O iteration
 * @srcmap: Source map for COW operations
 */
struct iomap_iter {
	struct inode *inode;
	loff_t pos;
	u64 len;
	s64 processed;
	unsigned flags;
	struct iomap iomap;
	struct iomap srcmap;
	void *private;
};


struct iomap_ops {
	/*
	 * Return the existing mapping at pos, or reserve space starting at
	 * pos for up to length, as long as we can do it as a single mapping.
	 * The actual length is returned in iomap->length.
	 */
	int (*iomap_begin)(struct inode *inode, loff_t pos, loff_t length,
			unsigned flags, struct iomap *iomap,
			struct iomap *srcmap);

	/*
	 * Commit and/or unreserve space previous allocated using iomap_begin.
	 * Written indicates the length of the successful write operation which
	 * needs to be commited, while the rest needs to be unreserved.
	 * Written might be zero if no data was written.
	 */
	int (*iomap_end)(struct inode *inode, loff_t pos, loff_t length,
			ssize_t written, unsigned flags, struct iomap *iomap);
};

//direct-io
struct iomap_dio {
	struct kiocb		*iocb;
	const struct iomap_dio_ops *dops;
	loff_t			i_size;
	loff_t			size;
	atomic_t		ref;
	unsigned		flags;
	int			error;
	size_t			done_before;
	bool			wait_for_completion;

	union {
		/* used during submission and for synchronous completion: */
		struct {
			struct iov_iter		*iter;
			struct task_struct	*waiter;
		} submit;

		/* used for aio completion: */
		struct {
			struct work_struct	work;
		} aio;
	};
};		

```
# 代码分析

```c

static ssize_t ext4_dio_write_iter(struct kiocb *iocb, struct iov_iter *from)
{
	ssize_t ret;
	handle_t *handle;
	struct inode *inode = file_inode(iocb->ki_filp);
	loff_t offset = iocb->ki_pos;
	size_t count = iov_iter_count(from);
	const struct iomap_ops *iomap_ops = &ext4_iomap_ops;
	bool extend = false, unwritten = false;
	bool ilock_shared = true;
	int dio_flags = 0;

	/*
	 * Quick check here without any i_rwsem lock to see if it is extending
	 * IO. A more reliable check is done in ext4_dio_write_checks() with
	 * proper locking in place.初步判断是否为扩展IO（向文件末尾追加写入）
	 */
	if (offset + count > i_size_read(inode))
		ilock_shared = false;

	if (iocb->ki_flags & IOCB_NOWAIT) {
		if (ilock_shared) {
			if (!inode_trylock_shared(inode))
				return -EAGAIN;
		} else {
			if (!inode_trylock(inode))//若为扩展IO，可能可能触发元数据更新和块分配，
				return -EAGAIN;
		}
	} else {
		if (ilock_shared)
			inode_lock_shared(inode);
		else
			inode_lock(inode);
	}

	/* Fallback to buffered I/O if the inode does not support direct I/O. */
	if (!ext4_should_use_dio(iocb, from)) {
		if (ilock_shared)
			inode_unlock_shared(inode);
		else
			inode_unlock(inode);
		return ext4_buffered_write_iter(iocb, from);//不支持direct IO，使用缓冲IO
	}

	/*
	 * Prevent inline data from being created since we are going to allocate
	 * blocks for DIO. We know the inode does not currently have inline data
	 * because ext4_should_use_dio() checked for it, but we have to clear
	 * the state flag before the write checks because a lock cycle could
	 * introduce races with other writers.
	 防止创建内联数据，因为我们将为DIO分配块。
	 我们知道该inode当前没有内联数据，因为ext4_should_use_dio()已检查过这一点，
	 但必须在写检查之前清除状态标志，因为锁循环可能引入与其他写入者的竞态条件。
	 */
	ext4_clear_inode_state(inode, EXT4_STATE_MAY_INLINE_DATA);

	ret = ext4_dio_write_checks(iocb, from, &ilock_shared, &extend,
				    &unwritten, &dio_flags);//确定上排他锁或者非排他锁
	if (ret <= 0)
		return ret;

	offset = iocb->ki_pos;
	count = ret;

	if (extend) {
		handle = ext4_journal_start(inode, EXT4_HT_INODE, 2);
		if (IS_ERR(handle)) {
			ret = PTR_ERR(handle);
			goto out;
		}

		ret = ext4_orphan_add(handle, inode);
		ext4_journal_stop(handle);
		if (ret)
			goto out;
	}

	if (ilock_shared && !unwritten)
		iomap_ops = &ext4_iomap_overwrite_ops;
	ret = iomap_dio_rw(iocb, from, iomap_ops, &ext4_dio_write_ops,
			   dio_flags, NULL, 0);
	if (ret == -ENOTBLK)
		ret = 0;
	if (extend) {
		/*
		 * We always perform extending DIO write synchronously so by
		 * now the IO is completed and ext4_handle_inode_extension()
		 * was called. Cleanup the inode in case of error or race with
		 * writeback of delalloc blocks.
		 */
		WARN_ON_ONCE(ret == -EIOCBQUEUED);
		ext4_inode_extension_cleanup(inode, ret < 0);
	}

out:
	if (ilock_shared)
		inode_unlock_shared(inode);
	else
		inode_unlock(inode);

	if (ret >= 0 && iov_iter_count(from)) {//若还设有数据未写入，则通过缓冲IO写入
		ssize_t err;
		loff_t endbyte;

		/*
		 * There is no support for atomic writes on buffered-io yet,
		 * we should never fallback to buffered-io for DIO atomic
		 * writes.
		 */
		WARN_ON_ONCE(iocb->ki_flags & IOCB_ATOMIC);

		offset = iocb->ki_pos;
		err = ext4_buffered_write_iter(iocb, from);//写入到缓存
		if (err < 0)
			return err;

		/*
		 * We need to ensure that the pages within the page cache for
		 * the range covered by this I/O are written to disk and
		 * invalidated. This is in attempt to preserve the expected
		 * direct I/O semantics in the case we fallback to buffered I/O
		 * to complete off the I/O request.
		 * 我们需要确保在页面缓存中，被此I/O操作覆盖的范围内的页面被写入磁盘并失效。
		 * 这是为了在回退到缓冲I/O来完成I/O请求的情况下，保持预期的直接I/O语义
		 */
		ret += err;
		endbyte = offset + err - 1;
		err = filemap_write_and_wait_range(iocb->ki_filp->f_mapping,
						   offset, endbyte);//将页面回写到磁盘并等待回写结束
		if (!err)
			invalidate_mapping_pages(iocb->ki_filp->f_mapping,
						 offset >> PAGE_SHIFT,
						 endbyte >> PAGE_SHIFT);//清除相关缓存
	}

	return ret;
}
```