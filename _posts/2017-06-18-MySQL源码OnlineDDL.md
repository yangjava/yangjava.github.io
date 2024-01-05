---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL源码Online DDL

## Online DDL简介

### Online DDL划分
在mysql 8.0上，对于Online DDL的讨论主要从两个角度进行了分类讨论，一个是通过加锁范围来区分不同ddl与dml的并发程度；另一个根据是否拷贝数据来划分不同的执行逻辑。

### 锁与并发度划分
 先说一下与DML语句的并发度方面来说明一下DDL语句的分类，其主要分为下面几类，可以在ddl语句中通过LOCK关键字来指定DDL期间加锁程度。其可选择的值如下：
NONE	允许并发查询和DML
SHARED	允许并发查询，但阻塞DML
DEFAULT	由数据库决定选择最大并发的模式，指定该类型与不指定LOCK关键字含义相同
EXCLUSIVE	阻塞查询和DML

默认的情况下，MySQL在执行DDL操作期间尽可能少的使用锁，以提高并发。当然也可以通过LOCK子句，来指定更加严格的锁。但是，如果LOCK子句指定的锁定级别低于特定DDL操作所允许的限制级别，则语句将失败，并出现错误。

否拷贝数据划分
另一种划分方式为是否拷贝数据，通过ALGORITHM关键字进行指定，值有如下几种：
COPY	采用拷表方式进行表变更，该过程中不允许并发DML
INPLACE	该模式避免进行表的拷贝，而是在让引擎层就地重新生成表，也就是仅需要进行引擎层数据改动，不涉及Server层。在操作的准备和执行阶段，表上的排他元数据锁可能会被短暂地占用。通常，支持并发DML。
INSTANT	该操作仅仅修改元数据。在准备和执行期间，表上没有独占的元数据锁，并且表数据不受影响，因此操作是即时的。允许并发DML。目前仅支持在表最后增加新列；
DEFAULT	系统决定，选择最优的算法执行DDL

如果没有指定ALGORITHM子句，系统决定，选择最优的算法执行DDL。 用户可以选用上述算法来执行，但本身收到DDL类型限制，如果指定的算法无法执行DDL，则ALTER操作会报错。

## 执行流程
Online DDL执行过程可以分为三个阶段：

### 初始化阶段
在初始化阶段，主要检测选择获得最优的LOCK和ALGORITHM设置，如果指定的LOCK和ALGORITHM选项不满足则报错，在该阶段主要持有的是可升级的元数据锁以保护当前的表定义。

### 执行阶段
该阶段主要包含语句的prepare和executed，这个阶段元数据锁是否升级为exclusive取决于在初始化阶段评估。如果需要排他元数据锁，则只在语句准备期间短暂持有，在执行阶段并不持有exclusive元数据锁。如果需要拷表或修改引擎层数据，则该阶段是最耗时的阶段；

### 提交表定义阶段
该阶段就是需要持有排它锁，进行新旧表切换。

从上面的大致过程可以看到，即使是INSTANT的情况下，也依然会需要在提交阶段获取元数据的排它锁，即使最大限度提高并发，仅仅也是通过缩短持有排它锁的时间来实现。所以在默写情况依然会阻塞。

典型的情况就是，DDL语句等待X类型的元数据锁，随后的DML语句则被阻塞等待

官方有一个举例，如下：

会话1 ：
```
mysql> CREATE TABLE t1 (c1 INT) ENGINE=InnoDB;
mysql> START TRANSACTION;
mysql> SELECT * FROM t1;
```
由于事务没有提交，此时会话1持有表t1的共享元数据锁

会话2：
```
mysql> ALTER TABLE t1 ADD COLUMN x INT, ALGORITHM=INPLACE, LOCK=NONE;
```
该语句虽然可以极短时间内完成，但其在提交阶段依然需要获取表t1的元数据排它锁，故阻塞等待会话1中的事务提交或者回滚。

会话3：
```
mysql> SELECT * FROM t1;
```
此时后续会话则被会话2阻塞。

