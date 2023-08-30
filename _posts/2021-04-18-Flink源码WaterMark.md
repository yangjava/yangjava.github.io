---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink源码WaterMark
flink的watermark（水位线）是针对EventTime的计算，它用来标识应该何时触发窗口计算，随着数据不断地进入flink程序，事件的EventTime也将慢慢增大，随之watermark水位线也将逐步上涨，当watermark上涨超过窗口的结束时间，将开始触发窗口计算。

本文主要分析一下watermark在flink内部的工作原理，包括watermark的产生、传播、触发计算的过程。

首先来看看WaterMark的结构,可以看到Watermark的结构非常简单，只有一个时间戳timestamp
```
public final class Watermark extends StreamElement {

   public static final Watermark MAX_WATERMARK = new Watermark(Long.MAX_VALUE);

   private final long timestamp;

   public Watermark(long timestamp) {
      this.timestamp = timestamp;
   }

   public long getTimestamp() {
      return timestamp;
   }

   @Override
   public boolean equals(Object o) {
      return this == o ||
            o != null && o.getClass() == Watermark.class && ((Watermark) o).timestamp == this.timestamp;
   }

   @Override
   public int hashCode() {
      return (int) (timestamp ^ (timestamp >>> 32));
   }

   @Override
   public String toString() {
      return "Watermark @ " + timestamp;
   }
}
```
## watermark样例

再来看一个简单的flink watermark计算例子
```
object WaterMarkStream {

  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    //设置流的执行时间为EventTime，watermark针对于EventTime，而不是ProcessTime，系统默认是ProcessTime
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    env.setParallelism(4)
    env.getConfig.setAutoWatermarkInterval(10000)

    val stream = env.socketTextStream("10.18.34.198", 9999).filter(!StringUtils.isBlank(_))

    stream
      .assignTimestampsAndWatermarks(new BoundedOutOfOrderness)
      .map((_, 1))
      .keyBy(0)
      .timeWindow(Time.seconds(60))
      .sum(1)
      .print()

    env.execute("WaterMark Test Stream")
  }

}

//基于事件时间界限的watermark，例如设置了窗口滚动时长为60s，最大延迟下界为10s，
// 如果来了一条时间为00:10:10的数据，此时watermark的值会变成00:10:00,
// 那么就会触发 [00:09:00, 00:10:00) 这个窗口进行数据计算，
// 计算完之后窗口会关闭，之后来的这个窗口时间范围内的事件默认情况下会被丢弃
class BoundedOutOfOrderness extends AssignerWithPeriodicWatermarks[String] {

  var maxEventTimestamp: Long = 0
  val maxOutOfOrderness = 10000L

  override def getCurrentWatermark: Watermark = {
    println(s"${System.currentTimeMillis} current max event time: $maxEventTimestamp, watermark: ${maxEventTimestamp - maxOutOfOrderness}")
    new Watermark(maxEventTimestamp - maxOutOfOrderness)
  }

  override def extractTimestamp(element: String, previousElementTimestamp: Long): Long = {

    val timestamp = element.split("\\s+")(0).toLong
    maxEventTimestamp = Math.max(timestamp, maxEventTimestamp)
    timestamp
  }

}
```
上述代码很简单，从socket接收数据，从数据中提取时间戳timestamp，watermark assigner类型是AssignerWithPeriodicWatermarks，即定期产生watermark，程序中设置的间隔是10s。getCurrentWatermark()方法产生的watermark是接收到所有数据的最大时间戳-maxOutOfOrderness 。窗口时间是60s。

首先来看assignTimestampsAndWatermarks()方法：
```
public SingleOutputStreamOperator<T> assignTimestampsAndWatermarks(
      AssignerWithPeriodicWatermarks<T> timestampAndWatermarkAssigner) {

   // match parallelism to input, otherwise dop=1 sources could lead to some strange
   // behaviour: the watermark will creep along very slowly because the elements
   // from the source go to each extraction operator round robin.
   final int inputParallelism = getTransformation().getParallelism();
   final AssignerWithPeriodicWatermarks<T> cleanedAssigner = clean(timestampAndWatermarkAssigner);

   TimestampsAndPeriodicWatermarksOperator<T> operator =
         new TimestampsAndPeriodicWatermarksOperator<>(cleanedAssigner);

   return transform("Timestamps/Watermarks", getTransformation().getOutputType(), operator)
         .setParallelism(inputParallelism);
}
```
结合之前分析的WordCount源码分析，可以看到assignTimestampsAndWatermarks()会产生一个新的DataStream，同时会创建一个TimestampsAndPeriodicWatermarksOperator，看来这个方法和普通的map、filter算子是类似的。

