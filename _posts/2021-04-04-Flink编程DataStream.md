---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink编程DataStream
本章将重点介绍如何利用DataStream API开发流式应用，其中包括基本的编程模型、常用操作、时间概念、窗口计算、作业链等。4.1节将介绍在使用DataStream接口编程中的基本操作，例如如何定义数据源、数据转换、数据输出等操作，以及每种操作在Flink中如何进行拓展。4.2节将重点介绍Flink在流式计算过程中，对时间概念的区分和使用，其中包括事件时间（Event Time）、注入时间（Ingestion Time）、处理时间（Process Time）等时间概念，其中在事件时间中，会涉及Watermark等概念的解释和说明，帮助用户如何通过使用水印技术处理乱序数据。4.3节将介绍Flink在流式计算中常见的窗口计算类型，如滚动窗口、滑动窗口、会话窗口等，以及每种窗口的使用和应用场景。4.4节将介绍Flink应用通过使用作业链条操作，对Flink的任务进行优化，保证资源的合理利用。

## DataStream编程模型
在Flink整个系统架构中，对流计算的支持是其最重要的功能之一，Flink基于Google提出的DataFlow模型，实现了支持原生数据流处理的计算引擎。Flink中定义了DataStream API让用户灵活且高效地编写Flink流式应用。DataStream API主要可为分为三个部分，DataSource模块、Transformation模块以及DataSink模块，其中Sources模块主要定义了数据接入功能，主要是将各种外部数据接入至Flink系统中，并将接入数据转换成对应的DataStream数据集。在Transformation模块定义了对DataStream数据集的各种转换操作，例如进行map、filter、windows等操作。最后，将结果数据通过DataSink模块写出到外部存储介质中，例如将数据输出到文件或Kafka消息中间件等。

## DataSources数据输入
DataSources模块定义了DataStream API中的数据输入操作，Flink将数据源主要分为的内置数据源和第三方数据源这两种类型。其中内置数据源包含文件、Socket网络端口以及集合类型数据，其不需要引入其他依赖库，且在Flink系统内部已经实现，用户可以直接调用相关方法使用。第三方数据源定义了Flink和外部系统数据交互的逻辑，包括数据的读写接口。在Flink中定义了非常丰富的第三方数据源连接器（Connector），例如Apache kafka Connector、Elatic Search Connector等。同时用户也可以自定义实现Flink中数据接入函数SourceFunction，并封装成第三方数据源的Connector，完成Flink与其他外部系统的数据交互。

### 内置数据源

### 文件数据源
Flink系统支持将文件内容读取到系统中，并转换成分布式数据集DataStream进行数据处理。在StreamExecutionEnvironment中，可以使用readTextFile方法直接读取文本文件，也可以使用readFile方法通过指定文件InputFormat来读取特定数据类型的文件，其中InputFormat可以是系统已经定义的InputFormat类，如CsvInputFormat等，也可以用户自定义实现InputFormat接口类。
```java
  //直接读取文本文件
  val textStream = env.readTextFile("/user/local/data_example.log")
  //通过指定CSVInputFormat读取CSV文件
  val csvStream = env.readFile(new CsvInputFormat[String](new Path("/user/local/data_example.csv")) {
    override def fillRecord(out: String, objects: Array[AnyRef]): String = {
    return null
    }
}, "/user/local/data_example.csv")
```
在DataStream API中，可以在readFile方法中指定文件读取类型（WatchType）、检测文件变换时间间隔（interval）、文件路径过滤条件（FilePathFilter）等参数，其中WatchType共分为两种模式——PROCESS_CONTINUOUSLY和PROCESS_ONCE模式。在PROCESS_CONTINUOUSLY模式下，一旦检测到文件内容发生变化，Flink会将该文件全部内容加载到Flink系统中进行处理。而在PROCESS_ONCE模式下，当文件内容发生变化时，只会将变化的数据读取至Flink中，在这种情况下数据只会被读取和处理一次。

可以看出，在PROCESS_CONTINUOUSLY模式下是无法实现Excatly Once级别数据一致性保障的，而在PROCESS_ONCE模式，可以保证数据Excatly Once级别的一致性保证。但是需要注意的是，如果使用文件作为数据源，当某个节点异常停止的时候，这种情况下Checkpoints不会更新，如果数据一直不断地在生成，将导致该节点数据形成积压，可能需要耗费非常长的时间从最新的checkpoint中恢复应用。

