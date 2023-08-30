---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink源码Task任务启动
TaskExecutor在创建Task实例之后就会调用task.startTaskThread()开始启动Task，Task实例本身实现了Runable接口，每个Task都会启动一个线程Thread来运行自身。

## 起点
即调用Task的run()方法。
```
public void run() {

     ...
     AbstractInvokable invokable = null;
     ...
      Environment env = new RuntimeEnvironment(
         jobId,
         vertexId,
         executionId,
         executionConfig,
         taskInfo,
         jobConfiguration,
         taskConfiguration,
         userCodeClassLoader,
         memoryManager,
         ioManager,
         broadcastVariableManager,
         taskStateManager,
         aggregateManager,
         accumulatorRegistry,
         kvStateRegistry,
         inputSplitProvider,
         distributedCacheEntries,
         producedPartitions,
         inputGates,
         network.getTaskEventDispatcher(),
         checkpointResponder,
         taskManagerConfig,
         metrics,
         this);

      // now load and instantiate the task's invokable code
      invokable = loadAndInstantiateInvokable(userCodeClassLoader, nameOfInvokableClass, env);

      // ----------------------------------------------------------------
      //  actual task core work
      // ----------------------------------------------------------------

      // we must make strictly sure that the invokable is accessible to the cancel() call
      // by the time we switched to running.
      this.invokable = invokable;

      // switch to the RUNNING state, if that fails, we have been canceled/failed in the meantime
      if (!transitionState(ExecutionState.DEPLOYING, ExecutionState.RUNNING)) {
         throw new CancelTaskException();
      }

      // notify everyone that we switched to running
      taskManagerActions.updateTaskExecutionState(new TaskExecutionState(jobId, executionId, ExecutionState.RUNNING));

      // make sure the user code classloader is accessible thread-locally
      executingThread.setContextClassLoader(userCodeClassLoader);

      // run the invokable
      invokable.invoke();

     ...
}
```
run()方法中最核心的就是创建Task的AbstractInvokable，然后调用invokable.invoke()执行Task。一般的这里的AbstractInvokable就是StreamTask。

## 构造StreamTask
首先我们来看看StreamTask的构造方法，因为loadAndInstantiateInvokable()会实例化这个StreamTask。
```
protected StreamTask(
      Environment environment,
      @Nullable ProcessingTimeService timeProvider) {

   super(environment);

   this.timerService = timeProvider;
   this.configuration = new StreamConfig(getTaskConfiguration());
   this.accumulatorMap = getEnvironment().getAccumulatorRegistry().getUserMap();
   this.recordWriters = createRecordWriters(configuration, environment);
}
```
构造方法里最主要就是创建recordWriters。看下它是怎么创建的。

