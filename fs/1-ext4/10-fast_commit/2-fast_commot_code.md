
# 追踪层



# 提交层

```c
/*
 * akpm: A new design for ext4_sync_file().
 *
 * This is only called from sys_fsync(), sys_fdatasync() and sys_msync().
 * There cannot be a transaction open by this task.
 * Another task could have dirtied this inode.  Its data can be in any
 * state in the journalling system.
 *
 * What we do is just kick off a commit and wait on it.  This will snapshot the
 * inode to disk.
 */
int ext4_sync_file(struct file *file, loff_t start, loff_t end, int datasync)
{
	int ret = 0, err;
	bool needs_barrier = false;
	struct inode *inode = file->f_mapping->host;

	ret = ext4_emergency_state(inode->i_sb);
	if (unlikely(ret))
		return ret;

	ASSERT(ext4_journal_current_handle() == NULL);

	trace_ext4_sync_file_enter(file, datasync);

	if (sb_rdonly(inode->i_sb))
		goto out;

	if (!EXT4_SB(inode->i_sb)->s_journal) {
		ret = ext4_fsync_nojournal(file, start, end, datasync,
					   &needs_barrier);
		if (needs_barrier)
			goto issue_flush;
		goto out;
	}

	ret = file_write_and_wait_range(file, start, end);
	if (ret)
		goto out;

	/*
	 *  The caller's filemap_fdatawrite()/wait will sync the data.
	 *  Metadata is in the journal, we wait for proper transaction to
	 *  commit here.
     调用者的  filemap_fdatawrite()  及后续的等待操作将负责同步数据。而元数据则存放在日志中，
     我们在此处等待元数据所属的事务提交
	 */
	ret = ext4_fsync_journal(inode, datasync, &needs_barrier);

issue_flush:
	if (needs_barrier) {
		err = blkdev_issue_flush(inode->i_sb->s_bdev);
		if (!ret)
			ret = err;
	}
out:
	err = file_check_and_advance_wb_err(file);
	if (ret == 0)
		ret = err;
	trace_ext4_sync_file_exit(inode, ret);
	return ret;
}


static int ext4_fsync_journal(struct inode *inode, bool datasync,
			     bool *needs_barrier)
{
	struct ext4_inode_info *ei = EXT4_I(inode);
	journal_t *journal = EXT4_SB(inode->i_sb)->s_journal;
	tid_t commit_tid = datasync ? ei->i_datasync_tid : ei->i_sync_tid;

	/*
	 * Fastcommit does not really support fsync on directories or other
	 * special files. Force a full commit.
	 */
	if (!S_ISREG(inode->i_mode))
		return ext4_force_commit(inode->i_sb);//如果是目录，强制full commit

	if (journal->j_flags & JBD2_BARRIER &&
	    !jbd2_trans_will_send_data_barrier(journal, commit_tid))
		*needs_barrier = true;//见## 第四部分　jbd2_trans_will_send_data_barrier 解析

	return ext4_fc_commit(journal, commit_tid);
}

/*
 * The main commit entry point. Performs a fast commit for transaction
 * commit_tid if needed. If it's not possible to perform a fast commit
 * due to various reasons, we fall back to full commit. Returns 0
 * on success, error otherwise.
 */
int ext4_fc_commit(journal_t *journal, tid_t commit_tid)
{
	struct super_block *sb = journal->j_private;
	struct ext4_sb_info *sbi = EXT4_SB(sb);
	int nblks = 0, ret, bsize = journal->j_blocksize;
	int subtid = atomic_read(&sbi->s_fc_subtid);
	int status = EXT4_FC_STATUS_OK, fc_bufs_before = 0;
	ktime_t start_time, commit_time;
	int old_ioprio, journal_ioprio;

	if (!test_opt2(sb, JOURNAL_FAST_COMMIT))
		return jbd2_complete_transaction(journal, commit_tid);

	trace_ext4_fc_commit_start(sb, commit_tid);

	start_time = ktime_get();
	old_ioprio = get_current_ioprio();

restart_fc:
	//检测是否需要提交，如需要则设置JBD2_FAST_COMMIT_ONGOING标记
	ret = jbd2_fc_begin_commit(journal, commit_tid);
	if (ret == -EALREADY) {
		/* There was an ongoing commit, check if we need to restart */
		/*
		jbd2_fc_begin_commit中情形 B 唤醒时，那个正在进行的提交已经结束（屏障已释放 + wake_up）。
		但"它结束了"不等于"它覆盖了你的 commit_tid。

		* subtid（本地变量）是进入本函数时对 s_fc_subtid 的快照；
		s_fc_subtid 这个计数器只在一次成功的 fast commit 后自增
		（line 1269 atomic_inc(&sbi->s_fc_subtid)），full commit 不会动它。

		* 所以 s_fc_subtid <= subtid 的实际含义是：
		从我进入函数到现在，没有其他进程"成功的 fast commit"提交过。
		换言之，刚才在jbd2_fc_begin_commit中情形 B挡住我、又已经结束的那个提交，
		是一个 full commit（它不会改变 subtid）

		* 反过来，如果 s_fc_subtid > subtid，说明刚才结束的是一个 fast commit，
		而且它把 subtid 推进了——意味着已经有另一个线程替我做了 fast commit

		* tid_gt(a,b) 即 a > b。这条成立表示 commit_tid 仍然严格
		新于最后一次已提交序列号，即还没有被 full commit 持久化，
		若不成立（commit_tid <= j_commit_sequence），
		说明我们的 tid 已经被 full commit 落盘了，fsync 的持久化保证已经满足
ext4_journal_start
	->jbd2__journal_start
		->start_this_handle
			->jbd2_get_transaction
				->transaction->t_tid = journal->j_transaction_sequence++;
				->journal->j_running_transaction = transaction;
jbd2_journal_commit_transaction
	->commit_transaction = journal->j_running_transaction;
	->WRITE_ONCE(journal->j_commit_sequence, commit_transaction->t_tid);
		*/
		if (atomic_read(&sbi->s_fc_subtid) <= subtid &&
		    tid_gt(commit_tid, journal->j_commit_sequence))
			goto restart_fc;
		ext4_fc_update_stats(sb, EXT4_FC_STATUS_SKIPPED, 0, 0,
				commit_tid);
		return 0;
	} else if (ret) {
		/*
		 * Commit couldn't start. Just update stats and perform a
		 * full commit.
		 */
		ext4_fc_update_stats(sb, EXT4_FC_STATUS_FAILED, 0, 0,
				commit_tid);
		return jbd2_complete_transaction(journal, commit_tid);
	}

	/*
	 * After establishing journal barrier via jbd2_fc_begin_commit(), check
	 * if we are fast commit ineligible.
	 */
	if (ext4_test_mount_flag(sb, EXT4_MF_FC_INELIGIBLE)) {
		status = EXT4_FC_STATUS_INELIGIBLE;
		goto fallback;
	}

	/*
	 * Now that we know that this thread is going to do a fast commit,
	 * elevate the priority to match that of the journal thread.
	 */
	if (journal->j_task->io_context)
		journal_ioprio = sbi->s_journal->j_task->io_context->ioprio;
	else
		journal_ioprio = EXT4_DEF_JOURNAL_IOPRIO;
	set_task_ioprio(current, journal_ioprio);
	fc_bufs_before = (sbi->s_fc_bytes + bsize - 1) / bsize;
	ret = ext4_fc_perform_commit(journal);//正式开始fast commit
	if (ret < 0) {
		status = EXT4_FC_STATUS_FAILED;
		goto fallback;
	}
	nblks = (sbi->s_fc_bytes + bsize - 1) / bsize - fc_bufs_before;
	ret = jbd2_fc_wait_bufs(journal, nblks);
	if (ret < 0) {
		status = EXT4_FC_STATUS_FAILED;
		goto fallback;
	}
	atomic_inc(&sbi->s_fc_subtid);
	ret = jbd2_fc_end_commit(journal);//先调用ext4_fc_cleanup，在唤醒在journal->j_fc_wait等待的进程
	set_task_ioprio(current, old_ioprio);
	/*
	 * weight the commit time higher than the average time so we
	 * don't react too strongly to vast changes in the commit time
	 */
	commit_time = ktime_to_ns(ktime_sub(ktime_get(), start_time));
	ext4_fc_update_stats(sb, status, commit_time, nblks, commit_tid);
	return ret;

fallback:
	set_task_ioprio(current, old_ioprio);
	ret = jbd2_fc_end_commit_fallback(journal);//回退到full commit，先清理fast commit唤醒后台commit进程，并等待commit完成
	ext4_fc_update_stats(sb, status, 0, 0, commit_tid);
	return ret;
}


/*
 * Start a fast commit. If there's an ongoing fast or full commit wait for
 * it to complete. Returns 0 if a new fast commit was started. Returns -EALREADY
 * if a fast commit is not needed, either because there's an already a commit
 * going on or this tid has already been committed. Returns -EINVAL if no jbd2
 * commit has yet been performed.
 */
int jbd2_fc_begin_commit(journal_t *journal, tid_t tid)
{
	if (unlikely(is_journal_aborted(journal)))
		return -EIO;
	/*
	 * Fast commits only allowed if at least one full commit has
	 * been processed.
	 */
	if (!journal->j_stats.ts_tid)
		return -EINVAL;

	/*情形 A: 你要 fast commit 的那个 commit_tid 已经被提交过了（被之前的 full commit 或 fast commit 覆盖）*/
	write_lock(&journal->j_state_lock);
	if (tid_geq(journal->j_commit_sequence, tid)) {
		write_unlock(&journal->j_state_lock);
		return -EALREADY;
	}
	/*情形 B：JBD2_FULL_COMMIT_ONGOING 或 JBD2_FAST_COMMIT_ONGOING 置位 —— 此刻有另一个进程正在提交（full 或 fast）。调用方被挂到 j_fc_wait 等待队列睡过去，唤醒后返回 -EALREADY*/
	if (journal->j_flags & JBD2_FULL_COMMIT_ONGOING ||
	    (journal->j_flags & JBD2_FAST_COMMIT_ONGOING)) {
		DEFINE_WAIT(wait);

		prepare_to_wait(&journal->j_fc_wait, &wait,
				TASK_UNINTERRUPTIBLE);
		write_unlock(&journal->j_state_lock);
		schedule();
		finish_wait(&journal->j_fc_wait, &wait);
		return -EALREADY;
	}
	journal->j_flags |= JBD2_FAST_COMMIT_ONGOING;
	write_unlock(&journal->j_state_lock);

	return 0;
}




static int ext4_fc_perform_commit(journal_t *journal)
{
	struct super_block *sb = journal->j_private;
	struct ext4_sb_info *sbi = EXT4_SB(sb);
	struct ext4_inode_info *iter;
	struct ext4_fc_head head;
	struct inode *inode;
	struct blk_plug plug;
	int ret = 0;
	u32 crc = 0;
	int alloc_ctx;

	/*
	 * Step 1: Mark all inodes on s_fc_q[MAIN] with
	 * EXT4_STATE_FC_FLUSHING_DATA. This prevents these inodes from being
	 * freed until the data flush is over.
	 标记每个inode状态为EXT4_STATE_FC_FLUSHING_DATA，防止被释放
	 */
	alloc_ctx = ext4_fc_lock(sb);
	list_for_each_entry(iter, &sbi->s_fc_q[FC_Q_MAIN], i_fc_list) {
		ext4_set_inode_state(&iter->vfs_inode,
				     EXT4_STATE_FC_FLUSHING_DATA);
	}
	ext4_fc_unlock(sb, alloc_ctx);

	/* Step 2: Flush data for all the eligible inodes. 
	 * 将所有inode里的脏数据罗盘（是数据，不是元数据）
	*/
	ret = ext4_fc_flush_data(journal);

	/*
	 * Step 3: Clear EXT4_STATE_FC_FLUSHING_DATA flag, before returning
	 * any error from step 2. This ensures that waiters waiting on
	 * EXT4_STATE_FC_FLUSHING_DATA can resume.
	 * 回写完成后，清楚EXT4_STATE_FC_FLUSHING_DATA标记，并唤醒睡眠的进程
	 */
	alloc_ctx = ext4_fc_lock(sb);
	list_for_each_entry(iter, &sbi->s_fc_q[FC_Q_MAIN], i_fc_list) {
		ext4_clear_inode_state(&iter->vfs_inode,
				       EXT4_STATE_FC_FLUSHING_DATA);
#if (BITS_PER_LONG < 64)
		wake_up_bit(&iter->i_state_flags, EXT4_STATE_FC_FLUSHING_DATA);
#else
		wake_up_bit(&iter->i_flags, EXT4_STATE_FC_FLUSHING_DATA);
#endif
	}

	/*
	 * Make sure clearing of EXT4_STATE_FC_FLUSHING_DATA is visible before
	 * the waiter checks the bit. Pairs with implicit barrier in
	 * prepare_to_wait() in ext4_fc_del().
	 */
	smp_mb();
	ext4_fc_unlock(sb, alloc_ctx);

	/*
	 * If we encountered error in Step 2, return it now after clearing
	 * EXT4_STATE_FC_FLUSHING_DATA bit.
	 */
	if (ret)
		return ret;


	/* Step 4: Mark all inodes as being committed. 
	 * 阻止任何新的更新 handle 启动，并阻塞到所有已存在的更新 handle 完成，
	 * 最终返回时 journal 处于"静止（quiescent）"状态——没有任何 handle 在跑，
	 * 也没有新 handle 能进来（见4-jbd2_journal_lock_updates_analysis.md文档）
	*/
	jbd2_journal_lock_updates(journal);
	/*
	 * The journal is now locked. No more handles can start and all the
	 * previous handles are now drained. We now mark the inodes on the
	 * commit queue as being committed.
	 */
	alloc_ctx = ext4_fc_lock(sb);
	list_for_each_entry(iter, &sbi->s_fc_q[FC_Q_MAIN], i_fc_list) {
		ext4_set_inode_state(&iter->vfs_inode,
				     EXT4_STATE_FC_COMMITTING);
	}
	ext4_fc_unlock(sb, alloc_ctx);
	jbd2_journal_unlock_updates(journal);

	/*
	 * Step 5: If file system device is different from journal device,
	 * issue a cache flush before we start writing fast commit blocks.
	  * 如果文件系统区和日志区不在一个dev上，先刷新文件系统区cache
	 */
	if (journal->j_fs_dev != journal->j_dev)
		blkdev_issue_flush(journal->j_fs_dev);

	blk_start_plug(&plug);
	alloc_ctx = ext4_fc_lock(sb);
	/* Step 6: Write fast commit blocks to disk. */
	if (sbi->s_fc_bytes == 0) {//这个tid的读一个tag，构造一个head tag
		/*
		 * Step 6.1: Add a head tag only if this is the first fast
		 * commit in this TID.
		 */
		head.fc_features = cpu_to_le32(EXT4_FC_SUPPORTED_FEATURES);
		head.fc_tid = cpu_to_le32(
			sbi->s_journal->j_running_transaction->t_tid);
		if (!ext4_fc_add_tlv(sb, EXT4_FC_TAG_HEAD, sizeof(head),
			(u8 *)&head, &crc)) {
			ret = -ENOSPC;
			goto out;
		}
	}

	/* Step 6.2: Now write all the dentry updates. 
	遍历每个目录inode（&sbi->s_fc_dentry_q），申请fc空间，构建tag
	*/
	ret = ext4_fc_commit_dentry_updates(journal, &crc);
	if (ret)
		goto out;

	/* Step 6.3: Now write all the changed inodes to disk. 
	 * 顺序与提交目录更新相反，先写inode数据，再写文件元数据
	*/
	list_for_each_entry(iter, &sbi->s_fc_q[FC_Q_MAIN], i_fc_list) {
		inode = &iter->vfs_inode;
		if (!ext4_test_inode_state(inode, EXT4_STATE_FC_COMMITTING))
			continue;

		ret = ext4_fc_write_inode_data(inode, &crc);
		if (ret)
			goto out;
		ret = ext4_fc_write_inode(inode, &crc);
		if (ret)
			goto out;
	}
	/* Step 6.4: Finally write tail tag to conclude this fast commit. 
	 * 写结尾tag(EXT4_FC_TAG_TAIL)，注意与块的结尾(EXT4_FC_TAG_PAD)不同
	 */
	ret = ext4_fc_write_tail(sb, crc);

out:
	ext4_fc_unlock(sb, alloc_ctx);
	blk_finish_plug(&plug);
	return ret;
}

/*
 * Adds tag, length, value and updates CRC. Returns true if tlv was added.
 * Returns false if there's not enough space.
 */
static bool ext4_fc_add_tlv(struct super_block *sb, u16 tag, u16 len, u8 *val,
			   u32 *crc)
{
	struct ext4_fc_tl tl;
	u8 *dst;

	dst = ext4_fc_reserve_space(sb, EXT4_FC_TAG_BASE_LEN + len, crc);
	if (!dst)
		return false;

	tl.fc_tag = cpu_to_le16(tag);
	tl.fc_len = cpu_to_le16(len);

	memcpy(dst, &tl, EXT4_FC_TAG_BASE_LEN);
	memcpy(dst + EXT4_FC_TAG_BASE_LEN, val, len);

	return true;
}


/* Ext4 commit path routines */

/*
 * Allocate len bytes on a fast commit buffer.
 *
 * During the commit time this function is used to manage fast commit
 * block space. We don't split a fast commit log onto different
 * blocks. So this function makes sure that if there's not enough space
 * on the current block, the remaining space in the current block is
 * marked as unused by adding EXT4_FC_TAG_PAD tag. In that case,
 * new block is from jbd2 and CRC is updated to reflect the padding
 * we added.
 */
static u8 *ext4_fc_reserve_space(struct super_block *sb, int len, u32 *crc)
{
	struct ext4_fc_tl tl;
	struct ext4_sb_info *sbi = EXT4_SB(sb);
	struct buffer_head *bh;
	int bsize = sbi->s_journal->j_blocksize;
	int ret, off = sbi->s_fc_bytes % bsize;
	int remaining;
	u8 *dst;

	/*
	 * If 'len' is too long to fit in any block alongside a PAD tlv, then we
	 * cannot fulfill the request.
	 * len 太长，一个block放不下，（尾部还有方一个TAG，再预留一个EXT4_FC_TAG_BASE_LEN）
	 */
	if (len > bsize - EXT4_FC_TAG_BASE_LEN)
		return NULL;

	//获取当写日志块的buffer head
	if (!sbi->s_fc_bh) {
		ret = jbd2_fc_get_buf(EXT4_SB(sb)->s_journal, &bh);
		if (ret)
			return NULL;
		sbi->s_fc_bh = bh;
	}
	dst = sbi->s_fc_bh->b_data + off;//目标地址

	/*
	 * Allocate the bytes in the current block if we can do so while still
	 * leaving enough space for a PAD tlv.
	 */
	remaining = bsize - EXT4_FC_TAG_BASE_LEN - off;
	if (len <= remaining) {//剩余的空间够放len，直接返回dst
		sbi->s_fc_bytes += len;
		return dst;
	}

	/*
	 * Else, terminate the current block with a PAD tlv, then allocate a new
	 * block and allocate the bytes at the start of that new block.剩余的空间不够放len，先放入PAD TAG，计算校验值，后边数据填0，最后计算校验值
	 */

	tl.fc_tag = cpu_to_le16(EXT4_FC_TAG_PAD);
	tl.fc_len = cpu_to_le16(remaining);
	memcpy(dst, &tl, EXT4_FC_TAG_BASE_LEN);
	memset(dst + EXT4_FC_TAG_BASE_LEN, 0, remaining);
	*crc = ext4_chksum(*crc, sbi->s_fc_bh->b_data, bsize);

	ext4_fc_submit_bh(sb, false);

	ret = jbd2_fc_get_buf(EXT4_SB(sb)->s_journal, &bh);//获取下一个新快
	if (ret)
		return NULL;
	sbi->s_fc_bh = bh;
	sbi->s_fc_bytes += bsize - off + len;//统计fc使用的总字节数，将上一个block的剩余用不上空间（bsize - off）加上本次需要的空间
	return sbi->s_fc_bh->b_data;
}

/* Commit all the directory entry updates */
static int ext4_fc_commit_dentry_updates(journal_t *journal, u32 *crc)
{
	struct super_block *sb = journal->j_private;
	struct ext4_sb_info *sbi = EXT4_SB(sb);
	struct ext4_fc_dentry_update *fc_dentry, *fc_dentry_n;
	struct inode *inode;
	struct ext4_inode_info *ei;
	int ret;

	if (list_empty(&sbi->s_fc_dentry_q[FC_Q_MAIN]))
		return 0;
	list_for_each_entry_safe(fc_dentry, fc_dentry_n,
				 &sbi->s_fc_dentry_q[FC_Q_MAIN], fcd_list) {
		if (fc_dentry->fcd_op != EXT4_FC_TAG_CREAT) {//非新建文件的tag（即 LINK / UNLINK）
			if (!ext4_fc_add_dentry_tlv(sb, crc, fc_dentry))
				return -ENOSPC;
			continue;
			/* 直接 ext4_fc_add_dentry_tlv 写出目录项 tag 然后 continue。
			因为这些 inode 早已存在且已在 s_fc_q[MAIN]，
			它们的 inode 映像 / extent 会由后面的 
			Step 6.3 负责写，这里只需补一个"目录项操作"tag。*/
		}
		/*
		 * With fcd_dilist we need not loop in sbi->s_fc_q to get the
		 * corresponding inode. Also, the corresponding inode could have been
		 * deleted, in which case, we don't need to do anything.
		 */
		if (list_empty(&fc_dentry->fcd_dilist))
			continue;// inode 已被 unlink，跳过
		ei = list_first_entry(&fc_dentry->fcd_dilist,
				struct ext4_inode_info, i_fc_dilist);
		inode = &ei->vfs_inode;
		WARN_ON(inode->i_ino != fc_dentry->fcd_ino);

		/*
		 * We first write the inode and then the create dirent. This
		 * allows the recovery code to create an unnamed inode first
		 * and then link it to a directory entry. This allows us
		 * to use namei.c routines almost as is and simplifies
		 * the recovery code.
		 * 在构造EXT4_FC_TAG_CREAT之前，先构造一个EXT4_FC_TAG_INODE和一个
		 */
		ret = ext4_fc_write_inode(inode, crc);
		if (ret)
			return ret;
		/*order语义决定数据先落盘， 通过查询当前状态，就可以告诉恢复层，接下来执行什么操作
		ext4_fc_flush_data 保证这些 tag 指向的数据先于 tag 落盘，在提交时刻，写入端（ext4_fc_write_inode_data）查询当前 extent 树（base 之上的终态），把"当前态相对 base 的差异"编码成 ADD_RANGE/DEL_RANGE tag 写入 journal；崩溃后，恢复端读取 base + 已编码的 tag 并执行*/
		ret = ext4_fc_write_inode_data(inode, crc);
		if (ret)
			return ret;
		if (!ext4_fc_add_dentry_tlv(sb, crc, fc_dentry))
			return -ENOSPC;
	}
	return 0;
}

```
**关键设计：CREAT 为什么"顺手"写 inode 映像+数据，而不等 Step 6.3?**
* Step 6.3（:1147）遍历的是 s_fc_q[FC_Q_MAIN]（inode 队列）来写每个 inode 的 data + inode。但一个新建文件的 inode 不一定在这个主 inode 队列上——它可能是本次事务里刚 ext4_create 出来的，还没走 ext4_fc_track_inode 入队；而它的 dentry CREAT 更新通过 fcd_dilist 反向关联到了它。

