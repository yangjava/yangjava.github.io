---
layout: post
categories: [JanusGraph]
description: none
keywords: JanusGraph
---
# JanusGraph源码分析

## 环境准备
下载源码：https://github.com/JanusGraph/janusgraph和http://janusgraph.org/
编译：
```
# 编译完整的
mvn -settings ~/opt/soft/apache-maven-3.5.0/conf/settings.xml -Dlicense.skip=true -DskipTests clean install
# 只编译core部分
mvn -pl janusgraph-core -am clean install -Dlicense.skip=true -DskipTests -P prod

-rf :janusgraph-test
mvn -pl janusgraph-test -am clean install -Dlicense.skip=true -DskipTests -P prod
```
我们在 janusgraph-test 下面编写一个例子 FirstTest：
```java
public class FirstTest {

    public static void main(String[] args) {

        /*
         * The example below will open a JanusGraph graph instance and load The Graph of the Gods dataset diagrammed above.
         * JanusGraphFactory provides a set of static open methods,
         * each of which takes a configuration as its argument and returns a graph instance.
         * This tutorial calls one of these open methods on a configuration
         * that uses the BerkeleyDB storage backend and the Elasticsearch index backend,
         * then loads The Graph of the Gods using the helper class GraphOfTheGodsFactory.
         * This section skips over the configuration details, but additional information about storage backends,
         * index backends, and their configuration are available in
         * Part III, “Storage Backends”, Part IV, “Index Backends”, and Chapter 13, Configuration Reference.
         */

        // Loading the Graph of the Gods Into JanusGraph
        JanusGraph graph = JanusGraphFactory
                .open("janusgraph-dist/src/assembly/cfilter/conf/janusgraph-berkeleyje-es.properties");

        GraphOfTheGodsFactory.load(graph);
        GraphTraversalSource g = graph.traversal();

        /*
         * The typical pattern for accessing data in a graph database is to first locate the entry point into the graph
         * using a graph index. That entry point is an element (or set of elements) 
         * — i.e. a vertex or edge. From the entry elements,
         * a Gremlin path description describes how to traverse to other elements in the graph via the explicit graph structure.
         * Given that there is a unique index on name property, the Saturn vertex can be retrieved.
         * The property map (i.e. the key/value pairs of Saturn) can then be examined.
         * As demonstrated, the Saturn vertex has a name of "saturn, " an age of 10000, and a type of "titan."
         * The grandchild of Saturn can be retrieved with a traversal that expresses:
         * "Who is Saturn’s grandchild?" (the inverse of "father" is "child"). The result is Hercules.
         */
        // Global Graph Indices
        Vertex saturn = g.V().has("name", "saturn").next();
        GraphTraversal<Vertex, Map<String, Object>> vertexMapGraphTraversal = g.V(saturn).valueMap();

        GraphTraversal<Vertex, Object> values = g.V(saturn).in("father").in("father").values("name");

        /*
         * The property place is also in a graph index. The property place is an edge property.
         * Therefore, JanusGraph can index edges in a graph index.
         * It is possible to query The Graph of the Gods for all events that have happened within 50 kilometers of Athens
          * (latitude:37.97 and long:23.72).
          * Then, given that information, which vertices were involved in those events.
         */
        System.out.println(g.E().has("place", geoWithin(Geoshape.circle(37.97, 23.72, 50))));
        System.out.println(g.E().has("place", geoWithin(Geoshape.circle(37.97, 23.72, 50)))
                .as("source").inV()
                .as("god2")
                .select("source").outV()
                .as("god1").select("god1", "god2")
                .by("name"));
    }

}
```
然后在"janusgraph-dist/src/assembly/cfilter/conf/janusgraph-berkeleyje-es.properties" 文件中，将注释掉的内容取消注释。

