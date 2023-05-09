---
layout: post
categories: [JanusGraph]
description: none
keywords: JanusGraph
---
# JanusGraph
JanusGraph是一个开源的分布式图数据库。它的前身是著名的开源图数据库Titan，但Titan被DataStax收购之后就不再开源了。JanusGraph是在原Titan的基础上继续以开源的形式开发和发布，并于2017年由Linux基金会(Linux Foundation)开放管理。它的授权许可是Apache2 license，具有很好的商用友好性。

## 常见图库对比
目前主流数据库分为两个派系：Neo4j派 和 Tinkerpop派

## JanusGraph 与 Tinkerpop 与 Gremlin
Tinkerpop 是Apache基金会下的一个开源的图数据库与图计算框架（OLTP与OLAP）。

Gremlin 是Tinkerpop的一个组件，它是一门路径导向语言，用于图操作和图遍历（也称查询语言）。Gremlin Console 和 Gremlin Server 分别提供了控制台和远程执行Gremlin查询语言的方式。Gremlin Server 在 JanusGraph 中被成为 JanusGraph Server。

JanusGraph是基于Tinkerpop这个框架来开发的，使用的查询语言也是Gremlin。

## 历史背景
JanusGraph图数据库，源自于TitanDB开源图数据库。TitanDB在2012年发布第一个版本，2015年被Datastax公司收购，后续不再维护导致项目停滞。

图数据库有2个最具代表性的查询语言：Cypher 及 Gremlin。Cypher是商业公司Neo4j出品，Neo4j图数据库在2007年发布了第一个版本，是商用图数据库领域的开拓者。Gremlin是Apache TinkerPop框架下规范的语言，TinkerPop属于当前图数据库领域最流行的框架规范，具备开源开放、功能丰富、生态完善等特点，拥有大量厂商支持（超过20家），Titan当属TinkerPop框架下最成功的开源图数据库实现，后续的不少图数据库或多或少借鉴了Titan的思想，Titan的几位核心作者包括：Dan、Matthias、Marko(okram)、Stephen(spmallette)等，其中的两位--Marko和Stephen同时也是TinkerPop的核心作者。个人在此致敬Titan和TinkerPop。

非常令人遗憾的是，Titan在2015年被收购后，其开源社区无人维护，否则以其前三年的势头来看，大有一统图江湖的趋势，不过没有如果。而Janus稍许弥补了这个遗憾，也算是后继之人，但是Janus绝大部分功能沿袭自Titan，没有更上一个台阶发扬光大，只是做了一些小修补。下面是对Titan/Janus的一些总结和分析。

2016年由其他人基于Titan源码Fork出了Janus，到目前（2020年）Janus已经合入了700多个Pull Request，总结来说，在大方向上Janus对Titan改进并不多。

## JanusGraph特征
JanusGraph 具有很好的扩展性，通过多机集群可支持存储和查询数百亿的顶点和边的图数据。JanusGraph是一个事务数据库，支持大量用户高并发地执行复杂的实时图遍历。

它提供了如下特性：
- 支持数据和用户增长的弹性和线性扩展;
- 通过数据分发和复制来提升性能和容错;
- 支持多数据中心的高可用和热备份;
- 支持ACID 和最终一致性;
- 支持多种后端存储：

## JanusGraph概述
JanusGraph是一个图形数据库引擎。其本身专注于紧凑图序列化、丰富图数据建模、高效的查询执行。另外，JanusGraph利用Hadoop进行图分析和批处理图处理。JanusGraph为数据持久性、数据索引、客户端访问实现了强大的模块化接口。JanusGraph的模块化体系结构使其可以与多种存储、索引、客户端技术进行互操作。它还简化了扩展JanusGraph以支持新的过程。
作为一款开源产品，JanusGraph主要定位是做图数据的是实时检索。特性支持动态扩展、良好的性能指标、支持TinkerPop框架、支持Gremlin语言及技术栈、支持多种底层存储。

## JanusGraph架构
Janus整体架构分为3层，中间层是图引擎，最底层是存储层，最上层是应用程序层：
- 图引擎
图数据库核心，对外提供图方式的读写API。处理写请求，将数据索引起来、按照特定的格式写到存储层；处理读请求，按照特定格式从存储层高效读取数据，组装成图结果返回上层。
- 存储层
用于持久化图数据，包括两方面数据：1、图数据（包括顶点和边），存储在Storage Backend里面，可以选择配置一种后端存储，比如Cassandra或HBase；2、Mixed模糊索引数据（包括范围索引和全文索引等），存储在External Index Backend里面，也可以选则配置一种后端存储，比如ES或Solr；如果不开启Mixed模糊索引，则无需配置索引后端。
- 应用层
业务程序可以有两种方式来使用：1、内嵌到程序中，与业务程序共用同一个进程JVM；2、以服务的方式单独启动，业务程序通过HTTP API来访问。

