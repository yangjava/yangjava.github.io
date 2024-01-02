---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码网络Netty
ElasticSearch的Client与Node节点之间的通信基于Netty，Node节点之间的通信也是基于Netty。Client与Node节点之间通信基于Http协议工作在应用层，节点之间Master-Node与Data-Node之间的通信基于TCP协议工作在网络层。

## NetworkModule初始化
在Node初始化节点，进行NetworkModule的初始化，代码如下：
```
//初始化NetworkModule
final NetworkModule networkModule = new NetworkModule(settings, false, pluginsService.filterPlugins(NetworkPlugin.class),
    threadPool, bigArrays, pageCacheRecycler, circuitBreakerService, namedWriteableRegistry, xContentRegistry,
    networkService, restController);
------------------------------------------------------------------------
/**
 * Creates a network module that custom networking classes can be plugged into.
 * @param settings The settings for the node
 * @param transportClient True if only transport classes should be allowed to be registered, false otherwise.
 */
public NetworkModule(Settings settings, boolean transportClient, List<NetworkPlugin> plugins, ThreadPool threadPool,
                     BigArrays bigArrays,
                     PageCacheRecycler pageCacheRecycler,
                     CircuitBreakerService circuitBreakerService,
                     NamedWriteableRegistry namedWriteableRegistry,
                     NamedXContentRegistry xContentRegistry,
                     NetworkService networkService, HttpServerTransport.Dispatcher dispatcher) {
    this.settings = settings;
    this.transportClient = transportClient;
}
```
List<NetworkPlugin> plugins来自pluginsService.filterPlugins(NetworkPlugin.class),PluginService初始化是加载本地插件：

plugins对应的是插件服务中，实现NetworkPlugin接口的类，对应如下三个实现类。

Netty4Plugin是属于transport-netty4 Moudle（默认实现，本文重点）；
NioTransportPlugin是属于transport-nio Plugin（NIO实现）；
Security是属于x-pack Plugin（安全通信 x-pack安全插件实现）；

## HttpServerTransport
遍历NetworkPlugin的插件，并分别注册了Rest和Transport的handler，实际使用时，取出来具体的handler来初始化。这里构建Netty4HttpServerTransport，是Node节点负责处理来自Client的请求。
```
//遍历NetworkPlugin的插件，并分别注册了Rest和Transport的handler，实际使用时，取出来具体的handler来初始化
for (NetworkPlugin plugin : plugins) {
    //构建Netty4HttpServerTransport 作为HttpServerTransport
    Map<String, Supplier<HttpServerTransport>> httpTransportFactory = plugin.getHttpTransports(settings, threadPool, bigArrays,
        pageCacheRecycler, circuitBreakerService, xContentRegistry, networkService, dispatcher);
    //TransprotClient已经废弃，transportClient只是false，表示来自Node的网络创建
    if (transportClient == false) {
        for (Map.Entry<String, Supplier<HttpServerTransport>> entry : httpTransportFactory.entrySet()) {
            // 1.Rest请求handler注册，处理来自Client的请求
            registerHttpTransport(entry.getKey(), entry.getValue());
        }
    }
```
plugin.getHttpTransports获取处理Http请求Map。
```
// 构建Netty4HttpServerTransport 作为HttpServerTransport（响应Client请求）
@Override
public Map<String, Supplier<HttpServerTransport>> getHttpTransports(Settings settings, ThreadPool threadPool, BigArrays bigArrays,
                                                                    PageCacheRecycler pageCacheRecycler,
                                                                    CircuitBreakerService circuitBreakerService,
                                                                    NamedXContentRegistry xContentRegistry,
                                                                    NetworkService networkService,
                                                                    HttpServerTransport.Dispatcher dispatcher) {
    return Collections.singletonMap(NETTY_HTTP_TRANSPORT_NAME,
        () -> new Netty4HttpServerTransport(settings, networkService, bigArrays, threadPool, xContentRegistry, dispatcher));
}
```
registerHttpTransport核心作用是注册Handler，放入transportHttpFactoriesMap 。
```
//负责对Client发Rest服务
private final Map<String, Supplier<HttpServerTransport>> transportHttpFactories = new HashMap<>();
```

## Transport
这里构建Netty4HttpServerTransport，是Node节点负责处理来自其他Node的请求。
```
//构建Netty4Transport作为Transport
Map<String, Supplier<Transport>> transportFactory = plugin.getTransports(settings, threadPool, pageCacheRecycler,
    circuitBreakerService, namedWriteableRegistry, networkService);
for (Map.Entry<String, Supplier<Transport>> entry : transportFactory.entrySet()) {
    // 2.Transport请求handler注册，处理来自同一集群下其他Node的请求
    registerTransport(entry.getKey(), entry.getValue());
}
```
plugin.getTransports 获取处理Node节点内部处理Map。
```
//构建Netty4Transport作为Transport（节点内部通信）
@Override
public Map<String, Supplier<Transport>> getTransports(Settings settings, ThreadPool threadPool, PageCacheRecycler pageCacheRecycler,
                                                      CircuitBreakerService circuitBreakerService,
                                                      NamedWriteableRegistry namedWriteableRegistry, NetworkService networkService) {
    return Collections.singletonMap(NETTY_TRANSPORT_NAME, () -> new Netty4Transport(settings, Version.CURRENT, threadPool,
        networkService, pageCacheRecycler, namedWriteableRegistry, circuitBreakerService));
}
```
registerTransport核心作用是把上一步生成的结果存入transportFactories。
```
//负责内部节点的RPC服务
private final Map<String, Supplier<Transport>> transportFactories = new HashMap<>();
```
TransportInterceptor
TransportInterceptor传输层拦截器，用于插件在发送和接收数据进行拦截（目前仅实现发送数据的拦截）这里可以忽略。

