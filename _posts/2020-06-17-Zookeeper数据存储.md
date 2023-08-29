---
layout: post
categories: [Zookeeper]
description: none
keywords: Zookeeper
---
# Zookeeper数据存储
ZooKeeper中，数据存储分为两部分，内存数据(ZKDatabase)与磁盘数据(事务日志 + 事务快照)。

## ZKDatabase
ZooKeeper的数据模型是一棵树。 而从使用角度看，ZooKeeper就像一个内存数据库一样，在内存数据库中，存储了整棵树的内容，包括所有的节点路径、节点数据以及ACL信息等。

ZKDatabase是ZooKeeper的内存数据库，负责管理ZooKeeper的所有会话、DataTree存储和事务日志。

ZKDatabase会定时向磁盘dump快照数据，同时在ZooKeeper服务器启动的时候，会通过磁盘上的事务日志和快照数据文件恢复成一个完整的内存数据库。
主要属性为：
```java
    protected DataTree dataTree;
//key为sessionId,value为会话过期时间
    protected ConcurrentHashMap<Long, Integer> sessionsWithTimeouts;
//用于和磁盘交互事务日志文件和快照文件的类
    protected FileTxnSnapLog snapLog;
//主从数据同步时使用
    protected long minCommittedLog, maxCommittedLog;
    public static final int commitLogCount = 500;
    protected static int commitLogBuffer = 700;
//todo
    protected LinkedList<Proposal> committedLog = new LinkedList<Proposal>();
```

## DateTree
Zookeeper的数据模型是一棵树，DataTree是内存数据存储的核心，代表了内存中一份完整的数据(最新)，包括所有的节点路径，节点数据和ACL信息，对应watches等。
```java
DataTree:
- nodes: ConcurrentHashMap<String, DataNode>
- ephemerals: ConcurrentHashMap<Long, HashSet<String>>
- dataWatches: WatchManager
- childWatches: WatchManager
-----------------------------------------------------
+ convertAcls(List<ACL>): Long
+ convertLong(Long): List<ACL>
+ addDataNode(String, DataNode): void
+ createNode(String, byte, List<ACL>, long, int, long, long): String
+ deleteNode(String, long)
+ setData(String, byte, int, long, long)
+ getData(String, Stat, Watcher)
+ ......
```
ConcurrentHashMap<String, DataNode> nodes存储所有ZooKeeper节点信息，Key为节点路径，Value为DataNode。

ConcurrentHashMap<Long, HashSet<String>> ephemerals存储所有临时节点的信息，便于实时访问和及时清理。Key为客户端SessionID，Value为该客户端创建的所有临时节点路径集合。
```java
    //节点路径为key,节点数据内容DataNode为value.实时存储了所有的zk节点，使用ConcurrentHashMap保证并发性
    private final ConcurrentHashMap<String, DataNode> nodes =new ConcurrentHashMap<String, DataNode>();

//节点数据对应的watch
    private final WatchManager dataWatches = new WatchManager();

//节点路径对应的watch
    private final WatchManager childWatches = new WatchManager();

//key为sessionId,value为该会话对应的临时节点路径，方便实时访问和清理
    private final Map<Long, HashSet<String>> ephemerals = new ConcurrentHashMap<Long, HashSet<String>>();

//This set contains the paths of all container nodes
    private final Set<String> containers =Collections.newSetFromMap(new ConcurrentHashMap<String, Boolean>());

//This set contains the paths of all ttl nodes
    private final Set<String> ttls =Collections.newSetFromMap(new ConcurrentHashMap<String, Boolean>());
//内存数据库的最大zxid
public volatile long lastProcessedZxid = 0;
```

## DataNode
DataNode 是数据存储的最小单元，内部保存节点的数据内容(data[])、ACL列表(acl)和节点状态(stat)，同时记录父节点(parent)的引用和子节点列表(children)。
```java
DataTree:
- parent: DataNode
- data: byte[]
- acl: Long
- stat: StatPersisted
- children: Set<String>
-----------------------
+ addChild(): boolean
+ removeChild(): boolean
+ setChildren(): void
+ getChildren(): Set<String>
+ copyStat(Stat): void
+ deserialize(InputArchive, String)
+ Serialize(OutputArchive, String)
+ ......
```
主要属性为：
```java
//节点内容
    byte data[];
    Long acl;
    //节点状态，包括一些节点的元数据，如ephemeralOwner，czxid等
    public StatPersisted stat;
//子节点相对父节点路径集合，不包括父节点路径
    private Set<String> children = null;
```

## 事务日志
事务日志文件默认存储于dataDir。 也可以为事务日志单独配置文件存储目录dataLogDir。
事务日志文件的文件名是一个十六进制数字，高32位为Leader选举周期(epoch)，低32为是事务ZXID。
日志文件是二进制格式存储，ZooKeeper提供了解码工具：
```java
Java LogFormatter 日志文件
```
事务日志信息
```
ZooKeeper Transactional Log File with dbid 0 txnlog format version 2
## 事务日志文件头信息。
..11:07:41 session 0x144699552020000 cxid 0x0 zxid 0x300000002 createSession 3000
一次客户端会话创建的事务操作日志。
事务操作时间 + 客户端会话ID + CXID + ZXID + 操作类型 + 会话超时时间
..11:08:40 session 0x144699552020000 cxid 0x2 zxid 0x300000003 create `/test_log,#7631,v{s{31,s{'world',anyone}}},F,2
节点创建操作的事务操作日志。
事务操作时间 + 客户端会话ID + CXID + ZXID + 操作类型 + 节点路径 + 节点数据内容
```
文件存储主要包括事务日志文件的存储和快照文件的存储，分别与FileTxnLog和FileSnap类有关。 FileTxnLog 实现了TxnLog接口，提供了API可以获取日志和写入日志，首先先看一下事务日志文件的格式
```java
LogFile:
//一个日志文件由以下三个部分组成
 *     FileHeader TxnList ZeroPad
//1.文件头
 * FileHeader: {
 *     magic 4bytes (ZKLG)   
 *     version 4bytes
 *     dbid 8bytes
 *   }
 //事务内容
 * TxnList:
 *     Txn || Txn TxnList

 * Txn:
//一条事务日志的组成部分
 *     checksum Txnlen TxnHeader Record 0x42

 * checksum: 8bytes Adler32 is currently used
 *   calculated across payload -- Txnlen, TxnHeader, Record and 0x42
 *
 * Txnlen:
 *     len 4bytes
 *
 * TxnHeader: {
 *     sessionid 8bytes
 *     cxid 4bytes
 *     zxid 8bytes
 *     time 8bytes
 *     type 4bytes
 *   }
 *
 * Record:
 *     See Jute definition file for details on the various record types
 *
 * ZeroPad:
 *     0 padded to EOF (filled during preallocation stage)
