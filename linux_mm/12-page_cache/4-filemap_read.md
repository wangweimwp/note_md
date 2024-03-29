```c
/**
 * filemap_read - Read data from the page cache.
 * @iocb: The iocb to read.
 * @iter: Destination for the data.
 * @already_read: Number of bytes already read by the caller.
 *
 * Copies data from the page cache.  If the data is not currently present,
 * uses the readahead and read_folio address_space operations to fetch it.
 *
 * Return: Total number of bytes copied, including those already read by
 * the caller.  If an error happens before any bytes are copied, returns
 * a negative error number.
 */
ssize_t filemap_read(struct kiocb *iocb, struct iov_iter *iter,
        ssize_t already_read)
{
    struct file *filp = iocb->ki_filp;
    struct file_ra_state *ra = &filp->f_ra;
    struct address_space *mapping = filp->f_mapping;
    struct inode *inode = mapping->host;
    struct folio_batch fbatch;
    int i, error = 0;
    bool writably_mapped;
    loff_t isize, end_offset;

    if (unlikely(iocb->ki_pos >= inode->i_sb->s_maxbytes))
        return 0;
    if (unlikely(!iov_iter_count(iter)))
        return 0;

    iov_iter_truncate(iter, inode->i_sb->s_maxbytes);
    folio_batch_init(&fbatch);

    do {
        cond_resched();

        /*
         * If we've already successfully copied some data, then we
         * can no longer safely return -EIOCBQUEUED. Hence mark
         * an async read NOWAIT at that point.
         */
        if ((iocb->ki_flags & IOCB_WAITQ) && already_read)
            iocb->ki_flags |= IOCB_NOWAIT;

        if (unlikely(iocb->ki_pos >= i_size_read(inode)))
            break;//本次要读的数据之前已经读过了

        error = filemap_get_pages(iocb, iter->count, &fbatch,
                      iov_iter_is_pipe(iter));
        if (error < 0)
            break;

        /*
         * i_size must be checked after we know the pages are Uptodate.
         *
         * Checking i_size after the check allows us to calculate
         * the correct value for "nr", which means the zero-filled
         * part of the page is not copied back to userspace (unless
         * another truncate extends the file - this is desired though).
         */
        isize = i_size_read(inode);
        if (unlikely(iocb->ki_pos >= isize))
            goto put_folios;/* 读到文件末尾，退出 */
        end_offset = min_t(loff_t, isize, iocb->ki_pos + iter->count);

        /*
         * Once we start copying data, we don't want to be touching any
         * cachelines that might be contended:
         */
        writably_mapped = mapping_writably_mapped(mapping);

        /*
         * When a read accesses the same folio several times, only
         * mark it as accessed the first time.
         */
        if (!pos_same_folio(iocb->ki_pos, ra->prev_pos - 1,
                            fbatch.folios[0]))
            folio_mark_accessed(fbatch.folios[0]);

        for (i = 0; i < folio_batch_count(&fbatch); i++) {
            struct folio *folio = fbatch.folios[i];
            size_t fsize = folio_size(folio);
            size_t offset = iocb->ki_pos & (fsize - 1);
            size_t bytes = min_t(loff_t, end_offset - iocb->ki_pos,
                         fsize - offset);
            size_t copied;

            if (end_offset < folio_pos(folio))
                break;
            if (i > 0)
                folio_mark_accessed(folio);
            /*
             * If users can be writing to this folio using arbitrary
             * virtual addresses, take care of potential aliasing
             * before reading the folio on the kernel side.
             */
            if (writably_mapped)
                flush_dcache_folio(folio);
            /*filemap_get_pages读到的page是PageUptodate的，拷贝到用户空间*/
            copied = copy_folio_to_iter(folio, offset, bytes, iter);

            already_read += copied;
            iocb->ki_pos += copied;
            ra->prev_pos = iocb->ki_pos;/* 更新index，指向下一个page */

            if (copied < bytes) {
                error = -EFAULT;
                break;
            }
        }
put_folios:
        for (i = 0; i < folio_batch_count(&fbatch); i++)
            folio_put(fbatch.folios[i]);
        folio_batch_init(&fbatch);
    } while (iov_iter_count(iter) && iocb->ki_pos < isize && !error);/* read需要的数据已经全部拷贝完了，read流程结束返回 */

    file_accessed(filp);

    return already_read ? already_read : error;
}
```

