---
layout: post
categories: [Action]
description: none
keywords: Action
---
# Hive优化器原理与源码解析系列—统计信息带谓词选择率Selectivity

## 背景
之前文章有写过关于基于Operator操作符Selectivity选择率讲解，“Hive优化器原理与源码解析系列—统计信息之选择性和基数”，其中有讲过详细讲解Cardinality基数和Selectivity选择率的计算。但这篇文章主要内容讲述stats统计信息模块关于Predicate谓词的Selectivity选择率的讲解，为了方便讲述。这里还是先简单提一下Cardinality基数和Selectivity选择率概念：

基数：某列唯一键的数量，称为基数，即某列非重复值的数量。
选择率：某列基数与总行数的比值再乘以100%，则称为某列选择率
使用Selectivity选择率来估算对应结果集的Cardinality基数的，Selectivity选择率和Cardinality之间的关系如下

Cardinality=NUM_ROWS*selectivity

其中，NUM_ROWS表示表的总行数。同时总行数Row Count也是成本模型Cost Model的记录数、IO、CPU元素之一。

基于成本优化器CBO是根据成本模型CostModel和统计信息，估算一个关系表达式RelNode成本高低，再使用动态规划算法选出整体成本最优的执行计划BestPlan。所以对于基于成本优化器的来讲，成本模型设计的是否合理和完善，统计信息收集是否准确，直接影响优化器生成的执行计划的准确性。谓词Selectivity选择率属于stats统计信息的重要组成部分。

前面文章都有提过Hive优化器模块基于Apache Calcite动态数据管理框架实现的。接下来补充一下Apache Calicte此篇文章用到简单的基础知识，后续会推出Apache Calcite知识的专题文章。

## Calcite基础知识

Apache Calcite关键术语
SQL 查询语句
SqlNode 表示为一个SQL的抽象语法树AST
RelNode 关系表达式，表示为逻辑执行计划logicPlan
RexNode 行表达式，可理解为基于字段级的表达式，select cast(a as int),id from table1中cast(a as int)，id字段的行表达式
RelCall 继承了RexNode，可理解为带有一个或多个操作数的运算符的调用表示的表达式如CASE ... WHEN ... END，cast()或 + 、-、* 、/ 加减乘除运算符的调用
一个SQL解析过程
一般数据库查询处理流程：

SQL查询提交后，数据库对SQL进行重写优化（可选），对SQL进行词法分析、语法分析再生成抽象语法树AST，绑定元数据信息Catalog进行语义验证，优化器再根据CostModel成本模型和stats统计信息来计算成本，并选出最优的执行计划，再生成物理执行计划去进行数据处理。

Apache Calcite处理流程也是类似：

Parser. Calcite通过Java CC将SQL解析成未经校验的AST
Validate. 校证Parser步骤中的AST是否合法,如验证SQL scheme、字段、函数等是否存在; SQL语句是否合法等. 生成了RelNode树
Optimize. 优化RelNode树的关键, 并将其转化成物理执行计划。主要涉及SQL规则优化如:基于规则优化(RBO)及基于代价(CBO)优化; Optimzer是可选的, 通过Validate后的RelNode树已经可以直接转化物理执行计划，但现代的SQL解析器基本上都包括有这一步，目的是优化SQL执行计划。此步得到的结果为物理执行计划。
Execute. 执行阶段。将物理执行计划转化成可在特定的平台执行的程序。如Hive与Flink都在在此阶段将物理执行计划CodeGen生成相应的可执行代码。
谓词Predicate

谓词定义：
谓词Predicate，通常为计算结果是TRUE、FALSE、UNKOWN的表达式。在SQL中的谓词，是被应用在Where从句、Having从句和Join 关联ON从句中或其他布尔值表达式中。谓词分为等值谓词、非等值谓词、常量谓词、AND连接谓词、OR连接谓词、函数谓词。

例如，SELECT * FROM EMP WHERE EMPNO = 123456;查询员工表，员工编号为123456的员工的所有信息。

在本例中，"EMPNO=123456" 就是一个谓词:

EMPNO结果不同，返回的结果为：

