---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码服务详解


## 服务-GRPCChannelManager
org.apache.skywalking.apm.agent.core.boot.BootService是SkyWalking Agent中所有服务的顶层接口，

这个接口定义了一个服务的生命周期，分为prepare、boot、onComplete、shutdown四个阶段
```
public interface BootService {
    /**
     * 准备阶段
     *
     * @throws Throwable
     */
    void prepare() throws Throwable;

    /**
     * 启动阶段
     *
     * @throws Throwable
     */
    void boot() throws Throwable;

    /**
     * 启动完成阶段
     *
     * @throws Throwable
     */
    void onComplete() throws Throwable;

    /**
     * 关闭阶段
     *
     * @throws Throwable
     */
    void shutdown() throws Throwable;

    /**
     * 指定服务的优先级,优先级高的服务先启动,关闭的时候后关闭
     * {@code BootService}s with higher priorities will be started earlier, and shut down later than those {@code BootService}s with lower priorities.
     *
     * @return the priority of this {@code BootService}.
     */
    default int priority() {
        return 0;
    }
}
```
GRPCChannelManager是负责Agent到OAP的网络连接，通过GRPCChannel将Agent采集的链路信息和JVM的指标发送到OAP做分析、展示

### GRPCChannel
```
public class GRPCChannel {
    /**
     * origin channel
     */
    private final ManagedChannel originChannel;
    /**
     * 附带一些额外功能的channel 两个对象指向同一个链接
     */
    private final Channel channelWithDecorators;

    private GRPCChannel(String host, int port, List<ChannelBuilder> channelBuilders,
                        List<ChannelDecorator> decorators) throws Exception {
        ManagedChannelBuilder channelBuilder = NettyChannelBuilder.forAddress(host, port);

        NameResolverRegistry.getDefaultRegistry().register(new DnsNameResolverProvider());

        // ChannelBuilder.build(channelBuilder)
        for (ChannelBuilder builder : channelBuilders) {
            channelBuilder = builder.build(channelBuilder);
        }

        this.originChannel = channelBuilder.build();

        // 使用装饰器ChannelDecorator装饰originChannel
        Channel channel = originChannel;
        for (ChannelDecorator decorator : decorators) {
            channel = decorator.build(channel);
        }

        channelWithDecorators = channel;
    }
```
GRPCChannel的构造函数中，先通过ChannelBuilder构建originChannel，然后使用装饰器ChannelDecorator装饰originChannel从而得到channelWithDecorators

ChannelBuilder接口代码如下：
```
public interface ChannelBuilder<B extends ManagedChannelBuilder> {
    B build(B managedChannelBuilder) throws Exception;
}
```
ChannelBuilder有两个实现StandardChannelBuilder和TLSChannelBuilder

StandardChannelBuilder设置了接收数据的最大容量，明文传输
```
public class StandardChannelBuilder implements ChannelBuilder {
    // 50M
    private final static int MAX_INBOUND_MESSAGE_SIZE = 1024 * 1024 * 50;

    @Override
    public ManagedChannelBuilder build(ManagedChannelBuilder managedChannelBuilder) {
        // 设置接收数据的最大容量,明文传输
        return managedChannelBuilder.maxInboundMessageSize(MAX_INBOUND_MESSAGE_SIZE)
                                    .usePlaintext();
    }
}
```

TLSChannelBuilder设置加密传输
```
public class TLSChannelBuilder implements ChannelBuilder<NettyChannelBuilder> {
    private static String CA_FILE_NAME = "ca" + Constants.PATH_SEPARATOR + "ca.crt";

    @Override
    public NettyChannelBuilder build(
        NettyChannelBuilder managedChannelBuilder) throws AgentPackageNotFoundException, SSLException {
        // 在Agent目录下找到ca文件做认证,设置加密传输
        File caFile = new File(AgentPackagePath.getPath(), CA_FILE_NAME);
        boolean isCAFileExist = caFile.exists() && caFile.isFile();
        if (Config.Agent.FORCE_TLS || isCAFileExist) {
            SslContextBuilder builder = GrpcSslContexts.forClient();
            if (isCAFileExist) {
                builder.trustManager(caFile);
            }
            managedChannelBuilder = managedChannelBuilder.negotiationType(NegotiationType.TLS)
                                                         .sslContext(builder.build());
        }
        return managedChannelBuilder;
    }
}
```

ChannelDecorator接口代码如下：
```
public interface ChannelDecorator {
    Channel build(Channel channel);
}
```

ChannelDecorator有两个实现AgentIDDecorator和AuthenticationDecorator

