---
layout: post
categories: [Zookeeper]
description: none
keywords: Zookeeper
---
# Zookeeper数据处理

## Zookeeper数据同步机制
zookeeper的数据同步是为了保证每个节点的数据一致性，大致分为2个流程，一个是正常的客户端数据提交流程，二是集群中某个节点宕机后数据恢复流程。

### 正常客户端数据提交流程
客户端写入数据提交流程大致为：leader接受到客户端的写请求，然后同步给各个子节点。

但是有童鞋就产生疑惑了，客户端一般连接的是所有节点，客户端并不知道哪个是leader呀。的确，客户端会和所有的节点建立链接，并且发起写入请求是挨个遍历节点进行的，比如第一次是节点1，第二次是节点2。以此类推。
如果客户端正好链接的节点的角色是leader，那就按照上面的流程走。那如果链接的节点不是leader，是follower呢
如果Client选择链接的节点是Follower的话，这个Follower会把请求转给当前Leader，然后Leader会把请求广播给所有的Follower，每个节点同步完数据后告诉Leader数据已经同步完成（但是还未提交），当Leader收到半数以上的节点ACK确认消息后，那么Leader就认为这个数据可以提交了，会广播给所有的Follower节点，所有的节点就可以提交数据。整个同步工作就结束了。

### 节点宕机后的数据同步流程
当zookeeper集群中的Leader宕机后，会触发新的选举，选举期间，整个集群是没法对外提供服务的。直到选出新的Leader之后，才能重新提供服务。

我们重新回到3个节点的例子，zk1，zk2，zk3，其中z2为Leader，z1，z3为Follower，假设zk2宕机后，触发了重新选举，按照选举规则，z3当选Leader。这时整个集群只整下z1和z3，如果这时整个集群又创建了一个节点数据，接着z2重启。这时z2的数据肯定比z1和z3要旧，那这时该如何同步数据呢。

zookeeper是通过ZXID事务ID来确认的，ZXID是一个长度为64位的数字，其中低32位是按照数字来递增，任何数据的变更都会导致低32位数字简单加1。高32位是leader周期编号，每当选举出一个新的Leader时，新的Leader就从本地事务日志中取出ZXID，然后解析出高32位的周期编号，进行加1，再将低32位的全部设置为0。这样就保证了每次选举新的Leader后，保证了ZXID的唯一性而且是保证递增的。
查看某个数据节点的ZXID的命令为：
```shell
先进入zk client命令行
./bin/zkCli.sh -server 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183 
stat加上数据节点名称
stat /test
```
可以看到，有3个ZXID，这3个ZXID各自代表：
- cZxid：该节点创建时的事务ID
- mZxid：该节点最近一次更新时的事务ID
- pZxid：该节点的子节点的最新一次创建/更新/删除的事务ID
查看节点最新ZXID的命令为：
```shell
echo stat|nc 127.0.0.1 2181
这个命令需提前在cfg文件后追加：4lw.commands.whitelist=*，然后重启
```
ZXID就是当前节点最后一次事务的ID。

如果整个集群数据为一致的，那么所有节点的ZXID应该一样。所以zookeeper就通过这个有序的ZXID来确保各个节点之间的数据的一致性，带着之前的问题，如果Leader宕机之后，再重启后，会去和目前的Leader去比较最新的ZXID，如果节点的ZXID比最新Leader里的ZXID要小，那么就会去同步数据。

## Zookeeper主从同步
server刚启动的时候都处于LOOKING状态，选举完成后根据选举结果和对应配置进入对应的状态，设置状态的方法是：
```java
private void setPeerState(long proposedLeader, SyncedLearnerTracker voteSet) {
    ServerState ss = (proposedLeader == self.getId()) ?
            ServerState.LEADING: learningState();
    self.setPeerState(ss);
    if (ss == ServerState.LEADING) {
        leadingVoteSet = voteSet;
    }
}
```
- 如果当前server.myId等于选举出的leader的myId——也就是proposedLeader，则当前server就是leader，设置peerState为ServerState.LEADING
- 否则判断当前server的具体角色，因为follower和observer都是learner，需要根据各自的配置来决定该server的状态(配置文件里面的key是peerType，可选的值是participant、observer，如果不配置learnerType默认是LearnerType.PARTICIPANT)
- 如果配置的learnerType是LearnerType.PARTICIPANT，则状态为ServerState.FOLLOWING。否则，状态为ServerState.OBSERVING

