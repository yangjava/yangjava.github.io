---
layout: post
categories: ElasticSearch
description: none
keywords: ElasticSearch
---
# ElasticSearch实战
整合ES的要求很低，只要能发送请求，那它就能操作ES，因此，下面来分析一下那种整合ES的方式更为优雅高效

## ES端口选择

ES服务器有两个可选端口，一个是9200（HTTP），一个是9300（TCP），可以通过操作TCP连接来通过9300端口进行ES操作，但是官方不建议使用9300来进行操作，后续版本会废弃相关的jar包，因此我们的端口选择只能是9200

## 第三方工具选择

既然是只能操作9200端口，那也就是能发送HTTP请求的工具都是可以的（比如都可以使用postman来进行es操作）
- JestClient：版本更新慢，更新周期长（非官方）
- RestTemplate：Spring提供可以发送任何HTTP请求，但是操作ES麻烦，需要拼接字符串
- HTTPClient，OKHTTP：也是发送HTTP请求的工具
- Elasticsearch-Rest-Client：官方的RestClient，对ES操作进行了封装，使用简单，版本更新也比较及时（最终选择）。


## Java整合ES

### Maven依赖
```xml
<dependency>
    <groupId>org.elasticsearch</groupId>
    <artifactId>elasticsearch</artifactId>
    <version>7.1.0</version>
</dependency>
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
    <version>7.1.0</version>
</dependency>
```

### 添加配置
```yml
elasticsearch:
  host: 127.0.0.1
  port: 9200
  connectTimeout: 3000
  socketTimeout: 5000
  connectionRequestTimeout: 500
```

### 初始化Es配置并创建客户端
创建`ElasticSearchConfig`，会从配置文件中读取到对应的参数，接着申明一个`restHighLevelClient`方法，返回的是一个 RestHighLevelClient，同时为它添加 @Bean(destroyMethod = “close”) 注解，当 destroy 的时候做一个关闭，这个方法主要是如何初始化并创建一个 RestHighLevelClient。

```java

import org.apache.http.HttpHost;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestClientBuilder;
import org.elasticsearch.client.RestHighLevelClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ElasticSearchConfig {

    @Value("${elasticSearch.host}")
    private String host;
    @Value("${elasticSearch.port}")
    private int port;

    @Value("${elasticSearch.connectTimeout}")
    private int connectTimeout;

    @Value("${elasticSearch.socketTimeout}")
    private int socketTimeout;

    @Value("${elasticSearch.connectionRequestTimeout}")
    private int connectionRequestTimeout;


    @Bean(destroyMethod = "close", name = "client")
    public RestHighLevelClient restHighLevelClient() {
        RestClientBuilder builder = RestClient.builder(new HttpHost(host, port))
                .setRequestConfigCallback(requestConfigBuilder -> requestConfigBuilder
                        .setConnectTimeout(connectTimeout)
                        .setSocketTimeout(socketTimeout)
                        .setConnectionRequestTimeout(connectionRequestTimeout));
        return new RestHighLevelClient(builder);
    }

}
```


