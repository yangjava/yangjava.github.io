---
layout: post
categories: [Hadoop]
description: none
keywords: Hadoop
---
# Hadoop实战HDFS

## HDFS产生背景
随着数据量越来越大，在一个操作系统存不下所有的数据，那么就分配到更多的操作系统管理的磁盘中，但是不方便管理和维护，迫切需要一种系统来管理多台机器上的文件，这就是分布式文件管理系统。HDFS只是分布式文件管理系统中的一种。

## HDFS定义
HDFS（Hadoop Distributed File System），它是一个文件系统，用于存储文件，通过目录树来定位文件；其次，它是分布式的，由很多服务器联合起来实现其功能，集群中的服务器有各自的角色。

HDFS的使用场景：适合一次写入，多次读出的场景。 一个文件经过创建、写入和关闭之后就不需要改变。 HDFS 主要适合去做批量数据出来，相对于数据请求时的反应时间，HDFS 更倾向于保障吞吐量。

## HDFS发展史
Doug Cutting 在做 Lucene 的时候, 需要编写一个爬虫服务，这个爬虫写的并不顺利，遇到了一些问题, 诸如：如何存储大规模的数据，如何保证集群的可伸缩性，如何动态容错等。
2013 年，Google 发布了三篇论文，被称作为三驾马车，其中有一篇叫做 GFS。 GFS 是描述了 Google 内部的一个叫做 GFS 的分布式大规模文件系统，具有强大的可伸缩性和容错性。
Doug Cutting 后来根据 GFS 的论文，创造了一个新的文件系统，叫做 HDFS

## HDFS优缺点
优点
- 高容错性
数据自动保存多个副本。它通过增加副本的形式，提高容错性。
某一个副本丢失后，它可以自动恢复。
- 适合处理大数据
数据规模：能够处理数据规模达到GB、TB甚至PB级别的数据。
文件规模：能够处理百万规模以上的文件数量，数量相当之大。
可构建在廉价机器上，通过多副本机制，提高可靠性。

缺点
不适合处理低延时数据访问，比如毫秒级的存储数据，是做不到的。
无法高效的对大量小文件进行存储
存储大量小文件的话，它会占用 NameNode 大量的内存来存储文件目录和块信息。这样是不可取的，因为 NameNode 的内存总是有限的。
小文件存储的寻址时间会超过读取时间，它违反了 HDFS 的设计目标。
不支持并发写入、文件随机修改
一个文件只能有一个写，不允许多个线程同时写。
仅支持数据 append，不支持文件的随机修改。

## HDFS组成架构
- NameNode
Master，它是一个主管、管理者。
管理 HDFS 的名称空间；
配置副本策略；
管理数据块（Block）映射信息；
处理客户端读取请求；

- DataNode
Slave，NameNode赋值下达命令，DataNode执行实际的操作。
存储实际的数据块；
执行数据块的读/写操作；

- Client： 客户端。
文件切分。文件上传 HDFS 的时候，Client 将文件切分成一个一个的Block，然后进行上传；
与 NameNode 交互，获取文件的位置信息；
与 DataNode 交互，读取或者写入数据；
Client 提供一些命令来管理 HDFS，比如对 NameNode 格式化；
Client 可以通过一些命令来访问 HDFS，比如对 HDFS 增删改查操作；

- SecondaryNameNode
并非 NameNode 的热备，当 NameNode 挂掉的时候，它并不能马上替换 NameNode 并提供服务。
辅助 NameNode，分担其工作量，比如定期合并 Fsimage 和 Edits，并推送给 NameNode；
在紧急情况下，可辅助恢复 NameNode；

## HDFS的重要特性
- 主从架构
HDFS 采用 master/slave 架构。一般一个 HDFS 集群是有一个 Namenode 和一定数目的 Datanode 组成。Namenode是HDFS主节点，Datanode是HDFS从节点，两种角色各司其职，共同协调完成分布式的文件存储服务。

### 分块机制
HDFS 中的文件在物理上是分块存储（block）的，块的大小可以通过配置参数来规定，参数位于 hdfs-default.xml 中：dfs.blocksize。默认大小在 Hadoop2.x/3.x 是128M（134217728），1.x 版本中是 64M。

