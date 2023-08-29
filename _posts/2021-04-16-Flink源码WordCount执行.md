---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink源码WordCount执行
涉及到flink的核心任务执行系统，适用于任何任务的执行，而不单单是WordCount这个例子。

## WordCount源码分析
WordCount代码经过jobgraph的构造，可以将任务划分为：
```
source(p=1)——>flatMap+map+keyBy(p=2)——>window+reduce(p=2)——>print(p=1)
```
p代表并行度

## 任务执行起点
flink任务执行是会调用Task.run()，核心又在invokable.invoke()方法中，invokable是AbstractInvokable类型,在WordCount中分为两类，一是SourceStreamTask,用于执行数据源逻辑； 一是OneInputStreamTask，除Source外都是OneInputStreamTask。

AbstractInvokable的直接继承类是StreamTask
```
@Internal
public abstract class StreamTask<OUT, OP extends StreamOperator<OUT>>
      extends AbstractInvokable
      implements AsyncExceptionHandler {

   /** The thread group that holds all trigger timer threads. */
   public static final ThreadGroup TRIGGER_THREAD_GROUP = new ThreadGroup("Triggers");

   /** The logger used by the StreamTask and its subclasses. */
   private static final Logger LOG = LoggerFactory.getLogger(StreamTask.class);
   ...
```
StreamTask的invoke()方法核心代码：
```
// -------- Invoke --------
LOG.debug("Invoking {}", getName());

// we need to make sure that any triggers scheduled in open() cannot be
// executed before all operators are opened
synchronized (lock) {

   // both the following operations are protected by the lock
   // so that we avoid race conditions in the case that initializeState()
   // registers a timer, that fires before the open() is called.

   initializeState();
   //打开所有的operator，此处会调用function的open方法
   openAllOperators();
}

// final check to exit early before starting to run
if (canceled) {
   throw new CancelTaskException();
}

// let the task do its work
isRunning = true;
//run，task正式开始执行逻辑的方法
run();

if (canceled) {
   throw new CancelTaskException();
}
```
可见，StreamTask的核心就是调用run()方法。

## 任务执行核心

## SourceStreamTask
SourceStreamTask继承了StreamTask，run()方法：
```
@Override
protected void run() throws Exception {
   headOperator.run(getCheckpointLock(), getStreamStatusMaintainer());
}
```
SourceStreamTask.run()其实就是调用的StreamSource.run()，核心便是调用SourceFunction.run()，采集数据源，并将数据分发到下游，WordCount程序里sourceFunction就是SocketTextStreamFunction，用的partitioner是RebalancePartitioner，数据轮询发送到下游

## OneInputStreamTask
OneInputStreamTask继承了StreamTask
```
public class OneInputStreamTask<IN, OUT> extends StreamTask<OUT, OneInputStreamOperator<IN, OUT>> {

   private StreamInputProcessor<IN> inputProcessor;

   private volatile boolean running = true;

   private final WatermarkGauge inputWatermarkGauge = new WatermarkGauge();

   /**
    * Constructor for initialization, possibly with initial state (recovery / savepoint / etc).
    *
    * @param env The task environment for this task.
    */
   public OneInputStreamTask(Environment env) {
      super(env);
   }
```

```
@Override
protected void run() throws Exception {
   // cache processor reference on the stack, to make the code more JIT friendly
   final StreamInputProcessor<IN> inputProcessor = this.inputProcessor;

   while (running && inputProcessor.processInput()) {
      // all the work happens in the "processInput" method
   }
}
```
run()方法核心就是调用inputProcessor.processInput()，inputProcessor是输入数据的核心处理器。这个operator就是headOperator，在这里的第一个就是StreamFlatMap。

