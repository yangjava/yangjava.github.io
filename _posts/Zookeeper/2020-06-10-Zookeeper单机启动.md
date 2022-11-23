---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---
# Zookeeper源码-单机启动
Zookeeper启动时，首先解析配置文件，根据配置文件选择启动单例还是集群模式。单机环境启动入口为ZooKeeperServerMain类。


## 入口方法

```java
org.apache.zookeeper.server.quorum.QuorumPeerMain#main
```

## 初始化QuorumPeerMain

初始化**QuorumPeerMain**对象，并执行initializeAndRun方法。

```
public static void main(String[] args) {
        QuorumPeerMain main = new QuorumPeerMain();
        try {
            main.initializeAndRun(args);
        } catch (IllegalArgumentException e) {
            LOG.error("Invalid arguments, exiting abnormally", e);
            LOG.info(USAGE);
            System.err.println(USAGE);
            System.exit(2);
        } catch (ConfigException e) {
            LOG.error("Invalid config, exiting abnormally", e);
            System.err.println("Invalid config, exiting abnormally");
            System.exit(2);
        } catch (DatadirException e) {
            LOG.error("Unable to access datadir, exiting abnormally", e);
            System.err.println("Unable to access datadir, exiting abnormally");
            System.exit(3);
        } catch (AdminServerException e) {
            LOG.error("Unable to start AdminServer, exiting abnormally", e);
            System.err.println("Unable to start AdminServer, exiting abnormally");
            System.exit(4);
        } catch (Exception e) {
            LOG.error("Unexpected exception, exiting abnormally", e);
            System.exit(1);
        }
        LOG.info("Exiting normally");
        System.exit(0);
    }
```
QuorumPeerMain.main()接受至少一个参数，一般就一个参数，参数为zoo.cfg文件路径。main方法中没有很多的业务代码，实例化了一个QuorumPeerMain 对象，然后main.initializeAndRun(args)进行了实例化


```
protected void initializeAndRun(String[] args)
        throws ConfigException, IOException, AdminServerException
    {
        QuorumPeerConfig config = new QuorumPeerConfig();
        if (args.length == 1) {
            config.parse(args[0]);
        }

        // Start and schedule the the purge task
        DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                .getDataDir(), config.getDataLogDir(), config
                .getSnapRetainCount(), config.getPurgeInterval());
        purgeMgr.start();

        // 当配置了多节点信息，return quorumVerifier!=null && (!standaloneEnabled || quorumVerifier.getVotingMembers().size() > 1);
        if (args.length == 1 && config.isDistributed()) {
            // 集群模式
            runFromConfig(config);
        } else {
            LOG.warn("Either no config or no quorum defined in config, running "
                    + " in standalone mode");
            // there is only server in the quorum -- run as standalone
            // 单机模式
            ZooKeeperServerMain.main(args);
        }
    }
```

## 配置解析

配置解析主要有两种情况
1. 使用配置文件
2. 使用命令行参数

### 配置文件解析

配置文件解析通过`org.apache.zookeeper.server.quorum.QuorumPeerConfig#parse`解析

```java

```



1. 先校验文件的合法性
2. 配置文件是使用Java的properties形式写的，所以可以通过Properties.load来解析
3. 将解析出来的key、value赋值给对应的配置



### 命令行参数解析



## 

## Zookeeper启动

### main方法入口

zookeeper一般使用命令工具启动，启动主要就是初始化所有组件，让server可以接收并处理来自client的请求。

我们一般使用命令行工具来部署zk server，zkServer.sh，这个脚本用来启动停止server，通过不同的参数和选项来达到不同的功能。该脚本最后会通过Java执行下面的main方法

```java
org.apache.zookeeper.server.quorum.QuorumPeerMain#main
```




不管单机还是集群都是使用`zkServer.sh`这个脚本来启动，只是参数不同，所以main方法入口也是一样的。所以这个入口方法主要是根据不同的入参判断是集群启动还是单机启动。