运行发现依赖挺麻烦。 首先运行报错了：
```
Exception in thread "main" java.lang.IllegalArgumentException: Could not find implementation class: org.janusgraph.diskstorage.berkeleyje.BerkeleyJEStoreManager
```
找到报错处的代码，我们发现 janusgraph-core 中通过反射创建一个类，但是这个类在 janusgraph-berkeleyje 中，而前者不依赖后者，所以找不到这个类，我们可以将后者加到前者的依赖， 但是我们发现后者依赖前者，如果加了依赖两个就相互依赖了，这是 Janus 官方设计的问题。我们只好在 FirstTest 所在的module中把两个依赖都加进来试试。 （注意，如果我们将所有的都打进一个包，这个问题就不存在了，但是在本地运行是不一样的，各自模块的编译输出文件在不同的地方。）在 janusgraph-test 中添加：
```xml
        <dependency>
            <groupId>org.janusgraph</groupId>
            <artifactId>janusgraph-berkeleyje</artifactId>
            <version>0.3.0-SNAPSHOT</version>
        </dependency>
```
发现 janusgraph-berkeleyje也依赖了 janusgraph-test,又相互依赖了，好麻烦。我们写写代码一定要注意这个问题。这里我的解决方法是直接把 代码放到 janusgraph-berkeleyje 中运行。
```
Exception in thread "main" java.lang.IllegalArgumentException: Could not find implementation class: org.janusgraph.diskstorage.es.ElasticSearchIndex
```
和上面一样，还依赖了 janusgraph-es,我只好吧代码复制到 janusgraph-es 的test代码块中运行（注意一点是test代码中），顺便在 janusgraph-es 中 添加上janusgraph-berkeleyje的依赖。 运行成功了，但是报了连接失败，是因为我本地没有启动es，我启动一下es：elasticsearch 然后在运行：
```
Exception in thread "main" org.janusgraph.core.SchemaViolationException: Adding this property for key [~T$SchemaName] and value [rtname] violates a uniqueness constraint [SystemIndex#~T$SchemaName]
```
然后我们可以在我们传入的配置文件找到：storage.directory=../db/berkeley ，直接删除这个目录，再重新运行，就成功了：
```
11:20:17,051  INFO GraphDatabaseConfiguration:1285 - Set default timestamp provider MICRO
11:20:17,296  INFO GraphDatabaseConfiguration:1492 - Generated unique-instance-id=c0a815a789637-dengzimings-MacBook-Pro-local1
11:20:17,547  INFO Backend:462 - Configuring index [search]
11:20:19,279  INFO Backend:177 - Initiated backend operations thread pool of size 8
11:20:19,461  INFO KCVSLog:753 - Loaded unidentified ReadMarker start time 2018-04-26T03:20:19.408Z into org.janusgraph.diskstorage.log.kcvs.KCVSLog$MessagePuller@73cd37c0
[GraphStep(edge,[]), HasStep([place.geoWithin(BUFFER (POINT (23.72 37.97), 0.44966))])]
[GraphStep(edge,[]), HasStep([place.geoWithin(BUFFER (POINT (23.72 37.97), 0.44966))])@[source], EdgeVertexStep(IN)@[god2], SelectOneStep(last,source), EdgeVertexStep(OUT)@[god1], SelectStep(last,[god1, god2],[value(name)])]
11:20:29,578  INFO ManagementLogger:192 - Received all acknowledgements for eviction [1]
```
然后我们可以去 ../db/berkeley 目录查看，多了一些文件，这些文件的作用我们后续再分析。 然后我们取es查看：curl -XGET 'localhost:9200/_cat/indices?v&pretty' ，发现多了两个index:
```
yellow open   janusgraph_edges    QT-E7AV6SMWr8Cu_ywKsXg   5   1          6            0     13.7kb         13.7kb
yellow open   janusgraph_vertices gE4TSXFATnSZUWYdAf46Xg   5   1          6            0     10.9kb         10.9kb
```
还可以具体查看内容。例如名字是titan的内容：curl -XGET 'localhost:9200/janusgraph_vertices/_search?q=name:titan&pretty'

到现在我们第一个案例就结束了。
```
g.E().has("place", geoWithin(Geoshape.circle(37.97, 23.72, 50)))
                .as("source").inV()
                .as("god2")
                .select("source").outV()
                .as("god1").select("god1", "god2")
                .by("name")
```
这种风格的代码实际上是groovy语言的代码，大家可以研究一下groovy语言。

注意事项： 上述第一次运行问题的原因是 janusgraph-core需要用到 janusgraph-berkeleyje的类， 但是janusgraph-berkeleyje是依赖 janusgraph-core的，所以两个相互依赖了。 janus的做法是在core中使用反射，所以编译通过了，打包到了一起就没问题了。但是本地运行没法成功。

## 开始调试
删除 db 文件夹，打上断点，开始debug，首先进入：JanusGraphFactory.open

JanusGraphFactory is used to open or instantiate a JanusGraph graph database. Opens a {@link JanusGraph} database configured according to the provided configuration.
```
public static JanusGraph open(ReadConfiguration configuration, String backupName) {
        final ModifiableConfiguration config = new ModifiableConfiguration(ROOT_NS, (WriteConfiguration) configuration, BasicConfiguration.Restriction.NONE);
        final String graphName = config.has(GRAPH_NAME) ? config.get(GRAPH_NAME) : backupName;
        final JanusGraphManager jgm = JanusGraphManagerUtility.getInstance();
        if (null != graphName) {
        Preconditions.checkState(jgm != null, JANUS_GRAPH_MANAGER_EXPECTED_STATE_MSG);
        return (JanusGraph) jgm.openGraph(graphName, gName -> new StandardJanusGraph(new GraphDatabaseConfiguration(configuration)));
        } else {
        if (jgm != null) {
        log.warn("...");
        }
        return new StandardJanusGraph(new GraphDatabaseConfiguration(configuration));
        }
        }
```