## Netty4HttpServerTransport初始化
Netty4HttpServerTransport初始化调用父类AbstractHttpServerTransport初始化，对应核心代码如下：
```
protected AbstractHttpServerTransport(Settings settings, NetworkService networkService, BigArrays bigArrays, ThreadPool threadPool,
                                      NamedXContentRegistry xContentRegistry, Dispatcher dispatcher) {
    ..........................................................
    // we can't make the network.bind_host a fallback since we already fall back to http.host hence the extra conditional here
    List<String> httpBindHost = SETTING_HTTP_BIND_HOST.get(settings);
    this.bindHosts = (httpBindHost.isEmpty() ? NetworkService.GLOBAL_NETWORK_BIND_HOST_SETTING.get(settings) : httpBindHost)
        .toArray(Strings.EMPTY_ARRAY);
    // we can't make the network.publish_host a fallback since we already fall back to http.host hence the extra conditional here
    List<String> httpPublishHost = SETTING_HTTP_PUBLISH_HOST.get(settings);
    this.publishHosts = (httpPublishHost.isEmpty() ? NetworkService.GLOBAL_NETWORK_PUBLISH_HOST_SETTING.get(settings) : httpPublishHost)
        .toArray(Strings.EMPTY_ARRAY);

    this.port = SETTING_HTTP_PORT.get(settings);

    this.maxContentLength = SETTING_HTTP_MAX_CONTENT_LENGTH.get(settings);
}
```
Netty4HttpServerTransport及其父类AbstractHttpServerTransport初始化核心是属性设置。

## Netty4Transport初始化
Netty4HttpServerTransport初始化调用父类TcpTransport初始化，对应核心代码如下：
```
public TcpTransport(Settings settings, Version version, ThreadPool threadPool, PageCacheRecycler pageCacheRecycler,
                    CircuitBreakerService circuitBreakerService, NamedWriteableRegistry namedWriteableRegistry,
                    NetworkService networkService) {
    ..........................................................
    //出站Handler
    this.outboundHandler = new OutboundHandler(nodeName, version, features, threadPool, bigArrays);
    //TCP 协议握手,发送一个实际的连接确定是否两节点之间在应用层面上可以通信
    this.handshaker = new TransportHandshaker(version, threadPool,
        //HandshakeRequestSender
        (node, channel, requestId, v) -> outboundHandler.sendRequest(node, channel, requestId,
            TransportHandshaker.HANDSHAKE_ACTION_NAME, new TransportHandshaker.HandshakeRequest(version),
            TransportRequestOptions.EMPTY, v, false, true),
        //HandshakeResponseSender
        (v, features1, channel, response, requestId) -> outboundHandler.sendResponse(v, features1, channel, requestId,
            TransportHandshaker.HANDSHAKE_ACTION_NAME, response, false, true));

    InboundMessage.Reader reader = new InboundMessage.Reader(version, namedWriteableRegistry, threadPool.getThreadContext());
    //长连接,虽然Netty中有SO_KEEPALIVE开关来保持TCP的长连接，但是应用层面的保活还是必要的。毕竟TCP是传输层，并不能保证应用的连通性
    this.keepAlive = new TransportKeepAlive(threadPool, this.outboundHandler::sendBytes);
    //入站Handler
    this.inboundHandler = new InboundHandler(threadPool, outboundHandler, reader, circuitBreakerService, handshaker,
        keepAlive);
}
```
这里有入站、出站Handler，长连接、TCP协议握手初始化工作。

