---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码Index Recovery
官网indices-recovery和cat-recovery给出Index Recovery发生的条件：

Node startup.节点启动（对应类型是local store recovery）
Primary shard replication.主分片复制
同一集群内Shard在不同Node之间的分配，
Snapshot restore operation.
Clone, shrink, or split operation.

## Index Recovery
分片分配成功后就进入数据恢复阶段。为什么需要recovery?

对于主分片来说，可能一些数据没来得及刷盘
对于副分片来说，除了没来及刷盘，也可能是写了主分片，还没来得及写副分片

### Index Recover顺序
Index Recover顺序，优先级如下：

（1）参数index.priority值（越大越早恢复），可以通过如下API设置不同的优先级
```
PUT index_1
{
  "settings": {
    "index.priority": 10
  }
}
```
（2）索引创建时间（时间越晚，越早恢复）

（3）索引名称

### Index Recovery响应
官网Indexs-Recovery，给出执行GET index1,index2/_recovery?human命令后，返回结果如下：

其中比较重要的是有两个属性

（1）type，source shard恢复的枚举类型，对应源码在RecoverySource类，每种恢复类型都有对应的实现类。
```
public enum Type {
    //EmptyStoreRecoverySource
    // 从本地恢复(主分片),新分配的Primary Shard或者执行cluster reroute API命令分配的Primary Shard
    EMPTY_STORE,
    //ExistingStoreRecoverySource
    // 从本地恢复(主分片),从现有存储恢复.Node启动或者是allocation分配已经存在的Primary Shard
    EXISTING_STORE,
    //PeerRecoverySource
    // 从其他Node的主分片恢复当前Node的副本分片. Shard 复制
    PEER,
    //SnapshotRecoverySource
    // 从快照恢复,从快照中恢复 从快照中恢复
    SNAPSHOT,
    //LocalShardsRecoverySource
    // 从本节点的其它分片恢复,从同一节点上另一个索引的其他分片恢复。 clone, shrink, or split shard操作相关.
    LOCAL_SHARDS
}
```
（2）stage，recovery stage对应代码在RecoveryState类
```
public enum Stage {
    //恢复尚未启动
    INIT((byte) 0),

    /**恢复Lucene文件，以及在节点间负责索引数据
     * recovery of lucene files, either reusing local ones are copying new ones
     */
    INDEX((byte) 1),

    /**验证索引
     * potentially running check index
     */
    VERIFY_INDEX((byte) 2),

    /**启动engine，重放translog，建立Lucene索引
     * starting up the engine, replaying the translog
     */
    TRANSLOG((byte) 3),

    /**清理工作
     * performing final task after all translog ops have been done
     */
    FINALIZE((byte) 4),

    //完毕
    DONE((byte) 5);
```
### 重要参数
indices.recovery.max_bytes_per_sec 索引恢复过程中，同一集群内Node节点之间流量，默认40Mb；

indices.recovery.max_concurrent_file_chunks 设置可以并发发送的文件数，默认是2个；

调用IndexService创建一个shard，其仅仅是生成一个IndexShard对象，并确定一些shard的元数据，比如shard的目录等信息。然后把创建好的shard保存在IndexService中shards成员变量中。shard的元数据创建好了，但shard的底层数据并没有创建好，所以会调用IndexShard的startRecovery函数开始为shard准备数据。
```
public IndexShard createShard(
        final ShardRouting shardRouting,
        final RecoveryState recoveryState,
        final PeerRecoveryTargetService recoveryTargetService,
        final PeerRecoveryTargetService.RecoveryListener recoveryListener,
        final RepositoriesService repositoriesService,
        final Consumer<IndexShard.ShardFailure> onShardFailure,
        final Consumer<ShardId> globalCheckpointSyncer,
        final RetentionLeaseSyncer retentionLeaseSyncer) throws IOException {
    Objects.requireNonNull(retentionLeaseSyncer);
    ensureChangesAllowed();
    IndexService indexService = indexService(shardRouting.index());
    //生成IndexShard对象
    IndexShard indexShard = indexService.createShard(shardRouting, globalCheckpointSyncer, retentionLeaseSyncer);
    indexShard.addShardFailureCallback(onShardFailure);
    indexShard.startRecovery(recoveryState, recoveryTargetService, recoveryListener, repositoriesService,
        (type, mapping) -> {
            assert recoveryState.getRecoverySource().getType() == RecoverySource.Type.LOCAL_SHARDS:
                "mapping update consumer only required by local shards recovery";
            client.admin().indices().preparePutMapping()
                .setConcreteIndex(shardRouting.index()) // concrete index - no name clash, it uses uuid
                .setType(type)
                .setSource(mapping.source().string(), XContentType.JSON)
                .get();
        }, this);
    return indexShard;
}
```
方法startRecovery根据不同的类型，执行数据恢复。
```
public void startRecovery(RecoveryState recoveryState, PeerRecoveryTargetService recoveryTargetService,
                          PeerRecoveryTargetService.RecoveryListener recoveryListener, RepositoriesService repositoriesService,
                          BiConsumer<String, MappingMetaData> mappingUpdateConsumer,
                          IndicesService indicesService) {
    assert recoveryState.getRecoverySource().equals(shardRouting.recoverySource());
    switch (recoveryState.getRecoverySource().getType()) {
        case EMPTY_STORE:
        case EXISTING_STORE:
            //主分片从translog恢复，Node启动或者是allocation分配已经存在的Primary Shard
            markAsRecovering("from store", recoveryState); // mark the shard as recovering on the cluster state thread
            threadPool.generic().execute(() -> {
                try {
                    if (recoverFromStore()) {
                        //恢复成功，向Master发送action为 internal:cluster/shard/started的RPC请求
                        recoveryListener.onRecoveryDone(recoveryState);
                    }
                } catch (Exception e) {
                    //恢复失败，关闭Engin，向Master发送 internal:cluster/shard/failure的RPC请求
                    recoveryListener.onRecoveryFailure(recoveryState,
                        new RecoveryFailedException(recoveryState, null, e), true);
                }
            });
            break;
        case PEER:
            //副本分片从主分片远程拉取，Shard 复制
            try {
                markAsRecovering("from " + recoveryState.getSourceNode(), recoveryState);
                recoveryTargetService.startRecovery(this, recoveryState.getSourceNode(), recoveryListener);
            } catch (Exception e) {
                failShard("corrupted preexisting index", e);
                recoveryListener.onRecoveryFailure(recoveryState,
                    new RecoveryFailedException(recoveryState, null, e), true);
            }
            break;
        case SNAPSHOT:
            //从快照中恢复
            markAsRecovering("from snapshot", recoveryState); // mark the shard as recovering on the cluster state thread
            SnapshotRecoverySource recoverySource = (SnapshotRecoverySource) recoveryState.getRecoverySource();
            threadPool.generic().execute(() -> {
                try {
                    final Repository repository = repositoriesService.repository(recoverySource.snapshot().getRepository());
                    if (restoreFromRepository(repository)) {
                        recoveryListener.onRecoveryDone(recoveryState);
                    }
                } catch (Exception e) {
                    recoveryListener.onRecoveryFailure(recoveryState,
                        new RecoveryFailedException(recoveryState, null, e), true);
                }
            });
            break;
        case LOCAL_SHARDS:
            // clone, shrink, or split shard操作相关
            // 从同一个节点的其他分片恢复Shrink使用这种恢复类型
            //省略了其它几种类型的恢复，因为split的主shard的恢复类型为这种
            final IndexMetaData indexMetaData = indexSettings().getIndexMetaData();
            //获取新的索引
            final Index resizeSourceIndex = indexMetaData.getResizeSourceIndex();
            final List<IndexShard> startedShards = new ArrayList<>();
            final IndexService sourceIndexService = indicesService.indexService(resizeSourceIndex);
            final Set<ShardId> requiredShards;
            final int numShards;
            if (sourceIndexService != null) {
                //选择恢复的源shard
                requiredShards = IndexMetaData.selectRecoverFromShards(shardId().id(),
                    sourceIndexService.getMetaData(), indexMetaData.getNumberOfShards());
                for (IndexShard shard : sourceIndexService) {
                    //判断所有的源shard是不是都处于started状态
                    if (shard.state() == IndexShardState.STARTED && requiredShards.contains(shard.shardId())) {
                        startedShards.add(shard);
                    }
                }
                numShards = requiredShards.size();
            } else {
                numShards = -1;
                requiredShards = Collections.emptySet();
            }

            //如果需要的源shard中有shard不属于started状态则报错
            if (numShards == startedShards.size()) {
                assert requiredShards.isEmpty() == false;
                markAsRecovering("from local shards", recoveryState); // mark the shard as recovering on the cluster state thread
                //另起一个线程开始执行数据恢复
                threadPool.generic().execute(() -> {
                    try {
                        //从本节点其他分片恢复(shrink时)
                        //数据恢复会新起一个线程，执行recoverFromLocalShards函数
                        if (recoverFromLocalShards(mappingUpdateConsumer, startedShards.stream()
                            .filter((s) -> requiredShards.contains(s.shardId())).collect(Collectors.toList()))) {
                            recoveryListener.onRecoveryDone(recoveryState);
                        }
                    } catch (Exception e) {
                        recoveryListener.onRecoveryFailure(recoveryState,
                            new RecoveryFailedException(recoveryState, null, e), true);
                    }
                });
            ......................
}
```
无论哪种恢复类型，最终都会在GENERIC类型线程池内执行数据恢复任务。

