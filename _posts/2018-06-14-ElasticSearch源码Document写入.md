---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码Document写入
向ES数据写入一个Document对应的是Index API称为 Index 请求，批量写入Document对应的是Bulk API称为 Bulk 请求。写单个和多个文档使用相同的处理逻辑，请求被统一封装为BulkRequest。官网Java High Level REST Client对应的API如下：

说明：本文会有官方Java代码和Rest API两种实现的说明，有时候会混合。

Rest API：https://www.elastic.co/guide/en/elasticsearch/client/java-rest/7.5/java-rest-high.html

Java代码：Document APIs | Elasticsearch Guide [7.5] | Elastic

## Index API
Index API 会在特定索引中添加或更新一个 JSON 类型的文档，使其可供搜索。官网给出几个示例：
```
IndexRequest indexRequest = new IndexRequest("index-name")
    .id("1")
    .source(builder);  
```
官网示例的区别是source方法的重载函数，IndexRequest类source方法的重载函数不止这几个：
```
public IndexRequest source(Map<String, ?> source) throws ElasticsearchGenerationException {
    return source(source, Requests.INDEX_CONTENT_TYPE);
}

public IndexRequest source(Map<String, ?> source, XContentType contentType) throws ElasticsearchGenerationException {
    try {
        XContentBuilder builder = XContentFactory.contentBuilder(contentType);
        builder.map(source);
        return source(builder);
}

public IndexRequest source(String source, XContentType xContentType) {
    return source(new BytesArray(source), xContentType);
}
```

### 参数
（1）路由：request.routing("routing")；

路由计算方式为下面的公式如下，默认情况下，_routing值就是文档id。
```
shard_num = hash(_routing) % num_primary_shards
```
如果显示设置路由值，该文档会被写入到对应的shard。缺点：查询的时候也需要设置路由值，不然极有可能搜索不到，核心原因是路由公式对应的shard和显示设置的shard不是同一个。

（2）超时
```
request.timeout(TimeValue.timeValueSeconds(1)); 
request.timeout("1s");
```
由于Primary Shard异常（被关闭或Allocate等原因）造成执行Index的请求异常，写操作失败。默认时间是1分钟。

（3）refresh

控制当前请求的更改对搜索何时可见。官网给出的请求包含：Index,Update,Delete, andBulk APIs。
```
request.setRefreshPolicy(WriteRequest.RefreshPolicy.WAIT_UNTIL);
```
参数：

true：当前请求的更改对搜索立即可见，不再是近实时（仅对当前shard有效），这个需要验证谨慎使用；
false：默认值。ES近实时特性，数据refresh到PageCache后，能够搜索；
wait_for：针对当前请求，不强制执行refresh，而是在请求中等待refersh的执行（由参数index.refresh_interval控制）

（4）Version

指定 version 参数时，索引 API 可选择性地允许开放式并发控制。默认情况下，内部版本控制从 1 开始，随每次更新（包括删除）而增加，主要用于乐观锁并发控制。
```
request.version(2); 
```

（5）Version type

取值是个枚举值：internal,external,external_gte,force
```
request.opType(DocWriteRequest.OpType.CREATE); 
```
internal ：如果给定版本与存储文档的版本相同，则仅索引文档。
external：如果给定版本高于存储文档的版本或当前没有文档，则仅索引文档。给定版本将 用作新版本，并将与新文档一起存储。提供的版本必须是非负数长值数字。
external_gte ：如果给定版本等于或高于存储文档的版本，则仅索引文档。如果当前没有文档，操作也会成功。 给定版本将用作新版本，并将与新文档一起存储。提供的版本必须是非负数long类型数字。

external_gte 版本类型用于特殊用例，应谨慎使用。如果使用不当，可能会导致数据丢失。还有一个选项 force ，它可能会导致主碎片与副本碎片不一致，因此已弃用。

（6）opType
```
request.opType(DocWriteRequest.OpType.CREATE);
```
INDEX(0), 对Document进行索引，相同Id存在的Document，会进行替换；
CREATE(1),对Document进行索引，新增一个不存在的索引，相同Id存在的Document，写操作异常；
UPDATE(2),更新document；
DELETE(3);删除document；

（7）pipeline

指定事先创建好的Pipline名称
```
request.setPipeline("pipeline");
```
以上属性设置来自Java-High-Level-Rest-Client官网，接下来的属性设置来自官网：Rest-Index API 。

当前版本7.5.2 官方建议使用Java-High-Level-Rest-Client或Java-Low-Level-Rest-Client，在7.15版本之后，官方建议使用Java-Client或Java-Low-Level-Rest-Client。

（8）if_seq_no

只有当index的if_seq_no是该值时才会处理该文档，否则409 Version Conflict。乐观锁控制，因为index每处理一个文档就会if_seq_no+1
```
Params withIfSeqNo(long ifSeqNo) {
    if (ifSeqNo != SequenceNumbers.UNASSIGNED_SEQ_NO) {
        return putParam("if_seq_no", Long.toString(ifSeqNo));
    }
    return this;
}
```
（9）if_primary_term