下面附上该代码的执行计划图，可以看到TimestampsAndPeriodicWatermarksOperator就是一个普通的operator，可以和filter/map operator进行chain。

## 打开WatermarksOperator
在Task启动时，会调用operator.open()方法，看看TimestampsAndPeriodicWatermarksOperator的open方法干了啥
```
public void open() throws Exception {
   super.open();

   currentWatermark = Long.MIN_VALUE;
   watermarkInterval = getExecutionConfig().getAutoWatermarkInterval();

   if (watermarkInterval > 0) {
      long now = getProcessingTimeService().getCurrentProcessingTime();
      getProcessingTimeService().registerTimer(now + watermarkInterval, this);
   }
}

//SystemProcessingTimeService类
public ScheduledFuture<?> registerTimer(long timestamp, ProcessingTimeCallback target) {

   // delay the firing of the timer by 1 ms to align the semantics with watermark. A watermark
   // T says we won't see elements in the future with a timestamp smaller or equal to T.
   // With processing time, we therefore need to delay firing the timer by one ms.
   long delay = Math.max(timestamp - getCurrentProcessingTime(), 0) + 1;

   // we directly try to register the timer and only react to the status on exception
   // that way we save unnecessary volatile accesses for each timer
   try {
       //部署一个定时任务
      return timerService.schedule(
            new TriggerTask(status, task, checkpointLock, target, timestamp), delay, TimeUnit.MILLISECONDS);
   }
    ...
}

private static final class TriggerTask implements Runnable {

   private final AtomicInteger serviceStatus;
   private final Object lock;
   private final ProcessingTimeCallback target;
   private final long timestamp;
   private final AsyncExceptionHandler exceptionHandler;

   private TriggerTask(
         final AtomicInteger serviceStatus,
         final AsyncExceptionHandler exceptionHandler,
         final Object lock,
         final ProcessingTimeCallback target,
         final long timestamp) {

      this.serviceStatus = Preconditions.checkNotNull(serviceStatus);
      this.exceptionHandler = Preconditions.checkNotNull(exceptionHandler);
      this.lock = Preconditions.checkNotNull(lock);
      this.target = Preconditions.checkNotNull(target);
      this.timestamp = timestamp;
   }

   @Override
   public void run() {
      synchronized (lock) {
         try {
            if (serviceStatus.get() == STATUS_ALIVE) {
                //定时任务的实现就是执行target.onProcessingTime，
                //这个target就是TimestampsAndPeriodicWatermarksOperator自身
               target.onProcessingTime(timestamp);
            }
         } catch (Throwable t) {
            TimerException asyncException = new TimerException(t);
            exceptionHandler.handleAsyncException("Caught exception while processing timer.", asyncException);
         }
      }
   }
}
```
可以看到open()方法就是注册了一个定时任务，这个定时任务触发的间隔时间就是在程序里设置的

setAutoWatermarkInterval(interval)这个值，默认是200ms，我们代码里就是10s。到达时间之后将会触发target.onProcessingTime(timestamp)，也就是TimestampsAndPeriodicWatermarksOperator.onProcessingTime()方法。

下面就来看看TimestampsAndPeriodicWatermarksOperator.onProcessingTime()方法

```
public void onProcessingTime(long timestamp) throws Exception {
   // register next timer
   Watermark newWatermark = userFunction.getCurrentWatermark();
   if (newWatermark != null && newWatermark.getTimestamp() > currentWatermark) {
      currentWatermark = newWatermark.getTimestamp();
      // emit watermark
      output.emitWatermark(newWatermark);
   }

   long now = getProcessingTimeService().getCurrentProcessingTime();
   getProcessingTimeService().registerTimer(now + watermarkInterval, this);
}
```
实现比较简单，大致为

- 通过用户提供的watermark assigner函数获取新的watermark

- 如果新的watermark比老的watermark时间戳要大，就把新的watermark发送出去

