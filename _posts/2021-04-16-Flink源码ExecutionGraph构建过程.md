---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink源码ExecutionGraph构建过程
ExecutionGraph是flink任务执行的最直接的执行图，而不是StreamGraph和JobGraph。之前分析过StreamGraph和JobGraph的构建，他们都是在客户端进行构建，由StreamGraph构建JobGraph，提交给JobMaster。ExecutionGraph则是在服务端由JobMaster根据JobGraph进行构建的。现在就来分析一下ExecutionGraph的构建过程。

## 构建起点
ExecutionGraph是JobMaster的成员。构建发生在JobMaster的构造方法中
```
public JobMaster(
      RpcService rpcService,
      JobMasterConfiguration jobMasterConfiguration,
      ResourceID resourceId,
      JobGraph jobGraph,
      HighAvailabilityServices highAvailabilityService,
      SlotPoolFactory slotPoolFactory,
      SchedulerFactory schedulerFactory,
      JobManagerSharedServices jobManagerSharedServices,
      HeartbeatServices heartbeatServices,
      JobManagerJobMetricGroupFactory jobMetricGroupFactory,
      OnCompletionActions jobCompletionActions,
      FatalErrorHandler fatalErrorHandler,
      ClassLoader userCodeLoader) throws Exception {

   super(rpcService, AkkaRpcServiceUtils.createRandomName(JOB_MANAGER_NAME));
    ...
    //构建executionGraph
   this.executionGraph = createAndRestoreExecutionGraph(jobManagerJobMetricGroup);
   ...
}
```
createAndRestoreExecutionGraph()方法最终会调用ExecutionGraphBuilder.buildGraph()进行构建
```
private ExecutionGraph createAndRestoreExecutionGraph(JobManagerJobMetricGroup currentJobManagerJobMetricGroup) throws Exception {

   ExecutionGraph newExecutionGraph = createExecutionGraph(currentJobManagerJobMetricGroup);
   ...
   return newExecutionGraph;
}

private ExecutionGraph createExecutionGraph(JobManagerJobMetricGroup currentJobManagerJobMetricGroup) throws JobExecutionException, JobException {
   return ExecutionGraphBuilder.buildGraph(
      null,
      jobGraph,
      jobMasterConfiguration.getConfiguration(),
      scheduledExecutorService,
      scheduledExecutorService,
      scheduler,
      userCodeLoader,
      highAvailabilityServices.getCheckpointRecoveryFactory(),
      rpcTimeout,
      restartStrategy,
      currentJobManagerJobMetricGroup,
      blobWriter,
      jobMasterConfiguration.getSlotRequestTimeout(),
      log);
}

//ExecutionGraphBuilder类
public static ExecutionGraph buildGraph(
      @Nullable ExecutionGraph prior,
      JobGraph jobGraph,
      Configuration jobManagerConfig,
      ScheduledExecutorService futureExecutor,
      Executor ioExecutor,
      SlotProvider slotProvider,
      ClassLoader classLoader,
      CheckpointRecoveryFactory recoveryFactory,
      Time rpcTimeout,
      RestartStrategy restartStrategy,
      MetricGroup metrics,
      BlobWriter blobWriter,
      Time allocationTimeout,
      Logger log)
   throws JobExecutionException, JobException {

   return buildGraph(
      prior,
      jobGraph,
      jobManagerConfig,
      futureExecutor,
      ioExecutor,
      slotProvider,
      classLoader,
      recoveryFactory,
      rpcTimeout,
      restartStrategy,
      metrics,
      -1,
      blobWriter,
      allocationTimeout,
      log);
}
```
buildGraph()的代码较长，这里只列出核心的代码片段
```
public static ExecutionGraph buildGraph(
      @Nullable ExecutionGraph prior,
      JobGraph jobGraph,
      Configuration jobManagerConfig,
      ScheduledExecutorService futureExecutor,
      Executor ioExecutor,
      SlotProvider slotProvider,
      ClassLoader classLoader,
      CheckpointRecoveryFactory recoveryFactory,
      Time rpcTimeout,
      RestartStrategy restartStrategy,
      MetricGroup metrics,
      int parallelismForAutoMax,
      BlobWriter blobWriter,
      Time allocationTimeout,
      Logger log)
   throws JobExecutionException, JobException {
    ...
   // create a new execution graph, if none exists so far
   //构建一个新的ExecutionGraph 
   final ExecutionGraph executionGraph;
   try {
      executionGraph = (prior != null) ? prior :
         new ExecutionGraph(
            jobInformation,
            futureExecutor,
            ioExecutor,
            rpcTimeout,
            restartStrategy,
            failoverStrategy,
            slotProvider,
            classLoader,
            blobWriter,
            allocationTimeout);
   } catch (IOException e) {
      throw new JobException("Could not create the ExecutionGraph.", e);
   }

   // set the basic properties
   executionGraph.setScheduleMode(jobGraph.getScheduleMode());
   executionGraph.setQueuedSchedulingAllowed(jobGraph.getAllowQueuedScheduling());

   try {
       //设置json格式的执行计划
      executionGraph.setJsonPlan(JsonPlanGenerator.generatePlan(jobGraph));
   }
   catch (Throwable t) {
      log.warn("Cannot create JSON plan for job", t);
      // give the graph an empty plan
      executionGraph.setJsonPlan("{}");
   }

   ...

   // topologically sort the job vertices and attach the graph to the existing one
   //按照拓扑顺序获取JobGraph的所有顶点JobVertex
   List<JobVertex> sortedTopology = jobGraph.getVerticesSortedTopologicallyFromSources();
   if (log.isDebugEnabled()) {
      log.debug("Adding {} vertices from job graph {} ({}).", sortedTopology.size(), jobName, jobId);
   }
   //核心方法，根据JobGraph来绑定生成ExecutionGraph的内部属性
   executionGraph.attachJobGraph(sortedTopology);

   if (log.isDebugEnabled()) {
      log.debug("Successfully created execution graph from job graph {} ({}).", jobName, jobId);
   }

   // configure the state checkpointing
   //设置checkpoint相关
   ...

   // create all the metrics for the Execution Graph
   //设置metric相关
    ...
    
   return executionGraph;
}
```
buildGraph()的总体大概实现如下：

