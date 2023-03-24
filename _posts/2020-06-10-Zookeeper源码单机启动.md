---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---
# Zookeeper源码单机启动
`Zookeeper`可以单机安装启动调试，这种应用模式主要用在测试或demo的情况下，在生产环境下一般不会采用。

## 单机模式
首先，我们先分析一下`Zookeeper`单机模式下，需要实现哪些核心功能。
- Zookeeper解析配置文件
- 创建服务器实例ZookeeperServer并运行，初始化执行链
- watch机制

下面我们基于以上功能点进行Zookeeper单机模式源码分析。

## Zookeeper单机启动入口
`Zookeeper`统一由`QuorumPeerMain`作为启动类。无论是单机版还是集群模式启动`Zookeeper`服务器，`QuorumPeerMain` 都作为启动入口。
```
org.apache.zookeeper.server.quorum.QuorumPeerMain#main
```

## Zookeeper解析配置文件
Zookeeper单机模式下需要解析配置文件，下面让我们来研究一下。首先我们从入口开始查看源码

### 初始化QuorumPeerMain
初始化`QuorumPeerMain`对象，并执行`initializeAndRun`方法。

```java
    public static void main(String[] args) {
        // 初始化QuorumPeerMain
        QuorumPeerMain main = new QuorumPeerMain();
        try {
        // 解析配置文件信息    
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
`QuorumPeerMain.main()`接受至少一个参数，该参数通过Java的main方法的参数入口传递进来，main方法中没有很多的业务代码，实例化了一个`QuorumPeerMain`对象，然后调用`main.initializeAndRun(args)`。


## 单机解析配置文件
单机模式下解析配置文件通过`main.initializeAndRun(args)`来完成。Zookeeper单机下配置有两种形式。
- 使用配置文件。通过一个参数，参数为zoo.cfg文件路径。
- 使用命令行参数。命令行配置多个（2-4）参数：`port dataDir tickTime maxClientCnxns`
直接在命令指定对应的配置，这种情况只有在单机的时候才会使用，包含以下几个参数
  - port，必填，sever监听的端口
  - dataDir，必填，数据所在的目录
  - tickTime，选填
  - maxClientCnxns，选填，最多可处理的客户端连接数
```java
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
- 解析配置，initializeAndRun方法实例化QuorumPeerConfig对象，如果传入的是配置文件(参数只有一个)，通过parse()来解析zoo.cfg文件中的配置文件并初始化QuorumPeerConfig，QuorumPeerConfig包含了Zookeeper整个应用的配置属性。
- 启动清理文件的线程。开启一个DatadirCleanupManager对象来开启一个Timer用于清除并创建管理新的DataDir相关的数据。
- 判断是单机还是集群，如果是集群：只有一个参数，并且配置了多个server。如果只有一个参数，但是没有配置多台server一般在启动的使用了单机配置。
最后进行程序的启动，因为Zookeeper分为单机和集群模式，所以分为两种不同的启动方式，当zoo.cfg文件中配置了standaloneEnabled=true为单机模式，如果配置server.0,server.1......集群节点，则为集群模式.

### 配置文件解析
配置文件解析通过`org.apache.zookeeper.server.quorum.QuorumPeerConfig#parse`解析

```java
    /**
     * Parse a ZooKeeper configuration file
     * @param path the patch of the configuration file
     * @throws ConfigException error processing configuration
     */
    public void parse(String path) throws ConfigException {
        LOG.info("Reading configuration from: " + path);

        try {
            File configFile = (new VerifyingFileFactory.Builder(LOG)
                .warnForRelativePath()
                .failForNonExistingPath()
                .build()).create(path);

            Properties cfg = new Properties();
            FileInputStream in = new FileInputStream(configFile);
            try {
                cfg.load(in);
                configFileStr = path;
            } finally {
                in.close();
            }

            parseProperties(cfg);
        } catch (IOException e) {
            throw new ConfigException("Error processing " + path, e);
        } catch (IllegalArgumentException e) {
            throw new ConfigException("Error processing " + path, e);
        }

        if (dynamicConfigFileStr!=null) {
           try {
               Properties dynamicCfg = new Properties();
               FileInputStream inConfig = new FileInputStream(dynamicConfigFileStr);
               try {
                   dynamicCfg.load(inConfig);
                   if (dynamicCfg.getProperty("version") != null) {
                       throw new ConfigException("dynamic file shouldn't have version inside");
                   }

                   String version = getVersionFromFilename(dynamicConfigFileStr);
                   // If there isn't any version associated with the filename,
                   // the default version is 0.
                   if (version != null) {
                       dynamicCfg.setProperty("version", version);
                   }
               } finally {
                   inConfig.close();
               }
               setupQuorumPeerConfig(dynamicCfg, false);

           } catch (IOException e) {
               throw new ConfigException("Error processing " + dynamicConfigFileStr, e);
           } catch (IllegalArgumentException e) {
               throw new ConfigException("Error processing " + dynamicConfigFileStr, e);
           }
           File nextDynamicConfigFile = new File(configFileStr + nextDynamicConfigFileSuffix);
           if (nextDynamicConfigFile.exists()) {
               try {
                   Properties dynamicConfigNextCfg = new Properties();
                   FileInputStream inConfigNext = new FileInputStream(nextDynamicConfigFile);
                   try {
                       dynamicConfigNextCfg.load(inConfigNext);
                   } finally {
                       inConfigNext.close();
                   }
                   boolean isHierarchical = false;
                   for (Entry<Object, Object> entry : dynamicConfigNextCfg.entrySet()) {
                       String key = entry.getKey().toString().trim();
                       if (key.startsWith("group") || key.startsWith("weight")) {
                           isHierarchical = true;
                           break;
                       }
                   }
                   lastSeenQuorumVerifier = createQuorumVerifier(dynamicConfigNextCfg, isHierarchical);
               } catch (IOException e) {
                   LOG.warn("NextQuorumVerifier is initiated to null");
               }
           }
        }
    }
```
- 先校验文件的合法性，主要判断是否是绝对路径，文件位置是否存在。
- 配置文件是使用Java的properties形式写的，所以可以通过Properties.load来解析，将解析出来的key、value赋值给对应的配置，最后将参数封装在`QuorumPeerConfig`中

