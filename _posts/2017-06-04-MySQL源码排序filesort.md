---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL源码排序（filesort）

本文使用源码版本：5.7.22
引擎为：Innodb
排序（filesort）作为 DBA 绕不开的话题，也经常有朋友讨论它，比如常见的问题如下：

排序的时候，用于排序的数据会不会如 Innodb 一样压缩空字符存储，比如 varchar(30)，我只是存储了 1 个字符是否会压缩，还是按照 30 个字符计算？
max_length_for_sort_data/max_sort_length 到底是什么含义？
original filesort algorithm(回表排序) 和 modified filesort algorithm(不回表排序) 的根本区别是什么？
为什么使用到排序的时候慢查询中的 Rows_examined 会更大，计算方式到底是什么样的？

在 MySQL 通常有如下算法来完成排序：

内存排序（优先队列 order by limit 返回少量行常用，提高排序效率，但是注意 order by limit n,m，如果 n 过大可能会涉及到排序算法的切换）
内存排序（快速排序）
外部排序（归并排序）
但是由于能力有限本文不解释这些算法，并且本文不考虑优先队列算法的分支逻辑，只以快速排序和归并排序作为基础进行流程剖析。我们在执行计划中如果出现 filesort 字样通常代表使用到了排序，但是执行计划中看不出来下面问题：

是否使用了临时文件。
是否使用了优先队列。
是 original filesort algorithm(回表排序) 还是 modified filesort algorithm(不回表排序)。
如何查看将在后面进行描述。本文还会给出大量的排序接口供感兴趣的朋友使用，也给自己留下笔记。

## 从一个问题出发
为了更好的说明后面的流程我们使用 2 个除了字段长度不同，其他完全一样的表来说明，但是需要注意这两个表数据量很少，不会出现外部排序，如果涉及外部排序的时候我们需要假设它们数据量很大。其次这里根据 original filesort algorithm 和 modified filesort algorithm 进行划分，但是这两种方法还没讲述，不用太多理会。

original filesort algorithm(回表排序)
```
mysql> show create table tests1 \G
*************************** 1. row ***************************
Table: tests1
CreateTable: CREATE TABLE `tests1`(
`a1` varchar(300) DEFAULT NULL,
`a2` varchar(300) DEFAULT NULL,
`a3` varchar(300) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row inset(0.00 sec)

mysql> select* from tests1;
+------+------+------+
| a1 | a2 | a3 |
+------+------+------+
| a | a | a |
| a | b | b |
| a | c | c |
| b | d | d |
| b | e | e |
| b | f | f |
| c | g | g |
| c | h | h |
+------+------+------+
8 rows inset(0.00 sec)
mysql> desc select* from tests1 where a1='b' order by a2,a3;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| id | select_type | table | partitions | type | possible_keys | key | key_len | ref| rows | filtered | Extra|
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| 1| SIMPLE | tests1 | NULL | ALL | NULL | NULL | NULL | NULL | 8| 12.50| Usingwhere; Using filesort |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
1 row inset, 1 warning (0.00 sec)
```
modified filesort algorithm(不回表排序)
```
mysql> desc select* from tests2 where a1='b' order by a2,a3;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| id | select_type | table | partitions | type | possible_keys | key | key_len | ref| rows | filtered | Extra|
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| 1| SIMPLE | tests2 | NULL | ALL | NULL | NULL | NULL | NULL | 8| 12.50| Usingwhere; Using filesort |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
1 row inset, 1 warning (0.00 sec)

mysql> show create table tests2 \G
*************************** 1. row ***************************
Table: tests2
CreateTable: CREATE TABLE `tests2`(
`a1` varchar(20) DEFAULT NULL,
`a2` varchar(20) DEFAULT NULL,
`a3` varchar(20) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row inset(0.00 sec)

mysql> select* from tests2;
+------+------+------+
| a1 | a2 | a3 |
+------+------+------+
| a | a | a |
| a | b | b |
| a | c | c |
| b | d | d |
| b | e | e |
| b | f | f |
| c | g | g |
| c | h | h |
+------+------+------+
8 rows inset(0.00 sec)

mysql> desc select* from tests2 where a1='b' order by a2,a3;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| id | select_type | table | partitions | type | possible_keys | key | key_len | ref| rows | filtered | Extra|
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| 1| SIMPLE | tests2 | NULL | ALL | NULL | NULL | NULL | NULL | 8| 12.50| Usingwhere; Using filesort |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
1 row inset, 1 warning (0.01 sec)
```
整个流程我们从 filesort 函数接口开始讨论。

