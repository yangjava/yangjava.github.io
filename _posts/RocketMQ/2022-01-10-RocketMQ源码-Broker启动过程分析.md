---
layout: post
categories: RocketMQ
description: none
keywords: RocketMQ
---

# Broker 启动过程分析

Broker 主要负责消息的存储，投递和查询以及保证服务的高可用。Broker负责接收生产者发送的消息并存储、同时为消费者消费消息提供支持。为了实现这些功能，Broker包含几个重要的子模块：

- 通信模块：负责处理来自客户端（生产者、消费者）的请求。
- 客户端管理模块:负责管理客户端（生产者、消费者）和维护消费者的Topic订阅信息。
- 存储模块：提供存储消息和查询消息的能力，方便Broker将消息存储到硬盘。
- 高可用服务（HA Service）：提供数据冗余的能力，保证数据存储到多个服务器上，将Master Broker的数据同步到Slavew Broker上。
- 索引服务（Index service）：对投递到Broker的消息建立索引，提供快速查询消息的能力。

Broker在启动的过程中，将进行初始化的工作，初始化上述模块所需要的配置和资源等。在Name Server启动以后，Broker就可以开始启动了，启动过程将所有路由信息都注册到Name server服务器上，生产者就可以发送消息到Broker，消费者也可以从Broker消费消息。接下来就来看看Broker的具体启动过程。

```java
//源代码位置：org.apache.rocketmq.broker.BrokerStartup#main
public class BrokerStartup {
    public static Properties properties = null;
    public static CommandLine commandLine = null;
    public static String configFile = null;
    public static InternalLogger log;
    // Broker在启动的过程中，将进行初始化的工作，初始化上述模块所需要的配置和资源等。在Name Server启动以后，Broker就可以开始启动了，
    // 启动过程将所有路由信息都注册到Name server服务器上，生产者就可以发送消息到Broker，消费者也可以从Broker消费消息。
    // BrokerStartup 负责了Broker的启动，其中的主方法main（）即为整个消息服务器的入口函数。
    public static void main(String[] args) {
        //BrokerStartup类是Broker的启动类，在BrokerStartup类的main方法中，
        // 首先创建用createBrokerController方法创建Broker控制器（BrokerController类），Broker控制器主要负责Broker启动过程的具体的相关逻辑实现。
        // 创建好Broker 控制器以后，就可以启动Broker 控制器
        start(createBrokerController(args));
    }
    ...
      
```

BrokerStartup类是Broker的启动类，在BrokerStartup类的main方法中，首先创建用createBrokerController方法创建Broker控制器（BrokerController类），Broker控制器主要负责Broker启动过程的具体的相关逻辑实现。创建好Broker 控制器以后，就可以启动Broker 控制器了，所以下面将从两个部分分析Broker的启动过程：

1. 创建Broker控制器
2. 初始化配置信息
3. 创建并初始化Broker控制
4. 注册Broker关闭的钩子
5. 启动Broker控制器

## 创建Broker控制器

### 初始化配置信息

Broker在启动的时候，会初始化一些配置，如Broker配置、netty服务端配置、netty客户端配置、消息存储配置，为Broker启动提供配置准备。

```java
//源代码位置：org.apache.rocketmq.broker.BrokerStartup#createBrokerController
 public static BrokerController createBrokerController(String[] args) {
    /**
    省略代码
    注释：
        1、设置RocketMQ的版本
        2、设置netty接收和发送请求的buffer大小
        3、构建命令行：将命令行进行解析封装
    **/

     //broker配置、netty服务端配置、netty客户端配置
     final BrokerConfig brokerConfig = new BrokerConfig();
     final NettyServerConfig nettyServerConfig = new NettyServerConfig();
     final NettyClientConfig nettyClientConfig = new NettyClientConfig();
     nettyClientConfig.setUseTLS(Boolean.parseBoolean(System.getProperty(TLS_ENABLE,
                String.valueOf(TlsSystemConfig.tlsMode == TlsMode.ENFORCING))));
     //设置netty监听接口
     nettyServerConfig.setListenPort(10911);
     //消息存储配置
     final MessageStoreConfig messageStoreConfig = new MessageStoreConfig();
     //如果broker的角色是slave，设置命中消息在内存的最大比例
     if (BrokerRole.SLAVE == messageStoreConfig.getBrokerRole()) {
         int ratio = messageStoreConfig.getAccessMessageInMemoryMaxRatio() - 10;
         messageStoreConfig.setAccessMessageInMemoryMaxRatio(ratio);
     }
     //省略代码
 }
```

上述将一些不重要的代码和不影响主要逻辑的代码省略，分部分析createBrokerController方法的主要代码逻辑，省略的代码加入了注释。

createBrokerController方法创建BrokerConfig、NettyServerConfig、NettyClientConfig、MessageStoreConfig配置类，BrokerConfig类是Brokerd配置类。

