---
layout: post
categories: RocketMQ
description: none
keywords: RocketMQ
---
# RocketMQ源码-生产者启动

RockerMQ生产者与Name Server、Broker进行通信，从Name Server获取到Broker相关的信息，这个过程称为Broker发现，获取到Broker相关信息，就可以与Broker服务进行通信了，发送信息给Broker，Broker将接收到生产者发送的信息保存起来。我们先看看RocketMQ是如何发送信息的：

```java
public static void main(String[] args) throws MQClientException, InterruptedException {

        //创建默认的生产者
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        //设置 Name Server服务器的地址
        producer.setNamesrvAddr("name-server1-ip:9876;name-server2-ip:9876");
        //启动生产者
        producer.start();
        //循环发送信息
        for (int i = 0; i < 1000; i++) {
            try {

                Message msg = new Message("TopicTest" ,"TagA" ,("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET) );
                SendResult sendResult = producer.send(msg);
                System.out.printf("%s%n", sendResult);
            } catch (Exception e) {
                e.printStackTrace();
                Thread.sleep(1000);
            }
        }

        producer.shutdown();
}
```

上述代码是RocketMQ给的生产者发送消费的例子，首先创建默认的DefaultMQProducer，DefaultMQProducer就是默认的消息生产者，用来消息生产以及消息发送的 ,然后设置Name Server的服务器地址，这样生产者与Name Server进行通信，从Name Server服务器获取Broker相关信息以及路由相关信息，设置好Name Server服务器地址以后，循环发送消息。这就是RocketMQ生产者发送消息的例子。

这里我首先会分析下RocketMQ生产者的启动过程，生产者发送消息的分析将在另外一篇文章进行分析。

```java
//代发的位置：org.apache.rocketmq.client.producer.DefaultMQProducer#DefaultMQProducer
public class DefaultMQProducer extends ClientConfig implements MQProducer {
    //代码省略
}
```

DefaultMQProducer继承ClientConfig类，以及实现MQProducer类，从名字看。ClientConfig就是客户端配置。MQProducer跟消息发送有关，实际上也是这样。DefaultMQProducer的重要属性如下：

```
 public class DefaultMQProducer extends ClientConfig implements MQProducer {

    /**
     * Wrapping internal implementations for virtually all methods presented in this class.
     */
    // 装饰者模式，用于包装消息的发送实现
    protected final transient DefaultMQProducerImpl defaultMQProducerImpl;
    private final InternalLogger log = ClientLogger.getLog();
    /**
     * Producer group conceptually aggregates all producer instances of exactly same role, which is particularly
     * important when transactional messages are involved. </p>
     *
     * For non-transactional messages, it does not matter as long as it's unique per process. </p>
     *
     * See {@linktourl http://rocketmq.apache.org/docs/core-concept/} for more discussion.
     */
    // 生产者所属组，消息服务器在回查事务状态时会随机选择该组中任何一个生产者发起事务回查请求。
    private String producerGroup;

    /**
     * Just for testing or demo program
     */
    //  默认topicKey 。 TBW102
    private String createTopicKey = MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC;

    /**
     * Number of queues to create per default topic.
     */
    // 默认主题在每一个Broker 队列数量。
    private volatile int defaultTopicQueueNums = 4;

    /**
     * Timeout for sending messages.
     */
    // 发送消息默认超时时间， 默认3s 。
    private int sendMsgTimeout = 3000;

    /**
     * Compress message body threshold, namely, message body larger than 4k will be compressed on default.
     */
    // 消息体超过该值则启用压缩，默认4K。
    private int compressMsgBodyOverHowmuch = 1024 * 4;

    /**
     * Maximum number of retry to perform internally before claiming sending failure in synchronous mode. </p>
     *
     * This may potentially cause message duplication which is up to application developers to resolve.
     */
    // 同步方式发送消息重试次数，默认为2 ，总共执行3 次。
    private int retryTimesWhenSendFailed = 2;

    /**
     * Maximum number of retry to perform internally before claiming sending failure in asynchronous mode. </p>
     *
     * This may potentially cause message duplication which is up to application developers to resolve.
     */
    // 异步方式发送消息重试次数，默认为2 。
    private int retryTimesWhenSendAsyncFailed = 2;

    /**
     * Indicate whether to retry another broker on sending failure internally.
     */
    // 消息重试时选择另外一个Broker 时, 是否不等待存储结果就返回， 默认为false 。
    private boolean retryAnotherBrokerWhenNotStoreOK = false;

    /**
     * Maximum allowed message size in bytes.
     */
    // 允许发送的最大消息长度，默认为4M ，最大值为2^32-1 。
    private int maxMessageSize = 1024 * 1024 * 4; // 4M
```

其中ClientConfig类中的重要配置信息

```
// 客户端通用配置，包含生产者和消费者
public class ClientConfig {
    public static final String SEND_MESSAGE_WITH_VIP_CHANNEL_PROPERTY = "com.rocketmq.sendMessageWithVIPChannel";
    // NameServer地址
    private String namesrvAddr = NameServerAddressUtils.getNameServerAddresses();
    // 客户端IP,当客户端是生产者，这个就是生产者IP，当客户端是消费者，这个就是消费者IP
    private String clientIP = RemotingUtil.getLocalAddress();
    // 实例名或者客户端名称，从rocketmq.client.name查找，默认是DEFAULT
    private String instanceName = System.getProperty("rocketmq.client.name", "DEFAULT");
    private int clientCallbackExecutorThreads = Runtime.getRuntime().availableProcessors();
    protected String namespace;
    protected AccessChannel accessChannel = AccessChannel.LOCAL;

    /**
     * Pulling topic information interval from the named server
     */
    // 从NameServer拉去Broker信息。默认是30s
    private int pollNameServerInterval = 1000 * 30;
    /**
     * Heartbeat interval in microseconds with message broker
     */
    // 心跳时间，默认是30s
    private int heartbeatBrokerInterval = 1000 * 30;
    /**
     * Offset persistent interval for consumer
     */
    // 消费者位移持久化时间，默认是5s
    private int persistConsumerOffsetInterval = 1000 * 5;
    // 当发生异常，默认延迟拉去消息时间，默认是1s
    private long pullTimeDelayMillsWhenException = 1000;
    ... 
    }
```

