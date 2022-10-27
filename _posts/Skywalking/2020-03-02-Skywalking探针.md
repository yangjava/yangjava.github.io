---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
## Probe(探针)

### 简介

在 SkyWalking 中，探针是指集成到目标系统中的代理或 SDK 库，负责收集遥测数据，包括跟踪和指标。根据目标系统技术堆栈，探针执行此类任务的方式有很大不同。但最终，他们都朝着同一个目标努力——收集和重新格式化数据，然后将它们发送到后端。

在高层次上，所有 SkyWalking 探测器都有三个典型类别。

- **基于语言的本地代理（Language based native agent）。**这些代理运行在目标服务用户空间中，例如用户代码的一部分。例如，SkyWalking Java 代理使用`-javaagent`命令行参数在运行时操作代码，这`manipulate`意味着更改和注入用户的代码。另一种代理使用目标库提供的某种钩子或拦截机制。如您所见，这些代理基于语言和库。
- **服务网格探针（Service Mesh probes）。**Service Mesh 探针从 Sidecar、Service Mesh 中的控制面板或代理收集数据。过去，proxy 只是作为整个集群的入口，但是有了 Service Mesh 和 sidecar，我们现在可以执行可观察性功能。
- **3rd 方仪器库（3rd-party instrument library）。**SkyWalking 接受许多广泛使用的仪器库数据格式。它分析数据，将其传输到 SkyWalking 的跟踪、度量或两者格式。此功能从接受 Zipkin 跨度数据开始。

您不需要同时使用**基于语言的本机代理**和**服务网格探测**，因为它们都用于收集指标数据。否则，您的系统将遭受两倍的有效负载，并且分析数将翻倍。

关于如何使用这些探针，有几种推荐的方法：

1. 仅使用**基于语言的本地代理**。
2. 仅使用**3rd 方仪器库**，例如 Zipkin 仪器生态系统。
3. 仅使用**服务网格探测**。
4. 在跟踪状态下使用带有**基于语言的本机代理**或**3rd 方仪器库的****Service Mesh 探针。**（高级用法）

**跟踪状态**中的含义是什么？

默认情况下，**基于语言的本机代理**和**3rd 方仪器库**都将分布式跟踪发送到后端，在后端对这些跟踪执行分析和聚合。**跟踪状态**意味着后端将这些跟踪视为日志。换句话说，后端保存它们，并在跟踪和指标之间建立链接，例如`which endpoint and service does the trace belong?`.

### Service Auto Instrument Agent

The service auto instrument agent是基于语言的本地代理的子集。这种代理基于一些特定于语言的特性，尤其是基于 VM 的语言的特性。

#### Auto Instrument

许多用户在第一次听说“无需更改任何一行代码”时就了解了这些代理。SkyWalking 也曾在其自述页面中提到这一点。然而，这并不能反映全貌。对于最终用户来说，确实在大多数情况下他们不再需要修改他们的代码。但重要的是要了解代码实际上仍由代理修改，这通常称为“运行时代码操作”。其底层逻辑是自动仪表代理使用VM接口进行代码修改，动态添加仪表代码，比如通过 `javaagent premain`.

事实上，尽管 SkyWalking 团队已经提到大多数自动仪器代理都是基于 VM 的，但您可以在编译时而不是运行时构建此类工具。

#### 有什么限制？

自动检测非常有用，因为您可以在编译期间执行自动检测，而无需依赖 VM 功能。但它也有一些限制：

- **在许多情况下，进程内传播的可能性更高（Higher possibility of in-process propagation in many cases）**。许多高级语言，如 Java 和 .NET，用于构建业务系统。大多数业务逻辑代码在每个请求的同一线程中运行，这导致传播基于线程 ID，以便堆栈模块确保上下文是安全的。
- **仅适用于某些框架或库（Only works in certain frameworks or libraries）**。由于代理负责在运行时修改代码，因此代理插件开发人员已经知道这些代码。通常有一个此类探针支持的框架或库的列表。
- **并非总是支持跨线程操作(Cross-thread operations are not always supported)**。就像上面提到的进程内传播一样，大多数代码（尤其是业务代码）在每个请求的单个线程中运行。但在其他一些情况下，它们会跨不同的线程进行操作，例如将任务分配给其他线程、任务池或批处理。一些语言甚至可能提供协程或类似的组件，例如`Goroutine`，允许开发人员以低负载运行异步进程。在这种情况下，auto instrument将面临问题。

所以，auto instrument没有什么神秘的。简而言之，代理开发人员编写激活脚本以使instrument代码为您工作。就是这样！

### 服务网格探针(Service Mesh Probe)

Service Mesh 探针使用 Service Mesh 实现者（如 Istio）中提供的可扩展机制。

#### 什么是服务网格？

以下解释来自 Istio 的文档。

> 术语“服务网格”通常用于描述构成此类应用程序的微服务网络以及它们之间的交互。随着服务网格的规模和复杂性的增长，它可能变得更难理解和管理。它的要求可能包括发现、负载平衡、故障恢复、度量和监控，以及通常更复杂的操作要求，例如 A/B 测试、金丝雀发布、速率限制、访问控制和端到端身份验证。

#### 探针从哪里收集数据？

Istio 是典型的 Service Mesh 设计和实现者。它定义了广泛使用的**控制面板**和**数据面板。**这是 Istio 架构：

![Skywalking-ServiceMesh](png\skywalking\Skywalking-ServiceMesh.svg)

Service Mesh 探针可以选择从**数据面板**收集数据。在 Istio 中，这意味着从 Envoy sidecar（数据面板）收集遥测数据。探针从每个请求的客户端和服务器端收集两个遥测实体。

#### Service Mesh 如何让后端工作？

在这种探针中，您可以看到没有与它们相关的痕迹。那么 SkyWalking 平台是如何运作的呢？

Service Mesh 探针从每个请求中收集遥测数据，因此他们了解源、目标、端点、延迟和状态等信息。从这些信息中，后端可以通过将这些调用组合成行来告诉整个拓扑图，以及每个节点通过其传入请求的度量。后端通过解析跟踪数据请求相同的指标数据。简而言之： **Service Mesh 指标的工作方式与跟踪解析器生成的指标完全相同。**