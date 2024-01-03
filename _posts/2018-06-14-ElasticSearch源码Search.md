---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码Search
ElasticSearch的搜索包含两部分：（1）结构化搜索，不涉及评分，_index、_type(es7后废弃，统一_doc) 和 id 三元组来确定唯一文档（2）全文索引，根据关键词从倒排索引中搜索，并根据相关性得分进行排序。

## Search API
全文索引的应用范围广阔，最明显的地方就是对日志的搜索，几乎每个开发人员都会涉及。全文索引对应的API如下：

Rest Client API及Rest API对比看出，Rest API是最全面的，Rest Client API能够满足日常绝大部分开发场景，和之前的文章类似，本文对API介绍部分两者都会包含。

SearchRequest支持documents搜索、aggregations、suggestions、结果高亮显示highlighting。
```
SearchRequest searchRequest = new SearchRequest(); 
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder(); 
searchSourceBuilder.query(QueryBuilders.matchAllQuery()); 
searchRequest.source(searchSourceBuilder);
```

### 参数
（1）indicesOptions

设置IndicesOptions控制如何解析不可用索引以及扩展通配符表达式

（2）preference

使用首选参数，例如，执行搜索优先选择本地分片。 默认值是随机化分片。

（3）SearchSourceBuilder

控制请求，基本上和Rest API保持一致。官网示例给出，QueryBuilders.termQuery/QueryBuilders.matchQuery，除此之外的QueryBuilder的具体实现都能使用。

构建一个或多个QueryBuilder，然后放入SearchRequest即可。

（4）排序SortBuilder

排序对应ScoreSortBuilder/FieldSortBuilder/GeoDistanceSortBuilder/ScriptSortBuilder四种类型。

（5）过滤

sourceBuilder.fetchSource(false);设置为false，则返回的值不再包含_source内容，除此之外还有include/exclude等过滤。

（6） 高亮Highlighting

当搜素命中关键词时，突出显示搜索结果，可以设置Hignhlighting。

官网示例包含聚合和Suggest相关内容，用到的时候，再总结

## 执行
和其他请求一样，同样分为异步和同步两种形式，异步调用示例如下。
```
ActionListener<SearchResponse> listener = new ActionListener<SearchResponse>() {
    @Override
    public void onResponse(SearchResponse searchResponse) {
        
    }

    @Override
    public void onFailure(Exception e) {
        
    }
};
-------------------------------------
client.searchAsync(searchRequest, RequestOptions.DEFAULT, listener); 
```

## 响应
官网给的示例时，在twitter索引中user字段搜索kimchy，响应结果如下：
```
{
  "took" : 5,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.3862944,
    "hits" : [
      {
        "_index" : "twitter",
        "_type" : "_doc",
        "_id" : "0",
        "_score" : 1.3862944,
        "_source" : {
          "date" : "2009-11-15T14:12:12",
          "likes" : 0,
          "message" : "trying out Elasticsearch",
          "user" : "kimchy"
}
```
took代表搜索执行时间，仅包含服务端的响应时间，不包含客户端到服务端请求及响应时间；
total代表本次搜索命中的文档数量；
max_score 为根据搜索词计算出的最大得分，代表文档匹配度；
hits为搜索命中的结果列表，默认为10条；

## 服务端响应
Search操作流程是，client发起请求，Server端响应请求并处理，最后返回结果。

### Rest响应
Search操作在Rest**Action对应的时RestSearchAction，prepareRequest方法首先解析请求，然后发起调用到Transport**Action层。
```
public RestChannelConsumer prepareRequest(final RestRequest request, final NodeClient client) throws IOException {
    SearchRequest searchRequest = new SearchRequest();
    IntConsumer setSize = size -> searchRequest.source().size(size);
    // 解析参数，将请求体RestRequest解析为SearchRequest 数据结构
    request.withContentOrSourceParamParserOrNull(parser ->
        parseSearchRequest(searchRequest, request, parser, setSize));

    // 具体的search 业务处理入口
    return channel -> {
        RestStatusToXContentListener<SearchResponse> listener = new RestStatusToXContentListener<>(channel);
        HttpChannelTaskHandler.INSTANCE.execute(client, request.getHttpChannel(), searchRequest, SearchAction.INSTANCE, listener);
    };
}
```
parseSearchRequest 将请求体RestRequest解析为SearchRequest 数据结构，客户端的请求会全部转换为SearchRequest。请求体解析过程对应的参数，可以和请求参数对应，更容易理解。

和其他请求不同的是，这里请求通过HttpChannelTaskHandler.INSTANCE 执行。
```
<Response extends ActionResponse> void execute(NodeClient client, HttpChannel httpChannel, ActionRequest request,
                                               ActionType<Response> actionType, ActionListener<Response> listener) {

    CloseListener closeListener = httpChannels.computeIfAbsent(httpChannel, channel -> new CloseListener(client));
    TaskHolder taskHolder = new TaskHolder();
    //任务执行
    Task task = client.executeLocally(actionType, request,
        new ActionListener<Response>() {
            @Override
            public void onResponse(Response searchResponse) {
                try {
                    closeListener.unregisterTask(taskHolder);
                } finally {
	            //响应客户端
                    listener.onResponse(searchResponse);
        });
    closeListener.registerTask(taskHolder, new TaskId(client.getLocalNodeId(), task.getId()));
    closeListener.maybeRegisterChannel(httpChannel);
}
```
之后的流程和其他请求一样，通过NodeClient继续执行。

