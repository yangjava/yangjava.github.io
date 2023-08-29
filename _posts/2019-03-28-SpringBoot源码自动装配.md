---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot源码自动装配
Spring Boot自动装配的本质就是通过Spring来读取META-INF/spring.factories中保存的配置类文件，然后加载bean定义的过程。

## 自动装配
Spring Boot 是一个快速开发、简化配置的框架，它支持自动装配。自动装配是指根据依赖关系，框架自动选择和配置合适的组件，无需手动编写代码。

Spring Boot 的自动装配通过注解实现，主要使用了 @EnableAutoConfiguration 注解。当我们在一个 Spring Boot 应用中添加了某些依赖，比如数据库驱动、web框架等，@EnableAutoConfiguration 注解会根据这些依赖自动进行相应的配置。

此外，Spring Boot 还支持自定义的自动装配。我们可以通过 @Conditional 注解、配置类等方式，自定义配置条件，以控制哪些组件会被自动装配。

## @EnableAutoConfiguration
EnableAutoConfiguration 只是一个简单地注解，自动装配核心功能的实现实际是通过 AutoConfigurationImportSelector类。
```java

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage //自动配置包
@Import({AutoConfigurationImportSelector.class}) //Spring的底层注解@Import，给容器中导入一个组件
public @interface EnableAutoConfiguration {
 
    //@AutoConfigurationPackage 注解的功能是由@Import 注解实现的，它是Spring框架的底层注解，它的作用就是给容器中导入某个组件类。
    //AutoConfigurationImportSelector可以帮助SpringBoot应用将所有符合条件@Configuration配置都加载到当前SpringBoot创建并使用的
    //IOC容器（ApplicationContext）中，AutoConfigurationImportSelector是通过SelectImports这个方法告诉SpringBoot都需要导入那些组件
 
    //AutoConfigurationImportSelector 组件是实现了 DeferredImportSelector 类，以及很多的 Aware 接口，这些 Aware 接口来实现一些回调方法，
    // 通过这些回调，把 AutoConfigurationImportSelector 的属性进行赋值。
 
 
    // 分析下 DeferredImportSelector 这个类
    // 有个内部接口 Group 接口，这个接口里面有两个方法 process() 和 selectImport()
    // 为什么要强度这两个方法，因为这两个方法在 SpringBoot 启动的时候会被调用。
    //跟自动配置逻辑相关的入口方法在 DeferredImportSelectorGrouping 类的 getImport() 方法处，
    // 所以我们就从 getImport() 方法开始入手。 先保留这个疑问，等到剖析run()方法时就会串起来的！！！
    //然后下面来看一下AutoConfigurationImportSelect组件类中的方法是怎么被调用的？
 
 
    //我们现在来看一下 DeferredImportSelectorGrouping 这个类：
    // 调用 DeferredImportSelectGrouping 的 getImport() 方法，在这个方法里面又会去调用 group.process() 和 group.selectImports(),
    // 去找这两个方法的具体实现
 
}
```



#### **@EnableAutoConfiguration**注解


- 开启自动配置功能
- 以前使用Spring需要配置的信息，Spring Boot帮助自动配置；
- **@EnableAutoConfiguration**通知SpringBoot开启自动配置功能，这样自动配置才能生效。

此注解顾名思义是可以自动配置，所以应该是springboot中最为重要的注解。在spring框架中就提供了各种以@Enable开头的注解，例如： @EnableScheduling、@EnableCaching、@EnableMBeanExport等； @EnableAutoConfiguration的理念和做事方式其实一脉相承简单概括一下就是，借助@Import的支持，收集和注册特定场景相关的bean定义。　　

- @EnableScheduling是通过@Import将Spring调度框架相关的bean定义都加载到IoC容器【定时任务、时间调度任务】
- @EnableMBeanExport是通过@Import将JMX相关的bean定义加载到IoC容器【监控JVM运行时状态】
- @EnableAutoConfiguration也是借助@Import的帮助，将所有符合自动配置条件的bean定义加载到IoC容器。

@EnableAutoConfiguration作为一个复合Annotation,其自身定义关键信息如下：

```
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage【重点注解】
@Import(AutoConfigurationImportSelector.class)【重点注解】
public @interface EnableAutoConfiguration {
...
}
```

### EnableAutoConfiguration注解

最重要的两个注解

- @AutoConfigurationPackage
- @Import(AutoConfigurationImportSelector.class)

#### AutoConfigurationPackage注解