## TransportService初始化
Node节点初始化，包含NetworkModule初始化（前两小节内容）
```
//初始化NetworkModule
final NetworkModule networkModule = new NetworkModule(settings, false, pluginsService.filterPlugins(NetworkPlugin.class),
    threadPool, bigArrays, pageCacheRecycler, circuitBreakerService, namedWriteableRegistry, xContentRegistry,
    networkService, restController);
Collection<UnaryOperator<Map<String, IndexTemplateMetaData>>> indexTemplateMetaDataUpgraders =
    pluginsService.filterPlugins(Plugin.class).stream()
        .map(Plugin::getIndexTemplateMetaDataUpgrader)
        .collect(Collectors.toList());
final MetaDataUpgrader metaDataUpgrader = new MetaDataUpgrader(indexTemplateMetaDataUpgraders);
final MetaDataIndexUpgradeService metaDataIndexUpgradeService = new MetaDataIndexUpgradeService(settings, xContentRegistry,
    indicesModule.getMapperRegistry(), settingsModule.getIndexScopedSettings());
new TemplateUpgradeService(client, clusterService, threadPool, indexTemplateMetaDataUpgraders);
//networkModule初始化后，获取对应的Transport
final Transport transport = networkModule.getTransportSupplier().get();
Set<String> taskHeaders = Stream.concat(
    pluginsService.filterPlugins(ActionPlugin.class).stream().flatMap(p -> p.getTaskHeaders().stream()),
    Stream.of(Task.X_OPAQUE_ID)
).collect(Collectors.toSet());
//网络服务封装
final TransportService transportService = newTransportService(settings, transport, threadPool,
    networkModule.getTransportInterceptor(), localNodeFactory, settingsModule.getClusterSettings(), taskHeaders);

//networkModule初始化后，获取Rest接口的handler
final HttpServerTransport httpServerTransport = newHttpTransport(networkModule);
```
getTransportSupplier
getTransportSupplier获取Rest接口Handler
```
/**
 * 在节点初始化时，会通过下面这个方法获取Rest接口的handler，依次读取transport.type和transport.type.default这两个配置。
 * 而ES默认的网络实现是通过transport-netty4插件实现的，在这个插件中，会设置transport.type.default配置。
 * 当用户没有自制自己的网络模块时，就会使用默认的netty实现。
 * 如果用户需要自定义时，只需要在插件中设置自己的网络模块名字，然后修改ES的transport.type.default配置就好
 */
public Supplier<Transport> getTransportSupplier() {
    final String name;
    if (TRANSPORT_TYPE_SETTING.exists(settings)) {
        //transport.type
        name = TRANSPORT_TYPE_SETTING.get(settings);
    } else {
        //transport.type.default
        name = TRANSPORT_DEFAULT_TYPE_SETTING.get(settings);
    }
    final Supplier<Transport> factory = transportFactories.get(name);
    if (factory == null) {
        throw new IllegalStateException("Unsupported transport.type [" + name + "]");
    }
    return factory;
}
```
newTransportService
TransportService初始化的核心作用是，Handler注册。即一个Action对应一个Handler注册到requestHandlers。
```
public TransportService(Settings settings, Transport transport, ThreadPool threadPool, TransportInterceptor transportInterceptor,
                        Function<BoundTransportAddress, DiscoveryNode> localNodeFactory, @Nullable ClusterSettings clusterSettings,
                        Set<String> taskHeaders, ConnectionManager connectionManager) {

    registerRequestHandler(
        HANDSHAKE_ACTION_NAME,
        ThreadPool.Names.SAME,
        false, false,
        HandshakeRequest::new,
        (request, channel, task) -> channel.sendResponse(
            new HandshakeResponse(localNode, clusterName, localNode.getVersion())));
}
----------------------------------------------------------------------------
/**registerRequestHandler注册Action和对应的处理类TransportRequest-Handler
 * 在ActionModule类中注册的RPC信息会自动在TransportService中添加映射关系。
 *
 * Registers a new request handler
 *
 * @param action                The action the request handler is associated with
 * @param requestReader               The request class that will be used to construct new instances for streaming
 * @param executor              The executor the request handling will be executed on
 * @param forceExecution        Force execution on the executor queue and never reject it
 * @param canTripCircuitBreaker Check the request size and raise an exception in case the limit is breached.
 * @param handler               The handler itself that implements the request handling
 */
public <Request extends TransportRequest> void registerRequestHandler(String action, //action对应handler的名字
                                                                      String executor,//executor对应了线程的类型，执行时会从线程池中拿相应类型的线程来处理request
                                                                      boolean forceExecution,
                                                                      boolean canTripCircuitBreaker,
                                                                      Writeable.Reader<Request> requestReader,//requestReader对应请求的reader，服务端根据reader来从网络IO流中反序列化出相应对象
                                                                      TransportRequestHandler<Request> handler) {//handler对应实际的handler，做实际的处理工作
    validateActionName(action);
    handler = interceptor.interceptHandler(action, executor, forceExecution, handler);
    RequestHandlerRegistry<Request> reg = new RequestHandlerRegistry<>(
        action, requestReader, taskManager, handler, executor, forceExecution, canTripCircuitBreaker);
    transport.registerRequestHandler(reg);
}
----------------------------------------------------------------------------
@Override
public synchronized <Request extends TransportRequest> void registerRequestHandler(RequestHandlerRegistry<Request> reg) {
    inboundHandler.registerRequestHandler(reg);
}
------------------------------------------------------------------------
//Map的key为Action名称，RequestHandlerRegistry 中封装了与RPC相关的Action名称、处理器等信息。
//通过它可以找到对应的处理模块。
requestHandlers = MapBuilder.newMapBuilder(requestHandlers).put(reg.getAction(), reg).immutableMap();
```
这里完成action与Handler的注册，方便接下来调用。

## newHttpTransport
NetWorkModule初始化后，根据name获取对应的HttpServerTransport。
```
/** Constructs a {@link org.elasticsearch.http.HttpServerTransport} which may be mocked for tests. */
protected HttpServerTransport newHttpTransport(NetworkModule networkModule) {
    return networkModule.getHttpServerTransportSupplier().get();
}
------------------------------------------------------------------------
/**
 * 在节点初始化时，会通过下面这个方法获取Rest接口的handler，依次读取http.type和http.default.type这两个配置。
 * 而ES默认的网络实现是通过transport-netty4插件实现的，在这个插件中，会设置http.default.type配置。
 * 当用户没有自制自己的网络模块时，就会使用默认的netty实现。
 * 如果用户需要自定义时，只需要在插件中设置自己的网络模块名字，然后修改ES的http.type配置就好
 */
public Supplier<HttpServerTransport> getHttpServerTransportSupplier() {
    final String name;
    if (HTTP_TYPE_SETTING.exists(settings)) {
        name = HTTP_TYPE_SETTING.get(settings);
    } else {
        name = HTTP_DEFAULT_TYPE_SETTING.get(settings);
    }
    final Supplier<HttpServerTransport> factory = transportHttpFactories.get(name);
    if (factory == null) {
        throw new IllegalStateException("Unsupported http.type [" + name + "]");
    }
    return factory;
}
```
## 用户自定义网络
ES默认的网络实现是通过transport-netty4插件实现的，在这个插件中，会设置http.default.type配置。

当用户没有自制自己的网络模块时，就会使用默认的netty实现。

如果用户需要自定义时，只需要在插件中设置自己的网络模块名字，然后修改ES的http.type配置并实现Module接口即可。

Node节点调用start方法：
```
/**
 * Start the node. If the node is already started, this method is no-op.
 */
public Node start() throws NodeValidationException {
    // 状态机，将local node的state设为STARTED状态
    if (!lifecycle.moveToStarted()) {
        return this;
    }

    logger.info("starting ...");
    // Node 的启动其实就是 node 里每个组件的启动。同样的，分别调用不同的实例的 start 方法来启动这个组件, 如下：
    pluginLifecycleComponents.forEach(LifecycleComponent::start);
```
LifecycleComponent::start是核心，启动所有服务，包括Netty4HttpServerTransport和Netty4Transport。

