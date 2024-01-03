---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码索引管理--集群状态应用

创建索引是Master节点响应请求，Master节点响应Client请求并把任务提交到ClusterService，最后由MasterService类执行具体任务。在MasterService类的 runTasks方法内，calculateTaskOutputs方法是个分界点：
```
final TaskOutputs taskOutputs = calculateTaskOutputs(taskInputs, previousClusterState);
```

## Master节点新的集群状态
Master节点的操作最终是计算出新的集群状态ClusterState。calculateTaskOutputs方法，调用创建/删除索引的服务，计算出新的集群状态元数据。
```
private TaskOutputs calculateTaskOutputs(TaskInputs taskInputs, ClusterState previousClusterState) {
    // 抽象类ClusterStateUpdateTask实现了ClusterStateTaskExecutor接口
    ClusterTasksResult<Object> clusterTasksResult = executeTasks(taskInputs, previousClusterState);
    //根据previousClusterState及任务执行结果clusterTasksResult，计算出newClusterState
    ClusterState newClusterState = patchVersions(previousClusterState, clusterTasksResult);
    return new TaskOutputs(taskInputs, previousClusterState, newClusterState, getNonFailedTasks(taskInputs, clusterTasksResult),
        clusterTasksResult.executionResults);
}
--------------------------------------------------------
private ClusterTasksResult<Object> executeTasks(TaskInputs taskInputs, ClusterState previousClusterState) {
    ClusterTasksResult<Object> clusterTasksResult;
    try {
        List<Object> inputs = taskInputs.updateTasks.stream().map(tUpdateTask -> tUpdateTask.task).collect(Collectors.toList());
        // 抽象类ClusterStateUpdateTask实现了ClusterStateTaskExecutor接口
        clusterTasksResult = taskInputs.executor.execute(previousClusterState, inputs);
        if (previousClusterState != clusterTasksResult.resultingState &&
            previousClusterState.nodes().isLocalNodeElectedMaster() &&
            (clusterTasksResult.resultingState.nodes().isLocalNodeElectedMaster() == false)) {
            throw new AssertionError("update task submitted to MasterService cannot remove master");
        }
    }
```

## 创建索引
executeTasks方法最终调用 IndexCreationTask类的execute方法。IndexCreationTask有详细描述，即完成Master节点的操作，构建出新的ClusterState。
```
clusterService.submitStateUpdateTask(
        "create-index [" + request.index() + "], cause [" + request.cause() + "]",
    //提交元数据变更任务，索引创建优先级为 URGENT
    new IndexCreationTask(
                logger,
                allocationService,
                request,
                listener,
                indicesService,
                aliasValidator,
                xContentRegistry,
                settings,
                //索引名称/索引Settings校验
                this::validate,
                indexScopedSettings));
```

## 删除索引
executeTasks方法最终调用内部类AckedClusterStateUpdateTask实现的execute方法。
```
clusterService.submitStateUpdateTask("delete-index " + Arrays.toString(request.indices()),
    new AckedClusterStateUpdateTask<ClusterStateUpdateResponse>(Priority.URGENT, request, listener) {

    @Override
    protected ClusterStateUpdateResponse newResponse(boolean acknowledged) {
        return new ClusterStateUpdateResponse(acknowledged);
    }

    @Override
    public ClusterState execute(final ClusterState currentState) {
        return deleteIndices(currentState, Sets.newHashSet(request.indices()));
    }
});
```

## 其他节点应用集群状态
根据calculateTaskOutputs方法的结果，Master节点执行RPC请求，publis最新的集群状态ClusterState，这个过程，远程节点处理Commit请求的实现方法：
```
private void handleApplyCommit(ApplyCommitRequest applyCommitRequest, ActionListener<Void> applyListener) {
    synchronized (mutex) {
        logger.trace("handleApplyCommit: applying commit {}", applyCommitRequest);
        // 将我们之前接收到的 acceptedState 的元数据改为 commit 状态
        coordinationState.get().handleCommit(applyCommitRequest);
        final ClusterState committedState = hideStateIfNotRecovered(coordinationState.get().getLastAcceptedState());
        applierState = mode == Mode.CANDIDATE ? clusterStateWithNoMasterBlock(committedState) : committedState;
        if (applyCommitRequest.getSourceNode().equals(getLocalNode())) {
            // master node applies the committed state at the end of the publication process, not here.
            // Master节点最后commit
            applyListener.onResponse(null);
        } else {
            // 元数据接收完毕，进入应用环节,数据节点 commit 元数据之后，元数据的发布流程就完毕了，之后数据节点进入异步应用环节
            clusterApplier.onNewClusterState(applyCommitRequest.toString(), () -> applierState,
                new ClusterApplyListener() {

                    @Override
                    public void onFailure(String source, Exception e) {
                        applyListener.onFailure(e);
                    }

                    @Override
                    public void onSuccess(String source) {
                        //成功返回
                        applyListener.onResponse(null);
                    }
                });
        }
    }
}
```
clusterApplier.onNewClusterState是异步应用集群状态的入口。其中apply方法applyChanges应用集群状态，该方法中正式调用各个模块的Applier和Listener，并更新本节点存储的集群状态.
```
//在正常情况下，进入applyChanges应用集群状态，该方法中正式调用各个模块的Applier和Listener，并更新本节点存储的集群状态。
//元数据应用流程核心逻辑在 ClusterApplierService.java 的 applyChanges 函数中，
// 主要是处理一些列元数据变更附属的任务，例如创建、删除索引、template 维护等，另外 master 节点上还会回调一些元数据变更完成后关联的 listener
private void applyChanges(UpdateTask task, ClusterState previousClusterState, ClusterState newClusterState, StopWatch stopWatch) {
    ClusterChangedEvent clusterChangedEvent = new ClusterChangedEvent(task.source, newClusterState, previousClusterState);
    // new cluster state, notify all listeners
    final DiscoveryNodes.Delta nodesDelta = clusterChangedEvent.nodesDelta();
    ..........................
    try (Releasable ignored = stopWatch.timing("connecting to new nodes")) {
        connectToNodesAndWait(newClusterState);
    }


    //通知所有Appliers
    logger.debug("apply cluster state with version {}", newClusterState.version());
    // 进行元数据应用，会按优先级分为 low/normal/high 来应用，例如创建索引属于 high
    callClusterStateAppliers(clusterChangedEvent, stopWatch);

    nodeConnectionsService.disconnectFromNodesExcept(newClusterState.nodes());


    //更新本 类存储的集群状态信息
    logger.debug("set locally applied cluster state to version {}", newClusterState.version());
    // 更新本地最新 commited 的 clusterState
    state.set(newClusterState);

    //通知所有Listeners
    // 这里一般是 master 节点发起元数据 commit 结束后，再回调相应的 listener
    callClusterStateListeners(clusterChangedEvent, stopWatch);
}
```