### ES操作实战
```java

import com.alibaba.fastjson.JSON;
import com.example.constant.Constant;
import com.example.entity.UserDocument;
import lombok.extern.slf4j.Slf4j;
import org.elasticsearch.action.admin.indices.delete.DeleteIndexRequest;
import org.elasticsearch.action.bulk.BulkRequest;
import org.elasticsearch.action.bulk.BulkResponse;
import org.elasticsearch.action.delete.DeleteRequest;
import org.elasticsearch.action.delete.DeleteResponse;
import org.elasticsearch.action.get.GetRequest;
import org.elasticsearch.action.get.GetResponse;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.index.IndexResponse;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.support.master.AcknowledgedResponse;
import org.elasticsearch.action.update.UpdateRequest;
import org.elasticsearch.action.update.UpdateResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.client.indices.CreateIndexRequest;
import org.elasticsearch.client.indices.CreateIndexResponse;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.xcontent.XContentType;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.index.query.TermQueryBuilder;
import org.elasticsearch.rest.RestStatus;
import org.elasticsearch.search.aggregations.AggregationBuilders;
import org.elasticsearch.search.aggregations.Aggregations;
import org.elasticsearch.search.aggregations.bucket.terms.Terms;
import org.elasticsearch.search.aggregations.bucket.terms.TermsAggregationBuilder;
import org.elasticsearch.search.aggregations.metrics.Avg;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;
@Slf4j
@Service
public class EsService {

    @Autowired
    @Qualifier("restHighLevelClient")
    public RestHighLevelClient client;

    public boolean createUserIndex(String index) throws IOException {

        //创建索引(建表)
        CreateIndexRequest createIndexRequest = new CreateIndexRequest(index);
        createIndexRequest.settings(Settings.builder()
                .put("index.number_of_shards", 1)
                .put("index.number_of_replicas", 0)
        );
        createIndexRequest.mapping("{\n" +
                "  \"properties\": {\n" +
                "    \"city\": {\n" +
                "      \"type\": \"keyword\"\n" +
                "    },\n" +
                "    \"sex\": {\n" +
                "      \"type\": \"keyword\"\n" +
                "    },\n" +
                "    \"name\": {\n" +
                "      \"type\": \"keyword\"\n" +
                "    },\n" +
                "    \"id\": {\n" +
                "      \"type\": \"keyword\"\n" +
                "    },\n" +
                "    \"age\": {\n" +
                "      \"type\": \"integer\"\n" +
                "    }\n" +
                "  }\n" +
                "}", XContentType.JSON);
        CreateIndexResponse createIndexResponse = client.indices().create(createIndexRequest, RequestOptions.DEFAULT);
        return createIndexResponse.isAcknowledged();
    }

    //删除索引(删表)
    public Boolean deleteUserIndex(String index) throws IOException {
        DeleteIndexRequest deleteIndexRequest = new DeleteIndexRequest(index);
        AcknowledgedResponse deleteIndexResponse = client.indices().delete(deleteIndexRequest, RequestOptions.DEFAULT);
        return deleteIndexResponse.isAcknowledged();
    }

    //创建文档(插入数据)
    public Boolean createUserDocument(UserDocument document) throws Exception {
        UUID uuid = UUID.randomUUID();
        document.setId(uuid.toString());
        IndexRequest indexRequest = new IndexRequest(Constant.INDEX)
                .id(document.getId())
                .source(JSON.toJSONString(document), XContentType.JSON);
        IndexResponse indexResponse = client.index(indexRequest, RequestOptions.DEFAULT);
        return indexResponse.status().equals(RestStatus.OK);
    }

    //批量创建文档
    public Boolean bulkCreateUserDocument(List<UserDocument> documents) throws IOException {
        BulkRequest bulkRequest = new BulkRequest();
        for (UserDocument document : documents) {
            String id = UUID.randomUUID().toString();
            document.setId(id);
            IndexRequest indexRequest = new IndexRequest(Constant.INDEX)
                    .id(id)
                    .source(JSON.toJSONString(document), XContentType.JSON);
            bulkRequest.add(indexRequest);
        }
        BulkResponse bulkResponse = client.bulk(bulkRequest, RequestOptions.DEFAULT);
        return bulkResponse.status().equals(RestStatus.OK);
    }

    //查看文档
    public UserDocument getUserDocument(String id) throws IOException {
        GetRequest getRequest = new GetRequest(Constant.INDEX, id);
        GetResponse getResponse = client.get(getRequest, RequestOptions.DEFAULT);
        UserDocument result = new UserDocument();
        if (getResponse.isExists()) {
            String sourceAsString = getResponse.getSourceAsString();
            result = JSON.parseObject(sourceAsString, UserDocument.class);
        } else {
            log.error("没有找到该 id 的文档");
        }
        return result;
    }
    //更新文档
    public Boolean updateUserDocument(UserDocument document) throws Exception {
        UserDocument resultDocument = getUserDocument(document.getId());
        UpdateRequest updateRequest = new UpdateRequest(Constant.INDEX, resultDocument.getId());
        updateRequest.doc(JSON.toJSONString(document), XContentType.JSON);
        UpdateResponse updateResponse = client.update(updateRequest, RequestOptions.DEFAULT);
        return updateResponse.status().equals(RestStatus.OK);
    }

    //删除文档
    public String deleteUserDocument(String id) throws Exception {
        DeleteRequest deleteRequest = new DeleteRequest(Constant.INDEX, id);
        DeleteResponse response = client.delete(deleteRequest, RequestOptions.DEFAULT);
        return response.getResult().name();
    }

    //搜索操作
    public List<UserDocument> searchUserByCity(String city) throws Exception {
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(Constant.INDEX);
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        TermQueryBuilder termQueryBuilder = QueryBuilders.termQuery("city", city);
        searchSourceBuilder.query(termQueryBuilder);
        searchRequest.source(searchSourceBuilder);
        SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);
        return getSearchResult(searchResponse);
    }

    //聚合搜索
    public List<UserCityDTO> aggregationsSearchUser() throws Exception {
        SearchRequest searchRequest = new SearchRequest(Constant.INDEX);
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        TermsAggregationBuilder aggregation = AggregationBuilders.terms("by_city")
                .field("city")
                .subAggregation(AggregationBuilders
                        .avg("average_age")
                        .field("age"));
        searchSourceBuilder.aggregation(aggregation);
        searchRequest.source(searchSourceBuilder);
        SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);
        Aggregations aggregations = searchResponse.getAggregations();
        Terms byCityAggregation = aggregations.get("by_city");
        List<UserCityDTO> userCityList = new ArrayList<>();
        for (Terms.Bucket buck : byCityAggregation.getBuckets()) {
            UserCityDTO userCityDTO = new UserCityDTO();
            userCityDTO.setCity(buck.getKeyAsString());
            userCityDTO.setCount(buck.getDocCount());
            // 获取子聚合
            Avg averageBalance = buck.getAggregations().get("average_age");
            userCityDTO.setAvgAge(averageBalance.getValue());
            userCityList.add(userCityDTO);
        }
        return userCityList;
    }
}
```