## 确认排序字段及顺序
这里主要将排序顺序存入到 Filesort 类的 sortorder 中，比如我们例子中的 order by a2,a3 就是 a2 和 a3 列，主要接口为 Filesort::make_sortorder，我们按照源码描述为 sort 字段（源码中为 sort_length），显然我们在排序的时候除了 sort 字段以外，还应该包含额外的字段，到底包含哪些字段就与方法 original filesort algorithm(回表排序) 和 modified filesort algorithm(不回表排序) 有关了，下面进行讨论。

## 计算 sort 字段的长度
这里主要调用使用 sortlength 函数，这一步将会带入 max_sort_length 参数的设置进行判断，默认情况下 max_sort_length 为 1024 字节。

这一步大概步骤为：

1、循环每一个 sort 字段
2、计算每一个 sort 字段的长度：公式为 ≈ 定义长度 * 2
比如这里例子中我定义了 a1 varchar(300)，那么它的计算长度 ≈ 300 2（600），为什么是 2 呢，这应该是和 Unicode 编码有关，这一步可以参考函数 my_strnxfrmlen_utf8。

同时需要注意这里是约等于，因为源码中还是其他的考虑（比如：字符是否为空），但是占用不多不考虑了。

3、带入 max_sort_length 参数进行计算
好了，有了上面一个 sort 字段的长度，那么这里就和 max_sort_length 进行比较。

如果这个这个 sort 字段大于 max_sort_length 的值，那么以 max_sort_length 设置为准，这步代码如下：
```
set_if_smaller(sortorder->length, thd->variables.max_sort_length);
```
因此，如果 sort 字段的某个字段超过了 max_sort_length 设置，那么排序可能不那么精确了。

到了这里，每个 sort 字段的长度以及 sort 字段的总长度已经计算出来，比如前面给的两个不同列子中：

（a2 varchar(300) a3 varchar(300) order by a2,a3）：每个 sort 字段约为 300 * 2 字节，两个字段的总长度约为 1200 字节。
（a2 varchar(20) a3 varchar(20) order by a2,a3）：每个 sort 字段约为 20 * 2 字节，两个字段的总长度约为 80 字节。
并且值得注意的是，这里是按照定义大小，如 varchar(300) ，以 300 个字符来计算长度的，而不是我们通常看到的 Innodb 中实际占用的字符数量。这是排序使用空间大于 Innodb 实际数据文件大小的一个原因。

下面我们以（a2 varchar(300) a3 varchar(300) order by a2,a3）为例实际看看 debug 的结果如下：
```
(gdb) p sortorder->field->field_name
$4 = 0x7ffe7800fadf"a3"
(gdb) p sortorder->length
$5 = 600
(gdb) p total_length
$6 = 1202（这里a2,a3 可以为NULL各自加了1个字节）
(gdb)
```
可以看出没有问题。

4、循环结束，计算出 sort 字段的总长度。
后面我们会看到 sort 字段不能使用压缩（pack）技术。

## 计算额外字段的空间
对于排序而言，我们很清楚除了sort 字段以外，通常我们需要的是实际的数据，那么无外乎两种方式如下：

original filesort algorithm：只存储 rowid 或者主键做为额外的字段，然后进行回表抽取数据。我们按照源码的描述，将这种关联回表的字段叫做 ref 字段（源码中变量叫做 ref_length）。
modified filesort algorithm：将处于 read_set(需要读取的字段) 全部放到额外字段中，这样不需要回表读取数据了。我们按照源码的描述，将这些额外存储的字段叫做 addon 字段（源码中变量叫做 addon_length）。
这里一步就是要来判断到底使用那种算法，其主要标准就是参数 max_length_for_sort_data，其默认大小为 1024 字节，但是后面会看到这里的计算为（sort字段长度 + addon 字段的总和）是否超过了 max_length_for_sort_data。其次如果使用了 modified filesort algorithm 算法，那么将会对 addon 字段的每个字段做一个 pack(打包)，主要目的在于压缩那些为空的字节，节省空间。

