---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
## 下载与安装

SkyWalking有两种版本，ES版本和非ES版。如果我们决定采用ElasticSearch作为存储，那么就下载es版本

agent目录将来要拷贝到各服务所在机器上用作探针

bin目录是服务启动脚本

config目录是配置文件

oap-libs目录是oap服务运行所需的jar包

webapp目录是web服务运行所需的jar包

接下来，要选择存储了，支持的存储有：

- H2
- ElasticSearch 6, 7
- MySQL
- TiDB
- **InfluxDB**

作为监控系统，首先排除H2和MySQL，这里推荐InfluxDB，它本身就是时序数据库，非常适合这种场景

## SpringBoot 实例部署

原来的启动方式为：

```bash
java -jar spring-boot-demo-0.0.1-SNAPSHOT.jar
```

那么使用 skywalking agent，启动命令为：

```bash
java -javaagent:/opt/apache-skywalking-apm-bin/agent/skywalking-agent.jar -Dskywalking.agent.service_name=xxxtest -Dskywalking.collector.backend_service=127.0.0.1:11800 -jar /opt/spring-boot-demo-0.0.1-SNAPSHOT.jar
```

说明：

- `-javaagent` 指定agent包位置。这里将apache-skywalking-apm-6.6.0.tar.gz解压到/opt目录了，因此路径为：/opt/apache-skywalking-apm-bin/agent/skywalking-agent.jar
- `-Dskywalking.agent.service_name` 指定服务名
- `-Dskywalking.collector.backend_service` 指定skywalking oap地址，由于在本机，地址为：127.0.0.1:11800
- `-jar` 指定jar包的路径，这里直接放到/opt/目录了。

访问ui

http://x.x.x.x:8088/