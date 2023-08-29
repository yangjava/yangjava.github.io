---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink源码StreamGraph构建过程
env.execute将进行任务的提交和执行，在执行之前会对任务进行StreamGraph和JobGraph的构建，然后再提交JobGraph。

## StreamGraph构建过程
那么现在就来分析一下StreamGraph的构建过程，在正式环境下，会调用StreamContextEnvironment.execute()方法：
```
public JobExecutionResult execute(String jobName) throws Exception {
   Preconditions.checkNotNull(jobName, "Streaming Job name should not be null.");
    //构建StreamGraph
   StreamGraph streamGraph = this.getStreamGraph();
   streamGraph.setJobName(jobName);
    //构建完StreamGraph之后，可以清空transformations列表
   transformations.clear();

   // execute the programs
   if (ctx instanceof DetachedEnvironment) {
      LOG.warn("Job was executed in detached mode, the results will be available on completion.");
      ((DetachedEnvironment) ctx).setDetachedPlan(streamGraph);
      return DetachedEnvironment.DetachedJobExecutionResult.INSTANCE;
   } else {
      return ctx
         .getClient()
         .run(streamGraph, ctx.getJars(), ctx.getClasspaths(), ctx.getUserCodeClassLoader(), ctx.getSavepointRestoreSettings())
         .getJobExecutionResult();
   }
}
```

## 构建的起点
可以看到StreamGraph的主要构建方法在getStreamGraph()。
```
@Internal
public StreamGraph getStreamGraph() {
   if (transformations.size() <= 0) {
      throw new IllegalStateException("No operators defined in streaming topology. Cannot execute.");
   }
   return StreamGraphGenerator.generate(this, transformations);
}
```
实现在StreamGraphGenerator.generate(this, transformations);这里的transformations列表就是在之前调用map、flatMap、filter算子时添加进去的生成的DataStream的StreamTransformation，是DataStream的描述信息。

接着看：
```
public static StreamGraph generate(StreamExecutionEnvironment env, List<StreamTransformation<?>> transformations) {
   return new StreamGraphGenerator(env).generateInternal(transformations);
}
```
这里首先构造了一个StreamGraphGenerator，然后调用generateInternal对StreamGraph进行构造。

先看StreamGraphGenerator
```
@Internal
public class StreamGraphGenerator {

   private static final Logger LOG = LoggerFactory.getLogger(StreamGraphGenerator.class);

   public static final int DEFAULT_LOWER_BOUND_MAX_PARALLELISM = KeyGroupRangeAssignment.DEFAULT_LOWER_BOUND_MAX_PARALLELISM;
   public static final int UPPER_BOUND_MAX_PARALLELISM = KeyGroupRangeAssignment.UPPER_BOUND_MAX_PARALLELISM;

   // The StreamGraph that is being built, this is initialized at the beginning.
   private final StreamGraph streamGraph;

   private final StreamExecutionEnvironment env;

   // This is used to assign a unique ID to iteration source/sink
   protected static Integer iterationIdCounter = 0;
   public static int getNewIterationNodeId() {
      iterationIdCounter--;
      return iterationIdCounter;
   }

   // Keep track of which Transforms we have already transformed, this is necessary because
   // we have loops, i.e. feedback edges.
   private Map<StreamTransformation<?>, Collection<Integer>> alreadyTransformed;


   /**
    * Private constructor. The generator should only be invoked using {@link #generate}.
    */
   private StreamGraphGenerator(StreamExecutionEnvironment env) {
      this.streamGraph = new StreamGraph(env);
      this.streamGraph.setChaining(env.isChainingEnabled());
      this.streamGraph.setStateBackend(env.getStateBackend());
      this.env = env;
      this.alreadyTransformed = new HashMap<>();
   }
```
streamGraph：StreamGraph，在构造方法中会创建一个空的StreamGraph。

alreadyTransformed： 一个HashMap，记录了已经transformed的StreamTransformation。

## transform(): 构建的核心
generateInternal()方法就是对transformations列表中的StreamTransformation进行transform，所以核心还是在transform()方法。
```
private StreamGraph generateInternal(List<StreamTransformation<?>> transformations) {
   for (StreamTransformation<?> transformation: transformations) {
      transform(transformation);
   }
   return streamGraph;
}
```

