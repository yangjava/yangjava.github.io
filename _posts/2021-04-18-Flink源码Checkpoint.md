---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink源码Checkpoint
checkpoint是flink中容错机制的保证，了解checkpoint机制对我们深入了解flink也有很大的帮助。现在来分析一下checkpoint的源码实现。

## JobManager端checkpoint调度
在JobManager端构建ExecutionGraph过程中（ExecutionGraphBuilder.buildGraph()方法），会调用ExecutionGraph.enableCheckpointing()方法，这个方法不管任务里有没有设置checkpoint都会调用的。在enableCheckpointing()方法里会创建CheckpointCoordinator，这是负责checkpoint的核心实现类，同时会给job添加一个监听器CheckpointCoordinatorDeActivator（只有设置了checkpoint才会注册这个监听器），CheckpointCoordinatorDeActivator负责checkpoint的启动和停止。源码如下：
```
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
      int parallelismForAutoMax,
      BlobWriter blobWriter,
      Time allocationTimeout,
      Logger log)
   throws JobExecutionException, JobException {
      ...
   // configure the state checkpointing
   JobCheckpointingSettings snapshotSettings = jobGraph.getCheckpointingSettings();
   if (snapshotSettings != null) {
        ...

      final CheckpointCoordinatorConfiguration chkConfig = snapshotSettings.getCheckpointCoordinatorConfiguration();

      executionGraph.enableCheckpointing(
         chkConfig.getCheckpointInterval(), //checkpoint间隔
         chkConfig.getCheckpointTimeout(), //checkpoint超时时间
         chkConfig.getMinPauseBetweenCheckpoints(), //两次checkpoint间隔最短时间
         chkConfig.getMaxConcurrentCheckpoints(), //同时可并发执行的checkpoint数量
         chkConfig.getCheckpointRetentionPolicy(), //checkpoint的保留策略
         triggerVertices, //需要触发checkpoint的任务，即所有的sourceTask
         ackVertices, //需要确认checkpoint的任务，这里是所有的任务
         confirmVertices, //checkpoint执行完需要进行commit的任务，也是所有的任务
         hooks,
         checkpointIdCounter, //checkpoint ID计数器,每执行一次就+1
         completedCheckpoints, //CompletedCheckpointStore, 存放已完成的checkpoint
         rootBackend,
         checkpointStatsTracker);
   }

    ...

   return executionGraph;
}
```

```
//ExecutionGraph类
public void enableCheckpointing(
      long interval,
      long checkpointTimeout,
      long minPauseBetweenCheckpoints,
      int maxConcurrentCheckpoints,
      CheckpointRetentionPolicy retentionPolicy,
      List<ExecutionJobVertex> verticesToTrigger,
      List<ExecutionJobVertex> verticesToWaitFor,
      List<ExecutionJobVertex> verticesToCommitTo,
      List<MasterTriggerRestoreHook<?>> masterHooks,
      CheckpointIDCounter checkpointIDCounter,
      CompletedCheckpointStore checkpointStore,
      StateBackend checkpointStateBackend,
      CheckpointStatsTracker statsTracker) {

   // simple sanity checks
   checkArgument(interval >= 10, "checkpoint interval must not be below 10ms");
   checkArgument(checkpointTimeout >= 10, "checkpoint timeout must not be below 10ms");

   checkState(state == JobStatus.CREATED, "Job must be in CREATED state");
   checkState(checkpointCoordinator == null, "checkpointing already enabled");

   ExecutionVertex[] tasksToTrigger = collectExecutionVertices(verticesToTrigger);
   ExecutionVertex[] tasksToWaitFor = collectExecutionVertices(verticesToWaitFor);
   ExecutionVertex[] tasksToCommitTo = collectExecutionVertices(verticesToCommitTo);

   checkpointStatsTracker = checkNotNull(statsTracker, "CheckpointStatsTracker");
   
   //创建checkpointCoordinator，这里的checkpointCoordinator不管有没有设置checkpoint都会创建
   // create the coordinator that triggers and commits checkpoints and holds the state
   checkpointCoordinator = new CheckpointCoordinator(
      jobInformation.getJobId(),
      interval,
      checkpointTimeout,
      minPauseBetweenCheckpoints,
      maxConcurrentCheckpoints,
      retentionPolicy,
      tasksToTrigger,
      tasksToWaitFor,
      tasksToCommitTo,
      checkpointIDCounter,
      checkpointStore,
      checkpointStateBackend,
      ioExecutor,
      SharedStateRegistry.DEFAULT_FACTORY);

   // register the master hooks on the checkpoint coordinator
   for (MasterTriggerRestoreHook<?> hook : masterHooks) {
      if (!checkpointCoordinator.addMasterHook(hook)) {
         LOG.warn("Trying to register multiple checkpoint hooks with the name: {}", hook.getIdentifier());
      }
   }

   checkpointCoordinator.setCheckpointStatsTracker(checkpointStatsTracker);

   //注册一个job状态监听器，在job任务出现改变时会进行一些相应的操作
   //注意如果没有设置checkpoint的话，则不会注册这个checkpoint监听器
   // interval of max long value indicates disable periodic checkpoint,
   // the CheckpointActivatorDeactivator should be created only if the interval is not max value
   if (interval != Long.MAX_VALUE) {
      // the periodic checkpoint scheduler is activated and deactivated as a result of
      // job status changes (running -> on, all other states -> off)
      registerJobStatusListener(checkpointCoordinator.createActivatorDeactivator());
   }
}

//CheckpointCoordinator类
public JobStatusListener createActivatorDeactivator() {
   synchronized (lock) {
      if (shutdown) {
         throw new IllegalArgumentException("Checkpoint coordinator is shut down");
      }

      if (jobStatusListener == null) {
         jobStatusListener = new CheckpointCoordinatorDeActivator(this);
      }

      return jobStatusListener;
   }
}
```
在JobManager端开始进行任务调度的时候，会对job的状态进行转换，由CREATED转成RUNNING,实现在transitionState()方法中，在这个过程中刚才设置的job监听器CheckpointCoordinatorDeActivator就开始启动checkpoint的定时任务了，调用链为ExecutionGraph.scheduleForExecution() -> transitionState() -> notifyJobStatusChange() -> CheckpointCoordinatorDeActivator.jobStatusChanges() -> CheckpointCoordinator.startCheckpointScheduler()源码如下：
```
//ExecutionGraph类
public void scheduleForExecution() throws JobException {

   assertRunningInJobMasterMainThread();

   final long currentGlobalModVersion = globalModVersion;

   if (transitionState(JobStatus.CREATED, JobStatus.RUNNING)) {

      ...
   }
   else {
      throw new IllegalStateException("Job may only be scheduled from state " + JobStatus.CREATED);
   }
}

private boolean transitionState(JobStatus current, JobStatus newState, Throwable error) {
   assertRunningInJobMasterMainThread();
    ...
   // now do the actual state transition
   if (STATE_UPDATER.compareAndSet(this, current, newState)) {
      LOG.info("Job {} ({}) switched from state {} to {}.", getJobName(), getJobID(), current, newState, error);

      stateTimestamps[newState.ordinal()] = System.currentTimeMillis();
      notifyJobStatusChange(newState, error);
      return true;
   }
   else {
      return false;
   }
}

private void notifyJobStatusChange(JobStatus newState, Throwable error) {
   if (jobStatusListeners.size() > 0) {
      final long timestamp = System.currentTimeMillis();
      final Throwable serializedError = error == null ? null : new SerializedThrowable(error);

      for (JobStatusListener listener : jobStatusListeners) {
         try {
            listener.jobStatusChanges(getJobID(), newState, timestamp, serializedError);
         } catch (Throwable t) {
            LOG.warn("Error while notifying JobStatusListener", t);
         }
      }
   }
}

//CheckpointCoordinatorDeActivator类
public void jobStatusChanges(JobID jobId, JobStatus newJobStatus, long timestamp, Throwable error) {
   if (newJobStatus == JobStatus.RUNNING) {
      // start the checkpoint scheduler
      coordinator.startCheckpointScheduler();
   } else {
      // anything else should stop the trigger for now
      coordinator.stopCheckpointScheduler();
   }
}
```
CheckpointCoordinator会部署一个定时任务，用于周期性的触发checkpoint，这个定时任务就是ScheduledTrigger类
```
public void startCheckpointScheduler() {
   synchronized (lock) {
      if (shutdown) {
         throw new IllegalArgumentException("Checkpoint coordinator is shut down");
      }

      // make sure all prior timers are cancelled
      stopCheckpointScheduler();

      periodicScheduling = true;
      long initialDelay = ThreadLocalRandom.current().nextLong(
         minPauseBetweenCheckpointsNanos / 1_000_000L, baseInterval + 1L);
      currentPeriodicTrigger = timer.scheduleAtFixedRate(
            new ScheduledTrigger(), initialDelay, baseInterval, TimeUnit.MILLISECONDS);
   }
}
```
## ScheduledTrigger定时触发checkpoint

