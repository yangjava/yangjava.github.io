---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---
# Prometheus入门简介
Prometheus，一个开源的监控系统，它从应用程序中实时获取时间序列数据，然后通过功能强大的规则引擎，帮助你识别监控环境所需的信息。

- Prometheus及其起源
- Prometheus的发展
- Prometheus架构和设计
- Prometheus数据模型
- Prometheus生态系统

## Prometheus起源
很久以前，加利福尼亚州山景城有一家名为Google的公司。该公司推出了大量产品，其中最著名的是广告系统和搜索引擎平台。为了运行这些不同的产品，该公司建立了一个名为Borg的平台。Borg系统是：“一个集群管理器，可以运行来自成千上万个不同应用程序的成千上万个作业，它跨越多个集群，每个集群都有数万台服务器。”开源容器管理平台Kubernetes的很多部分都是对Borg平台的传承。在Borg部署到Google后不久，该公司意识到这种复杂性需要一个同等水平的监控系统。Google建立了这个系统并命名为Borgmon。Borgmon是一个实时的时间序列监控系统，它使用时间序列数据来识别问题并发出警报。

Prometheus的灵感来自谷歌的Borgmon。它最初由前谷歌SRE Matt T.Proud开发，并转为一个研究项目。在Proud加入SoundCloud之后，他与另一位工程师Julius Volz合作开发了Prometheus。后来其他开发人员陆续加入了这个项目，并在SoundCloud内部继续开发，最终于2015年1月将其发布。

与Borgmon一样，Prometheus主要用于提供近实时的、基于动态云环境和容器的微服务、服务和应用程序的内省监控。SoundCloud是这些架构模式的早期采用者，而Prometheus的建立也是为了满足这些需求。如今，Prometheus被更多的公司广泛使用，通常用来满足类似的监控需求，但也用来监控传统架构的资源。

Prometheus专注于现在正在发生的事情，而不是追踪数周或数月前的数据。它基于这样一个前提，即大多数监控查询和警报都是从最近的（通常是一天内的）数据中生成的。Facebook在其内部时间序列数据库Gorilla的论文中验证了这一观点。Facebook发现85％的查询是针对26小时内的数据。Prometheus假定你尝试修复的问题可能是最近出现的，因此最有价值的是最近时间的数据，这反映在强大的查询语言和通常有限的监控数据保留期上。

Prometheus由开源编程语言Go编写，并且是在Apache 2.0许可证下授权的。它孵化于云原生计算基金会（Cloud Native Computing Foundation）

