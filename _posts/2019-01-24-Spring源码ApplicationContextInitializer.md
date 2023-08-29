---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring源码ApplicationContextInitializer
ApplicationContextInitializer 接口用于在 Spring 容器刷新之前执行的一个回调函数，通常用于向 SpringBoot 容器中注入属性。

其实在一般项目使用中，实现ApplicationContextInitializer机制主要是加一些我们自己的属性。

## ApplicationContextInitializer实现与使用
ApplicationContextInitializer是Spring框架原有的东西，这个类的主要作用就是在ConfigurableApplicationContext类型(或者子类型)的ApplicationContext做refresh之前，允许我们对ConfiurableApplicationContext的实例做进一步的设置和处理。

ApplicationContextInitializer接口是在spring容器刷新之前执行的一个回调函数。是在ConfigurableApplicationContext#refresh() 之前调用（当spring框架内部执行 ConfigurableApplicationContext#refresh() 方法的时候或者在SpringBoot的run()执行时），作用是初始化Spring ConfigurableApplicationContext的回调接口。

ApplicationContextInitializer是Spring框架原有的概念, 这个类的主要目的就是在ConfigurableApplicationContext类型（或者子类型）的ApplicationContext做refresh之前，允许我们对ConfigurableApplicationContext的实例做进一步的设置或者处理。
通常用于需要对应用程序上下文进行编程初始化的web应用程序中。例如，根据上下文环境注册属性源或激活概要文件。

参考ContextLoader和FrameworkServlet中支持定义contextInitializerClasses作为context-param或定义init-param。
ApplicationContextInitializer支持Order注解，表示执行顺序，越小越早执行；
```
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {

    /**
     * Initialize the given application context.
     * @param applicationContext the application to configure
     */
    void initialize(C applicationContext);

}
```
该接口典型的应用场景是web应用中需要编程方式对应用上下文做初始化。比如，注册属性源(property sources)或者针对上下文的环境信息environment激活相应的profile。

在一个Springboot应用中，classpath上会包含很多jar包，有些jar包需要在ConfigurableApplicationContext#refresh()调用之前对应用上下文做一些初始化动作，因此它们会提供自己的ApplicationContextInitializer实现类，然后放在自己的META-INF/spring.factories属性文件中，这样相应的ApplicationContextInitializer实现类就会被SpringApplication#initialize发现：
```
     // SpringApplication#initialize方法，在其构造函数内执行，从而确保在其run方法之前完成
     private void initialize(Object[] sources) {
        if (sources != null && sources.length > 0) {
            this.sources.addAll(Arrays.asList(sources));
        }
        this.webEnvironment = deduceWebEnvironment();
        setInitializers((Collection) getSpringFactoriesInstances(
                ApplicationContextInitializer.class));//   <===================
        setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
        this.mainApplicationClass = deduceMainApplicationClass();
    }
```

## 内置实现类
DelegatingApplicationContextInitializer
```
public class DelegatingApplicationContextInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {
    private static final String PROPERTY_NAME = "context.initializer.classes";
    private int order = 0;

    public DelegatingApplicationContextInitializer() {
    }

    public void initialize(ConfigurableApplicationContext context) {
        ConfigurableEnvironment environment = context.getEnvironment();
        List<Class<?>> initializerClasses = this.getInitializerClasses(environment);
        if (!initializerClasses.isEmpty()) {
            this.applyInitializerClasses(context, initializerClasses);
        }

    }
    // ...
}
```
使用环境属性 context.initializer.classes 指定的初始化器(initializers)进行初始化工作，如果没有指定则什么都不做。

用于在spring容器刷新之前初始化Spring ConfigurableApplicationContext的回调接口。（剪短说就是在容器刷新之前调用该类的 initialize 方法。并将 ConfigurableApplicationContext 类的实例传递给该方法）

通常用于需要对应用程序上下文进行编程初始化的web应用程序中。例如，根据上下文环境注册属性源或激活配置文件等。

可排序的（实现Ordered接口，或者添加@Order注解）

通过它使得我们可以把自定义实现类配置在 application.properties 里成为了可能。

### ContextIdApplicationContextInitializer
设置Spring应用上下文的ID,会参照环境属性。至于Id设置为什么值，将会参考环境属性：
* spring.application.name
* vcap.application.name
* spring.config.name
* spring.application.index
* vcap.application.instance_index

如果这些属性都没有，ID 使用 application。

### ConfigurationWarningsApplicationContextInitializer
对于一般配置错误在日志中作出警告

### ServerPortInfoApplicationContextInitializer
将内置 servlet容器实际使用的监听端口写入到 Environment 环境属性中。这样属性 local.server.port 就可以直接通过 @Value 注入到测试中，或者通过环境属性 Environment 获取。

