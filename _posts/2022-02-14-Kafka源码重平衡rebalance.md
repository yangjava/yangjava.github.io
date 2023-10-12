---
layout: post
categories: [Kafka]
description: none
keywords: Kafka
---
# Kafka源码重平衡rebalance
rebalance是重新进行负载均衡的过程。包括集群的rebalance，消费和生产者的rebalance 。

## 集群的rebalance
broker的rebalance：每个broker上的分区leader个数尽量保持一致，从而保证生产或者消费时节点的cpu,io，磁盘等保持基本一致。默认每个broker上的leader差异超过10%此时会触发broker上的leader迁移。

## 消费者
它本质上是一组协议，给消费者分配 partition 的行为叫做 rebalance，rebalance 只有在消费组状态发生改变时才会被触发，包括 成员加入组，离开组，成员metadata 修改等。

它规定了一个 consumer group 是如何达成一致来分配订阅 topic 的所有分区的。比方说Consumer Group A 有3个consumer 实例，它要消费一个拥有6个分区的topic，每个consumer消费2个分区，这就是rebalance。

rebalance是相对于consumer group 而言，每个consumer group会从kafka broker中选出一个作为组协调者（group coordinator）。coordinator负责对整个consumer group的状态进行管理，当有触发rebalance的条件发生时，促使生成新的分区分配方案。

主要涉及类
```
AbstractCoordinator：管理消费者组的协调过程，定义的接口，其还有具体的实现类用于实现分区分配算法
ConsumerCoordinator：对AbstractCoordinator的实现，管理消费者组的协调过程，直接与groupCoordinator进行交互
GroupCoordinator：coordinator负责对整个consumer group的状态进行管理
```
我们来分析一下他们的共同点Coordinator（协调器）

## Coordinator（协调器）
每个broker启动的时候都会创建一个GroupCoordinator，每个客户端都有一个ConsumerCoordinator协调器。 ConsumerCoordinator每间隔3S中就会和GroupCoordinator保持心跳，如果超时没有发送，并且再过Zookeeper.time.out = 10S中则会触发rebanlance。

GroupCoordinator
- 接受ConsumerCoordinator的JoinGroupRequest请求
- 选择一个consumer作为leader，一般第一个会作为leader
- 和consumer保持心跳，默认3S中，参数是：heartbeat.interval.ms
- 将consumer  commit过来的offset保存在__consumer_offsets中
这是内部topic，还有一个是：__transaction_state
这个topic默认50个分区，1个副本。副本数一般要修改为3，增加高可用性

ConsumerCoordinator
- 向GroupCoordinator发送JoinGroupRequest请求加入group
- 定时发送心跳给GroupCoordinator，默认是3s钟
- 发送SyncGroup请求（携带自己的分配策略），得到自己所分配到的partition
- 向GroupCoordinator定时发送offset偏移量，默认5Scommit一次
- 如果是leader Consumer，还要负责分区分配工作，

ConsumerCoordinator如何找到GroupCoordinator
- consumer向任何一个broker发送groupCoordinator发现请求，并携带上groupID
- broker通过groupId.hash % __consumer_offsets的partitions数量（默认50个分区）得到分区id、
此分区leader所在broker就是此groupCoordinator所在broker。
- consumerCoordinator找到GroupCoordinator之后，发送JoinGroupRequest请求加入group。
- GroupCoordinator调用handleJoinGroup方法处理请求。

源码分析三种consumer消费时分区分配策略

kafka新版本提供了三种consumer分区分配策略：
```
range 范围分配：org/apache/kafka/clients/consumer/RangeAssignor.java
round-robin 轮询：org/apache/kafka/clients/consumer/RoundRobinAssignor.java
sticky 
```
会在执行poll操作时进行制定分配策略。

分区分配策略由partition.assignment.strategy参数设置，默认为org.apache.kafka.clients.consumer.RangeAssignor。

消费者Rebalance
触发消费者的rebalance 分为多次，注册消费者时，消费者变动时。

Rebalance的不良影响：

发生 Rebalance 时，consumer group 下的所有 consumer 都会协调在一起共同参与，Kafka 使用分配策略尽可能达到最公平的分配
Rebalance 过程会对 consumer group 产生非常严重的影响，Rebalance 的过程中所有的消费者都将停止工作，直到 Rebalance 完成

注册consumer group时
Kafka的Consumer Rebalance的控制策略是由每一个Consumer通过在Zookeeper上注册Watch完成的。每个Consumer被创建时会触发 Consumer Group的Rebalance。也就是每个consumer消费那些topic的哪些partition

