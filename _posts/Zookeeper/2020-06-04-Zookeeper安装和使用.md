---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---

## Zookeeper实战

### 单机版安装

1.解压zookeeper安装包（本人重命名为zookeeper，并移动到/usr/local路径下），此处只有解压命令

```
　tar -zxvf zookeeper-3.4.5.tar.gz
```

2.进入到zookeeper文件夹下，并创建data和logs文件夹（一般解压后都有data文件夹）

```
　　[root@localhost zookeeper]# cd /usr/local/zookeeper/

　　[root@localhost zookeeper]# mkdir logs
```

3.在conf目录下修改zoo.cfg文件（如果没有此文件，则自己新建该文件），修改为如下内容:

```
tickTime=2000
dataDir=/usr/myapp/zookeeper-3.4.5/data
dataLogDir=/usr/myapp/zookeeper-3.4.5/logs
clientPort=2181
```

最低配置

| 参数名                                | 默认    | 描述                                                         |
| ------------------------------------- | ------- | ------------------------------------------------------------ |
| clientPort                            |         | 服务的监听端口                                               |
| dataDir                               |         | 用于存放内存数据快照的文件夹，同时用于集群的myid文件也存在这个文件夹里 |
| tickTime                              | 2000    | Zookeeper的时间单元。Zookeeper中所有时间都是以这个时间单元的整数倍去配置的。例如，session的最小超时时间是2*tickTime。（单位：毫秒） |
| dataLogDir                            |         | 事务日志写入该配置指定的目录，而不是“ dataDir ”所指定的目录。这将允许使用一个专用的日志设备并且帮助我们避免日志和快照之间的竞争 |
| globalOutstandingLimit                | 1,000   | 最大请求堆积数。默认是1000。Zookeeper运行过程中，尽管Server没有空闲来处理更多的客户端请求了，但是还是允许客户端将请求提交到服务器上来，以提高吞吐性能。当然，为了防止Server内存溢出，这个请求堆积数还是需要限制下的。 |
| preAllocSize                          | 64M     | 预先开辟磁盘空间，用于后续写入事务日志。默认是64M，每个事务日志大小就是64M。如果ZK的快照频率较大的话，建议适当减小这个参数。 |
| snapCount                             | 100,000 | 每进行snapCount次事务日志输出后，触发一次快照， 此时，Zookeeper会生成一个snapshot.*文件，同时创建一个新的事务日志文件log.*。默认是100,000. |
| traceFile                             |         | 用于记录所有请求的log，一般调试过程中可以使用，但是生产环境不建议使用，会严重影响性能。 |
| maxClientCnxns                        |         | 最大并发客户端数，用于防止DOS的，默认值是10，设置为0是不加限制 |
| clientPortAddress / maxSessionTimeout |         | 对于多网卡的机器，可以为每个IP指定不同的监听端口。默认情况是所有IP都监听 clientPort 指定的端口 |
| minSessionTimeout                     |         | Session超时时间限制，如果客户端设置的超时时间不在这个范围，那么会被强制设置为最大或最小时间。默认的Session超时时间是在2 * tickTime ~ 20 * tickTime 这个范围 |
| fsync.warningthresholdms              | 1000    | 事务日志输出时，如果调用fsync方法超过指定的超时时间，那么会在日志中输出警告信息。默认是1000ms。 |
| autopurge.snapRetainCount             |         | 参数指定了需要保留的事务日志和快照文件的数目。默认是保留3个。和autopurge.purgeInterval搭配使用 |
| autopurge.purgeInterval               |         | 在3.4.0及之后版本，Zookeeper提供了自动清理事务日志和快照文件的功能，这个参数指定了清理频率，单位是小时，需要配置一个1或更大的整数，默认是0，表示不开启自动清理功能 |
| syncEnabled                           |         | Observer写入日志和生成快照，这样可以减少Observer的恢复时间。默认为true。 |

4.进入bin目录，启动、停止、重启分和查看当前节点状态

