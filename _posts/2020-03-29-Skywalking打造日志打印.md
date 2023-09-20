---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking打造日志打印

## 日志
```
log4j1-plugin=com.skywalking.apm.plugin.logger.define.Log4j1Instrumentation
log4j2-plugin=com.skywalking.apm.plugin.logger.define.Log4j2ConverterInstrumentation
logback-plugin=com.skywalking.apm.plugin.logger.define.LogbackConvertInstrumentation
logback-plugin=com.skywalking.apm.plugin.logger.define.LogbackPatternInstrumentation
```

### 对于Log4j1增强
```
public class Log4j1Instrumentation extends ClassInstanceMethodsEnhancePluginDefine {

    private static final String ENHANCE_CLASS = "org.apache.log4j.spi.LoggingEvent";

    private static final String INTERCEPT_CLASS = "com.skywalking.apm.plugin.logger.Log4j1Interceptor";

    @Override
    protected ClassMatch enhanceClass() {
        return NameMatch.byName(ENHANCE_CLASS);
    }


    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[]{
                new InstanceMethodsInterceptPoint() {
                    @Override
                    public ElementMatcher<MethodDescription> getMethodsMatcher() {
                        return ElementMatchers.named("getMessage").or(ElementMatchers.named("getRenderedMessage"));
                    }

                    @Override
                    public String getMethodsInterceptor() {
                        return INTERCEPT_CLASS;
                    }

                    @Override
                    public boolean isOverrideArgs() {
                        return false;
                    }
                }
        };
    }

    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return null;
    }

}
```


对于Log4j1增强
```java
public class Log4j1Interceptor implements InstanceMethodsAroundInterceptor {

    private static final int maxLength = 500;

    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, MethodInterceptResult result) throws Throwable {

    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, Object ret) throws Throwable {
        //当前上下文中没有traceId时，不做增强，避免输出无用日志
        if (!ContextManager.isActive()) {
            return ret;
        }

        String msg = (String) ret;
        String temMsg = msg;
        if (msg.length() > maxLength) {
            temMsg = msg.substring(0, maxLength);
        }
        if (temMsg.contains(LogConst.QJD_TID)) {
            return ret;
        }
        return LogConst.getPatternMsg(msg);
    }

    @Override
    public void handleMethodException(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, Throwable t) {

    }

}
```

```
import org.apache.skywalking.apm.agent.core.context.ContextManager;

public class LogConst {

    public final static String TID = "tid=";


    public static String getPatternMsg(String msg) {
        return TID + ContextManager.getGlobalTraceId() + msg;
    }

    public static String getTranceIdMsg() {
        return "[" + TID + ContextManager.getGlobalTraceId() + "] ";
    }
}
```

对于Log4j2的增强
```
public class Log4j2ConverterInstrumentation extends ClassInstanceMethodsEnhancePluginDefine {

    private static final String ENHANCE_CLASS = "org.apache.logging.log4j.core.pattern.MessagePatternConverter";

    private static final String INTERCEPT_CLASS = "com.skywalking.apm.plugin.logger.Log4j2ConverterInterceptor";

    @Override
    protected ClassMatch enhanceClass() {
        return NameMatch.byName(ENHANCE_CLASS);
    }


    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[]{
                new InstanceMethodsInterceptPoint() {
                    @Override
                    public ElementMatcher<MethodDescription> getMethodsMatcher() {
                        return ElementMatchers.named("format");
                    }

                    @Override
                    public String getMethodsInterceptor() {
                        return INTERCEPT_CLASS;
                    }

                    @Override
                    public boolean isOverrideArgs() {
                        return true;
                    }
                }
        };
    }

    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return null;
    }

}
```

```
public class Log4j2ConverterInterceptor implements InstanceMethodsAroundInterceptor {


    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, MethodInterceptResult result) throws Throwable {

        if (!ContextManager.isActive()) {
            return;
        }
        ((StringBuilder) allArguments[1]).insert(0, LogConst.getTranceIdMsg());
    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, Object ret) throws Throwable {
        return ret;
    }

    @Override
    public void handleMethodException(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, Throwable t) throws Throwable {

    }
}
```

对于LogBack增强
```
public class LogbackConvertInstrumentation extends ClassInstanceMethodsEnhancePluginDefine {

    private static final String ENHANCE_CLASS = "ch.qos.logback.classic.pattern.MDCConverter";

    private static final String INTERCEPT_CLASS = "com.skywalking.apm.plugin.logger.LogbackConvertInterceptor";

    @Override
    protected ClassMatch enhanceClass() {
        return NameMatch.byName(ENHANCE_CLASS);
    }

    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return null;
    }

    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[]{
                new InstanceMethodsInterceptPoint() {
                    @Override
                    public ElementMatcher<MethodDescription> getMethodsMatcher() {
                        return named("convert");
                    }

                    @Override
                    public String getMethodsInterceptor() {
                        return INTERCEPT_CLASS;
                    }

                    @Override
                    public boolean isOverrideArgs() {
                        return false;
                    }
                }
        };
    }
}
```

```
public class LogbackConvertInterceptor implements InstanceMethodsAroundInterceptor {


    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, MethodInterceptResult result) throws Throwable {

    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, Object ret) throws Throwable {
        //当前上下文中没有traceId时，不做增强，避免输出无用日志
        if (!ContextManager.isActive() || ContextManager.getGlobalTraceId().equals("Ignored_Trace")) {
            return ret;
        }
        String msg = (String) ret;
        return LogConst.getPatternMsg(msg);
    }

    @Override
    public void handleMethodException(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, Throwable t) {

    }

}
```

对于Logback增强
```
public class LogbackPatternInstrumentation extends ClassInstanceMethodsEnhancePluginDefine {

    private static final String ENHANCE_CLASS = "ch.qos.logback.core.pattern.PatternLayoutEncoderBase";

    private static final String INTERCEPT_CLASS = "com.skywalking.apm.plugin.logger.LogbackPatternInterceptor";

    @Override
    protected ClassMatch enhanceClass() {
        return NameMatch.byName(ENHANCE_CLASS);
    }

    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return null;
    }

    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[]{
                new InstanceMethodsInterceptPoint() {
                    @Override
                    public ElementMatcher<MethodDescription> getMethodsMatcher() {
                        return named("setPattern");
                    }

                    @Override
                    public String getMethodsInterceptor() {
                        return INTERCEPT_CLASS;
                    }

                    @Override
                    public boolean isOverrideArgs() {
                        return true;
                    }
                }
        };
    }
}
```

```
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.EnhancedInstance;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.InstanceMethodsAroundInterceptor;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.MethodInterceptResult;

import java.lang.reflect.Method;

public class LogbackPatternInterceptor implements InstanceMethodsAroundInterceptor {


    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, MethodInterceptResult result) throws Throwable {
//        allArguments[0] = "[%mdc]" + allArguments[0];
        allArguments[0] = "[%X{qjd-tid}]" + allArguments[0];
    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, Object ret) throws Throwable {
        return ret;
    }

    @Override
    public void handleMethodException(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, Throwable t) {

    }

}
```





























