```c
//在使能mglru后，页面不在添加到lruvec->lists，而是添加到lruvec->lrugen->folios列表
void lruvec_add_folio(struct lruvec *lruvec, struct folio *folio)
{
    enum lru_list lru = folio_lru_list(folio);

    if (lru_gen_add_folio(lruvec, folio, false))
        return;

    update_lru_size(lruvec, lru, folio_zonenum(folio),
            folio_nr_pages(folio));
    if (lru != LRU_UNEVICTABLE)
        list_add(&folio->lru, &lruvec->lists[lru]);
}
```
