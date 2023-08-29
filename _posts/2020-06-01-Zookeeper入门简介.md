---
layout: post
categories: [Zookeeper]
description: none
keywords: Zookeeper
---
# Zookeeper入门简介
ZooKeeper是一个为分布式应用所设计的开源协调服务。主要用来解决分布式系统中数据一致性的问题，它可以为用户提供同步、配置管理、分组和命名等服务。

## Zookeeper简介
ApacheZooKeeper是 Apache 软件基金会的一个软件项目，它为大型分布式计算提供开源的分布式配置服务、同步服务和命名注册。ZooKeeper 曾经是 Hadoop 的一个子项目，但现在是一个独立的顶级项目。
官网介绍:
```
Apache ZooKeeper is an effort to develop and maintain an open-source server which enables highly reliable distributed coordination.
```
官网介绍：**Apache ZooKeeper 致力于开发和维护一个开源服务器，以高可靠的分布式协调服务。**

Zookeeper是根据谷歌的论文《The Chubby Lock Service for Loosely Couple Distribute System 》所做的开源实现。 ZooKeeper 是 Apache 的顶级项目。ZooKeeper 为分布式应用提供了高效且可靠的分布式协调服务，提供了诸如统一命名服务、配置管理和分布式锁等分布式的基础服务。在解决分布式数据一致性方面，ZooKeeper 并没有直接采用 Paxos 算法，而是采用了名为 ZAB 的一致性协议。
ZooKeeper 主要用来解决分布式集群中应用系统的一致性问题，它能提供基于类似于文件系统的目录节点树方式的数据存储。但是 ZooKeeper 并不是用来专门存储数据的，它的作用主要是用来维护和监控存储数据的状态变化。通过监控这些数据状态的变化，从而可以达到基于数据的集群管理。
ZooKeeper 本质上是一个分布式的小文件存储系统。提供基于类似于文件系统的目录树方式的数据存储,并且可以对树中的节点进行有效管理。zk里存储了客户端感兴趣的数据，一旦这些数据发生变化就会通知到客户端做处理，zk的本质就是文件系统+通知机制。
ZooKeeper 提供给客户端监控存储在zk内部数据的功能，从而可以达到基于数据的集群管理。诸如:统一命名服务(dubbo)分布式配置管理(olr的配置集中管理)、分布式消息队列(sub/pub) 、分布式锁、分布式协调等功能。

## 由来
Zookeeper最早起源于雅虎研究院的一个研究小组。在当时，研究人员发现，在雅虎内部很多大型系统基本都需要依赖一个类似的系统来进行分布式协调，但是这些系统往往都存在分布式单点问题。所以，雅虎的开发人员就试图开发一个通用的无单点问题的分布式协调框架，以便让开发人员将精力集中在处理业务逻辑上。
关于“ZooKeeper”这个项目的名字，其实也有一段趣闻。在立项初期，考虑到之前内部很多项目都是使用动物的名字来命名的（例如著名的Pig项目),雅虎的工程师希望给这个项目也取一个动物的名字。时任研究院的首席科学家RaghuRamakrishnan开玩笑地说：“在这样下去，我们这儿就变成动物园了！”此话一出，大家纷纷表示就叫动物园管理员吧一一一因为各个以动物命名的分布式组件放在一起，雅虎的整个分布式系统看上去就像一个大型的动物园了，而Zookeeper正好要用来进行分布式环境的协调一一于是，Zookeeper的名字也就由此诞生了。
顾名思义 zookeeper 就是动物园管理员，他是用来管 hadoop（大象）、Hive(蜜蜂)、pig(小猪)的管理员， Apache Hbase 和 Apache Solr 的分布式集群都用到了 zookeeper；Zookeeper:是一个分布式的、开源的程序协调服务，是 hadoop 项目下的一个子项目。他提供的主要功能包括：配置管理、名字服务、分布式锁、集群管理。

## ZooKeeper的设计目标
众所周知，分布式环境下的程序和活动为了达到协调一致的目的，通常具有某些共同的特点，例如，简单性、有序性等。ZooKeeper不但在这些目标的实现上有自身的特点，并且具有其独特的优势。
下面我们将简述ZooKeeper的设计目标。
- 简单化
ZooKeeper允许分布式的进程通过共享体系的命名空间来进行协调，这个命名空间的组织与标准的文件系统非常相似，它是由一些数据寄存器组成的。用ZooKeeper的语法来说，这些寄存器应称为Znode，它们和文件及目录非常相似。典型的文件系统是基于存储设备的，然而，ZooKeeper的数据却是存放在内存当中的，这就意味着ZooKeeper可以达到一个高的吞吐量，并且低延迟。ZooKeeper的实现非常重视高性能、高可靠性，以及严格的有序访问。
ZooKeeper性能上的特点决定了它能够用在大型的、分布式的系统当中。从可靠性方面来说，它并不会因为一个节点的错误而崩溃。除此之外，它严格的序列访问控制意味着复杂的控制原语可以应用在客户端上。
- 健壮性
组成ZooKeeper服务的服务器必须互相知道其他服务器的存在。它们维护着一个处于内存中的状态镜像，以及一个位于存储器中的交换日志和快照。只要大部分的服务器可用，那么ZooKeeper服务就可用。
如果客户端连接到单个ZooKeeper服务器上，那么这个客户端就管理着一个TCP连接，并且通过这个TCP连接来发送请求、获得响应、获取检测事件，以及发送心跳。如果连接到服务器上的TCP连接断开，客户端将连接到其他的服务器上。
- 有序性
ZooKeeper可以为每一次更新操作赋予一个版本号，并且此版本号是全局有序的，不存在重复的情况。ZooKeeper所提供的很多服务也是基于此有序性的特点来完成。
- 速度优势
它在读取主要负载时尤其快。ZooKeeper应用程序在上千台机器的节点上运行。另外，需要注意的是ZooKeeper有这样一个特点：当读工作比写工作更多的时候，它执行的性能会更好。
除此之外，ZooKeeper还具有原子性、单系统镜像、可靠性的及时效性等特点。