### HDFS文件块大小设置
HDFS 的块设置太小，会增加寻址时间，程序一直在找块的开始位置；
如果块设置的太大，从磁盘传输数据的时间会明显大于定位这个块开始位置所需的时间，导致程序在处理这块数据时，会非常慢。
总结：HDFS块的大小设置主要取决于磁盘传输速率。

### 副本机制
为了容错，文件的所有 block 都会有副本。每个文件的 block 大小（dfs.blocksize）和副本系数（dfs.replication）都是可配置的。应用程序可以指定某个文件的副本数目。副本系数可以在文件创建的时候指定，也可以在之后通过命令改变。

默认dfs.replication的值是3，也就是会额外再复制 2 份，连同本身总共 3 份副本。

### Namespace
HDFS 支持传统的层次型文件组织结构。用户可以创建目录，然后将文件保存在这些目录里。文件系统名字空间的层次结构和大多数现有的文件系统类似：用户可以创建、删除、移动或重命名文件。
Namenode 负责维护文件系统的 namespace 名称空间，任何对文件系统名称空间或属性的修改都将被 Namenode 记录下来。
HDFS 会给客户端提供一个统一的抽象目录树，客户端通过路径来访问文件，形如：hdfs://namenode:port/dir-a/dir-b/dir-c/file.data

### 元数据管理
在 HDFS 中，Namenode 管理的元数据具有两种类型：
- 文件自身属性信息
文件名称、权限，修改时间，文件大小，复制因子，数据块大小。
- 文件块位置映射信息
记录文件块和 DataNode 之间的映射信息，即哪个块位于哪个节点上。

### 数据块存储
文件的各个 block 的具体存储管理由 DataNode 节点承担。每一个 block 都可以在多个 DataNode 上存储。


## HDFS的Shell操作
基本语法
```
hadoop fs 具体命令 OR hdfs dfs 具体命令
```

上传

-moveFromLocal：从本地剪切粘贴到 HDFS
```
hadoop fs -moveFromLocal ./1.txt /test
```

copyFromLocal：从本地文件系统中拷贝文件到 HDFS 路径去
```
hadoop fs -copyFromLocal ./1.txt /test
```

-put：等同于 copyFromLocal，生产环境更习惯用 put
```
hadoop fs -put ./1.txt /test
```

-appendToFile：追加一个文件到已经存在的文件末尾
```
hadoop fs -appendToFile ./1.txt /test/1.txt
```

下载
-copyToLocal：从 HDFS 拷贝到本地
```
hadoop fs -copyToLocal /test/1.txt /output
```
-get：等同于 copyToLocal，生产环境更习惯用 get
```
hadoop fs -get /test/1.txt /output
```

HDFS直接操作
-ls：显示目录信息
```
hadoop fs -ls /test
```
-cat：显示文件内容
```
hadoop fs -cat /test/1.txt
```
-chgrp、-chmod、-chown：Linux 文件系统中的用法一样，修改文件所属权限
```
hadoop fs -chmod 666 /test/1.txt
hadoop fs -chown hadoop:hadoop /test/1.txt
```
-mkdir：创建路径
```
hadoop fs -mkdir /test
```
-cp：从 HDFS 的一个路径拷贝到 HDFS 的另一个路径
```
hadoop fs -cp /test1/1.txt /test2
```
-mv：在 HDFS 目录中移动文件
```
hadoop fs -mv /test1/1.txt /test2
```
-tail：显示一个文件的末尾 1kb 的数据
```
hadoop fs -tail /test/1.txt
```
-rm：删除文件或文件夹
```
hadoop fs -rm /test/1.txt
```
-rm -r：递归删除目录及目录里面内容
```
hadoop fs -rm -r /test
```
-du：统计文件夹的大小信息
```
[hadoop@hadoop1 hadoop-3.3.1]$ hadoop fs -du -s -h /input/1.txt
37  111  /input/1.txt
[hadoop@hadoop1 hadoop-3.3.1]$ hadoop fs -du -h /input
37  111  /input/1.txt
5   15   /input/2.txt
# 37表示文件大小；111表示37*3个副本；/input表示查看的目录
```
-setrep：设置 HDFS 中文件的副本数量
```
hadoop fs -setrep 10 /input/1.txt
# 这里设置的副本数只是记录在NameNode的元数据中，是否真的会有这么多副本，还得看DataNode的数量。因为目前只有3台设备，最多也就3个副本，只有节点数的增加到10台时，副本数才能达到10。
```

