---
layout: post
categories: [Action]
description: none
keywords: Action
---
# Hive优化器原理与源码解析系列—CBO成本模型CostModel

## 背景
关于CostModel模型相关概念请查阅上篇文章。

Hive可支持多种引擎，MR、SPARK、TEZ等，HiveDefaultCostModel是MR引擎使用的默认成本模型，通过源码分析可见默认成本模型的实现相对简单，TableScan、Aggregate、DefaultCost等Operator的CostModel成本模型计算方法都是父类继承的，默认都返回ZERO，只实现Join的成本模型计算和DefaultJoinAlgorithm（见上篇文章）。

Hive优化器原理与源码解析系列—CBO成本模型CostModel(一)

这篇文章是关于Tez引擎的CostModel成本模型：HiveTezCostModel

Tez引擎的成本模型，相对比较完善，包括HiveTableScan、HiveAggregate等Operator的成本计算，还有多种Join算法的成本计算。

接下来讲解实现了四种Join算法：CommonJoinAlgorithm，MapJoinAlgorithm，BucketJoinAlgorithm和SMBJoinAlgorithm。这四种Join算法分别对应底层三种连接算法Loop Join（嵌套连接） 、Hash Join（哈希连接）和Sort Merge Join（排序多路归并连接）。

为了便于下文对几种Join算法源码理解，这里对这些联Join Algorithms再做简要的说明：

Common Join
使用mapper按照连接键join keys上对表Table进行并行排序，然后将这些数据传递给reducers。所有具有相同键key的元组（记录）都被分配相同的reducer。一个reducer获取有多个键key获取元组（记录）。元组（记录）的键也将包含表Table ID，因此可以识别来自具有相同键key的两个不同表Table的排序输出。Reducers将Merge合并已排序的流以获得Join输出。

Map Join
此join算法将所有小表（维度表）保存在所有mapper的内存中，并将大表（事实表）放在到mapper中。对于每个小表（维度表），将使用join key键作为哈希键创建哈希表。这样就避免了上述common join关联算法内在的shuffle成本。此关联算法，对于星型模型join非常有用的。

Bucket Map Join
如果map join的连接键join key是分桶的，则替代在每个mapper内存中保留整个小表（维度表），而只保留匹配的存储桶。这会减少映射连接的内存占用。

SMB Join
SMB Join又称Sort Merge Bucket Join，是对上述Bucket Map Join关联算法的优化，如果要Join的数据已按Join key排序的，则避免创建哈希表，而是使用一个排序的sort merge join关联算法。

## 常用Operators操作符HiveCost计算
HiveCost由RowCount记录行数，CPU，IO三个指标构成的。

常用操作符的计算方法大部分由HiveAlgorithmsUtil内实现了，在上篇文章都做了详解。

### 1）TableScan的HiveCost计算

TableScan的成本：记录数RowCount、cpu默认为0，CPU:0   IO=分布式读 * 记录数 * 平均记录大小 （注意分布式读是迭代计算，其中含有Cpu成本和Net成本计算指标的）即IO=hdfsRead * cardinality * avgTupleSize

其中，

rowNumber：cardinality基数

avgTupleSize：平均元组（记录行）大小
```
public RelOptCost getScanCost(HiveTableScan ts) {
    return algoUtils.computeScanCost(ts.getRows(), RelMetadataQuery.instance().getAverageRowSize(ts));//HiveAlgorithmsUtil对象内方法
  }
```
### 2）Aggregate的HiveCost计算

判断是否为分桶的输入，如果是，则返回ZERO（记录数：0，CPU：0，IO：0）成本。否则，用RelMetadataQuery对象获取记录数RowCount，如果记录为null，则整体成本为null。都使用了Sort排序的IO、CPU成本计算方法（上篇文章CostModel（一）这些实现方法已经详解，这里不再赘述）。用HiveCost.FACTORY.makeCost工厂方法构建HiveCost作为返回值。
```
public RelOptCost getAggregateCost(HiveAggregate aggregate) {
  if (aggregate.isBucketedInput()) {
    return HiveCost.FACTORY.makeZeroCost();
  } else {
    RelMetadataQuery mq = RelMetadataQuery.instance();
    // 1. Sum of input cardinalities
    final Double rCount = mq.getRowCount(aggregate.getInput());
    if (rCount == null) {
      return null;
    }
    // 2. CPU cost = sorting cost
    final double cpuCost = algoUtils.computeSortCPUCost(rCount);//等于获取排序CPU成本
    // 3. IO cost = cost of writing intermediary results to local FS +
    //              cost of reading from local FS for transferring to GBy +
    //              cost of transferring map outputs to GBy operator
    final Double rAverageSize = mq.getAverageRowSize(aggregate.getInput());//平局记录大小
    if (rAverageSize == null) {
      return null;
    }
    final double ioCost = algoUtils.computeSortIOCost(new Pair<Double,Double>(rCount,rAverageSize));
    // 4. Result
    return HiveCost.FACTORY.makeCost(rCount, cpuCost, ioCost);
  }
}
```
### 3）DefaultCostc默认成本的HiveCost为ZERO（记录数：0，CPU：0，IO：0）成本。
```
public RelOptCost getDefaultCost() {
  return HiveCost.FACTORY.makeZeroCost();
}
```
Join Algorithms四种Join算法成本估算

