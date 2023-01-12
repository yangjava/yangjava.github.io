---
layout: post
categories: Observerbility
description: none
keywords: Observerbility
---
# Observerbility可观测性
而可观测性一词近两年火起来的导火索是 CNCF 在云原生定义中提到 Observerbility，并声称这是云原生时代的必备能力。于是从生产所需到概念发声，加之包括谷歌在内的众多大厂一拥而上，“可观测性”正式出道。

## 可观测性的定义

Observability是来自控制论的一个概念：
**In control theory, observability is a measure for how well internal states of a system can be inferred by knowledge of its external outputs. The observability and controllability of a system are mathematical duals. The concept ofobservability was introduced by American-Hungarian scientist Rudolf E. Kalmanfor linear dynamic systems.**

官方话语，感兴趣的读者可以自行翻译。

用相对严谨的话来说，可观测性指的是一种能力--是通过检查其输出来衡量系统内部状态的能⼒。这些输出体现内部系统状态的能力越强，可观测性也就越好。

简单来看，如果仅使⽤来⾃输出的信息（即传感器数据）可以估计当前状态，则系统被认为是“可观测的”。

## 可观测性的价值

谷歌给出可观测性的核心价值很简单：快速排障（troubleshooting）。

这个世界上没有不存在 Bug 的系统，而随着系统越来越精细，越来越复杂，越来越动态，越来越庞大，潜藏的问题和风险也就越来越多。

因此，任何一个软件的成功，不仅仅要依靠软件架构的合理设计，软件开发的代码质量，更要依靠软件系统的运行维护。而运行维护的基础，就是可观测性。

从银行的交易系统，互联网公司的业务平台，到运营商的云化核心网等运行在云上的各类软件系统，每时每刻都处在一定的风险之中。而保证这些系统能够风险可控，稳定运行，需要做的就是提前发现异常，快速定位根本原因，迅速排除或者规避故障。

因此，在 CNCF 对于云原生的定义中，已经明确将可观测性列为一项必备要素。

## 可观测性的三大支柱
业界对可观测性的共识，基于可观测性的三大支柱“metrics、logs、traces”。

1、logs（日志）

⽇志是在特定时间发⽣的事件的⽂本记录，包括说明事件发⽣时间的时间戳和提供上下⽂的有效负载。⽇志有三种格式：纯⽂本、结构化和⼆进制。纯⽂本是最常⻅的，但结构化⽇志⸺包括额外的数据和元数据并且更容易查询⸺正变得越来越流⾏。当系统出现问题时，⽇志通常也是您⾸先查看的地⽅。

2、metrics（指标）

指标是在⼀段时间内测量的数值，包括特定属性，例如时间戳、名称、KPI 和值。与⽇志不同，指标在默认情况下是结构化的，这使得查询和优化存储变得更加容易，让您能够将它们保留更⻓时间。

3、traces（跟踪）

跟踪表示请求通过分布式系统的端到端旅程。当请求通过主机系统时， 对其执⾏的每个操作（称为“跨度”）都使⽤与执⾏该操作的微服务相关的重要数据进⾏编码。通过查看跟踪，每个跟踪都包含⼀个或多个跨度，您可以通过分布式系统跟踪其进程并确定瓶颈或故障的原因。

从一个简单的“系统探查--日志搜集--日志统计”流程来看：三者之间的关系从 traces 开始，通过探查等手段采集众多信息，形成logs，logs详细记录了各种边界行为（如登陆，开启服务，关闭服务，退出系统，修改数据），而对于系统运行来说，更重要的是 特定事件发生的次数。这些信息可以从日志中提取，但是有一种更有效的方法：metrics。

至此，如果你的 metrics 与告警相关联，则可在系统关键节点设置阈值。如果指标超过了阈值，随叫随到的人员就会收到Slack或微软团队中的电子邮件、短信或消息。快速响应排出故障。

总结
可观测性简单来说就是通过检查其输出来衡量系统内部状态的能⼒。这些输出体现内部系统状态的能力越强，可观测性也就越好。

其价值在于快速排障（troubleshooting）。

当下，业界对可观测性的共识，基于三大支柱“metrics、logs、traces”。

那么，要构建一个优秀的可观测系统，仅有 metrics、logs、traces 是不是就够用了呢？我们下期再接着聊。

## OpenTracing
官网地址：https://opentracing.io/

OpenTracing是CNCF（Cloud Native Computing Foundation projects）的项目，它是一个与厂商无关的API，并提供了一种规范，可以帮助开发人员轻松的在他们的代码上集成tracing。官方提供了Go, JavaScript, Java, Python, Ruby, PHP, Objective-C, C++, C#等语言的支持。它是开发的不属于任何一家公司。事实上有很多公司正在支持OpenTracing，例如：Zipkin和Jaeger都遵循OpenTracing协议。

