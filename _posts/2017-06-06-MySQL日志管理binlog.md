---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL日志管理binlog

## 概述
mysql开启binlog后，对于事务引擎的DML操作，会以内部二阶段提交的方式对binlog和date文件进行写入。在非正常关闭后，重启mysql时，会根据binlog对事务引擎进行未完成事务进行恢复。

只有在mysql开启了binlog，并且同时存在事务引擎时，才会设置total_ha_2pc，即binlog和事务引擎的内部二阶段提交。
只有当mysqld非正常关闭时，当前使用的binlog的flag才会是正在使用状态，这时才需要通过binlog对事务引擎进行未完成事务的恢复。

## 源码分析
初始化
在初始化阶段，根据是否开启了二阶段提交（存在binlog和事务引擎），判断是否需要依据binlog进行recover。
```
......
init_server_components
| plugin_init                                  // 初始化各类插件
  | plugin_initialize   
    | ha_initialize_handlerton    
        if (hton->state == SHOW_OPTION_YES ...
        ...                                
        total_ha_2pc++;                    // 在初始化innodb、binlog插件时设置二阶段提交标志
| if (total_ha_2pc > 1 || (1 == total_ha_2pc && opt_bin_log))          
    tc_log= &mysql_bin_log;
  // 调用binlog recover函数，即MYSQL_BIN_LOG::open_binlog
| if (tc_log->open(opt_bin_log ? opt_bin_logname : opt_tc_log_file))    
```

### open_binlog函数
以open_binlog函数来作为binlog recover，这里open_binlog是被重载的函数，注意输入的参数。
```
MYSQL_BIN_LOG::open_binlog(const char *opt_name)
  // 在index里面查找第一个binlog文件
| if ((error= find_log_pos(&log_info, NullS, true/*need_lock_index=true*/)))
  // 查找异常关闭前使用的最后一个binlog
|  while (!(error= find_next_log(&log_info, true/*need_lock_index=true*/))
  // 打开最后一个binlog
|  if ((file= open_binlog_file(&log, log_name, &errmsg)) < 0)
|  if ((ev= Log_event::read_log_event(&log, 0, &fdle, ......                           // 开始读取event
   ev->common_header->flags & LOG_EVENT_BINLOG_IN_USE_F        // 判断是否为非正常关闭
   ...)
|       recover(&log, (Format_description_log_event *)ev, &valid_pos);
| if (valid_pos > 0)
    // 根据recover返回的有效size，对binlog进行删除操作
    my_chsize(file, valid_pos, 0, MYF(MY_WME)))
```
recover函数
根据最后一个binlog的日志内容，对事务引擎未完成的事务进行recover
```
MYSQL_BIN_LOG::recover(... valid_pos...)
| HASH xids;
| my_hash_init(&xids, &my_charset_bin, .....                    
| while ((ev= Log_event::read_log_event(log, 0, fdle, TRUE))        // 依次读取binlog中的even
  {
    //根据一定的规则校验binlog中内容的正确性
    //如果读取到异常binlog，那么会删除后续所有的内容
    ......     
    ......                                                                                       
    case :binary_log::XID_EVENT
    if (!x || my_hash_insert(&xids, x))                                       // 将binlog中记录的xid，插入到hash 桶中                           
  }
  // 将正常的binlog size传出
  // 后续会根据这个正常对binlog进行truncate,删除异常的内容
| *valid_pos= my_b_tell(log)                                                            
| if (total_ha_2pc > 1 && ha_recover(&xids))
    // 每引擎依次执行xarecover_handlerton
    | plugin_foreach(NULL, xarecover_handlerton,
```
xarecover_handlerton函数
对每个插件根据在binlog中读取的xid进行recover，传入参数为最后一个binlog中记录的所有xid
```
xarecover_handlerton(... xids ...)
  // 在初始化时，更新过total_ha_2pc并且存在recover函数的引擎
  // 这里以innodb进行说明
| if (hton->state == SHOW_OPTION_YES && hton->recover)
    // 调用引擎插件的recover函数
    // innodb会解析redo log，读取出所有处于prepare状态的事务，返回事务的xid
    while ((got= hton->recover(hton, info->list, info->len)) > 0)
        // 遍历引擎插件返回的xid数组
        // innodb的redo log中处于prepare状态的事务xid 
        for (int i= 0; i < got; i++)
            // 在最后一个binlog中读取的xid的hash桶（传入参数）查找xid
            // 如果找到了，说明该事务记录了binlog，则commit，找不到则rollback
            if ( my_hash_search(info->commit_list, (uchar *)&x, sizeof(x)) != 0 :
                hton->commit_by_xid(hton, info->list + i);
            else
                hton->rollback_by_xid(hton, info->list + i);
```
## 总结
如果mysql开启了binlog，那么事务引擎在宕机重启后，redo log中记录的所有的处于prepare状态的事务，会根据是否记录binlog来决定是否提交。如果记录了binlog则提交，反之则进行回滚。