```

主要分析下写入日志和日志截断的过程 写入日志
```java
public synchronized boolean append(TxnHeader hdr, Record txn)
        throws IOException
    {
        if (hdr == null) {
            return false;
        }
//lastZxidSeen:最大(新)的zxid
        if (hdr.getZxid() <= lastZxidSeen) {
            LOG.warn("Current zxid " + hdr.getZxid()
                    + " is <= " + lastZxidSeen + " for "
                    + hdr.getType());
        } else {
            lastZxidSeen = hdr.getZxid();
        }
//如果没有事务日志可写，需要关联一个新的文件流，写入日志文件头信息FileHeader，并马上强制刷盘
        if (logStream==null) {
           if(LOG.isInfoEnabled()){
                LOG.info("Creating new log file: " + Util.makeLogName(hdr.getZxid()));
           }

           logFileWrite = new File(logDir, Util.makeLogName(hdr.getZxid()));
           fos = new FileOutputStream(logFileWrite);
           logStream=new BufferedOutputStream(fos);
           oa = BinaryOutputArchive.getArchive(logStream);
           FileHeader fhdr = new FileHeader(TXNLOG_MAGIC,VERSION, dbId);
           fhdr.serialize(oa, "fileheader");
           // Make sure that the magic number is written before padding.
           logStream.flush();
           filePadding.setCurrentSize(fos.getChannel().position());
           streamsToFlush.add(fos);
        }
//确定事务日志文件是否需要扩容（预分配）
        filePadding.padFile(fos.getChannel());
//事务序列化
        byte[] buf = Util.marshallTxnEntry(hdr, txn);
        if (buf == null || buf.length == 0) {
            throw new IOException("Faulty serialization for header " +
                    "and txn");
        }
//生成Checksum
        Checksum crc = makeChecksumAlgorithm();
        crc.update(buf, 0, buf.length);
        oa.writeLong(crc.getValue(), "txnEntryCRC");
//写入事务日志文件流
        Util.writeTxnBytes(oa, buf);

        return true;
    }
```
主要流程为： 
- 确定是否有事务日志可写 当zookeeper服务器启动完成时需要进行第一次事务日志的写入，或是上一个事务日志写满时，都会处于与事务日志文件断开的状态。当logStream==null时需要关联一个新的文件流，写入日志文件头信息FileHeader，并马上强制刷盘。 
- 确定事务日志文件是否需要扩容（预分配）
```java
long padFile(FileChannel fileChannel) throws IOException {
//currentSize:当前文件的大小位置
//preAllocSize：默认64MB
        long newFileSize = calculateFileSizeWithPadding(fileChannel.position(), currentSize, preAllocSize);
        if (currentSize != newFileSize) {
            fileChannel.write((ByteBuffer) fill.position(0), newFileSize - fill.remaining());
            currentSize = newFileSize;
        }
        return currentSize;
    }
//判断是否需要扩容
public static long calculateFileSizeWithPadding(long position, long fileSize, long preAllocSize) {
        // If preAllocSize is positive and we are within 4KB of the known end of the file calculate a new file size
        if (preAllocSize > 0 && position + 4096 >= fileSize) {
            // If we have written more than we have previously preallocated we need to make sure the new
            // file size is larger than what we already have
            if (position > fileSize) {
                fileSize = position + preAllocSize;
                fileSize -= fileSize % preAllocSize;
            } else {
                fileSize += preAllocSize;
            }
        }

        return fileSize;
    }
```
从calculateFileSizeWithPadding中可以看出，当写入数据量超过4KB的时候便会将文件大小currentSize扩容到preAllocSize,默认为64MB,并将未写入部分填充0，好处是避免开辟新的磁盘块，减少磁盘Seek 
- 事务序列化 分别对事物头(TxnHeader)和事务体(Record)序列化，参考zookeeper源码分析(5)-序列化和协议 
- 生成Checksum 可校验事务日志文件的完整性和数据准确性 
- 写入事务日志文件流 将事物头，事务体和Checksum写入文件流中，由于使用的输出流是BufferedOutputStream，会先放到缓冲区中，不会真正写入 日志截断 在主从同步时，如果learner服务器的事务ID大于leader服务器的事务ID,将会要求learner服务器丢弃掉比leader服务器的事务ID大的事务日志。 
FileTxnIterator是可以指定zxid的事务日志迭代器，也就是说如果需要从zxid=11的位置开始创建一个迭代器，那么该台服务器上面在zxid=11之后的日志都会保存在该迭代器中。其主要属性为：
```java
public static class FileTxnIterator implements TxnLog.TxnIterator {
//事务日志的目录
        File logDir;
//需要从该事务ID处获得迭代器
        long zxid;
//zxid所在事务文件的文件头
        TxnHeader hdr;
//当前正在迭代的事务日志
        Record record;
//zxid所在的事务日志文件
        File logFile;
//输入流
        InputArchive ia;
        static final String CRC_ERROR="CRC check failed";
//输入流，可读取到zxid的位置
        PositionInputStream inputStream=null;
//比zxid所在事务日志文件大的事务文件集合
        private ArrayList<File> storedFiles;
··········省略代码·······
}
```

```java
public boolean truncate(long zxid) throws IOException {
        FileTxnIterator itr = null;
        try {
            itr = new FileTxnIterator(this.logDir, zxid);
            PositionInputStream input = itr.inputStream;
            if(input == null) {
                throw new IOException("No log files found to truncate! This could " +
                        "happen if you still have snapshots from an old setup or " +
                        "log files were deleted accidentally or dataLogDir was changed in zoo.cfg.");
            }
            long pos = input.getPosition();
            // now, truncate at the current position
            RandomAccessFile raf=new RandomAccessFile(itr.logFile,"rw");
            raf.setLength(pos);
            raf.close();
            while(itr.goToNextLog()) {
                if (!itr.logFile.delete()) {
                    LOG.warn("Unable to truncate {}", itr.logFile);
                }
            }
        } finally {
            close(itr);
        }
        return true;
    }
```
从代码中可以看出，截断的逻辑就是删掉zxid所在事务文件中比zxid大的事务日志，以及所有比该事务文件大的事务文件。

FileSnap 数据快照是用来记录zookeeper服务器在某一时刻的全量内存数据，并将其写入到指定位置磁盘上。存储内容包括DataTree信息和会话信息。FileSnap提供了快照相应的接口，，主要包括存储、序列化、反序列化、访问相应快照文件。

FileTxnSnapLog 封装了TxnLog和SnapShot，提供了从磁盘中恢复内存数据库的restore方法和保存快照的save方法,主要属性
```java
//the directory containing the
    //the transaction logs
    private final File dataDir;
    //the directory containing the snapshot directory
    private final File snapDir;
    private TxnLog txnLog;
    private SnapShot snapLog;
    // 版本号
    public final static int VERSION = 2;
    // 版本
    public final static String version = "version-";
