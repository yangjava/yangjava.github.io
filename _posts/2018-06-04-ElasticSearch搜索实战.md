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

### 空搜索
搜索API的最基础的形式是没有指定任何查询的空搜索，它简单地返回集群中所有索引下的所有文档：
```
GET /_search
```
返回的结果（为了界面简洁编辑过的）像这样：
```
{
   "hits" : {
      "total" :       14,
      "hits" : [
        {
          "_index":   "us",
          "_type":    "tweet",
          "_id":      "7",
          "_score":   1,
          "_source": {
             "date":    "2014-09-17",
             "name":    "John Smith",
             "tweet":   "The Query DSL is really powerful and flexible",
             "user_id": 2
          }
       },
        ... 9 RESULTS REMOVED ...
      ],
      "max_score" :   1
   },
   "took" :           4,
   "_shards" : {
      "failed" :      0,
      "successful" :  10,
      "total" :       10
   },
   "timed_out" :      false
}
```
- hits
返回结果中最重要的部分是 hits ，它包含 total 字段来表示匹配到的文档总数，并且一个 hits 数组包含所查询结果的前十个文档。

在 hits 数组中每个结果包含文档的 _index 、 _type 、 _id ，加上 _source 字段。这意味着我们可以直接从返回的搜索结果中使用整个文档。这不像其他的搜索引擎，仅仅返回文档的ID，需要你单独去获取文档。

每个结果还有一个 _score ，它衡量了文档与查询的匹配程度。默认情况下，首先返回最相关的文档结果，就是说，返回的文档是按照 _score 降序排列的。在这个例子中，我们没有指定任何查询，故所有的文档具有相同的相关性，因此对所有的结果而言 1 是中性的 _score 。

max_score 值是与查询所匹配文档的 _score 的最大值。

- took
took 值告诉我们执行整个搜索请求耗费了多少毫秒。

- shards
_shards 部分告诉我们在查询中参与分片的总数，以及这些分片成功了多少个失败了多少个。正常情况下我们不希望分片失败，但是分片失败是可能发生的。如果我们遭遇到一种灾难级别的故障，在这个故障中丢失了相同分片的原始数据和副本，那么对这个分片将没有可用副本来对搜索请求作出响应。假若这样，Elasticsearch 将报告这个分片是失败的，但是会继续返回剩余分片的结果。

- timeout
timed_out 值告诉我们查询是否超时。默认情况下，搜索请求不会超时。如果低响应时间比完成结果更重要，你可以指定 timeout 为 10 或者 10ms（10毫秒），或者 1s（1秒）：

- GET /_search?timeout=10ms
在请求超时之前，Elasticsearch 将会返回已经成功从每个分片获取的结果。

应当注意的是 timeout 不是停止执行查询，它仅仅是告知正在协调的节点返回到目前为止收集的结果并且关闭连接。在后台，其他的分片可能仍在执行查询即使是结果已经被发送了。

使用超时是因为 SLA(服务等级协议)对你是很重要的，而不是因为想去中止长时间运行的查询。

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
和 SQL 使用 LIMIT 关键字返回单个 page 结果的方法相同，Elasticsearch 接受 from 和 size 参数：
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

### 轻量搜索
一个 GET 是相当简单的，可以直接得到指定的文档。 现在尝试点儿稍微高级的功能，比如一个简单的搜索！

第一个尝试的几乎是最简单的搜索了。我们使用下列请求来搜索所有雇员：
```
GET /megacorp/employee/_search
```
返回结果不仅告知匹配了哪些文档，还包含了整个文档本身：显示搜索结果给最终用户所需的全部信息。

尝试下搜索姓氏为 ``Smith`` 的雇员。
```
GET /megacorp/employee/_search?q=last_name:Smith
```
我们仍然在请求路径中使用 _search 端点，并将查询本身赋值给参数 q= 。返回结果给出了所有的 Smith：

### 使用查询表达式搜索
Query-string 搜索通过命令非常方便地进行临时性的即席搜索 ，但它有自身的局限性。Elasticsearch 提供一个丰富灵活的查询语言叫做 查询表达式，它支持构建更加复杂和健壮的查询。

领域特定语言 （DSL）， 使用 JSON 构造了一个请求。我们可以像这样重写之前的查询所有名为 Smith 的搜索 ：
```
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```
返回结果与之前的查询一样，但还是可以看到有一些变化。其中之一是，不再使用 query-string 参数，而是一个请求体替代。这个请求使用 JSON 构造，并使用了一个 match 查询

现在尝试下更复杂的搜索。 同样搜索姓氏为 Smith 的员工，但这次我们只需要年龄大于 30 的。查询需要稍作调整，使用过滤器 filter ，它支持高效地执行一个结构化查询。
```
GET /megacorp/employee/_search
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}
```
与我们之前使用的 match 查询 一样。 一个 range 过滤器 ， 它能找到年龄大于 30 的文档，其中 gt 表示_大于_(great than)。

### 全文搜索
截止目前的搜索相对都很简单：单个姓名，通过年龄过滤。现在尝试下稍微高级点儿的全文搜索——一项 传统数据库确实很难搞定的任务。

搜索下所有喜欢攀岩（rock climbing）的员工：
```
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}
```
显然我们依旧使用之前的 match 查询在`about` 属性上搜索 “rock climbing” 。

Elasticsearch 默认按照相关性得分排序，即每个文档跟查询的匹配程度。第一个最高得分的结果很明显：John Smith 的 about 属性清楚地写着 “rock climbing” 。

但为什么 Jane Smith 也作为结果返回了呢？原因是她的 about 属性里提到了 “rock” 。因为只有 “rock” 而没有 “climbing” ，所以她的相关性得分低于 John 的。

这是一个很好的案例，阐明了 Elasticsearch 如何 在 全文属性上搜索并返回相关性最强的结果。Elasticsearch中的 相关性 概念非常重要，也是完全区别于传统关系型数据库的一个概念，数据库中的一条记录要么匹配要么不匹配。

### 短语搜索
找出一个属性中的独立单词是没有问题的，但有时候想要精确匹配一系列单词或者_短语_ 。 比如， 我们想执行这样一个查询，仅匹配同时包含 “rock” 和 “climbing” ，并且 二者以短语 “rock climbing” 的形式紧挨着的雇员记录。

为此对 match 查询稍作调整，使用一个叫做 match_phrase 的查询：
```
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
```
毫无悬念，返回结果仅有 John Smith 的文档。

### 高亮搜索
许多应用都倾向于在每个搜索结果中 高亮 部分文本片段，以便让用户知道为何该文档符合查询条件。在 Elasticsearch 中检索出高亮片段也很容易。

再次执行前面的查询，并增加一个新的 highlight 参数：
```
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}
```
当执行该查询时，返回结果与之前一样，与此同时结果中还多了一个叫做 highlight 的部分。这个部分包含了 about 属性匹配的文本片段，并以 HTML 标签 <em></em> 封装。

### 分析
终于到了最后一个业务需求：支持管理者对员工目录做分析。 Elasticsearch 有一个功能叫聚合（aggregations），允许我们基于数据生成一些精细的分析结果。聚合与 SQL 中的 GROUP BY 类似但更强大。

举个例子，挖掘出员工中最受欢迎的兴趣爱好：
```
GET /megacorp/employee/_search
{
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests" }
    }
  }
}
```
这些聚合的结果数据并非预先统计，而是根据匹配当前查询的文档即时生成的。如果想知道叫 Smith 的员工中最受欢迎的兴趣爱好，可以直接构造一个组合查询：
```
GET /megacorp/employee/_search
{
  "query": {
    "match": {
      "last_name": "smith"
    }
  },
  "aggs": {
    "all_interests": {
      "terms": {
        "field": "interests"
      }
    }
  }
}
```
聚合还支持分级汇总 。比如，查询特定兴趣爱好员工的平均年龄：
```
GET /megacorp/employee/_search
{
    "aggs" : {
        "all_interests" : {
            "terms" : { "field" : "interests" },
            "aggs" : {
                "avg_age" : {
                    "avg" : { "field" : "age" }
                }
            }
        }
    }
}
```

