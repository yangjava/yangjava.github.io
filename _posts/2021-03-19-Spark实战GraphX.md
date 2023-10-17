---
layout: post
categories: [Spark]
description: none
keywords: Spark
---
# Spark实战GraphX
介绍Spark平台中的GraphX图计算框架、GraphX编程相关图操作和图算法，以及针对不同的应用场景进行图计算应用实践。

## 概述
GraphX是Spark中用于图和图并行计算的组件，从整体上看，GraphX通过扩展Spark RDD引入一个新的图抽象，一个将有效信息放在顶点和边的有向多重图。为了支持图形计算，GraphX公开了一系列基本运算（如subgraph、joinVertices和aggregateMessages），以及一个优化后的Pregel API的变形。此外，GraphX包括越来越多的图形计算和builder构造器，以简化图形分析任务。与其他分布式图计算框架相比，GraphX最大的贡献是，在Spark之上提供了一站式解决方案，可以方便且高效地完成图计算的一整套流水作业。

## Spark GraphX架构
在GraphX设计时，点分割和GAS都已经成熟，所以GraphX一开始就站在了巨人的肩膀上，并在设计和编码中，针对这些问题进行了优化，在功能和性能之间寻找最佳的平衡点。

每个Spark子模块，如同Spark本身一样，都有一个核心的抽象。GraphX的核心抽象是弹性分布式属性图（resilient distributed property graph），一种点和边都带属性的有向多重图。它扩展了Spark RDD的抽象，拥有Table和Graph两种视图，而只需要一份物理存储。而这两种视图都有自己独有的操作符，从而获得灵活的操作和较高的执行效率。

大部分的impl包都是围绕Partition（EdgePartition、VertexPartition、MessagePartition、RoutingTablePartition）的优化进行实现，分割的存储和相应的计算优化，是图计算框架的重点和难点。

## GraphX编程
属性图是一个用户定义顶点和边的有向多重图。有向多重图是一个有向图，它可能有多个平行边共享相同的源顶点和目标顶点。多重图支持并行边的能力简化了有多重关系的建模场景。每个顶点是由具有64位长度的唯一标识符（VertexID）作为主键。GraphX并没有对顶点添加任何顺序的约束。同样，每条边具有相应的源顶点和目标顶点的标识符。

属性表的参数由顶点（VD）和边（ED）的类型决定。 GraphX优化了顶点和边的类型表示方法，针对以前的数据类型（如Int、Double等）通过存储在专门的阵列来减小内存使用。

在某些情况下，可能希望顶点在同一个图中有不同的属性类型，这可以通过继承来实现。

例如，以用户和产品型号为二分图我们可以做到以下几点：
```
class VertexProperty()
case class UserProperty(val name: String) extends VertexProperty
case class ProductProperty(val name: String, val price: Double) extends VertexProperty
// The graph might then have the type:
var graph: Graph[VertexProperty, String] = null
```
和RDD一样，属性图具有不可变、分布式和容错的特点。对属性图值或结构的改变是通过生成新图完成的。原图的主要部分（属性和索引）被重用，通过启发式执行顶点分区，在不同的执行器中进行顶点的划分，从而减少新属性图的生成成本；与RDD一样，在发生故障的情况下，图中的每个分区都可以重建。

从逻辑上讲，属性图包含自身的顶点Vetex RDD[VD]和边EdgeRDD[ED]（RDD），记录每个顶点和边的属性。因此，该图表类包含该图的顶点和边。
```
class Graph[VD, ED] {
  val vertices: VertexRDD[VD]
  val edges: EdgeRDD[ED]
}
```
类VertexRDD[VD]和EdgeRDD[ED]分别继承优化了的RDD[（VertexID，VD）]和RDD[Edge[ED]]，提供各地图的计算内置附加功能，并充分利用内部优化。

GraphX公开了属性图RDD顶点（VD）和边（ED）的视图，这是与每个顶点和边相关联对象的类型。GraphX将顶点和边保存在优化的数据结构，并且为这些数据结构提供额外的功能，顶点和边分别作为VertexRDD和EdgeRDD对象返回。