- 重新注册一个定时任务，跟open()方法中一致，等待下一个间隔再次调用onProcessingTime()方法

## 发送watermark
重要的是上述的步骤2，我们要看看watermark是如何发送的。
```
//CountingOutput类
public void emitWatermark(Watermark mark) {
   output.emitWatermark(mark);
}

//ChainingOutput类
public void emitWatermark(Watermark mark) {
   try {
      watermarkGauge.setCurrentWatermark(mark.getTimestamp());
      if (streamStatusProvider.getStreamStatus().isActive()) {
         operator.processWatermark(mark);
      }
   }
   catch (Exception e) {
      throw new ExceptionInChainedOperatorException(e);
   }
}

//AbstractStreamOperator类
public void processWatermark(Watermark mark) throws Exception {
    //mapOperator中timeServiceManager == null
   if (timeServiceManager != null) {
      timeServiceManager.advanceWatermark(mark);
   }
   output.emitWatermark(mark);
}

//CountingOutput类
public void emitWatermark(Watermark mark) {
   output.emitWatermark(mark);
}

//RecordWriterOutput类
public void emitWatermark(Watermark mark) {
   watermarkGauge.setCurrentWatermark(mark.getTimestamp());
   serializationDelegate.setInstance(mark);

   if (streamStatusProvider.getStreamStatus().isActive()) {
      try {
         recordWriter.broadcastEmit(serializationDelegate);
      } catch (Exception e) {
         throw new RuntimeException(e.getMessage(), e);
      }
   }
}

//RecordWriter类
public void broadcastEmit(T record) throws IOException, InterruptedException {
   checkErroneous();
   serializer.serializeRecord(record);

   boolean pruneAfterCopying = false;
   //这里是将watermark广播出去，每一个下游通道都要发送，也就是说会发送到下游的每一个task
   for (int channel : broadcastChannels) {
      if (copyFromSerializerToTargetChannel(channel)) {
         pruneAfterCopying = true;
      }
   }

   // Make sure we don't hold onto the large intermediate serialization buffer for too long
   if (pruneAfterCopying) {
      serializer.prune();
   }
}
```
上述代码说明了watermark在本例中的代码调用过程。

在本例中，watermarkOperator和map是chain在一起的，watermarkOperator会将watermark直接传递给map，mapOperator会调用processWatermark()方法来处理watermark，在mapOperator中timeServiceManager == null，所以mapOperator对watermark不做任务处理，而是直接将其送达出去。

mapOperator持有的Output是RecordWriterOutput，RecordWriterOutput它会通过RecordWriter将watermark广播到下游的所有通道，即发送给下游的所有task。也就是说，上游的一个task在更新了watermark之后，会将watermark广播给下游的所有task。

