---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL日志Binlog

## 二进制日志(Binary log)
binlog可以说是MySQL中比较重要的日志了，在日常开发及运维过程中，经常会遇到。

binlog即binary log，二进制日志文件，也叫作变更日志（update log）。它记录了数据库所有执行的DDL和DML等数据库更新事件的语句，但是不包含没有修改任何数据的语句（如数据查询语句select、show等）。

它以事件形式记录并保存在二进制文件中。通过这些信息，我们可以再现数据更新操作的全过程。

如果想要记录所有语句（例如，为了识别有问题的查询)，需要使用通用查询日志。

Binary log主要应用场景：
- 一是用于数据恢复，如果MySQL数据库意外停止，可以通过二进制日志文件来查看用户执行了哪些操作，对数据库服务器文件做了哪些修改，然后根据二进制日志文件中的记录来恢复数据库服务器。
- 二是用于数据复制，由于日志的延续性和时效性，master把它的二进制日志传递给slaves来达到master-slave数据一致的目的。
可以说MySQL数据库的数据备份、主备、单主、多主、MGR都离不开Binary log,需要依靠Binary log来同步数据，保证数据一致性。

## 查看默认情况
查看记录二进制日志是否开启：在MySQL8中默认情况下，二进制文件是开启的。
```
mysql>  show variables like '%log_bin%';
+---------------------------------+-----------------------------+
| Variable_name                   | Value                       |
+---------------------------------+-----------------------------+
| log_bin                         | ON                          |  //开关
| log_bin_basename                | /var/lib/mysql/binlog       | // 存放路径
| log_bin_index                   | /var/lib/mysql/binlog.index |
| log_bin_trust_function_creators | ON                          |//  函数创建 
| log_bin_use_v1_row_events       | OFF                         |
| sql_log_bin                     | ON                          |//变更sql记录下来
+---------------------------------+-----------------------------+
6 rows in set (0.01 sec)
```
- log_bin_basename:是binlog日志的基本文件名，后面会追加标识来表示每一个文件
- log_bin_index:是binlog文件的素引文件，这个文件管理了所有的binlog文件的目录
- log_bin_trust_function_creators:限制存储过程，前面我们已经讲过了，这是因为二进制日志的一个重要功能是用于主从复制，而存储函数有可能导致主从的数据不一致。所以当开启二进制日志后，需要限制存储函数的创建、修改、调用
- log_bin_use_v1_row_events此只读系统变量已弃用。ON表示使用版本1二进制日志行，OFF表示使用版本2二进制日志行（MySQL5.6的默认值为2)。

## 日志参数设置
### 方式 1 ：永久性方式
修改MySQL的my.cnf或my.ini文件可以设置二进制日志的相关参数：
```
[mysqld]
#启用二进制日志
log-bin=mysql-bin
binlog_expire_logs_seconds= 600
max_binlog_size=100M
```
提示:
```
log-bin=mysql-bin
```
打开日志(主机需要打开)，这个mysql-bin也可以自定义，这里也可以加上路径，如:/home/www/mysql_bin_log/mysql-bin
```
binlog_expire_logs_seconds
```
此参数控制二进制日志文件保留的时长单位是秒，默认2592000 30天 --14400 4小时;86400 1天; 259200 3天;
```
max_binlog_size
```
控制单个二进制日志大小，当前日志文件大小超过此变量时，执行切换动作。此参数的最大和默认值是1GB，该设置并不能严格控制Binlog的大小，尤其是Binlog比较靠近最大值而又遇到一个比较大事务时，为了保证事务的完整性，可能不做切换日志的动作只能将该事务的所有SQL都记录进当前日志，直到事务结束。一般情况下可采取默认值。

设置带文件夹的bin-log日志存放目录

如果想改变日志文件的目录和名称，可以对my.cnf或my.ini中的log_bin参数修改如下：
```
[mysqld]
log-bin="/var/lib/mysql/binlog/yang-bin"
```
注意：新建的文件夹需要使用mysql用户，使用下面的命令即可。
```
chown -R -v mysql:mysql binlog
```
提示 数据库文件最好不要与日志文件放在同一个磁盘上!这样，当数据库文件所在的磁盘发生故障时，可以使用日志文件恢复数据。

### 方式 2 ：临时性方式
如果不希望通过修改配置文件并重启的方式设置二进制日志的话，还可以使用如下指令，需要注意的是在mysql 8 中只有会话级别的设置，没有了global级别的设置。
```
# global 级别
mysql> set global sql_log_bin= 0 ;
ERROR 1228 (HY000): Variable 'sql_log_bin' is a SESSION variable and can`t be used
with SET GLOBAL