这一步的主要入口函数为 Filesort::get_addon_fields 下面是步骤解析。

1、循环本表全部字段
2、根据 read_set 过滤出不需要存储的字段
这里如果不需要访问到的字段自然不会包含在其中，下面这段源码过滤代码：
```
if(!bitmap_is_set(read_set, field->field_index)) //是否在read set中
continue;
```

3、获取字段的长度
这里就是实际的长度了比如我们的 a1 varchar(300)，且字符集为 UTF8，那么其长度 ≈ 300 * 3（900）。

4、获取可以 pack(打包) 字段的长度
和上面不同，对于 int 这些固定长度类型的字段，只有可变长度的类型的字段才需要进行打包技术。

5、循环结束，获取 addon 字段的总长度，获取可以 pack(打包) 字段的总长度
循环结束后可以获取 addon 字段的总长度，但是需要注意 addon 字段和 sort 字段可能包含重复的字段，比如例 2 中 sort 字段为 a2,a3，addon 字段为 a1,a2,a3。

如果满足如下条件：

addon 字段的总长度 + sort 字段的总长度 > max_length_for_sort_data
那么将使用 original filesort algorithm(回表排序) 的方式，否则使用 modified filesort algorithm 的方式进行。下面是这一句代码：
```
if(total_length + sortlength > max_length_for_sort_data) //如果长度大于了max_length_for_sort_data 则退出了
{
DBUG_ASSERT(addon_fields == NULL);
return NULL;
//返回NULL值 不打包了 使用 original filesort algorithm（回表排序）
}
```
我们在回到本文第二部分例子中的第 1 个案例，因为我们对 a1,a2,a3 都是需要访问的，且他们的大小均为 varchar(300) UTF8，那么 addon 字段长度大约为 300 3 3 = 2700 字节 ，其次我们前面计算了 sort 字段大约为 1202 字节，因此 2700 + 1202 是远远大于 max_length_for_sort_data 的默认设置 1024 字节的，因此会使用 original filesort algorithm 方式进行排序。

如果是第二部分例子中的第 2 个案例呢？显然要小很多了（每个字段 varchar(20)），大约就是 20 3 3（addon 字段）+ 82（sort 字段） 它是小于 1024 字节的，因此会使用 modified filesort algorithm 的排序方式，并且这些 addon 字段基本都可以使用打包（pack）技术，来节省空间。但是需要注意的是无论如何（sort 字段）是不能进行打包（pack）的，而固定长度类型不需要打包（pack）压缩空间。

## 确认每行的长度 
有了上面的就计算后每一行的长度（如果可以打包是打包前的长度），下面就是这个计算过程。
```
if(using_addon_fields())
//如果使用了 打包技术 检测 addon_fields 数组是否存在 使用modified filesort algorithm算法 不回表排序
{
res_length= addon_length; //总的长度 3个 varchar(300) uft8 为 3*300*3
}
else//使用original filesort algorithm算法
{
res_length= ref_length; //rowid(主键长度)
/*
The reference to the record is considered
as an additional sorted field
*/
sort_length+= ref_length; //实际上就是rowid(主键) +排序字段长度 回表排序
}
/*
Add hash at the end of sort key to order cut values correctly.
Needed for GROUPing, rather than for ORDERing.
*/
if(use_hash)
sort_length+= sizeof(ulonglong);

rec_length= sort_length + addon_length;
//modified filesort algorithm sort_length 为排序键长度 addon_lenth 为访问字段长度，original filesort algorithm rowid(主键) +排序字段长度 ，因为addon_length为0
```
好了我们稍微总结一下：

original filesort algorithm：每行长度为 sort 字段的总长度 + ref 字段长度（主键或者 rowid）。
modified filesort algorithm：每行的长度为 sort 字段的总长度 + addon 字段的长度（需要访问的字段总长度）。
当然到底使用那种算法参考上一部分。但是要注意了对于 varchar 这种可变长度是以定义的大小为准了，比如 UTF8 varchar(300) 就是 300 * 3 = 900 而不是实际存储的大小，而固定长度没有变化。好了，还是回头看看第二部分的两个例子，分别计算它们的行长度：