## 下游Task接收并处理watermark
在本例中，下游task是一个OneInputStreamTask，通过数据处理器StreamInputProcessor.processInput()来处理接收到的数据信息
```
public boolean processInput() throws Exception {
   ...

   while (true) {
      if (currentRecordDeserializer != null) {
         DeserializationResult result = currentRecordDeserializer.getNextRecord(deserializationDelegate);

         if (result.isBufferConsumed()) {
            currentRecordDeserializer.getCurrentBuffer().recycleBuffer();
            currentRecordDeserializer = null;
         }

         if (result.isFullRecord()) {
            StreamElement recordOrMark = deserializationDelegate.getInstance();

            if (recordOrMark.isWatermark()) {
               // handle watermark
               //处理接收到的watermark信息
               statusWatermarkValve.inputWatermark(recordOrMark.asWatermark(), currentChannel);
               continue;
            } 
            ...
         }
      }
    ...
}
```
StreamInputProcessor会调用statusWatermarkValve.inputWatermark()来处理接收到watermark信息。看下代码:
```
public void inputWatermark(Watermark watermark, int channelIndex) {
   // ignore the input watermark if its input channel, or all input channels are idle (i.e. overall the valve is idle).
   if (lastOutputStreamStatus.isActive() && channelStatuses[channelIndex].streamStatus.isActive()) {
      long watermarkMillis = watermark.getTimestamp();

      // if the input watermark's value is less than the last received watermark for its input channel, ignore it also.
      //channelStatuses是当前task对每个inputChannel（也就是每个上游task）的状态信息记录，
      //当新的watermark值大于inputChannel的watermark，就会进行调整
      if (watermarkMillis > channelStatuses[channelIndex].watermark) {
         channelStatuses[channelIndex].watermark = watermarkMillis;

         // previously unaligned input channels are now aligned if its watermark has caught up
         if (!channelStatuses[channelIndex].isWatermarkAligned && watermarkMillis >= lastOutputWatermark) {
            channelStatuses[channelIndex].isWatermarkAligned = true;
         }

         // now, attempt to find a new min watermark across all aligned channels
         //从各个inputChannel的watermark里找到最小的的watermark进行处理
         findAndOutputNewMinWatermarkAcrossAlignedChannels();
      }
   }
}

private void findAndOutputNewMinWatermarkAcrossAlignedChannels() {
   long newMinWatermark = Long.MAX_VALUE;
   boolean hasAlignedChannels = false;

   // determine new overall watermark by considering only watermark-aligned channels across all channels
   //从所有的inputChannels的watermark里找到最小的的watermark
   for (InputChannelStatus channelStatus : channelStatuses) {
      if (channelStatus.isWatermarkAligned) {
         hasAlignedChannels = true;
         newMinWatermark = Math.min(channelStatus.watermark, newMinWatermark);
      }
   }

   // we acknowledge and output the new overall watermark if it really is aggregated
   // from some remaining aligned channel, and is also larger than the last output watermark
   //如果最小的watermark大于之前发送过的watermark，则调用outputHandler进行处理
   if (hasAlignedChannels && newMinWatermark > lastOutputWatermark) {
      lastOutputWatermark = newMinWatermark;
      outputHandler.handleWatermark(new Watermark(lastOutputWatermark));
   }
}
```
上述代码的大致实现是，当上游一个task将watermark广播到下游的所有channel（可以理解成下游所有task）之后，下游的task会更新对上游inputChannel记录状态信息中的watermark值，下游每个task都记录这上游所有task的状态值。然后下游task再从所有上游inputChannel（即上游所有task）中选出一个最小值的watermark，如果这个watermark大于最近已经发送的watermark，那么就调用outputHandler对新watermark进行处理。一般情况下，这个outputHandler就是ForwardingValveOutputHandler。

## windowOperator触发窗口计算
ForwardingValveOutputHandler.handleWatermark()方法如下：
```
public void handleWatermark(Watermark watermark) {
   try {
      synchronized (lock) {
         watermarkGauge.setCurrentWatermark(watermark.getTimestamp());
         operator.processWatermark(watermark);
      }
   } catch (Exception e) {
      throw new RuntimeException("Exception occurred while processing valve output watermark: ", e);
   }
}
```
可以看到ForwardingValveOutputHandler.handleWatermark()也是将watermark交给operator去处理。在本例中，这个operator就到了windowOperator。

```
//AbstractStreamOperator类
public void processWatermark(Watermark mark) throws Exception {
    //windowOperator的timeServiceManager ！= null
   if (timeServiceManager != null) {
      timeServiceManager.advanceWatermark(mark);
   }
   output.emitWatermark(mark);
}

//InternalTimeServiceManager类
public void advanceWatermark(Watermark watermark) throws Exception {
   for (InternalTimerServiceImpl<?, ?> service : timerServices.values()) {
      service.advanceWatermark(watermark.getTimestamp());
   }
}

//InternalTimerServiceImpl类
public void advanceWatermark(long time) throws Exception {
   currentWatermark = time;

   InternalTimer<K, N> timer;
    //如果watermark大于timer的时间，从eventTimeTimersQueue中出队timer，
    //这个timer是在窗口对数据进行处理时对每个key创建一个timer添加到eventTimeTimersQueue中的，
    //出队也意味着触发了窗口计算
   while ((timer = eventTimeTimersQueue.peek()) != null && timer.getTimestamp() <= time) {
      eventTimeTimersQueue.poll();
      keyContext.setCurrentKey(timer.getKey());
      //triggerTarget就是windowOperator
      triggerTarget.onEventTime(timer);
   }
}

//WindowOperator
public void onEventTime(InternalTimer<K, W> timer) throws Exception {
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
      //走这里
      windowState.setCurrentNamespace(triggerContext.window);
      mergingWindows = null;
   }

   TriggerResult triggerResult = triggerContext.onEventTime(timer.getTimestamp());
    //触发计算时返回FIRE
   if (triggerResult.isFire()) {
      ACC contents = windowState.get();
      if (contents != null) {
          //发送当前key的状态数据
         emitWindowContents(triggerContext.window, contents);
      }
   }

   if (triggerResult.isPurge()) {
      windowState.clear();
   }
    //如果到了CleanupTime（=window.maxTimestamp() + allowedLateness），清理当前key的状态数据
   if (windowAssigner.isEventTime() && isCleanupTime(triggerContext.window, timer.getTimestamp())) {
      clearAllState(triggerContext.window, windowState, mergingWindows);
   }

   if (mergingWindows != null) {
      // need to make sure to update the merging state in state
      mergingWindows.persist();
   }
}
```
从上述代码可以看到，windowOperator的processWatermark()方法中实现大致是：

