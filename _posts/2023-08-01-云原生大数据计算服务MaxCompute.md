---
layout: post
categories: [Action]
description: none
keywords: Action
---
# 云原生大数据计算服务 MaxCompute

## 什么是MaxCompute
MaxCompute是适用于数据分析场景的企业级SaaS（Software as a Service）模式云数据仓库，以Serverless架构提供快速、全托管的在线数据仓库服务，消除了传统数据平台在资源扩展性和弹性方面的限制，最小化用户运维投入，使您可以经济并高效地分析处理海量数据。

MaxCompute还深度融合了阿里云如下产品：
- DataWorks
基于DataWorks实现一站式的数据同步、业务流程设计、数据开发、管理和运维功能。
- 机器学习PAI
基于机器学习平台的算法组件实现对MaxCompute数据进行模型训练等操作。
- 实时数仓Hologres
基于Hologres对MaxCompute数据进行外表查询加速，也可导出到Hologres进行交互式分析。
- Quick BI
基于Quick BI对MaxCompute数据进行报表制作，实现数据可视化分析。

## 核心概念的层次结构
通常MaxCompute的各层级概念的组织模式如下：

租户代表组织，以企业为例，一个企业可以在不同地域开通MaxCompute按量付费服务或预先购买包年包月计算资源（Quota）。

企业内的各个部门在开通服务的地域内创建和管理自己的项目（Project），用于存储该部门的数据。项目内可以存储多种类型对象，例如表（Table）、资源（Resource）、函数（Function）和实例（Instance）等。如果您有需要，也可以通过创建Schema，在项目之下进一步对上述对象进行归类，详情请参见Schema操作。各部门可以在项目内通过用户与角色的管控，对项目内的各类数据进行权限控制。

### 项目
项目（Project）是MaxCompute的基本组织单元，它类似于传统数据库的Database或Schema的概念，是进行多用户隔离和访问控制的主要边界。项目中包含多个对象，例如表（Table）、资源（Resource）、函数（Function）和实例（Instance）等。

### 配额
配额（Quota）是MaxCompute的计算资源池，为MaxCompute SQL、MapReduce、Spark、Mars、PAI等计算作业提供所需计算资源（CPU及内存）。

MaxCompute计算资源单位为CU，1 CU包含1 CPU及4 GB内存。

### 表
表是MaxCompute的数据存储单元。它在逻辑上是由行和列组成的二维结构，每行代表一条记录，每列表示相同数据类型的一个字段，一条记录可以包含一个或多个列，表的结构由各个列的名称和类型构成。

MaxCompute中不同类型计算任务的操作对象（输入、输出）都是表。您可以创建表、删除表以及向表中导入数据。

MaxCompute的表格有两种类型：内部表和外部表（MaxCompute 2.0版本开始支持外部表）。

对于内部表，所有的数据都被存储在MaxCompute中，表中列的数据类型可以是MaxCompute支持的任意一种数据类型版本说明。

对于外部表，MaxCompute并不真正持有数据，表格的数据可以存放在OSS或OTS中 。MaxCompute仅会记录表格的Meta信息，您可以通过MaxCompute的外部表机制处理OSS或OTS上的非结构化数据，例如，视频、音频、基因、气象、地理信息等。

### 分区
分区表是指拥有分区空间的表，即将表数据按照某个列或多个列进行划分，从而将表中的数据分散存储在不同的物理位置上。合理设计和使用分区，可以提高查询性能、简化数据管理，并支持更灵活的数据访问和操作。

分区可以理解为分类，通过分类把不同类型的数据放到不同的目录下。分类的标准就是分区字段，可以是一个，也可以是多个。

MaxCompute将分区列的每个值作为一个分区（目录），您可以指定多级分区，即将表的多个字段作为表的分区，分区之间类似多级目录的关系。

分区表的意义在于优化查询。查询表时通过WHERE子句查询指定所需查询的分区，避免全表扫描，提高处理效率，降低计算费用。使用数据时，如果指定需要访问的分区名称，则只会读取相应的分区。

### 生命周期
MaxCompute表的生命周期（Lifecycle），指表（分区）数据从最后一次更新的时间算起，在经过指定的时间后没有变动，则此表（分区）将被MaxCompute自动回收。这个指定的时间就是生命周期。

### 资源
资源（Resource）是MaxCompute的特有概念，如果您想使用MaxCompute的自定义函数（UDF）或MapReduce功能需要依赖资源来完成，如下所示：

SQL UDF：您编写UDF后，需要将编译好的JAR包以资源的形式上传到MaxCompute。运行此UDF时，MaxCompute会自动下载这个JAR包，获取您的代码来运行UDF，无需您干预。上传JAR包的过程就是在MaxCompute上创建资源的过程，这个JAR包是MaxCompute资源的一种。

MapReduce：您编写MapReduce程序后，将编译好的JAR包作为一种资源上传到MaxCompute。运行MapReduce作业时，MapReduce框架会自动下载这个JAR资源，获取您的代码。

### 函数
MaxCompute为您提供了SQL计算功能，您可以在MaxCompute SQL中使用系统的内建函数完成一定的计算和计数功能。但当内建函数无法满足要求时，您可以使用MaxCompute提供的Java或Python编程接口开发自定义函数（User Defined Function，以下简称UDF）。

自定义函数（UDF）可以进一步分为标量值函数（UDF），自定义聚合函数（UDAF）和自定义表值函数（UDTF）三种类型。

您在开发完成UDF代码后，需要将代码编译成Jar包，并将此Jar包以Jar资源的形式上传到MaxCompute，最后在MaxCompute中注册此UDF。

### 任务
任务（Task）是MaxCompute的基本计算单元，SQL及MapReduce功能都是通过任务完成的。

对于您提交的大多数任务，特别是计算型任务，例如SQL DML语句、MapReduce，MaxCompute会对其进行解析，得到任务的执行计划。执行计划由具有依赖关系的多个执行阶段（Stage）构成。

目前，执行计划逻辑上可以被看做一个有向图，图中的点是执行阶段，各个执行阶段的依赖关系是图的边。MaxCompute会依照图（执行计划）中的依赖关系执行各个阶段。在同一个执行阶段内，会有多个进程，也称之为Worker，共同完成该执行阶段的计算工作。同一个执行阶段的不同Worker只是处理的数据不同，执行逻辑完全相同。计算型任务在执行时，会被实例化，您可以对这个实例（Instance）进行操作，例如获取实例状态（Status Instance）、终止实例运行（Kill Instance）等。

### 任务实例
在MaxCompute中，SQL、Spark和Mapreduce任务在执行时会被实例化，以MaxCompute实例（下文简称为实例或Instance）的形式存在。实例会经历运行（Running）和结束（Terminated）两个阶段。


















