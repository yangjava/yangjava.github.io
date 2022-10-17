14、服务-GRPCChannelManager
org.apache.skywalking.apm.agent.core.boot.BootService是SkyWalking Agent中所有服务的顶层接口，这个接口定义了一个服务的生命周期，分为prepare、boot、onComplete、shutdown四个阶段

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
GRPCChannelManager是负责Agent到OAP的网络连接，通过GRPCChannel将Agent采集的链路信息和JVM的指标发送到OAP做分析、展示

1）、GRPCChannel
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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
GRPCChannel的构造函数中，先通过ChannelBuilder构建originChannel，然后使用装饰器ChannelDecorator装饰originChannel从而得到channelWithDecorators

ChannelBuilder接口代码如下：

public interface ChannelBuilder<B extends ManagedChannelBuilder> {
    B build(B managedChannelBuilder) throws Exception;
}
1
2
3

ChannelBuilder有两个实现StandardChannelBuilder和TLSChannelBuilder

StandardChannelBuilder设置了接收数据的最大容量，明文传输

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
1
2
3
4
5
6
7
8
9
10
11
TLSChannelBuilder设置加密传输

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
ChannelDecorator接口代码如下：

public interface ChannelDecorator {
    Channel build(Channel channel);
}
1
2
3

ChannelDecorator有两个实现AgentIDDecorator和AuthenticationDecorator

AgentIDDecorator装饰的Channel向OAP发送数据的时候，请求头中会带上Agent版本号

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
AuthenticationDecorator装饰的Channel向OAP发送数据的时候，请求头中带上的token

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
2）、GRPCChannelManager
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
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 26
 27
 28
 boot阶段时，先检查OAP地址，解析OAP地址列表，创建网络连接状态定时检查任务调度器

boot()方法中用到了DefaultNamedThreadFactory，代码如下：

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
boot()方法中初始化网络连接状态定时检查任务调度器connectCheckFuture，Runnable传入的是this，实际上执行的是GRPCChannelManager的run()方法，代码如下：

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
run()方法处理逻辑如下：

如果需要重连和刷新dns，刷新当前配置的域名对应的ip地址，进行格式化，组成OAP地址列表

如果需要重连网络连接：

1）如果本次选择的不是上次选择的OAP地址，关闭上次出问题的网络连接，重新构建网络连接，并通知所有使用该网络连接的其他BootService当前网络状态已经连上了

2）否则，尝试重连（GRPC自动重连到同一个Server），判断是否重连成功，如果重连成功，通知所有使用该网络连接的其他BootService当前网络状态已经连上了

run()方法中调用notify()通知所有的监听器当前网络状态是CONNECTED

GRPCChannelStatus中定义了网络连接的状态：

public enum GRPCChannelStatus {
    CONNECTED, DISCONNECT
}
1
2
3
notify()方法代码如下：

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
1
2
3
4
5
6
7
8
9
10
11
12
GRPCChannelListener中定义了statusChanged()方法：

public interface GRPCChannelListener {
    void statusChanged(GRPCChannelStatus status);
}
1
2
3
GRPCChannelManager中reportError()方法代码如下：

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
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
该方法用于上报异常，如果是网络异常时，设置重连标志，connectCheckFuture下次执行时进行重连操作（GRPCChannelManager中run()方法中实现）

小结：


15、服务-ServiceManagementClient
ServiceManagementClient实现了GRPCChannelListener接口，statusChanged()方法代码如下：