- 调用timeServiceManager.advanceWatermark(mark)，windowOperator中timeServiceManager != null，是InternalTimeServiceManager实例。最终会调用InternalTimerServiceImpl.advanceWatermark()方法，如果watermark大于eventTimeTimersQueue中头元素timer的时间，那么从eventTimeTimersQueue出队timer，也意味着触发了窗口计算。eventTimeTimersQueue是一个优先队列，头元素timer是timestamp值最小的。

- 窗口触发计算的形式和ProcessTime类似，都是一个key一个key的获取状态数据，然后依次发送出去，如果到了窗口状态清理时间（=window.maxTimestamp() + allowedLateness），还会清除当前key的状态数据

- 调用output.emitWatermark(mark)将watermark继续发送给下游

## EventTime下的窗口计算
上述代码中我们看到了窗口触发计算的情况，既然有窗口触发操作，那么就应该看一下EventTime下的窗口计算操作，实现在WindowOperator.processElement()
```
public void processElement(StreamRecord<IN> element) throws Exception {
    //首先为数据分配窗口
   final Collection<W> elementWindows = windowAssigner.assignWindows(
      element.getValue(), element.getTimestamp(), windowAssignerContext);

   //if element is handled by none of assigned elementWindows
   boolean isSkippedElement = true;

   final K key = this.<K>getKeyedStateBackend().getCurrentKey();

   if (windowAssigner instanceof MergingWindowAssigner) {
     ...
   } else {
      for (W window: elementWindows) {

         // drop if the window is already late
         //如果窗口已经延迟了，即watermark >= window.maxTimestamp() + allowedLateness
         //allowedLateness默认值是0
         if (isWindowLate(window)) {
            continue;
         }
         isSkippedElement = false;
        //将输入数据添加到windowState，数据核心处理的地方
         windowState.setCurrentNamespace(window);
         windowState.add(element.getValue());

         triggerContext.key = key;
         triggerContext.window = window;
        //为每个数据按key添加一个timer，key重复的不再添加。可能会返回FIRE
         TriggerResult triggerResult = triggerContext.onElement(element);

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
         //给每个key添加一个清理状态数据的timer，当watermark达到cleanup时间时，会触发清理状态数据
         registerCleanupTimer(window);
      }
   }

   // side output input event if
   // element not handled by any window
   // late arriving tag has been set
   // windowAssigner is event time and current timestamp + allowed lateness no less than element timestamp
   //给迟到数据添加侧输出
   if (isSkippedElement && isElementLate(element)) {
      if (lateDataOutputTag != null){
         sideOutput(element);
      } else {
         this.numLateRecordsDropped.inc();
      }
   }
}
```
processElement()的实现大致如下：

1、为接收到的数据分配窗口。

2、遍历每个窗口，如果窗口延迟了，即watermark >= window.maxTimestamp() + allowedLateness，则跳过该窗口，该数据不做处理，这也就是为什么迟到数据默认情况下会被丢弃，allowedLateness默认值是0。如果窗口没有延迟，则将数据添加到windowStat中

3、状态数据添加完数据，接着就为每个输入数据按照key添加一个timer到队列中，这部分实现在triggerContext.onElement(element)

4、为每个数据按照key（其实就是给每个key）添加一个清理状态数据的timer，用于到时清理内存中的状态数据

