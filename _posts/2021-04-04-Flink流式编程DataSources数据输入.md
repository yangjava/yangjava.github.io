---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink DataSources数据接入

## DataSources数据输入
DataSources模块定义了DataStream API中的数据输入操作，Flink将数据源主要分为的内置数据源和第三方数据源这两种类型。
- 内置数据源
  包含文件、Socket网络端口以及集合类型数据，其不需要引入其他依赖库，且在Flink系统内部已经实现，用户可以直接调用相关方法使用。

- 第三方数据源定义了Flink和外部系统数据交互的逻辑，包括数据的读写接口。
  在Flink中定义了非常丰富的第三方数据源连接器（Connector），例如Apache kafka Connector、Elastic Search Connector等。同时用户也可以自定义实现Flink中数据接入函数SourceFunction，并封装成第三方数据源的Connector，完成Flink与其他外部系统的数据交互。

## 内置数据源
- 集合数据源
- 文件数据源
- Socket数据源

### 文件数据源
Flink系统支持将文件内容读取到系统中，并转换成分布式数据集DataStream进行数据处理。在StreamExecutionEnvironment中，可以使用readTextFile方法直接读取文本文件，也可以使用readFile方法通过指定文件InputFormat来读取特定数据类型的文件，其中InputFormat可以是系统已经定义的InputFormat类，如CsvInputFormat等，也可以用户自定义实现InputFormat接口类。

在DataStream API中，可以在readFile方法中指定文件读取类型（WatchType）、检测文件变换时间间隔（interval）、文件路径过滤条件（FilePathFilter）等参数。
WatchType共分为两种模式——PROCESS_CONTINUOUSLY和PROCESS_ONCE模式。
- PROCESS_CONTINUOUSLY
  在PROCESS_CONTINUOUSLY模式下，一旦检测到文件内容发生变化，Flink会将该文件全部内容加载到Flink系统中进行处理。
- PROCESS_ONCE
  而在PROCESS_ONCE模式下，当文件内容发生变化时，只会将变化的数据读取至Flink中，在这种情况下数据只会被读取和处理一次。

可以看出，在PROCESS_CONTINUOUSLY模式下是无法实现Excatly Once级别数据一致性保障的，而在PROCESS_ONCE模式，可以保证数据Excatly Once级别的一致性保证。但是需要注意的是，如果使用文件作为数据源，当某个节点异常停止的时候，这种情况下Checkpoints不会更新，如果数据一直不断地在生成，将导致该节点数据形成积压，可能需要耗费非常长的时间从最新的checkpoint中恢复应用。

- 读取txt文件
```java
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<String> resultStream = env.readTextFile("D:\\flink\\helloworld.txt");
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream WordCount");
    }

}

```

- 读取CSV格式文件
```java

import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.io.CsvInputFormat;
import org.apache.flink.api.java.io.RowCsvInputFormat;
import org.apache.flink.core.fs.Path;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.FileProcessingMode;
import org.apache.flink.types.Row;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        // 读取csv
        CsvInputFormat csvInput = new RowCsvInputFormat(
                new Path("D:\\flink\\helloworld.txt"),                                        // 文件路径
                new TypeInformation[]{Types.STRING, Types.STRING, Types.STRING},// 字段类型
                "\n",                                             // 行分隔符
                " ");                                            // 字段分隔符
        csvInput.setSkipFirstLineAsHeader(true);
        // 指定 CsvInputFormat, 监控csv文件(两种模式), 时间间隔是10ms
        DataStream<Row> resultStream = env.readFile(csvInput, "D:\\flink\\helloworld.txt", FileProcessingMode.PROCESS_CONTINUOUSLY, 10);
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream WordCount");
    }

}

```

### Socket数据源
Flink支持从Socket端口中接入数据，在StreamExecutionEnvironment调用socket-TextStream方法。该方法参数分别为Ip地址和端口，也可以同时传入字符串切割符delimiter和最大尝试次数maxRetry，其中delimiter负责将数据切割成Records数据格式；maxRetry在端口异常的情况，通过指定次数进行重连，如果设定为0，则Flink程序直接停止，不再尝试和端口进行重连。如下代码是使用socketTextStream方法实现了将数据从本地9999端口中接入数据并转换成DataStream数据集的操作。

在Unix系统环境下，可以执行nc –lk 9999命令启动端口，在客户端中输入数据，Flink就可以接收端口中的数据。
```java
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<String> resultStream = env.socketTextStream("localhost", 9999);
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream WordCount");
    }

}
```

### 集合数据源
Flink可以直接将集合类（Collection）转换成DataStream数据集，本质上是将本地集合中的数据分发到远端并行执行的节点中。目前Flink支持从Java.util.Collection和java.util.Iterator序列中转换成DataStream数据集。这种方式非常适合调试Flink本地程序，但需要注意的是，集合内的数据结构类型必须要一致，否则可能会出现数据转换异常。

