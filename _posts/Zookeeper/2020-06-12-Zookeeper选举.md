---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---

## 选举

Zookeeper中分为Leader、Follower和Observer三个角色，各个角色扮演不同的业务功能。在Leader故障之后，Follower也会选举一个新的Leader。

Leader为集群中的主节点，一个集群只有一个Leader，Leader负责处理Zookeeper的事物操作，也就是更改Zookeeper数据和状态的操作。

Follower负责处理客户端的读请求和参与选举。同时负责处理Leader发出的事物提交请求，也就是提议（proposal）。

Observer用于提高Zookeeper集群的读取的吞吐量，响应读请求，和Follower不同的是，Observser不参与Leader的选举，也不响应Leader发出的proposal。

有角色就有选举。有选举就有策略，Zookeeper中的选举策略有三种实现：包括了LeaderElection、AuthFastLeaderElection和FastLeaderElection，目前Zookeeper默认采用FastLeaderElection，前两个选举算法已经设置为@Deprecated；

### **Zookeeper节点信息**

serverId：服务节点Id，也就是Zookeeper dataDir中配置的myid ，server.*上指定的id。0,1,2,3,4..... ，该Id启动后不变

zxid：数据状态Id，zookeeper每次更新状态之后增加，可理解为全局有序id ，zxid越大，表示数据越新。Zxid是一个64位的数字，高32位为epoch，低32位为递增计数。

epoch：选举时钟,也可以理解为选举轮次，没进行一次选举，该值会+1；

ServerState：服务状态，Zookeeper节点角色状态，分为LOOKING、FOLLOWING、LEADING和OBSERVING，分别对应于不同的角色，当处于选举时，节点处于Looking状态。

每次投票，一个Vote会包含Zookeeper节点信息。

Zookeeper在启动之后会马上进行选举操作，不断的向其他Follower节点发送选票信息，同时也接收别的Follower发送过来的选票信息。最终每个Follower都持有共同的一个选票池，通过同样的算法选出Leader，如果当前节点选为Leader，则向其他每个Follower发送信息，如果没有则向Leader发送信息。

Zookeeper定义了Election接口；其中lookForLeader()就是选举操作。

启动peer选举策略实际启动的为fast leader 选举策略，如果peer状态为LOOKING， 创建投票（最后提交的日志id，时间戳，peerId）。

fast leader 选举策略启动时实际上启动了一个消息处理器Messenger。 消息处理器内部有一个发送消息工作线程WorkerSender，出列一个需要发送的消息，并把它放入管理器QuorumCnxManager的队列； 一个消息接收线程WorkerReceiver处理从QuorumCnxManager接收的消息。

发送消息工作线程WorkerSender，从FastLeaderElection的发送队列poll消息，并把它放入管理器QuorumCnxManager的队列，如果需要则建立消息关联的peer，并发送协议版本，服务id及选举地址, 如果连接peer的id大于 当前peer的id，则关闭连接，否则启动发送工作线程SendWorker和接收线程RecvWorker。 同时QuorumCnxManager在启动时，启动监听，监听peer的连接。发送消息线程SendWorker，从消息队列拉取消息，并通过Socket的DataOutputStream，发送给peer。

消息接收线程WorkerReceiver从QuorumCnxManager的接收队列中拉取消息，并解析出peer的状态（LOOKING, 观察，Follower，或者leader）， 事务id，leaderId，leader选举时间戳，peer的时间戳等信息；如果peer不在当前投票的视图范围之内，同步当前peer的状态（构建通知消息（服务id，事务id，peer状态，时间戳等），并放到发送队列）， 然后更新通知(事务id，leaderId，leader选举时间戳,peer时间戳)，如果当前peer的状态为LOOKING，则添加通知消息到peer的消息接收队列，如果peer状态为LOOKING，则同步当前节点的投票信息给peer， 若果当前节点为非looker，而peer为looker，则发送当前peer相信的leader信息。

接收工作线程RecvWorker，主要是从Socket的Data输入流中读取数据，并组装成消息，放到QuorumCnxManager的消息接收队列，待消息接收线程WorkerReceiver处理。

peer状态有四种LOOKING， OBSERVING，FOLLOWING和LEADING几种状态；LOOKING为初态，Leader还没有选举成功，其他为终态。

当前QuorumPeer处于LOOKING提议投票阶段，启动一个ReadOnlyZooKeeperServer服务，并设置当前peer投票。 ReadOnlyZooKeeperServer内部的处理器链为ReadOnlyRequestProcessor->PrepRequestProcessor->FinalRequestProcessor。 ，只读处理器ReadOnlyRequestProcessor，对CRUD相关的操作，进行忽略，只处理check请求，并通过NettyServerCnxn发送ReplyHeader，头部主要的信息为内存数据库的最大事务id。

