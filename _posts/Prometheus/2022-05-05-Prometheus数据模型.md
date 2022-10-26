---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---

# 数据模型

prometheus将所有数据存储为时间序列：属于相同 metric名称和相同标签组（键值对）的时间戳值流。

metric 和 标签：
每一个时间序列都是由其 metric名称和一组标签（键值对）组成唯一标识。

metric名称代表了被监控系统的一般特征（如 http_requests_total代表接收到的HTTP请求总数）。它可能包含ASCII字母和数字，以及下划线和冒号，它必须匹配正则表达式[a-zA-Z_:][a-zA-Z0-9_:]*。

注意：冒号是为用户定义的记录规则保留的，不应该被exporter使用。

标签给prometheus建立了多维度数据模型：对于相同的 metric名称，标签的任何组合都可以标识该 metric的特定维度实例（例如：所有使用POST方法到 /api/tracks 接口的HTTP请求）。查询语言会基于这些维度进行过滤和聚合。更改任何标签值，包括添加或删除标签，都会创建一个新的时间序列。

标签名称可能包含ASCII字母、数字和下划线，它必须匹配正则表达式[a-zA-Z_][a-zA-Z0-9_]*。另外，以双下划线__开头的标签名称仅供内部使用。

标签值可以包含任何Unicode字符。标签值为空的标签被认为是不存在的标签。

表示法：
给定 metric名称和一组标签，通常使用以下表示法标识时间序列：

<metric name>{<label name>=<label value>, ...}
1
例如，一个时间序列的 metric名称是 api_http_requests_total，标签是 method="POST" 和 handler="/messages"。可以这样写：

api_http_requests_total{method="POST", handler="/messages"}
1
这和OpenTSDB的表示法是一样的。

| 类型      | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| Counter   | 值只能单调增加或重启时归零，可以用来表示处理的请求数、完成的任务数、出现的错误数量等 |
| Gauge     | 值可以任意增加或减少，可以用来测量温度、当前内存使用等       |
| Histogram | 取样观测结果，一般用来请求持续时间或响应大小，并在一个可配置的分布区间（bucket）内计算这些结果，提供所有观测结果的总和<br/>                        <br/>                        累加的 counter，代表观测区间：<basename>_bucket{le="<upper inclusive bound>"}<br/>                        所有观测值的总数：<basename>_sum<br/>                        观测的事件数量：<basenmae>_count |
| Summary   | 取样观测结果，一般用来请求持续时间或响应大小，提供观测次数及所有观测结果的总和，还可以通过一个滑动的时间窗口计算可分配的分位数<br/>                        观测的事件流φ-quantiles (0 ≤ φ ≤ 1)：<basename>{quantile="φ"}<br/>                        所有观测值的总和：<basename>_sum<br/>                        观测的事件数量：<basename>_count |

实例与任务：
在prometheus中，一个可以拉取数据的端点叫做实例（instance），一般等同于一个进程。一组有着同样目标的实例（例如为弹性或可用性而复制的进程副本）叫做任务（job）。

当prometheus拉取目标时，它会自动添加一些标签到时间序列中，用于标识被拉取的目标：

job：目标所属的任务名称

instance：目标URL中的<host>:<port>部分
如果两个标签在被拉取的数据中已经存在，那么就要看配置选项 honor_labels 的值来决定行为了。

每次对实例的拉取，prometheus会在以下的时间序列中保存一个样本（样本指的是在一个时间序列中特定时间点的一个值）：

up{job="<job-name>", instance="<instance-id>"}：如果实例健康（可达），则为 1 ，否则为 0

scrape_duration_seconds{job="<job-name>", instance="<instance-id>"}：拉取的时长

scrape_samples_post_metric_relabeling{job="<job-name>", instance="<instance-id>"}：在 metric relabeling 之后，留存的样本数量

scrape_samples_scraped{job="<job-name>", instance="<instance-id>"}：目标暴露出的样本数量
up 时间序列对于实例的可用性监控来说非常有用。


### Counter

Counter是**计数器类型**：

- Counter 用于累计值，例如记录请求次数、任务完成数、错误发生次数。
- 一直增加，不会减少。
- 重启进程后，会被重置。

