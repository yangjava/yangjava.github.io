---
layout: post
categories: [SpringCloud]
description: none
keywords: SpringCloud
---
# SpringCloud负载均衡

## Ribbon
目前主流的负载方案分为两种：一种是集中式负载均衡，在消费者和服务提供方中间使用独立的代理方式进行负载，有硬件的（比如F5），也有软件的（比如Nginx）。另一种则是客户端自己做负载均衡，根据自己的请求情况做负载，Ribbon就属于客户端自己做负载。

如果用一句话介绍，那就是：Ribbon是Netflix开源的一款用于客户端负载均衡的工具软件。

GitHub地址：https://github.com/Netflix/ribbon。

## Ribbon模块
Ribbon模块如下：
- ribbon-loadbalancer负载均衡模块，可独立使用，也可以和别的模块一起使用。
- Ribbon内置的负载均衡算法都实现在其中。
- ribbon-eureka：基于Eureka封装的模块，能够快速、方便地集成Eureka。
- ribbon-transport：基于Netty实现多协议的支持，比如HTTP、Tcp、Udp等。
- ribbon-httpclient：基于Apache HttpClient封装的REST客户端，集成了负载均衡模块，可以直接在项目中使用来调用接口。
- ribbon-example：Ribbon使用代码示例，通过这些示例能够让你的学习事半功倍。
- ribbon-core：一些比较核心且具有通用性的代码，客户端API的一些配置和其他API的定义。

## Ribbon使用
接下来我们使用Ribbon来实现一个最简单的负载均衡调用功能，接口就用我们3.3.2节中提供的/user/hello接口，需要启动两个服务，一个是8081的端口，一个是8083的端口。然后创建一个新的Maven项目ribbon-native-demo，在项目中集成Ribbon，在pom.xml中添加代码清单4-1所示的依赖。
```xml
<dependency>
    <groupId>com.netflix.ribbon</groupId>
    <artifactId>ribbon</artifactId>
    <version>2.2.2</version>
</dependency>
<dependency>
    <groupId>com.netflix.ribbon</groupId>
    <artifactId>ribbon-core</artifactId>
    <version>2.2.2</version>
</dependency>
<dependency>
    <groupId>com.netflix.ribbon</groupId>
    <artifactId>ribbon-loadbalancer</artifactId>
    <version>2.2.2</version>
</dependency>
<dependency>
    <groupId>io.reactivex</groupId>
    <artifactId>rxjava</artifactId>
    <version>1.0.10</version>
</dependency>
```
Ribbon调用接口
```
// 服务列表
List<Server> serverList = Lists.newArrayList(
                        new Server("localhost", 8081),
                        new Server("localhost", 8083));
// 构建负载实例
ILoadBalancer loadBalancer = LoadBalancerBuilder.newBuilder()
                        .buildFixedServerListLoadBalancer(serverList);
// 调用 5 次来测试效果
for (int i = 0; i < 5; i++) {
    String result = LoadBalancerCommand.<String>builder()
            .withLoadBalancer(loadBalancer)
            .build()
            .submit(new ServerOperation<String>() {
                public Observable<String> call(Server server) {
                    try {
                        String addr = "http://" + server.getHost() + ":" + server. getPort () + "/user/hello";
                        System.out.println(" 调用地址：" + addr);
                        URL url = new URL(addr);
                        HttpURLConnection conn = (HttpURLConnection)
                                        url.openConnection();
                        conn.setRequestMethod("GET");
                        conn.connect();
                        InputStream in = conn.getInputStream();
                        byte[] data = new byte[in.available()];
                        in.read(data);
                        return Observable.just(new String(data));
                    } catch (Exception e) {
                        return Observable.error(e);
                    }
            }
    }).toBlocking().first(); System.out.println(" 调用结果：" + result);
}
```
上述这个例子主要演示了Ribbon如何去做负载操作，调用接口用的最底层的HttpURLConnection。当然你也可以用别的客户端，或者直接用Ribbon Client执行程序。

