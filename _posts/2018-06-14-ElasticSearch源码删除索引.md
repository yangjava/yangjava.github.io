---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码删除索引

## TransportDeleteIndexAction
Master节点的操作大多需要继承TransportMasterNodeAction类，具体的执行流程看实现类的masterOperation方法即可。
```
protected void masterOperation(final DeleteIndexRequest request, final ClusterState state,
                               final ActionListener<AcknowledgedResponse> listener) {
    //获取要删除的索引
    final Set<Index> concreteIndices = new HashSet<>(Arrays.asList(indexNameExpressionResolver.concreteIndices(state, request)));
    if (concreteIndices.isEmpty()) {
        listener.onResponse(new AcknowledgedResponse(true));
        return;
    }

    //构建删除索引的请求
    DeleteIndexClusterStateUpdateRequest deleteRequest = new DeleteIndexClusterStateUpdateRequest()
        .ackTimeout(request.timeout()).masterNodeTimeout(request.masterNodeTimeout())
        .indices(concreteIndices.toArray(new Index[concreteIndices.size()]));

    //调用MetaDataDeleteIndexService服务执行删除索引
    deleteIndexService.deleteIndices(deleteRequest, new ActionListener<ClusterStateUpdateResponse>() {

        @Override
        public void onResponse(ClusterStateUpdateResponse response) {
            listener.onResponse(new AcknowledgedResponse(response.isAcknowledged()));
        }

        @Override
        public void onFailure(Exception t) {
            logger.debug(() -> new ParameterizedMessage("failed to delete indices [{}]", concreteIndices), t);
            listener.onFailure(t);
        }
    });
}
```
这里和创建索引的流程类似，依旧是调用MetaDataDeleteIndexService服务执行索引删除，外部留下监听器根据删除的结果响应不同的请求。

## MetaDataDeleteIndexService
这里是真正执行索引删除的服务，与MetaDataCreateIndexService功能是一样的，核心作用：

1.提交删除索引的任务；

2.定义具体删除索引任务的执行流程；

## 任务提交
与创建索引类似，这里向clusterService提交一个删除索引的任务，具体的任务是实现AckedClusterStateUpdateTask的内部类，之前的流程已经分析过，最终会调用execute方法。
```
public void deleteIndices(final DeleteIndexClusterStateUpdateRequest request,
        final ActionListener<ClusterStateUpdateResponse> listener) {
    ......................................
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
}
```

## 集群元数据
删除索引的数据封装在DeleteIndexClusterStateUpdateRequest类中，这里并没有真正的从磁盘删除索引，只是删除Master节点索引的元数据。

删除的基本流程是，从Request获取对应的索引名称，并加入到ClusterState的已删除索引集合，更新元数据，最后执行路由。
```
/**
 * Delete some indices from the cluster state.
 */
public ClusterState deleteIndices(ClusterState currentState, Set<Index> indices) {
    //获取待删除的索引
    final MetaData meta = currentState.metaData();
    final Set<Index> indicesToDelete = indices.stream().map(i -> meta.getIndexSafe(i).getIndex()).collect(toSet());

    // Check if index deletion conflicts with any running snapshots
    Set<Index> snapshottingIndices = SnapshotsService.snapshottingIndices(currentState, indicesToDelete);
    if (snapshottingIndices.isEmpty() == false) {
        throw new SnapshotInProgressException("Cannot delete indices that are being snapshotted: " + snapshottingIndices +
            ". Try again after snapshot finishes or cancel the currently running snapshot.");
    }

    RoutingTable.Builder routingTableBuilder = RoutingTable.builder(currentState.routingTable());
    MetaData.Builder metaDataBuilder = MetaData.builder(meta);
    ClusterBlocks.Builder clusterBlocksBuilder = ClusterBlocks.builder().blocks(currentState.blocks());

    //已经删除索引的集合（索引删除是异步进行）
    final IndexGraveyard.Builder graveyardBuilder = IndexGraveyard.builder(metaDataBuilder.indexGraveyard());
    final int previousGraveyardSize = graveyardBuilder.tombstones().size();
    //删除索引主要是修改clusterstate元数据，删数其中的index。
    for (final Index index : indices) {
        String indexName = index.getName();
        logger.info("{} deleting index", index);
        routingTableBuilder.remove(indexName);
        clusterBlocksBuilder.removeIndexBlocks(indexName);
        metaDataBuilder.remove(indexName);
    }
    // add tombstones to the cluster state for each deleted index
    // 待删除索引加入已删除索引集合
    final IndexGraveyard currentGraveyard = graveyardBuilder.addTombstones(indices).build(settings);
    metaDataBuilder.indexGraveyard(currentGraveyard); // the new graveyard set on the metadata
    logger.trace("{} tombstones purged from the cluster state. Previous tombstone size: {}. Current tombstone size: {}.",
        graveyardBuilder.getNumPurged(), previousGraveyardSize, currentGraveyard.getTombstones().size());

    MetaData newMetaData = metaDataBuilder.build();
    ClusterBlocks blocks = clusterBlocksBuilder.build();

    // update snapshot restore entries
    // 正在resotre的Index
    ImmutableOpenMap<String, ClusterState.Custom> customs = currentState.getCustoms();
    final RestoreInProgress restoreInProgress = currentState.custom(RestoreInProgress.TYPE);
    if (restoreInProgress != null) {
        RestoreInProgress updatedRestoreInProgress = RestoreService.updateRestoreStateWithDeletedIndices(restoreInProgress, indices);
        if (updatedRestoreInProgress != restoreInProgress) {
            ImmutableOpenMap.Builder<String, ClusterState.Custom> builder = ImmutableOpenMap.builder(customs);
            builder.put(RestoreInProgress.TYPE, updatedRestoreInProgress);
            customs = builder.build();
        }
    }

    //路由广播,对shard进行重新分配。
    return allocationService.reroute(
            ClusterState.builder(currentState)
                .routingTable(routingTableBuilder.build())
                .metaData(newMetaData)
                .blocks(blocks)
                .customs(customs)
                .build(),
            "deleted indices [" + indices + "]");
}
```
到这里删除所有也只是完成了Master节点元数据的更新，Data节点对应的索引数据并没有删除，具体的删除流程索引管理--集群状态发布。