在上述消息发送的例子中，首先创建了DefaultMQProducer，我们来看看在创建DefaultMQProducer的时候，具体做了什么事：

```java
//代发的位置：org.apache.rocketmq.client.producer.DefaultMQProducer#DefaultMQProducer
public DefaultMQProducer(final String namespace, final String producerGroup, RPCHook rpcHook) {
        //命名空间
        this.namespace = namespace;
        //生产者组
        this.producerGroup = producerGroup;
        //默认的消息发送实现
        defaultMQProducerImpl = new DefaultMQProducerImpl(this, rpcHook);
}
```

DefaultMQProducer的上述构造器方法里面，主要是为namespace、producerGroup、defaultMQProducerImpl属性进行赋值，namespace是命名空间，producerGroup是生产者组、defaultMQProducerImpl是默认的消息发送实现，消息的发送由defaultMQProducerImpl负责。继续往下看，我们看看DefaultMQProducerImpl的构造函数，如下：

```java
//代发的位置：org.apache.rocketmq.client.producer.DefaultMQProducerImpl#DefaultMQProducerImpl
public DefaultMQProducerImpl(final DefaultMQProducer defaultMQProducer, RPCHook rpcHook) {
        this.defaultMQProducer = defaultMQProducer;
        this.rpcHook = rpcHook;

        //异步发送线程队列
        this.asyncSenderThreadPoolQueue = new LinkedBlockingQueue<Runnable>(50000);
        //默认异步发送线程池
        this.defaultAsyncSenderExecutor = new ThreadPoolExecutor(
            Runtime.getRuntime().availableProcessors(),
            Runtime.getRuntime().availableProcessors(),
            1000 * 60,
            TimeUnit.MILLISECONDS,
            this.asyncSenderThreadPoolQueue,
            new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, "AsyncSenderExecutor_" + this.threadIndex.incrementAndGet());
                }
            });
    }
```

DefaultMQProducerImpl构造器为一些属性赋值，设置默认的生产者、钩子、以及异步发送线程队列、默认异步发送线程池。分析完DefaultMQProducer的构建函数，我们就可以看看生产者的启动了：

```java
//代码位置：org.apache.rocketmq.client.producer.DefaultMQProducer#start
public void start() throws MQClientException {
        //设置生产者组
        this.setProducerGroup(withNamespace(this.producerGroup));
        //默认的生产者实现启动
        this.defaultMQProducerImpl.start();
        //消息轨迹跟踪
        if (null != traceDispatcher) {
            try {
                traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
            } catch (MQClientException e) {
                log.warn("trace dispatcher start failed ", e);
            }
        }
}
```

start方法主要做了三件事，设置生产者的名字、启动默认的生产者实现、当消息轨迹跟踪不为空时，启动消息轨迹跟踪器。我们主要是分析生产者的启动过程，所以继续深入到this.defaultMQProducerImpl.start()中，this.defaultMQProducerImpl.start()方法有调用了有参数的start方法，这个方法主要做了下面几件事：

- 根据服务状态，创建和启动MQ实例客户端
- 发送心跳给Broker
- 定时任务扫描过期的请求

### 根据服务状态，创建和启动MQ实例客户端

```java
//代码位置：org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#start
public void start(final boolean startFactory) throws MQClientException {
        switch (this.serviceState) {
            //刚创建
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                //检验配置
                this.checkConfig();

                //如果生产组不等于CLIENT_INNER_PRODUCER，更改实例的名字为pid
                if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                    this.defaultMQProducer.changeInstanceNameToPID();
                }

                //创建MQ客户端实例
                this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);

                //注册生产者，保存在map中
                boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
                //如果注册不成功，则抛出异常，
                if (!registerOK) {
                    this.serviceState = ServiceState.CREATE_JUST;
                    throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                        + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                        null);
                }

                this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

                //启动MQ客户端实例
                if (startFactory) {
                    mQClientFactory.start();
                }

                //设置服务器状态
                log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                    this.defaultMQProducer.isSendMessageWithVIPChannel());
                this.serviceState = ServiceState.RUNNING;
                break;
            //其他状态，抛出异常
            case RUNNING:
            case START_FAILED:
            case SHUTDOWN_ALREADY:
                throw new MQClientException("The producer service state not OK, maybe started once, "
                    + this.serviceState
                    + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                    null);
            default:
                break;
        }

      //代码省略
  }
```

start方法代码的逻辑比较多，将代码分块分析，start方法首先根据serviceState(服务器状态)做不同的逻辑，如果是RUNNING(运行中)、START_FAILED（启动失败）、SHUTDOWN_ALREADY（已经关闭）状态时，抛出异常，这些状态时不能重复调用start方法的。当serviceState是CREATE_JUST（刚创建）时，说明是第一次启动生产者，this.checkConfig()检查下producerGroup（生产者组）的合法性，如果producerGroup不等于CLIENT_INNER_PRODUCER（客户端内部的生产者组），将instanceName（实例名）设置为运行服务的PID，PID就是服务的进程号。接着用MQClientManager类创建MQ客户端实例，然后注册生产者，如果注册失败，就抛出生产者组已经存在的异常，注册完生产者以后接着往topic信息表存入topic信息，最后是启动这个MQClientInstance对象，设置服务器的状态。这就是当第一次启动生产者时所做一些事情，比较粗略的分析了主要逻辑，接下来从细节更深入下去。

### **创建MQ客户端实例**

