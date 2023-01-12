---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---

#### **源码解读-插件加载**

SkyWalkingAgent#premain方法：

new PluginFinder(new PluginBootstrap().loadPlugins());来加载所有插件

```java
	//主要入口。使用byte-buddy 转换来增强所有在插件中定义的类。
    public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException {
        final PluginFinder pluginFinder;
       	//配置项加载完成后
        …………
         //加载插件  
		 try {
             //PluginBootstrap().loadPlugins()详见 1. loadPlugins方法解析
             //pluginFinder详见 2.pluginFinder解析
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

loadPlugins方法解析

```java
 /**
     * 加载所有插件
     *
     * @return plugin definition list.
     */
	//	AbstractClassEnhancePluginDefine 是所有插件的父类。提供了增强目标类的概述
    public List<AbstractClassEnhancePluginDefine> loadPlugins() throws AgentPackageNotFoundException {
        //初始化一个classLoder 详见1.1~1.4
        //作用：隔离资源，不同的classLoader具有不同的classpath，避免乱加载
        AgentClassLoader.initDefaultLoader();

        PluginResourcesResolver resolver = new PluginResourcesResolver();】
            
        //获取各插件包下的skywalking-plugin.def 配置文件 详见1.5~
        List<URL> resources = resolver.getResources();

        if (resources == null || resources.size() == 0) {
            LOGGER.info("no plugin files (skywalking-plugin.def) found, continue to start application.");
            return new ArrayList<AbstractClassEnhancePluginDefine>();
        }

        for (URL pluginUrl : resources) {
            try {
                //将skywalking-plugin.def配置文件读成K-V的PluginDefine类，然后放到 pluginClassList 中，缓存在内存中。                      
                PluginCfg.INSTANCE.load(pluginUrl.openStream());
            } catch (Throwable t) {
                LOGGER.error(t, "plugin file [{}] init failure.", pluginUrl);
            }
        }
		//PluginCfg提供了getPluginClassList方法 获取所有的pluginClassList    
        List<PluginDefine> pluginClassList = PluginCfg.INSTANCE.getPluginClassList();

        //创建插件定义集合
        List<AbstractClassEnhancePluginDefine> plugins = new ArrayList<AbstractClassEnhancePluginDefine>();
        //迭代获取插件定义
        for (PluginDefine pluginDefine : pluginClassList) {
            try {
                LOGGER.debug("loading plugin class {}.", pluginDefine.getDefineClass());
                //获取插件定义
                AbstractClassEnhancePluginDefine plugin = (AbstractClassEnhancePluginDefine) Class.forName(pluginDefine.getDefineClass(), true, AgentClassLoader
                    .getDefault()).newInstance();
                //放到插件集合中
                plugins.add(plugin);
            } catch (Throwable t) {
                LOGGER.error(t, "load plugin [{}] failure.", pluginDefine.getDefineClass());
            }
        }

        plugins.addAll(DynamicPluginLoader.INSTANCE.load(AgentClassLoader.getDefault()));

        return plugins;

    }
```

AgentClassLoader.initDefaultLoader();

初始化AgentClassLoader

```java
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
        //获取AgentPackagePath的路径（在配置项加载中已经说过）
        File agentDictionary = AgentPackagePath.getPath();
        //classpath 插件路径
        classpath = new LinkedList<>();
        //MOUNT对应的文件夹是"plugins"和"activations" 见 1.2
        Config.Plugin.MOUNT.forEach(mountFolder -> classpath.add(new File(agentDictionary, mountFolder)));
    }
```

在Config类中，对MOUNT的默认赋值为：

```java
public static List<String> MOUNT = Arrays.asList("plugins", "activations");
```

也就是说，加载/plugins和/activations文件夹下的所有插件。

**plugins**：是对各种框架进行增强的插件，比如springMVC,Dubbo,RocketMq，Mysql等……

**activations**：是对一些支持框架，比如日志、openTracing等工具。

AgentClassLoader中最开始有一个*registerAsParallelCapable 的*static方法。

用来尝试解决classloader死锁

https://github.com/apache/skywalking/pull/2016

```java
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

AgentClassLoader.findClass

将所有的jar包加载到内存中，做一个缓存。然后根据不同的class文件名，从缓存中提取文件。

```java
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

List<URL> resources = resolver.getResources();

加载skywalking-plugin.def配置文件

每个插件jar包中在resource文件下都有一个skywalking-plugin.def文件

**skywalking-plugin.def文件定义了插件的切入点。**



PluginFinder类解析

AbstractClassEnhancePluginDefine 分了3类：

nameMatchDefine：通过类名 **精确匹配**

signatureMatchDefine：通过签名/注解 或者其他条件 **间接匹配**

bootstrapClassMatchDefine：新增的bootstrap用的

```java
public class PluginFinder {
    // map的原因是skywalking-plugin.def配置中 key可能相同，但是实现有多中。
    private final Map<String, LinkedList<AbstractClassEnhancePluginDefine>> nameMatchDefine = new HashMap<String, LinkedList<AbstractClassEnhancePluginDefine>>();
    //
    private final List<AbstractClassEnhancePluginDefine> signatureMatchDefine = new ArrayList<AbstractClassEnhancePluginDefine>();
    //
    private final List<AbstractClassEnhancePluginDefine> bootstrapClassMatchDefine = new ArrayList<AbstractClassEnhancePluginDefine>();

    public PluginFinder(List<AbstractClassEnhancePluginDefine> plugins) {
        for (AbstractClassEnhancePluginDefine plugin : plugins) {
            //详见2.1
            ClassMatch match = plugin.enhanceClass();

            if (match == null) {
                continue;
            }
			//用一个明确的类名匹配该类。见2.1
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
			//bootstrap 插件中使用
            if (plugin.isBootstrapInstrumentation()) {
                bootstrapClassMatchDefine.add(plugin);
            }
        }
    }
	// TypeDescription 是对一个类型完整描述，包含了类全类名
    // 找到给定的一个类所有可以使用的全部插件
    // 分别从类名和辅助匹配条件两类插件中查找
    
    public List<AbstractClassEnhancePluginDefine> find(TypeDescription typeDescription) {
        List<AbstractClassEnhancePluginDefine> matchedPlugins = new LinkedList<AbstractClassEnhancePluginDefine>();
        
        //typeName就是全类名
        String typeName = typeDescription.getTypeName();
        if (nameMatchDefine.containsKey(typeName)) {
            matchedPlugins.addAll(nameMatchDefine.get(typeName));
        }
		//从间接匹配的插件中找
        for (AbstractClassEnhancePluginDefine pluginDefine : signatureMatchDefine) {
            IndirectMatch match = (IndirectMatch) pluginDefine.enhanceClass();
            if (match.isMatch(typeDescription)) {
                matchedPlugins.add(pluginDefine);
            }
        }

        return matchedPlugins;
    }
	//见下一章
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

enhanceClass实际是抽象接口，每个插件都有一个方法实现了这个接口。

以dubbo 2.7.X插件举例：

根据skywalking-plugin.def解析到，配置的切入点是DubboInstrumentation类，然后打开

发现是NameMatch.*byName，*即通过类名来进行匹配需要增强的类。