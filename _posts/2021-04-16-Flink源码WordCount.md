---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink源码WordCount源码解析
WordCount是学习flink的第一个最简单的例子，但是想要深入的理解flink，还得理解程序的源码实现才行。现在我们就来分析一下它的源码。

## WordCount代码示例
```
object SocketWindowWordCount {

  /** Main program method */
  def main(args: Array[String]) : Unit = {

    // the host and the port to connect to
    var hostname: String = "10.18.34.198"
    var port: Int = 9999

    // get the execution environment
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(2)

    // get input data by connecting to the socket
    val text: DataStream[String] = env.socketTextStream(hostname, port)

    // parse the data, group it, window it, and aggregate the counts
    val windowCounts = text
      .flatMap { w => w.split("\\s") }
      .map { w => WordWithCount(w, 1) }
      .keyBy("word")
      .timeWindow(Time.seconds(60))
      .reduce(_ + _)

    // print the results with a single thread, rather than in parallel
    windowCounts
    .print()
    .setParallelism(1)

    env.execute("Socket Window WordCount")
  }

  /** Data type for words with count */
  case class WordWithCount(word: String, count: Long) {
    def + (o: WordWithCount): WordWithCount = {
      WordWithCount(word, count + o.count)
    }
  }
```
可以看到一个简单的程序包含了下面几个步骤：
- 创建数据源，并添加数据源env.addSource()
- 经过一系列的数据转换、窗口、聚合操作
- 添加sink，输出数据，本例中就是print()算子

上述代码是scala代码，但是scala代码基本都是对java代码的一个封装，所以以下的源码都是java的相关代码。

下面来逐步分析一下WordCount的源码。

## addSource
在Flink中，数据源的构建是通过StreamExecutionEnviroment的具体实现的实例来构建的，如上述代码中的这句代码
```
val text: DataStream[String] = env.socketTextStream(hostname, port)
```
这句代码就在指定的host和port上构建了一个接受网络数据的数据源，接下来看其内部如何实现的
```
public DataStreamSource<String> socketTextStream(String hostname, int port) {
   return socketTextStream(hostname, port, "\n");
}

public DataStreamSource<String> socketTextStream(String hostname, int port, String delimiter) {
   return socketTextStream(hostname, port, delimiter, 0);
}

public DataStreamSource<String> socketTextStream(String hostname, int port, String delimiter, long maxRetry) {
   return addSource(new SocketTextStreamFunction(hostname, port, delimiter, maxRetry),
         "Socket Stream");
}
```
通过对socketTextStream的重载方法的依次调用，可以看到会根据传入的hostname、port，以及默认的行分隔符”\n”，和最大尝试次数0，构造一个SocketTextStreamFunction实例，并采用默认的数据源节点名称为”Socket Stream”。

SocketTextStreamFunction的类是SourceFunction的一个子类，而SourceFunction是Flink中数据源的基础接口。
```
@Public
public interface SourceFunction<T> extends Function, Serializable {

   void run(SourceContext<T> ctx) throws Exception;

   void cancel();
   
   interface SourceContext<T> {

      void collect(T element);

      @PublicEvolving
      void collectWithTimestamp(T element, long timestamp);

      void emitWatermark(Watermark mark);

      @PublicEvolving
      void markAsTemporarilyIdle();

      Object getCheckpointLock();

      void close();
   }
}
```
从定义中可以看出，其主要有两个方法run和cancel
- run(SourceContex)方法：就是实现数据获取逻辑的地方，并可以通过传入的参数ctx进行向下游节点的数据转发。
- cancel()方法：则是用来取消数据源的数据产生，一般在run方法中，会存在一个循环来持续产生数据，而cancel方法则可以使得该循环终止。