## 准备同步
当节点在选举后角色确认为 leader 后将会进入 LEADING 状态，源码如下：
```java

public class QuorumPeer extends ZooKeeperThread implements QuorumStats.Provider {
    @Override
    public void run() {
        // ...
        try {
            /*
             * Main loop
             */
            while (running) {
                switch (getPeerState()) {
                // ... 
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
           // ...
        }
    }
    
```
QuorumPeer 在节点状态变更为 LEADING 之后会创建 leader 实例，并触发 lead 过程。
```java
void lead() throws IOException, InterruptedException {
    try {
        // 省略

        /**
         * 开启线程用于接收 follower 的连接请求
         */
        cnxAcceptor = new LearnerCnxAcceptor();
        cnxAcceptor.start();
        
        readyToStart = true;

        /**
         * 阻塞等待计算新的 epoch 值，并设置 zxid
         */
        long epoch = getEpochToPropose(self.getId(), self.getAcceptedEpoch());          
        zk.setZxid(ZxidUtils.makeZxid(epoch, 0));
        
        
        /**
         * 阻塞等待接收过半的 follower 节点发送的 ACKEPOCH 信息； 此时说明已经确定了本轮选举后 epoch 值
         */
        waitForEpochAck(self.getId(), leaderStateSummary);
        self.setCurrentEpoch(epoch);

        try {
            /**
             * 阻塞等待 超过半数的节点 follower 发送了 NEWLEADER ACK 信息；此时说明过半的 follower 节点已经完成数据同步
             */
            waitForNewLeaderAck(self.getId(), zk.getZxid(), LearnerType.PARTICIPANT);
        } catch (InterruptedException e) {
            // 省略
        }

        /**
         * 启动 zk server，此时集群可以对外正式提供服务
         */
        startZkServer();

        // 省略
}
```
从 lead 方法的实现可得知，leader 与 follower 在数据同步过程中会执行如下过程：
- 接收 follower 连接
- 计算新的 epoch 值
- 通知统一 epoch 值
- 数据同步
- 启动 zk server 对外提供服务

下面看下 follower 节点进入 FOLLOWING 状态后的操作：
```java
public void run() {
    try {
        /*
         * Main loop
         */
        while (running) {
            switch (getPeerState()) {
            case LOOKING:
                // 省略
            case OBSERVING:
                // 省略
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
                    setPeerState(ServerState.LOOKING);
                }
                break;
            }
        }
    } finally {
        
    }
}
```
QuorumPeer 在节点状态变更为 FOLLOWING 之后会创建 follower 实例，并触发 followLeader 过程。
```java
void followLeader() throws InterruptedException {
    // 省略
    try {
        QuorumServer leaderServer = findLeader();            
        try {
            /**
             * follower 与 leader 建立连接
             */
            connectToLeader(leaderServer.addr, leaderServer.hostname);

            /**
             * follower 向 leader 提交节点信息用于计算新的 epoch 值
             */
            long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);

            
            /**
             * follower 与 leader 数据同步
             */
            syncWithLeader(newEpochZxid);                
            
             // 省略

        } catch (Exception e) {
             // 省略
        }
    } finally {
        // 省略
    }
}
```
从 followLeader 方法的实现可得知，follower 与 leader 在数据同步过程中会执行如下过程：
- 请求连接 leader
- 提交节点信息计算新的 epoch 值
- 数据同步

## Leader Follower 建立通信
follower 请求连接
```java
protected QuorumServer findLeader() {
    QuorumServer leaderServer = null;
    // Find the leader by id
    Vote current = self.getCurrentVote();
    for (QuorumServer s : self.getView().values()) {
        if (s.id == current.getId()) {
            // Ensure we have the leader's correct IP address before
            // attempting to connect.
            s.recreateSocketAddresses();
            leaderServer = s;
            break;
        }
    }
    if (leaderServer == null) {
        LOG.warn("Couldn't find the leader with id = "
                + current.getId());
    }
    return leaderServer;
} 
```

