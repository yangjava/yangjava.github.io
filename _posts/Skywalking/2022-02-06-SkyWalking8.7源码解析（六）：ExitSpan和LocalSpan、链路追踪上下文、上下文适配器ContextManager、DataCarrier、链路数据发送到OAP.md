27、ExitSpan和LocalSpan
1）、ExitSpan
ExitSpan代表服务消费侧，比如Feign、Okhttp。ExitSpan是链路中一个退出的点或者离开的Span。在一个RPC调用中，会有多层退出的点，而ExitSpan永远表示第一个。比如，Dubbox中使用HttpComponent发起远程调用。ExitSpan表示Dubbox的Span，并忽略HttpComponent的Span信息

EntrySpan和ExitSpan的区别就在于：

EntrySpan记录的是更靠近服务这一侧的信息
ExitSpan记录的是更靠近消费这一侧的信息
ExitSpan代码如下：

/**
 * ExitSpan代表服务消费侧,比如Feign、Okhttp
 * The <code>ExitSpan</code> represents a service consumer point, such as Feign, Okhttp client for an Http service.
 * <p>
 * 它是链路中一个退出的点或者离开的Span.在一个RPC调用中,会有多层退出的点
 * It is an exit point or a leaf span(our old name) of trace tree. In a single rpc call, because of a combination of
 * discovery libs, there maybe contain multi-layer exit point:
 * <p>
 * ExitSpan永远表示第一个
 * The <code>ExitSpan</code> only presents the first one.
 * <p>
 * 比如:Dubbox中使用HttpComponent发起远程调用.ExitSpan表示Dubbox的Span,并忽略HttpComponent的Span信息
 * Such as: Dubbox - Apache Httpcomponent - ...(Remote) The <code>ExitSpan</code> represents the Dubbox span, and ignore
 * the httpcomponent span's info.
 *
 * 区别就在于 EntrySpan记录的是更靠近服务这一侧的信息
 *          ExitSpan记录的是更靠近消费这一侧的信息
 */
public class ExitSpan extends StackBasedTracingSpan implements ExitTypeSpan {

    public ExitSpan(int spanId, int parentSpanId, String operationName, String peer, TracingContext owner) {
        super(spanId, parentSpanId, operationName, peer, owner);
    }

    public ExitSpan(int spanId, int parentSpanId, String operationName, TracingContext owner) {
        super(spanId, parentSpanId, operationName, owner);
    }

    /**
     * Set the {@link #startTime}, when the first start, which means the first service provided.
     */
     @Override
     public ExitSpan start() {
        // stackDepth = 1时,才调用start方法记录启动时间
        if (++stackDepth == 1) {
            super.start();
        }
        return this;
     }

    @Override
    public ExitSpan tag(String key, String value) {
        // stackDepth = 1时,才记录span信息
        if (stackDepth == 1 || isInAsyncMode) {
            super.tag(key, value);
        }
        return this;
    }

    @Override
    public AbstractTracingSpan tag(AbstractTag<?> tag, String value) {
        if (stackDepth == 1 || tag.isCanOverwrite() || isInAsyncMode) {
            super.tag(tag, value);
        }
        return this;
    }

    @Override
    public AbstractTracingSpan setLayer(SpanLayer layer) {
        if (stackDepth == 1 || isInAsyncMode) {
            return super.setLayer(layer);
        } else {
            return this;
        }
    }

    @Override
    public AbstractTracingSpan setComponent(Component component) {
        if (stackDepth == 1 || isInAsyncMode) {
            return super.setComponent(component);
        } else {
            return this;
        }
    }

    @Override
    public ExitSpan log(Throwable t) {
        super.log(t);
        return this;
    }

    @Override
    public AbstractTracingSpan setOperationName(String operationName) {
        if (stackDepth == 1 || isInAsyncMode) {
            return super.setOperationName(operationName);
        } else {
            return this;
        }
    }

    @Override
    public String getPeer() {
        return peer;
    }

    @Override
    public ExitSpan inject(final ContextCarrier carrier) {
        this.owner.inject(this, carrier);
        return this;
    }

    @Override
    public boolean isEntry() {
        return false;
    }

    @Override
    public boolean isExit() {
        return true;
    }
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
109
110
111

如上图，假设有一个应用部署在Tomcat上，使用SpringMVC提供一个getUser()的Controller方法，getUser()方法会在调用Redis和MySQL，对于这样一个流程的TraceSegment是怎样的？


Tomcat一进来就会创建EntrySpan，SpringMVC会复用Tomcat创建的EntrySpan。当访问Redis时会创建一个ExitSpan，peer会记录Redis地址

ExitSpan不要把理解为TraceSegment的结束，可以理解为离开当前TraceSegment的操作。当访问MySQL时也会创建一个ExitSpan，peer会记录MySQL地址。这里因为访问Redis和访问MySQL并不是嵌套关系，所以并不复用前面的ExitSpan


注意：

所谓ExitSpan和EntrySpan一样采用复用的机制，前提是在插件嵌套的情况下
多个ExitSpan不存在嵌套关系，是平行存在的时候，是允许同时存在多个ExitSpan
把ExitSpan简单理解为离开当前进程/线程的操作
TraceSegment里不一定非要有ExitSpan
2）、LocalSpan
LocalSpan继承关系如下：

LocalSpan代码如下：

/**
 * LocalSpan代表一个普通追踪的点,比如一个本地方法
 * The <code>LocalSpan</code> represents a normal tracing point, such as a local method.
 */
 public class LocalSpan extends AbstractTracingSpan {

    public LocalSpan(int spanId, int parentSpanId, String operationName, TracingContext owner) {
        super(spanId, parentSpanId, operationName, owner);
    }

    @Override
    public boolean isEntry() {
        return false;
    }

    @Override
    public boolean isExit() {
        return false;
    }