```
private Collection<Integer> transform(StreamTransformation<?> transform) {

   if (alreadyTransformed.containsKey(transform)) {
      return alreadyTransformed.get(transform);
   }

   LOG.debug("Transforming " + transform);

   if (transform.getMaxParallelism() <= 0) {

      // if the max parallelism hasn't been set, then first use the job wide max parallelism
      // from the ExecutionConfig.
      int globalMaxParallelismFromConfig = env.getConfig().getMaxParallelism();
      if (globalMaxParallelismFromConfig > 0) {
         transform.setMaxParallelism(globalMaxParallelismFromConfig);
      }
   }

   // call at least once to trigger exceptions about MissingTypeInfo
   transform.getOutputType();

   Collection<Integer> transformedIds;
   if (transform instanceof OneInputTransformation<?, ?>) {
      transformedIds = transformOneInputTransform((OneInputTransformation<?, ?>) transform);
   } else if (transform instanceof TwoInputTransformation<?, ?, ?>) {
      transformedIds = transformTwoInputTransform((TwoInputTransformation<?, ?, ?>) transform);
   } else if (transform instanceof SourceTransformation<?>) {
      transformedIds = transformSource((SourceTransformation<?>) transform);
   } else if (transform instanceof SinkTransformation<?>) {
      transformedIds = transformSink((SinkTransformation<?>) transform);
   } else if (transform instanceof UnionTransformation<?>) {
      transformedIds = transformUnion((UnionTransformation<?>) transform);
   } else if (transform instanceof SplitTransformation<?>) {
      transformedIds = transformSplit((SplitTransformation<?>) transform);
   } else if (transform instanceof SelectTransformation<?>) {
      transformedIds = transformSelect((SelectTransformation<?>) transform);
   } else if (transform instanceof FeedbackTransformation<?>) {
      transformedIds = transformFeedback((FeedbackTransformation<?>) transform);
   } else if (transform instanceof CoFeedbackTransformation<?>) {
      transformedIds = transformCoFeedback((CoFeedbackTransformation<?>) transform);
   } else if (transform instanceof PartitionTransformation<?>) {
      transformedIds = transformPartition((PartitionTransformation<?>) transform);
   } else if (transform instanceof SideOutputTransformation<?>) {
      transformedIds = transformSideOutput((SideOutputTransformation<?>) transform);
   } else {
      throw new IllegalStateException("Unknown transformation: " + transform);
   }

   // need this check because the iterate transformation adds itself before
   // transforming the feedback edges
   if (!alreadyTransformed.containsKey(transform)) {
      alreadyTransformed.put(transform, transformedIds);
   }

   if (transform.getBufferTimeout() >= 0) {
      streamGraph.setBufferTimeout(transform.getId(), transform.getBufferTimeout());
   }
   if (transform.getUid() != null) {
      streamGraph.setTransformationUID(transform.getId(), transform.getUid());
   }
   if (transform.getUserProvidedNodeHash() != null) {
      streamGraph.setTransformationUserHash(transform.getId(), transform.getUserProvidedNodeHash());
   }

   if (transform.getMinResources() != null && transform.getPreferredResources() != null) {
      streamGraph.setResources(transform.getId(), transform.getMinResources(), transform.getPreferredResources());
   }

   return transformedIds;
}
```
拿WordCount举例，在flatMap、map、reduce、addSink过程中会将生成的DataStream的StreamTransformation添加到transformations列表中。

addSource没有将StreamTransformation添加到transformations，但是flatMap生成的StreamTransformation的input持有SourceTransformation的引用。

keyBy算子会生成KeyedStream，但是它的StreamTransformation并不会添加到transformations列表中，不过reduce生成的DataStream中的StreamTransformation中持有了KeyedStream的StreamTransformation的引用，作为它的input。

所以，WordCount中有4个StreamTransformation，前3个算子均为OneInputTransformation，最后一个为SinkTransformation。