## SpringBoot整合ES
SpringBoot支持两种技术和es交互。一种的jest，还有一种就是SpringData-ElasticSearch。 其根据引入的依赖不同而选择不同的技术。反正作为spring全家桶目前是以springdata为主流使用技术。

### Maven依赖
```xml
        <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
                <version>2.2.0.RELEASE</version>
        </dependency>
```
注意这里要引入springBoot整合es的场景启动器。可以简单看下这个场景启动器里面都有啥依赖:
```xml
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
      <version>2.2.0.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.data</groupId>
      <artifactId>spring-data-elasticsearch</artifactId>
      <version>3.2.0.RELEASE</version>
      <scope>compile</scope>
      <exclusions>
        <exclusion>
          <artifactId>jcl-over-slf4j</artifactId>
          <groupId>org.slf4j</groupId>
        </exclusion>
        <exclusion>
          <artifactId>log4j-core</artifactId>
          <groupId>org.apache.logging.log4j</groupId>
        </exclusion>
      </exclusions>
    </dependency>
  </dependencies>
```

## SpringBoot常用的Es查询
```java

import lombok.extern.slf4j.Slf4j;
import org.apache.lucene.search.join.ScoreMode;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.index.query.BoolQueryBuilder;
import org.elasticsearch.index.query.NestedQueryBuilder;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.aggregations.AggregationBuilder;
import org.elasticsearch.search.aggregations.AggregationBuilders;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.elasticsearch.search.sort.SortOrder;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import java.io.IOException;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
@Slf4j
public class SearchJunit {

    @Value("${elasticSearch.segment.index}")
    private String segmentIndex;

    @Autowired
    private RestHighLevelClient restHighLevelClient;

    /***************单条件精确查询********************/
    /**
     * 单条件精确查询
     *
     * @throws Exception
     */
    @Test
    public void search0() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .query(QueryBuilders.termsQuery("trace_id", "252fc2514603450d96f536261e1b7e91"));

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }

    /***************多条件精确查询，取并集********************/
    /**
     * 多条件精确查询，取并集
     *
     * @throws Exception
     */
    @Test
    public void search1() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .query(QueryBuilders.termsQuery("trace_id", "252fc2514603450d96f536261e1b7e91", "7b08e1c22159419982f83c0cc9965d00"));

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /***************范围查询********************/


    /**
     * 范围查询，包括from、to
     *
     * @throws Exception
     */
    @Test
    public void search2() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .query(QueryBuilders.rangeQuery("age").from(20).to(32));

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * 范围查询，不包括from、to
     *
     * @throws Exception
     */
    @Test
    public void search3() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .query(QueryBuilders.rangeQuery("age").from(20, false).to(30, false));

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * 范围查询, lt:小于，gt:大于
     *
     * @throws Exception
     */
    @Test
    public void search4() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .query(QueryBuilders.rangeQuery("age").lt(30).gt(20));

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * 模糊查询，支持通配符
     *
     * @throws Exception
     */
    @Test
    public void search5() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .query(QueryBuilders.wildcardQuery("name", "张三"));

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * 不使用通配符的模糊查询，左右匹配
     *
     * @throws Exception
     */
    @Test
    public void search6() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .query(QueryBuilders.queryStringQuery("张三").field("name"));

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * 多字段模糊查询
     *
     * @throws Exception
     */
    @Test
    public void search7() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .query(QueryBuilders.multiMatchQuery("长", "name", "city"));

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }

    /**
     * 分页搜索
     *
     * @throws Exception
     */
    @Test
    public void search8() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .from(0).size(2);

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * 排序，字段的类型必须是：integer、double、long或者keyword
     *
     * @throws Exception
     */
    @Test
    public void search9() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .sort("createTime", SortOrder.ASC);

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * 精确统计筛选文档数,查询性能有所降低
     *
     * @throws Exception
     */
    @Test
    public void search10() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .trackTotalHits(true);

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * 设置源字段过虑,第一个参数结果集包括哪些字段，第二个参数表示结果集不包括哪些字段
     *
     * @throws Exception
     */
    @Test
    public void search11() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .fetchSource(new String[]{"name", "age", "city", "createTime"}, new String[]{});

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }

    /**
     * 根据id精确匹配
     *
     * @throws Exception
     */
    @Test
    public void search12() throws Exception {
        String[] ids = new String[]{"1", "2"};
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .query(QueryBuilders.termsQuery("_id", ids));

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * matchAllQuery搜索全部
     *
     * @throws Exception
     */
    @Test
    public void search21() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .query(QueryBuilders.matchAllQuery());

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * match搜索匹配
     *
     * @throws Exception
     */
    @Test
    public void search22() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder()
                .query(QueryBuilders.matchQuery("name", "张王"));

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * bool组合查询
     *
     * @throws Exception
     */
    @Test
    public void search23() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder();

        BoolQueryBuilder boolQueryBuilder = new BoolQueryBuilder();
        boolQueryBuilder.must(QueryBuilders.matchQuery("name", "张王"));
        boolQueryBuilder.must(QueryBuilders.rangeQuery("age").lte(30).gte(20));
        builder.query(boolQueryBuilder);

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * nested类型嵌套查询
     *
     * @throws Exception
     */
    @Test
    public void search24() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder();

        //条件查询
        BoolQueryBuilder mainBool = new BoolQueryBuilder();
        mainBool.must(QueryBuilders.matchQuery("name", "赵六"));

        //nested类型嵌套查询
        BoolQueryBuilder boolQueryBuilder = new BoolQueryBuilder();
        boolQueryBuilder.must(QueryBuilders.matchQuery("products.brand", "A"));
        boolQueryBuilder.must(QueryBuilders.matchQuery("products.title", "巧克力"));
        NestedQueryBuilder nested = QueryBuilders.nestedQuery("products", boolQueryBuilder, ScoreMode.None);
        mainBool.must(nested);

        builder.query(mainBool);

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }

    /**
     * 多条件查询 + 排序 + 分页
     *
     * @throws Exception
     */
    @Test
    public void search29() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder();

        //条件搜索
        BoolQueryBuilder boolQueryBuilder = new BoolQueryBuilder();
        boolQueryBuilder.must(QueryBuilders.matchQuery("name", "张王"));
        boolQueryBuilder.must(QueryBuilders.rangeQuery("age").lte(30).gte(20));
        builder.query(boolQueryBuilder);

        //结果集合分页
        builder.from(0).size(2);

        //排序
        builder.sort("createTime", SortOrder.ASC);

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * 聚合查询 sum
     *
     * @throws Exception
     */
    @Test
    public void search30() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder();

        //条件搜索
        builder.query(QueryBuilders.matchAllQuery());
        //聚合查询
        AggregationBuilder aggregation = AggregationBuilders.sum("sum_age").field("age");
        builder.aggregation(aggregation);

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * 聚合查询 avg
     *
     * @throws Exception
     */
    @Test
    public void search31() throws Exception {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder();

        //条件搜索
        builder.query(QueryBuilders.matchAllQuery());
        //聚合查询
        AggregationBuilder aggregation = AggregationBuilders.avg("avg_age").field("age");
        builder.aggregation(aggregation);

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }


    /**
     * 聚合查询 count
     *
     * @throws IOException
     */
    @Test
    public void search32() throws IOException {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder();

        //条件搜索
        builder.query(QueryBuilders.matchAllQuery());
        //聚合查询
        AggregationBuilder aggregation = AggregationBuilders.count("count_age").field("age");
        builder.aggregation(aggregation);

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }

    /**
     * 聚合查询 分组
     *
     * @throws IOException
     */
    @Test
    public void search33() throws IOException {
        // 创建请求
        SearchSourceBuilder builder = new SearchSourceBuilder();

        //条件搜索
        builder.query(QueryBuilders.matchAllQuery());
        //聚合查询
        AggregationBuilder aggregation = AggregationBuilders.terms("tag_createTime").field("createTime")
                .subAggregation(AggregationBuilders.count("count_age").field("age")) //计数
                .subAggregation(AggregationBuilders.sum("sum_age").field("age")) //求和
                .subAggregation(AggregationBuilders.avg("avg_age").field("age")); //求平均值

        builder.aggregation(aggregation);

        //不输出原始数据
        builder.size(0);

        //搜索
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(segmentIndex);
        searchRequest.source(builder);
        // 执行请求
        SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 解析查询结果
        System.out.println(response.toString());
    }

}
```

