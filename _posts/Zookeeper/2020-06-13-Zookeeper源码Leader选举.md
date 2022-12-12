---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---
# Zookeeper源码Leader选举
Zookeeper中分为Leader、Follower和Observer三个角色，各个角色扮演不同的业务功能。在Leader故障之后，Follower也会选举一个新的Leader。

## 选举概述
Leader为集群中的主节点，一个集群只有一个Leader，Leader负责处理Zookeeper的事物操作，也就是更改Zookeeper数据和状态的操作。

Follower负责处理客户端的读请求和参与选举。同时负责处理Leader发出的事物提交请求，也就是提议（proposal）。

Observer用于提高Zookeeper集群的读取的吞吐量，响应读请求，和Follower不同的是，Observser不参与Leader的选举，也不响应Leader发出的proposal。

有角色就有选举。有选举就有策略，Zookeeper中的选举策略有三种实现：包括了LeaderElection、AuthFastLeaderElection和FastLeaderElection，目前Zookeeper默认采用FastLeaderElection，前两个选举算法已经设置为@Deprecated；

在开始分析选举的原理之前，先了解几个重要的参数

### 服务器 ID（myid）

比如有三台服务器，编号分别是 1,2,3。

编号越大在选择算法中的权重越大。

### zxid 事务 id-（ZooKeeper transaction ID）

值越大说明数据越新，在选举算法中的权重也越大

### 逻辑时钟（epoch – logicalclock）

或者叫投票的次数，同一轮投票过程中的逻辑时钟值是相同的。每投完一次票这个数据就会增加，然后与接收到的其它服务器返回的投票信息中的数值相比，根据不同的值做出不同的判断。

### 选举状态

LOOKING，竞选状态。

FOLLOWING，随从状态，同步 leader 状态，参与投票。

OBSERVING，观察状态,同步 leader 状态，不参与投票。LEADING，领导者状态。


## leader选举
Zookeeper在启动之后会马上进行选举操作，不断的向其他Follower节点发送选票信息，同时也接收别的Follower发送过来的选票信息。最终每个Follower都持有共同的一个选票池，通过同样的算法选出Leader，如果当前节点选为Leader，则向其他每个Follower发送信息，如果没有则向Leader发送信息。
```java
    @Override
    public synchronized void start() {
        if (!getView().containsKey(myid)) {
            throw new RuntimeException("My id " + myid + " not in the peer list");
         }
        //调用zkDb.loadDataBase();加载数据
        loadDataBase();
        //还记得之前的cnxnFactory = ServerCnxnFactory.createFactory();
        //这里将会调用NIOServerCnxnFactory的start()方法
        startServerCnxnFactory();
        try {
            adminServer.start();
        } catch (AdminServerException e) {
            LOG.warn("Problem starting AdminServer", e);
            System.out.println(e);
        }
        //leader选举的重点
        startLeaderElection();
        //QuorumPeer继承了Thread,这里启动的是线程
        super.start();
    }
```
调用startLeaderElection()来下进行选举操作。startLeaderElection()中指定了选举算法。同时定义了为自己投一票（坚持你自己，年轻人！），一个Vote包含了投票节点、当前节点的zxid和当前的epoch。Zookeeper默认采取了FastLeaderElection选举算法。最后启动QuorumPeer线程，开始投票。
```java
synchronized public void startLeaderElection() {
        try {
        //刚启动的时候state默认是LOOKING
        //因为成员变量state = ServerState.LOOKING;
        if (getPeerState() == ServerState.LOOKING) {
        //构造一个投票(myid,zxid,epoch)
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
        udpSocket = new DatagramSocket(getQuorumAddress().getPort());
        responder = new ResponderThread();
        responder.start();
        } catch (SocketException e) {
        throw new RuntimeException(e);
        }
        }
        //开始选举算法,这里的electionType=3
        this.electionAlg = createElectionAlgorithm(electionType);
        }
```

我们在查看下createElectionAlgorithm(electionType)具体内容稍候再看，这里先简单概括一下它干了些什么事：
1. 启动QuorumCnxManager.Listener线程
2. 启动FastLeaderElection.Messenger.WorkerReceiver线程
3. 启动FastLeaderElection.Messenger.WorkerSender线程

