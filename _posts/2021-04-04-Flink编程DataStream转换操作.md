---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink DataStream转换操作

## Transformation转换操作
通过从一个或多个DataStream生成新的DataStream的过程被称为Transformation操作。Flink 程序被执行的时候，它会被映射为 Streaming Dataflow。

一个 Streaming Dataflow是由一组 Stream 和 Transformation Operator组成，它类似于一个 DAG 图，在启动的时候从一个或多个 Source Operator 开始，结束于一个或多个 Sink Operator。

数据转换的各种操作。有 `map/flatMap/filter/keyBy/reduce/fold/aggregations/window/windowAll/union/window join/split/select/project` 等，可以将数据转换计算成你想要的数据。

### map
映射函数。即：取出一个元素，根据规则处理后，并产生一个元素。调用用户定义的MapFunction对DataStream数据进行处理，形成新的Data-Stream，常用作对数据集内数据的清洗和转换。
类型转换：DataStream -> DataStream

使用 java.util.Collection 创建一个数据流，并将数据流中的数据 * 2，并输出。
```java
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //创建数据流
        DataStreamSource<Integer> dataStream = env.fromElements(1, 2, 3, 4, 5);
        //将Stream流中的数据 *2
        SingleOutputStreamOperator<Integer> resultStream = dataStream.map(num -> num * 2).returns(Types.INT);
        // 如果不设置并行度，Source默认并行度为cpu core数。
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream WordCount");
    }

}
```

### flatMap
拆分压平。即：取出一个元素，并产生零个、一个或多个元素。 flatMap 和 map 方法的使用相似，但是因为一般 Java 方法的返回值结果都是一个，引入 flatMap 后，我们可以将处理后的多个结果放到一个 Collections 集合中（类似于返回多个结果）
类型转换：DataStream -> DataStream

场景:使用 java.util.Collection 创建一个数据流，并将数据流中以 “S” 开头的数据返回。
```java
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

import java.util.Arrays;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        //创建数据流
        DataStreamSource<String> dataStreamSource = env.fromElements("Hadoop Flink Storm HBase", "Spark Tomcat Spring MyBatis", "Sqoop Flume Docker K8S Scala");
        //将Stream流中以 "S" 开头的数据，输出到 Collectior 集合中
        SingleOutputStreamOperator<String> resultStream = dataStreamSource.flatMap((String line, Collector<String> out) -> {
            Arrays.stream(line.split(" ")).forEach(str -> {
                if (str.startsWith("S")) {
                    out.collect(str);
                }
            });
        }).returns(Types.STRING);
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream");
    }

}

```

### filter
过滤。即：为取出的每个元素进行规则判断(返回true/false)，并保留该函数返回 true 的数据。
类型转换：DataStream -> DataStream

使用 java.util.Collection 创建一个数据流，并将数据流中以 “S” 开头的数据返回。
```java
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        //创建数据流
        DataStreamSource<String> dataStreamSource = env.fromElements("Hadoop", "Spark", "Tomcat", "Storm", "Flink", "Docker", "Hive", "Sqoop" );
        //将Stream流中以 "S" 开头的数据，输出
        SingleOutputStreamOperator<String> resultStream = dataStreamSource.filter(str -> str.startsWith("S")).returns(Types.STRING);
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream");
    }

}
```

### keyBy
按 key 进行分组，key相同的(一定)会进入到同一个组中。具有相同键的所有记录都会分配给同一分区。在内部，keyBy() 是通过哈希分区实现的，有多种指定密钥的方法。
类型转换：DataStream → KeyedStream

此转换返回一个KeyedStream。使用 keyBy 进行分区，分为以下下两种情况：

#### 根据字段名称指定
- POJO类型：以 “属性名” 进行分组(属性名错误或不存在，会提示错误)
```
dataStream.keyBy(“someKey”) // Key by field “someKey”
```
- Tuple(元组)类型：以“0、1、2”等进行分组(角标从0开始)
```
dataStream.keyBy(0) // Key by the first element of a Tuple
```
keyBy() ，也支持以多个字段进行分组。
- keyBy(0,1)：Tuple形式以第1和第2个字段进行分组
- keyBy(“province”,“city”)：POJO形式以"province"和"city"两个字段进行分组。多字段分组使用方法，与单个字段分组类似。

