---
layout: post
categories: ElasticSearch
description: none
keywords: ElasticSearch
---
# ElasticSearch相关性解读
ElasticSearch 5.0版本前相关性判断及打分使用的算法是 TF-IDF ，5.0 版本以后使用的是 BM25 算法。

## BM25 算法
TF-IDF : Term Frequency, Inverse Document Frequency 即词频和逆文档频率，TF= 词项在文档出现次数/该文档总字数，IDF= log(索引文档总数量/词项出现的文档数量)，简单来说，TF-IDF得分计算公式为：

文档得分 = TF(t) * IDF(t) * boost(t) * norm(t)

t 即查询串中每个词项，boost函数是ES提供的分数调整函数，norm函数根据文档长短返回一个值（文档短对应的值大）。

BM25 : 整体而言就是对 TF-IDF 算法的改进，对于 TF-IDF 算法，TF(t) 部分的值越大，整个公式返回的值就会越大，BM25 针对针对这点进行来优化，随着TF(t) 的逐步加大，该算法的返回值会趋于一个数值。

## es的查询
在es的查询中，有两个指标非常重要：
- 一是准确率，查询到的结果集中包含的正确结果数占比；
- 二是召回率，就是查到的结果集中正确结果在所有正确结果(包含查询到的和未查询到的)中的占比。
在单字符串多字段查询过程中，考虑到正确率，就是要把匹配度最高的放在最前面；考虑到召回率就是就可能多的把相关文档都查出来。

在es中，multi_match就是针对单字符串多字段查询的解决方案，包括三种查询：best_fields,most_fields,cross_fields。

## best_fields
多字段查询中，单字段匹配度最高的那个文档的算分最高。

插入测试数据:
```
PUT multi_query_index/_bulk
{"index":{"_id":1}}
{"title":"my bark dogs","body":"cats and dogs"}
{"index":{"_id":2}}
{"title":"my cat mimi","body":"of barking dogs"}
```
先用bool查询来一波
按照常规理解，第二个文档的body属性中有barking dogs，应该匹配度最高。但结果却是文档1算分最高，排在最前面。

但是考虑到以下几个原因：
- 一是分词会将barking和dogs还原为bark和dog，这样算起这两个词在文档1一共出现了三次，在文档2中出现了两次，文档1的匹配度最高。
- 二是，在文本搜索中barking dogs并非整体作为一个关键字去搜索，而是被分词后进行搜索。
- 三是，bool查询会对每一个match query进行加权平均算分。
```
GET multi_query_index/_search
{
  "query": {
     "bool": {
       "should": [
         {
           "match": {
             "title":  "barking dogs"
           }
         },
         {
           "match": {
             "body":  "barking dogs"
           }
         }
       ]
     }
  }
}
```
使用multi_query的best_fields来一波
multi_query默认是best_fields模式，即找出匹配度最高的字段的算分作为整个查询的最高算分。这样的话，第二个文档算分就最高了。
```
GET multi_query_index/_search
{
  "explain": true, 
  "query": {
    "multi_match": {
      "type": "best_fields", 
      "query": "barking dogs",
      "fields": ["title","body"]
    }
  }
}
```
当然，在这个查询中，我们可以指定字段权重，突出某个字段的评分
```
GET multi_query_index/_search
{
  "explain": true, 
  "query": {
    "multi_match": {
      "type": "best_fields", 
      "query": "barking dogs",
      "fields": ["title^10","body"]
    }
  }
}
```
本质上，multi_match/best_field查询被转换为一个disjunction query。

