---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码TracerContext

## 链路追踪上下文

## AbstractTracerContext
AbstractTracerContext代表链路追踪过程上下文管理器
```
/**
 * AbstractTracerContext代表链路追踪过程上下文管理器
 * The <code>AbstractTracerContext</code> represents the tracer context manager.
 */
public interface AbstractTracerContext {
    /**
     * 为跨进程传播做好准备.将数据放到contextCarrier中
     * Prepare for the cross-process propagation. How to initialize the carrier, depends on the implementation.
     *
     * @param carrier to carry the context for crossing process.
     */
    void inject(ContextCarrier carrier);

    /**
     * 在当前segment和跨进程segment之间构建引用
     * Build the reference between this segment and a cross-process segment. How to build, depends on the
     * implementation.
     *
     * @param carrier carried the context from a cross-process segment.
     */
    void extract(ContextCarrier carrier);

```

跨进程时，inject()方法将前一个TraceSegment的数据打包放到ContextCarrier中，传递到后一个TraceSegment，extract()方法负责解压ContextCarrier中的数据
```
public interface AbstractTracerContext {

    /**
     * 在跨线程传播时拍摄快照
     * Capture a snapshot for cross-thread propagation. It's a similar concept with ActiveSpan.Continuation in
     * OpenTracing-java How to build, depends on the implementation.
     *
     * @return the {@link ContextSnapshot} , which includes the reference context.
     */
    ContextSnapshot capture();

    /**
     * 在当前segment和跨线程segment之间构建引用
     * Build the reference between this segment and a cross-thread segment. How to build, depends on the
     * implementation.
     *
     * @param snapshot from {@link #capture()} in the parent thread.
     */
    void continued(ContextSnapshot snapshot);

```

inject()和extract()方法是在跨进程传播数据时使用的，capture()和continued()方法是在跨线程传播数据时使用的
```
public interface AbstractTracerContext {

    /**
     * 获取全局traceId
     * Get the global trace id, if needEnhance. How to build, depends on the implementation.
     *
     * @return the string represents the id.
     */
    String getReadablePrimaryTraceId();

    /**
     * 获取当前segmentId
     * Get the current segment id, if needEnhance. How to build, depends on the implementation.
     *
     * @return the string represents the id.
     */
    String getSegmentId();

    /**
     * 获取active span id
     * Get the active span id, if needEnhance. How to build, depends on the implementation.
     *
     * @return the string represents the id.
     */
    int getSpanId();

    /**
     * 创建EntrySpan
     * Create an entry span
     *
     * @param operationName most likely a service name
     * @return the span represents an entry point of this segment.
     */
    AbstractSpan createEntrySpan(String operationName);

    /**
     * 创建LocalSpan
     * Create a local span
     *
     * @param operationName most likely a local method signature, or business name.
     * @return the span represents a local logic block.
     */
    AbstractSpan createLocalSpan(String operationName);

    /**
     * 创建ExitSpan
     * Create an exit span
     *
     * @param operationName most likely a service name of remote
     * @param remotePeer    the network id(ip:port, hostname:port or ip1:port1,ip2,port, etc.). Remote peer could be set
     *                      later, but must be before injecting.
     * @return the span represent an exit point of this segment.
     */
    AbstractSpan createExitSpan(String operationName, String remotePeer);

    /**
     * 拿到active span
     * @return the active span of current tracing context(stack)
     */
    AbstractSpan activeSpan();

    /**
     * 停止span,传入的span应该是active span
     * Finish the given span, and the given span should be the active span of current tracing context(stack)
     *
     * @param span to finish
     * @return true when context should be clear.
     */
    boolean stopSpan(AbstractSpan span);

    /**
     * 等待异步span结束
     * Notify this context, current span is going to be finished async in another thread.
     *
     * @return The current context
     */
    AbstractTracerContext awaitFinishAsync();

    /**
     * 关闭异步span
     * The given span could be stopped officially.
     *
     * @param span to be stopped.
     */
    void asyncStop(AsyncSpan span);

```

## TracingContext
TracingContext是一个核心的链路追踪逻辑控制器，实现了AbstractTracerContext接口，使用栈的工作机制来构建TracingContext

TracingContext负责管理：

- 当前Segment和自己前后的Segment的引用TraceSegmentRef
- 当前Segment内的所有Span