### VertexRDD
VertexRDD[A]继承RDD[（VertexID，A）]，并增加了一些额外的限制，每个VertexID只出现一次。此外，VertexRDD[A]表示具有A属性的顶点集合。在内部通过将顶点属性存储在一个可重复使用的哈希表。因此，如果两个VertexRDD继承自相同的基类VertexRDD（如filter或mapValues），它们可以在常数时间内实现合并，而不需要重新计算散列值。

要充分利用这个索引数据结构，VertexRDD提供了以下附加功能：
```
class VertexRDD[VD] extends RDD[(VertexID, VD)] {
  // 过滤

Vertice集合


  def filter(pred: Tuple2[VertexId, VD] => Boolean): VertexRDD[VD]
  // 转换值而不改变

ids(保存内部索引

)
  def mapValues[VD2](map: VD => VD2): VertexRDD[VD2]
  def mapValues[VD2](map: (VertexId, VD) => VD2): VertexRDD[VD2]
  // 找出与其他集合相同的

Vertex，并在当前集合中删除


  def diff(other: VertexRDD[VD]): VertexRDD[VD]
  // Join operators利用内部索引加速连接

(显著

)
  def leftJoin[VD2, VD3](other: RDD[(VertexId, VD2)])(f: (VertexId, VD, Option[VD2])
  => VD3): VertexRDD[VD3]
  def innerJoin[U, VD2](other: RDD[(VertexId, U)])(f: (VertexId, VD, U) => VD2):
  VertexRDD[VD2]
  // 使用

RDD的索引加快输入数据执行

'reduceByKey'操作


  def aggregateUsingIndex[VD2](other: RDD[(VertexId, VD2)], reduceFunc: (VD2, VD2)
  => VD2): VertexRDD[VD2]
}

```
例如，filter操作符返回一个VertexRDD。filter是通过实现BitSet，从而复用索引和保持能快速与其他VertexRDD实现连接功能。类似的是，mapValues操作不允许map函数改变VertexID，从而可以复用同一HashMap中的数据结构。当leftJoin或innerJoin连接时，VertexRDD派生自同一hashmap，并且是通过线性扫描而非代价昂贵的逐点查询。

aggregateUsingIndex操作是一种新的有效的从RDD[（VertexID，A）]构建新的VertexRDD的方式。从概念上讲，如果在一组顶点上构建了一个VertexRDD[B]，这是一个在某些顶点RDD[（VertexID，A）]的超集，可以复用该超集为RDD[（VertexID，A）]建立索引。例如：
```
val setA: VertexRDD[Int] = VertexRDD(sc.parallelize(0L until 100L).map(id => (id, 1)))
val rddB: RDD[(VertexId, Double)] = sc.parallelize(0L until 100L).flatMap(id => List((id, 1.0), (id, 2.0)))
// 应该有

200个

rddB条目


rddB.count
val setB: VertexRDD[Double] = setA.aggregateUsingIndex(rddB, _ + _)
// 应该有

100个

rddB条目


setB.count
// A join B 应该是最快的


val setC: VertexRDD[Double] = setA.innerJoin(setB)((id, a, b) => a + b)
```

