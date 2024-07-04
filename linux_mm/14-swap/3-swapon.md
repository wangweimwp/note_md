```c
SYSCALL_DEFINE2(swapon, const char __user *, specialfile, int, swap_flags)
    ->alloc_swap_info
    ->//拿到名为specialfile的设备节点，struct address_space，inode等信息
    ->claim_swapfile //独占方式打开bdev,设置blcoksize为PAGESIZE
    ->read_mapping_page//读取swap分区的第一个page
    ->//根据读取的第一个page，获取 union swap_header
    ->read_swap_header//根据 swap_header初始化struct swap_info_struct 一些成员
                      //返回可以分配的总页数。取决于两点，swap_entry_t中swap offset的bit数
    ->swap_map = vzalloc(maxpages); //分配swap_map数组内存
    ->//初始化struct swap_info_struct的struct swap_cluster_info
    ->swap_cgroup_swapon
    ->setup_swap_map_and_extents//建立struct swap_extent, 该结构体用于将磁盘
    //上的block和内存的page联系起来，许多个 struct swap_extent挂载红黑树上
    ->init_swap_address_space    //* One swap address space for each 64M swap space */
    ->enable_swap_info//建立swap_map、struct swap_cluster_info
                        //的联系，设置以前全局变量
```

```c
static struct swap_info_struct *alloc_swap_info(void)
{
    struct swap_info_struct *p;
    struct swap_info_struct *defer = NULL;
    unsigned int type;
    int i;

    p = kvzalloc(struct_size(p, avail_lists, nr_node_ids), GFP_KERNEL);
    if (!p)
        return ERR_PTR(-ENOMEM);

    if (percpu_ref_init(&p->users, swap_users_ref_free,
                PERCPU_REF_INIT_DEAD, GFP_KERNEL)) {//初始化引用计数
        kvfree(p);
        return ERR_PTR(-ENOMEM);
    }

    spin_lock(&swap_lock);
    for (type = 0; type < nr_swapfiles; type++) {
        if (!(swap_info[type]->flags & SWP_USED))
            break;//找到一个不再使用的
    }
    if (type >= MAX_SWAPFILES) {//没有找到
        spin_unlock(&swap_lock);
        percpu_ref_exit(&p->users);
        kvfree(p);
        return ERR_PTR(-EPERM);
    }
    if (type >= nr_swapfiles) {
        p->type = type;
        /*
         * Publish the swap_info_struct after initializing it.
         * Note that kvzalloc() above zeroes all its fields.
         */
        smp_store_release(&swap_info[type], p); /* rcu_assign_pointer() */
        nr_swapfiles++;//增加位置
    } else {
        defer = p;
        p = swap_info[type];//复用旧的位置
        /*
         * Do not memset this entry: a racing procfs swap_next()
         * would be relying on p->type to remain valid.
         */
    }
    p->swap_extent_root = RB_ROOT;
    plist_node_init(&p->list, 0);
    for_each_node(i)
        plist_node_init(&p->avail_lists[i], 0);
    p->flags = SWP_USED;
    spin_unlock(&swap_lock);
    if (defer) {//有旧的位置用，释放掉申请的
        percpu_ref_exit(&defer->users);
        kvfree(defer);
    }
    spin_lock_init(&p->lock);
    spin_lock_init(&p->cont_lock);
    init_completion(&p->comp);

    return p;
}
```
