---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch简化开发


## 简介
Easy-Es是一款简化ElasticSearch搜索引擎操作的开源框架,全自动智能索引托管。

目前功能丰富度和易用度已全面领先SpringData-Elasticsearch。简化CRUD及其它高阶操作,可以更好的帮助开发者减轻开发负担。底层采用Es官方提供的RestHighLevelClient,保证其原生性能及拓展性。

## 优点
- 全自动索引托管: 全球开源首创的索引托管模式,开发者无需关心索引的创建更新及数据迁移等繁琐步骤,索引全生命周期皆可托管给框架,由框架自动完成,过程零停机,用户无感知,彻底解放开发者
- 屏蔽语言差异: 开发者只需要会MySQL语法即可使用Es
- 代码量极少: 与直接使用RestHighLevelClient相比,相同的查询平均可以节3-8倍左右的代码量
- 零魔法值: 字段名称直接从实体中获取,无需输入字段名称字符串这种魔法值
- 零额外学习成本: 开发者只要会国内最受欢迎的Mybatis-Plus语法,即可无缝迁移至Easy-Es
- 降低开发者门槛: 即便是只了解ES基础的初学者也可以轻松驾驭ES完成绝大多数需求的开发
- 功能强大: 支持MySQL的几乎全部功能,且对ES特有的分词,权重,高亮,嵌套,地理位置Geo,Ip地址查询等功能都支持
- 完善的中英文文档: 提供了中英文双语操作文档,文档全面可靠,帮助您节省更多时间

## 实战

### Maven依赖
```xml
<dependency>
    <groupId>cn.easy-es</groupId>
    <artifactId>easy-es-boot-starter</artifactId>
    <version>Latest Version</version>
</dependency>
```

需求:查询出文档标题为 "传统功夫"且作者为"码保国"的所有文档
```
// 使用Easy-Es仅需1行代码即可完成查询
    List<Document> documents = documentMapper.selectList(EsWrappers.lambdaQuery(Document.class).eq(Document::getTitle, "传统功夫").eq(Document::getCreator, "码保国"));
```

```
    // 传统方式, 直接用RestHighLevelClient进行查询 需要19行代码,还不包含下划线转驼峰,自定义字段处理及_id处理等代码
    String indexName = "document";
    SearchRequest searchRequest = new SearchRequest(indexName);
    BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
    TermQueryBuilder titleTerm = QueryBuilders.termQuery("title", "传统功夫");
    TermsQueryBuilder creatorTerm = QueryBuilders.termsQuery("creator", "码保国");
    boolQueryBuilder.must(titleTerm);
    boolQueryBuilder.must(creatorTerm);
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    searchSourceBuilder.query(boolQueryBuilder);
    searchRequest.source(searchSourceBuilder);
    try {
         SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
         List<Document> documents = Optional.ofNullable(searchResponse)
                .map(SearchResponse::getHits)
                .map(SearchHits::getHits)
                .map(hit->Document document = JSON.parseObject(hit.getSourceAsString(),Document.class))
                .collect(Collectors.toList());
        } catch (IOException e) {
            e.printStackTrace();
        }
```
































