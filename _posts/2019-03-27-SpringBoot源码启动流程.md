---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot启动流程
从源代码的角度来看看Spring Boot的启动过程到底是怎么样的


## SpringBoot启动
SpringBoot应用的启动过程。这个启动过程比较复杂，在此我只介绍核心的知识点。

其启动过程大概分为两步。
- 初始化SpringApplication对象
- 执行SpringApplication对象的run方法

## 启动SpringBoot启动类
springboot的启动类入口如下所示：

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
    
}
```
从上面代码可以看出，Annotation定义（@SpringBootApplication）和类定义（SpringApplication.run）最为耀眼，所以要揭开SpringBoot的神秘面纱，我们要从这两位开始就可以了。


进入源码**SpringApplication.run(Application.class, args);**

```java
	/**
	 * Static helper that can be used to run a {@link SpringApplication} from the
	 * specified source using default settings.
	 * @param primarySource the primary source to load
	 * @param args the application arguments (usually passed from a Java main method)
	 * @return the running {@link ApplicationContext}
	 */
	public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
		return run(new Class<?>[] { primarySource }, args);
	}

    /**
     * Static helper that can be used to run a {@link SpringApplication} from the
     * specified sources using default settings and user supplied arguments.
     * @param primarySources the primary sources to load
     * @param args the application arguments (usually passed from a Java main method)
     * @return the running {@link ApplicationContext}
     */
    public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
            return new SpringApplication(primarySources).run(args);
            }   
```

创建一个构造一个**SpringApplication**的实例，然后执行run

## SpringApplication构造器

```java
	/**
	 * Create a new {@link SpringApplication} instance. The application context will load
	 * beans from the specified primary sources (see {@link SpringApplication class-level}
	 * documentation for details. The instance can be customized before calling
	 * {@link #run(String...)}.
	 * @param resourceLoader the resource loader to use
	 * @param primarySources the primary bean sources
	 * @see #run(Class, String[])
	 * @see #setSources(Set)
	 */
	@SuppressWarnings({ "unchecked", "rawtypes" })
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
        //把Application.class设置为属性存储起来
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
        //设置应用类型是Standard还是Web
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
		this.bootstrapRegistryInitializers = getBootstrapRegistryInitializersFromSpringFactories();
        //设置初始化器(Initializer),最后会调用这些初始化器
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
        //设置监听器(Listener)
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

- 设置应用类型，通过deduceFromClasspath 方法来进行 Web 应用类型的推断
- **设置初始化器(Initializer):**使用SpringFactoriesLoader在应用的classpath中查找并加载所有可用的`ApplicationContextInitializer`。
- **设置监听器(Listener):**使用`SpringFactoriesLoader`在应用的classpath中查找并加载所有可用的`ApplicationListener`。
- 推断并设置main方法的定义类。

### 设置应用类型

SpringApplication 的构造方法中便调用了 WebApplication Type 的deduceFromClasspath 方法来进行 Web 应用类型的推断。
```java
public enum WebApplicationType {

	/**
	 * The application should not run as a web application and should not start an
	 * embedded web server.
	 */
	NONE,

	/**
	 * The application should run as a servlet-based web application and should start an
	 * embedded servlet web server.
	 */
	SERVLET,

	/**
	 * The application should run as a reactive web application and should start an
	 * embedded reactive web server.
	 */
	REACTIVE;

	private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };

	private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";

	private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";

	private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";

	private static final String SERVLET_APPLICATION_CONTEXT_CLASS = "org.springframework.web.context.WebApplicationContext";

	private static final String REACTIVE_APPLICATION_CONTEXT_CLASS = "org.springframework.boot.web.reactive.context.ReactiveWebApplicationContext";

	static WebApplicationType deduceFromClasspath() {
		if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
				&& !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
			return WebApplicationType.REACTIVE;
		}
		for (String className : SERVLET_INDICATOR_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return WebApplicationType.NONE;
			}
		}
		return WebApplicationType.SERVLET;
	}
	......
```
WebApplicationType 为枚举类， 里面定义了三个枚举值，分别是 NONE、SERVLET、REACTIVE。
下面三个枚举值的介绍：
| 枚举值                                  | 说明                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| NONE	                            | 应用程序不应作为 web 应用程序运行，也不应启动嵌入式 web 服务器。 |
| SERVLET                           | 应用程序应作为基于 servlet 的 web 应用程序运行，并应启动嵌入式 servlet web 服务器。     |
| REACTIVE	                        | 应用程序应作为反应式 web 应用程序运行，并应启动嵌入式反应式 web 服务器。                                      |

言外之意，我们的 Springboot 项目的程序类型有三种：
- 非 web 应用程序（不内嵌服务器）；
- 内嵌基于 servlet 的 web 服务器（如：Tomcat，Jetty，Undertow 等，其实现在大多Java网站应用都是采用的基于 Tomcat 的 servlet 类型服务器）；
- 内嵌基于反应式的 web 服务器（如： Netty）；

既然我们了解到了 Springboot 应用程序的三种类型，那么他又是如何确定当前程序类型的呢？
我们回到刚刚赋值应用类型的那句代码中，可以看到 WebApplicationType.deduceFromClasspath() 这个方法。

方法 deduceFromClasspath 是基于 classpath 中类是否存在来进行类型推断的，就是判断指定的类是否存在于 classpath 下， 并根据判断的结果来进行组合推断该应用属于什么类型。  
deduceFromClasspath 在判断的过程中用到了 ClassUtils 的 isPresent 方法。isPresent方法的核心机制就是通过反射创建指定的类，根据在创建过程中是否抛出异常来判断该类  

（1）当项目中存在 DispatcherHandler 这个类，且不存在 DispatcherServlet 类和ServletContainer时，程序的应用类型就是 REACTIVE，也就是他会加载嵌入一个反应式的 web 服务器。

（2）当项目中 Servlet 和 ConfigurableWebApplicationContext 其中一个不存在时，则程序的应用类型为 NONE，它并不会加载内嵌任何 web 服务器。