    @Override
    public AbstractSpan setPeer(String remotePeer) {
        return this;
    }
 }
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 小结：


SkyWalking中Span的继承关系如下图：


28、链路追踪上下文
1）、AbstractTracerContext
AbstractTracerContext代表链路追踪过程上下文管理器

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
  1
  2
  3
  4
  5
  6
  7
  8
  9
  10
  11
  12
  13
  14
  15
  16
  17
  18
  19
  20
  21


跨进程时，inject()方法将前一个TraceSegment的数据打包放到ContextCarrier中，传递到后一个TraceSegment，extract()方法负责解压ContextCarrier中的数据

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
inject()和extract()方法是在跨进程传播数据时使用的，capture()和continued()方法是在跨线程传播数据时使用的

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
2）、TracingContext
TracingContext是一个核心的链路追踪逻辑控制器，实现了AbstractTracerContext接口，使用栈的工作机制来构建TracingContext

TracingContext负责管理：

当前Segment和自己前后的Segment的引用TraceSegmentRef
当前Segment内的所有Span
TracingContext中定义的属性如下：

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67

TraceSegment中所有创建的Span都会入栈到activeSpanStack中，Span finish的时候会出站，栈顶的Span就是activeSpan

TracingContext中get方法：

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
activeSpan()方法就是取activeSpanStack栈顶的元素，所以说activeSpanStack栈顶的Span就是activeSpan

TracingContext中创建Span的方法：

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
TracingContext中创建Span的方法处理逻辑如下：

判断当前TraceSegment是否能创建更多的Span，如果不能初始化NoopXxxSpan然后入栈
如果能创建，弹出activeSpanStack栈顶的Span（也就是activeSpan）作为当前要创建的这个Span的parent，拿到parentSpan的Id，如果parent不存在，则parentSpanId=-1
创建EntrySpan和ExitSpan时，会判断如果parentSpan也是同类型的Span则复用，否则才会初始化并入栈。创建LocalSpan时直接初始化并入栈
TracingContext中stopSpan()方法：

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
stopSpan()方法处理逻辑如下：

传入的Span必须是activeSpanStack栈顶的Span，否则抛出异常

栈顶的Span出栈，如果栈顶的Span是AbstractTracingSpan，调用Span自身的finish方法

如果栈已经空了且当前TracingContext还在运行状态

1）关闭当前TraceSegment

2）将当前TraceSegment交给TracingContextListener去处理，TracingContextListener会将TraceSegment发送到OAP

3）修改当前TracingContext运行状态为false

TracingContext中inject()和extract()方法：

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
客户端A、服务端B两个应用服务，当发生一次A调用B的时候，跨进程传播的步骤如下：

客户端A创建一个ExitSpan，调用TracingContext的inject()方法初始化ContextCarrier
使用ContextCarrier的items()方法将ContextCarrier所有元素放到调用过程中的请求信息中，比如HTTP的请求头、Dubbo的attachments、Kafka的消息头中
ContextCarrier随请求传输到服务端
服务端B接收具有ContextCarrier的请求，并提取ContextCarrier相关的所有信息
服务端B创建EntrySpan，调用TracingContext的extract()方法绑定当前TraceSegment的traceSegmentRef、traceId以及EntrySpan的ref
TracingContext中capture()和continued()方法：

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
小结：


29、上下文适配器ContextManager
TraceSegment及其所包含的Span都在同一个线程内，ContextManager使用ThreadLocal来管理TraceSegment的上下文（也就是AbstractTracerContext）

ContextManager中getOrCreate()方法：

/**
 * TraceSegment及其所包含的Span都在同一个线程内,ContextManager使用ThreadLocal来管理TraceSegment的上下文(AbstractTracerContext)
 * {@link ContextManager} controls the whole context of {@link TraceSegment}. Any {@link TraceSegment} relates to
 * single-thread, so this context use {@link ThreadLocal} to maintain the context, and make sure, since a {@link
 * TraceSegment} starts, all ChildOf spans are in the same context. <p> What is 'ChildOf'?
 * https://github.com/opentracing/specification/blob/master/specification.md#references-between-spans
 *
 * ContextManager代理了AbstractTracerContext主要的方法
 * <p> Also, {@link ContextManager} delegates to all {@link AbstractTracerContext}'s major methods.
 */
 public class ContextManager implements BootService {

    private static ThreadLocal<AbstractTracerContext> CONTEXT = new ThreadLocal<AbstractTracerContext>();

    private static AbstractTracerContext getOrCreate(String operationName, boolean forceSampling) {
        // 从ThreadLocal中获取AbstractTracerContext,如果有就返回,没有就新建
        AbstractTracerContext context = CONTEXT.get();
        if (context == null) {
            // operationName为空创建IgnoredTracerContext
            if (StringUtil.isEmpty(operationName)) {
                if (LOGGER.isDebugEnable()) {
                    LOGGER.debug("No operation name, ignore this trace.");
                }
                context = new IgnoredTracerContext();
            } else {
                // 调用ContextManagerExtendService的createTraceContext方法创建AbstractTracerContext,并设置到ThreadLocal中
                if (EXTEND_SERVICE == null) {
                    EXTEND_SERVICE = ServiceManager.INSTANCE.findService(ContextManagerExtendService.class);
                }
                context = EXTEND_SERVICE.createTraceContext(operationName, forceSampling);

            }
            CONTEXT.set(context);
        }
        return context;
    }  
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 26
 27
 28
 29
 30
 31
 32
 33
 34
 35
 36
 getOrCreate()方法处理逻辑如下：

从ThreadLocal中获取AbstractTracerContext，如果有就返回，没有就新建
如果operationName为空创建IgnoredTracerContext
否则调用ContextManagerExtendService的createTraceContext()方法创建AbstractTracerContext，并设置到ThreadLocal中
ContextManagerExtendService的createTraceContext()方法代码如下：

@DefaultImplementor
public class ContextManagerExtendService implements BootService, GRPCChannelListener {