```
ZOOMAIN="org.apache.zookeeper.server.quorum.QuorumPeerMain"
#.......
nohup "$JAVA" $ZOO_DATADIR_AUTOCREATE "-Dzookeeper.log.dir=${ZOO_LOG_DIR}" \
    "-Dzookeeper.log.file=${ZOO_LOG_FILE}" "-Dzookeeper.root.logger=${ZOO_LOG4J_PROP}" \
    -XX:+HeapDumpOnOutOfMemoryError -XX:OnOutOfMemoryError='kill -9 %p' \
    -cp "$CLASSPATH" $JVMFLAGS $ZOOMAIN "$ZOOCFG" > "$_ZOO_DAEMON_OUT" 2>&1 < /dev/null &
    if [ $? -eq 0 ]  #.......
```

该main方法主要做了以下几件事

1. 解析配置，如果传入的是配置文件(参数只有一个)，解析配置文件并初始化QuorumPeerConfig
2. 启动清理文件的线程
3. 判断是单机还是集群
    1. 集群：只有一个参数，并且配置了多个server
    2. 单机：上面的条件不满足，一般在启动的使用了以下两种配置的一种
        1. 使用的是文件配置，但是没有配置多台server
        2. 命令行配置多个（2-4）参数：port dataDir [tickTime] [maxClientCnxns]

### Zookeeper启动流程

![Zookeeper启动流程](png\zookeeper\Zookeeper启动流程.png)

Zookeeper启动时，首先解析配置文件，根据配置文件选择启动单例还是集群模式。集群模式启动，首先从磁盘中加载数据到内存数据树DataTree， 并添加committed交易日志到DataTree中。然后启动ServerCnxnFactory,监听客户端的请求。实际上是启动了一个基于Netty服务，客户端发送的数据，交由NettyServerCnxn处理，NettyServerCnxn数据包的处理，实际委托给ZooKeeperServer。

### 配置解析

配置解析主要有两种情况

1. 使用配置文件
2. 使用命令行参数

#### 使用配置文件






使用配置文件的时候是使用`QuorumPeerConfig`来解析配置的

1. 先校验文件的合法性
2. 配置文件是使用Java的properties形式写的，所以可以通过Properties.load来解析
3. 将解析出来的key、value赋值给对应的配置

#### 使用命令行参数

直接在命令指定对应的配置，这种情况只有在单机的时候才会使用，包含以下几个参数

- port，必填，sever监听的端口
- dataDir，必填，数据所在的目录
- tickTime，选填
- maxClientCnxns，选填，最多可处理的客户端连接数

```
public static void main(String[] args) {
        QuorumPeerMain main = new QuorumPeerMain();
        try {
            main.initializeAndRun(args);
        } catch (IllegalArgumentException e) {
            LOG.error("Invalid arguments, exiting abnormally", e);
            LOG.info(USAGE);
            System.err.println(USAGE);
            System.exit(2);
        } catch (ConfigException e) {
            LOG.error("Invalid config, exiting abnormally", e);
            System.err.println("Invalid config, exiting abnormally");
            System.exit(2);
        } catch (DatadirException e) {
            LOG.error("Unable to access datadir, exiting abnormally", e);
            System.err.println("Unable to access datadir, exiting abnormally");
            System.exit(3);
        } catch (AdminServerException e) {
            LOG.error("Unable to start AdminServer, exiting abnormally", e);
            System.err.println("Unable to start AdminServer, exiting abnormally");
            System.exit(4);
        } catch (Exception e) {
            LOG.error("Unexpected exception, exiting abnormally", e);
            System.exit(1);
        }
        LOG.info("Exiting normally");
        System.exit(0);
    }
```

QuorumPeerMain.main()接受至少一个参数，一般就一个参数，参数为zoo.cfg文件路径。main方法中没有很多的业务代码，实例化了一个QuorumPeerMain 对象，然后main.initializeAndRun(args)进行了实例化

