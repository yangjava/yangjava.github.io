---
layout: post
categories: [Spark]
description: none
keywords: Spark
---
# Spark源码调度系统DAGScheduler


## 面向DAG的调度器DAGScheduler
DAGScheduler实现了面向DAG的高层次调度，即将DAG中的各个RDD划分到不同的Stage。DAGScheduler可以通过计算将DAG中的一系列RDD划分到不同的Stage，然后构建这些Stage之间的父子关系，最后将每个Stage按照Partition切分为多个Task，并以Task集合（即TaskSet）的形式提交给底层的TaskScheduler。

所有的组件都通过向DAGScheduler投递DAGSchedulerEvent来使用DAGScheduler。DAGScheduler内部的DAGSchedulerEventProcessLoop将处理这些DAGScheduler-Event，并调用DAGScheduler的不同方法。JobListener用于对作业中每个Task执行成功或失败进行监听，JobWaiter实现了JobListener并最终确定作业的成功与失败。在正式介绍DAGScheduler之前，我们先来看看DAGScheduler所依赖的组件DAGSchedulerEventProcessLoop、Job Listener及ActiveJob的实现。

## JobListener与JobWaiter
JobListener定义了所有Job的监听器的接口规范，其定义如下。
```
private[spark] trait JobListener {
  def taskSucceeded(index: Int, result: Any): Unit
  def jobFailed(exception: Exception): Unit
}

```
Job执行成功后将调用JobListener定义的taskSucceeded方法，而在Job失败后调用Job Listener定义的jobFailed方法。

JobListener有JobWaiter和ApproximateActionListener两个实现类。JobWaiter用于等待整个Job执行完毕，然后调用给定的处理函数对返回结果进行处理。Approximate Action Listener只对有单一返回结果的Action（如count()和非并行的reduce()）进行监听。

JobWaiter有以下成员属性。
- dagScheduler：即DAGScheduler，当前JobWaiter等待执行完成的Job的调度者。
- jobId：当前JobWaiter等待执行完成的Job的身份标识。
- totalTasks：等待完成的Job包括的Task数量。
- resultHandler：执行结果的处理器。resultHandler是JobWaiter构造器的一个函数参数，参数定义为：resultHandler:(Int,T)=>Unit)。
- finishedTasks：等待完成的Job中已经完成的Task数量。
- jobPromise：类型为scala.concurrent.Promise。jobPromise用来代表Job完成后的结果。如果totalTasks等于零，说明没有Task需要执行，此时jobPromise将被直接设置为Success。














