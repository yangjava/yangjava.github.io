---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码ContextManager

## 上下文适配器ContextManager
TraceSegment及其所包含的Span都在同一个线程内，ContextManager使用ThreadLocal来管理TraceSegment的上下文（也就是AbstractTracerContext）

ContextManager中getOrCreate()方法：
```
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

```
getOrCreate()方法处理逻辑如下：

- 从ThreadLocal中获取AbstractTracerContext，如果有就返回，没有就新建
- 如果operationName为空创建IgnoredTracerContext
- 否则调用ContextManagerExtendService的createTraceContext()方法创建AbstractTracerContext，并设置到ThreadLocal中

ContextManagerExtendService的createTraceContext()方法代码如下：
```
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

```
createTraceContext()方法处理逻辑如下：

- 如果OAP挂了不采样且网络连接断开，创建IgnoredTracerContext
- 如果operationName的后缀名在ignoreSuffixArray中（指定哪些后缀的请求不需要追踪），创建IgnoredTracerContext
- 如果是强制采样或尝试采样（SamplingService的trySampling()方法）返回true，创建TracingContext，否则创建IgnoredTracerContext

ContextManager中createEntrySpan()方法：
```
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

```
createEntrySpan()方法处理逻辑如下：

- 如果ContextCarrier不为空，强制采样，获取或创建TracingContext，创建EntrySpan，从ContextCarrier将数据提取出来放到TracingContext中
- 如果ContextCarrier为空，不需要强制采样，根据采样率来决定当前链路是否要采样

当创建EntrySpan时有两种情况：

- 请求刚刚进来处于链路的第一个TraceSegment上，如上图左边的TraceSegment，此时不需要强制采样，根据采样率来决定当前链路是否要采样
- 如上图右边的TraceSegment，左边TraceSegment的ExitSpan调用了右边的TraceSegment，上一个TraceSegment的数据需要传递到下一个TraceSegment，下游调用extract()方法从ContextCarrier将数据提取出来放到TracingContext中。此时一定要强制采样，因为链路中的前置TraceSegment已经存在，如果不强制采样，尝试采样（SamplingService的trySampling()方法）返回false，链路就断开了

ContextManager中创建LocalSpan和ExitSpan的方法：
```
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

```