2016年11月的时候CNCF技术委员会投票接受OpenTracing作为Hosted项目，这是CNCF的第三个项目，第一个是Kubernetes，第二个是Prometheus，可见CNCF对OpenTracing背后可观察性的重视。

OpenTracing希望围绕什么是Tracing以及如何和我们的应用程序集成形成一种通用语言。OpenTracing中的trace是一个有向无环图，其引用如下：

[Span A]  ←←←(the root span)
|
+------+------+
|             |
[Span B]      [Span C] ←←←(Span C is a `ChildOf` Span A)
|             |
[Span D]      +---+-------+
|           |
[Span E]    [Span F] >>> [Span G] >>> [Span H]
↑
↑
↑
(Span G `FollowsFrom` Span F)
## OpenCensus
官网地址：https://opencensus.io/

OpenCensus是Google开源的，作为最早提出Tracing概念的公司，OpenCensus也是Google Dapper的社区版本。

OpenSensus是允许你采集应用程序Metrics和分布式Traces，并且支持各种语言的库，比如：Go, Java, C#, Node.js, C++, Ruby, Erlang/Elixir, Python, Scala and PHP。支持将数据实时传输到后端。开发人员或管理员可以分析这些数据，以了解应用程序的运行状况或调试问题。

读到这里，我们可以很容易分别出OpenTracing和OpenCensus的差别，主要在于OpenCensus把Metrics包括进来了，不仅可以采集traces，还支持采集metrics，还有一点不同OpenCensus并不是单纯的规范制定，他还把包括数据采集的Agent、Collector。

## OpenCensus VS OpenTracing
1、OpenTracing只支持traces，OpenCensus支持traces和metrics。

2、OpenTracing支持单纯的制定规范，OpenCensus不仅制定规范，还包含了Agent和Collector。

总结：OpenTracing的支持厂商众多，例如：ElasticSearch、Uber、Skywalking、Couchbase。OpenCensus除了有google和微软两大巨头支持，还有Datadog、Jaeger、Zipkin、Prometheus，阵容非常强大。


## OpenTelemetry
官网地址：https://opentelemetry.io/

OpenTelemetry是CNCF的孵化项目。由OpenTracing和OpenCensus项目合并而成。OpenTelemetry是一组APIs、SDKs、工具和集成，旨在创建和管理遥测数据，如traces、metrics和logs。该项目提供了一个与厂商无关的实现，可以用来将遥测数据发送到你指定的后端，它支持各种流行的开源项目，包括Jaeger和Prometheus。

为什么需要OpenTelemetry及其功能
在微服务，云原生的技术栈中，分布式和多语言体系的架构越来越常见。分布式体系结构带来了各种操作挑战，包括如何快速解决可用性和性能问题。这些挑战导致了必须提高可观测性。

遥测数据为可观测性产品提供动力。传统上，遥测数据由开源项目或商业供应商提供。在缺乏标准化的情况下，最终结果是缺乏数据可移植性，并给用户带来维护的负担。

OpenTelemetry项目通过提供单一的、与厂商无关的解决方案来解决这些问题。该项目得到了云提供商、厂商和最终用户的广泛行业支持和采用。

OpenTelemetry庞大的生态系统由以下几部分组成：

跨语言的标准/规范

定义了跨语言的规范。除术语定义外，还包含：

API：定义了tracing、metrics和logging数据的类型和操作。

SDK：定义API的特定语言实现需求，这里还定义了配置、数据处理和导出概念。

数据：定义OpenTelemetry Line Protocol （OTLP）和厂商无关的语义约定，后端可以为其提供支持。

Collector: 采集（collect）、转换（transform）和导出（export）遥测数据的功能

OpenTelemetry Collector是一个与厂商无关的代理，可以接收、处理和导出遥测数据。它支持以多种格式（例如OTLP、Jaeger、Prometheus以及许多商业/专有工具）接收遥测数据，并将数据发送到一个或多个后端。它还支持在输出遥测数据之前对其进行处理和过滤。Collector contrib软件包支持更多数据格式和后端。

各种语言的SDK，包含：C++、.NET、Erlang/Elixir、Go、Java、JavaScript、PHP、Python、Ruby、Rust、Swift

OpenTelemetry还包含SDK，SDK是通过使用OpenTelemetry API使用您选择的语言生成遥测数据，并将该数据导出到后端。这些SDK还允许你为公共库或框架增强，一些开源的APM框架会开发很多这样的SDK，例如Skywalking支持多种中间件框架，所以会经常发布SDK。

