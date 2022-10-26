---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---

## 持久化

### 持久化总体框架

持久化的类主要在包org.apache.zookeeper.server.persistence下，此次也主要是对其下的类进行分析，其包下总体的类结构如下。

· TxnLog，接口类型，读取事务性日志的接口。

· FileTxnLog，实现TxnLog接口，添加了访问该事务性日志的API。

· Snapshot，接口类型，持久层快照接口。

· FileSnap，实现Snapshot接口，负责存储、序列化、反序列化、访问快照。

· FileTxnSnapLog，封装了TxnLog和SnapShot。

· Util，工具类，提供持久化所需的API。

### TxnLog源码分析

TxnLog是接口，规定了对日志的响应操作。

```
public interface TxnLog {
    
    /**
     * roll the current
     * log being appended to
     * @throws IOException 
     */
    // 回滚日志
    void rollLog() throws IOException;
    /**
     * Append a request to the transaction log
     * @param hdr the transaction header
     * @param r the transaction itself
     * returns true iff something appended, otw false 
     * @throws IOException
     */
    // 添加一个请求至事务性日志
    boolean append(TxnHeader hdr, Record r) throws IOException;

    /**
     * Start reading the transaction logs
     * from a given zxid
     * @param zxid
     * @return returns an iterator to read the 
     * next transaction in the logs.
     * @throws IOException
     */
    // 读取事务性日志
    TxnIterator read(long zxid) throws IOException;
    
    /**
     * the last zxid of the logged transactions.
     * @return the last zxid of the logged transactions.
     * @throws IOException
     */
    // 事务性操作的最新zxid
    long getLastLoggedZxid() throws IOException;
    
    /**
     * truncate the log to get in sync with the 
     * leader.
     * @param zxid the zxid to truncate at.
     * @throws IOException 
     */
    // 清空日志，与Leader保持同步
    boolean truncate(long zxid) throws IOException;
    
    /**
     * the dbid for this transaction log. 
     * @return the dbid for this transaction log.
     * @throws IOException
     */
    // 获取数据库的id
    long getDbId() throws IOException;
    
    /**
     * commmit the trasaction and make sure
     * they are persisted
     * @throws IOException
     */
    // 提交事务并进行确认
    void commit() throws IOException;
   
    /** 
     * close the transactions logs
     */
    // 关闭事务性日志
    void close() throws IOException;
    /**
     * an iterating interface for reading 
     * transaction logs. 
     */
    // 读取事务日志的迭代器接口
    public interface TxnIterator {
        /**
         * return the transaction header.
         * @return return the transaction header.
         */
        // 获取事务头部
        TxnHeader getHeader();
        
        /**
         * return the transaction record.
         * @return return the transaction record.
         */
        // 获取事务
        Record getTxn();
     
        /**
         * go to the next transaction record.
         * @throws IOException
         */
        // 下个事务
        boolean next() throws IOException;
        
        /**
         * close files and release the 
         * resources
         * @throws IOException
         */
        // 关闭文件释放资源
        void close() throws IOException;
    }
}
```

其中，TxnLog除了提供读写事务日志的API外，还提供了一个用于读取日志的迭代器接口TxnIterator。

对于LogFile而言，其格式可分为如下三部分

**LogFile:**

FileHeader TxnList ZeroPad

```
public class FileTxnLog implements TxnLog {
    private static final Logger LOG;
    
    // 预分配大小 64M
    static long preAllocSize =  65536 * 1024;
    
    // 魔术数字，默认为1514884167
    public final static int TXNLOG_MAGIC =
        ByteBuffer.wrap("ZKLG".getBytes()).getInt();

    // 版本号
    public final static int VERSION = 2;

    /** Maximum time we allow for elapsed fsync before WARNing */
    // 进行同步时，发出warn之前所能等待的最长时间
    private final static long fsyncWarningThresholdMS;

    // 静态属性，确定Logger、预分配空间大小和最长时间
    static {
        LOG = LoggerFactory.getLogger(FileTxnLog.class);

        String size = System.getProperty("zookeeper.preAllocSize");
        if (size != null) {
            try {
                preAllocSize = Long.parseLong(size) * 1024;
            } catch (NumberFormatException e) {
                LOG.warn(size + " is not a valid value for preAllocSize");
            }
        }
        fsyncWarningThresholdMS = Long.getLong("fsync.warningthresholdms", 1000);
    }
    
    // 最大(新)的zxid
    long lastZxidSeen;
    // 存储数据相关的流
    volatile BufferedOutputStream logStream = null;
    volatile OutputArchive oa;
    volatile FileOutputStream fos = null;

    // log目录文件
    File logDir;
    
    // 是否强制同步
    private final boolean forceSync = !System.getProperty("zookeeper.forceSync", "yes").equals("no");;
    
    // 数据库id
    long dbId;
    
    // 流列表
    private LinkedList<FileOutputStream> streamsToFlush =
        new LinkedList<FileOutputStream>();
    
    // 当前大小
    long currentSize;
    // 写日志文件
    File logFileWrite = null;
}
```