```
public static <OUT> List<RecordWriter<SerializationDelegate<StreamRecord<OUT>>>> createRecordWriters(
      StreamConfig configuration,
      Environment environment) {
   List<RecordWriter<SerializationDelegate<StreamRecord<OUT>>>> recordWriters = new ArrayList<>();
   //这个outEdgesInOrder就是构建JobGraph时的过渡边
   List<StreamEdge> outEdgesInOrder = configuration.getOutEdgesInOrder(environment.getUserClassLoader());
   Map<Integer, StreamConfig> chainedConfigs = configuration.getTransitiveChainedTaskConfigsWithSelf(environment.getUserClassLoader());
    //针对每个过渡边都创建一个RecordWriter
   for (int i = 0; i < outEdgesInOrder.size(); i++) {
      StreamEdge edge = outEdgesInOrder.get(i);
      recordWriters.add(
         createRecordWriter(
            edge,
            i,
            environment,
            environment.getTaskInfo().getTaskName(),
            chainedConfigs.get(edge.getSourceId()).getBufferTimeout()));
   }
   return recordWriters;
}


private static <OUT> RecordWriter<SerializationDelegate<StreamRecord<OUT>>> createRecordWriter(
      StreamEdge edge,
      int outputIndex,
      Environment environment,
      String taskName,
      long bufferTimeout) {
   @SuppressWarnings("unchecked")
   StreamPartitioner<OUT> outputPartitioner = (StreamPartitioner<OUT>) edge.getPartitioner();

   LOG.debug("Using partitioner {} for output {} of task {}", outputPartitioner, outputIndex, taskName);

   ResultPartitionWriter bufferWriter = environment.getWriter(outputIndex);

   // we initialize the partitioner here with the number of key groups (aka max. parallelism)
   if (outputPartitioner instanceof ConfigurableStreamPartitioner) {
      int numKeyGroups = bufferWriter.getNumTargetKeyGroups();
      if (0 < numKeyGroups) {
         ((ConfigurableStreamPartitioner) outputPartitioner).configure(numKeyGroups);
      }
   }

   RecordWriter<SerializationDelegate<StreamRecord<OUT>>> output =
      RecordWriter.createRecordWriter(bufferWriter, outputPartitioner, bufferTimeout, taskName);
   output.setMetricGroup(environment.getMetricGroup().getIOMetricGroup());
   return output;
}

//RecordWriter类，它持有channelSelector，这个就是partitioner
//writer就是构建Task创建的ResultPartition
public static RecordWriter createRecordWriter(
      ResultPartitionWriter writer,
      ChannelSelector channelSelector,
      long timeout,
      String taskName) {
   if (channelSelector.isBroadcast()) {
      return new BroadcastRecordWriter<>(writer, channelSelector, timeout, taskName);
   } else {
      return new RecordWriter<>(writer, channelSelector, timeout, taskName);
   }
}
```
上述代码针对每个过渡边都创建一个RecordWriter，过渡边即在构建JobGraph时的过渡边，即不能再往下进行chain的那条StreamEdge。RecordWriter它持有channelSelector，这个就是partitioner；writer就是构建Task创建的ResultPartition，ResultPartition是ResultPartitionWriter的实现子类。RecordWriter用于创建RecordWriterOutput

## invoke()核心方法
下面再来看看invoke()核心方法：
```
public final void invoke() throws Exception {

   boolean disposed = false;
      ...
      //创建stateBackend 
      stateBackend = createStateBackend();
      checkpointStorage = stateBackend.createCheckpointStorage(getEnvironment().getJobID());

      // if the clock is not already set, then assign a default TimeServiceProvider
      if (timerService == null) {
         ThreadFactory timerThreadFactory = new DispatcherThreadFactory(TRIGGER_THREAD_GROUP,
            "Time Trigger for " + getName(), getUserCodeClassLoader());

         timerService = new SystemProcessingTimeService(this, getCheckpointLock(), timerThreadFactory);
      }
    
    //构造operator chain，同时会给operator设置output
      operatorChain = new OperatorChain<>(this, recordWriters);
      headOperator = operatorChain.getHeadOperator();

      // task specific initialization
      //进行Task初始化操作
      init();
      
    ...
    
      // we need to make sure that any triggers scheduled in open() cannot be
      // executed before all operators are opened
      synchronized (lock) {

         // both the following operations are protected by the lock
         // so that we avoid race conditions in the case that initializeState()
         // registers a timer, that fires before the open() is called.
        //operator的初始化方法
         initializeState();
         //打开所有的operator，会调用operator的open方法
         openAllOperators();
      }

     ...

      // let the task do its work
      isRunning = true;
      //正式开始运行Task
      run();
      ...

}
```
下面来逐步分析invoke()中的实现。

## 创建StateBackend
首先是创建stateBackend，
```
private StateBackend createStateBackend() throws Exception {
   final StateBackend fromApplication = configuration.getStateBackend(getUserCodeClassLoader());

   return StateBackendLoader.fromApplicationOrConfigOrDefault(
         fromApplication,
         getEnvironment().getTaskManagerInfo().getConfiguration(),
         getUserCodeClassLoader(),
         LOG);
}
public static StateBackend fromApplicationOrConfigOrDefault(
      @Nullable StateBackend fromApplication,
      Configuration config,
      ClassLoader classLoader,
      @Nullable Logger logger) throws IllegalConfigurationException, DynamicCodeLoadingException, IOException {

   checkNotNull(config, "config");
   checkNotNull(classLoader, "classLoader");

   final StateBackend backend;

   // (1) the application defined state backend has precedence
   if (fromApplication != null) {
      if (logger != null) {
         logger.info("Using application-defined state backend: {}", fromApplication);
      }

      // see if this is supposed to pick up additional configuration parameters
      if (fromApplication instanceof ConfigurableStateBackend) {
         // needs to pick up configuration
         if (logger != null) {
            logger.info("Configuring application-defined state backend with job/cluster config");
         }

         backend = ((ConfigurableStateBackend) fromApplication).configure(config, classLoader);
      }
      else {
         // keep as is!
         backend = fromApplication;
      }
   }
   else {
      // (2) check if the config defines a state backend
      final StateBackend fromConfig = loadStateBackendFromConfig(config, classLoader, logger);
      if (fromConfig != null) {
         backend = fromConfig;
      }
      else {
         // (3) use the default
         backend = new MemoryStateBackendFactory().createFromConfig(config, classLoader);
         if (logger != null) {
            logger.info("No state backend has been configured, using default (Memory / JobManager) {}", backend);
         }
      }
   }
   ...
   return backend;
}
```
可以看到，如果用户没有设置StateBackend，那么就会默认创建MemoryStateBackend，flink内部可用的StateBackend包括MemoryStateBackend、FsStateBackend，此外还有在其他包里实现的

