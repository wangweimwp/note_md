
# 1，rename测试

## 1,1测试脚本

```shell
dd if=/dev/zero of=/tmp/ext4.img bs=1M count=100000
mkfs.ext4 -O fast_commit /tmp/ext4.img
mount -o loop /tmp/ext4.img /mnt/ext4
cd /mnt/ext4

#然和建立个文件test1，用mv重命令为test2
touch test1
mv test1 test2

#sync这个文件
sync test2

#此时可以观察到有fast commit提交记录
cat /proc/fs/ext4/loop10/fc_info
```
## 1,2原理分析

### 1,2,1 sync与sync test2

* sync是同步目录，sync test2是同步文件
* 执行sync 内核在`ext4_fsync_journal -> if (!S_ISREG(inode->i_mode))`判断成立，走`ext4_force_commit`,强行full commit

### 1,2,2 5秒kjournald2周期提交

* `jbd2_journal_commit_transaction()`函数开头
```c
journal->j_flags |= JBD2_FULL_COMMIT_ONGOING;                 // :409
while (journal->j_flags & JBD2_FAST_COMMIT_ONGOING) { ... }   // :等正在进行的 fc 结束
...
journal->j_fc_off = 0;                                        // :重置 fc 区域偏移

```
看出kjournald2周期提交强制走full commit

### 1,2,3 mv命令后，为什么观察到ext4_sync_file的调用

mv命令后，若不发送sync test2命令，会触发5秒kjournald2周期提交，强制走full commit，但是mv命令过后，观察到有`ext4_sync_file`的调用。抓取函数内核栈后发现是loop设备刷盘导致`ext4_sync_file`调用

```
->clone() by kworker/u512:0 PID = 171589 ret = 0
        vfs_fsync+61
        loop_process_work+723
        loop_rootcg_workfn+27
        process_one_work+431
        worker_thread+447
        kthread+251
        ret_from_fork+698
        ret_from_fork_asm+26

```