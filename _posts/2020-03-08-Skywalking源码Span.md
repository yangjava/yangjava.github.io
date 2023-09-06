---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码Span

## Span基本概念
Span的最顶层实现为AsyncSpan，代码如下：
```
/**
 * Span能够使用这些API来激活并扩展它的生命周期跨线程
 * Span could use these APIs to active and extend its lift cycle across thread.
 * <p>
 * 这个典型的使用是在异步插件,尤其是RPC插件
 * This is typical used in async plugin, especially RPC plugins.
 */
public interface AsyncSpan {
    /**
     * The span finish at current tracing context, but the current span is still alive, until {@link #asyncFinish}
     * called.
     * <p>
     * This method must be called
     * <p>
     * 1. In original thread(tracing context). 2. Current span is active span.
     * <p>
     * During alive, tags, logs and attributes of the span could be changed, in any thread.
     * <p>
     * The execution times of {@link #prepareForAsync} and {@link #asyncFinish()} must match.
     *
     * @return the current span
     */
    AbstractSpan prepareForAsync();

    /**
     * Notify the span, it could be finished.
     * <p>
     * The execution times of {@link #prepareForAsync} and {@link #asyncFinish()} must match.
     *
     * @return the current span
     */
    AbstractSpan asyncFinish();
}

```

AbstractSpan继承了AsyncSpan，代码如下：
```
/**
 * AbstractSpan定义了Span的骨架
 * The <code>AbstractSpan</code> represents the span's skeleton, which contains all open methods.
 */
public interface AbstractSpan extends AsyncSpan {
    /**
     * Set the component id, which defines in {@link ComponentsDefine}
     * 指定当前span表示的操作发生在哪个插件上
     *
     * @return the span for chaining.
     */
    AbstractSpan setComponent(Component component);

    /**
     * 指定当前span表示的操作所在的插件属于哪一种skywalking划分的类型
     *
     * @param layer
     * @return
     */
    AbstractSpan setLayer(SpanLayer layer);

```
setLayer()用于指定当前Span表示的操作所在的插件属于哪一种SkyWalking划分的类型，在SpanLayer中定义了五种类型：
```
public enum SpanLayer {
    DB(1), RPC_FRAMEWORK(2), HTTP(3), MQ(4), CACHE(5);
```

```
public  AbstractSpan extends AsyncSpan {
  
    /**
     * span上打标签
     */
    AbstractSpan tag(AbstractTag<?> tag, String value);

    /**
     * 记录异常,时间使用当前本地时间
     * Record an exception event of the current walltime timestamp.
     * wallTime:挂钟时间,本地时间
     * serverTime:服务器时间
     *
     * @param t any subclass of {@link Throwable}, which occurs in this span.
     * @return the Span, for chaining
     */
    AbstractSpan log(Throwable t);
  
    /**
     * 是否是entry span
     * @return true if the actual span is an entry span.
     */
    boolean isEntry();

    /**
     * 是否是exit span
     * @return true if the actual span is an exit span.
     */
    boolean isExit();

    /**
     * 记录指定时间发生的事件
     * Record an event at a specific timestamp.
     *
     * @param timestamp The explicit timestamp for the log record.
     * @param event     the events
     * @return the Span, for chaining
     */
    AbstractSpan log(long timestamp, Map<String, ?> event);

    /**
     * Sets the string name for the logical operation this span represents.
     * 如果当前span的操作是:
     *    一个http请求,那么operationName就是请求的url
     *    一条sql语句,那么operationName就是sql
     *    一个redis操作,那么operationName就是redis命令
     *
     * @return this Span instance, for chaining
     */
    AbstractSpan setOperationName(String operationName);
  
    /**
     * Start a span.
     *
     * @return this Span instance, for chaining
     */
    AbstractSpan start();

    /**
     * Get the id of span
     *
     * @return id value.
     */
    int getSpanId();

    String getOperationName();

    /**
     * Reference other trace segment.
     *
     * @param ref segment ref
     */
    void ref(TraceSegmentRef ref);

    AbstractSpan start(long startTime);

    /**
     * 什么叫peer,就是对端地址
     * 一个请求可能跨多个进程,操作多种中间件,那么每一次RPC,对面的服务的地址就是remotePeer
     *                                    每一次中间件的操作,中间件的地址就是remotePeer
     *
     * @param remotePeer
     * @return
     */
    AbstractSpan setPeer(String remotePeer);  

```

