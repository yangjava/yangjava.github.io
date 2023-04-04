---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
# Skywalking服务端原理解析

## Skywalking服务端OAP功能模块
我们需要对功能模块有个总体认识，方便理解我们后续链路信息处理模块。
oap-server模块是Skywalking的OAP的实现模块，有下列子模块：
- analyzer：是ageng上传的对链路信息进行加工处理。
- server-core：OAP服务核心模块
- oal-rt：oal类生成引擎
- server-alarm-plugin：报警插件
- server-cluster-plugin：集群信息插件，支持etcd、kubernetes、nacos、Zookeeper等等服务的插件
- oal-grammar：oal语言的语法
- server-configuration：这是负责管理OAP配置信息，包括Apollo的配置、nacos、etcd、Zookeeper等等
- server-library：公共模块部分
- server-query-plugin：这一块是对查询的处理，处理的是SKywalking前台发过来的查询请求
- server-receiver-plugin：接收请求，包括Metrics、istio、jvm、mesh、trace等等插件，用来把数据写入Skywalking中
- server-starter-es7：这是OAP的starter启动模块
- server-storage-plugin：存储插件，也就是支持什么存储，有ElasticSearch、zipkin、jdbc等等，这些存储形式的插件也很丰富。

## OAP服务启动
OAP 使用 ModuleManager（组件管理器）管理多个 Module（组件），一个 Module 可以对应多个 ModuleProvider（组件服务提供者），ModuleProvider 是 Module 底层真正的实现。在 OAP 服务启动时，一个 Module 只能选择使用一个 ModuleProvider 对外提供服务。一个 ModuleProvider 可能支撑了一个非常复杂的大功能，在一个 ModuleProvider 中，可以包含多个 Service ，一个 Service 实现了一个 ModuleProvider 中的一部分功能，通过将多个 Service 进行组装集成，可以得到 ModuleProvider 的完整功能。

### OAP服务启动入口
先打开模块：oap-server->server-starter-es7。如果使用elasticsearch7作为存储，则需要从这里启动。
Skywalking-oap的核心启动类OAPServerBootstrap.位置：org/apache/skywalking/oap/server/starter/OAPServerBootstrap.java:
```java
/**
 * Starter core. Load the core configuration file, and initialize the startup sequence through {@link ModuleManager}.
 */
// 由server-starter和server-starter-es7调用server-bootstrap
// server-starter和server-starter-es7的区别在于maven中引入的存储模块Module不同
// server-starter      storage-elasticsearch-plugin
// server-starter-es7  storage-elasticsearch7-plugin
@Slf4j
public class OAPServerBootstrap {
    public static void start() {
        // // 初始化mode为init或者no-init,表示是否初始化例如:底层存储组件等
        String mode = System.getProperty("mode");
        RunningMode.setMode(mode);

        // 初始化ApplicationConfigurationd的加载器
        ApplicationConfigLoader configLoader = new ApplicationConfigLoader();
        // 初始化Module的加载管理器
        ModuleManager manager = new ModuleManager();
        try {
            // 加载yml生成ApplicationConfiguration配置
            ApplicationConfiguration applicationConfiguration = configLoader.load();
            // 初始化模块 通过spi获取所有Module实现,基于yml配置加载spi中存在的相关实现
            manager.init(applicationConfiguration);

            // 对 telemetry 模块进行特殊处理, 添加 uptime 的 Gauge, 用于标识服务开始运行
            manager.find(TelemetryModule.NAME)
                   .provider()
                   .getService(MetricsCreator.class)
                   .createGauge("uptime", "oap server start up time", MetricsTag.EMPTY_KEY, MetricsTag.EMPTY_VALUE)
                   // Set uptime to second
                   .setValue(System.currentTimeMillis() / 1000d);
            // 如果是 Init 模式, 现在就直接退出, 否则就继续运行
            if (RunningMode.isInitMode()) {
                log.info("OAP starts up in init mode successfully, exit now...");
                System.exit(0);
            }
        } catch (Throwable t) {
            log.error(t.getMessage(), t);
            System.exit(1);
        }
    }
}
```
- 初始化mode
如果是初次启动skyalking oap，可以设置环境变量mode=init，这样启动后系统只会进行系统初始化动作，例如kafka建立topic；elasticsearch建立index，数据库建表。所有的模块都会执行初始化动作。初始化完成后，系统运行结束，不会进行信息采集。如果不设置mode环境变量，则是正常启动oap。
```java

/**
 * The running mode of the OAP server.
 */
// 运行Mode init和no-init
public class RunningMode {
    private static String MODE = "";

    private RunningMode() {
    }

    public static void setMode(String mode) {
        if (Strings.isNullOrEmpty(mode)) {
            return;
        }
        RunningMode.MODE = mode.toLowerCase();
    }

    /**
     * Init mode, do all initialization things, and process should exit.
     *
     * @return true if in this status
     */
    public static boolean isInitMode() {
        return "init".equals(MODE);
    }

    /**
     * No-init mode, the oap just starts up, but wouldn't do storage init.
     *
     * @return true if in this status.
     */
    public static boolean isNoInitMode() {
        return "no-init".equals(MODE);
    }
}
```
`Mode`有`init`和`no-init`。如果不设置`mode`环境变量，则是正常启动`oap`。