```bash
# Counter类型示例
http_response_total{method="GET",endpoint="/api/tracks"}  100
http_response_total{method="GET",endpoint="/api/tracks"}  160
```

Counter 类型数据可以让用户方便的了解事件产生的速率的变化，在PromQL内置的相关操作函数可以提供相应的分析，比如以HTTP应用请求量来进行说明

1）通过`rate()`函数获取HTTP请求量的增长率：`rate(http_requests_total[5m])`

2）查询当前系统中，访问量前10的HTTP地址：`topk(10, http_requests_total)`

### Gauge

Gauge是**测量器类型**：

- Gauge是常规数值，例如温度变化、内存使用变化。
- 可变大，可变小。
- 重启进程后，会被重置

```bash
# Gauge类型示例
memory_usage_bytes{host="master-01"}   100
memory_usage_bytes{host="master-01"}   30
memory_usage_bytes{host="master-01"}   50
memory_usage_bytes{host="master-01"}   80 
```

对于 Gauge 类型的监控指标，通过 PromQL 内置函数 `delta()` 可以获取样本在一段时间内的变化情况，例如，计算 CPU 温度在两小时内的差异：
`dalta(cpu_temp_celsius{host="zeus"}[2h])`

你还可以通过PromQL 内置函数 `predict_linear()` 基于简单线性回归的方式，对样本数据的变化趋势做出预测。例如，基于 2 小时的样本数据，来预测主机可用磁盘空间在 4 个小时之后的剩余情况：`predict_linear(node_filesystem_free{job="node"}[2h], 4 * 3600) < 0`

### Histogram

#### Histogram作用及特点

histogram是**柱状图**，在Prometheus系统的查询语言中，有三种作用：

1）在一段时间范围内对数据进行采样（通常是请求持续时间或响应大小等），并将其计入可配置的存储桶（`bucket`）中. 后续可通过指定区间筛选样本，也可以统计样本总数，最后一般将数据展示为直方图。

2）对每个采样点值累计和(`sum`)

3）对采样点的次数累计和(`count`)

**度量指标名称**: [basename]*上面三类的作用度量指标名称*

```bash
1）[basename]bucket{le="上边界"}, 这个值为小于等于上边界的所有采样点数量
2）[basename]_sum_
3）[basename]_count
```

小结：如果定义一个度量类型为Histogram，则Prometheus会自动生成三个对应的指标

#### 为什需要用histogram柱状图

在大多数情况下人们都倾向于使用某些量化指标的平均值，例如 CPU 的平均使用率、页面的平均响应时间。这种方式的问题很明显，以系统 API 调用的平均响应时间为例：**如果大多数 API 请求都维持在 100ms 的响应时间范围内，而个别请求的响应时间需要 5s，那么就会导致某些 WEB 页面的响应时间落到中位数的情况，而这种现象被称为`长尾问题`**。
​ 为了区分是平均的慢还是长尾的慢，最简单的方式就是按照请求延迟的范围进行分组。例如，**统计延迟在 0~10ms 之间的请求数有多少，而 10~20ms 之间的请求数又有多少。通过这种方式可以快速分析系统慢的原因**。Histogram 和 Summary 都是为了能够解决这样问题的存在，通过 Histogram 和 Summary 类型的监控指标，我们可以快速了解监控样本的分布情况。

Histogram 类型的样本会提供三种指标（假设指标名称为 `<basename>`）：

1）样本的值分布在 bucket 中的数量，命名为 `<basename>_bucket{le="<上边界>"}`。解释的更通俗易懂一点，这个值表示指标值小于等于上边界的所有样本数量。

```bash
# 1、在总共2次请求当中。http 请求响应时间 <=0.005 秒 的请求次数为0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.005",} 0.0

# 2、在总共2次请求当中。http 请求响应时间 <=0.01 秒 的请求次数为0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.01",} 0.0

# 3、在总共2次请求当中。http 请求响应时间 <=0.025 秒 的请求次数为0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.025",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.05",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.075",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.1",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.25",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.5",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.75",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="1.0",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="2.5",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="5.0",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="7.5",} 2.0

# 4、在总共2次请求当中。http 请求响应时间 <=10 秒 的请求次数为 2
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="10.0",} 2.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="+Inf",} 2.0
```

