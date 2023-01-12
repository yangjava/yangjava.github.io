---
layout: post
categories: ElasticSearch
description: none
keywords: ElasticSearch
---
# ElasticSearch简介
Elasticsearch 是一个分布式的开源搜索和分析引擎，在 Apache Lucene 的基础上开发而成。

## Elasticsearch
Lucene 是开源的搜索引擎工具包，Elasticsearch 充分利用Lucene，并对其进行了扩展，使存储、索引、搜索都变得更快、更容易， 而最重要的是， 正如名字中的“ elastic ”所示， 一切都是灵活、有弹性的。而且，应用代码也不是必须用Java 书写才可以和Elasticsearc兼容，完全可以通过JSON 格式的HTTP 请求来进行索引、搜索和管理Elasticsearch 集群。
Elasticsearch是一个基于Apache Lucene(TM)的开源搜索引擎，无论在开源还是专有领域，Lucene可以被认为是迄今为止最先进、性能最好的、功能最全的搜索引擎库。
但是，Lucene只是一个库。想要发挥其强大的作用，你需使用Java并要将其集成到你的应用中。Lucene非常复杂，你需要深入的了解检索相关知识来理解它是如何工作的。
Elasticsearch也是使用Java编写并使用Lucene来建立索引并实现搜索功能，但是它的目的是通过简单连贯的RESTful API让全文搜索变得简单并隐藏Lucene的复杂性。
不过，Elasticsearch不仅仅是Lucene和全文搜索引擎，它还提供：

- 分布式的实时文件存储，每个字段都被索引并可被搜索
- 实时分析的分布式搜索引擎
- 可以扩展到上百台服务器，处理PB级结构化或非结构化数据
而且，所有的这些功能被集成到一台服务器，你的应用可以通过简单的RESTful API、各种语言的客户端甚至命令行与之交互。上手Elasticsearch非常简单，它提供了许多合理的缺省值，并对初学者隐藏了复杂的搜索引擎理论。它开箱即用（安装即可使用），只需很少的学习既可在生产环境中使用。Elasticsearch在Apache 2 license下许可使用，可以免费下载、使用和修改。
随着知识的积累，你可以根据不同的问题领域定制Elasticsearch的高级特性，这一切都是可配置的，并且配置非常灵活。