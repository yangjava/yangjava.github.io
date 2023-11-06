---
layout: post
categories: [SpringCloud]
description: none
keywords: SpringCloud
---
# SpringCloud网关Gateway
Spring Cloud Gateway基于Spring Boot 2，是Spring Cloud的全新项目，该项目提供了一个构建在Spring生态之上的API网关。Spring Cloud Gateway旨在提供一种简单而有效的途径来转发请求，并为它们提供横切关注点，例如：安全性、监控/指标和弹性。

## 基础应用
Spring Cloud Gateway具有如下特征：
- 基于Java 8编码；
- 支持Spring Framework 5；
- 支持Spring Boot 2；
- 支持动态路由；
- 支持内置到Spring Handler映射中的路由匹配；
- 支持基于HTTP请求的路由匹配（Path、Method、Header、Host等）；
- 过滤器作用于匹配的路由；
- 过滤器可以修改下游HTTP请求和HTTP响应（增加/修改头部、增加/修改请求参数、改写请求路径等）；
- 通过API或配置驱动；
- 支持Spring Cloud DiscoveryClient配置路由，与服务发现与注册配合使用。

在Finchley正式版之前，Spring Cloud推荐的网关是Netflix提供的Zuul（笔者这里指的都是Zuul 1.x，是一个基于阻塞I/O的API Gateway）。与Zuul相比，Spring Cloud Gateway建立在Spring Framework 5、Project Reactor和Spring Boot 2之上，使用非阻塞API。Spring Cloud Gateway还支持WebSocket，并且与Spring紧密集成，拥有更好的开发体验。Zuul基于Servlet 2.5，使用阻塞架构，它不支持任何长连接，如WebSocket。Zuul的设计模式和Nginx较像，每次I/O操作都是从工作线程中选择一个执行，请求线程被阻塞直到工作线程完成，但是差别是Nginx用C++实现，Zuul用Java实现，而JVM本身会有第一次加载较慢的情况，使得Zuul的性能相对较差。Zuul已经发布了Zuul 2.x，基于Netty、非阻塞、支持长连接，但Spring Cloud目前还没有整合。Zuul 2.x的性能肯定会较Zuul 1.x有较大提升。在性能方面，根据官方提供的基准（benchmark）测试，Spring Cloud Gateway的RPS（每秒请求数）是Zuul的1.6倍。综合来说，Spring Cloud Gateway在提供的功能和实际性能方面，表现都很优异。

## Gateway介绍
Spring Cloud Gateway是Spring官方基于Spring 5.0、Spring Boot 2.0和Project Reactor等技术开发的网关，Spring Cloud Gateway旨在为微服务架构提供一种简单有效的、统一的API路由管理方式。Spring Cloud Gateway作为Spring Cloud生态系中的网关，其目标是替代Netflix Zuul，它不仅提供统一的路由方式，并且基于Filter链的方式提供了网关基本的功能，例如：安全、监控/埋点和限流等。

Spring Cloud Gateway依赖Spring Boot和Spring WebFlux，基于Netty运行。它不能在传统的servlet容器中工作，也不能构建成war包。

## Gateway核心概念
在Spring Cloud Gateway中有如下几个核心概念需要我们了解：
- Route
Route是网关的基础元素，由ID、目标URI、断言、过滤器组成。当请求到达网关时，由Gateway Handler Mapping通过断言进行路由匹配（Mapping），当断言为真时，匹配到路由。
- Predicate
Predicate是Java 8中提供的一个函数。输入类型是Spring Framework ServerWebExchange。它允许开发人员匹配来自HTTP的请求，例如请求头或者请求参数。简单来说它就是匹配条件。
- Filter
Filter是Gateway中的过滤器，可以在请求发出前后进行一些业务上的处理。

## Gateway快速上手
网关服务提供路由配置、路由断言和过滤器等功能。

创建一个Spring Boot的Maven项目，增加Spring Cloud Gateway的依赖。需要引入的依赖如下所示：
```xml
    <dependencies>
        <!--网关-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
    </dependencies>
```
启动类就按Spring Boot的方式即可，无须添加额外的注解。
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