上面就是Online DDL的大致过程，下面将结合源码分析具体执行过程。

## 源码分析
上面说明了MySQL进行在线创建索引时，会自动选择最优的方式进行，不太会对业务造成严重影响，这里要从从MySQL源码出发，分析MySQL中是如何实现的，同时也确认是否在回放DML时会报duplicate key。

### 核心结构
在线online处理的核心代码在文件row0log.cc中，有兴趣的可以进行详细解读，这里说明一下MySQL支持online ddl的细节逻辑是：通过一个日志缓存，保留在ddl期间的dml操作，然后进行缓存日志回复，类似于gh-ost工具，只不过后者采用binlog进行dml操作回放，而mysql内部是单独维护一个核心缓存结构——row_log_t

row_log_t
```
/** @brief Buffer for logging modifications during online index creation

All modifications to an index that is being created will be logged by
row_log_online_op() to this buffer.

All modifications to a table that is being rebuilt will be logged by
row_log_table_delete(), row_log_table_update(), row_log_table_insert()
to this buffer.

When head.blocks == tail.blocks, the reader will access tail.block
directly. When also head.bytes == tail.bytes, both counts will be
reset to 0 and the file will be truncated. */
struct row_log_t {
  int fd;              /*!< file descriptor */
  ib_mutex_t mutex;    /*!< mutex protecting error,
                       max_trx and tail */
  page_no_map *blobs;  /*!< map of page numbers of off-page columns
                       that have been freed during table-rebuilding
                       ALTER TABLE (row_log_table_*); protected by
                       index->lock X-latch only */
  dict_table_t *table; /*!< table that is being rebuilt,
                       or NULL when this is a secondary
                       index that is being created online */
  bool same_pk;        /*!< whether the definition of the PRIMARY KEY
                       has remained the same */
  const dtuple_t *add_cols;
  /*!< default values of added columns, or NULL */
  const ulint *col_map; /*!< mapping of old column numbers to
                        new ones, or NULL if !table */
  dberr_t error;        /*!< error that occurred during online
                        table rebuild */
  trx_id_t max_trx;     /*!< biggest observed trx_id in
                        row_log_online_op();
                        protected by mutex and index->lock S-latch,
                        or by index->lock X-latch only */
  row_log_buf_t tail;   /*!< writer context;
                        protected by mutex and index->lock S-latch,
                        or by index->lock X-latch only */
  row_log_buf_t head;   /*!< reader context; protected by MDL only;
                        modifiable by row_log_apply_ops() */
  ulint n_old_col;
  /*!< number of non-virtual column in
  old table */
  ulint n_old_vcol;
  /*!< number of virtual column in old table */
  const char *path; /*!< where to create temporary file during
                    log operation */
};
```
从说明可以看出，mysql内部将online ddl分为两类：

一类是增加索引类，即调用row_log_online_op函数来进行dml操作缓存填写；
一类是其他ddl。则调用row_log_table_delete, row_log_table_update, row_log_table_insert进行缓存区填充。

下面说要一下，核心结构体中row_log_t各字段含义：

fd，path ：分别是在ddl操作期间，用于保存dml操作记录的临时文件的句柄和文件名；从源码可以看到该目录为innodb_tmpdir指定，若该值为空，则设置为tmpdir对应目录。 其获取临时目录的函数为innobase_mysql_tmpdir()
blobs：记录的写入是按照记录块的方式，该字段表示记录块的数量；
table：不为null表示重建表，为null表示online 添加索引
tail，head：该成员就是记录块，分别用于写入和回放。具体结构 row_log_buf_t 下面会详细说明