其内部接口SourceContex则是用来进行数据发送的接口。了解了SourceFunction这个接口的功能后，来看下SocketTextStreamFunction的具体实现，也就是主要看其run方法的具体实现。
```
public void run(SourceContext<String> ctx) throws Exception {
   final StringBuilder buffer = new StringBuilder();
   long attempt = 0;
   //这里是第一层循环，只要当前处于运行状态，该循环就不会退出，会一直循环
   while (isRunning) {
      try (Socket socket = new Socket()) {
         //对指定的hostname和port，建立Socket连接，并构建一个BufferedReader，用来从Socket中读取数据
         currentSocket = socket;
         LOG.info("Connecting to server socket " + hostname + ':' + port);
         socket.connect(new InetSocketAddress(hostname, port), CONNECTION_TIMEOUT_TIME);
         BufferedReader reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));
         char[] cbuf = new char[8192];
         int bytesRead;
         //这里是第二层循环，对运行状态进行了双重校验，同时对从Socket中读取的字节数进行判断
         while (isRunning && (bytesRead = reader.read(cbuf)) != -1) {
            buffer.append(cbuf, 0, bytesRead);
            int delimPos;
            //这里是第三层循环，就是对从Socket中读取到的数据，按行分隔符进行分割，并将每行数据作为一个整体字符串向下游转发
            while (buffer.length() >= delimiter.length() && (delimPos = buffer.indexOf(delimiter)) != -1) {
               String record = buffer.substring(0, delimPos);
               if (delimiter.equals("\n") && record.endsWith("\r")) {
                  record = record.substring(0, record.length() - 1);
               }
               //用入参ctx，进行数据的转发
               ctx.collect(record);
               buffer.delete(0, delimPos + delimiter.length());
            }
         }
      }
      //如果由于遇到EOF字符，导致从循环中退出，则根据运行状态，以及设置的最大重试尝试次数，决定是否进行 sleep and retry，或者直接退出循环
      if (isRunning) {
         attempt++;
         if (maxNumRetries == -1 || attempt < maxNumRetries) {
            LOG.warn("Lost connection to server socket. Retrying in " + delayBetweenRetries + " msecs...");
            Thread.sleep(delayBetweenRetries);
         }
         else {
            break;
         }
      }
   }
   //在最外层的循环都退出后，最后检查下缓存中是否还有数据，如果有，则向下游转发
   if (buffer.length() > 0) {
      ctx.collect(buffer.toString());
   }
}
```
run方法的逻辑如上，逻辑很清晰，就是从指定的hostname和port持续不断的读取数据，按行分隔符划分成一个个字符串，然后转发到下游。

cancel方法的实现如下，就是将运行状态的标识isRunning属性设置为false，并根据需要关闭当前socket。
```
public void cancel() {
   isRunning = false;
   Socket theSocket = this.currentSocket;
   //如果当前socket不为null，则进行关闭操作
   if (theSocket != null) {
      IOUtils.closeSocket(theSocket);
   }
}
```
对SocketTextStreamFunction的实现清楚之后，回到StreamExecutionEnvironment中，看addSource方法。
```
public <OUT> DataStreamSource<OUT> addSource(SourceFunction<OUT> function, String sourceName) {
   return addSource(function, sourceName, null);
}

public <OUT> DataStreamSource<OUT> addSource(SourceFunction<OUT> function, String sourceName, TypeInformation<OUT> typeInfo) {
   //如果传入的输出数据类型信息为null，则尝试提取输出数据的类型信息
   if (typeInfo == null) {
      if (function instanceof ResultTypeQueryable) {
         //如果传入的function实现了ResultTypeQueryable接口, 则直接通过接口获取
         typeInfo = ((ResultTypeQueryable<OUT>) function).getProducedType();
      } else {
         try {
            //通过反射机制来提取类型信息
            typeInfo = TypeExtractor.createTypeInfo(
                  SourceFunction.class,
                  function.getClass(), 0, null, null);
         } catch (final InvalidTypesException e) {
            //提取失败, 则返回一个MissingTypeInfo实例
            typeInfo = (TypeInformation<OUT>) new MissingTypeInfo(sourceName, e);
         }
      }
   }
   //根据function是否是ParallelSourceFunction的子类实例来判断是否是一个并行数据源节点
   boolean isParallel = function instanceof ParallelSourceFunction;
   //闭包清理, 可减少序列化内容, 以及防止序列化出错
   clean(function);
   StreamSource<OUT, ?> sourceOperator;
   //根据function是否是StoppableFunction的子类实例, 来决定构建不同的StreamOperator
   if (function instanceof StoppableFunction) {
      sourceOperator = new StoppableStreamSource<>(cast2StoppableSourceFunction(function));
   } else {
      sourceOperator = new StreamSource<>(function);
   }
   //返回一个新构建的DataStreamSource实例
   return new DataStreamSource<>(this, typeInfo, sourceOperator, isParallel, sourceName);
}
```
代码的核心就是构建sourceOperator，sourceOperator是StreamOperator的一个实例，并根据sourceOperator构建一个新的DataStreamSource，DataStreamSource也是DataStream的一个实现类。这里的SocketTextStreamFunction是一个非并行的函数，所以isParallel=false。

