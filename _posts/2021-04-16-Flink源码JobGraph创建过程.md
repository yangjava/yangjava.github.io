---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink源码JobGraph的创建过程
在StreamGraph构建完毕之后会开始构建JobGraph，然后再提交JobGraph。

## JobGraph创建
```
public JobExecutionResult execute(String jobName) throws Exception {
   Preconditions.checkNotNull(jobName, "Streaming Job name should not be null.");

   StreamGraph streamGraph = this.getStreamGraph();
   streamGraph.setJobName(jobName);

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

//ClusterClient类
public JobSubmissionResult run(FlinkPlan compiledPlan,
      List<URL> libraries, List<URL> classpaths, ClassLoader classLoader, SavepointRestoreSettings savepointSettings)
      throws ProgramInvocationException {
   JobGraph job = getJobGraph(flinkConfig, compiledPlan, libraries, classpaths, savepointSettings);
   return submitJob(job, classLoader);
}
```

## JobGraph创建的起点
那么现在就来分析JobGraph的构建过程。首先会调用StreamGraph.getJobGraph()方法，最终会调用StreamingJobGraphGenerator.createJobGraph()方法。
```
public static JobGraph getJobGraph(Configuration flinkConfig, FlinkPlan optPlan, List<URL> jarFiles, List<URL> classpaths, SavepointRestoreSettings savepointSettings) {
   JobGraph job;
   if (optPlan instanceof StreamingPlan) {
       //会调用StreamGraph.getJobGraph()方法
      job = ((StreamingPlan) optPlan).getJobGraph();
      job.setSavepointRestoreSettings(savepointSettings);
   } else {
      JobGraphGenerator gen = new JobGraphGenerator(flinkConfig);
      job = gen.compileJobGraph((OptimizedPlan) optPlan);
   }

   for (URL jar : jarFiles) {
      try {
         job.addJar(new Path(jar.toURI()));
      } catch (URISyntaxException e) {
         throw new RuntimeException("URL is invalid. This should not happen.", e);
      }
   }

   job.setClasspaths(classpaths);

   return job;
}

//StreamingJobGraphGenerator类
public static JobGraph createJobGraph(StreamGraph streamGraph, @Nullable JobID jobID) {
   return new StreamingJobGraphGenerator(streamGraph, jobID).createJobGraph();
}
```
StreamingJobGraphGenerator.createJobGraph()实现如下

```
private JobGraph createJobGraph() {

   // make sure that all vertices start immediately
   jobGraph.setScheduleMode(ScheduleMode.EAGER);

   // Generate deterministic hashes for the nodes in order to identify them across
   // submission iff they didn't change.
   //遍历StreamGraph，为StreamGraph中的每个顶点StreamNode都生成一个hash值
   Map<Integer, byte[]> hashes = defaultStreamGraphHasher.traverseStreamGraphAndGenerateHashes(streamGraph);

   // Generate legacy version hashes for backwards compatibility
   //如果用户自己设置了每个顶点的hash值，也要拿来用。这种情况比较少
   List<Map<Integer, byte[]>> legacyHashes = new ArrayList<>(legacyStreamGraphHashers.size());
   for (StreamGraphHasher hasher : legacyStreamGraphHashers) {
      legacyHashes.add(hasher.traverseStreamGraphAndGenerateHashes(streamGraph));
   }

   Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes = new HashMap<>();
    //最重要的方法，会进行operator的chain操作，生成job的顶点JobVertex和边JobEdge
   setChaining(hashes, legacyHashes, chainedOperatorHashes);
    //将每个JobVertex的入边集合也序列化到该JobVertex的StreamConfig中
   setPhysicalEdges();
    //为每个 JobVertex 指定所属的 SlotSharingGroup
   setSlotSharingAndCoLocation();
    //配置checkpoint 
   configureCheckpointing();

   JobGraphGenerator.addUserArtifactEntries(streamGraph.getEnvironment().getCachedFiles(), jobGraph);

   // set the ExecutionConfig last when it has been finalized
   try {
      jobGraph.setExecutionConfig(streamGraph.getExecutionConfig());
   }
   catch (IOException e) {
      throw new IllegalConfigurationException("Could not serialize the ExecutionConfig." +
            "This indicates that non-serializable types (like custom serializers) were registered");
   }

   return jobGraph;
}
```