前面的部分先跳过，然后进入：
```java
1. return new StandardJanusGraph(new GraphDatabaseConfiguration(configuration));
    // 构造方法，分为静态代码和构造方法，这部分目前是跳过，但是后续是重点和核心。
    1. 父类：JanusGraphBlueprintsGraph
        static {
        TraversalStrategies graphStrategies = TraversalStrategies.GlobalCache.getStrategies(Graph.class).clone()
                .addStrategies(AdjacentVertexFilterOptimizerStrategy.instance(), JanusGraphLocalQueryOptimizerStrategy.instance(), JanusGraphStepStrategy.instance());

        //Register with cache
        TraversalStrategies.GlobalCache.registerStrategies(StandardJanusGraph.class, graphStrategies);
        TraversalStrategies.GlobalCache.registerStrategies(StandardJanusGraphTx.class, graphStrategies);
        }
    2. 新建配置，A graph database configuration is uniquely associated with a graph database and must not be used for multiple databases

    new GraphDatabaseConfiguration(configuration)
        1. storeManager 
        final KeyColumnValueStoreManager storeManager = Backend.getStorageManager(localBasicConfiguration);
        final StoreFeatures storeFeatures = storeManager.getFeatures();
        2. 检查参数，配置等

    3. 然后是构造方法
        1. 成员变量
        private final SchemaCache.StoreRetrieval typeCacheRetrieval = new SchemaCache.StoreRetrieval() {}
        2. backend
        this.backend = configuration.getBackend();
            1. Backend backend = new Backend(configuration);
                1. KeyColumnValueStoreManager manager = getStorageManager(configuration);
                2. indexes = getIndexes(configuration);

                3. //这里的 KCVS 是 keycolumnvaluestorageManager
                managementLogManager = getKCVSLogManager(MANAGEMENT_LOG);
                txLogManager = getKCVSLogManager(TRANSACTION_LOG);
                userLogManager = getLogManager(USER_LOG);

                4. scanner = new StandardScanner(storeManager);

            2. backend.initialize(configuration);
                1. store 新建
                KeyColumnValueStore idStore = storeManager.openDatabase(config.get(IDS_STORE_NAME));
                KeyColumnValueStore edgeStoreRaw = storeManagerLocking.openDatabase(EDGESTORE_NAME);
                KeyColumnValueStore indexStoreRaw = storeManagerLocking.openDatabase(INDEXSTORE_NAME);

                2. cacheEnabled
                edgeStore = new NoKCVSCache(edgeStoreRaw);
                indexStore = new NoKCVSCache(indexStoreRaw);
            3. storeFeatures = backend.getStoreFeatures();
        3. 初始化
        this.idAssigner = config.getIDAssigner(backend);
        this.idManager = idAssigner.getIDManager();
        this.serializer = config.getSerializer();
        StoreFeatures storeFeatures = backend.getStoreFeatures();
        this.indexSerializer = new IndexSerializer(configuration.getConfiguration(), this.serializer,
        this.backend.getIndexInformation(), storeFeatures.isDistributed() && storeFeatures.isKeyOrdered());
        this.edgeSerializer = new EdgeSerializer(this.serializer);
        this.vertexExistenceQuery = edgeSerializer.getQuery(BaseKey.VertexExists, Direction.OUT, new EdgeSerializer.TypedInterval[0]).setLimit(1);
        this.queryCache = new RelationQueryCache(this.edgeSerializer);
        this.schemaCache = configuration.getTypeCache(typeCacheRetrieval);
        this.times = configuration.getTimestampProvider();
```

然后是open完成后：GraphOfTheGodsFactory.load(graph);
```java
1. 得到management
JanusGraphManagement management = graph.openManagement();

    1. new ManagementSystem
        1. 启动 tx
        this.transaction = (StandardJanusGraphTx) graph.buildTransaction().disableBatchLoading().start();
            1.  graph.newTransaction(immutable);
                StandardJanusGraphTx tx = new StandardJanusGraphTx(this, configuration);
                tx.setBackendTransaction(openBackendTransaction(tx));
                openTransactions.add(tx);
2. 得到 PropertyKey
final PropertyKey name = management.makePropertyKey("name").dataType(String.class).make();
    1. return transaction.makePropertyKey(name);
        1. return new StandardPropertyKeyMaker(this, name, indexSerializer, attributeHandler);
            1. super(tx, name, indexSerializer, attributeHandler);
    2. public StandardPropertyKeyMaker dataType(Class<?> clazz)
    3. public PropertyKey make()
        1. TypeDefinitionMap definition = makeDefinition();        
        2. return tx.makePropertyKey(getName(), definition);
            1. return (PropertyKey) makeSchemaVertex(JanusGraphSchemaCategory.PROPERTYKEY, name, definition);
                1. ... 先跳过。

3. 新建 index
JanusGraphManagement.IndexBuilder nameIndexBuilder = management.buildIndex("name", Vertex.class).addKey(name);
    1.
```
调用：JanusGraphManagement management = graph.openManagement();然后：management.makeEdgeLabel("father").multiplicity(Multiplicity.MANY2ONE).make();

然后就是查询数据库：Vertex saturn = g.V().has("name", "saturn").next();

## 细节调试
这次我们多关注一点细节实现，包括几个部分：
```java
Backend backend = new Backend(configuration);
backend.~~~

this.idAssigner = config.getIDAssigner(backend);
this.idManager = idAssigner.getIDManager();

JanusGraphManagement management = graph.openManagement();
management.makePropertyKey("name").dataType(String.class).make();
management.buildIndex("name", Vertex.class).addKey(name);

Vertex tartarus = tx.addVertex(T.label, "location", "name", "tartarus");
jupiter.addEdge("father", saturn);
```

Backend
```java
public StandardJanusGraph(GraphDatabaseConfiguration configuration) 
{
    this.backend = configuration.getBackend();
    {
        Backend backend = new Backend(configuration);
        {
            this.configuration = configuration;
            KeyColumnValueStoreManager manager = getStorageManager(configuration);
            {
                反射生成一个 KeyColumnValueStoreManager 实现类
            }
            indexes = getIndexes(configuration);
            {
                IndexProvider provider = getImplementationClass(config.restrictTo(index), config.get(INDEX_BACKEND,index),
                    StandardIndexProvider.getAllProviderClasses());
                -- org.janusgraph.diskstorage.es.ElasticSearchIndex
                builder.put(index, provider);
                builder.build();
            }
            storeFeatures = storeManager.getFeatures();
            {
                ...
            }
            ...
        }

        backend.initialize(configuration);
        {
            KeyColumnValueStore idStore = storeManager.openDatabase(config.get(IDS_STORE_NAME));
            {
                openDatabase("janusgraph_ids", EMPTY)
                {
                    if (!stores.containsKey(name) || stores.get(name).isClosed()) {
                         OrderedKeyValueStoreAdapter store = wrapKeyValueStore(manager.openDatabase(name), keyLengths);
                         {
                             public BerkeleyJEKeyValueStore openDatabase(String name) throws BackendException 
                             {
                                 Database db = environment.openDatabase(null, name, dbConfig);
                                 BerkeleyJEKeyValueStore store = new BerkeleyJEKeyValueStore(name, db, this);
                                 stores.put(name, store);
                             }
                         }
                         stores.put(name, store);
                     }
                     return stores.get(name);
                }
            }

            KeyColumnValueStore edgeStoreRaw = storeManagerLocking.openDatabase(EDGESTORE_NAME);
            {
                同上：  
                openDatabase("edgestore", EMPTY)
            }
            KeyColumnValueStore indexStoreRaw = storeManagerLocking.openDatabase(INDEXSTORE_NAME);
            {
                同上：  
                openDatabase("graphindex", EMPTY)
            }

            txLogManager.openLog(SYSTEM_TX_LOG_NAME);
            managementLogManager.openLog(SYSTEM_MGMT_LOG_NAME);
            txLogStore = new NoKCVSCache(storeManager.openDatabase(SYSTEM_TX_LOG_NAME));

            KeyColumnValueStore systemConfigStore = storeManagerLocking.openDatabase(SYSTEM_PROPERTIES_STORE_NAME);
            {
                同上：  
                openDatabase("system_properties", EMPTY)
            }

        }
        storeFeatures = backend.getStoreFeatures();
    }

    this.idAssigner = config.getIDAssigner(backend);
    this.idManager = idAssigner.getIDManager();

}
```