上述步骤中的2中把数据添加到windowState的操作已经在分析WordCount任务运行时候分析过了，这里就不再重复细说了。来看看是如何定义窗口延迟的

```
protected boolean isWindowLate(W window) {
   return (windowAssigner.isEventTime() && (cleanupTime(window) <= internalTimerService.currentWatermark()));
}

private long cleanupTime(W window) {
   if (windowAssigner.isEventTime()) {
      long cleanupTime = window.maxTimestamp() + allowedLateness;
      return cleanupTime >= window.maxTimestamp() ? cleanupTime : Long.MAX_VALUE;
   } else {
      return window.maxTimestamp();
   }
}
```
即watermark >= window.maxTimestamp() + allowedLateness时被认定为窗口延迟，这时会跳过数据处理，便不再添加到windowState中了。

下面主要来看看上述的步骤3，triggerContext.onElement(element)的实现：
```
public TriggerResult onElement(StreamRecord<IN> element) throws Exception {
   return trigger.onElement(element.getValue(), element.getTimestamp(), window, this);
}

//EventTimeTrigger类
public TriggerResult onElement(Object element, long timestamp, TimeWindow window, TriggerContext ctx) throws Exception {
   if (window.maxTimestamp() <= ctx.getCurrentWatermark()) {
      // if the watermark is already past the window fire immediately
      return TriggerResult.FIRE;
   } else {
      ctx.registerEventTimeTimer(window.maxTimestamp());
      return TriggerResult.CONTINUE;
   }
}
```
可以看到，在调用triggerContext.onElement(element)时，是有可能返回TriggerResult.FIRE的，条件是window.maxTimestamp() <= ctx.getCurrentWatermark()；主要说下这个，个人在这上面困惑了很长时间，个人理解，正常情况下，在一个有序的数据流里面，数据的eventTime > watermark，而 window.maxTimestamp >= eventTime，这样来说，window.maxTimestamp > watermark就是一定的了，这种情况下几乎是不会触发FIRE的。FIRE的触发都是按照之前描述的情况，即在InternalTimerServiceImpl.advanceWatermark()中触发。

不过，上述是在数据整体有序的情况下，如果数据无序，就很难保证当前处理数据的eventTime > watermark，因此也不能保证window.maxTimestamp > watermark，如果此时又开启了窗口延迟功能，如果此时又正好处于 window.maxTimestamp =< watermark <= window.maxTimestamp+allowedLateness的情况，那么就会返回FIRE，该条数据就会和状态数据聚合然后触发计算。

下面也是重点ctx.registerEventTimeTimer(window.maxTimestamp())，给数据按照key添加timer。
```
public void registerEventTimeTimer(long time) {
   internalTimerService.registerEventTimeTimer(window, time);
}

//InternalTimerServiceImpl类
public void registerEventTimeTimer(N namespace, long time) {
   eventTimeTimersQueue.add(new TimerHeapInternalTimer<>(time, (K) keyContext.getCurrentKey(), namespace));
}

//TimerHeapInternalTimer构造方法，包含三个成员，timestamp key namespace(即是哪个window) 
TimerHeapInternalTimer(long timestamp, @Nonnull K key, @Nonnull N namespace) {
   this.timestamp = timestamp;
   this.key = key;
   this.namespace = namespace;
   this.timerHeapIndex = NOT_CONTAINED;
}

//HeapPriorityQueueSet类
public boolean add(@Nonnull T element) {
    //有重复的key就不用再添加了
   return getDedupMapForElement(element).putIfAbsent(element, element) == null && super.add(element);
}

```
可以看到，上述代码对每个数据都按照key添加一个TimerHeapInternalTimer（key重复的就不再继续添加了）到eventTimeTimersQueue里，TimerHeapInternalTimer包含了三个成员timestamp、 key 、namespace(即是哪个window) ，在这个同一个窗口的timer的timestamp都一样，都是window.maxTimestamp()，这是因为在调整watermark时，要么一个窗口的所有数据触发计算，要么都不触发，不会存在部分触发，部分不触发的情况；

eventTimeTimersQueue是一个优先队列（堆），队列头部是timestamp最小的timer，也就是说队列前面的都是时间靠前的窗口timer。等到watermark水位涨到window.maxTimestamp()时，队列前面的timer开始出队，此时窗口开始触发计算。

