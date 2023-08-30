---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink源码Task任务部署
JobMaster最后会调用startJobExecution()方法开始调度执行Job。那么本文就来分析分析一下Job的部署过程。

## 起点
```
public CompletableFuture<Acknowledge> start(final JobMasterId newJobMasterId) throws Exception {
   // make sure we receive RPC and async calls
   start();

   return callAsyncWithoutFencing(() -> startJobExecution(newJobMasterId), RpcUtils.INF_TIMEOUT);
}
```

```
private Acknowledge startJobExecution(JobMasterId newJobMasterId) throws Exception {
    ...
    
   resetAndScheduleExecutionGraph();

   ...
}
private void resetAndScheduleExecutionGraph() throws Exception {
   validateRunsInMainThread();

   final CompletableFuture<Void> executionGraphAssignedFuture;

   if (executionGraph.getState() == JobStatus.CREATED) {
      executionGraphAssignedFuture = CompletableFuture.completedFuture(null);
      executionGraph.start(getMainThreadExecutor());
   } else {
     ...
   }

   executionGraphAssignedFuture.thenRun(this::scheduleExecutionGraph);
}
private void scheduleExecutionGraph() {
  ...
   try {
      executionGraph.scheduleForExecution();
   }
   catch (Throwable t) {
      executionGraph.failGlobal(t);
   }
}
public void scheduleForExecution() throws JobException {

   ...

   if (transitionState(JobStatus.CREATED, JobStatus.RUNNING)) {

      final CompletableFuture<Void> newSchedulingFuture;

      switch (scheduleMode) {

         case LAZY_FROM_SOURCES:
            newSchedulingFuture = scheduleLazy(slotProvider);
            break;

         case EAGER:
            newSchedulingFuture = scheduleEager(slotProvider, allocationTimeout);
            break;

         default:
            throw new JobException("Schedule mode is invalid.");
      }
    ...
}
private CompletableFuture<Void> scheduleEager(SlotProvider slotProvider, final Time timeout) {
  ...
   // allocate the slots (obtain all their futures
   //这块是为job分配slot，本文先忽略掉这部分的细节
   for (ExecutionJobVertex ejv : getVerticesTopologically()) {
      // these calls are not blocking, they only return futures
      Collection<CompletableFuture<Execution>> allocationFutures = ejv.allocateResourcesForAll(
         slotProvider,
         queued,
         LocationPreferenceConstraint.ALL,
         allPreviousAllocationIds,
         timeout);

      allAllocationFutures.addAll(allocationFutures);
   }

   // this future is complete once all slot futures are complete.
   // the future fails once one slot future fails.
   final ConjunctFuture<Collection<Execution>> allAllocationsFuture = FutureUtils.combineAll(allAllocationFutures);

   return allAllocationsFuture.thenAccept(
      (Collection<Execution> executionsToDeploy) -> {
         for (Execution execution : executionsToDeploy) {
            try {
               execution.deploy();
            } catch (Throwable t) {
               throw new CompletionException(
                  new FlinkException(
                     String.format("Could not deploy execution %s.", execution),
                     t));
            }
         }
      })
  ...
}
```
上述调用链startJobExecution()-->resetAndScheduleExecutionGraph()-->scheduleExecutionGraph()-->scheduleEager()，scheduleEager()方法获取每个Task的物理执行Execution，这个Execution是在创建ExecutionGraph时为每个ExecutionVertex创建的。并给Execution分配slot，slot分配也是很重要的一个部分，本文暂时先忽略这部分的实现细节。

接着开始调用execution.deploy()来部署每个Task
```
public void deploy() throws JobException {
   assertRunningInJobMasterMainThread();

   final LogicalSlot slot  = assignedResource;

   ...
   //创建TaskDeploymentDescriptor，很重要
      final TaskDeploymentDescriptor deployment = vertex.createDeploymentDescriptor(
         attemptId,
         slot,
         taskRestore,
         attemptNumber);

      // null taskRestore to let it be GC'ed
      taskRestore = null;

      final TaskManagerGateway taskManagerGateway = slot.getTaskManagerGateway();

      final ComponentMainThreadExecutor jobMasterMainThreadExecutor =
         vertex.getExecutionGraph().getJobMasterMainThreadExecutor();


      // We run the submission in the future executor so that the serialization of large TDDs does not block
      // the main thread and sync back to the main thread once submission is completed.
     //调用rpc方法进行task提交
       CompletableFuture.supplyAsync(() -> taskManagerGateway.submitTask(deployment, rpcTimeout), executor)
      ...
}
```
可以看到execution.deploy()实现分为两部分：