### 多索引，多类型
经常的情况下，你想在一个或多个特殊的索引并且在一个或者多个特殊的类型中进行搜索。我们可以通过在URL中指定特殊的索引和类型达到这种效果，如下所示：
```
/_search
在所有的索引中搜索所有的类型
/gb/_search
在 gb 索引中搜索所有的类型
/gb,us/_search
在 gb 和 us 索引中搜索所有的文档
/g*,u*/_search
在任何以 g 或者 u 开头的索引中搜索所有的类型
/gb/user/_search
在 gb 索引中搜索 user 类型
/gb,us/user,tweet/_search
在 gb 和 us 索引中搜索 user 和 tweet 类型
/_all/user,tweet/_search
在所有的索引中搜索 user 和 tweet 类型
```
当在单一的索引下进行搜索的时候，Elasticsearch 转发请求到索引的每个分片中，可以是主分片也可以是副本分片，然后从每个分片中收集结果。多索引搜索恰好也是用相同的方式工作的—只是会涉及到更多的分片。

### 最重要的查询
虽然 Elasticsearch 自带了很多的查询，但经常用到的也就那么几个。我们将在 深入搜索 章节详细讨论那些查询的细节，接下来我们对最重要的几个查询进行简单介绍。
- match_all 查询
match_all 查询简单的匹配所有文档。在没有指定查询方式时，它是默认的查询：
```
{ "match_all": {}}
```
它经常与 filter 结合使用—例如，检索收件箱里的所有邮件。所有邮件被认为具有相同的相关性，所以都将获得分值为 1 的中性 _score。

- match 查询
无论你在任何字段上进行的是全文搜索还是精确查询，match 查询是你可用的标准查询。

如果你在一个全文字段上使用 match 查询，在执行查询前，它将用正确的分析器去分析查询字符串：
```
{ "match": { "tweet": "About Search" }}
```
如果在一个精确值的字段上使用它，例如数字、日期、布尔或者一个 not_analyzed 字符串字段，那么它将会精确匹配给定的值：
```
{ "match": { "age":    26           }}
{ "match": { "date":   "2014-09-01" }}
{ "match": { "public": true         }}
{ "match": { "tag":    "full_text"  }}
```

- multi_match 查询
multi_match 查询可以在多个字段上执行相同的 match 查询：
```
{
    "multi_match": {
        "query":    "full text search",
        "fields":   [ "title", "body" ]
    }
}
```

- range 查询
range 查询找出那些落在指定区间内的数字或者时间：
```
{
    "range": {
        "age": {
            "gte":  20,
            "lt":   30
        }
    }
}
```

- term 查询
term 查询被用于精确值匹配，这些精确值可能是数字、时间、布尔或者那些 not_analyzed 的字符串：
```
{ "term": { "age":    26           }}
{ "term": { "date":   "2014-09-01" }}
{ "term": { "public": true         }}
{ "term": { "tag":    "full_text"  }}
```

- terms 查询
terms 查询和 term 查询一样，但它允许你指定多值进行匹配。如果这个字段包含了指定值中的任何一个值，那么这个文档满足条件：
```
{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}
```

- exists 查询和 missing 查询
exists 查询和 missing 查询被用于查找那些指定字段中有值 (exists) 或无值 (missing) 的文档。这与SQL中的 IS_NULL (missing) 和 NOT IS_NULL (exists) 在本质上具有共性：
```
{
    "exists":   {
        "field":    "title"
    }
}
```

### 组合多查询
现实的查询需求从来都没有那么简单；它们需要在多个字段上查询多种多样的文本，并且根据一系列的标准来过滤。为了构建类似的高级查询，你需要一种能够将多查询组合成单一查询的查询方法。

你可以用 bool 查询来实现你的需求。这种查询将多查询组合在一起，成为用户自己想要的布尔查询。它接收以下参数：
- must
文档 必须 匹配这些条件才能被包含进来。
- must_not
文档 必须不 匹配这些条件才能被包含进来。
- should
如果满足这些语句中的任意语句，将增加 _score ，否则，无任何影响。它们主要用于修正每个文档的相关性得分。
- filter
必须 匹配，但它以不评分、过滤模式来进行。这些语句对评分没有贡献，只是根据过滤标准来排除或包含文档。

由于这是我们看到的第一个包含多个查询的查询，所以有必要讨论一下相关性得分是如何组合的。每一个子查询都独自地计算文档的相关性得分。一旦他们的得分被计算出来， bool 查询就将这些得分进行合并并且返回一个代表整个布尔操作的得分。

下面的查询用于查找 title 字段匹配 how to make millions 并且不被标识为 spam 的文档。那些被标识为 starred 或在2014之后的文档，将比另外那些文档拥有更高的排名。如果 两者 都满足，那么它排名将更高：
```
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }},
            { "range": { "date": { "gte": "2014-01-01" }}}
        ]
    }
}
```
如果没有 must 语句，那么至少需要能够匹配其中的一条 should 语句。但，如果存在至少一条 must 语句，则对 should 语句的匹配没有要求。

增加带过滤器（filtering）的查询
如果我们不想因为文档的时间而影响得分，可以用 filter 语句来重写前面的例子：
```
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }}
        ],
        "filter": {
          "range": { "date": { "gte": "2014-01-01" }} 
        }
    }
}
```
通过将 range 查询移到 filter 语句中，我们将它转成不评分的查询，将不再影响文档的相关性排名。由于它现在是一个不评分的查询，可以使用各种对 filter 查询有效的优化手段来提升性能。

所有查询都可以借鉴这种方式。将查询移到 bool 查询的 filter 语句中，这样它就自动的转成一个不评分的 filter 了。

constant_score 查询
尽管没有 bool 查询使用这么频繁，constant_score 查询也是你工具箱里有用的查询工具。它将一个不变的常量评分应用于所有匹配的文档。它被经常用于你只需要执行一个 filter 而没有其它查询（例如，评分查询）的情况下。

可以使用它来取代只有 filter 语句的 bool 查询。在性能上是完全相同的，但对于提高查询简洁性和清晰度有很大帮助。
```
{
    "constant_score":   {
        "filter": {
            "term": { "category": "ebooks" } 
        }
    }
}
```

## 验证查询
查询可以变得非常的复杂，尤其和不同的分析器与不同的字段映射结合时，理解起来就有点困难了。不过 validate-query API 可以用来验证查询是否合法。
```
GET /gb/tweet/_validate/query
{
   "query": {
      "tweet" : {
         "match" : "really powerful"
      }
   }
}
```
以上 validate 请求的应答告诉我们这个查询是不合法的：
```
{
  "valid" :         false,
  "_shards" : {
    "total" :       1,
    "successful" :  1,
    "failed" :      0
  }
}
```
理解错误信息
为了找出 查询不合法的原因，可以将 explain 参数 加到查询字符串中：
```
GET /gb/tweet/_validate/query?explain 
{
   "query": {
      "tweet" : {
         "match" : "really powerful"
      }
   }
}
```
很明显，我们将查询类型(match)与字段名称 (tweet)搞混了：
```
{
  "valid" :     false,
  "_shards" :   { ... },
  "explanations" : [ {
    "index" :   "gb",
    "valid" :   false,
    "error" :   "org.elasticsearch.index.query.QueryParsingException:
                 [gb] No query registered for [tweet]"
  } ]
}
```

## 理解查询语句
对于合法查询，使用 explain 参数将返回可读的描述，这对准确理解 Elasticsearch 是如何解析你的 query 是非常有用的：
```
GET /_validate/query?explain
{
   "query": {
      "match" : {
         "tweet" : "really powerful"
      }
   }
}
```
我们查询的每一个 index 都会返回对应的 explanation ，因为每一个 index 都有自己的映射和分析器：
```
{
  "valid" :         true,
  "_shards" :       { ... },
  "explanations" : [ {
    "index" :       "us",
    "valid" :       true,
    "explanation" : "tweet:really tweet:powerful"
  }, {
    "index" :       "gb",
    "valid" :       true,
    "explanation" : "tweet:realli tweet:power"
  } ]
}
```
从 explanation 中可以看出，匹配 really powerful 的 match 查询被重写为两个针对 tweet 字段的 single-term 查询，一个single-term查询对应查询字符串分出来的一个term。

