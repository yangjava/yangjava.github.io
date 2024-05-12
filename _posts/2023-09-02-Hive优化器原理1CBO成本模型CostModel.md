---
layout: post
categories: [Action]
description: none
keywords: Action
---
# Hive优化器原理与源码解析系列—CBO成本模型CostModel

## 背景

对于基于成本优化器CBO，除了收集统计信息如内存Momery大小、选择性Selectivity、基数Cardinal、还有是否排序Collation、是否是分布式Distribution及并行度Parallelism等物理属性作为成本估算的考虑因素外（在Calcite中，等价集合中的元素RelNode，再根据不同的物理属性细分RelSubSet，这样便于成本估算，选在出bestCost成本的RelNode），成本模型CostModel也是优化器计算一个关系表达式RelNode成本高低的重要依据。

Hive支持多种计算引擎MapReduce、Tez、Spark，还有Presto（Hive和Presto两者在SQL解析都使用Antlr）、Impala共用HiveMetaData元数据信息直接访问Hive数据等等。但除了MapReduce和Tez外，其他引擎都有自己优化器实现。

Hive成本模型是基于MapReduce和Te两种不同的引擎实现HiveDefaultCostModel和HiveOnTezCostModel实现来HiveCostModel抽象类。优化器在用户HiveConf配置的引擎信息，来使用不同的成本模型。通过源码实现来看，MR引擎使用HiveDefaultCostModel作为成本模型，其实现过于简单，没有Tez引擎的成本模型HiveOnTezCostModel那么完善。

优化器的成本模型CostModel设计的是否完善、是否科学直接决定着CBO优化器计算构建出执行计划plan是否准确，同样Hive优化器根据CostMode也是基于Hive Operator Tree操作树中节点Operator，如Join、Project、Fitlter、TableScan、Sort等等Operator来估算成本的。Hive成本模型构成：IO、记录数、CPU指标来估算的。

## 成本HiveCost
HiveCost对Calcite RelOptCost接口实现，从IO、记录数、CPU三个指标构成HiveCost成本对象的估算。定义了HiveCost的四类成本常量及工厂类来获取这些常量，成本之间的四则运算及Cost比较等方法。

- 成本常量
这些成本常量会在成本比较时，作为初始化值。如优化器比较Hive Operator Tree中一个Operator成本时，判断其是否达到了降低成本的目标时的比较值。

INFINITY：

HiveCost无限大，记录数、CPU、IO参数都为Double类型正无穷
```
public static final HiveCost INFINITY = new HiveCost(
                                  Double.POSITIVE_INFINITY,
                                  Double.POSITIVE_INFINITY,
                                  Double.POSITIVE_INFINITY)
```
HUGE：

HiveCost巨大，记录数、CPU、IO参数都为Double类型最大值
```
public static final HiveCost HUGE = new HiveCost(
                             Double.MAX_VALUE,
                             Double.MAX_VALUE,
                             Double.MAX_VALUE)
```
ZERO：

HiveCost为零，记录数、CPU、IO参数默认为0.0
```
public static final HiveCost ZERO = new HiveCost(0.0, 0.0, 0.0)
```
TINY：

HiveCost很小，记录数、CPU、IO参数默认为1.0
```
 public static final HiveCost ZERO = new HiveCost(1.0, 1.0, 1.0)
```

HiveCost成本四则运算
除了两个HiveCost进行除法运算计算稍微复杂点，其他加减乘都是记录数、CPU、IO分别加减乘生成新HiveCost返回外。