## 命令行接口
Hadoop自带一组命令行工具，而其中有关HDFS的命令是其工具集的一个子集。命令行工具虽然是最基础的文件操作方式，但却是最常用的。作为一名合格的Hadoop开发人员和运维人员，熟练掌握是非常有必要的。

执行hadoop dfs命令可以显示基本的使用信息。
```
[hadoop@master bin]$ hadoop dfs
Usage: java FsShell
　　　　　 [-ls <path>]
　　　　　 [-lsr <path>]
　　　　　 [-df [<path>]]
　　　　　 [-du <path>]
　　　　　 [-dus <path>]
　　　　　 [-count[-q] <path>]
　　　　　 [-mv <src> <dst>]
　　　　　 [-cp <src> <dst>]
　　　　　 [-rm [-skipTrash] <path>]
　　　　　 [-rmr [-skipTrash] <path>]
　　　　　 [-expunge]
　　　　　 [-put <localsrc> ... <dst>]
　　　　　 [-copyFromLocal <localsrc> ... <dst>]
　　　　　 [-moveFromLocal <localsrc> ... <dst>]
　　　　　 [-get [-ignoreCrc] [-crc] <src> <localdst>]
　　　　　 [-getmerge <src> <localdst> [addnl]]
　　　　　 [-cat <src>]
　　　　　 [-text <src>]
　　　　　 [-copyToLocal [-ignoreCrc] [-crc] <src> <localdst>]
　　　　　 [-moveToLocal [-crc] <src> <localdst>]
　　　　　 [-mkdir <path>]
　　　　　 [-setrep [-R] [-w] <rep> <path/file>]
　　　　　 [-touchz <path>]
　　　　　 [-test -[ezd] <path>]
　　　　　 [-stat [format] <path>]
　　　　　 [-tail [-f] <file>]
　　　　　 [-chmod [-R] <MODE[,MODE]... | OCTALMODE> PATH...]
　　　　　 [-chown [-R] [OWNER][:[GROUP]] PATH...]
　　　　　 [-chgrp [-R] GROUP PATH...]
　　　　　 [-help [cmd]]
```
- `hadoop dfs –ls <path>`  列出文件或目录内容  hadoop dfs –ls /
- `hadoop dfs –lsr <path>`  递归地列出目录内容  hadoop dfs –lsr /
- `hadoop dfs –df <path>`  查看目录的使用情况  hadoop dfs –df /
- `hadoop dfs –du <path>` 显示目录中所有文件及目录的大小  hadoop dfs –du /
- `hadoop dfs –dus <path>` 只显示<path>目录的总大小，与-du不同的是，-du会把<path>目录下所有的文件或目录大小都列举出来，而-dus只会将<path>目录的大小列出来  hadoop dfs –dus /
- `hadoop dfs –count [-q] <path>` 显示<path>下的目录数及文件数，输出格式为“目录数 文件数 大小 文件名”。如果加上-q还可以查看文件索引的情况  hadoop dfs –count /
- `hadoop dfs –mv <src> <dst>` 将HDFS上的文件移动到目的文件夹 hadoop dfs–mv/user/hadoop/ a.txt /user/test，将/user/ hadoop文件夹下的文件a.txt移动到/user/test文件夹下
- `Hadoop dfs –rm [-skipTrash] <path>` 将HDFS上路径为<path>的文件移动到回收站。如果加上-skipTrash，则直接删除  hadoop dfs –rm /test.txt
- `hadoop dfs –rmr [-skipTrash] <path>` 将HDFS上路径为<path>的目录以及目录下的文件移动到回收站。如果加上-skipTrash，则直接删除  hadoop dfs –rmr /test
- `hadoop dfs –expunge` 清空回收站 hadoop dfs –expunge
- `hadoop dfs –put <localsrc> … <dst>` 将<localsrc>本地文件上传到HDFS的<dst>目录下  hadoop dfs –put /home/hadoop/ test.txt / user/hadoop
- `hadoop dfs-copyFromLocal <localsrc> ... <dst>` 功能类似于put  hadoop dfs -copyFromLocal /home/ hadoop/test.txt /user/hadoop
- `hadoop dfs-moveFromLocal <localsrc> ... <dst>` 将<localsrc>本地文件移动到HDFS的<dst>目录下  hadoop dfs -moveFromLocal /home/ hadoop/test.txt /user/hadoop
- `hadoop dfs -get [-ignoreCrc] [-crc] <src> <localdst>` 将HDFS上<src>的文件下载到本地的<localdst>目录，可用-ignorecrc选项复制CRC校验失败的文件，使用-crc选项复制文件以及CRC信息  hadoop dfs -get /user/ hadoop/a.txt /home/hadoop
- `hadoop dfs -getmerge <src> <localdst> [addnl]` 将HDFS上<src>目录下的所有文件按文件名排序并合并成一个文件输出到本地的<localdst>目录，addnl是可选的，用于指定在每个文件结尾添加一个换行符  hadoop dfs-getmerge/user/ test /home/ hadoop/o
- `hadoop dfs -cat <src>` 浏览HDFS路径为<src>的文件的内容  hadoop dfs -cat /user/ hadoop/text.txt
- `hadoop dfs -text <src>` 将HDFS路径为<src>的文本文件输出  hadoop dfs -text /user/ test.txt
- `hadoop dfs -copyToLocal [-ignoreCrc] [-crc] <src> <localdst>` 功能类似于get  hadoop dfs -copyToLocal /user/ hadoop/a.txt/home/ hadoop
- `hadoop dfs -moveToLocal [-crc] <src> <localdst>` 将HDFS上路径为<src>的文件移动到本地<localdst>路径下  hadoop dfs -get /user/ hadoop/a.txt /home/hadoop
- `hadoop dfs -mkdir <path>` 在HDFS上创建路径为<path>的目录  hadoop dfs -mkdir /user/ test
- `hadoop dfs -setrep [-R] [-w] <rep> <path/file>` 设置文件的复制因子。该命令可以单独设置文件的复制因子，加上-R可以递归执行该操作 hadoop dfs -setrep 5 -R /user/test
- `hadoop dfs -touchz <path>` 创建一个路径为<path>的0字节的HDFS空文件 hadoop dfs -touchz /user/ hadoop/ test
- `hadoop dfs -test -[ezd] <path>` 检查HDFS上路径为<path>的文件，-e检查文件是否存在。如果存在则返回0，-z检查文件是否是0字节。如果是则返回0，-d如果路径是个目录，则返回1，否则返回0  hadoop dfs -test -e /user/ test.txt
- `hadoop dfs -stat [format] <path>` 显示HDFS上路径为<path>的文件或目录的统计信息。 hadoop fs -stat %b %n %o %r /user/test
- `hadoop dfs -tail [-f] <file>`  显示HDFS上路径为<file>的文件的最后1 KB的字节，-f选项会使显示的内容随着文件内容更新而更新  hadoop dfs -tail -f /user/ test.txt
- `hadoop dfs -chmod [-R] <MODE[,MODE]... ｜ OCTALMODE> PATH...`  改变HDFS上路径为PATH的文件的权限，-R选项表示递归执行该操作  hadoop dfs -chmod -R +r /user/ test，表示将/user/ test目录下的所有文件赋予读的权限
- `hadoop dfs -chown [-R] [OWNER][:[GROUP]] PATH... ` 改变HDFS上路径为PATH的文件的所属用户，-R选项表示递归执行该操作  hadoop dfs -chown -R hadoop: hadoop /user/test，表示将/user/test目录下所有文件的所属用户和所属组别改为hadoop
- `hadoop dfs -chgrp [-R] GROUP PATH... `改变HDFS上路径为PATH的文件的所属组别，-R选项表示递归执行该操作 hadoop dfs -chown -R hadoop /user/test，表示将/user/test目录下所有文件的所属组别改为hadoop
- `hadoop dfs -help` 显示所有dfs命令的帮助信息 hadoop dfs -help