#### 通过Key选择器指定
另外一种方式是通过定义Key Selector来选择数据集中的Key，如下代码所示，定义KeySelector，然后复写getKey方法，从Person对象中获取name为指定的Key。

场景:通过 Socket 方式，实时获取输入的数据，并对数据流中的单词进行分组求和计算（如何通过Socket输入数据，请参考：Java编写实时任务WordCount）
POJO(实体类)方式 keyBy
```java
public class WordCount {

    public String word;

    public int count;

    public WordCount() {
    }

    public WordCount(String word, int count) {
        this.word = word;
        this.count = count;
    }

    //of()方法，用来生成 WordCount 类(Flink源码均使用of()方法形式，省去每次new操作。诸如:Tuple2.of())
    public static WordCount of(String word,int count){
        return new WordCount(word, count);
    }

    @Override
    public String toString() {
        return "WordCount{" +
                "word='" + word + '\'' +
                ", count=" + count +
                '}';
    }
}
```
```java
import org.apache.flink.api.java.tuple.Tuple;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

import java.util.Arrays;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //通过Socket实时获取数据
        DataStreamSource<String> dataStreamSource = env.fromElements("Hadoop Spark Tomcat Storm Flink Docker Hive Sqoop Flink Sqoop Flink" );
        //将数据转换成 POJO 实体类形式
        SingleOutputStreamOperator<WordCount> streamOperator = dataStreamSource.flatMap((String line, Collector<WordCount> out) -> {
            Arrays.stream(line.split(" ")).forEach(str -> out.collect(WordCount.of(str, 1)));
        }).returns(WordCount.class);
        //keyBy()以属性名:word 进行分组
        KeyedStream<WordCount, Tuple> keyedStream = streamOperator.keyBy("word");
        //sum()以属性名:count 进行求和
        SingleOutputStreamOperator<WordCount> resultStream = keyedStream.sum("count");
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream");
    }

}
```

- Tuple(元组)方式 keyBy
```java
public class KeyByDemo {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //通过Socket实时获取数据
        DataStreamSource<String> dataSource = env.socketTextStream("localhost", 8888);
        //将数据转换成元组(word,1)形式
        SingleOutputStreamOperator<Tuple2<String, Integer>> streamOperator = dataSource.flatMap((String lines, Collector<Tuple2<String, Integer>> out) -> {
            Arrays.stream(lines.split(" ")).forEach(word -> out.collect(Tuple2.of(word, 1)));
        }).returns(Types.TUPLE(Types.STRING, Types.INT));
        //keyBy()以下标的形式,进行分组
        KeyedStream<Tuple2<String, Integer>, Tuple> keyedStream = streamOperator.keyBy(0);
        //sum()以下标的形式，对其进行求和
        SingleOutputStreamOperator<Tuple2<String, Integer>> summed = keyedStream.sum(1);

        summed.print();

        env.execute("KeyByDemo");
    }
}
```

### Reduce
归并操作。如果需要将数据流中的所有数据，归纳得到一个数据的情况，可以使用 reduce() 方法。如果需要对数据流中的数据进行求和操作、求最大/最小值等(都是归纳为一个数据的情况)，此处就可以用到 reduce() 方法
类型转换：KeyedStream → DataStream

reduce() 返回单个的结果值，并且 reduce 操作每处理一个元素总是会创建一个新的值。常用的聚合操作例如 min()、max() 等都可使用 reduce() 方法实现。Flink 中未实现的 average(平均值), count(计数) 等操作，也都可以通过 reduce()方法实现。