需要注意的是，每个分片有自己的分配流程：

不需要等所有主分片都分配完毕才执行副分片的分配；
不需要等所有分片都分配完才走 recovery 流程；
主分片不需要等副分片分配成功才进入主分片的 recovery，主副分片有自己的 recovery 流程；

## 主分片Recovery
EMPTY_STORE 和 EXISTING_STORE核心都是从Primary Shard恢复数据。 由于每次写操作都会记录事务（translog），以及相关的数据。因此将最后一次提交之后的translog 中进行重放，建立Lucene 索引，如此完成主分片的recovery。

### INIT阶段
RecoveryState类初始化代码stage = Stage.INIT 标记Recovery都是从INIT阶段开始。markAsRecovering("from store", recoveryState)方法判断IndexShardState状态，并通知对应的IndexShardState监听器，入参recoveryState对应的状态在Recovery类初始化完成。
```
public IndexShardState markAsRecovering(String reason, RecoveryState recoveryState) throws IndexShardStartedException,
    IndexShardRelocatedException, IndexShardRecoveringException, IndexShardClosedException {
    synchronized (mutex) {
        if (state == IndexShardState.CLOSED) {
            throw new IndexShardClosedException(shardId);
        }
        if (state == IndexShardState.STARTED) {
            throw new IndexShardStartedException(shardId);
        }
        if (state == IndexShardState.RECOVERING) {
            throw new IndexShardRecoveringException(shardId);
        }
        if (state == IndexShardState.POST_RECOVERY) {
            throw new IndexShardRecoveringException(shardId);
        }
        this.recoveryState = recoveryState;
        return changeState(IndexShardState.RECOVERING, reason);
    }
}
-------------------------------------------------------------
private IndexShardState changeState(IndexShardState newState, String reason) {
    assert Thread.holdsLock(mutex);
    logger.debug("state: [{}]->[{}], reason [{}]", state, newState, reason);
    IndexShardState previousState = state;
    state = newState;
    //通知IndexShardState对应的监听器
    this.indexEventListener.indexShardStateChanged(this, previousState, newState, reason);
    return previousState;
}
```

### INDEX阶段
recoverFromStore方法是Shard恢复的核心，从本地文件系统中恢复，首先需要知道Shard在磁盘的具体位置，最终调用recoverFromStore方法：
```
boolean recoverFromStore(final IndexShard indexShard) {
    if (canRecover(indexShard)) {
        RecoverySource.Type recoveryType = indexShard.recoveryState().getRecoverySource().getType();
        return executeRecovery(indexShard, () -> {
            internalRecoverFromStore(indexShard);
        });
    }
    return false;
}
-----------------------------------------------------
private void internalRecoverFromStore(IndexShard indexShard) throws IndexShardRecoveryException {
    final RecoveryState recoveryState = indexShard.recoveryState();
    //一个全新的集群启动，indexShouldExists是true
    final boolean indexShouldExists = recoveryState.getRecoverySource().getType() != RecoverySource.Type.EMPTY_STORE;
    //进入INDEX阶段
    indexShard.prepareForIndexRecovery();
    SegmentInfos si = null;
    final Store store = indexShard.store();
    store.incRef();
    try {
        try {
            store.failIfCorrupted();
            try {
                //获取Lucene最后一次提交的分段信息，得到其中的版本号
                si = store.readLastCommittedSegmentsInfo();
            .........................
            }
            //一个全新的集群，Lucene有Segment信息，清除Lucene对应的Segment信息
            if (si != null && indexShouldExists == false) {
                // it exists on the directory, but shouldn't exist on the FS, its a leftover (possibly dangling)
                // its a "new index create" API, we have to do something, so better to clean it than use same data
                logger.trace("cleaning existing shard, shouldn't exists");
                Lucene.cleanLuceneIndex(store.directory());
                si = null;
            }
	    .........................
		
	if (recoveryState.getRecoverySource().getType() == RecoverySource.Type.LOCAL_SHARDS) {
            assert indexShouldExists;
            bootstrap(indexShard, store);
            writeEmptyRetentionLeasesFile(indexShard);
        } else if (indexShouldExists) {
            if (recoveryState.getRecoverySource().shouldBootstrapNewHistoryUUID()) {
                store.bootstrapNewHistory();
                writeEmptyRetentionLeasesFile(indexShard);
            }
            // since we recover from local, just fill the files and size
            try {
                final RecoveryState.Index index = recoveryState.getIndex();
                if (si != null) {
                    addRecoveredFileDetails(si, store, index);
                }
            
        } else {
            store.createEmpty(indexShard.indexSettings().getIndexVersionCreated().luceneVersion);
            final String translogUUID = Translog.createEmptyTranslog(
                indexShard.shardPath().resolveTranslog(), SequenceNumbers.NO_OPS_PERFORMED, shardId,
                indexShard.getPendingPrimaryTerm());
            store.associateIndexWithNewTranslog(translogUUID);
            writeEmptyRetentionLeasesFile(indexShard);
        }
        //从tanslog恢复数据，进入VERIFY_INDEX阶段及TRANSLOG阶段
        indexShard.openEngineAndRecoverFromTranslog();
        indexShard.getEngine().fillSeqNoGaps(indexShard.getPendingPrimaryTerm());
        //进入FINALIZE状态
        indexShard.finalizeRecovery();
        //进入DONE阶段
        indexShard.postRecovery("post recovery from shard_store");
}
```
其中prepareForIndexRecovery方法标记进入INDEX阶段。
```
public void prepareForIndexRecovery() {
    if (state != IndexShardState.RECOVERING) {
        throw new IndexShardNotRecoveringException(shardId, state);
    }
    //标记进入INDEX阶段
    recoveryState.setStage(RecoveryState.Stage.INDEX);
    assert currentEngineReference.get() == null;
}
```
store.readLastCommittedSegmentsInfo()读取Lucene最近已经Commit的Segment信息

### VERIFY_INDEX阶段
官网index.shard.check_on_startup参数，是否进行Lucene索引检测。这个也很好理解，如果从已有的Lucene索引恢复数据，索引损坏的话恢复肯定异常。

