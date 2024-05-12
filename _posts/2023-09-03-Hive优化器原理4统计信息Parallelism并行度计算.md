---
layout: post
categories: [Action]
description: none
keywords: Action
---
# Hive优化器原理与源码解析系列—统计信息Parallelism并行度计算

## 背景

Parallelism是有关RelNode关系表达式的并行度以及如何将其Opeartor运算符分配给具有独立资源池的进程的元数据。同一个Operator操作符，并行执行和串性执行相比，在成本优化器CBO看来，并行执行的成本更低。

从并行性的概念来来讲，就是将大任务划分为较小的任务，其中每个小任务被分配分配给特定处理器，以完成部分主要任务。最后，从每个小任务中获得的部分结果将合并为一个最终结果。与串行执行的一个大任务相比，并行执行多个任务可以获得性能大幅度提升！

在Hive中，Parallelism并行度计算，除了参数指定，CPU cores硬件限制，Operator算法是否可以并行执行等因素的影响，主要与如TableScan、Sort、Join等等Operator的数据大小的拆分个数splitCount计算有关。

Parallelism并行度

讲述并行度之前先熟悉执行计划中Stage划分、Phase阶段定义和PhaseTransition过渡阶段判断的定义。
```
      isPhaseTransition方法返回一个实现了RelNode的物理操作符Operator相对于其输入RelNode是否属于不同的进程。在查询管道中，在一个特定Stage中，处理所有拆分Split的操作符Operators集合，称为Phase阶段。
```
一个Phase从一个叶子节点如TableScan或phase变换节点如Exchange开始，如Hadoop的shuffle操作符跨网络发送数据是一种Sort-Exchange的形式。

Hive执行计划Stage类型
在优化HiveQL时，都会查看执行计划，这些信息含有开头Stage依赖信息说明，操作符树，统计信息记录数、数据大小等，如图

那么这些Stage大致分为几类：

MAP/REDUCE STAGE
FILESYSTEM文件系统MOVE或RENAME操作STAGE。
METASTORE STAGE 元存储，统计信息收集操作等
等等
MAP/REDUCE STAGE里还有TabelScan、Sort、Filter、Project、Aggreate、Join等等各种Oprerator操作符树构成。

        METASTORE STAGE 元存储，统计信息收集操作，如上图

Stage: Stage-2     Stats-Aggr Operator

统计信息的收集设置相关参数，在参数为true的前提下，并在执行DML语句才会收集。强调的是， LOAD DATA数据的加载不会触发统计信息的收集。

hive.stats.autogather
Default Value: true
Added In: Hive 0.7 with HIVE-1361
This flag enables gathering and updating statistics automatically during Hive DML operations.

Statistics are not gathered for LOAD DATA statements.

PhaseTransition过渡阶段判断
判读Operator操作符的输入RelNode和自己是否跨进程，即父Operator与子Operator是否在一个相同的进程里。

1）HiveJoin是否为PhaseTransition的判断

是依据Join Operator的具体实现来判断的，不能的Join 算法会返回不同结果。

HiveDefaultCostModel的Join的isPhaseTransition默认是false。

HiveTezCostModel分为四种Join算法，每种算法都有isPhaseTransition判断方法，isPhaseTransition返回值如下

Common Join：true

Map Join：false

Bucket Map Join：false

Sort Merge Bucket Join：false
```
public Boolean isPhaseTransition(HiveJoin join, RelMetadataQuery mq) {
  return join.isPhaseTransition();
}
```
2）Sort Limit是否为PhaseTransition的判断
```
public Boolean isPhaseTransition(HiveSortLimit sort, RelMetadataQuery mq) {  //HiveSortLimit 默认是true
  // As Exchange operator is introduced later on, we make a
  // sort operator create a new stage for the moment
  return true;
}
```
3）TableScan、Values、Exchange等RelNode的PhaseTransition的判断，默认值True。

SplitCount拆分数
返回数据非重复拆分数，注意splits必须是非重复的，如广播broadcast方式，其每个拷贝都是相同的，所有splitCount为1。因此，split count拆分数与由每个Operator实例发送的数据成倍数关系。

        Parallelism并行处理就是对Split数据进行并行处理，在不考虑硬件CPU core和参数限制等因素影响的情况下，Split拆分数就是并行任务的个数。

1）Join的SplitCount拆分个数计算

是依据Join Operator的具体实现来判断的，不能的Join 算法会返回split count。

HiveDefaultCostModel的Join的split count为1。