### Socket数据源
Flink支持从Socket端口中接入数据，在StreamExecutionEnvironment调用socket-TextStream方法。该方法参数分别为Ip地址和端口，也可以同时传入字符串切割符delimiter和最大尝试次数maxRetry，其中delimiter负责将数据切割成Records数据格式；maxRetry在端口异常的情况，通过指定次数进行重连，如果设定为0，则Flink程序直接停止，不再尝试和端口进行重连。如下代码是使用socketTextStream方法实现了将数据从本地9999端口中接入数据并转换成DataStream数据集的操作。
```java
val socketDataStream = env.socketTextStream("localhost", 9999)
```
在Unix系统环境下，可以执行nc –lk 9999命令启动端口，在客户端中输入数据，Flink就可以接收端口中的数据。

## 集合数据源
Flink可以直接将Java或Scala程序中集合类（Collection）转换成DataStream数据集，本质上是将本地集合中的数据分发到远端并行执行的节点中。目前Flink支持从Java.util.Collection和java.util.Iterator序列中转换成DataStream数据集。这种方式非常适合调试Flink本地程序，但需要注意的是，集合内的数据结构类型必须要一致，否则可能会出现数据转换异常。

·通过fromElements从元素集合中创建DataStream数据集：
```java
val dataStream = env.fromElements(Tuple2(1L, 3L), Tuple2(1L, 5L), Tuple2(1L,
  7L), Tuple2(1L, 4L), Tuple2(1L, 2L))
```
·通过fromCollection从数组转创建DataStream数据集：
```java
String[] elements = new String[]{"hello", "flink"};
DataStream<String> dataStream = env.fromCollection(Arrays.asList(elements));
```
·将java.util.List转换成DataStream数据集：
````java
List<String> arrayList = new ArrayList<>()；
  arrayList.add("hello flink");
DataStream<String> dataList = env.fromCollection(arrayList);
````

### 外部数据源

### 数据源连接器
前面提到的数据源类型都是一些基本的数据接入方式，例如从文件、Socket端口中接入数据，其实质是实现了不同的SourceFunction，Flink将其封装成高级API，减少了用户的使用成本。对于流式计算类型的应用，数据大部分都是从外部第三方系统中获取，为此Flink通过实现SourceFunction定义了非常丰富的第三方数据连接器，基本覆盖了大部分的高性能存储介质以及中间件等，其中部分连接器是仅支持读取数据，例如Twitter Streaming API、Netty等；另外一部分仅支持数据输出（Sink），不支持数据输入（Source），例如Apache Cassandra、Elasticsearch、Hadoop FileSystem等。还有一部分是既支持数据输入，也支持数据输出，例如Apache Kafka、Amazon Kinesis、RabbitMQ等连接器。

以Kafka为例，主要因为Flink为了尽可能降低用户在使用Flink进行应用开发时的依赖复杂度，所有第三方连接器依赖配置放置在Flink基本依赖库以外，用户在使用过程中，根据需要将需要用到的Connector依赖库引入到应用工程中即可。
```java
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka-0.8_2.11</artifactId>
  <version>1.7.1</version>
</dependency>
```
在引入Maven依赖配置后，就可以在Flink应用工程中创建和使用相应的Connector，在kafka Connector中主要使用的其中参数有kafka topic、bootstrap.servers、zookeeper.connect。另外Schema参数的主要作用是根据事先定义好的Schema信息将数据序列化成该Schema定义的数据类型，默认是SimpleStringSchema，代表从Kafka中接入的数据将转换成String字符串类型处理
```java
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");
DataStream<String> input = env
.addSource(
new FlinkKafkaConsumer010<>(
properties.getString("input-data-topic"),
new SimpleStringSchema(),properties);
```
用户通过自定义Schema将接入数据转换成制定数据结构，主要是实现Deserialization-Schema接口来完成，代码清单4-4说明了KafkaEventSchema的定义。可以看到在SourceEventSchema代码中，通过实现deserialize方法完成数据从byte[]数据类型转换成SourceEvent的反序列化操作，以及通过实现getProducedType方法将数据类型转换成Flink系统所支持的数据类型，例如以下列代码中的TypeInformation<SourceEvent>类型。

