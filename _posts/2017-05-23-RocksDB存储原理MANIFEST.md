---
layout: post
categories: [SQL]
description: none
keywords: SQL
---
# RocksDB存储原理MANIFEST原理分析

## 概述
在RocksDB中MANIFEST保存了存储引擎的内部的一些状态元数据，简单来说当系统异常重启，或者程序异常被退出之后，RocksDB需要有一种机制能够恢复到一个一致性的状态， 而这个一致性的状态就是靠MANIFEST来保证的.

MANIFEST在RocksDB中是一个单独的文件，而这个文件所保存的数据基本是来自于VersionEdit这个结构.

MANIFEST包含了两个文件，一个log文件一个包含最新MANIFEST文件名的文件，Manifest的log文件名是这样 MANIFEST-(seqnumber),这个seq会一直增长.只有当 超过了指定的大小之后，MANIFEST会刷新一个新的文件,当新的文件刷新到磁盘(并且文件名更新)之后，老的文件会被删除掉.这里可以认为每一次MANIFEST的更新都代表一次snapshot.

下面就是MANIFEST的基本文件组成:
```
MANIFEST = { CURRENT, MANIFEST-<seq-no>* } 
CURRENT = File pointer to the latest manifest log
MANIFEST-<seq no> = Contains snapshot of RocksDB state and subsequent modifications
```
在RocksDB中任意时间存储引擎的状态都会保存为一个Version(也就是SST的集合)，而每次对Version的修改都是一个VersionEdit,而最终这些VersionEdit就是 组成manifest-log文件的内容.

下面就是MANIFEST的log文件的基本构成:
```
version-edit      = Any RocksDB state change
version           = { version-edit* }
manifest-log-file = { version, version-edit* }
                  = { version-edit* }
```
查看MANIFEST的工具

依旧使用ldb工具，不过我用的是mac自动安装的rocksdb_ldb工具
```
bogon:rocksdb-master baron$ rocksdb_ldb manifest_dump --path=./MANIFEST-000001

--------------- Column family "default"  (ID 0) --------------
log number: 13
comparator: <NO COMPARATOR>
--- level 0 --- version# 0 ---
 11:80860[' 
--------------- Column family "__system__"  (ID 1) --------------
log number: 24
comparator: RocksDB_SE_v3.10
--- level 0 --- version# 1 ---
 25:1094[' 
next_file_number 27 last_sequence 190  prev_log_number 0 max_column_family 1

```
创建 及 清除 MANIFEST
整个MANIFEST涉及到三个数据结构分别是VersionEdit/Version/VersionSet,其中前两个上面已经有介绍，而最后一个VersionSet顾名思义表示一堆Version的集合，其实就是 记录了各个版本的信息用来管理整个Version.
```
class VersionSet {
 public:
  VersionSet(const std::string& dbname, const ImmutableDBOptions* db_options,
             const EnvOptions& env_options, Cache* table_cache,
             WriteBufferManager* write_buffer_manager,
             WriteController* write_controller);
  ~VersionSet();
.......................
 private:
  struct ManifestWriter;

  friend class Version;
.................................
  // Opened lazily
  unique_ptr<log::Writer> descriptor_log_;
  // generates a increasing version number for every new version
  uint64_t current_version_number_;

  // Queue of writers to the manifest file
  std::deque<ManifestWriter*> manifest_writers_;
..........................................

```
这里最关键的两个数据结构是descriptor_log_和manifest_writers_,前一个表示了当前manifest-log文件，后一个表示需要写入到manifest-log文件中的内容.

下面就是ManifestWriter的结构，可以看到其中包含了一个VersionEdit的数组，这个数组就是即将要写入到manifest-log文件中的内容.
```
// this is used to batch writes to the manifest file
struct VersionSet::ManifestWriter {
  Status status;
  bool done;
  InstrumentedCondVar cv;
  ColumnFamilyData* cfd;
  const autovector<VersionEdit*>& edit_list;

  explicit ManifestWriter(InstrumentedMutex* mu, ColumnFamilyData* _cfd,
                          const autovector<VersionEdit*>& e)
      : done(false), cv(mu), cfd(_cfd), edit_list(e) {}
};

```
然后我们来看RocksDB如何来创建以及写入文件,下面所有的代码都是包含在VersionSet::LogAndApply这个函数中.

