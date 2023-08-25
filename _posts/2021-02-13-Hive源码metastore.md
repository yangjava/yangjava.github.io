---
layout: post
categories: [Hive]
description: none
keywords: Hive
---
# Hive源码metastore


## 整体架构
hive metastore的整体架构

组成结构：

如图我们可以看到，hive metastore的组成结构分为 客户端 服务端 ，那么下来我们逐一进行分析：

## 
我们从入口HIVE开始看，可以找到MetaStoreClient客户端的创建：
```
 1   private IMetaStoreClient createMetaStoreClient() throws MetaException {
 2 
 3     HiveMetaHookLoader hookLoader = new HiveMetaHookLoader() {
 4         @Override
 5         public HiveMetaHook getHook(
 6           org.apache.hadoop.hive.metastore.api.Table tbl)
 7           throws MetaException {
 8 
 9           try {
10             if (tbl == null) {
11               return null;
12             }
13             HiveStorageHandler storageHandler =
14               HiveUtils.getStorageHandler(conf,
15                 tbl.getParameters().get(META_TABLE_STORAGE));
16             if (storageHandler == null) {
17               return null;
18             }
19             return storageHandler.getMetaHook();
20           } catch (HiveException ex) {
21             LOG.error(StringUtils.stringifyException(ex));
22             throw new MetaException(
23               "Failed to load storage handler:  " + ex.getMessage());
24           }
25         }
26       };
27     return RetryingMetaStoreClient.getProxy(conf, hookLoader, metaCallTimeMap,
28         SessionHiveMetaStoreClient.class.getName());
29   }
```
我们可以看到，创建MetaStoreClient中，创建了HiveMetaHook,这个Hook的作用在于，每次对meta进行操作的时候，比如createTable的时候，如果建表的存储方式不是文件，比如集成hbase，HiveMetaStoreClient会调用hook的接口方法preCreateTable，进行建表前的准备，用来判断外部表与内部表，如果中途有失败的话，依旧调用hook中的rollbackCreateTable进行回滚。

```
 1   public void createTable(Table tbl, EnvironmentContext envContext) throws AlreadyExistsException,
 2       InvalidObjectException, MetaException, NoSuchObjectException, TException {
 3     HiveMetaHook hook = getHook(tbl);
 4     if (hook != null) {
 5       hook.preCreateTable(tbl);
 6     }
 7     boolean success = false;
 8     try {
 9       // Subclasses can override this step (for example, for temporary tables)
10       create_table_with_environment_context(tbl, envContext);
11       if (hook != null) {
12         hook.commitCreateTable(tbl);
13       }
14       success = true;
15     } finally {
16       if (!success && (hook != null)) {
17         hook.rollbackCreateTable(tbl);
18       }
19     }
20   }
```
在hbase表不存在的情况下，不能create external table ,会报doesn't exist while the table is declared as an external table,那么需直接创建create table 创建一个指向hbase的hive表。

建表语句如下：
```
CREATE TABLE hbase_table_1(key int, value string) /
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'  
WITH SERDEPROPERTIES ("hbase.columns.mapping" = ":key,cf1:val") 
TBLPROPERTIES ("hbase.table.name" = "tableName", "hbase.mapred.output.outputtable" = "tableName");
```
代码请查看HBaseStorageHandler的preCreateTable方法，这里就不贴出来啦。