## 路由转发示例
下面来实现一个最简单的转发功能——基于Path的匹配转发功能。

Gateway的路由配置对yml文件支持比较好，我们在resources下建一个application.yml的文件，内容如下：
```yaml
server:
  port: 9000
spring:
  cloud:
    gateway:
      routes:
        - id: path_route
          uri: http://localhost:8080/
          predicates:
            - Path=/hello
```
我们当前网关服务是gateway,端口号是localhost:9000。当你访问http://localhost:9000/hello 的时候就会转发到http://localhost:8080/hello 。

如果我们要支持多级Path，配置方式跟Zuul中一样，在后面加上两个*号即可，比如：
```yaml
server:
  port: 9000
spring:
  cloud:
    gateway:
      routes:
        - id: path_route
          uri: http://localhost:8080/
          predicates:
            - Path=/hello
        - id: path_route2
          uri: http://localhost:8080/
          predicates:
            - Path=/hello/**
```
这样一来，上面的配置就可以支持多级Path，比如访问http://localhost:9000/hello/world/1的时候就会转发到http://localhost:8080/hello/world/1。

## 整合Eureka路由
添加Eureka Client的依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```
配置基于Eureka的路由：
```yaml
server:
  port: 9000
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://eureka-client-user-service
          predicates:
            - Path=/user/**
  application:
    name: gateway
# 定义实例ID格式
eureka:
  instance:
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```
uri以lb：//开头（lb代表从注册中心获取服务），后面接的就是你需要转发到的服务名称，这个服务名称必须跟Eureka中的对应，否则会找不到服务，错误代码如下：
```
org.springframework.cloud.gateway.support.NotFoundException: Unable to find instance for user-service1
```

## 整合Eureka的默认路由
Zuul默认会为所有服务都进行转发操作，我们只需要在访问路径上指定要访问的服务即可，通过这种方式就不用为每个服务都去配置转发规则，当新加了服务的时候，不用去配置路由规则和重启网关。

在Spring Cloud Gateway中当然也有这样的功能，通过配置即可开启，配置如下：
```yaml
server:
  port: 9000

spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
  application:
    name: gateway
# 定义实例ID格式
eureka:
  instance:
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```
开启之后我们就可以通过地址去访问服务了，格式如下：
```
http://网关地址/服务名称（大写）/**
http://localhost:9000/EUREKA-CLIENT-USER-SERVICE/user/hello
```
这个大写的名称还是有很大的影响，通过源码笔者发现可以做到兼容处理，再增加一个配置即可：
```yaml
spring:
    cloud:
        gateway:
            discovery:
                locator:
                    lowerCaseServiceId: true
```
配置完成之后我们就可以通过小写的服务名称进行访问了，如下所示：
```
http://网关地址/服务名称（小写）/**
http://localhost:9000/eureka-client-user-service/user/hello
```
开启小写服务名称后大写的服务名称就不能使用，两者只能选其一。

配置源码在org.springframework.cloud.gateway.discovery.DiscoveryLocatorProperties类中
```
@ConfigurationProperties("spring.cloud.gateway.discovery.locator")
public class DiscoveryLocatorProperties {
    /**
     *  服务名称小写配置，默认为false
     * 
     */
    private boolean lowerCaseServiceId = false;

}
```

## Spring Cloud Gateway路由断言工厂
Spring Cloud Gateway内置了许多路由断言工厂，可以通过配置的方式直接使用，也可以组合使用多个路由断言工厂。接下来为大家介绍几个常用的路由断言工厂类。

### Path路由断言工厂
Path路由断言工厂接收一个参数，根据Path定义好的规则来判断访问的URI是否匹配。
```yaml
server:
  port: 9000
