---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码Gateway

## 集群文件
ElasticSearch集群生成的文件，从整体上看分为三个部分：
- Lucene文件（腊八粥：ElasticSearch源码：Lucene操作）；
- Translog文件；
- 集群元数据对应文件

本文详细介绍集群元数据对应文件持久化及恢复的流程，也可以认为是集群元数据持久化的补充。在4. 集群元数据持久化中，元数据只划分了集群层面和索引层面，这里进行细化，多了shard层面元数据。

## 元数据

### 元数据分类
元数据信息分为以下几种：

nodes/0/_state/*.st 集群层面元信息，对应ClusterState主要是 clusterUUID、settings、templates 等；
nodes/0/indices/{index_uuid}/_state/*.st 索引层面元信息，对应ClusterState主要是 numberOfShards、mappings 等；
nodes/0/indices/{index_uuid}/0/_state/*.st ，分片层面元信息，对应ClusterState主要是 version、indexUUID、primary 等；

元数据	集群层面元信息	索引层面元信息	分片层面元信息
路径	nodes/0/_state/*.st	nodes/0/indices/{index_uuid}/_state/*.st	nodes/0/indices/{index_uuid}/0/_state/*.st
对应ClusterState	MetaData（集群层）主要是 clusterUUID、settings、templates 等	lndexMetaData（索引层）主要是 numberOfShards、mappings 等	ShardStateMetaData（分片层）主要是 version、indexUUID、primary 等

MetaDataStateFormat 是元数据文件对应实现类的抽象类，有若干匿名内部实现类，对应不同文件的读写操作。其中，toXContent方法是对应写如文件字段信息的接口；fromXContent则是读取对应文件接口；子类根据需要具体实现。

## 集群元数据
集群元数据持久化的内容主要是前面描述的 MetaData 对象中去除 indices 索引部分。包括动态配置、模板信息、选举信息等。主要包含三部分：

manifest-数字.st：该文件是一个磁盘元数据管理入口，主要管理元数据的版本信息。
global-数字.st：MetaData 中的集群元数据的主要信息就是持久化到这个文件，包括动态配置、模板信息等。
node-数字.st：当前节点的元数据信息，包括节点的 nodeId 和版本信息。
集群元数据信息分三种类型：

API：接口API请求时，多返回的数据；

GATEWAY：持久化存储的元数据内容；

SNAPSHOT：形成snapshot类型的元数据内容；

（1）manifest文件

manifest文件的路径是：${data.path}/nodes/${node.id}/_state/manifest-xxx.st

manifest核心作用是管理元数据的版本信息，写入的元数据信息：current_term/cluster_state_version/generation/index_generations(数组类型)。
```
public static final MetaDataStateFormat<Manifest> FORMAT = new MetaDataStateFormat<Manifest>(MANIFEST_FILE_PREFIX) {

    //manifest文件写入
    @Override
    public void toXContent(XContentBuilder builder, Manifest state) throws IOException {
        state.toXContent(builder, MANIFEST_FORMAT_PARAMS);
};
------------------------------------------------------------
private static final ParseField CURRENT_TERM_PARSE_FIELD = new ParseField("current_term");
private static final ParseField CLUSTER_STATE_VERSION_PARSE_FIELD = new ParseField("cluster_state_version");
private static final ParseField GENERATION_PARSE_FIELD = new ParseField("generation");
private static final ParseField INDEX_GENERATIONS_PARSE_FIELD = new ParseField("index_generations");

@Override
public XContentBuilder toXContent(XContentBuilder builder, Params params) throws IOException {
    builder.field(CURRENT_TERM_PARSE_FIELD.getPreferredName(), currentTerm);
    builder.field(CLUSTER_STATE_VERSION_PARSE_FIELD.getPreferredName(), clusterStateVersion);
    builder.field(GENERATION_PARSE_FIELD.getPreferredName(), globalGeneration);
    builder.array(INDEX_GENERATIONS_PARSE_FIELD.getPreferredName(), indexEntryList().toArray());
    return builder;
}
```
（2）global文件

global文件的路径是：{data.path}/nodes/${node.id}/_state/global-xxx.st

集群元数据的主要信息就是持久化到global文件，持久化的内容如下：

Version用来表示元数据信息的版本号，版本号越大，说明元数据信息越全；
Cluster_uuid用于唯一表示是当前实例是属于哪个集群；
Template是指索引模版， 存放的是各个index的mapping；
```
//global-xx.st元数据文件
private static MetaDataStateFormat<MetaData> createMetaDataStateFormat(boolean preserveUnknownCustoms) {
    return new MetaDataStateFormat<MetaData>(GLOBAL_STATE_FILE_PREFIX) {

        @Override
    public void toXContent(XContentBuilder builder, MetaData state) throws IOException {
        Builder.toXContent(state, builder, FORMAT_PARAMS);
    }
-----------------------------------------------------------------
public static void toXContent(MetaData metaData, XContentBuilder builder, ToXContent.Params params) throws IOException {
    XContentContext context = XContentContext.valueOf(params.param(CONTEXT_MODE_PARAM, "API"));

    builder.startObject("meta-data");

    builder.field("version", metaData.version());
    builder.field("cluster_uuid", metaData.clusterUUID);
    builder.field("cluster_uuid_committed", metaData.clusterUUIDCommitted);

    builder.startObject("cluster_coordination");
    metaData.coordinationMetaData().toXContent(builder, params);
    builder.endObject();

    //持久化内容不为空
    if (!metaData.persistentSettings().isEmpty()) {
        builder.startObject("settings");
        metaData.persistentSettings().toXContent(builder, new MapParams(Collections.singletonMap("flat_settings", "true")));
        builder.endObject();
    }

    //临时设置信息，接口API请求时多返回的
    if (context == XContentContext.API && !metaData.transientSettings().isEmpty()) {
        builder.startObject("transient_settings");
        metaData.transientSettings().toXContent(builder, new MapParams(Collections.singletonMap("flat_settings", "true")));
        builder.endObject();
    }

    builder.startObject("templates");
    for (ObjectCursor<IndexTemplateMetaData> cursor : metaData.templates().values()) {
        IndexTemplateMetaData.Builder.toXContentWithTypes(cursor.value, builder, params);
    }
    builder.endObject();

    //索引信息，接口API请求时多返回的
    if (context == XContentContext.API && !metaData.indices().isEmpty()) {
        builder.startObject("indices");
        for (IndexMetaData indexMetaData : metaData) {
            IndexMetaData.Builder.toXContent(indexMetaData, builder, params);
        }
        builder.endObject();
    }

    //custom自定义类型元数据
    for (ObjectObjectCursor<String, Custom> cursor : metaData.customs()) {
        if (cursor.value.context().contains(context)) {
            builder.startObject(cursor.key);
            cursor.value.toXContent(builder, params);
            builder.endObject();
        }
    }
    builder.endObject();
}
```
（3）node文件

node文件的路径是：{data.path}/nodes/${node.id}/_state/node-xxx.st

node文件存储当前节点的元数据信息，包括节点的 nodeId 和版本信息，对应node_id和node_version字段。
```
static class NodeMetaDataStateFormat extends MetaDataStateFormat<NodeMetaData> {

    private ObjectParser<Builder, Void> objectParser;
	//node-xx.st文件存储信息
    @Override
    public void toXContent(XContentBuilder builder, NodeMetaData nodeMetaData) throws IOException {
        builder.field(NODE_ID_KEY, nodeMetaData.nodeId);
        builder.field(NODE_VERSION_KEY, nodeMetaData.nodeVersion.id);
    }
```
很明显看出，并不是所有的ClusterState字段内容都会持久化到磁盘，尤其是需要注意路由信息：RoutingTable/RoutingNodes/DiscoveryNodes等信息并持久化，仅仅存在JVM内存中。集群完全重启时，依靠gateway的recovery过程重建RoutingTable。

## 索引元数据
索引元数据存储对应文件的路径是：

${data.path}/nodes/${node.id}/indices/${index.UUID}/_state/state-xxx.st

索引元数据存储的核心字段信息及对应源码如下：

in_sync_allocations：每个分片分配之后都有一个唯一的 allocationId，该列表是主分片用来维护被认为跟自己保持数据一致的副本列表。从这个列表中踢出的分片不会接收新的写入，需要走分片恢复流程将数据追齐之后才能再被放入该队列。
mappings：索引的 mapping 信息，定义各个字段的类型、属性。
settings：索引自己的配置信息。
state：索引状态，OPEN/CLOSED。
aliases：索引别名信息。
routing_num_shards：索引分片数量。
primary_terms：每一轮分片切主都会产生新的 primary term，用于保持主从分片之间的一致性
```
public static final MetaDataStateFormat<IndexMetaData> FORMAT = new MetaDataStateFormat<IndexMetaData>(INDEX_STATE_FILE_PREFIX) {

    //索引元数据信息
    @Override
    public void toXContent(XContentBuilder builder, IndexMetaData state) throws IOException {
        Builder.toXContent(state, builder, FORMAT_PARAMS);
    }
-----------------------------------
public static void toXContent(IndexMetaData indexMetaData, XContentBuilder builder, ToXContent.Params params) throws IOException {
    builder.startObject(indexMetaData.getIndex().getName());

    //version
    builder.field(KEY_VERSION, indexMetaData.getVersion());
    //mapping_version
    builder.field(KEY_MAPPING_VERSION, indexMetaData.getMappingVersion());
    //settings_version
    builder.field(KEY_SETTINGS_VERSION, indexMetaData.getSettingsVersion());
    //aliases_version
    builder.field(KEY_ALIASES_VERSION, indexMetaData.getAliasesVersion());
    //routing_num_shards
    builder.field(KEY_ROUTING_NUM_SHARDS, indexMetaData.getRoutingNumShards());
    //state,索引状态OPEN or CLOSE.
    builder.field(KEY_STATE, indexMetaData.getState().toString().toLowerCase(Locale.ENGLISH));
    //是否使用二进制压缩
    boolean binary = params.paramAsBoolean("binary", false);

    //settings
    builder.startObject(KEY_SETTINGS);
    indexMetaData.getSettings().toXContent(builder, new MapParams(Collections.singletonMap("flat_settings", "true")));
    builder.endObject();
    //mappings
    builder.startArray(KEY_MAPPINGS);
    for (ObjectObjectCursor<String, MappingMetaData> cursor : indexMetaData.getMappings()) {
        if (binary) {
            builder.value(cursor.value.source().compressed());
        } else {
            builder.map(XContentHelper.convertToMap(new BytesArray(cursor.value.source().uncompressed()), true).v2());
        }
    }
    builder.endArray();

    for (ObjectObjectCursor<String, DiffableStringMap> cursor : indexMetaData.customData) {
        builder.field(cursor.key);
        builder.map(cursor.value);
    }

    //aliases
    builder.startObject(KEY_ALIASES);
    for (ObjectCursor<AliasMetaData> cursor : indexMetaData.getAliases().values()) {
        AliasMetaData.Builder.toXContent(cursor.value, builder, params);
    }
    builder.endObject();
    //primary_terms
    builder.startArray(KEY_PRIMARY_TERMS);
    for (int i = 0; i < indexMetaData.getNumberOfShards(); i++) {
        builder.value(indexMetaData.primaryTerm(i));
    }
    builder.endArray();
    //in_sync_allocations
    builder.startObject(KEY_IN_SYNC_ALLOCATIONS);
    for (IntObjectCursor<Set<String>> cursor : indexMetaData.inSyncAllocationIds) {
        builder.startArray(String.valueOf(cursor.key));
        for (String allocationId : cursor.value) {
            builder.value(allocationId);
        }
        builder.endArray();
    }
    builder.endObject();
    //rollover_info
    builder.startObject(KEY_ROLLOVER_INFOS);
    for (ObjectCursor<RolloverInfo> cursor : indexMetaData.getRolloverInfos().values()) {
        cursor.value.toXContent(builder, params);
    }
    builder.endObject();

    builder.endObject();
}
```

## Shard元数据
shard元数据存储对应文件的路径是：

${data.path}/nodes/${node.id}/indices/${index.UUID}/${shard.id}/_state/state-xxx.st

shard元数据存储字段信息：primary/index_uuid/allocation_id

Primary表示当前是不是Primary Shard。
Index_uuid表示是哪个索引的shard
```
public static final MetaDataStateFormat<ShardStateMetaData> FORMAT =
    new MetaDataStateFormat<ShardStateMetaData>(SHARD_STATE_FILE_PREFIX) {

    //shard层级元信息序列化
    @Override
    public void toXContent(XContentBuilder builder, ShardStateMetaData shardStateMetaData) throws IOException {
        //primary
        builder.field(PRIMARY_KEY, shardStateMetaData.primary);
        //index_uuid
        builder.field(INDEX_UUID_KEY, shardStateMetaData.indexUUID);
        if (shardStateMetaData.allocationId != null) {
            //allocation_id
            builder.field(ALLOCATION_ID_KEY, shardStateMetaData.allocationId);
        }
    }
```
Shard层级的元数据信息，实在创建shard或更新shard时存储或更新，而不是Node启动时。

## 元数据文件分布
（1）在 7.6.0 版本之前，元数据直接存放在每个节点磁盘上，且在专有 master 节点上每个索引一个元数据目录。对应的示意图如：

从目录上看，每个层级_state文件夹下的数据，都是对应层级的元数据信息。 集群/索引/shard层级元数据，都对应一个_state文件夹，相应的索引文件存储在对应的文件夹下。

本文源码版本是7.5.2，这里只讨论当前元数据文件分布情况
（2）在 7.6.0 版本之后，ES 将元数据放到节点本地独立的 lucene 索引中保存。data/nodes/0 里面保存的是 segment 文件，元数据以本地 Lucene index 方式持久化，收敛文件数量。其中 node-0.st 是旧版升级后的兼容文件：

## Gateway
Gateway 模块负责集群元信息的存储和集群重启时的恢复。Gateway 阶段恢复的集群状态中，我们已经知道集群一共有多少个索引，每个索引的主副分片各有多少个，但是不知道它们位于哪个节点，现在需要找到它们都位于哪个节点。集群完全重启的初始状态，所有分片都被标记为未分配状态，此处也被称作分片分配过程。因此分片分配的概念不仅仅是分配一个全新分片。对于索引某个特定分片的分配过程中，先分配其主分片，后分配其副分片。

## 元数据持久化
只有Master和Data类型节点能够持久化集群状态，当节点收到Master节点发布的ClusterState时，节点判断ClusterState是否发生变化，若发生变化则将其持久化到磁盘。

如果模块需要对集群状态进行处理，则需要从接口类ClusterStateApplier实现，实现其中的applyClusterState方法。例如：
```
private static class GatewayClusterApplier implements ClusterStateApplier {
    private final IncrementalClusterStateWriter incrementalClusterStateWriter;

    /**增加一个事件，更改集群状态
     *
     * 如果模块需要对集群状态进行处理，则需要从接口类ClusterStateApplier实现，实现其中的applyClusterState方法，例如:
     */
    @Override
    public void applyClusterState(ClusterChangedEvent event) {
        //判断是否禁止持久化
        if (event.state().blocks().disableStatePersistence()) {
            //禁止incrementalClusterStateWriter执行写操作
            incrementalClusterStateWriter.setIncrementalWrite(false);
            return;
        }

        try {
            //确保term是递增
            if (event.state().term() > incrementalClusterStateWriter.getPreviousManifest().getCurrentTerm()) {
                //写 manifest 文件，只更新term
                incrementalClusterStateWriter.setCurrentTerm(event.state().term());
            }

            incrementalClusterStateWriter.updateClusterState(event.state());
            //incrementalClusterStateWriter标记为可写状态
            incrementalClusterStateWriter.setIncrementalWrite(true);
        } catch (WriteStateException e) {
            logger.warn("Exception occurred when storing new meta data", e);
        }
    }
