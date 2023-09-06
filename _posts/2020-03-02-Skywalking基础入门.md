---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking基础入门
Skywalking是分布式系统的应用程序性能监视工具，专为微服务，云原生架构和基于容器（Docker，K8S,Mesos）架构而设计，它是一款优秀的APM（Application Performance Management）工具，包括了分布式追踪，性能指标分析和服务依赖分析等。

## 简介
Skywalking是一个国产的开源框架，2015年有吴晟个人开源，2017年加入Apache孵化器，国人开源的产品，主要开发人员来自于华为，2019年4月17日Apache董事会批准SkyWalking成为顶级项目，支持Java、.Net、NodeJs等探针，数据存储支持Mysql、Elasticsearch等，跟Pinpoint一样采用字节码注入的方式实现代码的无侵入，探针采集数据粒度粗，但性能表现优秀，且对云原生支持，目前增长势头强劲，社区活跃。

Skywalking是分布式系统的应用程序性能监视工具，专为微服务，云原生架构和基于容器（Docker，K8S,Mesos）架构而设计，它是一款优秀的APM（Application Performance Management）工具，包括了分布式追踪，性能指标分析和服务依赖分析等。

## 核心功能
- 服务、服务实例、端点指标分析
- 根本原因分析。在运行时分析代码
- 服务拓扑图分析
- 服务、服务实例和端点依赖分析
- 缓慢的服务和端点检测
- 性能优化
- 分布式跟踪和上下文传播
- 数据库访问指标。检测慢速数据库访问语句（包括SQL语句）
- 消息队列性能和消耗延迟监控
- 警报
- 浏览器性能监控
- 基础设施（VM、网络、磁盘等）监控
- 跨指标、跟踪和日志的协作
Skywalking 支持多种语言和框架，包括 Java、Go、Node.js 和 Python。它使用分布式追踪技术来监控应用程序内部和外部的所有调用，从而获得关于应用程序性能的完整见解。

Skywalking 提供了一系列强大的功能，包括性能监控、故障诊断和调试、数据分析等。它还提供了警报功能，当发生重要的性能问题时，可以向开发人员和 DevOps 团队发送通知。

Skywalking 还具有很好的可扩展性，可以与其他应用程序性能监控工具，如 Grafana 和 Elasticsearch 集成，从而提供更加强大的监控和分析能力。

总之，Apache Skywalking 是一款功能强大且易于使用的应用程序性能监控工具。它可以帮助开发人员和 DevOps 团队更好地了解应用程序的运行情况，并在发生性能问题时及时采取行动。

## SkyWalking中三个概念
- 服务(Service) ：表示对请求提供相同行为的一系列或一组工作负载，在使用Agent时，可以定义服务的名字；
- 服务实例(Service Instance) ：上述的一组工作负载中的每一个工作负载称为一个实例， 一个服务实例实际就是操作系统上的一个真实进程；
- 端点(Endpoint) ：对于特定服务所接收的请求路径, 如HTTP的URI路径和gRPC服务的类名 + 方法签名；

## 架构
SkyWalking 逻辑上分为四部分: Probes（探针）、Platform backend（平台后端）、Storage（存储） 和 用户界面（UI） 。

