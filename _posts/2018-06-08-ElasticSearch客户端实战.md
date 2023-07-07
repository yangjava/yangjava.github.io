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

## ElasticSearch客户端简介
在Elasticsearch中，客户端有初级客户端和高级客户端两种。它们均使用Elasticsearch提供了RESTful风格的API。在使用RESTful API时，一般通过9200端口与Elasticsearch进行通信。

初级客户端是Elasticsearch为用户提供的官方版初级客户端。初级客户端允许通过HTTP与Elasticsearch集群进行通信，它将请求封装发给Elasticsearch集群，将Elasticsearch集群的响应封装返回给用户。初级客户端与所有Elasticsearch版本都兼容。

高级客户端是用于弹性搜索的高级客户端，它基于初级客户端。高级客户端公开了API特定的方法，并负责处理未编组的请求和响应。

## Java整合ES

### Maven依赖
```xml
    <properties>
    <java.version>1.8</java.version>
    <maven.compiler.source>${java.version}</maven.compiler.source>
    <maven.compiler.target>${java.version}</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <lombok.version>1.18.16</lombok.version>
    <spring-boot-starter-parent.version>2.4.6</spring-boot-starter-parent.version>
</properties>

<dependencies>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
</dependencies>

<dependencyManagement>
<dependencies>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${lombok.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>${spring-boot-starter-parent.version}</version>
        <type>pom</type>
        <scope>import</scope>
    </dependency>
</dependencies>
</dependencyManagement>

<build>
<plugins>
    <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <version>2.0.1.RELEASE</version>
        <executions>
            <execution>
                <goals>
                    <goal>repackage</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
</plugins>
</build>
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

### 初始化Es配置添加密码
```

```

## 初级客户端功能
在介绍如何使用初级客户端之前，我们先要了解初级客户端的主要功能，其主要功能包括：
- 跨所有可用节点的负载平衡。
- 在节点故障和特定响应代码时的故障转移。
- 失败连接的惩罚机制。判断一个失败节点是否重试，取决于客户端连接时连续失败的次数；失败的尝试次数越多，客户端再次尝试同一节点之前等待的时间越长。
- 持久连接。
- 请求和响应的跟踪日志记录。
- 自动发现群集节点，该功能可选。

## 获取初级客户端
初级客户端保持了与Elasticsearch相同的发布周期。而客户端的版本和客户端可以通信的Elasticsearch的版本之间没有关系，即初级客户端可以与所有Elasticsearch版本兼容。
Maven依赖仓库如下:
```xml
    <dependencies>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client</artifactId>
            <version>7.12.1</version>
        </dependency>
    </dependencies>
```

## 客户端初始化
用户可以通过相应的RestClientBuilder类来构建RestClient实例，该类是通过RestClient builder（HttpHost…）静态方法创建的。

RestClient在初始化时，唯一需要的参数是客户端将与之通信的一个或多个主机作为HttpHost实例提供的，如下所示：
```java
import org.apache.http.HttpHost;
import org.elasticsearch.client.RestClient;

