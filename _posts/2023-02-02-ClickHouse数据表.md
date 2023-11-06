---
layout: post
categories: [ClickHouse]
description: none
keywords: ClickHouse
---
# ClickHouse数据表

## 如何定义数据表
在知晓了ClickHouse的主要数据类型之后，接下来我们开始介绍DDL操作及定义数据的方法。DDL查询提供了数据表的创建、修改和删除操作，是最常用的功能之一。

### 数据库
数据库起到了命名空间的作用，可以有效规避命名冲突的问题，也为后续的数据隔离提供了支撑。任何一张数据表，都必须归属在某个数据库之下。

创建数据库的完整语法如下所示：
```
CREATE DATABASE IF NOT EXISTS db_name [ENGINE = engine]
```
其中，IF NOT EXISTS表示如果已经存在一个同名的数据库，则会忽略后续的创建过程；[ENGINE=engine]表示数据库所使用的引擎类型（是的，你没看错，数据库也支持设置引擎）。

数据库目前一共支持5种引擎，如下所示。
- Ordinary：默认引擎，在绝大多数情况下我们都会使用默认引擎，使用时无须刻意声明。在此数据库下可以使用任意类型的表引擎。
- Dictionary：字典引擎，此类数据库会自动为所有数据字典创建它们的数据表。
- Memory：内存引擎，用于存放临时数据。此类数据库下的数据表只会停留在内存中，不会涉及任何磁盘操作，当服务重启后数据会被清除。
- Lazy：日志引擎，此类数据库下只能使用Log系列的表引擎。
- MySQL：MySQL引擎，此类数据库下会自动拉取远端MySQL中的数据，并为它们创建MySQL表引擎的数据表。

在绝大多数情况下都只需使用默认的数据库引擎。例如执行下面的语句，即能够创建属于我们的第一个数据库：
```
CREATE DATABASE DB_TEST
```
默认数据库的实质是物理磁盘上的一个文件目录，所以在语句执行之后，ClickHouse便会在安装路径下创建DB_TEST数据库的文件目录：
```
# pwd
/chbase/data
# ls
DB_TEST  default  system
```
与此同时，在metadata路径下也会一同创建用于恢复数据库的DB_TEST.sql文件：
```
# pwd
/chbase/data/metadata
# ls
DB_TEST  DB_TEST.sql  default  system
```
使用SHOW DATABASES查询，即能够返回ClickHouse当前的数据库列表：
```
SHOW DATABASES
┌─name───┐
│  DB_TEST  │
│  default  │
│  system   │
└───────┘
```
使用USE查询可以实现在多个数据库之间进行切换，而通过SHOW TABLES查询可以查看当前数据库的数据表列表。删除一个数据库，则需要用到下面的DROP查询。
```
DROP DATABASE [IF EXISTS] db_name
```

### 数据表
ClickHouse数据表的定义语法，是在标准SQL的基础之上建立的，所以熟悉数据库的读者们在看到接下来的语法时，应该会感到很熟悉。ClickHouse目前提供了三种最基本的建表方法，其中，第一种是常规定义方法，它的完整语法如下所示：
```
CREATE TABLE [IF NOT EXISTS] [db_name.]table_name (
    name1 [type] [DEFAULT|MATERIALIZED|ALIAS expr],
    name2 [type] [DEFAULT|MATERIALIZED|ALIAS expr],
    省略…
) ENGINE = engine

```
使用[db_name.]参数可以为数据表指定数据库，如果不指定此参数，则默认会使用default数据库。例如执行下面的语句：
```
CREATE TABLE hits_v1 ( 
    Title String,
    URL String ,
    EventTime DateTime
) ENGINE = Memory;
```
上述语句将会在default默认的数据库下创建一张内存表。注意末尾的ENGINE参数，它被用于指定数据表的引擎。表引擎决定了数据表的特性，也决定了数据将会被如何存储及加载。例如示例中使用的Memory表引擎，是ClickHouse最简单的表引擎，数据只会被保存在内存中，在服务重启时数据会丢失。

第二种定义方法是复制其他表的结构，具体语法如下所示：
```
CREATE TABLE [IF NOT EXISTS] [db_name1.]table_name AS [db_name2.] table_name2 [ENGINE = engine]
```
这种方式支持在不同的数据库之间复制表结构，例如下面的语句：
```
--创建新的数据库
CREATE DATABASE IF NOT EXISTS new_db 
--将default.hits_v1的结构复制到new_db.hits_v1
CREATE TABLE IF NOT EXISTS new_db.hits_v1 AS default.hits_v1 ENGINE = TinyLog
```
上述语句将会把default.hits_v1的表结构原样复制到new_db.hits_v1，并且ENGINE表引擎可以与原表不同。