false(默认值) Open shard并不检测索引是否损坏；
checksum 检查物理损坏；
true检查物理和逻辑损坏都检测，缺点是消耗大量的内存和CPU资源；
官网提示，如果打开一个大索引，检测索引是否损坏耗时会比较长
indexShard.openEngineAndRecoverFromTranslog()从tanslog恢复数据，进入VERIFY_INDEX阶段。
同时还进入TRANSLOG阶段，启动engine，重放translog，建立Lucene索引。
```
public void openEngineAndRecoverFromTranslog() throws IOException {
    assert recoveryState.getStage() == RecoveryState.Stage.INDEX : "unexpected recovery stage [" + recoveryState.getStage() + "]";
    //检测Lucene索引是否有损坏，进入VERIFY_INDEX阶段
    maybeCheckIndex();
    //进入TRANSLOG阶段，启动engine，重放translog，建立Lucene索引
    recoveryState.setStage(RecoveryState.Stage.TRANSLOG);
    final RecoveryState.Translog translogRecoveryStats = recoveryState.getTranslog();
    //启动Engine
    final Engine.TranslogRecoveryRunner translogRecoveryRunner = (engine, snapshot) -> {
        translogRecoveryStats.totalOperations(snapshot.totalOperations());
        translogRecoveryStats.totalOperationsOnStart(snapshot.totalOperations());
        return runTranslogRecovery(engine, snapshot, Engine.Operation.Origin.LOCAL_TRANSLOG_RECOVERY,
            translogRecoveryStats::incrementRecoveredOperations);
    };
    loadGlobalCheckpointToReplicationTracker();
    innerOpenEngineAndTranslog(replicationTracker);
    //在TranslogRecoveryRunner的run方法内，重放translog日志
    getEngine().recoverFromTranslog(translogRecoveryRunner, Long.MAX_VALUE);
}
```
检查Lucene索引是否有损害，和index.shard.check_on_startup配置有关系.
```
public void maybeCheckIndex() {
    recoveryState.setStage(RecoveryState.Stage.VERIFY_INDEX);
    if (Booleans.isTrue(checkIndexOnStartup) || "checksum".equals(checkIndexOnStartup)) {
        try {
            checkIndex();
        } catch (IOException ex) {
            throw new RecoveryFailedException(recoveryState, "check index failed", ex);
        }
    }
}
```
## TRANSLOG阶段
TRANSLOG阶段需要重放事务日志中尚未刷入磁盘的信息。根据最后一次提交的信息做快照，来确定事物日志中哪些数据需要重放，重放完毕后将新生成的Lucene数据刷入磁盘。对应的源码在InternalEngine的recoverFromTranslogInternal方法
```
private void recoverFromTranslogInternal(TranslogRecoveryRunner translogRecoveryRunner, long recoverUpToSeqNo) throws IOException {
    Translog.TranslogGeneration translogGeneration = translog.getGeneration();
    final int opsRecovered;
    //根据最后一次提交的信息生成translog快照
    final long translogFileGen = Long.parseLong(lastCommittedSegmentInfos.getUserData().get(Translog.TRANSLOG_GENERATION_KEY));
    //定义Translog的Snapshot
    try (Translog.Snapshot snapshot = translog.newSnapshotFromGen(
        new Translog.TranslogGeneration(translog.getTranslogUUID(), translogFileGen), recoverUpToSeqNo)) {
        //重放这些日志
        opsRecovered = translogRecoveryRunner.run(this, snapshot);
    } catch (Exception e) {
        throw new EngineException(shardId, "failed to recover from translog", e);
    }
    // flush if we recovered something or if we have references to older translogs
    // note: if opsRecovered == 0 and we have older translogs it means they are corrupted or 0 length.
    assert pendingTranslogRecovery.get() : "translogRecovery is not pending but should be";
    pendingTranslogRecovery.set(false); // we are good - now we can commit
    //将重放后新生成的数据刷入硬盘
    if (opsRecovered > 0) {
        logger.trace("flushing post recovery from translog. ops recovered [{}]. committed translog id [{}]. current id [{}]",
            opsRecovered, translogGeneration == null ? null :
                translogGeneration.translogFileGeneration, translog.currentFileGeneration());
        commitIndexWriter(indexWriter, translog, null);
        refreshLastCommittedSegmentInfos();
        refresh("translog_recovery");
    }
    translog.trimUnreferencedReaders();
}
```
重放Translog的具体实现在TranslogHandler类的run方法：
```
final Engine.TranslogRecoveryRunner translogRecoveryRunner = (engine, snapshot) -> {
    translogRecoveryStats.totalOperations(snapshot.totalOperations());
    translogRecoveryStats.totalOperationsOnStart(snapshot.totalOperations());
    //从当前Node的Shard恢复
    return runTranslogRecovery(engine, snapshot, Engine.Operation.Origin.LOCAL_TRANSLOG_RECOVERY,
        translogRecoveryStats::incrementRecoveredOperations);
};
```
（1）入参snapshot 的生成如下：
```
Translog.Snapshot snapshot = translog.newSnapshotFromGen(
            new Translog.TranslogGeneration(translog.getTranslogUUID(), translogFileGen), recoverUpToSeqNo)
```
（2）入参Engine.Operation.Origin是枚举类型
```
public enum Origin {
    //写主分片
    PRIMARY,
    //写副本分片
    REPLICA,
    //Peer类型，从其他Node的主分片恢复当前Node的副本分片
    PEER_RECOVERY,
    //同一Node的恢复
    LOCAL_TRANSLOG_RECOVERY,
    LOCAL_RESET;
```
（3）runTranslogRecovery的具体实现方法如下，通过applyTranslogOperation执行索引的INDEX DELETE等操作。
```
int runTranslogRecovery(Engine engine, Translog.Snapshot snapshot, Engine.Operation.Origin origin,
                        Runnable onOperationRecovered) throws IOException {
    int opsRecovered = 0;
    Translog.Operation operation;
    //Translog.Snapshot Translog文件的快照，迭代获取对应的操作
    while ((operation = snapshot.next()) != null) {
        try {
            logger.trace("[translog] recover op {}", operation);
            //执行Translog重放
            Engine.Result result = applyTranslogOperation(engine, operation, origin);
            switch (result.getResultType()) {
                case FAILURE:
                    throw result.getFailure();
                case MAPPING_UPDATE_REQUIRED:
                    throw new IllegalArgumentException("unexpected mapping update: " + result.getRequiredMappingUpdate());
                case SUCCESS:
                    break;
                default:
                    throw new AssertionError("Unknown result type [" + result.getResultType() + "]");
            }

            opsRecovered++;
            //数据记录
            onOperationRecovered.run();
        }
```

## FINALIZE阶段
indexShard.finalizeRecovery()进入到 FINALIZE阶段，执行refresh操作。refresh操作调用lucene的ReferenceManager，真正的refresh操作由lucene执行，将Index buffer中的数据生成一个segment并放入系统缓存区，同时清空Index buffer。
```
public void finalizeRecovery() {
    //进入FINALIZE阶段
    recoveryState().setStage(RecoveryState.Stage.FINALIZE);
    Engine engine = getEngine();
    //执行refresh操作
    engine.refresh("recovery_finalization");
    engine.config().setEnableGcDeletes(true);
}
```

## DONE阶段
通过indexShard.postRecovery方法进入DONE阶段，这里还好更新shard状态为POST_RECOVERY。
```
public void postRecovery(String reason) throws IndexShardStartedException, IndexShardRelocatedException, IndexShardClosedException {
    synchronized (postRecoveryMutex) {
        //再次执行刷新
        getEngine().refresh("post_recovery");
        synchronized (mutex) {
            if (state == IndexShardState.CLOSED) {
                throw new IndexShardClosedException(shardId);
            }
            if (state == IndexShardState.STARTED) {
                throw new IndexShardStartedException(shardId);
            }
            //Shard进入DONE阶段
            recoveryState.setStage(RecoveryState.Stage.DONE);
            //更新分片状态
            changeState(IndexShardState.POST_RECOVERY, reason);
        }
    }
```
internalRecoverFromStore核心方法执行完成后，需要向Master节点报告成功还是失败。

