---
layout: post
categories: [JanusGraph]
description: none
keywords: JanusGraph
---
# JanusGraph
JanusGraph是一个可扩展的图形数据库，用于存储和查询分布在多机集群中的包含数千亿顶点和边的图形。该项目由Titan出资，并于2017年由Linux基金会(Linux Foundation)开放管理。

## 常见图库对比
目前主流数据库分为两个派系：Neo4j派 和 Tinkerpop派

## JanusGraph 与 Tinkerpop 与 Gremlin
Tinkerpop 是Apache基金会下的一个开源的图数据库与图计算框架（OLTP与OLAP）。

Gremlin 是Tinkerpop的一个组件，它是一门路径导向语言，用于图操作和图遍历（也称查询语言）。Gremlin Console 和 Gremlin Server 分别提供了控制台和远程执行Gremlin查询语言的方式。Gremlin Server 在 JanusGraph 中被成为 JanusGraph Server。

JanusGraph是基于Tinkerpop这个框架来开发的，使用的查询语言也是Gremlin。

## JanusGraph概述
JanusGraph是一个图形数据库引擎。其本身专注于紧凑图序列化、丰富图数据建模、高效的查询执行。另外，JanusGraph利用Hadoop进行图分析和批处理图处理。JanusGraph为数据持久性、数据索引、客户端访问实现了强大的模块化接口。JanusGraph的模块化体系结构使其可以与多种存储、索引、客户端技术进行互操作。它还简化了扩展JanusGraph以支持新的过程。

## 交互
应用程序可以通过两种方式与JanusGraph交互：

- 嵌入式： 将JanusGraph嵌入到自己的图Gremlin查询应用中，与自己的应用公用同一JVM。
查询执行时，JanusGraph的缓存和事务处理都在与应用程序相同的JVM中进行，当数据模型大时，很容易OOM，并且耦合性太高，生产上一般不这么搞。

- 服务式： JanusGraph单独运行在一个或一组服务器上，对外提供服务。（推荐的模式）
客户端通过向服务器提交Gremlin查询，与远程JanusGraph实例进行交互。

## Docker安装JanusGraph
JanusGraph提供Docker image。 Docker使得在单台机器上安装和运行Janusgraph服务和客户端变得简单容易。有关Janusgraph的安装和使用说明请参照 docker guide。
下面就举例如何使用Docker技术来安装和运行JanusGraph:
```
$ docker run -it -p 8182:8182 janusgraph/janusgraph
```
运行docker命令，获取janusgraph的Dockerimage并运行于Docker容器，8182端口作为服务端口暴露对外。启动日志如下:

等待服务启动完成后，就可以启动Gremlin Console来连接新安装的janusgraph server:
Gremlin Console是一个交互式shell，允许您访问JanusGraph图数据。你可以通过运行位于项目的bin目录中的gremlin.sh脚本来运行。
```
$ bin/gremlin.sh

         \,,,/
         (o o)
-----oOOo-(3)-oOOo-----
gremlin> :remote connect tinkerpop.server conf/remote.yaml
==>Configured localhost/127.0.0.1:8182
```
Gremlin Console命令采用的是Apache Groovy,。Gremlin-Groovy集成自Groovy提供了许多基本或高级的图遍历方法。

为了方便起见，这里在同一台机器同时安装JanusGraph Server和Gremlin Console Client，首先在机器上创建Docker容器来运行JanusGraph Server:
```
$ docker run --name janusgraph-default janusgraph/janusgraph:latest
```
然后在这台机器上再另启一个容器来运行Gremlin Console Client并连接运行中的Janusgraph Server。
```
$ docker run --rm --link janusgraph-default:janusgraph -e GREMLIN_REMOTE_HOSTS=janusgraph \
    -it janusgraph/janusgraph:latest ./bin/gremlin.sh

         \,,,/
         (o o)
-----oOOo-(3)-oOOo-----
gremlin> :remote connect tinkerpop.server conf/remote.yaml
==>Configured janusgraph/172.17.0.2:8182
```

Gremlin server启动完成后，默认端口是8182。下面我们启动Gremlin Console使用如下:remote的远程连接命令，将Gremlin Console连接到Server上。
```
gremlin> :remote connect tinkerpop.server conf/remote.yaml
==>Configured localhost/127.0.0.1:8182
```
从日志中可以看出，在本例中，客户机和服务器运行在同一台机器上。当使用不同的设置时，您所要做的就是修改conf/remote.yaml文件。

## 将图加载到JanusGraph
将一个图数据加载到Janusgraph主要通过JanusGraphFactory和JanusGraphFactory两个类。首先利用JanusGraphFactory提供的静态open方法，以配置文件路径作为输入参数，运行并返回graph实例(示例中的配置参考配置参考章节中介绍)。然后利用GraphOfTheGodsFactory的load方法并将返回的graph实例作为参数运行，从而完成图加载到JanusGraph中。

将图加载到带索引后端的Janusgraph
下面的conf/janusgraph-berkeleyje-es.properties配置文件配置的是以BerkeleyDB作为存储后端，Elasticsearch作为索引后端的Janusgraph。
```
gremlin> graph = JanusGraphFactory.open('conf/janusgraph-berkeleyje-es.properties')
==>standardjanusgraph[berkeleyje:../db/berkeley]
gremlin> GraphOfTheGodsFactory.load(graph)
==>null
gremlin> g = graph.traversal()
==>graphtraversalsource[standardjanusgraph[berkeleyje:../db/berkeley], standard]
```
JanusGraphFactory.open()和GraphOfTheGodsFactory.load()完成以下操作
- 为图创建全局和vertex-centric的索引集合。
- 加载所有的顶点及其属性。
- 加载所有的边及其属性。







