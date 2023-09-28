---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL源码Derived table


## Derived table
在具体介绍MySQL的derived table之前，先介绍一下子查询的概念。

在MySQL中，包含2种类型的子查询：

From字句中的子查询，例如
```
select * from (select * from t1) tt;
```
tt是一个抽象的表概念，内部就是一个子查询，在PG的概念中叫做sublink，MySQL则叫做derived table、view

其他位置的子查询，如投影列中、条件中、having中，甚至group by/order by中。。例如
```
select * from t1 where t1.c1 in (select c2 from t1);
```
这是一个在where条件中IN子查询，在PG中叫做subquery，MySQL中称为subselect

无论哪种类型的子查询，都有相关和非相关两种，这篇文章主要介绍非相关的derived table，是MySQL在5.6/5.7中就已经支持的SQL语法，在8.0中又添加了对于具有相关性的derived table(lateral table)的支持，不过这篇文章先从基本说起，代码基于MySQL 8.0.18。

## 背景
在介绍derived table在MySQL中的实现之前，需要先大概了解下MySQL中描述一条查询的逻辑语法树结构，和传统实现中的逻辑算子树略有不同的是，MySQL没有显式在代码中有对算子做统一规范的描述，结构相对不正规但也算清晰，有3个最为核心的类需要先明确：

### SELECT_LEX_UNIT
MySQL中对于一个query expression的描述结构，其中可以包含union/union all等多个query block的集合运算，同时SELECT_LEX_UNIT也根据query的结构形成递归包含关系

### SELECT_LEX
MySQL中对于一个query block的描述结构，就是我们最为熟悉的SPJ + group by + order by + select list...这样的一个查询块，一个SELECT_LEX_UNIT中可能包含多个SELECT_LEX，而SELECT_LEX内部则也可能嵌套包含其他SELECT_LEX_UNIT

### Item
MySQL中对于expression的描述结构，例如on条件/where条件/having/投影列等，都是用这个类来描述一个个表达式的，Item系统是MySQL SQL层代码中最为复杂的子系统之一，例如Item_sum描述sum函数/Item_like描述LIKE字符匹配函数。。。Item系统构成了表达式树

## 什么是derived table
派生表为直接或者间接的通过一个查询表达式从一个或者多个表中得到的表。某种意义上来讲，MySQL中的视图也是派生表。

## Derived table处理流程
MySQL对于一条查询语句的处理基本分为3个阶段，分别是prepare -> optimize -> execute，我们从3个阶段分别看下对于derived table的处理。

### Prepare阶段
这个查询的处理在经过parse，基本就构成了最初级的抽象语法树(AST)，即这种SELECT_LEX_UNIT作为root的嵌套结构，prepare阶段主要完成2件事情

负责对AST对resolve，包括所有涉及的tables/columns，以及每一个查询中的表达式(Item)
基于启发式规则，完成一些query transformation，包括将子查询转换为semi-join，outer join简化为inner join，derived table展开到上层，消除常量表达式等等。。。
这一阶段的transformation是完全基于启发式的，不考虑代价因素。这里专注于对derived table的处理流程：

MySQL中对于derived table只有两种处理方式

