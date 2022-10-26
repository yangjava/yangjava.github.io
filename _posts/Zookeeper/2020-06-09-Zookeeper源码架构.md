---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---

# Zookeeper源码解析

关于Zookeeper，目前普遍的应用场景基本作为服务注册中心，用于服务发现。但这只是Zookeeper的一个的功能，根据Apache的官方概述：“The Apache ZooKeeper system for distributed coordination is a high-performance service for building distributed applications.” Zookeeper是一个用于构建分布式应用的coordination, 并且为高性能的。Zookeeper借助于它内部的节点结构和监听机制，能用于很大部分的分布式协调场景。配置管理、命名服务、分布式锁、服务发现和发布订阅等等，这些场景在Zookeeper中基本使用其节点的“变更+通知”来实现。因为分布式的重点在于通信，通信的作用也就是协调。

Zookeeper由Java语言编写（也有C语言的Api实现）,对于其原理，算是Paxos算法的实现，包含了Leader、Follower、Proposal等角色和选举之类的一些概念，但于Paxos还有一些不同（ZAB协议）。

## 源码获取

Zookeeper源码可以从Github（https://github.com/apache/zookeeper）上clone下来；

也可从Zookeeper官网（Apache）https://zookeeper.apache.org/releases.html上获取。

Zookeeper在3.5.5之前使用的是Ant构建，在3.5.5开始使用的是Maven构建。

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