1、创建TaskDeploymentDescriptor，即Task部署的描述信息

2、通过rpc方法提交Task

## 创建TaskDeploymentDescriptor
首先来看看vertex.createDeploymentDescriptor()方法，创建Task部署的描述信息，由ExecutionVertex调用。
```
TaskDeploymentDescriptor createDeploymentDescriptor(
      ExecutionAttemptID executionId,
      LogicalSlot targetSlot,
      @Nullable JobManagerTaskRestore taskRestore,
      int attemptNumber) throws ExecutionGraphException {

   // Produced intermediate results
   List<ResultPartitionDeploymentDescriptor> producedPartitions = new ArrayList<>(resultPartitions.size());

   // Consumed intermediate results
   List<InputGateDeploymentDescriptor> consumedPartitions = new ArrayList<>(inputEdges.length);

   boolean lazyScheduling = getExecutionGraph().getScheduleMode().allowLazyDeployment();

    //创建ResultPartitionDeploymentDescriptor，是Task的输出描述信息。partition是Task输出的那个IntermediateResultPartition 
   for (IntermediateResultPartition partition : resultPartitions.values()) {

      List<List<ExecutionEdge>> consumers = partition.getConsumers();

      if (consumers.isEmpty()) {
         //TODO this case only exists for test, currently there has to be exactly one consumer in real jobs!
         producedPartitions.add(ResultPartitionDeploymentDescriptor.from(
               partition,
               KeyGroupRangeAssignment.UPPER_BOUND_MAX_PARALLELISM,
               lazyScheduling));
      } else {
         Preconditions.checkState(1 == consumers.size(),
               "Only one consumer supported in the current implementation! Found: " + consumers.size());

         List<ExecutionEdge> consumer = consumers.get(0);
         ExecutionJobVertex vertex = consumer.get(0).getTarget().getJobVertex();
         int maxParallelism = vertex.getMaxParallelism();
         producedPartitions.add(ResultPartitionDeploymentDescriptor.from(partition, maxParallelism, lazyScheduling));
      }
   }

    //创建InputGateDeploymentDescriptor，是关于Task输入的描述信息
   for (ExecutionEdge[] edges : inputEdges) {
      InputChannelDeploymentDescriptor[] partitions = InputChannelDeploymentDescriptor.fromEdges(
         edges,
         targetSlot.getTaskManagerLocation().getResourceID(),
         lazyScheduling);

      // If the produced partition has multiple consumers registered, we
      // need to request the one matching our sub task index.
      // TODO Refactor after removing the consumers from the intermediate result partitions
      int numConsumerEdges = edges[0].getSource().getConsumers().get(0).size();

      int queueToRequest = subTaskIndex % numConsumerEdges;

      IntermediateResult consumedIntermediateResult = edges[0].getSource().getIntermediateResult();
      final IntermediateDataSetID resultId = consumedIntermediateResult.getId();
      final ResultPartitionType partitionType = consumedIntermediateResult.getResultType();

      consumedPartitions.add(new InputGateDeploymentDescriptor(resultId, partitionType, queueToRequest, partitions));
   }

   final Either<SerializedValue<JobInformation>, PermanentBlobKey> jobInformationOrBlobKey = getExecutionGraph().getJobInformationOrBlobKey();

   final TaskDeploymentDescriptor.MaybeOffloaded<JobInformation> serializedJobInformation;

   if (jobInformationOrBlobKey.isLeft()) {
      serializedJobInformation = new TaskDeploymentDescriptor.NonOffloaded<>(jobInformationOrBlobKey.left());
   } else {
      serializedJobInformation = new TaskDeploymentDescriptor.Offloaded<>(jobInformationOrBlobKey.right());
   }

   final Either<SerializedValue<TaskInformation>, PermanentBlobKey> taskInformationOrBlobKey;

   try {
      taskInformationOrBlobKey = jobVertex.getTaskInformationOrBlobKey();
   } catch (IOException e) {
      throw new ExecutionGraphException(
         "Could not create a serialized JobVertexInformation for " +
            jobVertex.getJobVertexId(), e);
   }

   final TaskDeploymentDescriptor.MaybeOffloaded<TaskInformation> serializedTaskInformation;

   if (taskInformationOrBlobKey.isLeft()) {
      serializedTaskInformation = new TaskDeploymentDescriptor.NonOffloaded<>(taskInformationOrBlobKey.left());
   } else {
      serializedTaskInformation = new TaskDeploymentDescriptor.Offloaded<>(taskInformationOrBlobKey.right());
   }

    //创建一个TaskDeploymentDescriptor，并返回
   return new TaskDeploymentDescriptor(
      getJobId(),
      serializedJobInformation,
      serializedTaskInformation,
      executionId,
      targetSlot.getAllocationId(),
      subTaskIndex,
      attemptNumber,
      targetSlot.getPhysicalSlotNumber(),
      taskRestore,
      producedPartitions,
      consumedPartitions);
}
```

