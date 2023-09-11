---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo集群组件Directory


## Directory 概念
在 Dubbo 中 存在 SPI 接口 org.apache.dubbo.rpc.cluster.Directory。即服务目录，用于存放服务提供列表。

Directory 即服务目录， 服务目录中存储了一些和服务提供者有关的信息，通过服务目录，服务消费者可获取到服务提供者的信息，比如 ip、端口、服务协议等。通过这些信息，服务消费者就可通过 Netty 等客户端进行远程调用。在一个服务集群中，服务提供者数量并不是一成不变的，如果集群中新增了一台机器，相应地在服务目录中就要新增一条服务提供者记录。或者，如果服务提供者的配置修改了，服务目录中的记录也要做相应的更新。如果这样说，服务目录和注册中心的功能不就雷同了吗？确实如此，这里这么说是为了方便大家理解。实际上服务目录在获取注册中心的服务配置信息后，会为每条配置信息生成一个 Invoker 对象，并把这个 Invoker 对象存储起来，这个 Invoker 才是服务目录最终持有的对象。Invoker 有什么用呢？看名字就知道了，这是一个具有远程调用功能的对象。讲到这大家应该知道了什么是服务目录了，它可以看做是 Invoker 集合，且这个集合中的元素会随注册中心的变化而进行动态调整。

简单来说 ：Directory 中保存了当前可以提供服务的服务提供者列表集合。当消费者进行服务调用时，会从 Directory 中按照某些规则挑选出一个服务提供者来提供服务。

## Directory 的种类
服务目录目前内置的实现有两个，分别为 StaticDirectory 和 RegistryDirectory，它们均是 AbstractDirectory 的子类。AbstractDirectory 实现了 Directory 接口，这个接口包含了一个重要的方法定义，即 list(Invocation)，用于列举 Invoker。

Directory 继承自 Node 接口，Node 这个接口继承者比较多，像 Registry、Monitor、Invoker 等均继承了这个接口。这个接口包含了一个获取配置信息的方法 Node#getUrl，实现该接口的类可以向外提供配置信息。除此之外， RegistryDirectory 实现了 NotifyListener 接口，当注册中心节点信息发生变化后，RegistryDirectory 可以通过此接口方法得到变更信息，并根据变更信息动态调整内部 Invoker 列表。

## AbstractDirectory
StaticDirectory 和 RegistryDirectory，它们均是 AbstractDirectory 的子类。所以我们这里先来看看 AbstractDirectory 中的公共逻辑。

AbstractDirectory 封装了 Invoker 列举流程，具体的列举逻辑则由子类实现，这是典型的模板模式。AbstractDirectory 的整个实现很简单
```java
public abstract class AbstractDirectory<T> implements Directory<T> {

    // logger
    private static final Logger logger = LoggerFactory.getLogger(AbstractDirectory.class);
	// 当前 注册中心URL 或者 直连URL
    private final URL url;
	
    private volatile boolean destroyed = false;
	// 消费者URL
    private volatile URL consumerUrl;

    protected RouterChain<T> routerChain;

    public AbstractDirectory(URL url) {
        this(url, null);
    }

    public AbstractDirectory(URL url, RouterChain<T> routerChain) {
        this(url, url, routerChain);
    }

    public AbstractDirectory(URL url, URL consumerUrl, RouterChain<T> routerChain) {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
		// 如果是注册中心的协议，则进行进一步解析
        if (url.getProtocol().equals(Constants.REGISTRY_PROTOCOL)) {
            Map<String, String> queryMap = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
            this.url = url.addParameters(queryMap).removeParameter(Constants.MONITOR_KEY);
        } else {
            this.url = url;
        }

        this.consumerUrl = consumerUrl;
        setRouterChain(routerChain);
    }
    
	// 根据调用信息获取到服务提供者列表
    @Override
    public List<Invoker<T>> list(Invocation invocation) throws RpcException {
        if (destroyed) {
            throw new RpcException("Directory already destroyed .url: " + getUrl());
        }
		// 这里直接交由子类实现，也就是 StaticDirectory 和 RegistryDirectory 的实现
        return doList(invocation);
    }

    @Override
    public URL getUrl() {
        return url;
    }

    public RouterChain<T> getRouterChain() {
        return routerChain;
    }

    public void setRouterChain(RouterChain<T> routerChain) {
        this.routerChain = routerChain;
    }

    protected void addRouters(List<Router> routers) {
        routers = routers == null ? Collections.emptyList() : routers;
        routerChain.addRouters(routers);
    }

    public URL getConsumerUrl() {
        return consumerUrl;
    }

    public void setConsumerUrl(URL consumerUrl) {
        this.consumerUrl = consumerUrl;
    }

    public boolean isDestroyed() {
        return destroyed;
    }

    @Override
    public void destroy() {
        destroyed = true;
    }

    protected abstract List<Invoker<T>> doList(Invocation invocation) throws RpcException;

}

```
这里需要关注的就是 AbstractDirectory#list 方法。当消费者进行服务调用时，Dubbo 会将调用相关信息封装成Invocation进行调用。在调用过程中会通过 AbstractDirectory#doList 来获取服务提供者列表。而 AbstractDirectory#doList的具体实现则交由子类 StaticDirectory 和 RegistryDirectory 来实现。