```
首先看下保存快照的save方法
```java
//syncSnap： sync the snapshot immediately after write
public void save(DataTree dataTree,
                     ConcurrentHashMap<Long, Integer> sessionsWithTimeouts,
                     boolean syncSnap)
        throws IOException {
        long lastZxid = dataTree.lastProcessedZxid;
        File snapshotFile = new File(snapDir, Util.makeSnapshotName(lastZxid));
        LOG.info("Snapshotting: 0x{} to {}", Long.toHexString(lastZxid),
                snapshotFile);
        snapLog.serialize(dataTree, sessionsWithTimeouts, snapshotFile, syncSnap);

    }
```
主要流程就是根据当前dataTree的最新事务id生成快照文件名，然后将dataTree的内容和sessionsWithTimeouts（会话信息）序列化，存到指定磁盘位置。
## 服务器启动期间的数据初始化
就是磁盘中最新快照文件（全量数据）和它之后的事务日志数据（增量数据）的反序列化到内存数据库中的过程。在服务器启动时，需要先初始化FileTxnSnapLog和初始化 ZKDatabase
初始化FileTxnSnapLog
```java
 public FileTxnSnapLog(File dataDir, File snapDir) throws IOException {
        LOG.debug("Opening datadir:{} snapDir:{}", dataDir, snapDir);

        this.dataDir = new File(dataDir, version + VERSION);
        this.snapDir = new File(snapDir, version + VERSION);

        // by default create snap/log dirs, but otherwise complain instead
        // See ZOOKEEPER-1161 for more details
        boolean enableAutocreate = Boolean.valueOf(
                System.getProperty(ZOOKEEPER_DATADIR_AUTOCREATE,
                        ZOOKEEPER_DATADIR_AUTOCREATE_DEFAULT));

        if (!this.dataDir.exists()) {
            if (!enableAutocreate) {
                throw new DatadirException("Missing data directory "
                        + this.dataDir
                        + ", automatic data directory creation is disabled ("
                        + ZOOKEEPER_DATADIR_AUTOCREATE
                        + " is false). Please create this directory manually.");
            }

            if (!this.dataDir.mkdirs()) {
                throw new DatadirException("Unable to create data directory "
                        + this.dataDir);
            }
        }
        if (!this.dataDir.canWrite()) {
            throw new DatadirException("Cannot write to data directory " + this.dataDir);
        }

        if (!this.snapDir.exists()) {
            // by default create this directory, but otherwise complain instead
            // See ZOOKEEPER-1161 for more details
            if (!enableAutocreate) {
                throw new DatadirException("Missing snap directory "
                        + this.snapDir
                        + ", automatic data directory creation is disabled ("
                        + ZOOKEEPER_DATADIR_AUTOCREATE
                        + " is false). Please create this directory manually.");
            }

            if (!this.snapDir.mkdirs()) {
                throw new DatadirException("Unable to create snap directory "
                        + this.snapDir);
            }
        }
        if (!this.snapDir.canWrite()) {
            throw new DatadirException("Cannot write to snap directory " + this.snapDir);
        }

        // check content of transaction log and snapshot dirs if they are two different directories
        // See ZOOKEEPER-2967 for more details
        if(!this.dataDir.getPath().equals(this.snapDir.getPath())){
//用来检查当dataDir和snapDir不同时，dataDir是否包含了快照文件，snapDir是否包含了事务日志文件
            checkLogDir();
            checkSnapDir();
        }

        txnLog = new FileTxnLog(this.dataDir);
        snapLog = new FileSnap(this.snapDir);

        autoCreateDB = Boolean.parseBoolean(System.getProperty(ZOOKEEPER_DB_AUTOCREATE,
                ZOOKEEPER_DB_AUTOCREATE_DEFAULT));
    }
```
可以看到会在传入的datadir和snapdir目录下新生成version-2的目录，并且会判断目录是否创建成功，之后会创建txnLog和snapLog。 
- 初始化 ZKDatabase
```java
 public ZKDatabase(FileTxnSnapLog snapLog) {
        dataTree = new DataTree();
        sessionsWithTimeouts = new ConcurrentHashMap<Long, Integer>();
        this.snapLog = snapLog;
·······
}
```
可以看到主要初始化了DataTree和sessionsWithTimeouts，前者会在zookeeper创建一些配置跟节点，如/,/zookeeper,/zookeeper/quota等节点，与zookeeper自身服务器相关的节点。 之后调用数据初始化的方法为ZooKeeperServer.loadData
```java
public void loadData() throws IOException, InterruptedException {
//如果是leader服务器，会在lead方法中再次调用该方法，此时zkDb.isInitialized()=true,仅做快照存储的工作
        if(zkDb.isInitialized()){
            setZxid(zkDb.getDataTreeLastProcessedZxid());
        }
        else {
//第一次初始化
            setZxid(zkDb.loadDataBase());
        }
·········会话过期清理的代码···········
        // Make a clean snapshot
        takeSnapshot();
    }

    public void takeSnapshot() {
        takeSnapshot(false);
    }

    public void takeSnapshot(boolean syncSnap){
       txnLogFactory.save(zkDb.getDataTree(), zkDb.getSessionWithTimeOuts(), syncSnap);
·········省略异常检查···········
     }
```
第一次初始化的时候会调用zkDb.loadDataBase(),该方法最终会返回内存数据库最新的事务id
```java
public long loadDataBase() throws IOException {
        long zxid = snapLog.restore(dataTree, sessionsWithTimeouts, commitProposalPlaybackListener);
        initialized = true;
        return zxid;
    }