下面我们来看ScheduledTrigger的实现，主要就在CheckpointCoordinator.triggerCheckpoint()中：

1、 在触发checkpoint之前先做一遍检查，检查当前正在处理的checkpoint是否超过设置的最大并发checkpoint数量，检查checkpoint的间隔是否达到设置的两次checkpoint的时间间隔，在都没有问题的情况下才可以触发checkpoint

2、检查需要触发的task是否都正常运行，即所有的source task

3、检查JobManager端需要确认checkpoint信息的task时候正常运行，这里就是所有运行task，即所有的task都需要向JobManager发送确认自己checkpoint的消息

4、正式开始触发checkpoint，创建一个PendingCheckpoint，包含了checkpointID和timestamp，向所有的source task去触发checkpoint。

```
//CheckpointCoordinator类
private final class ScheduledTrigger implements Runnable {

   @Override
   public void run() {
      try {
         triggerCheckpoint(System.currentTimeMillis(), true);
      }
      catch (Exception e) {
         LOG.error("Exception while triggering checkpoint for job {}.", job, e);
      }
   }
}

public boolean triggerCheckpoint(long timestamp, boolean isPeriodic) {
   return triggerCheckpoint(timestamp, checkpointProperties, null, isPeriodic).isSuccess();
}

public CheckpointTriggerResult triggerCheckpoint(
      long timestamp,
      CheckpointProperties props,
      @Nullable String externalSavepointLocation,
      boolean isPeriodic) {
    
    //首先做一些检查，看能不能触发checkpoint，主要就是检查最大并发checkpoint数，checkpoint间隔时间
   // make some eager pre-checks
   synchronized (lock) {
      // abort if the coordinator has been shutdown in the meantime
      if (shutdown) {
         return new CheckpointTriggerResult(CheckpointDeclineReason.COORDINATOR_SHUTDOWN);
      }

      // Don't allow periodic checkpoint if scheduling has been disabled
      if (isPeriodic && !periodicScheduling) {
         return new CheckpointTriggerResult(CheckpointDeclineReason.PERIODIC_SCHEDULER_SHUTDOWN);
      }

      // validate whether the checkpoint can be triggered, with respect to the limit of
      // concurrent checkpoints, and the minimum time between checkpoints.
      // these checks are not relevant for savepoints
      if (!props.forceCheckpoint()) {
         // sanity check: there should never be more than one trigger request queued
         if (triggerRequestQueued) {
            LOG.warn("Trying to trigger another checkpoint for job {} while one was queued already.", job);
            return new CheckpointTriggerResult(CheckpointDeclineReason.ALREADY_QUEUED);
         }

         // if too many checkpoints are currently in progress, we need to mark that a request is queued
         if (pendingCheckpoints.size() >= maxConcurrentCheckpointAttempts) {
            triggerRequestQueued = true;
            if (currentPeriodicTrigger != null) {
               currentPeriodicTrigger.cancel(false);
               currentPeriodicTrigger = null;
            }
            return new CheckpointTriggerResult(CheckpointDeclineReason.TOO_MANY_CONCURRENT_CHECKPOINTS);
         }

         // make sure the minimum interval between checkpoints has passed
         final long earliestNext = lastCheckpointCompletionNanos + minPauseBetweenCheckpointsNanos;
         final long durationTillNextMillis = (earliestNext - System.nanoTime()) / 1_000_000;

         if (durationTillNextMillis > 0) {
            if (currentPeriodicTrigger != null) {
               currentPeriodicTrigger.cancel(false);
               currentPeriodicTrigger = null;
            }
            // Reassign the new trigger to the currentPeriodicTrigger
            currentPeriodicTrigger = timer.scheduleAtFixedRate(
                  new ScheduledTrigger(),
                  durationTillNextMillis, baseInterval, TimeUnit.MILLISECONDS);

            return new CheckpointTriggerResult(CheckpointDeclineReason.MINIMUM_TIME_BETWEEN_CHECKPOINTS);
         }
      }
   }

   //检查需要触发的task是否都正常运行，即所有的source task
   // check if all tasks that we need to trigger are running.
   // if not, abort the checkpoint
   Execution[] executions = new Execution[tasksToTrigger.length];
   for (int i = 0; i < tasksToTrigger.length; i++) {
      Execution ee = tasksToTrigger[i].getCurrentExecutionAttempt();
      if (ee == null) {
         LOG.info("Checkpoint triggering task {} of job {} is not being executed at the moment. Aborting checkpoint.",
               tasksToTrigger[i].getTaskNameWithSubtaskIndex(),
               job);
         return new CheckpointTriggerResult(CheckpointDeclineReason.NOT_ALL_REQUIRED_TASKS_RUNNING);
      } else if (ee.getState() == ExecutionState.RUNNING) {
         executions[i] = ee;
      } else {
         LOG.info("Checkpoint triggering task {} of job {} is not in state {} but {} instead. Aborting checkpoint.",
               tasksToTrigger[i].getTaskNameWithSubtaskIndex(),
               job,
               ExecutionState.RUNNING,
               ee.getState());
         return new CheckpointTriggerResult(CheckpointDeclineReason.NOT_ALL_REQUIRED_TASKS_RUNNING);
      }
   }

   //检查JobManager端需要确认checkpoint信息的task时候正常运行
   // next, check if all tasks that need to acknowledge the checkpoint are running.
   // if not, abort the checkpoint
   Map<ExecutionAttemptID, ExecutionVertex> ackTasks = new HashMap<>(tasksToWaitFor.length);

   for (ExecutionVertex ev : tasksToWaitFor) {
      Execution ee = ev.getCurrentExecutionAttempt();
      if (ee != null) {
         ackTasks.put(ee.getAttemptId(), ev);
      } else {
         LOG.info("Checkpoint acknowledging task {} of job {} is not being executed at the moment. Aborting checkpoint.",
               ev.getTaskNameWithSubtaskIndex(),
               job);
         return new CheckpointTriggerResult(CheckpointDeclineReason.NOT_ALL_REQUIRED_TASKS_RUNNING);
      }
   }

   // we will actually trigger this checkpoint!

   // we lock with a special lock to make sure that trigger requests do not overtake each other.
   // this is not done with the coordinator-wide lock, because the 'checkpointIdCounter'
   // may issue blocking operations. Using a different lock than the coordinator-wide lock,
   // we avoid blocking the processing of 'acknowledge/decline' messages during that time.
   synchronized (triggerLock) {

      final CheckpointStorageLocation checkpointStorageLocation;
      final long checkpointID;

      try {
         // this must happen outside the coordinator-wide lock, because it communicates
         // with external services (in HA mode) and may block for a while.
         checkpointID = checkpointIdCounter.getAndIncrement();

         checkpointStorageLocation = props.isSavepoint() ?
               checkpointStorage.initializeLocationForSavepoint(checkpointID, externalSavepointLocation) :
               checkpointStorage.initializeLocationForCheckpoint(checkpointID);
      }
      catch (Throwable t) {
         ...
      }

      final PendingCheckpoint checkpoint = new PendingCheckpoint(
         job,
         checkpointID,
         timestamp,
         ackTasks,
         props,
         checkpointStorageLocation,
         executor);

          ...

      try {
         // re-acquire the coordinator-wide lock
         synchronized (lock) {
            // since we released the lock in the meantime, we need to re-check
            // that the conditions still hold.
            //这里又重新做了一遍checkpoint检查
            ...

            LOG.info("Triggering checkpoint {} @ {} for job {}.", checkpointID, timestamp, job);

            pendingCheckpoints.put(checkpointID, checkpoint);

            ...
         }
         // end of lock scope

         final CheckpointOptions checkpointOptions = new CheckpointOptions(
               props.getCheckpointType(),
               checkpointStorageLocation.getLocationReference());
               
         //给source task发消息触发checkpoint      
         // send the messages to the tasks that trigger their checkpoint
         for (Execution execution: executions) {
            execution.triggerCheckpoint(checkpointID, timestamp, checkpointOptions);
         }

         numUnsuccessfulCheckpointsTriggers.set(0);
         return new CheckpointTriggerResult(checkpoint);
      }
      catch (Throwable t) {
          ...
      }

   } // end trigger lock
}
```
Execution.triggerCheckpoint()就是远程调用TaskManager的triggerCheckpoint()方法

