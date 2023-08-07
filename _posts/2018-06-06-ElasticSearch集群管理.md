---
layout: post
categories: ElasticSearch
description: none
keywords: ElasticSearch
---
# ElasticSearch集群管理
Elasticsearch正是利用集群进行水平扩展节点，达到支持海量数据的能力。

## 集群内部原理
分布式系统的集群方式大致可以分为主从（Master-Slave）模式和无主模式。ES、HDFS、HBase使用主从模式，Cassandra使用无主模式。

主从模式可以简化系统设计，Master作为权威节点，部分操作仅由Master执行，并负责维护集群元信息。

缺点是Master节点存在单点故障，需要解决灾备问题，并且集群规模会受限于Master节点的管理能力。

因此，从集群节点角色的角度划分，至少存在主节点和数据节点，另外还有协调节点、预处理节点和部落节点，

下面分别介绍各种类型节点的职能。

## 集群节点角色

### 主节点（Master node）
主节点负责集群层面的相关操作，管理集群变更。

通过配置`node.master: true`（默认）使节点具有被选举为Master 的资格。主节点是全局唯一的，将从有资格成为Master的节点中进行选举。

主节点也可以作为数据节点，但尽可能做少量的工作，因此生产环境应尽量分离主节点和数据节点，创建独立主节点的配置：
```
node.master: true
node.data: false
```
为了防止数据丢失，每个主节点应该知道有资格成为主节点的数量，默认为1。

为避免网络分区时出现多主的情况，配置`discovery.zen.minimum_master_nodes` 原则上最小值应该是：`（master_eligible_nodes / 2）+ 1`

### 数据节点（Data node）
负责保存数据、执行数据相关操作：CRUD、搜索、聚合等。数据节点对CPU、内存、I/O要求较高。

一般情况下，数据读写流程只和数据节点交互，不会和主节点打交道（异常情况除外）。

通过配置`node.data: true`（默认）来使一个节点成为数据节点，也可以通过下面的配置创建一个数据节点：
```
node.master: false
node.data: true
node.ingest: false
```

### 预处理节点（Ingest node）
这是从5.0版本开始引入的概念。预处理操作允许在索引文档之前，即写入数据之前，通过事先定义好的一系列的processors（处理器）和pipeline（管道），对数据进行某种转换、富化。

processors和pipeline拦截bulk和index请求，在应用相关操作后将文档传回给 index 或 bulk API。

默认情况下，在所有的节点上启用ingest，如果想在某个节点上禁用ingest，则可以添加配置`node.ingest: false`，也可以通过下面的配置创建一个仅用于预处理的节点：
```
node.master: false
node.data: false
node.ingest: true
```

### 协调节点（Coordinating node）
客户端请求可以发送到集群的任何节点，每个节点都知道任意文档所处的位置，然后转发这些请求，收集数据并返回给客户端，处理客户端请求的节点称为协调节点。

协调节点将请求转发给保存数据的数据节点。每个数据节点在本地执行请求，并将结果返回协调节点。协调节点收集完数据后，将每个数据节点的结果合并为单个全局结果。对结果收集和排序的过程可能需要很多CPU和内存资源。

通过下面的配置创建一个仅用于协调的节点：
```
node.master: false
node.data: false
node.ingest: false
```

###  部落节点（Tribe node）
tribes（部落）功能允许部落节点在多个集群之间充当联合客户端。

在ES 5.0之前还有一个客户端节点（Node Client）的角色，客户端节点有以下属性：
```
node.master: false
node.data: false
```
它不做主节点，也不做数据节点，仅用于路由请求，本质上是一个智能负载均衡器（从负载均衡器的定义来说，智能和非智能的区别在于是否知道访问的内容存在于哪个节点），从5.0版本开始，这个角色被协调节点（Coordinating only node）取代。

## 集群节点监控
在Elasticsearch的运行期间，一个很重要的方面就是监控。这使得系统管理员能够检测并预防可能性的问题，或至少知道失败时会发生什么。
Elasticsearch提供了非常详细的信息，使你能够检查和监控单个节点或一个整体的集群。包括集群的健康值、有关服务器的信息、节点信息、索引和分片信息等。
对Elasticsearch监控的API主要有三类：一类是集群相关的，以_cluster开头，第二类是监控节点相关的，以_nodes开头，第三类是任务相关的，已_tasks开头。

## 集群健康值
集群健康值可以通过集群健康检查API_cluster/health得到简单情况。
```shell
GET http://127.0.0.1:9200/_cluster/health?pretty=true
```
返回值：
```json
{
  "cluster_name" : "elasticsearch",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 16,
  "active_shards" : 16,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 15,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 51.61290322580645
}
```
如果请求的后面加上索引的名称，则可以得到这个索引的健康检查情况。例如：
```shell
GET http://127.0.0.1:9200/_cluster/health/secisland?pretty=true
```
返回值中，status字段提供的值反应了集群整体的健康程度，它的状态是由系统中最差的分片决定的。值的意义如下：
- green——所有的主分片（Primary Shard）和副本分片（Replica Shard）都处于活动状态。
- yellow——所有的主分片都处于活动状态，但是并不是所有的副本分片都处于活动状态。
- red——不是所有的主分片都处于活动状态。

## 集群状态
整个集群的综合状态信息是可以通过_cluster/state参数查询的。
```shell
GET http://127.0.0.1:9200/_cluster/state
```

```json
{
  "cluster_name": "elasticsearch",
  "version": 24,
  "state_uuid": "qgy8HW89RqmQPjhXAgyWjQ",
  "master_node": "100K_m-cSjecB6NbKJx9ew",
  "blocks": {},
  "nodes": {
    "100K_m-cSjecB6NbKJx9ew": {
      "name": "secilog",
      "transport_address": "127.0.0.1:9300",
      "attributes": {}
    }
  },
  "metadata": {...},    //索引结构相关的信息
  "routing_table": {...}, //索引分片相关的信息
  "routing_nodes": {...} //路由节点相关的信息
}
```
默认情况下，集群状态请求的是主节点的状态。在state后面增加metadata只请求索引结构相关的信息，如果后面加上具体的索引，则只请求这个具体索引结构相关的信息。
例如：其他的几个例子：
```shell
GET http://127.0.0.1:9200/_cluster/state/metadata，routing_table/foo,bar
GET http://127.0.0.1:9200/_cluster/state/_all/foo,bar
GET http://127.0.0.1:9200/_cluster/state/blocks
```

## 集群统计
集群统计API_cluster/stats可以从一个集群的角度来统计集群状态。它返回两个最基本的信息，一个是索引的信息，比如分片的数量、存储的大小、内存的使用等；另一个是集群节点的信息，比如节点数量、角色、操作系统信息、JVM版本、内存使用率、CPU和插件的安装信息。

























