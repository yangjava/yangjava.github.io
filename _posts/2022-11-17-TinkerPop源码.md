---
layout: post
categories: [TinkerPop]
description: none
keywords: TinkerPop
---
# TinkerPop源码

## gremlin-core源码分析
该模块无法直接运行，需要借助tinkergraph-gremlin模块。tinkergraph-gremlin模块为TinkerGraph内存数据库的实现，在tinkergraph-gremlin模块中创建一个测试类Main，然后在其主函数中运行如下代码。
```xml
        <dependency>
            <groupId>org.apache.tinkerpop</groupId>
            <artifactId>tinkergraph-gremlin</artifactId>
            <version>3.6.2</version>
        </dependency>
```
简单的代码
```java
import org.apache.tinkerpop.gremlin.process.traversal.dsl.graph.GraphTraversal;
import org.apache.tinkerpop.gremlin.process.traversal.dsl.graph.GraphTraversalSource;
import org.apache.tinkerpop.gremlin.structure.Vertex;
import org.apache.tinkerpop.gremlin.tinkergraph.structure.TinkerGraph;

public class Main {

    public static void main(String[] args) throws Exception {
        TinkerGraph graph = TinkerGraph.open();
        Vertex marko = graph.addVertex("name", "marko", "age", 29);
        Vertex lop = graph.addVertex("name", "lop", "lang", "java", "height", 173);
        Vertex jay = graph.addVertex("name", "jay", "lang", "python", "height", 160);
        marko.addEdge("created", lop, "weight", 0.6d);
        marko.addEdge("created", jay, "weight", 0.7d);
        GraphTraversalSource g = graph.traversal();

        GraphTraversal allV = g.V();
        GraphTraversal hasName = allV.has("name", "marko");
        GraphTraversal outCreated = hasName.out("created");
        GraphTraversal valueName = outCreated.values("height");
        Object next = valueName.next();
        System.out.println(next);
    }

}
```
- 创建了一个TinkerGraph类型的空对象graph
- 为这个graph对象创建了三个顶点，以及在marko顶点上创建了两个边，每个顶点和边均有各自的属性值。
- 创建了一个GraphTraversalSource类型的对象g，该对象用于图遍历。
```
Object next = g.V().has("name", "marko").out("created").values("height");
```
图数据库不进行实质的查询操作，而是将要操作的指令进行存储，等调用next()函数时才进行真正的查询操作。

## TinkerPop
图查询语言 Gremlin 是 TinkerPop 框架为 图服务用户 ( User ) 提供的数据操作接口，而对于 图服务提供商 ( Provider ) 来说，需要了解 TinkerPop 框架的属性图模型与接口。

TinkerPop 将其接口粗略划分为 Structure 和 Process 两部分。Structure 部分定义了图的拓扑结构与功能，包括 Vertex、Edge、Property 等，由 图服务提供商 ( Provider ) 实现其接口，以填充数据；Process 部分定义了遍历图数据的 DSL ，并提供调用 Structure 接口的默认实现。

## Structure 接口结构与实现思路
作为刚接触 TinkerPop 没多久的 Provider，只需实现 Structure 接口即可完成 Gremlin 图查询功能的接入。
这也意味着没有图计算与事务功能，仅支持 Gremlin 查询。
TinkerPop Provider 文档中将 Provider 细分为：
- Graph System Provider
- Graph Database Provider
- Graph Processor Provider
- Graph Driver Provider 
- Graph Language Provider
- Graph Plugin Provider
本文中的 Provider 仅代表 Graph Database Provider 。
同样地，Structure 接口仅包括 Gremlin-Core 模块 ( 源码 structure/ 文件夹下 ) 的下述接口：

## Graph 接口
Graph 接口在整个 Structure 体系中具有核心地位，是整个图服务的入口与出口。
TinkerPop 的 Gremlin 执行过程依靠 Process 体系里的各种类 ( TraversalSource, Strategy, Traversal, Step, Traverser, etc. ) 实现，最终调用 Graph 接口，从存储层输入输出对应数据。

该接口作为图遍历的入口，在 Gremlin 脚本执行过程中位于起始位置，从存储层获取数据后，交给后续操作执行进一步处理。
按照惯例，用户将自己的 Graph 实现类命名为 XXXGraph，如官方样例提供的 TinkerGraph、Neo4jGraph 等。
```
public final class TinkerGraph implements Graph {...}

public final class Neo4jGraph implements Graph, WrappedGraph<Neo4jGraphAPI> {...}
```

又如 JanusGraph 源码中的 Graph 继承体系：
```
// StandardJanusGraph 继承 JanusGraphBlueprintsGraph
public class StandardJanusGraph extends JanusGraphBlueprintsGraph {...}

// JanusGraphBlueprintsGraph 实现 JanusGraph 接口
public abstract class JanusGraphBlueprintsGraph implements JanusGraph {...}

// JanusGraph 接口继承 Transaction 接口
public interface JanusGraph extends Transaction {...}

// Transaction 接口继承 Tinkerpop Graph 接口
public interface Transaction extends Graph, SchemaManager {...}
```
Tinkerpop Graph 接口提供了许多带有默认实现的方法，仅留下如下几个方法需要 Provider 自行实现：
- 添加节点方法 addVertex
Vertex addVertex(final Object... keyValues);
该方法对应着 g.addV() 与 graph.addVertex() 的调用方式，即为 Gremlin 语言 addV 功能提供支持。
实现思路为将传入的参数处理为自己的 Vertex 接口实现类的对象，将该对象持久化到存储层，并返回该对象。