HiveCost除法计算会分别先判读两个操作数记录数、CPU、IO的是否为空、是否为无穷大条件后，再累乘记录数、CPU、IO计算结果并记录每个指标参与每次累乘次数N，将累乘结果1/N指数计算作为结果返回。
```
public double divideBy(RelOptCost cost) { 
  // Compute the geometric average of the ratios of all of the factors
  // which are non-zero and finite.
  double d = 1;
  double n = 0;
  if ((this.rowCount != 0) && !Double.isInfinite(this.rowCount) && (cost.getRows() != 0)
      && !Double.isInfinite(cost.getRows())) {
    d *= this.rowCount / cost.getRows();//判断为非无穷大，并记录数不为0的情况下进行累乘计算
    ++n;
  }
  if ((this.cpu != 0) && !Double.isInfinite(this.cpu) && (cost.getCpu() != 0)
      && !Double.isInfinite(cost.getCpu())) {
    d *= this.cpu / cost.getCpu();
    ++n;
  }
  if ((this.io != 0) && !Double.isInfinite(this.io) && (cost.getIo() != 0)
      && !Double.isInfinite(cost.getIo())) {
    d *= this.io / cost.getIo();
    ++n;
  }
  if (n == 0) {
    return 1.0;
  }
  return Math.pow(d, 1 / n);// 开1/n根号
}
```
成本模型CostModel

HiveConf配置文件相关成本估算参数配置
计算成本IO、CPU、记录HiveConf配置文件默认值：

hive.cbo.costmodel.cpu = 0.000001
CPU一次计算或比较默认成本值0.000001

hive.cbo.costmodel.network = 150.0
通过网络传输一个byte的默认成本值150.0，表示为CPU成本的150倍

hive.cbo.costmodel.local.fs.write = 4.0
向本地文件系统写一个byte的成本值4.0，表示为network传输成本的4.0倍

hive.cbo.costmodel.local.fs.read = 4.0
从本地文件系统读取一个byte的成本值4.0，表示为network传输成本的4.0倍

hive.cbo.costmodel.hdfs.write = 10.0
向HDFS分布式文件系统写一个byte的成本值10.0，表示为fs.write传输成本的10.0倍

hive.cbo.costmodel.hdfs.read = 1.5
从HDFS分布式文件系统读取一个byte的成本值1.5，表示为fs.read传输成本的1.5倍

成本计算算法公式

        在Hive成本计算的指标初始化是从HiveConf配置文件获取，各个成本指标计算逻辑如下（ HiveConf.ConfVars与上述讲述的参数按顺序对应，HIVE_CBO_COST_MODEL_CPU = hive.cbo.costmodel.cpu，依次类推）：

CPU成本 = HiveConf.ConfVars.HIVE_CBO_COST_MODEL_CPU
网络成本 = CPU成本 * HiveConf.ConfVars.HIVE_CBO_COST_MODEL_NET
本地文件写成本 = 网络成本 * HiveConf.ConfVars.HIVE_CBO_COST_MODEL_LFS_WRITE
本地文件读成本 = 网络成本 * HiveConf.ConfVars.HIVE_CBO_COST_MODEL_LFS_READ
分布式文件写成本 = 本地文件写成本 * HiveConf.ConfVars.HIVE_CBO_COST_MODEL_HDFS_WRITE
分布式文件读成本 = 本地文件读成本 * HiveConf.ConfVars.HIVE_CBO_COST_MODEL_HDFS_READ
Join Algorithms各种Join 算法
Common Join

使用mapper按照连接键join keys上对表Table进行并行排序，然后将这些数据传递给reducers。所有具有相同键key的元组（记录）都被分配相同的reducer。一个reducer获取有多个键key获取元组（记录）。元组（记录）的键也将包含表Table ID，因此可以识别来自具有相同键key的两个不同表Table的排序输出。Reducers将Merge合并已排序的流以获得Join输出。

Map Join

此关联算法，对于星型模型join非常有用的，此join算法将所有小表（维度表）保存在所有mapper的内存中，并将大表（事实表）放在到mapper中。对于每个小表（维度表），将使用join key键作为哈希键创建哈希表。这样就避免了上述common join关联算法内在的shuffle成本。

Bucket Map Join

如果map join的连接键join key是分桶的，则替代在每个mapper内存中保留整个小表（维度表），而只保留匹配的存储桶。这会减少映射连接的内存占用。

SMB Join

SMB Join又称Sort Merge Bucket Join，是对上述Bucket Map Join关联算法的优化，如果要Join的数据已按Join key排序的，则避免创建哈希表，而是使用一个排序的sort merge join关联算法。

Join Algorithms算法IO、CPU成本估算
接下来对上述各种Join 算法分别IO、CPU成本计算方式进行源码解析。