AgentIDDecorator装饰的Channel向OAP发送数据的时候，请求头中会带上Agent版本号
```
public class AgentIDDecorator implements ChannelDecorator {
    private static final ILog LOGGER = LogManager.getLogger(AgentIDDecorator.class);
    private static final Metadata.Key<String> AGENT_VERSION_HEAD_HEADER_NAME = Metadata.Key.of("Agent-Version", Metadata.ASCII_STRING_MARSHALLER);
    private String version = "UNKNOWN";

    @Override
    public Channel build(Channel channel) {
        return ClientInterceptors.intercept(channel, new ClientInterceptor() {
            @Override
            public <REQ, RESP> ClientCall<REQ, RESP> interceptCall(MethodDescriptor<REQ, RESP> method,
                CallOptions options, Channel channel) {
                return new ForwardingClientCall.SimpleForwardingClientCall<REQ, RESP>(channel.newCall(method, options)) {
                    @Override
                    public void start(Listener<RESP> responseListener, Metadata headers) {
                        // 向OAP发送数据的时候,请求头中带上Agent版本号
                        headers.put(AGENT_VERSION_HEAD_HEADER_NAME, version);

                        super.start(responseListener, headers);
                    }
                };
            }
        });
    }
```

AuthenticationDecorator装饰的Channel向OAP发送数据的时候，请求头中带上的token
```
public class AuthenticationDecorator implements ChannelDecorator {
    private static final Metadata.Key<String> AUTH_HEAD_HEADER_NAME = Metadata.Key.of("Authentication", Metadata.ASCII_STRING_MARSHALLER);

    @Override
    public Channel build(Channel channel) {
        if (StringUtil.isEmpty(Config.Agent.AUTHENTICATION)) {
            return channel;
        }

        return ClientInterceptors.intercept(channel, new ClientInterceptor() {
            @Override
            public <REQ, RESP> ClientCall<REQ, RESP> interceptCall(MethodDescriptor<REQ, RESP> method,
                CallOptions options, Channel channel) {
                return new ForwardingClientCall.SimpleForwardingClientCall<REQ, RESP>(channel.newCall(method, options)) {
                    @Override
                    public void start(Listener<RESP> responseListener, Metadata headers) {
                        // 向OAP发送数据的时候,请求头中带上的Authentication(token)
                        headers.put(AUTH_HEAD_HEADER_NAME, Config.Agent.AUTHENTICATION);

                        super.start(responseListener, headers);
                    }
                };
            }
        });
    }
}
```

GRPCChannelManager
```
/**
 * Agent到OAP的网络连接
 */
@DefaultImplementor
public class GRPCChannelManager implements BootService, Runnable {
  
    private volatile ScheduledFuture<?> connectCheckFuture; // 网络连接状态定时检查任务调度器
    private volatile List<String> grpcServers; // OAP地址列表

    @Override
    public void boot() {
        // 检查OAP地址
        if (Config.Collector.BACKEND_SERVICE.trim().length() == 0) {
            LOGGER.error("Collector server addresses are not set.");
            LOGGER.error("Agent will not uplink any data.");
            return;
        }
        grpcServers = Arrays.asList(Config.Collector.BACKEND_SERVICE.split(","));
        connectCheckFuture = Executors.newSingleThreadScheduledExecutor(
            new DefaultNamedThreadFactory("GRPCChannelManager")
        ).scheduleAtFixedRate(
          	// Runnable传入this
            new RunnableWithExceptionProtection(
                this,
                t -> LOGGER.error("unexpected exception.", t)
            ), 0, Config.Collector.GRPC_CHANNEL_CHECK_INTERVAL, TimeUnit.SECONDS
        );
    }
```
boot阶段时，先检查OAP地址，解析OAP地址列表，创建网络连接状态定时检查任务调度器

boot()方法中用到了DefaultNamedThreadFactory，代码如下：
```
public class DefaultNamedThreadFactory implements ThreadFactory {
    // 这是第几个线程工厂
    private static final AtomicInteger BOOT_SERVICE_SEQ = new AtomicInteger(0);
    // 这是当前线程工厂创建的第几条线程
    private final AtomicInteger threadSeq = new AtomicInteger(0);
    private final String namePrefix;

    public DefaultNamedThreadFactory(String name) {
        namePrefix = "SkywalkingAgent-" + BOOT_SERVICE_SEQ.incrementAndGet() + "-" + name + "-";
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, namePrefix + threadSeq.getAndIncrement());
        t.setDaemon(true);
        return t;
    }
}
```