注册时consumer是如何分配partition的
分区分配策略决定了将topic中partition分配给consumer group中consumer实例的方式。

可以通过消费者客户端参数partition.assignment.strategy来设置消费者与主题之间的分区分配策略

### range 范围分配策略
原理是按消费者总数和分区总数进行整除运算来获得一个跨度，然后将分区按照跨度进行平均分配，以保证分区尽可能平均的分配给所有的消费者。

假设 n = 分区数/消费者数量，m= 分区数%消费者数量，那么前m个消费者每个分配n+1个分区，后面的（消费者数量-m）个消费者每个分配n个分区。
```
假如有10个分区，3个消费者，把分区按照序号排列0，1，2，3，4，5，6，7，8，9；
消费者为C1,C2,C3，那么用分区数除以消费者数来决定每个Consumer消费几个Partition，
除不尽的前面几个消费者将会多消费一个
最后分配结果如下

C1：0，1，2，3
C2：4，5，6
C3：7，8，9

```

### round-robin轮询分配策略
将消费者组内所有主题的分区按照字典序排序，然后通过轮询的方式逐个将分区一次分配给每个消费者。
比如假如有8个分区，3个消费者，把分区按照序号排列0，1，2，3，4，5，6，7；消费者为C0,C1,C2
首轮 c0 p0;c1 p1;c2 p2。往后依次分配C0,C1,C2


### sticky粘性分配策略
是从0.11.x版本开始引入的分配策略，它主要有两个目的：

（1）没有发生 rebalance 时，Striky 粘性分配策略和 RoundRobin 分配策略类似，分区的分配要尽可能均匀。

（2）发生reblance时，分区的分配尽可能与前分配的保持相同。

由上可知不管是轮询还是范围分配重启消费者后对于固定个数的分区和consumer个数，总是会分配固定的分区给consumer，对于sticky是更侧重reblance时使用

## 消费者变动
- Topic/Partition的增减
- Consumer的加入或者停止
- consumer失联

大致触发流程：
每个consumer 都会根据 heartbeat.interval.ms 参数指定的时间周期性地向group coordinator发送 hearbeat，group coordinator会给各个consumer响应，若发生了 rebalance，各个consumer收到的响应中会包含 REBALANCE_IN_PROGRESS 标识，这样各个consumer就知道已经发生了rebalance，同时 group coordinator也知道了各个consumer的存活情况。

具体触发rebalance场景如下。

- consumer失联，consumer挂掉或者consumer负载太高导致他不能正常和groupCoordinator保持3S中心跳，在等待session.time.out=10S还没有心跳，则会触发rebanlance。负载过高注意查看是否频繁gc
- 新的consumer加入group，一般是那种有重试机制的框架，比如
spark-streaming重新启动task任务加入group
新的task顶替失败的task，重新加入group，所以他第一步是先发现group协调器，然后join。
- groupCoordinator挂了
当consumer发送心跳给GroupCoordinator没有响应，则会认为group失败，此时则会重新查找groupCoordinator，然后join。
GroupCoordinator失效是一个很严重的问题，说明某个GroupID对应的consumer非常多，导致太繁忙没法及时反应，或者集群问题导致__consumer_offsets的分区leader重新选举也会有这样的。比如spark-streaming有很多任务，每个任务的groupID都是一样的，每个任务搞了几个executor。这种情况需要拆分为几个groupID，自己负责自己的就行了

Coordinator收到通知准备发起Rebalance操作。

rebalance的过程： 此时Consumer停止fetch数据，commit offset，并发送JoinGroupRequest给它的Coordinator（consumer group中的所有的consumer所在的所有broker,其中选出一个作为Coordinator），并在JoinGroupResponse中获得它应该拥有的所有 Partition列表和它所属的Group的新的Generation ID。此时Rebalance完成。

### 消费者避免rebalance
解决：增加session.timeout.ms时间，可以有效延长触发reblance。增加request.timeout.ms，设置合适的max.poll.records值。
request.timeout.ms为组协调组向kafak发送心跳的时间，证明消费者组中的消费者保持活动状态，在session时间内请求后判定消费者离线触发重平衡机制。session.timeout.ms/request.timeout.ms=心跳验证的次数。所以request.timeout.ms时间建议为不超过session.timeout.ms的1/3且必须不能超过session.timeout.ms。
max.poll.records：设置一次poll最大消息记录数，避免poll的数据一直消费不完，超过session.timeout.ms时间触发relance。







































