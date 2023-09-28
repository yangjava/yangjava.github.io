---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL源码GroupBy


## GroupBy
对于group by的优化，主要涉及的函数：
```
JOIN::optimize_distinct_group_order
JOIN::test_skip_sort
JOIN::make_tmp_tables_info
```
相关的主要变量包括
```
JOIN::group_list
JOIN::simple_group
JOIN::streaming_aggregation
Temp_table_param::precomputed_group_by
Temp_table_param::allow_group_via_temp_table
```
变量之间相互影响，比较复杂

group by 执行方式
有5种主要执行方式，前4种都涉及到tmp table，可能的数量是1个 or 2个：

基于第一个non const table的index，也就是ORDERED_INDEX_GROUP_BY，或者在第一个non-const table上加filesort，这两种都是streaming aggregation，input数据不断更新sum_field->aggregator，直到一个group完成，写入第一个tmp table，对应函数callback是end_write_group，为方便后续说明，这里就分别叫index grouping 和 stream grouping。
基于tmp table本身做grouping，创建tmp table时要指定group items(记在table->group)，tmp table中会在group列上建立hash index做去重，通过更新table相关列来做聚集计算，这里就叫做tmp table grouping，对应函数callback是end_update。
loose index scan保证了grouping+min/max的提前完成，此时tmp_table_param->precomputed_group_by==true，只是把分组聚集结果写入第一个tmp table，对应函数callback是end_write，这里就叫pre grouping。
以上条件都不满足，只能把全量join结果写入第一个tmp table，对应函数callback也是end_write。
不需要tmp table的group by(simple_group==true)，此时后面一定没有window function(有wf时,simple_group = false)，没有distinct，没有order(JOIN::set_temp_table_need_before_win逻辑)，group by通过index/driving table filesort来完成，直接send result。
各变量设置逻辑
JOIN::group_list
描述了分组item，在JOIN::optimize_distinct_group_order中，单表且满足特定条件下，会把distinct -> group_by，生成JOIN::group_list，后续的优化宗旨是，只要完成了group by的优化，JOIN::group_list将会设为NULL。

JOIN::simple_group
表示group操作只涉及简单item，且只和首个non-const表的字段相关，因此可以做index grouping / stream grouping 其设置逻辑如下： 1) JOIN::optimize_distinct_group_order中 -> 当完成对order/distinct可能的消除和转换后，simple_group=0，这相当于初始值，此时group_list是目标group列 -> remove_const尝试消除group列中的常量和重复，此时会遍历group item并检查

有window function
group item中有aggr/wf
多表join且有outer join且有rollup
group item是投影列中子查询
group item引用外层qb表列
引用了不只第一个driving table表列 以上情况simple_group一定为false，无法简单通过index或者filesort完成分组聚集。 -> 当检测到JOIN::order是group_list的prefix时，会将JOIN::order优化掉，将direction设置到group_list上 2) JOIN::test_skip_sort -> 如果已经是simple_group，且没有select_distinct，调用test_if_skip_sort_order看是否可以使用index，可以的话设置 JOIN::m_ordered_index_usage = JOIN::ORDERED_INDEX_GROUP_BY; -> 如果有distinct，怎么也要做group结果的物化，所以不设置ORDERED_INDEX_GROUP_BY，尝试使用tmp table grouping的方式 -> 如果不使用index grouping，且允许使用tmp table grouping的方式(allow_group_via_temp_table==true)，设置simple_group = false，使用tmp table grouping
Temp_table_param::allow_group_via_temp_table
表示允许（只是允许，不强制）使用tmp table grouping的方式做group by，具体决策和sum function相关。 1) Item_sum中，aggr(distinct) / group_concat / UDF ，都不允许 2) JOIN::optimize_rollup中，会设为false，因为rollup要求输入的待group数据有序，必须使用stream/index grouping的方式。 3) optimize_distinct_group_order中，如果distinct转换出的group后紧接order，没有sum/rollup/windows，且可以用index，则allow_group_via_temp_table = false，强制使用index的方式完成group+order，这时不会使用tmp table。 4) 每次count_field_types，检查各个Item_sum，只要有不能做的，设为false。

Temp_table_param::precomputed_group_by
这个主要控制tmp table中，sum function的处理方式，如果已有预计算结果,sum无需再算，按照普通函数处理 1) test_if_skip_sort_order -> 如果post join order优化调整访问方式时，使用了loose_index_scan（单表），则设置为true，相当于初始设置 2) JOIN::make_tmp_tables_info -> 先判断下使用的loose_index_scan是否可以满足precomputed_group_by，可以则设置为true，如果有非min/max或者distinct的sum函数，则不满足 -> 创建第2个tmp table前，如果判断是使用loose_index_scan(如果之前不满足precomputed_group_by，经过第一个tmp table后，group结果已经计算出来，可以满足了)，设置precomputed_group_by=true

Temp_table_param::streaming_aggregation
表示可以基于有序的数据，做流式聚集，对应end_write_group的方式，将结果写入tmp table在开始join ordering优化前，JOIN::estimate_rowcount -> add_loose_index_scan_and_skip_scan_keys，如果没有group_list没有distinct，有aggr(distinct)且可以用loose_index_scan时，streaming_aggregation=true，每个distinct值可以有序拿到，做流式聚集。
JOIN::optimize_distinct_group_order -> 没有group_list但有sum_func时，streaming_aggregation=false
create_intermediate_table -> 如果是simple_group，则设为true
JOIN::make_tmp_tables_info -> 创建第2个tmp table做group时，会在第1个tmp table上做filesort后，使用stream grouping + aggr，设置streaming_aggregation = true
核心函数 JOIN::make_tmp_tables_info
这几个变量交互作用，影响的就是做group by的方式，具体就是tmp table的使用方式，相关函数在JOIN::make_tmp_tables_info的前两个tmp table创建流程中，其逻辑用伪代码描述

创建第1个tmp table时：
```
if (simple_group == true)  {  // 第1个tmp table不会用来做distinct
 // 使用index / driving table filesort的方式，物化group结果到tmp table中
 write_func = end_write_group;
 group_list = null;             //group by完成
} else {
 // simple_group == false时，不能使用index/stream grouping
 if (tmp table grouping) {
     // 用temp table hash index 去重
     tmp table->group = group_list;
     write_func = end_update;
 } else {
     // join result 全量数据写入
     write_func = end_write;
 }
}
```
当第一个tmp table处理完成，数据有3种状态

全量join数据，precompute_group_by和无法做任何grouping的，都是这种方式，只不过前者实际是已经做完的
stream/index grouping，已经完成group操作，group_list == NULL
tmp table grouping，也已完成group操作，group_list == NULL 针对上面第1种情况
后续处理涉及第2个tmp table：
```
if (还有rollup/windows/distinct/order) {
 // 创建1个基本不做什么的tmp table or
 // 在第1个tmp table上加filesort，对group_list排序
 // 使用stream grouping完成对第2个表的写入(end_write_group)，如果是precomputed_group_by，只是直接拷贝数据
 创建第2个tmp table
} else {
 第1个tmp table + filesort
 // 直接发送数据
 write_func = end_send_group;	
}
```
和Oracle一样，MySQL对于rollup的处理必须要求输入数据有序，因此只能使用index/stream grouping来完成， 如果在第1个tmp table上做了stream/index grouping，则在这个end_write_group时，通过rollup_write_data完成计算，否则在第2个tmp table的end_write_group中，完成rollup计算。














































































