对数据流中的单词进行分组，分组后进行 count 计数操作
```java
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //通过Socket实时获取数据
        DataStreamSource<String> dataStreamSource = env.fromElements("Hadoop","Spark","Tomcat","Storm","Flink","Docker","Hive","Sqoop","Flink","Sqoop","Flink" );

        SingleOutputStreamOperator<Tuple2<String, Integer>> streamOperator = dataStreamSource.map(str -> Tuple2.of(str, 1)).returns(Types.TUPLE(Types.STRING,Types.INT));
        //keyBy() Tuple元组以下标的形式,进行分组
        KeyedStream<Tuple2<String, Integer>, Tuple> keyedStream = streamOperator.keyBy(0);
        //对分组后的数据进行 reduce() 操作
        //old,news 为两个 Tuple2<String,Integer>类型(通过f0,f1可以获得相对应下标的值)
        SingleOutputStreamOperator<Tuple2<String, Integer>> resultStream = keyedStream.reduce((old, news) -> {
            old.f1 += news.f1;
            return old;
        }).returns(Types.TUPLE(Types.STRING, Types.INT));

        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream");
    }

}
```

### sum、min、minBy、max、maxBy aggregations
Aggregations是DataStream接口提供的聚合算子，根据指定的字段进行聚合操作，滚动地产生一系列数据聚合结果。其实是将Reduce算子中的函数进行了封装，封装的聚合操作有sum、min、minBy、max、maxBy等，这样就不需要用户自己定义Reduce函数。
类型转换：KeyedStream → DataStream

sum()：求和
min()：返回最小值 max():返回最大值(指定的field是最小，但不是最小的那条记录)
minBy(): 返回最小值的元素 maxBy(): 返回最大值的元素(获取的最小值，同时也是最小值的那条记录)

指定数据集中第一个字段作为key，用第二个字段作为累加字段，然后滚动地对第二个字段的数值进行累加并输出。
```java
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //通过Socket实时获取数据
        DataStreamSource<String> dataStreamSource = env.fromElements("Hadoop","Spark","Tomcat","Storm","Flink","Docker","Hive","Sqoop","Flink","Sqoop","Flink" );

        SingleOutputStreamOperator<Tuple2<String, Integer>> streamOperator = dataStreamSource.map(str -> Tuple2.of(str, 1)).returns(Types.TUPLE(Types.STRING,Types.INT));
        //keyBy() Tuple元组以下标的形式,进行分组
        KeyedStream<Tuple2<String, Integer>, Tuple> keyedStream = streamOperator.keyBy(0);
        //对分组后的数据进行 reduce() 操作
        //old,news 为两个 Tuple2<String,Integer>类型(通过f0,f1可以获得相对应下标的值)
        SingleOutputStreamOperator<Tuple2<String, Integer>> resultStream = keyedStream.sum(1);

        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream");
    }

}
```

聚合函数中需要传入的字段类型必须是数值型，否则会抛出异常。对应其他的聚合函数的用法如下代码所示。
```
    SingleOutputStreamOperator<Tuple2<String, Integer>> min = keyedStream.min(1);
    SingleOutputStreamOperator<Tuple2<String, Integer>> max = keyedStream.max(1);
    SingleOutputStreamOperator<Tuple2<String, Integer>> minBy = keyedStream.minBy(1);
    SingleOutputStreamOperator<Tuple2<String, Integer>> maxBy = keyedStream.maxBy(1);
```

### fold
一个有初始值的分组数据流的滚动折叠操作。合并当前元素和前一次折叠操作的结果，并产生一个新的值。
类型转换：KeyedStream → DataStream

fold() 方法只是对分组中的数据进行折叠操作。比如有 3 个分组，然后我们通过如下代码来完成对分组中数据的折叠操作。分组如下：
组1：【11,22,33,44,55】
组2：【88】
组3：【98,99】
```java
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<Integer> source = env.fromElements(11, 11, 11, 22, 33, 44, 55);

        SingleOutputStreamOperator<Tuple2<Integer, Integer>> streamOperator = source.map(num -> Tuple2.of(num, 1)).returns(Types.TUPLE(Types.INT, Types.INT));

        KeyedStream<Tuple2<Integer, Integer>, Tuple> keyedStream = streamOperator.keyBy(0);

        DataStream<String> resultStream = keyedStream.fold("start", (current, tuple) -> current + "-" + tuple.f0).returns(Types.STRING);

        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream");
    }

}
```

