---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码异步链路

## 异步代码
不知道你是不是也出现过，明明打了@Trace，但是死活从http请求的链路进来看不到，反而，它还独自成一条链路，这样一来，根本连不成一个链路来追踪问题了。

此处，用了线程池去异步执行一个实现Runnable接口的类。
```
   @Trace
    public void trace() throws InterruptedException {
        Thread.sleep(10);
        doNothing();
        executor.submit(new MyRunnable());
    }
```

以及该接口具体实现
```
public class MyRunnable implements Runnable{

    @Override
    public void run() {
        System.out.println(TraceContext.traceId());
        doNothing();
    }

    @Trace
    private void doNothing(){
        return;
    }
}
```

链路断开示意图
理想中，我们自然也是希望异步线程中@Trace加注的方法也进入对应的链路，但是很遗憾，链路断成两条了：

我们会发现，链路仍然有两条存在，但是，/trace/local这个http请求中打印出来多了两个Span，一个是异步线程，一个是添加了@Trace的自定义的方法。而且，这两条链路的链路流水号是同一个。

## 实现方式
@TraceCrossThread
@Async
apm-jdk-threading-plugin

## @TraceCrossThread
@TraceCrossThread为skywalking提供的工具包中的注解，你可以认为其有一定的侵入性。使用方式则是在那个线程类上加上该注解，在类加载时，其构造方法会被代理agent做一次增强，具体的源码下一节说。

```
@TraceCrossThread
public class MyRunnable implements Runnable{

    @Override
    public void run() {
        System.out.println(TraceContext.traceId());
        doNothing();

    }
}
```

@Async
@Async为spring-context包下的注解，对于java服务端来说，这个注解基本可以算无侵入了，毕竟是spring体系。不过使用方式和平时需要实现一个Runnable接口或继承Thread类差别挺大，个人觉得方便，但是在古董项目中、亦或是说服那些一年学习经验十年使用经验的老古董来说，推广起来还是比较头疼的。当然，如果是一个巨石的老项目，不推荐这样来改造了。
```
@Service
public class TraceService implements InitializingBean {

    @Trace
    @Async
    public void traceAsync() throws InterruptedException {
        Thread.sleep(10);
        doNothing();

    }

    @Trace
    public void trace() throws InterruptedException {
        Thread.sleep(10);
        doNothing();
        executor.submit(new MyRunnable());
    }
}
```

apm-jdk-threading-plugin
apm-jdk-threading-plugin这是官方提供的一个插件，是真正的无侵入了，使用方式也很简单，这个插件位于${skywalking_dir}/agent/bootstrap-plugins目录下，我们需要做的就是将其复制到${skywalking_dir}/agent/plugins目录下即可。

另外，还需要对代理的配置进行修改${skywalking_dir}/agent/config/agent.config，告诉代理对那些包下的线程池进行增强,在其底部添加这样的配置：
```
jdkthreading.threading_class_prefixes=com.aaa.bbb,com.bbb.ccc
```
当有多个包的时候，可以用英文的逗号进行分隔。


无返回异步
RunnableWrapper.of()将lambda包裹即可。
```
   ExecutorService executorService = Executors.newFixedThreadPool(1);
    executorService.execute(RunnableWrapper.of(new Runnable() {
        @Override public void run() {
            //your code
        }
    }));
```

带返回异步
CallableWrapper.of()将lambda包裹即可
```
    ExecutorService executorService = Executors.newFixedThreadPool(1);
    executorService.submit(CallableWrapper.of(new Callable<String>() {
        @Override public String call() throws Exception {
            return null;
        }
    }));
```

Supplier异步
SupplierWrapper.of()将lambda包裹即可
```
   CompletableFuture.supplyAsync(SupplierWrapper.of(()->{
            return "SupplierWrapper";
    })).thenAccept(System.out::println);
```

链路为何断开
要知道链路为何断开，我们就需要知道，正常情况下的链路是如何工作的，几个Span之间是如何接在一起的。我们可以通过@Trace注解进行入手，这个注解会增加一个Span。

