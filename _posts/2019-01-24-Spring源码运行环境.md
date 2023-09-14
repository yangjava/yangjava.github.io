---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring运行环境
Spring在创建容器时，会创建Environment环境对象，用于保存spring应用程序的运行环境相关的信息。在创建环境时，需要创建属性源属性解析器，会解析属性值中的占位符，并进行替换。

创建环境时，会通过System.getProperties()获取JVM系统属性，会通过System.getenv()获取JVM环境属性。

## 概念
environment 翻译过来即是“环境”，表示应用程序运行环境。Environment表示当前Spring程序运行的环境，主要管理profiles和properties两种信息。

- profiles用来区分当前是dev(开发)环境还是test(测试)环境或者prod(生产)环境。
- properties表示所有的属性，包括操作系统环境变量，如PATH，JDK相关配置，如java.vm.specification.version(JDK版本)，还有我们通过properties文件和yml文件自定义的属性。

Spring 应用程序运行环境概念，主要包含两个方面的内容。profiles 决定了哪些 Bean 会被加载；properties 就是配置，就是键值对

看看 org.springframework.core.env.Environment 接口的部分注释：

```java
/**
 * Interface representing the environment in which the current application is running.
 * Models two key aspects of the application environment: profiles and
 * properties. Methods related to property access are exposed via the
 * superinterface.
 * ...  ...
 */
 public interface Environment extends PropertyResolver {
    // 
 }
```

org.springframework.core.env.Environment 是表示当前应用程序运行环境的接口，为应用程序运行环境的两个关键方面(profiles 和 properties)建模。

其中和 property 访问相关的 Methods 在 Environment 父接口 PropertyResolver 中定义。

由上可知 Environment 接口继承自接口 PropertyResolver，PropertyResolver 提供了 propertiy 访问的相关方法，Environment 本身提供了访问 profile 的相关方法。

- profile 是一个命名的,bean定义的逻辑组。bean 只有在给定的 profile 处于活动状态时，才会在容器中注册
- propertiy 即属性，可以简单理解为应用程序过程中一直存在的键值对集合

## Environment 体系结构
Environment 体系涉及到类或者接口还是比较多的，下面对其做简单介绍，可以仔细看看并尝试记住，需要对各个类或者接口有简单映像，不然阅读到后面容易迷糊。

- PropertyResolver：提供 properties 访问功能。
- ConfigurablePropertyResolver：继承自 PropertyResolver，额外提供 properties 类型转换(基于org.springframework.core.convert.ConversionService)功能，本文不做详细介绍。
- Environment：继承自 PropertyResolver，额外提供获取 profiles 的功能。
- ConfigurableEnvironment：继承自 ConfigurablePropertyResolver 和 Environment，提供设置活的 profile 和默认的 profile 的功能。
- ConfigurableWebEnvironment：继承自 ConfigurableEnvironment，并且提供配置 ServletContext 和ServletConfig 的功能。
- AbstractEnvironment：实现了 ConfigurableEnvironment 接口，默认属性和存储容器的定义，并且实现了 ConfigurableEnvironment 种的方法，并且为子类预留可覆盖了扩展方法。
- StandardEnvironment：继承自 AbstractEnvironment，非 Servlet(Web) 环境下的标准 Environment 实现。实现钩子方法，customizePropertySources(..)，初始化 systemEnvironment 和 systemProperties。
- StandardServletEnvironment：继承自 StandardEnvironment，Servlet(Web) 环境下的标准实现。实现钩子方法，customizePropertySources(..),初始化 servletContextInitParams、servletConfigInitParams 和 jndiProperties Web 应用环境需要的参数。

## properties

### PropertySource

先看开始正式介绍 properties 之前，先了解一下 PropertySource 抽象类。PropertySource 表示一个属性源（键值对集合）
```
public abstract class PropertySource {
    protected final String name;
    protected final T source;
    
    public boolean containsProperty(String name) {
        return (getProperty(name) != null);
    }
    // 抽象方法，留给字类实现
    @Nullable
    public abstract Object getProperty(String name);

    @Override
    public boolean equals(Object other) {
        return (this == other || (other instanceof PropertySource &&
                ObjectUtils.nullSafeEquals(this.name, ((PropertySource) other).name)));
    }

    // hashCode方法， 可以看到hash值仅和name有关系，这个 name 有点像 HashMap 里面的 key， 是唯一的
    @Override
    public int hashCode() {
        return ObjectUtils.nullSafeHashCode(this.name);
    }

     // 略。。。
}
```

源码很简单，预留了一个 getProperty 抽象方法给子类实现，重点需要关注的是覆写了的 equals 和 hashCode 方法，实际上只和 name 属性相关，这一点很重要，说明一个 PropertySource 实例绑定到一个唯一的 name，这个 name 有点像 HashMap 里面的 key。

PropertySource 的最常用子类是 MapPropertySource、PropertiesPropertySource、ResourcePropertySource、StubPropertySource 和 ComparisonPropertySource：