```java
//代码位置：org.apache.rocketmq.client.impl.MQClientManager#getOrCreateMQClientInstance
//获取或者创建MQ客户端实例
public MQClientInstance getOrCreateMQClientInstance(final ClientConfig clientConfig, RPCHook rpcHook) {
        //创建客户端实例Id  IP@实例名@unitName
        String clientId = clientConfig.buildMQClientId();
        //根据clientId获取MQ客户端实例
        MQClientInstance instance = this.factoryTable.get(clientId);
        //如果MQ客户端实例等于空，就创建,然后加入MQ实例表缓存起来
        if (null == instance) {
            instance =
                new MQClientInstance(clientConfig.cloneClientConfig(),
                    this.factoryIndexGenerator.getAndIncrement(), clientId, rpcHook);
            MQClientInstance prev = this.factoryTable.putIfAbsent(clientId, instance);
            if (prev != null) {
                instance = prev;
                log.warn("Returned Previous MQClientInstance for clientId:[{}]", clientId);
            } else {
                log.info("Created new MQClientInstance for clientId:[{}]", clientId);
            }
        }

        return instance;
}
```

getOrCreateMQClientInstance方法首先创建客户端实例ID，形式如IP@实例名@unitName，然后根据clientId从实例工厂表中获取MQ客户端实例instance，如果instance等于null，那么就创建一个MQ客户端实例添加到实例工厂表中保存，最后将找到的instance返回。

### **注册生产者**

```java
//代码位置：org.apache.rocketmq.client.impl.factory.MQClientInstance#registerProducer
public boolean registerProducer(final String group, final DefaultMQProducerImpl producer) {
        if (null == group || null == producer) {
            return false;
        }

        MQProducerInner prev = this.producerTable.putIfAbsent(group, producer);
        if (prev != null) {
            log.warn("the producer group[{}] exist already.", group);
            return false;
        }

        return true;
}
```

registerProducer方法首先判断方法入参的合法性，group和producer其中之一为空，直接返回注册不成功。参数校验合法以后，就以group为key，producer为value保存到生产者注册表中，如果已经存在生产者注册表了，那么就直接返回注册不成功，否则返回注册成功。

### **MQ客户端的启动**

```java
//代码位置：org.apache.rocketmq.client.impl.factory.MQClientInstance#start
public void start() throws MQClientException {

        synchronized (this) {
            switch (this.serviceState) {
                case CREATE_JUST:
                    this.serviceState = ServiceState.START_FAILED;
                    // If not specified,looking address from name server
                    // name server 地址为null，则通过http获取
                    if (null == this.clientConfig.getNamesrvAddr()) {
                        this.mQClientAPIImpl.fetchNameServerAddr();
                    }
                    // Start request-response channel
                    //启动请求-响应连接
                    this.mQClientAPIImpl.start();
                    // Start various schedule tasks
                    //启动多种定时任务
                    this.startScheduledTask();
                    // Start pull service
                    //启动拉取消息服务
                    this.pullMessageService.start();
                    // Start rebalance service
                    //启动负载均衡服务
                    this.rebalanceService.start();
                    // Start push service
                    //启动发送消息服务
                    this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                    log.info("the client factory [{}] start OK", this.clientId);
                    this.serviceState = ServiceState.RUNNING;
                    break;
                case START_FAILED:
                    throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
                default:
                    break;
            }
        }
    }
```

start方法里面用synchronized修饰，防止一时间多次调用这个方法，这样就不会被重复启动了。start方法根据serviceState（服务状态）判断选择走什么逻辑，当serviceState为START_FAILED（启动失败）时，抛出该实例之前已经被创建过的异常。当serviceState为CREATE_JUST时，主要做了下面几件事：

- 获取Name server地址
- 启动客户端网络连接（启动请求-响应连接）
- 启动多种定时任务
- 启动拉取消息服务
- 启动负载均衡服务
- 启动发送消息服务

### 获取Name server地址

```java
//代码位置：org.apache.rocketmq.client.impl.MQClientAPIImpl#fetchNameServerAddr
public String fetchNameServerAddr() {
        try {
            //通过http获取Name server地址
            String addrs = this.topAddressing.fetchNSAddr();
            //如果地址不为null，那么更新Name server地址
            if (addrs != null) {
                if (!addrs.equals(this.nameSrvAddr)) {
                    log.info("name server address changed, old=" + this.nameSrvAddr + ", new=" + addrs);
                    //更新Name server  地址
                    this.updateNameServerAddressList(addrs);
                    this.nameSrvAddr = addrs;
                    return nameSrvAddr;
                }
            }
        } catch (Exception e) {
            log.error("fetchNameServerAddr Exception", e);
        }
        return nameSrvAddr;
}
```

fetchNameServerAddr方法首先通过http请求获取到Name Server地址addrs，如果addrs不为空，判断addrs地址是否改变，如果改变，则更新Name Server地址。让我们深入看看获取Name Server的方法fetchNSAddr：

```java
//代码位置：org.apache.rocketmq.common.namesrv.TopAddressing#fetchNSAddr
public final String fetchNSAddr(boolean verbose, long timeoutMills) {
        String url = this.wsAddr;
        try {
            if (!UtilAll.isBlank(this.unitName)) {
                url = url + "-" + this.unitName + "?nofix=1";
            }
            HttpTinyClient.HttpResult result = HttpTinyClient.httpGet(url, null, null, "UTF-8", timeoutMills);
            if (200 == result.code) {
                String responseStr = result.content;
                if (responseStr != null) {
                    return clearNewLine(responseStr);
                } else {
                    log.error("fetch nameserver address is null");
                }
            } else {
                log.error("fetch nameserver address failed. statusCode=" + result.code);
            }
        } catch (IOException e) {
            if (verbose) {
                log.error("fetch name server address exception", e);
            }
        }

        if (verbose) {
            String errorMsg =
                "connect to " + url + " failed, maybe the domain name " + MixAll.getWSAddr() + " not bind in /etc/hosts";
            errorMsg += FAQUrl.suggestTodo(FAQUrl.NAME_SERVER_ADDR_NOT_EXIST_URL);

            log.warn(errorMsg);
        }
        return null;
}
```