## Span完整模型
抽象类AbstractTracingSpan实现了AbstractSpan接口，代码如下：
```
/**
 * The <code>AbstractTracingSpan</code> represents a group of {@link AbstractSpan} implementations, which belongs a real
 * distributed trace.
 */
public abstract class AbstractTracingSpan implements AbstractSpan {
    /**
     * span id从0开始
     * Span id starts from 0.
     */
    protected int spanId;
    /**
     * parent span id从0开始.-1代表没有parent span
     * Parent span id starts from 0. -1 means no parent span.
     */
    protected int parentSpanId;
    /**
     * span上的tag
     */
    protected List<TagValuePair> tags;
    protected String operationName;
    protected SpanLayer layer;
    /**
     * The span has been tagged in async mode, required async stop to finish.
     * 表示当前异步操作是否已经开始
     */
    protected volatile boolean isInAsyncMode = false;
    /**
     * The flag represents whether the span has been async stopped
     * 表示当前异步操作是否已经结束
     */
    private volatile boolean isAsyncStopped = false;

    /**
     * The context to which the span belongs
     * TracingContext用于管理一条链路上的segment和span
     */
    protected final TracingContext owner;

    /**
     * The start time of this Span.
     */
    protected long startTime;
    /**
     * The end time of this Span.
     */
    protected long endTime;
    /**
     * Error has occurred in the scope of span.
     */
    protected boolean errorOccurred = false;

    protected int componentId = 0;

    /**
     * Log is a concept from OpenTracing spec. https://github.com/opentracing/specification/blob/master/specification.md#log-structured-data
     */
    protected List<LogDataEntity> logs;

    /**
     * The refs of parent trace segments, except the primary one. For most RPC call, {@link #refs} contains only one
     * element, but if this segment is a start span of batch process, the segment faces multi parents, at this moment,
     * we use this {@link #refs} to link them.
     * 用于当前span指定自己所在的segment的前一个segment,除非这个span所在的segment是整条链路上的第一个segment
     */
    protected List<TraceSegmentRef> refs;
  
    /**
     * Finish the active Span. When it is finished, it will be archived by the given {@link TraceSegment}, which owners
     * it.
     * span结束的时候,添加到TraceSegment的spans中
     * @param owner of the Span.
     */
    public boolean finish(TraceSegment owner) {
        this.endTime = System.currentTimeMillis();
        owner.archive(this);
        return true;
    }  

```
SkyWalking中Trace数据模型的实现

一个Trace由多个TraceSegment组成，TraceSegment使用TraceSegmentRef指向它的上一个TraceSegment。每个TraceSegment中有多个Span，每个Span都有spanId和parentSpanId，spanId从0开始，parentSpanId指向上一个span的Id。

一个TraceSegment中第一个创建的Span叫EntrySpan，调用的本地方法对应LocalSpan，离开当前Segment对应ExitSpan。每个Span都有一个refs，每个TraceSegment的第一个Span的refs会指向它所在TraceSegment的上一个TraceSegment。

## StackBasedTracingSpan
StackBasedTracingSpan是一个内部具有栈结构的Span，它可以启动和关闭多次在一个类似栈的执行流程中

在看StackBasedTracingSpan源码之前，我们先来看下StackBasedTracingSpan的工作原理：

假设有一个应用部署在Tomcat上，使用SpringMVC提供一个getUser()的Controller方法，getUser()方法直接返回不会调用其他的第三方

在这样一个单体结构中，只会有一个TraceSegment，TraceSegment中会有一个EntrySpan