- 获取节点方法 vertices
Iterator<Vertex> vertices(final Object... vertexIds);
该方法支撑 Gremlin 语言 g.V() 调用。
实现思路为根据传入的节点 ID 参数，到存储层查询节点数据，最终根据所查到的节点生成一个 Vertex 接口迭代器，并将其作为返回值。

- 获取边方法 edges
Iterator<Edge> edges(final Object... edgeIds);
该方法支撑 Gremlin 语言 g.E() 调用。
实现思路类似 vertices 方法。

- 图退出方法 close
void close() throws Exception;
该方法提供了图服务退出时，保存持久化层与关闭事务等工作的调用勾子。

- 读取用户配置方法 variables 与 configuration
Variables variables();
Configuration configuration();
这俩方法可以随意应付，不影响支撑 Gremlin 语言功能。

- 启动事务方法 tx
Transaction tx();
可在实现中直接抛出异常，表明该图数据库不支持事务功能，不影响支持 Gremlin 语言功能。

- 启动图计算方法 compute
GraphComputer compute() throws IllegalArgumentException;
实现思路类似事务。

## Element 接口
属性图模型的基础类型接口，Vertex、Edge、VertexProperty 均继承该接口，表示图中的元素。
命名惯例同 Graph 接口，实现类为 XXXElement。
该接口声明了图元素共有的属性与方法，其中部分方法具有默认实现，需 Provider 自行实现的方法包括：

- ID
Object id();
图元素唯一标识符 id 的 Getter 方法。
此处 id 类型为 Object，但在具体实现时又会根据实际需要将 id 类型限制为数字或字符串，又或是不限制类型。

- Label
String label();
图元素标签 label 的 Getter 方法。
标签即为元素类型，在数据库中被称为 Meta，在知识图谱中被称为本体/概念。
TinkerPop 属性图模型似乎仅支持单标签，而 Neo4j 属性图模型可支持多标签，这点在 cypher-to-gremlin 项目的解析器中有所体现。

- Graph
Graph graph();
图元素所属的图实例的 Getter 方法。
此处暗示了 Element 实现类仅为内存中的对象，即数据库中的外模式，并非持久层对象。因此，要在 Graph 接口实现类的 vertices()、edges()、addVertex() 中，为相应的图元素设置 graph 属性，供后续遍历方法调用。
该方法支撑了其他关联查询接口，比如从已有图元素出发，继续查询相关联的节点或边的方法。
g.V().has('person', 'name', 'marko').out("knows")
在该 Gremlin 语句中 g.V().has() 从 Graph 接口中获得了起始节点，接着使用 out() 方法请求该节点的出边关联节点，而 out() 方法的实现中调用了该 graph() 方法。
此外，还需注意，对于一个图数据库来说，应该支持多个图实例管理，这意味着同一个 Gremlin 语句，目标 graph 不同，得到的结果也不同，该方法返回的对象也是不同的。

- 添加属性方法 property
<V> Property<V> property(final String key, final V value);
将属性的 Key，Value 传递给当前图元素。
实现思路为根据传入的参数构建自己的 Property 对象，接着对图元素的内模式做相应修改并持久化，最后返回该 Property 对象。

- 获取属性方法 properties
<V> Iterator<? extends Property<V>> properties(final String... propertyKeys);
根据传入的属性 Key 列表，从当前图元素的属性 Map 中过滤出相应的 Property 列表。

- 元素删除方法 remove
void remove();
可在实现中抛出异常，表示当前图数据库不支持删除数据。
或者直接返回，假装完成了删除操作。

## Vertex 接口
图模型中的节点元素，该接口继承 Element 接口，表示图数据库的节点外模式。
命名惯例同 Graph 接口，实现类为 XXXVertex，该类需继承上述 XXXElement 类。
在接口设计上，TinkerPop 要求每个 Vertex 实例可以自身为起点，找到相关联的入边 ( Incoming Edges ) 和出边 ( Outgoing Edges)，以及连边的另一端 Vertex 实例。
此设计天然适用于 无索引近邻 式的图处理结构。因此在实现 Vertex 接口时，可考虑在存储层之上构建该处理结构，用以加速查询。
该接口声明了一些需要 Provider 实现的重要方法，这些方法在 Gremlin 查询过程中起到了基石的作用。

- 添加出边 addEdge
Edge addEdge(final String label, final Vertex inVertex, final Object... keyValues);
实现思路为根据传入的边标签 label 和边属性 keyValues 构建 Edge 实例，并将其持久化。
如果实现了无索引近邻结构，需进一步更新与该边相关联的两点的索引内容。
如果实现了属性索引，还需为相应属性值与该 Edge 构建索引。

- 添加属性 property
<V> VertexProperty<V> property(final VertexProperty.Cardinality cardinality, final String key, final V value, final Object... keyValues);
参数 cardinality 表示属性基数，包括 single、list、set。
参数 key、value 无需多言。
此处需要理解 TinkerPop 中 VertexProperty 与 Property 的差异。
TinkerPop 将这种带有属性的属性表示为 VertexProperty，归属于 Element。而仅有属性值的属性表示为 Property。VertexProperty 的属性也是 Property 对象。
参数 keyValues 表示了该属性的属性，举个例子：张三 ( Vertex ) 的学历 ( Key ) 有小学、初中、高中 ( Value )，而每个学历值都有入学时间、毕业时间、学校名称等属性 ( keyValues ) 。
奇怪的是，TinkerPop 将 Edge 的属性表示为 Property，Property 对象没有下一级属性，这点可在 addEdge 方法与 Edge 接口中体会到。同时，与 Edge 有关的 Gremlin 处理步骤均无法设置属性的属性。
如此区分 Vertex 与 Edge 的属性，总让人觉得缺少对称的美感，也不兼容实际建模的需求。如果想要修改此行为，又将不可避免地入侵 TinkerPop 设计中未暴露接口的部分。若把属性的属性用 Map 存储或序列化为字符串作为 Edge 的属性，似乎也有不少问题，至少在标准 Gremlin 语法上无法查询 Edge 的属性的属性。

