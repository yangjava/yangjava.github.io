---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---

# Prometheus架构

## 架构图

![Prometheus](Prometheus.png)

从上图可发现，Prometheus整个生态圈组成主要包括prometheus server，Exporter，pushgateway，alertmanager，grafana，Web ui界面。

Prometheus server由三个部分组成，Retrieval，Storage，PromQL

- `Retrieval`负责在活跃的target主机上抓取监控指标数据
- `Storage`存储主要是把采集到的数据存储到磁盘中
- `PromQL`是Prometheus提供的查询语言模块。

## 工作流程

1）Prometheus server可定期从活跃的（up）目标主机上（target）拉取监控指标数据，目标主机的监控数据可通过配置静态job或者服务发现的方式被prometheus server采集到，这种方式默认的pull方式拉取指标；也可通过pushgateway把采集的数据上报到prometheus server中；还可通过一些组件自带的exporter采集相应组件的数据；

2）Prometheus server把采集到的监控指标数据保存到本地磁盘或者数据库；

3）Prometheus采集的监控指标数据按时间序列存储，通过配置报警规则，把触发的报警发送到alertmanager

4）Alertmanager通过配置报警接收方，发送报警到邮件，微信或者钉钉等

5）Prometheus 自带的web ui界面提供PromQL查询语言，可查询监控数据

6）Grafana可接入prometheus数据源，把监控数据以图形化形式展示出

## Prometheus**基础概念**

### 数据模型

Prometheus 存储的是[时序数据](https://en.wikipedia.org/wiki/Time_series), 即按照相同时序(相同的名字和标签)，以时间维度存储连续的数据的集合。时序(time series) 是由名字(Metric)，以及一组 key/value 标签定义的，具有相同的名字以及标签属于相同时序。时序的名字由 ASCII 字符，数字，下划线，以及冒号组成，它必须满足正则表达式 `[a-zA-Z_:][a-zA-Z0-9_:]*`, 其名字应该具有语义化，一般表示一个可以度量的指标，例如 `http_requests_total`, 可以表示 http 请求的总数。

时序的标签可以使 Prometheus 的数据更加丰富，能够区分具体不同的实例，例如 `http_requests_total{method="POST"}` 可以表示所有 http 中的 POST 请求。标签名称由 ASCII 字符，数字，以及下划线组成， 其中 `__` 开头属于 Prometheus 保留，标签的值可以是任何 Unicode 字符，支持中文。

### **指标类型**（Metric Types）

Prometheus 时序数据分为 [Counter](https://prometheus.io/docs/concepts/metric_types/#counter), [Gauge](https://prometheus.io/docs/concepts/metric_types/#gauge), [Histogram](https://prometheus.io/docs/concepts/metric_types/#histogram), [Summary](https://prometheus.io/docs/concepts/metric_types/#summary) 四种类型。

1. Counter：表示收集的数据是按照某个趋势（增加／减少）一直变化的，我们往往用它记录服务请求总量，错误总数等。例如 Prometheus server 中 `http_requests_total`, 表示 Prometheus 处理的 http 请求总数，我们可以使用data, 很容易得到任意区间数据的增量。
2. Gauge：表示搜集的数据是一个瞬时的，与时间没有关系，可以任意变高变低，往往可以用来记录内存使用率、磁盘使用率等。
3. Histogram：Histogram 由 `<basename>_bucket{le="<upper inclusive bound>"}`，`<basename>_bucket{le="+Inf"}`, `<basename>_sum`，`<basename>_count` 组成，主要用于表示一段时间范围内对数据进行采样，（通常是请求持续时间或响应大小），并能够对其指定区间以及总数进行统计，通常我们用它计算分位数的直方图。
4. Summary：Summary 和 Histogram 类似，由 `<basename>{quantile="<φ>"}`，`<basename>_sum`，`<basename>_count`组成，主要用于表示一段时间内数据采样结果，（通常是请求持续时间或响应大小），它直接存储了 quantile 数据，而不是根据统计区间计算出来的。区别在于：

a. 都包含 `<basename>_sum`，`<basename>_count。`

``        b. Histogram 需要通过 `<basename>_bucket` 计算 quantile, 而 Summary 直接存储了 quantile 的值。