### union
在 DataStream 上使用 union 算子可以合并多个同类型的数据流，并生成同类型的新数据流，即可以将多个 DataStream 合并为一个新的 DataStream。数据将按照先进先出（First In First Out） 的模式合并，且不去重。
类型转换：DataStream → DataStream

将创建的数据集source1和source2合并，如果想将多个数据集同时合并则在union()方法中传入被合并的数据集的序列即可。
```java
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<String> source1 = env.fromElements("Hello", "World", "Flink");
        DataStreamSource<String> source2 = env.fromElements("Hello", "World", "Spark");
        DataStream<String> resultStream = source1.union(source2);
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream");
    }

}
```

### connect,comap,coflatmap
connect 提供了和 union 类似的功能，用来连接两个数据流
类型转换：DataStream,DataStream → ConnectedStreams

它与union的区别在于：
- connect 只能连接两个数据流，union 可以连接多个数据流；
- connect 所连接的两个数据流的数据类型可以不一致，union所连接的两个数据流的数据类型必须一致。
  两个DataStream 经过 connect 之后被转化为 ConnectedStreams，ConnectedStreams 会对两个流的数据应用不同的处理方法，且双流之间可以共享状态。

对于 ConnectedStreams,我们需要重写CoMapFunction或CoFlatMapFunction。 这两个接口都提供了三个参数，这三个参数分别对应第一个输入流的数据类型、第二个输入流的数据类型和输出流的数据类型。

在重写方法时，都提供了两个方法（map1/map2 或 flatMap1/flatMap2）。在重写时，对于CoMapFunction，map1处理第一个流的数据，map2处理第二个流的数据；对于CoFlatMapFunction，flatMap1处理第一个流的数据，flatMap2处理第二个流的数据。Flink并不能保证两个函数调用顺序，两个函数的调用依赖于两个数据流数据的流入先后顺序，即第一个数据流有数据到达时，map1或flatMap1会被调用，第二个数据流有数据到达时，map2或flatMap2会被调用。

场景： 使用 connect() 方法，来完成对一个字符串流 和 一个整数流进行connect操作，并对流中出现的词进行计数操作
```java
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.ConnectedStreams;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.co.CoMapFunction;

public class SteamWordCount {


    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStreamSource<String> streamSource01 = env.fromElements("aa", "bb", "aa", "dd", "dd");

        DataStreamSource<Integer> streamSource02 = env.fromElements(11,22,33,22,11);

        ConnectedStreams<String, Integer> connectedStream = streamSource01.connect(streamSource02);

        SingleOutputStreamOperator<Tuple2<String, Integer>> outputStreamOperator = connectedStream.map(new CoMapFunction<String, Integer, Tuple2<String, Integer>>() {

            //处理第一个流数据
            @Override
            public Tuple2<String, Integer> map1(String str) throws Exception {
                return Tuple2.of(str,1);
            }
            //处理第二个流数据
            @Override
            public Tuple2<String, Integer> map2(Integer num) throws Exception {
                return Tuple2.of(String.valueOf(num),1);
            }
        });

        SingleOutputStreamOperator<Tuple2<String, Integer>> sum = outputStreamOperator.keyBy(0).sum(1);

        sum.print();

        env.execute("Stream");
    }
}
```

### split
按照指定标准，将指定的 DataStream流拆分成多个流，用SplitStream来表示。
类型转换：DataStream → SplitStream

将输入的元素，按照奇数/偶数分成两种流。
```java
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.collector.selector.OutputSelector;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.datastream.SplitStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

import java.util.ArrayList;
import java.util.List;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        DataStreamSource<Integer> streamSource = env.fromElements(1, 2, 3, 4, 5, 6, 7, 8, 9);

        SplitStream<Integer> splitStream = streamSource.split(new OutputSelector<Integer>() {
            @Override
            public Iterable<String> select(Integer value) {
                List<String> output = new ArrayList<String>();
                if (value % 2 == 0) {
                    output.add("even");
                } else {
                    output.add("odd");
                }
                return output;
            }
        });

        SingleOutputStreamOperator<Tuple2<Integer, Integer>> resultStream = splitStream.select("odd").map(num -> Tuple2.of(num, 1)).returns(Types.TUPLE(Types.INT, Types.INT));
        // 指定计算结果输出位置
        resultStream.print();
        // 指定名称并触发流式任务
        env.execute("Stream");
    }

}
```

