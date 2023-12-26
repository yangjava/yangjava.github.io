---
layout: post
categories: [Gateway]
description: none
keywords: Gateway
---
# 网关Gateway源码1AutoConfiguration

## 源码调试环境
spring-cloud-gateway 版本：2.2.6.RELEASE
github地址：spring-cloud/spring-cloud-gateway
核心模块：spring-cloud-gateway-server

## xxx-starter的通用分析思路
我们日常开发时，经常会看到各种与SpringBoot无缝集成的starter，要分析它们的大致做了哪些事，最简单的方式，就是马上去扫一眼它的自动装配配置目录：
```
resources/META-INF/spring.factories
```
里面涉及到自动装配相关的配置类，一般都会与框架自身的核心功能有关，我们可以通过这些配置类来一窥整体功能。

其实上述的配置并非存在于我们在项目pom中引用的spring-cloud-starter-gateway，而在于其pom中依赖的spring-cloud-gateway-server：
```xml
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-gateway-server</artifactId>
      <version>2.2.6.RELEASE</version>
      <scope>compile</scope>
    </dependency>
```
根据spring.factories中配置的类名，我们大致可以看出它们各自的作用。
```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.gateway.config.GatewayClassPathWarningAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayHystrixCircuitBreakerAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayResilience4JCircuitBreakerAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayLoadBalancerClientAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayNoLoadBalancerClientAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayMetricsAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayRedisAutoConfiguration,\
org.springframework.cloud.gateway.discovery.GatewayDiscoveryClientAutoConfiguration,\
org.springframework.cloud.gateway.config.SimpleUrlHandlerMappingGlobalCorsAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayReactiveLoadBalancerClientAutoConfiguration

org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.cloud.gateway.config.GatewayEnvironmentPostProcessor
```
我们首先要关心的就是：GatewayAutoConfiguration，毕竟它不像其他配置类的GateWayXxxAutoConfiguration的命名方式，说明它不是分管某一个小块配置，而是这个框架的核心配置类。
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnProperty(name = "spring.cloud.gateway.enabled", matchIfMissing = true)
@EnableConfigurationProperties
@AutoConfigureBefore({ HttpHandlerAutoConfiguration.class,
		WebFluxAutoConfiguration.class })
@AutoConfigureAfter({ GatewayLoadBalancerClientAutoConfiguration.class,
		GatewayClassPathWarningAutoConfiguration.class })
@ConditionalOnClass(DispatcherHandler.class)
public class GatewayAutoConfiguration {

	@Bean
	public StringToZonedDateTimeConverter stringToZonedDateTimeConverter() {
		return new StringToZonedDateTimeConverter();
	}

	@Bean
	public RouteLocatorBuilder routeLocatorBuilder(
			ConfigurableApplicationContext context) {
		return new RouteLocatorBuilder(context);
	}

	@Bean
	@ConditionalOnMissingBean
	public PropertiesRouteDefinitionLocator propertiesRouteDefinitionLocator(
			GatewayProperties properties) {
		return new PropertiesRouteDefinitionLocator(properties);
	}

	@Bean
	@ConditionalOnMissingBean(RouteDefinitionRepository.class)
	public InMemoryRouteDefinitionRepository inMemoryRouteDefinitionRepository() {
		return new InMemoryRouteDefinitionRepository();
	}

	@Bean
	@Primary
	public RouteDefinitionLocator routeDefinitionLocator(
			List<RouteDefinitionLocator> routeDefinitionLocators) {
		return new CompositeRouteDefinitionLocator(
				Flux.fromIterable(routeDefinitionLocators));
	}

	@Bean
	public ConfigurationService gatewayConfigurationService(BeanFactory beanFactory,
			@Qualifier("webFluxConversionService") ObjectProvider<ConversionService> conversionService,
			ObjectProvider<Validator> validator) {
		return new ConfigurationService(beanFactory, conversionService, validator);
	}

	@Bean
	public RouteLocator routeDefinitionRouteLocator(GatewayProperties properties,
			List<GatewayFilterFactory> gatewayFilters,
			List<RoutePredicateFactory> predicates,
			RouteDefinitionLocator routeDefinitionLocator,
			ConfigurationService configurationService) {
		return new RouteDefinitionRouteLocator(routeDefinitionLocator, predicates,
				gatewayFilters, properties, configurationService);
	}

	@Bean
	@Primary
	@ConditionalOnMissingBean(name = "cachedCompositeRouteLocator")
	// TODO: property to disable composite?
	public RouteLocator cachedCompositeRouteLocator(List<RouteLocator> routeLocators) {
		return new CachingRouteLocator(
				new CompositeRouteLocator(Flux.fromIterable(routeLocators)));
	}
	}
```
粗略扫一眼类上的注解：
```
@AutoConfigureBefore({ HttpHandlerAutoConfiguration.class,
      WebFluxAutoConfiguration.class })
```
@AutoConfigureBefore指定的是GatewayAutoConfiguration的Bean加载会早于在这两个配置类，其实就是指定配置的先后关系。这两个没在spring.factories中出现，并且属于Spring Web相关，暂时不管，继续往下看：
```
@AutoConfigureAfter({ GatewayLoadBalancerClientAutoConfiguration.class,
      GatewayClassPathWarningAutoConfiguration.class })