```
　　[root@localhost bin]# ./zkServer.sh start
　　[root@localhost bin]# ./zkServer.sh stop
　　[root@localhost bin]# ./zkServer.sh restart
　　[root@localhost bin]# ./zkServer.sh status
```

### 集群安装

zookeeper集群简介

Zookeeper集群中只要有过半的节点是正常的情况下,那么整个集群对外就是可用的。正是基于这个特性,要将 ZK 集群的节点数量要为奇数（2n+1），如 3、5、7 个节点)较为合适。

1.解压zookeeper安装包（本人重命名为zookeeper，并移动到/usr/local路径下），此处只有解压命令

```
$ tar -zxvf zookeeper-3.4.6.tar.gz
```

2.在各个zookeeper节点目录创建data、logs目录

```
　　[root@localhost zookeeper]# cd /usr/local/zookeeper/

　　[root@localhost zookeeper]# mkdir logs
```

3.修改zoo.cfg配置文件

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

​		①、tickTime：基本事件单元，这个时间是作为Zookeeper服务器之间或客户端与服务器之间维持心跳的时间间隔，每隔tickTime时间就会发送一个心跳；最小 的session过期时间为2倍tickTime

②、dataDir：存储内存中数据库快照的位置，除非另有说明，否则指向数据库更新的事务日志。注意：应该谨慎的选择日志存放的位置，使用专用的日志存储设备能够大大提高系统的性能，如果将日志存储在比较繁忙的存储设备上，那么将会很大程度上影像系统性能。

③、client：监听客户端连接的端口。

④、initLimit：允许follower连接并同步到Leader的初始化连接时间，以tickTime为单位。当初始化连接时间超过该值，则表示连接失败。

⑤、syncLimit：表示Leader与Follower之间发送消息时，请求和应答时间长度。如果follower在设置时间内不能与leader通信，那么此follower将会被丢弃。

⑥、server.A=B:C:D

A：其中 A 是一个数字，表示这个是服务器的编号；

B：是这个服务器的 ip 地址；

C：Zookeeper服务器之间的通信端口；

D：Leader选举的端口。

我们需要修改的第一个是 dataDir ,在指定的位置处创建好目录。

第二个需要新增的是 server.A=B:C:D 配置，其中 A 对应下面我们即将介绍的myid 文件。B是集群的各个IP地址，C:D 是端口配置。

*集群选项*

| 参数名                            | 默认 | 描述                                                         |
| --------------------------------- | ---- | ------------------------------------------------------------ |
| electionAlg                       |      | 之前的版本中， 这个参数配置是允许我们选择leader选举算法，但是由于在以后的版本中，只有“FastLeaderElection ”算法可用，所以这个参数目前看来没有用了。 |
| initLimit                         | 10   | Observer和Follower启动时，从Leader同步最新数据时，Leader允许initLimit * tickTime的时间内完成。如果同步的数据量很大，可以相应的把这个值设置的大一些。 |
| leaderServes                      | yes  | 默 认情况下，Leader是会接受客户端连接，并提供正常的读写服务。但是，如果你想让Leader专注于集群中机器的协调，那么可以将这个参数设置为 no，这样一来，会大大提高写操作的性能。一般机器数比较多的情况下可以设置为no，让Leader不接受客户端的连接。默认为yes |
| server.x=[hostname]:nnnnn[:nnnnn] |      | “x”是一个数字，与每个服务器的myid文件中的id是一样的。hostname是服务器的hostname，右边配置两个端口，第一个端口用于Follower和Leader之间的数据同步和其它通信，第二个端口用于Leader选举过程中投票通信。 |
| syncLimit                         |      | 表示Follower和Observer与Leader交互时的最大等待时间，只不过是在与leader同步完毕之后，进入正常请求转发或ping等消息交互时的超时时间。 |
| group.x=nnnnn[:nnnnn]             |      | “x”是一个数字，与每个服务器的myid文件中的id是一样的。对机器分组，后面的参数是myid文件中的ID |
| weight.x=nnnnn                    |      | “x”是一个数字，与每个服务器的myid文件中的id是一样的。机器的权重设置，后面的参数是权重值 |
| cnxTimeout                        | 5s   | 选举过程中打开一次连接的超时时间，默认是5s                   |
| standaloneEnabled                 |      | 当设置为false时，服务器在复制模式下启动                      |