## 创建服务器实例ZookeeperServer
当配置了standaloneEnabled=true 或者没有配置集群节点（sever.*）时，Zookeeper使用单机环境启动。单机环境启动入口为ZooKeeperServerMain类。

zk服务器首先会进行服务器实例的创建，然后对服务器实例进行初始化，包括连接器，内存数据库和请求处理器等组件的初始化。
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
ZooKeeperServerMain类中持有ServerCnxnFactory、ContainerManager和AdminServer对象。
- ServerCnxnFactory为Zookeeper中的核心组件，用于网络通信IO的实现和管理客户端连接，Zookeeper内部提供了两种实现，一种是基于JDK的NIO实现，一种是基于netty的实现。
- ContainerManager类，用于管理维护Zookeeper中节点Znode的信息，管理zkDatabase；
- AdminServer是一个Jetty服务，默认开启8080端口，用于提供Zookeeper的信息的查询接口。该功能从3.5的版本开始。

ZooKeeperServerMain的main方法中同QuorumPeerMain中一致，先实例化本身的对象，再进行init，加载配置文件，然后启动。
```java
    /*
     * Start up the ZooKeeper server.
     *
     * @param args the configfile or the port datadir [ticktime]
     */
    public static void main(String[] args) {
        ZooKeeperServerMain main = new ZooKeeperServerMain();
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
根据Zookeeper的配置文件信息来运行服务。
```java
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
## 运行Zookeeper服务器
运行Zookeeper服务器，执行`org.apache.zookeeper.server.ZooKeeperServerMain#runFromConfig`,
zookeeper服务器运行的主要逻辑如下：
- 初始化FileTxnSnapLog，创建了FileTxnLog实例和FIleSnap实例，并保存刚启动时候DataTree的snapshot
- 初始化zkServer对象ZooKeeperServer，维护一个处理器链表processor chain
- 实例化CountDownLatch线程计数器对象，在程序启动后，执行shutdownLatch.await();用于挂起主程序，并监听Zookeeper运行状态。
- 创建adminServer（Jetty）服务并开启。
- 创建ServerCnxnFactory对象，cnxnFactory = ServerCnxnFactory.createFactory(); Zookeeper默认使用NIOServerCnxnFactory来实现网络通信IO。

```java
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


            // 启动ZooKeeperServer
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
ZooKeeperServer是ZooKeeper中所有服务器的父类，其实现了Session.Expirer和ServerStats.Provider接口
```java
public class ZooKeeperServer implements SessionExpirer, ServerStats.Provider {}
```
SessionExpirer中定义了expire方法（表示会话过期）和getServerId方法（表示获取服务器ID），而Provider则主要定义了获取服务器某些数据的方法。
类中包含了心跳频率，会话跟踪器（处理会话）、事务日志快照、内存数据库、请求处理器、未处理的ChangeRecord、服务器统计信息等。
ZooKeeperServer(FileTxnSnapLog, int, int, int, DataTreeBuilder, ZKDatabase)型构造函数
```java
   public ZooKeeperServer(FileTxnSnapLog txnLogFactory, int tickTime,
            int minSessionTimeout, int maxSessionTimeout,
            DataTreeBuilder treeBuilder, ZKDatabase zkDb) {
        // 给属性赋值
        serverStats = new ServerStats(this);
        this.txnLogFactory = txnLogFactory;
        this.zkDb = zkDb;
        this.tickTime = tickTime;
        this.minSessionTimeout = minSessionTimeout;
        this.maxSessionTimeout = maxSessionTimeout;
        
        LOG.info("Created server with tickTime " + tickTime
                + " minSessionTimeout " + getMinSessionTimeout()
                + " maxSessionTimeout " + getMaxSessionTimeout()
                + " datadir " + txnLogFactory.getDataDir()
                + " snapdir " + txnLogFactory.getSnapDir());
    }