## Zookeeper作用
- 配置管理
在我们的应用中除了代码外，还有一些就是各种配置。比如数据库连接等。一般我们都是使用配置文件的方式，在代码中引入这些配置文件。当我们只有一种配置，只有一台服务器，并且不经常修改的时候，使用配置文件是一个很好的做法，但是如果我们配置非常多，有很多服务器都需要这个配置，这时使用配置文件就不是个好主意了。这个时候往往需要寻找一种集中管理配置的方法，我们在这个集中的地方修改了配置，所有对这个配置感兴趣的
都可以获得变更。Zookeeper 就是这种服务，它使用 Zab 这种一致性协议来提供一致性。现在有很多开源项目使用 Zookeeper 来维护配置，比如在 HBase 中，客户端就是连接一个Zookeeper，获得必要的 HBase 集群的配置信息，然后才可以进一步操作。还有在开源的消息队列 Kafka 中，也使用 Zookeeper 来维护 broker 的信息。在 Alibaba 开源的 SOA 框架 Dubbo中也广泛的使用 Zookeeper 管理一些配置来实现服务治理。
- 名字服务
名字服务这个就很好理解了。比如为了通过网络访问一个系统，我们得知道对方的 IP地址，但是 IP 地址对人非常不友好，这个时候我们就需要使用域名来访问。但是计算机是不能是域名的。怎么办呢？如果我们每台机器里都备有一份域名到 IP 地址的映射，这个倒是能解决一部分问题，但是如果域名对应的 IP 发生变化了又该怎么办呢？于是我们有了DNS 这个东西。我们只需要访问一个大家熟知的(known)的点，它就会告诉你这个域名对应的 IP 是什么。在我们的应用中也会存在很多这类问题，特别是在我们的服务特别多的时候，如果我们在本地保存服务的地址的时候将非常不方便，但是如果我们只需要访问一个大家都熟知的访问点，这里提供统一的入口，那么维护起来将方便得多了。
- 分布式锁
Zookeeper 是一个分布式协调服务。这样我们就可以利用 Zookeeper 来协调多个分布式进程之间的活动。比如在一个分布式环境中，为了提高可靠性，我们的集群的每台服务器上都部署着同样的服务。但是，一件事情如果集群中的每个服务器都进行的话，那相互之间就要协调，编程起来将非常复杂。而如果我们只让一个服务进行操作，那又存在单点。通常还有一种做法就是使用分布式锁，在某个时刻只让一个服务去干活，当这台服务出问题的时候锁释放，立即 fail over 到另外的服务。这在很多分布式系统中都是这么做，这种设计有一个更好听的名字叫 Leader Election(leader 选举)。比如 HBase的 Master 就是采用这种机制。但要注意的是分布式锁跟同一个进程的锁还是有区别的，所以使用的时候要比同一个进程里的锁更谨慎的使用。
- 集群管理
在分布式的集群中，经常会由于各种原因，比如硬件故障，软件故障，网络问题，有些节点会进进出出。有新的节点加入进来，也有老的节点退出集群。这个时候，集群中其他机器需要感知到这种变化，然后根据这种变化做出对应的决策。比如我们是一个分布式存储系统，有一个中央控制节点负责存储的分配，当有新的存储进来的时候我们要根据现在集群目前的状态来分配存储节点。这个时候我们就需要动态感知到集群目前的状态。还有，比如一
个分布式的 SOA 架构中，服务是一个集群提供的，当消费者访问某个服务时，就需要采用某种机制发现现在有哪些节点可以提供该服务(这也称之为服务发现，比如 Alibaba 开源的SOA 框架 Dubbo 就采用了 Zookeeper 作为服务发现的底层机制)。还有开源的 Kafka 队列就采用了 Zookeeper 作为 Cosnumer 的上下线管理。

## Zookeeper特性
ZooKeeper 具有以下特性：
- 顺序一致性
所有客户端看到的服务端数据模型都是一致的；从一个客户端发起的事务请求，最终都会严格按照其发起顺序被应用到 ZooKeeper 中。具体的实现可见下文：原子广播。
- 原子性
所有事务请求的处理结果在整个集群中所有机器上的应用情况是一致的，即整个集群要么都成功应用了某个事务，要么都没有应用。实现方式可见下文：事务。
- 单一视图
无论客户端连接的是哪个 Zookeeper 服务器，其看到的服务端数据模型都是一致的。
- 高性能
ZooKeeper 将数据全量存储在内存中，所以其性能很高。需要注意的是：由于 ZooKeeper 的所有更新和删除都是基于事务的，因此 ZooKeeper 在读多写少的应用场景中有性能表现较好，如果写操作频繁，性能会大大下滑。
- 高可用
ZooKeeper 的高可用是基于副本机制实现的，此外 ZooKeeper 支持故障恢复，可见下文：选举 Leader。