例子 1：根据我们的计算，它将使用 original filesort algorithm 排序方式，最终的计算行长度应该为（sort 字段长度 + rowid 长度）及 ≈ 1202 + 6 字节，下面是 debug 的结果：
```
(gdb) p rec_length
$1 = 1208
```
例子 2：根据我们的计算，它将使用 modified filesort algorithm 排序方式，最终计算行长度应该为（sort 字段长度 + addon 字段长度）及 ≈ 82 + 20 3 3 （结果为 262），注意这里是约等于没有计算非空等因素和可变长度因素，下面是 debug 的结果：
```
(gdb) p rec_length
$2 = 266
```
可以看出误差不大。

## 确认最大内存分配
这里的分配内存就和参数 sort_buffer_size 大小有关了。但是是不是每次都会分配至少 sort_buffer_size 大小的内存的呢？其实不是，MySQL 会判断是否表很小的情况，也就是做一个简单的运算，目的在于节省内存的开销，这里我们将来描述。

1、大概计算出 Innodb 层主键叶子结点的行数
这一步主要通过（聚集索引叶子结点的空间大小 / 聚集索引每行大小 * 2）计算出一个行的上限，调入函数 ha_innobase::estimate_rows_upper_bound，源码如下：
```
num_rows= table->file->estimate_rows_upper_bound();
//上限来自于Innodb 叶子聚集索引叶子结点/聚集索引长度 *2
```
然后将结果存储起来，如果表很小那么这个值会非常小。

2、根据前面计算的每行长度计算出 sort buffer 可以容下的最大行数
这一步将计算 sort buffer 可以容纳的最大行数如下：
```
ha_rows keys= memory_available / (param.rec_length + sizeof(char*));
//可以排序的 行数 sort buffer 中最大 可以排序的行数
```

3、对比两者的最小值，作为分配内存的标准
然后对比两个值以小值为准，如下：
```
param.max_keys_per_buffer= (uint) min(num_rows > 0? num_rows : 1, keys);
//存储行数上限 和 可以排序 行数的 小值
```

4、根据结果分配内存
分配如下：
```
table_sort.alloc_sort_buffer(param.max_keys_per_buffer, param.rec_length);
```
也就是根据总的计算出的行长度和计算出的行数进行分配。

## 读取数据，进行内存排序
到这里准备工作已经完成了，接下就是以行为单位读取数据了，然后对过滤掉 where 条件的剩下的数据进行排序。如果需要排序的数据很多，那么等排序内存写满后会进行内存排序，然后将排序的内容写入到排序临时文件中，等待下一步做外部的归并排序。

作为归并排序而言，每一个归并的文件片段必须是排序好的，否则归并排序是不能完成的，因此写满排序内存后需要做内存排序。如果写不满呢，那么做一次内存排序就好了。

下面我们来看看这个过程，整个过程集中在 find_all_keys 函数中。

1、读取需要的数据
实际上在这一步之前还会做 read_set 的更改，因为对于 original filesort algorithm(回表排序) 的算法来讲不会读取全部需要的字段，为了简单起见不做描述了。这一步就是读取一行数据了，这里会进入 Innodb 层读取数据，具体流程不做解释了，下面是这一行代码：
```
error= file->ha_rnd_next(sort_form->record[0]); //读取一行数据

```

2、将 Rows_examined 加 1
这里这个指标对应的就是慢查中的 Rows_examined 了，这个指标在有排序的情况下会出现重复计算的情况，但是这里还是正确的，重复的部分后面再说。

3、过滤掉 where 条件
这里将会过滤掉 where 条件中不满足条件的行，代码如下：
```
if(!error && !qep_tab->skip_record(thd, &skip_record) && !skip_record)
//这里做where过滤条件 的比较
```

将行数据写入到 sort buffer 中
这一步将会把数据写入到 sort buffer 中，需要注意这里不涉及排序操作，只是存储数据到内存中。其中分为了 2 部分：

