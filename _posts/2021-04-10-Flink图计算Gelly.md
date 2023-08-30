---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink Gelly图计算应用
早在2010年，Google就推出了著名的分布式图计算框架Pregel，之后Apache Spark社区也推出GraphX等图计算组件库，以帮助用户有效满足图计算领域的需求。Flink则通过封装DataSet API，形成图计算引擎Flink Gelly。同时Gelly中的Graph API，基本涵盖了从图创建，图转换到图校验等多个方面的图操作接口，让用户能够更加简便高效地开发图计算应用。本节将重点介绍如何通过Flink Gelly组件库来构建图计算应用。

## 基本概念
在使用Flink Gelly组件库之前，需要将Flink Gelly依赖库添加到工程中。用户如果使用Maven作为项目管理工具，需要在本地工程的Pom.xml文件中添加如下Maven Dependency配置。
```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-gelly_2.11</artifactId>
  <version>1.7.0</version>
</dependency>
```
对于使用Scala语言开发Flink Gelly应用的用户，需要添加如下Maven Dependency配置。
```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-gelly-scala_2.11</artifactId>
  <version>1.7.0</version>
</dependency>
```

## 数据结构












