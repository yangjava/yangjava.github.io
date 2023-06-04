---
layout: post
categories: [SpringCloud]
description: none
keywords: SpringCloud
---
# Cloud注册中心Eureka
注册中心在微服务架构中是必不可少的一部分，主要用来实现服务治理功能，如何用Netflix提供的Eureka作为注册中心，来实现服务治理的功能。

## Eureka简介
Spring Cloud Eureka是Spring Cloud Netflix微服务套件的一部分，基于Netflix Eureka做了二次封装，主要负责实现微服务架构中的服务治理功能。Spring Cloud Eureka是一个基于REST的服务，并且提供了基于Java的客户端组件，能够非常方便地将服务注册到Spring Cloud Eureka中进行统一管理。

服务治理是微服务架构中必不可少的一部分。服务治理必须要有一个注册中心，除了用Eureka作为注册中心外，我们还可以使用Consul、Etcd、Zookeeper等来作为服务的注册中心。

当你需要调用某一个服务的时候，你会先去Eureka中去拉取服务列表，查看你调用的服务在不在其中，在的话就拿到服务地址、端口等信息，然后调用。注册中心带来的好处就是，不需要知道有多少提供方，你只需要关注注册中心即可。

### 为什么Eureka比Zookeeper更适合作为注册中心呢？
主要是因为Eureka是基于AP原则构建的，而ZooKeeper是基于CP原则构建的。在分布式系统领域有个著名的CAP定理，即C为数据一致性；A为服务可用性；P为服务对网络分区故障的容错性。这三个特性在任何分布式系统中都不能同时满足，最多同时满足两个。

Zookeeper有一个Leader，而且在这个Leader无法使用的时候通过Paxos（ZAB）算法选举出一个新的Leader。这个Leader的任务就是保证写数据的时候只向这个Leader写入，Leader会同步信息到其他节点。通过这个操作就可以保证数据的一致性。

总而言之，想要保证AP就要用Eureka，想要保证CP就要用Zookeeper。Dubbo中大部分都是基于Zookeeper作为注册中心的。Spring Cloud中当然首选Eureka。

## 使用Eureka编写注册中心服务
首先创建一个Maven项目，取名为eureka-server，在pom.xml中配置Eureka的依赖信息。
```xml
    <dependencies>
        <!-- eureka -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
    </dependencies>
```

创建一个启动类EurekaServerApplication
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}

```
启动类和Spring Boot几乎完全一样，只是多了一个@EnableEurekaServer注解，表示开启Eureka Server。

接下来在src/main/resources下面创建一个application.properties属性文件，增加下面的配置：
```properties
spring.application.name=eureka-server
server.port=8761
# 由于该应用为注册中心，所以设置为false，代表不向注册中心注册自己 
eureka.client.register-with-eureka=false
# 由于注册中心的职责就是维护服务实例，它并不需要去检索服务，所以也设置为 false 
eureka.client.fetch-registry=false
```
eureka.client.register-with-eureka一定要配置为false，不然启动时会把自己当作客户端向自己注册，会报错。

接下来直接运行EurekaServerApplication就可以启动我们的注册中心服务了。我们在application.properties配置的端口是8761，则可以直接通过http://localhost:8761/去浏览器中访问，然后便会看到Eureka提供的Web控制台。

## 编写服务提供者
注册中心已经创建并且启动好了，接下来我们实现将一个服务提供者eureka-client-user-service注册到Eureka中，并提供一个接口给其他服务调用。

首先还是创建一个Maven项目，然后在pom.xml中增加相关依赖
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- eureka -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```
如果没有web依赖，会报错
```
[extShutdownHook] com.netflix.discovery.DiscoveryClient : Completed shut down of DiscoveryClient
```

