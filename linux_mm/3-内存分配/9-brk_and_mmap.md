# Linux 进程内存管理:brk、堆区与 mmap 全解析

> 本文整理自对以下问题的逐步探讨:
> 1. `mm_struct` 中的 `brk_start`(实为 `start_brk`)是做什么的?
> 2. 进程申请的虚拟地址是否都来自堆区?
> 3. `brk` 下方是否会出现空洞?glibc 如何处理?
> 4. `brk` 与 `mmap` 在内核中的管理方式是否不同?

---

## 目录

- [1. mm_struct 中的 start_brk:堆区的界定](#1-mm_struct-中的-start_brk堆区的界定)
- [2. 进程申请的虚拟地址来源:并非都来自堆](#2-进程申请的虚拟地址来源并非都来自堆)
- [3. 堆区空洞与 glibc (ptmalloc2) 的处理](#3-堆区空洞与-glibc-ptmalloc2-的处理)
- [4. 内核视角:brk 与 mmap 的管理差异](#4-内核视角brk-与-mmap-的管理差异)
- [5. 总结](#5-总结)

---

## 1. mm_struct 中的 start_brk:堆区的界定

### 1.1 名称纠正

Linux 内核里**没有 `brk_start` 这个字段**,正确名称是 **`start_brk`**(口语或文档里常被误称为 `brk_start`)。它与 `brk` 一起描述进程的**堆(heap)区间**。

### 1.2 相关字段(`include/linux/mm_types.h`)

```c
struct mm_struct {
    ...
    unsigned long start_code, end_code;   /* 代码段 (.text) 范围 */
    unsigned long start_data, end_data;   /* 已初始化数据段 (.data) */
    unsigned long start_brk, brk;         /* 堆区:start_brk 是底,brk 是顶 */
    unsigned long start_stack;            /* 栈起始(向下增长) */
    unsigned long arg_start, arg_end;     /* 命令行参数 */
    unsigned long env_start, env_end;     /* 环境变量 */
    unsigned long mmap_base;              /* mmap 区基地址 */
    ...
};
```

### 1.3 start_brk 与 brk 的区别

| 字段 | 含义 | 是否变化 |
|---|---|---|
| `start_brk` | 堆的**初始底部**,即 BSS 段末尾地址 | **固定不变**(进程加载时设定) |
| `brk` | 堆的**当前顶部** | **随内存分配向上移动** |

- 当前堆大小 = `brk - start_brk`。
- 进程加载时,`start_brk` 设为 BSS 段末尾,`brk` 初始等于 `start_brk`(堆大小为 0)。
- `brk()` / `sbrk()` 系统调用改变的是 `brk`,从而扩展或收缩堆;`start_brk` 永远不变,是堆的**下界**。

### 1.4 作用

`start_brk` 是进程堆区的固定起点。内核在 `sys_brk` 中用它作为合法性检查的下界,防止堆被收缩到 BSS/数据段以内。

---

## 2. 进程申请的虚拟地址来源:并非都来自堆

进程虚拟地址空间被划分为多个区,**地址来源取决于分配方式**:

| 分配方式 | 来自哪个区 | 地址机制 |
|---|---|---|
| 代码/全局/静态变量 | `.text` / `.data` / `.bss` | 由 ELF 文件布局决定,加载时固定 |
| 小内存 `malloc` | **堆区 (heap)** | 在 `start_brk`~`brk` 之间,靠 `brk()` 向上扩展 |
| 大内存 `malloc`(glibc 默认 ≥128KB) | **mmap 区** | 匿名映射,从 `mmap_base` 向下分配 |
| 显式 `mmap()` | **mmap 区** | 从 `mmap_base` 向下分配 |
| 局部变量 | **栈 (stack)** | `start_stack` 往下增长,地址由内核加载时设定 |
| 共享库 | mmap 区 | 动态链接器 `mmap` 载入 |

**关键结论**:
- 只有 `malloc` 申请的**小块**内存走堆区(`start_brk`~`brk`)。
- `malloc` 本身不全走堆:超过阈值的大块改用 `mmap` 匿名映射,落在 mmap 区,与堆无关(释放时直接 `munmap`,避免堆碎片)。
- 栈、代码段根本不碰堆,它们的地址在进程创建/加载时由内核和 ELF 决定。
- 堆、栈、mmap 区的地址范围相互隔离、各自管理,**不能代表整个进程虚拟地址来源**。

---

## 3. 堆区空洞与 glibc (ptmalloc2) 的处理

### 3.1 前提纠正

> 误区:"随着内存申请,`start_brk` 不断向上扩展。"
> 事实:**`start_brk` 固定不动,真正向上扩展的是 `brk`(堆顶)。**

用户补充正确:**空洞出现在 `brk` 地址下方、即堆区间内部**。glibc 的 `malloc`(内部 ptmalloc2)不直接把这片空间还给内核,而是登记为「空闲块(free chunk)」,用 **bin(箱子)+ 合并(coalescing)** 机制管理,供后续分配复用。

### 3.2 空洞的产生

- **堆顶**那块被释放时,glibc 可调用 `brk()` 把 `brk` **向下收缩**,空间真正归还内核。
- **堆中间**某块被释放、其下仍有未释放内存时,`brk` **无法向下收缩**(`brk` 只能移动连续堆顶),中间那块成为**空洞(hole)**。
- 空洞不消失、不还内核,由 malloc 用**空闲链表(bins)**记录,等下次有合适大小的 `malloc` 请求时**直接复用**。这就是经典的堆碎片/内部碎片。

```text
start_brk  ────────────────────────────────  (固定不动)
           [已用A][空洞B(已free)][已用C]...[brk]
                                 ↑
                          brk 不能越过 C 往下缩
                          所以 B 是永久空洞,靠 free list 复用
```

### 3.3 chunk 头与「空闲」判定

```c
struct malloc_chunk {
    size_t prev_size;   /* 仅当上一块空闲时,存上一块大小 */
    size_t size;        /* 低 3 位为标志:A=NON_MAIN_ARENA, M=IS_MMAPPED, P=PREV_INUSE */
    /* 用户数据区;若空闲则存放 fd / bk 等链表指针 */
};
```

- **`P`(PREV_INUSE)位**:写在「当前块」的 `size` 中,表示**紧邻的前一块**是否在使用中 → 向后合并依据。
- **下一块定位**:`下一块 = 当前地址 + size`,再看它的 `P` 位 → 向前合并依据。

### 3.4 free 时的两条路径

- **路径 A(小块)→ fastbin**:小于 `M_MXFAST`(64 位典型约 64~128 字节)的块,free 时**不合并**,直接挂到对应 size 的 fastbin 单向链表头部(LIFO)。最快,但暂不参与合并。
- **路径 B(其他块)→ 合并 → unsorted bin**:先尝试与前后相邻空闲块合并,合并结果放入 unsorted bin(中转站);若合并后的块紧贴堆顶(top chunk),则直接并入 top chunk。

### 3.5 合并算法 (coalescing)

```
释放块 P:
1) 向前合并:取下一块 N = P + P->size
   - 若 N 的 P 位 == 0(N 空闲) → 把 N 从 bin 摘下,N 与 P 合并
2) 向后合并:看 P 自己的 P 位
   - 若 P 的 P 位 == 0(上一块空闲) → 上一块 = P - P->prev_size
     把上一块从 bin 摘下,与 P 合并
3) 合并后的大块:
   - 若与 top chunk 相邻 → 并入 top
   - 否则 → 放进 unsorted bin
```

> fastbin 里的块在「被取出时」会被强制合并,所以不会长期堆积可合并的相邻块。

### 3.6 四类 bin 的职责

| bin 类型 | 存放内容 | 结构 | 特点 |
|---|---|---|---|
| **fastbins** | 很小的刚释放块(~64–128B) | 单向链表 LIFO | 不合并、最快,malloc 优先查 |
| **unsorted bin** | 刚合并/刚释放、未归类的块 | 双向循环链表 | 中转站,下次 malloc 时细分归类 |
| **small bins** | 小块(< ~1024B),每 16B 一桶 | 双向循环链表 FIFO | 同尺寸精确匹配,不分割 |
| **large bins** | 大块(≥ ~1024B),按尺寸区间分桶 | 双向链表 + `fd_nextsize/bk_nextsize` | 桶内按大小排序,分配时可**分割** |

另有 **last remainder** 机制:上次从大块切下一块后的剩余部分,专供后续「切小块」复用。

### 3.7 malloc 时如何复用空洞

```
malloc(sz):
 1. 查 fastbins:有对应尺寸小空闲块 → 直接返回
 2. 查 small bins:精确尺寸匹配 → 返回(FIFO)
 3. 扫描 unsorted bin:正好合适直接用,否则按尺寸塞进 small/large bin
 4. 查 large bins:找 ≥ sz 的最小块,分割,剩余放回 bin/last remainder
 5. 以上都没命中 → 从 top chunk 切(不够就 brk/sbrk 或 mmap 扩堆)
 6. 超大请求(> M_MMAP_THRESHOLD,默认 ~128KB) → 直接 mmap 单独映射
```

### 3.8 为什么 brk 下方的空洞回不到 OS

- **只有紧贴 `brk` 的空闲块(top chunk 顶部)** 能被归还:当顶端空闲超过 `M_TRIM_THRESHOLD`(默认 128KB),glibc 在 `free`/`malloc_trim()` 时调用 `brk()` 把 `brk` 向下缩。
- **`brk` 与某仍在使用的块之间的空洞**(即 brk 下方空洞)**永远无法用 `brk` 收缩归还**——`brk` 只能移动连续堆顶。这些空洞**一直留在 bin 里**,被本进程复用,但内核视角下仍计入该进程 RSS。
- 极少数情况下,glibc 会对大块空闲调用 `madvise(MADV_DONTNEED)` 让 OS 回收物理页,但普通中部小空洞不触发。
- **mmap 大块是例外**:不在堆里,free 时整块 `munmap` 立刻归还 OS,干净无空洞。

```text
start_brk ──────────────────────────────────── brk
           [用][空洞B][用][空洞D][用]...[top空闲]
              ↑B、D 在 brk 下方:留在 bin 复用,绝不还给 OS
                              ↑ top 空闲够大时,brk 才能往这儿缩,把顶端还给 OS
```

---

## 4. 内核视角:brk 与 mmap 的管理差异

**结论:不一样,且是两套内核机制。** `brk` 操作一个专用、连续的堆 VMA(由 `mm->start_brk`/`mm->brk` 描述);`mmap`(通常)每次创建独立 VMA,散落于地址空间任意空闲处。

### 4.1 总体对比

| 维度 | `brk` / `sbrk` | `mmap` |
|---|---|---|
| 系统调用 | `sys_brk(unsigned long brk)` | `sys_mmap` → `do_mmap` |
| 核心辅助函数 | `do_brk_flags()`(扩展堆) / `__do_munmap()`(收缩) | `mmap_region()`(通用映射) |
| VMA 模型 | **单个**专用堆 VMA(`[heap]`) | (通常)每次新建独立 VMA |
| 地址来源 | 只能在 `[start_brk, 向上]` 连续扩展 | `vm_unmapped_area()` 在任意空闲 gap(默认从 `mmap_base` 向下) |
| 连续性 | 必须连续,不能跳过 | 可分散、互不连续 |
| 下界 | 锁死在 `start_brk` | 无,任意空闲处 |
| 资源限制 | 受 **`RLIMIT_DATA`** 约束 | 受 **`RLIMIT_AS`** + overcommit 约束(不受 `RLIMIT_DATA` 限制) |
| 后端 | 仅匿名、私有、可写 | 文件/设备/共享内存/匿名 均可 |
| 释放 | 只能缩堆顶(`__do_munmap` 砍尾部) | 可释放任意片段(可把原 VMA 分裂) |

### 4.2 VMA 模型:一个 vs 一堆

- **brk**:进程地址空间里只有**一个堆 VMA**(在 `/proc/<pid>/maps` 显示 `[heap]`),范围 `start_brk`~`brk`。增长时 `do_brk_flags()` 用 `vma_merge()` 把该 VMA 向上延长;收缩时 `__do_munmap()` 砍尾部。始终同一块连续区域。
- **mmap**:每次映射(通常)新建 `vm_area_struct`,由 `mmap_region()` 在红黑树(`mm_rb`)里插入;仅当与相邻 VMA 权限/标志/后端完全一致时才 `vma_merge()`。故可见大量独立匿名映射段。

### 4.3 地址布局与相撞风险

经典布局:
```text
text | data | bss | [heap] ↑ ... ↓ [mmap] | stack
```
堆向上长、mmap 向下长,中间为空闲带,两者可能相撞(故引入 ASLR 随机化 `mmap_base`、`mmap_min_addr` 等防护)。

### 4.4 资源限制差异(很实用)

- `brk` 受 **`RLIMIT_DATA`**(数据段上限)限制。很多容器/沙箱把 `RLIMIT_DATA` 设得很小。
- 匿名 `mmap` 受 **`RLIMIT_AS`**(虚拟地址空间上限)与 overcommit 约束,**不受 `RLIMIT_DATA` 限制**。
- → 实际后果:同一程序在 `RLIMIT_DATA` 很小的环境里 `brk` 堆扩不动,但 `mmap` 还能分配。glibc `malloc` 正是利用这点:堆扩不动时自动改用 `mmap`(这也是「free 不掉内存时,`/proc/pid/maps` 里是一堆 `anon` 而非 `[heap]`」的原因)。

### 4.5 后端不同

- `do_brk_flags()` 只能做匿名、私有、可写映射;页面首次访问时由缺页异常处理程序分配**零页**(可换出到 swap)。
- `mmap` 通过 `vm_ops->fault()` 支持文件页缓存、设备驱动、共享内存等多种后端。

### 4.6 共同点(别夸大差异)

- **都是懒惰分配**:调用本身不立即分配物理内存,只建 VMA/页表,页面在**首次访问缺页时**才分配。
- **都计入 `mm->total_vm`、RSS,受 overcommit 检查**(`__vm_enough_memory`)。
- 匿名子集上,`brk` 堆 ≈ 一块连续的匿名 `mmap`,差异主要在**管理方式(VMA 数量/连续/限制)**而非内存性质。

---

## 5. 总结

1. **`start_brk`**(非 `brk_start`)是进程堆区的固定下界,加载时设定后不变;`brk` 是堆顶,随分配移动。
2. **进程虚拟地址并非都来自堆**:代码/数据/BSS 由 ELF 决定,小块 `malloc` 走堆,大块 `malloc` 与 `mmap` 走 mmap 区,局部变量走栈。
3. **`brk` 下方会产生空洞**(堆中间释放且下方仍有占用时)。glibc 用 fastbins / unsorted / small / large 四类 bin + 合并(coalescing)管理:相邻空闲块合并成更大块,分配时按尺寸匹配复用(大块可再分割)。**中间空洞只在进程内循环复用,无法用 `brk` 收缩还给内核**;只有堆顶空闲能经 `brk`/`malloc_trim` 真正归还 OS。
4. **`brk` 与 `mmap` 内核管理不同**:`brk` 是单 VMA 的专用堆、只能连续向上扩展、受 `RLIMIT_DATA` 约束、释放只能缩堆顶;`mmap` 是通用映射引擎、每次(通常)建独立 VMA、可落在任意空闲处、不受 `RLIMIT_DATA` 限制、可释放任意片段。

> 实践建议:排查「free 后内存不释放 / RSS 不降」时,用 `cat /proc/<pid>/maps` 和 `cat /proc/<pid>/smaps` 区分是 `[heap]`(堆碎片)还是大量 `anon` mmap;必要时显式调用 `malloc_trim(0)` 或在 `RLIMIT_DATA` 受限环境依赖 mmap 分配器。
