---
layout: post
categories: [LevelDB]
description: none
keywords: LevelDB
---
# LevelDB版本控制Version

## 版本控制
版本Version记录着LevelDB的元数据信息，LevelDB的元信息是按照版本控制的方式来管理的，一个版本Version便是一套完整的元信息记录。

LevelDB为何要引入版本控制？正常来说，当版本A升级到版本B后，版本A就是老版本了，这个时候实际上是可以删掉的。但是，在LevelDB中，版本A有可能还在服务读请求，因此，还不能扔掉，那么就需要同时保留这两个版本A、B，因此就形成了A->B这样的链接关系来记录，所有VersionSet就是利用链表来组织的，其中，Version dummy_versions_是双向链表的头指针，Version* current_是当前最新版本。当某一个Version不再服务读请求之后，这个Version就会从双向链表中移除。


下面我们来解释这些跟版本相关的概念：Version、VersionEdit、VersionSet。
版本Version
```
class Version {
  VersionSet* vset_;  // VersionSet to which this Version belongs
  Version* next_;     // Next version in linked list
  Version* prev_;     // Previous version in linked list
  int refs_;          // Number of live refs to this version
  // List of files per level
  std::vector<FileMetaData*> files_[config::kNumLevels];
  // Next file to compact based on seek stats.
  FileMetaData* file_to_compact_;
  int file_to_compact_level_;
  // Level that should be compacted next and its compaction score.
  // Score < 1 means compaction is not strictly needed.  These fields
  // are initialized by Finalize().
  double compaction_score_;
  int compaction_level_;
};
```
这是一个静态的概念，我们看到Version中的成员变量非常的多，可以分为几个功能：

1）前面4个成员变量与VersionSet相关的Version双向链表：这是将多个Version链接起来的。

2）每个版本Version都记录了它所包含的各个level中的所有文件，即：files_成员变量std::vector<FileMetaData*> files_[config::kNumLevels]，它是一个数组，数组下标就是level，每个level记录的是文件元数据集合，也就是该level所包含的全部文件元数据信息。

3）与文件压缩相关的信息：LevelDB通过合并操作来生成一个新的版本，因此，需要在当前Version中指定如何合并。当前level的一个文件需要与下一级里面的文件合并生成多个新文件，那就要求当前版中需要记录如何生成下一个版本，Version中就是通过最后4个compaction成员变量来记录的。

版本增量VersionEdit
```
class VersionEdit {
 private:
  friend class VersionSet;
  typedef std::set<std::pair<int, uint64_t>> DeletedFileSet;
  std::string comparator_;
  uint64_t log_number_;
  uint64_t prev_log_number_;
  uint64_t next_file_number_;
  SequenceNumber last_sequence_;
  bool has_comparator_;
  bool has_log_number_;
  bool has_prev_log_number_;
  bool has_next_file_number_;
  bool has_last_sequence_;
  std::vector<std::pair<int, InternalKey>> compact_pointers_;
  DeletedFileSet deleted_files_;
  std::vector<std::pair<int, FileMetaData>> new_files_;
};
```
这是一个动态的概念，delta类型，是相对当前版本的增量信息，同样包括了几个功能：

1）与文件序号相关的记录：Version在合并时，新生成的文件会使用统一的序号，所以新文件的序号也会发生变化。这些文件序号变化也需要做为更改的一部分记录下来，各种has_xxx指示符就是说明这个序号是否发生了变化。而序号的更改，并不是累加到展现到Version上，而是直接作用于VersionSet。

2）压缩指针compact_pointers_：由于SSTable文件数量可能庞大，合并操作并不是一次性完成的，而且过多的压缩Compaction也会占用机器CPU资源，因此，压缩是阶段式进行的。压缩指针compact_pointers_记录在版本增量VersionEdit中，并不记录在版本Version中，每个Level下一次合并的位置最终是记录在VersionSet中的compact_pointer_。

每一个Version需要合并的时候，都会经过一定的计算得到每个Level下一次合并的位置，在合并操作Apply()时，直接使用压缩指针compact_pointers_中记录的Level作为索引，因为是版本的Apply，这意味着总是后面的一个版本增量VersionEdit去覆盖前面一个版本增量VersionEdit的合并位置，因此可直接赋值覆盖VersionSet中的compact_pointer_。

3）与文件变化有关的记录：新增文件信息new_files_和删除文件信息deleted_files_，这两个成员变量都是一个std::pair<>对象，表示哪个level新增和删除的文件信息。

那么Version和VersionEdit是什么关系呢？

