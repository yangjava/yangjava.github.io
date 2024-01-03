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
## Document   文档
要索引的数据记录、文档在lucene中的表示，是索引、搜索的基本单元。一个Document由多个字段Field构成。就像数据库的记录-字段。

IndexWriter按加入的顺序为Document指定一个递增的id（从0开始），称为文档id。反向索引中存储的是这个id，文档存储中正向索引也是这个id。 业务数据的主键id只是文档的一个字段。

## Field
字段：由字段名name、字段值value（fieldsData）、字段类型 type 三部分构成。

字段值可以是文本（String、Reader 或 预分析的 TokenStream）、二进制值（byte[]）或数值。

IndexableField   Field API

## Document—Field 数据举例

新闻：新闻id，新闻标题、新闻内容、作者、所属分类、发表时间

网页搜索的网页：标题、内容、链接地址

商品： id、名称、图片链接、类别、价格、库存、商家、品牌、月销量、详情…

问题1：我们收集数据创建document对象来为其创建索引，数据的所有属性是否都需要加入到document中？如数据库表中的数据记录的所有字段是否都需要放到document中？哪些字段应加入到document中？

看具体的业务，只有需要被搜索和展示的字段才需要被加入到document中

问题2：是不是所有加入的字段都需要进行索引？是不是所有加入的字段都要保存到索引库中？什么样的字段该被索引？什么样的字段该被存储？

看具体的业务，需要被搜索的字段才该被索引，需要被展示的字段该被存储

问题3：各种要被索引的字段该以什么样的方式进行索引，全都是分词进行索引，还是有不同区别？

看是模糊查询还是精确查询，模糊查询的话就需要被分词索引，精确查询的话就不需要被分词索引

## IndexableFieldType
字段类型：描述该如何索引存储该字段。

字段可选择性地保存在索引中，这样在搜索结果中，这些保存的字段值就可获得。

一个Document应该包含一个或多个存储字段来唯一标识一个文档。为什么？

为从原数据中拿完整数据去展示

## IndexOptions 索引选项说明：
NONE：Not indexed 不索引

DOCS: 反向索引中只存储了包含该词的 文档id，没有词频、位置

DOCS_AND_FREQS: 反向索引中会存储 文档id、词频

DOCS_AND_FREQS_AND_POSITIONS:反向索引中存储 文档id、词频、位置

DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS :反向索引中存储 文档id、词频、位置、偏移量

## IndexableFieldType 中的 docValuesType方法 就是让你来为需要排序、分组、聚合的字段指定如何为该字段创建文档->字段值的正向索引的。
DocValuesType 选项说明:

NONE 不开启docvalue

NUMERIC 单值、数值字段，用这个

BINARY 单值、字节数组字段用

SORTED 单值、字符字段用， 会预先对值字节进行排序、去重存储

SORTED_NUMERIC 单值、数值数组字段用，会预先对数值数组进行排序

SORTED_SET 多值字段用，会预先对值字节进行排序、去重存储

具体使用选择：

字符串+单值 会选择SORTED作为docvalue存储

字符串+多值 会选择SORTED_SET作为docvalue存储

数值或日期或枚举字段+单值 会选择NUMERIC作为docvalue存储

数值或日期或枚举字段+多值 会选择SORTED_SET作为docvalue存储

注意：需要排序、分组、聚合、分类查询（面查询）的字段才创建docValues