### most_fields
most fields字段主要是利用multifields特性，在索引的mapping中对字段设置多个字段，该字段的分词器采用标准分词器，这样的话，分词只会根据空格拆分，而不会对单词进行主干提取。
```
PUT /multi_query_most_fields
{
    "settings": { "number_of_shards": 1 }, 
    "mappings": {
        "my_type": {
            "properties": {
                "title": { 
                    "type":     "string",
                    "analyzer": "english",
                    "fields": {
                        "std":   { 
                            "type":     "string",
                            "analyzer": "standard"
                        }
                    }
                }
            }
        }
    }
}
```
插入2条数据
```
PUT /multi_query_most_fields/_doc/1
{ "title": "My rabbit jumps" }

PUT /multi_query_most_fields/_doc/2
{ "title": "Jumping jack rabbits" }
```
针对title单字段来一波multi query查询
这波查询，两个文档的算分一样，但显然第二个文档的算分应该高一些，这就是因为title使用的english分词器会将jumping还原为jump，rabbits还原为rabbit。
```
GET multi_query_most_fields/_search
{
  "query": {
    "multi_match": {
      "query": "jumping rabbits",
      "fields": ["title"],
      "type": "most_fields"
    }
  }
}
```
针对title及其multifields来一波multi query查询
这波查询，多了对title.std的查询，结果就是第二个文档的算分高，排在最前面，这是因为title.std使用了standard分词器。
```
GET multi_query_most_fields/_search
{
  "query": {
    "multi_match": {
      "query": "jumping rabbits",
      "fields": ["title","title.std"],
      "type": "most_fields"
    }
  }
}
```
这里，再尝试一次bool查询。
结果发现，bool查询的结果和multi_query/most_fields查询一致。

其实，multi_query/most_fields原理上就是被封装为一个bool查询的。
```
GET multi_query_most_fields/_search
{
  "query": {
    "dis_max": {
      "boost": 1.2,
      "queries": [
        {
          "match": {
            "title": "jumping rabbits"
          }
        },
        {
         "match": {
            "title.std": "jumping rabbits"
          }
        }
      ]
    }
  }
}
```

## ES自定义评分机制
在ES的常规查询中，只有参与了匹配查询的字段才会参与记录的相关性得分score的计算。但很多时候我们希望能根据搜索记录的热度、浏览量、评分高低等来计算相关性得分，提高用户体验。

### 实战演示
说明：创建博客blog索引，只有2个字段，博客名title和访问量access_num。
用户根据博客名称搜索的时候，既希望名称能尽可能匹配，也希望访问量越多的排在最前面，因为一般访问量越多的博客质量会越好，这样可以提高用户的检索体验。
```
DELETE /blog

PUT /blog
{  
  "mappings": {
     "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },
      "access_num": {
        "type": "integer"
      }
    }
  }
}
```
添加测试数据
```
PUT blog/_doc/2
{
  "title": "java入门到精通",
  "access_num":30
}

PUT blog/_doc/3
{
  "title": "es入门到精通",
  "access_num":50
}

PUT blog/_doc/4
{
  "title": "mysql入门到精通",
  "access_num":30
}

PUT blog/_doc/5
{
  "title": "精通spark",
  "access_num":40
}
```

直接使用match查询，只会根据检索关键字和title字段值的相关性检索排序。
```
GET /blog/_search
{
    "query": {
       "match": {
                    "title": "java入门"
                }
    }
}
```
查询结果L
```
 "hits" : [
      {
        "_index" : "blog",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.3739232,
        "_source" : {
          "title" : "java入门",
          "access_num" : 20
        }
      },
      {
        "_index" : "blog",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0552295,
        "_source" : {
          "title" : "java入门到精通",
          "access_num" : 30
        }
      },
      ……
     ] 
```

## 采用function_score自定义评分
除了match匹配查询计算相关性得分，还引入了根据浏览量access_num计算得分。
```
GET /blog/_search
{
    "query": {
        "function_score": {
            "query": {
                "match": {
                    "title": "java入门"
                }
            },
            "functions": [
                {
                    "script_score": {
                        "script": {
                            "params": {
                                "access_num_ratio": 2.5
                            },
                            "lang": "painless",
                            "source": "doc['access_num'].value * params.access_num_ratio "
                        }
                    }
                }
            ]
        }
    }
}
```
查询结果：
说明：尽管博客名为java入门的名称和搜索词更加匹配，但由于博客名为java入门到精通的博客访问量更高，最终检索品分更高，排名更靠前。
```
 "hits" : [
      {
        "_index" : "blog",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 79.14222,
        "_source" : {
          "title" : "java入门到精通",
          "access_num" : 30
        }
      },
      {
        "_index" : "blog",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 68.69616,
        "_source" : {
          "title" : "java入门",
          "access_num" : 20
        }
      },
```

