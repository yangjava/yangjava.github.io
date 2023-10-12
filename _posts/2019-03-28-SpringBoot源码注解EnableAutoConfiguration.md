---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot源码注解@EnableAutoConfiguration
Spring boot如何使用通过EnableAutoConfiguration注解将Bean自动注入到Spring容器中。

## @EnableAutoConfiguration概述
简单点说就是Spring Boot根据依赖中的jar包，自动选择实例化某些配置，配置类必须有@Configuration注解。

@EnableAutoConfiguration：是一个加载Starter目录包之外的需要Spring自动生成bean对象（是否需要的依据是"META-INF/spring.factories"中org.springframework.boot.autoconfigure.EnableAutoConfiguration后面是有能找到那个bean）的带有@Configuration注解的类。一般就是对各种引入的spring-boot-starter依赖包指定的（spring.factories）类进行实例化。

## @EnableAutoConfiguration源码
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










































