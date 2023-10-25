## **挂载**

使用`swapon`命令启动swap分区时，内核调用对应系统调用

```c
SYSCALL_DEFINE2(swapon, const char __user *, specialfile, int, swap_flags)
```

`swapon`系统调用调用`init_swap_address_space`，初始化`struct address_space->a_ops = & swap_aops`,后续kswap线程向swap分区回写数据时会调用



## 回写

内存紧张时，

```c
shrink_inactive_list中if (stat.nr_unqueued_dirty == nr_taken)成立
    ->wakeup_flusher_threads(WB_REASON_VMSCAN)唤醒writeback回写进程
```

`wakeup_flusher_threads`会唤醒虽有文件系统的数据回写，内核使用delay work（延时工作队列）实现文件系统的数据回写


