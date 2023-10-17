---
layout: post
categories: [Spark]
description: none
keywords: Spark
---
# Spark源码调度Stage详解


## Stage详解
DAGScheduler会将Job的RDD划分到不同的Stage，并构建这些Stage的依赖关系。这样可以使得没有依赖关系的Stage并行执行，并保证有依赖关系的Stage顺序执行。并行执行能够有效利用集群资源，提升运行效率，而串行执行则适用于那些在时间和数据资源上存在强制依赖的场景。Stage分为需要处理Shuffle的ShuffleMapStage和最下游的ResultStage。上游Stage先于下游Stage执行，ResultStage是最后执行的Stage。

要了解Stage，应该从Stage的属性开始。Stage的属性如下。
- id：Stage的身份标识。
- rdd：当前Stage包含的RDD。
- numTasks：当前Stage的Task数量。
- parents：当前Stage的父Stage列表。这说明一个Stage可以有一到多个父亲Stage。
- firstJobId：第一个提交当前Stage的Job的身份标识（即Job的id）。当使用FIFO调度时，通过firstJobId首先计算来自较早Job的Stage，或者在发生故障时更快的恢复。
- callSite：应用程序中与当前Stage相关联的调用栈信息。
- numPartitions：当前Stage的分区数量。numPartitions实际为rdd的分区的数量。
- jobIds：当前Stage所属的Job的身份标识集合。这说明一个Stage可以属于一到多个Job。
- pendingPartitions：存储待处理分区的索引的集合。
- nextAttemptId：用于生成Stage下一次尝试的身份标识。
- _latestInfo：Stage最近一次尝试的信息，即StageInfo。
- fetchFailedAttemptIds：发生过FetchFailure的Stage尝试的身份标识的集合。此属性用于避免在发生FetchFailure后无止境的重试。

有了对Stage属性的了解，现在看看Stage提供的方法。
- clearFailures：清空fetchFailedAttemptIds。
- failedOnFetchAndShouldAbort：用于将发生FetchFailure的Stage尝试的身份标识添加到fetchFailedAttemptIds中，并返回发生FetchFailure的次数是否已经超过了允许发生FetchFailure的次数的状态。允许发生FetchFailure的次数固定为4。
- latestInfo：返回最近一次Stage尝试的StageInfo，即返回_latestInfo。
- findMissingPartitions：找到还未执行完成的分区。此方法需要子类实现。
- makeNewStageAttempt：用于创建新的Stage尝试
```
def makeNewStageAttempt(
    numPartitionsToCompute: Int,
    taskLocalityPreferences: Seq[Seq[TaskLocation]] = Seq.empty): Unit = {
  val metrics = new TaskMetrics
  metrics.register(rdd.sparkContext)
  _latestInfo = StageInfo.fromStage(
    this, nextAttemptId, Some(numPartitionsToCompute), metrics, taskLocalityPreferences)
  nextAttemptId += 1
}
```
makeNewStageAttempt的执行步骤如下。

1）调用StageInfo的fromStage方法（见代码清单7-19）创建新的StageInfo。

2）增加nextAttemptId。

抽象类Stage有两个实现子类，分别为ShuffleMapStage和ResultStage

## ResultStage的实现
ResultStage可以使用指定的函数对RDD中的分区进行计算并得出最终结果。ResultStage是最后执行的Stage，此阶段主要进行作业的收尾工作（例如，对各个分区的数据收拢、打印到控制台或写入到HDFS）。

ResultStage除继承自父类Stage的属性外，还包括以下属性。
- func：即对RDD的分区进行计算的函数。func是ResultStage的构造器参数，指定了函数的形式必须满足：
```
(TaskContext, Iterator[_]) => _
```
- partitions：由RDD的各个分区的索引组成的数组。
- _activeJob：ResultStage处理的ActiveJob。

ResultStage还提供了一些方法

```
def activeJob: Option[ActiveJob] = _activeJob
def setActiveJob(job: ActiveJob): Unit = {
  _activeJob = Option(job)
}
def removeActiveJob(): Unit = {
  _activeJob = None
}
override def findMissingPartitions(): Seq[Int] = {
  val job = activeJob.get
  (0 until job.numPartitions).filter(id => !job.finished(id))
}
```
这里特别对findMissingPartitions做一些解释：findMissingPartitions用于找出当前Job的所有分区中还没有完成的分区的索引。ResultStage判断一个分区是否完成，是通过ActiveJob的Boolean类型数组finished，因为finished记录了每个分区是否完成。

## ShuffleMapStage的实现
ShuffleMapStage是DAG调度流程的中间Stage，它可以包括一到多个ShuffleMap-Task，这些ShuffleMapTask将生成用于Shuffle的数据。ShuffleMapStage一般是ResultStage或者其他ShuffleMapStage的前置Stage，ShuffleMapTask则通过Shuffle与下游Stage中的Task串联起来。从ShuffleMapStage的命名可以看出，它将对Shuffle的数据映射到下游Stage的各个分区中。

