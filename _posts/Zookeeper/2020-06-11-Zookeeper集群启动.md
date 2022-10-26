---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---


### 集群模式启动

集群模式下启动和单机启动有相似的地方，但是也有各自的特点。集群模式的配置方式和单机模式也是不一样的

#### 概念介绍：角色，服务器状态

集群模式会有多台server，每台server根据不同的角色会有不同的状态，server状态的定义如下

```java
public enum ServerState {
    LOOKING, FOLLOWING, LEADING, OBSERVING;
}
```

LOOKING：表示服务器处于选举状态，说明集群正在进行投票选举，选出leader

FOLLOWING：表示服务器处于following状态，表示当前server的角色是follower

LEADING：表示服务器处于leading状态，当前server角色是leader

OBSERVING：表示服务器处于OBSERVING状态，当前server角色是OBSERVER

对应server的角色有:

##### leader

投票选出的leader，可以处理读写请求。处理写请求的时候收集各个参与投票者的选票，来决出投票结果

##### follower

作用：

1. 参与leader选举，可能被选为leader
2. 接收处理读请求
3. 接收写请求，转发给leader，并参与投票决定写操作是否提交

##### observer

为了支持zk集群可扩展性，如果直接增加follower的数量，会导致投票的性能下降。也就是防止参与投票的server太多，导致leader选举收敛速度较慢，选举所需时间过长。

observer和follower类似，但是不参与选举和投票，

1. 接收处理读请求
2. 接收写请求，转发给leader，但是不参与投票，接收leader的投票结果，同步数据

这样在支持集群可扩展性的同时又不会影响投票的性能

#### 组件启动

集群模式下服务器启动的组件一部分和单机模式下类似，只是启动的流程和时机有所差别

- FileTxnSnapLog
- NIOServerCnxnFactory
- Jetty

也是会启动上面三个组件，但是因为集群模式还有其他组件需要启动，所以具体启动的逻辑不太一样。

除了上面这些组件外，集群模式下还有一些用来支撑集群模式的组件

- QuorumPeer：用来启动各个组件，是选举过程的mainloop，在loop中判断当前server状态来决定做不同的处理
- FastLeaderElection：默认选举算法
- QuorumCnxManager：选举过程中的网络通信组件

#### QuorumPeer

解除出来的QuorumPeerConfig配置都设置到QuorumPeer对应的属性中，主线程启动完QuorumPeer后，调用该线程的join方法等待该线程退出。

#### QuorumCnxManager

负责各个server之间的通信，维护了和各个server之间的连接，下面的线程负责与其他server建立连接

```
org.apache.zookeeper.server.quorum.QuorumCnxManager.Listener
```

还维护了与其他每个server连接对应的发送队列，SendWorker线程负责发送packet给其他server

```
final ConcurrentHashMap<Long, ArrayBlockingQueue<ByteBuffer>> queueSendMap;
```

这个map的key是建立网络连接的server的myid，value是对应的发送队列。

还有接收队列，RecvWorker是用来接收其他server发来的Message的线程，将收到的Message放入队列中

```
org.apache.zookeeper.server.quorum.QuorumCnxManager#recvQueue
```

#### 集群模式启动

Zookeeper主程序QuorumPeerMain加载配置文件后，配置容器对象QuorumPeerConfig中持有一个QuorumVerifier对象，该对象会存储其他Zookeeper server节点信息，如果zoo.cfg中配置了server.*节点信息，会实例化一个QuorumVeriferi对象。其中AllMembers = VotingMembers + ObservingMembers

```
public interface QuorumVerifier {
    long getWeight(long id);
    boolean containsQuorum(Set<Long> set);
    long getVersion();
    void setVersion(long ver);
    Map<Long, QuorumServer> getAllMembers();
    Map<Long, QuorumServer> getVotingMembers();
    Map<Long, QuorumServer> getObservingMembers();
    boolean equals(Object o);
    String toString();
}
```

如果quorumVerifier.getVotingMembers().size() > 1 则使用集群模式启动。调用runFromConfig(QuorumPeerConfig config)，同时会实例化ServerCnxnFactory 对象，初始化一个QuorumPeer对象。

QuorumPeer为一个Zookeeper节点， QuorumPeer 为一个线程类，代表一个Zookeeper服务线程，最终会启动该线程。

