---
layout: post
categories: ElasticSearch
description: none
keywords: ElasticSearch
---
# ElasticSearch优化

## 写入速度优化
在 ES 的默认设置下，是综合考虑数据可靠性、搜索实时性、写入速度等因素的。当离开默认设置、追求极致的写入速度时，很多是以牺牲可靠性和搜索实时性为代价的。有时候，业务上对数据可靠性和搜索实时性要求并不高，反而对写入速度要求很高，此时可以调整一些策略，最大化写入速度。

接下来的优化基于集群正常运行的前提下，如果是集群首次批量导入数据，则可以将副本数设置为0，导入完毕再将副本数调整回去，这样副分片只需要复制，节省了构建索引过程。

综合来说，提升写入速度从以下几方面入手：
- 加大translog flush间隔，目的是降低iops、writeblock。
- 加大index refresh间隔，除了降低I/O，更重要的是降低了segment merge频率。
- 调整bulk请求。
- 优化磁盘间的任务均匀情况，将shard尽量均匀分布到物理主机的各个磁盘。
- 优化节点间的任务分布，将任务尽量均匀地发到各节点。
- 优化Lucene层建立索引的过程，目的是降低CPU占用率及I/O，例如，禁用_all字段。

### translog flush间隔调整
从ES 2.x开始，在默认设置下，translog的持久化策略为：每个请求都“flush”。对应配置项如下：
```
index.translog.durability: request
```
这是影响 ES 写入速度的最大因素。但是只有这样，写操作才有可能是可靠的。如果系统可以接受一定概率的数据丢失（例如，数据写入主分片成功，尚未复制到副分片时，主机断电。由于数据既没有刷到Lucene,translog也没有刷盘，恢复时translog中没有这个数据，数据丢失），则调整translog持久化策略为周期性和一定大小的时候“flush”，例如：
```properties
index.translog.durability: async
# 设置为async表示translog的刷盘策略按sync_interval配置指定的时间周期进行。
index.translog.sync_interval: 120s
# 加大translog刷盘间隔时间。默认为5s，不可低于100ms。
index.translog.flush_threshold_size: 1024mb
# 超过这个大小会导致refresh操作，产生新的Lucene分段。默认值为512MB。
```

### 索引刷新间隔refresh_interval
默认情况下索引的refresh_interval为1秒，这意味着数据写1秒后就可以被搜索到，每次索引的refresh会产生一个新的Lucene段，这会导致频繁的segment merge行为，如果不需要这么高的搜索实时性，应该降低索引refresh周期，例如：
```properties
index.refresh_interval: 120s
```

### 段合并优化
segment merge操作对系统I/O和内存占用都比较高，从ES 2.0开始，merge行为不再由ES控制，而是由Lucene控制，因此以下配置已被删除：
```properties
indices.store.throttle.type
indices.store.throttle.max_bytes_per_sec
index.store.throttle.type
index.store.throttle.max_bytes_per_sec
```
改为以下调整开关：
```properties
index.merge.scheduler.max_thread_count

index.merge.policy.*
```
最大线程数max_thread_count的默认值如下：
```
Math.max（1, Math.min（4, Runtime.getRuntime（）.availableProcessors（） / 2））
```
以上是一个比较理想的值，如果只有一块硬盘并且非 SSD，则应该把它设置为1，因为在旋转存储介质上并发写，由于寻址的原因，只会降低写入速度。

merge策略index.merge.policy有三种：
- tiered（默认策略）；
- log_byete_size；
- log_doc。

每个策略的具体描述可以参考 Mastering Elasticsearch:https://doc.yonyoucloud.com/doc/mastering-elasticsearch/chapter-3/36_README.html。目前我们使用默认策略，但是对策略的参数进行了一些调整。

索引创建时合并策略就已确定，不能更改，但是可以动态更新策略参数，可以不做此项调整。如果堆栈经常有很多merge，则可以尝试调整以下策略配置：
```
index.merge.policy.segments_per_tier
```
该属性指定了每层分段的数量，取值越小则最终segment越少，因此需要merge的操作更多，可以考虑适当增加此值。默认为10，其应该大于等于index.merge.policy.max_merge_at_once。
```
index.merge.policy.max_merged_segment
```
指定了单个segment的最大容量，默认为5GB，可以考虑适当降低此值。