### EdgeRDD
该EdgeRDD[ED，VD]继承自RDD[Edge[ED]，以各种分区策略（PartitionStrategy）将边划分成不同的块。在每个分区中，边属性和邻接结构分别存储，这使得更改属性值时能够实现最大限度的复用。

EdgeRDD是提供的其余3个函数：
```
// 改变属性

,保留邻接结构


def mapValues[ED2](f: Edge[ED] => ED2): EdgeRDD[ED2, VD]
// 复用属性和结构进行

reverse
def reverse: EdgeRDD[ED, VD]
// 加入两个

EdgeRDD的分区使用相同的分区策略


def innerJoin[ED2, ED3](other: EdgeRDD[ED2, VD])(f: (VertexId, VertexId, ED, ED2) => ED3): EdgeRDD[ED3, VD]
```
在大多数应用中，我们发现，在EdgeRDD中的操作是通过图形运算符实现的，或依靠在基类定义的RDD类操作。

## GraphX的图操作
以下列出了Graph图和GraphOps中同时定义的操作。为了简单起见，定义为Graph的成员函数。
```
/** 总结属性图的方法

 */
class Graph[VD, ED] {
  // 图的属性


  val numEdges: Long
  val numVertices: Long
  val inDegrees: VertexRDD[Int]
  val outDegrees: VertexRDD[Int]
  val degrees: VertexRDD[Int]
  // 图作为结合的视图


  val vertices: VertexRDD[VD]
  val edges: EdgeRDD[ED, VD]
  val triplets: RDD[EdgeTriplet[VD, ED]]
  // 图缓存


  def persist(newLevel: StorageLevel = StorageLevel.MEMORY_ONLY): Graph[VD, ED]
  def cache(): Graph[VD, ED]
  def unpersistVertices(blocking: Boolean = true): Graph[VD, ED]
  // Change the partitioning heuristic
  def partitionBy(partitionStrategy: PartitionStrategy): Graph[VD, ED]
  // 转换顶点和边的属性


  def mapVertices[VD2](map: (VertexID, VD) => VD2): Graph[VD2, ED]
  def mapEdges[ED2](map: Edge[ED] => ED2): Graph[VD, ED2]
  def mapEdges[ED2](map: (PartitionID, Iterator[Edge[ED]]) => Iterator[ED2]):
  Graph[VD, ED2]
  def mapTriplets[ED2](map: EdgeTriplet[VD, ED] => ED2): Graph[VD, ED2]
  def mapTriplets[ED2](map: (PartitionID, Iterator[EdgeTriplet[VD, ED]]) =>
  Iterator[ED2]): Graph[VD, ED2]
  // 修改图结构


  def reverse: Graph[VD, ED]
  def subgraph(
      epred: EdgeTriplet[VD,ED] => Boolean = (x => true),
      vpred: (VertexID, VD) => Boolean = ((v, d) => true)): Graph[VD, ED]
  def mask[VD2, ED2](other: Graph[VD2, ED2]): Graph[VD, ED]
  def groupEdges(merge: (ED, ED) => ED): Graph[VD, ED]
  // 图的

Join操作

 ===============================================================
  def joinVertices[U](table: RDD[(VertexID, U)])(mapFunc: (VertexID, VD, U) =>
  VD): Graph[VD, ED]
  def outerJoinVertices[U, VD2](other: RDD[(VertexID, U)])
      (mapFunc: (VertexID, VD, Option[U]) => VD2): Graph[VD2, ED]
  // 收集信息

 =================================================
  def collectNeighborIds(edgeDirection: EdgeDirection): VertexRDD[Array[VertexID]]
  def collectNeighbors(edgeDirection: EdgeDirection): VertexRDD[Array[(VertexID, VD)]]
  def mapReduceTriplets[A: ClassTag](
      mapFunc: EdgeTriplet[VD, ED] => Iterator[(VertexID, A)],
      reduceFunc: (A, A) => A,
      activeSetOpt: Option[(VertexRDD[_], EdgeDirection)] = None)
    : VertexRDD[A]
  // Iterative graph-parallel computation ======================================
  def pregel[A](initialMsg: A, maxIterations: Int, activeDirection: EdgeDirection)(
      vprog: (VertexID, VD, A) => VD,
      sendMsg: EdgeTriplet[VD, ED] => Iterator[(VertexID,A)],
      mergeMsg: (A, A) => A): Graph[VD, ED]
  // 基本图算法

 ==================================================================
  def pageRank(tol: Double, resetProb: Double = 0.15): Graph[Double, Double]
  def connectedComponents(): Graph[VertexID, ED]
  def triangleCount(): Graph[Int, ED]
  def stronglyConnectedComponents(numIter: Int): Graph[VertexID, ED]
```
构造图

构造图一般有两种方式，通过Graph object构造和通过Graph Builder构造。

最常用的方法是使用Graph object构造Graph，左边是属性图，也就是我们要生成的图，右边第一个是这个属性图的顶点信息，第二个是这个属性图的边信息。
```
// Assume the SparkContext has already been constructed
val sc: SparkContext
// 创建顶点

RDD
val users: RDD[(VertexId, (String, String))] =
                sc.parallelize(Array((3L, ("rxin", "student")),
                                    (7L, ("jgonzal", "postdoc")),
                (5L, ("franklin", "prof")),
                                    (2L, ("istoica", "prof"))))
// 创建边

RDD
val relationships: RDD[Edge[String]] =
                sc.parallelize(Array(Edge(3L, 7L, "collab"),
                                    Edge(5L, 3L, "advisor"),
                Edge(2L, 5L, "colleague"),
                                    Edge(5L, 7L, "pi")))
// 定义默认顶点的信息


val defaultUser = ("John Doe", "Missing")
// 初始化图


val graph = Graph(users, relationships, defaultUser)
```
首先，通过SparkContext的方法parallelize创建图的顶点RDD users和边RDD relationships，以及默认的顶点defaultUser，然后通过Graph的构造方法创建相应的图对象graph。

其次，GraphX提供多种从RDD或者硬盘中的节点和边中构建图。默认情况下，图的构造方法不会重新划分图的边，Graph Builder这种构造方法会重新划分图的边。相反，边会留在它们的默认分区（如原来的HDFS块）。Graph.groupEdges需要的图形进行重新分区，因为它假设相同的边将被放在同一个分区同一位置，所以必须在调用Graph.partitionBy之前调用groupEdges。
```
object GraphLoader {
  def edgeListFile(
      sc: SparkContext,
      path: String,
      canonicalOrientation: Boolean = false,
      minEdgePartitions: Int = 1)
    : Graph[Int, Int]
}
```
GraphLoader.edgeListFile提供了一种从磁盘上边的列表载入图的方式。在此解析了一个以下形式的邻接列表（源顶点ID，目的地顶点ID）对：
```
# This is a comment
2 1
4 1
1 2
```
从指定的边创建了一个图表，自动遍历提到的任何顶点。所有顶点和边的属性默认为1。canonicalOrientation参数允许重新定向边的正方向（srcId<dstId），这是必需的connected component算法。该minEdgePartitions参数指定边分区生成的最小数目；例如，HDFS文件具有多个块，那么就有多个边的分割，示例如下：
```
# FromNodeId ToNodeId
5    3
3    7
5    7
2    5
```
通过加载HDFS文件构造图：
```
package com.iflytek.graph.builder
import org.apache.spark._
import org.apache.spark.graphx._
import org.apache.spark.rdd.RDD
object BuilderGraph {
      def main(args: Array[String]): Unit = {
              val conf = new SparkConf().setAppName("Make RDD").setMaster ("spark:
              // *.*.*.*:7077")
              val sc = new SparkContext(conf)
              val graph = GraphLoader.edgeListFile(sc,"/user/datasource/builer_
              graph.txt")
              graph.vertices.take(10)
      }
}
```
生成随机图：
```
（

API 1.0.1）


GraphGenerators.logNormalGraph(sc, numVertices = 100).mapVertices( (id, _) => id.toDouble )
(API 1.1.0)
// 初始化一个随机图，节点的度符合对数正态分布

,边属性初始化为

1
val graph: Graph[Double, Int] =GraphGenerators.logNormalGraph(sc, 100, 100).mapVertices((id, _) => id.toDouble)
```

### 属性操作
Graph的基本成员，我们经常使用的是vertices、edges和triplets。
```
class Graph[VD, ED] {
  // 图的属性


  val numEdges: Long
  val numVertices: Long
  val inDegrees: VertexRDD[Int]
  val outDegrees: VertexRDD[Int]
  val degrees: VertexRDD[Int]
  // 图集合的视图


 val vertices: VertexRDD[VD]
  val edges: EdgeRDD[ED, VD]
  val triplets: RDD[EdgeTriplet[VD, ED]]
}
```
### 转换操作
和RDD的Map操作类似，属性图包含以下转换操作：
```
class Graph[VD, ED]｛


  def mapVertices[VD2](map: (VertexId, VD) => VD2): Graph[VD2, ED]
  def mapEdges[ED2](map: Edge[ED] => ED2): Graph[VD, ED2]
  def mapTriplets[ED2](map: EdgeTriplet[VD, ED] => ED2): Graph[VD, ED2]
}
```
每个运算产生一个新的图，这个图的顶点和边属性通过map方法修改。

在所有情况下图的结构不受影响，这是这些运算符的关键所在，它允许新图可以复用初始图的结构索引。
```
val newVertices = graph.vertices.map { case (id, attr) => (id, mapUdf(id, attr)) }
val newGraph = Graph(newVertices, graph.edges)
```
相反，使用mapVertices保存索引：
```
val newGraph = graph.mapVertices((id, attr) => mapUdf(id, attr))
```
这些操作经常用来初始化图的特定计算，或者去除不必要的属性。例如，给定一个将out degree作为顶点的属性图，初始化并作为PageRank：
```
val inputGraph: Graph[Int, String] =
graph.outerJoinVertices(graph.outDegrees)((vid, _, degOpt) => degOpt.getOrElse(0))
// 建立一个图表

,每个边都包含了最初

PageRank的参数和顶点


val outputGraph: Graph[Double, Double] =
  inputGraph.mapTriplets(triplet => 1.0 / triplet.srcAttr).mapVertices((id, _) => 1.0)
```
### 结构操作
当前GraphX只支持一组简单的常用结构化操作，我们希望将来增加更多的操作。以下是基本的结构运算符的列表。
```
class Graph[VD, ED] {
  def reverse: Graph[VD, ED]
  def subgraph(epred: EdgeTriplet[VD,ED] => Boolean,
               vpred: (VertexId, VD) => Boolean): Graph[VD, ED]
  def mask[VD2, ED2](other: Graph[VD2, ED2]): Graph[VD, ED]
  def groupEdges(merge: (ED, ED) => ED): Graph[VD,ED]
}
```
该reverse操作符返回一个新图，新图的边的方向都反转了，这是非常实用的。例如，想要计算逆向PageRank，因为反向操作不修改顶点或边属性或改变边的数目，它的实现不需要数据移动或复制。

该子图subgraph将顶点和边的预测作为参数，并返回一个图，它只包含满足顶点条件的顶点图（值为true），以及满足边条件并连接顶点的边。subgraph子运算符可应用于很多场景，以限制图表的顶点和边，或消除断开的链接。

例如，在下面的代码中，删除已损坏的链接：
```
// 为顶点创建

RDD
val users: RDD[(VertexId, (String, String))] =
      sc.parallelize(Array((3L, ("rxin", "student")),
                               (7L, ("jgonzal", "postdoc")),
              (5L, ("franklin", "prof")),
                               (2L, ("istoica", "p3rof")),
              (4L, ("peter", "student"))))
// 为边创建

RDD
val relationships: RDD[Edge[String]] =
  sc.parallelize(Array(Edge(3L, 7L, "collab"),
                       Edge(5L, 3L, "advisor"),
                       Edge(2L, 5L, "colleague"), Edge(5L, 7L, "pi"),
                       Edge(4L, 0L, "student"),   Edge(5L, 0L, "colleague")))
// 定义一个默认的顶点

,以防丢失数据


val defaultUser = ("John Doe", "Missing")
// 初始化图

,Edge(4L, 0L, "student"),   Edge(5L, 0L, "colleague")构建图时已经删除


val graph = Graph(users, relationships, defaultUser)
// 请注意

,有一个顶点

0(我们没有信息

)连接到该顶点


graph.triplets.map(
    triplet => triplet.srcAttr._1 + " is the " + triplet.attr + " of " + triplet.dstAttr._1
  ).collect.foreach(println(_))
// 删除多余的顶点和与之相连的边


val validGraph = graph.subgraph(vpred = (id, attr) => attr._2 != "Missing")
// 这个子图将删除顶点

4和

5与顶点

0相连的边


validGraph.vertices.collect.foreach(println(_))
validGraph.triplets.map(
    triplet => triplet.srcAttr._1 + " is the " + triplet.attr + " of " + triplet.dstAttr._1
  ).collect.foreach(println(_))
```
我们举一个例子：
```
// 即去掉了

ID为

3的顶点


val validGraph2 = graph.subgraph(vpred = (id, attr) => attr._2 != "student")
validGraph2.vertices.collect.foreach(println(_))
validGraph2.triplets.map(
    triplet => triplet.srcAttr._1 + " is the " + triplet.attr + " of " + triplet.
    dstAttr._1
  ).collect.foreach(println(_))
```
在上面的例子中，仅提供了顶点条件。如果不提供顶点或边的条件，在subgraph操作中默认为真。mask操作返回一个包含输入图中所有顶点和边的图。这可以用来和subgraph一起使用，以限制基于属性的另一个相关图。例如，我们用去掉顶点的图运行联通分量，并且限制输出为合法的子图。
```
// 运行关联组件


// 计算每个顶点的连接组件成员并返回一个图的顶点值


val ccGraph = graph.connectedComponents()
// 删除丢失的顶点和边


val validGraph = graph.subgraph(vpred = (id, attr) => attr._2 != "Missing")
// 重新构造子图


val validCCGraph = ccGraph.mask(validGraph)
```
该groupEdges操作合并多重图中的平行边（即重复顶点对之间的边）。在许多数值计算的应用中，平行边可以加入为单条边，从而降低了图形的大小。

### 关联操作
在许多情况下，有必要从外部集合（RDDS）中加入图形数据。例如，我们可能有额外的用户属性，想要与现有的图形合并，或者可能需要从一个图选取一些顶点属性到另一个图。这些任务都可以使用join操作完成。下面列出了关联操作方法：
```
class Graph[VD, ED] {
  def joinVertices[U](table: RDD[(VertexId, U)])(map: (VertexId, VD, U) => VD)
    : Graph[VD, ED]
  def outerJoinVertices[U, VD2](table: RDD[(VertexId, U)])(map: (VertexId, VD,
  Option[U]) => VD2)
    : Graph[VD2, ED]
}

```
该joinVertices运算符连接输入RDD的顶点，并返回一个新图，新图的顶点属性通过用户自定义的map功能作用在被连接的顶点上。没有匹配的RDD保留其原始值。
```
val graph = GraphLoader.edgeListFile(sc,"/user/zfwu/datasource/web-Google.txt")
graph.vertices.take(10)
val rawGraph = graph.mapVertices((id,attr) => 0)
val outDegrees = rawGraph.outDegrees
val tmp = rawGraph.joinVertices(outDegrees)( (_,_,optDeg) => optDeg )
tmp.vertices.take(10)
```
如果RDD顶点包含多于一个的值，其中只有一个将会被使用。因此，建议在输入RDD初始为唯一时，使用下面pre-index所得到的值以加快后续join。
```
val nonUniqueCosts: RDD[(VertexID, Double)]
val uniqueCosts: VertexRDD[Double] =
  graph.vertices.aggregateUsingIndex(nonUnique, (a,b) => a + b)
val joinedGraph = graph.joinVertices(uniqueCosts)(
  (id, oldCost, extraCost) => oldCost + extraCost)
```
outerJoinVertices操作类似joinVertices，并且可以将用户定义的map函数应用到所有的顶点，同时改变顶点的属性类型。因为不是所有的顶点都可以匹配输入的RDD值，所以map函数需要对类型进行选择。例如，我们可以通过用outDegree初始化顶点属性来设置一个图的PageRank。
```
val outDegrees: VertexRDD[Int] = graph.outDegrees
val degreeGraph = graph.outerJoinVertices(outDegrees) { (id, oldAttr, outDegOpt) =>
  outDegOpt match {
    case Some(outDeg) => outDeg
    case None => 0 // No outDegree means zero outDegree
  }
}
```
在上面的例子中采用了多个参数列表的curried函数模式（例如，f（a）（b））。虽然我们可以同样写f（a）（b）为f（a，b），这将意味着该类型推断b不依赖于a。其结果是，用户将需要提供类型标注给用户自定义的函数：
```
val joinedGraph = graph.joinVertices(uniqueCosts,
  (id: VertexID, oldCost: Double, extraCost: Double) => oldCost + extraCost)
```

### 聚合操作
GraphX中核心的聚合操作是mapReduceTriplets操作：
```
class Graph[VD, ED] {
  def mapReduceTriplets[A](
      map: EdgeTriplet[VD, ED] => Iterator[(VertexId, A)],
      reduce: (A, A) => A): VertexRDD[A]
```
该mapReduceTriplets运算符将用户定义的map函数作为输入，并且将map作用到每个triplet，并可以得到triplet上所有顶点的信息。为了便于优化预聚合，支持发往triplet的源或目标顶点信息。用户定义的reduce功能将合并所有目标顶点相同的信息。该mapReduceTriplets操作返回VertexRDD[A]，包含所有以顶点作为目标节点的集合消息（类型A）。没有收到消息的顶点不包含在返回VertexRDD中。

需要注意的是，mapReduceTriplets需要一个附加的ActiveSet，限制了VertexRDD地图提供的邻接边map阶段：
```
activeSetOpt: Option[(VertexRDD[_], EdgeDirection)] = None
```
该EdgeDirection指定了哪些和顶点相邻的边包含在map阶段。如果该方向是in，则用户定义的map函数将仅仅作用目标顶点的ActiveSet中。如果方向是out，则该map函数将作用在源顶点的ActiveSet中。如果方向是either，则map函数将作用在源顶点或目标顶点的ActiveSet中。如果方向是both，则map函数将作用在源顶点和目标顶点的ActiveSet中。ActiveSet必须来自图的顶点。

在下面的例子中我们使用mapReduceTriplets算子来计算平均年龄。
```
// 导入随机图生成库


import org.apache.spark.graphx.util.GraphGenerators
// 创建一个图

vertex属性

age
val graph: Graph[Double, Int] =
      GraphGenerators.logNormalGraph(sc, 100, 100)
                        .mapVertices((id, _) => id.toDouble)
// 计算

older followers的人数和他们的总年龄


val olderFollowers: VertexRDD[(Int, Double)] = graph.mapReduceTriplets[(Int, Double)](
  triplet => { // Map Function
    if (triplet.srcAttr > triplet.dstAttr) {
      // 发送信息到目标

vertex
      Iterator((triplet.dstId, (1, triplet.srcAttr)))
    } else {
      // 不发送信息


      Iterator.empty
    }
  },
  // 增加计数和年龄


  (a, b) => (a._1 + b._1, a._2 + b._2) // Reduce Function
)
// 获取平均年龄


val avgAgeOfOlderFollowers: VertexRDD[Double] =
  olderFollowers.mapValues( (id, value) => value match { case (count, totalAge)
  => totalAge / count } )
// 显示结果


avgAgeOfOlderFollowers.collect.foreach(println(_))
```
当消息是固定长度时，mapReduceTriplets操作执行。

### 缓存操作
```
class Graph[VD, ED] {
def persist(newLevel: StorageLevel = StorageLevel.MEMORY_ONLY): Graph[VD, ED]
def cache(): Graph[VD, ED]
def unpersistVertices(blocking: Boolean = true): Graph[VD, ED]
}
```
在Spark中，RDD默认并不保存在内存中。为了避免重复计算，当需要多次使用时，建议使用缓存。在GraphX中Graphs行为方式相同。当需要多次使用图形时，一定要首先调用Graph.cache（）。

在迭代计算中，为了获得最佳的性能，也可能需要清空缓存。在默认情况下，缓存的RDD和图表将保留在内存中，直到按照LRU顺序被删除。对于迭代计算，之前迭代的中间结果将填补缓存。虽然缓存最终将被删除，但是内存中不必要的数据还是会使垃圾回收机制变慢。有效策略是，一旦缓存不再需要，应用程序立即清空中间结果的缓存，还可以通过持久化图形，这样在下次迭代中可以有接使用。