```
也就是调用FileTxnSnapLog.restore，首先介绍下FileTxnSnapLog的内部类PlayBackListener 它是用来接收事务应用过程中的回调，在Zookeeper数据恢复后期，会有事务修正过程(增量数据的反序列化过程)，此过程会回调PlayBackListener.onTxnLoaded来进行对应的数据修正。这里传入的是commitProposalPlaybackListener
FileTxnSnapLog.restore
```java
//方法参数中DataTree dt, Map<Long, Integer> sessions是要恢复内存数据库的对象，其实就是ZKDatabase中的属性
//PlayBackListener是用来修正事务日志时回调用的
    public long restore(DataTree dt, Map<Long, Integer> sessions,
                        PlayBackListener listener) throws IOException {
//解析快照数据
        long deserializeResult = snapLog.deserialize(dt, sessions);
        FileTxnLog txnLog = new FileTxnLog(dataDir);
        boolean trustEmptyDB;
        File initFile = new File(dataDir.getParent(), "initialize");
        if (Files.deleteIfExists(initFile.toPath())) {
            LOG.info("Initialize file found, an empty database will not block voting participation");
            trustEmptyDB = true;
        } else {
//
            trustEmptyDB = autoCreateDB;
        }
        if (-1L == deserializeResult) {
            /* this means that we couldn't find any snapshot, so we need to
             * initialize an empty database (reported in ZOOKEEPER-2325) */
            if (txnLog.getLastLoggedZxid() != -1) {
                throw new IOException(
                        "No snapshot found, but there are log entries. " +
                        "Something is broken!");
            }
//默认相信空磁盘数据，因为服务器第一次启动的时候数据一般为空
            if (trustEmptyDB) {
                /* TODO: (br33d) we should either put a ConcurrentHashMap on restore()
                 *       or use Map on save() */
                save(dt, (ConcurrentHashMap<Long, Integer>)sessions, false);

                /* return a zxid of 0, since we know the database is empty */
                return 0L;
            } else {
                /* return a zxid of -1, since we are possibly missing data */
                LOG.warn("Unexpected empty data tree, setting zxid to -1");
                dt.lastProcessedZxid = -1L;
                return -1L;
            }
        }
        return fastForwardFromEdits(dt, sessions, listener);
    }
```
解析快照数据 解析快照数据到datatree和sessions中，取出最新的100个快照数据，依次解析判断快照文件是否有数据且是可用的snapLog.deserialize(dt, sessions),返回快照文件数据的最大ZXID
```java
public long deserialize(DataTree dt, Map<Long, Integer> sessions)
            throws IOException {
        // we run through 100 snapshots (not all of them)
        // if we cannot get it running within 100 snapshots
        // we should  give up
        List<File> snapList = findNValidSnapshots(100);
        if (snapList.size() == 0) {
            return -1L;
        }
        File snap = null;
        boolean foundValid = false;
        for (int i = 0, snapListSize = snapList.size(); i < snapListSize; i++) {
            snap = snapList.get(i);
            LOG.info("Reading snapshot " + snap);
            try (InputStream snapIS = new BufferedInputStream(new FileInputStream(snap));
                 CheckedInputStream crcIn = new CheckedInputStream(snapIS, new Adler32())) {
                InputArchive ia = BinaryInputArchive.getArchive(crcIn);
                deserialize(dt, sessions, ia);
                long checkSum = crcIn.getChecksum().getValue();
                long val = ia.readLong("val");
                if (val != checkSum) {
                    throw new IOException("CRC corruption in snapshot :  " + snap);
                }
                foundValid = true;
                break;
            } catch (IOException e) {
                LOG.warn("problem reading snap file " + snap, e);
            }
        }
        if (!foundValid) {
            throw new IOException("Not able to find valid snapshots in " + snapDir);
        }
        dt.lastProcessedZxid = Util.getZxidFromName(snap.getName(), SNAPSHOT_FILE_PREFIX);
        return dt.lastProcessedZxid;
    }
```
若返回-1，说明不存在快照文件： 如果事务日志文件zxid也为-1，说明磁盘数据为空，则将空数据快照一下，返回最大事务id,为0。否则，调用fastForwardFromEdits
获取最新的ZXID
```java
public long fastForwardFromEdits(DataTree dt, Map<Long, Integer> sessions,
                                     PlayBackListener listener) throws IOException {

        TxnIterator itr = txnLog.read(dt.lastProcessedZxid+1);
        long highestZxid = dt.lastProcessedZxid;
        TxnHeader hdr;
        try {
            while (true) {
                // iterator points to
                // the first valid txn when initialized
                hdr = itr.getHeader();
                if (hdr == null) {
                    //empty logs
                    return dt.lastProcessedZxid;
                }
                if (hdr.getZxid() < highestZxid && highestZxid != 0) {
                    LOG.error("{}(highestZxid) > {}(next log) for type {}",
                            highestZxid, hdr.getZxid(), hdr.getType());
                } else {
                    highestZxid = hdr.getZxid();
                }
                try {
                    processTransaction(hdr,dt,sessions, itr.getTxn());
                } catch(KeeperException.NoNodeException e) {
                   throw new IOException("Failed to process transaction type: " +
                         hdr.getType() + " error: " + e.getMessage(), e);
                }
                listener.onTxnLoaded(hdr, itr.getTxn());
                if (!itr.next())
                    break;
            }
        } finally {
            if (itr != null) {
                itr.close();
            }
        }
        return highestZxid;
    }
```
首先基于当前dt.lastProcessedZxid+1获取一个事务日志迭代器，这些事务日志是需要更新的增量数据。while循环一条条迭代这些事务日志，不断的更新highestZxid，最终将其返回。 5.应用事务 在循环过程中处理事务日志processTransaction,也就是根据事务日志类型不断的更新sessions 和DataTree中的数据内容 回调事务 回调listener.onTxnLoaded，就是ZKDatabase中的commitProposalPlaybackListener
```java
private final PlayBackListener commitProposalPlaybackListener = new PlayBackListener() {
        public void onTxnLoaded(TxnHeader hdr, Record txn){
            addCommittedProposal(hdr, txn);
        }
    };

private void addCommittedProposal(TxnHeader hdr, Record txn) {
        Request r = new Request(0, hdr.getCxid(), hdr.getType(), hdr, txn, hdr.getZxid());
        addCommittedProposal(r);
    }

    /**
     * maintains a list of last <i>committedLog</i>
     *  or so committed requests. This is used for
     * fast follower synchronization.
     * @param request committed request
     */
    public void addCommittedProposal(Request request) {
        WriteLock wl = logLock.writeLock();
        try {
            wl.lock();
            if (committedLog.size() > commitLogCount) {
                committedLog.removeFirst();
                minCommittedLog = committedLog.getFirst().packet.getZxid();
            }
            if (committedLog.isEmpty()) {
//
                minCommittedLog = request.zxid;
                maxCommittedLog = request.zxid;
            }

            byte[] data = SerializeUtils.serializeRequest(request);
            QuorumPacket pp = new QuorumPacket(Leader.PROPOSAL, request.zxid, data, null);
            Proposal p = new Proposal();
            p.packet = pp;
            p.request = request;
            committedLog.add(p);
            maxCommittedLog = p.packet.getZxid();
        } finally {
            wl.unlock();
        }
    }