```
前面我们提到了@AutoConfigureBefore，同理，@AutoConfigureAfter的作用与它相反，就不赘述了。这里配置的两个类是在spring.factories中出现过的，可以查看一下内容。

除去这两个，spring.factories中还配置了不少其他的XxxAutoConfiguration类，我们挨个去看看，发现了一些有趣的事：
```
spring.factories中所有配置类的书写顺序，就是这些配置初始化的顺序。
```
不得不感慨作者的敬业程度啊，可能这就是所谓的代码即文档吧。

GatewayAutoConfiguration中还配置了很多核心的bean，在后文会提到。

## spring.factories中部分配置类的功能简述

### GatewayClassPathWarningAutoConfiguration
spring-cloud-gateway 基于webflux实现，这个配置类是用于检查项目是否正确的依赖了spring-boot-starter-webflux，而不是依赖了spring-boot-starter-web。所以如果在项目中的某些依赖暗含了对spring-boot-starter-web的依赖，是需要排除依赖关系的，如：
```
<dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
     <exclusions>
         <exclusion>
             <artifactId>spring-boot-starter-web</artifactId>
             <groupId>org.springframework.boot</groupId>
         </exclusion>
     </exclusions>
</dependency>
```
源码如下：
```
@Configuration(proxyBeanMethods = false)
@AutoConfigureBefore(GatewayAutoConfiguration.class)
public class GatewayClassPathWarningAutoConfiguration {

	private static final Log log = LogFactory
			.getLog(GatewayClassPathWarningAutoConfiguration.class);

	private static final String BORDER = "\n\n**********************************************************\n\n";

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(name = "org.springframework.web.servlet.DispatcherServlet")
	protected static class SpringMvcFoundOnClasspathConfiguration {

		public SpringMvcFoundOnClasspathConfiguration() {
			log.warn(BORDER
					+ "Spring MVC found on classpath, which is incompatible with Spring Cloud Gateway at this time. "
					+ "Please remove spring-boot-starter-web dependency." + BORDER);
		}

	}

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingClass("org.springframework.web.reactive.DispatcherHandler")
	protected static class WebfluxMissingFromClasspathConfiguration {

		public WebfluxMissingFromClasspathConfiguration() {
			log.warn(BORDER + "Spring Webflux is missing from the classpath, "
					+ "which is required for Spring Cloud Gateway at this time. "
					+ "Please add spring-boot-starter-webflux dependency." + BORDER);
		}

	}

}
```

### GatewayAutoConfiguration
spring-cloud-gateway核心配置类，里面初始化了很多核心bean，列出部分：

- GatewayProperties
- NettyConfiguration
- RouteDefinitionLocator
- PropertiesRouteDefinitionLocator
- RouteLocator
- RouteRefreshListener
- RoutePredicateFactory
- RoutePredicateHandlerMapping
- FilteringWebHandler
- AdaptCachedBodyGlobalFilter
- RetryGatewayFilterFactory
- PrefixPathGatewayFilterFactory
- GatewayControllerEndpoint

大致协作流程为：

- route层：RouteDefinitionLocator加载Route配置，通过RouteLocator获取Route；
- handler层：Route先后被RoutePredicateHandlerMapping、FilteringWebHandler处理；
- filter层：Route被各种XxxFilter处理。

## GatewayRedisAutoConfiguration
顾名思义，是关于Redis的配置，在配置了Redis时会加载两个Bean：
```
@Configuration(proxyBeanMethods = false)
@AutoConfigureAfter(RedisReactiveAutoConfiguration.class)
@AutoConfigureBefore(GatewayAutoConfiguration.class)
@ConditionalOnBean(ReactiveRedisTemplate.class)
@ConditionalOnClass({ RedisTemplate.class, DispatcherHandler.class })
class GatewayRedisAutoConfiguration {

	@Bean
	@SuppressWarnings("unchecked")
	public RedisScript redisRequestRateLimiterScript() {
		DefaultRedisScript redisScript = new DefaultRedisScript<>();
		redisScript.setScriptSource(new ResourceScriptSource(
				new ClassPathResource("META-INF/scripts/request_rate_limiter.lua")));
		redisScript.setResultType(List.class);
		return redisScript;
	}

	@Bean
	@ConditionalOnMissingBean
	public RedisRateLimiter redisRateLimiter(ReactiveStringRedisTemplate redisTemplate,
			@Qualifier(RedisRateLimiter.REDIS_SCRIPT_NAME) RedisScript<List<Long>> redisScript,
			ConfigurationService configurationService) {
		return new RedisRateLimiter(redisTemplate, redisScript, configurationService);
	}

}
```
很显然，第一个就是把指定路径：下的lua限流脚本，封装到一个DefaultRedisScript类型的Bean中去。然后通过：
```
List result = redisTemplate.execute(defaultRedisScript, keyList, argvMap);
```
来动态的修改限流策略。

其他的配置类大同小异，都是根据引入了哪些功能配置相应的Bean。

本篇我们大致了解了spring-cloud-gateway自动配置了哪些内容，在不清楚实现原理的前提下，可以通过这些类名、配置先后顺序等，来推断它的大致启动流程及处理逻辑。