---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---
# Zookeeper源码集群启动
使用集群模式，就是多添加几个Zookeeper节点。需要添加对应的配置文件，启动了集群模式才好去分析分布式环境下的leader的选举等源码

## 集群模式启动
集群模式下启动和单机启动有相似的地方，但是也有各自的特点。集群模式的配置方式和单机模式也是不一样的

### 新增Zookeeper配置文件
新增集群配置文件 /zoo_sample*.cfg
修改每个/zoo_sample*.cfg配置文件，具体修改内容如下，不同的服务使用不同的clientPort端口

/zoo_sample1.cfg
```text
tickTime=2000
initLimit=10
syncLimit=5
dataDir=D:\\zookeeper\\data1
clientPort=2181
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
admin.serverPort=8081
```

/zoo_sample2.cfg
```text
tickTime=2000
initLimit=10
syncLimit=5
dataDir=D:\\zookeeper\\data2
clientPort=2182
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
admin.serverPort=8082
```

/zoo_sample3.cfg
```text
tickTime=2000
initLimit=10
syncLimit=5
dataDir=D:\\zookeeper\\data3
clientPort=2183
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
admin.serverPort=8083
```
**为啥需要添加admin.serverPort?**
查阅Zookeeper3.5的官方文档，发现这是Zookeeper3.5的新特性
这是Zookeeper AdminServer，默认使用8080端口，如果不配置启动集群时，会报端口冲突错误。

### 新增Zookeeper数据文件
修改数据文件/zookeeper/data*文件
在每个数据目录下创建myid文件，文件内容分别写入1、2、3

### 创建集群启动
zk服务端的启动入口是QuorumPeerMain类中的main方法

```text
如果不配置的话，无法输出日志
VM options：
-Dlog4j.configuration=file:conf\log4j.properties

Program arguments：
conf\zoo_sample.cfg
```
创建多个集群启动，配置文件分别加载zoo_sample1.cfg，zoo_sample2.cfg，zoo_sample3.cfg
依次启动即可。

### 启动日志分析

启动第一个节点 ，会报错，因为其他两个节点还没启动，连接报错 Cannot open channel to 3 at election address /127.0.0.1:3890
```text
[QuorumPeerListener:QuorumCnxManager$Listener@929] - 1 is accepting connections now, my election bind port: /127.0.0.1:3888
[QuorumPeer[myid=1](plain=[0:0:0:0:0:0:0:0]:2181)(secure=disabled):QuorumPeer@1178] - LOOKING
[QuorumPeer[myid=1](plain=[0:0:0:0:0:0:0:0]:2181)(secure=disabled):FastLeaderElection@903] - New election. My id =  1, proposed zxid=0x0
[WorkerReceiver[myid=1]:FastLeaderElection@697] - Notification: 2 (message format version), 1 (n.leader), 0x0 (n.zxid), 0x1 (n.round), LOOKING (n.state), 1 (n.sid), 0x4 (n.peerEPoch), LOOKING (my state)0 (n.config version)
[QuorumPeer[myid=1](plain=[0:0:0:0:0:0:0:0]:2181)(secure=disabled):FastLeaderElection@937] - Notification time out: 400
[QuorumPeer[myid=1](plain=[0:0:0:0:0:0:0:0]:2181)(secure=disabled):FastLeaderElection@937] - Notification time out: 800
[QuorumPeer[myid=1](plain=[0:0:0:0:0:0:0:0]:2181)(secure=disabled):FastLeaderElection@937] - Notification time out: 1600
[QuorumConnectionThread-[myid=1]-2:QuorumCnxManager@381] - Cannot open channel to 3 at election address /127.0.0.1:3890
```