## ZooKeeper数据模型
ZooKeeper 的数据模型是一个树形结构的文件系统。树中的节点被称为 znode，其中根节点为 /，每个节点上都会保存自己的数据和节点信息。znode 可以用于存储数据，并且有一个与之相关联的 ACL。ZooKeeper 的设计目标是实现协调服务，而不是真的作为一个文件存储，因此 znode 存储数据的**大小被限制在 1MB 以内。**

ZooKeeper 的数据访问具有原子性。其读写操作都是要么全部成功，要么全部失败。它的名称是由通过斜线分隔的路径名序列所组成的。ZooKeeper中的每一个节点都是通过路径来识别的。
znode 通过路径被引用。znode 节点路径必须是绝对路径。

znode 有两种类型：
- 临时的(EPHEMERAL)
客户端会话结束时，ZooKeeper 就会删除临时的 znode。
- 持久的(PERSISTENT)
除非客户端主动执行删除操作，否则 ZooKeeper 不会删除持久的 znode。

在Zookeeper中存储的创建的结点和存储的数据包含结点的创建时间、修改时间、结点id、结点中存储数据的版本、权限版本、孩子结点的个数、数据的长度等信息。
`/zookeeper`是一个保留词，不能用作一个路径组件。`Zookeeper`使用`/zookeeper`来保存管理信息

## ZooKeeper中的节点和临时节点
ZooKeeper中存在着节点的概念，这些节点是通过像树一样的结构来进行维护的，并且每一个节点通过路径来标识及访问。除此之外，每一个节点还拥有自身的一些信息，包括：数据、数据长度、创建时间、修改时间等。从节点的这些特性（既含有数据，又通过路径来标识）可以看出，它既可以被看作是一个文件，又可以被看作是一个目录，因为它同时具有二者的特点。为了便于表达，后面我们将使用Znode来表示所讨论的ZooKeeper节点。

具体地说，Znode维护着数据、访问控制列表（access control list, ACL）、时间戳等包含交换版本号信息的数据结构，通过对这些数据的管理使缓存中的数据生效，并且执行协调更新操作。每当Znode中的数据更新它所维护的版本号就会增加，这非常类似于数据库中计数器时间戳的操作方式。

另外Znode还具有原子性操作的特点：在命名空间中，每一个Znode的数据将被原子地读写。读操作将读取与Znode相关的所有数据，写操作将替换掉所有的数据。除此之外，每一个节点都有一个访问控制列表，这个访问控制列表规定了用户操作的权限。

ZooKeeper中同样存在临时节点。这些节点与session同时存在，当session生命周期结束时，这些临时节点也将被删除。临时节点在某些场合也发挥着非常重要的作用，例如Leader选举、锁服务等。

## Znode
Znode是Zookeeper中数据的最小单元，每个Znode上都可以保存数据，同时还可以挂载子节点，znode之间的层级关系就像文件系统的目录结构一样，zookeeper将全部的数据存储在内存中以此来提高服务器吞吐量、减少延迟的目的。
Znode有三种类型，Znode的类型在创建时确定并且不能修改。

### **持久节点(PERSISTENT):**
- 这种节点也是在 ZooKeeper 最为常用的，几乎所有业务场景中都会包含持久节点的创建。
- 之所以叫作持久节点是因为一旦将节点创建为持久节点，该数据节点会一直存储在 ZooKeeper 服务器上，即使创建该节点的客户端与服务端的会话关闭了，该节点依然不会被删除。
- 如果想删除持久节点，就要显式调用 delete 函数进行删除操作。
- Zookeeper规定所有非叶子节点必须是持久化节点

### **临时节点(EPHEMERAL):**

- 从名称上可以看出该节点的一个最重要的特性就是临时性。
- 所谓临时性是指，如果将节点创建为临时节点，那么该节点数据不会一直存储在 ZooKeeper 服务器上。
- 当创建该临时节点的客户端会话因超时或发生异常而关闭时，该节点也相应在 ZooKeeper 服务器上被删除。
- 同样，也可以像删除持久节点一样主动删除临时节点。
- 在创建临时Znode的客户端会话结束时，服务器会将临时节点删除。临时节点不能有子节点(即使是临时子节点)。虽然每个临时Znode都会绑定一个特定的客户端会话，但是它们对所有客户端都是可见的

### **有序节点(SEQUENTIAL):**

- 其实有序节点并不算是一种单独种类的节点，而是在之前提到的持久节点和临时节点特性的基础上，增加了一个节点有序的性质。
- 所谓节点有序是说在创建有序节点的时候，ZooKeeper 服务器会自动使用一个单调递增的数字作为后缀，追加到创建节点的后边。
- 例如一个客户端创建了一个路径为 works/task- 的有序节点，那么 ZooKeeper 将会生成一个序号并追加到该节点的路径后，最后该节点的路径为 works/task-1。
- 通过这种方式可以直观的查看到节点的创建顺序。

每个数据节点除了存储数据内容之外，还存储了数据节点本身的一些状态信息。Znode的状态(Stat)信息

