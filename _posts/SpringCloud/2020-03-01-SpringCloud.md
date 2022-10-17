# Spring Cloud

  * [1 Spring Cloud概述](#1-spring-cloud%E6%A6%82%E8%BF%B0)
    * [1\.1 微服务](#11-%E5%BE%AE%E6%9C%8D%E5%8A%A1)
    * [1\.2 Spring Cloud基本概念](#12-spring-cloud%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5)
    * [1\.3 Spring Cloud核心组件](#13-spring-cloud%E6%A0%B8%E5%BF%83%E7%BB%84%E4%BB%B6)
    * [1\.4 Spring Cloud其他组件](#14-spring-cloud%E5%85%B6%E4%BB%96%E7%BB%84%E4%BB%B6)
    * [1\.5 与Spring Boot的关系](#15-%E4%B8%8Espring-boot%E7%9A%84%E5%85%B3%E7%B3%BB)
  * [2 注册中心](#2-%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83)
    * [2\.1 Eureka](#21-eureka)
    * [2\.1\.1 基本概念](#211-%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5)
  * [3 服务调用](#3-%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8)
  * [4 熔断器Hystrix](#4-%E7%86%94%E6%96%AD%E5%99%A8hystrix)
    * [4\.1 Hystrix特性](#41-hystrix%E7%89%B9%E6%80%A7)
  * [5 熔断器监控Hystrix Dashboard和Turbine](#5-%E7%86%94%E6%96%AD%E5%99%A8%E7%9B%91%E6%8E%A7hystrix-dashboard%E5%92%8Cturbine)
    * [5\.1 Hystrix Dashboard](#51-hystrix-dashboard)

## 1 Spring Cloud概述

### 1.1 微服务

微服务是可以独立部署、水平扩展、独立访问（或者有独立的数据库）的服务单元。

### 1.2 Spring Cloud基本概念

Spring Cloud 是一系列框架的有序集合。

Spring Cloud 利用 Spring Boot 的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等，都可以用 Spring Boot 的开发风格做到一键启动和部署。

它将目前各家公司开发的比较成熟、经得起实际考验的服务框架组合起来，通过 Spring Boot 风格进行再封装屏蔽掉了复杂的配置和实现原理，最终给开发者留出了一套简单易懂、易部署和易维护的分布式系统开发工具包。

### 1.3 Spring Cloud核心组件

- **Spring Cloud Netflix**

​	重要组件之一，与各种Netflix OSS组件集成，组成微服务的核心。

- **Netflix Eureka**

  服务注册中心，云端服务发现，一个基于 REST 的服务，用于定位服务，以实现云端中间层服务发现和故障转移。

- **Netflix Hystrix**

  熔断器，容错管理工具，旨在通过熔断机制控制服务和第三方库的节点,从而对延迟和故障提供更强大的容错能力。

- **Netflix Zuul**

  Zuul 是在云平台上提供动态路由,监控,弹性,安全等边缘服务的框架。

- **Netflix Archaius**

  配置管理API，包含一系列配置管理API，提供动态类型化属性、线程安全配置操作、轮询框架、回调机制等功能。（可实现动态获取配置， 原理是每隔60s（默认，可配置）从配置源读取一次内容，这样修改了配置文件后不需要重启服务就可以使修改后的内容生效，前提使用archaius的API来读取。）

- **Spring Cloud Config**

​	配置中心，配置管理工具包，可以把配置放到远程服务器，集中化管理集群配置，目前支持本地存储、Git 以及 Svn。

- **Spring Cloud Bus**

​	事件、消息总线，用于在集群（例如，配置变化事件）中传播状态变化，可与Spring Cloud Config联合实现热部署。

- **Spring Cloud for Cloud Foundry**

​	Cloud Foundry是VMware推出的业界第一个开源PaaS云平台，它支持多种框架、语言、运行时环境、云平台及应用服务，使开发人员能够在几秒钟内进行应用程序的部署和扩展，无需担心任何基础架构的问题。

- **Spring Cloud Cluster**

​	Spring Cloud Cluster将取代Spring Integration。提供在分布式系统中的集群所需要的基础功能支持，如：选举、集群的状态一致性、全局锁、tokens等常见状态模式的抽象和实现。

- **Spring Cloud Consul**

​	Spring Cloud Consul 封装了Consul操作，consul是一个服务发现与配置工具，与Docker容器可以无缝集成。（Consul 是一个支持多数据中心分布式高可用的服务发现和配置共享的服务软件,由 HashiCorp 公司用 Go 语言开发, 基于 Mozilla Public License 2.0 的协议进行开源，Consul 支持健康检查,并允许 HTTP 和 DNS 协议调用 API 存储键值对。）

### 1.4 Spring Cloud其他组件

- **Spring Cloud Security**

​	基于 Spring security 的安全工具包，为应用程序添加安全控制。

- **Spring Cloud Sleuth**

​	日志收集工具包，封装了 Dapper 和 log-based 追踪以及 Zipkin 和 HTrace 操作，为 Spring Cloud 应用实现了一种分布式追踪解决方案。

- **Spring Cloud Data Flow**
  - 用于开发和执行大范围数据处理其模式包括ETL，批量运算和持续运算的统一编程模型和托管服务。
  - 是一个原生云可编配的服务。使用Spring Cloud data flow，开发者可以为像数据抽取，实时分析，和数据导入/导出这种常见用例创建和编配数据通道 （data pipelines）。
  - Spring Cloud data flow 是基于原生云对 spring XD的重新设计，该项目目标是简化大数据应用的开发。
  - Spring Cloud data flow 为基于微服务的分布式流处理和批处理数据通道提供了一系列模型和最佳实践。

- **Spring Cloud Stream**

​	Spring Cloud Stream是创建消息驱动微服务应用的框架。用来建立单独的／工业级spring应用，使用spring integration提供与消息代理之间的连接。数据流操作开发包，封装了与Redis,Rabbit、Kafka等发送接收消息。

- **Spring Cloud Task**

​	Spring Cloud Task 主要解决短命微服务的任务管理，任务调度的工作。

- **Spring Cloud Zookeeper**

​	操作Zookeeper的工具包，用于使用zookeeper方式进行服务发现和配置管理。

- **Spring Cloud Connectors**

​	简化了连接到服务的过程和从云平台获取操作的过程，有很强的扩展性，可利用Spring Cloud Connectors来构建自己的云平台。便于云端应用程序在各种PaaS平台连接到后端，如：数据库和消息代理服务。

- **Spring Cloud Starters**

​	为Spring Cloud提供开箱即用的依赖管理。

- **Spring Cloud CLI**

​	基于 Spring Boot CLI，可以以命令行方式快速建立云组件。

### 1.5 与Spring Boot的关系

spring -> spring Boot > Spring Cloud（->表示依赖关系）

- Spring Boot 是 Spring 的一套快速配置脚手架，可以基于 Spring Boot 快速开发单个微服务，Spring Cloud 是一个基于 Spring Boot 实现的云应用开发工具；
- Spring Boot 专注于快速、方便集成的单个个体，Spring Cloud是关注全局的服务治理框架；
- Spring Boot 使用了默认大于配置的理念，很多集成方案已经选择好了，能不配置就不配置，Spring Cloud很大的一部分是基于 Spring Boot 来实现,
- Spring Boot 可以离开 Spring Cloud 独立使用开发项目，但是 Spring Cloud 离不开 Spring Boot。他们属于依赖的关系。

## 2 注册中心

### 2.1 Eureka

### 2.1.1 基本概念

**1. 概念**

Eureka 是 Netflix 开源的一款提供服务注册和发现的产品，它提供了完整的 Service Registry 和 Service Discovery 实现。也是 springcloud 体系中最重要最核心的组件之一。

> 服务中心又称注册中心，管理各种服务功能包括服务的注册、发现、熔断、负载、降级等。

**2. 组成**

​	Eureka由两个组件组成：Eureka服务器和Eureka客户端。

​	Eureka服务器用作服务注册服务器。Eureka客户端是一个java客户端，用来简化与服务器的交互、作为轮询负载均衡器，并提供服务的故障切换支持。

​	上图简要描述了Eureka的基本架构，由3个角色组成：

- Eureka Server

  提供服务注册和发现

- Service Provider

  服务提供方：将自身服务注册到Eureka，从而使服务消费方能够找到

- Service Consumer

  服务消费方：从Eureka获取注册服务列表，从而能够消费服务

​	**注册中心必须集群搭建，否则遇到故障对整个生产环境的打击是毁灭性的。**

​	生产环境中需要三台或者大于三台的注册中心来保证服务的稳定性，配置的原理是将当前注册中心分别指向其它的所有的注册中心。

## 3 服务调用

## 4 熔断器Hystrix

​	熔断器（CircuitBreaker）：实现快速失败。若它在一段时间内侦测到许多类似的错误，会强迫其以后的多个调用快速失败，不再访问远程服务器，从而防止应用程序不断地尝试执行可能会失败的操作，使得应用程序继续执行而不用等待修正错误，或者浪费CPU时间去等到长时间的超时产生。

​	熔断器也可以使应用程序能够诊断错误是否已经修正，如果已经修正，应用程序会再次尝试调用操作。

​	**熔断器就是保护服务高可用的最后一道防线。**

### 4.1 Hystrix特性

​	**1. 断路器机制**

​	当Hystrix Command请求后端服务失败数量超过一定比例(默认50%), 断路器会切换到开路状态(Open). 这时所有请求会直接失败而不会发送到后端服务. 断路器保持在开路状态一段时间后(默认5秒), 自动切换到半开路状态(HALF-OPEN). 这时会判断下一次请求的返回情况, 如果请求成功, 断路器切回闭路状态(CLOSED), 否则重新切换到开路状态(OPEN)。

​	一旦后端服务不可用, 断路器会直接切断请求链, 避免发送大量无效请求影响系统吞吐量, 并且断路器有自我检测并恢复的能力。

​	**2. Fallback**

​	Fallback相当于是降级操作。对于查询操作, 可实现一个fallback方法, 当请求后端服务出现异常的时候, 使用fallback方法返回的值。

​	fallback方法的返回值一般是设置的默认值或者来自缓存。

​	**3. 资源隔离**

​	在Hystrix中, 主要通过线程池来实现资源隔离。

​	通常在使用的时候我们会根据调用的远程服务划分出多个线程池。 例如调用产品服务的Command放入A线程池, 调用账户服务的Command放入B线程池， 这样做的主要优点是运行环境被隔离开了，这样就算调用服务的代码存在bug或者由于其他原因导致自己所在线程池被耗尽时，不会对系统的其他服务造成影响。但是带来的代价就是维护多个线程池会对系统带来额外的性能开销。如果是对性能有严格要求而且确信自己调用服务的客户端代码不会出问题的话, 可以使用Hystrix的信号模式(Semaphores)来隔离资源。

## 5 熔断器监控Hystrix Dashboard和Turbine

​	Hystrix-dashboard是一款针对Hystrix进行实时监控的工具，通过Hystrix Dashboard我们可以在直观地看到各Hystrix Command的请求响应时间, 请求成功率等数据。但是Hystrix Dashboard只能看到单个应用内的服务信息。

​	Turbine能汇总系统内多个服务的数据并显示到Hystrix Dashboard上。

### 5.1 Hystrix Dashboard





zuul的生命周期

![1582987858514](assets/1582987858514.png)

# Spring Cloud

  * [1 Spring Cloud概述](#1-spring-cloud%E6%A6%82%E8%BF%B0)
    * [1\.1 微服务](#11-%E5%BE%AE%E6%9C%8D%E5%8A%A1)
    * [1\.2 Spring Cloud基本概念](#12-spring-cloud%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5)
    * [1\.3 Spring Cloud核心组件](#13-spring-cloud%E6%A0%B8%E5%BF%83%E7%BB%84%E4%BB%B6)
    * [1\.4 Spring Cloud其他组件](#14-spring-cloud%E5%85%B6%E4%BB%96%E7%BB%84%E4%BB%B6)
    * [1\.5 与Spring Boot的关系](#15-%E4%B8%8Espring-boot%E7%9A%84%E5%85%B3%E7%B3%BB)
  * [2 注册中心](#2-%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83)
    * [2\.1 Eureka](#21-eureka)
    * [2\.1\.1 基本概念](#211-%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5)
  * [3 服务调用](#3-%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8)
  * [4 熔断器Hystrix](#4-%E7%86%94%E6%96%AD%E5%99%A8hystrix)
    * [4\.1 Hystrix特性](#41-hystrix%E7%89%B9%E6%80%A7)
  * [5 熔断器监控Hystrix Dashboard和Turbine](#5-%E7%86%94%E6%96%AD%E5%99%A8%E7%9B%91%E6%8E%A7hystrix-dashboard%E5%92%8Cturbine)
    * [5\.1 Hystrix Dashboard](#51-hystrix-dashboard)

## 1 Spring Cloud概述

### 1.1 微服务

微服务是可以独立部署、水平扩展、独立访问（或者有独立的数据库）的服务单元。

### 1.2 Spring Cloud基本概念

Spring Cloud 是一系列框架的有序集合。

Spring Cloud 利用 Spring Boot 的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等，都可以用 Spring Boot 的开发风格做到一键启动和部署。

它将目前各家公司开发的比较成熟、经得起实际考验的服务框架组合起来，通过 Spring Boot 风格进行再封装屏蔽掉了复杂的配置和实现原理，最终给开发者留出了一套简单易懂、易部署和易维护的分布式系统开发工具包。

### 1.3 Spring Cloud核心组件

- **Spring Cloud Netflix**

​	重要组件之一，与各种Netflix OSS组件集成，组成微服务的核心。

- **Netflix Eureka**

  服务注册中心，云端服务发现，一个基于 REST 的服务，用于定位服务，以实现云端中间层服务发现和故障转移。

- **Netflix Hystrix**

  熔断器，容错管理工具，旨在通过熔断机制控制服务和第三方库的节点,从而对延迟和故障提供更强大的容错能力。

- **Netflix Zuul**

  Zuul 是在云平台上提供动态路由,监控,弹性,安全等边缘服务的框架。

- **Netflix Archaius**

  配置管理API，包含一系列配置管理API，提供动态类型化属性、线程安全配置操作、轮询框架、回调机制等功能。（可实现动态获取配置， 原理是每隔60s（默认，可配置）从配置源读取一次内容，这样修改了配置文件后不需要重启服务就可以使修改后的内容生效，前提使用archaius的API来读取。）

- **Spring Cloud Config**

​	配置中心，配置管理工具包，可以把配置放到远程服务器，集中化管理集群配置，目前支持本地存储、Git 以及 Svn。

- **Spring Cloud Bus**

​	事件、消息总线，用于在集群（例如，配置变化事件）中传播状态变化，可与Spring Cloud Config联合实现热部署。

- **Spring Cloud for Cloud Foundry**

​	Cloud Foundry是VMware推出的业界第一个开源PaaS云平台，它支持多种框架、语言、运行时环境、云平台及应用服务，使开发人员能够在几秒钟内进行应用程序的部署和扩展，无需担心任何基础架构的问题。

- **Spring Cloud Cluster**

​	Spring Cloud Cluster将取代Spring Integration。提供在分布式系统中的集群所需要的基础功能支持，如：选举、集群的状态一致性、全局锁、tokens等常见状态模式的抽象和实现。

- **Spring Cloud Consul**

​	Spring Cloud Consul 封装了Consul操作，consul是一个服务发现与配置工具，与Docker容器可以无缝集成。（Consul 是一个支持多数据中心分布式高可用的服务发现和配置共享的服务软件,由 HashiCorp 公司用 Go 语言开发, 基于 Mozilla Public License 2.0 的协议进行开源，Consul 支持健康检查,并允许 HTTP 和 DNS 协议调用 API 存储键值对。）

### 1.4 Spring Cloud其他组件

- **Spring Cloud Security**

​	基于 Spring security 的安全工具包，为应用程序添加安全控制。

- **Spring Cloud Sleuth**

​	日志收集工具包，封装了 Dapper 和 log-based 追踪以及 Zipkin 和 HTrace 操作，为 Spring Cloud 应用实现了一种分布式追踪解决方案。

- **Spring Cloud Data Flow**
  - 用于开发和执行大范围数据处理其模式包括ETL，批量运算和持续运算的统一编程模型和托管服务。
  - 是一个原生云可编配的服务。使用Spring Cloud data flow，开发者可以为像数据抽取，实时分析，和数据导入/导出这种常见用例创建和编配数据通道 （data pipelines）。
  - Spring Cloud data flow 是基于原生云对 spring XD的重新设计，该项目目标是简化大数据应用的开发。
  - Spring Cloud data flow 为基于微服务的分布式流处理和批处理数据通道提供了一系列模型和最佳实践。

- **Spring Cloud Stream**

​	Spring Cloud Stream是创建消息驱动微服务应用的框架。用来建立单独的／工业级spring应用，使用spring integration提供与消息代理之间的连接。数据流操作开发包，封装了与Redis,Rabbit、Kafka等发送接收消息。

- **Spring Cloud Task**

​	Spring Cloud Task 主要解决短命微服务的任务管理，任务调度的工作。

- **Spring Cloud Zookeeper**

​	操作Zookeeper的工具包，用于使用zookeeper方式进行服务发现和配置管理。

- **Spring Cloud Connectors**

​	简化了连接到服务的过程和从云平台获取操作的过程，有很强的扩展性，可利用Spring Cloud Connectors来构建自己的云平台。便于云端应用程序在各种PaaS平台连接到后端，如：数据库和消息代理服务。

- **Spring Cloud Starters**

​	为Spring Cloud提供开箱即用的依赖管理。

- **Spring Cloud CLI**

​	基于 Spring Boot CLI，可以以命令行方式快速建立云组件。

### 1.5 与Spring Boot的关系

spring -> spring Boot > Spring Cloud（->表示依赖关系）

- Spring Boot 是 Spring 的一套快速配置脚手架，可以基于 Spring Boot 快速开发单个微服务，Spring Cloud 是一个基于 Spring Boot 实现的云应用开发工具；
- Spring Boot 专注于快速、方便集成的单个个体，Spring Cloud是关注全局的服务治理框架；
- Spring Boot 使用了默认大于配置的理念，很多集成方案已经选择好了，能不配置就不配置，Spring Cloud很大的一部分是基于 Spring Boot 来实现,
- Spring Boot 可以离开 Spring Cloud 独立使用开发项目，但是 Spring Cloud 离不开 Spring Boot。他们属于依赖的关系。

## 2 注册中心

### 2.1 Eureka

### 2.1.1 基本概念

**1. 概念**

Eureka 是 Netflix 开源的一款提供服务注册和发现的产品，它提供了完整的 Service Registry 和 Service Discovery 实现。也是 springcloud 体系中最重要最核心的组件之一。

> 服务中心又称注册中心，管理各种服务功能包括服务的注册、发现、熔断、负载、降级等。

**2. 组成**

​	Eureka由两个组件组成：Eureka服务器和Eureka客户端。

​	Eureka服务器用作服务注册服务器。Eureka客户端是一个java客户端，用来简化与服务器的交互、作为轮询负载均衡器，并提供服务的故障切换支持。

​	上图简要描述了Eureka的基本架构，由3个角色组成：

- Eureka Server

  提供服务注册和发现

- Service Provider

  服务提供方：将自身服务注册到Eureka，从而使服务消费方能够找到

- Service Consumer

  服务消费方：从Eureka获取注册服务列表，从而能够消费服务

​	**注册中心必须集群搭建，否则遇到故障对整个生产环境的打击是毁灭性的。**

​	生产环境中需要三台或者大于三台的注册中心来保证服务的稳定性，配置的原理是将当前注册中心分别指向其它的所有的注册中心。

## 3 服务调用

## 4 熔断器Hystrix

​	熔断器（CircuitBreaker）：实现快速失败。若它在一段时间内侦测到许多类似的错误，会强迫其以后的多个调用快速失败，不再访问远程服务器，从而防止应用程序不断地尝试执行可能会失败的操作，使得应用程序继续执行而不用等待修正错误，或者浪费CPU时间去等到长时间的超时产生。

​	熔断器也可以使应用程序能够诊断错误是否已经修正，如果已经修正，应用程序会再次尝试调用操作。

​	**熔断器就是保护服务高可用的最后一道防线。**

### 4.1 Hystrix特性

​	**1. 断路器机制**

​	当Hystrix Command请求后端服务失败数量超过一定比例(默认50%), 断路器会切换到开路状态(Open). 这时所有请求会直接失败而不会发送到后端服务. 断路器保持在开路状态一段时间后(默认5秒), 自动切换到半开路状态(HALF-OPEN). 这时会判断下一次请求的返回情况, 如果请求成功, 断路器切回闭路状态(CLOSED), 否则重新切换到开路状态(OPEN)。

​	一旦后端服务不可用, 断路器会直接切断请求链, 避免发送大量无效请求影响系统吞吐量, 并且断路器有自我检测并恢复的能力。

​	**2. Fallback**

​	Fallback相当于是降级操作。对于查询操作, 可实现一个fallback方法, 当请求后端服务出现异常的时候, 使用fallback方法返回的值。

​	fallback方法的返回值一般是设置的默认值或者来自缓存。

​	**3. 资源隔离**

​	在Hystrix中, 主要通过线程池来实现资源隔离。

​	通常在使用的时候我们会根据调用的远程服务划分出多个线程池。 例如调用产品服务的Command放入A线程池, 调用账户服务的Command放入B线程池， 这样做的主要优点是运行环境被隔离开了，这样就算调用服务的代码存在bug或者由于其他原因导致自己所在线程池被耗尽时，不会对系统的其他服务造成影响。但是带来的代价就是维护多个线程池会对系统带来额外的性能开销。如果是对性能有严格要求而且确信自己调用服务的客户端代码不会出问题的话, 可以使用Hystrix的信号模式(Semaphores)来隔离资源。

## 5 熔断器监控Hystrix Dashboard和Turbine

​	Hystrix-dashboard是一款针对Hystrix进行实时监控的工具，通过Hystrix Dashboard我们可以在直观地看到各Hystrix Command的请求响应时间, 请求成功率等数据。但是Hystrix Dashboard只能看到单个应用内的服务信息。

​	Turbine能汇总系统内多个服务的数据并显示到Hystrix Dashboard上。

### 5.1 Hystrix Dashboard





zuul的生命周期

![1582987858514](F:/work/openGuide/Middleware/Spring/assets/1582987858514.png)

## 服务发现组件之 — Eureka

 发表于 2020-03-14 | 分类于 [Java ](https://www.mghio.cn/categories/Java/)， [服务发现 ](https://www.mghio.cn/categories/Java/服务发现/)， [Eureka ](https://www.mghio.cn/categories/Java/服务发现/Eureka/)| [0 ](https://www.mghio.cn/post/710bd10b.html#comments)| 本文总阅读量 205次

 字数统计: 2.7k 字 | 阅读时长 ≈ 9 分钟

#### 前言

现在流行的微服务体系结构正在改变我们构建应用程序的方式，从单一的单体服务转变为越来越小的可单独部署的服务（称为`微服务`），共同构成了我们的应用程序。当进行一个业务时不可避免就会存在多个服务之间调用，假如一个服务 A 要访问在另一台服务器部署的服务 B，那么前提是服务 A 要知道服务 B 所在机器的 IP 地址和服务对应的端口，最简单的方式就是让服务 A 自己去维护一份服务 B 的配置（包含 IP 地址和端口等信息），但是这种方式有几个明显的缺点：随着我们调用服务数量的增加，配置文件该如何维护；缺乏灵活性，如果服务 B 改变 IP 地址或者端口，服务 A 也要修改相应的文件配置；还有一个就是进行服务的动态扩容或缩小不方便。
一个比较好的解决方案就是 `服务发现（Service Discovery）`。它抽象出来了一个注册中心，当一个新的服务上线时，它会将自己的 IP 和端口注册到注册中心去，会对注册的服务进行定期的心跳检测，当发现服务状态异常时将其从注册中心剔除下线。服务 A 只要从注册中心中获取服务 B 的信息即可，即使当服务 B 的 IP 或者端口变更了，服务 A 也无需修改，从一定程度上解耦了服务。服务发现目前业界有很多开源的实现，比如 `apache` 的 [zookeeper](https://github.com/apache/zookeeper)、 `Netflix` 的 [eureka](https://github.com/Netflix/eureka)、`hashicorp` 的 [consul](https://github.com/hashicorp/consul)、 `CoreOS` 的 [etcd](https://github.com/etcd-io/etcd)。



#### Eureka 是什么

`Eureka` 在 [github](https://github.com/Netflix/eureka) 上对其的定义为

> Eureka is a REST (Representational State Transfer) based service that is primarily used in the AWS cloud for locating services for the purpose of load balancing and failover of middle-tier servers.
> At Netflix, Eureka is used for the following purposes apart from playing a critical part in mid-tier load balancing.

`Eureka` 是由 [Netflix](https://www.netflix.com/) 公司开源，采用的是 Client / Server 模式进行设计，基于 http 协议和使用 Restful Api 开发的服务注册与发现组件，提供了完整的服务注册和服务发现，可以和 `Spring Cloud` 无缝集成。其中 Server 端扮演着服务注册中心的角色，主要是为 Client 端提供服务注册和发现等功能，维护着 Client 端的服务注册信息，同时定期心跳检测已注册的服务当不可用时将服务剔除下线，Client 端可以通过 Server 端获取自身所依赖服务的注册信息，从而完成服务间的调用。遗憾的是从其官方的 [github wiki](https://github.com/Netflix/eureka/wik) 可以发现，2.0 版本已经不再开源。但是不影响我们对其进行深入了解，毕竟服务注册、服务发现相对来说还是比较基础和通用的，其它开源实现框架的思想也是想通的。

#### 服务注册中心（Eureka Server）

我们在项目中引入 `Eureka Server` 的相关依赖，然后在启动类加上注解 `@EnableEurekaServer`，就可以将其作为注册中心，启动服务后访问页面如下：

[![eureka-server-homepage.png](https://i.loli.net/2020/03/15/7TAmjGKnQ2PXuMd.png)](https://i.loli.net/2020/03/15/7TAmjGKnQ2PXuMd.png)

我们继续添加两个模块 `service-provider`，`service-consumer`，然后在启动类加上注解 `@EnableEurekaClient` 并指定注册中心地址为我们刚刚启动的 `Eureka Server`，再次访问可以看到两个服务都已经注册进来了。

[![instance-registered-currently.png](https://i.loli.net/2020/03/15/O7QpAjDRsqEBcz1.png)](https://i.loli.net/2020/03/15/O7QpAjDRsqEBcz1.png)

`Demo` 仓库地址：https://github.com/mghio/depth-in-springcloud

可以看到 `Eureka` 的使用非常简单，只需要添加几个注解和配置就实现了服务注册和服务发现，接下来我们看看它是如何实现这些功能的。

##### 服务注册（Register）

注册中心提供了服务注册接口，用于当有新的服务启动后进行调用来实现服务注册，或者心跳检测到服务状态异常时，变更对应服务的状态。服务注册就是发送一个 `POST` 请求带上当前实例信息到类 `ApplicationResource` 的 `addInstance` 方法进行服务注册。

[![eureka-server-applicationresource-addinstance.png](https://i.loli.net/2020/03/15/fsuYMRQgdZJ2BjK.png)](https://i.loli.net/2020/03/15/fsuYMRQgdZJ2BjK.png)

可以看到方法调用了类 `PeerAwareInstanceRegistryImpl` 的 `register` 方法，该方法主要分为两步：

1. 调用父类 `AbstractInstanceRegistry` 的 `register` 方法把当前服务注册到注册中心
2. 调用 `replicateToPeers` 方法使用异步的方式向其它的 `Eureka Server` 节点同步服务注册信息

服务注册信息保存在一个嵌套的 `map` 中，它的结构如下：

[![eureka-server-registry-structure.png](https://i.loli.net/2020/03/15/j1JAOcCbUIn5hu3.png)](https://i.loli.net/2020/03/15/j1JAOcCbUIn5hu3.png)

第一层 `map` 的 `key` 是应用名称（对应 `Demo` 里的 `SERVICE-PROVIDER`），第二层 `map` 的 `key` 是应用对应的实例名称（对应 `Demo` 里的 `mghio-mbp:service-provider:9999`），一个应用可以有多个实例，主要调用流程如下图所示：

[![eureka-server-register-sequence-chart.png](https://i.loli.net/2020/03/15/fWbIeUM3FGsT7nE.png)](https://i.loli.net/2020/03/15/fWbIeUM3FGsT7nE.png)

##### 服务续约（Renew）

服务续约会由服务提供者（比如 `Demo` 中的 `service-provider`）定期调用，类似于心跳，用来告知注册中心 `Eureka Server` 自己的状态，避免被 `Eureka Server` 认为服务时效将其剔除下线。服务续约就是发送一个 `PUT` 请求带上当前实例信息到类 `InstanceResource` 的 `renewLease` 方法进行服务续约操作。

[![eureka-server-instanceresource-renew.png](https://i.loli.net/2020/03/15/UXSiGIjPydWxuFD.png)](https://i.loli.net/2020/03/15/UXSiGIjPydWxuFD.png)

进入到 `PeerAwareInstanceRegistryImpl` 的 `renew` 方法可以看到，服务续约步骤大体上和服务注册一致，先更新当前 `Eureka Server` 节点的状态，服务续约成功后再用异步的方式同步状态到其它 `Eureka Server` 节上，主要调用流程如下图所示：

[![eureka-server-renew-sequence-chart.png](https://i.loli.net/2020/03/15/tDTNqfn8wsuU2XM.png)](https://i.loli.net/2020/03/15/tDTNqfn8wsuU2XM.png)

##### 服务下线（Cancel）

当服务提供者（比如 `Demo` 中的 `service-provider`）停止服务时，会发送请求告知注册中心 `Eureka Server` 进行服务剔除下线操作，防止服务消费者从注册中心调用到不存在的服务。服务下线就是发送一个 `DELETE` 请求带上当前实例信息到类 `InstanceResource` 的 `cancelLease` 方法进行服务剔除下线操作。

[![eureka-server-instanceresource-cancellease.png](https://i.loli.net/2020/03/15/PqTHv1VyC98YnSh.png)](https://i.loli.net/2020/03/15/PqTHv1VyC98YnSh.png)

进入到 `PeerAwareInstanceRegistryImpl` 的 `cancel` 方法可以看到，服务续约步骤大体上和服务注册一致，先在当前 `Eureka Server` 节点剔除下线该服务，服务下线成功后再用异步的方式同步状态到其它 `Eureka Server` 节上，主要调用流程如下图所示：

[![eureka-server-cancellease-sequence-chart.png](https://i.loli.net/2020/03/15/bViYUoXfKgcyMZB.png)](https://i.loli.net/2020/03/15/bViYUoXfKgcyMZB.png)

##### 服务剔除（Eviction）

服务剔除是注册中心 `Eureka Server` 在启动时就启动一个守护线程 `evictionTimer` 来定期（默认为 `60` 秒）执行检测服务的，判断标准就是超过一定时间没有进行 `Renew` 的服务，默认的失效时间是 `90` 秒，也就是说当一个已注册的服务在 `90` 秒内没有向注册中心 `Eureka Server` 进行服务续约（Renew），就会被从注册中心剔除下线。失效时间可以通过配置 `eureka.instance.leaseExpirationDurationInSeconds` 进行修改，定期执行检测服务可以通过配置 `eureka.server.evictionIntervalTimerInMs` 进行修改，主要调用流程如下图所示：

[![eureka-server-evict-sequence-chart.png](https://i.loli.net/2020/03/15/khRzlKOL2ZigUwa.png)](https://i.loli.net/2020/03/15/khRzlKOL2ZigUwa.png)

#### 服务提供者（Service Provider）

对于服务提供方（比如 `Demo` 中的 `service-provider` 服务）来说，主要有三大类操作，分别为 `服务注册（Register）`、`服务续约（Renew）`、`服务下线（Cancel）`，接下来看看这三个操作是如何实现的。

##### 服务注册（Register）

一个服务要对外提供服务，首先要在注册中心 `Eureka Server` 进行服务相关信息注册，能进行这一步的前提是你要配置 `eureka.client.register-with-eureka=true`，这个默认值为 `true`，注册中心不需要把自己注册到注册中心去，把这个配置设为 `false`，这个调用比较简单，主要调用流程如下图所示：

[![service-provider-register-sequence-chart.png](https://i.loli.net/2020/03/15/uv9GsO2JPeCMd3N.png)](https://i.loli.net/2020/03/15/uv9GsO2JPeCMd3N.png)

##### 服务续约（Renew）

服务续约是由服务提供者方定期（默认为 `30` 秒）发起心跳的，主要是用来告知注册中心 `Eureka Server` 自己状态是正常的还活着，可以通过配置 `eureka.instance.lease-renewal-interval-in-seconds` 来修改，当然服务续约的前提是要配置 `eureka.client.register-with-eureka=true`，将该服务注册到注册中心中去，主要调用流程如下图所示：

[![service-provider-renew-sequence-chart.png](https://i.loli.net/2020/03/15/1YVRCjrm45sPOag.png)](https://i.loli.net/2020/03/15/1YVRCjrm45sPOag.png)

##### 服务下线（Cancel）

当服务提供者方服务停止时，要发送 `DELETE` 请求告知注册中心 `Eureka Server` 自己已经下线，好让注册中心将自己剔除下线，防止服务消费方从注册中心获取到不可用的服务。这个过程实现比较简单，在类 `DiscoveryClient` 的 `shutdown` 方法加上注解 `@PreDestroy`，当服务停止时会自动触发服务剔除下线，执行服务下线逻辑，主要调用流程如下图所示：

[![service-provider-cancel-sequence-chart.png](https://i.loli.net/2020/03/15/UGIsoMjSh9xu23t.png)](https://i.loli.net/2020/03/15/UGIsoMjSh9xu23t.png)

#### 服务消费者（Service Consumer）

这里的服务消费者如果不需要被其它服务调用的话，其实只会涉及到两个操作，分别是从注册中心 `获取服务列表（Fetch）` 和 `更新服务列表（Update）`。如果同时也需要注册到注册中心对外提供服务的话，那么剩下的过程和上文提到的服务提供者是一致的，这里不再阐述，接下来看看这两个操作是如何实现的。

##### 获取服务列表（Fetch）

服务消费者方启动之后首先肯定是要先从注册中心 `Eureka Server` 获取到可用的服务列表同时本地也会缓存一份。这个获取服务列表的操作是在服务启动后 `DiscoverClient` 类实例化的时候执行的。

[![service-consumer-fetchregistry.png](https://i.loli.net/2020/03/15/ImUKf8lScgj37ZN.png)](https://i.loli.net/2020/03/15/ImUKf8lScgj37ZN.png)

可以看出，能发生这个获取服务列表的操作前提是要保证配置了 `eureka.client.fetch-registry=true`，该配置的默认值为 `true`，主要调用流程如下图所示：

[![service-consumer-fetch-sequence-chart.png](https://i.loli.net/2020/03/15/Dy5LA439fcpjYav.png)](https://i.loli.net/2020/03/15/Dy5LA439fcpjYav.png)

##### 更新服务列表（Update）

由上面的 `获取服务列表（Fetch）` 操作过程可知，本地也会缓存一份，所以这里需要定期的去到注册中心 `Eureka Server` 获取服务的最新配置，然后比较更新本地缓存，这个更新的间隔时间可以通过配置 `eureka.client.registry-fetch-interval-seconds` 修改，默认为 `30` 秒，能进行这一步更新服务列表的前提是你要配置 `eureka.client.register-with-eureka=true`，这个默认值为 `true`。主要调用流程如下图所示：

[![service-consumer-update-sequence-chart.png](https://i.loli.net/2020/03/15/5Zi6MstvO87UoSJ.png)](https://i.loli.net/2020/03/15/5Zi6MstvO87UoSJ.png)

#### 总结

工作中项目使用的是 `Spring Cloud` 技术栈，它有一套非常完善的开源代码来整合 `Eureka`，使用起来非常方便。之前都是直接加注解和修改几个配置属性一气呵成的，没有深入了解过源码实现，本文主要是阐述了服务注册、服务发现等相关过程和实现方式，对 `Eureka` 服务发现组件有了更近一步的了解。

------

参考文章
[Netflix Eureka](https://github.com/Netflix/eureka)
[Service Discovery in a Microservices Architecture](https://www.nginx.com/blog/service-discovery-in-a-microservices-architecture)

# [Spring Cloud 源码分析之OpenFeign](https://my.oschina.net/u/862741/blog/5440205)

OpenFeign是一个远程客户端请求代理，它的基本作用是让开发者能够以面向接口的方式来实现远程调用，从而屏蔽底层通信的复杂性，它的具体原理如下图所示。

![image-20211215192443739](https://oscimg.oschina.net/oscnet/up-920ae836c3abd1564c2e0c7c8170e6c5804.png)

在今天的内容中，我们需要详细分析OpenFeign它的工作原理及源码，我们继续回到这段代码。

```java
@Slf4j
@RestController
@RequestMapping("/order")
public class OrderController {

    @Autowired
    IGoodsServiceFeignClient goodsServiceFeignClient;
    @Autowired
    IPromotionServiceFeignClient promotionServiceFeignClient;
    @Autowired
    IOrderServiceFeignClient orderServiceFeignClient;

    /**
     * 下单
     */
    @GetMapping
    public String order(){
        String goodsInfo=goodsServiceFeignClient.getGoodsById();
        String promotionInfo=promotionServiceFeignClient.getPromotionById();
        String result=orderServiceFeignClient.createOrder(goodsInfo,promotionInfo);
        return result;
    }
}
```

从这段代码中，先引出对于OpenFeign功能实现的思考。

1. 声明`@FeignClient`注解的接口，如何被解析和注入的？
2. 通过`@Autowired`依赖注入，到底是注入一个什么样的实例
3. 基于FeignClient声明的接口被解析后，如何存储？
4. 在发起方法调用时，整体的工作流程是什么样的？
5. OpenFeign是如何集成Ribbon做负载均衡解析？

带着这些疑问，开始去逐项分析OpenFeign的核心源码

# OpenFeign注解扫描与解析

> 思考， 一个被声明了`@FeignClient`注解的接口，使用`@Autowired`进行依赖注入，而最终这个接口能够正常被注入实例。

从这个结果来看，可以得到两个结论

1. 被`@FeignClient`声明的接口，在Spring容器启动时，会被解析。
2. 由于被Spring容器加载的是接口，而接口又没有实现类，因此Spring容器解析时，会生成一个动态代理类。

## EnableFeignClient

`@FeignClient`注解是在什么时候被解析的呢？基于我们之前所有积累的知识，无非就以下这几种

- ImportSelector，批量导入bean
- ImportBeanDefinitionRegistrar，导入bean声明并进行注册
- BeanFactoryPostProcessor ， 一个bean被装载的前后处理器

在这几个选项中，似乎`ImportBeanDefinitionRegistrar`更合适，因为第一个是批量导入一个bean的string集合，不适合做动态Bean的声明。 而`BeanFactoryPostProcessor `是一个Bean初始化之前和之后被调用的处理器。

而在我们的FeignClient声明中，并没有Spring相关的注解，所以自然也不会被Spring容器加载和触发。

> 那么`@FeignClient`是在哪里被声明扫描的呢？

在集成FeignClient时，我们在SpringBoot的main方法中，声明了一个注解`@EnableFeignClients(basePackages = "com.gupaoedu.ms.api")`。这个注解需要填写一个指定的包名。

嗯，看到这里，基本上就能猜测出，这个注解必然和`@FeignClient`注解的解析有莫大的关系。

下面这段代码是`@EnableFeignClients`注解的声明，果然看到了一个很熟悉的面孔`FeignClientsRegistrar`。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
}
```

## FeignClientsRegistrar

FeignClientRegistrar，主要功能就是针对声明`@FeignClient`注解的接口进行扫描和注入到IOC容器。

```java
class FeignClientsRegistrar
      implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {
    
}
```

果然，这个类实现了`ImportBeanDefinitionRegistrar`接口

```java
public interface ImportBeanDefinitionRegistrar {
    default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry, BeanNameGenerator importBeanNameGenerator) {
        this.registerBeanDefinitions(importingClassMetadata, registry);
    }

    default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    }
}
```

这个接口有两个重载的方法，用来实现Bean的声明和注册。

简单给大家演示一下ImportBeanDefinitionRegistrar的作用。

> 在`gpmall-portal`这个项目的`com.gupaoedu`目录下，分别创建
>
> 1. HelloService.java
> 2. GpImportBeanDefinitionRegistrar.java
> 3. EnableGpRegistrar.java
> 4. TestMain

1. 定义一个需要被装载到IOC容器中的类HelloService

```java
public class HelloService {
}
```

1. 定义一个Registrar的实现，定义一个bean，装载到IOC容器

```java
public class GpImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                        BeanDefinitionRegistry registry) {
        BeanDefinition beanDefinition=new GenericBeanDefinition();
        beanDefinition.setBeanClassName(HelloService.class.getName());
        registry.registerBeanDefinition("helloService",beanDefinition);
    }
}
```

1. 定义一个注解类

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(GpImportBeanDefinitionRegistrar.class)
public @interface EnableGpRegistrar {
}
```

1. 写一个测试类

```java
@Configuration
@EnableGpRegistrar
public class TestMain {

    public static void main(String[] args) {
        ApplicationContext applicationContext=new AnnotationConfigApplicationContext(TestMain.class);
        System.out.println(applicationContext.getBean(HelloService.class));

    }
}
```

1. 通过结果演示可以发现，`HelloService`这个bean 已经装载到了IOC容器。

这就是动态装载的功能实现，它相比于@Configuration配置注入，会多了很多的灵活性。 ok，再回到FeignClient的解析中来。

## FeignClientsRegistrar.registerBeanDefinitions

- `registerDefaultConfiguration` 方法内部从 `SpringBoot` 启动类上检查是否有 `@EnableFeignClients`, 有该注解的话， 则完成`Feign`框架相关的一些配置内容注册。
- `registerFeignClients` 方法内部从 `classpath` 中， 扫描获得 `@FeignClient` 修饰的类， 将类的内容解析为 `BeanDefinition` , 最终通过调用 Spring 框架中的 `BeanDefinitionReaderUtils.resgisterBeanDefinition` 将解析处理过的 `FeignClient BeanDeifinition` 添加到 `spring` 容器中

```java
//BeanDefinitionReaderUtils.resgisterBeanDefinition 
@Override
public void registerBeanDefinitions(AnnotationMetadata metadata,
                                    BeanDefinitionRegistry registry) {
    //注册@EnableFeignClients中定义defaultConfiguration属性下的类，包装成FeignClientSpecification，注册到Spring容器。
    //在@FeignClient中有一个属性：configuration，这个属性是表示各个FeignClient自定义的配置类，后面也会通过调用registerClientConfiguration方法来注册成FeignClientSpecification到容器。
//所以，这里可以完全理解在@EnableFeignClients中配置的是做为兜底的配置，在各个@FeignClient配置的就是自定义的情况。
    registerDefaultConfiguration(metadata, registry);
    registerFeignClients(metadata, registry);
}
```

> 这里面需要重点分析的就是`registerFeignClients`方法，这个方法主要是扫描类路径下所有的`@FeignClient`注解，然后进行动态Bean的注入。它最终会调用`registerFeignClient`方法。

```java
public void registerFeignClients(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
    registerFeignClient(registry, annotationMetadata, attributes);
}
```

## FeignClientsRegistrar.registerFeignClients

registerFeignClients方法的定义如下。

```java
//# FeignClientsRegistrar.registerFeignClients
public void registerFeignClients(AnnotationMetadata metadata,
      BeanDefinitionRegistry registry) {
   
   LinkedHashSet<BeanDefinition> candidateComponents = new LinkedHashSet<>();
   //获取@EnableFeignClients注解的元数据
   Map<String, Object> attrs = metadata
         .getAnnotationAttributes(EnableFeignClients.class.getName());
    //获取@EnableFeignClients注解中的clients属性，可以配置@FeignClient声明的类，如果配置了，则需要扫描并加载。
   final Class<?>[] clients = attrs == null ? null
         : (Class<?>[]) attrs.get("clients");
   if (clients == null || clients.length == 0) {
       //默认TypeFilter生效，这种模式会查询出许多不符合你要求的class名
      ClassPathScanningCandidateComponentProvider scanner = getScanner();
      scanner.setResourceLoader(this.resourceLoader);
      scanner.addIncludeFilter(new AnnotationTypeFilter(FeignClient.class)); //添加包含过滤的属性@FeignClient。
      Set<String> basePackages = getBasePackages(metadata); //从@EnableFeignClients注解中获取basePackages配置。
      for (String basePackage : basePackages) {
          //scanner.findCandidateComponents(basePackage) 扫描basePackage下的@FeignClient注解声明的接口
         candidateComponents.addAll(scanner.findCandidateComponents(basePackage)); //添加到candidateComponents，也就是候选容器中。
      }
   }
   else {//如果配置了clients，则需要添加到candidateComponets中。
      for (Class<?> clazz : clients) {
         candidateComponents.add(new AnnotatedGenericBeanDefinition(clazz));
      }
   }
   //遍历候选容器列表。
   for (BeanDefinition candidateComponent : candidateComponents) { 
      if (candidateComponent instanceof AnnotatedBeanDefinition) { //如果属于AnnotatedBeanDefinition实例类型
         // verify annotated class is an interface
          //得到@FeignClient注解的beanDefinition
         AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
         AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();  //获取这个bean的注解元数据
         Assert.isTrue(annotationMetadata.isInterface(),
               "@FeignClient can only be specified on an interface"); 
         //获取元数据属性
         Map<String, Object> attributes = annotationMetadata
               .getAnnotationAttributes(FeignClient.class.getCanonicalName());
         //获取@FeignClient中配置的服务名称。
         String name = getClientName(attributes);
         registerClientConfiguration(registry, name,
               attributes.get("configuration"));

         registerFeignClient(registry, annotationMetadata, attributes);
      }
   }
}
```

# FeignClient Bean的注册

这个方法就是把FeignClient接口注册到Spring IOC容器，

> FeignClient是一个接口，那么这里注入到IOC容器中的对象是什么呢？

```java
private void registerFeignClient(BeanDefinitionRegistry registry,
      AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
   String className = annotationMetadata.getClassName();  //获取FeignClient接口的类全路径
   Class clazz = ClassUtils.resolveClassName(className, null); //加载这个接口，得到Class实例
    //构建ConfigurableBeanFactory，提供BeanFactory的配置能力
   ConfigurableBeanFactory beanFactory = registry instanceof ConfigurableBeanFactory 
         ? (ConfigurableBeanFactory) registry : null;
    //获取contextId
   String contextId = getContextId(beanFactory, attributes);
   String name = getName(attributes);
    //构建一个FeignClient FactoryBean，这个是工厂Bean。
   FeignClientFactoryBean factoryBean = new FeignClientFactoryBean();
    //设置工厂Bean的相关属性
   factoryBean.setBeanFactory(beanFactory);
   factoryBean.setName(name);
   factoryBean.setContextId(contextId);
   factoryBean.setType(clazz);
    
    //BeanDefinitionBuilder是用来构建BeanDefinition对象的建造器
   BeanDefinitionBuilder definition = BeanDefinitionBuilder
         .genericBeanDefinition(clazz, () -> {
             //把@FeignClient注解配置中的属性设置到FactoryBean中。
            factoryBean.setUrl(getUrl(beanFactory, attributes));
            factoryBean.setPath(getPath(beanFactory, attributes));
            factoryBean.setDecode404(Boolean
                  .parseBoolean(String.valueOf(attributes.get("decode404"))));
            Object fallback = attributes.get("fallback");
            if (fallback != null) {
               factoryBean.setFallback(fallback instanceof Class
                     ? (Class<?>) fallback
                     : ClassUtils.resolveClassName(fallback.toString(), null));
            }
            Object fallbackFactory = attributes.get("fallbackFactory");
            if (fallbackFactory != null) {
               factoryBean.setFallbackFactory(fallbackFactory instanceof Class
                     ? (Class<?>) fallbackFactory
                     : ClassUtils.resolveClassName(fallbackFactory.toString(),
                           null));
            }
            return factoryBean.getObject();  //factoryBean.getObject() ，基于工厂bean创造一个bean实例。
         });
   definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE); //设置注入模式，采用类型注入
   definition.setLazyInit(true); //设置延迟华
   validate(attributes); 
   //从BeanDefinitionBuilder中构建一个BeanDefinition，它用来描述一个bean的实例定义。
   AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
   beanDefinition.setAttribute(FactoryBean.OBJECT_TYPE_ATTRIBUTE, className);
   beanDefinition.setAttribute("feignClientsRegistrarFactoryBean", factoryBean);

   // has a default, won't be null
   boolean primary = (Boolean) attributes.get("primary");

   beanDefinition.setPrimary(primary);

   String[] qualifiers = getQualifiers(attributes);
   if (ObjectUtils.isEmpty(qualifiers)) {
      qualifiers = new String[] { contextId + "FeignClient" };
   }
   
   BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
         qualifiers);
   BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry); //把BeanDefinition的这个bean定义注册到IOC容器。
}
```

综上代码分析，其实实现逻辑很简单。

1. 创建一个BeanDefinitionBuilder。
2. 创建一个工厂Bean，并把从@FeignClient注解中解析的属性设置到这个FactoryBean中
3. 调用registerBeanDefinition注册到IOC容器中

## 关于FactoryBean

在上述的逻辑中，我们重点需要关注的是FactoryBean，这个大家可能接触得会比较少一点。

首先，需要注意的是FactoryBean和BeanFactory是不一样的，FactoryBean是一个工厂Bean，可以生成某一个类型Bean实例，它最大的一个作用是：可以让我们自定义Bean的创建过程。

而，BeanFactory是Spring容器中的一个基本类也是很重要的一个类，在BeanFactory中可以创建和管理Spring容器中的Bean。

下面这段代码是FactoryBean接口的定义。

```java
public interface FactoryBean<T> {
    String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";

    @Nullable
    T getObject() throws Exception;

    @Nullable
    Class<?> getObjectType();

    default boolean isSingleton() {
        return true;
    }
}
```

从上面的代码中我们发现在FactoryBean中定义了一个Spring Bean的很重要的三个特性：是否单例、Bean类型、Bean实例，这也应该是我们关于Spring中的一个Bean最直观的感受。

## FactoryBean自定义使用

下面我们来模拟一下@FeignClient解析以及工厂Bean的构建过程。

1. 先定义一个接口，这个接口可以类比为我们上面描述的FeignClient.

```java
public interface IHelloService {

    String say();
}
```

1. 接着，定义一个工厂Bean，这个工厂Bean中主要是针对IHelloService生成动态代理。

```java
public class DefineFactoryBean implements FactoryBean<IHelloService> {

    @Override
    public IHelloService getObject()  {
       IHelloService helloService=(IHelloService) Proxy.newProxyInstance(IHelloService.class.getClassLoader(),
               new Class<?>[]{IHelloService.class}, (proxy, method, args) -> {
           System.out.println("begin execute");
           return "Hello FactoryBean";
       });
        return helloService;
    }

    @Override
    public Class<?> getObjectType() {
        return IHelloService.class;
    }
}
```

1. 通过实现ImportBeanDefinitionRegistrar这个接口，来动态注入Bean实例

```java
public class GpImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                        BeanDefinitionRegistry registry) {

        DefineFactoryBean factoryBean=new DefineFactoryBean();
        BeanDefinitionBuilder definition= BeanDefinitionBuilder.genericBeanDefinition(
                IHelloService.class,()-> factoryBean.getObject());
        BeanDefinition beanDefinition=definition.getBeanDefinition();
        registry.registerBeanDefinition("helloService",beanDefinition);
    }
}
```

1. 声明一个注解，用来表示动态bean的注入导入。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(GpImportBeanDefinitionRegistrar.class)
public @interface EnableGpRegistrar {
}
```

1. 编写测试类，测试IHelloService这个接口的动态注入

```java
@Configuration
@EnableGpRegistrar
public class TestMain {

    public static void main(String[] args) {
        ApplicationContext applicationContext=new AnnotationConfigApplicationContext(TestMain.class);
        IHelloService is=applicationContext.getBean(IHelloService.class);
        System.out.println(is.say());

    }
}
```

1. 运行上述的测试方法，可以看到IHelloService这个接口，被动态代理并且注入到了IOC容器。

# FeignClientFactoryBean

由上述案例可知，Spring IOC容器在注入FactoryBean时，会调用FactoryBean的getObject()方法获得bean的实例。因此我们可以按照这个思路去分析FeignClientFactoryBean

```java
@Override
public Object getObject() {
   return getTarget();
}
```

构建对象Bean的实现代码如下，这个代码的实现较长，我们分为几个步骤来看

## Feign上下文的构建

先来看上下文的构建逻辑，代码部分如下。

```java
<T> T getTarget() {
   FeignContext context = beanFactory != null
         ? beanFactory.getBean(FeignContext.class)
         : applicationContext.getBean(FeignContext.class);
   Feign.Builder builder = feign(context);
}
```

两个关键的对象说明：

1. FeignContext是全局唯一的上下文，它继承了NamedContextFactory，它是用来来统一维护feign中各个feign客户端相互隔离的上下文，FeignContext注册到容器是在FeignAutoConfiguration上完成的，代码如下！

```java
//FeignAutoConfiguration.java

@Autowired(required = false)
private List<FeignClientSpecification> configurations = new ArrayList<>();
@Bean
public FeignContext feignContext() {
FeignContext context = new FeignContext();
context.setConfigurations(this.configurations);
return context;
}
```

在初始化FeignContext时，会把`configurations`放入到FeignContext中。`configuration`表示当前被扫描到的所有@FeignClient。

1. Feign.Builder用来构建Feign对象，基于builder实现上下文信息的构建，代码如下。

```java
protected Feign.Builder feign(FeignContext context) {
    FeignLoggerFactory loggerFactory = get(context, FeignLoggerFactory.class);
    Logger logger = loggerFactory.create(type);

    // @formatter:off
    Feign.Builder builder = get(context, Feign.Builder.class)
        // required values
        .logger(logger)
        .encoder(get(context, Encoder.class))
        .decoder(get(context, Decoder.class))
        .contract(get(context, Contract.class)); //contract协议，用来实现模版解析（后面再详细分析）
    // @formatter:on

    configureFeign(context, builder);
    applyBuildCustomizers(context, builder);

    return builder;
}
```

从代码中可以看到，`feign`方法，主要是针对不同的服务提供者生成Feign的上下文信息，比如`logger`、`encoder`、`decoder`等。因此，从这个分析过程中，我们不难猜测到它的原理结构，如下图所示

![image-20211215192527913](https://oscimg.oschina.net/oscnet/up-74c4cc300dad26a6eb616340c9058f1b5c5.png)

父子容器隔离的实现方式如下，当调用get方法时，会从`context`中去获取指定type的实例对象。

```java
//FeignContext.java
protected <T> T get(FeignContext context, Class<T> type) {
    T instance = context.getInstance(contextId, type);
    if (instance == null) {
        throw new IllegalStateException(
            "No bean found of type " + type + " for " + contextId);
    }
    return instance;
}
```

接着，调用`NamedContextFactory`中的`getInstance`方法。

```java
//NamedContextFactory.java
public <T> T getInstance(String name, Class<T> type) {
    //根据`name`获取容器上下文
    AnnotationConfigApplicationContext context = this.getContext(name);

    try {
        //再从容器上下文中获取指定类型的bean。
        return context.getBean(type);
    } catch (NoSuchBeanDefinitionException var5) {
        return null;
    }
}
```

`getContext`方法根据`name`从`contexts`容器中获得上下文对象，如果没有，则调用`createContext`创建。

```java
protected AnnotationConfigApplicationContext getContext(String name) {
    if (!this.contexts.containsKey(name)) {
        synchronized(this.contexts) {
            if (!this.contexts.containsKey(name)) {
                this.contexts.put(name, this.createContext(name));
            }
        }
    }

    return (AnnotationConfigApplicationContext)this.contexts.get(name);
}
```

## 生成动态代理

第二个部分，如果@FeignClient注解中，没有配置`url`，也就是不走绝对请求路径，则执行下面这段逻辑。

由于我们在@FeignClient注解中使用的是`name`，所以需要执行负载均衡策略的分支逻辑。

```java
<T> T getTarget() {
    //省略.....
     if (!StringUtils.hasText(url)) { //是@FeignClient中的一个属性，可以定义请求的绝对URL

      if (LOG.isInfoEnabled()) {
         LOG.info("For '" + name
               + "' URL not provided. Will try picking an instance via load-balancing.");
      }
      if (!name.startsWith("http")) {
         url = "http://" + name;
      }
      else {
         url = name;
      }
      url += cleanPath();
      //
      return (T) loadBalance(builder, context,
            new HardCodedTarget<>(type, name, url));
   }
    //省略....
}
```

loadBalance方法的代码实现如下，其中`Client`是Spring Boot自动装配的时候实现的，如果替换了其他的http协议框架，则client则对应为配置的协议api。

```java
protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
                            HardCodedTarget<T> target) {
    //Feign发送请求以及接受响应的http client，默认是Client.Default的实现，可以修改成OkHttp、HttpClient等。
    Client client = getOptional(context, Client.class);
    if (client != null) {
        builder.client(client); //针对当前Feign客户端，设置网络通信的client
        //targeter表示HystrixTarger实例，因为Feign可以集成Hystrix实现熔断，所以这里会一层包装。
        Targeter targeter = get(context, Targeter.class);
        return targeter.target(this, builder, context, target);
    }

    throw new IllegalStateException(
        "No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon or spring-cloud-starter-loadbalancer?");
}
```

`HystrixTarget.target`代码如下，我们没有集成Hystrix，因此不会触发Hystrix相关的处理逻辑。

```java
//HystrixTarget.java
@Override
public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
                    FeignContext context, Target.HardCodedTarget<T> target) {
    if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) { //没有配置Hystrix，则走这部分逻辑
        return feign.target(target);
    }
   //省略....

    return feign.target(target);
}
```

进入到`Feign.target`方法，代码如下。

```java
//Feign.java
public <T> T target(Target<T> target) {
    return this.build().newInstance(target);  //target.HardCodedTarget
}

public Feign build() {
    //这里会构建一个LoadBalanceClient
    Client client = (Client)Capability.enrich(this.client, this.capabilities);
    Retryer retryer = (Retryer)Capability.enrich(this.retryer, this.capabilities);
    List<RequestInterceptor> requestInterceptors = (List)this.requestInterceptors.stream().map((ri) -> {
        return (RequestInterceptor)Capability.enrich(ri, this.capabilities);
    }).collect(Collectors.toList());
    //OpenFeign Log配置
    Logger logger = (Logger)Capability.enrich(this.logger, this.capabilities);
    //模版解析协议（这个接口非常重要：它决定了哪些注解可以标注在接口/接口方法上是有效的，并且提取出有效的信息，组装成为MethodMetadata元信息。）
    Contract contract = (Contract)Capability.enrich(this.contract, this.capabilities);
    //封装Request请求的 连接超时=默认10s ，读取超时=默认60
    Options options = (Options)Capability.enrich(this.options, this.capabilities);
    //编码器
    Encoder encoder = (Encoder)Capability.enrich(this.encoder, this.capabilities);
    //解码器
    Decoder decoder = (Decoder)Capability.enrich(this.decoder, this.capabilities);
    
    InvocationHandlerFactory invocationHandlerFactory = (InvocationHandlerFactory)Capability.enrich(this.invocationHandlerFactory, this.capabilities);
    QueryMapEncoder queryMapEncoder = (QueryMapEncoder)Capability.enrich(this.queryMapEncoder, this.capabilities);
    //synchronousMethodHandlerFactory， 同步方法调用处理器（很重要，后续要用到）
    Factory synchronousMethodHandlerFactory = new Factory(client, retryer, requestInterceptors, logger, this.logLevel, this.decode404, this.closeAfterDecode, this.propagationPolicy, this.forceDecoding);
    //ParseHandlersByName的作用就是我们传入Target（封装了我们的模拟接口，要访问的域名），返回这个接口下的各个方法，对应的执行HTTP请求需要的一系列信息
    ParseHandlersByName handlersByName = new ParseHandlersByName(contract, options, encoder, decoder, queryMapEncoder, this.errorDecoder, synchronousMethodHandlerFactory);
    
    return new ReflectiveFeign(handlersByName, invocationHandlerFactory, queryMapEncoder);
}
```

`build`方法中，返回了一个`ReflectiveFeign`的实例对象，先来看`ReflectiveFeign`中的`newInstance`方法。

```java
public <T> T newInstance(Target<T> target) {    //修饰了@FeignClient注解的接口方法封装成方法处理器，把指定的target进行解析，得到需要处理的方法集合。    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);    //定义一个用来保存需要处理的方法的集合    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();    //JDK8以后，接口允许默认方法实现，这里是对默认方法进行封装处理。    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();	    //遍历@FeignClient接口的所有方法    for (Method method : target.type().getMethods()) {        //如果是Object中的方法，则直接跳过        if (method.getDeclaringClass() == Object.class) {            continue;                    } else if (Util.isDefault(method)) {//如果是默认方法，则把该方法绑定一个DefaultMethodHandler。            DefaultMethodHandler handler = new DefaultMethodHandler(method);            defaultMethodHandlers.add(handler);            methodToHandler.put(method, handler);        } else {//否则，添加MethodHandler(SynchronousMethodHandler)。            methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));        }    }    //创建动态代理类。    InvocationHandler handler = factory.create(target, methodToHandler);    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),                                         new Class<?>[] {target.type()}, handler);    for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {        defaultMethodHandler.bindTo(proxy);    }    return proxy;}
```

上述代码，其实也不难理解。

1. 解析@FeignClient接口声明的方法，根据不同方法绑定不同的处理器。
   1. 默认方法，绑定DefaultMethodHandler
   2. 远程方法，绑定SynchronousMethodHandler
2. 使用JDK提供的Proxy创建动态代理

MethodHandler，会把方法参数、方法返回值、参数集合、请求类型、请求路径进行解析存储，如下图所示。

![image-20211123152837678](https://oscimg.oschina.net/oscnet/up-5d4579f1f33903152215ace743104baab37.png)

## FeignClient接口解析

接口解析也是Feign很重要的一个逻辑，它能把接口声明的属性转化为HTTP通信的协议参数。

> 执行逻辑RerlectiveFeign.newInstance

```java
public <T> T newInstance(Target<T> target) {    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target); //here}
```

targetToHandlersByName.apply(target);会解析接口方法上的注解，从而解析出方法粒度的特定的配置信息，然后生产一个SynchronousMethodHandler 然后需要维护一个<method，MethodHandler>的map，放入InvocationHandler的实现FeignInvocationHandler中。

```java
public Map<String, MethodHandler> apply(Target target) {    List<MethodMetadata> metadata = contract.parseAndValidateMetadata(target.type());    Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();    for (MethodMetadata md : metadata) {        BuildTemplateByResolvingArgs buildTemplate;        if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {            buildTemplate =                new BuildFormEncodedTemplateFromArgs(md, encoder, queryMapEncoder, target);        } else if (md.bodyIndex() != null) {            buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder, queryMapEncoder, target);        } else {            buildTemplate = new BuildTemplateByResolvingArgs(md, queryMapEncoder, target);        }        if (md.isIgnored()) {            result.put(md.configKey(), args -> {                throw new IllegalStateException(md.configKey() + " is not a method handled by feign");            });        } else {            result.put(md.configKey(),                       factory.create(target, md, buildTemplate, options, decoder, errorDecoder));        }    }    return result;}
```

为了更好的理解上述逻辑，我们可以借助下面这个图来理解！

## 阶段性小结

通过上述过程分析，被声明为@FeignClient注解的类，在被注入时，最终会生成一个动态代理对象FeignInvocationHandler。

当触发方法调用时，会被FeignInvocationHandler#invoke拦截，FeignClientFactoryBean在实例化过程中所做的事情如下图所示。

![image-20211215192548769](https://oscimg.oschina.net/oscnet/up-fb6835f9fca99723914a7051c4d6b1165cc.png)

总结来说就几个点：

1. 解析Feign的上下文配置，针对当前的服务实例构建容器上下文并返回Feign对象
2. Feign根据上下围配置把 log、encode、decoder、等配置项设置到Feign对象中
3. 对目标服务，使用LoadBalance以及Hystrix进行包装
4. 通过Contract协议，把FeignClient接口的声明，解析成MethodHandler
5. 遍历MethodHandler列表，针对需要远程通信的方法，设置`SynchronousMethodHandler`处理器，用来实现同步远程调用。
6. 使用JDK中的动态代理机制构建动态代理对象。

# 远程通信实现

在Spring启动过程中，把一切的准备工作准备就绪后，就开始执行远程调用。

在前面的分析中，我们知道OpenFeign最终返回的是一个#ReflectiveFeign.FeignInvocationHandler的对象。

那么当客户端发起请求时，会进入到FeignInvocationHandler.invoke方法中，这个大家都知道，它是一个动态代理的实现。

```java
//FeignInvocationHandler.java@Overridepublic Object invoke(Object proxy, Method method, Object[] args) throws Throwable {    if ("equals".equals(method.getName())) {        try {            Object otherHandler =                args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;            return equals(otherHandler);        } catch (IllegalArgumentException e) {            return false;        }    } else if ("hashCode".equals(method.getName())) {        return hashCode();    } else if ("toString".equals(method.getName())) {        return toString();    }    return dispatch.get(method).invoke(args);}
```

接着，在invoke方法中，会调用`this.dispatch.get(method)).invoke(args)`。`this.dispatch.get(method)`会返回一个SynchronousMethodHandler,进行拦截处理。

this.dispatch，其实就是在初始化过程中创建的，`private final Map<Method, MethodHandler> dispatch;`实例。

## SynchronousMethodHandler.invoke

这个方法会根据参数生成完成的RequestTemplate对象，这个对象是Http请求的模版，代码如下，代码的实现如下：

```java
@Override
public Object invoke(Object[] argv) throws Throwable {    //argv，表示调用方法传递的参数。      
  RequestTemplate template = buildTemplateFromArgs.create(argv);  
  Options options = findOptions(argv); //获取配置项，连接超时时间、远程通信数据获取超时时间  
  Retryer retryer = this.retryer.clone(); //获取重试策略  
  while (true) {    
    try {      
      return executeAndDecode(template, options);    
    } catch (RetryableException e) {
      try {        
        retryer.continueOrPropagate(e);      
      } catch (RetryableException th) {
        Throwable cause = th.getCause();        
        if (propagationPolicy == UNWRAP && cause != null) {
          throw cause;        
        } else {
          throw th;
        }      
      }      
      if (logLevel != Logger.Level.NONE) {
        logger.logRetry(metadata.configKey(), logLevel);      
      }      
      continue;    
    }  
  }
}
```

上述代码的执行流程中，需要先构造一个Request对象，然后调用client.execute方法执行网络通信请求，整体实现流程如下。

![image-20211215192557345](https://oscimg.oschina.net/oscnet/up-5f9c816c60b3484d0154430ded14eb141a8.png)

## executeAndDecode

经过上述的代码，我们已经将RequestTemplate拼装完成，上面的代码中有一个`executeAndDecode()`方法，该方法通过RequestTemplate生成Request请求对象，然后利用Http Client获取response，来获取响应信息。

```java
Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
  Request request = targetRequest(template);  //把template转化为请求报文    
  if (logLevel != Logger.Level.NONE) { //设置日志level      
    logger.logRequest(metadata.configKey(), logLevel, request); 
  }    
  Response response;   
  long start = System.nanoTime();  
  try {     
    //发起请求，此时client是LoadBalanceFeignClient，需要先对服务名称进行解析负载    
    response = client.execute(request, options);      // ensure the request is set. TODO: remove in Feign 12          //获取返回结果   
    response = response.toBuilder().request(request).requestTemplate(template).build(); 
  } catch (IOException e) { 
    if (logLevel != Logger.Level.NONE) {   
      logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));   
    }     
    throw errorExecuting(request, e);  
  }  
  long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);   
  if (decoder != null) //如果设置了解码器，这需要对响应数据做解码    
    return decoder.decode(response, metadata.returnType());   
  CompletableFuture<Object> resultFuture = new CompletableFuture<>();  
  asyncResponseHandler.handleResponse(resultFuture, metadata.configKey(), response,        metadata.returnType(),        elapsedTime);  
  try {  
    if (!resultFuture.isDone())  
      throw new IllegalStateException("Response handling not done"); 
    return resultFuture.join();   
  } catch (CompletionException e) {     
    Throwable cause = e.getCause();   
    if (cause != null)    
      throw cause;    
    throw e;  
  } 
}
```

## LoadBalanceClient

```java
@Overridepublic
Response execute(Request request, Request.Options options) throws IOException { 
  try {   
    URI asUri = URI.create(request.url()); //获取请求uri，此时的地址还未被解析。 
    String clientName = asUri.getHost(); //获取host，实际上就是服务名称  
    URI uriWithoutHost = cleanUrl(request.url(), clientName);    
    FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(this.delegate, request, uriWithoutHost);		//加载客户端的配置信息     
    IClientConfig requestConfig = getClientConfig(options, clientName);       //创建负载均衡客户端(FeignLoadBalancer)，执行请求   
    return lbClient(clientName).executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();   
  } catch (ClientException e) {
    IOException io = findIOException(e); 
    if (io != null) {     
      throw io;    
    }     
    throw new RuntimeException(e); 
  }
}
```

从上面的代码可以看到，lbClient(clientName) 创建了一个负载均衡的客户端，它实际上就是生成的如下所述的类：

```java
public class FeignLoadBalancer extends		AbstractLoadBalancerAwareClient<FeignLoadBalancer.RibbonRequest, FeignLoadBalancer.RibbonResponse> {
```

## 整体总结

OpenFeign的整体通信原理解析，如下图所示。

![image-20211215192455398](https://oscimg.oschina.net/oscnet/up-daf37410d32d603d13607fd8b79d6e40090.png)

# [深入理解 Ribbon 的架构原理](https://my.oschina.net/u/4499317/blog/5344569)

本篇主要内容如下：

 

![img](https://static001.geekbang.org/infoq/87/87b525cb1bfae24c3a5028acc004b916.png)



## 前言

今天我们来看下微服务中非常重要的一个组件：`Ribbon`。它作为负载均衡器在分布式网络中扮演着非常重要的角色。

 

在介绍 Ribbon 之前，不得不说下负载均衡这个比较偏僻的名词。为什么说它偏僻了，因为在面试中，聊得最多的是消息队列和缓存来提高系统的性能，支持高并发，很少有人会问负载均衡，究其原因，负载均衡的组件选择和搭建一般都是运维团队或者架构师去做的，开发人员确实很少接触到。不过没关系，我们不止有 CRUD，还要有架构思维。

 

简单来说，负载均衡就是将网络流量（负载）分摊到不同的网络服务器（可以平均分配，也可以不平均），系统就可以实现服务的水平横向扩展。

 

**那么如果让你设计一个负载均衡组件，你会怎么设计？**

 

我们需要考虑这几个因素：

 

- 如何获取及同步服务器列表？涉及到与注册中心的交互。
- 如何将负载进行分摊？涉及到分摊策略。
- 如何将客户端请求进行拦截然后选择服务器进行转发？涉及到请求拦截。

 

抱着这几个问题，我们从负载均衡的原理 + Ribbon 的架构来学习如何设计一个负载均衡器，相信会带给你一些启发。



## 一、负载均衡



### 1.1 概念

上次我在这篇文章中详细讲解了何为高可用，里面没有涉及到负载均衡机制，其实负载均衡也是高可用网络的关键组件。

 

![img](https://static001.geekbang.org/infoq/7e/7e522817ac1afd3e2d125f02acbad5cb.png)

 

两个基本点：

 

- 选择哪个服务器来处理客户端请求。
- 将客户端请求转发出去。

 

一个核心原理：通过硬件或软件的方式维护一个服务列表清单。当用户发送请求时，会将请求发送给负载均衡器，然后根据负载均衡算法从可用的服务列表中选出一台服务器的地址，将请求进行转发，完成负载功能。



### 1.2 负载均衡的特性

![img](https://static001.geekbang.org/infoq/e4/e43e156d130edbbbbecb0ef6a1e1ca21.png)

 

**高性能**：可根据不同的分配规则自动将流量进行分摊。

 

**可扩展性**：可以很方便增加集群中设备或链路的数量。

 

**高可靠性**：系统中某个设备或链路发生故障，不会导致服务中断。

 

**易配置性**：配置和维护方便。

 

**透明性**：用户感知不到如何进行负载均衡的，也不用关心负载均衡。



### 1.3 负载均衡分类

负载均衡技术可以按照软件或硬件进行分类，也可以按照服务器列表存放的位置划分为服务端负载和客户端负载均衡。

 

![img](https://static001.geekbang.org/infoq/ad/ad01f595e24edae99ddf6c1a43e71459.png)



#### 1.3.1 硬件负载均衡

F5 就是常见的硬件负载均衡产品。

 

优点：性能稳定，具备很多软件负载均衡不具备的功能，如应用交换，会话交换、状态监控等。

 

缺点：设备价格昂贵、配置冗余，没有软件负载均衡灵活，不能满足定制化需求。



#### 1.3.2 软件负载均衡

Nginx：性能好，可以负载超过 1W。工作在网络的7层之上，可以针对http应用做一些分流的策略。Nginx也可作为静态网页和图片服务器。Nginx仅能支持http、https和Email协议。

 

LVS（Linux Virtual Server）：是一个虚拟服务器集群系统，采用 IP 地址均衡技术和内容请求分发技术实现负载均衡。接近硬件设备的网络吞吐和连接负载能力。抗负载能力强、是工作在网络4层之上仅作分发之用。自身有完整的双机热备方案，如LVS+Keepalived。软件本身不支持正则表达式处理，不能做动静分离。



#### 1.3.3 服务端负载均衡

Nginx 和 F5 都可以划分到服务端的负载均衡里面，后端的服务器地址列表是存储在后端服务器中或者存在专门的 Nginx 服务器或 F5 上。

 

服务器的地址列表的来源是通过注册中心或者手动配置的方式来的。



#### 1.3.4 客户端负载均衡

终于轮到 Ribbon 登场了，它属于客户端负载均衡器，客户端自己维护一份服务器的地址列表。这个维护的工作就是由 Ribbon 来干的。

 

Ribbon 会从 Eureka Server 读取服务信息列表，存储在 Ribbon 中。如果服务器宕机了，Ribbon 会从列表剔除宕机的服务器信息。

 

Ribbon 有多种负载均衡算法，我们可以自行设定规则从而请求到指定的服务器。



## 二、 均衡策略

上面已经介绍了各种负载均衡分类，接下来我们来看下这些负载均衡器如何通过负载均衡策略来选择服务器处理客户端请求。

 

常见的均衡策略如下。



### 2.1 轮循均衡（Round Robin）

原理：如果给服务器从 0 到 N 编号，轮询均衡策略会从 0 开始依次选择一个服务器作为处理本次请求的服务器。-

 

场景：适合所有父亲都有相同的软硬件配置，且请求频率相对平衡。



### 2.2 权重轮询均衡（Weighted Round Robin）

原理：按照服务器的不同处理能力，给服务器分配不同的权重，然后请求会按照权重分配给不同的服务器。

 

场景：服务器的性能不同，充分利用高性能的服务器，同时也能照顾到低性能的服务器。



### 2.3 随机均衡（Random）

原理：将请求随机分配给不同的服务器。

 

场景：适合客户端请求的频率比较随机的场景。



### 2.4 响应速度均衡（Response Time)

原理：负载均衡设备对每个服务器发送一个探测请求，看看哪台服务器的响应速度更快，

 

场景：适合服务器的响应性能不断变化的场景。

 

注意：响应速度是针对负载均衡设备和服务器之间的。



## 三、Ribbon 核心组件

接下来就是我们的重头戏了，来看下 Ribbon 这个 Spring Cloud 中负载均衡组件。

 

Ribbon 主要有五大功能组件：ServerList、Rule、Ping、ServerListFilter、ServerListUpdater。

 

![img](https://static001.geekbang.org/infoq/12/1218aa6fb7242881bcff27a6b03d446b.png)



### 3.1 负载均衡器 LoadBalancer

用于管理负载均衡的组件。初始化的时候通过加载 YMAL 配置文件创建出来的。



### 3.2 服务列表 ServerList

ServerList 主要用来获取所有服务的地址信息，并存到本地。

 

根据获取服务信息的方式不同，又分为静态存储和动态存储。

 

静态存储：从配置文件中获取服务节点列表并存储到本地。

 

动态存储：从注册中心获取服务节点列表并存储到本地



### 3.3 服务列表过滤 ServerListFilter

将获取到的服务列表按照过滤规则过滤。

 

- 通过 Eureka 的分区规则对服务实例进行过滤。
- 比较服务实例的通信失败数和并发连接数来剔除不够健康的实例。
- 根据所属区域过滤出同区域的服务实例。



### 3.4 服务列表更新 ServerListUpdater

服务列表更新就是 Ribbon 会从注册中心获取最新的注册表信息。是由这个接口 ServerListUpdater 定义的更新操作。而它有两个实现类，也就是有两种更新方式：

 

- 通过定时任务进行更新。由这个实现类 PollingServerListUpdater 做到的。
- 利用 Eureka 的事件监听器来更新。由这个实现类 EurekaNotificationServerListUpdater 做到的。



### 3.5 心跳检测 Ping

IPing 接口类用来检测哪些服务可用。如果不可用了，就剔除这些服务。

 

实现类主要有这几个：PingUrl、PingConstant、NoOpPing、DummyPing、NIWSDiscoveryPing。

 

心跳检测策略对象 IPingStrategy，默认实现是轮询检测。



### 3.6 负载均衡策略 Rule

Ribbon 的负载均衡策略和之前讲过的负载均衡策略有部分相同，先来个全面的图，看下 Ribbon 有哪几种负载均衡策略。

 

![img](https://static001.geekbang.org/infoq/b1/b1409154ebc64ac653522241e9baa234.png)

 

再来看下 Ribbon 源码中关于均衡策略的 UML 类图。

 

![img](https://static001.geekbang.org/infoq/ea/ea8d993917183ef28feab1ad81d0b1c4.png)

 

由图可以看到，主要由以下几种均衡策略：

 

- **线性轮询均衡** （RoundRobinRule）：轮流依次请求不同的服务器。优点是无需记录当前所有连接的状态，无状态调度。
- **可用服务过滤负载均衡**（AvailabilityFilteringRule）：过滤多次访问故障而处于断路器状态的服务，还有过滤并发连接数量超过阈值的服务，然后对剩余的服务列表按照轮询策略进行访问。默认情况下，如果最近三次连接均失败，则认为该服务实例断路。然后保持 30s 后进入回路关闭状态，如果此时仍然连接失败，那么等待进入关闭状态的时间会随着失败次数的增加呈指数级增长。
- **加权响应时间负载均衡**（WeightedResponseTimeRule）：为每个服务按响应时长自动分配权重，响应时间越长，权重越低，被选中的概率越低。
- **区域感知负载均衡**（ZoneAvoidanceRule）：更倾向于选择发出调用的服务所在的托管区域内的服务，降低延迟，节省成本。Spring Cloud Ribbon 中默认的策略。
- **重试负载均衡**（RetryRule)：通过轮询均衡策略选择一个服务器，如果请求失败或响应超时，可以选择重试当前服务节点，也可以选择其他节点。
- **高可用**（Best Available)：忽略请求失败的服务器，尽量找并发比较低的服务器。注意：这种会给服务器集群带来成倍的压力。
- **随机负载均衡**（RandomRule）：随机选择服务器。适合并发比较大的场景。



## 四、 Ribbon 拦截请求的原理

本文最开始提出了一个问题：负载均衡器如何将客户端请求进行拦截然后选择服务器进行转发？

 

结合上面介绍的 Ribbon 核心组件，我们可以画一张原理图来梳理下 Ribbon 拦截请求的原理：

 

![img](https://static001.geekbang.org/infoq/dd/dd4e11136936527d83ca5f575a26a445.png)

 

第一步：Ribbon 拦截所有标注`@loadBalance`注解的 RestTemplate。RestTemplate 是用来发送 HTTP 请求的。

 

第二步：将 Ribbon 默认的拦截器 LoadBalancerInterceptor 添加到 RestTemplate 的执行逻辑中，当 RestTemplate 每次发送 HTTP 请求时，都会被 Ribbon 拦截。

 

第三步：拦截后，Ribbon 会创建一个 ILoadBalancer 实例。

 

第四步：ILoadBalancer 实例会使用 RibbonClientConfiguration 完成自动配置。就会配置好 IRule，IPing，ServerList。

 

第五步：Ribbon 会从服务列表中选择一个服务，将请求转发给这个服务。



## 五、Ribbon 初始化的原理

当我们去剖析 Ribbon 源码的时候，需要找到一个突破口，而 @LoadBalanced 注解就是一个比较好的入口。

 

先来一张 Ribbon 初始化的流程图：

 

![img](https://static001.geekbang.org/infoq/58/5859a8c2b839ec29974221c9a66f277a.png)

 

添加注解的代码如下所示：

 

```
@LoadBalanced
@Bean
public RestTemplate getRestTemplate() {
  return new RestTemplate();
}
```

 

第一步：Ribbon 有一个自动配置类 LoadBalancerAutoConfiguration，SpringBoot 加载自动配置类，就会去初始化 Ribbon。

 

第二步：当我们给 RestTemplate 或者 AsyncRestTemplate 添加注解后，Ribbon 初始化时会收集加了 @LoadBalanced 注解的 RestTemplate 和 AsyncRestTemplate ，把它们放到一个 List 里面。

 

第三步：然后 Ribbon 里面的 RestTemplateCustomizer 会给每个 RestTemplate 进行定制化，也就是加上了拦截器：LoadBalancerInterceptor。

 

第四步：从 Eureka 注册中心获取服务列表，然后存到 Ribbon 中。

 

第五步：加载 YMAL 配置文件，配置好负载均衡配置，创建一个 ILoadbalancer 实例。



## 六、Ribbon 同步服务列表原理

Ribbon 首次从 Eureka 获取全量注册表后，就会隔一定时间获取注册表。原理图如下：

 

![img](https://static001.geekbang.org/infoq/93/93185dc9da2fbd66590b7be0a23ddad3.png)

 

之前我们提到过 Ribbon 的核心组件 ServerListUpdater，用来同步注册表的，它有一个实现类 PollingServerListUpdater ，专门用来做定时同步的。默认1s 后执行一个 Runnable 线程，后面就是每隔 30s 执行 Runnable 线程。这个 Runnable 线程就是去获取 Eureka 注册表的。



## 七、Eureka 心跳检测的原理

我们知道 Eureka 注册中心是通过心跳检测机制来判断服务是否可用的，如果不可用，可能会把这个服务摘除。为什么是可能呢？因为 Eureka 有自我保护机制，如果达到自我保护机制的阀值，后续就不会自动摘除。

 

这里我们可以再复习下 Eureka 的自我保护机制和服务摘除机制。

 

- **Eureka 心跳机制**：每个服务每隔 30s 自动向 Eureka Server 发送一次心跳，Eureka Server 更新这个服务的最后心跳时间。如果 180 s 内（版本1.7.2 bug ）未收到心跳，则任务服务故障了。
- **Eureka 自我保护机制**：如果上一分钟实际的心跳次数，比我们期望的心跳次数要小，就会触发自我保护机制，不会摘除任何实例。期望的心跳次数：服务实例数量 * 2 * 0.85。
- **Eureka 服务摘除机制**：不是一次性将服务实例摘除，每次最多随机摘除 15%。如果摘除不完，1 分钟之后再摘除。

 

说完 Eureka 的心跳机制和服务摘除机制后，我们来看下 Ribbon 的心跳机制。



## 八、Ribbon 心跳检测的原理

Ribbon 的心跳检测原理和 Eureka 还不一样，**Ribbon 不是通过每个服务向 Ribbon 发送心跳或者 Ribbon 给每个服务发送心跳来检测服务是否存活的**。

 

先来一张图看下 Ribbon 的心跳检测机制：

 

![img](https://static001.geekbang.org/infoq/bb/bbc288b0e40adfbf421cc9aa051e49d3.png)

 

Ribbon 心跳检测原理：对自己本地缓存的 Server List 进行遍历，看下每个服务的状态是不是 UP 的。具体的代码就是 isAlive 方法。

 

核心代码：

 

```
isAlive = status.equals(InstanceStatus.UP);
```

 

那么多久检测一次呢？

 

默认每隔 30s 执行以下 PingTask 调度任务，对每个服务执行 isAlive 方法，判断下状态。



## 九、Ribbon 常用配置项



### 9.1 禁用 Eureka

```
# 禁用 Eureka
ribbon.eureka.enabled=false
```

 

服务注册列表默认是从 Eureka 获取到的，如果不想使用 Eureka，可以禁用掉。然后我们需要手动配置服务列表。



### 9.2 配置服务列表

```
ribbon-config-passjava.ribbon.listOfServers=localhost:8081,localhost:8083
```

 

这个配置是针对具体服务的，前缀就是服务名称，配置完之后就可以和之前一样使用服务名称来调用接口了。



### 9.3 其他配置项

![img](https://static001.geekbang.org/infoq/21/21c3db938e5f5e37dfdb3a91bbf563b1.png)



## 十、总结

本篇深入讲解了 Spring Cloud 微服务中 负载均衡组件 Ribbon 架构原理，分为几大块：

 

- Ribbon 的六大核心组件
- Ribbon 如何拦截请求并进行转发的。
- Ribbon 初始化的原理。
- Ribbon 如何同步 Eureka 注册表的原理。
- Eureka 和 Ribbon 两种 心跳检测的原理
- Ribbon 的常用配置项。