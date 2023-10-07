```c
folio_alloc
    ->__folio_alloc_node
        ->__folio_alloc
```

![](C:\Users\wangwei180\AppData\Roaming\marktext\images\2023-10-07-17-04-08-image.png)

```c
alloc_pages
    ->alloc_pages_node
        ->__alloc_pages
            ->get_page_from_freelist
                ->prep_new_page
                    ->prep_compound_page //设置folio->_folio_order


void prep_transhuge_page(struct page *page)
{
    struct folio *folio = (struct folio *)page;

    VM_BUG_ON_FOLIO(folio_order(folio) < 2, folio);
    INIT_LIST_HEAD(&folio->_deferred_list);
    set_compound_page_dtor(page, TRANSHUGE_PAGE_DTOR);
}

static inline void set_compound_page_dtor(struct page *page,
        enum compound_dtor_id compound_dtor)
{
    struct folio *folio = (struct folio *)page;

    VM_BUG_ON_PAGE(compound_dtor >= NR_COMPOUND_DTORS, page);
    VM_BUG_ON_PAGE(!PageHead(page), page);
    folio->_folio_dtor = compound_dtor;
}

static inline void set_compound_page_dtor(struct page *page,
        enum compound_dtor_id compound_dtor)
{
    struct folio *folio = (struct folio *)page;

    VM_BUG_ON_PAGE(compound_dtor >= NR_COMPOUND_DTORS, page);
    VM_BUG_ON_PAGE(!PageHead(page), page);
    folio->_folio_dtor = compound_dtor;
}
```
