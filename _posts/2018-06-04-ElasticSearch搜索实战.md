---
layout: post
categories: ElasticSearch
description: none
keywords: ElasticSearch
---
# ElasticSearch搜索实战

## 创建索引
在Elasticsearch中创建第一个文档时，没有关心索引的建立，只是使用了如下命令：
```
curl -XPUT http://localhost:9200/blog/article/1 -d '{"title": "New   version of Elasticsearch released!", "content": "...", "tags":  ["announce", "elasticsearch", "release"] }'
```
这是可以的。如果这样的索引不存在，Elasticsearch会为我们自动创建索引。还可以通过运行以下命令来创建索引：
```
curl -XPUT http://localhost:9200/blog/
```

## 修改索引的自动创建
通过在elasticsearch.yml配置文件中添加以下指令来关闭自动创建索引：
```
action.auto_create_index: false
```

## 新创建索引的设定
想设置一些配置选项时，也需要手动创建索引，例如设置分片和副本的数量。来看看下面的例子：
```
curl -XPUT http://localhost:9200/blog/ -d '{ 
    "settings" : { 
        "number_of_shards" : 1, 
        "number_of_replicas" : 2 
    } 
}'
```

## 映射配置
可以在索引的创建过程中执行以下命令：
```
curl -XPUT http://localhost:9200/blog/?pretty -d '{ 
  "mappings" : {
    "article": {
      "numeric_detection" : true
    }
  }
}'
```
Elasticsearch设法猜测被提供的时间戳或与日期格式匹配的字符串。可以使用dynamic_date_formats 属性定义可被识别的日期格式列表，该属性允许指定一个格式的数组。看一下创建索引和类型的命令：
```
curl -XPUT 'http://localhost:9200/blog/' -d '{ 
  "mappings" : { 
    "article" : { 
      "dynamic_date_formats" : ["yyyy-MM-dd hh:mm"] 
    } 
  } 
}'
```

## 索引结构映射
模式映射（schema mapping，或简称映射）用于定义索引结构。你可能还记得，每个索引可以有多种类型，但现在会为了简单起见我们只专注一个类型。

核心类型
每个字段类型可以指定为Elasticsearch提供的一个特定核心类型。Elasticsearch有以下核心类型。
- string ：字符串；
- number ：数字；
- date ：日期；
- boolean ：布尔型；
- binary ：二进制。

公共属性

在继续描述所有核心类型之前，先讨论一些可用来描述所有类型（二进制除外）的公共属性。
- index_name ：该属性定义将存储在索引中的字段名称。若未定义，字段将以对象的名字来命名。
- index ：可设置值为analyzed 和no 。另外，对基于字符串的字段，也可以设置为not_analyzed 。如果设置为analyzed ，该字段将被编入索引以供搜索。如果设置为no ，将无法搜索该字段。默认值为analyzed 。在基于字符串的字段中，还有一个额外的选项not_analyzed 。此设置意味着字段将不经分析而编入索引，使用原始值被编入索引，在搜索的过程中必须全部匹配。索引属性设置为no 将使include_in_all 属性失效。
- store ：这个属性的值可以是yes 或no ，指定了该字段的原始值是否被写入索引中。默认值设置为no ，这意味着在结果中不能返回该字段（然而，如果你使用_source 字段，即使没有存储也可返回这个值），但是如果该值编入索引，仍可以基于它来搜索数据。
- boost ：该属性的默认值是1 。基本上，它定义了在文档中该字段的重要性。boost 的值越高，字段中值的重要性也越高。
- null_value ：如果该字段并非索引文档的一部分，此属性指定应写入索引的值。默认的行为是忽略该字段。
- copy_to ：此属性指定一个字段，字段的所有值都将复制到该指定字段。
- include_in_all ：此属性指定该字段是否应包括在_all 字段中。默认情况下，如果使用_all 字段，所有字段都会包括在其中。

