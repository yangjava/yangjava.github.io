---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码Get操作
官网对ES定义：Elasticsearch is the distributed search and analytics engine。

（1）Search核心包含两种情况：

a. 结构化查询，即精确值匹配。需要提供三元组：index、_type(ES7废弃，默认是_doc) 、_id，根据文档 id 从正排索引中获取内容；

b. 全文索引，根据关键词从倒排索引中搜索，并根据相关性得分进行排序；

（2）Analytics，注意是指聚合能力，例如分桶聚合(Bucket aggregations)、指标聚合(Metrics aggregations)等。

本文主要讨论，机构和查询相关的Get和MGet操作。官网Java High Level REST Client对应的API如下：

## Get API
与其他请求一样，Get操作首先构建GetRequest，并对参数赋值。
```
GetRequest getRequest = new GetRequest(
        "posts", //Index名称
        "1");    //Doucment 对于ID
```

### 参数
（1）Description_source 是否返回

request.fetchSourceContext(FetchSourceContext.DO_NOT_FETCH_SOURCE);

DO_NOT_FETCH_SOURCE：服务端不返回_source；

FETCH_SOURCE：服务端返回_source（默认值）；

（2）Source filtering

返回参数可以设置过滤，例如：包含某些字段或不包含某些字段：
```
String[] includes = new String[]{"message", "*Date"};
String[] excludes = Strings.EMPTY_ARRAY;
FetchSourceContext fetchSourceContext =
        new FetchSourceContext(true, includes, excludes);
request.fetchSourceContext(fetchSourceContext);
```
（3）Realtime

默认值Realtime属性为true，ES是近实时的特性，写入Document未形成Segment是不支持搜索的。若文档已经写入，默认会进行refresh使得文档可见支持搜索。
```
request.refresh(true);
```
（3）Routing
路由：request.routing("routing")；写入时指定route参数值，那么搜索的时候也需要指定（除非特殊情况，不建议设置）

（4）Preference

Document写入时是写入主分片，然后主分片转发到副本分片。搜索时，主分片和副本分片都支持搜索，如果指定Preference指定主分片或者从本地读取。

（5）Version

version拥有并发版本控制，如果设置version不存在，则返回错误。

（6）VersionType

取值是个枚举值：internal,external,external_gte,force。（5）Version type对于概念一致。
```
request.opType(DocWriteRequest.OpType.CREATE);
```
（7）stored_fields

如果设置stored_fields，则返回存储的fields（需要Mappings设置），而不是_source，默认值false。

## 执行
（1）同步执行

同步执行，可能抛出4xxor5xx等错误代码，ElasticsearchException 或ResponseException等异常：
```
GetResponse getResponse = client.get(getRequest, RequestOptions.DEFAULT);
```
（2）异步执行

异步执行，并不会阻塞当前执行，需要设置监听器监听返回结果：
```
client.getAsync(request, RequestOptions.DEFAULT, listener);
--------------------------------------------------------
ActionListener<GetResponse> listener = new ActionListener<GetResponse>() {
    @Override
    public void onResponse(GetResponse getResponse) {
        
    }

    @Override
    public void onFailure(Exception e) {
        
    }
};
```
## 响应结果
Java 代码实现处理响应结果的逻辑：
```
String index = getResponse.getIndex();
String id = getResponse.getId();
if (getResponse.isExists()) {
    long version = getResponse.getVersion();
    //获取Document（String类型）
    String sourceAsString = getResponse.getSourceAsString(); 
    //获取Document（Map类型）       
    Map<String, Object> sourceAsMap = getResponse.getSourceAsMap(); 
    //获取Document（Byte[]类型）
    byte[] sourceAsBytes = getResponse.getSourceAsBytes();          
} else {
    //异常处理
}
```
按照Restful格式响应内容：
```
{
    "_index" : "twitter",
    "_type" : "_doc",
    "_id" : "0",
    "_version" : 1,
    "_seq_no" : 10,
    "_primary_term" : 1,
    "found": true,
    "_source" : {
        "user" : "kimchy",
        "date" : "2009-11-15T14:12:12",
        "likes": 0,
        "message" : "trying out Elasticsearch"
    }
}
```
（1）_id，当前Document的唯一标识；

