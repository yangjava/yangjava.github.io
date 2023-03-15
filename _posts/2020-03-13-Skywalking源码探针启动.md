---
layout: post
categories: Skywalking
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

## 初始化配置
SkyWalking Java Agent 在 premain 方法中首先做的就是通过`SnifferConfigInitializer.initializeCoreConfig(agentArgs)`; 初始化核心配置。
代码如下
```java
    /**
     * If the specified agent config path is set, the agent will try to locate the specified agent config. If the
     * specified agent config path is not set , the agent will try to locate `agent.config`, which should be in the
     * /config directory of agent package.
     * <p>
     * Also try to override the config by system.properties. All the keys in this place should start with {@link
     * #ENV_KEY_PREFIX}. e.g. in env `skywalking.agent.service_name=yourAppName` to override `agent.service_name` in
     * config file.
     * <p>
     * At the end, `agent.service_name` and `collector.servers` must not be blank.
     */
    // 如果指定agent路径，加载指定配置信息
    // 如果没有指定agent路径，默认加载 /config/agent.config
    // 通过system.properties覆盖配置。该位置的所有键都应以skywalking开头。
    // `agent.service_name`和`collector.servers`不能为空
    public static void initializeCoreConfig(String agentOptions) {
        AGENT_SETTINGS = new Properties();
        // 加载配置文件信息
        try (final InputStreamReader configFileStream = loadConfig()) {
            AGENT_SETTINGS.load(configFileStream);
            for (String key : AGENT_SETTINGS.stringPropertyNames()) {
                String value = (String) AGENT_SETTINGS.get(key);
                // 占位符处理 ${SW_AGENT_NAME:boot_demo}，就是SW_AGENT_NAME默认为boot_demo
                AGENT_SETTINGS.put(key, PropertyPlaceholderHelper.INSTANCE.replacePlaceholders(value, AGENT_SETTINGS));
            }

        } catch (Exception e) {
            LOGGER.error(e, "Failed to read the config file, skywalking is going to run in default config.");
        }

        try {
            // 系统配置项覆盖
            overrideConfigBySystemProp();
        } catch (Exception e) {
            LOGGER.error(e, "Failed to read the system properties.");
        }
        // 使用agentOptions覆盖
        agentOptions = StringUtil.trim(agentOptions, ',');
        if (!StringUtil.isEmpty(agentOptions)) {
            try {
                agentOptions = agentOptions.trim();
                LOGGER.info("Agent options is {}.", agentOptions);

                overrideConfigByAgentOptions(agentOptions);
            } catch (Exception e) {
                LOGGER.error(e, "Failed to parse the agent options, val is {}.", agentOptions);
            }
        }
        // 配置文件放到Config类中
        initializeConfig(Config.class);
        // reconfigure logger after config initialization
        // 初始化Log处理
        configureLogger();
        LOGGER = LogManager.getLogger(SnifferConfigInitializer.class);
        // 如果agent.service_name为空，报错
        if (StringUtil.isEmpty(Config.Agent.SERVICE_NAME)) {
            throw new ExceptionInInitializerError("`agent.service_name` is missing.");
        }
        // collector.backend_service不能为空
        if (StringUtil.isEmpty(Config.Collector.BACKEND_SERVICE)) {
            throw new ExceptionInInitializerError("`collector.backend_service` is missing.");
        }
        if (Config.Plugin.PEER_MAX_LENGTH <= 3) {
            LOGGER.warn(
                "PEER_MAX_LENGTH configuration:{} error, the default value of 200 will be used.",
                Config.Plugin.PEER_MAX_LENGTH
            );
            Config.Plugin.PEER_MAX_LENGTH = 200;
        }

        IS_INIT_COMPLETED = true;
    }
```
SnifferConfigInitializer 类使用多种方式初始化配置，内部实现有以下几个重要步骤：
- `loadConfig()`加载配置文件
- `replacePlaceholders()` 解析占位符 placeholder
- `overrideConfigBySystemProp()`使用系统属性覆盖配置
- `overrideConfigByAgentOptions()` 解析agentArgs参数覆盖配置
- `initializeConfig()`将以上读取到的配置信息映射到 Config 类的静态属性
- `configureLogger()` 根据配置的 Config.Logging.RESOLVER 重配置 Log
- 验证非空参数 agent.service_name 和 collector.servers

