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
```