- BrokerConfig：属性主要包括Broker相关的配置属性，如Broker名字、Broker Id、Broker连接的Name server地址、集群名字等。
- NettyServerConfig：Broker netty服务端配置类，Broker netty服务端主要用来接收客户端的请求，NettyServerConfig类主要属性包括监听接口、服务工作线程数、接收和发送请求的buffer大小等。
- NettyClientConfig：netty客户端配置类，用于生产者、消费者这些客户端与Broker进行通信相关配置，配置属性主要包括客户端工作线程数、客户端回调线程数、连接超时时间、连接不活跃时间间隔、连接最大闲置时间等。
- MessageStoreConfig：消息存储配置类，配置属性包括存储路径、commitlog文件存储目录、CommitLog文件的大小、CommitLog刷盘的时间间隔等。

创建完这些配置类以后，接下来会为这些配置类的一些配置属性设置值，先看看如下代码：

```java
//源代码位置：org.apache.rocketmq.broker.BrokerStartup#createBrokerController
 public static BrokerController createBrokerController(String[] args) {
     //省略代码

     //如果命令中包含字母c，则读取配置文件，将配置文件的内容设置到配置类中
      if (commandLine.hasOption('c')) {
          String file = commandLine.getOptionValue('c');
          if (file != null) {
              configFile = file;
              InputStream in = new BufferedInputStream(new FileInputStream(file));
              properties = new Properties();
              properties.load(in);

              //读取配置文件的中namesrv地址
              properties2SystemEnv(properties);
              //将配置文件中的配置项映射到配置类中去
              MixAll.properties2Object(properties, brokerConfig);
              MixAll.properties2Object(properties, nettyServerConfig);
              MixAll.properties2Object(properties, nettyClientConfig);
              MixAll.properties2Object(properties, messageStoreConfig);

              //设置配置broker配置文件
              BrokerPathConfigHelper.setBrokerConfigPath(file);
              in.close();
          }
      }
       //设置broker配置类
       MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), brokerConfig);

     //省略代码
 }
```

上述主要的代码逻辑为如果命令行中存在命令参数为‘c’（c是configFile的缩写），那么就读取configFile文件的内容，将configFile配置文件的配置项映射到BrokerConfig、NettyServerConfig、NettyClientConfig、MessageStoreConfig配置类中。接下来createBrokerController方法做一些判断必要配置的合法性，如下代码所示：

```java
//源代码位置：org.apache.rocketmq.broker.BrokerStartup#createBrokerController
public static BrokerController createBrokerController(String[] args) {
     //省略代码

     //如果broker配置文件的rocketmqHome属性值为null，直接结束程序
      if (null == brokerConfig.getRocketmqHome()) {
             System.out.printf("Please set the %s variable in your environment to match the location of 
                                  the RocketMQ installation", MixAll.ROCKETMQ_HOME_ENV);
             System.exit(-2);
       }

        //如果name server服务器的地址不为null
       String namesrvAddr = brokerConfig.getNamesrvAddr();
       if (null != namesrvAddr) {
            try {
                 //namesrvAddr是以";"分割的多个地址
                 String[] addrArray = namesrvAddr.split(";");
                 //每个地址是ip:port的形式，检测下是否形如ip:port的形式
                 for (String addr : addrArray) {
                     RemotingUtil.string2SocketAddress(addr);
                 }
              } catch (Exception e) {
                  System.out.printf(
                        "The Name Server Address[%s] illegal, please set it as follows, 
                           \"127.0.0.1:9876;192.168.0.1:9876\"%n",namesrvAddr);
                  System.exit(-3);
              }
         }

         //设置BrokerId，broker master 的BrokerId设置为0,broker slave 设置为大于0的值
          switch (messageStoreConfig.getBrokerRole()) {
              case ASYNC_MASTER:
              case SYNC_MASTER:
                  brokerConfig.setBrokerId(MixAll.MASTER_ID);
                  break;
               case SLAVE:
                  //如果小于等于0，退出程序
                  if (brokerConfig.getBrokerId() <= 0) {
                      System.out.printf("Slave's brokerId must be > 0");
                      System.exit(-3);
                   }

                   break;
                 default:
                    break;
             }

          //省略代码

 }
```

首先会判断下RocketmqHome的值是否为空，RocketmqHome是Borker相关配置保存的文件目录，如果为空则直接退出程序，启动Broker失败；然后判断下Name server 地址是否为空，如果不为空则解析以“；”分割的name server地址，检测下地址的合法性，如果不合法则直接退出程序；最后判断下Broker的角色，如果是master，BrokerId设置为0，如果是SLAVE，则BrokerId设置为大于0的数，否则直接退出程序，Broker启动失败。

createBrokerController方法进行必要配置参数的判断以后，将进行日志的设置、以及打印配置信息，主要代码如下：

