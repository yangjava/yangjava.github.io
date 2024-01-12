---
layout: post
categories: [Action]
description: none
keywords: Action
---
# 云原生大数据计算服务 MaxCompute常用函数

## 日期与时间函数
- ADD_MONTHS 计算日期值增加指定月数后的日期。
- CURRENT_TIMESTAMP 返回当前TIMESTAMP类型的时间戳。
- CURRENT_TIMEZONE 返回当前系统的时区值。
- DATE_ADD 按照指定的幅度增减天数，与date_sub的增减逻辑相反。
- DATEADD 按照指定的单位和幅度修改日期值。
- DATE_FORMAT 将日期值转换为指定格式的字符串。
- DATE_SUB 按照指定的幅度增减天数，与date_add的增减逻辑相反。
- DATEDIFF 计算两个日期的差值并按照指定的单位表示。
- DATEPART 提取日期中符合指定时间单位的字段值。
- DATETRUNC 提取日期按照指定时间单位截取后的值。
- DAY 返回日期值的天。
- DAYOFMONTH 返回日部分的值。
- DAYOFWEEK 返回日期的星期值。
- DAYOFYEAR 返回日期是当年中第几天。
- EXTRACT 获取日期TIMESTAMP中指定单位的部分。
- FROM_UNIXTIME 将数字型的UNIX值转换为日期值。
- FROM_UTC_TIMESTAMP 将一个UTC时区的时间戳转换为一个指定时区的时间戳。
- GETDATE 获取当前系统时间。
- HOUR 返回日期小时部分的值。
- ISDATE 判断一个日期字符串能否根据指定的格式串转换为一个日期值。
- LAST_DAY 返回日期值所在月份的最后一天日期。
- LASTDAY 获取日期所在月的最后一天。
- MINUTE 返回日期分钟部分的值。
- MONTH 返回日期值所属月份。
- MONTHS_BETWEEN 返回指定日期值间的月数。
- NEXT_DAY 返回大于日期值且与指定周相匹配的第一个日期。
- NOW 返回当前系统日期与时间。
- QUARTER 返回日期值所属季度。
- SECOND 返回日期秒数部分的值。
- TO_CHAR 将日期按照指定格式转换为字符串。
- TO_DATE 将指定格式的字符串转换为日期值。
- TO_MILLIS 将指定日期转换为以毫秒为单位的UNIX时间戳。
- UNIX_TIMESTAMP 将日期转换为整型的UNIX格式的日期值。
- WEEKDAY 返回日期值是当前周的第几天。
- WEEKOFYEAR 返回日期值位于当年的第几周。
- YEAR 返回日期值的年。

## 数学函数
- ABS 计算绝对值。

## 窗口函数
- AVG 对窗口中的数据求平均值。
- COUNT 计算窗口中的记录数。
- ROW_NUMBER 计算行号。从1开始递增。
- SUM 对窗口中的数据求和。

## 聚合函数
聚合（Aggregate）函数的输入与输出是多对一的关系，即将多条输入记录聚合成一条输出值，可以与MaxCompute SQL中的group by语句配合使用。

## 日期数据格式转换
使用CAST函数，将STRING类型数据2009-07-01 16:09:00转换为TIMESTAMP类型。命令示例如下。

输入的STRING类型数据的格式至少要满足yyyy-mm-dd hh:mi:ss要求。
```
--返回2009-07-01 16:09:00.000。
select cast('2009-07-01 16:09:00' as timestamp);
```
使用CAST函数，将STRING类型数据2009-07-01 16:09:00转换为DATETIME类型。
```
--返回2009-07-01 16:09:00。
select cast('2009-07-01 16:09:00' as datetime);
```
使用TO_DATE函数，指定format参数，将STRING类型数据2009-07-01 16:09:00转换为DATETIME类型。命令示例如下。
```
--返回2009-07-01 16:09:00。
select to_date('2009-07-01 16:09:00','yyyy-mm-dd hh:mi:ss');
```

使用CAST函数，将TIMESTAMP类型数据2009-07-01 16:09:00转换为STRING类型。为构造TIMESTAMP类型数据，总共需要使用2次CAST函数。
```
--返回2009-07-01 16:09:00。
select cast(cast('2009-07-01 16:09:00' as timestamp) as string);
```
使用TO_CHAR函数，将TIMESTAMP类型数据2009-07-01 16:09:00转换为STRING类型。为构造TIMESTAMP类型数据，需要使用到1次CAST函数。命令示例如下。
```
--返回2009-07-01 16:09:00。
select to_char(cast('2009-07-01 16:09:00' as timestamp),'yyyy-mm-dd hh:mi:ss');
```
使用CAST函数，将TIMESTAMP类型数据2009-07-01 16:09:00转换为DATETIME类型。为构造TIMESTAMP类型数据，总共需要使用2次CAST函数。命令示例如下。
```
--返回2009-07-01 16:09:00。
select cast(cast('2009-07-01 16:09:00' as timestamp) as datetime);
```
使用TO_DATE函数，指定format参数，将TIMESTAMP类型数据2009-07-01 16:09:00转换为DATETIME类型。为构造TIMESTAMP类型数据，需要使用到1次CAST函数。命令示例如下。
```
--返回2009-07-01 16:09:00。
select to_date(cast('2009-07-01 16:09:00' as timestamp),'yyyy-mm-dd hh:mi:ss');
```
使用CAST函数，将DATETIME类型的日期值转换为TIMESTAMP类型。为构造DATETIME类型数据，需要使用到1次GETDATE函数。命令示例如下。
```
--返回2021-10-14 10:21:47.939。
select cast(getdate() as timestamp);
```
使用TO_CHAR函数，将DATETIME类型的日期值转换为指定格式的STRING类型。为构造DATETIME类型数据，需要使用到1次GETDATE函数。
```
--返回2021-10-14 10:21:47。
select to_char (getdate(),'yyyy-mm-dd hh:mi:ss');
--返回2021-10-14。
select to_char (getdate(),'yyyy-mm-dd');
--返回2021。
select to_char (getdate(),'yyyy');
```