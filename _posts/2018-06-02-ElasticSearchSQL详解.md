---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearchSQL详解
Elasticsearch SQL 是一个 X-Pack 组件，允许用户使用类似 SQL 的语法在 ES 中进行查询。

## Elasticsearch SQL
Elasticsearch SQL 是一个 X-Pack 组件，允许用户使用类似 SQL 的语法在 ES 中进行查询。用户可以在 REST、JDBC、命令行中使用 SQL 在 ES 执行数据检索和数据聚合操作。ES SQL 有以下几个特点：

本地集成，SQL 模块是 ES 自己构建的，直接集成到发布的版本中。
不需要外部的组件，使用 SQL 模块不需要额外的依赖，如硬件、运行时库等。
轻量高效，SQL 模块不抽象 ES 和其搜索能力，而是暴露 SQL 接口，允许以相同的声明性、简洁的方式进行适当的全文搜索。

## Elasticsearch SQL 使用
在开始使用 SQL 模块提供的功能前，在 kibana 执行以下指令来创建数据：
```
PUT /library/_bulk?refresh
{"index":{"_id": "Leviathan Wakes"}}
{"name": "Leviathan Wakes", "author": "James S.A. Corey", "release_date": "2011-06-02", "page_count": 561}
{"index":{"_id": "Hyperion"}}
{"name": "Hyperion", "author": "Dan Simmons", "release_date": "1989-05-26", "page_count": 482}
{"index":{"_id": "Dune"}}
{"name": "Dune", "author": "Frank Herbert", "release_date": "1965-06-01", "page_count": 604}
```
导入数据完成后，可以执行下面的 SQL 进行数据搜索了：
```
POST /_sql?format=txt
{
  "query": "SELECT * FROM library WHERE release_date < '2000-01-01'"
}
```
如上实例，使用 _sql 指明使用 SQL模块，在 query 字段中指定要执行的 SQL 语句。使用 format 指定返回数据的格式。

除了直接执行 SQL 外，还可以对结果进行过滤，使用 filter 字段在参数中指定过滤条件，可以使用标准的 ES DSL 查询语句过滤 SQL 运行的结果，其实例如下：
```
POST /_sql?format=txt
{
  "query": "SELECT * FROM library ORDER BY page_count DESC",
  "filter": {
    "range": {
      "page_count": {
        "gte" : 500,
        "lte" : 600
      }
    }
  },
  "fetch_size": 5
}
```

另外可以使用 ‘?’ 占位符来传递参数，然后将参数和语句组装成完整的 SQL 语句：
```
POST /_sql?format=txt
{
        "query": "SELECT YEAR(release_date) AS year FROM library WHERE page_count > ? AND author = ? GROUP BY year HAVING COUNT(*) > ?",
        "params": [300, "Frank Herbert", 0]
}

```

## 传统 SQL 和 Elasticsearch SQL 概念映射关系
虽然 SQL 和 Elasticsearch 对于数据的组织方式(以及不同的语义)有不同的术语，但本质上它们的用途是相同的。下面是它们的映射关系表：

| SQL    | Elasticsearch | 说明                                                                                                                                                                         |
|--------|---------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| column | field         | 在 Elasticsearch 字段时，SQL 将这样的条目调用为 column。注意，在 Elasticsearch，一个字段可以包含同一类型的多个值(本质上是一个列表) ，而在 SQL 中，一个列可以只包含一个表示类型的值。Elasticsearch SQL 将尽最大努力保留 SQL 语义，并根据查询的不同，拒绝那些返回多个值的字段。 |
| row	   | document      | 	列和字段本身不存在; 它们是行或文档的一部分。两者的语义略有不同: 行row往往是严格的（并且有更多的强制执行），而文档往往更灵活或更松散（同时仍然具有结构）。                                                                                          |
| table  | index         | 在 SQL 还是 Elasticsearch 中查询针对的目标                                                                                                                                            |
| schema | implicit      | 在关系型数据库中，schema 主要是表的名称空间，通常用作安全边界。Elasticsearch没有为它提供一个等价的概念。                                                                                                             |

虽然这些概念之间的映射在语义上有些不同，但它们间更多的是有共同点，而不是不同点。

## SQL Translate API
SQL Translate API 接收 JSON 格式的 SQL 语句，然后将其转换为 ES 的 DSL 查询语句，但是这个语句不会被执行，我们可以可以用这个 API 来将 SQL 翻译到 DSL 语句，其实例如下：
```
POST /_sql/translate
{
  "query": "SELECT * FROM library ORDER BY page_count DESC",
  "fetch_size": 10
}
```

如上实例，翻译出来的 DSL 如下：
```
{
  "size": 10,
  "_source": false,
  "fields": [
    { "field": "author" },
    { "field": "name" },
    { "field": "page_count" },
    {
      "field": "release_date",
      "format": "strict_date_optional_time_nanos"
    }
  ],
  "sort": [
    {
      "page_count": {
        "order": "desc",
        "missing": "_first",
        "unmapped_type": "short"
      }
    }
  ]
}
```