成本模型CostModel内会对JoinAlgorithms接口实现形成了Common Join、Map Join、Bucket Map Join、SMB Join等各种Join算法，成本模型CostModel在对操作符HiveJoin的Join算法集合成本比较后，选择成本最低成本算法，并设置HiveJoin要使用的哪种Join。

Sort 成本模型指标IO、CPU估算

IO成本估算：
Hive中Sort IO估算使用的是一趟排序算法，何为两趟排序算法或多趟排序算法，以后会推出相关文章详解，这里不做展开，总之，一次写，一次读，再加上中间的网络成本。

排序IO成本 = 记录数 * 平均记录大小 * 本地文件写成本

+ 记录数 * 平均记录大小 * 本地文件读成本

+记录数 * 平均记录大小 * 网络成本估算
```
public double computeSortIOCost(Pair<Double, Double> relationInfo) {
    //relationInfo：Pair类型<记录数，平均记录大小>
    // Sort-merge join
    double ioCost = 0.0;
    double cardinality = relationInfo.left;//基数
    double averageTupleSize = relationInfo.right;//平均元祖或记录大小
    // Write cost
    ioCost += cardinality * averageTupleSize * localFSWrite;
    // Read cost
    ioCost += cardinality * averageTupleSize * localFSRead;
    // Net transfer cost
    ioCost += cardinality * averageTupleSize * netCost;
    return ioCost; //返回总结果
  }
```
CPU成本估算：
排序CPU成本 = 基数  * CPU成本 * 基数自然对数

*注：基数自然对数log基数，作为排序算法复杂度来估算排序CPU成本
```
public double computeSortCPUCost(Double cardinality) {//传入参数为基数
  return cardinality * Math.log(cardinality) * cpuCost;
}
```
Sort Merge 成本模型指标IO、CPU估算

Sort Merge又称多路并归排序，两趟或多趟排序算法，这里不再展开，后续会相关文章推出。

IO成本估算：
relationInfos为Pair类型<记录数，平均记录大小>列表，然后遍历relationInfos列表，进行多路合并Sort IO成本的累加，每个Sort IO的估算可参考上述Sort IO成本指标估算方法。
```
public double computeSortMergeIOCost(
        ImmutableList<Pair<Double, Double>> relationInfos) {//Pair类型<记录数，平均记录大小>
  // Sort-merge join
  double ioCost = 0.0;
  for (Pair<Double,Double> relationInfo : relationInfos) {  //列表的遍历
    ioCost += computeSortIOCost(relationInfo);//累加了Sort IO估算
  }
  return ioCost;
}
```
CPU成本估算
计算分布式的归并排序 的CPU成本，cardinalities作为各路基数列表及对应基数sorted是否排序的位图信息。

如果当前数据无序数据，需要计算一次排序的CUP成本，

CPU成本 + = CPU成本 + 基数 * 记录数自然对数 * CPU成本。

否则，当前数据是排序的，跳过一次computeSortCPUCost累加计算，

总cpuCost = 累加 记录数 * CPU成本
```
public double computeSortMergeCPUCost(
       ImmutableList<Double> cardinalities,
       ImmutableBitSet sorted) {
 // Sort-merge join
 double cpuCost = 0.0;
 for (int i=0; i<cardinalities.size(); i++) {
   double cardinality = cardinalities.get(i);
   if (!sorted.get(i)) {//BitSet位图判断是否存在
     // Sort cost
     // 排序CPU成本 = 基数 * 基数自然对数 * CPU成本
     cpuCost += computeSortCPUCost(cardinality); //累加单个CPU成本
  }
   // Merge cost
   //合并的成本 = 记录数 * CPU成本
   cpuCost += cardinality * cpuCost; //合并cpu成本
}
 return cpuCost;
}
```
Map Join成本模型指标IO、CPU估算

如果Join关联的表有小到完全存放到内存中时，将使用Map Join，因此它非常快速，但文件大小的限制，启用hive.auto.convert.join后，hive将自动检查较小的表文件大小是否大于hive.mapjoin.smalltable.file size指定的值，然后hive将Join转换为Common Join。如果文件大小小于此阈值，它将尝试将Common Join转换为Map Join。