### indexing buffer
indexing buffer在为doc建立索引时使用，当缓冲满时会刷入磁盘，生成一个新的segment，这是除refresh_interval刷新索引外，另一个生成新segment的机会。每个shard有自己的indexing buffer，下面的这个buffer大小的配置需要除以这个节点上所有shard的数量：
```properties
indices.memory.index_buffer_size
# 默认为整个堆空间的10%。
indices.memory.min_index_buffer_size
# 默认为48MB。
indices.memory.max_index_buffer_size
# 默认为无限制。
```
在执行大量的索引操作时，indices.memory.index_buffer_size的默认设置可能不够，这和可用堆内存、单节点上的shard数量相关，可以考虑适当增大该值。

### 使用bulk请求
批量写比一个索引请求只写单个文档的效率高得多，但是要注意bulk请求的整体字节数不要太大，太大的请求可能会给集群带来内存压力，因此每个请求最好避免超过几十兆字节，即使较大的请求看上去执行得更好。

### bulk线程池和队列
建立索引的过程属于计算密集型任务，应该使用固定大小的线程池配置，来不及处理的任务放入队列。线程池最大线程数量应配置为CPU核心数+1，这也是bulk线程池的默认设置，可以避免过多的上下文切换。队列大小可以适当增加，但一定要严格控制大小，过大的队列导致较高的GC压力，并可能导致FGC频繁发生。

### 并发执行bulk请求
bulk写请求是个长任务，为了给系统增加足够的写入压力，写入过程应该多个客户端、多线程地并行执行，如果要验证系统的极限写入能力，那么目标就是把CPU压满。磁盘util、内存等一般都不是瓶颈。如果 CPU 没有压满，则应该提高写入端的并发数量。但是要注意 bulk线程池队列的reject情况，出现reject代表ES的bulk队列已满，客户端请求被拒绝，此时客户端会收到429错误（TOO_MANY_REQUESTS），客户端对此的处理策略应该是延迟重试。不可忽略这个异常，否则写入系统的数据会少于预期。即使客户端正确处理了429错误，我们仍然应该尽量避免产生reject。因此，在评估极限的写入能力时，客户端的极限写入并发量应该控制在不产生reject前提下的最大值为宜。

### 磁盘间的任务均衡
如果部署方案是为path.data配置多个路径来使用多块磁盘，则ES在分配shard时，落到各磁盘上的 shard 可能并不均匀，这种不均匀可能会导致某些磁盘繁忙，利用率在较长时间内持续达到100%。这种不均匀达到一定程度会对写入性能产生负面影响。

ES在处理多路径时，优先将shard分配到可用空间百分比最多的磁盘上，因此短时间内创建的shard可能被集中分配到这个磁盘上，即使可用空间是99%和98%的差别。后来ES在2.x版本中开始解决这个问题：预估一下 shard 会使用的空间，从磁盘可用空间中减去这部分，直到现在6.x版也是这种处理方式。但是实现也存在一些问题：

从可用空间减去预估大小

这种机制只存在于一次索引创建的过程中，下一次的索引创建，磁盘可用空间并不是上次做完减法以后的结果。这也可以理解，毕竟预估是不准的，一直减下去空间很快就减没了。

但是最终的效果是，这种机制并没有从根本上解决问题，即使没有完美的解决方案，这种机制的效果也不够好。

如果单一的机制不能解决所有的场景，那么至少应该为不同场景准备多种选择。

为此，我们为ES增加了两种策略。
- 简单轮询：在系统初始阶段，简单轮询的效果是最均匀的。
- 基于可用空间的动态加权轮询：以可用空间作为权重，在磁盘之间加权轮询。

### 节点间的任务均衡
为了节点间的任务尽量均衡，数据写入客户端应该把bulk请求轮询发送到各个节点。

当使用Java API或REST API的bulk接口发送数据时，客户端将会轮询发送到集群节点，节点列表取决于：
- 使用Java API时，当设置client.transport.sniff为true（默认为false）时，列表为所有数据节点，否则节点列表为构建客户端对象时传入的节点列表。
- 使用REST API时，列表为构建对象时添加进去的节点。

