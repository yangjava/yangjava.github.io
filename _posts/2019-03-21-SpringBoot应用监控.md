---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot应用监控
在企业级应用中，对系统进行运行状态监控通常是必不可少的。Spring Boot提供了Actuator模块实现应用的监控与管理，对应的起步依赖是spring-boot-starter-actuator。

## Actuator简介
在实际的生产系统中，怎样知道应用运行良好呢？往往需要对系统实际运行的情况（例如cpu、io、disk、db、业务功能等指标）进行监控运维，这需要耗费不少精力。

在SpringBoot中，完全不需要面对这样的难题。Spring Boot Actuator提供了众多HTTP接口端点（Endpoint），其中包含了丰富的Spring Boot应用程序运行时的内部状态信息。同时，我们还可以自定义监控端点，实现灵活定制。

Actuator是spring boot提供的对应用系统的自省和监控功能，Actuator对应用系统本身的自省功能，可以让我们方便快捷地实现线上运维监控的工作。这有点儿像DevOps。通过Actuator，可以使用数据化的指标去度量应用的运行情况。比如查看服务器的磁盘、内存、CPU等信息，系统运行了多少线程、gc的情况、运行状态等。

spring-boot-actuator模块提供了一个监控和管理生产环境的模块，可以使用http、jmx、ssh、telnet等管理和监控应用，提供了应用的审计（Auditing）、健康（Health）状态信息、数据采集（Metrics Gathering）统计等监控运维的功能。同时，我们可以扩展Actuator端点自定义监控指标。这些指标都是以JSON接口数据的方式呈现。而使用Spring Boot Admin可以实现这些JSON接口数据的界面展现。

这个模块是一个采集应用内部信息暴露给外部的模块，上述的功能都可以通过HTTP 和 JMX 访问。

因为暴露内部信息的特性，Actuator 也可以和一些外部的应用监控系统整合（Prometheus, Graphite, DataDog, Influx, Wavefront, New Relic等）。

Actuator使用Micrometer与这些外部应用程序监视系统集成。这样一来，只需很少的配置即可轻松集成外部的监控系统。

Micrometer 为 Java 平台上的性能数据收集提供了一个通用的 API，应用程序只需要使用 Micrometer 的通用 API 来收集性能指标即可。  

Micrometer 会负责完成与不同监控系统的适配工作。这就使得切换监控系统变得很容易。

## 启用Actuator
在Spring Boot项目中添加Actuator起步依赖即可启用Actuator功能。在Gradle项目配置文件build.gradle中添加如下代码：
```
dependencies {
    compile('org.springframework.boot:spring-boot-starter-actuator')
    ...
}
```
使用Maven配置
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

在Spring Boot 2.0中，Actuator模块做了较大更新，默认启用的端点如下：
```
{
    _links: {
        self: {
        href: "http:// 127.0.0.1:8008/actuator",
        templated: false
        },
        health: {
        href: "http:// 127.0.0.1:8008/actuator/health",
        templated: false
            },
        info: {
        href: "http:// 127.0.0.1:8008/actuator/info",
        templated: false
        }
    }
}

```
如果想启用所有端点，在application.properties中按如下配置：
```
#endpoints in Spring Boot 2.0
#http:// 127.0.0.1:8008/actuator
management.endpoints.enabled-by-default=true
management.endpoints.web.expose=*
```

## 常用的Actuator端点
Actuator监控分成两类：原生端点和用户自定义端点。

原生端点是Actuator组件内置的，在应用程序中提供了众多Web接口。

通过它们了解应用程序运行时的内部情况，原生端点可以分成3类：
- 应用配置类
可以查看应用在运行期的静态信息，比如自动配置信息、加载的Spring Bean信息、YML文件配置信息、环境信息、请求映射信息
- 度量指标类
主要是运行期的动态信息，如堆栈、请求连接、健康状态、系统性能等。
- 操作控制类
主要是指shutdown，用户可以发送一个请求将应用的监控功能关闭。