boot()方法中初始化网络连接状态定时检查任务调度器connectCheckFuture，Runnable传入的是this，实际上执行的是GRPCChannelManager的run()方法，代码如下：
```
@DefaultImplementor
public class GRPCChannelManager implements BootService, Runnable {

    private volatile GRPCChannel managedChannel = null; // 网络连接
    private volatile boolean reconnect = true; // 当前网络连接是否需要重连
    private final Random random = new Random();
    private final List<GRPCChannelListener> listeners = Collections.synchronizedList(new LinkedList<>());
    private volatile List<String> grpcServers; // OAP地址列表
    private volatile int selectedIdx = -1; // 上次选择的OAP地址下标索引
    private volatile int reconnectCount = 0; // 网络重连次数
  
    @Override
    public void run() {
        LOGGER.debug("Selected collector grpc service running, reconnect:{}.", reconnect);
        // 如果需要重连和刷新dns
        if (IS_RESOLVE_DNS_PERIODICALLY && reconnect) {
            // 拿到配置中的第一个后端服务地址
            String backendService = Config.Collector.BACKEND_SERVICE.split(",")[0];
            try {
                // 拆分成域名和端口的形式
                String[] domainAndPort = backendService.split(":");

                List<String> newGrpcServers = Arrays
                        // 找到域名对应的所有IP
                        .stream(InetAddress.getAllByName(domainAndPort[0]))
                        .map(InetAddress::getHostAddress)
                        .map(ip -> String.format("%s:%s", ip, domainAndPort[1]))
                        .collect(Collectors.toList());

                grpcServers = newGrpcServers;
            } catch (Throwable t) {
                LOGGER.error(t, "Failed to resolve {} of backend service.", backendService);
            }
        }

        // 如果需要重连网络连接
        if (reconnect) {
            if (grpcServers.size() > 0) {
                String server = "";
                try {
                    int index = Math.abs(random.nextInt()) % grpcServers.size();
                    // 如果本次选择的不是上次选择的OAP地址
                    if (index != selectedIdx) {
                        selectedIdx = index;

                        server = grpcServers.get(index);
                        String[] ipAndPort = server.split(":");

                        // 关闭上次出问题的网络连接
                        if (managedChannel != null) {
                            managedChannel.shutdownNow();
                        }

                        // 重新构建网络连接
                        managedChannel = GRPCChannel.newBuilder(ipAndPort[0], Integer.parseInt(ipAndPort[1]))
                                                    .addManagedChannelBuilder(new StandardChannelBuilder())
                                                    .addManagedChannelBuilder(new TLSChannelBuilder())
                                                    .addChannelDecorator(new AgentIDDecorator())
                                                    .addChannelDecorator(new AuthenticationDecorator())
                                                    .build();
                        // 通知所有使用该网络连接的其他BootService当前网络状态已经连上了
                        notify(GRPCChannelStatus.CONNECTED);
                        reconnectCount = 0;
                        reconnect = false;
                    }
                    // 重连次数小于设置的grpc强制重连周期,尝试重连;否则尝试创建新的连接
                    // 判断managedChannel是否连接上
                    else if (managedChannel.isConnected(++reconnectCount > Config.Agent.FORCE_RECONNECTION_PERIOD)) {
                        // grpc自动重连到同一个Server
                        // Reconnect to the same server is automatically done by GRPC,
                        // therefore we are responsible to check the connectivity and
                        // set the state and notify listeners
                        reconnectCount = 0;
                        notify(GRPCChannelStatus.CONNECTED);
                        reconnect = false;
                    }

                    return;
                } catch (Throwable t) {
                    LOGGER.error(t, "Create channel to {} fail.", server);
                }
            }

            LOGGER.debug(
                "Selected collector grpc service is not available. Wait {} seconds to retry",
                Config.Collector.GRPC_CHANNEL_CHECK_INTERVAL
            );
        }
    }
```
run()方法处理逻辑如下：

如果需要重连和刷新dns，刷新当前配置的域名对应的ip地址，进行格式化，组成OAP地址列表

如果需要重连网络连接：

1）如果本次选择的不是上次选择的OAP地址，关闭上次出问题的网络连接，重新构建网络连接，并通知所有使用该网络连接的其他BootService当前网络状态已经连上了

2）否则，尝试重连（GRPC自动重连到同一个Server），判断是否重连成功，如果重连成功，通知所有使用该网络连接的其他BootService当前网络状态已经连上了

run()方法中调用notify()通知所有的监听器当前网络状态是CONNECTED

GRPCChannelStatus中定义了网络连接的状态：
```
public enum GRPCChannelStatus {
    CONNECTED, DISCONNECT
}
```

notify()方法代码如下：
```
@DefaultImplementor
public class GRPCChannelManager implements BootService, Runnable {

		private void notify(GRPCChannelStatus status) {
        for (GRPCChannelListener listener : listeners) {
            try {
                listener.statusChanged(status);
            } catch (Throwable t) {
                LOGGER.error(t, "Fail to notify {} about channel connected.", listener.getClass().getName());
            }
        }
    }
```
GRPCChannelListener中定义了statusChanged()方法：
```
public interface GRPCChannelListener {
    void statusChanged(GRPCChannelStatus status);
}
```

GRPCChannelManager中reportError()方法代码如下：
```
@DefaultImplementor
public class GRPCChannelManager implements BootService, Runnable {
  
    /**
     * 如果使用网络连接的时候发生异常,设置重连标志,通知监听器
     * If the given exception is triggered by network problem, connect in background.
     */
    public void reportError(Throwable throwable) {
      	// 判断是否是网络异常
        if (isNetworkError(throwable)) {
          	// 设置重连标志
            reconnect = true;
            notify(GRPCChannelStatus.DISCONNECT);
        }
    }
```
该方法用于上报异常，如果是网络异常时，设置重连标志，connectCheckFuture下次执行时进行重连操作（GRPCChannelManager中run()方法中实现）














