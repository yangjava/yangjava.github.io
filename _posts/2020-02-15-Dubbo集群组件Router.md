---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo集群组件Router

## Router
服务目录在刷新 Invoker 列表的过程中，会通过 Router 进行服务路由，筛选出符合路由规则的服务提供者。在详细分析服务路由的源码之前，先来介绍一下服务路由是什么。服务路由包含一条路由规则，路由规则决定了服务消费者的调用目标，即规定了服务消费者可调用哪些服务提供者。

路由功能依赖于 Router 接口实现：
```java
public interface Router extends Comparable<Router> {
    /**
     * Get the router url.
     *
     * @return url
     */
     // 获取当前路由的url，即消费者的URl
    URL getUrl();

    // 完成请求路由的实际实现方法，根据一定规则将入参中的invokers 过滤，并返回
    <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
    
    // 通知路由器调用者列表。 调用者列表可能会不时更改。 此方法使路由器有机会在route(List, URL, Invocation)之前进行准备
    // 即当服务提供者的列表进行变化时会触发该方法。触发时机在  RegistryDirectory#refreshInvoker -》RouterChain#setInvokers
    default <T> void notify(List<Invoker<T>> invokers) {

    }

	// 决定此路由器是否需要在每次RPC到来时执行，还是仅在地址或规则更改时才执行。
    boolean isRuntime();

   // 要确定当任何调用者都无法匹配路由器规则时该路由器是否应生效，这意味着route(List, URL, Invocation)将为空。 大多数情况下，大多数路由器实现都会将此值默认设置为false。
    boolean isForce();

    // 路由优先级，由于存在多个路由，所以需要通过该参数决定路由执行优先级。越大优先级越高
    int getPriority();

    @Override
    default int compareTo(Router o) {
        if (o == null) {
            throw new IllegalArgumentException();
        }
        if (this.getPriority() == o.getPriority()) {
            if (o.getUrl() == null) {
                return 1;
            }
            if (getUrl() == null) {
                return -1;
            }
            return getUrl().toFullString().compareTo(o.getUrl().toFullString());
        } else {
            return getPriority() > o.getPriority() ? 1 : -1;
        }
    }
}
```
### 调用时机
Router 中有两个关键方法

- Router#notify
  完成了 路由信息的更新。当服务启动时或者标签路由更新时会通过此方法通知到当前服务。 调用时机在消费者端刷新本地的服务提供者列表时。即在 RegistryDirectory#refreshInvoker -》RouterChain#setInvokers 中。如下：
```java
    public void setInvokers(List<Invoker<T>> invokers) {
        this.invokers = (invokers == null ? Collections.emptyList() : invokers);
        routers.forEach(router -> router.notify(this.invokers));
    }

```
- Router#route
  完成了路由规则的实现。这里通过 RouterChain#routers 遍历来进行路由。
```java
   public List<Invoker<T>> route(URL url, Invocation invocation) {
        List<Invoker<T>> finalInvokers = invokers;
        for (Router router : routers) {
            finalInvokers = router.route(finalInvokers, url, invocation);
        }
        return finalInvokers;
    }
```
Dubbo 提供了下面六种 Router 的实现类，由对应的 RouterFactory 加载而来。
- ScriptRouter	脚本路由，脚本路由规则 4 支持 JDK 脚本引擎的所有脚本，比如：javascript, jruby, groovy 等，通过 type=javascript 参数设置脚本类型，缺省为 javascript
- ConditionRouter	条件路由，可以通过管理端设置一些匹配条件
- ServiceRouter	服务级别的路由，依赖于ConditionRouter 实现
- AppRouter	应用级路由器，依赖于ConditionRouter 实现
- TagRouter	标签路由，通过标签进行路由
- MockInvokersSelector	由 MockRouterFactory 加载而来，完成了 本地 mock 的功能
  默认流程下，RouterChain#routers 并不会加载所有的 Router。默认加载的是下面四个 ：
```java
// 其调用顺序如下：
	MockInvokersSelector =》 TagRouter =》 AppRouter =》ServiceRouter
```

## MockInvokersSelector
MockInvokersSelector 完成了本地mock 的功能

## TagRouter
- 标签路由
  标签路由通过将某一个或多个服务的提供者划分到同一个分组，约束流量只在指定分组中流转，从而实现流量隔离的目的，可以作为蓝绿发布、灰度发布等场景的能力基础。

标签主要是指对Provider端应用实例的分组，目前有两种方式可以完成实例分组，分别是动态规则打标和静态规则打标，其中动态规则相较于静态规则优先级更高，而当两种规则同时存在且出现冲突时，将以动态规则为准。

