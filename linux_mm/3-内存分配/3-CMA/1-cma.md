**编译内核时需要开启以下配置宏：**

（1）配置宏CONFIG_CMA，启用连续内存分配器。

（2）配置宏CONFIG_CMA_AREAS，指定CMA区域的最大数量，默认值是7。

（3）配置宏CONFIG_DMA_CMA，启用允许设备驱动分配内存的连续内存分配器。

CMA区域分为全局CMA区域和设备私有CMA区域。全局CMA区域是由所有设备驱动共享的，设备私有CMA区域由指定的一个或多个设备驱动使用。

<mark>**配置CMA区域有3种方法：**</mark>

**（1）通过内核参数“cma”配置全局CMA区域的大小。**

使用内核参数“cma=nn[MG]@[start[MG][-end[MG]]]”设置全局CMA区域的大小和物理地址范围。

**（2）通过配置宏配置全局CMA区域的大小。**

首先选择指定大小的方式：CONFIG_CMA_SIZE_SEL_MBYTES表示指定兆字节数，CONFIG_CMA_SIZE_SEL_PERCENTAGE表示指定物理内存容量的百分比，默认使用指定兆字节数的方式。

如果选择指定兆字节数的方式，那么通过配置宏CONFIG_CMA_SIZE_MBYTES配置大小。如果配置为0，表示禁止CMA，但是可以传递内核参数“cma=nn[MG]”以启用CMA。

如果选择指定物理内存容量的百分比的方式，那么通过配置宏CONFIG_CMA_SIZE_PERCENTAGE指定百分比。如果配置为0，表示禁用连续内存分配器，但是可以传递内核参数“cma=nn[MG]”以启用连续内存分配器。

**（3）通过设备树源文件的节点“/reserved-memory”配置CMA区域，如果子节点的属性“compatible”的值是“shared-dma-pool”，表示全局CMA区域，否则表示设备私有CMA区域。**

例如配置3个CMA区域：

1）全局CMA区域，节点名称是“linux,cma”，大小是64MB。

2）帧缓冲设备专用的CMA区域，节点名称是“framebuffer@78000000”，大小是8MB。

3）多媒体处理专用的CMA区域，节点名称是“multimedia-memory@77000000”，大小是64MB。

**问题1：确定CMA内存所在的zone（问问夏老板）**

1，内核启动时会根据虚拟地址的划分会确定物理地址的zone_normal和zone_highmem，并初始化到struct page（后续研究怎么初始化）

2，设备树中指出CMA的物理地址，由此在CMA初始化时候，可以确定CMA内存添加到哪个zone中

3，内核allocpage申请内存时，将从首选zone中分配页面，若MIGRATE_MOVABLE页面不足，则从MIGRATE_CMA中分配，

4，如还分配不出来，进而寻找下个zone中的MIGRATE_MOVABLE和MIGRATE_CMA页面