展开到外层query block中，等于消除掉了
物化为一个临时表供外层读取
```
-> SELECT_LEX_UNIT::prepare
   顶层query expression的处理，全局入口
-> open_tables_for_query
   需要访问dd dictionary,获取查询中涉及的所有表的元数据信息，包括schema、statistics等，但对于derived table，是一个查询内定义的逻辑表，并没有元数据，这里会跳过开表动作
-> SELECT_LEX::prepare
   对一个query block做处理
  -> resolve_placeholder_tables
     derived_table和view在查询中都属于placeholder，对其做处理
    -> resolve_derived
      -> 创建Query_result_union对象，在执行derived子查询时，用来向临时表里写入结果数据
      -> 调用内层嵌套SELECT_LEX_UNIT::prepare，对derived table对应的子查询做递归处理
        -> SELECT_LEX::prepare
          -> 判断derived中的子查询是否允许merge到外层，当满足如下任一条件时，“有可能”可以merge到外层：
             1. derived table属于最外层查询
             2. 属于最外层的子查询之中，且query是一个SELECT查询
            -> resolve_placeholder_tables 嵌套处理derived table这个子查询内部的derived table...
            ... 处理query block中其他的各个组件，包括condition/group by/rollup/distinct/order by...
            -> transform_scalar_subqueries_to_join_with_derived
               如果可能，将标量子查询转换为derived table，这个transform是为了MySQL的secondary engine而实现的，由于heat wave对于子查询的处理能力有限
               要尽可能将子查询转换为join，提升执行效率
            ... 一系列对子查询（Item中的）处理，这里先略过
            -> apply_local_transforms 做最后的一些查询变换
               1. 简化join，把嵌套的join表序列尽可能展开，去掉无用的join，outer join转inner join等
               2. 对分区表做静态剪枝
        -> 至此derived table对应的子查询部分resolve完成
    -> 判断当前derived table是否可以merge到外层，除了上面提到的基本约束外，对查询的结构要同时满足如下的要求：
        1. derived 中没有union
        2. 没有group by + 没有having + 没有distinct + 有table + 没有window + 没有limit
        可以看到MySQL对于derived 子查询merge的处理能力非常有限，只有最简单的SPJ查询可以展开到外层
    -> 确定可以展开到外层后，merge_derived 执行展开动作
       -> 再做一系列的检查看是否可以merge
          1. 外层query block是否允许merge，例如CREATE VIEW/SHOW CREATE这样的命令，不允许做merge
          2. 基于启发式，检查derived子查询的投影列是否有子查询，有则不做merge，这里就是认为用户这么定义derived table
             意思就是要做物化而不是展开的，所以不merge到外层
          3. 如果外层有straight_join，而derived子查询中有semi-join/anti-join，则不允许merge
          4. 外层表的数量达到MySQL能处理的最大值 (61个)
       -> 通过检查后，开始merge，这里有一系列复杂操作，不过总体原理很简单：
          1. 把内层join列表合并到外层中
          2. 把where条件与外层的where条件做AND组合
          3. 把投影列合并到外层投影列中
          当然代码上会涉及大量内部描述结构的修正，具体可以参考merge_derived的实现
    -> 对于不能展开的，采用物化方式执行，setup_materialized_derived
      -> setup_materialized_derived_tmp_table
        -> Query_result_union::create_result_table 创建一个存放物化结构的临时表，表列就是内层子查询的投影列
          -> create_tmp_table 创建SQL层的TABLE表对象，注意这里并不创建实际的存储引擎层的表
  -> resolve_placeholder_tables 处理完成
  -> check_view_privileges 如果是view，还有一些权限检查
  ... 其他query block中组件的处理
```
到这里derived table在prepare阶段的处理就完成了，基本思路非常简单，就是如果可以merge，就merge到外层，否则创建一个临时表对象，后续物化使用。

### Optimize阶段
到了优化阶段，MySQL主要处理无法merge到外层的derived子查询，也就是要物化的那一类，伪代码如下：
```
JOIN::optimize
  -> 对于没有merged的derived_table，调用optimize_derived先进行优化
    -> SELECT_LEX_UNIT::optimize
       ...对内层query expression做完整的优化，因为是要物化执行，必须走完优化流程
       这里的优化和MySQL对一个常规query expression的优化一致，涉及大量代码，这篇文章就不介绍了
       优化的结果会构建由Iterator组成的执行器结构，是标准的volcano执行模型，root是一个MaterializeIterator
    -> materializable_is_const判断derived table是否是一个只包含0/1行的const table
       这里判断依据前面步骤中optimize结果对于输出行数的估计estimatd_rowcount，如果估计值<=1，则认为是const
    -> 如果判断是const table，则直接进行求值
       这是MySQL优化流程非常诡异的地方，在优化期间，它会对认为求值结果很简单的子查询（包括table/subquery）直接进行计算
      -> create_materialized_table
        -> instantiate_tmp_table 真正创建存储引擎层的表，之前只是create_tmp_table建立了SQL层的TABLE对象
           MySQL8.0中对于临时结果表默认使用TempTable引擎
      -> materialize_derived
        -> 找到derived table对应的SELECT_LEX_UNIT
        -> SELECT_LEX_UNIT::exec 触发volcano执行模型，开始物化derived table
           这个是MySQL执行一个query expression的标准方法，由于在prepare阶段创建了Query_result_union对象
           也创建了存储数据的引擎表，执行中对调用Query_result_union::send_data，将结果写入物化表中
    -> 如果不是const table，这里不做执行，只生成Iterator subtree
  -> 外层优化完成，derived table对应的Iterator tree会挂在外层Iterator tree上，在外层执行时触发物化流程
```
可以看到，由于省略了对于query block优化的代码，整个过程变得非常简单，关于优化的逻辑实际就是MySQL的整个优化器，后续会专门写文章介绍。