## flatMap
接下来看flatMap算子。
```
public <R> SingleOutputStreamOperator<R> flatMap(FlatMapFunction<T, R> flatMapper) {

   TypeInformation<R> outType = TypeExtractor.getFlatMapReturnTypes(clean(flatMapper),
         getType(), Utils.getCallLocationName(), true);

   return transform("Flat Map", outType, new StreamFlatMap<>(clean(flatMapper)));

}
@PublicEvolving
public <R> SingleOutputStreamOperator<R> transform(String operatorName, TypeInformation<R> outTypeInfo, OneInputStreamOperator<T, R> operator) {

   // read the output type of the input Transform to coax out errors about MissingTypeInfo
   transformation.getOutputType();

   OneInputTransformation<T, R> resultTransform = new OneInputTransformation<>(
         this.transformation,
         operatorName,
         operator,
         outTypeInfo,
         environment.getParallelism());

   @SuppressWarnings({ "unchecked", "rawtypes" })
   SingleOutputStreamOperator<R> returnStream = new SingleOutputStreamOperator(environment, resultTransform);

   getExecutionEnvironment().addOperator(resultTransform);

   return returnStream;
}
@Internal
public void addOperator(StreamTransformation<?> transformation) {
   Preconditions.checkNotNull(transformation, "transformation must not be null.");
   this.transformations.add(transformation);
}
```
在介绍DataStream的结构时介绍过transform算子。

- flatMap算子会根据用户的flatMapFunction生成一个StreamFlatMap的operator，然后根据operator创建DataStream的StreamTransformation，这里的StreamTransform就是一个OneInputTransformation，它持有DataStreamSource的StreamTransformation作为input。
- 接着根据StreamTransformation创建一个新的DataStream，这个新的DataStream就是SingleOutputStreamOperator
- 将StreamTransformation添加到StreamExecutionEnvironment的transformations列表中，用于以后生成StreamGraph使用

由此可知，flatMap算子生成的是一个新的DataStream，这个新的DataStream就是SingleOutputStreamOperator

## map
```
public <R> SingleOutputStreamOperator<R> map(MapFunction<T, R> mapper) {

   TypeInformation<R> outType = TypeExtractor.getMapReturnTypes(clean(mapper), getType(),
         Utils.getCallLocationName(), true);

   return transform("Map", outType, new StreamMap<>(clean(mapper)));
}
```
map算子的实现和flatMap算子相似，不同的是map算子的operator是StreamMap

## keyBy
```
public KeyedStream<T, Tuple> keyBy(String... fields) {
   return keyBy(new Keys.ExpressionKeys<>(fields, getType()));
}

private KeyedStream<T, Tuple> keyBy(Keys<T> keys) {
   return new KeyedStream<>(this, clean(KeySelectorUtil.getSelectorForKeys(keys,
         getType(), getExecutionConfig())));
}
public KeyedStream(DataStream<T> dataStream, KeySelector<T, KEY> keySelector) {
   this(dataStream, keySelector, TypeExtractor.getKeySelectorTypes(keySelector, dataStream.getType()));
}
public KeyedStream(DataStream<T> dataStream, KeySelector<T, KEY> keySelector, TypeInformation<KEY> keyType) {
   this(
      dataStream,
      new PartitionTransformation<>(
         dataStream.getTransformation(),
         new KeyGroupStreamPartitioner<>(keySelector, StreamGraphGenerator.DEFAULT_LOWER_BOUND_MAX_PARALLELISM)),
      keySelector,
      keyType);
}
@Internal
KeyedStream(
   DataStream<T> stream,
   PartitionTransformation<T> partitionTransformation,
   KeySelector<T, KEY> keySelector,
   TypeInformation<KEY> keyType) {

   super(stream.getExecutionEnvironment(), partitionTransformation);
   this.keySelector = clean(keySelector);
   this.keyType = validateKeyType(keyType);
}
```
keyBy算子会生成一个KeyedStream，这也是DataStream的一个实例，它的StreamTransform是partitionTransformation，partitionTransformation中包含一个KeyGroupStreamPartitioner，他负责定义key的分区规则。但是partitionTransformation并没有添加到StreamExecutionEnvironment的transformations列表中