## 为StreamNode设置Hash值
首先看看是如何给StreamGraph中的每个顶点StreamNode设置hash值的。defaultStreamGraphHasher在程序中是StreamGraphHasherV2
```
public Map<Integer, byte[]> traverseStreamGraphAndGenerateHashes(StreamGraph streamGraph) {
   // The hash function used to generate the hash
   final HashFunction hashFunction = Hashing.murmur3_128(0);
   final Map<Integer, byte[]> hashes = new HashMap<>();

   Set<Integer> visited = new HashSet<>();
   Queue<StreamNode> remaining = new ArrayDeque<>();

   // We need to make the source order deterministic. The source IDs are
   // not returned in the same order, which means that submitting the same
   // program twice might result in different traversal, which breaks the
   // deterministic hash assignment.
   List<Integer> sources = new ArrayList<>();
   for (Integer sourceNodeId : streamGraph.getSourceIDs()) {
      sources.add(sourceNodeId);
   }
   Collections.sort(sources);

   //
   // Traverse the graph in a breadth-first manner. Keep in mind that
   // the graph is not a tree and multiple paths to nodes can exist.
   //

   // Start with source nodes
   for (Integer sourceNodeId : sources) {
      remaining.add(streamGraph.getStreamNode(sourceNodeId));
      visited.add(sourceNodeId);
   }

   StreamNode currentNode;
   while ((currentNode = remaining.poll()) != null) {
      // Generate the hash code. Because multiple path exist to each
      // node, we might not have all required inputs available to
      // generate the hash code.
      if (generateNodeHash(currentNode, hashFunction, hashes, streamGraph.isChainingEnabled(), streamGraph)) {
         // Add the child nodes
         for (StreamEdge outEdge : currentNode.getOutEdges()) {
            StreamNode child = streamGraph.getTargetVertex(outEdge);

            if (!visited.contains(child.getId())) {
               remaining.add(child);
               visited.add(child.getId());
            }
         }
      } else {
         // We will revisit this later.
         visited.remove(currentNode.getId());
      }
   }

   return hashes;
}
```

```
private boolean generateNodeHash(
      StreamNode node,
      HashFunction hashFunction,
      Map<Integer, byte[]> hashes,
      boolean isChainingEnabled,
      StreamGraph streamGraph) {

   // Check for user-specified ID
   String userSpecifiedHash = node.getTransformationUID();

   if (userSpecifiedHash == null) {
      // Check that all input nodes have their hashes computed
      for (StreamEdge inEdge : node.getInEdges()) {
         // If the input node has not been visited yet, the current
         // node will be visited again at a later point when all input
         // nodes have been visited and their hashes set.
         if (!hashes.containsKey(inEdge.getSourceId())) {
            return false;
         }
      }

      Hasher hasher = hashFunction.newHasher();
      byte[] hash = generateDeterministicHash(node, hasher, hashes, isChainingEnabled, streamGraph);

      if (hashes.put(node.getId(), hash) != null) {
         // Sanity check
         throw new IllegalStateException("Unexpected state. Tried to add node hash " +
               "twice. This is probably a bug in the JobGraph generator.");
      }

      return true;
   } else {
      Hasher hasher = hashFunction.newHasher();
      byte[] hash = generateUserSpecifiedHash(node, hasher);

      for (byte[] previousHash : hashes.values()) {
         if (Arrays.equals(previousHash, hash)) {
            throw new IllegalArgumentException("Hash collision on user-specified ID " +
                  "\"" + userSpecifiedHash + "\". " +
                  "Most likely cause is a non-unique ID. Please check that all IDs " +
                  "specified via `uid(String)` are unique.");
         }
      }

      if (hashes.put(node.getId(), hash) != null) {
         // Sanity check
         throw new IllegalStateException("Unexpected state. Tried to add node hash " +
               "twice. This is probably a bug in the JobGraph generator.");
      }

      return true;
   }
}
```
其大致的实现就是广度优先遍历StreamGraph，为StreamGraph中的每个StreamNode生成一个hash值，hash依赖StreamTransformation的uid，如果用户没有设置uid，系统将自行为StreamNode生成一个hash值，如果用户设置了uid，那么就根据用户的uid生成hash值。

