# 基本函数

## ext4_journal_start

```c

ext4_journal_start
    ->__ext4_journal_start
        ->__ext4_journal_start_sb
            ->ext4_journal_check_start
            ->jbd2__journal_start   //journal_start 的主要作用是取得一个原子操作描述符 handle_t，如果当前进程已经有一个，则直接返回；否则，需要新创建一个
                ->start_this_handle //start_this_handle 的主要作用是将该 handle 与当前正在运行的 transaction 相关联，如果没有正在运行的 transaction，则创建一个新的，并将其设置为正在运行的 


handle_t *jbd2__journal_start(journal_t *journal, int nblocks, int rsv_blocks,
			      int revoke_records, gfp_t gfp_mask,
			      unsigned int type, unsigned int line_no)
{
	handle_t *handle = journal_current_handle();
	int err;

	if (!journal)
		return ERR_PTR(-EROFS);

	if (handle) {
		J_ASSERT(handle->h_transaction->t_journal == journal);
		handle->h_ref++;
		return handle;
	}
    /*nblocks 是向日志申请 nblocks 个缓冲区的空间，
    也表明该原子操作预期将修改nblocks 个缓冲区。*/
	nblocks += DIV_ROUND_UP(revoke_records,
				journal->j_revoke_records_per_block);
	handle = new_handle(nblocks);//申请handle结构体
	if (!handle)
		return ERR_PTR(-ENOMEM);
	if (rsv_blocks) {
		handle_t *rsv_handle;

		rsv_handle = new_handle(rsv_blocks);
		if (!rsv_handle) {
			jbd2_free_handle(handle);
			return ERR_PTR(-ENOMEM);
		}
		rsv_handle->h_reserved = 1;
		rsv_handle->h_journal = journal;
		handle->h_rsv_handle = rsv_handle;
	}
	handle->h_revoke_credits = revoke_records;

	err = start_this_handle(journal, handle, gfp_mask);//与transaction建立关联
	if (err < 0) {
		if (handle->h_rsv_handle)
			jbd2_free_handle(handle->h_rsv_handle);
		jbd2_free_handle(handle);
		return ERR_PTR(err);
	}
	handle->h_type = type;
	handle->h_line_no = line_no;
	trace_jbd2_handle_start(journal->j_fs_dev->bd_dev,
				handle->h_transaction->t_tid, type,
				line_no, nblocks);

	return handle;
}


/*
 * start_this_handle: Given a handle, deal with any locking or stalling
 * needed to make sure that there is enough journal space for the handle
 * to begin.  Attach the handle to a transaction and set up the
 * transaction's buffer credits.
 */

static int start_this_handle(journal_t *journal, handle_t *handle,
			     gfp_t gfp_mask)
{
	transaction_t	*transaction, *new_transaction = NULL;
	int		blocks = handle->h_total_credits;
	int		rsv_blocks = 0;
	unsigned long ts = jiffies;

	if (handle->h_rsv_handle)
		rsv_blocks = handle->h_rsv_handle->h_total_credits;

	/*
	 * Limit the number of reserved credits to 1/2 of maximum transaction
	 * size and limit the number of total credits to not exceed maximum
	 * transaction size per operation.
	 */
	if (rsv_blocks > jbd2_max_user_trans_buffers(journal) / 2 ||
	    rsv_blocks + blocks > jbd2_max_user_trans_buffers(journal)) {
		printk(KERN_ERR "JBD2: %s wants too many credits "
		       "credits:%d rsv_credits:%d max:%d\n",
		       current->comm, blocks, rsv_blocks,
		       jbd2_max_user_trans_buffers(journal));
		WARN_ON(1);
		return -ENOSPC;
	}

alloc_transaction:
	/*
	 * This check is racy but it is just an optimization of allocating new
	 * transaction early if there are high chances we'll need it. If we
	 * guess wrong, we'll retry or free unused transaction.
	 */
	if (!data_race(journal->j_running_transaction)) {
		/*
		 * If __GFP_FS is not present, then we may be being called from
		 * inside the fs writeback layer, so we MUST NOT fail.
		 */
		if ((gfp_mask & __GFP_FS) == 0)
			gfp_mask |= __GFP_NOFAIL;
		new_transaction = kmem_cache_zalloc(transaction_cache,
						    gfp_mask);
		if (!new_transaction)
			return -ENOMEM;
	}

	jbd2_debug(3, "New handle %p going live.\n", handle);

	/*
	 * We need to hold j_state_lock until t_updates has been incremented,
	 * for proper journal barrier handling
	 */
repeat:
	read_lock(&journal->j_state_lock);
	BUG_ON(journal->j_flags & JBD2_UNMOUNT);
	if (is_journal_aborted(journal) ||
	    (journal->j_errno != 0 && !(journal->j_flags & JBD2_ACK_ERR))) {
		read_unlock(&journal->j_state_lock);
		jbd2_journal_free_transaction(new_transaction);
		return -EROFS;
	}

	/*
	 * Wait on the journal's transaction barrier if necessary. Specifically
	 * we allow reserved handles to proceed because otherwise commit could
	 * deadlock on page writeback not being able to complete.
	 */
	if (!handle->h_reserved && journal->j_barrier_count) {
		read_unlock(&journal->j_state_lock);
		wait_event(journal->j_wait_transaction_locked,
				journal->j_barrier_count == 0);
		goto repeat;
	}

	if (!journal->j_running_transaction) {
		read_unlock(&journal->j_state_lock);
		if (!new_transaction)
			goto alloc_transaction;
		write_lock(&journal->j_state_lock);
		if (!journal->j_running_transaction &&
		    (handle->h_reserved || !journal->j_barrier_count)) {
			//初始化new_transaction结构体，同时journal->j_running_transaction = new_transaction
			jbd2_get_transaction(journal, new_transaction);
			new_transaction = NULL;
		}
		write_unlock(&journal->j_state_lock);
		goto repeat;
	}
/*以上代码处理如果没有正在运行的 transaction，则创建一个新的，并将其设置为正在运行的*/

	transaction = journal->j_running_transaction;

	if (!handle->h_reserved) {
		/* We may have dropped j_state_lock - restart in that case */
		/* 查询transaction上的credits是否足够，不够进入等待队列，并返回1。否则返回0*/
		if (add_transaction_credits(journal, blocks, rsv_blocks)) {
			/*
			 * add_transaction_credits releases
			 * j_state_lock on a non-zero return
			 */
			__release(&journal->j_state_lock);
			goto repeat;
		}
	} else {
		/*
		 * We have handle reserved so we are allowed to join T_LOCKED
		 * transaction and we don't have to check for transaction size
		 * and journal space. But we still have to wait while running
		 * transaction is being switched to a committing one as it
		 * won't wait for any handles anymore.
		 */
		if (transaction->t_state == T_SWITCH) {
			wait_transaction_switching(journal);
			goto repeat;
		}
		sub_reserved_credits(journal, blocks);
		handle->h_reserved = 0;
	}

	/* OK, account for the buffers that this operation expects to
	 * use and add the handle to the running transaction.
	 * 初始化handle的结构体长远，完成与running transaction绑定
	 */
	update_t_max_wait(transaction, ts);//最近一次handle加入transaction的时间
	handle->h_transaction = transaction;
	handle->h_requested_credits = blocks;
	handle->h_revoke_credits_requested = handle->h_revoke_credits;
	handle->h_start_jiffies = jiffies;//handle开始的时间
	atomic_inc(&transaction->t_updates);
	atomic_inc(&transaction->t_handle_count);
	jbd2_debug(4, "Handle %p given %d credits (total %d, free %lu)\n",
		  handle, blocks,
		  atomic_read(&transaction->t_outstanding_credits),
		  jbd2_log_space_left(journal));
	read_unlock(&journal->j_state_lock);
	current->journal_info = handle;//handle与当前进程绑定

	rwsem_acquire_read(&journal->j_trans_commit_map, 0, 0, _THIS_IP_);
	jbd2_journal_free_transaction(new_transaction);
	/*
	 * Ensure that no allocations done while the transaction is open are
	 * going to recurse back to the fs layer.
	 */
	 //设置PF_MEMALLOC_NOFS标记到当前进程，返回旧的标记后边恢复
	handle->saved_alloc_context = memalloc_nofs_save();
	return 0;
}
```