### RestTemplate结合Ribbon使用
Spring提供了一种简单便捷的模板类来进行API的调用，那就是RestTemplate。

新建一个HouseController，并增加两个接口，一个通过@RequestParam来传递参数，返回一个对象信息；另一个通过@PathVariable来传递参数，返回一个字符串。请尽量通过两个接口组装不同的形式
```
@GetMapping("/house/data")
public HouseInfo getData(@RequestParam("name") String name) {
    return new HouseInfo(1L, "上海" "虹口" "东体小区");
}

@GetMapping("/house/data/{name}")
public String getData2(@PathVariable("name") String name) {
    return name;
}

```
新建一个HouseClientController用于测试，使用RestTemplate来调用我们刚刚定义的两个接口
```
@GetMapping("/call/data")
public HouseInfo getData(@RequestParam("name") String name) {
    return restTemplate.getForObject( "http://localhost:8081/house/data?name="+ name, HouseInfo.class);
}

@GetMapping("/call/data/{name}")
public String getData2(@PathVariable("name") String name) {
    return restTemplate.getForObject( "http://localhost:8081/house/data/{name}", String.class, name);
}
```

### 整合Ribbon
在Spring Cloud项目中集成Ribbon只需要在pom.xml中加入下面的依赖即可，其实也可以不用配置，因为Eureka中已经引用了Ribbon
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```
修改RestTemplate的配置，增加能够让RestTemplate具备负载均衡能力的注解@LoadBalanced。
```java
@Configuration
public class BeanConfiguration {
    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }
}

```
修改接口调用的代码，将IP+PORT改成服务名称，也就是注册到Eureka中的名称
```
@GetMapping("/call/data")
public HouseInfo getData(@RequestParam("name") String name) {
        return restTemplate.getForObject(
    "http://ribbon-eureka-demo/house/data?name="+name, HouseInfo.class);
}
```
接口调用的时候，框架内部会将服务名称替换成具体的服务IP信息，然后进行调用。

## @LoadBalanced注解原理
相信大家一定有一个疑问：为什么在RestTemplate上加了一个@LoadBalanced之后，RestTemplate就能够跟Eureka结合了，不但可以使用服务名称去调用接口，还可以负载均衡？

应该归功于Spring Cloud给我们做了大量的底层工作，因为它将这些都封装好了，我们用起来才会那么简单。框架就是为了简化代码，提高效率而产生的。

这里主要的逻辑就是给RestTemplate增加拦截器，在请求之前对请求的地址进行替换，或者根据具体的负载策略选择服务地址，然后再去调用，这就是@LoadBalanced的原理。

下面我们来实现一个简单的拦截器，看看在调用接口之前会不会进入这个拦截器。我们不做任何操作，就输出一句话，证明能进来就行了。
```java
public class MyLoadBalancerInterceptor implements ClientHttpRequestInterceptor
{

    private LoadBalancerClient loadBalancer;
    private LoadBalancerRequestFactory requestFactory;

    public MyLoadBalancerInterceptor(LoadBalancerClient loadBalancer, LoadBalancerRequestFactory requestFactory) {
        this.loadBalancer = loadBalancer;
        this.requestFactory = requestFactory;
    }