## StreamSink处理watermark
到此，windowOperator对Watermark的核心处理就完成了，下面windowOperator会继续将watermark发送给下游，在本例中也就是StreamSink。本例中windowOperator和StreamSink是chain在一起的，所以不需要广播给StreamSink的每一个实例，只有在上下游Task之间才会进行广播。

会调用StreamSink.processWatermark()来处理watermark。
```
//AbstractStreamOperator类
public void processWatermark(Watermark mark) throws Exception {
   if (timeServiceManager != null) {
      timeServiceManager.advanceWatermark(mark);
   }
   output.emitWatermark(mark);
}

//CountingOutput类
public void emitWatermark(Watermark mark) {
   output.emitWatermark(mark);
}

//BroadcastingOutputCollector类
public void emitWatermark(Watermark mark) {
   watermarkGauge.setCurrentWatermark(mark.getTimestamp());
   if (streamStatusProvider.getStreamStatus().isActive()) {
       //StreamSink没有outputs，所以watermark不需要再往下发了，到此为止
      for (Output<StreamRecord<T>> output : outputs) {
         output.emitWatermark(mark);
      }
   }
}
```
到StreamSink，watermark没有做什么有价值的操作，也不需要再往下发了。

到此，watermark的路程也就结束了。

## 总结

watermark相关的源码分析

1、Watermark的结构非常简单，只有一个时间戳timestamp，用户调用stream.assignTimestampsAndWatermarks(assigner)方法会创建一个DataStream，同时会生成一个TimestampsAndPeriodicWatermarksOperator（针对定期生成的watermark来说），它就是一个普通的operator，可以和filter/map operator进行chain

2、WatermarksOperator的open()方法中会注册一个定时线程，定时调用WatermarksOperator.onProcessTime()方法来将watermark发送出去(如果新的watermark>当前的watermark)，定时的间隔就是用户设置的周期间隔，如果WatermarksOperator和map等operator进行了chain，那么watermark就直接发送到chain的map等operator中，注意这里并不是对watermark进行广播

3、如果WatermarksOperator没有和map等operator进行chain，或者watermark已经发送到了chain operator的最后一个，那么watermark将会被进行广播到下游的每一个channel中，即上游的一个task会将watermark发送到下游的所有的task中。

4、下游的task通过statusWatermarkValve.inputWatermark()来处理watermark消息，下游的每一个task都会记录上游所有inputChannel（即所有上游task）的状态信息，状态信息里就包括watermark的值，下游task在接收到上游task广播的watermark之后，会更新状态信息的watermark值，同时从上游所有task的watermark中选择出最小的watermark值，如果这个值大于最近一次发送出去的watermark，就调用handleWatermark()来处理这个watermark消息。

5、如果下游的task还是map等operator，那么就直接将watermark再次进行发送，或直接发送个chain的其他operator，或者广播给下游task。如果这个operator是windowOperator，那么会调用InternalTimerServiceImpl.advanceWatermark()方法，大致实现是如果watermark大于eventTimeTimersQueue中头元素timer的时间，那么从eventTimeTimersQueue出队timer，也意味着触发了窗口计算。eventTimeTimersQueue是一个优先队列，头元素timer是timestamp值最小的。

6、窗口触发计算的形式和ProcessTime类似，都是一个key一个key的获取状态数据，然后依次发送出去，如果到了窗口状态清理时间（=window.maxTimestamp() + allowedLateness），还会清除当前key的状态数据，窗口计算完之后调用output.emitWatermark(mark)将watermark继续发送给下游。

7、在EventTime下的窗口计算，对于每个到达窗口的元素都会调用EventTimeTrigger.onElement()，在一般情况下，如果不返回FIRE，就会给当前数据按照key给添加一个timer到eventTimeTimersQueue中（如果key之前已经有了，就不再重复添加了）。onElement()返回FIRE的情况是比较特殊的，主要针对的迟到数据处于 window.maxTimestamp =< watermark <= window.maxTimestamp+allowedLateness的情况。

8、windowOperator在处理完窗口数据后，会接着将watermark往下游进行发送，直到最后到达StreamSink，StreamSink处于流的末尾，没有下一步的输出了，到此，watermark的旅程也就结束了。

到此，watermark的分析完毕