```
 [zk: 127.0.0.1:2181(CONNECTED) 1] get /jannal-create/user/password
    123456
    cZxid = 0x6000000e4
    ctime = Wed Oct 03 22:46:07 CST 2018
    mZxid = 0x6000000e4
    mtime = Wed Oct 03 22:46:07 CST 2018
    pZxid = 0x6000000e4
    cversion = 0
    dataVersion = 0
    aclVersion = 0
    ephemeralOwner = 0x0
    dataLength = 6
    numChildren = 0
```

每一个节点都有一个自己的状态属性，记录了节点本身的一些信息，这些属性包括的内容如下表所示：

| 状态属性           | 说明                                                            |
|----------------|---------------------------------------------------------------|
| cZxid          | Created ZXID表示该数据节点被创建时的事务ID                                  |
| ctime          | Created Time表示节点被创建的时间                                        |
| mZxid          | Modified ZXID 表示该节点最后一次被更新时的事务ID                              |
| mtime          | Modified Time表示节点最后一次被更新的时间                                   |
| pZxid          | 表示该节点的子节点列表最后一次被修改时的事务ID。只有子节点列表变更了才会变更pZxid,子节点内容变更不会影响pZxid |
| cversion       | 子节点的版本号这表示对此znode的子节点进行的更改次数                                  |
| dataVersion    | 数据节点版本号表示对该znode的数据所做的更改次数                                    |
| aclVersion     | 节点的ACL版本号表示对此znode的ACL进行更改的次数                                 |
| ephemeralOwner | 创建该临时节点的会话的SessionID。如果节点是持久节点，这个属性为0                         |
| dataLength     | 数据内容的长度                                                       |
| numChildren    | 当前节点的子节点个数                                                    |

Zookeeper中版本(version)表示的是对数据节点的数据内容、子节点列表或是节点ACL信息的修改次数，初始为0表示没有被修改，每当对数据内容进行更新，version就会加1。如果前后两次修改并没有使得数据内容发生变化，version的值依然会变化。修改时如果设置version=-1,表示客户端并不要求使用乐观锁，可以忽略版本对比。

```
    //第一次dataVersion是0
    [zk: 127.0.0.1:2181(CONNECTED) 1] get /jannal-create/user/password
    123456
    cZxid = 0x6000000e4
    ctime = Wed Oct 03 22:46:07 CST 2018
    mZxid = 0x6000000e4
    mtime = Wed Oct 03 22:46:07 CST 2018
    pZxid = 0x6000000e4
    cversion = 0
    dataVersion = 0
    aclVersion = 0
    ephemeralOwner = 0x0
    dataLength = 6
    numChildren = 0
    
    //将值修改为1234567后dataVersion变为1
    [zk: 127.0.0.1:2181(CONNECTED) 3] set /jannal-create/user/password 1234567
    cZxid = 0x6000000e4
    ctime = Wed Oct 03 22:46:07 CST 2018
    mZxid = 0x6000000e6
    mtime = Wed Oct 03 23:02:52 CST 2018
    pZxid = 0x6000000e4
    cversion = 0
    dataVersion = 1
    aclVersion = 0
    ephemeralOwner = 0x0
    dataLength = 7
    numChildren = 0
    
    //将值修改为原来的123456后dataVersion变为2
    [zk: 127.0.0.1:2181(CONNECTED) 4] set /jannal-create/user/password 123456
    cZxid = 0x6000000e4
    ctime = Wed Oct 03 22:46:07 CST 2018
    mZxid = 0x6000000e7
    mtime = Wed Oct 03 23:02:57 CST 2018
    pZxid = 0x6000000e4
    cversion = 0
    dataVersion = 2
    aclVersion = 0
    ephemeralOwner = 0x0
    dataLength = 6
    numChildren = 0

```

### ACL

ZooKeeper 采用 ACL（Access Control Lists）策略来进行权限控制。

每个 znode 创建时都会带有一个 ACL 列表，用于决定谁可以对它执行何种操作。

Zookeeper的ACL由<schema>:<id>:<acl>三段组成。

- schema：可以取下列值：world, auth, digest, host/ip
- id： 标识身份，值依赖于schema做解析。
- acl：就是权限：cdwra分别表示create, delete,write,read, admin

#### **scheme**:

scheme对应于采用哪种方案来进行权限管理，zookeeper实现了一个pluggable的ACL方案，可以通过扩展scheme，来扩展ACL的机制。zookeeper-3.4.4缺省支持下面几种scheme:

- - **world**: 它下面只有一个id, 叫anyone, world:anyone代表任何人，zookeeper中对所有人有权限的结点就是属于world:anyone的
- **auth**: 它不需要id, 只要是通过authentication的user都有权限（zookeeper支持通过kerberos来进行authencation, 也支持username/password形式的authentication)
- **digest**: 它对应的id为username:BASE64(SHA1(password))，它需要先通过username:password形式的authentication
- **ip**: 它对应的id为客户机的IP地址，设置的时候可以设置一个ip段，比如ip:192.168.1.0/16, 表示匹配前16个bit的IP段
- **super**: 在这种scheme情况下，对应的id拥有超级权限，可以做任何事情(cdrwa)