创建一个启动类App
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class EurekaUserApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaUserApplication.class, args);
    }
}
```
启动类的方法与之前没有多大区别，只是注解换成@EnableDiscoveryClient，表示当前服务是一个Eureka的客户端。

接下来在src/main/resources下面创建一个application.properties属性文件，增加下面的配置：
```properties
spring.application.name= eureka-client-user-service
server.port=8081
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
# 采用IP注册
eureka.instance.preferIpAddress=true
# 定义实例ID格式
eureka.instance.instance-id=${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```

eureka.client.serviceUrl.defaultZone的地址就是我们之前启动的Eureka服务的地址，在启动的时候需要将自身的信息注册到Eureka中去。

执行App启动服务，我们可以看到控制台中有输出注册信息的日志：
```
DiscoveryClient_EUREKA-CLIENT-USER-SERVICE/eureka-client-user-service:192.168.31.245:8081 - registration status: 204
```
我们可以进一步检查服务是否注册成功。回到之前打开的Eureka的Web控制台，刷新页面，就可以看到新注册的服务信息了。

## 编写提供接口
创建一个Controller，提供一个接口给其他服务查询
```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {
    @GetMapping("/user/hello")
    public String hello() {
        return "hello";
    }
}

```
重启服务，访问http://localhost:8081/user/hello，如果能看到我们返回的Hello字符串，就证明接口提供成功了。

## 编写服务消费者
创建服务消费者，消费我们刚刚编写的user/hello接口，同样需要先创建一个Maven项目eureka-client-role-service，然后添加依赖，依赖和服务提供者的一样。
```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- eureka -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
    </dependencies>
```

创建启动类App，启动代码与前面所讲也是一样的。唯一不同的就是application.properties文件中的配置信息：
```properties
spring.application.name= eureka-client-role-service
server.port=8082
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
# 采用IP注册
eureka.instance.preferIpAddress=true
# 定义实例ID格式
eureka.instance.instance-id=${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```

RestTemplate是Spring提供的用于访问Rest服务的客户端，RestTemplate提供了多种便捷访问远程Http服务的方法，能够大大提高客户端的编写效率。我们通过配置RestTemplate来调用接口
```
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfiguration {

    @Bean
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }
}

```
创建接口，在接口中调用user/hello接口
```
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import javax.annotation.Resource;

@RestController
public class RoleController {
    @Resource
    private RestTemplate restTemplate;

    @GetMapping("/role/callHello")
    public String callHello() {
        return restTemplate.getForObject( "http://localhost:8081/user/hello", String.class);
    }
}

```
执行App启动消费者服务，访问/role/callHello接口来看看有没有返回Hello字符串，如果返回了就证明调用成功。访问地址为http://localhost:8082/role/callHello。

## 通过Eureka来消费接口
既然用了注册中心，那么客户端调用的时候肯定是不需要关心有多少个服务提供接口，下面我们来改造之前的调用代码。

首先改造RestTemplate的配置，添加一个@LoadBalanced注解，这个注解会自动构造LoadBalancerClient接口的实现类并注册到Spring容器中。
```java
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfiguration {

    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }
}

```

接下来就是改造调用代码，我们不再直接写固定地址，而是写成服务的名称，这个名称就是我们注册到Eureka中的名称，是属性文件中的spring.application.name。
```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import javax.annotation.Resource;

@RestController
public class RoleController {
    @Resource
    private RestTemplate restTemplate;