创建投票，首先更新当前的投票信息，如果peer为参与者，首先投自己一票（当前peer的serverId，最大事务id，以及时间戳）,并发送通知到所有投票peer; 如果peer状态为LOOKING，且选举没有结束，则从接收消息队列拉取通知, 如果通知为空，则发送投票提议通知到所有投票peer, 否则判断下一轮投票视图是否包括当前通知的server和提议leader, 则判断peer的状态(LOOKING,OBSERVING,FOLLOWING,LEADING)。当前peer状态为LOOKING时,，如果通知的时间点，大于当前server时间点，则更新投票提议，并发送通知消息到所有投票peer。如果当前节点的Quorum Peer都进行投票回复，然后从接收队列中拉取通知投票消息，如果为空，则投票结束，更新当前投票状态为LEADING。当peer为OBSERVING,FOLLOWING状态，什么都不做;当peer状态为leading，则如果投票的时间戳和当前节点的投票时间戳一致，并且所有peer都回复，则结束投票。

### leader选举

```
public interface Election {
     public Vote lookForLeader() throws InterruptedException;
     public void shutdown();
}
```

说明：

选举的父接口为Election，其定义了lookForLeader和shutdown两个方法，lookForLeader表示寻找Leader，shutdown则表示关闭，如关闭服务端之间的连接。

AuthFastLeaderElection，同FastLeaderElection算法基本一致，只是在消息中加入了认证信息，其在3.4.0之后的版本中已经不建议使用。

FastLeaderElection，其是标准的fast paxos算法的实现，基于TCP协议进行选举。

LeaderElection，也表示一种选举算法，其在3.4.0之后的版本中已经不建议使用。

选举入口在下面的方法中

```
org.apache.zookeeper.server.quorum.FastLeaderElection#lookForLeader
```

说明：FastLeaderElection实现了Election接口，其需要实现接口中定义的lookForLeader方法和shutdown方法，其是标准的Fast Paxos算法的实现，各服务器之间基于TCP协议进行选举。

在上面的集群模式启动流程中，最后会调用startLeaderElection()来下进行选举操作。startLeaderElection()中指定了选举算法。同时定义了为自己投一票（坚持你自己，年轻人！），一个Vote包含了投票节点、当前节点的zxid和当前的epoch。Zookeeper默认采取了FastLeaderElection选举算法。最后启动QuorumPeer线程，开始投票。

```
synchronized public void startLeaderElection() {
       try {

           // 所有节点启动的初始状态都是LOOKING，因此这里都会是创建一张投自己为Leader的票
           if (getPeerState() == ServerState.LOOKING) {
               currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
           }
       } catch(IOException e) {
           RuntimeException re = new RuntimeException(e.getMessage());
           re.setStackTrace(e.getStackTrace());
           throw re;
       }

       // if (!getView().containsKey(myid)) {
      //      throw new RuntimeException("My id " + myid + " not in the peer list");
        //}
        if (electionType == 0) {
            try {
                udpSocket = new DatagramSocket(myQuorumAddr.getPort());
                responder = new ResponderThread();
                responder.start();
            } catch (SocketException e) {
                throw new RuntimeException(e);
            }
        }
        //初始化选举算法，electionType默认为3
        this.electionAlg = createElectionAlgorithm(electionType);
    }
```

FastLeaderElection类中定义三个内部类Notification、 ToSend 和 Messenger ，Messenger 中又定义了WorkerReceiver 和 WorkerSender ![Zookeeper选举类图](png\zookeeper\Zookeeper选举类图.png)

​		Notification类表示收到的选举投票信息（其他服务器发来的选举投票信息），其包含了被选举者的id、zxid、选举周期等信息。

ToSend类表示发送给其他服务器的选举投票信息，也包含了被选举者的id、zxid、选举周期等信息。

Message类为消息处理的类，用于发送和接收投票信息，包含了WorkerReceiver和WorkerSender两个线程类。

WorkerReceiver

说明：WorkerReceiver实现了Runnable接口，是选票接收器。其会不断地从QuorumCnxManager中获取其他服务器发来的选举消息，并将其转换成一个选票，然后保存到recvqueue中，在选票接收过程中，如果发现该外部选票的选举轮次小于当前服务器的，那么忽略该外部投票，同时立即发送自己的内部投票。其是将QuorumCnxManager的Message转化为FastLeaderElection的Notification。

其中，WorkerReceiver的主要逻辑在run方法中，其首先会从QuorumCnxManager中的recvQueue队列中取出其他服务器发来的选举消息，消息封装在Message数据结构中。然后判断消息中的服务器id是否包含在可以投票的服务器集合中，若不是，则会将本服务器的内部投票发送给该服务器

