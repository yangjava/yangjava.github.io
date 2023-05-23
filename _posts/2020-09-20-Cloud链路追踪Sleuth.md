---
layout: post
categories: [SpringCloud]
description: none
keywords: SpringCloud
---
# 服务链路追踪Sleuth
Spring Cloud提供的链路监控组件Spring Cloud Sleuth，这个组件提供了分布式链路追踪的解决方案，用以追踪微服务系统中一次请求的完整过程。

## 基础应用
Spring Cloud Sleuth为服务之间的调用提供链路追踪，通过Sleuth可以很清楚地了解到一个服务请求经过了哪些服务，每个服务处理花费了多长，从而可以很方便地理清各微服务间的调用关系。此外Sleuth还可以进行耗时分析，通过Sleuth可以很方便地了解到每个采样请求的耗时，从而分析出哪些服务调用比较耗时；Sleuth可以可视化错误，对于程序未捕捉的异常，可以通过集成Zipkin服务界面看到；通过Sleuth还能对链路优化，对于调用比较频繁的服务实施一些优化措施。

## 特性
Spring Cloud Sleuth具有如下的特性：
- 提供对常见分布式跟踪数据模型的抽象：Traces、Spans（形成DAG）、注解和键值注解。Spring Cloud Sleuth虽然基于HTrace，但与Zipkin（Dapper）兼容。
- Sleuth记录了耗时操作的信息以辅助延时分析。通过使用Sleuth，可以定位应用中的耗时原因。
- 为了不写入太多的日志，以至于使生产环境的应用程序崩溃，Sleuth做了如下的工作：
  - 生成调用链的结构化数据。
  - 包括诸如自定义的织入层信息，比如HTTP。
  - 包括采样率配置以控制日志的容量。
  - 查询和可视化完全兼容Zipkin。
- 为Spring应用织入通用的组件，如Servlet过滤器、Zuul过滤器和OpenFeign客户端等等。
- Sleuth可以在进程之间传播上下文（也称为背包）。因此，如果在Span上设置了背包元素，它将通过HTTP或消息传递到下游，发送到其他进程。
- Spring Cloud Sleuth实现了OpenTracing的规范。OpenTracing通过提供平台无关、厂商无关的API，使得开发人员能够方便地添加（或更换）追踪系统的实现。

## 项目准备
下面通过两个服务之间的相互调用，演示链路监控的场景。前期需要准备的项目有三个：
- Eureka Server：服务注册Server，作为其他客户端服务的注册中心。
- Sleuth Client A：客户端服务A，注册到Eureka Server。对外提供两个接口，一个接口暴露给客户端服务A，另一个接口调用客户端服务B，客户端服务之间通过OpenFeign进行HTTP调用。
- Sleuth Client B：同上。

### Eureka Server
首先，依赖需要引入Netflix Eureka Server，如下所示：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```
其次，配置Eureka Server的服务端信息，如下所示：
```yaml
spring:
    application:
        name: eureka-server-standalone
server:
    port: 8761
eureka:
    instance:
        hostname: standalone
        instance-id: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}
```

其中，我们配置了单机模式的Eureka Server，服务器的端口号为8761。

最后，添加应用程序的入口类，如下所示：
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}

```
如上三步完成，我们的注册服务即可以正常启动。

### Sleuth Client A
首先，也是引入客户端服务A所需要的依赖，如下所示：
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```
需要的依赖为：注册客户端、声明式客户端组件OpenFeign和Spring Boot Web包。

其次，客户端服务A的配置文件如下所示：
```yaml
eureka:
    instance:
        instance-id: ${spring.application.name}:${vcap.application.instance_id:$ {spring.application.instance_id:${random.value}}}
    client:
        service-url:
            defaultZone: http://localhost:8761/eureka/, http://localhost:8762/eureka/

spring:
    application:
        name: service-a
server:
    port: 9002
```
上述配置声明了服务名，OpenFeign通过服务名进行服务实例调用，指定了客户端服务A的端口号9002。A还需要注册到Eureka Server。

然后提供两个接口，其实现如下所示：
```java
@RestController
@RequestMapping("/api")
public class ServiceAController {
    @Autowired
    ServiceBClient serviceBClient;

    @GetMapping("/service-a")
    public String fromServiceA(){
        return "from serviceA";
    }
    @GetMapping("/service-b")
    public String fromServiceB(){
        return serviceBClient.fromB();
    }
}
```
服务A提供了/service-b接口，可以调用服务B，Feign客户端的实现如下所示，/service-a接口用于给服务B调用。
```java
@FeignClient("service-b")
public interface ServiceBClient {

    @RequestMapping(value = "/api/service-b")
    public String fromB();
}
```
OpenFeign客户端的注解中，指定了要调用的服务B的serviceId为service-b。

最后，应用程序的入口类如下：
```java
@SpringBootApplication
@EnableFeignClients
public class Chapter13ClientServiceaApplication {

    public static void main(String[] args) {
        SpringApplication.run(Chapter13ClientServiceaApplication.class, args);
    }
}
```
@EnableFeignClients注解会扫描那些声明为Feign客户端的接口。

至此，客户端服务A的实现大功告成。客户端服务B与A的实现基本类似，A与B之间实现互调。

## Spring Cloud Sleuth独立实现
当Spring Cloud Sleuth单独使用时，通过日志关联的方式将请求的链路串联起来，分别启动之前准备的三个服务，并访问地址http://localhost:9002/api/service-b，服务A调用了服务B，成功返回响应后，我们看一下控制台的Sleuth相关日志。

服务A日志如下所示：
```
INFO [service-a，7feec0479597d1b9,578bef9ed3901d9b,false]
```
服务B日志如下所示：
```
DEBUG [service-b,7feec0479597d1b9,578bef9ed3901d9b,false]
```
日志中输出的四部分信息分别为：
- appname：service-a/b，是设置的应用名称。
- traceId：7feec0479597d1b9，Spring Cloud Sleuth生成的TraceId，一次请求的唯一标识。
- spanId：578bef9ed3901d9b，Spring Cloud Sleuth生成的SpanId，标识一个基本的工作单元，此处是Feign进行的HTTP调用。
- exportable：false，是否将数据导出，如果只是想要在Span中封装一些操作并将其写日志时，此时就不需要将Span导出。

从上面的日志及输出信息的解释可以看出，发出的HTTP请求到达A后，A向B发起调用，这是一次完整的调用过程，TraceId在服务A和B中是相同的。Spring Cloud Sleuth在一次请求中，生成了如上四部分信息，TraceId唯一，可以有多个SpanId。当然数据是可以导出进行分析的，如将日志信息导出到Elasticsearch，使用日志分析工具如Kibana、Splunk或者其他工具，比较常用的组合是ELK：Elasticsearch+Logstash+Kibana。然而即便这样，还是需要我们自行对一些维度的数据进行分析，其实这些监控的数据维度基本比较确定，在业界也有一些优秀组件，如Zipkin、Pinpoint等，利用这些工具将会避免不必要的分析工作。