* 如果这个新建 inode 不在主队列，Step 6.3 就不会写它的映像，recovery 重放 CREAT 时就缺 inode 映像 / extent → 无法重建文件。**所以 CREAT 分支专门负责把这个"可能不在主队列里"的新建 inode 的 INODE + ADD_RANGE 写进 fc 区域**，fcd_dilist 让这个反查是 O(1) 而非扫描整个 s_fc_q。

* fcd_dilist 为空则跳过（:1016）：说明这个刚创建的 inode 在提交前已经被 unlink/evict（ext4_fc_del 摘掉了 fcd_dilist），此时整条 CREAT 记录作废——其 unlink 会作为另一条 dentry 记录另行处理。

**写顺序的讲究（与 Step 6.3 相反）**
* CREAT 分支：inode 映像 → 数据范围 → CREAT dentry（先建"无名 inode"再建链）
* Step 6.3：ext4_fc_write_inode_data → ext4_fc_write_inode（先数据后 inode）
* 注释（:1023-1029）点明原因：recovery 重放 CREAT 时，先"创建一个无名 inode"（INODE tag），再把它"链接"到目录项（CREAT tag），这样能几乎原样复用 namei.c 的建链例程，简化恢复代码。两者语义不同——6.3 是"已存在 inode 的增量追加 range"，CREAT 是"从零建 inode"，故顺序相反。