```
                        if(!self.getVotingView().containsKey(response.sid)){ // 当前的投票者集合不包含服务器
                            // 获取自己的投票
                            Vote current = self.getCurrentVote();
                            // 构造ToSend消息
                            ToSend notmsg = new ToSend(ToSend.mType.notification,
                                    current.getId(),
                                    current.getZxid(),
                                    logicalclock,
                                    self.getPeerState(),
                                    response.sid,
                                    current.getPeerEpoch());
                            // 放入sendqueue队列，等待发送
                            sendqueue.offer(notmsg);
                        }
```

若包含该服务器，则根据消息（Message）解析出投票服务器的投票信息并将其封装为Notification，然后判断当前服务器是否为LOOKING，若为LOOKING，则直接将Notification放入FastLeaderElection的recvqueue（区别于recvQueue）中。然后判断投票服务器是否为LOOKING状态，并且其选举周期小于当前服务器的逻辑时钟，则将本（当前）服务器的内部投票发送给该服务器，否则，直接忽略掉该投票。

WorkerSender

说明：WorkerSender也实现了Runnable接口，为选票发送器，其会不断地从sendqueue中获取待发送的选票，并将其传递到底层QuorumCnxManager中，其过程是将FastLeaderElection的ToSend转化为QuorumCnxManager的Message。

```
    protected class Messenger {
        // 选票发送器
        WorkerSender ws;
        // 选票接收器
        WorkerReceiver wr;
    }
```

说明：Messenger中维护了一个WorkerSender和WorkerReceiver，分别表示选票发送器和选票接收器。

```
       Messenger(QuorumCnxManager manager) {
            // 创建WorkerSender
            this.ws = new WorkerSender(manager);
            // 新创建线程
            Thread t = new Thread(this.ws,
                    "WorkerSender[myid=" + self.getId() + "]");
            // 设置为守护线程
            t.setDaemon(true);
            // 启动
            t.start();

            // 创建WorkerReceiver
            this.wr = new WorkerReceiver(manager);
            // 创建线程
            t = new Thread(this.wr,
                    "WorkerReceiver[myid=" + self.getId() + "]");
            // 设置为守护线程
            t.setDaemon(true);
            // 启动
            t.start();
        }
```

说明：会启动WorkerSender和WorkerReceiver，并设置为守护线程。

### 选举流程

#### 判断投票结果的策略

上面这个是其中的一种选举算法，选举过程中，各个server收到投票后需要进行投票结果抉择，判断投票结果的策略有两种

```java
// 按照分组权重
org.apache.zookeeper.server.quorum.flexible.QuorumHierarchical

// 简单按照是否是大多数，超过参与投票数的一半
org.apache.zookeeper.server.quorum.flexible.QuorumMaj
```

#### 选票的网络传输

zookeeper中选举使用的端口和正常处理client请求的端口是不一样的，而且由于投票的数据和处理请求的数据不一样，数据传输的方法也不一样。选举使用的网络传输相关的类和数据结构如下

![Zookeeper选举选票](png\zookeeper\Zookeeper选举选票.png)



### 选举过程

- 各自初始化选票
- - proposedLeader：一开始都是选举自己，myid
    - proposedZxid：最后一次处理成功的事务的zxid
    - proposedEpoch：上一次选举成功的leader的epoch，从currentEpoch 文件中读取
- 发送自己的选票给其他参选者
- 接收其他参选者的选票
- - 收到其他参选者的选票后会放入recvqueue，这个是阻塞队列，从里面超时获取
    - 如果超时没有获取到选票vote则采用退避算法，下次使用更长的超时时间
    - 校验选票的有效性，并且当前机器处于looking状态，开始判断是否接受
    - - 如果收到的选票的electionEpoch大于当前机器选票的logicalclock
        - - 进行选票pk，收到的选票和本机初始选票pk，如果收到的选票胜出则更新本地的选票为收到的选票
            - - pk的算法
                - - org.apache.zookeeper.server.quorum.FastLeaderElection#totalOrderPredicate
                    - 选取epoch较大的
                    - 如果epoch相等则取zxid较大的
                    - 如果zxid相等则取myid较大的
            - 如果本机初始选票胜出则更新为当前机器的选票
            - 更新完选票之后重新发出自己的选票
    - - 如果n.electionEpoch < logicalclock.get()则丢弃选票，继续准备接收其他选票
    - - 如果n.electionEpoch == logicalclock.get()并且收到的选票pk（pk算法totalOrderPredicate）之后胜出
        - - 更新本机选票并且，发送新的选票给其他参选者
        - 执行到这里，说明收到的这个选票有效，将选票记录下来，recvset
        - 统计选票
        - - org.apache.zookeeper.server.quorum.FastLeaderElection#getVoteTracker
            - 看看已经收到的投票中，和当前机器选票一致的票数
        - 判断投票结果
        - - org.apache.zookeeper.server.quorum.SyncedLearnerTracker#hasAllQuorums
            - 根据具体的策略判断
            - - QuorumHierarchical
                - QuorumMaj，默认是这个
                - - 判断投该票的主机数目是否占参与投票主机数的大部分，也就是大于1/2
            - 如果本轮选举成功
            - - 如果等finalizeWait时间后还没有其他选票的时候，就认为当前选举结束
                - 设置当前主机状态
                - 退出本轮选举

