---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot应用监控源码
在 Springboot 中， 端点可以通过 JMX 方式使用, 也可以使用 Http 方式使用，本文着重介绍与 Http 方式的使用和实现。

## Spring Boot Actuator简介
Spring Boot Actuator 的关键特性是在应用程序里提供众多 Web 接口，通过它们了解应用程序运行时的内部状况。

Actuator 提供了 13 个接口，可以分为三大类：配置接口、度量接口和其它接口，具体如下所示。
- /autoconfig	提供了一份自动配置报告，记录哪些自动配置条件通过了，哪些没通过
- /configprops	描述配置属性(包含默认值)如何注入Bean
- /beans	描述应用程序上下文里全部的Bean，以及它们的关系
- /dump	获取线程活动的快照
- /env	获取全部环境属性
- /env/{name}	根据名称获取特定的环境属性值
- /health	报告应用程序的健康指标，这些值由HealthIndicator的实现类提供
- /info	获取应用程序的定制信息，这些信息由info打头的属性提供
- /mappings	描述全部的URI路径，以及它们和控制器(包含Actuator端点)的映射关系
- /metrics	报告各种应用程序度量信息，比如内存用量和HTTP请求计数
- /metrics/{name}	报告指定名称的应用程序度量值
- /shutdown	关闭应用程序，要求endpoints.shutdown.enabled设置为true
- /trace	提供基本的HTTP请求跟踪信息(时间戳、HTTP头等)

## 使用介绍
actuator 的使用需要引入相应的依赖，如下：
```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```
需要注意 ，默认情况下 jmx是所有端点都暴露了, 而 http 方式只有 info 和 health 能用，我们可以通过如下配置来选择是否暴露其他端点

| 属性                                        | 意义       | 默认值          |
|-------------------------------------------|----------|--------------|
| management.endpoints.jmx.exposure.exclude | 暴露排除某些端点 |              |
| management.endpoints.jmx.exposure.include | 暴露某些端点   | *            |
| management.endpoints.web.exposure.exclude | 暴露排除某些端点 |              |
| management.endpoints.web.exposure.include | 暴露某些端点   | info, health |

