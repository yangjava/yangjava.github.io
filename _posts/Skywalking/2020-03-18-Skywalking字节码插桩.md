---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
## Skywalking字节码插桩

10、静态方法插桩
Transform的transform()方法中调用每个插件的define()方法去做字节码增强，AbstractClassEnhancePluginDefine的define()方法中再调用自己的enhance()方法做字节码增强，enhance()方法源码如下：

public abstract class AbstractClassEnhancePluginDefine {

    /**
     * Begin to define how to enhance class. After invoke this method, only means definition is finished.
     *
     * @param typeDescription target class description
     * @param newClassBuilder byte-buddy's builder to manipulate class bytecode.
     * @return new byte-buddy's builder for further manipulation.
     */
    protected DynamicType.Builder<?> enhance(TypeDescription typeDescription, DynamicType.Builder<?> newClassBuilder,
                                             ClassLoader classLoader, EnhanceContext context) throws PluginException {
        // 静态方法插桩
        newClassBuilder = this.enhanceClass(typeDescription, newClassBuilder, classLoader);
    
        // 构造器和实例方法插桩
        newClassBuilder = this.enhanceInstance(typeDescription, newClassBuilder, classLoader, context);
    
        return newClassBuilder;
    }
      
    /**
     * Enhance a class to intercept class static methods.
     *
     * @param typeDescription target class description
     * @param newClassBuilder byte-buddy's builder to manipulate class bytecode.
     * @return new byte-buddy's builder for further manipulation.
     */
    protected abstract DynamicType.Builder<?> enhanceClass(TypeDescription typeDescription, DynamicType.Builder<?> newClassBuilder,
                                                  ClassLoader classLoader) throws PluginException;
      
    /**
     * Enhance a class to intercept constructors and class instance methods.
     *
     * @param typeDescription target class description
     * @param newClassBuilder byte-buddy's builder to manipulate class bytecode.
     * @return new byte-buddy's builder for further manipulation.
     */
    protected abstract DynamicType.Builder<?> enhanceInstance(TypeDescription typeDescription,
                                                     DynamicType.Builder<?> newClassBuilder, ClassLoader classLoader,
                                                     EnhanceContext context) throws PluginException;  
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
enhance()方法中先调用enhanceClass()方法做静态方法插桩，再调用enhanceInstance()方法做构造器和实例方法插桩，本节先来看下静态方法插桩

ClassEnhancePluginDefine中实现了AbstractClassEnhancePluginDefine的抽象方法enhanceClass()：

public abstract class ClassEnhancePluginDefine extends AbstractClassEnhancePluginDefine {

    /**
     * Enhance a class to intercept class static methods.
     *
     * @param typeDescription target class description
     * @param newClassBuilder byte-buddy's builder to manipulate class bytecode.
     * @return new byte-buddy's builder for further manipulation.
     */
    protected DynamicType.Builder<?> enhanceClass(TypeDescription typeDescription, DynamicType.Builder<?> newClassBuilder,
        ClassLoader classLoader) throws PluginException {
        // 获取静态方法拦截点
        StaticMethodsInterceptPoint[] staticMethodsInterceptPoints = getStaticMethodsInterceptPoints();
        String enhanceOriginClassName = typeDescription.getTypeName();
        if (staticMethodsInterceptPoints == null || staticMethodsInterceptPoints.length == 0) {
            return newClassBuilder;
        }
    
        for (StaticMethodsInterceptPoint staticMethodsInterceptPoint : staticMethodsInterceptPoints) {
            String interceptor = staticMethodsInterceptPoint.getMethodsInterceptor();
            if (StringUtil.isEmpty(interceptor)) {
                throw new EnhanceException("no StaticMethodsAroundInterceptor define to enhance class " + enhanceOriginClassName);
            }
    
            // 是否要修改原方法入参
            if (staticMethodsInterceptPoint.isOverrideArgs()) {
                // 是否为JDK类库的类 被Bootstrap ClassLoader加载
                if (isBootstrapInstrumentation()) {
                    newClassBuilder = newClassBuilder.method(isStatic().and(staticMethodsInterceptPoint.getMethodsMatcher()))
                                                     .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                .withBinders(Morph.Binder.install(OverrideCallable.class))
                                                                                .to(BootstrapInstrumentBoost.forInternalDelegateClass(interceptor)));
                } else {
                    newClassBuilder = newClassBuilder.method(isStatic().and(staticMethodsInterceptPoint.getMethodsMatcher()))
                                                     .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                .withBinders(Morph.Binder.install(OverrideCallable.class))
                                                                                .to(new StaticMethodsInterWithOverrideArgs(interceptor)));
                }
            } else {
                if (isBootstrapInstrumentation()) {
                    newClassBuilder = newClassBuilder.method(isStatic().and(staticMethodsInterceptPoint.getMethodsMatcher()))
                                                     .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                .to(BootstrapInstrumentBoost.forInternalDelegateClass(interceptor)));
                } else {
                    newClassBuilder = newClassBuilder.method(isStatic().and(staticMethodsInterceptPoint.getMethodsMatcher()))
                                                     .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                .to(new StaticMethodsInter(interceptor)));
                }
            }
    
        }
    
        return newClassBuilder;
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
enhanceClass()方法处理逻辑如下：

获取静态方法拦截点
根据是否要修改原方法入参和是否为JDK类库的类走不通的分支处理逻辑
获取静态方法拦截点调用的是getStaticMethodsInterceptPoints()方法

public abstract class AbstractClassEnhancePluginDefine {

    /**
     * Static methods intercept point. See {@link StaticMethodsInterceptPoint}
     *
     * @return collections of {@link StaticMethodsInterceptPoint}
     */
    public abstract StaticMethodsInterceptPoint[] getStaticMethodsInterceptPoints();  
1
2
3
4
5
6
7
8
1）、不修改原方法入参
以mysql-8.x-plugin为例：

public class ConnectionImplCreateInstrumentation extends AbstractMysqlInstrumentation {

    private static final String JDBC_ENHANCE_CLASS = "com.mysql.cj.jdbc.ConnectionImpl";
    
    private static final String CONNECT_METHOD = "getInstance";
    
