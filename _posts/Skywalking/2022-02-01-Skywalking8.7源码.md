# Skywalking源码

## 简介

```
SkyWalking: an APM(application performance monitor) system, especially designed for microservices, cloud native and container-based architectures.
```

**官网介绍**：分布式系统的应用程序性能监控工具，专为微服务、云原生和基于容器的 (Kubernetes) 架构而设计。

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

### 架构

![Skywalking架构图](png\Skywalking架构图.png)

SkyWalking 在逻辑上分为四个部分：Probes（探针）、Platform backend（平台后端）、Storage（存储） 和 UI。

- **探测器（Probes）**收集数据并根据 SkyWalking 要求重新格式化（不同的探测器支持不同的来源）。
- **平台后端（Platform backend）**支持数据聚合、分析和流式处理，包括跟踪、度量和日志。
- **存储（Storage）**通过开放/可插入接口存储 SkyWalking 数据。您可以选择现有的实现，例如 ElasticSearch、H2、MySQL、TiDB、InfluxDB，也可以自己实现。欢迎为新的存储实现者打补丁！
- **UI**是一个高度可定制的基于 Web 的界面，允许 SkyWalking 最终用户可视化和管理 SkyWalking 数据。

## 官方文档

在 https://github.com/apache/skywalking/tree/master/docs 地址下，提供了 SkyWalking 的**英文**文档。

推荐先阅读 https://github.com/SkyAPM/document-cn-translation-of-skywalking 地址，提供了 SkyWalking 的**中文**文档。

使用 SkyWalking 的目的，是实现**分布式链路追踪**的功能，所以最好去了解下相关的知识。这里推荐阅读两篇文章：

- [《OpenTracing 官方标准 —— 中文版》](https://github.com/opentracing-contrib/opentracing-specification-zh)
- Google 论文 [《Dapper，大规模分布式系统的跟踪系统》](

# Skywalking源码解析