## jbd2_journal_stop
```c

/**
 * jbd2_journal_stop() - complete a transaction
 * @handle: transaction to complete.
 *
 * All done for a particular handle.
 *
 * There is not much action needed here.  We just return any remaining
 * buffer credits to the transaction and remove the handle.  The only
 * complication is that we need to start a commit operation if the
 * filesystem is marked for synchronous update.
 *
 * jbd2_journal_stop itself will not usually return an error, but it may
 * do so in unusual circumstances.  In particular, expect it to
 * return -EIO if a jbd2_journal_abort has been executed since the
 * transaction began.
 */
int jbd2_journal_stop(handle_t *handle)
{
	transaction_t *transaction = handle->h_transaction;
	journal_t *journal;
	int err = 0, wait_for_commit = 0;
	tid_t tid;
	pid_t pid;

	if (--handle->h_ref > 0) {
		jbd2_debug(4, "h_ref %d -> %d\n", handle->h_ref + 1,
						 handle->h_ref);
		if (is_handle_aborted(handle))
			return -EIO;
		return 0;
	}
	if (!transaction) {
		/*
		 * Handle is already detached from the transaction so there is
		 * nothing to do other than free the handle.
		 */
		memalloc_nofs_restore(handle->saved_alloc_context);
		goto free_and_exit;
	}
	journal = transaction->t_journal;
	tid = transaction->t_tid;//jbd2_get_transaction()函数中设置

	if (is_handle_aborted(handle))
		err = -EIO;

	jbd2_debug(4, "Handle %p going down\n", handle);
	trace_jbd2_handle_stats(journal->j_fs_dev->bd_dev,
				tid, handle->h_type, handle->h_line_no,
				jiffies - handle->h_start_jiffies,
				handle->h_sync, handle->h_requested_credits,
				(handle->h_requested_credits -
				 handle->h_total_credits));

	/*
	 * Implement synchronous transaction batching.  If the handle
	 * was synchronous, don't force a commit immediately.  Let's
	 * yield and let another thread piggyback onto this
	 * transaction.  Keep doing that while new threads continue to
	 * arrive.  It doesn't cost much - we're about to run a commit
	 * and sleep on IO anyway.  Speeds up many-threaded, many-dir
	 * operations by 30x or more...
	 *
	 * We try and optimize the sleep time against what the
	 * underlying disk can do, instead of having a static sleep
	 * time.  This is useful for the case where our storage is so
	 * fast that it is more optimal to go ahead and force a flush
	 * and wait for the transaction to be committed than it is to
	 * wait for an arbitrary amount of time for new writers to
	 * join the transaction.  We achieve this by measuring how
	 * long it takes to commit a transaction, and compare it with
	 * how long this transaction has been running, and if run time
	 * < commit time then we sleep for the delta and commit.  This
	 * greatly helps super fast disks that would see slowdowns as
	 * more threads started doing fsyncs.
	 *
	 * But don't do this if this process was the most recent one
	 * to perform a synchronous write.  We do this to detect the
	 * case where a single process is doing a stream of sync
	 * writes.  No point in waiting for joiners in that case.
	 *
	 * Setting max_batch_time to 0 disables this completely.
	 * 如果是同步操作，需要等日志commit完成，反正都要等，不如等的时间长一点
	 * 顺便等一下之前积累的commit
	 */
	pid = current->pid;
	if (handle->h_sync && journal->j_last_sync_writer != pid &&
	    journal->j_max_batch_time) {
		u64 commit_time, trans_time;

		journal->j_last_sync_writer = pid;

		read_lock(&journal->j_state_lock);
		commit_time = journal->j_average_commit_time;
		read_unlock(&journal->j_state_lock);

		trans_time = ktime_to_ns(ktime_sub(ktime_get(),
						   transaction->t_start_time));

		commit_time = max_t(u64, commit_time,
				    1000*journal->j_min_batch_time);
		commit_time = min_t(u64, commit_time,
				    1000*journal->j_max_batch_time);

		if (trans_time < commit_time) {
			ktime_t expires = ktime_add_ns(ktime_get(),
						       commit_time);
			set_current_state(TASK_UNINTERRUPTIBLE);
			schedule_hrtimeout(&expires, HRTIMER_MODE_ABS);
		}
	}

	if (handle->h_sync)
		transaction->t_synchronous_commit = 1;

	/*
	 * If the handle is marked SYNC, we need to set another commit
	 * going!  We also want to force a commit if the transaction is too
	 * old now.
	 * 设置同步标记，或者事务存在了很长时间，要强制提交
	 */
	if (handle->h_sync ||
	    time_after_eq(jiffies, transaction->t_expires)) {
		/* Do this even for aborted journals: an abort still
		 * completes the commit thread, it just doesn't write
		 * anything to disk. */

		jbd2_debug(2, "transaction too old, requesting commit for "
					"handle %p\n", handle);
		/* This is non-blocking */
		jbd2_log_start_commit(journal, tid);//唤醒在journal->j_wait_commit上等待的进程

		/*
		 * Special case: JBD2_SYNC synchronous updates require us
		 * to wait for the commit to complete.
		 */
		if (handle->h_sync && !(current->flags & PF_MEMALLOC))
			wait_for_commit = 1;
	}

	/*
	 * Once stop_this_handle() drops t_updates, the transaction could start
	 * committing on us and eventually disappear.  So we must not
	 * dereference transaction pointer again after calling
	 * stop_this_handle().
	 */
	stop_this_handle(handle);

	if (wait_for_commit)
		err = jbd2_log_wait_commit(journal, tid);//等待ID为tid的commit完成

free_and_exit:
	if (handle->h_rsv_handle)
		jbd2_free_handle(handle->h_rsv_handle);
	jbd2_free_handle(handle);
	return err;
}


static void stop_this_handle(handle_t *handle)
{
	transaction_t *transaction = handle->h_transaction;
	journal_t *journal = transaction->t_journal;
	int revokes;

	J_ASSERT(journal_current_handle() == handle);
	J_ASSERT(atomic_read(&transaction->t_updates) > 0);
	current->journal_info = NULL;
	/*
	 * Subtract necessary revoke descriptor blocks from handle credits. We
	 * take care to account only for revoke descriptor blocks the
	 * transaction will really need as large sequences of transactions with
	 * small numbers of revokes are relatively common.
	 */
	revokes = handle->h_revoke_credits_requested - handle->h_revoke_credits;
	if (revokes) {
		int t_revokes, revoke_descriptors;
		int rr_per_blk = journal->j_revoke_records_per_block;

		WARN_ON_ONCE(DIV_ROUND_UP(revokes, rr_per_blk)
				> handle->h_total_credits);
		t_revokes = atomic_add_return(revokes,
				&transaction->t_outstanding_revokes);
		revoke_descriptors =
			DIV_ROUND_UP(t_revokes, rr_per_blk) -
			DIV_ROUND_UP(t_revokes - revokes, rr_per_blk);
		handle->h_total_credits -= revoke_descriptors;
	}
	atomic_sub(handle->h_total_credits,
		   &transaction->t_outstanding_credits);
	if (handle->h_rsv_handle)
		__jbd2_journal_unreserve_handle(handle->h_rsv_handle,
						transaction);//唤醒journal->j_wait_reserved
	if (atomic_dec_and_test(&transaction->t_updates))
		wake_up(&journal->j_wait_updates);//等待handle完成的等待队列

	rwsem_release(&journal->j_trans_commit_map, _THIS_IP_);
	/*
	 * Scope of the GFP_NOFS context is over here and so we can restore the
	 * original alloc context.
	 */
	memalloc_nofs_restore(handle->saved_alloc_context);
}

```