## 自定义Actuator端点
Spring Boot支持自定义端点，只需要在我们定义的类中使用@Endpoint、@JmxEndpoint、@WebEndpoint等注解，实现对应的方法即可定义一个Actuator中的自定义端点。

从Spring Boot 2.x版本开始，Actuator支持CRUD（增删改查）模型，而不是旧的RW（读/写）模型。我们可以按照3种策略来自定义：

- 使用@Endpoint注解，同时支持JMX和HTTP方式。
- 使用@JmxEndpoint注解，只支持JMX技术。
- 使用@WebEndpoint注解，只支持HTTP。



















## 什么是Actuator?
致动器（actuator）是2018年公布的计算机科学技术名词。

百度百科在新窗口打开的解释如下： 致动器能将某种形式的能量转换为机械能的驱动装置。如热致动器、磁致动器等，在磁盘中是指将电能转换为机械能并带动磁头运动的装置。

官网给的解释是：An actuator is a manufacturing term that refers to a mechanical device for moving or controlling something. Actuators can generate a large amount of motion from a small change.

从上述的解释不难知道Spring 命名这个组件为Actuator，就是为了提供监测程序的能力。# 什么是Spring Boot Actuator？

Spring Boot Actuator可以帮助你监控和管理Spring Boot应用，比如健康检查、审计、统计和HTTP追踪、获取线程状态等。Actuator使用Micrometer来整合上面提到的外部应用监控系统。这使得只要通过非常小的配置就可以集成任何应用监控系统。


## 什么是Spring Boot Actuator？

Spring Boot Actuator提供了对SpringBoot应用程序（可以是生产环境）监视和管理的能力， 可以选择通过使用HTTP Endpoint或使用JMX来管理和监控SpringBoot应用程序。

## 什么是Actuator Endpoints？
Spring Boot Actuator 允许你通过Endpoints对Spring Boot进行监控和交互。

Spring Boot 内置的Endpoint包括（两种Endpoint： WEB和JMX， web方式考虑到安全性默认只开启了/health）：

当然你也可以自己定义暴露哪些endpoint

JMX
```xml
management:
  endpoints:
    jmx:
      exposure:
        include: "health,info"
```
WEB
```xml
management:
  endpoints:
    web:
      exposure:
        include: "*"
        exclude: "env,beans"
```

### 内置端点
SpringBoot 中默认提供的常用内置端点如下：

| ID  |   Endpoint功能描述  |
|-----|-----|
|     |     |
|     |     |
|     |     |
|     |     |
|     |     |
|     |     |
|     |     |
|     |     |
|     |     |



### HTTP Endpoints 监控

每个端点都有一个唯一的 id，访问时可以通过如下地址进行访问：http:ip:port/{id}（SpringBoot 1.x ）。

而在 SpringBoot 2.x 版本中，默认新增了一个 /actuator 作为基本路径，访问地址则对应为：http:ip:port/actuator/{id}。

虽然说这里的大部分端点都是默认开启的，但是默认暴露（允许对外访问）的只有 health 和 info 端点，所以如果需要允许端点对外暴露，可以通过如下配置（如果想要暴露所有的端点，则可以直接配置 "*" ）：
```yaml
management:
  endpoints:
    web:
      exposure:
        include: [health,info,mappings] //或者直接配置 "*"
```

### JMX 监控
JMX 全称为 Java Management Extensions，即 Java 管理扩展。它提供了对 Java 应用程序和 JVM 的监控管理。

通过 JMX 我们可以监控服务器中各种资源的使用情况以及线程，内存和 CPU 等使用情况。

## SpringBoot监控实战

### 引入actuator包
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
此时添加监控，但是没有暴漏端点，不会有监控

### yml配置

