---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch
在使用es时，我们经常会用到聚合查询。

## 聚合概念
聚合（aggs）不同于普通查询，是目前学到的第二种大的查询分类，第一种即“query”，因此在代码中的第一层嵌套由“query”变为了“aggs”。用于进行聚合的字段必须是exact value，分词字段不可进行聚合，对于text字段如果需要使用聚合，需要开启fielddata，但是通常不建议，因为fielddata是将聚合使用的数据结构由磁盘（docvalues）变为了堆内存（fielddata），大数据的聚合操作很容易导致OOM

聚合分类
- 分桶聚合（Bucket agregations）
类比SQL中的group by的作用，主要用于统计不同类型数据的数量
- 指标聚合（Metrics agregations）
主要用于最大值、最小值、平均值、字段之和等指标的统计
- 管道聚合（Pipeline agregations）
用于对聚合的结果进行二次聚合，如要统计绑定数量最多的标签bucket，就是要先按照标签进行分桶，再在分桶的结果上计算最大值。

语法
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

### 桶聚合：
场景：用于统计不同种类的文档的数量，可进行嵌套统计。

函数：terms

注意：聚合字段必须是exact value，如keyword

### 指标聚合
场景：用于统计某个指标，如最大值、最小值、平均值，可以结合桶聚合一起使用，如按照商品类型分桶，统计每个桶的平均价格。

函数：平均值：Avg、最大值：Max、最小值：Min、求和：Sum、详细信息：Stats、数量：Value count

### 管道聚合
场景：用于对聚合查询的二次聚合，如统计平均价格最低的商品分类，即先按照商品分类进行桶聚合，并计算其平均价格，然后对其平均价格计算最小值聚合

函数：Min bucket：最小桶、Max bucket：最大桶、Avg bucket：桶平均值、Sum bucket：桶求和、Stats bucket：桶信息

注意：bucketspath为管道聚合的关键字，其值从当前聚合统计的聚合函数开始计算为第一级。比如下面例子中，myaggs和myminbucket同级， myaggs就是bucketspath值的起始值。

```
json GET product/_search 
{
    "size": 0,
    "aggs": {
        "my_aggs": {
            "terms": {
                ...
            },
            "aggs": {
                "my_price_bucket": {
                    ...
                }
            }
        },
        "my_min_bucket": {
            "min_bucket": {
                "buckets_path": "my_aggs>price_bucket"
            }
        }
    }
}
```

### 嵌套聚合
语法：
```
json GET product/_search 
{
    "size": 0,
    "aggs": {
        "<agg_name>": {
            "<agg_type>": {
                "field": "<field_name>"
            },
            "aggs": {
                "<agg_name_child>": {
                    "<agg_type>": {
                        "field": "<field_name>"
                    }
                }
            }
        }
    }
}
```
用途：用于在某种聚合的计算结果之上再次聚合，如统计不同类型商品的平均价格，就是在按照商品类型桶聚合之后，在其结果之上计算平均价格

## 聚合和查询的相互关系

### 基于query或filter的聚合
```
json GET product/_search 
{
    "query": {
        ...
    },
    "aggs": {
        ...
    }
}
```
注意：以上语法，执行顺序为先query后aggs，顺序和谁在上谁在下没有关系。query中可以是查询、也可以是filter、或者bool query

### 基于聚合结果的查询
```
GET product/_search 
{
    "aggs": {
        ...
    },
    "post_filter": {
        ...
    }
}
```
注意：以上语法，执行顺序为先aggs后post_filter，顺序和谁在上谁在下没有关系。

### 查询条件的作用域
```
json GET product/_search 
{
    "size": 10,
    "query": {
        ...
    },
    "aggs": {
        "avg_price": {
            ...
        },
        "all_avg_price": {
            "global": {
                
            },
            "aggs": {
                ...
            }
        }
    }
}
```
上面例子中，avgprice的计算结果是基于query的查询结果的，而allavg_price的聚合是基于all data的

排序规则：
ordertype：count（数量） _key（聚合结果的key值） _term（废弃但是仍然可用，使用_key代替）

```
json GET product/_search 
{
    "aggs": {
        "type_agg": {
            "terms": {
                "field": "tags",
                "order": {
                    "<order_type>": "desc"
                },
                "size": 10
            }
        }
    }
}
```

多级排序：即排序的优先级，按照外层优先的顺序
```
json GET product/_search?size=0 
{
    "aggs": {
        "first_sort": {
            ..."aggs": {
                "second_sort": {
                    ...
                }
            }
        }
    }
}
```
上例中，先按照firstsort排序，再按照secondsort排序