（2）_version 版本号，当Document有Update操作时，都会新增；

（3）_seq_no针对当前Index操作分配值，用于乐观锁版本并发控制；

（4） _primary_term 针对当前Index操作Primary Shard的term， 用于乐观锁版本并发控制；

（5）_source 默认以Json形式返回Doucment。如果设置false或设置sorted_field则不返回。

（6）_fields 如果设置stored_fields，则返回存储的fields（需要Mappings设置），而不是_source。

## Multi get
Multi Get API首先构建MultiGetRequest，其中对于的Item和Get API单个请求保持一致，官网示例如下：
```
MultiGetRequest request = new MultiGetRequest();
request.add(new MultiGetRequest.Item(
    "index",         
    "example_id"));  
request.add(new MultiGetRequest.Item("index", "another_id"));
```
请求参数的设置，每个Item都可以设置独立的参数值。

请求同样划分为同步/异步两种形式。

请求成功和请求失败都会返回结果，需要客户端处理。

## Get/MGet处理流程

### Get处理流程图
官网示例取回单个文档，可以从主分片或者从其它任意副本分片检索，返回一个文档示意图：

从主分片或者副本分片检索文档的步骤顺序：

1、客户端向 Node 1 发送获取请求。

2、节点使用文档的 _id 来确定文档属于分片 0 。分片 0 的副本分片存在于所有的三个节点上。 在这种情况下，它将请求转发到 Node 2 。

3、Node 2 将文档返回给 Node 1 ，然后将文档返回给客户端。

在处理读取请求时，协调结点在每次请求的时候都会通过轮询所有的副本分片来达到负载均衡。

在文档被检索时，已经被索引的文档可能已经存在于主分片上但是还没有复制到副本分片。 在这种情况下，副本分片可能会报告文档不存在，但是主分片可能成功返回文档。 一旦索引请求成功返回给用户，文档在主分片和副本分片都是可用的。

### Get流程图
Get操作流程图如下：Get操作流程是，client发起请求，Server端响应请求并处理，最后返回结果。

区别：如创建/删除索引等操作只能在Master节点执行，在Transport**Action层对应TransportMasterNodeAction，其他则操作操作则没有这个要求；

（1）Rest层请求构建

在RestGetAction对应的Rest层操作，在prepareRequest方法内需要构建对应的GetRequest请求。
```
public RestChannelConsumer prepareRequest(final RestRequest request, final NodeClient client) throws IOException {
    //请求体构建
    GetRequest getRequest;
    if (request.hasParam("type")) {
        deprecationLogger.deprecatedAndMaybeLog("get_with_types", TYPES_DEPRECATION_MESSAGE);
        getRequest = new GetRequest(request.param("index"), request.param("type"), request.param("id"));
    } else {
        getRequest = new GetRequest(request.param("index"), request.param("id"));
    }

    //参数设置，详见客户端设置参赛值
    getRequest.refresh(request.paramAsBoolean("refresh", getRequest.refresh()));
    getRequest.routing(request.param("routing"));
    getRequest.preference(request.param("preference"));
    getRequest.realtime(request.paramAsBoolean("realtime", getRequest.realtime()));
    if (request.param("fields") != null) {
        throw new IllegalArgumentException("the parameter [fields] is no longer supported, " +
            "please use [stored_fields] to retrieve stored fields or [_source] to load the field from _source");
    }
    //需要排序字段
    final String fieldsParam = request.param("stored_fields");
    if (fieldsParam != null) {
        final String[] fields = Strings.splitStringByCommaToArray(fieldsParam);
        if (fields != null) {
            getRequest.storedFields(fields);
        }
    }

    getRequest.version(RestActions.parseVersion(request));
    getRequest.versionType(VersionType.fromString(request.param("version_type"), getRequest.versionType()));

    getRequest.fetchSourceContext(FetchSourceContext.parseFromRestRequest(request));
    //消费者构建
    return channel -> client.get(getRequest, new RestToXContentListener<GetResponse>(channel) {
        @Override
        protected RestStatus getStatus(final GetResponse response) {
            return response.isExists() ? OK : NOT_FOUND;
        }
    });
}
```
（2）Transport层业务处理

