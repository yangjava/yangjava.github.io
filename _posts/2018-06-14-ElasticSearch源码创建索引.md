---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码创建索引
客户端Client发起Rest请求到服务端Master节点，经过网络模块的Handler分配dispatch后，查找到对应的Rest**Action，然后转换为Transport**Action。

Transport**Action需要 一个或多个Service、Action、Validators、Helper等组件才能完成。

## Master节点处理总结

### 创建索引
创建索引是在TransportCreateIndexAction类的masterOperation方法内提交任务到MetaDataCreateIndexService服务。
```
protected void masterOperation(final CreateIndexRequest request, final ClusterState state,
                               final ActionListener<CreateIndexResponse> listener) {
    String cause = request.cause();
    if (cause.length() == 0) {
        cause = "api";
    }
    //根据request获取indexName
    final String indexName = indexNameExpressionResolver.resolveDateMathExpression(request.index());
    //客户端提交创建索引的基本信息（索引名称、分区数、副本数等），提交到服务端，
    //服务端将CreateIndexRequest封装成CreateIndexClusterStateUpdateRequest
    final CreateIndexClusterStateUpdateRequest updateRequest =
        new CreateIndexClusterStateUpdateRequest(cause, indexName, request.index())
            .ackTimeout(request.timeout()).masterNodeTimeout(request.masterNodeTimeout())
            .settings(request.settings()).mappings(request.mappings())
            .aliases(request.aliases())
            .waitForActiveShards(request.waitForActiveShards());

    createIndexService.createIndex(updateRequest, ActionListener.map(listener, response ->
        new CreateIndexResponse(response.isAcknowledged(), response.isShardsAcknowledged(), indexName)));
}
```