- 获取相邻边 edges
Iterator<Edge> edges(final Direction direction, final String... edgeLabels);
实现思路：将参数中的方向和边标签作为过滤条件，从无索引近邻结构或存储层中查询相关边。

- 获取相邻节点 vertices
Iterator<Vertex> vertices(final Direction direction, final String... edgeLabels);
实现思路类似 edges 方法。

- 获取节点属性 properties
<V> Iterator<VertexProperty<V>> properties(final String... propertyKeys);
实现思路为从节点的详细信息中获取属性列表，然后根据参数 propertyKeys 过滤出对应属性值。

## Edge 接口
图模型中的边元素，该接口继承 Element 接口，表示图数据库的边外模式。
命名惯例同 Graph 接口，实现类为 XXXEdge，该类需继承上述 XXXElement 类。
在接口设计上，TinkerPop 要求每个 Edge 实例可以自身为起点，找到相关联的起始节点和终止节点。
该接口提供了一些方法的默认实现，仅需 Provider 提供以下两个方法的具体实现。

- 获取相关节点 vertices
Iterator<Vertex> vertices(final Direction direction);
实现思路为根据当前 Edge 的信息以及传入的方向参数，从索引中获存储层查询相关联节点。

- 获取相关属性 properties
<V> Iterator<Property<V>> properties(final String... propertyKeys);
实现思路类似获取节点属性方法。

## Property 接口
图模型中的属性元素，表示图数据库的属性外模式。
命名惯例同 Graph 接口，实现类为 XXXProperty 。

A Property denotes a key/value pair associated with an Edge.

如上文讨论的那样，TinkerPop 在其文档中明确写道：属性是与边相关的 K/V 对，Key 只能是 String 类型，Value 只能是 Java 类型。
该接口提供了一些方法的默认实现，仅需 Provider 提供以下方法的具体实现。

- 获取属性键 key
String key();
该方法被调用时，往往已经获取了 Provider 实现的 XXXProperty 对象，只需将该对象 key 值返回即可。

- 获取属性值 value
V value() throws NoSuchElementException;
实现思路同上。

- 获取关联对象 element
Element element();
实现思路同上。

- 判断属性值是否存在
boolean isPresent();
当前 XXXProperty 的 value 属性非空时返回 true，否则返回 false。

- 删除当前属性 remove
void remove();
可在实现中抛出异常，表示当前图数据库不支持删除数据。
或者直接返回，假装完成了删除操作。

## VertexProperty 接口
图模型中的可携带属性的节点属性元素，表示图数据库的属性外模式。
命名惯例同 Graph 接口，实现类为 XXXVertexProperty，该类需继承 XXXElement 类。

A VertexProperty is similar to a Property in that it denotes a key/value pair associated with an Vertex, however it is different in the sense that it also represents an entity that it is an Element that can have properties of its own.

如上文讨论的那样，TinkerPop 在其文档中明确写道：节点属性是与节点相关的 K/V 对，同时，节点属性也是图元素的一种，可以携带自己的属性。
VertexProperty 接口提供了一些方法的默认实现，但由于该接口继承了 Property 接口，因此需要 Provider 提供上述 Propery 接口方法和以下方法的具体实现。

- 获取所属节点 element
Vertex element();
该方法覆写了 Property 接口的 element 方法，返回当前节点属性所属的 Vertex 对象。

- 获取属性 properties
<U> Iterator<Property<U>> properties(final String... propertyKeys);
返回当前节点属性的属性。

## 源码解析实战
tinkerpop 源码是JanusGraph 源码解析的第一步，我们需要大概有个了解。
```java

import org.apache.tinkerpop.gremlin.process.traversal.dsl.graph.GraphTraversal;
import org.apache.tinkerpop.gremlin.process.traversal.dsl.graph.GraphTraversalSource;
import org.apache.tinkerpop.gremlin.structure.Vertex;
import org.apache.tinkerpop.gremlin.tinkergraph.structure.TinkerGraph;

import static org.apache.tinkerpop.gremlin.structure.VertexProperty.Cardinality.list;

public class TinkerGraphDemo {

    public static void main(String[] args) {
        TinkerGraph graph = TinkerGraph.open();
        GraphTraversalSource g = graph.traversal();

        Vertex v = g.addV().property("name","marko").property("nam","marko a. rodriguez").next();

        GraphTraversal<Vertex, Long> name = g.V(v).properties("name").count();
        v.property(list, "name", "m. a. rodriguez");
        g.V(v).properties("name").count();
        g.V(v).properties();
        g.V(v).properties("name");
        g.V(v).properties("name").hasValue("marko");
        g.V(v).properties("name").hasValue("marko").property("acl","private"); //
        g.V(v).properties("name").hasValue("marko a. rodriguez");
        g.V(v).properties("name").hasValue("marko a. rodriguez").property("acl","public");
        g.V(v).properties("name").has("acl","public").value();
        g.V(v).properties("name").has("acl","public").drop(); //4\
        g.V(v).properties("name").has("acl","public").value();
        g.V(v).properties("name").has("acl","private").value();
        g.V(v).properties();
        g.V(v).properties().properties(); //5\
        g.V(v).properties().property("date",2014) ;//6\
        g.V(v).properties().property("creator","stephen");
        g.V(v).properties().properties();
        g.V(v).properties("name").valueMap();
        g.V(v).property("name","okram"); //7\
        g.V(v).properties("name");
        g.V(v).values("name"); //8
    }
}
```
然后从第一行开始 打断点，debug。首先注意 TinkerGraph 是一个很简单的图数据库，超级简单。