# session级别
mysql> SET sql_log_bin = 0 ;
Query OK, 0 rows affected (0.01 秒)
```

## 查看日志
当MySQL创建二进制日志文件时，先创建一个以“filename”为名称、以“.index”为后缀的文件，再创建一个以“filename”为名称、以“.000001”为后缀的文件。

MySQL服务重新启动一次，以“.000001”为后缀的文件就会增加一个，并且后缀名按 1 递增。即日志文件的数与MySQL服务启动的次数相同；如果日志长度超过了max_binlog_size的上限（默认是1GB），就会创建一个新的日志文件。

查看当前的二进制日志文件列表及大小。指令如下：
```
mysql> SHOW BINARY LOGS;
+--------------------+-----------+-----------+
| Log_name           | File_size | Encrypted |
+--------------------+-----------+-----------+
| atguigu-bin.000001 | 156       | No        |
+--------------------+-----------+-----------+
1 rows in set (0.00 sec)
```
所有对数据库的修改都会记录在binglog中。但binlog是二进制文件，无法直接查看，借助mysqlbinlog命令工具了。指令如下:在查看执行，先执行一条sQL语句，如下
```
update student set name='张三_back' where id=1;
[root@localhost ~]$ cd /var/lib/mysql
[root@localhost ~]$ mysqlbinlog  "/var/lib/mysql/lqhdb-binlog.000001"
```
执行结果可以看到，这是一个简单的日志文件，日志中记录了用户的一些操作，这里并没有出现具体的SQL语句，这是因为binlog关键字后面的内容是经过编码后的二进制日志。

这里一个update语句包含如下事件
- Query事件负责开始一个事务(BEGIN)
- Table_map事件负责映射需要的表
- Update_rows事件负责写入数据
- Xid事件负责结束事务
下面命令将行事件以伪SQL的形式表现出来
```
mysqlbinlog -v "/var/lib/mysql/binlog/test.000002"
```
前面的命令同时显示binlog格式的语句，使用如下命令不显示它
```
mysqlbinlog -v --base64-output=DECODE-ROWS "/var/lib/mysql/binlog/test.000002"
```
关于mysqlbinlog工具的使用技巧还有很多，例如只解析对某个库的操作或者某个时间段内的操作等。简单分享几个常用的语句，更多操作可以参考官方文档。
```
# 可查看参数帮助
mysqlbinlog --no-defaults --help

# 查看最后 100 行
mysqlbinlog --no-defaults --base64-output=decode-rows -vv atguigu-bin.000002 |tail - 100

# 根据position查找
mysqlbinlog --no-defaults --base64-output=decode-rows -vv atguigu-bin.000002 |grep -A 
20 '4939002'
```
上面这种办法读取出binlog日志的全文内容比较多，不容易分辨查看到pos点信息，下面介绍一种更为方便的查询命令：
```
mysql> show binlog events [IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count];
```
- IN 'log_name'：指定要查询的binlog文件名（不指定就是第一个binlog文件）
- FROM pos：指定从哪个pos起始点开始查起（不指定就是从整个文件首个pos点开始算）
- LIMIT [offset]：偏移量(不指定就是 0 )
- row_count :查询总条数（不指定就是所有行）

上面这条语句可以将指定的binlog日志文件，分成有效事件行的方式返回，并可使用limit指定pos点的起始偏移，查询条数。其它举例:
```
#a、查询第一个最早的binlog日志:
show binlog events\G ;

#b、指定查询mysql-bin.088802这个文件
show binlog events in 'atguigu-bin. 008002'\G;

#c、指定查询mysql-bin. 080802这个文件，从pos点:391开始查起:
show binlog events in 'atguigu-bin.008802' from 391\G;

#d、指定查询mysql-bin.000802这个文件，从pos点:391开始查起，查询5条（即5条语句)
show binlog events in 'atguigu-bin.000882' from 391 limit 5\G

#e、指定查询 mysql-bin.880002这个文件，从pos点:391开始查起，偏移2行〈即中间跳过2个）查询5条（即5条语句)。
show binlog events in 'atguigu-bin.088882' from 391 limit 2,5\G;
```
binlog格式查看
```
mysql> show variables like 'binlog_format';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   | //行格式
+---------------+-------+
1 rows in set (0.00 sec)
```
除此之外，binlog还有 2 种格式，分别是Statemen和Mixed
- Statement 每一条会修改数据的sql都会记录在binlog中。 优点：不需要记录每一行的变化，减少了binlog日志量，节约了IO，提高性能。
- Row 5.1.5版本的MySQL才开始支持row level 的复制，它不记录sql语句上下文相关信息，仅保存哪条记录被修改。 优点：row level 的日志内容会非常清楚的记录下每一行数据修改的细节。而且不会出现某些特定情况下的存储过程，或function，以及trigger的调用和触发无法被正确复制的问题。
- Mixed 从5.1.8版本开始，MySQL提供了Mixed格式，实际上就是Statement与Row的结合。

## 使用日志恢复数据
mysqlbinlog恢复数据的语法如下：
```
mysqlbinlog [option] filename|mysql –uuser -ppass;
```
这个命令可以这样理解：使用mysqlbinlog命令来读取filename中的内容，然后使用mysql命令将这些内容恢复到数据库中。

- filename：是日志文件名。
- option：可选项，比较重要的两对option参数是–start-date、–stop-date 和 --start-position、–stop-position。
- –start-date 和 - -stop-date：可以指定恢复数据库的起始时间点和结束时间点。
- –start-position和–stop-position：可以指定恢复数据的开始位置和结束位置。
注意：使用mysqlbinlog命令进行恢复操作时，必须是编号小的先恢复，例如atguigu-bin.000001必须在atguigu-bin.000002之前恢复。
```
flush logs; #可以生成新的binLog 文件，不然这个文件边恢复边变大是不行的。

