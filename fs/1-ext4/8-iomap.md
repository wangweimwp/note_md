```c
ext4_file_write_iter
	->ext4_dio_write_iter	//用iomap写入数据，没有先入的用缓冲IO写入
		->ext4_dio_write_checks	//确定在陷入过程中使用排他锁还是非排他锁
		->iomap_dio_rw
			->__iomap_dio_rw
				->iomap_iter
					->ops->iomap_end	ext4_iomap_end	//空操作
					->iomap_iter_advance	//iomap_iter向前推进
					->ops->iomap_begin   /*ext4_iomap_begin  根据iomap_iter，
										先从ext4文件系统申请block并建立映射，然后初始化iomap结构体*/    
				->iomap_dio_iter
					->iomap_dio_bio_iter
						->iomap_dio_alloc_bio      //	循环
						->bio_iov_iter_get_pages	/*	循环，将当前iov_iter指向的页面pin住，
														page指针放在bio中，并向前推进iov_iter*/
						->iomap_dio_submit_bio
							->submit_bio//提交bio请求
			->iomap_dio_complete
				->dops->end_io	ext4_dio_write_end_io	//文件inode标记为脏


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

//iomap机制主要依赖iomap_iter建立磁盘块映射和向前迭代，然后通过iov_iter与BIO向映射好的块写数据

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


/*
 * iomap_dio_rw() always completes O_[D]SYNC writes regardless of whether the IO
 * is being issued as AIO or not.  This allows us to optimise pure data writes
 * to use REQ_FUA rather than requiring generic_write_sync() to issue a
 * REQ_FLUSH post write. This is slightly tricky because a single request here
 * can be mapped into multiple disjoint IOs and only a subset of the IOs issued
 * may be pure data writes. In that case, we still need to do a full data sync
 * completion.
 *
 * When page faults are disabled and @dio_flags includes IOMAP_DIO_PARTIAL,
 * __iomap_dio_rw can return a partial result if it encounters a non-resident
 * page in @iter after preparing a transfer.  In that case, the non-resident
 * pages can be faulted in and the request resumed with @done_before set to the
 * number of bytes previously transferred.  The request will then complete with
 * the correct total number of bytes transferred; this is essential for
 * completing partial requests asynchronously.
 *
 * Returns -ENOTBLK In case of a page invalidation invalidation failure for
 * writes.  The callers needs to fall back to buffered I/O in this case.
 */
struct iomap_dio *
__iomap_dio_rw(struct kiocb *iocb, struct iov_iter *iter,
		const struct iomap_ops *ops, const struct iomap_dio_ops *dops,
		unsigned int dio_flags, void *private, size_t done_before)
{
	struct inode *inode = file_inode(iocb->ki_filp);
	struct iomap_iter iomi = {
		.inode		= inode,
		.pos		= iocb->ki_pos,
		.len		= iov_iter_count(iter),
		.flags		= IOMAP_DIRECT,
		.private	= private,
	};
	bool wait_for_completion =
		is_sync_kiocb(iocb) || (dio_flags & IOMAP_DIO_FORCE_WAIT);
	struct blk_plug plug;
	struct iomap_dio *dio;
	loff_t ret = 0;

	trace_iomap_dio_rw_begin(iocb, iter, dio_flags, done_before);

	if (!iomi.len)
		return NULL;

	dio = kmalloc(sizeof(*dio), GFP_KERNEL);
	if (!dio)
		return ERR_PTR(-ENOMEM);

	dio->iocb = iocb;
	atomic_set(&dio->ref, 1);
	dio->size = 0;
	dio->i_size = i_size_read(inode);
	dio->dops = dops;
	dio->error = 0;
	dio->flags = 0;
	dio->done_before = done_before;

	dio->submit.iter = iter;
	dio->submit.waiter = current;

	if (iocb->ki_flags & IOCB_NOWAIT)
		iomi.flags |= IOMAP_NOWAIT;

	if (iocb->ki_flags & IOCB_ATOMIC)
		iomi.flags |= IOMAP_ATOMIC;

	if (iov_iter_rw(iter) == READ) {//读操作
		/* reads can always complete inline */
		dio->flags |= IOMAP_DIO_INLINE_COMP;

		if (iomi.pos >= dio->i_size)
			goto out_free_dio;

		if (user_backed_iter(iter))
			dio->flags |= IOMAP_DIO_DIRTY;

		ret = kiocb_write_and_wait(iocb, iomi.len);//触发数据回写磁盘，最终调用do_writepages
		if (ret)
			goto out_free_dio;
	} else {//写操作
		iomi.flags |= IOMAP_WRITE;
		dio->flags |= IOMAP_DIO_WRITE;

		/*
		 * Flag as supporting deferred completions, if the issuer
		 * groks it. This can avoid a workqueue punt for writes.
		 * We may later clear this flag if we need to do other IO
		 * as part of this IO completion.
		 */
		if (iocb->ki_flags & IOCB_DIO_CALLER_COMP)
			dio->flags |= IOMAP_DIO_CALLER_COMP;

		if (dio_flags & IOMAP_DIO_OVERWRITE_ONLY) {//仅仅包含覆盖写入？
			ret = -EAGAIN;
			if (iomi.pos >= dio->i_size ||
			    iomi.pos + iomi.len > dio->i_size)
				goto out_free_dio;
			iomi.flags |= IOMAP_OVERWRITE_ONLY;
		}

		/* for data sync or sync, we need sync completion processing */
		if (iocb_is_dsync(iocb)) {
			dio->flags |= IOMAP_DIO_NEED_SYNC;

		       /*
			* For datasync only writes, we optimistically try using
			* WRITE_THROUGH for this IO. This flag requires either
			* FUA writes through the device's write cache, or a
			* normal write to a device without a volatile write
			* cache. For the former, Any non-FUA write that occurs
			* will clear this flag, hence we know before completion
			* whether a cache flush is necessary.
			*/
			if (!(iocb->ki_flags & IOCB_SYNC))
				dio->flags |= IOMAP_DIO_WRITE_THROUGH;
		}

		/*
		 * Try to invalidate cache pages for the range we are writing.
		 * If this invalidation fails, let the caller fall back to
		 * buffered I/O.将写数据范围的page cache释放掉，若释放失败则回退到缓冲IO
		 */
		ret = kiocb_invalidate_pages(iocb, iomi.len);
		if (ret) {
			if (ret != -EAGAIN) {
				trace_iomap_dio_invalidate_fail(inode, iomi.pos,
								iomi.len);
				if (iocb->ki_flags & IOCB_ATOMIC) {
					/*
					 * folio invalidation failed, maybe
					 * this is transient, unlock and see if
					 * the caller tries again.
					 */
					ret = -EAGAIN;
				} else {
					/* fall back to buffered write */
					ret = -ENOTBLK;
				}
			}
			goto out_free_dio;
		}

		if (!wait_for_completion && !inode->i_sb->s_dio_done_wq) {
			ret = sb_init_dio_done_wq(inode->i_sb);//为延迟直接IO创建工作队列
			if (ret < 0)
				goto out_free_dio;
		}
	}

	inode_dio_begin(inode);

	blk_start_plug(&plug);
	while ((ret = iomap_iter(&iomi, ops)) > 0) {//根据iomap_iter，先从ext4文件系统申请block并建立映射，然后初始化iomap结构体
		iomi.processed = iomap_dio_iter(&iomi, dio);//循环，将当前iov_iter指向的页面pin住，page指针放在bio中，并向前推进iov_iter

		/*
		 * We can only poll for single bio I/Os.
		 */
		iocb->ki_flags &= ~IOCB_HIPRI;
	}

	blk_finish_plug(&plug);

	/*
	 * We only report that we've read data up to i_size.
	 * Revert iter to a state corresponding to that as some callers (such
	 * as the splice code) rely on it.
	 * 我们仅报告已读取至 i_size 位置的数据。
	 * 将迭代器回滚至对应状态，因为某些调用者（如 splice 代码）依赖于此
	 */
	if (iov_iter_rw(iter) == READ && iomi.pos >= dio->i_size)
		iov_iter_revert(iter, iomi.pos - dio->i_size);

	if (ret == -EFAULT && dio->size && (dio_flags & IOMAP_DIO_PARTIAL)) {
		if (!(iocb->ki_flags & IOCB_NOWAIT))
			wait_for_completion = true;
		ret = 0;
	}

	/* magic error code to fall back to buffered I/O */
	if (ret == -ENOTBLK) {
		wait_for_completion = true;
		ret = 0;
	}
	if (ret < 0)
		iomap_dio_set_error(dio, ret);

	/*
	 * If all the writes we issued were already written through to the
	 * media, we don't need to flush the cache on IO completion. Clear the
	 * sync flag for this case.
	 */
	if (dio->flags & IOMAP_DIO_WRITE_THROUGH)
		dio->flags &= ~IOMAP_DIO_NEED_SYNC;

	/*
	 * We are about to drop our additional submission reference, which
	 * might be the last reference to the dio.  There are three different
	 * ways we can progress here:
	 *
	 *  (a) If this is the last reference we will always complete and free
	 *	the dio ourselves.
	 *  (b) If this is not the last reference, and we serve an asynchronous
	 *	iocb, we must never touch the dio after the decrement, the
	 *	I/O completion handler will complete and free it.
	 *  (c) If this is not the last reference, but we serve a synchronous
	 *	iocb, the I/O completion handler will wake us up on the drop
	 *	of the final reference, and we will complete and free it here
	 *	after we got woken by the I/O completion handler.
	 */
	dio->wait_for_completion = wait_for_completion;
	if (!atomic_dec_and_test(&dio->ref)) {
		if (!wait_for_completion) {
			trace_iomap_dio_rw_queued(inode, iomi.pos, iomi.len);
			return ERR_PTR(-EIOCBQUEUED);
		}

		for (;;) {
			set_current_state(TASK_UNINTERRUPTIBLE);
			if (!READ_ONCE(dio->submit.waiter))
				break;

			blk_io_schedule();
		}
		__set_current_state(TASK_RUNNING);
	}

	return dio;

out_free_dio:
	kfree(dio);
	if (ret)
		return ERR_PTR(ret);
	return NULL;
}



static loff_t iomap_dio_bio_iter(const struct iomap_iter *iter,
		struct iomap_dio *dio)
{
	const struct iomap *iomap = &iter->iomap;
	struct inode *inode = iter->inode;
	unsigned int fs_block_size = i_blocksize(inode), pad;
	const loff_t length = iomap_length(iter);
	bool atomic = iter->flags & IOMAP_ATOMIC;
	loff_t pos = iter->pos;
	blk_opf_t bio_opf;
	struct bio *bio;
	bool need_zeroout = false;
	bool use_fua = false;
	int nr_pages, ret = 0;
	size_t copied = 0;
	size_t orig_count;

	if (atomic && length != fs_block_size)
		return -EINVAL;

	if ((pos | length) & (bdev_logical_block_size(iomap->bdev) - 1) ||
	    !bdev_iter_is_aligned(iomap->bdev, dio->submit.iter))
		return -EINVAL;

	if (iomap->type == IOMAP_UNWRITTEN) {
		dio->flags |= IOMAP_DIO_UNWRITTEN;
		need_zeroout = true;
	}

	if (iomap->flags & IOMAP_F_SHARED)
		dio->flags |= IOMAP_DIO_COW;/*文件系统写时复制，即不覆盖原有数据块，新分配块写入数据，
									 即保留原数据也保留新数据，多用于文件快照和共享文件*/

	if (iomap->flags & IOMAP_F_NEW) {
		need_zeroout = true;
	} else if (iomap->type == IOMAP_MAPPED) {
		/*
		 * Use a FUA write if we need datasync semantics, this is a pure
		 * data IO that doesn't require any metadata updates (including
		 * after IO completion such as unwritten extent conversion) and
		 * the underlying device either supports FUA or doesn't have
		 * a volatile write cache. This allows us to avoid cache flushes
		 * on IO completion. If we can't use writethrough and need to
		 * sync, disable in-task completions as dio completion will
		 * need to call generic_write_sync() which will do a blocking
		 * fsync / cache flush call.
		 */
		if (!(iomap->flags & (IOMAP_F_SHARED|IOMAP_F_DIRTY)) &&
		    (dio->flags & IOMAP_DIO_WRITE_THROUGH) &&
		    (bdev_fua(iomap->bdev) || !bdev_write_cache(iomap->bdev)))
			use_fua = true;
		else if (dio->flags & IOMAP_DIO_NEED_SYNC)
			dio->flags &= ~IOMAP_DIO_CALLER_COMP;
	}

	/*
	 * Save the original count and trim the iter to just the extent we
	 * are operating on right now.  The iter will be re-expanded once
	 * we are done.
	 * 保存原始计数，并将迭代器截取到当前正在操作的文件区段。
	 * 操作完成后迭代器将重新扩展。
	 */
	orig_count = iov_iter_count(dio->submit.iter);
	iov_iter_truncate(dio->submit.iter, length);

	if (!iov_iter_count(dio->submit.iter))
		goto out;

	/*
	 * We can only do deferred completion for pure overwrites that
	 * don't require additional IO at completion. This rules out
	 * writes that need zeroing or extent conversion, extend
	 * the file size, or issue journal IO or cache flushes
	 * during completion processing.
	 */
	if (need_zeroout ||
	    ((dio->flags & IOMAP_DIO_NEED_SYNC) && !use_fua) ||
	    ((dio->flags & IOMAP_DIO_WRITE) && pos >= i_size_read(inode)))
		dio->flags &= ~IOMAP_DIO_CALLER_COMP;

	/*
	 * The rules for polled IO completions follow the guidelines as the
	 * ones we set for inline and deferred completions. If none of those
	 * are available for this IO, clear the polled flag.
	 */
	if (!(dio->flags & (IOMAP_DIO_INLINE_COMP|IOMAP_DIO_CALLER_COMP)))
		dio->iocb->ki_flags &= ~IOCB_HIPRI;

	if (need_zeroout) {//先将写的位置清0
		/* zero out from the start of the block to the write offset */
		pad = pos & (fs_block_size - 1);

		ret = iomap_dio_zero(iter, dio, pos - pad, pad);
		if (ret)
			goto out;
	}

	bio_opf = iomap_dio_bio_opflags(dio, iomap, use_fua, atomic);

	/*遍历iov_iter上的成员列表，统计总页数*/
	nr_pages = bio_iov_vecs_to_alloc(dio->submit.iter, BIO_MAX_VECS);
	do {
		size_t n;
		if (dio->error) {
			iov_iter_revert(dio->submit.iter, copied);
			copied = ret = 0;
			goto out;
		}

		bio = iomap_dio_alloc_bio(iter, dio, nr_pages, bio_opf);//申请BIO
		fscrypt_set_bio_crypt_ctx(bio, inode, pos >> inode->i_blkbits,
					  GFP_KERNEL);
		bio->bi_iter.bi_sector = iomap_sector(iomap, pos);
		bio->bi_write_hint = inode->i_write_hint;
		bio->bi_ioprio = dio->iocb->ki_ioprio;
		bio->bi_private = dio;
		bio->bi_end_io = iomap_dio_bio_end_io;
		
		/*1,申请数组，用于存放page结构体指针
		 *2,根据addr，将对应的页pin住，page结构体指针放到刚申请的数组中
		 *3,iov_iter向前迭代
		 *4,梳理bio中每个bv的bv_page,len,offset等*/
		ret = bio_iov_iter_get_pages(bio, dio->submit.iter);
		if (unlikely(ret)) {
			/*
			 * We have to stop part way through an IO. We must fall
			 * through to the sub-block tail zeroing here, otherwise
			 * this short IO may expose stale data in the tail of
			 * the block we haven't written data to.
			 */
			bio_put(bio);
			goto zero_tail;
		}

		n = bio->bi_iter.bi_size;
		if (WARN_ON_ONCE(atomic && n != length)) {
			/*
			 * This bio should have covered the complete length,
			 * which it doesn't, so error. We may need to zero out
			 * the tail (complete FS block), similar to when
			 * bio_iov_iter_get_pages() returns an error, above.
			 */
			ret = -EINVAL;
			bio_put(bio);
			goto zero_tail;
		}
		if (dio->flags & IOMAP_DIO_WRITE) {
			task_io_account_write(n);
		} else {
			if (dio->flags & IOMAP_DIO_DIRTY)
				bio_set_pages_dirty(bio);
		}

		dio->size += n;
		copied += n;

		nr_pages = bio_iov_vecs_to_alloc(dio->submit.iter,
						 BIO_MAX_VECS);//iov_iter是否还有page，计算下次bio请求的页数
		/*
		 * We can only poll for single bio I/Os.
		 */
		if (nr_pages)
			dio->iocb->ki_flags &= ~IOCB_HIPRI;
		iomap_dio_submit_bio(iter, dio, bio, pos);//提奖当前bio发送请求
		pos += n;
	} while (nr_pages);//下次bio请求页数不为0，继续循环

	/*
	 * We need to zeroout the tail of a sub-block write if the extent type
	 * requires zeroing or the write extends beyond EOF. If we don't zero
	 * the block tail in the latter case, we can expose stale data via mmap
	 * reads of the EOF block.
	 */
zero_tail:
	if (need_zeroout ||
	    ((dio->flags & IOMAP_DIO_WRITE) && pos >= i_size_read(inode))) {
		/* zero out from the end of the write to the end of the block */
		pad = pos & (fs_block_size - 1);
		if (pad)
			ret = iomap_dio_zero(iter, dio, pos,
					     fs_block_size - pad);
	}
out:
	/* Undo iter limitation to current extent */
	iov_iter_reexpand(dio->submit.iter, orig_count - copied);
	if (copied)
		return copied;
	return ret;
}
```