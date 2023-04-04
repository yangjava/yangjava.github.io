---
layout: post
categories: ElasticSearch
description: none
keywords: ElasticSearch
---
# ElasticSearch客户端实战
想要使用Elasticsearch服务，则要先获取一个Elasticsearch客户端。获取Elasticsearch客户端的方法很简单，最常见的就是创建一个可以连接到集群的传输客户端对象。

## ElasticSearch客户端简介
在Elasticsearch中，客户端有初级客户端和高级客户端两种。它们均使用Elasticsearch提供了RESTful风格的API，因此，本书中的客户端API使用也以RESTful风格的API为主。在使用RESTful API时，一般通过9200端口与Elasticsearch进行通信。

初级客户端是Elasticsearch为用户提供的官方版初级客户端。初级客户端允许通过HTTP与Elasticsearch集群进行通信，它将请求封装发给Elasticsearch集群，将Elasticsearch集群的响应封装返回给用户。初级客户端与所有Elasticsearch版本都兼容。

高级客户端是用于弹性搜索的高级客户端，它基于初级客户端。高级客户端公开了API特定的方法，并负责处理未编组的请求和响应。

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
```java
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