请求进来后，走到Tomcat，SkyWalking的Tomcat插件会尝试创建EntrySpan，如果发现自己是这个请求到达后第一个工作的插件就会创建EntrySpan，如果不是第一个就会复用之前插件创建的EntrySpan。Tomcat插件创建EntrySpan，并会在Span上记录tags、logs、component、layer等信息，代码如下：
```
public class TomcatInvokeInterceptor implements InstanceMethodsAroundInterceptor {

    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                             MethodInterceptResult result) throws Throwable {
        Request request = (Request) allArguments[0];
        ContextCarrier contextCarrier = new ContextCarrier();

        CarrierItem next = contextCarrier.items();
        while (next.hasNext()) {
            next = next.next();
            next.setHeadValue(request.getHeader(next.getHeadKey()));
        }

        // 创建EntrySpan
        AbstractSpan span = ContextManager.createEntrySpan(request.getRequestURI(), contextCarrier);
        Tags.URL.set(span, request.getRequestURL().toString());
        Tags.HTTP.METHOD.set(span, request.getMethod());
        span.setComponent(ComponentsDefine.TOMCAT);
        SpanLayer.asHttp(span);

        if (TomcatPluginConfig.Plugin.Tomcat.COLLECT_HTTP_PARAMS) {
            collectHttpParam(request, span);
        }
    }
```

请求经过Tomcat后交给SpringMVC，SpringMVC插件也会创建EntrySpan，代码如下：
```
public abstract class AbstractMethodInterceptor implements InstanceMethodsAroundInterceptor {
  
    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                             MethodInterceptResult result) throws Throwable {

        Boolean forwardRequestFlag = (Boolean) ContextManager.getRuntimeContext().get(FORWARD_REQUEST_FLAG);
        /**
         * Spring MVC plugin do nothing if current request is forward request.
         * Ref: https://github.com/apache/skywalking/pull/1325
         */
        if (forwardRequestFlag != null && forwardRequestFlag) {
            return;
        }

        String operationName;
        if (SpringMVCPluginConfig.Plugin.SpringMVC.USE_QUALIFIED_NAME_AS_ENDPOINT_NAME) {
            operationName = MethodUtil.generateOperationName(method);
        } else {
            EnhanceRequireObjectCache pathMappingCache = (EnhanceRequireObjectCache) objInst.getSkyWalkingDynamicField();
            String requestURL = pathMappingCache.findPathMapping(method);
            if (requestURL == null) {
                requestURL = getRequestURL(method);
                pathMappingCache.addPathMapping(method, requestURL);
                requestURL = pathMappingCache.findPathMapping(method);
            }
            operationName = getAcceptedMethodTypes(method) + requestURL;
        }

        Object request = ContextManager.getRuntimeContext().get(REQUEST_KEY_IN_RUNTIME_CONTEXT);

        if (request != null) {
            StackDepth stackDepth = (StackDepth) ContextManager.getRuntimeContext().get(CONTROLLER_METHOD_STACK_DEPTH);

            if (stackDepth == null) {
                final ContextCarrier contextCarrier = new ContextCarrier();

                if (IN_SERVLET_CONTAINER && HttpServletRequest.class.isAssignableFrom(request.getClass())) {
                    final HttpServletRequest httpServletRequest = (HttpServletRequest) request;
                    CarrierItem next = contextCarrier.items();
                    while (next.hasNext()) {
                        next = next.next();
                        next.setHeadValue(httpServletRequest.getHeader(next.getHeadKey()));
                    }

                    // 创建EntrySpan
                    AbstractSpan span = ContextManager.createEntrySpan(operationName, contextCarrier);
                    Tags.URL.set(span, httpServletRequest.getRequestURL().toString());
                    Tags.HTTP.METHOD.set(span, httpServletRequest.getMethod());
                    span.setComponent(ComponentsDefine.SPRING_MVC_ANNOTATION);
                    SpanLayer.asHttp(span);

                    if (SpringMVCPluginConfig.Plugin.SpringMVC.COLLECT_HTTP_PARAMS) {
                        RequestUtil.collectHttpParam(httpServletRequest, span);
                    }

                    if (!CollectionUtil.isEmpty(SpringMVCPluginConfig.Plugin.Http.INCLUDE_HTTP_HEADERS)) {
                        RequestUtil.collectHttpHeaders(httpServletRequest, span);
                    }
                } else if (ServerHttpRequest.class.isAssignableFrom(request.getClass())) {
                    final ServerHttpRequest serverHttpRequest = (ServerHttpRequest) request;
                    CarrierItem next = contextCarrier.items();
                    while (next.hasNext()) {
                        next = next.next();
                        next.setHeadValue(serverHttpRequest.getHeaders().getFirst(next.getHeadKey()));
                    }

                    // 创建EntrySpan
                    AbstractSpan span = ContextManager.createEntrySpan(operationName, contextCarrier);
                    Tags.URL.set(span, serverHttpRequest.getURI().toString());
                    Tags.HTTP.METHOD.set(span, serverHttpRequest.getMethodValue());
                    span.setComponent(ComponentsDefine.SPRING_MVC_ANNOTATION);
                    SpanLayer.asHttp(span);

                    if (SpringMVCPluginConfig.Plugin.SpringMVC.COLLECT_HTTP_PARAMS) {
                        RequestUtil.collectHttpParam(serverHttpRequest, span);
                    }

                    if (!CollectionUtil.isEmpty(SpringMVCPluginConfig.Plugin.Http.INCLUDE_HTTP_HEADERS)) {
                        RequestUtil.collectHttpHeaders(serverHttpRequest, span);
                    }
                } else {
                    throw new IllegalStateException("this line should not be reached");
                }

                stackDepth = new StackDepth();
                ContextManager.getRuntimeContext().put(CONTROLLER_METHOD_STACK_DEPTH, stackDepth);
            } else {
                AbstractSpan span = ContextManager.createLocalSpan(buildOperationName(objInst, method));
                span.setComponent(ComponentsDefine.SPRING_MVC_ANNOTATION);
            }

            stackDepth.increment();
        }
    }

```
Tomcat已经创建了EntrySpan，SpringMVC就不能再创建EntrySpan了，SpringMVC会复用Tomcat创建的EntrySpan。Tomcat已经在Span上记录tags、logs、component、layer等信息，SpringMVC这里会覆盖掉之前Tomcat在Span上记录的信息

