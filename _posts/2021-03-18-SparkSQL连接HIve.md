---
layout: post
categories: [Spark]
description: none
keywords: Spark
---
# SparkSQL连接Hive
目前业界主流的SQL引擎都支持与Hive的连接，Spark也不例外。在Spark 2.0版本之前，Spark SQL与Hive的连接中除Hive元数据（Metastore）外，还依赖Hive完成语法解析。从Spark 2.0版本开始，Spark SQL引入ANTLR 4作为SQL编译器，摆脱了对Hive编译器的依赖。

## Spark SQL连接Hive概述
从Spark 2.0版本开始，连接Hive不再需要创建单独的HiveContex，而是在SparkSession中通过enableHiveSupport方法开启Hive的支持。下面是example目录中连接Hive的简单应用代码。可见在Spark SQL连接Hive场景下，不再需要和DataFrame等API打交道，所有的处理与查询都直接通过SQL语句完成。
```

```
在这个例子中，当SparkSession中开启了Hive支持后，配置信息conf中会将Catalog信息（spark.sql.catalogIm plementation）设置为“hive”，这样在SparkSession根据配置信息反射获取SessionState对象时就会得到与Hive相关的对象。第3章已经介绍过，Spark SQL本身的核心类是SessionState，该类中整合了Catalog、Parser、Analyzer、SparkPlanner等所有必需的对象。同样的，在连接Hive时，也存在类似的对象HiveSessionSate。

为了支持Spark SQL连接Hive，在HiveSessionState中主要重载了3个对象：Catalog、Analyzer和SparkPlanner。其中，Catalog具体实现为HiveSessionCatalog，Analyzer中加入了Hive相关的分析规则，SparkPlanner中加入了Hive相关的策略，其他的部分(如Parser等)则直接复用Spark SQL本身的对象。

## Hive相关的规则和策略
从实现层面看，Spark SQL连接Hive的主要工作体现在元数据、分析规则和转换策略方面。 本节将从这3个方面详细介绍HiveSessionCatalog体系、Hive-Specific分析规则和Hive-Specific转换策略。

### HiveSessionCatalog体系
