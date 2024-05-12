---
layout: post
categories: [Action]
description: none
keywords: Action
---
# Hive优化器原理与源码解析系列—统计信息UniqueKeys列集合

## 背景

上篇介绍Hive优化器原理与源码解析系列—统计信息中间结果估算文章，TableScan，Project、Filter、Sort等等Operator操作符中间结果大小的估算受到两个因素的影响，选择率Selectivity和记录数RowCount。

但Join操作符还会受到关联两侧是否是UniqueKeys唯一键的影响。如两个RelNode进行Join时，Join返回记录数多少由的主键侧记录数选择率和外键侧非重复值的UniqueKeys唯一键共同决定的。通过对Join两侧的RelNode进行分析，确定哪一侧为重复PK side，哪一侧为含有非重复值FK side就显得异常重要了。如一张维度表DIM_DEPART部门为表、事实表FACT_EMPLOYEEE员工表两者使用DEPART_NO部门编号内关联，就JoinKey部门编号而言，维度表DIM_DEPART为非重复值FK side，事实表FACT_EMPLOYEEE员工表为重复PK side。Join的RowCount等于Math.min(1.0, 主键侧选择率 * 主键侧ndv缩放因子) * 非重复外键侧记录数。

强调一点，这里讲到主键侧PK side不是指其主键，是带有主键的那一侧，就JoinKey关联键外键而言，它是重复的，如员工表的外键部门编号就是含有重复值的，所以使用主键侧的选择率和外键的非重复记录数进行估算。

在统计信息模块在也不是对所有的列都会进行判断识别某列是否为唯一键，那样计算成本过于高昂。目前Hive统计信息模块是通过基于Project投影中用到的列进行分析判断是否UniqueKeys唯一键组成唯一键集合。

Hive优化器原理与源码解析系列—统计信息中间结果估算文章只是提到了UniqueKeys唯一键的使用，但没有展开UniqueKeys唯一键是如何识别的，接下来我们讲解分析。

UniqueKeys唯一键

1）RelNode查找TableScan操作符

传递一个RelNode树，并指定是否遍历Project投影关系表达式。从RelNode遍历查找TableScan操作符，目前只支持从Project和Filter操作符中进行查找，HeprelVertex将一个relnode包装为表示整个查询表达式的DAG中的顶点，则就取当前RelNode继续判断循环。对整颗操作符树进行自上而下遍历，直到找TableScan或null则停止并返回。强调的是，由于计算成本的考虑，既是找到TableScan，也是对TableScan的所有列进行分析判断UniqueKeys唯一键，也是基于Project投影中选择的列进行分析，下面讲解getUniqueKeys方法会用到。
```
static HiveTableScan getTableScan(RelNode r, boolean traverseProject) {

  while (r != null && !(r instanceof HiveTableScan)) {//如果不是TableScan就一直循环下去，只到叶子节点为null或TableScan

    if (r instanceof HepRelVertex) {  //HepRelVertex wraps a real RelNode as a vertex in a DAG representing the entire query expression.
      r = ((HepRelVertex) r).getCurrentRel(); //返回当前RelNode  Wrapped rel currently chosen for implementation of expression.
    } else if (r instanceof Filter) {
      r = ((Filter) r).getInput();  //Returns an array of this relational expression's inputs.
    } else if (traverseProject && r instanceof Project) {
      r = ((Project) r).getInput(); //返回此表达式输入的列表
    } else {
      r = null;
    }

  }
  return r == null ? null : (HiveTableScan) r;
}
```

2）列UniqueKeys识别

主要是从Project投影操作符中用到列进行分析判断是否为UniqueKeys列。

分两种情况：

Project操作符树中TableScan为null的情况：

遍历Project投影的输入子RelNode集合，定位RexInputRef索引信息存放到projectedCols，并找出RelNode是RexInputRef的位置索引与在Project中位置的映射关系mapInToOutPos。
如果mapInToOutPos为null，则UniqueKeys集合为null并返回。
通过RelMetadataQuery元数据信息获取子RelNode的UniqueKeys集合
遍历从元数据统计获取子节点唯一key是否是Project投影列的一部分，则存放到UniqueKeys集合并返回，如果不是则跳过。
Project中TableScan不为null的情况：

