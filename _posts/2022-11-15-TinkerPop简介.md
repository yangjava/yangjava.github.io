---
layout: post
categories: [TinkerPop]
description: none
keywords: TinkerPop
---
# TinkerPop简介
TinkerPop是一种开源图计算框架，既可用于联机事务处理（OLTP），又可用于联机分析处理系统（OLAP）。它可以用于处理单一机器以及分布式环境的庞大数据。

## 简介
该项目专注于为图数据库建立行业标准，包括一种名为Gremlin的标准查询语言。与此同时，Gremlin遍历机器旨在跨语言工作。

TinkerPop是Apache软件基金会旗下的一个顶级开源项目。

## Tinkerpop3 模型核心概念
- Graph: 维护节点&边的集合，提供访问底层数据库功能，如事务功能
- Element: 维护属性集合，和一个字符串label，表明这个element种类
- Vertex: 继承自Element，维护了一组入度，出度的边集合
- Edge: 继承自Element，维护一组入度，出度vertex节点集合.
- Property: kv键值对
- VertexProperty: 节点的属性，有一组健值对kv，还有额外的properties 集合。同时也继承自element，必须有自己的id, label.
- Cardinality: 「single, list, set」 节点属性对应的value是单值，还是列表，或者set。

## 整体结构
从官方文档的介绍可以了解到要想具备图数据库的基本功能，需要实现Core API，如果想要具备OLAP的功能需要实现Graph Computer的所有接口。

- Provider Strategies是一类优化的策略，可以应用在正式的图查询过程前，主要是对图查询的步骤进行优化。允许底层图系统在执行Gremlin查询时对其进行性能上的优化（如:多种索引查询, 特定算法优化, 遍历步骤重排）
- Gremlin Traversal Language是图查询语言，通常称为gremlin，以查询顶点为例，语法如下形式：
```
v = g.addV().property('name','marko').property('name','marko a. rodriguez').next()
```

- Gremlin Server是TinkerPop的服务器，它的作用是处理图数据库客户端发送过来的网络请求，解析gremlin查询语句，并对解析出来的每个step进行处理，得到查询结果。

## 源码结构
TinkerPop的源码结构大致有如下7个模块:
- gremlin-console
访问gremlin.sh的时候显示的控制台交互逻辑, 基本是groovy实现的。
- gremlin-groovy
实现Gremlin的代码, 以及在gremlin中支持groovy/其它DSL的写法, 可定制策略, 内容很杂很多。
- gremlin-core 
定义图模型结构 , OLTP和OLAP接口。 
  - 图的数据结(Vertex,VertexProperty,Edge,Property,Element,Transaction)。实现了以上所有的接口的图系统, 我们就可以说它实现了图的OLTP, 比如ugeGraph/JanusGraph/TinkerGraph。而实现后就可以使用Gremlin语法。
  - 数据的导入和导出
  - 特殊功能的图实现 (比如做缓存图)
  - 图的OLAP也需要单独实现, 比如和spark的结合已有的graphX或Giraph。

- spark-gremlin(OLAP)
是一个tinkerpop基于spark的OLAP实现, 还有hadoop-gremlin。
- giraph-gremlin(OLAP)
Giraph 源自Google在2010年发表的Pregel计算框架, 是它的开源版本,运行在hadoop之上,是一种仅有Map的MR作业。是类似GraphX/GraphLab的图计算框架。
- tinkergraph-gremlin
TinkerPop自带的内存图数据库, 带有OLAP+OLTP 的最简实现 ,上手测试用，是TinkerPop框架的参考实现。具体图数据库提供商实现可以看看,毕竟小巧。
- benchmark和driver
跑性能测试,和各种连接语言所用的数据库驱动测试之类。

## 图计算=图结构+图处理
图计算包括图结构（Structure）和图处理/过程（Process）两部分，图结构就是图的数据结构，有顶点、边、属性等。图过程就是分析图结构的手段，通常用称为遍历（traversal）。

一个图是顶点（节点，点）和边缘（弧，线）组成的数据结构。当在计算机中为图建模并将其应用于现代数据集和实践时，通用的面向数学的二进制图已扩展为既支持标签又支持键/值属性。这种结构称为属性图。更正式地说，它是有向的，二进制的，有属性的多图。下面（TinkerPop Modern）是一个示例属性图。

## TinkerPop 结构 API的主要组件
- Graph：维护一组顶点和边，并访问数据库功能（例如事务）
- Element：维护属性的集合和表示元素类型的字符串标签
- Vertex：扩展Element并维护一组传入和传出边缘
- Edge：扩展Element并维护传入和传出的顶点
- Property：与V值关联的字符串键
- VertexProperty：与V值以及Property属性集合关联的字符串键（仅顶点）

## TinkerPop 过程 API的主要组件
- TraversalSource：针对特定图形，领域特定语言（DSL）和执行引擎的遍历生成器
- Traversal：一个功能数据流过程，将类型的起始S对象转换为类型的终止E对象
- GraphTraversal：面向原始图的语义（即顶点，边等）的遍历DSL
- GraphComputer：并行处理图形的系统，并有可能分布在多计算机集群上
- VertexProgram：在所有顶点以逻辑并行方式执行的代码，并通过消息传递进行互通
- MapReduce：一种计算，可以并行分析图中的所有顶点，并产生一个简化的结果

## 如何使用TinkerPop构建一个图数据库
TinkerPop框架提供了图数据库的基础性功能，其中包括处理网络请求、解析gremlin查询语言、定义图中顶点和边结构的父类、定义Step的执行逻辑等。具体到源码中的neo4j-gremlin、hadoop-gremlin、tinkergraph-gremlin等模块，其作用是用于处理经TinkerPop框架处理后的数据的存储终点问题，neo4j-gremlin、hadoop-gremlin模块用于将数据最终存储在neo4j图数据库和hadoop文件系统中，而tinkergraph-gremlin模块用于将数据存储在内存中。

如果想把经TinkerPop框架处理后的数据存储在不同的中间件上，需要自己在TinkerPop框架基础上实现该功能，当然也可以使用源码中提供的类似neo4j-gremlin等模块。


# 参考资料
TinkerPopo官方地址 (https://tinkerpop.apache.org/download.html)[https://tinkerpop.apache.org/download.html]

