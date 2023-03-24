---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---
# Zookeeper工作原理

## ZooKeeper架构
- Leader
负责进行投票的发起和决议，更新系统状态，Leader 是由选举产生;
- Follower 
用于接受客户端请求并向客户端返回结果，在选主过程中参与投票;
- Observer
可以接受客户端连接，接受读写请求，写请求转发给 Leader，但 Observer 不参加投票过程 ，只同步 Leader 的状态，Observer 的目的是为了扩展系统， 提高读取速度 。

## ZooKeeper工作原理
Zookeeper的核心是原子广播机制，这个机制保证了各个server之间的同步。实现这个机制的协议叫做 Zab协议 。Zab协议有两种模式，它们分别是恢复模式和广播模式。

### 恢复模式
当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数server完成了和leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和server具有相同的系统状态。

### 广播模式
一旦Leader已经和多数的Follower进行了状态同步后，他就可以开始广播消息了，即进入广播状态。这时候当一个Server加入ZooKeeper服务中，它会在恢复模式下启动，发现Leader，并和Leader进行状态同步。待到同步结束，它也参与消息广播。ZooKeeper服务一直维持在Broadcast状态，直到Leader崩溃了或者Leader失去了大部分的Followers支持。

Broadcast模式极其类似于分布式事务中的2pc（two-phrase commit 两阶段提交）：即Leader提起一个决议，由Followers进行投票，Leader对投票结果进行计算决定是否通过该决议，如果通过执行该决议（事务），否则什么也不做。

在广播模式ZooKeeper Server会接受Client请求， 所有的写请求都被转发给leader ，再由领导者将更新广播给跟随者，而 查询和维护管理命令不用跟leader打交道 。当半数以上的跟随者已经将修改持久化之后，领导者才会提交这个更新，然后客户端才会收到一个更新成功的响应。这个用来达成共识的协议被设计成具有原子性，因此每个修改要么成功要么失败。

## 读操作
Leader/Follower/Observer 都可直接处理读请求，从本地内存中读取数据并返回给客户端即可。

由于处理读请求不需要服务器之间的交互，Follower/Observer 越多，整体系统的读请求吞吐量越大，也即读性能越好。


## 写操作
所有的写请求实际上都要交给 Leader 处理。Leader 将写请求以事务形式发给所有 Follower 并等待 ACK，一旦收到半数以上 Follower 的 ACK，即认为写操作成功。

## 写 Leader
通过 Leader 进行写操作，主要分为五步：
- 客户端向 Leader 发起写请求。 
- Leader 将写请求以事务 Proposal 的形式发给所有 Follower 并等待 ACK。
- Follower 收到 Leader 的事务 Proposal 后返回 ACK。
- Leader 得到过半数的 ACK（Leader 对自己默认有一个 ACK）后向所有的 Follower 和 Observer 发送 Commmit。
- Leader 将处理结果返回给客户端。

**注意**
- Leader 不需要得到 Observer 的 ACK，即 Observer 无投票权。
- Leader 不需要得到所有 Follower 的 ACK，只要收到过半的 ACK 即可，同时 Leader 本身对自己有一个 ACK。
- Observer 虽然无投票权，但仍须同步 Leader 的数据从而在处理读请求时可以返回尽可能新的数据。

## 写 Follower/Observer
Follower/Observer 均可接受写请求，但不能直接处理，而需要将写请求转发给 Leader 处理。
除了多了一步请求转发，其它流程与直接写 Leader 无任何区别。

## 事务
对于来自客户端的每个更新请求，ZooKeeper 具备严格的顺序访问控制能力。 为了保证事务的顺序一致性，ZooKeeper 采用了递增的事务 id 号（zxid）来标识事务。

Leader 服务会为每一个 Follower 服务器分配一个单独的队列，然后将事务 Proposal 依次放入队列中，并根据 FIFO(先进先出) 的策略进行消息发送。**Follower 服务在接收到 Proposal 后，会将其以事务日志的形式写入本地磁盘中，并在写入成功后反馈给 Leader 一个 Ack 响应。当 Leader 接收到超过半数 Follower 的 Ack 响应后，就会广播一个 Commit 消息给所有的 Follower 以通知其进行事务提交，之后 Leader 自身也会完成对事务的提交。而每一个 Follower 则在接收到 Commit 消息后，完成事务的提交。