- MapPropertySource：封装一个 Map 对象为属性源， 继承抽象类 EnumerablePropertySource，增加了一个抽象方法 getPropertyNames()。
- PropertiesPropertySource：继承自 MapPropertySource，封装一个 Properties 对象为属性源。
- ResourcePropertySource：继承自 PropertiesPropertySource，封装一个 EncodedResource 对象为属性源。
- RandomValuePropertySource：封装一个 Random 对象为属性源，位于 spring-boot.jar 中。
- ServletConfigPropertySource：封装一个 ServletConfig 对象为属性源，位于 spring-web.jar 中。
- ServletContextPropertySource：封装一个 ServletContext 对象为属性源，位于 spring-web.jar 中。
- CompositePropertySource: 封装 Set> propertySources 对象作为属性源。
- CommandLinePropertySource：以命令行参数作为属性源
- SimpleCommandLinePropertySource：继承自CommandLinePropertySource。
  简单命令行属性源，使用SimpleCommandLineArgsParser解析器对象解析输入的String数组，把返回的CommandLineArgs对象作为属性的来源。
- JOptCommandLinePropertySource：基于JOpt Simple的属性源实现，JOpt Simple是一个解析命令行选项参数的第三方库
- SystemEnvironmentPropertySource：系统环境属性源，继承 MapPropertySource，此属性源在根据name获取对应的value时，与父类实现不太一样。它认为name不区分大小写，且name中包含的'.'点与'_'下划线是等效的，因此在获取value之前，都会对name进行一次处理。
- StubPropertySource：PropertySource 的一个内部类，source 设置为 new Object()，实际上就是空实现。用于占位。
- ComparisonPropertySource：继承自 StubPropertySource，所有属性访问方法强制抛出异常，作用就是一个不可访问属性的空实现。

## Environment 存储容器

Environment 的静态属性和存储容器都是在 AbstractEnvironment 中定义的 。Environment 的静态属性和存储容器。

实际上，Environment 的存储容器就是 org.springframework.core.env.PropertySource 的子类集合，AbstractEnvironment 中使用的实例是 org.springframework.core.env.MutablePropertySources。

AbstractEnvironment 中的属性定义：

```
private final MutablePropertySources propertySources = new MutablePropertySources();
```

再来看看 MutablePropertySources 是个什么东西：

```
// PropertySources提供处理多个PropertySource的方法
public interface PropertySources extends Iterable> {
    boolean contains(String name);
    PropertySource get(String name);
}

// MutablePropertySources 实现了PropertySources
public class MutablePropertySources implements PropertySources {
    //内部管理一个 CopyOnWriteArrayList propertySourceList 对象，即 PropertySource 列表
    private final List> propertySourceList = new CopyOnWriteArrayList>();
    // 判断 propertySourceList 是否包含 name
    @Override
    public boolean contains(String name) {
        return this.propertySourceList.contains(PropertySource.named(name));
    }
     // 从 propertySourceList 获取名称为 name 的 PropertySource， 如果不存在，返回 null， 如果存在一个或多个，返回第一个
    @Override
    public PropertySource get(String name) {
        int index = this.propertySourceList.indexOf(PropertySource.named(name));
        return (index != -1 ? this.propertySourceList.get(index) : null);
    }
    // 迭代器同 propertySourceList 迭代器
    @Override
    public Iterator> iterator() {
        return this.propertySourceList.iterator();
    }

    //添加一个 PropertySource， 赋予最高优先级
    public void addFirst(PropertySource propertySource) {
        if (logger.isDebugEnabled()) {
            logger.debug(String.format("Adding [%s] PropertySource with highest search precedence",
                    propertySource.getName()));
        }
        removeIfPresent(propertySource);
        this.propertySourceList.add(0, propertySource);
    }

    //添加一个 PropertySource， 赋予最低优先级
    public void addLast(PropertySource propertySource) {
        if (logger.isDebugEnabled()) {
            logger.debug(String.format("Adding [%s] PropertySource with lowest search precedence",
                    propertySource.getName()));
        }
        removeIfPresent(propertySource);
        this.propertySourceList.add(propertySource);
    }

    //添加一个 PropertySource， 优先级在 relativePropertySourceName 的前一位
    public void addBefore(String relativePropertySourceName, PropertySource propertySource) {
        if (logger.isDebugEnabled()) {
            logger.debug(String.format("Adding [%s] PropertySource with search precedence immediately higher than [%s]",
                    propertySource.getName(), relativePropertySourceName));
        }
        // check 两个 name 是否相同， 如果相同抛出异常
        assertLegalRelativeAddition(relativePropertySourceName, propertySource);
        removeIfPresent(propertySource);
        int index = assertPresentAndGetIndex(relativePropertySourceName);
        addAtIndex(index, propertySource);
    }

    //添加一个 PropertySource， 优先级在 relativePropertySourceName 的后一位
    public void addAfter(String relativePropertySourceName, PropertySource propertySource) {
        if (logger.isDebugEnabled()) {
            logger.debug(String.format("Adding [%s] PropertySource with search precedence immediately lower than [%s]",
                    propertySource.getName(), relativePropertySourceName));
        }
        // check 两个 name 是否相同， 如果相同抛出异常
        assertLegalRelativeAddition(relativePropertySourceName, propertySource);
        removeIfPresent(propertySource);
        int index = assertPresentAndGetIndex(relativePropertySourceName);
        addAtIndex(index + 1, propertySource);
    }

    // 返回 PropertySource 的优先级
    public int precedenceOf(PropertySource propertySource) {
        return this.propertySourceList.indexOf(propertySource);
    }

    // 删除名称为 name 的 PropertySource
    public PropertySource remove(String name) {
        if (logger.isDebugEnabled()) {
            logger.debug(String.format("Removing [%s] PropertySource", name));
        }
        int index = this.propertySourceList.indexOf(PropertySource.named(name));
        return (index != -1 ? this.propertySourceList.remove(index) : null);
    }

    // 替换名称为 name 的 PropertySource
    public void replace(String name, PropertySource propertySource) {
        if (logger.isDebugEnabled()) {
            logger.debug(String.format("Replacing [%s] PropertySource with [%s]",
                    name, propertySource.getName()));
        }
        //判断 name 存在， 如果不存在， 抛出异常
        int index = assertPresentAndGetIndex(name);
        this.propertySourceList.set(index, propertySource);
    }

    // 如果  PropertySource 存在，则删除
    protected void removeIfPresent(PropertySource propertySource) {
        this.propertySourceList.remove(propertySource);
    }

    // 将 PropertySource 放置在列表的指定为止（优先级）
    private void addAtIndex(int index, PropertySource propertySource) {
        removeIfPresent(propertySource);
        this.propertySourceList.add(index, propertySource);
    }
}
```

