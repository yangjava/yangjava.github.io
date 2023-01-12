---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
### 基础概念

span 、trace segment 、contextCarrier 、contextSnapshot

#### span

Span 是分布式追踪系统中一个非常重要的概念，可以理解为一次方法调用、一个程序块的调用、一次 RPC 调用或者数据库访问。

依据是跨线程还是跨进程的链路，将 Span粗略分为两类：LocalSpan 和RemoteSpan。LocalSpan 代表一次普通的 Java 方法调用，与跨进程无关，多用于当前进程中关键逻辑代码块的记录，或在跨线程后记录异步线程执行的链路信息。RemoteSpan 可细分为 EntrySpan 和 ExitSpan：

-  EntrySpan 代表一个应用服务的提供端或服务端的人口端点，如 Web 容器的服务端的入口、RPC 服务器的消费者、消息队列的消费者;

- ExitSpan （SkyWalking 的早期版本中称其为 LeafSpan)，代表一个应用服务的客户端或消息队列的生产者，如 Redis 客户端的一次 Redis 调用、MySQL 客户端的一次数据库查询、RPC 组件的一次请求、消息队列生产者的生产消息。

#### Trace Segment

Trace Segment 是 SkyWalking 中特有的概念，通常指在支持多线程的语言中，一个线程中归属于同一个操作的所有 Span 的聚合。这些 Span 具有相同的唯一标识SegmentID。Trace Segment对应的实体类位于org.apache.skywalking.apm.agent,core.context.trace.TraceSegment，其中重要的属性如下。

- TraceSegmentld：此 Trace Segment 操作的唯一标识。使用雪花算法生成，保证全局唯一。
- Refs：此 Trace Segment 的上游引用。对于大多数上游是 RPC 调用的情况，Refs只有一个元素，但如果是消息队列或批处理框架，上游可能会是多个应用服务，所以就会存在多个元素。
- Spans：用于存储，从属于此 Trace Segment 的 Span 的集合。
- RelatedGlobalTraces ：此 Trace Segment 的 Trace Id。大多数时候它只包含一个元素，但如果是消息队列或批处理框架，上游是多个应用服务，会存在多个元素。
- lgnore ：是否忽略。如果Ignore 为true，则此Trace Segment 不会上传到SkyWalking 后端。
- IsSizeLimited：从属于此 Trace Segment 的 Span 数量限制，初始化大小可以通过config.agent.span_limit_per_segment 参数来配置，默认长度为 300。若超过配置值，在创建新的 Span 的时候，会变成 NoopSpan。NoopSpan 表示没有任何实际操作的 Span 实现，用于保持内存和 GC 成本尽可能低。
- CreateTime：此 Trace Segment 的创建时间。

#### ContextCarrier

分布式追踪要解决的一个重要问题是跨进程调用链的连接，ContextCarrier 的就是为了解决这个问题。如客户端 A、服务端 B 两个应用服务，当发生一次 A 调用 B 的时候,

跨进程传播的步骤如下。

- 1）客户端 A 创建空的 ContextCarrier。
- 2）通过 ContextManager#createExitSpan 方法创建一个ExitSpan，或者使用ContextManager#inject 在过程中传入并初始化 ContextCarrier。
- 3）使用 ContextCarrier.items()将 ContextCarrier 所有元素放到调用过程中的请求信息中，如 HTTP HEAD、Dubbo RPC 框架的 attachments、消息队列 Kafka 消息的 header 中。
- 4）ContextCarrier 随请求传输到服务端。
- 5）服务端 B 接收具有 ContextCarrier 的请求，并提取 ContextCarrier 相关的所有信息。
- 6）通过 ContextManager#createEntrySpan 方法创建 EntrySpan，或者使用ContextManager#extract 建立分布式调用关联，即绑定服务端 B 和客户端A。

#### ContextSnapshot

除了跨进程，跨线程也是需要支持的，例如异步线程（内存中的消息队列）在Java中很常见。跨线程和跨进程十分相似，都需要传播上下文，唯一的区别是，跨线程不需要序列化。以下是跨线程传播的步骤。