    /**
     * 哪些后缀的请求不需要追踪
     */
    private volatile String[] ignoreSuffixArray = new String[0];
      
    public AbstractTracerContext createTraceContext(String operationName, boolean forceSampling) {
        AbstractTracerContext context;
        /*
         * 如果OAP挂了不采样且网络连接断开,创建IgnoredTracerContext
         * Don't trace anything if the backend is not available.
         */
        if (!Config.Agent.KEEP_TRACING && GRPCChannelStatus.DISCONNECT.equals(status)) {
            return new IgnoredTracerContext();
        }
    
        int suffixIdx = operationName.lastIndexOf(".");
        // operationName的后缀名在ignoreSuffixArray中,创建IgnoredTracerContext
        if (suffixIdx > -1 && Arrays.stream(ignoreSuffixArray)
                                    .anyMatch(a -> a.equals(operationName.substring(suffixIdx)))) {
            context = new IgnoredTracerContext();
        } else {
            SamplingService samplingService = ServiceManager.INSTANCE.findService(SamplingService.class);
            // 如果是强制采样或尝试采样返回true,创建TracingContext
            if (forceSampling || samplingService.trySampling(operationName)) {
                context = new TracingContext(operationName, spanLimitWatcher);
            } else {
                context = new IgnoredTracerContext();
            }
        }
    
        return context;
    }  
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
createTraceContext()方法处理逻辑如下：

如果OAP挂了不采样且网络连接断开，创建IgnoredTracerContext
如果operationName的后缀名在ignoreSuffixArray中（指定哪些后缀的请求不需要追踪），创建IgnoredTracerContext
如果是强制采样或尝试采样（SamplingService的trySampling()方法）返回true，创建TracingContext，否则创建IgnoredTracerContext
ContextManager中createEntrySpan()方法：

public class ContextManager implements BootService {

    public static AbstractSpan createEntrySpan(String operationName, ContextCarrier carrier) {
        AbstractSpan span;
        AbstractTracerContext context;
        operationName = StringUtil.cut(operationName, OPERATION_NAME_THRESHOLD);
        if (carrier != null && carrier.isValid()) {
            SamplingService samplingService = ServiceManager.INSTANCE.findService(SamplingService.class);
            samplingService.forceSampled();
            // 一定要强制采样,因为链路中的前置TraceSegment已经存在,否则链路就可能会断开
            context = getOrCreate(operationName, true);
            span = context.createEntrySpan(operationName);
            context.extract(carrier);
        } else {
            // 不需要强制采样,根据采样率来决定当前链路是否要采样
            context = getOrCreate(operationName, false);
            span = context.createEntrySpan(operationName);
        }
        return span;
    }  
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
createEntrySpan()方法处理逻辑如下：

如果ContextCarrier不为空，强制采样，获取或创建TracingContext，创建EntrySpan，从ContextCarrier将数据提取出来放到TracingContext中
如果ContextCarrier为空，不需要强制采样，根据采样率来决定当前链路是否要采样


当创建EntrySpan时有两种情况：

请求刚刚进来处于链路的第一个TraceSegment上，如上图左边的TraceSegment，此时不需要强制采样，根据采样率来决定当前链路是否要采样
如上图右边的TraceSegment，左边TraceSegment的ExitSpan调用了右边的TraceSegment，上一个TraceSegment的数据需要传递到下一个TraceSegment，下游调用extract()方法从ContextCarrier将数据提取出来放到TracingContext中。此时一定要强制采样，因为链路中的前置TraceSegment已经存在，如果不强制采样，尝试采样（SamplingService的trySampling()方法）返回false，链路就断开了
ContextManager中创建LocalSpan和ExitSpan的方法：

public class ContextManager implements BootService {

    public static AbstractSpan createLocalSpan(String operationName) {
        operationName = StringUtil.cut(operationName, OPERATION_NAME_THRESHOLD);
        AbstractTracerContext context = getOrCreate(operationName, false);
        return context.createLocalSpan(operationName);
    }
      
    /**
     * 调用下一个受SkyWalking监控的进程,必须要ContextCarrier 比如调用Java服务
     *
     * @param operationName
     * @param carrier
     * @param remotePeer
     * @return
     */
    public static AbstractSpan createExitSpan(String operationName, ContextCarrier carrier, String remotePeer) {
        if (carrier == null) {
            throw new IllegalArgumentException("ContextCarrier can't be null.");
        }
        operationName = StringUtil.cut(operationName, OPERATION_NAME_THRESHOLD);
        AbstractTracerContext context = getOrCreate(operationName, false);
        AbstractSpan span = context.createExitSpan(operationName, remotePeer);
        context.inject(carrier);
        return span;
    }
    
    /**
     * 不需要往后传播的ExitSpan 比如调用MySQL
     *
     * @param operationName
     * @param remotePeer
     * @return
     */
    public static AbstractSpan createExitSpan(String operationName, String remotePeer) {
        operationName = StringUtil.cut(operationName, OPERATION_NAME_THRESHOLD);
        AbstractTracerContext context = getOrCreate(operationName, false);
        return context.createExitSpan(operationName, remotePeer);
    }  
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
小结：


30、DataCarrier Buffer

Agent采集到的链路数据会先放到DataCarrier中，由消费者线程读取DataCarrier中的数据上报到OAP

1）、QueueBuffer
DataCarrier是使用Buffer作为数据存储，Buffer的底层接口是QueueBuffer，代码如下：

/**
 * Queue buffer interface.
 */
 public interface QueueBuffer<T> {
    /**
     * 保存数据到队列中
     * Save data into the queue;
     *
     * @param data to add.
     * @return true if saved
     */
     boolean save(T data);