## 数据转换
setKeyContextElement1()方法就是根据keySelector设置当前的key，对于flatMap和map的operator来说，keySelector是null，这一步就不做任何操作了。
```
@Override
@SuppressWarnings({"unchecked", "rawtypes"})
public void setKeyContextElement1(StreamRecord record) throws Exception {
   setKeyContextElement(record, stateKeySelector1);
}

@Override
@SuppressWarnings({"unchecked", "rawtypes"})
public void setKeyContextElement2(StreamRecord record) throws Exception {
   setKeyContextElement(record, stateKeySelector2);
}

private <T> void setKeyContextElement(StreamRecord<T> record, KeySelector<T, ?> selector) throws Exception {
   if (selector != null) {
      Object key = selector.getKey(record.getValue());
      setCurrentKey(key);
   }
}
```
StreamFlatMap这个operator的processElement方法就是调用用户的flatMap函数，并收集起来发给下游。因为flatMap和map这两个operator是chain在一起的，所以在flatMap的Output的类型CopyingChainingOutput，CopyingChainingOutput.collect方法是将经过flatMap转换过的数据直接发给下一个operator，也就是StreamMap，中间的数据不经过序列化和网络传输。
```
@Override
public void processElement(StreamRecord<IN> element) throws Exception {
   collector.setTimestamp(element);
   userFunction.flatMap(element.getValue(), collector);
}
@Override
public void collect(StreamRecord<T> record) {
   if (this.outputTag != null) {
      // we are only responsible for emitting to the main input
      return;
   }

   pushToOperator(record);
}

@Override
protected <X> void pushToOperator(StreamRecord<X> record) {
   try {
      // we know that the given outputTag matches our OutputTag so the record
      // must be of the type that our operator (and Serializer) expects.
      @SuppressWarnings("unchecked")
      StreamRecord<T> castRecord = (StreamRecord<T>) record;

      numRecordsIn.inc();
      StreamRecord<T> copy = castRecord.copy(serializer.copy(castRecord.getValue()));
      operator.setKeyContextElement1(copy);
      operator.processElement(copy);
   } catch (ClassCastException e) {
```
StreamMap同样会调用setKeyContextElement1和processElement，同样StreamMap的setKeyContextElement1不做任何操作，processElement如下：
```
@Override
public void processElement(StreamRecord<IN> element) throws Exception {
   output.collect(element.replace(userFunction.map(element.getValue())));
}
```
即调用用户的map函数，然后收集起来发送到下游。keyBy算子不会产生任何operator，只是提供数据分区器。StreamMap这个operator处理后的数据将会经过网络传输发送到下游的window窗口。

收集StreamMap数据的output类型是RecordWriterOutput，它的collect方法调用的是pushToRecordWriter
```
@Override
public void collect(StreamRecord<OUT> record) {
   if (this.outputTag != null) {
      // we are only responsible for emitting to the main input
      return;
   }

   pushToRecordWriter(record);
}

private <X> void pushToRecordWriter(StreamRecord<X> record) {
   serializationDelegate.setInstance(record);

   try {
      recordWriter.emit(serializationDelegate);
   }
   catch (Exception e) {
      throw new RuntimeException(e.getMessage(), e);
   }
}

public void emit(T record) throws IOException, InterruptedException {
   checkErroneous();
   emit(record, channelSelector.selectChannel(record));
}
```
pushToRecordWriter最终会调用emit(record, channelSelector.selectChannel(record))来决定这条记录发往下游的哪个通道，即发送给下游的哪个task或者分区。这里的channelSelector就是keyBy算子过程中生成的KeyGroupStreamPartitioner，详细参考上节KeyGroupStreamPartitioner.selectChannel

## 窗口操作
下游的Task类型也是OneInputStreamTask。同样会调用inputProcessor.processInput()，也就是
```
StreamRecord<IN> record = recordOrMark.asRecord();
synchronized (lock) {
   numRecordsIn.inc();
   streamOperator.setKeyContextElement1(record);
   streamOperator.processElement(record);
}
```

