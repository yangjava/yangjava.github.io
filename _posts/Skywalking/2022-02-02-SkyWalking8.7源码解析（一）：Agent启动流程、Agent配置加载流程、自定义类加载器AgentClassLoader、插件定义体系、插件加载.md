# SkyWalking8.7源码解析（一）：Agent启动流程、Agent配置加载流程、自定义类加载器AgentClassLoader、插件定义体系、插件加载

1、Agent启动流程


找到入口方法SkyWalkingAgent的premain()方法，源码如下：

public class SkyWalkingAgent {

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
            // 1.初始化配置
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
            // 2.加载插件
            pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
        } catch (AgentPackageNotFoundException ape) {
            LOGGER.error(ape, "Locate agent.jar failure. Shutting down.");
            return;
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent initialized failure. Shutting down.");
            return;
        }
    
        // 3.定制化Agent行为
        final ByteBuddy byteBuddy = new ByteBuddy().with(TypeValidation.of(Config.Agent.IS_OPEN_DEBUGGING_CLASS));
    
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
    
        agentBuilder.type(pluginFinder.buildMatch())
                    .transform(new Transformer(pluginFinder))
                    .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
                    .with(new RedefinitionListener())
                    .with(new Listener())
                    .installOn(instrumentation);
    
        try {
            // 4.启动服务
            ServiceManager.INSTANCE.boot();
        } catch (Exception e) {
            LOGGER.error(e, "Skywalking agent boot failure.");
        }
    
        // 5.注册关闭钩子
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
小结：


2、Agent配置加载流程
Agent打包目录下的config目录中有个agent.config的文件，这就是Agent的配置文件

# 配置模块.配置项
agent.service_name = crmApp
collector.backend_service = 127.0.0.1:8080
logging.level=info
1
2
3
4
premain()中调用SnifferConfigInitializer的initializeCoreConfig()方法初始化配置，源码如下：

public class SnifferConfigInitializer {

    public static void initializeCoreConfig(String agentOptions) {
        // 加载配置信息 优先级:agent参数 > 系统环境变量 > /config/agent.config
        AGENT_SETTINGS = new Properties();
        // /config/agent.config
        try (final InputStreamReader configFileStream = loadConfig()) {
            AGENT_SETTINGS.load(configFileStream);
            for (String key : AGENT_SETTINGS.stringPropertyNames()) {
                String value = (String) AGENT_SETTINGS.get(key);
                // 配置值里的占位符替换
                // aaa = xxx
                // bbb = ${aaa}-yyy => xxx-yyy
                AGENT_SETTINGS.put(key, PropertyPlaceholderHelper.INSTANCE.replacePlaceholders(value, AGENT_SETTINGS));
            }
    
        } catch (Exception e) {
            LOGGER.error(e, "Failed to read the config file, skywalking is going to run in default config.");
        }
    
        try {
            // 系统环境变量
            overrideConfigBySystemProp();
        } catch (Exception e) {
            LOGGER.error(e, "Failed to read the system properties.");
        }
    
        // agent参数
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
    
        // 1)将配置信息映射到Config类
        initializeConfig(Config.class);
        // reconfigure logger after config initialization
        // 根据配置信息重新指定日志解析器
        configureLogger();
        LOGGER = LogManager.getLogger(SnifferConfigInitializer.class);
    
        // 检查agent名称和后端地址是否配置
        if (StringUtil.isEmpty(Config.Agent.SERVICE_NAME)) {
            throw new ExceptionInInitializerError("`agent.service_name` is missing.");
        }
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
    
        // 标记配置加载完成
        IS_INIT_COMPLETED = true;
    }
      
    /**
     * 根据配置信息重新指定日志解析器 JSON和PATTERN两种日志格式
     */
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
代码1)处将配置信息映射到Config类，Config类结构如下，Config类中就包含Agent所有的配置，可以看到一共有11个配置模块，每个配置模块下再是具体的配置项，和agent.config中指定的配置结构是相同的


小结：


3、自定义类加载器AgentClassLoader
插件加载的第一步会去初始化自定义类加载器AgentClassLoader，所以我们先来看到AgentClassLoader的源码

1）、类加载器的并行加载模式
AgentClassLoader的静态代码块里调用ClassLoader的registerAsParallelCapable()方法