只有当该文档所在的primary分片号是if_primary_term时才会处理该文档，否则409 Version Conflict。乐观锁控制。
```
Params withIfPrimaryTerm(long ifPrimaryTerm) {
    if (ifPrimaryTerm != SequenceNumbers.UNASSIGNED_PRIMARY_TERM) {
        return putParam("if_primary_term", Long.toString(ifPrimaryTerm));
    }
    return this;
}
```

（10）wait_for_active_shards

指定一定数量的副本可用是才能进行写入，否则进行重试直到超时。默认值是1，即主分片写入即可。这个属性会覆盖索引级别的设置index.write.wait_for_active_shards对应的值。

这个和Kafka 的ACK概念类似，ack = 0/1/-1和wait_for_active_shards的取值，都是在确定副本完成写操作的数量。

## 执行
同步执行，可能抛出4xxor5xx等错误代码，ElasticsearchException 或ResponseException等异常：
```
IndexResponse indexResponse = client.index(request, RequestOptions.DEFAULT);
```
异步执行，并不会阻塞当前执行，需要设置监听器监听返回结果：
```
client.indexAsync(request, RequestOptions.DEFAULT, listener); 
```
监听器的经典实现：
```
listener = new ActionListener<IndexResponse>() {
    @Override
    public void onResponse(IndexResponse indexResponse) {
        
    }

    @Override
    public void onFailure(Exception e) {
        
    }
};
```
## 响应结果
Java 代码实现处理响应结果的逻辑：
```
try {
    IndexResponse response = client.index(request, RequestOptions.DEFAULT);
    String index = indexResponse.getIndex();
    String id = indexResponse.getId();
    if (indexResponse.getResult() == DocWriteResponse.Result.CREATED) {
    
    } else if (indexResponse.getResult() == DocWriteResponse.Result.UPDATED) {
    
    }
    ReplicationResponse.ShardInfo shardInfo = indexResponse.getShardInfo();
    if (shardInfo.getTotal() != shardInfo.getSuccessful()) {
    
    }
    if (shardInfo.getFailed() > 0) {
        for (ReplicationResponse.ShardInfo.Failure failure :shardInfo.getFailures()) {
            String reason = failure.reason(); 
    }
  }
} catch(ElasticsearchException e) {
    if (e.status() == RestStatus.CONFLICT) {
        
    }
}
```
按照Restful格式响应内容：
```
{
    "_shards" : {
        "total" : 2,
        "failed" : 0,
        "successful" : 2
    },
    "_index" : "twitter",
    "_type" : "_doc",
    "_id" : "W0tpsmIBdwcYyG50zbta",
    "_version" : 1,
    "_seq_no" : 0,
    "_primary_term" : 1,
    "result": "created"
}
```
（1）_shard对应的是当前操作的信息，_shards.successful默认值是1，和wait_for_active_shards参数设置有关；

（2）_id，当前Document的唯一标识；

（3）_version 版本号，当Document有Update操作时，都会新增；

（4）_seq_no针对当前Index操作分配值，用于乐观锁版本并发控制；

（5） _primary_term 针对当前Index操作Primary Shard的term， 用于乐观锁版本并发控制；

（6）result对应两个枚举值，是 createdorupdated；

## Bulk API
Bulk API支持的操作是 index 、create 、delete 和update。Java API对应的示例：
```
BulkRequest request = new BulkRequest();
request.add(new DeleteRequest("posts", "3")); 
request.add(new UpdateRequest("posts", "2") 
        .doc(XContentType.JSON,"other", "test"));
request.add(new IndexRequest("posts").id("4")  
        .source(XContentType.JSON,"field", "baz"));
```
请求对应的参数和Index API 基本一致，执行的过程也有对应同步、异步两种方式，返回结果处理：
```
for (BulkItemResponse bulkItemResponse : bulkResponse) { 
    if (bulkItemResponse.isFailed()) { 
        BulkItemResponse.Failure failure =
                bulkItemResponse.getFailure(); 
    }
    DocWriteResponse itemResponse = bulkItemResponse.getResponse(); 

    switch (bulkItemResponse.getOpType()) {
    case INDEX:    
    case CREATE:
        IndexResponse indexResponse = (IndexResponse) itemResponse;
        break;
    case UPDATE:   
        UpdateResponse updateResponse = (UpdateResponse) itemResponse;
        break;
    case DELETE:   
        DeleteResponse deleteResponse = (DeleteResponse) itemResponse;
    }
}
```
官网还提供了BulkProcessor，简化了Bulk API的操作。

Rest API，查看docs-bulk即可。

## Index/Bulk处理流程
写操作一般会经过三种节点：协调节点、主分片所在节点、副本分片所在节点。

### Index/Bulk处理流程
（1）Index处理流程

1. 客户端向 Node 1 发送获取请求
2. 节点使用文档的 _id 来确定文档属于分片 0 。分片 0 的副本分片存在于所有的三个节点上。 在这种情况下，它将请求转发到 Node 2
3. Node 2 将文档返回给 Node 1 ，然后将文档返回给客户端

