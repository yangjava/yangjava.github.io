---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL源码主从dump线程分析
本文主要分析MySQL复制中主库dump线程的执行过程。当然dump线程分为语句gtid的dump以及文件位置dump。这里主要分析基于gtid的dump线程。也就是从库发送COM_BINLOG_DUMP_GTID指令给主库，主库dump线程入口函数为com_binlog_dump_gtid。

## 分步详解
检测阶段
权限检测，对于复制用户，需要有REPL_SLAVE_ACL权限，没有则直接拒绝
```
if (check_global_access(thd, REPL_SLAVE_ACL)) DBUG_RETURN(false);
```
获取从库相关信息，这里我们需要了解一下，dump指令格式

dump指令格式说明
从库发送到主库dump命令字符串信息格式如下：
```
命令类型COM_BINLOG_DUMP_GTID （1字节）

flags （两字节）

serverid （4字节）：从库server id

file_name_length （4字节） ：binlog文件名长度

file_name （file_name_length 字节）：binlog文件名

pos （8字节）：复制开始位置点

data_size （4字节）：表示后面从库executed gtid信息长度

data （data_size 字节）：gtid信息

```
从从库dump指令解析出上述信息。

## 如果该从库存在dump线程，则kill原来僵死dump线程
从库相关dump线程标识为：对于5.5版本之前通过server id，之后的版本通过uuid来标识。具体查找代码如下：
```
//Find_zombie_dump_thread类运算符重载
virtual bool operator()(THD *thd) {
  THD *cur_thd = current_thd;
  //找到另一个dump线程
  if (thd != cur_thd && (thd->get_command() == COM_BINLOG_DUMP ||
                         thd->get_command() == COM_BINLOG_DUMP_GTID)) {
    String tmp_uuid;
    bool is_zombie_thread = false;
    get_slave_uuid(thd, &tmp_uuid); //获得该dump线程的从库uuid
    if (m_slave_uuid.length()) { 
      //自身m_slave_uuid与找到的dump线程tmp_uuid比较，如果相同，说明找到了原来的dump线程
      is_zombie_thread = 
            (tmp_uuid.length() &&
             !strncmp(m_slave_uuid.c_ptr(), tmp_uuid.c_ptr(), UUID_LENGTH));
    } else {
      // 5.5 之前通过serverid 找到相同的dump线程
      is_zombie_thread =
            ((thd->server_id == cur_thd->server_id) && !tmp_uuid.length());
    }
    if (is_zombie_thread) {
      mysql_mutex_lock(&thd->LOCK_thd_data);
      return true;
    }
  }
  return false;
}
```
通过上述函数找到了僵死dump线程后，通过调用awake，关闭该线程。有关kill逻辑可参考《kill connection的实现简析》

构建binlog sender类，并进行相应检测。
检测主库是否是否开启binlog，没有则直接退出；
```
if (!mysql_bin_log.is_open()) {
    set_fatal_error("Binary log is not open");
    DBUG_VOID_RETURN;
  }
```
检测主库的server id是否为0，为0 则退出；
```
if (DBUG_EVALUATE_IF("simulate_no_server_id", true, server_id == 0)) {
    set_fatal_error("Misconfigured master - master server_id is 0");
    DBUG_VOID_RETURN;
  }
```
检测主库gtid_mode是否开启

从库GTID可行性检测，这一步主要包含如下两个检测操作：
从库给出的gtid中主库server_uuid的 GTID是否小于等于主库的GTID ，不是则报错；也就是从库不能不主库多事务（在主库uuid情况下）
```
if (!m_exclude_gtid->is_subset_for_sid(&gtid_executed_and_owned,
                                           gtid_state->get_server_sidno(),
                                           subset_sidno)) {
    errmsg = ER_THD(m_thd, ER_SLAVE_HAS_MORE_GTIDS_THAN_MASTER);
    global_sid_lock->unlock();
    set_fatal_error(errmsg);
    return 1;
}
```
检测主库purge gtid是否为从库executed gtid的子集，如果不是则报错；说明主库已经清理了从库拉取需要的GTID。
```
if (!gtid_state->get_lost_gtids()->is_subset(m_exclude_gtid)) {
      Gtid_set gtid_missing(gtid_state->get_lost_gtids()->get_sid_map());
      gtid_missing.add_gtid_set(gtid_state->get_lost_gtids());
      gtid_missing.remove_gtid_set(m_exclude_gtid);

      String tmp_uuid;
      get_slave_uuid(m_thd, &tmp_uuid);
      char *missing_gtids = NULL;
      gtid_missing.to_string(&missing_gtids, false, NULL);
      LogErr(WARNING_LEVEL, ER_FOUND_MISSING_GTIDS, tmp_uuid.ptr(),
             missing_gtids);
      my_free(missing_gtids);

      errmsg = ER_THD(m_thd, ER_MASTER_HAS_PURGED_REQUIRED_GTIDS);
      global_sid_lock->unlock();
      set_fatal_error(errmsg);
      return 1;
}
```
这里还有一步，如果gtid检测通过，则通过从库的gtids找到最旧一个不含有从库gtid的binlog文件。也就是dump开始文件

