---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---

## 内置Servlet容器源码分析（Tomcat）

Spring Boot默认使用Tomcat作为嵌入式的Servlet容器，只要引入了spring-boot-start-web依赖，则默认是用Tomcat作为Servlet容器：

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

## Servlet容器的使用

### 默认servlet容器

我们看看spring-boot-starter-web这个starter中有什么

```
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
      <version>2.2.0.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-json</artifactId>
      <version>2.2.0.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
      <version>2.2.0.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
      <version>2.2.0.RELEASE</version>
      <scope>compile</scope>
      <exclusions>
        <exclusion>
          <artifactId>tomcat-embed-el</artifactId>
          <groupId>org.apache.tomcat.embed</groupId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-web</artifactId>
      <version>5.2.0.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-webmvc</artifactId>
      <version>5.2.0.RELEASE</version>
      <scope>compile</scope>
    </dependency>
  </dependencies>
```

核心就是引入了tomcat和SpringMvc，我们先来看tomcat

Spring Boot默认支持Tomcat，Jetty，和Undertow作为底层容器。

org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory

而Spring Boot默认使用Tomcat，一旦引入spring-boot-starter-web模块，就默认使用Tomcat容器。

### 切换servlet容器

那如果我么想切换其他Servlet容器呢，只需如下两步：

- 将tomcat依赖移除掉
- 引入其他Servlet容器依赖

引入jetty：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <!--移除spring-boot-starter-web中的tomcat-->
            <artifactId>spring-boot-starter-tomcat</artifactId>
            <groupId>org.springframework.boot</groupId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <!--引入jetty-->
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

### Servlet容器自动配置原理

#### EmbeddedServletContainerAutoConfiguration

其中**EmbeddedServletContainerAutoConfiguration**是嵌入式Servlet容器的自动配置类，该类在**spring-boot-autoconfigure.jar中的web模块**可以找到。

```
org.springframework.boot.autoconfigure.web.embedded.EmbeddedWebServerFactoryCustomizerAutoConfiguration,
```

我们可以看到**EmbeddedServletContainerAutoConfiguration被配置在spring.factories中，**SpringBoot自动配置将EmbeddedServletContainerAutoConfiguration配置类加入到IOC容器中，接着我们来具体看看这个配置类：

```
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Configuration
@ConditionalOnWebApplication// 在Web环境下才会起作用
@Import(BeanPostProcessorsRegistrar.class)// 会Import一个内部类BeanPostProcessorsRegistrar
public class EmbeddedServletContainerAutoConfiguration {

    @Configuration
    // Tomcat类和Servlet类必须在classloader中存在
    // 文章开头我们已经导入了web的starter，其中包含tomcat和SpringMvc
    // 那么classPath下会存在Tomcat.class和Servlet.class
    @ConditionalOnClass({ Servlet.class, Tomcat.class })
    // 当前Spring容器中不存在EmbeddedServletContainerFactory类型的实例
    @ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class, search = SearchStrategy.CURRENT)
    public static class EmbeddedTomcat {

        @Bean
        public TomcatEmbeddedServletContainerFactory tomcatEmbeddedServletContainerFactory() {
            // 上述条件注解成立的话就会构造TomcatEmbeddedServletContainerFactory这个EmbeddedServletContainerFactory
            return new TomcatEmbeddedServletContainerFactory();
        }
    }
    
    @Configuration
    @ConditionalOnClass({ Servlet.class, Server.class, Loader.class,
            WebAppContext.class })
    @ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class, search = SearchStrategy.CURRENT)
    public static class EmbeddedJetty {

        @Bean
        public JettyEmbeddedServletContainerFactory jettyEmbeddedServletContainerFactory() {
            return new JettyEmbeddedServletContainerFactory();
        }

    }
    
    @Configuration
    @ConditionalOnClass({ Servlet.class, Undertow.class, SslClientAuthMode.class })
    @ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class, search = SearchStrategy.CURRENT)
    public static class EmbeddedUndertow {

        @Bean
        public UndertowEmbeddedServletContainerFactory undertowEmbeddedServletContainerFactory() {
            return new UndertowEmbeddedServletContainerFactory();
        }

    }
    
    //other code...
}
```

在这个自动配置类中配置了三个容器工厂的Bean，分别是：

- **TomcatEmbeddedServletContainerFactory**
- **JettyEmbeddedServletContainerFactory**
- **UndertowEmbeddedServletContainerFactory**

这里以大家熟悉的Tomcat为例，首先Spring Boot会判断当前环境中是否引入了Servlet和Tomcat依赖，并且当前容器中没有自定义的**EmbeddedServletContainerFactory**的情况下，则创建Tomcat容器工厂。其他Servlet容器工厂也是同样的道理。

#### EmbeddedServletContainerFactory

- 嵌入式Servlet容器工厂

```
public interface EmbeddedServletContainerFactory {

    EmbeddedServletContainer getEmbeddedServletContainer( ServletContextInitializer... initializers);
}
```

内部只有一个方法，用于获取嵌入式的Servlet容器。

该工厂接口主要有三个实现类，分别对应三种嵌入式Servlet容器的工厂类，如图所示：

#### TomcatEmbeddedServletContainerFactory

以Tomcat容器工厂TomcatEmbeddedServletContainerFactory类为例：

```
public class TomcatEmbeddedServletContainerFactory extends AbstractEmbeddedServletContainerFactory implements ResourceLoaderAware {
    
    //other code...
    
    @Override
    public EmbeddedServletContainer getEmbeddedServletContainer( ServletContextInitializer... initializers) {
        //创建一个Tomcat
        Tomcat tomcat = new Tomcat();
        
       //配置Tomcat的基本环节
        File baseDir = (this.baseDirectory != null ? this.baseDirectory: createTempDir("tomcat"));
        tomcat.setBaseDir(baseDir.getAbsolutePath());
        Connector connector = new Connector(this.protocol);
        tomcat.getService().addConnector(connector);
        customizeConnector(connector);
        tomcat.setConnector(connector);
        tomcat.getHost().setAutoDeploy(false);
        configureEngine(tomcat.getEngine());
        for (Connector additionalConnector : this.additionalTomcatConnectors) {
            tomcat.getService().addConnector(additionalConnector);
        }
        prepareContext(tomcat.getHost(), initializers);
        
        //包装tomcat对象，返回一个嵌入式Tomcat容器，内部会启动该tomcat容器
        return getTomcatEmbeddedServletContainer(tomcat);
    }
}
```