```java
//源代码位置：org.apache.rocketmq.broker.BrokerStartup#createBrokerController
public static BrokerController createBrokerController(String[] args) {
    //省略代码
    //注释：日志设置

    //printConfigItem 打印配置信息
    if (commandLine.hasOption('p')) {
        InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.BROKER_CONSOLE_NAME);
        MixAll.printObjectProperties(console, brokerConfig);
        MixAll.printObjectProperties(console, nettyServerConfig);
        MixAll.printObjectProperties(console, nettyClientConfig);
        MixAll.printObjectProperties(console, messageStoreConfig);
        System.exit(0);
    } else if (commandLine.hasOption('m')) {
    //printImportantConfig 打印重要配置信息
        InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.BROKER_CONSOLE_NAME);
        MixAll.printObjectProperties(console, brokerConfig, true);
        MixAll.printObjectProperties(console, nettyServerConfig, true);
        MixAll.printObjectProperties(console, nettyClientConfig, true);
        MixAll.printObjectProperties(console, messageStoreConfig, true);
        System.exit(0);
    }

    //打印配置信息
    log = InternalLoggerFactory.getLogger(LoggerName.BROKER_LOGGER_NAME);
    MixAll.printObjectProperties(log, brokerConfig);
    MixAll.printObjectProperties(log, nettyServerConfig);
    MixAll.printObjectProperties(log, nettyClientConfig);
    MixAll.printObjectProperties(log, messageStoreConfig);

    //代码省略

}
```

createBrokerController方法的以上代码逻辑打印配置信息，先判断命令行参数是否包含字母‘p’（printConfigItem是我缩写），如果包含字母‘p’，则打印配置信息，否则判断下命令行是否包含字母‘m’，则打印被@ImportantField注解的配置属性，也就是重要的配置属性。最后，不管命令行中是否存在字母‘p’或者字母‘m’，都打印配置信息。

以上就是初始化配置信息的全部代码，初始化配置信息主要是创建BrokerConfig、NettyServerConfig、NettyClientConfig、MessageStoreConfig配置类，并为这些配置类设置配置的值，同时根据命令行参数判断打印配置信息。

### 创建并初始化Broker控制器

```java
//源代码位置：org.apache.rocketmq.broker.BrokerStartup#createBrokerController
public static BrokerController createBrokerController(String[] args) {

    //省略代码

     //创建BrokerController（broker 控制器）
    final BrokerController controller = new BrokerController(
           brokerConfig,
           nettyServerConfig,
           nettyClientConfig,
           messageStoreConfig);
    // remember all configs to prevent discard
    //将所有的配置信息保存在内存
    controller.getConfiguration().registerConfig(properties);

   //初始化broker控制器
    boolean initResult = controller.initialize();
    //如果初始化失败，则退出
    if (!initResult) {
        controller.shutdown();
         System.exit(-3);
    }

    //省略代码
}
```

创建并初始化Broker控制的代码比较简单，创建以配置类作为参数的BrokerController对象，并将所有的配置信息保存在内容中，方便在其他地方使用；创建完Broker控制器对象以后，对控制器进行初始化，当初始化失败以后，则直接退出程序。

对控制器初始化initialize方法中，到底做了哪些工作？initialize方法主要是加载一些保存在本地的一些配置数据，总结起来做了如下几方面的事情：

- 加载topic配置、消费者位移数据、订阅组数据、消费者过滤配置
- 创建消息相关的组件，并加载消息数据
- 创建netty服务器
- 创建一系列线程
- 注册处理器
- 启动一系列定时任务
- 初始化初始化事务组件
- 初始化acl组件
- 注册RpcHook

### 加载topic配置、消费者位移数据、订阅组数据、消费者过滤配置

```java
//源代码位置：org.apache.rocketmq.broker.BrokerController#initialize
public boolean initialize() throws CloneNotSupportedException {
   //加载topic配置 topics.json
    boolean result = this.topicConfigManager.load();
    //加载消费者位移数据 consumerOffset.json
    result = result && this.consumerOffsetManager.load();
    //加载订阅组数据 subscriptionGroup.json
    result = result && this.subscriptionGroupManager.load();
    //加载消费者过滤 consumerFilter.json
    result = result && this.consumerFilterManager.load();

    //省略代码
}
```

load方法是抽象类ConfigManager的方法，该方法读取文件的内容解码成对应的配置对象，如果文件中的内容为空，就读取备份文件中的内容进行解码。读取的文件都是保存在user.home/store/config/下，user.home是用户目录，不同人的电脑user.home一般不同。topicConfigManager.load()读取topics.json文件，如果该文件的内容为空，那么就读取topics.json.bak文件内容，topics.json保存的是topic数据；同理，consumerOffsetManager.load()方法读取consumerOffset.json和consumerOffset.json.bak文件，保存的是消费者位移数据；subscriptionGroupManager.load()方法读取subscriptionGroup.json和subscriptionGroup.json.bak文件，保存订阅组数据（消费者分组数据）、consumerFilterManager.load()方法读取的是consumerFilter.json和consumerFilter.json.bak的内容，保存的是消费者过滤数据。

### 创建消息相关的组件，并加载消息数据