    /**
     * 设置队列满时的处理策略
     * Set different strategy when queue is full.
     */
     void setStrategy(BufferStrategy strategy);

    /**
     * 队列中的数据放到consumeList中并清空队列
     * Obtain the existing data from the queue
     */
     void obtain(List<T> consumeList);

    int getBufferSize();
 }
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 26
 27
 BufferStrategy定义了队列满时的处理策略：

public enum BufferStrategy {
    /**
     * 阻塞,等待队列有空位置
     */
    BLOCKING,
    /**
     * 能放就放,不能放就算了
     */
    IF_POSSIBLE
}
1
2
3
4
5
6
7
8
9
10

QueueBuffer有两个实现Buffer和ArrayBlockingQueueBuffer

2）、Buffer
Buffer是一个环形队列，代码如下：

/**
 * 实现了环形队列
 * Self implementation ring queue.
 */
    public class Buffer<T> implements QueueBuffer<T> {
    private final Object[] buffer; // 队列数据都存储到数组中
    private BufferStrategy strategy; // 队列满时的处理策略
    private AtomicRangeInteger index; // 索引

    Buffer(int bufferSize, BufferStrategy strategy) {
        buffer = new Object[bufferSize];
        this.strategy = strategy;
        index = new AtomicRangeInteger(0, bufferSize);
    }

    @Override
    public void setStrategy(BufferStrategy strategy) {
        this.strategy = strategy;
    }

    @Override
    public boolean save(T data) {
        // 拿到队列下一个位置的下标
        int i = index.getAndIncrement();
        if (buffer[i] != null) {
            switch (strategy) {
                case IF_POSSIBLE:
                    return false;
                default:
            }
        }
        buffer[i] = data;
        return true;
    }

    @Override
    public int getBufferSize() {
        return buffer.length;
    }

    @Override
    public void obtain(List<T> consumeList) {
        this.obtain(consumeList, 0, buffer.length);
    }

    void obtain(List<T> consumeList, int start, int end) {
        for (int i = start; i < end; i++) {
            if (buffer[i] != null) {
                consumeList.add((T) buffer[i]);
                buffer[i] = null;
            }
        }
    }

}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
Buffer的数据结构如下图：


AtomicRangeInteger是队列的索引，代码如下：

public class AtomicRangeInteger extends Number implements Serializable {
    private static final long serialVersionUID = -4099792402691141643L;
    // 一个可以原子化操作数组某一个元素的数组封装
    private AtomicIntegerArray values;

    private static final int VALUE_OFFSET = 15;
    
    private int startValue;
    private int endValue;
    
    public AtomicRangeInteger(int startValue, int maxValue) {
        // 简单理解为,创建了一个长度为31的数组
        this.values = new AtomicIntegerArray(31);
        // 在values这个数组的下标为15(即第16个元素)的位置的值设置为指定值(默认为0)
        this.values.set(VALUE_OFFSET, startValue);
        this.startValue = startValue;
        this.endValue = maxValue - 1;
    }
    
    public final int getAndIncrement() {
        int next;
        do {
            next = this.values.incrementAndGet(VALUE_OFFSET);
            // 如果取到的next>endValue,就意味着下标越界了
            // 这时候需要通过CAS操作将values的第16个元素的值重置为startValue,即0
            if (next > endValue && this.values.compareAndSet(VALUE_OFFSET, next, startValue)) {
                return endValue;
            }
        }
        while (next > endValue);
    
        return next - 1;
    }
    
    public final int get() {
        return this.values.get(VALUE_OFFSET);
    }
    
    @Override
    public int intValue() {
        return this.values.get(VALUE_OFFSET);
    }
    
    @Override
    public long longValue() {
        return this.values.get(VALUE_OFFSET);
    }
    
    @Override
    public float floatValue() {
        return this.values.get(VALUE_OFFSET);
    }
    
    @Override
    public double doubleValue() {
        return this.values.get(VALUE_OFFSET);
    }
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
AtomicRangeInteger是使用JDK的AtomicIntegerArray实现的，AtomicRangeInteger初始化了一个长度为31的数组，使用数组最中间的元素（下标为15的元素）代表索引值，索引值初始值为0。getAndIncrement()方法中先对索引值+1，如果此时索引值>endValue就意味着下标越界了，这时候需要通过CAS操作将索引值重置为0，这样就实现了环形队列

AtomicRangeInteger为什么使用AtomicIntegerArray创建一个长度为31的数组？如果只是为了原子性操作完全可以使用AtomicInteger实现

SkyWalking之前也是使用AtomicInteger实现的，后面为了避免伪共享从而提高性能改为了AtomicIntegerArray

对应PR：https://github.com/apache/skywalking/pull/2930

伪共享相关文章：https://blog.csdn.net/qq_40378034/article/details/101383233

3）、ArrayBlockingQueueBuffer
ArrayBlockingQueueBuffer是使用JDK的ArrayBlockingQueue实现的，代码如下：

/**
 * The buffer implementation based on JDK ArrayBlockingQueue.
 * <p>
 * This implementation has better performance in server side. We are still trying to research whether this is suitable
 * for agent side, which is more sensitive about blocks.
 */
 public class ArrayBlockingQueueBuffer<T> implements QueueBuffer<T> {
    private BufferStrategy strategy;
    private ArrayBlockingQueue<T> queue;
    private int bufferSize;

    ArrayBlockingQueueBuffer(int bufferSize, BufferStrategy strategy) {
        this.strategy = strategy;
        this.queue = new ArrayBlockingQueue<T>(bufferSize);
        this.bufferSize = bufferSize;
    }

    @Override
    public boolean save(T data) {
        //only BufferStrategy.BLOCKING
        try {
            queue.put(data);
        } catch (InterruptedException e) {
            // Ignore the error
            return false;
        }
        return true;
    }