所以createElectionAlgorithm(electionType)只是启动了3个线程而已,所以它会很快返回。
接着就会执行start()->super.start();,即QuorumPeer.run()方法。我们先来看看这个方法：
```java
public void run() {
    ...
    try {
        /*
            * Main loop
            */
        while (running) {
            ...
            switch (getPeerState()) {
            case LOOKING:
                LOG.info("LOOKING");
                ServerMetrics.getMetrics().LOOKING_COUNT.add(1);

                if (Boolean.getBoolean("readonlymode.enabled")) {
                    ...
                } else {
                    try {
                        reconfigFlagClear();
                        if (shuttingDownLE) {
                            shuttingDownLE = false;
                            startLeaderElection();
                        }
                        //makeLEStrategy().lookForLeader()内部也是个while(true)
                        //直接跳到FLE.lookForLeader()
                        //直到leader选举结束
                        setCurrentVote(makeLEStrategy().lookForLeader());
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception", e);
                        setPeerState(ServerState.LOOKING);
                    }
                }
                //leader选举结束后,确定了本节点是什么角色，然后进入对应的switch case
                break;
            case OBSERVING:
                ...
                break;
            case FOLLOWING:
                ...
                break;
            case LEADING:
                ..
                break;
            }
        }//end while
    } finally {
        ...
    }
}
```
QuorumPeer.run()真正需要关心的是这一句setCurrentVote(makeLEStrategy().lookForLeader());再跟进一步，其实我们最关心的是FasterLeaderElection.lookForLeader()方法。该方法内部是个while(true)循环,直到leader选举结束！

## Vote选票讲解
Vote选举过程中的选票，格式如下
```java
    public Vote(long id,
                    long zxid,
                    long peerEpoch) {
        this.version = 0x0;
        this.id = id;
        this.zxid = zxid;
        this.electionEpoch = -1;
        this.peerEpoch = peerEpoch;
        this.state = ServerState.LOOKING;
    }
```
初始化数据如下
```text
version=0
id=1
zxid=0
electionEpoch=-1
peerEpoch=0
state=ServerState.LOOKING
```
每次投票会包含所推举的服务器的 myid 和 ZXID、epoch，使用(myid, ZXID,epoch)来表示。
初始化选票如下所示(1, 0, 0)

## 投票选举
我们把刚刚跳过的createElectionAlgorithm(electionType)方法看一下。
```java
protected Election createElectionAlgorithm(int electionAlgorithm) {
    Election le = null;

    //TODO: use a factory rather than a switch
    switch (electionAlgorithm) {
    case 1:
        throw new UnsupportedOperationException("Election Algorithm 1 is not supported.");
    case 2:
        throw new UnsupportedOperationException("Election Algorithm 2 is not supported.");
    case 3:
        //QuorumCnxManager也是一个非常重要的类
        //负责发起网络请求(将投票发送出去或者接受别的节点发送过来的投票)
        QuorumCnxManager qcm = createCnxnManager();
        QuorumCnxManager oldQcm = qcmRef.getAndSet(qcm);
        if (oldQcm != null) {
            LOG.warn("Clobbering already-set QuorumCnxManager (restarting leader election?)");
            oldQcm.halt();
        }
        //QuorumCnxManager.Listener这个内部类Listener也是继承Thread
        //所以listener.start();之后我们去看Listener的run()方法
        //该Listener的作用是开启一个选举端口,比如我们在z1.cfg配置的2223端口
        //具体位置在QuorumCnxManager.Listener.ListenerHandler.run()->acceptConnections()->createNewServerSocket();
        //当elect Server收到投票票据后,按以下流程处理网络数据
        //QuorumCnxManager.receiveConnection()->handleConnection()
        //在handleConnection()内部new RecvWorker(sock, din, sid, sw)线程。(注意din是我们收到的网络投票数据)
        //RecvWorker线程把收到的投票数据扔到QuorumCnxManager的recvQueue阻塞队列
        //(FastLeaderElection.Messenger.WorkerReceiver线程会循环从recvQueue队列中拉数据,表现如下
        // manager.pollRecvQueue(3000, TimeUnit.MILLISECONDS);)至此,形成了一个闭环
        QuorumCnxManager.Listener listener = qcm.listener;
        if (listener != null) {
            listener.start();
            //FastLeaderElection是真正选举的地方
            //注意在这个构造方法里new了一个内部类new Messenger(manager)对象
            //同时Messenger又有两个内部类WorkerReceiver和WorkerSender(他们都继承了Thread)
            //后面看图讲解
            FastLeaderElection fle = new FastLeaderElection(this, qcm);
            //这个start()方法调用了messenger的start()方法
            //messenger.start()内部启动了WorkerReceiver和WorkerSender线程
            fle.start();
            //start()调用之后，立马返回;随后结束switch,跳出createElectionAlgorithm()方法
            le = fle;
            //总结一下: 退出createElectionAlgorithm()方法后,将会有3个线程在运行
            //1. QuorumCnxManager.Listener
            //2. FastLeaderElection.Messenger.WorkerReceiver
            //3. FastLeaderElection.Messenger.WorkerSender
        } else {
            LOG.error("Null listener when initializing cnx manager");
        }
        break;
    default:
        assert false;
    }
    return le;
}
```
上面那张图里的QuorumCnxManager是一个非常重要的类。它负责发起网络请求(将投票发送出去或者接受别的节点发送过来的投票)。