选举的整个流程为![Zookeeper选举详情](png\zookeeper\Zookeeper选举详情.png)

整个选举过程可大致理解不断的接收选票，比对选票，直到选出leader，每个zookeeper节点都持有自己的选票池，按照统一的比对算法，正常情况下最终选出来的leader是一致的。

### 源码详解

FastLeaderElection类：

说明：其维护了服务器之间的连接（用于发送消息）、发送消息队列、接收消息队列、推选者的一些信息（zxid、id）、是否停止选举流程标识等。

```
public class FastLeaderElection implements Election {
    //..........
    /**
     * Connection manager. Fast leader election uses TCP for
     * communication between peers, and QuorumCnxManager manages
     * such connections.
     */

    QuorumCnxManager manager;
    /*
        Notification表示收到的选举投票信息（其他服务器发来的选举投票信息），
        其包含了被选举者的id、zxid、选举周期等信息，
        其buildMsg方法将选举信息封装至ByteBuffer中再进行发送
     */
    static public class Notification {
       //..........
    }
    /**
     * Messages that a peer wants to send to other peers.
     * These messages can be both Notifications and Acks
     * of reception of notification.
     */
    /*
     ToSend表示发送给其他服务器的选举投票信息，也包含了被选举者的id、zxid、选举周期等信息
     */
    static public class ToSend {
        //..........
    }
    LinkedBlockingQueue<ToSend> sendqueue;
    LinkedBlockingQueue<Notification> recvqueue;

    /**
     * Multi-threaded implementation of message handler. Messenger
     * implements two sub-classes: WorkReceiver and  WorkSender. The
     * functionality of each is obvious from the name. Each of these
     * spawns a new thread.
     */
    protected class Messenger {
        /**
         * Receives messages from instance of QuorumCnxManager on
         * method run(), and processes such messages.
         */

        class WorkerReceiver extends ZooKeeperThread  {
             //..........
        }
        /**
         * This worker simply dequeues a message to send and
         * and queues it on the manager's queue.
         */

        class WorkerSender extends ZooKeeperThread {
            //..........
        }

        WorkerSender ws;
        WorkerReceiver wr;
        Thread wsThread = null;
        Thread wrThread = null;


    }
    //..........
    QuorumPeer self;
    Messenger messenger;
    AtomicLong logicalclock = new AtomicLong(); /* Election instance */
    long proposedLeader;
    long proposedZxid;
    long proposedEpoch;
    //..........
}
```

说明：构造函数中初始化了stop字段和manager字段，并且调用了starter函数

```
    public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager){
        // 字段赋值
        this.stop = false;
        this.manager = manager;
        // 初始化其他信息
        starter(self, manager);
    }
```

调用

```
    private void starter(QuorumPeer self, QuorumCnxManager manager) {
        // 赋值，对Leader和投票者的ID进行初始化操作
        this.self = self;
        proposedLeader = -1;
        proposedZxid = -1;
        
        // 初始化发送队列
        sendqueue = new LinkedBlockingQueue<ToSend>();
        // 初始化接收队列
        recvqueue = new LinkedBlockingQueue<Notification>();
        // 创建Messenger，会启动接收器和发送器线程
        this.messenger = new Messenger(manager);
    }
```

说明：其完成在构造函数中未完成的部分，如会初始化FastLeaderElection的sendqueue和recvqueue，并且启动接收器和发送器线程。

核心函数分析

sendNotifications函数

```
    private void sendNotifications() {
        for (QuorumServer server : self.getVotingView().values()) { // 遍历投票参与者集合
            long sid = server.id;
            
            // 构造发送消息
            ToSend notmsg = new ToSend(ToSend.mType.notification,
                    proposedLeader,
                    proposedZxid,
                    logicalclock,
                    QuorumPeer.ServerState.LOOKING,
                    sid,
                    proposedEpoch);
            if(LOG.isDebugEnabled()){
                LOG.debug("Sending Notification: " + proposedLeader + " (n.leader), 0x"  +
                      Long.toHexString(proposedZxid) + " (n.zxid), 0x" + Long.toHexString(logicalclock)  +
                      " (n.round), " + sid + " (recipient), " + self.getId() +
                      " (myid), 0x" + Long.toHexString(proposedEpoch) + " (n.peerEpoch)");
            }
            // 将发送消息放置于队列
            sendqueue.offer(notmsg);
        }
    }
```

