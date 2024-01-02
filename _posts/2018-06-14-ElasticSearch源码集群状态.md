---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码集群状态
ES集群元数据中封装在ClusterState 类，通过cluster/state API来获取集群状态。
```
curl -X GET "localhost: 9200/_cluster/state"
```

## 客户端获取集群状态
当客户端通过发送Http请求或发送Curl命令到ES的服务端，服务端的响应过程在。如果发送cluster/state API，对应的流程图如下：

### RestClusterStateAction
获取集群状态对应的Rest**Action是RestClusterStateAction，最后发起请求对应的代码如下
```
return channel -> client.admin().cluster().state(clusterStateRequest, new RestBuilderListener<ClusterStateResponse>(channel) {
    @Override
    public RestResponse buildResponse(ClusterStateResponse response, XContentBuilder builder) throws Exception {
        builder.startObject();
        if (clusterStateRequest.waitForMetaDataVersion() != null) {
            builder.field(Fields.WAIT_FOR_TIMED_OUT, response.isWaitForTimedOut());
        }
        builder.field(Fields.CLUSTER_NAME, response.getClusterName().value());
        response.getState().toXContent(builder, request);
        builder.endObject();
        return new BytesRestResponse(RestStatus.OK, builder);
    }
});
```
调用AbstractClient的state方法，这里注意接下来execute方法的入参action是ClusterStateAction.INSTANCE 。
```
public void state(final ClusterStateRequest request, final ActionListener<ClusterStateResponse> listener) {
    execute(ClusterStateAction.INSTANCE, request, listener);
}
```
接下来按照流程分析调用链在NodeClient的executeLocally方法，如下：
```
public <    Request extends ActionRequest,
            Response extends ActionResponse
        > Task executeLocally(ActionType<Response> action, Request request, ActionListener<Response> listener) {
    return transportAction(action).execute(request, listener);
}
```
transportAction(action)根据action获取TransportAction的实现类，ClusterStateAction.INSTANCE对应的action是TransportClusterStateAction。

### TransportClusterStateAction
TransportClusterStateAction继承TransportMasterNodeReadAction类，需要Master节点才能操作，对应的masterOperation方法核心代码：
```
protected void masterOperation(final ClusterStateRequest request, final ClusterState state,
                               final ActionListener<ClusterStateResponse> listener) throws IOException {

    //wait_for_metadata_version集群元数据的版本需要大于等于上次版本
    final Predicate<ClusterState> acceptableClusterStatePredicate
        = request.waitForMetaDataVersion() == null ? clusterState -> true
        : clusterState -> clusterState.metaData().version() >= request.waitForMetaDataVersion();

    //request.local()在/_cluster/state API中是否使用lcoal属性（lcoal：true返回当前节点对应的ClusterState,false返回Master节点的ClusterState）
    final Predicate<ClusterState> acceptableClusterStateOrNotMasterPredicate = request.local()
        ? acceptableClusterStatePredicate
        : acceptableClusterStatePredicate.or(clusterState -> clusterState.nodes().isLocalNodeElectedMaster() == false);

    if (acceptableClusterStatePredicate.test(state)) {
        ActionListener.completeWith(listener, () -> buildResponse(request, state));
    } else {
        assert acceptableClusterStateOrNotMasterPredicate.test(state) == false;
        new ClusterStateObserver(state, clusterService, request.waitForTimeout(), logger, threadPool.getThreadContext())
            .waitForNextChange(new ClusterStateObserver.Listener() {

            @Override
            public void onNewClusterState(ClusterState newState) {
                if (acceptableClusterStatePredicate.test(newState)) {
                    ActionListener.completeWith(listener, () -> buildResponse(request, newState));
                } else {
                    listener.onFailure(new NotMasterException(
                        "master stepped down waiting for metadata version " + request.waitForMetaDataVersion()));
                }
       .........................
```