MutablePropertySources 内部属性是：

```
private final List> propertySourceList = new CopyOnWriteArrayList<>();
```

这个属性就是最底层的存储容器，也就是环境属性都是存放在一个 CopyOnWriteArrayList 实例中。MutablePropertySources 是 PropertySources 的子类，

它提供了 get(String name)、addFirst、addLast、addBefore、addAfter、remove、replace 等便捷方法，方便操作 propertySourceList 集合的元素。

由这些方法的命名可以看出 MutablePropertySources 将 propertySourceList 维护成一个有序的集合，以作为 propertySourceList 获取配置的优先级控制。

Environment 存储容器 是一个 CopyOnWriteArrayList> 实例，MutablePropertySources 提供了有序访问的 api。

## Environment 加载过程
spring在创建容器时需要指定需要加载配置文件路径，在加载配置文件路径时，需要解析字符串中的占位符。

解析占位符时，需要环境信息，此时会创建一个标准的spring运行环境，即创建StandardEnvironment对象。

如果是web项目，即说明的加载过程使用的是 StandardServletEnvironment。

### 加载的配置文件
调用setConfigLocations方法给spring设置需要加载的配置文件的路径，源码如下：
```
// 将配置文件的路径放到configLocations 字符串数组中
public void setConfigLocations(@Nullable String... locations) {
	if (locations != null) {
		Assert.noNullElements(locations, "Config locations must not be null");
		// 设置了几个配置文件，就创一个多长的字符串数组，用来存放配置文件的路径
		this.configLocations = new String[locations.length];
		for (int i = 0; i < locations.length; i++) {
			//解析路径，将解析的路径存放到字符串数组中
			this.configLocations[i] = resolvePath(locations[i]).trim();
		}
	}
	else {
		this.configLocations = null;
	}
}
```

### 解析配置文件路径中的占位符
调用resolvePath方法解析配置文件路径中的占位符，源码如下：
```
// 解析给定的路径，必要时用相应的环境属性值替换占位符。应用于配置位置。
protected String resolvePath(String path) {
	// 获取环境，解决所需的占位符
	return getEnvironment().resolveRequiredPlaceholders(path);
}
```

### 获取环境信息
调用getEnvironment方法获取环境信息，如果没有指定spring的环境信息，通过createEnvironment获取默认的环境，也就是spring的标准环境。

getEnvironment方法源码如下：
```
// 获取spring的环境信息，如果没有指定，获取到的时默认的环境
@Override
public ConfigurableEnvironment getEnvironment() {
	if (this.environment == null) {
		this.environment = createEnvironment();
	}
	return this.environment;
}
```

### 获取默认的环境
调用createEnvironment方法获取默认的环境（spring的标准环境），使用StandardEnvironment无参构造创建对象。源码如下：
```
// 获取默认的环境
protected ConfigurableEnvironment createEnvironment() {
	return new StandardEnvironment();
}
```

## Spring的标准环境StandardEnvironment
Spring的标准环境StandardEnvironment适合在非web应用程序中使用。

在AbstractApplicationContext类的createEnvironment方法中会调用StandardEnvironment的无参构造方法创建环境对象。

### 调用StandardEnvironment
调用StandardEnvironment的无参构造方法，该方法中没有任何逻辑处理，源码如下：
```
public StandardEnvironment() {}
```

### AbstractEnvironment构造
StandardEnvironment类是AbstractEnvironment抽象类的子类，因此使用StandardEnvironment的无参构造创建对象时会调用父类AbstractEnvironment的无参构造方法。
```
	public AbstractEnvironment() {
		customizePropertySources(this.propertySources);
	}
	// 空方法，实现类可以自定义 PropertySources
    protected void customizePropertySources(MutablePropertySources propertySources) {
    }
```
重写AbstractEnvironment类中的customizePropertySources方法，用于设置属性源，该方法通过父类进行回调。