## StaticDirectory
StaticDirectory 即静态服务目录，顾名思义，它内部存放的 Invoker 是不会变动的。所以，理论上它和不可变 List 的功能很相似。下面我们来看一下这个类的实现。
```java
public class StaticDirectory<T> extends AbstractDirectory<T> {
    private static final Logger logger = LoggerFactory.getLogger(StaticDirectory.class);
	// Invoker 列表，在 StaticDirectory 构造时就已经初始化
    private final List<Invoker<T>> invokers;

    public StaticDirectory(List<Invoker<T>> invokers) {
        this(null, invokers, null);
    }

    public StaticDirectory(List<Invoker<T>> invokers, RouterChain<T> routerChain) {
        this(null, invokers, routerChain);
    }

    public StaticDirectory(URL url, List<Invoker<T>> invokers) {
        this(url, invokers, null);
    }

    public StaticDirectory(URL url, List<Invoker<T>> invokers, RouterChain<T> routerChain) {
        super(url == null && invokers != null && !invokers.isEmpty() ? invokers.get(0).getUrl() : url, routerChain);
        if (invokers == null || invokers.isEmpty())
            throw new IllegalArgumentException("invokers == null");
        this.invokers = invokers;
    }
	// 服务提供者的接口
    @Override
    public Class<T> getInterface() {
        return invokers.get(0).getInterface();
    }
	// 检测服务目录是否可用
    @Override
    public boolean isAvailable() {
        if (isDestroyed()) {
            return false;
        }
         // 只要有一个 Invoker 是可用的，就认为当前目录是可用的
        for (Invoker<T> invoker : invokers) {
            if (invoker.isAvailable()) {
                return true;
            }
        }
        return false;
    }

    @Override
    public void destroy() {
        if (isDestroyed()) {
            return;
        }
        // 调用父类销毁逻辑
        super.destroy();
         // 遍历 Invoker 列表，并执行相应的销毁逻辑
        for (Invoker<T> invoker : invokers) {
            invoker.destroy();
        }
        invokers.clear();
    }

	// 构建路由链
    public void buildRouterChain() {
        RouterChain<T> routerChain = RouterChain.buildChain(getUrl());
        routerChain.setInvokers(invokers);
        this.setRouterChain(routerChain);
    }
	
	// 通过路由链路来进行路由，获取最终的 Invoker 列表
    @Override
    protected List<Invoker<T>> doList(Invocation invocation) throws RpcException {
        List<Invoker<T>> finalInvokers = invokers;
        if (routerChain != null) {
            try {
            	// 进行服务路由，筛选出满足路由规则的服务列表。
                finalInvokers = routerChain.route(getConsumerUrl(), invocation);
            } catch (Throwable t) {
                logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
            }
        }
        return finalInvokers == null ? Collections.emptyList() : finalInvokers;
    }

}
```
StaticDirectory 的实现并不复杂，因为其是不可变的服务列表，相较于RegistryDirectory 较少了动态监听的功能。

