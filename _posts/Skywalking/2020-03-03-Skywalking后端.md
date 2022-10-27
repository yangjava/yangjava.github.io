---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
## Backend(后端)

### 可观测性分析平台（Observability Analysis Platform）

SkyWalking 是一个可观察性分析平台，可为在棕色和绿色区域中运行的服务以及使用混合模型的服务提供完全可观察性。

#### 能力

SkyWalking 涵盖了可观察性的所有 3 个领域，包括**Tracing**、**Metrics**和**Logging**。

- **追踪（Tracing）**。SkyWalking 原生数据格式，包括 Zipkin v1 和 v2，以及 Jaeger。
- **指标（Metrics）**。SkyWalking 与 Istio、Envoy 和 Linkerd 等 Service Mesh 平台集成，将可观察性构建到数据面板或控制面板中。此外，SkyWalking 原生代理可以在指标模式下运行，大大提高了性能。
- **日志（Logging）**。包括从磁盘或通过网络收集的日志。原生代理可以自动将跟踪上下文与日志绑定，或者使用 SkyWalking 通过文本内容绑定跟踪和日志。

有 3 个强大的本地语言引擎旨在分析来自上述领域的可观察性数据。

1. 可观察性分析语言处理原生跟踪和服务网格数据。
2. Meter Analysis Language负责对原生meter数据进行metrics计算，采用Prometheus、OpenTelemetry等稳定且应用广泛的metrics系统。
3. 日志分析语言专注于日志内容，与仪表分析语言协同工作。

### 可观察性分析语言（Observability Analysis Language）

OAL（可观察性分析语言）用于以流模式分析传入数据。

OAL 专注于服务、服务实例和端点中的指标。因此，该语言易于学习和使用。

`oal-rt`从 6.3 开始，OAL 引擎作为(OAL Runtime)嵌入在 OAP 服务器运行时中。现在可以在`/config`文件夹中找到 OAL 脚本，用户只需更改并重新启动服务器即可运行它们。但是，OAL 脚本是一种编译语言，OAL Runtime 会动态生成 java 代码。

您可以在系统环境中打开 set`SW_OAL_ENGINE_DEBUG=Y`以查看生成了哪些类。

#### 语法

脚本应该命名`*.oal`

```
// Declare the metrics.
METRICS_NAME = from(SCOPE.(* | [FIELD][,FIELD ...]))
[.filter(FIELD OP [INT | STRING])]
.FUNCTION([PARAM][, PARAM ...])

// Disable hard code 
disable(METRICS_NAME);
```

#### 范围

主要的**SCOPE**是`All`, `Service`, `ServiceInstance`, `Endpoint`, `ServiceRelation`, `ServiceInstanceRelation`, 和`EndpointRelation`。还有一些次要作用域属于主要作用域。