```
主要逻辑在addCommittedProposal方法中，构造了一个LinkedList<Proposal> committedLog,用来存储过来的每一条增量事务日志，minCommittedLog保存的是第一条增量事务日志的zxid, maxCommittedLog保存的是最后以条增量事务日志的zxid。这三个变量是用来主从做快速同步判断用的。 7.epoch校验 epoch标识了当前leader的周期，每次选举产生一个新的Leader服务器之后，就会生成一个新的epoch。集群间相互通信的过程中，都会带上这个epoch以确保彼此在同一个Leader周期内。 对于leader服务器，完成数据初始化时会将自己的currentEpoch和刚解析出来的最大zxid放到leaderStateSummary中，和主动连接的learner服务器的epoch和最大zxid对比，必须保证leader服务器的leaderStateSummary大于learner服务器的StateSummary才能说明leader服务器的数据是比learner服务器新的，然后leader服务器才可以开启新一轮的epoch，进行数据同步的工作。

## 主从服务器间的数据同步
Leader数据同步发送过程
```java
public void run() {
           ····省略接收ACKEPOCH消息之前的交互过程···
            //learner zxid
            peerLastZxid = ss.getLastZxid();
           
            // Take any necessary action if we need to send TRUNC or DIFF
            // startForwarding() will be called in all cases
            //确定是否需要进行全量同步
            boolean needSnap = syncFollower(peerLastZxid, leader.zk.getZKDatabase(), leader);
            
            LOG.debug("Sending NEWLEADER message to " + sid);
            // the version of this quorumVerifier will be set by leader.lead() in case
            // the leader is just being established. waitForEpochAck makes sure that readyToStart is true if
            // we got here, so the version was set
//发送NEWLEADER消息
            if (getVersion() < 0x10000) {
                QuorumPacket newLeaderQP = new QuorumPacket(Leader.NEWLEADER,
                        newLeaderZxid, null, null);
                oa.writeRecord(newLeaderQP, "packet");
            } else {
                QuorumPacket newLeaderQP = new QuorumPacket(Leader.NEWLEADER,
                        newLeaderZxid, leader.self.getLastSeenQuorumVerifier()
                                .toString().getBytes(), null);
                queuedPackets.add(newLeaderQP);
            }
//强刷，这里对应的DIFF/TRUNC/DIFF+TRUNC方式的同步
            bufferedOutput.flush();

            /* if we are not truncating or sending a diff just send a snapshot */
            if (needSnap) {
//全量同步
                boolean exemptFromThrottle = getLearnerType() != LearnerType.OBSERVER;
                LearnerSnapshot snapshot = 
                        leader.getLearnerSnapshotThrottler().beginSnapshot(exemptFromThrottle);
                try {
                    long zxidToSend = leader.zk.getZKDatabase().getDataTreeLastProcessedZxid();
                    oa.writeRecord(new QuorumPacket(Leader.SNAP, zxidToSend, null, null), "packet");
                    bufferedOutput.flush();
                    // Dump data to peer
                    leader.zk.getZKDatabase().serializeSnapshot(oa);
                    oa.writeString("BenWasHere", "signature");
//强刷，这里对应的SNAP方式的同步
            bufferedOutput.flush();
                    bufferedOutput.flush();
                } finally {
                    snapshot.close();
                }
            }

            // Start thread that blast packets in the queue to learner
            startSendingPackets();
            
        //等待learner服务器的同步完成的ACK通知
            qp = new QuorumPacket();
            ia.readRecord(qp, "packet");
            if(qp.getType() != Leader.ACK){
                LOG.error("Next packet was supposed to be an ACK,"
                    + " but received packet: {}", packetToString(qp));
                return;
            }

            if(LOG.isDebugEnabled()){
                LOG.debug("Received NEWLEADER-ACK message from " + sid);   
            }
            leader.waitForNewLeaderAck(getSid(), qp.getZxid());
//同步时间检测，不能超过tickTime*syncLimit
            syncLimitCheck.start();
            
            // now that the ack has been processed expect the syncLimit
            sock.setSoTimeout(leader.self.tickTime * leader.self.syncLimit);

            /*
             * Wait until leader starts up
             */
            synchronized(leader.zk){
                while(!leader.zk.isRunning() && !this.isInterrupted()){
                    leader.zk.wait(20);
                }
            }
            // Mutation packets will be queued during the serialize,
            // so we need to mark when the peer can actually start
            // using the data
            //
            LOG.debug("Sending UPTODATE message to " + sid);      
            queuedPackets.add(new QuorumPacket(Leader.UPTODATE, -1, null, null));

          ···············同步完成，开始与learner服务器的正常通信···········
    }