随之回归Hive类，Hive类可以说是整个元数据DDL操作的最顶层抽象。HiveMetaStoreClient实现了IMetaStoreClient的接口，在创建HiveMetaStoreClient时，会创建于server段HiveMetaStore的链接，并且会通过检查hive.metastore.local是否为true,来决定是在本地创建服务端，这里为在本地:
```java
1   public HiveMetaStoreClient(HiveConf conf, HiveMetaHookLoader hookLoader)
 2     throws MetaException {
 3 
 4     this.hookLoader = hookLoader;
 5     if (conf == null) {
 6       conf = new HiveConf(HiveMetaStoreClient.class);
 7     }
 8     this.conf = conf;
 9     filterHook = loadFilterHooks();
10 
11     String msUri = conf.getVar(HiveConf.ConfVars.METASTOREURIS);
12     localMetaStore = HiveConfUtil.isEmbeddedMetaStore(msUri);
13     if (localMetaStore) {
14       // instantiate the metastore server handler directly instead of connecting
15       // through the network
16       client = HiveMetaStore.newRetryingHMSHandler("hive client", conf, true);
17       isConnected = true;
18       snapshotActiveConf();
19       return;
20     }
```
随后，创建server端的HiveMetaStore.HMSHandler，HMSHandler继承自IHMSHandler，而IHMSHandler又继承自ThriftHiveMetastore.Iface，在HMSHandler中实现了所有操作的对外方法：
```
public class ThriftHiveMetastore {

  /**
   * This interface is live.
   */
  public interface Iface extends com.facebook.fb303.FacebookService.Iface {

    public String getMetaConf(String key) throws MetaException, org.apache.thrift.TException;

    public void setMetaConf(String key, String value) throws MetaException, org.apache.thrift.TException;

    public void create_database(Database database) throws AlreadyExistsException, InvalidObjectException, MetaException, org.apache.thrift.TException;

    public Database get_database(String name) throws NoSuchObjectException, MetaException, org.apache.thrift.TException;

    public void drop_database(String name, boolean deleteData, boolean cascade) throws NoSuchObjectException, InvalidOperationException, MetaException, org.apache.thrift.TException;

    public List<String> get_databases(String pattern) throws MetaException, org.apache.thrift.TException;

    public List<String> get_all_databases() throws MetaException, org.apache.thrift.TException;

    public void alter_database(String dbname, Database db) throws MetaException, NoSuchObjectException, org.apache.thrift.TException;

    public Type get_type(String name) throws MetaException, NoSuchObjectException, org.apache.thrift.TException;

    public boolean create_type(Type type) throws AlreadyExistsException, InvalidObjectException, MetaException, org.apache.thrift.TException;

    public boolean drop_type(String type) throws MetaException, NoSuchObjectException, org.apache.thrift.TException;
    
    ......
```