    @Override
    public void setStrategy(BufferStrategy strategy) {
        this.strategy = strategy;
    }

    @Override
    public void obtain(List<T> consumeList) {
        queue.drainTo(consumeList);
    }

    @Override
    public int getBufferSize() {
        return bufferSize;
    }
 }
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 26
 27
 28
 29
 30
 31
 32
 33
 34
 35
 36
 37
 38
 39
 40
 41
 42
 43
 44
 小结：


32、DataCarrier全解
1）、Channels
Channels中管理了多个Buffer，代码如下：

/**
 * Channels of Buffer It contains all buffer data which belongs to this channel. It supports several strategy when
 * buffer is full. The Default is BLOCKING <p> Created by wusheng on 2016/10/25.
 */
 public class Channels<T> {
    private final QueueBuffer<T>[] bufferChannels; // buffer数组
    private IDataPartitioner<T> dataPartitioner; // 数据分区器,确定每次操作哪个buffer
    private final BufferStrategy strategy;
    private final long size;

    public Channels(int channelSize, int bufferSize, IDataPartitioner<T> partitioner, BufferStrategy strategy) {
        this.dataPartitioner = partitioner;
        this.strategy = strategy;
        bufferChannels = new QueueBuffer[channelSize];
        for (int i = 0; i < channelSize; i++) {
            if (BufferStrategy.BLOCKING.equals(strategy)) {
                bufferChannels[i] = new ArrayBlockingQueueBuffer<>(bufferSize, strategy);
            } else {
                bufferChannels[i] = new Buffer<>(bufferSize, strategy);
            }
        }
        // noinspection PointlessArithmeticExpression
        size = 1L * channelSize * bufferSize; // it's not pointless, it prevents numeric overflow before assigning an integer to a long
    }

    public boolean save(T data) {
        // buffer的索引,即选择哪个buffer来存数据
        int index = dataPartitioner.partition(bufferChannels.length, data);
        int retryCountDown = 1;
        if (BufferStrategy.IF_POSSIBLE.equals(strategy)) {
            int maxRetryCount = dataPartitioner.maxRetryCount();
            if (maxRetryCount > 1) {
                retryCountDown = maxRetryCount;
            }
        }
        for (; retryCountDown > 0; retryCountDown--) {
            if (bufferChannels[index].save(data)) {
                return true;
            }
        }
        return false;
    }
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 26
 27
 28
 29
 30
 31
 32
 33
 34
 35
 36
 37
 38
 39
 40
 41
 42
 一个Channels中包含多个Buffer，数据结构如下图：


数据分区器IDataPartitioner接口代码如下：

public interface IDataPartitioner<T> {
    int partition(int total, T data);

    /**
     * @return an integer represents how many times should retry when {@link BufferStrategy#IF_POSSIBLE}.
     * <p>
     * Less or equal 1, means not support retry.
     */
    int maxRetryCount();
}
1
2
3
4
5
6
7
8
9
10

IDataPartitioner有两个实现SimpleRollingPartitioner和ProducerThreadPartitioner

SimpleRollingPartitioner分区是每次+1和total取模：

/**
 * use normal int to rolling.
 */
 public class SimpleRollingPartitioner<T> implements IDataPartitioner<T> {
    @SuppressWarnings("NonAtomicVolatileUpdate")
    private volatile int i = 0;

    @Override
    public int partition(int total, T data) {
        return Math.abs(i++ % total);
    }

    @Override
    public int maxRetryCount() {
        return 3;
    }
 }
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 ProducerThreadPartitioner分区是使用当前线程ID和total取模：

/**
 * use threadid % total to partition
 */
 public class ProducerThreadPartitioner<T> implements IDataPartitioner<T> {
    public ProducerThreadPartitioner() {
    }

    @Override
    public int partition(int total, T data) {
        return (int) Thread.currentThread().getId() % total;
    }

    @Override
    public int maxRetryCount() {
        return 1;
    }
 }
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 2）、消费者
 消费者读取DataCarrier中的数据上报到OAP，IConsumer是消费者的顶层接口：

public interface IConsumer<T> {
    void init();

    void consume(List<T> data);
    
    void onError(List<T> data, Throwable t);
    
    void onExit();
    
    /**
     * Notify the implementation, if there is nothing fetched from the queue. This could be used as a timer to trigger
     * reaction if the queue has no element.
     */
    default void nothingToConsume() {
        return;
    }
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
ConsumerThread代码如下：

public class ConsumerThread<T> extends Thread {
    private volatile boolean running;
    private IConsumer<T> consumer;
    private List<DataSource> dataSources;
    // 本次消费没有取到数据时,线程sleep的时间
    private long consumeCycle;

    ConsumerThread(String threadName, IConsumer<T> consumer, long consumeCycle) {
        super(threadName);
        this.consumer = consumer;
        running = false;
        dataSources = new ArrayList<DataSource>(1);
        this.consumeCycle = consumeCycle;
    }
    
    /**
     * add whole buffer to consume
     */
    void addDataSource(QueueBuffer<T> sourceBuffer) {
        this.dataSources.add(new DataSource(sourceBuffer));
    }
    
    @Override
    public void run() {
        running = true;
    
        final List<T> consumeList = new ArrayList<T>(1500);
        while (running) {
            if (!consume(consumeList)) {
                try {
                    // 没有消费到数据,线程sleep
                    Thread.sleep(consumeCycle);
                } catch (InterruptedException e) {
                }
            }
        }
    
        // consumer thread is going to stop
        // consume the last time
        consume(consumeList);
    
        consumer.onExit();
    }
    
