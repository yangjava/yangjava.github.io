---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---


## 调试环境搭建

##### 源码拉取

从官方仓库 https://github.com/OpenSkywalking/skywalking `Fork` 出属于自己的仓库。为什么要 `Fork` ？既然开始阅读、调试源码，我们可能会写一些注释，有了自己的仓库，可以进行自由的提交。

使用 `IntelliJ IDEA` 从 `Fork` 出来的仓库拉取代码。拉取完成后，`Maven` 会下载依赖包，可能会花费一些时间，耐心等待下。

本文基于 `master` 分支。

##### 启动 SkyWalking Collector

1. 在 IntelliJ IDEA Terminal 中，执行 `mvn compile -Dmaven.test.skip=true` 进行编译。
2. 设置 gRPC 的**自动生成**的代码目录，为**源码**目录 ：
    - /apm-network/target/generated-sources/protobuf/ 下的 `grpc-java` 和 `java` 目录
    - /apm-collector-remote/collector-remote-grpc-provider/target/generated-sources/protobuf/ 下的 `grpc-java` 和 `java` 目录
3. 运行 `org.skywalking.apm.collector.bootCollector.BootStartUp` 的 `#main(args)` 方法，启动 Collector 。
4. 访问 `http://127.0.0.1:10800/agent/jetty` 地址，返回 `["localhost:12800/"]` ，说明启动**成功**。

##### 启动 SkyWalking Agent

1. 在 IntelliJ IDEA Terminal 中，执行 `mvn compile -Dmaven.test.skip=true` 进行编译。在 /packages/skywalking-agent 目录下，我们可以看到编译出来的 Agent ：

2. 使用 Spring Boot 创建一个简单的 Web 项目。

   > 友情提示 ：**这里一定要注意下**。创建的 Web 项目，使用 IntelliJ IDEA 的**菜单** File / New / Module 或 File / New / Module from Existing Sources ，**保证 Web 项目和 skywalking 项目平级**。这样，才可以使用 IntelliJ IDEA 调试 Agent 。

3. 在 `org.skywalking.apm.agent.SkyWalkingAgent` 的 `#premain(...)` 方法，打上调试断点。

4. 运行 Web 项目的 Application 的 `#main(args)` 方法，并增加 JVM 启动参数，`-javaagent:/path/to/skywalking-agent/skywalking-agent.jar`。`/path/to` **参数值**为上面我们编译出来的 /packages/skywalking-agent 目录的绝对路径。

5. 如果在【**第三步**】的调试断点停住，说明 Agent 启动**成功**。

------

考虑到可能我们会在 Agent 上增加代码注释，这样每次不得不重新编译 Agent 。可以配置如下图，自动编译 Agent ：

- `-T 1C clean package -Dmaven.test.skip=true -Dmaven.compile.fork=true` 。

------

另外，使用 IntelliJ IDEA Remote 远程调试，也是可以的。

##### 启动 SkyWalking Web UI

考虑到调试过程中，我们要看下是否收集到追踪日志，可以安装 SkyWalking Web UI 进行查看。

## 模块

SkyWalking 源码的整体结构如下图所示：

1、apm-application-toolkit 模块：SkyWalking 提供给用户调用的工具箱。

该模块提供了对 log4j、log4j2、logback 等常见日志框架的接入接口，提供了 @Trace 注解等。
apm-application-toolkit 模块类似于暴露 API 定义，对应的处理逻辑在 apm-sniffer/apm-toolkit-activation 模块中实现。

2、apm-commons 模块：SkyWalking 的公共组件和工具类。

其中包含两个子模块，apm-datacarrier 模块提供了一个生产者-消费者模式的缓存组件（DataCarrier），无论是在 Agent 端还是 OAP 端都依赖该组件。
apm-util 模块则提供了一些常用的工具类，例如，字符串处理工具类（StringUtil）、占位符处理的工具类（PropertyPlaceholderHelper、PlaceholderConfigurerSupport）等等。
3、apm-protocol 模块：该模块中只有一个 apm-network 模块，我们需要关注的是其中定义的 .proto 文件，定义 Agent 与后端 OAP 使用 gRPC 交互时的协议。

4、apm-sniffer 模块：apm-sniffer 模块中有 4 个子模块：

apm-agent 模块：其中包含了刚才使用的 SkyWalkingAgent 这个类，是整个 Agent 的入口。

apm-agent-core 模块：SkyWalking Agent 的核心实现都在该模块中。

apm-sdk-plugin 模块：SkyWalking Agent 使用了微内核+插件的架构，该模块下包含了 SkyWalking Agent 的全部插件，

apm-toolkit-activation 模块：apm-application-toolkit 模块的具体实现。

5、apm-webapp 模块：SkyWalking Rocketbot 对应的后端。

6、oap-server 模块：SkyWalking OAP 的全部实现都在 oap-server 模块，其中包含了多个子模块，
exporter 模块：负责导出数据。

server-alarm-plugin 模块：负责实现 SkyWalking 的告警功能。

server-cluster-pulgin 模块：负责 OAP 的集群信息管理，其中提供了接入多种第三方组件的相关插件，

server-configuration 模块：负责管理 OAP 的配置信息，也提供了接入多种配置管理组件的相关插件，

server-core模块：SkyWalking OAP 的核心实现都在该模块中。

server-library 模块：OAP 以及 OAP 各个插件依赖的公共模块，其中提供了双队列 Buffer、请求远端的 Client 等工具类，这些模块都是对立于 SkyWalking OAP 体系之外的类库，我们可以直接拿走使用。

server-query-plugin 模块：SkyWalking Rocketbot 发送的请求首先由该模块接收处理，目前该模块只支持 GraphQL 查询。

server-receiver-plugin 模块：SkyWalking Agent 发送来的 Metrics、Trace 以及 Register 等写入请求都是首先由该模块接收处理的，不仅如此，该模块还提供了多种接收其他格式写入请求的插件，

server-starter 模块：OAP 服务启动的入口。

server-storage-plugin 模块：OAP 服务底层可以使用多种存储来保存 Metrics 数据以及Trace 数据，该模块中包含了接入相关存储的插件，

7、skywalking-ui 目录：SkyWalking Rocketbot 的前端。