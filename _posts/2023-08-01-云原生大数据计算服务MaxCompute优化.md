---
layout: post
categories: [Action]
description: none
keywords: Action
---
# 云原生大数据计算服务 MaxCompute优化

## 分组取出每组数据的前N条
取出每条数据的行号，再用 where 语句进行过滤。
```

SELECT * FROM (
  SELECT empno
  , ename
  , sal
  , job
  , ROW_NUMBER() OVER (PARTITION BY job ORDER BY sal) AS rn
  FROM emp
) tmp
WHERE rn < 10;
```

## 多行数据合并为一行数据
将class相同的name合并为一行，并对name去重。去重操作可通过嵌套子查询实现。
```
SELECT class, wm_concat(distinct ',', name) as names FROM students GROUP BY class;
```
统计不同class对应的男女人数。
```
SELECT 
class
,SUM(CASE WHEN gender = 'M' THEN 1 ELSE 0 END) AS cnt_m
,SUM(CASE WHEN gender = 'F' THEN 1 ELSE 0 END) AS cnt_f
FROM students
GROUP BY class;
```

## 行转列示例
使用case when表达式，灵活提取各科目（subject）的值作为单独的列，命令示例如下。
```
SELECT name AS 姓名,
       max(case subject when '语文' then result end) AS 语文,
       max(case subject when '数学' then result end) AS 数学,
       max(case subject when '物理' then result end) AS 物理 
FROM rowtocolumn 
GROUP BY name;
```

## 列转行示例
使用union all，将各科目（chinese、mathematics、physics）整合为一列，命令示例如下。
```
--解除order by必须带limit的限制，方便列转行SQL命令对结果按照姓名排序。
SET odps.sql.validate.orderby.limit=false;
--列转行SQL。
SELECT name AS 姓名, subject AS 科目, result AS 成绩 
FROM(
     SELECT name, '语文' AS subject, chinese AS result FROM columntorow 
     UNION all 
     SELECT name, '数学' AS subject, mathematics AS result FROM columntorow 
     UNION all 
     SELECT name, '物理' AS subject, physics AS result FROM columntorow) 
ORDER BY name;
```
借助MaxCompute提供的内建函数实现，先基于CONCAT函数拼接各科目和成绩，然后基于TRANS_ARRAY和SPLIT_PART函数逐层拆解科目和成绩作为单独的列。
```
SELECT name AS 姓名,
       split_part(subject,':',1) AS 科目,
       split_part(subject,':',2) AS 成绩
FROM(
       SELECT trans_array(1,';',name,subject) AS (name,subject) 
       FROM(
            SELECT name,
        concat('语文',':',chinese,';','数学',':',mathematics,';','物理',':',physics) AS subject 
            FROM columntorow)tt)tx;
```

## 关联操作
- INNER JOIN 输出符合关联条件的数据。
- LEFT JOIN 输出左表的所有记录，以及右表中符合关联条件的数据。右表中不符合关联条件的行，输出NULL。
- RIGHT JOIN 输出右表的所有记录，以及左表中符合关联条件的数据。左表中不符合关联条件的行，输出NULL。
- FULL JOIN 输出左表和右表的所有记录，对于不符合关联条件的数据，未关联的另一侧输出NULL。
- LEFT SEMI JOIN 对于左表中的一条数据，如果右表存在符合关联条件的行，则输出左表。
- LEFT ANTI JOIN 对于左表中的一条数据，如果右表中不存在符合关联条件的数据，则输出左表。

SQL语句中，同时存在JOIN和WHERE子句时，如下所示。
```
(SELECT * FROM A WHERE {subquery_where_condition} A) A
JOIN
(SELECT * FROM B WHERE {subquery_where_condition} B) B
ON {on_condition}
WHERE {where_condition}
```
计算顺序如下：
- 子查询中的WHERE子句（即{subquery_where_condition}）。
- JOIN子句中的关联条件（即{on_condition}）。
- JOIN结果集中的WHERE子句（即{where_condition}）。
因此，对于不同的JOIN类型，过滤条件在{subquery_where_condition}、{on_condition}和{where_condition}中时，查询结果可能一致，也可能不一致。