### 字符串
字符串是最基本的文本类型，我们能够用它存储一个或多个字符。字符串字段的示例定义如下所示：
```
"contents" : { "type" : "string", "store" : "no", "index" :"analyzed" }
```
字符串的字段还可以使用以下属性。
- term_vector ：此属性的值可以设置为no （默认值）、yes 、with_offsets 、with_positions 和with_positions_offsets 。它定义是否要计算该字段的Lucene词向量（term vector）。如果你使用高亮，那就需要计算这个词向量。
- omit_norms ：该属性可以设置为true 或false 。对于经过分析的字符串字段，默认值为false ，而对于未经分析但已编入索引的字符串字段，默认值设置为true 。当属性为true 时，它会禁用Lucene对该字段的加权基准计算（norms calculation），这样就无法使用索引期间的加权，从而可以为只用于过滤器中的字段节省内存（在计算所述文件的得分时不会被考虑在内）。
- analyzer ：该属性定义用于索引和搜索的分析器名称。它默认为全局定义的分析器名称。
- index_analyzer ：该属性定义了用于建立索引的分析器名称。
- search_analyzer ：该属性定义了的分析器，用于处理发送到特定字段的那部分查询字符串。
- norms.enabled ：此属性指定是否为字段加载加权基准（norms）。默认情况下，为已分析字段设置为true （这意味着字段可加载加权基准），而未经分析字段则设置为false 。
- norms.loading ：该属性可设置eager 和lazy 。第一个属性值表示此字段总是载入加权基准。第二个属性值是指只在需要时才载入。
- position_offset_gap ：此属性的默认值为0 ，它指定索引中在不同实例中具有相同名称的字段的差距。若想让基于位置的查询（如短语查询）只与一个字段实例相匹配，可将该属性值设为较高值。
- index_options ：该属性定义了信息列表（postings list）的索引选项。可能的值是docs （仅对文档编号建立索引），freqs （对文档编号和词频建立索引），positions （对文档编号、词频和它们的位置建立索引），offsets （对文档编号、词频、它们的位置和偏移量建立索引）。对于经分析的字段，此属性的默认值是positions ，对于未经分析的字段，默认值为docs 。
- ignore_above ：该属性定义字段中字符的最大值。当字段的长度高于指定值时，分析器会将其忽略。

### 数值
这一核心类型汇集了所有适用的数值字段类型。Elasticsearch中可使用以下类型（使用type属性指定）。
- byte ：定义字节值，例如1。
- short ：定义短整型值，例如12。
- integer ：定义整型值，例如134。
- long ：定义长整型值，例如123456789。
- float ：定义浮点值，例如12.23。
- double ：定义双精度值，例如123.45。
数值类型字段的定义如下所示：
```
"price" : { "type" : "float", "store" : "yes", "precision_step" : "4" }
```
除了公共属性，以下属性也适用于数值字段。
- precision_step ：此属性指定为某个字段中每个值生成的词条数。值越低，产生的词条数越高。对于每个值的词条数更高的字段，范围查询（range query）会更快，但索引会稍微大点，默认值为4。
- ignore_malformed ：此属性值可以设为true 或false 。默认值是false 。若要忽略格式错误的值，则应设置属性值为true 。

### 布尔值
布尔值核心类型是专为索引布尔值（true或false）设计的。基于布尔值类型的字段定义如下所示：
```
"allowed" : { "type" : "boolean", "store": "yes" }
```

### 二进制
二进制字段是存储在索引中的二进制数据的Base64表示，可用来存储以二进制形式正常写入的数据，例如图像。基于此类型的字段在默认情况下只被存储，而不索引，因此只能提取，但无法对其执行搜索操作。二进制类型只支持index_name 属性。基于binary 字段的字段定义如下所示：
```
"image" : { "type" : "binary" }
```

### 日期
日期核心类型被设计用于日期的索引。它遵循一个特定的、可改变的格式，并默认使用UTC保存。