### 删除索引
创建索引是在TransportDeleteIndexAction类的masterOperation方法内提交任务到MetaDataDeleteIndexService服务。
```
protected void masterOperation(final DeleteIndexRequest request, final ClusterState state,
                               final ActionListener<AcknowledgedResponse> listener) {
    final Set<Index> concreteIndices = new HashSet<>(Arrays.asList(indexNameExpressionResolver.concreteIndices(state, request)));
    if (concreteIndices.isEmpty()) {
        listener.onResponse(new AcknowledgedResponse(true));
        return;
    }

    DeleteIndexClusterStateUpdateRequest deleteRequest = new DeleteIndexClusterStateUpdateRequest()
        .ackTimeout(request.timeout()).masterNodeTimeout(request.masterNodeTimeout())
        .indices(concreteIndices.toArray(new Index[concreteIndices.size()]));

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

### 增加或删除索引别名
创建索引是在TransportIndicesAliasesAction类的masterOperation方法内提交任务到MetaDataIndexAliasesService服务。
```
@Override
protected void masterOperation(final IndicesAliasesRequest request, final ClusterState state,
                               final ActionListener<AcknowledgedResponse> listener) {

    ......................................................
    indexAliasesService.indicesAliases(updateRequest, new ActionListener<ClusterStateUpdateResponse>() {
        @Override
        public void onResponse(ClusterStateUpdateResponse response) {
            listener.onResponse(new AcknowledgedResponse(response.isAcknowledged()));
        }

        @Override
        public void onFailure(Exception t) {
            logger.debug("failed to perform aliases", t);
            listener.onFailure(t);
        }
    });
}
```
以上操作的主流程都是Transport**Action的masterOperation方法内把任务提交到对应的MetaData**Service服务上，然后再提交到ClusterService服务。

提交的任务是自定义类或匿名内部类，需要继承AckedClusterStateUpdateTask(允许在所有节点都已确认群集状态更新请求后返回给Client)

Master节点处理请求小结：

1.创建一个任务继承AckedClusterStateUpdateTask或ClusterStateUpdateTask；

2.把任务提交到对应的MetaData**Service组件，然后再提交到ClusterService组件；

## IndexCreationTask
IndexCreationTask类的execute方法内是执行创建索引的详细步骤，其中最重要的是indicesService.createIndex()创建索引服务。
```
public ClusterState execute(ClusterState currentState) throws Exception {
    Index createdIndex = null;
    String removalExtraInfo = null;
    //标识索引创建的状态
    IndexRemovalReason removalReason = IndexRemovalReason.FAILURE;
    try {
        /**
         * 阶段一：校验参数阶段
         * 校验当前ClusterState的currentState是否存在该索引，routingTable包含该index、metaData包含该index、alias别名等
         */
        //检查request的合法性，校验索引名/校验settings是否正常
        validator.validate(request, currentState);

        //校验是索引否已存在别名
        for (Alias alias : request.aliases()) {
            aliasValidator.validateAlias(alias, request.index(), currentState.metaData());
        }
        /**
         * 阶段二：配置合并阶段
         * 合并template和request传入的mapping、customs 数据，优先级上request配置优先于template。
         * 合并template和request的setting，优先级上request配置优先于template。
         */
        // we only find a template when its an API call (a new index)
        // find templates, highest order are better matching
        List<IndexTemplateMetaData> templates =
                MetaDataIndexTemplateService.findTemplates(currentState.metaData(), request.index());

        // add the request mapping
        Map<String, Map<String, Object>> mappings = new HashMap<>();

        Map<String, AliasMetaData> templatesAliases = new HashMap<>();

        List<String> templateNames = new ArrayList<>();

        // 保存request的mapping到临时变量mappings
        for (Map.Entry<String, String> entry : request.mappings().entrySet()) {
            Map<String, Object> mapping = MapperService.parseMapping(xContentRegistry, entry.getValue());
            assert mapping.size() == 1 : mapping;
            assert entry.getKey().equals(mapping.keySet().iterator().next()) : entry.getKey() + " != " + mapping;
            mappings.put(entry.getKey(), mapping);
        }

        final Index recoverFromIndex = request.recoverFrom();

        if (recoverFromIndex == null) {
            // apply templates, merging the mappings into the request mapping if exists
            // 合并template的mapping、customs、alias到request的参数当中
            for (IndexTemplateMetaData template : templates) {
                templateNames.add(template.getName());
                // 合并request和template的mapping变量
                for (ObjectObjectCursor<String, CompressedXContent> cursor : template.mappings()) {
                    String mappingString = cursor.value.string();
                    // 如果request包含该命名的mapping，就进行合并,mapping以request传入为主，合并命中的template
                    if (mappings.containsKey(cursor.key)) {
                        XContentHelper.mergeDefaults(mappings.get(cursor.key),
                            MapperService.parseMapping(xContentRegistry, mappingString));
                    } else if (mappings.size() == 1 && cursor.key.equals(MapperService.SINGLE_MAPPING_NAME)) {
                        // Typeless template with typed mapping
                        Map<String, Object> templateMapping = MapperService.parseMapping(xContentRegistry, mappingString);
                        assert templateMapping.size() == 1 : templateMapping;
                        assert cursor.key.equals(templateMapping.keySet().iterator().next()) :
                            cursor.key + " != " + templateMapping;
                        Map.Entry<String, Map<String, Object>> mappingEntry = mappings.entrySet().iterator().next();
                        templateMapping = Collections.singletonMap(
                                mappingEntry.getKey(),                       // reuse type name from the mapping
                                templateMapping.values().iterator().next()); // but actual mappings from the template
                        XContentHelper.mergeDefaults(mappingEntry.getValue(), templateMapping);
                    } else if (template.mappings().size() == 1 && mappings.containsKey(MapperService.SINGLE_MAPPING_NAME)) {
                        // Typed template with typeless mapping
                        Map<String, Object> templateMapping = MapperService.parseMapping(xContentRegistry, mappingString);
                        assert templateMapping.size() == 1 : templateMapping;
                        assert cursor.key.equals(templateMapping.keySet().iterator().next()) :
                            cursor.key + " != " + templateMapping;
                        Map<String, Object> mapping = mappings.get(MapperService.SINGLE_MAPPING_NAME);
                        templateMapping = Collections.singletonMap(
                                MapperService.SINGLE_MAPPING_NAME,           // make template mapping typeless
                                templateMapping.values().iterator().next());
                        XContentHelper.mergeDefaults(mapping, templateMapping);
                    } else {
                        // 如果request不包含该命名的mapping，就直接新增即可
                        mappings.put(cursor.key,
                            MapperService.parseMapping(xContentRegistry, mappingString));
                    }
                }
                //handle aliases
                // 以request带的alias作为为主，合并template的alias
                for (ObjectObjectCursor<String, AliasMetaData> cursor : template.aliases()) {
                    AliasMetaData aliasMetaData = cursor.value;
                    //if an alias with same name came with the create index request itself,
                    // ignore this one taken from the index template
                    if (request.aliases().contains(new Alias(aliasMetaData.alias()))) {
                        continue;
                    }
                    //if an alias with same name was already processed, ignore this one
                    if (templatesAliases.containsKey(cursor.key)) {
                        continue;
                    }

                    // Allow templatesAliases to be templated by replacing a token with the
                    // name of the index that we are applying it to
                    if (aliasMetaData.alias().contains("{index}")) {
                        String templatedAlias = aliasMetaData.alias().replace("{index}", request.index());
                        aliasMetaData = AliasMetaData.newAliasMetaData(aliasMetaData, templatedAlias);
                    }

                    aliasValidator.validateAliasMetaData(aliasMetaData, request.index(), currentState.metaData());
                    templatesAliases.put(aliasMetaData.alias(), aliasMetaData);
                }
            }
        }
        /***
         * 阶段三：构建IndexSettings阶段
         * 构建Settings.Builder indexSettingsBuilder对象，合并templates、request的数据，辅以默认配置值
         */
        // 合并templates的setting配置
        Settings.Builder indexSettingsBuilder = Settings.builder();
        if (recoverFromIndex == null) {
            // apply templates, here, in reverse order, since first ones are better matching
            for (int i = templates.size() - 1; i >= 0; i--) {
                indexSettingsBuilder.put(templates.get(i).settings());
            }
        }
        // now, put the request settings, so they override templates
        // 合并request的setting并覆盖templates的setting
        indexSettingsBuilder.put(request.settings());
        if (indexSettingsBuilder.get(IndexMetaData.SETTING_INDEX_VERSION_CREATED.getKey()) == null) {
            final DiscoveryNodes nodes = currentState.nodes();
            final Version createdVersion = Version.min(Version.CURRENT, nodes.getSmallestNonClientNodeVersion());
            indexSettingsBuilder.put(IndexMetaData.SETTING_INDEX_VERSION_CREATED.getKey(), createdVersion);
        }
        if (indexSettingsBuilder.get(SETTING_NUMBER_OF_SHARDS) == null) {
            final int numberOfShards = getNumberOfShards(indexSettingsBuilder);
            indexSettingsBuilder.put(SETTING_NUMBER_OF_SHARDS, settings.getAsInt(SETTING_NUMBER_OF_SHARDS, numberOfShards));
        }
        if (indexSettingsBuilder.get(SETTING_NUMBER_OF_REPLICAS) == null) {
            indexSettingsBuilder.put(SETTING_NUMBER_OF_REPLICAS, settings.getAsInt(SETTING_NUMBER_OF_REPLICAS, 1));
        }
        if (settings.get(SETTING_AUTO_EXPAND_REPLICAS) != null && indexSettingsBuilder.get(SETTING_AUTO_EXPAND_REPLICAS) == null) {
            indexSettingsBuilder.put(SETTING_AUTO_EXPAND_REPLICAS, settings.get(SETTING_AUTO_EXPAND_REPLICAS));
        }

        if (indexSettingsBuilder.get(SETTING_CREATION_DATE) == null) {
            indexSettingsBuilder.put(SETTING_CREATION_DATE, Instant.now().toEpochMilli());
        }
        indexSettingsBuilder.put(IndexMetaData.SETTING_INDEX_PROVIDED_NAME, request.getProvidedName());
        indexSettingsBuilder.put(SETTING_INDEX_UUID, UUIDs.randomBase64UUID());

        /**
         * 阶段四：构建IndexMetaData阶段
         *
         * 构建IndexMetaData.Builder的tmpImdBuilder对象并绑定indexSettingsBuilder生成的actualIndexSettings。
         * 通过tmpImdBuilder.build()构建构建IndexMetaData tmpImd对象。
         */
        // 组建IndexMetaData的builder
        final IndexMetaData.Builder tmpImdBuilder = IndexMetaData.builder(request.index());
        final Settings idxSettings = indexSettingsBuilder.build();
        int numTargetShards = IndexMetaData.INDEX_NUMBER_OF_SHARDS_SETTING.get(idxSettings);
        final int routingNumShards;
        final Version indexVersionCreated = IndexMetaData.SETTING_INDEX_VERSION_CREATED.get(idxSettings);
        final IndexMetaData sourceMetaData = recoverFromIndex == null ? null :
            currentState.metaData().getIndexSafe(recoverFromIndex);
        if (sourceMetaData == null || sourceMetaData.getNumberOfShards() == 1) {
            // in this case we either have no index to recover from or
            // we have a source index with 1 shard and without an explicit split factor
            // or one that is valid in that case we can split into whatever and auto-generate a new factor.
            if (IndexMetaData.INDEX_NUMBER_OF_ROUTING_SHARDS_SETTING.exists(idxSettings)) {
                routingNumShards = IndexMetaData.INDEX_NUMBER_OF_ROUTING_SHARDS_SETTING.get(idxSettings);
            } else {
                routingNumShards = calculateNumRoutingShards(numTargetShards, indexVersionCreated);
            }
        } else {
            assert IndexMetaData.INDEX_NUMBER_OF_ROUTING_SHARDS_SETTING.exists(indexSettingsBuilder.build()) == false
                : "index.number_of_routing_shards should not be present on the target index on resize";
            routingNumShards = sourceMetaData.getRoutingNumShards();
        }
        // remove the setting it's temporary and is only relevant once we create the index
        // 移除routing_shards配置
        indexSettingsBuilder.remove(IndexMetaData.INDEX_NUMBER_OF_ROUTING_SHARDS_SETTING.getKey());
        tmpImdBuilder.setRoutingNumShards(routingNumShards);

        if (recoverFromIndex != null) {
            assert request.resizeType() != null;
            prepareResizeIndexSettings(
                    currentState,
                    mappings.keySet(),
                    indexSettingsBuilder,
                    recoverFromIndex,
                    request.index(),
                    request.resizeType(),
                    request.copySettings(),
                    indexScopedSettings);
        }
        // 实际的IndexSetting
        final Settings actualIndexSettings = indexSettingsBuilder.build();

        /*
         * We can not check the shard limit until we have applied templates, otherwise we do not know the actual number of shards
         * that will be used to create this index.
         */
        checkShardLimit(actualIndexSettings, currentState);
        // IndexMetaData的builder添加实际的索引的设置
        tmpImdBuilder.settings(actualIndexSettings);

        if (recoverFromIndex != null) {
            /*
             * We need to arrange that the primary term on all the shards in the shrunken index is at least as large as
             * the maximum primary term on all the shards in the source index. This ensures that we have correct
             * document-level semantics regarding sequence numbers in the shrunken index.
             */
            final long primaryTerm =
                IntStream
                    .range(0, sourceMetaData.getNumberOfShards())
                    .mapToLong(sourceMetaData::primaryTerm)
                    .max()
                    .getAsLong();
            for (int shardId = 0; shardId < tmpImdBuilder.numberOfShards(); shardId++) {
                tmpImdBuilder.primaryTerm(shardId, primaryTerm);
            }
        }
        // Set up everything, now locally create the index to see that things are ok, and apply
        // 创建实际的IndexMetaData对象
        final IndexMetaData tmpImd = tmpImdBuilder.build();
        ActiveShardCount waitForActiveShards = request.waitForActiveShards();
        if (waitForActiveShards == ActiveShardCount.DEFAULT) {
            waitForActiveShards = tmpImd.getWaitForActiveShards();
        }
        if (waitForActiveShards.validate(tmpImd.getNumberOfReplicas()) == false) {
            throw new IllegalArgumentException("invalid wait_for_active_shards[" + request.waitForActiveShards() +
                "]: cannot be greater than number of shard copies [" +
                (tmpImd.getNumberOfReplicas() + 1) + "]");
        }
        /**阶段五：构建IndexService阶段
         * create the index here (on the master) to validate it can be created, as well as adding the mapping
         * 在master（当前）上创建索引，用作验证
         * 根据metadata创建IndexService, 如果创建成功这里就可以获取到对应的indexservice，否则会抛出异常
         */
        final IndexService indexService = indicesService.createIndex(tmpImd, Collections.emptyList());
        /**
         * 阶段六：获取Index和Map阶段: 获取新创建的index和mapping，
         * Index createdIndex = indexService.index()
         * MapperService mapperService = indexService.mapperService()。
         */
        createdIndex = indexService.index();
        // now add the mappings
        // 获取创建IndexService的mapperService。needsMapperService在创建索引时为true,mapperService不为null
        MapperService mapperService = indexService.mapperService();
        try {
            /**
             * 阶段七：更新mapping到mapperService阶段
             *
             * mapperService.merge()合并request和template合并后生成的最新的mappings。
             * 生成Map mappingsMetaData，key是创建mapping的指定的type
             */
            mapperService.merge(mappings, MergeReason.MAPPING_UPDATE);
        } catch (Exception e) {
            removalExtraInfo = "failed on parsing default mapping/mappings on index creation";
            throw e;
        }

        if (request.recoverFrom() == null) {
            // now that the mapping is merged we can validate the index sort.
            // we cannot validate for index shrinking since the mapping is empty
            // at this point. The validation will take place later in the process
            // (when all shards are copied in a single place).
            indexService.getIndexSortSupplier().get();
        }

        // the context is only used for validation so it's fine to pass fake values for the shard id and the current
        // timestamp
        final QueryShardContext queryShardContext =
            indexService.newQueryShardContext(0, null, () -> 0L, null);

        for (Alias alias : request.aliases()) {
            if (Strings.hasLength(alias.filter())) {
                aliasValidator.validateAliasFilter(alias.name(), alias.filter(), queryShardContext, xContentRegistry);
            }
        }
        for (AliasMetaData aliasMetaData : templatesAliases.values()) {
            if (aliasMetaData.filter() != null) {
                aliasValidator.validateAliasFilter(aliasMetaData.alias(), aliasMetaData.filter().uncompressed(),
                    queryShardContext, xContentRegistry);
            }
        }

        // now, update the mappings with the actual source
        Map<String, MappingMetaData> mappingsMetaData = new HashMap<>();
        for (DocumentMapper mapper : Arrays.asList(mapperService.documentMapper(),
                                                   mapperService.documentMapper(MapperService.DEFAULT_MAPPING))) {
            if (mapper != null) {
                MappingMetaData mappingMd = new MappingMetaData(mapper);
                mappingsMetaData.put(mapper.type(), mappingMd);
            }
        }

        /**
         * 阶段八：构建IndexMetaData阶段
         *
         * 构建IndexMetaData.Builder indexMetaDataBuilder的indexMetaDataBuilder对象并绑定actualIndexSettings和routingNumShards。
         * indexMetaDataBuilder通过primaryTerm设置primaryTerm、putMapping设置mappingMd，putAlias设置aliasMetaData（template和request请求），putCustom设置customIndexMetaData，indexMetaDataBuilder.state设置state。
         * 创建indexMetaData对象，通过indexMetaData = indexMetaDataBuilder.build()实现
         */
        // 创建indexMetaDataBuilder对象，执行真正的创建
        final IndexMetaData.Builder indexMetaDataBuilder = IndexMetaData.builder(request.index())
            .settings(actualIndexSettings)
            .setRoutingNumShards(routingNumShards);

        //将主分片ID加载到索引元数据中
        for (int shardId = 0; shardId < tmpImd.getNumberOfShards(); shardId++) {
            indexMetaDataBuilder.primaryTerm(shardId, tmpImd.primaryTerm(shardId));
        }

        //将mapping信息也加载到索引元数据中
        for (MappingMetaData mappingMd : mappingsMetaData.values()) {
            indexMetaDataBuilder.putMapping(mappingMd);
        }

        //将别名信息加载到元数据中
        for (AliasMetaData aliasMetaData : templatesAliases.values()) {
            indexMetaDataBuilder.putAlias(aliasMetaData);
        }
        for (Alias alias : request.aliases()) {
            AliasMetaData aliasMetaData = AliasMetaData.builder(alias.name()).filter(alias.filter())
                .indexRouting(alias.indexRouting()).searchRouting(alias.searchRouting()).writeIndex(alias.writeIndex()).build();
            indexMetaDataBuilder.putAlias(aliasMetaData);
        }

        indexMetaDataBuilder.state(IndexMetaData.State.OPEN);
        /**
         * 阶段九：更新IndexMetaData和更新MetaData
         */
        final IndexMetaData indexMetaData;
        try {
            indexMetaData = indexMetaDataBuilder.build();
        } catch (Exception e) {
            removalExtraInfo = "failed to build index metadata";
            throw e;
        }

        indexService.getIndexEventListener().beforeIndexAddedToCluster(indexMetaData.getIndex(),
            indexMetaData.getSettings());

        // 创建新的MetaData,将所有元数据信息更新到集群元数据信息中
        MetaData newMetaData = MetaData.builder(currentState.metaData())
            .put(indexMetaData, false)
            .build();

        logger.info("[{}] creating index, cause [{}], templates {}, shards [{}]/[{}], mappings {}",
            request.index(), request.cause(), templateNames, indexMetaData.getNumberOfShards(),
            indexMetaData.getNumberOfReplicas(), mappings.keySet());

        ClusterBlocks.Builder blocks = ClusterBlocks.builder().blocks(currentState.blocks());
        if (!request.blocks().isEmpty()) {
            for (ClusterBlock block : request.blocks()) {
                blocks.addIndexBlock(request.index(), block);
            }
        }
        blocks.updateBlocks(indexMetaData);

        // 阻塞集群，更新matadata
        ClusterState updatedState = ClusterState.builder(currentState).blocks(blocks).metaData(newMetaData).build();

        RoutingTable.Builder routingTableBuilder = RoutingTable.builder(updatedState.routingTable())
            .addAsNew(updatedState.metaData().index(request.index()));
        /**阶段十：集群路由更新，分配给其他节点（为新分片分配节点）*/
        updatedState = allocationService.reroute(
            ClusterState.builder(updatedState).routingTable(routingTableBuilder.build()).build(),
            "index [" + request.index() + "] created");
        removalExtraInfo = "cleaning up after validating index on master";
        removalReason = IndexRemovalReason.NO_LONGER_ASSIGNED;
        return updatedState;
    } finally {
        if (createdIndex != null) {
            // Index was already partially created - need to clean up
            //确认后再给索引删了
            indicesService.removeIndex(createdIndex, removalReason, removalExtraInfo);
        }
    }
```
createIndex是创建IndexService，并存入<索引Id, IndexService>的Map

## IndexService
### createIndex
该方法的核心是创建IndexService，每个索引创建一个对应的IndexService，存入Map<String, IndexService> indices中。
```
public synchronized IndexService createIndex(
        final IndexMetaData indexMetaData, final List<IndexEventListener> builtInListeners) throws IOException {
    ...........................................
    List<IndexEventListener> finalListeners = new ArrayList<>(builtInListeners);
    //创建索引事件监听器
    final IndexEventListener onStoreClose = new IndexEventListener() {
        @Override
        public void onStoreCreated(ShardId shardId) {
            indicesRefCount.incRef();
        }
        @Override
        public void onStoreClosed(ShardId shardId) {
            try {
                indicesQueryCache.onClose(shardId);
            } finally {
                indicesRefCount.decRef();
            }
        }
    };
    finalListeners.add(onStoreClose);
    finalListeners.add(oldShardsStats);
    final IndexService indexService =
            createIndexService(
                    CREATE_INDEX,
                    indexMetaData,
                    indicesQueryCache,
                    indicesFieldDataCache,
                    finalListeners,
                    indexingMemoryController);
    try {
        indexService.getIndexEventListener().afterIndexCreated(indexService);
        indices = newMapBuilder(indices).put(index.getUUID(), indexService).immutableMap();
        success = true;
        return indexService;
    }
```

### createIndexService
createIndexService核心是初始化IndexMoudle，并调用newIndexService方法创建IndexService。
```
/**
 * This creates a new IndexService without registering it
 */
private synchronized IndexService createIndexService(IndexService.IndexCreationContext indexCreationContext,
                                                     IndexMetaData indexMetaData,
                                                     IndicesQueryCache indicesQueryCache,
                                                     IndicesFieldDataCache indicesFieldDataCache,
                                                     List<IndexEventListener> builtInListeners,
                                                     IndexingOperationListener... indexingOperationListeners) throws IOException {
    //创建IndexSettings，索引设置参数会被合并或覆盖
    final IndexSettings idxSettings = new IndexSettings(indexMetaData, settings, indexScopedSettings);
    //documents禁止自动生成ID（7.X以后版本）
    if (idxSettings.getIndexVersionCreated().onOrAfter(Version.V_7_0_0)
        && EngineConfig.INDEX_OPTIMIZE_AUTO_GENERATED_IDS.exists(idxSettings.getSettings())) {
        throw new IllegalArgumentException(
            "Setting [" + EngineConfig.INDEX_OPTIMIZE_AUTO_GENERATED_IDS.getKey() + "] was removed in version 7.0.0");
    }
    // we ignore private settings since they are not registered settings
    indexScopedSettings.validate(indexMetaData.getSettings(), true, true, true);

    final IndexModule indexModule = new IndexModule(idxSettings, analysisRegistry, getEngineFactory(idxSettings), directoryFactories);
    for (IndexingOperationListener operationListener : indexingOperationListeners) {
        //添加索引IndexingMemoryController监听器
        indexModule.addIndexOperationListener(operationListener);
    }
    pluginsService.onIndexModule(indexModule);
    for (IndexEventListener listener : builtInListeners) {
        //索引事件监听器，创建索引时有两个IndexEventListener
        indexModule.addIndexEventListener(listener);
    }
    return indexModule.newIndexService(...);
}
```

### IndexModule
为创建IndexService做准备，这里需要关注getDirectoryFactory方法，获取DirectoryFactory；是否需要QueryCache；是否需要IndexAnalyzers（官网描述）；
```
public IndexService newIndexService(..........)
    final IndexEventListener eventListener = freeze();
    Function<IndexService, CheckedFunction<DirectoryReader, DirectoryReader, IOException>> readerWrapperFactory =
        indexReaderWrapper.get() == null ? (shard) -> null : indexReaderWrapper.get();
    //索引创建前CompositeIndexEventListener处理
    eventListener.beforeIndexCreated(indexSettings.getIndex(), indexSettings.getSettings());
    //DirectoryFactory默认FsDirectoryFactory
    final IndexStorePlugin.DirectoryFactory directoryFactory = getDirectoryFactory(indexSettings, directoryFactories);
    QueryCache queryCache = null;
    //针对非结构化数据的处理：https://www.elastic.co/guide/en/elasticsearch/reference/7.5/analysis.html
    IndexAnalyzers indexAnalyzers = null;
    boolean success = false;
    try {
        // whether to use the query cache，是否使用query cache
        if (indexSettings.getValue(INDEX_QUERY_CACHE_ENABLED_SETTING)) {
            //Setting for enabling or disabling document/field level security. Defaults to true
            // 默认forceQueryCacheProvider是消费者，具体实现在Security#onIndexModule方法内
            BiFunction<IndexSettings, IndicesQueryCache, QueryCache> queryCacheProvider = forceQueryCacheProvider.get();
            if (queryCacheProvider == null) {
                queryCache = new IndexQueryCache(indexSettings, indicesQueryCache);
            } else {
                //消费者的入参是（settings, cache）Security类onIndexModule方法定义消费者
                queryCache = queryCacheProvider.apply(indexSettings, indicesQueryCache);
            }
        } else {
            queryCache = new DisabledQueryCache(indexSettings);
        }
         //索引元数据处于OPEN状态且IndexCreationContext是META_DATA_VERIFICATION类型,使用MapperService
        if (IndexService.needsMapperService(indexSettings, indexCreationContext)) {
            //需要MapperService，构建charFilterFactories tokenizerFactories tokenFilterFactories analyzerFactories normalizerFactories
            indexAnalyzers = analysisRegistry.build(indexSettings);
        }
        final IndexService indexService = new IndexService(indexSettings, indexCreationContext, environment, xContentRegistry,
            new SimilarityService(indexSettings, scriptService, similarities), shardStoreDeleter, indexAnalyzers,
            engineFactory, circuitBreakerService, bigArrays, threadPool, scriptService, clusterService, client, queryCache,
            directoryFactory, eventListener, readerWrapperFactory, mapperRegistry, indicesFieldDataCache, searchOperationListeners,
            indexOperationListeners, namedWriteableRegistry);
        success = true;
        return indexService;
    }
```
存储类型对应官网描述：fs simplefs niofs mmapfs hybridfs，默认FsDirectoryFactory。
```
//https://www.elastic.co/guide/en/elasticsearch/reference/7.5/index-modules-store.html
private static IndexStorePlugin.DirectoryFactory getDirectoryFactory(
        final IndexSettings indexSettings, final Map<String, IndexStorePlugin.DirectoryFactory> indexStoreFactories) {
    //通过indexSettings查找index.store.type
    final String storeType = indexSettings.getValue(INDEX_STORE_TYPE_SETTING);
    final Type type;
    //MMap FS 类型通过将文件映射到内存（mmap）将分片索引存储在文件系统上（映射到 Lucene MMapDirectory ）
    final Boolean allowMmap = NODE_STORE_ALLOW_MMAP.get(indexSettings.getNodeSettings());
    if (storeType.isEmpty() || Type.FS.getSettingsKey().equals(storeType)) {
        //type类型：hybridfs mmapfs  niofs simplefs fs
        type = defaultStoreType(allowMmap);
    } else {
        if (isBuiltinType(storeType)) {
            type = Type.fromSettingsKey(storeType);
        } else {
            type = null;
        }
    }
    if (allowMmap == false && (type == Type.MMAPFS || type == Type.HYBRIDFS)) {
        throw new IllegalArgumentException("store type [" + storeType + "] is not allowed because mmap is disabled");
    }
    final IndexStorePlugin.DirectoryFactory factory;
    if (storeType.isEmpty() || isBuiltinType(storeType)) {
        //默认FsDirectoryFactory
        factory = DEFAULT_DIRECTORY_FACTORY;
    } else {
        factory = indexStoreFactories.get(storeType);
        if (factory == null) {
            throw new IllegalArgumentException("Unknown store type [" + storeType + "]");
        }
    }
    return factory;
}
```
IndexService服务创建完成后，需要allocationService.reroute把Master节点的信息路由到其他Node，这里会在AllocateService模块介绍。