## transformOneInputTransform
由于transformations中第一个为OneInputTransformation，所以代码首先会走到transformOneInputTransform((OneInputTransformation<?, ?>) transform)
```
private <IN, OUT> Collection<Integer> transformOneInputTransform(OneInputTransformation<IN, OUT> transform) {
    //首先递归的transform该OneInputTransformation的input
   Collection<Integer> inputIds = transform(transform.getInput());
    
    //如果递归transform input时发现input已经被transform，那么直接获取结果即可
   // the recursive call might have already transformed this
   if (alreadyTransformed.containsKey(transform)) {
      return alreadyTransformed.get(transform);
   }

    //获取共享slot资源组，默认分组名”default”
   String slotSharingGroup = determineSlotSharingGroup(transform.getSlotSharingGroup(), inputIds);
    //将该StreamTransformation添加到StreamGraph中，当做一个顶点
   streamGraph.addOperator(transform.getId(),
         slotSharingGroup,
         transform.getCoLocationGroupKey(),
         transform.getOperator(),
         transform.getInputType(),
         transform.getOutputType(),
         transform.getName());

   if (transform.getStateKeySelector() != null) {
      TypeSerializer<?> keySerializer = transform.getStateKeyType().createSerializer(env.getConfig());
      streamGraph.setOneInputStateKey(transform.getId(), transform.getStateKeySelector(), keySerializer);
   }
    //设置顶点的并行度和最大并行度
   streamGraph.setParallelism(transform.getId(), transform.getParallelism());
   streamGraph.setMaxParallelism(transform.getId(), transform.getMaxParallelism());
    //根据该StreamTransformation有多少个input，依次给StreamGraph添加边，即input——>current
   for (Integer inputId: inputIds) {
      streamGraph.addEdge(inputId, transform.getId(), 0);
   }
    //返回该StreamTransformation的id，OneInputTransformation只有自身的一个id列表
   return Collections.singleton(transform.getId());
}
```
transformOneInputTransform()方法的实现如下：
- 先递归的transform该OneInputTransformation的input，如果input已经transformed，那么直接从map中取数据即可
- 将该StreamTransformation作为一个图的顶点添加到StreamGraph中，并设置顶点的并行度和共享资源组
- 根据该StreamTransformation的input，构造图的边，有多少个input，就会生成多少边，不过OneInputTransformation顾名思义就是一个input，所以会构造一条边，即input——>currentId

## 构造顶点
添加顶点的代码逻辑如下
```
//StreamGraph类
public <IN, OUT> void addOperator(
      Integer vertexID,
      String slotSharingGroup,
      @Nullable String coLocationGroup,
      StreamOperator<OUT> operatorObject,
      TypeInformation<IN> inTypeInfo,
      TypeInformation<OUT> outTypeInfo,
      String operatorName) {
    //将StreamTransformation作为一个顶点，添加到streamNodes中
   if (operatorObject instanceof StoppableStreamSource) {
      addNode(vertexID, slotSharingGroup, coLocationGroup, StoppableSourceStreamTask.class, operatorObject, operatorName);
   } else if (operatorObject instanceof StreamSource) {
       //如果operator是StreamSource，则Task类型为SourceStreamTask
      addNode(vertexID, slotSharingGroup, coLocationGroup, SourceStreamTask.class, operatorObject, operatorName);
   } else {
       //如果operator不是StreamSource，Task类型为OneInputStreamTask
      addNode(vertexID, slotSharingGroup, coLocationGroup, OneInputStreamTask.class, operatorObject, operatorName);
   }

   TypeSerializer<IN> inSerializer = inTypeInfo != null && !(inTypeInfo instanceof MissingTypeInfo) ? inTypeInfo.createSerializer(executionConfig) : null;

   TypeSerializer<OUT> outSerializer = outTypeInfo != null && !(outTypeInfo instanceof MissingTypeInfo) ? outTypeInfo.createSerializer(executionConfig) : null;

   setSerializers(vertexID, inSerializer, null, outSerializer);

   if (operatorObject instanceof OutputTypeConfigurable && outTypeInfo != null) {
      @SuppressWarnings("unchecked")
      OutputTypeConfigurable<OUT> outputTypeConfigurable = (OutputTypeConfigurable<OUT>) operatorObject;
      // sets the output type which must be know at StreamGraph creation time
      outputTypeConfigurable.setOutputType(outTypeInfo, executionConfig);
   }

   if (operatorObject instanceof InputTypeConfigurable) {
      InputTypeConfigurable inputTypeConfigurable = (InputTypeConfigurable) operatorObject;
      inputTypeConfigurable.setInputType(inTypeInfo, executionConfig);
   }

   if (LOG.isDebugEnabled()) {
      LOG.debug("Vertex: {}", vertexID);
   }
}

protected StreamNode addNode(Integer vertexID,
   String slotSharingGroup,
   @Nullable String coLocationGroup,
   Class<? extends AbstractInvokable> vertexClass,
   StreamOperator<?> operatorObject,
   String operatorName) {

   if (streamNodes.containsKey(vertexID)) {
      throw new RuntimeException("Duplicate vertexID " + vertexID);
   }
    //构造顶点，添加到streamNodes中，streamNodes是一个Map
   StreamNode vertex = new StreamNode(environment,
      vertexID,
      slotSharingGroup,
      coLocationGroup,
      operatorObject,
      operatorName,
      new ArrayList<OutputSelector<?>>(),
      vertexClass);

   streamNodes.put(vertexID, vertex);

   return vertex;
}
```