### ClusterStateObserver
ClusterStateObserver类观测到集群状态发生变更，并通知对应的监听器，其中waitForNextChange方法核心代码：
```
/**
 * Wait for the next cluster state which satisfies statePredicate
 *
 * @param listener        callback listener
 * @param statePredicate predicate to check whether cluster state changes are relevant and the callback should be called
 * @param timeOutValue    a timeout for waiting. If null the global observer timeout will be used.
 */
public void waitForNextChange(Listener listener, Predicate<ClusterState> statePredicate, @Nullable TimeValue timeOutValue) {
    listener = new ContextPreservingListener(listener, contextHolder.newRestorableContext(false));

    //timeout相关参数设置
    Long timeoutTimeLeftMS;
     .......................    

    // sample a new state. This state maybe *older* than the supplied state if we are called from an applier,
    // which wants to wait for something else to happen
    ClusterState newState = clusterApplierService.state();
    //集群状态发生变更
    if (lastObservedState.get().isOlderOrDifferentMaster(newState) && statePredicate.test(newState)) {
        // good enough, let's go.
        logger.trace("observer: sampled state accepted by predicate ({})", newState);
        //存储新的集群状态
        lastObservedState.set(new StoredState(newState));
        //listener对应：ContextPreservingListener
        listener.onNewClusterState(newState);
    } else {
        logger.trace("observer: sampled state rejected by predicate ({}). adding listener to ClusterService", newState);
        final ObservingContext context = new ObservingContext(listener, statePredicate);
        if (!observingContext.compareAndSet(null, context)) {
            throw new ElasticsearchException("already waiting for a cluster state change");
        }
        clusterApplierService.addTimeoutListener(timeoutTimeLeftMS == null ?
            null : new TimeValue(timeoutTimeLeftMS), clusterStateListener);
    }
}
```
主要是根据集群状态是否发生变更执行不同的判断条件，若集群状态发生变更则通知监听器；若集群状态未发生变更，则在clusterApplierService增加一个超时监听器；

## onNewClusterState
集群状态发生变更，把新的ClusterState通知监听器返回给Client。
```
public void onNewClusterState(ClusterState state) {
    try (ThreadContext.StoredContext context  = contextSupplier.get()) {
        delegate.onNewClusterState(state);
    }
}
----------------------------------------
TransportClusterStateAction#masterOperation的监听器实现
.waitForNextChange(new ClusterStateObserver.Listener() {

@Override
public void onNewClusterState(ClusterState newState) {
    if (acceptableClusterStatePredicate.test(newState)) {
        ActionListener.completeWith(listener, () -> buildResponse(request, newState));
    } else {
        listener.onFailure(new NotMasterException(
            "master stepped down waiting for metadata version " + request.waitForMetaDataVersion()));
    }
}
----------------------------------------
private ClusterStateResponse buildResponse(final ClusterStateRequest request,
                                           final ClusterState currentState) {
    logger.trace("Serving cluster state request using version {}", currentState.version());
    ClusterState.Builder builder = ClusterState.builder(currentState.getClusterName());
    builder.version(currentState.version());
    builder.stateUUID(currentState.stateUUID());
    builder.minimumMasterNodesOnPublishingMaster(currentState.getMinimumMasterNodesOnPublishingMaster());

    if (request.nodes()) {
        builder.nodes(currentState.nodes());
    }
    if (request.routingTable()) {
        ............................
    }
    if (request.blocks()) {
        builder.blocks(currentState.blocks());
    }

    MetaData.Builder mdBuilder = MetaData.builder();
    mdBuilder.clusterUUID(currentState.metaData().clusterUUID());
    mdBuilder.coordinationMetaData(currentState.coordinationMetaData());

    if (request.metaData()) {
        ............................
    }
    builder.metaData(mdBuilder);

    if (request.customs()) {
        ............................
    }

    return new ClusterStateResponse(currentState.getClusterName(), builder.build(), false);
}
```
buildResponse方法根据ClusterStateRequest的不同参数设置，增加不同的返回结果。