management
```java
JanusGraphManagement management = graph.openManagement();
{
   new ManagementSystem(this,backend.getGlobalSystemConfig(),backend.getSystemMgmtLog(), managementLogger, schemaCache);
   //参数分别是 graph config Log managementLogger schemaCache
   {
       this.transaction = (StandardJanusGraphTx) graph.buildTransaction().disableBatchLoading().start();
       {
           graph.buildTransaction()
           {
               new StandardTransactionBuilder(getConfiguration(), this);
               {

               }
           }
           disableBatchLoading()
           {

           }
           start()
           {
               new ImmutableTxCfg
               graph.newTransaction(immutable);
               {
                    StandardJanusGraphTx tx = new StandardJanusGraphTx(this, configuration);
                    {
                        父类： JanusGraphBlueprintsTransaction
                        太过复杂，跳过
                    }
                    tx.setBackendTransaction(openBackendTransaction(tx));
                    {
                        openBackendTransaction(tx)
                        {
                            IndexSerializer.IndexInfoRetriever retriever = indexSerializer.getIndexInfoRetriever(tx);
                            return backend.beginTransaction(tx.getConfiguration(), retriever);
                            {
                                StoreTransaction tx = storeManagerLocking.beginTransaction(configuration);
                                CacheTransaction cacheTx = new CacheTransaction(tx, storeManagerLocking, bufferSize, maxWriteTime, configuration.hasEnabledBatchLoading());
                                final Map<String, IndexTransaction> indexTx = new HashMap<>(indexes.size());
                                for (Map.Entry<String, IndexProvider> entry : indexes.entrySet()) {
                                    indexTx.put(entry.getKey(), new IndexTransaction(entry.getValue(), indexKeyRetriever.get(entry.getKey()), configuration, maxWriteTime));
                                }
                                return new BackendTransaction(cacheTx, configuration, storeFeatures,
                                    edgeStore, indexStore, txLogStore,
                                    maxReadTime, indexTx, threadPool);
                            }
                        }
                    }
                    openTransactions.add(tx);
                    return tx;
               }
           }

       }
   }
}

final PropertyKey name = management.makePropertyKey("name").dataType(String.class).make();
{
    management.makePropertyKey("name")
    {
        transaction.makePropertyKey(name);
        {
            new StandardPropertyKeyMaker(this, name, indexSerializer, attributeHandler);
            {
                super
                {
                    StandardRelationTypeMaker
                }
            }
        }
    }
    dataType(String.class)
    {
        dataType = clazz;
    }
    make();
    {
        new TypeDefinitionMap();
        tx.makePropertyKey(getName(), definition);
        {
            (PropertyKey) makeSchemaVertex(JanusGraphSchemaCategory.PROPERTYKEY, name, definition);
            {
                schemaVertex = new PropertyKeyVertex(this, IDManager.getTemporaryVertexID(IDManager.VertexIDType.UserPropertyKey, temporaryIds.nextID()), ElementLifeCycle.New);
                {
                    //一层层嵌套

                }
            }
        }
    }
}

management.buildIndex("name", Vertex.class).addKey(name).unique().buildCompositeIndex();
{
    new IndexBuilder(indexName, ElementCategory.getByClazz(elementType));
    {

    }
    addKey(name)
    {
        keys.put(key, null);
    }
    unique()
    {
        unique = true;
    }
    buildCompositeIndex()
    {
        createCompositeIndex(indexName, elementCategory, unique, constraint, keyArr);
        {
            JanusGraphSchemaVertex indexVertex = transaction.makeSchemaVertex(JanusGraphSchemaCategory.GRAPHINDEX, indexName, def);
            {
                schemaVertex = new JanusGraphSchemaVertex(this, IDManager.getTemporaryVertexID(IDManager.VertexIDType.GenericSchemaType,temporaryIds.nextID()), ElementLifeCycle.New);
                {
                    //一层层嵌套

                }
            }
            addSchemaEdge(indexVertex, keys[i], TypeDefinitionCategory.INDEX_FIELD, paras);

            updateSchemaVertex(indexVertex);
            JanusGraphIndexWrapper index = new JanusGraphIndexWrapper(indexVertex.asIndexType());
            updateIndex(index, SchemaAction.REGISTER_INDEX);
            return index;
        }
    }

}
```

containsVertexLabel
mgmt.getVertexLabels().iterator() mgmt.containsVertexLabel(label) 这两个方法都可以得到 VertexLABEL