启动第二个节点后，节点正常了， 此时经过选举将节点二选举为leader节点，节点1为follower
```text
[main:QuorumCnxManager$Listener@878] - Election port bind maximum retries is 3
[QuorumPeerListener:QuorumCnxManager$Listener@929] - 2 is accepting connections now, my election bind port: /127.0.0.1:3889
[QuorumPeer[myid=2](plain=[0:0:0:0:0:0:0:0]:2182)(secure=disabled):QuorumPeer@1178] - LOOKING
[QuorumPeer[myid=2](plain=[0:0:0:0:0:0:0:0]:2182)(secure=disabled):FastLeaderElection@903] - New election. My id =  2, proposed zxid=0x0
[WorkerReceiver[myid=2]:FastLeaderElection@697] - Notification: 2 (message format version), 2 (n.leader), 0x0 (n.zxid), 0x1 (n.round), LOOKING (n.state), 2 (n.sid), 0x5 (n.peerEPoch), LOOKING (my state)0 (n.config version)
[WorkerReceiver[myid=2]:FastLeaderElection@697] - Notification: 2 (message format version), 1 (n.leader), 0x0 (n.zxid), 0x1 (n.round), LOOKING (n.state), 1 (n.sid), 0x5 (n.peerEPoch), LOOKING (my state)0 (n.config version)
[WorkerReceiver[myid=2]:FastLeaderElection@697] - Notification: 2 (message format version), 2 (n.leader), 0x0 (n.zxid), 0x1 (n.round), LOOKING (n.state), 1 (n.sid), 0x5 (n.peerEPoch), LOOKING (my state)0 (n.config version)
[QuorumPeer[myid=2](plain=[0:0:0:0:0:0:0:0]:2182)(secure=disabled):QuorumPeer@1266] - LEADING
```
节点1为follower
```text
[WorkerReceiver[myid=1]:FastLeaderElection@697] - Notification: 2 (message format version), 2 (n.leader), 0x0 (n.zxid), 0x1 (n.round), LOOKING (n.state), 2 (n.sid), 0x5 (n.peerEPoch), LOOKING (my state)0 (n.config version)
[WorkerReceiver[myid=1]:FastLeaderElection@697] - Notification: 2 (message format version), 2 (n.leader), 0x0 (n.zxid), 0x1 (n.round), LOOKING (n.state), 1 (n.sid), 0x5 (n.peerEPoch), LOOKING (my state)0 (n.config version)
[QuorumPeer[myid=1](plain=[0:0:0:0:0:0:0:0]:2181)(secure=disabled):QuorumPeer@1254] - FOLLOWING
[QuorumPeer[myid=1](plain=[0:0:0:0:0:0:0:0]:2181)(secure=disabled):Learner@91] - TCP NoDelay set to: true
```
启动第三个节点后， 加入到集群，节点三的同样也是follower节点
```text
[QuorumPeer[myid=3](plain=[0:0:0:0:0:0:0:0]:2183)(secure=disabled):ZooKeeperServer@181] - Created server with tickTime 2000 minSessionTimeout 4000 maxSessionTimeout 40000 datadir D:\zookeeper\data3\version-2 snapdir D:\zookeeper\data3\version-2
[QuorumPeer[myid=3](plain=[0:0:0:0:0:0:0:0]:2183)(secure=disabled):Follower@69] - FOLLOWING - LEADER ELECTION TOOK - 52 MS
[QuorumPeer[myid=3](plain=[0:0:0:0:0:0:0:0]:2183)(secure=disabled):Learner@395] - Getting a snapshot from leader 0x600000000
[QuorumPeer[myid=3](plain=[0:0:0:0:0:0:0:0]:2183)(secure=disabled):Learner@546] - Learner received NEWLEADER message
[QuorumPeer[myid=3](plain=[0:0:0:0:0:0:0:0]:2183)(secure=disabled):QuorumPeer@1590] - Dynamic reconfig is disabled, we don't store the last seen config.
[QuorumPeer[myid=3](plain=[0:0:0:0:0:0:0:0]:2183)(secure=disabled):FileTxnSnapLog@404] - Snapshotting: 0x600000000 to D:\zookeeper\data3\version-2\snapshot.600000000
[QuorumPeer[myid=3](plain=[0:0:0:0:0:0:0:0]:2183)(secure=disabled):Learner@529] - Learner received UPTODATE message
```

## Zookeeper集群概念介绍

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


### leader

投票选出的leader，可以处理读写请求。处理写请求的时候收集各个参与投票者的选票，来决出投票结果


### follower

作用：

1. 参与leader选举，可能被选为leader
2. 接收处理读请求
3. 接收写请求，转发给leader，并参与投票决定写操作是否提交


### observer

为了支持zk集群可扩展性，如果直接增加follower的数量，会导致投票的性能下降。也就是防止参与投票的server太多，导致leader选举收敛速度较慢，选举所需时间过长。

observer和follower类似，但是不参与选举和投票，

1. 接收处理读请求
2. 接收写请求，转发给leader，但是不参与投票，接收leader的投票结果，同步数据

这样在支持集群可扩展性的同时又不会影响投票的性能



