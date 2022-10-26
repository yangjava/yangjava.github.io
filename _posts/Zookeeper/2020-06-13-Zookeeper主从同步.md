---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---



## leader和follower同步

zookeeper集群启动的时候，首先读取配置，接着开始选举，选举完成以后，每个server根据选举的结果设置自己的角色，角色设置完成后leader需要和所有的follower同步。上面一篇介绍了leader选举过程，这篇接着介绍启动过程中的leader和follower同步过程。

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

1. 如果当前server.myId等于选举出的leader的myId——也就是proposedLeader，则当前server就是leader，设置peerState为ServerState.LEADING
2. 否则判断当前server的具体角色，因为follower和observer都是learner，需要根据各自的配置来决定该server的状态(配置文件里面的key是peerType，可选的值是participant、observer，如果不配置learnerType默认是LearnerType.PARTICIPANT)
    1. 如果配置的learnerType是LearnerType.PARTICIPANT，则状态为ServerState.FOLLOWING
    2. 否则，状态为ServerState.OBSERVING

### 准备同步

leader开始工作的入口就是leader.lead方法，这里的leader是Leader的实例

准备的过程是：

- 创建leader的实例，Leader，构造方法中传入LeaderZooKeeperServer的实例
- 调用leader.lead
    - 加载ZKDatabase
    - 监听指定的端口（配置的用来监听learner连接请求的端口，配置的第一个冒号后的端口），接收来自follower的请求
    - while循环，检查当前选举的状态是否发生变化需要重新进行选举

同时，follower设置完自己的状态后，也开始进行类似leader的工作

- 创建follower，也就是Follower的实例，同时创建FollowerZooKeeperServer
- 建立和leader的连接

### 进行同步

同步的总体过程如下：![Zookeeper同步过程](png\zookeeper\Zookeeper同步过程.png)

在准备阶段完成follower连接到leader，具备通信状态

1. leader阻塞等待follower发来的第一个packet
    1. 校验packet类型是否是Leader.FOLLOWERINFO或者Leader.OBSERVERINFO
    2. 读取learner信息
        1. sid
        2. protocolVersion
        3. 校验follower的version不能比leader的version还要新
2. leader发送packet(Leader.LEADERINFO)给follower
3. follower收到Leader.LEADERINFO后给leader回复Leader.ACKEPOCH
4. leader根据follower ack的packet内容来决定同步的策略
    1. lastProcessedZxid == peerLastZxid，leader的zxid和follower的相同
    2. peerLastZxid > maxCommittedLog && !isPeerNewEpochZxid follower超前，删除follower多出的txlog部分
    3. (maxCommittedLog >= peerLastZxid) && (minCommittedLog <= peerLastZxid) follower落后于leader，处于leader的中间 同步(peerLaxtZxid, maxZxid]之间的commitlog给follower
    4. peerLastZxid < minCommittedLog && txnLogSyncEnabled follower落后于leader，使用txlog和commitlog同步给follower
    5. 接下来leader会不断的发送packet给follower，follower处理leader发来的每个packet
5. 同步完成后follower回复ack给leader
6. leader、follower进入正式处理客户端请求的while循环

zookeeper为了保证启动后leader和follower的数据一致，在启动的时候就进行数据同步，leader与follower数据传输的端口和leader选举的端口不一样。数据同步完成后就可以接受client的请求进行处理了。