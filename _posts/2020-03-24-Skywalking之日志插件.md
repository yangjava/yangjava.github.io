------

# 1. 概述

本文主要分享 **traceId 集成到日志组件**，例如 log4j 、log4j2 、logback 等等。

我们首先看看**集成**的使用例子，再看看**集成**的实现代码。涉及代码如下：

- ![img](https://static.iocoder.cn/images/SkyWalking/2020_11_15/01.png)
- ![img](https://static.iocoder.cn/images/SkyWalking/2020_11_15/02.png)

本文以 **logback 1.x** 为例子。

# 2. 使用例子

1、**无需**引入相应的工具包，只需启动参数带上 `-javaagent:/Users/yunai/Java/skywalking/packages/skywalking-agent/skywalking-agent.jar` 。

2、在 `logback.xml` 配置 `%tid` 占位符：

![img](https://static.iocoder.cn/images/SkyWalking/2020_11_15/03.png)

3、使用 `logger.info(...)` ，会打印日志如下：

![img](https://static.iocoder.cn/images/SkyWalking/2020_11_15/04.png)

**注意**，traceId 打印到每条日志里，最终需要经过例如 ELK ，收集到日志中心。

# 3. 实现代码

## 3.1 TraceIdPatternLogbackLayout

[`org.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout`](https://www.iocoder.cn/SkyWalking/trace-id-integrate-into-logs/) ，实现 `ch.qos.logback.classic.PatternLayout` 类，实现支持 `%tid` 的占位符。代码如下：

- 第 33 行：添加 `tid` 的转换器为 [`org.skywalking.apm.toolkit.log.logback.v1.x.LogbackPatternConverter`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-application-toolkit/apm-toolkit-logback-1.x/src/main/java/org/skywalking/apm/toolkit/log/logback/v1/x/LogbackPatternConverter.java) 类。

## 3.2 LogbackPatternConverterActivation

[`org.skywalking.apm.toolkit.activation.log.logback.v1.x.LogbackPatternConverterActivation`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-logback-1.x-activation/src/main/java/org/skywalking/apm/toolkit/activation/log/logback/v1/x/LogbackPatternConverterActivation.java) ，实现 ClassInstanceMethodsEnhancePluginDefine 抽象类，定义了方法切面，代码如下：

![img](https://static.iocoder.cn/images/SkyWalking/2020_11_15/05.png)

------

[`org.skywalking.apm.toolkit.activation.log.logback.v1.x.PrintTraceIdInterceptor`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-logback-1.x-activation/src/main/java/org/skywalking/apm/toolkit/activation/log/logback/v1/x/PrintTraceIdInterceptor.java) ，实现 InstanceMethodsAroundInterceptor 接口，LogbackPatternConverterActivation 的拦截器。代码如下：

- [`#afterMethod(...)`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-logback-1.x-activation/src/main/java/org/skywalking/apm/toolkit/activation/log/logback/v1/x/PrintTraceIdInterceptor.java#L37) 方法，调用 `ContextManager#getGlobalTraceId()` 方法，使用全局链路追踪编号，而不是原有结果。



# Java Agent源码结构

- 链路数据模型抽象与封装

作用是封装链路数据协议约定的数据抽象，同时总体兼容了Opentracing规范。重点类有如下几个

- `org.apache.skywalking.apm.agent.core.context.trace.TraceSegment`
- `org.apache.skywalking.apm.agent.core.context.trace.TraceSegmentRef`
- `org.apache.skywalking.apm.agent.core.context.trace.EntrySpan`
- `org.apache.skywalking.apm.agent.core.context.trace.ExitSpan`
- `org.apache.skywalking.apm.agent.core.context.ids.DistributedTraceIds`

- 跨进程链路数据抽象与封装

重点类是`org.apache.skywalking.apm.agent.core.context.ContextCarrier`。即是对本文所讲解的协议规范的封装。

- 链路数据采集模型抽象与封装

重点类是`org.apache.skywalking.apm.agent.core.context.TracingContext`。TracingContext保存在ThreadLocal中，包含了链路中的所有数据，以及对数据的管控方法，主要是对Span的管控。同时提供对ContextCarrier数据的处理，包括：

- 将TraceSegment转换为ContextCarrier，即org.apache.skywalking.apm.agent.core.context.TracingContext#inject
- 从ContextCarrier数据中抽取TraceSegment数据，即org.apache.skywalking.apm.agent.core.context.TracingContext#extract

- 链路数据采集模块

重点是`org.apache.skywalking.apm.agent.core.context.ContextManager`类。ContextManager类是各种skywalking agent插件的枢纽。不同组件的skywalking插件，如mq，dubbo，tomcat，spring等skywalking agent插件均是通过调用ContextManager创建和管控TracingContext，TraceSegment，EntrySpan，ExitSpan，ContextCarrier等数据。可以说ContextManager管控着agent内的链路数据生命周期。

- 链路数据上传模块

重点是`org.apache.skywalking.apm.agent.core.remote`包中的`TraceSegmentServiceClient`类。
ContextManager采集节点内的链路数据片段（TraceSegment）后，通知TraceSegmentServiceClient将数据上报到Collector Server。其中涉及到
`TracingContextListener`与skywalking封装的`内存MQ`组件，后面会详细分析。

## Global(Distributed) Trace Id与Trace Segment Id

### 区别

skywalking中分别有两类全局唯一的id，即Global(Distributed) Trace Id与Trace Segment Id。Global(Distributed) Trace Id是指分布系统下的某条链路的唯一ID。Trace Segment Id是指链路所经过的分布系统服务节点事生成的链路片段ID。两者是一对多的关系。

### 生成流程

上述序列图表明，在实例化TracingContext时，会实例化TraceSegment，而在实例化TraceSegment时，同时生成了Global(Distributed) Trace Id与Trace Segment Id，源码如下：

```haxe
    public TraceSegment() {
        this.traceSegmentId = GlobalIdGenerator.generate();
        this.spans = new LinkedList<AbstractTracingSpan>();
        this.relatedGlobalTraces = new DistributedTraceIds();
        this.relatedGlobalTraces.append(new NewDistributedTraceId());
    }
```

- 生成trace segment id

其中，第1行代码`this.traceSegmentId = GlobalIdGenerator.generate();`，即是生成Trace Segment Id，调用了一次GlobalIdGenerator.generate()方法。
在第4行代码的`new NewDistributedTraceId()`创建了Global(Distributed) Trace Id。源码如下，可见同样是调用`GlobalIdGenerator.generate()`方法。

```scala
public class NewDistributedTraceId extends DistributedTraceId {
    public NewDistributedTraceId() {
        super(GlobalIdGenerator.generate());
    }
}
```

- 生成Global(Distributed) Trace Id

全局链路id的的逻辑较复杂，java agent中为这个id封装了`DistributedTraceIds`类，通过该类的append方法添加`NewDistributedTraceId `对象，表示一个Global(Distributed) Trace Id数据。NewDistributedTraceId是DistributedTraceId的子类，而DistributedTraceId类就是对ID类的包装，提供了一些工具方法。
`DistributedTraceIds`代表了相关链路id的集合，大多数情况下仅包含一条链路id。

### 关于GlobalIdGenerator

两个id均是调用同样的方法生成，即`org.apache.skywalking.apm.agent.core.context.ids.GlobalIdGenerator#generate`。
GlobalIdGenerator是生产全局ID字符串的逻辑封装类。下面看看`GlobalIdGenerator#generate`的源码。

```verilog
    public static ID generate() {
        if (RemoteDownstreamConfig.Agent.APPLICATION_INSTANCE_ID == DictionaryUtil.nullValue()) {
            throw new IllegalStateException();
        }
        IDContext context = THREAD_ID_SEQUENCE.get();

        return new ID(
            RemoteDownstreamConfig.Agent.APPLICATION_INSTANCE_ID,
            Thread.currentThread().getId(),
            context.nextSeq()
        );
    }
```

上述源码最终返回的即是根据协议规范封装的ID数据结构，第一个参数为当前应用的实例ID，第二个参数为当前线程号。第三个参数是通过IDContext.nextSeq()方法获取。而IDContext是从ThreadContext中获取的，所以IDContext是线程独有的。IDContext的源码比较好理解，IDContext.nextSeq()返回当前时间戳与线程内的自增序号组合的字符串。

## 总结

综上，在新链路开始，生成TraceSegment实例时，第一次调用`GlobalIdGenerator.generate()`作为`Trace Segment Id`，第二次调用`GlobalIdGenerator.generate()`作为`Global(Distributed) Trace Id`。