首先看 mgmt.getVertexLabels().iterator(), 这里面首先通过了 guava 的 abstractIterator 转到一个 ResultSetIterator
```java

public ResultSetIterator(Iterator<R> inner, int limit) {
    this.iter = inner;
    this.limit = limit;
    count = 0;
    this.current = null;
    this.next = nextInternal();
    {
        QueryProcessor$LimitAdajustingIterator.hasNext()
        {
            ....省去一步调用
            executor.execute(query, backendQuery, executionInfo, profiler);
            {
                iter = new SubqueryIterator(indexQuery.getQuery(0), indexSerializer, txHandle, indexCache, indexQuery.getLimit(), getConversionFunction(query.getResultType()),
                        retrievals.isEmpty() ? null: QueryUtil.processIntersectingRetrievals(retrievals, indexQuery.getLimit()));
                {
                    stream = indexSerializer.query(subQuery, tx).map(r -> {
                        currentIds.add(r);
                        return r;
                    });
                    {
                        final List<EntryList> rs = sq.execute(tx);
                        {
                            EntryList next =tx.indexQuery(ksq.updateLimit(getLimit()-total));
                            {
                                return exe.call();
                                {
                                    return cacheEnabled?indexStore.getSlice(query, storeTx):
                                        indexStore.getSliceNoCache(query, storeTx);
                                    {
                                        CassandraThriftKeyColumnValueStore.getNamesSlice(ImmutableList.of(key),query,txh);
                                    }
                                }
                            }
                        }

                    }
                }
            }
        }

    }
}
```
我们可以在比较关键的地方大断点，然后分析整个调用栈，进行进一步分析。哪里是关键点是需要一定经验判断的。

例如我们基于 hadoop spark 等框架的时候，我们写的代码就是关键的，打断点可以看到合适调用，怎么被调用。 我们关心怎么写数据，可以在和底层数据交互的地方打断点。总之我们关心谁就在哪里打断点。

记住：打断点的地方基本上是最终的调用点。

## 整体调试找关键
首先是存储类，我们使用本地文件存储，存储使用类是：com.sleepycat.je.Database 这个类具体功能是啥可以具体研究。我们发现它有 get delete put 等方法，我们可以打上断点。然后查看调用栈。

得到 普通 的调用信息：
```
"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
      at com.sleepycat.je.Database.put(Database.java:1574)
      at com.sleepycat.je.Database.put(Database.java:1627)
      at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEKeyValueStore.insert(BerkeleyJEKeyValueStore.java:195)
      at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEKeyValueStore.insert(BerkeleyJEKeyValueStore.java:184)
      at org.janusgraph.diskstorage.keycolumnvalue.keyvalue.OrderedKeyValueStoreAdapter.mutate(OrderedKeyValueStoreAdapter.java:99)
      at org.janusgraph.diskstorage.configuration.backend.KCVSConfiguration$2.call(KCVSConfiguration.java:154)
      at org.janusgraph.diskstorage.configuration.backend.KCVSConfiguration$2.call(KCVSConfiguration.java:149)
      at org.janusgraph.diskstorage.util.BackendOperation.execute(BackendOperation.java:147)
      at org.janusgraph.diskstorage.util.BackendOperation$1.call(BackendOperation.java:161)
      at org.janusgraph.diskstorage.util.BackendOperation.executeDirect(BackendOperation.java:68)
      at org.janusgraph.diskstorage.util.BackendOperation.execute(BackendOperation.java:54)
      at org.janusgraph.diskstorage.util.BackendOperation.execute(BackendOperation.java:158)
      at org.janusgraph.diskstorage.configuration.backend.KCVSConfiguration.set(KCVSConfiguration.java:149)
      at org.janusgraph.diskstorage.configuration.backend.KCVSConfiguration.set(KCVSConfiguration.java:126)
      at org.janusgraph.diskstorage.configuration.ModifiableConfiguration.set(ModifiableConfiguration.java:40)
      at org.janusgraph.diskstorage.configuration.ModifiableConfiguration.setAll(ModifiableConfiguration.java:47)
      at org.janusgraph.graphdb.configuration.GraphDatabaseConfiguration.<init>(GraphDatabaseConfiguration.java:1266)
      at org.janusgraph.core.JanusGraphFactory.open(JanusGraphFactory.java:160)
      at org.janusgraph.core.JanusGraphFactory.open(JanusGraphFactory.java:131)
      at org.janusgraph.core.JanusGraphFactory.open(JanusGraphFactory.java:78)
      at org.janusgraph.test.dengziming.FirstTest.main(FirstTest.java:37)
```

从下往上可以看出，顺序：
```
new GraphDatabaseConfiguration
ModifiableConfiguration.setAll(getGlobalSubset(localBasicConfiguration.getAll())); 
KCVSConfiguration.set(key,value,null,false);
BackendOperation.execute(new BackendOperation.Transactional<Boolean>() {@Override public Boolean call}
然后调用 上面new 的 BackendOperation.Transactional 的 call 方法
然后是 store.mutate
status = db.put(tx, key.as(ENTRY_FACTORY), value.as(ENTRY_FACTORY));
put(txn, key, data, Put.OVERWRITE, null);
result = cursor.putInternal(key, data, putType, options);
最终调用的是 cursor.putNotify 插入数据。
```

这个 put 会多次调用，config 会设置 "startup-time" 等属性，都是通过这个put方法实现。