HiveCostModel父类中，定义了内部静态接口JoinAlgorithm，这里四种Join算法都是对JoinAlgorithm接口的实现。成本模型由RowCount记录数、CPU、IO三部分构成的，接下来介绍四种Join算法的就从这三个方面来对估算成本进行讲解（下面计算相应Join算法的CPU、IO等方法都上篇文章讲述过，这里不再赘述，读者自行翻阅）

1） CommonJoinAlgorithm的Join成本估算

CommonJoin用Sort Merge Join 排序归并Join算法。

RowCount ：输入RelNode1左侧记录数 + 输入RelNode2右侧记录数之和

通过RelMetadataQuery对象分别获取左右两侧记录数

CPU ：通过computeSortMergeCPUCost方法左右两侧记录数集合计算CPU                成本。

IO：通过computeSortMergeIOCost方法，通过RelMetadataQuery对象获取记录数和平均记录大小，估算出IO成本。
```
public RelOptCost getCost(HiveJoin join) {
  RelMetadataQuery mq = RelMetadataQuery.instance();
  // 1. Sum of input cardinalities
  final Double leftRCount = mq.getRowCount(join.getLeft());
  final Double rightRCount = mq.getRowCount(join.getRight());
  if (leftRCount == null || rightRCount == null) {
    return null;
  }
  final double rCount = leftRCount + rightRCount;
  // 2. CPU cost = sorting cost (for each relation) +
  //               total merge cost
  ImmutableList<Double> cardinalities = new ImmutableList.Builder<Double>().
          add(leftRCount).
          add(rightRCount).
          build();
  double cpuCost;
  try {
    cpuCost = algoUtils.computeSortMergeCPUCost(cardinalities, join.getSortedInputs());
  } catch (CalciteSemanticException e) {
    LOG.trace("Failed to compute sort merge cpu cost ", e);
    return null;
  }
  // 3. IO cost = cost of writing intermediary results to local FS +
  //              cost of reading from local FS for transferring to join +
  //              cost of transferring map outputs to Join operator
  final Double leftRAverageSize = mq.getAverageRowSize(join.getLeft());
  final Double rightRAverageSize = mq.getAverageRowSize(join.getRight());
  if (leftRAverageSize == null || rightRAverageSize == null) {
    return null;
  }
  ImmutableList<Pair<Double,Double>> relationInfos = new ImmutableList.Builder<Pair<Double,Double>>().
          add(new Pair<Double,Double>(leftRCount,leftRAverageSize)).//记录数与相应记录平均大小的pair
          add(new Pair<Double,Double>(rightRCount,rightRAverageSize)).
          build();
  final double ioCost = algoUtils.computeSortMergeIOCost(relationInfos);//计算Sort Merge 的IO成本
  // 4. Result
  return HiveCost.FACTORY.makeCost(rCount, cpuCost, ioCost);
}
```
### 2）MapJoinAlgorithm的Join成本估算

Map Join 是Hash Join的实现。

RowCount ：输入RelNode1左侧记录数 + 输入RelNode2右侧记录数之和

通过RelMetadataQuery对象分别获取左右两侧记录数

CPU ：Map Join CPU成本估算只涉及基数cardinality一次计算或比较，而不涉及平均列大小。

如果为non stream表即根据join key创建HashTable保存到每个mapper的内存中的小表，需要在累加一次cpuCost。

Map Join CPU成本 = 基数 * HiveConf设置的或默认的CPU成本 的累加

IO：

relationInfos参数为Pair类型<记录数，平均记录大小>列表。

streaming参数判断是是否为流不可变BitSet

parallelism参数为并行度

