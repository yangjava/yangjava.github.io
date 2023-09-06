---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码探针启动
SkyWalking探针表示集成到目标系统中的代理或SDK库, 它负责收集遥测数据, 包括链路追踪和性能指标。根据目标系统的技术栈, 探针可能有差异巨大的方式来达到以上功能。 但从根本上来说都是一样的, 即收集并格式化数据, 并发送到后端。

## Skywalking探针方案
Skywalking Java Agen 使用 Java premain 作为 Agent 的技术方案，关于 Java Agent，其实有 2 种，一种是以 premain 作为挂载方式（启动时挂载），另外一种是以 agentmain 作为挂载方式，在程序运行期间随时挂载，例如著名的 arthas 就是使用的该方案；agentmain 会更加灵活，但局限会比 premain 多，例如不能增减父类，不能增加接口，新增的方法只能是 private static/final 的，不能修改字段，类访问符不能变化。而 premian 则没有这些限制。

Skywalking 是在 premian 方法中类加载时修改字节码的。使用 ByteBuddy 类库（基于 ASM）实现字节码插桩修改。入口类`SkyWalkingAgent#premain`。

Skywalking Agent 整体结构基于微内核的方式，即插件化，apm-agent-core 是核心代码，负责启动，加载配置，加载插件，修改字节码，记录调用数据，发送到后端等等。而 apm-sdk-plugin 模块则是各个中间件的插装插件，比如 Jedis，Dubbo，RocketMQ，Kafka 等各种客户端。
如果想要实现一个中间件的监控，只需要遵守 Skywalking 的插件规范，编写一个 Maven 模块就可以。Skywalking 内核会自动化的加载插件，并插桩字节码。
Skywalking 的作者曾说：不管是 Linux，Istio 还是 SkyWalking ，都有一个很大的特点：当项目被「高度模块化」之后，贡献者就会开始急剧的提高。
而模块化，插件化，也是一个软件不容易腐烂的重要特性。Skywalking 的就是遵循这个理念设计。

