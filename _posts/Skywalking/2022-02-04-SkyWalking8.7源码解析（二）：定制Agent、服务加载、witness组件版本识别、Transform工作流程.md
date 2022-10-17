6、定制Agent
public class SkyWalkingAgent {
    private static ILog LOGGER = LogManager.getLogger(SkyWalkingAgent.class);

    /**
     * Main entrance. Use byte-buddy transform to enhance all classes, which define in plugins.
     * -javaagent:/path/to/agent.jar=agentArgs
     * -javaagent:/path/to/agent.jar=k1=v1,k2=v2...
     * 等号之后都是参数,也就是入参agentArgs
     * -javaagent参数必须在-jar之前
     */
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
    
        // 1)将必要的类注入到Bootstrap ClassLoader
        JDK9ModuleExporter.EdgeClasses edgeClasses = new JDK9ModuleExporter.EdgeClasses();
        try {
            agentBuilder = BootstrapInstrumentBoost.inject(pluginFinder, instrumentation, agentBuilder, edgeClasses);
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent inject bootstrap instrumentation failure. Shutting down.");
            return;
        }
    
        // 解决JDK模块系统的跨模块类访问
        try {
            agentBuilder = JDK9ModuleExporter.openReadEdge(instrumentation, agentBuilder, edgeClasses);
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent open read edge in JDK 9+ failure. Shutting down.");
            return;
        }
    
        // 将修改后的字节码保存到磁盘/内存上
        if (Config.Agent.IS_CACHE_ENHANCED_CLASS) {
            try {
                agentBuilder = agentBuilder.with(new CacheableTransformerDecorator(Config.Agent.CLASS_CACHE_MODE));
                LOGGER.info("SkyWalking agent class cache [{}] activated.", Config.Agent.CLASS_CACHE_MODE);
            } catch (Exception e) {
                LOGGER.error(e, "SkyWalking agent can't active class cache.");
            }
        }
    
        agentBuilder.type(pluginFinder.buildMatch()) // 指定ByteBuddy要拦截的类
                    .transform(new Transformer(pluginFinder))
                    .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION) // redefine和retransform的区别在于是否保留修改前的内容
                    .with(new RedefinitionListener())
                    .with(new Listener())
                    .installOn(instrumentation);
    
        try {
            // 启动服务
            ServiceManager.INSTANCE.boot();
        } catch (Exception e) {
            LOGGER.error(e, "Skywalking agent boot failure.");
        }
    
        // 注册关闭钩子
        Runtime.getRuntime()
                .addShutdownHook(new Thread(ServiceManager.INSTANCE::shutdown, "skywalking service shutdown thread"));
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
代码1)处调用BootstrapInstrumentBoost的inject()方法，代码如下：

public class BootstrapInstrumentBoost {