遍历relationInfos列表获取基数cardinality和平均记录大小averageTupleSize，根据MapJoin算法得知non stream小表已经使用JoinKey创建了hashTable 需保存到每个mapper内存当中，涉及到多mapper、网络传输及数据大小。

Map Join IO成本 = 基数 * 平均记录大小 * 默认的网络netCost成本 * 并行度（多个mapper并行） 的累加
```
public RelOptCost getCost(HiveJoin join) {
  RelMetadataQuery mq = RelMetadataQuery.instance();
  // 1. Sum of input cardinalities
  final Double leftRCount = mq.getRowCount(join.getLeft());
  final Double rightRCount = mq.getRowCount(join.getRight());
  if (leftRCount == null || rightRCount == null) {
    return null;
  }
  final double rCount = leftRCount + rightRCount;
  // 2. CPU cost = HashTable  construction  cost  +
  //               join cost
  ImmutableList<Double> cardinalities = new ImmutableList.Builder<Double>().
          add(leftRCount).
          add(rightRCount).
          build();
  ImmutableBitSet.Builder streamingBuilder = ImmutableBitSet.builder();
  switch (join.getStreamingSide()) {
    case LEFT_RELATION:
      streamingBuilder.set(0);
      break;
    case RIGHT_RELATION:
      streamingBuilder.set(1);
      break;
    default:
      return null;
  }
  ImmutableBitSet streaming = streamingBuilder.build();
  final double cpuCost = HiveAlgorithmsUtil.computeMapJoinCPUCost(cardinalities, streaming);
  // 3. IO cost = cost of transferring small tables to join node *
  //              degree of parallelism
  final Double leftRAverageSize = mq.getAverageRowSize(join.getLeft());
  final Double rightRAverageSize = mq.getAverageRowSize(join.getRight());
  if (leftRAverageSize == null || rightRAverageSize == null) {
    return null;
  }
  ImmutableList<Pair<Double,Double>> relationInfos = new ImmutableList.Builder<Pair<Double,Double>>().
          add(new Pair<Double,Double>(leftRCount,leftRAverageSize)).
          add(new Pair<Double,Double>(rightRCount,rightRAverageSize)).
          build();
  JoinAlgorithm oldAlgo = join.getJoinAlgorithm();
  join.setJoinAlgorithm(TezMapJoinAlgorithm.INSTANCE);
  final int parallelism = mq.splitCount(join) == null
          ? 1 : mq.splitCount(join);
  join.setJoinAlgorithm(oldAlgo);
  final double ioCost = algoUtils.computeMapJoinIOCost(relationInfos, streaming, parallelism);
  // 4. Result
  return HiveCost.FACTORY.makeCost(rCount, cpuCost, ioCost);
}
```
### 3）BucketJoinAlgorithm的Join成本估算

RowCount ：输入RelNode1左侧记录数 + 输入RelNode2右侧记录数之和

通过RelMetadataQuery对象分别获取左右两侧记录数

CPU：Bucket  Join CPU成本 = 基数（非重复值个数）与初始化cpuCost的积,如果为non stream非流表即加载到内存的小表多一次cpuCost的计算

IO：这和Map Join的IO成本计算方法相同，只是Bucket Map Join是把匹配到Bucket存放到内存中，即non stream表分桶小表

Bucket  Join IO成本 = 基数 * 平均记录大小 * 默认的网络netCost成本 * 并行度 的累加
```
public RelOptCost getCost(HiveJoin join) {RelMetadataQuery mq = RelMetadataQuery.instance();
  // 1. Sum of input cardinalities
  final Double leftRCount = mq.getRowCount(join.getLeft());
  final Double rightRCount = mq.getRowCount(join.getRight());
  if (leftRCount == null || rightRCount == null) {
    return null;
  }
  final double rCount = leftRCount + rightRCount;
  // 2. CPU cost = HashTable  construction  cost  +
  //               join cost
  ImmutableList<Double> cardinalities = new ImmutableList.Builder<Double>().
          add(leftRCount).
          add(rightRCount).
          build();
  ImmutableBitSet.Builder streamingBuilder = ImmutableBitSet.builder();
  switch (join.getStreamingSide()) {
    case LEFT_RELATION:
      streamingBuilder.set(0);
      break;
    case RIGHT_RELATION:
      streamingBuilder.set(1);
      break;
    default:
      return null;
  }
  ImmutableBitSet streaming = streamingBuilder.build();
  final double cpuCost = algoUtils.computeBucketMapJoinCPUCost(cardinalities, streaming);
  // 3. IO cost = cost of transferring small tables to join node *
  //              degree of parallelism
  final Double leftRAverageSize = mq.getAverageRowSize(join.getLeft());
  final Double rightRAverageSize = mq.getAverageRowSize(join.getRight());
  if (leftRAverageSize == null || rightRAverageSize == null) {
    return null;
  }
  ImmutableList<Pair<Double,Double>> relationInfos = new ImmutableList.Builder<Pair<Double,Double>>().
          add(new Pair<Double,Double>(leftRCount,leftRAverageSize)).
          add(new Pair<Double,Double>(rightRCount,rightRAverageSize)).
          build();
  //TODO: No Of buckets is not same as no of splits
  JoinAlgorithm oldAlgo = join.getJoinAlgorithm();
  join.setJoinAlgorithm(TezBucketJoinAlgorithm.INSTANCE);
  final int parallelism = mq.splitCount(join) == null
          ? 1 : mq.splitCount(join);
  join.setJoinAlgorithm(oldAlgo);
  final double ioCost = algoUtils.computeBucketMapJoinIOCost(relationInfos, streaming, parallelism);
  // 4. Result
  return HiveCost.FACTORY.makeCost(rCount, cpuCost, ioCost);
}
```
### 4）SMBJoinAlgorithm的Join成本估算

