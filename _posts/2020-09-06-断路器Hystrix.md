---
layout: post
categories: [Hystrix]
description: none
keywords: Hystrix
---
# 断路器Hystrix
Hystrix是Netflix的一个开源项目，它能够在依赖服务失效的情况下，通过隔离系统依赖服务的方式，防止服务级联失败；同时Hystrix提供失败回滚机制，使系统能够更快地从异常中恢复。

## 基础应用
spring-cloud-netflix-hystrix对Hystrix进行封装和适配，使Hystrix能够更好地运行于Spring Cloud环境中，为微服务间的调用提供强有力的容错机制。

Hystrix具有如下的功能：
- 在通过第三方客户端访问（通常是通过网络）依赖服务出现高延迟或者失败时，为系统提供保护和控制。
- 在复杂的分布式系统中防止级联失败（服务雪崩效应）。
- 快速失败（Fail fast）同时能快速恢复。
- 提供失败回滚（Fallback）和优雅的服务降级机制。
- 提供近实时的监控、报警和运维控制手段。

## RestTemplate与Hystrix
可以搭建包含Hystrix依赖的SpringBoot项目。首先添加eureka-client和hystrix的相关依赖，如下所示：
```xml
<dependency> <!--eureka-client相关依赖-->
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency> <!--hystrix相关依赖-->
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```
在application.yml中为hystrix-service服务配置注册中心地址，代码如下所示：

```yaml
# application.yml
eureka:
    instance:
        instance-id: ${spring.application.name}:${vcap.application.instance_id:$ {spring.application.instance_id:${random.value}}}
    client:
        service-url:
            default-zone: http://localhost:8761/eureka/
spring:
    application:
        name: hystrix-service
server:
    port: 8876
```
通过@EnableCircuitBreaker注解开启Hystrix，同时注入一个可以进行负载均衡的RestTemplate，代码如下所示：
```java
@SpringBootApplication
@EnableCircuitBreaker // 开启Hystrix
public class Chapter6HystrixApplication {
    public static void main(String[] args) {
        SpringApplication.run(Chapter6HystrixApplication.class, args);
    }
    // 注入可以进行负载均衡的RestTemplate
    @Bean
    @LoadBalanced
    RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
```

编写相关的服务，代码如下所示：
```java
@Service
public class InstanceService {
    private static String DEFAULT_SERVICE_ID = "application";
    private static String DEFAULT_HOST = "localhost";
    private static int DEFAULT_PORT = 8080;
    private static Logger logger = LoggerFactory.getLogger(InstanceService.class);
    @Autowired
    RestTemplate restTemplate;
    @HystrixCommand(fallbackMethod = "instanceInfoGetFail")
    public Instance getInstanceByServiceIdWithRestTemplate(String serviceId){

        Instance instance = restTemplate.getForEntity("http://FEIGN-SERVICE/feign-service/instance/{serviceId}", Instance.class, serviceId).getBody();
        return instance;
    }
    private Instance instanceInfoGetFail(String serviceId){
        logger.info("Can not get Instance by serviceId {}", serviceId);
        return new Instance("error", "error", 0);
    }
}
```
通过@HystrixCommand注解为getInstanceByServiceIdWithRestTemplate方法指定回滚的方法instanceInfoGetFail，该方法返回了全是error的实体类信息。在getInstanceByServiceIdWithRestTemplate方法中通过restTemplate调用feign-service服务的相关的接口，期望返回结果，通过@HystrixCommand注解将该方法纳入到Hystrix的监控中。

编写相关的控制器类，调用getInstanceByServiceIdWithRestTemplate方法，获得结果并返回。代码如下所示：
```java
@RestController
@RequestMapping("/instance")
public class InstanceController {
    private static final Logger logger = LoggerFactory.getLogger(InstanceController.class);
    @Autowired
    InstanceService instanceService;
    @RequestMapping(value = "rest-template/{serviceId}", method = RequestMethod.GET)
    public Instance getInstanceByServiceIdWithRestTemplate(@PathVariable("serviceId") String serviceId){
        logger.info("Get Instance by serviceId {}", serviceId);
        return instanceService.getInstanceByServiceIdWithRestTemplate(serviceId);
    }
}
```
依次启动eureka-server、feign-service（feign-service为第5章的feign-service项目）以及本服务。访问http://localhost:8876/instance/rest-template/my-application接口。结果如下所示：
```java
{"serviceId":"my-application","host":"localhost","port":8080}
```
访问成功执行，返回预期结果。关闭feign-service，再次访问http://localhost:/8876/instance/rest-template/my-application接口。结果如下所示：
```java
{"serviceId":"error","host":"error","port":0}
```
这说明在feign-service服务不可用时，系统执行失败回滚方法，返回“error”结果。






