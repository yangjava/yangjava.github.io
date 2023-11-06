---
layout: post
categories: [ClickHouse]
description: none
keywords: ClickHouse
---
# ClickHouse

## OLAP常见架构分类
OLAP领域技术发展至今方兴未艾，分析型数据库百花齐放。OLAP名为联机分析，又可以称为多维分析，是由关系型数据库之父埃德加·科德（Edgar Frank Codd）于1993年提出的概念。顾名思义，它指的是通过多种不同的维度审视数据，进行深层次分析。维度可以看作观察数据的一种视角，例如人类能看到的世界是三维的，它包含长、宽、高三个维度。直接一点理解，维度就好比是一张数据表的字段，而多维分析则是基于这些字段进行聚合查询。

那么多维分析通常都包含哪些基本操作呢？为了更好地理解多维分析的概念，可以使用一个立方体的图像具象化操作，对于一张销售明细表，数据立方体可以进行如下操作。
- 下钻：从高层次向低层次明细数据穿透。例如从“省”下钻到“市”，从“湖北省”穿透到“武汉”和“宜昌”。
- 上卷：和下钻相反，从低层次向高层次汇聚。例如从“市”汇聚成“省”，将“武汉”“宜昌”汇聚成“湖北”。
- 切片：观察立方体的一层，将一个或多个维度设为单个固定值，然后观察剩余的维度，例如将商品维度固定为“足球”。
- 切块：与切片类似，只是将单个固定值变成多个值。例如将商品维度固定成“足球”“篮球”和“乒乓球”。
- 旋转：旋转立方体的一面，如果要将数据映射到一张二维表，那么就要进行旋转，这就等同于行列置换。

为了实现上述这些操作，将常见的OLAP架构大致分成三类。
- 第一类架构称为ROLAP（Relational OLAP，关系型OLAP）。顾名思义，它直接使用关系模型构建，数据模型常使用星型模型或者雪花模型。这是最先能够想到，也是最为直接的实现方法。因为OLAP概念在最初提出的时候，就是建立在关系型数据库之上的。多维分析的操作，可以直接转换成SQL查询。
- 第二类架构称为MOLAP（Multidimensional OLAP，多维型OLAP）。它的出现是为了缓解ROLAP性能问题。MOLAP使用多维数组的形式保存数据，其核心思想是借助预先聚合结果，使用空间换取时间的形式最终提升查询性能。也就是说，用更多的存储空间换得查询时间的减少。
- 第三类架构称为HOLAP（Hybrid OLAP，混合架构的OLAP）。这种思路可以理解成ROLAP和MOLAP两者的集成。

## ClickHouse名称
ClickHouse由雏形发展至今一共经历了四个阶段。它的初始设计目标是服务自己公司的一款名叫Yandex.Metrica的产品。Metrica是一款Web流量分析工具，基于前方探针采集行为数据，然后进行一系列的数据分析，类似数据仓库的OLAP分析。而在采集数据的过程中，一次页面click（点击），会产生一个event（事件）。至此，整个系统的逻辑就十分清晰了，那就是基于页面的点击事件流，面向数据仓库进行OLAP分析。所以ClickHouse的全称是Click Stream，Data WareHouse，简称ClickHouse

## ClickHouse适用的场景
因为ClickHouse在诞生之初是为了服务Yandex自家的Web流量分析产品Yandex.Metrica，所以在存储数据超过20万亿行的情况下，ClickHouse做到了90%的查询都能够在1秒内返回的惊人之举。随后，ClickHouse进一步被应用到Yandex内部大大小小数十个其他的分析场景中。可以说ClickHouse具备了人们对一款高性能OLAP数据库的美好向往，所以它基本能够胜任各种数据分析类的场景，并且随着数据体量的增大，它的优势也会变得越为明显。

ClickHouse非常适用于商业智能领域（也就是我们所说的BI领域），除此之外，它也能够被广泛应用于广告流量、Web、App流量、电信、金融、电子商务、信息安全、网络游戏、物联网等众多其他领域。

## ClickHouse不适用的场景
ClickHouse作为一款高性能OLAP数据库，虽然足够优秀，但也不是万能的。我们不应该把它用于任何OLTP事务性操作的场景，因为它有以下几点不足。

- 不支持事务。
- 不擅长根据主键按行粒度进行查询（虽然支持），故不应该把ClickHouse当作Key-Value数据库使用。
- 不擅长按行删除数据（虽然支持）。

这些弱点并不能视为ClickHouse的缺点，事实上其他同类高性能的OLAP数据库同样也不擅长上述的这些方面。因为对于一款OLAP数据库而言，上述这些能力并不是重点，只能说这是为了极致查询性能所做的权衡。

## 架构概述
ClickHouse是一款MPP架构的列式存储数据库，但MPP和列式存储并不是什么“稀罕”的设计。拥有类似架构的其他数据库产品也有很多，但是为什么偏偏只有ClickHouse的性能如此出众呢？

