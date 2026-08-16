# jbd2_journal_lock_updates() 详细解析

> 内核树：`fs/jbd2/transaction.c:859`，配对函数 `jbd2_journal_unlock_updates()` 在 `:896`。
> 由 `EXPORT_SYMBOL` 导出（`fs/jbd2/journal.c:60-61`），ext4 / ocfs2 等文件系统均使用。
> 本文结合 fast commit 提交流水线（Step 4，`fs/ext4/fast_commit.c:1102`）讲解其角色。

---

## 1. 函数定位与语义

`jbd2_journal_lock_updates()` 建立一道 **transaction barrier（事务屏障）**：

> Locks out any further updates from being started, and blocks until all
> existing updates have completed, returning only once the journal is in a
> quiescent state with no updates running.

中文：阻止任何**新的**更新 handle 启动，并阻塞到**所有已存在**的更新 handle 完成，最终返回时 journal 处于"静止（quiescent）"状态——没有任何 handle 在跑，也没有新 handle 能进来。

注意它**不提交任何事务**、**不写任何块**，纯粹是"让 journal 安静下来"的同步原语。

调用方一览（`fs/` 内）：

| 调用点 | 场景 |
|---|---|
| `ext4/fast_commit.c:1102` | fc 提交流水线 Step 4，置 `FC_COMMITTING` 位前 |
| `ext4/super.c:6349 / 6485` | `ext4_force_commit` / `ext4_sync_fs` |
| `ext4/ioctl.c:972 / 1104 / 1619 / 1747` | `EXT4_IOC_SETFREQ` / `RESIZE` / `SWAP_BOOT` / `MOVE_EXT` 等需要静止 journal 的操作 |
| `ext4/inode.c:6545` | `ext4_inode_journal_mode` 切换 |
| `ocfs2/*` | ocfs2 的 journal 静止操作 |

---

## 2. 函数本体（带逐行注释）

```c
void jbd2_journal_lock_updates(journal_t *journal)
{
    jbd2_might_wait_for_commit(journal);          // :861  might_sleep 类注解，声明会睡眠

    write_lock(&journal->j_state_lock);           // :863  持写锁改 journal 状态字段
    ++journal->j_barrier_count;                   // :864  ★ 屏障计数器 +1（核心标志）

    /* 等待没有 "reserved handles" 残留 */
    if (atomic_read(&journal->j_reserved_credits)) {          // :867
        write_unlock(&journal->j_state_lock);                 // :868  先放锁再睡
        wait_event(journal->j_wait_reserved,                 // :869  等预留额度归零
                   atomic_read(&journal->j_reserved_credits) == 0);
        write_lock(&journal->j_state_lock);                  // :871  重新持锁
    }

    /* 等待没有正在运行的 t_updates */
    jbd2_journal_wait_updates(journal);           // :875  ★ 排空当前所有 running handle

    write_unlock(&journal->j_state_lock);         // :877  放状态锁

    /*
     * 现在已经对"普通更新"建立了屏障，但还需要对"其他 lock_updates
     * 调用者"也建立屏障，确保特殊 journal-locked 操作之间也串行化。
     */
    mutex_lock(&journal->j_barrier);              // :885  ★ 互斥多个 jbd2_journal_lock_updates 调用者本身
}
```

三个 `★` 标记处就是该函数的四层机制（见第 3 节）。

---

## 3. 四层屏障机制

### 3.1 `j_barrier_count++` —— 屏障计数器（transaction.c:864）

`j_barrier_count` 是 `journal_t` 的一个 `int` 字段（`include/linux/jbd2.h:784`），注释明确：

> Number of processes waiting to create a barrier lock [j_state_lock, no lock for quick racy checks]

它本身**不阻塞任何东西**，只是一个"标志位 + 计数器"，告诉所有后续想 `jbd2__journal_start()` 的线程："现在有个 barrier 在生效"。屏障撤掉（`jbd2_journal_unlock_updates` 里 `--j_barrier_count` 后）会 `wake_up_all(&j_wait_transaction_locked)` 唤醒等待者。

### 3.2 等待 reserved credits 清空（transaction.c:867-872）

如果还有 `j_reserved_credits > 0`（即存在"已预留 credit 但还没真正 start"的 handle），在 `j_wait_reserved` 上等它们归零。这保证没有"悬而未决"的预留 handle 卡在半路——否则屏障刚撤、这些预留 handle 立刻 start，会破坏静止保证。

### 3.3 `jbd2_journal_wait_updates()` —— 排空运行中的 handle（transaction.c:875 → 816）

这是**最关键的一步**：等所有**已经在跑**的 handle 完成。实现（transaction.c:816-847）：