当然，对于索引 us ，这两个 term 分别是 really 和 powerful ，而对于索引 gb ，term 则分别是 realli 和 power 。之所以出现这个情况，是由于我们将索引 gb 中 tweet 字段的分析器修改为 english 分析器。

## 排序
为了按照相关性来排序，需要将相关性表示为一个数值。在 Elasticsearch 中， 相关性得分 由一个浮点数进行表示，并在搜索结果中通过 _score 参数返回， 默认排序是 _score 降序。

有时，相关性评分对你来说并没有意义。例如，下面的查询返回所有 user_id 字段包含 1 的结果：
```
GET /_search
{
    "query" : {
        "bool" : {
            "filter" : {
                "term" : {
                    "user_id" : 1
                }
            }
        }
    }
}
```
这里没有一个有意义的分数：因为我们使用的是 filter （过滤），这表明我们只希望获取匹配 user_id: 1 的文档，并没有试图确定这些文档的相关性。 实际上文档将按照随机顺序返回，并且每个文档都会评为零分。


如果评分为零对你造成了困扰，你可以使用 constant_score 查询进行替代：
```
GET /_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "term" : {
                    "user_id" : 1
                }
            }
        }
    }
}
```
这将让所有文档应用一个恒定分数（默认为 1 ）。它将执行与前述查询相同的查询，并且所有的文档将像之前一样随机返回，这些文档只是有了一个分数而不是零分。

### 按照字段的值排序
在这个案例中，通过时间来对 tweets 进行排序是有意义的，最新的 tweets 排在最前。 我们可以使用 sort 参数进行实现：
```
GET /_search
{
    "query" : {
        "bool" : {
            "filter" : { "term" : { "user_id" : 1 }}
        }
    },
    "sort": { "date": { "order": "desc" }}
}
```
你会注意到结果中的两个不同点：
```
"hits" : {
    "total" :           6,
    "max_score" :       null, 
    "hits" : [ {
        "_index" :      "us",
        "_type" :       "tweet",
        "_id" :         "14",
        "_score" :      null, 
        "_source" :     {
             "date":    "2014-09-24",
             ...
        },
        "sort" :        [ 1411516800000 ] 
    },
    ...
}
```
- _score 不被计算, 因为它并没有用于排序。
- date 字段的值表示为自 epoch (January 1, 1970 00:00:00 UTC)以来的毫秒数，通过 sort 字段的值进行返回。
首先我们在每个结果中有一个新的名为 sort 的元素，它包含了我们用于排序的值。 在这个案例中，我们按照 date 进行排序，在内部被索引为 自 epoch 以来的毫秒数 。 long 类型数 1411516800000 等价于日期字符串 2014-09-24 00:00:00 UTC 。

其次 _score 和 max_score 字段都是 null 。计算 _score 的花销巨大，通常仅用于排序； 我们并不根据相关性排序，所以记录 _score 是没有意义的。如果无论如何你都要计算 _score ， 你可以将 track_scores 参数设置为 true 。

一个简便方法是, 你可以指定一个字段用来排序：
```
"sort": "number_of_children"
```
字段将会默认升序排序，而按照 _score 的值进行降序排序。

## 多级排序
假定我们想要结合使用 date 和 _score 进行查询，并且匹配的结果首先按照日期排序，然后按照相关性排序：
```
GET /_search
{
    "query" : {
        "bool" : {
            "must":   { "match": { "tweet": "manage text search" }},
            "filter" : { "term" : { "user_id" : 2 }}
        }
    },
    "sort": [
        { "date":   { "order": "desc" }},
        { "_score": { "order": "desc" }}
    ]
}
```
排序条件的顺序是很重要的。结果首先按第一个条件排序，仅当结果集的第一个 sort 值完全相同时才会按照第二个条件进行排序，以此类推。

多级排序并不一定包含 _score 。你可以根据一些不同的字段进行排序，如地理距离或是脚本计算的特定值。

Query-string 搜索 也支持自定义排序，可以在查询字符串中使用 sort 参数：
```
GET /_search?sort=date:desc&sort=_score&q=search
```

## 多值字段的排序
一种情形是字段有多个值的排序， 需要记住这些值并没有固有的顺序；一个多值的字段仅仅是多个值的包装，这时应该选择哪个进行排序呢？

对于数字或日期，你可以将多值字段减为单值，这可以通过使用 min 、 max 、 avg 或是 sum 排序模式 。 例如你可以按照每个 date 字段中的最早日期进行排序，通过以下方法：
```
"sort": {
    "dates": {
        "order": "asc",
        "mode":  "min"
    }
}
```

## 字符串排序与多字段
被解析的字符串字段也是多值字段， 但是很少会按照你想要的方式进行排序。如果你想分析一个字符串，如 fine old art ， 这包含 3 项。我们很可能想要按第一项的字母排序，然后按第二项的字母排序，诸如此类，但是 Elasticsearch 在排序过程中没有这样的信息。

你可以使用 min 和 max 排序模式（默认是 min ），但是这会导致排序以 art 或是 old ，任何一个都不是所希望的。

为了以字符串字段进行排序，这个字段应仅包含一项： 整个 not_analyzed 字符串。 但是我们仍需要 analyzed 字段，这样才能以全文进行查询

一个简单的方法是用两种方式对同一个字符串进行索引，这将在文档中包括两个字段： analyzed 用于搜索， not_analyzed 用于排序

但是保存相同的字符串两次在 _source 字段是浪费空间的。 我们真正想要做的是传递一个 单字段 但是却用两种方式索引它。所有的 _core_field 类型 (strings, numbers, Booleans, dates) 接收一个 fields 参数

该参数允许你转化一个简单的映射如：
```
"tweet": {
    "type":     "string",
    "analyzer": "english"
}
```
为一个多字段映射如：
```
"tweet": { 
    "type":     "string",
    "analyzer": "english",
    "fields": {
        "raw": { 
            "type":  "string",
            "index": "not_analyzed"
        }
    }
}
```
- tweet 主字段与之前的一样: 是一个 analyzed 全文字段。
- 新的 tweet.raw 子字段是 not_analyzed.
现在，至少只要我们重新索引了我们的数据，使用 tweet 字段用于搜索，tweet.raw 字段用于排序：
```
GET /_search
{
    "query": {
        "match": {
            "tweet": "elasticsearch"
        }
    },
    "sort": "tweet.raw"
}
```

## 什么是相关性?
 但是什么是相关性？ 相关性如何计算？ 每个文档都有相关性评分，用一个正浮点数字段 _score 来表示 。 _score 的评分越高，相关性越高。

查询语句会为每个文档生成一个 _score 字段。评分的计算方式取决于查询类型 不同的查询语句用于不同的目的： fuzzy 查询会计算与关键词的拼写相似程度，terms 查询会计算 找到的内容与关键词组成部分匹配的百分比，但是通常我们说的 relevance 是我们用来计算全文本字段的值相对于全文本检索词相似程度的算法。

Elasticsearch 的相似度算法被定义为检索词频率/反向文档频率， TF/IDF ，包括以下内容：
- 检索词频率
检索词在该字段出现的频率？出现频率越高，相关性也越高。 字段中出现过 5 次要比只出现过 1 次的相关性高。
- 反向文档频率
每个检索词在索引中出现的频率？频率越高，相关性越低。检索词出现在多数文档中会比出现在少数文档中的权重更低。
- 字段长度准则
字段的长度是多少？长度越长，相关性越低。 检索词出现在一个短的 title 要比同样的词出现在一个长的 content 字段权重更大。
单个查询可以联合使用 TF/IDF 和其他方式，比如短语查询中检索词的距离或模糊查询里的检索词相似度。