TracingContext中定义的属性如下：
```
/**
 * TracingContext是一个核心的链路追踪逻辑控制器.使用栈的工作机制来构建TracingContext
 * The <code>TracingContext</code> represents a core tracing logic controller. It build the final {@link
 * TracingContext}, by the stack mechanism, which is similar with the codes work.
 * <p>
 * 在opentracing,在一个Segment中的所有Span是父子关系而不是兄弟关系
 * In opentracing concept, it means, all spans in a segment tracing context(thread) are CHILD_OF relationship, but no
 * FOLLOW_OF.
 * <p>
 * 在skywalking核心概念里,兄弟关系是一个抽象的概念,当跨进程MQ或跨线程做异步批量任务时,skywalking使用TraceSegmentRef来实现这个场景
 * 初始化TraceSegmentRef的数据来源于ContextCarrier(跨进程)或ContextSnapshot(跨线程)
 * In skywalking core concept, FOLLOW_OF is an abstract concept when cross-process MQ or cross-thread async/batch tasks
 * happen, we used {@link TraceSegmentRef} for these scenarios. Check {@link TraceSegmentRef} which is from {@link
 * ContextCarrier} or {@link ContextSnapshot}.
 *
 * TracingContext管理:
 *   当前Segment和自己前后的Segment的引用TraceSegmentRef
 *   当前Segment内的所有Span
 */
public class TracingContext implements AbstractTracerContext {

    /**
     * 一个TracingContext对应一个TraceSegment
     * The final {@link TraceSegment}, which includes all finished spans.
     */
    private TraceSegment segment;

    /**
     * active spans存储在一个栈里
     * Active spans stored in a Stack, usually called 'ActiveSpanStack'. This {@link LinkedList} is the in-memory
     * storage-structure. <p> I use {@link LinkedList#removeLast()}, {@link LinkedList#addLast(Object)} and {@link
     * LinkedList#getLast()} instead of {@link #pop()}, {@link #push(AbstractSpan)}, {@link #peek()}
     */
    private LinkedList<AbstractSpan> activeSpanStack = new LinkedList<>();
    /**
     * @since 7.0.0 SkyWalking support lazy injection through {@link ExitTypeSpan#inject(ContextCarrier)}. Due to that,
     * the {@link #activeSpanStack} could be blank by then, this is a pointer forever to the first span, even the main
     * thread tracing has been finished.
     */
    private AbstractSpan firstSpan = null;

    /**
     * span id生成器
     * A counter for the next span.
     */
    private int spanIdGenerator;

    /**
     * 异步span计数器.使用ASYNC_SPAN_COUNTER_UPDATER进行更新
     * The counter indicates
     */
    @SuppressWarnings("unused") // updated by ASYNC_SPAN_COUNTER_UPDATER
    private volatile int asyncSpanCounter;
    private static final AtomicIntegerFieldUpdater<TracingContext> ASYNC_SPAN_COUNTER_UPDATER =
        AtomicIntegerFieldUpdater.newUpdater(TracingContext.class, "asyncSpanCounter");
    private volatile boolean isRunningInAsyncMode;
    private volatile ReentrantLock asyncFinishLock;

    // 当前TracingContext是否在运行
    private volatile boolean running;

    // 当前TracingContext的创建时间
    private final long createTime;

    //CDS watcher
    // 每个Segment里可以放多少个Span配置项的监听器
    private final SpanLimitWatcher spanLimitWatcher;

```
TraceSegment中所有创建的Span都会入栈到activeSpanStack中，Span finish的时候会出站，栈顶的Span就是activeSpan

TracingContext中get方法：
```
public class TracingContext implements AbstractTracerContext {
  
    @Override
    public String getReadablePrimaryTraceId() {
        return getPrimaryTraceId().getId();
    }

    private DistributedTraceId getPrimaryTraceId() {
        return segment.getRelatedGlobalTrace();
    }

    @Override
    public String getSegmentId() {
        return segment.getTraceSegmentId();
    }

    @Override
    public int getSpanId() {
        return activeSpan().getSpanId();
    }
  
    /**
     * activeSpanStack栈顶的span就是activeSpan
     * @return the active span of current context, the top element of {@link #activeSpanStack}
     */
    @Override
    public AbstractSpan activeSpan() {
        AbstractSpan span = peek();
        if (span == null) {
            throw new IllegalStateException("No active span.");
        }
        return span;
    }
  
    /**
     * @return the top element of 'ActiveSpanStack' only.
     */
    private AbstractSpan peek() {
        if (activeSpanStack.isEmpty()) {
            return null;
        }
        return activeSpanStack.getLast();
    }  

```
activeSpan()方法就是取activeSpanStack栈顶的元素，所以说activeSpanStack栈顶的Span就是activeSpan