下游的headOperator是windowOperator，不同的是，windowOperator现在拥有了keySelector，setKeyContextElement1会设置当前的key。processElement方法如下
```
public void processElement(StreamRecord<IN> element) throws Exception {
    //为数据分配窗口
   final Collection<W> elementWindows = windowAssigner.assignWindows(
      element.getValue(), element.getTimestamp(), windowAssignerContext);

   //if element is handled by none of assigned elementWindows
   boolean isSkippedElement = true;
    //获取当前的key，key由setKeyContextElement1方法中设置
   final K key = this.<K>getKeyedStateBackend().getCurrentKey();
    //不走这个if
   if (windowAssigner instanceof MergingWindowAssigner) {
      MergingWindowSet<W> mergingWindows = getMergingWindowSet();

      for (W window: elementWindows) {

         // adding the new window might result in a merge, in that case the actualWindow
         // is the merged window and we work with that. If we don't merge then
         // actualWindow == window
         W actualWindow = mergingWindows.addWindow(window, new MergingWindowSet.MergeFunction<W>() {
            @Override
            public void merge(W mergeResult,
                  Collection<W> mergedWindows, W stateWindowResult,
                  Collection<W> mergedStateWindows) throws Exception {

               if ((windowAssigner.isEventTime() && mergeResult.maxTimestamp() + allowedLateness <= internalTimerService.currentWatermark())) {
                  throw new UnsupportedOperationException("The end timestamp of an " +
                        "event-time window cannot become earlier than the current watermark " +
                        "by merging. Current watermark: " + internalTimerService.currentWatermark() +
                        " window: " + mergeResult);
               } else if (!windowAssigner.isEventTime() && mergeResult.maxTimestamp() <= internalTimerService.currentProcessingTime()) {
                  throw new UnsupportedOperationException("The end timestamp of a " +
                        "processing-time window cannot become earlier than the current processing time " +
                        "by merging. Current processing time: " + internalTimerService.currentProcessingTime() +
                        " window: " + mergeResult);
               }

               triggerContext.key = key;
               triggerContext.window = mergeResult;

               triggerContext.onMerge(mergedWindows);

               for (W m: mergedWindows) {
                  triggerContext.window = m;
                  triggerContext.clear();
                  deleteCleanupTimer(m);
               }

               // merge the merged state windows into the newly resulting state window
               windowMergingState.mergeNamespaces(stateWindowResult, mergedStateWindows);
            }
         });

         // drop if the window is already late
         if (isWindowLate(actualWindow)) {
            mergingWindows.retireWindow(actualWindow);
            continue;
         }
         isSkippedElement = false;

         W stateWindow = mergingWindows.getStateWindow(actualWindow);
         if (stateWindow == null) {
            throw new IllegalStateException("Window " + window + " is not in in-flight window set.");
         }

         windowState.setCurrentNamespace(stateWindow);
         windowState.add(element.getValue());

         triggerContext.key = key;
         triggerContext.window = actualWindow;

         TriggerResult triggerResult = triggerContext.onElement(element);

         if (triggerResult.isFire()) {
            ACC contents = windowState.get();
            if (contents == null) {
               continue;
            }
            emitWindowContents(actualWindow, contents);
         }

         if (triggerResult.isPurge()) {
            windowState.clear();
         }
         registerCleanupTimer(actualWindow);
      }

      // need to make sure to update the merging state in state
      mergingWindows.persist();
   } else {
       //if else走else，因为是滚动窗口，所以数据只在一个窗口
      for (W window: elementWindows) {

         // drop if the window is already late
         if (isWindowLate(window)) {
            continue;
         }
         isSkippedElement = false;

         windowState.setCurrentNamespace(window);
         //将数据添加到当前的状态数据中，其中会调用reduceFunction将该记录和窗口的状态数据合并
         windowState.add(element.getValue());

         triggerContext.key = key;
         triggerContext.window = window;
        //对每个记录调用onElement，做的工作包括部署定时任务，用于触发窗口计算
        //为每个key添加一个timer到processTimeTimersQueue中
         TriggerResult triggerResult = triggerContext.onElement(element);
        //如果返回FIRE，则代表触发窗口计算，不过在本例中，这里返回的永远是CONTINUE
         if (triggerResult.isFire()) {
            ACC contents = windowState.get();
            if (contents == null) {
               continue;
            }
            emitWindowContents(window, contents);
         }

         if (triggerResult.isPurge()) {
            windowState.clear();
         }
         registerCleanupTimer(window);
      }
   }

   // side output input event if
   // element not handled by any window
   // late arriving tag has been set
   // windowAssigner is event time and current timestamp + allowed lateness no less than element timestamp
   if (isSkippedElement && isElementLate(element)) {
      if (lateDataOutputTag != null){
         sideOutput(element);
      } else {
         this.numLateRecordsDropped.inc();
      }
   }
}
```
在注释中解释了关键代码的作用，windowOperator的processElement方法做的工作主要包括

1、分接收到的数据分配窗口

2、将数据通过reduceFunction进行状态数据的合并，这里数据合并完就可以丢弃了，只保留状态数据即可

3、对每条记录调用triggerContext.onElement(element)，这主要用于部署定时任务用于触发窗口计算，对每个key添加一个timer到processTimeTimersQueue中，在窗口触发计算时分别出队这些timer，依次获取每个key的状态数据

4、如果触发窗口计算，将window中的所有状态数据发送出去

## 数据聚合
windowState.add(element.getValue())

