---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
# Skywalking简介
**官网介绍**：分布式系统的应用程序性能监控工具，专为微服务、云原生和基于容器的 (Kubernetes) 架构而设计。

```
SkyWalking: an APM(application performance monitor) system, especially designed for microservices, cloud native and container-based architectures.
```


核心功能如下：

- 服务、服务实例、端点指标分析
- 根本原因分析。在运行时分析代码
- 服务拓扑图分析
- 服务、服务实例和端点依赖分析
- 缓慢的服务和端点检测
- 性能优化
- 分布式跟踪和上下文传播
- 数据库访问指标。检测慢速数据库访问语句（包括SQL语句）
- 消息队列性能和消耗延迟监控
- 警报
- 浏览器性能监控
- 基础设施（VM、网络、磁盘等）监控
- 跨指标、跟踪和日志的协作


SkyWalking 支持从多个来源和多种格式收集遥测（指标、跟踪和日志）数据，包括

1. Java、.NET Core、NodeJS、PHP 和 Python 自动仪器代理。
2. Go、C++ 和 Rust SDK。
3. LUA 代理，特别适用于 Nginx、OpenResty 和 Apache APISIX。
4. 浏览器代理。
5. 服务网格可观察性。控制平面和数据平面。
6. 度量系统，包括 Prometheus、OpenTelemetry、Spring Sleuth(Micrometer)、Zabbix。
7. 日志。
8. Zipkin v1/v2 跟踪。（无分析）