```
//Execution 
public void triggerCheckpoint(long checkpointId, long timestamp, CheckpointOptions checkpointOptions) {
   final LogicalSlot slot = assignedResource;

   if (slot != null) {
      final TaskManagerGateway taskManagerGateway = slot.getTaskManagerGateway();

      taskManagerGateway.triggerCheckpoint(attemptId, getVertex().getJobId(), checkpointId, timestamp, checkpointOptions);
   } else {
      LOG.debug("The execution has no slot assigned. This indicates that the execution is " +
         "no longer running.");
   }
}

//RpcTaskManagerGateway类
public void triggerCheckpoint(ExecutionAttemptID executionAttemptID, JobID jobId, long checkpointId, long timestamp, CheckpointOptions checkpointOptions) {
   taskExecutorGateway.triggerCheckpoint(
      executionAttemptID,
      checkpointId,
      timestamp,
      checkpointOptions);
}
```

## SourceStreamTask的Checkpoint执行
TaskManager的triggerCheckpoint()方法首先获取到source task（即SourceStreamTask），调用Task.triggerCheckpointBarrier()，triggerCheckpointBarrier()会异步的去执行一个独立线程，这个线程来负责source task的checkpoint执行。

```
//TaskExecutor类
public CompletableFuture<Acknowledge> triggerCheckpoint(
      ExecutionAttemptID executionAttemptID,
      long checkpointId,
      long checkpointTimestamp,
      CheckpointOptions checkpointOptions) {
   log.debug("Trigger checkpoint {}@{} for {}.", checkpointId, checkpointTimestamp, executionAttemptID);

   final Task task = taskSlotTable.getTask(executionAttemptID);

   if (task != null) {
      task.triggerCheckpointBarrier(checkpointId, checkpointTimestamp, checkpointOptions);

      return CompletableFuture.completedFuture(Acknowledge.get());
   } else {
      final String message = "TaskManager received a checkpoint request for unknown task " + executionAttemptID + '.';

      log.debug(message);
      return FutureUtils.completedExceptionally(new CheckpointException(message));
   }
}

//Task类
public void triggerCheckpointBarrier(
      final long checkpointID,
      long checkpointTimestamp,
      final CheckpointOptions checkpointOptions) {

   final AbstractInvokable invokable = this.invokable;
   final CheckpointMetaData checkpointMetaData = new CheckpointMetaData(checkpointID, checkpointTimestamp);

   if (executionState == ExecutionState.RUNNING && invokable != null) {

      // build a local closure
      final String taskName = taskNameWithSubtask;
      final SafetyNetCloseableRegistry safetyNetCloseableRegistry =
         FileSystemSafetyNet.getSafetyNetCloseableRegistryForThread();

      Runnable runnable = new Runnable() {
         @Override
         public void run() {
            // set safety net from the task's context for checkpointing thread
            LOG.debug("Creating FileSystem stream leak safety net for {}", Thread.currentThread().getName());
            FileSystemSafetyNet.setSafetyNetCloseableRegistryForThread(safetyNetCloseableRegistry);

            try {
                //由一个单独的线程来调用执行，这个invokable在这里就是SourceStreamTask
               boolean success = invokable.triggerCheckpoint(checkpointMetaData, checkpointOptions);
               if (!success) {
                  checkpointResponder.declineCheckpoint(
                        getJobID(), getExecutionId(), checkpointID,
                        new CheckpointDeclineTaskNotReadyException(taskName));
               }
            }
            catch (Throwable t) {
               if (getExecutionState() == ExecutionState.RUNNING) {
                  failExternally(new Exception(
                     "Error while triggering checkpoint " + checkpointID + " for " +
                        taskNameWithSubtask, t));
               } else {
                  LOG.debug("Encountered error while triggering checkpoint {} for " +
                     "{} ({}) while being not in state running.", checkpointID,
                     taskNameWithSubtask, executionId, t);
               }
            } finally {
               FileSystemSafetyNet.setSafetyNetCloseableRegistryForThread(null);
            }
         }
      };
      //异步的执行这个线程
      executeAsyncCallRunnable(runnable, String.format("Checkpoint Trigger for %s (%s).", taskNameWithSubtask, executionId));
   }
   else {
     ...
   }
}
```
checkpoint的核心实现在StreamTask.performCheckpoint()方法中，该方法主要有三个步骤

1、在checkpoint之前做一些准备工作，通常情况下operator在这个阶段是不做什么操作的

2、立即向下游广播CheckpointBarrier，以便使下游的task能够及时的接收到CheckpointBarrier也开始进行checkpoint的操作

3、开始进行状态的快照，即checkpoint操作。

注意以上操作都是在同步代码块里进行的，获取到的这个lock锁就是用于checkpoint的锁，checkpoint线程和task任务线程用的是同一把锁，在进行performCheckpoint()时，task任务线程是不能够进行数据处理的

```
//SourceStreamTask类
public boolean triggerCheckpoint(CheckpointMetaData checkpointMetaData, CheckpointOptions checkpointOptions) throws Exception {
   if (!externallyInducedCheckpoints) {
      return super.triggerCheckpoint(checkpointMetaData, checkpointOptions);
   }
   else {
      // we do not trigger checkpoints here, we simply state whether we can trigger them
      synchronized (getCheckpointLock()) {
         return isRunning();
      }
   }
}

//StreamTask类
public boolean triggerCheckpoint(CheckpointMetaData checkpointMetaData, CheckpointOptions checkpointOptions) throws Exception {
   try {
      // No alignment if we inject a checkpoint
      CheckpointMetrics checkpointMetrics = new CheckpointMetrics()
            .setBytesBufferedInAlignment(0L)
            .setAlignmentDurationNanos(0L);

      return performCheckpoint(checkpointMetaData, checkpointOptions, checkpointMetrics);
   }
   catch (Exception e) {
      ...
   }
}

private boolean performCheckpoint(
      CheckpointMetaData checkpointMetaData,
      CheckpointOptions checkpointOptions,
      CheckpointMetrics checkpointMetrics) throws Exception {

   LOG.debug("Starting checkpoint ({}) {} on task {}",
      checkpointMetaData.getCheckpointId(), checkpointOptions.getCheckpointType(), getName());

   synchronized (lock) {
      if (isRunning) {
         // we can do a checkpoint

         // All of the following steps happen as an atomic step from the perspective of barriers and
         // records/watermarks/timers/callbacks.
         // We generally try to emit the checkpoint barrier as soon as possible to not affect downstream
         // checkpoint alignments

         // Step (1): Prepare the checkpoint, allow operators to do some pre-barrier work.
         //           The pre-barrier work should be nothing or minimal in the common case.
         operatorChain.prepareSnapshotPreBarrier(checkpointMetaData.getCheckpointId());

         // Step (2): Send the checkpoint barrier downstream
         operatorChain.broadcastCheckpointBarrier(
               checkpointMetaData.getCheckpointId(),
               checkpointMetaData.getTimestamp(),
               checkpointOptions);

         // Step (3): Take the state snapshot. This should be largely asynchronous, to not
         //           impact progress of the streaming topology
         checkpointState(checkpointMetaData, checkpointOptions, checkpointMetrics);
         return true;
      }
      else {
         ...
         return false;
      }
   }
}
```
首先来看第一步operatorChain.prepareSnapshotPreBarrier()，默认的实现其实什么也没做

```
//OperatorChain类
public void prepareSnapshotPreBarrier(long checkpointId) throws Exception {
   // go forward through the operator chain and tell each operator
   // to prepare the checkpoint
   final StreamOperator<?>[] operators = this.allOperators;
   for (int i = operators.length - 1; i >= 0; --i) {
      final StreamOperator<?> op = operators[i];
      if (op != null) {
         op.prepareSnapshotPreBarrier(checkpointId);
      }
   }
}

//AbstractStreamOperator类
public void prepareSnapshotPreBarrier(long checkpointId) throws Exception {
   // the default implementation does nothing and accepts the checkpoint
   // this is purely for subclasses to override
}
```

## 广播CheckpointBarrier

接下来我们看看operatorChain.broadcastCheckpointBarrier()，即CheckpointBarrier的广播过程。

其核心实现就是在task的输出ResultPartition里，给下游的每个通道channel都发送一个CheckpointBarrier，下游有多少个任务，那就会有多少个通道，给每个任务都会发送一个CheckpointBarrier，这个过程叫做广播。

这个CheckpointBarrier广播的过程也叫CheckpointBarrier注入，因为CheckpointBarrier并不是由执行source task的线程来写入的，而是由checkpoint线程来写入的，并且做了同步，在写入CheckpointBarrier时source task线程是被阻塞的。

这个源码如下：

