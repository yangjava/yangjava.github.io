---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch索引详解
Elasticsearch 索引管理主要包括如何进行索引的创建、索引的删除、副本的更新、索引读写权限、索引别名的配置等等内容。

## 索引删除
es 索引删除操作向 es 集群的 http 接口发送指定索引的 delete http 请求即可，可以通过 curl 命令，具体如下：
```
curl -X DELETE http://{es_host}:{es_http_port}/{index}
```
具体示例如下：
```
curl -X DELETE http://10.10.10.66:9200/my_index?pretty
```
如果删除成功，它会返回如下信息，为了返回的信息便于读取，增加了 pretty 参数：
```
{
"acknowledged" : true
}
```

## 索引别名
es 的索引别名就是给一个索引或者多个索引起的另一个名字，典型的应用场景是针对索引使用的平滑切换。

首先，创建索引 my_index，然后将别名 my_alias 指向它，示例如下：
```
PUT /my_index
PUT /my_index/_alias/my_alias
```
也可以通过如下形式：
```
POST /_aliases
{    
  "actions": [
    { "add": { "index": "my_index", "alias": "my_alias" }}
  ]
}
```
也可以在一次请求中增加别名和移除别名混合使用：
```
POST /_aliases
{    
  "actions": [
    { "remove": { "index": "my_index", "alias": "my_alias" }}
    { "add": { "index": "my_index_v2", "alias": "my_alias" }}
  ]
}
```
需要注意的是，如果别名与索引是一对一的，使用别名索引文档或者查询文档是可以的，但是如果别名和索引是一对多的，使用别名会发生错误，因为 es 不知道把文档写入哪个索引中去或者从哪个索引中读取文档。

## 索引设置
Elasticsearch 索引的配置项主要分为静态配置属性和动态配置属性，静态配置属性是索引创建后不能修改，而动态配置属性则可以随时修改。

## 索引设置
es 索引设置的 api 为 _settings，完整的示例如下：
```
PUT /my_index
{
  "settings": {
    "index": {
      "number_of_shards": "1",
      "number_of_replicas": "1",
      "refresh_interval": "60s",
      "analysis": {
        "filter": {
          "tsconvert": {
            "type": "stconvert",
            "convert_type": "t2s",
            "delimiter": ","
          },
          "synonym": {
            "type": "synonym",
            "synonyms_path": "analysis/synonyms.txt"
          }
        },
        "analyzer": {
          "ik_max_word_synonym": {
            "filter": [
              "synonym",
              "tsconvert",
              "standard",
              "lowercase",
              "stop"
            ],
            "tokenizer": "ik_max_word"
          },
          "ik_smart_synonym": {
            "filter": [
              "synonym",
              "standard",
              "lowercase",
              "stop"
            ],
            "tokenizer": "ik_smart"
          }
        },
			"mapping": {
				"coerce": "false",
				"ignore_malformed": "false"
			},
			"indexing": {
				"slowlog": {
					"threshold": {
						"index": {
							"warn": "2s",
							"info": "1s"
						}
					}
				}
			},
			"provided_name": "hospital_202101070533",
			"query": {
				"default_field": "timestamp",
				"parse": {
					"allow_unmapped_fields": "false"
				}
			},
			"requests": {
				"cache": {
					"enable": "true"
				}
			},
			"search": {
				"slowlog": {
					"threshold": {
						"fetch": {
							"warn": "1s",
							"info": "200ms"
						},
						"query": {
							"warn": "1s",
							"info": "500ms"
						}
					}
				}
			}
		}
	}
}
```
### 固定属性
- index.creation_date：顾名思义索引的创建时间戳。
- index.uuid：索引的 uuid 信息。
- index.version.created：索引的版本号。

### 索引静态配置
- index.number_of_shards：索引的主分片数，默认值是 5。这个配置在索引创建后不能修改；在 es 层面，可以通过 es.index.max_number_of_shards 属性设置索引最大的分片数，默认为 1024。
- index.codec：数据存储的压缩算法，默认值为 LZ4，可选择值还有 best_compression，它比 LZ4 可以获得更好的压缩比（即占据较小的磁盘空间，但存储性能比 LZ4 低）。
- index.routing_partition_size：路由分区数，如果设置了该参数，其路由算法为：( hash(_routing) + hash(_id) % index.routing_parttion_size ) % number_of_shards。如果该值不设置，则路由算法为 hash(_routing) % number_of_shardings，_routing 默认值为 _id。