```
在服务器数据初始化时候，我们提到内存数据库zkDatabase会保存最新快照之后的增量数据， LinkedList<Proposal> committedLog:用来存储过来的每一条增量事务日志 minCommittedLog:第一条增量事务日志的zxid maxCommittedLog:最后一条增量事务日志的zxid Leader服务器会根据learner服务器的最大事务ID: peerLastZxid和minCommittedLog/ maxCommittedLog之间的大小关系来最终确定是差异同步还是全量同步，主要逻辑在syncFollower(peerLastZxid, leader.zk.getZKDatabase(), leader)
```java
public boolean syncFollower(long peerLastZxid, ZKDatabase db, Leader leader) {
//learner服务器zxid是否为0
        boolean isPeerNewEpochZxid = (peerLastZxid & 0xffffffffL) == 0;
        // Keep track of the latest zxid which already queued
        long currentZxid = peerLastZxid;
        boolean needSnap = true;
//是否设置了快照大小参数，默认设置了，且snapshotSizeFactor=0.33
        boolean txnLogSyncEnabled = db.isTxnLogSyncEnabled();
        ReentrantReadWriteLock lock = db.getLogLock();
        ReadLock rl = lock.readLock();
        try {
            rl.lock();
            long maxCommittedLog = db.getmaxCommittedLog();
            long minCommittedLog = db.getminCommittedLog();
            long lastProcessedZxid = db.getDataTreeLastProcessedZxid();
            if (db.getCommittedLog().isEmpty()) {
                /*
                 * It is possible that committedLog is empty. In that case
                 * setting these value to the latest txn in leader db
                 * will reduce the case that we need to handle
                 *
                 * Here is how each case handle by the if block below
                 * 1. lastProcessZxid == peerZxid -> Handle by (2)
                 * 2. lastProcessZxid < peerZxid -> Handle by (3)
                 * 3. lastProcessZxid > peerZxid -> Handle by (5)
                 */
                minCommittedLog = lastProcessedZxid;
                maxCommittedLog = lastProcessedZxid;
            }

            /*
             * Here are the cases that we want to handle
             *
             * 1. Force sending snapshot (for testing purpose)
             * 2. Peer and leader is already sync, send empty diff
             * 3. Follower has txn that we haven't seen. This may be old leader
             *    so we need to send TRUNC. However, if peer has newEpochZxid,
             *    we cannot send TRUNC since the follower has no txnlog
             * 4. Follower is within committedLog range or already in-sync.
             *    We may need to send DIFF or TRUNC depending on follower's zxid
             *    We always send empty DIFF if follower is already in-sync
             * 5. Follower missed the committedLog. We will try to use on-disk
             *    txnlog + committedLog to sync with follower. If that fail,
             *    we will send snapshot
             */

            if (forceSnapSync) {
                // Force leader to use snapshot to sync with follower
                LOG.warn("Forcing snapshot sync - should not see this in production");
            } else if (lastProcessedZxid == peerLastZxid) {
                // Follower is already sync with us, send empty diff
             //将packet发送到queuedPackets中，queuedPackets是负责发送消息到learner服务器的队列
                queueOpPacket(Leader.DIFF, peerLastZxid);
                needOpPacket = false;
                needSnap = false;
            } else if (peerLastZxid > maxCommittedLog && !isPeerNewEpochZxid) {
                // Newer than committedLog, send trunc and done
                queueOpPacket(Leader.TRUNC, maxCommittedLog);
                currentZxid = maxCommittedLog;
                needOpPacket = false;
                needSnap = false;
            } else if ((maxCommittedLog >= peerLastZxid)
                    && (minCommittedLog <= peerLastZxid)) {
                // Follower is within commitLog range
                Iterator<Proposal> itr = db.getCommittedLog().iterator();
//差异化同步，发送(peerLaxtZxid, maxZxid]之间的消息给learner服务器
                currentZxid = queueCommittedProposals(itr, peerLastZxid,
                                                     null, maxCommittedLog);
                needSnap = false;
            } else if (peerLastZxid < minCommittedLog && txnLogSyncEnabled) {
                // Use txnlog and committedLog to sync

                // Calculate sizeLimit that we allow to retrieve txnlog from disk
                long sizeLimit = db.calculateTxnLogSizeLimit();
                // This method can return empty iterator if the requested zxid
                // is older than on-disk txnlog
                Iterator<Proposal> txnLogItr = db.getProposalsFromTxnLog(
                        peerLastZxid, sizeLimit);
                if (txnLogItr.hasNext()) {
                   
                    currentZxid = queueCommittedProposals(txnLogItr, peerLastZxid,
                                                         minCommittedLog, maxCommittedLog);

                    currentZxid = queueCommittedProposals(committedLogItr, currentZxid,
                                                         null, maxCommittedLog);
                    needSnap = false;
                }
                // closing the resources
                if (txnLogItr instanceof TxnLogProposalIterator) {
                    TxnLogProposalIterator txnProposalItr = (TxnLogProposalIterator) txnLogItr;
                    txnProposalItr.close();
                }
            } else {
                LOG.warn("Unhandled scenario for peer sid: " +  getSid());
            }
            LOG.debug("Start forwarding 0x" + Long.toHexString(currentZxid) +
                      " for peer sid: " +  getSid());
//lets the leader know that a follower is capable of following and is done syncing
//已经通过的提议但是还没来得及提交的Proposal
            leaderLastZxid = leader.startForwarding(this, currentZxid);
        } finally {
            rl.unlock();
        }
//needOpPacket：用来判断是否需要发送TRUNC或DIFF消息给发送队列，默认为true
        if (needOpPacket && !needSnap) {
            // This should never happen, but we should fall back to sending
            // snapshot just in case.
            LOG.error("Unhandled scenario for peer sid: " +  getSid() +
                     " fall back to use snapshot");
            needSnap = true;
        }
        return needSnap;
    }
```
可以看出同步方式可大致分为5种: 1.强制快照同步 可设置forceSnapSync为true，用于测试使用，默认为false 2.不需要同步 此时主从最大zxid一致，不需要同步，仅需要发送一个DIFF消息即可 3.回滚同步 learner服务器zxid peerLastZxid大于leader服务器zxid lastProcessedZxid，并且peerLastZxid>0,此时需要从服务器丢弃大于lastProcessedZxid的事务日志，会发送TRUNC消息给learner服务器queueOpPacket(Leader.TRUNC, maxCommittedLog); 4.差异化同步（TRUNC+DIFF同步）

peerLastZxid位于minCommittedLog和 maxCommittedLog之间，但peerLastZxid找不到这个范围内的值，则先回滚到离peerLastZxid最近的前一条消息prevProposalZxid，然后再进行(prevProposalZxid, maxZxid]之间的zxid同步
peerLastZxid位于minCommittedLog和 maxCommittedLog之间，且peerLastZxid真实存在，则只需要进行(peerLaxtZxid, maxZxid]之间的zxid同步，与上面一条的差别处理可见LearnerHanler.queueCommittedProposals

```java
protected long queueCommittedProposals(Iterator<Proposal> itr,
            long peerLastZxid, Long maxZxid, Long lastCommittedZxid) {
        boolean isPeerNewEpochZxid = (peerLastZxid & 0xffffffffL) == 0;
        long queuedZxid = peerLastZxid;
        // as we look through proposals, this variable keeps track of previous
        // proposal Id.
        long prevProposalZxid = -1;
        while (itr.hasNext()) {
            Proposal propose = itr.next();

            long packetZxid = propose.packet.getZxid();
            // abort if we hit the limit
            if ((maxZxid != null) && (packetZxid > maxZxid)) {
                break;
            }

            // skip the proposals the peer already has
            if (packetZxid < peerLastZxid) {
                prevProposalZxid = packetZxid;
                continue;
            }

            // If we are sending the first packet, figure out whether to trunc
            // or diff
            if (needOpPacket) {

                // Send diff when we see the follower's zxid in our history,情况5-1
                if (packetZxid == peerLastZxid) {
                    LOG.info("Sending DIFF zxid=0x" +
                             Long.toHexString(lastCommittedZxid) +
                             " for peer sid: " + getSid());
                    queueOpPacket(Leader.DIFF, lastCommittedZxid);
                    needOpPacket = false;
                    continue;
                }

                if (isPeerNewEpochZxid) {
                   // Send diff and fall through if zxid is of a new-epoch
                   LOG.info("Sending DIFF zxid=0x" +
                            Long.toHexString(lastCommittedZxid) +
                            " for peer sid: " + getSid());
                   queueOpPacket(Leader.DIFF, lastCommittedZxid);
                   needOpPacket = false;
                } else if (packetZxid > peerLastZxid  ) {
                    // Peer have some proposals that the leader hasn't seen yet，情况4
                    // it may used to be a leader
                    if (ZxidUtils.getEpochFromZxid(packetZxid) !=
                            ZxidUtils.getEpochFromZxid(peerLastZxid)) {
                        // We cannot send TRUNC that cross epoch boundary.
                        // The learner will crash if it is asked to do so.
                        // We will send snapshot this those cases.
                        LOG.warn("Cannot send TRUNC to peer sid: " + getSid() +
                                 " peer zxid is from different epoch" );
                        return queuedZxid;
                    }

                    LOG.info("Sending TRUNC zxid=0x" +
                            Long.toHexString(prevProposalZxid) +
                            " for peer sid: " + getSid());
                    queueOpPacket(Leader.TRUNC, prevProposalZxid);
                    needOpPacket = false;
                }
            }

            if (packetZxid <= queuedZxid) {
                // We can get here, if we don't have op packet to queue
                // or there is a duplicate txn in a given iterator
                continue;
            }

            // Since this is already a committed proposal, we need to follow
            // it by a commit packet
//发送PROPOSAL消息，包含数据信息
            queuePacket(propose.packet);
//发送COMMIT消息，仅包含需要提交的zxid信息
            queueOpPacket(Leader.COMMIT, packetZxid);
            queuedZxid = packetZxid;

        }

        if (needOpPacket && isPeerNewEpochZxid) {
            // We will send DIFF for this kind of zxid in any case. This if-block
            // is the catch when our history older than learner and there is
            // no new txn since then. So we need an empty diff
            LOG.info("Sending DIFF zxid=0x" +
                     Long.toHexString(lastCommittedZxid) +
                     " for peer sid: " + getSid());
            queueOpPacket(Leader.DIFF, lastCommittedZxid);
            needOpPacket = false;
        }

        return queuedZxid;
    }
