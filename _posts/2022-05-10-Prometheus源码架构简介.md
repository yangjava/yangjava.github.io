---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---
# Prometheus源码架构
Prometheus 是云原生监控领域的事实标准，越来越多的开源项目开始支持 Prometheus 监控数据格式。学习 Prometheus 的设计理念，阅读分析 Prometheus 源码，了解 Prometheus 的局限性与不足。

## Prometheus架构分析

任何应用服务想要接入 Prometheus，都需要提供 HTTP 接口（通常是 x.x.x.x/metrics 地址），并暴露 Prometheus 格式的监控数据。Prometheus Server 通过 HTTP 协议周期性抓取监控目标的监控数据、打时间戳、存储到本地。Prometheus 提供了 Client 库帮助开发人员在自己的应用中集成符合 Prometheus 格式标准的监控指标。
而对于不适合直接在代码中集成 Client 库的场景，比如应用来自第三方、不是由自己维护，应用不支持 HTTP 协议，那就需要为这些场景单独编写 Exporter 程序。Exporter 作为代理，把监控数据暴露出来。比如 Mysql Exporter，Node Exporter。
Prometheus 将采集到的数据存储在本地时序数据库中，但缺少数据副本。这也是 Prometheus 自身在数据持久化方面做的不足的地方。但这些存储问题都有其他的解决方案，Prometheus 支持 remote write 方式将数据存储到远端。
Prometheus 支持通过 Kubernetes、静态文本、Consul、DNS 等多种服务发现方式来获取抓取目标（targets）。最后，用户编写 PromQL 语句查询数据并进行可视化。

## 核心组件

Prometheus 的功能由多个互相协作的组件共同完成。这些组件也即本文开头所列出的模块，比如 Service Discovery Manager。我们后续会逐一介绍。Prometheus 源码入口 main() 函数 完成参数初始化工作，并依次启动各依赖组件。

首先，main 函数解析命令行参数（详见附录 A），并读取配置文件信息（由 --config.file 参数提供）。Prometheus 特别区分了命令行参数配置（flag-based configuration）和文件配置（file-based configuration）。前者用于简单的设置，并且不支持热更新，修改需要启停 Prometheus Server 一次；后者支持热更新。

main 函数完成初始化、启动所有的组件。这些组件包括：Termination Handler、Service Discovery Manager、Web Handler 等。各组件是独立的 Go Routine 在运行，之间又通过各种方式相互协调，包括使用 Channel、引用对象 Reference、传递 Context（Context 包的使用可以参考作者的 《Golang Context 包详解》一文）。

这些 Go Routine 的协作使用了 oklog/run 框架。oklog/run 是一套基于 Actor 设计模式的 Go Routine 编排框架，实现了多个 Go Routine 作为统一整体运行并有序依次退出。

## 源码结构
源码地址
```text
https://github.com/prometheus/prometheus
```
源码目录结构说明
```text
cmd目录是prometheus的入口和promtool规则校验工具的源码
discovery是prometheus的服务发现模块，主要是scrape targets，其中包含consul, zk, azure, file,aws, dns, gce等目录实现了不同的服务发现逻辑，可以看到静态文件也作为了一种服务发现的方式，毕竟静态文件也是动态发现服务的一种特殊形式
config用来解析yaml配置文件，其下的testdata目录中有非常丰富的各个配置项的用法和测试

notifier负责通知管理，规则触发告警后，由这里通知服务发现的告警服务，之下只有一个文件，不需要特别关注

pkg是内部的依赖

- relabel ：根据配置文件中的relabel对指标的label重置处理 

- pool：字节池

- timestamp：时间戳

- rulefmt：rule格式的验证

- runtime：获取运行时信息在程序启动时打印

prompb定义了三种协议，用来处理远程读写的远程存储协议，处理tsdb数据的rpc通信协议，被前两种协议使用的types协议，例如使用es做远程读写，需要远程端实现远程存储协议(grpc)，远程端获取到的数据格式来自于types中，就是这么个关系

promql处理查询用的promql语句的解析

rules负责告警规则的加载、计算和告警信息通知

scrape是核心的根据服务发现的targets获取指标存储的模块

storge处理存储，其中fanout是存储的门面，remote是远程存储，本地存储用的下面一个文件夹

tsdb时序数据库，用作本地存储
```


## 源码组件
```text
Scrape manager， 拉取指标的核心组件

Rule manager，告警处理组件

TSDB，本地存储组件

Notifier manager，通知组件

ScrapeDiscovery manager，用于target的动态发现

NotifyDiscovery manager，用于告警服务的动态发现

Web handler，查询的接口和页面的提供
```

## 架构分析

任何应用服务想要接入 Prometheus，都需要提供 HTTP 接口（通常是 x.x.x.x/metrics 地址），并暴露 Prometheus 格式的监控数据。Prometheus Server 通过 HTTP 协议周期性抓取监控目标的监控数据、打时间戳、存储到本地。Prometheus 提供了 Client 库帮助开发人员在自己的应用中集成符合 Prometheus 格式标准的监控指标。

而对于不适合直接在代码中集成 Client 库的场景，比如应用来自第三方、不是由自己维护，应用不支持 HTTP 协议，那就需要为这些场景单独编写 Exporter 程序。Exporter 作为代理，把监控数据暴露出来。比如 Mysql Exporter，Node Exporter。

Prometheus 将采集到的数据存储在本地时序数据库中，但缺少数据副本。这也是 Prometheus 自身在数据持久化方面做的不足的地方。但这些存储问题都有其他的解决方案，Prometheus 支持 remote write 方式将数据存储到远端。

Prometheus 支持通过 Kubernetes、静态文本、Consul、DNS 等多种服务发现方式来获取抓取目标（targets）。最后，用户编写 PromQL 语句查询数据并进行可视化。


