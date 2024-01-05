---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL事务系统row_search_mvcc
首先需要明白，innodb快照读的入口函数为row_search_mvcc，文中首先给出整个innodb行读取流程框图，然后再结合源码进行分步细致讲解，希望对各位innodb读取记录的源码阅读有所帮助。

## 流程说明
 查询执行前检查，这些检查主要是针对索引以及表的完整性进行检查

innodb全文索引采用的是倒排索引辅助表实现。如果是任何传统的索引查询不能使用全文索引
```
  if (prebuilt->index->type & DICT_FTS) { // 检查当前索引是否为FTS，全文索引
    return DB_END_OF_INDEX;
  }
```
检查相应表是否依然可用，如表是否被discard，表的ibd文件是否存在等等
```
  if (dict_table_is_discarded(prebuilt->table)) { // 表是否被标记为discarded
    return DB_TABLESPACE_DELETED;

  } else if (prebuilt->table->ibd_file_missing) { // 表中的idb文件是否存在
    return DB_TABLESPACE_NOT_FOUND;

  } else if (!prebuilt->index_usable) {   // 对于本次事务，这个索引是否可见
    return DB_MISSING_HISTORY;

  } else if (prebuilt->index->is_corrupted()) {  // 索引是否损坏
    return DB_CORRUPTION;
  }
```
是否可以直接从上一次读取的缓存record_buffer缓存中读取，这里是innodb对于加快读取的一个优化策略，如果缓存中存在，则无须后续读取操作，直接从缓存中获取，至于缓存使用情况，后面进行说明。
```
// 如果查询方向改变，则说明之前的缓存没有用了，清空即可
if (UNIV_UNLIKELY(direction != prebuilt->fetch_direction)) {
    ……
    prebuilt->n_rows_fetched = 0;
    prebuilt->n_fetch_cached = 0;
    prebuilt->fetch_cache_first = 0;
    ut_ad(record_buffer == nullptr);
} else if (UNIV_LIKELY(prebuilt->n_fetch_cached > 0)) { 
    // 方向相同，且缓存中还有未读取的行，直接使用缓存数据即可
    row_sel_dequeue_cached_row_for_mysql(buf, prebuilt);
    prebuilt->n_rows_fetched++;
    err = DB_SUCCESS;
    goto func_exit;
}
```
对唯一索引模式进行判断，何为唯一索引模式，通俗讲就是在操作范围内最多有一条记录满足要求，那如何在查询前就清楚是只有一条呢？需要满足下面条件

是准确查找，而不是最左匹配，即match_mode == ROW_SEL_EXACT；
索引为唯一索引，当然聚集索引也是唯一索引，如果非聚集索引的唯一索引还要求索引条件不能包含null；
查询条件涵盖所有索引列
```
if (match_mode == ROW_SEL_EXACT && dict_index_is_unique(index) &&
      dtuple_get_n_fields(search_tuple) == dict_index_get_n_unique(index) &&
      (index->is_clustered() || !dtuple_contains_null(search_tuple))) {
    /* Note above that a UNIQUE secondary index can contain many
    rows with the same key value if one of the columns is the SQL
    null. A clustered index under MySQL can never contain null
    columns because we demand that all the columns in primary key
    are non-null. */

    unique_search = TRUE;
    ……
}
```
判断唯一索引模式的好处则是：加锁优化，自适应hash索引等相关优化都是基于唯一索引模式
判断是否采用自适应hash查找，这里建议参考http://mysql.taobao.org/monthly/2015/09/01/

定位游标

因为innodb对于索引查找以及全表扫描的处理方式不同，所以打开游标的方式也不同。具体打开游标步骤如下：
确定后续是否需要间隙锁
```
if (prebuilt->table->skip_gap_locks() ||
      (trx->skip_gap_locks() && prebuilt->select_lock_type != LOCK_NONE &&
       trx->mysql_thd != NULL && thd_is_select(trx->mysql_thd))) {
    set_also_gap_locks = false;
}
```
也就是满足以下任意条件都不需要间隙锁：

条件1：操作的表为元数据表或者SDI表（非innodb表的元数据信息），即prebuilt->table->skip_gap_locks()

条件2：

事务隔离级别无须间隙锁，trx->skip_gap_locks()，即隔离级别小于RR;

执行select语句且采用快照读，不加锁，即prebuilt->select_lock_type != LOCK_NONE && thd_is_select(trx->mysql_thd)

所以在RC隔离级别下，有时候也会出现gap锁。