下面来看看KeyGroupStreamPartitioner是如何进行key的分区的
```
public interface ChannelSelector<T extends IOReadableWritable> {
    
   void setup(int numberOfChannels);

   int selectChannel(T record);

   boolean isBroadcast();
}
@Internal
public class KeyGroupStreamPartitioner<T, K> extends StreamPartitioner<T> implements ConfigurableStreamPartitioner {
   private static final long serialVersionUID = 1L;

   private final KeySelector<T, K> keySelector;

   private int maxParallelism;

   public KeyGroupStreamPartitioner(KeySelector<T, K> keySelector, int maxParallelism) {
      Preconditions.checkArgument(maxParallelism > 0, "Number of key-groups must be > 0!");
      this.keySelector = Preconditions.checkNotNull(keySelector);
      this.maxParallelism = maxParallelism;
   }

   public int getMaxParallelism() {
      return maxParallelism;
   }

   @Override
   public int selectChannel(SerializationDelegate<StreamRecord<T>> record) {
      K key;
      try {
         key = keySelector.getKey(record.getInstance().getValue());
      } catch (Exception e) {
         throw new RuntimeException("Could not extract key from " + record.getInstance().getValue(), e);
      }
      return KeyGroupRangeAssignment.assignKeyToParallelOperator(key, maxParallelism, numberOfChannels);
   }
public static int assignKeyToParallelOperator(Object key, int maxParallelism, int parallelism) {
   return computeOperatorIndexForKeyGroup(maxParallelism, parallelism, assignToKeyGroup(key, maxParallelism));
}

public static int assignToKeyGroup(Object key, int maxParallelism) {
   return computeKeyGroupForKeyHash(key.hashCode(), maxParallelism);
}

public static int computeKeyGroupForKeyHash(int keyHash, int maxParallelism) {
   return MathUtils.murmurHash(keyHash) % maxParallelism;
}

public static int computeOperatorIndexForKeyGroup(int maxParallelism, int parallelism, int keyGroupId) {
   return keyGroupId * parallelism / maxParallelism;
}
```
- KeyGroupStreamPartitioner实现了ChannelSelector接口，ChannelSelector.selectChannel决定了数据将发往下游DataStream的哪个task（也可以理解为哪个数据分区）。
- KeyGroupStreamPartitioner.selectChannel()实现， 先通过key的hashCode，算出maxParallelism的余数，也就是可以得到一个[0, maxParallelism)的整数；
- 在通过公式 keyGroupId * parallelism / maxParallelism ，计算出一个[0, parallelism)区间的整数，从而实现分区功能

## timeWindow
```
public WindowedStream<T, KEY, TimeWindow> timeWindow(Time size) {
   if (environment.getStreamTimeCharacteristic() == TimeCharacteristic.ProcessingTime) {
      return window(TumblingProcessingTimeWindows.of(size));
   } else {
      return window(TumblingEventTimeWindows.of(size));
   }
}

@PublicEvolving
public <W extends Window> WindowedStream<T, KEY, W> window(WindowAssigner<? super T, W> assigner) {
   return new WindowedStream<>(this, assigner);
}
```
timeWindow算子返回一个新的WindowedStream，不过这个WindowedStream并不是DataStream的子类。WindowedStream描述一个数据流中的元素会基于key进行分组，并且对于每个key，对应的元素会被划分到多个时间窗口内。然后窗口会基于触发器，将对应窗口中的数据转发到下游节点。