    @GetMapping("/role/callHello")
    public String callHello() {
        return restTemplate.getForObject( "http://eureka-client-user-service/user/hello", String.class);
    }
}
```

## 开启Eureka认证
Eureka自带了一个Web的管理页面，方便我们查询注册到上面的实例信息，但是有一个问题：如果在实际使用中，注册中心地址有公网IP的话，必然能直接访问到，这样是不安全的。所以我们需要对Eureka进行改造，加上权限认证来保证安全性。

改造我们的eureka-server，通过集成Spring-Security来进行安全认证。在pom.xml中添加Spring-Security的依赖包
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

然后在application.properties中加上认证的配置信息：
```properties
# 用户名
spring.security.user.name=yangjingjing
# 密码
spring.security.user.password=123456
```

增加Security配置类：
```
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //关闭csrf
        http.csrf().disable();
        // 支持httpBasic
        http.authorizeRequests()
                .anyRequest()
                .authenticated()
                .and()
                .httpBasic();
    }
}
```
重新启动注册中心，访问http://localhost:8761/，此时浏览器会提示你输入用户名和密码，输入正确后才能继续访问Eureka提供的管理页面。

在Eureka开启认证后，客户端注册的配置也要加上认证的用户名和密码信息：
```properties
eureka.client.serviceUrl.defaultZone= http://yangjingjing:123456@localhost:8761/eureka/
```

## Eureka高可用搭建
前面我们搭建的注册中心只适合本地开发使用，在生产环境中必须搭建一个集群来保证高可用。Eureka的集群搭建方法很简单：每一台Eureka只需要在配置中指定另外多个Eureka的地址就可以实现一个集群的搭建了。

下面我们以2个节点为例来说明搭建方式。假设我们有master和slaveone两台机器，需要做的就是：
- 将master注册到slaveone上面。
- 将slaveone注册到master上面。

如果是3台机器，以此类推： 
- 将master注册到slaveone和slavetwo上面。
- 将slaveone注册到master和slavetwo上面。
- 将slavetwo注册到master和slaveone上面。

### 搭建步骤
创建一个新的项目eureka-server-cluster，配置跟eureka-server一样。

首先，我们需要增加2个属性文件，在不同的环境下启动不同的实例。增加application-master.properties：
```properties
server.port=8761
# 指向你的从节点的Eureka
eureka.client.serviceUrl.defaultZone= http://用户名:密码@localhost:8762/eureka/
```

增加 application-slaveone.properties：
```properties
server.port=8762
# 指向你的主节点的Eureka
eureka.client.serviceUrl.defaultZone= http://用户名:密码 @localhost:8761/eureka/
```

在application.properties中添加下面的内容：
```properties
spring.application.name=eureka-server-cluster
# 由于该应用为注册中心，所以设置为false，代表不向注册中心注册自己 
eureka.client.register-with-eureka=false
# 由于注册中心的职责就是维护服务实例，并不需要检索服务，所以也设置为 false 
eureka.client.fetch-registry=false

spring.security.user.name=yangjingjing
spring.security.user.password=123456