Netty4HttpServerTransport类的父类AbstractHttpServerTransport继承自AbstractLifecycleComponent，实现了LifecycleComponent接口；
Netty4Transport类的父类TcpTransport继承自AbstractLifecycleComponent，实现了LifecycleComponent接口；
Netty4HttpServerTransport类和Netty4Transport类属于服务端Node节点启动，不是Client客户端。服务端Node节点一方面响应来自Client的请求，一方面处理来自其他Node（可以是Master节点也可以是Data节点）的请求。

## AbstractLifecycleComponent
LifecycleComponent::start调用LifecycleComponent接口的实现类AbstractLifecycleComponent，对应的doStart方法由具体的子类实现。 这里AbstractLifecycleComponent对应的有生命周期设置不同的状态，启动前和启动后都可以设置不同的监听器。
```
public void start() {
    synchronized (lifecycle) {
        if (!lifecycle.canMoveToStarted()) {
            return;
        }
        for (LifecycleListener listener : listeners) {
            listener.beforeStart();
        }
        //由具体子类实现
        doStart();
        lifecycle.moveToStarted();
        for (LifecycleListener listener : listeners) {
            listener.afterStart();
        }
    }
}
```
## NettyAllocator
NettyAllocator主要用来判断使用Netty的缓冲区还是使用ES自定义的缓冲区。

Netty通过ByteBufAllocator分配器来创建缓冲区和分配内存空间。Netty提供了两种实现：PoolByteBufAllocator和UnpooledByteBufAllocator。

1.PoolByteBufAllocator将ByteBuf实例放入池中，提高了性能，将内存碎片减少到最小，采用了jemalloc高效内存分配的策略。

2.UnpooledByteBufAllocator是普通的未池化ByteBuf分配器，没有将ByteBuf放入池中，每次被调用时，返回一个新的ByteBuf实例。通过java的垃圾回收机制进行回收。

默认的分配器为ByteBufAllocator.DEFAULT（池化），可以通过java系统参数配置选项io.netty.allocator.type进行配置，配置时使用字符串值：unpooled，pooled。

内存管理的策略可以灵活调整，这是使用Netty所带来的又一个好处。只需一行简单的配置，就能获得到池化缓冲区带来的好处。

getServerChannelType
根据ALLOCATOR 类型返回不同的SocketChannel。
```
public static Class<? extends ServerChannel> getServerChannelType() {
    if (ALLOCATOR instanceof NoDirectBuffers) {
        return CopyBytesServerSocketChannel.class;
    } else {
        //Netty默认
        return NioServerSocketChannel.class;
    }
```
getChannelType
根据ALLOCATOR 类型返回不同的SocketChannel。
```
public static Class<? extends Channel> getChannelType() {
    if (ALLOCATOR instanceof NoDirectBuffers) {
        //ES 自定义 allow the disabling of netty direct buffer pooling while allowing us to
        // control how bytes end up being copied to direct memory.
        return CopyBytesSocketChannel.class;
    } else {
        //Netty默认
        return NioSocketChannel.class;
    }
}
```
ByteBufAllocator
ByteBufAllocator可以通过java系统参数配置选项io.netty.allocator.type进行配置，配置时使用字符串值：unpooled，pooled。
```
static {
    if (Booleans.parseBoolean(System.getProperty(USE_NETTY_DEFAULT), false)) {
        //Netty参数unpooled: UnpooledByteBufAllocator.DEFAULT
        //Netty参数pooled: PooledByteBufAllocator.DEFAULT
        //使用Netty的DirectBuffers
        ALLOCATOR = ByteBufAllocator.DEFAULT;
    } else {
        ByteBufAllocator delegate;
        if (useUnpooled()) {
            //netty不使用池化
            delegate = UnpooledByteBufAllocator.DEFAULT;
        } else {
            int nHeapArena = PooledByteBufAllocator.defaultNumHeapArena();
            int pageSize = PooledByteBufAllocator.defaultPageSize();
            int maxOrder = PooledByteBufAllocator.defaultMaxOrder();
            int tinyCacheSize = PooledByteBufAllocator.defaultTinyCacheSize();
            int smallCacheSize = PooledByteBufAllocator.defaultSmallCacheSize();
            int normalCacheSize = PooledByteBufAllocator.defaultNormalCacheSize();
            boolean useCacheForAllThreads = PooledByteBufAllocator.defaultUseCacheForAllThreads();
            //netty用池化，构建PooledByteBufAllocator
            delegate = new PooledByteBufAllocator(false, nHeapArena, 0, pageSize, maxOrder, tinyCacheSize,
                smallCacheSize, normalCacheSize, useCacheForAllThreads);
        }
        //不使用Netty的DirectBuffers
        ALLOCATOR = new NoDirectBuffers(delegate);
    }
}
--------------------------------------------------------------------
//Netty是否使用池化
private static boolean useUnpooled() {
    if (System.getProperty(USE_UNPOOLED) != null) {
        return Booleans.parseBoolean(System.getProperty(USE_UNPOOLED));
    } else {
        long heapSize = JvmInfo.jvmInfo().getMem().getHeapMax().getBytes();
        //Netty未使用池化,堆应该小于2的30次方（1G）
        return heapSize <= 1 << 30;
    }
}
```