### SharedMetadataReaderFactoryContextInitializer
创建一个 SpringBoot和ConfigurationClassPostProcessor 共用的 CachingMetadataReaderFactory对象。实现类为：ConcurrentReferenceCachingMetadataReaderFactory

### ConditionEvaluationReportLoggingListener
将 ConditionEvaluationReport写入日志。

## 实现方式
首先新建三个自定义类，实现 ApplicationContextInitializer 接口

```
public class FirstInitializer implements ApplicationContextInitializer {

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        ConfigurableEnvironment environment = applicationContext.getEnvironment();

        Map<String, Object> map = new HashMap<>();
        map.put("key1", "First");

        MapPropertySource mapPropertySource = new MapPropertySource("firstInitializer", map);
        environment.getPropertySources().addLast(mapPropertySource);

        System.out.println("run firstInitializer");
    }

}

public class SecondInitializer implements ApplicationContextInitializer {

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        ConfigurableEnvironment environment = applicationContext.getEnvironment();

        Map<String, Object> map = new HashMap<>();
        map.put("key1", "Second");

        MapPropertySource mapPropertySource = new MapPropertySource("secondInitializer", map);
        environment.getPropertySources().addLast(mapPropertySource);

        System.out.println("run secondInitializer");
    }

}

public class ThirdInitializer implements ApplicationContextInitializer {

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        ConfigurableEnvironment environment = applicationContext.getEnvironment();

        Map<String, Object> map = new HashMap<>();
        map.put("key1", "Third");

        MapPropertySource mapPropertySource = new MapPropertySource("thirdInitializer", map);
        environment.getPropertySources().addLast(mapPropertySource);

        System.out.println("run thirdInitializer");
    }

}

```

在 resources/META-INF/spring.factories 中配置
```
org.springframework.context.ApplicationContextInitializer=com.learn.springboot.initializer.FirstInitializer

```

在 main 函数中添加
```
@SpringBootApplication
public class SpringbootApplication {

    public static void main(String[] args) {
//        SpringApplication.run(SpringbootApplication.class, args);
        SpringApplication springApplication = new SpringApplication(SpringbootApplication.class);
        springApplication.addInitializers(new SecondInitializer());
        springApplication.run();
    }

}
```
在配置文件中配置
```
context.initializer.classes=com.learn.springboot.initializer.ThirdInitializer
```
运行项目，查看控制台：
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.2.5.RELEASE)

run thirdInitializer
run firstInitializer
run secondInitializer
```
可以看到配置生效了，并且三种配置优先级不一样，配置文件优先级最高，spring.factories 其次，代码最后。

获取属性值
```
@RestController
public class HelloController {

    private ApplicationContext applicationContext;

    public HelloController(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    @RequestMapping("/getAttributes")
    public String getAttributes() {
        String value = applicationContext.getEnvironment().getProperty("key1");
        System.out.println(value);
        return value;
    }

}
```
启动项目，访问
```
http://localhost:8080/getAttributes
```
查看控制台输出：
```
Third
```
发现同名的 key,只会存在一个，并且只存第一次设置的值。

通过 @Order 注解修改执行顺序
注：@order 值越小，执行优先级越高

不同配置方式下，执行顺序
```
@Order(1)
public class SecondInitializer implements ApplicationContextInitializer {
    ......
}

@Order(2)
public class FirstInitializer implements ApplicationContextInitializer {
    ......
}

@Order(3)
public class ThirdInitializer implements ApplicationContextInitializer {
    ......
}
```
运行项目，查看控制台：
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.2.5.RELEASE)

run thirdInitializer
run secondInitializer
run firstInitializer
```
可以看到通过 @Order ** 注解是可以改变spring.factories** 和代码形式的执行顺序的，但是
```
application.properties
```
配置文件的优先级还是最高的。

同一配置下，执行顺序
```
@Order(1)
public class FourthInitializer implements ApplicationContextInitializer {

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        ConfigurableEnvironment environment = applicationContext.getEnvironment();

        Map<String, Object> map = new HashMap<>();
        map.put("key1", "Fourth");

        MapPropertySource mapPropertySource = new MapPropertySource("FourthInitializer", map);
        environment.getPropertySources().addLast(mapPropertySource);

        System.out.println("run fourthInitializer");
    }

}
```
在application.properties 文件中配置
```
context.initializer.classes=com.learn.springboot.initializer.ThirdInitializer,com.learn.springboot.initializer.FourthInitializer

```
运行项目，查看控制台：
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.2.5.RELEASE)