在WordCount中，windowState的类型是HeapReducingState, add方法调用了StateTable.transform()
```
protected final StateTable<K, N, SV> stateTable;

@Override
public void add(V value) throws IOException {

   if (value == null) {
      clear();
      return;
   }

   try {
      stateTable.transform(currentNamespace, value, reduceTransformation);
   } catch (Exception e) {
      throw new IOException("Exception while applying ReduceFunction in reducing state", e);
   }
}
```
在本例中StateTable的类型为CopyOnWriteStateTable，transform的具体操作便是用reduceFunction将key的状态数据和当前记录进行合并。
```
@Override
public <T> void transform(N namespace, T value, StateTransformationFunction<S, T> transformation) throws Exception {
   transform(keyContext.getCurrentKey(), namespace, value, transformation);
}

<T> void transform(
      K key,
      N namespace,
      T value,
      StateTransformationFunction<S, T> transformation) throws Exception {

   final StateTableEntry<K, N, S> entry = putEntry(key, namespace);

   // copy-on-write check for state
   entry.state = transformation.apply(
         (entry.stateVersion < highestRequiredSnapshotVersion) ?
               getStateSerializer().copy(entry.state) :
               entry.state,
         value);
   entry.stateVersion = stateTableVersion;
}

@Override
public V apply(V previousState, V value) throws Exception {
   return previousState != null ? reduceFunction.reduce(previousState, value) : value;
}
```
顺便提一下，状态数据在内存中存储形式是类似于HashMap的一张表，横向是数组，纵向是类似于链表的数据结构StateTableEntry。

## 窗口触发
windowOperator中会对每个记录调用triggerContext.onElement(element)
```
TriggerResult triggerResult = triggerContext.onElement(element);

public TriggerResult onElement(StreamRecord<IN> element) throws Exception {
   return trigger.onElement(element.getValue(), element.getTimestamp(), window, this);
}
```
在WordCount中，由于系统时间是ProcessTime，所以触发器trigger的类型是ProcessingTimeTrigger，会调用ctx.registerProcessingTimeTimer(window.maxTimestamp())来为每个key添加timer，并部署一个定时线程用于触发窗口计算
```
@PublicEvolving
public class ProcessingTimeTrigger extends Trigger<Object, TimeWindow> {
   private static final long serialVersionUID = 1L;

   private ProcessingTimeTrigger() {}

   @Override
   public TriggerResult onElement(Object element, long timestamp, TimeWindow window, TriggerContext ctx) {
      ctx.registerProcessingTimeTimer(window.maxTimestamp());
      return TriggerResult.CONTINUE;
   }

@Override
public void registerProcessingTimeTimer(long time) {
   internalTimerService.registerProcessingTimeTimer(window, time);
}

@Override
public void registerProcessingTimeTimer(N namespace, long time) {
   InternalTimer<K, N> oldHead = processingTimeTimersQueue.peek();
   //每个key（除了重复的key）都会构造一个TimerHeapInternalTimer添加到processingTimeTimersQueue
   //每个窗口的第一个元素会调用processingTimeService.registerTimer部署一个定时线程
   if (processingTimeTimersQueue.add(new TimerHeapInternalTimer<>(time, (K) keyContext.getCurrentKey(), namespace))) {
      long nextTriggerTime = oldHead != null ? oldHead.getTimestamp() : Long.MAX_VALUE;
      // check if we need to re-schedule our timer to earlier
      if (time < nextTriggerTime) {
         if (nextTimer != null) {
            nextTimer.cancel(false);
         }
         nextTimer = processingTimeService.registerTimer(time, this);
         
      }
   }
}
```
部署一个TriggerTask，用于定时触发窗口计算
```
@Override
public ScheduledFuture<?> registerTimer(long timestamp, ProcessingTimeCallback target) {

   // delay the firing of the timer by 1 ms to align the semantics with watermark. A watermark
   // T says we won't see elements in the future with a timestamp smaller or equal to T.
   // With processing time, we therefore need to delay firing the timer by one ms.
   long delay = Math.max(timestamp - getCurrentProcessingTime(), 0) + 1;

   // we directly try to register the timer and only react to the status on exception
   // that way we save unnecessary volatile accesses for each timer
   try {
      return timerService.schedule(
            new TriggerTask(status, task, checkpointLock, target, timestamp), delay, TimeUnit.MILLISECONDS);
   }
   catch (RejectedExecutionException e) {
      final int status = this.status.get();
      if (status == STATUS_QUIESCED) {
         return new NeverCompleteFuture(delay);
      }
      else if (status == STATUS_SHUTDOWN) {
         throw new IllegalStateException("Timer service is shut down");
      }
      else {
         // something else happened, so propagate the exception
         throw e;
      }
   }
}
```
TriggerTask的run()方法如下：调用ProcessingTimeCallback.onProcessingTime(timestamp)