（2）Bulk处理流程
1. 客户端向 Node 1 发送 bulk 请求
2. Node 1 为每个节点创建一个批量请求，并将这些请求并行转发到每个包含主分片的节点主机
3. 主分片一个接一个按顺序执行每个操作。当每个操作成功时，主分片并行转发新文档（或删除）到副本分片，然后执行下一个操作。 一旦所有的副本分片报告所有操作成功，该节点将向协调节点报告成功，协调节点将这些响应收集整理并返回给客户端

写一致性：5.0后用wait_for_active_shards代替原来的Consistency（Quorum机制等）。

5.0前，采用写入前检查存活shard的方式，consistency有三个枚举值：one all quorum

## 写入流程图
单个文档写入流程图如下：

写入操作流程是，client发起请求，Server端响应请求并处理，最后返回结果。

区别：如创建/删除索引等操作只能在Master节点执行，在Transport**Action层对应TransportMasterNodeAction，其他则的操作则没有这个要求；

Index API和Bulk API的区别只是在Rest**Action层不同的业务，最终执行都是在TransportBulkAction内，从代码层面看Index API的操作是Bulk API的一部分。综上，大部分Client发起请求，ES服务端响应过程核心业务逻辑在Rest**Action和Transport**Action两个类。

## 协调节点流程
Index API和Bulk API在协调者节点的执行业务逻辑在TransportBulkAction类的doExecute方法，其中 TransportBulkAction类注释明确，shard层面的操作由TransportShardBulkAction完成。

在日常工作中，索引创建应该由ES集群管理员执行，这里不讨论运行自动创建索引的流程。
```
protected void doExecute(Task task, BulkRequest bulkRequest, ActionListener<BulkResponse> listener) {
    //首先检查是否有pipeline操作，若有再查看本节点是否是IngestNode，
    // 是则处理，不是则随机选择是IngestNode的节点并将请求转发过去，处理就结束了
    if (hasIndexRequestsWithPipelines) {
        ..................
    //判断是否需要创建索引(默认打开)
    if (needToCheck()) {
       ..................
    } else {
        //如果自动创建索引已关闭
        executeBulk(task, bulkRequest, startTime, listener, responses, emptyMap());
    }
}
```
doExecute方法首先判断是否由pipeline操作，判断是否需要自动创建索引操作，最后是直接执行executeBulk方法。
```
void executeBulk(Task task, final BulkRequest bulkRequest, final long startTimeNanos, final ActionListener<BulkResponse> listener,
        final AtomicArray<BulkItemResponse> responses, Map<String, IndexNotFoundException> indicesThatCannotBeCreated) {
    new BulkOperation(task, bulkRequest, listener, responses, startTimeNanos, indicesThatCannotBeCreated).run();
}
```
executeBulk方法执行前，需要确保index已经创建完成，index的各shard已在数据节点上建立完成。

