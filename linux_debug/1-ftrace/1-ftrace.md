**记录延时**

```bash
cd /sys/kernel/debug/tracing
cat available_tracers

echo wakeup > current_tracer
echo 1 > tracing_on
echo 0 > tracing_on
cat trace
```

结果如下

# 



**ftrace检测函数调用**

```bash
echo 'p:myprobe1 __alloc_pages_nodemask' > kprobe_events
echo 'r:myprobe2 __alloc_pages_nodemask' >> kprobe_events #监控函数返回
echo 1 > events/kprobes/myprobe1/enable
echo 1 > events/kprobes/myprobe2/enable

```


