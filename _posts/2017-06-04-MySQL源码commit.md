---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL源码commit
MySQL 的 commit 命令提交事务时，内部会进行两阶段（Prepare 和 Commit）提交，这篇文章基于 MySQL 8.0.33 对 MySQL 的两阶段提交进行源码分析，带你了解提交事务过程中都经历了什么。

## Prepare 阶段
- Binlog Prepare
获取上一个事务最大的 sequence number 时间戳。
- InnoDB Prepare
事务状态设置为 prepared；
释放 RC 及以下隔离级别的 GAP Lock；
将 Undo log segment 的状态从 TRX_UNDO_ACTIVE 修改为 TRX_UNDO_PREPARED；
Undo log 写入事务 XID。

## Commit 阶段 
### Stage 0
保证从实例的 commit order。

### Flush Stage
根据 innodb_flush_log_at_trx_commit 参数进行 redo log 的刷盘操作

获取并清空 BINLOG_FLUSH_STAGE 和 COMMIT_ORDER_FLUSH_STAGE 队列
存储引擎层将 prepare 状态的 redo log 根据 innodb_flush_log_at_trx_commit 参数刷盘
不再阻塞 slave 的 preserve commit order 的执行
调用 get_server_sidno() 和 Gtid_state::get_automatic_gno() 生成 GTID
Flush binlog_cache_mngr

Flush stmt_cache
Flush trx_cache

生成 last_committed 和 sequence_number
flush GTID log event
将 trx_cache 中的数据 flush 到 binlog cache 中
准备提交事务后的 Binlog pos
递增 prepread XID
插桩调用after_flush，将已经 flush 的 binlog file 和 position 注册到半同步复制插件中
如果 sync_binlog!=1，在 flush stage 更新 Binlog 位点，并广播 update 信号，从库的 Dump 线程可以由此感知 Binlog 的更新

### Sync Stage
根据 sync_binlog 的参数设置进行刷盘前的等待并调用 fsync() 进行刷盘
如果 sync_binlog==1，在 sync stage 阶段更新 binog 位点，并广播 update 信号，从库的 Dump 线程可以由此感知 Binlog 的更新

### Commit Stage
after_sync hook（半同步复制 after_sync 的钩子）
更新全局的 m_max_committed_transaction（用作后续事务的 last_committed），并初始化事务上下文的 sequence number
Binlog 层提交，什么也不做
存储引擎层提交

为持久化 GTID 提前分配 update undo segment
更新数据字典中被修改表的 update_time 时间
分配 Mini-transaction handle和buffer
更新 undo 状态

- 对于 insert 状态从 TRX_UNDO_ACTIVE 修改为 TRX_UNDO_TO_FREE，update 修改为 TRX_UNDO_TO_PURGE
- 如果事务为 update 还需要将 rollback segments 分配 trx no，并将其添加到 purge 队列中
  将 update undo log header 添加到 history list 开头释放一些内存对象
  在系统事务表记录 binlog 位点
  关闭 mvcc read view
  持久化 GTID
  释放 insert undo log
  唤醒后台线程开始干活，如 master thread、purge thread、page_cleaner
  更新整组事务的 executed_gtid
  在存储引擎层提交之后，递减 Prepared 状态下的 XID 计数器
  after_sync hook（半同步复制 after_commit的钩子）
  广播 m_stage_cond_binlog 信号变量，唤醒挂起的 follower
  ha_commit_trans 函数主要判断是否需要写入 GTID 信息，并开始两阶段提交：
```
int ha_commit_trans(THD *thd, bool all, bool ignore_global_read_lock) {
  /*
    Save transaction owned gtid into table before transaction prepare
    if binlog is disabled, or binlog is enabled and log_replica_updates
    is disabled with slave SQL thread or slave worker thread.
  */
  std::tie(error, need_clear_owned_gtid) = commit_owned_gtids(thd, all);
...
  // Prepare 阶段
  if (!trn_ctx->no_2pc(trx_scope) && (trn_ctx->rw_ha_count(trx_scope) > 1))
    error = tc_log->prepare(thd, all);
...
  // Commit 阶段
 if (error || (error = tc_log->commit(thd, all))) {
    ha_rollback_trans(thd, all);
    error = 1;
    goto end;
  }
}
```