show binary logs; # 显示有哪些binLog 文件
```
恢复数据
```
mysqlbinlog [option] filename|mysql –uuser -ppass;

mysqlbinlog --no-defaults  --start-position=236  --stop-position=1071 --database=my_db1 /var/lib/mysql/lqhdb-bin.000002 | /usr/bin/mysql -root -p123456 -v my_db1
```

## 删除二进制日志
MySQL的二进制文件可以配置自动删除，同时MySQL也提供了安全的手动删除二进制文件的方法。PURGE MASTER LOGS只删除指定部分的二进制日志文件，RESET MASTER删除所有的二进制日志文件。具体如下：

PURGE MASTER LOGS：删除指定日志文件

PURGE MASTER LOGS语法如下：
```
PURGE {MASTER | BINARY} LOGS TO ‘指定日志文件名’

PURGE {MASTER | BINARY} LOGS BEFORE ‘指定日期’
```
举例 :使用PURGE MASTER LOGS语句删除创建时间比binlog.000005早的所有日志

(1)多次重新启动MysSQL服务，便于生成多个日志文件。然后用SHOW语句显示二进制日志文件列表
```
SHOW BINARY LOGS;
```

(2）执行PURGE MASTER LOGS语句删除创建时间比binlog.000005早的所有日志
```
PURGE MASTER LOGS T0 "binlog. 000005";
```
(3）显示二进制日志文件列表
```
SHGW BINARY LOGS;
```
举例:使用PURGE MASTER LOGS语句删除2023年3月17日前创建的所有日志文件。具体步骤如下:

(1) 显示二进制日志文件列表
```
SHOW BINARY LOGS;
```
(2）执行mysqlbinlog命令查看二进制日志文件binlog.000005的内容
```
mysqlbinlog --no-defaults "/var/lib/mysql/binlog/atguigu-bin.000005"
```
(3）使用PURGE MASTER LOGS语句删除2023年3月17日前创建的所有日志文件
```
PURGE MASTER LOGS before "20220317";
```
(4）显示二进制日志文件列表
```
SHOW BINARY LOGS;
```
2022年01月05号之前的二进制日志文件都已经被删除，最后一个没有删除，是因为当前在用，还未记录最后的时间，所以未被删除。

2.RESET MASTER:删除所有二进制日志文件
```
reset master;
```
其它场景
二进制日志可以通过数据库的全量备份和二进制日志中保存的增量信息，完成数据库的无损失恢复。但是，如果遇到数据量大、数据库和数据表很多（比如分库分表的应用）的场景，用二进制日志进行数据恢复，是很有挑战性的，因为起止位置不容易管理。

在这种情况下，一个有效的解决办法是配置主从数据库服务器，甚至是一主多从的架构，把二进制日志文件的内容通过中继日志，同步到从数据库服务器中，这样就可以有效避免数据库故障导致的数据异常等问题。

## 深入理解二进制日志
写入机制
binlog的写入时机也非常简单，事务执行过程中，先把日志写到binlog cache，事务提交的时候，再把binlog cache写到binlog文件中。因为一个事务的binlog不能被拆开，无论这个事务多大，也要确保一次性写入，所以系统会给每个线程分配一个块内存作为binlog cache。

我们可以通过binlog_cache_size参数控制单个线程binlog cache大，如果存储内容超过了这个参数，就要暂存到磁盘(Swap)。binlog日志刷盘流程如下:

write和fsync的时机，可以由参数sync_binlog控制，默认是 0 。

为 0 的时候，表示每次提交事务都只write，由系统自行判断什么时候执行fsync。虽然性能得到提升，但是机器宕机，page cache里面的binglog 会丢失。

为了安全起见，可以设置为 1 ，表示每次提交事务都会执行fsync，就如同 redo log 刷盘流程 一样。最后还有一种折中方式，可以设置为N(N>1)，表示每次提交事务都write，但累积N个事务后才fsync。

在出现IO瓶颈的场景里，将sync_binlog设置成一个比较大的值，可以提升性能。同样的，如果机器宕机，会丢失最近N个事务的binlog日志。

## binlog与redolog对比
redo log 它是物理日志，记录内容是“在某个数据页上做了什么修改”，属于 InnoDB 存储引擎层产生的。

而 binlog 是逻辑日志，记录内容是语句的原始逻辑，类似于“给 ID=2 这一行的 c 字段加 1”，属于MySQL Server 层

虽然它们都属于持久化的保证，但是则重点不同。

redo log让InnoDB存储引擎拥有了崩溃恢复能力。

binlog保证了MySQL集群架构的数据一致性。