```
protected void initializeAndRun(String[] args)
        throws ConfigException, IOException, AdminServerException
    {
        QuorumPeerConfig config = new QuorumPeerConfig();
        if (args.length == 1) {
            config.parse(args[0]);
        }

        // Start and schedule the the purge task
        DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                .getDataDir(), config.getDataLogDir(), config
                .getSnapRetainCount(), config.getPurgeInterval());
        purgeMgr.start();

        // 当配置了多节点信息，return quorumVerifier!=null && (!standaloneEnabled || quorumVerifier.getVotingMembers().size() > 1);
        if (args.length == 1 && config.isDistributed()) {
            // 集群模式
            runFromConfig(config);
        } else {
            LOG.warn("Either no config or no quorum defined in config, running "
                    + " in standalone mode");
            // there is only server in the quorum -- run as standalone
            // 单机模式
            ZooKeeperServerMain.main(args);
        }
    }
```

initializeAndRun方法则通过实例化QuorumPeerConfig对象，通过parseProperties()来解析zoo.cfg文件中的配置，QuorumPeerConfig包含了Zookeeper整个应用的配置属性。接着开启一个DatadirCleanupManager对象来开启一个Timer用于清除并创建管理新的DataDir相关的数据。

最后进行程序的启动，因为Zookeeper分为单机和集群模式，所以分为两种不同的启动方式，当zoo.cfg文件中配置了standaloneEnabled=true为单机模式，如果配置server.0,server.1......集群节点，则为集群模式.

### 单机模式启动

当配置了standaloneEnabled=true 或者没有配置集群节点（sever.*）时，Zookeeper使用单机环境启动。单机环境启动入口为ZooKeeperServerMain类，ZooKeeperServerMain类中持有ServerCnxnFactory、ContainerManager和AdminServer对象;

```
public class ZooKeeperServerMain {
    /*.............*/
    // ZooKeeper server supports two kinds of connection: unencrypted and encrypted.
    private ServerCnxnFactory cnxnFactory;
    private ServerCnxnFactory secureCnxnFactory;
    private ContainerManager containerManager;

    private AdminServer adminServer;
    /*.............*/
}
```

ServerCnxnFactory为Zookeeper中的核心组件，用于网络通信IO的实现和管理客户端连接，Zookeeper内部提供了两种实现，一种是基于JDK的NIO实现，一种是基于netty的实现。

​     **ContainerManager**类，用于管理维护Zookeeper中节点Znode的信息，管理zkDatabase；

**AdminServer**是一个Jetty服务，默认开启8080端口，用于提供Zookeeper的信息的查询接口。该功能从3.5的版本开始。

**ZooKeeperServerMain**的main方法中同QuorumPeerMain中一致，先实例化本身的对象，再进行init，加载配置文件，然后启动。

#### 组件启动

zookeeper包含的主要组件有

- FileTxnSnapLog：管理FileTxLog和FileSnap
- ZooKeeperServer：维护一个处理器链表processor chain
- NIOServerCnxnFactory：管理来自客户端的连接
- Jetty，用来通过http管理zk

zookeeper维护了自己的数据结构和物理文件，而且要接收并处理client发送来的网络请求，所以在zookeeper启动的时候，要做好下面的准备工作

1. 初始化FileTxnSnapLog，创建了FileTxnLog实例和FIleSnap实例，并保存刚启动时候DataTree的snapshot
2. 初始化ZooKeeperServer 对象；
3. 实例化CountDownLatch线程计数器对象，在程序启动后，执行shutdownLatch.await();用于挂起主程序，并监听Zookeeper运行状态。
4. 创建adminServer（Jetty）服务并开启。
5. 创建ServerCnxnFactory对象，cnxnFactory = ServerCnxnFactory.createFactory(); Zookeeper默认使用NIOServerCnxnFactory来实现网络通信IO。
    1. 从解析出的配置中配置NIOServerCnxnFactory
    2. 初始化网络连接管理类:NIOServerCnxnFactory
        1. 初始化：WorkerService：用来业务处理的线程池
        2. 线程启动：
           SelectorThread（有多个）：处理网络请求，write和read
           AcceptThread：用来接收连接请求，建立连接，zk也支持使用reactor多线程，accept线程用来建立连接，selector线程用来处理read、write
           ConnectionExpirerThread：关闭超时的连接，所有的session都放在`org.apache.zookeeper.server.ExpiryQueue#expiryMap`里面维护，这个线程不断从里面拿出超时的连接关闭
    3. 启动ZookeeperServer，主要是用来创建SessionTrackerImpl，这个类是用来管理session的

