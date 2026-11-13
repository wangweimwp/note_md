```bash
# 查看网卡 RSS 队列数
ethtool -l eth0

# 查看当前 RSS 哈希字段配置
ethtool -n eth0 rx-flow-hash tcp4

# 查看间接表（需要 ethtool 较新版本）
ethtool -x eth0

# 查看各队列的中断亲和性
cat /proc/interrupts | grep eth0
cat /proc/irq/{irq_num}/smp_affinity


#RPS 设置
echo f > /sys/class/net/eth0/queues/rx-0/rps_cpus
echo 4096 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
echo 32768 > /proc/sys/net/core/rps_sock_flow_entries



ethtool -S eth0             #Show adapter statistics

# 查看内核完整路由表 
ip route show cache



# 直接读 /proc
# 分片队列允许占用的最大内存（即你要的「最大值」）
freetech:~# cat /proc/sys/net/ipv4/ipfrag_high_thresh
4194304
# 回收触发下水位：内存降到此值才停止丢弃未完成队列
freetech:~# cat /proc/sys/net/ipv4/ipfrag_low_thresh
3145728
# 碎片哈希防碰撞最大距离（RFC 5722 反规避用）
freetech:~# cat /proc/sys/net/ipv4/ipfrag_max_dist
64
# 碎片在队列中的存活超时，超时即丢弃
freetech:~# cat /proc/sys/net/ipv4/ipfrag_time
32
# 哈希密钥重生成间隔，0 表示内核启动后密钥固定
freetech:~# cat /proc/sys/net/ipv4/ipfrag_secret_interval
0

#内存达到 high_thresh 后，内核开始回收（丢弃）未完成的碎片队列，回收到 low_thresh 才停止


##arp网络邻居

#我曾经在一个嵌入式项目里遇到过一个问题：两个设备之间通信偶尔会中断几秒钟。查了半天，
#发现是 reachable_time 设置得太短，导致邻居条目频繁进入STALE状态，每次都要重新探测。
#后来我把这个值调大了一些，问题就解决了。所以，/proc/sys/net/ipv4/neigh/eth0/base_reachable_time_ms 
#这个参数，值得你关注。

freetech:/soc/data/user_data# arp
? (192.168.2.2) at aa:bb:cc:dd:ee:01 [ether]  on eth1
? (172.20.3.34) at aa:bb:cc:dd:ee:21 [ether] PERM on eth0.3
? (172.20.4.36) at aa:bb:cc:dd:ee:23 [ether] PERM on eth0.4
? (172.20.0.2) at aa:bb:cc:dd:ee:01 [ether] PERM on eth0

## 看看 ip neigh show 的输出，说不定就是邻居状态在反复跳



```


arp表
展示IP地址和MAC地址的对应关系

交换机MAC
展示MAC地址和端口的对应关系

网关：
网关 = 跨网段的出口；默认网关 = 所有"出不去的包"的统一去向；而且网关必须和你同子网、且能 ARP 到你，否则你出不了网
