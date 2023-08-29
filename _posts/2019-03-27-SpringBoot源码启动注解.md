---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot启动注解
详细理解SpringBoot的组合注解 @SpringBootApplication。

## 启动类注解
SpringBoot的启动类入口如下所示：

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

在上述的 `Spring Boot` 入口类 `Application` 中，唯一的注解就是 `@SpringBootApplication`。下面看下 `@SpringBootApplication` 注解详情。

## @SpringBootApplication 介绍
先来看 @SpringBootApplication 的源码
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

    /**
     * 排除特定的自动配置类，使它们永远不会被应用 要排除（禁用）的类
     */
    @AliasFor(annotation = EnableAutoConfiguration.class)
    Class<?>[] exclude() default {};

    /**
     * 排除特定的自动配置类名称，以确保它们永远不会被应用 要排除的自动配置类名称
     */
    @AliasFor(annotation = EnableAutoConfiguration.class)
    String[] excludeName() default {};

    /**
     * 用于扫描带注解组件的基础包。使用 scanBasePackageClasses 可以使用类型安全的方式替代基于字符串的包名。
     * 注意：该设置仅对 @ComponentScan注解有效，不影响 @Entity扫描或 Spring Data 的 Repository扫描。
     * 对于这些情况，你应该添加@EntityScan和Repositories注解。
     */
    @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
    String[] scanBasePackages() default {};

    /**
     * 用于指定要扫描带注解组件的包的类型安全方式。将扫描指定类所在的包。
     * 考虑在每个包中创建一个特殊的空类或接口，只用于作为此属性引用的标记类。
     */
    @AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
    Class<?>[] scanBasePackageClasses() default {};

    /**
     * 用于在 Spring 容器中为检测到的组件命名的 BeanNameGenerator 类。
     */
    @AliasFor(annotation = ComponentScan.class, attribute = "nameGenerator")
    Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

    /**
     * 指定是否应代理 @Bean 方法以强制执行 bean 的生命周期行为，例如，即使在用户代码中直接调用 @Bean方法，
     * 也能返回共享的单例 bean 实例。此功能需要方法拦截，通过运行时生成的 CGLIB 子类来实现，其中包括一些限制，例如配置类及其方法不允许声明为 final。
     * 默认值为 {@code true}，允许在配置类内部进行 'inter-bean references'，同时允许从另一个配置类中调用此配置的 {@code @Bean} 方法。
     */
    @AliasFor(annotation = Configuration.class)
    boolean proxyBeanMethods() default true;

}
```
通过查看上述源码，我们可以看到 @SpringBootApplication 注解提供如下的成员属性【这里默认大家都是知道 注解中的成员变量是以方法的形式存在的】：

- exclude
根据类（Class）排除指定的自动配置，该成员属性覆盖了 @SpringBootApplication 中组合的 @EnableAutoConfiguration 中定义的 exclude 成员属性。

- excludeName 
根据类名排除指定的自动配置，覆盖了 @EnableAutoConfiguration 中定义的 excludeName 成员属性。

- scanBasePackages 
指定扫描的基础 package，用于扫描带注解组件的基础包，例如包含 @Component 等注解的组件。

- scanBasePackageClasses 
指定扫描的类，用于相关组件的初始化。

- nameGenerator
用于在 Spring 容器中为检测到的组件命名的 BeanNameGenerator 类。

- proxyBeanMethods 
指定是否代码 @Bean 方法以强制执行 bean 的生命周期行为。该功能需要通过运行时生成 CGLIB 子类来实现方法拦截。不过它包括一定的限制，例如配置类及其方法不允许声明为 final 等。proxyBeanMethods 的默认值为 true，允许配置类中进行 inter-bean references（bean 之间的引用）以及对该配置的 @Bean 方法的外部调用。如果 @Bean 方法都是自包含的，并且仅提供了容器使用的普通工程方法的功能，则可设置为 false，避免处理 CGLIB 子类。另外我们从源码中 @since 2.2 处也可以看出来，该属性是在 Spring Boot 2.2 版本新增的。

细心的读者，可能看过上面的源码会发现，@SpringBootApplication 注解的成员属性上大量使用了 @AliasFor 注解，那该注解有什么作用呢？

## @AliasFor介绍
@AliasFor 注解用于桥接其他注解，它的 annotation 属性中指定了所桥接的注解类。如果我们点到 annotation 属性配置的注解中，可以看出 @SpringBootApplication 注解的成员属性其实已经在其他注解中定义过了。

那之所以使用 @AliasFor 注解并重新在 @SpringBootApplication 中定义，主要就是为了减少用户在使用多注解上的麻烦。

### @AliasFor 的作用
简单总结一下 @AliasFor 的作用：
- 定义别名关系
通过在注解属性上使用 @AliasFor 注解，可以将一个属性与另一个属性建立别名关系。这意味着当使用注解时，你可以使用别名属性来设置目标属性的值。
- 属性互通
通过在两个属性上使用 @AliasFor 注解，并且将它们的 attribute 属性分别设置为对方，可以实现属性之间的双向关联。这意味着当设置其中一个属性的值时，另一个属性也会自动被赋予相同的值。
- 注解继承
当一个注解 A 使用 @AliasFor 注解指定了另一个注解 B 的属性为自己的别名属性时，如果类使用了注解 A，那么注解 B 的相关属性也会得到相应的设置。

## @SpringBootConfiguration
在 Spring Boot 早期的版本中并没有 @SpringBootConfiguration 注解，它是后面的版本新加的，其内组合了 @Configuration 注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
@Indexed
public @interface SpringBootConfiguration {

	/**
	 */
	@AliasFor(annotation = Configuration.class)
	boolean proxyBeanMethods() default true;

}
```
在springboot中被@Configuration或者@SpringBootConfiguration标注的类称之为配置类。

