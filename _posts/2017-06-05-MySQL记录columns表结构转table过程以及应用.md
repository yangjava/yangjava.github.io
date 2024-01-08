---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL记录columns表结构转table过程以及应用

## MySQL的dd表介绍
MySQL的dd表是用来存放表结构和各种建表信息的，客户端建的表都存在mysql.table和mysql.columns表里，还有一个表mysql.column_type_elements比较特殊，用来存放SET和ENUM类型的字段集合值信息。看一下下面这张表的mysql.columns表和mysql.column_type_elements信息。为了缩短显示长度，这里只展示几个重要的值。
```
#建表：
CREATE TABLE t1(id int  not null auto_increment primary key,col1 number,col2 VARCHAR(100),col3 pls_integer,
col4 enum('x','y') default 'x',col5 set('x1','y1'))  partition by hash(id) partitions 3;
SET SESSION debug='+d,skip_dd_table_access_check';
mysql> select name,ordinal_position,type,default_value_utf8,options,column_type_utf8 from mysql.columns where table_id=383;
+-------------+------------------+-----------------------+--------------------+-------------------+------------------+
| name        | ordinal_position | type                  | default_value_utf8 | options           | column_type_utf8 |
+-------------+------------------+-----------------------+--------------------+-------------------+------------------+
| col1        |                2 | MYSQL_TYPE_NEWDECIMAL | NULL               | interval_count=0; | decimal(65,0)    |
| col2        |                3 | MYSQL_TYPE_VARCHAR    | NULL               | interval_count=0; | varchar(100)     |
| col3        |                4 | MYSQL_TYPE_LONG       | NULL               | interval_count=0; | int              |
| col4        |                5 | MYSQL_TYPE_ENUM       | x                  | interval_count=2; | enum('x','y')    |
| col5        |                6 | MYSQL_TYPE_SET        | NULL               | interval_count=2; | set('x1','y1')   |
| DB_ROLL_PTR |                8 | MYSQL_TYPE_LONGLONG   | NULL               | NULL              |                  |
| DB_TRX_ID   |                7 | MYSQL_TYPE_INT24      | NULL               | NULL              |                  |
| id          |                1 | MYSQL_TYPE_LONG       | NULL               | interval_count=0; | int              |
+-------------+------------------+-----------------------+--------------------+-------------------+------------------+
8 rows in set (0.00 sec)
```
mysql.columns表说明如下：

ordinal_position是该字段在表里的偏移量，这里多了3个字段，DB_ROLL_PTR、DB_TRX_ID、id是用来执行undo的，记录了字段的版本信息。
default_value_utf8是用来保存默认值的。options里面有interval_count用来保存集合类型的数值数的。columns表的options的key一共有如下几种：
```
static const std::set<String_type> default_valid_option_keys = {
    "column_format", "geom_type",         "interval_count", "not_secondary",
    "storage",       "treat_bit_as_char", "zip_dict_id",    "is_array"};
```

```
mysql>  select * from mysql.column_type_elements where column_id=4286;
+-----------+---------------+------+
| column_id | element_index | name |
+-----------+---------------+------+
|      4286 |             1 | x    |
|      4286 |             2 | y    |
+-----------+---------------+------+
2 rows in set (0.01 sec)
#这里的column_id=4286是col4的id值，x和y分别对应了set定义时候的2个集合值。
```