写入 sort 字段。如果是 original filesort algorithm 那么 rowid(主键) 也包含在其中了。
写入 addon 字段，这是 modified filesort algorithm 才会有的，在写入之前还会调用 Field::pack 对可以打包（pack）的字段进行压缩操作。对于 varchar 字段的打包函数就是 Field_varstring::pack，简单的说存储的是实际的大小，而非定义的大小。
整个过程位于 find_all_keys->Sort_param::make_sortkey 函数中。这一步还涉及到了我们非常关心的一个问题，到底排序的数据如何存储的问题，需要仔细阅读。下面我们就 debug 一下第二部分中两个例子的不同存储方式。

既然要去看内存中的数据，我们只要看它最终拷贝的内存数据是什么就好了，那么真相将会大白，我们只需要将断点放到 find_all_keys 函数上，做完一行数据的 Sort_param::make_sortkey 操作后看内存就行了，如下：

例子 1（字段都是 varchar(300)）：它将使用 original filesort algorithm(回表排序) 的方式，最终应该存储的是 sort 字段（a2,a3）+ rowid。排序的结果如下：
```
mysql> select* from test.tests1 where a1='b' order by a2,a3;
+------+------+------+
| a1 | a2 | a3 |
+------+------+------+
| b | d | d |
| b | e | e |
| b | f | f |
+------+------+------+
3 rows inset(9.06 sec)

我们以第二行为查看目标
```
这里对于 sort 字段而言，比实际存储的数据大得多。

例子 2（字段都是 varchar(20)）：它将使用 modified filesort algorithm，最终应该存储的是 sort 字段（a2,a3）+ addon 字段（需要的字段，这里就是 a1,a2,a3）

## 如果 sort buffer 存满，对 sort buffer 中的数据进行排序，然后写入到临时文件。
如果需要排序的数据量很大的话，那么 sort buffer 肯定是不能容下的，因此如果写满后就进行一次内存排序操作，然后将排序好的数据写入到外部排序文件中去，这叫做一个 chunk。外部文件的位置由 tmpdir 参数指定，名字以 MY 开头，注意外部排序通常需要 2 个临时文件，这里是第 1 个用于存储内存排序结果的临时文件，以 chunk 的方式写入。如下：
```
if(fs_info->isfull()) //如果sort buffer满了 并且sort buffer已经排序完成
{
if(write_keys(param, fs_info, idx, chunk_file, tempfile)) //写入到物理文件 完成内存排序 如果内存不会满这里不会做 会在create_sort_index 中排序完成
{
num_records= HA_POS_ERROR;
goto cleanup;
}
idx= 0;
indexpos++;
}
```
最终会调入 write_keys 函数进行排序和写入外部排序文件，这里核心就是先排序，然后循环每条排序文件写入到外部排序文件。下面我来验证一下写入临时文件的长度，我将第二部分中的例子 2 数据扩大了 N 倍后，让其使用外部文件排序，下面是验证结果，断点 write_keys 即可：
```
1161if(my_b_write(tempfile, record, rec_length))
(gdb) p rec_length
$8 = 91
```
可以每行的长度还是 91 字节（打包压缩后），和前面看到的长度一致，说明这些数据会完完整整的写入到外部排序文件，这显然会比我们想象的大得多。

好了到这里数据已经找出来了，如果超过 sort buffer 的大小，外部排序需要的结果已经存储在临时文件 1 了，并且它是分片（chunk）存储到临时文件的，它以 MY 开头。

## 排序方式总结输出
这里对上面的排序过程做了一个阶段性的总结，代码如下：
```
Opt_trace_object(trace, "filesort_summary")
.add("rows", num_rows)
.add("examined_rows", param.examined_rows)
.add("number_of_tmp_files", num_chunks)
.add("sort_buffer_size", table_sort.sort_buffer_size())
.add_alnum("sort_mode",
param.using_packed_addons() ?
"<sort_key, packed_additional_fields>":
param.using_addon_fields() ?
"<sort_key, additional_fields>": "<sort_key, rowid>");
```
我们解析一下：

rows：排序的行数，也就是应用 where 过滤条件后剩下的行数。
examined_rows：Innodb 层扫描的行数，注意这不是慢查询中的 Rows_examined，这里是准确的结果，没有重复计数。
number_of_tmp_files：外部排序时，用于保存结果的临时文件的 chunk 数量，每次 sort buffer 满排序后写入到一个 chunk，但是所有 chunk 共同存在于一个临时文件中。
sort_buffer_size：内部排序使用的内存大小，并不一定是 sort_buffer_size 参数指定的大小。
sort_mode：这里解释如下
1、sort_key, packed_additional_fields：使用了 modified filesort algorithm(不回表排序)，并且有打包（pack）的字段，通常为可变字段比如 varchar。