## Netty4HttpServerTransport
Netty4HttpServerTransport类主要负责接受用户发来的Http请求，然后分发请求。对应的doStart方法如下：
```
protected void doStart() {
    boolean success = false;
    try {
        serverBootstrap = new ServerBootstrap();
        //NioEventLoopGroup可以认为是，mainReactor处理Socket对应的OP_ACCETP/OP_WRITE/OP_READ事件
        serverBootstrap.group(new NioEventLoopGroup(workerCount, daemonThreadFactory(settings,
            HTTP_SERVER_WORKER_THREAD_NAME_PREFIX)));

        // NettyAllocator will return the channel type designed to work with the configuredAllocator
        // 根据是否使用Netty默认的allocator区分CopyBytesServerSocketChannel（ES自定义）或NioServerSocketChannel
        serverBootstrap.channel(NettyAllocator.getServerChannelType());

        // Set the allocators for both the server channel and the child channels created
        // 与NettyAllocator.getChannelType()对应的NioSocketChannel保持一致
        serverBootstrap.option(ChannelOption.ALLOCATOR, NettyAllocator.getAllocator());
        serverBootstrap.childOption(ChannelOption.ALLOCATOR, NettyAllocator.getAllocator());
        /**
         * Channel初始化Netty4HttpRequestHandler（多种Handler真正的业务逻辑处理）
         * */
        serverBootstrap.childHandler(configureServerChannelHandler());
        /**
         * 异常处理Handler
         */
        serverBootstrap.handler(new ServerChannelExceptionHandler(this));

        serverBootstrap.childOption(ChannelOption.TCP_NODELAY, SETTING_HTTP_TCP_NO_DELAY.get(settings));
        serverBootstrap.childOption(ChannelOption.SO_KEEPALIVE, SETTING_HTTP_TCP_KEEP_ALIVE.get(settings));
        //transport.tcp.keep_alive  默认值为true
　　   .....................................

        final ByteSizeValue tcpSendBufferSize = SETTING_HTTP_TCP_SEND_BUFFER_SIZE.get(settings);
        if (tcpSendBufferSize.getBytes() > 0) {
            //网络socket发送数据缓存
            serverBootstrap.childOption(ChannelOption.SO_SNDBUF, Math.toIntExact(tcpSendBufferSize.getBytes()));
        }

        final ByteSizeValue tcpReceiveBufferSize = SETTING_HTTP_TCP_RECEIVE_BUFFER_SIZE.get(settings);
        if (tcpReceiveBufferSize.getBytes() > 0) {
            //网络socket接收数据缓存
            serverBootstrap.childOption(ChannelOption.SO_RCVBUF, Math.toIntExact(tcpReceiveBufferSize.getBytes()));
        }

        serverBootstrap.option(ChannelOption.RCVBUF_ALLOCATOR, recvByteBufAllocator);
        serverBootstrap.childOption(ChannelOption.RCVBUF_ALLOCATOR, recvByteBufAllocator);

        final boolean reuseAddress = SETTING_HTTP_TCP_REUSE_ADDRESS.get(settings);
        serverBootstrap.option(ChannelOption.SO_REUSEADDR, reuseAddress);
        serverBootstrap.childOption(ChannelOption.SO_REUSEADDR, reuseAddress);
        //创建一个 HTTP Server监听端口，当收到用户请求时，调用Netty4HttpRequestHandler的dispatchRequest对不同的请求执行相应的处理
        bindServer();
        success = true;
```
Netty4HttpServerTransport 会在自己启动的时候的中创建一个Netty服务器然后去监听9200端口。

## configureServerChannelHandler
Handler配置，Handler是业务处理的地方，涉及编码、解码、超时等处理器。
```
public ChannelHandler configureServerChannelHandler() {
    //handlingSettings是Http请求类的相关配置
    return new HttpChannelHandler(this, handlingSettings);
}
--------------------------------------------------------------
protected HttpChannelHandler(final Netty4HttpServerTransport transport, final HttpHandlingSettings handlingSettings) {
    this.transport = transport;
    this.handlingSettings = handlingSettings;
    this.requestHandler = new Netty4HttpRequestHandler(transport);
}

@Override
protected void initChannel(Channel ch) throws Exception {
    Netty4HttpChannel nettyHttpChannel = new Netty4HttpChannel(ch);
    ch.attr(HTTP_CHANNEL_KEY).set(nettyHttpChannel);
    //超时处理Handler
    ch.pipeline().addLast("read_timeout", new ReadTimeoutHandler(transport.readTimeoutMillis, TimeUnit.MILLISECONDS));
    final HttpRequestDecoder decoder = new HttpRequestDecoder(
        handlingSettings.getMaxInitialLineLength(),
        handlingSettings.getMaxHeaderSize(),
        handlingSettings.getMaxChunkSize());
    decoder.setCumulator(ByteToMessageDecoder.COMPOSITE_CUMULATOR);
    //解码Handler
    ch.pipeline().addLast("decoder", decoder);
    //请求内容压缩解码Handler
    ch.pipeline().addLast("decoder_compress", new HttpContentDecompressor());
    //编码Handler
    ch.pipeline().addLast("encoder", new HttpResponseEncoder());
    final HttpObjectAggregator aggregator = new HttpObjectAggregator(handlingSettings.getMaxContentLength());
    aggregator.setMaxCumulationBufferComponents(transport.maxCompositeBufferComponents);
    //超大Request和Response处理handler
    ch.pipeline().addLast("aggregator", aggregator);
    if (handlingSettings.isCompression()) {
        ch.pipeline().addLast("encoder_compress", new HttpContentCompressor(handlingSettings.getCompressionLevel()));
    }
    if (handlingSettings.isCorsEnabled()) {
        ch.pipeline().addLast("cors", new Netty4CorsHandler(transport.corsConfig));
    }
    ch.pipeline().addLast("pipelining", new Netty4HttpPipeliningHandler(logger, transport.pipeliningMaxEvents));
    //请求分发处理Handler
    ch.pipeline().addLast("handler", requestHandler);
    transport.serverAcceptedChannel(nettyHttpChannel);
}
```
Netty4HttpRequestHandler 继承了Netty中的抽象类SimpleChannelInboundHandler，只需要实现channelRead0这个抽象方法就能从网络IO中反序列化出来的HttpRequest对象。
```
Netty4HttpRequestHandler(Netty4HttpServerTransport serverTransport) {
    this.serverTransport = serverTransport;
}

//从网络IO中反序列化出来的HttpRequest对象。
@Override
protected void channelRead0(ChannelHandlerContext ctx, HttpPipelinedRequest<FullHttpRequest> msg) {
    //initChannel初始化设置HTTP_CHANNEL_KEY
    Netty4HttpChannel channel = ctx.channel().attr(Netty4HttpServerTransport.HTTP_CHANNEL_KEY).get();
    FullHttpRequest request = msg.getRequest();
    final FullHttpRequest copiedRequest;
    try {
        copiedRequest =
            new DefaultFullHttpRequest(
                request.protocolVersion(),
                request.method(),
                request.uri(),
                Unpooled.copiedBuffer(request.content()),
                request.headers(),
                request.trailingHeaders());
    } finally {
        // As we have copied the buffer, we can release the request
        request.release();
    }
    //从HttpPipelinedRequest读取数据
    Netty4HttpRequest httpRequest = new Netty4HttpRequest(copiedRequest, msg.getSequence());
    //若读取失败
    if (request.decoderResult().isFailure()) {
        Throwable cause = request.decoderResult().cause();
        if (cause instanceof Error) {
            ExceptionsHelper.maybeDieOnAnotherThread(cause);
            serverTransport.incomingRequestError(httpRequest, channel, new Exception(cause));
        } else {
            serverTransport.incomingRequestError(httpRequest, channel, (Exception) cause);
        }
    } else {
        //核心是调用dispatchRequest进行处理
        serverTransport.incomingRequest(httpRequest, channel);
    }
}
```