- 构造一个新的ExecutionGraph
- 获取按照Job的拓扑顺序排序的JobGraph的所有顶点JobVertex
- 根据JobGraph的所有顶点JobVertex来绑定ExecutionGraph的各种内部属性，也是核心实现，在attachJobGraph()方法中
- 设置ExecutionGraph的checkpoint和metric服务相关信息

其中最重要的代码就是attachJobGraph()了。在此之前先来看看ExecutionGraph的构造方法
```
public ExecutionGraph(
      JobInformation jobInformation,
      ScheduledExecutorService futureExecutor,
      Executor ioExecutor,
      Time rpcTimeout,
      RestartStrategy restartStrategy,
      FailoverStrategy.Factory failoverStrategyFactory,
      SlotProvider slotProvider,
      ClassLoader userClassLoader,
      BlobWriter blobWriter,
      Time allocationTimeout) throws IOException {

   checkNotNull(futureExecutor);

   this.jobInformation = Preconditions.checkNotNull(jobInformation);

   this.blobWriter = Preconditions.checkNotNull(blobWriter);

   this.jobInformationOrBlobKey = BlobWriter.serializeAndTryOffload(jobInformation, jobInformation.getJobId(), blobWriter);

   this.futureExecutor = Preconditions.checkNotNull(futureExecutor);
   this.ioExecutor = Preconditions.checkNotNull(ioExecutor);

   this.slotProvider = Preconditions.checkNotNull(slotProvider, "scheduler");
   this.userClassLoader = Preconditions.checkNotNull(userClassLoader, "userClassLoader");

    //类型ConcurrentHashMap<JobVertexID, ExecutionJobVertex>
   this.tasks = new ConcurrentHashMap<>(16);
   //类型ConcurrentHashMap<IntermediateDataSetID, IntermediateResult>
   this.intermediateResults = new ConcurrentHashMap<>(16);
   //类型List<ExecutionJobVertex>
   this.verticesInCreationOrder = new ArrayList<>(16);
   //类型ConcurrentHashMap<ExecutionAttemptID, Execution>
   this.currentExecutions = new ConcurrentHashMap<>(16);

   this.jobStatusListeners  = new CopyOnWriteArrayList<>();
   this.executionListeners = new CopyOnWriteArrayList<>();

   this.stateTimestamps = new long[JobStatus.values().length];
   this.stateTimestamps[JobStatus.CREATED.ordinal()] = System.currentTimeMillis();

   this.rpcTimeout = checkNotNull(rpcTimeout);
   this.allocationTimeout = checkNotNull(allocationTimeout);

   this.restartStrategy = restartStrategy;
   this.kvStateLocationRegistry = new KvStateLocationRegistry(jobInformation.getJobId(), getAllVertices());

   this.verticesFinished = new AtomicInteger();

   this.globalModVersion = 1L;

   // the failover strategy must be instantiated last, so that the execution graph
   // is ready by the time the failover strategy sees it
   this.failoverStrategy = checkNotNull(failoverStrategyFactory.create(this), "null failover strategy");

   this.schedulingFuture = null;
   this.jobMasterMainThreadExecutor =
      new ComponentMainThreadExecutor.DummyComponentMainThreadExecutor(
         "ExecutionGraph is not initialized with proper main thread executor. " +
            "Call to ExecutionGraph.start(...) required.");

   LOG.info("Job recovers via failover strategy: {}", failoverStrategy.getStrategyName());
}
```
跟执行图相关的重要几个成员：