（1）检测集群状态
```
//获取最新的集群元数据ClusterState
final ClusterState clusterState = observer.setAndGetObservedState();
//首先会检查集群无BlockException后（存在BlockedException会不断重试，Master节点不存在，会阻塞等待Master节点直至超时）
if (handleBlockExceptions(clusterState)) {
    return;
}
```
（2）遍历BulkRequest的所有子请求，然后根据请求的操作类型生成相应的逻辑
```
MetaData metaData = clusterState.metaData();
//遍历BulkRequest的所有子请求，然后根据请求的操作类型生成相应的逻辑
for (int i = 0; i < bulkRequest.requests.size(); i++) {
    DocWriteRequest<?> docWriteRequest = bulkRequest.requests.get(i);
    //the request can only be null because we set it to null in the previous step, so it gets ignored
    if (docWriteRequest == null) {
        continue;
    }
    if (addFailureIfIndexIsUnavailable(docWriteRequest, i, concreteIndices, metaData)) {
        continue;
    }
    Index concreteIndex = concreteIndices.resolveIfAbsent(docWriteRequest);
    try {
        switch (docWriteRequest.opType()) {
            case CREATE:
            case INDEX:
                //对于写入请求，会首先根据IndexMetaData信息，resolveRouting方法为每条IndexRequest生成路由信息，
                // 并通过process方法按需生成doc id（不指定的话默认是UUID）
                IndexRequest indexRequest = (IndexRequest) docWriteRequest;
                final IndexMetaData indexMetaData = metaData.index(concreteIndex);
                MappingMetaData mappingMd = indexMetaData.mappingOrDefault();
                Version indexCreated = indexMetaData.getCreationVersion();
                //首先根据IndexMetaData信息，resolveRouting方法为每条IndexRequest生成路由信息
                indexRequest.resolveRouting(metaData);
                //通过process方法按需生成doc id（不指定的话默认是UUID）
                indexRequest.process(indexCreated, mappingMd, concreteIndex.getName());
                break;
            case UPDATE:
```
（3）构建请求列表
```
//根据每个IndexRequest请求的路由信息（如果写入时未指定路由，则es默认使用doc id作为路由）得到所要写入的目标shard id，并将DocWriteRequest封装为BulkItemRequest且添加到对应shardId的请求列表中
//requestsByShard的key是shard id，value是对应的单个doc写入请求（会被封装成BulkItemRequest）的集合
Map<ShardId, List<BulkItemRequest>> requestsByShard = new HashMap<>();
for (int i = 0; i < bulkRequest.requests.size(); i++) {
    //从bulk请求中得到每个doc写入请求
    DocWriteRequest<?> request = bulkRequest.requests.get(i);
    if (request == null) {
        continue;
    }
    //路由代码
    String concreteIndex = concreteIndices.getConcreteIndex(request.index()).getName();
    //根据路由，找出doc写入的目标shard id
    ShardId shardId = clusterService.operationRouting().indexShards(clusterState, concreteIndex, request.id(),
        request.routing()).shardId();
    List<BulkItemRequest> shardRequests = requestsByShard.computeIfAbsent(shardId, shard -> new ArrayList<>());
    shardRequests.add(new BulkItemRequest(i, request));
}
```
（4）请求转发
```
shardBulkAction.execute(bulkShardRequest, new ActionListener<BulkShardResponse>() {
```
shardBulkAction对应TransportShardBulkAction实现类，在TransportBulkAction类的注释中也有说明。execute方法和其他Transport**Action处理流程一致，其中doExecute方法如下：
```
protected void doExecute(Task task, Request request, ActionListener<Response> listener) {
    assert request.shardId() != null : "request shardId must be set";
    //转发TransportShardBulkAction请求，最后进入到TransportReplicationAction#doExecute方法
    new ReroutePhase((ReplicationTask) task, request, listener).run();
}
```
在对应的doRun方法中的业务逻辑如下，无论是否为当前节点，最后都调用performAction方法。
```
//通过ClusterState获取到primary shard的路由信息
final ShardRouting primary = primary(state);
if (retryIfUnavailable(state, primary)) {
    return;
}
// 获取主分片所在的shard路由信息，得到主分片所在的node节点
final DiscoveryNode node = state.nodes().get(primary.currentNodeId());
if (primary.currentNodeId().equals(state.nodes().getLocalNodeId())) {
    //是当前节点，继续执行
    performLocalAction(state, primary, node, indexMetaData);
} else {
    //不是当前节点，转发到对应的node上进行处理
    performRemoteAction(state, primary, node);
}
```
performAction方法的具体实现，是ES中很常见的Node节点之间交互的逻辑：
```
transportService.sendRequest(node, action, requestToPerform, transportOptions, new TransportResponseHandler<Response>() 
```
接下来需要看Action注册的过程，经过registerRequestHandler方法，在其他Node上Primary分片执行的业务逻辑是handlePrimaryRequest方法，其他Node上Replica分片执行的业务逻辑上handleReplicaRequest方法。
```
this.transportPrimaryAction = actionName + "[p]";
this.transportReplicaAction = actionName + "[r]";

transportService.registerRequestHandler(transportPrimaryAction, executor, forceExecutionOnPrimary, true,
    in -> new ConcreteShardRequest<>(requestReader, in), this::handlePrimaryRequest);

// we must never reject on because of thread pool capacity on replicas
transportService.registerRequestHandler(transportReplicaAction, executor, true, true,
    in -> new ConcreteReplicaRequest<>(replicaRequestReader, in), this::handleReplicaRequest);
```

## 主分片节点流程
ES的代码风格还是比较一致的，这里又开启一个线程，异步执行主分片流程：
```
protected void handlePrimaryRequest(final ConcreteShardRequest<Request> request, final TransportChannel channel, final Task task) {
    new AsyncPrimaryAction(
        request, new ChannelActionListener<>(channel, transportPrimaryAction, request), (ReplicationTask) task).run();
}
-----------------------------------------------
protected void doRun() throws Exception {
   //检查请求： 1.校验当前分片是否仍然为主分片，必须确保当前操作是在主分片上执行的
    if (shardRouting.primary() == false) {
        throw new ReplicationOperation.RetryOnPrimaryException(shardId, "actual shard is not a primary " + shardRouting);
    }
    //检查请求：2.allocationId是否是预期值，校验主分片分配ID是否与请求分配ID一致
    final String actualAllocationId = shardRouting.allocationId().getId();
    if (actualAllocationId.equals(primaryRequest.getTargetAllocationID()) == false) {
        throw new ShardNotFoundException(shardId, "expected allocation id [{}] but found [{}]",
            primaryRequest.getTargetAllocationID(), actualAllocationId);
    }
    //检查请求：3.校验主分片期数(term)是否与请求期数一致
    final long actualTerm = indexShard.getPendingPrimaryTerm();
    if (actualTerm != primaryRequest.getPrimaryTerm()) {
        throw new ShardNotFoundException(shardId, "expected allocation id [{}] with term [{}] but found [{}]",
            primaryRequest.getTargetAllocationID(), primaryRequest.getPrimaryTerm(), actualTerm);
    }

//请求操作执行的主分片的许可证，成功后回调runWithPrimaryShardReference方法
    acquirePrimaryOperationPermit(
            indexShard,
            primaryRequest.getRequest(),
            ActionListener.wrap(
                    /**主分片写操作入口*/
		 releasable -> runWithPrimaryShardReference(new PrimaryShardReference(indexShard, releasable)),
```
acquirePrimaryOperationPermit核心是方法核心功能是向主分片请求许可证。如果请求成功，则执行回调函数runWithPrimaryShardReference，该回调函数接收PrimaryShardReference对象，该对象表示当前操作的主分片引用。如果请求许可证失败，则回调onFailure方法。

