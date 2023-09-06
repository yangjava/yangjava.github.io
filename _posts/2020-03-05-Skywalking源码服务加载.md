---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码服务加载
SkyWalking Agent中的服务就是实现了org.apache.skywalking.apm.agent.core.boot.BootService接口的类，这个接口定义了一个服务的生命周期

## 加载服务
SkyWalking Agent收集硬件、JVM监控指标以及指标上报等都是一个个服务模块组织起来的。

SkyWalking Agent中的服务就是实现了org.apache.skywalking.apm.agent.core.boot.BootService接口的类，这个接口定义了一个服务的生命周期
```
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
```

### SPI加载服务
通过ServiceManager加载所有的服务。ServiceManager基于ServerLoader实现，是JDK提供的一种SPI机制。

premain()中调用ServiceManager的boot()方法加载并启动服务，源码如下：
```
        try {
            ServiceManager.INSTANCE.boot();
        } catch (Exception e) {
            LOGGER.error(e, "Skywalking agent boot failure.");
        }
```


boot()方法：
```
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
```

bootedServices是什么？
```
 private Map<Class, BootService> bootedServices = Collections.emptyMap();
```
其中的BootService是一个接口，其包含了服务的生命周期，插件开始工作时，需要被启动，如下所示：
```
    public interface BootService {
        /**
         * 准备
         * @throws Throwable
         */
        void prepare() throws Throwable;
        /**
         * 启动
         * @throws Throwable
         */
        void boot() throws Throwable;
        /**
         * 完成
         * @throws Throwable
         */
        void onComplete() throws Throwable;
        /**
         * 停止
         * @throws Throwable
         */
        void shutdown() throws Throwable;

        /**
         * 优先级，优先级高的BootService会优先启动
         */
        default int priority() {
            return 0;
        }
    }
```
所有实现了BootService接口的服务都会在此被加载。

## loadAllServices()

这个方法通过jdk提供的SPI机制，ServiceLoader将需要的类加载进来，而这些类被定义在指定的配置文件当中：

配置文件内容如下：
```
    org.apache.skywalking.apm.agent.core.remote.TraceSegmentServiceClient
    org.apache.skywalking.apm.agent.core.context.ContextManager
    org.apache.skywalking.apm.agent.core.sampling.SamplingService
    org.apache.skywalking.apm.agent.core.remote.GRPCChannelManager
    org.apache.skywalking.apm.agent.core.jvm.JVMMetricsSender
    org.apache.skywalking.apm.agent.core.jvm.JVMService
    org.apache.skywalking.apm.agent.core.remote.ServiceManagementClient
    org.apache.skywalking.apm.agent.core.context.ContextManagerExtendService
    org.apache.skywalking.apm.agent.core.commands.CommandService
    org.apache.skywalking.apm.agent.core.commands.CommandExecutorService
    org.apache.skywalking.apm.agent.core.profile.ProfileTaskChannelService
    org.apache.skywalking.apm.agent.core.profile.ProfileSnapshotSender
    org.apache.skywalking.apm.agent.core.profile.ProfileTaskExecutionService
    org.apache.skywalking.apm.agent.core.meter.MeterService
    org.apache.skywalking.apm.agent.core.meter.MeterSender
    org.apache.skywalking.apm.agent.core.context.status.StatusCheckService
    org.apache.skywalking.apm.agent.core.remote.LogReportServiceClient
    org.apache.skywalking.apm.agent.core.conf.dynamic.ConfigurationDiscoveryService
    org.apache.skywalking.apm.agent.core.remote.EventReportServiceClient
    org.apache.skywalking.apm.agent.core.ServiceInstanceGenerator
```
通过下面的方法将上面配置的所有实现了bootService的类加载到allServices当中。
```
    void load(List<BootService> allServices) {
        for (final BootService bootService : ServiceLoader.load(BootService.class, AgentClassLoader.getDefault())) {
            allServices.add(bootService);
        }
    }
```

在上一步加载完全部的类之后，需要去遍历这些类，得到一个bootedServices的集合。在看代码逻辑之前，需要看下skywalking定义的两个注解：
- @DefaultImplementor 默认实现
- @OverrideImplementor 覆盖实现
  带有@DefaultImplementor注解的类，表示它会有类去继承它，继承它的类需要带有 @OverrideImplementor注解，并指定继承的类的名称，例如：

默认实现类：
```
    @DefaultImplementor
    public class JVMMetricsSender implements BootService, Runnable, GRPCChannelListener 
```
继承它的类：
```
    @OverrideImplementor(JVMMetricsSender.class)
    public class KafkaJVMMetricsSender extends JVMMetricsSender implements KafkaConnectionStatusListener 
```

