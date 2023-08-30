---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink源码WindowAll
window操作通常在keyBy操作之后，它会创建一个windowedStream，windowedStream经过reduce、process、apply等操作会创建一个新的DataStream，新的DataStream拥有一个windowOperator，数据的核心处理就在windowOperator.processElement()方法。

那么windowAll又是如何实现的呢？

先看如下示例代码：
```
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)
    val text: DataStream[String] = env.socketTextStream(hostname, port)

    // parse the data, group it, window it, and aggregate the counts
    val windowCounts = text
      .flatMap { w => w.split("\\s") }
      .map { w => WordWithCount(w, 1) }
      .keyBy("word")
      .timeWindow(Time.seconds(60))
      .reduce(_ + _)
      .timeWindowAll(Time.seconds(5))
      .apply{ (_: TimeWindow, input: Iterable[WordWithCount], out: Collector[WordWithCount]) => input.foreach{out.collect}  }

    // print the results with a single thread, rather than in parallel
    windowCounts.print().setParallelism(1)

    env.execute("Socket Window WordCount")
```
示例代码中有两个连续窗口操作，第一个窗口在keyBy操作后，是我们通常用的window操作，它是并行化处理的。reduce()方法会产生一个新的DataStream，然后再调用timeWindowAll()方法，创建一个全局的window。



## 创建AllWindowedStream
首先我们就从timeWindowAll()这个方法开始分析
```
public AllWindowedStream<T, TimeWindow> timeWindowAll(Time size) {
   if (environment.getStreamTimeCharacteristic() == TimeCharacteristic.ProcessingTime) {
      return windowAll(TumblingProcessingTimeWindows.of(size));
   } else {
      return windowAll(TumblingEventTimeWindows.of(size));
   }
}

public <W extends Window> AllWindowedStream<T, W> windowAll(WindowAssigner<? super T, W> assigner) {
   return new AllWindowedStream<>(this, assigner);
}

public AllWindowedStream(DataStream<T> input,
      WindowAssigner<? super T, W> windowAssigner) {
   //很重要
   this.input = input.keyBy(new NullByteKeySelector<T>());
   this.windowAssigner = windowAssigner;
   this.trigger = windowAssigner.getDefaultTrigger(input.getExecutionEnvironment());
}
```
可以看到windowAll操作生成了一个AllWindowedStream，但是AllWindowedStream的input却是在原DataStream上经过keyBy操作之后生成的KeyedStream, 而不是原调用windowAll的DataStream。keyBy操作的KeySelector是NullByteKeySelector。

NullByteKeySelector的getKey()方法直接返回0，跟参数value无关。
```
public class NullByteKeySelector<T> implements KeySelector<T, Byte> {

   private static final long serialVersionUID = 614256539098549020L;

   @Override
   public Byte getKey(T value) throws Exception {
      return 0;
   }
}
```
创建windowOperator和DataStream
再来看apply()方法：
```
def apply[R: TypeInformation](
    function: (W, Iterable[T], Collector[R]) => Unit): DataStream[R] = {
  
  val cleanedFunction = clean(function)
  val applyFunction = new ScalaAllWindowFunction[T, R, W](cleanedFunction)
  
  asScalaStream(javaStream.apply(applyFunction, implicitly[TypeInformation[R]]))
}

public <R> SingleOutputStreamOperator<R> apply(AllWindowFunction<T, R, W> function, TypeInformation<R> resultType) {
   String callLocation = Utils.getCallLocationName();
   function = input.getExecutionEnvironment().clean(function);
   return apply(new InternalIterableAllWindowFunction<>(function), resultType, callLocation);
}

private <R> SingleOutputStreamOperator<R> apply(InternalWindowFunction<Iterable<T>, R, Byte, W> function, TypeInformation<R> resultType, String callLocation) {

   String udfName = "AllWindowedStream." + callLocation;

   String opName;
   //这里的KeySelector就是NullByteKeySelector
   KeySelector<T, Byte> keySel = input.getKeySelector();

   WindowOperator<Byte, T, Iterable<T>, R, W> operator;

   if (evictor != null) {
      @SuppressWarnings({"unchecked", "rawtypes"})
      TypeSerializer<StreamRecord<T>> streamRecordSerializer =
            (TypeSerializer<StreamRecord<T>>) new StreamElementSerializer(input.getType().createSerializer(getExecutionEnvironment().getConfig()));

      ListStateDescriptor<StreamRecord<T>> stateDesc =
            new ListStateDescriptor<>("window-contents", streamRecordSerializer);

      opName = "TriggerWindow(" + windowAssigner + ", " + stateDesc + ", " + trigger + ", " + evictor + ", " + udfName + ")";

      operator =
         new EvictingWindowOperator<>(windowAssigner,
            windowAssigner.getWindowSerializer(getExecutionEnvironment().getConfig()),
            keySel,
            input.getKeyType().createSerializer(getExecutionEnvironment().getConfig()),
            stateDesc,
            function,
            trigger,
            evictor,
            allowedLateness,
            lateDataOutputTag);

   } else {
       //state为ListStateDescriptor
      ListStateDescriptor<T> stateDesc = new ListStateDescriptor<>("window-contents",
            input.getType().createSerializer(getExecutionEnvironment().getConfig()));

      opName = "TriggerWindow(" + windowAssigner + ", " + stateDesc + ", " + trigger + ", " + udfName + ")";

      operator =
         new WindowOperator<>(windowAssigner,
            windowAssigner.getWindowSerializer(getExecutionEnvironment().getConfig()),
            keySel,
            input.getKeyType().createSerializer(getExecutionEnvironment().getConfig()),
            stateDesc,
            function,
            trigger,
            allowedLateness,
            lateDataOutputTag);
   }
//这里的input是KeyedStream，transform会生成一个新的DataStream，会强制设置该DataStream的并行读为1
   return input.transform(opName, resultType, operator).forceNonParallel();
}

public SingleOutputStreamOperator<T> forceNonParallel() {
   transformation.setParallelism(1);
   transformation.setMaxParallelism(1);
   nonParallel = true;
   return this;
}
```
apply()方法会返回一个新的DataStream，这个DataStream由KeyedStream.transform生成，新的DataStream拥有一个WindowOperator。WindowOperator的keySelector是NullByteKeySelector，stateDesc是ListStateDescriptor，那么windowOperator的windowState就是HeapListState。windowOperator的窗口函数是InternalIterableAllWindowFunction。windowOperator的并行度被强制设定了1，即非并行处理