2）所有样本值的大小总和，命名为 `<basename>_sum`

```bash
# 实际含义： 发生的2次 http 请求总的响应时间为 13.107670803000001 秒
io_namespace_http_requests_latency_seconds_histogram_sum{path="/",method="GET",code="200",} 13.107670803000001
```

3）样本总数，命名为 `<basename>_count`，值和 `<basename>_bucket{le="+Inf"}` 相同

```bash
# 实际含义： 当前一共发生了 2 次 http 请求
io_namespace_http_requests_latency_seconds_histogram_count{path="/",method="GET",code="200",} 2.0
```

**注意**：
1）bucket 可以理解为是对数据指标值域的一个划分，划分的依据应该基于数据值的分布。注意后面的采样点是包含前面的采样点的，假设 `xxx_bucket{...,le="0.01"}` 的值为 10，而 `xxx_bucket{...,le="0.05"}` 的值为 30，那么意味着这 30 个采样点中，有 10 个是小于 0.01s的，其余 20 个采样点的响应时间是介于0.01s 和 0.05s之间的。

2）可以通过 `histogram_quantile()` 函数来计算 Histogram 类型样本的分位数。分位数可能不太好理解，你可以理解为分割数据的点。我举个例子，假设样本的 9 分位数**（quantile=0.9）的值为 x，即表示小于 x 的采样值的数量占总体采样值的 90%**。Histogram 还可以用来计算应用性能指标值（Apdex score）。

### Summary

与 Histogram 类型类似，用于表示一段时间内的数据采样结果（通常是请求持续时间或响应大小等），但它直接存储了分位数（通过客户端计算，然后展示出来），而不是通过区间来计算。它也有三种作用：

1）对于每个采样点进行统计，并形成分位图。（如：正态分布一样，统计低于60分不及格的同学比例，统计低于80分的同学比例，统计低于95分的同学比例）

2）统计班上所有同学的总成绩(sum)

3）统计班上同学的考试总人数(count)

带有度量指标的[basename]的summary 在抓取时间序列数据有如命名。

1、观察时间的`φ-quantiles (0 ≤ φ ≤ 1)`, 显示为`[basename]{分位数="[φ]"}`

2、`[basename]_sum`， 是指所有观察值的总和_

3、`[basename]_count`, 是指已观察到的事件计数值

样本值的分位数分布情况，命名为 `<basename>{quantile="<φ>"}`。

```bash
# 1、含义：这 12 次 http 请求中有 50% 的请求响应时间是 3.052404983s
io_namespace_http_requests_latency_seconds_summary{path="/",method="GET",code="200",quantile="0.5",} 3.052404983

# 2、含义：这 12 次 http 请求中有 90% 的请求响应时间是 8.003261666s
io_namespace_http_requests_latency_seconds_summary{path="/",method="GET",code="200",quantile="0.9",} 8.003261666
```

所有样本值的大小总和，命名为 `<basename>_sum`。

```bash
# 1、含义：这12次 http 请求的总响应时间为 51.029495508s
io_namespace_http_requests_latency_seconds_summary_sum{path="/",method="GET",code="200",} 51.029495508
```

样本总数，命名为 `<basename>_count`。

```bash
# 1、含义：当前一共发生了 12 次 http 请求
io_namespace_http_requests_latency_seconds_summary_count{path="/",method="GET",code="200",} 12.0
```

**Histogram 与 Summary 的异同**：

它们都包含了 `<basename>_sum` 和 `<basename>_count` 指标，Histogram 需要通过 `<basename>_bucket` 来计算分位数，而 Summary 则直接存储了分位数的值。

```bash
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.5"} 0.012352463
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.9"} 0.014458005
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.99"} 0.017316173
prometheus_tsdb_wal_fsync_duration_seconds_sum 2.888716127000002
prometheus_tsdb_wal_fsync_duration_seconds_count 216

# 从上面的样本中可以得知当前Promtheus Server进行wal_fsync操作的总次数为216次，耗时2.888716127000002s。其中中位数（quantile=0.5）的耗时为0.012352463，9分位数（quantile=0.9）的耗时为0.014458005s。
```