SkyWalking OAP 使用 STAM（流拓扑分析方法）来分析基于跟踪的代理场景中的拓扑，以获得更好的性能。阅读[STAM 的论文以](https://wu-sheng.github.io/STAM/)获取更多详细信息。

Skywalking是由国内开源爱好者吴晟（原OneAPM工程师，目前在华为）开源并提交到Apache孵化器的产品，它同时吸收了Zipkin/Pinpoint/CAT的设计思路，支持非侵入式埋点。是一款基于分布式跟踪的应用程序性能监控系统。另外社区还发展出了一个叫OpenTracing的组织，旨在推进调用链监控的一些规范和标准工作。


## 概念与架构

SkyWalking 是一个开源可观测平台，用于收集、分析、聚合和可视化来自服务和云原生基础设施的数据。SkyWalking 提供了一种简单的方法来保持分布式系统的清晰视图，甚至跨云。它是一种现代 APM，专为云原生、基于容器的分布式系统而设计。

### 为什么要使用 SkyWalking？

SkyWalking 为在许多不同场景中观察和监控分布式系统提供解决方案。首先，与传统方法一样，SkyWalking 为 Java、C#、Node.js、Go、PHP 和 Nginx LUA 等服务提供自动仪器代理。（呼吁 Python 和 C++ SDK 贡献）。在多语言、持续部署的环境中，云原生基础架构变得更加强大，但也更加复杂。SkyWalking 的服务网格接收器允许 SkyWalking 接收来自 Istio/Envoy 和 Linkerd 等服务网格框架的遥测数据，让用户了解整个分布式系统。

SkyWalking 为**服务**（Service）、**服务实例**（Service Instance）、**端点**（Endpoint ）提供可观察性能力。如今，Service、Instance 和 Endpoint 等术语随处可见，因此值得在 SkyWalking 的上下文中定义它们的具体含义：

- **服务**。表示为传入请求提供相同行为的一组/一组工作负载。您可以在使用仪器代理或 SDK 时定义服务名称。SkyWalking 还可以使用您在 Istio 等平台中定义的名称。
- **服务实例**。服务组中的每个单独的工作负载都称为一个实例。就像`pods`在 Kubernetes 中一样，它不需要是单个操作系统进程，但是，如果您使用仪器代理，则实例实际上是一个真正的操作系统进程。
- **端点**。用于传入请求的服务中的路径，例如 HTTP URI 路径或 gRPC 服务类 + 方法签名。

SkyWalking 允许用户了解Services 和Endpoints 的拓扑关系，查看每个Service/Service Instance/Endpoint 的指标，并设置报警规则。

此外，您还可以集成

1. 其他使用带有 Zipkin、Jaeger 和 OpenCensus 的 SkyWalking 本地代理和 SDK 的分布式跟踪。
2. 其他度量系统，如 Prometheus、Sleuth(Micrometer)、OpenTelemetry。

### 架构

SkyWalking 逻辑上分为四部分: Probes（探针）、Platform backend（平台后端）、Storage（存储） 和 用户界面（UI） 。

![SkyWalking_Architecture](https://skywalking.apache.org/images/SkyWalking_Architecture_20210424.png?t=20210424)

- **探针（Probes）**基于不同的来源可能是不一样的, 但作用都是收集数据, 将数据格式化为 SkyWalking 适用的格式。 
- **平台后端（Platform backend）**支持数据聚合, 数据分析以及驱动数据流从探针到用户界面的流程。分析包括 Skywalking 原生追踪和性能指标以及第三方来源，包括 Istio 及 Envoy telemetry , Zipkin 追踪格式化等。 你甚至可以使用 Observability Analysis Language 对原生度量指标 和 用于扩展度量的计量系统 自定义聚合分析。。
- **存储（Storage）**通过开放的插件化的接口存放 SkyWalking 数据. 你可以选择一个既有的存储系统, 如 ElasticSearch, H2 或 MySQL 集群(Sharding-Sphere 管理),也可以选择自己实现一个存储系统. 当然, 我们非常欢迎你贡献新的存储系统实现。
- **用户界面（UI）**一个基于接口高度定制化的Web系统，用户可以可视化查看和管理 SkyWalking 数据。

### 设计目标

本文档概述了 SkyWalking 项目的核心设计目标。

- **保持可观察性**。不管目标系统如何部署, SkyWalking 总要提供一种方案或集成方式来保持对目标系统的观测, 基于此, SkyWalking 提供了数种运行时探针。
- **拓扑结构, 性能指标和追踪一体化**。理解分布式系统的第一步是通过观察其拓扑结构图。拓扑图可以将复杂的系统在一张简单的图里面进行可视化展现. 基于拓扑图，运维支撑系统相关人员需要更多关于服务/实例/端点/调用的性能指标. 链路追踪(trace)作为详细的日志, 对于此种性能指标来说很有意义, 如你想知道什么时候端点延时变得很长, 想了解最慢的链路并找出原因. 因此你可以看到, 这些需求都是从大局到细节的, 都缺一不可。 SkyWalking 集成并提供了一系列特性来使得这些需求成为可能, 并且使之易于理解。
- **轻量级**。有两个方面需要保持轻量级. (1) 探针, 我们通常依赖于网络传输框架, 如 gRPC. 在这种情况下, 探针就应该尽可能小, 防止依赖库冲突以及虚拟机的负载压力(例如 JVM 永久代内存占用压力). (2) 作为一个观测平台, 在你的整个项目环境中只是次要系统, 因此我们使用自己的轻量级框架来构建后端核心服务. 所以你不需要部署并维护大数据相关的平台, SkyWalking 在技术栈方面应该足够简单。
- **可插拔**。SkyWalking 核心团队提供了很多默认的实现，但肯定是不够的，也不适合所有场景。因此，我们提供了许多可插拔的功能。
- **便携性**。SkyWalking 可以在多种环境中运行，包括： (1) 使用传统的注册中心，如 eureka。(2) 使用包含服务发现的RPC框架，如Spring Cloud、Apache Dubbo。(3) 在现代基础设施中使用 Service Mesh。(4) 使用云服务。(5) 跨云部署。SkyWalking 在所有这些情况下都应该运行良好。
- **互操作性**。可观测性领域如此广阔，以至于 SkyWalking 几乎不可能支持所有系统，即使在其社区的支持下也是如此。目前，它支持与其他 OSS 系统的互操作性，尤其是探针，如 Zipkin、Jaeger、OpenTracing 和 OpenCensus。SkyWalking 能够接受和读取这些数据格式对最终用户来说非常重要，因为用户不需要切换他们的库。

# skywalking参考资料
[Apache SkyWalking 社区](https://skywalking.apache.org/)
[Apache SkyWalking 官方文档](https://github.com/apache/skywalking/tree/master/docs)
[SkyWalking 文档中文版](https://skyapm.github.io/document-cn-translation-of-skywalking/)

在 https://github.com/apache/skywalking/tree/master/docs 地址下，提供了 SkyWalking 的**英文**文档。

推荐先阅读 https://github.com/SkyAPM/document-cn-translation-of-skywalking 地址，提供了 SkyWalking 的**中文**文档。

使用 SkyWalking 的目的，是实现**分布式链路追踪**的功能，所以最好去了解下相关的知识。这里推荐阅读两篇文章：

- [《OpenTracing 官方标准 —— 中文版》](https://github.com/opentracing-contrib/opentracing-specification-zh)
- Google 论文 [《Dapper，大规模分布式系统的跟踪系统》](http://www.iocoder.cn/Fight/Dapper-translation/?self)