确定是否需要回表，对于二级索引，如果索引不能满足所有要求列，则需要回表
```
clust_templ_for_sec = index != clust_index && prebuilt->need_to_access_clustered;
```
判断采用当前读还是一致性读，对于一致性读则分配快照，如果是当前读，则需要进行加表锁
```
if (!prebuilt->sql_stat_start) {
    /* 如果已经添加了表锁或者分配了快照则不进行相关操作 */
    ……
  } else if (prebuilt->select_lock_type == LOCK_NONE) {
    /* 一致性读，则分配一个视图 */
    if (!srv_read_only_mode) {
      trx_assign_read_view(trx);
    }

    prebuilt->sql_stat_start = FALSE;
  } else {
  wait_table_again:
    // 如果当前读，则进行表锁获取（可能IS或者IX）
    err = lock_table(0, index->table,
                     prebuilt->select_lock_type == LOCK_S ? LOCK_IS : LOCK_IX,
                     thr);

    if (err != DB_SUCCESS) {
      table_lock_waited = TRUE;
      goto lock_table_wait;
    }
    prebuilt->sql_stat_start = FALSE;
  }
```
真正打开游标，这里根据访问方式不同分别处理
```
if (dtuple_get_n_fields(search_tuple) > 0) { // 如果是所有扫描方式
    pcur->m_btr_cur.thr = thr;
    // 调研btr_pcur_open_with_no_init定位到相应条件的边界
    btr_pcur_open_with_no_init(index, search_tuple, mode, BTR_SEARCH_LEAF, pcur,
                               0, &mtr);

    pcur->m_trx_if_known = trx;
    // 获取游标记录
    rec = btr_pcur_get_rec(pcur);

    if (!moves_up && !page_rec_is_supremum(rec) && set_also_gap_locks &&
        !trx->skip_gap_locks() && prebuilt->select_lock_type != LOCK_NONE &&
        !dict_index_is_spatial(index)) {
      /* 对于降序扫描，第一条满足条件的下一条记录需要添加gap锁 */
      const rec_t *next_rec = page_rec_get_next_const(rec);

      offsets =
          rec_get_offsets(next_rec, index, offsets, ULINT_UNDEFINED, &heap);
      err = sel_set_rec_lock(pcur, next_rec, index, offsets,
                             prebuilt->select_mode, prebuilt->select_lock_type,
                             LOCK_GAP, thr, &mtr);

      switch (err) {
        case DB_SUCCESS_LOCKED_REC:
          err = DB_SUCCESS;
        case DB_SUCCESS:
          break;
        case DB_SKIP_LOCKED:
        case DB_LOCK_NOWAIT:
          ut_ad(0);
          goto next_rec;
        default:
          goto lock_wait_or_error;
      }
    }
  } else if (mode == PAGE_CUR_G || mode == PAGE_CUR_L) {
    // 如果全索引扫描，直接定位到索引最左边
    btr_pcur_open_at_index_side(mode == PAGE_CUR_G, index, BTR_SEARCH_LEAF,
                                pcur, false, 0, &mtr);
  }
```
获取匹配的记录