QuorumCnxManager有三个内部类Listener(线程)、RecvWoker(线程）、SendWorker(线程)。

Listener(线程)负责创建ServerSocket,用来接收投票信息。具体创建ServerSokcet的过程看代码注释。

当收到网络投票的时候QuorumCnxManager.Listener.ListenerHandler.acceptConnections()的client = serverSocket.accept();就会继续运行下去。最终如图中所示，会将网络投票数据添加到成员变量recvQueue<Message>(阻塞队列)中。

同时FastLeaderElection.start()内部启动了WorkerReceiver线程(见代码注释)，在不间断地poll QuorumCnxManager的recvQueue<Message>。拉到消息后，然后把消息封装成Notification放到FastLeaderElection中的recvQueue<Notification>中
详见如下代码
```java
//WorkerReceiver.run()
public void run() {
    Message response;
    while (!stop) {
        // Sleeps on receive
        try {
            response = manager.pollRecvQueue(3000, TimeUnit.MILLISECONDS);
            if (response == null) {
                continue;
            }
            ...
            // Instantiate Notification and set its attributes
            Notification n = new Notification();

            int rstate = response.buffer.getInt();
            long rleader = response.buffer.getLong();
            long rzxid = response.buffer.getLong();
            long relectionEpoch = response.buffer.getLong();
            long rpeerepoch;

            int version = 0x0;
            QuorumVerifier rqv = null;
            ...
            if (!validVoter(response.sid)) {
                ...
            } else {
                // Receive new message
                LOG.debug("Receive new notification message. My id = {}", self.getId());
                // State of peer that sent this message
                QuorumPeer.ServerState ackstate = QuorumPeer.ServerState.LOOKING;
                switch (rstate) {
                case 0:
                    ackstate = QuorumPeer.ServerState.LOOKING;
                    break;
                case 1:
                    ackstate = QuorumPeer.ServerState.FOLLOWING;
                    break;
                case 2:
                    ackstate = QuorumPeer.ServerState.LEADING;
                    break;
                case 3:
                    ackstate = QuorumPeer.ServerState.OBSERVING;
                    break;
                default:
                    continue;
                }

                n.leader = rleader;
                n.zxid = rzxid;
                n.electionEpoch = relectionEpoch;
                n.state = ackstate;
                n.sid = response.sid;
                n.peerEpoch = rpeerepoch;
                n.version = version;
                n.qv = rqv;
                ...
                //如果是LOOKING状态
                if (self.getPeerState() == QuorumPeer.ServerState.LOOKING) {
                    recvqueue.offer(n);
                    ....
                    if ((ackstate == QuorumPeer.ServerState.LOOKING)
                        && (n.electionEpoch < logicalclock.get())) {
                        ...
                    }
                } else {
                    ...
                }
            }
        } catch (InterruptedException e) {
            LOG.warn("Interrupted Exception while waiting for new message", e);
        }
    }//end while
}
```

## 选主逻辑




