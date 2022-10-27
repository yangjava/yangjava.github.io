---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---

#### **源码解读-服务启动**

SkyWalkingAgent#premain方法：

```java
	//主要入口。使用byte-buddy 转换来增强所有在插件中定义的类。
    public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException {
       
       	//（1）配置项加载
        …………
        //（2）加载插件，并进行分类  
        …………
        //（3）设置 ByteBuddy
        // skyWalking 使用了ByteBuddy技术来进行字节码增强
        // IS_OPEN_DEBUGGING_CLASS 是否开启debug模式。 当为true时，会把增强过得字节码文件放到/debugging文件夹下，方便debug。
        final ByteBuddy byteBuddy = new ByteBuddy().with(TypeValidation.of(Config.Agent.IS_OPEN_DEBUGGING_CLASS));
		//AgentBuilder提供了便捷的API，用于定义Java agent.
        AgentBuilder agentBuilder = new AgentBuilder.Default(byteBuddy)
                .ignore( //指定以这些类名为开头的 不属于要增强的范围
                nameStartsWith("net.bytebuddy.")
                        .or(nameStartsWith("org.slf4j."))
                        .or(nameStartsWith("org.groovy."))
                        .or(nameContains("javassist"))
                        .or(nameContains(".asm."))
                        .or(nameContains(".reflectasm."))
                        .or(nameStartsWith("sun.reflect"))
                        .or(allSkyWalkingAgentExcludeToolkit())
                        .or(ElementMatchers.isSynthetic()));

      …………

        agentBuilder.type(pluginFinder.buildMatch())// 我们要通过插件增强的类，buildMatch解释见1.1
                    .transform(new Transformer(pluginFinder)) //Transformer 实际增强的方法，下期讲解
                    .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
                    .with(new Listener())//监听器，打一些日志
                    .installOn(instrumentation);

        //（4）加载服务
        try {
            // 插件架构
            // agent-core 看做是内核
            // 所谓的服务就是各种插件，由core内核进行管理
            // 比如maven 就是插件架构
            // 插件架构的规范就是BootService，详见1.2
            ServiceManager.INSTANCE.boot();
        } catch (Exception e) {
            LOGGER.error(e, "Skywalking agent boot failure.");
        }
         //（5）注册关闭钩子
		// 注册一个JVM 的关闭钩子，当服务关闭时，调用shutdown方法释放资源。
        Runtime.getRuntime()
                .addShutdownHook(new Thread(ServiceManager.INSTANCE::shutdown, "skywalking service shutdown thread"));
    }
```

pluginFinder.buildMatch()详解

```java
//把所有需要增强的类构建成ElementMatcher
public ElementMatcher<? super TypeDescription> buildMatch() {
    ElementMatcher.Junction judge = new AbstractJunction<NamedElement>() {
        @Override
        public boolean matches(NamedElement target) {
            return nameMatchDefine.containsKey(target.getActualName());
        }
    };
    judge = judge.and(not(isInterface()));
    for (AbstractClassEnhancePluginDefine define : signatureMatchDefine) {
        ClassMatch match = define.enhanceClass();
        if (match instanceof IndirectMatch) {
            judge = judge.or(((IndirectMatch) match).buildJunction());
        }
    }
    // 注意 最后返回的是 ProtectiveShieldMatcher。ProtectiveShieldMatcher继承了ElementMatcher
    // 使用ProtectiveShieldMatcher的原因：
    /**
    * 在某些情况下，某些框架和库也使用binary技术。
    * 从社区反馈中，有些与byte-buddy有兼容的问题，会触发了“Can't resolve type description”异常。 
    * 因此，嵌套了一层。当原始matcher无法解析类型时，SkyWalking代理会忽略此类型。 
    * 注意：此忽略机制可能会遗漏某些检测手段，但在大多数情况下是没问题的。如果丢失，请注意警告日志。
    */
    return new ProtectiveShieldMatcher(judge);
}
```

ServiceManager.*INSTANCE*.boot();讲解

点进boot方法之后可以看到ServiceManager的代码如下：

```java
/**
 * agent 服务插件定义体系
 * 1. 所有的服务必须直接或者间接实现 BootService
 * 2. 使用@DefaultImplementor 表示一个服务的默认实现，使用@OverrideImplementor 表示一个服务的覆盖实现
 * 3. @OverrideImplementor 的 value 属性用于指定要覆盖服务的默认实现
 * 4. 覆盖实现必须明确指定一个默认实现，且只能覆盖默认实现，也就是说，一个覆盖实现不能去覆盖另一个覆盖实现。
 */
public enum ServiceManager {
    INSTANCE;

    private static final ILog LOGGER = LogManager.getLogger(ServiceManager.class);
    //BootService 见1.2.1
    private Map<Class, BootService> bootedServices = Collections.emptyMap();

    public void boot() {
        //加载所有的bootedServices 
        bootedServices = loadAllServices();
		//执行各生命周期方法
        prepare();
        startup();
        onComplete();
    }

    public void shutdown() {
        for (BootService service : bootedServices.values()) {
            try {
                service.shutdown();
            } catch (Throwable e) {
                LOGGER.error(e, "ServiceManager try to shutdown [{}] fail.", service.getClass().getName());
            }
        }
    }

    /**
     * 
     * @return
     */
    private Map<Class, BootService> loadAllServices() {
        Map<Class, BootService> bootedServices = new LinkedHashMap<>();
        List<BootService> allServices = new LinkedList<>();
        //加载所有服务
        load(allServices);
        for (final BootService bootService : allServices) {
            Class<? extends BootService> bootServiceClass = bootService.getClass();
            boolean isDefaultImplementor = bootServiceClass.isAnnotationPresent(DefaultImplementor.class);
            if (isDefaultImplementor) { // 有@DefaultImplementor注解
                if (!bootedServices.containsKey(bootServiceClass)) {
                    bootedServices.put(bootServiceClass, bootService);
                } else {
                    //ignore the default service
                }
            } else { // 没有@DefaultImplementor注解
                OverrideImplementor overrideImplementor = bootServiceClass.getAnnotation(OverrideImplementor.class);
                if (overrideImplementor == null) { //既没有@DefaultImplementor注解，又没有@DefaultImplementor注解
                    if (!bootedServices.containsKey(bootServiceClass)) {
                        bootedServices.put(bootServiceClass, bootService);
                    } else {
                        throw new ServiceConflictException("Duplicate service define for :" + bootServiceClass);
                    }
                } else { //有@DefaultImplementor注解
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

    private void prepare() {
        for (BootService service : bootedServices.values()) {
            try {
                service.prepare();
            } catch (Throwable e) {
                LOGGER.error(e, "ServiceManager try to pre-start [{}] fail.", service.getClass().getName());
            }
        }
    }

    private void startup() {
        for (BootService service : bootedServices.values()) {
            try {
                service.boot();
            } catch (Throwable e) {
                LOGGER.error(e, "ServiceManager try to start [{}] fail.", service.getClass().getName());
            }
        }
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
```

BootService 代码

```java
/**
 * 是所有远程接口，当插件机制开始起作用时，需要启动该接口。 
 * 
 */
public interface BootService {
    void prepare() throws Throwable;

   	//boot方法将在 BootService 启动时被调用。
    void boot() throws Throwable;

    void onComplete() throws Throwable;

    void shutdown() throws Throwable;
}
```

@DefaultImplementor注解和@OverrideImplementor注解

@DefaultImplementor注解表示这是一个默认服务实现

@OverrideImplementor注解 表示这是一个服务的覆盖实现，覆盖的服务由value指定。