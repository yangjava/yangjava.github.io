---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码重平衡
GatewayAllocator的补充，针对上一小节涉及shard分配及节点之间的RPC请求、PrimaryShard选举、数据复制等操作。

GatewayAllocator的核心方法如下，涉及primaryShardAllocator和replicaShardAllocator的allocateUnassigned方法。
```
protected static void innerAllocatedUnassigned(RoutingAllocation allocation,
                                               PrimaryShardAllocator primaryShardAllocator,
                                               ReplicaShardAllocator replicaShardAllocator) {
    //找到unassigned shard
    RoutingNodes.UnassignedShards unassigned = allocation.routingNodes().unassigned();
    unassigned.sort(PriorityComparator.getAllocationComparator(allocation)); // sort for priority ordering
    //主分片分配器
    primaryShardAllocator.allocateUnassigned(allocation);
    if (allocation.routingNodes().hasInactiveShards()) {
        // cancel existing recoveries if we have a better match
        replicaShardAllocator.processExistingRecoveries(allocation);
    }
    //副分片分配器
    replicaShardAllocator.allocateUnassigned(allocation);
}
```

## BaseGatewayShardAllocator
primaryShardAllocator和replicaShardAllocator的allocateUnassigned调用父类BaseGatewayShardAllocator的方法，两者的区别在 makeAllocationDecision，有不同的实现。
```
public void allocateUnassigned(RoutingAllocation allocation) {
    final RoutingNodes routingNodes = allocation.routingNodes();
    final RoutingNodes.UnassignedShards.UnassignedIterator unassignedIterator = routingNodes.unassigned().iterator();
    //遍历未分配的分片
    while (unassignedIterator.hasNext()) {
        final ShardRouting shard = unassignedIterator.next();
        //调用主分片分配器的makeAllocationDecision进行决策
        final AllocateUnassignedDecision allocateUnassignedDecision = makeAllocationDecision(shard, allocation, logger);

        if (allocateUnassignedDecision.isDecisionTaken() == false) {
            // no decision was taken by this allocator
            continue;
        }

        //根据决策器结果决定是否初始化分片
        if (allocateUnassignedDecision.getAllocationDecision() == AllocationDecision.YES) {
            //主副分片执行同的 unassigned!terator.initialize 函数，将分片的 unassigned 状态改为 initialize 状态
            unassignedIterator.initialize(allocateUnassignedDecision.getTargetNode().getId(),
                allocateUnassignedDecision.getAllocationId(),
                shard.primary() ? ShardRouting.UNAVAILABLE_EXPECTED_SHARD_SIZE :
                                  allocation.clusterInfo().getShardSize(shard, ShardRouting.UNAVAILABLE_EXPECTED_SHARD_SIZE),
                allocation.changes());
        } else {
            unassignedIterator.removeAndIgnore(allocateUnassignedDecision.getAllocationStatus(), allocation.changes());
        }
    }
}
```
GatewayAllocator类的PrimaryShardAllocator primaryShardAllocator和ReplicaShardAllocator replicaShardAllocator属性初始化：
```
public GatewayAllocator(RerouteService rerouteService,
                        TransportNodesListGatewayStartedShards startedAction,
                        TransportNodesListShardStoreMetaData storeAction) {
    this.rerouteService = rerouteService;
    this.primaryShardAllocator = new InternalPrimaryShardAllocator(startedAction);
    this.replicaShardAllocator = new InternalReplicaShardAllocator(storeAction);
}
```
## PrimaryShardAllocator
PrimaryShard的决策结果流程如下：
```
public AllocateUnassignedDecision makeAllocationDecision(final ShardRouting unassignedShard,
                                                         final RoutingAllocation allocation,
                                                         final Logger logger) {
    final boolean explain = allocation.debugDecision();
    //1.向其他Node的shard异步拉取数据
    final FetchResult<NodeGatewayStartedShards> shardState = fetchData(unassignedShard, allocation);
    if (shardState.hasData() == false) {
        //拉取失败，设置标志位
        allocation.setHasPendingAsyncFetch();
        List<NodeAllocationResult> nodeDecisions = null;
        if (explain) {
            nodeDecisions = buildDecisionsForAllNodes(unassignedShard, allocation);
        }
        return AllocateUnassignedDecision.no(AllocationStatus.FETCHING_SHARD_DATA, nodeDecisions);
    }

    // don't create a new IndexSetting object for every shard as this could cause a lot of garbage
    // on cluster restart if we allocate a boat load of shards
    //获取shard所在索引元数据
    final IndexMetaData indexMetaData = allocation.metaData().getIndexSafe(unassignedShard.index());
    //根据shard id从集群状态获取同步ID列表
    final Set<String> inSyncAllocationIds = indexMetaData.inSyncAllocationIds(unassignedShard.id());
    final boolean snapshotRestore = unassignedShard.recoverySource().getType() == RecoverySource.Type.SNAPSHOT;

    assert inSyncAllocationIds.isEmpty() == false;
    // use in-sync allocation ids to select nodes
    //构建匹配同步ID的目标节点列表
    final NodeShardsResult nodeShardsResult = buildNodeShardsResult(unassignedShard, snapshotRestore,
        allocation.getIgnoreNodes(unassignedShard.shardId()), inSyncAllocationIds, shardState, logger);
    final boolean enoughAllocationsFound = nodeShardsResult.orderedAllocationCandidates.size() > 0;
    logger.debug("[{}][{}]: found {} allocation candidates of {} based on allocation ids: [{}]", unassignedShard.index(),
        unassignedShard.id(), nodeShardsResult.orderedAllocationCandidates.size(), unassignedShard, inSyncAllocationIds);

    //对上一步得到的列表中的节点依次执行decider,决策结果
    NodesToAllocate nodesToAllocate = buildNodesToAllocate(
        allocation, nodeShardsResult.orderedAllocationCandidates, unassignedShard, false
    );
```
## 发起fetchData请求
(1) 发起请求