首先会创建一个Tomcat的对象，并设置一些属性配置，最后调用**getTomcatEmbeddedServletContainer(tomcat)方法，内部会启动tomcat，**我们来看看：

```
protected TomcatEmbeddedServletContainer getTomcatEmbeddedServletContainer(
    Tomcat tomcat) {
    return new TomcatEmbeddedServletContainer(tomcat, getPort() >= 0);
}
```

该函数很简单，就是来创建Tomcat容器并返回。看看TomcatEmbeddedServletContainer类：

```
public class TomcatEmbeddedServletContainer implements EmbeddedServletContainer {

    public TomcatEmbeddedServletContainer(Tomcat tomcat, boolean autoStart) {
        Assert.notNull(tomcat, "Tomcat Server must not be null");
        this.tomcat = tomcat;
        this.autoStart = autoStart;
        
        //初始化嵌入式Tomcat容器，并启动Tomcat
        initialize();
    }
    
    private void initialize() throws EmbeddedServletContainerException {
        TomcatEmbeddedServletContainer.logger
                .info("Tomcat initialized with port(s): " + getPortsDescription(false));
        synchronized (this.monitor) {
            try {
                addInstanceIdToEngineName();
                try {
                    final Context context = findContext();
                    context.addLifecycleListener(new LifecycleListener() {

                        @Override
                        public void lifecycleEvent(LifecycleEvent event) {
                            if (context.equals(event.getSource())
                                    && Lifecycle.START_EVENT.equals(event.getType())) {
                                // Remove service connectors so that protocol
                                // binding doesn't happen when the service is
                                // started.
                                removeServiceConnectors();
                            }
                        }

                    });

                    // Start the server to trigger initialization listeners
                    //启动tomcat
                    this.tomcat.start();

                    // We can re-throw failure exception directly in the main thread
                    rethrowDeferredStartupExceptions();

                    try {
                        ContextBindings.bindClassLoader(context, getNamingToken(context),
                                getClass().getClassLoader());
                    }
                    catch (NamingException ex) {
                        // Naming is not enabled. Continue
                    }

                    // Unlike Jetty, all Tomcat threads are daemon threads. We create a
                    // blocking non-daemon to stop immediate shutdown
                    startDaemonAwaitThread();
                }
                catch (Exception ex) {
                    containerCounter.decrementAndGet();
                    throw ex;
                }
            }
            catch (Exception ex) {
                stopSilently();
                throw new EmbeddedServletContainerException(
                        "Unable to start embedded Tomcat", ex);
            }
        }
    }
}
```

到这里就启动了嵌入式的Servlet容器，其他容器类似。

### Servlet容器启动原理

#### SpringBoot启动过程

我们回顾一下前面讲解的SpringBoot启动过程，也就是run方法：

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

我们回顾一下**第三步：创建Spring容器**

```
public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
            + "annotation.AnnotationConfigApplicationContext";

public static final String DEFAULT_WEB_CONTEXT_CLASS = "org.springframework."
            + "boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext";

protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            //根据应用环境，创建不同的IOC容器
            contextClass = Class.forName(this.webEnvironment
                                         ? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
}
```

创建IOC容器，如果是web应用，则创建AnnotationConfigEmbeddedWebApplicationContext的IOC容器；如果不是，则创建AnnotationConfigApplicationContext的IOC容器；很明显我们创建的容器是AnnotationConfigEmbeddedWebApplicationContext，接着我们来看看*第五步，刷新容器refreshContext(context);

```
private void refreshContext(ConfigurableApplicationContext context) {
    refresh(context);
}

protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    //调用容器的refresh()方法刷新容器
    ((AbstractApplicationContext) applicationContext).refresh();
}
```

#### 容器刷新过程

调用抽象父类AbstractApplicationContext的**refresh**()方法；

**`AbstractApplicationContext`**

```
 1 public void refresh() throws BeansException, IllegalStateException {
 2     synchronized (this.startupShutdownMonitor) {
 3         /**
 4          * 刷新上下文环境
 5          */
 6         prepareRefresh();
 7 
 8         /**
 9          * 初始化BeanFactory，解析XML，相当于之前的XmlBeanFactory的操作，
10          */
11         ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
12 
13         /**
14          * 为上下文准备BeanFactory，即对BeanFactory的各种功能进行填充，如常用的注解@Autowired @Qualifier等
15          * 添加ApplicationContextAwareProcessor处理器
16          * 在依赖注入忽略实现*Aware的接口，如EnvironmentAware、ApplicationEventPublisherAware等
17          * 注册依赖，如一个bean的属性中含有ApplicationEventPublisher(beanFactory)，则会将beanFactory的实例注入进去
18          */
19         prepareBeanFactory(beanFactory);
20 
21         try {
22             /**
23              * 提供子类覆盖的额外处理，即子类处理自定义的BeanFactoryPostProcess
24              */
25             postProcessBeanFactory(beanFactory);
26 
27             /**
28              * 激活各种BeanFactory处理器,包括BeanDefinitionRegistryBeanFactoryPostProcessor和普通的BeanFactoryPostProcessor
29              * 执行对应的postProcessBeanDefinitionRegistry方法 和  postProcessBeanFactory方法
30              */
31             invokeBeanFactoryPostProcessors(beanFactory);
32 
33             /**
34              * 注册拦截Bean创建的Bean处理器，即注册BeanPostProcessor，不是BeanFactoryPostProcessor，注意两者的区别
35              * 注意，这里仅仅是注册，并不会执行对应的方法，将在bean的实例化时执行对应的方法
36              */
37             registerBeanPostProcessors(beanFactory);
38 
39             /**
40              * 初始化上下文中的资源文件，如国际化文件的处理等
41              */
42             initMessageSource();
43 
44             /**
45              * 初始化上下文事件广播器，并放入applicatioEventMulticaster,如ApplicationEventPublisher
46              */
47             initApplicationEventMulticaster();
48 
49             /**
50              * 给子类扩展初始化其他Bean
51              */
52             onRefresh();
53 
54             /**
55              * 在所有bean中查找listener bean，然后注册到广播器中
56              */
57             registerListeners();
58 
59             /**
60              * 设置转换器
61              * 注册一个默认的属性值解析器
62              * 冻结所有的bean定义，说明注册的bean定义将不能被修改或进一步的处理
63              * 初始化剩余的非惰性的bean，即初始化非延迟加载的bean
64              */
65             finishBeanFactoryInitialization(beanFactory);
66 
67             /**
68              * 通过spring的事件发布机制发布ContextRefreshedEvent事件，以保证对应的监听器做进一步的处理
69              * 即对那种在spring启动后需要处理的一些类，这些类实现了ApplicationListener<ContextRefreshedEvent>，
70              * 这里就是要触发这些类的执行(执行onApplicationEvent方法)
71              * spring的内置Event有ContextClosedEvent、ContextRefreshedEvent、ContextStartedEvent、ContextStoppedEvent、RequestHandleEvent
72              * 完成初始化，通知生命周期处理器lifeCycleProcessor刷新过程，同时发出ContextRefreshEvent通知其他人
73              */
74             finishRefresh();
75         }
76 
77         finally {
78     
79             resetCommonCaches();
80         }
81     }
82 }
```