row_log_buf_t
```
/** Log block for modifications during online ALTER TABLE */
struct row_log_buf_t {
  byte *block;            /*!< file block buffer */
  ut_new_pfx_t block_pfx; /*!< opaque descriptor of "block". Set
                       by ut_allocator::allocate_large() and fed to
                       ut_allocator::deallocate_large(). */
  mrec_buf_t buf;         /*!< buffer for accessing a record
                          that spans two blocks */
  ulint blocks;           /*!< current position in blocks */
  ulint bytes;            /*!< current position within block */
  ulonglong total;        /*!< logical position, in bytes from
                          the start of the row_log_table log;
                          0 for row_log_online_op() and
                          row_log_apply(). */
};
```
上面就是online ddl记录块的内存结构体，在写入临时文件和读取临时文件都是以块为单位进行。 一个记录块可保存一条或多条增量DML日志。一条增量DML日志可能跨2个记录块。

block：表示正在进行读取或者写入的记录块；

bytes：是该记录块已使用的字节数；

blocks：表示已经往临时文件中写入多少个记录块；

buf：用于处理一条DML日志横跨2个记录块的场景；

total：表示记录总大小。
这里block块大小由参数innodb_sort_buffer_size指定 。
```
/** Allocate the memory for the log buffer.
@param[in,out]  log_buf Buffer used for log operation
@return true if success, false if not */
static MY_ATTRIBUTE((warn_unused_result)) bool row_log_block_allocate(
    row_log_buf_t &log_buf) {
  DBUG_TRACE;
  if (log_buf.block == NULL) {
    DBUG_EXECUTE_IF("simulate_row_log_allocation_failure", return false;);
    // 分配块空间，大小由innodb_sort_buffer_size指定
    log_buf.block = ut_allocator<byte>(mem_key_row_log_buf)
                        .allocate_large(srv_sort_buf_size, &log_buf.block_pfx);

    if (log_buf.block == NULL) {
      return false;
    }
  }
  return true;
}
```
下面以添加索引为例，来说明online ddl的执行过程，对于其他类型的ddl，可以自己分析。