## 自定义评分类型
function_score 查询提供了多种类型的评分函数。
- script_score script脚本评分
- weight 字段权重评分
- random_score 随机评分
- field_value_factor 字段值因子评分
- decay functions: gauss, linear, exp 衰减函数

### script脚本评分
script_score 函数允许您包装另一个查询并选择性地使用脚本表达式从文档中的其他数字字段值派生的计算自定义它的评分。 这是一个简单的示例：
```
GET /_search
{
    "query": {
        "function_score": {
            "query": {
                "match": { "message": "elasticsearch" }
            },
            "script_score" : {
                "script" : {
                  "source": "Math.log(2 + doc['likes'].value)"
                }
            }
        }
    }
}
```
请注意，与 custom_score 查询不同，查询的分数乘以脚本评分的结果。 如果你想禁止这个，设置 “boost_mode”: “replace”。

### weight 权重评分
weight函数是最简单的分支，它将得分乘以一个常数。请注意，普通的boost字段按照标准化来增加分数。
而weight函数却真真切切地将得分乘以确定的数值。

下面的例子意味着，在description字段中匹配了hadoop词条查询的文档，他们的分数将被乘以1.5.
```
GET /_search
{
    "query": {
        "function_score": {
            "query": {
                "match": { "message": "elasticsearch" }
            },
            "functions":[
	             {
		            "weight":1.5  ,
		            "filter": { "term": { "description": "hadoop" }}   
	             },
	             {
		            "weight":3  ,
		            "filter": { "term": { "description": "flink" }}   
	             }
            ]
        }
    }
}
```

### random_score随机评分
random_score 生成从 0 到但不包括 1 的均匀分布的分数。默认情况下，它使用内部 Lucene doc id 作为随机源。

如果您希望分数可重现，可以提供种子和字段。 然后将基于此种子、所考虑文档的字段最小值以及基于索引名称和分片 id 计算的盐计算最终分数，以便具有相同值但存储在不同索引中的文档得到 不同的分数。

请注意，位于同一个分片内且具有相同字段值的文档将获得相同的分数，因此通常希望使用对所有文档具有唯一值的字段。 一个好的默认选择可能是使用 _seq_no 字段，其唯一的缺点是如果文档更新，分数会改变，因为更新操作也会更新 _seq_no 字段的值。
```
GET /_search
{
    "query": {
        "function_score": {
            "random_score": {
                "seed": 10,
                "field": "_seq_no"
            }
        }
    }
}
```

### field_value_factor 字段值因子评分
field_value_factor 函数允许您使用文档中的字段来影响分数。 它类似于使用 script_score 函数，但是，它避免了脚本的开销。 如果用于多值字段，则在计算中仅使用该字段的第一个值。

举个例子，假设你有一个用数字 likes 字段索引的文档，并希望用这个字段影响文档的分数，一个这样做的例子看起来像：
```
GET /_search
{
    "query": {
        "function_score": {
            "field_value_factor": {
                "field": "likes",
                "factor": 1.2,
                "modifier": "sqrt",
                "missing": 1
            }
        }
    }
}
```
得分计算公式： sqrt(1.2 * doc['likes'].value)

参数说明：
- field 要从文档中提取的字段。
- factor 与字段值相乘的可选因子，默认为 1。
- modifier 应用于字段值的计算修饰符， none, log, log1p, log2p, ln, ln1p, ln2p, square, sqrt, or reciprocal，默认 none.
- missing 如果文档没有该字段，则使用的值。 修饰符和因子仍然适用于它，就好像它是从文档中读取的一样。

### Decay functions 衰减函数
衰减函数使用一个函数对文档进行评分，该函数根据文档的数字字段值与用户给定原点的距离而衰减。 这类似于范围查询，但具有平滑的边缘而不是框。