能被Elasticsearch理解的默认日期格式是相当普遍的，它允许指定日期，也可指定时间，例如，2012-12-24T12:10:22 。基于日期类型的字段的示例定义如下所示：
```
"published" : { "type" : "date", "store" : "yes", "format" : "YYYY-mm-dd" }
```
除了公共属性，日期类型的字段还可以设置以下属性。
- format ：此属性指定日期的格式。默认值为dateOptionalTime 。对于格式的完整列表，请访问http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/mapping-date-ormat.html 。
- precision_step ：此属性指定在该字段中的每个值生成的词条数。该值越低，产生的词条数越高，从而范围查询的速度越快（但索引大小增加）。默认值是4。
- ignore_malformed ：此属性值可以设为true 或false ，默认值是false 。若要忽略格式错误的值，则应设置属性值为true 。

多字段
有时候你希望两个字段中有相同的字段值，例如，一个字段用于搜索，一个字段用于排序；或一个经语言分析器分析，一个只基于空白字符来分析。Elasticsearch允许加入多字段对象来拓展字段定义，从而解决这个需求。它允许把几个核心类型映射到单个字段，并逐个分析。例如，想计算切面并在name字段中搜索，可以定义以下字段：
```
"name": { 
  "type": "string", 
  "fields": { 
    "facet": { "type" : "string", "index": "not_analyzed" } 
  } 
}
```
上述定义将创建两个字段：我们将第一个字段称为name ，第二个称为name.facet 。当然，你不必在索引的过程中指定两个独立字段，指定一个name 字段就足够了。Elasticsearch会处理余下的工作，将该字段的数值复制到多字段定义的所有字段。

### IP地址类型
Elasticsearch添加了IP字段类型，以数字形式简化IPv4地址的使用。此字段类型可以帮搜索作为IP地址索引的数据、对这些数据排序，并使用IP值做范围查询。

基于IP地址类型的字段示例定义如下所示：
```
"address" : { "type" : "ip", "store" : "yes" }
```
除公共属性外，IP地址类型的字段还可以设置precision_step 属性。该属性指定了字段中的每个值生成的词条数。值越低，词条数越高。对于每个值的词条数更高的字段，范围查询会更快，但索引会稍微大点，默认值为4。
```
{ 
  "name" : "Tom PC", 
  "address" : "192.168.2.123" 
}
```

### token_count 类型
token_count 字段类型允许存储有关索引的字数信息，而不是存储及检索该字段的文本。它接受与number 类型相同的配置选项，此外，还可以通过analyzer 属性来指定分析器。

基于token_count 类型的字段示例定义如下：
```
"address_count" : { "type" : "token_count", "store" : "yes" }
```

## 定义自己的分析器
除了前面提到的分析器，Elasticsearch还允许我们定义新的分析器，而无需编写Java代码。为此，需要在映射文件中加入settings 节，它包含Elasticsearch创建索引时所需要的有用信息。下面演示了如何自定义settings 节：
```
"settings" : { 
  "index" : { 
    "analysis": { 
      "analyzer": { 
        "en": { 
          "tokenizer": "standard", 
          "filter": [ 
            "asciifolding", 
            "lowercase", 
            "ourEnglishFilter" 
          ] 
        } 
      }, 
      "filter": { 
        "ourEnglishFilter": { 
          "type": "kstem" 
        }
      }
    }
  }
}
```
我们指定一个新的名为en 的分析器。每个分析器由一个分词器和多个过滤器构成。我们的en 分析器包括standard 分词器和三个过滤器：默认情况下可用的asciifolding 和lowercase ，以及一个自定义的ourEnglishFilter 。