另外，zookeeper-3.4.4的代码中还提供了对sasl的支持，不过缺省是没有开启的，需要配置才能启用，具体怎么配置在下文中介绍。

- **sasl**: sasl的对应的id，是一个通过sasl authentication用户的id，zookeeper-3.4.4中的sasl authentication是通过kerberos来实现的，也就是说用户只有通过了kerberos认证，才能访问它有权限的node.

#### **id**:

id与scheme是紧密相关的，具体的情况在上面介绍scheme的过程都已介绍，这里不再赘述。

#### **permission**:

zookeeper目前支持下面一些权限：

- CREATE(c): 创建权限，可以在在当前node下创建child node
- DELETE(d): 删除权限，可以删除当前的node
- READ(r): 读权限，可以获取当前node的数据，可以list当前node所有的child nodes
- WRITE(w): 写权限，可以向当前node写数据
- ADMIN(a): 管理权限，可以设置当前node的permission

注意：zookeeper对权限的控制是znode级别的，不具有继承性，即子节点不继承父节点的权限。这种设计在使用上还是有缺陷的，因为很多场景下，我们还是会把相关资源组织一下，放在同一个路径下面，这样就会有对一个路径统一授权的需求。

### Session

Session 指的是 ZooKeeper 服务器与客户端会话。在 ZooKeeper 中，一个客户端连接是指客户端和服务器之间的一个 TCP 长连接。客户端启动的时候，首先会与服务器建立一个 TCP 连接，从第一次连接建立开始，客户端会话的生命周期也开始了。通过这个连接，客户端能够通过心跳检测与服务器保持有效的会话，也能够向Zookeeper服务器发送请求并接受响应，同时还能够通过该连接接收来自服务器的Watch事件通知。 Session的sessionTimeout值用来设置一个客户端会话的超时时间。当由于服务器压力太大、网络故障或是客户端主动断开连接等各种原因导致客户端连接断开时，只要在sessionTimeout规定的时间内能够重新连接上集群中任意一台服务器，那么之前创建的会话仍然有效。

在为客户端创建会话之前，服务端首先会为每个客户端都分配一个sessionID。由于 sessionID 是 Zookeeper 会话的一个重要标识，许多与会话相关的运行机制都是基于这个 sessionID 的，因此，无论是哪台服务器为客户端分配的 sessionID，都务必保证全局唯一。

### Watcher

#### Watcher 监听机制：

Watcher,就是事件监听器，Zookeeper 中非常重要的特性，Zookeeper允许用户注册一些watcher,并且在一些事件特定触发时，ZooKeeper会将通知发给客户端。我们基于 zookeeper 上创建的节点，可以对这些节点绑定监听事件，比如可以监听节点数据变更、节点删除、子节点状态变更等事件，通过这个事件机制，可以基于 zookeeper实现分布式锁、集群管理等功能。

#### watcher 特性：

当数据发生变化的时候， zookeeper 会产生一个 watcher 事件，并且会发送到客户端。但是客户端只会收到一次通知。如果后续这个节点再次发生变化，那么之前设置 watcher 的客户端不会再次收到消息。（watcher 是一次性的操作）。 可以通过循环监听去达到永久监听效果。

#### 如何注册事件机制：

ZooKeeper 的 Watcher 机制，总的来说可以分为三个过程：**客户端注册 Watcher、服务器处理 Watcher 和客户端回调 Watcher客户端。**注册 watcher 有 3 种方式，getData、exists、getChildren

## 集群角色
Zookeeper 集群是一个基于主从复制的高可用集群，每个服务器承担如下三种角色中的一种。
- Leader
它负责 发起并维护与各 Follwer 及 Observer 间的心跳。所有的写操作必须要通过 Leader 完成再由 Leader 将写操作广播给其它服务器。一个 Zookeeper 集群同一时间只会有一个实际工作的 Leader。
- Follower
它会响应 Leader 的心跳。Follower 可直接处理并返回客户端的读请求，同时会将写请求转发给 Leader 处理，并且负责在 Leader 处理写请求时对请求进行投票。一个 Zookeeper 集群可能同时存在多个 Follower。
- Observer
角色与 Follower 类似，但是无投票权。