## 构建ResultPartitionDeploymentDescriptor
首先来看看ResultPartitionDeploymentDescriptor是什么

```
public static ResultPartitionDeploymentDescriptor from(
      IntermediateResultPartition partition, int maxParallelism, boolean lazyScheduling) {

   final IntermediateDataSetID resultId = partition.getIntermediateResult().getId();
   final IntermediateResultPartitionID partitionId = partition.getPartitionId();
   final ResultPartitionType partitionType = partition.getIntermediateResult().getResultType();

   // The produced data is partitioned among a number of subpartitions.
   //
   // If no consumers are known at this point, we use a single subpartition, otherwise we have
   // one for each consuming sub task.
   int numberOfSubpartitions = 1;

   if (!partition.getConsumers().isEmpty() && !partition.getConsumers().get(0).isEmpty()) {

      if (partition.getConsumers().size() > 1) {
         throw new IllegalStateException("Currently, only a single consumer group per partition is supported.");
      }
    //数量等于该partition的消费者数量
      numberOfSubpartitions = partition.getConsumers().get(0).size();
   }

   return new ResultPartitionDeploymentDescriptor(
         resultId, partitionId, partitionType, numberOfSubpartitions, maxParallelism, lazyScheduling);
}

//ResultPartitionDeploymentDescriptor构造方法
public ResultPartitionDeploymentDescriptor(
      IntermediateDataSetID resultId,
      IntermediateResultPartitionID partitionId,
      ResultPartitionType partitionType,
      int numberOfSubpartitions,
      int maxParallelism,
      boolean lazyScheduling) {

   this.resultId = checkNotNull(resultId);
   this.partitionId = checkNotNull(partitionId);
   this.partitionType = checkNotNull(partitionType);

   KeyGroupRangeAssignment.checkParallelismPreconditions(maxParallelism);
   checkArgument(numberOfSubpartitions >= 1);
   this.numberOfSubpartitions = numberOfSubpartitions;
   this.maxParallelism = maxParallelism;
   this.sendScheduleOrUpdateConsumersMessage = lazyScheduling;
}
```
可以看到，ResultPartitionDeploymentDescriptor是对Task输出信息的描述，描述的是该Task输出的是哪个IntermediateResult，哪个IntermediateResultPartition，该IntermediateResultPartition有多少个消费者等等。ResultPartitionDeploymentDescriptor的个数跟该Task要产生的结果集IntermediateResultPartition个数一致，一般就是一个。这部分的代码应该结合着ExecutionGraph去理解。