## 数据模型/存储结构（整体）
图概念简介：图的核心是顶点和边，顶点代表现实世界中的实体，边代表现实世界中的关系，“我喜欢你”就可以抽象为两个顶点和一条边，同时顶点和边还可以有属性。

图是如何被存储的？在Titan/Janus中，使用的是邻接表存储结构（另一种知名的结构是Neo4j的邻接链表）。Janus把每个顶点的数据存为一行，当插入图数据时，会为每个顶点分配一个递增的Long类型ID（vertex id），查询的时候使用这个ID来进行索引查找。顶点的数据包括两方面内容：顶点属性（property）和邻接边（edge），每个顶点属性存为一列，每条邻接边也存为一列。

一个顶点的各顶点属性和边数据是按照顺序排列的，内部规则如下：

顶点属性：每个顶点属性包括：属性类型（key id）、属性ID（property id）、属性值（property value），各顶点属性按照属性类型的顺序依次存储。

边数据：边复杂一些，这里简化下讲最重点的核心思想，每条邻接边包括：边类型（label id）、边终点（adjacent vertex id）、边属性（other properties），各邻接边按照边类型+边终点组合的顺序依次存储。

Janus这种存储结构的优点是：
- 可以高效的查询一个点的邻接边：当需要查询一个顶点的所有邻接边时，先通过该顶点的ID快速定位到某一行，然后对这一行内的所有列依次读取，高效返回数据。
- 可以高效的查询一个超级点的小部分邻接边：如果只需要查询一个顶点的某种类型的邻接边时，可以根据边类型进行条件筛选，忽略掉其它类型的邻接边，加快读取速度。
针对上述第2个场景，邻接链表的存储结构是难于进行类似性能优化的。因此当一个顶点的邻接边超多时，即使用户只需要查询其部分邻接边，也还是需要从磁盘先读取所有邻接边，从而导致性能低下。

## 图谱平台的功能组件
图谱平台不仅仅是一个图数据库。一个完整的图谱平台应该包括如下功能组件。

展示。通过js库实现可视化和交互。要实现体验比较要的前端，需要一定的投入。目前有d3.js等开源js库。

实时计算。图谱平台的核心组件，完成实时计算的功能，实时计算也是目前最多的应用场景。TinkerPop框架就是专注这一功能。实时计算的底层存储又分为几类，neo4j采用了自研算法和自建存储，JanusGraph等多个数据库兼容Hbase等分布式存储。在这一点上JanusGraph的技术栈选择更加通用，工程意义更大。

离线分析。针对大数据场景，需要完成各类离线计算功能。目前neo4j有一定的图谱遍历能力，但是不够通用，技术栈也和hadoop不够集成。JanusGraph，以及国内多个大厂产品如HugeGraph等本身不支持离线分析。比较有名的图数据离线分析软件是Spark GraphX，底层存储也可选用Hbase等。

管理。图的管理至少应包含本体管理、图谱管理、权限管理和运维监控。管理端应该有可视化前端支撑。本体管理实现本体构建、数据导入功能，图谱管理是图基础信息的管理，权限管理至少包括多租户管理，运维监控支撑日常运维工作。

## JanusGraph的安装运行
JanusGraph支持原生安装和Docker安装两种。

原生安装。官方提供了janusgraph-0.4.1-hadoop2，其中主要插件包含gremlin-console作为客户端，gremlin-server和janusgraph-server作为服务端。特别注意直接使用TinkerPop提供的gremlin-console要正确连接janusGraph，需要安装gremlin janusGraph插件，gremlin的官方文档这方面描述比较简单，不易安装。可以考虑直接使用janusgraph-0.4.1-hadoop2已配置完成的gremlin-console。

Docker安装。官方镜像janusgraph-0.4.1-hadoop2默认是在单个容器内运行gremlin-server janusGraph-server和多个内置存储，如Cassandra。考虑真实使用场景，应该将使用外部存储如Hbase和Es。官方镜像的主要意义就是提供了镜像模板，可根据需要加入正确的配置。

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