```java
//源代码位置：org.apache.rocketmq.broker.BrokerController#initialize
public boolean initialize() throws CloneNotSupportedException {

        //省略代码

        //如果上述都加载成功
        if (result) {
            try {
                //创建消息存储器
                this.messageStore = new DefaultMessageStore(this.messageStoreConfig, this.brokerStatsManager,                                this.messageArrivingListener,this.brokerConfig);
                //如果开启了容灾、主从自动切换，添加DLedger角色改变处理器
                if (messageStoreConfig.isEnableDLegerCommitLog()) {
                    DLedgerRoleChangeHandler roleChangeHandler = new DLedgerRoleChangeHandler(this, (DefaultMessageStore) messageStore);
                    ((DLedgerCommitLog)((DefaultMessageStore) messageStore).getCommitLog()).getdLedgerServer().getdLedgerLeaderElector().addRoleChangeHandler(roleChangeHandler);
                }
                //broker 相关统计
                this.brokerStats = new BrokerStats((DefaultMessageStore) this.messageStore);
                //load plugin
                //加载消息存储插件
                MessageStorePluginContext context = new MessageStorePluginContext(messageStoreConfig, brokerStatsManager, messageArrivingListener, brokerConfig);
                this.messageStore = MessageStoreFactory.build(context, this.messageStore);
                this.messageStore.getDispatcherList().addFirst(new CommitLogDispatcherCalcBitMap(this.brokerConfig, this.consumerFilterManager));
            } catch (IOException e) {
                result = false;
                log.error("Failed to initialize", e);
            }
        }

        //加载消息文件
        result = result && this.messageStore.load();

        //省略代码
}
```

如果加载topic配置、消费者位移数据、订阅组数据、消费者过滤配置成功以后，就创建消息相关的组件，并加载消息数据，这个过程创建了消息存储器、DLedger角色改变处理器、Broker统计相关组件以及消息存储插件，然后加载消息文件中的数据。接下来具体看看加载消息文件中的messageStore.load()方法：

```java
//代码位置：org.apache.rocketmq.store.DefaultMessageStore#load
public boolean load() {
        boolean result = true;

        try {
            //判断abort是否存在
            boolean lastExitOK = !this.isTempFileExist();
            log.info("last shutdown {}", lastExitOK ? "normally" : "abnormally");

            //加载定时消费服务器
            if (null != scheduleMessageService) {
                //读取delayOffset.json的内筒
                result = result && this.scheduleMessageService.load();
            }

            // load Commit Log
            //加载 Commit log 文件
            result = result && this.commitLog.load();

            // load Consume Queue
            //加载消费者队列 文件consumequeue
            result = result && this.loadConsumeQueue();

            if (result) {
                //加载检查点文件checkpoint
                this.storeCheckpoint =
                    new StoreCheckpoint(StorePathConfigHelper.getStoreCheckpoint(this.messageStoreConfig.getStorePathRootDir()));

                //加载索引文件
                this.indexService.load(lastExitOK);

                //数据恢复
                this.recover(lastExitOK);

                log.info("load over, and the max phy offset = {}", this.getMaxPhyOffset());
            }
        } catch (Exception e) {
            log.error("load exception", e);
            result = false;
        }

        if (!result) {
            this.allocateMappedFileService.shutdown();
        }

        return result;
}
```

load方法主要逻辑就是加载各种数据文件，主要有以下几方面进行加载数据：

- isTempFileExist方法判断abort是否存在，如果存在，说明Broker是正常关闭的，否则就是异常关闭。this.scheduleMessageService.load()方法读取user.home/store/config/下的delayOffset.json文件的内容，该文件内容保存的his延迟的位移数据。
- commitLog.load()加载 CommitLog 文件, CommitLog 文件保存的是消息内容，每个CommitLog文件大小为1G。this.loadConsumeQueue()方法加载consumequeue目录下的内容，ConsumeQueue（消息消费队列）是消费消息的索引，消费者通过ConsumeQueue可以快速找到查找待消费的消息，consumequeue目录下的文件组织方式是：topic/queueId/fileNmae，所以就可以快速找待消费的消息在哪一个Commit log 文件中。
- indexService.load(lastExitOK)加载索引文件，加载的是user.home/store/index/目录下文件，文件名fileName是以创建时的时间戳命名的，所以可以通过时间区间来快速查询消息，IndexFile的底层存储设计为在文件系统中实现HashMap结构，故底层实现为hash索引。
- this.recover(lastExitOK)方法将CommitLog 文件的内容加载到内存中以及topic队列。

### 创建netty服务器

```java
//源代码位置：org.apache.rocketmq.broker.BrokerController#initialize
public boolean initialize() throws CloneNotSupportedException {

    //省略代码
    if (result) {

        //创建netty远程服务器
        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.clientHousekeepingService);
         NettyServerConfig fastConfig = (NettyServerConfig) this.nettyServerConfig.clone();
         fastConfig.setListenPort(nettyServerConfig.getListenPort() - 2);
         this.fastRemotingServer = new NettyRemotingServer(fastConfig, this.clientHousekeepingService);

        //省略代码
    }

     //省略代码
}
```

