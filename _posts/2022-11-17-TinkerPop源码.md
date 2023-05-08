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