所有的提议（proposal）都在被提出的时候加上了 zxid。zxid 是一个 64 位的数字，它的高 32 位是 epoch 用来标识 Leader 关系是否改变，每次一个 Leader 被选出来，它都会有一个新的 epoch，标识当前属于那个 leader 的统治时期。低 32 位用于递增计数。

详细过程如下：
- Leader 等待 Server 连接；
- Follower 连接 Leader，将最大的 zxid 发送给 Leader；
- Leader 根据 Follower 的 zxid 确定同步点；
- 完成同步后通知 follower 已经成为 uptodate 状态；
- Follower 收到 uptodate 消息后，又可以重新接受 client 的请求进行服务了。

## Watch
客户端注册监听它关心的 znode，当 znode 状态发生变化（数据变化、子节点增减变化）时，ZooKeeper 服务会通知客户端。

客户端和服务端保持连接一般有两种形式：
- 客户端向服务端不断轮询
- 服务端向客户端推送状态

Zookeeper 的选择是服务端主动推送状态，也就是观察机制（ Watch ）。 ZooKeeper 的观察机制允许用户在指定节点上针对感兴趣的事件注册监听，当事件发生时，监听器会被触发，并将事件信息推送到客户端。

客户端使用 getData 等接口获取 znode 状态时传入了一个用于处理节点变更的回调，那么服务端就会主动向客户端推送节点的变更：

从这个方法中传入的 Watcher 对象实现了相应的 process 方法，每次对应节点出现了状态的改变，WatchManager 都会通过以下的方式调用传入 Watcher 的方法：

```
Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> supress) {
    WatchedEvent e = new WatchedEvent(type, KeeperState.SyncConnected, path);
    Set<Watcher> watchers;
    synchronized (this) {
        watchers = watchTable.remove(path);
    }
    for (Watcher w : watchers) {
        w.process(e);
    }
    return
```

Zookeeper 中的所有数据其实都是由一个名为 DataTree 的数据结构管理的，所有的读写数据的请求最终都会改变这颗树的内容，在发出读请求时可能会传入 Watcher 注册一个回调函数，而写请求就可能会触发相应的回调，由 WatchManager 通知客户端数据的变化。

通知机制的实现其实还是比较简单的，通过读请求设置 Watcher 监听事件，写请求在触发事件时就能将通知发送给指定的客户端。

## 会话
ZooKeeper 客户端通过 TCP 长连接连接到 ZooKeeper 服务集群。会话 (Session) 从第一次连接开始就已经建立，之后通过心跳检测机制来保持有效的会话状态。通过这个连接，客户端可以发送请求并接收响应，同时也可以接收到 Watch 事件的通知。

每个 ZooKeeper 客户端配置中都配置了 ZooKeeper 服务器集群列表。启动时，客户端会遍历列表去尝试建立连接。如果失败，它会尝试连接下一个服务器，依次类推。

一旦一台客户端与一台服务器建立连接，这台服务器会为这个客户端创建一个新的会话。**每个会话都会有一个超时时间，若服务器在超时时间内没有收到任何请求，则相应会话被视为过期。**一旦会话过期，就无法再重新打开，且任何与该会话相关的临时 znode 都会被删除。

通常来说，会话应该长期存在，而这需要由客户端来保证。客户端可以通过心跳方式（ping）来保持会话不过期。

ZooKeeper 的会话具有四个属性：
- sessionID
会话 ID，唯一标识一个会话，每次客户端创建新的会话时，Zookeeper 都会为其分配一个全局唯一的 sessionID。
- TimeOut
会话超时时间，客户端在构造 Zookeeper 实例时，会配置 sessionTimeout 参数用于指定会话的超时时间，Zookeeper 客户端向服务端发送这个超时时间后，服务端会根据自己的超时时间限制最终确定会话的超时时间。
- TickTime
下次会话超时时间点，为了便于 Zookeeper 对会话实行”分桶策略”管理，同时为了高效低耗地实现会话的超时检查与清理，Zookeeper 会为每个会话标记一个下次会话超时时间点，其值大致等于当前时间加上 TimeOut。
- isClosing
标记一个会话是否已经被关闭，当服务端检测到会话已经超时失效时，会将该会话的 isClosing 标记为”已关闭”，这样就能确保不再处理来自该会话的新请求了。