当Leader服务器出现网络中断、崩溃退出与重启等异常情况时，ZAB协议就会进入恢复模式并选举产生新的Leader。
这个过程大致是这样的：
**Leader election(选举阶段)**：节点在一开始都处于选举阶段，只要有一个节点得到超过半数节点的票数，它就可以当选准 Leader。
**Discovery(发现阶段)：**在这个阶段，Followers跟准Leader进行通信，同步Followers最近接收的事务提议。
**Synchronization(同步阶段)：**同步阶段主要是利用Leader前一阶段获得的最新提议历史，同步集群中所有的副本。同步完成之后准Leader才会成为真正的Leader。
**Broadcast(广播阶段)：** 到了这个阶段，Zookeeper集群才能正式对外提供事务服务，并且Leader 可以进行消息广播。同时如果有新的节点加入，还需要对新节点进行同步。zookeeper有三种角色：老大Leader(领导者）   2、老二Follower （跟随者） 3、老三Observer（观察者）。其中，Follower和Observer归类为Learner（学习者）

按重要性排序是Leader > Follower > Observer

### Leader
Leader在集群中只有一个节点，可以说是老大No.1，是zookeeper集群的中心，负责协调集群中的其他节点。从性能的角度考虑，leader可以选择不接受客户端的连接。

主要作用有：

1、发起与提交写请求。

所有的跟随者Follower与观察者Observer节点的写请求都会转交给领导者Leader执行。Leader接受到一个写请求后，首先会发送给所有的Follower，统计Follower写入成功的数量。当有超过半数的Follower写入成功后，Leader就会认为这个写请求提交成功，通知所有的Follower commit这个写操作，保证事后哪怕是集群崩溃恢复或者重启，这个写操作也不会丢失。

2、与learner保持心跳

3、崩溃恢复时负责恢复数据以及同步数据到Learner

### Follower

Follow在集群中有多个，主要的作用有：

1、与老大Leader保持心跳连接

2、当Leader挂了的时候，经过投票后成为新的leader。leader的重新选举是由老二Follower们内部投票决定的。

3、向leader发送消息与请求

4、处理leader发来的消息与请求

###  Observer

可以说Observer是zookeeper集群中最边缘的存在。Observer的主要作用是提高zookeeper集群的读性能。通过leader的介绍我们知道zookeeper的一个写操作是要经过半数以上的Follower确认才能够写成功的。那么当zookeeper集群中的节点越多时，zookeeper的写性能就 越差。为了在提高zookeeper读性能（也就是支持更多的客户端连接）的同时又不影响zookeeper的写性能，zookeeper集群多了一个儿子Observer，只负责：

1、与leader同步数据

2、不参与leader选举，没有投票权。也不参与写操作的提议过程。

3、数据没有事务化到硬盘。即Observer只会把数据加载到内存。


## Zookeeper安装
ZooKeeper有不同的运行环境，包括：单机环境、集群环境和集群伪分布环境。这里，我们将分别介绍不同环境下如何安装ZooKeeper服务，并简单介绍它们的区别与联系。

## 单机版安装
- 解压zookeeper安装包（本人重命名为zookeeper，并移动到/usr/local路径下），此处只有解压命令
```
　tar -zxvf zookeeper-3.4.5.tar.gz
```
- 进入到zookeeper文件夹下，并创建data和logs文件夹（一般解压后都有data文件夹）
```
　　[root@localhost zookeeper]# cd /usr/local/zookeeper/

　　[root@localhost zookeeper]# mkdir logs
```
- 在conf目录下修改zoo.cfg文件（如果没有此文件，则自己新建该文件），修改为如下内容:
```
tickTime=2000
dataDir=/usr/myapp/zookeeper-3.4.5/data
dataLogDir=/usr/myapp/zookeeper-3.4.5/logs
clientPort=2181
```
- 进入bin目录，启动、停止、重启分和查看当前节点状态
```
　　[root@localhost bin]# ./zkServer.sh start
　　[root@localhost bin]# ./zkServer.sh stop
　　[root@localhost bin]# ./zkServer.sh restart
　　[root@localhost bin]# ./zkServer.sh status
```
### zookeeper有关配置信息

| 参数名                                   | 默认      | 描述                                                                                                                      |
|---------------------------------------|---------|-------------------------------------------------------------------------------------------------------------------------|
| clientPort                            |         | 服务的监听端口                                                                                                                 |
| dataDir                               |         | 用于存放内存数据快照的文件夹，同时用于集群的myid文件也存在这个文件夹里                                                                                   |
| tickTime                              | 2000    | Zookeeper的时间单元。Zookeeper中所有时间都是以这个时间单元的整数倍去配置的。例如，session的最小超时时间是2*tickTime。（单位：毫秒）                                     |
| dataLogDir                            |         | 事务日志写入该配置指定的目录，而不是“ dataDir ”所指定的目录。这将允许使用一个专用的日志设备并且帮助我们避免日志和快照之间的竞争                                                   |
| globalOutstandingLimit                | 1,000   | 最大请求堆积数。默认是1000。Zookeeper运行过程中，尽管Server没有空闲来处理更多的客户端请求了，但是还是允许客户端将请求提交到服务器上来，以提高吞吐性能。当然，为了防止Server内存溢出，这个请求堆积数还是需要限制下的。 |
| preAllocSize                          | 64M     | 预先开辟磁盘空间，用于后续写入事务日志。默认是64M，每个事务日志大小就是64M。如果ZK的快照频率较大的话，建议适当减小这个参数。                                                      |
| snapCount                             | 100,000 | 每进行snapCount次事务日志输出后，触发一次快照， 此时，Zookeeper会生成一个snapshot.*文件，同时创建一个新的事务日志文件log.*。默认是100,000.                              |
| traceFile                             |         | 用于记录所有请求的log，一般调试过程中可以使用，但是生产环境不建议使用，会严重影响性能。                                                                           |
| maxClientCnxns                        |         | 最大并发客户端数，用于防止DOS的，默认值是10，设置为0是不加限制                                                                                      |
| clientPortAddress / maxSessionTimeout |         | 对于多网卡的机器，可以为每个IP指定不同的监听端口。默认情况是所有IP都监听 clientPort 指定的端口                                                                 |
| minSessionTimeout                     |         | Session超时时间限制，如果客户端设置的超时时间不在这个范围，那么会被强制设置为最大或最小时间。默认的Session超时时间是在2 * tickTime ~ 20 * tickTime 这个范围                     |
| fsync.warningthresholdms              | 1000    | 事务日志输出时，如果调用fsync方法超过指定的超时时间，那么会在日志中输出警告信息。默认是1000ms。                                                                   |
| autopurge.snapRetainCount             |         | 参数指定了需要保留的事务日志和快照文件的数目。默认是保留3个。和autopurge.purgeInterval搭配使用                                                             |
| autopurge.purgeInterval               |         | 在3.4.0及之后版本，Zookeeper提供了自动清理事务日志和快照文件的功能，这个参数指定了清理频率，单位是小时，需要配置一个1或更大的整数，默认是0，表示不开启自动清理功能                               |
| syncEnabled                           |         | Observer写入日志和生成快照，这样可以减少Observer的恢复时间。默认为true。                                                                          |

### 集群安装
Zookeeper集群中只要有过半的节点是正常的情况下,那么整个集群对外就是可用的。正是基于这个特性,要将 ZK 集群的节点数量要为奇数（2n+1），如 3、5、7 个节点)较为合适。
- 解压zookeeper安装包（本人重命名为zookeeper，并移动到/usr/local路径下），此处只有解压命令
```
$ tar -zxvf zookeeper-3.4.6.tar.gz
```
- 在各个zookeeper节点目录创建data、logs目录

