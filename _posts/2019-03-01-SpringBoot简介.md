---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot简介
盛年不重来，一日难再晨。及时当勉励，岁月不待人。  ——陶渊明·《杂诗》
官网介绍：Spring Boot使创建独立的、基于生产级Spring的应用程序变得很容易，您可以“直接运行”这些应用程序。我们对Spring平台和第三方库有自己的见解，这样您就可以轻松入门了。大多数Spring引导应用程序只需要很少的Spring配置。

**Spring Boot makes it easy to create stand-alone, production-grade Spring based Applications that you can "just run".**  
**We take an opinionated view of the Spring platform and third-party libraries so you can get started with minimum fuss. Most Spring Boot applications need minimal Spring configuration.**  

## 简介

Spring Boot 是基于 Spring 框架基础上推出的一个全新的框架, 旨在让开发者可以轻松地创建一个可独立运行的，生产级别的应用程序。利用控制反转的核心特性，并通过依赖注入实现控制反转来实现管理对象生命周期容器化，利用面向切面编程进行声明式的事务管理，整合多种持久化技术管理数据访问，提供大量优秀的Web框架方便开发等等。  
Spring Boot是由Pivotal团队提供的全新框架，其设计目的是用来简化Spring应用初始搭建以及开发过程。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。Spring Boot其实就是一个整合很多可插拔的组件（框架），内嵌了使用工具（比如内嵌了Tomcat、Jetty等），方便开发人员快速搭建和开发的一个框架。  

## 特征

- 搭建项目快，几秒钟就可以搭建完成；
- 让测试变的简单，内置了JUnit、Spring Boot Test等多种测试框架，方便测试；
- Spring Boot让配置变的简单，Spring Boot的核心理念：约定大约配置，约定了某种命名规范，可以不用配置，就可以完成功能开发，比如模型和表名一致就可以不用配置，直接进行CRUD（增删改查）的操作，只有表名和模型不一致的时候，配置名称即可；
- 内嵌容器，省去了配置Tomcat的繁琐；
- 方便监控，使用Spring Boot Actuator组件提供了应用的系统监控，可以查看应用配置的详细信息；

## 由来

在开始了解Spring Boot之前，我们需要先了解一下Spring，因为Spring Boot的诞生和Spring是息息相关的，Spring Boot是Spring发展到一定程度的一个产物，但并不是Spring的替代品，Spring Boot是为了让程序员更好的使用Spring。说到这里可能有些人会迷糊，那到底Spring和Spring Boot有着什么样的联系呢？

### Spring发展史

在开始之前我们先了解一下Spring，Spring的前身是interface21，这个框架最初是为了解决EJB开发笨重臃肿的问题，为J2EE提供了另一种简单又实用的解决方案，并在2004年3月发布了Spring 1.0正式版之后，就引起了Java界广泛的关注和热评，从此Spring在Java界势如破竹迅速走红，一路成为Java界一颗璀璨夺目的明星，至今无可替代，也一度成为J2EE开发中真正意义上的标准了，而他的创始人Rod Johnson也在之后声名大噪，名利双收，现在是一名优秀的天使投资人，走上了人生的巅峰。

### Spring Boot诞生

那既然Spring已经这么优秀了，为什么还有了之后Spring Boot？  

因为随着Spring发展的越来越火，Spring也慢慢从一个小而精的框架变成了，一个覆盖面广大而全的框架，另一方面随着新技术的发展，比如nodejs、golang、Ruby的兴起，让Spring逐渐看着笨重起来，大量繁琐的XML配置和第三方整合配置，让Spring使用者痛苦不已，这个时候急需一个解决方案，来解决这些问题。  

就在这个节骨眼上Spring Boot应运而生，2013年Spring Boot开始研发，2014年4月Spring Boot 1.0正式发布，Spring Boot诞生之初就受到业界的广泛关注，很多个人和企业陆续开始尝试，随着Spring Boot 2.0的发布，又一次把Spring Boot推向了公众的视野，也有越来越多了的中大型企业把Spring Boot使用到正式的生产环境了。值得一提的是Spring官方也把Spring Boot作为首要的推广项目，放到了官网的首位。  

### Spring Boot版本号说明

那版本号后面的英文代表什么含义呢？

具体含义，如下文所示：  

- SNAPSHOT：快照版，表示开发版本，随时可能修改；
- M1（Mn）：M是milestone的缩写，也就是里程碑版本；
- RC1（RCn）：RC是release candidates的缩写，也就是发布预览版；
- Release：正式版，也可能没有任何后缀也表示正式版；

## springBoot核心功能

- 独立运行的spring项目：Spring Boot可以以jar包形式直接运行，如java-jar xxxjar优点是：节省服务器资源
- 内嵌servlet 容器：Spring Boot 可以选择内嵌Tomcat，Jetty，这样我们无须以war包形式部署项目。
- 提供starter 简化Maven 配置：在Spring Boot 项目中为我们提供了很多的spring-boot-starter-xxx的项目（我们把这个依赖可以称之为**起步依赖**），我们导入指定的这些项目的坐标，就会自动导入和该模块相关的依赖包：
  例如我们后期再使用Spring Boot 进行web开发我们就需要导入spring-boot-starter-web这个项目的依赖，导入这个依赖以后！那么Spring Boot就会自动导入web开发所需要的其他的依赖包
- 自动配置 spring:Spring Boot 会根据在类路径中的jar包，类，为jar包里的类自动配置Bean，这样会极大减少我们要使用的配置。
  当然Spring Boot只考虑了大部分开发场景，并不是所有的场景，如果在实际的开发中我们需要自动配置Bean，而Spring Boot不能满足，则可以自定义自动配置。
- 准生产的应用监控：Spring Boot 提供基于http，sh，telnet对运行时的项目进行监控
- 无代码生成和xml配置：Spring Boot大量使用spring4.x提供的注解新特性来实现无代码生成和xml 配置。spring4.x提倡使用Java配置和注解配置组合，而Spring Boot不需要任何xml配置即可实现spring的所有配置。


## 参考文章  
SpringBoot官网 [https://spring.io/projects/spring-boot](https://spring.io/projects/spring-boot)
