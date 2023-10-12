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


