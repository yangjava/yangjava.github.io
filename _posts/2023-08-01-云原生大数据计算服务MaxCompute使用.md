---
layout: post
categories: [Action]
description: none
keywords: Action
---
# 云原生大数据计算服务 MaxCompute使用

## 创建表
命令格式
```
--创建新表。
 create [external] table [if not exists] <table_name>
 [primary key (<pk_col_name>, <pk_col_name2>),(<col_name> <data_type> [not null] [default <default_value>] [comment <col_comment>], ...)]
 [comment <table_comment>]
 [partitioned by (<col_name> <data_type> [comment <col_comment>], ...)]
 
--用于创建聚簇表时设置表的Shuffle和Sort属性。
 [clustered by | range clustered by (<col_name> [, <col_name>, ...]) [sorted by (<col_name> [asc | desc] [, <col_name> [asc | desc] ...])] into <number_of_buckets> buckets] 

--仅限外部表。
 [stored by StorageHandler] 
 --仅限外部表。
 [with serdeproperties (options)] 
 --仅限外部表。
 [location <osslocation>] 

--指定表为Transactional1.0表，后续可以对该表执行更新或删除表数据操作，但是Transactional表有部分使用限制，请根据需求创建。
 [tblproperties("transactional"="true")]

--指定表为Transactional2.0表，后续可以做upsert，增量查询，time-travel等操作
 [tblproperties ("transactional"="true" [, "write.bucket.num" = "N", "acid.data.retain.hours"="hours"...])] [lifecycle <days>]
;

--基于已存在的表创建新表并复制数据，但不复制分区属性。支持外部表和湖仓一体外部项目中的表。
create table [if not exists] <table_name> [lifecycle <days>] as <select_statement>;

--基于已存在的表创建具备相同结构的新表但不复制数据，支持外部表和湖仓一体外部项目中的表。
create table [if not exists] <table_name> like <existing_table_name> [lifecycle <days>];
```
创建非分区表bank_data和分区表bank_data_pt，用于存储业务数据；创建非分区表result_table1和result_table2，用于存储结果数据。

创建非分区表bank_data，命令示例如下。
```
create table if not exists bank_data
(
 age             BIGINT comment '年龄',
 job             STRING comment '工作类型',
 marital         STRING comment '婚否',
 education       STRING comment '教育程度',
 credit          STRING comment '是否有信用卡',
 housing         STRING comment '是否有房贷',
 loan            STRING comment '是否有贷款',
 contact         STRING comment '联系方式',
 month           STRING comment '月份',
 day_of_week     STRING comment '星期几',
 duration        STRING comment '持续时间',
 campaign        BIGINT comment '本次活动联系的次数',
 pdays           DOUBLE comment '与上一次联系的时间间隔',
 previous        DOUBLE comment '之前与客户联系的次数',
 poutcome        STRING comment '之前市场活动的结果',
 emp_var_rate    DOUBLE comment '就业变化速率',
 cons_price_idx  DOUBLE comment '消费者物价指数',
 cons_conf_idx   DOUBLE comment '消费者信心指数',
 euribor3m       DOUBLE comment '欧元存款利率',
 nr_employed     DOUBLE comment '职工人数',
 fixed_deposit   BIGINT comment '是否有定期存款'
);
```
创建分区表bank_data_pt，并添加分区，命令示例如下。
```
create table if not exists bank_data_pt
(
 age             BIGINT comment '年龄',
 job             STRING comment '工作类型',
 marital         STRING comment '婚否',
 education       STRING comment '教育程度',
 housing         STRING comment '是否有房贷',
 loan            STRING comment '是否有贷款',
 contact         STRING comment '联系方式',
 month           STRING comment '月份',
 day_of_week     STRING comment '星期几',
 duration        STRING comment '持续时间',
 campaign        BIGINT comment '本次活动联系的次数',
 pdays           DOUBLE comment '与上一次联系的时间间隔',
 previous        DOUBLE comment '之前与客户联系的次数',
 poutcome        STRING comment '之前市场活动的结果',
 emp_var_rate    DOUBLE comment '就业变化速率',
 cons_price_idx  DOUBLE comment '消费者物价指数',
 cons_conf_idx   DOUBLE comment '消费者信心指数',
 euribor3m       DOUBLE comment '欧元存款利率',
 nr_employed     DOUBLE comment '职工人数',
 fixed_deposit   BIGINT comment '是否有定期存款'
)partitioned by (credit STRING comment '是否有信用卡');

alter table bank_data_pt add if not exists partition (credit='yes') partition (credit='no') partition (credit='unknown');
```