## 构造边
上述代码里展示了如何添加顶点到StreamGraph，下面再来看看如何添加边，逻辑如下：
```
    public void addEdge(Integer upStreamVertexID, Integer downStreamVertexID, int typeNumber) {
   addEdgeInternal(upStreamVertexID,
         downStreamVertexID,
         typeNumber,
         null, //这里开始传递的partitioner都是null
         new ArrayList<String>(),
         null //outputTag对侧输出有效);

}

private void addEdgeInternal(Integer upStreamVertexID,
      Integer downStreamVertexID,
      int typeNumber,
      StreamPartitioner<?> partitioner,
      List<String> outputNames,
      OutputTag outputTag) {
   
   if (virtualSideOutputNodes.containsKey(upStreamVertexID)) {
        //针对侧输出
      int virtualId = upStreamVertexID;
      upStreamVertexID = virtualSideOutputNodes.get(virtualId).f0;
      if (outputTag == null) {
         outputTag = virtualSideOutputNodes.get(virtualId).f1;
      }
      addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, null, outputTag);
   } else if (virtualSelectNodes.containsKey(upStreamVertexID)) {
      int virtualId = upStreamVertexID;
      upStreamVertexID = virtualSelectNodes.get(virtualId).f0;
      if (outputNames.isEmpty()) {
         // selections that happen downstream override earlier selections
         outputNames = virtualSelectNodes.get(virtualId).f1;
      }
      addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, outputNames, outputTag);
   } else if (virtualPartitionNodes.containsKey(upStreamVertexID)) {
       //keyBy算子产生的PartitionTransform作为下游的input，下游的StreamTransformation添加边时会走到这
      int virtualId = upStreamVertexID;
      upStreamVertexID = virtualPartitionNodes.get(virtualId).f0;
      if (partitioner == null) {
         partitioner = virtualPartitionNodes.get(virtualId).f1;
      }
      addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, outputNames, outputTag);
   } else {
       //一般的OneInputTransform会走到这里
      StreamNode upstreamNode = getStreamNode(upStreamVertexID);
      StreamNode downstreamNode = getStreamNode(downStreamVertexID);

      // If no partitioner was specified and the parallelism of upstream and downstream
      // operator matches use forward partitioning, use rebalance otherwise.
      //如果上下顶点的并行度一致，则用ForwardPartitioner，否则用RebalancePartitioner
      if (partitioner == null && upstreamNode.getParallelism() == downstreamNode.getParallelism()) {
         partitioner = new ForwardPartitioner<Object>();
      } else if (partitioner == null) {
         partitioner = new RebalancePartitioner<Object>();
      }

      if (partitioner instanceof ForwardPartitioner) {
         if (upstreamNode.getParallelism() != downstreamNode.getParallelism()) {
            throw new UnsupportedOperationException("Forward partitioning does not allow " +
                  "change of parallelism. Upstream operation: " + upstreamNode + " parallelism: " + upstreamNode.getParallelism() +
                  ", downstream operation: " + downstreamNode + " parallelism: " + downstreamNode.getParallelism() +
                  " You must use another partitioning strategy, such as broadcast, rebalance, shuffle or global.");
         }
      }
        //一条边包括上下顶点，顶点之间的分区器等信息
      StreamEdge edge = new StreamEdge(upstreamNode, downstreamNode, typeNumber, outputNames, partitioner, outputTag);
      //分别给顶点添加出边和入边
      getStreamNode(edge.getSourceId()).addOutEdge(edge);
      getStreamNode(edge.getTargetId()).addInEdge(edge);
   }
}
```
边的属性包括上下游的顶点，和顶点之间的partitioner等信息。如果上下游的并行度一致，那么他们之间的partitioner是ForwardPartitioner，如果上下游的并行度不一致是RebalancePartitioner，当然这前提是没有设置partitioner的前提下。如果显示设置了partitioner的情况，例如keyBy算子，在内部就确定了分区器是KeyGroupStreamPartitioner，那么它们之间的分区器就是KeyGroupStreamPartitioner。

