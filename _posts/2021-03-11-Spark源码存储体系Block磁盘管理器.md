---
layout: post
categories: [Spark]
description: none
keywords: Spark
---
# Spark源码存储体系Block磁盘管理器
DiskBlockManager是存储体系的成员之一，它负责为逻辑的Block与数据写入磁盘的位置之间建立逻辑的映射关系。

## DiskBlockManager基本概念
对于不了解的事物，我们应该先看看它定义了哪些属性。

DiskBlockManager的属性如下。
- conf：即SparkConf。
- deleteFilesOnStop：停止DiskBlockManager的时候是否删除本地目录的布尔类型标记。当不指定外部的ShuffleClient（即spark.shuffle.service.enabled属性为false）或者当前实例是Driver时，此属性为true。
- localDirs：本地目录的数组。

## 本地目录结构
localDirs是DiskBlockManager管理的本地目录数组。localDirs是通过调用createLocal-Dirs方法创建的本地目录数组，其实质是调用了Utils工具类的getConfigured-LocalDirs方法获取本地路径。

getConfiguredLocalDirs方法默认获取spark.local.dir属性或者系统属性java.io.tmpdir指定的目录，目录可能有多个，并在每个路径下创建以blockmgr-为前缀，UUID为后缀的随机字符串的子目录，例如，blockmgr-4949e19c-490c-48fc-ad6a-d80f4dbe73df。