正常情况下@Trace添加Span
对skywalking源码有一定了解的你一定知道，其对类做修改增强的时候，会定义一个该类全类名的字符串，以及会用来增强该类的增强类的全类名，所以我们找到了TraceAnnotationActivation：
```java
/**
 * {@link TraceAnnotationActivation} enhance all method that annotated with <code>org.apache.skywalking.apm.toolkit.trace.annotation.Trace</code>
 * by <code>TraceAnnotationMethodInterceptor</code>.
 */
public class TraceAnnotationActivation extends ClassInstanceMethodsEnhancePluginDefine {
   // 用来增强的类
    public static final String TRACE_ANNOTATION_METHOD_INTERCEPTOR = "org.apache.skywalking.apm.toolkit.activation.trace.TraceAnnotationMethodInterceptor";
    // 被增强处理的注解
    public static final String TRACE_ANNOTATION = "org.apache.skywalking.apm.toolkit.trace.Trace";

    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return new ConstructorInterceptPoint[0];
    }

    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[] {
            new DeclaredInstanceMethodsInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    return isAnnotatedWith(named(TRACE_ANNOTATION));
                }

                @Override
                public String getMethodsInterceptor() {
                    return TRACE_ANNOTATION_METHOD_INTERCEPTOR;
                }

                @Override
                public boolean isOverrideArgs() {
                    return false;
                }
            }
        };
    }

    @Override
    protected ClassMatch enhanceClass() {
        return MethodAnnotationMatch.byMethodAnnotationMatch(TRACE_ANNOTATION);
    }
}
```
然后我们去翻查TraceAnnotationMethodInterceptor:
```
/**
 * {@link TraceAnnotationMethodInterceptor} create a local span and set the operation name which fetch from
 * <code>org.apache.skywalking.apm.toolkit.trace.annotation.Trace.operationName</code>. if the fetch value is blank
 * string, and the operation name will be the method name.
 */
public class TraceAnnotationMethodInterceptor implements InstanceMethodsAroundInterceptor {
    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                             MethodInterceptResult result) throws Throwable {
        Trace trace = method.getAnnotation(Trace.class);
        final AbstractSpan localSpan = ContextManager.createLocalSpan(operationName);

    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                              Object ret) throws Throwable {
   
        ContextManager.stopSpan();
        
        return ret;
    }

}

```
代码做了一定精简，我们可以看到，在beforeMethod()方法中，其使用ContextManager创建了一个LocalSpan，在afterMethod()方法中，使用了ContextManager停止了Span。而ContextManager是skywalking链路管理的一个核心的类，那么也一定是它在创建Span的时候没有续接上导致的。

ContextManager创建Span的核心
```
/**
 * {@link ContextManager} controls the whole context of {@link TraceSegment}. Any {@link TraceSegment} relates to
 * single-thread, so this context use {@link ThreadLocal} to maintain the context, and make sure, since a {@link
 * TraceSegment} starts, all ChildOf spans are in the same context. <p> What is 'ChildOf'?
 * https://github.com/opentracing/specification/blob/master/specification.md#references-between-spans
 *
 * <p> Also, {@link ContextManager} delegates to all {@link AbstractTracerContext}'s major methods.
 */
public class ContextManager implements BootService {
    private static final String EMPTY_TRACE_CONTEXT_ID = "N/A";
    private static final ILog LOGGER = LogManager.getLogger(ContextManager.class);
    private static ThreadLocal<AbstractTracerContext> CONTEXT = new ThreadLocal<AbstractTracerContext>();
    private static ThreadLocal<RuntimeContext> RUNTIME_CONTEXT = new ThreadLocal<RuntimeContext>();
    private static ContextManagerExtendService EXTEND_SERVICE;

    private static AbstractTracerContext getOrCreate(String operationName, boolean forceSampling) {
        AbstractTracerContext context = CONTEXT.get();
        if (context == null) {
            if (StringUtil.isEmpty(operationName)) {
                if (LOGGER.isDebugEnable()) {
                    LOGGER.debug("No operation name, ignore this trace.");
                }
                context = new IgnoredTracerContext();
            } else {
                if (EXTEND_SERVICE == null) {
                    EXTEND_SERVICE = ServiceManager.INSTANCE.findService(ContextManagerExtendService.class);
                }
                context = EXTEND_SERVICE.createTraceContext(operationName, forceSampling);

            }
            CONTEXT.set(context);
        }
        return context;
    }

    private static AbstractTracerContext get() {
        return CONTEXT.get();
    }
    
        public static AbstractSpan createLocalSpan(String operationName) {
        operationName = StringUtil.cut(operationName, OPERATION_NAME_THRESHOLD);
        AbstractTracerContext context = getOrCreate(operationName, false);
        return context.createLocalSpan(operationName);
    }
}

```
看到人家写的注释没，“Any TraceSegment relates to single-thread, so this context use ThreadLocal to maintain the context, and make sure, since a TraceSegment starts, all ChildOf spans are in the same context.” 一条链路就是一个单线程的，所以用了ThreadLocal来保存，让我们自己来保证，子Span是同一个上下文中的。

