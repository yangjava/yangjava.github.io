---
layout: post
categories: ElasticSearch
description: none
keywords: ElasticSearch
---
# ElasticSearch集群管理
Elasticsearch正是利用集群进行水平扩展节点，达到支持海量数据的能力。

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

























