---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot集成监控
大部分微服务应用都是基于 SpringBoot 来构建，所以了解 SpringBoot 的监控特性是非常有必要的，而 SpringBoot 也提供了一些特性来帮助我们监控应用。

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