spring:
  cloud:
    gateway:
      routes:
        - id: path_route
          uri: http://localhost:8080/
          predicates:
            - Path=/user/hello/{detail}
        - id: path_route2
          uri: http://localhost:8080/
          predicates:
            - Path=/user/hello/**
  application:
    name: gateway
# 定义实例ID格式
eureka:
  instance:
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```
如果请求路径为/user/hello/xxx，则此路由将匹配。也可以使用正则，例如/user/hello/**来匹配/user/hello/开头的多级URI。

我们访问本地的网关：http://localhost:9000/user/hello/36185 ，可以看到显示的是http://localhost:8081/user/hello/36185对应的内容。

### Query路由断言工厂
Query路由断言工厂接收两个参数，一个必需的参数和一个可选的正则表达式。
```yaml
spring:
    cloud:
        gateway:
            routes:
            - id: query_route
              uri: http://cxytiandi.com
              predicates:
                - Query=foo, ba.
```
如果请求包含一个值与ba匹配的foo查询参数，则此路由将匹配。bar和baz也会匹配，因为第二个参数是正则表达式。

测试链接：http://localhost：2001/？foo=baz。

### Method路由断言工厂
Method路由断言工厂接收一个参数，即要匹配的HTTP方法。
```yaml
spring:
    cloud:
        gateway:
            routes:
            - id: method_route
              uri: http://baidu.com
              predicates:
                - Method=GET
```

### Header路由断言工厂
Header路由断言工厂接收两个参数，分别是请求头名称和正则表达式。
```yaml
spring:
    cloud:
        gateway:
            routes:
            - id: header_route
              uri: http://example.org
              predicates:
                - Header=X-Request-Id, \d+
```
如果请求中带有请求头名为x-request-id，其值与\d+正则表达式匹配（值为一个或多个数字），则此路由匹配。

更多路由断言工厂的用法请参考官方文档进行学习。

### 自定义路由断言工厂

## Spring Cloud Gateway过滤器工厂
GatewayFilter Factory是Spring Cloud Gateway中提供的过滤器工厂。Spring Cloud Gateway的路由过滤器允许以某种方式修改传入的HTTP请求或输出的HTTP响应，只作用于特定的路由。Spring Cloud Gateway中内置了很多过滤器工厂，直接采用配置的方式使用即可，同时也支持自定义GatewayFilter Factory来实现更复杂的业务需求。

接下来为大家介绍几个常用的过滤器工厂类。

### AddRequestHeader过滤器工厂
通过名称我们可以快速明白这个过滤器工厂的作用是添加请求头。 符合规则匹配成功的请求，将添加X-Request-Foo：bar请求头，将其传递到后端服务中，后方服务可以直接获取请求头信息。
```yaml
server:
  port: 9000
spring:
  cloud:
    gateway:
      routes:
        - id: add_request_header_route
          uri: http://localhost:8081
          filters:
            - AddRequestHeader=X-Request-Foo, Bar
          predicates:
            - Path=/user/hello/**
  application:
    name: gateway
# 定义实例ID格式
eureka:
  instance:
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```
后端服务获取请求头
```
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.servlet.http.HttpServletRequest;

@RestController
public class UserController {
    @GetMapping("/user/hello")
    public String hello(HttpServletRequest request) {
        System.err.println(request.getHeader("X-Request-Foo"));
        return "hello";
    }
}
```

### RemoveRequestHeader过滤器工厂
RemoveRequestHeader是移除请求头的过滤器工厂，可以在请求转发到后端服务之前进行Header的移除操作。
```yaml
server:
  port: 9000
spring:
  cloud:
    gateway:
      routes:
        - id: remove_request_header_route
          uri: http://localhost:8081
          filters:
            - RemoveRequestHeader=X-Request-Foo, Bar
          predicates:
            - Path=/user/hello/**
  application:
    name: gateway
# 定义实例ID格式
eureka:
  instance:
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```

### SetStatus过滤器工厂
SetStatus过滤器工厂接收单个状态，用于设置Http请求的响应码。它必须是有效的Spring Httpstatus（org.springframework.http.HttpStatus）。它可以是整数值404或枚举类型NOT_FOUND。
```yaml
server:
  port: 9000
spring:
  cloud:
    gateway:
      routes:
        - id: set_status_route
          uri: http://localhost:8081
          filters:
            - AddRequestHeader=X-Request-Foo, Bar
            - SetStatus=401
          predicates:
            - Path=/user/hello/**
  application:
    name: gateway
# 定义实例ID格式
eureka:
  instance:
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```

### RedirectTo过滤器工厂
RedirectTo过滤器工厂用于重定向操作，比如我们需要重定向到百度。
```yaml
server:
  port: 9000
spring:
  cloud:
    gateway:
      routes:
        - id: redirect_to_route
          uri: http://localhost:8081
          filters:
            - RedirectTo=302, https://baidu.com
          predicates:
            - Path=/user/hello/**
  application:
    name: gateway
# 定义实例ID格式
eureka:
  instance:
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```

### 自定义Spring Cloud Gateway过滤器工厂
自定义Spring Cloud Gateway过滤器工厂需要继承AbstractGatewayFilterFactory类，重写apply方法的逻辑。

命名需要以GatewayFilterFactory结尾，比如CheckAuthGatewayFilterFactory，那么在使用的时候CheckAuth就是这个过滤器工厂的名称。

自定义过滤器工厂
```
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;

@Component
public class CheckAuth2GatewayFilterFactory extends AbstractGatewayFilterFactory<CheckAuth2GatewayFilterFactory.Config> {

    public CheckAuth2GatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            System.err.println("进入了CheckAuth2GatewayFilterFactory" + config.getName());
            ServerHttpRequest request = exchange.getRequest().mutate()
                    .build();
            return
                    chain.filter(exchange.mutate().request(request).build());
        };
    }

    public static class Config {

        private String name;

        public void setName(String name) {
            this.name = name;
        }

        public String getName() {
            return name;
        }

    }
}
```
拦截器的使用
```yaml
server:
  port: 9000
spring:
  cloud:
    gateway:
      routes:
        - id: check_auth2_route
          uri: http://localhost:8081
          filters:
            - name: CheckAuth2
              args:
                  name: yangjingjing
          predicates:
            - Path=/user/hello/**
  application:
    name: gateway
# 定义实例ID格式
eureka:
  instance:
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```
如果你的配置是Key、Value这种形式的，那么可以不用自己定义配置类，直接继承AbstractNameValueGatewayFilterFactory类即可。

AbstractNameValueGatewayFilterFactory类继承了AbstractGatewayFilterFactory，定义了一个NameValueConfig配置类，NameValueConfig中有name和value两个字段。

我们可以直接使用，AddRequestHeaderGatewayFilterFactory、AddRequestParameterGatewayFilterFactory等都是直接继承的AbstractNameValueGatewayFilterFactory。

继承AbstractNameValueGatewayFilterFactory方式定义过滤器工厂
```
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractNameValueGatewayFilterFactory;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;

@Component
public class CheckAuthGatewayFilterFactory extends AbstractNameValueGatewayFilterFactory {

    @Override
    public GatewayFilter apply(NameValueConfig config) {
        return (exchange, chain) -> {
            System.err.println("进入了CheckAuthGatewayFilterFactory" + config.getName() + "\t" + config.getValue());
            ServerHttpRequest request = exchange.getRequest().mutate()
                    .build();

            return chain.filter(exchange.mutate().request(request).build());
        };
    }

}
```
使用：
```yaml
server:
  port: 9000
spring:
  cloud:
    gateway:
      routes:
        - id: check_auth2_route
          uri: http://localhost:8081
          filters:
            - CheckAuth=yangjingjing,男
          predicates:
            - Path=/user/hello/**
  application:
    name: gateway
# 定义实例ID格式
eureka:
  instance:
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```

## 全局过滤器
全局过滤器作用于所有的路由，不需要单独配置，我们可以用它来实现很多统一化处理的业务需求，比如权限认证、IP访问限制等。

接口定义类org.springframework.cloud.gateway.filter.GlobalFilter
```
public interface GlobalFilter {
    Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}
```
Spring Cloud Gateway自带的GlobalFilter实现类有很多。有转发、路由、负载等相关的GlobalFilter，感兴趣的朋友可以去看下源码自行了解。

我们如何通过定义GlobalFilter来实现我们的业务逻辑？
```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import reactor.core.publisher.Mono;

@Configuration
public class ExampleConfiguration {
    private Logger log = LoggerFactory.getLogger(ExampleConfiguration.class);
    @Bean
    @Order(-1)
    public GlobalFilter a() {
        return (exchange, chain) -> {
            log.info("first pre filter");
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                log.info("third post filter");
            }));
        };
    }

    @Bean
    @Order(0)
    public GlobalFilter b() {
        return (exchange, chain) -> {
            log.info("second pre filter");
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                log.info("second post filter");
            }));
        };
    }

    @Bean
    @Order(1)
    public GlobalFilter c() {
        return (exchange, chain) -> {
            log.info("third pre filter");
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                log.info("first post filter");
            }));
        };
    }
}
```
上面定义了3个GlobalFilter，通过@Order来指定执行的顺序，数字越小，优先级越高。下面就是输出的日志，从日志就可以看出执行的顺序：
```
com.cloud.ExampleConfiguration           : first pre filter
com.cloud.ExampleConfiguration           : second pre filter
com.cloud.ExampleConfiguration           : third pre filter
com.cloud.ExampleConfiguration           : first post filter
com.cloud.ExampleConfiguration           : second post filter
com.cloud.ExampleConfiguration           : third post filter
```
当GlobalFilter的逻辑比较多时，笔者还是推荐大家单独写一个GlobalFilter来处理，比如我们要实现对IP的访问限制，即不在IP白名单中就不能调用的需求。

单独定义只需要实现GlobalFilter、Ordered两个接口就可以了
```java
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.nio.charset.StandardCharsets;

@Component
public class IPCheckFilter implements GlobalFilter, Ordered {

    @Override
    public int getOrder() {
        return 0;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        HttpHeaders headers = exchange.getRequest().getHeaders();
        // 此处写得非常绝对，只作演示用，实际中需要采取配置的方式
        if (getIp(headers).equals("127.0.0.1")) {
            ServerHttpResponse response = exchange.getResponse();
            byte[] datas = "进入黑名单非法请求".getBytes(StandardCharsets.UTF_8);
            DataBuffer buffer = response.bufferFactory().wrap(datas);
            response.setStatusCode(HttpStatus.UNAUTHORIZED);
            response.getHeaders().add("Content-Type", "application/json;charset=UTF-8");
            return response.writeWith(Mono.just(buffer));
        }
        return chain.filter(exchange);
    }

    // 这里从请求头中获取用户的实际IP,根据Nginx转发的请求头获取
    private String getIp(HttpHeaders headers) {
        return "127.0.0.1";
    }

}
```
过滤的使用虽然比较简单，但作用很大，可以处理很多需求，上面讲的IP认证拦截只是冰山一角，更多的功能需要我们自己基于过滤器去实现。

## 核心全局过滤器
GlobalFilter接口与GatewayFilter具有相同的签名。这些是特殊的过滤器，有条件地应用于所有路由。

GlobalFilter拦截式的契约，Web请求的链式处理，可用于实现横切、应用程序无关的需求，如Security、Timeout等。

## 组合全局过滤器和网关过滤器
当请求匹配路由时，过滤web处理程序将GlobalFilter的所有实例和GatewayFilter的所有特定路由实例添加到过滤器链中。这个组合过滤器链是由
org.springframework.core.Ordered接口排序的，你可以通过实现getOrder()方法来设置它。

### Forward Routing Filter
ForwardRoutingFilter在交换属性 ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR中查找URI。如果URL是前向模式(例如forward:///xxxx)，它会使用Spring DispatcherHandler来处理请求。请求URL中的路径部分被前向URL中的路径覆盖。未修改的原始URL被附加到ServerWebExchangeUtils中的列表中ServerWebExchangeUtils.GATEWAY_ORIGINAL_REQUEST_URL_ATTR属性。

### 负载均衡过滤器
ReactiveLoadBalancerClientFilter在名为 ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR的exchange属性中查找URI。如果URL具有lb模式(例如lb://myservice)，它会使用Spring Cloud ReactorLoadBalancer将名称解析为实际的主机和端口，并替换相同属性中的URI。未修改的原始URL被附加到ServerWebExchangeUtils中的列表中
GATEWAY_ORIGINAL_REQUEST_URL_ATTR属性。过滤器还在ServerWebExchangeUtils中查找 GATEWAY_SCHEME_PREFIX_ATTR属性，查看它是否等于lb。如果等于，则应用相同的规则。

## 限流实战
开发高并发系统时有三把利器用来保护系统：缓存、降级和限流。API网关作为所有请求的入口，请求量大，我们可以通过对并发访问的请求进行限速来保护系统的可用性。

Spring Cloud Gateway官方就提供了RequestRateLimiterGatewayFilterFactory这个类，适用Redis和lua脚本实现了令牌桶的方式。具体实现逻辑在RequestRateLimiterGatewayFilterFactory类中。

目前限流提供了基于Redis的实现，我们需要增加对应的依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```
我们可以通过KeyResolver来指定限流的Key，比如我们需要根据用户来做限流，或是根据IP来做限流等。

##  增加限流标识
限流通常要根据某一个参数值作为参考依据来限流，比如每个IP每秒钟只能访问2次，此时是以IP为参考依据，我们创建根据IP限流的对象，该对象需要实现接口KeyResolver
```
public class IpKeyResolver implements KeyResolver {

    /***
     * 根据IP限流
     * @param exchange
     * @return
     */
    @Override
    public Mono<String> resolve(ServerWebExchange exchange) {
        return Mono.just(exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
    }
}
```

### 常见的限流方式
常见的限流方式例如 IP限流  用户限流
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import reactor.core.publisher.Mono;

@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }

    // IP限流
    @Bean
    @Primary
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
    }

    // 用户限流
    @Bean
    KeyResolver userKeyResolver() {
        return exchange ->
                Mono.just(exchange.getRequest().getQueryParams().getFirst("userId"));
    }

    // 请求URL限流
    @Bean
    KeyResolver apiKeyResolver() {
        return exchange -> Mono.just(exchange.getRequest().getPath().value());
    }

}
```
然后配置限流的过滤器信息：
```

```
- filter名称必须是RequestRateLimiter。
- redis-rate-limiter.replenishRate：允许用户每秒处理多少个请求。
- redis-rate-limiter.burstCapacity：令牌桶的容量，允许在1s内完成的最大请求数。
- key-resolver：使用SpEL按名称引用bean。

