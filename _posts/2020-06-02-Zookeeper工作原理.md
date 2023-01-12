---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---

## ZooKeeper 工作原理

### 读操作

**Leader/Follower/Observer 都可直接处理读请求，从本地内存中读取数据并返回给客户端即可。**

由于处理读请求不需要服务器之间的交互，**Follower/Observer 越多，整体系统的读请求吞吐量越大**，也即读性能越好。

![Zookeeper读操作](png\zookeeper\Zookeeper读操作.png)

![img](https://oscimg.oschina.net/oscnet/up-c61929f0f90b482800ca0959e55b3da3f85.png)

### 写操作

所有的写请求实际上都要交给 Leader 处理。Leader 将写请求以事务形式发给所有 Follower 并等待 ACK，一旦收到半数以上 Follower 的 ACK，即认为写操作成功。

#### **写 Leader**

![Zookeeper写Lead](png\zookeeper\Zookeeper写Lead.png) ![img](https://oscimg.oschina.net/oscnet/up-166f5dc0cb39337f7e7dabf9de020af0f43.png)

由上图可见，通过 Leader 进行写操作，主要分为五步：

1. 客户端向 Leader 发起写请求。
2. Leader 将写请求以事务 Proposal 的形式发给所有 Follower 并等待 ACK。
3. Follower 收到 Leader 的事务 Proposal 后返回 ACK。
4. Leader 得到过半数的 ACK（Leader 对自己默认有一个 ACK）后向所有的 Follower 和 Observer 发送 Commmit。
5. Leader 将处理结果返回给客户端。

**注意**

- Leader 不需要得到 Observer 的 ACK，即 Observer 无投票权。

- Leader 不需要得到所有 Follower 的 ACK，只要收到过半的 ACK 即可，同时 Leader 本身对自己有一个 ACK。上图中有 4 个 Follower，只需其中两个返回 ACK 即可，因为(2+1)/(4+1)>1/2(2+1)/(4+1)>1/2。

- Observer 虽然无投票权，但仍须同步 Leader 的数据从而在处理读请求时可以返回尽可能新的数据。

#### 写 Follower/Observer

![Zookeeper写Follower](png\zookeeper\Zookeeper写Follower.png)

Follower/Observer 均可接受写请求，但不能直接处理，而需要将写请求转发给 Leader 处理。

除了多了一步请求转发，其它流程与直接写 Leader 无任何区别。

## 事务

对于来自客户端的每个更新请求，ZooKeeper 具备严格的顺序访问控制能力。

**为了保证事务的顺序一致性，ZooKeeper 采用了递增的事务 id 号（zxid）来标识事务。**

**Leader 服务会为每一个 Follower 服务器分配一个单独的队列，然后将事务 Proposal 依次放入队列中，并根据 FIFO(先进先出) 的策略进行消息发送。**Follower 服务在接收到 Proposal 后，会将其以事务日志的形式写入本地磁盘中，并在写入成功后反馈给 Leader 一个 Ack 响应。**当 Leader 接收到超过半数 Follower 的 Ack 响应后，就会广播一个 Commit 消息给所有的 Follower 以通知其进行事务提交，**之后 Leader 自身也会完成对事务的提交。而每一个 Follower 则在接收到 Commit 消息后，完成事务的提交。

所有的提议（proposal）都在被提出的时候加上了 zxid。zxid 是一个 64 位的数字，它的高 32 位是 epoch 用来标识 Leader 关系是否改变，每次一个 Leader 被选出来，它都会有一个新的 epoch，标识当前属于那个 leader 的统治时期。低 32 位用于递增计数。

详细过程如下：

- Leader 等待 Server 连接；
- Follower 连接 Leader，将最大的 zxid 发送给 Leader；
- Leader 根据 Follower 的 zxid 确定同步点；
- 完成同步后通知 follower 已经成为 uptodate 状态；
- Follower 收到 uptodate 消息后，又可以重新接受 client 的请求进行服务了。

## 观察

**客户端注册监听它关心的 znode，当 znode 状态发生变化（数据变化、子节点增减变化）时，ZooKeeper 服务会通知客户端。**

客户端和服务端保持连接一般有两种形式：

- 客户端向服务端不断轮询
- 服务端向客户端推送状态

Zookeeper 的选择是服务端主动推送状态，也就是观察机制（ Watch ）。

ZooKeeper 的观察机制允许用户在指定节点上针对感兴趣的事件注册监听，当事件发生时，监听器会被触发，并将事件信息推送到客户端。

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

**ZooKeeper 客户端通过 TCP 长连接连接到 ZooKeeper 服务集群。会话 (Session) 从第一次连接开始就已经建立，之后通过心跳检测机制来保持有效的会话状态**。通过这个连接，客户端可以发送请求并接收响应，同时也可以接收到 Watch 事件的通知。

每个 ZooKeeper 客户端配置中都配置了 ZooKeeper 服务器集群列表。启动时，客户端会遍历列表去尝试建立连接。如果失败，它会尝试连接下一个服务器，依次类推。

一旦一台客户端与一台服务器建立连接，这台服务器会为这个客户端创建一个新的会话。**每个会话都会有一个超时时间，若服务器在超时时间内没有收到任何请求，则相应会话被视为过期。**一旦会话过期，就无法再重新打开，且任何与该会话相关的临时 znode 都会被删除。

通常来说，会话应该长期存在，而这需要由客户端来保证。客户端可以通过心跳方式（ping）来保持会话不过期。

![Zookeeper会话](png\zookeeper\Zookeeper会话.png)

![img](https://oscimg.oschina.net/oscnet/up-15bf9f99ba1b79dd16f89ccb76627f67f2f.png)

ZooKeeper 的会话具有四个属性：

- **sessionID：**会话 ID，唯一标识一个会话，每次客户端创建新的会话时，Zookeeper 都会为其分配一个全局唯一的 sessionID。
- **TimeOut：**会话超时时间，客户端在构造 Zookeeper 实例时，会配置 sessionTimeout 参数用于指定会话的超时时间，Zookeeper 客户端向服务端发送这个超时时间后，服务端会根据自己的超时时间限制最终确定会话的超时时间。
- **TickTime：**下次会话超时时间点，为了便于 Zookeeper 对会话实行”分桶策略”管理，同时为了高效低耗地实现会话的超时检查与清理，Zookeeper 会为每个会话标记一个下次会话超时时间点，其值大致等于当前时间加上 TimeOut。
- **isClosing：**标记一个会话是否已经被关闭，当服务端检测到会话已经超时失效时，会将该会话的 isClosing 标记为”已关闭”，这样就能确保不再处理来自该会话的新请求了。

Zookeeper 的会话管理主要是通过 SessionTracker 来负责，其采用了**分桶策略**（将类似的会话放在同一区块中进行管理）进行管理，以便 Zookeeper 对会话进行不同区块的隔离处理以及同一区块的统一处理。