## RegistryDirectory
RegistryDirectory 是一种动态服务目录，实现了 NotifyListener 接口，也是我们最常使用的Directory。当注册中心服务配置发生变化后，RegistryDirectory 可收到与当前服务相关的变化。收到变更通知后，RegistryDirectory 可根据配置变更信息刷新 Invoker 列表。

RegistryDirectory 中有几个比较重要的逻辑：
- Invoker 的列举逻辑(RegistryDirectory#doList) ：服务目录功能的本质是提供服务提供者列表。而当消费者进行服务调用时，会通过Directory#list -> AbstractDirectory#doList 的方式来获取服务提供者列表。
- 接收服务配置变更的逻辑(RegistryDirectory#notify)：RegistryDirectory并非像StaticDirectory 一样服务列表不可变，这就代表着，RegistryDirectory有着接受服务变更的功能。
- Invoker 列表的刷新逻辑(RegistryDirectory#refreshOverrideAndInvoker) ：上面说了 RegistryDirectory 有接收服务列表变化的功能，那么就需要 RegistryDirectory 在接收到列表变更后刷新本地服务列表。

## Invoker 的列举逻辑
RegistryDirectory#doList 的实现如下：
```java
	@Override
    public List<Invoker<T>> doList(Invocation invocation) {
    	// 校测是否可用：没有服务提供者 || 提供者被禁用时会抛出异常
        if (forbidden) {
            // ... 抛出异常
        }
		// 如果group设置的是可以匹配多个组，则直接返回当前的 Invoker 集合
        if (multiGroup) {
            return this.invokers == null ? Collections.emptyList() : this.invokers;
        }
		// 否则通过 路由链路进行路由
        List<Invoker<T>> invokers = null;
        try {
            // Get invokers from cache, only runtime routers will be executed.
            invokers = routerChain.route(getConsumerUrl(), invocation);
        } catch (Throwable t) {
            logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
        }
        return invokers == null ? Collections.emptyList() : invokers;
    }
```
其中 RouterChain#route 的实现如下：
```java
    public List<Invoker<T>> route(URL url, Invocation invocation) {
        List<Invoker<T>> finalInvokers = invokers;
        for (Router router : routers) {
        	// 这里将 finalInvokers 交由路由按照规则进行过滤，返回最终过滤后的 finalInvokers。
            finalInvokers = router.route(finalInvokers, url, invocation);
        }
        return finalInvokers;
    }
```
这里可以看到整个服务列举的过程很简单：将本地缓存的服务列表交由服务路由 router 进行筛选，将筛选后的服务列表返回。

我们这里需要注意的是，当消费者调用多分组的服务时，路由规则会失效。

## 接收服务配置变更的逻辑
RegistryDirectory 是一个动态服务目录，会随注册中心配置的变化进行动态调整。因此 RegistryDirectory 实现了 NotifyListener 接口，通过这个接口获取注册中心变更通知。

即服务消费者在启动时，会订阅 注册中心 providers、configurators、routers 节点，并设置回调函数为 RegistryDirectory#notify，当节点更新时，会调用 RegistryDirectory#notify 方法来进行本地的更新(在启动时会立刻调用一次该回调方法，用于同步当前节点配置)。
```java
	// 在调用该方法之前，会调用 AbstractRegistry#notify 中将URL 按照类别划分，再分别调用 RegistryDirectory#notify 方法。
  	@Override
    public synchronized void notify(List<URL> urls) {
    	//  对 URLs 进行合法性过滤
        List<URL> categoryUrls = urls.stream()
        		// 合法性组别校验，默认 providers
                .filter(this::isValidCategory)
                .filter(this::isNotCompatibleFor26x)
                .collect(Collectors.toList());

        /**
         * TODO Try to refactor the processing of these three type of urls using Collectors.groupBy()?
         */
         // 筛选出配置信息URL 并转换成 configurators 
        this.configurators = Configurator.toConfigurators(classifyUrls(categoryUrls, UrlUtils::isConfigurator))
                .orElse(configurators);
		// 筛选出路由URL 并转换成Router 添加到 AbstractDirectory#routerChain 中
		// RouterChain保存了服务提供者的URL列表转换为invoker列表和可用服务提供者对应的invokers列表和路由规则信息
        toRouters(classifyUrls(categoryUrls, UrlUtils::isRoute)).ifPresent(this::addRouters);

        // providers
        // 筛选出 提供者URL 并进行服务提供者的更新
        refreshOverrideAndInvoker(classifyUrls(categoryUrls, UrlUtils::isProvider));
    }
```
简述逻辑：RegistryDirectory 在接收到服务配置变化后，会按照类型进行划分(configurators、routers，providers) ，并分别进行处理。

## Invoker 列表的刷新逻辑
在上面 RegistryDirectory#notify 中，对URL进行了分组后分别进行处理，对于providers 节点的刷新，即服务提供者的列表刷新是比较重要的部分，其实现在RegistryDirectory#refreshOverrideAndInvoker中 ：
```java
	private void refreshOverrideAndInvoker(List<URL> urls) {
        // mock zookeeper://xxx?mock=return null
        // 重写URL（也就是把mock=return null等信息拼接到URL中）并保存到overrideDirectoryUrl中
        overrideDirectoryUrl();
        // 刷新 服务提供者 URL，根据URL 生成Invoker
        refreshInvoker(urls);
    }
	// 刷新 服务列表
	private void refreshInvoker(List<URL> invokerUrls) {
        Assert.notNull(invokerUrls, "invokerUrls should not be null");
		// 如果只有一个 协议为 empty 的url，则表明需要销毁所有协议，因为empty 协议为空协议，个人理解就是为了防止空url存在而生成的无意义的url
		 
        if (invokerUrls.size() == 1 && invokerUrls.get(0) != null && Constants.EMPTY_PROTOCOL.equals(invokerUrls
                .get(0)
                .getProtocol())) {
              // 设置禁止访问
            this.forbidden = true; // Forbid to access
            // invoker设置为空集合
            this.invokers = Collections.emptyList();
            routerChain.setInvokers(this.invokers);
            destroyAllInvokers(); // Close all invokers
        } else {
        	// 设置允许访问
            this.forbidden = false; // Allow to access
            // urlInvokerMap 需要使用本地引用，因为 urlInvokerMap自身可能随时变化，可能指向 null
            Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
            if (invokerUrls == Collections.<URL>emptyList()) {
                invokerUrls = new ArrayList<>();
            }
            // 这里防止并发更新，cachedInvokerUrls 被  volatile 修饰
            if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
                invokerUrls.addAll(this.cachedInvokerUrls);
            } else {
                this.cachedInvokerUrls = new HashSet<>();
                this.cachedInvokerUrls.addAll(invokerUrls);//Cached invoker urls, convenient for comparison
            }
            if (invokerUrls.isEmpty()) {
                return;
            }
            // 将 url 转换成 Invoker,key为 url.toFullString(), value 为 Invoker
            Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map

            // state change
            // If the calculation is wrong, it is not processed.
            if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0) {
                logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls
                        .toString()));
                return;
            }
			// 转化为不可修改 list，防止并发修改
            List<Invoker<T>> newInvokers = Collections.unmodifiableList(new ArrayList<>(newUrlInvokerMap.values()));
            // pre-route and build cache, notice that route cache should build on original Invoker list.
            // toMergeMethodInvokerMap() will wrap some invokers having different groups, those wrapped invokers not should be routed.
            // 保存服务提供者 Invoker
            routerChain.setInvokers(newInvokers);
            //  如果匹配多个 group，则进行合并
            this.invokers = multiGroup ? toMergeInvokerList(newInvokers) : newInvokers;
            this.urlInvokerMap = newUrlInvokerMap;

            try {
            	// 销毁无用的 Invoker
                destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // Close the unused Invoker
            } catch (Exception e) {
                logger.warn("destroyUnusedInvokers error. ", e);
            }
        }
    }

```
上面可以看到整个过程还是比较简单：
- 如果 invokerUrls 只有一个 empty 协议，则认为需要销毁所有 Invoker，则对Invoker进行销毁。
- 否则将 URL 转换为 Invoker ，并进行校验后赋值给 RegistryDirectory#invokers 、RegistryDirectory#urlInvokerMap、AbstractDirectory#routerChain 中。随后销毁无用的 Invoker，避免服务消费者调用已下线的服务的服务。

## RegistryDirectory#toInvokers
可以看到上面的关键的逻辑 在于 URL 转换为 Invoker ，即如下，
```java
Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);
```
toInvokers 实现如下：
```java
	// org.apache.dubbo.registry.integration.RegistryDirectory#toInvokers
    private Map<String, Invoker<T>> toInvokers(List<URL> urls) {
        Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<String, Invoker<T>>();
        if (urls == null || urls.isEmpty()) {
            return newUrlInvokerMap;
        }
        Set<String> keys = new HashSet<String>();
        String queryProtocols = this.queryMap.get(Constants.PROTOCOL_KEY);
        for (URL providerUrl : urls) {
            // If protocol is configured at the reference side, only the matching protocol is selected	
            // 检测服务提供者协议是否被服务消费者所支持
            // 如果消费者指定了了调用服务的协议，则只使用指定的协议，即如果消费者指定提供者协议类型为 dubbo，则只会需要协议类型为dubbo的提供者。
            if (queryProtocols != null && queryProtocols.length() > 0) {
                boolean accept = false;
                String[] acceptProtocols = queryProtocols.split(",");
                for (String acceptProtocol : acceptProtocols) {
                    if (providerUrl.getProtocol().equals(acceptProtocol)) {
                        accept = true;
                        break;
                    }
                }
                if (!accept) {
                   // 若服务提供者协议头不被消费者所支持，则忽略当前 providerUrl
                    continue;
                }
            }
            // 跳过 空协议
            if (Constants.EMPTY_PROTOCOL.equals(providerUrl.getProtocol())) {
                continue;
            }
            // 没有能够处理该协议的 Protocol实现类，则跳过
            if (!ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(providerUrl.getProtocol())) {
             	// .... 日志打印
                continue;
            }
            // 合并提供者url 顺序是 ： override > -D >Consumer > Provider
            URL url = mergeUrl(providerUrl);

            String key = url.toFullString(); // The parameter urls are sorted
            // 避免重复解析
            if (keys.contains(key)) { // Repeated url
                continue;
            }
            keys.add(key);
            // Cache key is url that does not merge with consumer side parameters, regardless of how the consumer combines parameters, if the server url changes, then refer again
            // 从缓存中获取 Invoker
            Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap; // local reference
            // 获取与 url 对应的 Invoker
            Invoker<T> invoker = localUrlInvokerMap == null ? null : localUrlInvokerMap.get(key);
            // 如果缓存中不存在当前URL对应的invoker，则进行远程引用
            if (invoker == null) { // Not in the cache, refer again
                try {
                    boolean enabled = true;
                    // 对 disabled 和 enabled 参数的校验，即服务是否启用
                    if (url.hasParameter(Constants.DISABLED_KEY)) {
                        enabled = !url.getParameter(Constants.DISABLED_KEY, false);
                    } else {
                        enabled = url.getParameter(Constants.ENABLED_KEY, true);
                    }
                    if (enabled) {
                    	// 调用 生成invoker委托类
                        invoker = new InvokerDelegate<T>(protocol.refer(serviceType, url), url, providerUrl);
                    }
                } catch (Throwable t) {
                    logger.error("Failed to refer invoker for interface:" + serviceType + ",url:(" + url + ")" + t.getMessage(), t);
                }
                // 将 Invoker 保存到 缓存 newUrlInvokerMap 中
                if (invoker != null) { // Put new invoker in cache
                    newUrlInvokerMap.put(key, invoker);
                }
            } else {
                newUrlInvokerMap.put(key, invoker);
            }
        }
        keys.clear();
        // 返回 url 转换成的 Invoker Map 
        return newUrlInvokerMap;
    }
```
可以看到整个逻辑
- 如果消费者指定了服务提供者的协议类型，则按照指定协议来获取URL
- 如果第一步获取成功，或者没有指定协议类型，则会对URL进行合并、缓存校验等过程
- 如果缓存中不存在当前URL的Invoker ，则会通过下面这一句来创建Invoker。
```java
  invoker = new InvokerDelegate<T>(protocol.refer(serviceType, url), url, providerUrl);
```
这里我们可以看到关键的 逻辑在 protocol.refer(serviceType, url)，由于当前服务的协议是Dubbo，所以这里的调用顺序应为：
```java
	refprotocol.refer =》 XxxProtocolWrapper#refer  => DubboProtocol#refer
```

## DubboProtocol#refer
DubboProtocol#refer 根据URL 创建了 Invoker，在这里会建立与服务提供者的网络连接。其具体实现如下：
```java
    @Override
    public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    	// 序列化优化
        optimizeSerialization(url);
        // create rpc invoker.
        // 创建
        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
        invokers.add(invoker);
        return invoker;
    }

	...
	
	private ExchangeClient[] getClients(URL url) {
        // whether to share connection
        // 是否共享连接
        boolean service_share_connect = false;
        // 获取连接数，默认为0，表示未配置
        int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
        // if not configured, connection is shared, otherwise, one connection for one service
         // 如果未配置 connections，则共享连接
        if (connections == 0) {
            service_share_connect = true;
            connections = 1;
        }

        ExchangeClient[] clients = new ExchangeClient[connections];
        for (int i = 0; i < clients.length; i++) {
            if (service_share_connect) {
            	// 获取共享客户端 ： getSharedClient 中会从缓存中获取，如果没有命中，则会调用 initClient 方法创建客户端
                clients[i] = getSharedClient(url);
            } else {
            	// 初始化新的客户端
                clients[i] = initClient(url);
            }
        }
        return clients;
    }
```
这里需要注意的是在创建连接的过程中, 由于一台机器可以提供多个服务，那么消费者在引用这些服务时会考虑是与这些服务建立一个共享连接，还是与每一个服务单独建立一个连接。这里可以通过 connections 设置数量来决定创建多少客户端连接，默认是共享同一个客户端。

下面我们看一下 DubboProtocol#getSharedClient 方法的实现：
```java
 /**
     * Get shared connection
     * 获取共享客户端
     */
    private ExchangeClient getSharedClient(URL url) {
        String key = url.getAddress();
        // 获取带有“引用计数”功能的 ExchangeClient
        ReferenceCountExchangeClient client = referenceClientMap.get(key);
        if (client != null) {
            if (!client.isClosed()) {
            	// 增加引用计数
                client.incrementAndGetCount();
                return client;
            } else {
                referenceClientMap.remove(key);
            }
        }

        locks.putIfAbsent(key, new Object());
        synchronized (locks.get(key)) {
            if (referenceClientMap.containsKey(key)) {
                return referenceClientMap.get(key);
            }
  			// 如果缓存没命中，则创建 ExchangeClient 客户端
            ExchangeClient exchangeClient = initClient(url);
            // 将 ExchangeClient 实例传给 ReferenceCountExchangeClient，这里使用了装饰模式
            client = new ReferenceCountExchangeClient(exchangeClient, ghostClientMap);
            referenceClientMap.put(key, client);
            ghostClientMap.remove(key);
            locks.remove(key);
            return client;
        }
    }


  /**
     * Create new connection
     * 创建一个新的连接
     */
    private ExchangeClient initClient(URL url) {

        // client type setting.
        // 从url获取客户端类型，默认为 netty
        String str = url.getParameter(Constants.CLIENT_KEY, url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_CLIENT));
		// 添加编解码和心跳包参数到 url 中
        url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
        // enable heartbeat by default
        url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));

        // BIO is not allowed since it has severe performance issue.
         // 检测客户端类型是否存在，不存在则抛出异常
        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported client type: " + str + "," +
                    " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
        }

        ExchangeClient client;
        try {
            // connection should be lazy
            // 获取 lazy 配置，并根据配置值决定创建的客户端类型
            if (url.getParameter(Constants.LAZY_CONNECT_KEY, false)) {
            	// 创建懒加载 ExchangeClient 实例
                client = new LazyConnectExchangeClient(url, requestHandler);
            } else {
            	// 创建普通 ExchangeClient 实例
                client = Exchangers.connect(url, requestHandler);
            }
        } catch (RemotingException e) {
            throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
        }
        return client;
    }

```
这里我们可以发现，无论是共享连接还是非共享连接，都会调用 DubboProtocol#initClient 方法来创建客户端。这里需要注意在创建客户端的时候，可以根据 LAZY_CONNECT_KEY (默认为 lazy) 配置来决定是否懒加载客户端，如果懒加载客户端，则会在第一次调用服务时才会创建与服务端的连接(也是调用 Exchangers.connect 方法)，如果不是懒加载，则在服务启动的时候便会与提供者建立连接。默认是立即加载，即消费者在启动时就会与提供者建立连接。

同时我们可以注意到 在非懒加载的情况下，当URl 转换成 Invoker 时，消费者便已经和提供者建立了链接（通过 Exchangers.connect(url, requestHandler) )，也即是说，默认情况下消费者在启动的时候将所有提供者URL转化为Invoker，即代表消费者启动时便已经和所有的提供者建立了连接。

## Directory 的调用过程
经过上面的介绍，我们对 Directory 的功能了解的差不多，下面我们看看 Directory 的调用过程。根据 Directory 的功能我们就可以推测出其调用是在消费端。

## Directory 的创建
当消费者服务启动时会通过 ReferenceConfig#createProxy 创建提供者的代理类。而在 ReferenceConfig#createProxy 方法中，完成了 Directory 的创建过程。

ReferenceConfig#createProxy 中会根据注册中心的数量或直连URL的数量，如果为单一URL，则创建 RegistryDirectory ，否则创建 StaticDirectory(也会创建 RegistryDirectory )。

## RegistryProtocol的创建
上面讲了，无论是多URL还是单URL都会执行 RegistryProtocol#refer，而在 RegistryProtocol#refer方法中，创建了 RegistryDirectory。即是说，无论是单URL 还是多 URL 都会调用 RegistryProtocol#refer 创建RegistryDirectory。

## StaticDirectory 的创建
在多注册中心或者多服务提供者时会创建StaticDirectory 作为服务目录。创建过程在ReferenceConfig#createProxy中，部分代码如下，其中 refprotocol.refer(interfaceClass, url) 即1.1 RegistryProtocol#refer 中的 RegistryProtocol#refer 方法
```java
         List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
         URL registryURL = null;
         for (URL url : urls) {
             invokers.add(refprotocol.refer(interfaceClass, url));
             if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                 registryURL = url; // use last registry url
             }
         }
         if (registryURL != null) { // registry url is available
             // use RegistryAwareCluster only when register's cluster is available
             URL u = registryURL.addParameter(Constants.CLUSTER_KEY, RegistryAwareCluster.NAME);
             // The invoker wrap relation would be: RegistryAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, will execute route) -> Invoker
             invoker = cluster.join(new StaticDirectory(u, invokers));
         } else { // not a registry url, must be direct invoke.
             invoker = cluster.join(new StaticDirectory(invokers));
         }

```
另外当消费端引用多个分组的服务时，dubbo会对每个分组创建一个对应的StaticDirectory对象。这一部分是在RegistryDirectory#toMergeInvokerList 中完成。

## Diectory 的回调
Diectory 接收到服务列表变化会通过回调方法通知自己。而由于StaticDirectory是不可变列表，所以不存在回调逻辑。






