runWithPrimaryShardReference是主分片写操作入口，先判断主分片是否迁移，这里简化逻辑为主分片未发送迁移：
```
//2.主分片准备操作;3.转发请求给副本分片
new ReplicationOperation<>(primaryRequest.getRequest(), primaryShardReference,
    ActionListener.wrap(result -> result.respond(globalCheckpointSyncingListener), referenceClosingListener::onFailure),
    newReplicasProxy(), logger, actionName, primaryRequest.getPrimaryTerm()).execute();
----------------------------------------------------
//主分片写操作
public void execute() throws Exception {
    // 检查是否有足够的活跃shard数量（相比于配置中要求的），如果没有则阻塞等待
    final String activeShardCountFailure = checkActiveShardCount();
    final ShardRouting primaryRouting = primary.routingEntry();
    final ShardId primaryId = primaryRouting.shardId();
    if (activeShardCountFailure != null) {
        finishAsFailed(new UnavailableShardsException(primaryId,
            "{} Timeout: [{}], request: [{}]", activeShardCountFailure, request.timeout(), request));
        return;
    }

    totalShards.incrementAndGet();
    pendingActions.incrementAndGet(); // increase by 1 until we finish all primary coordination
    // perform关键，这里开始执行写主分片
    // handlePrimaryResult方法内，主分片完成后，副本分片执行操作
    primary.perform(request, ActionListener.wrap(this::handlePrimaryResult, resultListener::onFailure));
}
```
接下来调用TransportShardBulkAction的shardOperationOnPrimary方法。又是熟悉的代码风格，在TransportShardBulkAction类中有之前构建的请求，也有具体的业务逻辑实现。
```
public static void performOnPrimary(
    BulkShardRequest request,
    IndexShard primary,
    UpdateHelper updateHelper,
    LongSupplier nowInMillisSupplier,
    MappingUpdatePerformer mappingUpdater,
    Consumer<ActionListener<Void>> waitForMappingUpdate,
    ActionListener<PrimaryResult<BulkShardRequest, BulkShardResponse>> listener,
    ThreadPool threadPool) {

    new ActionRunnable<PrimaryResult<BulkShardRequest, BulkShardResponse>>(listener) {

        private final Executor executor = threadPool.executor(ThreadPool.Names.WRITE);

        private final BulkPrimaryExecutionContext context = new BulkPrimaryExecutionContext(request, primary);

        @Override
        protected void doRun() throws Exception {
            while (context.hasMoreOperationsToExecute()) {
                //执行Lucene与Translog写操作
                if (executeBulkItemRequest(context, updateHelper, nowInMillisSupplier, mappingUpdater, waitForMappingUpdate,
                    ActionListener.wrap(v -> executor.execute(this), this::onRejection)) == false) {
              
```
在executeBulkItemRequest方法内，通过调用applyIndexOperationOnPrimary方法完成Lucene索引与Translog文件的写入，在通用方法applyIndexOperation中最终调用return index(engine, operation);开始执行物理文件的写操作。

## 副本分片节点流程
主分片完成写操作后，开始执行副本分片的写操作。
```
primary.perform(request, ActionListener.wrap(this::handlePrimaryResult, resultListener::onFailure));
```
handlePrimaryResult处理主分片操作的结果：
```
//主分片处理写操作的结果
private void handlePrimaryResult(final PrimaryResultT primaryResult) {
    this.primaryResult = primaryResult;
    //主分片更新localCheckPoint
    primary.updateLocalCheckpointForShard(primary.routingEntry().allocationId().getId(), primary.localCheckpoint());
    //主分片更新GlobalChcekPoint
    primary.updateGlobalCheckpointForShard(primary.routingEntry().allocationId().getId(), primary.globalCheckpoint());
    final ReplicaRequest replicaRequest = primaryResult.replicaRequest();
    if (replicaRequest != null) {
        final long globalCheckpoint = primary.computedGlobalCheckpoint();
        final long maxSeqNoOfUpdatesOrDeletes = primary.maxSeqNoOfUpdatesOrDeletes();
        assert maxSeqNoOfUpdatesOrDeletes != SequenceNumbers.UNASSIGNED_SEQ_NO : "seqno_of_updates still uninitialized";
        final ReplicationGroup replicationGroup = primary.getReplicationGroup();
        markUnavailableShardsAsStale(replicaRequest, replicationGroup);
        /**副本分片写入流程入口*/
        performOnReplicas(replicaRequest, globalCheckpoint, maxSeqNoOfUpdatesOrDeletes, replicationGroup);
    }
    //标记主分片写成功
    successfulShards.incrementAndGet();
```
（1）发起请求

