---
layout: post
categories: [SQL]
description: none
keywords: SQL
---
# RocksDB存储原理WAL

## 前言
Rocksdb作为当下nosql中性能的代表被各个存储组件(mysql,tikv,pmdk,bluestore)作为存储引擎底座，其基于LSM tree的核心存储结构（将随机写通过数据结构转化为顺序写）来提供高性能的写吞吐时保证了读性能。同时大量的并发性配置来降低compaction的影响。且最近社区也推出了key-value分离存储的blobdb，在大value场景的写性能又有了进一步的提升。完善且全面的各种语言的SDK和社区，让rocskdb迅速占领存储引擎的内核区域。

所以为了提升存储引擎的核心开发能力，特此针对rocksdb的核心实现学习研究。

## Rocksdb写流程图
涉及到的几个核心文件：
- WAL 保存当前rocksdb的memtable中的文件信息，当memtable --》 immutable memtable中的数据刷到L0之后即之前的会被删除 — 即于DB目录下的 00012.log
- MANIFEST 保存当前db的状态信息（类似于快照），主要是SST文件的各个版本信息（当sst文件被改动，即会生成对应的versionEdit，并触发sync写manifest文件），用于异常断电后恢复— 即 MANIFEST-000001 的文件
- CURRENT 记录当前最新的manifest文件编号
- Memtable 常驻于内存中，在wal写之后，接受具体的key-value数据。每个memtable大小以及个数都有指定的参数进行控制，write_buffer_size 表示memtable的大小,max_write_buffer_number表示内存中最多可以同时存在多少个memtable的个数
- Immutable memtable ，当memtable被写满之后会生成一个新的memtable继续接受IO，旧的memtable就会变成 immutable memtable ，只读的状态，且开始flush到磁盘的L0。
- SST文件，核心key-value的存储文件。DB目录下的000023.sst形态。

分析IO过程主要是通过rocksdb的几个接口：
```
rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);
rocksdb::Status s = db->Get(rocksdb::ReadOptions(), key1, &pinnable_val);
rocksdb::Status s = db->Put(rocksdb::WriteOptions(), key2, value);
```
详细的接口可以参考basic operations

在介绍详细的写流程之前需要先整体得了解流程图中的各个重要文件的作用，以及其基本的实现过程。

## WAL 原理分析
概述
在RocksDB中每一次数据的更新都会涉及到两个结构，一个是内存中的memtable(后续会刷新到磁盘成为SST),第二个是WAL(WriteAheadLog)。

WAL主要的功能是当RocksDB异常退出后，能够恢复出错前的内存中(memtable)数据,因此RocksDB默认是每次用户写都会刷新数据到WAL. 每次当当前WAL对应的内存数据(memtable)刷新到磁盘之后，都会新建一个WAL.

所有的WAL文件都是保存在WAL目录(options.wal_dir),为了保证数据的状态，所有的WAL文件的名字都是按照顺序的(log_number).

文件格式
WAL文件由一堆变长的record组成，而每个record是由kBlockSize(32k)来分组，比如某一个record大于kBlockSize的话，他就会被切分为多个record（通过type来判断).
```
       +-----+-------------+--+----+----------+------+-- ... ----+
 File  | r0  |        r1   |P | r2 |    r3    |  r4  |           |
       +-----+-------------+--+----+----------+------+-- ... ----+
       <--- kBlockSize ------>|<-- kBlockSize ------>|

  rn = variable size records
  P = Padding

```
record的格式如下:
```
+---------+-----------+-----------+--- ... ---+
|CRC (4B) | Size (2B) | Type (1B) | Payload   |
+---------+-----------+-----------+--- ... ---+

CRC = 32bit hash computed over the payload using CRC
Size = Length of the payload data
Type = Type of record
       (kZeroType, kFullType, kFirstType, kLastType, kMiddleType )
       The type is used to group a bunch of records together to represent
       blocks that are larger than kBlockSize
Payload = Byte stream as long as specified by the payload size

```
最后是WAL的payload的格式,其中是一批操作的集合，从record中可以看出wal的写入是一批一批写入得。
```
// WriteBatch::rep_ :=
//    sequence: fixed64
//    count: fixed32
//    data: record[count]
// record :=
//    kTypeValue varstring varstring
//    kTypeDeletion varstring
//    kTypeSingleDeletion varstring
//    kTypeMerge varstring varstring
//    kTypeColumnFamilyValue varint32 varstring varstring
//    kTypeColumnFamilyDeletion varint32 varstring varstring
//    kTypeColumnFamilySingleDeletion varint32 varstring varstring
//    kTypeColumnFamilyMerge varint32 varstring varstring
//    kTypeBeginPrepareXID varstring
//    kTypeEndPrepareXID
//    kTypeCommitXID varstring
//    kTypeRollbackXID varstring
//    kTypeNoop
// varstring :=
//    len: varint32
//    data: uint8[len]

```
上面的格式中可以看到有一个sequence的值，这个值主要用来表示WAL中操作的时序，这里要注意每次sequence的更新是按照WriteBatch来更新的.
```
Status DBImpl::WriteToWAL(const WriteThread::WriteGroup& write_group,
                          log::Writer* log_writer, uint64_t* log_used,
                          bool need_log_sync, bool need_log_dir_sync,
                          SequenceNumber sequence) {
  Status status;
.........................................
  WriteBatchInternal::SetSequence(merged_batch, sequence);

```
查看WAL的工具
这里我是在mac上直接安装的rocksdb(brew install rocksdb)的工具来打印的,如果是在标准linux操作系统，编译好rocksdb代码之后会有ldb工具，两者是同一个工具
```
bogon:rocksdb-master baron$ rocksdb_ldb dump_wal --walfile=./000285.log --header

Sequence,Count,ByteSize,Physical Offset,Key(s)
1255,1,110,0,PUT(1) : 0x00000006000000000000013C
```
以上打印的是一个reocord，且当前record只有一个操作，如果有多个是一个bactch，那么也会添加到同一个record之中的。

