---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码Translog
Lucene的数据是先写入缓冲区，数据写入缓冲区中，没有保存到磁盘中，一旦发生程序崩溃或者服务器宕机，数据就会发生丢失。为保证可靠性，Elasticsearch引入了Translog（事务日志）。每次数据被添加或者删除，都要在Translog中添加一条记录。这样一旦发生崩溃，数据可以从Translog中恢复。

Translog仅仅用于Lucene的数据恢复，不是用于支持事务。

不过，数据被写到Translog也没有保存到磁盘。一般情况下，对磁盘文件的write操作，更新到pagecache，而脏页面不会立即更新到磁盘中，而是由操作系统统一调度，如由专门的flusher内核线程在满足一定条件时（如一定时间间隔、内存中的脏页面达到一定比例）将脏页面同步到磁盘上。因此如果服务器在write之后、磁盘同步之前宕机，则数据会丢失。这时需要调用操作系统提供的fsync功能来确保文件所有已修改的内容被正确同步到磁盘上。

Elasticsearch提供以下几个参数配置来控制Translog的同步，这些参数可以动态调整，每个索引都可以设置不同的值：
- index.translog.durability
该参数控制如何同步Translog数据。有两个选项：
- request（默认）：每次请求（包括index、delete、update、bulk）都会执行fsync，将Translog的数据同步到磁盘中。
- async：异步提交Translog数据。和index.translog.sync_interval参数配合使用，每隔sync_interval的时间将Translog数据提交到磁盘，这样一来性能会有所提升，但是在间隔时间内的数据就会有丢失的风险。

request — 每操作都写（默认策略），可靠性最高。

async — 异步定时写，可靠性跟时间间隔有关，可能存在数据丢失情况。异步情况下可以设置以下两个参数（如果使用默认值则没必要设置）：
- index.translog.flush_threshold_size：该参数控制Translog的大小，默认为512MB。防止过大的Translog影响数据恢复所消耗的时间。一旦达到了这个大小就会触发flush操作，生成一个新的Translog。
- index.translog.sync_interval translog isfsync到磁盘的频率，默认值是5s，不能小于100ms；

ES数据写入过程中是先写入Lucene的缓冲区，然后写入Translog文件。Lucene文件与Translog文件写入流程如下：

如果先写入Translog，然后写Lucene，会出现Lucene写失败的情况下，tanslog需要回滚的情况（如果translog已经刷到磁盘，还需要从磁盘删除）这对于不支持事务的ES来说，显然是不合理的。注意：这一点和MySql等数据库的操作是完全不一样的，数据库的数据写入，一般是先写日志文件再写数据到缓冲区。

每个node节点下会有多个索引，每个索引的shard下存在index目录的Lucene文件和translog目录下的translog文件，即一个shard下的索引文件和translog文件是对应关系。

Translog文件涉及写入的文件保护.ckp文件和.tlog文件。

（1）translog-N.tlog - 真正的日志文件，N表示generation（代）的意思，通过它跟索引文件关联

（2）tranlog.ckp - 日志的元数据文件，对于ChcekPoint类，涉及的字段信息如下：
```
final long offset;
final int numOps;
final long generation;
final long minSeqNo;
final long maxSeqNo;
final long globalCheckpoint;
final long minTranslogGeneration;
final long trimmedAboveSeqNo;

private static final int CURRENT_VERSION = 3; // introduction of trimmed above seq#

private static final String CHECKPOINT_CODEC = "ckp";

// size of 6.4.0 checkpoint
static final int V3_FILE_SIZE = CodecUtil.headerLength(CHECKPOINT_CODEC)
    + Integer.BYTES  // ops
    + Long.BYTES // offset
    + Long.BYTES // generation
    + Long.BYTES // minimum sequence number
    + Long.BYTES // maximum sequence number
    + Long.BYTES // global checkpoint
    + Long.BYTES // minimum translog generation in the translog
    + Long.BYTES // maximum reachable (trimmed) sequence number, introduced in 6.4.0
    + CodecUtil.footerLength();
```
Translog文件的写入是基于TranslogWriter类，读取是基于TranslogReader类。Translog文件写入的入口InternalEngine类的index方法，对于代码如下：
```
public IndexResult index(Index index) throws IOException {
           ...................
            //4. 索引先写入Lucene，写入成功后，再写入translog
            //ElasticSearch在处理写入索引时会先写Lucene，写入Lucene成功后，再写入Translog，否则会抛异常。为什么这么设计呢？
            //写Lucene目的是为了让文档可被搜索，写入时Lucene会对索引文档进行字段级别的校验，万一校验出错，说明该索引文档存在问题，
            // 是不应该被写入的，也就没有写入translog的必要了。
            //写Translog目的是保证数据写入的可靠性，便于数据回溯，恢复。
            //如果先写translog再写lucene，会出现“写translog成功，写lucene失败”的编码场景，会增加异常处理难度。
            if (index.origin().isFromTranslog() == false) {
                // 如果该索引操作不是从translog恢复的（Translog仅仅用于Lucene数据恢复）
                // 如果数据来自日志的话就不再重复记录日志，否则记录日志
                //如果index操作成功，则在translog中添加一个INDEX日志。如果index操作失败，则在translog中添加一个NO_OP日志
                final Translog.Location location;
                if (indexResult.getResultType() == Result.Type.SUCCESS) {
                    // 如果Lucene索引操作成功了，则调用translog.add将该索引操作写入translog日志中
                    location = translog.add(new Translog.Index(index, indexResult));
                } else if (indexResult.getSeqNo() != SequenceNumbers.UNASSIGNED_SEQ_NO) {
                    // if we have document failure, record it as a no-op in the translog and Lucene with the generated seq_no
                    // 写文档失败的情况，则将其作为no-op记录在translog，生成的seq_no记录在Lucene
                    final NoOp noOp = new NoOp(indexResult.getSeqNo(), index.primaryTerm(), index.origin(),
                        index.startTime(), indexResult.getFailure().toString());
                    location = innerNoOp(noOp).getTranslogLocation();
                } else {
                    location = null;
                }
                
                indexResult.setTranslogLocation(location);
            }
            ................................
    }
```
Translog类的核心方法如下：