我们知道数学中有表达式： A + B = C，表达的是两数相加得到另外一个数，这样的思想也体现在编程领域：base + delta = new，在LevelDB中，Version和VersionEdit之间也是这种关系，如果要用LevelDB的语言来描述，那么就会变成Version + VersionEdit = new Version。那么加号和等号是怎么体现的呢？那就是VersionSet::Builder，接下来看一下VersionSet::Builder是如何定义的：
```
class VersionSet::Builder {
 private:
  typedef std::set<FileMetaData*, BySmallestKey> FileSet;
  struct LevelState {
    std::set<uint64_t> deleted_files;
    FileSet* added_files;
  };
  VersionSet* vset_;
  Version* base_;
  LevelState levels_[config::kNumLevels];
};
```
A + B = C用LevelDB的语言来描述，就是如下：
```
Version* v = new Version(this);
{
  Builder builder(this, current_); //把current_作为基础项
  builder.Apply(edit); //累加增量版本edit：current_ + edit
  builder.SaveTo(v); //把结果保存到新版本v：current_ + edit = v
}
```
将增量版本中记录的删除和新增的文件记录到LevelState levels_[config::kNumLevels]中，然后和当前版本Current中记录的已存在版本，一起生成一个新版本Version。

具体如何合并(+)？我们来看builder.Apply(edit)的实现。

1）更新vset_中当前压缩指针：把增量版本中压缩指针的位置记录到vset_中。

2）把增量版本中删除的文件file记录到Builder.levels_的已删除文件中。

3）把增量版本中新增的文件file记录到Builder.levels_的已添加文件中。

代码：
```
// Apply all of the edits in *edit to the current state.
void Apply(VersionEdit* edit) {
  // Update compaction pointers 把增量版本中压缩指针的位置记录到vset_中
  |-for (size_t i = 0; i < edit->compact_pointers_.size(); i++) {
    const int level = edit->compact_pointers_[i].first;
    vset_->compact_pointer_[level] =
        edit->compact_pointers_[i].second.Encode().ToString();
  |-}
  // Delete files 把增量版本中删除的文件file记录到Builder.levels_的已删除文件中
  |-for (const auto& deleted_file_set_kvp : edit->deleted_files_) {
    const int level = deleted_file_set_kvp.first;
    const uint64_t number = deleted_file_set_kvp.second;
    levels_[level].deleted_files.insert(number);
  |-}
  // Add new files 把增量版本中新增的文件file记录到Builder.levels_的已添加文件中
  |-for (size_t i = 0; i < edit->new_files_.size(); i++) {
    const int level = edit->new_files_[i].first;
    FileMetaData* f = new FileMetaData(edit->new_files_[i].second);
    f->refs = 1;
    f->allowed_seeks = static_cast<int>((f->file_size / 16384U)); //为了提高读性能，文件大小除以16K可以得到允许的seek次数，超过这个次数，将触发文件被压缩Compaction。
    if (f->allowed_seeks < 100) f->allowed_seeks = 100;
    levels_[level].deleted_files.erase(f->number);
    levels_[level].added_files->insert(f);
  |-} 
```
等号(=)的处理：我们来看builder.SaveTo(v)的实现

遍历所有的层级level：

1）把base_files和added这两个有序的数组合并到一起。

2）遍历这些新增文件和base文件：看一下file是否是在要删除的列表里面，如果不在，那么添加到version v相应的level的文件列表中。

代码：
```
// Save the current state in *v.
void SaveTo(Version* v) {
    BySmallestKey cmp;//[zll]:这个对象的操作方法operator返回是否较小的key的bool值
    cmp.internal_comparator = &vset_->icmp_;
    for (int level = 0; level < config::kNumLevels; level++) {//[zll]:遍历所有的层级level
      // Merge the set of added files with the set of pre-existing files.
      // Drop any deleted files.  Store the result in *v.
      const std::vector<FileMetaData*>& base_files = base_->files_[level];//[zll]:每一层级中，
当前版本Current里面的已存在文件。
      std::vector<FileMetaData*>::const_iterator base_iter = base_files.begin();
      std::vector<FileMetaData*>::const_iterator base_end = base_files.end();
      const FileSet* added_files = levels_[level].added_files;//[zll]:Builder里面记录的增量版本
的新添加文件状态
      v->files_[level].reserve(base_files.size() + added_files->size());//[zll]:待保存的版本
Version中文件files_扩展为当前版本Current里面已存在的文件+增量版本中新增加的文件。
      for (const auto& added_file : *added_files) {//[zll]:遍历这些新增文件
        // Add all smaller files listed in base_
        for (std::vector<FileMetaData*>::const_iterator bpos =
                 std::upper_bound(base_iter, base_end, added_file, cmp);//[zll]:通过二分
查找，找到第一个不满足cmp规则的文件，即：那些较小的文件
             base_iter != bpos; ++base_iter) {
          MaybeAddFile(v, level, *base_iter);//[zll]:把这些较小的文件追加到待保存版本Version中
        }
        MaybeAddFile(v, level, added_file);//[zll]:把新增版本中的文件追加到待保存版本Version中
      }
      // Add remaining base files
      for (; base_iter != base_end; ++base_iter) {
        MaybeAddFile(v, level, *base_iter);//[zll]:然后把当前版本CurrentVersion中的文件追加
到待保存版本Version中
      }
    }
  }
```
看一下file是否是在要删除的列表里面，如果不在，那么添加到version v相应的level的文件列表中。
```
  void MaybeAddFile(Version* v, int level, FileMetaData* f) {
    if (levels_[level].deleted_files.count(f->number) > 0) {
      // File is deleted: do nothing
    } else {
      std::vector<FileMetaData*>* files = &v->files_[level];
      if (level > 0 && !files->empty()) {
        // Must not overlap
        assert(vset_->icmp_.Compare((*files)[files->size() - 1]->largest,
                                    f->smallest) < 0);
      }
      f->refs++;
      files->push_back(f);
    }
  }
};
```