```c
static int filemap_get_pages(struct kiocb *iocb, size_t count,
        struct folio_batch *fbatch, bool need_uptodate)
{
    /* 根据read系统调用及上一次的readahead信息，计算read的起始index页，
       结束last_index页,最后一个页面内的偏移量offset。 */
    struct file *filp = iocb->ki_filp;
    struct address_space *mapping = filp->f_mapping;
    struct file_ra_state *ra = &filp->f_ra;
    pgoff_t index = iocb->ki_pos >> PAGE_SHIFT;
    pgoff_t last_index;
    struct folio *folio;
    int err = 0;

    /* "last_index" is the index of the page beyond the end of the read */
    last_index = DIV_ROUND_UP(iocb->ki_pos + count, PAGE_SIZE);
retry:
    if (fatal_signal_pending(current))
        return -EINTR;
    /* 循环处理[index, last_index] page，先在page cache中查找,如果能找到并且
       是PageUptodate状态，则将page数据拷贝到用户态buf。如果找不到，触发同步预读 
       page_cache_sync_readahead从存储器件读入一批page(大于
       last_index - index个页面)。并将read需要的最后一个页面（last_index
       对应的页面）的下一个页面设置PageReadahead。访问到这个标记的页面时触发异步预读 
       page_cache_async_readahead从存储器件中再读入一批page，并把第一个page设置
       成PageReadahead。*/
    filemap_get_read_batch(mapping, index, last_index - 1, fbatch);//从adress_space的xarray中，通过index找到一系列folio，这个page是read需要用的
    if (!folio_batch_count(fbatch)) {//在adress_space中查找没有找到
        if (iocb->ki_flags & IOCB_NOIO)
            return -EAGAIN;
       /* page cache中没有找到index的page，只能从存储器件中读取数据了。既然需要
          通过disk io读取数据,就多预读一些page。参数index表示从哪个page开始读取，
          req_size = last_index - index个page表示read需要读多少个page。
          page_cache_sync_readahead一般会读取超过req_size个page，前req_size
          个page是本次read请求的，多读的page本次用不到，提前读出来给后继访问使用。
          多读的page中第一个page设置PageReadahead标记。
          page_cache_sync_readahead申请page内存，并加入到page cache
          中,然后通过submit_bio提交bio请求就返回了，至于器件有没有处理完这个io请求，
          page_cache_sync_readahead不关心，不会等待页面变成PageUptodate。*/
        page_cache_sync_readahead(mapping, ra, filp, index,
                last_index - index);//文件数据没有在page cache中，进行同步预读
        filemap_get_read_batch(mapping, index, last_index - 1, fbatch);
    }
    if (!folio_batch_count(fbatch)) {
        if (iocb->ki_flags & (IOCB_NOWAIT | IOCB_WAITQ))
            return -EAGAIN;
        /* 经过上面的预读,在page cache中大概率能找到index的page。
           如果内存不足申请不到page，page cache中不会有index的page */
        /*page_cache_sync_readahead不会进入内存分配慢速，有可能分配不到内存
        这里再次申请内存，并最终调到aops->read_folio读取单页数据*/
        err = filemap_create_folio(filp, mapping,
                iocb->ki_pos >> PAGE_SHIFT, fbatch);
        if (err == AOP_TRUNCATED_PAGE)
            goto retry;
        return err;
    }

    folio = fbatch->folios[folio_batch_count(fbatch) - 1] //多读的folio中第一个folio设置PageReadahead标记                           
    /* 如果这个page设置了PageReadahead标记，意味着预读窗口中未访问的page不多了，
       需要启动异步预读再读入一批page备用。 */;
    if (folio_test_readahead(folio)) {
        /*触发一次异步预读,因为同步读成功命中，重新取预读更多的页面，也会
          更新ra数据结构*/
        err = filemap_readahead(iocb, filp, mapping, folio, last_index);
        if (err)
            goto err;
    }
    if (!folio_test_uptodate(folio)) {
        if ((iocb->ki_flags & IOCB_WAITQ) &&
            folio_batch_count(fbatch) > 1)
            iocb->ki_flags |= IOCB_NOWAIT;
        /* 如果从page cache中找到的page不是PageUptodate状态，说明器件还没有处理完成，
           则通过wait_on_page_locked_killable等待io处理完成.器件处理完io请求后，用
           最新数据填充page,并释放page lock。*/
        /*由于IO阻塞，异步预读还没有进行，数据还没有更新，这会等待一段时间，
        之后若还数据任然没有更新，则最终调到aops->read_folio现场读取单页数据*/
        err = filemap_update_page(iocb, mapping, count, folio,
                      need_uptodate);
        if (err)
            goto err;
    }

    return 0;
err:
    if (err < 0)
        folio_put(folio);
    if (likely(--fbatch->nr))
        return 0;
    if (err == AOP_TRUNCATED_PAGE)
        goto retry;
    return err;
}
```