说明：其会遍历所有的参与者投票集合，然后将自己的选票信息发送至上述所有的投票者集合，其并非同步发送，而是将ToSend消息放置于sendqueue中，之后由WorkerSender进行发送。

QuorumPeer线程启动后会开启对ServerState的监听，如果当前服务节点属于Looking状态，则会执行选举操作。Zookeeper服务器启动后是Looking状态，所以服务启动后会马上进行选举操作。通过调用makeLEStrategy().lookForLeader()进行投票操作，也就是FastLeaderElection.lookForLeader();

QuorumPeer.run()：

```
public void run() {
        updateThreadName();

        //..........

        try {
            /*
             * Main loop
             */
            while (running) {
                switch (getPeerState()) {
                case LOOKING:
                    LOG.info("LOOKING");

                    if (Boolean.getBoolean("readonlymode.enabled")) {
                        final ReadOnlyZooKeeperServer roZk =
                            new ReadOnlyZooKeeperServer(logFactory, this, this.zkDb);
                        Thread roZkMgr = new Thread() {
                            public void run() {
                                try {
                                    // lower-bound grace period to 2 secs
                                    sleep(Math.max(2000, tickTime));
                                    if (ServerState.LOOKING.equals(getPeerState())) {
                                        roZk.startup();
                                    }
                                } catch (InterruptedException e) {
                                    LOG.info("Interrupted while attempting to start ReadOnlyZooKeeperServer, not started");
                                } catch (Exception e) {
                                    LOG.error("FAILED to start ReadOnlyZooKeeperServer", e);
                                }
                            }
                        };
                        try {
                            roZkMgr.start();
                            reconfigFlagClear();
                            if (shuttingDownLE) {
                                shuttingDownLE = false;
                                startLeaderElection();
                            }
                            setCurrentVote(makeLEStrategy().lookForLeader());
                        } catch (Exception e) {
                            LOG.warn("Unexpected exception", e);
                            setPeerState(ServerState.LOOKING);
                        } finally {
                            roZkMgr.interrupt();
                            roZk.shutdown();
                        }
                    } else {
                        try {
                           reconfigFlagClear();
                            if (shuttingDownLE) {
                               shuttingDownLE = false;
                               startLeaderElection();
                               }
                            setCurrentVote(makeLEStrategy().lookForLeader());
                        } catch (Exception e) {
                            LOG.warn("Unexpected exception", e);
                            setPeerState(ServerState.LOOKING);
                        }
                    }
                    break;
                case OBSERVING:
                    try {
                        LOG.info("OBSERVING");
                        setObserver(makeObserver(logFactory));
                        observer.observeLeader();
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception",e );
                    } finally {
                        observer.shutdown();
                        setObserver(null);
                       updateServerState();
                    }
                    break;
                case FOLLOWING:
                    try {
                       LOG.info("FOLLOWING");
                        setFollower(makeFollower(logFactory));
                        follower.followLeader();
                    } catch (Exception e) {
                       LOG.warn("Unexpected exception",e);
                    } finally {
                       follower.shutdown();
                       setFollower(null);
                       updateServerState();
                    }
                    break;
                case LEADING:
                    LOG.info("LEADING");
                    try {
                        setLeader(makeLeader(logFactory));
                        leader.lead();
                        setLeader(null);
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception",e);
                    } finally {
                        if (leader != null) {
                            leader.shutdown("Forcing shutdown");
                            setLeader(null);
                        }
                        updateServerState();
                    }
                    break;
                }
                start_fle = Time.currentElapsedTime();
            }
        } finally {
            LOG.warn("QuorumPeer main thread exited");
            MBeanRegistry instance = MBeanRegistry.getInstance();
            instance.unregister(jmxQuorumBean);
            instance.unregister(jmxLocalPeerBean);

            for (RemotePeerBean remotePeerBean : jmxRemotePeerBean.values()) {
                instance.unregister(remotePeerBean);
            }

            jmxQuorumBean = null;
            jmxLocalPeerBean = null;
            jmxRemotePeerBean = null;
        }
    }
```

FastLeaderElection.lookForLeader()：

