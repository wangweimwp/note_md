“迭代器” 是内核中常见的设计，通常用来描述一个对象的处理进度。 iov_iter最初主要用于描述一次IO流程中用户空间的处理进度，以*iov保存用户空间的内存地址，iov_offset和count记录当前处理进度，这两个参数会随IO的进行会不断变化。随后该机制拓展到内核其他功能中，以union形式定义了更多属性。

```c

struct kvec {
	void *iov_base; /* and that should *never* hold a userland pointer */
	size_t iov_len;
};
 
struct iov_iter {
	/*
	 * Bit 0 is the read/write bit, set if we're writing.
	 * Bit 1 is the BVEC_FLAG_NO_REF bit, set if type is a bvec and
	 * the caller isn't expecting to drop a page reference when done.
	 */
	unsigned int type;    //标识读or写，以及其他属性 
	size_t iov_offset;    //第一个iovec中，数据起始偏移
	size_t count;    //数据大小
	union {
		const struct iovec *iov;    //结构与kvec一致，描述用户态的一段空间
		const struct kvec *kvec;    //描述内核态的一段空间
		const struct bio_vec *bvec;    //描述一个内存页中的一段空间
		struct pipe_inode_info *pipe;
	};
	union {
		unsigned long nr_segs;    //iovec数量
		struct {
			int idx;
			int start_idx;
		};
	};

}
```



```c
struct kiocb {
	struct file		*ki_filp;    //open文件创建的file结构
 
	/* The 'ki_filp' pointer is shared in a union for aio */
	randomized_struct_fields_start
 
	loff_t			ki_pos;    //数据偏移
	void (*ki_complete)(struct kiocb *iocb, long ret, long ret2);    //IO完成回调
	void			*private;
	int			ki_flags;    //IO属性
	u16			ki_hint;
	u16			ki_ioprio; /* See linux/ioprio.h */
	unsigned int		ki_cookie; /* for ->iopoll */
 
	randomized_struct_fields_end
}
```

kiocb 中主要保存了一个file结构，以及记录读写偏移，相当于描述了一次IO中文件侧的处理进度。

iov_iter 和 kiocb 实际上分别描述了一次IO的两端，iov_iter描述内存侧，kiocb描述文件侧，文件系统提供两个接口基于这两个数据结构封装读写操作。