## 窗口数据处理
下面来看windowAll窗口对数据的处理过程,同样在StreamInputProcessor.processInput()方法中
```
StreamRecord<IN> record = recordOrMark.asRecord();
synchronized (lock) {
   numRecordsIn.inc();
   streamOperator.setKeyContextElement1(record);
   streamOperator.processElement(record);
}
```

```
public void setKeyContextElement1(StreamRecord record) throws Exception {
   setKeyContextElement(record, stateKeySelector1);
}

private <T> void setKeyContextElement(StreamRecord<T> record, KeySelector<T, ?> selector) throws Exception {
   if (selector != null) {
       //始终返回的是0
      Object key = selector.getKey(record.getValue());
      setCurrentKey(key);
   }
}

public void setCurrentKey(Object key) {
   if (keyedStateBackend != null) {
      try {
         // need to work around type restrictions
         @SuppressWarnings("unchecked,rawtypes")
         AbstractKeyedStateBackend rawBackend = (AbstractKeyedStateBackend) keyedStateBackend;

         rawBackend.setCurrentKey(key);
      } catch (Exception e) {
         throw new RuntimeException("Exception occurred while setting the current key context.", e);
      }
   }
}
```
对每条进入窗口的数据都要设置当前的key，对于windowAll的windowOperator，stateKeySelector1是NullByteKeySelector，对于每条进入窗口的数据，getKey()始终返回0，即所有数据的key都是一样的，这样，CurrentKey就跟数据无关了，始终为0.

再来看看windowOperator.processElement()方法，主要看看数据是如何添加到windowState中的。
```
windowState.setCurrentNamespace(stateWindow);
windowState.add(element.getValue());
```
这里的windowState是HeapListState, add()方法如下：

```
public void add(V value) {
   Preconditions.checkNotNull(value, "You cannot add null to a ListState.");

   final N namespace = currentNamespace;
//状态数据是一个list
   final StateTable<K, N, List<V>> map = stateTable;
   List<V> list = map.get(namespace);
//add方法就是将数据添加list中，未做任何处理
   if (list == null) {
      list = new ArrayList<>();
      map.put(namespace, list);
   }
   list.add(value);
}

//CopyOnWriteStateTable类
public S get(N namespace) {
    //getCurrentKey始终返回0，也就是state只跟namespace有关，即TimeWindow
   return get(keyContext.getCurrentKey(), namespace);
}

public S get(K key, N namespace) {

   final int hash = computeHashForOperationAndDoIncrementalRehash(key, namespace);
   final int requiredVersion = highestRequiredSnapshotVersion;
   final StateTableEntry<K, N, S>[] tab = selectActiveTable(hash);
   int index = hash & (tab.length - 1);

   for (StateTableEntry<K, N, S> e = tab[index]; e != null; e = e.next) {
      final K eKey = e.key;
      final N eNamespace = e.namespace;
      if ((e.hash == hash && key.equals(eKey) && namespace.equals(eNamespace))) {

         // copy-on-write check for state
         if (e.stateVersion < requiredVersion) {
            // copy-on-write check for entry
            if (e.entryVersion < requiredVersion) {
               e = handleChainedEntryCopyOnWrite(tab, hash & (tab.length - 1), e);
            }
            e.stateVersion = stateTableVersion;
            e.state = getStateSerializer().copy(e.state);
         }

         return e.state;
      }
   }

   return null;
}
```
windowState.add()方法就是根据当前数据的key和当前所在的窗口window获取状态数据，这里的状态数据是一个list。因为每条数据的key都会返回0，所以每条数据的状态数据只跟所在的窗口有关系了，根据窗口找到状态数据list，然后将数据添加到list。windowState.add()方法就完成了。可以分析出，既然所有数据key都一样，状态数据只取决于所在窗口，那么每个窗口的状态数据就只有一个！

