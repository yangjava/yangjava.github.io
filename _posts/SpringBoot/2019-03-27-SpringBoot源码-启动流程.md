---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot 启动流程
从源代码的角度来看看Spring Boot的启动过程到底是怎么样的


## 运行 SpringApplication.run() 方法

```
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
    
}
```
进入源码**SpringApplication.run(HelloWorldMainApplication.class, args);**

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
- 

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


### 设置监听器(Listener)