RockDBStateBackend。

## 创建operator chain
代码实现在OperatorChain的构造方法中
```
public OperatorChain(
      StreamTask<OUT, OP> containingTask,
      List<RecordWriter<SerializationDelegate<StreamRecord<OUT>>>> recordWriters) {

   final ClassLoader userCodeClassloader = containingTask.getUserCodeClassLoader();
   final StreamConfig configuration = containingTask.getConfiguration();

    //首先通过configuration反序列化获取headOperator
   headOperator = configuration.getStreamOperator(userCodeClassloader);

   // we read the chained configs, and the order of record writer registrations by output name
   Map<Integer, StreamConfig> chainedConfigs = configuration.getTransitiveChainedTaskConfigsWithSelf(userCodeClassloader);

   // create the final output stream writers
   // we iterate through all the out edges from this job vertex and create a stream output
   //这里的outEdgesInOrder就是创建JobGraph时的过渡边
   List<StreamEdge> outEdgesInOrder = configuration.getOutEdgesInOrder(userCodeClassloader);
   Map<StreamEdge, RecordWriterOutput<?>> streamOutputMap = new HashMap<>(outEdgesInOrder.size());
   this.streamOutputs = new RecordWriterOutput<?>[outEdgesInOrder.size()];

   // from here on, we need to make sure that the output writers are shut down again on failure
   boolean success = false;
   try {
       //对每一个过渡边都创建一个RecordWriterOutput
      for (int i = 0; i < outEdgesInOrder.size(); i++) {
         StreamEdge outEdge = outEdgesInOrder.get(i);

         RecordWriterOutput<?> streamOutput = createStreamOutput(
            recordWriters.get(i),
            outEdge,
            chainedConfigs.get(outEdge.getSourceId()),
            containingTask.getEnvironment());

         this.streamOutputs[i] = streamOutput;
         streamOutputMap.put(outEdge, streamOutput);
      }

      // we create the chain of operators and grab the collector that leads into the chain
      List<StreamOperator<?>> allOps = new ArrayList<>(chainedConfigs.size());
      //递归的创建chain上的每个operator，并设置operator的output，
      //递归方法中先创建chain末尾的operator，再创建前面的operator
      this.chainEntryPoint = createOutputCollector(
         containingTask,
         configuration,
         chainedConfigs,
         userCodeClassloader,
         streamOutputMap,
         allOps);

      if (headOperator != null) {
         WatermarkGaugeExposingOutput<StreamRecord<OUT>> output = getChainEntryPoint();
         //给headOperator设置output
         headOperator.setup(containingTask, configuration, output);

         headOperator.getMetricGroup().gauge(MetricNames.IO_CURRENT_OUTPUT_WATERMARK, output.getWatermarkGauge());
      }

      // add head operator to end of chain
      allOps.add(headOperator);

      this.allOperators = allOps.toArray(new StreamOperator<?>[allOps.size()]);

      success = true;
   }
   finally {
      // make sure we clean up after ourselves in case of a failure after acquiring
      // the first resources
      if (!success) {
         for (RecordWriterOutput<?> output : this.streamOutputs) {
            if (output != null) {
               output.close();
            }
         }
      }
   }
}
```
上述代码大致实现：

- 首先创建headOperator，headOperator是从StreamConfig中经过反序列读取的，在构造JobGraph的时候，代码会将operator序列化写到StreamConfig中。