### 完备的DBMS功能
ClickHouse拥有完备的管理功能，所以它称得上是一个DBMS（Database Management System，数据库管理系统），而不仅是一个数据库。作为一个DBMS，它具备了一些基本功能，如下所示。
- DDL（数据定义语言）：可以动态地创建、修改或删除数据库、表和视图，而无须重启服务。
- DML（数据操作语言）：可以动态查询、插入、修改或删除数据。
- 权限控制：可以按照用户粒度设置数据库或者表的操作权限，保障数据的安全性。
- 数据备份与恢复：提供了数据备份导出与导入恢复机制，满足生产环境的要求。
- 分布式管理：提供集群模式，能够自动管理多个数据库节点。

### 列式存储与数据压缩
列式存储和数据压缩，对于一款高性能数据库来说是必不可少的特性。一个非常流行的观点认为，如果你想让查询变得更快，最简单且有效的方法是减少数据扫描范围和数据传输时的大小，而列式存储和数据压缩就可以帮助我们实现上述两点。列式存储和数据压缩通常是伴生的，因为一般来说列式存储是数据压缩的前提。

ClickHouse就是一款使用列式存储的数据库，数据按列进行组织，属于同一列的数据会被保存在一起，列与列之间也会由不同的文件分别保存（这里主要指MergeTree表引擎，表引擎会在后续章节详细介绍）。数据默认使用LZ4算法压缩，在Yandex.Metrica的生产环境中，数据总体的压缩比可以达到8:1（未压缩前17PB，压缩后2PB）。列式存储除了降低IO和存储的压力之外，还为向量化执行做好了铺垫。

### 向量化执行引擎
向量化执行，可以简单地看作一项消除程序中循环的优化。这里用一个形象的例子比喻。小胡经营了一家果汁店，虽然店里的鲜榨苹果汁深受大家喜爱，但客户总是抱怨制作果汁的速度太慢。小胡的店里只有一台榨汁机，每次他都会从篮子里拿出一个苹果，放到榨汁机内等待出汁。如果有8个客户，每个客户都点了一杯苹果汁，那么小胡需要重复循环8次上述的榨汁流程，才能榨出8杯苹果汁。如果制作一杯果汁需要5分钟，那么全部制作完毕则需要40分钟。为了提升果汁的制作速度，小胡想出了一个办法。他将榨汁机的数量从1台增加到了8台，这么一来，他就可以从篮子里一次性拿出8个苹果，分别放入8台榨汁机同时榨汁。此时，小胡只需要5分钟就能够制作出8杯苹果汁。为了制作n杯果汁，非向量化执行的方式是用1台榨汁机重复循环制作n次，而向量化执行的方式是用n台榨汁机只执行1次。

为了实现向量化执行，需要利用CPU的SIMD指令。SIMD的全称是Single Instruction Multiple Data，即用单条指令操作多条数据。现代计算机系统概念中，它是通过数据并行以提高性能的一种实现方式（其他的还有指令级并行和线程级并行），它的原理是在CPU寄存器层面实现数据的并行操作。

ClickHouse目前利用SSE4.2指令集实现向量化执行。

### 关系模型与SQL查询
ClickHouse使用关系模型描述数据并提供了传统数据库的概念（数据库、表、视图和函数等）。与此同时，ClickHouse完全使用SQL作为查询语言（支持GROUP BY、ORDER BY、JOIN、IN等大部分标准SQL），这使得它平易近人，容易理解和学习。

在SQL解析方面，ClickHouse是大小写敏感的，这意味着SELECT a和SELECT A所代表的语义是不同的。

### 多样化的表引擎
也许因为Yandex.Metrica的最初架构是基于MySQL实现的，所以在ClickHouse的设计中，能够察觉到一些MySQL的影子，表引擎的设计就是其中之一。与MySQL类似，ClickHouse也将存储部分进行了抽象，把存储引擎作为一层独立的接口。ClickHouse共拥有合并树、内存、文件、接口和其他6大类20多种表引擎。其中每一种表引擎都有着各自的特点，用户可以根据实际业务场景的要求，选择合适的表引擎使用。

### 多线程与分布式
ClickHouse几乎具备现代化高性能数据库的所有典型特征，对于可以提升性能的手段可谓是一一用尽，对于多线程和分布式这类被广泛使用的技术，自然更是不在话下。

ClickHouse在数据存取方面，既支持分区（纵向扩展，利用多线程原理），也支持分片（横向扩展，利用分布式原理），可以说是将多线程和分布式的技术应用到了极致。