````java
public class SourceEventSchema implements    
DeserializationSchema<SourceEvent>{
  private static final long serialVersionUID = 6154188370191669789L;
  @Override
  public SourceEvent deserialize(byte[] message) throws IOException {
    return SourceEvent.fromString(new String(message));
  }
  @Override
  public boolean isEndOfStream(SourceEvent nextElement) {
    return false;
  }
  @Override
  public TypeInformation< SourceEvent > getProducedType() {
    return TypeInformation.of(SourceEvent.class);
  }
}
````
针对Kafka数据的解析，Flink提供了KeyedDeserializationSchema，其中deserialize方法定义为T deserialize(byte[] messageKey, byte[] message, String topic, int partition, long offset)，支持将Message中的key和value同时解析出来。

同时为了更方便地解析各种序列化类型的数据，Flink内部提供了常用的序列化协议的Schema，例如TypeInformationSerializationSchema、JsonDeserializationSchema和AvroDeserializationSchema等，用户可以根据需要选择使用。

### 自定义数据源连接器
Flink中已经实现了大多数主流的数据源连接器，但需要注意，Flink的整体架构非常开放，用户也可以自己定义连接器，以满足不同的数据源的接入需求。可以通过实现SourceFunction定义单个线程的接入的数据接入器，也可以通过实现ParallelSource-Function接口或继承RichParallelSourceFunction类定义并发数据源接入器。DataSoures定义完成后，可以通过使用SteamExecutionEnvironment的addSources方法添加数据源，这样就可以将外部系统中的数据转换成DataStream[T]数据集合，其中T类型是Source-Function返回值类型，然后就可以完成各种流式数据的转换操作。

## DataStream转换操作
即通过从一个或多个DataStream生成新的DataStream的过程被称为Transformation操作。在转换过程中，每种操作类型被定义为不同的Operator，Flink程序能够将多个Transformation组成一个DataFlow的拓扑。所有DataStream的转换操作可分为单Single-DataStream、Multi-DaataStream、物理分区三类类型。其中Single-DataStream操作定义了对单个DataStream数据集元素的处理逻辑，Multi-DataStream操作定义了对多个DataStream数据集元素的处理逻辑。物理分区定义了对数据集中的并行度和数据分区调整转换的处理逻辑。

### Single-DataStream操作

### Map [DataStream->DataStream]
调用用户定义的MapFunction对DataStream[T]数据进行处理，形成新的Data-Stream[T]，其中数据格式可能会发生变化，常用作对数据集内数据的清洗和转换。例如将输入数据集中的每个数值全部加1处理，并且将数据输出到下游数据集。

通过从集合中创建dataStream，并调用DataStream的map方法传入计算表达式，完成对第二个字段加1操作，最后得到新的数据集mapStream。
```java
val dataStream = env.fromElements(("a", 3), ("d", 4), ("c", 2), ("c", 5), ("a",
  5))
  //指定map计算表达式
val mapStream: DataStream[(String, Int)] = dataStream.map(t => (t._1, t._2 + 1))
```

除了可以在map方法中直接传入计算表达式，如下代码实现了MapFunction接口定义map函数逻辑，完成数据处理操作。其中MapFunction[(String, Int), (String, Int)]中共有两个参数，第一个参数(String, Int)代表输入数据集数据类型，第二个参数(String, Int)代表输出数据集数据类型。
```java
//通过指定MapFunction
val mapStream: DataStream[(String, Int)] = dataStream.map(new MapFunction[(String, Int), (String, Int)] {
  override def map(t: (String, Int)): (String, Int) = {
    (t._1, t._2 + 1)}
  })
```
以上两种方式得到的结果一样，但是第二种方式在使用Java语言的时候用得较多，用户可以根据自己的需要偏好使用。

### FlatMap [DataStream->DataStream]
该算子主要应用处理输入一个元素产生一个或者多个元素的计算场景，比较常见的是在经典例子WordCount中，将每一行的文本数据切割，生成单词序列对于输入DataStream[String]通过FlatMap函数进行处理，字符串数字按逗号切割，然后形成新的整数数据集。