/**
 * The <code>AgentClassLoader</code> represents a classloader, which is in charge of finding plugins and interceptors.
 * 自定义类加载器,负责查找插件和拦截器
 */
 public class AgentClassLoader extends ClassLoader {

    static {
        /*
         * Try to solve the classloader dead lock. See https://github.com/apache/skywalking/pull/2016
         * 为了解决ClassLoader死锁问题,开启类加载器的并行加载模式
         */
        registerAsParallelCapable();
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
 先说下，什么是具备并行能力的类加载器？

在JDK 1.7之前，类加载器在加载类的时候是串行加载的，比如有100个类需要加载，那么就排队，加载完上一个再加载下一个，这样加载效率就很低

在JDK 1.7之后，就提供了类加载器并行能力，就是把锁的粒度变小，之前ClassLoader加载类的时候加锁的时候是用自身作为锁的

接下来我们一起来看ClassLoader的registerAsParallelCapable()方法源码：

public abstract class ClassLoader {

    /**
     * 将调用该方法的类加载器注册为具备并行能力的
     * 同时满足以下两个条件时,注册才会成功
     * 1.调用该方法的类加载器实例还未创建
     * 2.调用该方法的类加载器所有父类(Object类除外)都注册为具备并行能力的
     */
    @CallerSensitive
    protected static boolean registerAsParallelCapable() {
      	// 把调用该方法的Class对象转换为ClassLoader的子类
        Class<? extends ClassLoader> callerClass =
                Reflection.getCallerClass().asSubclass(ClassLoader.class);
      	// 注册为具备并行能力的
        return ParallelLoaders.register(callerClass);
    }
    
    private static class ParallelLoaders {
        private ParallelLoaders() {}
    
        // loaderTypes中保存了所有具备并行能力的类加载器
        private static final Set<Class<? extends ClassLoader>> loaderTypes =
                Collections.newSetFromMap(new WeakHashMap<>());
        static {
            // ClassLoader本身就是支持并行加载的
            synchronized (loaderTypes) { loaderTypes.add(ClassLoader.class); }
        }
    
        static boolean register(Class<? extends ClassLoader> c) {
            synchronized (loaderTypes) {
                // 当且仅当该类加载器的所有父类都具备并行能力时,该类加载器才能被注册成功
                if (loaderTypes.contains(c.getSuperclass())) {
                    loaderTypes.add(c);
                    return true;
                } else {
                    return false;
                }
            }
        }
    
        /**
         * 判定给定的类加载器是否具备并行能力
         */
        static boolean isRegistered(Class<? extends ClassLoader> c) {
            synchronized (loaderTypes) {
                return loaderTypes.contains(c);
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
并行加载的实现我们一起来看下ClassLoader的loadClass()方法源码：

public abstract class ClassLoader {

  	private final ConcurrentHashMap<String, Object> parallelLockMap;
  	
  	private ClassLoader(Void unused, String name, ClassLoader parent) {
  	    this.name = name;
  	    this.parent = parent;
  	    this.unnamedModule = new Module(this);
  	  	// 判断当前类加载器是否具备并行能力,如果具备则对parallelLockMap进行初始化
  	    if (ParallelLoaders.isRegistered(this.getClass())) {
  	        parallelLockMap = new ConcurrentHashMap<>();
  	        package2certs = new ConcurrentHashMap<>();
  	        assertionLock = new Object();
  	    } else {
  	        // no finer-grained lock; lock on the classloader instance
  	        parallelLockMap = null;
  	        package2certs = new Hashtable<>();
  	        assertionLock = this;
  	    }
  	    this.nameAndId = nameAndId(this);
  	}
  	  
  	protected Class<?> loadClass(String name, boolean resolve)
  	    throws ClassNotFoundException
  	{
  	  	// 加锁,调用getClassLoadingLock方法获取类加载时的锁
  	    synchronized (getClassLoadingLock(name)) {
  	        // First, check if the class has already been loaded
  	        Class<?> c = findLoadedClass(name);
  	        if (c == null) {
  	            long t0 = System.nanoTime();
  	            try {
  	                if (parent != null) {
  	                    c = parent.loadClass(name, false);
  	                } else {
  	                    c = findBootstrapClassOrNull(name);
  	                }
  	            } catch (ClassNotFoundException e) {
  	                // ClassNotFoundException thrown if class not found
  	                // from the non-null parent class loader
  	            }
  	
  	            if (c == null) {
  	                // If still not found, then invoke findClass in order
  	                // to find the class.
  	                long t1 = System.nanoTime();
  	                c = findClass(name);
  	
  	                // this is the defining class loader; record the stats
  	                PerfCounter.getParentDelegationTime().addTime(t1 - t0);
  	                PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
  	                PerfCounter.getFindClasses().increment();
  	            }
  	        }
  	        if (resolve) {
  	            resolveClass(c);
  	        }
  	        return c;
  	    }
  	}
  	 
  	protected Object getClassLoadingLock(String className) {
  	  	// 默认锁是自身
  	    Object lock = this;
  	    if (parallelLockMap != null) {
  	        Object newLock = new Object();
  	      	// k:要加载的类名 v:新的锁
  	        lock = parallelLockMap.putIfAbsent(className, newLock);
  	      	// 如果是第一次put,则返回newLock
  	        if (lock == null) {
  	            lock = newLock;
  	        }
  	    }
  	    return lock;
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
并行加载的原理：

如果当前类加载器是支持并行加载的，就把加载类时锁的粒度降低到加载的具体的某一个类上，而不是锁掉整个类加载器

2）、AgentClassLoader源码解析
/**
 * The <code>AgentClassLoader</code> represents a classloader, which is in charge of finding plugins and interceptors.
 * 自定义类加载器,负责查找插件和拦截器
 */
 public class AgentClassLoader extends ClassLoader {

    static {
        /*
         * Try to solve the classloader dead lock. See https://github.com/apache/skywalking/pull/2016
         * 为了解决ClassLoader死锁问题,开启类加载器的并行加载模式
         */
        registerAsParallelCapable();
    }

    private static final ILog LOGGER = LogManager.getLogger(AgentClassLoader.class);
    /**
     * The default class loader for the agent.
     */
     private static AgentClassLoader DEFAULT_LOADER;

    private List<File> classpath;
    private List<Jar> allJars;
    private ReentrantLock jarScanLock = new ReentrantLock();

    public static AgentClassLoader getDefault() {
        return DEFAULT_LOADER;
    }

    /**
     * Init the default class loader.
     * 初始化默认的类加载器
     *
     * @throws AgentPackageNotFoundException if agent package is not found.
     */
     public static void initDefaultLoader() throws AgentPackageNotFoundException {
        if (DEFAULT_LOADER == null) {
            synchronized (AgentClassLoader.class) {
                if (DEFAULT_LOADER == null) {
                    // AgentClassLoader的父类加载器是加载PluginBootstrap的类加载器(AppClassLoader)
                    DEFAULT_LOADER = new AgentClassLoader(PluginBootstrap.class.getClassLoader());
                }
            }
        }
     }

    public AgentClassLoader(ClassLoader parent) throws AgentPackageNotFoundException {
        super(parent);
        // 获取agent主目录
        File agentDictionary = AgentPackagePath.getPath();
        classpath = new LinkedList<>();
        // 默认agent主目录下的plugins和activations目录作为AgentClassLoader的classpath
        // 只有这两个目录下的jar包才会被AgentClassLoader所加载
        Config.Plugin.MOUNT.forEach(mountFolder -> classpath.add(new File(agentDictionary, mountFolder)));
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Jar> allJars = getAllJars();
        String path = name.replace('.', '/').concat(".class");
        for (Jar jar : allJars) {
            JarEntry entry = jar.jarFile.getJarEntry(path);
            if (entry == null) {
                continue;
            }
            try {
                URL classFileUrl = new URL("jar:file:" + jar.sourceFile.getAbsolutePath() + "!/" + path);
                byte[] data;
                try (final BufferedInputStream is = new BufferedInputStream(
                    classFileUrl.openStream()); final ByteArrayOutputStream baos = new ByteArrayOutputStream()) {
                    int ch;
                    while ((ch = is.read()) != -1) {
                        baos.write(ch);
                    }
                    data = baos.toByteArray();
                }
                return processLoadedClass(defineClass(name, data, 0, data.length));
            } catch (IOException e) {
                LOGGER.error(e, "find class fail.");
            }
        }
        throw new ClassNotFoundException("Can't find " + name);
    }

    @Override
    protected URL findResource(String name) {
        List<Jar> allJars = getAllJars();
        for (Jar jar : allJars) {
            JarEntry entry = jar.jarFile.getJarEntry(name);
            if (entry != null) {
                try {
                    return new URL("jar:file:" + jar.sourceFile.getAbsolutePath() + "!/" + name);
                } catch (MalformedURLException ignored) {
                }
            }
        }
        return null;
    }

    @Override
    protected Enumeration<URL> findResources(String name) throws IOException {
        List<URL> allResources = new LinkedList<>();
        List<Jar> allJars = getAllJars();
        for (Jar jar : allJars) {
            JarEntry entry = jar.jarFile.getJarEntry(name);
            if (entry != null) {
                allResources.add(new URL("jar:file:" + jar.sourceFile.getAbsolutePath() + "!/" + name));
            }
        }

        final Iterator<URL> iterator = allResources.iterator();
        return new Enumeration<URL>() {
            @Override
            public boolean hasMoreElements() {
                return iterator.hasNext();
            }
     
            @Override
            public URL nextElement() {
                return iterator.next();
            }
        };
    }

    private Class<?> processLoadedClass(Class<?> loadedClass) {
        final PluginConfig pluginConfig = loadedClass.getAnnotation(PluginConfig.class);
        if (pluginConfig != null) {
            // Set up the plugin config when loaded by class loader at the first time.
            // Agent class loader just loaded limited classes in the plugin jar(s), so the cost of this
            // isAssignableFrom would be also very limited.
            // 如果被加载的类标注了@PluginConfig注解,会加载插件的配置
            SnifferConfigInitializer.initializeConfig(pluginConfig.root());
        }

        return loadedClass;
    }

    private List<Jar> getAllJars() {
        if (allJars == null) {
            jarScanLock.lock();
            try {
                if (allJars == null) {
                    allJars = doGetJars();
                }
            } finally {
                jarScanLock.unlock();
            }
        }

        return allJars;
    }

    private LinkedList<Jar> doGetJars() {
        LinkedList<Jar> jars = new LinkedList<>();
        for (File path : classpath) {
            if (path.exists() && path.isDirectory()) {
                String[] jarFileNames = path.list((dir, name) -> name.endsWith(".jar"));
                for (String fileName : jarFileNames) {
                    try {
                        File file = new File(path, fileName);
                        Jar jar = new Jar(new JarFile(file), file);
                        jars.add(jar);
                        LOGGER.info("{} loaded.", file.toString());
                    } catch (IOException e) {
                        LOGGER.error(e, "{} jar file can't be resolved", fileName);
                    }
                }
            }
        }
        return jars;
    }

    @RequiredArgsConstructor
    private static class Jar {
        private final JarFile jarFile;
        private final File sourceFile;
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
 124
 125
 126
 127
 128
 129
 130
 131
 132
 133
 134
 135
 136
 137
 138
 139
 140
 141
 142
 143
 144
 145
 146
 147
 148
 149
 150
 151
 152
 153
 154
 155
 156
 157
 158
 159
 160
 161
 162
 163
 164
 165
 166
 167
 168
 169
 170
 171
 172
 173
 174
 175
 176
 177
 除了并行类加载外，AgentClassLoader的几个关键点：

AgentClassLoader的父类加载器是AppClassLoader
AgentClassLoader的classpath默认是agent主目录下的plugins和activations目录
如果被加载的类标注了@PluginConfig注解，会加载插件的配置
小结：



4、插件定义体系
以dubbo 2.7.x的插件为例，我们来看下插件定义体系


1）、插件的定义
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
  拦截实例方法/构造器时，需要继承ClassInstanceMethodsEnhancePluginDefine

public abstract class ClassInstanceMethodsEnhancePluginDefine extends ClassEnhancePluginDefine {

    /**
     * @return null, means enhance no static methods.
     */
    @Override
    public StaticMethodsInterceptPoint[] getStaticMethodsInterceptPoints() {
        return null;
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
拦截静态方法时，需要继承ClassStaticMethodsEnhancePluginDefine

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
ClassInstanceMethodsEnhancePluginDefine和ClassStaticMethodsEnhancePluginDefine继承关系如下图：



AbstractClassEnhancePluginDefine是所有插件定义的顶级父类

2）、目标类匹配
ClassMatch是一个标志接口，表示类的匹配器

public interface ClassMatch {
}
1
2
NameMatch是通过完整类名精确匹配对应类

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
 除了NameMatch之外，还有比较常用的IndirectMatch用于间接匹配

public interface IndirectMatch extends ClassMatch {
    ElementMatcher.Junction buildJunction();

    // TypeDescription就是对类的描述,可以当做Class
    boolean isMatch(TypeDescription typeDescription);
}
1
2
3
4
5
6
IndirectMatch的子类，比如：PrefixMatch通过前缀匹配

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
3）、拦截器定义
public class DubboInterceptor implements InstanceMethodsAroundInterceptor {
1
DubboInterceptor继承了InstanceMethodsAroundInterceptor，InstanceMethodsAroundInterceptor源码如下：

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
类似于AOP的环绕增强，在方法执行前、方法执行后、方法抛出异常时进行代码增强

4）、插件的声明
resources目录下的skywalking-plugin.def文件是对插件的声明

# 插件名=插件定义全类名
dubbo=org.apache.skywalking.apm.plugin.asf.dubbo.DubboInstrumentation
1
2
小结：



5、插件加载
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
      	// ...
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
premain()中调用PluginBootstrap的loadPlugins()方法实例化所有插件，然后将返回值用于构造PluginFinder对象

1）、PluginBootstrap实例化所有插件
public class PluginBootstrap {
    private static final ILog LOGGER = LogManager.getLogger(PluginBootstrap.class);

    /**
     * load all plugins.
     *
     * @return plugin definition list.
     */
    public List<AbstractClassEnhancePluginDefine> loadPlugins() throws AgentPackageNotFoundException {
        // 初始化自定义类加载器AgentClassLoader
        AgentClassLoader.initDefaultLoader();
    
        PluginResourcesResolver resolver = new PluginResourcesResolver();
        // 1)拿到所有skywalking-plugin.def的资源
        List<URL> resources = resolver.getResources();
    
        if (resources == null || resources.size() == 0) {
            LOGGER.info("no plugin files (skywalking-plugin.def) found, continue to start application.");
            return new ArrayList<AbstractClassEnhancePluginDefine>();
        }
    
      	// 2)
        for (URL pluginUrl : resources) {
            try {
                PluginCfg.INSTANCE.load(pluginUrl.openStream());
            } catch (Throwable t) {
                LOGGER.error(t, "plugin file [{}] init failure.", pluginUrl);
            }
        }
    
        List<PluginDefine> pluginClassList = PluginCfg.INSTANCE.getPluginClassList();
    
        List<AbstractClassEnhancePluginDefine> plugins = new ArrayList<AbstractClassEnhancePluginDefine>();
        for (PluginDefine pluginDefine : pluginClassList) {
            try {
                LOGGER.debug("loading plugin class {}.", pluginDefine.getDefineClass());
                // 3)通过AgentClassLoader实例化插件定义类实例
                AbstractClassEnhancePluginDefine plugin = (AbstractClassEnhancePluginDefine) Class.forName(pluginDefine.getDefineClass(), true, AgentClassLoader
                    .getDefault()).newInstance();
                plugins.add(plugin);
            } catch (Throwable t) {
                LOGGER.error(t, "load plugin [{}] failure.", pluginDefine.getDefineClass());
            }
        }
    
        // 4)加载基于xml定义的插件
        plugins.addAll(DynamicPluginLoader.INSTANCE.load(AgentClassLoader.getDefault()));
    
        return plugins;
    
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
代码1)处调用PluginResourcesResolver的getResources()方法
代码2)处遍历skywalking-plugin.def的资源，调用PluginCfg的load()方法
代码3)处通过AgentClassLoader实例化插件定义类实例
由于SkyWalking Agent支持通过xml定义插件，代码4)处会加载基于xml定义的插件
先来看下PluginResourcesResolver的getResources()方法：

public class PluginResourcesResolver {
    private static final ILog LOGGER = LogManager.getLogger(PluginResourcesResolver.class);

    public List<URL> getResources() {
        List<URL> cfgUrlPaths = new ArrayList<URL>();
        Enumeration<URL> urls;
        try {
            urls = AgentClassLoader.getDefault().getResources("skywalking-plugin.def");
    
            while (urls.hasMoreElements()) {
                URL pluginUrl = urls.nextElement();
                cfgUrlPaths.add(pluginUrl);
                LOGGER.info("find skywalking plugin define in {}", pluginUrl);
            }
    
            return cfgUrlPaths;
        } catch (IOException e) {
            LOGGER.error("read resources failure.", e);
        }
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
这里实际上就是使用AgentClassLoader拿到所有skywalking-plugin.def的资源

PluginCfg的load()方法源码如下：

public enum PluginCfg {
    INSTANCE;

    private static final ILog LOGGER = LogManager.getLogger(PluginCfg.class);
    
    private List<PluginDefine> pluginClassList = new ArrayList<PluginDefine>();
    private PluginSelector pluginSelector = new PluginSelector();
    
    void load(InputStream input) throws IOException {
        try {
            BufferedReader reader = new BufferedReader(new InputStreamReader(input));
            String pluginDefine;
            while ((pluginDefine = reader.readLine()) != null) {
                try {
                    if (pluginDefine.trim().length() == 0 || pluginDefine.startsWith("#")) {
                        continue;
                    }
                    // skywalking-plugin.def转换为PluginDefine对象
                    PluginDefine plugin = PluginDefine.build(pluginDefine);
                    pluginClassList.add(plugin);
                } catch (IllegalPluginDefineException e) {
                    LOGGER.error(e, "Failed to format plugin({}) define.", pluginDefine);
                }
            }
            // 剔除配置文件中指定的不需要启用的插件
            pluginClassList = pluginSelector.select(pluginClassList);
        } finally {
            input.close();
        }
    }
    
    public List<PluginDefine> getPluginClassList() {
        return pluginClassList;
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
PluginCfg的load()方法会把skywalking-plugin.def转换为PluginDefine对象（属性：插件名和插件定义的类名），最后剔除配置文件中指定的不需要启用的插件

public class PluginDefine {
    /**
     * Plugin name.
     */
    private String name;

    /**
     * The class name of plugin defined.
     */
    private String defineClass;
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
2）、PluginFinder分类插件
public class PluginFinder {
    /**
     * 为什么这里的Map泛型是<String,LinkedList>
     * 因为对于同一个类,可能有多个插件都要对它进行字节码增强
     * key => 目标类
     * value => 所有可以对这个目标类生效的插件
     */
    private final Map<String, LinkedList<AbstractClassEnhancePluginDefine>> nameMatchDefine = new HashMap<String, LinkedList<AbstractClassEnhancePluginDefine>>();
    private final List<AbstractClassEnhancePluginDefine> signatureMatchDefine = new ArrayList<AbstractClassEnhancePluginDefine>();
    private final List<AbstractClassEnhancePluginDefine> bootstrapClassMatchDefine = new ArrayList<AbstractClassEnhancePluginDefine>();

    public PluginFinder(List<AbstractClassEnhancePluginDefine> plugins) {
        // 对加载的所有插件做分类
        for (AbstractClassEnhancePluginDefine plugin : plugins) {
            ClassMatch match = plugin.enhanceClass();
    
            if (match == null) {
                continue;
            }
    
            if (match instanceof NameMatch) {
                NameMatch nameMatch = (NameMatch) match;
                LinkedList<AbstractClassEnhancePluginDefine> pluginDefines = nameMatchDefine.get(nameMatch.getClassName());
                if (pluginDefines == null) {
                    pluginDefines = new LinkedList<AbstractClassEnhancePluginDefine>();
                    nameMatchDefine.put(nameMatch.getClassName(), pluginDefines);
                }
                pluginDefines.add(plugin);
            } else {
                signatureMatchDefine.add(plugin);
            }
    
            if (plugin.isBootstrapInstrumentation()) {
                bootstrapClassMatchDefine.add(plugin);
            }
        }
    }
    
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
    
    /**
     * 将所有插件中匹配类的逻辑做一个聚合
     *
     * @return 整个Agent需要拦截的类的所有条件的集合
     */
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
        return new ProtectiveShieldMatcher(judge);
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
小结：



参考：

SkyWalking8.7.0源码分析（如果你对SkyWalking Agent源码感兴趣的话，强烈建议看下该教程）
