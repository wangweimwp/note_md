
```c
ext4_ext_map_blocks
	->ext4_find_extent//找到一个合适的extern叶子节点
	->ext4_ext_handle_unwritten_extents//找到了合适的extern节点，需要进行分割
		->ext4_ext_convert_to_initialized//找到的extern节点可能不会恰好符合需求的block数，需要分割、合并、清零等操作
			->//执行合并、清零等操作
			->ext4_split_extent//执行分割操作
				->ext4_split_extent_at//将extern分割为2段
				->ext4_find_extent//分割后，更新路径
	->ext4_ext_determine_insert_hole
	->ext4_mb_new_blocks//没有找到合适的extern节点，新建extern
	->ext4_ext_insert_extent//把新申请的extent 插入B+树
		->ext4_ext_create_new_leaf	//执行到该函数，说明ext4 extent B+树中与newext->ee_block有关的叶子节点ext4_extent结构爆满了，需要扩容
			->ext4_ext_split	//有空间可存放索引节点，于是从ext4 extent B+树at那一层索引节点到叶子节点，针对每一层都创建新的索引节点，也创建叶子节点
			->ext4_ext_grow_indepth //到这个分支，ext4 extent B+树索引节点的ext4_extent_idx和叶子节点的ext4_extent个数全爆满 增加B+树的层级

/*black 申请/映射/预申请过程
ext4_ext_map_blocks()函数主要流程有如下几点：

1：先执行ext4_ext_find_extent()，试图找到逻辑块地址最接近map->m_lblk的
索引节点ext4_extent_idr结构和叶子节点ext4_extent结构并保存到path[]。
ext4_ext_find_extent()函数源码下文详解。如果找到匹配的叶子节点ext4_extent结构，
则ex = path[depth].p_ext保存这个找到的ext4_extent结构。此时if (ex)成立，
如果map->m_lblk在ex的逻辑块地址范围内，即if (in_range(map->m_lblk, ee_block, ee_len))成立，
则执行里边代码newblock = map->m_lblk - ee_block + ee_start和
allocated = ee_len - (map->m_lblk - ee_block)，
通过ex已经映射的逻辑块地址和物理块地址找到map->m_lblk映射的起始物理块，
allocated是找到的映射的物理块个数。简单说，
本次要映射的起始逻辑块地址map->m_lblk在ex的逻辑块地址范围内，
那就可以借助ex这个ext4_extent已有的逻辑块地址与物理块地址映射关系，
找到map->m_lblk映射的起始物理块地址，并找到已经映射过的allocated个物理块。

2：如果ex是已初始化状态，则if (!ext4_ext_is_uninitialized(ex))成立，
直接goto out 。否则ex未初始化状态，
则要执行ext4_ext_handle_uninitialized_extents()->ext4_ext_convert_to_initialized()
对ex的逻辑块地址进行分割，还有概率创建新的索引节点和叶子节点。
高版本内核 ext4_ext_handle_uninitialized_extents()函数名称改为 ext4_ext_handle_unwritten_extents()，
需注意，ext4_ext_convert_to_initialized()源码下文详解。

3：继续ext4_ext_map_blocks()代码，如果ex = path[depth].p_ext是NULL，则if (ex)不成立。
则执行下文newblock = ext4_mb_new_blocks(handle, &ar, &err)函数，
针对本次需映射的起始逻辑块地址map->m_lblk和需映射的逻辑块个数map->m_len，
分配map->m_len个连续的物理块并返回这map->m_len个物理块的第一个物理块号给newblock。

4：接着先执行newex.ee_block = cpu_to_le32(map->m_lblk)赋值本次要映射的起始逻辑块地址，
newex是针对本次逻辑块地址映射创建的ext4_extent结构。
然后执行ext4_ext_store_pblock(&newex, newblock + offset)
把本次逻辑块地址map->m_lblk ~( map->m_lblk+ map->m_len)
映射的起始物理块号newblock保存到newex的ee_start_lo和ee_start_hi。
并执行newex.ee_len = cpu_to_le16(ar.len)把成功映射的物理块数保存到newex.ee_len，即map->m_len。

5：执行err = ext4_ext_insert_extent(handle, inode, path,&newex, flags)
把代表本次逻辑块地址映射关系的newex插入到ext4  extent b+树。
*/
/*
 * Block allocation/map/preallocation routine for extents based files
 *
 *
 * Need to be called with
 * down_read(&EXT4_I(inode)->i_data_sem) if not allocating file system block
 * (ie, flags is zero). Otherwise down_write(&EXT4_I(inode)->i_data_sem)
 *
 * return > 0, number of blocks already mapped/allocated
 *          if flags doesn't contain EXT4_GET_BLOCKS_CREATE and these are pre-allocated blocks
 *          	buffer head is unmapped
 *          otherwise blocks are mapped
 *
 * return = 0, if plain look up failed (blocks have not been allocated)
 *          buffer head is unmapped
 *
 * return < 0, error case.
 */
int ext4_ext_map_blocks(handle_t *handle, struct inode *inode,
			struct ext4_map_blocks *map, int flags)
{
	struct ext4_ext_path *path = NULL;
	struct ext4_extent newex, *ex, ex2;
	struct ext4_sb_info *sbi = EXT4_SB(inode->i_sb);
	ext4_fsblk_t newblock = 0, pblk;
	int err = 0, depth;
	unsigned int allocated = 0, offset = 0;
	unsigned int allocated_clusters = 0;
	struct ext4_allocation_request ar;
	ext4_lblk_t cluster_offset;

	ext_debug(inode, "blocks %u/%u requested\n", map->m_lblk, map->m_len);
	trace_ext4_ext_map_blocks_enter(inode, map->m_lblk, map->m_len, flags);

	/* find extent for this block */
	path = ext4_find_extent(inode, map->m_lblk, NULL, 0);//下文详解
	if (IS_ERR(path)) {
		err = PTR_ERR(path);
		goto out;
	}

	depth = ext_depth(inode);

	/*
	 * consistent leaf must not be empty;
	 * this situation is possible, though, _during_ tree modification;
	 * this is why assert can't be put in ext4_find_extent()
	 */
	if (unlikely(path[depth].p_ext == NULL && depth != 0)) {
		EXT4_ERROR_INODE(inode, "bad extent address "
				 "lblock: %lu, depth: %d pblock %lld",
				 (unsigned long) map->m_lblk, depth,
				 path[depth].p_block);
		err = -EFSCORRUPTED;
		goto out;
	}

	ex = path[depth].p_ext;
	if (ex) {
		ext4_lblk_t ee_block = le32_to_cpu(ex->ee_block);
		ext4_fsblk_t ee_start = ext4_ext_pblock(ex);
		unsigned short ee_len;


		/*
		 * unwritten extents are treated as holes, except that
		 * we split out initialized portions during a write.
		 */
		ee_len = ext4_ext_get_actual_len(ex);

		trace_ext4_ext_show_extent(inode, ee_block, ee_start, ee_len);

		/* if found extent covers block, simply return it */
		if (in_range(map->m_lblk, ee_block, ee_len)) {
			newblock = map->m_lblk - ee_block + ee_start;//逻辑块的偏移+起始物理块=当前的物理块号
			/* number of remaining blocks in the extent */
			allocated = ee_len - (map->m_lblk - ee_block);//长度-(目标逻辑块-起始逻辑块)=没有用的块数
			ext_debug(inode, "%u fit into %u:%d -> %llu\n",
				  map->m_lblk, ee_block, ee_len, newblock);

			/*
			 * If the extent is initialized check whether the
			 * caller wants to convert it to unwritten.
				unwritten的extent的长度不可能是0
				ee_len的MSB为0或等于0x8000则是initiakized 其他则是 unwritten
			 */
			if ((!ext4_ext_is_unwritten(ex)) &&
			    (flags & EXT4_GET_BLOCKS_CONVERT_UNWRITTEN)) {
				path = convert_initialized_extent(handle,
					inode, map, path, &allocated);
				if (IS_ERR(path))
					err = PTR_ERR(path);
				goto out;
			} else if (!ext4_ext_is_unwritten(ex)) {
				map->m_flags |= EXT4_MAP_MAPPED;
				map->m_pblk = newblock;
				if (allocated > map->m_len)
					allocated = map->m_len;
				map->m_len = allocated;
				ext4_ext_show_leaf(inode, path);
				goto out;
			}

			path = ext4_ext_handle_unwritten_extents(
				handle, inode, map, path, flags,
				&allocated, newblock);
			if (IS_ERR(path))
				err = PTR_ERR(path);
			goto out;
		}
	}

	/*
	 * requested block isn't allocated yet;
	 * we couldn't try to create block if flags doesn't contain EXT4_GET_BLOCKS_CREATE
	 */
	if ((flags & EXT4_GET_BLOCKS_CREATE) == 0) {
		ext4_lblk_t len;

		len = ext4_ext_determine_insert_hole(inode, path, map->m_lblk);

		map->m_pblk = 0;
		map->m_len = min_t(unsigned int, map->m_len, len);
		goto out;
	}

	/*
	 * Okay, we need to do block allocation.
	 */
	newex.ee_block = cpu_to_le32(map->m_lblk);
	cluster_offset = EXT4_LBLK_COFF(sbi, map->m_lblk);

	/*
	 * If we are doing bigalloc, check to see if the extent returned
	 * by ext4_find_extent() implies a cluster we can use.
	 */
	if (cluster_offset && ex &&
	    get_implied_cluster_alloc(inode->i_sb, map, ex, path)) {
		ar.len = allocated = map->m_len;
		newblock = map->m_pblk;
		goto got_allocated_blocks;
	}

	/* find neighbour allocated blocks */
	ar.lleft = map->m_lblk;
	err = ext4_ext_search_left(inode, path, &ar.lleft, &ar.pleft);
	if (err)
		goto out;
	ar.lright = map->m_lblk;
	err = ext4_ext_search_right(inode, path, &ar.lright, &ar.pright, &ex2);
	if (err < 0)
		goto out;

	/* Check if the extent after searching to the right implies a
	 * cluster we can use. */
	if ((sbi->s_cluster_ratio > 1) && err &&
	    get_implied_cluster_alloc(inode->i_sb, map, &ex2, path)) {
		ar.len = allocated = map->m_len;
		newblock = map->m_pblk;
		err = 0;
		goto got_allocated_blocks;
	}

	/*
	 * See if request is beyond maximum number of blocks we can have in
	 * a single extent. For an initialized extent this limit is
	 * EXT_INIT_MAX_LEN and for an unwritten extent this limit is
	 * EXT_UNWRITTEN_MAX_LEN.
	 */
	if (map->m_len > EXT_INIT_MAX_LEN &&
	    !(flags & EXT4_GET_BLOCKS_UNWRIT_EXT))
		map->m_len = EXT_INIT_MAX_LEN;
	else if (map->m_len > EXT_UNWRITTEN_MAX_LEN &&
		 (flags & EXT4_GET_BLOCKS_UNWRIT_EXT))
		map->m_len = EXT_UNWRITTEN_MAX_LEN;

	/* Check if we can really insert (m_lblk)::(m_lblk + m_len) extent */
	newex.ee_len = cpu_to_le16(map->m_len);
	err = ext4_ext_check_overlap(sbi, inode, &newex, path);
	if (err)
		allocated = ext4_ext_get_actual_len(&newex);
	else
		allocated = map->m_len;

	/* allocate new block */
	ar.inode = inode;
	ar.goal = ext4_ext_find_goal(inode, path, map->m_lblk);//找到map->m_lblk映射的目标起始物理块地址并返回给ar.goal
	ar.logical = map->m_lblk;
	/*
	 * We calculate the offset from the beginning of the cluster
	 * for the logical block number, since when we allocate a
	 * physical cluster, the physical block should start at the
	 * same offset from the beginning of the cluster.  This is
	 * needed so that future calls to get_implied_cluster_alloc()
	 * work correctly.
	 */
	offset = EXT4_LBLK_COFF(sbi, map->m_lblk);
	ar.len = EXT4_NUM_B2C(sbi, offset+allocated);
	ar.goal -= offset;
	ar.logical -= offset;
	if (S_ISREG(inode->i_mode))
		ar.flags = EXT4_MB_HINT_DATA;
	else
		/* disable in-core preallocation for non-regular files */
		ar.flags = 0;
	if (flags & EXT4_GET_BLOCKS_NO_NORMALIZE)
		ar.flags |= EXT4_MB_HINT_NOPREALLOC;
	if (flags & EXT4_GET_BLOCKS_DELALLOC_RESERVE)
		ar.flags |= EXT4_MB_DELALLOC_RESERVED;
	if (flags & EXT4_GET_BLOCKS_METADATA_NOFAIL)
		ar.flags |= EXT4_MB_USE_RESERVED;
	/*分配map->m_len个物理块，这就是newex逻辑块地址映射的map->m_len个物理块，
	并返回这map->m_len个物理块的起始物理块号newblock。测试结果 newblock 
	和 ar.goal有时相等，有时不相等。本次映射的起始逻辑块地址是map->m_lblk，
	映射物理块个数map->m_len，ext4_mb_new_blocks()除了要找到newblock这个起始逻辑块地址，
	还得保证找到newblock打头的连续map->m_len个物理块，必须是连续的，这才是更重要的。*/
	newblock = ext4_mb_new_blocks(handle, &ar, &err);
	if (!newblock)
		goto out;
	allocated_clusters = ar.len;
	ar.len = EXT4_C2B(sbi, ar.len) - offset;
	ext_debug(inode, "allocate new block: goal %llu, found %llu/%u, requested %u\n",
		  ar.goal, newblock, ar.len, allocated);
	if (ar.len > allocated)
		ar.len = allocated;

got_allocated_blocks:
	/* try to insert new extent into found leaf and return */
	pblk = newblock + offset;
	ext4_ext_store_pblock(&newex, pblk); /*设置本次映射的map->m_len个物理块的起始物理块号(newblock)到newex，newex是针对本次映射分配的ext4_extent结构*/
	newex.ee_len = cpu_to_le16(ar.len);/*设置newex映射的物理块个数，与执行ext4_ext_mark_initialized()标记ex已初始化一个效果*/
	/* Mark unwritten */
	if (flags & EXT4_GET_BLOCKS_UNWRIT_EXT) {
		ext4_ext_mark_unwritten(&newex);
		map->m_flags |= EXT4_MAP_UNWRITTEN;
	}

	path = ext4_ext_insert_extent(handle, inode, path, &newex, flags);//把newex这个插入ext4 extent B+树
	if (IS_ERR(path)) {
		err = PTR_ERR(path);
		if (allocated_clusters) {
			int fb_flags = 0;

			/*
			 * free data blocks we just allocated.
			 * not a good idea to call discard here directly,
			 * but otherwise we'd need to call it every free().
			 */
			ext4_discard_preallocations(inode);
			if (flags & EXT4_GET_BLOCKS_DELALLOC_RESERVE)
				fb_flags = EXT4_FREE_BLOCKS_NO_QUOT_UPDATE;
			ext4_free_blocks(handle, inode, NULL, newblock,
					 EXT4_C2B(sbi, allocated_clusters),
					 fb_flags);
		}
		goto out;
	}

	/*
	 * Cache the extent and update transaction to commit on fdatasync only
	 * when it is _not_ an unwritten extent.
	 */
	if ((flags & EXT4_GET_BLOCKS_UNWRIT_EXT) == 0)
		ext4_update_inode_fsync_trans(handle, inode, 1);
	else
		ext4_update_inode_fsync_trans(handle, inode, 0);

	map->m_flags |= (EXT4_MAP_NEW | EXT4_MAP_MAPPED);
	map->m_pblk = pblk;
	map->m_len = ar.len; /*本次逻辑块地址完成映射的物理块数，并不能保证allocated等于传入的map->m_len，还有可能小于*/
	allocated = map->m_len;
	ext4_ext_show_leaf(inode, path);
out:
	ext4_free_ext_path(path);

	trace_ext4_ext_map_blocks_exit(inode, flags, map,
				       err ? err : allocated);
	return err ? err : allocated;
}




struct ext4_ext_path *
ext4_find_extent(struct inode *inode, ext4_lblk_t block,
		 struct ext4_ext_path *path, int flags)
{
	struct ext4_extent_header *eh;
	struct buffer_head *bh;
	short int depth, i, ppos = 0;
	int ret;
	gfp_t gfp_flags = GFP_NOFS;

	if (flags & EXT4_EX_NOFAIL)
		gfp_flags |= __GFP_NOFAIL;

	eh = ext_inode_hdr(inode);//拿到inode中的extent_header
	depth = ext_depth(inode);//解析extent深度
	if (depth < 0 || depth > EXT4_MAX_EXTENT_DEPTH) {
		EXT4_ERROR_INODE(inode, "inode has invalid extent depth: %d",
				 depth);
		ret = -EFSCORRUPTED;

		goto err;
	}

	if (path) {
		ext4_ext_drop_refs(path);
		if (depth > path[0].p_maxdepth) {
			kfree(path);
			path = NULL;
		}
	}
	if (!path) {
		/* account possible depth increase */
		path = kcalloc(depth + 2, sizeof(struct ext4_ext_path),
				gfp_flags);
		if (unlikely(!path))
			return ERR_PTR(-ENOMEM);
		path[0].p_maxdepth = depth + 1;
	}
	path[0].p_hdr = eh;
	path[0].p_bh = NULL;

	i = depth;
	if (!(flags & EXT4_EX_NOCACHE) && depth == 0)
		ext4_cache_extents(inode, eh);
	/* walk through the tree */
	while (i) {//遍历extent的每个层级，对于小文件，depth为0
		ext_debug(inode, "depth %d: num %d, max %d\n",
			  ppos, le16_to_cpu(eh->eh_entries), le16_to_cpu(eh->eh_max));

		ext4_ext_binsearch_idx(inode, path + ppos, block);//根据block查询ext4_extent_idx结构体，放到path数组中
		path[ppos].p_block = ext4_idx_pblock(path[ppos].p_idx);
		path[ppos].p_depth = i;
		path[ppos].p_ext = NULL;

		bh = read_extent_tree_block(inode, path[ppos].p_idx, --i, flags);//从存储介质中读出ext4_extent_idx指向的数据块
		if (IS_ERR(bh)) {
			ret = PTR_ERR(bh);
			goto err;
		}

		eh = ext_block_hdr(bh);
		ppos++;
		path[ppos].p_bh = bh;
		path[ppos].p_hdr = eh;
	}

	path[ppos].p_depth = i;
	path[ppos].p_ext = NULL;
	path[ppos].p_idx = NULL;

	/* find extent */
	ext4_ext_binsearch(inode, path + ppos, block);//找到叶子ext4_extent
	/* if not an empty leaf */
	if (path[ppos].p_ext)
		path[ppos].p_block = ext4_ext_pblock(path[ppos].p_ext);//找到物理块号

	ext4_ext_show_path(inode, path);

	return path;

err:
	ext4_free_ext_path(path);
	return ERR_PTR(ret);
}







/*如果本次要映射的物理块数(或者逻辑块数)map->len小于ex已经映射的逻辑块数ee_len，
则尝试把ex的map->len的逻辑块合并到它前边或者后边的ext4_extent结构(即abut_ex)。
合并条件苛刻，需要二者逻辑块地址和物理块地址紧挨着等等。
如果合并成功直接从ext4_ext_convert_to_initialized()函数返回。
否则执行ext4_split_extent()把ex的逻辑块地址进程分割成2段或者3段，
分割出的以map->m_lblk为起始地址且共allocated个逻辑块的逻辑块范围就是我们需要的，
这allocated个逻辑块可以保证映射了物理块。但allocated<=map->len，
即并不能保证map要求映射的map->len个逻辑块全映射完成。注意，ext4_split_extent()对ex分割后，
还剩下其他1~2段逻辑块范围，则要把它们对应的ext4_extent结构插入的ext4_extent B+树。

 * This function is called by ext4_ext_map_blocks() if someone tries to write
 * to an unwritten extent. It may result in splitting the unwritten
 * extent into multiple extents (up to three - one initialized and two
 * unwritten).
 * There are three possibilities:
 *   a> There is no split required: Entire extent should be initialized
 *   b> Splits in two extents: Write is happening at either end of the extent
 *   c> Splits in three extents: Somone is writing in middle of the extent
 *
 * Pre-conditions:
 *  - The extent pointed to by 'path' is unwritten.
 *  - The extent pointed to by 'path' contains a superset
 *    of the logical span [map->m_lblk, map->m_lblk + map->m_len).
 *
 * Post-conditions on success:
 *  - the returned value is the number of blocks beyond map->l_lblk
 *    that are allocated and initialized.
 *    It is guaranteed to be >= map->m_len.
 */
static struct ext4_ext_path *
ext4_ext_convert_to_initialized(handle_t *handle, struct inode *inode,
			struct ext4_map_blocks *map, struct ext4_ext_path *path,
			int flags, unsigned int *allocated)
{
	struct ext4_sb_info *sbi;
	struct ext4_extent_header *eh;
	struct ext4_map_blocks split_map;
	struct ext4_extent zero_ex1, zero_ex2;
	struct ext4_extent *ex, *abut_ex;
	ext4_lblk_t ee_block, eof_block;
	unsigned int ee_len, depth, map_len = map->m_len;
	int err = 0;
	int split_flag = EXT4_EXT_DATA_VALID2;
	unsigned int max_zeroout = 0;

	ext_debug(inode, "logical block %llu, max_blocks %u\n",
		  (unsigned long long)map->m_lblk, map_len);

	sbi = EXT4_SB(inode->i_sb);
	eof_block = (EXT4_I(inode)->i_disksize + inode->i_sb->s_blocksize - 1)
			>> inode->i_sb->s_blocksize_bits;//文件的大小
	if (eof_block < map->m_lblk + map_len)//文件有变大
		eof_block = map->m_lblk + map_len;

	depth = ext_depth(inode);
	eh = path[depth].p_hdr;
	ex = path[depth].p_ext;
	ee_block = le32_to_cpu(ex->ee_block);
	ee_len = ext4_ext_get_actual_len(ex);
	zero_ex1.ee_len = 0;
	zero_ex2.ee_len = 0;

	trace_ext4_ext_convert_to_initialized_enter(inode, map, ex);

	/* Pre-conditions */
	BUG_ON(!ext4_ext_is_unwritten(ex));
	BUG_ON(!in_range(map->m_lblk, ee_block, ee_len));

	/*
	 * Attempt to transfer newly initialized blocks from the currently
	 * unwritten extent to its neighbor. This is much cheaper
	 * than an insertion followed by a merge as those involve costly
	 * memmove() calls. Transferring to the left is the common case in
	 * steady state for workloads doing fallocate(FALLOC_FL_KEEP_SIZE)
	 * followed by append writes.
	 *
	 * Limitations of the current logic:
	 *  - L1: we do not deal with writes covering the whole extent.
	 *    This would require removing the extent if the transfer
	 *    is possible.
	 *  - L2: we only attempt to merge with an extent stored in the
	 *    same extent tree node.
	 */
	*allocated = 0;
	if ((map->m_lblk == ee_block) &&//本次映射的起始逻辑块等于找到的extern的起始逻辑块
		/* See if we can merge left */
		(map_len < ee_len) &&		/*L1*/ //本次映射的长度小于找到的extern的长度
		(ex > EXT_FIRST_EXTENT(eh))) {	/*L2*///不是第一个extern
		ext4_lblk_t prev_lblk;
		ext4_fsblk_t prev_pblk, ee_pblk;
		unsigned int prev_len;

		abut_ex = ex - 1;//当前extern的前一个extern
		prev_lblk = le32_to_cpu(abut_ex->ee_block);
		prev_len = ext4_ext_get_actual_len(abut_ex);
		prev_pblk = ext4_ext_pblock(abut_ex);
		ee_pblk = ext4_ext_pblock(ex);

		/*
		 * A transfer of blocks from 'ex' to 'abut_ex' is allowed
		 * upon those conditions:可以与前一个extent合并的条件
		 * - C1: abut_ex is initialized, 前一个extent是初始化过的
		 * - C2: abut_ex is logically abutting ex,  逻辑块相邻
		 * - C3: abut_ex is physically abutting ex, 物理块相邻
		 * - C4: abut_ex can receive the additional blocks without合并后长度不会超过限度
		 *   overflowing the (initialized) length limit.
		 */
		if ((!ext4_ext_is_unwritten(abut_ex)) &&		/*C1*/
			((prev_lblk + prev_len) == ee_block) &&		/*C2*/
			((prev_pblk + prev_len) == ee_pblk) &&		/*C3*/
			(prev_len < (EXT_INIT_MAX_LEN - map_len))) {	/*C4*/
			err = ext4_ext_get_access(handle, inode, path + depth);
			if (err)
				goto errout;

			trace_ext4_ext_convert_to_initialized_fastpath(inode,
				map, ex, abut_ex);

			/* Shift the start of ex by 'map_len' blocks */
			ex->ee_block = cpu_to_le32(ee_block + map_len);
			ext4_ext_store_pblock(ex, ee_pblk + map_len);
			ex->ee_len = cpu_to_le16(ee_len - map_len);
			ext4_ext_mark_unwritten(ex); /* Restore the flag */

			/* Extend abut_ex by 'map_len' blocks */
			abut_ex->ee_len = cpu_to_le16(prev_len + map_len);

			/* Result: number of initialized blocks past m_lblk */
			*allocated = map_len;//申请到map_len个block
		}
	} else if (((map->m_lblk + map_len) == (ee_block + ee_len)) &&//本次映射的结束逻辑块等于找到的extern的结束逻辑块
		   (map_len < ee_len) &&	/*L1*///本次映射的长度小于找到的extern的长度
		   ex < EXT_LAST_EXTENT(eh)) {	/*L2*///不是最后一个extern
		/* See if we can merge right */
		ext4_lblk_t next_lblk;
		ext4_fsblk_t next_pblk, ee_pblk;
		unsigned int next_len;

		abut_ex = ex + 1;//当前extern的下一个extern
		next_lblk = le32_to_cpu(abut_ex->ee_block);
		next_len = ext4_ext_get_actual_len(abut_ex);
		next_pblk = ext4_ext_pblock(abut_ex);
		ee_pblk = ext4_ext_pblock(ex);

		/*
		 * A transfer of blocks from 'ex' to 'abut_ex' is allowed
		 * upon those conditions:可以与下一个extent合并的条件
		 * - C1: abut_ex is initialized, 下一个extent是初始化过的
		 * - C2: abut_ex is logically abutting ex,逻辑块相邻
		 * - C3: abut_ex is physically abutting ex,物理块相邻
		 * - C4: abut_ex can receive the additional blocks without合并后长度不会超过限度
		 *   overflowing the (initialized) length limit.
		 */
		if ((!ext4_ext_is_unwritten(abut_ex)) &&		/*C1*/
		    ((map->m_lblk + map_len) == next_lblk) &&		/*C2*/
		    ((ee_pblk + ee_len) == next_pblk) &&		/*C3*/
		    (next_len < (EXT_INIT_MAX_LEN - map_len))) {	/*C4*/
			err = ext4_ext_get_access(handle, inode, path + depth);
			if (err)
				goto errout;

			trace_ext4_ext_convert_to_initialized_fastpath(inode,
				map, ex, abut_ex);

			/* Shift the start of abut_ex by 'map_len' blocks */
			abut_ex->ee_block = cpu_to_le32(next_lblk - map_len);
			ext4_ext_store_pblock(abut_ex, next_pblk - map_len);
			ex->ee_len = cpu_to_le16(ee_len - map_len);
			ext4_ext_mark_unwritten(ex); /* Restore the flag */

			/* Extend abut_ex by 'map_len' blocks */
			abut_ex->ee_len = cpu_to_le16(next_len + map_len);

			/* Result: number of initialized blocks past m_lblk */
			*allocated = map_len;//申请到map_len个block
		}
	}
	if (*allocated) {
		/* Mark the block containing both extents as dirty */
		err = ext4_ext_dirty(handle, inode, path + depth);

		/* Update path to point to the right extent */
		path[depth].p_ext = abut_ex;//指向将要被合并的extent
		if (err)
			goto errout;
		goto out;
	} else
		*allocated = ee_len - (map->m_lblk - ee_block);//上边两个合并条件不成立，能申请到的block不会跨越当前extent

	WARN_ON(map->m_lblk < ee_block);
	/*
	 * It is safe to convert extent to initialized via explicit
	 * zeroout only if extent is fully inside i_size or new_size.
	 */
	split_flag |= ee_block + ee_len <= eof_block ? EXT4_EXT_MAY_ZEROOUT : 0;

	if (EXT4_EXT_MAY_ZEROOUT & split_flag)
		max_zeroout = sbi->s_extent_max_zeroout_kb >>
			(inode->i_sb->s_blocksize_bits - 10);

	/*
	 * five cases:
	 * 1. split the extent into three extents.
	 * 2. split the extent into two extents, zeroout the head of the first
	 *    extent.
	 * 3. split the extent into two extents, zeroout the tail of the second
	 *    extent.
	 * 4. split the extent into two extents with out zeroout.
	 * 5. no splitting needed, just possibly zeroout the head and / or the
	 *    tail of the extent.
	 */
	split_map.m_lblk = map->m_lblk;
	split_map.m_len = map->m_len;

	if (max_zeroout && (*allocated > split_map.m_len)) {//申请到的block大于本次需要的block
		if (*allocated <= max_zeroout) {//申请到的block小于最大清零block数
			/* case 3 or 5 *///尾部的extent需要清零
			zero_ex1.ee_block =
				 cpu_to_le32(split_map.m_lblk +
					     split_map.m_len);
			zero_ex1.ee_len =
				cpu_to_le16(*allocated - split_map.m_len);
			ext4_ext_store_pblock(&zero_ex1,
				ext4_ext_pblock(ex) + split_map.m_lblk +
				split_map.m_len - ee_block);
			err = ext4_ext_zeroout(inode, &zero_ex1);
			if (err)
				goto fallback;
			split_map.m_len = *allocated;
		}
		if (split_map.m_lblk - ee_block + split_map.m_len <
								max_zeroout) {
			/* case 2 or 5 *///头部的extent需要清零
			if (split_map.m_lblk != ee_block) {//需求起始逻辑块大于找到的extern起始逻辑块
				zero_ex2.ee_block = ex->ee_block;
				zero_ex2.ee_len = cpu_to_le16(split_map.m_lblk -
							ee_block);
				ext4_ext_store_pblock(&zero_ex2,
						      ext4_ext_pblock(ex));
				err = ext4_ext_zeroout(inode, &zero_ex2);
				if (err)
					goto fallback;
			}

			split_map.m_len += split_map.m_lblk - ee_block;//split_map的长度加上ee_block到map->m_lblk的部分
			split_map.m_lblk = ee_block;//split_map的起始block等于找到的externt起始逻辑块
			*allocated = map->m_len;//申请到的block数量赋值为需要的block数
		}
	}

fallback:
	path = ext4_split_extent(handle, inode, path, &split_map, split_flag,
				 flags, NULL);
	if (IS_ERR(path))
		return path;
out:
	/* If we have gotten a failure, don't zero out status tree */
	ext4_zeroout_es(inode, &zero_ex1);
	ext4_zeroout_es(inode, &zero_ex2);
	return path;

errout:
	ext4_free_ext_path(path);
	return ERR_PTR(err);
}


/*
1 ：map->m_lblk +map->m_len 小于ee_block + ee_len时的分割
如果 map->m_lblk +map->m_len 小于ee_block + ee_len，
即map的结束逻辑块地址小于ex的结束逻辑块地址。
则把ex的逻辑块范围分割成3段ee_block~map->m_lblk 和 
map->m_lblk~(map->m_lblk +map->m_len) 和 (map->m_lblk +map->m_len)~(ee_block + ee_len)。
这种情况，就能保证本次要求映射的map->m_len个逻辑块都能完成映射，
即allocated =map->m_len。具体细节是:

a：if (map->m_lblk + map->m_len < ee_block + ee_len)成立，
split_flag1 |= EXT4_EXT_MARK_UNINIT1|EXT4_EXT_MARK_UNINIT2,
然后执行ext4_split_extent_at()以map->m_lblk + map->m_len这个逻辑块地址为分割点，
把path[depth].p_ext指向的ext4_extent结构(即ex)的逻辑块范围ee_block~(ee_block+ee_len)
分割成ee_block~(map->m_lblk + map->m_len)和(map->m_lblk + map->m_len)~(ee_block+ee_len)这两个ext4_extent。

b：前半段的ext4_extent还是ex，
只是映射的逻辑块个数减少了(ee_block+ee_len)-(map->m_lblk + map->m_len)。
后半段的是个新的ext4_extent。因为split_flag1 |= EXT4_EXT_MARK_UNINIT1|EXT4_EXT_MARK_UNINIT2，
则还要标记这两个ext4_extent结构"都是未初始化状态"。
然后把后半段 (map->m_lblk + map->m_len)~(ee_block+ee_len)
对应的ext4_extent结构添加到ext4 extent B+树。回到ext4_split_extent()函数，
ext4_ext_find_extent(inode, map->m_lblk, path)后path[depth].p_ext大概率还是老的ex。

c： if (map->m_lblk >= ee_block)肯定成立，
里边的if (uninitialized)成立，if (uninitialized)里边的
split_flag1 |= EXT4_EXT_MARK_UNINIT1，可能不会加上EXT4_EXT_MARK_UNINIT2标记。
因为split_flag1 |= split_flag & (EXT4_EXT_MAY_ZEROOUT |EXT4_EXT_MARK_UNINIT2)，
接着再次执行ext4_split_extent_at(),以map->m_lblk这个逻辑块地址为分割点，
把path[depth].p_ext指向的ext4_extent结构(即ex)的逻辑块范围ee_block~(ee_block+ee_len)
分割成ee_block~map->m_lblk和map->m_lblk~(ee_block+ee_len)两个ext4_extent结构。

前半段的ext4_extent结构还是ex，
但是逻辑块数减少了(ee_block+ee_len)-map->m_lblk个。
因为此时split_flag1有EXT4_EXT_MARK_UNINIT1标记，
可能没有EXT4_EXT_MARK_UNINIT2标记，则再对ex加上"未初始化状态"，
后半段的ext4_extent可能会被去掉"未初始化状态"，
因为split_flag1可能没有EXT4_EXT_MARK_UNINIT2标记。接着，
把后半段的ext4_extent结构添加到ext4 extent B+树。这里有个特例，
就是 if (map->m_lblk >= ee_block)里的map->m_lblk == ee_block，
即map的要映射的起始逻辑块地址等于ex的起始逻辑块地址，
则执行ext4_split_extent_at()函数时，不会再分割ex，里边if (split == ee_block)成立，
会执行ext4_ext_mark_initialized(ex)标记ex是"初始化状态"，ex终于转正了。

2： map->m_lblk +map->m_len 大于等于ee_block + ee_len时的分割
如果 map->m_lblk +map->m_len 大于等于ee_block + ee_len，
即map的结束逻辑块地址大于ex的结束逻辑块地址。
则把ex的逻辑块范围分割成2段ee_block~map->m_lblk 和 map->m_lblk~(ee_block + ee_len)，
这种情况，不能保证本次要求映射的map->m_len个逻辑块都完成映射。
只能映射 (ee_block + ee_len) - map->m_lblk个逻辑块，
即allocated =(ee_block + ee_len) - map->m_lblk。

 * ext4_split_extent() splits an extent and mark extent which is covered
 * by @map as split_flags indicates
 *
 * It may result in splitting the extent into multiple extents (up to three)
 * There are three possibilities:
 *   a> There is no split required
 *   b> Splits in two extents: Split is happening at either end of the extent
 *   c> Splits in three extents: Somone is splitting in middle of the extent
 *
 */
static struct ext4_ext_path *ext4_split_extent(handle_t *handle,
					       struct inode *inode,
					       struct ext4_ext_path *path,
					       struct ext4_map_blocks *map,
					       int split_flag, int flags,
					       unsigned int *allocated)
{
	ext4_lblk_t ee_block;
	struct ext4_extent *ex;
	unsigned int ee_len, depth;
	int unwritten;
	int split_flag1, flags1;

	depth = ext_depth(inode);
	ex = path[depth].p_ext;
	ee_block = le32_to_cpu(ex->ee_block);
	ee_len = ext4_ext_get_actual_len(ex);
	unwritten = ext4_ext_is_unwritten(ex);

	if (map->m_lblk + map->m_len < ee_block + ee_len) {//将尾部分割出来
		split_flag1 = split_flag & EXT4_EXT_MAY_ZEROOUT;
		flags1 = flags | EXT4_GET_BLOCKS_PRE_IO;
		if (unwritten)
			split_flag1 |= EXT4_EXT_MARK_UNWRIT1 |
				       EXT4_EXT_MARK_UNWRIT2;
		if (split_flag & EXT4_EXT_DATA_VALID2)
			split_flag1 |= EXT4_EXT_DATA_VALID1;
		path = ext4_split_extent_at(handle, inode, path,
				map->m_lblk + map->m_len, split_flag1, flags1);
		if (IS_ERR(path))
			return path;
		/*
		 * Update path is required because previous ext4_split_extent_at
		 * may result in split of original leaf or extent zeroout.
		 需要更新路径，因为先前的ext4_split_extent_at操作可能导致原始叶节点或区段清零的分割。
		 */
		path = ext4_find_extent(inode, map->m_lblk, path, flags);
		if (IS_ERR(path))
			return path;
		depth = ext_depth(inode);
		ex = path[depth].p_ext;
		if (!ex) {
			EXT4_ERROR_INODE(inode, "unexpected hole at %lu",
					(unsigned long) map->m_lblk);
			ext4_free_ext_path(path);
			return ERR_PTR(-EFSCORRUPTED);
		}
		unwritten = ext4_ext_is_unwritten(ex);
	}

	if (map->m_lblk >= ee_block) {//将头部分割出来
		split_flag1 = split_flag & EXT4_EXT_DATA_VALID2;
		if (unwritten) {
			split_flag1 |= EXT4_EXT_MARK_UNWRIT1;
			split_flag1 |= split_flag & (EXT4_EXT_MAY_ZEROOUT |
						     EXT4_EXT_MARK_UNWRIT2);
		}
		path = ext4_split_extent_at(handle, inode, path,
				map->m_lblk, split_flag1, flags);
		if (IS_ERR(path))
			return path;
	}

	if (allocated) {
		if (map->m_lblk + map->m_len > ee_block + ee_len)
			*allocated = ee_len - (map->m_lblk - ee_block);
		else
			*allocated = map->m_len;
	}
	ext4_ext_show_leaf(inode, path);
	return path;
}


/*
 * ext4_split_extent_at() splits an extent at given block.
 *
 * @handle: the journal handle
 * @inode: the file inode
 * @path: the path to the extent
 * @split: the logical block where the extent is splitted.
 * @split_flags: indicates if the extent could be zeroout if split fails, and
 *		 the states(init or unwritten) of new extents.
 * @flags: flags used to insert new extent to extent tree.
 *
 *
 * Splits extent [a, b] into two extents [a, @split) and [@split, b], states
 * of which are determined by split_flag.
 *
 * There are two cases:
 *  a> the extent are splitted into two extent.
 *  b> split is not needed, and just mark the extent.
 *
 * Return an extent path pointer on success, or an error pointer on failure.
 */
static struct ext4_ext_path *ext4_split_extent_at(handle_t *handle,
						  struct inode *inode,
						  struct ext4_ext_path *path,
						  ext4_lblk_t split,
						  int split_flag, int flags)
{
	ext4_fsblk_t newblock;
	ext4_lblk_t ee_block;
	struct ext4_extent *ex, newex, orig_ex, zero_ex;
	struct ext4_extent *ex2 = NULL;
	unsigned int ee_len, depth;
	int err = 0;

	BUG_ON((split_flag & (EXT4_EXT_DATA_VALID1 | EXT4_EXT_DATA_VALID2)) ==
	       (EXT4_EXT_DATA_VALID1 | EXT4_EXT_DATA_VALID2));

	ext_debug(inode, "logical block %llu\n", (unsigned long long)split);

	ext4_ext_show_leaf(inode, path);

	depth = ext_depth(inode);
	ex = path[depth].p_ext;
	ee_block = le32_to_cpu(ex->ee_block);
	ee_len = ext4_ext_get_actual_len(ex);
	newblock = split - ee_block + ext4_ext_pblock(ex);//分割点的物理块号

	BUG_ON(split < ee_block || split >= (ee_block + ee_len));
	BUG_ON(!ext4_ext_is_unwritten(ex) &&
	       split_flag & (EXT4_EXT_MAY_ZEROOUT |
			     EXT4_EXT_MARK_UNWRIT1 |
			     EXT4_EXT_MARK_UNWRIT2));

	err = ext4_ext_get_access(handle, inode, path + depth);
	if (err)
		goto out;

	if (split == ee_block) {//不需要分割
		/*
		 * case b: block @split is the block that the extent begins with
		 * then we just change the state of the extent, and splitting
		 * is not needed.
		 */
		if (split_flag & EXT4_EXT_MARK_UNWRIT2)
			ext4_ext_mark_unwritten(ex);
		else
			ext4_ext_mark_initialized(ex);

		if (!(flags & EXT4_GET_BLOCKS_PRE_IO))
			ext4_ext_try_to_merge(handle, inode, path, ex); //尝试把ex前后的ext4_extent结构的逻辑块和物理块地址合并到ex
		/*ext4_extent映射的逻辑块范围可能发生变化了，标记对应的物理块映射的bh或者文件inode脏*/
		err = ext4_ext_dirty(handle, inode, path + path->p_depth);
		goto out;
	}

	/* case a 需要分割*/
	memcpy(&orig_ex, ex, sizeof(orig_ex)); //orig_ex先保存ex原有数据
	/*重点，标记ex->ee_len为映射的block数，这样ex就是被标记初始化状态了，
	因为ex->ee_len只要不是没被标记EXT_INIT_MAX_LEN，就是初始化状态，
	但是一旦下边执行ext4_ext_mark_uninitialized(ex)，ex又成未初始化状态了*/
	ex->ee_len = cpu_to_le16(split - ee_block);
	if (split_flag & EXT4_EXT_MARK_UNWRIT1)
		ext4_ext_mark_unwritten(ex);

	/*
	 * path may lead to new leaf, not to original leaf any more
	 * after ext4_ext_insert_extent() returns,
	 */
	err = ext4_ext_dirty(handle, inode, path + depth);
	if (err)
		goto fix_extent_len;

	ex2 = &newex;
	ex2->ee_block = cpu_to_le32(split);
	ex2->ee_len   = cpu_to_le16(ee_len - (split - ee_block));
	ext4_ext_store_pblock(ex2, newblock);
	if (split_flag & EXT4_EXT_MARK_UNWRIT2)
		ext4_ext_mark_unwritten(ex2);
	//把ex分割的后半段ext4_extent结构即ex2添加到ext4 extent B+树
	path = ext4_ext_insert_extent(handle, inode, path, &newex, flags);
	if (!IS_ERR(path))
		goto out;

	err = PTR_ERR(path);
	if (err != -ENOSPC && err != -EDQUOT && err != -ENOMEM)
		return path;

	/*
	 * Get a new path to try to zeroout or fix the extent length.
	 * Using EXT4_EX_NOFAIL guarantees that ext4_find_extent()
	 * will not return -ENOMEM, otherwise -ENOMEM will cause a
	 * retry in do_writepages(), and a WARN_ON may be triggered
	 * in ext4_da_update_reserve_space() due to an incorrect
	 * ee_len causing the i_reserved_data_blocks exception.
	 */
	path = ext4_find_extent(inode, ee_block, NULL, flags | EXT4_EX_NOFAIL);
	if (IS_ERR(path)) {
		EXT4_ERROR_INODE(inode, "Failed split extent on %u, err %ld",
				 split, PTR_ERR(path));
		return path;
	}
	depth = ext_depth(inode);
	ex = path[depth].p_ext;

	if (EXT4_EXT_MAY_ZEROOUT & split_flag) {
		if (split_flag & (EXT4_EXT_DATA_VALID1|EXT4_EXT_DATA_VALID2)) {
			if (split_flag & EXT4_EXT_DATA_VALID1) {
				err = ext4_ext_zeroout(inode, ex2);
				zero_ex.ee_block = ex2->ee_block;
				zero_ex.ee_len = cpu_to_le16(
						ext4_ext_get_actual_len(ex2));
				ext4_ext_store_pblock(&zero_ex,
						      ext4_ext_pblock(ex2));
			} else {
				err = ext4_ext_zeroout(inode, ex);
				zero_ex.ee_block = ex->ee_block;
				zero_ex.ee_len = cpu_to_le16(
						ext4_ext_get_actual_len(ex));
				ext4_ext_store_pblock(&zero_ex,
						      ext4_ext_pblock(ex));
			}
		} else {
			err = ext4_ext_zeroout(inode, &orig_ex);
			zero_ex.ee_block = orig_ex.ee_block;
			zero_ex.ee_len = cpu_to_le16(
						ext4_ext_get_actual_len(&orig_ex));
			ext4_ext_store_pblock(&zero_ex,
					      ext4_ext_pblock(&orig_ex));
		}

		if (!err) {
			/* update the extent length and mark as initialized */
			ex->ee_len = cpu_to_le16(ee_len);
			ext4_ext_try_to_merge(handle, inode, path, ex);
			err = ext4_ext_dirty(handle, inode, path + path->p_depth);
			if (!err)
				/* update extent status tree */
				ext4_zeroout_es(inode, &zero_ex);
			/* If we failed at this point, we don't know in which
			 * state the extent tree exactly is so don't try to fix
			 * length of the original extent as it may do even more
			 * damage.
			 */
			goto out;
		}
	}

fix_extent_len:
	ex->ee_len = orig_ex.ee_len;
	/*
	 * Ignore ext4_ext_dirty return value since we are already in error path
	 * and err is a non-zero error code.
	 */
	ext4_ext_dirty(handle, inode, path + path->p_depth);
out:
	if (err) {
		ext4_free_ext_path(path);
		path = ERR_PTR(err);
	}
	ext4_ext_show_leaf(inode, path);
	return path;
}




/*
 * ext4_ext_insert_extent:
 * tries to merge requested extent into the existing extent or
 * inserts requested extent as new one into the tree,
 * creating new leaf in the no-space case.
 */
struct ext4_ext_path *
ext4_ext_insert_extent(handle_t *handle, struct inode *inode,
		       struct ext4_ext_path *path,
		       struct ext4_extent *newext, int gb_flags)
{
	struct ext4_extent_header *eh;
	struct ext4_extent *ex, *fex;
	struct ext4_extent *nearex; /* nearest extent */
	int depth, len, err = 0;
	ext4_lblk_t next;
	int mb_flags = 0, unwritten;

	if (gb_flags & EXT4_GET_BLOCKS_DELALLOC_RESERVE)
		mb_flags |= EXT4_MB_DELALLOC_RESERVED;
	if (unlikely(ext4_ext_get_actual_len(newext) == 0)) {
		EXT4_ERROR_INODE(inode, "ext4_ext_get_actual_len(newext) == 0");
		err = -EFSCORRUPTED;
		goto errout;
	}
	depth = ext_depth(inode);
	ex = path[depth].p_ext;
	eh = path[depth].p_hdr;
	if (unlikely(path[depth].p_hdr == NULL)) {
		EXT4_ERROR_INODE(inode, "path[%d].p_hdr == NULL", depth);
		err = -EFSCORRUPTED;
		goto errout;
	}

   /*下判断newex跟ex、ex前边的ext4_extent结构、ex后边的ext4_extent结构逻辑块地址范围是否紧挨着，
   是的话才能将二者合并。但能合并还要符合一个苛刻条件:
   参与合并的两个ext4_extent必须是initialized状态，否则无法合并*/
	/* try to insert block into found extent and return */
	if (ex && !(gb_flags & EXT4_GET_BLOCKS_PRE_IO)) {

		/*
		 * Try to see whether we should rather test the extent on
		 * right from ex, or from the left of ex. This is because
		 * ext4_find_extent() can return either extent on the
		 * left, or on the right from the searched position. This
		 * will make merging more effective.
		 */
		if (ex < EXT_LAST_EXTENT(eh) &&
		    (le32_to_cpu(ex->ee_block) +
		    ext4_ext_get_actual_len(ex) <
		    le32_to_cpu(newext->ee_block))) {
			ex += 1;
			goto prepend;
		} else if ((ex > EXT_FIRST_EXTENT(eh)) &&
			   (le32_to_cpu(newext->ee_block) +
			   ext4_ext_get_actual_len(newext) <
			   le32_to_cpu(ex->ee_block)))
			ex -= 1;

		/* Try to append newex to the ex 与前边extern合并*/
		if (ext4_can_extents_be_merged(inode, ex, newext)) {
			ext_debug(inode, "append [%d]%d block to %u:[%d]%d"
				  "(from %llu)\n",
				  ext4_ext_is_unwritten(newext),
				  ext4_ext_get_actual_len(newext),
				  le32_to_cpu(ex->ee_block),
				  ext4_ext_is_unwritten(ex),
				  ext4_ext_get_actual_len(ex),
				  ext4_ext_pblock(ex));
			err = ext4_ext_get_access(handle, inode,
						  path + depth);
			if (err)
				goto errout;
			unwritten = ext4_ext_is_unwritten(ex);
			ex->ee_len = cpu_to_le16(ext4_ext_get_actual_len(ex)
					+ ext4_ext_get_actual_len(newext));
			if (unwritten)
				ext4_ext_mark_unwritten(ex);
			nearex = ex;
			goto merge;
		}

prepend:
		/* Try to prepend newex to the ex 与后边extern合并*/
		if (ext4_can_extents_be_merged(inode, newext, ex)) {
			ext_debug(inode, "prepend %u[%d]%d block to %u:[%d]%d"
				  "(from %llu)\n",
				  le32_to_cpu(newext->ee_block),
				  ext4_ext_is_unwritten(newext),
				  ext4_ext_get_actual_len(newext),
				  le32_to_cpu(ex->ee_block),
				  ext4_ext_is_unwritten(ex),
				  ext4_ext_get_actual_len(ex),
				  ext4_ext_pblock(ex));
			err = ext4_ext_get_access(handle, inode,
						  path + depth);
			if (err)
				goto errout;

			unwritten = ext4_ext_is_unwritten(ex);
			ex->ee_block = newext->ee_block;
			ext4_ext_store_pblock(ex, ext4_ext_pblock(newext));
			ex->ee_len = cpu_to_le16(ext4_ext_get_actual_len(ex)
					+ ext4_ext_get_actual_len(newext));
			if (unwritten)
				ext4_ext_mark_unwritten(ex);
			nearex = ex;
			goto merge;
		}
	}

	depth = ext_depth(inode);
	eh = path[depth].p_hdr;
	/*eh->eh_max是ext4_extent B+树叶子节点最大ext4_extent个数，
	这是测试path[depth].p_hdr所在叶子节点的ext4_extent结构是否爆满，
	没有爆满才会跳到has_space分支*/
	if (le16_to_cpu(eh->eh_entries) < le16_to_cpu(eh->eh_max))
		goto has_space;

	/* probably next leaf has space for us? */
	fex = EXT_LAST_EXTENT(eh);
	next = EXT_MAX_BLOCKS;
	  /*如果要插入的newext起始逻辑块地址大于ext4 extent B+树叶子节点最后一个ext4_extent结构的，
	  说明当前的叶子节点逻辑块地址范围太小了，无法容纳一个完整的extent*/
	if (le32_to_cpu(newext->ee_block) > le32_to_cpu(fex->ee_block))
		next = ext4_ext_next_leaf_block(path);//在path上寻找下一个叶子extent_headr，返回逻辑块号
	if (next != EXT_MAX_BLOCKS) {
		struct ext4_ext_path *npath;

		ext_debug(inode, "next leaf block - %u\n", next);
		npath = ext4_find_extent(inode, next, NULL, gb_flags);//根据逻辑块号查找path
		if (IS_ERR(npath)) {
			err = PTR_ERR(npath);
			goto errout;
		}
		BUG_ON(npath->p_depth != path->p_depth);
		eh = npath[depth].p_hdr;
		if (le16_to_cpu(eh->eh_entries) < le16_to_cpu(eh->eh_max)) {//看下一个叶子extent_headr是否已满
			ext_debug(inode, "next leaf isn't full(%d)\n",
				  le16_to_cpu(eh->eh_entries));
			ext4_free_ext_path(path);
			path = npath;
			goto has_space;
		}
		ext_debug(inode, "next leaf has no free space(%d,%d)\n",
			  le16_to_cpu(eh->eh_entries), le16_to_cpu(eh->eh_max));
		ext4_free_ext_path(npath);
	}

	/*
	 * There is no free space in the found leaf.
	 * We're gonna add a new leaf in the tree.找到的两个叶子节点满了，新建一个
	 */
	if (gb_flags & EXT4_GET_BLOCKS_METADATA_NOFAIL)
		mb_flags |= EXT4_MB_USE_RESERVED;
	path = ext4_ext_create_new_leaf(handle, inode, mb_flags, gb_flags,
					path, newext);
	if (IS_ERR(path))
		return path;
	depth = ext_depth(inode);
	eh = path[depth].p_hdr;
/*到这里，最新的path[depth].p_ext所在叶子节点肯定有空闲的ext4_extent，
即空闲位置可以存放newext这个ext4_extent结构，则直接把newext插入到叶子节点某个合适的ext4_extent位置处*/
has_space:
/*nearex指向起始逻辑块地址最接近 newext->ee_block这个起始逻辑块地址的ext4_extent，
newext是本次要ext4 extent b+树的ext4_extent*/
	nearex = path[depth].p_ext;

	err = ext4_ext_get_access(handle, inode, path + depth);
	if (err)
		goto errout;

	if (!nearex) {//path[depth].p_ext所在叶子节点还没有使用过一个ext4_extent结构
		/* there is no extent in this leaf, create first one */
		ext_debug(inode, "first extent in the leaf: %u:%llu:[%d]%d\n",
				le32_to_cpu(newext->ee_block),
				ext4_ext_pblock(newext),
				ext4_ext_is_unwritten(newext),
				ext4_ext_get_actual_len(newext));
		//nearex指向叶子节点第一个ext4_extent结构，newext就插入到这里
		nearex = EXT_FIRST_EXTENT(eh);
	} else {
		if (le32_to_cpu(newext->ee_block)
			   > le32_to_cpu(nearex->ee_block)) {
			/* Insert after */
			ext_debug(inode, "insert %u:%llu:[%d]%d before: "
					"nearest %p\n",
					le32_to_cpu(newext->ee_block),
					ext4_ext_pblock(newext),
					ext4_ext_is_unwritten(newext),
					ext4_ext_get_actual_len(newext),
					nearex);
			nearex++;/*newext的起始逻辑块地址大于nearex的起始逻辑块地址，
			nearex++指向后边的一个ext4_extent结构*/
		} else {
			/* Insert before */
			BUG_ON(newext->ee_block == nearex->ee_block);
			ext_debug(inode, "insert %u:%llu:[%d]%d after: "
					"nearest %p\n",
					le32_to_cpu(newext->ee_block),
					ext4_ext_pblock(newext),
					ext4_ext_is_unwritten(newext),
					ext4_ext_get_actual_len(newext),
					nearex);
		}
		/*这是计算nearex这个ext4_extent结构到叶子节点最后一个
		ext4_extent结构(有效的)之间的ext4_extent结构个数。注意"有效的"3个字，
		比如叶子节点只使用了一个ext4_extent，则EXT_LAST_EXTENT(eh)是叶子节点第一个ext4_extent结构。*/
		len = EXT_LAST_EXTENT(eh) - nearex + 1;
		if (len > 0) {
			ext_debug(inode, "insert %u:%llu:[%d]%d: "
					"move %d extents from 0x%p to 0x%p\n",
					le32_to_cpu(newext->ee_block),
					ext4_ext_pblock(newext),
					ext4_ext_is_unwritten(newext),
					ext4_ext_get_actual_len(newext),
					len, nearex, nearex + 1);
			/*这是把nearex这个ext4_extent结构 ~ 最后一个ext4_extent结构(有效的)之间的
			所有ext4_extent结构的数据整体向后移动一个ext4_extent结构大小，
			腾出原来nearex这个ext4_extent结构的空间，下边正是把newext插入到这里，
			这样终于把newex插入ext4_extent B+树了*/
			memmove(nearex + 1, nearex,
				len * sizeof(struct ext4_extent));
		}
	}
	/*下边是把newext的起始逻辑块地址、起始物理块起始地址、逻辑块地址映射的物理块个数等信息赋值给nearex，
	相当于把newext添加到叶子节点原来nearex的位置。然后叶子节点ext4_extent个数加1。
	path[depth].p_ext指向newext*/
	le16_add_cpu(&eh->eh_entries, 1);
	path[depth].p_ext = nearex;
	nearex->ee_block = newext->ee_block;
	ext4_ext_store_pblock(nearex, ext4_ext_pblock(newext));
	nearex->ee_len = newext->ee_len;

merge:
	/*尝试把ex后的ext4_extent结构的逻辑块和物理块地址合并到ex。
	并且，如果ext4_extent B+树深度是1，并且叶子结点有很少的ext4_extent结构，
	则尝试把叶子结点的ext4_extent结构移动到root节点，节省空间而已*/
	/* try to merge extents */
	if (!(gb_flags & EXT4_GET_BLOCKS_PRE_IO))
		ext4_ext_try_to_merge(handle, inode, path, nearex);

	/* time to correct all indexes above */
	err = ext4_ext_correct_indexes(handle, inode, path);
	if (err)
		goto errout;

	err = ext4_ext_dirty(handle, inode, path + path->p_depth);
	if (err)
		goto errout;

	return path;

errout:
	ext4_free_ext_path(path);
	return ERR_PTR(err);
}


/*
 * ext4_ext_create_new_leaf:
 * finds empty index and adds new leaf.
 * if no free index is found, then it requests in-depth growing.
 */
static struct ext4_ext_path *
ext4_ext_create_new_leaf(handle_t *handle, struct inode *inode,
			 unsigned int mb_flags, unsigned int gb_flags,
			 struct ext4_ext_path *path,
			 struct ext4_extent *newext)
{
	struct ext4_ext_path *curp;
	int depth, i, err = 0;
	ext4_lblk_t ee_block = le32_to_cpu(newext->ee_block);

repeat:
	i = depth = ext_depth(inode);

	/* walk up to the tree and look for free index entry */
	/*该while是从ext4 extent B+树叶子节点开始，自下而上，向上一直到索引节点，
	看索引节点或者叶子节点的ext4_extent_idx或ext4_extent个数是否大于最大限制eh_max，
	超出限制EXT_HAS_FREE_INDEX(curp)返回0，否则返回1.从该while循环退出时，
	有两种可能，1:curp非NULL，curp指向的索引节点或叶子节点有空闲ext4_extent_idx或ext4_extent可使用，
	2:i是0，ext4 extent B+树索引节点或叶子节点ext4_extent_idx或ext4_extent个数爆满，
	没有空闲ext4_extent_idx或ext4_extent可使用*/
	curp = path + depth;
	while (i > 0 && !EXT_HAS_FREE_INDEX(curp)) {
		i--;
		curp--;
	}

	/* we use already allocated block for index block,
	 * so subsequent data blocks should be contiguous */
	/*ext4 extent B+树索引节点或者叶子节点有空闲ext4_extent_idx或ext4_extent可使用。
	此时的i表示ext4 extent B+树哪一层有空闲ext4_extent_idx或ext4_extent可使用。
	newext是要插入ext4_extent B+树的ext4_extent，
	插入ext4_extent B+树的第i层的叶子节点或者第i层索引节点下边的叶子节点*/
	if (EXT_HAS_FREE_INDEX(curp)) {
		/*凡是执行到ext4_ext_split()函数，说明ext4 extent B+树中与newext->ee_block有关的叶子节点ext4_extent结构爆满了。
		  但有空间可存放索引节点，于是从ext4 extent B+树at那一层索引节点到叶子节点，针对每一层都创建新的索引节点，
		  也创建叶子节点。还会尝试把索引节点path[at~depth].p_hdr指向的ext4_extent_idx结构的后边的ext4_extent_idx结构
		  和path[depth].p_ext指向的ext4_extent结构后边的ext4_extent结构，移动到新创建的叶子节点和索引节点。
		  这样就能保证ext4 extent B+树中，与newext->ee_block有关的叶子节点有空闲entry，就能存放newext这个ext4_extent结构了。*/
		  /* if we found index with free entry, then use that
		 * entry: create all needed subtree and add new leaf */
		err = ext4_ext_split(handle, inode, mb_flags, path, newext, i);
		if (err)
			goto errout;

		/* refill path */
		/*ext4_ext_split()对ext4_extent B+树做了重建和分割，
		这里再次在ext4_extent B+树查找起始逻辑块地址接近newext->ee_block的索引节点和叶子结点*/
		path = ext4_find_extent(inode, ee_block, path, gb_flags);
		return path;
	}

	/* tree is full, time to grow in depth */
	/*到这个分支，ext4 extent B+树索引节点的ext4_extent_idx和
	叶子节点的ext4_extent个数全爆满，没有空闲ext4_extent_idx或ext4_extent可使用，
	就是说ext4 extent B+树全爆满了，
	只能增加执行ext4_ext_grow_indepth()增加ext4 extent B+树叶子节点或者索引节点了*/
	err = ext4_ext_grow_indepth(handle, inode, mb_flags);
	if (err)
		goto errout;

	/* refill path */
	path = ext4_find_extent(inode, ee_block, path, gb_flags);
	if (IS_ERR(path))
		return path;

	/*
	 * only first (depth 0 -> 1) produces free space;
	 * in all other cases we have to split the grown tree
	 */
	depth = ext_depth(inode);
	if (path[depth].p_hdr->eh_entries == path[depth].p_hdr->eh_max) {
		/* now we need to split */
		goto repeat;
	}

	return path;

errout:
	ext4_free_ext_path(path);
	return ERR_PTR(err);
}


/*问题：这里为什么要移动叶子节点和所用节点？？？
 * ext4_ext_split:
 * inserts new subtree into the path, using free index entry
 * at depth @at:
 * - allocates all needed blocks (new leaf and all intermediate index blocks)
 * - makes decision where to split
 * - moves remaining extents and index entries (right to the split point)
 *   into the newly allocated blocks
 * - initializes subtree
 */
static int ext4_ext_split(handle_t *handle, struct inode *inode,
			  unsigned int flags,
			  struct ext4_ext_path *path,
			  struct ext4_extent *newext, int at)
{
	struct buffer_head *bh = NULL;
	int depth = ext_depth(inode);
	struct ext4_extent_header *neh;
	struct ext4_extent_idx *fidx;
	int i = at, k, m, a; /*newext是要插入ext4_extent B+树的ext4_extent，在ext4_extent B+树的第at层插入newext，第at层的索引节点有空闲entry*/
	ext4_fsblk_t newblock, oldblock;
	__le32 border;
	ext4_fsblk_t *ablocks = NULL; /* array of allocated blocks */
	gfp_t gfp_flags = GFP_NOFS;
	int err = 0;
	size_t ext_size = 0;

	if (flags & EXT4_EX_NOFAIL)
		gfp_flags |= __GFP_NOFAIL;

	/* make decision: where to split? */
	/* FIXME: now decision is simplest: at current extent */

	/* if current leaf will be split, then we should use
	 * border from split point */
	if (unlikely(path[depth].p_ext > EXT_MAX_EXTENT(path[depth].p_hdr))) {
		EXT4_ERROR_INODE(inode, "p_ext > EXT_MAX_EXTENT!");
		return -EFSCORRUPTED;
	}
	/*path[depth].p_ext是ext4 extent B+树叶子节点中，逻辑块地址最接近map->m_lblk这个起始逻辑块地址的ext4_extent*/
	if (path[depth].p_ext != EXT_MAX_EXTENT(path[depth].p_hdr)) {
		 /*path[depth].p_ext不是叶子节点最后一个ext4_extent结构，
		 那以它后边的ext4_extent结构path[depth].p_ext[1]的起始逻辑块地址作为分割点border*/
		border = path[depth].p_ext[1].ee_block;
		ext_debug(inode, "leaf will be split."
				" next leaf starts at %d\n",
				  le32_to_cpu(border));
	} else {
		/*这里说明path[depth].p_ext指向的是叶子节点最后一个ext4_extent结构*/
		border = newext->ee_block;
		ext_debug(inode, "leaf will be added."
				" next leaf starts at %d\n",
				le32_to_cpu(border));
	}

	/*
	 * If error occurs, then we break processing
	 * and mark filesystem read-only. index won't
	 * be inserted and tree will be in consistent
	 * state. Next mount will repair buffers too.
	 */

	/*
	 * Get array to track all allocated blocks.
	 * We need this to handle errors and free blocks
	 * upon them.
	 */
	ablocks = kcalloc(depth, sizeof(ext4_fsblk_t), gfp_flags);
	if (!ablocks)
		return -ENOMEM;

	/* allocate all needed blocks */
	ext_debug(inode, "allocate %d blocks for indexes/leaf\n", depth - at);
	for (a = 0; a < depth - at; a++) {
		 /*分配(depth - at)个物理块，newext是在ext4 extent B+的第at层插入，
		 从at层到depth层，每层分配一个物理块,for indexes/leaf*/
		newblock = ext4_ext_new_meta_block(handle, inode, path,
						   newext, &err, flags);
		if (newblock == 0)
			goto cleanup;
		ablocks[a] = newblock;
	}

	/* initialize new leaf 最后一层存放的总是叶子节点*/
	newblock = ablocks[--a];
	if (unlikely(newblock == 0)) {
		EXT4_ERROR_INODE(inode, "newblock == 0!");
		err = -EFSCORRUPTED;
		goto cleanup;
	}
	bh = sb_getblk_gfp(inode->i_sb, newblock, __GFP_MOVABLE | GFP_NOFS);
	if (unlikely(!bh)) {
		err = -ENOMEM;
		goto cleanup;
	}
	lock_buffer(bh);

	err = ext4_journal_get_create_access(handle, inode->i_sb, bh,
					     EXT4_JTR_NONE);
	if (err)
		goto cleanup;
	//初始化叶子结点数组的头
	neh = ext_block_hdr(bh);
	neh->eh_entries = 0;
	neh->eh_max = cpu_to_le16(ext4_ext_space_block(inode, 0));
	neh->eh_magic = EXT4_EXT_MAGIC;
	neh->eh_depth = 0;
	neh->eh_generation = 0;

	/*从path[depth].p_ext后边的ext4_extent结构到叶子节点最后一个ext4_extent结构之间，一共有m个ext4_extent结构*/
	/* move remainder of path[depth] to the new leaf */
	if (unlikely(path[depth].p_hdr->eh_entries !=
		     path[depth].p_hdr->eh_max)) {
		EXT4_ERROR_INODE(inode, "eh_entries %d != eh_max %d!",
				 path[depth].p_hdr->eh_entries,
				 path[depth].p_hdr->eh_max);
		err = -EFSCORRUPTED;
		goto cleanup;
	}
	/* start copy from next extent */
	m = EXT_MAX_EXTENT(path[depth].p_hdr) - path[depth].p_ext++;
	ext4_ext_show_move(inode, path, newblock, depth);
	if (m) {
		struct ext4_extent *ex;
		ex = EXT_FIRST_EXTENT(neh);
		//老的叶子节点path[depth].p_ext后的m个ext4_extent结构移动到上边新分配的叶子节点
		memmove(ex, path[depth].p_ext, sizeof(struct ext4_extent) * m);
		le16_add_cpu(&neh->eh_entries, m);
	}

	/* zero out unused area in the extent block */
	ext_size = sizeof(struct ext4_extent_header) +
		sizeof(struct ext4_extent) * le16_to_cpu(neh->eh_entries);
	memset(bh->b_data + ext_size, 0, inode->i_sb->s_blocksize - ext_size);
	ext4_extent_block_csum_set(inode, neh);
	set_buffer_uptodate(bh);
	unlock_buffer(bh);

	err = ext4_handle_dirty_metadata(handle, inode, bh);
	if (err)
		goto cleanup;
	brelse(bh);
	bh = NULL;

	/* correct old leaf */
	if (m) {
		err = ext4_ext_get_access(handle, inode, path + depth);
		if (err)
			goto cleanup;
		le16_add_cpu(&path[depth].p_hdr->eh_entries, -m);
		err = ext4_ext_dirty(handle, inode, path + depth);
		if (err)
			goto cleanup;

	}
	
	/*最后一层的叶子节点操作完成，现在要操作at层到叶子节点层之间的索引节点层*/
	/*ext4_extent B+树at那一层的索引节点到最后一层索引节点之间的层数，就是从at开始有多少层索引节点*/
	/* create intermediate indexes */
	k = depth - at - 1;
	if (unlikely(k < 0)) {
		EXT4_ERROR_INODE(inode, "k %d < 0!", k);
		err = -EFSCORRUPTED;
		goto cleanup;
	}
	if (k)
		ext_debug(inode, "create %d intermediate indices\n", k);
	/* insert new index into current index block */
	/* current depth stored in i var */
	i = depth - 1;
	while (k--) {
		oldblock = newblock;//存放叶子节点的block号
		newblock = ablocks[--a];
		bh = sb_getblk(inode->i_sb, newblock);
		if (unlikely(!bh)) {
			err = -ENOMEM;
			goto cleanup;
		}
		lock_buffer(bh);

		err = ext4_journal_get_create_access(handle, inode->i_sb, bh,
						     EXT4_JTR_NONE);
		if (err)
			goto cleanup;

		neh = ext_block_hdr(bh);
		neh->eh_entries = cpu_to_le16(1);
		neh->eh_magic = EXT4_EXT_MAGIC;
		neh->eh_max = cpu_to_le16(ext4_ext_space_block_idx(inode, 0));
		neh->eh_depth = cpu_to_le16(depth - i);
		neh->eh_generation = 0;
		fidx = EXT_FIRST_INDEX(neh);
		fidx->ei_block = border;
		ext4_idx_store_pblock(fidx, oldblock);//将叶子检点的block号设置到索引节点中

		ext_debug(inode, "int.index at %d (block %llu): %u -> %llu\n",
				i, newblock, le32_to_cpu(border), oldblock);

		/* move remainder of path[i] to the new index block */
		if (unlikely(EXT_MAX_INDEX(path[i].p_hdr) !=
					EXT_LAST_INDEX(path[i].p_hdr))) {
			EXT4_ERROR_INODE(inode,
					 "EXT_MAX_INDEX != EXT_LAST_INDEX ee_block %d!",
					 le32_to_cpu(path[i].p_ext->ee_block));
			err = -EFSCORRUPTED;
			goto cleanup;
		}
		/*计算 path[i].p_hdr这一层索引节点中，
		从path[i].p_idx指向的ext4_extent_idx结构到最后一个ext4_extent_idx结构之间ext4_extent_idx个数*/
		/* start copy indexes */
		m = EXT_MAX_INDEX(path[i].p_hdr) - path[i].p_idx++;
		ext_debug(inode, "cur 0x%p, last 0x%p\n", path[i].p_idx,
				EXT_MAX_INDEX(path[i].p_hdr));
		ext4_ext_show_move(inode, path, newblock, i);
		if (m) {
			/*把path[i].p_idx后边的m个ext4_extent_idx结构
			赋值到newblock这个物理块对应的索引节点开头的第1个ext4_extent_idx的后边，
			即fidx指向的ext4_extent_idx后边。这里是++fid，
			即fidx指向的索引节点的第2个ext4_extent_idx位置处，
			这是向索引节点第2个ext4_extent_idx处及后边复制m个ext4_extent_idx结构*/
			memmove(++fidx, path[i].p_idx,
				sizeof(struct ext4_extent_idx) * m);
			le16_add_cpu(&neh->eh_entries, m);
		}
		/* zero out unused area in the extent block */
		ext_size = sizeof(struct ext4_extent_header) +
		   (sizeof(struct ext4_extent) * le16_to_cpu(neh->eh_entries));
		memset(bh->b_data + ext_size, 0,
			inode->i_sb->s_blocksize - ext_size);
		ext4_extent_block_csum_set(inode, neh);
		set_buffer_uptodate(bh);
		unlock_buffer(bh);

		err = ext4_handle_dirty_metadata(handle, inode, bh);
		if (err)
			goto cleanup;
		brelse(bh);
		bh = NULL;

		/* correct old index */
		if (m) {
			err = ext4_ext_get_access(handle, inode, path + i);
			if (err)
				goto cleanup;
			le16_add_cpu(&path[i].p_hdr->eh_entries, -m);
			err = ext4_ext_dirty(handle, inode, path + i);
			if (err)
				goto cleanup;
		}

		i--;
	}

	/* insert new index */
	/*把新的索引节点的ext4_extent_idx结构(起始逻辑块地址border,
	物理块号newblock)插入到ext4 extent B+树at那一层
	索引节点(path + at)->p_idx指向的ext4_extent_idx结构前后。*/
	err = ext4_ext_insert_index(handle, inode, path + at,
				    le32_to_cpu(border), newblock);

cleanup:
	if (bh) {
		if (buffer_locked(bh))
			unlock_buffer(bh);
		brelse(bh);
	}

	if (err) {
		/* free all allocated blocks in error case */
		for (i = 0; i < depth; i++) {
			if (!ablocks[i])
				continue;
			ext4_free_blocks(handle, inode, NULL, ablocks[i], 1,
					 EXT4_FREE_BLOCKS_METADATA);
		}
	}
	kfree(ablocks);

	return err;
}

/*
1:首先确定ext4 extent B+树的分割点逻辑地址border。
如果path[depth].p_ext不是ext4_extent B+树叶子节点节点最后一个ext4 extent结构，
则分割点逻辑地址border是path[depth].p_ext后边的ext4_extent起始逻辑块地址，
即border=path[depth].p_ext[1].ee_block。否则border是新插入ext4 extent B+树的
ext4_extent的起始逻辑块地址，即newext->ee_block。

2:因为ext4_extent B+树at那一层索引节点有空闲entry，
则针对at~depth(B+树深度)之间的的每一层索引节点和叶子节点都分配新的索引节点和叶子结点，
每个索引节点和叶子结点都占一个block大小(4K)，分别保存N个ext4_extent_idx结构
和N个ext4_extent结构，还有ext4_extent_header。在while (k--)那个循环，
这些新分配的索引节点和叶子节点中，B+树倒数第2层的那个索引节点的第一个ext4_extent_idx的
物理块号成员(ei_leaf_lo和ei_leaf_hi)记录的新分配的保存叶子结点4K数据的物理块号
(代码是ext4_idx_store_pblock(fidx, oldblock))，第一个ext4_extent_idx的起始逻辑块地址是border
(代码是fidx->ei_block = border)。B+树倒数第3层的那个索引节点的第一个ext4_extent_idx的
物理块号成员记录的是保存倒数第2层的索引节点4K数据的物理块号，
这层索引节点第一个ext4_extent_idx的起始逻辑块地址是border
(代码是fidx->ei_block = border)........其他类推。at那一层新分配的索引节点
(物理块号是newblock，起始逻辑块地址border)，执行ext4_ext_insert_index()插入到
ext4_extent B+树at层原有的索引节点(path + at)->p_idx指向的ext4_extent_idx结构前后的
ext4_extent_idx结构位置处。插入过程是:把(path + at)->p_idx指向的索引节点的ext4_extent_idx
结构后的所有ext4_extent_idx结构向后移动一个ext4_extent_idx结构大小，这就在(path + at)->p_idx
指向的索引节点的ext4_extent_idx处腾出了一个空闲的ext4_extent_idx结构大小空间，
新分配的索引节点就是插入到这里。

3:要把ext4_extent B+树原来的at~depth层的 path[i].p_idx~path[depth-1].p_idx指向的ext4_extent_idx
结构后边的所有ext4_extent_idx结构 和 path[depth].p_ext指向的ext4_extent后的所有
ext4_extent结构都对接移动到上边针对ext4_extent B+树at~denth新分配索引节点和叶子节点物理块号映射bh内存。
这是对原有的ext4 extent B+树进行扩容的重点。

上边的解释没有诠释到本质。直击灵魂，
为什么会执行到ext4_ext_insert_extent()->ext4_ext_create_new_leaf()->ext4_ext_split()?有什么意义?
首先，ext4_split_extent_at()函数中，把path[depth].p_ext指向的ext4_extent结构(即ex)的逻辑块范围分割成两段，
两个ext4_extent结构。前边的ext4_extent结构还是ex，只是逻辑块范围减少了。
而后半段ext4_extent结构即newext就要插入插入到到ext4 extent B+树。
到ext4_ext_insert_extent()函数，如果此时ex所在叶子节点的ext4_extent结构爆满了，
即if (le16_to_cpu(eh->eh_entries) < le16_to_cpu(eh->eh_max))不成立，
但是if (le32_to_cpu(newext->ee_block) > le32_to_cpu(fex->ee_block))成立，
即newext的起始逻辑块地址小于ex所在叶子节点的最后一个ext4_extent结构的起始逻辑块地址，
则执行next = ext4_ext_next_leaf_block(path)等代码，回到上层索引节点，
找到起始逻辑块地址更大的索引节点和叶子节点，如果新的叶子节点的ext4_extent结构还是爆满，
那就要执行ext4_ext_create_new_leaf()增大ext4_extent B+树层数了。

来到ext4_ext_create_new_leaf()函数，从最底层的索引节点开始向上搜索，找到有空闲entry的索引节点。
如果找到则执行ext4_ext_split()。如果找不到则执行ext4_ext_grow_indepth()
在ext4_extent B+树root节点增加一层索引节点(或叶子节点)，然后也执行ext4_ext_split()。

当执行到ext4_ext_split()，at一层的ext4_extent B+树有空闲entry，则以从at层到叶子节点那一层，
创建新的索引节点和叶子节点，建立这些新的索引节点和叶子节点彼此的物理块号的联系。
我们假设ext4_ext_split()的if (path[depth].p_ext != EXT_MAX_EXTENT(path[depth].p_hdr))成立，
则这样执行：向新分配的叶子节点复制m个ext4_extent结构时，复制的第一个ext4_extent结构不是path[depth].p_ext，
而是它后边的 path[depth].p_ext[1]这个ext4_extent结构。并且，
下边新创建的索引节点的第一个ext4_extent_idx结构的起始逻辑器块地址都是border，
即path[depth].p_ext[1]的逻辑块地址，也是path[depth].p_ext[1].ee_block。
然后向新传创建的索引节点的第2个ext4_extent_idx结构处及之后复制m个ext4_extent_idx结构。
新传创建的索引节点的第一个ext4_extent_idx的起始逻辑块地址是border，单独使用，作为分割点的ext4_extent_idx结构。
如此，后续执行ext4_ext_find_extent(newext->ee_block)在老的ext4_extent B+树找到的path[depth].p_ext指向的ext4_extent还是老的，
但是path[depth].p_ext后边的m个ext4_extent结构移动到了新分配的叶子节点，
path[depth].p_ext所在叶子节点就有空间了，newext就插入到path[depth].p_ext指向的ext4_extent叶子节点后边。
这段代码在ext4_ext_insert_extent()的has_space 的if (!nearex)........} else{......}的else分支。

如果ext4_ext_split()的if (path[depth].p_ext != EXT_MAX_EXTENT(path[depth].p_hdr))不成立，则这样执行：
不会向新分配的叶子节点复制ext4_extent结构，m是0，因为path[depth].p_ext就是叶子节点最后一个ext4_extent结构，
下边的m = EXT_MAX_EXTENT(path[depth].p_hdr) - path[depth].p_ext++=0。并且，下边新创建的索引节点的第一个ext4_extent_idx结构
的起始逻辑器块地址都是newext->ee_block。这样后续执行ext4_ext_find_extent()在
ext4_extent B+树就能找到起始逻辑块地址是newext->ee_block的层层索引节点了，完美匹配。
那叶子节点呢?这个分支没有向新的叶子节点复制ext4_extent结构，空的，ext4_ext_find_extent()执行后，
path[ppos].depth指向新的叶子节点的头结点，
此时直接令该叶子节点的第一个ext4_extent结构的逻辑块地址是newext->ee_block，完美!
这段代码在ext4_ext_insert_extent()的has_space 的if (!nearex)分支。

注意，这是ext4_extent B+树叶子节点增加增加的第一个ext4_extent结构，
并且第一个ext4_extent结构的起始逻辑块地址与它上边的索引节点的ext4_extent_idx的起始逻辑块地址都是newext->ee_block，
再上层的索引节点的ext4_extent_idx的起始逻辑块地址也是newext->ee_block，直到第at层。

因此，我们看到ext4_ext_split()最核心的作用是：at一层的ext4_extent B+树有空闲entry，
则从at层开始创建新的索引节点和叶子节点，建立这些新的索引节点和叶子节点彼此的物理块号联系。
然后把path[depth].p_ext后边的ext4_extent结构移动到新的叶子节点，
把path[at~depth-1].p_idx这些索引节点后边的ext4_extent_idx结构依次移动到新创建的索引节点。
这样要么老的path[depth].p_ext所在叶子节点有了空闲的ext4_extent entry，
把newex插入到老的path[depth].p_ext所在叶子节点后边即可。或者新创建的at~denth的索引节点

和叶子节点，有大量空闲的entry，这些索引节点的起始逻辑块地址还是newext->ee_block，
则直接把newext插入到新创建的叶子节点第一个ext4_extent结构即可。

最后，对ext4_ext_split简单总结: 凡是执行到ext4_ext_split()函数，
说明ext4 extent B+树中与newext->ee_block有关的叶子节点ext4_extent结构爆满了。
于是从ext4 extent B+树at那一层索引节点到叶子节点，针对每一层都创建新的索引节点，
也创建叶子节点。还会尝试把索引节点path[at~depth].p_hdr指向的ext4_extent_idx
结构的后边的ext4_extent_idx结构和path[depth].p_ext指向的ext4_extent结构后边的ext4_extent结构，
移动到新创建的叶子节点和索引节点。这样可能保证ext4 extent B+树中，与newext->ee_block有关的叶子节点有空闲entry，能存放newext。

*/

/*
 * ext4_ext_insert_index:
 * insert new index [@logical;@ptr] into the block at @curp;
 * check where to insert: before @curp or after @curp
 */
static int ext4_ext_insert_index(handle_t *handle, struct inode *inode,
				 struct ext4_ext_path *curp,
				 int logical, ext4_fsblk_t ptr)
{
	struct ext4_extent_idx *ix;
	int len, err;
	
	/*curp->p_idx是ext4 extent B+树起始逻辑块地址最接近传入的起始逻辑块地址map->m_lblk的ext4_extent_idx结构，
	现在是把新的ext4_extent_idx(起始逻辑块地址是logical,起始物理块号ptr)插入到curp->p_idx指向的ext4_extent_idx结构前后*/
	err = ext4_ext_get_access(handle, inode, curp);
	if (err)
		return err;

	if (unlikely(logical == le32_to_cpu(curp->p_idx->ei_block))) {
		EXT4_ERROR_INODE(inode,
				 "logical %d == ei_block %d!",
				 logical, le32_to_cpu(curp->p_idx->ei_block));
		return -EFSCORRUPTED;
	}

	if (unlikely(le16_to_cpu(curp->p_hdr->eh_entries)
			     >= le16_to_cpu(curp->p_hdr->eh_max))) {
		EXT4_ERROR_INODE(inode,
				 "eh_entries %d >= eh_max %d!",
				 le16_to_cpu(curp->p_hdr->eh_entries),
				 le16_to_cpu(curp->p_hdr->eh_max));
		return -EFSCORRUPTED;
	}

	if (logical > le32_to_cpu(curp->p_idx->ei_block)) {
		/*待插入的ext4_extent_idx结构起始逻辑块地址logical大于curp->p_idx的起始逻辑块地址， 
		就要插入curp->p_idx这个ext4_extent_idx后边，(curp->p_idx + 1)这个ext4_extent_idx后边。
		插入前，下边memmove先把(curp->p_idx+1)后边的所有ext4_extent_idx结构全向后移动一个ext4_extent_idx结构大小，
		然后把新的ext4_extent_idx插入到curp->p_idx + 1位置处*/
		/* insert after */
		ext_debug(inode, "insert new index %d after: %llu\n",
			  logical, ptr);
		ix = curp->p_idx + 1;
	} else {
		/*待插入的ext4_extent_idx结构起始逻辑块地址logical更小，就插入到curp->p_idx这个ext4_extent_idx前边。插入前，
		下边memmove先把curp->p_idx后边的所有ext4_extent_idx结构全向后移动一个ext4_extent_idx结构大小，
		然后把新的ext4_extent_idx插入到curp->p_idx位置处*/
		/* insert before */
		ext_debug(inode, "insert new index %d before: %llu\n",
			  logical, ptr);
		ix = curp->p_idx;
	}

	if (unlikely(ix > EXT_MAX_INDEX(curp->p_hdr))) {
		EXT4_ERROR_INODE(inode, "ix > EXT_MAX_INDEX!");
		return -EFSCORRUPTED;
	}
    /*ix是curp->p_idx或者(curp->p_idx+1)。len是ix这个索引节点的ext4_extent_idx结构
	到索引节点最后一个ext4_extent_idx结构(有效的)之间所有的ext4_extent_idx结构个数。
	注意，EXT_LAST_INDEX(curp->p_hdr)是索引节点最后一个有效的ext4_extent_idx结构，
	如果索引节点只有一个ext4_extent_idx结构，那EXT_LAST_INDEX(curp->p_hdr)就指向这第一个ext4_extent_idx结构*/
	len = EXT_LAST_INDEX(curp->p_hdr) - ix + 1;
	BUG_ON(len < 0);
	if (len > 0) {
		ext_debug(inode, "insert new index %d: "
				"move %d indices from 0x%p to 0x%p\n",
				logical, len, ix, ix + 1);
		/*在分界点，给要插入的节点腾出一个位置*/
		memmove(ix + 1, ix, len * sizeof(struct ext4_extent_idx));
	}
	/*现在ix指向ext4_extent_idx结构是空闲的，用它保存要插入的逻辑块地址logial和对应的物理块号。
	相当于把本次要插入ext4 extent b+树的ext4_extent_idx插入到ix指向的ext4_extent_idx位置处*/
	ix->ei_block = cpu_to_le32(logical);
	ext4_idx_store_pblock(ix, ptr);
	le16_add_cpu(&curp->p_hdr->eh_entries, 1);

	if (unlikely(ix > EXT_LAST_INDEX(curp->p_hdr))) {
		EXT4_ERROR_INODE(inode, "ix > EXT_LAST_INDEX!");
		return -EFSCORRUPTED;
	}

	err = ext4_ext_dirty(handle, inode, curp);
	ext4_std_error(inode->i_sb, err);

	return err;
}

/*
 * ext4_ext_grow_indepth:
 * implements tree growing procedure:
 * - allocates new block
 * - moves top-level data (index block or leaf) into the new block
 * - initializes new top-level, creating index that points to the
 *   just created block
 */
static int ext4_ext_grow_indepth(handle_t *handle, struct inode *inode,
				 unsigned int flags)
{
	struct ext4_extent_header *neh;
	struct buffer_head *bh;
	ext4_fsblk_t newblock, goal = 0;
	struct ext4_super_block *es = EXT4_SB(inode->i_sb)->s_es;
	int err = 0;
	size_t ext_size = 0;

	/* Try to prepend new index to old one */
	if (ext_depth(inode))
		goal = ext4_idx_pblock(EXT_FIRST_INDEX(ext_inode_hdr(inode)));
	if (goal > le32_to_cpu(es->s_first_data_block)) {
		flags |= EXT4_MB_HINT_TRY_GOAL;
		goal--;
	} else
		goal = ext4_inode_to_goal_block(inode);
	
	/*分配一个新的物理块，返回物理块号newblock。这个物理块4K大小，
	是本次新创建的索引节点或者叶子节点，将来会保存索引节点的头结构ext4_extent_header+N个ext4_extent_idx
	或者或者叶子结点ext4_extent_header+N个ext4_extent结构*/
	newblock = ext4_new_meta_blocks(handle, inode, goal, flags,
					NULL, &err);
	if (newblock == 0)
		return err;
	//bh映射到newblock这个物理块
	bh = sb_getblk_gfp(inode->i_sb, newblock, __GFP_MOVABLE | GFP_NOFS);
	if (unlikely(!bh))
		return -ENOMEM;
	lock_buffer(bh);

	err = ext4_journal_get_create_access(handle, inode->i_sb, bh,
					     EXT4_JTR_NONE);
	if (err) {
		unlock_buffer(bh);
		goto out;
	}

	ext_size = sizeof(EXT4_I(inode)->i_data);
	/*把ext4 extent B+树的根节点的数据(头结构ext4_extent_header+4个ext4_extent_idx
	或者叶子结点ext4_extent_header+4个ext4_extent结构)复制到bh->b_data。
	相当于把根节点的数据复制到上边新创建的物理块，腾空根节点*/
	/* move top-level index/leaf into new block */
	memmove(bh->b_data, EXT4_I(inode)->i_data, ext_size);
	/* zero out unused area in the extent block */
	memset(bh->b_data + ext_size, 0, inode->i_sb->s_blocksize - ext_size);

	/*neh指向bh首地址，这些内存的数据是前边向bh->b_data复制的根节点的头结构ext4_extent_header*/
	/* set size of new block */
	neh = ext_block_hdr(bh);
	//如果ext4 extent B+树有索引节点，neh指向的内存作为索引节点
	/* old root could have indexes or leaves
	 * so calculate e_max right way */
	if (ext_depth(inode))
		neh->eh_max = cpu_to_le16(ext4_ext_space_block_idx(inode, 0));
	else//如果ext4 extent B+树没有索引节点，只有根节点，neh指向的内存作为叶子结点
		neh->eh_max = cpu_to_le16(ext4_ext_space_block(inode, 0));
	neh->eh_magic = EXT4_EXT_MAGIC;
	ext4_extent_block_csum_set(inode, neh);
	set_buffer_uptodate(bh);
	set_buffer_verified(bh);
	unlock_buffer(bh);

	err = ext4_handle_dirty_metadata(handle, inode, bh);
	if (err)
		goto out;

	/* Update top-level index: num,max,pointer */
	//现在neh又指向ext4 extent B+根节点
	neh = ext_inode_hdr(inode);
	//根节点现在只有一个叶子节点的ext4_extent结构在使用或者只有一个索引节点的ext4_extent_idx结构在使用
	neh->eh_entries = cpu_to_le16(1);
	/*这是把前边新创建的索引节点或者叶子节点的物理块号newblock
	记录到根节点第一个ext4_extent_idx结构的ei_leaf_lo和ei_leaf_hi成员。
	这样就建立了根节点与新创建的物理块号是newblock的叶子结点或索引节点的联系。
	因为通过根节点第一个ext4_extent_idx结构的ei_leaf_lo和ei_leaf_hi成员，
	就可以找到这个新创建的叶子节点或者索引节点的物理块号newblock*/
	ext4_idx_store_pblock(EXT_FIRST_INDEX(neh), newblock);
	//如果neh->eh_depth是0，说明之前ext4 extent B+树深度是0，即只有根节点
	if (neh->eh_depth == 0) {
		/* Root extent block becomes index block */
		/*以前B+树只有根节点，没有索引节点。现在根节点作为索引节点，这是计算根节点最多可容纳的ext4_extent_idx结构个数，4*/
		neh->eh_max = cpu_to_le16(ext4_ext_space_root_idx(inode, 0));
		/*以前B+树只有根节点，没有索引节点，根节点都是ext4_extent结构，
		现在B+树根节点下添加了newblock这个叶子节点。根节点成了根索引节点，
		因此原来第一个ext4_extent结构要换成ext4_extent_idx结构，
		下边赋值就是把原来的根节点第一个ext4_extent的起始逻辑块地址赋值给现在根节点的第一个ext4_extent_idx的起始逻辑块地址*/
		EXT_FIRST_INDEX(neh)->ei_block =
			EXT_FIRST_EXTENT(neh)->ee_block;
	}
	ext_debug(inode, "new root: num %d(%d), lblock %d, ptr %llu\n",
		  le16_to_cpu(neh->eh_entries), le16_to_cpu(neh->eh_max),
		  le32_to_cpu(EXT_FIRST_INDEX(neh)->ei_block),
		  ext4_idx_pblock(EXT_FIRST_INDEX(neh)));
	//ext4 extent B+树增加了一层索引节点或叶子结点，即物理块号是newblock的那个，树深度加1
	le16_add_cpu(&neh->eh_depth, 1);
	err = ext4_mark_inode_dirty(handle, inode);
out:
	brelse(bh);

	return err;
}
```