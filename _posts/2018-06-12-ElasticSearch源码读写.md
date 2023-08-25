---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码读写

## 写流程
在ES中，写入单个文档的请求称为Index请求，批量写入的请求称为Bulk请求。写单个和多个文档使用相同的处理逻辑，请求被统一封装为BulkRequest。

## 文档操作的定义
在ES中，对文档的操作有下面几种类型：
```
enum OpType {
INDEX（0），
CREATE（1），
UPDATE（2），
DELETE（3）；
}
```

- INDEX：向索引中“put”一个文档的操作称为“索引”一个文档。此处“索引”为动词。
- CREATE:put 请求可以通过 op_type 参数设置操作类型为 create，在这种操作下，如果文档已存在，则请求将失败。
- UPDATE：默认情况下，“put”一个文档时，如果文档已存在，则更新它。
- DELETE：删除文档。

在put API中，通过op_type参数来指定操作类型。

## 可选参数
Index API和Bulk API有一些可选参数，这些参数在请求的URI中指定，例如：
```
PUT my-index/my-type/my-id?pipeline=my_pipeline_id
{
＂foo＂: ＂bar＂
}
```

## Index/Bulk基本流程
新建、索引（这里的索引是动词，指写入操作，将文档添加到Lucene的过程称为索引一个文档）和删除请求都是写操作。写操作必须先在主分片执行成功后才能复制到相关的副分片。

写单个文档的流程
- 客户端向NODE1发送写请求。
- NODE1使用文档ID来确定文档属于分片0，通过集群状态中的内容路由表信息获知分片0的主分片位于NODE3，因此请求被转发到NODE3上。
- NODE3上的主分片执行写操作。如果写入成功，则它将请求并行转发到 NODE1和NODE2的副分片上，等待返回结果。当所有的副分片都报告成功，NODE3将向协调节点报告成功，协调节点再向客户端报告成功。

在客户端收到成功响应时，意味着写操作已经在主分片和所有副分片都执行完成。 写一致性的默认策略是quorum，即多数的分片（其中分片副本可以是主分片或副分片）在写入操作时处于可用状态。
```
quorum = int（ （primary + number_of_replicas） / 2 ） + 1
```

## Index/Bulk详细流程

### 协调节点流程