Zookeeper 的会话管理主要是通过 SessionTracker 来负责，其采用了**分桶策略**（将类似的会话放在同一区块中进行管理）进行管理，以便 Zookeeper 对会话进行不同区块的隔离处理以及同一区块的统一处理。

## ZAB 协议
ZooKeeper 并没有直接采用 Paxos 算法，而是采用了名为 ZAB 的一致性协议。ZAB 协议不是 Paxos 算法，只是比较类似，二者在操作上并不相同。
Zab协议 的全称是 Zookeeper Atomic Broadcast（原子广播）。 Zookeeper 是通过 Zab 协议来保证分布式事务的最终一致性。
ZAB 协议是 Zookeeper 专门设计的一种支持崩溃恢复的原子广播协议。 ZAB 协议是 ZooKeeper 的数据一致性和高可用解决方案。

ZAB 协议定义了两个可以无限循环的流程：
- 选举 Leader：用于故障恢复，从而保证高可用。
- 原子广播：用于主从同步，从而保证数据一致性。

Zookeeper 客户端会随机的链接到 zookeeper 集群中的一个节点，如果是读请求，就直接从当前节点中读取数据；如果是写请求，那么节点就会向 Leader 提交事务，Leader 接收到事务提交，会广播该事务，只要超过半数节点写入成功，该事务就会被提交。
**Zab 协议的特性**：
1）Zab 协议需要确保那些**已经在 Leader 服务器上提交（Commit）的事务最终被所有的服务器提交**。
2）Zab 协议需要确保**丢弃那些只在 Leader 上被提出而没有被提交的事务**。

### Zab 协议实现的作用
- **使用一个单一的主进程（Leader）来接收并处理客户端的事务请求**（也就是写请求），并采用了Zab的原子广播协议，将服务器数据的状态变更以 **事务proposal** （事务提议）的形式广播到所有的副本（Follower）进程上去。
- **保证一个全局的变更序列被顺序引用**。
Zookeeper是一个树形结构，很多操作都要先检查才能确定是否可以执行，比如P1的事务t1可能是创建节点"/a"，t2可能是创建节点"/a/bb"，只有先创建了父节点"/a"，才能创建子节点"/a/b"。
为了保证这一点，Zab要保证同一个Leader发起的事务要按顺序被apply，同时还要保证只有先前Leader的事务被apply之后，新选举出来的Leader才能再次发起事务。
- **当主进程出现异常的时候，整个zk集群依旧能正常工作**。

### Zab协议原理
Zab协议要求每个 Leader 都要经历三个阶段：**发现，同步，广播**。
- **发现**：要求zookeeper集群必须选举出一个 Leader 进程，同时 Leader 会维护一个 Follower 可用客户端列表。将来客户端可以和这些 Follower节点进行通信。
- **同步**：Leader 要负责将本身的数据与 Follower 完成同步，做到多副本存储。这样也是提现了CAP中的高可用和分区容错。Follower将队列中未处理完的请求消费完成后，写入本地事务日志中。
- **广播**：Leader 可以接受客户端新的事务Proposal请求，将新的Proposal请求广播给所有的 Follower。

### Zab协议核心
Zab协议的核心：**定义了事务请求的处理方式**
- 所有的事务请求必须由一个全局唯一的服务器来协调处理，这样的服务器被叫做 **Leader服务器**。其他剩余的服务器则是 **Follower服务器**。
- Leader服务器 负责将一个客户端事务请求，转换成一个 **事务Proposal**，并将该 Proposal 分发给集群中所有的 Follower 服务器，也就是向所有 Follower 节点发送数据广播请求（或数据复制）
- 分发之后Leader服务器需要等待所有Follower服务器的反馈（Ack请求），**在Zab协议中，只要超过半数的Follower服务器进行了正确的反馈**后（也就是收到半数以上的Follower的Ack请求），那么 Leader 就会再次向所有的 Follower服务器发送 Commit 消息，要求其将上一个 事务proposal 进行提交。