针对上述计算逻辑实现代码如下所示，通过调用resultStream接口中flatMap方法将定义好的FlatMapFunction传入，生成新的数据集。FlatMapFunction的接口定义为FlatMapFunction[T, O] { flatMap(T, Collector[O]): Unit }其中T为输入数据集的元素格式，O为输出数据集的元素格式。
```java
val dataStream:DataStream[String] = environment.fromCollections()
val resultStream[String] = dataStream.flatMap { str => str.split(" ") }
```

### Filter [DataStream->DataStream]
该算子将按照条件对输入数据集进行筛选操作，将符合条件的数据集输出，将不符合条件的数据过滤掉。将输入数据集中偶数过滤出来，奇数从数据集中去除。
```java
//通过通配符
val filter:DataStream[Int] = dataStream.filter { _ % 2 == 0 }
//或者指定运算表达式
val filter:DataStream[Int] = dataStream.filter { x => x % 2 == 0 }
```

### KeyBy [DataStream->KeyedStream]
该算子根据指定的Key将输入的DataStream[T]数据格式转换为KeyedStream[T]，也就是在数据集中执行Partition操作，将相同的Key值的数据放置在相同的分区中。
将白色方块和灰色方块通过颜色的Key值重新分区，将数据集分为具有灰色方块的数据集合。
将数据集中第一个参数作为Key，对数据集进行KeyBy函数操作，形成根据id分区的KeyedStream数据集。其中keyBy方法输入为DataStream[T]数据集。
```java
val dataStream = env.fromElements((1, 5),(2, 2),(2, 4),(1, 3))
//指定第一个字段为分区Key
val keyedStream: KeyedStream[(String,Int), Tuple] = dataStream.keyBy(0)
```
以下两种数据类型将不能使用KeyBy方法对数据集进行重分区：1）用户使用POJOs类型数据，但是POJOs类中没有复写hashCode()方法，而是依赖于Object.hasCode()；2）任何数据类型的数组结构。

### Reduce [KeyedStream->DataStream]
该算子和MapReduce中Reduce原理基本一致，主要目的是将输入的KeyedStream通过传入的用户自定义的ReduceFunction滚动地进行数据聚合处理，其中定义的ReduceFunciton必须满足运算结合律和交换律。如下代码对传入keyedStream数据集中相同的key值的数据独立进行求和运算，得到每个key所对应的求和值。
```java
val dataStream = env.fromElements(("a", 3), ("d", 4), ("c", 2), ("c",
5), ("a", 5))
  //指定第一个字段为分区Key
val keyedStream: KeyedStream[(String,Int), Tuple] = dataStream.keyBy(0)
//滚动对第二个字段进行reduce相加求和
val reduceStream = keyedStream.reduce { (t1, t2) =>
    (t1._1, t1._2 + t2._2)
}
```
用户也可以单独定义Reduce函数，如下代码所示：
```java
//通过实现ReduceFunction匿名类
val reduceStream1 = keyedStream.reduce(new ReduceFunction[(String, Int)] {
    override def reduce(t1: (String, Int), t2: (String, Int)): (String, Int)={ 
    (t1._1, t1._2 + t2._2)
  }})
```
运行代码的输出结果依次为：（c，2）（c，7）（a，3）（d，4）（a，8）。

### Aggregations[KeyedStream->DataStream]
Aggregations是DataStream接口提供的聚合算子，根据指定的字段进行聚合操作，滚动地产生一系列数据聚合结果。其实是将Reduce算子中的函数进行了封装，封装的聚合操作有sum、min、minBy、max、maxBy等，这样就不需要用户自己定义Reduce函数。如下代码所示，指定数据集中第一个字段作为key，用第二个字段作为累加字段，然后滚动地对第二个字段的数值进行累加并输出。
```java
//指定第一个字段为分区Key
val keyedStream: KeyedStream[(Int, Int), Tuple] = dataStream.keyBy(0)
//对第二个字段进行sum统计
val sumStream: DataStream[(Int, Int)] = keyedStream.sum(1)
//输出计算结果
sumStream.print()
```
代码执行完毕后结果输出在客户端，其中key为1的统计结果为（1，5）和（1，8），key为2的统计结果为（2，2）和（2，6）。可以看出，计算出来的统计值并不是一次将最终整个数据集的最后求和结果输出，而是将每条记录所叠加的结果输出。