文件元数据FileMetaData
```
struct FileMetaData {
  FileMetaData() : refs(0), allowed_seeks(1 << 30), file_size(0) {}
  int refs;
  int allowed_seeks;  // Seeks allowed until compaction
  uint64_t number;
  uint64_t file_size;    // File size in bytes
  InternalKey smallest;  // Smallest internal key served by table
  InternalKey largest;   // Largest internal key served by table
};
```
文件的元数据信息比较简单，包含文件号、文件大小、最小key、最大key、允许查找的次数、引用计数。对于查找key来说，最重要的是最小key和最大key，在一个文件中查找key，只要判断这个key是否在这个[smallest, largest]区间，就可以很快判定。

版本集VersionSet
```
class VersionSet {
  Env* const env_;
  const std::string dbname_;
  const Options* const options_;
  TableCache* const table_cache_;
  const InternalKeyComparator icmp_;
  uint64_t next_file_number_;    //下一个manifest文件编号，在manifest日志里
  uint64_t manifest_file_number_; //manifest文件编号，每次重启后递增
  uint64_t last_sequence_; //sequence号，用于snapshot，每次写入操作都会递增
  uint64_t log_number_; //WAL日志文件编号
  uint64_t prev_log_number_;  // 0 or backing store for memtable being compacted
  // Opened lazily
  WritableFile* descriptor_file_;
  log::Writer* descriptor_log_;
  Version dummy_versions_;  // Head of circular doubly-linked list of versions.
  Version* current_;        // == dummy_versions_.prev_
  // Per-level key at which the next compaction at that level should start.
  // Either an empty string, or a valid InternalKey.
  std::string compact_pointer_[config::kNumLevels];
};
```
用于管理所有的Version，当不断有新版本生成的时候，那么就需要不断地Append到VersionSet里面，当旧的Version不再服务读请求之后，这个Version就会从双向链表中移除，Version* current_用于不会被移出。VersionSet同样包括了几个功能：

1）与Compaction压缩信息相关：每个Level下一次合并的位置最终是记录在VersionSet中的compact_pointer_。

2）SSTable文件LRU缓存：TableCache* const table_cache_

3）InternalKey比较器：InternalKeyComparator icmp_

4）记录版本增量VersionEdit合并之后，新文件序号的最新值。

5）MANIFEST文件WritableFile* descriptor_file_

6）WAL日志文件log::Writer* descriptor_log_

7）版本链表：正常来说，当版本A升级到版本B后，版本A就是老版本了，这个时候实际上是可以删掉的。但是，在LevelDB中，版本A有可能还在服务读请求，因此，还不能扔掉，那么就需要同时保留这两个版本A、B，因此就形成了A->B这样的链接关系来记录，所有VersionSet就是利用链表来组织的，其中，Version dummy_versions_是双向链表的头指针，Version* current_是当前最新版本。当某一个Version不再服务读请求之后，这个Version就会从双向链表中移除。

Version结构中的链表指针如下：
```
class Version {
  VersionSet* vset_;  // VersionSet to which this Version belongs
  Version* next_;     // Next version in linked list
  Version* prev_;     // Previous version in linked list
  int refs_;          // Number of live refs to this version
    … …
};
```
注意的是：当前版本current_总是指向双向链表的最后一个元素。

代码：
```
void VersionSet::AppendVersion(Version* v) {
  // Make "v" current
  assert(v->refs_ == 0);
  assert(v != current_);
  if (current_ != nullptr) {
    current_->Unref();
  }
  current_ = v;//当前版本current_指向最新加的这个Version
  v->Ref();
  // Append to linked list 新版本Version追加到链表的末尾
  v->prev_ = dummy_versions_.prev_;
  v->next_ = &dummy_versions_;
  v->prev_->next_ = v;
  v->next_->prev_ = v;
}
```
至此，版本Version、版本增量VersionEdit、版本集VersionSet以及VersionSet::Builder之间的密切又复杂的关系梳理完毕，总体感觉LevelDB对版本的处理还是有点乱的。