```
//OperatorChain类
public void broadcastCheckpointBarrier(long id, long timestamp, CheckpointOptions checkpointOptions) throws IOException {
   CheckpointBarrier barrier = new CheckpointBarrier(id, timestamp, checkpointOptions);
   for (RecordWriterOutput<?> streamOutput : streamOutputs) {
      streamOutput.broadcastEvent(barrier);
   }
}

//RecordWriterOutput类
public void broadcastEvent(AbstractEvent event) throws IOException {
   recordWriter.broadcastEvent(event);
}

//RecordWriter类
public void broadcastEvent(AbstractEvent event) throws IOException {
   try (BufferConsumer eventBufferConsumer = EventSerializer.toBufferConsumer(event)) {
      for (int targetChannel = 0; targetChannel < numberOfChannels; targetChannel++) {
         tryFinishCurrentBufferBuilder(targetChannel);

         //这里的targetPartition就是ResultPartition，向下游每一个通道都发送一个CheckpointBarrier 
         // Retain the buffer so that it can be recycled by each channel of targetPartition
         targetPartition.addBufferConsumer(eventBufferConsumer.copy(), targetChannel);
      }

      if (flushAlways) {
         flushAll();
      }
   }
}
```

## 异步执行checkpoint

接下来就是关键的部分checkpointState()方法，也就是checkpoint的执行了，这个过程也是一个异步的过程，不能因为checkpoint而影响了正常数据流的处理。

StreamTask里的每个operator都会创建一个OperatorSnapshotFutures，OperatorSnapshotFutures 里包含了执行operator状态checkpoint的FutureTask，然后由另一个单独的线程异步的来执行这些operator的实际checkpoint操作
```
//StreamTask类
private void checkpointState(
      CheckpointMetaData checkpointMetaData,
      CheckpointOptions checkpointOptions,
      CheckpointMetrics checkpointMetrics) throws Exception {

   CheckpointStreamFactory storage = checkpointStorage.resolveCheckpointStorageLocation(
         checkpointMetaData.getCheckpointId(),
         checkpointOptions.getTargetLocation());

   CheckpointingOperation checkpointingOperation = new CheckpointingOperation(
      this,
      checkpointMetaData,
      checkpointOptions,
      storage,
      checkpointMetrics);

   checkpointingOperation.executeCheckpointing();
}
```

```
//StreamTask#CheckpointingOperation类
public void executeCheckpointing() throws Exception {
   startSyncPartNano = System.nanoTime();

   try {
      //对每个operator创建一个OperatorSnapshotFutures添加到operatorSnapshotsInProgress
      for (StreamOperator<?> op : allOperators) {
         checkpointStreamOperator(op);
      }

      startAsyncPartNano = System.nanoTime();

      checkpointMetrics.setSyncDurationMillis((startAsyncPartNano - startSyncPartNano) / 1_000_000);
        
      //由一个异步线程来执行实际的checkpoint操作，旨在不影响数据流的处理
      // we are transferring ownership over snapshotInProgressList for cleanup to the thread, active on submit
      AsyncCheckpointRunnable asyncCheckpointRunnable = new AsyncCheckpointRunnable(
         owner,
         operatorSnapshotsInProgress,
         checkpointMetaData,
         checkpointMetrics,
         startAsyncPartNano);

      owner.cancelables.registerCloseable(asyncCheckpointRunnable);
      owner.asyncOperationsThreadPool.submit(asyncCheckpointRunnable);

      ...
   } catch (Exception ex) {
      ...
      owner.synchronousCheckpointExceptionHandler.tryHandleCheckpointException(checkpointMetaData, ex);
   }
}

private void checkpointStreamOperator(StreamOperator<?> op) throws Exception {
   if (null != op) {
       //这个snapshotInProgress包含了operator checkpoint的FutureTask
      OperatorSnapshotFutures snapshotInProgress = op.snapshotState(
            checkpointMetaData.getCheckpointId(),
            checkpointMetaData.getTimestamp(),
            checkpointOptions,
            storageLocation);
      operatorSnapshotsInProgress.put(op.getOperatorID(), snapshotInProgress);
   }
}

```

## OperatorSnapshotFutures
我们首先来看看针对每个operator创建的OperatorSnapshotFutures是什么，OperatorSnapshotFutures持有了一个operator的状态数据快照过程的RunnableFuture，什么意思呢，RunnableFuture首先是一个Runnable，也就是把operator快照的操作写到run()方法里，但并不去执行，等到需要的时候再去执行run()方法，也就是去执行真正的快照操作。其次RunnableFuture是一个Future，在执行run()后可以调用get()来获取future的结果，这里的结果就是SnapshotResult。

```
public class OperatorSnapshotFutures {

   @Nonnull
   private RunnableFuture<SnapshotResult<KeyedStateHandle>> keyedStateManagedFuture;

   @Nonnull
   private RunnableFuture<SnapshotResult<KeyedStateHandle>> keyedStateRawFuture;

   @Nonnull
   private RunnableFuture<SnapshotResult<OperatorStateHandle>> operatorStateManagedFuture;

   @Nonnull
   private RunnableFuture<SnapshotResult<OperatorStateHandle>> operatorStateRawFuture;
```
从snapshotState()方法中可以看到OperatorSnapshotFutures中的keyedStateRawFuture和operatorStateRawFuture都是DoneFuture,也就是一个已经完成的RunnableFuture，那么这个过程就是一个同步的。

operatorState和keyedState的snapshot过程被封装到一个RunnableFuture（这个过程比较复杂，这里先不介绍），并不会立即执行，之后调用RunnableFuture.run()才会真正的执行snapshot，在这里RunnableFuture就是一个FutureTask，这个过程是一个异步的，会被一个异步快照线程执行
```
//AbstractStreamOperator类
public final OperatorSnapshotFutures snapshotState(long checkpointId, long timestamp, CheckpointOptions checkpointOptions,
      CheckpointStreamFactory factory) throws Exception {

   KeyGroupRange keyGroupRange = null != keyedStateBackend ?
         keyedStateBackend.getKeyGroupRange() : KeyGroupRange.EMPTY_KEY_GROUP_RANGE;

   OperatorSnapshotFutures snapshotInProgress = new OperatorSnapshotFutures();

   try (StateSnapshotContextSynchronousImpl snapshotContext = new StateSnapshotContextSynchronousImpl(
         checkpointId,
         timestamp,
         factory,
         keyGroupRange,
         getContainingTask().getCancelables())) {
      //将snapshotContext进行快照，这个过程也会调用CheckpointedFunction.snapshotState()方法
      snapshotState(snapshotContext);
    
      //keyedStateRawFuture和operatorStateRawFuture都是DoneFuture,也就是一个已经完成的RunnableFuture
      snapshotInProgress.setKeyedStateRawFuture(snapshotContext.getKeyedStateStreamFuture());
      snapshotInProgress.setOperatorStateRawFuture(snapshotContext.getOperatorStateStreamFuture());
    
      //operatorState和keyedState的snapshot过程被封装到一个RunnableFuture，并不会立即执行，
      //之后调用RunnableFuture.run()才会真正的执行snapshot
      if (null != operatorStateBackend) {
         snapshotInProgress.setOperatorStateManagedFuture(
            operatorStateBackend.snapshot(checkpointId, timestamp, factory, checkpointOptions));
      }

      if (null != keyedStateBackend) {
         snapshotInProgress.setKeyedStateManagedFuture(
            keyedStateBackend.snapshot(checkpointId, timestamp, factory, checkpointOptions));
      }
   } catch (Exception snapshotException) {
      ...
   }

   return snapshotInProgress;
}
```
## AsyncCheckpointRunnable

接下来我们看异步快照线程的实现，jobManagerTaskOperatorSubtaskStates是需要向JobManager ack的operator状态快照元数据信息，localTaskOperatorSubtaskStates是需要向TaskManager上报的operator状态快照元数据信息，localTaskOperatorSubtaskStates的作用是为状态数据保存一个备份，用户TaskManager快速的进行本地数据恢复，我们主要关心的还是向JobManager去Ack的TaskStateSnapshot。

1、针对每个operator创建一个OperatorSnapshotFinalizer，OperatorSnapshotFinalizer是状态数据快照的真正执行者，它真正的执行的operator的快照过程，也就是去执行那些FutureTask

