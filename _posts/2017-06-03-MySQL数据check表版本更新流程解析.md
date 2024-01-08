---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL数据check表版本更新流程解析

## MySQL的sp运行sql语句两个步骤介绍
MySQL的sp运行SQL语句需要执行2个步骤：prepare和execute。第一次执行的时候先执行prepare，进行相关语句parse、itemize、fix_fields等操作，然后才开始进行execute操作。等第二次再执行该sp的时候就直接运行execute而不需要再次进行重复的prepare操作，这样可以节省sp运行时候重复prepare的开销。但是，对于表操作就有一个问题产生，那就是如果执行第二遍的时候表的结构发生改变了，那么不进行reprepare而直接execute是会发生错误的。因此，本文章的目的在于寻找sp多次运行时候如何确认表版本更新并进行正确的操作。
```
先看下面的例子：
CREATE TABLE t1 (a INT, b VARCHAR(10));
INSERT INTO t1 VALUES (1,'11');
INSERT INTO t1 VALUES (2,'21');
MySQL> select * from t1;
+------+------+
| a    | b    |
+------+------+
|    1 | 11   |
|    2 | 21   |
+------+------+
2 rows in set (0.01 sec)
DELIMITER $$
CREATE PROCEDURE p1()
BEGIN
update t1 set b='aa' where a=1;
END $$
DELIMITER ;
MySQL> call p1; #这里第一次操作，因此会先执行update这句SQL的prepare再进行execute。
Query OK, 1 row affected (0.05 sec)

MySQL> select * from t1;
+------+------+
| a    | b    |
+------+------+
|    1 | aa   |
|    2 | 21   |
+------+------+
2 rows in set (0.01 sec)
MySQL> call p1; #这里第二次操作，直接执行update这句SQL的execute。
Query OK, 0 rows affected (13.78 sec)

#接着我们执行表结构的更新。
MySQL> alter table t1 add i int;
Query OK, 0 rows affected (0.41 sec)
Records: 0  Duplicates: 0  Warnings: 0

#然后再次执行sp，就会发现这次执行了这句SQL的prepare再进行execute。
MySQL> call p1;
Query OK, 0 rows affected (34.24 sec)
```