### 加载配置文件信息
loadConfig() 加载配置文件,Spring采用的配置文件默认是agent.config
从指定的配置文件路径读取配置文件内容，通过 -Dskywalking_config=/xxx/yyy 可以指定配置文件位置；
如果没有指定配置文件路径，则从默认配置文件 config/agent.config 读取；
将配置文件内容加载到 Properties；
```java

    private static final String SPECIFIED_CONFIG_PATH = "skywalking_config";
    private static final String DEFAULT_CONFIG_FILE_NAME = "/config/agent.config";
    /**
     * Load the specified config file or default config file
     *
     * @return the config file {@link InputStream}, or null if not needEnhance.
     */
    // 加载配置文件
    private static InputStreamReader loadConfig() throws AgentPackageNotFoundException, ConfigNotFoundException {
        // 加载指定的配置文件 skywalking_config
        String specifiedConfigPath = System.getProperty(SPECIFIED_CONFIG_PATH);
        // 指定配置文件为空，取默认文件 /config/agent.config
        File configFile = StringUtil.isEmpty(specifiedConfigPath) ? new File(
            AgentPackagePath.getPath(), DEFAULT_CONFIG_FILE_NAME) : new File(specifiedConfigPath);
        // 加载配置文件信息
        if (configFile.exists() && configFile.isFile()) {
            try {
                LOGGER.info("Config file found in {}.", configFile);

                return new InputStreamReader(new FileInputStream(configFile), StandardCharsets.UTF_8);
            } catch (FileNotFoundException e) {
                throw new ConfigNotFoundException("Failed to load agent.config", e);
            }
        }
        throw new ConfigNotFoundException("Failed to load agent.config.");
    }
```

### 解析占位符 placeholder
从配置文件中读取到的配置值都是以 placeholder 形式(比如 agent.service_name=${SW_AGENT_NAME:Your_ApplicationName})存在的，这里需要将占位符解析为实际值。
```java

/**
 * Replaces all placeholders of format {@code ${name}} with the corresponding property from the supplied {@link
 * Properties}.
 *
 * @param value      the value containing the placeholders to be replaced
 * @param properties the {@code Properties} to use for replacement
 * @return the supplied value with placeholders replaced inline
 */
public String replacePlaceholders(String value, final Properties properties) {
    return replacePlaceholders(value, new PlaceholderResolver() {
        @Override
        public String resolvePlaceholder(String placeholderName) {
            return getConfigValue(placeholderName, properties);
        }
    });
}
 
// 优先级 System.Properties(-D) > System environment variables > Config file
private String getConfigValue(String key, final Properties properties) {
    // 从Java虚拟机系统属性中获取(-D)
    String value = System.getProperty(key);
    if (value == null) {
        // 从操作系统环境变量获取, 比如 JAVA_HOME、Path 等环境变量
        value = System.getenv(key);
    }
    if (value == null) {
        // 从配置文件中获取
        value = properties.getProperty(key);
    }
    return value;
}
```

### 使用系统属性覆盖配置
overrideConfigBySystemProp() 读取 System.getProperties() 中以 skywalking. 开头的系统属性覆盖配置
```java
    /**
     * Override the config by system properties. The property key must start with `skywalking`, the result should be as
     * same as in `agent.config`
     * <p>
     * such as: Property key of `agent.service_name` should be `skywalking.agent.service_name`
     */
    // 加载系统配置，如果Key是skywalking开头，则覆盖agent.config中的配置
    private static void overrideConfigBySystemProp() throws IllegalAccessException {
        Properties systemProperties = System.getProperties();
        for (final Map.Entry<Object, Object> prop : systemProperties.entrySet()) {
            String key = prop.getKey().toString();
            if (key.startsWith(ENV_KEY_PREFIX)) {
                String realKey = key.substring(ENV_KEY_PREFIX.length());
                AGENT_SETTINGS.put(realKey, prop.getValue());
            }
        }
    }
```

