---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
# Skywalking源码架构
想要阅读Skywalking源码，首先需要确定阅读的源码版本，版本不同，其功能特性会存在区别，功能的实现细节也会存在区别。

## 下载Skywalking源码，版本v8.7.0
https://skywalking.apache.org/zh/2022-03-25-skywalking-source-code-analyzation/
[官方仓库 https://github.com/apache/skywalking](https://github.com/apache/skywalking)

首先在github上下载版本为v8.7.0的skywalking:
skywalking源码的目录结构和模块的作用如下:
```text
skywalking-v8.7.0
├─apm-application-toolkit       #SkyWalking 提供给用户调用的工具箱。                 
├─apm-commons                   #SkyWalking 的公共组件和工具类
├─apm-protocol                  # 交互协议模块
├─apm-sniffer                   # apm模块,收集链路信息，发送给 SkyWalking OAP 服务器
├─├─apm-agent                   # Skywalking的整个Agent的入口
├─├─apm-agent-core              # Agent的核心实现
├─├─apm-sdk-plugin              # Agent微内核+插件
├─├─apm-toolkit-activation      # apm-application-toolkit实现
├─apm-webapp                    # Rocketbot 对应的后端
├─skywalking-ui                 # Rocketbot 的前端
├─oap-server                    # OAP后端实现
├─├─analyzer                    # 负责分析数据指标项等信息
├─├─exporter                    # 负责导出数据。 
├─├─server-alarm-plugin       # 对外提供服务
├─├─server-cluster-pulgin            # 网络通信相关模块，远程调用接口，封装Netty底层通信
├─├─server-configuration              # namesrc 模块的相关工具，提供一些公用的工具方法，比如解析命令行参数
├─├─server-core             # 消息存储相关模块
├─├─server-library               # 测试相关模块
├─├─server-query-plugin                # 工具类模块，命令行管理工具，如mqadmin工具
├─├─server-receiver-plugin
├─├─server-starter
├─├─server-starter-es7
├─├─server-storage-plugin
```

## Skywalking源码模块
源码模块如下:
- **APM(application performance monitor)** ：负责从应用中，收集链路信息，发送给 SkyWalking OAP 服务器。
- **OAP** ：负责接收 Agent 发送的 Tracing 数据信息，然后进行分析(Analysis Core) ，存储到外部存储器( Storage )，最终提供查询( Query )功能。
- **Storage** ：Tracing 数据存储。目前支持 ES、MySQL、Sharding Sphere、TiDB、H2 多种存储器。
- **SkyWalking UI** ：负责提供控台，查看链路等等。

详细信息如下：
- apm-application-toolkit  #SkyWalking 提供给用户调用的工具箱。
  该模块提供了对 log4j、log4j2、logback 等常见日志框架的接入接口，提供了 @Trace 注解等。
  apm-application-toolkit 模块类似于暴露 API 定义，对应的处理逻辑在 apm-sniffer/apm-toolkit-activation 模块中实现。
- apm-commons              #SkyWalking 的公共组件和工具类
  其中包含两个子模块，apm-datacarrier 模块提供了一个生产者-消费者模式的缓存组件（DataCarrier），无论是在 Agent 端还是 OAP 端都依赖该组件。
  apm-util 模块则提供了一些常用的工具类，例如，字符串处理工具类（StringUtil）、占位符处理的工具类（PropertyPlaceholderHelper、PlaceholderConfigurerSupport）等等。
- apm-protocol             # 交互协议模块
  该模块中只有一个 apm-network 模块，我们需要关注的是其中定义的 .proto 文件，定义 Agent 与后端 OAP 使用 gRPC 交互时的协议。
-  apm-sniffer             # apm模块,收集链路信息，发送给 SkyWalking OAP 服务器

    - apm-agent 模块：其中包含了刚才使用的 SkyWalkingAgent 这个类，是整个 Agent 的入口。
    - apm-agent-core 模块：SkyWalking Agent 的核心实现都在该模块中。
    - apm-sdk-plugin 模块：SkyWalking Agent 使用了微内核+插件的架构，该模块下包含了 SkyWalking Agent 的全部插件，
    - apm-toolkit-activation 模块：apm-application-toolkit 模块的具体实现。

-  apm-webapp             # SkyWalking Rocketbot 对应的后端。
-  skywalking-ui          # SkyWalking Rocketbot 的前端。
-  oap-server             # OAP后端实现
    - analyzer     负责分析数据指标项等信息
    - exporter       负责导出数据。
    - server-alarm-plugin     负责实现 SkyWalking 的告警功能。
    - server-cluster-pulgin      负责 OAP 的集群信息管理，其中提供了接入多种第三方组件的相关插件，
    - server-configuration      负责管理 OAP 的配置信息，也提供了接入多种配置管理组件的相关插件，
    - server-core SkyWalking     OAP 的核心实现都在该模块中。
    - server-library             OAP 以及 OAP 各个插件依赖的公共模块，其中提供了双队列 Buffer、请求远端的 Client 等工具类，这些模块都是对立于 SkyWalking OAP 体系之外的类库，我们可以直接拿走使用。
    - server-query-plugin       SkyWalking Rocketbot 发送的请求首先由该模块接收处理，目前该模块只支持 GraphQL 查询。
    - server-receiver-plugin    SkyWalking Agent 发送来的 Metrics、Trace 以及 Register 等写入请求都是首先由该模块接收处理的，不仅如此，该模块还提供了多种接收其他格式写入请求的插件。
- server-starter            OAP 服务启动的入口
- server-starter-es7         OAP 服务启动的入口，使用es7
- server-storage-plugin 模块：OAP 服务底层可以使用多种存储来保存 Metrics 数据以及Trace 数据，该模块中包含了接入相关存储的插件，

## Skywalking执行流程

Skywalking主要由Agent、OAP、Storage、UI四大模块组成

Agent和业务程序运行在一起，采集链路及其它数据，通过gRPC发送给OAP（部分Agent采用http+json的方式）；OAP还原链路（图中的Tracing），并分析产生一些指标（图中的Metric），最终存储到Storage中。

## Skywalking源码调试
将skywalking源码拉取，即可进行调试
如何对skywalking全量构建
```shell
mvn clean package -DskipTests
```
高级编译
- 编译agent: mvn package -Pagent,dist -DskipTests
- 编译backend: mvn package -Pbackend,dist -DskipTests
- 编译UI包: mvn package -Pui,dist

### 启动 SkyWalking OAP
- 在OAP下执行如下命令`mvn compile -Dmaven.test.skip=true` 进行编译。
- 启动`org.apache.skywalking.oap.server.starter.OAPServerBootstrap`
- 后端AOP启动日志
```text
2022-11-01 17:24:34,647 main DEBUG Initializing configuration XmlConfiguration[location=D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml]
2022-11-01 17:24:34,710 main DEBUG Installed 2 script engines
2022-11-01 17:24:35,381 main DEBUG Oracle Nashorn version: 1.8.0_201, language: ECMAScript, threading: Not Thread Safe, compile: true, names: [nashorn, Nashorn, js, JS, JavaScript, javascript, ECMAScript, ecmascript], factory class: jdk.nashorn.api.scripting.NashornScriptEngineFactory
2022-11-01 17:24:35,381 main DEBUG MVEL (MVFLEX Expression Language) version: 2.3, language: mvel, threading: THREAD-ISOLATED, compile: true, names: [mvel], factory class: org.mvel2.jsr223.MvelScriptEngineFactory
2022-11-01 17:24:35,381 main DEBUG PluginManager 'Core' found 115 plugins
2022-11-01 17:24:35,381 main DEBUG PluginManager 'Level' found 0 plugins
2022-11-01 17:24:35,397 main DEBUG PluginManager 'Lookup' found 13 plugins
2022-11-01 17:24:35,397 main DEBUG Building Plugin[name=layout, class=org.apache.logging.log4j.core.layout.PatternLayout].
2022-11-01 17:24:35,471 main DEBUG PluginManager 'TypeConverter' found 26 plugins
2022-11-01 17:24:35,533 main DEBUG PatternLayout$Builder(pattern="%d - %c - %L [%t] %-5p %x - %m%n", PatternSelector=null, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Replace=null, charset="UTF-8", alwaysWriteExceptions="null", disableAnsi="null", noConsoleNoAnsi="null", header="null", footer="null")
2022-11-01 17:24:35,533 main DEBUG PluginManager 'Converter' found 42 plugins
2022-11-01 17:24:35,674 main DEBUG Building Plugin[name=appender, class=org.apache.logging.log4j.core.appender.ConsoleAppender].
......
```

需要单独说明的：

如果你是用的是8.7.0之前的版本，skywalking的服务端和agent的代码是放在同一个项目工程skywalking下的。

如果使用8.7.0之后的版本，agent的相关代码被抽离出skywalkin当中，不同的语言会对应不同的agent，需要根据需要去自行选择，比如java使用skywalking-java

## agent分离后的代码
Skywalking从V8.7.0以后将agent和OAP进行分包。

官方仓库 OAP地址：[https://github.com/apache/skywalking](https://github.com/apache/skywalking)
Java Agent地址：[https://github.com/apache/skywalking-java](https://github.com/apache/skywalking-java)

获取Skywalking源码命令如下：
```shell
git clone https://github.com/apache/skywalking.git
git submodule init
git submodule update
```

使用Maven编译
```shell
运行 ./mvnw clean package -DskipTests
```

所有打出来的包都在目录 /dist 下 (Linux 下为 .tar.gz, Windows 下为 .zip)

获取Java的Skywalking探针。项目为skywalking-java