ShuffleMapStage除继承自父类Stage的属性外，还包括以下属性。
- shuffleDep：与ShuffleMapStage相对应的ShuffleDependency。
- _mapStageJobs：与ShuffleMapStage相关联的ActiveJob的列表。
- _numAvailableOutputs：ShuffleMapStage可用的map任务的输出数量，这也代表了执行成功的map任务数。
- outputLocs：ShuffleMapStage的各个map任务与其对应的MapStatus列表的映射关系。由于map任务可能会运行多次，因而可能会有多个MapStatus。

ShuffleMapStage还提供了一些方法，分别如下。
- mapStageJobs：即读取_mapStageJobs的方法。
- addActiveJob与removeActiveJob：向ShuffleMapStage相关联的ActiveJob的列表中添加或删除ActiveJob。
```
def addActiveJob(job: ActiveJob): Unit = {
  _mapStageJobs = job :: _mapStageJobs
}

def removeActiveJob(job: ActiveJob): Unit = {
  _mapStageJobs = _mapStageJobs.filter(_ != job)
}
```
- numAvailableOutputs：即读取_numAvailableOutputs的方法。
- isAvailable：当_numAvailableOutputs与numPartitions相等时为true。也就是说，ShuffleMapStage的所有分区的map任务都执行成功后，ShuffleMapStage才是可用的。
- findMissingPartitions：找到所有还未执行成功而需要计算的分区。
```
override def findMissingPartitions(): Seq[Int] = {
  val missing = (0 until numPartitions).filter(id => outputLocs(id).isEmpty)
  assert(missing.size == numPartitions - _numAvailableOutputs,
    s"$ {missing.size} missing, expected $ {numPartitions - _numAvailableOutputs}")
  missing
}
```
- addOutputLoc：当某一分区的任务执行完成后，首先将分区与MapStatus的对应关系添加到outputLocs中，然后将可用的输出数加一。
```
def addOutputLoc(partition: Int, status: MapStatus): Unit = {
  val prevList = outputLocs(partition)
  outputLocs(partition) = status :: prevList
  if (prevList == Nil) {
    _numAvailableOutputs += 1
  }
}
```

## StageInfo
StageInfo用于描述Stage信息，并可以传递给SparkListener。StageInfo包括以下属性。
- stageId：Stage的id。
- attemptId：当前Stage尝试的id。
- name：当前Stage的名称。
- numTasks：当前Stage的Task数量。
- rddInfos：RDD信息（即RDDInfo）的序列。
- parentIds：当前Stage的父亲Stage的身份标识序列。
- details：详细的线程栈信息。
- taskMetrics：Task的度量信息。
- taskLocalityPreferences：类型为Seq[Seq[TaskLocation]]，用于存储任务的本地性偏好。
- submissionTime：DAGScheduler将当前Stage提交给TaskScheduler的时间。
- completionTime：当前Stage中的所有Task完成的时间（即Stage完成的时间）或者Stage被取消的时间。
- failureReason：如果Stage失败了，用于记录失败的原因。
- accumulables：存储了所有聚合器计算的最终值。

StageInfo提供了一个当Stage失败时要调用的方法stageFailed
```
def stageFailed(reason: String) {
  failureReason = Some(reason)
  completionTime = Some(System.currentTimeMillis)
}

```
在StageInfo的伴生对象中还提供了构建StageInfo的方法
```
def fromStage(
    stage: Stage,
    attemptId: Int,
    numTasks: Option[Int] = None,
    taskMetrics: TaskMetrics = null,
    taskLocalityPreferences: Seq[Seq[TaskLocation]] = Seq.empty
  ): StageInfo = {
  val ancestorRddInfos = stage.rdd.getNarrowAncestors.map(RDDInfo.fromRdd)
  val rddInfos = Seq(RDDInfo.fromRdd(stage.rdd)) ++ ancestorRddInfos
  new StageInfo(
    stage.id,
    attemptId,
    stage.name,
    numTasks.getOrElse(stage.numTasks),
    rddInfos,
    stage.parents.map(_.id),
    stage.details,
    taskMetrics,
    taskLocalityPreferences)
}
```
fromStage方法的执行步骤如下。

1）调用当前Stage的RDD的getNarrowAncestors方法（见代码清单7-6），获取RDD的祖先依赖中属于窄依赖的RDD序列。

2）对上一步中获得的RDD序列中的每个RDD，调用RDDInfo伴生对象的fromRdd方法（见代码清单7-12）创建RDDInfo对象。

3）给当前Stage的RDD创建对应的RDDInfo对象，将上一步中创建的所有RDDInfo对象与此RDDInfo对象放入序列rddInfos中。

4）创建StageInfo。