## 增量DML写入实现分析
上面知道，如果是创建索引，其最终调用的是row_log_online_op函数，首先我们来看一下其实现过程
```
/** Logs an operation to a secondary index that is (or was) being created. */
void row_log_online_op(
    dict_index_t *index,   /*!< in/out: index, S or X latched */
    const dtuple_t *tuple, /*!< in: index tuple */
    trx_id_t trx_id)       /*!< in: transaction ID for insert,
                           or 0 for delete */
{
  byte *b;
  ulint extra_size;
  ulint size;
  ulint mrec_size;
  ulint avail_size;
  row_log_t *log;

  ut_ad(dtuple_validate(tuple));
  ut_ad(dtuple_get_n_fields(tuple) == dict_index_get_n_fields(index));
  // 判断已经获取了索引的共享锁或者排它锁
  ut_ad(rw_lock_own(dict_index_get_lock(index), RW_LOCK_S) ||
        rw_lock_own(dict_index_get_lock(index), RW_LOCK_X));
  // 检测索引是否正常
  if (index->is_corrupted()) {
    return;
  }
  // 检测索引状态为ONLINE_INDEX_CREATION
  ut_ad(dict_index_is_online_ddl(index));

  /* Compute the size of the record. This differs from
  row_merge_buf_encode(), because here we do not encode
  extra_size+1 (and reserve 0 as the end-of-chunk marker). */
  // 获取插入记录的长度
  size = rec_get_converted_size_temp(index, tuple->fields, tuple->n_fields,
                                     NULL, &extra_size);
  ut_ad(size >= extra_size);
  ut_ad(size <= sizeof log->tail.buf);
  /*
  真实写入临时文件中的记录，包含：
    记录头（2字节，op和extra_size）
    size
    事务id（如果trx_id非0，即非删除记录）
  */
  mrec_size = ROW_LOG_HEADER_SIZE + (extra_size >= 0x80) + size +
              (trx_id ? DATA_TRX_ID_LEN : 0);

  log = index->online_log;
  mutex_enter(&log->mutex);  // 获取锁
  // 更新log->max_trx
  if (trx_id > log->max_trx) {
    log->max_trx = trx_id;
  }
  // 如果该记录块没有分配空间，则为log->tail->block分配空间
  if (!row_log_block_allocate(log->tail)) {
    log->error = DB_OUT_OF_MEMORY;
    goto err_exit;
  }

  UNIV_MEM_INVALID(log->tail.buf, sizeof log->tail.buf);
  // 计算剩余空间
  ut_ad(log->tail.bytes < srv_sort_buf_size);
  avail_size = srv_sort_buf_size - log->tail.bytes;
  // 如果记录大于剩余空间，则写入buf中，不然这直接写入block中
  if (mrec_size > avail_size) {
    b = log->tail.buf;
  } else {
    b = log->tail.block + log->tail.bytes;
  }
  // 写入操作标识符，trx_id，extra_size，以及数据
  if (trx_id != 0) {
    *b++ = ROW_OP_INSERT;
    trx_write_trx_id(b, trx_id);
    b += DATA_TRX_ID_LEN;
  } else {
    *b++ = ROW_OP_DELETE;
  }

  if (extra_size < 0x80) {
    *b++ = (byte)extra_size;
  } else {
    ut_ad(extra_size < 0x8000);
    *b++ = (byte)(0x80 | (extra_size >> 8));
    *b++ = (byte)extra_size;
  }

  rec_convert_dtuple_to_temp(b + extra_size, index, tuple->fields,
                             tuple->n_fields, NULL);
  b += size;
  /*
  如果记录大于剩余空间，则先将记录部分填入block，调用os_file_write_int_fd写入文件，并将剩余继续写入下一个block缓存;
  block还有空间，直接存入相应空间即可
  */
  if (mrec_size >= avail_size) {
    dberr_t err;
    IORequest request(IORequest::WRITE);
    const os_offset_t byte_offset =
        (os_offset_t)log->tail.blocks * srv_sort_buf_size;
    //如果写入后文件总大小超过innodb_online_alter_log_max_size，则报错
    if (byte_offset + srv_sort_buf_size >= srv_online_max_size) {
      goto write_failed;
    }

    if (mrec_size == avail_size) {
      ut_ad(b == &log->tail.block[srv_sort_buf_size]);
    } else {
      ut_ad(b == log->tail.buf + mrec_size);
      memcpy(log->tail.block + log->tail.bytes, log->tail.buf, avail_size);
    }

    UNIV_MEM_ASSERT_RW(log->tail.block, srv_sort_buf_size);

    if (row_log_tmpfile(log) < 0) {
      log->error = DB_OUT_OF_MEMORY;
      goto err_exit;
    }
    // 写入文件
    err = os_file_write_int_fd(request, "(modification log)", log->fd,
                               log->tail.block, byte_offset, srv_sort_buf_size);

    log->tail.blocks++;
    if (err != DB_SUCCESS) {
    write_failed:
      /* We set the flag directly instead of
      invoking dict_set_corrupted() here,
      because the index is not "public" yet. */
      index->type |= DICT_CORRUPT;
    }
    UNIV_MEM_INVALID(log->tail.block, srv_sort_buf_size);
    // 剩余记录放入下一个block
    memcpy(log->tail.block, log->tail.buf + avail_size, mrec_size - avail_size);
    log->tail.bytes = mrec_size - avail_size;
  } else {
    log->tail.bytes += mrec_size;
    ut_ad(b == log->tail.block + log->tail.bytes);
  }

  UNIV_MEM_INVALID(log->tail.buf, sizeof log->tail.buf);
err_exit:
  mutex_exit(&log->mutex); // 释放锁
}
```
从这里可以看出：