还有一个是用户提供的hash值，这个比较少见，也就是legacyStreamGraphHashers中的Hasher，实现很简单，就是把用户设置的hash值添加到map中。

```
public class StreamGraphUserHashHasher implements StreamGraphHasher {

   @Override
   public Map<Integer, byte[]> traverseStreamGraphAndGenerateHashes(StreamGraph streamGraph) {
      HashMap<Integer, byte[]> hashResult = new HashMap<>();
      for (StreamNode streamNode : streamGraph.getStreamNodes()) {

         String userHash = streamNode.getUserHash();

         if (null != userHash) {
            hashResult.put(streamNode.getId(), StringUtils.hexStringToByte(userHash));
         }
      }

      return hashResult;
   }
}
```
Hash值的应用：

这里就是根据StreamGraph的配置，给StreamGraph中的每个StreamNode产生一个长度为16的字节数组的散列值，这个散列值是用来后续生成JobGraph中对应的JobVertex的ID。在Flink中，任务存在从checkpoint中进行状态恢复的场景，而在恢复时，是以JobVertexID为依据的，所有就需要任务在重启的过程中，对于相同的任务，其各JobVertexID能够保持不变，而StreamGraph中各个StreamNode的ID，就是其包含的StreamTransformation的ID，而StreamTransformation的ID是在对数据流中的数据进行转换的过程中，通过一个静态的累加器生成的，比如有多个数据源时，每个数据源添加的顺序不一致，则有可能导致相同数据处理逻辑的任务，就会对应于不同的ID，所以为了得到确定的ID，在进行JobVertexID的产生时，需要以一种确定的方式来确定其值，要么是通过用户为每个ID直接指定对应的一个散列值，要么参考StreamGraph中的一些特征，为每个JobVertex产生一个确定的ID。

## 设置StreamNode chain
接下来是最重要的方法setChaining()，这个方法将会进行StreamNode chain（也可以理解成operator chain），生成JobGraph顶点和边。

