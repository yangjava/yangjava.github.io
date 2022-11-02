---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
# Skywalking源码架构

想要阅读Skywalking源码，首先需要确定阅读的源码版本，版本不同，其功能特性会存在区别，功能的实现细节也会存在区别。

## 下载Skywalking源码，版本v8.7.0

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

### Skywalking源码模块

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

-- apm-agent 模块：其中包含了刚才使用的 SkyWalkingAgent 这个类，是整个 Agent 的入口。
-- apm-agent-core 模块：SkyWalking Agent 的核心实现都在该模块中。
-- apm-sdk-plugin 模块：SkyWalking Agent 使用了微内核+插件的架构，该模块下包含了 SkyWalking Agent 的全部插件，
-- apm-toolkit-activation 模块：apm-application-toolkit 模块的具体实现。

-  apm-webapp             # SkyWalking Rocketbot 对应的后端。
-  skywalking-ui          # SkyWalking Rocketbot 的前端。
-  oap-server             # OAP后端实现
-- analyzer     负责分析数据指标项等信息
-- exporter       负责导出数据。
-- server-alarm-plugin     负责实现 SkyWalking 的告警功能。
-- server-cluster-pulgin      负责 OAP 的集群信息管理，其中提供了接入多种第三方组件的相关插件，
-- server-configuration      负责管理 OAP 的配置信息，也提供了接入多种配置管理组件的相关插件，
-- server-core SkyWalking     OAP 的核心实现都在该模块中。
-- server-library             OAP 以及 OAP 各个插件依赖的公共模块，其中提供了双队列 Buffer、请求远端的 Client 等工具类，这些模块都是对立于 SkyWalking OAP 体系之外的类库，我们可以直接拿走使用。
-- server-query-plugin       SkyWalking Rocketbot 发送的请求首先由该模块接收处理，目前该模块只支持 GraphQL 查询。
-- server-receiver-plugin    SkyWalking Agent 发送来的 Metrics、Trace 以及 Register 等写入请求都是首先由该模块接收处理的，不仅如此，该模块还提供了多种接收其他格式写入请求的插件，
-- server-starter            OAP 服务启动的入口
-- server-starter-es7         OAP 服务启动的入口，使用es7 
-- server-storage-plugin 模块：OAP 服务底层可以使用多种存储来保存 Metrics 数据以及Trace 数据，该模块中包含了接入相关存储的插件，

### Skywalking执行流程

Skywalking主要由Agent、OAP、Storage、UI四大模块组成

Agent和业务程序运行在一起，采集链路及其它数据，通过gRPC发送给OAP（部分Agent采用http+json的方式）；OAP还原链路（图中的Tracing），并分析产生一些指标（图中的Metric），最终存储到Storage中。

## Skywalking源码调试

将skywalking源码拉取，即可进行调试

**全量构建：**

- mvn clean package -DskipTests