# 指定不同的环境
spring.profiles.active=master
```

在A机器上默认用master启动，然后在B机器上加上--spring.profiles.active=slaveone启动即可。

这样就将master注册到了slaveone中，将slaveone注册到了master中，无论谁出现问题，应用都能继续使用存活的注册中心。

之前在客户端中我们通过配置eureka.client.serviceUrl.defaultZone来指定对应的注册中心，当我们的注册中心有多个节点后，就需要修改eureka.client.serviceUrl.defaultZone的配置为多个节点的地址，多个地址用英文逗号隔开即可：
```properties
eureka.client.serviceUrl.defaultZone=http://yangjingjing:123456@localhost:8761/eureka/,http://yangjingjing:123456@localhost:8762/eureka/
```

## 关闭自我保护
保护模式主要在一组客户端和Eureka Server之间存在网络分区场景时使用。一旦进入保护模式，Eureka Server将会尝试保护其服务的注册表中的信息，不再删除服务注册表中的数据。当网络故障恢复后，该Eureka Server节点会自动退出保护模式。

可以通过下面的配置将自我保护模式关闭，这个配置是在eureka-server中：
```
eureka.server.enableSelfPreservation=false
```

## 自定义Eureka的InstanceID
客户端在注册时，服务的Instance ID的默认值的格式如下：
```
${spring.cloud.client.hostname}:${spring.application.name}:${spring.application.instance_id:${server.port}}
```
翻译过来就是“主机名：服务名称：服务端口”。当我们在Eureka的Web控制台查看服务注册信息的时候，就是这样的一个格式：user-PC：eureka-client-user-service：8081。

很多时候我们想把IP显示在上述格式中，此时，只要把主机名替换成IP就可以了，或者调整顺序也可以。可以改成下面的样子，用“服务名称：服务所在IP：服务端口”的格式来定义：
```properties
eureka.instance.instance-id=${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```
定义之后我们看到的就是eureka-client-user-service：192.168.31.245：8081，一看就知道是哪个服务，在哪台机器上，端口是多少。

所以还需要加一个配置才能让跳转的链接变成我们想要的样子，使用IP进行注册
```properties
eureka.instance.preferIpAddress=true
```

## 自定义实例跳转链接
们通过配置实现了用IP进行注册，当点击Instance ID进行跳转的时候，就可以用IP跳转了，跳转的地址默认是IP+Port/info。我们可以自定义这个跳转的地址：
```properties
eureka.instance.status-page-url=http://xxx.com
```

## 快速移除已经失效的服务信息
在实际开发过程中，我们可能会不停地重启服务，由于Eureka有自己的保护机制，故节点下线后，服务信息还会一直存在于Eureka中。我们可以通过增加一些配置让移除的速度更快一点，当然只在开发环境下使用，生产环境下不推荐使用。

首先在我们的eureka-server中增加两个配置，分别是关闭自我保护和清理间隔：
```properties
eureka.server.enable-self-preservation=false
# 默认 60000 毫秒
eureka.server.eviction-interval-timer-in-ms=5000
```
然后在具体的客户端服务中配置下面的内容：
```properties
eureka.client.healthcheck.enabled=true
# 默认 30 秒
eureka.instance.lease-renewal-interval-in-seconds=5
# 默认 90 秒
eureka.instance.lease-expiration-duration-in-seconds=5
```
eureka.client.healthcheck.enabled用于开启健康检查，需要在pom.xml中引入actuator的依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
其中：
- eureka.instance.lease-renewal-interval-in-seconds表示Eureka Client发送心跳给server端的频率。
- eureka.instance.lease-expiration-duration-in-seconds表示Eureka Server至上一次收到client的心跳之后，等待下一次心跳的超时时间，在这个时间内若没收到下一次心跳，则移除该Instance。
- 更多的Instance配置信息可参考源码中的配置类：org.springframework.cloud.netflix.eureka.EurekaInstanceConfigBean。
- 更多的Server配置信息可参考源码中的配置类：org.springframework.cloud.netflix.eureka.server.EurekaServerConfigBean。

## Eureka REST API
Eureka作为注册中心，其本质是存储了每个客户端的注册信息，Ribbon在转发的时候会获取注册中心的服务列表，然后根据对应的路由规则来选择一个服务给Feign来进行调用。如果我们不是Spring Cloud技术选型，也想用Eureka，可以吗？完全可以。

如果不是Spring Cloud技术栈，笔者推荐用Zookeeper，这样会方便些，当然用Eureka也是可以的，这样的话就会涉及如何注册信息、如何获取注册信息等操作。其实Eureka也考虑到了这点，提供了很多REST接口来给我们调用。

具体的接口信息请查看官方文档：https://github.com/Netflix/eureka/wiki/Eureka-REST-operations 。

## 元数据使用
Eureka的元数据有两种类型，分别是框架定好了的标准元数据和用户自定义元数据。标准元数据指的是主机名、IP地址、端口号、状态页和健康检查等信息，这些信息都会被发布在服务注册表中，用于服务之间的调用。自定义元数据可以使用eureka.instance.metadataMap进行配置。

自定义元数据说得通俗点就是自定义配置，我们可以为每个Eureka Client定义一些属于自己的配置，这个配置不会影响Eureka的功能。自定义元数据可以用来做一些扩展信息，比如灰度发布之类的功能，可以用元数据来存储灰度发布的状态数据，Ribbon转发的时候就可以根据服务的元数据来做一些处理。当不需要灰度发布的时候可以调用Eureka提供的REST API将元数据清除掉。

下面我们来自定义一个简单的元数据，在属性文件中配置如下：
```properties
eureka.instance.metadataMap.yangjingjing=yangjingjing
```
上述代码定义了一个key为yangjingjing的配置，value是yangjingjing。重启服务，然后通过Eureka提供的REST API来查看刚刚配置的元数据是否已经存在于Eureka中。

获取某个服务的注册信息，可以直接GET请求：http://localhost:8761/eureka/apps/eureka-client-user-service 。其中，eureka-client-user-service是应用名称，也就是spring.application.name。

## EurekaClient使用
当我们的项目中集成了Eureka之后，可以通过EurekaClient来获取一些我们想要的数据，比如上节讲的元数据。我们就可以直接通过EurekaClient来获取，不用再去调用Eureka提供的REST API。
```
@Resource
private EurekaClient eurekaClient;
@GetMapping("/role/infos")
public Object serviceUrl() {
    return eurekaClient.getInstancesByVipAddress( "eureka-client-user-service", false);
}
```
通过EurekaClient获取的数据跟我们自己去掉API获取的数据是一样的，从使用角度来说前者比较方便。

除了使用EurekaClient，还可以使用DiscoveryClient，这个不是Feign自带的，是Spring Cloud重新封装的，类的路径为org.springframework.cloud.client.discovery.DiscoveryClient。
```
@Resource
private DiscoveryClient discoveryClient;
@GetMapping("/article/infos")
public Object serviceUrl() {
    return discoveryClient.getInstances("eureka-client-user-service");
}
```

## 健康检查
默认情况下，Eureka客户端是使用心跳和服务端通信来判断客户端是否存活，在某些场景下，比如MongoDB出现了异常，但你的应用进程还是存在的，这就意味着应用可以继续通过心跳上报，保持应用自己的信息在Eureka中不被剔除掉。

Spring Boot Actuator提供了/actuator/health端点，该端点可展示应用程序的健康信息，当MongoDB异常时，/actuator/health端点的状态会变成DOWN，由于应用本身确实处于存活状态，但是MongoDB的异常会影响某些功能，当请求到达应用之后会发生操作失败的情况。

在这种情况下，我们希望可以将健康信息传递给Eureka服务端。这样Eureka中就能及时将应用的实例信息下线，隔离正常请求，防止出错。通过配置如下内容开启健康检查：
```properties
eureka.client.healthcheck.enabled=true
```
我们可以通过扩展健康检查的端点来模拟异常情况，定义一个扩展端点，将状态设置为DOWN
```
@Component
public class CustomHealthIndicator extends AbstractHealthIndicator {

    @Override
    protected void doHealthCheck(Builder builder) throws Exception {
        builder.down().withDetail("status", false);
    }
}
```
扩展好后我们访问/actuator/health可以看到当前的状态是DOWN

## 服务上下线监控
在某些特定的需求下，我们需要对服务的上下线进行监控，上线或下线都进行邮件通知，Eureka中提供了事件监听的方式来扩展。

目前支持的事件如下：
- EurekaInstanceCanceledEvent服务下线事件。
- EurekaInstanceRegisteredEvent服务注册事件。
- EurekaInstanceRenewedEvent服务续约事件。
- EurekaRegistryAvailableEvent Eureka注册中心启动事件。
- EurekaServerStartedEvent Eureka Server启动事件。

基于Eureka提供的事件机制，可以监控服务的上下线过程，在过程发生中可以发送邮件来进行通知。
```java
@Component
public class EurekaStateChangeListener {

    @EventListener
    public void listen(EurekaInstanceCanceledEvent event) {
        System.err.println(event.getServerId() + "\t" +
            event.getAppName() + " 服务下线 ");
    }

    @EventListener
    public void listen(EurekaInstanceRegisteredEvent event) {
        InstanceInfo instanceInfo = event.getInstanceInfo(); 
        System.err.println(instanceInfo.getAppName() + " 进行注册 ");
    }
    @EventListener
    public void listen(EurekaInstanceRenewedEvent event) {
        System.err.println(event.getServerId() + "\t" +
                event.getAppName() + " 服务进行续约 ");
    }

    @EventListener
    public void listen(EurekaRegistryAvailableEvent event) {
        System.err.println(" 注册中心 启动 ");
    }
    @EventListener
    public void listen(EurekaServerStartedEvent event) {
        System.err.println("Eureka Server 启 动 ");
    }

}
```
在Eureka集群环境下，每个节点都会触发事件，这个时候需要控制下发送通知的行为，不控制的话每个节点都会发送通知。