第二次用到这个方法是 创建 VertexLabel 的时候会分配 id， 这时候我们可以看一下更详细的调用栈：
```
"JanusGraphID(0)(4)[0]@5358" prio=5 tid=0x24 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
      at com.sleepycat.je.dbi.CursorImpl.insertRecordInternal(CursorImpl.java:1364)
      at com.sleepycat.je.dbi.CursorImpl.insertOrUpdateRecord(CursorImpl.java:1221)
      at com.sleepycat.je.Cursor.putNoNotify(Cursor.java:2962)
      at com.sleepycat.je.Cursor.putNotify(Cursor.java:2800)
      at com.sleepycat.je.Cursor.putNoDups(Cursor.java:2647)
      at com.sleepycat.je.Cursor.putInternal(Cursor.java:2478)
      - locked <0x1536> (a com.sleepycat.je.Transaction)
      at com.sleepycat.je.Cursor.putInternal(Cursor.java:830)
      at com.sleepycat.je.Database.put(Database.java:1574)
      at com.sleepycat.je.Database.put(Database.java:1627)
      at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEKeyValueStore.insert(BerkeleyJEKeyValueStore.java:195)
      at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEKeyValueStore.insert(BerkeleyJEKeyValueStore.java:184)
      at org.janusgraph.diskstorage.keycolumnvalue.keyvalue.OrderedKeyValueStoreAdapter.mutate(OrderedKeyValueStoreAdapter.java:99)
      at org.janusgraph.diskstorage.idmanagement.ConsistentKeyIDAuthority.lambda$getIDBlock$1(ConsistentKeyIDAuthority.java:261)
      at org.janusgraph.diskstorage.idmanagement.ConsistentKeyIDAuthority$$Lambda$71.1795053717.call(Unknown Source:-1)
      at org.janusgraph.diskstorage.util.BackendOperation.execute(BackendOperation.java:147)
      at org.janusgraph.diskstorage.idmanagement.ConsistentKeyIDAuthority.getIDBlock(ConsistentKeyIDAuthority.java:260)
      - locked <0x14f8> (a org.janusgraph.diskstorage.idmanagement.ConsistentKeyIDAuthority)
      at org.janusgraph.graphdb.database.idassigner.StandardIDPool$IDBlockGetter.call(StandardIDPool.java:288)
      at org.janusgraph.graphdb.database.idassigner.StandardIDPool$IDBlockGetter.call(StandardIDPool.java:255)
      ...
      at java.lang.Thread.run(Thread.java:745)
```

上面的调用栈没有显示这么多，实际上我们也没必要关心 com.sleepycat.je.Database.put(Database.java:1627) 之后的东西， 因为这些东西都是 数据库的写 API，而生产环境我们会使用 hbase和cassandra ，所以每次只要 debug 到 KeyColumnValueStore 的 相应方法即可，再 debug 就是数据库的方法。

到这里我们明白，增删改查都是 通过 KeyColumnValueStore 类完成。接下来我们直接在 BerkeleyJEKeyValueStore 的 增删改查方法 打断点就行。

management.commit();
management 是用来操作 schema 的类，我们可以猜测 schema 也是以系统属性的方式存在数据库中。通过打断点发现，前面的操作都没有触发 BerkeleyJEKeyValueStore 的insert ，直到 commit， 先取出调用栈：
```
"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
      at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEKeyValueStore.insert(BerkeleyJEKeyValueStore.java:195)
      at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEKeyValueStore.insert(BerkeleyJEKeyValueStore.java:184)
      at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEStoreManager.mutateMany(BerkeleyJEStoreManager.java:208)
      at org.janusgraph.diskstorage.keycolumnvalue.keyvalue.OrderedKeyValueStoreManagerAdapter.mutateMany(OrderedKeyValueStoreManagerAdapter.java:125)
      at org.janusgraph.diskstorage.keycolumnvalue.cache.CacheTransaction$1.call(CacheTransaction.java:94)
      at org.janusgraph.diskstorage.keycolumnvalue.cache.CacheTransaction$1.call(CacheTransaction.java:91)
      at org.janusgraph.diskstorage.util.BackendOperation.executeDirect(BackendOperation.java:68)
      at org.janusgraph.diskstorage.util.BackendOperation.execute(BackendOperation.java:54)
      at org.janusgraph.diskstorage.keycolumnvalue.cache.CacheTransaction.persist(CacheTransaction.java:91)
      at org.janusgraph.diskstorage.keycolumnvalue.cache.CacheTransaction.flushInternal(CacheTransaction.java:139)
      at org.janusgraph.diskstorage.keycolumnvalue.cache.CacheTransaction.commit(CacheTransaction.java:196)
      at org.janusgraph.diskstorage.BackendTransaction.commitStorage(BackendTransaction.java:134)
      at org.janusgraph.graphdb.database.StandardJanusGraph.commit(StandardJanusGraph.java:733)
      at org.janusgraph.graphdb.transaction.StandardJanusGraphTx.commit(StandardJanusGraphTx.java:1372)
      - locked <0x113a> (a org.janusgraph.graphdb.transaction.StandardJanusGraphTx)
      at org.janusgraph.graphdb.database.management.ManagementSystem.commit(ManagementSystem.java:239)
      - locked <0x102b> (a org.janusgraph.graphdb.database.management.ManagementSystem)
      at org.janusgraph.example.GraphOfTheGodsFactory.load(GraphOfTheGodsFactory.java:111)
```

这里面好像还有锁，这个先不讨论。

主要的几个调用：
- StandardJanusGraphTx.commit()
- StandardJanusGraph.commit(addedRelations.getAll(), deletedRelations.values(), this); -- 这个 commit 的逻辑挺复杂，需要仔细查看。
- BackendTransaction.commitStorage();
- CacheTransaction.commit()
- OrderedKeyValueStoreManagerAdapter.mutateMany
- BerkeleyJEStoreManager.mutateMany(subMutations, tx);
- BerkeleyJEKeyValueStore.insert();

然后接下来就是一个个分析这几个类每一个的属性和方法。