    public static AgentBuilder inject(PluginFinder pluginFinder, Instrumentation instrumentation,
        AgentBuilder agentBuilder, JDK9ModuleExporter.EdgeClasses edgeClasses) throws PluginException {
        // 所有要注入到Bootstrap ClassLoader里的类
        Map<String, byte[]> classesTypeMap = new HashMap<>();
    
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
为什么要把这些类注入到Bootstrap ClassLoader中？

根据双亲委派模型，AgentClassLoader、AppClassLoader、ExtClassLoader、Bootstrap ClassLoader关系如下图：


如果是要对一个Bootstrap ClassLoader加载的类进行字节码增强，假设是HttpClient


HttpClient是由Bootstrap ClassLoader加载的，而修改HttpClient字节码的插件由AgentClassLoader加载，但是Bootstrap ClassLoader无法访问AgentClassLoader中加载的类（不能从上往下访问），所以需要把修改HttpClient字节码的插件注入到Bootstrap ClassLoader中


小结：



7、服务加载
1）、什么是服务
SkyWalking Agent收集硬件、JVM监控指标以及指标上报等都是一个个服务模块组织起来的

SkyWalking Agent中的服务就是实现了org.apache.skywalking.apm.agent.core.boot.BootService接口的类，这个接口定义了一个服务的生命周期

public interface BootService {
    /**
     * 准备阶段
     *
     * @throws Throwable
     */
    void prepare() throws Throwable;

    /**
     * 启动阶段
     *
     * @throws Throwable
     */
    void boot() throws Throwable;
    
    /**
     * 启动完成阶段
     *
     * @throws Throwable
     */
    void onComplete() throws Throwable;
    
    /**
     * 关闭阶段
     *
     * @throws Throwable
     */
    void shutdown() throws Throwable;
    
    /**
     * 指定服务的优先级,优先级高的服务先启动
     * {@code BootService}s with higher priorities will be started earlier, and shut down later than those {@code BootService}s with lower priorities.
     *
     * @return the priority of this {@code BootService}.
     */
    default int priority() {
        return 0;
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
2）、SPI加载服务
premain()中调用ServiceManager的boot()方法加载并启动服务，源码如下：

public enum ServiceManager {
    INSTANCE;

    private static final ILog LOGGER = LogManager.getLogger(ServiceManager.class);
    private Map<Class, BootService> bootedServices = Collections.emptyMap();
    
    public void boot() {
        // 加载所有服务
        bootedServices = loadAllServices();
    
        // 调用服务的生命周期
        prepare();
        startup();
        onComplete();
    }
      
    private Map<Class, BootService> loadAllServices() {
        Map<Class, BootService> bootedServices = new LinkedHashMap<>();
        List<BootService> allServices = new LinkedList<>();
        // 加载所有服务 保存到allServices
        load(allServices);
        for (final BootService bootService : allServices) {
            Class<? extends BootService> bootServiceClass = bootService.getClass();
            boolean isDefaultImplementor = bootServiceClass.isAnnotationPresent(DefaultImplementor.class);
            if (isDefaultImplementor) {
                if (!bootedServices.containsKey(bootServiceClass)) {
                    bootedServices.put(bootServiceClass, bootService);
                } else {
                    //ignore the default service
                }
            } else {
                OverrideImplementor overrideImplementor = bootServiceClass.getAnnotation(OverrideImplementor.class);
                if (overrideImplementor == null) {
                    if (!bootedServices.containsKey(bootServiceClass)) {
                        bootedServices.put(bootServiceClass, bootService);
                    } else {
                        throw new ServiceConflictException("Duplicate service define for :" + bootServiceClass);
                    }
                } else {
                    Class<? extends BootService> targetService = overrideImplementor.value();
                    if (bootedServices.containsKey(targetService)) {
                        boolean presentDefault = bootedServices.get(targetService)
                                                               .getClass()
                                                               .isAnnotationPresent(DefaultImplementor.class);
                        if (presentDefault) {
                            bootedServices.put(targetService, bootService);
                        } else {
                            throw new ServiceConflictException(
                                "Service " + bootServiceClass + " overrides conflict, " + "exist more than one service want to override :" + targetService);
                        }
                    } else {
                        bootedServices.put(targetService, bootService);
                    }
                }
            }
    
        }
        return bootedServices;
    }
      
    void load(List<BootService> allServices) {
        // 通过SPI使用AgentClassLoader加载所有BootService的实现类
        for (final BootService bootService : ServiceLoader.load(BootService.class, AgentClassLoader.getDefault())) {
            allServices.add(bootService);
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
ServiceManager的boot()方法中调用loadAllServices()方法，loadAllServices()方法中调用load()方法通过SPI使用AgentClassLoader加载所有BootService的实现类

在apm-agent-core模块的resources/META-INF目录下有SPI对应的配置文件，其中定义了SKyWalking Agent中所有的服务



3）、服务是如何组织起来的
SKyWalking Agent中的服务会有默认实现和覆盖实现，通常我们的实现逻辑是默认实现（DefaultA）实现A接口（ServiceA），覆盖实现（OverrideA）继承默认实现


SKyWalking中把抽象的接口和默认实现合并，默认实现标注@DefaultImplementor注解，覆盖实现继承默认实现并标注@OverrideImplementor(value=默认实现.class)


以采样服务为例：

默认实现：

@DefaultImplementor
public class SamplingService implements BootService {
1
2
覆盖实现：

@OverrideImplementor(SamplingService.class)
public class TraceIgnoreExtendService extends SamplingService {
1
2
了解了服务的组织形式，我们来看下ServiceManager的loadAllServices()方法加载所有服务后的处理逻辑：

public enum ServiceManager {

    private Map<Class, BootService> loadAllServices() {
        // key:默认实现类名 value:只有默认实现则为默认实现,既有默认实现又有覆盖实现则为覆盖实现
        Map<Class, BootService> bootedServices = new LinkedHashMap<>();
        List<BootService> allServices = new LinkedList<>();
        // 加载所有服务 保存到allServices
        load(allServices);
        for (final BootService bootService : allServices) {
            Class<? extends BootService> bootServiceClass = bootService.getClass();
            boolean isDefaultImplementor = bootServiceClass.isAnnotationPresent(DefaultImplementor.class);
            if (isDefaultImplementor) {
                // 有@DefaultImplementor
                if (!bootedServices.containsKey(bootServiceClass)) {
                    bootedServices.put(bootServiceClass, bootService);
                } else {
                    //ignore the default service
                }
            } else {
                OverrideImplementor overrideImplementor = bootServiceClass.getAnnotation(OverrideImplementor.class);
                if (overrideImplementor == null) {
                    // 既没有@DefaultImplementor 也有@OverrideImplementor
                    if (!bootedServices.containsKey(bootServiceClass)) {
                        bootedServices.put(bootServiceClass, bootService);
                    } else {
                        throw new ServiceConflictException("Duplicate service define for :" + bootServiceClass);
                    }
                } else {
                    // 没有@DefaultImplementor 但是有@OverrideImplementor
                    Class<? extends BootService> targetService = overrideImplementor.value();
                    if (bootedServices.containsKey(targetService)) {
                        // 当前 覆盖实现 要覆盖的 默认实现 已经被加载进来
                        boolean presentDefault = bootedServices.get(targetService)
                                                               .getClass()
                                                               .isAnnotationPresent(DefaultImplementor.class);
                        if (presentDefault) {
                            bootedServices.put(targetService, bootService);
                        } else {
                            throw new ServiceConflictException(
                                "Service " + bootServiceClass + " overrides conflict, " + "exist more than one service want to override :" + targetService);
                        }
                    } else {
                        // 当前 覆盖实现 要覆盖的 默认实现 还没有被加载进来,这时候就把这个 覆盖实现 当做是其服务的 默认实现
                        bootedServices.put(targetService, bootService);
                    }
                }
            }
    
        }
        return bootedServices;
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
4）、调用服务的生命周期
public enum ServiceManager {
    INSTANCE;

    private static final ILog LOGGER = LogManager.getLogger(ServiceManager.class);
    private Map<Class, BootService> bootedServices = Collections.emptyMap();
    
    public void boot() {
        // 加载所有服务
        bootedServices = loadAllServices();
    
        // 调用服务的生命周期
        prepare();
        startup();
        onComplete();
    }
    
    private void prepare() {
        bootedServices.values().stream().sorted(Comparator.comparingInt(BootService::priority)).forEach(service -> {
            try {
                service.prepare();
            } catch (Throwable e) {
                LOGGER.error(e, "ServiceManager try to pre-start [{}] fail.", service.getClass().getName());
            }
        });
    }
    
    private void startup() {
        bootedServices.values().stream().sorted(Comparator.comparingInt(BootService::priority)).forEach(service -> {
            try {
                service.boot();
            } catch (Throwable e) {
                LOGGER.error(e, "ServiceManager try to start [{}] fail.", service.getClass().getName());
            }
        });
    }
    
    private void onComplete() {
        for (BootService service : bootedServices.values()) {
            try {
                service.onComplete();
            } catch (Throwable e) {
                LOGGER.error(e, "Service [{}] AfterBoot process fails.", service.getClass().getName());
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
启动服务时，prepare()、startup()、onComplete()方法根据优先级顺序调用服务的对应方法

public enum ServiceManager {
    INSTANCE;

    private static final ILog LOGGER = LogManager.getLogger(ServiceManager.class);
    private Map<Class, BootService> bootedServices = Collections.emptyMap();
    
    public void shutdown() {
        bootedServices.values().stream().sorted(Comparator.comparingInt(BootService::priority).reversed()).forEach(service -> {
            try {
                service.shutdown();
            } catch (Throwable e) {
                LOGGER.error(e, "ServiceManager try to shutdown [{}] fail.", service.getClass().getName());
            }
        });
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
关闭服务时，根据优先级倒序调用服务的shutdown()方法

小结：



8、witness组件版本识别
1）、为什么需要进行组件版本识别
针对组件依赖包的不同版本，插件增强的处理逻辑不同



比较原始的做法是：插件判断组件对应版本号然后执行对应版本分支的处理逻辑，如上图插件中有很多的if else

这种做法有两个问题：第一如何拿到组件的版本号；第二每次组件出了新的版本都要再新增分支逻辑，扩展性很差

针对第二个问题，SkyWalking做了如下处理：


对组件不同版本的支持作为独立的插件来运行

但这种做法同样需要知道对应组件的版本，比如说有spring-v3-plugin、spring-v4-plugin、spring-v5-plugin三个版本的Spring插件，那在应用程序中使用了Spring Framework 4.x，此时应该是spring-v4-plugin生效，其他两个插件不生效，那这里又是如何知道组件对应的版本呢？

2）、组件版本识别技术
witnessClasses：是否存在一个或多个类仅同时存在于某一个版本中


假设应用中使用的是Spring 3.x，该版本包含A、B两个类不包含C类

当spring-v3-plugin插件生效前，判断应用中同时存在A、B两个类，满足该条件，所以spring-v3-plugin插件生效

当spring-v4-plugin、spring-v5-plugin插件生效前，判断应用中同时存在B、C两个类，不满足该条件，所以spring-v4-plugin、spring-v5-plugin插件不生效

witnessMethods：当witnessClasses判断不出组件版本时，就使用witnessMethods


假设Spring 5.x相对比Spring 4.x并没有新增新的类，不能通过类的差异化判断是Spring 4.x还是Spring 5.x，但是在Spring 4.x的C类中有getCache()方法返回值是int类型参数类型为a、b，在Spring5.x的C类中返回值改为了long类型参数类型改为a

这时候就判断当应用中的C类型包含int getCache(a, b)方法时，spring-v4-plugin插件生效；当应用中的C类型包含long getCache(a)方法时，spring-v5-plugin插件生效

witnessClasses和witnessMethods的作用都是用于识别当前应用使用的组件版本是什么

SKyWalking Agent如何判断witnessClasses中的类是否存在？

插件是由AgentClassLoader加载的，AgentClassLoader的父类加载器是AppClassLoader，通过双亲委派模型AgentClassLoader中找不到witnessClasses就向上委派给AppClassLoader，找不到再向上委派，通过这种方式就能判断witnessClasses中的类是否存在

3）、案例
public abstract class AbstractClassEnhancePluginDefine {

    protected String[] witnessClasses() {
        return new String[] {};
    }
    
    protected List<WitnessMethod> witnessMethods() {
        return null;
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
AbstractClassEnhancePluginDefine是所有插件定义的顶级父类，该类中包含witnessClasses()和witnessMethods()方法，插件定义时会根据需要重写这两个方法

witnessClasses案例：

public abstract class AbstractSpring3Instrumentation extends ClassInstanceMethodsEnhancePluginDefine {

    public static final String WITHNESS_CLASSES = "org.springframework.web.servlet.view.xslt.AbstractXsltView";
    
    @Override
    protected final String[] witnessClasses() {
        return new String[] {WITHNESS_CLASSES};
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
mvc-annotation-3.x-plugin（SpringMVC 3.x版本插件）生效应用中必须包含AbstractXsltView这个类

public abstract class AbstractSpring4Instrumentation extends ClassInstanceMethodsEnhancePluginDefine {
    public static final String WITHNESS_CLASSES = "org.springframework.cache.interceptor.SimpleKey";

    @Override
    protected String[] witnessClasses() {
        return new String[] {
            WITHNESS_CLASSES,
            "org.springframework.cache.interceptor.DefaultKeyGenerator"
        };
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
mvc-annotation-4.x-plugin生效应用中必须同时包含SimpleKey和DefaultKeyGenerator这两个类

public abstract class AbstractSpring5Instrumentation extends ClassInstanceMethodsEnhancePluginDefine {
    public static final String WITNESS_CLASSES = "org.springframework.web.servlet.resource.HttpResource";

    @Override
    protected final String[] witnessClasses() {
        return new String[] {WITNESS_CLASSES};
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
mvc-annotation-5.x-plugin生效应用中必须包含HttpResource这个类

witnessMethods案例：

public class AdapterActionFutureInstrumentation extends ClassEnhancePluginDefine {

    @Override
    protected String[] witnessClasses() {
        return new String[] {Constants.TASK_TRANSPORT_CHANNEL_WITNESS_CLASSES};
    }
    
    @Override
    protected List<WitnessMethod> witnessMethods() {
        return Collections.singletonList(new WitnessMethod(
            Constants.SEARCH_HITS_WITNESS_CLASSES,
            named("getTotalHits").and(takesArguments(0)).and(returns(long.class))
        ));
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
public class AdapterActionFutureInstrumentation extends ClassEnhancePluginDefine {

    @Override
    protected String[] witnessClasses() {
        return new String[] {Constants.TASK_TRANSPORT_CHANNEL_WITNESS_CLASSES};
    }
    
    @Override
    protected List<WitnessMethod> witnessMethods() {
        return Collections.singletonList(new WitnessMethod(
            Constants.SEARCH_HITS_WITNESS_CLASSES,
            named("getTotalHits").and(takesArguments(0)).and(returns(named("org.apache.lucene.search.TotalHits")))
        ));
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
在SKyWalking8.7版本中，witnessMethods仅用于识别elasticsearch6.x版本和7.x版本，6.x版本SearchHits中的getTotalHits()方法返回值为long类型，7.x版本SearchHits中的getTotalHits()方法返回值为TotalHits类型

小结：



9、Transform工作流程
public class SkyWalkingAgent {

    private static class Transformer implements AgentBuilder.Transformer {
        private PluginFinder pluginFinder;
    
        Transformer(PluginFinder pluginFinder) {
            this.pluginFinder = pluginFinder;
        }
    
        @Override
        public DynamicType.Builder<?> transform(final DynamicType.Builder<?> builder, // 当前拦截到的类的字节码
                                                final TypeDescription typeDescription, // 简单当成Class,它包含了类的描述信息
                                                final ClassLoader classLoader, // 加载[当前拦截到的类]的类加载器
                                                final JavaModule module) {
            LoadedLibraryCollector.registerURLClassLoader(classLoader);
            // 1)查找所有能够对当前被拦截到的类生效的插件
            List<AbstractClassEnhancePluginDefine> pluginDefines = pluginFinder.find(typeDescription);
            if (pluginDefines.size() > 0) {
                DynamicType.Builder<?> newBuilder = builder;
                // 2)增强上下文
                EnhanceContext context = new EnhanceContext();
                for (AbstractClassEnhancePluginDefine define : pluginDefines) {
                    // 3)调用每个插件的define()方法去做字节码增强
                    DynamicType.Builder<?> possibleNewBuilder = define.define(
                            typeDescription, newBuilder, classLoader, context);
                    if (possibleNewBuilder != null) {
                        newBuilder = possibleNewBuilder;
                    }
                }
                if (context.isEnhanced()) {
                    LOGGER.debug("Finish the prepare stage for {}.", typeDescription.getName());
                }
    
                return newBuilder; // 被所有可用插件修改完之后的最终字节码
            }
    
            LOGGER.debug("Matched class {}, but ignore by finding mechanism.", typeDescription.getTypeName());
            return builder; // 被拦截到的类的原生字节码
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
transform()方法处理逻辑如下：

查找所有能够对当前被拦截到的类生效的插件
如果有生效的插件，调用每个插件的define()方法去做字节码增强，返回被所有可用插件修改完之后的最终字节码
如果没有生效的插件，返回被拦截到的类的原生字节码
代码1)处先调用PluginFinder的find()方法查找所有能够对当前被拦截到的类生效的插件，这块在讲解插件加载时详细讲过：

public class PluginFinder {
    /**
     * 为什么这里的Map泛型是<String,LinkedList>
     * 因为对于同一个类,可能有多个插件都要对它进行字节码增强
     * key => 目标类
     * value => 所有可以对这个目标类生效的插件
     */
    private final Map<String, LinkedList<AbstractClassEnhancePluginDefine>> nameMatchDefine = new HashMap<String, LinkedList<AbstractClassEnhancePluginDefine>>();
    private final List<AbstractClassEnhancePluginDefine> signatureMatchDefine = new ArrayList<AbstractClassEnhancePluginDefine>();

    /**
     * 查找所有能够对指定类型生效的插件
     * 1.从命名插件里找
     * 2.从间接匹配插件里找
     *
     * @param typeDescription 可以看做是class
     * @return
     */
    public List<AbstractClassEnhancePluginDefine> find(TypeDescription typeDescription) {
        List<AbstractClassEnhancePluginDefine> matchedPlugins = new LinkedList<AbstractClassEnhancePluginDefine>();
        String typeName = typeDescription.getTypeName();
        if (nameMatchDefine.containsKey(typeName)) {
            matchedPlugins.addAll(nameMatchDefine.get(typeName));
        }
    
        for (AbstractClassEnhancePluginDefine pluginDefine : signatureMatchDefine) {
            IndirectMatch match = (IndirectMatch) pluginDefine.enhanceClass();
            if (match.isMatch(typeDescription)) {
                matchedPlugins.add(pluginDefine);
            }
        }
    
        return matchedPlugins;
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
代码2)处用到了增强上下文EnhanceContext：

/**
 * 状态记录类,记录当前被拦截到的类 是否被改了字节码 和 是否新增了新的字段或接口
 * The <code>EnhanceContext</code> represents the context or status for processing a class.
 * <p>
 * Based on this context, the plugin core {@link ClassEnhancePluginDefine} knows how to process the specific steps for
 * every particular plugin.
 */
 public class EnhanceContext {
    /**
     * 是否被增强
     */
     private boolean isEnhanced = false;
     /**
     * 是否新增了新的字段或者实现了新的接口
     * The object has already been enhanced or extended. e.g. added the new field, or implemented the new interface
     */
     private boolean objectExtended = false;

    public boolean isEnhanced() {
        return isEnhanced;
    }

    public void initializationStageCompleted() {
        isEnhanced = true;
    }

    public boolean isObjectExtended() {
        return objectExtended;
    }

    public void extendObjectCompleted() {
        objectExtended = true;
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
 代码3)处调用每个插件的define()方法去做字节码增强，实际调用了AbstractClassEnhancePluginDefine的define()方法：

public abstract class AbstractClassEnhancePluginDefine {
    private static final ILog LOGGER = LogManager.getLogger(AbstractClassEnhancePluginDefine.class);

    /**
     * Main entrance of enhancing the class.
     *
     * @param typeDescription target class description.
     * @param builder         byte-buddy's builder to manipulate target class's bytecode.
     * @param classLoader     load the given transformClass
     * @return the new builder, or <code>null</code> if not be enhanced.
     * @throws PluginException when set builder failure.
     */
    public DynamicType.Builder<?> define(TypeDescription typeDescription, DynamicType.Builder<?> builder,
        ClassLoader classLoader, EnhanceContext context) throws PluginException {
        // 当前插件的全类名
        String interceptorDefineClassName = this.getClass().getName();
        // 当前被拦截到的类的全类名
        String transformClassName = typeDescription.getTypeName();
        if (StringUtil.isEmpty(transformClassName)) {
            LOGGER.warn("classname of being intercepted is not defined by {}.", interceptorDefineClassName);
            return null;
        }
    
        LOGGER.debug("prepare to enhance class {} by {}.", transformClassName, interceptorDefineClassName);
        // witness机制校验当前插件是否可用
        WitnessFinder finder = WitnessFinder.INSTANCE;
        /**
         * find witness classes for enhance class
         */
        String[] witnessClasses = witnessClasses();
        if (witnessClasses != null) {
            for (String witnessClass : witnessClasses) {
              	// 代码1)
                if (!finder.exist(witnessClass, classLoader)) {
                    LOGGER.warn("enhance class {} by plugin {} is not working. Because witness class {} is not existed.", transformClassName, interceptorDefineClassName, witnessClass);
                    return null;
                }
            }
        }
        List<WitnessMethod> witnessMethods = witnessMethods();
        if (!CollectionUtil.isEmpty(witnessMethods)) {
            for (WitnessMethod witnessMethod : witnessMethods) {
              	// 代码2)
                if (!finder.exist(witnessMethod, classLoader)) {
                    LOGGER.warn("enhance class {} by plugin {} is not working. Because witness method {} is not existed.", transformClassName, interceptorDefineClassName, witnessMethod);
                    return null;
                }
            }
        }
    
        /**
         * 字节码增强流程
         * find origin class source code for interceptor
         */
        DynamicType.Builder<?> newClassBuilder = this.enhance(typeDescription, builder, classLoader, context);
    
        // 将记录状态的上下文EnhanceContext设置为已增强
        context.initializationStageCompleted();
        LOGGER.debug("enhance class {} by {} completely.", transformClassName, interceptorDefineClassName);
    
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
define()方法处理逻辑如下：

witness机制校验当前插件是否可用
调用enhance()方法进行字节码增强流程
将记录状态的上下文EnhanceContext设置为已增强
代码1)处和代码2)处调用WitnessFinder的exist()方法判断witnessClass和witnessMethod是否存在，从而判断当前插件是否可用

public enum WitnessFinder {
    INSTANCE;

    /**
     * key:ClassLoader value:这个ClassLoader的类型池,也就是这个ClassLoader所有能加载的类
     */
    private final Map<ClassLoader, TypePool> poolMap = new HashMap<ClassLoader, TypePool>();
    
    /**
     * @param classLoader for finding the witnessClass
     * @return true, if the given witnessClass exists, through the given classLoader.
     */
    public boolean exist(String witnessClass, ClassLoader classLoader) {
        return getResolution(witnessClass, classLoader)
                .isResolved();
    }
    
    /**
     * get TypePool.Resolution of the witness class
     * @param witnessClass class name
     * @param classLoader classLoader for finding the witnessClass
     * @return TypePool.Resolution
     */
    private TypePool.Resolution getResolution(String witnessClass, ClassLoader classLoader) {
        ClassLoader mappingKey = classLoader == null ? NullClassLoader.INSTANCE : classLoader;
        if (!poolMap.containsKey(mappingKey)) {
            synchronized (poolMap) {
                if (!poolMap.containsKey(mappingKey)) {
                    // classLoader == null,基于BootStrapClassLoader构造TypePool
                    // 否则基于自身的classLoader构造TypePool
                    TypePool classTypePool = classLoader == null ? TypePool.Default.ofBootLoader() : TypePool.Default.of(classLoader);
                    poolMap.put(mappingKey, classTypePool);
                }
            }
        }
        TypePool typePool = poolMap.get(mappingKey);
        // 判断传入的类是否存在
        return typePool.describe(witnessClass);
    }
    
    /**
     * @param classLoader for finding the witness method
     * @return true, if the given witness method exists, through the given classLoader.
     */
    public boolean exist(WitnessMethod witnessMethod, ClassLoader classLoader) {
        // 方法所在的类是否在这个ClassLoader中
        TypePool.Resolution resolution = getResolution(witnessMethod.getDeclaringClassName(), classLoader);
        if (!resolution.isResolved()) {
            return false;
        }
        // 判断该方法是否存在
        return !resolution.resolve()
                .getDeclaredMethods()
                .filter(witnessMethod.getElementMatcher())
                .isEmpty();
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
witnessClass校验时，会基于传入的classLoader构造TypePool来判断witnessClass是否存在，TypePool最终会存储到Map中
witnessMethod校验时，先判断该方法所在的类是否在这个ClassLoader中（走witnessClass校验的流程），再判断该方法是否存在
小结：



参考：

SkyWalking8.7.0源码分析（如果你对SkyWalking Agent源码感兴趣的话，强烈建议看下该教程）