Java API的TransportClient和REST API的RestClient都是线程安全的，如果写入程序自己创建线程池控制并发，则应该使用同一个Client对象。在此建议使用REST API,Java API会在未来的版本中废弃，REST API有良好的版本兼容性好。理论上，Java API在序列化上有性能优势，但是只有在吞吐量非常大时才值得考虑序列化的开销带来的影响，通常搜索并不是高吞吐量的业务。

要观察bulk请求在不同节点间的均衡性，可以通过cat接口观察bulk线程池和队列情况：

_cat/thread_pool

### 索引过程调整和优化

### 自动生成doc ID
通过ES写入流程可以看出，写入doc时如果外部指定了id，则ES会先尝试读取原来doc的版本号，以判断是否需要更新。这会涉及一次读取磁盘的操作，通过自动生成doc ID可以避免这个环节。

### 调整字段Mappings
- 减少字段数量，对于不需要建立索引的字段，不写入ES。
- 将不需要建立索引的字段index属性设置为not_analyzed或no。对字段不分词，或者不索引，可以减少很多运算操作，降低CPU占用。尤其是binary类型，默认情况下占用CPU非常高，而这种类型进行分词通常没有什么意义。
- 减少字段内容长度，如果原始数据的大段内容无须全部建立索引，则可以尽量减少不必要的内容。
- 使用不同的分析器（analyzer），不同的分析器在索引过程中运算复杂度也有较大的差异。

### 调整_source字段
_source 字段用于存储 doc 原始数据，对于部分不需要存储的字段，可以通过 includes excludes过滤，或者将_source禁用，一般用于索引和数据分离。

这样可以降低 I/O 的压力，不过实际场景中大多不会禁用_source，而即使过滤掉某些字段，对于写入速度的提升作用也不大，满负荷写入情况下，基本是 CPU 先跑满了，瓶颈在于CPU。


### 禁用_all字段
从ES 6.0开始，_all字段默认为不启用，而在此前的版本中，_all字段默认是开启的。_all字段中包含所有字段分词后的关键词，作用是可以在搜索的时候不指定特定字段，从所有字段中检索。ES 6.0默认禁用_all字段主要有以下几点原因：
- 由于需要从其他的全部字段复制所有字段值，导致_all字段占用非常大的空间。
- _all 字段有自己的分析器，在进行某些查询时（例如，同义词），结果不符合预期，因为没有匹配同一个分析器。
- 由于数据重复引起的额外建立索引的开销。
- 想要调试时，其内容不容易检查。
- 有些用户甚至不知道存在这个字段，导致了查询混乱。
- 有更好的替代方法。
关于这个问题的更多讨论可以参考https://github.com/elastic/elasticsearch/issues/19784。

在ES 6.0之前的版本中，可以在mapping中将enabled设置为false来禁用_all字段：禁用_all字段可以明显降低对CPU和I/O的压力。

### 对Analyzed的字段禁用Norms
Norms用于在搜索时计算doc的评分，如果不需要评分，则可以将其禁用：
```
＂title＂: {＂type＂: ＂string＂,＂norms＂: {＂enabled＂: false}}
```

### index_options 设置
index_options用于控制在建立倒排索引过程中，哪些内容会被添加到倒排索引，例如，doc数量、词频、positions、offsets等信息，优化这些设置可以一定程度降低索引过程中的运算任务，节省CPU占用率。

不过在实际场景中，通常很难确定业务将来会不会用到这些信息，除非一开始方案就明确是这样设计的。

### 参考配置
下面是笔者的线上环境使用的全局模板和配置文件的部分内容，省略掉了节点名称、节点列表等基础配置字段，仅列出与本文相关内容。

从ES 5.x开始，索引级设置需要写在模板中，或者在创建索引时指定，我们把各个索引通用的配置写到了模板中，这个模板匹配全部的索引，并且具有最低的优先级，让用户定义的模板有更高的优先级，以覆盖这个模板中的配置。
```
{
    "template": "*",
    "order" : 0,
    "settings": {
        "index.merge.policy.max_merged_segment" : "2gb",
        "index.merge.policy.segments per_tier" : "24",
        "index.number_of_replicas" : "1",
        "index.number_of_shards" : "24",
        "index.optimize_auto_generated_id" : "true",
        "index.refresh_interval" : "120s",
        "index.translog.durability" : "async",
        "index.translog.flush_threshold_size" : "1000mb",
        "index.translog. sync_ interval" : "120s",
        "index.unassigned.node_left.delayed_timeout" : "5d"
    }
}
```
elasticsearch.yml中的配置：
```
indices.memory.index_buffer_size: 30%
```