### StandardEnvironment 的 customizePropertySources()方法
再看看 StandardEnvironment 的 customizePropertySources()方法
```
// 设置属性源，JVM系统属性中的属性将优先于环境属性中的属性。
@Override
protected void customizePropertySources(MutablePropertySources propertySources) {
	/**
	 * 1、MutablePropertySources类中使用CopyOnWriteArrayList存储属性源，集合中存储PropertySource的子类
	 * 2、PropertiesPropertySource是PropertySource的子类，PropertySource类中有三个成员变量
	 * 		1）logger：日志对象
	 * 		2）name：用来保存属性源名称
	 * 		3）source：用来保存属性源中的属性
	 */
	// getSystemProperties()：通过System.getProperties()获取JVM属性键值对，并转成Map
	propertySources.addLast(
			new PropertiesPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
	// getSystemEnvironment()：通过System.getenv()获取环境属性键值对，并撰成Map
	propertySources.addLast(
			new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
}
```
该方法会获取JVM系统属性和环境属性并设置到MutablePropertySources类中存放属性源的CopyOnWriteArrayList中。
- JVM系统属性通过System.getProperties()获取；
- 环境属性通过System.getenv()获取。

StandardEnvironment类中设置了两个静态常量：
- systemEnvironment：以系统环境为属性源
- systemProperties：以JVM系统属性为属性源
```
/** System environment property source name: {@value}.
* 系统环境属性源名:{@value}。 */
  public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";

/** JVM system properties property source name: {@value}.
* JVM系统属性属性源名称:{@value}。*/
  public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";
```

###  StandardServletEnvironment 的 customizePropertySources()
```
// StandardServletEnvironment 的 customizePropertySources()方法
// 此处在 MutablePropertySources 中放入两个 StubPropertySource 类型的 PropertySource，
// servletConfigInitParams 和 servletContextInitParams
// 由上文，我们知道 StubPropertySource 是一个空实现，原因是 Servlet 相关的配置得到 ServletContext 初始化时才有，
// 此处仅起到一个占位的作用， 在 AnnotationConfigEmbeddedWebApplicationContext 初始化时，
// 才会通过 initPropertySources 方重新赋值
@Override
protected void customizePropertySources(MutablePropertySources propertySources) {
    // 构造名称为 servletConfigInitParams 的 StubPropertySource，并放入 MutablePropertySources 
    propertySources.addLast(new StubPropertySource(SERVLET_CONFIG_PROPERTY_SOURCE_NAME));
    // 构造名称为 servletContextInitParams 的 StubPropertySource，并放入 MutablePropertySources 
    propertySources.addLast(new StubPropertySource(SERVLET_CONTEXT_PROPERTY_SOURCE_NAME));
    // 如果存在 JNDI 环境， 构造名称为 jndiProperties 的 JndiPropertySource，并放入 MutablePropertySources
    if (JndiLocatorDelegate.isDefaultJndiEnvironmentAvailable()) {
        propertySources.addLast(new JndiPropertySource(JNDI_PROPERTY_SOURCE_NAME));
    }
    // 调用父类的customizePropertySources(), 即调用 StandardServletEnvironment 的 customizePropertySources()
    super.customizePropertySources(propertySources);
}
```

```
// 被 AnnotationConfigEmbeddedWebApplicationContext 调用，洗初始化 servletConfigInitParams 和 servletContextInitParams
@Override
public void initPropertySources(ServletContext servletContext, ServletConfig servletConfig) {
    WebApplicationContextUtils.initServletPropertySources(getPropertySources(), servletContext, servletConfig);
}

// WebApplicationContextUtils#initServletPropertySources
// 重新构建
public static void initServletPropertySources(
            MutablePropertySources propertySources, ServletContext servletContext, ServletConfig servletConfig) {

    Assert.notNull(propertySources, "'propertySources' must not be null");
    if (servletContext != null && propertySources.contains(StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME) &&
                propertySources.get(StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME) instanceof StubPropertySource) {
    // 使用 servletContext 重新构造名称为 servletContextInitParams 的 ServletContextPropertySource，
    // 并替换原有的占位StubPropertySource
    propertySources.replace(StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME, new ServletContextPropertySource(StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME, servletContext));
    }
    if (servletConfig != null && propertySources.contains(StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME) &&
                propertySources.get(StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME) instanceof StubPropertySource) {
    // 使用 servletConfig 重新构造名称为 servletConfigInitParams 的 ServletConfigPropertySource，
    // 并替换原有的占位StubPropertySource
    propertySources.replace(StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME, new ServletConfigPropertySource(StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME, servletConfig));
        }
    }
```

看完代码，可以简单总结一下代码执行顺序，
- AbstractEnvironment 的构造方法 调用了 customizePropertySources（）。
- StandardEnvironment 和 StandardServletEnvironment 重写 customizePropertySources（）。
- StandardServletEnvironment 在 customizePropertySources（）中构造了 StubPropertySource 类型的 servletConfigInitParams 和 servletContextInitParams 放入 MutablePropertySources 中占位， 当servlet环境准备好， 重新初始化和替换，还构建了 jndiProperties 放入 MutablePropertySources 中。最后调用 StandardEnvironment#customizePropertySources（）。
- StandardEnvironment 在 customizePropertySources 方法中，初始化了 systemProperties 和 systemEnvironment。

上面的逻辑都是在 StandardServletEnvironment 构造中完成的，且都是用了 addLast（）方法操作，由执行顺序可得，配置的优先级递减排序：
  - servletConfigInitParams
  - servletContextInitParams
  - jndiProperties
  - systemProperties
  - systemEnvironment
