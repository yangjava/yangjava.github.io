---
layout: post
categories: [Spark]
description: none
keywords: Spark
---
# Spark源码存储体系Block信息管理器
BlockInfoManager的确对BlockInfo进行了一些简单的管理，但是BlockInfoManager将主要对Block的锁资源进行管理。

## Block锁的基本概念
BlockInfoManager是BlockManager内部的子组件之一，BlockInfoManager对Block的锁管理采用了共享锁与排他锁，其中读锁是共享锁，写锁是排他锁。

要了解BlockInfoManager的工作原理，首先应从了解它的属性开始。

BlockInfoManager的成员属性包括以下几个。
- infos：BlockId与BlockInfo之间映射关系的缓存。
- writeLocksByTask：每次任务执行尝试的标识TaskAttemptId与执行获取的Block的写锁之间的映射关系。TaskAttemptId与写锁之间是一对多的关系，即一次任务尝试执行会获取零到多个Block的写锁。类型TaskAttemptId的本质是Long类型，其定义如下。
```
private type TaskAttemptId = Long
```
- readLocksByTask：每次任务尝试执行的标识TaskAttemptId与执行获取的Block的读锁之间的映射关系。TaskAttemptId与读锁之间是一对多的关系，即一次任务尝试执行会获取零到多个Block的读锁，并且会记录对于同一个Block的读锁的占用次数。

## Block锁的实现
有了BlockInfoManager对Block的锁管理的了解，现在来看看BlockInfoManager提供的方法是如何实现Block的锁管理机制的。