### addTimeoutListener
集群状态未发生变更，增加一个超时监听器，等待其他操作完成查看集群状态是否有变化。

addTimeoutListener方法的入参TimeoutClusterStateListener的最重要的实现是clusterChanged方法：
```
public void clusterChanged(ClusterChangedEvent event) {
    ObservingContext context = observingContext.get();
     .......................
    final ClusterState state = event.state();
    if (context.statePredicate.test(state)) {
        if (observingContext.compareAndSet(context, null)) {
            clusterApplierService.removeTimeoutListener(this);
            logger.trace("observer: accepting cluster state change ({})", state);
            //集群状态存储在lastObservedState
            lastObservedState.set(new StoredState(state));
            //通知监听器集群状态发生变更
            context.listener.onNewClusterState(state);
        }
```
addTimeoutListener方法内会定时执行任务检测是否超时，这个定时任务的优先级是最高（Priority.HIGH）
```
public void addTimeoutListener(@Nullable final TimeValue timeout, final TimeoutClusterStateListener listener) {
    .......................
    // call the post added notification on the same event thread
    try {
        threadPoolExecutor.execute(new SourcePrioritizedRunnable(Priority.HIGH, "_add_listener_") {
            @Override
            public void run() {
                if (timeout != null) {
                    NotifyTimeout notifyTimeout = new NotifyTimeout(listener, timeout);
                    notifyTimeout.cancellable = threadPool.schedule(notifyTimeout, timeout, ThreadPool.Names.GENERIC);
                    onGoingTimeouts.add(notifyTimeout);
                }
                timeoutClusterStateListeners.add(listener);
                listener.postAdded();
            }
        });
```

## 集群元数据
通过cluster/state API来获取集群状态，得到到返回结果进行JSON格式化，与ClusterState类对比：

Cluster是集群级别的元数据信息，有以下几个有复杂的结构

1. DiscoveryNodes nodes

DiscoveryNodes 是节点级别的元数据信息。

DiscoveryNodes 对象描述的是集群的节点信息。里面包含如下主要的成员信息：
```
// 当前 master
private final String masterNodeId;
// 本地节点
private final String localNodeId;
// 完整的节点列表
private final ImmutableOpenMap<String, DiscoveryNode> nodes;

// 其它分类型节点列表
private final ImmutableOpenMap<String, DiscoveryNode> dataNodes;
private final ImmutableOpenMap<String, DiscoveryNode> masterNodes;
private final ImmutableOpenMap<String, DiscoveryNode> ingestNodes;
```
最后按照不同节点类型，把节点列表分类为不同类型的节点列表。

## MetaData metaData

MetaData 包含集群全局的配置信息。
- 索引列表
IndexMetaData主要是索引的Settings Mappings aliases等信息
```
//Meta of all Indexes
private final ImmutableOpenMap<String, IndexMetaData> indices;
```
- 索引模板
```
//Meta of all templates
private final ImmutableOpenMap<String, IndexTemplateMetaData> templates;
```
- index-graveyard
```
//索引墓碑。记录已删除的索引，并保存一段时间。索引删除是主节点通过下发集群状态来执行的
//各节点处理集群状态是异步的过程。例如，索引分片分布在5个节点上，删除索引期间，某个节点是“down”掉的，没有执行删除逻辑
//当这个节点恢复的时候，其存储的已删除的索引会被当作孤立资源加入集群,索引死而复活。
//墓碑的作用就是防止发生这种情况
public static final String TYPE = "index-graveyard";
//已删除的索引列表
private static final ParseField TOMBSTONES_FIELD = new ParseField("tombstones");
```
- CoordinationMetaData
```
集群选举管理信息
private final CoordinationMetaData coordinationMetaData;
----------------------------------------------------------
// raft 选举使用的 term
private final long term;

// 最近提交的选举节点列表，内部是一个 Set<String> nodeIds
private final VotingConfiguration lastCommittedConfiguration;

// 最近接收的选举节点列表
private final VotingConfiguration lastAcceptedConfiguration;

// 用户通过 _cluster/voting_config_exclusions 接口设定的选举排除节点列表
private final Set<VotingConfigExclusion> votingConfigExclusions;
```
- RoutingTable routingTable