    private boolean consume(List<T> consumeList) {
        for (DataSource dataSource : dataSources) {
            // 将buffer中的数据放到consumeList中,并清空buffer
            dataSource.obtain(consumeList);
        }
    
        if (!consumeList.isEmpty()) {
            try {
                // 调用消费者的消费逻辑
                consumer.consume(consumeList);
            } catch (Throwable t) {
                consumer.onError(consumeList, t);
            } finally {
                consumeList.clear();
            }
            return true;
        }
        consumer.nothingToConsume();
        return false;
    }
    
    void shutdown() {
        running = false;
    }
    
    /**
     * DataSource is a refer to {@link Buffer}.
     */
    class DataSource {
        private QueueBuffer<T> sourceBuffer;
    
        DataSource(QueueBuffer<T> sourceBuffer) {
            this.sourceBuffer = sourceBuffer;
        }
    
        void obtain(List<T> consumeList) {
            sourceBuffer.obtain(consumeList);
        }
    }
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
ConsumerThread的数据结构如下图：


一个ConsumerThread中包含多个DataSource，DataSource里包装了Buffer。同时一个ConsumerThread绑定了一个Consumer，Consumer会消费ConsumerThread中的DataSource

MultipleChannelsConsumer代表一个单消费者线程，但支持多个Channels和它们的消费者，代码如下：

/**
 * MultipleChannelsConsumer代表一个单消费者线程,但支持多个channels和它们的消费者
 * MultipleChannelsConsumer represent a single consumer thread, but support multiple channels with their {@link
 * IConsumer}s
 */
 public class MultipleChannelsConsumer extends Thread {
    private volatile boolean running;
    private volatile ArrayList<Group> consumeTargets;
    @SuppressWarnings("NonAtomicVolatileUpdate")
    private volatile long size;
    private final long consumeCycle;

    public MultipleChannelsConsumer(String threadName, long consumeCycle) {
        super(threadName);
        this.consumeTargets = new ArrayList<Group>();
        this.consumeCycle = consumeCycle;
    }

    @Override
    public void run() {
        running = true;

        final List consumeList = new ArrayList(2000);
        while (running) {
            boolean hasData = false;
            for (Group target : consumeTargets) {
                boolean consume = consume(target, consumeList);
                hasData = hasData || consume;
            }
     
            if (!hasData) {
                try {
                    Thread.sleep(consumeCycle);
                } catch (InterruptedException e) {
                }
            }
        }
     
        // consumer thread is going to stop
        // consume the last time
        for (Group target : consumeTargets) {
            consume(target, consumeList);
     
            target.consumer.onExit();
        }
    }

    private boolean consume(Group target, List consumeList) {
        // 遍历channels中的buffer,将buffer中的数据放到consumeList中,并清空buffer
        for (int i = 0; i < target.channels.getChannelSize(); i++) {
            QueueBuffer buffer = target.channels.getBuffer(i);
            buffer.obtain(consumeList);
        }

        if (!consumeList.isEmpty()) {
            try {
                // 调用消费者的消费逻辑
                target.consumer.consume(consumeList);
            } catch (Throwable t) {
                target.consumer.onError(consumeList, t);
            } finally {
                consumeList.clear();
            }
            return true;
        }
        target.consumer.nothingToConsume();
        return false;
    }

    /**
     * Add a new target channels.
     */
     public void addNewTarget(Channels channels, IConsumer consumer) {
        Group group = new Group(channels, consumer);
        // Recreate the new list to avoid change list while the list is used in consuming.
        ArrayList<Group> newList = new ArrayList<Group>();
        for (Group target : consumeTargets) {
            newList.add(target);
        }
        newList.add(group);
        consumeTargets = newList;
        size += channels.size();
     }

    public long size() {
        return size;
    }

    void shutdown() {
        running = false;
    }

    private static class Group {
        private Channels channels; // 一个channels对应多个buffer
        private IConsumer consumer; // consumer会消费channels中所有的buffer

        public Group(Channels channels, IConsumer consumer) {
            this.channels = channels;
            this.consumer = consumer;
        }
    }
 }
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 26
 27
 28
 29
 30
 31
 32
 33
 34
 35
 36
 37
 38
 39
 40
 41
 42
 43
 44
 45
 46
 47
 48
 49
 50
 51
 52
 53
 54
 55
 56
 57
 58
 59
 60
 61
 62
 63
 64
 65
 66
 67
 68
 69
 70
 71
 72
 73
 74
 75
 76
 77
 78
 79
 80
 81
 82
 83
 84
 85
 86
 87
 88
 89
 90
 91
 92
 93
 94
 95
 96
 97
 98
 99
 100
 101
 102
 Group的数据结构如下图：


一个Group中包含一个Consumer和一个Channels，一个Channels包含多个Buffer，Consumer会消费Channels中所有的Buffer

一个MultipleChannelsConsumer包含多个Group，实际上是管理多个Consumer以及它们对应的Buffer，数据结构如下图：


3）、消费者驱动
IDriver代码如下：

/**
 * The driver of consumer.
 */
 public interface IDriver {
    boolean isRunning(Channels channels);

    void close(Channels channels);

    void begin(Channels channels);
 }
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 IDriver实现关系如下图：


ConsumeDriver代码如下：

/**
 * Pool of consumers <p> Created by wusheng on 2016/10/25.
 * 一堆消费者线程拿着一堆buffer,按allocateBuffer2Thread()的策略进行分配消费
 */
 public class ConsumeDriver<T> implements IDriver {
    private boolean running;
    private ConsumerThread[] consumerThreads;
    private Channels<T> channels;
    private ReentrantLock lock;

    public ConsumeDriver(String name, Channels<T> channels, Class<? extends IConsumer<T>> consumerClass, int num,
        long consumeCycle) {
        this(channels, num);
        for (int i = 0; i < num; i++) {
            consumerThreads[i] = new ConsumerThread("DataCarrier." + name + ".Consumer." + i + ".Thread", getNewConsumerInstance(consumerClass), consumeCycle);
            consumerThreads[i].setDaemon(true);
        }
    }