以上构造过程中的配置加载过程已经说完了。再来看看 spring boot web 服务启动过程中的配置加载过程。

### 命令行参数

本文不过多介绍spring boot web 服务启动流程，直接到启动过程中，环境准备阶段代码。
由上文描述介绍 Environment， 我们知道，Environment 的配置是存储在一个 MutablePropertySources propertySources 对象中，而 ConfigurableEnvironment 暴露了获取该对象的方法

```
MutablePropertySources getPropertySources();
```

这就允许程序在任何地方，只要能获取到 Environment 对象，就可以获取到 MutablePropertySources propertySources 对象，并对齐做修改。
此处介绍两个spring boot web 程序启动过程中的两个 case, 学习一下如何修改，主要是修改的地方太多了，没法枚举。

#### SpringApplication#prepareEnvironment() 命令行参数 commandLineArgs

```
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,ApplicationArguments applicationArguments) {
    // 构造函数创建 Environment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    //  配置 Environment 的 PropertySources 和 Profiles
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    // 发布 ApplicationEnvironmentPreparedEvent 事件， 此处不介绍 spring 事件机制， 
    // 事件发布后，其实也有很多地方修改该了Environment配置，此处不做介绍
    listeners.environmentPrepared(environment);
    if (!this.webEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertToStandardEnvironmentIfNecessary(environment);
    }
    return environment;
}

// 配置 Environment 的 PropertySources 和 Profiles
protected void configureEnvironment(ConfigurableEnvironment environment,
        String[] args) {
    // 配置 PropertySources
    configurePropertySources(environment, args);
    // 配置 Profiles
    configureProfiles(environment, args);
}

protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
    MutablePropertySources sources = environment.getPropertySources();
    if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
        sources.addLast(new MapPropertySource("defaultProperties", this.defaultProperties));
    }
    // 如果 存在命令行参数
    if (this.addCommandLineProperties && args.length > 0) {
        String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
        // MutablePropertySources 已存在 commandLineArgs， 
        // 则将已有的 commandLineArgs 与 新生成的 SimpleCommandLinePropertySource 变成一个合成的 CompositePropertySource， 
        // 替换MutablePropertySources中的 commandLineArgs
        if (sources.contains(name)) {
            PropertySource source = sources.get(name);
            CompositePropertySource composite = new CompositePropertySource(name);
            composite.addPropertySource(new SimpleCommandLinePropertySource(name + "-" + args.hashCode(), args));
            composite.addPropertySource(source);
            sources.replace(name, composite);
        }
        // MutablePropertySources 不存在 commandLineArgs， 
        // 新生成 SimpleCommandLinePropertySource 放入MutablePropertySources的最高优先级位置
        else {
            sources.addFirst(new SimpleCommandLinePropertySource(args));
        }
    }
}
```
执行configureEnvironment（）之后， 可以看到，多了一个的commandLineArgs， Environment debug截图：

### spring-cloud-config 加载外部配置

本文不会详细介绍 spring-cloud-config, 只会介绍其中一段关于 Environment 配置变更的部分。涉及到spring 初始化机制，本文也不会详细介绍。

```
// 省略不相关代码
// 实现了 ApplicationContextInitializer 接口， 在spring boot启动过程中，会调用 initialize（）方法，此处不做详细介绍
@Configuration
@EnableConfigurationProperties(PropertySourceBootstrapProperties.class)
public class PropertySourceBootstrapConfiguration implements ApplicationContextInitializer, Ordered {

    // 配置加载器， 主要作用是： 通过http请求，去config-server拉取配置
    @Autowired(required = false)
    private List propertySourceLocators = new ArrayList<>();

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        // 创建名称为 bootstrapProperties 合成资源
        CompositePropertySource composite = new CompositePropertySource(BOOTSTRAP_PROPERTY_SOURCE_NAME);
        // 对配置加载器排序
        AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
        boolean empty = true;
        // 当前应用上下文的 Environment
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        // 遍历多个配置加载器，加载配置
        for (PropertySourceLocator locator : this.propertySourceLocators) {
            //配置加载器返回结果， 如果为空， 继续下次循环， 如果配置不为空，添加到合成资源 CompositePropertySource 中
            PropertySource source = null;
            source = locator.locate(environment);
            if (source == null) {
                continue;
            }
            logger.info("Located property source: " + source);
            composite.addPropertySource(source);
            empty = false;
        }
        // 配置加载器返回的配置非空，即 CompositePropertySource 非空
        if (!empty) {
            MutablePropertySources propertySources = environment.getPropertySources();
            String logConfig = environment.resolvePlaceholders("${logging.config:}");
            LogFile logFile = LogFile.get(environment);
            // 如果 Environment 配置中已经包含 bootstrapProperties， 删除
            if (propertySources.contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
                propertySources.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
            }
            // 将 CompositePropertySource 添加到 MutablePropertySources 中
            insertPropertySources(propertySources, composite);
            reinitializeLoggingSystem(environment, logConfig, logFile);
            setLogLevels(environment);
            handleIncludedProfiles(environment);
        }
    }
    // 将 CompositePropertySource 添加到 MutablePropertySources 中
    private void insertPropertySources(MutablePropertySources propertySources, CompositePropertySource composite) {
        MutablePropertySources incoming = new MutablePropertySources();
        incoming.addFirst(composite);
        // 构造 PropertySourceBootstrapProperties remoteProperties， 使用 RelaxedDataBinder 为 remoteProperties 赋值
        PropertySourceBootstrapProperties remoteProperties = new PropertySourceBootstrapProperties();
        new RelaxedDataBinder(remoteProperties, "spring.cloud.config").bind(new PropertySourcesPropertyValues(incoming));
        // 如果 配置为远程覆盖本地，则将 composite 放在 propertySources 最高优先级位置
        if (!remoteProperties.isAllowOverride() 
                  || (!remoteProperties.isOverrideNone() && remoteProperties.isOverrideSystemProperties())) {
            propertySources.addFirst(composite);
            return;
        }
        // 如果 配置为远程不覆盖本地， 则将 composite 放在 propertySources 最低优先级位置
        if (remoteProperties.isOverrideNone()) {
            propertySources.addLast(composite);
            return;
        }
        // 当 systemEnvironment 存在，
        if (propertySources.contains(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME)) {
            //  如果配置 overrideSystemProperties 为 false, 将 composite 放在 systemEnvironment 优先级后面
            // 否则， 放在 systemEnvironment 优先级前面
            if (!remoteProperties.isOverrideSystemProperties()) {
                propertySources.addAfter(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME,
                        composite);
            } else {
                propertySources.addBefore(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME,
                        composite);
            }
        // 当 systemEnvironment 不存在，  则将 composite 放在 propertySources 最低优先级位置
        } else {
            propertySources.addLast(composite);
        }
    }
}
```

