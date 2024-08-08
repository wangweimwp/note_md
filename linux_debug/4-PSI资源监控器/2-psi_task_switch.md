每次向/proc/pressure/memory写入"some 150000 1000000，会触发一个psi_trigger_create，
并初始化struct psi_trigger，同事初始化该结构体里包含的一个等待队列，用于poll机制，
向用户空间反馈阻塞情况

```c
psi_task_switch
    ->psi_group_change//将psi_avgs_work加入延时工作队列，每隔2秒运行一次
        ->psi_avgs_work
            ->update_triggers
                ->wake_up_interruptible(&t->event_wait);//唤醒睡眠的poll进程


```

```c
void psi_task_switch(struct task_struct *prev, struct task_struct *next,
             bool sleep)
{
    struct psi_group *group, *common = NULL;
    int cpu = task_cpu(prev);
    u64 now = cpu_clock(cpu);

    if (next->pid) {
        psi_flags_change(next, 0, TSK_ONCPU);//切上来的进程标记为oncpu
        /*
         * Set TSK_ONCPU on @next's cgroups. If @next shares any
         * ancestors with @prev, those will already have @prev's
         * TSK_ONCPU bit set, and we can stop the iteration there.
         */
        group = task_psi_group(next);
        do {
            if (per_cpu_ptr(group->pcpu, cpu)->state_mask &
                PSI_ONCPU) {
                common = group;//切上来的进程所的上级cgroup已经标记oncpu了
                break;
            }

            psi_group_change(group, cpu, 0, TSK_ONCPU, now, true);
        } while ((group = group->parent));//修改cgroup的SPI标记，
    }

    if (prev->pid) {
        int clear = TSK_ONCPU, set = 0;
        bool wake_clock = true;

        /*
         * When we're going to sleep, psi_dequeue() lets us
         * handle TSK_RUNNING, TSK_MEMSTALL_RUNNING and
         * TSK_IOWAIT here, where we can combine it with
         * TSK_ONCPU and save walking common ancestors twice.
         */
        if (sleep) {
            clear |= TSK_RUNNING;
            if (prev->in_memstall)
                clear |= TSK_MEMSTALL_RUNNING;
            if (prev->in_iowait)
                set |= TSK_IOWAIT;

            /*
             * Periodic aggregation shuts off if there is a period of no
             * task changes, so we wake it back up if necessary. However,
             * don't do this if the task change is the aggregation worker
             * itself going to sleep, or we'll ping-pong forever.
             */
            if (unlikely((prev->flags & PF_WQ_WORKER) &&
                     wq_worker_last_func(prev) == psi_avgs_work))
                wake_clock = false;
        }

        psi_flags_change(prev, clear, set);

        group = task_psi_group(prev);
        do {
            if (group == common)
                break;//切下去的进程和切上来的进程在同一个cgroup中
            psi_group_change(group, cpu, clear, set, now, wake_clock);
        } while ((group = group->parent));

        /*
         * TSK_ONCPU is handled up to the common ancestor. If there are
         * any other differences between the two tasks (e.g. prev goes
         * to sleep, or only one task is memstall), finish propagating
         * those differences all the way up to the root.
         * TSK_ONCPU 标记传播到两个进程的公共祖先cgroup就停止了。
         * 若两个进程的其他标记为有差异，则将这个差异一直传播到root cgroup
         */
        if ((prev->psi_flags ^ next->psi_flags) & ~TSK_ONCPU) {
            clear &= ~TSK_ONCPU;
            for (; group; group = group->parent)
                psi_group_change(group, cpu, clear, set, now, wake_clock);
        }
    }
}
```

psi_group_change

```c
static void psi_group_change(struct psi_group *group, int cpu,
                 unsigned int clear, unsigned int set, u64 now,
                 bool wake_clock)
{
    struct psi_group_cpu *groupc;
    unsigned int t, m;
    enum psi_states s;
    u32 state_mask;

    groupc = per_cpu_ptr(group->pcpu, cpu);

    /*
     * First we update the task counts according to the state
     * change requested through the @clear and @set bits.
     *
     * Then if the cgroup PSI stats accounting enabled, we
     * assess the aggregate resource states this CPU's tasks
     * have been in since the last change, and account any
     * SOME and FULL time these may have resulted in.
     */
    write_seqcount_begin(&groupc->seq);

    /*
     * Start with TSK_ONCPU, which doesn't have a corresponding
     * task count - it's just a boolean flag directly encoded in
     * the state mask. Clear, set, or carry the current state if
     * no changes are requested.
     */
    if (unlikely(clear & TSK_ONCPU)) {
        state_mask = 0;
        clear &= ~TSK_ONCPU;
    } else if (unlikely(set & TSK_ONCPU)) {
        state_mask = PSI_ONCPU;
        set &= ~TSK_ONCPU;
    } else {
        state_mask = groupc->state_mask & PSI_ONCPU;
    }

    /*
     * The rest of the state mask is calculated based on the task
     * counts. Update those first, then construct the mask.
     */
    for (t = 0, m = clear; m; m &= ~(1 << t), t++) {
        if (!(m & (1 << t)))
            continue;
        if (groupc->tasks[t]) {
            groupc->tasks[t]--;//该psi group每个PSI状态下的进程数量
        } else if (!psi_bug) {
            printk_deferred(KERN_ERR "psi: task underflow! cpu=%d t=%d tasks=[%u %u %u %u] clear=%x set=%x\n",
                    cpu, t, groupc->tasks[0],
                    groupc->tasks[1], groupc->tasks[2],
                    groupc->tasks[3], clear, set);
            psi_bug = 1;
        }
    }

    for (t = 0; set; set &= ~(1 << t), t++)
        if (set & (1 << t))
            groupc->tasks[t]++;//该psi group每个PSI状态下的进程数量

    if (!group->enabled) {
        /*
         * On the first group change after disabling PSI, conclude
         * the current state and flush its time. This is unlikely
         * to matter to the user, but aggregation (get_recent_times)
         * may have already incorporated the live state into times_prev;
         * avoid a delta sample underflow when PSI is later re-enabled.
         */
        if (unlikely(groupc->state_mask & (1 << PSI_NONIDLE)))
            record_times(groupc, now);

        groupc->state_mask = state_mask;

        write_seqcount_end(&groupc->seq);
        return;
    }

    for (s = 0; s < NR_PSI_STATES; s++) {
        if (test_state(groupc->tasks, s, state_mask & PSI_ONCPU))
            state_mask |= (1 << s);//遍历每个PSI状态，看是否有进程处于当前状态下
    }

    /*
     * Since we care about lost potential, a memstall is FULL
     * when there are no other working tasks, but also when
     * the CPU is actively reclaiming and nothing productive
     * could run even if it were runnable. So when the current
     * task in a cgroup is in_memstall, the corresponding groupc
     * on that cpu is in PSI_MEM_FULL state.
     */
    if (unlikely((state_mask & PSI_ONCPU) && cpu_curr(cpu)->in_memstall))
        state_mask |= (1 << PSI_MEM_FULL);

    record_times(groupc, now);

    groupc->state_mask = state_mask;

    write_seqcount_end(&groupc->seq);

    if (state_mask & group->rtpoll_states)
        psi_schedule_rtpoll_work(group, 1, false);

    if (wake_clock && !delayed_work_pending(&group->avgs_work))
        schedule_delayed_work(&group->avgs_work, PSI_FREQ);//每隔2秒放入工作队列
}
```