创建netty服务器的时候创建了两个，一个是普通的，一个是快速的，remotingServer用来与生产者、消费者进行通信。当isSendMessageWithVIPChannel=true的时候会选择port-2的fastRemotingServer进行的消息的处理，为了防止某些很重要的业务阻塞，就再开启了一个remotingServer进行处理，但是现在默认是不开启的，fastRemotingServer主要是为了兼容老版本的RocketMQ.。

### 创建一系列线程

```java
//源代码位置：org.apache.rocketmq.broker.BrokerController#initialize
public boolean initialize() throws CloneNotSupportedException {

            //代码省略

            //发送消息线程池
            this.sendMessageExecutor = new BrokerFixedThreadPoolExecutor(
                this.brokerConfig.getSendMessageThreadPoolNums(),
                this.brokerConfig.getSendMessageThreadPoolNums(),
                1000 * 60,
                TimeUnit.MILLISECONDS,
                this.sendThreadPoolQueue,
                new ThreadFactoryImpl("SendMessageThread_"));

            //拉取消息线程池
            //this.pullMessageExecutor 

            //回复消息线程池
            //this.replyMessageExecutor 

            //查询消息线程池
            //this.queryMessageExecutor 

            //broker 管理线程池
            //this.adminBrokerExecutor

            //客户端管理线程池
            //this.clientManageExecutor 

            //心跳线程池
            //this.heartbeatExecutor 
            //事务线程池
           // this.endTransactionExecutor 

            //消费者管理线程池
            this.consumerManageExecutor =
                Executors.newFixedThreadPool(this.brokerConfig.getConsumerManageThreadPoolNums(), new ThreadFactoryImpl(
                    "ConsumerManageThread_"));

            //代码省略
}
```

创建一系列线程的逻辑比较简单，就是直接new一个线程池对象，这段代码只保存首尾完整的代码，中间的代码跟首尾差不多，所有就写了注释。创建的线程池对象有发送消息线程池、拉取消息线程池、回复消息线程池、查询消息线程池、broker 管理线程池、客户端管理线程池、心跳线程池、事务线程池、消费者管理线程池。

```java
//源代码位置：org.apache.rocketmq.broker.BrokerController#initialize
public boolean initialize() throws CloneNotSupportedException {
    //省略代码

    //注册处理器
     this.registerProcessor();

    //省略代码
}
```

注册处理器只有一行代码：this.registerProcessor()，逻辑都在registerProcessor()方法中，registerProcessor()方法如下：

```java
//源代码位置：org.apache.rocketmq.broker.BrokerController#registerProcessor
public void registerProcessor() {

        //发送消息处理器
        SendMessageProcessor sendProcessor = new SendMessageProcessor(this);
        sendProcessor.registerSendMessageHook(sendMessageHookList);
        sendProcessor.registerConsumeMessageHook(consumeMessageHookList);

        //远程服务注册发送消息处理器
        this.remotingServer.registerProcessor(RequestCode.SEND_MESSAGE, sendProcessor, this.sendMessageExecutor);
        this.remotingServer.registerProcessor(RequestCode.SEND_MESSAGE_V2, sendProcessor, this.sendMessageExecutor);
        this.remotingServer.registerProcessor(RequestCode.SEND_BATCH_MESSAGE, sendProcessor, this.sendMessageExecutor);
        this.remotingServer.registerProcessor(RequestCode.CONSUMER_SEND_MSG_BACK, sendProcessor, this.sendMessageExecutor);
        this.fastRemotingServer.registerProcessor(RequestCode.SEND_MESSAGE, sendProcessor, this.sendMessageExecutor);
        this.fastRemotingServer.registerProcessor(RequestCode.SEND_MESSAGE_V2, sendProcessor, this.sendMessageExecutor);
        this.fastRemotingServer.registerProcessor(RequestCode.SEND_BATCH_MESSAGE, sendProcessor, this.sendMessageExecutor);
        this.fastRemotingServer.registerProcessor(RequestCode.CONSUMER_SEND_MSG_BACK, sendProcessor, this.sendMessageExecutor);

        //注册拉消息处理器
        //注册回复消息处理器
        //注册查询消息处理器
        //注册客户端管理处理器
        //注册消费者管理处理器
        //注册事务处理器

         //注册broker处理器
        AdminBrokerProcessor adminProcessor = new AdminBrokerProcessor(this);
        this.remotingServer.registerDefaultProcessor(adminProcessor, this.adminBrokerExecutor);
        this.fastRemotingServer.registerDefaultProcessor(adminProcessor, this.adminBrokerExecutor);
}
```

上述代码只将registerProcessor方法的首尾代码写出来，中间的代码省略了，只用注释说明被省略了的代码的作用。registerProcessor方法注册了发送消息处理器、远程服务注册发送消息处理器、拉消息处理器、回复消息处理器、查询消息处理器、客户端管理处理器、消费者管理处理器、事务处理器、broker处理器。registerProcessor注册方法也很简单，就是以RequestCode作为key，以Pair<处理器，线程池>作为Value保存在名字为processorTable的HashMap中。每个请求都是在线程池中处理的，这样可以提高处理请求的性能。对于每个传入的请求，根据RequestCode就可以在processorTable查找处理器来处理请求。每个处理器都有有一个processRequest方法进行处理请求。