2、状态快照执行完毕之后上JobManager上报checkpoint的信息
```
//AsyncCheckpointRunnable类
public void run() {
   FileSystemSafetyNet.initializeSafetyNetForThread();
   try {

      TaskStateSnapshot jobManagerTaskOperatorSubtaskStates =
         new TaskStateSnapshot(operatorSnapshotsInProgress.size());

      TaskStateSnapshot localTaskOperatorSubtaskStates =
         new TaskStateSnapshot(operatorSnapshotsInProgress.size());

      for (Map.Entry<OperatorID, OperatorSnapshotFutures> entry : operatorSnapshotsInProgress.entrySet()) {

         OperatorID operatorID = entry.getKey();
         OperatorSnapshotFutures snapshotInProgress = entry.getValue();
        
        //这里真正的执行的operator的快照过程，会执行那些FutureTask
         // finalize the async part of all by executing all snapshot runnables
         OperatorSnapshotFinalizer finalizedSnapshots =
            new OperatorSnapshotFinalizer(snapshotInProgress);

         jobManagerTaskOperatorSubtaskStates.putSubtaskStateByOperatorID(
            operatorID,
            finalizedSnapshots.getJobManagerOwnedState());

         localTaskOperatorSubtaskStates.putSubtaskStateByOperatorID(
            operatorID,
            finalizedSnapshots.getTaskLocalState());
      }

      final long asyncEndNanos = System.nanoTime();
      final long asyncDurationMillis = (asyncEndNanos - asyncStartNanos) / 1_000_000L;

      checkpointMetrics.setAsyncDurationMillis(asyncDurationMillis);

      if (asyncCheckpointState.compareAndSet(CheckpointingOperation.AsyncCheckpointState.RUNNING,
         CheckpointingOperation.AsyncCheckpointState.COMPLETED)) {
        //上JobManager上报checkpoint的信息
         reportCompletedSnapshotStates(
            jobManagerTaskOperatorSubtaskStates,
            localTaskOperatorSubtaskStates,
            asyncDurationMillis);

      } else {
         LOG.debug("{} - asynchronous part of checkpoint {} could not be completed because it was closed before.",
            owner.getName(),
            checkpointMetaData.getCheckpointId());
      }
   } catch (Exception e) {
      handleExecutionException(e);
   } finally {
      owner.cancelables.unregisterCloseable(this);
      FileSystemSafetyNet.closeSafetyNetAndGuardedResourcesForThread();
   }
}
```
可以看到在创建OperatorSnapshotFinalizer实例的时候就会去运行那些FutureTask，执行快照过程，比如将状态数据写到文件系统如HDFS等。
```
//OperatorSnapshotFinalizer类
public OperatorSnapshotFinalizer(
   @Nonnull OperatorSnapshotFutures snapshotFutures) throws ExecutionException, InterruptedException {

   SnapshotResult<KeyedStateHandle> keyedManaged =
      FutureUtils.runIfNotDoneAndGet(snapshotFutures.getKeyedStateManagedFuture());

   SnapshotResult<KeyedStateHandle> keyedRaw =
      FutureUtils.runIfNotDoneAndGet(snapshotFutures.getKeyedStateRawFuture());

   SnapshotResult<OperatorStateHandle> operatorManaged =
      FutureUtils.runIfNotDoneAndGet(snapshotFutures.getOperatorStateManagedFuture());

   SnapshotResult<OperatorStateHandle> operatorRaw =
      FutureUtils.runIfNotDoneAndGet(snapshotFutures.getOperatorStateRawFuture());

   jobManagerOwnedState = new OperatorSubtaskState(
      operatorManaged.getJobManagerOwnedSnapshot(),
      operatorRaw.getJobManagerOwnedSnapshot(),
      keyedManaged.getJobManagerOwnedSnapshot(),
      keyedRaw.getJobManagerOwnedSnapshot()
   );

   taskLocalState = new OperatorSubtaskState(
      operatorManaged.getTaskLocalSnapshot(),
      operatorRaw.getTaskLocalSnapshot(),
      keyedManaged.getTaskLocalSnapshot(),
      keyedRaw.getTaskLocalSnapshot()
   );
}
```

## Task上报checkpoint信息
上述说到Task在执行完checkpoint后会向JobManager上报checkpoint的元数据信息，最终会调用到CheckpointCoordinator.receiveAcknowledgeMessage()方法
```
//TaskStateManagerImpl类
public void reportTaskStateSnapshots(
   @Nonnull CheckpointMetaData checkpointMetaData,
   @Nonnull CheckpointMetrics checkpointMetrics,
   @Nullable TaskStateSnapshot acknowledgedState,
   @Nullable TaskStateSnapshot localState) {

   long checkpointId = checkpointMetaData.getCheckpointId();

   localStateStore.storeLocalState(checkpointId, localState);

   checkpointResponder.acknowledgeCheckpoint(
      jobId,
      executionAttemptID,
      checkpointId,
      checkpointMetrics,
      acknowledgedState);
}

//JobMaster类
public void acknowledgeCheckpoint(
      final JobID jobID,
      final ExecutionAttemptID executionAttemptID,
      final long checkpointId,
      final CheckpointMetrics checkpointMetrics,
      final TaskStateSnapshot checkpointState) {

   final CheckpointCoordinator checkpointCoordinator = executionGraph.getCheckpointCoordinator();
   final AcknowledgeCheckpoint ackMessage = new AcknowledgeCheckpoint(
      jobID,
      executionAttemptID,
      checkpointId,
      checkpointMetrics,
      checkpointState);

   if (checkpointCoordinator != null) {
      getRpcService().execute(() -> {
         try {
            checkpointCoordinator.receiveAcknowledgeMessage(ackMessage);
         } catch (Throwable t) {
            log.warn("Error while processing checkpoint acknowledgement message", t);
         }
      });
   } else {
      ...
   }
}
```
CheckpointCoordinator最后会调用PendingCheckpoint.acknowledgeTask()方法，该方法就是将task上报的元数据信息（checkpoint的路径地址，状态数据大小等等）添加到PendingCheckpoint里，如果接收到了全部task上报的的Ack消息，就执行completePendingCheckpoint()
```
//CheckpointCoordinator
public boolean receiveAcknowledgeMessage(AcknowledgeCheckpoint message) throws CheckpointException {
   ...
   
   final long checkpointId = message.getCheckpointId();

   synchronized (lock) {
      // we need to check inside the lock for being shutdown as well, otherwise we
      // get races and invalid error log messages
      if (shutdown) {
         return false;
      }

      final PendingCheckpoint checkpoint = pendingCheckpoints.get(checkpointId);

      if (checkpoint != null && !checkpoint.isDiscarded()) {

         switch (checkpoint.acknowledgeTask(message.getTaskExecutionId(), message.getSubtaskState(), message.getCheckpointMetrics())) {
            case SUCCESS:
               if (checkpoint.isFullyAcknowledged()) {
                  completePendingCheckpoint(checkpoint);
               }
               break;
            ...
          }

         return true;
      }
      ...
   }
}

//PendingCheckpoint类
public TaskAcknowledgeResult acknowledgeTask(
      ExecutionAttemptID executionAttemptId,
      TaskStateSnapshot operatorSubtaskStates,
      CheckpointMetrics metrics) {

   synchronized (lock) {
      if (discarded) {
         return TaskAcknowledgeResult.DISCARDED;
      }

      final ExecutionVertex vertex = notYetAcknowledgedTasks.remove(executionAttemptId);

      if (vertex == null) {
         if (acknowledgedTasks.contains(executionAttemptId)) {
            return TaskAcknowledgeResult.DUPLICATE;
         } else {
            return TaskAcknowledgeResult.UNKNOWN;
         }
      } else {
         acknowledgedTasks.add(executionAttemptId);
      }

      List<OperatorID> operatorIDs = vertex.getJobVertex().getOperatorIDs();
      int subtaskIndex = vertex.getParallelSubtaskIndex();
      long ackTimestamp = System.currentTimeMillis();

      long stateSize = 0L;
      //将上报的operatorSubtaskStates也就是每个Task的状态快照元数据添加到PendingCheckpoint的operatorStates里面
      if (operatorSubtaskStates != null) {
         for (OperatorID operatorID : operatorIDs) {

            OperatorSubtaskState operatorSubtaskState =
               operatorSubtaskStates.getSubtaskStateByOperatorID(operatorID);

            // if no real operatorSubtaskState was reported, we insert an empty state
            if (operatorSubtaskState == null) {
               operatorSubtaskState = new OperatorSubtaskState();
            }

            OperatorState operatorState = operatorStates.get(operatorID);

            if (operatorState == null) {
               operatorState = new OperatorState(
                  operatorID,
                  vertex.getTotalNumberOfParallelSubtasks(),
                  vertex.getMaxParallelism());
               operatorStates.put(operatorID, operatorState);
            }

            operatorState.putState(subtaskIndex, operatorSubtaskState);
            stateSize += operatorSubtaskState.getStateSize();
         }
      }

      ++numAcknowledgedTasks;

    ...

      return TaskAcknowledgeResult.SUCCESS;
   }
}
```
JobManager完成所有task的ack之后，会做以下操作：

1、将PendingCheckpoint 转成CompletedCheckpoint，标志着checkpoint过程完成，CompletedCheckpoint里包含了checkpoint的元数据信息，包括checkpoint的路径地址，状态数据大小等等，同时也会将元数据信息进行持久化，目录为$checkpointDir/$uuid/chk-***/_metadata，也会把过期的checkpoint数据给删除

2、通知所有的task进行commit操作，一般来说，task的commit操作其实不需要做什么，但是像那种TwoPhaseCommitSinkFunction，比如FlinkKafkaProducer，就会进行一些事物的提交操作等。