```
//从source开始，生成StreamNode chain
private void setChaining(Map<Integer, byte[]> hashes, List<Map<Integer, byte[]>> legacyHashes, Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes) {
   for (Integer sourceNodeId : streamGraph.getSourceIDs()) {
      createChain(sourceNodeId, sourceNodeId, hashes, legacyHashes, 0, chainedOperatorHashes);
   }
}

private List<StreamEdge> createChain(
      Integer startNodeId,
      Integer currentNodeId,
      Map<Integer, byte[]> hashes,
      List<Map<Integer, byte[]>> legacyHashes,
      int chainIndex,
      Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes) {

   if (!builtVertices.contains(startNodeId)) {
        //过渡的出边集合，就是两个StreamNode不能再进行chain的那条边，用于生成JobEdge
      List<StreamEdge> transitiveOutEdges = new ArrayList<StreamEdge>();
        //两个StreamNode可以进行chain的出边集合
      List<StreamEdge> chainableOutputs = new ArrayList<StreamEdge>();
      //两个StreamNode不能进行chain的出边
      List<StreamEdge> nonChainableOutputs = new ArrayList<StreamEdge>();
    //判断该边的两个StreamNode是否能进行chain，分别添加到不同的列表中
      for (StreamEdge outEdge : streamGraph.getStreamNode(currentNodeId).getOutEdges()) {
         if (isChainable(outEdge, streamGraph)) {
            chainableOutputs.add(outEdge);
         } else {
            nonChainableOutputs.add(outEdge);
         }
      }
    //如果存在可以chain的边，那么就继续往这条边的target operator进行chain。transitiveOutEdges最终返回给首次调用栈的是不能再继续chain的那条边
      for (StreamEdge chainable : chainableOutputs) {
         transitiveOutEdges.addAll(
               createChain(startNodeId, chainable.getTargetId(), hashes, legacyHashes, chainIndex + 1, chainedOperatorHashes));
      }
    //如果存在了不可chain的边，说明该边就是StreamNode chain之间的过渡边，添加到transitiveOutEdges中，
    //继续对该边的target StreamNode进行新的createChain操作，意味着一个新的chain
      for (StreamEdge nonChainable : nonChainableOutputs) {
         transitiveOutEdges.add(nonChainable);
         createChain(nonChainable.getTargetId(), nonChainable.getTargetId(), hashes, legacyHashes, 0, chainedOperatorHashes);
      }
    
      List<Tuple2<byte[], byte[]>> operatorHashes =
         chainedOperatorHashes.computeIfAbsent(startNodeId, k -> new ArrayList<>());
    //当前StreamNode的主hash值
      byte[] primaryHashBytes = hashes.get(currentNodeId);

      for (Map<Integer, byte[]> legacyHash : legacyHashes) {
         operatorHashes.add(new Tuple2<>(primaryHashBytes, legacyHash.get(currentNodeId)));
      }
    //设置当前StreamNode的名称。例如 2: flatMap -> Map
      chainedNames.put(currentNodeId, createChainedName(currentNodeId, chainableOutputs));
      chainedMinResources.put(currentNodeId, createChainedMinResources(currentNodeId, chainableOutputs));
      chainedPreferredResources.put(currentNodeId, createChainedPreferredResources(currentNodeId, chainableOutputs));
    //在递归处理完chain上的StreamNode后，在处理头结点时会创建jobVertex
      StreamConfig config = currentNodeId.equals(startNodeId)
            ? createJobVertex(startNodeId, hashes, legacyHashes, chainedOperatorHashes)
            : new StreamConfig(new Configuration());

      setVertexConfig(currentNodeId, config, chainableOutputs, nonChainableOutputs);

      if (currentNodeId.equals(startNodeId)) {
        //如果是chain的头结点，由于递归的方法调用，先处理完子节点，最后才处理头结点
         config.setChainStart();
         config.setChainIndex(0);
         config.setOperatorName(streamGraph.getStreamNode(currentNodeId).getOperatorName());
         config.setOutEdgesInOrder(transitiveOutEdges);
         config.setOutEdges(streamGraph.getStreamNode(currentNodeId).getOutEdges());
        //递归处理完chain上的StreamNode后，在处理头结点时，会将头结点和过渡边进行connect，生成JobEdge和中间结果集IntermediateDataSet 
         for (StreamEdge edge : transitiveOutEdges) {
            connect(startNodeId, edge);
         }

         config.setTransitiveChainedTaskConfigs(chainedConfigs.get(startNodeId));

      } else {
        //如果是 chain 中的子节点
         Map<Integer, StreamConfig> chainedConfs = chainedConfigs.get(startNodeId);

         if (chainedConfs == null) {
            chainedConfigs.put(startNodeId, new HashMap<Integer, StreamConfig>());
         }
         //设置该StreamNode在chain上的位置
         config.setChainIndex(chainIndex);
         StreamNode node = streamGraph.getStreamNode(currentNodeId);
         config.setOperatorName(node.getOperatorName());
         chainedConfigs.get(startNodeId).put(currentNodeId, config);
      }

      config.setOperatorID(new OperatorID(primaryHashBytes));
    //没有可以chain的出边了，说明chain到头了，设置为ChainEnd
      if (chainableOutputs.isEmpty()) {
         config.setChainEnd();
      }
      //返回过渡的边，返回值可能会添加到上一层栈的transitiveOutEdges中，通常是chain的中间方法调用过程中
      return transitiveOutEdges;

   } else {
      return new ArrayList<>();
   }
}
```

两个StreamNode能进行chain的条件：
```
public static boolean isChainable(StreamEdge edge, StreamGraph streamGraph) {
   StreamNode upStreamVertex = streamGraph.getSourceVertex(edge);
   StreamNode downStreamVertex = streamGraph.getTargetVertex(edge);

   StreamOperator<?> headOperator = upStreamVertex.getOperator();
   StreamOperator<?> outOperator = downStreamVertex.getOperator();

   return downStreamVertex.getInEdges().size() == 1
         && outOperator != null
         && headOperator != null
         && upStreamVertex.isSameSlotSharingGroup(downStreamVertex)
         && outOperator.getChainingStrategy() == ChainingStrategy.ALWAYS
         && (headOperator.getChainingStrategy() == ChainingStrategy.HEAD ||
            headOperator.getChainingStrategy() == ChainingStrategy.ALWAYS)
         && (edge.getPartitioner() instanceof ForwardPartitioner)
         && upStreamVertex.getParallelism() == downStreamVertex.getParallelism()
         && streamGraph.isChainingEnabled();
}
```
条件一共有9个：
1、下游节点只有一个输入
2、下游节点的操作符不为null
3、上游节点的操作符不为null
4、上下游节点在一个槽位共享组内
5、下游节点的连接策略是 ALWAYS
6、上游节点的连接策略是 HEAD 或者 ALWAYS
7、edge 的分区函数是 ForwardPartitioner 的实例
8、上下游节点的并行度相等
9、可以进行节点连接操作