## 异步应用集群状态 callClusterStateAppliers
在onNewClusterState的具体实现内，通过submitStateUpdateTask方法使用线程池threadPoolExecutor执行UpdateTask任务，这一过程是异步进行。

在UpdateTask的run方法内，调用applyChanges方法应用集群状态。

callClusterStateAppliers进行元数据应用，会按优先级分为 low/normal/high 来应用，例如创建索引属于 high。
```
//遍历Applier,依次调用各模块的处理函数
private void callClusterStateAppliers(ClusterChangedEvent clusterChangedEvent, StopWatch stopWatch) {
    //遍历全部的Applier,依次调用各模块对集群状态的处理
    clusterStateAppliers.forEach(applier -> {
        logger.trace("calling [{}] with change to version [{}]", applier, clusterChangedEvent.state().version());
        try (Releasable ignored = stopWatch.timing("running applier [" + applier + "]")) {
            //调用各模块实现的applyClusterState
            applier.applyClusterState(clusterChangedEvent);
        }
    });
}
```
以创建/删除索引为例，最重要的applyClusterState方法是IndicesClusterStateService的实现。
```
//以创建索引来介绍元数据变更主流程，上述 callClusterStateAppliers apply 元数据的环节会进到
// IndicesClusterStateService.java applyClusterState 函数，该函数有一连串应用最新 state 的流程
@Override
public synchronized void applyClusterState(final ClusterChangedEvent event) {
    //总体来看，master 经过两阶段提交元数据后，进入元数据应用流程，
    // 各个节点对比自己本地的信息和接收的元数据，根据差异处理相关流程。
    if (!lifecycle.started()) {
        return;
    }

    // 最新 commit 的元数据
    final ClusterState state = event.state();

    // we need to clean the shards and indices we have on this node, since we
    // are going to recover them again once state persistence is disabled (no master / not recovered)
    // TODO: feels hacky, a block disables state persistence, and then we clean the allocated shards, maybe another flag in blocks?
    if (state.blocks().disableStatePersistence()) {
        for (AllocatedIndex<? extends Shard> indexService : indicesService) {
            indicesService.removeIndex(indexService.index(), NO_LONGER_ASSIGNED,
                "cleaning index (disabled block persistence)"); // also cleans shards
        }
        return;
    }

    updateFailedShardsCache(state);

    // 删除索引、分片，清理磁盘分片目录
    deleteIndices(event); // also deletes shards of deleted indices

    // 删除索引、分片，只是清理内存对象，主要是针对 Close/Open 操作
    removeIndices(event); // also removes shards of removed indices

    failMissingShards(state);

    removeShards(state);   // removes any local shards that doesn't match what the master expects

    // 更新索引 settings、mapping 等
    updateIndices(event); // can also fail shards, but these are then guaranteed to be in failedShardsCache

    // 创建索引
    createIndices(state);

    // 索引存在看看有没需要创建或更新的分片
    createOrUpdateShards(state);
}
```
（1）删除索引

创建索引方法最重要的是更新索引配置信息deleteIndices、removeIndices

deleteIndices
删除索引、分片，清理磁盘分片目录。
```
private void deleteIndices(final ClusterChangedEvent event) {
    final ClusterState previousState = event.previousState();
    final ClusterState state = event.state();
    final String localNodeId = state.nodes().getLocalNodeId();
    assert localNodeId != null;

    //遍历待删除索引,indicesDeleted:计算待删除的索引（IndexGraveyard和集群状态对比两种请求）
    for (Index index : event.indicesDeleted()) {
        if (logger.isDebugEnabled()) {
            logger.debug("[{}] cleaning index, no longer part of the metadata", index);
        }
        AllocatedIndex<? extends Shard> indexService = indicesService.indexService(index);
        final IndexSettings indexSettings;
        if (indexService != null) {
            //直接删除索引
            indexSettings = indexService.getIndexSettings();
            indicesService.removeIndex(index, DELETED, "index no longer part of the metadata");
        } else if (previousState.metaData().hasIndex(index.getName())) {
            // The deleted index was part of the previous cluster state, but not loaded on the local node
            // previousState包含待删除的索引，肯定是要删除的
            final IndexMetaData metaData = previousState.metaData().index(index);
            indexSettings = new IndexSettings(metaData, settings);
            indicesService.deleteUnassignedIndex("deleted index was not assigned to local node", metaData, state);
        } else {
            assert previousState.blocks().hasGlobalBlock(GatewayService.STATE_NOT_RECOVERED_BLOCK);
            //验证索引是否被删除
            final IndexMetaData metaData = indicesService.verifyIndexIsDeleted(index, event.state());
            if (metaData != null) {
                indexSettings = new IndexSettings(metaData, settings);
            } else {
                indexSettings = null;
            }
        }
        if (indexSettings != null) {
            threadPool.generic().execute(new AbstractRunnable() {
                @Override
                public void onFailure(Exception e) {
                    logger.warn(() -> new ParameterizedMessage("[{}] failed to complete pending deletion for index", index), e);
                }

                @Override
                protected void doRun() throws Exception {
                    try {
                        indicesService.processPendingDeletes(index, indexSettings, new TimeValue(30, TimeUnit.MINUTES));
                    } catch (ShardLockObtainFailedException exc) {
}
```
removeIndices
removeIndices处理得是集群元数据，只是清理内存对象。
```
private void removeIndices(final ClusterChangedEvent event) {
    final ClusterState state = event.state();
    final String localNodeId = state.nodes().getLocalNodeId();
    assert localNodeId != null;

    final Set<Index> indicesWithShards = new HashSet<>();
    RoutingNode localRoutingNode = state.getRoutingNodes().node(localNodeId);
    if (localRoutingNode != null) { // null e.g. if we are not a data node
        for (ShardRouting shardRouting : localRoutingNode) {
            indicesWithShards.add(shardRouting.index());
        }
    }

    for (AllocatedIndex<? extends Shard> indexService : indicesService) {
        final Index index = indexService.index();
        final IndexMetaData indexMetaData = state.metaData().index(index);
        final IndexMetaData existingMetaData = indexService.getIndexSettings().getIndexMetaData();

        AllocatedIndices.IndexRemovalReason reason = null;
        if (indexMetaData != null && indexMetaData.getState() != existingMetaData.getState()) {
            reason = indexMetaData.getState() == IndexMetaData.State.CLOSE ? CLOSED : REOPENED;
        } else if (indicesWithShards.contains(index) == false) {
            reason = indexMetaData != null && indexMetaData.getState() == IndexMetaData.State.CLOSE ? CLOSED : NO_LONGER_ASSIGNED;
        }

        if (reason != null) {
            logger.debug("{} removing index ({})", index, reason);
            indicesService.removeIndex(index, reason, "removing index (" + reason + ")");
        }
    }
}
```
（2）创建索引

