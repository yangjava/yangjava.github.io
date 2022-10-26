---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---

#  **Prometheus简介**

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