相关性并不只是全文本检索的专利。也适用于 yes|no 的子句，匹配的子句越多，相关性评分越高。 如果多条查询子句被合并为一条复合查询语句，比如 bool 查询，则每个查询子句计算得出的评分会被合并到总的相关性评分中。

### 理解评分标准
当调试一条复杂的查询语句时，想要理解 _score 究竟是如何计算是比较困难的。Elasticsearch 在 每个查询语句中都有一个 explain 参数，将 explain 设为 true 就可以得到更详细的信息。
```
GET /_search?explain 
{
   "query"   : { "match" : { "tweet" : "honeymoon" }}
}
```
explain 参数可以让返回结果添加一个 _score 评分的得来依据。

它提供了 _explanation 。每个入口都包含一个 description 、 value 、 details 字段，它分别告诉你计算的类型、计算结果和任何我们需要的计算细节。
```
"_explanation": { 
   "description": "weight(tweet:honeymoon in 0)
                  [PerFieldSimilarity], result of:",
   "value":       0.076713204,
   "details": [
      {
         "description": "fieldWeight in 0, product of:",
         "value":       0.076713204,
         "details": [
            {  
               "description": "tf(freq=1.0), with freq of:",
               "value":       1,
               "details": [
                  {
                     "description": "termFreq=1.0",
                     "value":       1
                  }
               ]
            },
            { 
               "description": "idf(docFreq=1, maxDocs=1)",
               "value":       0.30685282
            },
            { 
               "description": "fieldNorm(doc=0)",
               "value":        0.25,
            }
         ]
      }
   ]
}
```
然后它提供了权重是如何计算的细节：
- 检索词频率:
检索词 `honeymoon` 在这个文档的 `tweet` 字段中的出现次数。
- 反向文档频率:
检索词 `honeymoon` 在索引上所有文档的 `tweet` 字段中出现的次数。
- 字段长度准则:
在这个文档中， `tweet` 字段内容的长度 -- 内容越长，值越小。

## 理解文档是如何被匹配到的
当 explain 选项加到某一文档上时， explain api 会帮助你理解为何这个文档会被匹配，更重要的是，一个文档为何没有被匹配。

请求路径为 /index/type/id/_explain ，如下所示：
```
GET /us/tweet/12/_explain
{
   "query" : {
      "bool" : {
         "filter" : { "term" :  { "user_id" : 2           }},
         "must" :  { "match" : { "tweet" :   "honeymoon" }}
      }
   }
}
```
不只是我们之前看到的充分解释 ，我们现在有了一个 description 元素，它将告诉我们：
```
"failure to match filter: cache(user_id:[2 TO 2])"
```
也就是说我们的 user_id 过滤子句使该文档不能匹配到。


## 结构化搜索
结构化搜索（Structured search） 是指有关探询那些具有内在结构数据的过程。比如日期、时间和数字都是结构化的：它们有精确的格式，我们可以对这些格式进行逻辑操作。比较常见的操作包括比较数字或时间的范围，或判定两个值的大小。

文本也可以是结构化的。如彩色笔可以有离散的颜色集合： 红（red） 、 绿（green） 、 蓝（blue） 。一个博客可能被标记了关键词 分布式（distributed） 和 搜索（search） 。电商网站上的商品都有 UPCs（通用产品码 Universal Product Codes）或其他的唯一标识，它们都需要遵从严格规定的、结构化的格式。

在结构化查询中，我们得到的结果 总是 非是即否，要么存于集合之中，要么存在集合之外。结构化查询不关心文件的相关度或评分；它简单的对文档包括或排除处理。

这在逻辑上是能说通的，因为一个数字不能比其他数字 更 适合存于某个相同范围。结果只能是：存于范围之中，抑或反之。同样，对于结构化文本来说，一个值要么相等，要么不等。没有 更似 这种概念。

## 多字段搜索
查询很少是简单一句话的 match 匹配查询。通常我们需要用相同或不同的字符串查询一个或多个字段，也就是说，需要对多个查询语句以及它们相关度评分进行合理的合并。

### 多字符串查询
最简单的多字段查询可以将搜索项映射到具体的字段。如果我们知道 War and Peace 是标题，Leo Tolstoy 是作者，很容易就能把两个条件用 match 语句表示，并将它们用 bool 查询 组合起来：
```
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title":  "War and Peace" }},
        { "match": { "author": "Leo Tolstoy"   }}
      ]
    }
  }
}
```
bool 查询采取 more-matches-is-better 匹配越多越好的方式，所以每条 match 语句的评分结果会被加在一起，从而为每个文档提供最终的分数 _score 。能与两条语句同时匹配的文档比只与一条语句匹配的文档得分要高。

当然，并不是只能使用 match 语句：可以用 bool 查询来包裹组合任意其他类型的查询，甚至包括其他的 bool 查询。我们可以在上面的示例中添加一条语句来指定译者版本的偏好：
```
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title":  "War and Peace" }},
        { "match": { "author": "Leo Tolstoy"   }},
        { "bool":  {
          "should": [
            { "match": { "translator": "Constance Garnett" }},
            { "match": { "translator": "Louise Maude"      }}
          ]
        }}
      ]
    }
  }
}
```
为什么将译者条件语句放入另一个独立的 bool 查询中呢？所有的四个 match 查询都是 should 语句，所以为什么不将 translator 语句与其他如 title 、 author 这样的语句放在同一层呢？

答案在于评分的计算方式。 bool 查询运行每个 match 查询，再把评分加在一起，然后将结果与所有匹配的语句数量相乘，最后除以所有的语句数量。处于同一层的每条语句具有相同的权重。在前面这个例子中，包含 translator 语句的 bool 查询，只占总评分的三分之一。如果将 translator 语句与 title 和 author 两条语句放入同一层，那么 title 和 author 语句只贡献四分之一评分。

## multi_match 查询
multi_match 查询为能在多个字段上反复执行相同查询提供了一种便捷方式。

multi_match 多匹配查询的类型有多种，其中的三种恰巧与 了解我们的数据 中介绍的三个场景对应，即： best_fields 、 most_fields 和 cross_fields （最佳字段、多数字段、跨字段）。

默认情况下，查询的类型是 best_fields ，这表示它会为每个字段生成一个 match 查询，然后将它们组合到 dis_max 查询的内部，如下：
```
{
    "multi_match": {
        "query":                "Quick brown fox",
        "type":                 "best_fields", 
        "fields":               [ "title", "body" ],
        "tie_breaker":          0.3,
        "minimum_should_match": "30%" 
    }
}
```
best_fields 类型是默认值，可以不指定。
如 minimum_should_match 或 operator 这样的参数会被传递到生成的 match 查询中。

## 查询字段名称的模糊匹配
字段名称可以用模糊匹配的方式给出：任何与模糊模式正则匹配的字段都会被包括在搜索条件中，例如可以使用以下方式同时匹配 book_title 、 chapter_title 和 section_title （书名、章名、节名）这三个字段：
```
{
    "multi_match": {
        "query":  "Quick brown fox",
        "fields": "*_title"
    }
}
```

## 提升单个字段的权重
可以使用 ^ 字符语法为单个字段提升权重，在字段名称的末尾添加 ^boost ，其中 boost 是一个浮点数：
```
{
    "multi_match": {
        "query":  "Quick brown fox",
        "fields": [ "*_title", "chapter_title^2" ] 
    }
}
```
chapter_title 这个字段的 boost 值为 2 ，而其他两个字段 book_title 和 section_title 字段的默认 boost 值为 1 。

## 聚合查询
聚合（aggs）不同于普通查询，是目前学到的第二种大的查询分类，第一种即“query”，因此在代码中的第一层嵌套由“query”变为了“aggs”。用于进行聚合的字段必须是exact value，分词字段不可进行聚合，对于text字段如果需要使用聚合，需要开启fielddata，但是通常不建议，因为fielddata是将聚合使用的数据结构由磁盘（docvalues）变为了堆内存（fielddata），大数据的聚合操作很容易导致OOM，详细原理会在进阶篇中阐述。

