---
layout: post
categories: [OpenFeign]
description: none
keywords: OpenFeign
---
# OpenFeign
OpenFeign是一个声明式RESTful网络请求客户端，使得编写Web服务客户端更加方便和快捷。只需要使用OpenFeign提供的注解修饰定义网络请求的接口类，就可以使用该接口的实例发送RESTful风格的网络请求。OpenFeign还可以集成Ribbon和Hytrix来提供负载均衡和网络断路器的功能。

## OpenFeign简介
OpenFeign是一个声明式RESTful网络请求客户端。OpenFeign会根据带有注解的函数信息构建出网络请求的模板，在发送网络请求之前，OpenFeign会将函数的参数值设置到这些请求模板中。虽然OpenFeign只能支持基于文本的网络请求，但是它可以极大简化网络请求的实现，方便编程人员快速构建自己的网络请求应用。

使用OpenFeign的Spring应用架构一般分为三个部分，分别为服务注册中心、服务提供者和服务消费者。服务提供者向服务注册中心注册自己，然后服务消费者通过OpenFeign发送请求时，OpenFeign会向服务注册中心获取关于服务提供者的信息，然后再向服务提供者发送网络请求。

## 服务注册中心
OpenFeign可以配合Eureka等服务注册中心同时使用。Eureka作为服务注册中心，为OpenFeign提供服务端信息的获取，比如说服务的IP地址和端口。

## 服务提供者
Spring Cloud OpenFeign是声明式RESTful网络请求客户端，所以对服务提供者的实现方式没有任何影响。也就是说，服务提供者只需要提供对外的网络请求接口就可，至于其具体实现既可以使用Spring MVC，也可以使用Jersey。只需要确保该服务提供者被注册到服务注册中心上即可。
```java
@RestController
@RequestMapping("/feign-service")
public class FeignServiceController {
    @RequestMapping(value = "/instance/{serviceId}", method = RequestMethod.GET)
    public Instance getInstanceByServiceId(@PathVariable("serviceId") String serviceId){
        return new Instance(serviceId);
    }
}
```
如上述代码所示，通过@RestController和@RequestMapping声明了获取Instance资源的网络接口。除了实现网络API接口之外，还需要将该服务注册到Eureka Server上。需要在application.yml文件中设置服务注册中心的相关信息和该应用的名称，相关配置如下所示：
```yaml
eureka:
    instance:
        instance-id: server:1
    client:
        service-url:
            default-zone: http://localhost:8761/eureka/
spring:
    application:
        name: feign-service
server:
    port: 9000
```

## 服务消费者
OpenFeign是声明式RESTful客户端，所以构建OpenFeign项目的关键在于构建服务消费者。通过下面的方法可以创建一个Spring Cloud OpenFeign的服务消费者项目。

首先需要在pom文件中添加Eureka和OpenFeign相关的依赖。然后在工程的入口类上添加@EnableFeignClients注解开启Spring Cloud OpenFeign的自动化配置功能，代码如下所示：
```java
@SpringBootApplication
@EnableFeignClients
public class FeignClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ChapterFeignClientApplication.class, args);
    }
}
```
@EnableFeignClients就像是一个开关，只有使用了该注解，OpenFeign相关的组件和配置机制才会生效。@EnableFeignClients还可以对OpenFeign相关组件进行自定义配置，它的方法和原理会在本章的源码分析章节再做具体的讲解。

接下来需要定义一个FeignServiceClient接口，通过@FeignClient注解来指定调用的远程服务名称。这一类被@FeignClient注解修饰的接口类一般被称为FeignClient。在FeignClient接口类中，可以使用@RequestMapping定义网络请求相关的方法，如下所示：
```java
@FeignClient("feign-service")
@RequestMapping("/feign-service")
public interface FeignServiceClient {
    @RequestMapping(value = "/instance/{serviceId}", method = RequestMethod.GET)
    public Instance getInstanceByServiceId(@PathVariable("serviceId") String serviceId);
}
```
如上面代码片段所显示的，如果你调用FeignServiceClient对象的getInstanceByServiceId方法，那么OpenFeign就会向feign-service服务的/feign-service/instance/{serviceId}接口发送网络请求。

最后，在服务消费端项目的application.yml文件中配置Eureka服务注册中心的相关属性，具体配置如下所示：
```yaml
eureka:
    instance:
        instance-id: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}
    client:
        service-url:
            default-zone: http://localhost:8761/eureka/
spring:
    application:
        name: feign-client
server:
    port: 8770
```

## Decoder与Encoder的定制化
Encoder用于将Object对象转化为HTTP的请求Body，而Decoder用于将网络响应转化为对应的Object对象。对于二者，OpenFeign都提供了默认的实现，但是使用者可以根据自己的业务来选择其他的编解码方式。只需要在自定义配置类中给出Decoder和Encoder的自定义Bean实例，那么OpenFeign就可以根据配置，自动使用我们提供的自定义实例进行编解码操作。如下代码所示，CustomFeignConfig配置类将ResponseEntityDecoder和SpringEncoder配置为Feign的Decoder与Encoder实例。
```java
public class CustomFeignConfig {
    @Bean
    public Decoder feignDecoder() {
        HttpMessageConverter jacksonConverter = new MappingJackson2HttpMessageConverter (customObjectMapper());
        ObjectFactory<HttpMessageConverters> objectFactory = () -> new HttpMessage Converters(jacksonConverter);
        return new ResponseEntityDecoder(new SpringDecoder(objectFactory));
    }
    @Bean
    public Encoder feignEncoder(){
        HttpMessageConverter jacksonConverter = new MappingJackson2HttpMessageConverter (customObjectMapper());
        ObjectFactory<HttpMessageConverters> objectFactory = () -> new HttpMessage Converters(jacksonConverter);
        return new SpringEncoder(objectFactory);
    }
    public ObjectMapper customObjectMapper(){
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.configure(DeserializationFeature.ACCEPT_EMPTY_STRING_AS_NULL_OBJECT, true);
        return objectMapper;
    }
}
```
MappingJackson2HttpMessageConverter是转换JSON的底层转换器

## 请求/响应压缩
可以通过下面的属性配置来让OpenFeign在发送请求时进行GZIP压缩：
```java
feign.compression.request.enabled=true
feign.compression.response.enabled=true
```
OpenFeign的压缩配置属性和一般的Web Server配置类似。这些属性允许选择性地压缩某种类型的请求并设置最小的请求阈值，配置如下所示：

你也可以使用FeignContentGzipEncodingInterceptor来实现请求的压缩，需要在自定义配置文件中初始化该类型的实例，供OpenFeign使用，具体实现如下所示：
```java
feign.compression.request.enabled=true
feign.compression.request.mime-types=text/xml,application/xml,application/json
feign.compression.request.min-request-size=2048
```







