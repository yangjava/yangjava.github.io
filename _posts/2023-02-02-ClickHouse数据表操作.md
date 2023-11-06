---
layout: post
categories: [ClickHouse]
description: none
keywords: ClickHouse
---
# ClickHouse数据表操作
目前只有MergeTree、Merge和Distributed这三类表引擎支持ALTER查询，如果现在还不明白这些表引擎的作用也不必担心，目前只需简单了解这些信息即可，后面会有专门章节对它们进行介绍。

## 追加新字段
假如需要对一张数据表追加新的字段，可以使用如下语法：
```
ALTER TABLE tb_name ADD COLUMN [IF NOT EXISTS] name [type] [default_expr] [AFTER name_after]
```
例如，在数据表的末尾增加新字段：
```
ALTER TABLE testcol_v1 ADD COLUMN OS String DEFAULT 'mac'
```
或是通过AFTER修饰符，在指定字段的后面增加新字段：
```
ALTER TABLE testcol_v1 ADD COLUMN IP String AFTER ID
```
对于数据表中已经存在的旧数据而言，新追加的字段会使用默认值补全。