    public MyLoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
        this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
    }

    @Override
    public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
            final ClientHttpRequestExecution execution) throws IOException {
        final URI originalUri = request.getURI();
        String serviceName = originalUri.getHost();
        System.out.println("进入自定义的请求拦截器中" + serviceName);
        Assert.state(serviceName != null, “Request URI does not contain a valid hostname: " + originalUri);
        return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
    }

}
```
拦截器设置好了之后，我们再定义一个注解，并复制@LoadBalanced的代码，改个名称就可以了
```java
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface MyLoadBalanced {
}
```
然后定义一个配置类，给RestTemplate注入拦截器
```java
@Configuration
public class MyLoadBalancerAutoConfiguration {
    @MyLoadBalanced 
    @Autowired(required = false)
    private List<RestTemplate> restTemplates = Collections.emptyList();
    @Bean
    public MyLoadBalancerInterceptor myLoadBalancerInterceptor() {
        return new MyLoadBalancerInterceptor();
    }
    @Bean
    public SmartInitializingSingleton
                    myLoadBalancedRestTemplateInitializer() {
        return new SmartInitializingSingleton() {
            @Override
            public void afterSingletonsInstantiated() {
                for (RestTemplate restTemplate : 
                    MyLoadBalancerAutoConfiguration.this.restTemplates){ 
                    List<ClientHttpRequestInterceptor> list = new
                            ArrayList<>(restTemplate.getInterceptors()); 
                    list.add(myLoadBalancerInterceptor()); 
                    restTemplate.setInterceptors(list);
                }
            }
        };
    }
}
```
维护一个@MyLoadBalanced的RestTemplate列表，在SmartInitializingSingleton中对RestTemplate进行拦截器设置。

然后改造我们之前的RestTemplate配置，将@LoadBalanced改成我们自定义的@MyLoadBalanced
```java
@Bean
//@LoadBalanced
@MyLoadBalanced
public RestTemplate getRestTemplate() {
    return new RestTemplate();
}
```

## @LoadBalanced的工作原理
首先看配置类，如何为RestTemplate设置拦截器，代码在spring-cloud-commons.jar中的org.springframework.cloud.client.loadbalancer.LoadBalancerAutoConfiguration类里面通过查看LoadBalancerAutoConfiguration的源码，可以看到这里也是维护了一个@LoadBalanced的RestTemplate列表
```
@LoadBalanced
@Autowired(required = false)
private List<RestTemplate> restTemplates = Collections.emptyList();
@Bean
public SmartInitializingSingleton loadBalancedRestTemplateInitializer(
                final List<RestTemplateCustomizer> customizers) {
    return new SmartInitializingSingleton() {
        @Override
        public void afterSingletonsInstantiated() {
            for (RestTemplate restTemplate : 
                LoadBalancerAutoConfiguration. this.restTemplates) {
                for (RestTemplateCustomizer customizer : customizers) {
                    customizer .customize(restTemplate);

                }
            }
        }
    };
}
```
通过查看拦截器的配置可以知道，拦截器用的是LoadBalancerInterceptor，RestTemplate Customizer用来添加拦截器
```java
@Configuration
@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
static class LoadBalancerInterceptorConfig {

    @Bean
    public LoadBalancerInterceptor ribbonInterceptor(
                    LoadBalancerClient loadBalancerClient, 
                    LoadBalancerRequestFactory requestFactory) {
        return new LoadBalancerInterceptor(loadBalancerClient,
                    requestFactory);
    }