```
　　[root@localhost zookeeper]# cd /usr/local/zookeeper/

　　[root@localhost zookeeper]# mkdir logs
```
- 修改zoo.cfg配置文件

```
tickTime=2000
initLimit=10 
syncLimit=5 
dataDir=/usr/myapp/zookeeper-3.4.5/data
dataLogDir=/usr/myapp/zookeeper-3.4.5/logs
clientPort=2181
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2888:3888
server.3=127.0.0.1:2888:3888
```
①、tickTime：基本事件单元，这个时间是作为Zookeeper服务器之间或客户端与服务器之间维持心跳的时间间隔，每隔tickTime时间就会发送一个心跳；最小 的session过期时间为2倍tickTime
②、dataDir：存储内存中数据库快照的位置，除非另有说明，否则指向数据库更新的事务日志。注意：应该谨慎的选择日志存放的位置，使用专用的日志存储设备能够大大提高系统的性能，如果将日志存储在比较繁忙的存储设备上，那么将会很大程度上影像系统性能。
③、client：监听客户端连接的端口。
④、initLimit：允许follower连接并同步到Leader的初始化连接时间，以tickTime为单位。当初始化连接时间超过该值，则表示连接失败。
⑤、syncLimit：表示Leader与Follower之间发送消息时，请求和应答时间长度。如果follower在设置时间内不能与leader通信，那么此follower将会被丢弃。
⑥、server.A=B:C:D  A：其中 A 是一个数字，表示这个是服务器的编号；  B：是这个服务器的 ip 地址； C：Zookeeper服务器之间的通信端口； D：Leader选举的端口。
我们需要修改的第一个是 dataDir ,在指定的位置处创建好目录。
第二个需要新增的是 server.A=B:C:D 配置，其中 A 对应下面我们即将介绍的myid 文件。B是集群的各个IP地址，C:D 是端口配置。

### zookeeper有关配置信息集群选项

| 参数名                               | 默认  | 描述                                                                                                                                       |
|-----------------------------------|-----|------------------------------------------------------------------------------------------------------------------------------------------|
| electionAlg                       |     | 之前的版本中， 这个参数配置是允许我们选择leader选举算法，但是由于在以后的版本中，只有“FastLeaderElection ”算法可用，所以这个参数目前看来没有用了。                                                  |
| initLimit                         | 10  | Observer和Follower启动时，从Leader同步最新数据时，Leader允许initLimit * tickTime的时间内完成。如果同步的数据量很大，可以相应的把这个值设置的大一些。                                       |
| leaderServes                      | yes | 默 认情况下，Leader是会接受客户端连接，并提供正常的读写服务。但是，如果你想让Leader专注于集群中机器的协调，那么可以将这个参数设置为 no，这样一来，会大大提高写操作的性能。一般机器数比较多的情况下可以设置为no，让Leader不接受客户端的连接。默认为yes |
| server.x=[hostname]:nnnnn[:nnnnn] |     | “x”是一个数字，与每个服务器的myid文件中的id是一样的。hostname是服务器的hostname，右边配置两个端口，第一个端口用于Follower和Leader之间的数据同步和其它通信，第二个端口用于Leader选举过程中投票通信。                 |
| syncLimit                         |     | 表示Follower和Observer与Leader交互时的最大等待时间，只不过是在与leader同步完毕之后，进入正常请求转发或ping等消息交互时的超时时间。                                                        |
| group.x=nnnnn[:nnnnn]             |     | “x”是一个数字，与每个服务器的myid文件中的id是一样的。对机器分组，后面的参数是myid文件中的ID                                                                                    |
| weight.x=nnnnn                    |     | “x”是一个数字，与每个服务器的myid文件中的id是一样的。机器的权重设置，后面的参数是权重值                                                                                         |
| cnxTimeout                        | 5s  | 选举过程中打开一次连接的超时时间，默认是5s                                                                                                                   |
| standaloneEnabled                 |     | 当设置为false时，服务器在复制模式下启动                                                                                                                   |