### Execute阶段
执行阶段就是典型的volcano模型，iterator不断递归调用，从root开始触发子tree的执行，对于物化的derived table来说，对于的子树以MaterializeIterator作为root，执行流程如下：
```
MaterializeIterator::Init 完成物化流程
  -> 如果还没有建立引擎层的表，调用create_materialized_table
    -> instantiate_tmp_table 上面已经讲到了这个方法
    -> MaterializeQueryBlock
       对于MaterializeIterator来说，所有需要物化的query block放在m_query_blocks_to_materialize这个数组中
       依次调用MaterializeQueryBlock执行物化操作
      -> 获取query block对应的subquery_iterator，也就是这个qb对应执行树的root iterator
      -> subquery_iterator->init()
      -> 循环调用subquery_iterator->read() 触发执行流程
         每获取一行，调用ha_write_row写入到物化表中
    -> m_table_iterator->init() 
       在内部所有query block的物化都完成后，结果已经写入了创建的物化表中，这个m_table_iterator用来从表中读数据
MaterializeIterator::Read 从物化表中读物化结果，传递给上层iterator
  -> m_table_iterator->Read() 
```
总结
到这里对于derived table的大体处理流程已经讲完了，由于涉及代码细节非常多，这里介绍的粒度还是比较粗的，供感兴趣的同学可以进一步研究源码时作为outline。

总得来说，相比Item中的subquery，MySQL对于derived table的处理还是要更简单清晰一些的，subquery无论从优化逻辑还是执行方式，都会更加多样复杂，下一篇会专门介绍subquery。

顺便聊两句Steinar H. Gunderson从Oracle离职时对于MySQL的吐槽问题。。。估计只要是MySQL的内核开发人员，或多或少都会有些感同身受的，就是MySQL在计算层的混乱性（innodb的代码个人感觉还是很不错的，规范又高效）。这点社区的Norvald. H. Ryeng 也在一次talk中给出了历史性原因的解释，包括功能添加的随意性和初始设计的理论规范性差等等，这点从这篇介绍derived table的文章中就能略探一二：

优化以query block为单位，内部单独优化，各个query block之间是独立的，这和传统的全局一个query tree做优化完全不同，也阻止了一些高效的跨query block的transformation（aggregation pushdown, condition pullup...）
对于derived table的子查询，先优化子查询，再优化外层，而对于条件中的子查询，是先优化外层，再优化子查询，顺序相反
在优化中会对一些子查询完成计算
这只是子查询相关的一些槽点，其他还有很多。。。但无论如何，借着互联网的东风，MySQL仍然成为了当今最受欢迎的开源关系型数据库。

社区对计算层的重构由来已久，目前基本已经完成执行层的重写，还有prepare与optimize的完全解耦，但优化器的重构才刚刚开始（Gunderson是iterator执行器重构和hypergraph优化器的主要实现人员，也是计算层的主开发，他的离开会严重影响社区后续的发展。。），后续MySQL的优化器能够改造到什么程度，还要看Oracle对于MySQL的战略重视和资源投入情况了。








































































































