当等待缓存的增量DML日志量mrec_size大于等于当前记录块的可用空间avail_size时，会触发将记录块写入临时文件的操作，将当前的DML日志先写入tail.buf字段，并拷贝DML日志前面部分到当前记录块，将其填满。再调用os_file_write_int_fd将记录块写入临时文件。完成当前记录块写入临时文件后，把DML日志的剩余部分拷贝到已经空闲的tail.block上
如果mrec_size等于avail_size，那么直接写入当前记录块。
DML日志不会全部缓存在内存中，而是会写入到临时文件中，内存中仅保留最后一个记录块。
因此，不存在执行时间过长引起内存空间占用过多的问题。相对来说，临时文件磁盘空间消耗，问题会小很多。
下面来看一下row_log_online_op的调用情况，可以看到只有两个处调用：row_log_online_op_try和row_upd_sec_index_entry_low，其中row_log_online_op_try用于记录的增加和删除，而row_upd_sec_index_entry_low用于记录的更新，下面来看一下具体情况：
```
/** Try to log an operation to a secondary index that is
 (or was) being created.
 @retval true if the operation was logged or can be ignored
 @retval false if online index creation is not taking place */
UNIV_INLINE
bool row_log_online_op_try(
    dict_index_t *index,   /*!< in/out: index, S or X latched */
    const dtuple_t *tuple, /*!< in: index tuple */
    trx_id_t trx_id)       /*!< in: transaction ID for insert,
                           or 0 for delete */
{
  // 获取锁
  ut_ad(rw_lock_own_flagged(dict_index_get_lock(index),
                            RW_LOCK_FLAG_S | RW_LOCK_FLAG_X | RW_LOCK_FLAG_SX));

  switch (dict_index_get_online_status(index)) {
    case ONLINE_INDEX_COMPLETE:
      /* This is a normal index. Do not log anything.
      The caller must perform the operation on the
      index tree directly. */
      return (false);
    case ONLINE_INDEX_CREATION:
      /* The index is being created online. Log the
      operation. */
      row_log_online_op(index, tuple, trx_id);
      break;
    case ONLINE_INDEX_ABORTED:
    case ONLINE_INDEX_ABORTED_DROPPED:
      /* The index was created online, but the operation was
      aborted. Do not log the operation and tell the caller
      to skip the operation. */
      break;
  }

  return (true);
}

/** Updates a secondary index entry of a row.
@param[in]  node        row update node
@param[in]  old_entry   the old entry to search, or nullptr then it
                                has to be created in this function
@param[in]  thr     query thread
@return DB_SUCCESS if operation successfully completed, else error
code or DB_LOCK_WAIT */
static MY_ATTRIBUTE((warn_unused_result)) dberr_t
    row_upd_sec_index_entry_low(upd_node_t *node, dtuple_t *old_entry,
                                que_thr_t *thr) {
    …………
    // 获取锁
    mtr_s_lock(dict_index_get_lock(index), &mtr);

    switch (dict_index_get_online_status(index)) {
      case ONLINE_INDEX_COMPLETE:
        /* This is a normal index. Do not log anything.
        Perform the update on the index tree directly. */
        break;
      case ONLINE_INDEX_CREATION:
        /* Log a DELETE and optionally INSERT. */
        // 先插入删除操作
        row_log_online_op(index, entry, 0);

        if (!node->is_delete) {
          mem_heap_empty(heap);
          entry =
              row_build_index_entry(node->upd_row, node->upd_ext, index, heap);
          ut_a(entry);
          // 再进行插入操作
          row_log_online_op(index, entry, trx->id);
        }
        /* fall through */
      case ONLINE_INDEX_ABORTED:
      case ONLINE_INDEX_ABORTED_DROPPED:
        mtr_commit(&mtr);
        goto func_exit;
    }
    ……
}

```
从上面可以看出：