我们看第52行的方法：

```
protected void onRefresh() throws BeansException {

}
```

很明显抽象父类AbstractApplicationContext中的onRefresh是一个空方法，并且使用protected修饰，也就是其子类可以重写onRefresh方法，那我们看看其子类AnnotationConfigEmbeddedWebApplicationContext中的onRefresh方法是如何重写的，AnnotationConfigEmbeddedWebApplicationContext又继承EmbeddedWebApplicationContext，如下：

```
public class AnnotationConfigEmbeddedWebApplicationContext extends EmbeddedWebApplicationContext {
```

那我们看看其父类EmbeddedWebApplicationContext 是如何重写onRefresh方法的：

**EmbeddedWebApplicationContext**

```
@Override
protected void onRefresh() {
    super.onRefresh();
    try {
        //核心方法：会获取嵌入式的Servlet容器工厂，并通过工厂来获取Servlet容器
        createEmbeddedServletContainer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start embedded container", ex);
    }
}
```

在createEmbeddedServletContainer方法中会获取嵌入式的Servlet容器工厂，并通过工厂来获取Servlet容器：

```
 1 private void createEmbeddedServletContainer() {
 2     EmbeddedServletContainer localContainer = this.embeddedServletContainer;
 3     ServletContext localServletContext = getServletContext();
 4     if (localContainer == null && localServletContext == null) {
 5         //先获取嵌入式Servlet容器工厂
 6         EmbeddedServletContainerFactory containerFactory = getEmbeddedServletContainerFactory();
 7         //根据容器工厂来获取对应的嵌入式Servlet容器
 8         this.embeddedServletContainer = containerFactory.getEmbeddedServletContainer(getSelfInitializer());
 9     }
10     else if (localServletContext != null) {
11         try {
12             getSelfInitializer().onStartup(localServletContext);
13         }
14         catch (ServletException ex) {
15             throw new ApplicationContextException("Cannot initialize servlet context",ex);
16         }
17     }
18     initPropertySources();
19 }
```

关键代码在第6和第8行，**先获取Servlet容器工厂，然后****根据容器工厂来获取对应的嵌入式Servlet容器**

#### 获取Servlet容器工厂

```
protected EmbeddedServletContainerFactory getEmbeddedServletContainerFactory() {
    //从Spring的IOC容器中获取EmbeddedServletContainerFactory.class类型的Bean
    String[] beanNames = getBeanFactory().getBeanNamesForType(EmbeddedServletContainerFactory.class);
    //调用getBean实例化EmbeddedServletContainerFactory.class
    return getBeanFactory().getBean(beanNames[0], EmbeddedServletContainerFactory.class);
}
```

我们看到先从Spring的IOC容器中获取EmbeddedServletContainerFactory.class类型的Bean，然后调用getBean实例化EmbeddedServletContainerFactory.class，大家还记得我们第一节Servlet容器自动配置类EmbeddedServletContainerAutoConfiguration中注入Spring容器的对象是什么吗？当我们引入spring-boot-starter-web这个启动器后，会注入TomcatEmbeddedServletContainerFactory这个对象到Spring容器中，所以这里获取到的**Servlet容器工厂是TomcatEmbeddedServletContainerFactory，然后调用

TomcatEmbeddedServletContainerFactory的getEmbeddedServletContainer方法获取Servlet容器，并且启动Tomcat，大家可以看看文章开头的getEmbeddedServletContainer方法。

大家看一下第8行代码获取Servlet容器方法的参数getSelfInitializer()，这是个啥？我们点进去看看

```
private ServletContextInitializer getSelfInitializer() {
    //创建一个ServletContextInitializer对象，并重写onStartup方法，很明显是一个回调方法
    return new ServletContextInitializer() {
        public void onStartup(ServletContext servletContext) throws ServletException {
            EmbeddedWebApplicationContext.this.selfInitialize(servletContext);
        }
    };
}
```

创建一个ServletContextInitializer对象，并重写onStartup方法，很明显是一个回调方法，这里给大家留一点疑问：

- **ServletContextInitializer对象创建过程是怎样的？**
- **onStartup是何时调用的？**
- **onStartup方法的作用是什么？**

**`ServletContextInitializer`是 Servlet 容器初始化的时候，提供的初始化接口。**

## SpringBoot如何实现SpringMvc的？

### 自定义Servlet、Filter、Listener

Spring容器中声明ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean

