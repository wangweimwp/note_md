内核补丁`commit 34e58cac6d8f ("mm: workingset: let cache workingset challenge anon")`中提到`Reclaim can still decide how to balance pressure among the two LRUsdepending on the IO situation`，寻找如何平衡看补丁`mm: balance LRU lists based on relative thrashing`

在每次进行内存回收时候，会对每个LRU列表该扫描多收页面进行计算（`shrink_lruvec->get_scan_count`），计算原理是-如果页缓存(page cache)颠簸，则缓存页在内存中需要更多的时间，而匿名链表上可能有较冷的页，那么就要更多的扫描匿名页面。同样，如果换出的页出swapin，则表明我们回收匿名页过于激进，应该退回一点。

计算回收成本的数据结构时`strcut lruvec->cost_cost  和  strcut lruvec->file_cost`，

```c
//三个地方记录回收
shrink_inactive_list->lru_note_cost
shrink_active_list->lru_note_cost
workingset_refault->lru_note_cost_refault


```