```yaml
server:
  port: 8080

management:
  endpoints:
    enabled-by-default: false
    web:
      base-path: /monitor
      exposure:
        include: 'info,health,env,beans'
  endpoint:
    info:
      enabled: true
    health:
      enabled: true
    env:
      enabled: true
    beans:
      enabled: true

```
启动时日志
```text
Exposing 4 endpoint(s) beneath base path '/monitor'
```
上述配置只暴露info,health,env,beans四个endpoints, web通过可以http://localhost:8080/monitor访问
```text
{"_links":{"self":{"href":"http://localhost:8080/monitor","templated":false},"beans":{"href":"http://localhost:8080/monitor/beans","templated":false},"health":{"href":"http://localhost:8080/monitor/health","templated":false},"health-path":{"href":"http://localhost:8080/monitor/health/{*path}","templated":true},"info":{"href":"http://localhost:8080/monitor/info","templated":false},"env":{"href":"http://localhost:8080/monitor/env","templated":false},"env-toMatch":{"href":"http://localhost:8080/monitor/env/{toMatch}","templated":true}}}
```

### 所有监控端点
```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```
web通过可以http://localhost:8080/actuator访问

输出日志
```text
{"_links":{"self":{"href":"http://localhost:8080/actuator","templated":false},"beans":{"href":"http://localhost:8080/actuator/beans","templated":false},"caches-cache":{"href":"http://localhost:8080/actuator/caches/{cache}","templated":true},"caches":{"href":"http://localhost:8080/actuator/caches","templated":false},"health":{"href":"http://localhost:8080/actuator/health","templated":false},"health-path":{"href":"http://localhost:8080/actuator/health/{*path}","templated":true},"info":{"href":"http://localhost:8080/actuator/info","templated":false},"conditions":{"href":"http://localhost:8080/actuator/conditions","templated":false},"configprops":{"href":"http://localhost:8080/actuator/configprops","templated":false},"configprops-prefix":{"href":"http://localhost:8080/actuator/configprops/{prefix}","templated":true},"env":{"href":"http://localhost:8080/actuator/env","templated":false},"env-toMatch":{"href":"http://localhost:8080/actuator/env/{toMatch}","templated":true},"loggers":{"href":"http://localhost:8080/actuator/loggers","templated":false},"loggers-name":{"href":"http://localhost:8080/actuator/loggers/{name}","templated":true},"heapdump":{"href":"http://localhost:8080/actuator/heapdump","templated":false},"threaddump":{"href":"http://localhost:8080/actuator/threaddump","templated":false},"metrics-requiredMetricName":{"href":"http://localhost:8080/actuator/metrics/{requiredMetricName}","templated":true},"metrics":{"href":"http://localhost:8080/actuator/metrics","templated":false},"scheduledtasks":{"href":"http://localhost:8080/actuator/scheduledtasks","templated":false},"mappings":{"href":"http://localhost:8080/actuator/mappings","templated":false}}}
```

### health 端点

health 断点默认只是展示当前应用健康信息，但是我们可以通过另一个配置打开详细信息，这样不仅仅会监控当前应用，还会监控与当前应用相关的其他第三方应用，如 Redis。
```yaml
management:
  endpoint:
    health:
      show-details: always
```
这个配置打开之后，我们连接上 Redis 之后再次访问 health 端点，就可以展示 Redis 服务的健康信息了：
```text
{"status":"DOWN","components":{"diskSpace":{"status":"UP","details":{"total":368642617344,"free":45893025792,"threshold":10485760,"exists":true}},"ping":{"status":"UP"},"redis":{"status":"DOWN","details":{"error":"org.springframework.data.redis.RedisConnectionFailureException: Unable to connect to Redis; nested exception is io.lettuce.core.RedisConnectionException: Unable to connect to localhost:6379"}}}}
```
添加了`spring-boot-starter-data-redis`但是服务没有启动Redis，项目也没有使用Redis，Redis状态也是Down。


### loggers 端点