## 代码跟踪
现在重新启动数据库，跟踪一下这个columns表怎么转为代码里面的TABLE的field对象。首先找到表的dd信息然后打开表获取field信息。
```
mysql> select * from t1;
```
输入该命令后找到columns表转为field的代码：
```
#0  fill_column_from_dd (
    thd=0x555558b35a06 <std::char_traits<char>::compare(char const*, char const*, unsigned long)+61>, 
    share=0x7fffe83f1b60, 
    col_obj=0x555558bb0a5e <std::__cxx11::basic_string<char, std::char_traits<char>, Stateless_allocator<char, dd::String_type_alloc, My_free_functor> >::compare(std::__cxx11::basic_string<char, std::char_traits<char>, Stateless_allocator<char, dd::String_type_alloc, My_free_functor> > const&) const+142>, 
    null_pos=0x7fffe83f1880 "\005", null_bit_pos=32767, rec_pos=0x7fff2c05ac10 "explicit_encryption", 
    field_nr=0) at /mysql/sql/dd_table_share.cc:955
#1  0x00005555593c4c17 in fill_columns_from_dd (thd=0x7fff2c006890, share=0x7fff2cbf19e8, 
    tab_obj=0x7fff2cbb9b38) at /mysql/sql/dd_table_share.cc:1235
#2  0x00005555593c9e54 in open_table_def (thd=0x7fff2c006890, share=0x7fff2cbf19e8, table_def=...)
    at /mysql/sql/dd_table_share.cc:2408
#3  0x0000555558e76a13 in get_table_share (thd=0x7fff2c006890, db=0x7fff2cbeeff0 "db1", 
    table_name=0x7fff2cc03210 "t1", key=0x7fff2cbeed87 "db1", key_length=7, open_view=true, 
    open_secondary=false) at /mysql/sql/sql_base.cc:801
#4  0x0000555558e76f08 in get_table_share_with_discover (thd=0x7fff2c006890, table_list=0x7fff2cbee9b8, 
    key=0x7fff2cbeed87 "db1", key_length=7, open_secondary=false, error=0x7fffe83f1ea4)
    at /mysql/sql/sql_base.cc:889
#5  0x0000555558e7cd34 in open_table (thd=0x7fff2c006890, table_list=0x7fff2cbee9b8, 
    ot_ctx=0x7fffe83f2380) at /mysql/sql/sql_base.cc:3230
#6  0x0000555558e81769 in open_and_process_table (thd=0x7fff2c006890, lex=0x7fff2c01bdf0, 
    tables=0x7fff2cbee9b8, counter=0x7fff2c01be48, prelocking_strategy=0x7fffe83f2408, 
    has_prelocking_list=false, ot_ctx=0x7fffe83f2380)
    at /mysql/sql/sql_base.cc:5118
#7  0x0000555558e833bd in open_tables (thd=0x7fff2c006890, start=0x7fffe83f23f0, counter=0x7fff2c01be48, 
    flags=0, prelocking_strategy=0x7fffe83f2408)
    at /mysql/sql/sql_base.cc:5928
#8  0x0000555558e85626 in open_tables_for_query (thd=0x7fff2c006890, tables=0x7fff2cbee9b8, flags=0)
    at /mysql/sql/sql_base.cc:6904
#9  0x0000555559075720 in Sql_cmd_dml::prepare (this=0x7fff2cbef400, thd=0x7fff2c006890)
    at /mysql/sql/sql_select.cc:372
#10 0x00005555590760bf in Sql_cmd_dml::execute (this=0x7fff2cbef400, thd=0x7fff2c006890)
    at /mysql/sql/sql_select.cc:527
#11 0x0000555558fedc8e in mysql_execute_command (thd=0x7fff2c006890, first_level=true)
    at /mysql/sql/sql_parse.cc:4794
#12 0x0000555558fefe25 in dispatch_sql_command (thd=0x7fff2c006890, parser_state=0x7fffe83f3990, 
    update_userstat=false) at /mysql/sql/sql_parse.cc:5399
#13 0x0000555558fe52d3 in dispatch_command (thd=0x7fff2c006890, com_data=0x7fffe83f4b70, 
    command=COM_QUERY) at /mysql/sql/sql_parse.cc:2000
#14 0x0000555558fe3643 in do_command (thd=0x7fff2c006890)
    at /mysql/sql/sql_parse.cc:1448
#15 0x000055555920e200 in handle_connection (arg=0x555560a65790)
    at /mysql/sql/conn_handler/connection_handler_per_thread.cc:307
#16 0x000055555ae36375 in pfs_spawn_thread (arg=0x5555608a2e20)
    at /mysql/storage/perfschema/pfs.cc:2899
#17 0x00007ffff77a6609 in start_thread (arg=<optimized out>) at pthread_create.c:477
#18 0x00007ffff76cb163 in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
```
#fill_column_from_dd函数里面最重要的是make_field函数，把字段从dd::Column转为table的field然后赋值给TABLE_SHARE。
reg_field = make_field(*col_obj, charset, share, rec_pos, null_pos, null_bit_pos);

## 知识应用
session每次获取表的信息都是在第一次打开表的时候就做好了，下次如果表没有变化就从Table_cache直接获取表信息。现在假设我们要改col4字段的集合值又不想通过alter语句来执行，那就可以直接对dd表进行操作。注意，该操作对生产环境有很大风险，这里只用来进行知识实践，不能用来在生产环境实际操作。