```
@Override
public void run() {
   synchronized (lock) {
      try {
         if (serviceStatus.get() == STATUS_ALIVE) {
             //onProcessingTime用于触发窗口计算
            target.onProcessingTime(timestamp);
         }
      } catch (Throwable t) {
         TimerException asyncException = new TimerException(t);
         exceptionHandler.handleAsyncException("Caught exception while processing timer.", asyncException);
      }
   }
}

@Override
public void onProcessingTime(long time) throws Exception {
   // null out the timer in case the Triggerable calls registerProcessingTimeTimer()
   // inside the callback.
   nextTimer = null;

   InternalTimer<K, N> timer;
    //出队processingTimeTimersQueue中的timer，依次处理每个key的状态数据
   while ((timer = processingTimeTimersQueue.peek()) != null && timer.getTimestamp() <= time) {
      processingTimeTimersQueue.poll();
      keyContext.setCurrentKey(timer.getKey());
      triggerTarget.onProcessingTime(timer);
   }

   if (timer != null && nextTimer == null) {
      nextTimer = processingTimeService.registerTimer(timer.getTimestamp(), this);
   }
}

```
WindowOperator.onProcessingTime()源码,最终会调用到ProcessingTimeTrigger.onProcessTime返回FIRE来触发窗口计算。

```
@Override
public void onProcessingTime(InternalTimer<K, W> timer) throws Exception {
   triggerContext.key = timer.getKey();
   triggerContext.window = timer.getNamespace();

   MergingWindowSet<W> mergingWindows;

   if (windowAssigner instanceof MergingWindowAssigner) {
      mergingWindows = getMergingWindowSet();
      W stateWindow = mergingWindows.getStateWindow(triggerContext.window);
      if (stateWindow == null) {
         // Timer firing for non-existent window, this can only happen if a
         // trigger did not clean up timers. We have already cleared the merging
         // window and therefore the Trigger state, however, so nothing to do.
         return;
      } else {
         windowState.setCurrentNamespace(stateWindow);
      }
   } else {
      windowState.setCurrentNamespace(triggerContext.window);
      mergingWindows = null;
   }

   TriggerResult triggerResult = triggerContext.onProcessingTime(timer.getTimestamp());
    //获取当前key的state状态数据，然后发送出去
   if (triggerResult.isFire()) {
      ACC contents = windowState.get();
      if (contents != null) {
         emitWindowContents(triggerContext.window, contents);
      }
   }

   if (triggerResult.isPurge()) {
      windowState.clear();
   }

   if (!windowAssigner.isEventTime() && isCleanupTime(triggerContext.window, timer.getTimestamp())) {
      clearAllState(triggerContext.window, windowState, mergingWindows);
   }

   if (mergingWindows != null) {
      // need to make sure to update the merging state in state
      mergingWindows.persist();
   }
}

//Context类
public TriggerResult onProcessingTime(long time) throws Exception {
   return trigger.onProcessingTime(time, window, this);
}

//ProcessingTimeTrigger类
@Override
public TriggerResult onProcessingTime(long time, TimeWindow window, TriggerContext ctx) {
   return TriggerResult.FIRE;
}
```

## 数据发送
emitWindowContents,获取当前key的state状态数据，然后发送出去。在WordCount最终每个key的状态数据都会调用PassThroughWindowFunction.apply进行收集数据并发送，其他的应用会根据用户传入的windowFunction进行计算。发送过程于上述StreamMap将数据发送到Window中类似，不过这里的partitioner是RebalancePartitioner。
```
private void emitWindowContents(W window, ACC contents) throws Exception {
   timestampedCollector.setAbsoluteTimestamp(window.maxTimestamp());
   processContext.window = window;
   userFunction.process(triggerContext.key, window, processContext, contents, timestampedCollector);
}

//InternalSingleValueWindowFunction类
@Override
public void process(KEY key, W window, InternalWindowContext context, IN input, Collector<OUT> out) throws Exception {
   wrappedFunction.apply(key, window, Collections.singletonList(input), out);
}

//PassThroughWindowFunction类
@Override
public void apply(K k, W window, Iterable<T> input, Collector<T> out) throws Exception {
   for (T in: input) {
      out.collect(in);
   }
}
```