所以，定义了分析器的映射文件最终将如下所示：
```json
{ 
  "settings" : { 
    "index" : { 
      "analysis": { 
        "analyzer": { 
          "en": { 
            "tokenizer": "standard", 
            "filter": [ 
              "asciifolding", 
              "lowercase", 
              "ourEnglishFilter" 
            ] 
          }
        },
        "filter": { 
          "ourEnglishFilter": { 
            "type": "kstem" 
          }
        }
      }
    }
  },
  "mappings" : { 
    "post" : { 
      "properties" : { 
        "id": { "type" : "long", "store" : "yes", 
        "precision_step" : "0" }, 
        "name": { "type" : "string", "store" : "yes", "index" : 
        "analyzed", "analyzer": "en" } 
      }
    }
  }
}
```
可以通过Analyze API了解分析器的工作情况
```
curl -XGET 'localhost:9200/posts/_analyze?pretty&field=post.name' -d 'robots cars'
```

## _source 字段
_source 字段可以在生成索引过程中存储发送到Elasticsearch的原始JSON文档。默认情况下，_source 字段会被开启，因为部分Elasticsearch功能依赖于这个字段（如局部更新功能）。除此之外，当某字段没有存储时，_source 字段可用作高亮功能的数据源。但如果不需要这样的功能，可禁用_source 字段避免存储开销。为此，需设置_source 对象的enabled 属性值为false ，如下所示：
```
{
  "book" : {
    "_source" : {
      "enabled" : false
    },
    "properties" : {
      .
      .
      .
    }
  }
}
```
还可以告诉Elasticsearch我们希望从_source 字段中排除哪些字段，包含哪些字段。可以通过在_source 字段定义中添加includes 或excludes 属性来实现。如果想从_source 中排除author 路径下的所有字段，映射将如下所示：
```
{
  "book" : {
    "_source" : {
      "excludes" : [ "author.*" ]
    },
    "properties" : {
      .
      .
      .
    }
  }
}
```

## 路由介绍
默认情况下，Elasticsearch会在所有索引的分片中均匀地分配文档。然而，这并不总是理想情况。为了获得文档，Elasticsearch必须查询所有分片并合并结果。然而，如果你可以把数据按照一定的依据来划分（例如，客户端标识符），就可以使用一个强大的文档和查询分布控制机制：路由。简而言之，它允许选择用于索引和搜索数据的分片。

## 默认索引过程
在创建索引的过程中，当你发送文档时，Elasticsearch会根据文档的标识符，选择文档应编入索引的分片。默认情况下，Elasticsearch计算文档标识符的散列值，以此为基础将文档放置于一个可用的主分片上。

## 查询Elasticsearch

### 简单查询
查询Elasticsearch最简单的办法是使用URI请求查询。例如，为了搜索title 字段中的crime 一词，使用下面的命令：
```
curl -XGET 'localhost:9200/library/book/_search?q=title:crime&pretty=true'
```
如果从Elasticsearch的查询DSL的视点来看，上面的查询是一种query_string 查询，它查询title 字段中含有crime 一词的文档，可以这样写：
```
{ 
  "query" : { 
    "query_string" : { "query" : "title:crime" } 
  } 
}
```
发送HTTP GET请求到_search 这个REST端点，并在请求主体中附上查询。来看看下面的命令：
```
curl -XGET 'localhost:9200/library/book/_search?pretty=true' -d '{ 
  "query" : { 
    "query_string" : { "query" : "title:crime" } 
  } 
}'
```

### 分页和结果集大小
Elasticsearch能控制想要的最多结果数以及想从哪个结果开始。下面是可以在请求体中添加的两个额外参数。

- from ：该属性指定我们希望在结果中返回的起始文档。它的默认值是0 ，表示想要得到从第一个文档开始的结果。
- size ：该属性指定了一次查询中返回的最大文档数，默认值为10 。如果只对切面结果感兴趣，并不关心文档本身，可以把这个参数设置成0 。

如果想让查询从第10个文档开始返回20个文档，可以发送如下查询：
```json
{ 
  "from" : 9, 
  "size" : 20, 
  "query" : { 
    "query_string" : { "query" : "title:crime" } 
  } 
}
```