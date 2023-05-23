---
layout: post
categories: [SpringCloud,Eureka]
description: none
keywords: Eureka
---
# Eureka注册中心
注册中心在微服务架构中是必不可少的一部分，主要用来实现服务治理功能，我们将学习如何用Netflix提供的Eureka作为注册中心，来实现服务治理的功能。

## Eureka简介
Netflix Eureka是由Netflix开源的一款基于REST的服务发现组件，包括Eureka Server及Eureka Client。从2012年9月在GitHub上发布1.1.2版本以来，至今已经发布了231次，最新版本为2018年8月份发布的1.9.4版本。期间有进行2.x版本的开发，不过由于各种原因内部已经冻结开发，目前还是以1.x版本为主。Spring Cloud Netflix Eureka是Pivotal公司为了将Netflix Eureka整合于Spring Cloud生态系统提供的版本。

Eureka是Netflix公司开源的一款服务发现组件，该组件提供的服务发现可以为负载均衡、failover等提供支持。Eureka包括Eureka Server及Eureka Client。Eureka Server提供REST服务，而Eureka Client则是使用Java编写的客户端，用于简化与Eureka Server的交互。

Eureka这个词来源于古希腊语，意为“我找到了！我发现了！”。据传，阿基米德在洗澡时发现浮力原理，高兴得来不及穿上衣服，跑到街上大喊：“Eureka！”。

在Netflix中，Eureka是一个RESTful风格的服务注册与发现的基础服务组件。Eureka由两部分组成，一个是Eureka Server，提供服务注册和发现功能，即我们上面所说的服务器端；另一个是Eureka Client，它简化了客户端与服务端之间的交互。Eureka Client会定时将自己的信息注册到Eureka Server中，并从Server中发现其他服务。Eureka Client中内置一个负载均衡器，用来进行基本的负载均衡。

## Spring Cloud Eureka入门案例
基于Spring Cloud的Finchley.RELEASE版本，spring-cloud-netflix版本基于2.0.0.RELEASE，Eureka版本基于v1.9.2。
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 创建Eureka Server工程
配置Eureka Server工程的pom.xml文件，只需要添加spring-cloud-starter-netflix-eureka-server即可

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```
对于Eureka的启动主类，在启动类中添加注解@EnableEurekaServer，作为程序的入口
```java
@SpringBootApplication
//会为项目自动配置必须的配置类，标识该服务为注册中心
@EnableEurekaServer
public class Ch21EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(Ch21EurekaServerApplication.class, args);
    }
}
```

Eureka Server需要的配置文件
```yaml
server:
  port: 8761
eureka:
  instance:
    hostname: standalone
    instance-id: ${spring.application.name}:${vcap.application.instance_id:$ {spring.application.instance_id:${random.value}}}
  client:
    register-with-eureka: false  # 表明该服务不会向Eureka Server注册自己的信息
    fetch-registry: false # 表明该服务不会向Eureka Server获取注册信息
    service-url:  # Eureka Server注册中心的地址，用于client与server进行交流
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
spring:
  application:
    name: eureka-service
```
InstanceId是Eureka服务的唯一标记，主要用于区分同一服务集群的不同实例。一般来讲，一个Eureka服务实例默认注册的InstanceId是它的主机名（即一个主机只有一个服务）。但是这样会引发一个问题，一台主机不能启动多个属于同一服务的服务实例。
为了解决这种情况，spring-cloud-netflix-eureka提供了一个合理的实现，如上面代码中的InstanceId设置样式。通过设置random.value可以使得每一个服务实例的InstanceId独一无二，从而可以唯一标记它自身。

Eureka Server既可以独立部署，也可以集群部署。在集群部署的情况下，Eureka Server间会进行注册表信息同步的操作，这时被同步注册表信息的Eureka Server将会被其他同步注册表信息的Eureka Server称为peer。

请注意，上述配置中的service-url指向的注册中心为实例本身。通常来讲，一个Eureka Server也是一个Eureka Client，它会尝试注册自己，所以需要至少一个注册中心的URL来定位对等点peer。如果不提供这样一个注册端点，注册中心也能工作，但是会在日志中打印无法向peer注册自己的信息。在独立（Standalone）Eureka Server的模式下，Eureka Server一般会关闭作为客户端注册自己的行为。

Eureka Server与Eureka Client之间的联系主要通过心跳的方式实现。心跳（Heartbeat）即Eureka Client定时向Eureka Server汇报本服务实例当前的状态，维护本服务实例在注册表中租约的有效性。

Eureka Server需要随时维持最新的服务实例信息，所以在注册表中的每个服务实例都需要定期发送心跳到Server中以使自己的注册保持最新的状态（数据一般直接保存在内存中）。为了避免Eureka Client在每次服务间调用都向注册中心请求依赖服务实例的信息，Eureka Client将定时从Eureka Server中拉取注册表中的信息，并将这些信息缓存到本地，用于服务发现。

启动Eureka Server后，应用会有一个主页面用来展示当前注册表中的服务实例信息并同时暴露一些基于HTTP协议的端点在/eureka路径下，这些端点将由Eureka Client用于注册自身、获取注册表信息以及发送心跳等。

### 创建Eureka Client组件工程
配置Eureka Client工程的pom.xml文件，只需要引入spring-cloud-starter-netflix-eureka-client即可

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```
添加Eureka Client的启动主类
```java
@SpringBootApplication
@EnableDiscoveryClient
public class Ch21EurekaClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(Ch21EurekaClientApplication.class, args);
    }
}
```
在Spring Cloud的Finchley版本中，只要引入spring-cloud-starter-netflix-eureka-client的依赖，应用就会自动注册到Eureka Server，但是需要在配置文件中添加Eureka Server的地址。在application.yml添加以下配置：
Eureka Client的配置文件
```yaml
eureka:
  instance:
    hostname: client
    instance-id: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}

  client:
    service-url: # Eureka Server注册中心的地址，用于Client与Server进行交流
      defaultZone: http://localhost:8761/eureka/

server:
  port: 8760
spring:
  application:
    name: eureka-client-service
```