EntrySpan就是使用StackBasedTracingSpan这种基于栈的Span来实现的，EntrySpan中有两个属性：当前栈深stackDepth和当前最大栈深currentMaxDepth

Tomcat创建EntrySpan，EntrySpan中当前栈深=1，当前最大栈深=1

SpringMVC复用Tomcat创建的EntrySpan，会把当前栈深和当前最大栈深都+1，此时当前栈深=2，当前最大栈深=2

当getUser()方法执行完后，首先返回到SpringMVC，会把当前栈深-1，当前最大栈深是只增不减的，此时当前栈深=1，当前最大栈深=2

当返回到Tomcat时，当前栈深-1，此时当前栈深=0，当前最大栈深=2，当前栈深=0时，就代表EntrySpan出栈了

如何判断当前EntrySpan是复用前面的呢？只需要判断currentMaxDepth不等于1就是复用前面的EntrySpan，如果等于1就是当前插件创建的EntrySpan。记录Span信息的时候都是请求进来EntrySpan入栈的流程，只要stackDepth=currentMaxDepth时就是请求进来的流程，所以只有stackDepth=currentMaxDepth时才允许记录Span的信息

### EntrySpan有如下几个特性：

在一个TraceSegment里面只能存在一个EntrySpan
后面的插件复用前面插件创建的EntrySpan时会覆盖掉前面插件设置的Span信息
EntrySpan记录的信息永远是最靠近服务提供侧的信息

EntrySpan和ExitSpan都是通过StackBasedTracingSpan来实现的，继承关系如下：

StackBasedTracingSpan中包含stackDepth属性，代码如下：
```
/**
 * StackBasedTracingSpan是一个内部具有栈结构的Span
 * The <code>StackBasedTracingSpan</code> represents a span with an inside stack construction.
 * <p>
 * 这种类型的Span可以启动和关闭多次在一个类似栈的执行流程中
 * This kind of span can start and finish multi times in a stack-like invoke line.
 */
public abstract class StackBasedTracingSpan extends AbstractTracingSpan {
    protected int stackDepth;
  
    @Override
    public boolean finish(TraceSegment owner) {
        if (--stackDepth == 0) {
            return super.finish(owner);
        } else {
            return false;
        }
    }  

```
finish()方法中，当stackDepth等于0时，栈就空了，就代表EntrySpan结束了

