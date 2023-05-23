---
layout: post
categories: [Gateway]
description: none
keywords: Gateway
---
# 网关Gateway源码解析

## 源码解析
作为后端服务的统一入口，API网关可提供请求路由、协议转换、安全认证、服务鉴权、流量控制与日志监控等服务。使用微服务架构将所有的应用管理起来，那么API网关就起到了微服务网关的作用；如果只是使用REST方式进行服务之间的访问，使用API网关对调用进行管理，那么API网关起到的就是API服务治理的作用。不管哪一种使用方式，都不影响API网关核心功能的实现。

具体步骤如下：
- 请求发送到网关，DispatcherHandler是HTTP请求的中央分发器，将请求匹配到相应的HandlerMapping。
- 请求与处理器之间有一个映射关系，网关将会对请求进行路由，handler此处会匹配到RoutePredicateHandlerMapping，以匹配请求所对应的Route。
- 随后到达网关的Web处理器，该WebHandler代理了一系列网关过滤器和全局过滤器的实例，如对请求或者响应的头部进行处理（增加或者移除某个头部）。
- 最后，转发到具体的代理服务。

这里比较重要的功能点是路由的过滤和路由的定位，Spring Cloud Gateway提供了非常丰富的路由过滤器和路由断言。下面将会按照自上而下的顺序分析这部分的源码。

## 初始化配置
在引入Spring Cloud Gateway的依赖后，Starter的jar包将会自动初始化一些类：
- GatewayLoadBalancerClientAutoConfiguration，客户端负载均衡配置类。
- GatewayRedisAutoConfiguration，Redis的自动配置类。
- GatewayDiscoveryClientAutoConfiguration，服务发现自动配置类。
- GatewayClassPathWarningAutoConfiguration，WebFlux依赖检查的配置类。
- GatewayAutoConfiguration，核心配置类，配置路由规则、过滤器等。

这些类的配置方式就不一一列出讲解了，主要看一下涉及的网关属性配置定义，很多对象的初始化都依赖于应用服务中配置的网关属性，GatewayProperties是网关中主要的配置属性类，代码如下所示：
```java
@ConfigurationProperties("spring.cloud.gateway")
@Validated
public class GatewayProperties {

    //路由列表
    @NotNull
    @Valid
    private List<RouteDefinition> routes = new ArrayList<>();

    private List<FilterDefinition> defaultFilters = new ArrayList<>();

    private List<MediaType> streamingMediaTypes = Arrays.asList(MediaType.TEXT_EVENT_STREAM,
            MediaType.APPLICATION_STREAM_JSON);
    ...
}
```
GatewayProperties中有三个属性，分别是路由、默认过滤器和MediaType的配置，在之前的基础应用的例子中演示了配置的前两个属性。routes是一个列表，对应的对象属性是路由定义RouteDefinition；defaultFilters是默认的路由过滤器，会应用到每个路由中；streamingMediaTypes默认支持两种类型：APPLICATION_STREAM_JSON和TEXT_EVENT_STREAM。

## 网关处理器
请求到达网关之后，会有各种Web处理器对请求进行匹配与处理。按照以下顺序讲解负责请求路由选择和定位的处理器：
```java
DispatcherHandler -> RoutePredicateHandlerMapping -> FilteringWebHandler -> DefaultGatewayFilterChain
```
### 请求的分发器
Spring Cloud Gateway引入了Spring WebFlux，DispatcherHandler是其访问入口，请求分发处理器。在之前的项目中，引入了Spring MVC，而它的分发处理器是DispatcherServlet。下面具体看一下网关收到请求后，如何匹配HandlerMapping，代码如下所示：
```java
public class DispatcherHandler implements WebHandler, ApplicationContextAware {
    @Override
    public Mono<Void> handle(ServerWebExchange exchange) {
        if (this.handlerMappings == null) {
            //不存在handlerMappings则报错
            return Mono.error(HANDLER_NOT_FOUND_EXCEPTION);
        }
        return Flux.fromIterable(this.handlerMappings)
                .concatMap(mapping -> mapping.getHandler(exchange))
                .next()
                .switchIfEmpty(Mono.error(HANDLER_NOT_FOUND_EXCEPTION))
                .flatMap(handler -> invokeHandler(exchange, handler))
                .flatMap(result -> handleResult(exchange, result));
    }
    ...
}
```
DispatcherHandler实现了WebHandler接口，WebHandler接口用于处理Web请求。DispatcherHandler的构造函数会初始化HandlerMapping。核心处理的方法是handle（ServerWebExchange exchange），而HandlerMapping是一个定义了请求与处理器对象映射的接口且有多个实现类，如ControllerEndpointHandlerMapping和RouterFunctionMapping。

可以看到handler映射共有六种实现，网关主要关注的是RoutePredicateHandlerMapping。RoutePredicateHandlerMapping继承了抽象类AbstractHandlerMapping，getHandler（exchange）方法就定义在该抽象类中，如下所示：
```java
public abstract class AbstractHandlerMapping extends ApplicationObjectSupport implements HandlerMapping, Ordered {
    @Override
    public Mono<Object> getHandler(ServerWebExchange exchange) {
        return getHandlerInternal(exchange).map(handler -> {
            ...
            return handler;
        });
    }

    protected abstract Mono<?> getHandlerInternal(ServerWebExchange exchange);
}

```
可以看出，抽象类在handler映射中用于抽取公用的功能，不是我们关注的重点，此处代码省略。具体的实现定义在HandlerMapping子类中。

AbstractHandlerMapping#getHandler返回了相应的Web处理器，随后到达DispatcherHandler#invokeHandler。

mapping#getHandler返回的是FilteringWebHandler。DispatcherHandler#invokeHandler方法调用相应的WebHandler，获取该WebHandler有对应的适配器。