## 窗口函数优化
如果SQL语句中使用了窗口函数，通常每个窗口函数会形成一个Reduce作业。如果窗口函数较多，会消耗过多的资源。您可以对符合下述条件的窗口函数进行优化：
窗口函数在OVER关键字后面要完全相同，要有相同的分组和排序条件。
多个窗口函数在同一层SQL中执行。
符合上述2个条件的窗口函数会合并为一个Reduce执行。SQL示例如下所示。
```
SELECT
RANK()OVER(PARTITION BY A ORDER BY B desc) AS RANK,
ROW_NUMBER()OVER(PARTITION BY A ORDER BY B desc) AS row_num
FROM MyTable;
```

## 子查询优化
```
SELECT * FROM table_a a WHERE a.col1 IN (SELECT col1 FROM table_b b WHERE xxx);
```
当此语句中的table_b子查询返回的col1的个数超过9999个时，系统会报错为records returned from subquery exceeded limit of 9999。此时您可以使用Join语句来代替，如下所示。
```
SELECT a.* FROM table_a a JOIN (SELECT DISTINCT col1 FROM table_b b WHERE xxx) c ON (a.col1 = c.col1)
```

## Join语句优化
当两个表进行Join操作时，建议在如下位置使用WHERE子句：
主表的分区限制条件可以写在WHERE子句中（建议先用子查询过滤）。
主表的WHERE子句建议写在SQL语句最后。
从表分区限制条件不要写在WHERE子句中，建议写在ON条件或者子查询中。
示例如下。
```
SELECT * FROM A JOIN (SELECT * FROM B WHERE dt=20150301)B ON B.id=A.id WHERE A.dt=20150301；
SELECT * FROM A JOIN B ON B.id=A.id WHERE B.dt=20150301；--不建议使用。此语句会先执行Join操作后进行分区裁剪，导致数据量变大，性能下降。
SELECT * FROM (SELECT * FROM A WHERE dt=20150301)A JOIN (SELECT * FROM B WHERE dt=20150301)B ON B.id=A.id；
```

## 聚合函数优化
使用wm_concat函数替代collect_list函数，实现聚合函数的优化，使用示例如下。
```
-- collect_list实现
select concat_ws(',', sort_array(collect_list(key))) from src;
-- wm_concat实现更优
select wm_concat(',', key) WITHIN group (order by key) from src;


-- collect_list实现
select array_join(collect_list(key), ',') from src;
-- wm_concat实现更优
select wm_concat(',', key) from src;
```

## 数据倾斜调优

### MapReduce
在了解数据倾斜之前首先需要了解什么是MapReduce，MapReduce是一种典型的分布式计算框架，它采用分治法的思想，将一些规模较大或者难以直接求解的问题分割成较小规模或容易处理的若干子问题，对这些子问题进行求解后将结果合并成最终结果。MapReduce相较于传统并行编程框架，具有高容错性、易使用性以及较好的扩展性等优点。在MapReduce实现并行程序无需考虑分布式集群中的编程无关问题，如数据存储、节点间的信息交流和传输机制等，大大简化了其用户的分布式编程方式。

### 数据倾斜
数据倾斜多发生在Reducer端，Mapper按Input files切分，一般相对均匀，数据倾斜指表中数据分布不均衡的情况分配给不同的Worker。数据不均匀的时候，导致有的Worker很快就计算完成了，但是有的Worker却需要运行很长时间。在实际生产中，大部分数据存在偏斜，这符合“二八”定律，例如一个论坛20%的活跃用户贡献了80%的帖子，或者一个网站80%的访问量由20%的用户提供。在数据量爆炸式增长的大数据时代，数据倾斜问题会严重影响分布式程序的执行效率。作业运行表现为作业的执行进度一直停留在99%，作业执行感觉被卡住了。

### 数据倾斜排查及解决方法
根据使用经验总结，引起数据倾斜的主要原因有如下几类：

Join

GroupBy

Count（Distinct）

ROW_NUMBER（TopN）

动态分区

其中出现的频率排序为JOIN > GroupBy > Count(Distinct) > ROW_NUMBER > 动态分区。

针对Join端产生的数据倾斜，会存在多种不同的情况，例如大表和小表Join、大表和中表Join、Join热值长尾。