```java
protected void connectToLeader(InetSocketAddress addr, String hostname)
            throws IOException, ConnectException, InterruptedException {
    sock = new Socket();        
    sock.setSoTimeout(self.tickTime * self.initLimit);
    for (int tries = 0; tries < 5; tries++) {
        try {
            sock.connect(addr, self.tickTime * self.syncLimit);
            sock.setTcpNoDelay(nodelay);
            break;
        } catch (IOException e) {
            if (tries == 4) {
                LOG.error("Unexpected exception",e);
                throw e;
            } else {
                LOG.warn("Unexpected exception, tries="+tries+
                        ", connecting to " + addr,e);
                sock = new Socket();
                sock.setSoTimeout(self.tickTime * self.initLimit);
            }
        }
        Thread.sleep(1000);
    }

    self.authLearner.authenticate(sock, hostname);

    leaderIs = BinaryInputArchive.getArchive(new BufferedInputStream(
            sock.getInputStream()));
    bufferedOutput = new BufferedOutputStream(sock.getOutputStream());
    leaderOs = BinaryOutputArchive.getArchive(bufferedOutput);
}
```
follower 会通过选举后的投票信息确认 leader 节点地址，并发起连接（总共有 5 次尝试连接的机会，若连接不通则重新进入选举过程）

leader 接收连接
```java
class LearnerCnxAcceptor extends ZooKeeperThread{
    private volatile boolean stop = false;

    public LearnerCnxAcceptor() {
        super("LearnerCnxAcceptor-" + ss.getLocalSocketAddress());
    }

    @Override
    public void run() {
        try {
            while (!stop) {
                try{
                    /**
                     * 接收 follower 的连接，并开启 LearnerHandler 线程用于处理二者之间的通信
                     */
                    Socket s = ss.accept();
                    s.setSoTimeout(self.tickTime * self.initLimit);
                    s.setTcpNoDelay(nodelay);

                    BufferedInputStream is = new BufferedInputStream(
                            s.getInputStream());
                    LearnerHandler fh = new LearnerHandler(s, is, Leader.this);
                    fh.start();
                } catch (SocketException e) {
                    // 省略
                } catch (SaslException e){
                    LOG.error("Exception while connecting to quorum learner", e);
                }
            }
        } catch (Exception e) {
            LOG.warn("Exception while accepting follower", e);
        }
    }
}
```
从 LearnerCnxAcceptor 实现可以看出 leader 节点在为每个 follower 节点连接建立之后都会为之分配一个 LearnerHandler 线程用于处理二者之间的通信。

### 计算新的 epoch 值
follower 在与 leader 建立连接之后，会发出 FOLLOWERINFO 信息,
```java
long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);
```

```java
protected long registerWithLeader(int pktType) throws IOException{
    /**
     * 发送 follower info 信息，包括 last zxid 和 sid
     */
    long lastLoggedZxid = self.getLastLoggedZxid();
    QuorumPacket qp = new QuorumPacket();                
    qp.setType(pktType);
    qp.setZxid(ZxidUtils.makeZxid(self.getAcceptedEpoch(), 0));
    
    /*
     * Add sid to payload
     */
    LearnerInfo li = new LearnerInfo(self.getId(), 0x10000);
    ByteArrayOutputStream bsid = new ByteArrayOutputStream();
    BinaryOutputArchive boa = BinaryOutputArchive.getArchive(bsid);
    boa.writeRecord(li, "LearnerInfo");
    qp.setData(bsid.toByteArray());
    
    /**
     * follower 向 leader 发送 FOLLOWERINFO 信息，包括 zxid，sid，protocol version
     */
    writePacket(qp, true);
    
    // 省略
} 
```