```
public Vote lookForLeader() throws InterruptedException {
        try {
            self.jmxLeaderElectionBean = new LeaderElectionBean();
            MBeanRegistry.getInstance().register(
                    self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
        } catch (Exception e) {
            LOG.warn("Failed to register with JMX", e);
            self.jmxLeaderElectionBean = null;
        }
        if (self.start_fle == 0) {
           self.start_fle = Time.currentElapsedTime();
        }
        try {
            HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();

            HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();
            //等待200毫秒
            int notTimeout = finalizeWait;

            synchronized(this){
                //逻辑时钟自增+1
                logicalclock.incrementAndGet();
                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
            }

            LOG.info("New election. My id =  " + self.getId() +
                    ", proposed zxid=0x" + Long.toHexString(proposedZxid));
            //发送投票信息
            sendNotifications();

            /*
             * Loop in which we exchange notifications until we find a leader
             */
            //判断是否为Looking状态
            while ((self.getPeerState() == ServerState.LOOKING) &&
                    (!stop)){
                /*
                 * Remove next notification from queue, times out after 2 times
                 * the termination time
                 */
                //获取接收其他Follow发送的投票信息
                Notification n = recvqueue.poll(notTimeout,
                        TimeUnit.MILLISECONDS);

                /*
                 * Sends more notifications if haven't received enough.
                 * Otherwise processes new notification.
                 */
                //未收到投票信息
                if(n == null){
                    //判断是否和集群离线了
                    if(manager.haveDelivered()){
                        //未断开，发送投票
                        sendNotifications();
                    } else {
                        //断开，重连
                        manager.connectAll();
                    }
                    /*
                     * Exponential backoff
                     */
                    int tmpTimeOut = notTimeout*2;
                    notTimeout = (tmpTimeOut < maxNotificationInterval?
                            tmpTimeOut : maxNotificationInterval);
                    LOG.info("Notification time out: " + notTimeout);
                } //接收到了投票，则处理收到的投票信息
                else if (validVoter(n.sid) && validVoter(n.leader)) {
                    /*
                     * Only proceed if the vote comes from a replica in the current or next
                     * voting view for a replica in the current or next voting view.
                     */
                    //其他节点的Server.state
                    switch (n.state) {
                    case LOOKING:
                        //如果其他节点也为Looking状态，说明当前正处于选举阶段，则处理投票信息。

                        // If notification > current, replace and send messages out
                        //如果当前的epoch（投票轮次）小于其他的投票信息，则说明自己的投票轮次已经过时，则更新自己的投票轮次
                        if (n.electionEpoch > logicalclock.get()) {
                            //更新投票轮次
                            logicalclock.set(n.electionEpoch);
                            //清除收到的投票
                            recvset.clear();
                            //比对投票信息
                            //如果本身的投票信息 低于 收到的的投票信息，则使用收到的投票信息，否则再次使用自身的投票信息进行发送投票。
                            if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                    getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                                //使用收到的投票信息
                                updateProposal(n.leader, n.zxid, n.peerEpoch);
                            } else {
                                //使用自己的投票信息
                                updateProposal(getInitId(),
                                        getInitLastLoggedZxid(),
                                        getPeerEpoch());
                            }
                            //发送投票信息
                            sendNotifications();
                        } else if (n.electionEpoch < logicalclock.get()) {
                            //如果其他节点的epoch小于当前的epoch则丢弃
                            if(LOG.isDebugEnabled()){
                                LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                                        + Long.toHexString(n.electionEpoch)
                                        + ", logicalclock=0x" + Long.toHexString(logicalclock.get()));
                            }
                            break;
                        } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                proposedLeader, proposedZxid, proposedEpoch)) {
                            //同样的epoch,正常情况，所有节点基本处于同一轮次
                            //如果自身投票信息 低于 收到的投票信息，则更新投票信息。并发送
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                            sendNotifications();
                        }

                        if(LOG.isDebugEnabled()){
                            LOG.debug("Adding vote: from=" + n.sid +
                                    ", proposed leader=" + n.leader +
                                    ", proposed zxid=0x" + Long.toHexString(n.zxid) +
                                    ", proposed election epoch=0x" + Long.toHexString(n.electionEpoch));
                        }
                        //投票信息Vote归档，收到的有效选票 票池
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

                        //统计投票结果 ，判断是否能结束选举
                        if (termPredicate(recvset,
                                new Vote(proposedLeader, proposedZxid,
                                        logicalclock.get(), proposedEpoch))) {
                            //如果已经选出leader

                            // Verify if there is any change in the proposed leader
                            while((n = recvqueue.poll(finalizeWait,
                                    TimeUnit.MILLISECONDS)) != null){
                                if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                        proposedLeader, proposedZxid, proposedEpoch)){
                                    recvqueue.put(n);
                                    break;
                                }
                            }

                            /*
                             * This predicate is true once we don't read any new
                             * relevant message from the reception queue
                             */
                            //如果选票结果为当前节点，则更新ServerState，否则设置为Follwer
                            if (n == null) {
                                self.setPeerState((proposedLeader == self.getId()) ?
                                        ServerState.LEADING: learningState());

                                Vote endVote = new Vote(proposedLeader,
                                        proposedZxid, proposedEpoch);
                                leaveInstance(endVote);
                                return endVote;
                            }
                        }
                        break;
                    case OBSERVING:
                        LOG.debug("Notification from observer: " + n.sid);
                        break;
                    case FOLLOWING:
                    case LEADING:
                        /*
                         * Consider all notifications from the same epoch
                         * together.
                         */
                        //如果其他节点已经确定为Leader
                        //如果同一个的投票轮次，则加入选票池
                        //判断是否能过半选举出leader ，如果是，则checkLeader
                        /*checkLeader：
                         * 【是否能选举出leader】and
                         * 【(如果投票leader为自身，且轮次一致） or
                         * （如果所选leader不是自身信息在outofelection不为空，且leader的ServerState状态已经为leader）】
                         *
                         */
                        if(n.electionEpoch == logicalclock.get()){
                            recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                            if(termPredicate(recvset, new Vote(n.leader,
                                            n.zxid, n.electionEpoch, n.peerEpoch, n.state))
                                            && checkLeader(outofelection, n.leader, n.electionEpoch)) {
                                self.setPeerState((n.leader == self.getId()) ?
                                        ServerState.LEADING: learningState());

                                Vote endVote = new Vote(n.leader, n.zxid, n.peerEpoch);
                                leaveInstance(endVote);
                                return endVote;
                            }
                        }

                        /*
                         * Before joining an established ensemble, verify that
                         * a majority are following the same leader.
                         * Only peer epoch is used to check that the votes come
                         * from the same ensemble. This is because there is at
                         * least one corner case in which the ensemble can be
                         * created with inconsistent zxid and election epoch
                         * info. However, given that only one ensemble can be
                         * running at a single point in time and that each
                         * epoch is used only once, using only the epoch to
                         * compare the votes is sufficient.
                         *
                         * @see https://issues.apache.org/jira/browse/ZOOKEEPER-1732
                         */
                        outofelection.put(n.sid, new Vote(n.leader,
                                IGNOREVALUE, IGNOREVALUE, n.peerEpoch, n.state));
                        //说明此时 集群中存在别的轮次选举已经有了选举结果
                        //比对outofelection选票池，是否能结束选举，同时检查leader信息
                        //如果能结束选举 接收到的选票产生的leader通过checkLeader为true，则更新当前节点信息
                        if (termPredicate(outofelection, new Vote(n.leader,
                                IGNOREVALUE, IGNOREVALUE, n.peerEpoch, n.state))
                                && checkLeader(outofelection, n.leader, IGNOREVALUE)) {
                            synchronized(this){
                                logicalclock.set(n.electionEpoch);
                                self.setPeerState((n.leader == self.getId()) ?
                                        ServerState.LEADING: learningState());
                            }
                            Vote endVote = new Vote(n.leader, n.zxid, n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                        break;
                    default:
                        LOG.warn("Notification state unrecoginized: " + n.state
                              + " (n.state), " + n.sid + " (n.sid)");
                        break;
                    }
                } else {
                    if (!validVoter(n.leader)) {
                        LOG.warn("Ignoring notification for non-cluster member sid {} from sid {}", n.leader, n.sid);
                    }
                    if (!validVoter(n.sid)) {
                        LOG.warn("Ignoring notification for sid {} from non-quorum member sid {}", n.leader, n.sid);
                    }
                }
            }
            return null;
        } finally {
            try {
                if(self.jmxLeaderElectionBean != null){
                    MBeanRegistry.getInstance().unregister(
                            self.jmxLeaderElectionBean);
                }
            } catch (Exception e) {
                LOG.warn("Failed to unregister with JMX", e);
            }
            self.jmxLeaderElectionBean = null;
            LOG.debug("Number of connection processing threads: {}",
                    manager.getConnectionThreadCount());
        }
    }
```