### Transport层处理
Search操作在Transport层对应的是TransportSearchAction类，其中doExecute方法：
```
protected void doExecute(Task task, SearchRequest searchRequest, ActionListener<SearchResponse> listener) {
    final long relativeStartNanos = System.nanoTime();
    //搜索时间记录
    final SearchTimeProvider timeProvider =
        new SearchTimeProvider(searchRequest.getOrCreateAbsoluteStartMillis(), relativeStartNanos, System::nanoTime);
    ActionListener<SearchSourceBuilder> rewriteListener = ActionListener.wrap(source -> {
        if (source != searchRequest.source()) {
            // only set it if it changed - we don't allow null values to be set but it might be already null. this way we catch
            // situations when source is rewritten to null due to a bug
            searchRequest.source(source);
        }
        final ClusterState clusterState = clusterService.state();
        //1、会对其他集群的索引和当前集群的索引分组,
        // 将请求涉及的本集群shard列表和远程集群的shard列表(远程集群用于跨集群访问)合并
        final Map<String, OriginalIndices> remoteClusterIndices = remoteClusterService.groupIndices(searchRequest.indicesOptions(),
            searchRequest.indices(), idx -> indexNameExpressionResolver.hasIndexOrAlias(idx, clusterState));
        //获取本地集群列表包含索引
        OriginalIndices localIndices = remoteClusterIndices.remove(RemoteClusterAware.LOCAL_CLUSTER_GROUP_KEY);
        if (remoteClusterIndices.isEmpty()) {
            //远程集群对应数据为空，只在当前Node执行搜索
            executeLocalSearch(task, timeProvider, searchRequest, localIndices, clusterState, listener);
        }
```
这里localSearch是当前集群搜索，不涉及跨集群访问（本文讨论内容），还有一个是ccsRemoteReduce核心是跨集群的访问。

搜索的核心分为查询节点和搜索阶段，两个步骤。

## 查询阶段
服务端响应内容，同样是协调者节点的响应客户都请求的过程。接下来进入到查询阶段，官网给的示意图如下：

查询阶段包含以下三个步骤:

客户端发送一个 search 请求到 Node 3 ， Node 3 会创建一个大小为 from + size 的空优先队列。
Node 3 将查询请求转发到索引的每个主分片或副本分片中。每个分片在本地执行查询并添加结果到大小为 from + size 的本地有序优先队列中。
每个分片返回各自优先队列中所有文档的 ID 和排序值给协调节点，也就是 Node 3 ，它合并这些值到自己的优先队列中来产生一个全局排序后的结果列表。
当一个搜索请求被发送到某个节点时，这个节点就变成了协调节点。 这个节点的任务是广播查询请求到所有相关分片并将它们的响应整合成全局排序后的结果集合，这个结果集合会返回给客户端。

第一步是广播请求到索引中每一个节点的分片拷贝。就像 document GET requests 所描述的， 查询请求可以被某个主分片或某个副本分片处理， 这就是为什么更多的副本（当结合更多的硬件）能够增加搜索吞吐率。 协调节点将在之后的请求中轮询所有的分片拷贝来分摊负载。

每个分片在本地执行查询请求并且创建一个长度为 from + size 的优先队列—也就是说，每个分片创建的结果集足够大，均可以满足全局的搜索请求。 分片返回一个轻量级的结果列表到协调节点，它仅包含文档 ID 集合以及任何排序需要用到的值，例如 _score 。

协调节点将这些分片级的结果合并到自己的有序优先队列里，它代表了全局排序结果集合。至此查询过程结束。
```
一个索引可以由一个或几个主分片组成， 所以一个针对单个索引的搜索请求需要能够把来自多个分片的结果组合起来。 针对 multiple 或者 all 索引的搜索工作方式也是完全一致的—​仅仅是包含了更多的分片而已。
```