把col4的x,y值改为a,b：
首先试着插入col4=x的记录，现在还没改dd表插入成功。
```
mysql> insert into t1 values(1,1,'aa',1,'x','x1');
Query OK, 1 row affected (0.03 sec)
```
接着开始改col4的集合值：
```
mysql> SET SESSION debug='+d,skip_dd_table_access_check';
Query OK, 0 rows affected (0.02 sec)
mysql> update mysql.columns set default_value_utf8='a' ,column_type_utf8='enum(\'a\',\'b\'))' where table_id=383 and name='col4';
Query OK, 1 row affected (0.03 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select name,ordinal_position,type,default_value_utf8,options,column_type_utf8 from mysql.columns where table_id=383;
+-------------+------------------+-----------------------+--------------------+-------------------+------------------+
| name        | ordinal_position | type                  | default_value_utf8 | options           | column_type_utf8 |
+-------------+------------------+-----------------------+--------------------+-------------------+------------------+
| col1        |                2 | MYSQL_TYPE_NEWDECIMAL | NULL               | interval_count=0; | decimal(65,0)    |
| col2        |                3 | MYSQL_TYPE_VARCHAR    | NULL               | interval_count=0; | varchar(100)     |
| col3        |                4 | MYSQL_TYPE_LONG       | NULL               | interval_count=0; | int              |
| col4        |                5 | MYSQL_TYPE_ENUM       | a                  | interval_count=2; | enum('a','b'))   |集合值已改
| col5        |                6 | MYSQL_TYPE_SET        | NULL               | interval_count=2; | set('x1','y1')   |
| DB_ROLL_PTR |                8 | MYSQL_TYPE_LONGLONG   | NULL               | NULL              |                  |
| DB_TRX_ID   |                7 | MYSQL_TYPE_INT24      | NULL               | NULL              |                  |
| id          |                1 | MYSQL_TYPE_LONG       | NULL               | interval_count=0; | int              |
+-------------+------------------+-----------------------+--------------------+-------------------+------------------+
8 rows in set (0.00 sec)

mysql> update mysql.column_type_elements set name='a' where column_id=4286 and element_index=1;
Query OK, 1 row affected (0.03 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> update mysql.column_type_elements set name='b' where column_id=4286 and element_index=2;
Query OK, 1 row affected (0.03 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql>  select * from mysql.column_type_elements where column_id=4286;
+-----------+---------------+------+
| column_id | element_index | name |
+-----------+---------------+------+
|      4286 |             1 | a    |集合值已改
|      4286 |             2 | b    |集合值已改
+-----------+---------------+------+
2 rows in set (0.00 sec)
```
现在再插入一条col4=x的记录发现还是成功的，这是因为t1没有重新从dd表转为TABLE信息，需要重启后再试。
```
mysql> insert into t1 values(2,1,'aa',1,'x','x1');
Query OK, 1 row affected (0.02 sec)
```
重启数据库，然后登录。再次插入col4=x发现报错了，因为这时候的TABLE信息是重新从dd表转化的。
```
mysql> insert into t1 values(2,1,'aa',1,'x','x1');
ERROR 1265 (01000): Data truncated for column 'col4' at row 1
```
插入col4=a的记录成功，说明更改成功。
```
mysql> insert into t1 values(3,1,'aa',1,'a','x1');
Query OK, 1 row affected (0.02 sec)
```
查看建表信息,发现已经成功更改。
```
mysql> show create table t1;
+-------+-------------------------+
| Table | Create Table     |
+-------+-------------------------+
| t1    | CREATE TABLE `t1` (
  `id` int NOT NULL AUTO_INCREMENT,
  `col1` decimal(65,0) DEFAULT NULL,
  `col2` varchar(100) DEFAULT NULL,
  `col3` int DEFAULT NULL,
  `col4` enum('a','b') DEFAULT 'a',更改成功
  `col5` set('x1','y1') DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
/*!50100 PARTITION BY HASH (`id`)
PARTITIONS 3 */ 
+-------+-------------------------+
```

实际上更改表结构如果通过alter命令来改流程跟上面也是一样的，也是通过更新dd表来实现表结构的变更，这里只是从更底层来介绍。以上的操作在实际生产中不能直接操作，风险很大，会影响现有的记录和相关的功能。这里只是作为一个案例来更好的说明dd的工作流程，帮助大家遇到问题知道怎么从底层排查。