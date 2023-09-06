---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
### 源码分析-发送Trace

考虑到减少**外部组件**的依赖，Agent 收集到 Trace 数据后，不是写入外部消息队列( 例如，Kafka )或者日志文件，而是 Agent 写入**内存消息队列**，**后台线程**【**异步**】发送给 Collector 。

我们先来看看 TraceSegmentServiceClient 的**属性**：

- `TIMEOUT` **静态**属性，发送等待超时时长，单位：毫秒。
- `lastLogTime` 属性，最后打印日志时间。该属性主要用于开发调试。
- `segmentUplinkedCounter` 属性，TraceSegment 发送数量。
- `segmentAbandonedCounter` 属性，TraceSegment 被丢弃数量。在 Agent 未连接上 Collector 时，产生的 TraceSegment 将被丢弃。
- `carrier` 属性，内存队列。
- `serviceStub` 属性，**非阻塞** Stub 。
- `status` 属性，连接状态。

**TracingContextListener**

这里先帮助看官老爷回顾一个点，TracingContext.finish() 方法在关闭 TraceSegment 的时候，会调用下面这行代码：

在 ListenerManager 中记录了所有 TracingContextListener 监听器：

在ListenerManager.notify() 方法中会遍历所有TracingContextListener监听器，通知他们该 TraceSegment 将会被关闭了：

**TraceSegmentServiceClient**

TraceSegmentServiceClient 实现了下面四个接口：

BootService

IConsumer

TracingContextListener

GRPCChannelListener

先来看看它的核心字段：

再来看TraceSegmentReportService 服务的 proto定义：

prepare() 方法会将当前TraceSegmentServiceClient对象作为 Listener 注册到 GRPCChannelManager 上，监听链接状态。当链接状态发生变化时，会通过 statusChanged() 方法（这是对GRPCChannelListener接口的实现）修改 status 字段值。

boot() 方法中会初始化上述核心字段，其中会创建 DataCarrier 对象，并调用其 consume() 方法启动 ConsumerThread 线程：

onComplete() 方法会将当前TraceSegmentServiceClient对象作为 Listener 注册到TracingContext.LISTENERS 集合中，easy，不展开说了。

再来看TraceSegmentServiceClient对 TracingContextListener 接口的实现，其 afterFinished() 方法会调用 DataCarrier.produce() 方法，将 TraceSegment 写入 DataCarrier 缓冲区暂存。

再来看 TraceSegmentServiceClient 对 IConsumer 接口的实现，在其 consume() 方法中会将 TraceSegment（从DataCarrier中读取到的）通过 gRPC 的方式发送到服务端，具体实现如下：

简单看一下GRPCStreamServiceStatus 的实现，其中封装了一个 volatile boolean 字段，来表示发送状态：

还提供了 wait4Finish() 方法用来等待整个发送过程结束：

最后，GRPCStreamServiceStatus.finish()方法会修改 status 的值为true，这样， wait4Finish() 方法也可以退出。

###### 发送Trace

XXXSegmentServiceClient有一个消费线程，不停的拉取缓冲队列中的数据，

- 如果是TraceSegmentServiceClient，则通过gRPC发送给后端OAP
- 如果是KafkaTraceSegmentServiceClient，则丢到Kafka中。