聚合函数中需要传入的字段类型必须是数值型，否则会抛出异常。对应其他的聚合函数的用法如下代码所示。
```java
val minStream: DataStream[(Int, Int)] = keyedStream.min(1)
//滚动计算指定key的最大值
val maxStream: DataStream[(Int, Int)] = keyedStream.max(1)
//滚动计算指定key的最小值，返回最大值对应的元素
val minByStream: DataStream[(Int, Int)] = keyedStream.minBy(1)
//滚动计算指定key的最大值，返回最大值对应的元素
val maxByStream: DataStream[(Int, Int)] = keyedStream.maxBy(1)
```

### Multi-DataStream操作

### Union[DataStream ->DataStream]
Union算子主要是将两个或者多个输入的数据集合并成一个数据集，需要保证两个数据集的格式一致，输出的数据集的格式和输入的数据集格式保持一致，如图4-5所示，将灰色方块数据集和黑色方块数据集合并成一个大的数据集。

可以直接调用DataStream API中的union()方法来合并多个数据集，方法中传入需要合并的DataStream数据集。如下代码所示，分别将创建的数据集dataStream_01和dataStream_02合并，如果想将多个数据集同时合并则在union()方法中传入被合并的数据集的序列即可。
```java
//创建不同的数据集
val dataStream1: DataStream[(String, Int)] = env.fromElements(("a", 3), ("d", 4), ("c", 2), ("c", 5), ("a", 5))
val dataStream2: DataStream[(String, Int)] = env.fromElements(("d", 1), ("s", 2), ("a", 4), ("e", 5), ("a", 6))
val dataStream3: DataStream[(String, Int)] = env.fromElements(("a", 2), ("d", 1), ("s", 2), ("c", 3), ("b", 1))
//合并两个DataStream数据集
val unionStream = dataStream1.union(dataStream_02)
//合并多个DataStream数据集
val allUnionStream = dataStream1.union(dataStream2, dataStream3)
```

### Connect，CoMap，CoFlatMap[DataStream ->DataStream]
Connect算子主要是为了合并两种或者多种不同数据类型的数据集，合并后会保留原来数据集的数据类型。连接操作允许共享状态数据，也就是说在多个数据集之间可以操作和查看对方数据集的状态，关于状态操作将会在后续章节中重点介绍。如下代码所示，dataStream1数据集为(String, Int)元祖类型，dataStream2数据集为Int类型，通过connect连接算子将两个不同数据类型的算子结合在一起，形成格式为ConnectedStreams的数据集，其内部数据为[(String, Int), Int]的混合数据类型，保留了两个原始数据集的数据类型。
```java
//创建不同数据类型的数据集
val dataStream1: DataStream[(String, Int)] = env.fromElements(("a", 3), ("d", 4), ("c", 2), ("c", 5), ("a", 5))
val dataStream2: DataStream[Int] = env.fromElements(1, 2, 4, 5, 6)
//连接两个DataStream数据集
val connectedStream: ConnectedStreams[(String, Int), Int] = dataStream1.connect(dataStream2)
```
需要注意的是，对于ConnectedStreams类型的数据集不能直接进行类似Print()的操作，需要再转换成DataStream类型数据集，在Flink中ConnectedStreams提供的map()方法和flatMap() 需要定义CoMapFunciton或CoFlatMapFunction分别处理输入的DataStream数据集，或者直接传入两个MapFunction来分别处理两个数据集。如下代码所示，通过定义CoMapFunction处理ConnectedStreams数据集中的数据，指定的参数类型有三个，其中（String，Int）和Int分别指定的是第一个和第二个数据集的数据类型，（Int，String）指定的是输出数据集的数据类型，在函数定义中需要实现map1和map2两个方法，分别处理输入两个数据集，同时两个方法返回的数据类型必须一致。