```c
/*
 * Fast commit cleanup routine. This is called after every fast commit and
 * full commit. full is true if we are called after a full commit.
 */
static void ext4_fc_cleanup(journal_t *journal, int full, tid_t tid)
{
	struct super_block *sb = journal->j_private;
	struct ext4_sb_info *sbi = EXT4_SB(sb);
	struct ext4_inode_info *ei;
	struct ext4_fc_dentry_update *fc_dentry;
	int alloc_ctx;

	if (full && sbi->s_fc_bh)
		sbi->s_fc_bh = NULL;

	trace_ext4_fc_cleanup(journal, full, tid);
	/*fast commit提交成功的块在jbd2_fc_wait_bufs释放掉了，
	中途失败的回退到full commit的块在这里释放*/
	jbd2_fc_release_bufs(journal);

	alloc_ctx = ext4_fc_lock(sb);
	while (!list_empty(&sbi->s_fc_q[FC_Q_MAIN])) {
		ei = list_first_entry(&sbi->s_fc_q[FC_Q_MAIN],
					struct ext4_inode_info,
					i_fc_list);
		list_del_init(&ei->i_fc_list);
		ext4_clear_inode_state(&ei->vfs_inode,
				       EXT4_STATE_FC_COMMITTING);
		/* 判断"当前结束的事务 tid 是否已经覆盖该 inode 最近一次被 track 时的 tid
		 * fast commit 路径故意传 tid=0——因为 fast commit 复用运行中的 jbd2 事务
		*/
		if (tid_geq(tid, ei->i_sync_tid)) {// 
			ext4_fc_reset_inode(&ei->vfs_inode);
		} else if (full) {
			/*
			 * 没覆盖（说明 commit 跑的过程中该 inode 又有了更晚的改动 tid）
			 * 且本次是 full commit → 把它重入 STAGING 队列，稍后被 splice 回 MAIN，
			 留给下一轮 fc.
			 * We are called after a full commit, inode has been
			 * modified while the commit was running. Re-enqueue
			 * the inode into STAGING, which will then be splice
			 * back into MAIN. This cannot happen during
			 * fastcommit because the journal is locked all the
			 * time in that case (and tid doesn't increase so
			 * tid check above isn't reliable).
			 */
			list_add_tail(&ei->i_fc_list,
				      &sbi->s_fc_q[FC_Q_STAGING]);
		}
		/*
		 * Make sure clearing of EXT4_STATE_FC_COMMITTING is
		 * visible before we send the wakeup. Pairs with implicit
		 * barrier in prepare_to_wait() in ext4_fc_track_inode().
		 */
		smp_mb();
#if (BITS_PER_LONG < 64)
		wake_up_bit(&ei->i_state_flags, EXT4_STATE_FC_COMMITTING);
#else
		wake_up_bit(&ei->i_flags, EXT4_STATE_FC_COMMITTING);
#endif
	}
	/*释放s_fc_dentry_q队列里的目录entry的内存*/
	while (!list_empty(&sbi->s_fc_dentry_q[FC_Q_MAIN])) {
		fc_dentry = list_first_entry(&sbi->s_fc_dentry_q[FC_Q_MAIN],
					     struct ext4_fc_dentry_update,
					     fcd_list);
		list_del_init(&fc_dentry->fcd_list);
		list_del_init(&fc_dentry->fcd_dilist);

		release_dentry_name_snapshot(&fc_dentry->fcd_name);
		kmem_cache_free(ext4_fc_dentry_cachep, fc_dentry);
	}

	/* 将 STAGING 队列搬回 MAIN 队列 */
	list_splice_init(&sbi->s_fc_dentry_q[FC_Q_STAGING],
				&sbi->s_fc_dentry_q[FC_Q_MAIN]);
	list_splice_init(&sbi->s_fc_q[FC_Q_STAGING],
				&sbi->s_fc_q[FC_Q_MAIN]);

	/*曾因某操作（如 rename 目录、加密文件名等）
	把文件系统标记为"本轮不可 fast commit"，
	并记录触发时的 tid 到 s_fc_ineligible_tid,
	当一次 full commit 的 tid T 已经 ≥ 那个 tid 时，
	说明"惹祸"的事务已彻底提交，资格自动恢复*/
	if (tid_geq(tid, sbi->s_fc_ineligible_tid)) {
		sbi->s_fc_ineligible_tid = 0;
		ext4_clear_mount_flag(sb, EXT4_MF_FC_INELIGIBLE);
	}

	if (full)
		sbi->s_fc_bytes = 0;
	ext4_fc_unlock(sb, alloc_ctx);
	trace_ext4_fc_stats(sb);
}
```