聚合分类
分桶聚合（Bucket agregations）：类比SQL中的group by的作用，主要用于统计不同类型数据的数量
指标聚合（Metrics agregations）：主要用于最大值、最小值、平均值、字段之和等指标的统计
管道聚合（Pipeline agregations）：用于对聚合的结果进行二次聚合，如要统计绑定数量最多的标签bucket，就是要先按照标签进行分桶，再在分桶的结果上计算最大值。

```
json GET product/_search 
{
    "aggs": {
        "<aggs_name>": {
            "<agg_type>": {
                "field": "<field_name>"
            }
        }
    }
}
```
- aggs_name：聚合函数的名称
- agg_type：聚合种类，比如是桶聚合（terms）或者是指标聚合（avg、sum、min、max等）
- field_name：字段名称或者叫域名。

## 查询
Elasticsearch 查询语句采用基于 RESTful 风格的接口封装成 JSON 格式的对象，称之为 Query DSL。Elasticsearch 查询分类大致分为全文查询、词项查询、复合查询、嵌套查询、位置查询、特殊查询。

Elasticsearch 查询从机制分为两种，一种是根据用户输入的查询词，通过排序模型计算文档与查询词之间的相关度，并根据评分高低排序返回；另一种是过滤机制，只根据过滤条件对文档进行过滤，不计算评分，速度相对较快。

### 全文查询
es 全文查询主要用于在全文字段上，主要考虑查询词与文档的相关性（Relevance）。

### match query
match query 用于搜索单个字段，首先会针对查询语句进行解析（经过 analyzer），主要是对查询语句进行分词，分词后查询语句的任何一个词项被匹配，文档就会被搜到，默认情况下相当于对分词后词项进行 or 匹配操作。
```
GET article/_search
{
  "query": {
    "match": {
      "title": {
        "query": "Elasticsearch 查询优化"
      }
    }
  }
}
```
等同于 or 匹配操作，如下：
```
GET article/_search
{
  "query": {
    "match": {
      "title": {
        "query": "Elasticsearch 查询优化",
        "operator": "or"
      }
    }
  }
}
```
如果想查询匹配所有关键词的文档，可以用 and 操作符连接，如下：
```
GET article/_search
{
  "query": {
    "match": {
      "title": {
        "query": "Elasticsearch 查询优化",
        "operator": "and"
      }
    }
  }
}
```

### match_phrase query
match_phrase query 首先会把 query 内容分词，分词器可以自定义，同时文档还要满足以下两个条件才会被搜索到：
- 分词后所有词项都要出现在该字段中（相当于 and 操作）。
- 字段中的词项顺序要一致。

例如，有以下 3 个文档，使用 match_phrase 查询 “what a wonderful life”，只有前两个文档会被匹配：
```
PUT test_idx/test_tp/1
{ "desc": "what a wonderful life" }

PUT test_idx/test_tp/2
{ "desc": "what a life"}

PUT test_idx/test_tp/3
{ "desc": "life is what"}

GET test_idx/test_tp/_search
{
  "query": {
    "match_phrase": {
      "desc": "what life"
    }
  }
}
```

### match_phrase_prefix query
match_phrase_prefix 和 match_phrase 类似，只不过 match_phrase_prefix 支持最后一个 term 的前缀匹配。
```
GET test_idx/test_tp/_search
{
  "query": {
    "match_phrase_prefix": {
      "desc": "what li"
    }
  }
}
```

### multi_match query
multi_match 是 match 的升级，用于搜索多个字段。查询语句为 “java 编程”，查询域为 title 和 description，查询语句如下：
```
GET books/_search
{
  "query": {
    "multi_match": {
      "query": "java 编程",
      "fields": ["title", "description"]
    }
  }
}
```

multi_match 支持对要搜索的字段的名称使用通配符，示例如下：
```
GET books/_search
{
  "query": {
    "multi_match": {
      "query": "java 编程",
      "fields": ["title", "*_name"]
    }
  }
}
```
同时，也可以用指数符指定搜索字段的权重。指定关键词出现在 title 中的权重是出现在 description 字段中的 3 倍，命令如下：
```
GET books/_search
{
  "query": {
    "multi_match": {
      "query": "java 编程",
      "fields": ["title^3", "description"]
    }
  }
}
```

### common_terms query
common_terms query 是一种在不牺牲性能的情况下替代停用词提高搜索准确率和召回率的方案。

查询中的每个词项都有一定的代价，以搜索 “The brown fox” 为例，query 会被解析成三个词项 “the”“brown” 和“fox”，每个词项都会到索引中执行一次查询。很显然包含 “the” 的文档非常多，相比其他词项，“the”的重要性会低很多。传统的解决方案是把 “the” 当作停用词处理，去除停用词之后可以减少索引大小，同时在搜索时减少对停用词的收缩。

虽然停用词对文档评分影响不大，但是当停用词仍然有重要意义的时候，去除停用词就不是完美的解决方案了。如果去除停用词，就无法区分 “happy” 和“not happy”, “The”“To be or not to be”就不会在索引中存在，搜索的准确率和召回率就会降低。

common_terms query 提供了一种解决方案，它把 query 分词后的词项分成重要词项（低频词项）和不重要的词项（高频词，也就是之前的停用词）。在搜索的时候，首先搜索和重要词项匹配的文档，这些文档是词项出现较少并且词项对其评分影响较大的文档。然后执行第二次查询，搜索对评分影响较小的高频词项，但是不计算所有文档的评分，而是只计算第一次查询已经匹配的文档得分。如果一个查询中只包含高频词，那么会通过 and 连接符执行一个单独的查询，换言之，会搜索所有的词项。

词项是高频词还是低频词是通过 cutoff frequency 来设置阀值的，取值可以是绝对频率（频率大于 1）或者相对频率（0～1）。common_terms query 最有趣之处在于它能自适应特定领域的停用词，例如，在视频托管网站上，诸如 “clip” 或“video”之类的高频词项将自动表现为停用词，无须保留手动列表。

例如，文档频率高于 0.1% 的词项将会被当作高频词项，词频之间可以用 low_freq_operator、high_freq_operator 参数连接。设置低频词操作符为 “and” 使所有的低频词都是必须搜索的，示例代码如下：
```
GET books/_search
{
	"query": {
		"common": {
			"body": {
				"query": "nelly the elephant as a cartoon",
				"cutoff_frequency": 0.001,
				"low_freq_operator": "and"
			}
		}
	}
}
```
上述操作等价于：
```
GET books/_search
{
	"query": {
		"bool": {
			"must": [
			  { "term": { "body": "nelly" } },
			  { "term": { "body": "elephant" } },
			  { "term": { "body": "cartoon" } }
			],
			"should": [
			  { "term": { "body": "the" } },
			  { "term": { "body": "as" } },
			  { "term": { "body": "a" } }
			]
		}
	}
}
```

### query_string query
query_string query 是与 Lucene 查询语句的语法结合非常紧密的一种查询，允许在一个查询语句中使用多个特殊条件关键字（如：AND | OR | NOT）对多个字段进行查询，建议熟悉 Lucene 查询语法的用户去使用。

### simple_query_string
simple_query_string 是一种适合直接暴露给用户，并且具有非常完善的查询语法的查询语句，接受 Lucene 查询语法，解析过程中发生错误不会抛出异常。例子如下：
```
GET books/_search
{
  "query": {
    "simple_query_string": {
      "query": "\"fried eggs\" +(eggplant | potato) -frittata",
      "analyzer": "snowball",
      "fields": ["body^5", "_all"],
      "default_operator": "and"
    }
  }
}
```

### 词项查询
全文查询在执行查询之前会分析查询字符串，词项查询时对倒排索引中存储的词项进行精确匹配操作。词项级别的查询通常用于结构化数据，如数字、日期和枚举类型。