### Zab协议内容
Zab 协议包括两种基本的模式：**崩溃恢复** 和 **消息广播**
协议过程

当整个集群启动过程中，或者当 Leader 服务器出现网络中弄断、崩溃退出或重启等异常时，Zab协议就会 **进入崩溃恢复模式**，选举产生新的Leader。

当选举产生了新的 Leader，同时集群中有过半的机器与该 Leader 服务器完成了状态同步（即数据同步）之后，Zab协议就会退出崩溃恢复模式，**进入消息广播模式**。

这时，如果有一台遵守Zab协议的服务器加入集群，因为此时集群中已经存在一个Leader服务器在广播消息，那么该新加入的服务器自动进入恢复模式：找到Leader服务器，并且完成数据同步。同步完成后，作为新的Follower一起参与到消息广播流程中。

协议状态切换

当Leader出现崩溃退出或者机器重启，亦或是集群中不存在超过半数的服务器与Leader保存正常通信，Zab就会再一次进入崩溃恢复，发起新一轮Leader选举并实现数据同步。同步完成后又会进入消息广播模式，接收事务请求。

保证消息有序

在整个消息广播中，Leader会将每一个事务请求转换成对应的 proposal 来进行广播，并且在广播 事务Proposal 之前，Leader服务器会首先为这个事务Proposal分配一个全局单递增的唯一ID，称之为事务ID（即zxid），由于Zab协议需要保证每一个消息的严格的顺序关系，因此必须将每一个proposal按照其zxid的先后顺序进行排序和处理。

#### 消息广播

1）在zookeeper集群中，数据副本的传递策略就是采用消息广播模式。zookeeper中农数据副本的同步方式与二段提交相似，但是却又不同。二段提交要求协调者必须等到所有的参与者全部反馈ACK确认消息后，再发送commit消息。要求所有的参与者要么全部成功，要么全部失败。二段提交会产生严重的阻塞问题。

2）Zab协议中 Leader 等待 Follower 的ACK反馈消息是指“只要半数以上的Follower成功反馈即可，不需要收到全部Follower反馈”

消息广播具体步骤

1）客户端发起一个写操作请求。针对客户端的事务请求，leader服务器会先将该事物写到本地的log文件中

2）Leader 服务器将客户端的请求转化为事务 Proposal 提案，同时为每个 Proposal 分配一个全局的ID，即zxid。

3）Leader 服务器为每个 Follower 服务器分配一个单独的队列，然后将需要广播的 Proposal 依次放到队列中取，并且根据 FIFO 策略进行消息发送。

4）Follower 接收到 Proposal 后，会首先将其以事务日志的方式写入本地磁盘中，写入成功后向 Leader 反馈一个 Ack 响应消息。

- 如果写成功了，则给leader返回一个ACK消息
- 如果写失败了，follower认为该事务不能执行，就会返回no

5）Leader 接收到超过半数以上 Follower 的 Ack 响应消息后，即认为消息发送成功，可以发送 commit 消息。

6）Leader 向所有 Follower 广播 commit 消息，同时自身也会完成事务提交。Follower 接收到 commit 消息后，会将上一条事务提交。

**zookeeper 采用 Zab 协议的核心，就是只要有一台服务器提交了 Proposal，就要确保所有的服务器最终都能正确提交 Proposal。这也是 CAP/BASE 实现最终一致性的一个体现。**

**Leader 服务器与每一个 Follower 服务器之间都维护了一个单独的 FIFO 消息队列进行收发消息，使用队列消息可以做到异步解耦。 Leader 和 Follower 之间只需要往队列中发消息即可。如果使用同步的方式会引起阻塞，性能要下降很多。**

> 如果follower记录失败，但是leader去要求执行这个请求，follower会向leader发送一个请求，请求重新获取刚才的信息重新记录重新执行