在创建HiveMetaStore的init方法中，同时创建了三种Listener---MetaStorePreEventListener，MetaStoreEventListener，MetaStoreEndFunctionListener用于对每一步事件的监听。
```
1       initListeners = MetaStoreUtils.getMetaStoreListeners(
 2           MetaStoreInitListener.class, hiveConf,
 3           hiveConf.getVar(HiveConf.ConfVars.METASTORE_INIT_HOOKS));
 4       for (MetaStoreInitListener singleInitListener: initListeners) {
 5           MetaStoreInitContext context = new MetaStoreInitContext();
 6           singleInitListener.onInit(context);
 7       }
 8 
 9       String alterHandlerName = hiveConf.get("hive.metastore.alter.impl",
10           HiveAlterHandler.class.getName());
11       alterHandler = (AlterHandler) ReflectionUtils.newInstance(MetaStoreUtils.getClass(
12           alterHandlerName), hiveConf);
13       wh = new Warehouse(hiveConf);
14 
15       synchronized (HMSHandler.class) {
16         if (currentUrl == null || !currentUrl.equals(MetaStoreInit.getConnectionURL(hiveConf))) {
17           createDefaultDB();
18           createDefaultRoles();
19           addAdminUsers();
20           currentUrl = MetaStoreInit.getConnectionURL(hiveConf);
21         }
22       }
23 
24       if (hiveConf.getBoolean("hive.metastore.metrics.enabled", false)) {
25         try {
26           Metrics.init();
27         } catch (Exception e) {
28           // log exception, but ignore inability to start
29           LOG.error("error in Metrics init: " + e.getClass().getName() + " "
30               + e.getMessage(), e);
31         }
32       }
33 
34       preListeners = MetaStoreUtils.getMetaStoreListeners(MetaStorePreEventListener.class,
35           hiveConf,
36           hiveConf.getVar(HiveConf.ConfVars.METASTORE_PRE_EVENT_LISTENERS));
37       listeners = MetaStoreUtils.getMetaStoreListeners(MetaStoreEventListener.class, hiveConf,
38           hiveConf.getVar(HiveConf.ConfVars.METASTORE_EVENT_LISTENERS));
39       listeners.add(new SessionPropertiesListener(hiveConf));
40       endFunctionListeners = MetaStoreUtils.getMetaStoreListeners(
41           MetaStoreEndFunctionListener.class, hiveConf,
42           hiveConf.getVar(HiveConf.ConfVars.METASTORE_END_FUNCTION_LISTENERS));
```
同时创建了AlterHandler，它是HiveAlterHandler的接口，是将修改表和修改partition的操作抽离了出来单独实现（修改表很复杂的。。）。
```
 1 public interface AlterHandler extends Configurable {
 2 
 3   /**
 4    * handles alter table
 5    *
 6    * @param msdb
 7    *          object to get metadata
 8    * @param wh
 9    *          TODO
10    * @param dbname
11    *          database of the table being altered
12    * @param name
13    *          original name of the table being altered. same as
14    *          <i>newTable.tableName</i> if alter op is not a rename.
15    * @param newTable
16    *          new table object
17    * @throws InvalidOperationException
18    *           thrown if the newTable object is invalid
19    * @throws MetaException
20    *           thrown if there is any other error
21    */
22   public abstract void alterTable(RawStore msdb, Warehouse wh, String dbname,
23       String name, Table newTable) throws InvalidOperationException,
24       MetaException;
25 
26   /**
27    * handles alter table, the changes could be cascaded to partitions if applicable
28    *
29    * @param msdb
30    *          object to get metadata
31    * @param wh
32    *          Hive Warehouse where table data is stored
33    * @param dbname
34    *          database of the table being altered
35    * @param name
36    *          original name of the table being altered. same as
37    *          <i>newTable.tableName</i> if alter op is not a rename.
38    * @param newTable
39    *          new table object
40    * @param cascade
41    *          if the changes will be cascaded to its partitions if applicable
42    * @throws InvalidOperationException
43    *           thrown if the newTable object is invalid
44    * @throws MetaException
45    *           thrown if there is any other error
46    */
47   public abstract void alterTable(RawStore msdb, Warehouse wh, String dbname,
48       String name, Table newTable, boolean cascade) throws InvalidOperationException,
49       MetaException;
```
最重要的是RawStore的创建。RawStore不光是定义了一套最终的物理操作，使用JDO将一个对象当作表进行存储。ObjectStore中的transaction机制也是通过JDO提供的transaction实现的。当commit失败时，将rollback所有操作。
```java
 1   @Override
 2   public void createDatabase(Database db) throws InvalidObjectException, MetaException {
 3     boolean commited = false;
 4     MDatabase mdb = new MDatabase();
 5     mdb.setName(db.getName().toLowerCase());
 6     mdb.setLocationUri(db.getLocationUri());
 7     mdb.setDescription(db.getDescription());
 8     mdb.setParameters(db.getParameters());
 9     mdb.setOwnerName(db.getOwnerName());
10     PrincipalType ownerType = db.getOwnerType();
11     mdb.setOwnerType((null == ownerType ? PrincipalType.USER.name() : ownerType.name()));
12     try {
13       openTransaction();
14       pm.makePersistent(mdb);
15       commited = commitTransaction();
16     } finally {
17       if (!commited) {
18         rollbackTransaction();
19       }
20     }
21   }
22 
23   @SuppressWarnings("nls")
24   private MDatabase getMDatabase(String name) throws NoSuchObjectException {
25     MDatabase mdb = null;
26     boolean commited = false;
27     try {
28       openTransaction();
29       name = HiveStringUtils.normalizeIdentifier(name);
30       Query query = pm.newQuery(MDatabase.class, "name == dbname");
31       query.declareParameters("java.lang.String dbname");
32       query.setUnique(true);
33       mdb = (MDatabase) query.execute(name);
34       pm.retrieve(mdb);
35       commited = commitTransaction();
36     } finally {
37       if (!commited) {
38         rollbackTransaction();
39       }
40     }
41     if (mdb == null) {
42       throw new NoSuchObjectException("There is no database named " + name);
43     }
44     return mdb;
45   }
```

