### 标签格式
- Key 明确规则体作用到哪个应用。必填。
- enabled=true 当前路由规则是否生效，可不填，缺省生效。
- force=false 当路由结果为空时，是否强制执行，如果不强制执行，路由结果为空的路由规则将自动失效，可不填，缺省为 false。
- runtime=false 是否在每次调用时执行路由规则，否则只在提供者地址列表变更时预先执行并缓存结果，调用时直接从缓存中获取路由结果。如果用了参数路由，必须设为 true，需要注意设置会影响调用的性能，可不填，缺省为 false。
- priority=1 路由规则的优先级，用于排序，优先级越大越靠前执行，可不填，缺省为 0。
- tags 定义具体的标签分组内容，可定义任意n（n>=1）个标签并为每个标签指定实例列表。必填。
- name， 标签名称
- addresses， 当前标签包含的实例列表

## 路由降级约定
request.tag=tag1 时优先选择 标记了tag=tag1 的 provider。若集群中不存在与请求标记对应的服务，默认将降级请求 tag为空的provider；如果要改变这种默认行为，即找不到匹配tag1的provider返回异常，需设置request.tag.force=true。

request.tag未设置时，只会匹配tag为空的provider。即使集群中存在可用的服务，若 tag 不匹配也就无法调用，这与约定1不同，携带标签的请求可以降级访问到无标签的服务，但不携带标签/携带其他种类标签的请求永远无法访问到其他标签的服务。

## 简单演示
两个服务的实现分别为：
```java
// 20880 端口
@Service(version = "1.0.0", group = "dubbo")
public class DemoServiceImpl implements DemoService {
    @Override
    public String sayHello(String name) {
        return "Spring Dubbo DemoServiceImpl name = " + name;
    }
}
// 9999 端口
@Service(version = "1.0.0", group = "dubbo")
public class DemoServiceImpl implements DemoService {
    @Override
    public String sayHello(String name) {
        return "MainDubbo DemoServiceImpl name = " + name;
    }
}
```
即代表，tag 为 spring 的访问端口为9999服务，tag 为main 的请求访问端口为 20880的服务。这里需要注意，addresses 中的 ip 需要和 服务列表中ip相同，不能写localhost、127.0.0.1 等。这里后面会分析。
请求访问，这里直接通过main 方法的方式访问
```java
	public static void main(String[] args) throws InterruptedException {
		// 自定义的方法 获取 ReferenceConfig
        ReferenceConfig<DemoService> referenceConfig = DubboUtil.referenceConfig("spring-dubbo-provider");
        referenceConfig.setCheck(false);
        DemoService demoService = referenceConfig.get();

        referenceConfig.setMonitor("http://localhost:8080");
 // 	 也可以通过这种方式设置全局的 tag,但是优先级基于 上下文设置的tag
 //      Map<String, String> map = Maps.newHashMap();
 //      map.put(Constants.TAG_KEY, "main");
 //      referenceConfig.setParameters(map);
        // 上下文设置tag
        RpcContext.getContext().setAttachment(Constants.TAG_KEY,"main");
        String result = demoService.sayHello("demo");
        // 输出 main result = Main Dubbo DemoServiceImpl name = demo
        System.out.println("main result = " + result);

        RpcContext.getContext().setAttachment(Constants.TAG_KEY,"spring");
        result = demoService.sayHello("demo");
		// 输出 spring result = Spring Dubbo DemoServiceImpl name = demo
        System.out.println("spring result = " + result);
    }
```
这里可以看到，对于 tag 配置为 main 的请求访问到了20880端口的服务上，对于 tag 为 spring的请求访问到了 9999 端口的服务上。

