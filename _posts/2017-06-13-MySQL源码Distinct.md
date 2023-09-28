---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL源码Distinct


## Distinct
由于MySQL对于group by/windows/distinct/order by的实现，逻辑非常复杂混乱，完全没有清晰的代码边界，理解起来很困难，因此在对其进行梳理后，整理成这篇文档做个记录，也希望对希望了解这部分的同学有所帮助，如有理解不对的地方，请多多指正，代码基于8.0.18。

Distinct
distinct的优化比较简单，主要涉及
```
JOIN::optimize_distinct_group_order
JOIN::test_skip_sort
JOIN::make_tmp_tables_info
```
这3个函数中引入的2个tmp table 和 windows的最后一个tmp table

相关变量
```
JOIN::select_distinct
TABLE::is_distinct
QEP_TAB::need_duplicate_removal
```
distinct的处理方式有两种：

1. 使用tmp table的unique index，在insert rows时做去重，这样tmp table中数据插入完成时已去重

2. 标记QEP_TAB::need_duplicate_removal，后续通过对表中数据建立内存hash结构 或 暴力扫表来做去重，这种方式在iterator模式下，变更为filesort来去重。

和distinct优化的相关逻辑包括：

make_join_plan后，如果plan是const的，select_distinct=false
optimize_distinct_group_order中
单表，没有sum func，可以用unique index时，去掉distinct，select_distinct=false
单表，没有group/windows/sum时，尝试将distinct -> group by，select_distinct=false
优化掉group by后，结果集只有一行，select_distinct=false
make_tmp_tables_info中 (如果有windows，distinct一定在最后一个windows tmp table上，通过unique index完成去重，所以这里不考虑有windows情况)
创建第1个tmp table时
如果没有group by + 没有windows，可以用tmp table建立unique index来做去重，标记table->is_distinct = true，select_distinct = false
如果有group by，distinct的执行根据group by的执行方式，分3种情况
tmp table是全量数据，需要用第2个tmp table做stream grouping，则建立第2个tmp table，标记其need_duplicate_removal=true，表示利用表中数据(group完成)后续做去重
tmp table grouping / stream grouping，这时第1个tmp table已经完成了group计算，再分两种情况
有rollup(有rollup时不会使用tmp table grouping，这里只能是stream grouping并已完成rollup)，建立第2个tmp table，用unique index做去重，标记table->is_distinct变量
没有rollup，就标记第1个tmp table的need_duplicate_removal=true，利用分好组的数据后续做去重
这些完成后，select_distinct = false，表示distinct已经完成了

总结一下distinct的实现方式：

1. 在第1个tmp table做插入时直接去重(没有group by/windows时)，having推入qep_tab->having

2. 在第1个tmp table上标记need_duplicate_removal=true，用tmp table中已经分组的数据做去重(group by + no rollup)，having推入第1个qep_tab->having

3. 在第2个tmp table做插入时直接去重(streaming group by + rollup)，having在第1个tmp table上

4. 在第2个tmp table上标记need_duplicate_removal=true，用tmp table中已经分组的数据做去重(全量数据2阶段group by)，having推入第2个qep_tab->having

distinct的各种实现方式不破坏order的输入输出顺序！

## Window
在总结完group by/distinct的流程后，当存在windows时，它面对的输入可能有3种情况：

1. 没有group by + distinct，且是单表/多表join但第一个window不需要sorting时，输入的数据是未物化的join结果

2. 有group by + non rollup + simple_group，且是单表/多表join但第一个window不需要sorting时，输入是流式的group数据

3. 前面有tmp table，其中是join的结果数据，或是已完成group + distinct的数据

windows的实现是完全依赖tmp table的：

遍历每个windows对象

如需要frame buffering，创建一个tmp table来保存frame数据，这个table对象设置给window->m_frame_buffer
创建qep_tab对应的tmp table，作为计算结果的out table
如果当前windows有partition by (order by)，则构造order list，需要在前一个qep_tab上加filesort，按照这些item有序数据
第一个window对象如有sorting需求，如果是single table，则在上面加filesort，否则一定会建tmp table，则在最后的tmp table上加filesort。

对于第一个window，输入的3种情况：

1. 流式join结果，使用end_write_wf函数完成windows计算

2. 流式group数据，如果是precomputed_group_by，使用end_write_wf，如果是streaming group，使用end_write_group

3. 已物化tmp table，使用end_write_wf的方式




























































































