    public ConsumeDriver(String name, Channels<T> channels, IConsumer<T> prototype, int num, long consumeCycle) {
        this(channels, num);
        prototype.init();
        for (int i = 0; i < num; i++) {
            consumerThreads[i] = new ConsumerThread("DataCarrier." + name + ".Consumer." + i + ".Thread", prototype, consumeCycle);
            consumerThreads[i].setDaemon(true);
        }

    }

    private ConsumeDriver(Channels<T> channels, int num) {
        running = false;
        this.channels = channels;
        consumerThreads = new ConsumerThread[num];
        lock = new ReentrantLock();
    }

    private IConsumer<T> getNewConsumerInstance(Class<? extends IConsumer<T>> consumerClass) {
        try {
            IConsumer<T> inst = consumerClass.getDeclaredConstructor().newInstance();
            inst.init();
            return inst;
        } catch (InstantiationException e) {
            throw new ConsumerCannotBeCreatedException(e);
        } catch (IllegalAccessException e) {
            throw new ConsumerCannotBeCreatedException(e);
        } catch (NoSuchMethodException e) {
            throw new ConsumerCannotBeCreatedException(e);
        } catch (InvocationTargetException e) {
            throw new ConsumerCannotBeCreatedException(e);
        }
    }

    @Override
    public void begin(Channels channels) {
        // begin只能调用一次
        if (running) {
            return;
        }
        lock.lock();
        try {
            this.allocateBuffer2Thread();
            for (ConsumerThread consumerThread : consumerThreads) {
                consumerThread.start();
            }
            running = true;
        } finally {
            lock.unlock();
        }
    }

    @Override
    public boolean isRunning(Channels channels) {
        return running;
    }

    private void allocateBuffer2Thread() {
        // buffer的数量
        int channelSize = this.channels.getChannelSize();
        /**
         * 因为channels里面有多个buffer,同时这里也有多个消费者线程
         * 这一步的操作就是将这些buffer分配给不同的消费者线程去消费
         *
         * if consumerThreads.length < channelSize
         * each consumer will process several channels.
         *
         * if consumerThreads.length == channelSize
         * each consumer will process one channel.
         *
         * if consumerThreads.length > channelSize
         * there will be some threads do nothing.
         */
        for (int channelIndex = 0; channelIndex < channelSize; channelIndex++) {
            // 消费者线程索引 = buffer的下标和消费者线程数取模
            int consumerIndex = channelIndex % consumerThreads.length;
            consumerThreads[consumerIndex].addDataSource(channels.getBuffer(channelIndex));
        }

    }

    @Override
    public void close(Channels channels) {
        lock.lock();
        try {
            this.running = false;
            for (ConsumerThread consumerThread : consumerThreads) {
                consumerThread.shutdown();
            }
        } finally {
            lock.unlock();
        }
    }
 }
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 26
 27
 28
 29
 30
 31
 32
 33
 34
 35
 36
 37
 38
 39
 40
 41
 42
 43
 44
 45
 46
 47
 48
 49
 50
 51
 52
 53
 54
 55
 56
 57
 58
 59
 60
 61
 62
 63
 64
 65
 66
 67
 68
 69
 70
 71
 72
 73
 74
 75
 76
 77
 78
 79
 80
 81
 82
 83
 84
 85
 86
 87
 88
 89
 90
 91
 92
 93
 94
 95
 96
 97
 98
 99
 100
 101
 102
 103
 104
 105
 106
 107
 108
 109
 110
 111
 112
 一个ConsumeDriver包含多个ConsumerThread

/**
 * BulkConsumePool works for consuming data from multiple channels(DataCarrier instances), with multiple {@link
 * MultipleChannelsConsumer}s.
 * <p>
 * In typical case, the number of {@link MultipleChannelsConsumer} should be less than the number of channels.
 */
 public class BulkConsumePool implements ConsumerPool {
    private List<MultipleChannelsConsumer> allConsumers;
    private volatile boolean isStarted = false;

    public BulkConsumePool(String name, int size, long consumeCycle) {
        size = EnvUtil.getInt(name + "_THREAD", size);
        allConsumers = new ArrayList<MultipleChannelsConsumer>(size);
        for (int i = 0; i < size; i++) {
            MultipleChannelsConsumer multipleChannelsConsumer = new MultipleChannelsConsumer("DataCarrier." + name + ".BulkConsumePool." + i + ".Thread", consumeCycle);
            multipleChannelsConsumer.setDaemon(true);
            allConsumers.add(multipleChannelsConsumer);
        }
    }

    @Override
    synchronized public void add(String name, Channels channels, IConsumer consumer) {
        MultipleChannelsConsumer multipleChannelsConsumer = getLowestPayload();
        multipleChannelsConsumer.addNewTarget(channels, consumer);
    }

    /**
     * 拿到负载最低的消费者线程
     * Get the lowest payload consumer thread based on current allocate status.
     *
     * @return the lowest consumer.
     */
     private MultipleChannelsConsumer getLowestPayload() {
        MultipleChannelsConsumer winner = allConsumers.get(0);
        for (int i = 1; i < allConsumers.size(); i++) {
            MultipleChannelsConsumer option = allConsumers.get(i);
            // 比较consumer的size(consumer中buffer的总数)
            if (option.size() < winner.size()) {
                winner = option;
            }
        }
        return winner;
     }

    /**
     *
     */
    @Override
    public boolean isRunning(Channels channels) {
        return isStarted;
    }

    @Override
    public void close(Channels channels) {
        for (MultipleChannelsConsumer consumer : allConsumers) {
            consumer.shutdown();
        }
    }

    @Override
    public void begin(Channels channels) {
        if (isStarted) {
            return;
        }
        for (MultipleChannelsConsumer consumer : allConsumers) {
            consumer.start();
        }
        isStarted = true;
    }

