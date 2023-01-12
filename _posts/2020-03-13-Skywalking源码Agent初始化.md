---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
#### 源码分析--Agent 初始化

**SkyWalking Agent 启动初始化的过程**。

SkyWalking Agent 基于 **JavaAgent** 机制，实现应用**透明**接入 SkyWalking 。关于 JavaAgent 机制，笔者推荐如下两篇文章 ：

- [《Instrumentation 新功能》](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/index.html)
- [《JVM源码分析之javaagent原理完全解读》](http://www.infoq.com/cn/articles/javaagent-illustrated)

> 友情提示 ：建议自己手撸一个简单的 JavaAgent ，更容易理解 SkyWalking Agent 。

`org.apache.skywalking.apm.agent.SkyWalkingAgent` ，在 `apm-sniffer/apm-agent` Maven 模块项目里，SkyWalking Agent **启动入口**。为什么说它是启动入口呢？在 `apm-sniffer/apm-agent` 的 [`pom.xml`](https://github.com/OpenSkywalking/skywalking/blob/23133f7d97d17b471f69e7214a01885ebcd2e882/apm-sniffer/apm-agent/pom.xml#L53) 文件的【我们可以看到 SkyWalkingAgent 被配置成 JavaAgent 的 **PremainClass** 。

```
 <premain.class>org.apache.skywalking.apm.agent.SkyWalkingAgent</premain.class>
```

- 调用 `SnifferConfigInitializer#initialize()` 方法，初始化 Agent 配置。
- 调用 `PluginBootstrap#loadPlugins()` 方法，加载 Agent 插件们。而后，创建 PluginFinder 。
- 调用 `ServiceManager#boot()` 方法，初始化 Agent 服务管理。在这过程中，Agent 服务们会被初始化。
- 基于 [byte-buddy](https://github.com/raphw/byte-buddy) ，初始化 Instrumentation 的 [`java.lang.instrument.ClassFileTransformer`](https://docs.oracle.com/javase/7/docs/api/java/lang/instrument/ClassFileTransformer.html) 。

#### **源码解读-初始化 Agent 配置**

SkyWalkingAgent#premain方法：

SnifferConfigInitializer.initializeCoreConfig(agentArgs);来进行配置的初始化

```
	//主要入口。使用byte-buddy 转换来增强所有在插件中定义的类。
    public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException {
        final PluginFinder pluginFinder;
        try {
            //初始化配置信息：javaagent:/……/agent.jar=k1=v1,k2=v2,……
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

        ……
    }
```

initializeCoreConfig方法

```
/**
     * 如果设置了指定的代理配置路径，则代理将尝试查找指定的代理配置。如果未设置指定的代理配置路径，则代理将尝试找到“ agent.config”，该目录应位于代理软件包的/ config目录中。
     * <p>
     * 另外，尝试通过system.properties覆盖配置。该位置的所有键都应以skywalking开头。例如在`skywalking.agent.service_name = yourAppName`中覆盖配置文件中的`agent.service_name`。
     就是javaagent:/……/agent.jar=k1=v1,k2=v2,……通过这种方式来覆盖配置的信息
     * <p>
     * 最后，“ agent.service_name”和“ collector.servers”不能为空。
     */ 

public static void initializeCoreConfig(String agentOptions) {
        AGENT_SETTINGS = new Properties();
     	//loadConfig来进行加载配置项
        try (final InputStreamReader configFileStream = loadConfig()) {
            AGENT_SETTINGS.load(configFileStream);
            for (String key : AGENT_SETTINGS.stringPropertyNames()) {
                String value = (String) AGENT_SETTINGS.get(key);
                //这里来处理${SW_AGENT_NAME:boot_demo} =》boot——demo
                AGENT_SETTINGS.put(key, PropertyPlaceholderHelper.INSTANCE.replacePlaceholders(value, AGENT_SETTINGS));
            }

        } catch (Exception e) {
            LOGGER.error(e, "Failed to read the config file, skywalking is going to run in default config.");
        }

       ……
    }
```

在initializeCoreConfig方法中，调用了*loadConfig*()来进行初始化配置内容

```
 private static ILog LOGGER = LogManager.getLogger(SnifferConfigInitializer.class);
    private static final String SPECIFIED_CONFIG_PATH = "skywalking_config";
    private static final String DEFAULT_CONFIG_FILE_NAME = "/config/agent.config";
    private static final String ENV_KEY_PREFIX = "skywalking.";
    private static Properties AGENT_SETTINGS;
    private static boolean IS_INIT_COMPLETED = false;	

	/**
     * 加载指定的配置文件或默认配置文件
     *
     * @return the config file {@link InputStream}, or null if not needEnhance.
     */
    private static InputStreamReader loadConfig() throws AgentPackageNotFoundException, ConfigNotFoundException {
        //获取skywalking_config配置的环境变量：
        String specifiedConfigPath = System.getProperty(SPECIFIED_CONFIG_PATH);
        //如果环境变量没有配置，就读取默认的配置信息：文件目录+/config/agent.config
        File configFile = StringUtil.isEmpty(specifiedConfigPath) ? new File(
            AgentPackagePath.getPath(), DEFAULT_CONFIG_FILE_NAME) : new File(specifiedConfigPath);

        if (configFile.exists() && configFile.isFile()) {
            try {
                //log日志中会显示这段信息
                LOGGER.info("Config file found in {}.", configFile);

                return new InputStreamReader(new FileInputStream(configFile), StandardCharsets.UTF_8);
            } catch (FileNotFoundException e) {
                throw new ConfigNotFoundException("Failed to load agent.config", e);
            }
        }
        throw new ConfigNotFoundException("Failed to load agent.config.");
    }
```

*最后使用initializeConfig*(Config.class);方法将配置文件放入到Config类中。

最终调用：ConfigInitializer.*initialize*(*AGENT_SETTINGS*, configClass);对Config的属性进行复制。