## 协调者阶段
接上步骤，接下来进入搜索的步骤executeSearch方法内：
```
private void executeSearch(SearchTask task, SearchTimeProvider timeProvider, SearchRequest searchRequest,
                           OriginalIndices localIndices, List<SearchShardIterator> remoteShardIterators,
                           BiFunction<String, String, DiscoveryNode> remoteConnections, ClusterState clusterState,
                           Map<String, AliasFilter> remoteAliasMap, ActionListener<SearchResponse> listener,
                           SearchResponse.Clusters clusters) {

    //1 集群的状态是否可用，是有有异常
    clusterState.blocks().globalBlockedRaiseException(ClusterBlockLevel.READ);
    //2 解析索引名称 正常的别名或者索引es不允许以_开始
    final Index[] indices = resolveLocalIndices(localIndices, searchRequest.indicesOptions(), clusterState, timeProvider);
    //3、构建别名过滤器 如果是内部索引查询比如.ml-config，是获取不到任何的别名过滤器的
    Map<String, AliasFilter> aliasFilter = buildPerIndexAliasFilter(searchRequest, clusterState, indices, remoteAliasMap);
    Map<String, Set<String>> routingMap = indexNameExpressionResolver.resolveSearchRouting(clusterState, searchRequest.routing(),
        searchRequest.indices());
    routingMap = routingMap == null ? Collections.emptyMap() : Collections.unmodifiableMap(routingMap);
    //4、处理请求中的boost
    Map<String, Float> concreteIndexBoosts = resolveIndexBoosts(searchRequest, clusterState);

    if (shouldSplitIndices(searchRequest)) {
        List<String> writeIndicesList = new ArrayList<>();
        List<String> readOnlyIndicesList = new ArrayList<>();
        //索引划分为只读和只写两种类型
        splitIndices(indices, clusterState, writeIndicesList, readOnlyIndicesList);
        String[] writeIndices = writeIndicesList.toArray(new String[0]);
        String[] readOnlyIndices = readOnlyIndicesList.toArray(new String[0]);

        if (readOnlyIndices.length == 0) {
            //搜索只写索引
            executeSearch(task, timeProvider, searchRequest, localIndices, writeIndices, routingMap,
                aliasFilter, concreteIndexBoosts, remoteShardIterators, remoteConnections, clusterState, listener, clusters);
        }
```
搜索无论是哪种形式，最终由executeSearch方法执行：

