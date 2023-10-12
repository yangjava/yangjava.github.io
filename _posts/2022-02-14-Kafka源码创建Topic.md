---
layout: post
categories: [Kafka]
description: none
keywords: Kafka
---
# Kafka源码创建Topic

## 创建方式
有两种创建方式：
- 通过shell命令 kafka-topics.sh 创建一个 topic，可以设置相应的副本数让 Server 端自动进行 replica 分配，也可以直接指定手动 replica 的分配。
- Server 端如果 auto.create.topics.enable 设置为 true 时，那么当 Producer 向一个不存在的 topic 发送数据时，该 topic 同样会被创建出来，此时，副本数默认是1。

## 创建 topic流程分析
在 Kafka 的安装目录下，通过下面这条命令可以创建一个 partition 为3，replica 为2的 topic（test）
```
./bin/kafka-topics.sh --create --topic test --zookeeper XXXX --partitions 3 --replication-factor 2
```
无论是生产者创建还是shell kafka-topics.sh 创建实际上是调用 kafka.admin.TopicCommand 的方法来创建 topic，其实现如下：
```
  def createTopic(zkClient: KafkaZkClient, opts: TopicCommandOptions) {
    val topic = opts.options.valueOf(opts.topicOpt)
    val configs = parseTopicConfigsToBeAdded(opts)
    val ifNotExists = opts.options.has(opts.ifNotExistsOpt)
    if (Topic.hasCollisionChars(topic))
      println("WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.")
    val adminZkClient = new AdminZkClient(zkClient)
    try {
      if (opts.options.has(opts.replicaAssignmentOpt)) { //判断是否指定了副本分配
        val assignment = parseReplicaAssignment(opts.options.valueOf(opts.replicaAssignmentOpt))
        adminZkClient.createOrUpdateTopicPartitionAssignmentPathInZK(topic, assignment, configs, update = false)
      } else { //没有指定副本时，使用默认的算法
        CommandLineUtils.checkRequiredArgs(opts.parser, opts.options, opts.partitionsOpt, opts.replicationFactorOpt)
        val partitions = opts.options.valueOf(opts.partitionsOpt).intValue
        val replicas = opts.options.valueOf(opts.replicationFactorOpt).intValue
        val rackAwareMode = if (opts.options.has(opts.disableRackAware)) RackAwareMode.Disabled
                            else RackAwareMode.Enforced
       //执行创建topic逻辑，副本分配方式在此实现，详情见下方副本分配方式
        adminZkClient.createTopic(topic, partitions, replicas, configs, rackAwareMode) 
      }
      println("Created topic \"%s\".".format(topic))
    } catch  {
      case e: TopicExistsException => if (!ifNotExists) throw e
    }
  }

```
副本的分配方式
为了在不考虑机架的情况下
- 从broker节点列表中的随机选择出首个副本分配位置，首个选出后通过循环分配每个分区的第一个副本。
- 以递增的移位量分配每个分区的其余副本。

具体源码实现在下面，可以发现主要步骤如下
```
  private def assignReplicasToBrokersRackUnaware(nPartitions: Int,//分区数
                          replicationFactor: Int,//副本因子，即几个副本
                          brokerList: Seq[Int],//集群中broker列表
                          fixedStartIndex: Int,//起始索引，第一个索引的分配位置，默认不设置为-1，起到循环中递增作用
                          					   //ret结果的replicaBuffer第一列的2-1-0的实现即是靠fixedStartIndex
                          startPartitionId: Int): Map[Int, Seq[Int]] = {

    // ret表示<partition,Seq[replica所在brokerId]>的关系
    val ret = mutable.Map[Int, Seq[Int]]()
    val brokerArray = brokerList.toArray
    //通过随机方式拿到首个副本分配节点
    val startIndex = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(brokerArray.length)
    var currentPartitionId = math.max(0, startPartitionId) //startPartitionId指定从哪个分区开始分配，默认-1

    //随机获取副本分配的间隔，随机数为1到brokerArray.length
    var nextReplicaShift = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(brokerArray.length)

    // 轮询所有分区，将每个分区的副本分配到不同的broker上，计算过程为每次计算一个分区的所有副本分配位置，分配完毕开始下一个分区计算。
    for (_ <- 0 until nPartitions) {
      if (currentPartitionId > 0 && (currentPartitionId % brokerArray.length == 0)) {
        nextReplicaShift += 1

      }
      //算出首个副本分配的broker,第一个分区分配后，则第二次的分区分配为第一次分区位置+1
      val firstReplicaIndex = (currentPartitionId + startIndex) % brokerArray.length
      //replicaBuffer存储分配的副本集合
      val replicaBuffer = mutable.ArrayBuffer(brokerArray(firstReplicaIndex))
      for (j <- 0 until replicationFactor - 1)
        replicaBuffer += brokerArray(replicaIndex(firstReplicaIndex, nextReplicaShift, j, brokerArray.length))
      ret.put(currentPartitionId, replicaBuffer)

      // 继续为下一个分区分配副本
      currentPartitionId += 1
    }
    ret
  }

```
最终ret为分配结果，结果大致如下，buffer中的第一个为Leader副本。
```
分区0 replicaBuffer(2, 1, 0)
分区1 replicaBuffer(0, 2, 1)
分区2 replicaBuffer(1, 0, 2)
分区3 replicaBuffer(2, 0, 1)
分区4 replicaBuffer(0, 1, 2)
分区5 replicaBuffer(1, 2, 0)
```
分配结果拿到后进行topic和分区有效性验证，然后在zookeeper上注册topic信息，完成创建