```
@Bean
public ServletRegistrationBean customServlet() {
    return new ServletRegistrationBean(new CustomServlet(), "/custom");
}

private static class CustomServlet extends HttpServlet {
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().write("receive by custom servlet");
    }
}
```

先自定义一个**Servlet，重写**service实现自己的业务逻辑，然后通过@Bean注解往Spring容器中注入一个**ServletRegistrationBean类型的bean实例，并且实例化一个自定义的Servlet作为参数，这样就将自定义的Servlet加入Tomcat中了。**

### @ServletComponentScan注解和@WebServlet、@WebFilter以及@WebListener注解配合使用

@ServletComponentScan注解启用ImportServletComponentScanRegistrar类，是个ImportBeanDefinitionRegistrar接口的实现类，会被Spring容器所解析。ServletComponentScanRegistrar内部会解析@ServletComponentScan注解，然后会在Spring容器中注册ServletComponentRegisteringPostProcessor，是个BeanFactoryPostProcessor，会去解析扫描出来的类是不是有@WebServlet、@WebListener、@WebFilter这3种注解，有的话把这3种类型的类转换成ServletRegistrationBean、FilterRegistrationBean或者ServletListenerRegistrationBean，然后让Spring容器去解析：

```
@SpringBootApplication
@ServletComponentScan
public class EmbeddedServletApplication {
 ... 
}

@WebServlet(urlPatterns = "/simple")
public class SimpleServlet extends HttpServlet {

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().write("receive by SimpleServlet");
    }
}
```

### 在Spring容器中声明Servlet、Filter或者Listener

```
@Bean(name = "dispatcherServlet")
public DispatcherServlet myDispatcherServlet() {
    return new DispatcherServlet();
}
```

我们发现往Tomcat中添加Servlet、Filter或者Listener还是挺容易的，大家还记得以前SpringMVC是怎么配置**DispatcherServlet**的吗？在web.xml中：

```
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring-mvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

和我们SpringBoot中配置Servlet相比是不是复杂很多，虽然SpringBoot中自定义Servlet很简单，但是其底层却不简单，下面我们来分析一下其原理

## ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean

我们来看看这几个特殊的类:

### **ServletRegistrationBean**

```
public class ServletRegistrationBean extends RegistrationBean {
    //存放目标Servlet实例
    private Servlet servlet;
    //存放Servlet的urlMapping
    private Set<String> urlMappings;
    private boolean alwaysMapUrl;
    private int loadOnStartup;
    private MultipartConfigElement multipartConfig;


    public ServletRegistrationBean(Servlet servlet, String... urlMappings) {
        this(servlet, true, urlMappings);
    }

    public ServletRegistrationBean(Servlet servlet, boolean alwaysMapUrl, String... urlMappings) {
        this.urlMappings = new LinkedHashSet();
        this.alwaysMapUrl = true;
        this.loadOnStartup = -1;
        Assert.notNull(servlet, "Servlet must not be null");
        Assert.notNull(urlMappings, "UrlMappings must not be null");
        this.servlet = servlet;
        this.alwaysMapUrl = alwaysMapUrl;
        this.urlMappings.addAll(Arrays.asList(urlMappings));
    }
    
    public void onStartup(ServletContext servletContext) throws ServletException {
        Assert.notNull(this.servlet, "Servlet must not be null");
        String name = this.getServletName();
        if (!this.isEnabled()) {
            logger.info("Servlet " + name + " was not registered (disabled)");
        } else {
            logger.info("Mapping servlet: '" + name + "' to " + this.urlMappings);
            Dynamic added = servletContext.addServlet(name, this.servlet);
            if (added == null) {
                logger.info("Servlet " + name + " was not registered (possibly already registered?)");
            } else {
                this.configure(added);
            }
        }
    }
    
    //略
}
```

在我们例子中我们通过return new ServletRegistrationBean(new CustomServlet(), "/custom");就知道，ServletRegistrationBean里会存放目标Servlet实例和urlMapping,并且继承RegistrationBean这个类

### **FilterRegistrationBean**

```
public class FilterRegistrationBean extends AbstractFilterRegistrationBean {
    //存放目标Filter对象
    private Filter filter;

    public FilterRegistrationBean() {
        super(new ServletRegistrationBean[0]);
    }

    public FilterRegistrationBean(Filter filter, ServletRegistrationBean... servletRegistrationBeans) {
        super(servletRegistrationBeans);
        Assert.notNull(filter, "Filter must not be null");
        this.filter = filter;
    }

    public Filter getFilter() {
        return this.filter;
    }

    public void setFilter(Filter filter) {
        Assert.notNull(filter, "Filter must not be null");
        this.filter = filter;
    }
}

abstract class AbstractFilterRegistrationBean extends RegistrationBean {
    private static final EnumSet<DispatcherType> ASYNC_DISPATCHER_TYPES;
    private static final EnumSet<DispatcherType> NON_ASYNC_DISPATCHER_TYPES;
    private static final String[] DEFAULT_URL_MAPPINGS;
    private Set<ServletRegistrationBean> servletRegistrationBeans = new LinkedHashSet();
    private Set<String> servletNames = new LinkedHashSet();
    private Set<String> urlPatterns = new LinkedHashSet();
    //重写onStartup方法
    public void onStartup(ServletContext servletContext) throws ServletException {
        Filter filter = this.getFilter();
        Assert.notNull(filter, "Filter must not be null");
        String name = this.getOrDeduceName(filter);
        if (!this.isEnabled()) {
            this.logger.info("Filter " + name + " was not registered (disabled)");
        } else {
            Dynamic added = servletContext.addFilter(name, filter);
            if (added == null) {
                this.logger.info("Filter " + name + " was not registered (possibly already registered?)");
            } else {
                this.configure(added);
            }
        }
    }
    //略...
}
```

我们看到FilterRegistrationBean 中也保存了**目标Filter对象，并且继承了RegistrationBean

### **ServletListenerRegistrationBean**

```
public class ServletListenerRegistrationBean<T extends EventListener> extends RegistrationBean {
    //存放了目标listener
    private T listener;