## Skywalking启动流程
Skywalking的原理是java-agent，所以整个核心的启动方法也就是premain方法，Skywalking启动入口`org.apache.skywalking.apm.agent.SkyWalkingAgent#premain`。主要执行代码如下：
```java
/**
 * The main entrance of sky-walking agent, based on javaagent mechanism.
 * agentArgs: -javaagent:/path/to/agent.jar=agentArgs,配置参数后得参数
 * Instrumentation：插庄服务的接口，在计算机科学技术中的英文释义是插桩、植入
 */
// skywalking-agent的入口，基于javaagent原理
public class SkyWalkingAgent {
    // SkyWalkingAgent日志实现
    private static ILog LOGGER = LogManager.getLogger(SkyWalkingAgent.class);

    /**
     * Main entrance. Use byte-buddy transform to enhance all classes, which define in plugins.
     */
    // Main入口. 使用byte-buddy字节码增强plugins
    public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException {
        final PluginFinder pluginFinder;
        try {
            // 初始化配置文件信息
            SnifferConfigInitializer.initializeCoreConfig(agentArgs);
        } catch (Exception e) {
            // try to resolve a new logger, and use the new logger to write the error log here
            // 初始化配置异常,打印日志信息
            LogManager.getLogger(SkyWalkingAgent.class)
                    .error(e, "SkyWalking agent initialized failure. Shutting down.");
            return;
        } finally {
            // refresh logger again after initialization finishes
            LOGGER = LogManager.getLogger(SkyWalkingAgent.class);
        }

        try {
            // 加载所有插件
            pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
        } catch (AgentPackageNotFoundException ape) {
            LOGGER.error(ape, "Locate agent.jar failure. Shutting down.");
            return;
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent initialized failure. Shutting down.");
            return;
        }

        // 使用ByteBuddy技术来进行字节码增强
        // IS_OPEN_DEBUGGING_CLASS 是否开启debug模式。 当为true时，会把增强过得字节码文件放到/debugging文件夹下，方便debug。
        final ByteBuddy byteBuddy = new ByteBuddy().with(TypeValidation.of(Config.Agent.IS_OPEN_DEBUGGING_CLASS));
        // ByteBuddy增强，忽略某个类
        AgentBuilder agentBuilder = new AgentBuilder.Default(byteBuddy).ignore(
                // 指定以这些类名为开头的 不属于要增强的范围
                nameStartsWith("net.bytebuddy.")
                        .or(nameStartsWith("org.slf4j."))
                        .or(nameStartsWith("org.groovy."))
                        .or(nameContains("javassist"))
                        .or(nameContains(".asm."))
                        .or(nameContains(".reflectasm."))
                        .or(nameStartsWith("sun.reflect"))
                        .or(allSkyWalkingAgentExcludeToolkit())
                        .or(ElementMatchers.isSynthetic()));

        JDK9ModuleExporter.EdgeClasses edgeClasses = new JDK9ModuleExporter.EdgeClasses();
        try {
            agentBuilder = BootstrapInstrumentBoost.inject(pluginFinder, instrumentation, agentBuilder, edgeClasses);
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent inject bootstrap instrumentation failure. Shutting down.");
            return;
        }

        try {
            agentBuilder = JDK9ModuleExporter.openReadEdge(instrumentation, agentBuilder, edgeClasses);
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent open read edge in JDK 9+ failure. Shutting down.");
            return;
        }

        if (Config.Agent.IS_CACHE_ENHANCED_CLASS) {
            try {
                agentBuilder = agentBuilder.with(new CacheableTransformerDecorator(Config.Agent.CLASS_CACHE_MODE));
                LOGGER.info("SkyWalking agent class cache [{}] activated.", Config.Agent.CLASS_CACHE_MODE);
            } catch (Exception e) {
                LOGGER.error(e, "SkyWalking agent can't active class cache.");
            }
        }
        // 通过插件增强的类
        agentBuilder.type(pluginFinder.buildMatch())
                //Transformer 实际增强的方法
                .transform(new Transformer(pluginFinder))
                .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
                // 监听器
                .with(new RedefinitionListener())
                .with(new Listener())
                .installOn(instrumentation);

        try {
            // 加载服务
            ServiceManager.INSTANCE.boot();
        } catch (Exception e) {
            LOGGER.error(e, "Skywalking agent boot failure.");
        }
        // 注册JVM的关闭钩子，当服务关闭时，调用shutdown方法释放资源。
        Runtime.getRuntime()
                .addShutdownHook(new Thread(ServiceManager.INSTANCE::shutdown, "skywalking service shutdown thread"));
    }
}
```
源码主要流程
- 初始化配置，加载所有的配置项
- 查找并解析SkyWalking-plugin.def插件文件；使用java spi 找到插件加载，创建描述文件，并初始化到容器中。
- 设置agent byteBuddy 用于动态创建字节码对象。过滤非必要的包名
- 加载到的所有插件，初始化
- buildMatch命中的类 判断是否需要动态字节码代理
- 启动服务，监控所有的匹配的插件服务，然后执行prepare、startup、onComplete
- jvm 关闭钩子

### 初始化配置，加载所有的配置项
```
            // 初始化配置文件信息
            SnifferConfigInitializer.initializeCoreConfig(agentArgs);
```
SkyWalking Java Agent 在 premain 方法中首先做的就是通过`SnifferConfigInitializer.initializeCoreConfig(agentArgs)`初始化核心配置。

加载所有的Skywalking配置信息，转变为Skywalking中的静态`Config`类，以后获取参数可以通过Config配置类获取。

#### 加载所有的插件
```
	//主要入口。使用byte-buddy 转换来增强所有在插件中定义的类。
    public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException {
        final PluginFinder pluginFinder;
       	//配置项加载完成后
        …………
         //加载插件  
		 try {
             //PluginBootstrap().loadPlugins() 加载所有的插件定义
             //pluginFinder详见 查找插件
            pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
        } catch (AgentPackageNotFoundException ape) {
            LOGGER.error(ape, "Locate agent.jar failure. Shutting down.");
            return;
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent initialized failure. Shutting down.");
            return;
        }
        …………
    }
```
查找文件夹是"plugins"和"activations"下的插件。插件通过skywalking-plugin.def定义了插件的切入点。我们通过插件定义获取插件定义类。

然后通过插件查找器将插件定义转化为插件增强类。

## 创建ByteBuddy实例
```
 // 使用ByteBuddy技术来进行字节码增强
 // IS_OPEN_DEBUGGING_CLASS 是否开启debug模式。 当为true时，会把增强过得字节码文件放到/debugging文件夹下，方便debug。
final ByteBuddy byteBuddy = new ByteBuddy().with(TypeValidation.of(Config.Agent.IS_OPEN_DEBUGGING_CLASS));
```

