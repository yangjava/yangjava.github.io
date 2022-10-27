---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---

#### **源码解读-Agent 插件体系**

SkyWalkingAgent.Transformer#transform方法

```java
	 private static class Transformer implements AgentBuilder.Transformer {
        private PluginFinder pluginFinder;

        Transformer(PluginFinder pluginFinder) {
            this.pluginFinder = pluginFinder;
        }

        @Override
        public DynamicType.Builder<?> transform(final DynamicType.Builder<?> builder,
                                                final TypeDescription typeDescription,
                                                final ClassLoader classLoader,
                                                final JavaModule module) {
            List<AbstractClassEnhancePluginDefine> pluginDefines = pluginFinder.find(typeDescription);
            if (pluginDefines.size() > 0) {
                DynamicType.Builder<?> newBuilder = builder;
                EnhanceContext context = new EnhanceContext();
                for (AbstractClassEnhancePluginDefine define : pluginDefines) {
                    DynamicType.Builder<?> possibleNewBuilder = define.define(
                            typeDescription, newBuilder, classLoader, context);
                    if (possibleNewBuilder != null) {
                        newBuilder = possibleNewBuilder;
                    }
                }
                if (context.isEnhanced()) {
                    LOGGER.debug("Finish the prepare stage for {}.", typeDescription.getName());
                }

                return newBuilder;
            }

            LOGGER.debug("Matched class {}, but ignore by finding mechanism.", typeDescription.getTypeName());
            return builder;
        }
    }
```

插件结构（以dubbo-2.7.x-plugin为例）

DubboInstrumentation 代码解析

```java
/**
 * 任何XxxInstrumentation 用于定义插件拦截点的
 * 拦截点：指定类的指定方法（实例方法、构造方法、静态方法）
 * 一个Instrumentation 只能拦截一个类，即只能有一个enhanceClass，但是可以拦截这个类里面的多个方法，即getInstanceMethodsInterceptPoints()中可以new 多个InstanceMethodsInterceptPoint()
 */
public class DubboInstrumentation extends ClassInstanceMethodsEnhancePluginDefine {

    private static final String ENHANCE_CLASS = "org.apache.dubbo.monitor.support.MonitorFilter";

    private static final String INTERCEPT_CLASS = "org.apache.skywalking.apm.plugin.asf.dubbo.DubboInterceptor";

    /**
     * 指定插件要拦截的类
     */
    @Override
    protected ClassMatch enhanceClass() {
        return NameMatch.byName(ENHANCE_CLASS);
    }

    /**
     * 指定插件要拦截的方法，不拦截返回null
     */
    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return null;
    }

    /**
     * 指定插件要拦截的实例方法（同样会有静态方法、构造方法，最后会总结）
     */
    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[]{
                new InstanceMethodsInterceptPoint() {
                    /**
                     * 返回要拦截的方法名
                     */
                    @Override
                    public ElementMatcher<MethodDescription> getMethodsMatcher() {
                        return named("invoke");
                    }

                    /**
                     * 指定拦截器全类名，用于拦截到指定方法之后做具体操作的
                     */
                    @Override
                    public String getMethodsInterceptor() {
                        //详见1.2DubboInterceptor
                        return INTERCEPT_CLASS;
                    }

                    /**
                     * 指定是否需要在拦截的时候对原方法参数进行修改
                     */
                    @Override
                    public boolean isOverrideArgs() {
                        return false;
                    }
                }
        };
    }
}
```

DubboInterceptor