## Prepare 阶段
两阶段提交的 Prepare 阶段相对简单，以下是 commit 命令入口及 Prepare 阶段的堆栈和相关作用：
```
|mysql_execute_command
|--trans_commit
|----ha_commit_trans
|------MYSQL_BIN_LOG::prepare

// 开启 binlog prepare 和 innodb prepare
|--------ha_prepare_low       

// Binlog prepare：获取上一个事务最大的 sequence number 时间戳
|----------binlog_prepare   

// innodb prepare
|----------innobase_xa_prepare            　　　　
|------------trx_prepare_for_mysql

// 1. 调用 trx_prepare_low 
// 2. 事务状态设置为Prepared 
// 3. 释放 RC 及以下隔离级别的 GAP Lock 
// 4. 刷盘 Redo（已推迟到 Commit 阶段的 Flush stage）
|--------------trx_prepare                        
|----------------trx_prepare_low

// 1. 将 undo log segment 的状态从 TRX_UNDO_ACTIVE 修改为 TRX_UNDO_PREPARED 
// 2. undo log 写入事务 XID
|------------------trx_undo_set_state_at_prepare  
```
## Commit 阶段
Commit 阶段的功能实现主要集中在 MYSQL_BIN_LOG::ordered_commit 函数中。

## Flush 阶段
首先看下 Stage 0 和 Stage 1，stage 0 主要是 8.0 新增的一个阶段，主要是针对从库保证 commit order。stage 1 就是大家耳熟能详的 Commit 阶段的三个小阶段其一的 Flush 阶段了：
```
int MYSQL_BIN_LOG::ordered_commit(THD *thd, bool all, bool skip_commit) {
 
  /*
    Stage #0: 保证从实例的 SQL 线程按照 Relay log 的事务顺序进行提交
  */
  if (Commit_order_manager::wait_for_its_turn_before_flush_stage(thd) ||
      ending_trans(thd, all) ||
      Commit_order_manager::get_rollback_status(thd)) {
    if (Commit_order_manager::wait(thd)) {
      return thd->commit_error;
    }
  }
 
  /*
    Stage #1: flushing transactions to binary log
 
    While flushing, we allow new threads to enter and will process
    them in due time. Once the queue was empty, we cannot reap
    anything more since it is possible that a thread entered and
    appointed itself leader for the flush phase.
  */
 
  if (change_stage(thd, Commit_stage_manager::BINLOG_FLUSH_STAGE, thd, nullptr,
                   &LOCK_log)) {
    DBUG_PRINT("return", ("Thread ID: %u, commit_error: %d", thd->thread_id(),
                          thd->commit_error));
    return finish_commit(thd);
  }
 
  THD *wait_queue = nullptr, *final_queue = nullptr;
  mysql_mutex_t *leave_mutex_before_commit_stage = nullptr;
  my_off_t flush_end_pos = 0;
  bool update_binlog_end_pos_after_sync;
 
  // Flush 阶段主要的处理逻辑
  flush_error =
      process_flush_stage_queue(&total_bytes, &do_rotate, &wait_queue);
 
  if (flush_error == 0 && total_bytes > 0)
    /*
      flush binlog cache到file cache
    */
    flush_error = flush_cache_to_file(&flush_end_pos);
 
  // 后面根据 sync_binlog 参数决定更新 binlog pos 的位置并广播 Binlog 更新信号
  update_binlog_end_pos_after_sync = (get_sync_period() == 1);
 
  /*
    If the flush finished successfully, we can call the after_flush
    hook. Being invoked here, we have the guarantee that the hook is
    executed before the before/after_send_hooks on the dump thread
    preventing race conditions among these plug-ins.
  */
  if (flush_error == 0) {
    const char *file_name_ptr = log_file_name + dirname_length(log_file_name);
    assert(flush_end_pos != 0);
    /*
      插桩调用 after_flush，将已经 flush 的 binlog file 和 position 注册到半同步复制插件中，
      用于后续对比 slave 应答接受到的 binlog position。
    */
    if (RUN_HOOK(binlog_storage, after_flush,
                 (thd, file_name_ptr, flush_end_pos))) {
      LogErr(ERROR_LEVEL, ER_BINLOG_FAILED_TO_RUN_AFTER_FLUSH_HOOK);
      flush_error = ER_ERROR_ON_WRITE;
    }
 
    // 如果 sync_binlog!=1，在 flush stage 更新 binlog 位点并广播 update 信号，从库的 Dump 线程可以由此感知 Binlog 的更新
    if (!update_binlog_end_pos_after_sync) update_binlog_end_pos();
  }
```
Flush stage 的主要处理逻辑集中在 process_flush_stage_queue：
```

int MYSQL_BIN_LOG::process_flush_stage_queue(my_off_t *total_bytes_var,
                                             bool *rotate_var,
                                             THD **out_queue_var) {
 
  int no_flushes = 0;
  my_off_t total_bytes = 0;
  mysql_mutex_assert_owner(&LOCK_log);
  // 根据 innodb_flush_log_at_trx_commit 参数进行 redo log 的刷盘操作
  THD *first_seen = fetch_and_process_flush_stage_queue();
 
  // 调用 get_server_sidno() 和 Gtid_state::get_automatic_gno 生成 GTID
  assign_automatic_gtids_to_flush_group(first_seen);
  /* Flush thread caches to binary log. */
  for (THD *head = first_seen; head; head = head->next_to_commit) {
    Thd_backup_and_restore switch_thd(current_thd, head);
    /*
      flush binlog_cache_mngr的stmt_cache和trx_cache。
      flush trx_cache：
        - 生成 last_committed 和 sequence_number
        - flush GTID log event
        - 将 trx_cache 中的数据 flush 到 binlog cache 中
        - 准备提交事务后的 Binlog pos
        - 递增 prepread XID
    */
    std::pair<int, my_off_t> result = flush_thread_caches(head);
    total_bytes += result.second;
    if (flush_error == 1) flush_error = result.first;
#ifndef NDEBUG
    no_flushes++;
#endif
  }
 
  *out_queue_var = first_seen;
  *total_bytes_var = total_bytes;
  if (total_bytes > 0 &&
      (m_binlog_file->get_real_file_size() >= (my_off_t)max_size ||
       DBUG_EVALUATE_IF("simulate_max_binlog_size", true, false)))
    *rotate_var = true;
#ifndef NDEBUG
  DBUG_PRINT("info", ("no_flushes:= %d", no_flushes));
  no_flushes = 0;
#endif
  return flush_error;
}
```
redo log 刷盘的堆栈如下：
```

// 获取并清空 BINLOG_FLUSH_STAGE 和 COMMIT_ORDER_FLUSH_STAGE 队列，flush 事务到磁盘；不再阻塞 slave 的 preserve commit order 的执行
|fetch_and_process_flush_stage_queue  

// 存储引擎层将 prepare 状态的 redo log 根据 innodb_flush_log_at_trx_commit 参数刷盘
|--ha_flush_logs                      
|----innobase_flush_logs
|------log_buffer_flush_to_disk
```
SYNC 阶段
Sync 阶段的代码如下：
```
/*
  Stage #2: Syncing binary log file to disk
*/
 
if (change_stage(thd, Commit_stage_manager::SYNC_STAGE, wait_queue, &LOCK_log,
                 &LOCK_sync)) {
  DBUG_PRINT("return", ("Thread ID: %u, commit_error: %d", thd->thread_id(),
                        thd->commit_error));
  return finish_commit(thd);
}
 
/*
  - sync_counter：commit group的数量
  - get_sync_period()：获取sync_binlog参数的值
  - 如果sync stage队列中的commit group大于等于sync_binlog的值，当前leader就调用fsync()进行刷盘操作（sync_binlog_file(false)），
    在sync之前可能会进行等待，等待更多的commit group入队，等待的时间为binlog_group_commit_sync_no_delay_count或binlog_group_commit_sync_delay，默认都为0。
  - 如果sync stage队列中的commit group小于sync_binlog的值，当前leader不会调用fsync()进行刷盘也不会等待
  - 如果sync_binlog为0，每个commit group都会触发等待动作，但是不会sync
  - 如果sync_binlog为1，每个commit group都会触发等待动作，且会sync
*/
if (!flush_error && (sync_counter + 1 >= get_sync_period()))
  Commit_stage_manager::get_instance().wait_count_or_timeout(
      opt_binlog_group_commit_sync_no_delay_count,
      opt_binlog_group_commit_sync_delay, Commit_stage_manager::SYNC_STAGE);
 
final_queue = Commit_stage_manager::get_instance().fetch_queue_acquire_lock(
    Commit_stage_manager::SYNC_STAGE);
 
if (flush_error == 0 && total_bytes > 0) {
  DEBUG_SYNC(thd, "before_sync_binlog_file");
  std::pair<bool, bool> result = sync_binlog_file(false);
  sync_error = result.first;
}
 
/*
 如果sync_binlog==1,在sync stage阶段更新binog位点，并广播update信号，从库的Dump线程可以由此感知Binlog的更新
 （位点在flush stage中的process_flush_stage_queue()
                       |--flush_thread_caches()
                       |-----set_trans_pos()函数中设置）
*/
if (update_binlog_end_pos_after_sync && flush_error == 0 && sync_error == 0) {
  THD *tmp_thd = final_queue;
  const char *binlog_file = nullptr;
  my_off_t pos = 0;
 
  while (tmp_thd != nullptr) {
    if (tmp_thd->commit_error == THD::CE_NONE) {
      tmp_thd->get_trans_fixed_pos(&binlog_file, &pos);
    }
    tmp_thd = tmp_thd->next_to_commit;
  }
 
  if (binlog_file != nullptr && pos > 0) {
    update_binlog_end_pos(binlog_file, pos);
  }
}
 
DEBUG_SYNC(thd, "bgc_after_sync_stage_before_commit_stage");
 
leave_mutex_before_commit_stage = &LOCK_sync;
```