```c
void jbd2_journal_wait_updates(journal_t *journal)
{
    DEFINE_WAIT(wait);
    while (1) {
        transaction_t *transaction = journal->j_running_transaction;  // 每次重新取，防 UAF
        if (!transaction)
            break;
        prepare_to_wait(&journal->j_wait_updates, &wait, TASK_UNINTERRUPTIBLE);
        if (!atomic_read(&transaction->t_updates)) {   // t_updates == 0 说明本事务无活动 handle
            finish_wait(&journal->j_wait_updates, &wait);
            break;
        }
        write_unlock(&journal->j_state_lock);
        schedule();                                    // 睡在 j_wait_updates
        finish_wait(&journal->j_wait_updates, &wait);
        write_lock(&journal->j_state_lock);
    }
}
```

- `t_updates`：当前 running transaction 上**活动 handle 的数量**。每个 `jbd2_journal_start` +1，`jbd2_journal_stop` -1。
- 每当 `t_updates` 减到 0，`jbd2_journal_stop` 会 `wake_up(&journal->j_wait_updates)` 唤醒这里的等待者。
- 注释（:821-830）强调每次循环都要**重新读取** `j_running_transaction`，因为它可能在 commit 路径中被 `jbd2_journal_free_transaction()` 释放——这是经典的 use-after-free 防护。

### 3.4 `mutex_lock(&j_barrier)` —— 串行化 lock_updates 调用者（transaction.c:885）

前面三步只保证"对普通更新建立屏障"。但如果有**两个并发的 `jbd2_journal_lock_updates` 调用者**（比如一个 ioctl 和一个 fsync 同时进来），它们需要互相互斥，否则"特殊 journal-locked 操作"之间会交错。`:880-884` 的注释正是这个意思，于是最后用 `j_barrier` 互斥锁串行化。

---

## 4. 新 handle 启动侧如何被拦截（transaction.c:378）

屏障要生效，必须有"拦截方"。`jbd2__journal_start()` 在分配 handle 前检查：

```c
/* :373-383 注释：允许 reserved handle 通过，否则 commit 可能在 page writeback
 * 上死锁（writeback 需要 reserved handle 来释放 buffer）。 */
if (!handle->h_reserved && journal->j_barrier_count) {   // :378
    read_unlock(&journal->j_state_lock);
    wait_event(journal->j_wait_transaction_locked,
               journal->j_barrier_count == 0);            // :380  睡到屏障撤掉
    goto repeat;                                          // :382  醒来重试
}
```

要点：

- **普通 handle（`h_reserved == 0`）**：`j_barrier_count != 0` 时，在 `j_wait_transaction_locked` 上睡，直到 `unlock` 里 `wake_up_all`。
- **reserved handle（`h_reserved != 0`）例外放行**：这是为了避免死锁。`jbd2_journal_commit_transaction()` 自身在提交过程中可能触发 page writeback，而 writeback 需要 reserved handle 来释放 buffer；若 reserved handle 也被 barrier 挡住，commit 就会和自己死锁。所以 reserved handle 永远能穿过 barrier。

---

## 5. unlock 如何解除屏障（transaction.c:896-905）

```c
void jbd2_journal_unlock_updates (journal_t *journal)
{
    J_ASSERT(journal->j_barrier_count != 0);             // :898  防御：必须成对调用
    mutex_unlock(&journal->j_barrier);                   // :900  释放互斥锁
    write_lock(&journal->j_state_lock);
    --journal->j_barrier_count;                          // :902  ★ 屏障计数 -1
    write_unlock(&journal->j_state_lock);
    wake_up_all(&journal->j_wait_transaction_locked);    // :904  ★ 唤醒所有等屏障的新 handle
}
```

`--j_barrier_count` 把屏障标志撤掉，`wake_up_all` 唤醒第 4 节里睡在 `j_wait_transaction_locked` 上的所有新 handle——它们 `goto repeat` 重新尝试 `start`，此时 `j_running_transaction` 通常已有（或能新建），于是继续。

---

## 6. 在 fast commit 流水线中的角色（Step 4，fast_commit.c:1102）

回到你之前关注的 fc 提交流水线。`ext4_fc_perform_commit()` 的 Step 4：

```c
/* Step 4: Mark all inodes as being committed. */
jbd2_journal_lock_updates(journal);                 // :1102  ★ 进入静止态
/*
 * 现在 journal 已锁：没有新 handle 能开始，旧 handle 已全部排空。
 * 把提交队列上的 inode 标记为"正在提交"。
 */
alloc_ctx = ext4_fc_lock(sb);
list_for_each_entry(iter, &sbi->s_fc_q[FC_Q_MAIN], i_fc_list) {
    ext4_set_inode_state(&iter->vfs_inode, EXT4_STATE_FC_COMMITTING);  // :1110-1111
}
ext4_fc_unlock(sb, alloc_ctx);
jbd2_journal_unlock_updates(journal);               // :1114  ★ 立刻解锁
```

