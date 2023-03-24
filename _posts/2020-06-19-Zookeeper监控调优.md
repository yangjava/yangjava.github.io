---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---
# Zookeeper监控调优
zookeeper的监控命令需要通过telnet或者nc工具向zookeeper服务进行提交。

## 开启监控命令
连接zookeeper成功可以使用四字监控命令进行操作。在连接建立之后输入对应的命令后回车。
在使用监控命令之前，需要修改zookeeper的配置文件，开启四字监控命令，否则会报错
修改zoo.cfg文件，在配置文件中加入如下配置：
```
4lw.commands.whitelist=*
```

## 四字命令

### conf
能够获取到zookeeper的配置信息，包括
```
clientPort=2181
dataDir=/export/servers/zookeeper-3.4.6/data/version-2
dataLogDir=/export/servers/zookeeper-3.4.6/logs/version-2
tickTime=2000
maxClientCnxns=60
minSessionTimeout=4000
maxSessionTimeout=40000
serverId=2
initLimit=10
syncLimit=5
electionAlg=3
electionPort=3888
quorumPort=2888
peerType=0
```
clientPort:客户端端口号
dataDir:数据快照文件保存目录（默认情况下，每10w次事务操作保存一次快照）
dataLogDir:事务日志目录（该配置如果没有在zoo.cfg单独配置的，会和dataDir在同一目录下）
tickTime:服务器与服务器之前或者客户端与服务器之间维持连接的心跳时长
maxClientCnxns:最大连接数
minSessionTimeout:最小session超时时间=tickTime*2
maxSessionTimeout:最大session超时时间=tickTime*20
serverId:服务器序号
initLimit:集群中的follow节点和leader节点在初始连接时能容忍的最大心跳数（也就是初始连接时在多少次心跳后放弃）
syncLimit:集群中的follow节点和leader节点在请求和应答时能容忍的最大心跳数
electionAlg:失去leader节点后重新选举leader的算法类型，默认为3，0-基于UDP的LeaderELection 1-基于UDP的FastLeaderElection 2-基于TCP的FastLeaderElection
electionPort:当前节点在集群进行leader选举时使用的端口号
quorumPort:当前节点在集群进行数据互通使用的端口号
peerType:当前节点是否是观察者节点observer，0-不是 1-是

### cons
返回所有连接到这台服务器的客户端/会话信息
```
/127.0.0.1:54340[1](queued=0,recved=1,sent=1,sid=0x10000b785090000,lop=SESS,est=1622032985181,to=30000,lcxid=0x0,lzxid=0x2b,lresp=12136376,llat=20,minlat=0,avglat=20,maxlat=20)
/127.0.0.1:54348[0](queued=0,recved=1,sent=0)
```
ip    客户端ip地址
port	客户端发送数据的端口号
queued	当前会话等待被处理的请求数，请求缓存在队列中
recved	当前会话对服务器操作后，服务器收到的数据包个数
sent	当前会话操作，导致服务器发送的包个数
sid	会话id
lop	当前会话在最后一次操作的类型：GETD-读取数据、DELE-删除数据、CREA-创建数据
est	客户端创建连接时的时间戳
to	当前会话的超时时间
lcxid	当前会话的操作id，每当客户端操作一次zookeeper，这给id就会自增1（十六进制数）
lzxid	当前最大事务id
lresp	最后一次操作的响应时间（时间戳）
minlat	最小延迟
avglat	平均延迟
maxlat	最大延迟
llat	最后延迟

### crst
重置当前这台服务器所有连接/会话的统计信息，即重置了 cons 命令查询的信息。重置连接状态，是一个execute 操作 不是一个select 操作
执行后返回一个状态信息：
```
Connection stats reset.
```

### dump
输出所有等待队列中的会话和临时节点的信息
```
SessionTracker dump:
Session Sets (3)/(1):
0 expire at Wed May 26 21:21:28 CST 2021:
0 expire at Wed May 26 21:21:38 CST 2021:
1 expire at Wed May 26 21:21:44 CST 2021:
        0x10000b785090000
ephemeral nodes dump:
Sessions with Ephemerals (1):
0x10000b785090000:
        /temp
Connections dump:
Connections Sets (3)/(2):
0 expire at Wed May 26 21:21:28 CST 2021:
1 expire at Wed May 26 21:21:38 CST 2021:
        ip: /127.0.0.1:56079 sessionId: 0x0
1 expire at Wed May 26 21:21:48 CST 2021:
        ip: /127.0.0.1:54340 sessionId: 0x10000b785090000

```

