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
```shell
<metric name>{<label name>=<label value>, ...}
```
例如，指标名称为api_http_requests_total，标签为 method="POST" 和 handler="/messages" 的时间序列可以表示为：
```shell
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




















参考链接
·Prometheus官网：https://prometheus.io/。

·Prometheus文档：https://prometheus.io/docs/。

·Prometheus GitHub主页：https://github.com/prometheus/。

·Prometheus GitHub源码：https://github.com/prometheus/prometheus。

·Prometheus参考视频：大规模Prometheus和时间序列设计（https://www.youtube.com/watch?v=gNmWzkGViAY）。

·Grafana官网：https://grafana.com/。
------------------------------------------------
# Prometheus简介
Prometheus是由SoundCloud开发的开源监控报警系统和时序列数据库(TSDB)。
**Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud. Since its inception in 2012, many companies and organizations have adopted Prometheus, and the project has a very active developer and user community. It is now a standalone open source project and maintained independently of any company. To emphasize this, and to clarify the project's governance structure, Prometheus joined the Cloud Native Computing Foundation in 2016 as the second hosted project, after Kubernetes.**

### **什么是Prometheus？**

Prometheus是一个开源监控系统，它前身是SoundCloud的警告工具包。从2012年开始，许多公司和组织开始使用Prometheus。该项目的开发人员和用户社区非常活跃，越来越多的开发人员和用户参与到该项目中。目前它是一个独立的开源项目，且不依赖与任何公司。为了强调这点和明确该项目治理结构，Prometheus在2016年继Kurberntes之后，加入了Cloud Native Computing Foundation。

Prometheus是由SoundCloud开发的开源监控报警系统和时序列数据库(TSDB)。Prometheus使用Go语言开发，是Google BorgMon监控系统的开源版本。
2016年由Google发起Linux基金会旗下的原生云基金会(Cloud Native Computing Foundation), 将Prometheus纳入其下第二大开源项目。
Prometheus目前在开源社区相当活跃。
Prometheus和Heapster(Heapster是K8S的一个子项目，用于获取集群的性能数据。相比功能更完善、更全面。Prometheus性能也足够支撑上万台规模的集群。

### 特点

1）多维度数据模型

每一个时间序列数据都由metric度量指标名称和它的标签labels键值对集合唯一确定：这个metric度量指标名称指定监控目标系统的测量特征（如：http_requests_total- 接收http请求的总计数）。

labels开启了Prometheus的多维数据模型：对于相同的度量名称，通过不同标签列表的结合, 会形成特定的度量维度实例。(例如：所有包含度量名称为/api/tracks的http请求，打上method=POST的标签，则形成了具体的http请求)。这个查询语言在这些度量和标签列表的基础上进行过滤和聚合。改变任何度量上的任何标签值，则会形成新的时间序列图。

2）灵活的查询语言（PromQL）：可以对采集的metrics指标进行加法，乘法，连接等操作；

3）可以直接在本地部署，不依赖其他分布式存储；

4）通过基于HTTP的pull方式采集时序数据；

5）可以通过中间网关pushgateway的方式把时间序列数据推送到prometheus server端；

6）可通过服务发现或者静态配置来发现目标服务对象（targets）。

7）有多种可视化图像界面，如Grafana等。

8）高效的存储，每个采样数据占3.5 bytes左右，300万的时间序列，30s间隔，保留60天，消耗磁盘大概200G。

9）做高可用，可以对数据做异地备份，联邦集群，部署多套prometheus，pushgateway上报数据

### 组件

1）`Prometheus Server`: 用于收集和存储时间序列数据。

2）`Client Library`: 客户端库，检测应用程序代码，当Prometheus抓取实例的HTTP端点时，客户端库会将所有跟踪的metrics指标的当前状态发送到prometheus server端。

3）`Exporters`: prometheus支持多种exporter，通过exporter可以采集metrics数据，然后发送到prometheus server端，所有向promtheus server提供监控数据的程序都可以被称为exporter

4）`Alertmanager`: 从 Prometheus server 端接收到 alerts 后，会进行去重，分组，并路由到相应的接收方，发出报警，常见的接收方式有：电子邮件，微信，钉钉, slack等。

5）`Grafana`：监控仪表盘，可视化监控数据

6）`pushgateway`: 各个目标主机可上报数据到pushgateway，然后prometheus server统一从pushgateway拉取数据。

### **适用场景**

Prometheus在记录纯数字时间序列方面表现非常好。它既适用于面向服务器等硬件指标的监控，也适用于高动态的面向服务架构的监控。对于现在流行的微服务，Prometheus的多维度数据收集和数据筛选查询语言也是非常的强大。Prometheus是为服务的可靠性而设计的，当服务出现故障时，它可以使你快速定位和诊断问题。它的搭建过程对硬件和服务没有很强的依赖关系。

Prometheus，它的价值在于可靠性，甚至在很恶劣的环境下，你都可以随时访问它和查看系统服务各种指标的统计信息。 如果你对统计数据需要100%的精确，它并不适用，例如：它**不适用于实时计费系统**。



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