**为什么 fc 需要全局静止？** fc 要把每个追踪 inode 的 extent 映射（ADD_RANGE/DEL_RANGE tag）序列化进 journal。在写 tag 期间，必须保证**没有任何正在运行的 handle 还能修改这些 inode 的 extent 树**——否则 fc 写出的 tag 会和并发修改产生不一致，crash 恢复时重放就会出错。`lock_updates` 把"运行期只记账"的窗口彻底关上，让 fc 能在**稳定的快照**上序列化。

**关键细节：持锁窗口极窄。** `lock_updates` 只包住"置 `FC_COMMITTING` 位"这 12 行（:1102-1114），置完位**立刻 `unlock`**。真正的写 tag（Step 6+）在 `unlock` 之后进行。因为 `FC_COMMITTING` 位本身就会拦住后续 `ext4_fc_track_inode()`（之前讨论过：track 时若看到 `EXT4_STATE_FC_COMMITTING` 会在 bit waitqueue 上等），所以不需要全程持锁——位本身就是"软屏障"的延续。

**调用线程自身不在 handle 上下文。** `ext4_fc_commit` 由 `ext4_sync_file` 调用，而 fsync 路径在进入前已经 `jbd2_journal_stop` 了 handle。`lock_updates` 拦的是**其他进程**的更新，不影响当前线程继续写 fc tag。

---

## 7. 与 full commit 的对比（commit.c:454）

`jbd2_journal_commit_transaction()` 在 `:454` 也调了 `jbd2_journal_wait_updates()`：

```c
// commit.c:453-454  waits for any t_updates to finish
jbd2_journal_wait_updates(journal);
commit_transaction->t_state = T_SWITCH;
```

但 full commit **不调 `lock_updates`**。区别是：

| | full commit（commit.c） | lock_updates（fc / ioctl） |
|---|---|---|
| 目的 | 提交**一个已确定**的事务镜像 | 让 journal **全局静止** |
| 对新 handle | 允许开到**下一个** running transaction | 禁止开到**任何**事务 |
| 等待范围 | 只等当前事务 `t_updates` 归零 | 等 `t_updates` 归零 + `reserved` 归零 + 互斥其他 lock 调用 |
| 机制 | 把 `j_running_transaction` 切到 `j_committing_transaction`，新 handle 自动去新事务（transaction.c:385-396） | 置 `j_barrier_count`，新 handle 在 `j_wait_transaction_locked` 睡 |

本质：full commit 提交的是一个**独立、封闭**的事务镜像，新 handle 去新事务完全无妨；fc（以及 resize / swap_boot 等）需要在**当前运行事务的稳定状态**上做操作，因此要求更严格的"全局静止"。

---

## 8. 与上一轮 `ext4_fc_flush_data` 的衔接

把 Step 2（flush data）和 Step 4（lock updates）连起来看，顺序是有讲究的：

1. **Step 1**：置 `EXT4_STATE_FC_FLUSHING_DATA`，防 flush 期间 inode 被 evict。
2. **Step 2**：`ext4_fc_flush_data()` → `jbd2_submit_inode_data` 把脏页推到磁盘（**不持 barrier**，因为 flush 期间仍允许其他进程写数据，只是要保证数据先落盘）。
3. **Step 3**：清 `FLUSHING_DATA` + `smp_mb()`，唤醒等待者。
4. **Step 4**：`jbd2_journal_lock_updates()` —— 现在才建立全局静止，置 `FC_COMMITTING`。
5. **Step 6+**：在静止态 + `FC_COMMITTING` 位保护下，写 HEAD/dentry/inode/extent/TAIL tag。

即：先让**数据**就位（Step 2，可并发），再让 **journal 静止**（Step 4，排他），最后才写**元数据 tag**。这正是 `data=ordered` 语义在 fast commit 路径上的体现。

---

## 9. 一句话总结

`jbd2_journal_lock_updates()` 是 JBD2 的"全局静止"原语：通过 `j_barrier_count++` 标志让新普通 handle 在 `j_wait_transaction_locked` 上睡（reserved handle 例外以免死锁）、等待 `j_reserved_credits` 与 `t_updates` 全部归零排空运行中的 handle、再用 `j_barrier` 互斥锁串行化多个 lock 调用者；`unlock` 时 `--j_barrier_count` + `wake_up_all` 解除屏障。在 fast commit 里它用于 Step 4——在写 extent tag 前让 journal 进入静止态，保证序列化的映射快照与运行期修改不冲突；而 full commit 只需 `wait_updates` 等当前事务排空、不阻止新事务，二者静止严格度不同。
