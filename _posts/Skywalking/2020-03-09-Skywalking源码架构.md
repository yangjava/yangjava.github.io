---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
## 源码

SkyWalking 源码的整体结构如下图所示：

1、apm-application-toolkit 模块：SkyWalking 提供给用户调用的工具箱。

该模块提供了对 log4j、log4j2、logback 等常见日志框架的接入接口，提供了 @Trace 注解等。
apm-application-toolkit 模块类似于暴露 API 定义，对应的处理逻辑在 apm-sniffer/apm-toolkit-activation 模块中实现。

2、apm-commons 模块：SkyWalking 的公共组件和工具类。

其中包含两个子模块，apm-datacarrier 模块提供了一个生产者-消费者模式的缓存组件（DataCarrier），无论是在 Agent 端还是 OAP 端都依赖该组件。
apm-util 模块则提供了一些常用的工具类，例如，字符串处理工具类（StringUtil）、占位符处理的工具类（PropertyPlaceholderHelper、PlaceholderConfigurerSupport）等等。


3、apm-protocol 模块：该模块中只有一个 apm-network 模块，我们需要关注的是其中定义的 .proto 文件，定义 Agent 与后端 OAP 使用 gRPC 交互时的协议。

4、apm-sniffer 模块：apm-sniffer 模块中有 4 个子模块：

apm-agent 模块：其中包含了刚才使用的 SkyWalkingAgent 这个类，是整个 Agent 的入口。

apm-agent-core 模块：SkyWalking Agent 的核心实现都在该模块中。

apm-sdk-plugin 模块：SkyWalking Agent 使用了微内核+插件的架构，该模块下包含了 SkyWalking Agent 的全部插件，

apm-toolkit-activation 模块：apm-application-toolkit 模块的具体实现。

5、apm-webapp 模块：SkyWalking Rocketbot 对应的后端。

6、oap-server 模块：SkyWalking OAP 的全部实现都在 oap-server 模块，其中包含了多个子模块，
exporter 模块：负责导出数据。

server-alarm-plugin 模块：负责实现 SkyWalking 的告警功能。

server-cluster-pulgin 模块：负责 OAP 的集群信息管理，其中提供了接入多种第三方组件的相关插件，

server-configuration 模块：负责管理 OAP 的配置信息，也提供了接入多种配置管理组件的相关插件，

server-core模块：SkyWalking OAP 的核心实现都在该模块中。

server-library 模块：OAP 以及 OAP 各个插件依赖的公共模块，其中提供了双队列 Buffer、请求远端的 Client 等工具类，这些模块都是对立于 SkyWalking OAP 体系之外的类库，我们可以直接拿走使用。

server-query-plugin 模块：SkyWalking Rocketbot 发送的请求首先由该模块接收处理，目前该模块只支持 GraphQL 查询。

server-receiver-plugin 模块：SkyWalking Agent 发送来的 Metrics、Trace 以及 Register 等写入请求都是首先由该模块接收处理的，不仅如此，该模块还提供了多种接收其他格式写入请求的插件，

server-starter 模块：OAP 服务启动的入口。

server-storage-plugin 模块：OAP 服务底层可以使用多种存储来保存 Metrics 数据以及Trace 数据，该模块中包含了接入相关存储的插件，

7、skywalking-ui 目录：SkyWalking Rocketbot 的前端。

# 流程

Skywalking主要由Agent、OAP、Storage、UI四大模块组成（如下图）：Agent和业务程序运行在一起，采集链路及其它数据，通过gRPC发送给OAP（部分Agent采用http+json的方式）；OAP还原链路（图中的Tracing），并分析产生一些指标（图中的Metric），最终存储到Storage中。




## 架构

SkyWalking 的基础如下架构，可以说几乎所有的的分布式调用都是由以下几个组件组成的

- **Agent** ：负责从应用中，收集链路信息，发送给 SkyWalking OAP 服务器。目前支持 SkyWalking、Zikpin、Jaeger 等提供的 Tracing 数据信息。而我们目前采用的是，SkyWalking Agent 收集 SkyWalking Tracing 数据，传递给服务器。
- **SkyWalking OAP** ：负责接收 Agent 发送的 Tracing 数据信息，然后进行分析(Analysis Core) ，存储到外部存储器( Storage )，最终提供查询( Query )功能。
- **Storage** ：Tracing 数据存储。目前支持 ES、MySQL、Sharding Sphere、TiDB、H2 多种存储器。
- **SkyWalking UI** ：负责提供控台，查看链路等等。

## 官方文档

在 https://github.com/apache/skywalking/tree/master/docs 地址下，提供了 SkyWalking 的**英文**文档。

推荐先阅读 https://github.com/SkyAPM/document-cn-translation-of-skywalking 地址，提供了 SkyWalking 的**中文**文档。

使用 SkyWalking 的目的，是实现**分布式链路追踪**的功能，所以最好去了解下相关的知识。这里推荐阅读两篇文章：

- [《OpenTracing 官方标准 —— 中文版》](https://github.com/opentracing-contrib/opentracing-specification-zh)
- Google 论文 [《Dapper，大规模分布式系统的跟踪系统》](http://www.iocoder.cn/Fight/Dapper-translation/?self)