### 启动一系列定时任务

Broker初始化方法initialize中，会启动一系列的后台定时线程任务，这些后台任务包括都是由scheduledExecutorService线程池执行的，scheduledExecutorService是单线程线程池（ Executors.newSingleThreadScheduledExecutor()），只用单线程线程池执行后台定时任务有一个好处就是减少线程过多，反而导致线程为了抢占CPU加剧了竞争。这一些后台定时线程任务如下：

- 每24小时打印昨天产生了多少消息，消费了多少消息
- 每五秒保存消费者位移到文件中
- 每10秒保存消费者过滤到文件中
- 每3分钟定时检测消费的进度
- 每秒打印队列的大小以及队列头部元素存在的时间
- 每分钟打印已存储在CommitLog中但尚未分派到消费队列的字节数
- 每两分钟定时获取获取name server 地址
- 每分钟定时打印slave 数据同步落后多少

```java
//源代码位置：org.apache.rocketmq.broker.BrokerController#initialize
public boolean initialize() throws CloneNotSupportedException {
    //省略代码
    final long period = 1000 * 60 * 60 * 24;
    //每24小时打印昨天产生了多少消息，消费了多少消息
     this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
              try {
                   BrokerController.this.getBrokerStats().record();
               } catch (Throwable e) {
                    log.error("schedule record error.", e);
               }
             }
      }, initialDelay, period, TimeUnit.MILLISECONDS);
    //省略代码
}
```

每24小时打印昨天产生了多少消息，消费了多少消息的定时任务比较简单，就是将昨天消息的生产和消费的数量统计出来，然后把这两个指标打印出来。

```java
//源代码位置：org.apache.rocketmq.broker.BrokerController#initialize
public boolean initialize() throws CloneNotSupportedException {
    //省略代码
    //每五秒保存消费者位移
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
       @Override
       public void run() {
           try {
                  BrokerController.this.consumerOffsetManager.persist();
            } catch (Throwable e) {
                  log.error("schedule persist consumerOffset error.", e);
            }
        }
    }, 1000 * 10, this.brokerConfig.getFlushConsumerOffsetInterval(), TimeUnit.MILLISECONDS);

    //每10秒保存消费者过滤
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
         @Override
        public void run() {
          try {
              BrokerController.this.consumerFilterManager.persist();
          } catch (Throwable e) {
               log.error("schedule persist consumer filter error.", e);
          }
        }
    }, 1000 * 10, 1000 * 10, TimeUnit.MILLISECONDS);
    //省略代码
}
```

每五秒保存消费者位移和每10秒保存消费者过滤定时任务都是保存在文件中，每五秒保存消费者位移定时任务将消费者位移保存在consumerOffset.json文件中，每10秒保存消费者过滤定时任务将消费者过滤保存在consumerFilter.json文件中。

```java
//源代码位置：org.apache.rocketmq.broker.BrokerController#initialize
public boolean initialize() throws CloneNotSupportedException {
    //省略代码
    //每3分钟定时检测消费的进度
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
           @Override
           public void run() {
                try {
                    BrokerController.this.protectBroker();
                } catch (Throwable e) {
                    log.error("protectBroker error.", e);
                }
            }
     }, 3, 3, TimeUnit.MINUTES);
    //省略代码
}
```

每3分钟定时检测消费的进度定时任务的作用是检测消费者的消费进度，当消费者消费消息的进度落后消费者落后阈值的时候，就停止消费者消费，具体的实现看protectBroker的源码：

```java
//源代码位置：org.apache.rocketmq.broker.BrokerController#protectBroker
public void protectBroker() {
        //是否开启慢消费检测开关
        if (this.brokerConfig.isDisableConsumeIfConsumerReadSlowly()) {
            //遍历统计项
            final Iterator<Map.Entry<String, MomentStatsItem>> it = this.brokerStatsManager.getMomentStatsItemSetFallSize().getStatsItemTable().entrySet().iterator();
            while (it.hasNext()) {
                final Map.Entry<String, MomentStatsItem> next = it.next();
                final long fallBehindBytes = next.getValue().getValue().get();
                //消费者消费消息的进度落后消费者落后阈值
                if (fallBehindBytes > this.brokerConfig.getConsumerFallbehindThreshold()) {
                    final String[] split = next.getValue().getStatsKey().split("@");
                    final String group = split[2];
                    LOG_PROTECTION.info("[PROTECT_BROKER] the consumer[{}] consume slowly, {} bytes, disable it", group, fallBehindBytes);
                    //设置消费者消费的标志，关闭消费
                    this.subscriptionGroupManager.disableConsume(group);
                }
            }
        }
}
```

protectBroker方法首先判别是否开启慢消费检测开关，如果开启了，就进行遍历统计项，判断消费者消费消息的进度落后消费者落后阈值的时候，就停止该消费者停止消费来保护broker，如果消费者消费比较慢，那么在Broker的消费会越来越多，积压在Broker上，所以停止该慢消费者消费消息，让其他消费者消费，减少消息的积压。