说明：该函数用于开始新一轮的Leader选举，其首先会将逻辑时钟自增，然后更新本服务器的选票信息（初始化选票），之后将选票信息放入sendqueue等待发送给其他服务器

之后每台服务器会不断地从recvqueue队列中获取外部选票。如果服务器发现无法获取到任何外部投票，就立即确认自己是否和集群中其他服务器保持着有效的连接，如果没有连接，则马上建立连接，如果已经建立了连接，则再次发送自己当前的内部投票

在发送完初始化选票之后，接着开始处理外部投票。在处理外部投票时，会根据选举轮次来进行不同的处理。

**· 外部投票的选举轮次大于内部投票**。若服务器自身的选举轮次落后于该外部投票对应服务器的选举轮次，那么就会立即更新自己的选举轮次(logicalclock)，并且清空所有已经收到的投票，然后使用初始化的投票来进行PK以确定是否变更内部投票。最终再将内部投票发送出去。

**· 外部投票的选举轮次小于内部投****票**。若服务器接收的外选票的选举轮次落后于自身的选举轮次，那么Zookeeper就会直接忽略该外部投票，不做任何处理。

**· 外部投票的选举轮次等于内部投票**。此时可以开始进行选票PK，如果消息中的选票更优，则需要更新本服务器内部选票，再发送给其他服务器。