- 根据RecordWriter和构建JobGraph时的过渡边来创建RecordWriterOutput，每一个过渡边都创建一个RecordWriterOutput。大部分情况下每个JobGraph的顶点只有一条过渡边，也就是一个RecordWriterOutput

- 调用createOutputCollector()方法进行递归的创建chain上的operator和output，递归方法会首先创建末尾的operator，在依次从后向前创建operator，并且给每个operator都设置output，operator的构建也是从StreamConfig中反序列获取。末尾的operator的output就是RecordWriterOutput（StreamSink除外）。

- 给headOperator设置output，并将headOperator添加到所有的operator列表中

上述代码最重要的就是步骤3中的createOutputCollector()

```
private <T> WatermarkGaugeExposingOutput<StreamRecord<T>> createOutputCollector(
      StreamTask<?, ?> containingTask,
      StreamConfig operatorConfig,
      Map<Integer, StreamConfig> chainedConfigs,
      ClassLoader userCodeClassloader,
      Map<StreamEdge, RecordWriterOutput<?>> streamOutputs,
      List<StreamOperator<?>> allOperators) {
   List<Tuple2<WatermarkGaugeExposingOutput<StreamRecord<T>>, StreamEdge>> allOutputs = new ArrayList<>(4);

   // create collectors for the network outputs
   for (StreamEdge outputEdge : operatorConfig.getNonChainedOutputs(userCodeClassloader)) {
      @SuppressWarnings("unchecked")
      RecordWriterOutput<T> output = (RecordWriterOutput<T>) streamOutputs.get(outputEdge);

      allOutputs.add(new Tuple2<>(output, outputEdge));
   }

   // Create collectors for the chained outputs
   for (StreamEdge outputEdge : operatorConfig.getChainedOutputs(userCodeClassloader)) {
      int outputId = outputEdge.getTargetId();
      StreamConfig chainedOpConfig = chainedConfigs.get(outputId);

      WatermarkGaugeExposingOutput<StreamRecord<T>> output = createChainedOperator(
         containingTask,
         chainedOpConfig,
         chainedConfigs,
         userCodeClassloader,
         streamOutputs,
         allOperators,
         outputEdge.getOutputTag());
      allOutputs.add(new Tuple2<>(output, outputEdge));
   }

   // if there are multiple outputs, or the outputs are directed, we need to
   // wrap them as one output

   List<OutputSelector<T>> selectors = operatorConfig.getOutputSelectors(userCodeClassloader);

   if (selectors == null || selectors.isEmpty()) {
      // simple path, no selector necessary
      if (allOutputs.size() == 1) {
         return allOutputs.get(0).f0;
      }
      else {
         // send to N outputs. Note that this includes the special case
         // of sending to zero outputs
         @SuppressWarnings({"unchecked", "rawtypes"})
         Output<StreamRecord<T>>[] asArray = new Output[allOutputs.size()];
         for (int i = 0; i < allOutputs.size(); i++) {
            asArray[i] = allOutputs.get(i).f0;
         }

         // This is the inverse of creating the normal ChainingOutput.
         // If the chaining output does not copy we need to copy in the broadcast output,
         // otherwise multi-chaining would not work correctly.
         if (containingTask.getExecutionConfig().isObjectReuseEnabled()) {
            return new CopyingBroadcastingOutputCollector<>(asArray, this);
         } else  {
            return new BroadcastingOutputCollector<>(asArray, this);
         }
      }
   }
   else {
      // selector present, more complex routing necessary

      // This is the inverse of creating the normal ChainingOutput.
      // If the chaining output does not copy we need to copy in the broadcast output,
      // otherwise multi-chaining would not work correctly.
      if (containingTask.getExecutionConfig().isObjectReuseEnabled()) {
         return new CopyingDirectedOutput<>(selectors, allOutputs);
      } else {
         return new DirectedOutput<>(selectors, allOutputs);
      }

   }
}


//这个方法和createOutputCollector()互为递归调用
private <IN, OUT> WatermarkGaugeExposingOutput<StreamRecord<IN>> createChainedOperator(
      StreamTask<?, ?> containingTask,
      StreamConfig operatorConfig,
      Map<Integer, StreamConfig> chainedConfigs,
      ClassLoader userCodeClassloader,
      Map<StreamEdge, RecordWriterOutput<?>> streamOutputs,
      List<StreamOperator<?>> allOperators,
      OutputTag<IN> outputTag) {
   // create the output that the operator writes to first. this may recursively create more operators
   WatermarkGaugeExposingOutput<StreamRecord<OUT>> chainedOperatorOutput = createOutputCollector(
      containingTask,
      operatorConfig,
      chainedConfigs,
      userCodeClassloader,
      streamOutputs,
      allOperators);

   // now create the operator and give it the output collector to write its output to
   OneInputStreamOperator<IN, OUT> chainedOperator = operatorConfig.getStreamOperator(userCodeClassloader);

   chainedOperator.setup(containingTask, operatorConfig, chainedOperatorOutput);

   allOperators.add(chainedOperator);

   WatermarkGaugeExposingOutput<StreamRecord<IN>> currentOperatorOutput;
   if (containingTask.getExecutionConfig().isObjectReuseEnabled()) {
      currentOperatorOutput = new ChainingOutput<>(chainedOperator, this, outputTag);
   }
   else {
      TypeSerializer<IN> inSerializer = operatorConfig.getTypeSerializerIn1(userCodeClassloader);
      currentOperatorOutput = new CopyingChainingOutput<>(chainedOperator, inSerializer, outputTag, this);
   }

   // wrap watermark gauges since registered metrics must be unique
   chainedOperator.getMetricGroup().gauge(MetricNames.IO_CURRENT_INPUT_WATERMARK, currentOperatorOutput.getWatermarkGauge()::getValue);
   chainedOperator.getMetricGroup().gauge(MetricNames.IO_CURRENT_OUTPUT_WATERMARK, chainedOperatorOutput.getWatermarkGauge()::getValue);

   return currentOperatorOutput;
}
```
上述代码会从chain的头部依次向下的遍历ChainedOutputs，即可以chain的StreamEdge，递归的进行创建下游的operator，类似于后序遍历，即先创建出末尾的operator，再依次向前创建出operator，operator的创建也是从StreamConfig中通过反序列化获取，末尾operator的output是RecordWriterOutput（除StreamSink，StreamSink的output是BroadcastingOutputCollector），其余chain上的operator的output是CopyingChainingOutput，包括headOperator。