第三种定义方法是通过SELECT子句的形式创建，它的完整语法如下：
```
CREATE TABLE [IF NOT EXISTS] [db_name.]table_name ENGINE = engine AS SELECT …
```
在这种方式下，不仅会根据SELECT子句建立相应的表结构，同时还会将SELECT子句查询的数据顺带写入，例如执行下面的语句：
```
CREATE TABLE IF NOT EXISTS hits_v1_1 ENGINE = Memory AS SELECT * FROM hits_v1
```
上述语句会将SELECT * FROM hits_v1的查询结果一并写入数据表。

ClickHouse和大多数数据库一样，使用DESC查询可以返回数据表的定义结构。如果想删除一张数据表，则可以使用下面的DROP语句：
```
DROP TABLE [IF EXISTS] [db_name.]table_name
```

### 默认值表达式
表字段支持三种默认值表达式的定义方法，分别是DEFAULT、MATERIALIZED和ALIAS。无论使用哪种形式，表字段一旦被定义了默认值，它便不再强制要求定义数据类型，因为ClickHouse会根据默认值进行类型推断。如果同时对表字段定义了数据类型和默认值表达式，则以明确定义的数据类型为主，例如下面的例子：
```
CREATE TABLE dfv_v1 ( 
    id String,
    c1 DEFAULT 1000,
    c2 String DEFAULT c1
) ENGINE = TinyLog
```
c1字段没有定义数据类型，默认值为整型1000；c2字段定义了数据类型和默认值，且默认值等于c1，现在写入测试数据：
```
INSERT INTO dfv_v1(id) VALUES ('A000')
```
在写入之后执行以下查询：
```
SELECT c1, c2, toTypeName(c1), toTypeName(c2) from dfv_v1
┌──c1─┬─c2──┬─toTypeName(c1)─┬─toTypeName(c2)─┐
│ 1000 │ 1000 │ UInt16         │ String          │
└────┴────┴───────────┴──────────┘
```
由查询结果可以验证，默认值的优先级符合我们的预期，其中c1字段根据默认值被推断为UInt16；而c2字段由于同时定义了数据类型和默认值，所以它最终的数据类型来自明确定义的String。

默认值表达式的三种定义方法之间也存在着不同之处，可以从如下三个方面进行比较。
- 数据写入
在数据写入时，只有DEFAULT类型的字段可以出现在INSERT语句中。而MATERIALIZED和ALIAS都不能被显式赋值，它们只能依靠计算取值。例如试图为MATERIALIZED类型的字段写入数据，将会得到如下的错误。
```
DB::Exception: Cannot insert column URL, because it is MATERIALIZED column..
```
- 数据查询
在数据查询时，只有DEFAULT类型的字段可以通过SELECT *返回。而MATERIALIZED和ALIAS类型的字段不会出现在SELECT *查询的返回结果集中。
- 数据存储
在数据存储时，只有DEFAULT和MATERIALIZED类型的字段才支持持久化。如果使用的表引擎支持物理存储（例如TinyLog表引擎），那么这些列字段将会拥有物理存储。而ALIAS类型的字段不支持持久化，它的取值总是需要依靠计算产生，数据不会落到磁盘。

可以使用ALTER语句修改默认值，例如：
```
ALTER TABLE [db_name.]table MODIFY COLUMN col_name DEFAULT value
```
修改动作并不会影响数据表内先前已经存在的数据。但是默认值的修改有诸多限制，例如在合并树表引擎中，它的主键字段是无法被修改的；而某些表引擎则完全不支持修改（例如TinyLog）。

### 临时表
ClickHouse也有临时表的概念，创建临时表的方法是在普通表的基础之上添加TEMPORARY关键字，它的完整语法如下所示：
```
CREATE TEMPORARY TABLE [IF NOT EXISTS] table_name (
    name1 [type] [DEFAULT|MATERIALIZED|ALIAS expr],
    name2 [type] [DEFAULT|MATERIALIZED|ALIAS expr],
)
```
相比普通表而言，临时表有如下两点特殊之处：
- 它的生命周期是会话绑定的，所以它只支持Memory表引擎，如果会话结束，数据表就会被销毁；
- 临时表不属于任何数据库，所以在它的建表语句中，既没有数据库参数也没有表引擎参数。

针对第二个特殊项，读者心中难免会产生一个疑问：既然临时表不属于任何数据库，如果临时表和普通表名称相同，会出现什么状况呢？接下来不妨做个测试。首先在DEFAULT数据库创建测试表并写入数据：
```
CREATE TABLE tmp_v1 ( 
    title String
) ENGINE = Memory;
INSERT INTO tmp_v1 VALUES ('click')
```
接着创建一张名称相同的临时表并写入数据：
```
CREATE TEMPORARY TABLE tmp_v1 (createtime Datetime)
INSERT INTO tmp_v1 VALUES (now())
```
现在查询tmp_v1看看会发生什么：
```
SELECT * FROM tmp_v1
┌──────createtime─┐
│ 2019-08-30 10:20:29 │
└─────────────┘
```
通过返回结果可以得出结论：临时表的优先级是大于普通表的。当两张数据表名称相同的时候，会优先读取临时表的数据。

