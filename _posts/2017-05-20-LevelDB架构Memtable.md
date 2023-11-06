---
layout: post
categories: [LevelDB]
description: none
keywords: LevelDB
---
# LevelDB架构Memtable


## Key结构
Memtable是一个KV存储结构，那么这个key就是个重点了，我们有必要仔细分析一下Memtable对key的使用。在读源码过程中，我发现LevelDB中有5个key的概念，非常容易让人混淆，下面我们就来一个一个的分析下这些key是什么？

### InternalKey & ParsedInternalKey & User Key
InternalKey是一个复合概念，是有几个部分组合成的一个key，ParsedInternalKey就是对InternalKey分拆后的结果，先来看看ParsedInternalKey的定义，这是一个结构体struct：
```
struct ParsedInternalKey {
  Slice user_key;
  SequenceNumber sequence;
  ValueType type;
};
```
也就是说InternalKey是由User key + SequenceNumber + ValueType组合而成的。如下几个Key相关的函数，它们是了解Internal Key和User Key的关键，首先是InternalKey和ParsedInternalKey相互转换的两个函数，如下：
```
ParsedInternalKey(const Slice& u, const SequenceNumber& seq, ValueType t)
      : user_key(u), sequence(seq), type(t) {}


void AppendInternalKey(std::string* result, const ParsedInternalKey& key) {
  result->append(key.user_key.data(), key.user_key.size());
  PutFixed64(result, PackSequenceAndType(key.sequence, key.type));
}
```
函数实现很简单，就是字符串的拼接与把字符串按字节拆分。根据实现，容易得到InternalKey的格式为：
```
| User key (string) | sequence number (7 bytes) | value type (1 byte) |
```
由此还可知道sequence number大小是7 bytes，sequence number是所有基于操作日志系统的关键数据，它唯一指定了不同操作的时间顺序。

User Key就是输入的原始Key，把User Key放到前面的原因是，这样对同一个User Key的操作就可以按照sequence number顺序连续存放了，不同的User Key是互不相干的，因此把它们的操作放在一起也没有什么意义。另外用户可以为User Key定制比较函数，系统默认是字母序的。

下面的两个函数是分别从InternalKey中拆分出User Key和Value Type：

代码：
```
inline Slice ExtractUserKey(const Slice& internal_key)
{
  assert(internal_key.size() >= 8);
  return Slice(internal_key.data(), internal_key.size() - 8);
}


inline ValueType ExtractValueType(const Slice& internal_key)
{
  assert(internal_key.size() >= 8);
  const size_t n = internal_key.size();
  uint64_t num = DecodeFixed64(internal_key.data() + n - 8);
  unsigned char c = num & 0xff;
  return static_cast<ValueType>(c);
}
```
在Memtabl核心的数据结构跳表SkipList中，里面保存的每一个节点Node（table_.Insert(buf)），就是一个key-value，其格式：
```
internal_key_size|internal_key|val_size|value
```
而key是一个internal_key。

### LookupKey & Memtable Key
Memtable的查询接口传入的是LookupKey，它也是由User Key和Sequence Number组合而成的，数据结构如下所示：
```
class LookupKey {
 private:
  // We construct a char array of the form:
  //    klength  varint32               <-- start_
  //    userkey  char[klength]          <-- kstart_
  //    tag      uint64
  //                                    <-- end_
  // The array is a suitable MemTable key.
  // The suffix starting with "userkey" can be used as an InternalKey.
  const char* start_;
  const char* kstart_;
  const char* end_;
  char space_[200];  // Avoid allocation for short keys
};
```
从其构造函数：LookupKey(const Slice& user_key, SequenceNumber s)中，可以分析出LookupKey的格式为：

解释三点：

1）这里的klength是user key长度+8，也就是整个LoopupKey字符串的长度。

2）value type是kValueTypeForSeek，它等于kTypeValue。

static const ValueType kValueTypeForSeek = kTypeValue;
3）由于LookupKey的size是变长存储的，因此它使用kstart_记录了user key string的起始地址，否则将不能正确的获取size和user key；

LookupKey导出了三个函数，可以分别从LookupKey得到Internal Key，Memtable Key和User Key，如下：
```
  // Return a key suitable for lookup in a MemTable.
  Slice memtable_key() const { return Slice(start_, end_ - start_); }
 
  // Return an internal key (suitable for passing to an internal iterator)
  Slice internal_key() const { return Slice(kstart_, end_ - kstart_); }


  // Return the user key
  Slice user_key() const { return Slice(kstart_, end_ - kstart_ - 8); }
```
其中start_是LookupKey字符串的开始，end_是结束，kstart_是start_+4，也就是user key字符串的起始地址。

从memtable_key()可知MemtableKey就是LookupKey，即：memtableKey == lookupKey，而不是internalKey，是klength+internalKey。

internalKey是：| user key (string) | sequence number (7 bytes) | value type (1 byte) |

## Memtable:
在LevelDB整体架构中，我们知道MemTable就是一个在内存中进行数据组织和维护的数据结构，其本质是一个跳表SkipList，绝大多数情况下时间复杂度为O(log n)，这符合LevelDB快速查找Key的需要。在MemTable中，所有的数据按用户定义的排序方法排序后再进行有序存储。其数据结构如下所示：
```
class MemTable {
  typedef SkipList<const char*, KeyComparator> Table;
  KeyComparator comparator_;
  int refs_;
  Arena arena_;
  Table table_;
};
```
其中Table便是跳表SkipList对象，而Arena是Memtable里面的内存分配器，负责向跳表中插入、删除元素是内存的分配和释放。