注意点：
由于上述我们配置了三个KeyResolver，gateway启动的时候必须指定哪一个KeyResolver为主，否则将会报如下图错误：
```
Parameter 1 of method requestRateLimiterGatewayFilterFactory in org.springframework.cloud.gateway.config.GatewayAutoConfiguration required a single bean, but 3 were found:- hostNameAddressKeyResolver: defined in file [C:\Users\admin\Desktop\springcloud_Hoxton\springcloud-apigateway-gateway9528\target\classes\com\wsh\springcloud\keyResolver\HostNameAddressKeyResolver.class]- uriKeyResolver: defined in file [C:\Users\admin\Desktop\springcloud_Hoxton\springcloud-apigateway-gateway9528\target\classes\com\wsh\springcloud\keyResolver\UriKeyResolver.class]- userKeyResolver: defined in file [C:\Users\admin\Desktop\springcloud_Hoxton\springcloud-apigateway-gateway9528\target\classes\com\wsh\springcloud\keyResolver\UserKeyResolver.class]Action:Consider marking one of the beans as @Primary, updating the consumer to accept multiple beans, or using @Qualifier to identify the bean that should be consumed
```
解决方法：标识其中一个KeyResolver为主要的限流Key，在类声明处加上： @Primary注解即可。


# 参考资料
Spring Cloud微服务架构进阶

Spring Cloud微服务：入门、实战与进阶

重新定义Spring Cloud实战

