首先看一下类的继承结构
```
SchemaInspector    
    StandardJanusGraphTx (org.janusgraph.graphdb.transaction)
    SchemaManager (org.janusgraph.core.schema)
        Transaction (org.janusgraph.core)
            JanusGraphTransaction (org.janusgraph.core)
                JanusGraphBlueprintsTransaction (org.janusgraph.graphdb.tinkerpop)
                    StandardJanusGraphTx (org.janusgraph.graphdb.transaction)
            JanusGraph (org.janusgraph.core)
                JanusGraphBlueprintsGraph (org.janusgraph.graphdb.tinkerpop)
                    StandardJanusGraph (org.janusgraph.graphdb.database)
        JanusGraphManagement (org.janusgraph.core.schema)
            ManagementSystem (org.janusgraph.graphdb.database.management)
```
SchemaInspector 接口定义了检查 schema 的一些方法， 例如：containsRelationType getRelationType containsPropertyKey getOrCreatePropertyKey getEdgeLabel getOrCreateVertexLabel 这些方法有四类，分别是是 RelationType 相关的，PropertyKey 相关，EdgeLabel 相关，VertexLabel 相关。这四个代表啥大家应该都清楚了。

SchemaManager 接口 在 SchemaInspector 的基础上添加了 6 个方法 ：makePropertyKey makeEdgeLabel makeVertexLabel addProperties addProperties addConnection 。 其实前三个返回的是 Maker，后面三个返回的就是 Label。这六个方法左右主要是给 schema 添加更多信息，例如添加 properties。

Transaction 继承自 SchemaManager 和 Graph ，定义了 addVertex 和 query 等操作。很奇怪为什么只有 addVertex 没有 addEdge 和 addProperty 的操作。

JanusGraphManagement 继承自 SchemaManager 和 JanusGraphConfiguration ，定义了 buildEdgeIndex buildPropertyIndex commit 等操作 大部分都和 index 相关，例如构建查询更新。还有 getRelationTypes getVertexLabels 两个方法。

ManagementSystem 继承自 JanusGraphManagement ，通过代理 StandardJanusGraphTx ，实现了 getGraphIndex commit 等操作。

JanusGraphTransaction 继承自 Transaction ，定义了 addVertex getVertex commit rollback 等，和 Transaction 不同的是他的这些方法操作的都是 id，而后者操作的是 用户传入的 String

JanusGraphBlueprintsTransaction 继承自 JanusGraphTransaction ，目前看到的就是简单封装一下抽象方法，同时实现了 addVertex 方法。

StandardJanusGraphTx 继承自 JanusGraphBlueprintsTransaction ，实现了抽象的方法。

JanusGraph 继承自 Transaction， 定义了 buildTransaction openManagement close 等方法。

JanusGraphBlueprintsGraph 继承自 JanusGraph ，通过 ThreadLocal 实现线程隔离。 StandardJanusGraph 继承自 JanusGraphBlueprintsGraph 就是我们使用的 Graph 。

所以了解janus比较重要的是 StandardJanusGraphTx ，了解多线程的 JanusGraphBlueprintsGraph。

从继承结构大概可以看出所有的操作分为数据操作和 schema 操作，而分别由 JanusGraph 和 JanusGraphManagement 完成，实际上都是代理或者适配装饰了 StandardJanusGraphTx。StandardJanusGraphTx 内容很多。

## StandardJanusGraph
上面我们已经看出了实际上最重要的就是 StandardJanusGraphTx 的实现逻辑，我们就以他为入口，而不是 main 方法。它的构造方法里面需要用到 StandardJanusGraph ，我们先大概了解一下 。

我们先看一下它的属性：
```
log
config
backend
idManager
idAssigner
times
indexSerializer
edgeSerializer
serializer
vertexExistenceQuery
queryCache
schemaCache
managementLogger
shutdownHook
isOpen
txCounter
openTransactions
name
typeCacheRetrieval
SCHEMA_FILTER
NO_SCHEMA_FILTER
NO_FILTER
```
GraphDatabaseConfiguration config 是图的配置，由于配置也是保存在数据库，所以也是需要访问数据库的。

Backend backend 是在 config.getBackend 中初始化的，Backend 的构造方法很复杂，主要创建出了 StoreManager indexes txLogManager 等管理存储很重要的属性。

idManager 和 idAssigner 都是和 id 相关的。 所属类为 IDManager ，VertexIDAssigner，有比较复杂的id分配算法。

IndexSerializer 和 EdgeSerializer 、Serializer 用于序列化，Serializer 在 config 中初始化，其他两个都是基于 Serializer 的封装。

vertexExistenceQuery:SliceQuery queryCache:RelationQueryCache schemaCache:SchemaCache 都是 cache 相关。

managementLogger 是 用来记录操作日志的。

typeCacheRetrieval ，看到 Retrieval 就知道是获取某些属性用的，他通过 QueryUtil.getVertices(consistentTx, BaseKey.SchemaName, typeName) 获得 JanusGraphVertex。

然后再看方法
除了 getset 以外，主要是：
```
isOpen
isClosed
close
closeInternal
prepareCommit
commit
openManagement
newTransaction
buildTransaction
newThreadBoundTransaction
newTransaction
openBackendTransaction
closeTransaction
getVertexIDs
edgeQuery
edgeMultiQuery
assignID
assignID
acquireLock
acquireLock
getTTL
getTTL
```
和 transaction 有关的打开关闭提交等，查询边和顶点，分配id，获得锁。这里的 edgeQuery 并不是查询边，而是查询 edgestore 这个表格，这个表格存放了所有的数据。 细心分析发现，这些方法主要都是进行查询操作，得到查询结果 List，并没有进行数据增删改查的操作 API。

ManagementSystem
StandardJanusGraph 用来操作数据，而 ManagementSystem 主要是管理 schema。