### 多主架构
HDFS、Spark、HBase和Elasticsearch这类分布式系统，都采用了Master-Slave主从架构，由一个管控节点作为Leader统筹全局。而ClickHouse则采用Multi-Master多主架构，集群中的每个节点角色对等，客户端访问任意一个节点都能得到相同的效果。这种多主的架构有许多优势，例如对等的角色使系统架构变得更加简单，不用再区分主控节点、数据节点和计算节点，集群中的所有节点功能相同。所以它天然规避了单点故障的问题，非常适合用于多数据中心、异地多活的场景。

### 数据分片与分布式查询
ClickHouse并不像其他分布式系统那样，拥有高度自动化的分片功能。ClickHouse提供了本地表（Local Table）与分布式表（Distributed Table）的概念。一张本地表等同于一份数据的分片。而分布式表本身不存储任何数据，它是本地表的访问代理，其作用类似分库中间件。借助分布式表，能够代理访问多个数据分片，从而实现分布式查询。

这种设计类似数据库的分库和分表，十分灵活。例如在业务系统上线的初期，数据体量并不高，此时数据表并不需要多个分片。所以使用单个节点的本地表（单个数据分片）即可满足业务需求，待到业务增长、数据量增大的时候，再通过新增数据分片的方式分流数据，并通过分布式表实现分布式查询。这就好比一辆手动挡赛车，它将所有的选择权都交到了使用者的手中。

## 架构设计
接下来会说明ClickHouse底层设计中的一些概念，这些概念可以帮助我们了解ClickHouse。

### Column与Field
Column和Field是ClickHouse数据最基础的映射单元。作为一款百分之百的列式存储数据库，ClickHouse按列存储数据，内存中的一列数据由一个Column对象表示。Column对象分为接口和实现两个部分，在IColumn接口对象中，定义了对数据进行各种关系运算的方法，例如插入数据的insertRangeFrom和insertFrom方法、用于分页的cut，以及用于过滤的filter方法等。而这些方法的具体实现对象则根据数据类型的不同，由相应的对象实现，例如ColumnString、ColumnArray和ColumnTuple等。在大多数场合，ClickHouse都会以整列的方式操作数据，但凡事也有例外。如果需要操作单个具体的数值（也就是单列中的一行数据），则需要使用Field对象，Field对象代表一个单值。与Column对象的泛化设计思路不同，Field对象使用了聚合的设计模式。在Field对象内部聚合了Null、UInt64、String和Array等13种数据类型及相应的处理逻辑。

### DataType
数据的序列化和反序列化工作由DataType负责。IDataType接口定义了许多正反序列化的方法，它们成对出现，例如serializeBinary和deserializeBinary、serializeTextJSON和deserializeTextJSON等，涵盖了常用的二进制、文本、JSON、XML、CSV和Protobuf等多种格式类型。IDataType也使用了泛化的设计模式，具体方法的实现逻辑由对应数据类型的实例承载，例如DataTypeString、DataTypeArray及DataTypeTuple等。

DataType虽然负责序列化相关工作，但它并不直接负责数据的读取，而是转由从Column或Field对象获取。在DataType的实现类中，聚合了相应数据类型的Column对象和Field对象。例如，DataTypeString会引用字符串类型的ColumnString，而DataTypeArray则会引用数组类型的ColumnArray，以此类推。

### Block与Block流
ClickHouse内部的数据操作是面向Block对象进行的，并且采用了流的形式。虽然Column和Filed组成了数据的基本映射单元，但对应到实际操作，它们还缺少了一些必要的信息，比如数据的类型及列的名称。于是ClickHouse设计了Block对象，Block对象可以看作数据表的子集。Block对象的本质是由数据对象、数据类型和列名称组成的三元组，即Column、DataType及列名称字符串。Column提供了数据的读取能力，而DataType知道如何正反序列化，所以Block在这些对象的基础之上实现了进一步的抽象和封装，从而简化了整个使用的过程，仅通过Block对象就能完成一系列的数据操作。在具体的实现过程中，Block并没有直接聚合Column和DataType对象，而是通过ColumnWithTypeAndName对象进行间接引用。

有了Block对象这一层封装之后，对Block流的设计就是水到渠成的事情了。流操作有两组顶层接口：IBlockInputStream负责数据的读取和关系运算，IBlockOutputStream负责将数据输出到下一环节。Block流也使用了泛化的设计模式，对数据的各种操作最终都会转换成其中一种流的实现。IBlockInputStream接口定义了读取数据的若干个read虚方法，而具体的实现逻辑则交由它的实现类来填充。

IBlockInputStream接口总共有60多个实现类，它们涵盖了ClickHouse数据摄取的方方面面。这些实现类大致可以分为三类：第一类用于处理数据定义的DDL操作，例如DDLQueryStatusInputStream等；第二类用于处理关系运算的相关操作，例如LimitBlockInput-Stream、JoinBlockInputStream及AggregatingBlockInputStream等；第三类则是与表引擎呼应，每一种表引擎都拥有与之对应的BlockInputStream实现，例如MergeTreeBaseSelect-BlockInputStream（MergeTree表引擎）、TinyLogBlockInputStream（TinyLog表引擎）及KafkaBlockInputStream（Kafka表引擎）等。