在了解了skywalking的默认类和继承类的机制后，有代码逻辑如下：
```
    private Map<Class, BootService> loadAllServices() {
        Map<Class, BootService> bootedServices = new LinkedHashMap<>();
        List<BootService> allServices = new LinkedList<>();
        // SPI加载
        load(allServices);
        // 遍历
        for (final BootService bootService : allServices) {
            Class<? extends BootService> bootServiceClass = bootService.getClass();
            // 是否带有默认实现
            boolean isDefaultImplementor = bootServiceClass.isAnnotationPresent(DefaultImplementor.class);
            if (isDefaultImplementor) {// 是默认实现
                // 是默认实现，没有添加到bootedServices
                if (!bootedServices.containsKey(bootServiceClass)) {
                    //加入bootedServices
                    bootedServices.put(bootServiceClass, bootService);
                } else {
                    //ignore the default service
                }
            } else {// 不是默认实现
                // 是否是覆盖实现
                OverrideImplementor overrideImplementor = bootServiceClass.getAnnotation(OverrideImplementor.class);
                // 不是覆盖
                if (overrideImplementor == null) {
                    // bootedServices没有
                    if (!bootedServices.containsKey(bootServiceClass)) {
                        //加入bootedServices
                        bootedServices.put(bootServiceClass, bootService);
                    } else {
                        throw new ServiceConflictException("Duplicate service define for :" + bootServiceClass);
                    }
                } else {
                    // 是覆盖，value获取的是其继承的类targetService
                    Class<? extends BootService> targetService = overrideImplementor.value();
                    // 如果bootedServices已经包含targetService
                    if (bootedServices.containsKey(targetService)) {
                        // 判断targetServices是否是默认实现
                        boolean presentDefault = bootedServices.get(targetService)
                                                               .getClass()
                                                               .isAnnotationPresent(DefaultImplementor.class);
                        // 是默认实现
                        if (presentDefault) {
                            // 添加进去
                            bootedServices.put(targetService, bootService);
                        } else {
                            // 没有默认实现，不能覆盖，抛出异常
                            throw new ServiceConflictException(
                                "Service " + bootServiceClass + " overrides conflict, " + "exist more than one service want to override :" + targetService);
                        }
                    } else {
                        // 是覆盖实现，它覆盖的默认实现还没有被加载进来
                        bootedServices.put(targetService, bootService);
                    }
                }
            }

        }
        return bootedServices;
    }
```

prepare(),startup(),onComplete()

在加载完全部的类之后，还有准备，启动和完成等分操作，它们的代码实现相同，如下所示：
```
private void prepare() {
    // 获取所有的类
    bootedServices.values().stream()
            // 根据优先级排序，BootService的priority
            .sorted(Comparator.comparingInt(BootService::priority))
            // 遍历
            .forEach(service -> {
        try {
            // 执行每一个BootService的实现类的prepare()方法
            service.prepare();
        } catch (Throwable e) {
            LOGGER.error(e, "ServiceManager try to pre-start [{}] fail.", service.getClass().getName());
        }
    });
}

private void startup() {
    bootedServices.values().stream()
            // 根据优先级排序，BootService的priority
            .sorted(Comparator.comparingInt(BootService::priority))
            // 遍历
            .forEach(service -> {
        try {
            // 执行每一个BootService的实现类的boot()方法
            service.boot();
        } catch (Throwable e) {
            LOGGER.error(e, "ServiceManager try to start [{}] fail.", service.getClass().getName());
        }
    });
}

private void onComplete() {
    // 遍历
    for (BootService service : bootedServices.values()) {
        try {
            // 执行每一个BootService的实现类的onComplete()方法
            service.onComplete();
        } catch (Throwable e) {
            LOGGER.error(e, "Service [{}] AfterBoot process fails.", service.getClass().getName());
        }
    }
}
```

## ShutdownHook
为skywalking的运行服务添加一个shutdown的钩子。
```
Runtime.getRuntime()
        .addShutdownHook(new Thread(ServiceManager.INSTANCE::shutdown, "skywalking service shutdown thread"));
```
shutdown方法，与准备和启动方法不同之处在于，shutdown的排序方式是按照优先级的倒序排序，为了优雅的停机，后启动的服务，先停机：
```
public void shutdown() {
    bootedServices.values().stream().
            // 排序，按照优先级的倒序
            sorted(Comparator.comparingInt(BootService::priority).reversed())
            .forEach(service -> {
        try {
            // 执行每个服务的shutdown
            service.shutdown();
        } catch (Throwable e) {
            LOGGER.error(e, "ServiceManager try to shutdown [{}] fail.", service.getClass().getName());
        }
    });
}
```