（3）除了上面两种情况外，其余的都按 SERVLET 类型处理，会内嵌一个 servlet 类型的 web 服务器。

而上面的这些类的判定，都来源于 Spring 的相关依赖包，而这依赖包是否需要导入，也是开发者所决定的，所以说开发者可以决定程序的应用类型，并不是 Srpingboot 本身决定的。

当我们引入

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```
WebApplicationType的值为SERVLET。
但我们去掉依赖时，则WebApplicationType的值为NONE。

### 设置初始化器(Initializer)

```
//设置初始化器(Initializer),最后会调用这些初始化器
setInitializers((Collection) getSpringFactoriesInstances( ApplicationContextInitializer.class));
```

我们先来看看**getSpringFactoriesInstances( ApplicationContextInitializer.class)**

```
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

// 这里的入参type就是ApplicationContextInitializer.class
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
        Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    // 使用Set保存names来避免重复元素
    Set<String> names = new LinkedHashSet<>(
            SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 根据names来进行实例化
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
            classLoader, args, names);
    // 对实例进行排序
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

这里面首先会根据入参type读取所有的names(是一个String集合)，然后根据这个集合来完成对应的实例化操作：

```
// 入参就是ApplicationContextInitializer.class
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
  String factoryClassName = factoryClass.getName();

  try {
      //从类路径的META-INF/spring.factories中加载所有默认的自动配置类
      Enumeration<URL> urls = classLoader != null?classLoader.getResources("META-INF/spring.factories"):ClassLoader.getSystemResources("META-INF/spring.factories");
      ArrayList result = new ArrayList();

      while(urls.hasMoreElements()) {
          URL url = (URL)urls.nextElement();
          Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
          //获取ApplicationContextInitializer.class的所有值
          String factoryClassNames = properties.getProperty(factoryClassName);
          result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
      }

      return result;
  } catch (IOException var8) {
      throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() + "] factories from location [" + "META-INF/spring.factories" + "]", var8);
  }
}
```

这个方法会尝试从类路径的META-INF/spring.factories处读取相应配置文件，然后进行遍历，读取配置文件中Key为：org.springframework.context.ApplicationContextInitializer的value。以spring-boot-autoconfigure这个包为例，它的META-INF/spring.factories部分定义如下所示：

```
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.AutoConfigurationReportLoggingInitializer
```

这两个类名会被读取出来，然后放入到Set<String>集合中，准备开始下面的实例化操作：

```
// parameterTypes: 上一步得到的names集合
private <T> List<T> createSpringFactoriesInstances(Class<T> type,
        Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args,
        Set<String> names) {
    List<T> instances = new ArrayList<T>(names.size());
    for (String name : names) {
        try {
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            //确认被加载类是ApplicationContextInitializer的子类
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            //反射实例化对象
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            //加入List集合中
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException(
                    "Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}
```

确认被加载的类确实是org.springframework.context.ApplicationContextInitializer的子类，然后就是得到构造器进行初始化，最后放入到实例列表中。

因此，所谓的初始化器就是org.springframework.context.ApplicationContextInitializer的实现类，这个接口是这样定义的：

```
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {

    void initialize(C applicationContext);

}
```

在Spring上下文被刷新之前进行初始化的操作。典型地比如在Web应用中，注册Property Sources或者是激活Profiles。Property Sources比较好理解，就是配置文件。Profiles是Spring为了在不同环境下(如DEV，TEST，PRODUCTION等)，加载不同的配置项而抽象出来的一个实体。

### 设置监听器(Listener)


下面开始设置监听器：

```
setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
```

我们还是跟进代码看看**getSpringFactoriesInstances**

```
// 这里的入参type是：org.springframework.context.ApplicationListener.class
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
        Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    // Use names and ensure unique to protect against duplicates
    Set<String> names = new LinkedHashSet<String>(
            SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
            classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

可以发现，这个加载相应的类名，然后完成实例化的过程和上面在设置初始化器时如出一辙，同样，还是以spring-boot-autoconfigure这个包中的spring.factories为例，看看相应的Key-Value：

```
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
```

这10个监听器会贯穿springBoot整个生命周期。至此，对于SpringApplication实例的初始化过程就结束了。

### SpringApplication.run方法


完成了SpringApplication实例化，下面开始调用run方法：

```
public ConfigurableApplicationContext run(String... args) {
    // 计时工具
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();

    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();

    configureHeadlessProperty();

    // 第一步：获取并启动监听器
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting();
    
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

        // 第二步：根据SpringApplicationRunListeners以及参数来准备环境
        ConfigurableEnvironment environment = prepareEnvironment(listeners,applicationArguments);
        configureIgnoreBeanInfo(environment);

        // 准备Banner打印器 - 就是启动Spring Boot的时候打印在console上的ASCII艺术字体
        Banner printedBanner = printBanner(environment);

        // 第三步：创建Spring容器
        context = createApplicationContext();

        exceptionReporters = getSpringFactoriesInstances(
                SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);

        // 第四步：Spring容器前置处理
        prepareContext(context, environment, listeners, applicationArguments,printedBanner);

        // 第五步：刷新容器
        refreshContext(context);
　　　　 // 第六步：Spring容器后置处理
        afterRefresh(context, applicationArguments);

  　　　 // 第七步：发出结束执行的事件
        listeners.started(context);
        // 第八步：执行Runners
        this.callRunners(context, applicationArguments);
        stopWatch.stop();
        // 返回容器
        return context;
    }
    catch (Throwable ex) {
        handleRunFailure(context, listeners, exceptionReporters, ex);
        throw new IllegalStateException(ex);
    }
}
```

- 第一步：获取并启动监听器
- 第二步：根据SpringApplicationRunListeners以及参数来准备环境
- 第三步：创建Spring容器
- 第四步：Spring容器前置处理
- 第五步：刷新容器
- 第六步：Spring容器后置处理
- 第七步：发出结束执行的事件
- 第八步：执行Runners

下面具体分析。

#### 第一步：获取并启动监听器

从命名我们就可以知道它是一个监听者，那纵观整个启动流程我们会发现，它其实是用来在整个启动流程中接收不同执行点事件通知的监听者。源码如下：

**获取监听器**

跟进`getRunListeners`方法：

```
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