HiveTezCostModel分为四种Join算法Common Join、Map Join、Bucket Map Join和Sort Merge Bucket的split count计算逻辑相同：

都用HiveAlgorithmsUtil.getSplitCountWithoutRepartition(join)方法实现的，

splitCount = （总行数*平均记录大小） / maxSplitSize，其中maxSplitSize是HiveAlgorithmsConf算法配置项初始化的每个split大小的最大值。
```
public Integer splitCount(HiveJoin join, RelMetadataQuery mq) {
  return join.getSplitCount();
}//默认值
```
2）TableScan的SplitCount拆分个数计算

Hive中实现的StorageDescriptor存储类中方法，判断分桶个数，如果bucketCols分桶集合为null，则为0，否则分桶个数和分桶列集合
```
public List<String> getBucketCols() {
        return this.bucketCols;
    }//分桶列集合
public int getBucketColsSize() {
    return this.bucketCols == null ? 0 : this.bucketCols.size();
}
```
如果分桶列列表bucketCols不为null，使用getNumBuckets()获取分桶数作为splitCount拆分数。否则使用splitCountRepartition方法通过元数据统计信息计算出splitCount拆分数（splitCount为null，则抛出异常）。splitCountRepartition的计算逻辑在下文有讲解。
```
public Integer splitCount(HiveTableScan scan, RelMetadataQuery mq) {
  Integer splitCount;

  RelOptHiveTable table = (RelOptHiveTable) scan.getTable();
  List<String> bucketCols = table.getHiveTableMD().getBucketCols();//从表的元数据信息中，获取分桶的列的列表
  if (bucketCols != null && !bucketCols.isEmpty()) { //如果桶列的列表为空，则取桶个数，作为拆分个数
    splitCount = table.getHiveTableMD().getNumBuckets();
  } else {
    splitCount = splitCountRepartition(scan, mq); //否则取重新分区数，作为拆分个数
    if (splitCount == null) {
      throw new RuntimeException("Could not get split count for table: "
          + scan.getTable().getQualifiedName());
    }
  }
  return splitCount;
}
```
3）RelNode的SplitCount拆分个数计算

首先判断此RelNode的是否为过渡阶段Phase，如果是过渡阶段Phase，则使用splitCountRepartition方法访问元数据统计信息计算拆分数（此方法在下面有介绍）。

其次，如果不是过渡阶段Phase，则遍历此RelNode的所有输入RelNode，通过RelMetadataQuery对象获取元数据统计信息splitCount并进行累加。

总SplitCount = splitCount1 + splitCount2 + splitCount3...
```
public Integer splitCount(RelNode rel, RelMetadataQuery mq) {
  Boolean newPhase = mq.isPhaseTransition(rel);

  if (newPhase == null) {
    return null;
  }
  if (newPhase) {
    // We repartition: new number of splits 从新分区，并返回重新分区数
    return splitCountRepartition(rel, mq);
  }
  // We do not repartition: take number of splits from children
  //如果不是分区的，则从子节点中获取分区，并累加返回，拆分个数
  Integer splitCount = 0;
  for (RelNode input : rel.getInputs()) {
    splitCount += mq.splitCount(input);
  }
  return splitCount;
}
```
Repartition重新分区数计算
根据RelMetadataQuery对象获取指定RelNode的统计信息。记录数RowCount、平均记录大小等统计信息。

计算逻辑如下：

Step 1:平均记录大小AverageRowSize
Step 2:总行数RowCount
Step 3:总大小TotalSize = 每行大小 * 总行数
Step 4:重新分区个数splitCount = TotalSize / maxSplitSize
其中maxSplitSize是HiveRelMDParallelism的属性生成对象时需初始化的每个split大小的最大值。
```
public Integer splitCountRepartition(RelNode rel, RelMetadataQuery mq) {
  // We repartition: new number of splits
  final Double averageRowSize = mq.getAverageRowSize(rel);//通过元数据信息，获取平均行记录大小
  final Double rowCount = mq.getRowCount(rel); //获取记录数
  if (averageRowSize == null || rowCount == null) {
    return null;
  }
  final Double totalSize = averageRowSize * rowCount;  //计算出RelNode的总大小
  final Double splitCount = totalSize / maxSplitSize;  //。总大小 / 最大拆分大小 = 拆分数
  return splitCount.intValue();
}
```
总结

在不考虑并行度参数设置和硬件的情况下，一个Operator操作符的并行度，在允许并行执行的前提下，由splitCount拆分个数决定的，上述主要讲解了几个常用Operator的并行度的计算。