tasks: 类型ConcurrentHashMap<JobVertexID, ExecutionJobVertex>, ExecutionJobVertex是ExecutionGraph中的一级顶点，等同于JobGraph中的顶点JobVertex

intermediateResults：类型ConcurrentHashMap<IntermediateDataSetID, IntermediateResult>，IntermediateResult是ExecutionGraph中的中间结果集，等同于JobGraph中的IntermediateDataSet

verticesInCreationOrder：类型List<ExecutionJobVertex>，ExecutionGraph中所有的ExecutionJobVertex集合，按照ExecutionJobVertex的创建顺序排序的

currentExecutions：类型ConcurrentHashMap<ExecutionAttemptID, Execution>，Execution是最小的一个执行单元，是一个task的物理执行。

## 核心方法attachJobGraph()
再来看看最重要的方法attachJobGraph()
```
public void attachJobGraph(List<JobVertex> topologiallySorted) throws JobException {

   assertRunningInJobMasterMainThread();

   LOG.debug("Attaching {} topologically sorted vertices to existing job graph with {} " +
         "vertices and {} intermediate results.",
      topologiallySorted.size(),
      tasks.size(),
      intermediateResults.size());

   final ArrayList<ExecutionJobVertex> newExecJobVertices = new ArrayList<>(topologiallySorted.size());
   final long createTimestamp = System.currentTimeMillis();
   
    //针对JobGraph中每个顶点jobVertex生成一个ExecutionGraph的顶点ExecutionJobVertex
   for (JobVertex jobVertex : topologiallySorted) {

      if (jobVertex.isInputVertex() && !jobVertex.isStoppable()) {
         this.isStoppable = false;
      }

      // create the execution job vertex and attach it to the graph
      //构造方法很重要
      ExecutionJobVertex ejv = new ExecutionJobVertex(
         this,
         jobVertex,
         1,
         rpcTimeout,
         globalModVersion,
         createTimestamp);
        //重要方法，连接ExecutionEdge、ExecutionVertex、IntermediateResultPartition
      ejv.connectToPredecessors(this.intermediateResults);

      ExecutionJobVertex previousTask = this.tasks.putIfAbsent(jobVertex.getID(), ejv);
      if (previousTask != null) {
         throw new JobException(String.format("Encountered two job vertices with ID %s : previous=[%s] / new=[%s]",
            jobVertex.getID(), ejv, previousTask));
      }
      //ExecutionJobVertex下游对应的是IntermediateResult，将其添加到intermediateResults
      //作为下游ExecutionJobVertex的source(或input) IntermediateResult使用
      for (IntermediateResult res : ejv.getProducedDataSets()) {
         IntermediateResult previousDataSet = this.intermediateResults.putIfAbsent(res.getId(), res);
         if (previousDataSet != null) {
            throw new JobException(String.format("Encountered two intermediate data set with ID %s : previous=[%s] / new=[%s]",
               res.getId(), res, previousDataSet));
         }
      }
      //将ExecutionJobVertex添加到verticesInCreationOrder中
      this.verticesInCreationOrder.add(ejv);
      this.numVerticesTotal += ejv.getParallelism();
      newExecJobVertices.add(ejv);
   }

   terminationFuture = new CompletableFuture<>();
   failoverStrategy.notifyNewVertices(newExecJobVertices);
}
```
attachJobGraph()的实现大致步骤是
- 针对JobGraph中每个顶点jobVertex生成一个ExecutionGraph的顶点ExecutionJobVertex
- 连接ExecutionEdge、ExecutionVertex、IntermediateResultPartition

