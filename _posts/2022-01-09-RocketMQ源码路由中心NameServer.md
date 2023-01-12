---
layout: post
categories: RocketMQ
description: none
keywords: RocketMQ
---
# RocketMQ源码路由中心NameServer
NameServer 是整个RocketMQ 的“大脑”，主要负责RocketMQ 路由管理、服务注册及服务发现的机制。

## RocketMQ 的架构

RocketMQ架构设计[RocketMQ架构设计](https://rocketmq.apache.org/assets/images/RocketMQ%E9%83%A8%E7%BD%B2%E6%9E%B6%E6%9E%84-ee0435f80da5faecf47bca69b1c831cb.png)

分析NameServer时，首先看下RocketMQ的架构图，RocketMQ架构主要分为四部分，如上图所示：

- Producer：消息发布者，负责生产消息，提供多种发送方式将消息投递到Broker，如同步发送、异步发送、顺序发送、单向发送，为了确保消息投递到Broker，同步和异步方式都需要Broker确认消息，单向发送消息不需要。
- Consumer：消息消费者，负责消息消费，从Broker服务器获取消息提供应用程序，获取消息的方式主要为push退和pull拉两种模式，另外也支持集群方式和广播方式的消费。
- BrokerServer：Broker 主要负责消息的存储，投递和查询以及保证服务的高可用。Broker负责接收生产者发送的消息并存储、同时为消费者消费消息提供支持。
- NameServer：NameServer是简单的Topic路由注册中心，主要有两个功能：

1. Broker管理：Broker启动的时候会将自己的注册消息提供给NameServer，注册消息主要包括Broker地址、Broker名字、Broker Id、topic配置信息等作为路由信息的基本数据，提供心跳检测机制检测Broker是否存活。Broker集群中每一台Broker服务器都向NameServer集群服务的每一台NameServer注册自己的路由信息。
2. 路由信息管理：每个NameServer保存Broker集群服务器的所有的路由信息，提供给生产者和消费者获取路由信息，获取到路由信息就可以与Broker服务进行消息的投递和消费。由于每个NameServer都保存着完整的路由信息，即使一台NameServer服务下线了，生产者和消费者也能从其他的NameServer服务器获取到完整Broker路由细信息。

## NameServer

本章主要介绍RocketMQ 路由管理、服务注册及服务发现的机制，  。相信大家对“服务发现”这个词语并不陌生，分布式服务SOA 架构体系中会有服务注册中心，分布式服务SOA 的注册中心主要提供服务调用的解析服务，指引服务调用方（消费者）找到“远方”的服务提供者，完成网络通信，那么RocketMQ 的路由中心存储的是什么数据呢？作为一款高性能的消息中间件，如何避免NameServer 的单点故障，提供高可用性呢？让我们带着上述疑问， 一起进入RocketMQ NameServer 的精彩世界中来。

## NameServer 架构设计

消息中间件的设计思路一般基于主题的订阅发布机制消息生产者（ Producer ）发送某一主题的消息到消息服务器，消息服务器负责该消息的持久化存储，消息消费者(Consumer）订阅感兴趣的主题，**消息服务器根据订阅信息（路由信息）将消息推送到消费者（ PUSH 模式）或者消息消费者主动向消息服务器拉取消息（ PULL 模式），从而实现消息生产者与消息消费者解调**。为了避免消息服务器的单点故障导致的整个系统瘫痪，**通常会部署多台消息服务器共同承担消息的存储**。那消息生产者如何知道消息要发往哪台消息服务器呢？如果某一台消息服务器若机了，那么生产者如何在不重启服务的情况下感知呢？

NameServer 就是为了解决上述问题而设计的。

Broker 消息服务器在启动时向所有Name Server 注册，消息生产者（Producer）在发送消息之前先从Name Server 获取Broker 服务器地址列表，然后根据负载算法从列表中选择一台消息服务器进行消息发送。NameServer 与每台Broker 服务器保持长连接，并间隔30s 检测Broker 是否存活，如果检测到Broker 右机， 则从路由注册表中将其移除。**但是路由变化不会马上通知消息生产者，为什么要这样设计呢？这是为了降低NameServer 实现的复杂性，在消息发送端提供容错机制来保证消息发送的高可用性**。

NameServer 本身的高可用可通过部署多台Nameserver 服务器来实现，但彼此之间互不通信，也就是NameServer 服务器之间在某一时刻的数据并不会完全相同，但这对消息发送不会造成任何影响，这也是RocketMQ NameServer 设计的一个亮点， RocketMQNameServer 设计追求简单高效。

了解完NameServer的架构，接下来结合源码的方式从下面几方面来详细剖析下NameServer：

- NameServer的启动过程分析
- Broker管理和路由信息的管理

### NameServer的启动过程分析

NameServer作为Broker管理和路由信息管理的服务器，首先需要启动才能为Broker提供注册topic的功能、提供心跳检测Broker是否存活的功能、为生产者和消费者提供获取路由消息的功能，NameServer服务器相关的源码在namesrv模块下，目录结构如下：

``` yaml
namesrv
├─kvconfig                        	# KV配置管理
	├─KVConfigManager   			# KV config管理器
	├─KVConfigSerializeWrapper  	# kvConfig序列化包装器
├─processor               			# 处理器
	├─ClusterTestRequestProcessor 	#
	├─DefaultRequestProcessor    	# 默认处理器
├─routeinfo                      	# 路由信息
	├─BrokerHousekeepingService  	#
	├─RouteInfoManager           	# 维护路由信息管理，提供路由注册/查询等核心功能
	├─BrokerLiveInfo             	#
├─NamesrvController               	# Namesrv控制器
├─NamesrvStartup                  	# Namesrv启动器
```

NameServer 启动类： `org.apache.rocketmq.namesrv.NamesrvStartup`。NamesrvStartup类就是NameServer服务器启动的启动类。

```java
//代码位置：org.apache.rocketmq.namesrv.NamesrvStartup

// NameServer作为Broker管理和路由信息管理的服务器，
// 首先需要启动才能为Broker提供注册topic的功能、提供心跳检测Broker是否存活的功能、为生产者和消费者提供获取路由消息的功能
public class NamesrvStartup {

 ...   
 }
```

NamesrvStartup类中有一个main启动类，main方法调用main0，main0主要流程代码 （删除无关紧要或者不影响逻辑的代码，接下来所有有关源码的分析都只会分析主要流程，并且源码的分析采用从上到下，从宏观到微观的方法）如下：

```java
//代码位置：org.apache.rocketmq.namesrv.NamesrvStartup#main

    // NamesrvStartup类就是Name Server服务器启动的启动类，NamesrvStartup类中有一个main启动类，
    // main方法调用main0，main0主要流程代码
    public static void main(String[] args) {
        main0(args);
    }
    // main0 方法的主要作用就是创建Name Server服务器的控制器，并且启动Name Server服务器的控制器。
    public static NamesrvController main0(String[] args) {

        try {
            // //创建Name Server服务器的控制器
            NamesrvController controller = createNamesrvController(args);
            //启动Name Server服务器的控制器
            start(controller);
            //打印启动日志
            String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
            log.info(tip);
            System.out.printf("%s%n", tip);
            return controller;
        } catch (Throwable e) {
            e.printStackTrace();
            System.exit(-1);
        }

        return null;
    }
```

main0 方法的主要作用就是创建Name Server服务器的控制器，并且启动Name Server服务器的控制器。NamesrvController类的作用就是为Name Server服务的启动提供具体的逻辑实现，主要包括**配置信息的加载、远程通信服务器的创建和加载、默认处理器的注册以及心跳检测机器监控Broker的健康状态**等。Name Server服务器的控制器的创建方法为createNamesrvController方法，createNamesrvController方法的主要流程代码如下：

```java
//代码位置：org.apache.rocketmq.namesrv.NamesrvStartup#createNamesrvController

    // createNamesrvController方法主要做了几件事，
    // 读取和解析配置信息，包括Name Server服务的配置信息、Netty 服务器的配置信息、打印读取或者解析的配置信息、保存配置信息到本地文件中，
    // 以及根据namesrvConfig配置和nettyServerConfig配置作为参数创建nameServer 服务器的控制器。
    public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
        // 设置rocketMQ的版本信息，REMOTING_VERSION_KEY的值为：rocketmq.remoting.version，CURRENT_VERSION的值为：V4_7_0
        System.setProperty(RemotingCommand.REMOTING_VERSION_KEY, Integer.toString(MQVersion.CURRENT_VERSION));
        //PackageConflictDetect.detectFastjson();
        // 构建命令行，添加帮助命令和Name Server的提示命令，将createNamesrvController方法的args参数进行解析
        // 构造org.apache.commons.cli.Options,并添加-h -n参数，-h参数是打印帮助信息，-n参数是指定namesrvAddr
        Options options = ServerUtil.buildCommandlineOptions(new Options());
        //初始化commandLine，并在options中添加-c -p参数，-c指定nameserver的配置文件路径，-p标识打印配置信息
        commandLine = ServerUtil.parseCmdLine("mqnamesrv", args, buildCommandlineOptions(options), new PosixParser());
        if (null == commandLine) {
            System.exit(-1);
            return null;
        }
        // nameserver配置类，业务参数
        final NamesrvConfig namesrvConfig = new NamesrvConfig();
        // netty服务器配置类，网络参数
        final NettyServerConfig nettyServerConfig = new NettyServerConfig();
        // 设置nameserver的端口号
        nettyServerConfig.setListenPort(9876);
        // 判断上述构建的命令行是否有configFile（缩写为C）配置文件,如果有的话，则读取configFile配置文件的配置信息，
        // 并将转为NamesrvConfig和NettyServerConfig的配置信息
        if (commandLine.hasOption('c')) {
            String file = commandLine.getOptionValue('c');
            if (file != null) {
                InputStream in = new BufferedInputStream(new FileInputStream(file));
                properties = new Properties();
                properties.load(in);
                // 首先将构建的命令行转换为Properties，
                // 然后将通过反射的方式将Properties的属性转换为namesrvConfig的配置项和配置值。
                MixAll.properties2Object(properties, namesrvConfig);
                MixAll.properties2Object(properties, nettyServerConfig);
                //设置配置文件路径
                namesrvConfig.setConfigStorePath(file);

                System.out.printf("load config properties file OK, %s%n", file);
                in.close();
            }
        }
        // 如果构建的命令行存在字符'p'，就打印所有的配置信息病区退出方法
        if (commandLine.hasOption('p')) {
            // 打印nameServer 服务器配置类和 netty 服务器配置类的配置信息
            InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
            MixAll.printObjectProperties(console, namesrvConfig);
            MixAll.printObjectProperties(console, nettyServerConfig);
            // 打印参数命令不需要启动nameserver服务，只需要打印参数即可
            System.exit(0);
        }
        // 首先将构建的命令行转换为Properties，然后将通过反射的方式将Properties的属性转换为namesrvConfig的配置项和配置值。
        MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);
        // 检查ROCKETMQ_HOME，不能为空
        if (null == namesrvConfig.getRocketmqHome()) {
            System.out.printf("Please set the %s variable in your environment to match the location of the RocketMQ installation%n", MixAll.ROCKETMQ_HOME_ENV);
            System.exit(-2);
        }
        // 初始化logback日志工厂，rocketmq默认使用logback作为日志输出
        LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
        JoranConfigurator configurator = new JoranConfigurator();
        configurator.setContext(lc);
        lc.reset();
        configurator.doConfigure(namesrvConfig.getRocketmqHome() + "/conf/logback_namesrv.xml");

        log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);
        // 打印nameServer 服务器配置类和 netty 服务器配置类的配置信息
        MixAll.printObjectProperties(log, namesrvConfig);
        MixAll.printObjectProperties(log, nettyServerConfig);
        // 将namesrvConfig和nettyServerConfig作为参数创建nameServer 服务器的控制器
        final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

        // remember all configs to prevent discard
        // 将所有的配置保存在内存中（Properties）
        controller.getConfiguration().registerConfig(properties);

        return controller;
    }
```

createNamesrvController方法主要做了几件事，读取和解析配置信息，包括Name Server服务的配置信息、Netty 服务器的配置信息、打印读取或者解析的配置信息、保存配置信息到本地文件中，以及根据namesrvConfig配置和nettyServerConfig配置作为参数创建nameServer 服务器的控制器。创建好Name server控制器以后，就可以启动它了。启动Name Server的方法的主流程如下：

```java
//代码位置：org.apache.rocketmq.namesrv.NamesrvStartup#start
    // start方法主要作用就是进行初始化工作，然后进行启动Name Server控制器
    public static NamesrvController start(final NamesrvController controller) throws Exception {

        if (null == controller) {
            throw new IllegalArgumentException("NamesrvController is null");
        }
        // 初始化nameserver 服务器，如果初始化失败则退出
        boolean initResult = controller.initialize();
        if (!initResult) {
            controller.shutdown();
            System.exit(-3);
        }
        // 添加关闭的钩子，进行内存清理、对象销毁等操作
        // 如果代码中使用了线程池，一种优雅停机的方式就是注册一个JVM 钩子函数，在JVM进程关闭之前，先将线程池关闭，及时释放资源。
        Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
            @Override
            public Void call() throws Exception {
                // 优雅停机实现方法，释放资源
                controller.shutdown();
                return null;
            }
        }));
        // 启动
        controller.start();

        return controller;
    }
```

start方法没什么逻辑，主要作用就是进行初始化工作，然后进行启动Name Server控制器，接下来看看进行了哪些初始化工作以及如何启动Name Server的，初始化initialize方法的主要流程如下：

```java
//代码位置：org.apache.rocketmq.namesrv.NamesrvStartup#initialize
public boolean initialize() {
     // key-value 配置加载
     this.kvConfigManager.load();

     // //创建netty远程服务器，用来进行网络传输以及通信
     this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

      //远程服务器线程池
        this.remotingExecutor =
            Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

       //注册处理器
       this.registerProcessor();

      //每10秒扫描不活跃的broker
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.routeInfoManager.scanNotActiveBroker();
            }
        }, 5, 10, TimeUnit.SECONDS);

        //每10秒打印配置信息（key-value）
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.kvConfigManager.printAllPeriodically();
            }
        }, 1, 10, TimeUnit.MINUTES);

        //省略部分代码

        return true;

 }
```

initialize方法的主要逻辑如下：

- 加载配置文件。读取文件名为"**user.home**/namesrv/kvConfig.json"(其中**user.home**为用户的目录)，然后将读取的文件内容转为KVConfigSerializeWrapper类，最后将所有的key-value保存在如下map中：

```text
//用来保存不同命名空间的key-value
private final HashMap<String/* Namespace */, HashMap<String/* Key */, String/* Value */>> 
configTable = new HashMap<String, HashMap<String, String>>();
```

- 创建Netty服务器。Name Server 用netty与生产者、消费者以及Boker进行通信。
- 注册处理器。这里主要注册的是默认的处理器DefaultRequestProcessor，注册的逻辑主要是初始化DefaultRequestProcessor并保存着，待需要使用的时候直接使用。处理器的作用就是处理生产者、消费者以及Broker服务器的不同请求，比如获取生产者和消费者获取所有的路由信息，Broker服务器注册路由信息等。处理器DefaultRequestProcessor处理不同的请求将会在下面进行讲述。
- 执行定时任务。主要有两个定时任务，一个是每十秒扫描不活跃的Broker。并且将过期的Broker清理掉。另外一个是每十秒打印key-valu的配置信息。

上面就是initialize方法的主要逻辑，特别需要注意每10秒扫描不活跃的broker的定时任务：

```java
//NamesrvController.this.routeInfoManager.scanNotActiveBroker();
//代码位置：org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#scanNotActiveBroker
 public void scanNotActiveBroker() {
     //所有存活的Broker
     Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
     //遍历Broker
     while (it.hasNext()) {
            Entry<String, BrokerLiveInfo> next = it.next();
            long last = next.getValue().getLastUpdateTimestamp();
            //最后更新时间加上broker过期时间（120秒）小于当前时间，则关闭与broker的远程连接。并且将缓存在map中的broker信息删除
            if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
                RemotingUtil.closeChannel(next.getValue().getChannel());
                it.remove();
                //将过期的Channel连接清理掉。以及删除缓存的Broker
                this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
            }
        }
 }
```

scanNotActiveBroker方法的逻辑主要是遍历缓存在brokerLiveTable的Broker，将Broker最后更新时间加上120秒的结果是否小于当前时间，如果小于当前时间，说明Broker已经过期，可能是已经下线了，所以可以清除Broker信息，并且关闭Name Server 服务器与Broker服务器连接，这样被清除的Broker就不会与Name Server服务器进行远程通信了。brokerLiveTable的结果如下：

```java
//保存broker地址与broker存活信息的对应关系
private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
```

brokerLiveTable缓存着以brokerAddr为key（Broker 地址）,以BrokerLiveInfo为value的结果，BrokerLiveInfo是Broker存活对象，主要有如下几个属性：

```java
class BrokerLiveInfo {
    //最后更新时间
    private long lastUpdateTimestamp;
    //版本信息
    private DataVersion dataVersion;
    //连接
    private Channel channel;
    //高可用服务器地址
    private String haServerAddr;

    //省略代码
}
```

从BrokerLiveInfo中删除了过期的Broker后，还需要做清理Name Server服务器与Broker服务器的连接，onChannelDestroy方法主要是清理缓存在如下map的信息：

```java
////代码位置：org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager

//保存broker地址与broker存活信息的对应关系
private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
//保存broker地址与过滤服务器的对应关系，Filter Server 与消息过滤有关系
private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
//保存broker 名字与 broker元数据的关系
private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
//保存集群名字与集群下所有broker名字对应的关系
private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
//保存topic与topic下所有队列元数据的对应关系
 private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
```

在扫描过期的broker时，首先找到不活跃的broker，然后onChannelDestroy方法清理与该不活跃broker有关的缓存，清理的主要流程如下：

- 清理不活跃的broker存活信息。首先遍历brokerLiveTable找到不活跃的broker，然后删除brokerLiveTable中的与该不活跃的broker有关的缓存信息。
- 清理与消息过滤有关的缓存。找到不活跃的broker存活信息，删除filterServerTable中的与该broker地址有关的消息过滤的服务信息。
- 清理与不活跃broker的元素居。brokerAddrTable保存着broker名字与broker元素居对应的信息，BrokerData类保存着cluster、brokerName、brokerId与broker name。遍历brokerAddrTable找到与该不活跃broker的名字相等的broker元素进行删除。
- 清理集群下对应的不活跃broker名字。clusterAddrTable保存集群名字与集群下所有broker名字对应的关系，遍历clusterAddrTable的所有key，从clusterAddrTable中找到与不活跃broker名字相等的元素，然后删除。
- 清理与该不活跃broker的topic对应队列数据。topicQueueTable保存topic与topic下所有队列元数据的对应关系，QueueData保存着brokerName、readQueueNums（可读队列数量）、writeQueueNums（可写队列数量）等。遍历topicQueueTable的key，找到与不活跃broker名字相同的QueueData进行删除。

初始化nameserver 服务器以后，接下来就可以启动nameserver 服务器：

```java
//代码位置：org.apache.rocketmq.namesrv.NamesrvController#start
public void start() throws Exception {
    //启动远程服务器（netty 服务器）
    this.remotingServer.start();

    //启动文件监听线程
    if (this.fileWatchService != null) {
        this.fileWatchService.start();
    }
}
```

start方法做了两件事，第一件就是启动netty服务器，netty服务器主要负责与Broker、生产者与消费者之间的通信，处理Broker、生产者与消费者的不同请求。根据nettyConfig配置，设置启动的配置和各种处理器，然后采用netty服务器启动的模板启动服务器，具体的代码就不分析了，有兴趣的可以看看netty启动代码模板是怎么样的。第二件事就是启动文件监听线程，监听tts相关文件是否发生变化。

Name Server 服务器启动流程的源代码分析到此为止了，在这里总结下Name Server 服务器启动流程主要做了什么事：

- 加载和读取配置。设置Name Server 服务器启动的配置NamesrvConfig和启动Netty服务器启动的配置NettyServerConfig。
- 初始化相关的组件。netty服务类、远程服务线程池、处理器以及定时任务的初始化。
- 启动Netty服务器。Netty服务器用来与broker、生产者、消费者进行通信、处理与它们之间的各种请求，并且对请求的响应结果进行处理。

### Broker管理和路由信息的管理

Name Server 服务器的作用主要有两个：Broker管理和路由信息管理。

## **Broker管理**

在上面分析的Name Server 服务器的启动过程中，也有一个与Broker管理相关的分析，那就是启动一个定时线程池每十秒去扫描不活跃的Broker。将不活跃的Broker清理掉。除了在Name Server 服务器启动时启动定时任务去扫描不活跃的Broker外，Name Server 服务器启动以后，通过netty服务器接收Broker、生产者、消费者的不同请求，将接收到请求会交给在Name Server服务器启动时注册的处理器DefaultRequestProcessor类的processRequest方法处理。processRequest方法根据请求的不同类型，将请求交给不同的方法进行处理。有关Broker管理的请求主要有注册Broker、注销Broker，processRequest方法处理注册Broker、注销Broker请求的代吗如下：

```java
//代码位置：org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor#processRequest

public RemotingCommand processRequest(ChannelHandlerContext ctx,RemotingCommand request)  {
         switch (request.getCode()) {
            //省略无关代码

            //注册Broker
            case RequestCode.REGISTER_BROKER:
                Version brokerVersion = MQVersion.value2Version(request.getVersion());
                if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                    return this.registerBrokerWithFilterServer(ctx, request);
                } else {
                    return this.registerBroker(ctx, request);
                }
            //注销Broker
            case RequestCode.UNREGISTER_BROKER:
                return this.unregisterBroker(ctx, request);
           //省略无关代码
         }
 }
```

### Broker注册

Broker 服务器启动时，会向Name Server 服务器发送Broker 相关的信息，如集群的名字、Broker地址、Broker名字、topic相关信息等，注册Broker主要的代码比较长，接下来会分成好几部分进行讲解。如下：

```java
//代码位置：org.apache.rocketmq.namesrv.processor.RouteInfoManager#registerBroker
public RegisterBrokerResult registerBroker(
        final String clusterName,
        final String brokerAddr,
        final String brokerName,
        final long brokerId,
        final String haServerAddr,
        final TopicConfigSerializeWrapper topicConfigWrapper,
        final List<String> filterServerList,
        final Channel channel) {

     RegisterBrokerResult result = new RegisterBrokerResult();
     this.lock.writeLock().lockInterruptibly();
     //根据集群的名字获取所有的broker名字
     Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
     if (null == brokerNames) {
           brokerNames = new HashSet<String>();
           this.clusterAddrTable.put(clusterName, brokerNames);
      }
      //名字保存在broker名字中
      brokerNames.add(brokerName);

    //省略代码
}
```

registerBroker方法根据集群的名字获取该集群下所有的Broker名字的Set，如果不存在就创建并添加进clusterAddrTable中，clusterAddrTable保存着集群名字与该集群下所有的Broker名字对应关系，最后将broker名字保存在set中。

```java
public RegisterBrokerResult registerBroker(
        final String clusterName,
        final String brokerAddr,
        final String brokerName,
        final long brokerId,
        final String haServerAddr,
        final TopicConfigSerializeWrapper topicConfigWrapper,
        final List<String> filterServerList,
        final Channel channel) {

     //省略代码

     boolean registerFirst = false;
     //获取broker 元数据
     BrokerData brokerData = this.brokerAddrTable.get(brokerName);
     if (null == brokerData) {
           registerFirst = true;
           brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
           this.brokerAddrTable.put(brokerName, brokerData);
      }

     //获取所有的broker地址
     Map<Long, String> brokerAddrsMap = brokerData.getBrokerAddrs();
     Iterator<Entry<Long, String>> it = brokerAddrsMap.entrySet().iterator();
     while (it.hasNext()) {
           Entry<Long, String> item = it.next();
           if (null != brokerAddr && brokerAddr.equals(item.getValue()) && brokerId != item.getKey()) {
                  it.remove();
            }
     }

     String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
     registerFirst = registerFirst || (null == oldAddr);

   //省略代码

}
```

上述代码主要做了两件事：

- 缓存broker元数据信息。首先根据broker名字从brokerAddrTable中获取Broker元数据brokerData，如果brokerData不存在，说明是第一次注册，创建Broker元数据并添加进brokerAddrTable中，brokerAddrTable保存着Broker名字与Broker元数据对应的信息。
- 从Broker元数据brokerData中获取该元数据中的所有Broker地址信息brokerAddrsMap。brokerAddrsMap保存着brokerId与所有Broker名字对应信息。遍历brokerAddrsMap中的所有broker地址，查找与参数brokerAddr相同但是与参数borkerId不同的进行删除，保证一个broker名字对应着BrokerId，最后将参数brokerId与参数brokerAddr保存到brokerData元数据的brokerAddrsMap中进行缓存。

```java
public RegisterBrokerResult registerBroker(
        final String clusterName,
        final String brokerAddr,
        final String brokerName,
        final long brokerId,
        final String haServerAddr,
        final TopicConfigSerializeWrapper topicConfigWrapper,
        final List<String> filterServerList,
        final Channel channel) {

     //省略代码

     //如果topic的配置不空并且是broker master
     if (null != topicConfigWrapper && MixAll.MASTER_ID == brokerId) {
          //如果topic配置改变或者是第一次注册
          if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())|| registerFirst) {
                  //获取所有的topic配置
                  ConcurrentMap<String, TopicConfig> tcTable = topicConfigWrapper.getTopicConfigTable();
                  if (tcTable != null) {
                       //遍历topic配置，创建并更新队列元素
                       for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                             this.createAndUpdateQueueData(brokerName, entry.getValue());
                        }
                    }
             }
       }

    //省略代码

}
```

如果参数topicConfigWrapper不等于空，并且brokerId等于0时，判断topic是否改变，如果topic改变或者是第一次注册，获取所有的topic配置，并创建和更新队列元数据。QueueData保存着队列元数据，如Broker名字、写队列数量、读队列数量，如果队列缓存中不存在该队列元数据，则添加，否则遍历缓存map找到该队列元数据进行删除，如果是新添加的则添加进队列缓存中。

```java
public RegisterBrokerResult registerBroker(
        final String clusterName,
        final String brokerAddr,
        final String brokerName,
        final long brokerId,
        final String haServerAddr,
        final TopicConfigSerializeWrapper topicConfigWrapper,
        final List<String> filterServerList,
        final Channel channel) {

   //省略代码

    //创建broker存活对象，并进行保存
    BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,new BrokerLiveInfo(
                        System.currentTimeMillis(),
                        topicConfigWrapper.getDataVersion(),
                        channel,
                        haServerAddr));

    if (null == prevBrokerLiveInfo) {
           log.info("new broker registered, {} HAServer: {}", brokerAddr, haServerAddr);
    }

    //如果过滤服务地址不为空，则缓存到filterServerTable
    if (filterServerList != null) {
            if (filterServerList.isEmpty()) {
                   this.filterServerTable.remove(brokerAddr);
            } else {
                   this.filterServerTable.put(brokerAddr, filterServerList);
            }
     }

     //如果不是broker master，获取高可用服务器地址以及master地址
     if (MixAll.MASTER_ID != brokerId) {
              String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
              if (masterAddr != null) {
                   BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                   if (brokerLiveInfo != null) {
                         result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                         result.setMasterAddr(masterAddr);
                    }
               }
      }

      return result;

 }
```

最后代码片段，主要做了三件事，首先创建了Broker存活对象BrokerLiveInfo，添加到brokerLiveTable中缓存，在Name Server 启动时，供定时线程任务每十秒进行扫描。以确保非正常的Broker被清理掉。然后是判断参数filterServerList是否为空，如果不为空，则添加到filterServerTable缓存，filterServerTable保存着与消息过滤相关的过滤服务。最后，判断该注册的Broker不是Broker master，则设置高可用服务器地址以及master地址。到此为止，Broker注册的代码就分析完成了，总而言之，Broker注册就是Broker将相关的元数据信息，如Broker名字，Broker地址、topic信息发送给Name Server服务器，Name Server接收到以后将这些元数据缓存起来，以供后续能够快速找到这些元数据，生产者和消费者也可以通过Name Server服务器获取到Broke相关的信息，这样，生产者和消费者就可以和Broker服务器进行通信了，生产者发送消息给Broker服务器，消费者从Broker服务器消费消息。

### Broker注销

Broker注销的过程刚好跟Broker注册的过程相反，Broker注册是将Broker相关信息和Topic配置信息缓存起来，以供生产者和消费者使用。而Broker注销则是将Broker注销缓存的Broker信息从缓存中删除，Broker注销unregisterBroker方法主要代码流程如下：

```java
//代码位置：org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#unregisterBroker
public void unregisterBroker(
        final String clusterName,
        final String brokerAddr,
        final String brokerName,
        final long brokerId) {

    this.lock.writeLock().lockInterruptibly();
    //将缓存的broker存活对象删除
    BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.remove(brokerAddr);

     //将所有的过滤服务删除
     this.filterServerTable.remove(brokerAddr);

    boolean removeBrokerName = false;
    //删除broker元数据
     if (null != brokerData) {
           String addr = brokerData.getBrokerAddrs().remove(brokerId);
           if (brokerData.getBrokerAddrs().isEmpty()) {
                this.brokerAddrTable.remove(brokerName);
                removeBrokerName = true;
            }
      }

      //如果删除broker元数据成功
      if (removeBrokerName) {
           Set<String> nameSet = this.clusterAddrTable.get(clusterName);
           if (nameSet != null) {
                boolean removed = nameSet.remove(brokerName);
                if (nameSet.isEmpty()) {
                      this.clusterAddrTable.remove(clusterName);
                 }
            }
            //根据brokerName删除topic配置信息
            this.removeTopicByBrokerName(brokerName);
      }
      this.lock.writeLock().unlock();
}
```

unregisterBroker方法的参数有集群名字、broker地址、broker名字、brokerId，主要逻辑为：

- 根据broker地址删除broker存活对象。
- 根据broker地址删除所有消息过滤服务。
- 删除broker元数据。
- 如果删除元数据成功，则根据集群名字删除该集群的所有broker名字，以及根据根据brokerName删除topic配置信息。

### 路由信息的管理

处理器DefaultRequestProcessor类的processRequest方法除了处理Broker注册和Broker注销的请求外，还处路由信息管理有关的请求，接收到生产者和消费者的路由信息相关的请求，会交给处理器DefaultRequestProcessor类的processRequest方法处理，processRequest方法则会根据不同的请求类型将请求交给RouteInfoManager类的不同方法处理。RouteInfoManager类用map进行缓存路由相关信息，map如下：

```java
//topic与队列数据对应映射关系
private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
//broker 名字与broker 元数据对应映射关系
private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
//保存cluster的所有broker name
private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
//broker 地址 与 BrokerLiveInfo存活对象的对应映射关系
private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
//broker 地址 的所有过滤服务
private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```

RouteInfoManager类利用上面几个map缓存了Broker信息，topic相关信息、集群信息、消息过滤服务信息等，如果这些缓存的信息有变化，就是网这些map新增或删除缓存。这就是Name Server服务的路由信息管理。processRequest方法是如何处理路由信息管理的，具体实现可以阅读具体的代码，无非就是将不同的请求委托给RouteInfoManager的不同方法，RouteInfoManager的不同实现了上面缓存信息的管理。
NameServer 路由实现类： org.apache.rocketmq.namesrv.routeinfo.RoutelnfoManager ， 在了解路由注册之前，我们首先看一下NameServer 到底存储哪些信息。

```java
public class RouteInfoManager {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);
    // NameServer 与 Broker 空闲时长，默认2分钟，在2分钟内 Nameserver 没有收到 Broker 的心跳包，则关闭该连接。
    private final static long BROKER_CHANNEL_EXPIRED_TIME = 1000 * 60 * 2;
    //读写锁
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    // Topic,以及对应的队列信息 --消息发送时根据路由表进行负载均衡 。
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    // 以BrokerName为单位的Broker集合(Broker基础信息，包含 brokerName、所属集群名称、主备 Broker地址。)
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    // 集群以及属于该进群的Broker列表(根据一个集群名，获得对应的一组BrokerName的列表)
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    // 存活的Broker地址列表 (NameServer 每次收到心跳包时会 替换该信息 )
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    // Broker对应的Filter Server列表-消费端拉取消息用到
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
    ...省略...
} 
```

* topicQueueTable: Topic 消息队列路由信息，消息发送时根据路由表进行负载均衡。
* brokerAddrTable : Broker 基础信息， 包含brokerName 、所属集群名称、主备Broker地址。
* clusterAddrTable: Broker 集群信息，存储集群中所有Broker 名称。
* brokerLiveTable: Broker 状态信息。NameServer 每次收到心跳包时会替换该信息。
* filterServerTable : Broker 上的FilterServer 列表，用于类模式消息过滤。


RocketMQ 基于订阅发布机制， 一个Topic 拥有多个消息队列，一个Broker 为每一主题默认创建4 个读队列4 个写队列。多个Broker 组成一个集群， BrokerName 由相同的多台Broker组成Master-Slave 架构， brokerId 为0 代表Master ， 大于0 表示Slave 。BrokerLivelnfo 中的lastUpdateTimestamp 存储上次收到Broker 心跳包的时间。
QueueData 属性解析:

```java
/**
 * 队列信息
 */
public class QueueData implements Comparable<QueueData> {
    // 队列所属的Broker名称
    private String brokerName;
    // 读队列数量 默认：16
    private int readQueueNums;
    // 写队列数量 默认：16
    private int writeQueueNums;
    // Topic的读写权限(2是写 4是读 6是读写)
    private int perm;
    /** 同步复制还是异步复制--对应TopicConfig.topicSysFlag
     * {@link org.apache.rocketmq.common.sysflag.TopicSysFlag}
     */
    private int topicSynFlag;
        ...省略...
 }
```

map: topicQueueTable 数据格式demo(json)：

```java
{
    "TopicTest":[
        {
            "brokerName":"broker-a",
            "perm":6,
            "readQueueNums":4,
            "topicSynFlag":0,
            "writeQueueNums":4
        }
    ]
}
```

BrokerData 属性解析:

```java
/**
 * broker的数据:Master与Slave 的对应关系通过指定相同的BrokerName，不同的BrokerId来定义，BrokerId为0 表示Master，非0表示Slave。
 */
public class BrokerData implements Comparable<BrokerData> {
    // broker所属集群
    private String cluster;
    // brokerName
    private String brokerName;
    // 同一个brokerName下可以有一个Master和多个Slave,所以brokerAddrs是一个集合
    // brokerld=0表示 Master，大于0表示从 Slave
    private HashMap<Long/* brokerId */, String/* broker address */> brokerAddrs;
    // 用于查找broker地址
    private final Random random = new Random();
    ...省略...
 }
```

map: brokerAddrTable 数据格式demo(json)：

```json
{
    "broker-a":{
        "brokerAddrs":{
            "0":"172.16.62.75:10911"
        },
        "brokerName":"broker-a",
        "cluster":"DefaultCluster"
    }
}
```

BrokerLiveInfo 属性解析：

```java
/**
 *  存放存活的Broker信息，当前存活的 Broker,该信息不是实时的，NameServer 每10S扫描一次所有的 broker,根据心跳包的时间得知 broker的状态，
 *  该机制也是导致当一个 Broker 进程假死后，消息生产者无法立即感知，可能继续向其发送消息，导致失败（非高可用）
 */
class BrokerLiveInfo {
    //最后一次更新时间
    private long lastUpdateTimestamp;
    //版本号信息
    private DataVersion dataVersion;
    //Netty的Channel
    private Channel channel;
    //HA Broker的地址 是Slave从Master拉取数据时链接的地址,由brokerIp2+HA端口构成
    private String haServerAddr;
    ...省略...
 }
```

map: brokerLiveTable 数据格式demo(json)：

```json
 {
    "172.16.62.75:10911":{
        "channel":{
            "active":true,
            "inputShutdown":false,
            "open":true,
            "outputShutdown":false,
            "registered":true,
            "writable":true
        },
        "dataVersion":{
            "counter":2,
            "timestamp":1630907813571
        },
        "haServerAddr":"172.16.62.75:10912",
        "lastUpdateTimestamp":1630907814074
    }
}
```

brokerAddrTable -Map 数据格式demo(json)

```json
{"DefaultCluster":["broker-a"]}
```

从RouteInfoManager维护的HashMap数据结构和QueueData、BrokerData、BrokerLiveInfo类属性得知,NameServer维护的信息既简单但极其重要。