```java
// spring AOP 切面时，@Around注解，方法前后
public class DubboInterceptor implements InstanceMethodsAroundInterceptor {

    public static final String ARGUMENTS = "arguments";

    /**
     * <h2>Consumer:</h2> The serialized trace context data will
     * inject to the {@link RpcContext#attachments} for transport to provider side.
     * <p>
     * <h2>Provider:</h2> The serialized trace context data will extract from
     * {@link RpcContext#attachments}. current trace segment will ref if the serialize context data is not null.
     */
    //原方法调用前
    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                             MethodInterceptResult result) throws Throwable {
        Invoker invoker = (Invoker) allArguments[0];
        Invocation invocation = (Invocation) allArguments[1];
        RpcContext rpcContext = RpcContext.getContext();
        boolean isConsumer = rpcContext.isConsumerSide();
        URL requestURL = invoker.getUrl();

        AbstractSpan span;

        final String host = requestURL.getHost();
        final int port = requestURL.getPort();

        boolean needCollectArguments;
        int argumentsLengthThreshold;
        if (isConsumer) {
            final ContextCarrier contextCarrier = new ContextCarrier();
            span = ContextManager.createExitSpan(generateOperationName(requestURL, invocation), contextCarrier, host + ":" + port);
            //invocation.getAttachments().put("contextData", contextDataStr);
            //@see https://github.com/alibaba/dubbo/blob/dubbo-2.5.3/dubbo-rpc/dubbo-rpc-api/src/main/java/com/alibaba/dubbo/rpc/RpcInvocation.java#L154-L161
            CarrierItem next = contextCarrier.items();
            while (next.hasNext()) {
                next = next.next();
                rpcContext.getAttachments().put(next.getHeadKey(), next.getHeadValue());
                if (invocation.getAttachments().containsKey(next.getHeadKey())) {
                    invocation.getAttachments().remove(next.getHeadKey());
                }
            }
            needCollectArguments = DubboPluginConfig.Plugin.Dubbo.COLLECT_CONSUMER_ARGUMENTS;
            argumentsLengthThreshold = DubboPluginConfig.Plugin.Dubbo.CONSUMER_ARGUMENTS_LENGTH_THRESHOLD;
        } else {
            ContextCarrier contextCarrier = new ContextCarrier();
            CarrierItem next = contextCarrier.items();
            while (next.hasNext()) {
                next = next.next();
                next.setHeadValue(rpcContext.getAttachment(next.getHeadKey()));
            }

            span = ContextManager.createEntrySpan(generateOperationName(requestURL, invocation), contextCarrier);
            span.setPeer(rpcContext.getRemoteAddressString());
            needCollectArguments = DubboPluginConfig.Plugin.Dubbo.COLLECT_PROVIDER_ARGUMENTS;
            argumentsLengthThreshold = DubboPluginConfig.Plugin.Dubbo.PROVIDER_ARGUMENTS_LENGTH_THRESHOLD;
        }

        Tags.URL.set(span, generateRequestURL(requestURL, invocation));
        collectArguments(needCollectArguments, argumentsLengthThreshold, span, invocation);
        span.setComponent(ComponentsDefine.DUBBO);
        SpanLayer.asRPCFramework(span);
    }
	//调用后
    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                              Object ret) throws Throwable {
        Result result = (Result) ret;
        if (result != null && result.getException() != null) {
            dealException(result.getException());
        }

        ContextManager.stopSpan();
        return ret;
    }
	//调用异常时
    @Override
    public void handleMethodException(EnhancedInstance objInst, Method method, Object[] allArguments,
                                      Class<?>[] argumentsTypes, Throwable t) {
        dealException(t);
    }

    /**
     * Log the throwable, which occurs in Dubbo RPC service.
     */
    private void dealException(Throwable throwable) {
        AbstractSpan span = ContextManager.activeSpan();
        span.log(throwable);
    }

    /**
     * Format operation name. e.g. org.apache.skywalking.apm.plugin.test.Test.test(String)
     *
     * @return operation name.
     */
    private String generateOperationName(URL requestURL, Invocation invocation) {
        StringBuilder operationName = new StringBuilder();
        operationName.append(requestURL.getPath());
        operationName.append("." + invocation.getMethodName() + "(");
        for (Class<?> classes : invocation.getParameterTypes()) {
            operationName.append(classes.getSimpleName() + ",");
        }

        if (invocation.getParameterTypes().length > 0) {
            operationName.delete(operationName.length() - 1, operationName.length());
        }

        operationName.append(")");

        return operationName.toString();
    }

    /**
     * Format request url. e.g. dubbo://127.0.0.1:20880/org.apache.skywalking.apm.plugin.test.Test.test(String).
     *
     * @return request url.
     */
    private String generateRequestURL(URL url, Invocation invocation) {
        StringBuilder requestURL = new StringBuilder();
        requestURL.append(url.getProtocol() + "://");
        requestURL.append(url.getHost());
        requestURL.append(":" + url.getPort() + "/");
        requestURL.append(generateOperationName(url, invocation));
        return requestURL.toString();
    }

    private void collectArguments(boolean needCollectArguments, int argumentsLengthThreshold, AbstractSpan span, Invocation invocation) {
        if (needCollectArguments && argumentsLengthThreshold > 0) {
            Object[] parameters = invocation.getArguments();
            if (parameters != null && parameters.length > 0) {
                StringBuilder stringBuilder = new StringBuilder();
                boolean first = true;
                for (Object parameter : parameters) {
                    if (!first) {
                        stringBuilder.append(",");
                    }
                    stringBuilder.append(parameter);
                    if (stringBuilder.length() > argumentsLengthThreshold) {
                        stringBuilder.append("...");
                        break;
                    }
                    first = false;
                }
                span.tag(ARGUMENTS, stringBuilder.toString());
            }
        }
    }
}
```