### TagRouter#notify
```java
    @Override
    public <T> void notify(List<Invoker<T>> invokers) {
        if (invokers == null || invokers.isEmpty()) {
            return;
        }

        Invoker<T> invoker = invokers.get(0);
        URL url = invoker.getUrl();
        // 获取服务提供者的 dubbo.application.name
        String providerApplication = url.getParameter(Constants.REMOTE_APPLICATION_KEY);

        if (StringUtils.isEmpty(providerApplication)) {
            return;
        }

        synchronized (this) {
        	// 判断是否是当前的服务提供者服务发生改变
            if (!providerApplication.equals(application)) {
                if (!StringUtils.isEmpty(application)) {
                    configuration.removeListener(application + RULE_SUFFIX, this);
                }
                // 更新配置中心 /dubbo/config 的监听。设置自身为监听，当节点更新时会调用process方法
              	// 我们设置的路由规则会保存到 /dubbo/config/applicationname 节点。
                String key = providerApplication + RULE_SUFFIX;
                configuration.addListener(key, this);
                application = providerApplication;
                // 获取最新的规则，并进行同步
                String rawRule = configuration.getConfig(key);
                if (rawRule != null) {
                    this.process(new ConfigChangeEvent(key, rawRule));
                }
            }
        }
    }
```
这样完成了，消费者启动后会触发TagRouter#notify 方法，而在 TagRouter#notify 方法中， TagRouter 完成了 /dubbo/config 的监听。当有路由设置进来时，会触发 TagRouter 的监听方法，即TagRouter#process，TagRouter#process 的 实现如下:
```java
 @Override
    public synchronized void process(ConfigChangeEvent event) {
        try {	
        	// 如果事件类型为 delete，则移除本地的路由规则
            if (event.getChangeType().equals(ConfigChangeType.DELETED)) {
                this.tagRouterRule = null;
            } else {
            	// 解析路由规则。这里的 event.getValue() 即我们在注册中心设置的yaml格式的值
                this.tagRouterRule = TagRuleParser.parse(event.getValue());
            }
        } catch (Exception e) {
        }
    }
```
这里的逻辑比较简单，如果是删除时间，则清空本地的路由规则，否则重新解析赋值。event.getValue() 值即为我们在注册中心设置的值 ：
```java
enabled: true
force: false
key: spring-dubbo-provider
priority: 0
runtime: true
tags:
- addresses:
  - 192.168.111.1:9999
  name: spring
- addresses:
  - 192.168.111.1:20880
  name: main
```
tagRouterRule 在解析后，保存了Tags 集合(其中保存了tags 节点的信息)，并通过两个map保存了 ip -> tag 和 tag -> ip 的映射

## TagRouter#route
当有请求通过时，会经过此方法路由，其实现如下：
```java
 	@Override
    public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException {
        if (CollectionUtils.isEmpty(invokers)) {
            return invokers;
        }
		// 如果动态路由没有配置，则匹配静态路由
        if (tagRouterRule == null || !tagRouterRule.isValid() || !tagRouterRule.isEnabled()) {
        	// 1. 对静态标签的匹配
            return filterUsingStaticTag(invokers, url, invocation);
        }
		// 2. 对动态路由的配置
        List<Invoker<T>> result = invokers;
        // 获取上下文路由tag，上下文路由优先级最高
        String tag = StringUtils.isEmpty(invocation.getAttachment(TAG_KEY)) ? url.getParameter(TAG_KEY) :
                invocation.getAttachment(TAG_KEY);

        // if we are requesting for a Provider with a specific tag
        // 如果当前请求指定了 tag
        if (StringUtils.isNotEmpty(tag)) {
        	// 获取路由tag指定的服务地址
            List<String> addresses = tagRouterRule.getTagnameToAddresses().get(tag);
            // filter by dynamic tag group first
            if (CollectionUtils.isNotEmpty(addresses)) {
            	// 从地址中过滤出来和当前请求URL匹配的 result
                result = filterInvoker(invokers, invoker -> addressMatches(invoker.getUrl(), addresses));
                // if result is not null OR it's null but force=true, return result directly
                // 如果过滤出来的结果不为空(则代表当前的tag 路由的服务存在） || force =true(强制使用tag路由)
                if (CollectionUtils.isNotEmpty(result) || tagRouterRule.isForce()) {
                    return result;
                }
            } else {
                // dynamic tag group doesn't have any item about the requested app OR it's null after filtered by
                // dynamic tag group but force=false. check static tag
                // 动态路由匹配失败，匹配静态路由
                result = filterInvoker(invokers, invoker -> tag.equals(invoker.getUrl().getParameter(TAG_KEY)));
            }
            // If there's no tagged providers that can match the current tagged request. force.tag is set by default
            // to false, which means it will invoke any providers without a tag unless it's explicitly disallowed.
            // 如果没有可以与当前已标记请求匹配的已标记提供程序。默认情况下，force.tag设置为false，这意味着除非明确禁止，否则它将调用任何没有标签的提供程序。
            if (CollectionUtils.isNotEmpty(result) || isForceUseTag(invocation)) {
                return result;
            }
            // FAILOVER: return all Providers without any tags.
            else {
                List<Invoker<T>> tmp = filterInvoker(invokers, invoker -> addressNotMatches(invoker.getUrl(),
                        tagRouterRule.getAddresses()));
                return filterInvoker(tmp, invoker -> StringUtils.isEmpty(invoker.getUrl().getParameter(TAG_KEY)));
            }
        } else {
            // List<String> addresses = tagRouterRule.filter(providerApp);
            // return all addresses in dynamic tag group.
            // 如果动态标签路由不为空，则需要将Invokers 中标签路由中 地址剔除。即下面所说的规则二场景。
            List<String> addresses = tagRouterRule.getAddresses();
            if (CollectionUtils.isNotEmpty(addresses)) {
            	// 如果 Invoker 代表的提供者，在动态路由标签中已经配置，则不允许再返回。
                result = filterInvoker(invokers, invoker -> addressNotMatches(invoker.getUrl(), addresses));
                // 1. all addresses are in dynamic tag group, return empty list.
                // 所有地址都在动态标签组中，返回空列表。
                if (CollectionUtils.isEmpty(result)) {
                    return result;
                }
                // 2. if there are some addresses that are not in any dynamic tag group, continue to filter using the
                // static tag group.
            }
            // 最后再与本地 tag 匹配校验
            return filterInvoker(result, invoker -> {
                String localTag = invoker.getUrl().getParameter(TAG_KEY);
                return StringUtils.isEmpty(localTag) || !tagRouterRule.getTagNames().contains(localTag);
            });
        }
    }
    
  // 对静态标签进行过滤
  private <T> List<Invoker<T>> filterUsingStaticTag(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        List<Invoker<T>> result = invokers;
        // Dynamic param
        // 1. 获取 当前需要匹配的 tag 参数信息。
        // 这里可以看到， 上下文的配置优于 url中的配置
        String tag = StringUtils.isEmpty(invocation.getAttachment(TAG_KEY)) ? url.getParameter(TAG_KEY) :
                invocation.getAttachment(TAG_KEY);
        // Tag request
        // 2. 如果需要进行tag 匹配则进行过滤
        if (!StringUtils.isEmpty(tag)) {
            result = filterInvoker(invokers, invoker -> tag.equals(invoker.getUrl().getParameter(Constants.TAG_KEY)));
            // 如果没有匹配上 && 并非强制匹配，则获取 tag = null 的 服务提供者 invoker 
            if (CollectionUtils.isEmpty(result) && !isForceUseTag(invocation)) {
                result = filterInvoker(invokers, invoker -> StringUtils.isEmpty(invoker.getUrl().getParameter(Constants.TAG_KEY)));
            }
        } else {
        	// 不需要tag匹配，则获取 tag = null 的服务提供者 invoker
            result = filterInvoker(invokers, invoker -> StringUtils.isEmpty(invoker.getUrl().getParameter(Constants.TAG_KEY)));
        }
        return result;
    }
    
    // 过滤 Invoker
    private <T> List<Invoker<T>> filterInvoker(List<Invoker<T>> invokers, Predicate<Invoker<T>> predicate) {
        return invokers.stream()
                .filter(predicate)
                .collect(Collectors.toList());
    }
    
  // 对  FORCE_USE_TAG 参数(dubbo.force.tag)校验，如果设置为 true，则强制匹配 tag，不可降级。
   private boolean isForceUseTag(Invocation invocation) {
        return Boolean.valueOf(invocation.getAttachment(FORCE_USE_TAG, url.getParameter(FORCE_USE_TAG, "false")));
    }

```
官方总结的规则描述如下：