/**
 * 1.将当前Agent Client的基本信息汇报给OAP
 * 2.和OAP保持心跳
 */
 @DefaultImplementor
 public class ServiceManagementClient implements BootService, Runnable, GRPCChannelListener {

    private volatile GRPCChannelStatus status = GRPCChannelStatus.DISCONNECT; // 当前网络连接状态  
    private volatile ManagementServiceGrpc.ManagementServiceBlockingStub managementServiceBlockingStub; // 网络服务

    @Override
    public void statusChanged(GRPCChannelStatus status) {
        // 网络是否是已连接状态
        if (GRPCChannelStatus.CONNECTED.equals(status)) {
            // 找到GRPCChannelManager服务,拿到网络连接
            Channel channel = ServiceManager.INSTANCE.findService(GRPCChannelManager.class).getChannel();
            // grpc的stub可以理解为在protobuf中定义的XxxService
            managementServiceBlockingStub = ManagementServiceGrpc.newBlockingStub(channel);
        } else {
            managementServiceBlockingStub = null;
        }
        this.status = status;
    }
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 prepare阶段代码如下：

@DefaultImplementor
public class ServiceManagementClient implements BootService, Runnable, GRPCChannelListener {

    private static List<KeyStringValuePair> SERVICE_INSTANCE_PROPERTIES; // Agent Client的信息
    
    @Override
    public void prepare() {
        // 向GRPCChannelManager注册自己为监听器
        ServiceManager.INSTANCE.findService(GRPCChannelManager.class).addChannelListener(this);
    
        SERVICE_INSTANCE_PROPERTIES = new ArrayList<>();
        // 把配置文件中的Agent Client信息放入集合,等待发送
        for (String key : Config.Agent.INSTANCE_PROPERTIES.keySet()) {
            SERVICE_INSTANCE_PROPERTIES.add(KeyStringValuePair.newBuilder()
                                                              .setKey(key)
                                                              .setValue(Config.Agent.INSTANCE_PROPERTIES.get(key))
                                                              .build());
        }
    }  
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
prepare阶段时，先向GRPCChannelManager注册自己为监听器，然后把配置文件中的Agent Client信息放入集合，等待发送

boot阶段代码如下：

@DefaultImplementor
public class ServiceManagementClient implements BootService, Runnable, GRPCChannelListener {

    private volatile ScheduledFuture<?> heartbeatFuture; // 心跳定时任务
    
    	@Override
    public void boot() {
        heartbeatFuture = Executors.newSingleThreadScheduledExecutor(
            new DefaultNamedThreadFactory("ServiceManagementClient")
        ).scheduleAtFixedRate(
            new RunnableWithExceptionProtection(
                this,
                t -> LOGGER.error("unexpected exception.", t)
            ), 0, Config.Collector.HEARTBEAT_PERIOD,
            TimeUnit.SECONDS
        );
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
boot()方法中初始化心跳定时任务heartbeatFuture，Runnable传入的是this，实际上执行的是ServiceManagementClient的run()方法，代码如下：

@DefaultImplementor
public class ServiceManagementClient implements BootService, Runnable, GRPCChannelListener {

    private volatile ManagementServiceGrpc.ManagementServiceBlockingStub managementServiceBlockingStub; // 网络服务
    private volatile AtomicInteger sendPropertiesCounter = new AtomicInteger(0); // Agent Client信息发送次数计数器
    
    @Override
    public void run() {
        LOGGER.debug("ServiceManagementClient running, status:{}.", status);
    
        // 网络是否是已连接状态
        if (GRPCChannelStatus.CONNECTED.equals(status)) {
            try {
                if (managementServiceBlockingStub != null) {
                    // 心跳周期 = 30s, 信息汇报频率因子 = 10 => 每5分钟向OAP汇报一次Agent Client Properties
                    // Round1 counter=0 0%10=0
                    // Round2 counter=1 1%10=1
                    if (Math.abs(sendPropertiesCounter.getAndAdd(1)) % Config.Collector.PROPERTIES_REPORT_PERIOD_FACTOR == 0) {
    
                        managementServiceBlockingStub
                            // 设置请求超时时间,默认30秒    
                            .withDeadlineAfter(GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS)
                            .reportInstanceProperties(InstanceProperties.newBuilder()
                                                                        .setService(Config.Agent.SERVICE_NAME)
                                                                        .setServiceInstance(Config.Agent.INSTANCE_NAME)
                                                                        // 当前操作系统的信息
                                                                        .addAllProperties(OSUtil.buildOSInfo(
                                                                            Config.OsInfo.IPV4_LIST_SIZE))
                                                                        .addAllProperties(SERVICE_INSTANCE_PROPERTIES)
                                                                        // JVM信息
                                                                        .addAllProperties(LoadedLibraryCollector.buildJVMInfo())
                                                                        .build());
                    } else {
                        // 服务端给到的响应交给CommandService去处理
                        final Commands commands = managementServiceBlockingStub.withDeadlineAfter(
                            GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS
                        ).keepAlive(InstancePingPkg.newBuilder()
                                                   .setService(Config.Agent.SERVICE_NAME)
                                                   .setServiceInstance(Config.Agent.INSTANCE_NAME)
                                                   .build());
    
                        ServiceManager.INSTANCE.findService(CommandService.class).receiveCommand(commands);
                    }
                }
            } catch (Throwable t) {
                LOGGER.error(t, "ServiceManagementClient execute fail.");
                ServiceManager.INSTANCE.findService(GRPCChannelManager.class).reportError(t);
            }
        }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
run()方法处理逻辑如下：

心跳周期为30s，信息汇报频率因子为10，所以每5分钟向OAP汇报一次Agent Client Properties

判断本次是否是Agent信息上报

1）如果本次是信息上报，上报Agent信息，包括：服务名、实例名、Agent Client的信息、当前操作系统的信息、JVM信息

2）如果本次不是信息上报，请求服务端，将服务端给到的响应交给CommandService去处理

如果配置文件中未指定实例名，ServiceInstanceGenerator会生成一个实例名

@Getter
public class ServiceInstanceGenerator implements BootService {
    @Override
    public void prepare() throws Throwable {
        if (!isEmpty(Config.Agent.INSTANCE_NAME)) {
            return;
        }
        // 生成Agent实例名
        Config.Agent.INSTANCE_NAME = UUID.randomUUID().toString().replaceAll("-", "") + "@" + OSUtil.getIPV4();
    }
1
2
3
4
5
6
7
8
9
10
小结：


16、服务-CommandService
1）、CommandService
/**
 * Command Scheduler命令的调度器
 * 收集OAP返回的Command,然后分发给不同的处理器去处理
 */
 @DefaultImplementor
 public class CommandService implements BootService, Runnable {
  
    private ExecutorService executorService = Executors.newSingleThreadExecutor(
        new DefaultNamedThreadFactory("CommandService")
    
    @Override
    public void boot() throws Throwable {
        executorService.submit(
            new RunnableWithExceptionProtection(this, t -> LOGGER.error(t, "CommandService failed to execute commands"))
        );
    }      
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 boot()方法中使用executorService提交任务，Runnable传入的是this，实际上执行的是CommandService的run()方法，代码如下：

@DefaultImplementor
public class CommandService implements BootService, Runnable {

    private volatile boolean isRunning = true; // 命令的处理流程是否在运行
    private LinkedBlockingQueue<BaseCommand> commands = new LinkedBlockingQueue<>(64); // 待处理命令列表
    private CommandSerialNumberCache serialNumberCache = new CommandSerialNumberCache(); // 命令的序列号缓存
    
    /**
     * 不断从命令队列(任务队列)里取出任务,交给执行器去执行
     */
    @Override
    public void run() {
        final CommandExecutorService commandExecutorService = ServiceManager.INSTANCE.findService(CommandExecutorService.class);
    
        while (isRunning) {
            try {
                BaseCommand command = commands.take();
                // 同一个命令不要重复执行
                if (isCommandExecuted(command)) {
                    continue;
                }
    
                // 执行command
                commandExecutorService.execute(command);
                serialNumberCache.add(command.getSerialNumber());
            } catch (InterruptedException e) {
                LOGGER.error(e, "Failed to take commands.");
            } catch (CommandExecutionException e) {
                LOGGER.error(e, "Failed to execute command[{}].", e.command().getCommand());
            } catch (Throwable e) {
                LOGGER.error(e, "There is unexpected exception");
            }
        }
    }
    
    private boolean isCommandExecuted(BaseCommand command) {
        return serialNumberCache.contain(command.getSerialNumber());
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
run()方法中不断从命令队列里取出任务，交给执行器去执行，使用CommandSerialNumberCache作为命令的序列号缓存，执行完命令后在序列号缓存中添加该命令的序列号，保证同一个命令不会重复执行

CommandSerialNumberCache代码如下：

/**
 * 命令的序列号缓存,序列号被放到一个队列里面,并且做了容量控制
 */
 public class CommandSerialNumberCache {
    private static final int DEFAULT_MAX_CAPACITY = 64;
    private final Deque<String> queue;
    private final int maxCapacity;

    public CommandSerialNumberCache() {
        this(DEFAULT_MAX_CAPACITY);
    }

    public CommandSerialNumberCache(int maxCapacity) {
        queue = new LinkedBlockingDeque<String>(maxCapacity);
        this.maxCapacity = maxCapacity;
    }

    public void add(String number) {
        if (queue.size() >= maxCapacity) {
            queue.pollFirst();
        }

        queue.add(number);
    }

    public boolean contain(String command) {
        return queue.contains(command);
    }
 }
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 26
 27
 28
 29
 ServiceManagementClient的run()方法中拿到服务端给的命令，会调用CommandService的receiveCommand()方法去处理

                        // 服务端给到的响应交给CommandService去处理
                        final Commands commands = managementServiceBlockingStub.withDeadlineAfter(
                            GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS
                        ).keepAlive(InstancePingPkg.newBuilder()
                                                   .setService(Config.Agent.SERVICE_NAME)
                                                   .setServiceInstance(Config.Agent.INSTANCE_NAME)
                                                   .build());
     
                        ServiceManager.INSTANCE.findService(CommandService.class).receiveCommand(commands);
 1
 2
 3
 4
 5
 6
 7
 8
 9
 CommandService的receiveCommand()方法代码如下：

@DefaultImplementor
public class CommandService implements BootService, Runnable {

    private LinkedBlockingQueue<BaseCommand> commands = new LinkedBlockingQueue<>(64); // 待处理命令列表
    
    public void receiveCommand(Commands commands) {
        for (Command command : commands.getCommandsList()) {
            try {
                // 反序列化
                BaseCommand baseCommand = CommandDeserializer.deserialize(command);
    
                if (isCommandExecuted(baseCommand)) {
                    LOGGER.warn("Command[{}] is executed, ignored", baseCommand.getCommand());
                    continue;
                }
    
                // 添加到待处理命令列表
                boolean success = this.commands.offer(baseCommand);
    
                if (!success && LOGGER.isWarnEnable()) {
                    LOGGER.warn(
                        "Command[{}, {}] cannot add to command list. because the command list is full.",
                        baseCommand.getCommand(), baseCommand.getSerialNumber()
                    );
                }
            } catch (UnsupportedCommandException e) {
                if (LOGGER.isWarnEnable()) {
                    LOGGER.warn("Received unsupported command[{}].", e.getCommand().getCommand());
                }
            }
        }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
CommandDeserializer的deserialize()方法进行反序列化，代码如下：

public class CommandDeserializer {

    public static BaseCommand deserialize(final Command command) {
        final String commandName = command.getCommand();
        if (ProfileTaskCommand.NAME.equals(commandName)) { // 性能追踪
            return ProfileTaskCommand.DESERIALIZER.deserialize(command);
        } else if (ConfigurationDiscoveryCommand.NAME.equals(commandName)) { // 配置更改
            return ConfigurationDiscoveryCommand.DESERIALIZER.deserialize(command);
        }
        throw new UnsupportedCommandException(command);
    }

}
1
2
3
4
5
6
7
8
9
10
11
12
13
OAP下发的命令包含两类：

ProfileTaskCommand：在SkyWalking UI性能剖析功能中，新建任务，会下发给Agent性能追踪任务
ConfigurationDiscoveryCommand：当前版本SkyWalking Agent支持运行时动态调整配置
ConfigurationDiscoveryCommand反序列化代码如下：

public class ConfigurationDiscoveryCommand extends BaseCommand implements Serializable, Deserializable<ConfigurationDiscoveryCommand> {

    public static final String NAME = "ConfigurationDiscoveryCommand";
    
    public static final String UUID_CONST_NAME = "UUID";
    public static final String SERIAL_NUMBER_CONST_NAME = "SerialNumber";
      
    /*
     * 如果配置没有变,那么OAP返回的UUID就是一样的
     * If config is unchanged, then could response the same uuid, and config is not required.
     */
    private String uuid;
    /*
     * The configuration of service.
     */
    private List<KeyStringValuePair> config;
    
    public ConfigurationDiscoveryCommand(String serialNumber,
                                         String uuid,
                                         List<KeyStringValuePair> config) {
        super(NAME, serialNumber);
        this.uuid = uuid;
        this.config = config;
    }
    
    @Override
    public ConfigurationDiscoveryCommand deserialize(Command command) {
        String serialNumber = null;
        String uuid = null;
        List<KeyStringValuePair> config = new ArrayList<>();
    
        for (final KeyStringValuePair pair : command.getArgsList()) {
            if (SERIAL_NUMBER_CONST_NAME.equals(pair.getKey())) {
                serialNumber = pair.getValue();
            } else if (UUID_CONST_NAME.equals(pair.getKey())) {
                uuid = pair.getValue();
            } else {
                config.add(pair);
            }
        }
        return new ConfigurationDiscoveryCommand(serialNumber, uuid, config);
    }  
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
2）、执行command
@DefaultImplementor
public class CommandService implements BootService, Runnable {

    @Override
    public void run() {
        final CommandExecutorService commandExecutorService = ServiceManager.INSTANCE.findService(CommandExecutorService.class);
    
        while (isRunning) {
            try {
                BaseCommand command = commands.take();
                // 同一个命令不要重复执行
                if (isCommandExecuted(command)) {
                    continue;
                }
    
                // 执行command
                commandExecutorService.execute(command);
                serialNumberCache.add(command.getSerialNumber());
            } catch (InterruptedException e) {
                LOGGER.error(e, "Failed to take commands.");
            } catch (CommandExecutionException e) {
                LOGGER.error(e, "Failed to execute command[{}].", e.command().getCommand());
            } catch (Throwable e) {
                LOGGER.error(e, "There is unexpected exception");
            }
        }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
CommandService的run()方法中调用CommandExecutorService的execute()方法执行command，CommandExecutorService代码如下：

@DefaultImplementor
public class CommandExecutorService implements BootService, CommandExecutor {
    /**
     * key: 命令的名字 value:对应的命令执行器
     */
    private Map<String, CommandExecutor> commandExecutorMap;

    @Override
    public void prepare() throws Throwable {
        commandExecutorMap = new HashMap<String, CommandExecutor>();
    
        // 性能追踪命令执行器
        // Profile task executor
        commandExecutorMap.put(ProfileTaskCommand.NAME, new ProfileTaskCommandExecutor());
    
        // 配置变更命令执行器
        //Get ConfigurationDiscoveryCommand executor.
        commandExecutorMap.put(ConfigurationDiscoveryCommand.NAME, new ConfigurationDiscoveryCommandExecutor());
    }
    
    @Override
    public void boot() throws Throwable {
    
    }
    
    @Override
    public void onComplete() throws Throwable {
    
    }
    
    @Override
    public void shutdown() throws Throwable {
    
    }
    
    @Override
    public void execute(final BaseCommand command) throws CommandExecutionException {
        executorForCommand(command).execute(command);
    }
    
    private CommandExecutor executorForCommand(final BaseCommand command) {
      	// 根据command类型找到对应的命令执行器
        final CommandExecutor executor = commandExecutorMap.get(command.getCommand());
        if (executor != null) {
            return executor;
        }
        return NoopCommandExecutor.INSTANCE;
    }
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
execute()方法就是根据command类型找到对应的命令执行器，再调用对应的execute()实现

接下来来看下配置变更命令执行器的实现，ConfigurationDiscoveryCommandExecutor代码如下：

public class ConfigurationDiscoveryCommandExecutor implements CommandExecutor {

    private static final ILog LOGGER = LogManager.getLogger(ConfigurationDiscoveryCommandExecutor.class);
    
    @Override
    public void execute(BaseCommand command) throws CommandExecutionException {
        try {
            ConfigurationDiscoveryCommand agentDynamicConfigurationCommand = (ConfigurationDiscoveryCommand) command;
    
            ServiceManager.INSTANCE.findService(ConfigurationDiscoveryService.class)
                                   .handleConfigurationDiscoveryCommand(agentDynamicConfigurationCommand);
        } catch (Exception e) {
            LOGGER.error(e, "Handle ConfigurationDiscoveryCommand error, command:{}", command.toString());
        }
    }
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
execute()方法中调用ConfigurationDiscoveryService的handleConfigurationDiscoveryCommand()方法真正来处理配置变更的命令

3）、ConfigurationDiscoveryService
@DefaultImplementor
public class ConfigurationDiscoveryService implements BootService, GRPCChannelListener {

    private final Register register = new Register();
      
    /**
     * Register dynamic configuration watcher.
     *
     * @param watcher dynamic configuration watcher
     */
    public void registerAgentConfigChangeWatcher(AgentConfigChangeWatcher watcher) {
        WatcherHolder holder = new WatcherHolder(watcher);
        if (register.containsKey(holder.getKey())) {
            throw new IllegalStateException("Duplicate register, watcher=" + watcher);
        }
        register.put(holder.getKey(), holder);
    }
    
    /**
     * Local dynamic configuration center.
     */
    public static class Register {
        private final Map<String, WatcherHolder> register = new HashMap<>();
    
        private boolean containsKey(String key) {
            return register.containsKey(key);
        }
    
        private void put(String key, WatcherHolder holder) {
            register.put(key, holder);
        }
    
        public WatcherHolder get(String name) {
            return register.get(name);
        }
    
        public Set<String> keys() {
            return register.keySet();
        }
    
        @Override
        public String toString() {
            ArrayList<String> registerTableDescription = new ArrayList<>(register.size());
            register.forEach((key, holder) -> {
                AgentConfigChangeWatcher watcher = holder.getWatcher();
                registerTableDescription.add(new StringBuilder().append("key:")
                                                                .append(key)
                                                                .append("value(current):")
                                                                .append(watcher.value()).toString());
            });
            return registerTableDescription.stream().collect(Collectors.joining(",", "[", "]"));
        }
    }
    
    @Getter
    private static class WatcherHolder {
        private final AgentConfigChangeWatcher watcher;
        private final String key;
    
        public WatcherHolder(AgentConfigChangeWatcher watcher) {
            this.watcher = watcher;
            this.key = watcher.getPropertyKey();
        }
    }  
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
ConfigurationDiscoveryService中有一个内部类Register，实际上是一个Map，value类型是内部类WatcherHolder，WatcherHolder中包含了AgentConfigChangeWatcher属性，通过registerAgentConfigChangeWatcher()方法注册watcher

AgentConfigChangeWatcher用于监听SkyWalking Agent的某项配置的值的变化，代码如下：

/**
 * 监听agent的某项配置的值的变化
 */
 @Getter
 public abstract class AgentConfigChangeWatcher {
    // Config key, should match KEY in the Table of Agent Configuration Properties.
    // 这个key来源于agent.config,也就是说只有agent配置文件中合法的key才能在这里被使用
    private final String propertyKey;

    public AgentConfigChangeWatcher(String propertyKey) {
        this.propertyKey = propertyKey;
    }

    /**
     * 配置变更通知对应的watcher
     * Notify the watcher, the new value received.
     *
     * @param value of new.
     */
     public abstract void notify(ConfigChangeEvent value);

    /**
     * @return current value of current config.
     */
     public abstract String value();
  
    /**
     * 配置变更事件
     */
     @Getter
     @RequiredArgsConstructor
     public static class ConfigChangeEvent {
        // 新的配置值
        private final String newValue;
        // 事件类型
        private final EventType eventType;
     }

    public enum EventType {
        ADD, MODIFY, DELETE
    }  
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 26
 27
 28
 29
 30
 31
 32
 33
 34
 35
 36
 37
 38
 39
 40
 41


AgentConfigChangeWatcher有以上四种实现，后面使用到对应的ChangeWatcher时会讲到

上述的整个嵌套结构如下图：


再回来看ConfigurationDiscoveryService的相关方法：

@DefaultImplementor
public class ConfigurationDiscoveryService implements BootService, GRPCChannelListener {

    private volatile ScheduledFuture<?> getDynamicConfigurationFuture; // 获取动态配置器
    
    @Override
    public void boot() throws Throwable {
        getDynamicConfigurationFuture = Executors.newSingleThreadScheduledExecutor(
            new DefaultNamedThreadFactory("ConfigurationDiscoveryService")
        ).scheduleAtFixedRate(
            new RunnableWithExceptionProtection(
                this::getAgentDynamicConfig,
                t -> LOGGER.error("Sync config from OAP error.", t)
            ),
            Config.Collector.GET_AGENT_DYNAMIC_CONFIG_INTERVAL,
            Config.Collector.GET_AGENT_DYNAMIC_CONFIG_INTERVAL,
            TimeUnit.SECONDS
        );
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
boot()方法中初始化获取动态配置器getDynamicConfigurationFuture，实际上执行的是ConfigurationDiscoveryService的getAgentDynamicConfig()方法，代码如下：

@DefaultImplementor
public class ConfigurationDiscoveryService implements BootService, GRPCChannelListener {

    /**
     * UUID of the last return value.
     */
    private String uuid;
    private final Register register = new Register();
    
    private volatile int lastRegisterWatcherSize; // 上一次计算的watcher的数量
      
    private volatile ConfigurationDiscoveryServiceGrpc.ConfigurationDiscoveryServiceBlockingStub configurationDiscoveryServiceBlockingStub;  
    
    	/**
     * 通过grpc获取Agent动态配置
     * get agent dynamic config through gRPC.
     */
    private void getAgentDynamicConfig() {
        LOGGER.debug("ConfigurationDiscoveryService running, status:{}.", status);
    
        if (GRPCChannelStatus.CONNECTED.equals(status)) {
            try {
                ConfigurationSyncRequest.Builder builder = ConfigurationSyncRequest.newBuilder();
                builder.setService(Config.Agent.SERVICE_NAME);
    
                // 有些插件会延迟注册watcher
                // Some plugin will register watcher later.
                final int size = register.keys().size();
                // 如果本次计算的watcher的数量和上一次不相同
                if (lastRegisterWatcherSize != size) {
                    // 当watcher的数量发生了变动,代表有新的配置项需要监听
                    // 重置uuid,避免同样的uuid导致配置信息没有被更新
                    // reset uuid, avoid the same uuid causing the configuration not to be updated.
                    uuid = null;
                    lastRegisterWatcherSize = size;
                }
    
                if (null != uuid) {
                    builder.setUuid(uuid);
                }
    
                if (configurationDiscoveryServiceBlockingStub != null) {
                    // 拉取配置 反序列化后为ConfigurationDiscoveryCommand类型
                    final Commands commands = configurationDiscoveryServiceBlockingStub.withDeadlineAfter(
                        GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS
                    ).fetchConfigurations(builder.build());
                    ServiceManager.INSTANCE.findService(CommandService.class).receiveCommand(commands);
                }
            } catch (Throwable t) {
                LOGGER.error(t, "ConfigurationDiscoveryService execute fail.");
                ServiceManager.INSTANCE.findService(GRPCChannelManager.class).reportError(t);
            }
        }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
getAgentDynamicConfig()方法处理逻辑如下：

因为有些插件会延迟注册watcher，当watcher的数量发生了变动，代表有新的配置项需要监听，需要重置uuid，避免同样的uuid导致配置信息没有被更新
调用ConfigurationDiscoveryServiceBlockingStub的fetchConfigurations()方法拉取配置，调用CommandService的receiveCommand()将命令添加到待处理命令列表，该方法拿到的command序列化后为ConfigurationDiscoveryCommand类型，走执行command中讲的流程，最终由ConfigurationDiscoveryService的handleConfigurationDiscoveryCommand()来处理配置变更的命令
ConfigurationDiscoveryService的handleConfigurationDiscoveryCommand()方法代码如下：

@DefaultImplementor
public class ConfigurationDiscoveryService implements BootService, GRPCChannelListener {

    /**
     * Process ConfigurationDiscoveryCommand and notify each configuration watcher.
     *
     * @param configurationDiscoveryCommand Describe dynamic configuration information
     */
    public void handleConfigurationDiscoveryCommand(ConfigurationDiscoveryCommand configurationDiscoveryCommand) {
        final String responseUuid = configurationDiscoveryCommand.getUuid();
    
        // uuid相同说明配置没有变化
        if (responseUuid != null && Objects.equals(this.uuid, responseUuid)) {
            return;
        }
    
        // configurationDiscoveryCommand可能返回了10个配置变更
        // 但是register里面只有5个配置项的监听器
        // 这里就要过滤出有监听器的配置项
        List<KeyStringValuePair> config = readConfig(configurationDiscoveryCommand);
    
        config.forEach(property -> {
            String propertyKey = property.getKey();
            WatcherHolder holder = register.get(propertyKey);
            if (holder != null) {
                AgentConfigChangeWatcher watcher = holder.getWatcher();
                String newPropertyValue = property.getValue();
                if (StringUtil.isBlank(newPropertyValue)) {
                    if (watcher.value() != null) {
                        // 通知watcher,删除该配置
                        // Notify watcher, the new value is null with delete event type.
                        watcher.notify(
                            new AgentConfigChangeWatcher.ConfigChangeEvent(
                                null, AgentConfigChangeWatcher.EventType.DELETE
                            ));
                    } else {
                        // Don't need to notify, stay in null.
                    }
                } else {
                    if (!newPropertyValue.equals(watcher.value())) {
                        // 通知watcher,配置变更
                        watcher.notify(new AgentConfigChangeWatcher.ConfigChangeEvent(
                            newPropertyValue, AgentConfigChangeWatcher.EventType.MODIFY
                        ));
                    } else {
                        // Don't need to notify, stay in the same config value.
                    }
                }
            } else {
                LOGGER.warn("Config {} from OAP, doesn't match any watcher, ignore.", propertyKey);
            }
        });
      	// 更新uuid
        this.uuid = responseUuid;
    
        LOGGER.trace("Current configurations after the sync, configurations:{}", register.toString());
    }  
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
handleConfigurationDiscoveryCommand()方法处理逻辑如下：

如果uuid相同，说明配置没有变化，直接返回
过滤出有监听器的配置项，判断如果配置变更或者被删除，通知watcher，最后更新uuid
小结：



17、服务-SamplingService
SamplingService是来控制是否要采样该链路。每条链路都是被追踪到的，但是考虑到序列化/反序列化的CPU消耗以及网络带宽，如果开启采样，SkyWalking Agent并不会把所有的链路都发送给OAP。默认采样是开启的，可以通过修改agent.config中的agent.sample_n_per_3_secs配置项控制每3秒最多采样多少条链路

/**
 * SamplingService是来控制是否要采样该链路
 * 每条链路都是被追踪到的,但是考虑到序列化/反序列化的CPU消耗以及网络带宽,如果开启采样,agent并不会把所有的链路都发送给OAP
 * The <code>SamplingService</code> take charge of how to sample the {@link TraceSegment}. Every {@link TraceSegment}s
 * have been traced, but, considering CPU cost of serialization/deserialization, and network bandwidth, the agent do NOT
 * send all of them to collector, if SAMPLING is on.
 * <p>
 * 默认采样是开启的
 * By default, SAMPLING is on, and  {@link Config.Agent#SAMPLE_N_PER_3_SECS }
 * SAMPLE_N_PER_3_SECS:每3秒最多采样多少条链路,如果配置的是负数或者0就表示采样机制关闭(即所有的链路都会被采集并发送)
 */
 @DefaultImplementor
 public class SamplingService implements BootService {

    private SamplingRateWatcher samplingRateWatcher;
  
    @Override
    public void boot() {
        samplingRateWatcher = new SamplingRateWatcher("agent.sample_n_per_3_secs", this);
        // 注册配置变更监听器
        ServiceManager.INSTANCE.findService(ConfigurationDiscoveryService.class)
                               .registerAgentConfigChangeWatcher(samplingRateWatcher);

        handleSamplingRateChanged();
    }  
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 boot()方法中初始化SamplingRateWatcher，并向ConfigurationDiscoveryService注册配置变更监听器

SamplingRateWatcher是AgentConfigChangeWatcher的子类，监听的是agent.sample_n_per_3_secs这个动态配置

public class SamplingRateWatcher extends AgentConfigChangeWatcher {
    private static final ILog LOGGER = LogManager.getLogger(SamplingRateWatcher.class);

    private final AtomicInteger samplingRate; // 每3秒能够采集的最大链路数
    private final SamplingService samplingService;
    
    public SamplingRateWatcher(final String propertyKey, SamplingService samplingService) {
        super(propertyKey);
        this.samplingRate = new AtomicInteger(getDefaultValue());
        this.samplingService = samplingService;
    }
    
    private void activeSetting(String config) {
        if (LOGGER.isDebugEnable()) {
            LOGGER.debug("Updating using new static config: {}", config);
        }
        try {
            this.samplingRate.set(Integer.parseInt(config));
    
            /*
             * 通知samplingService每3秒能够采集的最大链路数的配置变更
             * We need to notify samplingService the samplingRate changed.
             */
            samplingService.handleSamplingRateChanged();
        } catch (NumberFormatException ex) {
            LOGGER.error(ex, "Cannot load {} from: {}", getPropertyKey(), config);
        }
    }
    
    @Override
    public void notify(final ConfigChangeEvent value) {
        if (EventType.DELETE.equals(value.getEventType())) {
            // 如果删除动态配置,使用agent.config中的配置
            activeSetting(String.valueOf(getDefaultValue()));
        } else {
            activeSetting(value.getNewValue());
        }
    }
    
    private int getDefaultValue() {
        return Config.Agent.SAMPLE_N_PER_3_SECS;
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
如果该配置项发生变更，会调用SamplingRateWatcher的notify()方法，SamplingRateWatcher修改配置后，再调用SamplingService的handleSamplingRateChanged()方法，通知每3秒能够采集的最大链路数的配置变更

@DefaultImplementor
public class SamplingService implements BootService {
    private static final ILog LOGGER = LogManager.getLogger(SamplingService.class);

    private volatile boolean on = false;
    private volatile AtomicInteger samplingFactorHolder; // 用于累加3秒内已经采样的次数
    private volatile ScheduledFuture<?> scheduledFuture; // 每3秒重置一次samplingFactorHolder
    
    /**
     * Handle the samplingRate changed.
     */
    public void handleSamplingRateChanged() {
        if (samplingRateWatcher.getSamplingRate() > 0) {
            if (!on) {
                on = true;
                this.resetSamplingFactor();
                ScheduledExecutorService service = Executors.newSingleThreadScheduledExecutor(
                    new DefaultNamedThreadFactory("SamplingService"));
                scheduledFuture = service.scheduleAtFixedRate(new RunnableWithExceptionProtection(
                    this::resetSamplingFactor, t -> LOGGER.error("unexpected exception.", t)), 0, 3, TimeUnit.SECONDS);
                LOGGER.debug(
                    "Agent sampling mechanism started. Sample {} traces in 3 seconds.",
                    samplingRateWatcher.getSamplingRate()
                );
            }
        } else {
            if (on) {
                if (scheduledFuture != null) {
                    scheduledFuture.cancel(true);
                }
                on = false;
            }
        }
    }
      
    private void resetSamplingFactor() {
        samplingFactorHolder = new AtomicInteger(0);
    }  
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
handleSamplingRateChanged()方法中判断如果开启采样，初始化定时任务，每3秒重置一次samplingFactorHolder，samplingFactorHolder是用于累加3秒内已经采样的次数

每一条链路是否要采样取决于SamplingService的trySampling()方法的返回，如果返回值为true，则该链路会被采样，trySampling()方法代码如下：

@DefaultImplementor
public class SamplingService implements BootService {

    private volatile AtomicInteger samplingFactorHolder; // 用于累加3秒内已经采样的次数
      
    /**
     * 如果采样机制没有开启,即on=false,那么就表示每一条采集到的链路都会上报给OAP
     * @param operationName The first operation name of the new tracing context.
     * @return true, if sampling mechanism is on, and getDefault the sampling factor successfully.
     */
    public boolean trySampling(String operationName) {
        if (on) {
            int factor = samplingFactorHolder.get();
            // 3秒内已经采样的次数<每3秒能够采集的最大链路数
            if (factor < samplingRateWatcher.getSamplingRate()) {
                // 3秒内已经采样的次数+1,CAS修改成功则该链路会被采样
                return samplingFactorHolder.compareAndSet(factor, factor + 1);
            } else {
                return false;
            }
        }
        return true;
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
SamplingService的`forceSampled()方法强制进行采样，代码如下：

@DefaultImplementor
public class SamplingService implements BootService {

    /**
     * Increase the sampling factor by force, to avoid sampling too many traces. If many distributed traces require
     * sampled, the trace beginning at local, has less chance to be sampled.
     */
    public void forceSampled() {
        if (on) {
            samplingFactorHolder.incrementAndGet();
        }
    }
1
2
3
4
5
6
7
8
9
10
11
12
小结：


18、服务-JVMService
JVMService是一个定时器，来收集JVM的cpu、内存、内存池、gc、线程和类的信息，然后将收集到的信息通过GRPCChannelManager提供的网络连接发送给OAP

/**
 * JVMService是一个定时器,来收集jvm的cpu、内存、内存池、gc、线程和类的信息,
 * 然后将收集到的信息通过GRPCChannelManager提供的网络连接发送给OAP
 * The <code>JVMService</code> represents a timer, which collectors JVM cpu, memory, memorypool, gc, thread and class info,
 * and send the collected info to Collector through the channel provided by {@link GRPCChannelManager}
 */
 @DefaultImplementor
 public class JVMService implements BootService, Runnable {
    private static final ILog LOGGER = LogManager.getLogger(JVMService.class);
    private volatile ScheduledFuture<?> collectMetricFuture; // 收集JVM信息的定时任务
    private volatile ScheduledFuture<?> sendMetricFuture; // 发送JVM信息的定时任务
    private JVMMetricsSender sender; // JVM信息的发送工具
  
    @Override
    public void prepare() throws Throwable {
        sender = ServiceManager.INSTANCE.findService(JVMMetricsSender.class);
    }

    @Override
    public void boot() throws Throwable {
        collectMetricFuture = Executors.newSingleThreadScheduledExecutor(
            new DefaultNamedThreadFactory("JVMService-produce"))
                                       .scheduleAtFixedRate(new RunnableWithExceptionProtection(
                                           this,
                                           new RunnableWithExceptionProtection.CallbackWhenException() {
                                               @Override
                                               public void handle(Throwable t) {
                                                   LOGGER.error("JVMService produces metrics failure.", t);
                                               }
                                           }
                                       ), 0, 1, TimeUnit.SECONDS);
        sendMetricFuture = Executors.newSingleThreadScheduledExecutor(
            new DefaultNamedThreadFactory("JVMService-consume"))
                                    .scheduleAtFixedRate(new RunnableWithExceptionProtection(
                                        sender,
                                        new RunnableWithExceptionProtection.CallbackWhenException() {
                                            @Override
                                            public void handle(Throwable t) {
                                                LOGGER.error("JVMService consumes and upload failure.", t);
                                            }
                                        }
                                    ), 0, 1, TimeUnit.SECONDS);
    }
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 26
 27
 28
 29
 30
 31
 32
 33
 34
 35
 36
 37
 38
 39
 40
 41
 42
 43
 prepare()方法中初始化JVM信息的发送工具，boot()方法中初始化收集JVM信息的定时任务collectMetricFuture和发送JVM信息的定时任务sendMetricFuture，collectMetricFuture实际上执行的是JVMService的run()方法，sendMetricFuture实际上执行的是JVMMetricsSender的run()方法

先来看下JVMService的run()方法，代码如下：

@DefaultImplementor
public class JVMService implements BootService, Runnable {

    private JVMMetricsSender sender; // JVM信息的发送工具
    
    /**
     * 构建jvm信息交给sender,由sender负责发送给OAP
     */
    @Override
    public void run() {
        long currentTimeMillis = System.currentTimeMillis();
        try {
            JVMMetric.Builder jvmBuilder = JVMMetric.newBuilder();
            jvmBuilder.setTime(currentTimeMillis);
            jvmBuilder.setCpu(CPUProvider.INSTANCE.getCpuMetric());
            jvmBuilder.addAllMemory(MemoryProvider.INSTANCE.getMemoryMetricList());
            jvmBuilder.addAllMemoryPool(MemoryPoolProvider.INSTANCE.getMemoryPoolMetricsList());
            jvmBuilder.addAllGc(GCProvider.INSTANCE.getGCList());
            jvmBuilder.setThread(ThreadProvider.INSTANCE.getThreadMetrics());
            jvmBuilder.setClazz(ClassProvider.INSTANCE.getClassMetrics());
    
            sender.offer(jvmBuilder.build());
        } catch (Exception e) {
            LOGGER.error(e, "Collect JVM info fail.");
        }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
run()方法就是构建JVM信息交给JVMMetricsSender，后续由JVMMetricsSender负责发送给OAP，JVMMetricsSender代码如下：

@DefaultImplementor
public class JVMMetricsSender implements BootService, Runnable, GRPCChannelListener {
    private static final ILog LOGGER = LogManager.getLogger(JVMMetricsSender.class);

    private volatile GRPCChannelStatus status = GRPCChannelStatus.DISCONNECT;
    private volatile JVMMetricReportServiceGrpc.JVMMetricReportServiceBlockingStub stub = null;
    
    private LinkedBlockingQueue<JVMMetric> queue;
    
    @Override
    public void prepare() {
        // 初始化queue,默认最多存储600个数据
        queue = new LinkedBlockingQueue<>(Config.Jvm.BUFFER_SIZE);
        ServiceManager.INSTANCE.findService(GRPCChannelManager.class).addChannelListener(this);
    }
    
    @Override
    public void boot() {
    
    }
    
    public void offer(JVMMetric metric) {
        // drop last message and re-deliver
        if (!queue.offer(metric)) {
            queue.poll();
            queue.offer(metric);
        }
    }
    
    @Override
    public void run() {
        if (status == GRPCChannelStatus.CONNECTED) {
            try {
                JVMMetricCollection.Builder builder = JVMMetricCollection.newBuilder();
                LinkedList<JVMMetric> buffer = new LinkedList<>();
                // 将queue中的数据移到buffer中
                queue.drainTo(buffer);
                if (buffer.size() > 0) {
                    builder.addAllMetrics(buffer);
                    builder.setService(Config.Agent.SERVICE_NAME);
                    builder.setServiceInstance(Config.Agent.INSTANCE_NAME);
                    // 发送到OAP,OAP返回的Command交给CommandService去处理
                    Commands commands = stub.withDeadlineAfter(GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS)
                                            .collect(builder.build());
                    ServiceManager.INSTANCE.findService(CommandService.class).receiveCommand(commands);
                }
            } catch (Throwable t) {
                LOGGER.error(t, "send JVM metrics to Collector fail.");
                ServiceManager.INSTANCE.findService(GRPCChannelManager.class).reportError(t);
            }
        }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
JVMMetricsSender中的queue存储未发送的JVM信息，run()方法负责将JVM信息发送给OAP，OAP返回的Command交给CommandService去处理

小结：


19、服务-KafkaXxxService
SkyWalking Agent上报数据有两种模式：

使用GRPC直连OAP上报数据
Agent发送数据到Kafka，OAP从Kafka中消费数据

如果使用发送Kafka的模式，Agent和OAP依然存在GRPC直连，只是大部分的采集数据上报都改为发送Kafka的模式

KafkaXxxService就是对应服务的Kafka实现，例如：GRPCChannelManager是负责Agent到OAP的网络连接，对应KafkaProducerManager是负责Agent到OAP的Kafka的连接；KafkaJVMMetricsSender负责JVM信息的发送对应GRPC的JVMMetricsSender；KafkaServiceManagementServiceClient负责Agent Client信息的上报对应GRPC的ServiceManagementClient

小结：


20、服务-StatusCheckService
StatusCheckService是用来判断哪些异常不算异常，核心就是statuscheck.ignored_exceptions和statuscheck.max_recursive_depth两个配置项，代码如下：

/**
 * The <code>StatusCheckService</code> determines whether the span should be tagged in error status if an exception
 * captured in the scope.
 * 用来判断哪些异常不算异常
 */
 @DefaultImplementor
 public class StatusCheckService implements BootService {

    @Getter
    private String[] ignoredExceptionNames;

    private StatusChecker statusChecker;

    @Override
    public void prepare() throws Throwable {
        // 一条链路如果某个环节出现了异常,默认情况会把异常信息发送给OAP,在SkyWalking UI中看到链路中那个地方出现了异常,方便排查问题
        // 但是在一些场景中,异常是用来控制业务流程的,而不应该把它当做错误
        // 这个配置就是需要忽略的异常
        ignoredExceptionNames = Arrays.stream(Config.StatusCheck.IGNORED_EXCEPTIONS.split(","))
                                      .filter(StringUtil::isNotEmpty)
                                      .toArray(String[]::new);
        // 检查异常时的最大递归程度
        // AException
        //  BException
        //   CException
        // 如果IGNORED_EXCEPTIONS配置的是AException,此时抛出的是CException需要递归找一下是否属于AException的子类
        statusChecker = Config.StatusCheck.MAX_RECURSIVE_DEPTH > 0 ? HIERARCHY_MATCH : OFF;
    }
 1
 2
 3
 4
 5
 6
 7
 8
 9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 26
 27
 28
 小结：


参考：

SkyWalking8.7.0源码分析（如果你对SkyWalking Agent源码感兴趣的话，强烈建议看下该教程）