```c
static int filemap_update_page(struct kiocb *iocb,
        struct address_space *mapping, size_t count,
        struct folio *folio, bool need_uptodate)
{
    int error;

    if (iocb->ki_flags & IOCB_NOWAIT) {
        if (!filemap_invalidate_trylock_shared(mapping))
            return -EAGAIN;
    } else {
        filemap_invalidate_lock_shared(mapping);
    }

    if (!folio_trylock(folio)) {
        error = -EAGAIN;
        if (iocb->ki_flags & (IOCB_NOWAIT | IOCB_NOIO))
            goto unlock_mapping;
        if (!(iocb->ki_flags & IOCB_WAITQ)) {
            filemap_invalidate_unlock_shared(mapping);
            /*
             * This is where we usually end up waiting for a
             * previously submitted readahead to finish.
             */ 
            /* 器件处理完io，用最新数据填充page，然后unlock page */
            folio_put_wait_locked(folio, TASK_KILLABLE);
            return AOP_TRUNCATED_PAGE;
        }
        error = __folio_lock_async(folio, iocb->ki_waitq);
        if (error)
            goto unlock_mapping;
    }

    error = AOP_TRUNCATED_PAGE;
    if (!folio->mapping)
        goto unlock;

    error = 0;
/* 大概率是Uptodate状态.除非存在竞争场景，其他执行流先拿到了page lock，
   并做了修改page状态 */ 
/* page的部分数据是uptodate状态。page大小与存储器件block大小不一样时会出现这种情况，
   一个page可能包含多个block，page中某些block数据是uptodate */
     if (filemap_range_uptodate(mapping, iocb->ki_pos, count, folio,
                   need_uptodate))
        goto unlock;//page是Uptodate状态，或部分数据是uptodate状态

    error = -EAGAIN;
    if (iocb->ki_flags & (IOCB_NOIO | IOCB_NOWAIT | IOCB_WAITQ))
        goto unlock;
    /* 正常情况下，通过同步预读或异步预读加入到page cache的page，最终都
       应该是PageUptodate状态。但存在一种可能，page虽然加入了page cache，
       但同步或异步预读发生了io err导致了page中无有效有效数据。这个时候就要
       通过apping->a_ops->readpage重新读取单个page数据 
        或者mapping->a_ops->is_partially_uptodate部分PageUptodate判断失败*/
    error = filemap_read_folio(iocb->ki_filp, mapping->a_ops->read_folio,
            folio);
    goto unlock_mapping;
unlock:
    folio_unlock(folio);
unlock_mapping:
    filemap_invalidate_unlock_shared(mapping);
    if (error == AOP_TRUNCATED_PAGE)
        folio_put(folio);
    return error;
}
```