2、sort_key, additional_fields：使用了 modified filesort algorithm(不回表排序)，但是没有需要打包（pack）的字段，比如都是固定长度字段。

3、sort_key, rowid：使用了 original filesort algorithm(回表排序)。

## 进行最终排序
这里涉及 2 个部分如下：

如果 sort buffer 不满，则这里开始进行排序，调入函数 save_index。
如果 sort buffer 满了，则进行归并排序，调入函数 merge_many_buff->merge_buffers，最后调入 merge_index 完成归并排序。
对于归并排序来讲，这里可能会生成另外 2 个临时文件用于存储最终排序的结果，它们依然以 MY 开头，且依然是存储在 tmpdir 参数指定的位置。因此在外部排序中将可能会生成 3 个临时文件，总结如下：

临时文件 1：用于存储内存排序的结果，以 chunk 为单位，一个 chunk 的大小就是 sort buffer 的大小。
临时文件 2：以前面的临时文件 1 为基础，做归并排序。
临时文件 3：将最后的归并排序结果存储，去掉 sort 字段，只保留 addon 字段（需要访问的字段）或者 ref 字段（ROWID 或者主键），因此它一般会比前面 2 个临时文件小。
但是它们不会同时存在，要么 临时文件 1 和临时文件 2 存在，要么 临时文件 2 和临时文件 3 存在。

这个很容易验证，将断点放到 merge_buffers 和 merge_index 上就可以验证了，如下：
```
临时文件1和临时文件2同时存在：

[root@gp1 test]# lsof|grep tmp/MY

mysqld     8769     mysql   70u      REG              252,3    79167488    2249135/mysqldata/mysql3340/tmp/MYt1QIvr(deleted)

mysqld     8769     mysql   71u      REG              252,3    58327040    2249242/mysqldata/mysql3340/tmp/MY4CrO4m (deleted)



临时文件2和临时文件3共同存在：

[root@gp1 test]# lsof|grep tmp/MY

mysqld     8769     mysql   70u      REG              252,3      360448    2249135/mysqldata/mysql3340/tmp/MYg109Wp(deleted)

mysqld     8769     mysql   71u      REG              252,3    79167488    2249242/mysqldata/mysql3340/tmp/MY4CrO4m (deleted)
```
但是由于能力有限对于归并排序的具体过程我并没有仔细学习了，这里给一个大概的接口。

注意这里每次调用 merge_buffers 将会增加 Sort_merge_passes 1 次，应该是归并的次数，这个值增量的大小可以侧面反映出外部排序使用临时文件的大小。

## 排序的其他问题
这里将描述 2 个额外的排序问题。

1、original filesort algorithm(回表排序) 的回表
最后对于 original filesort algorithm(回表排序) 排序方式而言，可能还需要做一个回表获取数据的操作，这一步可能会用到参数 read_rnd_buffer_size 定义的内存大小。

比如我们第二部分中第 1 个例子将会使用到 original filesort algorithm(回表排序)，但是对于回表操作有如下标准：

