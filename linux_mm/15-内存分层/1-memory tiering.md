**概念**

很久以前，计算机只有一种类型的内存 memory，所以一个特定系统内的内存是可以互相替换使用的。非统一内存访问（NUMA）系统的出现使情况大大复杂化，因为有些内存的访问速度会比其他内存快，内存管理算法必须适应这一点，否则性能会受到影响。但 NUMA 只是一个开始；如今出现的分层内存系统（tiered-memory system），可能会包括具有不同性能特性的多层内存，这正在给我们增加新的挑战。目前正在审查的几个相关 patch set 有助于说明这里有哪些类型的问题需要我们解决。

NUMA 系统的核心挑战是要确保内存能要能根据后面需要在哪里使用，而决定从哪个 node 上来分配。一个进程如果能主要在本地节点（local node）的内存上运行，就会比使用大量远程内存的进程表现得更好。因此，为某个特定 page 来找到合适的分配来源是一个一次性的任务；一旦这个 page 和它的用户都进入了同一个 NUMA 节点，问题就解决了，剩下的唯一问题就是避免再次将它们分开。

Tiered memory （分层内存）是建立在 NUMA 概念之上的，但有一些区别。可能会有一段内存区域是作为一个没有 CPU 的 NUMA 节点来管理的，因此该内存将不会被视为系统中任何进程的本地内存（local memory）。一般来说，这些无 CPU 的 node 上的内存要比正常的系统 DRAM 慢。例如它可能是一个 size 巨大的 persistent memory


**内核实现**
内核用`struct memory_tier`描述内存分层
```
static int top_tier_adistance;
/*
 * node_demotion[] examples:
 *
 * Example 1:
 *
 * Node 0 & 1 are CPU + DRAM nodes, node 2 & 3 are PMEM nodes.
 *
 * node distances:
 * node   0    1    2    3
 *    0  10   20   30   40
 *    1  20   10   40   30
 *    2  30   40   10   40
 *    3  40   30   40   10
 *
 * memory_tiers0 = 0-1
 * memory_tiers1 = 2-3
 *
 * node_demotion[0].preferred = 2
 * node_demotion[1].preferred = 3
 * node_demotion[2].preferred = <empty>
 * node_demotion[3].preferred = <empty>
 *
 * Example 2:
 *
 * Node 0 & 1 are CPU + DRAM nodes, node 2 is memory-only DRAM node.
 *
 * node distances:
 * node   0    1    2
 *    0  10   20   30
 *    1  20   10   30
 *    2  30   30   10
 *
 * memory_tiers0 = 0-2
 *
 * node_demotion[0].preferred = <empty>
 * node_demotion[1].preferred = <empty>
 * node_demotion[2].preferred = <empty>
 *
 * Example 3:
 *
 * Node 0 is CPU + DRAM nodes, Node 1 is HBM node, node 2 is PMEM node.
 *
 * node distances:
 * node   0    1    2
 *    0  10   20   30
 *    1  20   10   40
 *    2  30   40   10
 *
 * memory_tiers0 = 1
 * memory_tiers1 = 0
 * memory_tiers2 = 2
 *
 * node_demotion[0].preferred = 2
 * node_demotion[1].preferred = 0
 * node_demotion[2].preferred = <empty>
 *
 */
```


ACPI hmat（ACPI Heterogeneous Memory Attribute Table）使用了内存分层
```c
hmat_init
    ->解析ACPI table中获取hmat信息
    ->hmat_set_default_dram_perf  初始化每个节点的内存读写贷款和读写延时
    ->register_mt_adistance_algorithm 注册abstrac distance算法
```



memory_tier初始化

```c
memory_tier_init
    ->set_node_memory_tier//初始化每个内存节点的pgdat->memtier

```



直接访问（Direct Access，DAX） 中的应用

dax kmem

```c
dev_dax_kmem_probe
    ->mt_calc_adistance    //计算距离
    ->kmem_find_alloc_memory_type    //申请struct memory_dev_type 并挂到kmem_memory_types列表
    ->init_node_memory_type    // 将 struct memory_dev_type放到node_memory_types数组中
```

扫描LRU列表时

```c
shrink_folio_list
    ->demote_folio_list
        ->next_demotion_node    //获取demotion路径上的下一个节点
        ->node_get_allowed_targets    //获取pgdat->memtier->lower_tier_mask
        ->migrate_pages        //页面迁移
```