这里仍然利用了getSpringFactoriesInstances方法来获取实例，大家可以看看前面的这个方法分析，从META-INF/spring.factories中读取Key为org.springframework.boot.**SpringApplicationRunListener**的Values：

```
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
```

getSpringFactoriesInstances中反射获取实例时会触发`EventPublishingRunListener`的构造函数，我们来看看`EventPublishingRunListener`的构造函数：

```
 public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
    private final SpringApplication application;
    private final String[] args;
    //广播器
    private final SimpleApplicationEventMulticaster initialMulticaster;

    public EventPublishingRunListener(SpringApplication application, String[] args) {
        this.application = application;
        this.args = args;
        this.initialMulticaster = new SimpleApplicationEventMulticaster();
        Iterator var3 = application.getListeners().iterator();

        while(var3.hasNext()) {
            ApplicationListener<?> listener = (ApplicationListener)var3.next();
            //将上面设置到SpringApplication的十一个监听器全部添加到SimpleApplicationEventMulticaster这个广播器中
            this.initialMulticaster.addApplicationListener(listener);
        }

    }
    //略...
}
```

我们看到**EventPublishingRunListener里面有一个广播器，EventPublishingRunListener 的构造方法将SpringApplication的十一个监听器全部添加到SimpleApplicationEventMulticaster这个广播器中，**我们来看看是如何添加到广播器：

```
 public abstract class AbstractApplicationEventMulticaster implements ApplicationEventMulticaster, BeanClassLoaderAware, BeanFactoryAware {
    //广播器的父类中存放保存监听器的内部内
    private final AbstractApplicationEventMulticaster.ListenerRetriever defaultRetriever = new AbstractApplicationEventMulticaster.ListenerRetriever(false);

    @Override
    public void addApplicationListener(ApplicationListener<?> listener) {
        synchronized (this.retrievalMutex) {
            Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
            if (singletonTarget instanceof ApplicationListener) {
                this.defaultRetriever.applicationListeners.remove(singletonTarget);
            }
            //内部类对象
            this.defaultRetriever.applicationListeners.add(listener);
            this.retrieverCache.clear();
        }
    }

    private class ListenerRetriever {
        //保存所有的监听器
        public final Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet();
        public final Set<String> applicationListenerBeans = new LinkedHashSet();
        private final boolean preFiltered;

        public ListenerRetriever(boolean preFiltered) {
            this.preFiltered = preFiltered;
        }

        public Collection<ApplicationListener<?>> getApplicationListeners() {
            LinkedList<ApplicationListener<?>> allListeners = new LinkedList();
            Iterator var2 = this.applicationListeners.iterator();

            while(var2.hasNext()) {
                ApplicationListener<?> listener = (ApplicationListener)var2.next();
                allListeners.add(listener);
            }

            if (!this.applicationListenerBeans.isEmpty()) {
                BeanFactory beanFactory = AbstractApplicationEventMulticaster.this.getBeanFactory();
                Iterator var8 = this.applicationListenerBeans.iterator();

                while(var8.hasNext()) {
                    String listenerBeanName = (String)var8.next();

                    try {
                        ApplicationListener<?> listenerx = (ApplicationListener)beanFactory.getBean(listenerBeanName, ApplicationListener.class);
                        if (this.preFiltered || !allListeners.contains(listenerx)) {
                            allListeners.add(listenerx);
                        }
                    } catch (NoSuchBeanDefinitionException var6) {
                        ;
                    }
                }
            }

            AnnotationAwareOrderComparator.sort(allListeners);
            return allListeners;
        }
    }
    //略...
}
```

上述方法定义在SimpleApplicationEventMulticaster父类AbstractApplicationEventMulticaster中。关键代码为this.defaultRetriever.applicationListeners.add(listener);，这是一个内部类，用来保存所有的监听器。也就是在这一步，将spring.factories中的监听器传递到SimpleApplicationEventMulticaster中。我们现在知道EventPublishingRunListener中有一个广播器SimpleApplicationEventMulticaster，SimpleApplicationEventMulticaster广播器中又存放所有的监听器。

**启动监听器**

我们上面一步通过`getRunListeners`方法获取的监听器为`EventPublishingRunListener，从名字可以看出是启动事件发布监听器，主要用来发布启动事件。`

```
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
    private final SpringApplication application;
    private final String[] args;
    private final SimpleApplicationEventMulticaster initialMulticaster;
```

我们先来看看**SpringApplicationRunListener这个接口**

```
package org.springframework.boot;
public interface SpringApplicationRunListener {

    // 在run()方法开始执行时，该方法就立即被调用，可用于在初始化最早期时做一些工作
    void starting();
    // 当environment构建完成，ApplicationContext创建之前，该方法被调用
    void environmentPrepared(ConfigurableEnvironment environment);
    // 当ApplicationContext构建完成时，该方法被调用
    void contextPrepared(ConfigurableApplicationContext context);
    // 在ApplicationContext完成加载，但没有被刷新前，该方法被调用
    void contextLoaded(ConfigurableApplicationContext context);
    // 在ApplicationContext刷新并启动后，CommandLineRunners和ApplicationRunner未被调用前，该方法被调用
    void started(ConfigurableApplicationContext context);
    // 在run()方法执行完成前该方法被调用
    void running(ConfigurableApplicationContext context);
    // 当应用运行出错时该方法被调用
    void failed(ConfigurableApplicationContext context, Throwable exception);
}
```

SpringApplicationRunListener接口在Spring Boot 启动初始化的过程中各种状态时执行，我们也可以添加自己的监听器，在SpringBoot初始化时监听事件执行自定义逻辑，我们先来看看SpringBoot启动时第一个启动事件listeners.starting()：