接下来我们看下 leader 在接收到 FOLLOWERINFO 信息之后做什么(参考 LearnerHandler)
```java
public void run() {
    try {
        // 省略
        /**
         * leader 接收 follower 发送的 FOLLOWERINFO 信息，包括 follower 节点的 zxid，sid，protocol version
         * @see Learner.registerWithleader()
         */
        QuorumPacket qp = new QuorumPacket();
        ia.readRecord(qp, "packet");

        byte learnerInfoData[] = qp.getData();
        if (learnerInfoData != null) {
            if (learnerInfoData.length == 8) {
                // 省略
            } else {
                /**
                 * 高版本的 learnerInfoData 包括 long 类型的 sid, int 类型的 protocol version 占用 12 字节
                 */
                LearnerInfo li = new LearnerInfo();
                ByteBufferInputStream.byteBuffer2Record(ByteBuffer.wrap(learnerInfoData), li);
                this.sid = li.getServerid();
                this.version = li.getProtocolVersion();
            }
        }

        /**
         * 通过 follower 发送的 zxid，解析出 foloower 节点的 epoch 值
         */
        long lastAcceptedEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());
        
        long peerLastZxid;
        StateSummary ss = null;
        long zxid = qp.getZxid();

        /**
         * 阻塞等待计算新的 epoch 值
         */
        long newEpoch = leader.getEpochToPropose(this.getSid(), lastAcceptedEpoch);
      
        // 省略
    }
```
从上述代码可知，leader 在接收到 follower 发送的 FOLLOWERINFO 信息之后，会解析出 follower 节点的 acceptedEpoch 值并参与到新的 epoch 值计算中。 （具体计算逻辑参考方法 getEpochToPropose）
```java
public long getEpochToPropose(long sid, long lastAcceptedEpoch) throws InterruptedException, IOException {
    synchronized(connectingFollowers) {
        if (!waitingForNewEpoch) {
            return epoch;
        }
        // epoch 用来记录计算后的选举周期值
        // follower 或 leader 的 acceptedEpoch 值与 epoch 比较；若前者大则将其加一
        if (lastAcceptedEpoch >= epoch) {
            epoch = lastAcceptedEpoch+1;
        }
        // connectingFollowers 用来记录与 leader 已连接的 follower
        connectingFollowers.add(sid);
        QuorumVerifier verifier = self.getQuorumVerifier();
        // 判断是否已计算出新的 epoch 值的条件是 leader 已经参与了 epoch 值计算，以及超过一半的节点参与了计算
        if (connectingFollowers.contains(self.getId()) && 
                                        verifier.containsQuorum(connectingFollowers)) {
            // 将 waitingForNewEpoch 设置为 false 说明不需要等待计算新的 epoch 值了
            waitingForNewEpoch = false;
            // 设置 leader 的 acceptedEpoch 值
            self.setAcceptedEpoch(epoch);
            // 唤醒 connectingFollowers wait 的线程
            connectingFollowers.notifyAll();
        } else {
            long start = Time.currentElapsedTime();
            long cur = start;
            long end = start + self.getInitLimit()*self.getTickTime();
            while(waitingForNewEpoch && cur < end) {
                // 若未完成新的 epoch 值计算则阻塞等待
                connectingFollowers.wait(end - cur);
                cur = Time.currentElapsedTime();
            }
            if (waitingForNewEpoch) {
                throw new InterruptedException("Timeout while waiting for epoch from quorum");        
            }
        }
        return epoch;
    }
}
```
从方法 getEpochToPropose 可知 leader 会收集集群中过半的 follower acceptedEpoch 信息后，选出一个最大值然后加 1 就是 newEpoch 值； 在此过程中 leader 会进入阻塞状态直到过半的 follower 参与到计算才会进入下一阶段。

### 通知新的 epoch 值
leader 在计算出新的 newEpoch 值后，会进入下一阶段发送 LEADERINFO 信息 （同样参考 LearnerHandler）
```java
public void run() {
    try {
        // 省略

        /**
         * 阻塞等待计算新的 epoch 值
         */
        long newEpoch = leader.getEpochToPropose(this.getSid(), lastAcceptedEpoch);
            
        if (this.getVersion() < 0x10000) {
            // we are going to have to extrapolate the epoch information
            long epoch = ZxidUtils.getEpochFromZxid(zxid);
            ss = new StateSummary(epoch, zxid);
            // fake the message
            leader.waitForEpochAck(this.getSid(), ss);
        } else {
            byte ver[] = new byte[4];
            ByteBuffer.wrap(ver).putInt(0x10000);
            /**
             * 计算出新的 epoch 值后，leader 向 follower 发送 LEADERINFO 信息；包括新的 newEpoch
             */
            QuorumPacket newEpochPacket = new QuorumPacket(Leader.LEADERINFO, ZxidUtils.makeZxid(newEpoch, 0), ver, null);
            oa.writeRecord(newEpochPacket, "packet");
            bufferedOutput.flush();

            // 省略
        }
    }
    // 省略
}

```

