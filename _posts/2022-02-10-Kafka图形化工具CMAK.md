---
layout: post
categories: [Kafka]
description: none
keywords: Kafka
---
# Kafka图形化工具CMAK
为了简化开发者和服务工程师维护Kafka集群的工作，yahoo构建了一个叫做Kafka管理器的基于Web工具，叫做 Kafka Manager（已改名为 cmak）。

## Kafka可视化管理工具-CMAK
这个管理工具可以很容易地发现分布在集群中的哪些topic分布不均匀，或者是分区在整个集群分布不均匀的的情况。

它支持管理多个集群、选择副本、副本重新分配以及创建Topic。同时，这个管理工具也是一个非常好的可以快速浏览这个集群的工具，有如下功能：
- 管理多个kafka集群
- 便捷的检查kafka集群状态(topics,brokers,备份分布情况,分区分布情况)
- 选择你要运行的副本
- 基于当前分区状况进行
- 可以选择topic配置并创建topic(0.8.1.1和0.8.2的配置不同)
- 删除topic(只支持0.8.2以上的版本并且要在broker配置中设置delete.topic.enable=true)
- Topic list会指明哪些topic被删除（在0.8.2以上版本适用）
- 为已存在的topic增加分区
- 为已存在的topic更新配置
- 在多个topic上批量重分区
- 在多个topic上批量重分区(可选partition broker位置)

kafka-manager 项目地址：https://github.com/yahoo/kafka-manager

## 环境
注意：cmak环境要求JDK版本为11
```
1、jdk
java version "11.0.15.1"
 
2、kafka集群信息
服务器：
ip1:9092
ip2:9093
ip3:9094
软件：
kafka_2.11-2.1.1
zookeeper-3.4.14
```
项目下载地址：https://github.com/yahoo/CMAK/releases

配置文件（conf/application.conf）

修改 application.conf
将 kafka-manager.zkhosts="kafka-manager-zookeeper:2181" 中的 zookeeper 地址换成自己安装的，原配置的 kafka-manager.zkhosts ，cmak.zkhosts注释，参考下面：
```

#play.i18n.langs=["en"]
play.i18n.langs=["ch"]
 
play.http.requestHandler = "play.http.DefaultHttpRequestHandler"
play.http.context = "/"
play.application.loader=loader.KafkaManagerLoader
 
# Settings prefixed with 'kafka-manager.' will be deprecated, use 'cmak.' instead.
# https://github.com/yahoo/CMAK/issues/713
#kafka-manager.zkhosts="kafka-manager-zookeeper:2181"
#kafka-manager.zkhosts=${?ZK_HOSTS}
kafka-manager.zkhosts="192.168.10.9:2181"
#cmak.zkhosts="kafka-manager-zookeeper:2181"
#cmak.zkhosts=${?ZK_HOSTS}
cmak.zkhosts="192.168.10.9:2181"
```
开通端口
- 各个宿主机（zookeeper 开通端口/或防火墙，保证cmak 服务器可访问对应端口）
- cmak 服务器开通页面访问端口（默认9000，若有使用冲突，可启动配置其他端口）

启动

确保自己本地的ZK已经启动了之后，我们来启动Kafka-manager。

kafka-manager 默认的端口是9000。

可通过 -Dhttp.port，指定端口; -Dconfig.file=conf/application.conf指定配置文件:

临时启动：
bin/kafka-manager -Dhttp.port=10001

后台启动（最好使用脚本，存储pid）：
nohup bin/kafka-manager -Dhttp.port=10001 &

使用ip地址：端口访问测试

## 测试CMAK
点击【Cluster】>【Add Cluster】打开如下添加集群的配置界面：

输入集群的名字（如Kafka-Cluster-1）和 Zookeeper 服务器地址（如localhost:2181），选择最接近的Kafka版本（如2.2.0）

注意：如果没有在 Kafka 中配置过 JMX_PORT，千万不要选择第一个复选框。
Enable JMX Polling
如果选择了该复选框，Kafka-manager 可能会无法启动。