    public ServletListenerRegistrationBean() {
    }

    public ServletListenerRegistrationBean(T listener) {
        Assert.notNull(listener, "Listener must not be null");
        Assert.isTrue(isSupportedType(listener), "Listener is not of a supported type");
        this.listener = listener;
    }

    public void setListener(T listener) {
        Assert.notNull(listener, "Listener must not be null");
        Assert.isTrue(isSupportedType(listener), "Listener is not of a supported type");
        this.listener = listener;
    }

    public void onStartup(ServletContext servletContext) throws ServletException {
        if (!this.isEnabled()) {
            logger.info("Listener " + this.listener + " was not registered (disabled)");
        } else {
            try {
                servletContext.addListener(this.listener);
            } catch (RuntimeException var3) {
                throw new IllegalStateException("Failed to add listener '" + this.listener + "' to servlet context", var3);
            }
        }
    }
    //略...
}
```

ServletListenerRegistrationBean也是一样，那我们来看看RegistrationBean这个类

```
public abstract class RegistrationBean implements ServletContextInitializer, Ordered {
    ...
}
public interface ServletContextInitializer {
    void onStartup(ServletContext var1) throws ServletException;
}
```

我们发现**RegistrationBean 实现了****ServletContextInitializer这个接口，并且有一个onStartup方法，**ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean都实现了**onStartup方法。

ServletContextInitializer是 Servlet 容器初始化的时候，提供的初始化接口。所以，Servlet 容器初始化会获取并触发所有的`FilterRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean实例中onStartup方法

那到底是何时触发这些类的onStartup方法呢？

当Tomcat容器启动时，会执行`callInitializers`，然后获取所有的**`ServletContextInitializer，循环执行`**`onStartup`方法触发回调方法。那`FilterRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean实例是何时加入到`Initializers集合的呢？这要回顾一下我们上一篇文章Tomcat的启动过程

## Servlet容器的启动

大家可以看看我上一篇文章，我这里简单的复制一下代码

**EmbeddedWebApplicationContext**

```
 1 @Override
 2 protected void onRefresh() {
 3     super.onRefresh();
 4     try {
 5         //核心方法：会获取嵌入式的Servlet容器工厂，并通过工厂来获取Servlet容器
 6         createEmbeddedServletContainer();
 7     }
 8     catch (Throwable ex) {
 9         throw new ApplicationContextException("Unable to start embedded container", ex);
10     }
11 }
12 
13 private void createEmbeddedServletContainer() {
14     EmbeddedServletContainer localContainer = this.embeddedServletContainer;
15     ServletContext localServletContext = getServletContext();
16     if (localContainer == null && localServletContext == null) {
17         //先获取嵌入式Servlet容器工厂
18         EmbeddedServletContainerFactory containerFactory = getEmbeddedServletContainerFactory();
19         //根据容器工厂来获取对应的嵌入式Servlet容器
20         this.embeddedServletContainer = containerFactory.getEmbeddedServletContainer(getSelfInitializer());
21     }
22     else if (localServletContext != null) {
23         try {
24             getSelfInitializer().onStartup(localServletContext);
25         }
26         catch (ServletException ex) {
27             throw new ApplicationContextException("Cannot initialize servlet context",ex);
28         }
29     }
30     initPropertySources();
31 }
```

关键代码在第20行，先通过getSelfInitializer()获取到所有的Initializer，传入Servlet容器中，那核心就在getSelfInitializer()方法：

```
1 private ServletContextInitializer getSelfInitializer() {
2     //只是创建了一个ServletContextInitializer实例返回
3     //所以Servlet容器启动的时候，会调用这个对象的onStartup方法
4     return new ServletContextInitializer() {
5         public void onStartup(ServletContext servletContext) throws ServletException {
6             EmbeddedWebApplicationContext.this.selfInitialize(servletContext);
7         }
8     };
9 }
```

我们看到只是创建了一个ServletContextInitializer实例返回，所以Servlet容器启动的时候，会调用这个对象的onStartup方法，那我们来分析其onStartup中的逻辑，也就是selfInitialize方法，并将Servlet上下文对象传进去了

**selfInitialize**

```
 1 private void selfInitialize(ServletContext servletContext) throws ServletException {
 2     prepareWebApplicationContext(servletContext);
 3     ConfigurableListableBeanFactory beanFactory = getBeanFactory();
 4     ExistingWebApplicationScopes existingScopes = new ExistingWebApplicationScopes(beanFactory);
 5     WebApplicationContextUtils.registerWebApplicationScopes(beanFactory,getServletContext());
 6     existingScopes.restore();
 7     WebApplicationContextUtils.registerEnvironmentBeans(beanFactory,getServletContext());
 8     //这里便是获取所有的 ServletContextInitializer 实现类，会获取所有的注册组件
 9     for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
10         //执行所有ServletContextInitializer的onStartup方法
11         beans.onStartup(servletContext);
12     }
13 }
```

关键代码在第9和第11行，先获取所有的ServletContextInitializer 实现类，然后遍历执行所有ServletContextInitializer的onStartup方法



### **获取所有的ServletContextInitializer**

我们来看看getServletContextInitializerBeans方法**
**

```
protected Collection<ServletContextInitializer> getServletContextInitializerBeans() {
    return new ServletContextInitializerBeans(getBeanFactory());
}
```

ServletContextInitializerBeans对象是对`ServletContextInitializer`的一种包装：