## 创建顶点
创建顶点是在处理chain的头结点时进行的
```
private StreamConfig createJobVertex(
      Integer streamNodeId,
      Map<Integer, byte[]> hashes,
      List<Map<Integer, byte[]>> legacyHashes,
      Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes) {

   JobVertex jobVertex;
   StreamNode streamNode = streamGraph.getStreamNode(streamNodeId);
    //头结点的hash值
   byte[] hash = hashes.get(streamNodeId);

   if (hash == null) {
      throw new IllegalStateException("Cannot find node hash. " +
            "Did you generate them before calling this method?");
   }
    //根据头结点的hash值生成JobVertexID
   JobVertexID jobVertexId = new JobVertexID(hash);

   List<JobVertexID> legacyJobVertexIds = new ArrayList<>(legacyHashes.size());
   for (Map<Integer, byte[]> legacyHash : legacyHashes) {
      hash = legacyHash.get(streamNodeId);
      if (null != hash) {
         legacyJobVertexIds.add(new JobVertexID(hash));
      }
   }

   List<Tuple2<byte[], byte[]>> chainedOperators = chainedOperatorHashes.get(streamNodeId);
   List<OperatorID> chainedOperatorVertexIds = new ArrayList<>();
   List<OperatorID> userDefinedChainedOperatorVertexIds = new ArrayList<>();
   if (chainedOperators != null) {
      for (Tuple2<byte[], byte[]> chainedOperator : chainedOperators) {
         chainedOperatorVertexIds.add(new OperatorID(chainedOperator.f0));
         userDefinedChainedOperatorVertexIds.add(chainedOperator.f1 != null ? new OperatorID(chainedOperator.f1) : null);
      }
   }

   if (streamNode.getInputFormat() != null) {
      jobVertex = new InputFormatVertex(
            chainedNames.get(streamNodeId),
            jobVertexId,
            legacyJobVertexIds,
            chainedOperatorVertexIds,
            userDefinedChainedOperatorVertexIds);
      TaskConfig taskConfig = new TaskConfig(jobVertex.getConfiguration());
      taskConfig.setStubWrapper(new UserCodeObjectWrapper<Object>(streamNode.getInputFormat()));
   } else {
       //创建JobVertex，包含由chain的头结点的jobVertexId，和chain上所有的operator的id
       //name的形式为FlatMap -> Map
      jobVertex = new JobVertex(
            chainedNames.get(streamNodeId),
            jobVertexId,
            legacyJobVertexIds,
            chainedOperatorVertexIds,
            userDefinedChainedOperatorVertexIds);
   }

   jobVertex.setResources(chainedMinResources.get(streamNodeId), chainedPreferredResources.get(streamNodeId));
    //设置核心Task的类型，例如OneInputStreamTask.class
   jobVertex.setInvokableClass(streamNode.getJobVertexClass());
    //设置节点并行度
   int parallelism = streamNode.getParallelism();

   if (parallelism > 0) {
      jobVertex.setParallelism(parallelism);
   } else {
      parallelism = jobVertex.getParallelism();
   }

   jobVertex.setMaxParallelism(streamNode.getMaxParallelism());

   if (LOG.isDebugEnabled()) {
      LOG.debug("Parallelism set: {} for {}", parallelism, streamNodeId);
   }

   // TODO: inherit InputDependencyConstraint from the head operator
   jobVertex.setInputDependencyConstraint(streamGraph.getExecutionConfig().getDefaultInputDependencyConstraint());
    //将streamNodeId和jobVertex对应关系存放在jobVertices中
   jobVertices.put(streamNodeId, jobVertex);
   builtVertices.add(streamNodeId);
   //将生成的顶点添加到jobGraph中
   jobGraph.addVertex(jobVertex);

   return new StreamConfig(jobVertex.getConfiguration());
}
```
上述代码大致实现逻辑为
- 根据StreamNode chain的头结点的Hash值构建顶点的ID JobVertexID，也就是说JobVertexID是根据头结点而决定。
- 创建chainedOperatorVertexIds，即StreamNode chain上所有的OperatorID
- 根据jobVertexId和chainedOperatorVertexIds创建顶点JobVertex。JobVertex同时还需要设置并行读、资源数，运行StreamTask的类型，例如OneInputStreamTask
- 将生成的顶点添加到jobVertices和JobGraph中，JobGraph会将顶点JobVertex添加到自身的一个Map数据结构taskVertices中