```java
val resultStream = connectedStream.map(new CoMapFunction[(String, Int), Int,
  (Int, String)] {
//定义第一个数据集函数处理逻辑，输入值为第一个DataSteam
    override def map1(in1: (String, Int)): (Int, String) = {
      (in1._2, in1._1)
    }
  //定义第二个函数处理逻辑，输入值为第二个DataStream
    override def map2(in2: Int): (Int, String) = {
      (in2, "default")
    }})
```
在以上实例中，两个函数会多线程交替执行产生结果，最终将两个数据集根据定义合并成目标数据集。和CoMapFunction相似，在flatmap()方法中需要指定CoFlatMapFunction。如下代码所示，通过实现CoFlatMapFunction接口中flatMap1()方法和flatMap2()方法，分别对两个数据集进行处理，同时可以在两个函数之间共享number变量，完成两个数据集的数据合并整合。

```java
val resultStream2 = connectedStream.flatMap(new CoFlatMapFunction[(String,
  Int), Int, (String, Int, Int)] {
    //定义共享变量
    var number = 0
    //定义第一个数据集处理函数
    override def flatMap1(in1: (String, Int), collector: Collector[(String, Int, Int)]): Unit = {
      collector.collect((in1._1, in1._2, number))
    }
    //定义第二个数据集处理函数
    override def flatMap2(in2: Int, collector: Collector[(String, Int, Int)]): Unit = {
      number = in2
    }
}
)
```
通常情况下，上述CoMapFunction或者CoFlatMapFunction函数并不能有效地解决数据集关联的问题，产生的结果可能也不是用户想使用的，因为用户可能想通过指定的条件对两个数据集进行关联，然后产生相关性比较强的结果数据集。这个时候就需要借助keyBy函数或broadcast广播变量实现。

```java
// 通过keyby函数根据指定的key连接两个数据集
val keyedConnect: ConnectedStreams[(String, Int), Int] = dataStream1.connect(dataStream2).keyBy(1, 0)
// 通过broadcast关联两个数据集
val broadcastConnect: BroadcastConnectedStream[(String, Int), Int] = dataStream1.connect(dataStream2.broadcast())
```
通过使用keyby函数会将相同的key的数据路由在一个相同的Operator中，而BroadCast广播变量会在执行计算逻辑之前将dataStream2数据集广播到所有并行计算的Operator中，这样就能够根据条件对数据集进行关联，这其实也是分布式Join算子的基本实现方式。

CoMapFunction和CoFlaMapFunction中的两个方法，在Paralism>1的情况下，不会按照指定的顺序指定，因此有可能会影响输出数据的顺序和结果，这点用户在使用过程中需要注意。

### Split [DataStream->SplitStream]
Split算子是将一个DataStream数据集按照条件进行拆分，形成两个数据集的过程，也是union算子的逆向实现。每个接入的数据都会被路由到一个或者多个输出数据集中。如图4-6所示，将输入数据集根据颜色切分成两个数据集。

在使用split函数中，需要定义split函数中的切分逻辑，如下代码所示，通过调用split函数，然后指定条件判断函数，将根据第二个字段的奇偶性将数据集标记出来，如果是偶数则标记为even，如果是奇数则标记为odd，然后通过集合将标记返回，最终生成格式SplitStream的数据集。
```java
//创建数据集
val dataStream1: DataStream[(String, Int)] = env.fromElements(("a", 3), ("d", 4), ("c", 2), ("c", 5), ("a", 5))
//合并两个DataStream数据集
val splitedStream: SplitStream[(String, Int)] = dataStream1.split(t => if (t._2 % 2 == 0) Seq("even") else Seq("odd"))
```

### Select [SplitStream ->DataStream]
split函数本身只是对输入数据集进行标记，并没有将数据集真正的实现切分，因此需要借助Select函数根据标记将数据切分成不同的数据集。如下代码所示，通过调用SplitStream数据集的select()方法，传入前面已经标记好的标签信息，然后将符合条件的数据筛选出来，形成新的数据集。
```java
//筛选出偶数数据集
val evenStream: DataStream[(String, Int)] = splitedStream.select("even")
//筛选出奇数数据集
val oddStream: DataStream[(String, Int)] = splitedStream.select("odd")
//筛选出奇数和偶数数据集
val allStream: DataStream[(String, Int)] = splitedStream.select("even", "odd")
```

