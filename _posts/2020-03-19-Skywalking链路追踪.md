---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
## Skywalking链路追踪
skywalking通过使用字节码增强技术实现对特定方法的监控，并收集数据用于分析。

## 分布式调用链标准（OpenTracing）
OpenTracing最初是由Ben Sigelman在2016年发布的一篇论文“Distributed Tracing with OpenTracing”。该论文提出了分布式追踪的概念，并阐述了OpenTracing的设计目标、基本原则和实现方式。
论文指出，分布式追踪的核心挑战在于需要在多个服务之间建立上下文，并跨越多个异步和同步调用。OpenTracing通过使用上下文传递和操作ID来解决这个问题，并提供了一种可扩展的方式来生成和收集跟踪数据。

OpenTracing 是一个中立的分布式追踪的 API 规范，提供了统一接口方便开发者在自己的服务中集成一种或者多种分布式追踪的实现，使得开发人员能够方便的添加或更换追踪系统的实现。OpenTracing 可以解决不同的分布式追踪系统 API 不兼容的问题，各个分布式追踪系统都来实现这套接口。
OpenTracing 的数据模型，主要有以下三个：
- Trace：可以理解为一个完整请求链路，也可以认为是由多个 span 组成的有向无环图（DAG）；
- Span：span 代表系统中具有开始时间和执行时长的逻辑运行单元，只要是一个完整生命周期的程序访问都可以认为是一个 span，比如一次数据库访问，一次方法的调用，一次 MQ 消息的发送等；每个 span 包含了操作名称、起始时间、结束时间、阶段标签集合（Span Tag）、阶段日志（Span Logs）、阶段上下文（SpanContext）、引用关系（Reference）；
- SpanContext： Trace 的全局上下文信息，span 的状态通过 SpanContext 跨越进程边界进行传递，比如包含 trace id，span id，Baggage Items（一个键值对集合）。

TraceId是这个请求的全局标识。内部的每一次调用就称为一个 Span，每个 Span 都要带上全局的 TraceId，这样才可把全局 TraceId 与每个调用关联起来。这个 TraceId 是通过 SpanContext 传输的，既然要传输，显然都要遵循协议来调用。如果我们把传输协议比作车，把 SpanContext 比作货，把 Span 比作路应该会更好理解一些。

## Skywalking链路模型
Skywalking链路模型主要如下：
- Trace
表示一个完整调用链，包括跨进程、跨线程的所有`Segment`的集合。也可以认为是由多个`span`组成的有向无环图(DAG)

- Segment
在`Skywalking`的设计中，在`Trace`级别和`Span`级别之间加了一个`Segment`的概念，表示一个进程（JVM）或线程内的所有操作的集合，即包含若干个`Span`。

- Span
`Span`代表系统中具有开始时间和执行时长的逻辑运行单元，只要是一个完整生命周期的程序访问都可以认为是一个`Span`，比如一次数据库访问，一次方法的调用，一次 MQ 消息的发送等。
每个`Span`包含了操作名称、起始时间、结束时间、阶段标签集合（Span Tag）、阶段日志（Span Logs）、阶段上下文（SpanContext）、引用关系（Reference）。

Skywalking 中的Span分为三类：
  - EntrySpan
入栈Span。表示服务端的入口。Segment的入口，一个Segment有且仅有一个Entry Span，比如HTTP或者RPC的入口，或者MQ消费端的入口等。
  - LocalSpan
通常用于记录一个本地方法的调用。
  - ExitSpan
出栈Span。Segment的出口，一个Segment可以有若干个Exit Span，比如HTTP或者RPC的出口，MQ生产端，或者DB、Cache的调用等。

- SpanContext
Trace 的全局上下文信息，span 的状态通过 SpanContext 跨越进程边界进行传递，比如包含 trace id，span id，Baggage Items（一个键值对集合）。

## 上下文载体ContextCarrier
为了实现分布式追踪, 需要绑定跨进程的追踪, 并且上下文应该在整个过程中随之传播. 这就是 ContextCarrier 的职责。

以下是有关如何在 A -> B 分布式调用中使用 ContextCarrier 的步骤。
- 在客户端, 创建一个新的空的 ContextCarrier。
- 通过 ContextManager#createExitSpan 创建一个 ExitSpan 或者使用 ContextManager#inject 来初始化 ContextCarrier。
- 将 ContextCarrier 所有信息放到请求头 (如 HTTP HEAD), 附件(如 Dubbo RPC 框架), 或者消息 (如 Kafka) 中
- 通过服务调用, 将 ContextCarrier 传递到服务端。
- 在服务端, 在对应组件的头部, 附件或消息中获取 ContextCarrier 所有内容。
- 通过 ContextManager#createEntrySpan 创建 EntrySpan 或者使用 ContextManager#extract 来绑定服务端和客户端。