output的作用就是输出经过operator处理之后的数据。chain中上一个operator的CopyingChainingOutput会持有下一个operator的引用，上一个operator处理完数据之后，CopyingChainingOutput会将数据推送到下一个operator处理，直至末尾的operator处理完数据，末尾operator的RecordWriterOutput会将数据发送给网络，推送到下游的Task处理。

## 初始化StreamTask
在创建完operator chain之后，invoke()的下一步就是调用init()进行Task的初始化操作。来分别看下OneInputStreamTask和SourceStreamTask的实现。

首先是OneInputStreamTask，如下：
```
public void init() throws Exception {
   StreamConfig configuration = getConfiguration();

   TypeSerializer<IN> inSerializer = configuration.getTypeSerializerIn1(getUserCodeClassLoader());
   int numberOfInputs = configuration.getNumberOfInputs();

   if (numberOfInputs > 0) {
      InputGate[] inputGates = getEnvironment().getAllInputGates();

    //创建StreamInputProcessor，它是核心的数据处理器
      inputProcessor = new StreamInputProcessor<>(
            inputGates,
            inSerializer,
            this,
            configuration.getCheckpointMode(),
            getCheckpointLock(),
            getEnvironment().getIOManager(),
            getEnvironment().getTaskManagerInfo().getConfiguration(),
            getStreamStatusMaintainer(),
            this.headOperator,
            getEnvironment().getMetricGroup().getIOMetricGroup(),
            inputWatermarkGauge);
   }
   headOperator.getMetricGroup().gauge(MetricNames.IO_CURRENT_INPUT_WATERMARK, this.inputWatermarkGauge);
   // wrap watermark gauge since registered metrics must be unique
   getEnvironment().getMetricGroup().gauge(MetricNames.IO_CURRENT_INPUT_WATERMARK, this.inputWatermarkGauge::getValue);
}
```