runFromConfig方法中设置了一些列属性。包括选举类型、server Id、节点数据库等信息。最后通过quorumPeer.start();启动Zookeeper节点。



     public void runFromConfig(QuorumPeerConfig config)
                throws IOException, AdminServerException
        {
          try {
              // 注册jmx
              ManagedUtil.registerLog4jMBeans();
          } catch (JMException e) {
              LOG.warn("Unable to register log4j JMX control", e);
          }
     LOG.info("Starting quorum peer");
      try {
          ServerCnxnFactory cnxnFactory = null;
          ServerCnxnFactory secureCnxnFactory = null;
    
          if (config.getClientPortAddress() != null) {
              cnxnFactory = ServerCnxnFactory.createFactory();
              // 配置客户端连接端口
              cnxnFactory.configure(config.getClientPortAddress(),
                      config.getMaxClientCnxns(),
                      false);
          }
    
          if (config.getSecureClientPortAddress() != null) {
              secureCnxnFactory = ServerCnxnFactory.createFactory();
              // 配置安全连接端口
              secureCnxnFactory.configure(config.getSecureClientPortAddress(),
                      config.getMaxClientCnxns(),
                      true);
          }
    
          // ------------初始化当前zk服务节点的配置----------------
          // 设置数据和快照操作
          quorumPeer = getQuorumPeer();
          quorumPeer.setTxnFactory(new FileTxnSnapLog(
                      config.getDataLogDir(),
                      config.getDataDir()));
          quorumPeer.enableLocalSessions(config.areLocalSessionsEnabled());
          quorumPeer.enableLocalSessionsUpgrading(
              config.isLocalSessionsUpgradingEnabled());
          //quorumPeer.setQuorumPeers(config.getAllMembers());
          // 选举类型
          quorumPeer.setElectionType(config.getElectionAlg());
          // server Id
          quorumPeer.setMyid(config.getServerId());
          quorumPeer.setTickTime(config.getTickTime());
          quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
          quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
          quorumPeer.setInitLimit(config.getInitLimit());
          quorumPeer.setSyncLimit(config.getSyncLimit());
          quorumPeer.setConfigFileName(config.getConfigFilename());
    
          // 设置zk的节点数据库
          quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
          quorumPeer.setQuorumVerifier(config.getQuorumVerifier(), false);
          if (config.getLastSeenQuorumVerifier()!=null) {
              quorumPeer.setLastSeenQuorumVerifier(config.getLastSeenQuorumVerifier(), false);
          }
    
          // 初始化zk数据库
          quorumPeer.initConfigInZKDatabase();
          quorumPeer.setCnxnFactory(cnxnFactory);
          quorumPeer.setSecureCnxnFactory(secureCnxnFactory);
          quorumPeer.setLearnerType(config.getPeerType());
          quorumPeer.setSyncEnabled(config.getSyncEnabled());
          quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());
    
          // sets quorum sasl authentication configurations
          quorumPeer.setQuorumSaslEnabled(config.quorumEnableSasl);
          if(quorumPeer.isQuorumSaslAuthEnabled()){
              quorumPeer.setQuorumServerSaslRequired(config.quorumServerRequireSasl);
              quorumPeer.setQuorumLearnerSaslRequired(config.quorumLearnerRequireSasl);
              quorumPeer.setQuorumServicePrincipal(config.quorumServicePrincipal);
              quorumPeer.setQuorumServerLoginContext(config.quorumServerLoginContext);
              quorumPeer.setQuorumLearnerLoginContext(config.quorumLearnerLoginContext);
          }
          quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
    
          // -------------初始化当前zk服务节点的配置---------------
          quorumPeer.initialize();
    
          //启动
          quorumPeer.start();
          quorumPeer.join();
      } catch (InterruptedException e) {
          // warn, but generally this is ok
          LOG.warn("Quorum Peer interrupted", e);
      }
    }

quorumPeer.start(); Zookeeper会首先加载本地磁盘数据，如果之前存在一些Zookeeper信息，则会加载到Zookeeper内存数据库中。通过FileTxnSnapLog中的loadDatabse();

```
public synchronized void start() {

        // 校验serverid如果不在peer列表中，抛异常
        if (!getView().containsKey(myid)) {
            throw new RuntimeException("My id " + myid + " not in the peer list");
         }

        // 加载zk数据库:载入之前持久化的一些信息
        loadDataBase();

        // 启动连接服务端
        startServerCnxnFactory();
        try {
            adminServer.start();
        } catch (AdminServerException e) {
            LOG.warn("Problem starting AdminServer", e);
            System.out.println(e);
        }
        // 启动之后马上进行选举，主要是创建选举必须的环境，比如：启动相关线程
        startLeaderElection();

        // 执行选举逻辑
        super.start();
    }
```

加载数据完之后同单机模式启动一样，会调用ServerCnxnFactory.start(),启动NIOServerCnxnFactory服务和Zookeeper服务，最后启动AdminServer服务。

与单机模式启动不同的是，集群会在启动之后马上进行选举操作，会在配置的所有Zookeeper server节点中选举出一个leader角色。startLeaderElection(); 