```java
protected long registerWithLeader(int pktType) throws IOException{
    // 省略

    /**
     * follower 向 leader 发送 FOLLOWERINFO 信息，包括 zxid，sid，protocol version
     */
    writePacket(qp, true);

    /**
     * follower 接收 leader 发送的 LEADERINFO 信息
     */
    readPacket(qp);

    /**
     * 解析 leader 发送的 new epoch 值
     */        
    final long newEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());
    if (qp.getType() == Leader.LEADERINFO) {
        // we are connected to a 1.0 server so accept the new epoch and read the next packet
        leaderProtocolVersion = ByteBuffer.wrap(qp.getData()).getInt();
        byte epochBytes[] = new byte[4];
        final ByteBuffer wrappedEpochBytes = ByteBuffer.wrap(epochBytes);

        /**
         * new epoch > current accepted epoch 则更新 acceptedEpoch 值
         */
        if (newEpoch > self.getAcceptedEpoch()) {
            wrappedEpochBytes.putInt((int)self.getCurrentEpoch());
            self.setAcceptedEpoch(newEpoch);
        } else if (newEpoch == self.getAcceptedEpoch()) {           
            wrappedEpochBytes.putInt(-1);
        } else {
            throw new IOException("Leaders epoch, " + newEpoch + " is less than accepted epoch, " + self.getAcceptedEpoch());
        }

        /**
         * follower 向 leader 发送 ACKEPOCH 信息，包括 last zxid
         */
        QuorumPacket ackNewEpoch = new QuorumPacket(Leader.ACKEPOCH, lastLoggedZxid, epochBytes, null);
        writePacket(ackNewEpoch, true);
        return ZxidUtils.makeZxid(newEpoch, 0);
    } 
} 
```
从上述代码可以看出在完成 newEpoch 值计算后的 leader 与 follower 的交互过程：
- leader 向 follower 发送 LEADERINFO 信息，告知 follower 新的 epoch 值
- follower 接收解析 LEADERINFO 信息，若 new epoch 值大于 current accepted epoch 值则更新 acceptedEpoch
- follower 向 leader 发送 ACKEPOCH 信息，反馈 leader 已收到新的 epoch 值，并附带 follower 节点的 last zxid