如果没有使用到外部排序临时文件则说明排序量不大，则使用普通的回表方式，调入函数 rr_from_pointers，也就是单行回表方式。
如果使用到了外部排序临时文件则说明排序量较大，需要使用到批量回表方式，这个时候大概的步骤就是读取 rowid(主键) 排序，然后批量回表，这将会在 read_rnd_buffer_size 指定的内存中完成，调入函数 rr_from_cache。这也是一种优化方式，因为回表一般是散列的，代价很大。
2、关于排序中 Rows_examined 的计算
首先这个值我说的是慢查询的中的 Rows_examined，在排序中会出现重复计数的可能，前面第八部分已经说明了一下，这个值在第八部分还是正确的，但是最后符合 where 条件的数据在返回的时候还会调用函数 evaluate_join_record，结果 Rows_examined 会增加符合 where 条件的行数。还是以我们第二部分的两个例子为例：
```
mysql> select* from test.tests1 where a1='b' order by a2,a3;
+------+------+------+
| a1 | a2 | a3 |
+------+------+------+
| b | d | d |
| b | e | e |
| b | f | f |
+------+------+------+
3 rows inset(5.11 sec)

mysql> select* from test.tests2 where a1='b' order by a2,a3;
+------+------+------+
| a1 | a2 | a3 |
+------+------+------+
| b | d | d |
| b | e | e |
| b | f | f |
+------+------+------+
3 rows inset(5.28 sec)

mysql> desc select* from tests2 where a1='b' order by a2,a3;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| id | select_type | table | partitions | type | possible_keys | key | key_len | ref| rows | filtered | Extra|
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| 1| SIMPLE | tests2 | NULL | ALL | NULL | NULL | NULL | NULL | 8| 12.50| Usingwhere; Using filesort |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
1 row inset, 1 warning (0.00 sec)

8 rows inset(0.00 sec)

mysql> desc select* from tests2 where a1='b' order by a2,a3;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| id | select_type | table | partitions | type | possible_keys | key | key_len | ref| rows | filtered | Extra|
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| 1| SIMPLE | tests2 | NULL | ALL | NULL | NULL | NULL | NULL | 8| 12.50| Usingwhere; Using filesort |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
1 row inset, 1 warning (0.01 sec)
```

## 通过 OPTIMIZER_TRACE 查看排序结果
要使用 OPTIMIZER_TRACE 只需要“SET optimizer_trace="enabled=on";”，跑完语句后查看 information_schema.OPTIMIZER_TRACE 即可。前面第九部分我们解释了排序方式总结输出的含义，这里我们来看看具体的结果，我们还是以第二部分的 2 个例子为例：
```
"filesort_priority_queue_optimization": {
"usable": false,
"cause": "not applicable (no LIMIT)"
},
"filesort_execution": [
],
"filesort_summary": {
"rows": 3,
"examined_rows": 8,
"number_of_tmp_files": 0,
"sort_buffer_size": 1285312,
"sort_mode": "<sort_key, rowid>"
```

```
"filesort_priority_queue_optimization": {
"usable": false,
"cause": "not applicable (no LIMIT)"
},
"filesort_execution": [
],
"filesort_summary": {
"rows": 3,
"examined_rows": 8,
"number_of_tmp_files": 0,
"sort_buffer_size": 322920,
"sort_mode": "<sort_key, packed_additional_fields>"
```
现在我们清楚了，这些总结实际上是在执行阶段生成的，需要注意几点如下：

这里的 examined_rows 和慢查询中的 Rows_examined 不一样，因为这里不会有重复计数，是准确的。
这里还会说明是否使用了优先队列排序即“filesort_priority_queue_optimization”部分。
通过“sort_buffer_size”可以发现，这里并没有分配参数 sort_buffer_size 指定的大小，节约了内存，这在第七部分说明了。
其他指标在第九部分已经说明过了，不再描述。

## 全文总结
本文写了很多，这里需要做一个详细的总结：

总结 1：排序中一行记录如何组织？
一行排序记录，由 sort 字段 + addon 字段组成，其中 sort 字段为 order by 后面的字段，而 addon 字段为需要访问的字段，比如‘select a1,a2,a3 from test order by a2,a3’，其中 sort 字段为‘a2,a3’，addon 字段为‘a1,a2,a3’。sort 字段中的可变长度字段不能打包（pack）压缩，比如 varchar，使用的是定义的大小计算空间，注意这是排序使用空间较大的一个重要因素。
如果在计算 sort 字段空间的时候，某个字段的空间大小大于了 max_sort_length 大小则按照 max_sort_length 指定的大小计算。
一行排序记录，如果 sort 字段 + addon 字段的长度大于了 max_length_for_sort_data 的大小，那么 addon 字段将不会存储，而使用 sort 字段 + ref 字段代替，ref 字段为主键或者 ROWID，这个时候就会使用 original filesort algorithm(回表排序) 的方式了。
如果 addon 字段包含可变字段比如 varchar 字段，则会使用打包（pack）技术进行压缩，节省空间。可以参考第三、第四、第五、第六、第八部分。
总结 2：排序使用什么样的方法进行？
original filesort algorithm(回表排序)
如果使用的是 sort 字段 + ref 字段进行排序，那么必须要回表获取需要的数据，如果排序使用了临时文件（也就是说使用外部归并排序，排序量较大）则会使用批量回表，批量回表会涉及到 read_rnd_buffer_size 参数指定的内存大小，主要用于排序和结果返回。