## 构建InputGateDeploymentDescriptor
再来看看InputGateDeploymentDescriptor，它是对Task输入的描述，同时还包含了InputChannelDeploymentDescriptor，是对Task的输入数据来自不同的InputChannel的描述信息，包含了Task消费的PartitionId和位置信息partitionLocation
```
public static InputChannelDeploymentDescriptor[] fromEdges(
      ExecutionEdge[] edges,
      ResourceID consumerResourceId,
      boolean allowLazyDeployment) throws ExecutionGraphException {

   final InputChannelDeploymentDescriptor[] icdd = new InputChannelDeploymentDescriptor[edges.length];

   // Each edge is connected to a different result partition
   for (int i = 0; i < edges.length; i++) {
      final IntermediateResultPartition consumedPartition = edges[i].getSource();
      final Execution producer = consumedPartition.getProducer().getCurrentExecutionAttempt();

      final ExecutionState producerState = producer.getState();
      final LogicalSlot producerSlot = producer.getAssignedResource();

      final ResultPartitionLocation partitionLocation;

      // The producing task needs to be RUNNING or already FINISHED
      if ((consumedPartition.getResultType().isPipelined() || consumedPartition.isConsumable()) &&
         producerSlot != null &&
            (producerState == ExecutionState.RUNNING ||
               producerState == ExecutionState.FINISHED ||
               producerState == ExecutionState.SCHEDULED ||
               producerState == ExecutionState.DEPLOYING)) {

         final TaskManagerLocation partitionTaskManagerLocation = producerSlot.getTaskManagerLocation();
         final ResourceID partitionTaskManager = partitionTaskManagerLocation.getResourceID();

        //根据partition的TaskManager的位置来决定partitionLocation的类型，如LOCAL、REMOTE、UNKNOWN
         if (partitionTaskManager.equals(consumerResourceId)) {
            // Consuming task is deployed to the same TaskManager as the partition => local
            partitionLocation = ResultPartitionLocation.createLocal();
         }
         else {
            // Different instances => remote
            final ConnectionID connectionId = new ConnectionID(
                  partitionTaskManagerLocation,
                  consumedPartition.getIntermediateResult().getConnectionIndex());

            partitionLocation = ResultPartitionLocation.createRemote(connectionId);
         }
      }
      else if (allowLazyDeployment) {
         // The producing task might not have registered the partition yet
         partitionLocation = ResultPartitionLocation.createUnknown();
      }
      else if (producerState == ExecutionState.CANCELING
               || producerState == ExecutionState.CANCELED
               || producerState == ExecutionState.FAILED) {
         String msg = "Trying to schedule a task whose inputs were canceled or failed. " +
            "The producer is in state " + producerState + ".";
         throw new ExecutionGraphException(msg);
      }
      else {
         String msg = String.format("Trying to eagerly schedule a task whose inputs " +
            "are not ready (result type: %s, partition consumable: %s, producer state: %s, producer slot: %s).",
               consumedPartition.getResultType(),
               consumedPartition.isConsumable(),
               producerState,
               producerSlot);
         throw new ExecutionGraphException(msg);
      }

      final ResultPartitionID consumedPartitionId = new ResultPartitionID(
            consumedPartition.getPartitionId(), producer.getAttemptId());

    //InputChannelDeploymentDescriptor包含了消费的PartitionId和位置信息partitionLocation
      icdd[i] = new InputChannelDeploymentDescriptor(
            consumedPartitionId, partitionLocation);
   }

   return icdd;
}

//InputGateDeploymentDescriptor构造方法，包含了输入的是哪个IntermediateResult，有多少个inputChannels
public InputGateDeploymentDescriptor(
      IntermediateDataSetID consumedResultId,
      ResultPartitionType consumedPartitionType,
      int consumedSubpartitionIndex,
      InputChannelDeploymentDescriptor[] inputChannels) {

   this.consumedResultId = checkNotNull(consumedResultId);
   this.consumedPartitionType = checkNotNull(consumedPartitionType);

   checkArgument(consumedSubpartitionIndex >= 0);
   this.consumedSubpartitionIndex = consumedSubpartitionIndex;

   this.inputChannels = checkNotNull(inputChannels);
}
```
可以看到InputGateDeploymentDescriptor是对Task输入数据的描述信息，包含了输入的是哪个IntermediateResult，有多少个inputChannels，自已消费该IntermediateResultPartition的哪个consumedSubpartitionIndex等等。InputGateDeploymentDescriptor的个数跟Task消费的IntermediateResult种类是一致的，即Task消费多少个IntermediateResult，就会创建多少个InputGateDeploymentDescriptor。这部分的代码应该结合着ExecutionGraph去理解