## SpringBoot操作ES索引
ES的高级客户端中，提供了一个`indices()`方法，这个方法可以获取一个专门用于操作index索引的API。
```java
// 获取索引客户端
IndicesClient indicesClient = restHighLevelClient.indices();
```

### 创建索引
ES创建索引，需要创建一个索引的请求对象【CreateIndexRequest】，创建索引的时候，需要指定索引的名称，可以通过构造方法指定索引名称，并且ES中的每一个API几乎都提供了一个RequestOptions配置项参数，通过这个参数可以设置这次HTTP请求的相关参数。如果不想设置，也可以采用默认的配置项，只需要通过【RequestOptions.DEFAULT】获取即可。
**注意：当索引已经存在的时候，再次创建索引就会抛出一个异常。**

### 删除索引
ES删除索引，可以通过【DeleteIndexRequest】请求指定要删除的索引名称，可以同时删除多个索引。
**注意：当索引不存在的时候，删除不存在的索引会抛出异常。**

### 获取索引
获取索引需要创建一个【GetIndexRequest】对象，该对象构造方法可以传递需要查询的索引名称，可以传递多个索引名称。
**注意：如果查询的索引不存在，那么就会抛出一个索引不存在的异常ElasticsearchStatusException。**

### 判断索引是否存在
上面查询索引的时候，如果索引不存在，就会抛出异常，为了避免出现异常，我们可以首先判断一下索引是否存在，如果存在则继续执行相关操作，否则创建新索引、或者给出提示信息之类的。 可以通过调用【exists()】方法判断某个索引是否存在，该方法返回的是一个boolean类型的值。
**注意：当判断多个索引的时候，所有的索引都存在，才会返回true，否则返回false。**