### term query
term 查询用来查找指定字段中包含给定单词的文档，term 查询不被解析，只有查询词和文档中的词精确匹配才会被搜索到，应用场景为查询人名、地名等需要精准匹配的需求。比如，查询 title 字段中含有关键词 “思想” 的书籍，查询命令如下：
```
GET books/_search
{
  "query": {
    "term": {
      "title": "思想"
    }
  }
}
```
避免 term 查询对 text 字段使用查询。 默认情况下，Elasticsearch 针对 text 字段的值进行解析分词，这会使查找 text 字段值的精确匹配变得困难。 要搜索 text 字段值，需改用 match 查询。

### terms query
terms 查询是 term 查询的升级，可以用来查询文档中包含多个词的文档。比如，想查询 title 字段中包含关键词 “java” 或 “python” 的文档，构造查询语句如下：
```
{
  "query": {
    "terms": {
      "title": ["java", "python"]
    }
  }
}
```

### range query
range query 即范围查询，用于匹配在某一范围内的数值型、日期类型或者字符串型字段的文档，比如搜索哪些书籍的价格在 50 到 100 之间、哪些书籍的出版时间在 2015 年到 2019 年之间。使用 range 查询只能查询一个字段，不能作用在多个字段上。

range 查询支持的参数有以下几种：
- gt 大于，查询范围的最小值，也就是下界，但是不包含临界值。
- gte 大于等于，和 gt 的区别在于包含临界值。
- lt 小于，查询范围的最大值，也就是上界，但是不包含临界值。
- lte 小于等于，和 lt 的区别在于包含临界值。

例如，想要查询价格大于 50，小于等于 70 的书籍，即 50 < price <= 70，构造查询语句如下：
```
GET bookes/_search
{
  "query": {
    "range": {
      "price": {
        "gt": 50,
        "lte": 70
      }
    }
  }
}
```

查询出版日期在 2015 年 1 月 1 日和 2019 年 12 月 31 之间的书籍，对 publish_time 字段进行范围查询，命令如下：
```
{
  "query": {
    "range": {
      "publish_ time": {
        "gte": "2015-01-01",
        "lte": "2019-12-31",
        "format": "yyyy-MM-dd"
      }
    }
  }
}
```

### exists query
exists 查询会返回字段中至少有一个非空值的文档。

举例说明如下：
```
{
  "query": {
    "exists": {
      "field": "user"
    }
  }
}
```
以下文档会匹配上面的查询：
- { "user" : "jane" } 有 user 字段，且不为空。
- { "user" : "" } 有 user 字段，值为空字符串。
- { "user" : "-" } 有 user 字段，值不为空。
- { "user" : [ "jane" ] } 有 user 字段，值不为空。
- { "user" : [ "jane", null ] } 有 user 字段，至少一个值不为空即可。

下面的文档都不会被匹配：
- { "user" : null } 虽然有 user 字段，但是值为空。
- { "user" : [] } 虽然有 user 字段，但是值为空。
- { "user" : [null] } 虽然有 user 字段，但是值为空。
- { "foo" : "bar" } 没有 user 字段。

### prefix query
prefix 查询用于查询某个字段中以给定前缀开始的文档，比如查询 title 中含有以 java 为前缀的关键词的文档，那么含有 java、javascript、javaee 等所有以 java 开头关键词的文档都会被匹配。查询 description 字段中包含有以 win 为前缀的关键词的文档，查询语句如下：
```
GET books/_search
{
	"query": {
		"prefix": {
			"description": "win"
		}
	}
}
```

### wildcard query
wildcard query 中文译为通配符查询，支持单字符通配符和多字符通配符，? 用来匹配一个任意字符，* 用来匹配零个或者多个字符。

以 H?tland 为例，Hatland、Hbtland 等都可以匹配，但是不能匹配 Htland，? 只能代表一位。H*tland 可以匹配 Htland、Habctland 等，* 可以代表 0 至多个字符。和 prefix 查询一样，wildcard 查询的查询性能也不是很高，需要消耗较多的 CPU 资源。

下面举一个 wildcard 查询的例子，假设需要找某一作者写的书，但是忘记了作者名字的全称，只记住了前两个字，那么就可以使用通配符查询，查询语句如下：
```
GET books/_search
{
  "query": {
    "wildcard": {
      "author": "李永*"
    }
  }
}
```

### regexp query
Elasticsearch 也支持正则表达式查询，通过 regexp query 可以查询指定字段包含与指定正则表达式匹配的文档。可以代表任意字符, “a.c.e” 和 “ab...” 都可以匹配 “abcde”，a{3}b{3}、a{2,3}b{2,4}、a{2,}{2,} 都可以匹配字符串 “aaabbb”。

例如需要匹配以 W 开头紧跟着数字的邮政编码，使用正则表达式查询构造查询语句如下：
```
GET books/_search
{
	"query": {
		"regexp": {
			"postcode": "W[0-9].+"
		}
	}
}
```

### fuzzy query
编辑距离又称 Levenshtein 距离，是指两个字串之间，由一个转成另一个所需的最少编辑操作次数。许可的编辑操作包括将一个字符替换成另一个字符，插入一个字符，删除一个字符。fuzzy 查询就是通过计算词项与文档的编辑距离来得到结果的，但是使用 fuzzy 查询需要消耗的资源比较大，查询效率不高，适用于需要模糊查询的场景。举例如下，用户在输入查询关键词时不小心把 “javascript” 拼成 “javascritp”，在存在拼写错误的情况下使用模糊查询仍然可以搜索到含有 “javascript” 的文档，查询语句如下：
```
GET books/_search
{
	"query": {
		"fuzzy": {
			"title": "javascritp"
		}
	}
}
```

### type query
type query 用于查询具有指定类型的文档。例如查询 Elasticsearch 中 type 为 computer 的文档，查询语句如下：
```
GET books/_search
{
	"query": {
		"type": {
			"value": "computer"
		}
	}
}
```

### ids query
ids query 用于查询具有指定 id 的文档。类型是可选的，也可以省略，也可以接受一个数组。如果未指定任何类型，则会查询索引中的所有类型。例如，查询类型为 computer，id 为 1、3、5 的文档，本质上是对文档 _id 的查询，所以对应的 value 是字符串类型，查询语句如下：
```
GET books/_search
{
  "query": {
    "ids": {
      "type": "computer",
      "values": ["1", "3", "5"]
    }
  }
}
```

### 复合查询
复合查询就是把一些简单查询组合在一起实现更复杂的查询需求，除此之外，复合查询还可以控制另外一个查询的行为。

### bool query
bool 查询可以把任意多个简单查询组合在一起，使用 must、should、must_not、filter 选项来表示简单查询之间的逻辑，每个选项都可以出现 0 次到多次，它们的含义如下：

- must 文档必须匹配 must 选项下的查询条件，相当于逻辑运算的 AND，且参与文档相关度的评分。
- should 文档可以匹配 should 选项下的查询条件也可以不匹配，相当于逻辑运算的 OR，且参与文档相关度的评分。
- must_not 与 must 相反，匹配该选项下的查询条件的文档不会被返回；需要注意的是，must_not 语句不会影响评分，它的作用只是将不相关的文档排除。
- filter 和 must 一样，匹配 filter 选项下的查询条件的文档才会被返回，但是 filter 不评分，只起到过滤功能，与 must_not 相反。

假设要查询 title 中包含关键词 java，并且 price 不能高于 70，description 可以包含也可以不包含虚拟机的书籍，构造 bool 查询语句如下：
```
GET books/_search
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "status": 1
        }
      },
      "must_not": {
        "range": {
          "price": {
            "gte": 70
          }
        }
      },
      "must": {
        "match": {
          "title": "java"
        }
      },
      "should": [
        {
          "match": {
            "description": "虚拟机"
          }
        }
      ],
      "minimum_should_match": 1
    }
  }
}
```

### boosting query
boosting 查询用于需要对两个查询的评分进行调整的场景，boosting 查询会把两个查询封装在一起并降低其中一个查询的评分。