```
public StreamInputProcessor(
      InputGate[] inputGates,
      TypeSerializer<IN> inputSerializer,
      StreamTask<?, ?> checkpointedTask,
      CheckpointingMode checkpointMode,
      Object lock,
      IOManager ioManager,
      Configuration taskManagerConfig,
      StreamStatusMaintainer streamStatusMaintainer,
      OneInputStreamOperator<IN, ?> streamOperator,
      TaskIOMetricGroup metrics,
      WatermarkGauge watermarkGauge) throws IOException {

   InputGate inputGate = InputGateUtil.createInputGate(inputGates);
    //barrierHandler负责获取buffer，产生数据。其内部就是通过inputGate来获取的，
    //可以说inputGate负责为StreamInputProcessor提供数据输入
   this.barrierHandler = InputProcessorUtil.createCheckpointBarrierHandler(
      checkpointedTask, checkpointMode, ioManager, inputGate, taskManagerConfig);

   this.lock = checkNotNull(lock);

   StreamElementSerializer<IN> ser = new StreamElementSerializer<>(inputSerializer);
   this.deserializationDelegate = new NonReusingDeserializationDelegate<>(ser);

   // Initialize one deserializer per input channel
   this.recordDeserializers = new SpillingAdaptiveSpanningRecordDeserializer[inputGate.getNumberOfInputChannels()];

   for (int i = 0; i < recordDeserializers.length; i++) {
      recordDeserializers[i] = new SpillingAdaptiveSpanningRecordDeserializer<>(
         ioManager.getSpillingDirectoriesPaths());
   }

   this.numInputChannels = inputGate.getNumberOfInputChannels();

   this.streamStatusMaintainer = checkNotNull(streamStatusMaintainer);
   //headOperator，即operator chain的第一个operator
   this.streamOperator = checkNotNull(streamOperator);

   this.statusWatermarkValve = new StatusWatermarkValve(
         numInputChannels,
         new ForwardingValveOutputHandler(streamOperator, lock));

   this.watermarkGauge = watermarkGauge;
   metrics.gauge("checkpointAlignmentTime", barrierHandler::getAlignmentDurationNanos);
}
```
OneInputStreamTask的init()方法主要是创建StreamInputProcessor，StreamInputProcessor是它的核心数据处理器。StreamInputProcessor中持有一个barrierHandler，barrierHandler负责获取数据buffer，为StreamInputProcessor提供数据输入，其内部实现就是通过inputGate来获取的。

再来看SourceStreamTask
```
protected void init() {
   // we check if the source is actually inducing the checkpoints, rather
   // than the trigger
   SourceFunction<?> source = headOperator.getUserFunction();
   if (source instanceof ExternallyInducedSource) {
      externallyInducedCheckpoints = true;

      ExternallyInducedSource.CheckpointTrigger triggerHook = new ExternallyInducedSource.CheckpointTrigger() {

         @Override
         public void triggerCheckpoint(long checkpointId) throws FlinkException {
            // TODO - we need to see how to derive those. We should probably not encode this in the
            // TODO -   source's trigger message, but do a handshake in this task between the trigger
            // TODO -   message from the master, and the source's trigger notification
            final CheckpointOptions checkpointOptions = CheckpointOptions.forCheckpointWithDefaultLocation();
            final long timestamp = System.currentTimeMillis();

            final CheckpointMetaData checkpointMetaData = new CheckpointMetaData(checkpointId, timestamp);

            try {
               SourceStreamTask.super.triggerCheckpoint(checkpointMetaData, checkpointOptions);
            }
            catch (RuntimeException | FlinkException e) {
               throw e;
            }
            catch (Exception e) {
               throw new FlinkException(e.getMessage(), e);
            }
         }
      };

      ((ExternallyInducedSource<?, ?>) source).setCheckpointTrigger(triggerHook);
   }
}
```
SourceStreamTask的init()没做太多的东西，就是判断SourceFunction的类型如果是ExternallyInducedSource，就给它设置checkpoint相关的东西。

## 初始化operator
在invoke()方法中，接下来就是进行chain上所有operator的的初始化操作了。