performOnReplicas方法开启副本分片写操作流程：遍历需要发送副本分片的列表，发送给副本的内容包含SequenceID、PrimaryTerm、GlobalCheckPoint、version。最终调用 replicasProxy.performOn把请求发送到其他Node。
```
private void performOnReplicas(final ReplicaRequest replicaRequest, final long globalCheckpoint,
                               final long maxSeqNoOfUpdatesOrDeletes, final ReplicationGroup replicationGroup) {
    // for total stats, add number of unassigned shards and
    // number of initializing shards that are not ready yet to receive operations (recovery has not opened engine yet on the target)
    totalShards.addAndGet(replicationGroup.getSkippedShards().size());

    //获取对应的shardRouting
    final ShardRouting primaryRouting = primary.routingEntry();

    //现在已经为要写的副本shard准备了一个列表，循环处理每个shard，跳过unassigned状态的 shard，向目标节点发送请求，等待响应。这个过程是异步并行的
    for (final ShardRouting shard : replicationGroup.getReplicationTargets()) {
        if (shard.isSameAllocation(primaryRouting) == false) {
            //转发请求时会将SequenceID、PrimaryTerm、GlobalCheckPoint、version 等传递给副分片
            performOnReplica(shard, replicaRequest, globalCheckpoint, maxSeqNoOfUpdatesOrDeletes);
        }
    }
}
-------------------------------------------------
private void performOnReplica(final ShardRouting shard, final ReplicaRequest replicaRequest,
                              final long globalCheckpoint, final long maxSeqNoOfUpdatesOrDeletes) {
    totalShards.incrementAndGet();
    pendingActions.incrementAndGet();
    replicasProxy.performOn(shard, replicaRequest, primaryTerm, globalCheckpoint, maxSeqNoOfUpdatesOrDeletes,
        //在等待Response的过程中，本节点发出了多少个Request，就要等待多少个Response。
        // 无论这些Response是成功的还是失败的，直到超时。收集到全部的Response后，执行finish()。给协调节点返回消息，告知其哪些成功、哪些失败了
        new ActionListener<ReplicaResponse>() {
            @Override
            public void onResponse(ReplicaResponse response) {
                successfulShards.incrementAndGet();
                try {
                    //主分片更新localCheckPoint
                    primary.updateLocalCheckpointForShard(shard.allocationId().getId(), response.localCheckpoint());
                    //主分片更新GlobalChcekPoint
                    primary.updateGlobalCheckpointForShard(shard.allocationId().getId(), response.globalCheckpoint());
                }
```
replicasProxy.performOn后请求成功的监听结果，依旧是主分片更新localCheckPoint和GlobalChcekPoint。接下来人就是熟悉的Transport**Action的处理过程：
```
public void performOn(
        final ShardRouting replica,
        final ReplicaRequest request,
        final long primaryTerm,
        final long globalCheckpoint,
        final long maxSeqNoOfUpdatesOrDeletes,
        final ActionListener<ReplicationOperation.ReplicaResponse> listener) {
    String nodeId = replica.currentNodeId();
    final DiscoveryNode node = clusterService.state().nodes().get(nodeId);
    if (node == null) {
        listener.onFailure(new NoNodeAvailableException("unknown node [" + nodeId + "]"));
        return;
    }
    final ConcreteReplicaRequest<ReplicaRequest> replicaRequest = new ConcreteReplicaRequest<>(
        request, replica.allocationId().getId(), primaryTerm, globalCheckpoint, maxSeqNoOfUpdatesOrDeletes);
    final ActionListenerResponseHandler<ReplicaResponse> handler = new ActionListenerResponseHandler<>(listener,
        ReplicaResponse::new);
    transportService.sendRequest(node, transportReplicaAction, replicaRequest, transportOptions, handler);
}
```
（2）副本所在Node开始写操作