### 效果展示
- Eureka Server主页
访问Eureka Server的主页`http://localhost:8761`。

从图中可以看到以下信息：
  - 展示当前注册到Eureka Server上的服务实例信息。
  - 展示Eureka Server运行环境的通用信息。
  - 展示Eureka Server实例的信息。

Eureka中标准元数据有主机名、IP地址、端口号、状态页url和健康检查url等，这些元数据都会保存在Eureka Server的注册表中，Eureka Client根据服务名读取这些元数据，来发现和调用其他服务实例。

- 服务间调用
RestTemplate将根据服务名eureka-client-service通过预先从eureka-service缓存到本地的注册表中获取到eureka-client-service服务的具体地址，从而发起服务间调用。

## 服务发现原理
SpringCloud的Finchley版本采用的Eureka版本为1.9，架构图主要基于Eureka v1进行介绍。

### Region与Availability Zone
Eureka最初设计的目的是AWS（亚马逊网络服务系统）中用于部署分布式系统，所以首先对AWS上的区域（Regin）和可用区（Availability Zone）进行简单的介绍。
- 区域：AWS根据地理位置把某个地区的基础设施服务集合称为一个区域，区域之间相对独立。在架构图上，us-east-1c、us-east-1d、us-east-1e表示AWS中的三个设施服务区域，这些区域中分别部署了一个Eureka集群。
- 可用区：AWS的每个区域都是由多个可用区组成的，而一个可用区一般都是由多个数据中心（简单理解成一个原子服务设施）组成的。可用区与可用区之间是相互独立的，有独立的网络和供电等，保证了应用程序的高可用性。一个可用区中可能部署了多个Eureka，一个区域中有多个可用区，这些Eureka共同组成了一个Eureka集群。

### 组件与行为
下面介绍组件与行为：
- Application Service：是一个Eureka Client，扮演服务提供者的角色，提供业务服务，向Eureka Server注册和更新自己的信息，同时能从Eureka Server注册表中获取到其他服务的信息。
- Eureka Server：扮演服务注册中心的角色，提供服务注册和发现的功能。每个Eureka Cient向Eureka Server注册自己的信息，也可以通过Eureka Server获取到其他服务的信息达到发现和调用其他服务的目的。
- Application Client：是一个Eureka Client，扮演了服务消费者的角色，通过Eureka Server获取注册到其上其他服务的信息，从而根据信息找到所需的服务发起远程调用。
- Replicate：Eureka Server之间注册表信息的同步复制，使Eureka Server集群中不同注册表中服务实例信息保持一致。
- Make Remote Call：服务之间的远程调用。
- Register：注册服务实例，Client端向Server端注册自身的元数据以供服务发现。
- Renew：续约，通过发送心跳到Server以维持和更新注册表中服务实例元数据的有效性。当在一定时长内，Server没有收到Client的心跳信息，将默认服务下线，会把服务实例的信息从注册表中删除。
- Cancel：服务下线，Client在关闭时主动向Server注销服务实例元数据，这时Client的服务实例数据将从Server的注册表中删除。
- Get Registry：获取注册表，Client向Server请求注册表信息，用于服务发现，从而发起服务间远程调用。

## Eureka Instance和Client的元数据
在EurekaInstanceConfigBean中，相当大一部分内容是关于Eureka Client服务实例的信息，这部分信息称为元数据，它是用来描述自身服务实例的相关信息。Eureka中的标准元数据有主机名、IP地址、端口号、状态页url和健康检查url等用于服务注册与发现的重要信息。开发者可以自定义元数据，这部分额外的数据可以通过键值对（key-value）的形式放在eureka.instance.metadataMap，如下所示：
```yaml
eureka:
    instance:
        metadataMap:
                metadata-map:
                    mymetaData: mydata
```
这里定义了一个键为mymetaData，值为mydata的自定义元数据，metadata-map在EurekaInstanceConfigBean会被配置为以下的属性：
```java
private Map<String, String> metadataMap = new HashMap<>();
```
这些自定义的元数据可以按照自身业务需要或者根据其他的特殊需要进行定制。

