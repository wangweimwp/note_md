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



//两个关键结构体
//表明申请页面的路径
struct codetag {
	unsigned int flags; /* used in later patches */
	unsigned int lineno;
	const char *modname;
	const char *function;
	const char *filename;
} __aligned(8);

union codetag_ref {
	struct codetag *ct;
};


//不仅包含了页面申请路径还包含了申请次数和内存大小
struct alloc_tag {
	struct codetag			ct;
	struct alloc_tag_counters __percpu	*counters;
} __aligned(8);



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


## 申请页面时
```c
申请page时
get_page_from_freelist
	->prep_new_page
		->post_alloc_hook
			->pgalloc_tag_add
			

static inline void pgalloc_tag_add(struct page *page, struct task_struct *task,
				   unsigned int nr)
{
	if (mem_alloc_profiling_enabled()) {
		union pgtag_ref_handle handle;
		union codetag_ref ref;

		if (get_page_tag_ref(page, &ref, &handle)) {//拿到这个page的struct codetag结构体放到ref中
			//增加struct alloc_tag的调用次数和字节数 并把 alloc_tag->ct赋值给ref->ct
			alloc_tag_add(&ref, task->alloc_tag, PAGE_SIZE * nr);
			update_page_tag_ref(handle, &ref);//更新page->flag中的idx（偏移值）指向新的struct codetag（struct codetag表明了page的申请路劲）。
			put_page_tag_ref(handle);
		}
	}
}

/* Should be called only if mem_alloc_profiling_enabled() */
static inline bool get_page_tag_ref(struct page *page, union codetag_ref *ref,
				    union pgtag_ref_handle *handle)
{
	if (!page)
		return false;

	if (static_key_enabled(&mem_profiling_compressed)) {
		pgalloc_tag_idx idx;
		/*
		Page flags: | [SECTION] | [NODE] | ZONE | [LAST_CPUPID] | ... | FLAGS | 
		从page flag里存放了struct codetag的偏移值，拿到idx
		*/
		idx = (page->flags >> alloc_tag_ref_offs) & alloc_tag_ref_mask;
		idx_to_ref(idx, ref);//根据idx（偏移值）从__start_alloc_tags段中找到对应的struct codetag结构体
		handle->page = page;
	} else {
		struct page_ext *page_ext;
		union codetag_ref *tmp;

		page_ext = page_ext_get(page);
		if (!page_ext)
			return false;

		tmp = (union codetag_ref *)page_ext_data(page_ext, &page_alloc_tagging_ops);
		ref->ct = tmp->ct;
		handle->ref = tmp;
	}

	return true;
}

static inline void update_page_tag_ref(union pgtag_ref_handle handle, union codetag_ref *ref)
{
	if (static_key_enabled(&mem_profiling_compressed)) {
		struct page *page = handle.page;
		unsigned long old_flags;
		unsigned long flags;
		unsigned long idx;

		if (WARN_ON(!page || !ref))
			return;

		idx = (unsigned long)ref_to_idx(ref);//根据struct codetag在__start_alloc_tags段中的偏移值计算idx
		idx = (idx & alloc_tag_ref_mask) << alloc_tag_ref_offs;
		do {
			old_flags = READ_ONCE(page->flags);
			flags = old_flags;
			flags &= ~(alloc_tag_ref_mask << alloc_tag_ref_offs);
			flags |= idx;//将新的idx放到page->flags中
		} while (unlikely(!try_cmpxchg(&page->flags, &old_flags, flags)));
	} else {
		if (WARN_ON(!handle.ref || !ref))
			return;

		handle.ref->ct = ref->ct;
	}
}
```


## 打印内存申请信息时
```c
struct codetag *codetag_next_ct(struct codetag_iterator *iter)
{
	struct codetag_type *cttype = iter->cttype;
	struct codetag_module *cmod;
	struct codetag *ct;

	lockdep_assert_held(&cttype->mod_lock);

	if (unlikely(idr_is_empty(&cttype->mod_idr)))
		return NULL;

	ct = NULL;
	while (true) {
        /*
        cttype上挂了各个驱动和内核本身的codetag_type_desc
        每个codetag_type_desc有包含了alloc_tag段的起始结束地址，先通过mod_id拿到一个模块的codetag_type_desc
        */
		cmod = idr_find(&cttype->mod_idr, iter->mod_id);

		/* If module was removed move to the next one */
		if (!cmod)
			cmod = idr_get_next_ul(&cttype->mod_idr,
					       &iter->mod_id);

		/* Exit if no more modules */
		if (!cmod)
			break;

		if (cmod != iter->cmod) {
			iter->cmod = cmod;
			ct = get_first_module_ct(cmod);
		} else
			ct = get_next_module_ct(iter);//然后遍历这个codetag_type_desc上的每个codetag

		if (ct)
			break;

		iter->mod_id++;//若ct==NULL说明这个模块遍历完了。mod_id++遍历下个模块
	}

	iter->ct = ct;
	return ct;
}
```