RoutingTable 对应的则是索引分片的路由信息，包含主要的数据路由信息，在查询、写入请求的时候提供分片到节点的路由，动态构建不会持久化。

一个 RoutingTable 包含集群的所有索引，一个索引包含多个分片，一个分片包含一个主、多个从本分片，最底层的 ShardRouting 描述具体的某个副本分片所在的节点以及正在搬迁的目标节点信息。用户请求时只指定索引信息，请求到达协调节点，由协调节点根据该路由表来获取底层分片所在节点并转发请求。

接下来看看对应的数据结构，RoutingTable 顶层主要的成员只有一个：
```
// key 为索引名，IndexRoutingTable 为该索引包含的分片信息
private final ImmutableOpenMap<String, IndexRoutingTable> indicesRouting;
```
IndexRoutingTable 对象：
```
// 索引信息，包含索引名称、uuid 等。
private final Index index;

// 索引的分片列表，key 为分片的编号，例如 2 号分片，3 号分片
private final ImmutableOpenIntMap<IndexShardRoutingTable> shards;
```
一套分片信息包含一个主、多个从分片，其对象结构 IndexShardRoutingTable：
```
// 当前分片信息，主要包含分片的编号、分片对应的索引信息
final ShardId shardId;

// 当前分片的主分片信息
final ShardRouting primary;

// 当前分片的多个从分片列表
final List<ShardRouting> replicas;

// 当前分片全量的分片列表，包含主、副本
final List<ShardRouting> shards;

// 当前分片已经 started 的活跃分片列表
final List<ShardRouting> activeShards;

// 当前分片已经分配了节点的分片列表
final List<ShardRouting> assignedShards;
```
分片最底层的数据结构是 ShardRouting ，它描述这个分片的状态、节点归属等信息：
```
private final ShardId shardId;

// 分片所在的当前节点 id
private final String currentNodeId;

// 如果分片正在搬迁，这里为目标节点 id
private final String relocatingNodeId;

// 是否是主分片
private final boolean primary;

// 分片的状态，UNASSIGNED/INITIALIZING/STARTED/RELOCATING
private final ShardRoutingState state;

// 每一个分片分配后都有一个唯一标识
private final AllocationId allocationId;
```
RoutingNodes routingNodes

RoutingNodes 为节点到分片的映射关系。主要用于统计每个节点的分片分配情况，在分片分配、均衡的时候使用，需要时根据 RoutingTable 动态构建，不会持久化。

RoutingNodes 的主要成员包括：
```
// 节点到分片的映射，key 为 nodeId，value 为一个包含分片列表的节点信息
private final Map<String, RoutingNode> nodesToShards = new HashMap<>();

// 未分配的分片列表，每次 reroute 从这里获取分片分配
private final UnassignedShards unassignedShards = new UnassignedShards(this);

// 已经分配的分片列表
private final Map<ShardId, List<ShardRouting>> assignedShards = new HashMap<>();

// 当前分片远程恢复的统计数据 incoming/outcoming
private final Map<String, Recoveries> recoveriesPerNode = new HashMap<>();
```
ES集群元数据信息事实应该比这些多，这里不是全部，用到的时候再找吧。

### 集群元数据持久化
元数据内存结构中，只有部分信息会被持久化存储到磁盘上，其它结构都是在节点启动后动态构建或者直接在内存中动态维护的。持久化的内容包含两部分，索引元数据和集群元数据也叫全局元数据，下面我们分别来介绍这两部分持久化的内容。

### 索引元数据（index metadata）
索引元数据维护的是各个索引独有的配置信息，持久化的内容主要包括：