为什么leader会收到follower返回的no？（为什么follower记录失败？）

- 网络问题。例如产生数据丢失导致follower没有请求
- follower在记录日志的时候发现日志被占用
- 磁盘问题。例如磁盘已满、磁盘损坏

#### 崩溃恢复

**一旦 Leader 服务器出现崩溃或者由于网络原因导致 Leader 服务器失去了与过半 Follower 的联系，那么就会进入崩溃恢复模式。**

1. 当leader服务器出现崩溃、重启等场景，或者因为网络问题导致过半的follower不能与leader服务器保持正常通信的时候，Zookeeper集群就会进入崩溃恢复模式
2. 进入崩溃恢复模式后，只要集群中存在过半的服务器能够彼此正常通信，那么就可以选举产生一个新的leader
3. 每次新选举的leader会自动分配一个全局递增的编号，即epochid
4. 当选举产生了新的leader服务器，同时集群中已经有过半的机器与该leader服务器完成了状态同步之后，ZAB协议就会退出恢复模式。其中，所谓的状态同步是指数据同步，用来保证集群中存在过半的机器能够和leader服务器的数据状态保持一致
5. 当集群中已经有过半的follower服务器完成了和leader服务器的状态同步，那么整个服务框架就可以进入消息广播模式了
6. 当一台同样遵守ZAB协议的服务器启动后加入到集群中时，如果此时集群中已经存在一个Leader服务器在负责进行消息广播，那么新加入的服务器就会自觉地进入数据恢复模式：
    - 找到leader所在的服务器，并与其进行数据同步
    - 然后一起参与到消息广播流程中

> **作用**：避免单点故障
>
> 事务id由64位二进制数字（16位十六进制）组成 ，其中高32位对应了epochid，低32位对应了实际的事务id

在 Zab 协议中，为了保证程序的正确运行，整个恢复过程结束后需要选举出一个新的 Leader 服务器。因此 Zab 协议需要一个高效且可靠的 Leader 选举算法，从而确保能够快速选举出新的 Leader 。

Leader 选举算法不仅仅需要让 Leader 自己知道自己已经被选举为 Leader ，同时还需要让集群中的所有其他机器也能够快速感知到选举产生的新 Leader 服务器。

崩溃恢复主要包括两部分：**Leader选举** 和 **数据恢复**

Zab 协议如何保证数据一致性

假设两种异常情况：
1、一个事务在 Leader 上提交了，并且过半的 Folower 都响应 Ack 了，但是 Leader 在 Commit 消息发出之前挂了。
2、假设一个事务在 Leader 提出之后，Leader 挂了。

要确保如果发生上述两种情况，数据还能保持一致性，那么 Zab 协议选举算法必须满足以下要求：

**Zab 协议崩溃恢复要求满足以下两个要求**：
1）**确保已经被 Leader 提交的 Proposal 必须最终被所有的 Follower 服务器提交**。
2）**确保丢弃已经被 Leader 提出的但是没有被提交的 Proposal**。

根据上述要求
Zab协议需要保证选举出来的Leader需要满足以下条件：
1）**新选举出来的 Leader 不能包含未提交的 Proposal** 。
即新选举的 Leader 必须都是已经提交了 Proposal 的 Follower 服务器节点。
2）**新选举的 Leader 节点中含有最大的 zxid** 。
这样做的好处是可以避免 Leader 服务器检查 Proposal 的提交和丢弃工作。


Zab 如何数据同步

1）完成 Leader 选举后（新的 Leader 具有最高的zxid），在正式开始工作之前（接收事务请求，然后提出新的 Proposal），Leader 服务器会首先确认事务日志中的所有的 Proposal 是否已经被集群中过半的服务器 Commit。

2）Leader 服务器需要确保所有的 Follower 服务器能够接收到每一条事务的 Proposal ，并且能将所有已经提交的事务 Proposal 应用到内存数据中。等到 Follower 将所有尚未同步的事务 Proposal 都从 Leader 服务器上同步过啦并且应用到内存数据中以后，Leader 才会把该 Follower 加入到真正可用的 Follower 列表中。


Zab 数据同步过程中，如何处理需要丢弃的 Proposal

