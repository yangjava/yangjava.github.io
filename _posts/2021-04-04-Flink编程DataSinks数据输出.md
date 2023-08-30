---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# FlinkDataSinks数据输出


## DataSinks数据输出
通常情况下，用户希望将结果数据输出在外部存储介质或者传输到下游的消息中间件内，在Flink中将DataStream数据输出到外部系统的过程被定义为DataSink操作。
在Flink内部定义的第三方外部系统连接器中，支持数据输出的有Apache Kafka、Apache Cassandra、Kinesis、ElasticSearch、Hadoop FileSystem、RabbitMQ、NIFI等，除了Flink内部支持的第三方数据连接器之外，其他例如Apache Bahir框架也支持了相应的数据连接器，其中包括ActiveMQ、Flume、Redis、Akka、Netty等常用第三方系统。用户使用这些第三方Connector将DataStream数据集写入到外部系统中，需要将第三方连接器的依赖库引入到工程中。

### 基本数据输出
基本数据输出包含了文件输出、客户端输出、Socket网络端口等，这些输出方法已经在Flink DataStream API中完成定义，使用过程不需要依赖其他第三方的库。

如下代码所示，实现将DataStream数据集分别输出在本地文件系统和Socket网络端口。
- 输出到txt
```java
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        DataStreamSource<Integer> stream = env.fromElements(1, 2, 3, 4, 5, 6, 7, 8);

        stream.writeAsText("D:\\flink\\sink.txt");
        env.execute("Stream");
    }
}
```

-输出到csv文件中
```java
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        DataStreamSource<Tuple2> stream = env.fromElements(new Tuple2<>("Hello","Wolrd"));

        stream.writeAsText("D:\\flink\\sink.csv");
        env.execute("Stream");
    }
}
```

### 第三方数据输出
Flink中提供了DataSink类操作算子来专门处理数据的输出，所有的数据输出都可以基于实现SinkFunction完成定义。

#### Kafka
Flink中将数据输出到Kafka的操作。
```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka_2.11</artifactId>
  <version>${flink.version}</version>
</dependency>
```

```java
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer;
import org.apache.flink.streaming.connectors.kafka.KafkaSerializationSchema;
import org.apache.kafka.clients.producer.ProducerRecord;

import javax.annotation.Nullable;
import java.util.Properties;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        // 指定Kafka的相关配置属性
        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers", "localhost:9092");

        // 接收Kafka上的数据
        DataStream<String> stream = env
                .addSource(new FlinkKafkaConsumer<>("flink-stream-in-topic", new SimpleStringSchema(), properties));

        KafkaSerializationSchema<String> kafkaSerializationSchema = new KafkaSerializationSchema<String>() {
            @Override
            public ProducerRecord<byte[], byte[]> serialize(String element, @Nullable Long timestamp) {
                return new ProducerRecord<>("flink-stream-out-topic", element.getBytes());
            }
        };

        // 写出到Kafka
        FlinkKafkaProducer<String> kafkaProducer = new FlinkKafkaProducer<>("test",
                kafkaSerializationSchema,
                properties,
                FlinkKafkaProducer.Semantic.AT_LEAST_ONCE, 5);

        stream.addSink(kafkaProducer);
        stream.map((MapFunction<String, String>) value -> value + value).addSink(kafkaProducer);
        env.execute("Stream");
    }
}

```

#### 自定义 Sink
Flink 还支持使用自定义的 Sink 来满足多样化的输出需求。想要实现自定义的 Sink ，需要直接或者间接实现 SinkFunction 接口。通常情况下，我们都是实现其抽象类 RichSinkFunction，相比于 SinkFunction ，其提供了更多的与生命周期相关的方法。

自定义一个 FlinkToMySQLSink 为例，将计算结果写出到 MySQL 数据库中，具体步骤如下：
```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.16</version>
</dependency>
```

继承自 RichSinkFunction，实现自定义的 Sink ：
```java
public class FlinkToMySQLSink extends RichSinkFunction<Employee> {

    private PreparedStatement stmt;
    private Connection conn;

    @Override
    public void open(Configuration parameters) throws Exception {
        Class.forName("com.mysql.cj.jdbc.Driver");
        conn = DriverManager.getConnection("jdbc:mysql:/localhost:3306/test" +
                                           "?characterEncoding=UTF-8&serverTimezone=UTC&useSSL=false", 
                                           "root", 
                                           "123456");
        String sql = "insert into emp(name, age, birthday) values(?, ?, ?)";
        stmt = conn.prepareStatement(sql);
    }

    @Override
    public void invoke(Employee value, Context context) throws Exception {
        stmt.setString(1, value.getName());
        stmt.setInt(2, value.getAge());
        stmt.setDate(3, value.getBirthday());
        stmt.executeUpdate();
    }

    @Override
    public void close() throws Exception {
        super.close();
        if (stmt != null) {
            stmt.close();
        }
        if (conn != null) {
            conn.close();
        }
    }

}
```

想要使用自定义的 Sink，同样是需要调用 addSink 方法，具体如下：
```java
import java.util.Date;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {

        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        Date date = new Date(System.currentTimeMillis());
        DataStreamSource<Employee> streamSource = env.fromElements(
                new Employee("张三", 10, date),
                new Employee("李四", 20, date),
                new Employee("王五", 30, date));
        streamSource.addSink(new FlinkToMySQLSink());
        env.execute();
    }
}

```