首先在每次LogAndApply的时候都会创建一个新的ManifesWriter加入到manifest_writers_队列中.这里只有当之前保存在队列中 的所有Writer都写入完毕之后才会加入到队列，否则就会等待 （后续会在详细的写流程中说明writer的作用）
```
  // queue our request
  ManifestWriter w(mu, column_family_data, edit_list);
  manifest_writers_.push_back(&w);
  while (!w.done && &w != manifest_writers_.front()) {
    w.cv.Wait();
  }
  if (w.done) {
    return w.status;
  }

```
接下来就是保存对应的数据到batch_edits中(manifest_writers_).
```
autovector<VersionEdit*> batch_edits;
....................................
 if (w.edit_list.front()->IsColumnFamilyManipulation()) {
    // no group commits for column family add or drop
    LogAndApplyCFHelper(w.edit_list.front());
    batch_edits.push_back(w.edit_list.front());
  } else {
    v = new Version(column_family_data, this, current_version_number_++);
........................................................
    for (const auto& writer : manifest_writers_) {
      if (writer->edit_list.front()->IsColumnFamilyManipulation() ||
          writer->cfd->GetID() != column_family_data->GetID()) {
        break;
      }
      last_writer = writer;
      for (const auto& edit : writer->edit_list) {
...........................................
        batch_edits.push_back(edit);
      }
    }
    builder->SaveTo(v->storage_info());
  }

```
然后就是创建新的manifest-log文件的逻辑.这里可以看到要么是第一次进入，要么文件大小大于option对应的值才会创建新的文件
```
if (!descriptor_log_ ||
      manifest_file_size_ > db_options_->max_manifest_file_size) {
    pending_manifest_file_number_ = NewFileNumber();
    batch_edits.back()->SetNextFile(next_file_number_.load());
    new_descriptor_log = true;
  } else {
    pending_manifest_file_number_ = manifest_file_number_;
  }

  if (new_descriptor_log) {
    // if we're writing out new snapshot make sure to persist max column family
    if (column_family_set_->GetMaxColumnFamily() > 0) {
      w.edit_list.front()->SetMaxColumnFamily(
          column_family_set_->GetMaxColumnFamily());
    }
  }

```
如果需要创建新的manifest-log文件，则开始构造对应的文件信息并创建文件.
```
if (new_descriptor_log) {
      // create manifest file
      ROCKS_LOG_INFO(db_options_->info_log, "Creating manifest %" PRIu64 "\n",
                     pending_manifest_file_number_);
      unique_ptr<WritableFile> descriptor_file;
      EnvOptions opt_env_opts = env_->OptimizeForManifestWrite(env_options_);
      s = NewWritableFile(
          env_, DescriptorFileName(dbname_, pending_manifest_file_number_),
          &descriptor_file, opt_env_opts);
      if (s.ok()) {
        descriptor_file->SetPreallocationBlockSize(
            db_options_->manifest_preallocation_size);

        unique_ptr<WritableFileWriter> file_writer(
            new WritableFileWriter(std::move(descriptor_file), opt_env_opts));
        descriptor_log_.reset(
            new log::Writer(std::move(file_writer), 0, false));
        s = WriteSnapshot(descriptor_log_.get());
      }
    }

```
开始写入对应的VersionEdit的record到文件(最后我们会来看这个record的构成),这里看到写入完成后会调用Sync来刷新内容到磁盘,等这些操作都做完之后，则会更新Current文件也就是更新最新的manifest-log文件名到CURRENT文件中.
```
  for (auto& e : batch_edits) {
    std::string record;
    if (!e->EncodeTo(&record)) {
      s = Status::Corruption(
          "Unable to Encode VersionEdit:" + e->DebugString(true));
      break;
    }
    TEST_KILL_RANDOM("VersionSet::LogAndApply:BeforeAddRecord",
                     rocksdb_kill_odds * REDUCE_ODDS2);
    s = descriptor_log_->AddRecord(record);
    if (!s.ok()) {
      break;
    }
  }
  if (s.ok()) {
    s = SyncManifest(env_, db_options_, descriptor_log_->file());
  }
.............................
// If we just created a new descriptor file, install it by writing a
// new CURRENT file that points to it.
if (s.ok() && new_descriptor_log) {
  s = SetCurrentFile(env_, dbname_, pending_manifest_file_number_,
                     db_directory);
}

```
CURRENT文件更新完毕之后，就可以删除老的mainfest文件了.
```
  // Append the old mainfest file to the obsolete_manifests_ list to be deleted
  // by PurgeObsoleteFiles later.
  if (s.ok() && new_descriptor_log) {
    obsolete_manifests_.emplace_back(
        DescriptorFileName("", manifest_file_number_));
  }
```
最后则是更新manifest_writers_队列，唤醒之前阻塞的内容.
```
  // wake up all the waiting writers
  while (true) {
    ManifestWriter* ready = manifest_writers_.front();
    manifest_writers_.pop_front();
    if (ready != &w) {
      ready->status = s;
      ready->done = true;
      ready->cv.Signal();
    }
    if (ready == last_writer) break;
  }
  // Notify new head of write queue
  if (!manifest_writers_.empty()) {
    manifest_writers_.front()->cv.Signal();
  }

```
文件内容
具体的文件格式可以参考官方wikiMANIFEST