EntrySpan中包含currentMaxDepth属性，代码如下：
```
/**
 * The <code>EntrySpan</code> represents a service provider point, such as Tomcat server entrance.
 * <p>
 * It is a start point of {@link TraceSegment}, even in a complex application, there maybe have multi-layer entry point,
 * the <code>EntrySpan</code> only represents the first one.
 * <p>
 * But with the last <code>EntrySpan</code>'s tags and logs, which have more details about a service provider.
 * <p>
 * Such as: Tomcat Embed - Dubbox The <code>EntrySpan</code> represents the Dubbox span.
 */
public class EntrySpan extends StackBasedTracingSpan {

    private int currentMaxDepth;

    public EntrySpan(int spanId, int parentSpanId, String operationName, TracingContext owner) {
        super(spanId, parentSpanId, operationName, owner);
        this.currentMaxDepth = 0;
    }

    /**
     * Set the {@link #startTime}, when the first start, which means the first service provided.
     * EntrySpan只会由第一个插件创建,但是后面的插件复用EntrySpan时都要来调用一次start方法
     * 因为每一个插件都认为自己是第一个创建这个EntrySpan的
     */
    @Override
    public EntrySpan start() {
        // currentMaxDepth = stackDepth = 1时,才调用start方法记录启动时间
        if ((currentMaxDepth = ++stackDepth) == 1) {
            super.start();
        }
        // 复用span时清空之前插件记录的span信息
        clearWhenRestart();
        return this;
    }

    @Override
    public EntrySpan tag(String key, String value) {
        // stackDepth = currentMaxDepth时,才记录span信息
        if (stackDepth == currentMaxDepth || isInAsyncMode) {
            super.tag(key, value);
        }
        return this;
    }

    @Override
    public AbstractTracingSpan setLayer(SpanLayer layer) {
        if (stackDepth == currentMaxDepth || isInAsyncMode) {
            return super.setLayer(layer);
        } else {
            return this;
        }
    }

    @Override
    public AbstractTracingSpan setComponent(Component component) {
        if (stackDepth == currentMaxDepth || isInAsyncMode) {
            return super.setComponent(component);
        } else {
            return this;
        }
    }

    @Override
    public AbstractTracingSpan setOperationName(String operationName) {
        if (stackDepth == currentMaxDepth || isInAsyncMode) {
            return super.setOperationName(operationName);
        } else {
            return this;
        }
    }

    @Override
    public EntrySpan log(Throwable t) {
        super.log(t);
        return this;
    }

    @Override
    public boolean isEntry() {
        return true;
    }

    @Override
    public boolean isExit() {
        return false;
    }

    private void clearWhenRestart() {
        this.componentId = DictionaryUtil.nullValue();
        this.layer = null;
        this.logs = null;
        this.tags = null;
    }
}

```

## ExitSpan
ExitSpan代表服务消费侧，比如Feign、Okhttp。ExitSpan是链路中一个退出的点或者离开的Span。在一个RPC调用中，会有多层退出的点，而ExitSpan永远表示第一个。比如，Dubbox中使用HttpComponent发起远程调用。ExitSpan表示Dubbox的Span，并忽略HttpComponent的Span信息

EntrySpan和ExitSpan的区别就在于：

EntrySpan记录的是更靠近服务这一侧的信息
ExitSpan记录的是更靠近消费这一侧的信息

ExitSpan代码如下：
```
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

```
如上图，假设有一个应用部署在Tomcat上，使用SpringMVC提供一个getUser()的Controller方法，getUser()方法会在调用Redis和MySQL，对于这样一个流程的TraceSegment是怎样的？

Tomcat一进来就会创建EntrySpan，SpringMVC会复用Tomcat创建的EntrySpan。当访问Redis时会创建一个ExitSpan，peer会记录Redis地址

ExitSpan不要把理解为TraceSegment的结束，可以理解为离开当前TraceSegment的操作。当访问MySQL时也会创建一个ExitSpan，peer会记录MySQL地址。这里因为访问Redis和访问MySQL并不是嵌套关系，所以并不复用前面的ExitSpan

注意：

所谓ExitSpan和EntrySpan一样采用复用的机制，前提是在插件嵌套的情况下
多个ExitSpan不存在嵌套关系，是平行存在的时候，是允许同时存在多个ExitSpan
把ExitSpan简单理解为离开当前进程/线程的操作
TraceSegment里不一定非要有ExitSpan


## LocalSpan
LocalSpan继承关系如下：

LocalSpan代码如下：
```
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

```