binlog传输观察类的传输开始位置挂钩，调用transmit_start函数，半同步插件通过该接口，实现监听新从库的加入，以及相关状态更新；

## binlog文件处理逻辑
进入binlog传输，只有当遇到错误或者被其他线程kill才退出binlog dump逻辑
```
while (!has_error() && !m_thd->killed) {
//dump 逻辑
}
```
在处理每个binlog文件开始前，首先发送一个rotate event（该event在binlog中不存在的），其作用就是告知从库接下来传输的binlog文件名以及位置。

发送一个format_description event，由于binlog格式有多个版本，该事件主要是说明格式，辅助从库解析binlog，所有不管有没有指定start pos，都会先发送一个format_description event。

检测文件头是否包含previous_gtid_event，没有就返回，说明binlog文件有问题。

进入单个文件event传输流程，这是一个循环过程，不断的从当前binlog文件中读取event，并发送给从库。具体过程在11-22详解讲解

处理完一个binlog文件之后，获取binlog index文件锁，并打开文件

更加偏移量找到下一个待处理的binlog文件，这里需要注意：由于是在dump线程中，查找下一个binlog文件是通过该文件在index文件中的便宜量来操作，具体代码如下：
```
  my_b_seek(&index_file, linfo->index_file_offset);  //偏移量设置

  linfo->index_file_start_offset = linfo->index_file_offset;
  if ((length = my_b_gets(&index_file, fname, FN_REFLEN)) <= 1) { //读取文件名字
    error = !index_file.error ? LOG_INFO_EOF : LOG_INFO_IO;
    goto err;
  }

  if (fname[0] != 0) {
    if (normalize_binlog_name(full_fname, fname, is_relay_log)) {
      error = LOG_INFO_EOF;
      goto err;
    }
    length = strlen(full_fname);
  }

  linfo->index_file_offset = my_b_tell(&index_file);  //更新偏移量到新记录以备下一次使用
```
所以如果手动修改index文件，由于dump线程中的index偏移量没有被修改，从而造成复制问题，所有如果手动修改了index文件，一定要先停止复制，然后执行flush binary logs语句，在开启复制，刷新dump线程中偏移量。

找到下一个文件后，重新设置开始位置，又跳转到第7步，处理下一个binlog文件
```
start_pos = BIN_LOG_HEADER_SIZE;
```

## event处理流程
下面详细说明event发送过程。

得到文件end pos点
这一步是event传输的关键步骤，不同情况的处理不同，下面分别讨论：