run fourthInitializer
run thirdInitializer
run secondInitializer
run firstInitializer
```
可以看到同一配置方式， @Order 注解也可以起作用。

## 系统初始化器原理解析
在 resources/META-INF/spring.factories 中配置实现原理
在 SpringApplication 初始化时通过 SpringFactoriesLoader 获取到配置在
```
META-INF/spring.factories
```
文件中的 ApplicationContextInitializer 的所有实现类.
```
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    ......
    // 设置系统初始化器
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    ......
}
// 获取工厂实例对象
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}
// 获取工厂实例对象
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    // 获取类加载器
    ClassLoader classLoader = getClassLoader();
    // 使用名称并确保唯一以防止重复
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 创建工厂实例对象
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    // 对工厂实例对象列表进行排序
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}

// 创建工厂实例对象
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
        ClassLoader classLoader, Object[] args, Set<String> names) {
    List<T> instances = new ArrayList<>(names.size());
    for (String name : names) {
        try {
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}
```
在 run 方法中回调 ApplicationContextInitializer 接口函数

```
public ConfigurableApplicationContext run(String... args) {
    ......
    // 准备上下文环境注入系统初始化信息 
    prepareContext(context, environment, listeners, applicationArguments, printedBanner);
    ......
}
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
        SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    ......
    // 应用初始化器   
    applyInitializers(context);
    ......
}
protected void applyInitializers(ConfigurableApplicationContext context) {
    for (ApplicationContextInitializer initializer : getInitializers()) {
        // 判断子类是否是 ConfigurableApplicationContext 类型
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(),
                ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        // 回调 ApplicationContextInitializer接口的 initialize 方法
        initializer.initialize(context);
    }
}
```
获取初始化器列表
```
// 获取在 SpringApplication 构造函数中设置的初始化器列表
public Set<ApplicationContextInitializer<?>> getInitializers() {
    return asUnmodifiableOrderedSet(this.initializers);
}
// 对初始化器列表进行排序
private static <E> Set<E> asUnmodifiableOrderedSet(Collection<E> elements) {
    List<E> list = new ArrayList<>(elements);
    list.sort(AnnotationAwareOrderComparator.INSTANCE);
    return new LinkedHashSet<>(list);
}
```
在 main 函数配置实现原理
在之前我们知道 SpringApplication 初始化之后,就已经把 META-INF/spring.factories 中配置的初始化实现类添加到 initializers列表中了，然后通过
addInitializers 方法，添加自定义的实现类：
```
public static void main(String[] args) {
    // SpringApplication.run(SpringbootApplication.class, args);
    SpringApplication springApplication = new SpringApplication(SpringbootApplication.class);
    springApplication.addInitializers(new ThirdInitializer());
    springApplication.run();
}
public void addInitializers(ApplicationContextInitializer<?>... initializers) {
    this.initializers.addAll(Arrays.asList(initializers));
}
```

在配置文件中配置实现原理
在配置文件中配置方式，主要通过内置的 DelegatingApplicationContextInitializer 实现的,它实现了 Order
方法，所以优先级最高
```
private int order = 0;  

@Override
public int getOrder() {
    return this.order;
}
```
然后我们看下它的initialize 方法实现：
```
@Override
public void initialize(ConfigurableApplicationContext context) {
    // 获取上下文环境变量
    ConfigurableEnvironment environment = context.getEnvironment();
    // 从上下文环境变量中获取指定初始化类列表
    List<Class<?>> initializerClasses = getInitializerClasses(environment);
    if (!initializerClasses.isEmpty()) {
        // 应用初始化器
        applyInitializerClasses(context, initializerClasses);
    }
}
```
从上下文环境变量获取指定的属性名,并实例化对象

```
private static final String PROPERTY_NAME = "context.initializer.classes";

private List<Class<?>> getInitializerClasses(ConfigurableEnvironment env) {
    // 从上下文环境变量获取指定的属性名
    String classNames = env.getProperty(PROPERTY_NAME);
    List<Class<?>> classes = new ArrayList<>();
    if (StringUtils.hasLength(classNames)) {
        // 将逗号分割的属性值逐个取出
        for (String className : StringUtils.tokenizeToStringArray(classNames, ",")) {
            // 实例化对象并添加到列表中
            classes.add(getInitializerClass(className));
        }
    }
    return classes;
}
private Class<?> getInitializerClass(String className) throws LinkageError {
    try {
        Class<?> initializerClass = ClassUtils.forName(className, ClassUtils.getDefaultClassLoader());
        Assert.isAssignable(ApplicationContextInitializer.class, initializerClass);
        return initializerClass;
    }
    catch (ClassNotFoundException ex) {
        throw new ApplicationContextException("Failed to load context initializer class [" + className + "]", ex);
    }
}
```

## ApplicationContextInitializer核心机制

### 如何加载Initializer?
其实在一开始new SpringApplication的时候，它干了一件事情，看一下SpringApplication的构造方法。
```
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
   this.resourceLoader = resourceLoader;
   Assert.notNull(primarySources, "PrimarySources must not be null");
   this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
   this.webApplicationType = WebApplicationType.deduceFromClasspath();
   this.bootstrapRegistryInitializers = new ArrayList<>(
         getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
   setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
   setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
   this.mainApplicationClass = deduceMainApplicationClass();
}
```
上面我标红的这一行代码，setInitializers(),其中有一个getSpringFactoriesInstances()方法。

```
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
   ClassLoader classLoader = getClassLoader();
   // Use names and ensure unique to protect against duplicates
   Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
   List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
   AnnotationAwareOrderComparator.sort(instances);
   return instances;
}
```
跟进来就会发现，它里面用的就是SpringFactoriesLoader.loadFactoryNames()方法，通过这个Spring核心类就拿到ApplicationContextInitializer下所有的实现类。

然后调用createSpringFactoriesInstances()，创建Spring的实例。通过反射创建这些对象，然后setInitializers(Collection<T>)，就Set到Application当中了。

### 如何Initializer？
接着看一下怎么使用Initializer。

使用其实就在SpringApplication.run()方法里面，这个run()方法是平时我们了解springboot的重点，每一个方法就代表一个很重要的步骤，我简单注释了一下。
```
public ConfigurableApplicationContext run(String... args) {
   long startTime = System.nanoTime();
   # 创建上下文
   DefaultBootstrapContext bootstrapContext = createBootstrapContext();
   ConfigurableApplicationContext context = null;
   configureHeadlessProperty();
   # 拿到SpringApplication的Listenners
   SpringApplicationRunListeners listeners = getRunListeners(args);
   listeners.starting(bootstrapContext, this.mainApplicationClass);
   try {
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      # 准备环境
      ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
      configureIgnoreBeanInfo(environment);
      # 打印Banner
      Banner printedBanner = printBanner(environment);
      # 创建ApplicationContext
      context = createApplicationContext();
      # 设置启动类
      context.setApplicationStartup(this.applicationStartup);
      # 进行Context准备工作,就是准备一些属性
      prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
      # 刷新Context,这个是最重要的 像都知道的IOC容器,底层的applicationContext.refresh()就是通过spring refresh创建的,
      refreshContext(context);
      afterRefresh(context, applicationArguments);
      Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
      }
      listeners.started(context, timeTakenToStartup);
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, listeners);
      throw new IllegalStateException(ex);
   }
   try {
      Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
      listeners.ready(context, timeTakenToReady);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, null);
      throw new IllegalStateException(ex);
   }
   return context;
}
```
上面的注释主要描述 整个，run()方法执行的过程。

回到正题，如何使用的Initializer呢？其实是在准备阶段，就是prepareContext()方法里。
```
private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context,
      ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
      ApplicationArguments applicationArguments, Banner printedBanner) {
   context.setEnvironment(environment);
   postProcessApplicationContext(context);
   applyInitializers(context);
   listeners.contextPrepared(context);
   bootstrapContext.close(context);
   if (this.logStartupInfo) {
      logStartupInfo(context.getParent() == null);
      logStartupProfileInfo(context);
   }
   // Add boot specific singleton beans
   ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
   beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
   if (printedBanner != null) {
      beanFactory.registerSingleton("springBootBanner", printedBanner);
   }
   if (beanFactory instanceof AbstractAutowireCapableBeanFactory) {
      ((AbstractAutowireCapableBeanFactory) beanFactory).setAllowCircularReferences(this.allowCircularReferences);
      if (beanFactory instanceof DefaultListableBeanFactory) {
         ((DefaultListableBeanFactory) beanFactory)
               .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
      }
   }
   if (this.lazyInitialization) {
      context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
   }
   // Load the sources
   Set<Object> sources = getAllSources();
   Assert.notEmpty(sources, "Sources must not be empty");
   load(context, sources.toArray(new Object[0]));
   listeners.contextLoaded(context);
}
```
在初始化容器准备过程中，其中有一个applyInitializers(context)方法。跟进去看一下
```
protected void applyInitializers(ConfigurableApplicationContext context) {
   for (ApplicationContextInitializer initializer : getInitializers()) {
      Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(),
            ApplicationContextInitializer.class);
      Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
      initializer.initialize(context);
   }
}
```
这个方法里主要是找到之前Set的所有Initializers，然后调它的initializer.initialize(context)方法。

这样我们自定义的MyApplicationContextInitializer的initialize()方法就被执行了。

## SpringBoot Initializer示例
最后看一个Srpingboot简单的ApplicationContextInitializer的实现类ContextIdApplicationContextInitializer，看它主要干了什么事
```
# Application Context Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer,\
org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
```
实现都很简单，就是看我们有没有配置spring.application.name，没有则用默认的“application”。