### 加载配置
这里是读取初始目录下面，config目录中的application.yml文件，根据配置加载各个模块的各个候选实现。
```java
/**
 * Initialize collector settings with following sources. Use application.yml as primary setting, and fix missing setting
 * by default settings in application-default.yml.
 * <p>
 * At last, override setting by system.properties and system.envs if the key matches moduleName.provideName.settingKey.
 */
// application.yml加载类, 三层结构:模块定义名.模块提供名.属性key
// ApplicationConfiguration 的辅助类，将 application.yml 配置文件加载到内存， 设置 selector 对应的 Provider 的配置信息
@Slf4j
public class ApplicationConfigLoader implements ConfigLoader<ApplicationConfiguration> {
    // 当不配置模块提供者时，使用"-"
    private static final String DISABLE_SELECTOR = "-";
    // 该字段选择模块提供者
    private static final String SELECTOR = "selector";

    private final Yaml yaml = new Yaml();

    @Override
    public ApplicationConfiguration load() throws ConfigFileNotFoundException {
        // 初始化ApplicationConfiguration对象
        ApplicationConfiguration configuration = new ApplicationConfiguration();
        this.loadConfig(configuration);
        // 读取系统属性对值进行覆盖处理, 使用的是
        this.overrideConfigBySystemEnv(configuration);
        return configuration;
    }

    @SuppressWarnings("unchecked")
    private void loadConfig(ApplicationConfiguration configuration) throws ConfigFileNotFoundException {
        try {
            // 读取application.yml
            Reader applicationReader = ResourceUtils.read("application.yml");
            // 将 application.yml 转换为 ModuleName, Map<String, Object> 的结构
            Map<String, Map<String, Object>> moduleConfig = yaml.loadAs(applicationReader, Map.class);

            if (CollectionUtils.isNotEmpty(moduleConfig)) {
                // 读取每个ModuleName下的 Map, 先读取到selector属性, 如果不存在就跳过此端配置
                selectConfig(moduleConfig);

                moduleConfig.forEach((moduleName, providerConfig) -> {
                    if (providerConfig.size() > 0) {
                        log.info("Get a module define from application.yml, module name: {}", moduleName);
                        ApplicationConfiguration.ModuleConfiguration moduleConfiguration = configuration.addModule(
                            moduleName);
                        providerConfig.forEach((providerName, config) -> {
                            log.info(
                                "Get a provider define belong to {} module, provider name: {}", moduleName,
                                providerName
                            );
                            final Map<String, ?> propertiesConfig = (Map<String, ?>) config;
                            final Properties properties = new Properties();
                            if (propertiesConfig != null) {
                                propertiesConfig.forEach((propertyName, propertyValue) -> {
                                    if (propertyValue instanceof Map) {
                                        Properties subProperties = new Properties();
                                        ((Map) propertyValue).forEach((key, value) -> {
                                            subProperties.put(key, value);
                                            replacePropertyAndLog(key, value, subProperties, providerName);
                                        });
                                        properties.put(propertyName, subProperties);
                                    } else {
                                        properties.put(propertyName, propertyValue);
                                        replacePropertyAndLog(propertyName, propertyValue, properties, providerName);
                                    }
                                });
                            }
                            moduleConfiguration.addProviderConfiguration(providerName, properties);
                        });
                    } else {
                        log.warn(
                            "Get a module define from application.yml, but no provider define, use default, module name: {}",
                            moduleName
                        );
                    }
                });
            }
        } catch (FileNotFoundException e) {
            throw new ConfigFileNotFoundException(e.getMessage(), e);
        }
    }
    // ... 
```
### 初始化ApplicationConfiguration对象
ApplicationConfiguration对象使用Map存储配置，使用modules存储组件。然后使用providers提供多个组件提供方。
application.yml加载类, 三层结构:模块定义名.模块提供名.属性key
```java
// OAP应用配置类
public class ApplicationConfiguration {
    // 模块定义配置map
    private HashMap<String, ModuleConfiguration> modules = new HashMap<>();
    // 模块配置名列表
    public String[] moduleList() {
        return modules.keySet().toArray(new String[0]);
    }
    // 添加模块定义配置
    public ModuleConfiguration addModule(String moduleName) {
        ModuleConfiguration newModule = new ModuleConfiguration();
        modules.put(moduleName, newModule);
        return newModule;
    }
    // 判断指定模块名是否存在模块定义配置map中
    public boolean has(String moduleName) {
        return modules.containsKey(moduleName);
    }
    // 获取模块定义配置
    public ModuleConfiguration getModuleConfiguration(String name) {
        return modules.get(name);
    }

    /**
     * The configurations about a certain module.
     */
    // 模块定义配置类
    public static class ModuleConfiguration {

        // 模块提供对象map
        private HashMap<String, ProviderConfiguration> providers = new HashMap<>();

        private ModuleConfiguration() {
        }
        // 获取模块提供配置
        public Properties getProviderConfiguration(String name) {
            return providers.get(name).getProperties();
        }
        // 是否存在模块提供配置
        public boolean has(String name) {
            return providers.containsKey(name);
        }
        // 添加模块提供配置
        public ModuleConfiguration addProviderConfiguration(String name, Properties properties) {
            ProviderConfiguration newProvider = new ProviderConfiguration(properties);
            providers.put(name, newProvider);
            return this;
        }
    }

    /**
     * The configuration about a certain provider of a module.
     */
    // 模块提供配置类
    public static class ProviderConfiguration {
        // 模块提供属性
        private Properties properties;

        ProviderConfiguration(Properties properties) {
            this.properties = properties;
        }

        private Properties getProperties() {
            return properties;
        }
    }
}
```
最后将读取到的配置转化为ApplicationConfiguration对象。
例如以core模块为例,ApplicationConfiguration中的modules这个map，key是core，
providers的key是default
从 application.yml 配置文件，可以看出。
模块是以三层结构来定义的：
第一层：模块定义名
第二层：模块提供名/ selector
第三层：模块提供配置信息/ selector 选择的模块提供配置
```yaml
core:
  selector: ${SW_CORE:default}
  default:
    # Mixed: Receive agent data, Level 1 aggregate, Level 2 aggregate
    # Receiver: Receive agent data, Level 1 aggregate
    # Aggregator: Level 2 aggregate
    role: ${SW_CORE_ROLE:Mixed} # Mixed/Receiver/Aggregator
    restHost: ${SW_CORE_REST_HOST:0.0.0.0}
    restPort: ${SW_CORE_REST_PORT:12800}
    restContextPath: ${SW_CORE_REST_CONTEXT_PATH:/}
    restMinThreads: ${SW_CORE_REST_JETTY_MIN_THREADS:1}

```