```
 1 public class ServletContextInitializerBeans extends AbstractCollection<ServletContextInitializer> {
 2     private final MultiValueMap<Class<?>, ServletContextInitializer> initializers = new LinkedMultiValueMap();
 3     //存放所有的ServletContextInitializer
 4     private List<ServletContextInitializer> sortedList;
 5 
 6     public ServletContextInitializerBeans(ListableBeanFactory beanFactory) {
 7         //执行addServletContextInitializerBeans
 8         this.addServletContextInitializerBeans(beanFactory);
 9         //执行addAdaptableBeans
10         this.addAdaptableBeans(beanFactory);
11         List<ServletContextInitializer> sortedInitializers = new ArrayList();
12         Iterator var3 = this.initializers.entrySet().iterator();
13 
14         while(var3.hasNext()) {
15             Entry<?, List<ServletContextInitializer>> entry = (Entry)var3.next();
16             AnnotationAwareOrderComparator.sort((List)entry.getValue());
17             sortedInitializers.addAll((Collection)entry.getValue());
18         }
19         this.sortedList = Collections.unmodifiableList(sortedInitializers);
20     }
21 
22     private void addServletContextInitializerBeans(ListableBeanFactory beanFactory) {
23         Iterator var2 = this.getOrderedBeansOfType(beanFactory, ServletContextInitializer.class).iterator();
24 
25         while(var2.hasNext()) {
26             Entry<String, ServletContextInitializer> initializerBean = (Entry)var2.next();
27             this.addServletContextInitializerBean((String)initializerBean.getKey(), (ServletContextInitializer)initializerBean.getValue(), beanFactory);
28         }
29 
30     }
31 
32     private void addServletContextInitializerBean(String beanName, ServletContextInitializer initializer, ListableBeanFactory beanFactory) {
33         if (initializer instanceof ServletRegistrationBean) {
34             Servlet source = ((ServletRegistrationBean)initializer).getServlet();
35             this.addServletContextInitializerBean(Servlet.class, beanName, initializer, beanFactory, source);
36         } else if (initializer instanceof FilterRegistrationBean) {
37             Filter source = ((FilterRegistrationBean)initializer).getFilter();
38             this.addServletContextInitializerBean(Filter.class, beanName, initializer, beanFactory, source);
39         } else if (initializer instanceof DelegatingFilterProxyRegistrationBean) {
40             String source = ((DelegatingFilterProxyRegistrationBean)initializer).getTargetBeanName();
41             this.addServletContextInitializerBean(Filter.class, beanName, initializer, beanFactory, source);
42         } else if (initializer instanceof ServletListenerRegistrationBean) {
43             EventListener source = ((ServletListenerRegistrationBean)initializer).getListener();
44             this.addServletContextInitializerBean(EventListener.class, beanName, initializer, beanFactory, source);
45         } else {
46             this.addServletContextInitializerBean(ServletContextInitializer.class, beanName, initializer, beanFactory, initializer);
47         }
48 
49     }
50 
51     private void addServletContextInitializerBean(Class<?> type, String beanName, ServletContextInitializer initializer, ListableBeanFactory beanFactory, Object source) {
52         this.initializers.add(type, initializer);
53         if (source != null) {
54             this.seen.add(source);
55         }
56 
57         if (logger.isDebugEnabled()) {
58             String resourceDescription = this.getResourceDescription(beanName, beanFactory);
59             int order = this.getOrder(initializer);
60             logger.debug("Added existing " + type.getSimpleName() + " initializer bean '" + beanName + "'; order=" + order + ", resource=" + resourceDescription);
61         }
62 
63     }
64 
65     private void addAdaptableBeans(ListableBeanFactory beanFactory) {
66         MultipartConfigElement multipartConfig = this.getMultipartConfig(beanFactory);
67         this.addAsRegistrationBean(beanFactory, Servlet.class, new ServletContextInitializerBeans.ServletRegistrationBeanAdapter(multipartConfig));
68         this.addAsRegistrationBean(beanFactory, Filter.class, new ServletContextInitializerBeans.FilterRegistrationBeanAdapter(null));
69         Iterator var3 = ServletListenerRegistrationBean.getSupportedTypes().iterator();
70 
71         while(var3.hasNext()) {
72             Class<?> listenerType = (Class)var3.next();
73             this.addAsRegistrationBean(beanFactory, EventListener.class, listenerType, new ServletContextInitializerBeans.ServletListenerRegistrationBeanAdapter(null));
74         }
75 
76     }
77     
78     public Iterator<ServletContextInitializer> iterator() {
79         //返回所有的ServletContextInitializer
80         return this.sortedList.iterator();
81     }
82 
83     //略...
84 }
```

我们看到ServletContextInitializerBeans 中有一个存放所有ServletContextInitializer的集合sortedList，就是在其构造方法中获取所有的ServletContextInitializer，并放入sortedList集合中，那我们来看看其构造方法的逻辑，看到第8行先调用

addServletContextInitializerBeans方法：　　

```
1 private void addServletContextInitializerBeans(ListableBeanFactory beanFactory) {
2     //从Spring容器中获取所有ServletContextInitializer.class 类型的Bean
3     for (Entry<String, ServletContextInitializer> initializerBean : getOrderedBeansOfType(beanFactory, ServletContextInitializer.class)) {
4         //添加到具体的集合中
5         addServletContextInitializerBean(initializerBean.getKey(),initializerBean.getValue(), beanFactory);
6     }
7 }
```

我们看到先从Spring容器中获取所有**ServletContextInitializer.class** 类型的Bean，这里我们自定义的ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean就被获取到了，然后调用addServletContextInitializerBean方法：

```
 1 private void addServletContextInitializerBean(String beanName, ServletContextInitializer initializer, ListableBeanFactory beanFactory) {
 2     //判断ServletRegistrationBean类型
 3     if (initializer instanceof ServletRegistrationBean) {
 4         Servlet source = ((ServletRegistrationBean)initializer).getServlet();
 5         //将ServletRegistrationBean加入到集合中
 6         this.addServletContextInitializerBean(Servlet.class, beanName, initializer, beanFactory, source);
 7     //判断FilterRegistrationBean类型
 8     } else if (initializer instanceof FilterRegistrationBean) {
 9         Filter source = ((FilterRegistrationBean)initializer).getFilter();
10         //将ServletRegistrationBean加入到集合中
11         this.addServletContextInitializerBean(Filter.class, beanName, initializer, beanFactory, source);
12     } else if (initializer instanceof DelegatingFilterProxyRegistrationBean) {
13         String source = ((DelegatingFilterProxyRegistrationBean)initializer).getTargetBeanName();
14         this.addServletContextInitializerBean(Filter.class, beanName, initializer, beanFactory, source);
15     } else if (initializer instanceof ServletListenerRegistrationBean) {
16         EventListener source = ((ServletListenerRegistrationBean)initializer).getListener();
17         this.addServletContextInitializerBean(EventListener.class, beanName, initializer, beanFactory, source);
18     } else {
19         this.addServletContextInitializerBean(ServletContextInitializer.class, beanName, initializer, beanFactory, initializer);
20     }
21 
22 }
23 
24 private void addServletContextInitializerBean(Class<?> type, String beanName, 
25                             ServletContextInitializer initializer, ListableBeanFactory beanFactory, Object source) {
26     //加入到initializers中
27     this.initializers.add(type, initializer);
28 }
```