在ClickHouse的日常使用中，通常不会刻意使用临时表。它更多被运用在ClickHouse的内部，是数据在集群间传播的载体。

## 分区表
数据分区（partition）和数据分片（shard）是完全不同的两个概念。数据分区是针对本地数据而言的，是数据的一种纵向切分。而数据分片是数据的一种横向切分。数据分区对于一款OLAP数据库而言意义非凡：借助数据分区，在后续的查询过程中能够跳过不必要的数据目录，从而提升查询的性能。合理地利用分区特性，还可以变相实现数据的更新操作，因为数据分区支持删除、替换和重置操作。假设数据表按照月份分区，那么数据就可以按月份的粒度被替换更新。

分区虽好，但不是所有的表引擎都可以使用这项特性，目前只有合并树（MergeTree）家族系列的表引擎才支持数据分区。接下来通过一个简单的例子演示分区表的使用方法。首先由PARTITION BY指定分区键，例如下面的数据表partition_v1使用了日期字段作为分区键，并将其格式化为年月的形式：
```
CREATE TABLE partition_v1 ( 
    ID String,
    URL String,
    EventTime Date
) ENGINE =  MergeTree()
PARTITION BY toYYYYMM(EventTime) 
ORDER BY ID
```
接着写入不同月份的测试数据：
```
INSERT INTO partition_v1 VALUES 
('A000','www.nauu.com', '2019-05-01'),
('A001','www.brunce.com', '2019-06-02')
```
最后通过system.parts系统表，查询数据表的分区状态：
```
SELECT table,partition,path from system.parts WHERE table = 'partition_v1' 
┌─table─────┬─partition─┬─path─────────────────────────┐
│ partition_v1 │ 201905     │ /chbase/data/default/partition_v1/201905_1_1_0/│
│ partition_v1 │ 201906     │ /chbase/data/default/partition_v1/201906_2_2_0/│
└─────────┴────────┴─────────────────────────────┘
```
可以看到，partition_v1按年月划分后，目前拥有两个数据分区，且每个分区都对应一个独立的文件目录，用于保存各自部分的数据。

合理设计分区键非常重要，通常会按照数据表的查询场景进行针对性设计。例如在刚才的示例中数据表按年月分区，如果后续的查询按照分区键过滤，例如：
```
SELECT * FROM  partition_v1 WHERE EventTime ='2019-05-01'
```
那么在后续的查询过程中，可以利用分区索引跳过6月份的分区目录，只加载5月份的数据，从而带来查询的性能提升。

当然，使用不合理的分区键也会适得其反，分区键不应该使用粒度过细的数据字段。例如，按照小时分区，将会带来分区数量的急剧增长，从而导致性能下降。

## 视图
ClickHouse拥有普通和物化两种视图，其中物化视图拥有独立的存储，而普通视图只是一层简单的查询代理。创建普通视图的完整语法如下所示：
```
CREATE VIEW [IF NOT EXISTS] [db_name.]view_name AS SELECT ...
```
普通视图不会存储任何数据，它只是一层单纯的SELECT查询映射，起着简化查询、明晰语义的作用，对查询性能不会有任何增强。假设有一张普通视图view_tb_v1，它是基于数据表tb_v1创建的，那么下面的两条SELECT查询是完全等价的：
```
--普通表
SELECT * FROM tb_v1
-- tb_v1的视图
SELECT * FROM view_tb_v1
```
物化视图支持表引擎，数据保存形式由它的表引擎决定，创建物化视图的完整语法如下所示：
```
CREATE [MATERIALIZED] VIEW [IF NOT EXISTS] [db.]table_name [TO[db.]name] [ENGINE = engine] [POPULATE] AS SELECT ...
```
物化视图创建好之后，如果源表被写入新数据，那么物化视图也会同步更新。POPULATE修饰符决定了物化视图的初始化策略：如果使用了POPULATE修饰符，那么在创建视图的过程中，会连带将源表中已存在的数据一并导入，如同执行了SELECT INTO一般；反之，如果不使用POPULATE修饰符，那么物化视图在创建之后是没有数据的，它只会同步在此之后被写入源表的数据。物化视图目前并不支持同步删除，如果在源表中删除了数据，物化视图的数据仍会保留。

物化视图本质是一张特殊的数据表，例如使用SHOW TABLE查看数据表的列表：
```
SHOW TABLES
┌─name────────┐
│ .inner.view_test2 │
│ .inner.view_test3 │
└───────────┘
```
由上可以发现，物化视图也在其中，它们是使用了.inner.特殊前缀的数据表，所以删除视图的方法是直接使用DROP TABLE查询，例如：
```
DROP TABLE view_name
```