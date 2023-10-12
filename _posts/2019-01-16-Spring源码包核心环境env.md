---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring源码包核心环境env
Spring在创建容器时，会创建Environment环境对象，用于保存spring应用程序的运行环境相关的信息。在创建环境时，需要创建属性源属性解析器，会解析属性值中的占位符，并进行替换。

创建环境时，会通过System.getProperties()获取JVM系统属性，会通过System.getenv()获取JVM环境属性。

## environment概述
environment 翻译过来即是“环境”，表示应用程序运行环境。Environment表示当前Spring程序运行的环境，主要管理profiles和properties两种信息。

Environment=profile(配置)+propertyResolver(资源解析器)；
- profiles用来区分当前是dev(开发)环境还是test(测试)环境或者prod(生产)环境。
- properties表示所有的属性，包括操作系统环境变量，如PATH，JDK相关配置，如java.vm.specification.version(JDK版本)，还有我们通过properties文件和yml文件自定义的属性。

Spring 应用程序运行环境概念，主要包含两个方面的内容。profiles 决定了哪些 Bean 会被加载；properties 就是配置，就是键值对

和Environment相关的接口可以有那么多:
```
PropertyResolver, 
EnvironmentCapable, 
ConfigurableEnvironment, 
AbstractEnvironment, 
StandardEnvironment, 
org.springframework.context.EnvironmentAware, 
org.springframework.context.ConfigurableApplicationContext.getEnvironment, 
org.springframework.context.ConfigurableApplicationContext.setEnvironment, 
org.springframework.context.support.AbstractApplicationContext.createEnvironment
```

### PropertyResolver
PropertyResolver接口定义了按属性名获取对应属性配置的接口以及解析字符串中的属性表达式的接口
```

```