### TinkerGraph
TinkerGraph.open() 方法会新建一个 TinkerGraph：
```java
   private TinkerGraph(final Configuration configuration) {
        this.configuration = configuration;
        vertexIdManager = selectIdManager(configuration, GREMLIN_TINKERGRAPH_VERTEX_ID_MANAGER, Vertex.class);
        edgeIdManager = selectIdManager(configuration, GREMLIN_TINKERGRAPH_EDGE_ID_MANAGER, Edge.class);
        vertexPropertyIdManager = selectIdManager(configuration, GREMLIN_TINKERGRAPH_VERTEX_PROPERTY_ID_MANAGER, VertexProperty.class);
        defaultVertexPropertyCardinality = VertexProperty.Cardinality.valueOf(
                configuration.getString(GREMLIN_TINKERGRAPH_DEFAULT_VERTEX_PROPERTY_CARDINALITY, VertexProperty.Cardinality.single.name()));

        graphLocation = configuration.getString(GREMLIN_TINKERGRAPH_GRAPH_LOCATION, null);
        graphFormat = configuration.getString(GREMLIN_TINKERGRAPH_GRAPH_FORMAT, null);

        if ((graphLocation != null && null == graphFormat) || (null == graphLocation && graphFormat != null))
            throw new IllegalStateException(String.format("The %s and %s must both be specified if either is present",
                    GREMLIN_TINKERGRAPH_GRAPH_LOCATION, GREMLIN_TINKERGRAPH_GRAPH_FORMAT));

        if (graphLocation != null) loadGraph();
    }

```
这里的 configuration 类似一个map。然后有几个 IDManager 和其他变量赋值。

然后是 GraphTraversalSource g = graph.traversal(); 这一句仅仅是 return new GraphTraversalSource(this);
```java
    public GraphTraversalSource(final Graph graph) {
        this(graph, TraversalStrategies.GlobalCache.getStrategies(graph.getClass()));
    }
```
这时候我们取看一下几个特殊的类： Traversal Traversal.Admin 。 他们两个都是接口，而且 Admin 是继承自 Traversal。 Traversal 代表遍历，主要方法包括 next， asNext，iterator toList 等，可以看成一个迭代器，而 Admin 的主要方法和他没有什么关系。

GraphTraversal extends Traversal , 新增加了很多和 gremlin 相关的方法。这属于 tinkerpop 的内容

然后是 addVertex
```java
public Vertex addVertex(final Object... keyValues) {
    ElementHelper.legalPropertyKeyValueArray(keyValues);
    Object idValue = vertexIdManager.convert(ElementHelper.getIdValue(keyValues).orElse(null));
    final String label = ElementHelper.getLabelValue(keyValues).orElse(Vertex.DEFAULT_LABEL);

    if (null != idValue) {
        if (this.vertices.containsKey(idValue))
            throw Exceptions.vertexWithIdAlreadyExists(idValue);
    } else {
        idValue = vertexIdManager.getNextId(this);
    }

    final Vertex vertex = new TinkerVertex(idValue, label, this);
    this.vertices.put(vertex.id(), vertex);

    ElementHelper.attachProperties(vertex, VertexProperty.Cardinality.list, keyValues);
    return vertex;
}
```
我们可以看出源码十分简单，而且 property 等方法就更简单了，所以 TinkerGraph 可以用来进行后续的分析。这样有关图操作部分就不会有理解难度。

### GraphTraversalSource
构造方法：
```
GraphTraversalSource(Graph graph) GraphTraversalSource(Graph graph, TraversalStrategies traversalStrategies)
```
graph.traversal() 方法返回一个 GraphTraversalSource ，我们挺好奇怎么转化为计算逻辑的，可能类似spark一样，记录操作，然后计算。但是不同的数据库怎么切换逻辑呢？

GraphTraversal 代表了遍历，在graph中的一条路径。

Traverser 代表 GraphTraversal 中的对象遍历过程的当前状态

### 我们使用自带的一个库进行调试：
```java

import org.apache.tinkerpop.gremlin.process.traversal.dsl.graph.GraphTraversalSource;
import org.apache.tinkerpop.gremlin.tinkergraph.structure.TinkerFactory;
import org.apache.tinkerpop.gremlin.tinkergraph.structure.TinkerGraph;

public class TinkerDemo2 {

    public static void main(String[] args) {
        TinkerGraph graph = TinkerFactory.createModern();

        GraphTraversalSource g = graph.traversal();
        Object next = g.V(1).outE("knows").inV().values("name").next();

        System.out.println(next.toString());
    }
}

```
然后我们的断点设置在 TinkerGraph 的增删改查方法上。

第一次是 org.apache.tinkerpop.gremlin.tinkergraph.structure.TinkerGraph.addVertex(TinkerGraph.java:161)，这个方法的内容比较简单，得到 id label ，新建 TinkerVertex。