## COMMIT 阶段
Commit 阶段的代码如下：
```
  /*
    Stage #3: Commit all transactions in order.
  */
commit_stage:
  /* binlog_order_commits：是否进行 order commit，即保持 redo 和 binlog 的提交顺序一致 */
  if ((opt_binlog_order_commits || Clone_handler::need_commit_order()) &&
      (sync_error == 0 || binlog_error_action != ABORT_SERVER)) {
    if (change_stage(thd, Commit_stage_manager::COMMIT_STAGE, final_queue,
                     leave_mutex_before_commit_stage, &LOCK_commit)) {
      DBUG_PRINT("return", ("Thread ID: %u, commit_error: %d", thd->thread_id(),
                            thd->commit_error));
      return finish_commit(thd);
    }
    THD *commit_queue =
        Commit_stage_manager::get_instance().fetch_queue_acquire_lock(
            Commit_stage_manager::COMMIT_STAGE);
    DBUG_EXECUTE_IF("semi_sync_3-way_deadlock",
                    DEBUG_SYNC(thd, "before_process_commit_stage_queue"););
 
    if (flush_error == 0 && sync_error == 0)
      /* after_sync hook */
      sync_error = call_after_sync_hook(commit_queue);
 
    /*
      Commit 阶段的主要处理逻辑
    */
    process_commit_stage_queue(thd, commit_queue);
 
    /**
     * After commit stage
     */
    if (change_stage(thd, Commit_stage_manager::AFTER_COMMIT_STAGE,
                     commit_queue, &LOCK_commit, &LOCK_after_commit)) {
      DBUG_PRINT("return", ("Thread ID: %u, commit_error: %d", thd->thread_id(),
                            thd->commit_error));
      return finish_commit(thd);
    }
 
    THD *after_commit_queue =
        Commit_stage_manager::get_instance().fetch_queue_acquire_lock(
            Commit_stage_manager::AFTER_COMMIT_STAGE);
    /* after_commit hook */
    process_after_commit_stage_queue(thd, after_commit_queue);
 
    final_queue = after_commit_queue;
    mysql_mutex_unlock(&LOCK_after_commit);
  } else {
    if (leave_mutex_before_commit_stage)
      mysql_mutex_unlock(leave_mutex_before_commit_stage);
    if (flush_error == 0 && sync_error == 0)
      sync_error = call_after_sync_hook(final_queue);
  }
 
 
  /* 广播 m_stage_cond_binlog 信号变量，唤醒挂起的 follower */
  Commit_stage_manager::get_instance().signal_done(final_queue);
  DBUG_EXECUTE_IF("block_leader_after_delete", {
    const char action[] = "now SIGNAL leader_proceed";
    assert(!debug_sync_set_action(thd, STRING_WITH_LEN(action)));
  };);
 
  /*
    Finish the commit before executing a rotate, or run the risk of a
    deadlock. We don't need the return value here since it is in
    thd->commit_error, which is returned below.
  */
  (void)finish_commit(thd);
  DEBUG_SYNC(thd, "bgc_after_commit_stage_before_rotation");
 
  return thd->commit_error == THD::CE_COMMIT_ERROR;
}
```
Commit 阶段的主要处理逻辑集中在 process_commit_stage_queue 函数中：
```
void MYSQL_BIN_LOG::process_commit_stage_queue(THD *thd, THD *first) {
  mysql_mutex_assert_owner(&LOCK_commit);
#ifndef NDEBUG
  thd->get_transaction()->m_flags.ready_preempt =
      true;  // formality by the leader
#endif
  for (THD *head = first; head; head = head->next_to_commit) {
    DBUG_PRINT("debug", ("Thread ID: %u, commit_error: %d, commit_pending: %s",
                         head->thread_id(), head->commit_error,
                         YESNO(head->tx_commit_pending)));
    DBUG_EXECUTE_IF(
        "block_leader_after_delete",
        if (thd != head) { DBUG_SET("+d,after_delete_wait"); };);
    /*
      If flushing failed, set commit_error for the session, skip the
      transaction and proceed with the next transaction instead. This
      will mark all threads as failed, since the flush failed.
 
      If flush succeeded, attach to the session and commit it in the
      engines.
    */
#ifndef NDEBUG
    Commit_stage_manager::get_instance().clear_preempt_status(head);
#endif
    /*
      更新全局的 m_max_committed_transaction（用作后续事务的 last_committed），
      并初始本事务上下文的 sequence number
    */
    if (head->get_transaction()->sequence_number != SEQ_UNINIT) {
      mysql_mutex_lock(&LOCK_replica_trans_dep_tracker);
      m_dependency_tracker.update_max_committed(head);
      mysql_mutex_unlock(&LOCK_replica_trans_dep_tracker);
    }
    /*
      Flush/Sync error should be ignored and continue
      to commit phase. And thd->commit_error cannot be
      COMMIT_ERROR at this moment.
    */
    assert(head->commit_error != THD::CE_COMMIT_ERROR);
    Thd_backup_and_restore switch_thd(thd, head);
    bool all = head->get_transaction()->m_flags.real_commit;
    assert(!head->get_transaction()->m_flags.commit_low ||
           head->get_transaction()->m_flags.ready_preempt);<br>　　// Binlog Commit、Innodb Commit
    ::finish_transaction_in_engines(head, all, false);
    DBUG_PRINT("debug", ("commit_error: %d, commit_pending: %s",
                         head->commit_error, YESNO(head->tx_commit_pending)));
  }
 
  /*
    锁定 sidno，更新整组事务 的executed_gtid
    - 如果没开启 binlog，@@GLOBAL.GTID_PURGED 的值是从 executed_gtid 获取的，
      此时 @@GLOBAL.GTID_PURGED 的值和 @@GLOBAL.GTID_EXECUTED 永远是一致的，
      就不需要在记录 lost_gtids
    - 如果开启了 binlog，但是未开启 log_replica_updates，slave 的 SQL 线程或 slave worker 线程
      将自身的 GTID 更新到 executed_gtids、lost_gtids
  */
  gtid_state->update_commit_group(first);
 
  for (THD *head = first; head; head = head->next_to_commit) {
    Thd_backup_and_restore switch_thd(thd, head);
    auto all = head->get_transaction()->m_flags.real_commit;
    // 只针对外部 XA 事务，在存储引擎层将事务标记为 Prepared
    trx_coordinator::set_prepared_in_tc_in_engines(head, all);
    /*
      在存储引擎层提交之后，递减 Prepared 状态下的 XID 计数器
    */
    if (head->get_transaction()->m_flags.xid_written) dec_prep_xids(head);
  }
}
```
其中::finish_transaction_in_engines函数是主要的存储引擎层提交逻辑，相关堆栈如下：
```
|::finish_transaction_in_engines
|--trx_coordinator::commit_in_engines
|----ha_commit_low

// Binlog 层提交什么也不做（空函数）
|------binlog_commit

// 存储引擎层提交
|------innobase_commit                                
|--------innobase_commit_low
|----------trx_commit_for_mysql

// 为持久化 GTID 提前分配 update undo segment
|------------trx_undo_gtid_add_update_undo  

// 更新数据字典中被修改表的 update_time 时间
|------------trx_update_mod_tables_timestamp     

// 分配 Mini-transaction handle 和 buffer
|------------trx_commit          

// 提交 mini-transaction
|--------------trx_commit_low                         
|----------------trx_write_serialisation_history

// 更新 undo 状态：
// 对于 insert 状态从 TRX_UNDO_ACTIVE 修改为 TRX_UNDO_TO_FREE
// update 修改为 TRX_UNDO_TO_PURGE
// 如果事务为 update 还需要将 rollback segments 分配 trx no，并将其添加到 purge 队列中
|------------------trx_undo_set_state_at_finish      

//将 update undo log header 添加到 history list 开头释放一些内存对象;
|------------------trx_undo_update_cleanup  

 // 在系统事务表记录 binlog 位点
|------------------trx_sys_update_mysql_binlog_offset 
|----------------trx_commit_in_memory

//- 关闭 mvcc read view
//- 持久化 GTID
//- 释放 insert undo log
//- 唤醒后台线程开始干活，如：master thread、purge thread、page_cleaner
```
阶段转换
阶段转换的逻辑主要是由 change_stage 中的 enroll_for 函数实现：