静态配置里，有重要的部分是配置分析器（config analyzers）。
- index.analysis：分析器最外层的配置项，内部主要分为 char_filter、tokenizer、filter 和 analyzer。
  - char_filter：定义新的字符过滤器件。
  - tokenizer：定义新的分词器。
  - filter：定义新的 token filter，如同义词 filter。
  - analyzer：配置新的分析器，一般是char_filter、tokenizer 和一些 token filter 的组合。

### 索引动态配置
index.number_of_replicas：索引主分片的副本数，默认值是 1，该值必须大于等于 0，这个配置可以随时修改。
index.refresh_interval：执行新索引数据的刷新操作频率，该操作使对索引的最新更改对搜索可见，默认为 1s。也可以设置为 -1 以禁用刷新。更详细信息参考 Elasticsearch 动态修改 refresh_interval 刷新间隔设置。

## 索引映射类型及mapping属性详解
在 Elasticsearch 中，映射指的是 mapping，用来定义一个文档以及其所包含的字段如何被存储和索引，可以在映射中事先定义字段的数据类型、字段的权重、分词器等属性，就如同在关系型数据库中创建数据表时会设置字段的类型。

Elasticsearch 中一个字段可以是核心类型之一，如字符串或者数值型，也可以是一个从核心类型派生的复杂类型，如数值。

### 映射分类
在 Elasticsearch 中，映射可分为动态映射和静态映射。在关系型数据库中写入数据之前首先要建表，在建表语句中声明字段的属性，在 Elasticsearch 中，则不必如此，Elasticsearch 最重要的功能之一就是让你尽可能快地开始探索数据，文档写入 Elasticsearch 中，它会根据字段的类型自动识别，这种机制称为动态映射，而静态映射则是写入数据之前对字段的属性进行手工设置。

### 静态映射
静态映射是在创建索引时手工指定索引映射，和 SQL 中在建表语句中指定字段属性类似。相比动态映射，通过静态映射可以添加更详细、更精准的配置信息，例子如下：
```
PUT my_index
{
  "mappings": {
    "article": {
      "_all": {
        "enabled": false
      },
      "properties": {
        "title": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_smart",
          "similarity": "BM25",
          "store": true
        },
        "summary": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_smart",
          "similarity": "BM25"
        },
        "content": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_smart",
          "similarity": "BM25",
          "store": true
        }
      }
    },
    "question_and_anwser": {
      "_all": {
        "enabled": false
      },
      "properties": {
        "question": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_smart",
          "similarity": "BM25"
        },
        "anwser": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_smart",
          "similarity": "BM25"
        },
        "question_user_id": {
          "type": "long"
        },
        "anwser_user_id": {
          "type": "long"
        },
        "create_time": {
          "type": "date",
          "format": "strict_date_optional_time || epoch_mills"
        }
      }
    }
  }
}
```

### 动态映射
动态映射是一种偷懒的方式，可直接创建索引并写入文档，文档中字段的类型是 Elasticsearch 自动识别的，不需要在创建索引的时候设置字段的类型。在实际项目中，如果遇到的业务在导入数据之前不确定有哪些字段，也不清楚字段的类型是什么，使用动态映射非常合适。当 Elasticsearch 在文档中碰到一个以前没见过的字段时，它会利用动态映射来决定该字段的类型，并自动把该字段添加到映射中，根据字段的取值自动推测字段类型的规则见下表：
- JSON 格式的数据	自动推测的字段类型
- null	没有字段被添加
- true or false	boolean 类型
- 浮点类型数字	float 类型
- 数字	long 类型
- JSON 对象	object 类型
- 数组	由数组中第一个非空值决定
- string	有可能是 date 类型（若开启日期检测）、double 或 long 类型、text 类型、keyword 类型

下面举一个例子认识动态 mapping，在 Elasticsearch 中创建一个新的索引并查看它的 mapping，命令如下：
```
PUT books
GET books/_mapping
```
此时 books 索引的 mapping 是空的，返回结果如下：
```
{
  "books": {
    "mappings": {}
  }
}
```
再往 books 索引中写入一条文档，命令如下：
```
PUT books/it/1
{
  "id": 1,
  "publish_date": "2019-11-10",
  "name": "master Elasticsearch"
}
```
文档写入完成之后，再次查看 mapping，返回结果如下：
```
{
  "books": {
    "mappings": {
      "it": {
        "properties": {
          "id": {
            "type": "long"
          },
          "name": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          },
          "publish_date": {
            "type": "date"
          }
        }
      }
    }
  }
}
```
id、publish_date、name 三个字段分别被推测为 long 类型、date 类型和 text 类型，这就是动态 mapping 的功劳。