WindowedStream包含一个WindowAssigner，WindowAssigner负责为接收到的数据分配所在的窗口。我们的程序里使用的是默认的ProcessingTime，所以这个WindowAssigner是TumblingProcessingTimeWindows。窗口分配方法在assignWindows
```
public static TumblingProcessingTimeWindows of(Time size) {
   return new TumblingProcessingTimeWindows(size.toMilliseconds(), 0);
}

public class TumblingProcessingTimeWindows extends WindowAssigner<Object, TimeWindow> {
   private static final long serialVersionUID = 1L;

   private final long size;

   private final long offset;

   private TumblingProcessingTimeWindows(long size, long offset) {
      if (Math.abs(offset) >= size) {
         throw new IllegalArgumentException("TumblingProcessingTimeWindows parameters must satisfy abs(offset) < size");
      }

      this.size = size;
      this.offset = offset;
   }

//为数据分配窗口
   @Override
   public Collection<TimeWindow> assignWindows(Object element, long timestamp, WindowAssignerContext context) {
      final long now = context.getCurrentProcessingTime();
      long start = TimeWindow.getWindowStartWithOffset(now, offset, size);
      return Collections.singletonList(new TimeWindow(start, start + size));
   }
```
可见在滚动窗口的实现中，数据只能在一个窗口中。

## reduce
```
public SingleOutputStreamOperator<T> reduce(ReduceFunction<T> function) {
   if (function instanceof RichFunction) {
      throw new UnsupportedOperationException("ReduceFunction of reduce can not be a RichFunction. " +
         "Please use reduce(ReduceFunction, WindowFunction) instead.");
   }

   //clean the closure
   //默认会传递一个PassThroughWindowFunction当做窗口函数，它的作用就是收集每个key的状态数据发送到下游
   function = input.getExecutionEnvironment().clean(function);
   return reduce(function, new PassThroughWindowFunction<K, W, T>());
}

public <R> SingleOutputStreamOperator<R> reduce(
      ReduceFunction<T> reduceFunction,
      WindowFunction<T, R, K, W> function) {

   TypeInformation<T> inType = input.getType();
   TypeInformation<R> resultType = getWindowFunctionReturnType(function, inType);
   return reduce(reduceFunction, function, resultType);
}
//返回一个新的DataStream，为SingleOutputStreamOperator
public <R> SingleOutputStreamOperator<R> reduce(
      ReduceFunction<T> reduceFunction,
      WindowFunction<T, R, K, W> function,
      TypeInformation<R> resultType) {

   if (reduceFunction instanceof RichFunction) {
      throw new UnsupportedOperationException("ReduceFunction of reduce can not be a RichFunction.");
   }

   //clean the closures
   function = input.getExecutionEnvironment().clean(function);
   reduceFunction = input.getExecutionEnvironment().clean(reduceFunction);

   final String opName = generateOperatorName(windowAssigner, trigger, evictor, reduceFunction, function);
   KeySelector<T, K> keySel = input.getKeySelector();

   OneInputStreamOperator<T, R> operator;

   if (evictor != null) {
      @SuppressWarnings({"unchecked", "rawtypes"})
      TypeSerializer<StreamRecord<T>> streamRecordSerializer =
         (TypeSerializer<StreamRecord<T>>) new StreamElementSerializer(input.getType().createSerializer(getExecutionEnvironment().getConfig()));

      ListStateDescriptor<StreamRecord<T>> stateDesc =
         new ListStateDescriptor<>("window-contents", streamRecordSerializer);

      operator =
         new EvictingWindowOperator<>(windowAssigner,
            windowAssigner.getWindowSerializer(getExecutionEnvironment().getConfig()),
            keySel,
            input.getKeyType().createSerializer(getExecutionEnvironment().getConfig()),
            stateDesc,
            new InternalIterableWindowFunction<>(new ReduceApplyWindowFunction<>(reduceFunction, function)),
            trigger,
            evictor,
            allowedLateness,
            lateDataOutputTag);

   } else {
       //代码会执行到这一步，创建ReducingStateDescriptor，持有reduceFunction，
       //会对进入窗口的数据调用reduceFunction和状态数据进行聚合
      ReducingStateDescriptor<T> stateDesc = new ReducingStateDescriptor<>("window-contents",
         reduceFunction,
         input.getType().createSerializer(getExecutionEnvironment().getConfig()));
         
      operator =
         new WindowOperator<>(windowAssigner,
            windowAssigner.getWindowSerializer(getExecutionEnvironment().getConfig()),
            keySel,
            input.getKeyType().createSerializer(getExecutionEnvironment().getConfig()),
            stateDesc,
            new InternalSingleValueWindowFunction<>(function),
            trigger,
            allowedLateness,
            lateDataOutputTag);
   }
    //返回一个新的DataStream，operator为WindowOperator
   return input.transform(opName, resultType, operator);
}

@Override
@PublicEvolving
public <R> SingleOutputStreamOperator<R> transform(String operatorName,
      TypeInformation<R> outTypeInfo, OneInputStreamOperator<T, R> operator) {

   SingleOutputStreamOperator<R> returnStream = super.transform(operatorName, outTypeInfo, operator);

   // inject the key selector and key type
   //跟DataStream相比方法多了设置keySelector
   OneInputTransformation<T, R> transform = (OneInputTransformation<T, R>) returnStream.getTransformation();
   transform.setStateKeySelector(keySelector);
   transform.setStateKeyType(keyType);

   return returnStream;
}
```
reduce算子会返回一个新的DataStream，是一个SingleOutputStreamOperator，会给这个SingleOutputStreamOperator设置keySelector，同时这个DataStream的operator是WindowOperator，WindowOperator中包含了一个ReducingStateDescriptor会对进入窗口的数据和状态数据做聚合操作。WindowOperator是窗口操作的核心实现