自动增强（Automatic instrumentation）和 contrib packages

注：Instrumentation大家可以参考Java里面的概念，主要是通过字节码增强的方式实现。例如我们通过增强Springboot，去采集每次Http请求的rt和错误码。

通过如下图，简单理解OpenTelemetry：



OpenTelemetry支持的数据源
1、Metrics

metric是关于一个service的度量（measurement），在运行时捕获。从逻辑上讲，捕获其中一个量度的时刻称为metric event，它不仅包含量度本身，还包括获取它的时间和相关元数据。

应用程序和请求metrics是可用性和性能的重要指标。自定义指标可以深入了解可用性如何影响用户体验和业务。自定义metrics可以深入理解可用性metrics是如何影响用户体验或业务的。

OpenTelemetry目前定义了三个metric工具：

counter: 一个随时间求和的值，可以理解成汽车的里程表，它只会上升。

measure: 随时间聚合的值。它表示某个定义范围内的值。

observer: 捕捉特定时间点的一组当前值，如车辆中的燃油表。

2、Logs

日志是带有时间戳的文本记录，可以是带有元数据结构化（推荐）的，也可以是非结构化的。虽然每个日志都是一个独立的数据源，但它们也可以附加到trace的span中。我们日常使用调用的时候，那个节点出问题了，伴随着也可能看到日志。

在OpenTelemetry中，任何不属于分布式trace或metrics的数据都是日志。例如，event是一种特定类型的日志。日志通常用于确定问题的根本原因，通常包含有关谁更改了内容以及更改结果的信息。

3、Traces

trarce是指单个请求的追踪，请求可以由应用程序发起，也可以由用户发起。例如：订单应用通过RPC服务器访问商品库存应用，或是用户在浏览器端执行一次下单请求。分布式tracing是一种跨网络，跨应用程序的追踪形式。每个工作单元在trace中被称为span，一个trace由一个树形的 span 组成。span表示经过应用程序所设计的服务或组件所做工作的对象，span还提供了可用于调试可用性和性能问题的请求、错误和持续时间的metrics。span包含了一个span上下文，它是一组全局唯一标识符，表示每个span所属的唯一请求。通常我们称之为trace id。

4、Baggage

除了 trace 的传播，OpenTelemetry 还提供了 Baggage 来传播键值对。Baggage 用于索引一个服务中的可观察事件，该服务包含同一事务中先前的服务提供的属性，有助于在事件之间建立因果关系。

虽然 Baggage 可以用作其他横切关注点的原型，但这种机制主要是为了传递 OpenTelemetry 可观测性系统的值。

这些值可以从 Baggage 中消费，并作为度量的附加维度，或日志和跟踪的附加上下文使用。一些例子:

web 服务可以从包含发送请求的服务的上下文中获益

SaaS 提供商可以包含有关负责该请求的 API 用户或令牌的上下文

确定特定浏览器版本与图像处理服务中的故障相关联

w3c baggage : https://w3c.github.io/baggage/

OpenTelemetry不是什么？
OpenTelemetry不是像Jaeger、Skywalking、Prometheus这框架具备存储，查询，dashboard的服务。相反，它支持将数据导出到各种开源和商业后端。它提供了一个可插拔的体系结构，因此可以轻松添加附加的技术协议和格式。


五、什么是遥测数据
遥测数据是通过传感器被遥测终端接收到的实时数据。来自遥测对象，反映遥测对象的数字特征或状态，可作为科研和决策分析的数据依据。

以上是百度百科对遥测数据的说明，对于可观测领域，我们可以理解成遥测数据是从可观测端采集上来的数据，例如：浏览器对DOM的运行数据、服务器端CPU、IO等数据、网络数据、应用程序中公共库和中间件的运行数据。

六、总结
OpenTelemetry定义了数据标准，同时支持了采集和传输，这类公司是相当繁重的。我们平常一个项目使用到的公共库和开源框架可能就有十几种，还可能存在版本的差异，而且项目和项目的语言可能还不一样，如果要对多语言，多开源框架进行遥测数据的采集，工作量之大可想而知。

如果考虑做监控系统，可以考虑使用OpenTelemetry，可以帮助我们省去这些繁琐的数据采集的开发，将精力集中在数据的分析、存储和产品化上。让APM产品更易用，而从提高开发和运维对系统可观测性的需求。

参考文档：

https://opentelemetry.io/

https://www.bmc.com/blogs/opentracing-opencensus-openmetrics/