然后是TinkerGraph.vertices,完整调用信息：
```
org.apache.tinkerpop.gremlin.tinkergraph.structure.TinkerGraph.vertices(TinkerGraph.java:245)
org.apache.tinkerpop.gremlin.tinkergraph.process.traversal.step.sideEffect.TinkerGraphStep.vertices(TinkerGraphStep.java:85)
org.apache.tinkerpop.gremlin.tinkergraph.process.traversal.step.sideEffect.TinkerGraphStep.lambda$new$0(TinkerGraphStep.java:59)
org.apache.tinkerpop.gremlin.tinkergraph.process.traversal.step.sideEffect.TinkerGraphStep$$Lambda$23.846254484.get(Unknown Source:-1)
org.apache.tinkerpop.gremlin.process.traversal.step.map.GraphStep.processNextStart(GraphStep.java:142)
org.apache.tinkerpop.gremlin.process.traversal.step.util.AbstractStep.hasNext(AbstractStep.java:143)
org.apache.tinkerpop.gremlin.process.traversal.step.util.ExpandableStepIterator.next(ExpandableStepIterator.java:50)
org.apache.tinkerpop.gremlin.process.traversal.step.map.FlatMapStep.processNextStart(FlatMapStep.java:48)
org.apache.tinkerpop.gremlin.process.traversal.step.util.AbstractStep.hasNext(AbstractStep.java:143)
org.apache.tinkerpop.gremlin.process.traversal.step.util.ExpandableStepIterator.next(ExpandableStepIterator.java:50)
org.apache.tinkerpop.gremlin.process.traversal.step.map.FlatMapStep.processNextStart(FlatMapStep.java:48)
org.apache.tinkerpop.gremlin.process.traversal.step.util.AbstractStep.next(AbstractStep.java:128)
org.apache.tinkerpop.gremlin.process.traversal.step.util.AbstractStep.next(AbstractStep.java:38)
org.apache.tinkerpop.gremlin.process.traversal.util.DefaultTraversal.next(DefaultTraversal.java:200)
我们的代码：Object next = g.V(1).outE("knows").inV().values("name").next();
```
一共出现了 TinkerGraphStep，GraphStep，AbstractStep，FlatMapStep 四类 Step，最后一个 TinkerGraphStep 是怎么出现的？按理说都是tinkerpop 的实现，其实是 TinkerGraphStepStrategy 的缘故。

接下来我们吧 Object next = g.V(1).outE("knows").inV().values("name").next(); 拆开分析：
```java
GraphTraversal<Vertex, Vertex> v = g.V(1);
GraphTraversal<Vertex, Edge> knows = v.outE("knows");
GraphTraversal<Vertex, Vertex> inV = knows.inV();
GraphTraversal<Vertex, Object> name = inV.values("name");
Object next = name.next();
```
每一步我们都断点，然后查看当前 GraphTraversal 的属性，状态等。
```
注册 traversalStrategies ：
        traversalStrategies = {ArrayList@1426}  size = 14
        0 = {ConnectiveStrategy@1428} "ConnectiveStrategy"
        1 = {IncidentToAdjacentStrategy@1429} "IncidentToAdjacentStrategy"
        2 = {MatchPredicateStrategy@1430} "MatchPredicateStrategy"
        3 = {FilterRankingStrategy@1431} "FilterRankingStrategy"
        4 = {InlineFilterStrategy@1432} "InlineFilterStrategy"
        5 = {AdjacentToIncidentStrategy@1433} "AdjacentToIncidentStrategy"
        6 = {RepeatUnrollStrategy@1434} "RepeatUnrollStrategy"
        7 = {PathRetractionStrategy@1435} "PathRetractionStrategy"
        8 = {CountStrategy@1436} "CountStrategy"
        9 = {LazyBarrierStrategy@1437} "LazyBarrierStrategy"
        10 = {TinkerGraphCountStrategy@1438} "TinkerGraphCountStrategy"
        11 = {TinkerGraphStepStrategy@1439} "TinkerGraphStepStrategy"
        12 = {ProfileStrategy@1440} "ProfileStrategy"
        13 = {StandardVerificationStrategy@1441} "StandardVerificationStrategy"
```

### g.V(id)
```java
public GraphTraversal<Vertex, Vertex> V(final Object... vertexIds) {
    final GraphTraversalSource clone = this.clone(); // 克隆 对象
    clone.bytecode.addStep(GraphTraversal.Symbols.V, vertexIds); // 记录
    final GraphTraversal.Admin<Vertex, Vertex> traversal = new DefaultGraphTraversal<>(clone);
    {// 这里逻辑很简单，直接赋值
        this.graph = graph;
        this.strategies = traversalStrategies;
        this.bytecode = bytecode;
    }
    return traversal.addStep(new GraphStep<>(traversal, Vertex.class, true, vertexIds));
    {
    // 1. new GraphStep<>(traversal, Vertex.class, true, vertexIds)
        {
        super(traversal);// AbstractStep
        // 这一步要注意，这里的 iteratorSupplier 使用了 getGraph。如果graph 是 JanusGraph，那么就会查询 JanusGraph 的相关方法。
        this.iteratorSupplier = () -> (Iterator<E>) (Vertex.class.isAssignableFrom(this.returnClass) ?
             this.getTraversal().getGraph().get().vertices(this.ids) :
             this.getTraversal().getGraph().get().edges(this.ids));
        }

    // 2. traversal.addStep
    {
    return (GraphTraversal.Admin<S, E2>) Traversal.Admin.super.addStep((Step) step);
    {
        public <S2, E2> Traversal.Admin<S2, E2> addStep(final int index, final Step<?, ?> step) throws IllegalStateException {
            if (this.locked) throw Exceptions.traversalIsLocked();
            step.setId(this.stepPosition.nextXId());
            this.steps.add(index, step); // 添加到集合
            final Step previousStep = this.steps.size() > 0 && index != 0 ? steps.get(index - 1) : null;
            final Step nextStep = this.steps.size() > index + 1 ? steps.get(index + 1) : null;
            // 设置前后依赖
            step.setPreviousStep(null != previousStep ? previousStep : EmptyStep.instance());
            step.setNextStep(null != nextStep ? nextStep : EmptyStep.instance());
            if (null != previousStep) previousStep.setNextStep(step);
            if (null != nextStep) nextStep.setPreviousStep(step);
            step.setTraversal(this);
            return (Traversal.Admin<S2, E2>) this;
        }
    }
    }
    }
}
```

