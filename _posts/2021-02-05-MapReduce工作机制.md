---
layout: post
categories: [Hadoop]
description: none
keywords: Hadoop
---
# MapReduce工作机制
本章将从MapReduce作业的执行情况、作业运行过程中的错误机制、作业的调度策略、shuffle和排序、任务的执行等几个方面详细讲解MapReduce，让大家更加深入地了解MapReduce的运行机制，为深入学习使用Hadoop和Hadoop子项目打下基础。

## MapReduce作业的执行流程
从MapReduce编程实例中可以看出，只要在mian（）函数中调用Job的启动接口，然后将程序提交到Hadoop上，MapReduce作业就可以Hadoop上运行。另外，在前面的章节中也从Task运行角度介绍了Map和Reduce的过程。但是从运行“Hadoop JAR”到看到作业运行结果，这中间实际上还涉及很多其他细节。那么Hadoop运行MapReduce作业的完整步骤是什么呢？每一步又是如何具体实现的呢？

一个MapReduce作业的执行流程是：代码编写→作业配置→作业提交→Map任务的分配和执行→处理中间结果→Reduce任务的分配和执行→作业完成，而在每个任务的执行过程中，又包含输入准备→任务执行→输出结果。

MapReduce作业的执行可以分为11个步骤，涉及4个独立的实体。它们在MapReduce执行过程中的主要作用是：
- 客户端（Client）：编写MapReduce代码，配置作业，提交作业；
- JobTracker：初始化作业，分配作业，与TaskTracker通信，协调整个作业的执行；
- TaskTracker：保持与JobTracker的通信，在分配的数据片段上执行Map或Reduce任务，需要注意的是，图6-1中TaskTracker节点后的省略号表示Hadoop集群中可以包含多个TaskTracker；
- HDFS：保存作业的数据、配置信息等，保存作业结果。

## 提交作业