让我们通过 Apache HttpComponent client 插件和 Tomcat 7 服务器插件演示, 步骤如下:

客户端 Apache HttpComponent client 插件
```
   ContextCarrier contextCarrier = new ContextCarrier();
   span = ContextManager.createExitSpan("/span/operation/name", contextCarrier, "ip:port");
   CarrierItem next = contextCarrier.items();
   while (next.hasNext()) {
       next = next.next();
       httpRequest.setHeader(next.getHeadKey(), next.getHeadValue());
   }
```

服务端 Tomcat 7 服务器插件
```
    ContextCarrier contextCarrier = new ContextCarrier();
    CarrierItem next = contextCarrier.items();
    while (next.hasNext()) {
        next = next.next();
        next.setHeadValue(request.getHeader(next.getHeadKey()));
        }

    span = ContextManager.createEntrySpan(“/span/operation/name”, contextCarrier);
```

## 上下文快照ContextSnapshot
除了跨进程, 跨线程也是需要支持的, 例如异步线程（内存中的消息队列）和批处理在 Java 中很常见. 跨进程和跨线程十分相似, 因为都是需要传播上下文. 唯一的区别是, 跨线程不需要序列化.

以下是有关跨线程传播的三个步骤：
- 使用 ContextManager#capture 方法获取 ContextSnapshot 对象.
- 让子线程以任何方式, 通过方法参数或由现有参数携带来访问 ContextSnapshot
- 在子线程中使用 ContextManager#continued

## 上下文管理器 (ContextManager)
ContextManager 提供所有主要 API.

创建 EntrySpan
```
public static AbstractSpan createEntrySpan(String endpointName, ContextCarrier carrier)
```
根据操作名称(例如服务名称, uri) 和 上下文载体 (ContextCarrier) 创建 EntrySpan.

创建 LocalSpan
```
public static AbstractSpan createLocalSpan(String endpointName)
```
根据操作名称(例如完整的方法签名)创建 (e.g. full method signature)

创建 ExitSpan
```
public static AbstractSpan createExitSpan(String endpointName, ContextCarrier carrier, String remotePeer)
```
根据操作名称(例如服务名称, uri), 上下文载体 (ContextCarrier) 以及对等端 (peer) 地址 (例如 ip + port 或 hostname + port) 创建 ExitSpan.







#### **Trace基本概念**