public class RestClientDemo {
    public RestClient getRestClient() {
        return RestClient.builder(
                new HttpHost("localhost", 9200, "http"),
                new HttpHost("localhost", 9200, "http")
        ).build();
    }
}
```
对于RestClient类而言，RestClient类是线程安全的。在理想情况下，它与使用它的应用程序具有相同的生命周期。因此，当不再需要时，应该关闭它，以便释放它使用的所有资源及底层HTTP客户机实例及其线程，这一点很重要。关闭的方法如下所示：
```
restClient.close();
```
RestClientBuilder允许在构建RestClient时选择性地设置以下配置参数。
- 请求头配置方法。
- 配置监听器。
- 配置节点选择器。
- 设置超时时间。

## 提交请求
在创建好客户端之后，执行请求之前还需构建请求对象。用户可以通过调用客户端的performRequest和performRequestAsync方法来发送请求。

其中，performRequest是同步请求方法，它将阻塞调用线程，并在请求成功时返回响应，或者在请求失败时引发异常。

而performRequestAsync是异步方法，它接收一个ResponseListener对象作为参数。如果请求成功，则该参数使用响应进行调用；如果请求失败，则使用异常进行调用。

### 构建请求对象Request
请求对象Request的请求方式与The HTTP的请求方式相同，如GET、POST、HEAD等，代码如下所示：

### 请求的执行
在构建请求（Request）后，即可执行请求。请求有同步执行和异步执行两种方式。下面将分别展示如何使用performRequest方法和performRequestAsync方法发送请求。
- 同步方式
  当以同步方式执行Request时，客户端会等待Elasticsearch服务器返回的查询结果Response。在收到Response后，客户端继续执行相关的逻辑代码。
- 异步方式
  当以异步方式执行请求时，初级客户端不必同步等待请求结果的返回，可以直接向接口调用方返回异步接口执行成功的结果。为了处理异步返回的响应信息或处理在请求执行过程中引发的异常信息，用户需要指定监听器。
  以异步方式调用的代码如下所示。

### 可选参数配置
不论同步请求，还是异步请求，我们都可以在请求中添加参数，添加方法如下所示：

### 多个并行异步操作
除单个操作的执行外，Elasticsearch的客户端还可以并行执行许多操作。下面通过ServiceImpl实现层MeetElasticSearchServiceImpl类中的示例，展示如何对多文档进行并行索引。代码如下：

## 对请求结果的解析
请求对象有两种请求方式，分别是同步请求和异步请求，因此对于请求的响应结果Response的解析也分为两种。

同步请求得到的响应对象是由performRequest方法返回的；而异步请求得到的响应对象是通过ResponseListener类下onSuccess（Response）方法中的参数接收的。响应对象中包装HTTP客户端返回的响应对象，并公开一些附加信息。

下面通过代码学习对请求结果的解析。以同步请求方式为例，对请求结果的解析代码如下所示。

## 常见通用设置
除上述客户端API外，客户端还支持一些常见通用设置，如超时设置、线程数设置、节点选择器设置和配置嗅探器等。

### 超时设置
我们可以在构建RestClient时提供requestconfigCallback的实例来完成超时设置。该接口有一个方法，它接收org.apache.http.client.config.requestconfig.builder的实例作为参数，并且具有相同的返回类型。

用户可以修改请求配置生成器org.apache.http.client.config.requestconfig.builder的实例，然后返回。

在下面的示例中，增加了连接超时（默认为1s）和套接字超时（默认为30s），代码如下所示：


在具体使用时，详见MeetElasticSearchServiceImpl中的代码，部分代码如下所示：


### 线程数设置

Apache HTTP异步客户端默认启动一个调度程序线程，连接管理器使用的多个工作线程。一般线程数与本地检测到的处理器数量相同，线程数主要取决于Runtime.getRuntime（）.availableProcessors（）返回的结果。

Elasticsearch允许用户修改线程数，修改代码如下所示，详见MeetElasticSearchServiceImpl类：


### 节点选择器设置
在默认情况下，客户端会以轮询的方式将每个请求发送到配置的各个节点中。

Elasticsearch允许用户自由选择需要连接的节点。一般通过初始化客户端来配置节点选择器，以便筛选节点。

该功能在启用嗅探器时很有用，以防止HTTP请求只命中专用的主节点。

配置后，对于每个请求，客户端都通过节点选择器来筛选备选节点。

代码如下所示，详见MeetElasticSearchServiceImpl类：



### 配置嗅探器
嗅探器允许自动发现运行中的Elasticsearch集群中的节点，并将其设置为现有的RestClient实例。

在默认情况下，嗅探器使用nodes info API检索属于集群的节点，并使用jackson解析获得的JSON响应。

目前，嗅探器与Elasticsearch 2.X及更高版本兼容。

在使用嗅探器之前需添加相关的依赖，代码如下所示：


在创建好RestClient实例（如初始化中代码所示）后，就可以将嗅探器与其进行关联了。嗅探器利用RestClient提供的定期机制（在默认情况下定期时间为5min），从集群中获取当前节点的列表，并通过调用RestClient类中的setNodes方法来更新它们。

嗅探器的使用代码详见ServiceImpl实现层的MeetElasticSearchServiceImpl类，部分代码如下所示：


当然，除在客户端启动时配置嗅探器外，还可以在失败时启用嗅探器。这意味着在每次失败后，节点列表都会立即更新，而不是在接下来的普通嗅探循环中更新。

在这种情况下，首先需要创建一个SniffOnFailureListener，然后在创建RestClient时配置。在创建嗅探器后，同一个SniffOnFailureListener实例会相互关联，以便在每次失败时都通知该实例，并使用嗅探器执行嗅探动作。

嗅探器SniffOnFailureListener的使用代码详见ServiceImpl实现层的MeetElasticSearchServiceImpl类，部分代码如下所示：



由于Elasticsearch节点信息API不会返回连接到节点时要使用的协议，而是只返回它们的host：port，因此在默认情况下会使用HTTP。如果需要使用HTTPS，则必须手动创建并提供ElasticSearchNodesNiffer实例，相关代码如下所示：


## 高级客户端初始化
目前，官方计划在Elasticsearch 7.0版本中关闭TransportClient，并且在8.0版本中完全删除TransportClient。作为替代品，我们应该使用高级客户端。高级客户端可以执行HTTP请求，而不是序列化Java请求。

高级客户端基于初级客户端来实现。

高级客户端的主要目标是公开特定的API方法，这些API方法将接收请求作为参数并返回响应结果，以便由客户端本身处理请求和响应结果。

与初级客户端一样，高级客户端也有同步、异步两种API调用方式。其中，以同步调用方式调用后直接返回响应对象，而异步调用方式则需要配置一个监听器参数才能调用，该参数在收到响应或错误后会得到相应的结果通知。

高级客户端需要Java 1.8及其以上的JDK环境，而且依赖于Elasticsearch core项目，它接收与TransportClient相同的请求参数，并返回相同的响应对象。这其实也是为了提高兼容性而做的设计。

高级客户端的版本与Elasticsearch版本同步。高级客户端能够与运行着相同主版本和更高版本上的任何Elasticsearch节点进行有效通信。高级客户端无须与它通信的Elasticsearch节点处于同一个小版本，这是因为向前兼容设计的缘故。这也意味着高级客户端支持与Elasticsearch的较新版本进行有效通信。

举例来说，6.0版本的客户端可以与任何6.X版本的Elasticsearch节点进行通信，而6.1版本的客户端可以确保与6.1版本、6.2版本和任何更高版本的6.X节点进行通信，但在与低于6.0版本的Elasticsearch节点进行通信时可能会出现不兼容问题。

下面介绍高级客户端的使用，首先介绍高级客户端的初始化方法。

### 如何使用JSON进行查询
```
SearchRequest searchRequest = new SearchRequest("indexName");
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder().query(QueryBuilders.wrapperQuery("your json goes here"));
searchRequest.source(searchSourceBuilder);
```

```
    public void queryJSON1() throws Exception {
        SearchRequest searchRequest = new SearchRequest("all-company-online");
        String query = "{}";
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        SearchModule searchModule = new SearchModule(Settings.EMPTY, false, Collections.emptyList());
        NamedXContentRegistry namedXContentRegistry = new NamedXContentRegistry(searchModule.getNamedXContents());

        try (XContentParser parser = XContentFactory.xContent(XContentType.JSON).createParser(new NamedXContentRegistry(searchModule.getNamedXContents()), null, query)) {
            searchSourceBuilder.parseXContent(parser);
        }
        searchRequest.source(searchSourceBuilder);
        SearchResponse search = searchClient.search(searchRequest, RequestOptions.DEFAULT);
        log.info("search:{}", search);
    }
```

```
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.xcontent.NamedXContentRegistry;
import org.elasticsearch.search.SearchModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Collections;

@Configuration
public class NamedXContentRegistryConfig {

    @Bean
    public NamedXContentRegistry namedXContentRegistry() {
        SearchModule searchModule = new SearchModule(Settings.EMPTY, false, Collections.emptyList());
        return new NamedXContentRegistry(searchModule.getNamedXContents());
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

## 获取所有索引
```java
        GetAliasesRequest request = new GetAliasesRequest();
        GetAliasesResponse getAliasesResponse =  restHighLevelClient.indices().getAlias(request,RequestOptions.DEFAULT);
```