```
如果peerLastZxid < minCommittedLog,但是所处事务日志文件txnLog位置之后的事务大小小于最近快照中后snapSize * snapshotSizeFactor的大小，则采用txnLog + committedLog的方式同步，分为两部分：
```java
currentZxid = queueCommittedProposals(txnLogItr, peerLastZxid,
                                                         minCommittedLog, maxCommittedLog);

                    currentZxid = queueCommittedProposals(committedLogItr, currentZxid,
                                                         null, maxCommittedLog);
```
全量同步 如果peerLastZxid小于以上情况，则进行全量同步，该方法返回true，回到LearnerHandler.run,会发送SNAP消息，并将整个ZKDatabase序列化，发送出去 之后会开启线程异步发送queuedPackets队列消息，等待learner服务器的同步完成ACK消息。

## Learner数据同步接收过程
当Learner服务器发送完ACKEPOCH消息后，便会进入同步过程Learner.syncWithLeader(Follewer/Observer都会调用此方法)
```java
protected void syncWithLeader(long newLeaderZxid) throws Exception{
        QuorumPacket ack = new QuorumPacket(Leader.ACK, 0, null, null);
        QuorumPacket qp = new QuorumPacket();
        long newEpoch = ZxidUtils.getEpochFromZxid(newLeaderZxid);
        
        QuorumVerifier newLeaderQV = null;
        
        // In the DIFF case we don't need to do a snapshot because the transactions will sync on top of any existing snapshot
        // For SNAP and TRUNC the snapshot is needed to save that history
        boolean snapshotNeeded = true;
        boolean syncSnapshot = false;
        readPacket(qp);
        LinkedList<Long> packetsCommitted = new LinkedList<Long>();
        LinkedList<PacketInFlight> packetsNotCommitted = new LinkedList<PacketInFlight>();
        synchronized (zk) {
            if (qp.getType() == Leader.DIFF) {
                LOG.info("Getting a diff from the leader 0x{}", Long.toHexString(qp.getZxid()));
                snapshotNeeded = false;
            }
            else if (qp.getType() == Leader.SNAP) {
                LOG.info("Getting a snapshot from leader 0x" + Long.toHexString(qp.getZxid()));
                // The leader is going to dump the database
                // db is clear as part of deserializeSnapshot()
                zk.getZKDatabase().deserializeSnapshot(leaderIs);
                // ZOOKEEPER-2819: overwrite config node content extracted
                // from leader snapshot with local config, to avoid potential
                // inconsistency of config node content during rolling restart.
                if (!QuorumPeerConfig.isReconfigEnabled()) {
                    LOG.debug("Reset config node content from local config after deserialization of snapshot.");
                    zk.getZKDatabase().initConfigInZKDatabase(self.getQuorumVerifier());
                }
                String signature = leaderIs.readString("signature");
                if (!signature.equals("BenWasHere")) {
                    LOG.error("Missing signature. Got " + signature);
                    throw new IOException("Missing signature");                   
                }
                zk.getZKDatabase().setlastProcessedZxid(qp.getZxid());

                // immediately persist the latest snapshot when there is txn log gap
                syncSnapshot = true;
            } else if (qp.getType() == Leader.TRUNC) {
                //we need to truncate the log to the lastzxid of the leader
                LOG.warn("Truncating log to get in sync with the leader 0x"
                        + Long.toHexString(qp.getZxid()));
                boolean truncated=zk.getZKDatabase().truncateLog(qp.getZxid());
                if (!truncated) {
                    // not able to truncate the log
                    LOG.error("Not able to truncate the log "
                            + Long.toHexString(qp.getZxid()));
                    System.exit(13);
                }
                zk.getZKDatabase().setlastProcessedZxid(qp.getZxid());

            }
            else {
                LOG.error("Got unexpected packet from leader: {}, exiting ... ",
                          LearnerHandler.packetToString(qp));
                System.exit(13);

            }
            zk.getZKDatabase().initConfigInZKDatabase(self.getQuorumVerifier());
            zk.createSessionTracker();            
            
            long lastQueued = 0;

            // in Zab V1.0 (ZK 3.4+) we might take a snapshot when we get the NEWLEADER message, but in pre V1.0
            // we take the snapshot on the UPDATE message, since Zab V1.0 also gets the UPDATE (after the NEWLEADER)
            // we need to make sure that we don't take the snapshot twice.
            boolean isPreZAB1_0 = true;
            //If we are not going to take the snapshot be sure the transactions are not applied in memory
            // but written out to the transaction log
            boolean writeToTxnLog = !snapshotNeeded;
            // we are now going to start getting transactions to apply followed by an UPTODATE
            outerLoop:
            while (self.isRunning()) {
                readPacket(qp);
                switch(qp.getType()) {
                case Leader.PROPOSAL:
                    PacketInFlight pif = new PacketInFlight();
                    pif.hdr = new TxnHeader();
                    pif.rec = SerializeUtils.deserializeTxn(qp.getData(), pif.hdr);
                    if (pif.hdr.getZxid() != lastQueued + 1) {
                    LOG.warn("Got zxid 0x"
                            + Long.toHexString(pif.hdr.getZxid())
                            + " expected 0x"
                            + Long.toHexString(lastQueued + 1));
                    }
                    lastQueued = pif.hdr.getZxid();
                    
                    if (pif.hdr.getType() == OpCode.reconfig){                
                        SetDataTxn setDataTxn = (SetDataTxn) pif.rec;       
                       QuorumVerifier qv = self.configFromString(new String(setDataTxn.getData()));
                       self.setLastSeenQuorumVerifier(qv, true);                               
                    }
                    
                    packetsNotCommitted.add(pif);
                    break;
                case Leader.COMMIT:
                case Leader.COMMITANDACTIVATE:
                    pif = packetsNotCommitted.peekFirst();
                    if (pif.hdr.getZxid() == qp.getZxid() && qp.getType() == Leader.COMMITANDACTIVATE) {
                        QuorumVerifier qv = self.configFromString(new String(((SetDataTxn) pif.rec).getData()));
                        boolean majorChange = self.processReconfig(qv, ByteBuffer.wrap(qp.getData()).getLong(),
                                qp.getZxid(), true);
                        if (majorChange) {
                            throw new Exception("changes proposed in reconfig");
                        }
                    }
                    if (!writeToTxnLog) {
                        if (pif.hdr.getZxid() != qp.getZxid()) {
                            LOG.warn("Committing " + qp.getZxid() + ", but next proposal is " + pif.hdr.getZxid());
                        } else {
                            zk.processTxn(pif.hdr, pif.rec);
                            packetsNotCommitted.remove();
                        }
                    } else {
                        packetsCommitted.add(qp.getZxid());
                    }
                    break;
                case Leader.INFORM:
                case Leader.INFORMANDACTIVATE:
                    PacketInFlight packet = new PacketInFlight();
                    packet.hdr = new TxnHeader();

                    if (qp.getType() == Leader.INFORMANDACTIVATE) {
                        ByteBuffer buffer = ByteBuffer.wrap(qp.getData());
                        long suggestedLeaderId = buffer.getLong();
                        byte[] remainingdata = new byte[buffer.remaining()];
                        buffer.get(remainingdata);
                        packet.rec = SerializeUtils.deserializeTxn(remainingdata, packet.hdr);
                        QuorumVerifier qv = self.configFromString(new String(((SetDataTxn)packet.rec).getData()));
                        boolean majorChange =
                                self.processReconfig(qv, suggestedLeaderId, qp.getZxid(), true);
                        if (majorChange) {
                            throw new Exception("changes proposed in reconfig");
                        }
                    } else {
                        packet.rec = SerializeUtils.deserializeTxn(qp.getData(), packet.hdr);
                        // Log warning message if txn comes out-of-order
                        if (packet.hdr.getZxid() != lastQueued + 1) {
                            LOG.warn("Got zxid 0x"
                                    + Long.toHexString(packet.hdr.getZxid())
                                    + " expected 0x"
                                    + Long.toHexString(lastQueued + 1));
                        }
                        lastQueued = packet.hdr.getZxid();
                    }
                    if (!writeToTxnLog) {
                        // Apply to db directly if we haven't taken the snapshot
                        zk.processTxn(packet.hdr, packet.rec);
                    } else {
                        packetsNotCommitted.add(packet);
                        packetsCommitted.add(qp.getZxid());
                    }

                    break;                
                case Leader.UPTODATE:
                    LOG.info("Learner received UPTODATE message");                                      
                    if (newLeaderQV!=null) {
                       boolean majorChange =
                           self.processReconfig(newLeaderQV, null, null, true);
                       if (majorChange) {
                           throw new Exception("changes proposed in reconfig");
                       }
                    }
                    if (isPreZAB1_0) {
                        zk.takeSnapshot(syncSnapshot);
                        self.setCurrentEpoch(newEpoch);
                    }
                    self.setZooKeeperServer(zk);
                    self.adminServer.setZooKeeperServer(zk);
                    break outerLoop;
                case Leader.NEWLEADER: // Getting NEWLEADER here instead of in discovery 
                    // means this is Zab 1.0
                   LOG.info("Learner received NEWLEADER message");
                   if (qp.getData()!=null && qp.getData().length > 1) {
                       try {                       
                           QuorumVerifier qv = self.configFromString(new String(qp.getData()));
                           self.setLastSeenQuorumVerifier(qv, true);
                           newLeaderQV = qv;
                       } catch (Exception e) {
                           e.printStackTrace();
                       }
                   }

                   if (snapshotNeeded) {
                       zk.takeSnapshot(syncSnapshot);
                   }
                   
                    self.setCurrentEpoch(newEpoch);
                    writeToTxnLog = true; //Anything after this needs to go to the transaction log, not applied directly in memory
                    isPreZAB1_0 = false;
                    writePacket(new QuorumPacket(Leader.ACK, newLeaderZxid, null, null), true);
                    break;
                }
            }
        }
        ack.setZxid(ZxidUtils.makeZxid(newEpoch, 0));
        writePacket(ack, true);
        sock.setSoTimeout(self.tickTime * self.syncLimit);
        zk.startup();
        /*
         * Update the election vote here to ensure that all members of the
         * ensemble report the same vote to new servers that start up and
         * send leader election notifications to the ensemble.
         * 
         * @see https://issues.apache.org/jira/browse/ZOOKEEPER-1732
         */
        self.updateElectionVote(newEpoch);

        // We need to log the stuff that came in between the snapshot and the uptodate
        if (zk instanceof FollowerZooKeeperServer) {
            FollowerZooKeeperServer fzk = (FollowerZooKeeperServer)zk;
            for(PacketInFlight p: packetsNotCommitted) {
                fzk.logRequest(p.hdr, p.rec);
            }
            for(Long zxid: packetsCommitted) {
                fzk.commit(zxid);
            }
        } else if (zk instanceof ObserverZooKeeperServer) {
            // Similar to follower, we need to log requests between the snapshot
            // and UPTODATE
            ObserverZooKeeperServer ozk = (ObserverZooKeeperServer) zk;
            for (PacketInFlight p : packetsNotCommitted) {
                Long zxid = packetsCommitted.peekFirst();
                if (p.hdr.getZxid() != zxid) {
                    // log warning message if there is no matching commit
                    // old leader send outstanding proposal to observer
                    LOG.warn("Committing " + Long.toHexString(zxid)
                            + ", but next proposal is "
                            + Long.toHexString(p.hdr.getZxid()));
                    continue;
                }
                packetsCommitted.remove();
                Request request = new Request(null, p.hdr.getClientId(),
                        p.hdr.getCxid(), p.hdr.getType(), null, null);
                request.setTxn(p.rec);
                request.setHdr(p.hdr);
                ozk.commitRequest(request);
            }
        } else {
            // New server type need to handle in-flight packets
            throw new UnsupportedOperationException("Unknown server type");
        }
    }
```
大致流程为：首先会判断第一个接收到的消息类型是DIFF，SNAP还是TRUNC，分别进行不同的数据同步准备。然后开始不断读取同步消息，直到接收到NEWLEADER消息后，发送ACK给leader服务器，等待leader服务器的UPTODATE消息，表示同步完成，然后再发送ACK给leader服务器，表示learner服务器也知道了，开始启动zkServer,对外提供服务。