## 思考与总结
- 方法比结论重要。一个系统性问题往往是多种因素造成的，在处理集群的写入性能问题上，先将问题分解，在单台上进行压测，观察哪种系统资源达到极限，例如，CPU或磁盘利用率、I/O block、线程切换、堆栈状态等。然后分析并调整参数，优化单台上的能力，先解决局部问题，在此基础上解决整体问题会容易得多。
- 可以使用更好的 CPU，或者使用 SSD，对写入性能提升明显。在我们的测试中，在相同条件下，E5 2650V4比E5 2430 v2的写入速度高60%左右。
- 在我们的压测环境中，写入速度稳定在平均单机每秒3万条以上，使用的测试数据：每个文档的字段数量为10个左右，文档大小约100字节，CPU使用E5 2430 v2。

## Elasticsearch 源码解析与优化实战: 第19章 搜索速度的优化

### 为文件系统cache预留足够的内存
在一般情况下，应用程序的读写都会被操作系统 “cache” (除了direct 方式)，cache 保存在系统物理内存中(线上应该禁用swap)，命中cache可以降低对磁盘的直接访问频率。搜索很依赖对系统cache的命中，如果某个请求需要从磁盘读取数据，则一定会产生相对较高的延迟。应该至少为系统cache预留一半的可用物理内存，更大的内存有更高的cache命中率。

### 使用更快的硬件
写入性能对CPU的性能更敏感，而搜索性能在一般情况下更多的是在于I/O能力，使用SSD会比旋转类存储介质好得多。 尽量避免使用NFS等远程文件系统，如果NFS比本地存储慢3倍，则在搜索场景下响应速度可能会慢10倍左右。这可能是因为搜索请求有更多的随机访问。

如果搜索类型属于计算比较多，则可以考虑使用更快的CPU。

### 文档模型
为了让搜索时的成本更低，文档应该合理建模。特别是应该避免join操作，嵌套(nested)会使查询慢几倍，父子(parent-child)关系可能使查询慢数百倍，因此，如果可以通过非规范化(denormalizing) 文档来回答相同的问题，则可以显著地提高搜索速度。

### 预索引数据
还可以针对某些查询的模式来优化数据的索引方式。例如，如果所有文档都有一个price字段，并且大多数查询在一个固定的范围上运行range聚合，那么可以通过将范围“pre-indexing”到索引中并使用terms聚合来加快聚合速度。

例如，文档起初是这样的:
```
PUT index/type/1
{
    "designation": " spoon",
    "price": 13
}

采用如下的搜索方式:
GET index/_search
{
    "aggs": {
        "price_ranges": {
            "range": {
                "field": "price",
                "ranges" : {
                    { "to": 10 },
                    { "from": 10, "to": 100 },
                    { "from": 100 }
                }
            }
        }
    }
}
```
那么我们的优化是，在建立索引时对文档进行富化，增加price_range 字段，mapping 为 keyword类型:
```
PUT index
{
    "mappings": {
        "type": {
            "properties": {
                "price_range": {
                    "type": "keyword"
                }
            }
        }
    }
}

PUT index/type/1
 {
    "designation": "spoon",
    "price": 13,
    "price_range": "10-100"
}
```

### 字段映射
有些字段的内容是数值，但并不意味着其总是应该被映射为数值类型。 例如，一些标识符，将它们映射为keyword可能会比integer或long更好。

### 避免使用脚本
一般来说，应该避免使用脚本。如果一定要用，则应该优先考虑painless和expressions。

### 优化日期搜索
在使用日期范围检索时，使用now的查询通常不能缓存，因为匹配到的范围一直在变化。但是，从用户体验的角度来看，切换到一个完整的日期通常是可以接受的，这样可以更好地利用查询缓存。

### 为只读索引执行force-merge
为不再更新的只读索引执行force merge，将Lucene索引合并为单个分段，可以提升查询速度。当一个Lucene索引存在多个分段时，每个分段会单独执行搜索再将结果合并，将只读索引强制合并为一个Lucene分段不仅可以优化搜索过程，对索引恢复速度也有好处。