TRUE，如果EMPNO = 123456
FALSE，如果EMPNO 不为123456，
UNKNOW，如果EMPNO is NULL
谓词也分类可分类为等值谓词、非等值谓词、常量谓词、AND连接谓词、OR连接谓词、函数谓词
AND、OR、NOT
>, <, >=, <=, <> 或 !=
[NOT] IN
[NOT] Exists
LIKE
BETWEEN
IS [NOT] NULL
详解带谓词选择率Selectivity计算

        谓词选择率Selectivity是基于RexCall行表达式的。RexCall是对RexNode行表达式继承实现的。RexCall可理解为带有一个或多个操作数的运算符的调用表示的表达式，如a > b 表达式，表示为 ">"大于运算符对操作数a、b调用的RexCall；还如( a>b ) and ( c > b)也是RexCall。

    从RexCall来判断操作符的类型，来判断是何种谓词，在根据不同的谓词来估算不同的谓词选择率。

这里提一下Calcite框架中列引用类的定义RexInputRef，下面源码解析时会提到，它是一个输入表达式RelNode的字段引用变量。字段序号是0开始的，如果有多个字段，序号递增表示的，如join的两个输入RelNode表达式。RexInputRef(int index, RelDataType type)

例如，这里有两张表关联的例子：

    员工表（员工编号，员工名称，部门编号）

    部门表（部门编号，部门名称）

也可Calcite 中可表示为Input RelNode（TableScan）：

     Input #0: EMP(EMPNO, ENAME, DEPTNO) 

     Input #1: DEPT(DEPTNO AS DEPTNO2, DNAME)

员工表和部门表两张表作为Input RelNode输入表达式，然后两张表使用部门编号进行内关联INNER JOIN：

SELECT

    EMP.EMPNO,      

    EMP.ENAME,      

    EMP.DEPTNO,      

    DEPT.DEPTNO2，    

    DEPT.DNAME， 

FROM EMP INNER JOIN DEPT ON EMP.DEPTNO = DEPT.DEPTNO2

那么它们对应的字段名称和序号Index如下对应关系：

    Field #0: EMPNO

    Field #1: ENAME

    Field #2: DEPTNO (from EMP)

    Field #3: DEPTNO2 (from DEPT)    

    Field #4: DNAME

这里 RexInputRef(3, Integer) 是从Input RelNode输入关系表达式DEPT对字段DEPTNO2的引用，其中3是字段DEPTNO2的序号，Integer是字段的数据类型。

下面都Selectivity都会用到Input RelNode输入关系表达式的列应用信息。

### 1）从统计信息中，获取最大为NULL列的记录数MaxNulls

在HiveMeta元数据信息表TAB_COL_STATS或PART_COL_STATS收集了每列的为null的记录数，通过表的所有为null列的比较找到null列的最大记录数MaxNulls。再通过总记录TotalRowCount - MaxNulls估算出非空记录数。

从RexCall调用表达式中获取，HiveCalciteUtil.getInputRefs方法返回列引用的序号集合，在通过TableScan获取每列的统计信息ColStatistics列表，就是上述讲到TAB_COL_STATS或PART_COL_STATS收集的MaxNulls信息。求出最大值并返回。
```
private long getMaxNulls(RexCall call, HiveTableScan t) {
  long tmpNoNulls = 0;
  long maxNoNulls = 0;

  Set<Integer> iRefSet = HiveCalciteUtil.getInputRefs(call); //输入参数引用列索引号集合
  List<ColStatistics> colStats = t.getColStat(new ArrayList<Integer>(iRefSet)); //获取 输入引用列 统计信息列表，遍历这些列表，取得最大为空的号

  for (ColStatistics cs : colStats) { //遍历这些统计信息，基于列的在Hive元数据库中，Tal_col_stats 和 par_cols_stats两表内分别存放最大为空的记录数
    tmpNoNulls = cs.getNumNulls();
    if (tmpNoNulls > maxNoNulls) {
      maxNoNulls = tmpNoNulls;
    }
  }
  return maxNoNulls;
}
```
### 2）从统计信息，获取NUM_DISTINCTS每列非重复记录数