    /**
     * The creator for {@link BulkConsumePool}.
     */
     public static class Creator implements Callable<ConsumerPool> {
        private String name;
        private int size;
        private long consumeCycle;

        public Creator(String name, int poolSize, long consumeCycle) {
            this.name = name;
            this.size = poolSize;
            this.consumeCycle = consumeCycle;
        }

        @Override
        public ConsumerPool call() {
            return new BulkConsumePool(name, size, consumeCycle);
        }

        public static int recommendMaxSize() {
            return Runtime.getRuntime().availableProcessors() * 2;
        }
     }
  }
  1
  2
  3
  4
  5
  6
  7
  8
  9
  10
  11
  12
  13
  14
  15
  16
  17
  18
  19
  20
  21
  22
  23
  24
  25
  26
  27
  28
  29
  30
  31
  32
  33
  34
  35
  36
  37
  38
  39
  40
  41
  42
  43
  44
  45
  46
  47
  48
  49
  50
  51
  52
  53
  54
  55
  56
  57
  58
  59
  60
  61
  62
  63
  64
  65
  66
  67
  68
  69
  70
  71
  72
  73
  74
  75
  76
  77
  78
  79
  80
  81
  82
  83
  84
  85
  86
  87
  88
  89
  90
  91
  92
  93
  94
  一个BulkConsumePool包含多个MultipleChannelsConsumer

小结：


33、链路数据发送到OAP
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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
TracingContext的finish()方法关闭当前TraceSegment后，会调用ListenerManager的notifyFinish()方法传入当前关闭的TraceSegment。ListenerManager的notifyFinish()方法会迭代所有注册的TracingContextListener调用它们的afterFinished()方法

TraceSegmentServiceClient实现了TracingContextListener接口，并向ListenerManager注册了自己，afterFinished()方法代码如下：

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
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 afterFinished()方法中会将TraceSegment放到DataCarrier中

TraceSegmentServiceClient也实现了IConsumer接口，消费DataCarrier中的TraceSegment数据，consume()方法代码如下：

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
小结：


34、链路追踪案例

上图是SkyWalking UI中展示的一条链路，这条链路的流程如下：

入口是demo1的/api/demo1接口，demo1先调用MySQL，然后通过HttpClient调用demo2的/api/demo2接口
应用demo2的/api/demo2接口直接返回响应
demo1收到demo2的/api/demo2接口的响应后返回，整条链路结束
下面来分析下SkyWalking Agent对这条链路的追踪过程：

1）、demo1入口接收请求
请求到达demo1后，走到Tomcat，Tomcat插件（TomcatInvokeInterceptor）创建EntrySpan（ContextManager.createEntrySpan()）。因为ThreadLocal中的TracingContext为空，会先创建TracingContext然后放到ThreadLocal中，然后使用TracingContext创建EntrySpan（TracingContext.createEntrySpan()）。TracingContext中activeSpanStack为空，创建了第一个EntrySpan（spanId=0，parentSpanId=-1）并入栈到activeSpanStack中

Tomcat插件创建的EntrySpan入栈后：


请求走到SpringMVC后，SpringMVC插件（AbstractMethodInterceptor）使用ThreadLocal中的TracingContext创建EntrySpan。这时TracingContext中activeSpanStack栈顶的Span是EntrySpan，所以直接复用，并覆盖了Tomcat插件记录的信息

SpringMVC插件复用Tomcat插件创建的EntrySpan：


2）、demo1调用MySQL
demo1调用MySQL，MySQL插件（PreparedStatementExecuteMethodsInterceptor）使用ThreadLocal中的TracingContext创建ExitSpan。拿到TracingContext中activeSpanStack栈顶的Span（EntrySpan#SpringMVC）作为parentSpan，创建ExitSpan（spanId=1，parentSpanId=0）并入栈到activeSpanStack中

MySQL插件创建的ExitSpan入栈后：


访问MySQL操作结束后，MySQL插件的后置处理使用ThreadLocal中的TracingContext stopSpan（TracingContext.stopSpan()）。TracingContext中activeSpanStack栈顶的Span出栈，放到TracingContext中TraceSegment的spans集合中（执行完的Span会放到TraceSegment的spans集合中，等待后续发送到OAP）

MySQL插件创建的ExitSpan出栈后：


3）、demo1调用demo2接口
demo1通过HttpClient调用demo2接口，HttpClient插件（HttpClientExecuteInterceptor）使用ThreadLocal中的TracingContext创建ExitSpan。拿到TracingContext中activeSpanStack栈顶的Span（EntrySpan#SpringMVC）作为parentSpan，创建ExitSpan（spanId=2，parentSpanId=0）并入栈到activeSpanStack中

HttpClient插件创建的ExitSpan入栈后：


创建完ExitSpan后，调用TracingContext.inject()给ContextCarrier赋值，包括TraceId、TraceSegmentId、SpanId（当前ExitSpan的Id）、ParentService、ParentServiceInstance等信息。然后会把ContextCarrier中的数据放到Http请求头中，通过这种方式让链路信息传递下去

demo2接收到demo1的请求后，创建EntrySpan的流程和demo1入口接收请求一致，这里会多一步，就是从Http请求头中拿到demo1传递的链路信息赋值给ContextCarrier，调用TracingContext.extract()绑定当前TraceSegment的traceSegmentRef、traceId以及EntrySpan的ref

demo2的响应返回后，demo1中插件后置处理依次调用TracingContext.stopSpan()，TracingContext中activeSpanStack中的Span依次出栈，最后activeSpanStack栈为空时，TracingContext结束

上述这条链路如下图所示：



参考：

SkyWalking8.7.0源码分析（如果你对SkyWalking Agent源码感兴趣的话，强烈建议看下该教程）