AsyncSingleAction构造方法中最核心的逻辑就是获取索引分片迭代器-ShardsIterator，迭代器包含了该索引所有可用分片。
```
private AsyncSingleAction(Request request, ActionListener<Response> listener) {
    ClusterState clusterState = clusterService.state();
    //集群nodes列表
    nodes = clusterState.nodes();
    ClusterBlockException blockException = checkGlobalBlock(clusterState);
    if (blockException != null) {
        throw blockException;
    }

    String concreteSingleIndex;//concrete具体的
    if (resolveIndex(request)) {
        concreteSingleIndex = indexNameExpressionResolver.concreteSingleIndex(clusterState, request).getName();
    } else {
        concreteSingleIndex = request.index();
    }
    this.internalRequest = new InternalRequest(request, concreteSingleIndex);
    //解析请求，更新自定义routing
    resolveRequest(clusterState, internalRequest);
    //判断是否有禁止读操作
    blockException = checkRequestBlock(clusterState, internalRequest);
    if (blockException != null) {
        throw blockException;
    }

    //根据路由算法计算得到目的shard迭代器，或者根据优先级选择目标节点
    this.shardIt = shards(clusterState, internalRequest);
}
```
start方法内，判断是在本地协调者节点读取还是在远程数据节点读取。
```
public void start() {
    if (shardIt == null) {
        // just execute it on the local node
        final Writeable.Reader<Response> reader = getResponseReader();
        transportService.sendRequest(clusterService.localNode(), transportShardAction, internalRequest.request(),
    } else {
        perform(null);
    }
---------------------------------------------
private void perform(@Nullable final Exception currentFailure) {
    ........................
    final ShardRouting shardRouting = shardIt.nextOrNull();
    
    DiscoveryNode node = nodes.get(shardRouting.currentNodeId());
    if (node == null) {
        onFailure(shardRouting, new NoShardAvailableActionException(shardRouting.shardId()));
    } else {
        internalRequest.request().internalShardId = shardRouting.shardId();
        
        final Writeable.Reader<Response> reader = getResponseReader();
        //RPC服务,可能是当前节点也可能是远程节点
        //作为协调节点，向目标节点转发请求，或者目标是本地节点，直接读取数据
        transportService.sendRequest(node, transportShardAction, internalRequest.request(),
         ........................
```
## Get操作读取与RPC流程
当前协调者节点读取和通过RPC远程节点读取流程图如下：

（1）RPC处理器

在抽象类TransportSingleShardAction中，完成RPC的注册，TransportGetAction及TransportShardMultiGetAction对应的处理器是ShardTransportHandler。
```
protected TransportSingleShardAction(String actionName, ThreadPool threadPool, ClusterService clusterService,
    .....................
    //TransportGetAction:indices:data/read/get
    //TransportShardMultiGetAction: indices:data/read/mget/shard
    this.transportShardAction = actionName + "[s]";
    this.executor = executor;

    if (!isSubAction()) {
        //MainAction(不包含TransportShardMultiGetAction和TransportShardMultiTermsVectorAction)
        //TransportGetAction对应注册信息,远程和本地代码都执行TransportHandler
        transportService.registerRequestHandler(actionName, ThreadPool.Names.SAME, request, new TransportHandler());
    }
    //TransportGetAction及TransportShardMultiGetAction对应注册信息
    transportService.registerRequestHandler(transportShardAction, ThreadPool.Names.SAME, request, new ShardTransportHandler());
}
```
（2）当前节点读取

数据分片在当前节点，则通过分片数据读取模块获取数据，对应的处理逻辑在TransportService。
```
private void sendLocalRequest(long requestId, final String action, final TransportRequest request, TransportRequestOptions options) {
    //构建ResponseChannel后续用来发送Response信息给客户端
    final DirectResponseChannel channel = new DirectResponseChannel(localNode, action, requestId, this, threadPool);
    try {
        onRequestSent(localNode, requestId, action, request, options);
        onRequestReceived(requestId, action);
        //获取当前action对应的RequestHandlerRegistry处理类
        final RequestHandlerRegistry reg = getRequestHandler(action);
        if (reg == null) {
            throw new ActionNotFoundTransportException("Action [" + action + "] not found");
        }
        final String executor = reg.getExecutor();
        //sendRequest会检查目的节点是否是本节点，
        // 如果是本节点，则在sendLocalRequest方法中直接通过action 获取对应的Handler,调用对应的处理函数。
        if (ThreadPool.Names.SAME.equals(executor)) {
            //noinspection unchecked
            //使用当前线程处理
            reg.processMessageReceived(request, channel);
        } else {
            //创建新的线程处理
            threadPool.executor(executor).execute(new AbstractRunnable() {
                @Override
                protected void doRun() throws Exception {
                    //noinspection unchecked
                    reg.processMessageReceived(request, channel);
                }
    -------------------------------------
    private class ShardTransportHandler implements TransportRequestHandler<Request> {
        @Override
        public void messageReceived(final Request request, final TransportChannel channel, Task task) throws Exception {
            asyncShardOperation(request, request.internalShardId, new ChannelActionListener<>(channel, transportShardAction, request));
        }	
```
（3）远程节点读取