检查当前游标记录是否合法，如果存在问题则直接报错
虽然上面通过索引定位获取到了游标，但是也可能是由于可以通过移动游标然后调整到rec_loop处，所以这里需要重新进行比较
```
if (match_mode == ROW_SEL_EXACT) {
    /* 如果是完全匹配 */
    if (0 != cmp_dtuple_rec(search_tuple, rec, index, offsets)) {
      if (set_also_gap_locks && !trx->skip_gap_locks() &&
          prebuilt->select_lock_type != LOCK_NONE &&
          !dict_index_is_spatial(index)) { //匹配不满足，可能也需要添加gap锁
        err = sel_set_rec_lock(pcur, rec, index, offsets, prebuilt->select_mode,
                               prebuilt->select_lock_type, LOCK_GAP, thr, &mtr);

        switch (err) {
          case DB_SUCCESS_LOCKED_REC:
          case DB_SUCCESS:
            break;
          case DB_SKIP_LOCKED:
          case DB_LOCK_NOWAIT:
            ut_ad(0);
          default:
            goto lock_wait_or_error;
        }
      }

      btr_pcur_store_position(pcur, &mtr);

      /* The found record was not a match, but may be used
      as NEXT record (index_next). Set the relative position
      to BTR_PCUR_BEFORE, to reflect that the position of
      the persistent cursor is before the found/stored row
      (pcur->m_old_rec). */
      ut_ad(pcur->m_rel_pos == BTR_PCUR_ON);
      pcur->m_rel_pos = BTR_PCUR_BEFORE;

      err = DB_RECORD_NOT_FOUND;
      goto normal_return;
    }

  } else if (match_mode == ROW_SEL_EXACT_PREFIX) { // 最左匹配
    if (!cmp_dtuple_is_prefix_of_rec(search_tuple, rec, index, offsets)) {
      if (set_also_gap_locks && !trx->skip_gap_locks() &&
          prebuilt->select_lock_type != LOCK_NONE &&
          !dict_index_is_spatial(index)) {
        err = sel_set_rec_lock(pcur, rec, index, offsets, prebuilt->select_mode,
                               prebuilt->select_lock_type, LOCK_GAP, thr, &mtr);

        switch (err) {
          case DB_SUCCESS_LOCKED_REC:
          case DB_SUCCESS:
            break;
          case DB_SKIP_LOCKED:
          case DB_LOCK_NOWAIT:
            ut_ad(0);
          default:
            goto lock_wait_or_error;
        }
      }

      btr_pcur_store_position(pcur, &mtr);

      /* The found record was not a match, but may be used
      as NEXT record (index_next). Set the relative position
      to BTR_PCUR_BEFORE, to reflect that the position of
      the persistent cursor is before the found/stored row
      (pcur->old_rec). */
      ut_ad(pcur->m_rel_pos == BTR_PCUR_ON);
      pcur->m_rel_pos = BTR_PCUR_BEFORE;

      err = DB_RECORD_NOT_FOUND;
      goto normal_return;
    }
  }
```
根据读取模式（快照 or 当前）进行对应处理
```
if (prebuilt->select_lock_type != LOCK_NONE) {  // 当前读模式
    auto row_to_range_relation = row_compare_row_to_range(
        set_also_gap_locks, trx, unique_search, index, clust_index, rec, comp,
        mode, direction, search_tuple, offsets, moves_up, prebuilt);  // 检查记录以及对应的gap是否在范围内

    ulint lock_type;
    if (row_to_range_relation.row_can_be_in_range) {
      if (row_to_range_relation.gap_can_intersect_range) {
        lock_type = LOCK_ORDINARY;  // 1、如果记录与gap都在索引范围内，加next_lock
      } else {
        lock_type = LOCK_REC_NOT_GAP; // 2、如果仅仅记录在索引范围内，加记录锁
      }
    } else {
      if (row_to_range_relation.gap_can_intersect_range) {
        lock_type = LOCK_GAP;  // 3、如果仅仅gap在索引范围内，加gap锁
      } else {
        err = DB_RECORD_NOT_FOUND;  // 4、如果都不在索引范围内，未找到
        goto normal_return;
      }
    }

    err = sel_set_rec_lock(pcur, rec, index, offsets, prebuilt->select_mode,
                           prebuilt->select_lock_type, lock_type, thr, &mtr);  // 进行加锁

    switch (err) {
      ……  // 根据加锁情况，分别处理
    }
  locks_ok:
    if (err == DB_SUCCESS && !row_to_range_relation.row_can_be_in_range) {
      err = DB_RECORD_NOT_FOUND;
      goto normal_return;
    }
  } else {
    /* 一致性读 */

    if (trx->isolation_level == TRX_ISO_READ_UNCOMMITTED) {
      /* 如果是RU，则无须操作 */

    } else if (index == clust_index) {
      /* 如果是主键索引，可以直接构建结果 */

      if (srv_force_recovery < 5 &&
          !lock_clust_rec_cons_read_sees(rec, index, offsets,
                                         trx_get_read_view(trx))) { // 当前行不可见
        rec_t *old_vers;
        /* 如果当前行不可见，使用undo log构建历史版本，直到可见版本 */
        err = row_sel_build_prev_vers_for_mysql(
            trx->read_view, clust_index, prebuilt, rec, &offsets, &heap,
            &old_vers, need_vrow ? &vrow : NULL, &mtr,
            prebuilt->get_lob_undo());

        if (err != DB_SUCCESS) {
          goto lock_wait_or_error;
        }

        if (old_vers == NULL) {
          /* The row did not exist yet in
          the read view */

          goto next_rec;
        }

        rec = old_vers;
        prev_rec = rec;
      }
    } else {
      /* 对于二级索引 */

      ut_ad(!index->is_clustered());

      if (!srv_read_only_mode &&
          !lock_sec_rec_cons_read_sees(rec, index, trx->read_view)) {
        /* 如果不可见，需要先进行索引下推判断，然后进行回表操作 */
        switch (row_search_idx_cond_check(buf, prebuilt, rec, offsets)) {
          case ICP_NO_MATCH:
            goto next_rec;
          case ICP_OUT_OF_RANGE:
            err = DB_RECORD_NOT_FOUND;
            goto idx_cond_failed;
          case ICP_MATCH:
            goto requires_clust_rec;
        }

        ut_error;
      }
    }
  }
```
回表读取逻辑如下
```
if (index != clust_index && prebuilt->need_to_access_clustered) { 
  requires_clust_rec:
    ……
    mtr_has_extra_clust_latch = TRUE;
    ut_ad(!vrow);

    /* 通过二级索引回表获取聚集索引记录，注意此时已经讲旧版本也读取了. */
    err = row_sel_get_clust_rec_for_mysql(
        prebuilt, index, rec, thr, &clust_rec, &offsets, &heap,
        need_vrow ? &vrow : NULL, &mtr, prebuilt->get_lob_undo());
    switch (err) {
      ……
    }

    if (rec_get_deleted_flag(clust_rec, comp)) {
      /* 跳过被删除的记录 */
      ……
      goto next_rec;
    }

    if (need_vrow && !vrow) {  // 存在虚拟列，则构建
      if (!heap) {
        heap = mem_heap_create(100);
      }
      row_sel_fill_vrow(rec, index, &vrow, heap);
    }

    result_rec = clust_rec;
    ut_ad(rec_offs_validate(result_rec, clust_index, offsets));

    if (prebuilt->idx_cond) { // 如果存在ICP，进行格式转换
      /* Convert the record to MySQL format. We were
      unable to do this in row_search_idx_cond_check(),
      because the condition is on the secondary index
      and the requested column is in the clustered index.
      We convert all fields, including those that
      may have been used in ICP, because the
      secondary index may contain a column prefix
      rather than the full column. Also, as noted
      in Bug #56680, the column in the secondary
      index may be in the wrong case, and the
      authoritative case is in result_rec, the
      appropriate version of the clustered index record. */
      if (!row_sel_store_mysql_rec(buf, prebuilt, result_rec, vrow, TRUE,
                                   clust_index, offsets, false, nullptr,
                                   nullptr)) {
        goto next_rec;
      }
    }
    ……
  } else {
    result_rec = rec;
  }
```
到达这一步，此时已经获得了可见的记录，此时需要对记录的删除标记进行判断
```
f (rec_get_deleted_flag(rec, comp)) { // 记录存在删除标记
    /* The record is delete-marked: we can skip it */

    if (trx->allow_semi_consistent() &&
        prebuilt->select_lock_type != LOCK_NONE && !did_semi_consistent_read) {
      /* No need to keep a lock on a delete-marked record
      if we do not want to use next-key locking. */

      row_unlock_for_mysql(prebuilt, TRUE);
    }

    /* 如果是唯一索引，则退出 */
    if (index == clust_index && unique_search && !prebuilt->used_in_HANDLER) {
      err = DB_RECORD_NOT_FOUND;

      goto normal_return;
    }

    goto next_rec;
  }
```
进行缓存存储，为了优化读取性能，我们可以一次性读取多行，然后缓存住，从而减少后续读取开销，具体什么情况可以进行缓存呢？
```
if (record_buffer != nullptr ||
      ((match_mode == ROW_SEL_EXACT ||
        prebuilt->n_rows_fetched >= MYSQL_FETCH_CACHE_THRESHOLD) &&
       prebuilt->can_prefetch_records())) 
```
也就是在精准匹配模式下，或者练习相同模式获取了多行。

讲查询结果转换为mysql行格式并返回，主要函数为row_sel_store_mysql_rec

如果上述过程中，出现跳转next_rec标签的，则在这里之后移动游标，进行后续记录读取
```
next_rec:
  ……
  if (moves_up) {  // 如果按索引顺序查找
    bool move;

    ……
      move = btr_pcur_move_to_next(pcur, &mtr); //游标移动到下一条记录

    if (!move) { // 如果达到最后一条记录
    not_moved:
      ……
      if (match_mode != 0) {
        err = DB_RECORD_NOT_FOUND; // 匹配模式下，则返回未找到
      } else {
        err = DB_END_OF_INDEX; // 全表扫描模式下，则提升已到达索引结尾
      }

      goto normal_return;
    }
  } else { // 如果逆序查找，则游标向前移动，移动的处理与上述一致
    if (UNIV_UNLIKELY(!btr_pcur_move_to_prev(pcur, &mtr))) {
      goto not_moved;
    }
  }

  goto rec_loop; // 移动完成则借助处理该条记录
```