## witness组件版本识别

### 为什么需要进行组件版本识别
针对组件依赖包的不同版本，插件增强的处理逻辑不同。比较原始的做法是：插件判断组件对应版本号然后执行对应版本分支的处理逻辑，如上图插件中有很多的if else

这种做法有两个问题：第一如何拿到组件的版本号；第二每次组件出了新的版本都要再新增分支逻辑，扩展性很差

针对第二个问题，SkyWalking做了如下处理： 对组件不同版本的支持作为独立的插件来运行

但这种做法同样需要知道对应组件的版本，比如说有spring-v3-plugin、spring-v4-plugin、spring-v5-plugin三个版本的Spring插件，那在应用程序中使用了Spring Framework 4.x，此时应该是spring-v4-plugin生效，其他两个插件不生效，那这里又是如何知道组件对应的版本呢？

### 组件版本识别技术
witnessClasses：是否存在一个或多个类仅同时存在于某一个版本中。

假设应用中使用的是Spring 3.x，该版本包含A、B两个类不包含C类

当spring-v3-plugin插件生效前，判断应用中同时存在A、B两个类，满足该条件，所以spring-v3-plugin插件生效

当spring-v4-plugin、spring-v5-plugin插件生效前，判断应用中同时存在B、C两个类，不满足该条件，所以spring-v4-plugin、spring-v5-plugin插件不生效

witnessMethods：当witnessClasses判断不出组件版本时，就使用witnessMethods

假设Spring 5.x相对比Spring 4.x并没有新增新的类，不能通过类的差异化判断是Spring 4.x还是Spring 5.x，但是在Spring 4.x的C类中有getCache()方法返回值是int类型参数类型为a、b，在Spring5.x的C类中返回值改为了long类型参数类型改为a

这时候就判断当应用中的C类型包含int getCache(a, b)方法时，spring-v4-plugin插件生效；当应用中的C类型包含long getCache(a)方法时，spring-v5-plugin插件生效

witnessClasses和witnessMethods的作用都是用于识别当前应用使用的组件版本是什么

SKyWalking Agent如何判断witnessClasses中的类是否存在？

插件是由AgentClassLoader加载的，AgentClassLoader的父类加载器是AppClassLoader，通过双亲委派模型AgentClassLoader中找不到witnessClasses就向上委派给AppClassLoader，找不到再向上委派，通过这种方式就能判断witnessClasses中的类是否存在

案例
```
public abstract class AbstractClassEnhancePluginDefine {

    protected String[] witnessClasses() {
        return new String[] {};
    }

    protected List<WitnessMethod> witnessMethods() {
        return null;
    }
```
AbstractClassEnhancePluginDefine是所有插件定义的顶级父类，该类中包含witnessClasses()和witnessMethods()方法，插件定义时会根据需要重写这两个方法

witnessClasses案例：
```
public abstract class AbstractSpring3Instrumentation extends ClassInstanceMethodsEnhancePluginDefine {

    public static final String WITHNESS_CLASSES = "org.springframework.web.servlet.view.xslt.AbstractXsltView";

    @Override
    protected final String[] witnessClasses() {
        return new String[] {WITHNESS_CLASSES};
    }
}
```

mvc-annotation-3.x-plugin（SpringMVC 3.x版本插件）生效应用中必须包含AbstractXsltView这个类
```
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
```

mvc-annotation-4.x-plugin生效应用中必须同时包含SimpleKey和DefaultKeyGenerator这两个类
```
public abstract class AbstractSpring5Instrumentation extends ClassInstanceMethodsEnhancePluginDefine {
    public static final String WITNESS_CLASSES = "org.springframework.web.servlet.resource.HttpResource";

    @Override
    protected final String[] witnessClasses() {
        return new String[] {WITNESS_CLASSES};
    }
}
```
mvc-annotation-5.x-plugin生效应用中必须包含HttpResource这个类

witnessMethods案例：
```
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
```

```
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
```
在SKyWalking8.7版本中，witnessMethods仅用于识别elasticsearch6.x版本和7.x版本，6.x版本SearchHits中的getTotalHits()方法返回值为long类型，7.x版本SearchHits中的getTotalHits()方法返回值为TotalHits类型
