### 初始化模块
初始化模块 通过spi获取所有Module实现,基于yml配置加载spi中存在的相关实现
```java
// 模块管理类，管理模块的生命周期
// Module 是 Skywalking 在 OAP 提供的一种管理功能特性的机制。通过 Module 机制，可以方便的定义模块，并且可以提供多种实现，在配置文件中任意选择实现。
public class ModuleManager implements ModuleDefineHolder {
    // 所有模块是否已经通过准备阶段
    private boolean isInPrepareStage = true;
    // 所有被加载的模块定义对象map
    private final Map<String, ModuleDefine> loadedModules = new HashMap<>();

    /**
     * Init the given modules
     */
    // 初始化所有配置的模块
    public void init(
        ApplicationConfiguration applicationConfiguration) throws ModuleNotFoundException, ProviderNotFoundException, ServiceNotProvidedException, CycleDependencyException, ModuleConfigException, ModuleStartException {
        // 获取配置类中的模块名
        String[] moduleNames = applicationConfiguration.moduleList();
        // SPI加载所有模块定义对象
        ServiceLoader<ModuleDefine> moduleServiceLoader = ServiceLoader.load(ModuleDefine.class);
        // SPI加载所有模块提供对象
        ServiceLoader<ModuleProvider> moduleProviderLoader = ServiceLoader.load(ModuleProvider.class);
        // 所有配置类中定义的模块，进行准备阶段
        HashSet<String> moduleSet = new HashSet<>(Arrays.asList(moduleNames));
        for (ModuleDefine module : moduleServiceLoader) {
            // 判断 ModuleDefine.name 是否在 ApplicationConfiguration 配置中存在, 不存在跳过
            if (moduleSet.contains(module.name())) {
                // 存在时, 分别调用每个 ModuleDefine#prepare 进行初始化, 此处主要工作为配置初始化, 完成后放入 loadedModules 中
                module.prepare(this, applicationConfiguration.getModuleConfiguration(module.name()), moduleProviderLoader);
                loadedModules.put(module.name(), module);
                moduleSet.remove(module.name());
            }
        }
        // Finish prepare stage
        // 准备阶段结束
        isInPrepareStage = false;

        if (moduleSet.size() > 0) {
            throw new ModuleNotFoundException(moduleSet.toString() + " missing.");
        }
        // 使用loadedModules 对 BootstrapFlow 进行初始化, BootstrapFlow 会对模块的依赖关系进行排序
        BootstrapFlow bootstrapFlow = new BootstrapFlow(loadedModules);
        // 调用每个模块的启动阶段
        bootstrapFlow.start(this);
        // 所有模块进入完成后通知阶段 所有模块都已经加载完成
        bootstrapFlow.notifyAfterCompleted();
    }

    // 判断是否有该模块
    @Override
    public boolean has(String moduleName) {
        return loadedModules.get(moduleName) != null;
    }

    // 通过模块名获取模块定义对象
    @Override
    public ModuleProviderHolder find(String moduleName) throws ModuleNotFoundRuntimeException {
        assertPreparedStage();
        ModuleDefine module = loadedModules.get(moduleName);
        if (module != null)
            return module;
        throw new ModuleNotFoundRuntimeException(moduleName + " missing.");
    }
    // 断言是否还在准备阶段，如果还在准备阶段，则抛出异常
    private void assertPreparedStage() {
        if (isInPrepareStage) {
            throw new AssertionError("Still in preparing stage.");
        }
    }
}
```
Skywalking 中模块管理相关功能都在 org.apache.skywalking.oap.server.library.module 包下。
模块配置： ApplicationConfiguration 、 ModuleConfiguration 、 ProviderConfiguration
刚好对应 application.yml 三层结构：模块->模块实现->某个模块实现的配置。
模块定义类： ModuleDefine
模块提供类： ModuleProvider
服务： Service
管理类： ModuleManager
一些辅助类
ModuleDefineHolder ：模块管理类需要实现的接口，提供查找模块相关功能
ModuleProviderHolder ：模块定义类需要实现的接口，提供获取模块的服务类功能
ModuleServiceHolder ：模块提供类需要实现的接口，提供注册服务实现、获取服务对象的功能
ModuleConfig ：模块配置类，模块定义类会将ProviderConfiguration 映射为 ModuleConfig
ApplicationConfigLoader：ApplicationConfiguration 的辅助类，将 application.yml 配置文件加载到内存， 设置 selector 对应的 Provider 的配置信息
- 模块定义类
```java
/**
 * A module definition.
 */
// 模块定义
public abstract class ModuleDefine implements ModuleProviderHolder {

    private static final Logger LOGGER = LoggerFactory.getLogger(ModuleDefine.class);

    private ModuleProvider loadedProvider = null;
    // 模块名
    private final String name;

    public ModuleDefine(String name) {
        this.name = name;
    }

    /**
     * @return the module name
     *
     */
    public final String name() {
        return name;
    }

    /**
     * @return the {@link Service} provided by this module.
     */
    // 实现类可以定义模块提供的服务类
    public abstract Class[] services();

    /**
     * Run the prepare stage for the module, including finding all potential providers, and asking them to prepare.
     *
     * @param moduleManager of this module
     * @param configuration of this module
     * @throws ProviderNotFoundException when even don't find a single one providers.
     */
    // 准备阶段，找到configuration配置类对应的ModuleProvider对象，进行初始化操作
    void prepare(ModuleManager moduleManager, ApplicationConfiguration.ModuleConfiguration configuration,
        ServiceLoader<ModuleProvider> moduleProviderLoader) throws ProviderNotFoundException, ServiceNotProvidedException, ModuleConfigException, ModuleStartException {
        // 读取所有 ModuleProvider 实例, 找到和当前 ModuleDefine 匹配的 Provider
        for (ModuleProvider provider : moduleProviderLoader) {
            if (!configuration.has(provider.name())) {
                continue;
            }
            // 在 ModuleProvider 注入 ModuleDefine实例和 moduleManager 实例, 并将 provider 设置到当前 ModuleDefine 中
            if (provider.module().equals(getClass())) {
                if (loadedProvider == null) {
                    loadedProvider = provider;
                    loadedProvider.setManager(moduleManager);
                    loadedProvider.setModuleDefine(this);
                } else {
                    throw new DuplicateProviderException(this.name() + " module has one " + loadedProvider.name() + "[" + loadedProvider
                        .getClass()
                        .getName() + "] provider already, " + provider.name() + "[" + provider.getClass()
                                                                                              .getName() + "] is defined as 2nd provider.");
                }
            }

        }

        if (loadedProvider == null) {
            throw new ProviderNotFoundException(this.name() + " module no provider found.");
        }

        LOGGER.info("Prepare the {} provider in {} module.", loadedProvider.name(), this.name());

        // 执行 ModuleProvider 配置初始化, 通过 ModuleDefine#copyProperties 方法 从 ApplicationConfiguration 中复制属性到当前 provider中
        try {
            copyProperties(loadedProvider.createConfigBeanIfAbsent(), configuration.getProviderConfiguration(loadedProvider
                .name()), this.name(), loadedProvider.name());
        } catch (IllegalAccessException e) {
            throw new ModuleConfigException(this.name() + " module config transport to config bean failure.", e);
        }
        loadedProvider.prepare();
    }
```
- 模块提供类
```java
// 模块提供抽象类，所有的模块提供类都需要继承该抽象类，一个模块定义可以配置多个模块提供类，通过在application.yml进行切换
public abstract class ModuleProvider implements ModuleServiceHolder {
    // 模块管理器
    @Setter
    private ModuleManager manager;
    // 模块定义对象
    @Setter
    private ModuleDefine moduleDefine;

    // 模块提供对应的服务对象map
    private final Map<Class<? extends Service>, Service> services = new HashMap<>();

    public ModuleProvider() {
    }

    protected final ModuleManager getManager() {
        return manager;
    }

    /**
     * @return the name of this provider.
     */
    // 获取服务提供实现类的name，需要子类实现
    public abstract String name();

    /**
     * @return the moduleDefine name
     */
    // 定义模块提供者所实现的模块定义类
    public abstract Class<? extends ModuleDefine> module();

    /**
     *
     */
    // 创建模块定义配置对象
    public abstract ModuleConfig createConfigBeanIfAbsent();

    /**
     * In prepare stage, the moduleDefine should initialize things which are irrelative other modules.
     */
    // 准备阶段（初始化与其他模块无关的事情）
    public abstract void prepare() throws ServiceNotProvidedException, ModuleStartException;

    /**
     * In start stage, the moduleDefine has been ready for interop.
     */
    // 启动阶段（该阶段模块间可以互相操作）
    public abstract void start() throws ServiceNotProvidedException, ModuleStartException;

    /**
     * This callback executes after all modules start up successfully.
     */
    // 完成后通知阶段（在所有模块成功启动后执行）
    public abstract void notifyAfterCompleted() throws ServiceNotProvidedException, ModuleStartException;

    /**
     * @return moduleDefine names which does this moduleDefine require?
     */
    // 该模块需要依赖的其他模块名
    public abstract String[] requiredModules();

    /**
     * Register an implementation for the service of this moduleDefine provider.
     */
    // 注册服务实现类
    @Override
    public final void registerServiceImplementation(Class<? extends Service> serviceType,
        Service service) throws ServiceNotProvidedException {
        if (serviceType.isInstance(service)) {
            this.services.put(serviceType, service);
        } else {
            throw new ServiceNotProvidedException(serviceType + " is not implemented by " + service);
        }
    }

```



