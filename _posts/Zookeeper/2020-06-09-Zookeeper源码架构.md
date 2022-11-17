---
layout: post
categories: Zookeeper
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






## 概要框架设计

![Zookeeper架构图](png\zookeeper\Zookeeper架构图.png)

Zookeeper整体架构主要分为数据的存储，消息，leader选举和数据同步这几个模块。

leader选举主要是在集群处于混沌的状态下，从集群peer的提议中选择集群的leader，其他为follower或observer，维护集群peer的统一视图，保证整个集群的数据一致性，如果在leader选举成功后，存在follower日志落后的情况，则将事务日志同步给follower。

针对消息模块，peer之间的通信包需要序列化和反序列才能发送和处理，具体的消息处理由集群相应角色的消息处理器链来处理。

针对客户单的节点的创建，数据修改等操作，将会先写到内存数据库，如果有提交请求，则将数据写到事务日志，同时Zookeeper会定时将内存数据库写到快照日志，以防止没有提交的日志，在宕机的情况下丢失。

数据同步模块将leader的事务日志同步给Follower，保证整个集群数据的一致性。

## 工程结构

其中src中包含了C和Java源码，本次忽略C的Api。conf下为配置文件，也就是Zookeeper启动的配置文件。bin为Zookeeper启动脚本（server/client）。

org.apache.jute为Zookeeper的通信协议和序列化相关的组件，其通信协议基于TCP协议，它提供了Record接口用于序列化和反序列化，OutputArchive/InputArchive接口.

org.apache.zookeeper下为Zookeeper核心代码。包含了核心的业务实现。