创建索引方法最重要的是更新索引配置信息updateIndices、createIndices创建索引及createOrUpdateShards创建分片。

updateIndices 创建索引
```
private void updateIndices(ClusterChangedEvent event) {
    //索引配置信息未发生变更
    if (!event.metaDataChanged()) {
        return;
    }
    final ClusterState state = event.state();
    //遍历全部索引
    for (AllocatedIndex<? extends Shard> indexService : indicesService) {
        final Index index = indexService.index();
        //当前索引配置信息
        final IndexMetaData currentIndexMetaData = indexService.getIndexSettings().getIndexMetaData();
        //新的索引配置信息
        final IndexMetaData newIndexMetaData = state.metaData().index(index);
        assert newIndexMetaData != null : "index " + index + " should have been removed by deleteIndices";
        if (ClusterChangedEvent.indexMetaDataChanged(currentIndexMetaData, newIndexMetaData)) {
            String reason = null;
            try {
                reason = "metadata update failed";
                try {
                    //更新当前节点的集群元数据
                    indexService.updateMetaData(currentIndexMetaData, newIndexMetaData);
                } catch (Exception e) {
                    assert false : e;
                    throw e;
                }

                reason = "mapping update failed";
                //是否需要更新Mappings && indices.cluster.send_refresh_mapping（默认值：true）
                if (indexService.updateMapping(currentIndexMetaData, newIndexMetaData) && sendRefreshMapping) {
                    //向Master节点发送更新请求
                    nodeMappingRefreshAction.nodeMappingRefresh(state.nodes().getMasterNode(),
                        new NodeMappingRefreshAction.NodeMappingRefreshRequest(newIndexMetaData.getIndex().getName(),
                            newIndexMetaData.getIndexUUID(), state.nodes().getLocalNodeId())
                    );
                }
            }
```
为什么nodeMappingRefreshAction.nodeMappingRefresh方法向Master节点发送NodeMappingRefreshAction对应的请求？

新的currentIndexMetaData 和 currentIndexMetaData数据不一致（可能是因为集群元数据的更新不是事务），再通过Master节点把新的索引元数据publish给集群内其他节点。

createIndices创建索引
```
//根据接收到的元数据，对比内存最新的索引列表，找出需要创建的索引列表进行创建
private void createIndices(final ClusterState state) {
    // we only create indices for shards that are allocated
    // 通过节点到分片的映射，获取属于该节点的索引分片信息
    RoutingNode localRoutingNode = state.getRoutingNodes().node(state.nodes().getLocalNodeId());
    if (localRoutingNode == null) {
        return;
    }
    // create map of indices to create with shards to fail if index creation fails
    // 对比接收到的元数据和本节点内存的索引列表，找出新增的需要创建的索引
    final Map<Index, List<ShardRouting>> indicesToCreate = new HashMap<>();
    for (ShardRouting shardRouting : localRoutingNode) {
        if (failedShardsCache.containsKey(shardRouting.shardId()) == false) {
            final Index index = shardRouting.index();
            if (indicesService.indexService(index) == null) {
                // 不存在的索引就是需要创建的
                indicesToCreate.computeIfAbsent(index, k -> new ArrayList<>()).add(shardRouting);
            }
        }
    }

    // 过滤出来的待创建索引列表，遍历依次创建
    for (Map.Entry<Index, List<ShardRouting>> entry : indicesToCreate.entrySet()) {
        final Index index = entry.getKey();
        final IndexMetaData indexMetaData = state.metaData().index(index);
        logger.debug("[{}] creating index", index);

        AllocatedIndex<? extends Shard> indexService = null;
        try {
            // 创建索引
            indexService = indicesService.createIndex(indexMetaData, buildInIndexListener);
            //是否需要更新Mappings && indices.cluster.send_refresh_mapping（默认值：true）
            if (indexService.updateMapping(null, indexMetaData) && sendRefreshMapping) {
                nodeMappingRefreshAction.nodeMappingRefresh(state.nodes().getMasterNode(),
                    new NodeMappingRefreshAction.NodeMappingRefreshRequest(indexMetaData.getIndex().getName(),
                        indexMetaData.getIndexUUID(), state.nodes().getLocalNodeId())
                );
            }
        }
```
createOrUpdateShards 索引存在看看有没需要创建或更新的分片
```
private void createOrUpdateShards(final ClusterState state) {
    RoutingNode localRoutingNode = state.getRoutingNodes().node(state.nodes().getLocalNodeId());
    if (localRoutingNode == null) {
        return;
    }

    DiscoveryNodes nodes = state.nodes();
    RoutingTable routingTable = state.routingTable();

    // 创建索引对应的本地分片
    for (final ShardRouting shardRouting : localRoutingNode) {
        ShardId shardId = shardRouting.shardId();
        if (failedShardsCache.containsKey(shardId) == false) {
            AllocatedIndex<? extends Shard> indexService = indicesService.indexService(shardId.getIndex());
            assert indexService != null : "index " + shardId.getIndex() + " should have been created by createIndices";
            Shard shard = indexService.getShardOrNull(shardId.id());
            if (shard == null) {
                assert shardRouting.initializing() : shardRouting + " should have been removed by failMissingShards";
                //创建Shard
                createShard(nodes, routingTable, shardRouting, state);
            } else {
                //更新Shard
                updateShard(nodes, shardRouting, shard, routingTable, state);
            }
}
```
## 调用监听器
当执行完各模块的applyClusterState后，遍历通知所有的Listener：
```
private void callClusterStateListeners(ClusterChangedEvent clusterChangedEvent, StopWatch stopWatch) {
    //遍历通知全部Listene
    Stream.concat(clusterStateListeners.stream(), timeoutClusterStateListeners.stream()).forEach(listener -> {
        try {
            logger.trace("calling [{}] with change to version [{}]", listener, clusterChangedEvent.state().version());
            try (Releasable ignored = stopWatch.timing("notifying listener [" + listener + "]")) {
                //调用各模块实现的clusterChanged
                listener.clusterChanged(clusterChangedEvent);
            }
        } catch (Exception ex) {
            //某个模块执行出现异常时打印日志，但通知过程不会终止
            logger.warn("failed to notify ClusterStateListener", ex);
        }
    });
}
```
当其他服务监听到集群状态发生变化，触发对应监听器执行不同的业务逻辑。这里的监听器还包括超时监听器timeoutClusterStateListeners，不仅仅是业务逻辑相关。