TracingContext中创建Span的方法：
```
public class TracingContext implements AbstractTracerContext {

    /**
     * Create an entry span
     *
     * @param operationName most likely a service name
     * @return span instance. Ref to {@link EntrySpan}
     */
    @Override
    public AbstractSpan createEntrySpan(final String operationName) {
        if (isLimitMechanismWorking()) {
            NoopSpan span = new NoopSpan();
            return push(span);
        }
        AbstractSpan entrySpan;
        TracingContext owner = this;
        // 弹出栈顶span作为当前要创建的这个span的parent
        final AbstractSpan parentSpan = peek();
        // 拿到parentSpan的id,如果parent不存在,则parentSpanId = -1
        final int parentSpanId = parentSpan == null ? -1 : parentSpan.getSpanId();
        // 如果parentSpan是EntrySpan则复用,否则new EntrySpan并入栈
        if (parentSpan != null && parentSpan.isEntry()) {
            /*
             * Only add the profiling recheck on creating entry span,
             * as the operation name could be overrided.
             */
            profilingRecheck(parentSpan, operationName);
            parentSpan.setOperationName(operationName);
            entrySpan = parentSpan;
            return entrySpan.start();
        } else {
            entrySpan = new EntrySpan(
                spanIdGenerator++, parentSpanId,
                operationName, owner
            );
            entrySpan.start();
            return push(entrySpan);
        }
    }

    /**
     *
     * @return true 表示不允许再创建更多的Span false 相反
     */
    private boolean isLimitMechanismWorking() {
        if (spanIdGenerator >= spanLimitWatcher.getSpanLimit()) {
            long currentTimeMillis = System.currentTimeMillis();
            if (currentTimeMillis - lastWarningTimestamp > 30 * 1000) {
                LOGGER.warn(
                    new RuntimeException("Shadow tracing context. Thread dump"),
                    "More than {} spans required to create", spanLimitWatcher.getSpanLimit()
                );
                lastWarningTimestamp = currentTimeMillis;
            }
            return true;
        } else {
            return false;
        }
    }

    /**
     * Create a local span
     *
     * @param operationName most likely a local method signature, or business name.
     * @return the span represents a local logic block. Ref to {@link LocalSpan}
     */
    @Override
    public AbstractSpan createLocalSpan(final String operationName) {
        if (isLimitMechanismWorking()) {
            NoopSpan span = new NoopSpan();
            return push(span);
        }
        AbstractSpan parentSpan = peek();
        final int parentSpanId = parentSpan == null ? -1 : parentSpan.getSpanId();
        AbstractTracingSpan span = new LocalSpan(spanIdGenerator++, parentSpanId, operationName, this);
        span.start();
        return push(span);
    }
  
    /**
     * Create an exit span
     *
     * @param operationName most likely a service name of remote
     * @param remotePeer    the network id(ip:port, hostname:port or ip1:port1,ip2,port, etc.). Remote peer could be set
     *                      later, but must be before injecting.
     * @return the span represent an exit point of this segment.
     * @see ExitSpan
     */
    @Override
    public AbstractSpan createExitSpan(final String operationName, final String remotePeer) {
        if (isLimitMechanismWorking()) {
            NoopExitSpan span = new NoopExitSpan(remotePeer);
            return push(span);
        }

        AbstractSpan exitSpan;
        AbstractSpan parentSpan = peek();
        TracingContext owner = this;
        if (parentSpan != null && parentSpan.isExit()) {
            exitSpan = parentSpan;
        } else {
            final int parentSpanId = parentSpan == null ? -1 : parentSpan.getSpanId();
            exitSpan = new ExitSpan(spanIdGenerator++, parentSpanId, operationName, remotePeer, owner);
            push(exitSpan);
        }
        exitSpan.start();
        return exitSpan;
    }  

```
TracingContext中创建Span的方法处理逻辑如下：

- 判断当前TraceSegment是否能创建更多的Span，如果不能初始化NoopXxxSpan然后入栈
- 如果能创建，弹出activeSpanStack栈顶的Span（也就是activeSpan）作为当前要创建的这个Span的parent，拿到parentSpan的Id，如果parent不存在，则parentSpanId=-1
- 创建EntrySpan和ExitSpan时，会判断如果parentSpan也是同类型的Span则复用，否则才会初始化并入栈。创建LocalSpan时直接初始化并入栈