## 反馈Master节点结果
无论成功还是失败，Shard恢复的结果都会通过RPC请求通知Master节点。
```
try {
    if (recoverFromStore()) {
        //恢复成功，向Master发送action为 internal:cluster/shard/started的RPC请求
        recoveryListener.onRecoveryDone(recoveryState);
    }
} catch (Exception e) {
    //恢复失败，关闭Engin，向Master发送 internal:cluster/shard/failure的RPC请求
    recoveryListener.onRecoveryFailure(recoveryState,
        new RecoveryFailedException(recoveryState, null, e), true);
}
```
恢复成功对应的Action是SHARD_STARTED_ACTION_NAME，这里和创建Shard的逻辑一致。

当前Node的副本分片恢复的核心思想是从其他Node的主分片拉取Lucene分段和translog进行恢复。按数据传递的方向，主分片节点称为Source，副分片称为Traget。

为什么需要拉取主分片的translog？

副本分片数是可以修改的，如果副本分片从主分片拉取数据恢复期间，主分片不能写则对应整个索引不能执行写操作，这确实是很难接受。因此副本恢复数据期间，主分片应该支持写操作。从复制Lucene分段的那一刻开始，所恢复的副分片数据不包括新增的内容，而这些内容存在于主分片中的translog中，因此副分片需要从主分片节点拉取translog进行重放，以获取新增内容。这就是需要主分片节点的translog不被清理。为防止主分片节点的translog被清理，6.0版本开始，引入 TranslogDeletionPolicy (事务日志删除策略)的概念，负责维护活跃的translog文件。

个人认为，ES对Translog文件的读写与删除，受Lucene索引的读写与删除影响还是比较大的。Lucene索引的写入逻辑是IndexWriter，读取文是IndexReader，删除策略是IndexDeletionPolicy。Translog文件写入时TranslogWriter，文件读取时TranslogReader，删除策略是TranslogDeletionPolicy。

副分片的恢复需要与主分片一致，同时，恢复期间允许新的索引操作。在6.x 版中分为两个阶段：

阶段一：在主分片所在节点，获取translog 保留锁（保留translog 不受刷盘清空影响）。然后调用Lucene 接口把shard做成快照进行复制。在阶段二之前，副分片就可以正常处理写请求了
阶段二：重放新数据。对trannslog 做快照，这个快照包含从阶段一开始到阶段二快照期间对新增索引

PEER恢复类型：其中PeerRecoverySourceService主要负责Source端的工作，PeerRecoveryTargetService则主要负责Target端的工作。PeerRecoveryTargetService的初始化方法：
```
public PeerRecoverySourceService(TransportService transportService, IndicesService indicesService,
                                 RecoverySettings recoverySettings) {
    this.transportService = transportService;
    this.indicesService = indicesService;
    this.recoverySettings = recoverySettings;
    //PeerRecoverySourceService构造函数
    //StartRecoveryTransportRequestHandler主要负责处理Target端启动恢复的
    //请求，创建RecoverySourceHandler对象实例，开始恢复流程
    transportService.registerRequestHandler(Actions.START_RECOVERY, ThreadPool.Names.GENERIC, StartRecoveryRequest::new,
        new StartRecoveryTransportRequestHandler());
}
```
PeerRecoverySourceService初始化方法，Action注册信息如下：
```
public PeerRecoveryTargetService(ThreadPool threadPool, TransportService transportService,
        RecoverySettings recoverySettings, ClusterService clusterService) {

    //负责接受处理Source端Shard快照文件
    transportService.registerRequestHandler(Actions.FILES_INFO, ThreadPool.Names.GENERIC, RecoveryFilesInfoRequest::new,
        new FilesInfoRequestHandler());
    //负责接受Source端发送的Shard文件块
    transportService.registerRequestHandler(Actions.FILE_CHUNK, ThreadPool.Names.GENERIC, RecoveryFileChunkRequest::new,
        new FileChunkTransportRequestHandler());
    //负责将接受到的Shard文件更改为正式文件
    transportService.registerRequestHandler(Actions.CLEAN_FILES, ThreadPool.Names.GENERIC,
        RecoveryCleanFilesRequest::new, new CleanFilesRequestHandler());
    //负责打开Engine，准备接受Source端Translog进行日志重放
    transportService.registerRequestHandler(Actions.PREPARE_TRANSLOG, ThreadPool.Names.GENERIC,
            RecoveryPrepareForTranslogOperationsRequest::new, new PrepareForTranslogOperationsRequestHandler());
    //负责接受Source端Translog，进行日志重放
    transportService.registerRequestHandler(Actions.TRANSLOG_OPS, ThreadPool.Names.GENERIC, RecoveryTranslogOperationsRequest::new,
        new TranslogOperationsRequestHandler());
    //负责刷新Engine，让新的Segment生效，回收旧的文件，更新global checkpoint等
    transportService.registerRequestHandler(Actions.FINALIZE, ThreadPool.Names.GENERIC, RecoveryFinalizeRecoveryRequest::new,
        new FinalizeRecoveryRequestHandler());
    //如果是主分片relocate，则负责接受主分片身份
    transportService.registerRequestHandler(
            Actions.HANDOFF_PRIMARY_CONTEXT,
            ThreadPool.Names.GENERIC,
            RecoveryHandoffPrimaryContextRequest::new,
            new HandoffPrimaryContextRequestHandler());
}
```
在PEER恢复类型中，根据PeerRecoverySourceService和PeerRecoveryTargetService初始化服务Action注册信息，能够清晰理解副本分片恢复流程。

## 副分片Recovery
从其他Node的主分片恢复当前Node的副本分片，对应的就是PEER类型，对应的入口是：
```
case PEER:
    //副本分片从主分片远程拉取，Shard 复制
    try {
        markAsRecovering("from " + recoveryState.getSourceNode(), recoveryState);
        recoveryTargetService.startRecovery(this, recoveryState.getSourceNode(), recoveryListener);
    }
```
startRecovery方法也是通过 generic类型的线程池执行数据恢复，最终由RecoveryRunner的run 方法执行。
```
public void startRecovery(final IndexShard indexShard, final DiscoveryNode sourceNode, final RecoveryListener listener) {
    // create a new recovery status, and process...
    final long recoveryId = onGoingRecoveries.startRecovery(indexShard, sourceNode, listener, recoverySettings.activityTimeout());
    threadPool.generic().execute(new RecoveryRunner(recoveryId));
}
```

### INIT 阶段
PeerRecoveryTargetService是当前节点副本分片恢复数据的核心类，对应的方法是doRecovery
```
private void doRecovery(final long recoveryId) {
    final StartRecoveryRequest request;
    final RecoveryState.Timer timer;
    CancellableThreads cancellableThreads;
    try (RecoveryRef recoveryRef = onGoingRecoveries.getRecovery(recoveryId)) {
        final RecoveryTarget recoveryTarget = recoveryRef.target();
        timer = recoveryTarget.state().getTimer();
        cancellableThreads = recoveryTarget.cancellableThreads();
        try {
            assert recoveryTarget.sourceNode() != null : "can not do a recovery without a source node";
            logger.trace("{} preparing shard for peer recovery", recoveryTarget.shardId());
            //进入到INIT阶段
            recoveryTarget.indexShard().prepareForIndexRecovery();
            final long startingSeqNo = recoveryTarget.indexShard().recoverLocallyUpToGlobalCheckpoint();
            assert startingSeqNo == UNASSIGNED_SEQ_NO || recoveryTarget.state().getStage() == RecoveryState.Stage.TRANSLOG :
                "unexpected recovery stage [" + recoveryTarget.state().getStage() + "] starting seqno [ " + startingSeqNo + "]";
            request = getStartRecoveryRequest(logger, clusterService.localNode(), recoveryTarget, startingSeqNo);
        }
		....................
        //在GENERIC线程池内，使用新的cancellableThreads线程（线程可以被中断）执行RPC操作
        cancellableThreads.executeIO(() ->
        //发送RPC请求，对应的ACTION是PeerRecoverySourceService.Actions.START_RECOVERY
        transportService.submitRequest(request.sourceNode(), PeerRecoverySourceService.Actions.START_RECOVERY, request,
            new TransportResponseHandler<RecoveryResponse>() {
                @Override
                public void handleResponse(RecoveryResponse recoveryResponse) {
                    final TimeValue recoveryTime = new TimeValue(timer.time());
                    // do this through ongoing recoveries to remove it from the collection
                    //响应成功，设置为DONE阶段
                    onGoingRecoveries.markRecoveryAsDone(recoveryId);
    );
```
其中recoveryTarget.indexShard().prepareForIndexRecovery();是进入到INIT阶段
```
public void prepareForIndexRecovery() {
    if (state != IndexShardState.RECOVERING) {
        throw new IndexShardNotRecoveringException(shardId, state);
    }
    //标记进入INDEX阶段
    recoveryState.setStage(RecoveryState.Stage.INDEX);
    assert currentEngineReference.get() == null;
}
```

