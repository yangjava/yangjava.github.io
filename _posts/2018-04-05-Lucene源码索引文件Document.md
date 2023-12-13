---
layout: post
categories: [Lucene]
description: none
keywords: Lucene
---
# Lucene源码索引文件Document

## 创建文档Document对象，并加入域(Field)
```
        Document doc = new Document();
        doc.add(new Field("title", text, TextField.TYPE_STORED));
```
Document对象主要包括以下部分：
- 一个ArrayList保存此文档所有的域
- 每一个域包括域名，域值，和一些标志位，和fnm，fdx，fdt中的描述相对应。

## Filed
字段，一个Document会由一个或多个Field组成，数据库中可以对应一列，当然一列可以有多个field，每个Field有其对应的属性。比如 - 全文对应的TextField - 文本对应的StringField - 数值类型int对应的IntPoint，long对应的longPoint等等。且可以根据属性定制Field。较早版本中数值类型都是一种Field，而使用BKD-Tree之后，不同类型数值 分别对应一种Field，如IntPoint，对应为点的概念。

往document添加field，field有很多选项，是否分词是否添加存储等，根据实际情况选择。

## Field.Store.*
Field类是文档索引期间很重要的类，控制着被索引的域值。Field.Store.* 域存储选项通过倒排序索引来控制文本是否可以搜索
```
Field.Store.YES//表示会把这个域中的内容完全存储到文件中，方便进行还原[对于主键，标题可以是这种方式存储] 
Field.Store.NO//表示把这个域的内容不存储到文件中，但是可以被索引，此时内容无法完全还原（doc.get()）[对于内容而言，没有必要进行存储，可以设置为No]
```