基于日期进行轮询的索引的旧数据一般都不会再更新。此前的章节中说过，应该避免持续地写一个固定的索引，直到它巨大无比，而应该按一定的策略，例如，每天生成一个新的索引，然后用别名关联，或者使用索引通配符。这样，可以每天选一个时间点对昨天的索引执行force-merge、Shrink等操作。

### 预热全局序号 ( global ordinals )
全局序号是一种数据结构，用于在keyword字段上运行terms聚合。它用一个数值来代表；字段中的字符串值，然后为每一数值分配一个 bucket。这需要一个对 global ordinals 和bucket的构建过程。默认情况下，它们被延迟构建，因为ES不知道哪些字段将用于terms聚合，哪些字段不会。可以通过配置映射在刷新(refresh) 时告诉ES预先加载全局序数：

### execution hint
terms聚合有两种不同的机制：
- 通过直接使用字段值来聚合每个桶的数据(map)。
- 通过使用字段的全局序号并为每个全局序号分配一个bucket (global_ordinals)。
ES使用global_ordinals作为keyword 字段的默认选项，它使用全局序号动态地分配bucket，因此内存使用与聚合结果中的字段数量是线性关系。在大部分情况下，这种方式的速度很快。

当查询只会匹配少量文档时，可以考虑使用map。默认情况下，map只在脚本上运行聚合时使用，因为它们没有序数。

### 预热文件系统cache
如果ES主机重启，则文件系统缓存将为空，此时搜索会比较慢。可以使用index.store.preload设置，通过指定文件扩展名，显式地告诉操作系统应该将哪些文件加载到内存中。

### 转换查询表达式
在组合查询中可以通过bool过滤器进行and、or 和not的多个逻辑组合检索，这种组合查询中的表达式在下面的情况下可以做等价转换：(A I B) & (C | D) ==> (A & C) | (A & D) | (B & C) | (B & D )

### 调节搜索请求中的 batched_reduce_size
该字段是搜索请求中的一个参数。默认情况下，聚合操作在协调节点需要等所有的分片都取回结果后才执行，使用batched_reduce_size参数可以不等待全部分片返回结果，而是在指定数量的分片返回结果之后就可以先处理一部分(reduce)。 这样可以避免协调节点在等待全部结果的过程中占用大量内存，避免极端情况下可能导致的OOM。该字段的默认值为512，从 ES 5.4 开始支持。

### 使用近似聚合
近似聚合以牺牲少量的精确度为代价，大幅提高了执行效率，降低了内存使用。近似聚合的使用方式可以参考官方手册：

Percentiles Aggregation ( https://www.elastic.co/guide/en/elasticsearch/ reference/current/search-aggregations-metrics percentile-aggregation.html)
Cardinality Aggregation ( https://www.elastic.co/guide/en/elasticsearch/reference/current/search- aggregations-metrics-cardinality-aggregation.html)

### 深度优先还是广度优先
ES有两种不同的聚合方式：深度优先和广度优先。深度优先是默认设置，先构建完整的树，然后修剪无用节点。大多数情况下深度聚合都能正常工作，但是有些特殊的场景更适合广度优先，先执行第一层聚合，再继续下一层聚合之前会先做修剪，官方有一个例子可以参考：https://www.elastic.co/guide/cn/elasticsearch/guide/current/_preventing_combinatorial_explosions.html

### 限制搜索请求的分片数
一个搜索请求涉及的分片数量越多，协调节点的CPU和内存压力就越大。默认情况下，ES会拒绝超过1000个分片的搜索请求。 我们应该更好地组织数据，让搜索请求的分片数更少。如果想调节这个值，则可以通过action.search.shard count 配置项进行修改。

虽然限制搜索的分片数并不能直接提升单个搜索请求的速度，但协调节点的压力会间接影响搜索速度。 例如，占用更多内存会产生更多的GC压力，可能导致更多的stop-the-world时间等，因此间接影响了协调节点的性能，所以我们仍把它列作本章的一部分。

### 利用自适应副本选择( ARS)提升ES响应速度
为了充分利用计算资源和负载均衡，协调节点将搜索请求轮询转发到分片的每个副本，轮询策略是负载均衡过程中最简单的策略，任何一个负载均衡器都具备这种基础的策略，缺点是不考虑后端实际系统压力和健康水平。