```
@Override
public void starting() {
    //关键代码，先创建application启动事件`ApplicationStartingEvent`
    this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
}
```

这里先创建了一个启动事件**ApplicationStartingEvent**，我们继续跟进SimpleApplicationEventMulticaster，有个核心方法：

```
@Override
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    //通过事件类型ApplicationStartingEvent获取对应的监听器
    for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        //获取线程池，如果为空则同步处理。这里线程池为空，还未没初始化。
        Executor executor = getTaskExecutor();
        if (executor != null) {
            //异步发送事件
            executor.execute(() -> invokeListener(listener, event));
        }
        else {
            //同步发送事件
            invokeListener(listener, event);
        }
    }
}
```

这里会根据事件类型`ApplicationStartingEvent`获取对应的监听器，在容器启动之后执行响应的动作，有如下4种监听器：

```
0 = {LoggingApplicationListener@1339} 
1 = {BackgroundPreinitializer@1340} 
2 = {DelegatingApplicationListener@1341} 
3 = {LiquibaseServiceLocatorApplicationListener@1342} 
```

我们选择springBoot 的日志监听器来进行讲解，核心代码如下：

```
@Override
public void onApplicationEvent(ApplicationEvent event) {
    //在springboot启动的时候
    if (event instanceof ApplicationStartedEvent) {
        onApplicationStartedEvent((ApplicationStartedEvent) event);
    }
    //springboot的Environment环境准备完成的时候
    else if (event instanceof ApplicationEnvironmentPreparedEvent) {
        onApplicationEnvironmentPreparedEvent(
                (ApplicationEnvironmentPreparedEvent) event);
    }
    //在springboot容器的环境设置完成以后
    else if (event instanceof ApplicationPreparedEvent) {
        onApplicationPreparedEvent((ApplicationPreparedEvent) event);
    }
    //容器关闭的时候
    else if (event instanceof ContextClosedEvent && ((ContextClosedEvent) event)
            .getApplicationContext().getParent() == null) {
        onContextClosedEvent();
    }
    //容器启动失败的时候
    else if (event instanceof ApplicationFailedEvent) {
        onApplicationFailedEvent();
    }
}
```

因为我们的事件类型为`ApplicationEvent`，所以会执行`onApplicationStartedEvent((ApplicationStartedEvent) event);`。springBoot会在运行过程中的不同阶段，发送各种事件，来执行对应监听器的对应方法。

对于开发者来说，基本没有什么常见的场景要求我们必须实现一个自定义的SpringApplicationRunListener，即使是SpringBoot中也只默认实现了一个`org.springframework.boot.context.eventEventPublishingRunListener`, 用来在SpringBoot的整个启动流程中的不同时间点发布不同类型的应用事件(SpringApplicationEvent)。那些对这些应用事件感兴趣的ApplicationListener可以接受并处理(这也解释了为什么在SpringApplication实例化的时候加载了一批ApplicationListener，但在run方法执行的过程中并没有被使用)。

如果我们真的在实际场景中自定义实现SpringApplicationRunListener,有一个点需要注意：**任何一个SpringApplicationRunListener实现类的构造方法都需要有两个构造参数，一个参数的类型就是我们的org.springframework.boot.SpringApplication,另外一个参数就是args参数列表的String[]**:

```
public class DemoSpringApplicationRunListener implements SpringApplicationRunListener {
    private final SpringApplication application;
    private final String[] args;
    public DemoSpringApplicationRunListener(SpringApplication sa, String[] args) {
        this.application = sa;
        this.args = args;
    }

    @Override
    public void starting() {
        System.out.println("自定义starting");
    }

    @Override
    public void environmentPrepared(ConfigurableEnvironment environment) {
        System.out.println("自定义environmentPrepared");
    }

    @Override
    public void contextPrepared(ConfigurableApplicationContext context) {
        System.out.println("自定义contextPrepared");
    }

    @Override
    public void contextLoaded(ConfigurableApplicationContext context) {
        System.out.println("自定义contextLoaded");
    }

    @Override
    public void finished(ConfigurableApplicationContext context, Throwable exception) {
        System.out.println("自定义finished");
    }
}
```

接着，我们还要满足SpringFactoriesLoader的约定，在当前SpringBoot项目的classpath下新建META-INF目录，并在该目录下新建spring.fatories文件，文件内容如下:

```properties
org.springframework.boot.SpringApplicationRunListener=\
    com.springbootdemo.DemoSpringApplicationRunListener
```

#### 第二步：环境构建

```
ConfigurableEnvironment environment = prepareEnvironment(listeners,applicationArguments);
```

跟进去该方法：

```
private ConfigurableEnvironment prepareEnvironment(
        SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
    //获取对应的ConfigurableEnvironment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    //配置
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    //发布环境已准备事件，这是第二次发布事件
    listeners.environmentPrepared(environment);
    bindToSpringApplication(environment);
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

来看一下`getOrCreateEnvironment()`方法，前面已经提到，`environment`已经被设置了`servlet`类型，所以这里创建的是环境对象是`StandardServletEnvironment`。

```
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    if (this.webApplicationType == WebApplicationType.SERVLET) {
        return new StandardServletEnvironment();
    }
    return new StandardEnvironment();
}
```

接下来看一下`listeners.environmentPrepared(environment);`，上面已经提到了，这里是第二次发布事件。什么事件呢？来看一下根据事件类型获取到的监听器：

```
0 = {ConfigFileApplicationListener@1747} 
1 = {AnsiOutputApplicationListener@1748} 
2 = {LoggingApplicationListener@1339} 
3 = {ClasspathLoggingApplicationListener@1749} 
4 = {BackgroundPreinitializer@1340} 
5 = {DelegatingApplicationListener@1341} 
6 = {FileEncodingApplicationListener@1750} 
```

主要来看一下**`ConfigFileApplicationListener`**，该监听器非常核心，主要用来处理项目配置。项目中的 properties 和yml文件都是其内部类所加载。具体来看一下：

```
   private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
        List<EnvironmentPostProcessor> postProcessors = this.loadPostProcessors();
        postProcessors.add(this);
        AnnotationAwareOrderComparator.sort(postProcessors);
        Iterator var3 = postProcessors.iterator();

        while(var3.hasNext()) {
            EnvironmentPostProcessor postProcessor = (EnvironmentPostProcessor)var3.next();
            postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
        }

    }