数据分片在其它节点，通过Netty远程通信模块，将请求转发至分片所在节点，再通过分片数据读取模块获取数据返回给协调节点。

在远程节点读取的处理器也是ShardTransportHandler，同意执行读取逻辑也是在TransportGetAction。显然无论是当前节点和远程节点，最终读取的逻辑都在TransportGetAction类的asyncShardOperation方法。

（4）asyncShardOperation
```
protected void asyncShardOperation(GetRequest request, ShardId shardId, ActionListener<GetResponse> listener) throws IOException {
    IndexService indexService = indicesService.indexServiceSafe(shardId.getIndex());
    //获取数据所在目标分片
    IndexShard indexShard = indexService.getShard(shardId.id());
    if (request.realtime()) { // we are not tied to a refresh cycle here anyway
        //实时执行，即数据未刷盘也可从 translog 读取
        super.asyncShardOperation(request, shardId, listener);
    } else {
        //等待数据刷入内核缓冲区后读取(默认刷新间隔为1s)
        indexShard.awaitShardSearchActive(b -> {
            try {
                super.asyncShardOperation(request, shardId, listener);
            } catch (Exception ex) {
                listener.onFailure(ex);
            }
        });
    }
}
```
super.asyncShardOperation通过在TransportSingleShardAction类内实现线程池异步操作，然后调用TransportGetAction类的shardOperation方法：
```
protected GetResponse shardOperation(GetRequest request, ShardId shardId) {
    IndexService indexService = indicesService.indexServiceSafe(shardId.getIndex());
    IndexShard indexShard = indexService.getShard(shardId.id());

    //request.refresh()表示是否在读取前执行刷新操作
    if (request.refresh() && !request.realtime()) {
        indexShard.refresh("refresh_flag_get");
    }

    //获取分片数据
    GetResult result = indexShard.getService().get(request.type(), request.id(), request.storedFields(),
            request.realtime(), request.version(), request.versionType(), request.fetchSourceContext());
    return new GetResponse(result);
}
```
接下来通过IndexShard进行读取

## MGet处理流程与对比
MGet操作对应流程图如下：和Get操作一样，数据读取流程区分为当前协调者节点读取和Data节点读取：

MGet操作与Get操作区别最大的地方在Transport层，对应TransportMultiGetAction类。先遍历请求构建MultiGetShardRequest。
```
 for (int i = 0; i < request.items.size(); i++) {
       MultiGetRequest.Item item = request.items.get(i);
```
然后遍历MultiGetShardRequest，执行读取请求：
```
for (final MultiGetShardRequest shardRequest : shardRequests.values()) {
    //TransportShardMultiGetAction:shardAction
    shardAction.execute(shardRequest, new ActionListener<MultiGetShardResponse>() {
```
然后通过TransportSingleShardAction的异步线程池

TransportSingleShardAction在Get操作具体实现类是TransportGetAction，MGet操作对应的具体实现类是TransportShardMultiGetAction，对应的类图关系如下图。TransportSingleShardAction是单个shard读取操作的抽象类，还有很多其他实现类，这里不再展开。

注意
GET API默认是实时的，实时的意思是写完了可以立刻读取，但仅限于GET、MGET操作，不包括搜索。

在5.x版本之前，GET/MGET的实时读取依赖于从translog中读取实现，5.x 版本之后的版本改为refresh，因此系统对实时读取的支持会对写入速度有负面影响。