## SpringBoot ES 索引实战
```java

import com.alibaba.fastjson.JSON;
import org.elasticsearch.action.admin.indices.delete.DeleteIndexRequest;
import org.elasticsearch.action.bulk.BulkRequest;
import org.elasticsearch.action.bulk.BulkResponse;
import org.elasticsearch.action.delete.DeleteRequest;
import org.elasticsearch.action.delete.DeleteResponse;
import org.elasticsearch.action.get.GetRequest;
import org.elasticsearch.action.get.GetResponse;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.index.IndexResponse;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.support.master.AcknowledgedResponse;
import org.elasticsearch.action.update.UpdateRequest;
import org.elasticsearch.action.update.UpdateResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.client.indices.CreateIndexRequest;
import org.elasticsearch.client.indices.CreateIndexResponse;
import org.elasticsearch.client.indices.GetIndexRequest;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.common.xcontent.XContentType;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.rest.RestStatus;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.elasticsearch.search.fetch.subphase.FetchSourceContext;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Component;
import java.io.IOException;
import java.util.List;
import java.util.concurrent.TimeUnit;
@Component
public class EsUtils<T> {
    @Autowired
    @Qualifier("restHighLevelClient")
    private RestHighLevelClient client;
    /**
     * 判断索引是否存在
     *
     * @param index
     * @return
     * @throws IOException
     */
    public boolean existsIndex(String index) throws IOException {
        GetIndexRequest request = new GetIndexRequest(index);
        boolean exists = client.indices().exists(request, RequestOptions.DEFAULT);
        return exists;
    }
    /**
     * 创建索引
     *
     * @param index
     * @throws IOException
     */
    public boolean createIndex(String index) throws IOException {
        CreateIndexRequest request = new CreateIndexRequest(index);
        CreateIndexResponse createIndexResponse = client.indices()
                .create(request, RequestOptions.DEFAULT);
        return createIndexResponse.isAcknowledged();
    }
    /**
     * 删除索引
     *
     * @param index
     * @return
     * @throws IOException
     */
    public boolean deleteIndex(String index) throws IOException {
        DeleteIndexRequest deleteIndexRequest = new DeleteIndexRequest(index);
        AcknowledgedResponse response = client.indices()
                .delete(deleteIndexRequest, RequestOptions.DEFAULT);
        return response.isAcknowledged();
    }
    /**
     * 判断某索引下文档id是否存在
     *
     * @param index
     * @param id
     * @return
     * @throws IOException
     */
    public boolean docExists(String index, String id) throws IOException {
        GetRequest getRequest = new GetRequest(index, id);
        //只判断索引是否存在不需要获取_source
        getRequest.fetchSourceContext(new FetchSourceContext(false));
        getRequest.storedFields("_none_");
        boolean exists = client.exists(getRequest, RequestOptions.DEFAULT);
        return exists;
    }
    /**
     * 添加文档记录
     *
     * @param index
     * @param id
     * @param t 要添加的数据实体类
     * @return
     * @throws IOException
     */
    public boolean addDoc(String index, String id, T t) throws IOException {
        IndexRequest request = new IndexRequest(index);
        request.id(id);
        //timeout
        request.timeout(TimeValue.timeValueSeconds(1));
        request.timeout("1s");
        request.source(JSON.toJSONString(t), XContentType.JSON);
        IndexResponse indexResponse = client.index(request, RequestOptions.DEFAULT);
        RestStatus Status = indexResponse.status();
        return Status == RestStatus.OK || Status == RestStatus.CREATED;
    }
    /**
     * 根据id来获取记录
     *
     * @param index
     * @param id
     * @return
     * @throws IOException
     */
    public GetResponse getDoc(String index, String id) throws IOException {
        GetRequest request = new GetRequest(index, id);
        GetResponse getResponse = client.get(request,RequestOptions.DEFAULT);
        return getResponse;
    }
    /**
     * 批量添加文档记录
     * 没有设置id ES会自动生成一个，如果要设置 IndexRequest的对象.id()即可
     *
     * @param index
     * @param list
     * @return
     * @throws IOException
     */
    public boolean bulkAdd(String index, List<T> list) throws IOException {
        BulkRequest bulkRequest = new BulkRequest();
        //timeout
        bulkRequest.timeout(TimeValue.timeValueMinutes(2));
        bulkRequest.timeout("2m");
        for (int i = 0; i < list.size(); i++) {
            bulkRequest.add(new IndexRequest(index).source(JSON.toJSONString(list.get(i))));
        }
        BulkResponse bulkResponse = client.bulk(bulkRequest,RequestOptions.DEFAULT);
        return !bulkResponse.hasFailures();
    }
    /**
     * 更新文档记录
     * @param index
     * @param id
     * @param t
     * @return
     * @throws IOException
     */
    public boolean updateDoc(String index, String id, T t) throws IOException {
        UpdateRequest request = new UpdateRequest(index, id);
        request.doc(JSON.toJSONString(t));
        request.timeout(TimeValue.timeValueSeconds(1));
        request.timeout("1s");
        UpdateResponse updateResponse = client.update(request, RequestOptions.DEFAULT);
        return updateResponse.status() == RestStatus.OK;
    }
    /**
     * 删除文档记录
     *
     * @param index
     * @param id
     * @return
     * @throws IOException
     */
    public boolean deleteDoc(String index, String id) throws IOException {
        DeleteRequest request = new DeleteRequest(index, id);
        //timeout
        request.timeout(TimeValue.timeValueSeconds(1));
        request.timeout("1s");
        DeleteResponse deleteResponse = client.delete(request, RequestOptions.DEFAULT);
        return deleteResponse.status() == RestStatus.OK;
    }
    /**
     * 根据某字段来搜索
     *
     * @param index
     * @param field
     * @param key   要收搜的关键字
     * @throws IOException
     */
    public void search(String index, String field, String key, Integer
            from, Integer size) throws IOException {
        SearchRequest searchRequest = new SearchRequest(index);
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.query(QueryBuilders.termQuery(field, key));
        //控制搜素
        sourceBuilder.from(from);
        sourceBuilder.size(size);
        //最大搜索时间。
        sourceBuilder.timeout(new TimeValue(60, TimeUnit.SECONDS));
        searchRequest.source(sourceBuilder);
        SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);
        System.out.println(JSON.toJSONString(searchResponse.getHits()));
    }
}
```