## 构建TaskDeploymentDescriptor
接下来就是TaskDeploymentDescriptor
```
public TaskDeploymentDescriptor(
   JobID jobId,
   MaybeOffloaded<JobInformation> serializedJobInformation,
   MaybeOffloaded<TaskInformation> serializedTaskInformation,
   ExecutionAttemptID executionAttemptId,
   AllocationID allocationId,
   int subtaskIndex,
   int attemptNumber,
   int targetSlotNumber,
   @Nullable JobManagerTaskRestore taskRestore,
   Collection<ResultPartitionDeploymentDescriptor> resultPartitionDeploymentDescriptors,
   Collection<InputGateDeploymentDescriptor> inputGateDeploymentDescriptors) {

   this.jobId = Preconditions.checkNotNull(jobId);

   this.serializedJobInformation = Preconditions.checkNotNull(serializedJobInformation);
   this.serializedTaskInformation = Preconditions.checkNotNull(serializedTaskInformation);

   this.executionId = Preconditions.checkNotNull(executionAttemptId);
   this.allocationId = Preconditions.checkNotNull(allocationId);

   Preconditions.checkArgument(0 <= subtaskIndex, "The subtask index must be positive.");
   this.subtaskIndex = subtaskIndex;

   Preconditions.checkArgument(0 <= attemptNumber, "The attempt number must be positive.");
   this.attemptNumber = attemptNumber;

   Preconditions.checkArgument(0 <= targetSlotNumber, "The target slot number must be positive.");
   this.targetSlotNumber = targetSlotNumber;

   this.taskRestore = taskRestore;

   this.producedPartitions = Preconditions.checkNotNull(resultPartitionDeploymentDescriptors);
   this.inputGates = Preconditions.checkNotNull(inputGateDeploymentDescriptors);
}
```
TaskDeploymentDescriptor包含了上述所说的ResultPartitionDeploymentDescriptor和InputGateDeploymentDescriptor，同时还包含了序列化之后的serializedJobInformation和serializedTaskInformation等等。

## 提交Task
构造完TaskDeploymentDescriptor完之后，之后就是通过rpc方法进行submitTask了。最终会调用TaskExecutor.submitTask()方法
```
public CompletableFuture<Acknowledge> submitTask(TaskDeploymentDescriptor tdd, Time timeout) {
   return taskExecutorGateway.submitTask(tdd, jobMasterId, timeout);
}
public CompletableFuture<Acknowledge> submitTask(
      TaskDeploymentDescriptor tdd,
      JobMasterId jobMasterId,
      Time timeout) {
    ...
    //构造Task实例
      Task task = new Task(
         jobInformation,
         taskInformation,
         tdd.getExecutionAttemptId(),
         tdd.getAllocationId(),
         tdd.getSubtaskIndex(),
         tdd.getAttemptNumber(),
         tdd.getProducedPartitions(),
         tdd.getInputGates(),
         tdd.getTargetSlotNumber(),
         taskExecutorServices.getMemoryManager(),
         taskExecutorServices.getIOManager(),
         taskExecutorServices.getNetworkEnvironment(),
         taskExecutorServices.getBroadcastVariableManager(),
         taskStateManager,
         taskManagerActions,
         inputSplitProvider,
         checkpointResponder,
         aggregateManager,
         blobCacheService,
         libraryCache,
         fileCache,
         taskManagerConfiguration,
         taskMetricGroup,
         resultPartitionConsumableNotifier,
         partitionStateChecker,
         getRpcService().getExecutor());

     ...

      if (taskAdded) {
         task.startTaskThread();

         return CompletableFuture.completedFuture(Acknowledge.get());
      }
      ...
}
```
submitTask()方法最核心的实现就是构建Task实例，然后启动Task。