在开始介绍发送 Trace 的具体实现之前，先简单说一下 Trace 相关的基本概念，要是客官们已经了解了这些概念，可以跳过这部分。读者也可以参考一下《OpenTracing语义标准》(https://github.com/opentracing-contrib/opentracing-specification-zh/blob/master/specification.md) 。

OpenTracing中的一条Trace(调用链)可以被认为是一个由多个Span组成的有向无环图(DAG图)，Span 与 Span 的关系被命名为 References。Span 可以被理解为一次方法调用、一次 RPC 或是一次 DB 访问。

每个 Span 包含以下的状态:

An operation name，操作名称

A start timestamp，起始时间

A finish timestamp，结束时间

Span Tag，一组键值对构成的Span标签集合。键值对中，键必须为string，值可以是字符串，布尔，或者数字类型。

Span Log，一组span的日志集合。每次log操作包含一个键值对，以及一个时间戳。键值对中，键必须为string，值可以是任意类型。但是需要注意，不是所有的支持OpenTracing的Tracer,都需要支持所有的值类型。

SpanContext，Span上下文对象 (下面会详细说明)

References(Span间关系)，相关的零个或者多个Span（Span间通过SpanContext建立这种关系）

每一个 SpanContext 包含以下状态：

任何一个 OpenTracing 的实现，都需要将当前调用链的状态（例如：trace id 和 span id），依赖一个独特的 Span 去跨进程边界传输

Baggage Items，Trace 的随行数据，是一个键值对集合，它存在于 Trace 中，也需要跨进程边界传输

Span间关系

一个 Span 可以与一个或者多个**SpanContexts**存在因果关系。OpenTracing 目前定义了两种关系：（父子） 和（跟随）。这两种关系明确的给出了两个父子关系的Span的因果模型。

**ChildOf 引用：**一个 Span 可能是一个父级 Span 的孩子，即"ChildOf"关系。在 ChildOf 引用关系下，父级 Span 某种程度上取决于子 Span。下面这些情况会构成 ChildOf 关系：

一个 RPC 调用的服务端的 Span，和RPC服务客户端的 Span 构成 ChildOf 关系。

一个 sql insert 操作的 Span，和 ORM 的 save 方法的 Span 构成 ChildOf 关系。

很多 Span 可以并行工作（或者分布式工作）都可能是一个父级的 Span 的子项，它会合并所有子 Span 的执行结果，并在指定期限内返回。

**FollowFrom 引用：**个人觉得不是很常见，不说了。

回到 Skywalking ，在 Skywalking 的设计中，在 Trace级别 和 Span 级别之间加了一个 Segment 的概念，用于表示一个 OS 里面的 Span 集合。



Skywalking 中具体的 Span 实现在后面分析收集 Trace 的小节中再详细分析，这里先不深入分析了。

**TraceSegment**

TraceSegment 是一条 Trace 的一段，TraceSegment 用于记录当前线程的 Trace 信息。分布式系统基本会涉及跨线程的操作，例如， RPC、MQ等，怎么也要有 DB 访问吧，╮(╯_╰)╭，所以一条 Trace 基本都是由多个 TraceSegment 构成的。

我们常见的 RPC 调用啊、Http 调用啊之类的，每个 TraceSegment 只有一个 parent。但是当一个 Consumer 批量处理 MQ 消息的时候，其中的每条消息都来自不同的 Producer ，就会有多个 parent 了，同时这个 TraceSegment 也就属于多个 Trace 了。

traceSegmentId字段的类型是ID，它由三个long类型的字段(part1、part2、part3)构成，分别记录了 service_instance_id、线程Id、Context生成序列。

Context生成序列的格式是：

TraceSegment Id 的最终格式是：

再来是GlobalIdGenerator，它是用来生成 ID 对象的，直接看它的 generate() 方法：

**DistributedTraceId**

DistributedTraceId 用于生成全局的 TraceId，其中封装了一个 ID 类型的字段。DistributedTraceId 是个抽象类，它有两个实现类，如下图所示：

其中 NewDistirbutedTraceId 负责为新 Trace 生成编号，在请求进入我们的系统、第一次创建 TraceSegment 对象的时候，会创建NewDistirbutedTraceId对象，在其构造方法内部会调用 GlobalIdGenerator.generate() 方法生成创建 ID 对象。

PropagatedTraceId 负责处理 Trace 传播过程中的 TraceId，PropagatedTraceId的构造方法接收一个 String 类型的 TraceId ，解析之后得到 ID 对象。

TraceSegment中的 relatedGlobalTraces字段是DistributedTraceIds 类型，它的底层封装了一个 LinkedList 集合，用于记录当前 TraceSegment 关联的 TraceId。为什么是一个集合呢？与上面提到的，一个 TraceSegment 可能有多个 parent 的情况一样。

**Span**

TraceSegment 是由多个 Span 构成的，下图是 Span 的继承关系：

```
|-AsyncSpan
	|-AbstractSpan
		|-AbstractTracingSpan
			|-StackBasedTracingSpan
					|-ExitSpan
					|-EntrySpan
			|-LocalSpan
		|-NoopSpan
```

从顶层开始看呗，AsyncSpan 接口定义了一个异步 Span 的基本行为：

prepareForAsync()方法：当前 Span 在当前线程结束了，但是当前 Span 未被彻底关闭，依然是存活的。

asyncFinish()方法：当前Span 真正关闭。这与 prepareForAsync() 方法成对出现。

这两个方法在异步 RPC 中会见到。

AbstractSpan 也是一个接口，其中定义了 Span 的基本行为：

getSpanId() 方法:获得当前 Span 的编号，Span 编号是一个整数，在 TraceSegment 内唯一，从 0 开始自增，在创建 Span 对象时生成。

setOperationName()/setOperationId() 方法：设置操作名/操作编号。这两个方法是互斥的，在 AbstractTracingSpan 这个实现中，有 operationId 和 operationName 两个字段，只能有一个字段有值。

setComponent() 方法：设置组件。它有两个重载，在 AbstractTracingSpan 这个实现中，有 componentId 和 componentName 两个字段，两个重载分别用于设置这两个字段。在 ComponentsDefine 中可以找到 Skywalking目前支持的组件。

setLayer() 方法：设置 SpanLayer，也就是当前 Span 所处的层次。SpanLayer 是个枚举，可选项有 DB、RPC_FRAMEWORK、HTTP、MQ、CACHE。

tag(AbstractTag, String) 方法：为当前 Span 添加键值对的标签。一个 Span 可以投多个标签，AbstractTag 中就封装了 String 类型的 Key ，没啥可说的。

log() 方法：记录当前 Span 中发生的关键日志，一个 Span 可以包含多条日志。

start() 方法：开始 Span 。其实就是设置当前 Span 的开始时间以及调用层级等信息。

isEntry() 方法：当前是否是入口 Span。

isExit() 方法：当前是否是出口 Span。

ref() 方法：设置关联的 TraceSegment 。

AbstractTracingSpan 实现了 AbstractSpan 接口，其中定义了一些 Span 的基础字段，其中很多字段一眼看过去就知道是啥意思了，就不一一展开介绍了，其中有一个字段：

AbstractTracingSpan 中的方法也比较简单，基本都是 getter/setter 方法，其中有一个 finish() 方法，会更新 endTime 字段并将当前 Span 记录到给定的 TraceSegment 中。

StackBasedTracingSpan 这种 Span 可以多次调用 start() 方法和 end() 方法，就类似一个栈。其中多了两个字段：

StackBasedTracingSpan.finish() 方法会在栈彻底退出的时候，才会将当前 Span 添加到 TraceSegment 中：

EntrySpan 表示的是一个服务的入口 Span，主要用在服务提供方的入口，例如，Dubbo Provider、Tomcat、Spring MVC 等等。EntrySpan 是 TraceSegment 的第一个 Span ，这也是为什么称之为"入口" Span 的原因。那么为什么 EntrySpan 继承 StackBasedTracingSpan？从前面对 Skywalking Agent 的分析来看，Agent 只会根据插件在相应的位置

对方法进行增强，具体的增强逻辑就包含创建 EntrySpan 对象(后面在分析具体插件实现的时候，会看到具体的实现代码)，例如，Tomcat插件 和 Spring MVC 插件。很多 Web 项目会同时使用这两个插件，难道一个 TraceSegment 要有两个 EntrySpan 吗？

显然不合适，所以 EntrySpan 继承了 StackBasedTracingSpan，当请求经过 Tomcat 插件的时候，会创建 EntrySpan，当请求经过 Spring MVC 插件的时候，不会再创建新的 EntrySpan 了，只是 currentMaxDepth 字段加 1。

currentMaxDepth 字段是 EntrySpan 中用于记录当前 EntrySpan 的深度的，前面介绍StackBasedTracingSpan.finish() 方法代码时看到，只有 stackDepth 为 1 的时候，才能结束当前 Span。

EntrySpan 要关注的是其 start() 方法：

虽然 EntrySpan 是在第一个增强逻辑中创建的，但是后续每次 start()方法都会清空所有字段，所以 EntrySpan 除了 startTime 和 endTime 以外的字段都是以最后一次调用 start() 方法写入的为准。在 EntrySpan 中的 set* 方法会检测currentMaxDepth是否为最底层，如果不是，设置相关字段没有什么意义，例如 tag()方法：

ExitSpan 表示的是出口 Span，主要用于服务的消费者，例如，Dubbo Consumer、HttPClient等等。如果在一个调用栈里面出现多个插件创建 ExitSpan，则只会在第一个插件中创建 ExitSpan，后续调用的 ExitSpan.start() 方法并不会更新 startTime，其他的 set*() 方法也会做判断，只有 stackDepth 为1的时候，才会写入相应字段，也就是说，ExitSpan 中记录的信息是创建 ExitSpan 时填入的，与 EntrySpan 正好相反。

举个栗子，假如有一次通过 Http 方式进行的 Dubbo 调用，Dubbo A --> HttpClient --> Dubbo B，此时在 Dubbo A 的出口处，Dubbo 的插件会创建 ExitSpan 并调用 start() 方法，在 HttpClient 的出口处则只是再次调用了 start() 方法，该 ExitSpan 中记录的信息都是Dubbo A 出口处记录的。

一个 TraceSegment 可以有多个 ExitSpan，例如，Dubbo A 服务在处理一个请求时，会调用 Dubbo B 服务得到相应之后，紧接着调用了 Dubbo C 服务，这样，该 TraceSegment 就有了两个完全独立的 ExitSpan。

LocalSpan 表示的是一个本地方法调用，继承了 AbstractTracingSpan，没啥可说的，也不能递归，╮(╯_╰)╭。

行吧，Span 核心的内容就说到这里，继续往下看。

**TraceSegmentRef**

TraceSegment 通过 refs 集合记录父 TraceSegment 中的一个 Span，TraceSegmentRef 中的核心字段如下：

其中最重要的还是traceSegmentId 字段和spanId字段。

**总结**

好了，Trace 的基本概念以及 Skywalking 中的基础组建类，都大概介绍完了，主要就是：TraceSegment、ID、DistributedTraceId、Span、TraceSegmentRef。这些组建中的字段也并不复杂，都是为了确定从属关系( Span 属于 TraceSegment )以及父子关系( Span的父子关系、TraceSegment 的父子关系)。这些组建中的方法也都比较简单，基本都是getter/setter方法，没有超过10行的哈，easy，easy。

#### Trace Context

从这一小节开始，我们将开始介绍 Trace Context 相关的组件。

**Context**

AbstractTracerContext 是 Skywalking 抽象链路上下文的顶层抽象类，在 一个线程中 Context 与 TraceSegment 一一对应，其中定义了链路上下文的的基本行为：

TracingContext 是 AbstractTracerContext 实现类，其核心方法如下：

下面开始分析 TracingContext 中的核心方法，首先是 createEntrySpan() 方法，它负责创建 EntrySpan，如果已经存在父 Span，当然没法再创建 EntrySpan咯，重新调用一下 start()方法咯，╮(╯_╰)╭，大致实现如下（其中省略了 DictionaryManager 的相关代码，后面再详细介绍这货）：

接下来看 createLocalSpan()方法，就是创建个 LocalSpan对象，然后加到 activeSpanStack 集合中，大致实现如下：

再往下自然就到了 createExitSpan() 方法，创建或是重用 ExitSpan，实现如下：

最后来看 stopSpan() 方法，它负责关闭指定的 Span对象：

在 TracingContext.finish() 方法中会关闭关联的 TraceSegment 对象，完成采样操作，还会通知相关的 Listener：

在开始介绍 inject() 方法和 extract() 方法之前，需要先介绍一下他们的参数——ContextCarrier，它记录了当前 TracingContext的一些基本信息，并实现了Serializable 接口哦，ContextCarrier 中的字段不再详细介绍，望文生义即可，嘎嘎嘎。

TracingContext.inject() 方法的功能就是将 TracingContext 对象填充到 ContextCarrier 中，看着很长，其实就是一些简单的字段，实现如下：

在 TracingContext 的 extract() 方法就是将ContextCarrier中各个字段，解出来放到 TraceSegmentRef 的相应字段中：

下面看一下 capture() 方法和 continued() 方法的参数—— ContextSnapshot，它与前面的 ContextCarrier 类似。因为只是跨线程传递，所以像 service_intance_id 这种字段就没必要传递了。capture() 方法和 continued() 方法的行为与 inject() 方法和 extract() 方法类似，这里不再展开分析了。

**ContextManager**

在前面介绍 ServiceManager SPI 加载 BootService 接口实现类的时候，我们看到有一个叫 ContextManager 的实现类，就是它来控制 Context 的。ContextManager 中的 prepare()、boot()、onComplete() 方法都是空实现。

ContextManager 的核心字段如下：

在 ContextManager 提供的 createEntrySpan()、createExitSpan() 以及 createLocalSpan()等方法中，都会调用其 getOrCreate() 这个 static 方法，在该方法中负责创建 TracingContext，如下所示：

之后会再调用TracingContext的相应方法，创建指定类型的 Span。

ContextManager 中的其他方法都是先调用 get() 这个静态方法拿到当前线程的 TracingContext，然后调用 TracingContext 的相应方法实现的，不展开了╮(╯_╰)╭。

**SamplingService**

前面的分析中多次看到 SamplingService，它负责进行采样，其核心字段如下：

在 SamplingService.boot()方法中会根据配置初始化上述字段：

在 SamplingService.trySampling()方法中会尝试增加 samplingFactorHolder 的值，当其值超过配置指定的值时，会返回false，表示该 Trace 未被采样到，具体实现如下：

**总结**

Span的Context记录分两种：

- ContextCarrier -- 用于跨进程传递上下文数据。
- ContextSnapshot -- 用于跨线程传递上下文数据。

好了，Context 相关的组件基本就介绍完了，其中包括 AbstractTracerContext 及其实现类、ContextManager，主要提供了如下方法：

创建 EntrySpan、LocalSpan、ExitSpan 三类Span 的方法。

关闭 Span 的 stopSpan() 方法。

用于跨进程传播的 inject()、extract() 方法，以及涉及到的 ContextCarrier 组件

用于跨线程传播的 capture()、continue()方法，以及涉及到的 ContextSnapshot组件。

#### DictionaryManager

上一节介绍了 Context 相关的组件，Skywalking Agent 会将 Context 中的数据发到 Skywalking 的服务端，其中有 operationName、peerHost、tag KV 等等一堆字符串，每次都传递一堆字符串会很浪费带宽的。常见的解决方案就是将字符串映射成数字，然后传输一组数字即可，Skywalking 也是这么搞得，涉及到的组件是 DictionaryManager，也会本节介绍的重点。

Skywalking 中有两个 DictionaryManager，一个是EndpointNameDictionary ，另一个NetworkAddressDictionary，注意，这俩货都通过枚举的方式实现了单例哈。

**EndpointNameDictionary**

本小节先来看EndpointNameDictionary，其中封装了两个集合：

OperationNameKey中包含四个字段，其 equals()方法会将这四个字段都考虑进去，如下所示：

EndpointNameDictionary中的核心方法是 find0() 方法，它负责查找指定的 OperationNameKey，查找成功就返回 Found，查找失败就记录到unRegisterEndpoints集合等待同步：

syncRemoteDictionary() 方法会定期将 unRegisterEndpoints集合同步到服务端，服务端会返回分配的映射id，并记录到endpointDictionary 集合：

这个 gRPC 服务定义在哪里呢？回到前面 skywalking-agent 的注册过程——Register.proto，之前只介绍了doServiceRegister和doServiceInstanceRegister，现在看其中的doEndpointRegister：

其中 Endpoints 参数的定义如下，与 OperationNameKey 中的字段同款：

返回值 EndpointMapping 的定义如下，其中的 endpointId 就是服务端为该 OperationName 分配的 id：

**NetworkAddressDictionary**

NetworkAddressDictionary 与 EndpointNameDictionary的功能类似，实现了 networkAddress 与数字之间的映射，其核心字段如下：

NetworkAddressDictionary 与服务端定期同步的方法是syncRemoteDictionary()方法，具体实现如下：

在前面介绍 skywalking-agent 初始化的时候，Agent 会定期向服务端发送心跳，在心跳发送完之后，就会调用 EndpointNameDictionary、NetworkAddressDictionary 的syncRemoteDictionary() 方法进行同步，看看代码ServiceAndEndpointRegisterClient 的153行和154行就知道了，嘎嘎嘎。

**PossibleFound**

最后，在EndpointNameDictionary、NetworkAddressDictionary 中提供的 find*()方法的返回值都是 PossibleFound 类型，它是一个抽象类，表示是否找到了对应的id：

PossibleFound 提供了 两个重载的 doInCondition()方法，根据查找结果执行不同的行为，直接上代码吧：

Found 接口以及 FoundAndObtain 接口在前面介绍 TracingContext 以及 StackBasedTracingSpan 中都有实现（只不过我给省略了，嘎嘎嘎），你可以自己翻一下，这几个接口的定义如下：

这里以 StackBasedTracingSpan.setPeer()方法的实现为例，简单看一下 Found 和 NotFound 接口的使用：

### 源码分析-Tomcat 插件

本节通过分析几个插件的实现，深入了解一下 ContextManager、TracingContext、TraceSegment 这些组建是怎么玩的，了解一下 skywalking-agent 中的数据流向是什么啥样的。首先是 Tomcat 插件，Skywalking 提供的 Tomcat 插件本身比较简单，但是要看懂这个插件的原理，需要对 Tomcat 本身的结构有一些了解。

**Tomcat 架构简析**

先来简单看一下的 Tomcat 的架构，如下图所示

Connector 组件是 Tomcat 中两个核心组件之一，它的主要任务是负责接收客户端发起的 TCP 连接请求（其实就是创建相应的 Request 和 Response 对象），而请求的处理则是由 Container 来负责的。

Container 是容器的父接口，所有子容器都必须实现这个接口，Container 容器的设计用的是典型的责任链模式，它有四个子容器组件构成，分别是：Engine、Host、Context、Wrapper，这四个组件不是平行的，而是父子关系，Engine 包含 Host，Host 包含 Context，Context 包含 Wrapper。通常一个 Servlet class 对应一个 Wrapper，如果有多个 Servlet 就可以定义多个 Wrapper，如果有多个 Wrapper 就要定义一个更高的 Container 了，如 Context。Context 还可以定义在父容器 Host 中，Host 不是必须的，但是要运行 war 程序，就必须要 Host，因为 war 中必有 web.xml 文件，这个文件的解析就需要 Host 了。如果要有多个 Host 就要定义一个顶级容器 Engine 了。而 Engine 没有父容器了，一个 Engine 代表一个完整的 Servlet 引擎。这些组件在 Tomcat 的 server.xml 文件中都能找到相应的配置，用过 Tomcat 的童鞋都知道，不解释了。

下面这张图大致展示了从 Connector 开始接收请求，然后请求一步步经过 Engine、Host、Context、Wrapper，最终 Servlet 的流程

其实容器的本质是一个 Pipeline，我们可以在这个Pipeline 上增加任意的 Valve，处理请求 Tomcat 线程会挨个执行这些 Valve 最终完成请求的处理，而且四个组件都会有自己的一套 Valve 集合，例如上图中的 StandEngineValve、StandHostValve、StandContextValve、StandWrapperValve，我们也可以在 Tomcat 的 server.xml 文件中自定义Valve（实际工作中只会撸业务代码，很少有人这么玩）。这些标准的 Valve 都是当前 Container 中最后一个 Valve，它们会负责将请求传给它们的子容器，以保证处理逻辑能继续向下执行。看下面这张图就比较明确了哈

接着来看四个级别的容器分别是干啥的。

**Engine**作为顶层容器，接口比较简单，它只定义了一些基本的关联关系，它可以添加 Host 类型的子容器，没啥可说的。

一个**Host**在 Engine 中代表一个虚拟主机，这个虚拟主机的作用就是运行多个应用，它负责安装和展开这些应用，并且标识这个应用以便能够区分它们。它的子容器通常是 Context，它除了关联子容器外，还有就是保存一个主机应该有的信息。

**Context**代表 Servlet 的 Context，它具备了 Servlet 运行的基本环境，理论上只要有 Context 就能运行 Servlet 了，也就是说 Tomcat 可以没有 Engine 和 Host。Context 最重要的功能就是管理它里面的 Servlet 实例，并和 Request 一起正确地找到处理请求的 Servlet。Servlet 实例在 Context 中是以 Wrapper 出现的。

**Wrapper**代表一个 Servlet，它负责管理一个 Servlet，包括的 Servlet 的装载、初始化、执行以及资源回收。Wrapper 是最底层的容器，它没有子容器了。

**Tomcat 插件分析**

来看tomcat-7.x-8.x-plugin 这个插件，这是 Skywalking 提供给 Tomcat 7 和 Tomcat 8的插件。在其 skywalking-plugin.def 文件中定义了TomcatInstrumentation 和 ApplicationDispatcherInstrumentation 插件类。

先看 TomcatInstrumentation.enhanceClass()方法，确定它拦截的是 Tomcat 中的哪个类呢？

StandardHostValve 是 Host容器中最后一个Valve，核心方法是 invoke() 方法，该方法中会通过 request 找到匹配的 Context 对象，并调用其 Pipeline 中的第一个 Valve处理请求，大致实现如下所示：

接着看 TomcatInstrumentation.getInstanceMethodsInterceptPoints()方法，它返回了两个 InstanceMethodsInterceptPoint 对象，一个拦截 invoke()方法，一个拦截 throwable()方法。

先来看拦截 invoke()方法的 TomcatInvokeInterceptor，它的 beforeMethod() 方法核心就是创建 EntrySpan，大致上线如下：

CarrierItem 有三个核心字段：

这样，CarrierItem既可以存储键值对，也可以串成一个链表，好吧，CarrierItem 还真实现了 Iterator 接口。

在 ContextCarrier.items() 方法中，会根据当前 Skywalking Agent 的配置创建一个CarrierItem链表：

从TomcatInvokeInterceptor.beforeMethod()方法的逻辑中可以看到，之后从 Http 请求的 Header 中获取对应的 value 值记录到对应 CarrierItem 中。setHeadValue() 方法会调用 ContextCarrier.deserialize() 方法解析该 value 值并初始化 ContextCarrier 中的各个字段，这里以 SW6CarrierItem 为例进行介绍（这里省略一些try/catch代码块和边界检查）：

这就填满了 ContextCarrier 的 8 个字段咯，╮(╯_╰)╭ 。

接下来看，ContextManager.createEntry() 方法的实现，前面说过其核心是调用 getOrCreate() 方法获取/创建当前 TracingContext 对象，然后调用 TracingContext.createEntry() 方法创建（或是重新 start ）当前 EntrySpan 对象，这里更详细的说一下一些实现细节吧。

ContextManager.createEntry() 方法首先会检测当前 ContextCarrier 是否合法，其实就是检查ContextCarrier的8个核心字段是否填充好了，如果合法，就证明是上游有 Trace 信息传递下来了：

行吧，TomcatInvokeInterceptor 的 beforeMethod()方法大概就是这样，它的 afterMethod()方法就简单很多了：

TracingContext.stopSpan() 方法的具体在前面已经详细分析过了，其中会调用StackBasedTracingSpan.finish() 方法尝试关闭当前 Span，这里会检测该 Span 的 operationId 字段，如果为空，则尝试再次通过 DictionaryManager组件用 operationName 换取 operationId，具体代码就不贴了。

接下来看tomcat-7.x-8.x-plugin 中的另一个插件类——ApplicationDispatcherInstrumentation，它拦截的是 Tomcat 的 ApplicationDispatcher.forward()方法以及ApplicationDispatcher 的全部构造方法。forward()方法主要处理 forward 跳转，写过 JSP 和 Servlet 程序的童鞋应该都知道 forward 和 redirect 的知识点，不展开说了。

ForwardInterceptor的实现比较简单，其onConstruct()方法如下：

beforeMethod()方法和 afterMethod()方法的实现大致如下：

到这里，tomcat-7.x-8.x-plugin 插件的具体实现就分析完了。

### 源码分析-**Dubbo 插件分析**

要搞清楚 Skywalking 提供的 Dubbo 插件的工作原理，需要先了解一下 Dubbo 中的 Filter 机制。Filter 在很多框架中都有使用过这个概念，基本上的作用都是类似的，在请求处理前后做一些通用的逻辑，而且Filter可以有多个，支持层层嵌套。

Dubbo 的Filter 概念基本上符合我们正常的预期理解，而且 Dubbo 官方针对 Filter 做了很多的原生支持，包括我们熟知的 RpcContext、accesslog、monitor 功能都是通过 Filter 来实现的。Filter 也是 Dubbo 用来实现功能扩展的重要机制，我们可以通过自定义 Filter 实现、启停指定 Filter 来改变 Dubbo 的行为来实现需求。

行吧，简单看一下Dubbo Filter 相关的知识点。首选是构造DubboFilter Chain的入口是在 ProtocolFilterWrapper.buildInvokerChain() 方法处，它将加载到的 Dubbo Filter 实例串成一个 Chain（这里）：

这里的核心有两步，上面明显能看出来的是将 Filter 实例串成 Chain，另一个核心步骤就是通过ExtensionLoader 加载 Filter 对象，原理是SPI，但是 Dubbo 的 SPI 实现有点优化，但是原理和思想基本一样，看一下实现吧：

介绍完 Dubbo Filter 的原理之后，我们来看 MonitorFilter 实现，它用于记录一些监控信息，来看看它的 invoke() 方法的实现：

有个地方记录开始时间、增加并发量，必然有个地方计算请求耗时、减掉并发量，在哪里呢？在 MonitorListener 里面，它是 MonitorFilter 配套的 Listener，其 invoke() 方法实现如下：

在 collect()方法里面，会将监控信息整理成 URL 并缓存起来：

DubboMonitor 实现了 Monitor 接口，其中有个 Map用于缓存 URL，然后在其构造方法中会启动一个定时任务，定时发送 URL：

在 DubboMonitor.collect()方法中会将相同的 URL 中的监控值累加，然后形成一个新的 URL 填充回statisticsMap 集合，这里就不展开介绍了。

行了，Dubbo MonitorFilter 的基础知识介绍完了，开始 Skywalking Dubbo 插件的分析吧（终于开始正题了）。apm-dubbo-2.7.x-plugin 插件的 skywalking-plugin.def 中定义的插件是 DubboInstrumentation，它拦截的是 MonitorFilter.invoke()方法，具体的 Interceptor 实现是 DubboInterceptor，在其 beforeMethod() 方法中会根据当前 MonitorFilter 所在的服务角色（Consumer/Provider）创建对应的 Span（ExitSpan/EntrySpan），具体实现如下：

在 ContextManager.createExitSpan()方法中除了创建 ExitSpan 之外，还会调用 inject() 方法将 Trace 信息记录到CarrierContext 中，这样后面通过CarrierItem 持久化的时候才有值。

DubboInterceptor.afterMethod() 方法实现比较简单，有异常就是通过 log 方式记录到当前 Span，最后尝试关闭当前 Span。