## Kafka-fetcher-plugin模块
kafka-fetcher-plugin 是Skywalking oap的一个可选module，名称为 "kafka-fetcher" ，它用来从kafka读取agent上送信息，一般与agent端的kafka-reporter-plugin配合使用。

通过在oap的配置文件application.yml 中启用该module，并配置相关kafka参数。
```yaml
kafka-fetcher:
  selector: ${SW_KAFKA_FETCHER:default}
  default:
    bootstrapServers: ${SW_KAFKA_FETCHER_SERVERS:localhost:9092}
```

### topic的配置和自动创建topic
在KafkaFetcherConfig类中有topic的默认名字，如果不在配置文件中修改，则默认使用之。
```java
    private String configPath = "meter-analyzer-config";
    private String topicNameOfMetrics = "skywalking-metrics";
    private String topicNameOfProfiling = "skywalking-profilings";
    private String topicNameOfTracingSegments = "skywalking-segments";
    private String topicNameOfManagements = "skywalking-managements";
    private String topicNameOfMeters = "skywalking-meters";
    private String topicNameOfLogs = "skywalking-logs";
    private boolean createTopicIfNotExist = true;

    private boolean createTopicIfNotExist = true; 
    private int partitions = 3;
    private int replicationFactor = 2;
```
另外，默认情况下，oap启动后会去检查相关topic是否已经创建，否则会自动创建。 见代码 KafkaFetcherHandlerRegister.java的构造函数
```java
        if (!missedTopics.isEmpty()) {
            log.info("Topics" + missedTopics.toString() + " not exist.");
            List<NewTopic> newTopicList = missedTopics.stream()
                                                      .map(topic -> new NewTopic(
                                                          topic,
                                                          config.getPartitions(),
                                                          (short) config.getReplicationFactor()
                                                      )).collect(Collectors.toList());

            try {
                adminClient.createTopics(newTopicList).all().get();
            } catch (Exception e) {
                throw new ModuleStartException("Failed to create Kafka Topics" + missedTopics + ".", e);
            }
        }
```
在该构造函数中还有线程池的创建：
```java
       executor = new ThreadPoolExecutor(threadPoolSize, threadPoolSize,
                                          60, TimeUnit.SECONDS,
                                          new ArrayBlockingQueue(threadPoolQueueSize),
                                          new CustomThreadFactory("KafkaConsumer"),
                                          new ThreadPoolExecutor.CallerRunsPolicy()
        );
```
这里使用的拒绝策略是CallerRunsPolicy， 意思是当线程池和队列满了后，调用者线程会来执行这个任务。

