# 结构体

```c
alloc_hooks(alloc_pages_bulk_noprof(__VA_ARGS__))

#define alloc_hooks(_do_alloc)						\
({									\
	DEFINE_ALLOC_TAG(_alloc_tag);					\
	alloc_hooks_tag(&_alloc_tag, _do_alloc);			\
})

#define DEFINE_ALLOC_TAG(_alloc_tag)						\
	static struct alloc_tag _alloc_tag __used __aligned(8)			\
	__section(ALLOC_TAG_SECTION_NAME) = {					\
		.ct = CODE_TAG_INIT,						\
		.counters = &_shared_alloc_tag };
		

	static struct alloc_tag _alloc_tag  = {					\
		.ct = CODE_TAG_INIT,						\
		.counters = &_shared_alloc_tag };

{									\
	typeof(alloc_pages_bulk_noprof) _res;						\
	if (mem_alloc_profiling_enabled()) {				\
		struct alloc_tag * __maybe_unused _old;			\
		_old = alloc_tag_save(_alloc_tag);				\
		_res = alloc_pages_bulk_noprof;					\
		alloc_tag_restore(_alloc_tag, _old);				\
	} else								\
		_res = alloc_pages_bulk_noprof;					\
	_res;								\
}		

/* SPDX-License-Identifier: GPL-2.0-only */
	#ifndef __ASM_GENERIC_CODETAG_LDS_H
	#define __ASM_GENERIC_CODETAG_LDS_H
	
	#define SECTION_WITH_BOUNDARIES(_name)  \
	        . = ALIGN(8);                   \
	        __start_##_name = .;            \
	        KEEP(*(_name))                  \
	        __stop_##_name = .;
	
	#define CODETAG_SECTIONS()              \
	        SECTION_WITH_BOUNDARIES(alloc_tags)
	
	#endif /* __ASM_GENERIC_CODETAG_LDS_H */		
	

struct codetag_type {
	struct list_head link;		//所有codetag_type在一个列表上
	unsigned int count;
	struct idr mod_idr;
	struct rw_semaphore mod_lock; /* protects mod_idr */
	struct codetag_type_desc desc;	//codetag描述符
};

struct codetag_type_desc {
	const char *section;//codetag名称
	size_t tag_size;
	void (*module_load)(struct codetag_type *cttype,
			    struct codetag_module *cmod);
	void (*module_unload)(struct codetag_type *cttype,
			      struct codetag_module *cmod);
#ifdef CONFIG_MODULES
	void (*module_replaced)(struct module *mod, struct module *new_mod);
	bool (*needs_section_mem)(struct module *mod, unsigned long size);
	void *(*alloc_section_mem)(struct module *mod, unsigned long size,
				   unsigned int prepend, unsigned long align);
	void (*free_section_mem)(struct module *mod, bool used);
#endif
}
__start_alloc_tags

__stop_alloc_tags
```


# 初始化代码
```c
static struct codetag_type *alloc_tag_cttype

static int __init alloc_tag_init(void)
{
	const struct codetag_type_desc desc = {//codetag描述符
		.section		= ALLOC_TAG_SECTION_NAME,//"alloc_tags"
		.tag_size		= sizeof(struct alloc_tag),
#ifdef CONFIG_MODULES
		.needs_section_mem	= needs_section_mem,
		.alloc_section_mem	= reserve_module_tags,
		.free_section_mem	= release_module_tags,
		.module_replaced	= replace_module,
#endif
	};
	int res;
	
	//为vm_module_tags全局变量申请struct vm_struct虚拟内存空间，
	//同时分配物理页面
	//虚拟空间大小 (100000UL * sizeof(struct alloc_tag))
	//只申请了vmemp虚拟地址空间，没有做映射
	res = alloc_mod_tags_mem();
	if (res)
		return res;
	//注册全局alloc_tag_cttype
	//申请个struct codetag_type，这里包含struct codetag_type_desc
	//把申请的struct codetag_type挂到codetag_types全局链表里
	alloc_tag_cttype = codetag_register_type(&desc);
	if (IS_ERR(alloc_tag_cttype)) {
		free_mod_tags_mem();
		return PTR_ERR(alloc_tag_cttype);
	}

	sysctl_init();
	procfs_init();

	return 0;
}

struct codetag_type *
codetag_register_type(const struct codetag_type_desc *desc)
{
	struct codetag_type *cttype;
	int err;

	BUG_ON(desc->tag_size <= 0);

	cttype = kzalloc(sizeof(*cttype), GFP_KERNEL);
	if (unlikely(!cttype))
		return ERR_PTR(-ENOMEM);

	cttype->desc = *desc;
	idr_init(&cttype->mod_idr);//IDR机制，整数和指针做关联
	init_rwsem(&cttype->mod_lock);

	//初始化一个codetag_module结构体，加入到struct codetag_type的IDR中
	//获取alloc_tags段的开始（__start_alloc_tags）和结束（__stop_alloc_tags）地址
	//填入codetag_module结构体中，
	err = codetag_module_init(cttype, NULL);
	if (unlikely(err)) {
		kfree(cttype);
		return ERR_PTR(err);
	}

	mutex_lock(&codetag_lock);
	list_add_tail(&cttype->link, &codetag_types);//挂到codetag_types全局链表里
	mutex_unlock(&codetag_lock);

	return cttype;
}

//驱动加载时
load_module
	->codetag_load_module
	
void codetag_load_module(struct module *mod)
{
	struct codetag_type *cttype;

	if (!mod)
		return;

	mutex_lock(&codetag_lock);
	
	//初始化一个codetag_module结构体，加入到struct codetag_type的IDR中
	//在给定的mod中寻找alloc_tags段的开始（__start_alloc_tags）和结束（__stop_alloc_tags）地址
	//见scripts/module.lds.S
	//遍历codetag_types全局列表，对于每个codetag_types成员都为codetag_module申请一个IDR
	list_for_each_entry(cttype, &codetag_types, link)
		codetag_module_init(cttype, mod);
	mutex_unlock(&codetag_lock);
}
```