boosting 查询包括 positive、negative 和 negative_boost 三个部分，positive 中的查询评分保持不变，negative 中的查询会降低文档评分，negative_boost 指明 negative 中降低的权值。如果我们想对 2015 年之前出版的书降低评分，可以构造一个 boosting 查询，查询语句如下：
```
GET books/_search
{
	"query": {
		"boosting": {
			"positive": {
				"match": {
					"title": "python"
				}
			},
			"negative": {
				"range": {
					"publish_time": {
						"lte": "2015-01-01"
					}
				}
			},
			"negative_boost": 0.2
		}
	}
}
```
boosting 查询中指定了抑制因子为 0.2，publish_time 的值在 2015-01-01 之后的文档得分不变，publish_time 的值在 2015-01-01 之前的文档得分为原得分的 0.2 倍。

### constant_score query
constant_score query 包装一个 filter query，并返回匹配过滤器查询条件的文档，且它们的相关性评分都等于 boost 参数值（可以理解为原有的基于 tf-idf 或 bm25 的相关分固定为 1.0，所以最终评分为 1.0 * boost，即等于 boost 参数值）。下面的查询语句会返回 title 字段中含有关键词 elasticsearch 的文档，所有文档的评分都是 1.8：
```
GET books/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "title": "elasticsearch"
        }
      },
      "boost": 1.8
    }
  }
}
```

### dis_max query
dis_max query 与 bool query 有一定联系也有一定区别，dis_max query 支持多并发查询，可返回与任意查询条件子句匹配的任何文档类型。与 bool 查询可以将所有匹配查询的分数相结合使用的方式不同，dis_max 查询只使用最佳匹配查询条件的分数。请看下面的例子：
```
GET books/_search
{
	"query": {
		"dis_max": {
			"tie_breaker": 0.7,
			"boost": 1.2,
			"queries": [{
					"term": {
						"age": 34
					}
				},
				{
					"term": {
						"age": 35
					}
				}
			]
		}
	}
}
```

### function_score query
function_score query 可以修改查询的文档得分，这个查询在有些情况下非常有用，比如通过评分函数计算文档得分代价较高，可以改用过滤器加自定义评分函数的方式来取代传统的评分方式。

使用 function_score query，用户需要定义一个查询和一至多个评分函数，评分函数会对查询到的每个文档分别计算得分。

下面这条查询语句会返回 books 索引中的所有文档，文档的最大得分为 5，每个文档的得分随机生成，权重的计算模式为相乘模式。
```
GET books/_search
{
  "query": {
    "function_score": {
      "query": {
        "match all": {}
      },
      "boost": "5",
      "random_score": {},
      "boost_mode": "multiply"
    }
  }
}
```

使用脚本自定义评分公式，这里把 price 值的十分之一开方作为每个文档的得分，查询语句如下：
```
GET books/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "title": "java"
        }
      },
      "script_score": {
        "inline": "Math.sqrt(doc['price'].value/10)"
      }
    }
  }
}
```

### indices query
indices query 适用于需要在多个索引之间进行查询的场景，它允许指定一个索引名字列表和内部查询。indices query 中有 query 和 no_match_query 两部分，query 中用于搜索指定索引列表中的文档，no_match_query 中的查询条件用于搜索指定索引列表之外的文档。下面的查询语句实现了搜索索引 books、books2 中 title 字段包含关键字 javascript，其他索引中 title 字段包含 basketball 的文档，查询语句如下：
```
GET books/_search
{
	"query": {
		"indices": {
			"indices": ["books", "books2"],
			"query": {
				"match": {
					"title": "javascript"
				}
			},
			"no_match_query": {
				"term": {
					"title": "basketball"
				}
			}
		}
	}
}
```

## 嵌套查询
在 Elasticsearch 这样的分布式系统中执行全 SQL 风格的连接查询代价昂贵，是不可行的。相应地，为了实现水平规模地扩展，Elasticsearch 提供了以下两种形式的 join：
- nested query（嵌套查询）
文档中可能包含嵌套类型的字段，这些字段用来索引一些数组对象，每个对象都可以作为一条独立的文档被查询出来。
- has_child query（有子查询）和 has_parent query（有父查询）
父子关系可以存在单个的索引的两个类型的文档之间。has_child 查询将返回其子文档能满足特定查询的父文档，而 has_parent 则返回其父文档能满足特定查询的子文档。

### nested query
文档中可能包含嵌套类型的字段，这些字段用来索引一些数组对象，每个对象都可以作为一条独立的文档被查询出来（用嵌套查询）。
```
PUT /my_index
{
	"mappings": {
		"type1": {
			"properties": {
				"obj1": {
					"type": "nested"
				}
			}
		}
	}
}
```

### has_child query
文档的父子关系创建索引时在映射中声明，这里以员工（employee）和工作城市（branch）为例，它们属于不同的类型，相当于数据库中的两张表，如果想把员工和他们工作的城市关联起来，需要告诉 Elasticsearch 文档之间的父子关系，这里 employee 是 child type，branch 是 parent type，在映射中声明，执行命令：
```
PUT /company
{
	"mappings": {
		"branch": {},
		"employee": {
			"parent": { "type": "branch" }
		}
	}
}
```

使用 bulk api 索引 branch 类型下的文档，命令如下：
```
POST company/branch/_bulk
{ "index": { "_id": "london" }}
{ "name": "London Westminster","city": "London","country": "UK" }
{ "index": { "_id": "liverpool" }}
{ "name": "Liverpool Central","city": "Liverpool","country": "UK" }
{ "index": { "_id": "paris" }}
{ "name": "Champs Elysees","city": "Paris","country": "France" }
```

添加员工数据：
```
POST company/employee/_bulk
{ "index": { "_id": 1,"parent":"london" }}
{ "name": "Alice Smith","dob": "1970-10-24","hobby": "hiking" }
{ "index": { "_id": 2,"parent":"london" }}
{ "name": "Mark Tomas","dob": "1982-05-16","hobby": "diving" }
{ "index": { "_id": 3,"parent":"liverpool" }}
{ "name": "Barry Smith","dob": "1979-04-01","hobby": "hiking" }
{ "index": { "_id": 4,"parent":"paris" }}
{ "name": "Adrien Grand","dob": "1987-05-11","hobby": "horses" }
```

通过子文档查询父文档要使用 has_child 查询。例如，搜索 1980 年以后出生的员工所在的分支机构，employee 中 1980 年以后出生的有 Mark Thomas 和 Adrien Grand，他们分别在 london 和 paris，执行以下查询命令进行验证：
```
GET company/branch/_search
{
	"query": {
		"has_child": {
			"type": "employee",
			"query": {
				"range": { "dob": { "gte": "1980-01-01" } }
			}
		}
	}
}
```

搜索哪些机构中有名为 “Alice Smith” 的员工，因为使用 match 查询，会解析为 “Alice” 和 “Smith”，所以 Alice Smith 和 Barry Smith 所在的机构会被匹配，执行以下查询命令进行验证：
```
GET company/branch/_search
{
	"query": {
		"has_child": {
			"type": "employee",
			"score_mode": "max",
			"query": {
				"match": { "name": "Alice Smith" }
			}
		}
	}
}
```

可以使用 min_children 指定子文档的最小个数。例如，搜索最少含有两个 employee 的机构，查询命令如下：
```
GET company/branch/_search?pretty
{
	"query": {
		"has_child": {
			"type": "employee",
			"min_children": 2,
			"query": {
				"match_all": {}
			}
		}
	}
}
```

### has_parent query
通过父文档查询子文档使用 has_parent 查询。比如，搜索哪些 employee 工作在 UK，查询命令如下：
```
GET company/employee/_search
{
	"query": {
		"has_parent": {
			"parent_type": "branch",
			"query": {
				"match": { "country": "UK }
			}
		}
	}
}
```