## 构建ExecutionJobVertex
先来看ExecutionJobVertex的构造方法，ExecutionJobVertex和JobGraph的JobVertex是一一对应的。
```
public ExecutionJobVertex(
      ExecutionGraph graph,
      JobVertex jobVertex,
      int defaultParallelism,
      Time timeout,
      long initialGlobalModVersion,
      long createTimestamp) throws JobException {

   if (graph == null || jobVertex == null) {
      throw new NullPointerException();
   }

   this.graph = graph;
   this.jobVertex = jobVertex;

   int vertexParallelism = jobVertex.getParallelism();
   int numTaskVertices = vertexParallelism > 0 ? vertexParallelism : defaultParallelism;

   final int configuredMaxParallelism = jobVertex.getMaxParallelism();

   this.maxParallelismConfigured = (VALUE_NOT_SET != configuredMaxParallelism);

   // if no max parallelism was configured by the user, we calculate and set a default
   setMaxParallelismInternal(maxParallelismConfigured ?
         configuredMaxParallelism : KeyGroupRangeAssignment.computeDefaultMaxParallelism(numTaskVertices));

   // verify that our parallelism is not higher than the maximum parallelism
   if (numTaskVertices > maxParallelism) {
      throw new JobException(
         String.format("Vertex %s's parallelism (%s) is higher than the max parallelism (%s). Please lower the parallelism or increase the max parallelism.",
            jobVertex.getName(),
            numTaskVertices,
            maxParallelism));
   }

   this.parallelism = numTaskVertices;

   this.taskVertices = new ExecutionVertex[numTaskVertices];
   this.operatorIDs = Collections.unmodifiableList(jobVertex.getOperatorIDs());
   this.userDefinedOperatorIds = Collections.unmodifiableList(jobVertex.getUserDefinedOperatorIDs());

   this.inputs = new ArrayList<>(jobVertex.getInputs().size());

   // take the sharing group
   this.slotSharingGroup = jobVertex.getSlotSharingGroup();
   this.coLocationGroup = jobVertex.getCoLocationGroup();

   // setup the coLocation group
   if (coLocationGroup != null && slotSharingGroup == null) {
      throw new JobException("Vertex uses a co-location constraint without using slot sharing");
   }

   // create the intermediate results
   this.producedDataSets = new IntermediateResult[jobVertex.getNumberOfProducedIntermediateDataSets()];

    //根据jobVertex产出的IntermediateDataSet来生成IntermediateResult
    //可见IntermediateDataSet和IntermediateResult是一一对应的关系
   for (int i = 0; i < jobVertex.getProducedDataSets().size(); i++) {
      final IntermediateDataSet result = jobVertex.getProducedDataSets().get(i);

      this.producedDataSets[i] = new IntermediateResult(
            result.getId(),
            this,
            numTaskVertices,
            result.getResultType());
   }

   Configuration jobConfiguration = graph.getJobConfiguration();
   int maxPriorAttemptsHistoryLength = jobConfiguration != null ?
         jobConfiguration.getInteger(JobManagerOptions.MAX_ATTEMPTS_HISTORY_SIZE) :
         JobManagerOptions.MAX_ATTEMPTS_HISTORY_SIZE.defaultValue();

   // create all task vertices
   //有多少个task就会生成多少个ExecutionVertex
   for (int i = 0; i < numTaskVertices; i++) {
      ExecutionVertex vertex = new ExecutionVertex(
            this,
            i,
            producedDataSets,
            timeout,
            initialGlobalModVersion,
            createTimestamp,
            maxPriorAttemptsHistoryLength);

      this.taskVertices[i] = vertex;
   }

   // sanity check for the double referencing between intermediate result partitions and execution vertices
   for (IntermediateResult ir : this.producedDataSets) {
      if (ir.getNumberOfAssignedPartitions() != parallelism) {
         throw new RuntimeException("The intermediate result's partitions were not correctly assigned.");
      }
   }

   // set up the input splits, if the vertex has any
   try {
      @SuppressWarnings("unchecked")
      InputSplitSource<InputSplit> splitSource = (InputSplitSource<InputSplit>) jobVertex.getInputSplitSource();

      if (splitSource != null) {
         Thread currentThread = Thread.currentThread();
         ClassLoader oldContextClassLoader = currentThread.getContextClassLoader();
         currentThread.setContextClassLoader(graph.getUserClassLoader());
         try {
            inputSplits = splitSource.createInputSplits(numTaskVertices);

            if (inputSplits != null) {
               splitAssigner = splitSource.getInputSplitAssigner(inputSplits);
            }
         } finally {
            currentThread.setContextClassLoader(oldContextClassLoader);
         }
      }
      else {
         inputSplits = null;
      }
   }
   catch (Throwable t) {
      throw new JobException("Creating the input splits caused an error: " + t.getMessage(), t);
   }
}
```
与执行图相关几个重要的成员