IO成本估算：
relationInfos参数为Pair类型<记录数，平均记录大小>列表。

streaming参数判断是是否为流不可变BitSet

parallelism参数为并行度

遍历relationInfos列表获取基数cardinality和平均记录大小averageTupleSize，根据MapJoin算法得知non stream小表已经使用JoinKey创建了hashTable 需保存到每个mapper内存当中，涉及到多mapper、网络传输及数据大小。

Map Join IO成本 = 基数 * 平均记录大小 * 默认的网络netCost成本 * 并行度（多个mapper并行） 的累加
```
public double computeMapJoinIOCost(
        ImmutableList<Pair<Double, Double>> relationInfos,////Pair类型<记录数，平均记录大小>
        ImmutableBitSet streaming, int parallelism) {
  // Hash-join
  double ioCost = 0.0;
  for (int i=0; i<relationInfos.size(); i++) {
    double cardinality = relationInfos.get(i).left; //获取基数大小
    double averageTupleSize = relationInfos.get(i).right;//平均记录大小
    if (!streaming.get(i)) {//判断为不在mapper内存中的
      ioCost += cardinality * averageTupleSize * netCost * parallelism;
    }
  }
  return ioCost;
}
```
CPU成本估算：
Map Join CPU成本估算只涉及基数cardinality一次计算或比较，而不涉及平均列大小。

如果为non stream表即根据join key创建HashTable保存到每个mapper的内存中的小表，需要在累加一次cpuCost。

Map Join CPU成本 = 基数 * HiveConf设置的或默认的CPU成本 的累加
```
public static double computeMapJoinCPUCost(
        ImmutableList<Double> cardinalities,
        ImmutableBitSet streaming) {
  // Hash-join
  double cpuCost = 0.0;
  for (int i=0; i<cardinalities.size(); i++) {
    double cardinality = cardinalities.get(i);
    if (!streaming.get(i)) {//判断进mapper内存中的
      cpuCost += cardinality;
    }
    cpuCost += cardinality * cpuCost;
  }
  return cpuCost;
}
```
Bucket Map Join成本模型指标IO、CPU估算

Bucket Map Join是应用于bucket表的一种特殊类型的map join。在Bucket Map Join中，所有关联表都必须是bucket表，并在bucket列上Join。此外，大表中的存储桶数必须是小表中存储桶数的倍数。是对Map Join的一种优化，替代在每个mapper内存中保留整个小表（维度表），而只保留匹配的存储桶。这会减少映射连接的内存占用。

IO成本估算：
这和Map Join的IO成本计算方法相同，只是Bucket Map Join是把匹配到Bucket存放到内存中，即non stream表分桶小表

Bucket  Join IO成本 = 基数 * 平均记录大小 * 默认的网络netCost成本 * 并行度 的累加
```
public double computeBucketMapJoinIOCost(
        ImmutableList<Pair<Double, Double>> relationInfos,//Pair类型<记录数，平均记录大小>
        ImmutableBitSet streaming, int parallelism) {
  // Hash-join
  double ioCost = 0.0;
  for (int i=0; i<relationInfos.size(); i++) {
    double cardinality = relationInfos.get(i).left;
    double averageTupleSize = relationInfos.get(i).right;//平均记录（元组）大小
    if (!streaming.get(i)) {
      ioCost += cardinality * averageTupleSize * netCost * parallelism;
    }
  }
  return ioCost;
}
```
CPU成本估算：
Bucket  Join CPU成本 = 基数（非重复值个数）与初始化cpuCost的积,如果为non stream非流表即加载到内存的小表多一次cpuCost的计算
```
public double computeBucketMapJoinCPUCost(
        ImmutableList<Double> cardinalities,
        ImmutableBitSet streaming) {
  // Hash-join
  double cpuCost = 0.0;
  for (int i=0; i<cardinalities.size(); i++) {
    double cardinality = cardinalities.get(i);//基数
    if (!streaming.get(i)) {
      cpuCost += cardinality * cpuCost;
    }
    cpuCost += cardinality * cpuCost;
  }
  return cpuCost;
}
```
Sort Bucket Map Join成本模型指标IO、CPU估算