## transformSource
上述说道，transformOneInputTransform会先递归的transform该OneInputTransform的input，那么对于WordCount中在transform第一个OneInputTransform时会首先transform它的input，也就是SourceTransformation，方法在transformSource()

```
private <T> Collection<Integer> transformSource(SourceTransformation<T> source) {
   String slotSharingGroup = determineSlotSharingGroup(source.getSlotSharingGroup(), Collections.emptyList());
    //将source作为图的顶点和图的source添加StreamGraph中
   streamGraph.addSource(source.getId(),
         slotSharingGroup,
         source.getCoLocationGroupKey(),
         source.getOperator(),
         null,
         source.getOutputType(),
         "Source: " + source.getName());
   if (source.getOperator().getUserFunction() instanceof InputFormatSourceFunction) {
      InputFormatSourceFunction<T> fs = (InputFormatSourceFunction<T>) source.getOperator().getUserFunction();
      streamGraph.setInputFormat(source.getId(), fs.getFormat());
   }
   //设置source的并行度
   streamGraph.setParallelism(source.getId(), source.getParallelism());
   streamGraph.setMaxParallelism(source.getId(), source.getMaxParallelism());
   //返回自身id的单列表
   return Collections.singleton(source.getId());
}

//StreamGraph类
public <IN, OUT> void addSource(Integer vertexID,
   String slotSharingGroup,
   @Nullable String coLocationGroup,
   StreamOperator<OUT> operatorObject,
   TypeInformation<IN> inTypeInfo,
   TypeInformation<OUT> outTypeInfo,
   String operatorName) {
   addOperator(vertexID, slotSharingGroup, coLocationGroup, operatorObject, inTypeInfo, outTypeInfo, operatorName);
   sources.add(vertexID);
}
```
transformSource()逻辑也比较简单，就是将source添加到StreamGraph中，注意Source是图的根节点，没有input，所以它不需要添加边。

## transformPartition
在WordCount中，由reduce生成的DataStream的StreamTransformation是一个OneInputTransformation，同样，在transform它的时候，会首先transform它的input，而它的input就是KeyedStream中生成的PartitionTransformation。所以代码会执行transformPartition((PartitionTransformation<?>) transform)

```
private <T> Collection<Integer> transformPartition(PartitionTransformation<T> partition) {
   StreamTransformation<T> input = partition.getInput();
   List<Integer> resultIds = new ArrayList<>();
    //首先会transform PartitionTransformation的input
   Collection<Integer> transformedIds = transform(input);
   for (Integer transformedId: transformedIds) {
       //生成一个新的虚拟节点
      int virtualId = StreamTransformation.getNewNodeId();
      //将虚拟节点添加到StreamGraph中
      streamGraph.addVirtualPartitionNode(transformedId, virtualId, partition.getPartitioner());
      resultIds.add(virtualId);
   }

   return resultIds;
}

//StreamGraph类
public void addVirtualPartitionNode(Integer originalId, Integer virtualId, StreamPartitioner<?> partitioner) {

   if (virtualPartitionNodes.containsKey(virtualId)) {
      throw new IllegalStateException("Already has virtual partition node with id " + virtualId);
   }
   // virtualPartitionNodes是一个Map，存储虚拟的Partition节点
   virtualPartitionNodes.put(virtualId,
         new Tuple2<Integer, StreamPartitioner<?>>(originalId, partitioner));
}
```
transformPartition会为PartitionTransformation生成一个新的虚拟节点，同时将该虚拟节点保存到StreamGraph的virtualPartitionNodes中，并且会保存该PartitionTransformation的input，将PartitionTransformation的partitioner作为其input的分区器，在WordCount中，也就是作为map算子生成的DataStream的分区器。

在进行transform reduce生成的OneInputTransform时，它的inputIds便是transformPartition时为PartitionTransformation生成新的虚拟节点，在添加边的时候会走到下面的代码。边的形式大致是OneInputTransform(map)—KeyGroupStreamPartitioner—>OneInputTransform(window)
```
 } else if (virtualPartitionNodes.containsKey(upStreamVertexID)) {
       //keyBy算子产生的PartitionTransform作为下游的input，下游的StreamTransform添加边时会走到这
      int virtualId = upStreamVertexID;
      upStreamVertexID = virtualPartitionNodes.get(virtualId).f0;
      if (partitioner == null) {
         partitioner = virtualPartitionNodes.get(virtualId).f1;
      }
      addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, outputNames, outputTag);
```