```
private void executeSearch(SearchTask task, SearchTimeProvider timeProvider, SearchRequest searchRequest,
                           OriginalIndices localIndices, String[] concreteIndices, Map<String, Set<String>> routingMap,
                           Map<String, AliasFilter> aliasFilter, Map<String, Float> concreteIndexBoosts,
                           List<SearchShardIterator> remoteShardIterators, BiFunction<String, String, DiscoveryNode> remoteConnections,
                           ClusterState clusterState, ActionListener<SearchResponse> listener, SearchResponse.Clusters clusters) {

    Map<String, Long> nodeSearchCounts = searchTransportService.getPendingSearchRequests();

    //6 根据路由获取对应的分片
    GroupShardsIterator<ShardIterator> localShardsIterator = clusterService.operationRouting().searchShards(clusterState,
            concreteIndices, routingMap, searchRequest.preference(), searchService.getResponseCollectorService(), nodeSearchCounts);
    //是将本地集群和其他集群的迭代器对象列表进行合并
    GroupShardsIterator<SearchShardIterator> shardIterators = mergeShardsIterators(localShardsIterator, localIndices,
        searchRequest.getLocalClusterAlias(), remoteShardIterators);

    failIfOverShardCountLimit(clusterService, shardIterators.size());

    // optimize search type for cases where there is only one shard group to search on
    if (shardIterators.size() == 1) {
        // if we only have one group, then we always want Q_T_F, no need for DFS, and no need to do THEN since we hit one shard
        searchRequest.searchType(QUERY_THEN_FETCH);
    }
    //8 如果用户不设置 此处默认给修改为true
    if (searchRequest.allowPartialSearchResults() == null) {
       // No user preference defined in search request - apply cluster service default
        searchRequest.allowPartialSearchResults(searchService.defaultAllowPartialSearchResults());
    }
    if (searchRequest.isSuggestOnly()) {
        // disable request cache if we have only suggest
        searchRequest.requestCache(false);
        if (searchRequest.searchType() == DFS_QUERY_THEN_FETCH) {
            // convert to Q_T_F if we have only suggest
            searchRequest.searchType(QUERY_THEN_FETCH);
        }
    }

    //9 获取当前的节点 查看当前集群中的节点数量会调用ClusterState中的nodes方法
    final DiscoveryNodes nodes = clusterState.nodes();
    BiFunction<String, String, Transport.Connection> connectionLookup = buildConnectionLookup(searchRequest.getLocalClusterAlias(),
        nodes::get, remoteConnections, searchTransportService::getConnection);
    // 判断是否需要在查询前做目标分片过滤：pre_filter_shard_size
    // https://www.elastic.co/guide/en/elasticsearch/reference/current/search-search.html
    // shouldPreFilterSearchShards,判断为true需要同时满足3个条件：
    // 1、查询类型为QUERY_THEN_FETCH
    // 2、是否能通过查询重写预判出查询结果为空或者有字段排序
    // 3、实际的查询分片数量> preFilterShardSize(默认128)
    // 需要注意的是：pre-filter 最主要的作用不是降低查询延迟，而是 pre-filter 阶段可以不占用search theadpool，减少了这个线程池的占用情况。
    boolean preFilterSearchShards = shouldPreFilterSearchShards(searchRequest, shardIterators);
    //10 异步搜索
    searchAsyncAction(task, searchRequest, shardIterators, timeProvider, connectionLookup, clusterState.version(),
        Collections.unmodifiableMap(aliasFilter), concreteIndexBoosts, routingMap, listener, preFilterSearchShards, clusters).start();
}
```
其中，搜索类型构建过程如下：
```
AbstractSearchAsyncAction<? extends SearchPhaseResult> searchAsyncAction;
switch (searchRequest.searchType()) {
    case DFS_QUERY_THEN_FETCH:
        // 与 “Query Then Fetch” 相同，除了初始分散阶段，该阶段计算分布式term频率以获得更准确的评分。
        searchAsyncAction = new SearchDfsQueryThenFetchAsyncAction(logger, searchTransportService, connectionLookup,
            aliasFilter, concreteIndexBoosts, indexRoutings, searchPhaseController, executor, searchRequest, listener,
            shardIterators, timeProvider, clusterStateVersion, task, clusters);
        break;
    case QUERY_THEN_FETCH:
        // 请求分两个阶段处理。 在第一阶段，查询被转发到所有涉及的分片。 每个分片执行搜索并生成对该分片本地的结果的排序列表。
        // 每个分片只向协调节点返回足够的信息，以允许其合并并将分片级结果重新排序为全局排序的最大长度大小的结果集。
        // 在第二阶段期间，协调节点仅从相关分片请求文档内容（以及高亮显示的片段，如果有的话）。
        // 如果您未在请求中指定 search_type，那么这是默认设置。
        searchAsyncAction = new SearchQueryThenFetchAsyncAction(logger, searchTransportService, connectionLookup,
            aliasFilter, concreteIndexBoosts, indexRoutings, searchPhaseController, executor, searchRequest, listener,
            shardIterators, timeProvider, clusterStateVersion, task, clusters);
        break;
    default:
        throw new IllegalStateException("Unknown search type: [" + searchRequest.searchType() + "]");
}
return searchAsyncAction;
```
接下来进入到AbstractSearchAsyncAction的start方法内：
```
public final void start() {
    executePhase(this);
}
-------------------------
private void executePhase(SearchPhase phase) {
    try {
         phase.run();
    }
-------------------------
public final void run() {
    // 遍历所有的分片，然后执行:
    // 如果列表中有N个shard位于同一个节点，则向其发送N个请求，并不会把请求合并成一个。
    for (int index = 0; index < shardsIts.size(); index++) {
        final SearchShardIterator shardRoutings = shardsIts.get(index);
        assert shardRoutings.skip() == false;
        //对每个分片进行搜索查询
        performPhaseOnShard(index, shardRoutings, shardRoutings.nextOrNull());
    }
}	
```
performPhaseOnShard方法执行的是对每个shard的搜索过程。
```
private void performPhaseOnShard(final int shardIndex, final SearchShardIterator shardIt, final ShardRouting shard) {
    if (shard == null) {
        fork(() -> onShardFailure(shardIndex, null, null, shardIt, new NoShardAvailableActionException(shardIt.shardId())));
    } else {
        final PendingExecutions pendingExecutions = throttleConcurrentRequests ?
            pendingExecutionsPerNode.computeIfAbsent(shard.currentNodeId(), n -> new PendingExecutions(maxConcurrentRequestsPerNode))
            : null;
        Runnable r = () -> {
            final Thread thread = Thread.currentThread();
            try {
                // 定义Listener，用来处理搜索结果Response, 在单shard上搜索
                executePhaseOnShard(shardIt, shard,
                    new SearchActionListener<Result>(shardIt.newSearchShardTarget(shard.currentNodeId()), shardIndex) {
                        @Override
                        public void innerOnResponse(Result result) {
                            try {
                                //onShardResult方法，把查询结果记录在协调节点保存的数组结构results中，并增加计数
                                onShardResult(result, shardIt);
                            } finally {
                                executeNext(pendingExecutions, thread);
                            }
                        }
```
默认搜索类型对应的搜索实现是SearchQueryThenFetchAsyncAction，对应的executePhaseOnShard方法：
```
protected void executePhaseOnShard(final SearchShardIterator shardIt, final ShardRouting shard,
                                   final SearchActionListener<SearchPhaseResult> listener) {
    getSearchTransport().sendExecuteQuery(getConnection(shardIt.getClusterAlias(), shard.currentNodeId()),
        buildShardSearchRequest(shardIt), getTask(), listener);
}
```
然后对应的是SearchService的业务逻辑，核心依旧是RPC，对应的Action名字是：QUERY_ACTION_NAME
```
public void sendExecuteQuery(Transport.Connection connection, final ShardSearchRequest request, SearchTask task,
                             final SearchActionListener<SearchPhaseResult> listener) {
    // we optimize this and expect a QueryFetchSearchResult if we only have a single shard in the search request
    // this used to be the QUERY_AND_FETCH which doesn't exist anymore.
    final boolean fetchDocuments = request.numberOfShards() == 1;
    Writeable.Reader<SearchPhaseResult> reader = fetchDocuments ? QueryFetchSearchResult::new : QuerySearchResult::new;

    final ActionListener handler = responseWrapper.apply(connection, listener);
    transportService.sendChildRequest(connection, QUERY_ACTION_NAME, request, task,
            new ConnectionCountingHandler<>(handler, reader, clientConnections, connection.getNode().getId()));
}
```
和其他RPC Action一样，需要关注RPC的注册信息：
```
transportService.registerRequestHandler(QUERY_ACTION_NAME, ThreadPool.Names.SAME, ShardSearchRequest::new,
    (request, channel, task) -> {
        searchService.executeQueryPhase(request, (SearchTask) task, new ChannelActionListener<>(
            channel, QUERY_ACTION_NAME, request));
    });
```
## Data节点
接下来进入到查询阶段，Data节点的处理过程：
```
//Data节点执行搜索QueryPhase阶段
public void executeQueryPhase(ShardSearchRequest request, SearchTask task, ActionListener<SearchPhaseResult> listener) {
    //executeQueryPhase：执行搜索
    rewriteShardRequest(request, ActionListener.map(listener, r -> executeQueryPhase(r, task)));
}
-------------------------
private SearchPhaseResult executeQueryPhase(ShardSearchRequest request, SearchTask task) throws Exception {
//调用createAndPutContext创建context
final SearchContext context = createAndPutContext(request);
context.incRef();
try {
    context.setTask(task);
    final long afterQueryTime;
    try (SearchOperationListenerExecutor executor = new SearchOperationListenerExecutor(context)) {
        contextProcessing(context);
        //加载缓存或者查询lucene
        loadOrExecuteQueryPhase(request, context);
        //如果是hasScoreDocs&&包含Scorll查询，则释放search context
        if (context.queryResult().hasSearchContext() == false && context.scrollContext() == null) {
            freeContext(context.id());
        } else {
            contextProcessedSuccessfully(context);
        }
        afterQueryTime = executor.success();
    }
    //有一个成功，即执行FetchPhase
    if (request.numberOfShards() == 1) {
        return executeFetchPhase(context, afterQueryTime);
    }
    return context.queryResult();
}
```
loadOrExecuteQueryPhase功能是加载缓存或者查询lucene，结果放入SearchContext中。
```
private void loadOrExecuteQueryPhase(final ShardSearchRequest request, final SearchContext context) throws Exception {
    final boolean canCache = indicesService.canCache(request, context);
    context.getQueryShardContext().freezeContext();
    // 执行 search, 得到 DocId 信息，放入context中
    if (canCache) {
        indicesService.loadIntoContext(request, context, queryPhase);
    } else {
        queryPhase.execute(context);
    }
}
```
然后进入到QueryPhase类的具体实现方法：
```
public void execute(SearchContext searchContext) throws QueryPhaseExecutionException {
    
    final ContextIndexSearcher searcher = searchContext.searcher();
    //调用Lucene、searcher.search()实现搜索
    boolean rescore = execute(searchContext, searchContext.searcher(), searcher::setCheckCancelled);

    if (rescore) { // only if we do a regular search
        //全文检索且需要打分
        rescorePhase.execute(searchContext);
    }
    //自动补全及纠错
    suggestPhase.execute(searchContext);
    //实现聚合
    aggregationPhase.execute(searchContext);

    if (searchContext.getProfilers() != null) {
        ProfileShardResult shardResults = SearchProfileShardResults
                .buildShardResults(searchContext.getProfilers());
        searchContext.queryResult().profileResults(shardResults);
    }
}
```
Lucene搜索的核心在execute方法，对应的代码是：
```
try {
    //Lucene搜索核心, 调用lucene接口，执行真正的查询
    searcher.search(query, queryCollector);
} catch (EarlyTerminatingCollector.EarlyTerminationException e) {
    queryResult.terminatedEarly(true);
}
```
查询阶段完成后，在performPhaseOnShard方法的监听器内，收集结果。
```
private void performPhaseOnShard(final int shardIndex, final SearchShardIterator shardIt, final ShardRouting shard) {
    if (shard == null) {
        fork(() -> onShardFailure(shardIndex, null, null, shardIt, new NoShardAvailableActionException(shardIt.shardId())));
    } else {
        final PendingExecutions pendingExecutions = throttleConcurrentRequests ?
            pendingExecutionsPerNode.computeIfAbsent(shard.currentNodeId(), n -> new PendingExecutions(maxConcurrentRequestsPerNode))
            : null;
        Runnable r = () -> {
            final Thread thread = Thread.currentThread();
            try {
                // 定义Listener，用来处理搜索结果Response, 在单shard上搜索
                executePhaseOnShard(shardIt, shard,
                    new SearchActionListener<Result>(shardIt.newSearchShardTarget(shard.currentNodeId()), shardIndex) {
                        @Override
                        public void innerOnResponse(Result result) {
                            try {
                                //onShardResult方法，把查询结果记录在协调节点保存的数组结构results中，并增加计数
                                onShardResult(result, shardIt);
```
onShardResult方法是取回阶段的入口。