使用动态 mapping 要结合实际业务需求来综合考虑，如果将 Elasticsearch 当作主要的数据存储使用，并且希望出现未知字段时抛出异常来提醒你注意这一问题，那么开启动态 mapping 并不适用。在 mapping 中可以通过 dynamic 设置来控制是否自动新增字段，接受以下参数：
- true：默认值为 true，自动添加字段。
- false：忽略新的字段。
- strict：严格模式，发现新的字段抛出异常。
下面通过例子和实际操作来学习 dynamic 控制新增字段的方法。创建一个 books 索引并指定 mapping，设置 it 类型下 dynamic 属性的取值为 strict，也就是说，it 类型下的文档中出现 mapping 中没有定义的字段会抛出异常，命令如下：
```
PUT books
{
  "mappings": {
    "it": {
      "dynamic": "strict",
      "properties": {
        "title": {
          "type": "text"
        },
        "publish_date": {
          "type": "date"
        }
      }
    }
  }
}
```
写入三个文档：
```
PUT books/it/1
{
  "title": "master Elasticsearch",
  "publish_date": "2017-06-01"
}
PUT books/it/2
{
  "title": "master Elasticsearch"
}
PUT books/it/3
{
  "title": "master Elasticsearch",
  "publish_date": "2017-06-01",
  "author":"Tom"
}
```
文档 books/it/1 和 books/it/2 会创建成功，books/it/3 中出现了新的字段 author，该字段在 mapping 中并没有定义，会抛出 strict_dynamic_mapping_exception 异常。

## 核心类型
Elasticsearch 字段类型的核心类型有字符串类型、数字类型、日期类型、布尔类型、二进制类型、范围类型等。
- 字符串类型	string、text、keyword
- 数字类型	long、integer、short、byte、double、float、half_float、scaled_float
- 日期类型	date
- 布尔类型	boolean
- 二进制类型	binary
- 范围类型	range

### 字符串类型
字符串是最直接的，如果在索引字符，字段就应该是 text 类型。它们也是最有趣，因为在映射中有很多选项来分析它们。解析文本、转变文本、将其分解为基本元素使得搜索更为相关，这个过程叫作分析。

Elasticsearch 5.X 之后的字段类型不再支持 string，由 text 或 keyword 取代。如果仍使用 string，会给出警告。

- text
如果一个字段是要被全文搜索的，比如邮件内容、产品描述、新闻内容，应该使用 text 类型。设置 text 类型以后，字段内容会被分析，在生成倒排索引以前，字符串会被分词器分成一个一个词项（term）。text 类型的字段不用于排序，且很少用于聚合（Terms Aggregation 除外）。

- keyword
keyword 类型适用于索引结构化的字段，比如 email 地址、主机名、状态码和标签，通常用于过滤（比如，查找已发布博客中 status 属性为 published 的文章）、排序、聚合。类型为 keyword 的字段只能通过精确值搜索到，区别于 text 类型。

如上所提到的，text 类型字段可以在映射中指定相关的分析选项，通过 index 选项指定。
index 选项可以设置为 analyzed（默认）、not_analyzed 或 no。
- analyzed
默认情况下，index 被设置为 analyzed，并产生了如下行为：分析器将所有字符转为小写，并将字符串分解为单词。当期望每个单词完整匹配时，请使用这种选项。举个例子，如果用户搜索 “elasticsearch”，他们希望在结果列表里看到 “Principles and Practice of Elasticsearch”。
- not_analyzed
将 index 设置为 not_analyzed，将会产生相反的行为：分析过程被略过，整个字符串被当作单独的词条进行索引。当进行精准的匹配时，请使用这个选项，如搜索标签。你可能希望 “big data” 出现在搜索 “big data” 的结果中，而不是出现在搜索 “data” 的结果中。同样，对于多数的词条计数聚集，也需要这个。如果想知道最常出现的标签，可能需要 “big data” 作为一整个词条统计，而不是 “big” 和 “data” 分开统计。
- no
如果将 index 设置为 no，索引就被略过了，也没有词条产生，因此无法在那个字段上进行搜索。当无须在这个字段上搜索时，这个选项节省了存储空间，也缩短了索引和搜索的时间。例如，可以存储活动的评论。尽管存储和展示这些评论是很有价值的，但是可能并不需要搜索它们。在这种情况下，关闭那个字段的索引，使得索引的过程更快，并节省了存储空间。