首先看initializeState()操作，它的主要实现是初始化operatorStateBackend和keyedStateBackend以及keyedStateStore
```
private void initializeState() throws Exception {

   StreamOperator<?>[] allOperators = operatorChain.getAllOperators();

   for (StreamOperator<?> operator : allOperators) {
      if (null != operator) {
         operator.initializeState();
      }
   }
}

//AbstractStreamOperator类
public final void initializeState() throws Exception {

   final TypeSerializer<?> keySerializer = config.getStateKeySerializer(getUserCodeClassloader());

   final StreamTask<?, ?> containingTask =
      Preconditions.checkNotNull(getContainingTask());
   final CloseableRegistry streamTaskCloseableRegistry =
      Preconditions.checkNotNull(containingTask.getCancelables());
   final StreamTaskStateInitializer streamTaskStateManager =
      Preconditions.checkNotNull(containingTask.createStreamTaskStateInitializer());

   final StreamOperatorStateContext context =
      streamTaskStateManager.streamOperatorStateContext(
         getOperatorID(),
         getClass().getSimpleName(),
         this,
         keySerializer,
         streamTaskCloseableRegistry,
         metrics);

   this.operatorStateBackend = context.operatorStateBackend();
   this.keyedStateBackend = context.keyedStateBackend();

   if (keyedStateBackend != null) {
      this.keyedStateStore = new DefaultKeyedStateStore(keyedStateBackend, getExecutionConfig());
   }

   timeServiceManager = context.internalTimerServiceManager();

   CloseableIterable<KeyGroupStatePartitionStreamProvider> keyedStateInputs = context.rawKeyedStateInputs();
   CloseableIterable<StatePartitionStreamProvider> operatorStateInputs = context.rawOperatorStateInputs();

   try {
      StateInitializationContext initializationContext = new StateInitializationContextImpl(
         context.isRestored(), // information whether we restore or start for the first time
         operatorStateBackend, // access to operator state backend
         keyedStateStore, // access to keyed state backend
         keyedStateInputs, // access to keyed state stream
         operatorStateInputs); // access to operator state stream

      initializeState(initializationContext);
   } finally {
      closeFromRegistry(operatorStateInputs, streamTaskCloseableRegistry);
      closeFromRegistry(keyedStateInputs, streamTaskCloseableRegistry);
   }
}
```
接下来看openAllOperators()方法，它的实现就是调用operator.open()方法，对于我们的大部分map、flatMap算子等AbstractUdfStreamOperator，它的open就是调用RichFunction的open()方法

```
private void openAllOperators() throws Exception {
   for (StreamOperator<?> operator : operatorChain.getAllOperators()) {
      if (operator != null) {
         operator.open();
      }
   }
}

//AbstractUdfStreamOperator类
public void open() throws Exception {
   super.open();
   FunctionUtils.openFunction(userFunction, new Configuration());
}

public static void openFunction(Function function, Configuration parameters) throws Exception{
   if (function instanceof RichFunction) {
      RichFunction richFunction = (RichFunction) function;
      richFunction.open(parameters);
   }
}
```

## 开始执行Task
在进行完operator的初始化方法之后，接下来就是invoke()中执行run()方法了，终于到了run()方法，这也意味着Task终于开始执行了！

首先来看OneInputStreamTask的run()方法
```
protected void run() throws Exception {
   // cache processor reference on the stack, to make the code more JIT friendly
   final StreamInputProcessor<IN> inputProcessor = this.inputProcessor;

   while (running && inputProcessor.processInput()) {
      // all the work happens in the "processInput" method
   }
}
```
对于OneInputStreamTask就是调用核心数据处理器inputProcessor.processInput()方法来进行数据的处理。

