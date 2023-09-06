---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码服务Kafka


## 服务-KafkaXxxService
SkyWalking Agent上报数据有两种模式：

使用GRPC直连OAP上报数据
Agent发送数据到Kafka，OAP从Kafka中消费数据

如果使用发送Kafka的模式，Agent和OAP依然存在GRPC直连，只是大部分的采集数据上报都改为发送Kafka的模式

KafkaXxxService就是对应服务的Kafka实现，例如：GRPCChannelManager是负责Agent到OAP的网络连接，对应KafkaProducerManager是负责Agent到OAP的Kafka的连接；KafkaJVMMetricsSender负责JVM信息的发送对应GRPC的JVMMetricsSender；KafkaServiceManagementServiceClient负责Agent Client信息的上报对应GRPC的ServiceManagementClient