- 自动配置包注解

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
 
}
```

**@Import(AutoConfigurationPackages.Registrar.class)**：默认将主配置类(**@SpringBootApplication**)所在的包及其子包里面的所有组件扫描到Spring容器中。如下

```
@Order(Ordered.HIGHEST_PRECEDENCE)
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
            BeanDefinitionRegistry registry) {
          //默认将会扫描@SpringBootApplication标注的主配置类所在的包及其子包下所有组件
        register(registry, new PackageImport(metadata).getPackageName());
    }

    @Override
    public Set<Object> determineImports(AnnotationMetadata metadata) {
        return Collections.<Object>singleton(new PackageImport(metadata));
    }
}
```

- 注册当前启动类的根package；
- 注册org.springframework.boot.autoconfigure.AutoConfigurationPackages的BeanDefinition。

#### Import(AutoConfigurationImportSelector.class)注解

**AutoConfigurationImportSelector**： 导入哪些组件的选择器，将所有需要导入的组件以全类名的方式返回，这些组件就会被添加到容器中。

**AutoConfigurationImportSelector 实现**了 DeferredImportSelector 从 ImportSelector继承的方法：**selectImports**。

```
@Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        }
        AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
                .loadMetadata(this.beanClassLoader);
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        List<String> configurations = getCandidateConfigurations(annotationMetadata,
                attributes);
        configurations = removeDuplicates(configurations);
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);
        configurations = filter(configurations, autoConfigurationMetadata);
        fireAutoConfigurationImportEvents(configurations, exclusions);
        return StringUtils.toStringArray(configurations);
    }
```

我们主要看List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes)会给容器中注入众多的自动配置类（xxxAutoConfiguration）。

```
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
            AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
        getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
    //...
    return configurations;
}

protected Class<?> getSpringFactoriesLoaderFactoryClass() {
    return EnableAutoConfiguration.class;
}

public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
    String factoryClassName = factoryClass.getName();
    try {
        //从类路径的META-INF/spring.factories中加载所有默认的自动配置类
        Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                                 ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        List<String> result = new ArrayList<String>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
            //获取EnableAutoConfiguration指定的所有值,也就是EnableAutoConfiguration.class的值
            String factoryClassNames = properties.getProperty(factoryClassName);
            result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
        }
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() + "] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```

就是给容器中导入这个场景需要的所有组件，并配置好这些组件。其实是去加载各个组件jar下的  public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"外部文件。

SpringBoot启动的时候从类路径下的 **META-INF/spring.factories**中获取EnableAutoConfiguration指定的值，并将这些值作为自动配置类导入到容器中，自动配置类就会生效，最后完成自动配置工作。EnableAutoConfiguration默认在spring-boot-autoconfigure这个包中，如下图

```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.boot.autoconfigure.integration.IntegrationPropertiesEnvironmentPostProcessor

... ...
```

该方法在springboot启动流程——bean实例化前被执行，返回要实例化的类信息列表；

如果获取到类信息，spring可以通过类加载器将类加载到jvm中，现在我们已经通过spring-boot的starter依赖方式依赖了我们需要的组件，那么这些组件的类信息在select方法中就可以被获取到。

```
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
 List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
 Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
 return configurations;
 }
```

其返回一个自动配置类的类名列表，方法调用了loadFactoryNames方法，查看该方法

```
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
 String factoryClassName = factoryClass.getName();
 return (List)loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
 }
```

自动配置器会跟根据传入的factoryClass.getName()到项目系统路径下所有的spring.factories文件中找到相应的key，从而加载里面的类。

**（重点）**其中，最关键的要属@Import(AutoConfigurationImportSelector.class)，借助AutoConfigurationImportSelector，@EnableAutoConfiguration可以帮助SpringBoot应用将所有符合条件(spring.factories)的**bean定义**（如Java Config@Configuration配置）都加载到当前SpringBoot创建并使用的IoC容器。

### SpringFactoriesLoader详解

借助于Spring框架**原有**的一个工具类：**SpringFactoriesLoader**的支持，@EnableAutoConfiguration可以智能的自动配置功效才得以大功告成！

SpringFactoriesLoader属于Spring框架私有的一种扩展方案，其主要功能就是从**指定的配置文件META-INF/spring.factories**加载配置,**加载工厂类**。

SpringFactoriesLoader为Spring工厂加载器，该对象提供了loadFactoryNames方法，入参为factoryClass和classLoader即需要传入**工厂类**名称和对应的类加载器，方法会根据指定的classLoader，加载该类加器搜索路径下的指定文件，即spring.factories文件；

传入的工厂类为接口，而文件中对应的类则是接口的实现类，或最终作为实现类。

```
public abstract class SpringFactoriesLoader {
//...
　　public static <T> List<T> loadFactories(Class<T> factoryClass, ClassLoader classLoader) {
　　　　...
　　}
   
   
　　public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
　　　　....
　　}
}
```

配合@EnableAutoConfiguration使用的话，它更多是提供一种配置查找的功能支持，即根据@EnableAutoConfiguration的完整类名org.springframework.boot.autoconfigure.EnableAutoConfiguration作为查找的Key,获取对应的一组@Configuration类　　

