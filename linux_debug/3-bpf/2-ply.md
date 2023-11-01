**使用**

必须挂载debugfs

```bash
ply 'kprobe:__alloc_pages_nodemask { @[stack] = count(); }'

perf stat -a -- sleep 10


perf record -a -g -- sleep 10
perf report
#生成火焰图
perf script -i perf.data &> perf.unfold
FlameGraph/stackcollapse-perf.pl perf.unfold &> perf.folded
FlameGraph/flamegraph.pl perf.folded > perf.svg
```

**ply的栈格式转火焰图**

1，  将`{ `替换为`@[` , 将 ` }:`替换为`]:`

2，`./FlameGraph/stackcollapse-bpftrace.pl ply_nozram_21.data > ply_nozram_21.folded`

3，`./FlameGraph/flamegraph.pl ply_nozram_21.folded > ply_nozram_21.svg`