## INDEX阶段
INDEX 阶段负责将主分片的 Lucene 数据复制到副分片节点。在INIT阶段，副本分片向主分片所在Node发送Actions.START_RECOVERY请求，这里开始响应主分片所在Node处理结果的响应。

（1）Actions.FILES_INFO对应的处理器，核心是把对应的元数据信息进行记录：
```
class FilesInfoRequestHandler implements TransportRequestHandler<RecoveryFilesInfoRequest> {

    //Target端接收FILES_INFO请求后，会使用注册的FilesInfoRequestHandler
    @Override
    public void messageReceived(RecoveryFilesInfoRequest request, TransportChannel channel, Task task) throws Exception {
        try (RecoveryRef recoveryRef = onGoingRecoveries.getRecoverySafe(request.recoveryId(), request.shardId())) {
            final ActionListener<TransportResponse> listener = new ChannelActionListener<>(channel, Actions.FILES_INFO, request);
            recoveryRef.target().receiveFileInfo(
                request.phase1FileNames, request.phase1FileSizes, request.phase1ExistingFileNames, request.phase1ExistingFileSizes,
                request.totalTranslogOps, ActionListener.map(listener, nullVal -> TransportResponse.Empty.INSTANCE));
        }
    }
--------------------------------------------------------
public void receiveFileInfo(List<String> phase1FileNames,
                            List<Long> phase1FileSizes,
                            List<String> phase1ExistingFileNames,
                            List<Long> phase1ExistingFileSizes,
                            int totalTranslogOps,
                            ActionListener<Void> listener) {
    ActionListener.completeWith(listener, () -> {
        indexShard.resetRecoveryStage();
        indexShard.prepareForIndexRecovery();
        //Target在此主要记录发送过来的快照文件的相关信息，如文件名、文件大小、事务日志等相关信息
        final RecoveryState.Index index = state().getIndex();
        for (int i = 0; i < phase1ExistingFileNames.size(); i++) {
            index.addFileDetail(phase1ExistingFileNames.get(i), phase1ExistingFileSizes.get(i), true);
        }
        for (int i = 0; i < phase1FileNames.size(); i++) {
            index.addFileDetail(phase1FileNames.get(i), phase1FileSizes.get(i), false);
        }
        state().getTranslog().totalOperations(totalTranslogOps);
        state().getTranslog().totalOperationsOnStart(totalTranslogOps);
        return null;
    });
}
```
（2）Actions.FILES_INFO核心是Lucene文件恢复
```
class FileChunkTransportRequestHandler implements TransportRequestHandler<RecoveryFileChunkRequest> {

    // How many bytes we've copied since we last called RateLimiter.pause
    final AtomicLong bytesSinceLastPause = new AtomicLong();

    //Target端接收Source端发送过来的文件
    @Override
    public void messageReceived(final RecoveryFileChunkRequest request, TransportChannel channel, Task task) throws Exception {
        //根据RecoveryId获取对应的RecoveryRef
        try (RecoveryRef recoveryRef = onGoingRecoveries.getRecoverySafe(request.recoveryId(), request.shardId())) {
            //获取对应的RecoveryTarget
            final RecoveryTarget recoveryTarget = recoveryRef.target();
            final RecoveryState.Index indexState = recoveryTarget.state().getIndex();
            if (request.sourceThrottleTimeInNanos() != RecoveryState.Index.UNKNOWN) {
                indexState.addSourceThrottling(request.sourceThrottleTimeInNanos());
            }

            //这里默认使用Lucene的SimpleRateLimiter实现限速
            RateLimiter rateLimiter = recoverySettings.rateLimiter();

            final ActionListener<TransportResponse> listener = new ChannelActionListener<>(channel, Actions.FILE_CHUNK, request);
            //Lucene文件恢复
            recoveryTarget.writeFileChunk(request.metadata(), request.position(), request.content(), request.lastChunk(),
                request.totalTranslogOps(), ActionListener.map(listener, nullVal -> TransportResponse.Empty.INSTANCE));
        }
    }
--------------------------------------------------
public void writeFileChunk(StoreFileMetaData fileMetaData, long position, BytesReference content,
                           boolean lastChunk, int totalTranslogOps, ActionListener<Void> listener) {
        state().getTranslog().totalOperations(totalTranslogOps);
        //单个Writer，副本shard对应Lucene文件写入
        multiFileWriter.writeFileChunk(fileMetaData, position, content, lastChunk);
        listener.onResponse(null);
```
（3）Actions.CLEAN_FILES 负责把Shard文件更改为正式文件
```
class CleanFilesRequestHandler implements TransportRequestHandler<RecoveryCleanFilesRequest> {

    //Source端发送完所有差异文件之后，会发送CLEAN_FILES请求给Target端，Target端处理CLEAN_FILES的handler是
    @Override
    public void messageReceived(RecoveryCleanFilesRequest request, TransportChannel channel, Task task) throws Exception {
        try (RecoveryRef recoveryRef = onGoingRecoveries.getRecoverySafe(request.recoveryId(), request.shardId())) {
            final ActionListener<TransportResponse> listener = new ChannelActionListener<>(channel, Actions.CLEAN_FILES, request);
            recoveryRef.target().cleanFiles(request.totalTranslogOps(), request.getGlobalCheckpoint(), request.sourceMetaSnapshot(),
                ActionListener.map(listener, nullVal -> TransportResponse.Empty.INSTANCE));
        }
    }
}
---------------------------------
    public void cleanFiles(int totalTranslogOps, long globalCheckpoint, Store.MetadataSnapshot sourceMetaData,
                       ActionListener<Void> listener) {
    ActionListener.completeWith(listener, () -> {
        state().getTranslog().totalOperations(totalTranslogOps);
        // first, we go and move files that were created with the recovery id suffix to
        // the actual names, its ok if we have a corrupted index here, since we have replicas
        // to recover from in case of a full cluster shutdown just when this code executes...
        //将从Source端接收文件时创建的临时命名为正式文件，根据注释可知可能会覆盖Target端已有文件。
        multiFileWriter.renameAllTempFiles();
        final Store store = store();
        store.incRef();
        try {
            //Target端清理本地原有文件，但是根据Source端发送过来恢复信息判断恢复之后不需要的文件，并对接收的文件进行验证
            store.cleanupAndVerify("recovery CleanFilesRequestHandler", sourceMetaData);
            if (indexShard.indexSettings().getIndexVersionCreated().before(Version.V_6_0_0_rc1)) {
                store.ensureIndexHasHistoryUUID();
            }
            final String translogUUID = Translog.createEmptyTranslog(
                indexShard.shardPath().resolveTranslog(), globalCheckpoint, shardId, indexShard.getPendingPrimaryTerm());
            store.associateIndexWithNewTranslog(translogUUID);

            //进入VERIFY_INDEX阶段
            indexShard.maybeCheckIndex();
            //开始进入TransLog阶段
            state().setStage(RecoveryState.Stage.TRANSLOG);
        }
```