## updateFailedShardsCache
当前节点处理
该方法作用有在源码中的注释已经说的比较明确：

1.若当前节点不再由当前Master节点（Mater节点发生变更情况）分配，则从failedShardsCache移除所有数据；

2.移除不在failedShardsCache中的节点数据；

3.把上次分配失败的shard发送到master节点，并由master节点处理；
```
private void updateFailedShardsCache(final ClusterState state) {
    //state.nodes().getLocalNodeId():当前节点nodeId
    //根据当前nodeId获取当前节点对应的节点信息
    RoutingNode localRoutingNode = state.getRoutingNodes().node(state.nodes().getLocalNodeId());
    if (localRoutingNode == null) {
        //1.Removes shard entries from the failed shards cache that are no longer allocated to this node by the master.
        failedShardsCache.clear();
        return;
    }

    DiscoveryNode masterNode = state.nodes().getMasterNode();

    // remove items from cache which are not in our routing table anymore and resend failures that have not executed on master yet
    for (Iterator<Map.Entry<ShardId, ShardRouting>> iterator = failedShardsCache.entrySet().iterator(); iterator.hasNext(); ) {
        ShardRouting failedShardRouting = iterator.next().getValue();
        //根据shardId获取对应的shard信息
        ShardRouting matchedRouting = localRoutingNode.getByShardId(failedShardRouting.shardId());
        if (matchedRouting == null || matchedRouting.isSameAllocation(failedShardRouting) == false) {
            iterator.remove();
        } else {
            //3.Resends shard failures for shards that are still marked as allocated to this node but previously failed.
            if (masterNode != null) { // TODO: can we remove this? Is resending shard failures the responsibility of shardStateAction?
                String message = "master " + masterNode + " has not removed previously failed shard. resending shard failure";
                logger.trace("[{}] re-sending failed shard [{}], reason [{}]", matchedRouting.shardId(), matchedRouting, message);
                shardStateAction.localShardFailed(matchedRouting, message, null, SHARD_STATE_ACTION_LISTENER, state);
            }
}
```
其中，failedShardsCache存储shard分配失败情况下对应的数据
```
// a list of shards that failed during recovery
// we keep track of these shards in order to prevent repeated recovery of these shards on each cluster state update
final ConcurrentMap<ShardId, ShardRouting> failedShardsCache = ConcurrentCollections.newConcurrentMap();
```
## master节点处理
当前节点调用shardStateAction.localShardFailed方法，通过sendShardAction方法把Action名称是SHARD_FAILED_ACTION_NAME 的信息发送到Master节点。