```
private void completePendingCheckpoint(PendingCheckpoint pendingCheckpoint) throws CheckpointException {
   final long checkpointId = pendingCheckpoint.getCheckpointId();
   final CompletedCheckpoint completedCheckpoint;

   // As a first step to complete the checkpoint, we register its state with the registry
   Map<OperatorID, OperatorState> operatorStates = pendingCheckpoint.getOperatorStates();
   sharedStateRegistry.registerAll(operatorStates.values());

   try {
      try {
         completedCheckpoint = pendingCheckpoint.finalizeCheckpoint();
      }
      catch (Exception e1) {
          ...
       }

      // the pending checkpoint must be discarded after the finalization
      Preconditions.checkState(pendingCheckpoint.isDiscarded() && completedCheckpoint != null);

      try {
         completedCheckpointStore.addCheckpoint(completedCheckpoint);
      } catch (Exception exception) {
      ...
     }
   } finally {
      pendingCheckpoints.remove(checkpointId);

      triggerQueuedRequests();
   }

   rememberRecentCheckpointId(checkpointId);

   // drop those pending checkpoints that are at prior to the completed one
   dropSubsumedCheckpoints(checkpointId);

   // record the time when this was completed, to calculate
   // the 'min delay between checkpoints'
   lastCheckpointCompletionNanos = System.nanoTime();

  ...

   // send the "notify complete" call to all vertices
   final long timestamp = completedCheckpoint.getTimestamp();

   for (ExecutionVertex ev : tasksToCommitTo) {
      Execution ee = ev.getCurrentExecutionAttempt();
      if (ee != null) {
         ee.notifyCheckpointComplete(checkpointId, timestamp);
      }
   }
}
```

## JobManager通知Task进行commit
task在接收到消息之后会调用Task.notifyCheckpointComplete()方法，最后会调用StreamOperator.notifyCheckpointComplete()，一般来说不做什么操作。但是像AbstractUdfStreamOperator这种的可能还会由一些其他操作

```
public void notifyCheckpointComplete(final long checkpointID) {
   final AbstractInvokable invokable = this.invokable;

   if (executionState == ExecutionState.RUNNING && invokable != null) {

      Runnable runnable = new Runnable() {
         @Override
         public void run() {
            try {
               invokable.notifyCheckpointComplete(checkpointID);
               taskStateManager.notifyCheckpointComplete(checkpointID);
            } catch (Throwable t) {
               ...
            }
         }
      };
      executeAsyncCallRunnable(runnable, "Checkpoint Confirmation for " +
         taskNameWithSubtask);
   }
   else {
      LOG.debug("Ignoring checkpoint commit notification for non-running task {}.", taskNameWithSubtask);
   }
}
```

```
//StreamTask类
public void notifyCheckpointComplete(long checkpointId) throws Exception {
   synchronized (lock) {
      if (isRunning) {
         LOG.debug("Notification of complete checkpoint for task {}", getName());

         for (StreamOperator<?> operator : operatorChain.getAllOperators()) {
            if (operator != null) {
               operator.notifyCheckpointComplete(checkpointId);
            }
         }
      }
      else {
         LOG.debug("Ignoring notification of complete checkpoint for not-running task {}", getName());
      }
   }
}

//AbstractStreamOperator类
public void notifyCheckpointComplete(long checkpointId) throws Exception {
   if (keyedStateBackend != null) {
      keyedStateBackend.notifyCheckpointComplete(checkpointId);
   }
}

//HeapKeyedStateBackend类
public void notifyCheckpointComplete(long checkpointId) {
   //Nothing to do
}
```
AbstractUdfStreamOperator主要是针对用户自定义函数的operator，像StreamMap，StreamSource等等，如果用户定义的Function实现了CheckpointListener接口，则会进行额外的一些处理，例如FlinkKafkaConsumerBase会向kafka提交消费的offset，TwoPhaseCommitSinkFunction类会进行事务的提交，例如FlinkKafkaProducer。值得一提的是，TwoPhaseCommitSinkFunction是保证flink端到端exactly-once的保证
```
//AbstractUdfStreamOperator
public void notifyCheckpointComplete(long checkpointId) throws Exception {
   super.notifyCheckpointComplete(checkpointId);

   if (userFunction instanceof CheckpointListener) {
      ((CheckpointListener) userFunction).notifyCheckpointComplete(checkpointId);
   }
}
```

```
//FlinkKafkaConsumerBase类
public final void notifyCheckpointComplete(long checkpointId) throws Exception {
   if (!running) {
      LOG.debug("notifyCheckpointComplete() called on closed source");
      return;
   }

   final AbstractFetcher<?, ?> fetcher = this.kafkaFetcher;
   if (fetcher == null) {
      LOG.debug("notifyCheckpointComplete() called on uninitialized source");
      return;
   }

   if (offsetCommitMode == OffsetCommitMode.ON_CHECKPOINTS) {
      // only one commit operation must be in progress
      if (LOG.isDebugEnabled()) {
         LOG.debug("Committing offsets to Kafka/ZooKeeper for checkpoint " + checkpointId);
      }

      try {
         ...
         @SuppressWarnings("unchecked")
         Map<KafkaTopicPartition, Long> offsets =
            (Map<KafkaTopicPartition, Long>) pendingOffsetsToCommit.remove(posInMap);

         ...
         //向kafka提交offset
         fetcher.commitInternalOffsetsToKafka(offsets, offsetCommitCallback);
      } catch (Exception e) {
         ...
      }
   }
}
```

```
//TwoPhaseCommitSinkFunction类
public final void notifyCheckpointComplete(long checkpointId) throws Exception {
   
   Iterator<Map.Entry<Long, TransactionHolder<TXN>>> pendingTransactionIterator = pendingCommitTransactions.entrySet().iterator();
   checkState(pendingTransactionIterator.hasNext(), "checkpoint completed, but no transaction pending");
   Throwable firstError = null;

   while (pendingTransactionIterator.hasNext()) {
      ...
      try {
          //提交事务
         commit(pendingTransaction.handle);
      } catch (Throwable t) {
        ...
      }
    ...
   }
    ...
}

//FlinkKafkaProducer类
protected void commit(FlinkKafkaProducer.KafkaTransactionState transaction) {
   if (transaction.isTransactional()) {
      try {
         transaction.producer.commitTransaction();
      } finally {
         recycleTransactionalProducer(transaction.producer);
      }
   }
}
```
所有的task完成了notifyCheckpointComplete()方法后，一个完整的checkpoint流程就完成了。

## 非SourceStreamTask的checkpoint实现
上述只说了source task的checkpoint实现，source task的checkpoint是由JobManager来触发的，那么非source task的checkpoint流程又是如何的呢？

上面说了source task会向下游广播发送CheckpointBarrier，那么下游的task就会接收到source task发送的CheckpointBarrier，checkpoint的起始位置也在接收到CheckpointBarrier。起点就在StreamTask中的CheckpointBarrierHandler.getNextNonBlocked()中
```
//StreamInputProcessor类
public boolean processInput() throws Exception {
   ...

   while (true) {
     ...
    //
      final BufferOrEvent bufferOrEvent = barrierHandler.getNextNonBlocked();
      ...
   }
}
```
CheckpointBarrierHandler会根据CheckpointingMode模式不同生成不同的Handler，如果是EXACTLY_ONCE，就会生成BarrierBuffer，会进行barrier对齐，BarrierBuffer中的CachedBufferBlocker是用来缓存barrier对齐时从被阻塞channel接收到的数据。如果CheckpointingMode是AT_LEAST_ONCE，那就会生成BarrierTracker，不会进行barrier对齐。
```
public static CheckpointBarrierHandler createCheckpointBarrierHandler(
      StreamTask<?, ?> checkpointedTask,
      CheckpointingMode checkpointMode,
      IOManager ioManager,
      InputGate inputGate,
      Configuration taskManagerConfig) throws IOException {

   CheckpointBarrierHandler barrierHandler;
   if (checkpointMode == CheckpointingMode.EXACTLY_ONCE) {
      long maxAlign = taskManagerConfig.getLong(TaskManagerOptions.TASK_CHECKPOINT_ALIGNMENT_BYTES_LIMIT);
      if (!(maxAlign == -1 || maxAlign > 0)) {
         throw new IllegalConfigurationException(
            TaskManagerOptions.TASK_CHECKPOINT_ALIGNMENT_BYTES_LIMIT.key()
            + " must be positive or -1 (infinite)");
      }

      if (taskManagerConfig.getBoolean(TaskManagerOptions.NETWORK_CREDIT_MODEL)) {
         barrierHandler = new BarrierBuffer(inputGate, new CachedBufferBlocker(inputGate.getPageSize()), maxAlign);
      } else {
         barrierHandler = new BarrierBuffer(inputGate, new BufferSpiller(ioManager, inputGate.getPageSize()), maxAlign);
      }
   } else if (checkpointMode == CheckpointingMode.AT_LEAST_ONCE) {
      barrierHandler = new BarrierTracker(inputGate);
   } else {
      throw new IllegalArgumentException("Unrecognized Checkpointing Mode: " + checkpointMode);
   }

   if (checkpointedTask != null) {
      barrierHandler.registerCheckpointEventHandler(checkpointedTask);
   }

   return barrierHandler;
}
```
## EXACTLY_ONCE下checkpoint的实现

