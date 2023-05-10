---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---
# Prometheus时序数据库源码
Prometheus的监控数据以指标（metric）的形式保存在内置的时间序列数据库（TSDB）当中。

## Gorilla
Prometheus的存储结构-TSDB是参考了Facebook的Gorilla之后，自行实现的。所以阅读 这篇文章《Gorilla: A Fast, Scalable, In-Memory Time Series Database》 ，可以对Prometheus为何采用这样的存储结构有着清晰的理解。

## 数据模型（DataModel）
Prometheus 的监控数据以指标（metric）的形式保存在内置的时间序列数据库（TSDB）当中。

### 指标名称和标签(metric names, labels)
每一条时间序列由指标名称（Metrics Name）以及一组标签labels（键值KV对）唯一标识。
TIPS: 改变标签中的K或者V值（包括添加或删除），都会创建新的时间序列。

### 样本Samples
在时间序列中的每一个点称为一个样本（sample），样本由以下三部分组成：
- 指标（metric）：指标名称Metric names和标签labels；
- 时间戳（timestamp）：一个精确到毫秒的时间戳；
- 样本值（value）： 一个 folat64 的浮点型数据表示当前样本的值。

如下就是三个样本数据：
```
<--------------------------- metric ------------------------><-----timestamp-----><–value–>
<—metric name–><--------------labels--------------><------timestamp------><–value–>
http_request_total{status=“200”, method=“GET”}@1434417560938 => 94355
http_request_total{status=“200”, method=“GET”}@1434417561287 => 94334
http_request_total{status=“200”, method=“POST”}@1434417561287 => 93656
```
当我们根据时间顺序，把样本数据统一放在一起，就可以形成一条监控曲线

### Series
这样的特定的metric、timestamp、value构成的时间序列(TimeSeries) 在Prometheus中被称作Series。

时间序列 (Time Series) 指的是某个指标随时间变化的所有历史数据，而样本 (Sample) 指的是历史数据中该变量的瞬时值。