## ByteBuddy增强，忽略某个类
```
   AgentBuilder agentBuilder = new AgentBuilder.Default(byteBuddy).ignore(
                // 指定以这些类名为开头的 不属于要增强的范围
                nameStartsWith("net.bytebuddy.")
                        .or(nameStartsWith("org.slf4j."))
                        .or(nameStartsWith("org.groovy."))
                        .or(nameContains("javassist"))
                        .or(nameContains(".asm."))
                        .or(nameContains(".reflectasm."))
                        .or(nameStartsWith("sun.reflect"))
                        .or(allSkyWalkingAgentExcludeToolkit())
                        .or(ElementMatchers.isSynthetic()));
```

## 将必要的类注入到Bootstrap ClassLoader
```
        // 1)将必要的类注入到Bootstrap ClassLoader
        JDK9ModuleExporter.EdgeClasses edgeClasses = new JDK9ModuleExporter.EdgeClasses();
        try {
            agentBuilder = BootstrapInstrumentBoost.inject(pluginFinder, instrumentation, agentBuilder, edgeClasses);
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent inject bootstrap instrumentation failure. Shutting down.");
            return;
        }

```
定义边缘类集合：EdgeClasses
```
    JDK9ModuleExporter.EdgeClasses edgeClasses = new JDK9ModuleExporter.EdgeClasses();
```
其实EdgeClasses就是一个List，其中会包含ByteBuddy的核心类。
```
    public class ByteBuddyCoreClasses {
        private static final String SHADE_PACKAGE = "org.apache.skywalking.apm.dependencies.";

        public static final String[] CLASSES = {
            SHADE_PACKAGE + "net.bytebuddy.implementation.bind.annotation.RuntimeType",
            SHADE_PACKAGE + "net.bytebuddy.implementation.bind.annotation.This",
            SHADE_PACKAGE + "net.bytebuddy.implementation.bind.annotation.AllArguments",
            SHADE_PACKAGE + "net.bytebuddy.implementation.bind.annotation.AllArguments$Assignment",
            SHADE_PACKAGE + "net.bytebuddy.implementation.bind.annotation.SuperCall",
            SHADE_PACKAGE + "net.bytebuddy.implementation.bind.annotation.Origin",
            SHADE_PACKAGE + "net.bytebuddy.implementation.bind.annotation.Morph",
            };
    }
```
通过这些报名，我们发现其实它们是一些annotation(注解)。

## 解决JDK模块系统的跨模块类访问
```
        // 解决JDK模块系统的跨模块类访问
        try {
            agentBuilder = JDK9ModuleExporter.openReadEdge(instrumentation, agentBuilder, edgeClasses);
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent open read edge in JDK 9+ failure. Shutting down.");
            return;
        }
```

## 将修改后的字节码保存到磁盘/内存上
```
        // 将修改后的字节码保存到磁盘/内存上
        if (Config.Agent.IS_CACHE_ENHANCED_CLASS) {
            try {
                agentBuilder = agentBuilder.with(new CacheableTransformerDecorator(Config.Agent.CLASS_CACHE_MODE));
                LOGGER.info("SkyWalking agent class cache [{}] activated.", Config.Agent.CLASS_CACHE_MODE);
            } catch (Exception e) {
                LOGGER.error(e, "SkyWalking agent can't active class cache.");
            }
        }
```

## ByteBuddy增强
```
       agentBuilder.type(pluginFinder.buildMatch()) // 指定ByteBuddy要拦截的类
                    .transform(new Transformer(pluginFinder))
                    .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION) // redefine和retransform的区别在于是否保留修改前的内容
                    .with(new RedefinitionListener())
                    .with(new Listener())
                    .installOn(instrumentation);

```

## 启动服务
```
        try {
            // 启动服务
            ServiceManager.INSTANCE.boot();
        } catch (Exception e) {
            LOGGER.error(e, "Skywalking agent boot failure.");
        }
```

## 注册关闭钩子
```
    // 注册关闭钩子
        Runtime.getRuntime()
                .addShutdownHook(new Thread(ServiceManager.INSTANCE::shutdown, "skywalking service shutdown thread"));

```