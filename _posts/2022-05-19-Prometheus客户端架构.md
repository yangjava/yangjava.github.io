---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus客户端架构

## 基础知识

### TSDB
Prometheus会将所有采集到的样本数据以时间序列（time-series）的方式保存在内存数据库中，并且定时保存到硬盘上。
time-series是按照时间戳和值的序列顺序存放的，我们称之为向量(vector). 每条time-series通过指标名称(metrics name)和一组标签集(labelset)命名。

### 样本(Sample)
在time-series中的每一个点称为一个样本（sample），样本由以下三部分组成：

| name           | desc                           |
|----------------|:-------------------------------|
| 指标(metric)     | metric name和描述当前样本特征的labelsets |
| 时间戳(timestamp) | 一个精确到毫秒的时间戳;                   |
| 样本值(value)     | 一个float64的浮点型数据表示当前样本的值        |

### Metric

Metric 含有一个name,比如node_cpu
Metric含有一组标签键值对,比如node_cpu{cpu=“cpu0”,mode=“idle”} cpu和mode称为label标签,cpu0和idle称为label_value标签值
常见的metric有如下分类

| name      | desc                       |
|-----------|----------------------------|
|  Counter  | 计数器,递增量                    |
| Gauge     | 仪表盘,状态量                    |
| Histogram | 直方图,服务端计算                  |
| Summary   | 摘要图,客户端计算,PromeQL性能更好      |

### exporter
Prometheus官方提供的一些黑盒监控采集工具，如node_exporter等等
java的simple-client,exporter表示http端点等指标暴露端点,collector表示监控采集相关容器
```xml
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient</artifactId>
    <version>0.16.0</version>
</dependency>
```

## 架构说明

| 类                              | 说明                                                                                                          |
|--------------------------------|-------------------------------------------------------------------------------------------------------------|
| CollectorRegister              | 所有Collector的容器,exporter从CollectorRegister获取所有的Metrics度量信息                                                   |
| Collector                      | 一个Collector为一个metrics的收集器,收集该metrics的labels对应的所有label_values的指标信息,常见子类Counter,Gauge,直方图Histogram,分位图summary |
| Child                          | 每一个Metric有多个label_name数据,(abel_name对应多组label_values)一组label_values对应一个child,对目标对象监控时,child为监控对象指标的容器        |
| MetricFamilySamples            | CollectorRegister进行统一收集时,将每个Collector的child中的数据转化为MetricFamilySamples                                       |
| Sample                         | 一组sample构成一个MetricFamilySamples,一个sample对应一个Collector.child,对应一组label_values                                |
| Exemplar                       | Sample被prometheus采集后用来构建线条, Exemplar为一些打点数据,                                                                |
| Builder                        | 负责构建Collector,建造者模式                                                                                         |
| MetricFamilySamplesEnumeration | CollectorRegister负责Collector的注册, MetricFamilySamplesEnumeration负责汇总对prometheus的指标采集                         |