请参阅[范围定义](https://skywalking.apache.org/docs/main/v8.6.0/en/concepts-and-designs/scope-definitions)，您可以在其中找到所有现有的范围和字段。

##### 筛选

使用过滤器通过使用字段名称和表达式来构建字段值的条件。

表达式支持通过`and`,`or`和进行链接`(...)`。OP 支持`==`, `!=`, `>`, `<`, `>=`, `<=`, `in [...]`, `like %...`, `like ...%`, `like %...%`, `contain`and `not contain`, 以及基于字段类型的类型检测。在不兼容的情况下，可能会触发编译或代码生成错误。

##### 聚合函数

默认功能由 SkyWalking OAP 核心提供，可以实现附加功能。

提供的功能

- `longAvg`. 每个范围实体的所有输入的平均值。输入字段必须很长。

> instance_jvm_memory_max = from(ServiceInstanceJVMMemory.max).longAvg();

在这种情况下，输入代表每个 ServiceInstanceJVMMemory 范围的请求，而 avg 是基于 field 的`max`。

- `doubleAvg`. 每个范围实体的所有输入的平均值。输入字段必须是双精度。

> instance_jvm_cpu = from(ServiceInstanceJVMCPU.usePercent).doubleAvg();

在这种情况下，输入表示每个 ServiceInstanceJVMCPU 范围的请求，而 avg 是基于 field 的`usePercent`。

- `percent`. 数字或比率表示为 100 的分数，其中输入与条件匹配。

> endpoint_percent = from(Endpoint.*).percent(status == true);

在这种情况下，所有输入代表每个端点的请求，条件是`endpoint.status == true`。

- `rate`. 比率表示为 100 的分数，其中输入与条件匹配。

> browser_app_error_rate = from(BrowserAppTraffic.*).rate(trafficCategory == BrowserAppTrafficCategory.FIRST_ERROR, trafficCategory == BrowserAppTrafficCategory.NORMAL);

在这种情况下，所有输入代表每个浏览器应用程序流量的请求，`numerator`条件是`trafficCategory == BrowserAppTrafficCategory.FIRST_ERROR`，`denominator`条件是`trafficCategory == BrowserAppTrafficCategory.NORMAL`。参数（1）是`numerator`条件。参数（2）是`denominator`条件。

- `count`. 每个范围实体的调用总和。

> service_calls_sum = from(Service.*).count();

在这种情况下，每个服务的调用次数。

- `histogram`. 请参阅[WIKI](https://en.wikipedia.org/wiki/Heat_map)中的热图。

> all_heatmap = from(All.latency).histogram(100, 20);

在这种情况下，所有传入请求的热力学热图。参数（1）是延迟计算的精度，比如上面的例子，113ms和193ms被认为在101-200ms组中是一样的。参数（2）是组数量。在上述情况下，21（参数值 + 1）组为 0-100ms、101-200ms、... 1901-2000ms、2000+ms

- `apdex`. 请参阅[WIKI 中的 Apdex](https://en.wikipedia.org/wiki/Apdex)。

> service_apdex = from(Service.latency).apdex(name, status);

在这种情况下，每个服务的 apdex 分数。参数（1）为服务名称，反映从config文件夹中的service-apdex-threshold.yml加载的Apdex阈值。参数（2）是这个请求的状态。状态（成功/失败）反映 Apdex 计算。

- `p99`, `p95`, `p90`, `p75`, `p50`. 请参阅[WIKI 中的百分位数](https://en.wikipedia.org/wiki/Percentile)。

> all_percentile = from(All.latency).percentile(10);

**percentile**是自 7.0.0 以来引入的第一个多值指标。作为具有多个值的指标，可以通过`getMultipleLinearIntValues`GraphQL 查询进行查询。在这种情况下，请参阅所有传入请求的`p99`、`p95`、`p90`、`p75`和`p50`。该参数精确到 p99 的延迟，例如在上述情况下，120ms 和 124ms 被认为产生相同的响应时间。在 7.0.0 之前，`p99`, `p95`, `p90`, `p75`, `p50`func(s) 用于分别计算指标。它们在 7.x 中仍受支持，但不再推荐使用，也不包含在当前的官方 OAL 脚本中。

> all_p99 = from(All.latency).p99(10);

在这种情况下，所有传入请求的 p99 值。该参数精确到 p99 的延迟，例如在上述情况下，120ms 和 124ms 被认为产生相同的响应时间。

##### 指标名称

存储实施者、警报和查询模块的指标名称。类型推断由核心支持。

##### GROUP

所有指标数据将按 Scope.ID 和最小级别 TimeBucket 分组。

- 在`Endpoint`范围内，Scope.ID 与Endpoint ID 相同（即基于服务及其端点的唯一ID）。

##### 禁用

`Disable`是 OAL 中的高级语句，仅在某些情况下使用。一些聚合和指标是通过核心硬代码定义的。示例包括`segment`和`top_n_database_statement`。该`disable`语句旨在使它们处于非活动状态。默认情况下，它们都没有被禁用。

**注意**，所有禁用语句都应该在`oal/disable.oal`脚本文件中。

#### 例子

```
// Calculate p99 of both Endpoint1 and Endpoint2
endpoint_p99 = from(Endpoint.latency).filter(name in ("Endpoint1", "Endpoint2")).summary(0.99)

// Calculate p99 of Endpoint name started with `serv`
serv_Endpoint_p99 = from(Endpoint.latency).filter(name like "serv%").summary(0.99)

// Calculate the avg response time of each Endpoint
endpoint_avg = from(Endpoint.latency).avg()

// Calculate the p50, p75, p90, p95 and p99 of each Endpoint by 50 ms steps.
endpoint_percentile = from(Endpoint.latency).percentile(10)

// Calculate the percent of response status is true, for each service.
endpoint_success = from(Endpoint.*).filter(status == true).percent()

// Calculate the sum of response code in [404, 500, 503], for each service.
endpoint_abnormal = from(Endpoint.*).filter(responseCode in [404, 500, 503]).count()

// Calculate the sum of request type in [RequestType.RPC, RequestType.gRPC], for each service.
endpoint_rpc_calls_sum = from(Endpoint.*).filter(type in [RequestType.RPC, RequestType.gRPC]).count()

// Calculate the sum of endpoint name in ["/v1", "/v2"], for each service.
endpoint_url_sum = from(Endpoint.*).filter(name in ["/v1", "/v2"]).count()

// Calculate the sum of calls for each service.
endpoint_calls = from(Endpoint.*).count()

// Calculate the CPM with the GET method for each service.The value is made up with `tagKey:tagValue`.
service_cpm_http_get = from(Service.*).filter(tags contain "http.method:GET").cpm()

// Calculate the CPM with the HTTP method except for the GET method for each service.The value is made up with `tagKey:tagValue`.
service_cpm_http_other = from(Service.*).filter(tags not contain "http.method:GET").cpm()

disable(segment);
disable(endpoint_relation_server_side);
disable(top_n_database_statement);
```

### 指标分析语言（Meter Analysis Language）

指标系统提供一种称为 MAL（仪表分析语言）的功能分析语言，允许用户在 OAP 流系统中分析和汇总仪表数据。表达式的结果可以由代理分析器或 OC/Prometheus 分析器获取。

### 语言数据类型

在 MAL 中，表达式或子表达式可以计算为以下两种类型之一：

- **样本族**：一组样本（指标），包含一系列名称相同的指标。
- **标量**：一个简单的数值，支持整数/长整数和浮点/双精度。

## Sample family（样本族）

一组样本，作为 MAL 中的基本单元。例如：

```
instance_trace_count
```

上面的示例系列可能包含以下由外部模块提供的示例，例如代理分析器：

```
instance_trace_count{region="us-west",az="az-1"} 100
instance_trace_count{region="us-east",az="az-3"} 20
instance_trace_count{region="asia-north",az="az-1"} 33
```

### Tag filter（标签过滤）

MAL 支持四种类型的操作来过滤样本族中的样本：

- tagEqual：过滤标签与提供的字符串完全相同。
- tagNotEqual：过滤标签不等于提供的字符串。
- tagMatch：过滤与提供的字符串进行正则表达式匹配的标签。
- tagNotMatch：过滤与提供的字符串不匹配的标签。

例如，这会过滤 us-west 和 asia-north 区域以及 az-1 az 的所有 instance_trace_count 样本：

```
instance_trace_count.tagMatch("region", "us-west|asia-north").tagEqual("az", "az-1")
```

### Value filter（值过滤）

MAL 支持六种类型的操作来按值过滤样本族中的样本：

- valueEqual：过滤值与提供的值完全相等。
- valueNotEqual：过滤值等于提供的值。
- valueGreater：过滤大于提供值的值。
- valueGreaterEqual：过滤大于或等于提供值的值。
- valueLess：过滤小于提供值的值。
- valueLessEqual：过滤小于或等于提供值的值。

例如，这会过滤所有 instance_trace_count 样本的值 >= 33：

```
instance_trace_count.valueGreaterEqual(33)
```

### Tag manipulator（标签操纵器）

MAL 允许标签操作者更改（即添加/删除/更新）标签及其值。

### K8s

MAL 支持使用 K8s 的元数据来操作标签及其值。此功能需要授权 OAP Server 访问 K8s 的`API Server`.

##### retagByK8sMeta

`retagByK8sMeta(newLabelName, K8sRetagType, existingLabelName, namespaceLabelName)`. 根据现有标签的值向示例系列添加新标签。提供多种内部转换类型，包括

- K8sRetagType.Pod2Service

为样本添加一个标签，使用`service`作为键，`$serviceName.$namespace`作为值，并根据标签键的给定值，表示一个pod的名称。

例如：

```
container_cpu_usage_seconds_total{namespace=default, container=my-nginx, cpu=total, pod=my-nginx-5dc4865748-mbczh} 2
```

表达：

```
container_cpu_usage_seconds_total.retagByK8sMeta('service' , K8sRetagType.Pod2Service , 'pod' , 'namespace')
```

输出：

```
container_cpu_usage_seconds_total{namespace=default, container=my-nginx, cpu=total, pod=my-nginx-5dc4865748-mbczh, service='nginx-service.default'} 2
```

##### 二元运算符

MAL 中提供以下二元算术运算符：

- +（加法）
- \- （减法）
- *（乘法）
- / （分配）

二元运算符在标量/标量、sampleFamily/scalar 和 sampleFamily/sampleFamily 值对之间定义。

在两个标量之间：它们计算为另一个标量，该标量是运算符应用于两个标量操作数的结果：

```
1 + 2
```

在样本族和标量之间，运算符应用于样本族中每个样本的值。例如：

```
instance_trace_count + 2
```

或者

```
2 + instance_trace_count
```

结果是

```
instance_trace_count{region="us-west",az="az-1"} 102 // 100 + 2
instance_trace_count{region="us-east",az="az-3"} 22 // 20 + 2
instance_trace_count{region="asia-north",az="az-1"} 35 // 33 + 2
```

在两个样本族之间，对左侧样本族中的每个样本及其右侧样本族中的匹配样本应用二元运算符。将生成一个名称为空的新样本族。只有匹配的标签将被保留。右侧样本族中没有匹配样本的样本将不会出现在结果中。

另一个样本家庭`instance_trace_analysis_error_count`是

```
instance_trace_analysis_error_count{region="us-west",az="az-1"} 20
instance_trace_analysis_error_count{region="asia-north",az="az-1"} 11 
```

示例表达式：

```
instance_trace_analysis_error_count / instance_trace_count
```

这将返回一个包含跟踪分析错误率的结果样本系列。区域为 us-west 和 az az-3 的样本不匹配，也不会出现在结果中：

```
{region="us-west",az="az-1"} 0.8  // 20 / 100
{region="asia-north",az="az-1"} 0.3333  // 11 / 33
```

##### 聚合操作

样本族支持以下聚合操作，可用于聚合单个样本族的样本，从而生成具有较少样本（有时只有一个样本）且具有聚合值的新样本族：

- sum（计算维度的总和）
- min（选择最小尺寸）
- 最大（选择最大尺寸）
- avg（计算维度的平均值）

这些操作可用于聚合整体标签尺寸或通过输入`by`参数保留不同的尺寸。

```
<aggr-op>(by: <tag1, tag2, ...>)
```

示例表达式：

```
instance_trace_count.sum(by: ['az'])
```

将输出以下结果：

```
instance_trace_count{az="az-1"} 133 // 100 + 33
instance_trace_count{az="az-3"} 20
```

### 功能

`Duraton`是时间范围的文本表示。接受的格式基于 ISO-8601 持续时间格式 {@code PnDTnHnMn.nS}，其中一天被视为 24 小时。

例子：

- “PT20.345S”——解析为“20.345 秒”
- “PT15M” – 解析为“15 分钟”（其中一分钟为 60 秒）
- “PT10H”——解析为“10 小时”（其中 1 小时为 3600 秒）
- “P2D”——解析为“2 天”（一天是 24 小时或 86400 秒）
- “P2DT3H4M”——解析为“2天3小时4分钟”
- “P-6H3M” – 解析为“-6 小时 +3 分钟”
- “-P6H3M” – 解析为“-6 小时 -3 分钟”
- “-P-6H+3M” – 解析为“+6 小时 -3 分钟”

#### increase（增加）

`increase(Duration)`：计算时间范围内的增量。

#### rate（平均速率）

`rate(Duration)`：计算时间范围内每秒的平均增长率。

#### irate（瞬时速率）

`irate()`：计算时间范围内每秒的瞬时增长率。

#### tag（标签）

`tag({allTags -> })`：更新样本的标签。用户可以添加、删除、重命名和更新标签。

#### histogram（直方图）

`histogram(le: '<the tag name of le>')`：将基于较少的直方图桶转换为计量系统直方图桶。 `le`参数表示存储桶的标签名称。

#### histogram_percentile

`histogram_percentile([<p scalar>])`. 表示计量系统从桶中计算 p 百分位数 (0 ≤ p ≤ 100)。

#### time（时间）

`time()`：返回自 1970 年 1 月 1 日 UTC 以来的秒数。

#### 下采样操作

MAL 应该指导计量系统如何对指标进行下采样。它不仅指聚合原始样本到 `minute`级别，还表示来自`minute`更高级别的数据，例如`hour`和`day`。

在 MAL中调用下采样函数`downsampling`，它接受以下类型：

- AVG
- SUM
- LATEST
- MIN (TODO)
- MAX (TODO)
- MEAN (TODO)
- COUNT (TODO)

默认类型是`AVG`.

如果用户想从以下位置获取最新时间`last_server_state_sync_time_in_seconds`：

```
last_server_state_sync_time_in_seconds.tagEqual('production', 'catalog').downsampling(LATEST)
```

#### 度量级函数

Metric 分为三个级别：服务、实例和端点。他们从度量标签中提取级别相关标签，然后通知仪表系统该度量所属的级别。

- `servcie([svc_label1, svc_label2...])`从数组参数中提取服务级别标签。
- `instance([svc_label1, svc_label2...], [ins_label1, ins_label2...])`从第一个数组参数中提取服务级别标签，从第二个数组参数中提取实例级别标签。
- `endpoint([svc_label1, svc_label2...], [ep_label1, ep_label2...])`从第一个数组参数中提取服务级别标签，从第二个数组参数中提取端点级别标签。