## 创建WAL
首先是一个新的DB被打开的时候会创建一个WAL;
```
Status DB::Open(const DBOptions& db_options, const std::string& dbname,
                const std::vector<ColumnFamilyDescriptor>& column_families,
                std::vector<ColumnFamilyHandle*>* handles, DB** dbptr) {
......................................................................
  s = impl->Recover(column_families);
  if (s.ok()) {
    uint64_t new_log_number = impl->versions_->NewFileNumber();
.............................................
    s = NewWritableFile(
        impl->immutable_db_options_.env,
        LogFileName(impl->immutable_db_options_.wal_dir, new_log_number),
        &lfile, opt_env_options);
................................................

```
第二个情况是当一个CF(column family)被刷新到磁盘之后，也会创建新的WAL,这种情况下创建WAL是用过SwitchMemtable函数. 这个函数主要是用来切换memtable,也就是做flush之前的切换(生成新的memtable,然后把老的刷新到磁盘)
```
Status DBImpl::SwitchMemtable(ColumnFamilyData* cfd, WriteContext* context) {
..................................................
  {
    if (creating_new_log) {
...............................................
      } else {
        s = NewWritableFile(
            env_, LogFileName(immutable_db_options_.wal_dir, new_log_number),
            &lfile, opt_env_opt);
      }
.................................
    }
...............................................
  return s;
}

```
通过上面的两个函数我们可以看到每次新建WAL都会有一个new_log_number,这个值就是对应的WAL的文件名前缀，可以看到每次生成新的log_number， 基本都会调用NewFileNumber函数.这里注意如果option设置了recycle_log_file_num的话，是有可能重用老的log_number的。我们先来看下NewFileNumber函数:
```
uint64_t NewFileNumber() { return next_file_number_.fetch_add(1); }
```
可以看到函数实现很简单，就是每次log_number加一，因此一般来说WAL的文件格式都是类似0000001.log这样子

## 清理WAL
WAL的删除只有当包含在此WAL中的所有的数据都已经被持久化为SST之后(也有可能会延迟删除，因为有时候需要master发送transcation Log到slave来回放). 先来看DBImpl::FIndObsoleteFiles函数,这个函数很长，我们只关注对应的WAL部分，这里逻辑很简单，就是遍历所有的WAL，然后找出log_number小于当前min_log_number的文件然后加入到对应的结构(log_delete_files).
```
if (!alive_log_files_.empty() && !logs_.empty()) {
    uint64_t min_log_number = job_context->log_number;
    size_t num_alive_log_files = alive_log_files_.size();
    // find newly obsoleted log files
    while (alive_log_files_.begin()->number < min_log_number) {
      auto& earliest = *alive_log_files_.begin();
      if (immutable_db_options_.recycle_log_file_num >
          log_recycle_files.size()) {
        ROCKS_LOG_INFO(immutable_db_options_.info_log,
                       "adding log %" PRIu64 " to recycle list\n",
                       earliest.number);
        log_recycle_files.push_back(earliest.number);
      } else {
        job_context->log_delete_files.push_back(earliest.number);
      }
.....................................................................
    }
    while (!logs_.empty() && logs_.front().number < min_log_number) {
      auto& log = logs_.front();
      if (log.getting_synced) {
        log_sync_cv_.Wait();
        // logs_ could have changed while we were waiting.
        continue;
      }
      logs_to_free_.push_back(log.ReleaseWriter());
      {
        InstrumentedMutexLock wl(&log_write_mutex_);
        logs_.pop_front();
      }
    }
    // Current log cannot be obsolete.
    assert(!logs_.empty());
  }

```
这里可以看到有两个核心的数据结构alive_log_files和logs_，他们的区别就是前一个表示有写入的WAL,而后一个则是包括了所有的WAL(比如open一个DB,而没有写入数据，此时也会生成WAL).

最终删除WAL的操作是在DBImpl::DeleteObsoleteFileImpl这个函数,而WAL删除不会单独触发，而是和temp/sst这类文件一起被删除的(PurgeObsoleteFiles).