大表Join小表。 数据倾斜示例。 如下示例中t1是一张大表，t2、t3是小表。
```
SELECT  t1.ip
        ,t1.is_anon
        ,t1.user_id
        ,t1.user_agent
        ,t1.referer
        ,t2.ssl_ciphers
        ,t3.shop_province_name
        ,t3.shop_city_name
FROM    <viewtable> t1
LEFT OUTER JOIN <other_viewtable> t2
ON t1.header_eagleeye_traceid = t2.eagleeye_traceid
LEFT OUTER JOIN (  SELECT  shop_id
                            ,city_name AS shop_city_name
                            ,province_name AS shop_province_name
                    FROM    <tenanttable>
                    WHERE   ds = MAX_PT('<tenanttable>')
                    AND     is_valid = 1
                ) t3
ON t1.shopid = t3.shop_id
```
解决方案。 使用MAPJOIN HINT语法，如下所示。
```
SELECT  /*+ mapjoin(t2,t3)*/
        t1.ip
        ,t1.is_anon
        ,t1.user_id
        ,t1.user_agent
        ,t1.referer
        ,t2.ssl_ciphers
        ,t3.shop_province_name
        ,t3.shop_city_name
FROM    <viewtable> t1
LEFT OUTER JOIN (<other_viewtable>) t2
ON t1.header_eagleeye_traceid = t2.eagleeye_traceid
LEFT OUTER JOIN (  SELECT  shop_id
                            ,city_name AS shop_city_name
                            ,province_name AS shop_province_name
                    FROM    <tenanttable>
                    WHERE   ds = MAX_PT('<tenanttable>')
                    AND     is_valid = 1
                ) t3
ON t1.shopid = t3.shop_id
```
注意事项。

引用小表或子查询时，需要引用别名。

MapJoin支持小表为子查询。

在MapJoin中可以使用不等值连接或or连接多个条件。您可以通过不写on语句而通过mapjoin on 1 = 1的形式，实现笛卡尔乘积的计算。例如select /*+ mapjoin(a) */ a.id from shop a join table_name b on 1=1;，但此操作可能带来数据量膨胀问题。

MapJoin中多个小表用半角逗号（,）分隔，例如/*+ mapjoin(a,b,c)*/。

MapJoin在Map阶段会将指定表的数据全部加载在内存中，因此指定的表仅能为小表且表被加载到内存后占用的总内存不得超过512 MB。由于MaxCompute是压缩存储，因此小表在被加载到内存后，数据大小会急剧膨胀。此处的512 MB是指加载到内存后的空间大小。

### Join热值长尾。
数据倾斜示例

在下面这个表中，eleme_uid中存在很多热点数据，容易发生数据倾斜。
```
SELECT
eleme_uid,
...
FROM (
    SELECT
    eleme_uid,
    ...
    FROM <viewtable>
)t1
LEFT JOIN(
    SELECT
    eleme_uid,
    ...
    FROM <customertable>
)  t2
on t1.eleme_uid = t2.eleme_uid;
```
- 方案一 手动切分热值 
将热点值分析出来后，从主表中过滤出热点值记录，先进行MapJoin，再将剩余非热点值记录进行MergeJoin，最后合并两部分的Join结果。
- 方案二 设置SkewJoin参数
set odps.sql.skewjoin=true;。
- 方案三
SkewJoin Hint
使用Hint提示：/*+ skewJoin(<table_name>[(<column1_name>[,<column2_name>,...])][((<value11>,<value12>)[,(<value21>,<value22>)...])]*/。SkewJoin Hint的方式相当于多了一次找倾斜Key的操作，会让Query运行时间加长；如果用户已经知道倾斜Key了，就可以通过设置SkewJoin参数的方式，能节省一些时间。
- 方案四
倍数表取模相等Join 利用倍数表。

倍数表取模相等Join。

该方案和前三个方案逻辑不同，不是分而治之的思路，而是利用一个倍数表，其值只有一列： int列，比如可以是从1到N（具体可根据倾斜程度确定），利用这个倍数表可以将用户行为表放大N倍，然后Join时使用用户ID和number两个关联键。这样原先只按照用户ID分发导致的数据倾斜就会由于加入了number关联条件而减少为原先的1/N。但是这样做也会导致数据膨胀N倍。
```
SELECT
eleme_uid,
...
FROM (
    SELECT
    eleme_uid,
    ...
    FROM <viewtable>
)t1
LEFT JOIN(
    SELECT
    /*+mapjoin(<multipletable>)*/
    eleme_uid,
    number
    ...
    FROM <customertable>
    JOIN <multipletable>
)  t2
on t1.eleme_uid = t2.eleme_uid
and mod(t1.<value_col>,10)+1 = t2.number;
```