之后再对选票进行归档操作，无论是否变更了投票，都会将刚刚收到的那份外部投票放入选票集合recvset中进行归档，其中recvset用于记录当前服务器在本轮次的Leader选举中收到的所有外部投票，然后开始统计投票，统计投票是为了统计集群中是否已经有过半的服务器认可了当前的内部投票，如果确定已经有过半服务器认可了该投票，然后再进行最后一次确认，判断是否又有更优的选票产生，若无，则终止投票，然后最终的选票

若选票中的服务器状态为FOLLOWING或者LEADING时，其大致步骤会判断选举周期是否等于逻辑时钟，归档选票，是否已经完成了Leader选举，设置服务器状态，修改逻辑时钟等于选举周期，返回最终选票



lookForLeader方法中把当前选票和收到的选举进行不断的比对和更新，最终选出leader，其中比对选票的方法为totalOrderPredicate(): 其中的比对投票信息方式为：

1.　　首先判断epoch(选举轮次)，也就是选择epoch值更大的节点；如果收到的epoch更大，则当前阶段落后，更新自己的epoch，否则丢弃。

2.　　如果同于轮次中，则选择zxid更大的节点，因为zxid越大说明数据越新。

3.　　如果同一轮次，且zxid一样，则选择serverId最大的节点。

综上3点可理解为越大越棒！

totalOrderPredicate():

```
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
        LOG.debug("id: " + newId + ", proposed id: " + curId + ", zxid: 0x" +
                Long.toHexString(newZxid) + ", proposed zxid: 0x" + Long.toHexString(curZxid));
        if(self.getQuorumVerifier().getWeight(newId) == 0){
            return false;
        }

        /*
         * We return true if one of the following three cases hold:
         * 1- New epoch is higher
         * 2- New epoch is the same as current epoch, but new zxid is higher
         * 3- New epoch is the same as current epoch, new zxid is the same
         *  as current zxid, but server id is higher.
         */

        return ((newEpoch > curEpoch) ||
                ((newEpoch == curEpoch) &&
                ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));
    }
```

说明：该函数将接收的投票与自身投票进行PK，查看是否消息中包含的服务器id是否更优，其按照epoch、zxid、id的优先级进行PK。

```
   protected boolean termPredicate(
            HashMap<Long, Vote> votes,
            Vote vote) {

        HashSet<Long> set = new HashSet<Long>();

        /*
         * First make the views consistent. Sometimes peers will have
         * different zxids for a server depending on timing.
         */
        for (Map.Entry<Long,Vote> entry : votes.entrySet()) { // 遍历已经接收的投票集合
            if (vote.equals(entry.getValue())){ // 将等于当前投票的项放入set
                set.add(entry.getKey());
            }
        }

        //统计set，查看投某个id的票数是否超过一半
        return self.getQuorumVerifier().containsQuorum(set);
    }
```

说明：该函数用于判断Leader选举是否结束，即是否有一半以上的服务器选出了相同的Leader，其过程是将收到的选票与当前选票进行对比，选票相同的放入同一个集合，之后判断选票相同的集合是否超过了半数。

```
    protected boolean checkLeader(
            HashMap<Long, Vote> votes,
            long leader,
            long electionEpoch){
        
        boolean predicate = true;

        /*
         * If everyone else thinks I'm the leader, I must be the leader.
         * The other two checks are just for the case in which I'm not the
         * leader. If I'm not the leader and I haven't received a message
         * from leader stating that it is leading, then predicate is false.
         */

        if(leader != self.getId()){ // 自己不为leader
            if(votes.get(leader) == null) predicate = false; // 还未选出leader
            else if(votes.get(leader).getState() != ServerState.LEADING) predicate = false; // 选出的leader还未给出ack信号，其他服务器还不知道leader
        } else if(logicalclock != electionEpoch) { // 逻辑时钟不等于选举周期
            predicate = false;
        } 

        return predicate;
    }
```

说明：该函数检查是否已经完成了Leader的选举，此时Leader的状态应该是LEADING状态。