### v.outE("knows");
```java
public default GraphTraversal<S, Edge> outE(final String... edgeLabels) {
    this.asAdmin().getBytecode().addStep(Symbols.outE, edgeLabels); // 记录
    return this.asAdmin().addStep(new VertexStep<>(this.asAdmin(), Edge.class, Direction.OUT, edgeLabels));
    {
    // 1. new VertexStep<>(this.asAdmin(), Edge.class, Direction.OUT, edgeLabels)
        {
        super(traversal);// FlatMapStep -> AbstractStep
        this.direction = direction;
        this.edgeLabels = edgeLabels;
        this.returnClass = returnClass;
        }
    // 2. addStep
    // 和上面一样
    }
}
```

### knows.inV()
```java
public default GraphTraversal<S, Vertex> inV() {
    this.asAdmin().getBytecode().addStep(Symbols.inV); 
    return this.asAdmin().addStep(new EdgeVertexStep(this.asAdmin(), Direction.IN));
    {
    // 1. new EdgeVertexStep(this.asAdmin(), Direction.IN)
    {
        public EdgeVertexStep(final Traversal.Admin traversal, final Direction direction) {
        super(traversal);
        this.direction = direction;
        }
    }
    // 2. 和上面一样
    }
}
```

### inV.values("name");
```java
public default <E2> GraphTraversal<S, E2> values(final String... propertyKeys) {
    this.asAdmin().getBytecode().addStep(Symbols.values, propertyKeys);
    return this.asAdmin().addStep(new PropertiesStep<>(this.asAdmin(), PropertyType.VALUE, propertyKeys));
    {
    // 1. new PropertiesStep<>(this.asAdmin(), PropertyType.VALUE, propertyKeys)
    {
        public PropertiesStep(final Traversal.Admin traversal, final PropertyType propertyType, final String... propertyKeys) {
        super(traversal);
        this.returnType = propertyType;
        this.propertyKeys = propertyKeys;
        }
    }
    }
}
```

### name.next();
next 方法类似spark中的 action，
```java
public E next() {
    try {
        if (!this.locked) this.applyStrategies();
        if (this.lastTraverser.bulk() == 0L) // 这个判断的意义是什么。
            this.lastTraverser = this.finalEndStep.next();
        this.lastTraverser.setBulk(this.lastTraverser.bulk() - 1L);
        return this.lastTraverser.get();
    } catch (final FastNoSuchElementException e) {
        throw this.parent instanceof EmptyStep ? new NoSuchElementException() : e;
    }
}
```
可以看出这里调用了 this.finalEndStep.next()，有点像spark，将逻辑转到 Step 中。

这个 next 方法触发了查询等操作，所以逻辑会比其他的复杂，我们可以看看几个关键步骤：
```java
this.applyStrategies();
this.lastTraverser = this.finalEndStep.next();
this.lastTraverser.setBulk(this.lastTraverser.bulk() - 1L);
```
涉及到几个属性：
```java
private Traverser.Admin<E> lastTraverser = EmptyTraverser.instance();
private Step<?, E> finalEndStep = EmptyStep.instance();  
protected List<Step> steps = new ArrayList<>();
```
在 applyStrategies 方法中有 this.finalEndStep = this.getEndStep(); 我们可以知道一个 Traversal 有很多steps，前后都有依赖关系，有一个 finalEndStep。 然后是 this.lastTraverser = this.finalEndStep.next(); 可以看出 step 的 next 方法很重要，返回一个 Traverser ,Traverser 能干嘛我们现在还不知道。 但是我们现在看出 Traverser 有 bulk 属性，如果 bulk >0 证明当前已经执行过了，每次调用 hasNext 或者 next 的时候就直接返回数据，否则需要查询。 而查询的逻辑很明显就在 this.lastTraverser = this.finalEndStep.next();
```java
AbstractStep.next()
```
Step 继承自Iterator，Step.next() 返回一个 Traverser，中间涉及了查询的操作。next 实现 在 AbstractStep 中，还有两个类实现了重写。重写也会调用父类。
```java
public Traverser.Admin<E> next() {
    if (null != this.nextEnd) { // nextEnd 应该是缓存，如果不为空直接返回，否则查询。
        try {
            return this.prepareTraversalForNextStep(this.nextEnd);
        } finally {
            this.nextEnd = null;
        }
    } else {
        while (true) {
            if (Thread.interrupted()) throw new TraversalInterruptedException();
            final Traverser.Admin<E> traverser = this.processNextStart();
            if (null != traverser.get() && 0 != traverser.bulk())
                return this.prepareTraversalForNextStep(traverser);
        }
    }
}
```
然后调用 processNextStart()，这就是真正的逻辑发生的地方。processNextStart 在 AbstractStep 中是一个抽象方法，需要底层不同的实现。就拿我们这里的几个例子来说：
```
0 = {GraphStep@1408} "GraphStep(vertex,[1])"    GraphStep 是最原始的step，它的next 方法是从数据库拿数据。所以 会保存一个 iterator，还有个 iteratorSupplier。
1 = {VertexStep@1409} "VertexStep(OUT,[knows],edge)" 继承自 FlatMapStep，内部保存了一个 ExpandableStepIterator starts 实现 processNextStart 方法，
2 = {EdgeVertexStep@1410} "EdgeVertexStep(IN)"   和 VertexStep 一样，只是实现了自己的 flatMap 方法。
3 = {PropertiesStep@1411} "PropertiesStep([name],value)" 同上。
```
看到这里我们稍微清楚了一点，还是有很多问题， FlatMapStep 的 processNextStart 逻辑是在干啥我们并没有理清楚。

