
```c


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
```