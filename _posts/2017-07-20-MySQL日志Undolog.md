---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL日志Undolog

## 什么是Undo Log？
Undo:意为撤销或取消，以撤销操作为目的，返回某个状态的操作。

Undo Log:数据库事务开始之前，会将要修改的记录放到Undo日志里，当事务回滚时或者数据库崩溃时，可以利用UndoLog撤销未提交事务对数据库产生的影响。

Undo Log是事务原子性的保证。在事务中更新数据的前置操作其实是要先写入一个Undo Log

## 如何理解Undo Log
事务需要保证原子性，也就是事务中的操作要么全部完成，要么什么也不做。但有时候事务执行到一半会出现一些情况，比如：
- 情况一：事务执行过程中可能遇到各种错误，比如服务器本身的错误，操作系统错误，甚至是突然断电导致的错误。
- 情况二：DBA可以在事务执行过程中手动输入ROLLBACK语句结束当前事务的执行。以上情况出现，我们需要把数据改回原先的样子，这个过程称之为回滚。

每当我们要对一条记录做改动时(这里的改动可以指INSERT、DELETE、UPDATE)，都需要把回滚时所需的东西记下来。比如:
- 你插入一条记录时，至少要把这条记录的主键值记下来，之后回滚的时候只需要把这个主键值对应的记录删掉就好了。(对于每个INSERT, InnoDB存储引擎会完成一个DELETE)
- 你删除了一条记录，至少要把这条记录中的内容都记下来，这样之后回滚时再把由这些内容组成的记录插入到表中就好了。(对于每个DELETE,InnoDB存储引擎会执行一个INSERT)
- 你修改了一条记录，至少要把修改这条记录前的旧值都记录下来，这样之后回滚时再把这条记录更新为旧值就好了。(对于每个UPDATE，InnoDB存储引擎会执行一个相反的UPDATE，将修改前的行放回去)

