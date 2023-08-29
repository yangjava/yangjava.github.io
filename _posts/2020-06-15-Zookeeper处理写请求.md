---
layout: post
categories: [Zookeeper]
description: none
keywords: Zookeeper
---

## 处理写请求

### Leader消息处理

观察者、跟随者、和领导者启动server，分别为，LeaderZooKeeperServer,FollowerZooKeeperServer,ObserverZooKeeperServer. Leader的消息处处理器链为LeaderRequestProcessor->PrepRequestProcessor->ProposalRequestProcessor->CommitProcessor->ToBeAppliedRequestProcessor->FinalRequestProcessor; Follower的消息处理器链为SendAckRequestProcessor->SyncRequestProcessor->FollowerRequestProcessor->CommitProcessor->FinalRequestProcessor. Observer的消息处处理器链为SyncRequestProcessor->ObserverRequestProcessor->CommitProcessor->FinalRequestProcessor;

LeaderRequestProcessor处理器主要做的本地会话检查，并更新会话保活信息。 PrepRequestProcessor处理消息，首先添加到内部的提交请求队列，然后启动线程预处理请求。 预处理消息，主要是针对事务性的CUD， 则构建响应的请求，比如CreateRequest，SetDataRequest等。针对非事务性R，则检查会话的有效性。 事务性预处理请求，主要是将请求包装成事件变更记录ZooKeeperServer，并保存到Zookeeper的请求变更记录集outstandingChanges中。 ProposalRequestProcessor处理器，主要处理同步请求消息，针对同步请求，则发送消息到响应的server。 CommitProcessor，主要是过滤出需要提交的请求，比如CRUD等，并交由下一个处理器处理。 ToBeAppliedRequestProcessor处理器，主要是保证提议为最新。 FinalRequestProcessor首先由ZooKeeperServer处理CUD相关的请求操作，针对R类的相关操作，直接查询ZooKeeperServer的内存数据库。 ZooKeeperServer处理CUD操作，委托表给ZKDatabase，ZKDatabase委托给DataTree， DataTree根据CUD相关请求操作，CUD相应路径的 DataNode。针对R类操作，获取dataTree的DataNode的相关信息。

### Follower消息处理

### Observer消息处理

针对跟随者，SendAckRequestProcessor处理器，针对非同步操作，回复ACK。 SyncRequestProcessor处理器从请求队列拉取请求,针对刷新队列不为空的情况，如果请求队列为空，则提交请求日志，并刷新到磁盘，否则根据日志计数器和快照计数器计算是否需要拍摄快照。 FollowerRequestProcessor处理器，从请求队列拉取请求，如果请求为同步请求，则添加请求到同步队列, 并转发请求给leader，如果为CRUD相关的操作，直接转发请求给leader。

针对观察者，观察者请求处理器，从请求队列拉取请求，如果请求为同步请求，则添加请求到同步队列, 并转发请求给leader，如果为CRUD相关的操作，直接转发请求给leader。

### 处理写请求过程

zk为了保证分布式数据一致性，使用ZAB协议，在客户端发起一次写请求的时候时候，假设该请求请求到的是follower，follower不会直接处理这个请求，而是转发给leader，由leader发起投票决定该请求最终能否执行成功，所以整个过程client、被请求的follower、leader、其他follower都参与其中。以创建一个节点为例，总体流程如下

![Zookeeper处理写请求](png\zookeeper\Zookeeper处理写请求.png)

从图中可以看出，创建流程为

1. follower接受请求，解析请求
2. follower通过FollowerRequestProcessor将请求转发给leader
3. leader接收请求
4. leader发送proposal给follower
5. follower收到请求记录txlog、snapshot
6. follower发送ack给leader
7. leader收到ack后进行commit，并且通知所有的learner，发送commit packet给所有的learner

### Zookeeper的服务器

这里说的follower、leader都是server，在zk里面server总共有这么几种

![Zookeeper中的Server](png\zookeeper\Zookeeper中的Server.png)

由于server角色不同，对于请求所做的处理不同，每种server包含的processor也不同，下面细说下具体有哪些processor。

ZooKeeperServer，为所有服务器的父类，其请求处理链为PrepRequestProcessor -> SyncRequestProcessor -> FinalRequestProcessor。

QuorumZooKeeperServer，其是所有参与选举的服务器的父类，是抽象类，其继承了ZooKeeperServer类。

LeaderZooKeeperServer，Leader服务器，继承了QuorumZooKeeperServer类，其请求处理链为PrepRequestProcessor -> ProposalRequestProcessor -> CommitProcessor -> Leader.ToBeAppliedRequestProcessor -> FinalRequestProcessor。

LearnerZooKeeper，其是Learner服务器的父类，为抽象类，也继承了QuorumZooKeeperServer类。

FollowerZooKeeperServer，Follower服务器，继承了LearnerZooKeeper，其请求处理链为FollowerRequestProcessor -> CommitProcessor -> FinalRequestProcessor。

ObserverZooKeeperServer，Observer服务器，继承了LearnerZooKeeper。

ReadOnlyZooKeeperServer，只读服务器，不提供写服务，继承QuorumZooKeeperServer，其处理链的第一个处理器为ReadOnlyRequestProcessor。

#### ZooKeeperServer源码分析

```
public class ZooKeeperServer implements SessionExpirer, ServerStats.Provider {}
```

说明：ZooKeeperServer是ZooKeeper中所有服务器的父类，其实现了Session.Expirer和ServerStats.Provider接口，SessionExpirer中定义了expire方法（表示会话过期）和getServerId方法（表示获取服务器ID），而Provider则主要定义了获取服务器某些数据的方法。

内部类

类的内部类

1. DataTreeBuilder类

```
    public interface DataTreeBuilder {
        // 构建DataTree
        public DataTree build();
    }
```

说明：其定义了构建树DataTree的接口。

2. BasicDataTreeBuilder类

```
    static public class BasicDataTreeBuilder implements DataTreeBuilder {
        public DataTree build() {
            return new DataTree();
        }
    }
```

说明：实现DataTreeBuilder接口，返回新创建的树DataTree。

3. MissingSessionException类