### select
split函数本身只是对输入数据集进行标记，并没有将数据集真正的实现切分，因此需要借助Select函数根据标记将数据切分成不同的数据集。
类型转换：SplitStream → DataStream

通过调用SplitStream数据集的select()方法，传入前面已经标记好的标签信息，然后将符合条件的数据筛选出来，形成新的数据集。
```java
import org.apache.flink.streaming.api.collector.selector.OutputSelector;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SplitStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

import java.util.ArrayList;
import java.util.List;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        DataStreamSource<Integer> streamSource = env.fromElements(1, 2, 3, 4, 5, 6, 7, 8, 9);

        SplitStream<Integer> splitStream = streamSource.split(new OutputSelector<Integer>() {
            @Override
            public Iterable<String> select(Integer value) {
                List<String> output = new ArrayList<String>();
                if (value % 2 == 0) {
                    output.add("even");
                } else {
                    output.add("odd");
                }
                return output;
            }
        });
        //筛选出奇数数据集
        splitStream.select("odd").print();
        //筛选出偶数数据集
//        splitStream.select("even").print();
        //筛选出奇数和偶数数据集
//        splitStream.select("even","odd").print();

        // 指定名称并触发流式任务
        env.execute("Stream");
    }

}
```

### iterate
Iterate算子适合于迭代计算场景，通过每一次的迭代计算，并将计算结果反馈到下一次迭代计算中。
类型转换：DataStream→IterativeStream→DataStream

如下代码所示
```java
import org.apache.flink.api.common.functions.FilterFunction;
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.IterativeStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

public class SteamWordCount {
    
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStreamSource<String> streamSource01 = env.fromElements("Hello World Flink Hello World Spark");

        // 做分词
        DataStream<Tuple2<String, Integer>> flatStream1 = streamSource01.flatMap(new FlatMapFunction<String, Tuple2<String, Integer>>() {
            public void flatMap(String s, Collector<Tuple2<String, Integer>> collector) throws Exception {
                String[] arr = s.split(" ");
                for (String item : arr) {
                    collector.collect(new Tuple2<>(item, 1));
                }
            }
        });

        IterativeStream<Tuple2<String, Integer>> iteration1 = flatStream1.iterate(5 * 60 * 1000); //迭代头
        DataStream<Tuple2<String, Integer>> iteratedStream1 = iteration1.filter((FilterFunction<Tuple2<String, Integer>>) tuple2 ->
                !tuple2.f0.isEmpty()
        ).keyBy(0).sum(1);
        DataStream<Tuple2<String, Integer>> feedbackStream1 = iteratedStream1.filter((FilterFunction<Tuple2<String, Integer>>) stringIntegerTuple2 -> !stringIntegerTuple2.f0.equals("Spark"));
        iteration1.closeWith(feedbackStream1);

        iteratedStream1.print();

        env.execute("Stream");
    }
}
```
## 用户自定义函数（UDF）

### 函数类
对于大部分操作而言，都需要传入一个用户自定义函数，实现相关操作的接口。Flink暴露了所有UDF函数的接口，具体实现的方式为接口或者抽象类，如MapFunction、FilterFunction、ReduceFunction等。

Flink中提供了大量的函数供用户使用，例如以下代码通过定义MyMapFunction Class实现MapFunction接口，然后调用DataStream的map()方法将MyMapFunction实现类传入，完成对实现将数据集中字符串记录转换成大写的数据处理。
```java
  public  static  class myFlatMap implements FlatMapFunction<String, Tuple2<String,Integer>> {
        @Override
        public void flatMap(String s, Collector<Tuple2<String, Integer>> collector) throws Exception {
            String [] words=  s.split(" ");
            for ( String word: words) {
                collector.collect(new Tuple2<>(word,1));
            }
        }
    }
```