下面看看CheckpointBarrierHandler的处理逻辑，首先看BarrierBuffer

1、首先获取到下一个BufferOrEvent，可能是从当前缓存中获取获取，如果缓存没了，就从inputGate获取

2、如果获取的事件是已经被设置为阻塞的channel的，就将其放到缓存中，如果channel已经接收到CheckpointBarrier，就会将其设置为block状态

3、如果接收的事件是CheckpointBarrier，处理接收到的CheckpointBarrier，如果接收到的是正常的数据就返回
```
//BarrierBuffer类
public BufferOrEvent getNextNonBlocked() throws Exception {
   while (true) {
      // process buffered BufferOrEvents before grabbing new ones
      Optional<BufferOrEvent> next;
      if (currentBuffered == null) {
         next = inputGate.getNextBufferOrEvent();
      }
      else {
         next = Optional.ofNullable(currentBuffered.getNext());
         if (!next.isPresent()) {
            completeBufferedSequence();
            return getNextNonBlocked();
         }
      }

      ...

      BufferOrEvent bufferOrEvent = next.get();
      if (isBlocked(bufferOrEvent.getChannelIndex())) {
          //如果数据是被阻塞的channel，就将其添加到缓存，如果channel已经接收到CheckpointBarrier，就会将其设置为block状态
         // if the channel is blocked, we just store the BufferOrEvent
         bufferBlocker.add(bufferOrEvent);
         checkSizeLimit();
      }
      else if (bufferOrEvent.isBuffer()) {
         return bufferOrEvent;
      }
      else if (bufferOrEvent.getEvent().getClass() == CheckpointBarrier.class) {
         if (!endOfStream) {
             //处理接收到的CheckpointBarrier
            // process barriers only if there is a chance of the checkpoint completing
            processBarrier((CheckpointBarrier) bufferOrEvent.getEvent(), bufferOrEvent.getChannelIndex());
         }
      }
      else if (bufferOrEvent.getEvent().getClass() == CancelCheckpointMarker.class) {
         processCancellationBarrier((CancelCheckpointMarker) bufferOrEvent.getEvent());
      }
      else {
         if (bufferOrEvent.getEvent().getClass() == EndOfPartitionEvent.class) {
            processEndOfPartition();
         }
         return bufferOrEvent;
      }
   }
}

//CachedBufferBlocker类
public void add(BufferOrEvent boe) {
   bytesBlocked += pageSize;
    //currentBuffers是一个ArrayDeque<BufferOrEvent>
   currentBuffers.add(boe);
}
```
核心处理逻辑在processBarrier()方法中，实现如下：

1、如果只有一个channel，也就是说只有一个上游任务，就立马触发checkpoint

2、如果有多个上游任务，也就是有多个channel，那么会有以下几种情况的判断

1)、如果是首次接收到barrier，就开始进行barrier对齐，并将该channel设置为阻塞状态

2)、如果不是首次接收到barrier，但也不是最后一个barrier，就只给channel设置为阻塞状态

3)、如果在没完成当前checkpoint的时候又接收到了下一次checkpoint的barrier，就终止当前的checkpoint，重新开始新的一次checkpoint，这种情况并不常见

4)、如果接收到全部channel，即上游所有任务的barrier，就开始触发checkpoint，并取消所有channel的阻塞状态，开始处理那些被添加到缓存的事件
```
private void processBarrier(CheckpointBarrier receivedBarrier, int channelIndex) throws Exception {
   final long barrierId = receivedBarrier.getId();

   // fast path for single channel cases
   if (totalNumberOfInputChannels == 1) {
      if (barrierId > currentCheckpointId) {
         // new checkpoint
         currentCheckpointId = barrierId;
         notifyCheckpoint(receivedBarrier);
      }
      return;
   }

   // -- general code path for multiple input channels --

   if (numBarriersReceived > 0) {
      // this is only true if some alignment is already progress and was not canceled

      if (barrierId == currentCheckpointId) {
         // regular case
         onBarrier(channelIndex);
      }
      else if (barrierId > currentCheckpointId) {
          //这种情况就是上一个checkpoint还没完呢，就接收到下一个checkpoint的barrier，这种情况，就终止当前的checkpoint
         // we did not complete the current checkpoint, another started before
         ...
         // let the task know we are not completing this
         notifyAbort(currentCheckpointId, new CheckpointDeclineSubsumedException(barrierId));

         // abort the current checkpoint
         releaseBlocksAndResetBarriers();

         // begin a the new checkpoint
         beginNewAlignment(barrierId, channelIndex);
      }
      else {
         // ignore trailing barrier from an earlier checkpoint (obsolete now)
         return;
      }
   }
   else if (barrierId > currentCheckpointId) {
      // first barrier of a new checkpoint
      beginNewAlignment(barrierId, channelIndex);
   }
   else {
      // either the current checkpoint was canceled (numBarriers == 0) or
      // this barrier is from an old subsumed checkpoint
      return;
   }
    //接收到所有channel的barrier，开始触发checkpoint
   // check if we have all barriers - since canceled checkpoints always have zero barriers
   // this can only happen on a non canceled checkpoint
   if (numBarriersReceived + numClosedChannels == totalNumberOfInputChannels) {
      // actually trigger checkpoint
      ...
      //释放所有channel的阻塞状态
      releaseBlocksAndResetBarriers();
      //开始触发checkpoint
      notifyCheckpoint(receivedBarrier);
   }
}

//开始barrier对齐
private void beginNewAlignment(long checkpointId, int channelIndex) throws IOException {
   currentCheckpointId = checkpointId;
   onBarrier(channelIndex);

   startOfAlignmentTimestamp = System.nanoTime();

   ...
}

//onBarrier()就是将其channel设置为阻塞状态
private void onBarrier(int channelIndex) throws IOException {
   if (!blockedChannels[channelIndex]) {
      blockedChannels[channelIndex] = true;

      numBarriersReceived++;

      ...
   }
   else {
      throw new IOException("Stream corrupt: Repeated barrier for same checkpoint on input " + channelIndex);
   }
}
```
触发checkpoint的逻辑在notifyCheckpoint()方法中，toNotifyOnCheckpoint就是StreamTask，最后还是调用了StreamTask.triggerCheckpointOnBarrier()，就跟上述描述的source task的checkpoint逻辑是一样的了。
```
private void notifyCheckpoint(CheckpointBarrier checkpointBarrier) throws Exception {
   if (toNotifyOnCheckpoint != null) {
      CheckpointMetaData checkpointMetaData =
            new CheckpointMetaData(checkpointBarrier.getId(), checkpointBarrier.getTimestamp());

      long bytesBuffered = currentBuffered != null ? currentBuffered.size() : 0L;

      CheckpointMetrics checkpointMetrics = new CheckpointMetrics()
            .setBytesBufferedInAlignment(bytesBuffered)
            .setAlignmentDurationNanos(latestAlignmentDurationNanos);

      toNotifyOnCheckpoint.triggerCheckpointOnBarrier(
         checkpointMetaData,
         checkpointBarrier.getCheckpointOptions(),
         checkpointMetrics);
   }
}

//StreamTask
public void triggerCheckpointOnBarrier(
      CheckpointMetaData checkpointMetaData,
      CheckpointOptions checkpointOptions,
      CheckpointMetrics checkpointMetrics) throws Exception {

   try {
      performCheckpoint(checkpointMetaData, checkpointOptions, checkpointMetrics);
   }
   catch (CancelTaskException e) {
      ...
   }
}
```

## AT_LEAST_ONCE下的checkpoint实现

在AT_LEAST_ONCE模式下，CheckpointBarrierHandler是BarrierTracker

BarrierTracker.getNextNonBlocked()的方法中，事件都是从inputGate中获取的，并且也没有缓存那些已经接收到barrier的channel后续过来的数据，而是继续处理
```
//BarrierTracker类
public BufferOrEvent getNextNonBlocked() throws Exception {
   while (true) {
      Optional<BufferOrEvent> next = inputGate.getNextBufferOrEvent();
      if (!next.isPresent()) {
         // buffer or input exhausted
         return null;
      }

      BufferOrEvent bufferOrEvent = next.get();
      if (bufferOrEvent.isBuffer()) {
         return bufferOrEvent;
      }
      else if (bufferOrEvent.getEvent().getClass() == CheckpointBarrier.class) {
         processBarrier((CheckpointBarrier) bufferOrEvent.getEvent(), bufferOrEvent.getChannelIndex());
      }
      else if (bufferOrEvent.getEvent().getClass() == CancelCheckpointMarker.class) {
         processCheckpointAbortBarrier((CancelCheckpointMarker) bufferOrEvent.getEvent(), bufferOrEvent.getChannelIndex());
      }
      else {
         // some other event
         return bufferOrEvent;
      }
   }
}
```
核心实现还是在processBarrier()中，实现如下：

