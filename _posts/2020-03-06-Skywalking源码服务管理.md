---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码服务管理


## 服务-ServiceManagementClient
ServiceManagementClient实现了GRPCChannelListener接口，statusChanged()方法代码如下：
```
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

```

prepare阶段代码如下：
```
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

```

prepare阶段时，先向GRPCChannelManager注册自己为监听器，然后把配置文件中的Agent Client信息放入集合，等待发送

boot阶段代码如下：
```
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

```

boot()方法中初始化心跳定时任务heartbeatFuture，Runnable传入的是this，实际上执行的是ServiceManagementClient的run()方法，代码如下：
```
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

```
run()方法处理逻辑如下：

心跳周期为30s，信息汇报频率因子为10，所以每5分钟向OAP汇报一次Agent Client Properties

判断本次是否是Agent信息上报
- 如果本次是信息上报，上报Agent信息，包括：服务名、实例名、Agent Client的信息、当前操作系统的信息、JVM信息
- 如果本次不是信息上报，请求服务端，将服务端给到的响应交给CommandService去处理

如果配置文件中未指定实例名，ServiceInstanceGenerator会生成一个实例名
```
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
```