## 数据同步
LearnerHandler 中 leader 在收到过半的 ACKEPOCH 信息之后将进入数据同步阶段
```java
public void run() {
        try {
            // 省略
            // peerLastZxid 为 follower 的 last zxid
            peerLastZxid = ss.getLastZxid();
            
            /* the default to send to the follower */
            int packetToSend = Leader.SNAP;
            long zxidToSend = 0;
            long leaderLastZxid = 0;
            /** the packets that the follower needs to get updates from **/
            long updates = peerLastZxid;
           
            ReentrantReadWriteLock lock = leader.zk.getZKDatabase().getLogLock();
            ReadLock rl = lock.readLock();
            try {
                rl.lock();        
                final long maxCommittedLog = leader.zk.getZKDatabase().getmaxCommittedLog();
                final long minCommittedLog = leader.zk.getZKDatabase().getminCommittedLog();

                LinkedList<Proposal> proposals = leader.zk.getZKDatabase().getCommittedLog();

                if (peerLastZxid == leader.zk.getZKDatabase().getDataTreeLastProcessedZxid()) {
                    /**
                     * follower 与 leader 的 zxid 相同说明 二者数据一致；同步方式为差量同步 DIFF，同步的zxid 为 peerLastZxid， 也就是不需要同步
                     */
                    packetToSend = Leader.DIFF;
                    zxidToSend = peerLastZxid;
                } else if (proposals.size() != 0) {
                    // peerLastZxid 介于 minCommittedLog ，maxCommittedLog 中间
                    if ((maxCommittedLog >= peerLastZxid)
                            && (minCommittedLog <= peerLastZxid)) {
                        /**
                         * 在遍历 proposals 时，用来记录上一个 proposal 的 zxid
                         */
                        long prevProposalZxid = minCommittedLog;

                        boolean firstPacket=true;
                        packetToSend = Leader.DIFF;
                        zxidToSend = maxCommittedLog;

                        for (Proposal propose: proposals) {
                            // 跳过 follower 已经存在的提案
                            if (propose.packet.getZxid() <= peerLastZxid) {
                                prevProposalZxid = propose.packet.getZxid();
                                continue;
                            } else {
                                if (firstPacket) {
                                    firstPacket = false;
                                    if (prevProposalZxid < peerLastZxid) {
                                        /**
                                         * 此时说明有部分 proposals 提案在 leader 节点上不存在，则需告诉 follower 丢弃这部分 proposals
                                         * 也就是告诉 follower 先执行回滚 TRUNC ，需要回滚到 prevProposalZxid 处，也就是 follower 需要丢弃 prevProposalZxid ~ peerLastZxid 范围内的数据
                                         * 剩余的 proposals 则通过 DIFF 进行同步
                                         */
                                        packetToSend = Leader.TRUNC;                                        
                                        zxidToSend = prevProposalZxid;
                                        updates = zxidToSend;
                                    }
                                }

                                /**
                                 * 将剩余待 DIFF 同步的提案放入到队列中，等待发送
                                 */
                                queuePacket(propose.packet);
                                /**
                                 * 每个提案后对应一个 COMMIT 报文
                                 */
                                QuorumPacket qcommit = new QuorumPacket(Leader.COMMIT, propose.packet.getZxid(),
                                        null, null);
                                queuePacket(qcommit);
                            }
                        }
                    } else if (peerLastZxid > maxCommittedLog) {                    
                        /**
                         * follower 的 zxid 比 leader 大 ，则告诉 follower 执行 TRUNC 回滚
                         */
                        packetToSend = Leader.TRUNC;
                        zxidToSend = maxCommittedLog;
                        updates = zxidToSend;
                    } else {
                    }
                } 

            } finally {
                rl.unlock();
            }

             QuorumPacket newLeaderQP = new QuorumPacket(Leader.NEWLEADER,
                    ZxidUtils.makeZxid(newEpoch, 0), null, null);
             if (getVersion() < 0x10000) {
                oa.writeRecord(newLeaderQP, "packet");
            } else {
                 // 数据同步完成之后会发送 NEWLEADER 信息
                queuedPackets.add(newLeaderQP);
            }
            bufferedOutput.flush();
            //Need to set the zxidToSend to the latest zxid
            if (packetToSend == Leader.SNAP) {
                zxidToSend = leader.zk.getZKDatabase().getDataTreeLastProcessedZxid();
            }
            /**
             * 发送数据同步方式信息，告诉 follower 按什么方式进行数据同步
             */
            oa.writeRecord(new QuorumPacket(packetToSend, zxidToSend, null, null), "packet");
            bufferedOutput.flush();
            
            /* if we are not truncating or sending a diff just send a snapshot */
            if (packetToSend == Leader.SNAP) {
                /**
                 * 如果是全量同步的话，则将 leader 本地数据序列化写入 follower 的输出流
                 */
                leader.zk.getZKDatabase().serializeSnapshot(oa);
                oa.writeString("BenWasHere", "signature");
            }
            bufferedOutput.flush();
            
            /**
             * 开启个线程执行 packet 发送
             */
            sendPackets();
            
            /**
             * 接收 follower ack 响应
             */
            qp = new QuorumPacket();
            ia.readRecord(qp, "packet");

            /**
             * 阻塞等待过半的 follower ack
             */
            leader.waitForNewLeaderAck(getSid(), qp.getZxid(), getLearnerType());

            /**
             * leader 向 follower 发送 UPTODATE，告知其可对外提供服务
             */
            queuedPackets.add(new QuorumPacket(Leader.UPTODATE, -1, null, null));

            // 省略
        } 
    }
```
从上述代码可以看出 leader 和 follower 在进行数据同步时会通过 peerLastZxid 与 maxCommittedLog， minCommittedLog 两个值比较最终决定数据同步方式。

### DIFF(差异化同步)
- follower 的 peerLastZxid 等于 leader 的 peerLastZxid,此时说明 follower 与 leader 数据一致，采用 DIFF 方式同步，也即是无需同步

- follower 的 peerLastZxid 介于 maxCommittedLog， minCommittedLog 两者之间
此时说明 follower 与 leader 数据存在差异，需对差异的部分进行同步；首先 leader 会向 follower 发送 DIFF 报文告知其同步方式，随后会发送差异的提案及提案提交报文