注意：
zookeeper的启动日志在/bin目录下的zookeeper.out文件
在启动第一个节点后，查看日志信息会看到如下异常：

```
java.net.ConnectException: Connection refused at java.net.PlainSocketImpl.socketConnect(Native Method) at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:339) at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:200) at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:182) at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392) at java.net.Socket.connect(Socket.java:579) at org.apache.zookeeper.server.quorum.QuorumCnxManager.connectOne(QuorumCnxManager.java:368) at org.apache.zookeeper.server.quorum.QuorumCnxManager.connectAll(QuorumCnxManager.java:402) at org.apache.zookeeper.server.quorum.FastLeaderElection.lookForLeader(FastLeaderElection.java:840) at org.apache.zookeeper.server.quorum.QuorumPeer.run(QuorumPeer.java:762) 2016-07-30 17:13:16,032 [myid:1] - INFO [QuorumPeer[myid=1]/0:0:0:0:0:0:0:0:2181:FastLeaderElection@849] - Notification time out: 51200`
```
这是正常的，因为配置文件中配置了此节点是属于集群中的一个节点，zookeeper集群只有在过半的节点是正常的情况下，此节点才会正常，它是一直在检测集群其他两个节点的启动的情况。
那在我们启动第二个节点之后，我们会看到原先启动的第一个节点不会在报错，因为这时候已经有过半的节点是正常的了。

### 开启observer
- 在ZooKeeper中，observer默认是不开启的，需要手动开启
在zoo.cfg中添加如下属性：
```
peerType=observer
```
- 在要配置为观察者的主机后添加观察者标记。例如：
```
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2888:3888
server.3=127.0.0.1:2888:3888:observer #表示将该节点设置为观察者
```
- observer不投票不选举，所以observer的存活与否不影响这个集群的服务。
例如：一共25个节点，其中1个leader和6个follower，剩余18个都是observer。那么即使所有的observer全部宕机，ZooKeeper依然对外提供服务，但是如果4个follower宕机，即使所有的observer都存活，这个ZooKeeper也不对外提供服务 在ZooKeeper中过半性考虑的是leader和follower，而不包括observer==

## 常见指令

### 服务器端指令

| 指令                       | 说明        |
|--------------------------|-----------|
| `sh zkServer.sh start`   | 启动服务器端    |
| `sh zkServer.sh stop`    | 停止服务器端    |
| `sh zkServer.sh restart` | 重启服务器端    |
| `sh zkServer.sh status`  | 查看服务器端的状态 |
| `sh zkCli.sh`            | 启动客户端     |

### 客户端指令

| 命令                                 | 解释                                    |
|------------------------------------|---------------------------------------|
| help                               | 帮助命令                                  |
| `quit`                             | 退出客户端                                 |
| `ls /`                             | 查看根路径下的节点                             |
| `create /log 'manage log servers'` | 在根节点下创建一个子节点log                       |
| `creat -e /node2 ''`               | 在根节点下创建临时节点node2，客户端关闭后会删除            |
| `create -s /video ''`              | 在根节点下创建一个顺序节点`/video000000X`          |
| `creare -e -s /node4 ''`           | 创建一个临时顺序节点`/node4000000X`,客户端关闭后删除    |
| `get /video`                       | 查看video节点的数据以及节点信息                    |
| `delete /log`                      | 删除根节点下的子节点log<br />==要求被删除的节点下没有子节点== |
| `rmr /video`                       | 递归删除                                  |
| `set /video 'videos'`              | 修改节点数据                                |

### 常用四字命令

-  可以通过命令：echo stat|nc 127.0.0.1 2181 来查看哪个节点被选择作为follower或者leader
-  使用echo ruok|nc 127.0.0.1 2181 测试是否启动了该Server，若回复imok表示已经启动。
- echo dump| nc 127.0.0.1 2181 ,列出未经处理的会话和临时节点。
-  echo kill | nc 127.0.0.1 2181 ,关掉server
- echo conf | nc 127.0.0.1 2181 ,输出相关服务配置的详细信息。
-  echo cons | nc 127.0.0.1 2181 ,列出所有连接到服务器的客户端的完全的连接 / 会话的详细信息。
- echo envi |nc 127.0.0.1 2181 ,输出关于服务环境的详细信息（区别于 conf 命令）。
- echo reqs | nc 127.0.0.1 2181 ,列出未经处理的请求。
- echo wchs | nc 127.0.0.1 2181 ,列出服务器 watch 的详细信息。
- echo wchc | nc 127.0.0.1 2181 ,通过 session 列出服务器 watch 的详细信息，它的输出是一个与 watch 相关的会话的列表。
- echo wchp | nc 127.0.0.1 2181 ,通过路径列出服务器 watch 的详细信息。它输出一个与 session 相关的路径。

# 参考资料

Apache ZooKeeper 官网：[https://zookeeper.apache.org/](https://zookeeper.apache.org/)

Apache ZooKeeper r3.7.0原版文档：[https://zookeeper.apache.org/doc/r3.7.0/index.html](https://zookeeper.apache.org/doc/r3.7.0/index.html)

Apache ZooKeeper r3.5.6中文文档：[https://www.docs4dev.com/docs/zh/zookeeper/r3.5.6/reference/](https://www.docs4dev.com/docs/zh/zookeeper/r3.5.6/reference/)