从所有数据节点异步获取某个特定分片的信息，这里显然是一个RPC请求。
```
/**
 * Async fetches data for the provided shard with the set of nodes that need to be fetched from.
 */
// visible for testing
void asyncFetch(final DiscoveryNode[] nodes, long fetchingRound) {
    logger.trace("{} fetching [{}] from {}", shardId, type, nodes);
    //nodes为节点列表
    action.list(shardId, nodes, new ActionListener<BaseNodesResponse<T>>() {
        @Override
        public void onResponse(BaseNodesResponse<T> response) {
            //收到成功的回复
            processAsyncFetch(response.getNodes(), response.failures(), fetchingRound);
        }

        @Override
        public void onFailure(Exception e) {
            //收到失败的回复
            List<FailedNodeException> failures = new ArrayList<>(nodes.length);
            for (final DiscoveryNode node: nodes) {
                failures.add(new FailedNodeException(node.getId(), "total failure in fetching", e));
            }
            processAsyncFetch(null, failures, fetchingRound);
        }
    });
}
```
action.list最终调用TransportNodesAction.AsyncAction.start()方法发起异步请求。

RpcAction的注册信息如下，请求对应的是NodeRequest，其他Node处理的逻辑是NodeTransportHandler。
```
transportService.registerRequestHandler(
            transportNodeAction, nodeExecutor, nodeRequest, new NodeTransportHandler());
```
其他节点响应

InternalPrimaryShardAllocator主分片发起RPC请求，其他Node处理对应逻辑是TransportNodesListGatewayStartedShards的nodeOperation方法，请求流程如下：
```
class NodeTransportHandler implements TransportRequestHandler<NodeRequest> {

    @Override
    public void messageReceived(NodeRequest request, TransportChannel channel, Task task) throws Exception {
        channel.sendResponse(nodeOperation(request, task));
    }
}
--------------------------------------
protected NodeGatewayStartedShards nodeOperation(NodeRequest request) {
   ..........
}
```

发送节点处理其他节点响应结果

发送节点处理其他节点响应结果在AsyncShardFetch类的processAsyncFetch方法。
```
protected synchronized void processAsyncFetch(List<T> responses, List<FailedNodeException> failures, long fetchingRound) {
     //处理响应结果的核心方法
     reroute(shardId, "post_response");
}
--------------------------------------------
protected void reroute(ShardId shardId, String reason) {
    logger.trace("{} scheduling reroute for {}", shardId, reason);
    assert rerouteService != null;
    rerouteService.reroute("async_shard_fetch", Priority.HIGH, ActionListener.wrap(
        r -> logger.trace("{} scheduled reroute completed for {}", shardId, reason),
        e -> logger.debug(new ParameterizedMessage("{} scheduled reroute failed for {}", shardId, reason), e)));
}
```
最终向ClusterService提交一个cluster_reroute请求。

## 主分片选举实现
ES的主分片选举是从inSyncAllocationIds列表中选择一个作为主分片。

这个思想在Kafka中也有体现，Kafka的partation对应的逻辑关系AR= ISR + OSR；
ES的主分片选举和Kafka的preferred leader选举，也具有相同的思想；
5.x 以下的版本：通过对比节点分片元信息的版本号。但这种方式不一定能选到最新的数据，比如高版本的接点还未启动的时候
5.x 及以上的版本：给每个分片都设置一个UUID，在集群元信息里面标识那个是最新的