### 数字类型
数字类型支持 byte、short、integer、long、float、double、half_float 和 scaled_float。

对于 float、half_float 和 scaled_float，-0.0 和 +0.0 是不同的值，使用 term 查询查找 -0.0 不会匹配 +0.0，同样 range 查询中上边界是 -0.0，也不会匹配 +0.0，下边界是 +0.0，也不会匹配 -0.0。

对于数字类型的字段，在满足需求的情况下，要尽可能选择范围小的数据类型。比如，某个字段的取值最大值不会超过 100，那么选择 byte 类型即可。迄今为止，吉尼斯世界记录的人类的年龄的最大值为 134 岁，对于年龄字段，short 类型足矣。字段的长度越短，索引和搜索的效率越高。

处理浮点数时，优先考虑使用 scaled_float 类型。scaled_float 是通过缩放因子把浮点数变成 long 类型，比如价格只需要精确到分，price 字段的取值为 57.34，设置放大因子为 100，存储起来就是 5734。所有的 API 都会把 price 的取值当作浮点数，事实上 Elasticsearch 底层存储的是整数类型，因为压缩整数比压缩浮点数更加节省存储空间。

数字类型配置映射的例子如下：
```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "number_of bytes": {
          "type": "integer"
        },
        "time_in_seconds": {
          "type": "float"
        },
        "price": {
          "type": "scaled_float",
          "scaling_factor": 100
        }
      }
    }
  }
}
```

### 日期类型
JSON 中没有日期类型，所以在 Elasticsearch 中的日期可以是以下几种形式：
- 格式化日期的字符串，如 “2015-01-01” 或 “2015/01/01 12:10:30”。
- 代表 milliseconds-since-the-epoch 的长整型数（epoch 指的是一个特定的时间：1970-01-01 00:00:00 UTC）。
- 代表 seconds-since-the-epoch 的整型数。
Elasticsearch 内部会把日期转换为 UTC（世界标准时间），并将其存储为表示 milliseconds-since-the-epoch 的长整型数，这样做的原因是和字符串相比，数值在存储和处理时更快。日期格式可以自定义，如果没有自定义，默认格式如下：
```
"strict_date_optional_time||epoch_millis"
```

日期类型配置映射的例子如下：
```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "dt": {
          "type": "date"
        }
      }
    }
  }
}
```

写入 3 个文档：
```
PUT my_index/my_type/1
{
"dt": "2019-11-14"
}
PUT my_index/my_type/2
{
"dt": "2019-11-14T13:16:302"
}
PUT my_index/my_type/3
{
"dt": 1573664468000
}
```
默认情况下，以上 3 个文档的日期格式都可以被解析，内部存储的是毫秒计时的长整型数。

### 布尔类型
如果一个字段是布尔类型，可接受的值为 true、false。Elasticsearch 5.4 版本以前，可以接受被解释为 true 或 false 的字符串和数字，5.4 版本以后只接受 true、false、"true"、"false"。

布尔类型配置映射的例子如下：
```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "is_published": {
          "type": "boolean"
        }
      }
    }
  }
}
```

写入 3 条文档：
```
PUT my_index/my_type/1
{
  "is_published": true
}
PUT my_index/my_type/2
{
  "is_published": "true"
}
PUT my_index/my_type/3
{
  "is_published": false
}
```

执行以下搜索，文档 1 和文档 2 都可以被搜索：
```
GET my_index/_search
{
  "query": {
    "term": {
      "is_published": true
    }
  }
}
```

### binary 类型
binary 类型接受 base64 编码的字符串，默认不存储（这里的不存储是指 store 属性取值为 false），也不可搜索。

布尔类型配置映射的例子如下：
```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "name": {
          "type": "text"
        },
        "blob": {
          "type": "binary"
        }
      }
    }
  }
}
```
写入一条文档，其中 blob 的值为字符串 "Some binary blob" 的 Base64 编码：
```
PUT my_index/my_type/1
{
  "name": "Some binary blob",
  "blob": "U29t2SBiaW5hcnkgYmxvYg=="
}
```

### range 类型
range 类型的使用场景包括网页中的时间选择表单、年龄范围选择表单等