## 代码跟踪
现在跟踪一下这个sp看看上面在哪里check表版本并且能正确执行reprepare的。
```
#首先看一下这个sp涉及的instr。
MySQL> show procedure code p1;
+-----+---------------------------------------+
| Pos | Instruction                           |
+-----+---------------------------------------+
|   0 | stmt "update t1 set b='aa' where a=1" |
+-----+---------------------------------------+
1 row in set (0.01 sec)
可以看到这个sp只涉及了sp_instr_stmt::execute的运行，因此跟踪一下代码找到sp_lex_instr::validate_lex_and_execute_core，可以发现这个函数里面有一个无限while循环，如果发现is_invalid()的话就重新执行parse动作，如果！is_invalid()就直接执行exec_core，这个跟上面的运行步骤就对的上了。但是表结构变更后在哪里被判定为rc=true的呢，那就从reset_lex_and_exec_core这个函数继续跟踪下去。
bool sp_lex_instr::validate_lex_and_execute_core(THD *thd, uint *nextp,
                                                 bool open_tables) {
  while (true) {
    if (is_invalid() || (m_lex->has_udf() && !m_first_execution)) {
      LEX *lex = parse_expr(thd, thd->sp_runtime_ctx->sp);
    }
    bool rc = reset_lex_and_exec_core(thd, nextp, open_tables);
    if (!rc) return false;
    thd->clear_error();
    invalidate();
  }
}
#跟踪代码发现有一个check_and_update_table_version函数是用来check表版本是否一致的
#打印堆栈看一下代码调用过程：
Thread 51 "mysqld" hit Breakpoint 6, check_and_update_table_version (thd=0x7fff70001060, tables=0x7fff702c4e20, 
    table_share=0x7fff70297640) at /mysql/sql/sql_base.cc:3722
3722   if (!tables->is_table_ref_id_equal(table_share)) {
(gdb) bt
#0  check_and_update_table_version (thd=0x7fff70001060, tables=0x7fff702c4e20, table_share=0x7fff70297640)
    at /mysql/sql/sql_base.cc:3722
#1  0x0000000003340f71 in open_and_process_table (thd=0x7fff70001060, lex=0x7fff702c2650, tables=0x7fff702c4e20, 
    counter=0x7fff702c26a8, prelocking_strategy=0x7fffec2e7478, has_prelocking_list=false, ot_ctx=0x7fffec2e7368)
    at /MySQL/sql/sql_base.cc:5223
#2  0x000000000333f788 in open_tables (thd=0x7fff70001060, start=0x7fffec2e7488, counter=0x7fff702c26a8, flags=0, 
    prelocking_strategy=0x7fffec2e7478) at /mysql/sql/sql_base.cc:5968
#3  0x0000000003343c96 in open_tables_for_query (thd=0x7fff70001060, tables=0x7fff702c4e20, flags=0)
    at /MySQL/sql/sql_base.cc:6958
#4  0x0000000003514334 in Sql_cmd_dml::execute (this=0x7fff702c6138, thd=0x7fff70001060)
    at /MySQL/sql/sql_select.cc:543
#5  0x0000000003475097 in mysql_execute_command (thd=0x7fff70001060, first_level=false)
    at /MySQL/sql/sql_parse.cc:3832
#6  0x00000000033075c6 in sp_instr_stmt::exec_core (this=0x7fff70249a80, thd=0x7fff70001060, nextp=0x7fffec2eac38)
    at /MySQL/sql/sp_instr.cc:1008
#7  0x00000000033052ed in sp_lex_instr::reset_lex_and_exec_core (this=0x7fff70249a80, thd=0x7fff70001060, 
    nextp=0x7fffec2eac38, open_tables=false) at /mysql/sql/sp_instr.cc:457
#8  0x00000000033060a4 in sp_lex_instr::validate_lex_and_execute_core (this=0x7fff70249a80, thd=0x7fff70001060, 
    nextp=0x7fffec2eac38, open_tables=false) at /mysql/sql/sp_instr.cc:741
#9  0x0000000003306748 in sp_instr_stmt::execute (this=0x7fff70249a80, thd=0x7fff70001060, nextp=0x7fffec2eac38)
    at /MySQL/sql/sp_instr.cc:925
#10 0x00000000032f4d74 in sp_head::execute (this=0x7fff7018e7a0, thd=0x7fff70001060, merge_da_on_success=true)
    at /MySQL/sql/sp_head.cc:2272
#11 0x00000000032f80e1 in sp_head::execute_procedure (this=0x7fff7018e7a0, thd=0x7fff70001060, args=0x0)
    at /MySQL/sql/sp_head.cc:2977
#可以发现open_tables函数调用了这个函数，这个函数调用了ask_to_reprepare，
#在sp运行中这个ask_to_reprepare返回的是true。因此这里就解开了之前的问题，
#为何表版本更新了会return true然后重新进行parse操作。
static bool check_and_update_table_version(THD *thd, TABLE_LIST *tables,
                                           TABLE_SHARE *table_share) {
  if (!tables->is_table_ref_id_equal(table_share)) {
    /*
      Version of the table share is different from the
      previous execution of the prepared statement, and it is
      unacceptable for this SQLCOM.
    */
    if (ask_to_reprepare(thd)) return true;
    /* Always maintain the latest version and type */
    tables->set_table_ref_id(table_share);
  }
  return false;
}
```

## 知识应用
如果想开发一种动态表类型，需要每次执行sp都重新parse这个表，那就可以借用这个ask_to_reprepare函数来保证多次执行sp的时候都会进行重新reprepare。

## 总结
在MySQL的sp操作中涉及表操作的sql语句一定会执行check_and_update_table_version这个函数，每次会根据这个函数的结果来确定要不要重新parse该sql语句，如果没有版本改变就直接进行execute操作，如果有改变就重新parse，先prepare再execute，这样可以保证每次执行sp的SQL语句的时候表结构一定是最新的。