## ReplicaShardAllocator
ReplicaShard的决策结果流程如下：
```
public AllocateUnassignedDecision makeAllocationDecision(final ShardRouting unassignedShard,
                                                         final RoutingAllocation allocation,
                                                         final Logger logger) {

    final RoutingNodes routingNodes = allocation.routingNodes();
    final boolean explain = allocation.debugDecision();
    // pre-check if it can be allocated to any node that currently exists, so we won't list the store for it for nothing
    Tuple<Decision, Map<String, NodeAllocationResult>> result = canBeAllocatedToAtLeastOneNode(unassignedShard, allocation);
    Decision allocateDecision = result.v1();
    
    //拉取数据
    AsyncShardFetch.FetchResult<NodeStoreFilesMetaData> shardStores = fetchData(unassignedShard, allocation);

    //查找对应主分片
    ShardRouting primaryShard = routingNodes.activePrimary(unassignedShard.shardId());
   
    assert primaryShard.currentNodeId() != null;
    final DiscoveryNode primaryNode = allocation.nodes().get(primaryShard.currentNodeId());
    //查找已经存储缓存（主分片已经Fetch）的shard元数据
    final TransportNodesListShardStoreMetaData.StoreFilesMetaData primaryStore = findStore(primaryNode, shardStores);

  ..............................................
```
副本分片fetchData基本逻辑可以参考主分片fetchData

## initialize
如果决策结果为YES，shard将进行初始化，把shard从unassigned 状态转换到 initialize 状态。
```
public ShardRouting initializeShard(ShardRouting unassignedShard, String nodeId, @Nullable String existingAllocationId,
                                    long expectedSize, RoutingChangesObserver routingChangesObserver) {
    ensureMutable();
    assert unassignedShard.unassigned() : "expected an unassigned shard " + unassignedShard;
    //初始化一个ShardRouting
    ShardRouting initializedShard = unassignedShard.initialize(nodeId, existingAllocationId, expectedSize);
    //添加到目的节点的shard列表
    node(nodeId).add(initializedShard);
    inactiveShardCount++;
    if (initializedShard.primary()) {
        inactivePrimaryCount++;
    }
    addRecovery(initializedShard);
    //添加到assignedshards列表
    assignedShardsAdd(initializedShard);
    //设置状态已更新(routingChangesObserver:对应DelegatingRoutingChangesObserver)
    routingChangesObserver.shardInitialized(unassignedShard, initializedShard);
    return initializedShard;
}
```
unassignedShard.initialize是初始化一个未分配的shard，逻辑比较简单，就是生成一个allocationId 并初始化ShardRouting。
```
public ShardRouting initialize(String nodeId, @Nullable String existingAllocationId, long expectedShardSize) {
    final AllocationId allocationId;
    if (existingAllocationId == null) {
        allocationId = AllocationId.newInitializing();
    } else {
        allocationId = AllocationId.newInitializing(existingAllocationId);
    }
    return new ShardRouting(shardId, nodeId, null, primary, ShardRoutingState.INITIALIZING, recoverySource,
        unassignedInfo, allocationId, expectedShardSize);
}
```
routingChangesObserver.shardInitialized初始化调用RestoreInProgressUpdater、RoutingNodesChangedObserver、IndexMetaDataUpdater的初始化方法。RestoreInProgressUpdater记录shard restore进度； RoutingNodesChangedObserver设置shard的recoverSource；IndexMetaDataUpdater是针对PrimaryShard设置状态。
```
public void shardInitialized(ShardRouting unassignedShard, ShardRouting initializedShard) {
    for (RoutingChangesObserver routingChangesObserver : routingChangesObservers) {
        //nodesChangedObserver, indexMetaDataUpdater, restoreInProgressUpdater
        routingChangesObserver.shardInitialized(unassignedShard, initializedShard);
    }
}
```

## 小结
（1）主备之间的异常检测

租约机制： 主副本定期向备副本获取租约
可能出现的异常及解决方式：

如果主节点在一定时间（lease period）内未收到来自某个副本节点的租约回复，则告诉配置管理器（Master），将其移除，同时自己也将降级，不在作为主节点；
如果副本节点在一定时间（grace period）内没有收到来自主节点的租约请求，则告诉配置管理器，将主节点移除，然后将自己提升为主节点，多个从节点竞争提升则哪个先执行成功，哪个从节点就提升为主节点
理论上讲，在没有时钟漂移的情况下,只要grace period >= lease period 就会保证，主节点先感知到租约失效，因此保证了新主节点生成时，旧主节点已经失效，不会出现脑裂现象。
（2）分配读写

primary shard 主分片，承担 读写请求负载；

replica shard 副本分片，是主分片的副本，负责容错，以及承担读请求负载；