```
    public static class MissingSessionException extends IOException {
        private static final long serialVersionUID = 7467414635467261007L;

        public MissingSessionException(String msg) {
            super(msg);
        }
    }
```

说明：表示会话缺失异常。

4. ChangeRecord类

```
    static class ChangeRecord {
        ChangeRecord(long zxid, String path, StatPersisted stat, int childCount,
                List<ACL> acl) {
            // 属性赋值
            this.zxid = zxid;
            this.path = path;
            this.stat = stat;
            this.childCount = childCount;
            this.acl = acl;
        }
        
        // zxid
        long zxid;

        // 路径
        String path;

        // 统计数据
        StatPersisted stat; /* Make sure to create a new object when changing */

        // 子节点个数
        int childCount;

        // ACL列表
        List<ACL> acl; /* Make sure to create a new object when changing */

        @SuppressWarnings("unchecked")
        // 拷贝
        ChangeRecord duplicate(long zxid) {
            StatPersisted stat = new StatPersisted();
            if (this.stat != null) {
                DataTree.copyStatPersisted(this.stat, stat);
            }
            return new ChangeRecord(zxid, path, stat, childCount,
                    acl == null ? new ArrayList<ACL>() : new ArrayList(acl));
        }
    }
```

说明：ChangeRecord数据结构是用于方便PrepRequestProcessor和FinalRequestProcessor之间进行信息共享，其包含了一个拷贝方法duplicate，用于返回属性相同的ChangeRecord实例。

类的属性

说明：类中包含了心跳频率，会话跟踪器（处理会话）、事务日志快照、内存数据库、请求处理器、未处理的ChangeRecord、服务器统计信息等。

类的构造函数

1. ZooKeeperServer()型构造函数

```
    public ZooKeeperServer() {
        serverStats = new ServerStats(this);
    }
```

说明：其只初始化了服务器的统计信息。

2. ZooKeeperServer(FileTxnSnapLog, int, int, int, DataTreeBuilder, ZKDatabase)型构造函数

```
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

说明：该构造函数会初始化服务器统计数据、事务日志工厂、心跳时间、会话时间（最短超时时间和最长超时时间）。

3. ZooKeeperServer(FileTxnSnapLog, int, DataTreeBuilder)型构造函数

```
    public ZooKeeperServer(FileTxnSnapLog txnLogFactory, int tickTime,
            DataTreeBuilder treeBuilder) throws IOException {
        this(txnLogFactory, tickTime, -1, -1, treeBuilder,
                new ZKDatabase(txnLogFactory));
    }
```

说明：其首先会生成ZooKeeper内存数据库后，然后调用第二个构造函数进行初始化操作。

4. ZooKeeperServer(File, File, int)型构造函数

```
    public ZooKeeperServer(File snapDir, File logDir, int tickTime)
            throws IOException {
        this( new FileTxnSnapLog(snapDir, logDir),
                tickTime, new BasicDataTreeBuilder());
    }
```

说明：其会调用同名构造函数进行初始化操作。

5. ZooKeeperServer(FileTxnSnapLog, DataTreeBuilder)型构造函数

```
    public ZooKeeperServer(FileTxnSnapLog txnLogFactory,
            DataTreeBuilder treeBuilder)
        throws IOException
    {
        this(txnLogFactory, DEFAULT_TICK_TIME, -1, -1, treeBuilder,
                new ZKDatabase(txnLogFactory));
    }
```

说明：其生成内存数据库之后再调用同名构造函数进行初始化操作。

核心函数分析

loadData函数

```
    public void loadData() throws IOException, InterruptedException {
        /*
         * When a new leader starts executing Leader#lead, it 
         * invokes this method. The database, however, has been
         * initialized before running leader election so that
         * the server could pick its zxid for its initial vote.
         * It does it by invoking QuorumPeer#getLastLoggedZxid.
         * Consequently, we don't need to initialize it once more
         * and avoid the penalty of loading it a second time. Not 
         * reloading it is particularly important for applications
         * that host a large database.
         * 
         * The following if block checks whether the database has
         * been initialized or not. Note that this method is
         * invoked by at least one other method: 
         * ZooKeeperServer#startdata.
         *  
         * See ZOOKEEPER-1642 for more detail.
         */
        if(zkDb.isInitialized()){ // 内存数据库已被初始化
            // 设置为最后处理的Zxid
            setZxid(zkDb.getDataTreeLastProcessedZxid());
        }
        else { // 未被初始化，则加载数据库
            setZxid(zkDb.loadDataBase());
        }
        
        // Clean up dead sessions
        LinkedList<Long> deadSessions = new LinkedList<Long>();
        for (Long session : zkDb.getSessions()) { // 遍历所有的会话
            if (zkDb.getSessionWithTimeOuts().get(session) == null) { // 删除过期的会话
                deadSessions.add(session);
            }
        }
        // 完成DataTree的初始化
        zkDb.setDataTreeInit(true);
        for (long session : deadSessions) { // 遍历过期会话
            // XXX: Is lastProcessedZxid really the best thing to use?
            // 删除会话
            killSession(session, zkDb.getDataTreeLastProcessedZxid());
        }
    }
```

说明：该函数用于加载数据，其首先会判断内存库是否已经加载设置zxid，之后会调用killSession函数删除过期的会话，killSession会从sessionTracker中删除session，并且killSession最后会调用DataTree的killSession函数，其源码如下

```
    void killSession(long session, long zxid) {
        // the list is already removed from the ephemerals
        // so we do not have to worry about synchronizing on
        // the list. This is only called from FinalRequestProcessor
        // so there is no need for synchronization. The list is not
        // changed here. Only create and delete change the list which
        // are again called from FinalRequestProcessor in sequence.
        // 移除session，并获取该session对应的所有临时节点
        HashSet<String> list = ephemerals.remove(session);
        if (list != null) {
            for (String path : list) { // 遍历所有临时节点
                try {
                    // 删除路径对应的节点
                    deleteNode(path, zxid);
                    if (LOG.isDebugEnabled()) {
                        LOG
                                .debug("Deleting ephemeral node " + path
                                        + " for session 0x"
                                        + Long.toHexString(session));
                    }
                } catch (NoNodeException e) {
                    LOG.warn("Ignoring NoNodeException for path " + path
                            + " while removing ephemeral for dead session 0x"
                            + Long.toHexString(session));
                }
            }
        }
    }