当前打开的binlog文件为活跃文件（也就是正在记录binlog的文件），并且此时dump线程读取位置 < binlog文件结束位置，则进入16-23步发送event逻辑；
当前打开的binlog文件为活跃文件，而且！(dump线程读取位置 < binlog文件结束位置），进入等待（14-15）；
当前打开的binlog文件为非活跃文件，将end pos设置为0，进入16-23步发送event逻辑。
上述逻辑的具体代码如下：
```
int Binlog_sender::get_binlog_end_pos(File_reader *reader, my_off_t *end_pos) {
  DBUG_ENTER("Binlog_sender::get_binlog_end_pos()");
  my_off_t read_pos = reader->position();  //获得当前dump读取位置

  do {
    // 获取binlog文件的结束位置，注意：该位置可能是动态的
    *end_pos = mysql_bin_log.get_binlog_end_pos();

    //如果当前binlog为非活跃状态，则直接设置end_pos 为 0 ，并返回进入event发送逻辑
    if (unlikely(!mysql_bin_log.is_active(m_linfo.log_file_name))) {
      *end_pos = 0;
      DBUG_RETURN(0);
    }

    DBUG_PRINT("info", ("Reading file %s, seek pos %llu, end_pos is %llu",
                        m_linfo.log_file_name, read_pos, *end_pos));
    DBUG_PRINT("info", ("Active file is %s", mysql_bin_log.get_log_fname()));
 //此时binlog为活跃状态，但是dump读取位置小于end pos，说明有新事物写入binlog，返回进入event发送逻辑
    if (read_pos < *end_pos) DBUG_RETURN(0);

    /* Some data may be in net buffer, it should be flushed before waiting */
    if (!m_wait_new_events || flush_net()) DBUG_RETURN(1);
    //执行到这里，说明没有新的binlog，进入等待逻辑
    if (unlikely(wait_new_events(read_pos))) DBUG_RETURN(1);
  } while (unlikely(!m_thd->killed));

  DBUG_RETURN(1);
}
```
等待逻辑
进入等待逻辑后，修改dump线程状态
```
m_thd->ENTER_COND(mysql_bin_log.get_log_cond(),
                    mysql_bin_log.get_binlog_end_pos_lock(),
                    &stage_master_has_sent_all_binlog_to_slave, &old_stage);
```
在等待过程中，根据从库心跳周期设置，会定期发送心跳包，实现如下：

等待逻辑
```
if (!timeout)  //从库心跳周期时间
    mysql_cond_wait(&update_cond, &LOCK_binlog_end_pos);
  else
    //设置了超时的条件等待
    ret = mysql_cond_timedwait(&update_cond, &LOCK_binlog_end_pos,
                               const_cast<struct timespec *>(timeout));
```
等待返回处理逻辑
```
do {
    set_timespec_nsec(&ts, m_heartbeat_period);
    ret = mysql_bin_log.wait_for_update(&ts); //等待逻辑
    if (!is_timeout(ret)) break; //如果不是超时返回，说明被唤醒，有新binlog，退出
    if (send_heartbeat_event(log_pos)) return 1; //超时返回，发送心跳event
} while (!m_thd->killed);
```

event发送逻辑
读取一个event
```
if (unlikely(read_event(reader, &event_ptr, &event_len))) DBUG_RETURN(1);
```
binlog传输观察类的event传输开始位置挂钩，调用注册的before_send_event函数
跳过从库执行的gtid，这里的跳过是跳过该gtid下一组event
```
/*
参数：
    event_ptr 指向读取的event
    in_exclude_group 表明该event是否在exclude的gtid事务内
返回值：（调用是又赋值给参数in_exclude_group）
    ture 说明在exclude gtid内，应该过滤掉
该函数的处理逻辑就是：事务所有event循环进入该函数，通过in_exclude_group标识event是否应该被过滤掉，只有通过gtid event才会重置in_exclude_group。
*/
inline bool Binlog_sender::skip_event(const uchar *event_ptr,
                                      bool in_exclude_group) {
  DBUG_ENTER("Binlog_sender::skip_event");

  uint8 event_type = (Log_event_type)event_ptr[LOG_EVENT_OFFSET];
  switch (event_type) { //GTID EVENT需要判断以更新in_exclude_group
    case binary_log::GTID_LOG_EVENT: { 
      Format_description_log_event fd_ev;
      fd_ev.common_footer->checksum_alg = m_event_checksum_alg;
      Gtid_log_event gtid_ev(reinterpret_cast<const char *>(event_ptr), &fd_ev);
      Gtid gtid;
      gtid.sidno = gtid_ev.get_sidno(m_exclude_gtid->get_sid_map());
      gtid.gno = gtid_ev.get_gno();
      DBUG_RETURN(m_exclude_gtid->contains_gtid(gtid));
    }
    case binary_log::ROTATE_EVENT:  //rotate event 必然不属于任何事务，所以放回false
      DBUG_RETURN(false);
  }
  DBUG_RETURN(in_exclude_group);  //其他event是否过滤根据参数in_exclude_group
}
```
可能由于跳过的event很多，造成在心跳周期内没有event发送给从库，从库可能认为主库宕机，为避免这种情况，在跳过event过程中，也会进行周期时间检查，定期发送心跳事件。
发送event，通过了过滤检查，即可发送event到从库
```
if (unlikely(send_packet())) DBUG_RETURN(1);
```
binlog传输观察类的event传输结束位置挂钩，调用注册的after_send_event函数