以上内容说明了 命令行参数 和 spring-cloud-config 获取到的远程配置 加载到 Environment 中的过程。

## Environment 属性访问

上文提到， Environment 的配置都存储在 MutablePropertySources propertySources 中，那 Environment 是如何访问到配置的呢？
代码在 AbstractEnvironment 中实现：

```
@Override
public String getProperty(String key) {
    return this.propertyResolver.getProperty(key);
}

@Override
public String getProperty(String key, String defaultValue) {
    return this.propertyResolver.getProperty(key, defaultValue);
}

@Override
public  T getProperty(String key, Class targetType) {
    return this.propertyResolver.getProperty(key, targetType);
}

@Override
public  T getProperty(String key, Class targetType, T defaultValue) {
    return this.propertyResolver.getProperty(key, targetType, defaultValue);
}
```

可以看到， 属性访问都委托给了 propertyResolver 对象, 而 PropertySourcesPropertyResolver 的构造函数 需要传入一个 propertySources

```
private final ConfigurablePropertyResolver propertyResolver = new PropertySourcesPropertyResolver(this.propertySources);

public PropertySourcesPropertyResolver(PropertySources propertySources) {
    this.propertySources = propertySources;
}
```

先看看 的类型结构

- PropertyResolver 决定基础 source 的接口
- ConfigurablePropertyResolver 继承 PropertyResolver， 配置接口，用来定制 ConversionService
- AbstractPropertyResolver 是 ConfigurablePropertyResolver 抽象实现， 决定基础 source 的抽象类
- PropertySourcesPropertyResolver 继承 AbstractPropertyResolver

### PropertyResolver-占位符处理

PropertyResolver是 PropertySourcesPropertyResolver 的顶层接口，主要提供属性检索和解析带占位符的文本。

```
public interface PropertyResolver {
    boolean containsProperty(String key);
    String getProperty(String key);
    String getProperty(String key, String defaultValue);
     T getProperty(String key, Class targetType);
     T getProperty(String key, Class targetType, T defaultValue);
    @Deprecated
     Class getPropertyAsClass(String key, Class targetType);
    String getRequiredProperty(String key) throws IllegalStateException;
     T getRequiredProperty(String key, Class targetType) throws IllegalStateException;
    String resolvePlaceholders(String text);
    String resolveRequiredPlaceholders(String text) throws IllegalArgumentException;
}
```

ConfigurablePropertyResolver 确定解析占位符的一些配置方法。

```
public interface ConfigurablePropertyResolver extends PropertyResolver {
    ConfigurableConversionService getConversionService();
    void setConversionService(ConfigurableConversionService conversionService);
    void setPlaceholderPrefix(String placeholderPrefix);
    void setPlaceholderSuffix(String placeholderSuffix);
    void setValueSeparator(String valueSeparator);
    void setIgnoreUnresolvableNestedPlaceholders(boolean ignoreUnresolvableNestedPlaceholders);
    void setRequiredProperties(String... requiredProperties);
    void validateRequiredProperties() throws MissingRequiredPropertiesException;
}
```

AbstractPropertyResolver 封装了解析占位符的具体实现，而 PropertySourcesPropertyResolver 主要是负责提供数据源。AbstractPropertyResolver 中有两个成员变量都是 PropertyPlaceholderHelper 对象，区别在于构造参数不同。