```
该构造函数会初始化服务器统计数据、事务日志工厂、心跳时间、会话时间（最短超时时间和最长超时时间）。

### 创建zookeeper数据管理器FileTxnSnapLog
FileTxnSnapLog是zk上层服务器和底层数据存储的对接层，提供了一系列操作数据文件的接口，如事务日志文件和快照数据文件，Zookeeper根据zoo.cfg文件中解析出的快照数据目录dataDir和事务日志目录dataLogDir来创建FileTxnSnapLog。
```java
    public FileTxnSnapLog(File dataDir, File snapDir) throws IOException {
        LOG.debug("Opening datadir:{} snapDir:{}", dataDir, snapDir);

        this.dataDir = new File(dataDir, version + VERSION);
        this.snapDir = new File(snapDir, version + VERSION);

        // by default create snap/log dirs, but otherwise complain instead
        // See ZOOKEEPER-1161 for more details
        boolean enableAutocreate = Boolean.valueOf(
                System.getProperty(ZOOKEEPER_DATADIR_AUTOCREATE,
                        ZOOKEEPER_DATADIR_AUTOCREATE_DEFAULT));

        trustEmptySnapshot = Boolean.getBoolean(ZOOKEEPER_SNAPSHOT_TRUST_EMPTY);
        LOG.info(ZOOKEEPER_SNAPSHOT_TRUST_EMPTY + " : " + trustEmptySnapshot);

        if (!this.dataDir.exists()) {
            if (!enableAutocreate) {
                throw new DatadirException("Missing data directory "
                        + this.dataDir
                        + ", automatic data directory creation is disabled ("
                        + ZOOKEEPER_DATADIR_AUTOCREATE
                        + " is false). Please create this directory manually.");
            }

            if (!this.dataDir.mkdirs() && !this.dataDir.exists()) {
                throw new DatadirException("Unable to create data directory "
                        + this.dataDir);
            }
        }
        if (!this.dataDir.canWrite()) {
            throw new DatadirException("Cannot write to data directory " + this.dataDir);
        }

        if (!this.snapDir.exists()) {
            // by default create this directory, but otherwise complain instead
            // See ZOOKEEPER-1161 for more details
            if (!enableAutocreate) {
                throw new DatadirException("Missing snap directory "
                        + this.snapDir
                        + ", automatic data directory creation is disabled ("
                        + ZOOKEEPER_DATADIR_AUTOCREATE
                        + " is false). Please create this directory manually.");
            }

            if (!this.snapDir.mkdirs() && !this.snapDir.exists()) {
                throw new DatadirException("Unable to create snap directory "
                        + this.snapDir);
            }
        }
        if (!this.snapDir.canWrite()) {
            throw new DatadirException("Cannot write to snap directory " + this.snapDir);
        }

        // check content of transaction log and snapshot dirs if they are two different directories
        // See ZOOKEEPER-2967 for more details
        if(!this.dataDir.getPath().equals(this.snapDir.getPath())){
            checkLogDir();
            checkSnapDir();
        }

        txnLog = new FileTxnLog(this.dataDir);
        snapLog = new FileSnap(this.snapDir);
    }
```
zookeeper维护了自己的数据结构和物理文件，而且要接收并处理client发送来的网络请求，初始化FileTxnSnapLog，创建了FileTxnLog实例和FIleSnap实例，并保存刚启动时候DataTree的snapshot。

### 创建服务器统计器ServerStats
ServerStats是zk服务器运行时的统计器
```java
 public ZooKeeperServer() {
        serverStats = new ServerStats(this);
        listener = new ZooKeeperServerListenerImpl(this);
    }
```

### 创建ServerCnxnFactory
通过配置系统属性zookeper.serverCnxnFactory来指定使用Zookeeper自己实现的NIO还是使用Netty框架作为Zookeeper服务端网络连接工厂。

Zookeeper中 ServerCnxnFactory默认采用了NIOServerCnxnFactory来实现，也可以通过配置系统属性zookeeper.serverCnxnFactory 来设置使用Netty实现；
```java
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

### 启动ServerCnxnFactory主线程
cnxnFactory.startup(zkServer);方法启动了ServerCnxnFactory ，同时启动ZooKeeper服务
```java
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

### 恢复本地数据
启动时，需要从本地快照数据文件和事务日志文件进行数据恢复。

```java
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

### 创建并启动会话管理器。

```java
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

#### 初始化Zookeeper的请求处理链
Zookeeper请求处理方式为责任链模式的实现。会有多个请求处理器依次处理一个客户端请求，在服务器启动时，会将这些请求处理器串联成一个请求处理链。

```java
protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        RequestProcessor syncProcessor = new SyncRequestProcessor(this,
                finalProcessor);
        ((SyncRequestProcessor)syncProcessor).start();
        firstProcessor = new PrepRequestProcessor(this, syncProcessor);
        ((PrepRequestProcessor)firstProcessor).start();
    }
```

## 注册Zookeeper服务器实例

NIOServerCnxnFactory 类的方法 setZooKeeperServer(zks);

至此，单机版的Zookeeper服务器实例已经启动。