## 状态页和健康检查页端口设置
Eureka服务实例状态页和健康检查页的默认url是/actuator/info和/actuator/health，通常是使用spring-boot-actuator中相关的端点提供实现。一般情况下这些端点的配置都不需要修改，但是当spring-boot没有使用默认的应用上下文路径（context path）或者主分发器路径（Dispatch path）时，将会影响Eureka Server无法通过/actuator/health对Eureka Client进行健康检查，以及无法通过/actuator/info访问Eureka Client的信息接口。例如设置成应用上下文路径为：
```yaml
server:
    servlet:
        context-path: /path
```
或者设置主分发器路径为：
```yaml
server:
    servlet:
        path: /path
```
为此需要对这些端点的URL进行更改，如下所示：
```yaml
server:
servlet:
    path: /path

eureka:
    instance:
        statusPageUrlPath: ${server.servlet.path}/actuator/info
        healthCheckUrlPath: ${server.servlet.path}/actuator/health
```
同样可以通过绝对路径的方式进行更改。

## 区域与可用区
在基础应用中，我们介绍了AWS的区域以及可用区的概念。

一般来说一个Eureka Client只属于一个区域，一个区域下有多个可用区，每个可用区下可能有多个Eureka Server（一个Eureka Server可以属于多个可用区）。
```yaml
eureka:
    client:
        region: us-east-1
        availability-zones:
            us-east-1: us-east-zone-1, us-east-zone-2
            us-west-2: us-west-zone-1, us-west-zone-2
        service-url:
             us-east-zone-1: http://xxx1,http://xxx2
             us-east-zone-2: http://xxx1,http://xxx2
             us-west-zone-1: http://xxx1,http://xxx2
             us-west-zone-2: http://xxx1,http://xxx2
```
以上是一份比较完整的关于区域与可用区的配置。获取serverUrls的过程是层层递进的，从区域到可用区，优先添加服务实例所处的可用区的serverUrls，为了保证容错性，又把本区域中的其他的可用区也添加到了serverUrls。通过这种多层次的设计，提供Eureka在区域内的容错性，保证了网络分区容忍性（Partition tolerance）。

当然也可以直接告诉Eureka Client服务实例所处的可用区，并希望使用同一个可用区的Eureka Server进行注册（在Eureka Client无法与Eureke Server进行通信时，它将轮询向配置中其他的Eureka Server注册直到成功为止），可以添加如下配置：
```yaml
eureka:
    instance:
        metadataMap:
            zone: us-east-zone-2
    client:
        prefer-same-zone-eureka: true
```
这样配置的话，使用的Eureka Server的url将会是us-east-zone-2：http://xxx1，http://xxx2。

## 高可用性服务注册中心
Eureka Server可以变得更有弹性和具备高可用性，通过部署多个注册中心实例，并让它们之间互相注册。在Standalone模式中，只能依赖Server和Client之间的缓存，并需要弹性的机制保证Server实例一直存活，单例的注册中心崩溃了，Client之间就很难互相发现和调用。

在配置文件中添加如下配置：
```yaml
---
spring:
    profiles: peer1
    application:
        name: eureka-server-peer
server:
    port: 8762
eureka:
    instance:
        hostname: peer1
        instance-id: ${spring.application.name}:${vcap.application.instance_id:$ {spring.application.instance_id:${random.value}}}
    client:
        service-url:
            defaultZone: http://localhost:8763/eureka/
---
spring:
    profiles: peer2
    application:
        name: eureka-server-peer
server:
    port: 8763
eureka:
    instance:
        hostname: peer2
        instance-id: ${spring.application.name}:${vcap.application.instance_id:$ {spring.application.instance_id:${random.value}}}
    client:
        service-url:
            defaultZone: http://localhost:8762/eureka/
---
spring:
    profiles:
        active: peer1
```
可以通过设置不同的spring.profiles.active启动不同配置的Eureka Server，上述配置声明了两个Eureka Server的配置，它们之间是相互注册的。可以添加多个peer，只要这些Eureka Server中存在一个连通点，那么这些注册中心的数据就能够进行同步，这就通过服务器的冗余增加了高可用性，即使其中一台Eureka Server宕机了，也不会导致系统崩溃。

同时对Eureka Client添加多个注册节点，使得其能够尝试向其他的注册中心发起请求（当注册节点前面的Eureka Server无法通信时），如下所示：
```yaml
eureka:
    client:
        service-url:
            defaultZone: http://localhost:8761/eureka/, http://localhost:8762/eureka/
```