还有就是给key添加timer到processingTimeTimersQueue中，因为所有数据的key都是一样的，所以每个窗口processingTimeTimersQueue只有一个timer。

## 窗口触发
下面再来看窗口计算的触发，windowAll和window是一样的，
```
//InternalTimerServiceImpl类
public void onProcessingTime(long time) throws Exception {
   // null out the timer in case the Triggerable calls registerProcessingTimeTimer()
   // inside the callback.
   nextTimer = null;

   InternalTimer<K, N> timer;

   while ((timer = processingTimeTimersQueue.peek()) != null && timer.getTimestamp() <= time) {
      processingTimeTimersQueue.poll();
      keyContext.setCurrentKey(timer.getKey());
      triggerTarget.onProcessingTime(timer);
   }

   if (timer != null && nextTimer == null) {
      nextTimer = processingTimeService.registerTimer(timer.getTimestamp(), this);
   }
}
//WindowOperator类
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

   if (triggerResult.isFire()) {
       //这里的contents就是一个List
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

private void emitWindowContents(W window, ACC contents) throws Exception {
   timestampedCollector.setAbsoluteTimestamp(window.maxTimestamp());
   processContext.window = window;
   //这里的userFunction就是InternalIterableAllWindowFunction
   userFunction.process(triggerContext.key, window, processContext, contents, timestampedCollector);
}

//InternalIterableAllWindowFunction类，wrappedFunction就是用户指定的apply方法里的function
public void process(Byte key, W window, InternalWindowContext context, Iterable<IN> input, Collector<OUT> out) throws Exception {
   wrappedFunction.apply(window, input, out);
}
```
当窗口触发计算时，跟普通的窗口一样，只不过这里从windowState里获取的状态数据是一个list，然后交给InternalIterableAllWindowFunction处理，InternalIterableAllWindowFunction里封装了用户apply方法里面指定的function，最终会调用用户的function来处理这个状态数据list。

当然，本文所分析的windowState是HeapListState只是windowAll其中的一种情况，状态数据不一定就只是list，这要看用户在windowAll后指定的是什么操作，例如如果是reduce操作，那么windowState就是HeapReducingState，还有很多其他的state。

到此，windowAll的实现就分析完了。

## windowAll的核心实现：

- DataStream调用windowAll()方法会生成一个AllWindowedStream，但是AllWindowedStream的input却是在原DataStream上经过keyBy操作之后生成的KeyedStream, 而不是原调用windowAll的DataStream。keyBy操作的KeySelector是NullByteKeySelector。NullByteKeySelector的getKey()方法直接返回0，跟参数value无关。

- AllWindowedStream调用apply()、reduce()、process()等方法会生成一个新的DataStream，新DataStream拥有一个windowOperator，windowOperator拥有keySelector，它就是NullByteKeySelector。windowOperator还持有stateDesc和windowFunction，拿apply()方法举例，stateDesc是ListStateDescriptor，那么windowOperator的windowState就是HeapListState，状态数据是一个List。windowOperator的窗口函数是InternalIterableAllWindowFunction，封装了apply方法里指定的function。

- 在数据进入窗口时，每条数据所得到的key都是一样的，都是0，所以每条数据的状态数据只跟namespace有关，即跟所在的窗口有关，这也就是说，每个窗口，所有处于该窗口的数据只有一个状态数据，拿apply()方法来说，状态数据就是一个list，窗口中的每条数据会添加到这个list中。

- 由于每条数据生成的key都是一样的，所在在添加key的timer到processingTimeTimersQueue这个过程，每个窗口都只会添加一个timer到processingTimeTimersQueue中，在窗口触发计算时，也只会出队这一个timer，拿到这一个timer中key所对应的状态数据，key即是0，状态数据就是整个窗口的状态数据。

- 拿apply()方法举例，这个状态数据就是说一个List，这个List最终会交给用户在apply()方法中指定的function来处理。