```

说明：DataTree的killSession函数的逻辑首先移除session，然后取得该session下的所有临时节点，然后逐一删除临时节点。

submit函数

```
    public void submitRequest(Request si) {
        if (firstProcessor == null) { // 第一个处理器为空
            synchronized (this) {
                try {
                    while (!running) { // 直到running为true，否则继续等待
                        wait(1000);
                    }
                } catch (InterruptedException e) {
                    LOG.warn("Unexpected interruption", e);
                }
                if (firstProcessor == null) {
                    throw new RuntimeException("Not started");
                }
            }
        }
        try {
            touch(si.cnxn);
            // 是否为合法的packet
            boolean validpacket = Request.isValid(si.type);
            if (validpacket) { 
                // 处理请求
                firstProcessor.processRequest(si);
                if (si.cnxn != null) {
                    incInProcess();
                }
            } else {
                LOG.warn("Received packet at server of unknown type " + si.type);
                new UnimplementedRequestProcessor().processRequest(si);
            }
        } catch (MissingSessionException e) {
            if (LOG.isDebugEnabled()) {
                LOG.debug("Dropping request: " + e.getMessage());
            }
        } catch (RequestProcessorException e) {
            LOG.error("Unable to process request:" + e.getMessage(), e);
        }
    }
```

说明：当firstProcessor为空时，并且running标志为false时，其会一直等待，直到running标志为true，之后调用touch函数判断session是否存在或者已经超时，之后判断请求的类型是否合法，合法则使用请求处理器进行处理。

processConnectRequest函数

说明：其首先将传递的ByteBuffer进行反序列化，转化为相应的ConnectRequest，之后进行一系列判断（可能抛出异常），然后获取并判断该ConnectRequest中会话id是否为0，若为0，则表示可以创建会话，否则，重新打开会话。

processPacket函数

说明：该函数首先将传递的ByteBuffer进行反序列，转化为相应的RequestHeader，然后根据该RequestHeader判断是否需要认证，若认证失败，则构造认证失败的响应并发送给客户端，然后关闭连接，并且再补接收任何packet。若认证成功，则构造认证成功的响应并发送给客户端。若不需要认证，则再判断其是否为SASL类型，若是，则进行处理，然后构造响应并发送给客户端，否则，构造请求并且提交请求。

#### LeaderZooKeeperServer源码分析

类的继承关系

```
public class LeaderZooKeeperServer extends QuorumZooKeeperServer {}
```

说明：LeaderZooKeeperServer继承QuorumZooKeeperServer抽象类，其会继承ZooKeeperServer中的很多方法。

类的属性

```
public class LeaderZooKeeperServer extends QuorumZooKeeperServer {
    // 提交请求处理器
    CommitProcessor commitProcessor;
}
```

说明：其只有一个CommitProcessor类，表示提交请求处理器，其在处理链中的位置位于ProposalRequestProcessor之后，ToBeAppliedRequestProcessor之前。

类的构造函数

```
    LeaderZooKeeperServer(FileTxnSnapLog logFactory, QuorumPeer self,
            DataTreeBuilder treeBuilder, ZKDatabase zkDb) throws IOException {
        super(logFactory, self.tickTime, self.minSessionTimeout,
                self.maxSessionTimeout, treeBuilder, zkDb, self);
    }
```

说明：其直接调用父类QuorumZooKeeperServer的构造函数，然后再调用ZooKeeperServer的构造函数，逐级构造。

核心函数分析

1. setupRequestProcessors函数

```
    protected void setupRequestProcessors() {
        // 创建FinalRequestProcessor
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        // 创建ToBeAppliedRequestProcessor
        RequestProcessor toBeAppliedProcessor = new Leader.ToBeAppliedRequestProcessor(
                finalProcessor, getLeader().toBeApplied);
        // 创建CommitProcessor
        commitProcessor = new CommitProcessor(toBeAppliedProcessor,
                Long.toString(getServerId()), false);
        // 启动CommitProcessor
        commitProcessor.start();
        // 创建ProposalRequestProcessor
        ProposalRequestProcessor proposalProcessor = new ProposalRequestProcessor(this,
                commitProcessor);
        // 初始化ProposalProcessor
        proposalProcessor.initialize();
        // firstProcessor为PrepRequestProcessor
        firstProcessor = new PrepRequestProcessor(this, proposalProcessor);
        // 启动PrepRequestProcessor
        ((PrepRequestProcessor)firstProcessor).start();
    }
```

说明：该函数表示创建处理链，可以看到其处理链的顺序为PrepRequestProcessor -> ProposalRequestProcessor -> CommitProcessor -> Leader.ToBeAppliedRequestProcessor -> FinalRequestProcessor。

2. registerJMX函数

```
    protected void registerJMX() {
        // register with JMX
        try {
            // 创建DataTreeBean
            jmxDataTreeBean = new DataTreeBean(getZKDatabase().getDataTree());
            // 进行注册
            MBeanRegistry.getInstance().register(jmxDataTreeBean, jmxServerBean);
        } catch (Exception e) {
            LOG.warn("Failed to register with JMX", e);
            jmxDataTreeBean = null;
        }
    }