```java
//源代码位置：org.apache.rocketmq.broker.BrokerController#initialize
public boolean initialize() throws CloneNotSupportedException {
    //代码省略
    //每秒打印队列的大小以及队列头部元素存在的时间
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
         @Override
         public void run() {
             try {
                  BrokerController.this.printWaterMark();
              } catch (Throwable e) {
                   log.error("printWaterMark error.", e);
             }
        }
     }, 10, 1, TimeUnit.SECONDS);

    //代码省略
}
```

每秒打印队列的大小以及队列头部元素存在的时间定时任务，会打印发送消息线程池队列、拉取消息线程池队列、查询消息线程池队列、结束事务线程池队列的大小，以及打印队列头部元素存在的时间，这个时间等于当前时间减去头部元素创建的时间，就是该元素创建到现在已经花费了多长时间。具体的代码如下：

```java
//源代码位置：org.apache.rocketmq.broker.BrokerController#headSlowTimeMills
public long headSlowTimeMills(BlockingQueue<Runnable> q) {
        long slowTimeMills = 0;
        //队列的头
        final Runnable peek = q.peek();
        if (peek != null) {
            RequestTask rt = BrokerFastFailure.castRunnable(peek);
            //当前时间减去创建时间
            slowTimeMills = rt == null ? 0 : this.messageStore.now() - rt.getCreateTimestamp();
        }

        if (slowTimeMills < 0) {
            slowTimeMills = 0;
        }

        return slowTimeMills;
}
```

每分钟打印已存储在CommitLog中但尚未分派到消费队列的字节数、每两分钟定时获取获取name server 地址、每分钟定时打印slave 数据同步落后多少，这三个定时任务比较简单，都不把源码写上，读者可以自己深入分析。

### 初始化事务消息

```java
//源码位置：org.apache.rocketmq.broker.BrokerController#initialTransaction
private void initialTransaction() {
        //加载transactionalMessageService，利用spi
        this.transactionalMessageService = ServiceProvider.loadClass(ServiceProvider.TRANSACTION_SERVICE_ID, TransactionalMessageService.class);
        if (null == this.transactionalMessageService) {
            this.transactionalMessageService = new TransactionalMessageServiceImpl(new TransactionalMessageBridge(this, this.getMessageStore()));
            log.warn("Load default transaction message hook service: {}", TransactionalMessageServiceImpl.class.getSimpleName());
        }
        //创建transactionalMessage检查监听器
        this.transactionalMessageCheckListener = ServiceProvider.loadClass(ServiceProvider.TRANSACTION_LISTENER_ID, AbstractTransactionalMessageCheckListener.class);
        if (null == this.transactionalMessageCheckListener) {
            this.transactionalMessageCheckListener = new DefaultTransactionalMessageCheckListener();
            log.warn("Load default discard message hook service: {}", DefaultTransactionalMessageCheckListener.class.getSimpleName());
        }
        this.transactionalMessageCheckListener.setBrokerController(this);
        //创建事务消息检查服务
        this.transactionalMessageCheckService = new TransactionalMessageCheckService(this);
 }
```

initialTransaction方法主要创建与事务消息相关的类，创建transactionalMessageService（事务消息服务）、transactionalMessageCheckListener（事务消息检查监听器）、transactionalMessageCheckService（事务消息检查服务）。transactionalMessageService用于处理事务消息，transactionalMessageCheckListener主要用来回查消息监听，transactionalMessageCheckService用于检查超时的 Half 消息是否需要回查。RocketMQ发送事务消息是将消费先写入到事务相关的topic的中，这个消息就称为半消息，当本地事务成功执行，那么半消息会还原为原来的消息，然后再进行保存。initialTransaction在创建transactionalMessageService和transactionalMessageCheckListener都使用了ServiceProvider.loadClass方法，这个方法就是采用jSPI原理，SPI原理就是利用反射加载META-INF/service目录下的某个接口的所有实现，只要实现接口，然后META-INF/service目录下添加文件名为全类名的文件，这样SPI就可以加载具体的实现类，具有可拓展性。

### 初始化acl组件

```java
//源码位置：org.apache.rocketmq.broker.BrokerController#initialAcl
private void initialAcl() {
        if (!this.brokerConfig.isAclEnable()) {
            log.info("The broker dose not enable acl");
            return;
        }

        //利用SPI加载权限相关的校验器
        List<AccessValidator> accessValidators = ServiceProvider.load(ServiceProvider.ACL_VALIDATOR_ID, AccessValidator.class);
        if (accessValidators == null || accessValidators.isEmpty()) {
            log.info("The broker dose not load the AccessValidator");
            return;
        }

        //将所有的权限校验器进行缓存以及注册
        for (AccessValidator accessValidator: accessValidators) {
            final AccessValidator validator = accessValidator;
            accessValidatorMap.put(validator.getClass(),validator);
            this.registerServerRPCHook(new RPCHook() {

                @Override
                public void doBeforeRequest(String remoteAddr, RemotingCommand request) {
                    //Do not catch the exception
                    validator.validate(validator.parse(request, remoteAddr));
                }

                @Override
                public void doAfterResponse(String remoteAddr, RemotingCommand request, RemotingCommand response) {
                }
            });
        }
    }
```

