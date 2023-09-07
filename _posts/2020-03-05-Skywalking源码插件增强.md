---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码插件增强


## 插件增强
用BootstrapInstrumentBoost的inject()方法，代码如下：
```
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
```
为什么要把这些类注入到Bootstrap ClassLoader中？

根据双亲委派模型，AgentClassLoader、AppClassLoader、ExtClassLoader、Bootstrap ClassLoader关系。

自定义的ClassLoader只能在最下层，而AgentClassLoader通过字节码修改的类，是不能够被BootStrapClassLoader直接使用的，所以需要注入进去。

如果是要对一个Bootstrap ClassLoader加载的类进行字节码增强，假设是HttpClient

HttpClient是由Bootstrap ClassLoader加载的，而修改HttpClient字节码的插件由AgentClassLoader加载，但是Bootstrap ClassLoader无法访问AgentClassLoader中加载的类（不能从上往下访问），所以需要把修改HttpClient字节码的插件注入到Bootstrap ClassLoader中

## AgentBuilder属性设置
```
    agentBuilder
            // 指定ByteBuddy修改的符合条件的类
            .type(pluginFinder.buildMatch())
            // 指定字节码增强工具
            .transform(new Transformer(pluginFinder))
            //指定字节码增强模式，REDEFINITION覆盖修改内容，RETRANSFORMATION保留修改内容（修改名称），
            .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
            // 注册监听器-监听异常情况，输出日子
            .with(new RedefinitionListener())
            // 对transform和异常情况监听，输出日志
            .with(new Listener())
            // 将定义好的agent注册到instrumentation
            .installOn(instrumentation);
```

## Transform工作流程
```
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
```
transform()方法处理逻辑如下：
- 查找所有能够对当前被拦截到的类生效的插件
- 如果有生效的插件，调用每个插件的define()方法去做字节码增强，返回被所有可用插件修改完之后的最终字节码
- 如果没有生效的插件，返回被拦截到的类的原生字节码

先调用PluginFinder的find()方法查找所有能够对当前被拦截到的类生效的插件：
```
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
```

用到了增强上下文EnhanceContext：
```
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
```

调用每个插件的define()方法去做字节码增强，实际调用了AbstractClassEnhancePluginDefine的define()方法：
```
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
```
define()方法处理逻辑如下：
- witness机制校验当前插件是否可用
- 调用enhance()方法进行字节码增强流程
- 将记录状态的上下文EnhanceContext设置为已增强

调用WitnessFinder的exist()方法判断witnessClass和witnessMethod是否存在，从而判断当前插件是否可用
```
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
```
- witnessClass校验时，会基于传入的classLoader构造TypePool来判断witnessClass是否存在，TypePool最终会存储到Map中
- witnessMethod校验时，先判断该方法所在的类是否在这个ClassLoader中（走witnessClass校验的流程），再判断该方法是否存在

## Transform工作流程
```
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
```
transform()方法处理逻辑如下：

- 查找所有能够对当前被拦截到的类生效的插件
- 如果有生效的插件，调用每个插件的define()方法去做字节码增强，返回被所有可用插件修改完之后的最终字节码
- 如果没有生效的插件，返回被拦截到的类的原生字节码

先调用PluginFinder的find()方法查找所有能够对当前被拦截到的类生效的插件，这块在讲解插件加载时详细讲过：
```
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
```

用到了增强上下文EnhanceContext：
```
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
```

调用每个插件的define()方法去做字节码增强，实际调用了AbstractClassEnhancePluginDefine的define()方法：
```
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
```
define()方法处理逻辑如下：

- witness机制校验当前插件是否可用
- 调用enhance()方法进行字节码增强流程
- 将记录状态的上下文EnhanceContext设置为已增强

调用WitnessFinder的exist()方法判断witnessClass和witnessMethod是否存在，从而判断当前插件是否可用
```
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
```
- witnessClass校验时，会基于传入的classLoader构造TypePool来判断witnessClass是否存在，TypePool最终会存储到Map中

- witnessMethod校验时，先判断该方法所在的类是否在这个ClassLoader中（走witnessClass校验的流程），再判断该方法是否存在