SMB（Sort Bucket Map Join）是对具有相同排序、存储桶和关联条件列的bucket桶表执行的Join。它从两个bucket桶表中读取数据，并对分桶表执行common join（map和reduce触发）。

IO成本估算：
如果是加载到内存的桶表，涉及到IO

IO 成本 =  基数 * 平均记录大小 * 默认的网络netCost成本 * 并行度 的累加
```
public double computeSMBMapJoinIOCost(
        ImmutableList<Pair<Double, Double>> relationInfos, //Pair类型<记录数，平均记录大小>
        ImmutableBitSet streaming, int parallelism) {
  // Hash-join
  double ioCost = 0.0;
  for (int i=0; i<relationInfos.size(); i++) {
    double cardinality = relationInfos.get(i).left;
    double averageTupleSize = relationInfos.get(i).right;
    if (!streaming.get(i)) {
      ioCost += cardinality * averageTupleSize * netCost * parallelism;
    }
  }
  return ioCost;
}
```
CPU成本估算：
所有基数列表遍历，基数*cpuCost的累加
```
public static double computeSMBMapJoinCPUCost(
        ImmutableList<Double> cardinalities) { //基数列表
  // Hash-join
  double cpuCost = 0.0;
  for (int i=0; i<cardinalities.size(); i++) {
    cpuCost += cardinalities.get(i) * cpuCost;
  }
  return cpuCost;
}
```
物理分布类型
DISTRIBUTED是对一个RelNode关系表达式物理属性的描述，数据分布方式

ANY 任意
虽不是一个有效的分布，但是表明一个使用者将要接受任何类型的分布
BROADCAST_DISTRIBUTED 广播分布
有多个数据流实例时，并且所有的记录都会出现在实例中，即所有记录广播到所有实例
HASH_DISTRIBUTED 哈希分布
有多个数据流实例时，根据记录的keys的Hash Value散列到不同的数据流实例
RANDOM_DISTRIBUTED 随机分布
有多个数据流实例时，记录被随机分配到不同的数据流实例
RANGE_DISTRIBUTED 范围分布
有多个数据流实例时，记录根据key值范围落到不同的数据流实例
ROUND_ROBIN_DISTRIBUTED 轮询分布
有多个数据流实例时，记录按顺序依次分配到不同的数据流实例
SINGLETON 单例模式
仅有一个数据流实例
接下来详解MR引擎成本模型实现逻辑。

HiveDefaultCostModel默认成本模型

HiveDefaultCostModel是MR引擎的使用的默认成本模型，非常不完善，大部分实现还比较简陋，大多数Operator的默认HiveCost为ZERO（记录数为0、IO为0，CPU为0），Join算法也只实现一种。

HiveDefaultCostModel继承了HiveCostModel成本模型抽象类，实现了TableScan、Aggregate、DefaultCost方法，但是返回HiveCost都是0成本，通过工厂类方法返回zero常量，即IO、记录数、Cpu成本都为0的HiveCost对象。

Operator默认成本实现
```
  /***
 * 以下默认返回的都是0成本
 *
 * @return
 */
//默认成本HiveCost
@Override
public RelOptCost getDefaultCost() {
  return HiveCost.FACTORY.makeZeroCost();//new HiveCost(0.0, 0.0, 0.0)
}

//TableScan成本HiveCost
@Override
public RelOptCost getScanCost(HiveTableScan ts) {
  return HiveCost.FACTORY.makeZeroCost();//new HiveCost(0.0, 0.0, 0.0)
}

//Aggregate成本HiveCost
@Override
public RelOptCost getAggregateCost(HiveAggregate aggregate) {
  return HiveCost.FACTORY.makeZeroCost();//new HiveCost(0.0, 0.0, 0.0)
}
```
Join的HiveCost成本估算

其从HiveCostModel父类继承的Join的成本估算方法。