RequestHandler注册的信息如下：
```
transportService.registerRequestHandler(SHARD_FAILED_ACTION_NAME, ThreadPool.Names.SAME, FailedShardEntry::new,
    new ShardFailedTransportHandler(clusterService,
        new ShardFailedClusterStateTaskExecutor(allocationService, rerouteService, () -> followUpRerouteTaskPriority, logger),
        logger));
```
Master节点收到其他节点发来的请求依旧是通过RequestHandlerRegistry类的processMessageReceived方法完成。
```
public void processMessageReceived(Request request, TransportChannel channel) throws Exception {
    //本地Node任务暂存
    final Task task = taskManager.register(channel.getChannelType(), action, request);
    boolean success = false;
    try {
        handler.messageReceived(request, new TaskTransportChannel(taskManager, task, channel), task);
        success = true;
    }
```
ShardFailedTransportHandler类的messageReceived实现了Action名称是SHARD_FAILED_ACTION_NAME具体的业务逻辑。具体业务逻辑和Master节点从client响应Http请求逻辑基本一致，依旧是向clusterService提交任务（对应名称是shard-failed），更新Master节点对应的ClusterState元数据。
```
public void messageReceived(FailedShardEntry request, TransportChannel channel, Task task) throws Exception {
    logger.debug(() -> new ParameterizedMessage("{} received shard failed for {}",
        request.shardId, request), request.failure);
    clusterService.submitStateUpdateTask(
        "shard-failed",
        request,
        //任务优先级是HIGH
        ClusterStateTaskConfig.build(Priority.HIGH),
        shardFailedClusterStateTaskExecutor,
        new ClusterStateTaskListener() {  
```
## deleteIndices
1.执行DELETE 删除索引的操作，被删除的索引会被写入到集群metaData中的tombstones集合中，且metaData信息是存储在master节点的本地文件中的（global-x.st）；
2.master节点启动时，会从本地路径下读取对应的文件，并将集群信息加载到metaData中；
3.在master节点同步集群状态过程中，会验证处于tombstones中的索引是否被有效删除（本地索引存储目录是否被有效删除）;
4.如果tombstones中的索引文件依然存在，则会在此过程中被删除；
```
private void deleteIndices(final ClusterChangedEvent event) {
    final ClusterState previousState = event.previousState();
    final ClusterState state = event.state();
    final String localNodeId = state.nodes().getLocalNodeId();
    assert localNodeId != null;

    //遍历待删除索引,indicesDeleted:计算待删除的索引（IndexGraveyard和集群状态对比两种请求）
    for (Index index : event.indicesDeleted()) {

        //根据Index获取对应的IndexService(每个索引都包含对应的IndexService)
        AllocatedIndex<? extends Shard> indexService = indicesService.indexService(index);
        final IndexSettings indexSettings;
        if (indexService != null) {
            //直接删除索引
            indexSettings = indexService.getIndexSettings();
            indicesService.removeIndex(index, DELETED, "index no longer part of the metadata");
        } else if (previousState.metaData().hasIndex(index.getName())) {
            // The deleted index was part of the previous cluster state, but not loaded on the local node
            // previousState包含待删除的索引，肯定是要删除的
            final IndexMetaData metaData = previousState.metaData().index(index);
            indexSettings = new IndexSettings(metaData, settings);
            indicesService.deleteUnassignedIndex("deleted index was not assigned to local node", metaData, state);
        } else {
            //验证索引是否被删除
            final IndexMetaData metaData = indicesService.verifyIndexIsDeleted(index, event.state());
            if (metaData != null) {
                indexSettings = new IndexSettings(metaData, settings);
            } else {
                indexSettings = null;
            }
        }
        if (indexSettings != null) {
            threadPool.generic().execute(new AbstractRunnable() {
                @Override
                public void onFailure(Exception e) {
                    logger.warn(() -> new ParameterizedMessage("[{}] failed to complete pending deletion for index", index), e);
                }

                @Override
                protected void doRun() throws Exception {
                    try {
                        indicesService.processPendingDeletes(index, indexSettings, new TimeValue(30, TimeUnit.MINUTES));
            });
        }
    }
```

## indicesDeleted
indicesDeleted返回的是ClusterChangedEvent事件下需要删除的索引。
```
public List<Index> indicesDeleted() {
    if (previousState.blocks().hasGlobalBlock(GatewayService.STATE_NOT_RECOVERED_BLOCK)) {
        // working off of a non-initialized previous state, so use the tombstones for index deletions
        //重启场景下的event获取的deleted状态的索引主要是通过集群metaData中的tombstones拿到的，
        // 这个也很好理解因为ES节点是重启操作，因此不会依赖对比previous与当前集群metaData来获取结果值。
        return indicesDeletedFromTombstones();
    } else {
        // examine the diffs in index metadata between the previous and new cluster states to get the deleted indices
        return indicesDeletedFromClusterState();
    }
}
```
（1）indicesDeletedFromTombstones

indicesDeletedFromTombstones对应的是从Tombstones中获取。

索引墓碑。记录已删除的索引，并保存一段时间。索引删除是主节点通过下发集群状态来执行的各节点处理集群状态是异步的过程。
例如，索引分片分布在5个节点上，删除索引期间，某个节点是“down”掉的，没有执行删除逻辑
当这个节点恢复的时候，其存储的已删除的索引会被当作孤立资源加入集群,索引死而复活。
墓碑的作用就是防止发生这种情况

在ES中存在索引墓碑，同样的思想也出现在Kafka对topic删除的过程中。对应的源码的英文注释也是对这种情况的解释。
```
private List<Index> indicesDeletedFromTombstones() {
    // We look at the full tombstones list to see which indices need to be deleted.  In the case of
    // a valid previous cluster state, indicesDeletedFromClusterState() will be used to get the deleted
    // list, so a diff doesn't make sense here.  When a node (re)joins the cluster, its possible for it
    // to re-process the same deletes or process deletes about indices it never knew about.  This is not
    // an issue because there are safeguards in place in the delete store operation in case the index
    // folder doesn't exist on the file system.
    List<IndexGraveyard.Tombstone> tombstones = state.metaData().indexGraveyard().getTombstones();
    return tombstones.stream().map(IndexGraveyard.Tombstone::getIndex).collect(Collectors.toList());
}
```
（2）indicesDeletedFromClusterState

indicesDeletedFromClusterState返回的是新的ClusterState和之前的ClusterState不同的索引，如果之前的ClusterState有对应的索引，新的ClusterState确没有，这种情况下肯定是需要删除的。
```
private List<Index> indicesDeletedFromClusterState() {
    List<Index> deleted = null;
    for (ObjectCursor<IndexMetaData> cursor : previousState.metaData().indices().values()) {
        IndexMetaData index = cursor.value;
        IndexMetaData current = state.metaData().index(index.getIndex());
        if (current == null) {
            if (deleted == null) {
                deleted = new ArrayList<>();
            }
            deleted.add(index.getIndex());
        }
    }
    return deleted == null ? Collections.<Index>emptyList() : deleted;
}
```