### Iterate[DataStream->IterativeStream->DataStream]
Iterate算子适合于迭代计算场景，通过每一次的迭代计算，并将计算结果反馈到下一次迭代计算中。如下代码所示，调用dataStream的iterate()方法对数据集进行迭代操作，如果事件指标加1后等于2，则将计算指标反馈到下一次迭代的通道中，如果事件指标加1不等于2则直接输出到下游DataStream中。其中在执行之前需要对数据集做map处理主要目的是为了对数据分区根据默认并行度进行重平衡，在iterate()内参数类型为ConnectedStreams，然后调用ConnectedStreams的方法内分别执行两个map方法，第一个map方法执行反馈操作，第二个map函数将数据输出到下游数据集。
```java
val dataStream = env.fromElements(3, 1, 2, 1, 5).map { t: Int => t }
val iterated = dataStream.iterate((input: ConnectedStreams[Int, String]) => {
//分别定义两个map方法完成对输入ConnectedStreams数据集数据的处理
    val head = input.map(i => (i + 1).toString, s => s)
    (head.filter(_ == "2"), head.filter(_ != "2"))
  }, 1000)//1000指定最长迭代等待时间，单位为ms，超过该时间没有数据接入则终止迭代
```

## 物理分区操作
物理分区（Physical Partitioning）操作的作用是根据指定的分区策略将数据重新分配到不同节点的Task案例上执行。当使用DataStream提供的API对数据处理过程中，依赖于算子本身对数据的分区控制，如果用户希望自己控制数据分区，例如当数据发生了数据倾斜的时候，就需要通过定义物理分区策略的方式对数据集进行重新分布处理。Flink中已经提供了常见的分区策略，例如随机分区（Random Partitioning）、平衡分区（Roundobin Partitioning）、按比例分区（Roundrobin Partitioning）等。当然如果给定的分区策略无法满足需求，也可以根据Flink提供的分区控制接口创建分区器，实现自定义分区控制。

Flink内部提供的常见数据重分区策略如下所述。

### 随机分区（Random Partitioning）：[DataStream ->DataStream]
通过随机的方式将数据分配在下游算子的每个分区中，分区相对均衡，但是较容易失去原有数据的分区结构。

```java
//通过调用DataStream API中的shuffle方法实现数据集的随机分区
  val shuffleStream = dataStream.shuffle
```

### Roundrobin Partitioning：[DataStream ->DataStream]
通过循环的方式对数据集中的数据进行重分区，能够尽可能保证每个分区的数据平衡，当数据集发生数据倾斜的时候使用这种策略就是比较有效的优化方法。
```java
//通过调用DataStream API中rebalance()方法实现数据的重平衡分区
  val shuffleStream = dataStream.rebalance();
```

### Rescaling Partitioning：[DataStream ->DataStream]
和Roundrobin Partitioning一样，Rescaling Partitioning也是一种通过循环的方式进行数据重平衡的分区策略。但是不同的是，当使用Roundrobin Partitioning时，数据会全局性地通过网络介质传输到其他的节点完成数据的重新平衡，而Rescaling Partitioning仅仅会对上下游继承的算子数据进行重平衡，具体的分区主要根据上下游算子的并行度决定。例如上游算子的并发度为2，下游算子的并发度为4，就会发生上游算子中一个分区的数据按照同等比例将数据路由在下游的固定的两个分区中，另外一个分区同理路由到下游两个分区中。
```java
//通过调用DataStream API中rescale()方法实现Rescaling Partitioning操作
  val shuffleStream = dataStream.rescale();
```

### 广播操作（Broadcasting）：[DataStream ->DataStream]
广播策略将输入的数据集复制到下游算子的并行的Tasks实例中，下游算子中的Tasks可以直接从本地内存中获取广播数据集，不再依赖于网络传输。这种分区策略适合于小数据集，例如当大数据集关联小数据集时，可以通过广播的方式将小数据集分发到算子的每个分区中。
```java
//可以通过调用DataStream API 的broadcast()方法实现广播分区
val shuffleStream = dataStream.broadcast();
```