## 创建边和中间结果集
```
private void connect(Integer headOfChain, StreamEdge edge) {

   physicalEdgesInOrder.add(edge);

   Integer downStreamvertexID = edge.getTargetId();

   JobVertex headVertex = jobVertices.get(headOfChain);
   JobVertex downStreamVertex = jobVertices.get(downStreamvertexID);

   StreamConfig downStreamConfig = new StreamConfig(downStreamVertex.getConfiguration());

   downStreamConfig.setNumberOfInputs(downStreamConfig.getNumberOfInputs() + 1);
    //获取过渡边的分区器，根据分区器的不同创建对应的JobEdge
   StreamPartitioner<?> partitioner = edge.getPartitioner();
   JobEdge jobEdge;
   if (partitioner instanceof ForwardPartitioner || partitioner instanceof RescalePartitioner) {
      jobEdge = downStreamVertex.connectNewDataSetAsInput(
         headVertex,
         DistributionPattern.POINTWISE,
         ResultPartitionType.PIPELINED_BOUNDED);
   } else {
       //REBALANCE和HASH走这里，创建中间结果集和JobEdge
      jobEdge = downStreamVertex.connectNewDataSetAsInput(
            headVertex,
            DistributionPattern.ALL_TO_ALL,
            ResultPartitionType.PIPELINED_BOUNDED);
   }
   // set strategy name so that web interface can show it.
   jobEdge.setShipStrategyName(partitioner.toString());

   if (LOG.isDebugEnabled()) {
      LOG.debug("CONNECTED: {} - {} -> {}", partitioner.getClass().getSimpleName(),
            headOfChain, downStreamvertexID);
   }
}

//JobVertex类
public JobEdge connectNewDataSetAsInput(
      JobVertex input,
      DistributionPattern distPattern,
      ResultPartitionType partitionType) {

   IntermediateDataSet dataSet = input.createAndAddResultDataSet(partitionType);
    //根据中间结果集和下游的JobVertex构建JobEdge
   JobEdge edge = new JobEdge(dataSet, this, distPattern);
   //将JobEdge作为下游顶点的输入
   this.inputs.add(edge);
   //设置JobEdge为中间结果集的消费者
   dataSet.addConsumer(edge);
   return edge;
}

public IntermediateDataSet createAndAddResultDataSet(ResultPartitionType partitionType) {
   return createAndAddResultDataSet(new IntermediateDataSetID(), partitionType);
}

public IntermediateDataSet createAndAddResultDataSet(
      IntermediateDataSetID id,
      ResultPartitionType partitionType) {

   IntermediateDataSet result = new IntermediateDataSet(id, partitionType, this);
   //将中间结果集IntermediateDataSet作为上游顶点的输出结果
   this.results.add(result);
   return result;
}

//IntermediateDataSet类
public class IntermediateDataSet implements java.io.Serializable {
   
   private static final long serialVersionUID = 1L;

   
   private final IntermediateDataSetID id;       // the identifier
   
   private final JobVertex producer;        // the operation that produced this data set
   
   private final List<JobEdge> consumers = new ArrayList<JobEdge>();

   // The type of partition to use at runtime
   private final ResultPartitionType resultType;
   
   // --------------------------------------------------------------------------------------------
    //将上游顶点JobVertex作为中间结果集的生产者producer 
   public IntermediateDataSet(IntermediateDataSetID id, ResultPartitionType resultType, JobVertex producer) {
      this.id = checkNotNull(id);
      this.producer = checkNotNull(producer);
      this.resultType = checkNotNull(resultType);
   }
   ...
```
上述代码的大致实现如下：
- connect的参数为上游chain的头结点，chain的中间过渡边，即不能再进行chain的那条边
- 根据头结点和边从jobVertices中找到对应的JobGraph的上下游顶点JobVertex
- 获取过渡边的分区器，创建对应的中间结果集IntermediateDataSet和JobEdge。IntermediateDataSet由上游的顶点JobVertex创建，IntermediateDataSet在创建时会将上游的顶点JobVertex作为它的生产者producer，IntermediateDataSet作为上游顶点的输出。JobEdge中包含了中间结果集IntermediateDataSet和下游的顶点JobVertex， JobEdge作为中间结果集IntermediateDataSet的消费者，JobEdge作为下游顶点JobVertex的input。整个过程就是

