---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot源码环境搭建
Spring Boot是由Pivotal团队提供的全新框架，其设计目的是用来简化新Spring应用的初始搭建以及开发过程。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。

## Springboot 源码

源码地址[https://github.com/spring-projects/spring-boot](https://github.com/spring-projects/spring-boot)
获取SpringBoot源码，获取版本为2.5.X的源码。

## Springboot 目录结构

- spring-boot-project：Spring Boot核心项目代码，包含核心、工具、安全、文档、starters等项目。
- spring-boot-tests：Spring Boot部署及集成的测试。

spring-boot-project目录是在Spring Boot 2.0版本发布后新增的目录层级，该模块包含了Spring Boot所有的核心功能。

| 名称                                  | 说明                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| spring-boot	                            | Spring Boot核心代码，也是入口类SpringApplication类所在项目，是本书重点介绍的内容。 |
| spring-boot-actuator                      | 提供应用程序的监控、统计、管理及自定义等相关功能。     |
| spring-boot-actuator-autoconfigure	    | Spring Boot自动配置核心功能，默认集成了多种常见框架的自动配置类等。                                      |
| spring-boot-cli	                | 命令工具，提供快速搭建项目原型、启动服务、执行Groovy脚本等功能。                                          |
| spring-boot-dependencies	            | 依赖和插件的版本信息。         |
| spring-boot-docs          | 参考文档相关内容。                                         |
| spring-boot-parent	            | spring-boot-dependencies的子模块，是其他项目的父模块。|
| spring-boot-properties-migrator		            | Spring Boot 2.0版本新增的模块，支持升级版本配置属性的迁移|
| spring-boot-starters		        | Spring Boot以预定义的方式集成了其他应用的starter集合。|
| spring-boot-test		            | 测试功能相关代码。|
| spring-boot-test-autoconfigure		        | 测试功能自动配置相关代码。|
| spring-boot-tools		            | Spring Boot工具支持模块，包含Ant、Maven、Gradle等构建工具。|


### spring-boot 模块
spring-boot 模块，Spring Boot 的核心实现
- 在 org.springframework.boot.SpringApplication 类，提供了大量的静态方法，可以很容易运行一个独立的 Spring 应用程序。
- 在 org.springframework.boot.web 包下实现，带有可选容器的嵌入式 Web 应用程序（Tomcat、Jetty、Undertow） 的支持。

### spring-boot-autoconfigure 模块
spring-boot-autoconfigure 可以根据类路径的内容，自动配置大部分常用应用程序。通过使用 org.springframework.boot.autoconfigure.@EnableAutoConfiguration 注解，会触发 Spring 上下文的自动配置。

### spring-boot-actuator 模块
spring-boot-actuator 模块。正如其模块的英文 actuator ，它完全是一个用于暴露应用自身信息的模块
- 提供了一个监控和管理生产环境的模块，可以使用 http、jmx、ssh、telnet 等管理和监控应用。
- 审计（Auditing）、 健康（health）、数据采集（metrics gathering）会自动加入到应用里面。

### spring-boot-actuator-autoconfigure 模块
它提供了 spring-boot-actuator 的自动配置功能。

### spring-boot-starters 模块
spring-boot-starters 模块，它不存在任何的代码，而是提供我们常用框架的 Starter 模块。例如：

- spring-boot-starter-web 模块，提供了对 Spring MVC 的 Starter 模块。
- spring-boot-starter-data-jpa 模块，提供了对 Spring Data JPA 的 Starter 模块。
而每个 Starter 模块，里面只存在一个 pom 文件，这是为什么呢？简单来说，Spring Boot 可以根据项目中是否存在指定类，并且是否未生成对应的 Bean 对象，那么就自动创建 Bean 对象。因为有这样的机制，我们只需要使用 pom 文件，配置需要引入的框架，就可以实现该框架的使用所需要的类的自动装配。

## 设计理念与目标

Spring所拥有的强大功能之一就是可以集成各种开源软件。Spring Boot本身并不提供Spring的核心功能，而是作为Spring的脚手架框架，以达到快速构建项目、预置三方配置、开箱即用的目的。

### 设计理念

约定优于配置（Convention Over Configuration），又称为按约定编程，是一种软件设计范式，旨在减少软件开发人员需要做决定的数量，执行起来简单而又不失灵活。

Spring Boot的功能从细节到整体都是基于“约定优于配置”开发的，从基础框架的搭建、配置文件、中间件的集成、内置容器以及其生态中各种Starters，无不遵从此设计范式。Starter作为Spring Boot的核心功能之一，基于自动配置代码提供了自动配置模块及依赖，让软件集成变得简单、易用。与此同时，Spring Boot也在鼓励各方软件组织创建自己的Starter

### 设计目标

Spring Boot框架的设计理念完美遵从了它所属企业的目标，为平台和开发者带来一种全新的体验：整合成熟技术框架、屏蔽系统复杂性、简化已有技术的使用，从而降低软件的使用门槛，提升软件开发和运维的效率。



## 源码调试

```java
@SpringBootApplication
public class Application { 
   
    public static void main(String[] args) { 
   
        SpringApplication.run(Application.class, args);
    }
}
```