## 属性图底层存储（Hbase）
Tinkerpop 下有较多的属性图实现：IBM Graph、Titan、JanusGraph、HugeGraph，均支持多后端存储，多模式也是目前图数据库发展的的一个大方向。多模式无疑可以满足更多的用户，降低了数据迁移和维护的成本。但从另一方面来看，多个后端存储也带来了一些弊端：
- 我们就需要在软件架构进行抽象，增加一个可以适配多个存储的数据格式（StaticBuffer），数据无论是写入还是读取，都需要先转化成中间格式，这里带来了序列化和反序列化的一些性能损耗；
- 抽象后的架构，对外是统一的，不利于我们发挥后端的存储查询优势（如 Hbase 的 Coprocessor，是可以加速查询的），为了使用这种能力，我们需要破坏这种统一的架构去适配后端存储。

下面主要以 JanusGraph + Hbase 这套组合为例，介绍其存储过程（不同的存储后端存储格式不一样）。

JanusGraph 采用的分片方式（也有按照点切割的图数据库）是按Edge切割，而且是对于每一条边，都会被切断。切断后，该边会在起始 Vertex 上和目的 Vertex 上各存储一次（多浪费了空间）。通过这种方式，JanusGraph 不管是通过起始 Vertex，还是从目的 Vertex，都能快速找到对端 Vertex。

- Hbase 每一行存储一个顶点，RowKey 为 Vertex Id；
- 一个 Vertex 的 Properties 信息，以及与该 Vertex 相关的 Edges，都以独立的列存储，而且被存成了一行数据；
- 表示 Edge 的列中，包含了 Label 信息，Edge ID，相邻 Vertex 信息，属性等信息；
- 表示 Vertex Property 的列中，包含了 Property 的 ID，以及 Property 的值；

注意，Vertex/Edge/Property 在创建时，都会分配一个 ID，主要的逻辑在Janusgraph-core包中的org.janusgraph.graphdb.idmanagement.IDManger类中，
下面是给顶点增加 ID 的过程。
```
public long getVertexID(long count, long partition, VertexIDType vertexType) {
  	Preconditions.checkArgument(VertexIDType.UserVertex.is(vertexType.suffix()),"Not a user vertex type: %s",vertexType);
  	Preconditions.checkArgument(count>0 && count<vertexCountBound,"Invalid count for bound: %s", vertexCountBound);
  	if (vertexType==VertexIDType.PartitionedVertex) {
    		Preconditions.checkArgument(partition==PARTITIONED_VERTEX_PARTITION);
    		return getCanonicalVertexIdFromCount(count);
  	} else {
    		return constructId(count, partition, vertexType);
  	}
}

```

## JanusGraph 查询示例
以下面的查询语句为例，具体的查询过程如下所示：
```
g.v("vid").out.out.has(name, "jack")
```
- v("vid")：把 id 为 “vid” 的节点找出来，返回该节点，这里可能会用到索引；
- out：从上一步结果集合中，拉出一个，即 “vid” 的 id，并把该点对应的那行数据从hbase里读取出来（即该点的属性、相邻点、相邻边），返回出度节点，返回结果edgeList1；
- out：从上一步结果edgeList1中，拉出一个，即把第一个出度点拉出来，并把该点对应的那行数据从 hbase 里读取出来（即该点的属性、相邻点、相邻边），找出出度节点，返回结果edgeList2；
- has：把edgeList2中的第一个节点拉出来，把该点对应的属性字段从 hbase 里读取出来，并进行name为jack的过滤，返回结果；
- 迭代执行第4步，直至edgeList2遍历完毕；
- 返回第3步，直至edgeList1遍历完毕；
- 返回结果。

## JanusGraph 索引
JanusGraph 支持两种类型的索引：graph index 和 vertex-centric index。graph index 常用于根据属性查询 Vertex 或 Edge 的场景；vertex index 在图遍历场景非常高效，尤其是当 Vertex 有很多 Edge 的情况下。

### Graph Index
Composite index：Composite index通过一个或多个固定的key（schema）组合来获取 Vertex Key 或 Edge，也即查询条件是在Index中固定的，Composite index 只支持精确匹配，不支持范围查询。Composite index 依赖存储后端，不需要索引后端。

Mixed Index：支持通过其中的任意 key 的组合查询 Vertex 或者 Edge，使用上更加灵活，而且支持范围查询等，但 Mixed index 效率要比 Composite Index 低。与 Composite key 不同，Mixed Index 需要配置索引后端，JanusGraph 可以在一次安装中支持多个索引后端。