##### 总结一下

##### 拦截实例方法

XxxImplInstrumentation 继承 ClassInstanceMethodsEnhancePluginDefine

重写 getInstanceMethodsInterceptPoints（）来定义实例方法拦截点

XxxInterceptor implements InstanceMethodsAroundInterceptor

重写 beforeMethod afterMethod handleMethodException 方法实现拦截增强

##### 拦截构造方法

XxxImplInstrumentation 继承 ClassInstanceMethodsEnhancePluginDefine

重写 getConstructorsInterceptPoints（）来定义构造方法拦截点

XxxInterceptor implements InstanceConstructorInterceptor

重写 onConstruct 方法实现拦截增强

##### 拦截静态方法

XxxImplInstrumentation 继承 ClassStaticMethodsEnhancePluginDefine

重写 getStaticMethodsInterceptPoints（）来定义静态方法拦截点

XxxInterceptor implements StaticMethodsAroundInterceptor

重写 beforeMethod afterMethod handleMethodException 方法实现拦截增强

##### 对于实例方法和静态方法拦截点里的三个接口

getMethodsMatcher() 定义拦截哪个方法

getMethodsInterceptor() 定义拦截到方法之后要使用那个拦截器

isOverrideArgs() 指定在拦截器工作的时候是否可以对原方法入参进行修改

#### **源码解读-WitnessClass 机制**

AbstractClassEnhancePluginDefine#witnessClasses（）

AbstractClassEnhancePluginDefine是所有插件的父类

```java
/*
* Witness classname。为什么需要Witness classname？
* 比如：一个库存在两个发行的版本（例如1.0、2.0），
* 它们包含相同的目标类，但是由于版本迭代器的缘故，
* 它们可能具有相同的名称，但方法不同，或者方法参数列表不同。
* 因此，如果我要针对特定的版本（例如1.0），显然版本号没法判断；
* 这时需要“WitnessClasses”。您可以仅在此特定发行版本中添加任何类
* （类似于 com.company.1.x.A，仅在1.0中），您可以实现目标。
*/
protected String[] witnessClasses() {
    //数组的判断逻辑是 && ，有多个类时，必须全部满足，才可以使用。
    return new String[] {};
}
```

**示例**

mysql5.x

```java
public class Constants {
    public static final String WITNESS_MYSQL_5X_CLASS = "com.mysql.jdbc.ConnectionImpl";
}
```

mysql6.x

```java
public class Constants {
    public static final String WITNESS_MYSQL_6X_CLASS = "com.mysql.cj.api.MysqlConnection";
}
```

mysql8.x

```java
public class Constants {
    public static final String WITNESS_MYSQL_8X_CLASS = "com.mysql.cj.interceptors.QueryInterceptor";
}
```

解决了什么问题

解决了 我们拿不到业务系统所有用的组件版本号时，可以准确匹配所使用的agent增强插件。