## 分区
创建分区。
```
--创建一个二级分区表，以日期为一级分区，地域为二级分区
CREATE TABLE src (shop_name string, customer_id bigint) PARTITIONED BY (pt string,region string);
```
使用分区列作为过滤条件查询数据。
```
--正确使用方式。MaxCompute在生成查询计划时只会将'20170601'分区下region为'hangzhou'二级分区的数据纳入输入中。
select * from src where pt='20170601'and region='hangzhou'; 
--错误的使用方式。在这样的使用方式下，MaxCompute并不能保障分区过滤机制的有效性。pt是STRING类型，当STRING类型与BIGINT（20170601）比较时，MaxCompute会将二者转换为DOUBLE类型，此时有可能会有精度损失。
select * from src where pt = 20170601; 
```

## 创建或更新视图
创建或更新视图
```
create [or replace] view [if not exists] <view_name>
    [(<col_name> [comment <col_comment>], ...)]
    [comment <view_comment>]
    as <select_statement>;
```
基于表sale_detail创建视图sale_detail_view。
```
create view if not exists sale_detail_view
(store_name, customer_id, price, sale_date, region)
comment 'a view for table sale_detail'
as select * from sale_detail;
```

## 插入或覆写数据
MaxCompute支持通过insert into或insert overwrite操作向目标表或静态分区中插入、更新数据。

在使用MaxCompute SQL处理数据时，insert into或insert overwrite操作可以将select查询的结果保存至目标表中。二者的区别是：

insert into：直接向表或静态分区中插入数据。您可以在insert语句中直接指定分区值，将数据插入指定的分区。如果您需要插入少量测试数据，可以配合VALUES使用。

insert overwrite：先清空表或静态分区中的原有数据，再向表或静态分区中插入数据。
```
insert {into|overwrite} table <table_name> [partition (<pt_spec>)] [(<col_name> [,<col_name> ...)]]
<select_statement>
from <from_statement>
[zorder by <zcol_name> [, <zcol_name> ...]];
```

## 插入或覆写动态分区数据（DYNAMIC PARTITION）
```
insert {into|overwrite} table <table_name> partition (<ptcol_name>[, <ptcol_name> ...]) 
<select_statement> from <from_statement>;
```

## SELECT语法
```
[with <cte>[, ...] ]
SELECT [all | distinct] <SELECT_expr>[, <except_expr>][, <replace_expr>] ...
       from <table_reference>
       [where <where_condition>]
       [group by {<col_list>|rollup(<col_list>)}]
           [having <having_condition>]
       [order by <order_condition>]
       [distribute by <distribute_condition> [sort by <sort_condition>]|[ cluster by <cluster_condition>] ]
       [limit <number>]
       [window <window_clause>]
```
在选取的列名前可以使用distinct去掉重复字段，只返回去重后的值。使用all会返回字段中所有重复的值。

在列表达式（SELECT_expr）中，如果被重命名的列字段（赋予了列别名）使用了函数，则不能在where子句中引用列别名。错误命令示例如下。
```
SELECT  task_name
        ,inst_id
        ,settings
        ,GET_JSON_OBJECT(settings, '$.SKYNET_ID') as skynet_id
        ,GET_JSON_OBJECT(settings, '$.SKYNET_NODENAME') as user_agent
from    Information_Schema.TASKS_HISTORY
where   ds = '20211215' and skynet_id is not null
limit 10;
```

