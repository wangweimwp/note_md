1. DAMON的madvise用法，动态监测页面访问情况，对符合要求的页面调用madvise系统调用
```text

以下命令应用了一种策略，其内容为：“如果某个大小在[4KiB，8KiB]范围内的内存区域，在[10，20]的聚合时间段内每秒的访问次数处于[0，5]之间，则对该区域进行页面置换操作。在进行页面置换时，每秒的耗时上限为10毫秒，并且每次页面置换的内存大小上限为1GiB。在上述限制条件下，应优先置换较旧的内存区域。此外，每5秒检查一次系统的可用内存率，当可用内存率低于50%时启动监测和页面置换操作，但若可用内存率高于60%或低于30%则停止该操作。”

# cd <sysfs>/kernel/mm/damon/admin
# # populate directories
# echo 1 > kdamonds/nr_kdamonds; echo 1 > kdamonds/0/contexts/nr_contexts;
# echo 1 > kdamonds/0/contexts/0/schemes/nr_schemes
# cd kdamonds/0/contexts/0/schemes/0
# # set the basic access pattern and the action
# echo 4096 > access_pattern/sz/min
# echo 8192 > access_pattern/sz/max
# echo 0 > access_pattern/nr_accesses/min
# echo 5 > access_pattern/nr_accesses/max
# echo 10 > access_pattern/age/min
# echo 20 > access_pattern/age/max
# echo pageout > action
# # set quotas
# echo 10 > quotas/ms
# echo $((1024*1024*1024)) > quotas/bytes
# echo 1000 > quotas/reset_interval_ms
# # set watermark
# echo free_mem_rate > watermarks/metric
# echo 5000000 > watermarks/interval_us
# echo 600 > watermarks/high
# echo 500 > watermarks/mid
# echo 300 > watermarks/low

```

2. DAMON的LRU排序用法，对符合要求的页面，调整这些页面在LRU上的位置。
```text
如下是一个运行时的命令示例，使DAMON_LRU_SORT查找访问频率超过50%的区域并对其进行LRU的优先级的提升，同时降低那些超过120秒无人访问的内存区域的优先级。优先级的处理被限制在最多1%的CPU以避免DAMON_LRU_SORT消费过多CPU时间。在系统空闲内存超过50%时DAMON_LRU_SORT停止工作，并在低于40%时重新开始工作。如果DAMON_RECLAIM没有取得进展且空闲内存低于20%，再次让DAMON_LRU_SORT停止工作，以此回退到以LRU链表为基础以页面为单位的内存回收上。 

# cd /sys/modules/damon_lru_sort/parameters
# echo 500 > hot_thres_access_freq
# echo 120000000 > cold_min_age
# echo 10 > quota_ms
# echo 1000 > quota_reset_interval_ms
# echo 500 > wmarks_high
# echo 400 > wmarks_mid
# echo 200 > wmarks_low
# echo Y > enabled


问题1：问频率超过50%怎么理解？
问题2：进行LRU的优先级的提升怎么理解？
```

3. DAMON的reclaim用法 超过一定时间未被访问，回收这些页面
```text
下面的运行示例命令使DAMON_RECLAIM找到30秒或更长时间没有访问的内存区域并“回收”？为了避免DAMON_RECLAIM在分页操作中消耗过多的CPU时间，回收被限制在每秒1GiB以内。它还要求DAMON_RECLAIM在系统的可用内存率超过50%时不做任何事情，但如果它低于40%时就开始真正的工作。如果DAMON_RECLAIM没有取得进展，因此空闲内存率低于20%，它会要求DAMON_RECLAIM再次什么都不做，这样我们就可以退回到基于LRU列表的页面粒度回收了::

# cd /sys/module/damon_reclaim/parameters
# echo 30000000 > min_age
# echo $((1 * 1024 * 1024 * 1024)) > quota_sz
# echo 1000 > quota_reset_interval_ms
# echo 500 > wmarks_high
# echo 400 > wmarks_mid
# echo 200 > wmarks_low
# echo Y > enabled
```