IBlockOutputStream的设计与IBlockInputStream如出一辙。IBlockOutputStream接口同样也定义了若干写入数据的write虚方法。它的实现类比IBlockInputStream要少许多，一共只有20多种。这些实现类基本用于表引擎的相关处理，负责将数据写入下一环节或者最终目的地，例如MergeTreeBlockOutputStream、TinyLogBlockOutputStream及StorageFileBlock-OutputStream等。

### Table
在数据表的底层设计中并没有所谓的Table对象，它直接使用IStorage接口指代数据表。表引擎是ClickHouse的一个显著特性，不同的表引擎由不同的子类实现，例如IStorageSystemOneBlock（系统表）、StorageMergeTree（合并树表引擎）和StorageTinyLog（日志表引擎）等。IStorage接口定义了DDL（如ALTER、RENAME、OPTIMIZE和DROP等）、read和write方法，它们分别负责数据的定义、查询与写入。在数据查询时，IStorage负责根据AST查询语句的指示要求，返回指定列的原始数据。后续对数据的进一步加工、计算和过滤，则会统一交由Interpreter解释器对象处理。对Table发起的一次操作通常都会经历这样的过程，接收AST查询语句，根据AST返回指定列的数据，之后再将数据交由Interpreter做进一步处理。

### Parser与Interpreter
Parser和Interpreter是非常重要的两组接口：Parser分析器负责创建AST对象；而Interpreter解释器则负责解释AST，并进一步创建查询的执行管道。它们与IStorage一起，串联起了整个数据查询的过程。Parser分析器可以将一条SQL语句以递归下降的方法解析成AST语法树的形式。不同的SQL语句，会经由不同的Parser实现类解析。例如，有负责解析DDL查询语句的ParserRenameQuery、ParserDropQuery和ParserAlterQuery解析器，也有负责解析INSERT语句的ParserInsertQuery解析器，还有负责SELECT语句的ParserSelectQuery等。

Interpreter解释器的作用就像Service服务层一样，起到串联整个查询过程的作用，它会根据解释器的类型，聚合它所需要的资源。首先它会解析AST对象；然后执行“业务逻辑”（例如分支判断、设置参数、调用接口等）；最终返回IBlock对象，以线程的形式建立起一个查询执行管道。

### Functions与Aggregate Functions
ClickHouse主要提供两类函数——普通函数和聚合函数。普通函数由IFunction接口定义，拥有数十种函数实现，例如FunctionFormatDateTime、FunctionSubstring等。除了一些常见的函数（诸如四则运算、日期转换等）之外，也不乏一些非常实用的函数，例如网址提取函数、IP地址脱敏函数等。普通函数是没有状态的，函数效果作用于每行数据之上。当然，在函数具体执行的过程中，并不会一行一行地运算，而是采用向量化的方式直接作用于一整列数据。

聚合函数由IAggregateFunction接口定义，相比无状态的普通函数，聚合函数是有状态的。以COUNT聚合函数为例，其AggregateFunctionCount的状态使用整型UInt64记录。聚合函数的状态支持序列化与反序列化，所以能够在分布式节点之间进行传输，以实现增量计算。

### Cluster与Replication
ClickHouse的集群由分片（Shard）组成，而每个分片又通过副本（Replica）组成。这种分层的概念，在一些流行的分布式系统中十分普遍。例如，在Elasticsearch的概念中，一个索引由分片和副本组成，副本可以看作一种特殊的分片。如果一个索引由5个分片组成，副本的基数是1，那么这个索引一共会拥有10个分片（每1个分片对应1个副本）。

如果你用同样的思路来理解ClickHouse的分片，那么很可能会在这里栽个跟头。ClickHouse的某些设计总是显得独树一帜，而集群与分片就是其中之一。这里有几个与众不同的特性。
- ClickHouse的1个节点只能拥有1个分片，也就是说如果要实现1分片、1副本，则至少需要部署2个服务节点。
- 分片只是一个逻辑概念，其物理承载还是由副本承担的。

### 算法在前，抽象在后
以字符串为例，有一本专门讲解字符串搜索的书，名为“Handbook of Exact String Matching Algorithms”，列举了35种常见的字符串搜索算法。各位猜一猜ClickHouse使用了其中的哪一种？答案是一种都没有。这是为什么呢？因为性能不够快。在字符串搜索方面，针对不同的场景，ClickHouse最终选择了这些算法：对于常量，使用Volnitsky算法；对于非常量，使用CPU的向量化执行SIMD，暴力优化；正则匹配使用re2和hyperscan算法。性能是算法选择的首要考量指标。