    @Bean 
    @ConditionalOnMissingBean
    public RestTemplateCustomizer restTemplateCustomizer(
                final LoadBalancerInterceptor loadBalancerInterceptor) {
        return new RestTemplateCustomizer() {
            @Override
            public void customize(RestTemplate restTemplate) {
            List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                restTemplate.getInterceptors());

                list.add(loadBalancerInterceptor); 
                restTemplate.set Interceptors (list);
            }
        };
    }
}
```
拦截器的代码在org.springframework.cloud.client.loadbalancer.LoadBalancerInterceptor中
```java
public class LoadBalancerInterceptor implements
                                        ClientHttpRequestInterceptor {
    private LoadBalancerClient loadBalancer;
    private LoadBalancerRequestFactory requestFactory;

    public LoadBalancerInterceptor(LoadBalancerClient loadBalancer,
                        LoadBalancerRequestFactory requestFactory) {
        this.loadBalancer = loadBalancer;
        this.requestFactory = requestFactory;
    }

    public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
        this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
    }

    @Override
    public ClientHttpResponse intercept(final HttpRequest request,
            final byte[] body, final ClientHttpRequestExecution
                        execution) throws IOException {
        final URI originalUri = request.getURI(); 
        String serviceName = originalUri .getHost();
        Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
        return this.loadBalancer.execute(serviceName, requestFactory.createRequest (request, body, execution));
    }
}
```
主要的逻辑在intercept中，执行交给了LoadBalancerClient来处理，通过LoadBalancer RequestFactory来构建一个LoadBalancerRequest对象
```
public LoadBalancerRequest<ClientHttpResponse> createRequest(final
                                HttpRequest request, final byte[] body,
                        final ClientHttpRequestExecution execution) {
    return new LoadBalancerRequest<ClientHttpResponse>() {
        @Override
        public ClientHttpResponse apply(final ServiceInstance instance)
                                throws Exception {
            HttpRequest serviceRequest = new
                ServiceRequestWrapper(request, instance, loadBalancer);
            if (transformers != null) {
                for (LoadBalancerRequestTransformer transformer : transformers) {
                    serviceRequest = 
                        transformer.transformRequest(serviceRequest,instance);
                }
            }
            return execution.execute(serviceRequest, body);
        }
    };
}
```
createRequest中通过ServiceRequestWrapper来执行替换URI的逻辑，ServiceRequest Wrapper中将URI的获取交给了org.springframework.cloud.client.loadbalancer.LoadBalancer Client#reconstructURI方法。 以上就是整个RestTemplate结合@LoadBalanced的执行流程

## 负载均衡策略介绍
Ribbon作为一款客户端负载均衡框架，默认的负载策略是轮询，同时也提供了很多其他的策略，能够让用户根据自身的业务需求进行选择。

- BestAvailabl：选择一个最小的并发请求的Server，逐个考察Server，如果Server被标记为错误，则跳过，然后再选择ActiveRequestCount中最小的Server。
- AvailabilityFilteringRule：过滤掉那些一直连接失败的且被标记为circuit tripped的后端Server，并过滤掉那些高并发的后端Server或者使用一个AvailabilityPredicate来包含过滤Server的逻辑。其实就是检查Status里记录的各个Server的运行状态。
- ZoneAvoidanceRule：使用ZoneAvoidancePredicate和AvailabilityPredicate来判断是否选择某个Server，前一个判断判定一个Zone的运行性能是否可用，剔除不可用的Zone（的所有Server），AvailabilityPredicate用于过滤掉连接数过多的Server。
- RandomRule：随机选择一个Server。
- RoundRobinRule：轮询选择，轮询index，选择index对应位置的Server。
- RetryRule：对选定的负载均衡策略机上重试机制，也就是说当选定了某个策略进行请求负载时在一个配置时间段内若选择Server不成功，则一直尝试使用subRule的方式选择一个可用的Server。
- ResponseTimeWeightedRule：作用同WeightedResponseTimeRule，ResponseTime-Weighted Rule后来改名为WeightedResponseTimeRule。
- WeightedResponseTimeRule：根据响应时间分配一个Weight（权重），响应时间越长，Weight越小，被选中的可能性越低。

## 自定义负载策略
通过实现IRule接口可以自定义负载策略，主要的选择服务逻辑在choose方法中。我们这边只是演示怎么自定义负载策略，所以没写选择的逻辑，直接返回服务列表中第一个服务。
```java
public class MyRule implements IRule {

    private ILoadBalancer lb;

    @Override
    public Server choose(Object key) { 
        List<Server> servers = lb.getAllServers(); 
        for (Server server : servers) {
            System.out.println(server.getHostPort());
        }
        return servers.get(0);
    }

    @Override
    public void setLoadBalancer(ILoadBalancer lb){
        this.lb = lb;
    }

    @Override
    public ILoadBalancer getLoadBalancer(){
        return lb;
    }

}
```
在Spring Cloud中，可通过配置的方式使用自定义的负载策略，ribbon-config-demo是调用的服务名称。
```
ribbon-config-demo.ribbon.NFLoadBalancerRuleClassName=com.cxytiandi.ribbon_eureka_demo.rule.MyRule
```
重启服务，访问调用了其他服务的接口，可以看到控制台的输出信息中已经有了我们自定义策略中输出的服务信息，并且每次都是调用第一个服务。这跟我们的逻辑是相匹配的。


# 参考资料
Spring Cloud微服务架构进阶
Spring Cloud微服务：入门、实战与进阶