要对具有数字字段的查询使用距离评分，用户必须为每个字段定义原点和比例。 需要原点来定义计算距离的“中心点”，以及定义衰减率的比例尺。
```
GET /_search
{
    "query": {
        "function_score": {
          "functions": [
            {
              "gauss": {
                "price": {
                  "origin": "0",
                  "scale": "20"
                }
              }
            },
            {
              "gauss": {
                "location": {
                  "origin": "11, 12",
                  "scale": "2km"
                }
              }
            }
          ],
          "query": {
            "match": {
              "properties": "balcony"
            }
          },
          "score_mode": "multiply"
        }
    }
}
```

## 合并得分
```
GET /_search
{
    "query": {
        "function_score": {
          "query": { "match_all": {} },
          "boost": "5", 
          "functions": [
              {
                  "filter": { "match": { "test": "bar" } },
                  "random_score": {}, 
                  "weight": 23
              },
              {
                  "filter": { "match": { "test": "cat" } },
                  "weight": 42
              }
          ],
          "max_boost": 42,
          "score_mode": "max",
          "boost_mode": "multiply",
          "min_score" : 42
        }
    }
}
```
参数说明：
- max_boost  可以通过设置 max_boost 参数将新分数限制为不超过某个限制。 max_boost 的默认值是 FLT_MAX。
- min_score 默认情况下，修改分数不会更改匹配的文档。 要排除不满足某个分数阈值的文档，可以将 min_score 参数设置为所需的分数阈值。

参数 score_mode 指定如何组合计算的分数：
- multiply 相乘 (default)
- sum 求和
- avg 平均分
- first 使用具有匹配过滤器的第一个函数的得分
- max 使用最高分
- min 使用最低分

boost_mode定义新计算的分数与查询的分数相结合。 具体选项：
- multiply 查询得分和函数得分相乘，默认
- replace 仅使用函数得分，查询得分被忽略
- sum 查询得分和函数得分求和
- avg 查询得分和函数得分取平均值
- max 取查询得分和函数得分的最大值
- min 取查询得分和函数得分的最小值

## Script Score
现在让我们进入灵活度最高的排序打分世界！ES提供的Script Score查询可以以编写脚本的方式对文档进行灵活打分，以实现自定义干预结果排名的目的。Script Score默认的脚本语言为Painless，在Painless中可以访问文档字段，也可以使用ES内置的函数，甚至可以通过给脚本传递参数这种方式联通内部和外部数据。


## 查询时设置权重
在默认情况下，这些查询的权重都为1，也就是查询之间都是平等的。有时我们希望某些查询的权重高一些，也就是在其他条件相同的情况下，匹配该查询的文档得分更高。此时应该怎么做呢？本节将介绍的boosting查询和boost设置可以满足上述查询需求。

## 查询时boost参数的设置
在ES中可以通过查询的boost值对某个查询设定其权重。在默认情况下，所有查询的boost值为1。但是当设置某个查询的boost为2时，不代表匹配该查询的文档评分是原来的2倍，而是代表匹配该查询的文档得分相对于其他文档得分被提升了。例如，可以为查询A设定boost值为3，为查询B设定boost值为6，则在进行相关性计算时，查询B的权重将比查询A相对更高一些。

更近一步说，设置boost参数为某个值后并不是将查询命中的文档分数乘以该值，而是将BM25中的boost参数乘以该数值。

## boosting查询
虽然使用boost值可以对查询的权重进行调整，但是仅限于term查询和类match查询。 有时需要调整更多类型的查询，如搜索酒店时，需要将房价低于200的酒店权重降低，此时可能需要用到range查询，但是range查询不能使用boost参数，这时可以使用ES的boosting查询进行封装。

ES的boosting查询分为两部分，一部分是positive查询，代表正向查询，另一部分是negative查询，代表负向查询。 可以通过negative_boost参数设置负向查询的权重系数，该值的范围为0～1。最终的文档得分为：正向匹配值+负向匹配值×negative_boost。






