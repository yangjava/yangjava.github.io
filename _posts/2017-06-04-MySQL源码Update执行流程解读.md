---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL源码Update执行流程解读

## update跟踪执行配置
使用内部程序堆栈跟踪工具path_viewer，跟踪mysql update 一行数据的执行过程，配置执行脚本：call_update.sh
```
DROP DATABASE IF EXISTS d1;
CREATE DATABASE d1;
use d1;
drop table if exists test;
CREATE TABLE test (c0 int NOT NULL AUTO_INCREMENT,c1 date DEFAULT NULL,c2 time DEFAULT NULL,
  c3 datetime DEFAULT NULL,
  c4 year DEFAULT NULL,
  c5 int DEFAULT NULL,
  c6 decimal(10,6) DEFAULT NULL,
  c7 double DEFAULT NULL,
  c8 float DEFAULT NULL,
  c9 varchar(255) DEFAULT NULL,
  PRIMARY KEY (c0)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

INSERT INTO test VALUES ('1', '2021-04-26', '15:36:37', NULL, '2021', NULL, '6.000000', '7.1', '118.169', 'a8.168111');
INSERT INTO test VALUES ('2', '2021-04-28', '15:36:37', '2021-04-27 15:36:41', '2021', '6', '7877.126000', '8.1', '119.187', 'a9.16');
INSERT INTO test VALUES ('3', '2021-04-29', '15:36:37', '2021-04-27 15:36:41', '2021', '6', '7877.126890', '8.1', '119.187', 'a9.1682');
INSERT INTO test VALUES ('4', '2021-04-30', '15:36:37', '2021-04-27 15:36:41', '2021', '6', '7877.126890', '8.1', '119.187', 'a9.168333');
EOF

sleep 1

mysql -h127.0.0.1 -P3300 -uroot <<EOF 
use d1;

\! ./util/start_trace.sh

update test set c7=10.1 where c0=1;

\! ./util/end_trace.sh
EOF

# analyse the data
./util/seperate.sh $TOPIC 10000 5
~                                                    
```
执行后，生成执行结果：
```
path_viewer/call_update$ ls
362688_5.txt  362689.dat    362690.inf    362692_5.txt  362699.dat    362706.inf    362717_L.txt  mark.txt
362688.dat    362689.inf    362691_5.txt  362692.dat    362699.inf    362717_5.txt  362717_M.txt  TDATA.trc
362688.inf    362690_5.txt  362691.dat    362692.inf    362706_5.txt  362717.dat    362717_S.txt  THREADS.inf
362689_5.txt  362690.dat    362691.inf    362699_5.txt  362706.dat    362717.inf    FUNC_DEF
```
查看SQL执行的函数调用过程362717_5.txt
```
 > my_net_set_read_timeout(NET*, unsigned int)
        > vio_timeout(Vio*, unsigned int, int)
            > vio_socket_timeout(Vio*, unsigned int, bool)
            < vio_socket_timeout(Vio*, unsigned int, bool)
        < vio_timeout(Vio*, unsigned int, int)
    < my_net_set_read_timeout(NET*, unsigned int)
    > dispatch_command(THD*, COM_DATA const*, enum_server_command)
        > PROFILING::start_new_query(char const*)
        < PROFILING::start_new_query(char const*)
        > inline_mysql_refine_statement(PSI_statement_locker*, unsigned int)
            > pfs_refine_statement_v2(PSI_statement_locker*, unsigned int)
                > find_statement_class(unsigned int)
                < find_statement_class(unsigned int)
            < pfs_refine_statement_v2(PSI_statement_locker*, unsigned int)
        < inline_mysql_refine_statement(PSI_statement_locker*, unsigned int)
        > THD::set_command(enum_server_command)
            > pfs_set_thread_command_vc(int)
            < pfs_set_thread_command_vc(int)
        < THD::set_command(enum_server_command)
        > THD::clear_slow_extended()
            > inline_mysql_mutex_unlock(mysql_mutex_t*, char const*, unsigned int)
                > my_mutex_unlock(my_mutex_t*)
                    > native_mutex_unlock(pthread_mutex_t*)
                    < native_mutex_unlock(pthread_mutex_t*)
                < my_mutex_unlock(my_mutex_t*)
            < inline_mysql_mutex_unlock(mysql_mutex_t*, char const*, unsigned int)
        < THD::clear_slow_extended()
```