## 存储机制
Prometheus将最近的数据保存在内存中，这样查询最近的数据会变得非常快，然后通过一个compactor定时将数据打包到磁盘。数据在内存中最少保留2个小时(storage.tsdb.min-block-duration。为了防止程序崩溃导致数据丢失，实现了WAL（write-ahead-log）机制，启动时会以写入日志(WAL)的方式来实现重播，从而恢复数据。

至于为什么设置2小时这个值，应该是Gorilla那篇论文中观察得出的结论 ! 即压缩率在2小时时候达到最高，如果保留的时间更短，就无法最大化的压缩。

## prometheus/tsdb
prometheus/tsdb 是 prometheus 2.0 版本起使用的底层存储, 它的数据块编码也使用了 facebook 的 gorilla, 并具备了完整的持久化方案

## 项目结构
```
├── Documentation
│   └── format
│       ├── chunks.md
│       ├── index.md
│       └── tombstones.md
├── LICENSE
├── README.md
├── block.go
├── block_test.go
├── chunkenc
│   ├── bstream.go
│   ├── chunk.go
│   ├── chunk_test.go
│   └── xor.go
├── chunks
│   └── chunks.go
├── cmd
│   └── tsdb
│       ├── Makefile
│       ├── README.md
│       └── main.go
├── compact.go
├── compact_test.go
├── db.go
├── db_test.go
├── encoding_helpers.go
├── fileutil
│   ├── dir_unix.go
│   ├── dir_windows.go
│   ├── fileutil.go
│   ├── mmap.go
│   ├── mmap_unix.go
│   ├── mmap_windows.go
│   ├── preallocate.go
│   ├── preallocate_darwin.go
│   ├── preallocate_linux.go
│   ├── preallocate_other.go
│   ├── sync.go
│   ├── sync_darwin.go
│   └── sync_linux.go
├── head.go
├── head_test.go
├── index
│   ├── encoding_helpers.go
│   ├── index.go
│   ├── index_test.go
│   ├── postings.go
│   └── postings_test.go
├── labels
│   ├── labels.go
│   ├── labels_test.go
│   └── selector.go
├── querier.go
├── querier_test.go
├── test
│   ├── conv_test.go
│   ├── hash_test.go
│   └── labels_test.go
├── testdata
│   └── 20kseries.json
├── testutil
│   └── testutil.go
├── tombstones.go
├── tombstones_test.go
├── tsdbutil
│   ├── buffer.go
│   └── buffer_test.go
├── wal.go
└── wal_test.go
```

## 监控数据点
监控数据都是由一个一个数据点组成，所以可以用下面的结构来保存最基本的存储单元
```
type sample struct {
	t int64
	v float64
}
```
同时我们还需要注意到的信息是，我们需要知道这些点属于什么机器的哪种监控。这种信息在Promtheus中就用Label(标签来表示)。一个监控项一般会有多个Label，所以一般用labels []Label。

## 内存序列(memSeries)
接下来，我们看下具体的数据结构

```
type memSeries stuct {
	......
	ref uint64 // 其id
	lst labels.Labels // 对应的标签集合
	chunks []*memChunk // 数据集合
	headChunk *memChunk // 正在被写入的chunk
	......
}
```
其中memChunk是真正保存数据的内存块。我们先来观察下memSeries在内存中的组织。 由此我们可以看到，针对一个最终端的监控项(包含抓取的所有标签，以及新添加的标签,例如ip)，我们都在内存有一个memSeries结构。

## 寻址memSeries
如果通过一堆标签快速找到对应的memSeries。自然的,Prometheus就采用了hash。主要结构体为:

```
type stripeSeries struct {
	series [stripeSize]map[uint64]*memSeries // 记录refId到memSeries的映射
	hashes [stripeSize]seriesHashmap // 记录hash值到memSeries,hash冲突采用拉链法
	locks  [stripeSize]stripeLock // 分段锁
}
type seriesHashmap map[uint64][]*memSeries
```
由于在Prometheus中会频繁的对`map[hash/refId]memSeries`进行操作，例如检查这个labelSet对应的memSeries是否存在，不存在则创建等。由于golang的map非线程安全，所以其采用了分段锁去拆分锁。 而hash值是依据labelSets的值而算出来。

## 数据点的存储
为了让Prometheus在内存和磁盘中保存更大的数据量，势必需要进行压缩。而memChunk在内存中保存的正是采用XOR算法压缩过的数据。在这里，笔者只给出Gorilla论文中的XOR描述更具体的算法在论文中有详细描述。总之，使用了XOR算法后，平均每个数据点能从16bytes压缩到1.37bytes，也就是说所用空间直接降为原来的1/12!

## 内存中的倒排索引
上面讨论的是标签全部给出的查询情况。那么我们怎么快速找到某个或某几个标签(非全部标签)的数据呢。这就需要引入以Label为key的倒排索引。我们先给出一组标签集合

```
{__name__:http_requests}{group:canary}{instance:0}{job:api-server}   
{__name__:http_requests}{group:canary}{instance:1}{job:api-server}
{__name__:http_requests}{group:production}{instance:1}{job,api-server}
{__name__:http_requests}{group:production}{instance:0}{job,api-server}
```
可以看到，由于标签取值不同，我们会有四种不同的memSeries。如果一次性给定4个标签，应该是很容易从map中直接获取出对应的memSeries(尽管Prometheus并没有这么做)。但大部分我们的promql只是给定了部分标签，如何快速的查找符合标签的数据呢？
这就引入倒排索引。 先看一下，上面例子中的memSeries在内存中会有4种，同时内存中还夹杂着其它监控项的series
如果我们想知道job:api-server,group为production在一段时间内所有的http请求数量,那么必须获取标签携带 `({__name__:http_requests}{job:api-server}{group:production})`的所有监控数据。
如果没有倒排索引，那么我们必须遍历内存中所有的memSeries(数万乃至数十万)，一一按照Labels去比对,这显然在性能上是不可接受的。而有了倒排索引，我们就可以通过求交集的手段迅速的获取需要哪些memSeries。
注意，这边倒排索引存储的refId必须是有序的。这样，我们就可以在O(n)复杂度下顺利的算出交集,另外，针对其它请求形式，还有并集/差集的操作,对应实现结构体为:

```
type intersectPostings struct {...}  // 交集
type mergedPostings struct {...} // 并集
type removedPostings struct {...} // 差集
```

倒排索引的插入组织即为Prometheus下面的代码

```
Add(labels,t,v) 
	|->getOrCreateWithID
		|->memPostings.Add
		
// Add a label set to the postings index.
func (p *MemPostings) Add(id uint64, lset labels.Labels) {
	p.mtx.Lock()
	// 将新创建的memSeries refId都加到对应的Label倒排里去
	for _, l := range lset {
		p.addFor(id, l)
	}
	p.addFor(id, allPostingsKey) // allPostingKey "","" every one都加进去

	p.mtx.Unlock()
}
```
### 正则支持

事实上，给定特定的Label:Value还是无法满足我们的需求。我们还需要对标签正则化，例如取出所有ip为1.1.*前缀的http_requests监控数据。为了这种正则，Prometheus还维护了一个标签所有可能的取值。对应代码为:

```
Add(labels,t,v) 
	|->getOrCreateWithID
		|->memPostings.Add
func (h *Head) getOrCreateWithID(id, hash uint64, lset labels.Labels){
	...
	for _, l := range lset {
		valset, ok := h.values[l.Name]
		if !ok {
			valset = stringset{}
			h.values[l.Name] = valset
		}
		// 将可能取值塞入stringset中
		valset.set(l.Value)
		// 符号表的维护
		h.symbols[l.Name] = struct{}{}
		h.symbols[l.Value] = struct{}{}
	}
	...
}
```

那么，在内存中，我们就有了如下的表 图中所示，有了label和对应所有value集合的表，一个正则请求就可以很容的分解为若干个非正则请求并最后求交/并/查集，即可得到最终的结果。

## 磁盘目录结构

##  Block
一个Block就是一个独立的小型数据库，其保存了一段时间内所有查询所用到的信息。包括标签/索引/符号表数据等等。Block的实质就是将一段时间里的内存数据组织成文件形式保存下来。
最近的Block一般是存储了2小时的数据，而较为久远的Block则会通过compactor进行合并，一个Block可能存储了若干小时的信息。值得注意的是，合并操作只是减少了索引的大小(尤其是符号表的合并)，而本身数据(chunks)的大小并没有任何改变。

## ## Chunks结构

## 监控数据的插入
数据是如何插入Prometheus的过程做下阐述。对应方法

```
func (a *headAppender) Add(lset labels.Labels, t int64, v float64) (uint64, error) {
	......
	// 如果lset对应的series没有，则建一个。同时把新建的series放入倒排Posting映射里面
	s, created := a.head.getOrCreate(lset.Hash(), lset) 
	if created { // 如果新创建了一个，则将新建的也放到a.series里面
		a.series = append(a.series, record.RefSeries{
			Ref:    s.ref,
			Labels: lset,
		})
	}
	return s.ref, a.AddFast(s.ref, t, v)
}
```
我们就以下面的add函数调用为例:
```
app.Add(labels.FromStrings("foo", "bar"), 0, 0)
```
首先是getOrCreate,顾名思义，不存在则创建一个。创建的过程包含了seriesHashMap/Postings(倒排索引)/LabelIndex的维护。
然后是AddFast方法

```
func (a *headAppender) AddFast(ref uint64, t int64, v float64) error{
		// 拿出对应的memSeries
		s := a.head.series.getByID(ref)
		......
		// 设置为等待提交状态
		s.pendingCommit=true
		......
		// 为了事务概念，放入temp存储，等待真正commit时候再写入memSeries
		a.samples = append(a.samples, record.RefSample{Ref: ref,T:   t,V:   v,})
		// 
}
```
Prometheus在add数据点的时候并没有直接add到memSeries(也就是query所用到的结构体里),而是加入到一个临时的samples切片里面。同时还将这个数据点对应的memSeries同步增加到另一个sampleSeries里面。

## 事务可见性
为什么要这么做呢？就是为了实现commit语义，只有commit过后数据才可见(能被查询到)。否则，无法见到这些数据。而commit的动作主要就是WAL(Write Ahead Log)以及将headerAppender.samples数据写到其对应的memSeries中。这样，查询就可见这些数据了

## WAL
由于Prometheus最近的数据是保存在内存里面的，未防止服务器宕机丢失数据。其在commit之前先写了日志WAL。等服务重启的时候，再从WAL日志里面获取信息并重放。

为了性能，Prometheus了另一个goroutine去做文件的sync操作，所以并不能保证WAL不丢。进而也不能保证监控数据完全不丢。这点也是监控业务的特性决定的。

写入代码为:

```
commit()
|=>
func (a *headAppender) log() error {
	......
	// 往WAL写入对应的series信息
	if len(a.series) > 0 {
		rec = enc.Series(a.series, buf)
		buf = rec[:0]

		if err := a.head.wal.Log(rec); err != nil {
			return errors.Wrap(err, "log series")
		}
	}
	......
	// 往WAL写入真正的samples
	if len(a.samples) > 0 {
		rec = enc.Samples(a.samples, buf)
		buf = rec[:0]

		if err := a.head.wal.Log(rec); err != nil {
			return errors.Wrap(err, "log samples")
		}
	}
}
```

对应的WAL日志格式为:



#### Series records

```
┌────────────────────────────────────────────┐
│ type = 1 <1b>                              │
├────────────────────────────────────────────┤
│ ┌─────────┬──────────────────────────────┐ │
│ │ id <8b> │ n = len(labels) <uvarint>    │ │
│ ├─────────┴────────────┬─────────────────┤ │
│ │ len(str_1) <uvarint> │ str_1 <bytes>   │ │
│ ├──────────────────────┴─────────────────┤ │
│ │  ...                                   │ │
│ ├───────────────────────┬────────────────┤ │
│ │ len(str_2n) <uvarint> │ str_2n <bytes> │ │
│ └───────────────────────┴────────────────┘ │
│                  . . .                     │
└────────────────────────────────────────────┘
```



#### Sample records

```
┌──────────────────────────────────────────────────────────────────┐
│ type = 2 <1b>                                                    │
├──────────────────────────────────────────────────────────────────┤
│ ┌────────────────────┬───────────────────────────┐               │
│ │ id <8b>            │ timestamp <8b>            │               │
│ └────────────────────┴───────────────────────────┘               │
│ ┌────────────────────┬───────────────────────────┬─────────────┐ │
│ │ id_delta <uvarint> │ timestamp_delta <uvarint> │ value <8b>  │ │
│ └────────────────────┴───────────────────────────┴─────────────┘ │
│                              . . .                               │
└──────────────────────────────────────────────────────────────────┘
```

见Prometheus WAL.md