很明显，判断从Spring容器中获取的ServletContextInitializer类型，如ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean，并加入到initializers集合中去，我们再来看构造器中的另外一个方法**addAdaptableBeans(beanFactory)：**

```
 1 private void addAdaptableBeans(ListableBeanFactory beanFactory) {
 2     //从beanFactory获取所有Servlet.class和Filter.class类型的Bean，并封装成RegistrationBean对象，加入到集合中
 3     this.addAsRegistrationBean(beanFactory, Servlet.class, new ServletContextInitializerBeans.ServletRegistrationBeanAdapter(multipartConfig));
 4     this.addAsRegistrationBean(beanFactory, Filter.class, new ServletContextInitializerBeans.FilterRegistrationBeanAdapter(null));
 5 }
 6 
 7 private <T, B extends T> void addAsRegistrationBean(ListableBeanFactory beanFactory, Class<T> type, Class<B> beanType, ServletContextInitializerBeans.RegistrationBeanAdapter<T> adapter) {
 8     //从Spring容器中获取所有的Servlet.class和Filter.class类型的Bean
 9     List<Entry<String, B>> beans = this.getOrderedBeansOfType(beanFactory, beanType, this.seen);
10     Iterator var6 = beans.iterator();
11 
12     while(var6.hasNext()) {
13         Entry<String, B> bean = (Entry)var6.next();
14         if (this.seen.add(bean.getValue())) {
15             int order = this.getOrder(bean.getValue());
16             String beanName = (String)bean.getKey();
17             //创建Servlet.class和Filter.class包装成RegistrationBean对象
18             RegistrationBean registration = adapter.createRegistrationBean(beanName, bean.getValue(), beans.size());
19             registration.setName(beanName);
20             registration.setOrder(order);
21             this.initializers.add(type, registration);
22             if (logger.isDebugEnabled()) {
23                 logger.debug("Created " + type.getSimpleName() + " initializer for bean '" + beanName + "'; order=" + order + ", resource=" + this.getResourceDescription(beanName, beanFactory));
24             }
25         }
26     }
27 
28 }
```

我们看到先从beanFactory获取所有Servlet.class和Filter.class类型的Bean，然后通过**ServletRegistrationBeanAdapter和****FilterRegistrationBeanAdapter两个适配器将Servlet.class和Filter.class封装成****RegistrationBean**

```
private static class ServletRegistrationBeanAdapter implements ServletContextInitializerBeans.RegistrationBeanAdapter<Servlet> {
    private final MultipartConfigElement multipartConfig;

    ServletRegistrationBeanAdapter(MultipartConfigElement multipartConfig) {
        this.multipartConfig = multipartConfig;
    }

    public RegistrationBean createRegistrationBean(String name, Servlet source, int totalNumberOfSourceBeans) {
        String url = totalNumberOfSourceBeans == 1 ? "/" : "/" + name + "/";
        if (name.equals("dispatcherServlet")) {
            url = "/";
        }
        //还是将Servlet.class实例封装成ServletRegistrationBean对象
        //这和我们自己创建ServletRegistrationBean对象是一模一样的
        ServletRegistrationBean bean = new ServletRegistrationBean(source, new String[]{url});
        bean.setMultipartConfig(this.multipartConfig);
        return bean;
    }
}

private static class FilterRegistrationBeanAdapter implements ServletContextInitializerBeans.RegistrationBeanAdapter<Filter> {
    private FilterRegistrationBeanAdapter() {
    }

    public RegistrationBean createRegistrationBean(String name, Filter source, int totalNumberOfSourceBeans) {
        //Filter.class实例封装成FilterRegistrationBean对象
        return new FilterRegistrationBean(source, new ServletRegistrationBean[0]);
    }
}
```

代码中注释很清楚了还是将Servlet.class实例封装成ServletRegistrationBean对象，将Filter.class实例封装成FilterRegistrationBean对象，这和我们自己定义ServletRegistrationBean对象是一模一样的，现在所有的ServletRegistrationBean、FilterRegistrationBean

Servlet.class、Filter.class都添加到List<ServletContextInitializer> sortedList这个集合中去了，接着就是遍历这个集合，执行其**onStartup**方法了

### ServletContextInitializer的**onStartup**方法

**ServletRegistrationBean**