### 自定义分区（Custom Partitioning）：[DataStream ->DataStream]
除了使用已有的分区器之外，用户也可以实现自定义分区器，然后调用DatSstream API上partitionCustom()方法将创建的分区器应用到数据集上。如以下代码所示自定义分区器代码实现了当字段中包含“flink”关键字的数据放在partition为0的分区中，其余数据随机进行分区的策略，其中num Partitions是从系统中获取的并行度参数。
```java
object customPartitioner extends Partitioner[String] {
  //获取随机数生成器
  val r = scala.util.Random
  override def partition(key: String, numPartitions: Int): Int = {
    //定义分区策略，key中如果包含a则放在0分区中，其他情况则根据Partitions num随机分区
    if (key.contains("flink")) 0 else r.nextInt(numPartitions)
  }
}
```
自定义分区器定义好之后就可以调用DataSteam API的partitionCustom来应用分区器，第二个参数指定分区器使用到的字段，对于Tuple类型数据，分区字段可以通过字段名称指定，其他类型数据集则通过位置索引指定。
```java
//通过数据集字段名称指定分区字段
dataStream.partitionCustom(customPartitioner, "filed_name");
//通过数据集字段索引指定分区字段
dataStream.partitionCustom(customPartitioner, 0);
```

## DataSinks数据输出
经过各种数据Transformation操作后，最终形成用户需要的结果数据集。通常情况下，用户希望将结果数据输出在外部存储介质或者传输到下游的消息中间件内，在Flink中将DataStream数据输出到外部系统的过程被定义为DataSink操作。在Flink内部定义的第三方外部系统连接器中，支持数据输出的有Apache Kafka、Apache Cassandra、Kinesis、ElasticSearch、Hadoop FileSystem、RabbitMQ、NIFI等，除了Flink内部支持的第三方数据连接器之外，其他例如Apache Bahir框架也支持了相应的数据连接器，其中包括ActiveMQ、Flume、Redis、Akka、Netty等常用第三方系统。用户使用这些第三方Connector将DataStream数据集写入到外部系统中，需要将第三方连接器的依赖库引入到工程中。

### 基本数据输出
基本数据输出包含了文件输出、客户端输出、Socket网络端口等，这些输出方法已经在Flink DataStream API中完成定义，使用过程不需要依赖其他第三方的库。如下代码所示，实现将DataStream数据集分别输出在本地文件系统和Socket网络端口。
```java
val personStream = env.fromElements(("Alex", 18), ("Peter", 43))
//通过writeAsCsv方法将数据转换成CSV文件输出，并执行输出模式为OVERWRITE
personStream.writeAsCsv("file:///path/to/person.csv",WriteMode.OVERWRITE)
//通过writeAsText方法将数据直接输出到本地文件系统
personStream.writeAsText("file:///path/to/person.txt")
//通过writeToSocket方法将DataStream数据集输出到指定Socket端口
personStream.writeToSocket(outputHost, outputPort, new SimpleStringSchema())
```

### 第三方数据输出
通常情况下，基于Flink提供的基本数据输出方式并不能完全地满足现实场景的需要，用户一般都会有自己的存储系统，因此需要将Flink系统中计算完成的结果数据通过第三方连接器输出到外部系统中。Flink中提供了DataSink类操作算子来专门处理数据的输出，所有的数据输出都可以基于实现SinkFunction完成定义。例如在Flink中定义了FlinkKafkaProducer类来完成将数据输出到Kafka的操作，需要根据不同的Kafka版本需要选择不同的FlinkKafkaProducer，目前FlinkKafkaProducer类支持Kafka大于1.0.0的版本，FlinkKafkaProducer11或者010支持Kafka0.10.0.x的版本。通过使用FlinkKafkaProducer11将DataStream中的数据写入Kafka的Topic中。
```java
val wordStream = env.fromElements("Alex", "Peter", "Linda")
//定义FlinkKafkaProducer011 Sink算子
val kafkaProducer = new FlinkKafkaProducer011[String](
    "localhost:9092", // 指定Broker List参数
    "kafka-topic", // 指定目标Kafka Topic名称
    new SimpleStringSchema) // 设定序列化Schema
  /通过addsink添加kafkaProducer到算子拓扑中
  wordStream.addSink(kafkaProducer)
```

在以上代码中使用FlinkKafkaProducer往Kafka中写入数据的操作相对比较基础，还可以配置一些高级选项，例如可以配置自定义properties类，将自定义的参数通过properties类传入FlinkKafkaProducer中。另外还可以自定义Partitioner将DataStream中的数据按照指定分区策略写入Kafka的分区中。也可以使用KeyedSerializationSchema对序列化Schema进行优化，从而能够实现一个Producer往多个Topic中写入数据的操作。