示例： 假设 leader 节点的提案缓存队列对应的 zxid 依次是：
```
 0x500000001, 0x500000002, 0x500000003, 0x500000004, 0x500000005
```
而 follower 节点的 peerLastZxid 为 0x500000003，则需要将 0x500000004， 0x500000005 两个提案进行同步；那么数据包发送过程如下表：
```
报文类型	ZXID
DIFF	0x500000005
PROPOSAL	0x500000004
COMMIT	0x500000004
PROPOSAL	0x500000005
COMMIT	0x500000005
```
TRUNC+DIFF(先回滚再差异化同步)
在上文 DIFF 差异化同步时会存在一个特殊场景就是 虽然 follower 的 peerLastZxid 介于 maxCommittedLog， minCommittedLog 两者之间，但是 follower 的 peerLastZxid 在 leader 节点中不存在； 此时 leader 需告知 follower 先回滚到 peerLastZxid 的前一个 zxid, 回滚后再进行差异化同步。
TRUNC(回滚同步)
若 follower 的 peerLastZxid 大于 leader 的 maxCommittedLog，则告知 follower 回滚至 maxCommittedLog； 该场景可以认为是 TRUNC+DIFF 的简化模式
SNAP(全量同步)
若 follower 的 peerLastZxid 小于 leader 的 minCommittedLog 或者 leader 节点上不存在提案缓存队列时，将采用 SNAP 全量同步方式。 该模式下 leader 首先会向 follower 发送 SNAP 报文，随后从内存数据库中获取全量数据序列化传输给 follower， follower 在接收全量数据后会进行反序列化加载到内存数据库中。

leader 在完成数据同步之后，会向 follower 发送 NEWLEADER 报文，在收到过半的 follower 响应的 ACK 之后此时说明过半的节点完成了数据同步，接下来 leader 会向 follower 发送 UPTODATE 报文告知 follower 节点可以对外提供服务了，此时 leader 会启动 zk server 开始对外提供服务。

## FOLLOWER 数据同步
下面我们在看下数据同步阶段 FOLLOWER 是如何处理的，参考 Learner.syncWithLeader
```java
protected void syncWithLeader(long newLeaderZxid) throws IOException, InterruptedException{
        QuorumPacket ack = new QuorumPacket(Leader.ACK, 0, null, null);
        QuorumPacket qp = new QuorumPacket();
        long newEpoch = ZxidUtils.getEpochFromZxid(newLeaderZxid);

        /**
         * 接收 leader 发送的数据同步方式报文
         */
        readPacket(qp);
        
        synchronized (zk) {
            if (qp.getType() == Leader.DIFF) {
                
            }
            else if (qp.getType() == Leader.SNAP) {
                // 执行加载全量数据
            } else if (qp.getType() == Leader.TRUNC) {
                // 执行回滚
            }
            else {
            
            }
            
            outerLoop:
            while (self.isRunning()) {
                readPacket(qp);
                switch(qp.getType()) {
                case Leader.PROPOSAL:
                    // 处理提案
                    break;
                case Leader.COMMIT:
                    // commit proposal
                    break;
                case Leader.INFORM:
                    // 忽略
                    break;
                case Leader.UPTODATE:
                    // 设置 zk server
                    self.cnxnFactory.setZooKeeperServer(zk);
                    // 退出循环                
                    break outerLoop;
                case Leader.NEWLEADER: // Getting NEWLEADER here instead of in discovery 
                    /**
                     * follower 响应 NEWLEADER ACK
                     */
                    writePacket(new QuorumPacket(Leader.ACK, newLeaderZxid, null, null), true);
                    break;
                }
            }
        }
        ack.setZxid(ZxidUtils.makeZxid(newEpoch, 0));
        writePacket(ack, true);
        // 启动 zk server
        zk.startup();
        
    }
```
从上述代码中可以看出 follower 在数据同步阶段的处理流程如下：
- follower 接收 leader 发送的数据同步方式（DIFF/TRUNC/SANP）报文并进行相应处理
- 当 follower 收到 leader 发送的 NEWLEADER 报文后，会向 leader 响应 ACK (leader 在收到过半的 ACK 消息之后会发送 UPTODATE)
- 当 follower 收到 leader 发送的 UPTODATE 报文后，说明此时可以对外提供服务，此时将启动 zk server