```
.................................

                   < MYSQLparse(THD*, Parse_tree_root**)
                    > LEX::make_sql_cmd(Parse_tree_root*)
                        > PT_update::make_cmd(THD*) 
                            > Parse_context::Parse_context(THD*, Query_block*)
                            < Parse_context::Parse_context(THD*, Query_block*)
                            > bool (anonymous namespace)::contextualize_safe<Parse_context, PT_with_clause*>(Parse_context*, PT_with_clause*)
                            < bool (anonymous namespace)::contextualize_safe<Parse_context, PT_with_clause*>(Parse_context*, PT_with_clause*)
                            > PT_table_factor_table_ident::contextualize(Parse_context*)
                                > Query_block::add_table_to_list(THD*, Table_ident*, char const*, unsigned long, thr_lock_type, enum_mdl_type, List<Index_hint>*, List<String>*, MYSQL_LEX_STRING*, Parse_context*)
                                    > check_table_name(char const*, unsigned long)
                                    < check_table_name(char const*, unsigned long)
                                    > memdup_root(MEM_ROOT*, void const*, unsigned long)
.................................
             > Sql_cmd_dml::execute(THD*)  # 执行DML
                    > is_timer_applicable_to_statement(THD*)
                    < is_timer_applicable_to_statement(THD*)
                    > THD::push_internal_handler(Internal_error_handler*)
                    < THD::push_internal_handler(Internal_error_handler*)
                    > Sql_cmd_dml::prepare(THD*)
                        > Sql_cmd_update::precheck(THD*)
                            > check_one_table_access(THD*, unsigned long, TABLE_LIST*)
                                > check_single_table_access(THD*, unsigned long, TABLE_LIST*, bool)
                                    > check_access(THD*, unsigned long, char const*, unsigned long*, GRANT_INTERNAL_INFO*, bool, bool)
                                        > get_cached_schema_access(GRANT_INTERNAL_INFO*, char const*)
                                            > ACL_internal_schema_registry::lookup(char const*)
                                            < ACL_internal_schema_registry::lookup(char const*)
```

```
.................................
                            < THD::is_dml_gtid_compatible(bool, bool, bool)
                        < THD::decide_logging_format(TABLE_LIST*)
                    < lock_tables(THD*, TABLE_LIST*, unsigned int, unsigned int)
                    > Sql_cmd_update::execute_inner(THD*)
                        > Sql_cmd_update::update_single_table(THD*)
                            > Query_expression::set_limit(THD*, Query_block*)
                                > Query_block::get_offset(THD*)
                                < Query_block::get_offset(THD*)
                                > Query_block::get_limit(THD*)
                                < Query_block::get_limit(THD*)
                            < Query_expression::set_limit(THD*, Query_block*)
                            > COPY_INFO::get_function_default_columns(TABLE*)
                                > allocate_column_bitmap(TABLE*, MY_BITMAP**)
                                    > multi_alloc_root(MEM_ROOT*, ...)
                                    < multi_alloc_root(MEM_ROOT*, ...)
                                < allocate_column_bitmap(TABLE*, MY_BITMAP**)
                                > bitmap_is_clear_all(MY_BITMAP const*)
                                < bitmap_is_clear_all(MY_BITMAP const*)
```