从RexCall调用表达式获取Operands操作数集合（区别于Operator操作符），如果操作数operator是RexInputRef引用列对象，则HiveRelMdDistinctRowCount.getDistinctRowCount获取列序号，从HiveMeta元数据从中获取NUM_DISTINCTS每列的非空记录数。遍历这些操作数operator的NDV（非空记录数）并从中选择最大非重复记录数。如操作数operator不是是RexInputRef引用列对象，则对操作数operator进行遍历模式找出引用的列索引，之后同上述一张找出最大非重复记录数。
```
private Double getMaxNDV(RexCall call) {
  double tmpNDV;
  double maxNDV = 1.0;
  InputReferencedVisitor irv;
  RelMetadataQuery mq = RelMetadataQuery.instance();
  for (RexNode op : call.getOperands()) {
    if (op instanceof RexInputRef) {
      tmpNDV = HiveRelMdDistinctRowCount.getDistinctRowCount(this.childRel, mq,
          ((RexInputRef) op).getIndex());//
      if (tmpNDV > maxNDV)
        maxNDV = tmpNDV;
    } else {
      irv = new InputReferencedVisitor();
      irv.apply(op);
      for (Integer childProjIndx : irv.inputPosReferenced) {
        tmpNDV = HiveRelMdDistinctRowCount.getDistinctRowCount(this.childRel,
            mq, childProjIndx);
        if (tmpNDV > maxNDV)
          maxNDV = tmpNDV;
      }
    }
  }
  return maxNDV;
}
```
### 3）常量谓词Selectivity

行表达式常量，如果常量一直为False，则选择率为0. 如果一直为True，则选择率为1，即100%
```
 //访问常量，如果是false为0，如果是true为1
  public Double visitLiteral(RexLiteral literal) {
    if (literal.isAlwaysFalse()) {
      return 0.0;
    } else if (literal.isAlwaysTrue()) {
      return 1.0;
    } else {
      assert false;
    }
    return null;
  }
}
```
### 4）AND连接的谓词的选择率Selectivity

从RexCall的操作数operand集合并遍历获取每个RexNode的Selectivity。

AND连接谓词的命中率=各个子连接谓词元素的选择率Selectivity的累乘，即谓词1的Selectivity * 谓词2的Selectivity * 谓词3的Selectivity…。如果谓词选择率为null，则选择率为100%。
```
private Double computeConjunctionSelectivity(RexCall call) {
  Double tmpSelectivity;
  double selectivity = 1;
  for (RexNode cje : call.getOperands()) {
    tmpSelectivity = cje.accept(this);//对RexVisitorImpl当前对象的遍历，并返回选择率
    if (tmpSelectivity != null) {
      selectivity *= tmpSelectivity;
    }
  }

  return selectivity;
}
```
### 5）OR连接的谓词的选择率Selectivity

选择率取值范围[0-1]，如果选择率大于1，则最大值1，即100%，如果小于0，则取值0.

从RexCall的操作数operand集合并遍历获取每个RexNode的Selectivity。如果选择率Selectivity为null，默认值0.99。用当前RelNode对象基数childCardinality计算和每个operator的选择率Selectivity计算出其基数tmpCardinality。

如果当前operator的操作数基数范围[1-childCardinality]，则当前operator的选择率Selectivity：

选择率Selectivity = 1-当前operator的基数Cardinality / 总基数。否则为100*

那么，OR连接的谓词的选择率Selectivity = 1 - AND连接的谓词的选择率Selectivity

*注：AND连接的谓词的选择率Selectivity = 所有Operator的选择率Selectivity累乘
```
private Double computeDisjunctionSelectivity(RexCall call) {
  Double tmpCardinality;
  Double tmpSelectivity;
  double selectivity = 1;

  for (RexNode dje : call.getOperands()) {
    tmpSelectivity = dje.accept(this);
    if (tmpSelectivity == null) {
      tmpSelectivity = 0.99;
    }
    tmpCardinality = childCardinality * tmpSelectivity;
    if (tmpCardinality > 1 && tmpCardinality < childCardinality) {
      tmpSelectivity = (1 - tmpCardinality / childCardinality);
    } else {
      tmpSelectivity = 1.0;//不满足条件则返回100%
    }
    selectivity *= tmpSelectivity;
  }

  if (selectivity < 0.0)
    selectivity = 0.0;

  return (1 - selectivity); //OR连接的谓词的选择率Selectivity = 1 - AND连接的谓词的选择率Selectivity
}
```
### 6）函数Functions的选择率Selectivity