fetchNSAddr()方法又调用了fetchNSAddr(boolean verbose, long timeoutMills)方法，verbose参数为是否打印日志，timeoutMills参数为http请求的超时时间。http请求的地址url默认是“[http://jmenv.tbsite.net:8080/rocketmq/nsaddr](https://link.zhihu.com/?target=http%3A//jmenv.tbsite.net%3A8080/rocketmq/nsaddr)”，url是在创建TopAddressing对象时构造的。当请求成功时，则将响应返回。让我们回到fetchNameServerAddr方法中，当请求的结果不为空，那么就更新Name server 地址：

```java
//代码位置：org.apache.rocketmq.client.impl.MQClientAPIImpl#updateNameServerAddressList
public void updateNameServerAddressList(final String addrs) {
        String[] addrArray = addrs.split(";");
        List<String> list = Arrays.asList(addrArray);
        this.remotingClient.updateNameServerAddressList(list);
}
```

updateNameServerAddressList方法首先将入参addrs以“；”分割得到地址列表，然后通信客户端进行更新Name Server列表。remotingClient是netty网络通信客户端，remotingClient.updateNameServerAddressList方法的代码如下：

```java
//代码位置:org.apache.rocketmq.remoting.netty.NettyRemotingClient#updateNameServerAddressList
public void updateNameServerAddressList(List<String> addrs) {
        List<String> old = this.namesrvAddrList.get();
        boolean update = false;

        if (!addrs.isEmpty()) {
            //name server 地址是否变化
            if (null == old) {
                update = true;
            } else if (addrs.size() != old.size()) {
                update = true;
            } else {
                for (int i = 0; i < addrs.size() && !update; i++) {
                    if (!old.contains(addrs.get(i))) {
                        update = true;
                    }
                }
            }

            if (update) {
                Collections.shuffle(addrs);
                log.info("name server address updated. NEW : {} , OLD: {}", addrs, old);
                this.namesrvAddrList.set(addrs);
            }
        }
}
```

updateNameServerAddressList方法首先判断Name Server地址是否发生了变化，如果旧的Name Server地址列表为null、新旧Name server地址列表大小不一样、旧的Name Server地址列表不包括新的Name Server地址的元素，那么Name Server地址就发生改变了，如果发生改变了，那么就将新的Name Server地址列表打乱，然后保存在Name Server列表namesrvAddrList中，namesrvAddrList是客户端用来缓存Name Server的。

### 启动客户端网络连接

```java
//代码位置：org.apache.rocketmq.client.impl.MQClientAPIImpl#start
public void start() {
        this.remotingClient.start();
}
```

start方法只有一行代码，启动客户端网络连接，本质就是启动netty客户端连接，我们再深入 this.remotingClient.start()， this.remotingClient.start()代码主要做了三件事：

- 初始化netty客户端
- 定时扫描响应
- 启动netty事件监听器

初始化netty客户端代码如下：

```java
//代码位置：org.apache.rocketmq.remoting.netty.NettyRemotingClient#start
public void start() {

        //默认的事件线程组
        this.defaultEventExecutorGroup = new DefaultEventExecutorGroup(
            nettyClientConfig.getClientWorkerThreads(),
            new ThreadFactory() {

                private AtomicInteger threadIndex = new AtomicInteger(0);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, "NettyClientWorkerThread_" + this.threadIndex.incrementAndGet());
                }
            });

        //初始化netty客户端启动器
        Bootstrap handler = this.bootstrap.group(this.eventLoopGroupWorker).channel(NioSocketChannel.class)
                //tcp不延迟
            .option(ChannelOption.TCP_NODELAY, true)
                //保持长连接
            .option(ChannelOption.SO_KEEPALIVE, false)
                //连接超时时间
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, nettyClientConfig.getConnectTimeoutMillis())
                //发送socket buffer的大小
            .option(ChannelOption.SO_SNDBUF, nettyClientConfig.getClientSocketSndBufSize())
                //接收socket buffer的大小
            .option(ChannelOption.SO_RCVBUF, nettyClientConfig.getClientSocketRcvBufSize())
            .handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ChannelPipeline pipeline = ch.pipeline();
                    if (nettyClientConfig.isUseTLS()) {
                        if (null != sslContext) {
                            pipeline.addFirst(defaultEventExecutorGroup, "sslHandler", sslContext.newHandler(ch.alloc()));
                            log.info("Prepend SSL handler");
                        } else {
                            log.warn("Connections are insecure as SSLContext is null!");
                        }
                    }
                    pipeline.addLast(
                        defaultEventExecutorGroup,
                        new NettyEncoder(),//编码器
                        new NettyDecoder(),//解码器
                        new IdleStateHandler(0, 0, nettyClientConfig.getClientChannelMaxIdleTimeSeconds()),//连接空闲处理器
                        new NettyConnectManageHandler(),//netty 连接处理器
                        new NettyClientHandler());//消息处理器
                }
            });
      //代码省略      
 }
```

初始化netty客户端时，首先初始化了defaultEventExecutorGroup（默认的事件线程组），默认处理多线程任务。然后/初始化netty客户端启动器，设置了连接的相关属性、设置了一些处理器，如消息的编码器、消息解码器、空闲连接处理器以及消息处理器。NettyClientHandler就是消息处理器，根据消息类型（请求、响应）走不同的处理逻辑。

```java
//代码位置：org.apache.rocketmq.remoting.netty.NettyRemotingClient#start
public void start() {
    //初始化netty客户端
    //定时扫描响应
    this.timer.scheduleAtFixedRate(new TimerTask() {
            @Override
          public void run() {
             try {
                   NettyRemotingClient.this.scanResponseTable();
              } catch (Throwable e) {
                  log.error("scanResponseTable exception", e);
              }
          }
    }, 1000 * 3, 1000);

    //代码省略
}
```

定时扫描响应，利用timer每秒中扫描响应结果，看是否已经过期，具体的逻辑如下：

```java
//代码位置：org.apache.rocketmq.remoting.netty.NettyRemotingClient#scanResponseTable
public void scanResponseTable() {
        final List<ResponseFuture> rfList = new LinkedList<ResponseFuture>();
        Iterator<Entry<Integer, ResponseFuture>> it = this.responseTable.entrySet().iterator();
        while (it.hasNext()) {
            Entry<Integer, ResponseFuture> next = it.next();
            ResponseFuture rep = next.getValue();

            //删除超时的请求，将过超时的请求保存在list
            if ((rep.getBeginTimestamp() + rep.getTimeoutMillis() + 1000) <= System.currentTimeMillis()) {
                rep.release();
                it.remove();
                rfList.add(rep);
                log.warn("remove timeout request, " + rep);
            }
        }

        //对超时的请求,在线程中执行响应结果回调，对结果进行处理
        for (ResponseFuture rf : rfList) {
            try {
                executeInvokeCallback(rf);
            } catch (Throwable e) {
                log.warn("scanResponseTable, operationComplete Exception", e);
            }
        }

    //省略代码
}
```

scanResponseTable方法中responseTable保存的是requestId（请求的唯一id）与响应结果，通过requestId就可以找到对应的响应结果了，由于请求都是异步处理的，所以ResponseFuture是异步响应结果。scanResponseTable方法首先遍历responseTable，找到已经超时的请求，将超时的请求从responseTable删除掉，，并添加到一个列表中保存起来。请求的时间加上超时时间再加上1000小于等于当前时间，那么说明该请求已经超时。当找到所有的超时请求时，对所有的超时请求执行响应结果回调，对结果进行处理，具体看看结果处理方法：

```java
//代码位置：org.apache.rocketmq.remoting.netty.NettyRemotingAbstract#executeInvokeCallback
private void executeInvokeCallback(final ResponseFuture responseFuture) {
        boolean runInThisThread = false;
        //获取线程池
        ExecutorService executor = this.getCallbackExecutor();
        //如果线程池不为空
        if (executor != null) {
            try {
                executor.submit(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            responseFuture.executeInvokeCallback();
                        } catch (Throwable e) {
                            log.warn("execute callback in executor exception, and callback throw", e);
                        } finally {
                            responseFuture.release();
                        }
                    }
                });
            } catch (Exception e) {
                runInThisThread = true;
                log.warn("execute callback in executor exception, maybe executor busy", e);
            }
        } else {
            runInThisThread = true;
        }

        if (runInThisThread) {
            try {
                responseFuture.executeInvokeCallback();
            } catch (Throwable e) {
                log.warn("executeInvokeCallback Exception", e);
            } finally {
                responseFuture.release();
            }
        }
 }
```

executeInvokeCallback方法首先获取回调线程池，如果回调线程池不为空，则在回调线程池中执行响应结果的回调方法，否则，直接在本线程中执行响应结果的回调方法。接着我们回到start方法：

```java
//代码位置：org.apache.rocketmq.remoting.netty.NettyRemotingClient#start
public void start() {
    //初始化netty客户端
    //定时扫描响应
    //启动netty事件监听器
      if (this.channelEventListener != null) {
            this.nettyEventExecutor.start();
     }
}
```

启动netty事件监听器，首先判断当channelEventListener（连接事件连接）不等于null，就启动netty事件线程。ChannelEventListener类是接口类，有几个抽象方法：onChannelConnect（连接监听处理方法）、onChannelClose（连接关闭处理方法）、onChannelException（连接异常处理方法）、onChannelIdle（连接空闲处理方法）。NettyEventExecutor是线程，当this.nettyEventExecutor.start()执行时，会调用NettyEventExecutor的start方法，start方法如下：

```java
//代码位置：org.apache.rocketmq.remoting.netty.NettyRemotingAbstract.NettyEventExecutor#run
public void run() {
            log.info(this.getServiceName() + " service started");

            //获取连接事件监听器
            final ChannelEventListener listener = NettyRemotingAbstract.this.getChannelEventListener();

            //从队列中一直获取连接事件，根据事件类型
            while (!this.isStopped()) {
                try {
                    NettyEvent event = this.eventQueue.poll(3000, TimeUnit.MILLISECONDS);
                    if (event != null && listener != null) {
                        switch (event.getType()) {
                            //闲置的
                            case IDLE:
                                listener.onChannelIdle(event.getRemoteAddr(), event.getChannel());
                                break;
                              //关闭
                            case CLOSE:
                                listener.onChannelClose(event.getRemoteAddr(), event.getChannel());
                                break;
                               //连接
                            case CONNECT:
                                listener.onChannelConnect(event.getRemoteAddr(), event.getChannel());
                                break;
                                //发生异常
                            case EXCEPTION:
                                listener.onChannelException(event.getRemoteAddr(), event.getChannel());
                                break;
                            default:
                                break;

                        }
                    }
                } catch (Exception e) {
                    log.warn(this.getServiceName() + " service has exception. ", e);
                }
            }

            log.info(this.getServiceName() + " service end");
 }
```

run方法首先获取连接事件监听器，然后从事件队列（eventQueue）中不断获取事件，eventQueue是阻塞队列，当事件不等于null和监听器不等于null时，根据事件类型调用监听器的不同方法，这些方法就是ChannelEventListener的抽象方法，其实在这里，不管是什么事件状态，实现的逻辑都是一样，在这里只挑选IDLE（闲置的）的事件进行分析：

```java
public void onChannelIdle(String remoteAddr, Channel channel) {
     //生产者管理关闭连接
     this.brokerController.getProducerManager().doChannelCloseEvent(remoteAddr, channel);
     //消费者管理关闭连接
     this.brokerController.getConsumerManager().doChannelCloseEvent(remoteAddr, channel);
     //过滤服务管理关闭连接
     this.brokerController.getFilterServerManager().doChannelCloseEvent(remoteAddr, channel);
}
```

onChannelIdle方法主要做了几件事：生产者管理关闭连接、消费者管理关闭连接、过滤服务管理关闭连接。这几件事的逻辑大体都是相同，就是从缓存表中找到响应的连接，然后进行关闭或者移除，更深入的逻辑就不分析，也比较简单，感兴趣的可以自己深入看看。

### 启动多种定时任务

startScheduledTask方法启动多种定时任务，主要启动下面的任务：

- 每2分钟获取Name server 地址
- 每30秒从Name server拉取路由消息并更新
- 每三十秒清理下线的Broker和发送心跳
- 每5秒持久化消费位移
- 每分钟调整线程池

具体代码如下：

```java
//代码位置：org.apache.rocketmq.client.impl.factory.MQClientInstance#startScheduledTask
private void startScheduledTask() {
        if (null == this.clientConfig.getNamesrvAddr()) {
            //每2分钟获取Name server 地址
            this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

                @Override
                public void run() {
                    try {
                        MQClientInstance.this.mQClientAPIImpl.fetchNameServerAddr();
                    } catch (Exception e) {
                        log.error("ScheduledTask fetchNameServerAddr exception", e);
                    }
                }
            }, 1000 * 10, 1000 * 60 * 2, TimeUnit.MILLISECONDS);
        }

        //每30秒从Name server拉取路由消息并更新
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    MQClientInstance.this.updateTopicRouteInfoFromNameServer();
                } catch (Exception e) {
                    log.error("ScheduledTask updateTopicRouteInfoFromNameServer exception", e);
                }
            }
        }, 10, this.clientConfig.getPollNameServerInterval(), TimeUnit.MILLISECONDS);

        //每三十秒清理下线的Broker和发送心跳
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    MQClientInstance.this.cleanOfflineBroker();
                    MQClientInstance.this.sendHeartbeatToAllBrokerWithLock();
                } catch (Exception e) {
                    log.error("ScheduledTask sendHeartbeatToAllBroker exception", e);
                }
            }
        }, 1000, this.clientConfig.getHeartbeatBrokerInterval(), TimeUnit.MILLISECONDS);

        //每5秒持久化消费位移
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    MQClientInstance.this.persistAllConsumerOffset();
                } catch (Exception e) {
                    log.error("ScheduledTask persistAllConsumerOffset exception", e);
                }
            }
        }, 1000 * 10, this.clientConfig.getPersistConsumerOffsetInterval(), TimeUnit.MILLISECONDS);

        //每分钟调整线程池
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    MQClientInstance.this.adjustThreadPool();
                } catch (Exception e) {
                    log.error("ScheduledTask adjustThreadPool exception", e);
                }
            }
        }, 1, 1, TimeUnit.MINUTES);
}
```

每2分钟获取Name server 地址的定时任务与上面获取Name Server地址的分析是一样的，代码共用，这里就不分析这个定时任务了。

**每30秒从Name server拉取路由消息并更新**

我们分析下每30秒从Name server拉取路由消息并更新的定时任务，代码如下：

```java
//代码位置：org.apache.rocketmq.client.impl.factory.MQClientInstance#updateTopicRouteInfoFromNameServer
public void updateTopicRouteInfoFromNameServer() {
        Set<String> topicList = new HashSet<String>();

        // Consumer
        {
            //遍历消费者
            Iterator<Entry<String, MQConsumerInner>> it = this.consumerTable.entrySet().iterator();
            while (it.hasNext()) {
                Entry<String, MQConsumerInner> entry = it.next();
                MQConsumerInner impl = entry.getValue();
                if (impl != null) {
                    //所有订阅数据
                    Set<SubscriptionData> subList = impl.subscriptions();
                    if (subList != null) {
                        for (SubscriptionData subData : subList) {
                            //所有的topic
                            topicList.add(subData.getTopic());
                        }
                    }
                }
            }
        }

        // Producer
        {
            Iterator<Entry<String, MQProducerInner>> it = this.producerTable.entrySet().iterator();
            while (it.hasNext()) {
                Entry<String, MQProducerInner> entry = it.next();
                MQProducerInner impl = entry.getValue();
                if (impl != null) {
                    Set<String> lst = impl.getPublishTopicList();
                    topicList.addAll(lst);
                }
            }
        }

        //更新topic路由信息
        for (String topic : topicList) {
            this.updateTopicRouteInfoFromNameServer(topic);
        }
}
```

consumerTable、producerTable分别保存着消费者和生产者的相关信息，updateTopicRouteInfoFromNameServer方法首先遍历consumerTable、producerTable，从两者之中获取所有的topic存到列表中，然后从Name Server服务器拉取topic，比较topic是否改变，如果改变了，将客户端的保存的路由信息进行更新，具体看看updateTopicRouteInfoFromNameServer的逻辑：

```java
//代码位置：org.apache.rocketmq.client.impl.factory.MQClientInstance#updateTopicRouteInfoFromNameServer
public boolean updateTopicRouteInfoFromNameServer(final String topic) {
   return updateTopicRouteInfoFromNameServer(topic, false, null);
}
```

updateTopicRouteInfoFromNameServer方法又调用具有三个参数的updateTopicRouteInfoFromNameServer方法，另外两个参数分别是isDefault、defaultMQProducer，isDefault传入的值是false，defaultMQProducer传入的值是null。有三个参数的updateTopicRouteInfoFromNameServer方法主要做了两件事：

- 从Name Server获取topic路由数据
- 更新路由相关信息

**从Name Server获取topic路由数据**

```java
//代码位置：org.apache.rocketmq.client.impl.factory.MQClientInstance#updateTopicRouteInfoFromNameServer
public boolean updateTopicRouteInfoFromNameServer(final String topic, boolean isDefault,DefaultMQProducer defaultMQProducer) {
        //省略代码
        TopicRouteData topicRouteData;
         if (isDefault && defaultMQProducer != null) {
              //从name server获取路由数据
              topicRouteData = this.mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer(defaultMQProducer.getCreateTopicKey(),
                                1000 * 3);
              if (topicRouteData != null) {
                     for (QueueData data : topicRouteData.getQueueDatas()) {
                        //设置topic的读写队列数
                         int queueNums = Math.min(defaultMQProducer.getDefaultTopicQueueNums(), data.getReadQueueNums());
                          data.setReadQueueNums(queueNums);
                          data.setWriteQueueNums(queueNums);
                      }
               }
          } else {
                topicRouteData = this.mQClientAPIImpl.getTopicRouteInfoFromNameServer(topic, 1000 * 3);
          }
        //省略代码
}
```

上面将与一些不影响逻辑的代码省略了，上面if分支都是从Name Server获取路由数据，但是获取路由数据传入的topic不同。isDefault参数的值是false，defaultMQProducer传入的值是null，所以走的是else分支，通过传入的topic参数获取路由数据。getTopicRouteInfoFromNameServer方法最终会通过netty客户端请求Name Server服务器的路由信息，getTopicRouteInfoFromNameServer获取构造请求头以及请求体，调用netty客户端的请求发送方法。

**更新路由相关信息**

```java
//代码位置：org.apache.rocketmq.client.impl.factory.MQClientInstance#updateTopicRouteInfoFromNameServer
public boolean updateTopicRouteInfoFromNameServer(final String topic, boolean isDefault,DefaultMQProducer defaultMQProducer) {
    //代码省略
    if (topicRouteData != null) {
           TopicRouteData old = this.topicRouteTable.get(topic);
            //比较topic路由信息是否已经改变
            boolean changed = topicRouteDataIsChange(old, topicRouteData);
            if (!changed) {
                 //是否需要更新topic路由
                  changed = this.isNeedUpdateTopicRouteInfo(topic);
             } else {
                   log.info("the topic[{}] route info changed, old[{}] ,new[{}]", topic, old, topicRouteData);
             }
            //如果改变
              if (changed) {
                   TopicRouteData cloneTopicRouteData = topicRouteData.cloneTopicRouteData();

                    //如果更改了，保存broker 名字和地址的对应关系
                     for (BrokerData bd : topicRouteData.getBrokerDatas()) {
                         this.brokerAddrTable.put(bd.getBrokerName(), bd.getBrokerAddrs());
                     }

                      // Update Pub info
                      //更新发布的topic info
                       {
                           TopicPublishInfo publishInfo = topicRouteData2TopicPublishInfo(topic, topicRouteData);
                            publishInfo.setHaveTopicRouterInfo(true);
                            Iterator<Entry<String, MQProducerInner>> it = this.producerTable.entrySet().iterator();
                            while (it.hasNext()) {
                                Entry<String, MQProducerInner> entry = it.next();
                                MQProducerInner impl = entry.getValue();
                                if (impl != null) {
                                    impl.updateTopicPublishInfo(topic, publishInfo);
                                 }
                             }
                        }

                         // Update sub info
                         //更新订阅的topic info
                         {
                             Set<MessageQueue> subscribeInfo = topicRouteData2TopicSubscribeInfo(topic, topicRouteData);
                             Iterator<Entry<String, MQConsumerInner>> it = this.consumerTable.entrySet().iterator();
                             while (it.hasNext()) {
                                  Entry<String, MQConsumerInner> entry = it.next();
                                  MQConsumerInner impl = entry.getValue();
                                  if (impl != null) {
                                      impl.updateTopicSubscribeInfo(topic, subscribeInfo);
                                  }
                              }
                          }
                          log.info("topicRouteTable.put. Topic = {}, TopicRouteData[{}]", topic, cloneTopicRouteData);
                          this.topicRouteTable.put(topic, cloneTopicRouteData);
                          return true;
             }
    } else {
          log.warn("updateTopicRouteInfoFromNameServer, getTopicRouteInfoFromNameServer return null, Topic: {}", topic);
    }
    //代码省略
}
```

首先比较新老topic路由信息是否一样，如果不一样，说明topic路由信息已经改变。如果topic路由信息已经改变，那么就更新Broker元数据、更新发布的topic info、更新订阅的topic info，发布的topic info实际就是生产者发布消息有关的topic，订阅的topic info实际就是消费者消费消息有关的topic，最后更新topic路由信息，保存在topicRouteTable表中。

**每三十秒清理下线的Broker和发送心跳**

首先分析下清理下线的Broker，代码如下：

```java
//代码位置：org.apache.rocketmq.client.impl.factory.MQClientInstance#cleanOfflineBroker
 private void cleanOfflineBroker() {
        try {
            //上锁
            if (this.lockNamesrv.tryLock(LOCK_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS))
                try {
                  ConcurrentHashMap<String, HashMap<Long, String>> updatedTable = new ConcurrentHashMap<String, HashMap<Long, String>>();
                    //遍历所有的Broker
                    Iterator<Entry<String, HashMap<Long, String>>> itBrokerTable = this.brokerAddrTable.entrySet().iterator();
                    while (itBrokerTable.hasNext()) {
                        Entry<String, HashMap<Long, String>> entry = itBrokerTable.next();
                        String brokerName = entry.getKey();
                        HashMap<Long, String> oneTable = entry.getValue();

                        HashMap<Long, String> cloneAddrTable = new HashMap<Long, String>();
                        cloneAddrTable.putAll(oneTable);

                        Iterator<Entry<Long, String>> it = cloneAddrTable.entrySet().iterator();
                        while (it.hasNext()) {
                            Entry<Long, String> ee = it.next();
                            String addr = ee.getValue();
                            //broker地址不存在topic路由地址map中，则删除broker
                            if (!this.isBrokerAddrExistInTopicRouteTable(addr)) {
                                it.remove();
                                log.info("the broker addr[{} {}] is offline, remove it", brokerName, addr);
                            }
                        }

                        //删除下线Broker是否没有其他Broker，如果为空，说明没有元素了，直接删除
                        if (cloneAddrTable.isEmpty()) {
                            itBrokerTable.remove();
                            log.info("the broker[{}] name's host is offline, remove it", brokerName);
                        } else {
                            //说明还剩下有未下线的Broker
                            updatedTable.put(brokerName, cloneAddrTable);
                        }
                    }

                    //如果有未下线的元素，则保存在brokerAddrTable中
                    if (!updatedTable.isEmpty()) {
                        this.brokerAddrTable.putAll(updatedTable);
                    }
                } finally {
                    this.lockNamesrv.unlock();
                }
        } catch (InterruptedException e) {
            log.warn("cleanOfflineBroker Exception", e);
        }
}
```

brokerAddrTable保存着< Broker Name ,>的对应关系，Broker Name对应一个的map，首先遍历brokerAddrTable，获取Broker Name对应的 的map，然后继续遍历 的map，判断broker地址是否存在topic 路由表中。如果不存在的话，则删除掉broker地址对应的元素，如果删除下线的Broker以后，没有map没有其他元素，那么就删除brokerAddrTable中Broker Name 对应的 元素，如果还有未下线的元素，则保存回brokerAddrTable表中。isBrokerAddrExistInTopicRouteTable方法是判断Broker地址是否存在topic 路由表中，具体逻辑也比较简单，就是遍历topicRouteTable的元素，当topicRouteTable中元素的BrokerData（Broker元数据）包含Broker地址，那么就返回true，否则返回false。代码如下：

```java
//代码位置：org.apache.rocketmq.client.impl.factory.MQClientInstance#isBrokerAddrExistInTopicRouteTable
private boolean isBrokerAddrExistInTopicRouteTable(final String addr) {
        Iterator<Entry<String, TopicRouteData>> it = this.topicRouteTable.entrySet().iterator();
        while (it.hasNext()) {
            Entry<String, TopicRouteData> entry = it.next();
            TopicRouteData topicRouteData = entry.getValue();
            List<BrokerData> bds = topicRouteData.getBrokerDatas();
            for (BrokerData bd : bds) {
                if (bd.getBrokerAddrs() != null) {
                    boolean exist = bd.getBrokerAddrs().containsValue(addr);
                    if (exist)
                        return true;
                }
            }
        }

        return false;
}
```

分析完清理下线的Broker后，看看发送心跳给Broker，检测客户端与Broker的连接是否正常：

```java
//代码位置：org.apache.rocketmq.client.impl.factory.MQClientInstance#sendHeartbeatToAllBroker
private void sendHeartbeatToAllBroker() {
        //封装消费者、生产者心跳数据
        final HeartbeatData heartbeatData = this.prepareHeartbeatData();
        final boolean producerEmpty = heartbeatData.getProducerDataSet().isEmpty();
        final boolean consumerEmpty = heartbeatData.getConsumerDataSet().isEmpty();
        //如果没有生产者和消费者，不发送心跳，直接返回
        if (producerEmpty && consumerEmpty) {
            log.warn("sending heartbeat, but no consumer and no producer");
            return;
        }

        if (!this.brokerAddrTable.isEmpty()) {
            //发送心跳的次数
            long times = this.sendHeartbeatTimesTotal.getAndIncrement();
            Iterator<Entry<String, HashMap<Long, String>>> it = this.brokerAddrTable.entrySet().iterator();
            //遍历broker地址
            while (it.hasNext()) {
                Entry<String, HashMap<Long, String>> entry = it.next();
                String brokerName = entry.getKey();
                HashMap<Long, String> oneTable = entry.getValue();
                if (oneTable != null) {
                    for (Map.Entry<Long, String> entry1 : oneTable.entrySet()) {
                        Long id = entry1.getKey();
                        String addr = entry1.getValue();
                        if (addr != null) {
                            //消费者为空
                            if (consumerEmpty) {
                                //不是master
                                if (id != MixAll.MASTER_ID)
                                    continue;
                            }

                            try {
                                //版本
                                int version = this.mQClientAPIImpl.sendHearbeat(addr, heartbeatData, 3000);
                                if (!this.brokerVersionTable.containsKey(brokerName)) {
                                    this.brokerVersionTable.put(brokerName, new HashMap<String, Integer>(4));
                                }
                                this.brokerVersionTable.get(brokerName).put(addr, version);
                                //每累计20次打印心跳日志
                                if (times % 20 == 0) {
                                    log.info("send heart beat to broker[{} {} {}] success", brokerName, id, addr);
                                    log.info(heartbeatData.toString());
                                }
                            } catch (Exception e) {
                                if (this.isBrokerInNameServer(addr)) {
                                    log.info("send heart beat to broker[{} {} {}] failed", brokerName, id, addr, e);
                                } else {
                                    log.info("send heart beat to broker[{} {} {}] exception, because the broker not up, forget it", brokerName,
                                        id, addr, e);
                                }
                            }
                        }
                    }
                }
            }
        }
}
```

sendHeartbeatToAllBroker方法首先通过prepareHeartbeatData方法封装心跳数据（HeartbeatData），prepareHeartbeatData方法将遍历生产者和消费者集合，生成心跳数据，如果生产者和消费都为空，那么就不用发送心跳数据给Broker服务器了，直接返回。如果Broker地址不为空，那么遍历Broker地址的表，给每一个Broker地址，通过netty客户端给Broker服务器发送心跳，并将返回的版本保存起来，每累计20次数打印心跳日志。

**每5秒持久化消费位移**

```java
//代码位置：org.apache.rocketmq.client.impl.factory.MQClientInstance#persistAllConsumerOffset
private void persistAllConsumerOffset() {
        Iterator<Entry<String, MQConsumerInner>> it = this.consumerTable.entrySet().iterator();
        while (it.hasNext()) {
            Entry<String, MQConsumerInner> entry = it.next();
            MQConsumerInner impl = entry.getValue();
            impl.persistConsumerOffset();
        }
}
```

consumerTable保存着所有的 消费者，遍历consumerTable，得到消费者，然后调用消费者的persistConsumerOffset方法进行消费位移持久化。persistConsumerOffset方法有三种实现，主要看消费者是以哪种方式进行消费消息的，消费者消费消息有推拉两种方式，所以消费位移持久化persistConsumerOffset方法也有推拉的实现。

**每分钟调整线程池**

每分钟调整线程池的定时任务，实际上没有调整线程池大小的具体实现，这里就不分析了，居停实现调整线程池的大小是一个空的方法。

多种定时任务启动以后，接下来是启动拉取消息线程服务、启动负载均衡线程服务、以及启动消息发送服务，这些就不在这里分析了，等到分析消息的消费、客户端的负载均衡再具体分析下。到此为止，生产者启动的过程就分析完成。