```
.................................
                            > handler::ha_fast_update(THD*, mem_root_deque<Item*>&, mem_root_deque<Item*>&, Item*)
                            < handler::ha_fast_update(THD*, mem_root_deque<Item*>&, mem_root_deque<Item*>&, Item*)
                            > IndexRangeScanIterator::Read()
                                > QUICK_RANGE_SELECT::get_next()
                                    > handler::ha_multi_range_read_next(char**)
                                        > ha_innobase::multi_range_read_next(char**)
> DsMrr_impl::dsmrr_next(char**)
                                                > handler::multi_range_read_next(char**)
                                                    > quick_range_seq_next(void*, KEY_MULTI_RANGE*)
                                                    < quick_range_seq_next(void*, KEY_MULTI_RANGE*)
                                                    > ha_innobase::read_range_first(key_range const*, key_range const*, bool, bool)
                                                        > handler::read_range_first(key_range const*, key_range const*, bool, bool)
.................................

                            > handler::ha_update_row(unsigned char const*, unsigned char*)
                                > handler::mark_trx_read_write()
                                < handler::mark_trx_read_write()
                                > pfs_start_table_io_wait_v1(PSI_table_locker_state*, PSI_table*, PSI_table_io_operation, unsigned int, char const*, unsigned int)
                                < pfs_start_table_io_wait_v1(PSI_table_locker_state*, PSI_table*, PSI_table_io_operation, unsigned int, char const*, unsigned int)
                                > ha_innobase::update_row(unsigned char const*, unsigned char*)
                                    > handler::ha_statistic_increment(unsigned long long System_status_var::*) const
                                    < handler::ha_statistic_increment(unsigned long long System_status_var::*) const
                                    > row_get_prebuilt_update_vector(row_prebuilt_t*)
                                        > row_create_update_node_for_mysql(dict_table_t*, mem_block_info_t*)
                                            > upd_node_create(mem_block_info_t*)
                                                > mem_heap_zalloc(mem_block_info_t*, unsigned long)
                                                    > mem_heap_alloc(mem_block_info_t*, unsigned long)
                                                        > mem_block_get_len(mem_block_info_t*)
                                                        < mem_block_get_len(mem_block_info_t*)
                                                        > mem_block_get_free(mem_block_info_t*)
                                                        < mem_block_get_free(mem_block_info_t*)
                                                        > mem_block_get_free(mem_block_info_t*)
                                                        < mem_block_get_free(mem_block_info_t*)
                                                        > mem_block_set_free(mem_block_info_t*, unsigned long)
                                                        < mem_block_set_free(mem_block_info_t*, unsigned long)
                                                    < mem_heap_alloc(mem_block_info_t*, unsigned long)
                                                < mem_heap_zalloc(mem_block_info_t*, unsigned long)
                                                > mem_heap_create_func(unsigned long, unsigned long)
.................................

                                                > __gnu_cxx::__exchange_and_add(int volatile*, int)
                                                < __gnu_cxx::__exchange_and_add(int volatile*, int)
                                            < __gnu_cxx::__exchange_and_add_dispatch(int*, int)
                                        < QUICK_RANGE_SELECT::~QUICK_RANGE_SELECT()
                                    < QUICK_RANGE_SELECT::~QUICK_RANGE_SELECT()
                                < QEP_shared_owner::qs_cleanup()
                            < QEP_TAB::cleanup()
                        < Sql_cmd_update::update_single_table(THD*)
                    < Sql_cmd_update::execute_inner(THD*)
                    > THD::pop_internal_handler()
                    < THD::pop_internal_handler()
                    > Query_expression::cleanup(THD*, bool)
                        > Query_block::cleanup(THD*, bool)
                        < Query_block::cleanup(THD*, bool)
                    < Query_expression::cleanup(THD*, bool)                                                                                                                                               
```

## 执行分析
主要过程函数
```
do_command(THD*)      --从连接中读取命令
dispatch_sql_command   --分发命令
THD::sql_parser()      --SQL引擎层，词法语法分析
parse_sql       --SQL转换为AST语句
  LEX::make_sql_cmd(Parse_tree_root*)  --解析树翻译成AST语法树
   PT_update::make_cmd(THD*)   --更新树节点翻译成AST语法树
   
mysql_execute_command   --命令执行
Sql_cmd_dml::execute
  Sql_cmd_dml::prepare(THD*)  --引用消解
   Sql_cmd_update::precheck(THD*)  --更新语句实际执行的引用消解

Sql_cmd_update::execute_inner(THD*)  --SQL引擎层，调用存储引擎接口执行
Sql_cmd_update::update_single_table(THD*)
optimize_cond  --执行优化器优化路径
 handler::ha_fast_update 
ha_innobase::update_row   --innodb更新buffer pool 中table 的row
trans_commit_stmt(THD*, bool)
```
innoDB关键更新执行过程
```
ha_innobase::update_row：
row_get_prebuilt_update_vector
calc_row_difference
row_update_for_mysql
 row_upd_step
 row_upd   --执行更新
  btr_pcur_t::restore_position
  rec_get_offsets_func
  btr_cur_update_in_place
  btr_cur_upd_lock_and_undo
  trx_undo_report_row_operation
  trx_undo_create
  trx_undo_seg_create
   _fil_io
   Fil_shard::do_io
   pfs_os_aio_func
  mtr_t::Command::execute
   log_buffer_reserve
   mtr_t::Command::add_dirty_blocks_to_flush_list ---脏数据快准备刷入磁盘
  trx_undo_page_report_modify
row_upd_rec_sys_fields 
row_upd_rec_in_place
btr_cur_update_in_place_log
```
更新过程描述:

InnoDB 存储引擎，当更新一条数据时，会先更新 buffer pool 中的数据，Master Thread 刷新缓冲池中的脏页数据到磁盘中；更新一条记录前，会先生成一条undolog，记录更新操作，再生成redolog；事务提交时，将事务生成的redolog刷入磁盘。

## 执行总结
update执行流程
1.执行语句连接数据库
2.分析器通过词法、语法分析知道这是一条更新语句
3.优化器确定执行路径
4.执行器具体执行，找到这一行，更新数据，然后通过InnoDB存储具体更新操作
5.InnoDB 存储引擎更新内存数据后写入磁盘，过程中会更新各个日志文件binlog、undolog、redolog










