通常>, >=, <, <=, =也当成Fuction函数来计算选择率Selectivity

Functions的选择率Selectivity = 1 / RexCall最大非重复个数，如f(x,y,z)选择率 =  1/maxNDV(x,y,z)。
```
private Double computeFunctionSelectivity(RexCall call) {
  return 1 / getMaxNDV(call);//求最大非重复个数，
}
```
7）非等值谓词的选择率Selectivity

非等值谓词选择率Selectivity，如<> 或 != 或 Not取反的选择率Selectivity计算。

非等值谓词的选择率Selectivity = 1 - 1/getMaxNDV(call)
```
private Double computeNotEqualitySelectivity(RexCall call) {
  double tmpNDV = getMaxNDV(call);
  if (tmpNDV > 1)
    return (tmpNDV - (double) 1) / tmpNDV;
  else
    return 1.0;
}
```
8) 计算各种谓词选择率Selectivity的汇总：

这是一个返回谓词选择率的visitCall汇总函数，通过判断RexCall谓词类型返回相应的谓词选择率，AND、OR、NOT或非等值，IS NOT NULL，IN，大于、等于、大于等于、小于、小于等于（默认选择率为1/3），其余默认谓词选择率为函数选择率。
```
public Double visitCall(RexCall call) {
  if (!deep) {
    return 1.0;
  }
  /*
   * Ignore any predicates on partition columns because we have already
   * accounted for these in the Table row count.
   * 忽略分区上的，因为已经从全局Table中取得记录数
   */
  if (isPartitionPredicate(call, this.childRel)) {//判断是否为分区上的谓词，如果是父node需要分解，递归继续调用
    return 1.0;
  }

  Double selectivity = null;
  SqlKind op = getOp(call);

  switch (op) {
  case AND: {
    selectivity = computeConjunctionSelectivity(call);//分解为and连接命中率
    break;
  }
  case OR: {
    selectivity = computeDisjunctionSelectivity(call); //分解为or连接命中率
    break;
  }
  case NOT:
  case NOT_EQUALS: {
    selectivity = computeNotEqualitySelectivity(call); //分解为非等值命中率
    break;
  }
  case IS_NOT_NULL: {
    if (childRel instanceof HiveTableScan) {
      double noOfNulls = getMaxNulls(call, (HiveTableScan) childRel);
      double totalNoOfTuples = childRel.getRows();

      if (totalNoOfTuples >= noOfNulls) {
        selectivity = (totalNoOfTuples - noOfNulls) / Math.max(totalNoOfTuples, 1);
      } else {
        throw new RuntimeException("Invalid Stats number of null > no of tuples");
      }
    } else {
      selectivity = computeNotEqualitySelectivity(call);
    }
    break;
  }

  case LESS_THAN_OR_EQUAL:
  case GREATER_THAN_OR_EQUAL:
  case LESS_THAN:
  case GREATER_THAN: {
    selectivity = ((double) 1 / (double) 3);  //小于或等于、大于或等于，小于、大于默认的命中率都为1/3
    break;
  }

  case IN: {
    // TODO: 1) check for duplicates 2) We assume in clause values to be
    // present in NDV which may not be correct (Range check can find it) 3) We
    // assume values in NDV set is uniformly distributed over col values
    // (account for skewness - histogram).
    selectivity = computeFunctionSelectivity(call) * (call.operands.size() - 1);
    if (selectivity <= 0.0) {
      selectivity = 0.10;
    } else if (selectivity >= 1.0) {
      selectivity = 1.0;
    }
    break;
  }

  default:
    selectivity = computeFunctionSelectivity(call);//默认值：1/最大不重复记录数
  }
  return selectivity;
}
```
总结

Selectivity的计算详解选择率计算的准确性是CBO构建bestPlan执行计划的很重要的一部分。谓词选择率可分类为等值谓词、非等值谓词、常量谓词、AND连接谓词、OR连接谓词、函数谓词等选择率Selectivity的计算。