numTaskVertices：可以解释为当前的ExecutionJobVertex会产生多少个Task，这和JobVertex的并行度是一致的。

taskVertices：类型ExecutionVertex[]数组，数组的个数是Task个数是一致的。ExecutionVertex对应的就是单个Task的顶点了，我们这里吧ExecutionVertex称作二级顶点，ExecutionJobVertex是一级顶点，对应的是Job的一个拓扑单元。

inputs：类型List<IntermediateResult>，ExecutionGraph中只有ExecutionJobVertex和IntermediateResult这两个一级大类，并没有边的概念，所以下游ExecutionJobVertex的input就是IntermediateResult，IntermediateResult由上游的ExecutionJobVertex产生

producedDataSets：类型IntermediateResult[]，ExecutionJobVertex产生的IntermediateResult集合，ExecutionJobVertex作为IntermediateResult的生产者。IntermediateResult[]数组的个数和JobGraph中JobVertex产出的IntermediateDataSet是一致的，也就是说IntermediateResult和IntermediateDataSet是一一对应的

由此，构造方法的实现大致为：
- 根据jobVertex产出的IntermediateDataSet来生成IntermediateResult，IntermediateResult作为ExecutionJobVertex的产出。可见IntermediateDataSet和IntermediateResult是一一对应的关系
- 有多少个task就会生成多少个ExecutionVertex，ExecutionVertex对应的是Task的顶点

来看看IntermediateResult和ExecutionVertex的构造方法。

## IntermediateResult
首先来看IntermediateResult
```
public IntermediateResult(
      IntermediateDataSetID id,
      ExecutionJobVertex producer,
      int numParallelProducers,
      ResultPartitionType resultType) {

   this.id = checkNotNull(id);
   this.producer = checkNotNull(producer);

   checkArgument(numParallelProducers >= 1);
   this.numParallelProducers = numParallelProducers;

   this.partitions = new IntermediateResultPartition[numParallelProducers];

   this.numberOfRunningProducers = new AtomicInteger(numParallelProducers);

   // we do not set the intermediate result partitions here, because we let them be initialized by
   // the execution vertex that produces them

   // assign a random connection index
   this.connectionIndex = (int) (Math.random() * Integer.MAX_VALUE);

   // The runtime type for this produced result
   this.resultType = checkNotNull(resultType);
}
```
与执行图相关的重要成员：

producer：类型ExecutionJobVertex，中间结果数据集IntermediateResult的生产者

numParallelProducers：类型int，有多少个并行生产者，数量等于ExecutionJobVertex的并行度，即Task的个数

partitions：类型IntermediateResultPartition[]，中间结果数据集IntermediateResult的分区集合，在逻辑上将IntermediateResult进行了分区，分区个数就是ExecutionJobVertex的并行度，即Task的个数

## ExecutionVertex
下面看ExecutionVertex的构造方法
```
public ExecutionVertex(
      ExecutionJobVertex jobVertex,
      int subTaskIndex,
      IntermediateResult[] producedDataSets,
      Time timeout,
      long initialGlobalModVersion,
      long createTimestamp,
      int maxPriorExecutionHistoryLength) {

   this.jobVertex = jobVertex;
   this.subTaskIndex = subTaskIndex;
   this.taskNameWithSubtask = String.format("%s (%d/%d)",
         jobVertex.getJobVertex().getName(), subTaskIndex + 1, jobVertex.getParallelism());

   this.resultPartitions = new LinkedHashMap<>(producedDataSets.length, 1);
    //给ExecutionVertex设置输出结果IntermediateResultPartition
   for (IntermediateResult result : producedDataSets) {
      IntermediateResultPartition irp = new IntermediateResultPartition(result, this, subTaskIndex);
      result.setPartition(subTaskIndex, irp);

      resultPartitions.put(irp.getPartitionId(), irp);
   }

   this.inputEdges = new ExecutionEdge[jobVertex.getJobVertex().getInputs().size()][];

   this.priorExecutions = new EvictingBoundedList<>(maxPriorExecutionHistoryLength);

   this.currentExecution = new Execution(
      getExecutionGraph().getFutureExecutor(),
      this,
      0,
      initialGlobalModVersion,
      createTimestamp,
      timeout);

   // create a co-location scheduling hint, if necessary
   CoLocationGroup clg = jobVertex.getCoLocationGroup();
   if (clg != null) {
      this.locationConstraint = clg.getLocationConstraint(subTaskIndex);
   }
   else {
      this.locationConstraint = null;
   }

   getExecutionGraph().registerExecution(currentExecution);

   this.timeout = timeout;
}
```
与执行图相关的重要成员

