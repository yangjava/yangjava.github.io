---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码链路发送

## 链路数据发送到OAP
```
public class TracingContext implements AbstractTracerContext {

    /**
     * 结束TracingContext
     * Finish this context, and notify all {@link TracingContextListener}s, managed by {@link
     * TracingContext.ListenerManager} and {@link TracingContext.TracingThreadListenerManager}
     */
    private void finish() {
        if (isRunningInAsyncMode) {
            asyncFinishLock.lock();
        }
        try {
            // 栈已经空了 且 当前TracingContext还在运行状态
            boolean isFinishedInMainThread = activeSpanStack.isEmpty() && running;
            if (isFinishedInMainThread) {
                /*
                 * Notify after tracing finished in the main thread.
                 */
                TracingThreadListenerManager.notifyFinish(this);
            }

            if (isFinishedInMainThread && (!isRunningInAsyncMode || asyncSpanCounter == 0)) {
                // 关闭当前TraceSegment
                TraceSegment finishedSegment = segment.finish(isLimitMechanismWorking());
                // 将当前TraceSegment交给TracingContextListener去处理,TracingContextListener会将TraceSegment发送到OAP
                TracingContext.ListenerManager.notifyFinish(finishedSegment);
                // 修改当前TracingContext运行状态为false
                running = false;
            }
        } finally {
            if (isRunningInAsyncMode) {
                asyncFinishLock.unlock();
            }
        }
    }
  
    /**
     * The <code>ListenerManager</code> represents an event notify for every registered listener, which are notified
     * when the <code>TracingContext</code> finished, and {@link #segment} is ready for further process.
     */
    public static class ListenerManager {
        private static List<TracingContextListener> LISTENERS = new LinkedList<>();

        /**
         * Add the given {@link TracingContextListener} to {@link #LISTENERS} list.
         *
         * @param listener the new listener.
         */
        public static synchronized void add(TracingContextListener listener) {
            LISTENERS.add(listener);
        }

        /**
         * Notify the {@link TracingContext.ListenerManager} about the given {@link TraceSegment} have finished. And
         * trigger {@link TracingContext.ListenerManager} to notify all {@link #LISTENERS} 's {@link
         * TracingContextListener#afterFinished(TraceSegment)}
         *
         * @param finishedSegment the segment that has finished
         */
        static void notifyFinish(TraceSegment finishedSegment) {
            for (TracingContextListener listener : LISTENERS) {
                listener.afterFinished(finishedSegment);
            }
        }

        /**
         * Clear the given {@link TracingContextListener}
         */
        public static synchronized void remove(TracingContextListener listener) {
            LISTENERS.remove(listener);
        }

    }  

```
TracingContext的finish()方法关闭当前TraceSegment后，会调用ListenerManager的notifyFinish()方法传入当前关闭的TraceSegment。ListenerManager的notifyFinish()方法会迭代所有注册的TracingContextListener调用它们的afterFinished()方法

TraceSegmentServiceClient实现了TracingContextListener接口，并向ListenerManager注册了自己，afterFinished()方法代码如下：
```
/**
 * 将TraceSegment发送到OAP
 */
@DefaultImplementor
public class TraceSegmentServiceClient implements BootService, IConsumer<TraceSegment>, TracingContextListener, GRPCChannelListener {

    private volatile DataCarrier<TraceSegment> carrier;

    @Override
    public void afterFinished(TraceSegment traceSegment) {
        if (traceSegment.isIgnore()) {
            return;
        }
        // 将traceSegment放到dataCarrier中
        if (!carrier.produce(traceSegment)) {
            if (LOGGER.isDebugEnable()) {
                LOGGER.debug("One trace segment has been abandoned, cause by buffer is full.");
            }
        }
    }

```

afterFinished()方法中会将TraceSegment放到DataCarrier中

TraceSegmentServiceClient也实现了IConsumer接口，消费DataCarrier中的TraceSegment数据，consume()方法代码如下：
```
@DefaultImplementor
public class TraceSegmentServiceClient implements BootService, IConsumer<TraceSegment>, TracingContextListener, GRPCChannelListener {

    // 上一次打印传输traceSegment情况的日志的时间
    private long lastLogTime;
    // 成功发送的traceSegment数量
    private long segmentUplinkedCounter;
    // 因网络原因丢弃的traceSegment数量
    private long segmentAbandonedCounter;
    private volatile TraceSegmentReportServiceGrpc.TraceSegmentReportServiceStub serviceStub;
    private volatile GRPCChannelStatus status = GRPCChannelStatus.DISCONNECT;

    @Override
    public void consume(List<TraceSegment> data) {
        if (CONNECTED.equals(status)) {
            final GRPCStreamServiceStatus status = new GRPCStreamServiceStatus(false);
            StreamObserver<SegmentObject> upstreamSegmentStreamObserver = serviceStub.withDeadlineAfter(
                Config.Collector.GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS
            ).collect(new StreamObserver<Commands>() {
                @Override
                public void onNext(Commands commands) {
                    ServiceManager.INSTANCE.findService(CommandService.class)
                                           .receiveCommand(commands);
                }

                @Override
                public void onError(
                    Throwable throwable) {
                    status.finished();
                    if (LOGGER.isErrorEnable()) {
                        LOGGER.error(
                            throwable,
                            "Send UpstreamSegment to collector fail with a grpc internal exception."
                        );
                    }
                    ServiceManager.INSTANCE
                        .findService(GRPCChannelManager.class)
                        .reportError(throwable);
                }

                @Override
                public void onCompleted() {
                    status.finished();
                }
            });

            try {
                for (TraceSegment segment : data) {
                    SegmentObject upstreamSegment = segment.transform();
                    // 发送到OAP
                    upstreamSegmentStreamObserver.onNext(upstreamSegment);
                }
            } catch (Throwable t) {
                LOGGER.error(t, "Transform and send UpstreamSegment to collector fail.");
            }

            upstreamSegmentStreamObserver.onCompleted();
            // 强制等待所有的traceSegment都发送完成
            status.wait4Finish();
            segmentUplinkedCounter += data.size();
        } else {
            segmentAbandonedCounter += data.size();
        }

        printUplinkStatus();
    }

```

































