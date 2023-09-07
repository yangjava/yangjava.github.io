---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码加载插件
Skywalking中每个组件的埋点实现就是一个插件，针对有差异的版本也可能一个组件有多个实现，分别应对不同的版本。

## 插件化架构设计
插件化架构另一种叫法是微内核架构，是一种面向功能进行拆分的可扩展性架构,通常用于存在多个版本、需要下载安装或激活才能使用的客户端应用；包含两类组件：核心系统（core system）和插件模块（plug-in modules）
- 核心系统 Core System 功能比较稳定，不会因为业务功能扩展而不断修改,通常负责和具体业务功能无关的通用功能，例如模块加载等
- 插件模块负责实现具体的业务逻辑，可以根据业务功能的需要不断地扩展，将变化部分封装在插件里面，从而达到快速灵活扩展的目的，而又不影响整体系统的稳定

## 加载插件
```text
skywalking-v8.7.0
├─apm-sniffer                   # apm模块,收集链路信息，发送给 SkyWalking OAP 服务器
├─├─apm-agent                   # Skywalking的整个Agent的入口
├─├─apm-agent-core              # Agent的核心实现
├─├─org.apache.skywalking.apm.agent.core.plugin
```
我们在 `org.apache.skywalking.apm.agent.core.plugin`包下看到关于Skywalking的插件的代码，代码如下
- Config 配置解析后生产的对象

## 加载插件
```
        final PluginFinder pluginFinder;
       	//配置项加载完成后
        …………
         //加载插件  
		 try { 
             //pluginFinder 加载插件
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

## loadPlugins方法解析
```
    // 加载所有插件
    //	AbstractClassEnhancePluginDefine 是所有插件的父类。提供了增强目标类的概述
    // 插件启动器。使用PluginResourcesResolver在插件包中加载skywalking-plugin.def配置文件，使用PluginCfg加载到内存中
public class PluginBootstrap {
    private static final ILog LOGGER = LogManager.getLogger(PluginBootstrap.class);

    /**
     * load all plugins.
     *
     * @return plugin definition list.
     */
    // 加载所有的插件
    public List<AbstractClassEnhancePluginDefine> loadPlugins() throws AgentPackageNotFoundException {
        // 初始化类加载器。资源隔离，不同的classLoader具有不同的classpath，避免乱加载
        // 初始化自定义类加载器AgentClassLoader
        AgentClassLoader.initDefaultLoader();
        // 获取各插件包下的skywalking-plugin.def配置文件
        PluginResourcesResolver resolver = new PluginResourcesResolver();
        List<URL> resources = resolver.getResources();
        // 如果没有plugin，返回
        if (resources == null || resources.size() == 0) {
            LOGGER.info("no plugin files (skywalking-plugin.def) found, continue to start application.");
            return new ArrayList<AbstractClassEnhancePluginDefine>();
        }
        //将skywalking-plugin.def配置文件读成K-V的PluginDefine类，然后放到 pluginClassList中，缓存在内存中。
        for (URL pluginUrl : resources) {
            try {
                PluginCfg.INSTANCE.load(pluginUrl.openStream());
            } catch (Throwable t) {
                LOGGER.error(t, "plugin file [{}] init failure.", pluginUrl);
            }
        }
        // 创建插件定义集合。PluginCfg提供了getPluginClassList方法 获取所有的pluginClassList   
        List<PluginDefine> pluginClassList = PluginCfg.INSTANCE.getPluginClassList();
        // 迭代获取插件定义
        List<AbstractClassEnhancePluginDefine> plugins = new ArrayList<AbstractClassEnhancePluginDefine>();
        for (PluginDefine pluginDefine : pluginClassList) {
            try {
                LOGGER.debug("loading plugin class {}.", pluginDefine.getDefineClass());
                // 将class进行实例化转为AbstractClassEnhancePluginDefine
                AbstractClassEnhancePluginDefine plugin = (AbstractClassEnhancePluginDefine) Class.forName(pluginDefine.getDefineClass(), true, AgentClassLoader
                    .getDefault()).newInstance();
                plugins.add(plugin);
            } catch (Throwable t) {
                LOGGER.error(t, "load plugin [{}] failure.", pluginDefine.getDefineClass());
            }
        }
        // 加载所有的插件 加载基于xml定义的插件
        plugins.addAll(DynamicPluginLoader.INSTANCE.load(AgentClassLoader.getDefault()));

        return plugins;

    }

}
```
加载所有插件流程如下：
- 初始化AgentClassLoader，隔离资源，不同的classLoader具有不同的classpath，避免乱加载
- 调用PluginResourcesResolver的getResources()方法
- 遍历skywalking-plugin.def的资源，调用PluginCfg的load()方法
- 通过AgentClassLoader实例化插件定义类实例
- 由于SkyWalking Agent支持通过xml定义插件，代码4)处会加载基于xml定义的插件

### 初始化AgentClassLoader
插件加载的第一步会去初始化自定义类加载器AgentClassLoader，所以我们先来看到AgentClassLoader的源码

初始化AgentClassLoader使用AgentClassLoader.initDefaultLoader()，源码如下
```
/**
     * 初始化默认classLoader
     *
     * @throws AgentPackageNotFoundException if agent package is not found.
     */
    public static void initDefaultLoader() throws AgentPackageNotFoundException {
        if (DEFAULT_LOADER == null) {
            synchronized (AgentClassLoader.class) {
                if (DEFAULT_LOADER == null) {
                    //父类是PluginBootstrap的ClassLoader()
                    DEFAULT_LOADER = new AgentClassLoader(PluginBootstrap.class.getClassLoader());
                }
            }
        }
    }

    public AgentClassLoader(ClassLoader parent) throws AgentPackageNotFoundException {
        super(parent);
        //获取AgentPackagePath的路径
        File agentDictionary = AgentPackagePath.getPath();
        //classpath 插件路径
        classpath = new LinkedList<>();
        //MOUNT对应的文件夹是"plugins"和"activations" 
        Config.Plugin.MOUNT.forEach(mountFolder -> classpath.add(new File(agentDictionary, mountFolder)));
    }