从网络IO序列号出HttpRequest对象，转换为Netty4HttpRequest，接下来调用serverTransport.incomingRequest方法分发不同的请求。这一步骤和Netty无关，和SpringMVC类似，发起一个Http请求由DispatchServletRequest对象执行分发。

## bindServer
创建一个 HTTP Server监听端口，当收到用户请求时，调用Netty4HttpRequestHandler的dispatchRequest对不同的请求执行相应的处理。

核心逻辑是执行bind，并把封装的httpServerChannel放入httpServerChannels（Set<HttpServerChannel>）
```
protected void bindServer() {
    // Bind and start to accept incoming connections.
    InetAddress hostAddresses[];
    try {
        hostAddresses = networkService.resolveBindHostAddresses(bindHosts);
    } catch (IOException e) {
        throw new BindHttpException("Failed to resolve host [" + Arrays.toString(bindHosts) + "]", e);
    }
    //hostAddresses可以是多个
    List<TransportAddress> boundAddresses = new ArrayList<>(hostAddresses.length);
    for (InetAddress address : hostAddresses) {
        //执行bind
        boundAddresses.add(bindAddress(address));
    }

    //publishHosts设置，只能是一个真实地址，若没有配置取默认值
    final InetAddress publishInetAddress;
    try {
        publishInetAddress = networkService.resolvePublishHostAddresses(publishHosts);
    } catch (Exception e) {
        throw new BindTransportException("Failed to resolve publish address", e);
    }

    final int publishPort = resolvePublishPort(settings, boundAddresses, publishInetAddress);
    TransportAddress publishAddress = new TransportAddress(new InetSocketAddress(publishInetAddress, publishPort));
    this.boundAddress = new BoundTransportAddress(boundAddresses.toArray(new TransportAddress[0]), publishAddress);
    logger.info("{}", boundAddress);
}
----------------------------------------------------------------------------------
private TransportAddress bindAddress(final InetAddress hostAddress) {
    final AtomicReference<Exception> lastException = new AtomicReference<>();
    final AtomicReference<InetSocketAddress> boundSocket = new AtomicReference<>();
    //bind的channel放入httpServerChannels
    boolean success = port.iterate(portNumber -> {
        try {
            synchronized (httpServerChannels) {
                //bind核心
                HttpServerChannel httpServerChannel = bind(new InetSocketAddress(hostAddress, portNumber));
                httpServerChannels.add(httpServerChannel);
                //InetSocketAddress 设置
                boundSocket.set(httpServerChannel.getLocalAddress());
            }
    ........................................
    return new TransportAddress(boundSocket.get());
}
----------------------------------------------------------------------------------
Netty4HttpServerTransport类bind方法
protected HttpServerChannel bind(InetSocketAddress socketAddress) throws Exception {
    //bind的核心
    ChannelFuture future = serverBootstrap.bind(socketAddress).sync();
    Channel channel = future.channel();
    Netty4HttpServerChannel httpServerChannel = new Netty4HttpServerChannel(channel);
    channel.attr(HTTP_SERVER_CHANNEL_KEY).set(httpServerChannel);
    return httpServerChannel;
}
```
network.bind_host对应bindHosts用于响应来自Client的请求，可以配置多个，默认是network.host。

network.publish_host对应publishInetAddress用于ES集群内Node节点之间通信。只能配置一个，若没有配置取默认值。

## Netty4Transport
Netty4Transport主要负责集群内Node的通信，应该是Elasticsearch 的RPC。显然一个Node节点既能作Client向其他Node发起请求，又能作Server响应其他节点处理业务逻辑。Netty4Transport的doStart方法如下：
```
protected void doStart() {
    boolean success = false;
    try {
        ThreadFactory threadFactory = daemonThreadFactory(settings, TRANSPORT_WORKER_THREAD_NAME_PREFIX);
        //NioEventLoopGroup可以认为是，mainReactor处理Socket对应的OP_ACCETP/OP_WRITE/OP_READ事件
        eventLoopGroup = new NioEventLoopGroup(workerCount, threadFactory);
        //初始化Client
        clientBootstrap = createClientBootstrap(eventLoopGroup);
        if (NetworkService.NETWORK_SERVER.get(settings)) {
            for (ProfileSettings profileSettings : profileSettings) {
                //根据配置创建服务端初始化Server
                createServerBootstrap(profileSettings, eventLoopGroup);
                //创建完一个服务端ServerBootstrap进行端口绑定
                bindServer(profileSettings);
            }
        }
        super.doStart();
        success = true;
    } finally {
        if (success == false) {
            doStop();
        }
    }
}
```