在 Zab 的事务编号 zxid 设计中，zxid是一个64位的数字。

其中低32位可以看成一个简单的单增计数器，针对客户端每一个事务请求，Leader 在产生新的 Proposal 事务时，都会对该计数器加1。而高32位则代表了 Leader 周期的 epoch 编号。

> epoch 编号可以理解为当前集群所处的年代，或者周期。每次Leader变更之后都会在 epoch 的基础上加1，这样旧的 Leader 崩溃恢复之后，其他Follower 也不会听它的了，因为 Follower 只服从epoch最高的 Leader 命令。

每当选举产生一个新的 Leader ，就会从这个 Leader 服务器上取出本地事务日志充最大编号 Proposal 的 zxid，并从 zxid 中解析得到对应的 epoch 编号，然后再对其加1，之后该编号就作为新的 epoch 值，并将低32位数字归零，由0开始重新生成zxid。

**Zab 协议通过 epoch 编号来区分 Leader 变化周期**，能够有效避免不同的 Leader 错误的使用了相同的 zxid 编号提出了不一样的 Proposal 的异常情况。

基于以上策略
**当一个包含了上一个 Leader 周期中尚未提交过的事务 Proposal 的服务器启动时，当这台机器加入集群中，以 Follower 角色连上 Leader 服务器后，Leader 服务器会根据自己服务器上最后提交的 Proposal 来和 Follower 服务器的 Proposal 进行比对，比对的结果肯定是 Leader 要求 Follower 进行一个回退操作，回退到一个确实已经被集群中过半机器 Commit 的最新 Proposal**。

#### 什么情况下zab协议会进入崩溃恢复模式？

- 1、当服务器启动时
- 2、当leader 服务器出现网络中断，崩溃或者重启的情况
- 3、当集群中已经不存在过半的服务器与Leader服务器保持正常通信。

#### zab协议进入崩溃恢复模式会做什么？

1、当leader出现问题，zab协议进入崩溃恢复模式，并且选举出新的leader。当新的leader选举出来以后，如果集群中已经有过半机器完成了leader服务器的状态同（数据同步），退出崩溃恢复，进入消息广播模式。

2、当新的机器加入到集群中的时候，如果已经存在leader服务器，那么新加入的服务器就会自觉进入崩溃恢复模式，找到leader进行数据同步。

#### 特殊情况下需要解决的两个问题：

##### 问题一：已经被处理的事务请求（proposal）不能丢（commit的）

> 当 leader 收到合法数量 follower 的 ACKs 后，就向各个 follower 广播 COMMIT 命令，同时也会在本地执行 COMMIT 并向连接的客户端返回「成功」。但是如果在各个 follower 在收到 COMMIT 命令前 leader 就挂了，导致剩下的服务器并没有执行都这条消息。

- 如何解决 *已经被处理的事务请求（proposal）不能丢（commit的）* 呢？

1、选举拥有 proposal 最大值（即 zxid 最大） 的节点作为新的 leader。

> 由于所有提案被 COMMIT 之前必须有合法数量的 follower ACK，即必须有合法数量的服务器的事务日志上有该提案的 proposal，因此，zxid最大也就是数据最新的节点保存了所有被 COMMIT 消息的 proposal 状态。

2、新的 leader 将自己事务日志中 proposal 但未 COMMIT 的消息处理。

3、新的 leader 与 follower 建立先进先出的队列， 先将自身有而 follower 没有的 proposal 发送给 follower，再将这些 proposal 的 COMMIT 命令发送给 follower，以保证所有的 follower 都保存了所有的 proposal、所有的 follower 都处理了所有的消息。通过以上策略，能保证已经被处理的消息不会丢。

##### 问题二：没被处理的事务请求（proposal）不能再次出现什么时候会出现事务请求被丢失呢？

> 当 leader 接收到消息请求生成 proposal 后就挂了，其他 follower 并没有收到此 proposal，因此经过恢复模式重新选了 leader 后，这条消息是被跳过的。 此时，之前挂了的 leader 重新启动并注册成了 follower，他保留了被跳过消息的 proposal 状态，与整个系统的状态是不一致的，需要将其删除。