#### 	启动单机模式

```
// 解析单机模式的配置对象，并启动单机模式
    protected void initializeAndRun(String[] args)
        throws ConfigException, IOException, AdminServerException
    {
        try {

            //注册jmx
           // JMX的全称为Java Management Extensions.是管理Java的一种扩展。
           // 这种机制可以方便的管理、监控正在运行中的Java程序。常用于管理线程，内存，日志Level，服务重启，系统环境等
            ManagedUtil.registerLog4jMBeans();
        } catch (JMException e) {
            LOG.warn("Unable to register log4j JMX control", e);
        }

        // 创建服务配置对象
        ServerConfig config = new ServerConfig();

        //如果入参只有一个，则认为是配置文件的路径
        if (args.length == 1) {
            // 解析配置文件
            config.parse(args[0]);
        } else {
            // 参数有多个，解析参数
            config.parse(args);
        }

        // 根据配置运行服务
        runFromConfig(config);
    }
```

服务启动

```
public void runFromConfig(ServerConfig config)
            throws IOException, AdminServerException {
        LOG.info("Starting server");
        FileTxnSnapLog txnLog = null;
        try {
            // Note that this thread isn't going to be doing anything else,
            // so rather than spawning another thread, we will just call
            // run() in this thread.
            // create a file logger url from the command line args
            //初始化日志文件
            txnLog = new FileTxnSnapLog(config.dataLogDir, config.dataDir);

           // 初始化zkServer对象
            final ZooKeeperServer zkServer = new ZooKeeperServer(txnLog,
                    config.tickTime, config.minSessionTimeout, config.maxSessionTimeout, null);

            // 服务结束钩子，用于知道服务器错误或关闭状态更改。
            final CountDownLatch shutdownLatch = new CountDownLatch(1);
            zkServer.registerServerShutdownHandler(
                    new ZooKeeperServerShutdownHandler(shutdownLatch));


            // Start Admin server
            // 创建admin服务，用于接收请求(创建jetty服务)
            adminServer = AdminServerFactory.createAdminServer();
            // 设置zookeeper服务
            adminServer.setZooKeeperServer(zkServer);
            // AdminServer是3.5.0之后支持的特性,启动了一个jettyserver,默认端口是8080,访问此端口可以获取Zookeeper运行时的相关信息
            adminServer.start();

            boolean needStartZKServer = true;


            //---启动ZooKeeperServer
            //判断配置文件中 clientportAddress是否为null
            if (config.getClientPortAddress() != null) {
                //ServerCnxnFactory是Zookeeper中的重要组件,负责处理客户端与服务器的连接
                //初始化server端IO对象，默认是NIOServerCnxnFactory:Java原生NIO处理网络IO事件
                cnxnFactory = ServerCnxnFactory.createFactory();

                //初始化配置信息
                cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), false);

                //启动服务:此方法除了启动ServerCnxnFactory,还会启动ZooKeeper
                cnxnFactory.startup(zkServer);
                // zkServer has been started. So we don't need to start it again in secureCnxnFactory.
                needStartZKServer = false;
            }
            if (config.getSecureClientPortAddress() != null) {
                secureCnxnFactory = ServerCnxnFactory.createFactory();
                secureCnxnFactory.configure(config.getSecureClientPortAddress(), config.getMaxClientCnxns(), true);
                secureCnxnFactory.startup(zkServer, needStartZKServer);
            }

            // 定时清除容器节点
            //container ZNodes是3.6版本之后新增的节点类型，Container类型的节点会在它没有子节点时
            // 被删除（新创建的Container节点除外），该类就是用来周期性的进行检查清理工作
            containerManager = new ContainerManager(zkServer.getZKDatabase(), zkServer.firstProcessor,
                    Integer.getInteger("znode.container.checkIntervalMs", (int) TimeUnit.MINUTES.toMillis(1)),
                    Integer.getInteger("znode.container.maxPerMinute", 10000)
            );
            containerManager.start();

            // Watch status of ZooKeeper server. It will do a graceful shutdown
            // if the server is not running or hits an internal error.

            // ZooKeeperServerShutdownHandler处理逻辑，只有在服务运行不正常的情况下，才会往下执行
            shutdownLatch.await();

            // 关闭服务
            shutdown();

            if (cnxnFactory != null) {
                cnxnFactory.join();
            }
            if (secureCnxnFactory != null) {
                secureCnxnFactory.join();
            }
            if (zkServer.canShutdown()) {
                zkServer.shutdown(true);
            }
        } catch (InterruptedException e) {
            // warn, but generally this is ok
            LOG.warn("Server interrupted", e);
        } finally {
            if (txnLog != null) {
                txnLog.close();
            }
        }
    }
```