根据registerRequestHandler方法，看出副本Node的逻辑处理流程在handleReplicaRequest方法。
```
// we must never reject on because of thread pool capacity on replicas
transportService.registerRequestHandler(transportReplicaAction, executor, true, true,
    in -> new ConcreteReplicaRequest<>(replicaRequestReader, in), this::handleReplicaRequest);

```
相同的代码风格AsyncReplicaAction的run方法，开启另一个线程执行。
```
protected void handleReplicaRequest(final ConcreteReplicaRequest<ReplicaRequest> replicaRequest,
                                    final TransportChannel channel, final Task task) {
    new AsyncReplicaAction(
        replicaRequest, new ChannelActionListener<>(channel, transportReplicaAction, replicaRequest), (ReplicationTask) task).run();
}
```
接下来进入到IndexShard类的innerAcquireReplicaOperationPermit方法。
```
protected void doRun() throws Exception {
    //进入到副本阶段
    setPhase(task, "replica");
    final String actualAllocationId = this.replica.routingEntry().allocationId().getId();
    //检查AllocationId
    if (actualAllocationId.equals(replicaRequest.getTargetAllocationID()) == false) {
        throw new ShardNotFoundException(this.replica.shardId(), "expected allocation id [{}] but found [{}]",
            replicaRequest.getTargetAllocationID(), actualAllocationId);
    }
    acquireReplicaOperationPermit(replica, replicaRequest.getRequest(), this, replicaRequest.getPrimaryTerm(),
        replicaRequest.getGlobalCheckpoint(), replicaRequest.getMaxSeqNoOfUpdatesOrDeletes());
}
```
运行执行后，在OnReponse方法内执行副本分片写入：
```
public void onResponse(Releasable releasable) {
    try {
        assert replica.getActiveOperationsCount() != 0 : "must perform shard operation under a permit";
        /**副本分片写操作入口*/
        final ReplicaResult replicaResult = shardOperationOnReplica(replicaRequest.getRequest(), replica);
        releasable.close(); // release shard operation lock before responding to caller
        final TransportReplicationAction.ReplicaResponse response =
                new ReplicaResponse(replica.getLocalCheckpoint(), replica.getLastSyncedGlobalCheckpoint());
        replicaResult.respond(new ResponseListener(response));
    }
```
shardOperationOnReplica最终调用performOpOnReplica实现副本分片写操作：
```
public static Translog.Location performOnReplica(BulkShardRequest request, IndexShard replica) throws Exception {
    Translog.Location location = null;
    for (int i = 0; i < request.items().length; i++) {
        final BulkItemRequest item = request.items()[i];
        final BulkItemResponse response = item.getPrimaryResponse();
        final Engine.Result operationResult;
        //主分片响应结果失败
        if (item.getPrimaryResponse().isFailed()) {
            if (response.getFailure().getSeqNo() == SequenceNumbers.UNASSIGNED_SEQ_NO) {
                continue; // ignore replication as we didn't generate a sequence number for this request.
            }
            operationResult = replica.markSeqNoAsNoop(response.getFailure().getSeqNo(), response.getFailure().getMessage());
        } else {
            if (response.getResponse().getResult() == DocWriteResponse.Result.NOOP) {
                continue; // ignore replication as it's a noop
            }
            assert response.getResponse().getSeqNo() != SequenceNumbers.UNASSIGNED_SEQ_NO;
            //副本分片开始写入
            operationResult = performOpOnReplica(response.getResponse(), item.request(), replica);
        }
        assert operationResult != null : "operation result must never be null when primary response has no failure";
        location = syncOperationResultOrThrow(operationResult, location);
    }
    return location;
}
```
在performOpOnReplica方法内，通过调用applyIndexOperationOnReplica方法完成Lucene索引与Translog文件的写入，在通用方法applyIndexOperation中最终调用return index(engine, operation);开始执行物理文件的写操作。

## Sequence IDs
Es 从 6.0 开始引入了Sequence IDs概念，使用唯一的 ID 来标记每个索引操作，能够对索引操作进行总排序。

Primary Terms 和 Sequence Numbers
根据Index API发起请求响应结果包含 "_primary_term"和"_seq_no"（用于乐观锁并发控制）。

乐观并发控制（optimistic-concurrency-control）：ES是分布式应用，created, updated, or deleted等操作可以并行执行，必须确保新版本的操作不能被老版本操作覆盖。

（1）Sequence Numbers：标记发生在某个分片上的写操作。由主分片分配，只对写操作分配。

（2）Primary Terms：由Master节点分配给每个主分片，每次主分片发生变化时递增。Primary shard变更一次，对应的值就会发生变化。

这和 Raft 中的 term，Kafka Controller controller_epoch 值概念类似。

主分片每次向副分片转发写请求时，会带上这两个值。

## GlobalCheckpoint 和 LocalCheckpoint
有了 Primary Terms 和 Sequence Numbers，在理论上能够检测出分片之间差异，并在主分片失效时，重新对齐他们的工具。旧主分片就可以恢复为与拥有更高 primary term 值的新主分片一致：从旧主分片中删除新主分片操作历史中不存在的操作，并将缺少的操作索引到旧主分片。

但是，同时为每秒成百上千的事件做索引，比较数百万个操作的历史是不切实际的。存储成本变得非常昂贵，直接比较的计算工作量太长了。为了解决这个问题，es 维护了一个名为“全局检查点”（global checkpoint）的安全标记。

（1）GlobalCheckpoint

主分片负责推进全局检查点，它通过跟踪在副分片上完成的操作来实现。一旦它检测到所有副分片已经超出给定序列号，它将相应地更新全局检查点。

全局检查点是所有活跃分片历史都已对齐到的序列号，换句话说，所有低于全局检查点的操作都保证已被所有活跃的分片处理完毕。当主分片失效，我们只需要比较新主分片与其他副分片之间的最后一个全局检查点之后的操作。当旧主分片恢复时，我们使用它知道的全局检查点，与新主分片进行比较。这样，我们只有小部分操作需要比较。不用比较全部。

（2）LocalCheckpoint

副分片不会跟踪所有操作，而是维护一个类似全局检查点局部变量，称为本地检查点。本地检查点也是一个序列号，所有序列号低于它的操作都已在该分片上处理（lucene 和 translog 写成功，不一定刷盘）完毕。当副分片确认(ack)一个写操作到主分片节点时，他们也会更新本地检查点。使用本地检查点，主分片节点能够更新全局检查点，然后在下一次索引操作时将其发送到所有分片副本。

GlobalCheckpoint：一个写操作在Primary Shard/Replica Shard的整体进度；
LocalCheckPoint：一个写操作在当前shard的进度；
GlobalCheckpoint和LocalCheckPoint都有对应的物理文件；

（3）全局检测点和本地检查点在内存中维护，但也会保存在每个 lucene 提交的元数据中。

全局/本地检查点的更新情况（来自官网）：某索引由 1 个主分片，2 个副分片，初始状态，没有数据，全局检查点和本地检查点都在 0 位置。