## createClientBootstrap
客户端的创建没有涉及具体的handler注册，因为这里返回的客户端Bootstrap仅仅起到一个模板的作用，后续具体需要发起通信（发送请求）时，会根据此Bootstrap克隆出一个具体的Bootstrap，然后注册handler。
```
//下面主要根据配置设置bootstrap的一些属性
private Bootstrap createClientBootstrap(NioEventLoopGroup eventLoopGroup) {
    final Bootstrap bootstrap = new Bootstrap();
    //NioEventLoopGroup可以认为是，mainReactor处理Socket对应的OP_ACCETP/OP_WRITE/OP_READ事件
    bootstrap.group(eventLoopGroup);

    // NettyAllocator will return the channel type designed to work with the configured allocator
    // 根据是否使用Netty默认的allocator区分CopyBytesServerSocketChannel（ES自定义）或NioServerSocketChannel
    bootstrap.channel(NettyAllocator.getChannelType());
    // 与NettyAllocator.getChannelType()对应的NioSocketChannel保持一致
    bootstrap.option(ChannelOption.ALLOCATOR, NettyAllocator.getAllocator());

    //是否使用Nagle算法属性配置
    bootstrap.option(ChannelOption.TCP_NODELAY, TransportSettings.TCP_NO_DELAY.get(settings));
    //连接属性设置，默认值true
    bootstrap.option(ChannelOption.SO_KEEPALIVE, TransportSettings.TCP_KEEP_ALIVE.get(settings));
    //transport.tcp.keep_alive  默认值为true
    if (TransportSettings.TCP_KEEP_ALIVE.get(settings)) {
       .........................
    }

    final ByteSizeValue tcpSendBufferSize = TransportSettings.TCP_SEND_BUFFER_SIZE.get(settings);
    if (tcpSendBufferSize.getBytes() > 0) {
        //网络socket发送数据缓存
        bootstrap.option(ChannelOption.SO_SNDBUF, Math.toIntExact(tcpSendBufferSize.getBytes()));
    }

    final ByteSizeValue tcpReceiveBufferSize = TransportSettings.TCP_RECEIVE_BUFFER_SIZE.get(settings);
    if (tcpReceiveBufferSize.getBytes() > 0) {
        //网络socket接收数据缓存
        bootstrap.option(ChannelOption.SO_RCVBUF, Math.toIntExact(tcpReceiveBufferSize.getBytes()));
    }
    //Netty适用对象池，重用缓冲区
    bootstrap.option(ChannelOption.RCVBUF_ALLOCATOR, recvByteBufAllocator);

    final boolean reuseAddress = TransportSettings.TCP_REUSE_ADDRESS.get(settings);
    bootstrap.option(ChannelOption.SO_REUSEADDR, reuseAddress);

    return bootstrap;
}
```