- 通过fromElements从元素集合中创建DataStream数据集：
```java
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<Tuple2<String, Integer>> resultStream = env.fromElements(Tuple2.of("Hello", 1), Tuple2.of("World", 1));
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream WordCount");
    }

}
```

- 通过fromCollection从数组转创建DataStream数据集：
```java
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

import java.util.Arrays;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<String> resultStream = env.fromCollection(Arrays.asList("Hello", "World"));
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream WordCount");
    }

}
```


## 外部数据源

### 数据源连接器
流式计算类型的应用，数据大部分都是从外部第三方系统中获取，为此Flink通过实现SourceFunction定义了非常丰富的第三方数据连接器，基本覆盖了大部分的高性能存储介质以及中间件等，其中部分连接器是仅支持读取数据，例如Twitter Streaming API、Netty等；另外一部分仅支持数据输出（Sink），不支持数据输入（Source），例如Apache Cassandra、Elasticsearch、Hadoop FileSystem等。还有一部分是既支持数据输入，也支持数据输出，例如Apache Kafka、Amazon Kinesis、RabbitMQ等连接器。

### kafka-connector
以Kafka为例，主要因为Flink为了尽可能降低用户在使用Flink进行应用开发时的依赖复杂度，所有第三方连接器依赖配置放置在Flink基本依赖库以外，用户在使用过程中，根据需要将需要用到的Connector依赖库引入到应用工程中即可。
```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka-0.8_2.11</artifactId>
  <version>1.7.1</version>
</dependency>
```

在引入Maven依赖配置后，就可以在Flink应用工程中创建和使用相应的Connector，在kafka Connector中主要使用的其中参数有kafka topic、bootstrap.servers、zookeeper.connect。另外Schema参数的主要作用是根据事先定义好的Schema信息将数据序列化成该Schema定义的数据类型，默认是SimpleStringSchema，代表从Kafka中接入的数据将转换成String字符串类型处理
```java
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer08;

import java.util.Properties;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers", "localhost:9092");
        properties.setProperty("zookeeper.connect", "localhost:2181");
        properties.setProperty("group.id", "test");
        DataStream<String> resultStream = env
                .addSource(new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties));
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream WordCount");
    }

}

```

用户通过自定义Schema将接入数据转换成制定数据结构，主要是实现Deserialization-Schema接口来完成。可以看到在SourceEventSchema代码中，通过实现deserialize方法完成数据从byte[]数据类型转换成SourceEvent的反序列化操作，以及通过实现getProducedType方法将数据类型转换成Flink系统所支持的数据类型，
例如以下列代码中的TypeInformation<SourceEvent>类型。
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
用户也可以自己定义连接器，以满足不同的数据源的接入需求。
- 通过实现SourceFunction定义单个线程的接入的数据接入器
- 通过实现ParallelSource-Function接口或继承RichParallelSourceFunction类定义并发数据源接入器。

DataSoures定义完成后，可以通过使用SteamExecutionEnvironment的addSources方法添加数据源，这样就可以将外部系统中的数据转换成`DataStream[T]`数据集合，其中T类型是Source-Function返回值类型，然后就可以完成各种流式数据的转换操作。

- 自定义单并行度Source
```java
import org.apache.flink.streaming.api.functions.source.SourceFunction;

public class MyNoParallelSource implements SourceFunction<Long> {

    private long number = 1L;
    private boolean isRunning = true;

    @Override
    public void run(SourceContext<Long> sct) throws Exception {
        while (isRunning) {
            sct.collect(number);
            number++;
            //每秒生成一条数据
            Thread.sleep(1000);
        }
    }

    @Override
    public void cancel() {
        isRunning = false;
    }

}
```
定义Consume，消费Source的数据，并打印输出
```java
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<Long> resultStream = env.addSource(new MyNoParallelSource());
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream WordCount");
    }

}
```

- 自定义多并行度Source
  通过实现ParallelSourceFunction 接口or 继承RichParallelSourceFunction 来自定义有并行度的source。
```java
import org.apache.flink.streaming.api.functions.source.ParallelSourceFunction;

public class MyParallelSource implements ParallelSourceFunction<Long> {

    private long number = 1L;
    private boolean isRunning = true;
    @Override
    public void run(SourceContext<Long> sct) throws Exception {
        while (isRunning){
            sct.collect(number);
            number++;
            //每秒生成一条数据
            Thread.sleep(1000);
        }
    }
    @Override
    public void cancel() {
        isRunning=false;
    }
}

```
定义Consume，消费Source的数据，并打印输出
```java
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(3);
        // 如果不设置并行度，Source默认并行度为cpu core数。
        DataStreamSource<Long> resultStream = env.addSource(new MyParallelSource());
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream WordCount");
    }

}
```