![SkyWalking_Architecture](https://skywalking.apache.org/images/SkyWalking_Architecture_20210424.png?t=20210424)

- 探针(Probes)
基于不同的来源可能是不一样的, 但作用都是收集数据, 将数据格式化为 SkyWalking 适用的格式。
- 平台后端（Platform backend）
支持数据聚合, 数据分析以及驱动数据流从探针到用户界面的流程。分析包括 Skywalking 原生追踪和性能指标以及第三方来源，包括 Istio 及 Envoy telemetry , Zipkin 追踪格式化等。 你甚至可以使用 Observability Analysis Language 对原生度量指标 和 用于扩展度量的计量系统 自定义聚合分析。。
- 存储（Storage）
通过开放的插件化的接口存放 SkyWalking 数据. 你可以选择一个既有的存储系统, 如 ElasticSearch, H2 或 MySQL 集群(Sharding-Sphere 管理),也可以选择自己实现一个存储系统. 当然, 我们非常欢迎你贡献新的存储系统实现。
- 用户界面（UI）
一个基于接口高度定制化的Web系统，用户可以可视化查看和管理 SkyWalking 数据。

## 下载
下载：http://skywalking.apache.org/downloads/

## SkyWalking环境搭建部署
- skywalking agent和业务系统绑定在一起，负责收集各种监控数据。
- skywalking oapservice是负责处理监控数据的，比如接受skywalking agent的监控数据，并存储在数据库中;接受skywalking webapp的前端请求，从数据库查询数据，并返回数据给前端。Skywalking oapservice通常以集群的形式存在。
- skywalking webapp，前端界面，用于展示数据。
- 用于存储监控数据的数据库，比如mysql、elasticsearch等。

## 基于Docker的Skywalking部署
Docker部署Skywalking步骤
### 部署Elasticsearch

拉取镜像
```shell
docker pull elasticsearch:7.6.2
```
指定单机启动
注：通过ES_JAVA_OPTS设置ES初始化内存，否则在验证时可能会起不来
```shell
docker run --restart=always -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms512m -Xmx512m" \
--name='elasticsearch' --cpuset-cpus="1" -m 2G -d elasticsearch:7.6.2
```
验证es安装成功
浏览器地址栏输入：http://127.0.0.1:9200/，浏览器页面显示如下内容：
```shell
{
  "name" : "6eebe74f081b",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "jgCr_SQbQXiimyAyOEqk9g",
  "version" : {
    "number" : "7.6.2",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "ef48eb35cf30adf4db14086e8aabd07ef6fb113f",
    "build_date" : "2020-03-26T06:34:37.794943Z",
    "build_snapshot" : false,
    "lucene_version" : "8.4.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```
### 部署Skywalking OAP

拉取镜像
```shell
docker pull apache/skywalking-oap-server:8.3.0-es7
```
启动Skywalking OAP
注：–link后面的第一个参数和elasticsearch容器名一致； -e SW_STORAGE_ES_CLUSTER_NODES：es7也可改为你es服务器部署的Ip地址，即ip:9200
```shell
docker run --name oap --restart always -d --restart=always -e TZ=Asia/Shanghai -p 12800:12800 -p 11800:11800 --link elasticsearch:elasticsearch -e SW_STORAGE=elasticsearch7 -e SW_STORAGE_ES_CLUSTER_NODES=elasticsearch:9200 apache/skywalking-oap-server:8.3.0-es7
```

### 部署Skywalking UI
拉取镜像
```shell
docker pull apache/skywalking-ui:8.3.0
```
启动Skywalking UI
注：–link后面的第一个参数和skywalking OAP容器名一致；
```shell
docker run -d --name skywalking-ui \
--restart=always \
-e TZ=Asia/Shanghai \
-p 8088:8080 \
--link oap:oap \
-e SW_OAP_ADDRESS=oap:12800 \
apache/skywalking-ui:8.3.0
```
### 应用程序配合Skywalking Agent部署
官网下载skywalking-agent
下载地址：https://archive.apache.org/dist/skywalking/8.3.0/

准备一个springboot程序，打成可执行jar包，写一个shell脚本，在启动项目的Shell脚本上，通过 -javaagent 参数进行配置SkyWalking Agent来跟踪微服务；startup.sh脚本：
```shell
#!/bin/sh  
# SkyWalking Agent配置  
export SW_AGENT_NAME=springboot‐skywalking‐demo #Agent名字,一般使用`spring.application.name`  
export SW_AGENT_COLLECTOR_BACKEND_SERVICES=127.0.0.1:11800 #配置 Collector 地址。  
export SW_AGENT_SPAN_LIMIT=2000 #配置链路的最大Span数量，默认为 300。  
export JAVA_AGENT=‐javaagent:/agent/skywalking‐agent.jar  
java $JAVA_AGENT ‐jar springboot‐skywalking‐demo‐0.0.1‐SNAPSHOT.jar #jar启动
```
等同于
```shell
java ‐javaagent:/agent/skywalking‐agent.jar  
‐DSW_AGENT_COLLECTOR_BACKEND_SERVICES=127.0.0.1:11800  
‐DSW_AGENT_NAME=springboot‐skywalking‐demo ‐jar springboot‐skywalking‐demo‐0.0.1‐SNAPSHOT.jar
```
参数名对应agent/config/agent.config配置文件中的属性。
属性对应的源码：org.apache.skywalking.apm.agent.core.conf.Config.java
```shell
# The service name in UI  
agent.service_name=${SW_AGENT_NAME:Your_ApplicationName}  
# Backend service addresses.  
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:127.0.0.1:11800}
```
我们也可以使用skywalking.+配置文件中的配置名作为系统配置项来进行覆盖。javaagent参数配置方式优先级更高。

## 自定义SkyWalking链路追踪
如果我们希望对项目中的业务方法，实现链路追踪，方便我们排查问题，可以使用如下的代码
引入依赖
```
<!‐‐ SkyWalking 工具类 ‐‐>  
<dependency>  
    <groupId>org.apache.skywalking</groupId>  
    <artifactId>apm‐toolkit‐trace</artifactId>  
    <version>8.6.0</version>  
</dependency>
```
### @Trace将方法加入追踪链路
如果一个业务方法想在ui界面的跟踪链路上显示出来，只需要在业务方法上加上@Trace注解即可

### 加入@Tags或@Tag
我们还可以为追踪链路增加其他额外的信息，比如记录参数和返回信息。实现方式：在方法上增加@Tag或者@Tags。@Tag 注解中 key = 方法名 ；value = returnedObj 返回值 arg[0] 参数
```
    @Trace  
    @Tag(key = "list", value = "returnedObj")  
    public List<User> list(){  
        return userMapper.list();  
    }  
  
    @Trace  
    @Tags({@Tag(key = "param", value = "arg[0]"),  
           @Tag(key = "user", value = "returnedObj")})  
    public User getById(Integer id){  
    return userMapper.getById(id);  
}
```

### 获取追踪的ID
Skywalking提供了Trace工具包，用于在追踪链路的时候进行信息的打印或者获取对应的追踪ID。
```java
import org.apache.skywalking.apm.toolkit.trace.ActiveSpan;
import org.apache.skywalking.apm.toolkit.trace.TraceContext;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;


@RestController
public class HelloController {

    @RequestMapping(value = "/hello")
    public String hello() {
        return "你好啊";
    }

    /**
     * TraceContext.traceId() 可以打印出当前追踪的ID，方便在Rocketbot中搜索
     * ActiveSpan提供了三个方法进行信息打印：
     *   error：会将本次调用转为失败状态，同时可以打印对应的堆栈信息和错误提示
     *   info：打印info级别额信息
     *   debug：打印debug级别的信息
     *
     * @return
     */
    @RequestMapping(value = "/exception")
    public String exception() {
        ActiveSpan.info("打印info信息");
        ActiveSpan.debug("打印debug信息");
        //使得当前的链路报错，并且提示报错信息
        try {
            int i = 10 / 0;
        } catch (Exception e) {
            ActiveSpan.error(new RuntimeException("报错了"));
            //返回trace id
            return TraceContext.traceId();
        }

        return "你怎么可以这个样子";
    }
}
```
访问接口，获取追踪的ID：可以搜索到对应的追踪记录，但是显示调用是失败的，这是因为使用了ActiveSpan.error方法，点开追踪的详细信息。

### Skywalking内置Tags

org.apache.skywalking.apm.agent.core.context.tag.Tags

| 取值 | 对应Tag      |
| ---- | ------------ |
| 1    | url          |
| 2    | status_code  |
| 3    | db.type      |
| 4    | db.instance  |
| 5    | db.statement |
| 6    | db.bind_vars |
| 7    | mq.queue     |
| 8    | mq.broker    |
| 9    | mq.topic     |
| 10   | http.method  |
| 11   | http.params  |
| 12   | x-le         |
| 13   | http.body    |
| 14   | http.headers |

我们提到了“skywalking收集链路时，使用的URL都是通配符，在链路中，无法针对某个pageId，或者其他通配符的具体的值进行查找。或许skywalking出于性能考虑，但是对于这种不定的通用大接口，的确无法用于针对性的性能分析了。”

那么在skywalking 8.2版本中引入的对tag搜索的支持，就能够解决这个问题，我们可以根据tag对链路进行一次过滤，得到一类的链路。让我们来看看如何配置的。
我们根据一个叫`biz.id`的标签，查找过滤其值为"bbbc"的链路。那么这种自定义标签`biz.id`，如何配置呢？（skywalking 默认支持`http.method`等标签的搜索）
- 配置oap端application.yml
skywalking分析端是一个java进程，使用application.yml作为配置文件，其位置位于“apache-skywalking-apm-bin/config”文件夹中。

core.default.searchableTracesTags 配置项为可搜索标签的配置项，其默认为：“${SW_SEARCHABLE_TAG_KEYS:http.method,status_code,db.type,db.instance,mq.queue,mq.topic,mq.broker}”。如果没有配置环境变量SW_SEARCHABLE_TAG_KEYS，那么其默认就支持这几个在skywalking 中有使用到的几个tag。 那么我在里面修改了配置，加上了我们用到的“biz.id”、“biz.type”。
修改配置后，重启skywalking oap端即可支持。
- 代码进行打标签
tracer#activeSpan()方法将会将自身作为构造去生成Span，最终仍是同一个Span。

使用H2数据库的时候，通过tag进行查询就会失效，会查不出链路，通过debug是可以看到对应的sql并无问题，拼出了biz.id的查询条件，具体原因还未查找，通过切换存储为es6解决了问题。(猜测普通的关系型数据库不支持，需要列式存储的数据库才可以)

源码分析

SegmentAnalysisListener

```
   private void appendSearchableTags(SpanObject span) {
        HashSet<Tag> segmentTags = new HashSet<>();
        span.getTagsList().forEach(tag -> {
            if (searchableTagKeys.contains(tag.getKey())) {
                final Tag spanTag = new Tag(tag.getKey(), tag.getValue());
                if (!segmentTags.contains(spanTag)) {
                    segmentTags.add(spanTag);
                }

            }
        });
        segment.getTags().addAll(segmentTags);
    }
```

### 性能分析
skywalking的性能分析，在根据服务名称、端点名称，以及相应的规则建立了任务列表后，在调用了此任务列表的端点后。skywalking会自动记录，剖析当前端口，生成剖析结果

# skywalking参考资料
Apache SkyWalking 社区: [https://skywalking.apache.org/](https://skywalking.apache.org/)

Apache SkyWalking 官方文档: [https://github.com/apache/skywalking/tree/master/docs](https://github.com/apache/skywalking/tree/master/docs)

SkyWalking 文档中文版: [https://skyapm.github.io/document-cn-translation-of-skywalking/](https://skyapm.github.io/document-cn-translation-of-skywalking/)

OpenTracing 官方标准(中文版): [https://github.com/opentracing-contrib/opentracing-specification-zh](https://github.com/opentracing-contrib/opentracing-specification-zh)

Google论文:Dapper，大规模分布式系统的跟踪系统[http://www.iocoder.cn/Fight/Dapper-translation/?self](http://www.iocoder.cn/Fight/Dapper-translation/?self)