## 构建Task实例
来看Task的构造方法
```
public Task(
   JobInformation jobInformation,
   TaskInformation taskInformation,
   ExecutionAttemptID executionAttemptID,
   AllocationID slotAllocationId,
   int subtaskIndex,
   int attemptNumber,
   Collection<ResultPartitionDeploymentDescriptor> resultPartitionDeploymentDescriptors,
   Collection<InputGateDeploymentDescriptor> inputGateDeploymentDescriptors,
   int targetSlotNumber,
   MemoryManager memManager,
   IOManager ioManager,
   NetworkEnvironment networkEnvironment,
   BroadcastVariableManager bcVarManager,
   TaskStateManager taskStateManager,
   TaskManagerActions taskManagerActions,
   InputSplitProvider inputSplitProvider,
   CheckpointResponder checkpointResponder,
   GlobalAggregateManager aggregateManager,
   BlobCacheService blobService,
   LibraryCacheManager libraryCache,
   FileCache fileCache,
   TaskManagerRuntimeInfo taskManagerConfig,
   @Nonnull TaskMetricGroup metricGroup,
   ResultPartitionConsumableNotifier resultPartitionConsumableNotifier,
   PartitionProducerStateChecker partitionProducerStateChecker,
   Executor executor) {

   Preconditions.checkNotNull(jobInformation);
   Preconditions.checkNotNull(taskInformation);

   Preconditions.checkArgument(0 <= subtaskIndex, "The subtask index must be positive.");
   Preconditions.checkArgument(0 <= attemptNumber, "The attempt number must be positive.");
   Preconditions.checkArgument(0 <= targetSlotNumber, "The target slot number must be positive.");

   this.taskInfo = new TaskInfo(
         taskInformation.getTaskName(),
         taskInformation.getMaxNumberOfSubtaks(),
         subtaskIndex,
         taskInformation.getNumberOfSubtasks(),
         attemptNumber,
         String.valueOf(slotAllocationId));

   this.jobId = jobInformation.getJobId();
   this.vertexId = taskInformation.getJobVertexId();
   this.executionId  = Preconditions.checkNotNull(executionAttemptID);
   this.allocationId = Preconditions.checkNotNull(slotAllocationId);
   this.taskNameWithSubtask = taskInfo.getTaskNameWithSubtasks();
   this.jobConfiguration = jobInformation.getJobConfiguration();
   this.taskConfiguration = taskInformation.getTaskConfiguration();
   this.requiredJarFiles = jobInformation.getRequiredJarFileBlobKeys();
   this.requiredClasspaths = jobInformation.getRequiredClasspathURLs();
   //Task的实际调用运行类，例如StreamTask
   this.nameOfInvokableClass = taskInformation.getInvokableClassName();
   this.serializedExecutionConfig = jobInformation.getSerializedExecutionConfig();

   Configuration tmConfig = taskManagerConfig.getConfiguration();
   this.taskCancellationInterval = tmConfig.getLong(TaskManagerOptions.TASK_CANCELLATION_INTERVAL);
   this.taskCancellationTimeout = tmConfig.getLong(TaskManagerOptions.TASK_CANCELLATION_TIMEOUT);
    //设置memoryManager ioManager taskStateManager 
   this.memoryManager = Preconditions.checkNotNull(memManager);
   this.ioManager = Preconditions.checkNotNull(ioManager);
   this.broadcastVariableManager = Preconditions.checkNotNull(bcVarManager);
   this.taskStateManager = Preconditions.checkNotNull(taskStateManager);
   this.accumulatorRegistry = new AccumulatorRegistry(jobId, executionId);

   this.inputSplitProvider = Preconditions.checkNotNull(inputSplitProvider);
   this.checkpointResponder = Preconditions.checkNotNull(checkpointResponder);
   this.aggregateManager = Preconditions.checkNotNull(aggregateManager);
   this.taskManagerActions = checkNotNull(taskManagerActions);

   this.blobService = Preconditions.checkNotNull(blobService);
   this.libraryCache = Preconditions.checkNotNull(libraryCache);
   this.fileCache = Preconditions.checkNotNull(fileCache);
   this.network = Preconditions.checkNotNull(networkEnvironment);
   this.taskManagerConfig = Preconditions.checkNotNull(taskManagerConfig);

   this.metrics = metricGroup;

   this.partitionProducerStateChecker = Preconditions.checkNotNull(partitionProducerStateChecker);
   this.executor = Preconditions.checkNotNull(executor);

   // create the reader and writer structures

   final String taskNameWithSubtaskAndId = taskNameWithSubtask + " (" + executionId + ')';

   // Produced intermediate result partitions
   //创建ResultPartition，即Task的输出操作
   this.producedPartitions = new ResultPartition[resultPartitionDeploymentDescriptors.size()];

   int counter = 0;

   for (ResultPartitionDeploymentDescriptor desc: resultPartitionDeploymentDescriptors) {
      ResultPartitionID partitionId = new ResultPartitionID(desc.getPartitionId(), executionId);

      this.producedPartitions[counter] = new ResultPartition(
         taskNameWithSubtaskAndId,
         this,
         jobId,
         partitionId,
         desc.getPartitionType(),
         desc.getNumberOfSubpartitions(),
         desc.getMaxParallelism(),
         networkEnvironment.getResultPartitionManager(),
         resultPartitionConsumableNotifier,
         ioManager,
         desc.sendScheduleOrUpdateConsumersMessage());

      ++counter;
   }

   // Consumed intermediate result partitions
   //创建SingleInputGate，即Task的输入
   this.inputGates = new SingleInputGate[inputGateDeploymentDescriptors.size()];
   this.inputGatesById = new HashMap<>();

   counter = 0;

   for (InputGateDeploymentDescriptor inputGateDeploymentDescriptor: inputGateDeploymentDescriptors) {
      SingleInputGate gate = SingleInputGate.create(
         taskNameWithSubtaskAndId,
         jobId,
         executionId,
         inputGateDeploymentDescriptor,
         networkEnvironment,
         this,
         metricGroup.getIOMetricGroup());

      inputGates[counter] = gate;
      inputGatesById.put(gate.getConsumedResultId(), gate);

      ++counter;
   }

   invokableHasBeenCanceled = new AtomicBoolean(false);

   // finally, create the executing thread, but do not start it
   //创建Task的执行线程，也就是自己，Task实现了runnable接口
   executingThread = new Thread(TASK_THREADS_GROUP, this, taskNameWithSubtask);
}
```
可以看到Task的初始化里设置了程序调用类nameOfInvokableClass、设置memoryManager ioManager taskStateManager，根据resultPartitionDeploymentDescriptors创建producedPartitions，即Task的输出操作，它是ResultPartition的数组，ResultPartition是ResultPartitionWriter的实现类。根据inputGateDeploymentDescriptor创建inputGates ，即Task的输入，创建执行线程等等操作。可以看到Flink的每个Task都由一个线程来执行。接下来就到了运行Task.run()方法了。