1. 主分片写入一条数据成功后，本地检查点向前推进，主分片将写请求转发到副分片

2. 副分片本地处理成功后，将本地检查点向前推进

3. 主分片收到到所有副分片都处理成功的消息，根据汇报的各副本上的本地检查点，更新全局检查点

4. 在下一次索引操作时，主分片节点将全局检查点发送到所有分片副本

通过上述内容看出，checkpoint更新会发生的两种场景：

（1）第一次在handlePrimaryResult方法内，主分片完成操作后，转发副本分片前；
```
//主分片处理写操作的结果
private void handlePrimaryResult(final PrimaryResultT primaryResult) {
    this.primaryResult = primaryResult;
    //主分片更新localCheckPoint
    primary.updateLocalCheckpointForShard(primary.routingEntry().allocationId().getId(), primary.localCheckpoint());
    //主分片更新GlobalChcekPoint
    primary.updateGlobalCheckpointForShard(primary.routingEntry().allocationId().getId(), primary.globalCheckpoint());
```
（2）第二次在performOnReplica方法内，副本分片完成写操作后；
```
replicasProxy.performOn(shard, replicaRequest, primaryTerm, globalCheckpoint, maxSeqNoOfUpdatesOrDeletes,
    //在等待Response的过程中，本节点发出了多少个Request，就要等待多少个Response。
    // 无论这些Response是成功的还是失败的，直到超时。收集到全部的Response后，执行finish()。给协调节点返回消息，告知其哪些成功、哪些失败了
    new ActionListener<ReplicaResponse>() {
        @Override
        public void onResponse(ReplicaResponse response) {
            successfulShards.incrementAndGet();
            try {
                //主分片更新localCheckPoint
                primary.updateLocalCheckpointForShard(shard.allocationId().getId(), response.localCheckpoint());
                //主分片更新GlobalChcekPoint
                primary.updateGlobalCheckpointForShard(shard.allocationId().getId(), response.globalCheckpoint());
            }
```
updateGlobalCheckpointForShard调用IndexShard的updateLocalCheckpointForShard方法；
```
public synchronized void updateLocalCheckpoint(final String allocationId, final long localCheckpoint) {
    //获取主分片本地的checkpoints,包括LocalCheckpoint和GlobalCheckpoint
    CheckpointState cps = checkpoints.get(allocationId);
    if (cps == null) {
        // can happen if replica was removed from cluster but replication process is unaware of it yet
        return;
    }
    // 检查是否需要更新LocalCheckpoint，即需要更新的值是否大于当前已有值
    boolean increasedLocalCheckpoint = updateLocalCheckpoint(allocationId, cps, localCheckpoint);
    // pendingInSync是一个保存等待更新LocalCheckpoint的Set，存放allocation IDs
    boolean pending = pendingInSync.contains(allocationId);
    // 如果是待更新的，且当前的localCheckpoint大于等于GlobalCheckpoint(每次都是先更新Local再Global，正常情况下，Local应该大于等于Global)
    if (pending && cps.localCheckpoint >= getGlobalCheckpoint()) {
        //从待更新集合中移除
        pendingInSync.remove(allocationId);
        pending = false;
        //此分片是否同步，用于更新GlobalCheckpoint时使用
        cps.inSync = true;
        replicationGroup = calculateReplicationGroup();
        logger.trace("marked [{}] as in-sync", allocationId);
        notifyAllWaiters();
    }

    if (increasedLocalCheckpoint && pending == false) {
        //更新GlobalCheckpoint
        updateGlobalCheckpointOnPrimary();
    }
```
updateLocalCheckpointForShard调用IndexShard的updateGlobalCheckpointForShard方法；
```
private synchronized void updateGlobalCheckpointOnPrimary() {
    assert primaryMode;
    final CheckpointState cps = checkpoints.get(shardAllocationId);
    final long globalCheckpoint = cps.globalCheckpoint;
    // 计算GlobalCheckpoint，即检验无误后，取Math.min(cps.localCheckpoint, Long.MAX_VALUE)
    final long computedGlobalCheckpoint = computeGlobalCheckpoint(pendingInSync, checkpoints.values(), getGlobalCheckpoint());
    // 需要更新到的GlobalCheckpoint值比当前的global值大，则需要更新
    if (globalCheckpoint != computedGlobalCheckpoint) {
        cps.globalCheckpoint = computedGlobalCheckpoint;
        logger.trace("updated global checkpoint to [{}]", computedGlobalCheckpoint);
        onGlobalCheckpointUpdated.accept(computedGlobalCheckpoint);
    }
}
```
ES是一个典型的主备模式(primary-backup model)的分布式系统，对应特性：

数据可靠性：ES通过副本和Translog保障数据的安全；
服务可用性：在可用性和一致性取舍上，ES更倾向于可用性，只要主分片可用即可执行写入操作；
数据一致性：默认只要主写入成功，数据就可以被读取，所以查询时操作在主分片和副本分片上可能会得到不同的结果；
原子性：索引的读写、别名操作都是原子操作，不会出现中间状态。但是bulk不是原子操作，不能用来实现事物；