### 路由断言的HandlerMapping
RoutePredicateHandlerMapping用于匹配具体的Route，并返回处理Route的FilteringWebHandler，如下所示：
```java
public class RoutePredicateHandlerMapping extends AbstractHandlerMapping {

    public RoutePredicateHandlerMapping(FilteringWebHandler webHandler, RouteLocator routeLocator) {
        this.webHandler = webHandler;
        this.routeLocator = routeLocator;
        setOrder(1);
    }
    ...
}
```
RoutePredicateHandlerMapping的构造函数接收两个参数：FilteringWebHandler网关过滤器和RouteLocator路由定位器，setOrder（1）用于设置该对象初始化的优先级。Spring Cloud Gateway的GatewayWebfluxEndpoint提供的HTTP API不需要经过网关转发，它通过RequestMappingHandlerMapping进行请求匹配处理，因此需要将RoutePredicateHandlerMapping的优先级设置为低于RequestMappingHandlerMapping。
```java
// RoutePredicateHandlerMapping.java
protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
    //设置GATEWAY_HANDLER_MAPPER_ATTR为 RoutePredicateHandlerMapping
    exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getClass().getSimpleName());

    return lookupRoute(exchange)
        .flatMap((Function<Route, Mono<?>>) r -> {
            //设置 GATEWAY_ROUTE_ATTR为匹配的Route
            exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
            return Mono.just(webHandler);
        }).switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> {
            //logger
        })));
}
//顺序匹配请求对应的Route
protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
    return this.routeLocator.getRoutes()
        .filterWhen(route ->  {
            exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, route.getId());
            return route.getPredicate().apply(exchange);
        })
        .next()
        .map(route -> {
            //校验 Route的有效性
            validateRoute(route, exchange);
            return route;
        });
}
```
如上为获取handler的方法，用于匹配请求的Route，并返回处理Route的FilteringWebHandler。首先设置GATEWAY_HANDLER_MAPPER_ATTR为RoutePredicateHandlerMapping的类名；然后顺序匹配请求对应的Route，RouteLocator接口用于获取在网关中定义的路由，并根据请求的信息，与路由定义的断言进行匹配（路由的定义也有优先级，按照优先级顺序匹配）。最后设置GATEWAY_ROUTE_ATTR为匹配的Route，并返回相应的handler。

### 过滤器的Web处理器

FilteringWebHandler通过创建所请求Route对应的GatewayFilterChain，在网关处进行过滤处理，实现代码如下：
```java
public class FilteringWebHandler implements WebHandler {
    private final List<GatewayFilter> globalFilters;

    public FilteringWebHandler(List<GlobalFilter> globalFilters) {
        this.globalFilters = loadFilters(globalFilters);
    }
    private static List<GatewayFilter> loadFilters(List<GlobalFilter> filters) {
        return filters.stream()
            .map(filter -> {
                //适配器模式，用以适配GlobalFilter
                GatewayFilterAdapter gatewayFilter = new GatewayFilterAdapter(filter);
                //判断是否实现Ordered接口
                if (filter instanceof Ordered) {
                    //实现了Ordered接口，则返回的是OrderedGatewayFilter对象
                    int order = ((Ordered) filter).getOrder();
                    return new OrderedGatewayFilter(gatewayFilter, order);
                }
                return gatewayFilter;
            }).collect(Collectors.toList());
    }
    ...
}

```
其中，全局变量globalFilters是Spring Cloud Gateway中定义的全局过滤器。构造函数通过传入的全局过滤器，对这些过滤器进行适配处理。因为过滤器的定义有优先级，这里的处理主要是判断是否实现Ordered接口，如果实现了Ordered接口，则返回的是OrderedGatewayFilter对象。否则，返回过滤器的适配器，用以适配GlobalFilter，适配器类比较简单，不再列出。最后将这些过滤器设置为全局变量globalFilters，如下所示：
```java
// FilteringWebHandler.java
public Mono<Void> handle(ServerWebExchange exchange) {
    Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
    List<GatewayFilter> gatewayFilters = route.getFilters();
    //加入全局过滤器
    List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
    combined.addAll(gatewayFilters);
    //过滤器排序
    AnnotationAwareOrderComparator.sort(combined);
    //按照优先级，对该请求进行过滤
    return new DefaultGatewayFilterChain(combined).filter(exchange);
}

private static class DefaultGatewayFilterChain implements GatewayFilterChain {
private final int index;
private final List<GatewayFilter> filters;
public DefaultGatewayFilterChain(List<GatewayFilter> filters) {
    this.filters = filters;
    this.index = 0;
}
private DefaultGatewayFilterChain(DefaultGatewayFilterChain parent, int index) {
    this.filters = parent.getFilters();
    this.index = index;
}
public List<GatewayFilter> getFilters() {
    return filters;
}

@Override
public Mono<Void> filter(ServerWebExchange exchange) {
    return Mono.defer(() -> {
        if (this.index < filters.size()) {
            GatewayFilter filter = filters.get(this.index);
            DefaultGatewayFilterChain chain = new DefaultGatewayFilterChain (this, this.index + 1);
            return filter.filter(exchange, chain);
        } else {
            return Mono.empty();
        }
    });
}
```
FilteringWebHandler#handle方法首先获取请求对应的路由的过滤器和全局过滤器，将两部分组合；然后对过滤器列表排序，AnnotationAwareOrderComparator是OrderComparator的子类，支持Spring的Ordered接口的优先级排序；最后按照优先级，生成过滤器链，对该请求进行过滤处理。这里过滤器链是通过内部静态类DefaultGatewayFilterChain实现，该类实现了GatewayFilterChain接口，用于按优先级过滤。

## 路由定义定位器