```c
/*
 * A minimal readahead algorithm for trivial sequential/random reads.
 @offset: 从哪个page开始读 
   @req_size: 需要读多少个page，这是read需要的数量
   如果是顺序读，ondemand_readahead读取至少req_size个page。
   如果是随机读，ondemand_readahead仅读取req_szie个page。

*/
static void ondemand_readahead(struct readahead_control *ractl,
        struct folio *folio, unsigned long req_size)
{
    struct backing_dev_info *bdi = inode_to_bdi(ractl->mapping->host);
    struct file_ra_state *ra = ractl->ra;
    unsigned long max_pages = ra->ra_pages;
    unsigned long add_pages;
    pgoff_t index = readahead_index(ractl);
    pgoff_t expected, prev_index;
    unsigned int order = folio ? folio_order(folio) : 0;

    /*
     * If the request exceeds the readahead window, allow the read to
     * be up to the optimal hardware IO size
     */
/* read请求的page大于预读窗口大小，预读的page数量上调至存储器件单次io最大的page数量 */
    if (req_size > max_pages && bdi->io_pages > max_pages)
        max_pages = min(req_size, bdi->io_pages);

    /*
     * start of file
     */
    if (!index)
        goto initial_readahead;

    /*
     * It's the expected callback index, assume sequential access.
     * Ramp up sizes, and push forward the readahead window.
     */
/*  条件1：offset == ra->start + ra->size - ra->async_size

             /          同步建立的预读窗口            \
            /                                        \
           |------------------------------------------|
           |  |  |  |  | ^ |  |  |  |  |  |  |  |  |  |   
           |-------------|----------------------------|
                         |
             ra->start + ra->size - ra->async_size
             该page设置了PageReadahead


           上图是同步预读建立的预读窗口，从第0个page到第ra->size - ra->async_size
           个page，是read需要用的，后面的page是预留给下次read用的。
           读到第ra->size - ra->async_size个pag，表明预读窗口已经开始使用，
           需要启动异步预读，读入一批page建立一个新的预读窗口，为下下次read做准备。

    条件2：offset == ra->start + ra->size
              
              前一次     /          异步建立的预读窗口           \
             预读窗口   /                                        \
           ------------|------------------------------------------|
              |  |  |  | ^ |  |  |  |  |  |  |  |  |  |  |  |  |  |   
          ---------------|----------------------------------------|
                         |
          ra->start + ra->size(前一次预读用完了，执行下一个预读窗口的第一个page)
          该page设置了PageReadahead

          异步预读建立的预读窗口中，第一个page标记为PageReadahead，访问到这个标记的
          页面，表明新预读窗口已经开始使用，需要启动异步预读，读入一批page建立一个新的
          预读窗口，为下下次read做准备。

    满足这两个条件，可认为本次read预上一次read是顺序读类型 (见原理部分描述）。
    这个条件是不准确的，但是代码实现简单高效。*/
    expected = round_up(ra->start + ra->size - ra->async_size,
            1UL << order);
    if (index == expected || index == (ra->start + ra->size)) {
        ra->start += ra->size;
        /*既然是顺序读，就扩大预读窗口：
      如果当前预读窗口大小ra->size < 1/16 * 允许读的最多page数
      1）则预读窗口在原基础上扩大4倍。否则扩大2倍。
      2）扩大后的窗口大小，不能超过“允许读的最多page数量”
        */
        ra->size = get_next_ra_size(ra, max_pages);
        ra->async_size = ra->size;
        goto readit;
    }

    /*
     * Hit a marked folio without valid readahead state.
     * E.g. interleaved reads.
     * Query the pagecache for async_size, which normally equals to
     * readahead size. Ramp it up and use it as the new readahead size.
     */
/* 多线程顺序读一个文件（interleaved read），破坏了file->ra_state，所以没法根据前面的条件
   判断出是顺序读类型。如果是顺序读，最终一定会访问到预读窗口中的PageReadahead标记的page。
   所以，反过来推，访问到PageReadahead标记的page，就认为是顺序读。当然这样的判断是不准确的，
   但准确率高，代码简单。*/
    if (folio) {    //hit_readahead_marker
        pgoff_t start;
/* offset + 1开始，找到第一个不在page cache中的文件数据页，作为预读的开始 */
        rcu_read_lock();
        start = page_cache_next_miss(ractl->mapping, index + 1,
                max_pages);
        rcu_read_unlock();

        if (!start || start - index > max_pages)
            return;

        ra->start = start;
        ra->size = start - index;    /* old async_size */
        ra->size += req_size;
        ra->size = get_next_ra_size(ra, max_pages);
        ra->async_size = ra->size;
        goto readit;
    }

    /*
     * oversize read
     */
    if (req_size > max_pages)
        goto initial_readahead;

    /*
     * sequential cache miss
     * trivial case: (index - prev_index) == 1
     * unaligned reads: (index - prev_index) == 0
     */
/* 上面的场景根据下面3个条件判断是否是顺序读：
   offset == ra->start + ra->size - ra->async_size
   offset == ra->start + ra->size
   hit_readahead_marker
   代码执行到这里，属于随机读场景。
*/
    prev_index = (unsigned long long)ra->prev_pos >> PAGE_SHIFT;
    if (index - prev_index <= 1UL)
        goto initial_readahead;

    /*
     * Query the page cache and look for the traces(cached history pages)
     * that a sequential stream would leave behind.
     */
    if (try_context_readahead(ractl->mapping, ra, index, req_size,
            max_pages))
        goto readit;

    /*
     * standalone, small random read
     * Read as is, and do not pollute the readahead state.
     */
/* read请求req_size，__do_page_cache_readahead只读req_size个page。注意2点：
   1）__do_page_cache_readahead-->read_pages条件io就返回了，不会等待page变成PageUptodate。
   2）__do_page_cache_readahead不会更改 struct file_ra_state。 
*/
    do_page_cache_ra(ractl, req_size, 0);
    return;

initial_readahead:
/* 第一次读文件，初始化预读窗口 */
    ra->start = index;
    ra->size = get_init_ra_size(req_size, max_pages);
    ra->async_size = ra->size > req_size ? ra->size - req_size : ra->size;

readit:
    /*
     * Will this read hit the readahead marker made by itself?
     * If so, trigger the readahead marker hit now, and merge
     * the resulted next readahead window into the current one.
     * Take care of maximum IO pages as above.
     */
/* hit_readahead_marker预期建立了一个预读窗口A，待访问offset刚好是这个预读窗口的第一个page，
   这说明访问A窗口，后面没有预留的预读窗口了，需建立下一个预读窗口B备用，既然A、B都还没预读，
   就合并后一起预读。*/
    if (index == ra->start && ra->size == ra->async_size) {
        add_pages = get_next_ra_size(ra, max_pages);
        if (ra->size + add_pages <= max_pages) {
            ra->async_size = add_pages;
            ra->size += add_pages;
        } else {
            ra->size = max_pages;
            ra->async_size = max_pages >> 1;
        }
    }
/* ra_submit --> __do_page_cache_readahead尽量分配ra->size个page，
   分配失败就停止分配，这些page加入page_pool链表。

   read-->page_cache_sync_readahead，假设read需要读取nr_to_read个page，
   预读M个page，则第nr_to_read个page（从0开始计数）设置PageReadahead标记。

   read-->page_cache_async_readahead，则预读的第0个page（从0开始计数）设置
   PageReadahead标记。

   ra_submit -->__do_page_cache_readahead --> read_pages调用
   mapping->a_ops->readpages或者mapping->a_ops->readpage依次读入page_pool
   链表中的page。

   ra_submit提交io后就返回，不会等待page变成PageUptodate。
*/
    ractl->_index = ra->start;
    page_cache_ra_order(ractl, ra, order);
}
```