## 特性
- 通过指标名称和标签(key/value对）区分的多维度、时间序列数据模型
- 灵活的查询语法 PromQL
- 不需要依赖额外的存储，一个服务节点就可以工作
- 利用http协议，通过pull模式来收集时间序列数据
- 需要push模式的应用可以通过中间件gateway来实现
- 监控目标支持服务发现和静态配置
- 支持各种各样的图表和监控面板组件

## Prometheus架构
Prometheus通过抓取或拉取应用程序中暴露的时间序列数据来工作。时间序列数据通常由应用程序本身通过客户端库或称为exporter（导出器）的代理来作为HTTP端点暴露。目前已经存在很多exporter和客户端库，支持多种编程语言、框架和开源应用程序，如Apache Web服务器和MySQL数据库等。

Prometheus还有一个推送网关（push gateway），可用于接收少量数据——例如，来自无法拉取的目标数据（如临时作业或者防火墙后面的目标）。

## 指标收集
Prometheus称其可以抓取的指标来源为端点（endpoint）。端点通常对应单个进程、主机、服务或应用程序。为了抓取端点数据，Prometheus定义了名为目标（target）的配置。这是执行抓取所需的信息——例如，如何进行连接，要应用哪些元数据，连接需要哪些身份验证，或者定义抓取将如何执行的其他信息。一组目标被称为作业（job）。作业通常是具有相同角色的目标组——例如，负载均衡器后面的Apache服务器集群，它们实际上是一组相似的进程。

生成的时间序列数据将被收集并存储在Prometheus服务器本地，也可以设置从服务器发送数据到外部存储器或其他时间序列数据库。

## 服务发现
可以通过多种方式来处理要监控的资源的发现，包括：
- 用户提供的静态资源列表。
- 基于文件的发现。例如，使用配置管理工具生成在Prometheus中可以自动更新的资源列表。
- 自动发现。例如，查询Consul等数据存储，在Amazon或Google中运行实例，或使用DNS SRV记录来生成资源列表。

## 聚合和警报
服务器还可以查询和聚合时间序列数据，并创建规则来记录常用的查询和聚合。这允许你从现有时间序列中创建新的时间序列，例如，计算变化率和比率，或者产生类似求和等聚合。这样就不必重新创建常用的聚合，例如用于调试，并且预计算可能比每次需要时运行查询性能更好。

Prometheus还可以定义警报规则。这些是为系统配置的在满足条件时触发警报的标准，例如，资源时间序列开始显示异常的CPU使用率。Prometheus服务器没有内置警报工具，而是将警报从Prometheus服务器推送到名为Alertmanager（警报管理器）的单独服务器。Alertmanager可以管理、整合和分发各种警报到不同目的地——例如，它可以在发出警报时发送电子邮件，并能够防止重复发送。

## 查询数据
Prometheus服务器还提供了一套内置查询语言PromQL、一个表达式浏览器以及一个用于浏览服务器上数据的图形界面。

## 自治
每个Prometheus服务器都设计为尽可能自治，旨在支持扩展到数千台主机的数百万个时间序列的规模。数据存储格式被设计为尽可能降低磁盘的使用率，并在查询和聚合期间快速检索时间序列。

## 冗余和高可用性
冗余和高可用性侧重弹性而非数据持久性。Prometheus团队建议将Prometheus服务器部署到特定环境和团队，而不是仅部署一个单体Prometheus服务器。如果你确实要部署高可用HA模式，则可以使用两个或多个配置相同的Prometheus服务器来收集时间序列数据，并且所有生成的警报都由可消除重复警报的高可用Alertmanager集群来处理。

## 可视化
可视化通过内置表达式浏览器提供，并与开源仪表板Grafana集成。此外，Prometheus也支持其他仪表板。

## Prometheus数据模型
Prometheus收集时间序列数据。为了处理这些数据，它使用一个多维时间序列数据模型。这个时间序列数据模型结合了时间序列名称和称为标签（label）的键/值对，这些标签提供了维度。每个时间序列由时间序列名称和标签的组合唯一标识。

## 指标名称
时间序列名称通常描述收集的时间序列数据的一般性质——例如，website_visits_total为网站访问的总数。 名称可以包含ASCII字符、数字、下划线和冒号。

## 标签
标签为Prometheus数据模型提供了维度。它们为特定时间序列添加上下文。例如，total_website_visits时间序列可以使用能够识别网站名称、请求IP或其他特殊标识的标签。Prometheus可以在一个时间序列、一组时间序列或者所有相关的时间序列上进行查询。

标签共有两大类：插桩标签（instrumentation label）和目标标签（target label）。插桩标签来自被监控的资源——例如，对于与HTTP相关的时间序列，标签可能会显示所使用的特定HTTP动词。这些标签在由诸如客户端或exporter抓取之前会被添加到时间序列中。目标标签更多地与架构相关——它们可能会识别时间序列所在的数据中心。目标标签由Prometheus在抓取期间和之后添加。

时间序列由名称和标签标识（尽管从技术上讲，名称本身也是名为__name__的标签）。如果你在时间序列中添加或更改标签，那么Prometheus会将其视为新的时间序列。你可以理解label就是键/值形式的标签，并且新的标签会创建新的时间序列。

## 采样数据
时间序列的真实值是采样（sample）的结果，在时间序列中的每一个点称为一个样本（sample），样本由以下三部分组成：
- 指标（metric）：指标名称和描述当前样本特征的 labelsets；
- 时间戳（timestamp）：一个精确到毫秒的时间戳；
- 样本值（value）： 一个 folat64 的浮点型数据表示当前样本的值。

表示方式：通过如下表达方式表示指定指标名称和指定标签集合的时间序列：
```
<metric name>{<label name>=<label value>, ...}
```
例如，指标名称为api_http_requests_total，标签为 method="POST" 和 handler="/messages" 的时间序列可以表示为：
```
api_http_requests_total{method="POST", handler="/messages"}
```
首先是时间序列名称，后面跟着一组键/值对标签。通常所有时间序列都有一个instance标签（标识源主机或应用程序）以及一个job标签（包含抓取特定时间序列的作业名称）。

## 保留时间
Prometheus专为短期监控和警报需求而设计。默认情况下，它在其数据库中保留15天的时间序列数据。如果要保留更长时间的数据，则建议将所需数据发送到远程的第三方平台。Prometheus能够写入外部数据存储

## 安全模型
Prometheus可以通过多种方式进行配置和部署，关于安全有以下两个假设：
- 不受信任的用户将能够访问Prometheus服务器的HTTP API，从而访问数据库中的所有数据。
- 只有受信任的用户才能访问Prometheus命令行、配置文件、规则文件和运行时配置。
因此，Prometheus及其组件不提供任何服务器端的身份验证、授权或加密。如果你在一个更加安全的环境中工作，则需要自己实施安全控制——例如，通过反向代理访问Prometheus服务器或者正向代理exporter。由于不同版本的配置会潜在地发生较大变化，因此本书没有记录如何执行这些操作。

## Prometheus生态系统
Prometheus生态系统由Prometheus项目本身提供的组件以及丰富的开源工具和套件组成。生态系统的核心是Prometheus服务器。此外还有Alertmanager，它为Prometheus提供警报引擎并进行管理。

Prometheus项目还包括一系列exporter，用于监控应用程序和服务，并在端点上公开相关指标以进行抓取。核心exporter支持常见工具，如Web服务器、数据库等。许多其他exporter都是开源的，你可以从Prometheus社区查看。

Prometheus还发布了一系列客户端库，支持监控由多种语言编写的应用程序和服务。它们包括主流编程语言，如Python、Ruby、Go和Java。其他客户端库也可以从开源社区获取。

## 核心组件
整个Prometheus生态包含多个组件，除了Prometheus server组件其余都是可选的
- Prometheus Server：主要的核心组件，用来收集和存储时间序列数据。
- Client Library:：客户端库，为需要监控的服务生成相应的 metrics 并暴露给 Prometheus server。当 Prometheus server 来 pull 时，直接返回实时状态的 metrics。
- push gateway：主要用于短期的 jobs。由于这类 jobs 存在时间较短，可能在 Prometheus 来 pull 之前就消失了。为此，这次 jobs 可以直接向 Prometheus server 端推送它们的 metrics。这种方式主要用于服务层面的 metrics，对于机器层面的 metrices，需要使用 node exporter。
- Exporters: 用于暴露已有的第三方服务的 metrics 给 Prometheus。
- Alertmanager: 从 Prometheus server 端接收到 alerts 后，会进行去除重复数据，分组，并路由到对收的接受方式，发出报警。常见的接收方式有：电子邮件，pagerduty，OpsGenie, webhook 等。
- 各种支持工具。


## Prometheus 框架图
Prometheus 的主要模块包含:Server, Exporters, Pushgateway, PromQL, Alertmanager, WebUI 等。我们逐一认识一下各个模块的功能作用。

模块介绍
- Retrieval是负责定时去暴露的目标页面上去抓取采样指标数据。
- Storage 是负责将采样数据写入指定的时序数据库存储。
- PromQL 是Prometheus提供的查询语言模块。可以和一些webui比如grfana集成。
- Jobs / Exporters:Prometheus 可以从 Jobs 或 Exporters 中拉取监控数据。Exporter 以 Web API 的形式对外暴露数据采集接口。
- Prometheus Server:Prometheus 还可以从其他的 Prometheus Server 中拉取数据。
- Pushgateway:对于一些以临时性 Job 运行的组件，Prometheus 可能还没有来得及从中 pull 监控数据的情况下，这些 Job 已经结束了，Job 运行时可以在运行时将监控数据推送到 Pushgateway 中，Prometheus 从 Pushgateway 中拉取数据，防止监控数据丢失。
- Service discovery:是指 Prometheus 可以动态的发现一些服务，拉取数据进行监控，如从DNS，Kubernetes，Consul 中发现, file_sd 是静态配置的文件。
- AlertManager:是一个独立于 Prometheus 的外部组件，用于监控系统的告警，通过配置文件可以配置一些告警规则，Prometheus 会把告警推送到 AlertManager。

## 工作流程
大概的工作流程如下：
- Prometheus server 定期从配置好的 jobs 或者 exporters 中拉 metrics，或者接收来自 Pushgateway 发过来的 metrics，或者从其他的 Prometheus server 中拉 metrics。
- Prometheus server 在本地存储收集到的 metrics，并运行已定义好的 alert.rules，记录新的时间序列或者向 Alertmanager 推送警报。
- Alertmanager 根据配置文件，对接收到的警报进行处理，发出告警。
- 在图形界面中，可视化采集数据。

## Prometheus相关概念

### 内部存储机制
Prometheus有着非常高效的时间序列数据存储方法，每个采样数据仅仅占用3.5byte左右空间，上百万条时间序列，30秒间隔，保留60天，大概花了200多G（引用官方PPT）。

Prometheus内部主要分为三大块：
- Retrieval是负责定时去暴露的目标页面上去抓取采样指标数据
- Storage是负责将采样数据写磁盘
- PromQL是Prometheus提供的查询语言模块。

### 什么是样本

**样本**：在时间序列中的每一个点称为一个样本（sample），样本由以下三部分组成：

- 指标（metric）：指标名称和描述当前样本特征的 labelsets；
- 时间戳（timestamp）：一个精确到毫秒的时间戳；
- 样本值（value）： 一个 folat64 的浮点型数据表示当前样本的值。

表示方式：通过如下表达方式表示指定指标名称和指定标签集合的时间序列：

```
<metric name>{<label name>=<label value>, ...}
```

例如，指标名称为api_http_requests_total，标签为 method="POST" 和 handler="/messages" 的时间序列可以表示为：

```
api_http_requests_total{method="POST", handler="/messages"}
```

### 数据模型
Prometheus 存储的所有数据都是时间序列数据（Time Serie Data，简称时序数据）。时序数据是具有时间戳的数据流，该数据流属于某个度量指标（Metric）和该度量指标下的多个标签（Label）。

每个Metric name代表了一类的指标，他们可以携带不同的Labels，每个Metric name + Label组合成代表了一条时间序列的数据。

在Prometheus的世界里面，所有的数值都是64bit的。每条时间序列里面记录的其实就是64bit timestamp(时间戳) + 64bit value(采样值)。

- Metric name（指标名称）：该名字应该具有语义，一般用于表示 metric 的功能，例如：http_requests_total, 表示 http 请求的总数。其中，metric 名字由 ASCII 字符，数字，下划线，以及冒号组成，且必须满足正则表达式 [a-zA-Z_:][a-zA-Z0-9_:]*。
- Lables（标签）：使同一个时间序列有了不同维度的识别。例如 http_requests_total{method=“Get”} 表示所有 http 请求中的 Get 请求。当 method=“post” 时，则为新的一个 metric。标签中的键由 ASCII 字符，数字，以及下划线组成，且必须满足正则表达式 [a-zA-Z_:][a-zA-Z0-9_:]*。
- timestamp(时间戳)：数据点的时间，表示数据记录的时间。
- Sample Value（采样值）：实际的时间序列，每个序列包括一个 float64 的值和一个毫秒级的时间戳。

例如数据：
```
http_requests_total{status="200",method="GET"}
http_requests_total{status="404",method="GET"}
```
根据上面的分析，时间序列的存储似乎可以设计成key-value存储的方式（基于BigTable）。

现在Prometheus内部的表现形式了，__name__是特定的label标签，代表了metric name。

## Metric类型
Prometheus定义了4种不同的指标类型(metric type)：Counter（计数器）、Gauge（仪表盘）、Histogram（直方图）、Summary（摘要）

### Counter（计数器）
一种累加的 metric，典型的应用如：请求的个数，结束的任务数， 出现的错误数等等。
【例如】查询 http_requests_total{method=“get”, job=“Prometheus”, handler=“query”} 返回 8，10 秒后，再次查询，则返回 14。

### Gauge（仪表盘）
数据是一个瞬时值，如果当前内存用量，它随着时间变化忽高忽低。
【例如】go_goroutines{instance=“172.17.0.2”, job=“Prometheus”} 返回值 147，10 秒后返回 124。

### Histogram（直方图）
Histogram 取样观测的结果（一般是请求持续时间或响应大小）并在一个可配置的分布区间（bucket）内计算这些结果。其也提供所有观测结果的总和。
Histogram 有一个基本 metric名称 <basename>，在一次抓取中展现多个时间序列：
累加的 counter，代表观测区间：<basename>_bucket{le=""}
所有观测值的总数：<basename>_sum
观测到的事件数量：<basenmae>_count
例如 Prometheus server 中prometheus_local_storage_series_chunks_persisted, 表示 Prometheus 中每个时序需要存储的 chunks 数量，我们可以用它计算待持久化的数据的分位数。

### Summary（摘要）
和 histogram 相似，summary 取样观测的结果（一般是请求持续时间或响应大小）。但是它还提供观测的次数和所有值的总和，它通过一个滑动的时间窗口计算可配置的分位数。
Summary 有一个基本的 metric名称 <basename>，在一次抓取中展现多个时间序列：
观测事件的流式φ-分位数（0 ≤ φ ≤ 1）：{quantile=“φ”}
所有观测值的总和：<basename>_sum
观测的事件数量：<basename>_count
例如 Prometheus server 中 prometheus_target_interval_length_seconds。

## 任务（JOBS）与实例（INSTANCES）
用Prometheus术语来说，可以抓取的端点称为instance，通常对应于单个进程。
具有相同目的的instances 的集合（例如，出于可伸缩性或可靠性而复制的过程）称为job。

例如，一个具有四个复制实例的API服务器作业:
```
job: api-server
    instance 1: 1.2.3.4:5670
    instance 2: 1.2.3.4:5671
    instance 3: 5.6.7.8:5670
    instance 4: 5.6.7.8:5671
```
- instance: 一个单独 scrape 的目标， 一般对应于一个进程。:
- jobs: 一组同种类型的 instances（主要用于保证可扩展性和可靠性）

## Node exporter
Node exporter 主要用于暴露 metrics 给 Prometheus，其中 metrics 包括：cpu 的负载，内存的使用情况，网络等。

## Pushgateway
Pushgateway 是 Prometheus 生态中一个重要工具，使用它的原因主要是：

Prometheus 采用 pull 模式，可能由于不在一个子网或者防火墙原因，导致Prometheus 无法直接拉取各个 target数据。
在监控业务数据的时候，需要将不同数据汇总, 由 Prometheus 统一收集。
由于以上原因，不得不使用 pushgateway，但在使用之前，有必要了解一下它的一些弊端：

将多个节点数据汇总到 pushgateway, 如果 pushgateway 挂了，受影响比多个 target 大。
Prometheus 拉取状态 up 只针对 pushgateway, 无法做到对每个节点有效。
Pushgateway 可以持久化推送给它的所有监控数据。因此，即使你的监控已经下线，prometheus 还会拉取到旧的监控数据，需要手动清理 pushgateway 不要的数据。

## TSDB简介
TSDB(Time Series Database)时序列数据库，我们可以简单的理解为一个优化后用来处理时间序列数据的软件，并且数据中的数组是由时间进行索引的。

### 时间序列数据库的特点
大部分时间都是写入操作。
写入操作几乎是顺序添加，大多数时候数据到达后都以时间排序。
写操作很少写入很久之前的数据，也很少更新数据。大多数情况在数据被采集到数秒或者数分钟后就会被写入数据库。
删除操作一般为区块删除，选定开始的历史时间并指定后续的区块。很少单独删除某个时间或者分开的随机时间的数据。
基本数据大，一般超过内存大小。一般选取的只是其一小部分且没有规律，缓存几乎不起任何作用。
读操作是十分典型的升序或者降序的顺序读。
高并发的读操作十分常见。

### 常见的时间序列数据库
- influxDB	https://influxdata.com/
- RRDtool	http://oss.oetiker.ch/rrdtool/
- Graphite	http://graphiteapp.org/
- OpenTSDB	http://opentsdb.net/
- Kdb+	http://kx.com/
- Druid	http://druid.io/
- KairosDB	http://kairosdb.github.io/
- Prometheus	https://prometheus.io/

# 参考链接
Prometheus官网：https://prometheus.io/。

Prometheus文档：https://prometheus.io/docs/。

Prometheus GitHub主页：https://github.com/prometheus/。

Prometheus GitHub源码：https://github.com/prometheus/prometheus。

Prometheus参考视频：大规模Prometheus和时间序列设计（https://www.youtube.com/watch?v=gNmWzkGViAY）。

Grafana官网：https://grafana.com/。