下面代码创建了一个 range_index 索引，expected_attendees 字段为 integer_range 类型，time_frame 字段为 date_range 类型。
```
PUT range_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "expected_attendees": {
          "type": "integer_range"
        },
        "time_frame": {
          "type": "date_range",
          "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
        }
      }
    }
  }
}
```

索引一条文档，expected_attendees 的取值为 10 到 20，time_frame 的取值是 2015-10-31 12:00:00 至 2015-11-01，命令如下：
```
PUT my_index/my_type/1
{
  "expected_attendees": {
    "gte": 10,
    "lte": 20
  },
  "time_frame": {
    "gte": "2019-10-31 12:00:00",
    "lte": "2019-11-01"
  }
}
```

### 复合类型
Elasticsearch 字段类型的复合类型有数组类型、对象类型和嵌套类型。

数组类型
Elasticsearch 没有专用的数组类型，默认情况下任何字段都可以包含0个或者多个值，但是一个数组中的值必须是同一种类型。例如：
- 字符数组：["one","two"]
- 整型数组：[1,3]
- 嵌套数组：[1,[2,3]]，等价于 [1,2,3]
- 对象数组：[{"city":"amsterdam","country":"nederland"},{"city":"brussels","country":"belgium"}]
动态添加数据时，数组的第一个值的类型决定整个数组的类型。混合数组类型是不支持的，比如：[1，"abc"]。数组可以包含 null 值，空数组 [ ] 会被当作 missing field 对待。

在文档中使用 array 类型不需要提前做任何配置，默认支持。例如写入一条带有数组类型的文档，命令如下：
```
PUT my_index/my_type/1
{
  "message": "some arrays in this document...",
  "tags": ["elasticsearch", "wow"],
  "lists": [{
    "name": "prog_list",
    "description": "programming list"
  },
  {
    "name": "cool_list",
    "description": "cool stuff list"
  }]
}
```
搜索 lists 字段下的 name，命令如下：
```
GET my_index/_search
{
  "query": {
    "match": {
      "lists.name": "cool_list"
    }
  }
}
```

### object 类型
JSON 本质上具有层级关系，文档包含内部对象，内部对象本身还包含内部对象，请看下面的例子：
```
{
  "region": "US",
  "manager": {
    "age": 30,
    "name": {
      "first": "John",
      "last": "Smith"
    }
  }
}
```

上面的文档中，整体是一个 JSON 对象，JSON 中包含一个 manager 对象，manager 对象又包含名为 name 的内部对象。写入到 Elasticsearch 之后，文档会被索引成简单的扁平 key-value 对，格式如下：
```
{
  "region": "US",
  "manager.age": 30,
  "manager.name.first": "John",
  "manager.name.last": "Smith"
}
```