## 取回阶段
分布式阶段由以下步骤构成：

协调节点辨别出哪些文档需要被取回并向相关的分片提交多个 GET 请求。
每个分片加载并 丰富 文档，如果有需要的话，接着返回文档给协调节点。
一旦所有的文档都被取回了，协调节点返回结果给客户端。
协调节点首先决定哪些文档 确实 需要被取回。例如，如果我们的查询指定了 { "from": 90, "size": 10 } ，最初的90个结果会被丢弃，只有从第91个开始的10个结果需要被取回。这些文档可能来自和最初搜索请求有关的一个、多个甚至全部分片。

协调节点给持有相关文档的每个分片创建一个 multi-get request ，并发送请求给同样处理查询阶段的分片副本。

分片加载文档体-- _source 字段—​如果有需要，用元数据和 search snippet highlighting 丰富结果文档。 一旦协调节点接收到所有的结果文档，它就组装这些结果为单个响应返回给客户端。

## 协调者节点
把查询结果记录在协调节点保存的数组结构results中，并增加计数：
```
private void onShardResult(Result result, SearchShardIterator shardIt) {
    assert result.getShardIndex() != -1 : "shard index is not set";
    assert result.getSearchShardTarget() != null : "search shard target must not be null";
    //成功计数
    successfulOps.incrementAndGet();
    //searchPhaseController.newSearchPhaseResults(request, shardsIts.size())
    results.consumeResult(result);
    AtomicArray<ShardSearchFailure> shardFailures = this.shardFailures.get();
    if (shardFailures != null) {
        shardFailures.set(result.getShardIndex(), null);
    }
    successfulShardExecution(shardIt);
}
```
在接下来的successfulShardExecution方法内，进入取回阶段
```
private void successfulShardExecution(SearchShardIterator shardsIt) {
    final int remainingOpsOnIterator;
    if (shardsIt.skip()) {
        remainingOpsOnIterator = shardsIt.remaining();
    } else {
        remainingOpsOnIterator = shardsIt.remaining() + 1;
    }
    final int xTotalOps = totalOps.addAndGet(remainingOpsOnIterator);
    // 检查是否收到全部回复
    if (xTotalOps == expectedTotalOps) {
        //当返回结果的分片数等于预期的总分片数时，协调节点会进入当前Phase的结束处理，启动下一个阶段Fetch Phase的执行。
        // 注意，ES中只需要一个分片执行成功，就会进行后续Phase处理得到部分结果，当然它会在结果中提示用户实际有多少分片执行成功。
        onPhaseDone();
    }
```
onPhaseDone执行取回阶段，getNextPhase不同的搜索方式有不同的实现。
```
final void onPhaseDone() {  // as a tribute to @kimchy aka. finishHim()
      // SearchQueryThenFetchAsyncAction中getNextPhase会返回：FetchSearchPhase
      executeNextPhase(this, getNextPhase(results, this));
 }
```
executePhase中入参nextPhase对应的是FetchSearchPhase
```
public final void executeNextPhase(SearchPhase currentPhase, SearchPhase nextPhase) {  
    if (successfulOps.get() == 0) { // we have 0 successful results that means we shortcut stuff and return a failure
        //没有成功请求，截断，返回给客户都异常
        final ShardOperationFailedException[] shardSearchFailures = ExceptionsHelper.groupBy(buildShardFailures());
        Throwable cause = shardSearchFailures.length == 0 ? null :
            ElasticsearchException.guessRootCauses(shardSearchFailures[0].getCause())[0];
        logger.debug(() -> new ParameterizedMessage("All shards failed for phase: [{}]", getName()), cause);
        onPhaseFailure(currentPhase, "all shards failed", cause);
    } else {
        Boolean allowPartialResults = request.allowPartialSearchResults();
        assert allowPartialResults != null : "SearchRequest missing setting for allowPartialSearchResults";
        //判断释放只有部分shard请求成功
        if (allowPartialResults == false && successfulOps.get() != getNumShards()) {
        .....................
       	
       //默认nextPhase对应SearchQueryThenFetchAsyncAction的getNextPhase方法是FetchSearchPhase
        executePhase(nextPhase);
    }
}
----------------------------------
private void executePhase(SearchPhase phase) {
    try {
        phase.run();
    }
```
接下来进入到FetchSearchPahse的run方法：
```
public void run() {
    context.execute(new AbstractRunnable() {
        @Override
        protected void doRun() throws Exception {
            innerRun();
        }
-----------------------
private void innerRun() throws IOException {
........................
ScoreDoc[] scoreDocs = reducedQueryPhase.sortedTopDocs.scoreDocs;
final IntArrayList[] docIdsToLoad = searchPhaseController.fillDocIdsToLoad(numShards, scoreDocs);
if (scoreDocs.length == 0) { // no docs to fetch -- sidestep everything and return
    phaseResults.stream()
        .map(SearchPhaseResult::queryResult)
        .forEach(this::releaseIrrelevantSearchContext); // we have to release contexts here to free up resources
    finishPhase.run();
} else {
    final ScoreDoc[] lastEmittedDocPerShard = isScrollSearch ?
        searchPhaseController.getLastEmittedDocPerShard(reducedQueryPhase, numShards)
        : null;
    final CountedCollector<FetchSearchResult> counter = new CountedCollector<>(r -> fetchResults.set(r.getShardIndex(), r),
        docIdsToLoad.length, // we count down every shard in the result no matter if we got any results or not
        finishPhase, context);
    // 从查询阶段的shard列表中遍历，跳过查询结果为空的shard，
    // 对特定目标shard执行executeFetch方法来获取数据，其中包括分页信息。
    for (int i = 0; i < docIdsToLoad.length; i++) {
        IntArrayList entry = docIdsToLoad[i];
        SearchPhaseResult queryResult = queryResults.get(i);
        if (entry == null) { // no results for this shard ID
            if (queryResult != null) {
                // if we got some hits from this shard we have to release the context there
                // we do this as we go since it will free up resources and passing on the request on the
                // transport layer is cheap.
                releaseIrrelevantSearchContext(queryResult.queryResult());
            }
            // in any case we count down this result since we don't talk to this shard anymore
            counter.countDown();
        } else {
            SearchShardTarget searchShardTarget = queryResult.getSearchShardTarget();
            //获取链接
            Transport.Connection connection = context.getConnection(searchShardTarget.getClusterAlias(),
                searchShardTarget.getNodeId());
            //创建请求ShardFetchSearchRequest
            ShardFetchSearchRequest fetchSearchRequest = createFetchRequest(queryResult.queryResult().getRequestId(), i, entry,
                lastEmittedDocPerShard, searchShardTarget.getOriginalIndices());
            //执行FetchPhase阶段
            executeFetch(i, searchShardTarget, counter, fetchSearchRequest, queryResult.queryResult(),connection);
        }
    }		
```
在Query查询阶段，获取结果为空，则认为是失败。查询成功的请求，执行executeFetch方法。
```
// executeFetch的参数querySearchResult包含分页信息，
// 同时定义了Listener，每成功获取一个shard数据之后就执行counter.onResult，
// 其中调用对结果的处理回调(final CountedCollector<FetchSearchResult> counter)，把结果保存到数组中，然后执行countDown。
private void executeFetch(final int shardIndex, final SearchShardTarget shardTarget,
                          final CountedCollector<FetchSearchResult> counter,
                          final ShardFetchSearchRequest fetchSearchRequest, final QuerySearchResult querySearchResult,
                          final Transport.Connection connection) {
    context.getSearchTransport().sendExecuteFetch(connection, fetchSearchRequest, context.getTask(),
        new SearchActionListener<FetchSearchResult>(shardTarget, shardIndex) {
            @Override
            public void innerOnResponse(FetchSearchResult result) {
                try {
                    counter.onResult(result);
                } catch (Exception e) {
                    context.onPhaseFailure(FetchSearchPhase.this, "", e);
                }
            }
```
这里仍旧是分两个逻辑：（1）sendExecuteFetch发送RPC请求到Data数据节点；（2）监听器监听成功响应，执行counter.onResult处理监听结果。