initialAcl方法主要是加载权限相关校验器，RocketMQ的相关的管理的权限验证和安全就交给这里的加载的校验器了。initialAcl方法也利用SPI原理加载接口的具体实现类，将所有加载的校验器缓存在map中，然后再注册RPC钩子，在请求之前调用校验器的validate的方法。

### 注册RpcHook

```java
//源码位置：org.apache.rocketmq.broker.BrokerController#initialRpcHooks
private void initialRpcHooks() {

      //利用SPI加载钩子
      List<RPCHook> rpcHooks = ServiceProvider.load(ServiceProvider.RPC_HOOK_ID, RPCHook.class);
      if (rpcHooks == null || rpcHooks.isEmpty()) {
           return;
       }
        //注册钩子
       for (RPCHook rpcHook: rpcHooks) {
           this.registerServerRPCHook(rpcHook);
       }
}
```

initialRpcHooks方法加RPC钩子，利用SPI原理加载具体的钩子实现，然后将所有的钩子进行注册，钩子的注册是将钩子保存在List中。

以上分析就是创建Broker控制器的全过程，这个过程首先进行一些必要的初始化配置，如Broker配置、网络通信Neety配置以及存储相关配置等。然后在创建并初始化Broker控制器，创建并初始化Broker控制器的过程中，又进行了多个步骤，如加载topic配置、消费者位移数据、启动一系列后台定时任务、创建事务消息相关组件等。分析完Broker控制器的创建。接下来分析Broker控制器的启动。

## Broker控制器的启动

```java
//源码位置：org.apache.rocketmq.broker.BrokerController#start
public static BrokerController start(BrokerController controller) {
        try {

            //Broker控制器启动
            controller.start();

            //打印Broker成功的消息
            String tip = "The broker[" + controller.getBrokerConfig().getBrokerName() + ", "
                + controller.getBrokerAddr() + "] boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();

            if (null != controller.getBrokerConfig().getNamesrvAddr()) {
                tip += " and name server is " + controller.getBrokerConfig().getNamesrvAddr();
            }

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

start方法的逻辑比较简单，首先启动Broker控制器，然后打印成功启动Broker控制器的日志。Broker控制器启动的逻辑主要在controller.start()中，接下来，分析下controller.start()方法的作用，controller.start()方法主要是启动各种组件：

- 启动消息消息存储器
- netty服务的启动
- 文件监听器启动
- broker 对外api启动
- 长轮询拉取消息服务启动
- 客户端长连接服务启动
- 过滤服务管理启动
- broker 相关统计启动
- broker 快速失败启动

```java
//源码位置：org.apache.rocketmq.broker.BrokerController#start
public void start() throws Exception {
        if (this.messageStore != null) {
            //启动消息消息存储
            this.messageStore.start();
        }

        if (this.remotingServer != null) {
            //netty服务的启动
            this.remotingServer.start();
        }

        if (this.fastRemotingServer != null) {
            this.fastRemotingServer.start();
        }

        //文件改变监听启动
        if (this.fileWatchService != null) {
            this.fileWatchService.start();
        }

        //broker 对外api启动
        if (this.brokerOuterAPI != null) {
            this.brokerOuterAPI.start();
        }

        //长轮询拉取消息服务启动
        if (this.pullRequestHoldService != null) {
            this.pullRequestHoldService.start();
        }

        //客户端长连接服务启动
        if (this.clientHousekeepingService != null) {
            this.clientHousekeepingService.start();
        }

        //过滤服务管理启动
        if (this.filterServerManager != null) {
            this.filterServerManager.start();
        }

        //如果没有采用主从切换（多副本）
        if (!messageStoreConfig.isEnableDLegerCommitLog()) {
            startProcessorByHa(messageStoreConfig.getBrokerRole());
            handleSlaveSynchronize(messageStoreConfig.getBrokerRole());
            this.registerBrokerAll(true, false, true);
        }

        //定时注册broker
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
                } catch (Throwable e) {
                    log.error("registerBrokerAll Exception", e);
                }
            }
        }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);

        //broker 相关统计启动
        if (this.brokerStatsManager != null) {
            this.brokerStatsManager.start();
        }

        //broker 快速失败启动
        if (this.brokerFastFailure != null) {
            this.brokerFastFailure.start();
        }
 }
```

总体而言，Broker的启动过程还是比较复杂的，启动过程可以分为两个部分，创建Broker控制器和启动Broker控制器。创建Broker控制器的过程中。初始化配置信息、创建各种组件、创建和启动一些后台线程服务、以及初始化各种组件；启动Broker控制的过程就是各种组件的启动，另外还启动定时注册Broker的任务。从宏观的角度大体分析了Broker的启动过程，还有很多细节没有进行深入，这些细节的深入将在后续的源码分析中体现。