```

EnvironmentPostProcessor级别元素为

```
0 = {SystemEnvironmentPropertySourceEnvironmentPostProcessor@1614} 
1 = {SpringApplicationJsonEnvironmentPostProcessor@1615} 
2 = {CloudFoundryVcapEnvironmentPostProcessor@1616} 
3 = {ConfigFileApplicationListener@1589} 
4 = {SpringBootTestRandomPortEnvironmentPostProcessor@1617} 
5 = {DebugAgentEnvironmentPostProcessor@1618} 
```

首先还是会去读`spring.factories` 文件，`List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();`获取的处理类有以下四种：

```
# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=
org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor，
org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor，
org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor
```

在执行完上述三个监听器流程后，`ConfigFileApplicationListener`会执行该类本身的逻辑。由其内部类`Loader`加载项目制定路径下的配置文件：

```
private static final String DEFAULT_SEARCH_LOCATIONS = "classpath:/,classpath:/config/,file:./,file:./config/";
```

至此，项目的变量配置已全部加载完毕，来一起看一下：

SpringApplication类中

```
    private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments) {
        ConfigurableEnvironment environment = this.getOrCreateEnvironment();
        this.configureEnvironment((ConfigurableEnvironment)environment, applicationArguments.getSourceArgs());
        ConfigurationPropertySources.attach((Environment)environment);
        listeners.environmentPrepared((ConfigurableEnvironment)environment);
        this.bindToSpringApplication((ConfigurableEnvironment)environment);
        if (!this.isCustomEnvironment) {
            environment = (new EnvironmentConverter(this.getClassLoader())).convertEnvironmentIfNecessary((ConfigurableEnvironment)environment, this.deduceEnvironmentClass());
        }

        ConfigurationPropertySources.attach((Environment)environment);
        return (ConfigurableEnvironment)environment;
    }
```

this.bindToSpringApplication((ConfigurableEnvironment)environment);中的propertySourceList

```
propertySourceList = {CopyOnWriteArrayList@2668}  size = 7
 0 = {ConfigurationPropertySourcesPropertySource@2670} "ConfigurationPropertySourcesPropertySource {name='configurationProperties'}"
 1 = {PropertySource$StubPropertySource@2671} "StubPropertySource {name='servletConfigInitParams'}"
 2 = {PropertySource$StubPropertySource@2672} "StubPropertySource {name='servletContextInitParams'}"
 3 = {PropertiesPropertySource@2673} "PropertiesPropertySource {name='systemProperties'}"
 4 = {SystemEnvironmentPropertySourceEnvironmentPostProcessor$OriginAwareSystemEnvironmentPropertySource@2674} "OriginAwareSystemEnvironmentPropertySource {name='systemEnvironment'}"
 5 = {RandomValuePropertySource@2675} "RandomValuePropertySource {name='random'}"
 6 = {OriginTrackedMapPropertySource@2676} "OriginTrackedMapPropertySource {name='applicationConfig: [classpath:/application.properties]'}"