subTaskIndex：类型int，代表当前的ExecutionVertex对应的是第几个Task

resultPartitions ：类型Map<IntermediateResultPartitionID, IntermediateResultPartition>，ExecutionVertex输出结果分区集合。IntermediateResultPartition是ExecutionVertex的输出结果

inputEdges ：类型ExecutionEdge[][]，ExecutionVertex的input集合，虽然一级ExecutionJobVertex没有边的概念，但是二级顶点ExecutionVertex就有边ExecutionEdge，和JobGraph中的结构类似。因为下游一个task的数据往往都来自上游所有的task的输出结果，所以ExecutionEdge一般会有多个。为什么inputEdges是二维的呢，因为ExecutionJobVertex可能有多个inputs IntermediateResult，每个input IntermediateResult又有多个分区。所以就是二维的了，第一维对应的多少个inputs，第二维对应的是每个input IntermediateResult的多少个分区

currentExecution ：类型Execution，是ExecutionVertex对应Task的物理执行

ExecutionVertex中构造方法的实现大致为：
- 根据ExecutionJobVertex的producedDataSets和当前ExecutionVertex的subTaskIndex，来创建当前ExecutionVertex应该输出的producedDataSets的分区，即IntermediateResultPartition，同时给producedDataSets添加分区对象
- 为Task创建一个物理执行Execution

上述ExecutionVertex构造方法创建了两个新的结构，IntermediateResultPartition和Execution。

同样看看他们的结构

## IntermediateResultPartition
首先看IntermediateResultPartition
```
public IntermediateResultPartition(IntermediateResult totalResult, ExecutionVertex producer, int partitionNumber) {
   this.totalResult = totalResult;
   this.producer = producer;
   this.partitionNumber = partitionNumber;
   this.consumers = new ArrayList<List<ExecutionEdge>>(0);
   this.partitionId = new IntermediateResultPartitionID();
}
```
和执行图相关的重要成员：

producer ：类型ExecutionVertex，即IntermediateResultPartition的生产者是ExecutionVertex

partitionNumber ：类型int，分区号，即是IntermediateResult的第几个分区

consumers ：类型是ArrayList<List<ExecutionEdge>>，是一个二维List。代表该分区数据的消费者，为什么是二维的呢，因为IntermediateResult可能有多个下游ExecutionJobVertex，那么每个ExecutionJobVertex都会有一组消费者来消费IntermediateResultPartition，所以consumers的第一维就是针对每个ExecutionJobVertex的

## Execution
再来看看Execution的结构
```
public Execution(
      Executor executor,
      ExecutionVertex vertex,
      int attemptNumber,
      long globalModVersion,
      long startTimestamp,
      Time rpcTimeout) {

   this.executor = checkNotNull(executor);
   this.vertex = checkNotNull(vertex);
   this.attemptId = new ExecutionAttemptID();
   this.rpcTimeout = checkNotNull(rpcTimeout);

   this.globalModVersion = globalModVersion;
   this.attemptNumber = attemptNumber;

   this.stateTimestamps = new long[ExecutionState.values().length];
   markTimestamp(CREATED, startTimestamp);

   this.partialInputChannelDeploymentDescriptors = new ConcurrentLinkedQueue<>();
   this.terminalStateFuture = new CompletableFuture<>();
   this.releaseFuture = new CompletableFuture<>();
   this.taskManagerLocationFuture = new CompletableFuture<>();

   this.assignedResource = null;
}
```
到此，attachJobGraph()方法中的第一步ExecutionJobVertex的构造算是完成了。

总结一下：

ExecutionJobVertex创建了中间结果数据集IntermediateResult和每个Task对应的顶点ExecutionVertex，ExecutionVertex构建时又创建了中间结果数据集的分区IntermediateResultPartition和Task的物理执行Execution