给一个类上标注@Configuration注解，就是为了告诉Spring Boot该类是一个配置类，然后，我们就可以在该配置类里面使用@Bean注解标注在方法上向容器中注册组件了，而且，注册的组件默认还是单实例的哟！

值得注意的是@Configuration注解标注的类本身也是一个组件哟，也就是说配置类本身也是容器中的一个组件。

## @EnableAutoConfiguration
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	/**
     * 可以用于覆盖自动配置是否启用的环境属性
	 */
	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	/**
     * 排除特定的自动配置类，以使它们永远不会应用
	 */
	Class<?>[] exclude() default {};

	/**
     * 排除特定的自动配置类名，以使它们永远不会应用
	 */
	String[] excludeName() default {};
}
```
在没有使用 Spring Boot 的情况下，Bean 的生命周期都是由 Spring 来管理的，并且 Spring 是无法自动配置 @Configuration 注解的类。Spring Boot 的核心功能之一就是根据约定自动管理 @Configuration 注解的类 ，其中 @EnableAutoConfiguration 注解就是实现该功能的组件之一。

@EnableAutoConfiguration 注解 位于 spring-boot-autoconfigure 包内，当使用 @SpringBootApplication 注解时，它也就会自动生效。

通过查看源码，我们可以看到 @EnableAutoConfiguration 注解提供了一个常量 和 两个成员变量：

- ENABLED_OVERRIDE_PROPERTY : 用于覆盖自动配置是否启用的环境属性
- exclude ：排除特定的自动配置类
- excludeName ：排除特定的自动配置类名

正如前面所说， @EnableAutoConfiguration 会尝试猜测并配置你可能需要的 Bean，但实际情况如果是我们不需要这些预配置的 Bean，那么也可以通过它的两个成员变量 exclude 和 excludeName 来排除指定的自动配置。
```
// 通过 @SpringBootApplication 排除 DataSourceAutoConfiguration
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class DemoApplication {

}
```
或者：
```
// 通过 @EnableAutoConfiguration 排除 DataSourceAutoConfiguration
@Configuration
@EnableAutoConfiguration(exclude = DataSourceAutoConfiguration.class)
public class DemoConfiguration {
}
```

Spring Boot 在进行实体类扫描时，会从 @EnableAutoConfiguration 注解标注的类所在的包开始扫描。这也是在使用 @SpringBootApplication 注解时需要将被注解的类放在顶级 package 下的原因，如果放在较低层级，它所在 package 的同级或上级中的类就无法被扫描到，从而无法正常使用相关注解（如 @Entity）。

在 Spring Boot 应用程序中，入口类只是一个用来引导应用程序的类，而真正的自动配置和功能开启是通过 @SpringBootApplication 和 @EnableAutoConfiguration 注解所用的其他类完成的。