再来看看SourceStreamTask的run()方法：
```
protected void run() throws Exception {
   headOperator.run(getCheckpointLock(), getStreamStatusMaintainer());
}
```
调用的headOperator.run()方法，在SourceStreamTask中，headOperator是StreamSource，看它的run()是干嘛的。
```
public void run(final Object lockingObject, final StreamStatusMaintainer streamStatusMaintainer) throws Exception {
   run(lockingObject, streamStatusMaintainer, output);
}

public void run(final Object lockingObject,
      final StreamStatusMaintainer streamStatusMaintainer,
      final Output<StreamRecord<OUT>> collector) throws Exception {

   final TimeCharacteristic timeCharacteristic = getOperatorConfig().getTimeCharacteristic();

   final Configuration configuration = this.getContainingTask().getEnvironment().getTaskManagerInfo().getConfiguration();
   final long latencyTrackingInterval = getExecutionConfig().isLatencyTrackingConfigured()
      ? getExecutionConfig().getLatencyTrackingInterval()
      : configuration.getLong(MetricOptions.LATENCY_INTERVAL);

   LatencyMarksEmitter<OUT> latencyEmitter = null;
   if (latencyTrackingInterval > 0) {
      latencyEmitter = new LatencyMarksEmitter<>(
         getProcessingTimeService(),
         collector,
         latencyTrackingInterval,
         this.getOperatorID(),
         getRuntimeContext().getIndexOfThisSubtask());
   }

   final long watermarkInterval = getRuntimeContext().getExecutionConfig().getAutoWatermarkInterval();

   this.ctx = StreamSourceContexts.getSourceContext(
      timeCharacteristic,
      getProcessingTimeService(),
      lockingObject,
      streamStatusMaintainer,
      collector,
      watermarkInterval,
      -1);

   try {
       //核心就是调用用户的SourceFunction.run()方法，ctx中持有output
      userFunction.run(ctx);

      // if we get here, then the user function either exited after being done (finite source)
      // or the function was canceled or stopped. For the finite source case, we should emit
      // a final watermark that indicates that we reached the end of event-time
      if (!isCanceledOrStopped()) {
         ctx.emitWatermark(Watermark.MAX_WATERMARK);
      }
   } finally {
      // make sure that the context is closed in any case
      ctx.close();
      if (latencyEmitter != null) {
         latencyEmitter.close();
      }
   }
}
```
可以看到StreamSource.run()的核心方法就是调用SourceFunction.run()方法，ctx中持有output引用。

到了run()方法之后就开始Task的正式执行了，那么它的启动过程也就完毕了。

## 总结：

Task启动的大致步骤为：

1、TaskExecutor在创建Task实例之后就会调用task.startTaskThread()开始启动Task，Task本身实现了Runable接口，每个Task都会启动一个线程Thread来运行自身。即调用Task的run()方法。

2、Task.run()方法中最核心的就是创建Task的AbstractInvokable实例，然后调用invokable.invoke()执行Task的可执行类，一般的这里的AbstractInvokable就是StreamTask，实现子类一般是OneInputStreamTask和SourceStreamTask。StreamTask在构造时期会根据过渡边和它的输出ResultPartition创建RecordWriter，RecordWriter用于创建RecordWriterOutput

3、invokable.invoke()方法首先会创建StateBackend，默认的情况下即用户没有设置StateBackend的情况下，程序生成的MemoryStateBackend，可选的还包括FsStateBackend，第三方jar包中RockDBStateBackend

4、invoke()方法接下来的操作是创建operatorChain

1）、首先创建headOperator，headOperator是从StreamConfig中经过反序列读取的（在构造JobGraph的时候，代码会将operator序列化写到StreamConfig中）

2）程序会从chain的头部依次向下的遍历ChainedOutputs，即可以chain的StreamEdge，递归的进行创建下游的operator，类似于后序遍历，即先创建出末尾的operator，再依次向前创建出operator，operator的创建是从StreamConfig中通过反序列化获取，末尾operator的output是RecordWriterOutput（除StreamSink，StreamSink的output是BroadcastingOutputCollector），其余chain上的operator的output是CopyingChainingOutput，包括headOperator。

3）output的作用就是输出经过operator处理之后的数据。chain中上一个operator的CopyingChainingOutput会持有下一个operator的引用，上一个operator处理完数据之后，CopyingChainingOutput会将数据推送到下一个operator处理，直至末尾的operator处理完数据，末尾operator的RecordWriterOutput会将数据发送给网络，推送到下游的Task处理

5、创建完operatorChain之后，invoke会调用init()进行StreamTask的初始化操作，OneInputStreamTask的初始化就是创建核心数据处理器StreamInputProcessor，StreamInputProcessor中持有一个barrierHandler，barrierHandler负责获取数据buffer，为StreamInputProcessor提供数据输入，其内部实现就是通过inputGate来获取的

6、接下来就是进行operator的的初始化操作了，会调用operator.initializeState()，创建operator的operatorStateBackend和keyedStateBackend以及keyedStateStore；调用operator.open()方法来打开operator，最常见的就是调用用户自定义RichFunction的open()方法进行初始化。

7、operator初始化之后就开始调用run()方法来正式开始执行Task了，对于OneInputStreamTask就是调用核心数据处理器inputProcessor.processInput()方法来进行数据的处理；对于SourceStreamTask就是调用SourceFunction.run()来生产数据

到此，Task的启动过程就完毕了，接下来就是Task的执行过程了