- 1）使用 ContextManager#capture 方法获取 ContextSuapshot 对象。
- 2）让子线程以任何方式，通过方法参数或由现有参数携带来访问 ContextSnapshot。
- 3）在子线程中使用 ContextManager#continued。

## UI

仪表盘：查看被监控服务的运行状态

拓扑图：以拓扑图的方式展现服务直接的关系，并以此为入口查看相关信息

追踪：以接口列表的方式展现，追踪接口内部调用过程

性能剖析：单独端点进行采样分析，并可查看堆栈信息

告警：触发告警的告警列表，包括实例，请求超时等。

自动刷新：刷新当前数据内容（我这好像没有自动刷新）

Global、Server、Instance、Endpoint不同展示面板，可以调整内部内容

- Services load：服务每分钟请求数
- Slow Services：慢响应服务，单位ms
- Un-Health services(Apdex):Apdex性能指标，1为满分。
- Global Response Latency：百分比响应延时，不同百分比的延时时间，单位ms
- Global Heatmap：服务响应时间热力分布图，根据时间段内不同响应时间的数量显示颜色深度

#### Service服务维度

Service Apdex（数字）:当前服务的评分

Service Apdex（折线图）：不同时间的Apdex评分

Successful Rate（数字）：请求成功率

Successful Rate（折线图）：不同时间的请求成功率

Servce Load（数字）：每分钟请求数

Servce Load（折线图）：不同时间的每分钟请求数

Service Avg Response Times：平均响应延时，单位ms

Global Response Time Percentile：百分比响应延时

#### Instance实例维度

Servce Instances Load：每个服务实例的每分钟请求数

Show Service Instance：每个服务实例的最大延时

Service Instance Successful Rate：每个服务实例的请求成功率

Service Instance Load：当前实例的每分钟请求数

Service Instance Successful Rate：当前实例的请求成功率

Service Instance Latency：当前实例的响应延时

JVM CPU:jvm占用CPU的百分比

JVM Memory：JVM内存占用大小，单位m

JVM GC Time：JVM垃圾回收时间，包含YGC和OGC

JVM GC Count：JVM垃圾回收次数，包含YGC和OGC

CLR XX：类似JVM虚拟机，这里用不上就不做解释了

#### Endpoint端点（API）维度

Endpoint Load in Current Service：每个端点的每分钟请求数

Slow Endpoints in Current Service：每个端点的最慢请求时间，单位ms

Successful Rate in Current Service：每个端点的请求成功率

Endpoint Load：当前端点每个时间段的请求数据

Endpoint Avg Response Time：当前端点每个时间段的请求行响应时间

Endpoint Response Time Percentile：当前端点每个时间段的响应时间占比

Endpoint Successful Rate：当前端点每个时间段的请求成功率

### DataSource展示栏

当前数据库：选择查看数据库指标

Database Avg Response Time：当前数据库事件平均响应时间，单位ms

Database Access Successful Rate：当前数据库访问成功率

Database Traffic：CPM，当前数据库每分钟请求数

Database Access Latency Percentile：数据库不同比例的响应时间，单位ms

Slow Statements：前N个慢查询，单位ms

All Database Loads：所有数据库中CPM排名

Un-Health Databases：所有数据库健康排名，请求成功率排名

## 拓扑图

1：选择不同的服务关联拓扑

2：查看单个服务相关内容

3：服务间连接情况

4：分组展示服务拓扑

：服务告警信息

：服务端点追踪信息

：服务实例性能信息

：api信息面板

## 追踪

-  左侧：api接口列表，红色-异常请求，蓝色-正常请求
-  右侧：api追踪列表，api请求连接各端点的先后顺序和时间

## 性能剖析

新建任务：新建需要分析的端点

左侧列表：任务及对应的采样请求

右侧：端点链路及每个端点的堆栈信息

新建任务

服务：需要分析的服务

端点：链路监控中端点的名称，可以再链路追踪中查看端点名称

监控时间：采集数据的开始时间

监控持续时间：监控采集多长时间

起始监控时间：多少秒后进行采集

监控间隔：多少秒采集一次

最大采集数：最大采集多少样本