```
// 解析占位符，忽略不能解析的占位符
public String resolvePlaceholders(String text) {
    if (this.nonStrictHelper == null) {
        this.nonStrictHelper = createPlaceholderHelper(true);
    }
    return doResolvePlaceholders(text, this.nonStrictHelper);
}

// 解析占位符，不忽略不能解析的占位符， 遇到不能解析的占位符， 抛出异常
public String resolveRequiredPlaceholders(String text) throws IllegalArgumentException {
    if (this.strictHelper == null) {
        this.strictHelper = createPlaceholderHelper(false);
    }
    return doResolvePlaceholders(text, this.strictHelper);
}

// 将解析占位符的逻辑委托给 PropertyPlaceholderHelper helper对象
private String doResolvePlaceholders(String text, PropertyPlaceholderHelper helper) {
    return helper.replacePlaceholders(text, new PropertyPlaceholderHelper.PlaceholderResolver() {
        // 入参是占位符， 出参是 PropertySource 存储的原生value, 依赖抽象方法 getPropertyAsRawString()
        @Override
        public String resolvePlaceholder(String placeholderName) {
            return getPropertyAsRawString(placeholderName);
        }
    });
}

// 获取 PropertySource 存储的原生value, 由字类实现
protected abstract String getPropertyAsRawString(String key);
```

在看看PropertyPlaceholderHelper代码：

```
// 占位符处理器
public class PropertyPlaceholderHelper {
    private static final Map wellKnownSimplePrefixes = new HashMap(4);
    static {
        wellKnownSimplePrefixes.put("}", "{");
        wellKnownSimplePrefixes.put("]", "[");
        wellKnownSimplePrefixes.put(")", "(");
    }


    private final String placeholderPrefix; // 默认 "${"
    private final String placeholderSuffix; // 默认 "}"
    private final String simplePrefix;         
    private final String valueSeparator;    // 默认 ":"
    private final boolean ignoreUnresolvablePlaceholders;  // 是否忽略占位符中不存在的数据源，false会抛出IllegalArgumentException。

     // 构造函数
    public PropertyPlaceholderHelper(String placeholderPrefix, String placeholderSuffix, String valueSeparator, boolean ignoreUnresolvablePlaceholders) {
        Assert.notNull(placeholderPrefix, "'placeholderPrefix' must not be null");
        Assert.notNull(placeholderSuffix, "'placeholderSuffix' must not be null");

        this.placeholderPrefix = placeholderPrefix;
        this.placeholderSuffix = placeholderSuffix;
        String simplePrefixForSuffix = wellKnownSimplePrefixes.get(this.placeholderSuffix);
        if (simplePrefixForSuffix != null && this.placeholderPrefix.endsWith(simplePrefixForSuffix)) {
            this.simplePrefix = simplePrefixForSuffix;
        }
        else {
            this.simplePrefix = this.placeholderPrefix;
        }
        this.valueSeparator = valueSeparator;
        this.ignoreUnresolvablePlaceholders = ignoreUnresolvablePlaceholders;
    }


    // 用来将 value 中占位符替换为从 Properties 取得的值
    public String replacePlaceholders(String value, final Properties properties) {
        Assert.notNull(properties, "'properties' must not be null");
        return replacePlaceholders(value, new PlaceholderResolver() {
            @Override
            public String resolvePlaceholder(String placeholderName) {
                return properties.getProperty(placeholderName);
            }
        });
    }

    // 用来将 value 中占位符替换为从 PlaceholderResolver 取得的值
    public String replacePlaceholders(String value, PlaceholderResolver placeholderResolver) {
        Assert.notNull(value, "'value' must not be null");
        return parseStringValue(value, placeholderResolver, new HashSet());
    }
     
    //  用来将 value 中占位符替换为从 PlaceholderResolver 取得的值， 具体实现逻辑
    // 这是一个递归的解析过程，遇到${开头就会查找最后一个}符号，
    // 将最外层占位符内的内容作为新的 value 再次传入 parseStringValue() 方法中，
    // 这样最深层次也就是最先返回的就是最里层的占位符名字。
    // 调用placeholderResolver将占位符名字转换成它代表的值。
    // 如果值为null，则考虑使用默认值 ( valueSeparator 后的内容) 赋值给 propVal。
    // 由于 placeholderResolver 转换过的值有可能还会包含占位符，
    // 所以在此调用 parseStringValue() 方法将带有占位符的 propVal 传入返回真正的值,用 propVal 替换占位符。
    // 如果 propVal==null，判断是否允许忽略不能解析的占位符，
    // 如果可以，重置 startIndex，继续解析同一层次的占位符。否则抛出异常，这个函数的返回值就是它上一层次的占位符解析值
    protected String parseStringValue(String value, PlaceholderResolver placeholderResolver, Set visitedPlaceholders) {

        StringBuilder result = new StringBuilder(value);

        int startIndex = value.indexOf(this.placeholderPrefix);
        // 当 包含 placeholderPrefix（“${”）时
        while (startIndex != -1) {
            int endIndex = findPlaceholderEndIndex(result, startIndex);
            if (endIndex != -1) {
                // 解析到占位符
                String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
                String originalPlaceholder = placeholder;
                // 将解析到的占位符添加到visitedPlaceholders， 如果添加失败，表示由循环的占位符，抛出异常
                if (!visitedPlaceholders.add(originalPlaceholder)) {
                    throw new IllegalArgumentException("Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
                }
                // 递归调用, 继续转化占位符， 因为转化到的接口，可能仍然包含占位符
                placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
                // 获取占位符里面的 key 对应的值
                String propVal = placeholderResolver.resolvePlaceholder(placeholder);
                // 如果 propVal 为空且 this.valueSeparator 不为空
                if (propVal == null && this.valueSeparator != null) {
                                        int separatorIndex = placeholder.indexOf(this.valueSeparator);       
                    // 如果占位符包含默认值分割符   
                    if (separatorIndex != -1) {
                        // 获取正真的占位符， separatorIndex 之前的部分
                        String actualPlaceholder = placeholder.substring(0, separatorIndex);
                        // 默认值， separatorIndex 之后的部分
                        String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
                        // 重新解析正真的占位符， 如果值还是不存在，则使用默认值
                        propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
                        if (propVal == null) {
                            propVal = defaultValue;
                        }
                    }
                }
                // 如果 propVal 不为空
                if (propVal != null) {
                    // 递归调用， 继续解析 获取到的占位符值，即 propVal
                    propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
                    // 解析到结果 替换结果中
                    result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
                    if (logger.isTraceEnabled()) {
                        logger.trace("Resolved placeholder '" + placeholder + "'");
                    }
                    // 找到下一个 “${” 开始的位置， 赋值给 startIndex
                    startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
                } else if (this.ignoreUnresolvablePlaceholders) {
                    // 如果解析不到结果， ignoreUnresolvablePlaceholders == true的情况下
                    // 找到下一个 “${” 开始的位置， 赋值给 startIndex 
                    startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
                } else {
                    // 如果解析不到结果， ignoreUnresolvablePlaceholders == false 的情况下, 抛出异常
                    throw new IllegalArgumentException("Could not resolve placeholder '" +
                            placeholder + "'" + " in value \"" + value + "\"");
                }
                // 移除解析成功的占位符
                visitedPlaceholders.remove(originalPlaceholder);
            } else {
                // 占位符中 不包含“${”, 结束循环
                startIndex = -1;
            }
        }

        return result.toString();
    }
     // 判断嵌套的占位符是依据simplePrefix，找到当前占位符前缀对应的尾缀 
    private int findPlaceholderEndIndex(CharSequence buf, int startIndex) {
        int index = startIndex + this.placeholderPrefix.length();
        int withinNestedPlaceholder = 0;
        while (index < buf.length()) {
            if (StringUtils.substringMatch(buf, index, this.placeholderSuffix)) {
                if (withinNestedPlaceholder > 0) {
                    withinNestedPlaceholder--;
                    index = index + this.placeholderSuffix.length();
                }
                else {
                    return index;
                }
            }
            else if (StringUtils.substringMatch(buf, index, this.simplePrefix)) {
                withinNestedPlaceholder++;
                index = index + this.simplePrefix.length();
            }
            else {
                index++;
            }
        }
        return -1;
    }


    // PlaceholderResolver是一个接口定了一个方法，入参为占位符参数名，出参为占位符代表的值。
    public interface PlaceholderResolver {
        String resolvePlaceholder(String placeholderName);
    }
}
```

