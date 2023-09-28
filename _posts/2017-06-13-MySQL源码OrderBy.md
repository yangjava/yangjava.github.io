---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL源码OrderBy


## OrderBy
order by的处理流程和group/window/distinct紧密相关，主要涉及的变量有

JOIN::order
JOIN::simple_order
JOIN::skip_sort_order
JOIN::m_ordered_index_usage
流程主要涉及

JOIN::optimize_distinct_group_order()
JOIN::test_skip_sort()
JOIN::make_tmp_tables_info()
这3个函数，可能有1-2张临时表参与。

optimize_distinct_group_order 这个函数中会做一些预优化处理
-> 设置simple_order，逻辑和上篇文章中simple_group一样，且如果order都是const，将order去掉 skip_sort_order = 1

-> 在distinct 转换为 group by的优化中，将order中投影的部分放在group前面，这样也是在做流式group时，对order性能的优化

-> 如果order by key是group by key前缀，去掉order

-> 单行结果集，去掉order

2. 有join buffer，simple_order = false

3. order item有UDF，simple_order = false

4. test_skip_sort中，如果没有group by但有order by时，看是否可以用index来保序，可以的话设置m_ordered_index_usage = ORDERED_INDEX_ORDER_BY

5. make_tmp_tables_info中会有各种情况，这里换个思路考虑，枚举order之前所有可能情况：

JOIN
simple_order == true: 这时need_tmp_before_win==false，不会创建tmp table
ORDERED_INDEX_ORDER_BY，则直接用索引
没有index可用，会在make_tmp_tables_info的相关逻辑中，在non-const_table上加filesort
simple_order == false: 这时need_tmp_before_win==true，创建tmp table，第1个tmp table接所有join数据后，在其上加+file sort,完成排序
Group By
如果group by + order by同时存在，两者是不同的顺序，不能使用index

无论哪种group by实现，都会创建tmp table，order会在第1个/第2个tmp table上+filesort

Windows
有group，这时simple_group==false且simple_order==false => need_tmp_before_win==true，做group by的方式同上，这时会在last window table上做filesort
没有group，这时只有windows + order
simple_order==true :
没有window_sorting:
可以用index (m_ordered_index_usage = ORDERED_INDEX_ORDER_BY) => need_tmp_before_win==false，不用tmp table，直接保证输出数据有序
无法用index时，在last window table上+filesort
有window_sorting，必然无法用index
单表或第一个windows没有sort要求 => need_tmp_before_win==false ，在last window table上+filesort
多表，第一个windows有sort要求 => need_tmp_before_win==true ，第1个tmp table接住join所有数据(第1个windows要在这个tmp table上+filesort来满足计算要求)，order仍在last window table上+filesort
simpler_order==false，必然无法用index，逻辑除了没有其上用index的部分，其余一样
DISTINCT
有group，有windows，这时 simple_order==false
distinct在last window table做，order在last window table
有group，无windows
distinct在最后一个group tmp table上做(need_duplicate_removal/table->is_distinct)，order也在这个tmp table+filesort
无group，有windows
distinct在last window table做，order在last window table
无group，无windows => need_tmp_before_win==true
会使用第1个tmp table做distinct，这时会调用JOIN::optimize_distinct()
如果之前测试可以使用index，这时就不用再做order了，order=null，也就是说这种distinct方式不会破坏index顺序!!
如果不能用index，会在做distinct的tmp table上+filesort































































































