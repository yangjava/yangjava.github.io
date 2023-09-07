---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码

## 插件定义体系
以dubbo 2.7.x的插件为例，我们来看下插件定义体系

### 插件的定义
```
/**
 * 插件的定义,继承xxxPluginDefine,通常命名为xxxInstrumentation
 */
public class DubboInstrumentation extends ClassInstanceMethodsEnhancePluginDefine {

    private static final String ENHANCE_CLASS = "org.apache.dubbo.monitor.support.MonitorFilter";

    private static final String INTERCEPT_CLASS = "org.apache.skywalking.apm.plugin.asf.dubbo.DubboInterceptor";

    /**
     * 指定插件增强哪个类的字节码
     */
    @Override
    protected ClassMatch enhanceClass() {
        return NameMatch.byName(ENHANCE_CLASS);
    }

    /**
     * 拿到构造方法的拦截点
     */
    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return null;
    }

    /**
     * 拿到实例方法的拦截点
     */
    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[] {
            new InstanceMethodsInterceptPoint() {
                /**
                 * 该插件增强的是MonitorFilter的invoke()方法
                 */
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    return named("invoke");
                }

                /**
                 * 交给DubboInterceptor进行字节码增强
                 */
                @Override
                public String getMethodsInterceptor() {
                    return INTERCEPT_CLASS;
                }

                /**
                 * 在字节码增强的过程中,是否要对原方法的入参进行改变
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

拦截实例方法/构造器时，需要继承ClassInstanceMethodsEnhancePluginDefine
```
public abstract class ClassInstanceMethodsEnhancePluginDefine extends ClassEnhancePluginDefine {

    /**
     * @return null, means enhance no static methods.
     */
    @Override
    public StaticMethodsInterceptPoint[] getStaticMethodsInterceptPoints() {
        return null;
    }

}

```

拦截静态方法时，需要继承ClassStaticMethodsEnhancePluginDefine
```
public abstract class ClassStaticMethodsEnhancePluginDefine extends ClassEnhancePluginDefine {

    /**
     * @return null, means enhance no constructors.
     */
    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return null;
    }

    /**
     * @return null, means enhance no instance methods.
     */
    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return null;
    }
}

```
AbstractClassEnhancePluginDefine是所有插件定义的顶级父类

## 目标类匹配
ClassMatch是一个标志接口，表示类的匹配器
```
public interface ClassMatch {
}
```
NameMatch是通过完整类名精确匹配对应类
```
/**
 * Match the class with an explicit class name.
 * 通过完整类名精确匹配对应类
 */
public class NameMatch implements ClassMatch {
    private String className;

    private NameMatch(String className) {
        this.className = className;
    }

    public String getClassName() {
        return className;
    }

    public static NameMatch byName(String className) {
        return new NameMatch(className);
    }
}

```

除了NameMatch之外，还有比较常用的IndirectMatch用于间接匹配
```
public interface IndirectMatch extends ClassMatch {
    ElementMatcher.Junction buildJunction();

    // TypeDescription就是对类的描述,可以当做Class
    boolean isMatch(TypeDescription typeDescription);
}

```

IndirectMatch的子类，比如：PrefixMatch通过前缀匹配
```
public class PrefixMatch implements IndirectMatch {
    private String[] prefixes;

    private PrefixMatch(String... prefixes) {
        if (prefixes == null || prefixes.length == 0) {
            throw new IllegalArgumentException("prefixes argument is null or empty");
        }
        this.prefixes = prefixes;
    }

    @Override
    public ElementMatcher.Junction buildJunction() {
        ElementMatcher.Junction junction = null;

        // 拼接匹配条件
        // 有多个前缀,只要一个匹配即可
        for (String prefix : prefixes) {
            if (junction == null) {
                junction = ElementMatchers.nameStartsWith(prefix);
            } else {
                junction = junction.or(ElementMatchers.nameStartsWith(prefix));
            }
        }

        return junction;
    }

    @Override
    public boolean isMatch(TypeDescription typeDescription) {
        for (final String prefix : prefixes) {
            // typeDescription.getName()就是全类名
            if (typeDescription.getName().startsWith(prefix)) {
                return true;
            }
        }
        return false;
    }

    public static PrefixMatch nameStartsWith(final String... prefixes) {
        return new PrefixMatch(prefixes);
    }
}

```

## 拦截器定义
```
public class DubboInterceptor implements InstanceMethodsAroundInterceptor {
```

DubboInterceptor继承了InstanceMethodsAroundInterceptor，InstanceMethodsAroundInterceptor源码如下：
```
public interface InstanceMethodsAroundInterceptor {
    /**
     * called before target method invocation.
     *
     * @param result change this result, if you want to truncate the method.
     */
    void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
        MethodInterceptResult result) throws Throwable;

    /**
     * called after target method invocation. Even method's invocation triggers an exception.
     *
     * @param ret the method's original return value. May be null if the method triggers an exception.
     * @return the method's actual return value.
     */
    Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
        Object ret) throws Throwable;

    /**
     * called when occur exception.
     *
     * @param t the exception occur.
     */
    void handleMethodException(EnhancedInstance objInst, Method method, Object[] allArguments,
        Class<?>[] argumentsTypes, Throwable t);
}

```
类似于AOP的环绕增强，在方法执行前、方法执行后、方法抛出异常时进行代码增强

## 插件的声明
resources目录下的skywalking-plugin.def文件是对插件的声明
```
# 插件名=插件定义全类名
dubbo=org.apache.skywalking.apm.plugin.asf.dubbo.DubboInstrumentation
```


































