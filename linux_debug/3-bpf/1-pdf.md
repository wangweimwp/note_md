环境搭建
```bash
sudo apt-get install bpfcc-tools 

sudo apt install bpftrace -y
```



```bash
tracepoint类的参数用args->count       kprobe类的参数用arg0，arg1.....argN

sudo bpftrace -l 'kprobe:vfs_*' | wc -l   sudo bpftrace -l 'k:vfs_*' | wc -l 

bpftrace -l 'software:*'            bpftrace -l 'hardware:*'

bpftrace -lv tracepoint:syscalls:sys_enter_read            bpftrace -lv t:syscalls:sys_enter_read

bpftrace -e 'tracepoint:syscalls:sys_enter_clone {printf("->clone() by %s PID %d\n", comm, pid)} tracepoint:syscalls:sys_exit_clone {printf("<-clone() return%d PID %d \n", args->ret, pid)}'

bpftrace -e 'tracepoint:syscalls:sys_enter_clone {printf("->clone() by %s PID %d arg=%d\n", comm, pid, $1)}'  123

bpftrace -l 'usdt:/usr/bin/busybox'

bpftrace -e 't:block:block_rq_insert {printf("%s\n", kstack);}'

sudo bpftrace -e 'k:mem_cgroup_can_attach {printf("%s\n", kstack);}'

bpftrace -e 't:block:block_rq_insert { @[kstack] = count()  }'

bpftrace --unsafe -e 'tracepoint:syscalls:sys_enter_execve  { system("ps -p %d \n", pid ); }'

bpftrace -d -e 'k:vfs_read {@[pid] = count(); }'

bpftrace -d -e 'kretprobe:vfs_read {printf("%d\n", retval); }'

../FlameGraph/stackcollapse-perf.pl perf_script        //将其他格式转化成火焰图格式
perf script --header | ./stackcollapse-perf.pl | ./flamegraph.pl > flame1.svg     //火焰图  


cat /proc/kallsyms | grep "readahead"
funccount-bpfcc '*readahead*'


list *(handle_mm_fault+263)
```