TracingContext中stopSpan()方法：
```
public class TracingContext implements AbstractTracerContext {

    /**
     * 停止span,当前仅当给定的span是activeSpanStack栈顶元素
     * Stop the given span, if and only if this one is the top element of {@link #activeSpanStack}. Because the tracing
     * core must make sure the span must match in a stack module, like any program did.
     *
     * @param span to finish
     */
    @Override
    public boolean stopSpan(AbstractSpan span) {
        AbstractSpan lastSpan = peek();
        // 如果传入的span是activeSpanStack栈顶的span,栈顶的span出栈
        if (lastSpan == span) {
            // 如果是AbstractTracingSpan,调用自身的finish方法
            if (lastSpan instanceof AbstractTracingSpan) {
                AbstractTracingSpan toFinishSpan = (AbstractTracingSpan) lastSpan;
                if (toFinishSpan.finish(segment)) {
                    pop();
                }
            } else {
                pop();
            }
        } else {
            throw new IllegalStateException("Stopping the unexpected span = " + span);
        }

        finish();

        return activeSpanStack.isEmpty();
    }
  
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

```
stopSpan()方法处理逻辑如下：

- 传入的Span必须是activeSpanStack栈顶的Span，否则抛出异常
- 栈顶的Span出栈，如果栈顶的Span是AbstractTracingSpan，调用Span自身的finish方法

如果栈已经空了且当前TracingContext还在运行状态
- 关闭当前TraceSegment
- 将当前TraceSegment交给TracingContextListener去处理，TracingContextListener会将TraceSegment发送到OAP
- 修改当前TracingContext运行状态为false

## inject()和extract()
TracingContext中inject()和extract()方法：
```
public class TracingContext implements AbstractTracerContext {

    /**
     * Inject the context into the given carrier, only when the active span is an exit one.
     *
     * @param carrier to carry the context for crossing process.
     * @throws IllegalStateException if (1) the active span isn't an exit one. (2) doesn't include peer. Ref to {@link
     *                               AbstractTracerContext#inject(ContextCarrier)}
     */
    @Override
    public void inject(ContextCarrier carrier) {
        this.inject(this.activeSpan(), carrier);
    }

    /**
     * 当前仅当active span是ExitSpan时,才将上下文注入给定的ContextCarrier和span
     * Inject the context into the given carrier and given span, only when the active span is an exit one. This method
     * wouldn't be opened in {@link ContextManager} like {@link #inject(ContextCarrier)}, it is only supported to be
     * called inside the {@link ExitTypeSpan#inject(ContextCarrier)}
     *
     * @param carrier  to carry the context for crossing process.
     * @param exitSpan to represent the scope of current injection.
     * @throws IllegalStateException if (1) the span isn't an exit one. (2) doesn't include peer.
     */
    public void inject(AbstractSpan exitSpan, ContextCarrier carrier) {
        if (!exitSpan.isExit()) {
            throw new IllegalStateException("Inject can be done only in Exit Span");
        }

        ExitTypeSpan spanWithPeer = (ExitTypeSpan) exitSpan;
        String peer = spanWithPeer.getPeer();
        if (StringUtil.isEmpty(peer)) {
            throw new IllegalStateException("Exit span doesn't include meaningful peer information.");
        }

        carrier.setTraceId(getReadablePrimaryTraceId());
        carrier.setTraceSegmentId(this.segment.getTraceSegmentId());
        // 下一个TraceSegment第一个EntrySpan的parentId就是carrier中设置的spanId(TraceSegmentRef中使用)
        carrier.setSpanId(exitSpan.getSpanId());
        carrier.setParentService(Config.Agent.SERVICE_NAME);
        carrier.setParentServiceInstance(Config.Agent.INSTANCE_NAME);
        // 栈底span(EntrySpan)的OperationName
        carrier.setParentEndpoint(first().getOperationName());
        carrier.setAddressUsedAtClient(peer);

        this.correlationContext.inject(carrier);
        this.extensionContext.inject(carrier);
    }
  
    /**
     * Extract the carrier to build the reference for the pre segment.
     *
     * @param carrier carried the context from a cross-process segment. Ref to {@link AbstractTracerContext#extract(ContextCarrier)}
     */
    @Override
    public void extract(ContextCarrier carrier) {
        TraceSegmentRef ref = new TraceSegmentRef(carrier);
        // 设置当前TraceSegment的TraceSegmentRef和traceId
        this.segment.ref(ref);
        this.segment.relatedGlobalTrace(new PropagatedTraceId(carrier.getTraceId()));
        // 如果栈顶span是EntrySpan,设置TraceSegmentRef
        AbstractSpan span = this.activeSpan();
        if (span instanceof EntrySpan) {
            span.ref(ref);
        }

        carrier.extractExtensionTo(this);
        carrier.extractCorrelationTo(this);
    }  

```
客户端A、服务端B两个应用服务，当发生一次A调用B的时候，跨进程传播的步骤如下：