```
（1）manifest文件更新term

Manifest文件写入首先完成Manifest对象的初始化。previousManifest对象来自Node初始化时，对GatewayMetaState的初始化，在对应的start方法内，更新集群元数据及对应的Manifest对象。
```
//Node启动时，更新元数据信息
upgradeMetaData(settings, metaStateService, metaDataIndexUpgradeService, metaDataUpgrader); 
//加载ClusterState
manifestClusterStateTuple = loadStateAndManifest(ClusterName.CLUSTER_NAME_SETTING.get(settings), metaStateService);
```
完成Manifest对象初始化，开始写入manifest文件。
```
void setCurrentTerm(long currentTerm) throws WriteStateException {
    Manifest manifest = new Manifest(currentTerm, previousManifest.getClusterStateVersion(), previousManifest.getGlobalGeneration(),
        new HashMap<>(previousManifest.getIndexGenerations()));
    //writes manifest file
    metaStateService.writeManifestAndCleanup("current term changed", manifest);
    previousManifest = manifest;
}
---------------------------------------------
public void writeManifestAndCleanup(String reason, Manifest manifest) throws WriteStateException {
    logger.trace("[_meta] writing state, reason [{}]", reason);
    try {
        long generation = MANIFEST_FORMAT.writeAndCleanup(manifest, nodeEnv.nodeDataPaths());
        logger.trace("[_meta] state written (generation: {})", generation);
    } catch (WriteStateException ex) {
        throw new WriteStateException(ex.isDirty(), "[_meta]: failed to write meta state", ex);
    }
}
```
writeManifestAndCleanup会写Manifest文件，并删除老的Manifest文件。

MANIFEST_FORMAT 对应的是Manifest.FORMAT 即MetaDataStateFormat类（1.2 集群元数据）

（2）元数据更新

updateClusterState方法实现元数据的更新。
```
void updateClusterState(ClusterState newState) throws WriteStateException {
    //获取Master节点发布的最新MetaData
    MetaData newMetaData = newState.metaData();
    final AtomicClusterStateWriter writer = new AtomicClusterStateWriter(metaStateService, previousManifest);
    //写集群元数据
    long globalStateGeneration = writeGlobalState(writer, newMetaData);
    //写索引元数据
    Map<Index, Long> indexGenerations = writeIndicesMetadata(writer, newState);
    Manifest manifest = new Manifest(previousManifest.getCurrentTerm(), newState.version(), globalStateGeneration, indexGenerations);
    //manifest-**.st文件写入
    writeManifest(writer, manifest);
    previousManifest = manifest;
    previousClusterState = newState;
```
（3）writeGlobalState

集群元数据发生变更，写global文件。
```
private long writeGlobalState(AtomicClusterStateWriter writer, MetaData newMetaData) throws WriteStateException {
    //集群元数据发生变更
    if (incrementalWrite == false || MetaData.isGlobalStateEquals(previousClusterState.metaData(), newMetaData) == false) {
        return writer.writeGlobalState("changed", newMetaData);
    }
    return previousManifest.getGlobalGeneration();
}
----------------------------
long writeGlobalState(String reason, MetaData metaData) throws WriteStateException {
assert finished == false : FINISHED_MSG;
try {
    //只是增加写操作，并没有真正执行写
    rollbackCleanupActions.add(() -> metaStateService.cleanupGlobalState(previousManifest.getGlobalGeneration()));
    //Writes the global state
    long generation = metaStateService.writeGlobalState(reason, metaData);
    //只是增加写操作，并没有真正执行写
    commitCleanupActions.add(() -> metaStateService.cleanupGlobalState(generation));
    return generation;
} catch (WriteStateException e) {
    rollback();
    throw e;
}
```
writeGlobalState是执行Global文件的写入，写入的数据是MetaData，其中META_DATA_FORMAT来自1.2 集群元数据的匿名内部类。
```
long writeGlobalState(String reason, MetaData metaData) throws WriteStateException {
    logger.trace("[_global] writing state, reason [{}]", reason);
    try {
        long generation = META_DATA_FORMAT.write(metaData, nodeEnv.nodeDataPaths());
        logger.trace("[_global] state written");
        return generation;
    }
```
（4）writeIndicesMetadata

写索引元数据
```
private Map<Index, Long> writeIndicesMetadata(AtomicClusterStateWriter writer, ClusterState newState)
    throws WriteStateException {
    Map<Index, Long> previouslyWrittenIndices = previousManifest.getIndexGenerations();
    //获取对应索引（区分Master节点和Data节点）
    Set<Index> relevantIndices = getRelevantIndices(newState);

    Map<Index, Long> newIndices = new HashMap<>();

    MetaData previousMetaData = incrementalWrite ? previousClusterState.metaData() : null;
    //获取索引元数据
    Iterable<IndexMetaDataAction> actions = resolveIndexMetaDataActions(previouslyWrittenIndices, relevantIndices, previousMetaData,
        newState.metaData());

    for (IndexMetaDataAction action : actions) {
        //划分:新增；修改；保持不变三种实现
        long generation = action.execute(writer);
        newIndices.put(action.getIndex(), generation);
    }

    return newIndices;
}
-------------------------------
static List<IndexMetaDataAction> resolveIndexMetaDataActions(Map<Index, Long> previouslyWrittenIndices,
                                                             Set<Index> relevantIndices,
                                                             MetaData previousMetaData,
                                                             MetaData newMetaData) {
    List<IndexMetaDataAction> actions = new ArrayList<>();
    for (Index index : relevantIndices) {
        IndexMetaData newIndexMetaData = newMetaData.getIndexSafe(index);
        IndexMetaData previousIndexMetaData = previousMetaData == null ? null : previousMetaData.index(index);

        if (previouslyWrittenIndices.containsKey(index) == false || previousIndexMetaData == null) {
            //新增索引元数据
            actions.add(new WriteNewIndexMetaData(newIndexMetaData));
        } else if (previousIndexMetaData.getVersion() != newIndexMetaData.getVersion()) {
            //索引元数据修改
            actions.add(new WriteChangedIndexMetaData(previousIndexMetaData, newIndexMetaData));
        } else {
            //索引元数据保持不变
            actions.add(new KeepPreviousGeneration(index, previouslyWrittenIndices.get(index)));
        }
    }
    return actions;
}
```
针对索引元数据有新增/修改/保持不变三种情形有三种具体实现，修改或新增索引的具体实现如下，INDEX_META_DATA_FORMAT的具体实现是1.3 索引元数据中的匿名内部类：
```
long writeIndex(String reason, IndexMetaData metaData) throws WriteStateException {
    assert finished == false : FINISHED_MSG;
    try {
        Index index = metaData.getIndex();
        Long previousGeneration = previousManifest.getIndexGenerations().get(index);
        if (previousGeneration != null) {
            rollbackCleanupActions.add(() -> metaStateService.cleanupIndex(index, previousGeneration));
        }
        //真实索引元数据写操作过程
        long generation = metaStateService.writeIndex(reason, metaData);
        //这里也只是定义写操作，并没有真实执行
        commitCleanupActions.add(() -> metaStateService.cleanupIndex(index, generation));
        return generation;
    }
---------------------
public long writeIndex(String reason, IndexMetaData indexMetaData) throws WriteStateException {
    final Index index = indexMetaData.getIndex();
    logger.trace("[{}] writing state, reason [{}]", index, reason);
    try {
        long generation = INDEX_META_DATA_FORMAT.write(indexMetaData,
                nodeEnv.indexPaths(indexMetaData.getIndex()));
        logger.trace("[{}] state written", index);
        return generation;
    }
```
（5）writeManifest

写Manifest文件（之前只是更新term），还有个注意的是定义在commitCleanupActions的写操作，开始执行。
```
private void writeManifest(AtomicClusterStateWriter writer, Manifest manifest) throws WriteStateException {
    if (manifest.equals(previousManifest) == false) {
        //Manifest对应数据发生变化
        writer.writeManifestAndCleanup("changed", manifest);
    }
}
-------------------
void writeManifestAndCleanup(String reason, Manifest manifest) throws WriteStateException {
assert finished == false : FINISHED_MSG;
try {
    //磁盘文件写入
    metaStateService.writeManifestAndCleanup(reason, manifest);
    //集群元数据、索引元数据定义的写操作开始执行
    commitCleanupActions.forEach(Runnable::run);
    finished = true;
}
```
GatewayClusterApplier实现ClusterStateApplier，也是集群元数据在Gateway模块的应用。

## shard元数据
集群元数据在IndicesClusterStateService服务的应用，最后createOrUpdateShards方法提到shard的更新或创建。在初始化IndexShard，对应的方法内persistMetadata是持久化shard元数据：
```
/**
 * Shard级别元数据信息持久化
 */
persistMetadata(path, indexSettings, shardRouting, null, logger);
--------------------------------------------
//shard级别元数据持久化到磁盘
private static void persistMetadata(
        final ShardPath shardPath,
        final IndexSettings indexSettings,
        final ShardRouting newRouting,
        final @Nullable ShardRouting currentRouting,
        final Logger logger) throws IOException {
    assert newRouting != null : "newRouting must not be null";

    // only persist metadata if routing information that is persisted in shard state metadata actually changed
    final ShardId shardId = newRouting.shardId();
    if (currentRouting == null
        || currentRouting.primary() != newRouting.primary()
        || currentRouting.allocationId().equals(newRouting.allocationId()) == false) {
        final ShardStateMetaData newShardStateMetadata =
                new ShardStateMetaData(newRouting.primary(), indexSettings.getUUID(), newRouting.allocationId());
        /**持久化shard级别元数据信息*/
        ShardStateMetaData.FORMAT.writeAndCleanup(newShardStateMetadata, shardPath.getShardStatePath());
    }
```
其中ShardStateMetaData.FORMAT定义在 1.4 Shard元数据.

## 元数据恢复
当ES集群全部进行重启时，Recovery速度优化提到相关配置：
```
gateway.expected_data_nodes: 10
gateway.recover_after_time: 5m
gateway.recover_after_data_nodes: 8
```
此时的集群在5分钟内，有10个data节点都加入集群，或者5分钟后有8个以上的data节点加入集群，都会启动recovery的过程。

在Node初始化时，调用GatewayMetaState的start方法，针对Gateway模块增加集群状态应用处理器。
```
 /**增加集群状态应用处理器*/
clusterService.addLowPriorityApplier(new GatewayClusterApplier(incrementalClusterStateWriter));
```
其中，GatewayClusterApplier实现ClusterStateApplier。Node初始化时，集群状态发生变化开始执行GatewayClusterApplier中的applyClusterState方法。在IncrementalClusterStateWriter中完成ClusterState previousClusterState构建。

GatewayService在Node初始化时完成，核心是初始化相关参数。7.X版本后，集群启动默认使用Coordinator。
```
public GatewayService(final Settings settings, final AllocationService allocationService, final ClusterService clusterService,
                      final ThreadPool threadPool,
                      final TransportNodesListGatewayMetaState listGatewayMetaState,
                      final Discovery discovery) {
    // allow to control a delay of when indices will get created
    // gateway.expected_nodes:在集群启动过程中，一旦有了多少个节点成功启动，就执行recovery过程，这个命令将master节点（有master资格的节点）和data节点都算在内
    this.expectedNodes = EXPECTED_NODES_SETTING.get(settings);
    //gateway.expected_data_nodes:有几个data节点启动成功，就执行recovery的过程
    this.expectedDataNodes = EXPECTED_DATA_NODES_SETTING.get(settings);
    //gateway.expected_master_nodes:有几个master节点启动成功，就执行recovery的过程
    this.expectedMasterNodes = EXPECTED_MASTER_NODES_SETTING.get(settings);

    //gateway.recover_after_time
    if (RECOVER_AFTER_TIME_SETTING.exists(settings)) {
        recoverAfterTime = RECOVER_AFTER_TIME_SETTING.get(settings);
    } else if (expectedNodes >= 0 || expectedDataNodes >= 0 || expectedMasterNodes >= 0) {
        recoverAfterTime = DEFAULT_RECOVER_AFTER_TIME_IF_EXPECTED_NODES_IS_SET;
    } else {
        recoverAfterTime = null;
    }
    //gateway.recover_after_nodes
    this.recoverAfterNodes = RECOVER_AFTER_NODES_SETTING.get(settings);
    //gateway.recover_after_data_nodes
    this.recoverAfterDataNodes = RECOVER_AFTER_DATA_NODES_SETTING.get(settings);
    // default the recover after master nodes to the minimum master nodes in the discovery
    // gateway.recover_after_master_nodes
    if (RECOVER_AFTER_MASTER_NODES_SETTING.exists(settings)) {
        recoverAfterMasterNodes = RECOVER_AFTER_MASTER_NODES_SETTING.get(settings);
    } else if (discovery instanceof ZenDiscovery) {
        recoverAfterMasterNodes = settings.getAsInt("discovery.zen.minimum_master_nodes", -1);
    } else {
        recoverAfterMasterNodes = -1;
    }

    if (discovery instanceof Coordinator) {
        //7.x版本之后默认时Coordinator
        recoveryRunnable = () ->
                clusterService.submitStateUpdateTask("local-gateway-elected-state", new RecoverStateUpdateTask());
    }
```
当集群状态发生变化时，GatewayService对应的处理逻辑如下：

```
public void clusterChanged(final ClusterChangedEvent event) {    
	final ClusterState state = event.state();

    //当前节点不是Master节点
    if (state.nodes().isLocalNodeElectedMaster() == false) {
        // not our job to recover
        return;
    }
    //Master选举成功之后，判断其持有的集群状态中是否存在STATE_NOT_RECOVERED_BLOCK，
    // 如果不存在，则说明元数据已经恢复，跳过gateway恢复过程，否则等待
    if (state.blocks().hasGlobalBlock(STATE_NOT_RECOVERED_BLOCK) == false) {
        // already recovered
        return;
    }

    final DiscoveryNodes nodes = state.nodes();
    //元数据恢复，需要满足的条件：
    //（1）要求master选举成功（2）满足配置文件中给出的条件
    if (state.nodes().getMasterNodeId() == null) {
        logger.debug("not recovering from gateway, no master elected yet");
    } else if (recoverAfterNodes != -1 && (nodes.getMasterAndDataNodes().size()) < recoverAfterNodes) {
    ............................
        performStateRecovery(enforceRecoverAfterTime, reason);
    }
}
private void performStateRecovery(final boolean enforceRecoverAfterTime, final String reason) {
    if (enforceRecoverAfterTime && recoverAfterTime != null) {
        if (scheduledRecovery.compareAndSet(false, true)) {
            logger.info("delaying initial state recovery for [{}]. {}", recoverAfterTime, reason);
            //定时任务执行
            threadPool.schedule(new AbstractRunnable() {
                @Override
                protected void doRun() {
                    if (recoveryInProgress.compareAndSet(false, true)) {
                        logger.info("recover_after_time [{}] elapsed. performing state recovery...", recoverAfterTime);
                        recoveryRunnable.run();
                    }
                }
            }, recoverAfterTime, ThreadPool.Names.GENERIC);
```
recoveryRunnable对应的是：
```
recoveryRunnable = () ->
                    clusterService.submitStateUpdateTask("local-gateway-elected-state", new RecoverStateUpdateTask());
```
在当前流程execute方法的具体实现在 RecoverStateUpdateTask类。
```
//7.x之后，GatewayService对应
class RecoverStateUpdateTask extends ClusterStateUpdateTask {

    @Override
    public ClusterState execute(final ClusterState currentState) {
        //从获取的元数据信息中选择版本号最大的作为最新元数据，包括集群级、索引级
        final ClusterState newState = Function.<ClusterState>identity()
            //生成路由表
                .andThen(ClusterStateUpdaters::updateRoutingTable)
                .andThen(ClusterStateUpdaters::removeStateNotRecoveredBlock)
                .apply(currentState);

        //调用allocation模块的reroute，对未分配的分片执行分配，主分片分配过程中会异步获取各个shard级别元数据
        return allocationService.reroute(newState, "state recovered");
    }
```
通过allocationService.reroute生成新的ClusterState，最后，把新的ClusterState广播到集群中其他Node。