## 数据输出
print由于并行度和window操作不一致，所以无法和window操作进行chain起来，它也是一个单独的Task，下游也会执行OneInputStreamTask。上述window发送的数据将会到达sink，sink的operator是StreamSink，会调用processElement方法
```
@Override
public void processElement(StreamRecord<IN> element) throws Exception {
   sinkContext.element = element;
   userFunction.invoke(element.getValue(), sinkContext);
}
```

在WordCount中userFunction是PrintSinkFunction，invoke方法的实现就是将数据打印到控制台
```
@PublicEvolving
public class PrintSinkFunction<IN> extends RichSinkFunction<IN> {

   private static final long serialVersionUID = 1L;

   private final PrintSinkOutputWriter<IN> writer;

   /**
    * Instantiates a print sink function that prints to standard out.
    */
   public PrintSinkFunction() {
      writer = new PrintSinkOutputWriter<>(false);
   }

@Override
public void invoke(IN record) {
   writer.write(record);
}

```

至此，WordCount的处理逻辑分析完毕。



## 总结：

WordCount任务执行原理

- Stream会启动任务执行OneInputStreamTask，OneInputStreamTask中包含一个流处理器StreamInputProcessor，核心方法在processInput()，StreamInputProcessor中持有StreamTask的headOperator，processInput()方法便会调用headOperator.setKeyContextElement1()对stateBackEnd设置当前的key，不过这个方法一般只在窗口操作对数据进行聚合的时候才有效。

- 接着，processInput()会调用headOperator.processElement进行Record数据的核心处理，例如应用flatMapFunction，数据处理完后数据收集器collector会将结果数据直接push到operator chain的下一个operator进行处理，中间不会经过序列化和网络传输。下一个operator依旧会调用operator.setKeyContextElement1()和operator.processElement()对Record进行处理

- 在map这个operator处理完毕后，这时operator chain已经没有下一个operator了，output会将结果发送到下游的task。具体数据应该发往下游的哪个task需要根据keyBy算子过程中生成的KeyGroupStreamPartitioner.selectChannel()决定。

- 下游的Task依旧是OneInputStreamTask，StreamInputProcessor中的headOperator是windowOperator，windowOperator拥有keySelector1，会进行setKeyContextElement的操作，给当前的keyedStateBackEnd设置当前key(currentKey)为record记录的key。接下来会进行windowOperator.processElement()。processElement()首先会根据windowAssigner对数据进行窗口分配，即数据落在哪个窗口。windowOperator中包含一个windowState，类型为HeapReducingState，这个就是reduce函数要操作的窗口状态数据，会实时的将当前记录和key对应的状态数据进行数据聚合。状态数据在内存中的储存形式类似于HashMap的一种数据结构叫StateTable，横向为数组，纵向是链表。StateBackEnd的类型为HeapKeyedStateBackEnd。

- processElement()在将数据添加到状态数据中之后，会对每个记录调用trigger.onElement()，在系统时间为ProcessTime时这个trigger就是ProcessingTimeTrigger。这个方法的作用就是在窗口到达第一个元素时为窗口部署一个定时线程，用于到达窗口结束时间时定时触发窗口操作，同时为每个key构造一个timer添加到processTimeTimerQueue中(重复的key不再添加)，这用于在窗口计算中依次出队每个key，依次发送每个key的状态数据到windowFunction。定时任务到点会调用trigger.onProcessTime，返回FIRE，意味着窗口触发计算。

- 窗口触发计算时，会从processTimeTimerQueue出队每个key的timer，然后取出每个key的状态数据，调用emitWindowContents()进行发送给windowFunction，在默认情况下，如果用户没有指定windowFunction，数据会调用PassThoughWindowFunction.apply()方法，该方法的实现就是将每个key的状态数据发送到下游，partitioner是RebalancePartitioner

- 下游的Task依然是OneInputStreamTask，StreamInputProcessor中的operator是StreamSink，StreamSink的processElement方法就是对接收到的数据调用PrintFunction的invoke()方法，将数据打印到控制台。

到此，WordCount的执行逻辑就完了