## transformSink
在WordCount中，transformations的最后一个StreamTransformation是SinkTransformation，方法在

transformSink()
```
private <T> Collection<Integer> transformSink(SinkTransformation<T> sink) {
    //首先transform input
   Collection<Integer> inputIds = transform(sink.getInput());

   String slotSharingGroup = determineSlotSharingGroup(sink.getSlotSharingGroup(), inputIds);
    //给图添加sink，并且构造并添加图的顶点
   streamGraph.addSink(sink.getId(),
         slotSharingGroup,
         sink.getCoLocationGroupKey(),
         sink.getOperator(),
         sink.getInput().getOutputType(),
         null,
         "Sink: " + sink.getName());

   streamGraph.setParallelism(sink.getId(), sink.getParallelism());
   streamGraph.setMaxParallelism(sink.getId(), sink.getMaxParallelism());
    //构造并添加StreamGraph的边
   for (Integer inputId: inputIds) {
      streamGraph.addEdge(inputId,
            sink.getId(),
            0
      );
   }

   if (sink.getStateKeySelector() != null) {
      TypeSerializer<?> keySerializer = sink.getStateKeyType().createSerializer(env.getConfig());
      streamGraph.setOneInputStateKey(sink.getId(), sink.getStateKeySelector(), keySerializer);
   }
    //因为sink是图的末尾节点，没有下游的输出，所以返回空了
   return Collections.emptyList();
}

//StreamGraph类
public <IN, OUT> void addSink(Integer vertexID,
   String slotSharingGroup,
   @Nullable String coLocationGroup,
   StreamOperator<OUT> operatorObject,
   TypeInformation<IN> inTypeInfo,
   TypeInformation<OUT> outTypeInfo,
   String operatorName) {
   addOperator(vertexID, slotSharingGroup, coLocationGroup, operatorObject, inTypeInfo, outTypeInfo, operatorName);
   sinks.add(vertexID);
}
```
transformSink()中的逻辑也比较简单，和transformSource()类似，不同的是sink是有边的，而且sink的下游没有输出了，也就不需要作为下游的input，所以返回空列表。

在transformSink()之后，也就把所有的StreamTransformation的都进行transform了，那么这时候StreamGraph中的顶点、边、partition的虚拟顶点都构建好了，返回StreamGraph即可。下一步就是根据StreamGraph构造JobGraph了。

可以看出StreamGraph的结构还是比较简单的，每个DataStream的StreamTransformation会作为一个图的顶点（PartitionTransform是虚拟顶点），根据StreamTransformation的input来构建图的边。

## 总结
- 构建StreamGraph的起点getStreamGraph()，最终会调用StreamGraphGenerator.generateInternal进行构造。StreamGraphGenerator的构造方法中会初始化一个空的StreamGraph。

- generateInternal()的实现主要是对transformations中的StreamTransformation进行转换transform，transformations列表中的StreamTransformation就是map、flatMap、reduce、sink等算子生成新的DataStream或者DataStreamSink时添加到其中的StreamTransformation。

- 在对StreamTransformation进行transform时会先递归的对其input进行transform，得到该StreamTransformation的inputIds。一般的，在程序中都会对SourceTransformation、OneInputTransformation、PartitionTransformation、SinkTransformation进行transform（注意对PartitionTransformation的transform生成了虚拟节点，作为下游的input，用于构建StreamGraph的边）。SourceTransformation、OneInputTransformation、SinkTransformation会生成StreamGraph图的顶点，顶点包含了StreamTransformation的id、operator、AbstractInvokable类型等信息。

- 添加完顶点时，会根据顶点StreamTransformation的input生成StreamGraph的边，边的属性包含了上下游的顶点，顶点之间的partitioner等信息，如果代码中没有设置partitioner且上下游的并行度一致，那么这个partitioner是ForwardPartitioner，如果并行度不一致那么就是RebalancePartitioner；如果代码里设置了partitioner或者是像keyBy这样的算子在内部设置了partitioner，partitioner就是设置的partitioner。

- 通过最后的SinkTransformation进行transform后，整个StreamGraph就构建完了

到此，StreamGraph的构建就完成了