1、如果一个channel，也即上游只有一个channel，就立即触发checkpoint

2、在多个channel的情况下，也即多个上游任务，有如下几种情况

1)、如果是首次接收到barrier，只更新一下当前的checkpointID

2)、不是首次，但也不是最后一次接收到barrier，基本什么也不做，就累加一个计数器

3)、如果接收到所有channel的barrier，就开始触发checkpoint

因为BarrierTracker在等待所有channel的barrier到来的时候还会处理其他已经到来barrier的channel的数据，所以在进行状态数据快照的时候就会造成多了一部分数据的状态，快照并不具备一致性。在程序从checkpoint进行恢复的时候会重复处理这部分数据，所以才会出现AT_LEAST_ONCE
```
//BarrierTracker类
private void processBarrier(CheckpointBarrier receivedBarrier, int channelIndex) throws Exception {
   final long barrierId = receivedBarrier.getId();

   // fast path for single channel trackers
   if (totalNumberOfInputChannels == 1) {
      notifyCheckpoint(barrierId, receivedBarrier.getTimestamp(), receivedBarrier.getCheckpointOptions());
      return;
   }

   // general path for multiple input channels
   if (LOG.isDebugEnabled()) {
      LOG.debug("Received barrier for checkpoint {} from channel {}", barrierId, channelIndex);
   }

   // find the checkpoint barrier in the queue of pending barriers
   CheckpointBarrierCount cbc = null;
   int pos = 0;

   for (CheckpointBarrierCount next : pendingCheckpoints) {
      if (next.checkpointId == barrierId) {
         cbc = next;
         break;
      }
      pos++;
   }

   if (cbc != null) {
      // add one to the count to that barrier and check for completion
      int numBarriersNew = cbc.incrementBarrierCount();
      //当接收到所有channel的barrier，开始触发checkpoint
      if (numBarriersNew == totalNumberOfInputChannels) {
         // checkpoint can be triggered (or is aborted and all barriers have been seen)
         // first, remove this checkpoint and all all prior pending
         // checkpoints (which are now subsumed)
         for (int i = 0; i <= pos; i++) {
            pendingCheckpoints.pollFirst();
         }

         // notify the listener
         if (!cbc.isAborted()) {
            if (LOG.isDebugEnabled()) {
               LOG.debug("Received all barriers for checkpoint {}", barrierId);
            }

            notifyCheckpoint(receivedBarrier.getId(), receivedBarrier.getTimestamp(), receivedBarrier.getCheckpointOptions());
         }
      }
   }
   else {
      // first barrier for that checkpoint ID
      // add it only if it is newer than the latest checkpoint.
      // if it is not newer than the latest checkpoint ID, then there cannot be a
      // successful checkpoint for that ID anyways
      if (barrierId > latestPendingCheckpointID) {
         latestPendingCheckpointID = barrierId;
         pendingCheckpoints.addLast(new CheckpointBarrierCount(barrierId));

         // make sure we do not track too many checkpoints
         if (pendingCheckpoints.size() > MAX_CHECKPOINTS_TO_TRACK) {
            pendingCheckpoints.pollFirst();
         }
      }
   }
}
```
notifyCheckpoint()方法也是调用了StreamTask.triggerCheckpointOnBarrier()，跟BarrierBuffer是一样的。
```
//BarrierTracker类
private void notifyCheckpoint(long checkpointId, long timestamp, CheckpointOptions checkpointOptions) throws Exception {
   if (toNotifyOnCheckpoint != null) {
      CheckpointMetaData checkpointMetaData = new CheckpointMetaData(checkpointId, timestamp);
      CheckpointMetrics checkpointMetrics = new CheckpointMetrics()
         .setBytesBufferedInAlignment(0L)
         .setAlignmentDurationNanos(0L);

      toNotifyOnCheckpoint.triggerCheckpointOnBarrier(checkpointMetaData, checkpointOptions, checkpointMetrics);
   }
}
```
上述的流程会一直执行到sink task，sink task执行完checkpoint，整个checkpoint就完成了。

## 总结：

checkpoint的过程包含了JobManager和Taskmanager端task的执行过程，按照步骤为：

1、在JobManager端构建ExecutionGraph过程中会创建CheckpointCoordinator，这是负责checkpoint的核心实现类，同时会给job添加一个监听器CheckpointCoordinatorDeActivator（只有设置了checkpoint才会注册这个监听器），在JobManager端开始进行任务调度的时候，会对job的状态进行转换，由CREATED转成RUNNING，job监听器CheckpointCoordinatorDeActivator就开始启动checkpoint的定时任务了，最终会调用CheckpointCoordinator.startCheckpointScheduler()

2、CheckpointCoordinator会部署一个定时任务，用于周期性的触发checkpoint，这个定时任务就是ScheduledTrigger，在触发checkpoint之前先做一遍检查，检查当前正在处理的checkpoint是否超过设置的最大并发checkpoint数量，检查checkpoint的间隔是否达到设置的两次checkpoint的时间间隔，在都没有问题的情况下向所有的source task去触发checkpoint，远程调用TaskManager的triggerCheckpoint()方法

3、TaskManager的triggerCheckpoint()方法首先获取到source task（即SourceStreamTask），调用Task.triggerCheckpointBarrier()，triggerCheckpointBarrier()会异步的去执行一个独立线程，这个线程来负责source task的checkpoint执行。checkpoint的核心实现在StreamTask.performCheckpoint()方法中，该方法主要有三个步骤

1)、在checkpoint之前做一些准备工作，通常情况下operator在这个阶段是不做什么操作的

2)、立即向下游广播CheckpointBarrier，以便使下游的task能够及时的接收到CheckpointBarrier也开始进行checkpoint的操作

3)、开始进行状态的快照，即checkpoint操作。

注意以上操作都是在同步代码块里进行的，获取到的这个lock锁就是用于checkpoint的锁，checkpoint线程和task任务线程用的是同一把锁，在进行performCheckpoint()时，task任务线程是不能够进行数据处理的

4、checkpoint的执行过程是一个异步的过程，保证不能因为checkpoint而影响了正常数据流的处理。StreamTask里的每个operator都会创建一个OperatorSnapshotFutures，OperatorSnapshotFutures 里包含了执行operator状态checkpoint的FutureTask，然后由另一个单独的线程异步的来执行这些operator的实际checkpoint操作，就是执行这些FutureTask。这个异步线程叫做AsyncCheckpointRunnable，checkpoint的执行就是将状态数据推送到远程的存储介质中

5、对于非Source Task，checkpoint的标志性开始在接收到上游的CheckpointBarrier，方法在StreamTask中的CheckpointBarrierHandler.getNextNonBlocked()。CheckpointBarrierHandler会根据CheckpointingMode模式不同生成不同的Handler，如果是EXACTLY_ONCE，就会生成BarrierBuffer，会进行barrier对齐，保证数据的一致性，BarrierBuffer中的CachedBufferBlocker是用来缓存barrier对齐时从被阻塞channel接收到的数据。如果CheckpointingMode是AT_LEAST_ONCE，那就会生成BarrierTracker，不会进行barrier对齐，而是继续处理数据，在接收到上游task所有的CheckpointBarrier才开始进程checkpoint，这样就会checkpoint(n)的状态会包含checkpoint(n+1)的数据，数据不一致。非Source Task的checkpoint执行跟步骤3、4是一样的，只不过触发的线程是Task工作线程，跟source task不一样

6、Task在执行完checkpoint后会向JobManager上报checkpoint的元数据信息，JobManager端的CheckpointCoordinator会调用PendingCheckpoint.acknowledgeTask()方法，该方法就是将task上报的元数据信息（checkpoint的路径地址，状态数据大小等等）添加到PendingCheckpoint里

7、task的checkpoint会一直进行到sink task。JobManager如果接收到了全部task上报的的Ack消息，就执行completePendingCheckpoint()，会将checkpoint元数据信息进行持久化，然后通知所有的task进行commit操作，一般来说，task的commit操作其实不需要做什么，但是像那种TwoPhaseCommitSinkFunction，比如FlinkKafkaProducer就会进行一些事物的提交操作等，或者像FlinkKafkaConsumer会进行offset的提交

8、所有task执行完commit操作后（实际上执行的是operator.notifyCheckpointComplete()方法），一个完整的checkpoint流程就完成了