```

在Config类中，对MOUNT的默认赋值为：
```
public static List<String> MOUNT = Arrays.asList("plugins", "activations");
```
也就是说，加载/plugins和/activations文件夹下的所有插件。

- plugins
是对各种框架进行增强的插件，比如springMVC,Dubbo,RocketMq，Mysql等……
- activations
是对一些支持框架，比如日志、openTracing等工具。

### 类加载器的并行加载模式
AgentClassLoader的静态代码块里调用ClassLoader的registerAsParallelCapable()方法

AgentClassLoader中最开始有一个registerAsParallelCapable 的static方法。 用来尝试解决classloader死锁。https://github.com/apache/skywalking/pull/2016
```
自定义类加载器,负责查找插件和拦截器
/**
 * The <code>AgentClassLoader</code> represents a classloader, which is in charge of finding plugins and interceptors.
 * 自定义类加载器,负责查找插件和拦截器
 */
public class AgentClassLoader extends ClassLoader {

  // 为了解决ClassLoader死锁问题,开启类加载器的并行加载模式 
 static {
        registerAsParallelCapable();
    }
/**
* 将给定的类加载器类型注册为并行的classLoader。
*/
static boolean register(Class<? extends ClassLoader> c) {
    synchronized (loaderTypes) {
        if (loaderTypes.contains(c.getSuperclass())) {
            //如果且仅当其所有父类都具并行功能有时，才将类加载器注册为并行加载。 
            //注意：给定当前的类加载顺序，如果父类具有并行功能，所有更高级别的父类也必须具有并行能力。
            loaderTypes.add(c);
            return true;
        } else {
            return false;
        }
    }
}
```
先说下，什么是具备并行能力的类加载器？

在JDK 1.7之前，类加载器在加载类的时候是串行加载的，比如有100个类需要加载，那么就排队，加载完上一个再加载下一个，这样加载效率就很低

在JDK 1.7之后，就提供了类加载器并行能力，就是把锁的粒度变小，之前ClassLoader加载类的时候加锁的时候是用自身作为锁的

接下来我们一起来看ClassLoader的registerAsParallelCapable()方法源码：
```
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

```
并行加载的实现我们一起来看下ClassLoader的loadClass()方法源码：
```
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

```
并行加载的原理：

如果当前类加载器是支持并行加载的，就把加载类时锁的粒度降低到加载的具体的某一个类上，而不是锁掉整个类加载器

## AgentClassLoader源码解析
使用类加载器，需要重写findClas方法AgentClassLoader.findClass

除了并行类加载外，AgentClassLoader的几个关键点：

- AgentClassLoader的父类加载器是AppClassLoader
- AgentClassLoader的classpath默认是agent主目录下的plugins和activations目录
- 如果被加载的类标注了@PluginConfig注解，会加载插件的配置

将所有的jar包加载到内存中，做一个缓存。然后根据不同的class文件名，从缓存中提取文件。
```
 @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        //找到所有的jar包
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
private List<Jar> allJars;
private ReentrantLock jarScanLock = new ReentrantLock();

private List<Jar> getAllJars() {
    //	这里其实相当于一个缓存
    if (allJars == null) {
        //加锁 锁类型为可重入锁：ReentrantLock
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
```

## 加载skywalking-plugin.def配置文件
```
       PluginResourcesResolver resolver = new PluginResourcesResolver();
        List<URL> resources = resolver.getResources();
```
每个插件jar包中在resource文件下都有一个skywalking-plugin.def文件
```
// 插件资源解析器，读取所有插件的定义⽂件。使用当前classloader读取所有的插件def文件。文件名是skywalking-plugin.def
public class PluginResourcesResolver {
    private static final ILog LOGGER = LogManager.getLogger(PluginResourcesResolver.class);

    // skywalking-plugin.def文件定义了插件的切入点,加载所有的插件plugin的URL
    public List<URL> getResources() {
        List<URL> cfgUrlPaths = new ArrayList<URL>();
        Enumeration<URL> urls;
        try {
            // 文件名是skywalking-plugin.def
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
}
```

skywalking-plugin.def文件定义了插件的切入点。例如Spring的配置文件 `skywalking-plugin.def` 
```
spring-core-patch=org.apache.skywalking.apm.plugin.spring.patch.define.AopProxyFactoryInstrumentation
spring-core-patch=org.apache.skywalking.apm.plugin.spring.patch.define.AutowiredAnnotationProcessorInstrumentation
spring-core-patch=org.apache.skywalking.apm.plugin.spring.patch.define.AopExpressionMatchInstrumentation
spring-core-patch=org.apache.skywalking.apm.plugin.spring.patch.define.AspectJExpressionPointCutInstrumentation
spring-core-patch=org.apache.skywalking.apm.plugin.spring.patch.define.BeanWrapperImplInstrumentation
```

```
        if (resources == null || resources.size() == 0) {
            LOGGER.info("no plugin files (skywalking-plugin.def) found, continue to start application.");
            return new ArrayList<AbstractClassEnhancePluginDefine>();
        }
```
如果没有需要增强的插件，则返回空数组。

## 将插件配置转化为插件定义类
```
// 插件定义配置，读取skywalking-plugin.def⽂件，⽣成插件定义pluginDefine数组
public enum PluginCfg {
    INSTANCE;

    private static final ILog LOGGER = LogManager.getLogger(PluginCfg.class);

    private List<PluginDefine> pluginClassList = new ArrayList<PluginDefine>();
    private PluginSelector pluginSelector = new PluginSelector();

    // 读取 skywalking-plugin.def ⽂件，添加到 pluginClassList
    void load(InputStream input) throws IOException {
        try {
            BufferedReader reader = new BufferedReader(new InputStreamReader(input));
            String pluginDefine;
            while ((pluginDefine = reader.readLine()) != null) {
                try {
                    // 如果是# 表示注释，忽略
                    if (pluginDefine.trim().length() == 0 || pluginDefine.startsWith("#")) {
                        continue;
                    }
                    // 解析文件为PluginDefine
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
```

对于某些不需要增强的类，可以通过插件排除器进行解决，排除器配置在Skywalking配置为文件中。代码如下
```
/**
 * Select some plugins in activated plugins
 */
// 插件选择器
public class PluginSelector {
    /**
     * Exclude activated plugins
     *
     * @param pluginDefines the pluginDefines is loaded from activations directory or plugins directory
     * @return real activate plugins
     * @see Config.Plugin#EXCLUDE_PLUGINS
     */
    public List<PluginDefine> select(List<PluginDefine> pluginDefines) {
        if (!EXCLUDE_PLUGINS.isEmpty()) {
            List<String> excludes = Arrays.asList(EXCLUDE_PLUGINS.toLowerCase().split(","));
            return pluginDefines.stream()
                                .filter(item -> !excludes.contains(item.getName().toLowerCase()))
                                .collect(Collectors.toList());
        }
        return pluginDefines;
    }
}
```
转化后的插件定义类如下：
```
// 插件定义:作用是获取.def文件信息,按照name=defineClass格式进行切分
public class PluginDefine {
    /**
     * Plugin name.
     */
    // 插件名称
    private String name;

    /**
     * The class name of plugin defined.
     */
    // 插件类名
    private String defineClass;

    private PluginDefine(String name, String defineClass) {
        this.name = name;
        this.defineClass = defineClass;
    }
    // 按照key=value方式获取
    public static PluginDefine build(String define) throws IllegalPluginDefineException {
        if (StringUtil.isEmpty(define)) {
            throw new IllegalPluginDefineException(define);
        }

        String[] pluginDefine = define.split("=");
        if (pluginDefine.length != 2) {
            throw new IllegalPluginDefineException(define);
        }

        String pluginName = pluginDefine[0];
        String defineClass = pluginDefine[1];
        return new PluginDefine(pluginName, defineClass);
    }

    public String getDefineClass() {
        return defineClass;
    }

    public String getName() {
        return name;
    }
}
```

## 定义获取增强的类
```
        List<AbstractClassEnhancePluginDefine> plugins = new ArrayList<AbstractClassEnhancePluginDefine>();
        for (PluginDefine pluginDefine : pluginClassList) {
            try {
                LOGGER.debug("loading plugin class {}.", pluginDefine.getDefineClass());
                AbstractClassEnhancePluginDefine plugin = (AbstractClassEnhancePluginDefine) Class.forName(pluginDefine.getDefineClass(), true, AgentClassLoader
                    .getDefault()).newInstance();
                plugins.add(plugin);
            } catch (Throwable t) {
                LOGGER.error(t, "load plugin [{}] failure.", pluginDefine.getDefineClass());
            }
        }

        plugins.addAll(DynamicPluginLoader.INSTANCE.load(AgentClassLoader.getDefault()));
```
根据定义配置类转化为需要增强的类
```
// 类增强插件定义抽象基类。不同插件通过实现AbstractClassEnhancePluginDefine抽象类，定义不同框架的切⾯，记录调⽤链路。
public abstract class AbstractClassEnhancePluginDefine {
    private static final ILog LOGGER = LogManager.getLogger(AbstractClassEnhancePluginDefine.class);

    /**
     * New field name.
     */
    public static final String CONTEXT_ATTR_NAME = "_$EnhancedClassField_ws";

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
        String interceptorDefineClassName = this.getClass().getName();
        String transformClassName = typeDescription.getTypeName();
        if (StringUtil.isEmpty(transformClassName)) {
            LOGGER.warn("classname of being intercepted is not defined by {}.", interceptorDefineClassName);
            return null;
        }

        LOGGER.debug("prepare to enhance class {} by {}.", transformClassName, interceptorDefineClassName);
        WitnessFinder finder = WitnessFinder.INSTANCE;
        /**
         * find witness classes for enhance class
         */
        String[] witnessClasses = witnessClasses();
        if (witnessClasses != null) {
            for (String witnessClass : witnessClasses) {
                if (!finder.exist(witnessClass, classLoader)) {
                    LOGGER.warn("enhance class {} by plugin {} is not working. Because witness class {} is not existed.", transformClassName, interceptorDefineClassName, witnessClass);
                    return null;
                }
            }
        }
        List<WitnessMethod> witnessMethods = witnessMethods();
        if (!CollectionUtil.isEmpty(witnessMethods)) {
            for (WitnessMethod witnessMethod : witnessMethods) {
                if (!finder.exist(witnessMethod, classLoader)) {
                    LOGGER.warn("enhance class {} by plugin {} is not working. Because witness method {} is not existed.", transformClassName, interceptorDefineClassName, witnessMethod);
                    return null;
                }
            }
        }

        /**
         * find origin class source code for interceptor
         */
        DynamicType.Builder<?> newClassBuilder = this.enhance(typeDescription, builder, classLoader, context);

        context.initializationStageCompleted();
        LOGGER.debug("enhance class {} by {} completely.", transformClassName, interceptorDefineClassName);

        return newClassBuilder;
    }


    /**
     * Begin to define how to enhance class. After invoke this method, only means definition is finished.
     *
     * @param typeDescription target class description
     * @param newClassBuilder byte-buddy's builder to manipulate class bytecode.
     * @return new byte-buddy's builder for further manipulation.
     */
    protected DynamicType.Builder<?> enhance(TypeDescription typeDescription, DynamicType.Builder<?> newClassBuilder,
                                             ClassLoader classLoader, EnhanceContext context) throws PluginException {
        newClassBuilder = this.enhanceClass(typeDescription, newClassBuilder, classLoader);

        newClassBuilder = this.enhanceInstance(typeDescription, newClassBuilder, classLoader, context);

        return newClassBuilder;
    }

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
     * Define the {@link ClassMatch} for filtering class.
     *
     * @return {@link ClassMatch}
     */
    protected abstract ClassMatch enhanceClass();

    /**
     * Witness classname list. Why need witness classname? Let's see like this: A library existed two released versions
     * (like 1.0, 2.0), which include the same target classes, but because of version iterator, they may have the same
     * name, but different methods, or different method arguments list. So, if I want to target the particular version
     * (let's say 1.0 for example), version number is obvious not an option, this is the moment you need "Witness
     * classes". You can add any classes only in this particular release version ( something like class
     * com.company.1.x.A, only in 1.0 ), and you can achieve the goal.
     */
    protected String[] witnessClasses() {
        return new String[] {};
    }

    protected List<WitnessMethod> witnessMethods() {
        return null;
    }

    public boolean isBootstrapInstrumentation() {
        return false;
    }

    /**
     * Constructor methods intercept point. See {@link ConstructorInterceptPoint}
     *
     * @return collections of {@link ConstructorInterceptPoint}
     */
    public abstract ConstructorInterceptPoint[] getConstructorsInterceptPoints();

    /**
     * Instance methods intercept point. See {@link InstanceMethodsInterceptPoint}
     *
     * @return collections of {@link InstanceMethodsInterceptPoint}
     */
    public abstract InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints();

    /**
     * Instance methods intercept v2 point. See {@link InstanceMethodsInterceptV2Point}
     *
     * @return collections of {@link InstanceMethodsInterceptV2Point}
     */
    public abstract InstanceMethodsInterceptV2Point[] getInstanceMethodsInterceptV2Points();

    /**
     * Static methods intercept point. See {@link StaticMethodsInterceptPoint}
     *
     * @return collections of {@link StaticMethodsInterceptPoint}
     */
    public abstract StaticMethodsInterceptPoint[] getStaticMethodsInterceptPoints();

    /**
     * Instance methods intercept v2 point. See {@link InstanceMethodsInterceptV2Point}
     *
     * @return collections of {@link InstanceMethodsInterceptV2Point}
     */
    public abstract StaticMethodsInterceptV2Point[] getStaticMethodsInterceptV2Points();
}
```
通过定义各种不同的增强方法来实现增强。

## PluginFinder类解析
```
pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
```
AbstractClassEnhancePluginDefine分为3类：
- nameMatchDefine：通过类名精确匹配
- signatureMatchDefine：通过签名/注解或者其他条件间接匹配
- bootstrapClassMatchDefine：新增的bootstrap用的
```
    /**
     * 为什么这里的Map泛型是<String,LinkedList>
     * 因为对于同一个类,可能有多个插件都要对它进行字节码增强
     * key => 目标类
     * value => 所有可以对这个目标类生效的插件
     */
    private final Map<String, LinkedList<AbstractClassEnhancePluginDefine>> nameMatchDefine = new HashMap<String, LinkedList<AbstractClassEnhancePluginDefine>>();
    private final List<AbstractClassEnhancePluginDefine> signatureMatchDefine = new ArrayList<AbstractClassEnhancePluginDefine>();
    private final List<AbstractClassEnhancePluginDefine> bootstrapClassMatchDefine = new ArrayList<AbstractClassEnhancePluginDefine>();
```

插件发现者通过find方法获取类增强定义
```
// 插件发现者。其提供 #find(...) ⽅法，获得类增强插件定义
public class PluginFinder {
    // 按照类名精确匹配
    private final Map<String, LinkedList<AbstractClassEnhancePluginDefine>> nameMatchDefine = new HashMap<String, LinkedList<AbstractClassEnhancePluginDefine>>();
    // 通过签名/注解 或者其他条件 间接匹配
    private final List<AbstractClassEnhancePluginDefine> signatureMatchDefine = new ArrayList<AbstractClassEnhancePluginDefine>();
    // 新增的bootstrap用的
    private final List<AbstractClassEnhancePluginDefine> bootstrapClassMatchDefine = new ArrayList<AbstractClassEnhancePluginDefine>();

    // 插件查找器
    // nameMatchDefine：通过类名 精确匹配
    // signatureMatchDefine：通过签名/注解或者其他条件间接匹配
    // bootstrapClassMatchDefine：新增的bootstrap用的
    public PluginFinder(List<AbstractClassEnhancePluginDefine> plugins) {

        // 循环类增强插件定义对象数组，添加到nameMatchDefine/signatureMatchDefine属性，⽅便#find(...) ⽅法查找AbstractClassEnhancePluginDefine对象。
        for (AbstractClassEnhancePluginDefine plugin : plugins) {
            // 获取类增强插件定义对象的增强类匹配
            ClassMatch match = plugin.enhanceClass();

            if (match == null) {
                continue;
            }
            // 处理NameMatch为匹配的AbstractClassEnhancePluginDefine对象，添加到nameMatchDefine属性
            if (match instanceof NameMatch) {
                // NameMatch的增强可以为多个，以ClassName为Key，LinkedList<AbstractClassEnhancePluginDefine>为value
                NameMatch nameMatch = (NameMatch) match;
                LinkedList<AbstractClassEnhancePluginDefine> pluginDefines = nameMatchDefine.get(nameMatch.getClassName());
                if (pluginDefines == null) {
                    pluginDefines = new LinkedList<AbstractClassEnhancePluginDefine>();
                    nameMatchDefine.put(nameMatch.getClassName(), pluginDefines);
                }
                pluginDefines.add(plugin);
            } else {
                // 处理⾮NameMatch为匹配的AbstractClassEnhancePluginDefine对象，添加到signatureMatchDefine属性。
                signatureMatchDefine.add(plugin);
            }

            if (plugin.isBootstrapInstrumentation()) {
                bootstrapClassMatchDefine.add(plugin);
            }
        }
    }

    // 获得类增强插件定义AbstractClassEnhancePluginDefine对象
    public List<AbstractClassEnhancePluginDefine> find(TypeDescription typeDescription) {
        // 以 nameMatchDefine 属性来匹配 AbstractClassEnhancePluginDefine 对象
        List<AbstractClassEnhancePluginDefine> matchedPlugins = new LinkedList<AbstractClassEnhancePluginDefine>();
        String typeName = typeDescription.getTypeName();
        if (nameMatchDefine.containsKey(typeName)) {
            matchedPlugins.addAll(nameMatchDefine.get(typeName));
        }
        // 以 signatureMatchDefine 属性来匹配 AbstractClassEnhancePluginDefine 对象。在这个过程中，会调⽤IndirectMatch#isMatch(TypeDescription) ⽅法，进⾏匹配。
        for (AbstractClassEnhancePluginDefine pluginDefine : signatureMatchDefine) {
            IndirectMatch match = (IndirectMatch) pluginDefine.enhanceClass();
            if (match.isMatch(typeDescription)) {
                matchedPlugins.add(pluginDefine);
            }
        }

        return matchedPlugins;
    }
    // 把所有需要增强的类构建成ElementMatcher，获得全部插件的类匹配，多个插件的类匹配条件以 or 分隔
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

    public List<AbstractClassEnhancePluginDefine> getBootstrapClassMatchDefine() {
        return bootstrapClassMatchDefine;
    }
}
```
ClassMatch match = plugin.enhanceClass() 解析

enhanceClass实际是抽象接口，每个插件都有一个方法实现了这个接口。通过该方法的类型，可以获取所有插件的增强定义。
并按照类型分为按照名称精确匹配等。