## VERIFY_INDEX阶段
indexShard.maybeCheckIndex() 进入VERIFY_INDEX阶段，可以通过index.shard.check_on_startup参数配置，检测Lucene索引 是否损坏（2.3 VERIFY_INDEX阶段）
```
//检查Lucene索引是否有损害，和INDEX_CHECK_ON_STARTUP配置有关系
public void maybeCheckIndex() {
    recoveryState.setStage(RecoveryState.Stage.VERIFY_INDEX);
    if (Booleans.isTrue(checkIndexOnStartup) || "checksum".equals(checkIndexOnStartup)) {
        try {
            checkIndex();
        } catch (IOException ex) {
            throw new RecoveryFailedException(recoveryState, "check index failed", ex);
        }
    }
}
```
Actions.PREPARE_TRANSLOG: 副分片对此action的主要处理是启动Engine，使副分片可以正常接收写请求
```
class PrepareForTranslogOperationsRequestHandler implements TransportRequestHandler<RecoveryPrepareForTranslogOperationsRequest> {

    @Override
    public void messageReceived(RecoveryPrepareForTranslogOperationsRequest request, TransportChannel channel, Task task) {
        try (RecoveryRef recoveryRef = onGoingRecoveries.getRecoverySafe(request.recoveryId(), request.shardId())) {
            final ActionListener<TransportResponse> listener = new ChannelActionListener<>(channel, Actions.PREPARE_TRANSLOG, request);
            //副本分片启动Engine
            recoveryRef.target().prepareForTranslogOperations(request.totalTranslogOps(),
                ActionListener.map(listener, nullVal -> TransportResponse.Empty.INSTANCE));
        }
    }
------------------------------------------------
public void prepareForTranslogOperations(int totalTranslogOps, ActionListener<Void> listener) {
    ActionListener.completeWith(listener, () -> {
        state().getTranslog().totalOperations(totalTranslogOps);
        indexShard().openEngineAndSkipTranslogRecovery();
        return null;
    });
}
```

## TRANSLOG阶段
Actions.TRANSLOG_OPS：接受Source端Translog，进行Translog重放。
```
class TranslogOperationsRequestHandler implements TransportRequestHandler<RecoveryTranslogOperationsRequest> {

    @Override
    public void messageReceived(final RecoveryTranslogOperationsRequest request, final TransportChannel channel,
                                Task task) throws IOException {
        //根据RecoveryId获取对应的RecoveryRef
        try (RecoveryRef recoveryRef =
                 onGoingRecoveries.getRecoverySafe(request.recoveryId(), request.shardId())) {
            final ClusterStateObserver observer = new ClusterStateObserver(clusterService, null, logger, threadPool.getThreadContext());
            //获取对应的RecoveryTarget
            final RecoveryTarget recoveryTarget = recoveryRef.target();
            final ActionListener<RecoveryTranslogOperationsResponse> listener =
                new ChannelActionListener<>(channel, Actions.TRANSLOG_OPS, request);
            
            final IndexMetaData indexMetaData = clusterService.state().metaData().index(request.shardId().getIndex());
            final long mappingVersionOnTarget = indexMetaData != null ? indexMetaData.getMappingVersion() : 0L;
            recoveryTarget.indexTranslogOperations(
-----------------------------------
public void indexTranslogOperations(
        ....................................
        final ActionListener<Long> listener) {
    ActionListener.completeWith(listener, () -> {
        final RecoveryState.Translog translog = state().getTranslog();
        translog.totalOperations(totalTranslogOps);
        assert indexShard().recoveryState() == state();
        if (indexShard().state() != IndexShardState.RECOVERING) {
            throw new IndexShardNotRecoveringException(shardId, indexShard().state());
        }
       
        indexShard().updateRetentionLeasesOnReplica(retentionLeases);
        for (Translog.Operation operation : operations) {
            //TranslogOperation:Index、Delete
            Engine.Result result = indexShard().applyTranslogOperation(operation, Engine.Operation.Origin.PEER_RECOVERY);
```
## FINALIZE阶段
Actions.FINALIZE：负责刷新Engine，让新的Segment生效，回收旧的文件，更新global checkpoint。
```
class FinalizeRecoveryRequestHandler implements TransportRequestHandler<RecoveryFinalizeRecoveryRequest> {

    @Override
    public void messageReceived(RecoveryFinalizeRecoveryRequest request, TransportChannel channel, Task task) throws Exception {
        //根据RecoveryId获取对应的RecoveryRef
        try (RecoveryRef recoveryRef = onGoingRecoveries.getRecoverySafe(request.recoveryId(), request.shardId())) {
            final ActionListener<TransportResponse> listener = new ChannelActionListener<>(channel, Actions.FINALIZE, request);
            recoveryRef.target().finalizeRecovery(request.globalCheckpoint(), request.trimAboveSeqNo(),
                ActionListener.map(listener, nullVal -> TransportResponse.Empty.INSTANCE));
        }
    }
}
-------------------------------------------------
public void finalizeRecovery(final long globalCheckpoint, final long trimAboveSeqNo, ActionListener<Void> listener) {
    ActionListener.completeWith(listener, () -> {
        indexShard.updateGlobalCheckpointOnReplica(globalCheckpoint, "finalizing recovery");
        // Persist the global checkpoint.
        // global checkpoint数据持久化
        indexShard.sync();
        indexShard.persistRetentionLeases();
        
        //进入FINALIZE阶段
        indexShard.finalizeRecovery();
        return null;
    });
}
```
## 主分片操作
注意，这里是主分片节点Source端的处理逻辑。PeerRecoveryTargetService类初始化RPC注册信息：

### START_RECOVERY
接下来副本分片所在的Node发送RPC请求，对应的ACTION是PeerRecoverySourceService.Actions.START_RECOVERY。发送STARTRECOVERY请求给Source端，启动恢复流程。其中PeerRecoverySourceService类在Guice中初始化是进行registerRequestHandler的注册：
```
@Inject
public PeerRecoverySourceService(TransportService transportService, IndicesService indicesService,
                                 RecoverySettings recoverySettings) {
    this.transportService = transportService;
    this.indicesService = indicesService;
    this.recoverySettings = recoverySettings;
    transportService.registerRequestHandler(Actions.START_RECOVERY, ThreadPool.Names.GENERIC, StartRecoveryRequest::new,
        new StartRecoveryTransportRequestHandler());
}
```
通过registerRequestHandler方法很容易看出来，在其他Node 主分片响应当前Node的RPC请求对应的处理过程。其他Node响应RPC请求处理过程，入口还是熟悉的messageReceived方法。
```
class StartRecoveryTransportRequestHandler implements TransportRequestHandler<StartRecoveryRequest> {
    @Override
    public void messageReceived(final StartRecoveryRequest request, final TransportChannel channel, Task task) throws Exception {
        recover(request, new ChannelActionListener<>(channel, Actions.START_RECOVERY, request));
    }
}
```
接下来的核心代码在RecoverySourceHandler类的recoverToTarget方法内：
```
private void recover(StartRecoveryRequest request, ActionListener<RecoveryResponse> listener) {
    final IndexService indexService = indicesService.indexServiceSafe(request.shardId().getIndex());
    //根据shardId获取对应的IndexShard
    final IndexShard shard = indexService.getShard(request.shardId().id());
    //获取对应的ShardRouting（标识分片的状态、所属Node等信息）
    final ShardRouting routingEntry = shard.routingEntry();

    //判断routingEntry是否为主、routingEntry是否active，否抛错
    if (routingEntry.primary() == false || routingEntry.active() == false) {
        throw new DelayRecoveryException("source shard [" + routingEntry + "] is not an active primary");
    }

    //根据request得到shard并构造RecoverySourceHandler对象
    // 这里创建RecoverySourceHandler用于主导恢复流程
    RecoverySourceHandler handler = ongoingRecoveries.addNewRecovery(request, shard);
    logger.trace("[{}][{}] starting recovery to {}", request.shardId().getIndex().getName(), request.shardId().id(),
        request.targetNode());
    //实际的恢复流程函数，Source端所有的恢复流程都是这个函数定义的，后续
    //的恢复流程都是Source主导，发送各种请求让Target端进行文件恢复Translog重放等。
    handler.recoverToTarget(ActionListener.runAfter(listener, () -> ongoingRecoveries.remove(shard, handler)));
}
```
实际创建Handler在shardContext.addNewRecovery(request, shard) 通过createRecoverySourceHandler创建：
```
private RecoverySourceHandler createRecoverySourceHandler(StartRecoveryRequest request, IndexShard shard) {
    RecoverySourceHandler handler;
    //RecoverySourceHandler会持有一个RemoteRecoveryTargetHandler
    //用于和Target端进行交互，后续各种请求都是通过这个对象进行交互的。
    final RemoteRecoveryTargetHandler recoveryTarget =
        new RemoteRecoveryTargetHandler(request.recoveryId(), request.shardId(), transportService,
            request.targetNode(), recoverySettings, throttleTime -> shard.recoveryStats().addThrottleTime(throttleTime));
    handler = new RecoverySourceHandler(shard, recoveryTarget, shard.getThreadPool(), request,
        Math.toIntExact(recoverySettings.getChunkSize().getBytes()), recoverySettings.getMaxConcurrentFileChunks());
    return handler;
}
```
接下来分析RecoverySourceHandler类的recoverToTarget方法，启动恢复流程。核心流程是phase1和phase2两个方法。phase1阶段主要向Target端发送三种请求:

（1）单个FILES_INFO告知Target即将接收的快照文件的相关信息；

（2）多个FILE_CHUNK进行文件传输；

（3）CLEAN_FILES告诉Target端清理原本存在但是恢复到Source端状态却不再需要的文件，CLEAN_FILES也会要求Target端对接收到的文件进行验证；

能够跳过phase1方法是canSkipPhase1，如果主副分片有相同的synid且doc数量相同，则跳过phase1
```
boolean canSkipPhase1(Store.MetadataSnapshot source, Store.MetadataSnapshot target) {
    //条件1:源的syncid和目标的syncid一致，且不为null
    if (source.getSyncId() == null || source.getSyncId().equals(target.getSyncId()) == false) {
        return false;
    }
    //条件2:源的docnum和目标的docnum一致
    if (source.getNumDocs() != target.getNumDocs()) {
        throw new IllegalStateException("try to recover " + request.shardId() + " from primary shard with sync id but number " +
            "of docs differ: " + source.getNumDocs() + " (" + request.sourceNode().getName() + ", primary) vs " + target.getNumDocs()
            + "(" + request.targetNode().getName() + ")");
    }
    SequenceNumbers.CommitInfo sourceSeqNos = SequenceNumbers.loadSeqNoInfoFromLuceneCommit(source.getCommitUserData().entrySet());
    SequenceNumbers.CommitInfo targetSeqNos = SequenceNumbers.loadSeqNoInfoFromLuceneCommit(target.getCommitUserData().entrySet());
    //条件3: 基于源seqno的本地chckpoint和基于目标seqno的本地checkpoint一致
    if (sourceSeqNos.localCheckpoint != targetSeqNos.localCheckpoint || targetSeqNos.maxSeqNo != sourceSeqNos.maxSeqNo) {
        final String message = "try to recover " + request.shardId() + " with sync id but " +
            "seq_no stats are mismatched: [" + source.getCommitUserData() + "] vs [" + target.getCommitUserData() + "]";
        assert false : message;
        throw new IllegalStateException(message);
    }
    //可以跳过phase1，源分片和目标分片的syncId一致且doc数量相同
    return true;
}
```
## FILES_INFO
通过FILESINFO请求将Shard快照文件信息发给Target端。在recoverToTarget方法内，先计算出阶段1需要发送文件的FilesName，FileSize等信息。
```
//RemoteRecoveryTargetHandler recoveryTarget
// 发送PeerRecoveryTargetService.Actions.FILES_INFO对应的Action请求
recoveryTarget.receiveFileInfo(phase1FileNames, phase1FileSizes, phase1ExistingFileNames,
        phase1ExistingFileSizes, translogOps.getAsInt(), sendFileInfoStep);
--------------------------------------------------
public void receiveFileInfo(List<String> phase1FileNames, List<Long> phase1FileSizes, List<String> phase1ExistingFileNames,
                            List<Long> phase1ExistingFileSizes, int totalTranslogOps, ActionListener<Void> listener) {
    //Request构建
    RecoveryFilesInfoRequest recoveryInfoFilesRequest = new RecoveryFilesInfoRequest(recoveryId, shardId,
        phase1FileNames, phase1FileSizes, phase1ExistingFileNames, phase1ExistingFileSizes, totalTranslogOps);

    //发送RPC请求，对应Actions.FILES_INFO
    transportService.submitRequest(targetNode, PeerRecoveryTargetService.Actions.FILES_INFO, recoveryInfoFilesRequest,
        TransportRequestOptions.builder().withTimeout(recoverySettings.internalActionTimeout()).build(),
        new ActionListenerResponseHandler<>(ActionListener.map(listener, r -> null),
            in -> TransportResponse.Empty.INSTANCE, ThreadPool.Names.GENERIC));
}
```

## FILE_CHUNK
通过多个FILE_CHUNK请求将Shard快照文件发送给Target端。主分片所在Node和副本分片所在Node数据传输过程必定是耗时比较长的，显然是使用异步方式更合理，ES在这里使用AsyncIOProcessor（允许批量的IO操作）。
```
sendFileInfoStep.whenComplete(r ->
    //phase1阶段1内，发送文件内容，通过文件块FILE_CHUNK的方式，将文件发送给Target端
    sendFiles(store, phase1Files.toArray(new StoreFileMetaData[0]), translogOps, sendFilesStep), listener::onFailure);
--------------------------------------------------
void sendFiles(Store store, StoreFileMetaData[] files, IntSupplier translogOps, ActionListener<Void> listener) {
    ArrayUtil.timSort(files, Comparator.comparingLong(StoreFileMetaData::length)); // send smallest first
    final ThreadContext threadContext = threadPool.getThreadContext();
    //匿名内部类，依靠AsyncIOProcessor异步发送塑化剂
    final MultiFileTransfer<FileChunk> multiFileSender =
        new MultiFileTransfer<FileChunk>(logger, threadContext, listener, maxConcurrentFileChunks, Arrays.asList(files)) {

            @Override
            protected void sendChunkRequest(FileChunk request, ActionListener<Void> listener) {
                cancellableThreads.checkForCancel();
                //数据发送
                recoveryTarget.writeFileChunk(
                    request.md, request.position, request.content, request.lastChunk, translogOps.getAsInt(), listener);
            }

    resources.add(multiFileSender);
    //发送数据交给AsyncIOProcessor处理
    multiFileSender.start();
}
--------------------------------------------------
public void writeFileChunk(StoreFileMetaData fileMetaData, long position, BytesReference content,
                           boolean lastChunk, int totalTranslogOps, ActionListener<Void> listener) {
    // Pause using the rate limiter, if desired, to throttle the recovery
    final long throttleTimeInNanos;
    // always fetch the ratelimiter - it might be updated in real-time on the recovery settings
    //Lucene的限流器
    final RateLimiter rl = recoverySettings.rateLimiter();
    //发送RPC请求，对应Actions.FILE_CHUNK
    transportService.submitRequest(targetNode, PeerRecoveryTargetService.Actions.FILE_CHUNK,
        new RecoveryFileChunkRequest(recoveryId, shardId, fileMetaData, position, content, lastChunk,
```