上面文档结构的显式映射如下：
```
{
  "mappings": {
    "my_type": {
      "properties": {
        "region": {
          "type": "keyword"
        },
        "manager": {
          "properties": {
            "age": {
              "type": "integer"
            },
            "name": {
              "properties": {
                "first": {
                  "type": "text"
                },
                "last": {
                  "type": "text"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### nested 类型
nested 类型是 object 类型中的一个特例，可以让对象数组独立索引和查询。Lucene 没有内部对象的概念，所以 Elasticsearch 将对象层次扁平化，转化成字段名字和值构成的简单列表。

使用 Object 类型有时会出现问题，比如文档 my_index/my_type/1 的结构如下：
```
PUT my_index/my_type/1
{
  "group": "fans",
  "user": [{
    "first": "John",
    "last": "Smith"
  }, {
    "first": "Alice",
    "last": "White"
  }]
}
```

user 字段会被动态添加为 Object 类型，最后会被转换为以下平整的形式：
```
{
  "group": "fans",
  "user.first": ["alice", "john"],
  "user.last": ["smith", "white"]
}
```

user.first 和 user.last 扁平化以后变为多值字段，alice 和 white 的关联关系丢失了。执行以下搜索会搜索到上述文档：
```
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [{
        "match": {
          "user.first": "Alice"
        }
      },
      {
        "match": {
          "user.last": "Smith"
        }
      }]
    }
  }
}
```

事实上是不应该匹配的，如果需要索引对象数组并避免上述问题的产生，应该使用 nested 对象类型而不是 object 类型，nested 对象类型可以保持数组中每个对象的独立性。Nested 类型将数组中每个对象作为独立隐藏文档来索引，这意味着每个嵌套对象都可以独立被搜索，映射中指定 user 字段为 nested 类型：
```
PUT /my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "user": {
          "type": "nested"
        }
      }
    }
  }
}
```

再次执行上述查询语句，文档不会被匹配。

索引一个包含 100 个 nested 字段的文档实际上就是索引 101 个文档，每个嵌套文档都作为一个独立文档来索引。为了防止过度定义嵌套字段的数量，每个索引可以定义的嵌套字段被限制在 50 个。

### 地理类型
Elasticsearch 的地理相关类型有地理坐标类型和地理图形类型。

geo_point 地理坐标
geo point 类型用于存储地理位置信息的经纬度，可用于以下几种场景：
- 查找一定范围内的地理位置。
- 通过地理位置或者相对中心点的距离来聚合文档。
- 把距离因素整合到文档的评分中。
- 通过距离对文档排序。

geo_shape 地理图形
geo_point 类型可以存储一个坐标点，geo_shape 类型可以存储一块区域，比如矩形、三角形或者其他多边形。GeoJSON 是一种对各种地理数据结构进行编码的格式，对象可以表示几何、特征或者特征集合，支持点、线、面、多点、多线、多面等几何类型。GeoJSON 里的特征包含一个几何对象和其他属性，特征集合表示一系列特征。

### 特殊类型
Elasticsearch 特殊类型有 IP 类型、范围类型、令牌计数类型、附件类型和抽取类型。

ip 类型
ip 类型的字段用于存储 IPv4 或者 IPv6 的地址。在映射中指定字段为 ip 类型的映射和查询语句如下：
```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "id_addr": {
          "type": "ip"
        }
      }
    }
  }
}
PUT my_index/my_type/1
{
  "ip_addr": "192.168.1.1"
}
GET my_index/_search
{
  "query": {
    "term": {
      "ip_addr": "192.168.0.0/16"
    }
  }
}
```

## mapping 属性
elasticsearch 的 mapping 中的字段属性非常多
- type	字段类型，常用的有 text、integer 等等。
- store	是否存储指定字段，可选值为 true|false，设置 true 意味着需要开辟单独的存储空间为这个字段做存储，而且这个存储是独立于 _source 的存储的。
- norms	是否使用归一化因子，可选值为 true|false，不需要对某字段进行打分排序时，可禁用它，节省空间；type 为 text 时，默认为 true；而 type 为 keyword 时，默认为 false。
- index_options 索引选项控制添加到倒排索引（Inverted Index）的信息，这些信息用于搜索（Search）和高亮显示：
  - docs 只索引文档编号(Doc Number)；
  - freqs：索引文档编号和词频率（term frequency）；
  - positions：索引文档编号，词频率和词位置（序号）；
  - offsets：索引文档编号，词频率，词偏移量（开始和结束位置）和词位置（序号）。

默认情况下，被分析的字符串（analyzed string）字段使用 positions，其他字段默认使用 docs。

此外，需要注意的是 index_option 是 elasticsearch 特有的设置属性；临近搜索和短语查询时，index_option 必须设置为 offsets，同时高亮也可使用 postings highlighter。

- term_vector 索引选项控制词向量相关信息：
  - no：默认值，表示不存储词向量相关信息；
  - yes：只存储词向量信息；
  - with_positions：存储词项和词项位置；
  - with_offsets：存储词项和字符偏移位置；
  - with_positions_offsets：存储词项、词项位置、字符偏移位置。
term_vector 是 lucene 层面的索引设置。

- similarity 指定文档相似度算法（也可以叫评分模型）： BM25：es 5 之后的默认设置。
- copy_to	复制到自定义 _all 字段，值是数组形式，即表明可以指定多个自定义的字段。
- analyzer	指定索引和搜索时的分析器，如果同时指定 search_analyzer 则搜索时会优先使用 search_analyzer。
- search_analyzer	指定搜索时的分析器，搜索时的优先级最高。
- fielddata  默认是 false，因为 doc_values 不支持 text 类型，所以有了 fielddata，fielddata 是 text 版本的 doc_values，也是为了优化字段进行排序、聚合和脚本访问。
和 doc_values 不同的是，fielddata 使用的是内存，而不是磁盘；因为 fielddata 会消耗大量的堆内存，fielddata 一旦加载到堆中，在 segment 的生命周期之内都将一致保持在堆中，所以谨慎使用。

## 查询索引
Elasticsearch的最多使用的场景就是用它的查询API，它提供完备的查询功能以满足现实中的各种需求。

### 多个index、多个type查询
Elasticsearch的搜索api支持一个索引（index）的多个类型（type）查询以及多个索引（index）的查询。

例如，我们可以搜索twitter索引下面所有匹配条件的所有类型中文档，如下：
```
GET /twitter/_search?q=user:shay
```

我们也可以搜索一个索引下面指定多个type下匹配条件的文档，如下：
```
GET /twitter/tweet,user/_search?q=user:banon
```

我们也可以搜索多个索引下匹配条件的文档，如下：
```
GET /twitter,elasticsearch/_search?q=tag:wow
```

此外我们也可以搜索所有索引下匹配条件的文档，用_all表示所有索引，如下：
```
GET /_all/_search?q=tag:wow
```

甚至我们可以搜索所有索引及所有type下匹配条件的文档，如下：
```
GET /_search?q=tag:wow
```

### URI搜索
Elasticsearch支持用uri搜索，可用get请求里面拼接相关的参数，并用curl相关的命令就可以进行测试。

如下有一个示例：
```
GET twitter/_search?q=user:kimchy
```

如下是上一个请求的相应实体：
```
{
    "timed_out": false,
    "took": 62,
    "_shards":{
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
    },
    "hits":{
        "total" : 1,
        "max_score": 1.3862944,
        "hits" : [
            {
                "_index" : "twitter",
                "_type" : "_doc",
                "_id" : "0",
                "_score": 1.3862944,
                "_source" : {
                    "user" : "kimchy",
                    "date" : "2009-11-15T14:12:12",
                    "message" : "trying out Elasticsearch",
                    "likes": 0
                }
            }
        ]
    }
}
```

### 参数选项
- q	查询字符串，映射到query_string查询
- df	在查询中未定义字段前缀时使用的默认字段
- analyzer	查询字符串时指定的分词器
- analyze_wildcard	是否允许通配符和前缀查询，默认设置为false
- batched_reduce_size	应在协调节点上一次减少的分片结果数。如果请求中潜在的分片数量很大，则应将此值用作保护机制，以减少每个搜索请求的内存开销
- default_operator	默认使用的匹配运算符，可以是AND或者OR，默认是OR
- lenient	如果设置为true，将会忽略由于格式化引起的问题（如向数据字段提供文本），默认为false
- explain	对于每个hit，包含了具体如何计算得分的解释
- _source	请求文档内容的参数，默认true；设置false的话，不返回_source字段，可以使用_source_include和_source_exclude参数分别指定返回字段和不返回的字段
- stored_fields	指定每个匹配返回的文档中的存储字段，多个用逗号分隔。不指定任何值将导致没有字段返回
- sort	排序方式，可以是fieldName、fieldName:asc或者fieldName:desc的形式。fieldName可以是文档中的实际字段，也可以是诸如_score字段，其表示基于分数的排序。此外可以指定多个sort参数（顺序很重要）
- track_scores	当排序时，若设置true，返回每个命中文档的分数
- track_total_hits	是否返回匹配条件命中的总文档数，默认为true
- timeout	设置搜索的超时时间，默认无超时时间
- terminate_after	在达到查询终止条件之前，指定每个分片收集的最大文档数。如果设置，则在响应中多了一个terminated_early的布尔字段，以指示查询执行是否实际上已终止。默认为no terminate_after
- from	从第几条（索引以0开始）结果开始返回，默认为0
- size	返回命中的文档数，默认为10
- search_type	搜索的方式，可以是dfs_query_then_fetch或query_then_fetch。默认为query_then_fetch
- allow_partial_search_results	是否可以返回部分结果。如设置为false，表示如果请求产生部分结果，则设置为返回整体故障；默认为true，表示允许请求在超时或部分失败的情况下获得部分结果

## 查询流程
在Elasticsearch中，查询是一个比较复杂的执行模式，因为我们不知道那些document会被匹配到，任何一个shard上都有可能，所以一个search请求必须查询一个索引或多个索引里面的所有shard才能完整的查询到我们想要的结果。

找到所有匹配的结果是查询的第一步，来自多个shard上的数据集在分页返回到客户端之前会被合并到一个排序后的list列表，由于需要经过一步取top N的操作，所以search需要进过两个阶段才能完成，分别是query和fetch。