## createServerBootstrap
createServerBootstrap的代码和createClientBootstrap、Netty4HttpServerTransport的doStart方法，核心流程基本一致，只是设置参数不同。
```
private void createServerBootstrap(ProfileSettings profileSettings, NioEventLoopGroup eventLoopGroup) {
    String name = profileSettings.profileName;
    ................
    final ServerBootstrap serverBootstrap = new ServerBootstrap();
    //eventLoopGroupBoss可以认为是，mainReactor处理Socket对应的OP_ACCETP/OP_WRITE/OP_READ事件
    serverBootstrap.group(eventLoopGroup);

    // NettyAllocator will return the channel type designed to work with the configuredAllocator
    // 根据是否使用Netty默认的allocator区分CopyBytesServerSocketChannel（ES自定义）或NioServerSocketChannel
    serverBootstrap.channel(NettyAllocator.getServerChannelType());

    // Set the allocators for both the server channel and the child channels created
    // 与NettyAllocator.getChannelType()对应的NioSocketChannel保持一致
    serverBootstrap.option(ChannelOption.ALLOCATOR, NettyAllocator.getAllocator());
    serverBootstrap.childOption(ChannelOption.ALLOCATOR, NettyAllocator.getAllocator());
    /**
     * Channel初始化ServerChannelInitializer（多种Handler真正的业务逻辑处理）
     * 当客户端连接成功后向其pipeline设置handler，注册的handler则涉及到具体的编码、解码、粘包/拆包解决、报文处理等
     * */
    serverBootstrap.childHandler(getServerChannelInitializer(name));
    /**
     * 异常处理Handler
     */
    serverBootstrap.handler(new ServerChannelExceptionHandler());

    serverBootstrap.childOption(ChannelOption.TCP_NODELAY, profileSettings.tcpNoDelay);
    serverBootstrap.childOption(ChannelOption.SO_KEEPALIVE, profileSettings.tcpKeepAlive);
    if (profileSettings.tcpKeepAlive) {
        ................
    }

    if (profileSettings.sendBufferSize.getBytes() != -1) {
        //网络socket发送数据缓存
        serverBootstrap.childOption(ChannelOption.SO_SNDBUF, Math.toIntExact(profileSettings.sendBufferSize.getBytes()));
    }

    if (profileSettings.receiveBufferSize.getBytes() != -1) {
        //网络socket接收数据缓存
        serverBootstrap.childOption(ChannelOption.SO_RCVBUF, Math.toIntExact(profileSettings.receiveBufferSize.bytesAsInt()));
    }

    serverBootstrap.option(ChannelOption.RCVBUF_ALLOCATOR, recvByteBufAllocator);
    serverBootstrap.childOption(ChannelOption.RCVBUF_ALLOCATOR, recvByteBufAllocator);

    serverBootstrap.option(ChannelOption.SO_REUSEADDR, profileSettings.reuseAddress);
    serverBootstrap.childOption(ChannelOption.SO_REUSEADDR, profileSettings.reuseAddress);
    serverBootstrap.validate();
    //profileName与创建的serverBootstrap放入Map<String, ServerBootstrap>
    //每个Node都启动一个ServerBootstrap
    serverBootstraps.put(name, serverBootstrap);
}
```
getServerChannelInitializer
Handler配置，包含编码、解码、分发等handler。
```
protected ChannelHandler getServerChannelInitializer(String name) {
    return new ServerChannelInitializer(name);
}
-----------------------------------------------------------------
protected ServerChannelInitializer(String name) {
    this.name = name;
}

@Override
protected void initChannel(Channel ch) throws Exception {
    addClosedExceptionLogger(ch);
    Netty4TcpChannel nettyTcpChannel = new Netty4TcpChannel(ch, true, name, ch.newSucceededFuture());
    ch.attr(CHANNEL_KEY).set(nettyTcpChannel);
    //注册负责记录log的handler，但是进入ESLoggingHandler具体实现
    //可以看到其没有做日志记录操作，源码注释说明因为TcpTransport会做日志记录
    ch.pipeline().addLast("logging", new ESLoggingHandler());
    //注册解码器，这里没有注册编码器因为编码是在TcpTransport实现的，
    //需要发送的报文到达Channel已经是编码之后的格式了
    ch.pipeline().addLast("size", new Netty4SizeHeaderFrameDecoder());
    //负责对报文进行处理，主要识别是request还是response，然后进行相应的处理
    ch.pipeline().addLast("dispatcher", new Netty4MessageChannelHandler(Netty4Transport.this));
    serverAcceptedChannel(nettyTcpChannel);
}
```
Netty4MessageChannelHandler，消息入站处理Handler，通过transport.inboundMessage方法执行业务逻辑。
```
Netty4MessageChannelHandler(Netty4Transport transport) {
    this.transport = transport;
}

@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    assert Transports.assertTransportThread();
    assert msg instanceof ByteBuf : "Expected message type ByteBuf, found: " + msg.getClass();
    //Netty上下文处理
    final ByteBuf buffer = (ByteBuf) msg;
    try {
        Channel channel = ctx.channel();
        //AttributeKey.newInstance("es-channel")
        Attribute<Netty4TcpChannel> channelAttribute = channel.attr(Netty4Transport.CHANNEL_KEY);
        //调用TcpTransport.inboundMessage进行消息入站处理
        transport.inboundMessage(channelAttribute.get(), Netty4Utils.toBytesReference(buffer));
    } finally {
        buffer.release();
    }
}
```
bindServer
根据profileSettings进行端口绑定，和Netty4HttpServerTransport的bindServer可以对照着看。
```
//根据profileSettings进行端口绑定
protected void bindServer(ProfileSettings profileSettings) {
    // Bind and start to accept incoming connections.
    InetAddress[] hostAddresses;
    List<String> profileBindHosts = profileSettings.bindHosts;
    try {
        hostAddresses = networkService.resolveBindHostAddresses(profileBindHosts.toArray(Strings.EMPTY_ARRAY));
    } catch (IOException e) {
    .......................

    List<InetSocketAddress> boundAddresses = new ArrayList<>();
    for (InetAddress hostAddress : hostAddresses) {
        //端口绑定
        boundAddresses.add(bindToPort(profileSettings.profileName, hostAddress, profileSettings.portOrRange));
    }

    //构建TransportAddress[] boundAddresses 和 TransportAddress publishAddress
    final BoundTransportAddress boundTransportAddress = createBoundTransportAddress(profileSettings, boundAddresses);

    if (profileSettings.isDefaultProfile) {
        this.boundAddress = boundTransportAddress;
    } else {
        profileBoundAddresses.put(profileSettings.profileName, boundTransportAddress);
    }
}
---------------------------------------------------------
private InetSocketAddress bindToPort(final String name, final InetAddress hostAddress, String port) {
    PortsRange portsRange = new PortsRange(port);
    final AtomicReference<Exception> lastException = new AtomicReference<>();
    final AtomicReference<InetSocketAddress> boundSocket = new AtomicReference<>();
    closeLock.writeLock().lock();
    try {
        // No need for locking here since Lifecycle objects can't move from STARTED to INITIALIZED
        if (lifecycle.initialized() == false && lifecycle.started() == false) {
            throw new IllegalStateException("transport has been stopped");
        }
        boolean success = portsRange.iterate(portNumber -> {
            try {
                //服务绑定
                TcpServerChannel channel = bind(name, new InetSocketAddress(hostAddress, portNumber));
                serverChannels.computeIfAbsent(name, k -> new ArrayList<>()).add(channel);
                boundSocket.set(channel.getLocalAddress());
            } catch (Exception e) {
             
}
--------------------------------------------------------
Netty4Transport类bind方法
protected Netty4TcpServerChannel bind(String name, InetSocketAddress address) {
    //通过profileName获取对应的serverBootstrap，执行bind
    Channel channel = serverBootstraps.get(name).bind(address).syncUninterruptibly().channel();
    Netty4TcpServerChannel esChannel = new Netty4TcpServerChannel(channel);
    channel.attr(SERVER_CHANNEL_KEY).set(esChannel);
    return esChannel;
}
```
transport.portA bind port range. Defaults to 9300-9400.
transport.publish_port有防火墙或代理服务器，端口不对外暴露时，可以设置。不设置的话默认值是：transport.port.
这样的参数设置和Kafka网络相关参数设置基本一致，一个对外提供，一个内部节点交流，如果有防火墙或代理服务器则设置其他值。