- 如果解决呢？

Zab 通过巧妙的设计 zxid 来实现这一目的。

一个 zxid 是64位，高 32 是纪元（epoch）编号，每经过一次 leader 选举产生一个新的 leader，新 leader 会将 epoch 号 +1。低 32 位是消息计数器，每接收到一条消息这个值 +1，新 leader 选举后这个值重置为 0。

这样设计的好处是旧的 leader 挂了后重启，它不会被选举为 leader，因为此时它的 zxid 肯定小于当前的新 leader。当旧的 leader 作为 follower 接入新的 leader 后，新的 leader 会让它将所有的拥有旧的 epoch 号的未被 COMMIT 的 proposal 清除。

## 选举 Leader

**ZooKeeper 的故障恢复**

ZooKeeper 集群采用一主（称为 Leader）多从（称为 Follower）模式，主从节点通过副本机制保证数据一致。

- 如果 Follower 节点挂了 - ZooKeeper 集群中的每个节点都会单独在内存中维护自身的状态，并且各节点之间都保持着通讯，只要集群中有半数机器能够正常工作，那么整个集群就可以正常提供服务。
- 如果 Leader 节点挂了 - 如果 Leader 节点挂了，系统就不能正常工作了。此时，需要通过 ZAB 协议的选举 Leader 机制来进行故障恢复。

ZAB 协议的选举 Leader 机制简单来说，就是：基于过半选举机制产生新的 Leader，之后其他机器将从新的 Leader 上同步状态，当有过半机器完成状态同步后，就退出选举 Leader 模式，进入原子广播模式。

### 术语

**myid：**每个 Zookeeper 服务器，都需要在数据文件夹下创建一个名为 myid 的文件，该文件包含整个 Zookeeper 集群唯一的 ID（整数）。

**zxid：**类似于 RDBMS 中的事务 ID，用于标识一次更新操作的 Proposal ID。为了保证顺序性，该 zkid 必须单调递增。因此 Zookeeper 使用一个 64 位的数来表示，高 32 位是 Leader 的 epoch，从 1 开始，每次选出新的 Leader，epoch 加一。低 32 位为该 epoch 内的序号，每次 epoch 变化，都将低 32 位的序号重置。这样保证了 zkid 的全局递增性。

### 服务器状态

- **LOOKING：**不确定 Leader 状态。该状态下的服务器认为当前集群中没有 Leader，会发起 Leader 选举。
- **FOLLOWING：**跟随者状态。表明当前服务器角色是 Follower，并且它知道 Leader 是谁。
- **LEADING：**领导者状态。表明当前服务器角色是 Leader，它会维护与 Follower 间的心跳。
- **OBSERVING：**观察者状态。表明当前服务器角色是 Observer，与 Folower 唯一的不同在于不参与选举，也不参与集群写操作时的投票。

### 选票数据结构

每个服务器在进行领导选举时，会发送如下关键信息：

- **logicClock：**每个服务器会维护一个自增的整数，名为 logicClock，它表示这是该服务器发起的第多少轮投票。
- **state：**当前服务器的状态。
- **self_id：**当前服务器的 myid。
- **self_zxid：**当前服务器上所保存的数据的最大 zxid。
- **vote_id：**被推举的服务器的 myid。
- **vote_zxid：**被推举的服务器上所保存的数据的最大 zxid。

### 投票流程
**（1）自增选举轮次**

Zookeeper 规定所有有效的投票都必须在同一轮次中。每个服务器在开始新一轮投票时，会先对自己维护的 logicClock 进行自增操作。

**（2）初始化选票**

每个服务器在广播自己的选票前，会将自己的投票箱清空。该投票箱记录了所收到的选票。例：服务器 2 投票给服务器 3，服务器 3 投票给服务器 1，则服务器 1 的投票箱为(2, 3), (3, 1), (1, 1)。票箱中只会记录每一投票者的最后一票，如投票者更新自己的选票，则其它服务器收到该新选票后会在自己票箱中更新该服务器的选票。

**（3）发送初始化选票**