举例：
- Composite Index:
```
// 顶点中含有name属性且值为jack的所有顶点
g.V().has('name', 'jack')
```
- Mixed Index:
```
// 顶点中含有age属性且小于50的所有顶点
g.V().has('age', lt(50))
```

### Vertex-Centric Index
Vertex-centric index（顶点中心索引）是为每个 vertex 建立的本地索引结构，在大型 graph 中，每个 vertex 有数千条Edge，在这些 vertex 中遍历效率将会非常低（需要在内存中过滤符合要求的 Edge）。Vertex-centric index 可以通过使用本地索引结构加速遍历效率。

举例下面的查询中，如果对'battled'类型的边属性'rating'建立了属性，则是可以利用上索引的。
```
h = g.V().has('name','hercules').next()
g.V(h).outE('battled').has('rating', gt(3.0)).inV()
```
注意：JanusGraph 自动为每个 edge label 的每个 property key 建立了 vertex-centric label，因此即使有数千个边也能高效查询。

## JanusGraph批量导入
janus的导入的常用方案
- 基于JanusGraph Api的批量导入
- 基于Gremlin Server的批量导入
- 使用JanusGraph-utils的批量导入
- 基于bulk loader 导入方式

### 基于JanusGraph Api的数据导入
```java
public JanusGraphVertex addVertex(Object... keyValues);
public JanusGraphEdge addEdge(String label, Vertex vertex, Object... keyValues);
```
在janusGraph的业务项目中，可以开发一个数据导入模块，使用提供的类似于java api等，进行数据的导入；
```java
import org.apache.tinkerpop.gremlin.process.traversal.dsl.graph.GraphTraversalSource;
import org.apache.tinkerpop.gremlin.structure.Vertex;

import static org.apache.tinkerpop.gremlin.process.traversal.AnonymousTraversalSource.traversal;

public class JanusGraphImportDemo {
    public static void main(String[] args) throws Exception {

        GraphTraversalSource g = traversal().withRemote("conf/remote-graph.properties");

        Vertex next0 = g.addV("Company").property("name", "西安仁昌建筑装饰工程有限公司").property("alias", "西安仁昌").next();
        Vertex next1 = g.addV("Company").property("name", "福州兴嘉创装饰涂料有限公司闽侯县上街分公司").property("alias", "兴嘉创").next();
        Vertex next2 = g.addV("Company").property("name", "福州高弘达建筑装饰有限公司").property("alias", "高弘达").next();
        Vertex next3 = g.addV("Company").property("name","上海老板电器销售有限公司崇明分公司").property("alias","老板电器").next();
        Vertex next4 = g.addV("Company").property("name","广州白云区前睿商贸有限公司").property("alias","前睿商贸").next();

        Vertex next11 = g.addV("Person").property("name","张三").property("alias","zhangsan").property("sex","男").next();
        Vertex next12 = g.addV("Person").property("name","李四").property("alias","lisi").property("sex","女").next();
        Vertex next13 = g.addV("Person").property("name","王五").property("alias","wangwu").next();

        g.addE("法定代表人").from(next0).to(next11).property("relation","法定代表人").next();
        g.addE("法定代表人").from(next1).to(next12).property("relation","法定代表人").next();
        g.addE("法定代表人").from(next2).to(next13).property("relation","法定代表人").next();
        g.addE("法定代表人").from(next3).to(next11).property("relation","法定代表人").next();
        g.addE("法定代表人").from(next4).to(next12).property("relation","法定代表人").next();

        g.addE("实际控制人").from(next0).to(next11).property("relation","实际控制人").property("percent","50%").next();
        g.addE("实际控制人").from(next0).to(next12).property("relation","实际控制人").property("percent","30%").next();
        g.addE("实际控制人").from(next0).to(next13).property("relation","实际控制人").property("percent","20%").next();

    }
}
```

### IBM的janusgraph-utils
主要也是通过多线程对数据进行导入；
自己手动组装对应的schema文件，将schema导入到数据库； 然后将组装为特定格式的csv文件中的数据，导入到图库中；
github地址： https://github.com/IBM/janusgraph-utils


# 参考资料
中文文档(http://janusgraph.cn/index.html)[http://janusgraph.cn/index.html]

官网文档(https://docs.janusgraph.org/)[https://docs.janusgraph.org/]



