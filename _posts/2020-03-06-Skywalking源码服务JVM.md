---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码服务JVM


## 服务-JVMService
JVMService是一个定时器，来收集JVM的cpu、内存、内存池、gc、线程和类的信息，然后将收集到的信息通过GRPCChannelManager提供的网络连接发送给OAP

```
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
```

prepare()方法中初始化JVM信息的发送工具，boot()方法中初始化收集JVM信息的定时任务collectMetricFuture和发送JVM信息的定时任务sendMetricFuture，collectMetricFuture实际上执行的是JVMService的run()方法，sendMetricFuture实际上执行的是JVMMetricsSender的run()方法

先来看下JVMService的run()方法，代码如下：
```
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

```
run()方法就是构建JVM信息交给JVMMetricsSender，后续由JVMMetricsSender负责发送给OAP，JVMMetricsSender代码如下：

```
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

```
JVMMetricsSender中的queue存储未发送的JVM信息，run()方法负责将JVM信息发送给OAP，OAP返回的Command交给CommandService去处理







