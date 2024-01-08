---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL数据dump详解
在进行数据库备份的时候主要分为了逻辑备份和物理备份这两种方式。在数据迁移和备份恢复中使用mysqldump将数据生成sql进行保存是最常用的方式之一。

本文将围绕着mysqldump的使用，工作原理，以及对于InnoDB和MyISAM两种不同引擎如何实现数据一致性这三个方面进行介绍。

## mysqldump 简介
mysqldump是MySQL自带的逻辑备份工具。
它的备份原理是通过协议连接到 MySQL数据库，将需要备份的数据查询出来，
将查询出的数据转换成对应的insert语句，当我们需要还原这些数据时，
只要执行这些insert语句，即可将对应的数据还原。

## 备份的命令
命令的格式
```
1.mysqldump [选项] 数据库名 [表名] > 脚本名
2.mysqldump [选项] --数据库名 [选项 表名] > 脚本名
3.mysqldump [选项] --all-databases [选项]  > 脚本名
```

选项说明
```
参数名	缩写	含义
--host	-h	服务器IP地址
--port	-P	服务器端口号
--user	-u	MySQL 用户名
--pasword	-p	MySQL 密码
--databases		指定要备份的数据库
--all-databases		备份mysql服务器上的所有数据库
--compact		压缩模式，产生更少的输出
--comments		添加注释信息
--complete-insert		输出完成的插入语句
--lock-tables		备份前，锁定所有数据库表
--no-create-db/--no-create-info		禁止生成创建数据库语句
--force		当出现错误时仍然继续备份操作
--default-character-set		指定默认字符集
--add-locks		备份数据库表时锁定数据库表
```

## 还原的命令
系统行命令
```
mysqladmin -uroot -p create db_name 
mysql -uroot -p  db_name < /backup/mysqldump/db_name.db

注：在导入备份数据库前，db_name如果没有，是需要创建的； 而且与db_name.db中数据库名是一样的才可以导入。
```

source方式
```
mysql > use db_name;
mysql > source /backup/mysqldump/db_name.db;
```

## mysqldump实现的原理
备份流程如下
```
1.调用FWRL（flush tables with read lock）,全局禁止读写
2.开启快照读，获取此期间的快照（仅仅对innodb起作用）
3.备份非innodb表数据（*.frm,*.myi,*.myd等）
4.非innodb表备份完毕之后，释放FTWRL
5.逐一备份innodb表数据
6.备份完成
```

执行mysqldump，分析备份日志
```
#  执行语句
[root@localhost backup]# mysqldump -uroot -proot -h127.0.0.1 --all-databases --single-transaction --routines --events --triggers --master-data=2 --hex-blob --default-character-set=utf8mb4 --flush-logs --quick > all.sql
mysqldump: [Warning] Using a password on the command line interface can be insecure.
[root@localhost ~]# tail -f  /var/lib/mysql/localhost.log
第一步：
  FLUSH /*!40101 LOCAL */ TABLES
# 这里是刷新表


第二步：
  FLUSH TABLES WITH READ LOCK
# 因为开启了--master-data=2，这时就需要flush tables with read lock锁住全库，
  记录当时的master_log_file和master_log_pos点
  这里有一个疑问？
  执行flush tables操作，并加一个全局读锁，那么以上两个命令貌似是重复的，
  为什么不在第一次执行flush tables操作的时候加上锁呢？
  简而言之，是为了避免较长的事务操作造成FLUSH TABLES WITH READ LOCK操作迟迟得不到
  锁，但同时又阻塞了其它客户端操作。
  
  
第三步：
  SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ
#  --single-transaction参数的作用，设置事务的隔离级别为可重复读，
  即REPEATABLE READ，这样能保证在一个事务中所有相同的查询读取到同样的数据，
  也就大概保证了在dump期间，如果其他innodb引擎的线程修改了表的数据并提交，
  对该dump线程的数据并无影响，然而这个还不够，还需要看下一条


第四步：
  START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */
# 获取当前数据库的快照，这个是由mysqldump中--single-transaction决定的。
# WITH CONSISTENT SNAPSHOT能够保证在事务开启的时候，第一次查询的结果就是
  事务开始时的数据A，即使这时其他线程将其数据修改为B，查的结果依然是A。简而言之，就是开启事务并对所有表执行了一次SELECT操作，这样可保证备份时，
  在任意时间点执行select * from table得到的数据和
  执行START TRANSACTION WITH CONSISTENT SNAPSHOT时的数据一致。
 【注意】，WITH CONSISTENT SNAPSHOT只在RR隔离级别下有效。

第五步：
  SHOW MASTER STATUS
# 这个是由--master-data决定的，记录了开始备份时，binlog的状态信息，
  包括MASTER_LOG_FILE和MASTER_LOG_POS

这里需要特别区分一下master-data和dump-slave
master-data:
--master-data=2表示在dump过程中记录主库的binlog和pos点，并在dump文件中注释掉这一行；
--master-data=1表示在dump过程中记录主库的binlog和pos点，并在dump文件中不注释掉这一行，即恢复时会执行；
dump-slave
--dump-slave=2表示在dump过程中，在从库dump，mysqldump进程也要在从库执行，
  记录当时主库的binlog和pos点，并在dump文件中注释掉这一行；
--dump-slave=1表示在dump过程中，在从库dump，mysqldump进程也要在从库执行，
  记录当时主库的binlog和pos点，并在dump文件中不注释掉这一行；
 
第六步：
  UNLOCK TABLES
# 释放锁。
```

## mysqldump对InnoDB和MyISAM两种存储引擎进行备份的差异。
对于支持事务的引擎如InnoDB，参数上是在备份的时候加上 –single-transaction 保证数据一致性
```
–single-transaction 实际上通过做了下面两个操作 ：

① 在开始的时候把该 session 的事务隔离级别设置成 repeatable read ；

② 然后启动一个事务（执行 begin ），备份结束的时候结束该事务（执行 commit ）

有了这两个操作，在备份过程中，该 session 读到的数据都是启动备份时的数据（同一个点）。可以理解为对于 InnoDB 引擎来说加了该参数，备份开始时就已经把要备份的数据定下来了，
备份过程中的提交的事务时是看不到的，也不会备份进去。
```
对于不支持事务的引擎如MyISAM，只能通过锁表来保证数据一致性，这里分两种情况：
```
1）导出全库：加 –lock-all-tables 参数，这会在备份开始的时候启动一个全局读锁 
  （执行 flush tables with read lock），其他 session 可以读取但不能更新数据，
   备份过程中数据没有变化，所以最终得到的数据肯定是完全一致的；

2）导出单个库：加 –lock-tables 参数，这会在备份开始的时候锁该库的所有表，
   其他 session 可以读但不能更新该库的所有表，该库的数据一致；
```