对于二次索引的插入和删除操作，指定调用row_log_online_op_try函数；
对于二次索引的更新操作，调用row_upd_sec_index_entry_low，并且内部是先记录删除操作，然后再插入；
所有操作都是在index锁保护之下进行。
思考通过上面看，应该没有主键冲突才是呀，为什么官方提示存在主键冲突的可能呢？
```
When running an in-place online DDL operation, the thread that runs the ALTER TABLE statement applies an online log of DML operations that were run concurrently on the same table from other connection threads. When the DML operations are applied, it is possible to encounter a duplicate key entry error (ERROR 1062 (23000): Duplicate entry), even if the duplicate entry is only temporary and would be reverted by a later entry in the online log. This is similar to the idea of a foreign key constraint check in InnoDB in which constraints must hold during a transaction.
```
可以从row_log_online_op_try的调用情况分析：
```
 if (check) {
    DEBUG_SYNC_C("row_ins_sec_index_enter");
    if (mode == BTR_MODIFY_LEAF) {
      search_mode |= BTR_ALREADY_S_LATCHED;
      mtr_s_lock(dict_index_get_lock(index), &mtr);
    } else {
      mtr_sx_lock(dict_index_get_lock(index), &mtr);
    }
    // 插入dml操作到缓存区
    if (row_log_online_op_try(index, entry, thr_get_trx(thr)->id)) {
      goto func_exit;
    }
  }
  ……
  if (row_ins_sec_mtr_start_and_check_if_aborted(&mtr, index, check,
                                                   search_mode)) {
      goto func_exit;
    }
    // 检测是否主键冲突
    err = row_ins_scan_sec_index_for_duplicate(flags, index, entry, thr, check,
                                               &mtr, offsets_heap);

    mtr_commit(&mtr);

    switch (err) {
      case DB_SUCCESS:
        break;
      case DB_DUPLICATE_KEY:  // 如果主键冲突，则设置所以corrupted，但是err为DB_SUCCESS
        if (!index->is_committed()) {
          ut_ad(!thr_get_trx(thr)->dict_operation_lock_mode);

          dict_set_corrupted(index);
          /* Do not return any error to the
          caller. The duplicate will be reported
          by ALTER TABLE or CREATE UNIQUE INDEX.
          Unfortunately we cannot report the
          duplicate key value to the DDL thread,
          because the altered_table object is
          private to its call stack. */
          err = DB_SUCCESS;
        }
        /* fall through */
      default:
        if (dict_index_is_spatial(index)) {
          rtr_clean_rtr_info(&rtr_info, true);
        }
        return err;
    }
```
从上面的代码可以看到，虽然已经检测到了主键冲突，但是对于dml客户端来说并不报错，而是将index设置为corrupted，从而导致在回访dml记录时报错-主键冲突。

