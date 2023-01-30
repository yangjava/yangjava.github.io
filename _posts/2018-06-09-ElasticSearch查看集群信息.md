---
layout: post
categories: ElasticSearch
description: none
keywords: ElasticSearch
---
# ElasticSearch查看集群信息

## 查看集群状态使用频率最高的方法
http://127.0.0.1:9200/
```java
{
  "name" : "qjd.prd.skywalking39",
  "cluster_name" : "qjd-sw-online",
  "cluster_uuid" : "SOk2MBBcQ86RF0Sy7d16dg",
  "version" : {
    "number" : "7.9.3",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "c4138e51121ef06a6404866cddc601906fe5c868",
    "build_date" : "2020-10-16T10:36:16.141335Z",
    "build_snapshot" : false,
    "lucene_version" : "8.6.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```
一般我们通过这个方式来验证ES服务器是否启动成功。

## _cat/health 查看集群健康状态
curl http://127.0.0.1:9200/_cat/health?v
```text
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1673682409 07:46:49  qjd-sw-online yellow          1         1    593 593    0    0      373             0                  -                 61.4%
```

参数说明：
- cluster：集群名称
- status：集群状态 green 表示集群一切正常；yellow 表示集群不可靠但可用(单节点状态)；red 集群不可用，有故障。
- node.total：节点总数量
- node.data：数据节点的数量
- shards：存活的分片数量
- pri：主分片数量
- relo：迁移中的分片数量
- init：初始化中的分片数量
- unassign：未分配的分片
- pending_tasks：准备中的任务
- max_task_wait_time：任务最长等待时间
- active_shards_percent：激活的分片百分比

## _cat/shards 查看分片信息

查看所有索引的分片信息
```text
curl http://127.0.0.1:9200/_cat/shards?v
```

查看指定索引的分片信息

```text
curl http://127.0.0.1:9200/_cat/shards/opt_log?v
```
参数说明：

- index：索引名称
- shard：分片数
- prirep：分片类型，p为主分片，r为复制分片
- state：分片状态，STARTED为正常
- docs：记录数
- store：存储大小
- ip：节点ip
- node：节点名称

## _cat/nodes 查看集群的节点信息

```text
curl http://127.0.0.1:9200/_cat/nodes?v
```
参数说明：

- ip：节点ip
- heap.percent：堆内存使用百分比
- ram.percent： 运行内存使用百分比
- cpu：cpu使用百分比
- master：带* 表明该节点是主节点，带-表明该节点是从节点
- name：节点名称

## _cat/indices 查看索引信息
查看所有索引的分片信息
```text
curl http://127.0.0.1:9200/_cat/indices?v
```
参数说明：

- index： 索引名称
- docs.count：文档总数
- docs.deleted：已删除文档数
- store.size： 存储的总容量
- pri.store.size：主分片的存储总容量


## 集群命令汇总

/_cat/allocation
/_cat/shards
/_cat/shards/{index}
/_cat/master
/_cat/nodes
/_cat/tasks
/_cat/indices
/_cat/indices/{index}
/_cat/segments
/_cat/segments/{index}
/_cat/count
/_cat/count/{index}
/_cat/recovery
/_cat/recovery/{index}
/_cat/health
/_cat/pending_tasks
/_cat/aliases
/_cat/aliases/{alias}
/_cat/thread_pool
/_cat/thread_pool/{thread_pools}
/_cat/plugins
/_cat/fielddata
/_cat/fielddata/{fields}
/_cat/nodeattrs
/_cat/repositories
/_cat/snapshots/{repository}
/_cat/templates