再来看看 PropertySourcesPropertyResolver 代码

```
public class PropertySourcesPropertyResolver extends AbstractPropertyResolver {
    // 数据源
    private final PropertySources propertySources;

    // 获取指定类型获取配置
    // key
    // targetValueType 目标类型
    // resolveNestedPlaceholders 是否处理占位符
    protected  T getProperty(String key, Class targetValueType, boolean resolveNestedPlaceholders) {
        if (this.propertySources != null) {
            // 顺序遍历 propertySources， 体现了优先级， 之前文中也提到了 MutablePropertySources 维护的是有序列表
            for (PropertySource propertySource : this.propertySources) {
                if (logger.isTraceEnabled()) {
                    logger.trace(String.format("Searching for key '%s' in [%s]", key, propertySource.getName()));
                }
                // 原生的配置 value
                Object value = propertySource.getProperty(key);
                if (value != null) {
                    // 判断是否处理占位符
                    if (resolveNestedPlaceholders && value instanceof String) {
                        value = resolveNestedPlaceholders((String) value);
                    }
                    logKeyFound(key, propertySource, value);
                    // 类型转换，转换成目标类型 targetValueType
                    return convertValueIfNecessary(value, targetValueType);
                }
            }
        }
        if (logger.isDebugEnabled()) {
            logger.debug(String.format("Could not find key '%s' in any property source", key));
        }
        return null;
    }
}
```

### ConversionService-类型转换

上文提到，在占位符处理完成之后， 最后一步是 convertValueIfNecessary(), 类型转化，代码还是在 AbstractPropertyResolver 中：

```
protected  T convertValueIfNecessary(Object value, Class targetType) {
    if (targetType == null) {
        return (T) value;
    }
    // 如果自定义 ConversionService， 则使用 DefaultConversionService
    ConversionService conversionServiceToUse = this.conversionService;
    if (conversionServiceToUse == null) {
        // Avoid initialization of shared DefaultConversionService if
        // no standard type conversion is needed in the first place...
        if (ClassUtils.isAssignableValue(targetType, value)) {
            return (T) value;
        }
        conversionServiceToUse = DefaultConversionService.getSharedInstance();
    }
    // 转化成功后，返回
    return conversionServiceToUse.convert(value, targetType);
}
```