## print
```
@PublicEvolving
public DataStreamSink<T> print() {
   PrintSinkFunction<T> printFunction = new PrintSinkFunction<>();
   return addSink(printFunction).name("Print to Std. Out");
}

public DataStreamSink<T> addSink(SinkFunction<T> sinkFunction) {

   // read the output type of the input Transform to coax out errors about MissingTypeInfo
   transformation.getOutputType();

   // configure the type if needed
   if (sinkFunction instanceof InputTypeConfigurable) {
      ((InputTypeConfigurable) sinkFunction).setInputType(getType(), getExecutionConfig());
   }

   StreamSink<T> sinkOperator = new StreamSink<>(clean(sinkFunction));

   DataStreamSink<T> sink = new DataStreamSink<>(this, sinkOperator);

   getExecutionEnvironment().addOperator(sink.getTransformation());
   return sink;
}

@Public
public class DataStreamSink<T> {

   private final SinkTransformation<T> transformation;

   @SuppressWarnings("unchecked")
   protected DataStreamSink(DataStream<T> inputStream, StreamSink<T> operator) {
      this.transformation = new SinkTransformation<T>(inputStream.getTransformation(), "Unnamed", operator, inputStream.getExecutionEnvironment().getParallelism());
   }
   ...
```
print操作也是数据输出的步骤，也就是addSink操作。期间会生成一个StreamSink的operator，它包含了一个sinkFunction，在介绍StreamOperator的时候提到过，StreamSink主要就是调用SinkFunction.invoke()对输入到sink的数据进行处理。在这里就是PrintSinkFunction，负责将数据结果打印到控制台。

DataStreamSink并不是一个DataStream，而是单独的一个数据结构，他里面也包含有StreamTransform，是SinkTransformation，这个SinkTransformation也会添加到StreamExecutionEnvironment的transformations中，用于构建StreamGraph。

经过上述步骤，就完成了数据流的源构造、数据流的转换操作、数据流的Sink构造，在这个过程中，每次转换都会产生一个新的数据流，而每个数据流下几乎都有一个StreamTransformation的子类实例，对于像flatMap、reduce这些转换得到的数据流里的StreamTransformation会被添加到StreamExecutionEnvironment的属性transformations这个列表中，这个属性在后续构建StreamGraph时会使用到。

另外在这个数据流的构建与转换过程中，每个DataStream中的StreamTransformation的具体子类中都有一个input属性，该属性会记录该StreamTransformation的上游的DataStream的StreamTransformation引用，从而使得整个数据流中的StreamTransformation构成了一个隐式的链表，由于一个数据流可能会转换成多个输出数据流，以及多个输入数据流又有可能会合并成一个输出数据流，确定的说，不是隐式列表，而是一张隐式的图。

## execute
上述数据转换完成后，就会进行任务的执行，就是执行如下代码：
```
env.execute("Socket Window WordCount");
```
这里就会根据上述的转换过程，先生成StreamGraph，再根据StreamGraph生成JobGraph，然后通过客户端提交到集群进行调度执行。