```

这里一共7个配置文件，取值顺序由上到下。也就是说前面的配置变量会覆盖后面同名的配置变量。项目配置变量的时候需要注意这点。



#### 第三步：创建容器

```
context = createApplicationContext();
```

继续跟进该方法：

```
public static final String DEFAULT_WEB_CONTEXT_CLASS = "org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext";
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            switch (this.webApplicationType) {
            case SERVLET:
                contextClass = Class.forName(DEFAULT_WEB_CONTEXT_CLASS);
                break;
            case REACTIVE:
                contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                break;
            default:
                contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Unable create a default ApplicationContext, "
                            + "please specify an ApplicationContextClass",
                    ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

这里创建容器的类型 还是根据`webApplicationType`进行判断的，该类型为`SERVLET`类型，所以会通过反射装载对应的字节码，也就是**AnnotationConfigServletWebServerApplicationContext**



#### **第四步：Spring容器前置处理**

这一步主要是在容器刷新之前的准备动作。包含一个非常关键的操作：**将启动类注入容器，为后续开启自动化配置奠定基础。**

```
prepareContext(context, environment, listeners, applicationArguments,printedBanner);
```

继续跟进该方法：

```
private void prepareContext(ConfigurableApplicationContext context,
        ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments, Banner printedBanner) {
    //设置容器环境，包括各种变量
    context.setEnvironment(environment);
    //执行容器后置处理
    postProcessApplicationContext(context);
    //执行容器中的ApplicationContextInitializer（包括 spring.factories和自定义的实例）
    applyInitializers(context);
　　//发送容器已经准备好的事件，通知各监听器
    listeners.contextPrepared(context);

    //注册启动参数bean，这里将容器指定的参数封装成bean，注入容器
    context.getBeanFactory().registerSingleton("springApplicationArguments",
            applicationArguments);
    //设置banner
    if (printedBanner != null) {
        context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
    }
    //获取我们的启动类指定的参数，可以是多个
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    //加载我们的启动类，将启动类注入容器
    load(context, sources.toArray(new Object[0]));
    //发布容器已加载事件。
    listeners.contextLoaded(context);
}
```

**调用初始化器**

```
protected void applyInitializers(ConfigurableApplicationContext context) {
    // 1. 从SpringApplication类中的initializers集合获取所有的ApplicationContextInitializer
    for (ApplicationContextInitializer initializer : getInitializers()) {
        // 2. 循环调用ApplicationContextInitializer中的initialize方法
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
                initializer.getClass(), ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
    }
}
```

这里终于用到了在创建SpringApplication实例时设置的初始化器了，依次对它们进行遍历，并调用initialize方法。我们也可以自定义初始化器，并实现**initialize**方法，然后放入META-INF/spring.factories配置文件中Key为：org.springframework.context.ApplicationContextInitializer的value中，这里我们自定义的初始化器就会被调用，是我们项目初始化的一种方式

**加载启动指定类（重点）**

大家先回到文章最开始看看，在创建**SpringApplication**实例时，先将**HelloWorldMainApplication.class**存储在this.primarySources属性中，现在就是用到这个属性的时候了，我们来看看**getAllSources（）**

```
public Set<Object> getAllSources() {
    Set<Object> allSources = new LinkedHashSet();
    if (!CollectionUtils.isEmpty(this.primarySources)) {
        //获取primarySources属性，也就是之前存储的HelloWorldMainApplication.class
        allSources.addAll(this.primarySources);
    }

    if (!CollectionUtils.isEmpty(this.sources)) {
        allSources.addAll(this.sources);
    }

    return Collections.unmodifiableSet(allSources);
}
```

很明显，获取了this.primarySources属性，也就是我们的启动类**HelloWorldMainApplication.class，**我们接着看load(context, sources.toArray(new Object[0]));

```
protected void load(ApplicationContext context, Object[] sources) {
    BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
    if (this.beanNameGenerator != null) {
        loader.setBeanNameGenerator(this.beanNameGenerator);
    }
    if (this.resourceLoader != null) {
        loader.setResourceLoader(this.resourceLoader);
    }
    if (this.environment != null) {
        loader.setEnvironment(this.environment);
    }
    loader.load();
}

private int load(Class<?> source) {
    if (isGroovyPresent()
            && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
        // Any GroovyLoaders added in beans{} DSL can contribute beans here
        GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source,
                GroovyBeanDefinitionSource.class);
        load(loader);
    }
    if (isComponent(source)) {
        //以注解的方式，将启动类bean信息存入beanDefinitionMap，也就是将HelloWorldMainApplication.class存入了beanDefinitionMap
        this.annotatedReader.register(source);
        return 1;
    }
    return 0;
}
```

启动类HelloWorldApplication.class被加载到 beanDefinitionMap中，后续该启动类将作为开启自动化配置的入口。

**通知监听器，容器已准备就绪**

```
listeners.contextLoaded(context);
```

主还是针对一些日志等监听器的响应处理。

#### 第五步：刷新容器

执行到这里，springBoot相关的处理工作已经结束，接下的工作就交给了spring。我们来看看**refreshContext(context);**

```
protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    //调用创建的容器applicationContext中的refresh()方法
    ((AbstractApplicationContext)applicationContext).refresh();
}
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        /**
         * 刷新上下文环境
         */
        prepareRefresh();

        /**
         * 初始化BeanFactory，解析XML，相当于之前的XmlBeanFactory的操作，
         */
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        /**
         * 为上下文准备BeanFactory，即对BeanFactory的各种功能进行填充，如常用的注解@Autowired @Qualifier等
         * 添加ApplicationContextAwareProcessor处理器
         * 在依赖注入忽略实现*Aware的接口，如EnvironmentAware、ApplicationEventPublisherAware等
         * 注册依赖，如一个bean的属性中含有ApplicationEventPublisher(beanFactory)，则会将beanFactory的实例注入进去
         */
        prepareBeanFactory(beanFactory);

        try {
            /**
             * 提供子类覆盖的额外处理，即子类处理自定义的BeanFactoryPostProcess
             */
            postProcessBeanFactory(beanFactory);

            /**
             * 激活各种BeanFactory处理器,包括BeanDefinitionRegistryBeanFactoryPostProcessor和普通的BeanFactoryPostProcessor
             * 执行对应的postProcessBeanDefinitionRegistry方法 和  postProcessBeanFactory方法
             */
            invokeBeanFactoryPostProcessors(beanFactory);

            /**
             * 注册拦截Bean创建的Bean处理器，即注册BeanPostProcessor，不是BeanFactoryPostProcessor，注意两者的区别
             * 注意，这里仅仅是注册，并不会执行对应的方法，将在bean的实例化时执行对应的方法
             */
            registerBeanPostProcessors(beanFactory);

            /**
             * 初始化上下文中的资源文件，如国际化文件的处理等
             */
            initMessageSource();

            /**
             * 初始化上下文事件广播器，并放入applicatioEventMulticaster,如ApplicationEventPublisher
             */
            initApplicationEventMulticaster();

            /**
             * 给子类扩展初始化其他Bean
             */
            onRefresh();

            /**
             * 在所有bean中查找listener bean，然后注册到广播器中
             */
            registerListeners();

            /**
             * 设置转换器
             * 注册一个默认的属性值解析器
             * 冻结所有的bean定义，说明注册的bean定义将不能被修改或进一步的处理
             * 初始化剩余的非惰性的bean，即初始化非延迟加载的bean
             */
            finishBeanFactoryInitialization(beanFactory);

            /**
             * 通过spring的事件发布机制发布ContextRefreshedEvent事件，以保证对应的监听器做进一步的处理
             * 即对那种在spring启动后需要处理的一些类，这些类实现了ApplicationListener<ContextRefreshedEvent>，
             * 这里就是要触发这些类的执行(执行onApplicationEvent方法)
             * 另外，spring的内置Event有ContextClosedEvent、ContextRefreshedEvent、ContextStartedEvent、ContextStoppedEvent、RequestHandleEvent
             * 完成初始化，通知生命周期处理器lifeCycleProcessor刷新过程，同时发出ContextRefreshEvent通知其他人
             */
            finishRefresh();
        }

        finally {
    
            resetCommonCaches();
        }
    }
}
```

`refresh`方法在spring整个源码体系中举足轻重，是实现 ioc 和 aop的关键。我之前也有文章分析过这个过程，大家可以去看看



#### 第六步：Spring容器后置处理

```
protected void afterRefresh(ConfigurableApplicationContext context,
        ApplicationArguments args) {
}
```

扩展接口，设计模式中的模板方法，默认为空实现。如果有自定义需求，可以重写该方法。比如打印一些启动结束log，或者一些其它后置处理。



#### 第七步：发出结束执行的事件

```
public void started(ConfigurableApplicationContext context) {
    //这里就是获取的EventPublishingRunListener
    Iterator var2 = this.listeners.iterator();

    while(var2.hasNext()) {
        SpringApplicationRunListener listener = (SpringApplicationRunListener)var2.next();
        //执行EventPublishingRunListener的started方法
        listener.started(context);
    }
}

public void started(ConfigurableApplicationContext context) {
    //创建ApplicationStartedEvent事件，并且发布事件
    //我们看到是执行的ConfigurableApplicationContext这个容器的publishEvent方法，和前面的starting是不同的
    context.publishEvent(new ApplicationStartedEvent(this.application, this.args, context));
}
```

获取EventPublishingRunListener监听器，并执行其started方法，并且将创建的Spring容器传进去了，创建一个ApplicationStartedEvent事件，并执行ConfigurableApplicationContext 的publishEvent方法，也就是说这里是在Spring容器中发布事件，并不是在SpringApplication中发布事件，和前面的starting是不同的，前面的starting是直接向SpringApplication中的11个监听器发布启动事件。



#### 第八步：执行Runners
我们再来看看最后一步**callRunners**(context, applicationArguments);

```
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<Object>();
    //获取容器中所有的ApplicationRunner的Bean实例
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    //获取容器中所有的CommandLineRunner的Bean实例
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    for (Object runner : new LinkedHashSet<Object>(runners)) {
        if (runner instanceof ApplicationRunner) {
            //执行ApplicationRunner的run方法
            callRunner((ApplicationRunner) runner, args);
        }
        if (runner instanceof CommandLineRunner) {
            //执行CommandLineRunner的run方法
            callRunner((CommandLineRunner) runner, args);
        }
    }
}
```
我们也可以自定义一些ApplicationRunner或者CommandLineRunner，实现其run方法，并注入到Spring容器中,在SpringBoot启动完成后，会执行所有的runner的run方法

## 启动时初始化数据

在我们用 springboot 搭建项目的时候，有时候会碰到在项目启动时初始化一些操作的需求 ，针对这种需求 spring boot为我们提供了以下几种方案供我们选择:

- `ApplicationRunner `与 `CommandLineRunner `接口
- `Spring容器初始化时InitializingBean接口和@PostConstruct`
- Spring的事件机制

### ApplicationRunner与CommandLineRunner

我们可以实现 `ApplicationRunner `或 `CommandLineRunner `接口， 这两个接口工作方式相同，都只提供单一的run方法，该方法在SpringApplication.run(…)完成之前调用，不知道大家还对我上一篇文章结尾有没有印象，我们先来看看这两个接口

```
public interface ApplicationRunner {
    void run(ApplicationArguments var1) throws Exception;
}

public interface CommandLineRunner {
    void run(String... var1) throws Exception;
}
```

都只提供单一的run方法，接下来我们来看看具体的使用

#### ApplicationRunner

构造一个类实现ApplicationRunner接口

```
//需要加入到Spring容器中
@Component
public class ApplicationRunnerTest implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("ApplicationRunner");
    }
}
```

很简单，首先要使用**@Component**将实现类加入到Spring容器中，为什么要这样做我们待会再看，然后实现其run方法实现自己的初始化数据逻辑就可以了

#### CommandLineRunner

对于这两个接口而言，我们可以通过Order注解或者使用Ordered接口来指定调用顺序， `@Order() `中的值越小，优先级越高

```
//需要加入到Spring容器中
@Component
@Order(1)
public class CommandLineRunnerTest implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
        System.out.println("CommandLineRunner...");
    }
}
```

同样需要加入到Spring容器中，CommandLineRunner的参数是最原始的参数，没有进行任何处理，ApplicationRunner的参数是ApplicationArguments,是对原始参数的进一步封装

CommandLineRunner并不是Spring框架原有的概念，它属于SpringBoot应用特定的回调扩展接口：

```
public interface CommandLineRunner {
    /**
     * Callback used to run the bean.
     * @param args incoming main method arguments
     * @throws Exception on error
     */
    void run(String... args) throws Exception;

}
```

关于这货，我们需要关注的点有两个：

1. 所有CommandLineRunner的执行时间点是在SpringBoot应用的Application完全初始化工作之后(这里我们可以认为是SpringBoot应用启动类main方法执行完成之前的最后一步)。
2. 当前SpringBoot应用的ApplicationContext中的所有CommandLinerRunner都会被加载执行(无论是手动注册还是被自动扫描注册到IoC容器中)。

跟其他几个扩展点接口类型相似，我们建议CommandLineRunner的实现类使用@org.springframework.core.annotation.Order进行标注或者实现`org.springframework.core.Ordered`接口，便于对他们的执行顺序进行排序调整，这是非常有必要的，因为我们不希望不合适的CommandLineRunner实现类阻塞了后面其他CommandLineRunner的执行。**这个接口非常有用和重要，我们需要重点关注。**

#### 源码分析

SpringApplication.run方法的最后一步第八步：执行Runners源码如下：

```
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<Object>();
    //获取容器中所有的ApplicationRunner的Bean实例
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    //获取容器中所有的CommandLineRunner的Bean实例
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    for (Object runner : new LinkedHashSet<Object>(runners)) {
        if (runner instanceof ApplicationRunner) {
            //执行ApplicationRunner的run方法
            callRunner((ApplicationRunner) runner, args);
        }
        if (runner instanceof CommandLineRunner) {
            //执行CommandLineRunner的run方法
            callRunner((CommandLineRunner) runner, args);
        }
    }
}
```

很明显，是直接从Spring容器中获取ApplicationRunner和CommandLineRunner的实例，并调用其run方法，这也就是为什么我要使用@Component将ApplicationRunner和CommandLineRunner接口的实现类加入到Spring容器中了。

## InitializingBean

在spring初始化bean的时候，如果bean实现了 `InitializingBean `接口，在对象的所有属性被初始化后之后才会调用afterPropertiesSet()方法

```
@Component
public class InitialingzingBeanTest implements InitializingBean {

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("InitializingBean..");
    }
}
```

我们可以看出spring初始化bean肯定会在 ApplicationRunner和CommandLineRunner接口调用之前。

## @PostConstruct

```
@Component
public class PostConstructTest {

    @PostConstruct
    public void postConstruct() {
        System.out.println("init...");
    }
}
```

我们可以看到，只用在方法上添加**@PostConstruct注解，**并将类注入到Spring容器中就可以了。我们来看看**@PostConstruct注解的方法是何时执行的**

在Spring初始化bean时，对bean的实例赋值时，populateBean方法下面有一个initializeBean(beanName, exposedObject, mbd)方法，这个就是用来执行用户设定的初始化操作。我们看下方法体：

```
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            // 激活 Aware 方法
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        // 对特殊的 bean 处理：Aware、BeanClassLoaderAware、BeanFactoryAware
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 后处理器
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // 激活用户自定义的 init 方法
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        // 后处理器
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```

我们看到会先执行后处理器然后执行**invokeInitMethods方法**，我们来看下applyBeanPostProcessorsBeforeInitialization

```
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)  
        throws BeansException {  

    Object result = existingBean;  
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {  
        result = beanProcessor.postProcessBeforeInitialization(result, beanName);  
        if (result == null) {  
            return result;  
        }  
    }  
    return result;  
}

public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)  
        throws BeansException {  

    Object result = existingBean;  
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {  
        result = beanProcessor.postProcessAfterInitialization(result, beanName);  
        if (result == null) {  
            return result;  
        }  
    }  
    return result;  
}
```

获取容器中所有的后置处理器，循环调用后置处理器的**postProcessBeforeInitialization方法，这里我们来看一个BeanPostProcessor**

```
public class CommonAnnotationBeanPostProcessor extends InitDestroyAnnotationBeanPostProcessor implements InstantiationAwareBeanPostProcessor, BeanFactoryAware, Serializable {
    public CommonAnnotationBeanPostProcessor() {
        this.setOrder(2147483644);
        //设置初始化参数为PostConstruct.class
        this.setInitAnnotationType(PostConstruct.class);
        this.setDestroyAnnotationType(PreDestroy.class);
        this.ignoreResourceType("javax.xml.ws.WebServiceContext");
    }
    //略...
}
```

在构造器中设置了一个属性为**PostConstruct.class，**再次观察CommonAnnotationBeanPostProcessor这个类，它继承自InitDestroyAnnotationBeanPostProcessor。InitDestroyAnnotationBeanPostProcessor顾名思义，就是在Bean初始化和销毁的时候所作的一个前置/后置处理器。查看InitDestroyAnnotationBeanPostProcessor类下的postProcessBeforeInitialization方法：

```
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {  
   LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());  
   try {  
       metadata.invokeInitMethods(bean, beanName);  
   }  
   catch (InvocationTargetException ex) {  
       throw new BeanCreationException(beanName, "Invocation of init method failed", ex.getTargetException());  
   }  
   catch (Throwable ex) {  
       throw new BeanCreationException(beanName, "Couldn't invoke init method", ex);  
   }  
    return bean;  
}  

private LifecycleMetadata buildLifecycleMetadata(final Class clazz) {  
       final LifecycleMetadata newMetadata = new LifecycleMetadata();  
       final boolean debug = logger.isDebugEnabled();  
       ReflectionUtils.doWithMethods(clazz, new ReflectionUtils.MethodCallback() {  
           public void doWith(Method method) {  
              if (initAnnotationType != null) {  
                   //判断clazz中的methon是否有initAnnotationType注解，也就是PostConstruct.class注解
                  if (method.getAnnotation(initAnnotationType) != null) {  
                     //如果有就将方法添加进LifecycleMetadata中
                     newMetadata.addInitMethod(method);  
                     if (debug) {  
                         logger.debug("Found init method on class [" + clazz.getName() + "]: " + method);  
                     }  
                  }  
              }  
              if (destroyAnnotationType != null) {  
                    //判断clazz中的methon是否有destroyAnnotationType注解
                  if (method.getAnnotation(destroyAnnotationType) != null) {  
                     newMetadata.addDestroyMethod(method);  
                     if (debug) {  
                         logger.debug("Found destroy method on class [" + clazz.getName() + "]: " + method);  
                     }  
                  }  
              }  
           }  
       });  
       return newMetadata;  
} 
```

在这里会去判断某方法是否有**PostConstruct.class注解**，如果有，则添加到init/destroy队列中，后续一一执行。**@PostConstruct注解的方法会在此时执行，我们接着来看invokeInitMethods**

```
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
        throws Throwable {

    // 是否实现 InitializingBean
    // 如果实现了 InitializingBean 接口，则只掉调用bean的 afterPropertiesSet()
    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (logger.isDebugEnabled()) {
            logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
        }
        if (System.getSecurityManager() != null) {
            try {
                AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
                    ((InitializingBean) bean).afterPropertiesSet();
                    return null;
                }, getAccessControlContext());
            }
            catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        }
        else {
            // 直接调用 afterPropertiesSet()
            ((InitializingBean) bean).afterPropertiesSet();
        }
    }

    if (mbd != null && bean.getClass() != NullBean.class) {
        // 判断是否指定了 init-method()，
        // 如果指定了 init-method()，则再调用制定的init-method
        String initMethodName = mbd.getInitMethodName();
        if (StringUtils.hasLength(initMethodName) &&
                !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
            // 利用反射机制执行
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```

首先检测当前 bean 是否实现了 InitializingBean 接口，如果实现了则调用其 `afterPropertiesSet()`，然后再检查是否也指定了 `init-method()`，如果指定了则通过反射机制调用指定的 `init-method()`。

我们也可以发现**@PostConstruct会在实现 InitializingBean 接口的afterPropertiesSet()方法之前执行**