ThreadLocal<AbstractTracerContext> CONTEXT 这一个变量，用来存Span，那难怪了，新的线程中，它就是断开的。

## 链路如何续接

我们搞清楚了，断开是因为CONTEXT是存在ThreadLocal中的，导致新的线程中没有上下文，那么我们只要将父线程的上下文传入进去，就可以完成续接。

那让我们来看看skywalking是怎么做的。我们以@TraceCrossThread为例，其他方式大体思路是一致的。
```
/**
 * {@link CallableOrRunnableActivation} presents that skywalking intercepts all Class with annotation
 * "org.skywalking.apm.toolkit.trace.TraceCrossThread" and method named "call" or "run".
 */
public class CallableOrRunnableActivation extends ClassInstanceMethodsEnhancePluginDefine {

    public static final String ANNOTATION_NAME = "org.apache.skywalking.apm.toolkit.trace.TraceCrossThread";
    private static final String INIT_METHOD_INTERCEPTOR = "org.apache.skywalking.apm.toolkit.activation.trace.CallableOrRunnableConstructInterceptor";
    private static final String CALL_METHOD_INTERCEPTOR = "org.apache.skywalking.apm.toolkit.activation.trace.CallableOrRunnableInvokeInterceptor";
    private static final String CALL_METHOD_NAME = "call";
    private static final String RUN_METHOD_NAME = "run";
    private static final String GET_METHOD_NAME = "get";

    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return new ConstructorInterceptPoint[] {
            new ConstructorInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getConstructorMatcher() {
                    return any();
                }

                @Override
                public String getConstructorInterceptor() {
                    return INIT_METHOD_INTERCEPTOR;
                }
            }
        };
    }

    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[] {
            new InstanceMethodsInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    return named(CALL_METHOD_NAME)
                        .and(takesArguments(0))
                        .or(named(RUN_METHOD_NAME).and(takesArguments(0)))
                        .or(named(GET_METHOD_NAME).and(takesArguments(0)));
                }

                @Override
                public String getMethodsInterceptor() {
                    return CALL_METHOD_INTERCEPTOR;
                }

                @Override
                public boolean isOverrideArgs() {
                    return false;
                }
            }
        };
    }

    @Override
    protected ClassMatch enhanceClass() {
        return byClassAnnotationMatch(new String[] {ANNOTATION_NAME});
    }

}

```
通过全局搜索，我们找到CallableOrRunnableActivation，它完成了对"org.skywalking.apm.toolkit.trace.TraceCrossThread" and method named “call” or "run"的增强。增强方式分为构造时增强、以及对方法的增强。

CallableOrRunnableConstructInterceptor
```
public class CallableOrRunnableConstructInterceptor implements InstanceConstructorInterceptor {
    @Override
    public void onConstruct(EnhancedInstance objInst, Object[] allArguments) {
        if (ContextManager.isActive()) {
            objInst.setSkyWalkingDynamicField(ContextManager.capture());
        }
    }

}
```
在构造的时候，ContextManager对当前的上下文做了一次快照，并存到skyWalkingDynamicField这个动态属性中，共子线程来取。

CallableOrRunnableInvokeInterceptor
```
public class CallableOrRunnableInvokeInterceptor implements InstanceMethodsAroundInterceptor {
    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
        MethodInterceptResult result) throws Throwable {
        ContextManager.createLocalSpan("Thread/" + objInst.getClass().getName() + "/" + method.getName());
        ContextSnapshot cachedObjects = (ContextSnapshot) objInst.getSkyWalkingDynamicField();
        if (cachedObjects != null) {
            ContextManager.continued(cachedObjects);
        }
    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
        Object ret) throws Throwable {
        ContextManager.stopSpan();
        // clear ContextSnapshot
        objInst.setSkyWalkingDynamicField(null);
        return ret;
    }

}


```

这个类，将skyWalkingDynamicField这个动态属性中的内容取出，并通过“ContextManager.continued(cachedObjects);”完成了续接。最后，在afterMethod()中，也完成了对Span的关闭。

TracingContext#continued
```
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
            this.segment.relatedGlobalTraces(snapshot.getTraceId());
            this.correlationContext.continued(snapshot);
            this.extensionContext.continued(snapshot);
            this.extensionContext.handle(this.activeSpan());
        }
    }

```
这个注释也很明白的说明了，这个上下文会将父线程的快照进行续接。

第一步，通过对对象的构造方法进行增强，将链路上下文快照作为动态属性赋值给子线程；第二步，子线程的异步方法在开始前，将快照续接上并创建新的Span，方法结束后将Span关闭。