    @Override
    public StaticMethodsInterceptPoint[] getStaticMethodsInterceptPoints() {
        return new StaticMethodsInterceptPoint[] {
            new StaticMethodsInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    return named(CONNECT_METHOD);
                }
    
                @Override
                public String getMethodsInterceptor() {
                    return "org.apache.skywalking.apm.plugin.jdbc.mysql.v8.ConnectionCreateInterceptor";
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
        return byName(JDBC_ENHANCE_CLASS);
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
该插件拦截的是ConnectionImpl类中的静态方法getInstance()，不需要修改原方法入参，交给拦截器ConnectionCreateInterceptor来处理

public abstract class ClassEnhancePluginDefine extends AbstractClassEnhancePluginDefine {

    /**
     * Enhance a class to intercept class static methods.
     *
     * @param typeDescription target class description
     * @param newClassBuilder byte-buddy's builder to manipulate class bytecode.
     * @return new byte-buddy's builder for further manipulation.
     */
    protected DynamicType.Builder<?> enhanceClass(TypeDescription typeDescription, DynamicType.Builder<?> newClassBuilder,
        ClassLoader classLoader) throws PluginException {
        // 获取静态方法拦截点
        StaticMethodsInterceptPoint[] staticMethodsInterceptPoints = getStaticMethodsInterceptPoints();
        String enhanceOriginClassName = typeDescription.getTypeName();
        if (staticMethodsInterceptPoints == null || staticMethodsInterceptPoints.length == 0) {
            return newClassBuilder;
        }
    
        for (StaticMethodsInterceptPoint staticMethodsInterceptPoint : staticMethodsInterceptPoints) {
            String interceptor = staticMethodsInterceptPoint.getMethodsInterceptor();
            if (StringUtil.isEmpty(interceptor)) {
                throw new EnhanceException("no StaticMethodsAroundInterceptor define to enhance class " + enhanceOriginClassName);
            }
    
            // 是否要修改原方法入参
            if (staticMethodsInterceptPoint.isOverrideArgs()) {
                // 是否为JDK类库的类 被Bootstrap ClassLoader加载
                if (isBootstrapInstrumentation()) {
                    newClassBuilder = newClassBuilder.method(isStatic().and(staticMethodsInterceptPoint.getMethodsMatcher()))
                                                     .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                .withBinders(Morph.Binder.install(OverrideCallable.class))
                                                                                .to(BootstrapInstrumentBoost.forInternalDelegateClass(interceptor)));
                } else {
                    newClassBuilder = newClassBuilder.method(isStatic().and(staticMethodsInterceptPoint.getMethodsMatcher()))
                                                     .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                .withBinders(Morph.Binder.install(OverrideCallable.class))
                                                                                .to(new StaticMethodsInterWithOverrideArgs(interceptor)));
                }
            } else {
                if (isBootstrapInstrumentation()) {
                    newClassBuilder = newClassBuilder.method(isStatic().and(staticMethodsInterceptPoint.getMethodsMatcher()))
                                                     .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                .to(BootstrapInstrumentBoost.forInternalDelegateClass(interceptor)));
                } else {
                  	// 1)
                    newClassBuilder = newClassBuilder.method(isStatic().and(staticMethodsInterceptPoint.getMethodsMatcher()))
                                                     .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                .to(new StaticMethodsInter(interceptor)));
                }
            }
    
        }
    
        return newClassBuilder;
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
mysql-8.x-plugin不需要修改原方法入参，并且拦截的类不是JDK类库的类，所以走的是代码1)处的分支处理逻辑

调用bytebuddy API，指定该方法为静态方法（isStatic()），指定方法名（staticMethodsInterceptPoint.getMethodsMatcher()），传入interceptor实例交给StaticMethodsInter去处理，StaticMethodsInter去做真正的字节码增强

StaticMethodsInter源码如下：

public class StaticMethodsInter {
private static final ILog LOGGER = LogManager.getLogger(StaticMethodsInter.class);

    /**
     * A class full name, and instanceof {@link StaticMethodsAroundInterceptor} This name should only stay in {@link
     * String}, the real {@link Class} type will trigger classloader failure. If you want to know more, please check on
     * books about Classloader or Classloader appointment mechanism.
     */
    private String staticMethodsAroundInterceptorClassName;
    
    /**
     * Set the name of {@link StaticMethodsInter#staticMethodsAroundInterceptorClassName}
     *
     * @param staticMethodsAroundInterceptorClassName class full name.
     */
    public StaticMethodsInter(String staticMethodsAroundInterceptorClassName) {
        this.staticMethodsAroundInterceptorClassName = staticMethodsAroundInterceptorClassName;
    }
    