### 解析 agentArgs 参数配置覆盖配置
agentArgs 就是 premain 方法的第一个参数，以 -javaagent:/path/to/skywalking-agent.jar=k1=v1,k2=v2的形式传值。
```java
  private static void overrideConfigByAgentOptions(String agentOptions) throws IllegalArgumentException {
        for (List<String> terms : parseAgentOptions(agentOptions)) {
            if (terms.size() != 2) {
                throw new IllegalArgumentException("[" + terms + "] is not a key-value pair.");
            }
            AGENT_SETTINGS.put(terms.get(0), terms.get(1));
        }
    }
```


### 配置信息映射到 Config 类的静态属性
initializeConfig() 将以上读取到的配置信息映射到 Config 类的静态属性
```java
    /**
     * Initialize field values of any given config class.
     *
     * @param configClass to host the settings for code access.
     */
    // 初始化配置文件成Config对象信息
    public static void initializeConfig(Class configClass) {
        if (AGENT_SETTINGS == null) {
            LOGGER.error("Plugin configs have to be initialized after core config initialization.");
            return;
        }
        try {
            ConfigInitializer.initialize(AGENT_SETTINGS, configClass);
        } catch (IllegalAccessException e) {
            LOGGER.error(e,
                         "Failed to set the agent settings {}"
                             + " to Config={} ",
                         AGENT_SETTINGS, configClass
            );
        }
    }
```
在我们的日常开发中一般是直接从 Properties 读取需要的配置项，SkyWalking Java Agent 并没有这么做，而是定义一个配置类 Config，将配置项映射到 Config 类的静态属性中，其他地方需要配置项的时候，直接从类的静态属性获取就可以了，非常方便使用。
ConfigInitializer 就是负责将 Properties 中的 key/value 键值对映射到类（比如 Config 类）的静态属性，其中 key 对应类的静态属性，value 赋值给静态属性的值。
```java

/**
 * This is the core config in sniffer agent.
 */
public class Config {
 
    public static class Agent {
        /**
         * Namespace isolates headers in cross process propagation. The HEADER name will be `HeaderName:Namespace`.
         */
        public static String NAMESPACE = "";
 
        /**
         * Service name is showed in skywalking-ui. Suggestion: set a unique name for each service, service instance
         * nodes share the same code
         */
        @Length(50)
        public static String SERVICE_NAME = "";
     
        // 省略部分代码....
    }
    
    public static class Collector {
        /**
         * Collector skywalking trace receiver service addresses.
         */
        public static String BACKEND_SERVICE = "";
        
        // 省略部分代码....
    }
    
    // 省略部分代码....
 
    public static class Logging {
        /**
         * Log file name.
         */
        public static String FILE_NAME = "skywalking-api.log";
 
        /**
         * Log files directory. Default is blank string, means, use "{theSkywalkingAgentJarDir}/logs  " to output logs.
         * {theSkywalkingAgentJarDir} is the directory where the skywalking agent jar file is located.
         * <p>
         * Ref to {@link WriterFactory#getLogWriter()}
         */
        public static String DIR = "";
    }
 
    // 省略部分代码....
 
}

```
比如通过 agent.config 配置文件配置服务名称
```
agent.service_name=${SW_AGENT_NAME:Your_ApplicationName}
```
agent 对应 Config 类的静态内部类 Agent ；
service_name 对应静态内部类 Agent 的静态属性 SERVICE_NAME。
SkyWalking Java Agent 在这里面使用了下划线而不是驼峰来命名配置项，将类的静态属性名称转换成下划线配置名称非常方便，直接转成小写就可以通过 Properties 获取对应的值了。
```java
/**
 * Init a class's static fields by a {@link Properties}, including static fields and static inner classes.
 * <p>
 */
public class ConfigInitializer {
    public static void initialize(Properties properties, Class<?> rootConfigType) throws IllegalAccessException {
        initNextLevel(properties, rootConfigType, new ConfigDesc());
    }
 
    private static void initNextLevel(Properties properties, Class<?> recentConfigType,
                                      ConfigDesc parentDesc) throws IllegalArgumentException, IllegalAccessException {
        for (Field field : recentConfigType.getFields()) {
            if (Modifier.isPublic(field.getModifiers()) && Modifier.isStatic(field.getModifiers())) {
                String configKey = (parentDesc + "." + field.getName()).toLowerCase();
                Class<?> type = field.getType();
 
                if (type.equals(Map.class)) {
                    /*
                     * Map config format is, config_key[map_key]=map_value
                     * Such as plugin.opgroup.resttemplate.rule[abc]=/url/path
                     */
                    // Deduct two generic types of the map
                    ParameterizedType genericType = (ParameterizedType) field.getGenericType();
                    Type[] argumentTypes = genericType.getActualTypeArguments();
 
                    Type keyType = null;
                    Type valueType = null;
                    if (argumentTypes != null && argumentTypes.length == 2) {
                        // Get key type and value type of the map
                        keyType = argumentTypes[0];
                        valueType = argumentTypes[1];
                    }
                    Map map = (Map) field.get(null);
                    // Set the map from config key and properties
                    setForMapType(configKey, map, properties, keyType, valueType);
                } else {
                    /*
                     * Others typical field type
                     */
                    String value = properties.getProperty(configKey);
                    // Convert the value into real type
                    final Length lengthDefine = field.getAnnotation(Length.class);
                    if (lengthDefine != null) {
                        if (value != null && value.length() > lengthDefine.value()) {
                            value = value.substring(0, lengthDefine.value());
                        }
                    }
                    Object convertedValue = convertToTypicalType(type, value);
                    if (convertedValue != null) {
                        // 通过反射给静态属性设置值
                        field.set(null, convertedValue);
                    }
                }
            }
        }
        // recentConfigType.getClasses() 获取 public 的 classes 和 interfaces
        for (Class<?> innerConfiguration : recentConfigType.getClasses()) {
            // parentDesc 将类（接口）名入栈
            parentDesc.append(innerConfiguration.getSimpleName());
            // 递归调用
            initNextLevel(properties, innerConfiguration, parentDesc);
            // parentDesc 将类（接口）名出栈
            parentDesc.removeLastDesc();
        }
    }
 
    // 省略部分代码....
}
 
class ConfigDesc {
    private LinkedList<String> descs = new LinkedList<>();
 
    void append(String currentDesc) {
        if (StringUtil.isNotEmpty(currentDesc)) {
            descs.addLast(currentDesc);
        }
    }
 
    void removeLastDesc() {
        descs.removeLast();
    }
 
    @Override
    public String toString() {
        return String.join(".", descs);
    }
}

```
ConfigInitializer.initNextLevel 方法涉及到的技术点有反射、递归调用、栈等。

### configureLogger() 根据配置的 Config.Logging.RESOLVER 重配置 Log

```java

static void configureLogger() {
    switch (Config.Logging.RESOLVER) {
        case JSON:
            LogManager.setLogResolver(new JsonLogResolver());
            break;
        case PATTERN:
        default:
            LogManager.setLogResolver(new PatternLogResolver());
    }
}

```

### 验证非空参数
```java
if (StringUtil.isEmpty(Config.Agent.SERVICE_NAME)) {
    throw new ExceptionInInitializerError("`agent.service_name` is missing.");
}
if (StringUtil.isEmpty(Config.Collector.BACKEND_SERVICE)) {
    throw new Excep
//标记完成配置初始化
IS_INIT_COMPLETED = true;
```

## 插件加载机制