- 客户端A创建一个ExitSpan，调用TracingContext的inject()方法初始化ContextCarrier
- 使用ContextCarrier的items()方法将ContextCarrier所有元素放到调用过程中的请求信息中，比如HTTP的请求头、Dubbo的attachments、Kafka的消息头中
- ContextCarrier随请求传输到服务端
- 服务端B接收具有ContextCarrier的请求，并提取ContextCarrier相关的所有信息
- 服务端B创建EntrySpan，调用TracingContext的extract()方法绑定当前TraceSegment的traceSegmentRef、traceId以及EntrySpan的ref

## 异步 Trace
SKywalking Agent 用来存储 Span 的容器是 ThreadLocal，便于在单个线程中，随时取出 Span 对象。当栈为空时，则会删除 ThreadLocal 对象，防止内存泄漏。

那如果是异步 Trace，该怎么办呢？SKywalking 提供了 capture 和 continued(snapshot)，前者表示将当前栈顶的 Span 复制并返回一个快照，continued 表示将快照恢复为当前栈顶 Span 的父 Span，以此来完成 Span 和 Span 之间的链接。

例如，当我们使用异步线程执行任务时，SKywalking 在默认情况下，是无法链接当前线程的 Span 和异步线程的 Span 的，除非我们在 Runnable 实现类使用 TraceCrossThread 类似的注解，表示这个 Runnable 需要跨线程追踪，那么，SKywalking 就会做出 capture 和 continued(snapshot) 操作，将主线程的 Span+Segment 复制到 Runnable 中，并将这 2 个 Span 进行链接。

主线程复制当前线程 Segment 和 Span 的基本信息，包括 Segment ID，Span ID，Name 等信息。然后在子线程中，进行回放，回放的操作，就是将这个 快照 的信息，保存到 Span 的父 Span 中，标记子线程的父 Span 就是这个 Span。

TracingContext中capture()和continued()方法：
```
public class TracingContext implements AbstractTracerContext {

    /**
     * Capture the snapshot of current context.
     *
     * @return the snapshot of context for cross-thread propagation Ref to {@link AbstractTracerContext#capture()}
     */
    @Override
    public ContextSnapshot capture() {
        // 初始化ContextSnapshot
        ContextSnapshot snapshot = new ContextSnapshot(
            segment.getTraceSegmentId(),
            activeSpan().getSpanId(),
            getPrimaryTraceId(),
            first().getOperationName(),
            this.correlationContext,
            this.extensionContext
        );

        return snapshot;
    }

    /**
     * Continue the context from the given snapshot of parent thread.
     *
     * @param snapshot from {@link #capture()} in the parent thread. Ref to {@link AbstractTracerContext#continued(ContextSnapshot)}
     */
    @Override
    public void continued(ContextSnapshot snapshot) {
        if (snapshot.isValid()) {
            TraceSegmentRef segmentRef = new TraceSegmentRef(snapshot);
            this.segment.ref(segmentRef);
            this.activeSpan().ref(segmentRef);
            this.segment.relatedGlobalTrace(snapshot.getTraceId());
            this.correlationContext.continued(snapshot);
            this.extensionContext.continued(snapshot);
            this.extensionContext.handle(this.activeSpan());
        }
    }

```
还有一种场景的异步 Span，比如在 A 线程开启，在 B 线程关闭，我们需要记录这个 Span 的耗时。比方说，异步 HttpClient，我们在主线程开启了访问，在异步线程得到结果，就复合刚刚我们说的场景。

SKywalking 为我们提供了 prepareForAsync 和 asyncFinish 这两个方法，当我们在 A 线程创建了一个 Span，我们可以执行 span.prepareForAsync 方法，表示这个 span 开始了访问，即将进入异步线程。

当在 B 线程得到结果后，执行 span.asyncFinish 则表示，这个 span 执行结束了，那么， A 线程就可以将整个 Segment标记结束，并返回到 OAP server 中进行统计。那么如何在 B 线程里得到这个 Span 的实例，然后调用 asyncFinish 方法呢？实际上，是需要插件开发者自己想办法传递的，比如在被拦截对象的参数里、构造函数里传递。

那么这 2 种异步模式的区别是什么呢？说实话，我在刚刚看到这两个的时候，脑子也有点迷糊，经过总结，发现两者虽然看起来相似，当谁也代替不了谁。

简单来说，prepareForAsync 和 asyncFinish 只是为了统计一个 Span 跨越 2 个线程的场景，例如上面的提到 HttpAsyncClient 场景。在 A 线程创建，在 B 线程结束，我们需要在 B 线程拿到返回值和耗时。

而 capture 和 continued(snapshot) 的使用场景是为了连接 2 个线程的不同 Span。我们将主线程的最后一个 Span 和子线程的第一个 Span 相连接。