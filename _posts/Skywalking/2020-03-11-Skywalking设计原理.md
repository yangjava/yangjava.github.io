---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---

# SkyWalking源码

## 设计

如何要实现一个完整的分布式追踪系统。有以下几个问题需要我们仔细思考一下

- 怎么自动采集 span 数据：自动采集，对业务代码无侵入
- 如何跨进程传递 context
- traceId 如何保证全局唯一
- 请求量这么多采集会不会影响性能

接下我来看看 SkyWalking 是如何解决以上四个问题的

##### **怎么自动采集 span 数据**

SkyWalking 采用了**插件化 + javaagent** 的形式来实现了 span 数据的自动采集，这样可以做到对代码的无侵入性，插件化意味着可插拔，扩展性好(后文会介绍如何定义自己的插件)

##### 如何跨进程传递 context

我们知道数据一般分为 header 和 body, 就像 http 有 header 和 body, RocketMQ 也有 MessageHeader，Message Body, body 一般放着业务数据，所以不宜在 body 中传递 context，应该在 header 中传递 context

##### traceId 如何保证全局唯一

要保证全局唯一 ，我们可以采用分布式或者本地生成的 ID，使用分布式话需要有一个发号器，每次请求都要先请求一下发号器，会有一次网络调用的开销，所以 SkyWalking 最终采用了本地生成 ID 的方式，它采用了大名鼎鼎的 snowflow 算法，性能很高。

##### **请求量这么多，全部采集会不会影响性能?**

如果对每个请求调用都采集，那毫无疑问数据量会非常大，但反过来想一下，是否真的有必要对每个请求都采集呢，其实没有必要，我们可以设置采样频率，只采样部分数据，SkyWalking 默认设置了 3 秒采样 3 次，其余请求不采样