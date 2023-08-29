---
layout: post
categories: [Zookeeper]
description: none
keywords: Zookeeper
---
# Zookeeper源码环境搭建
Zookeeper是开源高可用的分布式协同服务，在分布式系统中应用广泛，代码量适中，适合阅读和学习。首先从开发环境的搭建开始介绍。

## 源码获取
Zookeeper源码Github地址[https://github.com/apache/zookeeper](https://github.com/apache/zookeeper)

Zookeeper官网（Apache）[https://zookeeper.apache.org/releases.html](https://zookeeper.apache.org/releases.html)

Zookeeper在3.5.5之前使用的是Ant构建，在3.5.5开始使用的是Maven构建。

## 工程结构

| 名称                                  | 说明                                                         |
|-------------------------------------| ------------------------------------------------------------ |
| bin	                                | 包含访问zookeeper服务器和命令行客户端的脚本 |
| conf                                | 启动zookeeper默认的配置文件目录    |
| zookeeper-assembly		            | 基础服务打包目录。                                      |
| zookeeper-client		                 | 客户端，目前只支持c。                                          |
| zookeeper-compatibility-tests		 | 兼容性测试目录。         |
| zookeeper-contrib                   | 附加的功能,比如zookeeper可视化客户端工具。                                         |
| zookeeper-doc		                  | zookeeper文档。|
| zookeeper-it		                      | 供fatjar使用，进行系统测试依赖的类|
| zookeeper-jute		                    | zookeeper序列化组件。|
| zookeeper-metrics-providers			      | 监控相关，目前支持普罗米修斯 prometheus。|
| zookeeper-recipes			                | zookeeper提供的一些功能例子，包括选举election，lock和queue。|
| zookeeper-server		                  | zookeeper服务端。|

## 启动服务

### maven执行install

运行命令：mvn -DskipTests clean install -U

构建信息
```text
[INFO] Apache ZooKeeper                                                   [pom]
[INFO] Apache ZooKeeper - Documentation                                   [jar]
[INFO] Apache ZooKeeper - Jute                                            [jar]
[INFO] Apache ZooKeeper - Server                                          [jar]
[INFO] Apache ZooKeeper - Client                                          [pom]
[INFO] Apache ZooKeeper - Recipes                                         [pom]
[INFO] Apache ZooKeeper - Recipes - Election                              [jar]
[INFO] Apache ZooKeeper - Recipes - Lock                                  [jar]
[INFO] Apache ZooKeeper - Recipes - Queue                                 [jar]
[INFO] Apache ZooKeeper - Assembly                                        [pom]
```

注：建议跳过test，不然需要很长时间，另外有些包是provider的，在打包时候不会打进去，所以需要改下，不然启动的时候会报class not found，这些依赖如下：

```xml
<dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-server</artifactId>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-servlet</artifactId>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <scope>provided</scope>
</dependency>
```


### 启动入口

zk服务端的启动入口是QuorumPeerMain类中的main方法

```text
如果不配置的话，无法输出日志
VM options：
-Dlog4j.configuration=file:conf\log4j.properties

单机版启动需要参数
Program arguments：
conf\zoo_sample.cfg
```

启动成功后的日志
```text
[main:QuorumPeerConfig@136] - Reading configuration from: D:\Github\zookeeper\conf\zoo_sample.cfg
[myid:] - WARN  [main:VerifyingFileFactory@59] - \tmp\zookeeper is relative. Prepend .\ to indicate that you're sure!
[myid:] - INFO  [main:QuorumPeerConfig@388] - clientPortAddress is 0.0.0.0:2181
[myid:] - INFO  [main:QuorumPeerConfig@392] - secureClientPort is not set
[myid:] - INFO  [main:DatadirCleanupManager@78] - autopurge.snapRetainCount set to 3
[myid:] - INFO  [main:DatadirCleanupManager@79] - autopurge.purgeInterval set to 0
[myid:] - INFO  [main:DatadirCleanupManager@101] - Purge task is not scheduled.
[myid:] - WARN  [main:QuorumPeerMain@125] - Either no config or no quorum defined in config, running  in standalone mode
[myid:] - INFO  [main:ManagedUtil@45] - Log4j 1.2 jmx support found and enabled.
[myid:] - INFO  [main:QuorumPeerConfig@136] - Reading configuration from: D:\Github\zookeeper\conf\zoo_sample.cfg
[myid:] - WARN  [main:VerifyingFileFactory@59] - \tmp\zookeeper is relative. Prepend .\ to indicate that you're sure!
[myid:] - INFO  [main:QuorumPeerConfig@388] - clientPortAddress is 0.0.0.0:2181
[myid:] - INFO  [main:QuorumPeerConfig@392] - secureClientPort is not set
[myid:] - INFO  [main:ZooKeeperServerMain@118] - Starting server
......
[myid:] - INFO  [main:Server@415] - Started @1399ms
[myid:] - INFO  [main:JettyAdminServer@116] - Started AdminServer on address 0.0.0.0, port 8080 and command URL /commands
[myid:] - INFO  [main:ServerCnxnFactory@135] - Using org.apache.zookeeper.server.NIOServerCnxnFactory as server connection factory
[myid:] - INFO  [main:NIOServerCnxnFactory@673] - Configuring NIO connection handler with 10s sessionless connection timeout, 2 selector thread(s), 32 worker threads, and 64 kB direct buffers.
[myid:] - INFO  [main:NIOServerCnxnFactory@686] - binding to port 0.0.0.0/0.0.0.0:2181
[myid:] - INFO  [main:ZKDatabase@117] - zookeeper.snapshotSizeFactor = 0.33
[myid:] - INFO  [main:FileTxnSnapLog@404] - Snapshotting: 0x0 to \tmp\zookeeper\version-2\snapshot.0
[myid:] - INFO  [main:FileTxnSnapLog@404] - Snapshotting: 0x0 to \tmp\zookeeper\version-2\snapshot.0
[myid:] - INFO  [ProcessThread(sid:0 cport:2181)::PrepRequestProcessor@132] - PrepRequestProcessor (sid:0) started, reconfigEnabled=false
[myid:] - INFO  [main:ContainerManager@64] - Using checkIntervalMs=60000 maxPerMinute=10000
[myid:] - INFO  [SyncThread:0:FileTxnLog@218] - Creating new log file: log.1

```