1.add
add方法向当前的Translog中添加一个操作（Operation），并返回该操作在Translog中的位置信息（Location)。对translog的操作写入，Operation 对应 Delete Index 和 NoOp三个操作。
```
public Location add(final Operation operation) throws IOException {
    //初始化16KB空间, An bytes stream output that allows providing a BigArrays
    final ReleasableBytesStreamOutput out = new ReleasableBytesStreamOutput(bigArrays);
    try {
        final long start = out.position();
        //跳过4字节的位置以便之后写入操作大小(整型占4个字节)
        out.skip(Integer.BYTES);
        /**operation 数据写入*/
        //将操作序列化到缓冲区中。该方法在写入操作后会计算校验和并将校验和写入`out`的当前位置
        writeOperationNoSize(new BufferedChecksumStreamOutput(out), operation);
        final long end = out.position();
        //操作的空间大小
        final int operationSize = (int) (end - Integer.BYTES - start);
        out.seek(start);
        out.writeInt(operationSize);
        out.seek(end);

        //ReleasablePagedBytesReference初始化
        final ReleasablePagedBytesReference bytes = out.bytes();
        //读锁，Translog写入的时，禁止读,防止读取错误
        try (ReleasableLock ignored = readLock.acquire()) {
            //TranslogWriter current
            return current.add(bytes, operation.seqNo());
        }
    }
```
current对于的是TranslogWriter，这点代码逻辑个人觉得是受Lucene的影响。Lucene索引的写入逻辑是IndexWriter，读取文是IndexReader，删除策略是IndexDeletionPolicy。Translog文件写入时TranslogWriter，文件读取时TranslogReader，删除策略是TranslogDeletionPolicy。