每个服务器最开始都是通过广播把票投给自己。

**（4）接收外部投票**

服务器会尝试从其它服务器获取投票，并记入自己的投票箱内。如果无法获取任何外部投票，则会确认自己是否与集群中其它服务器保持着有效连接。如果是，则再次发送自己的投票；如果否，则马上与之建立连接。

**（5）判断选举轮次**

收到外部投票后，首先会根据投票信息中所包含的 logicClock 来进行不同处理：

- 外部投票的 logicClock **大于**自己的 logicClock。说明该服务器的选举轮次落后于其它服务器的选举轮次，立即清空自己的投票箱并将自己的 logicClock 更新为收到的 logicClock，然后再对比自己之前的投票与收到的投票以确定是否需要变更自己的投票，最终再次将自己的投票广播出去。
- 外部投票的 logicClock **小于**自己的 logicClock。当前服务器直接忽略该投票，继续处理下一个投票。
- 外部投票的 logickClock 与自己的**相等**。当时进行选票 PK。

**（6）选票 PK**

选票 PK 是基于(self_id, self_zxid)与(vote_id, vote_zxid)的对比：

- 外部投票的 logicClock **大于**自己的 logicClock，则将自己的 logicClock 及自己的选票的 logicClock 变更为收到的 logicClock。
- 若 **logicClock** **一致**，则对比二者的 vote_zxid，若外部投票的 vote_zxid 比较大，则将自己的票中的 vote_zxid 与 vote_myid 更新为收到的票中的 vote_zxid 与 vote_myid 并广播出去，另外将收到的票及自己更新后的票放入自己的票箱。如果票箱内已存在(self_myid, self_zxid)相同的选票，则直接覆盖。
- 若二者 **vote_zxid** 一致，则比较二者的 vote_myid，若外部投票的 vote_myid 比较大，则将自己的票中的 vote_myid 更新为收到的票中的 vote_myid 并广播出去，另外将收到的票及自己更新后的票放入自己的票箱。

**（7）统计选票**

如果已经确定有过半服务器认可了自己的投票（可能是更新后的投票），则终止投票。否则继续接收其它服务器的投票。

**（8）更新服务器状态**

投票终止后，服务器开始更新自身状态。若过半的票投给了自己，则将自己的服务器状态更新为 LEADING，否则将自己的状态更新为 FOLLOWING。

通过以上流程分析，我们不难看出：要使 Leader 获得多数 Server 的支持，则 **ZooKeeper 集群节点数必须是奇数。且存活的节点数目不得少于 N + 1** 。

每个 Server 启动后都会重复以上流程。在恢复模式下，如果是刚从崩溃状态恢复的或者刚启动的 server 还会从磁盘快照中恢复数据和会话信息，zk 会记录事务日志并定期进行快照，方便在恢复时进行状态恢复。

## 原子广播（Atomic Broadcast）

**ZooKeeper 通过副本机制来实现高可用。**

那么，ZooKeeper 是如何实现副本机制的呢？答案是：ZAB 协议的原子广播。

ZAB 协议的原子广播要求：

**所有的写请求都会被转发给 Leader，Leader 会以原子广播的方式通知 Follow。当半数以上的 Follow 已经更新状态持久化后，Leader 才会提交这个更新，然后客户端才会收到一个更新成功的响应。**这有些类似数据库中的两阶段提交协议。

在整个消息的广播过程中，Leader 服务器会每个事物请求生成对应的 Proposal，并为其分配一个全局唯一的递增的事务 ID(ZXID)，之后再对其进行广播。

## 选举过程中的问题

### 脑裂问题

**脑裂问题**出现在集群中leader死掉，follower选出了新leader而原leader又复活了的情况下，因为ZK的过半机制是允许损失一定数量的机器而扔能正常提供给服务，当leader死亡判断不一致时就会出现多个leader。

方案：

ZK的过半机制一定程度上也减少了脑裂情况的出现，起码不会出现三个leader同时。ZK中的Epoch机制（时钟）每次选举都是递增+1，当通信时需要判断epoch是否一致，小于自己的则抛弃，大于自己则重置自己，等于则选举