    /**
     * Intercept the target static method.
     *
     * @param clazz        target class 要修改字节码的目标类
     * @param allArguments all method arguments 原方法所有的入参
     * @param method       method description. 原方法
     * @param zuper        the origin call ref. 原方法的调用 zuper.call()代表调用原方法
     * @return the return value of target static method.
     * @throws Exception only throw exception because of zuper.call() or unexpected exception in sky-walking ( This is a
     *                   bug, if anything triggers this condition ).
     */
    @RuntimeType
    public Object intercept(@Origin Class<?> clazz, @AllArguments Object[] allArguments, @Origin Method method,
        @SuperCall Callable<?> zuper) throws Throwable {
        // 实例化自定义的拦截器
        StaticMethodsAroundInterceptor interceptor = InterceptorInstanceLoader.load(staticMethodsAroundInterceptorClassName, clazz
            .getClassLoader());
    
        MethodInterceptResult result = new MethodInterceptResult();
        try {
            interceptor.beforeMethod(clazz, method, allArguments, method.getParameterTypes(), result);
        } catch (Throwable t) {
            LOGGER.error(t, "class[{}] before static method[{}] intercept failure", clazz, method.getName());
        }
    
        Object ret = null;
        try {
            // 是否执行原方法
            if (!result.isContinue()) {
                ret = result._ret();
            } else {
                // 原方法的调用
                ret = zuper.call();
            }
        } catch (Throwable t) {
            try {
                interceptor.handleMethodException(clazz, method, allArguments, method.getParameterTypes(), t);
            } catch (Throwable t2) {
                LOGGER.error(t2, "class[{}] handle static method[{}] exception failure", clazz, method.getName(), t2.getMessage());
            }
            throw t;
        } finally {
            try {
                ret = interceptor.afterMethod(clazz, method, allArguments, method.getParameterTypes(), ret);
            } catch (Throwable t) {
                LOGGER.error(t, "class[{}] after static method[{}] intercept failure:{}", clazz, method.getName(), t.getMessage());
            }
        }
        return ret;
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
intercept()方法处理逻辑如下：

实例化自定义的拦截器
执行beforeMethod()方法
如果需要执行原方法，执行原方法调用，否则调用_ret()方法
如果方法执行抛出异常，调用handleMethodException()方法
最终调用finally中afterMethod()方法
public class MethodInterceptResult {
private boolean isContinue = true;

    private Object ret = null;
    
    /**
     * define the new return value.
     *
     * @param ret new return value.
     */
    public void defineReturnValue(Object ret) {
        this.isContinue = false;
        this.ret = ret;
    }
    
    /**
     * @return true, will trigger method interceptor({@link InstMethodsInter} and {@link StaticMethodsInter}) to invoke
     * the origin method. Otherwise, not.
     */
    public boolean isContinue() {
        return isContinue;
    }
    
    /**
     * @return the new return value.
     */
    public Object _ret() {
        return ret;
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
这里是否执行原方法默认为true，如果插件的beforeMethod()方法实现中调用了defineReturnValue()传入了返回值，则不会再调用原方法，直接返回传入的返回值

2）、修改原方法入参
允许修改原方法入参会交给StaticMethodsInterWithOverrideArgs去处理

public class StaticMethodsInterWithOverrideArgs {
private static final ILog LOGGER = LogManager.getLogger(StaticMethodsInterWithOverrideArgs.class);

    /**
     * A class full name, and instanceof {@link StaticMethodsAroundInterceptor} This name should only stay in {@link
     * String}, the real {@link Class} type will trigger classloader failure. If you want to know more, please check on
     * books about Classloader or Classloader appointment mechanism.
     */
    private String staticMethodsAroundInterceptorClassName;
    
    /**
     * Set the name of {@link StaticMethodsInterWithOverrideArgs#staticMethodsAroundInterceptorClassName}
     *
     * @param staticMethodsAroundInterceptorClassName class full name.
     */
    public StaticMethodsInterWithOverrideArgs(String staticMethodsAroundInterceptorClassName) {
        this.staticMethodsAroundInterceptorClassName = staticMethodsAroundInterceptorClassName;
    }
    
    /**
     * Intercept the target static method.
     *
     * @param clazz        target class
     * @param allArguments all method arguments
     * @param method       method description.
     * @param zuper        the origin call ref.
     * @return the return value of target static method.
     * @throws Exception only throw exception because of zuper.call() or unexpected exception in sky-walking ( This is a
     *                   bug, if anything triggers this condition ).
     */
    @RuntimeType
    public Object intercept(@Origin Class<?> clazz, @AllArguments Object[] allArguments, @Origin Method method,
        @Morph OverrideCallable zuper) throws Throwable {
        StaticMethodsAroundInterceptor interceptor = InterceptorInstanceLoader.load(staticMethodsAroundInterceptorClassName, clazz
            .getClassLoader());
    
        MethodInterceptResult result = new MethodInterceptResult();
        try {
            // beforeMethod可以修改原方法入参
            interceptor.beforeMethod(clazz, method, allArguments, method.getParameterTypes(), result);
        } catch (Throwable t) {
            LOGGER.error(t, "class[{}] before static method[{}] intercept failure", clazz, method.getName());
        }
    
        Object ret = null;
        try {
            if (!result.isContinue()) {
                ret = result._ret();
            } else {
                // 原方法的调用时传入修改后的原方法入参
                ret = zuper.call(allArguments);
            }
        } catch (Throwable t) {
            try {
                interceptor.handleMethodException(clazz, method, allArguments, method.getParameterTypes(), t);
            } catch (Throwable t2) {
                LOGGER.error(t2, "class[{}] handle static method[{}] exception failure", clazz, method.getName(), t2.getMessage());
            }
            throw t;
        } finally {
            try {
                ret = interceptor.afterMethod(clazz, method, allArguments, method.getParameterTypes(), ret);
            } catch (Throwable t) {
                LOGGER.error(t, "class[{}] after static method[{}] intercept failure:{}", clazz, method.getName(), t.getMessage());
            }
        }
        return ret;
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
StaticMethodsInterWithOverrideArgs和StaticMethodsInter的区别在于最后一个入参类型为OverrideCallable

public interface OverrideCallable {
Object call(Object[] args);
}
1
2
3
插件的beforeMethod()方法实现中会修改原方法入参，然后在原方法的调用时传入修改后的原方法入参

小结：



11、构造器和实例方法插桩
ClassEnhancePluginDefine中实现了AbstractClassEnhancePluginDefine的抽象方法enhanceInstance()：

public abstract class ClassEnhancePluginDefine extends AbstractClassEnhancePluginDefine {

    /**
     * Enhance a class to intercept constructors and class instance methods.
     *
     * @param typeDescription target class description
     * @param newClassBuilder byte-buddy's builder to manipulate class bytecode.
     * @return new byte-buddy's builder for further manipulation.
     */
    protected DynamicType.Builder<?> enhanceInstance(TypeDescription typeDescription,
        DynamicType.Builder<?> newClassBuilder, ClassLoader classLoader,
        EnhanceContext context) throws PluginException {
        // 构造器拦截点
        ConstructorInterceptPoint[] constructorInterceptPoints = getConstructorsInterceptPoints();
        // 实例方法拦截点
        InstanceMethodsInterceptPoint[] instanceMethodsInterceptPoints = getInstanceMethodsInterceptPoints();
        String enhanceOriginClassName = typeDescription.getTypeName();
        boolean existedConstructorInterceptPoint = false;
        if (constructorInterceptPoints != null && constructorInterceptPoints.length > 0) {
            existedConstructorInterceptPoint = true;
        }
        boolean existedMethodsInterceptPoints = false;
        if (instanceMethodsInterceptPoints != null && instanceMethodsInterceptPoints.length > 0) {
            existedMethodsInterceptPoints = true;
        }
    
        /**
         * nothing need to be enhanced in class instance, maybe need enhance static methods.
         */
        if (!existedConstructorInterceptPoint && !existedMethodsInterceptPoints) {
            return newClassBuilder;
        }
    
        /**
         * Manipulate class source code.<br/>
         *
         * new class need:<br/>
         * 1.Add field, name {@link #CONTEXT_ATTR_NAME}.
         * 2.Add a field accessor for this field.
         *
         * And make sure the source codes manipulation only occurs once.
         * 这个操作只会发生一次
         */
        // 如果当前拦截的类没有实现EnhancedInstance接口
        if (!typeDescription.isAssignableTo(EnhancedInstance.class)) {
            // 没有新增新的字段或者实现新的接口
            if (!context.isObjectExtended()) {
                // 新增一个private volatile的Object类型字段 _$EnhancedClassField_ws
                // 实现EnhancedInstance接口的get/set作为新增字段的get/set方法
                newClassBuilder = newClassBuilder.defineField(
                    CONTEXT_ATTR_NAME, Object.class, ACC_PRIVATE | ACC_VOLATILE)
                                                 .implement(EnhancedInstance.class)
                                                 .intercept(FieldAccessor.ofField(CONTEXT_ATTR_NAME));
                // 将记录状态的上下文EnhanceContext设置为已新增新的字段或者实现新的接口
                context.extendObjectCompleted();
            }
        }
    
        /**
         * 2. enhance constructors
         * 增强构造器
         */
        if (existedConstructorInterceptPoint) {
            for (ConstructorInterceptPoint constructorInterceptPoint : constructorInterceptPoints) {
                if (isBootstrapInstrumentation()) {
                    newClassBuilder = newClassBuilder.constructor(constructorInterceptPoint.getConstructorMatcher())
                                                     .intercept(SuperMethodCall.INSTANCE.andThen(MethodDelegation.withDefaultConfiguration()
                                                                                                                 .to(BootstrapInstrumentBoost
                                                                                                                     .forInternalDelegateClass(constructorInterceptPoint
                                                                                                                         .getConstructorInterceptor()))));
                } else {
                    newClassBuilder = newClassBuilder.constructor(constructorInterceptPoint.getConstructorMatcher())
                                                     .intercept(SuperMethodCall.INSTANCE.andThen(MethodDelegation.withDefaultConfiguration()
                                                                                                                 .to(new ConstructorInter(constructorInterceptPoint
                                                                                                                     .getConstructorInterceptor(), classLoader))));
                }
            }
        }
    
        /**
         * 3. enhance instance methods
         * 增强实例方法
         */
        if (existedMethodsInterceptPoints) {
            for (InstanceMethodsInterceptPoint instanceMethodsInterceptPoint : instanceMethodsInterceptPoints) {
                String interceptor = instanceMethodsInterceptPoint.getMethodsInterceptor();
                if (StringUtil.isEmpty(interceptor)) {
                    throw new EnhanceException("no InstanceMethodsAroundInterceptor define to enhance class " + enhanceOriginClassName);
                }
                ElementMatcher.Junction<MethodDescription> junction = not(isStatic()).and(instanceMethodsInterceptPoint.getMethodsMatcher());
              	// 如果拦截点为DeclaredInstanceMethodsInterceptPoint
                if (instanceMethodsInterceptPoint instanceof DeclaredInstanceMethodsInterceptPoint) {
                    // 拿到的方法必须是当前类上的 通过注解匹配可能匹配到很多方法不是当前类上的
                    junction = junction.and(ElementMatchers.<MethodDescription>isDeclaredBy(typeDescription));
                }
                if (instanceMethodsInterceptPoint.isOverrideArgs()) {
                    if (isBootstrapInstrumentation()) {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .withBinders(Morph.Binder.install(OverrideCallable.class))
                                                                                    .to(BootstrapInstrumentBoost.forInternalDelegateClass(interceptor)));
                    } else {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .withBinders(Morph.Binder.install(OverrideCallable.class))
                                                                                    .to(new InstMethodsInterWithOverrideArgs(interceptor, classLoader)));
                    }
                } else {
                    if (isBootstrapInstrumentation()) {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .to(BootstrapInstrumentBoost.forInternalDelegateClass(interceptor)));
                    } else {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .to(new InstMethodsInter(interceptor, classLoader)));
                    }
                }
            }
        }
    
        return newClassBuilder;
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
113
114
115
116
117
118
119
120
121
122
123
enhanceInstance()方法处理逻辑如下：

如果当前拦截的类没有实现EnhancedInstance接口且没有新增新的字段或者实现新的接口，则会新增一个private volatile的Object类型字段_$EnhancedClassField_ws，并实现EnhancedInstance接口的get/set作为新增字段的get/set方法，最后设置标记位，保证该操作只会发生一次
增强构造器
增强实例方法
1）、构造器插桩
构造器插桩会交给ConstructorInter去处理

public class ConstructorInter {
private static final ILog LOGGER = LogManager.getLogger(ConstructorInter.class);

    /**
     * An {@link InstanceConstructorInterceptor} This name should only stay in {@link String}, the real {@link Class}
     * type will trigger classloader failure. If you want to know more, please check on books about Classloader or
     * Classloader appointment mechanism.
     */
    private InstanceConstructorInterceptor interceptor;
    
    /**
     * @param constructorInterceptorClassName class full name.
     */
    public ConstructorInter(String constructorInterceptorClassName, ClassLoader classLoader) throws PluginException {
        try {
            // 实例化自定义的拦截器
            interceptor = InterceptorInstanceLoader.load(constructorInterceptorClassName, classLoader);
        } catch (Throwable t) {
            throw new PluginException("Can't create InstanceConstructorInterceptorV2.", t);
        }
    }
    
    /**
     * Intercept the target constructor.
     *
     * @param obj          target class instance. 目标类的实例
     * @param allArguments all constructor arguments
     */
    @RuntimeType
    public void intercept(@This Object obj, @AllArguments Object[] allArguments) {
        try {
            EnhancedInstance targetObject = (EnhancedInstance) obj;
            // 在原生构造器执行之后再执行后调用onConstruct()方法
            // 只能访问到EnhancedInstance类型的字段 _$EnhancedClassField_ws
            // 拦截器的onConstruct把某些数据存储到_$EnhancedClassField_ws字段中
            interceptor.onConstruct(targetObject, allArguments);
        } catch (Throwable t) {
            LOGGER.error("ConstructorInter failure.", t);
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
ConstructorInter处理逻辑如下：

构造函数中实例化自定义的拦截器
intercept()方法中调用拦截器的onConstruct()方法（在原生构造器执行之后再执行后）
public interface InstanceConstructorInterceptor {
/**
* Called after the origin constructor invocation.
* 在原生构造器执行之后再执行
*/
void onConstruct(EnhancedInstance objInst, Object[] allArguments) throws Throwable;
}
1
2
3
4
5
6
7
案例：

以activemq-5.x-plugin为例：

public class ActiveMQProducerInstrumentation extends ClassInstanceMethodsEnhancePluginDefine {
public static final String INTERCEPTOR_CLASS = "org.apache.skywalking.apm.plugin.activemq.ActiveMQProducerInterceptor";
public static final String ENHANCE_CLASS_PRODUCER = "org.apache.activemq.ActiveMQMessageProducer";
public static final String CONSTRUCTOR_INTERCEPTOR_CLASS = "org.apache.skywalking.apm.plugin.activemq.ActiveMQProducerConstructorInterceptor";
public static final String ENHANCE_METHOD = "send";
public static final String CONSTRUCTOR_INTERCEPT_TYPE = "org.apache.activemq.ActiveMQSession";

    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return new ConstructorInterceptPoint[] {
            new ConstructorInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getConstructorMatcher() {
                    return takesArgumentWithType(0, CONSTRUCTOR_INTERCEPT_TYPE);
                }
    
                @Override
                public String getConstructorInterceptor() {
                    return CONSTRUCTOR_INTERCEPTOR_CLASS;
                }
            }
        };
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
该插件拦截的是ActiveMQMessageProducer类中的第一个参数类型为ActiveMQSession的构造器，交给拦截器ActiveMQProducerInterceptor来处理

public class ActiveMQProducerConstructorInterceptor implements InstanceConstructorInterceptor {
@Override
public void onConstruct(EnhancedInstance objInst, Object[] allArguments) {
ActiveMQSession session = (ActiveMQSession) allArguments[0];
// 将broker地址保存到_$EnhancedClassField_ws字段中
objInst.setSkyWalkingDynamicField(session.getConnection().getTransport().getRemoteAddress().split("//")[1]);
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
所以插件中的构造器插桩是为了在_$EnhancedClassField_ws字段中保存一些数据方便后续使用，这也是ClassEnhancePluginDefine的enhanceInstance()方法中为什么让当前拦截的类新增_$EnhancedClassField_ws字段并实现EnhancedInstance接口的get/set作为新增字段的get/set方法

2）、实例方法插桩
public abstract class ClassEnhancePluginDefine extends AbstractClassEnhancePluginDefine {

    /**
     * Enhance a class to intercept constructors and class instance methods.
     *
     * @param typeDescription target class description
     * @param newClassBuilder byte-buddy's builder to manipulate class bytecode.
     * @return new byte-buddy's builder for further manipulation.
     */
    protected DynamicType.Builder<?> enhanceInstance(TypeDescription typeDescription,
        DynamicType.Builder<?> newClassBuilder, ClassLoader classLoader,
        EnhanceContext context) throws PluginException {
        // 构造器拦截点
        ConstructorInterceptPoint[] constructorInterceptPoints = getConstructorsInterceptPoints();
        // 实例方法拦截点
        InstanceMethodsInterceptPoint[] instanceMethodsInterceptPoints = getInstanceMethodsInterceptPoints();
        String enhanceOriginClassName = typeDescription.getTypeName();
        boolean existedConstructorInterceptPoint = false;
        if (constructorInterceptPoints != null && constructorInterceptPoints.length > 0) {
            existedConstructorInterceptPoint = true;
        }
        boolean existedMethodsInterceptPoints = false;
        if (instanceMethodsInterceptPoints != null && instanceMethodsInterceptPoints.length > 0) {
            existedMethodsInterceptPoints = true;
        }
    
        /**
         * nothing need to be enhanced in class instance, maybe need enhance static methods.
         */
        if (!existedConstructorInterceptPoint && !existedMethodsInterceptPoints) {
            return newClassBuilder;
        }
    
        /**
         * Manipulate class source code.<br/>
         *
         * new class need:<br/>
         * 1.Add field, name {@link #CONTEXT_ATTR_NAME}.
         * 2.Add a field accessor for this field.
         *
         * And make sure the source codes manipulation only occurs once.
         * 这个操作只会发生一次
         */
        // 如果当前拦截的类没有实现EnhancedInstance接口
        if (!typeDescription.isAssignableTo(EnhancedInstance.class)) {
            // 没有新增新的字段或者实现新的接口
            if (!context.isObjectExtended()) {
                // 新增一个private volatile的Object类型字段 _$EnhancedClassField_ws
                // 实现EnhancedInstance接口的get/set作为新增字段的get/set方法
                newClassBuilder = newClassBuilder.defineField(
                    CONTEXT_ATTR_NAME, Object.class, ACC_PRIVATE | ACC_VOLATILE)
                                                 .implement(EnhancedInstance.class)
                                                 .intercept(FieldAccessor.ofField(CONTEXT_ATTR_NAME));
                // 将记录状态的上下文EnhanceContext设置为已新增新的字段或者实现新的接口
                context.extendObjectCompleted();
            }
        }
    
        /**
         * 2. enhance constructors
         * 增强构造器
         */
        if (existedConstructorInterceptPoint) {
            for (ConstructorInterceptPoint constructorInterceptPoint : constructorInterceptPoints) {
                if (isBootstrapInstrumentation()) {
                    newClassBuilder = newClassBuilder.constructor(constructorInterceptPoint.getConstructorMatcher())
                                                     .intercept(SuperMethodCall.INSTANCE.andThen(MethodDelegation.withDefaultConfiguration()
                                                                                                                 .to(BootstrapInstrumentBoost
                                                                                                                     .forInternalDelegateClass(constructorInterceptPoint
                                                                                                                         .getConstructorInterceptor()))));
                } else {
                    newClassBuilder = newClassBuilder.constructor(constructorInterceptPoint.getConstructorMatcher())
                                                     .intercept(SuperMethodCall.INSTANCE.andThen(MethodDelegation.withDefaultConfiguration()
                                                                                                                 .to(new ConstructorInter(constructorInterceptPoint
                                                                                                                     .getConstructorInterceptor(), classLoader))));
                }
            }
        }
    
        /**
         * 3. enhance instance methods
         * 增强实例方法
         */
        if (existedMethodsInterceptPoints) {
            for (InstanceMethodsInterceptPoint instanceMethodsInterceptPoint : instanceMethodsInterceptPoints) {
                String interceptor = instanceMethodsInterceptPoint.getMethodsInterceptor();
                if (StringUtil.isEmpty(interceptor)) {
                    throw new EnhanceException("no InstanceMethodsAroundInterceptor define to enhance class " + enhanceOriginClassName);
                }
                ElementMatcher.Junction<MethodDescription> junction = not(isStatic()).and(instanceMethodsInterceptPoint.getMethodsMatcher());
              	// 1)如果拦截点为DeclaredInstanceMethodsInterceptPoint
                if (instanceMethodsInterceptPoint instanceof DeclaredInstanceMethodsInterceptPoint) {
                    // 拿到的方法必须是当前类上的 通过注解匹配可能匹配到很多方法不是当前类上的
                    junction = junction.and(ElementMatchers.<MethodDescription>isDeclaredBy(typeDescription));
                }
                if (instanceMethodsInterceptPoint.isOverrideArgs()) {
                    if (isBootstrapInstrumentation()) {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .withBinders(Morph.Binder.install(OverrideCallable.class))
                                                                                    .to(BootstrapInstrumentBoost.forInternalDelegateClass(interceptor)));
                    } else {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .withBinders(Morph.Binder.install(OverrideCallable.class))
                                                                                    .to(new InstMethodsInterWithOverrideArgs(interceptor, classLoader)));
                    }
                } else {
                    if (isBootstrapInstrumentation()) {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .to(BootstrapInstrumentBoost.forInternalDelegateClass(interceptor)));
                    } else {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .to(new InstMethodsInter(interceptor, classLoader)));
                    }
                }
            }
        }
    
        return newClassBuilder;
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
113
114
115
116
117
118
119
120
121
122
123
代码1)处判断如果拦截点为DeclaredInstanceMethodsInterceptPoint，会新增匹配条件：拿到的方法必须是当前类上的

/**
* this interface for those who only want to enhance declared method in case of some unexpected issue, such as spring
* controller
*
* Person
*  sayHello();
*
* UserController
*  addUser/saveUser
*  removeUser
*
* //@GetMapping @PostMapping
  */
  public interface DeclaredInstanceMethodsInterceptPoint extends InstanceMethodsInterceptPoint {
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
  如果要增强Person的sayHello()方法，那么可以直接通过类名方法名指定，但是如果需要增强所有Controller的方法，需要通过注解指定

通过注解匹配可能匹配到很多方法不是当前类上的，所以判断如果拦截点为DeclaredInstanceMethodsInterceptPoint会新增匹配条件：拿到的方法必须是当前类上的

1）实例方法插桩不修改原方法入参会交给InstMethodsInter去处理

public class InstMethodsInter {
private static final ILog LOGGER = LogManager.getLogger(InstMethodsInter.class);

    /**
     * An {@link InstanceMethodsAroundInterceptor} This name should only stay in {@link String}, the real {@link Class}
     * type will trigger classloader failure. If you want to know more, please check on books about Classloader or
     * Classloader appointment mechanism.
     */
    private InstanceMethodsAroundInterceptor interceptor;
    
    /**
     * @param instanceMethodsAroundInterceptorClassName class full name.
     */
    public InstMethodsInter(String instanceMethodsAroundInterceptorClassName, ClassLoader classLoader) {
        try {
            // 对于同一份字节码,如果由不同的类加载器进行加载,则加载出来的两个实例不相同
            interceptor = InterceptorInstanceLoader.load(instanceMethodsAroundInterceptorClassName, classLoader);
        } catch (Throwable t) {
            throw new PluginException("Can't create InstanceMethodsAroundInterceptor.", t);
        }
    }
    
    /**
     * Intercept the target instance method.
     *
     * @param obj          target class instance.
     * @param allArguments all method arguments
     * @param method       method description.
     * @param zuper        the origin call ref.
     * @return the return value of target instance method.
     * @throws Exception only throw exception because of zuper.call() or unexpected exception in sky-walking ( This is a
     *                   bug, if anything triggers this condition ).
     */
    @RuntimeType
    public Object intercept(@This Object obj, @AllArguments Object[] allArguments, @SuperCall Callable<?> zuper,
        @Origin Method method) throws Throwable {
        EnhancedInstance targetObject = (EnhancedInstance) obj;
    
        MethodInterceptResult result = new MethodInterceptResult();
        try {
            interceptor.beforeMethod(targetObject, method, allArguments, method.getParameterTypes(), result);
        } catch (Throwable t) {
            LOGGER.error(t, "class[{}] before method[{}] intercept failure", obj.getClass(), method.getName());
        }
    
        Object ret = null;
        try {
            if (!result.isContinue()) {
                ret = result._ret();
            } else {
                ret = zuper.call();
            }
        } catch (Throwable t) {
            try {
                interceptor.handleMethodException(targetObject, method, allArguments, method.getParameterTypes(), t);
            } catch (Throwable t2) {
                LOGGER.error(t2, "class[{}] handle method[{}] exception failure", obj.getClass(), method.getName());
            }
            throw t;
        } finally {
            try {
                ret = interceptor.afterMethod(targetObject, method, allArguments, method.getParameterTypes(), ret);
            } catch (Throwable t) {
                LOGGER.error(t, "class[{}] after method[{}] intercept failure", obj.getClass(), method.getName());
            }
        }
        return ret;
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
对于同一份字节码，如果由不同的类加载器进行加载，则加载出来的两个实例不相同，所以在加载拦截器的时候传入了classLoader

2）实例方法插桩修改原方法入参会交给InstMethodsInterWithOverrideArgs去处理

public class InstMethodsInterWithOverrideArgs {
private static final ILog LOGGER = LogManager.getLogger(InstMethodsInterWithOverrideArgs.class);

    /**
     * An {@link InstanceMethodsAroundInterceptor} This name should only stay in {@link String}, the real {@link Class}
     * type will trigger classloader failure. If you want to know more, please check on books about Classloader or
     * Classloader appointment mechanism.
     */
    private InstanceMethodsAroundInterceptor interceptor;
    
    /**
     * @param instanceMethodsAroundInterceptorClassName class full name.
     */
    public InstMethodsInterWithOverrideArgs(String instanceMethodsAroundInterceptorClassName, ClassLoader classLoader) {
        try {
            interceptor = InterceptorInstanceLoader.load(instanceMethodsAroundInterceptorClassName, classLoader);
        } catch (Throwable t) {
            throw new PluginException("Can't create InstanceMethodsAroundInterceptor.", t);
        }
    }
    
    /**
     * Intercept the target instance method.
     *
     * @param obj          target class instance.
     * @param allArguments all method arguments
     * @param method       method description.
     * @param zuper        the origin call ref.
     * @return the return value of target instance method.
     * @throws Exception only throw exception because of zuper.call() or unexpected exception in sky-walking ( This is a
     *                   bug, if anything triggers this condition ).
     */
    @RuntimeType
    public Object intercept(@This Object obj, @AllArguments Object[] allArguments, @Origin Method method,
        @Morph OverrideCallable zuper) throws Throwable {
        EnhancedInstance targetObject = (EnhancedInstance) obj;
    
        MethodInterceptResult result = new MethodInterceptResult();
        try {
            interceptor.beforeMethod(targetObject, method, allArguments, method.getParameterTypes(), result);
        } catch (Throwable t) {
            LOGGER.error(t, "class[{}] before method[{}] intercept failure", obj.getClass(), method.getName());
        }
    
        Object ret = null;
        try {
            if (!result.isContinue()) {
                ret = result._ret();
            } else {
                ret = zuper.call(allArguments);
            }
        } catch (Throwable t) {
            try {
                interceptor.handleMethodException(targetObject, method, allArguments, method.getParameterTypes(), t);
            } catch (Throwable t2) {
                LOGGER.error(t2, "class[{}] handle method[{}] exception failure", obj.getClass(), method.getName());
            }
            throw t;
        } finally {
            try {
                ret = interceptor.afterMethod(targetObject, method, allArguments, method.getParameterTypes(), ret);
            } catch (Throwable t) {
                LOGGER.error(t, "class[{}] after method[{}] intercept failure", obj.getClass(), method.getName());
            }
        }
        return ret;
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
不修改原方法入参和修改原方法入参的处理逻辑和静态方法插桩的处理逻辑相同

问题1：为什么StaticMethodsInter可以直接通过clazz.getClassLoader()获取类加载器，而InstMethodsInter需要从上层传递ClassLoader

StaticMethodsInter可以直接通过clazz.getClassLoader()获取类加载器是因为静态方法直接绑定了类

InstMethodsInter需要从上层传递ClassLoader有两个原因：第一个是一份字节码可能被多个ClassLoader加载，这样加载出来的每个实例都不相等，所以必须要绑定好ClassLoader。第二个原因是InstMethodsInter里对拦截器的加载前置到了构造方法，这是因为可能出现无法加载拦截器成功的情况，如果放到intercept()方法里去延后加载拦截器，那么拦截器加载失败产生的异常将和字节码修改导致的异常、业务异常出现混乱，这里是为了让异常边界清晰而做的处理

问题2：intercept()方法中通过zuper.call()执行原方法的调用，这里为什么不能替换成method.invoke(clazz.newInstance())

public class StaticMethodsInter {

    @RuntimeType
    public Object intercept(@Origin Class<?> clazz, @AllArguments Object[] allArguments, @Origin Method method,
        @SuperCall Callable<?> zuper) throws Throwable {
        // 实例化自定义的拦截器
        StaticMethodsAroundInterceptor interceptor = InterceptorInstanceLoader.load(staticMethodsAroundInterceptorClassName, clazz
            .getClassLoader());
    
        MethodInterceptResult result = new MethodInterceptResult();
        try {
            interceptor.beforeMethod(clazz, method, allArguments, method.getParameterTypes(), result);
        } catch (Throwable t) {
            LOGGER.error(t, "class[{}] before static method[{}] intercept failure", clazz, method.getName());
        }
    
        Object ret = null;
        try {
            // 是否执行原方法
            if (!result.isContinue()) {
                ret = result._ret();
            } else {
                // 原方法的调用
                ret = zuper.call();
//                ret = method.invoke(clazz.newInstance());
}
} catch (Throwable t) {
try {
interceptor.handleMethodException(clazz, method, allArguments, method.getParameterTypes(), t);
} catch (Throwable t2) {
LOGGER.error(t2, "class[{}] handle static method[{}] exception failure", clazz, method.getName(), t2.getMessage());
}
throw t;
} finally {
try {
ret = interceptor.afterMethod(clazz, method, allArguments, method.getParameterTypes(), ret);
} catch (Throwable t) {
LOGGER.error(t, "class[{}] after static method[{}] intercept failure:{}", clazz, method.getName(), t.getMessage());
}
}
return ret;
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
解答这个问题需要借助一个Java实时反编译工具friday，查看挂载SkyWalking Agent之后反编译的内容

@RestController
@RequestMapping("/api/hello")
public class UserController {

    @GetMapping
    public String sayHello() {
        return "hello";
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
UserController反编译的内容：

@RestController
@RequestMapping(value={"/api/hello"})
public class UserController
implements EnhancedInstance {
private volatile Object _$EnhancedClassField_ws;
public static volatile /* synthetic */ InstMethodsInter delegate$mvblfc0;
public static volatile /* synthetic */ InstMethodsInter delegate$hfbkh30;
public static volatile /* synthetic */ ConstructorInter delegate$gr07501;
private static final /* synthetic */ Method cachedValue$kkbY4FHP$ldstch2;
public static volatile /* synthetic */ InstMethodsInter delegate$lvp69q1;
public static volatile /* synthetic */ InstMethodsInter delegate$mpv7fs0;
public static volatile /* synthetic */ ConstructorInter delegate$v0q1e31;
private static final /* synthetic */ Method cachedValue$Hx3zGNqH$ldstch2;

    public UserController() {
        this(null);
        delegate$v0q1e31.intercept((Object)this, new Object[0]);
    }
    
    private /* synthetic */ UserController(auxiliary.YsFzTfDy ysFzTfDy) {
    }
    
    @GetMapping
    public String sayHello() {
        return (String)delegate$lvp69q1.intercept((Object)this, new Object[0], (Callable)new auxiliary.pEJy33Ip(this), cachedValue$Hx3zGNqH$ldstch2);
    }
    
    private /* synthetic */ String sayHello$original$70VVkKcL() {
        return "hello";
    }
    
    static {
        ClassLoader.getSystemClassLoader().loadClass("org.apache.skywalking.apm.dependencies.net.bytebuddy.dynamic.Nexus").getMethod("initialize", Class.class, Integer.TYPE).invoke(null, UserController.class, 544534948);
        cachedValue$Hx3zGNqH$ldstch2 = UserController.class.getMethod("sayHello", new Class[0]);
    }
    
    final /* synthetic */ String sayHello$original$70VVkKcL$accessor$Hx3zGNqH() {
        return this.sayHello$original$70VVkKcL();
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
SkyWalkingAgent的premain()方法中在构建agentBuilder时使用的策略是RETRANSFORMATION

public class SkyWalkingAgent {

    public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException {
      	// ...
        agentBuilder.type(pluginFinder.buildMatch()) // 指定ByteBuddy要拦截的类
                    .transform(new Transformer(pluginFinder))
                    .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION) // redefine和retransform的区别在于是否保留修改前的内容
                    .with(new RedefinitionListener())
                    .with(new Listener())
                    .installOn(instrumentation);
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
redefine和retransform的区别在于是否保留修改前的内容，从反编译的内容可以看到sayHello()方法的内容已经被修改掉了，原方法被重命名为sayHello$original$+随机字符串

新的sayHello()方法中调用了InstMethodsInter的intercept()方法，最后传入的intercept()的method参数是被修改后的sayHello()方法，不再指向原生的sayHello()方法，如果在InstMethodsInter使用method.invoke(clazz.newInstance())就相当于自己调自己，就会死递归下去

小结：



12、插件拦截器加载流程
无论是静态方法插桩还是构造器和实例方法插桩都会调用InterceptorInstanceLoader的load()方法来实例化插件拦截器

public class InterceptorInstanceLoader {

    private static ConcurrentHashMap<String, Object> INSTANCE_CACHE = new ConcurrentHashMap<String, Object>();
    private static ReentrantLock INSTANCE_LOAD_LOCK = new ReentrantLock();
    /**
     * key:加载当前插件要拦截的那个类的类加载器
     * value:既能加载拦截器,又能加载要拦截的那个类的类加载器
     */
    private static Map<ClassLoader, ClassLoader> EXTEND_PLUGIN_CLASSLOADERS = new HashMap<ClassLoader, ClassLoader>();
    
    /**
     * Load an instance of interceptor, and keep it singleton. Create {@link AgentClassLoader} for each
     * targetClassLoader, as an extend classloader. It can load interceptor classes from plugins, activations folders.
     *
     * @param className         the interceptor class, which is expected to be found 插件拦截器全类名
     * @param targetClassLoader the class loader for current application context 当前应用上下文的类加载器
     * @param <T>               expected type
     * @return the type reference.
     */
    public static <T> T load(String className,
        ClassLoader targetClassLoader) throws IllegalAccessException, InstantiationException, ClassNotFoundException, AgentPackageNotFoundException {
        if (targetClassLoader == null) {
            targetClassLoader = InterceptorInstanceLoader.class.getClassLoader();
        }
        // org.example.Hello_OF_org.example.classloader.MyClassLoader@xxxxx
        String instanceKey = className + "_OF_" + targetClassLoader.getClass()
                                                                   .getName() + "@" + Integer.toHexString(targetClassLoader
            .hashCode());
        // className所代表的拦截器的实例 对于同一个classloader而言相同的类只加载一次
        Object inst = INSTANCE_CACHE.get(instanceKey);
        if (inst == null) {
            INSTANCE_LOAD_LOCK.lock();
            ClassLoader pluginLoader;
            try {
                pluginLoader = EXTEND_PLUGIN_CLASSLOADERS.get(targetClassLoader);
                if (pluginLoader == null) {
                    // targetClassLoader作为AgentClassLoader的父类加载器
                    pluginLoader = new AgentClassLoader(targetClassLoader);
                    EXTEND_PLUGIN_CLASSLOADERS.put(targetClassLoader, pluginLoader);
                }
            } finally {
                INSTANCE_LOAD_LOCK.unlock();
            }
            // 通过pluginLoader来实例化拦截器对象
            inst = Class.forName(className, true, pluginLoader).newInstance();
            if (inst != null) {
                INSTANCE_CACHE.put(instanceKey, inst);
            }
        }
    
        return (T) inst;
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
为什么针对不同的targetClassLoader，要初始化多个AgentClassLoader实例，并用targetClassLoader作为AgentClassLoader的parent

如果只实例化一个AgentClassLoader实例，由于应用系统中的类不存在于AgentClassLoader的classpath下，那此时AgentClassLoader加载不到应用系统中的类

针对每个targetClassLoader都初始化一个AgentClassLoader实例，并用targetClassLoader作为AgentClassLoader的父类加载器，通过双亲委派模型模型，targetClassLoader可以加载应用系统中的类



以dubbo插件为例，假设应用系统中Dubbo的类是由AppClassLoader加载的

DubboInterceptor要修改MonitorFilter的字节码，两个类需要能交互，前提就是DubboInterceptor能通过某种方式访问到MonitorFilter



让AgentClassLoader的父类加载器指向加载Dubbo的AppClassLoader，当DubboInterceptor去操作MonitorFilter的时候，通过双亲委派模型模型，AgentClassLoader的父类加载器AppClassLoader能加载到MonitorFilter

13、JDK类库插件工作原理
1）、Agent启动流程：将必要的类注入到Bootstrap ClassLoader
public class SkyWalkingAgent {

    public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException {
        final PluginFinder pluginFinder;
        try {
            // 初始化配置
            SnifferConfigInitializer.initializeCoreConfig(agentArgs);
        } catch (Exception e) {
            // try to resolve a new logger, and use the new logger to write the error log here
            LogManager.getLogger(SkyWalkingAgent.class)
                    .error(e, "SkyWalking agent initialized failure. Shutting down.");
            return;
        } finally {
            // refresh logger again after initialization finishes
            LOGGER = LogManager.getLogger(SkyWalkingAgent.class);
        }
    
        try {
            // 加载插件
            pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
        } catch (AgentPackageNotFoundException ape) {
            LOGGER.error(ape, "Locate agent.jar failure. Shutting down.");
            return;
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent initialized failure. Shutting down.");
            return;
        }
    
        // 定制化Agent行为
        // 创建ByteBuddy实例
        final ByteBuddy byteBuddy = new ByteBuddy().with(TypeValidation.of(Config.Agent.IS_OPEN_DEBUGGING_CLASS));
    
        // 指定ByteBuddy要忽略的类
        AgentBuilder agentBuilder = new AgentBuilder.Default(byteBuddy).ignore(
                nameStartsWith("net.bytebuddy.")
                        .or(nameStartsWith("org.slf4j."))
                        .or(nameStartsWith("org.groovy."))
                        .or(nameContains("javassist"))
                        .or(nameContains(".asm."))
                        .or(nameContains(".reflectasm."))
                        .or(nameStartsWith("sun.reflect"))
                        .or(allSkyWalkingAgentExcludeToolkit())
                        .or(ElementMatchers.isSynthetic()));
    
        // 将必要的类注入到Bootstrap ClassLoader
        JDK9ModuleExporter.EdgeClasses edgeClasses = new JDK9ModuleExporter.EdgeClasses();
        try {
            agentBuilder = BootstrapInstrumentBoost.inject(pluginFinder, instrumentation, agentBuilder, edgeClasses);
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent inject bootstrap instrumentation failure. Shutting down.");
            return;
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
SkyWalkingAgent的premain()方法中会调用BootstrapInstrumentBoost的inject()方法将必要的类注入到Bootstrap ClassLoader，inject()方法源码如下：

public class BootstrapInstrumentBoost {

    public static AgentBuilder inject(PluginFinder pluginFinder, Instrumentation instrumentation,
        AgentBuilder agentBuilder, JDK9ModuleExporter.EdgeClasses edgeClasses) throws PluginException {
        // 所有要注入到Bootstrap ClassLoader里的类
        Map<String, byte[]> classesTypeMap = new HashMap<>();
    
        /**
         * 针对于目标类是JDK核心类库的插件,根据插件的拦截点的不同(实例方法、静态方法、构造方法)
         * 使用不同的模板(xxxTemplate)来定义新的拦截器的核心处理逻辑,并且将插件本身定义的拦截器的全类名
         * 赋值给模板的TARGET_INTERCEPTOR字段
         * 最终,这些新的拦截器的核心处理逻辑都会被放入到BootstrapClassLoader中
         */
      	// 1)
        if (!prepareJREInstrumentation(pluginFinder, classesTypeMap)) {
            return agentBuilder;
        }
    
        if (!prepareJREInstrumentationV2(pluginFinder, classesTypeMap)) {
            return agentBuilder;
        }
    
        for (String highPriorityClass : HIGH_PRIORITY_CLASSES) {
            loadHighPriorityClass(classesTypeMap, highPriorityClass);
        }
        for (String highPriorityClass : ByteBuddyCoreClasses.CLASSES) {
            loadHighPriorityClass(classesTypeMap, highPriorityClass);
        }
    
        /**
         * Prepare to open edge of necessary classes.
         */
        for (String generatedClass : classesTypeMap.keySet()) {
            edgeClasses.add(generatedClass);
        }
    
        /**
         * 将这些类注入到Bootstrap ClassLoader
         * Inject the classes into bootstrap class loader by using Unsafe Strategy.
         * ByteBuddy adapts the sun.misc.Unsafe and jdk.internal.misc.Unsafe automatically.
         */
        ClassInjector.UsingUnsafe.Factory factory = ClassInjector.UsingUnsafe.Factory.resolve(instrumentation);
        factory.make(null, null).injectRaw(classesTypeMap);
        agentBuilder = agentBuilder.with(new AgentBuilder.InjectionStrategy.UsingUnsafe.OfFactory(factory));
    
        return agentBuilder;
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
代码1)处先调用prepareJREInstrumentation()，源码如下：

public class BootstrapInstrumentBoost {

    private static boolean prepareJREInstrumentation(PluginFinder pluginFinder,
        Map<String, byte[]> classesTypeMap) throws PluginException {
        TypePool typePool = TypePool.Default.of(BootstrapInstrumentBoost.class.getClassLoader());
        // 1)所有要对JDK核心类库生效的插件
        List<AbstractClassEnhancePluginDefine> bootstrapClassMatchDefines = pluginFinder.getBootstrapClassMatchDefine();
        for (AbstractClassEnhancePluginDefine define : bootstrapClassMatchDefines) {
            // 是否定义实例方法拦截点
            if (Objects.nonNull(define.getInstanceMethodsInterceptPoints())) {
                for (InstanceMethodsInterceptPoint point : define.getInstanceMethodsInterceptPoints()) {
                  	// 2)
                    if (point.isOverrideArgs()) {
                        generateDelegator(
                            classesTypeMap, typePool, INSTANCE_METHOD_WITH_OVERRIDE_ARGS_DELEGATE_TEMPLATE, point
                                .getMethodsInterceptor());
                    } else {
                      	// 3)
                        generateDelegator(
                            classesTypeMap, typePool, INSTANCE_METHOD_DELEGATE_TEMPLATE, point.getMethodsInterceptor());
                    }
                }
            }
    
            // 是否定义构造器拦截点
            if (Objects.nonNull(define.getConstructorsInterceptPoints())) {
                for (ConstructorInterceptPoint point : define.getConstructorsInterceptPoints()) {
                    generateDelegator(
                        classesTypeMap, typePool, CONSTRUCTOR_DELEGATE_TEMPLATE, point.getConstructorInterceptor());
                }
            }
    
            // 是否定义静态方法拦截点
            if (Objects.nonNull(define.getStaticMethodsInterceptPoints())) {
                for (StaticMethodsInterceptPoint point : define.getStaticMethodsInterceptPoints()) {
                    if (point.isOverrideArgs()) {
                        generateDelegator(
                            classesTypeMap, typePool, STATIC_METHOD_WITH_OVERRIDE_ARGS_DELEGATE_TEMPLATE, point
                                .getMethodsInterceptor());
                    } else {
                        generateDelegator(
                            classesTypeMap, typePool, STATIC_METHOD_DELEGATE_TEMPLATE, point.getMethodsInterceptor());
                    }
                }
            }
        }
        return bootstrapClassMatchDefines.size() > 0;
    }
1
2