## removeIndex
removeIndex释放资源后，把shards files, state and transaction logs从磁盘删除。
```
public void removeIndex(final Index index, final IndexRemovalReason reason, final String extraInfo) {
    final String indexName = index.getName();
    try {
        final IndexService indexService;
        //索引监听器能够跨越整个索引生命周期
        final IndexEventListener listener;
        //IndexEventListener可能跨多个索引并发操作，这里需要同步
        synchronized (this) {
            //索引不存在
            if (hasIndex(index) == false) {
                return;
            }

            logger.debug("[{}] closing ... (reason [{}])", indexName, reason);
            Map<String, IndexService> newIndices = new HashMap<>(indices);
            indexService = newIndices.remove(index.getUUID());
            assert indexService != null : "IndexService is null for index: " + index;
            indices = unmodifiableMap(newIndices);
            listener = indexService.getIndexEventListener();
        }

        listener.beforeIndexRemoved(indexService, reason);
        logger.debug("{} closing index service (reason [{}][{}])", index, reason, extraInfo);
        //资源释放
        indexService.close(extraInfo, reason == IndexRemovalReason.DELETED);
        logger.debug("{} closed... (reason [{}][{}])", index, reason, extraInfo);
        final IndexSettings indexSettings = indexService.getIndexSettings();
        listener.afterIndexRemoved(indexService.index(), indexSettings, reason);
        //DELETED状态，从磁盘删除
        if (reason == IndexRemovalReason.DELETED) {
            // now we are done - try to wipe data on disk if possible
            //IndexRemovalReason.DELETED:资源释放后，从磁盘把shards files, state and transaction logs删除
            deleteIndexStore(extraInfo, indexService.index(), indexSettings);
        }
    }
```
资源释放，缓存数据刷新到磁盘，从磁盘把shards files, state and transaction logs删除。
```
public void removeIndex(final Index index, final IndexRemovalReason reason, final String extraInfo) {
    final String indexName = index.getName();
    try {
        final IndexService indexService;
        //索引监听器能够跨越整个索引生命周期
        final IndexEventListener listener;
        //IndexEventListener可能跨多个索引并发操作，这里需要同步
        synchronized (this) {
            //索引不存在
            if (hasIndex(index) == false) {
                return;
            }

            logger.debug("[{}] closing ... (reason [{}])", indexName, reason);
            Map<String, IndexService> newIndices = new HashMap<>(indices);
            indexService = newIndices.remove(index.getUUID());
            assert indexService != null : "IndexService is null for index: " + index;
            indices = unmodifiableMap(newIndices);
            listener = indexService.getIndexEventListener();
        }

        listener.beforeIndexRemoved(indexService, reason);
        logger.debug("{} closing index service (reason [{}][{}])", index, reason, extraInfo);
        //资源释放，缓存数据刷新到磁盘
        indexService.close(extraInfo, reason == IndexRemovalReason.DELETED);
        logger.debug("{} closed... (reason [{}][{}])", index, reason, extraInfo);
        final IndexSettings indexSettings = indexService.getIndexSettings();
        listener.afterIndexRemoved(indexService.index(), indexSettings, reason);
        //DELETED状态，从磁盘删除
        if (reason == IndexRemovalReason.DELETED) {
            // now we are done - try to wipe data on disk if possible
            //IndexRemovalReason.DELETED:资源释放后，从磁盘把shards files, state and transaction logs删除
            deleteIndexStore(extraInfo, indexService.index(), indexSettings);
        }
}
```
### deleteUnassignedIndex
删除前node的unassignedShard，从磁盘中删除，内存的数据结构不处理。
```
public void deleteUnassignedIndex(String reason, IndexMetaData metaData, ClusterState clusterState) {
    if (nodeEnv.hasNodeFile()) {
        String indexName = metaData.getIndex().getName();
        try {
            if (clusterState.metaData().hasIndex(indexName)) {
                final IndexMetaData index = clusterState.metaData().index(indexName);
                throw new IllegalStateException("Can't delete unassigned index store for [" + indexName + "] - it's still part of " +
                                                "the cluster state [" + index.getIndexUUID() + "] [" + metaData.getIndexUUID() + "]");
            }
            //从磁盘中删除，但是内存的数据不处理
            deleteIndexStore(reason, metaData, clusterState);
```
verifyIndexIsDeleted
主要是用来验证索引是否被删除，这个删除操作主要是指data节点本地存储路径下的索引目录是否被有效删除
```
public IndexMetaData verifyIndexIsDeleted(final Index index, final ClusterState clusterState) {
    // this method should only be called when we know the index (name + uuid) is not part of the cluster state
    if (clusterState.metaData().index(index) != null) {
        throw new IllegalStateException("Cannot delete index [" + index + "], it is still part of the cluster state.");
    }
    //首先判断当前data节点的nodeEnv对象的nodePath与lock是否为null
    if (nodeEnv.hasNodeFile() && FileSystemUtils.exists(nodeEnv.indexPaths(index))) {
        //若均不为null且本地存在index的完整路径
        //获取当前索引的metaData
        final IndexMetaData metaData;
        try {
            //从本地文件读取
            metaData = metaStateService.loadIndexState(index);
            if (metaData == null) {
                return null;
            }
        } catch (Exception e) {
        }
        //通过metaData获取indexSettings信息
        final IndexSettings indexSettings = buildIndexSettings(metaData);
        try {
            //执行删除
            deleteIndexStoreIfDeletionAllowed("stale deleted index", index, indexSettings, ALWAYS_TRUE);
}
```
## processPendingDeletes
从PendingDelete删除数据，前提是获取shard的锁。
```
public void processPendingDeletes(Index index, IndexSettings indexSettings, TimeValue timeout)
        throws IOException, InterruptedException, ShardLockObtainFailedException {
    logger.debug("{} processing pending deletes", index);
    final long startTimeNS = System.nanoTime();
    //对当前索引的所有shard加锁
    final List<ShardLock> shardLocks = nodeEnv.lockAllForIndex(index, indexSettings, "process pending deletes", timeout.millis());
    int numRemoved = 0;
    try {
        Map<ShardId, ShardLock> locks = new HashMap<>();
        for (ShardLock lock : shardLocks) {
            locks.put(lock.getShardId(), lock);
        }
        final List<PendingDelete> remove;
        synchronized (pendingDeletes) {
             remove = pendingDeletes.remove(index);
        }
        if (remove != null && remove.isEmpty() == false) {
            numRemoved = remove.size();
            CollectionUtil.timSort(remove); // make sure we delete indices first
            final long maxSleepTimeMs = 10 * 1000; // ensure we retry after 10 sec
            long sleepTime = 10;
            do {
                if (remove.isEmpty()) {
                    break;
                }
                Iterator<PendingDelete> iterator = remove.iterator();
                while (iterator.hasNext()) {
                    PendingDelete delete = iterator.next();

                    if (delete.deleteIndex) {
                        assert delete.shardId == -1;
                        logger.debug("{} deleting index store reason [{}]", index, "pending delete");
                        try {
                            //shard获取锁的情况下，从磁盘把数据删除
                            nodeEnv.deleteIndexDirectoryUnderLock(index, indexSettings);
                            iterator.remove();
                        } catch (IOException ex) {
                            logger.debug(() -> new ParameterizedMessage("{} retry pending delete", index), ex);
                        }
                    } else {
                        assert delete.shardId != -1;
                        //获取锁
                        ShardLock shardLock = locks.get(new ShardId(delete.index, delete.shardId));
                        if (shardLock != null) {
                            try {
                                //同样调用deleteShardDirectoryUnderLock方法删除
                                deleteShardStore("pending delete", shardLock, delete.settings);
                                iterator.remove();
                            } catch (IOException ex) {
                       ................................
                }
                if (remove.isEmpty() == false) {
                    logger.warn("{} still pending deletes present for shards {} - retrying", index, remove.toString());
                    Thread.sleep(sleepTime);
                    sleepTime = Math.min(maxSleepTimeMs, sleepTime * 2); // increase the sleep time gradually
                    logger.debug("{} schedule pending delete retry after {} ms", index, sleepTime);
                }
            } while ((System.nanoTime() - startTimeNS) < timeout.nanos());
        }
```
## removeIndices
删除索引、分片，只是清理内存对象. 主要是针对 Close/Open 操作。indicesService.removeIndex方法核心是释放资源，Close/Open操作并不从磁盘中把对应数据删除，详细的代码可以看2.2 removeIndex。
```
private void removeIndices(final ClusterChangedEvent event) {
    final ClusterState state = event.state();
    final String localNodeId = state.nodes().getLocalNodeId();
    assert localNodeId != null;

    final Set<Index> indicesWithShards = new HashSet<>();
    // 节点到分片的映射
    RoutingNode localRoutingNode = state.getRoutingNodes().node(localNodeId);
    if (localRoutingNode != null) { // null e.g. if we are not a data node
        for (ShardRouting shardRouting : localRoutingNode) {
            indicesWithShards.add(shardRouting.index());
        }
    }

    for (AllocatedIndex<? extends Shard> indexService : indicesService) {
        final Index index = indexService.index();
        //从Master节点发来的新IndexMetaData
        final IndexMetaData indexMetaData = state.metaData().index(index);
        //当前节点内存的IndexMetaData
        final IndexMetaData existingMetaData = indexService.getIndexSettings().getIndexMetaData();

        AllocatedIndices.IndexRemovalReason reason = null;
        if (indexMetaData != null && indexMetaData.getState() != existingMetaData.getState()) {
            //新旧集群状态内，当前索引状态元数据不一致
            reason = indexMetaData.getState() == IndexMetaData.State.CLOSE ? CLOSED : REOPENED;
        } else if (indicesWithShards.contains(index) == false) {
            //当前节点不包含对应索引，说明是索引被删除或是新集群
            reason = indexMetaData != null && indexMetaData.getState() == IndexMetaData.State.CLOSE ? CLOSED : NO_LONGER_ASSIGNED;
        }

        if (reason != null) {
            logger.debug("{} removing index ({})", index, reason);
            //释放资源
            indicesService.removeIndex(index, reason, "removing index (" + reason + ")");
        }
    }
}
```
## failMissingShards
Master节点认为当前节点的shard处于Active状态，但是当前节点的shard确并不是Active状态。sendFailShard 包含向Master节点发送消息。
```
private void failMissingShards(final ClusterState state) {
    // 当前节点到分片的映射
    RoutingNode localRoutingNode = state.getRoutingNodes().node(state.nodes().getLocalNodeId());
    if (localRoutingNode == null) {
        return;
    }
    for (final ShardRouting shardRouting : localRoutingNode) {
        ShardId shardId = shardRouting.shardId();
        if (shardRouting.initializing() == false &&
            failedShardsCache.containsKey(shardId) == false &&
            indicesService.getShardOrNull(shardId) == null) {
            // the master thinks we are active, but we don't have this shard at all, mark it as failed
            //通知Master节点，当前shard不存在
            sendFailShard(shardRouting, "master marked shard as active, but shard has not been created, mark shard as failed", null,
                state);
        }
    }
}
```
## removeShards
当前节点对应的shard信息和从Master节点收到的shard信息不一致情况处理。
```
private void removeShards(final ClusterState state) {
    final String localNodeId = state.nodes().getLocalNodeId();
    assert localNodeId != null;

    // remove shards based on routing nodes (no deletion of data)
    // 当前节点到分片的映射
    RoutingNode localRoutingNode = state.getRoutingNodes().node(localNodeId);
    for (AllocatedIndex<? extends Shard> indexService : indicesService) {
        //先遍历索引，再遍历shard
        for (Shard shard : indexService) {
            ShardRouting currentRoutingEntry = shard.routingEntry();
            ShardId shardId = currentRoutingEntry.shardId();
            ShardRouting newShardRouting = localRoutingNode == null ? null : localRoutingNode.getByShardId(shardId);
            if (newShardRouting == null) {
                logger.debug("{} removing shard (not allocated)", shardId);
                //移除未分配的shardId
                indexService.removeShard(shardId.id(), "removing shard (not allocated)");
            } else if (newShardRouting.isSameAllocation(currentRoutingEntry) == false) {
                logger.debug("{} removing shard (stale allocation id, stale {}, new {})", shardId,
                    currentRoutingEntry, newShardRouting);
                //allocation id不一致，移除shard
                indexService.removeShard(shardId.id(), "removing shard (stale copy)");
            } else if (newShardRouting.initializing() && currentRoutingEntry.active()) {
                logger.debug("{} removing shard (not active, current {}, new {})", shardId, currentRoutingEntry, newShardRouting);
                //当前Node 脱离集群的情况
                indexService.removeShard(shardId.id(), "removing shard (stale copy)");
            } else if (newShardRouting.primary() && currentRoutingEntry.primary() == false && newShardRouting.initializing()) {
                logger.debug("{} removing shard (not active, current {}, new {})", shardId, currentRoutingEntry, newShardRouting);
                //集群状态在一个batch内存在Active shard/关闭索引等操作
                indexService.removeShard(shardId.id(), "removing shard (stale copy)");
            }
```
## updateIndices
更新索引 settings、mapping 等信息
```
private void updateIndices(ClusterChangedEvent event) {
    //索引配置信息未发生变更
    if (!event.metaDataChanged()) {
        return;
    }
    final ClusterState state = event.state();
    //遍历全部索引
    for (AllocatedIndex<? extends Shard> indexService : indicesService) {
        final Index index = indexService.index();
        //当前索引配置信息
        final IndexMetaData currentIndexMetaData = indexService.getIndexSettings().getIndexMetaData();
        //新的索引配置信息
        final IndexMetaData newIndexMetaData = state.metaData().index(index);
        assert newIndexMetaData != null : "index " + index + " should have been removed by deleteIndices";
        if (ClusterChangedEvent.indexMetaDataChanged(currentIndexMetaData, newIndexMetaData)) {
            String reason = null;
            try {
                reason = "metadata update failed";
                try {
                    //更新当前节点的集群元数据
                    indexService.updateMetaData(currentIndexMetaData, newIndexMetaData);
                }

                reason = "mapping update failed";
                //是否需要更新Mappings && indices.cluster.send_refresh_mapping（默认值：true）
                if (indexService.updateMapping(currentIndexMetaData, newIndexMetaData) && sendRefreshMapping) {
                    //向Master节点发送更新请求
                    nodeMappingRefreshAction.nodeMappingRefresh(state.nodes().getMasterNode(),
                        new NodeMappingRefreshAction.NodeMappingRefreshRequest(newIndexMetaData.getIndex().getName(),
                            newIndexMetaData.getIndexUUID(), state.nodes().getLocalNodeId())
                    );
```
## createIndices
根据接收到的元数据，对比内存最新的索引列表，找出需要创建的索引列表进行创建
```
private void createIndices(final ClusterState state) {
    // we only create indices for shards that are allocated
    // 通过节点到分片的映射，获取属于该节点的索引分片信息,从这里可以看到每个节点只创建那些分配给自己的索引
    //从这里可以看到每个节点只创建那些分配给自己的索引
    RoutingNode localRoutingNode = state.getRoutingNodes().node(state.nodes().getLocalNodeId());
    if (localRoutingNode == null) {
        return;
    }
    // create map of indices to create with shards to fail if index creation fails
    // 对比接收到的元数据和本节点内存的索引列表，找出新增的需要创建的索引
    final Map<Index, List<ShardRouting>> indicesToCreate = new HashMap<>();
    //遍历分配到该节点的每个shard
    for (ShardRouting shardRouting : localRoutingNode) {
        if (failedShardsCache.containsKey(shardRouting.shardId()) == false) {
            final Index index = shardRouting.index();
            if (indicesService.indexService(index) == null) {
                //如果该节点上还没有创建该索引，则放入indicesToCreate，表示需要创建
                indicesToCreate.computeIfAbsent(index, k -> new ArrayList<>()).add(shardRouting);
            }
        }
    }

    // 过滤出来的待创建索引列表，遍历依次创建
    for (Map.Entry<Index, List<ShardRouting>> entry : indicesToCreate.entrySet()) {
        final Index index = entry.getKey();
        final IndexMetaData indexMetaData = state.metaData().index(index);
        logger.debug("[{}] creating index", index);

        AllocatedIndex<? extends Shard> indexService = null;
        try {
            //创建索引也是调用IndicesService的createIndex函数
            indexService = indicesService.createIndex(indexMetaData, buildInIndexListener);
            //是否需要更新Mappings && indices.cluster.send_refresh_mapping（默认值：true）
            if (indexService.updateMapping(null, indexMetaData) && sendRefreshMapping) {
                nodeMappingRefreshAction.nodeMappingRefresh(state.nodes().getMasterNode(),
                    new NodeMappingRefreshAction.NodeMappingRefreshRequest(indexMetaData.getIndex().getName(),
                        indexMetaData.getIndexUUID(), state.nodes().getLocalNodeId())
                );
            }
        }
```
## createOrUpdateShards
索引存在看看有没需要创建或更新的分片
```
private void createOrUpdateShards(final ClusterState state) {
    //同样只创建分配给自己的shard
    RoutingNode localRoutingNode = state.getRoutingNodes().node(state.nodes().getLocalNodeId());
    if (localRoutingNode == null) {
        return;
    }

    DiscoveryNodes nodes = state.nodes();
    RoutingTable routingTable = state.routingTable();

    // 创建索引对应的本地分片, 遍历每一个shard
    for (final ShardRouting shardRouting : localRoutingNode) {
        ShardId shardId = shardRouting.shardId();
        if (failedShardsCache.containsKey(shardId) == false) {
            AllocatedIndex<? extends Shard> indexService = indicesService.indexService(shardId.getIndex());
            //索引必须在shard之前建好
            assert indexService != null : "index " + shardId.getIndex() + " should have been created by createIndices";
            Shard shard = indexService.getShardOrNull(shardId.id());
            if (shard == null) {
                assert shardRouting.initializing() : shardRouting + " should have been removed by failMissingShards";
                //创建Shard,如果shard还没创建，则调用createShard创建shard
                createShard(nodes, routingTable, shardRouting, state);
            } else {
                //更新Shard
                updateShard(nodes, shardRouting, shard, routingTable, state);
}
```