上游JobVertex——>IntermediateDataSet——>JobEdge——>下游JobVertex

- 虽然代码中并未将中间结果集和JobEdge显示的添加到JobGraph，但是JobGraph中包含有顶点JobVertex，顶点JobVertex中持有了对中间结果集IntermediateDataSet的引用，而IntermediateDataSet 又持有对JobEdge的引用，就相当于一张图形结构

到此，JobGraph的顶点JobVertex和边JobEdge和中间结果集IntermediateDataSet 都已经被创建出来了。接下来就是为顶点设置共享solt组、设置checkpoint配置等非核心操作了，最后返回JobGraph，JobGraph的构建就完毕了。

## 总结：
- 在StreamGraph构建完毕之后会开始构建JobGraph，然后再提交JobGraph。JobGraph也是由StreamGraph来创建的
- StreamingJobGraphGenerator.createJobGraph()是构建JobGraph的核心实现，实现中首先会广度优先遍历StreamGraph，为其中的每个StreamNode生成一个Hash值，如果用户设置了operator的uid，那么就根据uid来生成Hash值，否则系统会自己为每个StreamNode生成一个Hash值。如果用户自己为operator提供了Hash值，也会拿来用。生成Hash值的作用主要应用在从checkpoint中的数据恢复
- 在生成Hash值之后，会调用setChaining()方法，这是创建operator chain、构建JobGraph顶点JobVertex、边JobEdge、中间结果集IntermediateDataSet的核心方法。
   - 创建StreamNode chain(operator chain)
从source开始，处理出边StreamEdge和target节点(edge的下游节点)，递归的向下处理StreamEdge上和target StreamNode，直到找到那条过渡边，即不能再进行chain的那条边为止。那么这中间的StreamNode可以作为一个chain。这种递归向下的方式使得程序先chain完StreamGraph后面的节点，再处理头结点，类似于后序递归遍历。
   - 创建顶点JobVertex
顶点的创建在创建StreamNode chain的过程中，当已经完成了一个StreamNode chain的创建，在处理这个chain的头结点时会创建顶点JobVertex，顶点的JobVertexID根据头结点的Hash值而决定。同时JobVertex持有了chain上的所有operatorID。因为是后续遍历，所有JobVertex的创建过程是从后往前进行创建，即从sink端到source端
   - 创建边JobEdge和IntermediateDataSet
JobEdge的创建是在完成一个StreamNode chain，在处理头结点并创建完顶点JobVertex之后、根据头结点和过渡边进行connect操作时进行的，连接的是当前的JobVertex和下游的JobVertex，因为JobVertex的创建是由下至上的。

根据头结点和边从jobVertices中找到对应的JobGraph的上下游顶点JobVertex，获取过渡边的分区器，创建对应的中间结果集IntermediateDataSet和JobEdge。IntermediateDataSet由上游的顶点JobVertex创建，上游顶点JobVertex作为它的生产者producer，IntermediateDataSet作为上游顶点的输出。JobEdge中持有了中间结果集IntermediateDataSet和下游的顶点JobVertex的引用， JobEdge作为中间结果集IntermediateDataSet的消费者，JobEdge作为下游顶点JobVertex的input。整个过程就是

上游JobVertex——>IntermediateDataSet——>JobEdge——>下游JobVertex

- 接下来就是为顶点设置共享solt组、设置checkpoint配置等操作了，最后返回JobGraph，JobGraph的构建就完毕了