AbstractStep 有一个 protected ExpandableStepIterator<S> starts = new ExpandableStepIterator<>(this); 它的 next 方法如下：
```java
public Traverser.Admin<S> next() {
    if (!this.traverserSet.isEmpty())
        return this.traverserSet.remove();
    /////////////
    if (this.hostStep.getPreviousStep().hasNext())
        return this.hostStep.getPreviousStep().next();
    /////////////
    return this.traverserSet.remove();
}
```
我们可以看出，其实并没有什么逻辑，就是 return this.hostStep.getPreviousStep().next()，而前面会加一个 traverserSet 判断，而且 traverserSet 是空的，需要显式调用 add 方法才能添加，所以 ExpandableStepIterator 实际上就是对 Step 的一个拓展代理，让 Step 拥有更多功能。

然后我们回到 FlatMapStep 的 processNextStart 方法，我们可以看出，实际上逻辑就是 flatMap( getPreviousStep().next() ) ，也就是先调用父Step的next方法，然后执行一个 flatMap。 具体的flatMap 方法逻辑由子类实现。

### debug简单总结
通过这次debug我们大概知道了整个程序运行的逻辑。
- g 是一个 GraphTraversalSource 类型对象，g 作为一个 遍历源每次初始会 new DefaultGraphTraversal<>(GraphTraversalSource)；
- 首先我们的代码，g.V().outE().inV().values() 每次调用都会添加一个 Step。 例如 g.V() 添加 GraphStep，outE() inV() values() 分别 VertexStep EdgeVertexStep PropertiesStep，他们都继承自 FlatMapStep。step之间有依赖关系。
- 当调用 Traversal 的 next 等action 方法，会触发查询。实际上会调用 finalEndStep 的next 方法。实际上会调用 AbstractStep 的 processNextStart。 不同的 processNextStart 会有不同的实现，例如 GraphStep 就是查库，FlatMapStep 就是调用上一个 Step 的 next 然后执行 flatMap 方法。类似 spark 的实现方式。

前面我们大概看了tinkerpop 的代码怎么一步步变成 Traversal 和 Step，然后怎么调用和执行。我们忽略了 strategy 的相关操作。

在查看源码之前我们尽量对她进行了解熟悉。TraversalStrategy 分析 Traversal，如果 Traversal 符合其标准尺度，则可以相应地进行改变。 strategies 在编译期执行，是 Gremlin traversal machine 编译器的基础，分为五种类型：
- 可以直接嵌入 traversal 逻辑的 application-level 的功能（decoration）；
- 能够更高效通过 TinkerPop3 语法表达 traversal （optimization）；
- 能够更高效在 graph 层面表达traversal（provider optimization）；
- 在执行 traversal 之前 一些必须的最终适配、清理、分析 (finalization)；
- 对于当前的程序或者存储系统来说，一些不合法的操作(verification)。

以 IdentityRemovalStrategy 为例：
```java
public final class IdentityRemovalStrategy extends AbstractTraversalStrategy<TraversalStrategy.OptimizationStrategy> implements TraversalStrategy.OptimizationStrategy {

    private static final IdentityRemovalStrategy INSTANCE = new IdentityRemovalStrategy();

    private IdentityRemovalStrategy() {
    }

    @Override
    public void apply(Traversal.Admin<?, ?> traversal) {
        if (traversal.getSteps().size() <= 1)
            return;

        for (IdentityStep<?> identityStep : TraversalHelper.getStepsOfClass(IdentityStep.class, traversal)) {
            if (identityStep.getLabels().isEmpty() || !(identityStep.getPreviousStep() instanceof EmptyStep)) {
                TraversalHelper.copyLabels(identityStep, identityStep.getPreviousStep(), false);
                traversal.removeStep(identityStep);
            }
        }
    }

    public static IdentityRemovalStrategy instance() {
        return INSTANCE;
    }
}
```
可以看出继承自 AbstractTraversalStrategy， 和 TraversalStrategy.OptimizationStrategy，并有一个 apply(Traversal.Admin<?, ?> traversal) 方法。 这里是直接在 traversal 中找到 identityStep 并移除。

### applyStrategies
strategies 初始化的时候就放入graph 中，我们这里看看如何嵌入到查询语句。

