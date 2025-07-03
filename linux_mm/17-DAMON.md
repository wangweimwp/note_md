
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