### envi
输出zookeeper所在服务器的环境配置信息
```
Environment:
zookeeper.version=3.4.6-1569965, built on 02/20/2014 09:09 GMT
host.name=hhz112
java.version=1.7.0_60
java.vendor=Oracle Corporation
java.home=/export/servers/jdk1.7.0_60/jre
java.class.path=/export/servers/zookeeper-3.4.6/bin/../build/classes:/export/servers/zookeeper-3.4.6/bin/../build/lib/*.jar:/export/servers/zookeeper-3.4.6/bin/../lib/slf4j-log4j12-1.6.1.jar:/export/servers/zookeeper-3.4.6/bin/../lib/slf4j-api-1.6.1.jar:/export/servers/zookeeper-3.4.6/bin/../lib/netty-3.7.0.Final.jar:/export/servers/zookeeper-3.4.6/bin/../lib/log4j-1.2.16.jar:/export/servers/zookeeper-3.4.6/bin/../lib/jline-0.9.94.jar:/export/servers/zookeeper-3.4.6/bin/../zookeeper-3.4.6.jar:/export/servers/zookeeper-3.4.6/bin/../src/java/lib/*.jar:/export/servers/zookeeper-3.4.6/bin/../conf:/export/servers/zookeeper-3.4.6/bin/../build/classes:/export/servers/zookeeper-3.4.6/bin/../build/lib/*.jar:/export/servers/zookeeper-3.4.6/bin/../lib/slf4j-log4j12-1.6.1.jar:/export/servers/zookeeper-3.4.6/bin/../lib/slf4j-api-1.6.1.jar:/export/servers/zookeeper-3.4.6/bin/../lib/netty-3.7.0.Final.jar:/export/servers/zookeeper-3.4.6/bin/../lib/log4j-1.2.16.jar:/export/servers/zookeeper-3.4.6/bin/../lib/jline-0.9.94.jar:/export/servers/zookeeper-3.4.6/bin/../zookeeper-3.4.6.jar:/export/servers/zookeeper-3.4.6/bin/../src/java/lib/*.jar:/export/servers/zookeeper-3.4.6/bin/../conf:.:/export/servers/jdk1.6.0_25/lib/dt.jar:/export/servers/jdk1.6.0_25/lib/tools.jar
java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
java.io.tmpdir=/tmp
java.compiler=<NA>
os.name=Linux
os.arch=amd64
os.version=2.6.32-358.el6.x86_64
user.name=hhz
user.home=/home/hhz
user.dir=/export/servers/zookeeper-3.4.6
```

当前server的环境信息：

版本信息

主机的host

jvm相关参数：version，classpath，lib等等

os相关参数：name，version等等

当前host用户信息：name，dir等等

6


### ruok
查询当前server状态是否正常 若正常返回imok
```
imok
```

### srst:

同样是一个execute操作而不是select，重置server状态：

Server stats reset.


### srvr
server的简要信息：

版本

延时

收包数

发包数

连接数

状态等信息

Zookeeper version: 3.4.6-1569965, built on 02/20/2014 09:09 GMT
Latency min/avg/max: 0/0/182
Received: 97182
Sent: 97153
Connections: 22
Outstanding: 8
Zxid: 0x68000af381
Mode: follower
Node count: 101065

### stat:
一些状态信息和连接信息，是前面一些信息的组合：

Zookeeper version: 3.4.6-1569965, built on 02/20/2014 09:09 GMT
Clients:
/192.168.147.102:56168[1](queued=0,recved=41,sent=41)
/192.168.144.102:34378[1](queued=0,recved=54,sent=54)
/192.168.162.16:43108[1](queued=0,recved=40,sent=40)
/192.168.144.107:39948[1](queued=0,recved=1421,sent=1421)
/192.168.162.16:43112[1](queued=0,recved=54,sent=54)
/192.168.162.16:43107[1](queued=0,recved=54,sent=54)
/192.168.162.16:43110[1](queued=0,recved=53,sent=53)
/192.168.144.98:34702[1](queued=0,recved=41,sent=41)
/192.168.144.98:34135[1](queued=0,recved=61,sent=65)
/192.168.162.16:43109[1](queued=0,recved=54,sent=54)
/192.168.147.102:56038[1](queued=0,recved=165313,sent=165314)
/192.168.147.102:56039[1](queued=0,recved=165526,sent=165527)
/192.168.147.101:44124[1](queued=0,recved=162811,sent=162812)
/192.168.147.102:39271[1](queued=0,recved=41,sent=41)
/192.168.144.107:45476[1](queued=0,recved=166422,sent=166423)
/192.168.144.103:45100[1](queued=0,recved=54,sent=54)
/192.168.162.16:43133[0](queued=0,recved=1,sent=0)
/192.168.144.107:39945[1](queued=0,recved=1825,sent=1825)
/192.168.144.107:39919[1](queued=0,recved=325,sent=325)
/192.168.144.106:47163[1](queued=0,recved=17891,sent=17891)
/192.168.144.107:45488[1](queued=0,recved=166554,sent=166555)
/172.17.36.11:32728[1](queued=0,recved=54,sent=54)
/192.168.162.16:43115[1](queued=0,recved=54,sent=54)