进入队列的第一个线程会作为整组事务的 leader
后续进入队列的线程会作为整组事务的 follower
follower 线程挂起等待m_stage_cond_binlog信号变量唤醒
leader 负责提交整组事务，提交完成后，发送m_stage_cond_binlog 信号变量唤醒挂起的 follower
队列转化的主要逻辑是线程先入下个阶段的队列，然后再释放上一个阶段的 mutex，然后再获取下一个阶段的 mutex
Flush Stage 不会获取 mutex
Sync Stage 需要获取 LOCK_sync
Commit Stage 需要获取 LOCK_commit mutex
After Commit Stage 需要获取 LOCK_after_commit mutex
```
bool Commit_stage_manager::enroll_for(StageID stage, THD *thd,
                                     mysql_mutex_t *stage_mutex,
                                     mysql_mutex_t *enter_mutex) {
 // 如果队列为空，线程就是 leader
 thd->rpl_thd_ctx.binlog_group_commit_ctx().assign_ticket();
 bool leader = this->append_to(stage, thd);

 /*
  如果 FLUSH stage 队列（(BINLOG_FLUSH_STAGE 或 COMMIT_ORDER_FLUSH_STAGE）不为空，此线程就不能成为 leader。leader
  需要获取 enter_mutex
 */
 if (leader) {
   if (stage == COMMIT_ORDER_FLUSH_STAGE) {
     leader = m_queue[BINLOG_FLUSH_STAGE].is_empty();
   /*
     leader 转换的逻辑。
     session 的队列有5种：
       - Binlog flush queue: flush redo 并写 Binlog File
       - Commit order flush queue: 针对 commit order 的事务，但是会参与 group commit 的开头部分，直到引擎层的 flush。
       - Sync queue: sync transaction
       - Commit queue: 提交事务
       - After commit queue: 调用事务的 after_commit hook
    */
   } else if (stage == BINLOG_FLUSH_STAGE &&  // 当前线程是 BINLOG_FLUSH_STAGE 中的第一个线程；但是 COMMIT_ORDER_FLUSH_STAGE
                                              // 已经有了 leader，此时当前线程会挂起，等待 COMMIT_ORDER_FLUSH_STAGE 的 leader 的信号唤醒
              !m_queue[COMMIT_ORDER_FLUSH_STAGE].is_empty()) {
     /*
       当前事务是 binlog queue 中的第一个线程，但是在 commit order queue 中已经有了一个 leader。
       此时当前线程会作为 leader，而 commit order leader 会转变为 follower。
       改变 leader 的原因是 commit order leader 不能作为 binlog 线程的 leader，因为 commit order threads
       必须在 binlog threads 操作完之前离开 commit group。
       转变 leader 为 followers 的步骤如下：
       1. commit order thread 首先进入 flush stage，并成为 commit order leader。
       2. commit order leader 尝试获取 stage mutex，这可能会需要一些时间，比如 mutex 已经被上一个
       commit group的leader获取。
       3. 在此期间，一个 binlog 线程进入了 flush stage。它需要等待来自 commit order leader 的信号。
       4. commit order leader 获取了 stage mutex，然后它会检查是否有  binlog thread进入了 flush stage，
       如果发现了就转变 leader。
       5. commit order leader 给  binlog leader发送一个信号，并成为 follower，等待 commit 的完成
       （和其他 follower 的行为一致）。
       6. binlog leader 被 commit order leader 的信号唤醒并执行 group commit。
     */
     CONDITIONAL_SYNC_POINT_FOR_TIMESTAMP("before_binlog_leader_wait");
     while (thd->tx_commit_pending)
       mysql_cond_wait(&m_stage_cond_leader,
                       &m_queue_lock[BINLOG_FLUSH_STAGE]);
   }
 }

 unlock_queue(stage);

 /*
   通知下一个组提交事务进入队列
 */
 if (stage == BINLOG_FLUSH_STAGE) {
   Commit_order_manager::finish_one(thd);
   CONDITIONAL_SYNC_POINT_FOR_TIMESTAMP("after_binlog_leader_wait");
 } else if (stage == COMMIT_ORDER_FLUSH_STAGE) {
   Commit_order_manager::finish_one(thd);
 }

 /*
   当进入第一个 stage 时，可以不用获取 stage mutex
 */
 if (stage_mutex && need_unlock_stage_mutex) mysql_mutex_unlock(stage_mutex);

 /*
   如果队列非空，当前线程作为 follower 等待 leader 处理队列
 */
 if (!leader) {
   CONDITIONAL_SYNC_POINT_FOR_TIMESTAMP("before_follower_wait");
   mysql_mutex_lock(&m_lock_done);
#ifndef NDEBUG
   thd->get_transaction()->m_flags.ready_preempt = true;
   if (leader_await_preempt_status) mysql_cond_signal(&m_cond_preempt);
#endif
   // tx_commit_pending：还有事务 commit 操作未完成
   while (thd->tx_commit_pending) {
     if (stage == COMMIT_ORDER_FLUSH_STAGE) {
       mysql_cond_wait(&m_stage_cond_commit_order, &m_lock_done);
     } else {
       // follower 线程在此处挂起，等待 leader 提交事务完成后被唤醒
       mysql_cond_wait(&m_stage_cond_binlog, &m_lock_done);
     }
   }

   mysql_mutex_unlock(&m_lock_done);
   return false;
 }

#ifndef NDEBUG
 if (stage == Commit_stage_manager::SYNC_STAGE)
   DEBUG_SYNC(thd, "bgc_between_flush_and_sync");
#endif

 bool need_lock_enter_mutex = false;
 if (leader && enter_mutex != nullptr) {
   /*
     如果由于在轮替 Binlog 时已经获取了 LOCK_log，就不在需要获取 enter_mutex。
   */
   need_lock_enter_mutex = !(mysql_bin_log.is_rotating_caused_by_incident &&
                             enter_mutex == mysql_bin_log.get_log_lock());

   if (need_lock_enter_mutex)
     mysql_mutex_lock(enter_mutex);
   else
     mysql_mutex_assert_owner(enter_mutex);
 }

 // leader 转换的逻辑
 if (stage == COMMIT_ORDER_FLUSH_STAGE) {
   CONDITIONAL_SYNC_POINT_FOR_TIMESTAMP(
       "after_commit_order_thread_becomes_leader");
   lock_queue(stage);

   if (!m_queue[BINLOG_FLUSH_STAGE].is_empty()) {
     if (need_lock_enter_mutex) mysql_mutex_unlock(enter_mutex);

     THD *binlog_leader = m_queue[BINLOG_FLUSH_STAGE].get_leader();
     binlog_leader->tx_commit_pending = false;

     mysql_cond_signal(&m_stage_cond_leader);
     unlock_queue(stage);

     mysql_mutex_lock(&m_lock_done);
     /* wait for signal from binlog leader */
     CONDITIONAL_SYNC_POINT_FOR_TIMESTAMP(
         "before_commit_order_leader_waits_for_binlog_leader");
     while (thd->tx_commit_pending)
       mysql_cond_wait(&m_stage_cond_commit_order, &m_lock_done);
     mysql_mutex_unlock(&m_lock_done);

     leader = false;
     return leader;
   }
 }

 return leader;
}
```