## CLEAN_FILES
发送CLEAN FILES告诉Target端将临时文件更名为正式文件。
```
createRetentionLeaseStep.whenComplete(retentionLease ->
    {
        final long lastKnownGlobalCheckpoint = shard.getLastKnownGlobalCheckpoint();
        //发送CLEAN_FILES请求，告诉Target端临时文件中的文件命名为正式名称，删除旧的文件等。
        cleanFiles(store, recoverySourceMetadata, translogOps, lastKnownGlobalCheckpoint, cleanFilesStep);
    },
    listener::onFailure);
-------------------------------------------------------
public void cleanFiles(int totalTranslogOps, long globalCheckpoint, Store.MetadataSnapshot sourceMetaData,
                       ActionListener<Void> listener) {
    //发送RPC请求，对应Actions.CLEAN_FILES
    transportService.submitRequest(targetNode, PeerRecoveryTargetService.Actions.CLEAN_FILES,
            new RecoveryCleanFilesRequest(recoveryId, shardId, sourceMetaData, totalTranslogOps, globalCheckpoint),
            TransportRequestOptions.builder().withTimeout(recoverySettings.internalActionTimeout()).build(),
            new ActionListenerResponseHandler<>(ActionListener.map(listener, r -> null),
                in -> TransportResponse.Empty.INSTANCE, ThreadPool.Names.GENERIC));
}
```
## PREPARE_TRANSLOG
发送PREPARE TRANSLOG请求让Target打开Engine，准备接收Translog重放，这是个在Phase1和Phase2之间的Action。
```
sendFileStep.whenComplete(r -> {
    assert Transports.assertNotTransportThread(RecoverySourceHandler.this + "[prepareTargetForTranslog]");
    // For a sequence based recovery, the target can keep its local translog
    // 发送PREPARE TRANSLOG请求让Target打开Engine，准备接收Translog重放
    prepareTargetForTranslog(
        shard.estimateNumberOfHistoryOperations("peer-recovery", historySource, startingSeqNo), prepareEngineStep);
}, onFailure);
-------------------------------------
public void prepareForTranslogOperations(int totalTranslogOps, ActionListener<Void> listener) {
    //发送RPC请求，对应Actions.PREPARE_TRANSLOG
    transportService.submitRequest(targetNode, PeerRecoveryTargetService.Actions.PREPARE_TRANSLOG,
        new RecoveryPrepareForTranslogOperationsRequest(recoveryId, shardId, totalTranslogOps),
        TransportRequestOptions.builder().withTimeout(recoverySettings.internalActionTimeout()).build(),
        new ActionListenerResponseHandler<>(ActionListener.map(listener, r -> null),
            in -> TransportResponse.Empty.INSTANCE, ThreadPool.Names.GENERIC));
}
```

### TRANSLOG_OPS
接下来进入到phase2阶段，通过TRANSLOG OPS将Translog发送给Target重放。
```
sendBatch(
        readNextBatch,
        true,
        SequenceNumbers.UNASSIGNED_SEQ_NO,
        snapshot.totalOperations(),
        maxSeenAutoIdTimestamp,
        maxSeqNoOfUpdatesOrDeletes,
        retentionLeases,
        mappingVersion,
        batchedListener);
-------------------------------------------------
recoveryTarget.indexTranslogOperations(
    operations,
    totalTranslogOps,
    maxSeenAutoIdTimestamp,
    maxSeqNoOfUpdatesOrDeletes,
    retentionLeases,
    mappingVersionOnPrimary,
    ActionListener.wrap(
-------------------------------------------------
public void indexTranslogOperations(
        final List<Translog.Operation> operations,
        final int totalTranslogOps,
        final long maxSeenAutoIdTimestampOnPrimary,
        final long maxSeqNoOfDeletesOrUpdatesOnPrimary,
        final RetentionLeases retentionLeases,
        final long mappingVersionOnPrimary,
        final ActionListener<Long> listener) {
    //Request构建
    final RecoveryTranslogOperationsRequest request = new RecoveryTranslogOperationsRequest(
            
    //发送RPC请求，对应Actions.TRANSLOG_OPS
    transportService.submitRequest(targetNode, PeerRecoveryTargetService.Actions.TRANSLOG_OPS, request, translogOpsRequestOptions,
        new ActionListenerResponseHandler<>(ActionListener.map(listener, r -> r.localCheckpoint),
            RecoveryTranslogOperationsResponse::new, ThreadPool.Names.GENERIC));
}
```
## FINALIZE
发送FINALIZE请求让Target刷新Engine，让新的Segment生效，回收旧的文件，更新global checkpoint等
```
//发送FINALIZE请求让Target刷新Engine，让新的Segment生效，回收旧的文件，更新global checkpoint等
sendSnapshotStep.whenComplete(r -> finalizeRecovery(r.targetLocalCheckpoint, trimAboveSeqNo, finalizeStep), onFailure);
----------------------------------------------------
recoveryTarget.finalizeRecovery(globalCheckpoint, trimAboveSeqNo, finalizeListener);
----------------------------------------------------
public void finalizeRecovery(final long globalCheckpoint, final long trimAboveSeqNo, final ActionListener<Void> listener) {
    //发送RPC请求，对应Actions.FINALIZE
    transportService.submitRequest(targetNode, PeerRecoveryTargetService.Actions.FINALIZE,
        new RecoveryFinalizeRecoveryRequest(recoveryId, shardId, globalCheckpoint, trimAboveSeqNo),
        TransportRequestOptions.builder().withTimeout(recoverySettings.internalActionLongTimeout()).build(),
        new ActionListenerResponseHandler<>(ActionListener.map(listener, r -> null),
            in -> TransportResponse.Empty.INSTANCE, ThreadPool.Names.GENERIC));
}
```

## 小结
### recovery相关监控命令
（1）_cat/recovery

列出活跃的和已完成的recovery信息

（2）{index}/_recovery

此API展示特定索引的recovery所处阶段，以及每个分片、每个阶段的详细信息

（3）_stats

返回分片的sync_id、local_checkpoint、global_checkpoint等信息

### 对比
主分片恢复的主要阶段是 TRANSLOG 阶段
副分片恢复的主要阶段是 INDEX和TRANSLOG 阶段
只有phase1有限速配置，phase2不限速

## Recovery速度优化
一个大数据集群的recovery过程要消耗额外的资源，CPU、内存、节点间的网络带宽等。可能导致集群的服务性能下降，甚至部分功能暂时无法使用。

（1）减少集群full restart造成的数据来回拷贝

当遇到整个集群重启，如果一个Node向启动就执行Recovery显然不合理，会造成数据多次拷贝。

在集群启动过程中，一旦有了多少个节点成功启动，就执行recovery过程，这个命令将master节点（有master资格的节点）和data节点都算在内。gateway.expected_nodes: 3
有几个master节点启动成功，就执行recovery的过程。gateway.expected_master_nodes: 3
有几个data节点启动成功，就执行recovery的过程。gateway.expected_data_nodes: 3
gateway.recover_after_time recovery过程会等待gateway.recover_after_time指定的时间，一旦等待超时
示例，如果有以下配置的集群：
```
gateway.expected_data_nodes: 10
gateway.recover_after_time: 5m
gateway.recover_after_data_nodes: 8
```
此时的集群在5分钟内，有10个data节点都加入集群，或者5分钟后有8个以上的data节点加入集群，都会启动recovery的过程。

（2）减少主副本之间的数据复制

如果不是full restart，而是重启单个节点，也会造成不同节点之间来复制，为了避免这个问题，可以在重启之前，关闭集群的shard allocation。
```
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.enable":"none"
  }
}
```
当节点重启后，再重新打开：
```
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.enable":"all"
  }
}
```