```

说明：该函数用于注册JMX服务，首先使用DataTree初始化DataTreeBean，然后使用DataTreeBean和ServerBean调用register函数进行注册，其源码如下

```
    public void register(ZKMBeanInfo bean, ZKMBeanInfo parent)
        throws JMException
    {
        // 确保bean不为空
        assert bean != null;
        String path = null;
        if (parent != null) { // parent(ServerBean)不为空
            // 通过parent从bean2Path中获取path
            path = mapBean2Path.get(parent);
            // 确保path不为空
            assert path != null;
        }
        // 补充为完整的路径
        path = makeFullPath(path, parent);
        if(bean.isHidden())
            return;
        // 使用路径来创建名字
        ObjectName oname = makeObjectName(path, bean);
        try {
            // 注册Server
            mBeanServer.registerMBean(bean, oname);
            // 将bean和对应path放入mapBean2Path
            mapBean2Path.put(bean, path);
            // 将name和bean放入mapName2Bean
            mapName2Bean.put(bean.getName(), bean);
        } catch (JMException e) {
            LOG.warn("Failed to register MBean " + bean.getName());
            throw e;
      
```

说明：可以看到会通过parent来获取路径，然后创建名字，然后注册bean，之后将相应字段放入mBeanServer和mapBean2Path中，即完成注册过程。

3. unregisterJMX函数

```
    protected void unregisterJMX() {
        // unregister from JMX
        try {
            if (jmxDataTreeBean != null) {
                // 取消注册
                MBeanRegistry.getInstance().unregister(jmxDataTreeBean);
            }
        } catch (Exception e) {
            LOG.warn("Failed to unregister with JMX", e);
        }
        jmxDataTreeBean = null;
    }
```

说明：该函数用于取消注册JMX服务，其会调用unregister函数，其源码如下

```
    public void unregister(ZKMBeanInfo bean) {
        if(bean==null)
            return;
        // 获取对应路径
        String path=mapBean2Path.get(bean);
        try {
            // 取消注册
            unregister(path,bean);
        } catch (JMException e) {
            LOG.warn("Error during unregister", e);
        }
        // 从mapBean2Path和mapName2Bean中移除bean
        mapBean2Path.remove(bean);
        mapName2Bean.remove(bean.getName());
    }
```

说明：unregister与register的过程恰好相反，是移除bean的过程。

#### **FollowerZooKeeperServer源码分析**

类的继承关系

```
public class FollowerZooKeeperServer extends LearnerZooKeeperServer {}
```

说明：其继承LearnerZooKeeperServer抽象类，角色为Follower。其请求处理链为FollowerRequestProcessor -> CommitProcessor -> FinalRequestProcessor。

类的属性

```
public class FollowerZooKeeperServer extends LearnerZooKeeperServer {
    private static final Logger LOG =
        LoggerFactory.getLogger(FollowerZooKeeperServer.class);
    // 提交请求处理器
    CommitProcessor commitProcessor;
    
    // 同步请求处理器
    SyncRequestProcessor syncProcessor;

    /*
     * Pending sync requests
     */
    // 待同步请求
    ConcurrentLinkedQueue<Request> pendingSyncs;
    
    // 待处理的事务请求
    LinkedBlockingQueue<Request> pendingTxns = new LinkedBlockingQueue<Request>();
}
```

说明：FollowerZooKeeperServer中维护着提交请求处理器和同步请求处理器，并且维护了所有待同步请求队列和待处理的事务请求队列。

2.3 类的构造函数

```
    FollowerZooKeeperServer(FileTxnSnapLog logFactory,QuorumPeer self,
            DataTreeBuilder treeBuilder, ZKDatabase zkDb) throws IOException {
        super(logFactory, self.tickTime, self.minSessionTimeout,
                self.maxSessionTimeout, treeBuilder, zkDb, self);
        // 初始化pendingSyncs
        this.pendingSyncs = new ConcurrentLinkedQueue<Request>();
    }
```

说明：其首先调用父类的构造函数，然后初始化pendingSyncs为空队列。

核心函数分析

1. logRequest函数

```
    public void logRequest(TxnHeader hdr, Record txn) {
        // 创建请求
        Request request = new Request(null, hdr.getClientId(), hdr.getCxid(),
                hdr.getType(), null, null);
        // 赋值请求头、事务体、zxid
        request.hdr = hdr;
        request.txn = txn;
        request.zxid = hdr.getZxid();
        if ((request.zxid & 0xffffffffL) != 0) { // zxid不为0，表示本服务器已经处理过请求
            // 则需要将该请求放入pendingTxns中
            pendingTxns.add(request);
        }
        // 使用SyncRequestProcessor处理请求(其会将请求放在队列中，异步进行处理)
        syncProcessor.processRequest(request);
    }
```

说明：该函数将请求进行记录（放入到对应的队列中），等待处理。

2. commit函数

```
    public void commit(long zxid) {
        if (pendingTxns.size() == 0) { // 没有还在等待处理的事务
            LOG.warn("Committing " + Long.toHexString(zxid)
                    + " without seeing txn");
            return;
        }
        // 队首元素的zxid
        long firstElementZxid = pendingTxns.element().zxid;
        if (firstElementZxid != zxid) { // 如果队首元素的zxid不等于需要提交的zxid，则退出程序
            LOG.error("Committing zxid 0x" + Long.toHexString(zxid)
                    + " but next pending txn 0x"
                    + Long.toHexString(firstElementZxid));
            System.exit(12);
        }
        // 从待处理事务请求队列中移除队首请求
        Request request = pendingTxns.remove();
        // 提交该请求
        commitProcessor.commit(request);
    }
```

说明：该函数会提交zxid对应的请求（pendingTxns的队首元素），其首先会判断队首请求对应的zxid是否为传入的zxid，然后再进行移除和提交（放在committedRequests队列中）。

3. sync函数

```
    synchronized public void sync(){
        if(pendingSyncs.size() ==0){ // 没有需要同步的请求
            LOG.warn("Not expecting a sync.");
            return;
        }
        // 从待同步队列中移除队首请求
        Request r = pendingSyncs.remove();
        // 提交该请求
        commitProcessor.commit(r);
    }
```

说明：该函数会将待同步请求队列中的元素进行提交，也是将该请求放入committedRequests队列中。

### follower的processor链

这里的follower是FollowerZooKeeperServer，通过setupRequestProcessors来设置自己的processor链

```java
FollowerRequestProcessor -> CommitProcessor ->FinalRequestProcessor
```

每个processor对应的功能为:

#### FollowerRequestProcessor：

作用：将请求先转发给下一个processor，然后根据不同的OpCode做不同的操作

如果是：sync，先加入org.apache.zookeeper.server.quorum.FollowerZooKeeperServer#pendingSyncs，然后发送给leader

如果是：create等，直接转发

如果是：createSession或者closeSession，如果不是localsession则转发给leader

#### CommitProcessor ：

有一个WorkerService，将请求封装为CommitWorkRequest执行

作用：

转发请求，读请求直接转发给下一个processor

写请求先放在pendingRequests对应的sessionId下的list中，收到leader返回的commitRequest再处理

1. 处理读请求（不会改变服务器状态的请求）
2. 处理committed的写请求（经过leader 处理完成的请求）

维护一个线程池服务WorkerService，每个请求都在单独的线程中处理

1. 每个session的请求必须按顺序执行
2. 写请求必须按照zxid顺序执行
3. 确认一个session中写请求之间没有竞争

#### FinalRequestProcessor：

总是processor chain上最后一个processor

作用：

1. 实际处理事务请求的processor
2. 处理query请求
3. 返回response给客户端

#### SyncRequestProcessor：

作用：

1. 接收leader的proposal进行处理
2. 从org.apache.zookeeper.server.SyncRequestProcessor#queuedRequests中取出请求记录txlog和snapshot
3. 然后加入toFlush，从toFlush中取出请求交给org.apache.zookeeper.server.quorum.SendAckRequestProcessor#flush处理

### leader的processor链

这里的leader就是LeaderZooKeeperServer

通过setupRequestProcessors来设置自己的processor链

```java
PrepRequestProcessor -> ProposalRequestProcessor ->CommitProcessor -> Leader.ToBeAppliedRequestProcessor ->FinalRequestProcessor
```

具体情况可以参看代码：

```
@Override
    protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        RequestProcessor toBeAppliedProcessor = new Leader.ToBeAppliedRequestProcessor(finalProcessor, getLeader());
        commitProcessor = new CommitProcessor(toBeAppliedProcessor,
                Long.toString(getServerId()), false,
                getZooKeeperServerListener());
        commitProcessor.start();
        ProposalRequestProcessor proposalProcessor = new ProposalRequestProcessor(this,
                commitProcessor);
        proposalProcessor.initialize();
        prepRequestProcessor = new PrepRequestProcessor(this, proposalProcessor);
        prepRequestProcessor.start();
        firstProcessor = new LeaderRequestProcessor(this, prepRequestProcessor);

        setupContainerManager();
    }
```



#### 客户端发起写请求

在客户端启动的时候会创建Zookeeper实例，client会连接到server，后面client在创建节点的时候就可以直接和server通信，client发起创建创建节点请求的过程是：

```java
org.apache.zookeeper.ZooKeeper#create(java.lang.String, byte[], java.util.List<org.apache.zookeeper.data.ACL>, org.apache.zookeeper.CreateMode)
org.apache.zookeeper.ClientCnxn#submitRequest
org.apache.zookeeper.ClientCnxn#queuePacket
```

1. 在`ZooKeeper#create`方法中构造请求的request

2. 在

   ```
   ClientCnxn#queuePacket
   ```

   方法中将request封装到packet中，将packet放入发送队列outgoingQueue中等待发送

    1. SendThread不断从发送队列outgoingQueue中取出packet发送

3. 通过 packet.wait等待server返回

#### follower和leader交互过程

client发出请求后，follower会接收并处理该请求。选举结束后follower确定了自己的角色为follower，一个端口和client通信，一个端口和leader通信。监听到来自client的连接口建立新的session，监听对应的socket上的读写事件，如果client有请求发到follower，follower会用下面的方法处理

```java
org.apache.zookeeper.server.NIOServerCnxn#readPayload
org.apache.zookeeper.server.NIOServerCnxn#readRequest
```

readPayload这个方法里面会判断是连接请求还是非连接请求，连接请求在之前session建立的文章中介绍过，这里从处理非连接请求开始。

#### follower接收client请求

follower收到请求之后，先请求请求的opCode类型（这里是create）构造对应的request，然后交给第一个processor执行，follower的第一个processor是FollowerRequestProcessor.

#### follower转发请求给leader

由于在zk中follower是不能处理写请求的，需要转交给leader处理，在FollowerRequestProcessor中将请求转发给leader，转发请求的调用堆栈是

```java
serialize(OutputArchive, String):82, QuorumPacket (org.apache.zookeeper.server.quorum), QuorumPacket.java
 writeRecord(Record, String):123, BinaryOutputArchive (org.apache.jute), BinaryOutputArchive.java
 writePacket(QuorumPacket, boolean):139, Learner (org.apache.zookeeper.server.quorum), Learner.java
 request(Request):191, Learner (org.apache.zookeeper.server.quorum), Learner.java
 run():96, FollowerRequestProcessor (org.apache.zookeeper.server.quorum), FollowerRequestProcessor.java
```

FollowerRequestProcessor是一个线程在zk启动的时候就开始运行，主要逻辑在run方法里面，run方法的主要逻辑是

先把请求提交给CommitProcessor（后面leader发送给follower的commit请求对应到这里），然后将请求转发给leader，转发给leader的过程就是构造一个QuorumPacket，通过之前选举通信的端口发送给leader。

### leader接收follower请求

leader获取leader地位以后，启动learnhandler，然后一直在LearnerHandler#run循环，接收来自learner的packet，处理流程是：

```java
processRequest(Request):1003, org.apache.zookeeper.server.PrepRequestProcessor.java
submitLearnerRequest(Request):150, org.apache.zookeeper.server.quorum.LeaderZooKeeperServer.java
run():625, org.apache.zookeeper.server.quorum.LearnerHandler.java
```

handler判断是REQUEST请求的话交给leader的processor链处理，将请求放入org.apache.zookeeper.server.PrepRequestProcessor#submittedRequests，即leader的第一个processor。这个processor也是一个线程，从submittedRequests中不断拿出请求处理

processor主要做了：

1. 交给CommitProcessor等待提交
2. 交给leader的下一个processor处理：ProposalRequestProcessor

#### leader 发送proposal给follower

ProposalRequestProcessor主要作用就是讲请求交给下一个processor并且发起投票，将proposal发送给所有的follower。

```
public void processRequest(Request request) throws RequestProcessorException {
        // LOG.warn("Ack>>> cxid = " + request.cxid + " type = " +
        // request.type + " id = " + request.sessionId);
        // request.addRQRec(">prop");


        /* In the following IF-THEN-ELSE block, we process syncs on the leader.
         * If the sync is coming from a follower, then the follower
         * handler adds it to syncHandler. Otherwise, if it is a client of
         * the leader that issued the sync command, then syncHandler won't
         * contain the handler. In this case, we add it to syncHandler, and
         * call processRequest on the next processor.
         */

        if (request instanceof LearnerSyncRequest){
            zks.getLeader().processSync((LearnerSyncRequest)request);
        } else {
            nextProcessor.processRequest(request);
            if (request.getHdr() != null) {
                // We need to sync and get consensus on any transactions
                try {
                    zks.getLeader().propose(request);
                } catch (XidRolloverException e) {
                    throw new RequestProcessorException(e.getMessage(), e);
                }
                syncProcessor.processRequest(request);
            }
        }
    }
    
```



```java
// org.apache.zookeeper.server.quorum.Leader#sendPacket
void sendPacket(QuorumPacket qp) {
    synchronized (forwardingFollowers) {
        // 所有的follower，observer没有投票权
        for (LearnerHandler f : forwardingFollowers) {
            f.queuePacket(qp);
        }
    }
}
```

#### follower 收到proposal

follower处理proposal请求的调用堆栈

```java
processRequest(Request):214, org.apache.zookeeper.server.SyncRequestProcessor.java
 logRequest(TxnHeader, Record):89, org.apache.zookeeper.server.quorumFollowerZooKeeperServer.java
 processPacket(QuorumPacket):147, org.apache.zookeeper.server.quorum.Follower.java
 followLeader():102, org.apache.zookeeper.server.quorum.Follower.java
 run():1199, org.apache.zookeeper.server.quorum.QuorumPeer.java
```

将请求放入org.apache.zookeeper.server.SyncRequestProcessor#queuedRequests

#### follower 发送ack

线程SyncRequestProcessor#run从org.apache.zookeeper.server.SyncRequestProcessor#toFlush中取出请求flush，处理过程

1. follower开始commit，记录txLog和snapShot
2. 发送commit成功请求给leader，也就是follower给leader的ACK

#### leader收到ack

leader 收到ack后判断是否收到大多数的follower的ack，如果是说明可以commit，commit后同步给follower

#### follower收到commit

还是Follower#followLeader里面的while循环收到leader的commit请求后，调用下面的方法处理

org.apache.zookeeper.server.quorum.FollowerZooKeeperServer#commit

最终加入CommitProcessor.committedRequests队列，CommitProcessor主线程发现队列不空表明需要把这个request转发到下一个processor

#### follower发送请求给客户端

follower的最后一个processor是FinalRequestProcessor，最后会创建对应的节点并且构造response返回给client



### leader处理请求源码

如下所示：

```
 /**
     * Keep a count of acks that are received by the leader for a particular
     * proposal
     *
     * @param zxid, the zxid of the proposal sent out
     * @param sid, the id of the server that sent the ack
     * @param followerAddr
     */
    synchronized public void processAck(long sid, long zxid, SocketAddress followerAddr) {        
        if (!allowedToCommit) return; // last op committed was a leader change - from now on 
                                     // the new leader should commit        
        if (LOG.isTraceEnabled()) {
            LOG.trace("Ack zxid: 0x{}", Long.toHexString(zxid));
            for (Proposal p : outstandingProposals.values()) {
                long packetZxid = p.packet.getZxid();
                LOG.trace("outstanding proposal: 0x{}",
                        Long.toHexString(packetZxid));
            }
            LOG.trace("outstanding proposals all");
        }
        
        if ((zxid & 0xffffffffL) == 0) {
            /*
             * We no longer process NEWLEADER ack with this method. However,
             * the learner sends an ack back to the leader after it gets
             * UPTODATE, so we just ignore the message.
             */
            return;
        }
            
            
        if (outstandingProposals.size() == 0) {
            if (LOG.isDebugEnabled()) {
                LOG.debug("outstanding is 0");
            }
            return;
        }
        if (lastCommitted >= zxid) {
            if (LOG.isDebugEnabled()) {
                LOG.debug("proposal has already been committed, pzxid: 0x{} zxid: 0x{}",
                        Long.toHexString(lastCommitted), Long.toHexString(zxid));
            }
            // The proposal has already been committed
            return;
        }
        Proposal p = outstandingProposals.get(zxid);
        if (p == null) {
            LOG.warn("Trying to commit future proposal: zxid 0x{} from {}",
                    Long.toHexString(zxid), followerAddr);
            return;
        }
        
        p.addAck(sid);        
        /*if (LOG.isDebugEnabled()) {
            LOG.debug("Count for zxid: 0x{} is {}",
                    Long.toHexString(zxid), p.ackSet.size());
        }*/
        
        boolean hasCommitted = tryToCommit(p, zxid, followerAddr);

        // If p is a reconfiguration, multiple other operations may be ready to be committed,
        // since operations wait for different sets of acks.
       // Currently we only permit one outstanding reconfiguration at a time
       // such that the reconfiguration and subsequent outstanding ops proposed while the reconfig is
       // pending all wait for a quorum of old and new config, so its not possible to get enough acks
       // for an operation without getting enough acks for preceding ops. But in the future if multiple
       // concurrent reconfigs are allowed, this can happen and then we need to check whether some pending
        // ops may already have enough acks and can be committed, which is what this code does.

        if (hasCommitted && p.request!=null && p.request.getHdr().getType() == OpCode.reconfig){
               long curZxid = zxid;
           while (allowedToCommit && hasCommitted && p!=null){
               curZxid++;
               p = outstandingProposals.get(curZxid);
               if (p !=null) hasCommitted = tryToCommit(p, curZxid, null);             
           }
        }
    }
    
```

调用实现，最终由CommitProcessor 接着处理请求：

```
 /**
     * @return True if committed, otherwise false.
     * @param a proposal p
     **/
    synchronized public boolean tryToCommit(Proposal p, long zxid, SocketAddress followerAddr) {       
       // make sure that ops are committed in order. With reconfigurations it is now possible
       // that different operations wait for different sets of acks, and we still want to enforce
       // that they are committed in order. Currently we only permit one outstanding reconfiguration
       // such that the reconfiguration and subsequent outstanding ops proposed while the reconfig is
       // pending all wait for a quorum of old and new config, so its not possible to get enough acks
       // for an operation without getting enough acks for preceding ops. But in the future if multiple
       // concurrent reconfigs are allowed, this can happen.
       if (outstandingProposals.containsKey(zxid - 1)) return false;
       
       // getting a quorum from all necessary configurations
        if (!p.hasAllQuorums()) {
           return false;                 
        }
        
        // commit proposals in order
        if (zxid != lastCommitted+1) {    
           LOG.warn("Commiting zxid 0x" + Long.toHexString(zxid)
                    + " from " + followerAddr + " not first!");
            LOG.warn("First is "
                    + (lastCommitted+1));
        }     
        
        // in order to be committed, a proposal must be accepted by a quorum              
        
        outstandingProposals.remove(zxid);
        
        if (p.request != null) {
             toBeApplied.add(p);
        }

        if (p.request == null) {
            LOG.warn("Going to commmit null: " + p);
        } else if (p.request.getHdr().getType() == OpCode.reconfig) {                                   
            LOG.debug("Committing a reconfiguration! " + outstandingProposals.size()); 
                 
            //if this server is voter in new config with the same quorum address, 
            //then it will remain the leader
            //otherwise an up-to-date follower will be designated as leader. This saves
            //leader election time, unless the designated leader fails                             
            Long designatedLeader = getDesignatedLeader(p, zxid);
            //LOG.warn("designated leader is: " + designatedLeader);

            QuorumVerifier newQV = p.qvAcksetPairs.get(p.qvAcksetPairs.size()-1).getQuorumVerifier();
       
            self.processReconfig(newQV, designatedLeader, zk.getZxid(), true);
       
            if (designatedLeader != self.getId()) {
                allowedToCommit = false;
            }
                   
            // we're sending the designated leader, and if the leader is changing the followers are 
            // responsible for closing the connection - this way we are sure that at least a majority of them 
            // receive the commit message.
            commitAndActivate(zxid, designatedLeader);
            informAndActivate(p, designatedLeader);
            //turnOffFollowers();
        } else {
            commit(zxid);
            inform(p);
        }
        zk.commitProcessor.commit(p.request);
        if(pendingSyncs.containsKey(zxid)){
            for(LearnerSyncRequest r: pendingSyncs.remove(zxid)) {
                sendSync(r);
            }               
        } 
        
        return  true;   
    }
```

该程序第一步是发送一个请求到Quorum的所有成员

```
    /**
     * Create a commit packet and send it to all the members of the quorum
     *
     * @param zxid
     */
    public void commit(long zxid) {
        synchronized(this){
            lastCommitted = zxid;
        }
        QuorumPacket qp = new QuorumPacket(Leader.COMMIT, zxid, null, null);
        sendPacket(qp);
    }
```

发送报文如下：

```
    /**
     * send a packet to all the followers ready to follow
     *
     * @param qp
     *                the packet to be sent
     */
    void sendPacket(QuorumPacket qp) {
        synchronized (forwardingFollowers) {
            for (LearnerHandler f : forwardingFollowers) {
                f.queuePacket(qp);
            }
        }
    }
```

第二步是通知Observer

```
    /**
     * Create an inform packet and send it to all observers.
     * @param zxid
     * @param proposal
     */
    public void inform(Proposal proposal) {
        QuorumPacket qp = new QuorumPacket(Leader.INFORM, proposal.request.zxid,
                                            proposal.packet.getData(), null);
        sendObserverPacket(qp);
    }
```

发送observer程序如下：

```
    /**
     * send a packet to all observers
     */
    void sendObserverPacket(QuorumPacket qp) {
        for (LearnerHandler f : getObservingLearners()) {
            f.queuePacket(qp);
        }
    }
```

第三步到

```
 zk.commitProcessor.commit(p.request);
```

2. CommitProcessor

CommitProcessor是多线程的，线程间通信通过queue，atomic和wait/notifyAll同步。CommitProcessor扮演一个网关角色，允许请求到剩下的处理管道。在同一瞬间，它支持多个读请求而仅支持一个写请求，这是为了保证写请求在事务中的顺序。

1个commit处理主线程，它监控请求队列，并将请求分发到工作线程，分发过程基于sessionId，这样特定session的读写请求通常分发到同一个线程，因而可以保证运行的顺序。

0~N个工作进程，他们在请求上运行剩下的请求处理管道。如果配置为0个工作线程，主commit线程将会直接运行管道。

经典(默认)线程数是：在32核的机器上，一个commit处理线程和32个工作线程。

多线程的限制：

每个session的请求处理必须是顺序的。

写请求处理必须按照zxid顺序。

必须保证一个session内不会出现写条件竞争，条件竞争可能导致另外一个session的读请求触发监控。

当前实现解决第三个限制，仅仅通过不允许在写请求时允许读进程的处理。

```
 @Override
    public void run() {
        Request request;
        try {
            while (!stopped) {
                synchronized(this) {
                    while (
                        !stopped &&
                        ((queuedRequests.isEmpty() || isWaitingForCommit() || isProcessingCommit()) &&
                         (committedRequests.isEmpty() || isProcessingRequest()))) {
                        wait();
                    }
                }

                /*
                 * Processing queuedRequests: Process the next requests until we
                 * find one for which we need to wait for a commit. We cannot
                 * process a read request while we are processing write request.
                 */
                while (!stopped && !isWaitingForCommit() &&
                       !isProcessingCommit() &&
                       (request = queuedRequests.poll()) != null) {
                    if (needCommit(request)) {
                        nextPending.set(request);
                    } else {
                        sendToNextProcessor(request);
                    }
                }

                /*
                 * Processing committedRequests: check and see if the commit
                 * came in for the pending request. We can only commit a
                 * request when there is no other request being processed.
                 */
                processCommitted();
            }
        } catch (Throwable e) {
            handleException(this.getName(), e);
        }
        LOG.info("CommitProcessor exited loop!");
    }
```

主逻辑程序如下：

```
 /*
     * Separated this method from the main run loop
     * for test purposes (ZOOKEEPER-1863)
     */
    protected void processCommitted() {
        Request request;

        if (!stopped && !isProcessingRequest() &&
                (committedRequests.peek() != null)) {

            /*
             * ZOOKEEPER-1863: continue only if there is no new request
             * waiting in queuedRequests or it is waiting for a
             * commit. 
             */
            if ( !isWaitingForCommit() && !queuedRequests.isEmpty()) {
                return;
            }
            request = committedRequests.poll();

            /*
             * We match with nextPending so that we can move to the
             * next request when it is committed. We also want to
             * use nextPending because it has the cnxn member set
             * properly.
             */
            Request pending = nextPending.get();
            if (pending != null &&
                pending.sessionId == request.sessionId &&
                pending.cxid == request.cxid) {
                // we want to send our version of the request.
                // the pointer to the connection in the request
                pending.setHdr(request.getHdr());
                pending.setTxn(request.getTxn());
                pending.zxid = request.zxid;
                // Set currentlyCommitting so we will block until this
                // completes. Cleared by CommitWorkRequest after
                // nextProcessor returns.
                currentlyCommitting.set(pending);
                nextPending.set(null);
                sendToNextProcessor(pending);
            } else {
                // this request came from someone else so just
                // send the commit packet
                currentlyCommitting.set(request);
                sendToNextProcessor(request);
            }
        }      
    }
```

启动多线程处理程序

```
    /**
     * Schedule final request processing; if a worker thread pool is not being
     * used, processing is done directly by this thread.
     */
    private void sendToNextProcessor(Request request) {
        numRequestsProcessing.incrementAndGet();
        workerPool.schedule(new CommitWorkRequest(request), request.sessionId);
    }
```

真实逻辑是

```
 /**
     * Schedule work to be done by the thread assigned to this id. Thread
     * assignment is a single mod operation on the number of threads.  If a
     * worker thread pool is not being used, work is done directly by
     * this thread.
     */
    public void schedule(WorkRequest workRequest, long id) {
        if (stopped) {
            workRequest.cleanup();
            return;
        }

        ScheduledWorkRequest scheduledWorkRequest =
            new ScheduledWorkRequest(workRequest);

        // If we have a worker thread pool, use that; otherwise, do the work
        // directly.
        int size = workers.size();
        if (size > 0) {
            try {
                // make sure to map negative ids as well to [0, size-1]
                int workerNum = ((int) (id % size) + size) % size;
                ExecutorService worker = workers.get(workerNum);
                worker.execute(scheduledWorkRequest);
            } catch (RejectedExecutionException e) {
                LOG.warn("ExecutorService rejected execution", e);
                workRequest.cleanup();
            }
        } else {
            // When there is no worker thread pool, do the work directly
            // and wait for its completion
            scheduledWorkRequest.start();
            try {
                scheduledWorkRequest.join();
            } catch (InterruptedException e) {
                LOG.warn("Unexpected exception", e);
                Thread.currentThread().interrupt();
            }
        }
    }
```

请求处理线程run方法：

```
       @Override
        public void run() {
            try {
                // Check if stopped while request was on queue
                if (stopped) {
                    workRequest.cleanup();
                    return;
                }
                workRequest.doWork();
            } catch (Exception e) {
                LOG.warn("Unexpected exception", e);
                workRequest.cleanup();
            }
        }
```

调用commitProcessor的doWork方法

```
        public void doWork() throws RequestProcessorException {
            try {
                nextProcessor.processRequest(request);
            } finally {
                // If this request is the commit request that was blocking
                // the processor, clear.
                currentlyCommitting.compareAndSet(request, null);

                /*
                 * Decrement outstanding request count. The processor may be
                 * blocked at the moment because it is waiting for the pipeline
                 * to drain. In that case, wake it up if there are pending
                 * requests.
                 */
                if (numRequestsProcessing.decrementAndGet() == 0) {
                    if (!queuedRequests.isEmpty() ||
                        !committedRequests.isEmpty()) {
                        wakeup();
                    }
                }
            }
        }
```

将请求传递给下一个RP：Leader.ToBeAppliedRequestProcessor

3.Leader.ToBeAppliedRequestProcessor

Leader.ToBeAppliedRequestProcessor仅仅维护一个toBeApplied列表。

```
 /**
         * This request processor simply maintains the toBeApplied list. For
         * this to work next must be a FinalRequestProcessor and
         * FinalRequestProcessor.processRequest MUST process the request
         * synchronously!
         *
         * @param next
         *                a reference to the FinalRequestProcessor
         */
        ToBeAppliedRequestProcessor(RequestProcessor next, Leader leader) {
            if (!(next instanceof FinalRequestProcessor)) {
                throw new RuntimeException(ToBeAppliedRequestProcessor.class
                        .getName()
                        + " must be connected to "
                        + FinalRequestProcessor.class.getName()
                        + " not "
                        + next.getClass().getName());
            }
            this.leader = leader;
            this.next = next;
        }

        /*
         * (non-Javadoc)
         *
         * @see org.apache.zookeeper.server.RequestProcessor#processRequest(org.apache.zookeeper.server.Request)
         */
        public void processRequest(Request request) throws RequestProcessorException {
            next.processRequest(request);

            // The only requests that should be on toBeApplied are write
            // requests, for which we will have a hdr. We can't simply use
            // request.zxid here because that is set on read requests to equal
            // the zxid of the last write op.
            if (request.getHdr() != null) {
                long zxid = request.getHdr().getZxid();
                Iterator<Proposal> iter = leader.toBeApplied.iterator();
                if (iter.hasNext()) {
                    Proposal p = iter.next();
                    if (p.request != null && p.request.zxid == zxid) {
                        iter.remove();
                        return;
                    }
                }
                LOG.error("Committed request not found on toBeApplied: "
                          + request);
            }
        }
```

4. *FinalRequestProcessor前文已经说明，本文不在赘述。*

小结：从上面的分析可以知道，leader处理请求的顺序分别是：PrepRequestProcessor -> ProposalRequestProcessor ->CommitProcessor -> Leader.ToBeAppliedRequestProcessor ->**FinalRequestProcessor。**

请求先通过PrepRequestProcessor接收请求，并进行包装，然后请求类型的不同，设置同享数据；主要负责通知所有follower和observer；CommitProcessor 启动多线程处理请求；Leader.ToBeAppliedRequestProcessor仅仅维护一个toBeApplied列表；

FinalRequestProcessor来作为消息处理器的终结者，发送响应消息，并触发watcher的处理程序。