## 连接执行图
下面来看attachJobGraph()方法中的第二步connectToPredecessors()
```
public void connectToPredecessors(Map<IntermediateDataSetID, IntermediateResult> intermediateDataSets) throws JobException {

   List<JobEdge> inputs = jobVertex.getInputs();

   if (LOG.isDebugEnabled()) {
      LOG.debug(String.format("Connecting ExecutionJobVertex %s (%s) to %d predecessors.", jobVertex.getID(), jobVertex.getName(), inputs.size()));
   }

    //遍历jobVertex的input JobEdge 
   for (int num = 0; num < inputs.size(); num++) {
      JobEdge edge = inputs.get(num);

      if (LOG.isDebugEnabled()) {
         if (edge.getSource() == null) {
            LOG.debug(String.format("Connecting input %d of vertex %s (%s) to intermediate result referenced via ID %s.",
                  num, jobVertex.getID(), jobVertex.getName(), edge.getSourceId()));
         } else {
            LOG.debug(String.format("Connecting input %d of vertex %s (%s) to intermediate result referenced via predecessor %s (%s).",
                  num, jobVertex.getID(), jobVertex.getName(), edge.getSource().getProducer().getID(), edge.getSource().getProducer().getName()));
         }
      }

      // fetch the intermediate result via ID. if it does not exist, then it either has not been created, or the order
      // in which this method is called for the job vertices is not a topological order
      //获取input IntermediateResult，即ExecutionJobVertex的数据源source
      IntermediateResult ires = intermediateDataSets.get(edge.getSourceId());
      if (ires == null) {
         throw new JobException("Cannot connect this job graph to the previous graph. No previous intermediate result found for ID "
               + edge.getSourceId());
      }

      this.inputs.add(ires);
      //IntermediateResult添加消费者组
      int consumerIndex = ires.registerConsumer();

      for (int i = 0; i < parallelism; i++) {
         ExecutionVertex ev = taskVertices[i];
         //给IntermediateResult添加消费者ExecutionEdge，给ExecutionVertex添加inputs ExecutionEdge
         ev.connectSource(num, ires, edge, consumerIndex);
      }
   }
}
```
上述代码的大致实现为：
- 获取JobVertex的所有input JobEdge
- 遍历所有JobEdge，针对每个JobEdge获取ExecutionJobVertex的数据源IntermediateResult，当做ExecutionJobVertex的输入input。
- 既然IntermediateResult作为ExecutionJobVertex的input了，就给该IntermediateResult添加一个消费者组，具体来说是给每个分区IntermediateResultPartition 添加消费者组，但这时并没有添加具体的消费者ExecutionEdge。
- 针对ExecutionJobVertex的每个ExecutionVertex，给ExecutionVertex添加inputs ExecutionEdge，并且相应的给IntermediateResult中的每个分区IntermediateResultPartition添加消费者

IntermediateResult注册消费者组的代码在registerConsumer()。其实现就是对IntermediateResult的每个分区IntermediateResultPartition 添加消费者组，但这时并没有添加具体的消费者ExecutionEdge
```
public int registerConsumer() {
   final int index = numConsumers;
   numConsumers++;

   for (IntermediateResultPartition p : partitions) {
      if (p.addConsumerGroup() != index) {
         throw new RuntimeException("Inconsistent consumer mapping between intermediate result partitions.");
      }
   }
   return index;
}

//IntermediateResultPartition类
int addConsumerGroup() {
   int pos = consumers.size();

   // NOTE: currently we support only one consumer per result!!!
   if (pos != 0) {
      throw new RuntimeException("Currently, each intermediate result can only have one consumer.");
   }

   consumers.add(new ArrayList<ExecutionEdge>());
   return pos;
}
```
具体的给IntermediateResultPartition 添加消费者并且给ExecutionVertex添加input的实现在ExecutionVertex.connectSource()
```
public void connectSource(int inputNumber, IntermediateResult source, JobEdge edge, int consumerNumber) {

   final DistributionPattern pattern = edge.getDistributionPattern();
   final IntermediateResultPartition[] sourcePartitions = source.getPartitions();

   ExecutionEdge[] edges;

   switch (pattern) {
      case POINTWISE:
         edges = connectPointwise(sourcePartitions, inputNumber);
         break;

      case ALL_TO_ALL:
          //获取ExecutionVertex的所有input ExecutionEdge
         edges = connectAllToAll(sourcePartitions, inputNumber);
         break;

      default:
         throw new RuntimeException("Unrecognized distribution pattern.");

   }

   this.inputEdges[inputNumber] = edges;

   // add the consumers to the source
   // for now (until the receiver initiated handshake is in place), we need to register the
   // edges as the execution graph
   //将ExecutionEdge作为消费者添加到ExecutionEdge的数据源IntermediateResultPartition的consumers中
   for (ExecutionEdge ee : edges) {
      ee.getSource().addConsumer(ee, consumerNumber);
   }
}

private ExecutionEdge[] connectAllToAll(IntermediateResultPartition[] sourcePartitions, int inputNumber) {
   ExecutionEdge[] edges = new ExecutionEdge[sourcePartitions.length];
    //数据源IntermediateResult有多少个分区，就会创建多少条边，这个因为下游Task的数据来自于上游所有task的输出，
    //也就是说数据源IntermediateResult中的每个分区都会被下游的单个ExecutionVertex消费
   for (int i = 0; i < sourcePartitions.length; i++) {
      IntermediateResultPartition irp = sourcePartitions[i];
      edges[i] = new ExecutionEdge(irp, this, inputNumber);
   }

   return edges;
}

//IntermediateResultPartition类
void addConsumer(ExecutionEdge edge, int consumerNumber) {
   consumers.get(consumerNumber).add(edge);
}
```
connectSource()的实现大致如下：
- 获取ExecutionVertex的所有input ExecutionEdge，ExecutionJobVertex的input IntermediateResult有多少个分区，就会创建多少条边ExecutionEdge，这个因为下游Task的数据来自于上游所有task的输出，也就是说数据源IntermediateResult中的每个分区都会被下游的单个ExecutionVertex消费。IntermediateResult分区的个数和上游Task的并行度一致
- 将ExecutionEdge作为消费者添加到ExecutionEdge的数据源IntermediateResultPartition的consumers中。
- 经过connectSource()实现后，拓扑逻辑大概是IntermediateResultPartition-->ExecutionEdge-->ExecutionVertex