上图就是从SpringBoot的autoconfigure依赖包中的META-INF/spring.factories配置文件中摘录的一段内容，可以很好地说明问题。

（重点）所以，@EnableAutoConfiguration自动配置的魔法其实就变成了：

从classpath中搜寻所有的META-INF/spring.factories配置文件，并将其中org.springframework.boot.autoconfigure.EnableAutoConfiguration对应的**配置项**通过**反射（Java Refletion）**实例化为对应的标注了**@Configuration**的JavaConfig形式的IoC容器配置类，然后汇总为一个并加载到IoC容器。

### 自动配置奥秘

`@EnableAutoConfiguration`借助SpringFactoriesLoader可以将标注了`@Configuration`这个注解的JavaConfig类一并汇总并加载到最终的ApplicationContext，这么说只是很简单的解释，其实基于`@EnableAutoConfiguration`的自动配置功能拥有非常强大的调控能力。比如我们可以通过配合基于条件的配置能力或定制化加载顺序，对自动化配置进行更加细粒度的调整和控制。

#### 基于条件的自动配置

这个基于条件的自动配置来源于Spring框架中的"基于条件的配置"特性。在Spring框架中，我们可以使用`@Conditional`这个注解配合`@Configuration`或`@Bean`等注解来干预一个配置或bean定义是否能够生效，它最终实现的效果或者语义类如下伪代码：

```
if (复合@Conditional规定的条件) {
    加载当前配置（Enable Current Configuration）或者注册当前bean定义;
}
```

要实现基于条件的配置，我们需要通过`@Conditional`注解指定自己Condition实现类就可以了(可以应用于类型Type的注解或者方法Method的注解)

```
@Conditional({DemoCondition1.class, DemoCondition2.class})
```

最重要的是，`@Conditional`注解可以作为一个Meta Annotaion用来标注其他注解实现类，从而构建各种复合注解，比如SpringBoot的autoconfigre模块就基于这一优良的革命传统，实现了一批这样的注解(在org.springframework.boot.autoconfigure.condition包下):

- @ConditionalOnClass
- @ConditionalOnBean
- @CondtionalOnMissingClass
- @CondtionalOnMissingBean
- @CondtionalOnProperty
- ……

有了这些复合Annotation的配合，我们就可以结合@EnableAutoConfiguration实现基于条件的自动配置了。其实说白了，SpringBoot能够如此的盛行，很重要的一部分就是它默认提供了一系列自动配置的依赖模块，而这些依赖模块都是基于以上的@Conditional复合注解实现的，这也就说明这些所有的依赖模块都是按需加载的，只有复合某些特定的条件，这些依赖模块才会生效，这也解释了为什么自动配置是“智能”的。

#### 定制化自动配置的顺序

在实现自动配置的过程中，我们除了可以提供基于条件的配置之外，我们还能对当前要提供的配置或组件的加载顺序进行个性化调整，以便让这些配置或者组件之间的依赖分析和组装能够顺利完成。

最经典的是我们可以通过使用`@org.springframework.boot.autoconfigure.AutoConfigureBefore`或者`@org.springframework.boot.autoconfigure.AutoConfigureAfter`让当前配置或者组件在某个其他组件之前或者之后进行配置。例如，假如我们希望某些JMX操作相关的bean定义在MBeanServer配置完成以后在进行配置，那我们就可以提供如下配置：

```
@Configuration
@AutoConfigureAfter(JmxAutoConfiguration.class)
public class AfterMBeanServerReadyConfiguration {
    @Autowired
    MBeanServer mBeanServer;

    // 通过@Bean添加其他必要的bean定义
}
```
现在我们结合getAutoConfigurationEntry()的源码来详细分析一下：
```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
    //第1步：判断自动装配开关是否打开
   if (!isEnabled(annotationMetadata)) {
      return EMPTY_ENTRY;
   }
    //第2步：用于获取注解中的exclude和excludeName。
    //获取注解属性
   AnnotationAttributes attributes = getAttributes(annotationMetadata); 
    //第3步：获取需要自动装配的所有配置类，读取META-INF/spring.factories
    //读取所有预配置类
   List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    //第4步：符合条件加载
    //去掉重复的配置类
   configurations = removeDuplicates(configurations);
    //执行
   Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    //校验
   checkExcludedClasses(configurations, exclusions);
    //删除
   configurations.removeAll(exclusions);
    //过滤
   configurations = getConfigurationClassFilter().filter(configurations);
   fireAutoConfigurationImportEvents(configurations, exclusions);
    //创建自动配置的对象
   return new AutoConfigurationEntry(configurations, exclusions);
}
```