遍历Project投影的输入子RelNode集合，定位RexInputRef索引信息存放到projectedCols，并找出RelNode是RexInputRef的位置索引与在Project中位置的映射关系。
返回TableScan的记录数
根据定位RexInputRef索引信息存放到的projectedCols，从元数据信息中获取，每列的统计信息。
遍历每列的统计信息的NDV（Number of Distinct Value）与中记录数进行表，如果非重复个数大于或等于总记录数数，说明此列为UniqueKey。 另，Hive自判断统计信息范围最大值减去最小值加1，小于1.0E-5D也为UniqueKey列，把这些UniqueKey列加载到不可变位图集合并返回。
```
public Set<ImmutableBitSet> getUniqueKeys(Project rel, RelMetadataQuery mq, boolean ignoreNulls) {

  HiveTableScan tScan = getTableScan(rel.getInput(), false);

  if (tScan == null) {
    /***
     *
     * 如果TableScan没被发现，例如，没有Project或Filter操作符等序列。将会执行原始等getUniqueKeys方法。
     * LogicalProject逻辑Project映射行的集合到一个不同的集合
     * 在不知道Mapping函数（是否保留唯一的）情况下，当映射是f(a) => a。从一个Project孩子获得唯一性信息是唯一安全的。
     * 而且，来自孩子节点唯一位图，需要映射匹配Project的输出
     *
     *
     * 这里就是使用执行原始等getUniqueKeys方法来获取唯一key的方法。
     */
    final Map<Integer, Integer> mapInToOutPos = new HashMap<>();// 存放列输入和输出的位置索引映射关系
    final List<RexNode> projExprs = rel.getProjects();          //存放获取RelNode的投影RexNode
    final Set<ImmutableBitSet> projUniqueKeySet = new HashSet<>();  //存放投影中唯一键的位图集合

    // Build an input to output position map. 构建列的输入位置和输出位置的映射关系
    for (int i = 0; i < projExprs.size(); i++) {//遍历投影中行表达式RexNode
      RexNode projExpr = projExprs.get(i);
      if (projExpr instanceof RexInputRef) {//输入的rexnode是输入列引用对象（列索引，数据类型）的对象
        mapInToOutPos.put(((RexInputRef) projExpr).getIndex(), i);//取得RexInputRef的索引，和投影中的输出位置索引，构成了列输入和输出的位置映射关系
      }
    }
    if (mapInToOutPos.isEmpty()) {//如果列输入和输出的位置映射关系为null
      // if there's no RexInputRef in the projected expressions
      // return empty set.
      // 如果在投影中没有RexInputRef，则返回空的唯一键集合
      return projUniqueKeySet;
    }
    Set<ImmutableBitSet> childUniqueKeySet =
        mq.getUniqueKeys(rel.getInput(), ignoreNulls);//使用RelNode的输入，从MetaData获取唯一key，子RelNode的唯一键集合

    if (childUniqueKeySet != null) {
      // Now add to the projUniqueKeySet the child keys that are fully
      // projected. 把完全投影的孩子节点的keys添加到projUniqueKeySet
      for (ImmutableBitSet colMask : childUniqueKeySet) {   //遍历子RelNode的唯一键位图集合
        ImmutableBitSet.Builder tmpMask = ImmutableBitSet.builder();
        boolean completeKeyProjected = true;
        for (int bit : colMask) { //再遍历每个子RelNode唯一键位图，
          if (mapInToOutPos.containsKey(bit)) { //判断子节点唯一key是否是投影的一部分，如果不是则跳过
            tmpMask.set(mapInToOutPos.get(bit));
          } else {
            // Skip the child unique key if part of it is not
            // projected.
            completeKeyProjected = false; //判断条件内，boolean打标，作为下一个if的判读条件，技巧
            break;
          }
        }
        if (completeKeyProjected) { //如果是投影输入的位图信息内，则添加到projUniqueKeySet
          projUniqueKeySet.add(tmpMask.build());
        }
      }
    }

    return projUniqueKeySet;
  }
  /**
   * 以上是TableScan不存在的情况下，从拿投影Project的列的输入和输出位置映射关系和子RelNode的投影中进行比   较来筛选
   * 唯一键集合，并作为返回值
   *下面是Tablescan存在的情况：
   */

  Map<Integer, Integer> posMap = new HashMap<Integer, Integer>(); //列统计位置和投影索引映射关系
  int projectPos = 0;
  int colStatsPos = 0;

  BitSet projectedCols = new BitSet();//存放投影列位图集合

  for (RexNode r : rel.getProjects()) {
    if (r instanceof RexInputRef) {
      projectedCols.set(((RexInputRef) r).getIndex());//获取的行表达式，行直接索引直接存入bitset位图
      posMap.put(colStatsPos, projectPos);//记录列统计位置信息，和投影位置信息映射关系
      colStatsPos++;
    }
    projectPos++;
  }
  /**
   * tScan不为空的情况下：
   * 1、获取TableScna记录数
   * 2、获取Project投影列的列统计信息ColStatistics
   * 3、
   */
  double numRows = tScan.getRows(); //获取总记录数
  List<ColStatistics> colStats = tScan.getColStat(BitSets
      .toList(projectedCols));//tablescan获取指定投影列位图集合的列的统计信息列表

  Set<ImmutableBitSet> keys = new HashSet<ImmutableBitSet>();

  colStatsPos = 0;
  for (ColStatistics cStat : colStats) {//遍历每列 统计信息
    boolean isKey = false;
    if (cStat.getCountDistint() >= numRows) { //如果非重复的记录数 >= 总记录数，则为唯一键。
      isKey = true;
    }

    if ( !isKey && cStat.getRange() != null &&
        cStat.getRange().maxValue != null  &&
        cStat.getRange().minValue != null) {

      double r = cStat.getRange().maxValue.doubleValue() - 
          cStat.getRange().minValue.doubleValue() + 1;

      isKey = (Math.abs(numRows - r) < RelOptUtil.EPSILON); //RelOptUtil defines static utility methods for use in optimizing RelNodes.
                                        //EPSILON = 1.0E-5D
    }
    if ( isKey ) { // 如果上述判断是唯一键，从上述//列统计位置和投影索引映射关系中，获取投影的唯一键信息，转换为不可变位图，并加入位图集合的集合中
      ImmutableBitSet key = ImmutableBitSet.of(posMap.get(colStatsPos));//波动colStatsPos位置来遍历
      keys.add(key); //加入到唯一键集合中
    }
    colStatsPos++;//统计信息的位置递增，
  }

  return keys;//返回非重复的keys列表   判断每列是否为主键列，组成的集合并返回。
}
```
总结

    此文重要讲解了统计信息中中间结果估算中，Join操作符中间结果部分，通过选择率Selectivity和记录数RowCount，来判断哪一侧为PK Side和FK Side侧，进而使用PK side的选择率和FK Side侧非重复记录数来估算中间结果的如何获取UniqueKey的详细解释。