in_sync_allocations：每个分片分配之后都有一个唯一的 allocationId，该列表是主分片用来维护被认为跟自己保持数据一致的副本列表。从这个列表中踢出的分片不会接收新的写入，需要走分片恢复流程将数据追齐之后才能再被放入该队列。
mappings：索引的 mapping 信息，定义各个字段的类型、属性。
settings：索引自己的配置信息。
state：索引状态，OPEN/CLOSED。
aliases：索引别名信息。
routing_num_shards：索引分片数量。
primary_terms：每一轮分片切主都会产生新的 primary term，用于保持主从分片之间的一致性。

### 集群元数据（global metadata）
全局元数据持久化的内容主要是前面描述的 MetaData 对象中去除 indices 索引部分。包括动态配置、模板信息、选举信息等。主要包含三部分：

manifest-数字.st：该文件是一个磁盘元数据管理入口，主要管理元数据的版本信息。
global-数字.st：MetaData 中的全局元数据的主要信息就是持久化到这个文件，包括动态配置、模板信息等。
node-数字.st：当前节点的元数据信息，包括节点的 nodeId 和版本信息。

### 元数据文件分布
在 7.6.0 版本之前，元数据直接存放在每个节点磁盘上，且在专有 master 节点上每个索引一个元数据目录。7.6.0 之前持久化的数据目录及数据包含的内容：
```
//索引元数据目录结构
${data.path}/nodes/${node.id}/indices/${index.UUID}/${shard.id}/_state/state-xxx.st

//集群元数据目录结构
${data.path}/nodes/${node.id}/_state/node-xxx.st
```

### 集群元数据发布
集群状态ClusterState对应的元数据只有Master节点才能更新，集群状态发布是Master节点更新集群的元数据后发布给同一集群内的其他节点。集群状态的发布涉及publish_state和commit_state两个核心流程，对应的逻辑：

publish_state，Master节点向其他节点发送publis_state的请求；
其他节点收到publis_state请求，处于newly-received状态，并发送ack给Master；
commit_state，Master节点等待其他节点处理结果，收到的结果足够的正常响应结果，发送commit_state请求；
其他节点收到commit_state请求，执行now-committed更新集群状态，并发送ack给Master；
master节点监听结果并处理；
Master节点与其他节点的交互，显然是RPC请求，对应两个Action：
```
//7.X之后
public static final String PUBLISH_STATE_ACTION_NAME = "internal:cluster/coordination/publish_state";
//7.X之后
public static final String COMMIT_STATE_ACTION_NAME = "internal:cluster/coordination/commit_state";
```
RPC请求对应的Handler的注册
```
public PublicationTransportHandler(TransportService transportService, NamedWriteableRegistry namedWriteableRegistry,
                                   Function<PublishRequest, PublishWithJoinResponse> handlePublishRequest,
                                   BiConsumer<ApplyCommitRequest, ActionListener<Void>> handleApplyCommit) {
    this.transportService = transportService;
    this.namedWriteableRegistry = namedWriteableRegistry;
    this.handlePublishRequest = handlePublishRequest;

    //7.X之后
    transportService.registerRequestHandler(PUBLISH_STATE_ACTION_NAME, ThreadPool.Names.GENERIC, false, false,
        BytesTransportRequest::new, (request, channel, task) -> channel.sendResponse(handleIncomingPublishRequest(request)));

    //7.X之后
    transportService.registerRequestHandler(COMMIT_STATE_ACTION_NAME, ThreadPool.Names.GENERIC, false, false,
        ApplyCommitRequest::new,
        (request, channel, task) -> handleApplyCommit.accept(request, transportCommitCallback(channel)));
}
```
简述了Hander注册方法registerRequestHandler。

action：PUBLISH_STATE_ACTION_NAME对应的handler是(request, channel, task)-> channel.sendResponse(handleIncomingPublishRequest(request)
action：COMMIT_STATE_ACTION_NAME对应的handler是(request, channel, task)-> handleApplyCommit.accept(request,transportCommitCallback(channel)

































