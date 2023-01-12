---
layout: post
categories: [OpenTelemetry,Trace]
description: none
keywords: OpenTelemetry
---
# OpenTelemetry简介
OpenTelemetry 的自身定位很明确：数据采集和标准规范的统一，对于数据如何去使用、存储、展示、告警，官方是不涉及的

OpenTelemetry 要解决的是对可观测性的大一统，它提供了一组 API 和 SDK 来标准化遥测数据的采集和传输，opentelemetry 并不想对所有的组件都进行重写，而是最大程度复用目前业界在各大领域常用工具，通过提供了一个安全，厂商中立的能用协议、组件，这样就可以按照需要形成 pipeline 将数据发往不同的后端。

架构图
image.png
receivers(接收者): 定义从 client 端的数据要以何种数据模型进行接收, 支持很多种数据模型.
processors: 将 receivers 的数据进行某些处理，比如批量、性能分析等.
exporters: 将 processors 后的数据导出到特定的后端，比如 metrics 数据存储到 prometheus 中.
OTLP协议: otlp 是 opentelemetry 中比较核心的存在，在遥测数据及 Collector 之间制定了包括编码(encoding)、传输(transport)、传递(delivery)等协议。

image.png
核心概念
Specification: 是定义了 API/SDK 及数据模型，所有包含第三方实现的都需要遵循 spec 定义的规范。
Semantic Conventions: OpenTelemetry 项目保证所有的 instrumentation(不论任何语言)都包含相同的语义信息
Resource: resource 附加于某个 process 产生的所有 trace 的键值对，在初始化阶段指定并传递到 collector 中
Baggage: 为添加在 metrics、log、traces 中的注解信息，键值对需要唯一，无法更改
Propagators: 传播器，比如在多进程的调用中，开启传播器用于跨服务传播 spanContext。
Attributes: 其实就是 tag, 可以给 span 添加 metadata 数据,SetAttributes属性可以多次添加，不存在就添加，存在就覆盖
Events: 类似于日志, 比如在 trace 中嵌入请求体跟响应体。
Collector: 它负责遥测数据源的接收、处理和导出三部分，提供了与供应商无关的实现。
Data sources: OpenTelemetry 提供的数据源, 目前包含:
Traces
Metrics
Logs
Baggage

## Maven依赖
```xml
<dependencies>   
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-api</artifactId>
    </dependency>
    <dependency>
          <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-sdk-trace</artifactId>
    </dependency>
    <dependency>
          <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-sdk</artifactId>
    </dependency>
    <dependency>
          <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-logging</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.opentelemetry</groupId>
      <artifactId>opentelemetry-bom</artifactId>
      <version>1.2.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```