## WITH子句（cte）
WITH子句包含一个或多个常用的表达式CTE。CTE充当当前运行环境中的临时表，您可以在之后的查询中引用该表。

## GROUP BY分组查询（col_list）
SELECT的所有列中没有使用聚合函数的列，必须出现在GROUP BY中，否则返回报错。错误命令示例如下。
```
SELECT region, total_price from sale_detail group by region;
```

## HAVING子句（having_condition）
```
--为直观展示数据呈现效果，向sale_detail表中追加数据。
insert into sale_detail partition (sale_date='2014', region='shanghai') values ('null','c5',null),('s6','c6',100.4),('s7','c7',100.5);
--使用having子句配合聚合函数实现过滤。
SELECT region,sum(total_price) from sale_detail 
group by region 
having sum(total_price)<305;
```

## ORDER BY全局排序（order_condition）

## SORT BY局部排序（sort_condition）

## 窗口子句（window_clause）

## WINDOW关键字
```
WINDOW <window_name> AS (<window_definition>)
    [, <window_name> AS (<window_definition>)]
    ...
```

```
--没有window关键字写法
SELECT deptno,
       ename,
       sal,
       row_number() OVER (partition by deptno order by sal desc) AS nums
FROM emp;

--有window关键字写法
SELECT deptno,
       ename,
       sal,
       row_number() OVER w1 AS nums
FROM emp
WINDOW w1 AS (partition by deptno order by sal desc);
```

## QUALIFY
QUALIFY语法过滤Window函数数据类似于HAVING语法处理经过聚合函数和GROUP BY后的数据。

通过使用QUALIFY语法可以将原有的语法简化。
```
改造前SQL命令及结果：

SELECT col1, col2
FROM
(
SELECT
t.a as col1,
sum(t.a) over (partition by t.b) as col2
FROM values (1, 2),(2,3) t(a, b)
)
WHERE col2 > 1;

改写后SQL命令及结果：
SELECT 
t.a as col1, 
sum(t.a) over (partition by t.b) as col2 
FROM values (1, 2),(2,3) t(a, b) 
QUALIFY col2 > 1;

也可以不使用别名，直接对Window函数进行过滤，改写如下。

SELECT t.a as col1,
sum(t.a) over (partition by t.b) as col2
FROM values (1, 2),(2,3) t(a, b)
QUALIFY sum(t.a) over (partition by t.b)  > 1;
```
QUALIFY和WHERE、HAVING的使用方法相同，只是执行顺序不同，所以QUALIFY语法允许您写一些复杂的条件，示例如下。
```
SELECT * 
FROM values (1, 2),(2,3) t(a, b) 
QUALIFY sum(t.a) over (partition by t.b)  IN (SELECT a FROM <table_name>);
```
QUALIFY执行于窗口函数生效后，如下较复杂的示例可以直观感受QUALIFY语法的执行顺序。
```
SELECT a, b, max(c) 
FROM values (1, 2, 3),(1, 2, 4),(1, 3, 5),(2, 3, 6),(2, 4, 7),(3, 4, 8) t(a, b, c) 
WHERE a < 3 
GROUP BY a, b 
HAVING max(c) > 5 
QUALIFY sum(b) over (partition by a) > 3;

--返回结果
+------+------+------+
| a    | b    | _c2  |
+------+------+------+
| 2    | 3    | 6    |
| 2    | 4    | 7    |
+------+------+------+
```

## PIVOT、UNPIVOT
通过PIVOT关键字可以基于聚合将一个或者多个指定值的行转换为列；通过UNPIVOT关键字可以将一个或者多个列转换为行。
```
select * from (select season, 
                      tran_amt 
                 from mf_cop_sales) 
                      pivot (sum(tran_amt) for season in ('Q1' as spring, 
                                              						'Q2' as summer, 
                                              						'Q3' as autumn, 
                                              						'Q4' as winter) 
                ); 
```