## Zookeeper处理写请求
zk为了保证分布式数据一致性，使用ZAB协议，在客户端发起一次写请求的时候时候，假设该请求请求到的是follower，follower不会直接处理这个请求，而是转发给leader，由leader发起投票决定该请求最终能否执行成功，所以整个过程client、被请求的follower、leader、其他follower都参与其中。

创建流程为
- follower接受请求，解析请求
- follower通过FollowerRequestProcessor将请求转发给leader
- leader接收请求
- leader发送proposal给follower
- follower收到请求记录txlog、snapshot
- follower发送ack给leader
- leader收到ack后进行commit，并且通知所有的learner，发送commit packet给所有的learner

### 客户端发起写请求
在客户端启动的时候会创建Zookeeper实例，client会连接到server，后面client在创建节点的时候就可以直接和server通信，client发起创建创建节点请求的过程是：
```java
org.apache.zookeeper.ZooKeeper#create(java.lang.String, byte[], java.util.List<org.apache.zookeeper.data.ACL>, org.apache.zookeeper.CreateMode)
org.apache.zookeeper.ClientCnxn#submitRequest
org.apache.zookeeper.ClientCnxn#queuePacket
```
在ZooKeeper#create方法中构造请求的request。在ClientCnxn#queuePacket方法中将request封装到packet中，将packet放入发送队列outgoingQueue中等待发送
SendThread不断从发送队列outgoingQueue中取出packet发送。通过 packet.wait等待server返回。

### follower和leader交互过程
client发出请求后，follower会接收并处理该请求。选举结束后follower确定了自己的角色为follower，一个端口和client通信，一个端口和leader通信。监听到来自client的连接口建立新的session，监听对应的socket上的读写事件，如果client有请求发到follower，follower会用下面的方法处理
```java
org.apache.zookeeper.server.NIOServerCnxn#readPayload
org.apache.zookeeper.server.NIOServerCnxn#readRequest
```
readPayload这个方法里面会判断是连接请求还是非连接请求，这里从处理非连接请求开始。

### follower接收client请求
follower收到请求之后，先请求请求的opCode类型（这里是create）构造对应的request，然后交给第一个processor执行，follower的第一个processor是FollowerRequestProcessor。

### follower转发请求给leader
由于在zk中follower是不能处理写请求的，需要转交给leader处理，在FollowerRequestProcessor中将请求转发给leader
FollowerRequestProcessor是一个线程在zk启动的时候就开始运行，主要逻辑在run方法里面先把请求提交给CommitProcessor（后面leader发送给follower的commit请求对应到这里），然后将请求转发给leader，转发给leader的过程就是构造一个QuorumPacket，通过之前选举通信的端口发送给leader。

### leader接收follower请求
leader获取leader地位以后，启动learnhandler，然后一直在LearnerHandler#run循环，接收来自learner的packet，处理流程是：
```java
processRequest(Request):1003, org.apache.zookeeper.server.PrepRequestProcessor.java
submitLearnerRequest(Request):150, org.apache.zookeeper.server.quorum.LeaderZooKeeperServer.java
run():625, org.apache.zookeeper.server.quorum.LearnerHandler.java
```
handler判断是REQUEST请求的话交给leader的processor链处理，将请求放入org.apache.zookeeper.server.PrepRequestProcessor#submittedRequests，即leader的第一个processor。这个processor也是一个线程，从submittedRequests中不断拿出请求处理

### leader 发送proposal给follower
ProposalRequestProcessor主要作用就是讲请求交给下一个processor并且发起投票，将proposal发送给所有的follower。
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

### follower 收到proposal
follower处理proposal请求的调用堆栈
```java
processRequest(Request):214, org.apache.zookeeper.server.SyncRequestProcessor.java
 logRequest(TxnHeader, Record):89, org.apache.zookeeper.server.quorumFollowerZooKeeperServer.java
 processPacket(QuorumPacket):147, org.apache.zookeeper.server.quorum.Follower.java
 followLeader():102, org.apache.zookeeper.server.quorum.Follower.java
 run():1199, org.apache.zookeeper.server.quorum.QuorumPeer.java
```
将请求放入org.apache.zookeeper.server.SyncRequestProcessor#queuedRequests

### follower 发送ack