规则一： request.tag=red 时优先选择 tag=red 的 provider。若集群中不存在与请求标记对应的服务，可以降级请求 tag=null 的 provider，即默认 provider。这里需要注意， 如果在上下文或者 URL参数中设置了 FORCE_USE_TAG 为 true (上下文配置优先于 URL 参数)，则表示强制匹配tag，则不会再降级匹配 tag = null 的服务。
规则二：request.tag=null 时，只会匹配 tag=null 的 provider。即使集群中存在可用的服务，若 tag 不匹配就无法调用，这与规则1不同，携带标签的请求可以降级访问到无标签的服务，但不携带标签/携带其他种类标签的请求永远无法访问到其他标签的服务。

## ConditionRouter
条件路由官方文档介绍的很清楚

基本使用 ：https://dubbo.apache.org/zh/docs/v2.7/user/examples/routing-rule/
源码分析：https://dubbo.apache.org/zh/docs/v2.7/dev/source/router/

## 多分组情况下路由失效
如果消费者多分组调用时，并且存在至少一个服务提供者的情况下，服务路由不起作用。 原因在于 RegistryDirectory#doList 中针对多分组情况直接返回了注册中心所有 Invokers。
首先需要明确下面的调用链路
```java
MockClusterInvoker#doMockInvoke =》 MockClusterInvoker#selectMockInvoker =》  RegistryDirectory#doList => RouterChain#route（在此方法中进行路由）
```
RegistryDirectory#doList 方法简化如下：
```java
    @Override
    public List<Invoker<T>> doList(Invocation invocation) {
    	// 如果是多分组情况下，会直接返回invokers,并不会进行路由
        if (multiGroup) {
            return this.invokers == null ? Collections.emptyList() : this.invokers;
        }
        List<Invoker<T>> invokers = null;
        // 在此进行服务路由。
        invokers = routerChain.route(getConsumerUrl(), invocation);
        return invokers == null ? Collections.emptyList() : invokers;
    }
```