比如compaction过程中针对某个sst文件的edit结果会记录到MANIFEST,格式如下：
```
+--------------+-------------+--------------+------------+----------------+--------------+----------------+----------------+
| kNewFile4    | level       | file number  | file size  | smallest_key   | largest_key  | smallest_seqno | largest_seq_no |
+--------------+-------------+--------------+------------+----------------+--------------+----------------+----------------+
|<-- var32  -->|<-- var32 -->|<-- var64  -->|<-  var64 ->|<-- String   -->|<-- String -->|<-- var64    -->|<-- var64    -->|

+--------------+------------------+---------+------+----------------+--------------------+---------+------------+
|  CustomTag1  | Field 1 size n1  | field1  | ...  |  CustomTag(m)  | Field m size n(m)  | field(m)| kTerminate |
+--------------+------------------+---------+------+----------------+--------------------+---------+------------+
<-- var32   -->|<-- var32      -->|<- n1  ->|      |<-- var32   - ->|<--    var32     -->|<- n(m)->|<- var32 -->|

```
通过上面的分析我们可以看到最终是通过VersionEdit::EncodeTo来序列化数据,而VersionEdit主要包含了比如log_number/last_sequence_这些字段，这里还有一个比较关键的信息被序列化了，那就是FileMetaData,也就是SST文件的元信息.
```
struct FileMetaData {
  FileDescriptor fd;
  InternalKey smallest;            // Smallest internal key served by table
  InternalKey largest;             // Largest internal key served by table
  SequenceNumber smallest_seqno;   // The smallest seqno in this file
  SequenceNumber largest_seqno;    // The largest seqno in this file

.........................................
  // File size compensated by deletion entry.
  // This is updated in Version::UpdateAccumulatedStats() first time when the
  // file is created or loaded.  After it is updated (!= 0), it is immutable.
  uint64_t compensated_file_size;
  // These values can mutate, but they can only be read or written from
  // single-threaded LogAndApply thread
  uint64_t num_entries;            // the number of entries.
  uint64_t num_deletions;          // the number of deletion entries.
  uint64_t raw_key_size;           // total uncompressed key size.
  uint64_t raw_value_size;         // total uncompressed value size.

  int refs;  // Reference count

  bool being_compacted;        // Is this file undergoing compaction?
  bool init_stats_from_file;   // true if the data-entry stats of this file
                               // has initialized from file.

  bool marked_for_compaction;  // True if client asked us nicely to compact this
                               // file.
};

```