## 增量DML回放实现分析
dml记录回访的接口函数为row_log_apply，而具体的操作是在函数row_log_apply_ops中，下面来看一下该函数的简化代码：
```
next_block:
  ut_ad(has_index_lock);
  // 判断已经获得索引锁
  ut_ad(rw_lock_own(dict_index_get_lock(index), RW_LOCK_X));
  ut_ad(index->online_log->head.bytes == 0);

  // 事务是否被中断
  if (trx_is_interrupted(trx)) {
    goto interrupted;
  }

  error = index->online_log->error;
  if (error != DB_SUCCESS) {
    goto func_exit;
  }
  //检测索引是否正常，或者是否错误
  if (index->is_corrupted()) {
    error = DB_INDEX_CORRUPT;
    goto func_exit;
  }

  /*
  如果是最后一个记录块，则直接在内存中获取，并且删除文件,保存持锁状态
  如果是非最后一个记录块，则从文件中读取记录块，解除index锁状态进行回放
  */
  if (index->online_log->head.blocks == index->online_log->tail.blocks) {
    if (index->online_log->head.blocks) {
#ifdef HAVE_FTRUNCATE
      /* Truncate the file in order to save space. */
      if (index->online_log->fd > 0 &&
          ftruncate(index->online_log->fd, 0) == -1) {
        perror("ftruncate");
      }
#endif /* HAVE_FTRUNCATE */
      index->online_log->head.blocks = index->online_log->tail.blocks = 0;
    }
    // 内存获取最后一个记录块
    next_mrec = index->online_log->tail.block;
    next_mrec_end = next_mrec + index->online_log->tail.bytes;
   // 非最后一块
  } else {
    os_offset_t ofs;

    ofs = (os_offset_t)index->online_log->head.blocks * srv_sort_buf_size;

    ut_ad(has_index_lock);
    has_index_lock = false;
    // 不是最后一块记录则在回放中会释放index锁 
    rw_lock_x_unlock(dict_index_get_lock(index));
    // 分配空间
    if (!row_log_block_allocate(index->online_log->head)) {
      error = DB_OUT_OF_MEMORY;
      goto func_exit;
    }
    // 读取文件记录块
    IORequest request;
    dberr_t err = os_file_read_no_error_handling_int_fd(
        request, index->online_log->path, index->online_log->fd,
        index->online_log->head.block, ofs, srv_sort_buf_size, NULL);

    if (err != DB_SUCCESS) {
      ib::error(ER_IB_MSG_963) << "Unable to read temporary file"
                                  " for index "
                               << index->name;
      goto corruption;
    }
    next_mrec = index->online_log->head.block;
    next_mrec_end = next_mrec + srv_sort_buf_size;
  }
 /*
 调用row_log_apply_op循环回放相应的记录操作，
 当一个块回放完成后，则调到next_block标记处进行下一个块的回放
 */
  while (!trx_is_interrupted(trx)) {
    mrec = next_mrec;
    ut_ad(mrec < mrec_end);

    if (!has_index_lock) {
      /* We are applying operations from a different
      block than the one that is being written to.
      We do not hold index->lock in order to
      allow other threads to concurrently buffer
      modifications. */
      ut_ad(mrec >= index->online_log->head.block);
      ut_ad(mrec_end == index->online_log->head.block + srv_sort_buf_size);
      ut_ad(index->online_log->head.bytes < srv_sort_buf_size);

      /* Take the opportunity to do a redo log
      checkpoint if needed. */
      log_free_check();
    } else {
      /* We are applying operations from the last block.
      Do not allow other threads to buffer anything,
      so that we can finally catch up and synchronize. */
      ut_ad(index->online_log->head.blocks == 0);
      ut_ad(index->online_log->tail.blocks == 0);
      ut_ad(mrec_end ==
            index->online_log->tail.block + index->online_log->tail.bytes);
      ut_ad(mrec >= index->online_log->tail.block);
    }

    next_mrec = row_log_apply_op(index, dup, &error, offsets_heap, heap,
                                 has_index_lock, mrec, mrec_end, offsets);

    if (error != DB_SUCCESS) {
      goto func_exit;
    } else if (next_mrec == next_mrec_end) {
      /* The record happened to end on a block boundary.
      Do we have more blocks left? */
      if (has_index_lock) {
        /* The index will be locked while
        applying the last block. */
        goto all_done;
      }

      mrec = NULL;
    process_next_block:
      rw_lock_x_lock(dict_index_get_lock(index));
      has_index_lock = true;

      index->online_log->head.bytes = 0;
      index->online_log->head.blocks++;
      goto next_block;
    } else if (next_mrec != NULL) {
      ut_ad(next_mrec < next_mrec_end);
      index->online_log->head.bytes += next_mrec - mrec;
    } else if (has_index_lock) {
      /* When mrec is within tail.block, it should
      be a complete record, because we are holding
      index->lock and thus excluding the writer. */
      ut_ad(index->online_log->tail.blocks == 0);
      ut_ad(mrec_end ==
            index->online_log->tail.block + index->online_log->tail.bytes);
      ut_ad(0);
      goto unexpected_eof;
    } else {
      memcpy(index->online_log->head.buf, mrec, mrec_end - mrec);
      mrec_end += index->online_log->head.buf - mrec;
      mrec = index->online_log->head.buf;
      goto process_next_block;
    }
  }
```
从代码分析可以得出：

虽然进入该函数时加了index锁，但在处理非最后一个block时，会释放锁，然后读取文件上的对应日志块并进行回放，只有当处理最后一个内存块时一直持锁；
处理最后一个block时不需要从日志文件中读取block，因为最后一个block还缓存在内存中。因此，在开始处理前会先将用于缓存增量DML日志的临时文件truncate掉，避免无意义的存储资源消耗；
创建二级索引时会通过trx_is_interrupted判断创建操作是否被中断，也就是说可以通过kill等方式终止创建操作
回放过程通过index->is_corrupted()判断索引是否正常，上面的主键冲突由于设置了该索引corrupted，故被终止。
