### 匿名函数（Lambda表达式）
Flink的所有算子都可以适应Lambda表达式的方式来进行编码，但当Lambda表达式使用Java的泛型时，我们需要显示的声明类型信息，使用returns(new TypeHint<Tuple2<Integer, SomeType>>(){})

直接在map()方法中创建匿名实现类的方式定义函数计算逻辑。
```
inputDataStream.flatMap(new FlatMapFunction<String, Tuple2<String, Integer>>() {
                            @Override
                            public void flatMap(String s, Collector<Tuple2<String, Integer>> collector) throws Exception {
                                String[] words = s.split(" ");
                                for (String word : words) {
                                    collector.collect(new Tuple2<>(word, 1));
                                }
                            }
                        })
```

### 富函数类（Rich Function Classes）
所有Flink函数类都有其Rich版本。富函数类一般是以抽象类的形式出现，如：RichMapFunction、RichFilterFunction、 RichReduceFunction 等。

富函数类有比常规的函数类提供更多、更丰富的功能，可以获取运行环境的上下文，并拥有一些生命周期方法。

open()方法：Rich Function的初始化方法，开启一个算子的生命周期，当一个算子的实际工作方法如map()或者filter()方法被调用之前，open()会首先被调用。像文件IO的创建、数据库连接的创建、配置文件的读取等这样一次性的工作，都适合在open()方法中完成
close()方法：生命周期中的最后一个调用的方法
另外，富函数类提供了getRuntimeContext()方法，可以获取到运行时上下文的一些信息，例如程序执行的并行度、任务名称、状态。
```java
import org.apache.flink.api.common.functions.RichMapFunction;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class TransformRichFunctionTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        //从元素中读取数据
        DataStreamSource<Event> stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 2000L),
                new Event("Alice", "./prod?id=100", 3000L),
                new Event("Bob", "./prod?id=1", 3300L),
                new Event("Alice", "./prod?id=200", 3200L),
                new Event("Bob", "./home", 3500L),
                new Event("Bob", "./prod?id=2", 3800L),
                new Event("Bob", "./prod?id=3", 4200L)
        );
        stream.map(new MyRichMapper()).print();


        env.execute();
    }

    //实现一个自定义的富函数类
    private static class MyRichMapper extends RichMapFunction<Event,Integer>{

        @Override
        public void open(Configuration parameters) throws Exception {
            super.open(parameters);
            System.out.println("open生命周期被调用 " + getRuntimeContext().getIndexOfThisSubtask() + "号任务启动");
        }

        @Override
        public Integer map(Event value) throws Exception {
            return value.url.length();
        }

        @Override
        public void close() throws Exception {
            super.close();
            System.out.println("close生命周期被调用 " + getRuntimeContext().getIndexOfThisSubtask() + "号任务结束");
        }
    }
}
```

## 物理分区
物理分区（Physical Partitioning）操作的作用是根据指定的分区策略将数据重新分配到不同节点的Task案例上执行。当使用DataStream提供的API对数据处理过程中，依赖于算子本身对数据的分区控制，如果用户希望自己控制数据分区，例如当数据发生了数据倾斜的时候，就需要通过定义物理分区策略的方式对数据集进行重新分布处理。
Flink中已经提供了常见的分区策略，例如随机分区（Random Partitioning）、平衡分区（Roundobin Partitioning）、按比例分区（Roundrobin Partitioning）等。当然如果给定的分区策略无法满足需求，也可以根据Flink提供的分区控制接口创建分区器，实现自定义分区控制。

### 随机分区（Random）
最简单的重分区方式就是直接“洗牌”。通过调用 DataStream 的.shuffle()方法，将数据随机地分配到下游算子的并行任务中去。
类型转换：DataStream ->DataStream

```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {
    
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.socketTextStream("Flink",9999).shuffle()
                .print()
                .setParallelism(4);
        env.execute("Stream");
    }
}
```