Zookeeper中 ServerCnxnFactory默认采用了NIOServerCnxnFactory来实现，也可以通过配置系统属性zookeeper.serverCnxnFactory 来设置使用Netty实现；

```
static public ServerCnxnFactory createFactory() throws IOException {
        String serverCnxnFactoryName =
            System.getProperty(ZOOKEEPER_SERVER_CNXN_FACTORY);
        if (serverCnxnFactoryName == null) {
            //如果未指定实现类，默认使用NIOServerCnxnFactory
            serverCnxnFactoryName = NIOServerCnxnFactory.class.getName();
        }
        try {
            ServerCnxnFactory serverCnxnFactory = (ServerCnxnFactory) Class.forName(serverCnxnFactoryName)
                    .getDeclaredConstructor().newInstance();
            LOG.info("Using {} as server connection factory", serverCnxnFactoryName);
            return serverCnxnFactory;
        } catch (Exception e) {
            IOException ioe = new IOException("Couldn't instantiate "
                    + serverCnxnFactoryName);
            ioe.initCause(e);
            throw ioe;
        }
    }
```

cnxnFactory.startup(zkServer);方法启动了ServerCnxnFactory ，同时启动ZooKeeper服务

```
public void startup(ZooKeeperServer zks, boolean startServer)
            throws IOException, InterruptedException {
        // 启动相关线程
        //开启NIOWorker线程池，
        //启动NIO Selector线程
        //启动客户端连接处理acceptThread线程
        start();
        setZooKeeperServer(zks);

        //启动服务
        if (startServer) {
            // 加载数据到zkDataBase
            zks.startdata();
            // 启动定时清除session的管理器,注册jmx,添加请求处理器
            zks.startup();
        }
    }
```

zks.startdata();

```
public void startdata() throws IOException, InterruptedException {
        //初始化ZKDatabase，该数据结构用来保存ZK上面存储的所有数据
        //check to see if zkDb is not null
        if (zkDb == null) {
            //初始化数据数据，这里会加入一些原始节点，例如/zookeeper
            zkDb = new ZKDatabase(this.txnLogFactory);
        }
        //加载磁盘上已经存储的数据，如果有的话
        if (!zkDb.isInitialized()) {
            loadData();
        }
    }
```

zks.startup();

```
public synchronized void startup() {
        //初始化session追踪器
        if (sessionTracker == null) {
            createSessionTracker();
        }
        //启动session追踪器
        startSessionTracker();

        //建立请求处理链路
        setupRequestProcessors();

        //注册jmx
        registerJMX();

        setState(State.RUNNING);
        notifyAll();
    }
```

最终Zookeeper应用服务启动，并处于监听状态。