MySQL把这些为了回滚而记录的这些内容称之为撤销日志或者回滚日志(即Undo Log)。注意，由于查询操作(SELECT）并不会修改任何用户记录，所以在杳询操作行时，并不需要记录相应的Undo日志

此外，Undo Log会产生Redo Log，也就是Undo Log的产生会伴随着Redo Log的产生，这是因为Undo Log也需要持久性的保护。

## Undo Log的功能
- 提供数据回滚-原子性
当事务回滚时或者数据库崩溃时，可以利用Undo Log来进行数据回滚。
- 多版本并发控制(MVCC)-隔离性
即在InnoDB存储引擎中MVCC的实现是通过Undo Log来完成。当用户读取一行记录时，若该记录已经被其他事务占用，当前事务可以通过Undo Log读取之前的行版本信息，以此实现非锁定读取。

## Undo Log的存储结构

### 回滚段与Undo页
InnoDB对Undo Log的管理采用段的方式，也就是回滚段（rollback segment）。每个回滚段记录了1024 个Undo Log segment，而在每个Undo Log segment段中进行Undo页的申请。

在InnoDB1.1版本之前（不包括1.1版本），只有一个rollback segment，因此支持同时在线的事务限制为 1024。虽然对绝大多数的应用来说都已经够用。

从1.1版本开始InnoDB支持最大128个rollback segment，故其支持同时在线的事务限制提高到了128*1024。
```
mysql> show variables like 'innodb_undo_logs';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| innodb_undo_logs | 128   |
+------------------+-------+
```
虽然InnoDB1.1版本支持了128个rollback segment，但是这些rollback segment都存储于共享表空间ibdata中。从lnnoDB1.2版本开始，可通过参数对rollback segment做进一步的设置。

这些参数包括:
- innodb_undo_directory:设置rollback segment文件所在的路径。这意味着rollback segment可以存放在共享表空间以外的位置，即可以设置为独立表空间。该参数的默认值为“./”，表示当前InnoDB存储引擎的目录。
- innodb_undo_logs:设置rollback segment的个数，默认值为128。在InnoDB1.2版本中，该参数用来替换之前版本的参数innodb_rollback_segments。
- innodb_undo_tablespaces:设置构成rollback segment文件的数量，这样rollback segment可以较为平均地分布在多个文件中。设置该参数后，会在路径innodb_undo_directory看到undo为前缀的文件，该文件就代表rollback segment文件。

## 回滚段与事务
- 每个事务只会使用一个回滚段（rollback segment），一个回滚段在同一时刻可能会服务于多个事务。
- 当一个事务开始的时候，会制定一个回滚段，在事务进行的过程中，当数据被修改时，原始的数据会被复制到回滚段。
- 在回滚段中，事务会不断填充盘区，直到事务结束或所有的空间被用完。如果当前的盘区不够用，事务会在段中请求扩展下一个盘区，如果所有已分配的盘区都被用完，事务会覆盖最初的盘区或者在回滚段允许的情况下扩展新的盘区来使用。
- 回滚段存在于Undo表空间中，在数据库中可以存在多个Undo表空间，但同一时刻只能使用一个Undo表空间。
- 当事务提交时，InnoDB存储引擎会做以下两件事情：
1.将Undo Log放入列表中，以供之后的purge(清洗、清除)操作
2.判断Undo Log所在的页是否可以重用(低于3/4可以重用)，若可以分配给下个事务使用

## 回滚段中的数据分类
未提交的回滚数据(uncommitted undo information)：该数据所关联的事务并未提交，用于实现读一致性，所以该数据不能被其他事务的数据覆盖。

已经提交但未过期的回滚数据(committed undo information)：该数据关联的事务已经提交，但是仍受到undo retention参数的保持时间的影响。

事务已经提交并过期的数据(expired undo information)：事务已经提交，而且数据保存时间已经超过undo retention参数指定的时间，属于已经过期的数据。当回滚段满了之后，会优先覆盖"事务已经提交并过期的数据"。

## Undo页的重用
当我们开启一个事务需要写Undo log的时候，就得先去Undo Log segment中去找到一个空闲的位置，当有空位的时候，就去申请Undo页，在这个申请到的Undo页中进行Undo Log的写入。我们知道MySQL默认一页的大小是16k。

为每一个事务分配一个页，是非常浪费的(除非你的事务非常长)，假设你的应用的TPS(每秒处理的事务数目)为1000，那么1s就需要1000个页，大概需要16M的存储，1分钟大概需要1G的存储。如果照这样下去除非MySQL清理的非常勤快，否则随着时间的推移，磁盘空间会增长的非常快，而且很多空间都是浪费的。

于是Undo页就被设计的可以重用了，当事务提交时，并不会立刻删除Undo页。因为重用，所以这个Undo页可能混杂着其他事务的Undo Log。Undo Log在commit后，会被放到一个链表中，然后判断Undo页的使用空间是否小于3/4，如果小于3/4的话，则表示当前的Undo页可以被重用，那么它就不会被回收，其他事务的Undo Log可以记录在当前Undo页的后面。由于Undo Log是离散的，所以清理对应的磁盘空间时，效率不高。

## Undo Log日志的存储机制
可以看到，Undo Log日志里面不仅存放着数据更新前的记录，还记录着RowID、事务ID、回滚指针。其中事务ID每次递增，回滚指针第一次如果是INSERT语句的话，回滚指针为NULL，第二次UPDATE之后的Undo Log的回滚指针就会指向刚刚那一条Undo Log日志，以此类推，就会形成一条Undo Log的回滚链，方便找到该条记录的历史版本。

## Undo Log的工作原理
在更新数据之前，MySQL会提前生成Undo Log日志，当事务提交的时候，并不会立即删除Undo Log，因为后面可能需要进行回滚操作，要执行回滚（ROLLBACK）操作时，从缓存中读取数据。Undo Log日志的删除是通过通过后台purge线程进行回收处理的。

- 1、事务A执行UPDATE操作，此时事务还没提交，会将数据进行备份到对应的Undo Buffer，然后由Undo Buffer持久化到磁盘中的Undo Log文件中，此时Undo Log保存了未提交之前的操作日志，接着将操作的数据，也就是test表的数据持久保存到InnoDB的数据文件IBD。
- 2、此时事务B进行查询操作，直接从Undo Buffer缓存中进行读取，这时事务A还没提交事务，如果要回滚（ROLLBACK）事务，是不读磁盘的，先直接从Undo Buffer缓存读取。

## Undo Log的类型
在InnoDB存储引擎中，Undo Log分为：

- insert Undo Log
insert Undo Log是指在insert操作中产生的Undo Log。因为insert操作的记录，只对事务本身可见，对其他事务不可见(这是事务隔离性的要求)，故该Undo Log可以在事务提交后直接删除。不需要进行purge操作。

- update Undo Log
update Undo Log记录的是对delete和update操作产生的Undo Log。该Undo Log可能需要提供MVCC机制，因此不能在事务提交时就进行删除。提交时放入Undo Log链表，等待purge线程进行最后的删除。

## Undo Log的生命周期
简要生成过程
以下是Undo+Redo事务的简化过程:
假设有2个数值，分别为 A=1 和 B=2 ，然后将A修改为3，B修改为4
```
1. start transaction;
2．记录A=1到Undo Log;
3. update A = 3;
4．记录A=3 到Redo Log;
5．记录B=2到Undo Log;
6. update B = 4;
7．记录B=4到Redo Log;
8．将Redo Log刷新到磁盘;
9. commit
```
- 在1-8步骤的任意一步系统宕机，事务未提交，该事务就不会对磁盘上的数据做任何影响。
- 如果在8-9之间宕机。
- Redo Log 进行恢复
- Undo Log 发现有事务没完成进行回滚。
- 若在9之后系统宕机，内存映射中变更的数据还来不及刷回磁盘，那么系统恢复之后，可以根据Redo Log把数据刷回磁盘。

## Undo Log的配置参数
innodb_max_undo_log_size:Undo日志文件的最大值，默认1GB，初始化大小10M

innodb_undo_log_truncate:标识是否开启自动收缩Undo Log表空间的操作

innodb_undo_tablespaces:设置独立表空间的个数，默认为0，标识不开启独立表空间，Undo日志保存在ibdata1中

innodb_undo_directory:Undo日志存储的目录位置 innodb_undo_logs: 回滚的个数 默认128




















