## 总结

Task部署过程实现步骤大致如下

1、JobMaster在启动后开始调用startJobExecution()，开始调度Job

2、经过调用链startJobExecution()-->resetAndScheduleExecutionGraph()-->scheduleExecutionGraph()-->scheduleEager()，scheduleEager()方法开始为每个Task创建Execution，并分配slot。接着开始调用execution.deploy()来部署每个Task

3、execution.deploy()首先会创建TaskDeploymentDescriptor，即Task部署的描述信息，创建TaskDeploymentDescriptor的过程中又会创建ResultPartitionDeploymentDescriptor和InputGateDeploymentDescriptor，这部分代码要结合ExecutionGraph理解

1）ResultPartitionDeploymentDescriptor是对Task输出信息的描述，描述的是该Task输出的是哪个IntermediateResult的哪个IntermediateResultPartition，该IntermediateResultPartition有多少个消费者等等。ResultPartitionDeploymentDescriptor的个数跟该Task要产生的结果集IntermediateResultPartition个数一致

2）InputGateDeploymentDescriptor是对Task输入数据的描述信息，包含了输入的是哪个IntermediateResult，有多少个inputChannels，自已消费该IntermediateResultPartition的哪个consumedSubpartitionIndex等等。InputGateDeploymentDescriptor的个数跟Task消费的IntermediateResult种类是一致的，即Task消费多少个IntermediateResult，就会创建多少个InputGateDeploymentDescriptor

3）TaskDeploymentDescriptor包含了上述所说的ResultPartitionDeploymentDescriptor和InputGateDeploymentDescriptor，同时还包含了序列化之后的serializedJobInformation和serializedTaskInformation等等

4、构建完TaskDeploymentDescriptor之后，就开始调用rpc方法进行submit Task了，最终会调用TaskExecutor.submitTask()方法。

5、submitTask()方法中会构建Task实例，Task构造方法里设置了程序调用类nameOfInvokableClass、设置memoryManager、ioManager、taskStateManager，根据resultPartitionDeploymentDescriptors创建producedPartitions，即Task的输出，它是ResultPartition的数组，ResultPartition是ResultPartitionWriter的实现类。根据inputGateDeploymentDescriptor创建inputGates，是SingleInputGate数组，即Task的输入，创建执行线程等等操作

6、TaskExecutor在创建Task实例之后就会调用task.startTaskThread()开始启动Task，Task本身实现了Runable接口，每个Task都会启动一个线程Thread来运行自身。即调用Task的run()方法

到此，Task部署过程完毕，接下来就是Task的启动过程，这个过程会初始化Task和operator的一些东西