在我们调用 traversal.next() 之前，有个步骤就是 applyStrategies。在执行 traversal 之前大概是这样的：
```java
this = {DefaultGraphTraversal@1359} "[GraphStep(vertex,[1]), VertexStep(OUT,[knows],edge), EdgeVertexStep(IN), PropertiesStep([name],value)]"
 lastTraverser = {EmptyTraverser@1384} 
 finalEndStep = {EmptyStep@1385} 
 stepPosition = {StepPosition@1362} "4.0.0()"
 graph = {TinkerGraph@1386} "tinkergraph[vertices:6 edges:6]"
 steps = {ArrayList@1387}  size = 4
 unmodifiableSteps = {Collections$UnmodifiableRandomAccessList@1388}  size = 4
 parent = {EmptyStep@1385} 
 sideEffects = {DefaultTraversalSideEffects@1389} "sideEffects[size:0]"
 strategies = {DefaultTraversalStrategies@1361} "strategies[ConnectiveStrategy, IncidentToAdjacentStrategy, MatchPredicateStrategy, FilterRankingStrategy, InlineFilterStrategy, AdjacentToIncidentStrategy, RepeatUnrollStrategy, PathRetractionStrategy, CountStrategy, LazyBarrierStrategy, TinkerGraphCountStrategy, TinkerGraphStepStrategy, ProfileStrategy, StandardVerificationStrategy]"
 generator = null
 requirements = null
 locked = false
 bytecode = {Bytecode@1390} "[[], [V(1), outE(knows), inV(), values(name)]]"
```
可以看出是 DefaultGraphTraversal 的实体类，lastTraverser finalEndStep parent sideEffects 都是刚初始化，steps unmodifiableSteps 是4个。 然后我们执行完这个方法再来看看：
```java
this = {DefaultGraphTraversal@1359} "[TinkerGraphStep(vertex,[1]), VertexStep(OUT,[knows],vertex), PropertiesStep([name],value)]"
 lastTraverser = {EmptyTraverser@1384} 
 finalEndStep = {PropertiesStep@1432} "PropertiesStep([name],value)"
 stepPosition = {StepPosition@1362} "6.0.0()"
 graph = {TinkerGraph@1386} "tinkergraph[vertices:6 edges:6]"
 steps = {ArrayList@1387}  size = 3
 unmodifiableSteps = {Collections$UnmodifiableRandomAccessList@1388}  size = 3
 parent = {EmptyStep@1385} 
 sideEffects = {DefaultTraversalSideEffects@1389} "sideEffects[size:0]"
 strategies = {DefaultTraversalStrategies@1361} "strategies[ConnectiveStrategy, IncidentToAdjacentStrategy, MatchPredicateStrategy, FilterRankingStrategy, InlineFilterStrategy, AdjacentToIncidentStrategy, RepeatUnrollStrategy, PathRetractionStrategy, CountStrategy, LazyBarrierStrategy, TinkerGraphCountStrategy, TinkerGraphStepStrategy, ProfileStrategy, StandardVerificationStrategy]"
 generator = null
 requirements = {Collections$UnmodifiableSet@1431}  size = 1
 locked = true
 bytecode = {Bytecode@1390} "[[], [V(1), outE(knows), inV(), values(name)]]"
```
finalEndStep 变了，stepPosition 变了，steps unmodifiableSteps 都少了一步，requirements 多了一个。然后我们还是具体看看方法执行步骤：
```java
public void applyStrategies() throws IllegalStateException {
    if (this.locked) throw Traversal.Exceptions.traversalIsLocked(); // 判断是否执行过。
    TraversalHelper.reIdSteps(this.stepPosition, this);
    this.strategies.applyStrategies(this); // 这一步循环调用 strategy 的 apply 方法。
    {
        for (final TraversalStrategy<?> traversalStrategy : this.traversalStrategies) {
            traversalStrategy.apply(traversal);
        }
    }
    boolean hasGraph = null != this.graph;

    // 判断 TraversalParent 的所有 globalChild 也要进行 applyStrategies
    for (int i = 0, j = this.steps.size(); i < j; i++) { // "foreach" can lead to ConcurrentModificationExceptions
        final Step step = this.steps.get(i);
        if (step instanceof TraversalParent) {
            for (final Traversal.Admin<?, ?> globalChild : ((TraversalParent) step).getGlobalChildren()) {
                globalChild.setStrategies(this.strategies);
                globalChild.setSideEffects(this.sideEffects);
                if (hasGraph) globalChild.setGraph(this.graph);
                globalChild.applyStrategies();
            }
            for (final Traversal.Admin<?, ?> localChild : ((TraversalParent) step).getLocalChildren()) {
                localChild.setStrategies(this.strategies);
                localChild.setSideEffects(this.sideEffects);
                if (hasGraph) localChild.setGraph(this.graph);
                localChild.applyStrategies();
            }
        }
    }
    // 得到 finalEndStep 
    this.finalEndStep = this.getEndStep();
    // finalize requirements
    if (this.getParent() instanceof EmptyStep) {
        this.requirements = null;
        this.getTraverserRequirements();
    }
    this.locked = true;
}
```
整段代码并不复杂，复杂的是每个 traversalStrategy 具体都做了什么。

我们知道 tinkerpop 是一个独立的项目，里面是不带任何和 janus 相关的内容，如何会有 JanusGraphStep 呢， 其实在很多地方（例如 fill:175, Traversal (org.apache.tinkerpop.gremlin.process.traversal)）会调用 this.asAdmin().applyStrategies(); 会循环调用 traversalStrategy.apply(traversal)。在 apply 方法中得到了所有的 GraphStep。然后 new JanusGraphStep ，并且替换掉原有的 step。