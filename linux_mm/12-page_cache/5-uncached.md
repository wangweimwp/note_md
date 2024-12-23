补丁邮件
https://lore.kernel.org/all/20241220154831.1086649-1-axboe@kernel.dk/

将uncached特性与refault distance特性相结合
```c
diff --git a/mm/workingset.c b/mm/workingset.c
index ba2b12df9..7070ef5ba 100644
--- a/mm/workingset.c
+++ b/mm/workingset.c
@@ -567,10 +567,13 @@ void workingset_refault(struct folio *folio, void *shadow)
        mod_lruvec_state(lruvec, WORKINGSET_REFAULT_BASE + file, nr);

        if (!workingset_test_recent(shadow, file, &workingset, true)){
-               __folio_set_uncached(folio);
+               if(file && folio->mapping){
+                       printk("auto set uncached\n");
+                       folio_set_uncached(folio);
+               }
                return;
        }
-
+       printk("active\n");
        folio_set_active(folio);
        workingset_age_nonresident(lruvec, nr);
        mod_lruvec_state(lruvec, WORKINGSET_ACTIVATE_BASE + file, nr);

diff --git a/include/linux/pagemap.h b/include/linux/pagemap.h
index 374872acb..946eb4c94 100644
--- a/include/linux/pagemap.h
+++ b/include/linux/pagemap.h
@@ -36,7 +36,7 @@ void kiocb_invalidate_post_direct_write(struct kiocb *iocb, size_t count);
 int filemap_invalidate_pages(struct address_space *mapping,
                             loff_t pos, loff_t end, bool nowait);
 int folio_unmap_invalidate(struct address_space *mapping, struct folio *folio,
-                          gfp_t gfp);
+                          gfp_t gfp, void *shadow);

 int write_inode_now(struct inode *, int sync);
 int filemap_fdatawrite(struct address_space *);
diff --git a/mm/filemap.c b/mm/filemap.c
index a03a9b912..6fa59ab83 100644
--- a/mm/filemap.c
+++ b/mm/filemap.c
@@ -1633,7 +1633,7 @@ static void folio_end_uncached(struct folio *folio)
         */
        if (in_task() && folio_trylock(folio)) {
                if (folio->mapping)
-                       folio_unmap_invalidate(folio->mapping, folio, 0);
+                       folio_unmap_invalidate(folio->mapping, folio, 0, NULL);
                folio_unlock(folio);
        }
 }
@@ -2652,15 +2652,20 @@ static inline bool pos_same_folio(loff_t pos1, loff_t pos2, struct folio *folio)
 }

 static void filemap_uncached_read(struct address_space *mapping,
-                                 struct folio *folio)
+                                 struct folio *folio, int ki_flags)
 {
+       void *shadow = NULL;
+
        if (!folio_test_uncached(folio))
                return;
        if (folio_test_writeback(folio) || folio_test_dirty(folio))
                return;
        if (folio_trylock(folio)) {
                if (folio_test_clear_uncached(folio))
-                       folio_unmap_invalidate(mapping, folio, 0);
+                       if(!(ki_flags & IOCB_UNCACHED))
+                               shadow = workingset_eviction(folio, folio_memcg(folio));
+                       printk("2 uncached filio = %llx flags = %x\n", folio, folio->flags);
+                       folio_unmap_invalidate(mapping, folio, 0, shadow);
                folio_unlock(folio);
        }
 }

```

1. folio第一次读取加入inactive lru
2. folio被清出
3. folio第二次被读入，与第一次读入的时间比，若时间段则加入active列表
	若时间长则设置uncached标记
4. folio因设置uncached被清出，同时记录此时的shadow
5. folio第三次被读入，与第二次读入的时间比，若时间段则加入active列表，
	若时间长则设置uncached标记