### 位置查询
Elasticsearch 可以对地理位置点 geo_point 类型和地理位置形状 geo_shape 类型的数据进行搜索。为了学习方便，这里准备一些城市的地理坐标作为测试数据，每一条文档都包含城市名称和地理坐标这两个字段，这里的坐标点取的是各个城市中心的一个位置。首先把下面的内容保存到 geo.json 文件中：
```
{"index":{ "_index":"geo","_type":"city","_id":"1" }}
{"name":"北京","location":"39.9088145109,116.3973999023"}
{"index":{ "_index":"geo","_type":"city","_id": "2" }}
{"name":"乌鲁木齐","location":"43.8266300000,87.6168800000"}
{"index":{ "_index":"geo","_type":"city","_id":"3" }}
{"name":"西安","location":"34.3412700000,108.9398400000"}
{"index":{ "_index":"geo","_type":"city","_id":"4" }}
{"name":"郑州","location":"34.7447157466,113.6587142944"}
{"index":{ "_index":"geo","_type":"city","_id":"5" }}
{"name":"杭州","location":"30.2294080260,120.1492309570"}
{"index":{ "_index":"geo","_type":"city","_id":"6" }}
{"name":"济南","location":"36.6518400000,117.1200900000"}
```

创建一个索引并设置映射：
```
PUT geo
{
	"mappings": {
		"city": {
			"properties": {
				"name": {
					"type": "keyword"
				},
				"location": {
					"type": "geo_point"
				}
			}
		}
	}
}
```

然后执行批量导入命令：
```
curl -XPOST "http://localhost:9200/_bulk?pretty" --data-binary @geo.json
```

### geo_distance query
geo_distance query 可以查找在一个中心点指定范围内的地理点文档。例如，查找距离天津 200km 以内的城市，搜索结果中会返回北京，命令如下：
```
GET geo/_search
{
	"query": {
		"bool": {
			"must": {
				"match_all": {}
			},
			"filter": {
				"geo_distance": {
					"distance": "200km",
					"location": {
						"lat": 39.0851000000,
						"lon": 117.1993700000
					}
				}
			}
		}
	}
}
```

按各城市离北京的距离排序：
```
GET geo/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [{
    "_geo_distance": {
      "location": "39.9088145109,116.3973999023",
      "unit": "km",
      "order": "asc",
      "distance_type": "plane"
    }
  }]
}
```
其中 location 对应的经纬度字段；unit 为 km 表示将距离以 km 为单位写入到每个返回结果的 sort 键中；distance_type 为 plane 表示使用快速但精度略差的 plane 计算方式。

### geo_bounding_box query
geo_bounding_box query 用于查找落入指定的矩形内的地理坐标。查询中由两个点确定一个矩形，然后在矩形区域内查询匹配的文档。
```
GET geo/_search
{
	"query": {
		"bool": {
			"must": {
				"match_all": {}
			},
			"filter": {
				"geo_bounding_box": {
					"location": {
						"top_left": {
							"lat": 38.4864400000,
							"lon": 106.2324800000
						},
						"bottom_right": {
							"lat": 28.6820200000,
							"lon": 115.8579400000
						}
					}
				}
			}
		}
	}
}
```

### geo_polygon query
geo_polygon query 用于查找在指定多边形内的地理点。例如，呼和浩特、重庆、上海三地组成一个三角形，查询位置在该三角形区域内的城市，命令如下：
```
GET geo/_search
{
	"query": {
		"bool": {
			"must": {
				"match_all": {}
			}
		},
		"filter": {
			"geo_polygon": {
				"location": {
					"points": [{
						"lat": 40.8414900000,
						"lon": 111.7519900000
					}, {
						"lat": 29.5647100000,
						"lon": 106.5507300000
					}, {
						"lat": 31.2303700000,
						"lon": 121.4737000000
					}]
				}
			}
		}
	}
}
```

### geo_shape query
geo_shape query 用于查询 geo_shape 类型的地理数据，地理形状之间的关系有相交、包含、不相交三种。创建一个新的索引用于测试，其中 location 字段的类型设为 geo_shape 类型。
```
PUT geoshape
{
	"mappings": {
		"city": {
			"properties": {
				"name": {
					"type": "keyword"
				},
				"location": {
					"type": "geo_shape"
				}
			}
		}
	}
}
```

关于经纬度的顺序这里做一个说明，geo_point 类型的字段纬度在前经度在后，但是对于 geo_shape 类型中的点，是经度在前纬度在后，这一点需要特别注意。

把西安和郑州连成的线写入索引：
```
POST geoshape/city/1
{
	"name": "西安-郑州",
	"location": {
		"type": "linestring",
		"coordinates": [
			[108.9398400000, 34.3412700000],
			[113.6587142944, 34.7447157466]
		]
	}
}
```

查询包含在由银川和南昌作为对角线上的点组成的矩形的地理形状，由于西安和郑州组成的直线落在该矩形区域内，因此可以被查询到。命令如下：
```
GET geoshape/_search
{
	"query": {
		"bool": {
			"must": {
				"match_all": {}
			},
			"filter": {
				"geo_shape": {
					"location": {
						"shape": {
							"type": "envelope",
							"coordinates": [
								[106.23248, 38.48644],
								[115.85794, 28.68202]
							]
						},
						"relation": "within"
					}
				}
			}
		}
	}
}
```

## 特殊查询
### more_like_this query
more_like_this query 可以查询和提供文本类似的文档，通常用于近似文本的推荐等场景。查询命令如下：
```
GET books/_search
{
	"query": {
		"more_like_ this": {
			"fields": ["title", "description"],
			"like": "java virtual machine",
			"min_term_freq": 1,
			"max_query_terms": 12
		}
	}
}
```
可选的参数及取值说明如下：
- fields 要匹配的字段，默认是 _all 字段。
- like 要匹配的文本。
- min_term_freq 文档中词项的最低频率，默认是 2，低于此频率的文档会被忽略。
- max_query_terms query 中能包含的最大词项数目，默认为 25。
- min_doc_freq 最小的文档频率，默认为 5。
- max_doc_freq 最大文档频率。
- min_word length 单词的最小长度。
- max_word length 单词的最大长度。
- stop_words 停用词列表。
- analyzer 分词器。
- minimum_should_match 文档应匹配的最小词项数，默认为 query 分词后词项数的 30%。
- boost terms 词项的权重。
- include 是否把输入文档作为结果返回。
- boost 整个 query 的权重，默认为 1.0。

### script query
Elasticsearch 支持使用脚本进行查询。例如，查询价格大于 180 的文档，命令如下：
```
GET books/_search
{
  "query": {
    "script": {
      "script": {
        "inline": "doc['price'].value > 180",
        "lang": "painless"
      }
    }
  }
}
```

### percolate query
一般情况下，我们是先把文档写入到 Elasticsearch 中，通过查询语句对文档进行搜索。percolate query 则是反其道而行之的做法，它会先注册查询条件，根据文档来查询 query。例如，在 my-index 索引中有一个 laptop 类型，文档有 price 和 name 两个字段，在映射中声明一个 percolator 类型的 query，命令如下：
```
PUT my-index
{
	"mappings": {
		"laptop": {
			"properties": {
				"price": { "type": "long" },
				"name": { "type": "text" }
			},
			"queries": {
				"properties": {
					"query": { "type": "percolator" }
				}
			}
		}
	}
}
```

注册一个 bool query，bool query 中包含一个 range query，要求 price 字段的取值小于等于 10000，并且 name 字段中含有关键词 macbook：
```
PUT /my-index/queries/1?refresh
{
	"query": {
		"bool": {
			"must": [{
				"range": { "price": { "lte": 10000 } }
			}, {
				"match": { "name": "macbook" }
			}]
		}
	}
}
```

通过文档查询 query：
```
GET /my-index/_search
{
	"query": {
		"percolate": {
			"field": "query",
			"document_type": "laptop",
			"document": {
				"price": 9999,
				"name": "macbook pro on sale"
			}
		}
	}
}
```
文档符合 query 中的条件，返回结果中可以查到上文中注册的 bool query。percolate query 的这种特性适用于数据分类、数据路由、事件监控和预警等场景。