Latency min/avg/max: 0/0/599
Received: 224869
Sent: 224817
Connections: 23
Outstanding: 0
Zxid: 0x68000af707
Mode: follower
Node count: 101081


wchs:
有watch path的连接数 以及watch的path数 和 watcher数

13 connections watching 102 paths
Total watches:172


wchc:
连接监听的所有path：(考虑吧cons命令 信息整合)

0x24b3673bb14001f
/hbase/root-region-server
/hbase/master


wchp:

path被那些连接监听：（考虑把cons命令 信息整合）


/dubbo/FeedInterface/configurators
0x4b3673ce4a1a4d
/dubbo/UserInterface/providers
0x14b36741ee41b17
0x4b3673ce4a1a4d
0x24b3673bb1401d2
0x4b3673ce4a1ab7


mntr：

用于监控zookeeper server 健康状态的各种指标：

版本

延时

收包

发包

连接数

未完成客户端请求数

leader/follower 状态

znode 数

watch 数

临时节点数

近似数据大小 应该是一个总和的值

打开文件描述符 数

最大文件描述符 数

fllower数

等等

zk_version	3.4.6-1569965, built on 02/20/2014 09:09 GMT
zk_avg_latency	0
zk_max_latency	2155
zk_min_latency	0
zk_packets_received	64610660
zk_packets_sent	64577070
zk_num_alive_connections	42
zk_outstanding_requests	0
zk_server_state	leader
zk_znode_count	101125
zk_watch_count	315
zk_ephemerals_count	633
zk_approximate_data_size	27753592
zk_open_file_descriptor_count	72
zk_max_file_descriptor_count	4096
zk_followers	2
zk_synced_followers	2
zk_pending_syncs	0


## ZooKeeper监控
```
zk_avg/min/max_latency    响应一个客户端请求的时间，建议这个时间大于10个Tick就报警
 
zk_outstanding_requests        排队请求的数量，当ZooKeeper超过了它的处理能力时，这个值会增大，建议设置报警阀值为10
 
zk_packets_received      接收到客户端请求的包数量
 
zk_packets_sent        发送给客户单的包数量，主要是响应和通知
 
zk_max_file_descriptor_count   最大允许打开的文件数，由ulimit控制
 
zk_open_file_descriptor_count    打开文件数量，当这个值大于允许值得85%时报警
 
Mode                运行的角色，如果没有加入集群就是standalone,加入集群式follower或者leader
 
zk_followers          leader角色才会有这个输出,集合中follower的个数。正常的值应该是集合成员的数量减1
 
zk_pending_syncs       leader角色才会有这个输出，pending syncs的数量
 
zk_znode_count         znodes的数量
 
zk_watch_count         watches的数量
 
Java Heap Size         ZooKeeper Java进程的

```

## 实例
```shell
# echo ruok|nc 127.0.0.1 2181
imok
 
# echo mntr|nc 127.0.0.1 2181
zk_version	3.4.6-1569965, built on 02/20/2014 09:09 GMT
zk_avg_latency	0
zk_max_latency	0
zk_min_latency	0
zk_packets_received	11
zk_packets_sent	10
zk_num_alive_connections	1
zk_outstanding_requests	0
zk_server_state	leader
zk_znode_count	17159
zk_watch_count	0
zk_ephemerals_count	1
zk_approximate_data_size	6666471
zk_open_file_descriptor_count	29
zk_max_file_descriptor_count	102400
zk_followers	2
zk_synced_followers	2
zk_pending_syncs	0

# echo srvr|nc 127.0.0.1 2181
Zookeeper version: 3.4.6-1569965, built on 02/20/2014 09:09 GMT
Latency min/avg/max: 0/0/0
Received: 26
Sent: 25
Connections: 1
Outstanding: 0
Zxid: 0x500000000
Mode: leader
Node count: 17159
```