### 分区元素循轮询(Round-robin)
分区元素循轮询，为每个分区创建相等的负载。有助于在数据不对称的情况下优化性能。在存在数据偏斜的情况下对性能优化有用。
类型转换：DataStream ->DataStream

```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {
    
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.socketTextStream("Flink",9999).rebalance()
                .print()
                .setParallelism(4);
        env.execute("Stream");
    }
}

```

### 重缩放分区（Rescaling）
该策略和RoundRobin分区策略一样，Rescaling分区策略通过一种循环方式将数据发送给下游的任务节点。但是不同点是，RoundRobin分区会将上游的数据全局性的以轮询的方式分发给下游节点。而Rescaling分区仅仅会对下游继承的算子进行负载均衡。例如上游并行度是2下游并行度是4，上有就会按照下游分区，等比例分发给下游的任务。
类型转换：DataStream ->DataStream

```java

import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {
    
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.socketTextStream("Flink",9999).rescale()
                .print()
                .setParallelism(4);
        env.execute("Stream");
    }
}

```

### 广播操作（Broadcasting）
广播策略将输入的数据集复制到下游算子的并行的Tasks实例中，下游算子中的Tasks可以直接从本地内存中获取广播数据集，不再依赖于网络传输。这种分区策略适合于小数据集，例如当大数据集关联小数据集时，可以通过广播的方式将小数据集分发到算子的每个分区中。
类型转换：DataStream ->DataStream

```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {
    
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.socketTextStream("Flink",9999).broadcast()
                .print()
                .setParallelism(4);
        env.execute("Stream");
    }
}
```

### 全局分区（global）
全局分区也是一种特殊的分区方式。这种做法非常极端，通过调用.global()方法，会将所有的输入流数据都发送到下游算子的第一个并行子任务中去。这就相当于强行让下游任务并行度变成了 1，所以使用这个操作需要非常谨慎，可能对程序造成很大的压力。

### 自定义分区（Custom Partitioning）
当Flink提供的分区策略都不适用时,我们可以使用partitionCustom()方法来自定义分区策略.这个方法接收一个Partitioner对象,这个对象需要实现分区逻辑以及定义针对流的哪一个字段或者key来进行分区.
类型转换：DataStream ->DataStream

```java
import org.apache.flink.api.common.functions.Partitioner;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class SteamWordCount {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        // 将自然数按照奇偶分区
        env.fromElements(1, 2, 3, 4, 5, 6, 7, 8).partitionCustom(new Partitioner<Integer>() {
            @Override
            public int partition(Integer key, int numPartitions) {
                return key % 2;
            }
        }, new KeySelector<Integer, Integer>() {
            @Override
            public Integer getKey(Integer value) throws Exception {
                return value;
            }
        }).print().setParallelism(2);
        env.execute("Stream");
    }
}

```

Flink 中改变并行度，默认RebalancePartitioner分区策略。 分区策略总结：
- BroadcastPartitioner广播分区会将上游数据输出到下游算子的每个实例()，适合于大数据和小数据集做JOIN场景。
- CustomPartitionerWrapper自定义分区需要用户根据自己实现Partitioner接口，来定义自己的分区逻辑。
- ForwarPartitioner用户将记录输出到下游本地的算子实例。它要求上下游算子并行度一样。简单的说，ForwarPartitioner可以来做控制台打印。
- GlobaPartitioner数据会被分发到下游算子的第一个实例中进行处理
- KeyGroupStreamPartitioner Hash分区器，会将数据按照key的Hash值输出到下游的实例中
- RebalancePartitioner数据会被循环发送到下游的每一个实例额的Task中进行处理。
- RescalePartitioner这种分区器会根据上下游算子的并行度，循环的方式输出到下游算子的每个实例。这里有点难理解，家核上游并行度为2，编号为A和B。下游并行度为4，编号为1,2,3,4.那么A则把数据循环发送给1,和2，B则把数据循环发送给3和4.
- ShufflePartitioner数据会被随即分发到下游算子的每一个实例中进行处理。