## ExecutionEdge
上述connectSource()方法创建了ExecutionEdge，看下ExecutionEdge的结构
```
public class ExecutionEdge {

   private final IntermediateResultPartition source;

   private final ExecutionVertex target;

   private final int inputNum;

   public ExecutionEdge(IntermediateResultPartition source, ExecutionVertex target, int inputNum) {
      this.source = source;
      this.target = target;
      this.inputNum = inputNum;
   }
```
与执行图相关的成员：

source：类型IntermediateResultPartition ，ExecutionEdge的数据源source

target：类型ExecutionVertex ，ExecutionEdge的终端

到此，attachJobGraph()方法中的第二步connectToPredecessors()也就完成了，此时，一个ExecutionJobVertex的创建便完成了。接下来对JobGraph中每个JobVertex都做重复的操作，最后，整个ExecutionGrap便完成了。


## 总结：

ExecutionGraph构建的大致步骤
- ExecutionGraph是JobMaster的成员。构建发生在JobMaster的构造方法中，最终会调用ExecutionGraphBuilder.buildGraph()进行构建

- buildGraph()会构造一个新的ExecutionGraph，按照Job的拓扑顺序排序的JobGraph的所有顶点JobVertex，根据JobGraph的所有顶点JobVertex来绑定ExecutionGraph的各种内部属性，核心方法在attachJobGraph()中

- 针对JobGraph中每个顶点jobVertex生成一个ExecutionGraph的顶点ExecutionJobVertex，JobVertex和ExecutionJobVertex是一一对应关系。在ExecutionJobVertex的构造方法中会根据jobVertex产出的IntermediateDataSet来生成IntermediateResult，IntermediateResult作为ExecutionJobVertex的产出。IntermediateDataSet和IntermediateResult也是一一对应的关系

- 创建ExecutionVertex，根据jobVertex的并行度，有多少个并行度就会生成多少个ExecutionVertex，ExecutionVertex的概念可以理解为对应的是Task的顶点

- ExecutionVertex中构造方法又会根据ExecutionJobVertex的producedDataSets（即IntermediateResult）和当前ExecutionVertex的subTaskIndex，来创建当前ExecutionVertex输出的IntermediateResult的分区，即IntermediateResultPartition，同时给IntermediateResult添加分区对象。然后会为Task创建一个物理执行Execution

- 执行attachJobGraph()的第二部分，连接ExecutionEdge、ExecutionVertex、IntermediateResultPartition

- 遍历JobVertex所有JobEdge，针对每个JobEdge获取ExecutionJobVertex的数据源IntermediateResult，当做ExecutionJobVertex的输入input。然后给该IntermediateResult添加一个消费者组，具体来说是给每个分区IntermediateResultPartition 添加消费者组，但这时并没有添加具体的消费者ExecutionEdge。

- 针对ExecutionJobVertex的每个ExecutionVertex添加inputs ExecutionEdge，并且相应的给IntermediateResult中的每个分区IntermediateResultPartition添加消费者ExecutionEdge。具体实现在ExecutionVertex.connectSource()中。

到此，ExecutionGraph构建完毕






