遍历具体实现joinAlgorithms接口的Join算法集合，选取并比较成本大小，选取最小的join成本作为返回值，并设置HiveJoin对象的当前成本最小的Join算法和成本大小值。在这里来确定Join 算法可减少优化器的搜索空间，提高效率。
```
public RelOptCost getJoinCost(HiveJoin join) { //获取join成本，选取最小成本的算法。
  // Select algorithm with min cost
  JoinAlgorithm joinAlgorithm = null;
  RelOptCost minJoinCost = null;

  if (LOG.isTraceEnabled()) {
    LOG.trace("Join algorithm selection for:\n" + RelOptUtil.toString(join));
  }

  //遍历join算法集合，选取比较大小，选取最小的join成本
  for (JoinAlgorithm possibleAlgorithm : this.joinAlgorithms) {//遍历HiveCost集合，从Join算法中选择最小的成本作为返回值
    if (!possibleAlgorithm.isExecutable(join)) {
      continue;
    }
    RelOptCost joinCost = possibleAlgorithm.getCost(join);//获取初始化的第一个成本
    if (LOG.isTraceEnabled()) {
      LOG.trace(possibleAlgorithm + " cost: " + joinCost);
    }
    if (minJoinCost == null || joinCost.isLt(minJoinCost) ) {
      joinAlgorithm = possibleAlgorithm; //次最小成本
      minJoinCost = joinCost;
    }
  }
  if (LOG.isTraceEnabled()) {
    LOG.trace(joinAlgorithm + " selected");
  }
//当前成本最小的Join算法和成本大小
  join.setJoinAlgorithm(joinAlgorithm);
  join.setJoinCost(minJoinCost);

  return minJoinCost;
}
```
DefaultJoinAlgorithm默认的Join算法实现

DefaultJoinAlgorithm对HiveCost内JoinAlgorithm接口的实现，相对Tez引擎于成本模型，Join算法实现的还是比较简陋。默认是可执行的，默认拆分个数为1，默认物理分布类型为单例，默认成本基于基数join左右两侧的记录数之和、IO为0、CPU为0。getCost方法也是实现左右两侧基数之和，IO为0、CPU为0的成本。
```
public static class DefaultJoinAlgorithm implements JoinAlgorithm {

  public static final JoinAlgorithm INSTANCE = new DefaultJoinAlgorithm();
  private static final String ALGORITHM_NAME = "none";

  @Override
  public String toString() {
    return ALGORITHM_NAME;
  }

  //默认可执行的
  @Override
  public boolean isExecutable(HiveJoin join) {
    return true;
  }
  //默认值只有行数（左右两侧记录数之和），内存，IO为0
  @Override
  public RelOptCost getCost(HiveJoin join) {
    RelMetadataQuery mq = RelMetadataQuery.instance();
    double leftRCount = mq.getRowCount(join.getLeft());//获取左侧记录数
    double rightRCount = mq.getRowCount(join.getRight());//获取右侧记录数
    return HiveCost.FACTORY.makeCost(leftRCount + rightRCount, 0.0, 0.0); //Creates a cost object
  }
  @Override
  public ImmutableList<RelCollation> getCollation(HiveJoin join) { //默认为空
    return ImmutableList.of();
  }

  //默认的分布式类型 单例
  @Override
  public RelDistribution getDistribution(HiveJoin join) {//默认值，单例
    return RelDistributions.SINGLETON; //物理分布类型，单例类型
  }
  @Override
  public Double getMemory(HiveJoin join) {//默认值，内存null为空
    return null;
  }

  @Override
  public Double getCumulativeMemoryWithinPhaseSplit(HiveJoin join) { //默认值，阶段性内存null为空
    return null;
  }
  @Override
  public Boolean isPhaseTransition(HiveJoin join) {//默认值，非事务
    return false;
  }
  //拆分个数默认为1个
  @Override
  public Integer getSplitCount(HiveJoin join) {//默认值，只有一个拆分
    return 1;
  }
}
```
总结

HiveDefaultCostModel的实现的相对于Tez引擎的成本模型CostModel来说，实现的比较简陋，大部分Operator的HiveCost默认为ZERO(记录数为0、IO为0，CPU为0),只实现了一种DefualtJoin算法，并且DefaultJoin算法内的，默认是可执行的，默认拆分个数为1，默认物理分布类型为单例。