RPC Action对应的是FETCH_ID_ACTION_NAME，执行RPC请求：
```
public void sendExecuteFetch(Transport.Connection connection, final ShardFetchSearchRequest request, SearchTask task,
                             final SearchActionListener<FetchSearchResult> listener) {
    sendExecuteFetch(connection, FETCH_ID_ACTION_NAME, request, task, listener);
}
```
和其他RPC Action一样，需要关注RPC的注册信息：

```
transportService.registerRequestHandler(FETCH_ID_ACTION_NAME, ThreadPool.Names.SAME, true, true, ShardFetchSearchRequest::new,
    (request, channel, task) -> {
        searchService.executeFetchPhase(request, (SearchTask)task, new ChannelActionListener<>(channel, FETCH_ID_ACTION_NAME,
            request));
    });
```

## Data节点
Data节点响应FETCH_ID_ACTION_NAME Action对应的逻辑是searchService.executeFetchPhase方法。
```
public void executeFetchPhase(ShardFetchRequest request, SearchTask task, ActionListener<FetchSearchResult> listener) {
    runAsync(request.id(), () -> {
        final SearchContext context = findContext(request.id(), request);
        context.incRef();
        try {
            context.setTask(task);
            contextProcessing(context);
            if (request.lastEmittedDoc() != null) {
                context.scrollContext().lastEmittedDoc = request.lastEmittedDoc();
            }
            context.docIdsToLoad(request.docIds(), 0, request.docIdsSize());
            try (SearchOperationListenerExecutor executor = new SearchOperationListenerExecutor(context, true, System.nanoTime())) {
                //Data节点执行搜索，结果放入context中
                fetchPhase.execute(context);
                //ScrollContext或ScrollContext为空则释放Context
                if (fetchPhaseShouldFreeContext(context)) {
                    freeContext(request.id());
                } else {
                    contextProcessedSuccessfully(context);
                }
                executor.success();
            }
            return context.fetchResult();
        }
```
接下来进入到FetchPhase类的execute方法，根据DocId获取真实的文档：
```
//查询到docId后，查询字段信息过程
@Override
public void execute(SearchContext context) {
    .............................
    try {
        SearchHit[] hits = new SearchHit[context.docIdsToLoadSize()];
        FetchSubPhase.HitContext hitContext = new FetchSubPhase.HitContext();
        for (int index = 0; index < context.docIdsToLoadSize(); index++) {
            if (context.isCancelled()) {
                throw new TaskCancelledException("cancelled");
            }
            int docId = context.docIdsToLoad()[context.docIdsToLoadFrom() + index];
            int readerIndex = ReaderUtil.subIndex(docId, context.searcher().getIndexReader().leaves());
            //获取Reader上下文,Engine.Searcher.getIndexReader()
            LeafReaderContext subReaderContext = context.searcher().getIndexReader().leaves().get(readerIndex);
            int subDocId = docId - subReaderContext.docBase;

            final SearchHit searchHit;
            //遍历每个要fetch的文档，判断文档是否为Nested结构，然后分别调用createNestedSearchHit()或者createSearchHit()得到SearchHit
            int rootDocId = findRootDocumentIfNested(context, subReaderContext, subDocId);
            if (rootDocId != -1) {
                searchHit = createNestedSearchHit(context, docId, subDocId, rootDocId,
                    storedToRequestedFields, subReaderContext);
            } else {
                searchHit = createSearchHit(context, fieldsVisitor, docId, subDocId,
                    storedToRequestedFields, subReaderContext);
            }

            hits[index] = searchHit;
            hitContext.reset(searchHit, subReaderContext, subDocId, context.searcher());
            //获取SearchHit后再补充如下阶段的结果：ScriptFieldsPhase, PartialFieldsPhase, MatchedQueriesPhase等等
            for (FetchSubPhase fetchSubPhase : fetchSubPhases) {
                fetchSubPhase.hitExecute(context, hitContext);
            }
        }
        if (context.isCancelled()) {
            throw new TaskCancelledException("cancelled");
        }

        for (FetchSubPhase fetchSubPhase : fetchSubPhases) {
            fetchSubPhase.hitsExecute(context, hits);
            if (context.isCancelled()) {
                throw new TaskCancelledException("cancelled");
            }
        }

        TotalHits totalHits = context.queryResult().getTotalHits();
        //将hit结果集hits、全部命中结果数totalHits和最大得分maxScore放入context.fetchResult().hits对象中
        context.fetchResult().hits(new SearchHits(hits, totalHits, context.queryResult().getMaxScore()));
    } catch (IOException e) {
```
createSearchHit中包含Lucene的搜索过程。
```
private SearchHit createSearchHit(SearchContext context,
                                  FieldsVisitor fieldsVisitor,
                                  int docId,
                                  int subDocId,
                                  Map<String, Set<String>> storedToRequestedFields,
                                  LeafReaderContext subReaderContext) {
    DocumentMapper documentMapper = context.mapperService().documentMapper();
    Text typeText = documentMapper.typeText();
    if (fieldsVisitor == null) {
        return new SearchHit(docId, null, typeText, null);
    }

    //在创建SearchHit时，使用loadStoredFields从lucene中获取已经存储的字段信息
    Map<String, DocumentField> searchFields = getSearchFields(context, fieldsVisitor, subDocId,
        storedToRequestedFields, subReaderContext);
}
-----------------------------------
//从lucene中获取已经存储的字段信息
private Map<String, DocumentField> getSearchFields(SearchContext context,
                                                       FieldsVisitor fieldsVisitor,
                                                       int subDocId,
                                                       Map<String, Set<String>> storedToRequestedFields,
                                                       LeafReaderContext subReaderContext) {
        //从lucene中获取已经存储的字段信息
        loadStoredFields(context.shardTarget(), subReaderContext, fieldsVisitor, subDocId);
```
使用loadStoredFields从lucene中获取已经存储的字段信息
```
private void loadStoredFields(SearchShardTarget shardTarget, LeafReaderContext readerContext, FieldsVisitor fieldVisitor, int docId) {
    fieldVisitor.reset();
    try {
        //使用loadStoredFields从lucene中获取已经存储的字段信息
        //ElasticsearchDirectoryReader继承BaseCompositeReader，BaseCompositeReader实现document方法
        readerContext.reader().document(docId, fieldVisitor);
    }
```
readerContext.reader()对应的是ElasticsearchDirectoryReader，document方法的具体实现在BaseCompositeReader对应的类图关系如下：

Search操作相对Get复杂的多，这里还有个更重要的逻辑是评分的计算，当前版本ES使用的评分计算方式是BM25。

另一个影响评分的重要因素是搜索Field的权重，不同Field赋值不同的权重，可以使得搜素结果更加符合需求。