### 消费消息
这里只看我们用到的kafka-fetcher-plugin模块，SPI机制会系统初始化时调用KafkaFetcherProvider类的prepare方法和start方法，来初始化kafka配置，例如按照application.yml文件的配置，建立好和kafka消息中间件的连接，如果没有topic就新建topic。
随后，skywalking会等待java agent上报数据，包括消息类型有metrics/trace/manage/profile（jvm快照）等信息，分别对应相应的消息队列，并将kafka的消息队列和处理插件进行关联：
```java
@Override
public void start() throws ServiceNotProvidedException, ModuleStartException {
    handlerRegister.register(new JVMMetricsHandler(getManager(), config));
    handlerRegister.register(new ServiceManagementHandler(getManager(), config));
    handlerRegister.register(new TraceSegmentHandler(getManager(), config));
    handlerRegister.register(new ProfileTaskHandler(getManager(), config));
    handlerRegister.register(new MeterServiceHandler(getManager(), config));
 
    if (config.isEnableNativeProtoLog()) {
        handlerRegister.register(new LogHandler(getManager(), config));
    }
    if (config.isEnableNativeJsonLog()) {
        handlerRegister.register(new JsonLogHandler(getManager(), config));
    }
 
    handlerRegister.start();
}
```
在kafkaFetcherProvider 被oap加载后，会调用它的start方法，在start方法中会调用handlerRegister.start(); 如果java agent上报trace信息时，就会调用run方法：
```java
@Override
public void run() {
    while (true) {
        try {
            ConsumerRecords<String, Bytes> consumerRecords = consumer.poll(Duration.ofMillis(500L));
            if (!consumerRecords.isEmpty()) {
                for (final ConsumerRecord<String, Bytes> record : consumerRecords) {
                    executor.submit(() -> handlerMap.get(record.topic()).handle(record));
                }
                if (!enableKafkaMessageAutoCommit) {
                    consumer.commitAsync();
                }
            }
        } catch (Exception e) {
            log.error("Kafka handle message error.", e);
        }
    }
}

```
每隔500毫秒去访问一次kafka，oap对每一个消息都会新建一个线程去处理。处理完（consumer.commitAsync();）会更新kafka读取的游标，以免重复读取。根据之前的队列绑定关系，如果消息是一个segment信息，则会调用TraceSegmentHandler的handle方法。Handle方法会调用核心分析模块去进一步处理segment信息，也就是agent-analyzer模块：
SegmentParserServiceImpl类中的send方法：traceAnalyzer.doAnalysis(segment);完成数据转换并存入我们配置好的elasticsearch7存储中。