```
LOGGER
CURRENT_INSTANCE_SUFFIX
graph
sysLog
managementLogger
transactionalConfig
modifyConfig
userConfig
schemaCache
transaction
updatedTypes
evictGraphFromCache
updatedTypeTriggers
txStartTime
graphShutdownRequired
isOpen
configVerifier
```
graph 和 managementLogger 就是上面的 StandardJanusGraph 和 managementLogger。sysLog 也是和日志有关。

TransactionalConfiguration 是事务的配置，实际上他应该是记录了变化，能够判断是否有改变，从而进行 commit 和 rollback

SchemaCache 就是 StandardJanusGraph 的 SchemaCache。

transaction 是 StandardJanusGraphTx。

updatedTypes 应该也是记录更新

其他的暂时还不太懂。
```
IndexBuilder
GraphCacheEvictionCompleteTrigger
EmptyIndexJobFuture
UpdateStatusTrigger
IndexJobStatus
IndexIdentifier
ManagementSystem

getOpenInstancesInternal
getOpenInstances
forceCloseInstance
ensureOpen
commit
rollback
isOpen
close
getWrappedTx
addSchemaEdge
getSchemaElement
buildEdgeIndex
buildEdgeIndex
buildPropertyIndex
buildPropertyIndex
buildRelationTypeIndex
composeRelationTypeIndexName
containsRelationIndex
getRelationIndex
getRelationIndexes
getGraphIndexDirect
containsGraphIndex
getGraphIndex
getGraphIndexes
awaitGraphIndexStatus
awaitRelationIndexStatus
checkIndexName
createMixedIndex
addIndexKey
createCompositeIndex
buildIndex
updateIndex
evictGraphFromCache
setUpdateTrigger
setStatus
setStatusVertex
setStatusEdges
getIndexJobStatus
changeName
updateConnectionEdgeConstraints
getSchemaVertex
updateSchemaVertex
getConsistency
setConsistency
getTTL
setTTL
setTypeModifier
containsRelationType
getRelationType
containsPropertyKey
getPropertyKey
containsEdgeLabel
getOrCreateEdgeLabel
getOrCreatePropertyKey
getEdgeLabel
makePropertyKey
makeEdgeLabel
getRelationTypes
containsVertexLabel
getVertexLabel
getOrCreateVertexLabel
makeVertexLabel
addProperties
addProperties
addConnection
getVertexLabels
get
set
```
强制关闭、操作事务、添加顶点边Label属性索引。

索引都是 buildRelationTypeIndex 方法，说明 RelationType(PropertyKey 和 EdgeLabel)才有索引，分别是 graphIndex 和 vertexIncdicentIndex ，VertexLabel 没有索引。 而 getVertexLabels 等带s的方法 是 调用 QueryUtil.getVertices ，说明得到所有的需要查询数据库。

很多方法都是直接调用 StandardJanusGraphTx 的 对应方法。但是 build Index 并没有使用到 StandardJanusGraphTx。说明 index 并不是马上就插入数据库？或者因为 Index 建完以后还要等待？？

StandardJanusGraphTx
上面大致了解了 StandardJanusGraph 和 ManagementSystem ，StandardJanusGraphTx 内部才是最重要的，

## Index
索引肯定是数据库的重点，我们到目前没有分析过和所以有关的内容。IndexTransaction 是我们遇到的可能和索引相关的内容了，就从 他开始。 IndexTransaction 中有个 BaseTransaction 的对象用来实现事务，通过 IndexProvider 来产生。我们以 ElasticSearchIndex 为例，可以看看他的方法。

例如 register 方法会创建索引，还有 restore 等操作事务的方法。在 ManagementSystem 的 updateIndex 方法中，定义了各种操作 index 的方法。

Index 类继承了 JanusGraphSchemaElement，主要有两类实现类 JanusGraphIndex 和 RelationTypeIndex 。

JanusGraphIndex 的实现类是 JanusGraphIndexWrapper 。可以通过 JanusGraphManagement#buildIndex(String, Class) 新建 。

RelationTypeIndex 的实现类是 RelationTypeIndexWrapper，可以通过 JanusGraphManagement#buildEdgeIndex(org.janusgraph.core.EdgeLabel, String, org.apache.tinkerpop.gremlin.structure.Direction, org.apache.tinkerpop.gremlin.process.traversal.Order, org.janusgraph.core.PropertyKey...) 和 JanusGraphManagement#buildPropertyIndex(org.janusgraph.core.PropertyKey, String, org.apache.tinkerpop.gremlin.process.traversal.Order, org.janusgraph.core.PropertyKey...) 两个方法建 RelationTypeIndex。

IndexType 定义所有的 JanusGraphIndex，实现包括 CompositeIndexType 和 MixedIndexType。

IndexType IndexProvider 和 Index 的不同在于，Index 和他的实现类 JanusGraphIndex RelationTypeIndexWrapper 都是继承自 JanusGraphSchemaElement ，和 Vertex 一样，代表的是 janus 中的一个顶点。 IndexType 代表了所以类型 ，IndexProvider 则代表的是和索引相关的操作方法 例如 ElasticSearchIndex SolrIndex LuceneIndex。

StandardScanner
在 Backend 构造方法最后有一句 new StandardScanner。我们看看这个是干啥用的，主要调用地方是 buildStoreIndexScanJob 这个方法，我们发现这个新建了一个 Job。 buildEdgeScanJob 主要就是在 ManagementSystem 的 updateIndex 方法使用，根据方法名可以看出，这是在遍历数据库的job。

StandardScanner 的重点很明显就是它的内部类 Builder。Builder 内部有一个 ScanJob 的变量，实际上 Builder 就是有个 execute 方法，能够执行 ScanJob ，例如 IndexUpdateJob 和 IndexRepairJob。



