注意：
zookeeper的启动日志在/bin目录下的zookeeper.out文件
在启动第一个节点后，查看日志信息会看到如下异常：

```
java.net.ConnectException: Connection refused at java.net.PlainSocketImpl.socketConnect(Native Method) at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:339) at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:200) at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:182) at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392) at java.net.Socket.connect(Socket.java:579) at org.apache.zookeeper.server.quorum.QuorumCnxManager.connectOne(QuorumCnxManager.java:368) at org.apache.zookeeper.server.quorum.QuorumCnxManager.connectAll(QuorumCnxManager.java:402) at org.apache.zookeeper.server.quorum.FastLeaderElection.lookForLeader(FastLeaderElection.java:840) at org.apache.zookeeper.server.quorum.QuorumPeer.run(QuorumPeer.java:762) 2016-07-30 17:13:16,032 [myid:1] - INFO [QuorumPeer[myid=1]/0:0:0:0:0:0:0:0:2181:FastLeaderElection@849] - Notification time out: 51200`
```

这是正常的，因为配置文件中配置了此节点是属于集群中的一个节点，zookeeper集群只有在过半的节点是正常的情况下，此节点才会正常，它是一直在检测集群其他两个节点的启动的情况。
那在我们启动第二个节点之后，我们会看到原先启动的第一个节点不会在报错，因为这时候已经有过半的节点是正常的了。

### 开启observer

> 在ZooKeeper中，observer默认是不开启的，需要手动开启

在zoo.cfg中添加如下属性：

```
peerType=observer
```

在要配置为观察者的主机后添加观察者标记。例如：

```
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2888:3888
server.3=127.0.0.1:2888:3888:observer #表示将该节点设置为观察者
```

> observer不投票不选举，所以observer的存活与否不影响这个集群的服务。
>
> 例如：一共25个节点，其中1个leader和6个follower，剩余18个都是observer。那么即使所有的observer全部宕机，ZooKeeper依然对外提供服务，但是如果4个follower宕机，即使所有的observer都存活，这个ZooKeeper也不对外提供服务
>
> ==在ZooKeeper中过半性考虑的是leader和follower，而不包括observer==

## 常见指令

### 服务器端指令

| 指令                     | 说明               |
| ------------------------ | ------------------ |
| `sh zkServer.sh start`   | 启动服务器端       |
| `sh zkServer.sh stop`    | 停止服务器端       |
| `sh zkServer.sh restart` | 重启服务器端       |
| `sh zkServer.sh status`  | 查看服务器端的状态 |
| `sh zkCli.sh`            | 启动客户端         |

### 客户端指令

| 命令                               | 解释                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| help                               | 帮助命令                                                     |
| `quit`                             | 退出客户端                                                   |
| `ls /`                             | 查看根路径下的节点                                           |
| `create /log 'manage log servers'` | 在根节点下创建一个子节点log                                  |
| `creat -e /node2 ''`               | 在根节点下创建临时节点node2，客户端关闭后会删除              |
| `create -s /video ''`              | 在根节点下创建一个顺序节点`/video000000X`                    |
| `creare -e -s /node4 ''`           | 创建一个临时顺序节点`/node4000000X`,客户端关闭后删除         |
| `get /video`                       | 查看video节点的数据以及节点信息                              |
| `delete /log`                      | 删除根节点下的子节点log<br />==要求被删除的节点下没有子节点== |
| `rmr /video`                       | 递归删除                                                     |
| `set /video 'videos'`              | 修改节点数据                                                 |

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