### writeOperationNoSize
translog-N.tlog写入的内容源码流程如下，先进行CRC的校验，然后数据写入。
```
public static void writeOperationNoSize(BufferedChecksumStreamOutput out, Translog.Operation op) throws IOException {
    
    //Checksum值重置初始化（CRC32校验）
    out.resetDigest();
    //数据写入
    Translog.Operation.writeOperation(out, op);
    //写入数据总和计算并写入
    long checksum = out.getChecksum();
    out.writeInt((int) checksum);
}
----------------------------------------------------
private void write(final StreamOutput out) throws IOException {
    final int format = out.getVersion().onOrAfter(Version.V_7_0_0) ? SERIALIZATION_FORMAT : FORMAT_6_0;
    out.writeVInt(format);
    out.writeString(id);
    out.writeString(type);
    out.writeBytesReference(source);
    out.writeOptionalString(routing);
    if (format < FORMAT_NO_PARENT) {
         out.writeOptionalString(null); // _parent
    }
    out.writeLong(version);
    if (format < FORMAT_NO_VERSION_TYPE) {
        out.writeByte(VersionType.EXTERNAL.getValue());
    }
    out.writeLong(autoGeneratedIdTimestamp);
    out.writeLong(seqNo);
    out.writeLong(primaryTerm);
}
```
current.add
data 表示写入到 Translog 的操作，seqNo 表示该操作的序列号，序列号用于实现 Elasticsearch 的并发控制机制。
```
public synchronized Translog.Location add(final BytesReference data, final long seqNo) throws IOException {
    ensureOpen();
    final long offset = totalOffset;
    try {
        //writeTo方法对应BytesReference抽象类的writeTo方法, 将操作写入缓冲区
        //ReleasablePagedBytesReference : data    ;;  BufferedChannelOutputStream : outputStream
        data.writeTo(outputStream);
    } catch (final Exception ex) {
        closeWithTragicEvent(ex);
        throw ex;
    }
    //totalOffset表示此文件的总偏移量，包括写入文件以及缓冲区的字节数
    totalOffset += data.length();

    //minSeqNo是初始化的seqNo
    if (minSeqNo == SequenceNumbers.NO_OPS_PERFORMED) {
        assert operationCounter == 0;
    }
    //maxSeqNo是初始化的seqNo
    if (maxSeqNo == SequenceNumbers.NO_OPS_PERFORMED) {
        assert operationCounter == 0;
    }

    //minSeqNo 和 maxSeqNo 分别表示 Translog 文件中所有操作的最小和最大序列号
    minSeqNo = SequenceNumbers.min(minSeqNo, seqNo);
    maxSeqNo = SequenceNumbers.max(maxSeqNo, seqNo);

    //未同步到磁盘的SeqNo集合
    //nonFsyncedSequenceNumbers记录了所有未被持久化到磁盘的操作序列号
    nonFsyncedSequenceNumbers.add(seqNo);
    //operationCounter记录了当前TranslogWriter缓冲区中的操作数量
    operationCounter++;

    //SeqNo冲突检测
    //检查新写入的操作是否与已经缓存的操作序列号发生冲突。如果发生冲突，说明该操作已经被写入到 Translog 中，那么该操作就被丢弃
    assert assertNoSeqNumberConflict(seqNo, data);

    //返回由id，位置及长度确定的操作位置信息。
    return new Translog.Location(generation, offset, data.length());
}
```
rollGeneration
生成一个新的Translog文件，核心是生成一个TranslogWriter对象，其中完成translog-{generation}-.ckp文件复制与重命名。
```
public void rollGeneration() throws IOException {
    // make sure we move most of the data to disk outside of the writeLock
    // in order to reduce the time the lock is held since it's blocking all threads
    sync();
    try (Releasable ignored = writeLock.acquire()) {
        try {
            final TranslogReader reader = current.closeIntoReader();
            readers.add(reader);
            assert Checkpoint.read(location.resolve(CHECKPOINT_FILE_NAME)).generation == current.getGeneration();
            //translog-{generation}-.ckp文件复制与重命名
            copyCheckpointTo(location.resolve(getCommitCheckpointFileName(current.getGeneration())));
            // create a new translog file; this will sync it and update the checkpoint data;
            current = createWriter(current.getGeneration() + 1);
            logger.trace("current translog set to [{}]", current.getGeneration());
        }
```
### syncUpTo
给定一个offset（offset存储在checkpoint文件），把offset之前的数据都从内存sync到磁盘。
```
final boolean syncUpTo(long offset) throws IOException {
    //上次同步的offset小于需要同步的offset
    if (lastSyncedCheckpoint.offset < offset && syncNeeded()) {
        //这里可能存在并发执行，需要加锁
        synchronized (syncLock) { // only one sync/checkpoint should happen concurrently but we wait
            if (lastSyncedCheckpoint.offset < offset && syncNeeded()) {
                final Checkpoint checkpointToSync;
                final LongArrayList flushedSequenceNumbers;
                //强制执行fsync，再加锁
                synchronized (this) {
                    ensureOpen();
                    try {
                        outputStream.flush();
                        checkpointToSync = getCheckpoint();
                        flushedSequenceNumbers = nonFsyncedSequenceNumbers;
                        nonFsyncedSequenceNumbers = new LongArrayList(64);
                    } catch (final Exception ex) {
                }
                try {
                    channel.force(false);
                    //checkPoint文件(.ckp)写入
                    writeCheckpoint(channelFactory, path.getParent(), checkpointToSync);
                } catch (final Exception ex) {
                    closeWithTragicEvent(ex);
                    throw ex;
                }
                //persistedSequenceNumberConsumer：
                //seqNo -> {
                //    final LocalCheckpointTracker tracker = getLocalCheckpointTracker();
                //    assert tracker != null || getTranslog().isOpen() == false;
                //    if (tracker != null) {
                //        tracker.markSeqNoAsPersisted(seqNo);
                //    }
                //});
                flushedSequenceNumbers.forEach((LongProcedure) persistedSequenceNumberConsumer::accept);
                assert lastSyncedCheckpoint.offset <= checkpointToSync.offset :
                    "illegal state: " + lastSyncedCheckpoint.offset + " <= " + checkpointToSync.offset;
                lastSyncedCheckpoint = checkpointToSync; // write protected by syncLock
                return true;
            }
        }
    }
    return false;
}
```