**高级编译：**

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
2022-11-01 17:24:35,729 main DEBUG ConsoleAppender$Builder(target="SYSTEM_OUT", follow="null", direct="null", bufferedIo="null", bufferSize="null", immediateFlush="null", ignoreExceptions="null", PatternLayout(%d - %c - %L [%t] %-5p %x - %m%n), name="Console", Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,729 main DEBUG Jansi is not installed, cannot find org.fusesource.jansi.WindowsAnsiOutputStream
2022-11-01 17:24:35,729 main DEBUG Starting OutputStreamManager SYSTEM_OUT.false.false
2022-11-01 17:24:35,729 main DEBUG Building Plugin[name=appenders, class=org.apache.logging.log4j.core.config.AppendersPlugin].
2022-11-01 17:24:35,729 main DEBUG createAppenders(={Console})
2022-11-01 17:24:35,729 main DEBUG Building Plugin[name=logger, class=org.apache.logging.log4j.core.config.LoggerConfig].
2022-11-01 17:24:35,745 main DEBUG createLogger(additivity="true", level="INFO", name="org.eclipse.jetty", includeLocation="null", ={}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,760 main DEBUG Building Plugin[name=logger, class=org.apache.logging.log4j.core.config.LoggerConfig].
2022-11-01 17:24:35,760 main DEBUG createLogger(additivity="true", level="INFO", name="org.apache.zookeeper", includeLocation="null", ={}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,760 main DEBUG Building Plugin[name=logger, class=org.apache.logging.log4j.core.config.LoggerConfig].
2022-11-01 17:24:35,760 main DEBUG createLogger(additivity="true", level="INFO", name="org.elasticsearch.common.network.IfConfig", includeLocation="null", ={}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,760 main DEBUG Building Plugin[name=logger, class=org.apache.logging.log4j.core.config.LoggerConfig].
2022-11-01 17:24:35,760 main DEBUG createLogger(additivity="true", level="INFO", name="org.elasticsearch.client.RestClient", includeLocation="null", ={}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,760 main DEBUG Building Plugin[name=logger, class=org.apache.logging.log4j.core.config.LoggerConfig].
2022-11-01 17:24:35,760 main DEBUG createLogger(additivity="true", level="INFO", name="io.grpc.netty", includeLocation="null", ={}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,760 main DEBUG Building Plugin[name=logger, class=org.apache.logging.log4j.core.config.LoggerConfig].
2022-11-01 17:24:35,760 main DEBUG createLogger(additivity="true", level="INFO", name="io.netty", includeLocation="null", ={}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,760 main DEBUG Building Plugin[name=logger, class=org.apache.logging.log4j.core.config.LoggerConfig].
2022-11-01 17:24:35,776 main DEBUG createLogger(additivity="true", level="INFO", name="org.apache.http", includeLocation="null", ={}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,776 main DEBUG Building Plugin[name=logger, class=org.apache.logging.log4j.core.config.LoggerConfig].
2022-11-01 17:24:35,776 main DEBUG createLogger(additivity="true", level="DEBUG", name="org.apache.skywalking.oap.server.core.alarm.AlarmStandardPersistence", includeLocation="null", ={}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,776 main DEBUG Building Plugin[name=logger, class=org.apache.logging.log4j.core.config.LoggerConfig].
2022-11-01 17:24:35,776 main DEBUG createLogger(additivity="true", level="DEBUG", name="org.apache.skywalking.oap.server.core", includeLocation="null", ={}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,776 main DEBUG Building Plugin[name=logger, class=org.apache.logging.log4j.core.config.LoggerConfig].
2022-11-01 17:24:35,776 main DEBUG createLogger(additivity="true", level="INFO", name="org.apache.skywalking.oap.server.core.storage.PersistenceTimer", includeLocation="null", ={}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,776 main DEBUG Building Plugin[name=logger, class=org.apache.logging.log4j.core.config.LoggerConfig].
2022-11-01 17:24:35,776 main DEBUG createLogger(additivity="true", level="DEBUG", name="org.apache.skywalking.oap.server.core.analysis.worker", includeLocation="null", ={}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,776 main DEBUG Building Plugin[name=logger, class=org.apache.logging.log4j.core.config.LoggerConfig].
2022-11-01 17:24:35,776 main DEBUG createLogger(additivity="true", level="DEBUG", name="org.apache.skywalking.oap.server.core.remote.client", includeLocation="null", ={}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,776 main DEBUG Building Plugin[name=logger, class=org.apache.logging.log4j.core.config.LoggerConfig].
2022-11-01 17:24:35,776 main DEBUG createLogger(additivity="true", level="INFO", name="org.apache.skywalking.oap.server.library.buffer", includeLocation="null", ={}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,776 main DEBUG Building Plugin[name=AppenderRef, class=org.apache.logging.log4j.core.config.AppenderRef].
2022-11-01 17:24:35,791 main DEBUG createAppenderRef(ref="Console", level="null", Filter=null)
2022-11-01 17:24:35,791 main DEBUG Building Plugin[name=root, class=org.apache.logging.log4j.core.config.LoggerConfig$RootLogger].
2022-11-01 17:24:35,791 main DEBUG createLogger(additivity="null", level="DEBUG", includeLocation="null", ={Console}, ={}, Configuration(D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml), Filter=null)
2022-11-01 17:24:35,791 main DEBUG Building Plugin[name=loggers, class=org.apache.logging.log4j.core.config.LoggersPlugin].
2022-11-01 17:24:35,791 main DEBUG createLoggers(={org.eclipse.jetty, org.apache.zookeeper, org.elasticsearch.common.network.IfConfig, org.elasticsearch.client.RestClient, io.grpc.netty, io.netty, org.apache.http, org.apache.skywalking.oap.server.core.alarm.AlarmStandardPersistence, org.apache.skywalking.oap.server.core, org.apache.skywalking.oap.server.core.storage.PersistenceTimer, org.apache.skywalking.oap.server.core.analysis.worker, org.apache.skywalking.oap.server.core.remote.client, org.apache.skywalking.oap.server.library.buffer, root})
2022-11-01 17:24:35,791 main DEBUG Configuration XmlConfiguration[location=D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml] initialized
2022-11-01 17:24:35,807 main DEBUG Starting configuration XmlConfiguration[location=D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml]
2022-11-01 17:24:35,807 main DEBUG Started configuration XmlConfiguration[location=D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml] OK.
2022-11-01 17:24:35,807 main DEBUG Shutting down OutputStreamManager SYSTEM_OUT.false.false-1
2022-11-01 17:24:35,807 main DEBUG Shut down OutputStreamManager SYSTEM_OUT.false.false-1, all resources released: true
2022-11-01 17:24:35,807 main DEBUG Appender DefaultConsole-1 stopped with status true
2022-11-01 17:24:35,807 main DEBUG Stopped org.apache.logging.log4j.core.config.DefaultConfiguration@78ac1102 OK
2022-11-01 17:24:35,921 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2
2022-11-01 17:24:35,937 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=StatusLogger
2022-11-01 17:24:35,953 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=ContextSelector
2022-11-01 17:24:35,953 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=org.elasticsearch.client.RestClient
2022-11-01 17:24:35,953 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=
2022-11-01 17:24:35,953 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=org.eclipse.jetty
2022-11-01 17:24:35,953 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=org.apache.skywalking.oap.server.core.alarm.AlarmStandardPersistence
2022-11-01 17:24:35,968 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=org.apache.zookeeper
2022-11-01 17:24:35,968 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=org.apache.http
2022-11-01 17:24:35,968 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=org.apache.skywalking.oap.server.core
2022-11-01 17:24:35,968 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=io.grpc.netty
2022-11-01 17:24:35,968 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=io.netty
2022-11-01 17:24:35,968 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=org.apache.skywalking.oap.server.library.buffer
2022-11-01 17:24:35,968 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=org.elasticsearch.common.network.IfConfig
2022-11-01 17:24:35,968 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=org.apache.skywalking.oap.server.core.remote.client
2022-11-01 17:24:35,968 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=org.apache.skywalking.oap.server.core.analysis.worker
2022-11-01 17:24:35,968 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Loggers,name=org.apache.skywalking.oap.server.core.storage.PersistenceTimer
2022-11-01 17:24:35,968 main DEBUG Registering MBean org.apache.logging.log4j2:type=18b4aac2,component=Appenders,name=Console
2022-11-01 17:24:36,000 main DEBUG Reconfiguration complete for context[name=18b4aac2] at URI D:\Qjd\skywalking\oap-server\server-bootstrap\target\classes\log4j2.xml (org.apache.logging.log4j.core.LoggerContext@57ea113a) with optional ClassLoader: null
2022-11-01 17:24:36,000 main DEBUG Shutdown hook enabled. Registering a new one.
2022-11-01 17:24:36,000 main DEBUG LoggerContext[name=18b4aac2, org.apache.logging.log4j.core.LoggerContext@57ea113a] started OK.
2022-11-01 17:24:36,421 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 163 [main] INFO  [] - Remove module receiver-zabbix without any provider
2022-11-01 17:24:36,452 main DEBUG AsyncLogger.ThreadNameStrategy=CACHED
2022-11-01 17:24:36,456 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 163 [main] INFO  [] - Remove module prometheus-fetcher without any provider
2022-11-01 17:24:36,458 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 163 [main] INFO  [] - Remove module kafka-fetcher without any provider
2022-11-01 17:24:36,458 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 163 [main] INFO  [] - Remove module receiver-otel without any provider
2022-11-01 17:24:36,458 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 163 [main] INFO  [] - Remove module receiver_zipkin without any provider
2022-11-01 17:24:36,458 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 163 [main] INFO  [] - Remove module receiver_jaeger without any provider
2022-11-01 17:24:36,458 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 163 [main] INFO  [] - Remove module exporter without any provider
2022-11-01 17:24:36,458 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 163 [main] INFO  [] - Remove module health-checker without any provider
2022-11-01 17:24:36,460 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: cluster
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to cluster module, provider name: standalone
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: core
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to core module, provider name: default
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=role has been set as Mixed
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restHost has been set as 0.0.0.0
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restPort has been set as 12800
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restContextPath has been set as /
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restMinThreads has been set as 1
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restMaxThreads has been set as 200
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restIdleTimeOut has been set as 30000
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restAcceptorPriorityDelta has been set as 0
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restAcceptQueueSize has been set as 0
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCHost has been set as 0.0.0.0
2022-11-01 17:24:36,462 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCPort has been set as 11800
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=maxConcurrentCallsPerConnection has been set as 0
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=maxMessageSize has been set as 0
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCThreadPoolQueueSize has been set as -1
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCThreadPoolSize has been set as -1
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCSslEnabled has been set as false
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCSslKeyPath has been set as 
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCSslCertChainPath has been set as 
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCSslTrustedCAPath has been set as 
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=downsampling has been set as [Hour, Day]
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=enableDataKeeperExecutor has been set as true
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=dataKeeperExecutePeriod has been set as 5
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=recordDataTTL has been set as 3
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=metricsDataTTL has been set as 7
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=enableDatabaseSession has been set as true
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=topNReportPeriod has been set as 10
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=activeExtraModelColumns has been set as false
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=serviceNameMaxLength has been set as 70
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=instanceNameMaxLength has been set as 70
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=endpointNameMaxLength has been set as 150
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=searchableTracesTags has been set as http.method,status_code,db.type,db.instance,mq.queue,mq.topic,mq.broker
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=searchableLogsTags has been set as level
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=searchableAlarmTags has been set as level
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=syncThreads has been set as 2
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=maxSyncOperationNum has been set as 50000
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: storage
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to storage module, provider name: h2
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=h2 config=driver has been set as org.h2.jdbcx.JdbcDataSource
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=h2 config=url has been set as jdbc:h2:mem:skywalking-oap-db;DB_CLOSE_DELAY=-1
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=h2 config=user has been set as sa
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=h2 config=metadataQueryMaxSize has been set as 5000
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=h2 config=maxSizeOfArrayColumn has been set as 20
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=h2 config=numOfSearchableValuesPerTag has been set as 2
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: agent-analyzer
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to agent-analyzer module, provider name: default
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=sampleRate has been set as 10000
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=slowDBAccessThreshold has been set as default:200,mongodb:100
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=forceSampleErrorSegment has been set as true
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=segmentStatusAnalysisStrategy has been set as FROM_SPAN_STATUS
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=noUpstreamRealAddressAgents has been set as 6000,9000
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=slowTraceSegmentThreshold has been set as -1
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=meterAnalyzerActiveFiles has been set as spring-sleuth
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: log-analyzer
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to log-analyzer module, provider name: default
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=lalFiles has been set as default
2022-11-01 17:24:36,478 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=malFiles has been set as 
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: event-analyzer
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to event-analyzer module, provider name: default
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: receiver-sharing-server
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to receiver-sharing-server module, provider name: default
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restHost has been set as 0.0.0.0
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restPort has been set as 0
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restContextPath has been set as /
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restMinThreads has been set as 1
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restMaxThreads has been set as 200
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restIdleTimeOut has been set as 30000
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restAcceptorPriorityDelta has been set as 0
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=restAcceptQueueSize has been set as 0
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCHost has been set as 0.0.0.0
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCPort has been set as 0
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=maxConcurrentCallsPerConnection has been set as 0
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=maxMessageSize has been set as 0
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCThreadPoolQueueSize has been set as 0
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCThreadPoolSize has been set as 0
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCSslEnabled has been set as false
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCSslKeyPath has been set as 
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=gRPCSslCertChainPath has been set as 
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=authentication has been set as 
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: receiver-register
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to receiver-register module, provider name: default
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: receiver-trace
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to receiver-trace module, provider name: default
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: receiver-jvm
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to receiver-jvm module, provider name: default
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: receiver-clr
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to receiver-clr module, provider name: default
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: receiver-profile
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to receiver-profile module, provider name: default
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: service-mesh
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to service-mesh module, provider name: default
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: envoy-metric
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to envoy-metric module, provider name: default
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=acceptMetricsService has been set as true
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=alsHTTPAnalysis has been set as 
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=alsTCPAnalysis has been set as 
2022-11-01 17:24:36,494 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=k8sServiceNameRule has been set as ${pod.metadata.labels.(service.istio.io/canonical-name)}
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: receiver-meter
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to receiver-meter module, provider name: default
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: receiver-browser
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to receiver-browser module, provider name: default
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=sampleRate has been set as 10000
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: receiver-log
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to receiver-log module, provider name: default
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: query
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to query module, provider name: graphql
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=graphql config=path has been set as /graphql
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: alarm
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to alarm module, provider name: default
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: telemetry
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to telemetry module, provider name: none
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: configuration
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to configuration module, provider name: none
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: configuration-discovery
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to configuration-discovery module, provider name: default
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 117 [main] INFO  [] - Provider=default config=disableMessageDigest has been set as false
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 64 [main] INFO  [] - Get a module define from application.yml, module name: receiver-event
2022-11-01 17:24:36,509 - org.apache.skywalking.oap.server.starter.config.ApplicationConfigLoader - 68 [main] INFO  [] - Get a provider define belong to receiver-event module, provider name: default
2022-11-01 17:24:37,587 - org.apache.skywalking.oap.server.library.module.ModuleDefine - 89 [main] INFO  [] - Prepare the h2 provider in storage module.
2022-11-01 17:26:26,744 - org.apache.skywalking.oap.server.library.module.ModuleDefine - 89 [main] INFO  [] - Prepare the standalone provider in cluster module.
2022-11-01 17:26:26,837 - org.apache.skywalking.oap.server.library.module.ModuleDefine - 89 [main] INFO  [] - Prepare the default provider in core module.
2022-11-01 17:26:35,954 - org.apache.skywalking.oap.server.library.server.grpc.GRPCServer - 117 [main] INFO  [] - Server started, host 0.0.0.0 listening on 11800
2022-11-01 17:26:36,026 - org.eclipse.jetty.util.log - 169 [main] INFO  [] - Logging initialized @126544ms to org.eclipse.jetty.util.log.Slf4jLog
2022-11-01 17:26:36,355 - org.apache.skywalking.oap.server.library.server.jetty.JettyServer - 73 [main] INFO  [] - http server root context path: /
2022-11-01 17:26:36,677 - org.apache.skywalking.oap.server.library.module.ModuleDefine - 89 [main] INFO  [] - Prepare the graphql provider in query module.
2022-11-01 17:26:38,015 - com.coxautodev.graphql.tools.SchemaClassScanner - 155 [main] WARN  [] - Schema type was defined but can never be accessed, and can be safely deleted: NodeType
2022-11-01 17:26:38,015 - com.coxautodev.graphql.tools.SchemaClassScanner - 155 [main] WARN  [] - Schema type was defined but can never be accessed, and can be safely deleted: Long
2022-11-01 17:26:39,333 - org.apache.skywalking.oap.server.library.module.ModuleDefine - 89 [main] INFO  [] - Prepare the default provider in alarm module.

```