### SnapShot源码分析

SnapShot是FileTxnLog的父类，接口类型，其方法如下

```
public interface SnapShot {
    
    /**
     * deserialize a data tree from the last valid snapshot and 
     * return the last zxid that was deserialized
     * @param dt the datatree to be deserialized into
     * @param sessions the sessions to be deserialized into
     * @return the last zxid that was deserialized from the snapshot
     * @throws IOException
     */
    // 反序列化
    long deserialize(DataTree dt, Map<Long, Integer> sessions) 
        throws IOException;
    
    /**
     * persist the datatree and the sessions into a persistence storage
     * @param dt the datatree to be serialized
     * @param sessions 
     * @throws IOException
     */
    // 序列化
    void serialize(DataTree dt, Map<Long, Integer> sessions, 
            File name) 
        throws IOException;
    
    /**
     * find the most recent snapshot file
     * @return the most recent snapshot file
     * @throws IOException
     */
    // 查找最新的snapshot文件
    File findMostRecentSnapshot() throws IOException;
    
    /**
     * free resources from this snapshot immediately
     * @throws IOException
     */
    // 释放资源
    void close() throws IOException;
} 
```

说明：可以看到SnapShot只定义了四个方法，反序列化、序列化、查找最新的snapshot文件、释放资源。

FileSnap实现了SnapShot接口，主要用作存储、序列化、反序列化、访问相应snapshot文件。

```
public class FileSnap implements SnapShot {
    // snapshot目录文件
    File snapDir;
    // 是否已经关闭标识
    private volatile boolean close = false;
    // 版本号
    private static final int VERSION=2;
    // database id
    private static final long dbId=-1;
    // Logger
    private static final Logger LOG = LoggerFactory.getLogger(FileSnap.class);
    // snapshot文件的魔数(类似class文件的魔数)
    public final static int SNAP_MAGIC
        = ByteBuffer.wrap("ZKSN".getBytes()).getInt();
}
```

### FileTxnSnapLog源码分析

```
public class FileTxnSnapLog {
    //the direcotry containing the 
    //the transaction logs
    // 日志文件目录
    private final File dataDir;
    //the directory containing the
    //the snapshot directory
    // 快照文件目录
    private final File snapDir;
    // 事务日志
    private TxnLog txnLog;
    // 快照
    private SnapShot snapLog;
    // 版本号
    public final static int VERSION = 2;
    // 版本
    public final static String version = "version-";

    // Logger
    private static final Logger LOG = LoggerFactory.getLogger(FileTxnSnapLog.class);
}
```

说明：类的属性中包含了TxnLog和SnapShot接口，即对FileTxnSnapLog的很多操作都会转发给TxnLog和SnapLog进行操作，这是一种典型的组合方法。

FileTxnSnapLog包含了PlayBackListener内部类，用来接收事务应用过程中的回调，在Zookeeper数据恢复后期，会有事务修正过程，此过程会回调PlayBackListener来进行对应的数据修正。

## Watcher机制

对于Watcher机制而言，主要涉及的类主要如下。

​		Watcher，接口类型，其定义了process方法，需子类实现。

Event，接口类型，Watcher的内部类，无任何方法。

KeeperState，枚举类型，Event的内部类，表示Zookeeper所处的状态。

EventType，枚举类型，Event的内部类，表示Zookeeper中发生的事件类型。

WatchedEvent，表示对ZooKeeper上发生变化后的反馈，包含了KeeperState和EventType。

ClientWatchManager，接口类型，表示客户端的Watcher管理者，其定义了materialized方法，需子类实现。

ZKWatchManager，Zookeeper的内部类，继承ClientWatchManager。

MyWatcher，ZooKeeperMain的内部类，继承Watcher。

ServerCnxn，接口类型，继承Watcher，表示客户端与服务端的一个连接。

WatchManager，管理Watcher。