```
public class ServletRegistrationBean extends RegistrationBean {
    private static final Log logger = LogFactory.getLog(ServletRegistrationBean.class);
    private static final String[] DEFAULT_MAPPINGS = new String[]{"/*"};
    private Servlet servlet;
    
    public void onStartup(ServletContext servletContext) throws ServletException {
        Assert.notNull(this.servlet, "Servlet must not be null");
        String name = this.getServletName();
        //调用ServletContext的addServlet
        Dynamic added = servletContext.addServlet(name, this.servlet);
    }
    
    //略...
}

private javax.servlet.ServletRegistration.Dynamic addServlet(String servletName, String servletClass, Servlet servlet, Map<String, String> initParams) throws IllegalStateException {
    if (servletName != null && !servletName.equals("")) {
        if (!this.context.getState().equals(LifecycleState.STARTING_PREP)) {
            throw new IllegalStateException(sm.getString("applicationContext.addServlet.ise", new Object[]{this.getContextPath()}));
        } else {
            Wrapper wrapper = (Wrapper)this.context.findChild(servletName);
            if (wrapper == null) {
                wrapper = this.context.createWrapper();
                wrapper.setName(servletName);
                this.context.addChild(wrapper);
            } else if (wrapper.getName() != null && wrapper.getServletClass() != null) {
                if (!wrapper.isOverridable()) {
                    return null;
                }

                wrapper.setOverridable(false);
            }

            if (servlet == null) {
                wrapper.setServletClass(servletClass);
            } else {
                wrapper.setServletClass(servlet.getClass().getName());
                wrapper.setServlet(servlet);
            }

            if (initParams != null) {
                Iterator i$ = initParams.entrySet().iterator();

                while(i$.hasNext()) {
                    Entry<String, String> initParam = (Entry)i$.next();
                    wrapper.addInitParameter((String)initParam.getKey(), (String)initParam.getValue());
                }
            }

            return this.context.dynamicServletAdded(wrapper);
        }
    } else {
        throw new IllegalArgumentException(sm.getString("applicationContext.invalidServletName", new Object[]{servletName}));
    }
}
```

看到没，ServletRegistrationBean 中的 onStartup先获取Servlet的name，然后调用ServletContext的addServlet将Servlet加入到Tomcat中，这样我们就能发请求给这个Servlet了。

**AbstractFilterRegistrationBean**

```
public void onStartup(ServletContext servletContext) throws ServletException {
    Filter filter = this.getFilter();
    Assert.notNull(filter, "Filter must not be null");
    String name = this.getOrDeduceName(filter);
    //调用ServletContext的addFilter
    Dynamic added = servletContext.addFilter(name, filter);
}
```

AbstractFilterRegistrationBean也是同样的原理，先获取目标Filter，然后调用ServletContext的**addFilter**将Filter加入到Tomcat中，这样Filter就能拦截我们请求了。

## DispatcherServletAutoConfiguration

最熟悉的莫过于，在Spring Boot在自动配置SpringMVC的时候，会自动注册SpringMVC前端控制器：**DispatcherServlet**，该控制器主要在**DispatcherServletAutoConfiguration**自动配置类中进行注册的。DispatcherServlet是SpringMVC中的核心分发器。DispatcherServletAutoConfiguration也在spring.factories中配置了

**DispatcherServletConfiguration**

```
 1 @Configuration
 2 @ConditionalOnWebApplication
 3 // 先看下ClassPath下是否有DispatcherServlet.class字节码
 4 // 我们引入了spring-boot-starter-web，同时引入了tomcat和SpringMvc,肯定会存在DispatcherServlet.class字节码
 5 @ConditionalOnClass({DispatcherServlet.class})
 6 // 这个配置类的执行要在EmbeddedServletContainerAutoConfiguration配置类生效之后执行
 7 // 毕竟要等Tomcat启动后才能往其中注入DispatcherServlet
 8 @AutoConfigureAfter({EmbeddedServletContainerAutoConfiguration.class})
 9 protected static class DispatcherServletConfiguration {
10   public static final String DEFAULT_DISPATCHER_SERVLET_BEAN_NAME = "dispatcherServlet";
11   public static final String DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME = "dispatcherServletRegistration";
12   @Autowired
13   private ServerProperties server;
14 
15   @Autowired
16   private WebMvcProperties webMvcProperties;
17 
18   @Autowired(required = false)
19   private MultipartConfigElement multipartConfig;
20 
21   // Spring容器注册DispatcherServlet
22   @Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
23   public DispatcherServlet dispatcherServlet() {
24     // 直接构造DispatcherServlet，并设置WebMvcProperties中的一些配置
25     DispatcherServlet dispatcherServlet = new DispatcherServlet();
26     dispatcherServlet.setDispatchOptionsRequest(this.webMvcProperties.isDispatchOptionsRequest());
27     dispatcherServlet.setDispatchTraceRequest(this.webMvcProperties.isDispatchTraceRequest());
28     dispatcherServlet.setThrowExceptionIfNoHandlerFound(this.webMvcProperties.isThrowExceptionIfNoHandlerFound());
29     return dispatcherServlet;
30   }
31 
32   @Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
33   public ServletRegistrationBean dispatcherServletRegistration() {
34     // 直接使用DispatcherServlet和server配置中的servletPath路径构造ServletRegistrationBean
35     // ServletRegistrationBean实现了ServletContextInitializer接口，在onStartup方法中对应的Servlet注册到Servlet容器中
36     // 所以这里DispatcherServlet会被注册到Servlet容器中，对应的urlMapping为server.servletPath配置
37     ServletRegistrationBean registration = new ServletRegistrationBean(dispatcherServlet(), this.server.getServletMapping());
38     registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
39     if (this.multipartConfig != null) {
40       registration.setMultipartConfig(this.multipartConfig);
41     }
42     return registration;
43   }
44 
45   @Bean // 构造文件上传相关的bean
46   @ConditionalOnBean(MultipartResolver.class)
47   @ConditionalOnMissingBean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
48   public MultipartResolver multipartResolver(MultipartResolver resolver) {
49     return resolver;
50   }
51 
52 }
```

先看下ClassPath下是否有DispatcherServlet.class字节码， 我们引入了spring-boot-starter-web，同时引入了tomcat和SpringMvc,肯定会存在DispatcherServlet.class字节码，如果没有导入spring-boot-starter-web，则这个配置类将不会生效

然后往Spring容器中注册DispatcherServlet实例，接着又加入ServletRegistrationBean实例，并把DispatcherServlet实例作为参数，上面我们已经学过了ServletRegistrationBean的逻辑，在Tomcat启动的时候，会获取所有的ServletRegistrationBean，并执行其中的onstartup方法，将DispatcherServlet注册到Servlet容器中，这样就类似原来的web.xml中配置的dispatcherServlet。

```
<servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>dispatcherServlet</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

所以只要导入了spring-boot-starter-web这个starter，SpringBoot就有了Tomcat容器，并且往Tomcat容器中注册了DispatcherServlet对象，这样就能接收到我们的请求了