RowCount ：输入RelNode1左侧记录数 + 输入RelNode2右侧记录数之和

通过RelMetadataQuery对象分别获取左右两侧记录数

CPU：所有基数列表遍历，基数*cpuCost的累加

IO：如果是加载到内存的桶表，涉及到IO

IO 成本 =  基数 * 平均记录大小 * 默认的网络netCost成本 * 并行度 的累加
```
public RelOptCost getCost(HiveJoin join) {
  RelMetadataQuery mq = RelMetadataQuery.instance();
  // 1. Sum of input cardinalities
  final Double leftRCount = mq.getRowCount(join.getLeft());
  final Double rightRCount = mq.getRowCount(join.getRight());
  if (leftRCount == null || rightRCount == null) {
    return null;
  }
  final double rCount = leftRCount + rightRCount;
  // 2. CPU cost = HashTable  construction  cost  +
  //               join cost
  ImmutableList<Double> cardinalities = new ImmutableList.Builder<Double>().
          add(leftRCount).
          add(rightRCount).
          build();
  ImmutableBitSet.Builder streamingBuilder = ImmutableBitSet.builder();
  switch (join.getStreamingSide()) {
    case LEFT_RELATION:
      streamingBuilder.set(0);
      break;
    case RIGHT_RELATION:
      streamingBuilder.set(1);
      break;
    default:
      return null;
  }
  ImmutableBitSet streaming = streamingBuilder.build();
  final double cpuCost = HiveAlgorithmsUtil.computeSMBMapJoinCPUCost(cardinalities);
  // 3. IO cost = cost of transferring small tables to join node *
  //              degree of parallelism
  final Double leftRAverageSize = mq.getAverageRowSize(join.getLeft());
  final Double rightRAverageSize = mq.getAverageRowSize(join.getRight());
  if (leftRAverageSize == null || rightRAverageSize == null) {
    return null;
  }
  ImmutableList<Pair<Double,Double>> relationInfos = new ImmutableList.Builder<Pair<Double,Double>>().
          add(new Pair<Double,Double>(leftRCount,leftRAverageSize)).
          add(new Pair<Double,Double>(rightRCount,rightRAverageSize)).
          build();
  // TODO: Split count is not the same as no of buckets
  JoinAlgorithm oldAlgo = join.getJoinAlgorithm();
  join.setJoinAlgorithm(TezSMBJoinAlgorithm.INSTANCE);
  final int parallelism = mq.splitCount(join) == null ? 1 : mq
      .splitCount(join);
  join.setJoinAlgorithm(oldAlgo);
  final double ioCost = algoUtils.computeSMBMapJoinIOCost(relationInfos, streaming, parallelism);
  // 4. Result
  return HiveCost.FACTORY.makeCost(rCount, cpuCost, ioCost);
}
```
总结

这篇文章是关于Tez引擎的HiveTezCostModel成本模型，包括HiveTableScan、HiveAggregate等Operator的成本计算，还有多种Join算法的成本计算。实现了四种Join算法：CommonJoinAlgorithm，MapJoinAlgorithm，BucketJoinAlgorithm和SMBJoinAlgorithm。相对于MR引擎的默认成本模型要完善了很多，越准确的成本模型越有利于CBO构建出越优化的执行计划。