如果排序没有使用临时文件（内存排序就可以完成，排序量较小）则采用单行回表。

modified filesort algorithm(不回表排序)
如果使用的是 sort 字段 + addon 字段进行排序，那么使用不回表排序，所有需要的字段均在排序过程中进行存储，addon 字段中的可变长度字段可以进行打包（pack）压缩节省空间。

其次 sort 字段和 addon 字段中可能有重复的字段，比如例 2 中，sort 字段为 a2、a3，addon 字段为 a1、a2、a3，这是排序使用空间较大的另外一个原因。

在 OPTIMIZER_TRACE 中可以查看到使用了那种方法，参考第十二部分。

总结 3：每次排序一定会分配 sort_buffer_size 参数指定的内存大小吗？
不是这样的，MySQL 会做一个初步的计算，通过比较 Innodb 中聚集索引可能存储的行上限和 sort_buffer_size 参数指定大小内存可以容纳的行上限，获取它们小值进行确认最终内存分配的大小，目的在于节省内存空间。

在 OPTIMIZER_TRACE 中可以看到使用的内存大小，参考第八、第十二部分。

总结 4：关于 OPTIMIZER_TRACE 中的 examined_rows 和慢查询中的 Rows_examined 有什么区别？
慢查询中的 Rows_examined 包含了重复计数，重复的部分为 where 条件过滤后做了排序的部分。
OPTIMIZER_TRACE 中的 examined_rows 不包含重复计数，为实际 Innodb 层扫描的行数。
可以参考第十一部分。

总结 5：外部排序临时文件的使用是什么样的？
实际上一个语句的临时文件不止一个，但是它们都以 MY 开头，并且都放到了 tmpdir 目录下，lsof 可以看到这种文件。

临时文件 1：用于存储内存排序的结果，以 chunk 为单位，一个 chunk 的大小就是 sort buffe r的大小。
临时文件 2：以前面的临时文件 1 为基础，做归并排序。
临时文件 3：将最后的归并排序结果存储，去掉 sort 字段，只保留 addon 字段（需要访问的字段）或者 ref 字段（ROWID 或者主键），因此它一般会比前面 2 个临时文件小。
但是它们不会同时存在，要么临时文件1和临时文件 2 存在，要么临时文件 2 和临时文件 3 存在。对于临时文件的使用可以查看 Sort_merge_passes，本值多少会侧面反应出外部排序量的大小。

可以参考第十部分。

总结 6：排序使用了哪种算法？
虽然本文不涉及算法，但是内部排序有 2 种算法需要知道：

内存排序（优先队列 order by limit 返回少量行常用，提高排序效率，但是注意 order by limit n,m 如果 n 过大可能会涉及到排序算法的切换）
内存排序（快速排序）
在通过 OPTIMIZER_TRACE 可以查看是否使用使用了优先队列算法，参考第十二部分。

总结 7：“Creating sort index”到底是什么状态？
我们前面讲的全部排序流程都会包含在这个状态下，包括：

获取排序需要的数据（比如例子中全表扫描从 Innodb 层获取数据）
根据 where 条件过滤数据
内存排序
外部排序
总结 8：如何避免临时文件过大的情况？
首先应该考虑是否可以使用索引来避免排序，如果不能则需要考虑下面的要点：

order by 后面的字段满足需求即可，尽可能的少。
order by 后面涉及的字段尽量为固定长度的字段类型，而不是可变字段类型如 varchar。因为 sort 字段不能压缩。
不要过大的定义可变字段长度，应该合理定义，例如 varchar(10) 能够满足需求不要使用 varchar(50)，这些空间虽然在 Innodb 层存储会压缩，但是 MySQL 层确可能使用全长度（比如 sort 字段）。
在查询中尽量不要用（select ） 而使用需要查询的字段，这将会减少 addon 字段的个数，在我另外一个文章还讲述了（select ）的其他的缺点