如：在 Springboot 中我们可以通过如下配置开发所有端点或指定端点：
```yaml
management:
  endpoints:
    web:
      exposure:
      	# 指定开放的端点路径， * 代表全部
        include: "*"
  endpoint:
    beans:
	   # 是否启用
      enabled: true
```
当 服务启动后，我们可以通过 `http://{ip}:{port}/{context-path}/actuator` 来获取 `actuator` 的接口。(默认情况下，端点的访问根路径是 `actuator`，我们可以通过 `management.endpoints.base-path` 配置来配置访问路径）。

## 基础注解
下面我们来介绍一下 Springboot 提供的一些注解 。

### @Endpoint
`@Endpoint` 标注当前类作为一个端点类来处理。`@Endpoint` 会通过 `Web` 和 `Jmx` 两种方式暴露端点，如果我们想单独暴露 `Web` 场景或者 `Jmx` 场景，可以使用其子注解 `@WebEndpoint` 和 `@JmxEndpoint` 来进行单独暴露。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Endpoint {
	// 端点 id，作为访问路径
	String id() default "";
	// 是否启用，默认启用
	boolean enableByDefault() default true;
}
```
下面我们以`@WebEndpoint` 注解为例，我们可以看到相较于 `@Endpoint` 来说 `@WebEndpoint`多了一个 `@FilteredEndpoint` 注解， `@FilteredEndpoint` 作为 端点过滤器注解，可在 `@Endpoint` 上使用以实现隐式过滤的注释，通常用作技术特定端点注释的元注释。

简单来说 ：当前端点是否会暴露其中一个条件就是满足 `@FilteredEndpoint` 指定的过滤器的过滤条件。`WebEndpointFilter` 的作用则是校验当前端点发现器是否是 `WebEndpointDiscoverer`，如果是则当前端点可以考虑暴露（还有其他校验条件），否则当前端点不进行暴露，从而满足了 `@WebEndpoint` 注解暴露在Web 环境下的功能。

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
// 继承 @Endpoint 注解
@Endpoint
// 指定 WebEndpointFilter过滤器
@FilteredEndpoint(WebEndpointFilter.class)
public @interface WebEndpoint {
	// 端点 id，作为访问路径
	@AliasFor(annotation = Endpoint.class)
	String id();
	// 是否启用，默认启用
	@AliasFor(annotation = Endpoint.class)
	boolean enableByDefault() default true;
}
```

### @XxxOperation
@XxxOperation 指的是如下三个注解，该注解适用于方法上，标注方法的功能和访问方式，如下：

- @ReadOperation ：标识该方法为读操作，使用 GET 方式访问
- @WriteOperation ：标识该方法为写操作，使用 POST 方式访问
- @DeleteOperation ：标识该方法为删除操作，使用 DELETE 方式访问

额外的 还存在 @Selector 注解，用于标识方法入参，如下方法:
```
	// 我们可以通过使用 delete 方式调用 http://localhost:8080/actuator/demo/{i} 访问该接口
    @DeleteOperation
    public Integer delete(@Selector Integer i) {
        return num;
    }
```

### @EndpointExtension
@EndpointExtension 可以作为已有端点的扩展，允许将额外的技术特定operations添加到现有端点。一个端点只能存在一个扩展，并且当端点扩展类和原始端点中都存在同一个方法时，端点扩展里的方法会覆盖原始端点的方法调用。同样的 @EndpointExtension 也存在两个子注解 @EndpointWebExtension 和 @EndpointJmxExtension 分别应用于 Web 环境和 Jmx 环境的扩展，这里不再赘述。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface EndpointExtension {
	// 端点过滤器，指定过滤哪些方法
	Class<? extends EndpointFilter<?>> filter();
	// 作用于哪个端点
	Class<?> endpoint() default Void.class;
}
```

## 自定义 Endpoint
介绍完上面的注解，我们便可以通过 Springboot 提供的 @Endpoint 注解来完成自定义 Endpoint 的功能，自定义的过程也很简单：

- 使用 @Endpoint 注解标注当前类作为一个端点注册，指定id为访问路径
- 通过 @ReadOperation、@WriteOperation、@DeleteOperation 注解来标注暴露的方法，通过 @Selector 注解来指定参数。

如下定义一个简单的Endpoint ：
```
@Component
@Endpoint(id = "demo")
public class DemoEndpoint {
    private int num = 0;
    
	// GET请求 http://localhost:8080/actuator/demo
    @ReadOperation
    public int read() {
        return num;
    }
    
	// POST 请求 http://localhost:8080/actuator/demo/99
    @WriteOperation
    public int write(@Selector int i) {
        num += i;
        return num;
    }
    
	// DELETE请求 http://localhost:8080/actuator/demo/99
    @DeleteOperation
    public int delete(@Selector int i) {
        num -= 1;
        return num;
    }
}
```
上面我们可以看到自定义一个 Endpoint，需要使用 @Endpoint(id = "demo") 来指定一个类，其中id为这个 Endpoint 的id，可以简单认为即是暴露后的http访问路径。在需要暴露的方法上还需要加上如下三个注解：

- @ReadOperation ：标识该方法为读操作，使用 GET 方式访问
- @WriteOperation ：标识该方法为写操作，使用 POST 方式访问
- @DeleteOperation ：标识该方法为删除操作，使用 DELETE 方式访问

除此之外我们可以定义EndpointExtension 来扩展操作，如下：
```
@Component
@EndpointWebExtension(endpoint = DemoEndpoint.class)
// 可以指定过滤器，即待扩展的端点需要满足 类型是 DemoEndpoint 并且满足 DemoEndpointFilter 的过滤条件。
//@EndpointExtension(filter = DemoEndpointFilter.class, endpoint = DemoEndpoint.class)
public class DemoEndpointExtension {
    private int num = 0;

    @ReadOperation
    public int read() {
        System.out.println("DemoEndpointExtension.read");
        return num;
    }

    @WriteOperation
    public int write(@Selector int i) {
        System.out.println("DemoEndpointExtension.write");
        num += i;
        return num;
    }

    @DeleteOperation
    public int delete(@Selector int i) {
        System.out.println("DemoEndpointExtension.delete");
        num -= 1;
        return num;
    }
}
```
上面我们介绍了 Actuator 的基础功能和使用，下面我们来看看在 Springboot 中 Actuator 是如何实现的。

## Endpoint 自动引入
我们在使用 Actuator 功能时需要先引入 spring-boot-starter-actuator 依赖包，其中会引入 spring-boot-actuator-autoconfigure，而 spring-boot-actuator-autoconfigure 的利用Springboot 自动装配的特性引在 spring.factories 中引入了很多类完成Actuator 功能的启用 。

spring.factories 引入的类很多，我们这里不在贴出所有的类，下面我们看其中的几个关键类

- EndpointAutoConfiguration ：Actuator 的 起始配置类。
- WebEndpointAutoConfiguration ：Web 环境下的关键配置类，引入了很多核心类
- EndpointDiscoverer ：抽象类，是所有端点发现类的父类，完成了端点发现的功能，核心功能实现类。

### EndpointAutoConfiguration
EndpointAutoConfiguration 看名字也可以知道是一个配置类，具体如下：
```
@Configuration(proxyBeanMethods = false)
public class EndpointAutoConfiguration {
	// 参数类型映射转换，会对返回的参数类型做转换，
	@Bean
	@ConditionalOnMissingBean
	public ParameterValueMapper endpointOperationParameterMapper(
			@EndpointConverter ObjectProvider<Converter<?, ?>> converters,
			@EndpointConverter ObjectProvider<GenericConverter> genericConverters) {
		ConversionService conversionService = createConversionService(
				converters.orderedStream().collect(Collectors.toList()),
				genericConverters.orderedStream().collect(Collectors.toList()));
		return new ConversionServiceParameterValueMapper(conversionService);
	}

	private ConversionService createConversionService(List<Converter<?, ?>> converters,
			List<GenericConverter> genericConverters) {
		if (genericConverters.isEmpty() && converters.isEmpty()) {
			return ApplicationConversionService.getSharedInstance();
		}
		ApplicationConversionService conversionService = new ApplicationConversionService();
		converters.forEach(conversionService::addConverter);
		genericConverters.forEach(conversionService::addConverter);
		return conversionService;
	}
	// 提供结果缓存的支持
	@Bean
	@ConditionalOnMissingBean
	public CachingOperationInvokerAdvisor endpointCachingOperationInvokerAdvisor(Environment environment) {
		return new CachingOperationInvokerAdvisor(new EndpointIdTimeToLivePropertyFunction(environment));
	}
}
```
EndpointAutoConfiguration 向 Spring 容器中注入了两个类 ：

- ParameterValueMapper
参数类型映射。会将参数类型转换为所需的类型。比如我们请求的参数 是 String 类型的 “123”, 但是 Endpoint 的入参确实 Integer 类型，这时就会通过该类来对参数进行类型转换，从 String转换 Integer。
- CachingOperationInvokerAdvisor
看名字可以知道，这是一个缓存顾问，用于缓存 Operation。

### WebEndpointAutoConfiguration
WebEndpointAutoConfiguration 也是一个配置类，其中引入了 实现Actuator 功能的关键类，这里直接说明具体类的左右，不再贴出具体引入代码。

| 引入的Bean                                                   | 描述                                                                           |
|-----------------------------------------------------------|------------------------------------------------------------------------------|
| PathMapper                                                | 配置参数映射。对应 management.endpoints.web.path-mapping 参数 。                         |
| EndpointMediaTypes                                        | 默认情况下，由端点生成和使用的媒体类型                                                          |
| WebEndpointDiscoverer                                     | web 端点发现者，发现被@Endpoint 注解修饰的类作为端点。                                           |
| ControllerEndpointDiscoverer                              | controller 端点发现者，发现被 @ControllerEndpoint 或 @RestControllerEndpoint 修饰的l类作为端点 |
| PathMappedEndpoints                                       | 保存了所有 Endpoint 信息 以及访问基础路径                                                   |
| IncludeExcludeEndpointFilter<ExposableWebEndpoint>        | WEB 下的过滤器，默认只开启 /info 和 /health，可以通过配置控制哪些路径的端点开启或者关闭                        |
| IncludeExcludeEndpointFilter<ExposableControllerEndpoint> | controller 下的过滤器，可以通过配置控制哪些路径的端点开启或者关闭                                       |
| ServletEndpointDiscoverer                                 | servlet 上下文加载的 Servlet 端点发现者。                                                |

这里我们注意 WebEndpointAutoConfiguration 中注入了三个端点发现器 ：WebEndpointDiscoverer 、ControllerEndpointDiscoverer 、ServletEndpointDiscoverer 。这三个类都是 EndpointDiscoverer 的子类。

## EndpointDiscoverer
EndpointDiscoverer 是一个抽象类，是所有端点发现器的父类，借由其提供的一些方法，可以发现容器中的端点。

Springboot 默认提供了下面四种实现器，均继承于 EndpointDiscoverer，各自具备不同的端点发现规则和处理方式。

- JmxEndpointDiscoverer ： Jmx 端点发现器
- ServletEndpointDiscoverer ： Servlet 环境下的的端点发现器
- WebEndpointDiscoverer ：Web 的端点发现器。 存在子类 CloudFoundryWebEndpointDiscoverer
- ControllerEndpointDiscoverer ：Controller 的端点发现器

对于每一个端点发现器，其内部都存在一个 endpoints集合用来保存符合自身发现条件端点。而这个集合是通过 调用EndpointDiscoverer#getEndpoints 方法来发现的，如下：
```
	private volatile Collection<E> endpoints;
	
	.....
		
	@Override
	public final Collection<E> getEndpoints() {
		if (this.endpoints == null) {
			// 发现所有 Endpoint 并保存到 endpoints  中
			this.endpoints = discoverEndpoints();
		}
		return this.endpoints;
	}

	private Collection<E> discoverEndpoints() {
		// 1. 获取所有 EndpointBean
		Collection<EndpointBean> endpointBeans = createEndpointBeans();
		// 2. 添加扩展 Bean
		addExtensionBeans(endpointBeans);
		// 3. 转换为指定类型的 Endpoint
		return convertToEndpoints(endpointBeans);
	}
```
EndpointDiscoverer#getEndpoints 方法作为功能核心方法，会发现所有端点：如果当前发现器端点集合没有初始化，则通过 discoverEndpoints() 来发现端点。































