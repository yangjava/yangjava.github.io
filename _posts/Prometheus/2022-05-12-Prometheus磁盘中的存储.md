---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---

# Prometheus[时序数据库](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fso.csdn.net%2Fso%2Fsearch%3Fq%3D%E6%97%B6%E5%BA%8F%E6%95%B0%E6%8D%AE%E5%BA%93%26spm%3D1001.2101.3001.7020)-磁盘中的存储结构



## 前言

之前的文章里，笔者详细描述了监控数据在Prometheus内存中的结构。而其在磁盘中的存储结构，也是非常有意思的,关于这部分内容，将在本篇文章进行阐述。



## 磁盘目录结构

首先我们来看Prometheus运行后，所形成的文件目录结构
![img](https://img-blog.csdnimg.cn/img_convert/2f49f2b47a94427ce4aeda75c6b06623.png)
在笔者自己的机器上的具体结构如下:

```
 
```

1.

`prometheus-data`

2.

`|-01EY0EH5JA3ABCB0PXHAPP999D (block)`

3.

`|-01EY0EH5JA3QCQB0PXHAPP999D (block)`

4.

`|-chunks`

5.

`|-000001`

6.

`|-000002`

7.

`.....`

8.

`|-000021`

9.

`|-index`

10.

`|-meta.json`

11.

`|-tombstones`

12.

`|-wal`

13.

`|-chunks_head`



### Block

一个Block就是一个独立的小型数据库，其保存了一段时间内所有查询所用到的信息。包括标签/索引/符号表数据等等。Block的实质就是将一段时间里的内存数据组织成文件形式保存下来。
![img](https://img-blog.csdnimg.cn/img_convert/16cfd571730ded5921ee827a354d5837.png)
最近的Block一般是存储了2小时的数据，而较为久远的Block则会通过compactor进行合并，一个Block可能存储了若干小时的信息。值得注意的是，合并操作只是减少了索引的大小(尤其是符号表的合并)，而本身数据(chunks)的大小并没有任何改变。



#### meta.json

我们可以通过检查meta.json来得到当前Block的一些元信息。

```
 
```

1.

`{`

2.

`"ulid":"01EY0EH5JA3QCQB0PXHAPP999D"`

3.

`*// maxTime-minTime = 7200s => 2 h*`

4.

`"minTime": 1611664000000`

5.

`"maxTime": 1611671200000`

6.

`"stats": {`

7.

`"numSamples": 1505855631,`

8.

`"numSeries": 12063563,`

9.

`"numChunks": 12063563`

10.

`}`

11.

`"compaction":{`

12.

`"level" : 1`

13.

`"sources: [`

14.

`"01EY0EH5JA3QCQB0PXHAPP999D"`

15.

`]`

16.

`}`

17.

`"version":1`

18.

`}`

19.



其中的元信息非常清楚明了。这个Block记录了2个小时的数据。
![img](https://img-blog.csdnimg.cn/img_convert/2e0d61829f9c16801ea65c7f0c8dfc78.png)
让我们再找一个比较陈旧的Block看下它的meta.json.

```
 
```

1.

`"ulid":"01EXTEH5JA3QCQB0PXHAPP999D",`

2.

`*// maxTime - maxTime =>162h*`

3.

`"minTime":1610964800000,`

4.

`"maxTime":1611548000000`

5.

`......`

6.

`"compaction":{`

7.

`"level": 5,`

8.

`"sources: [`

9.

`31个01EX......`

10.

`]`

11.

`},`

12.

`"parents: [`

13.

`{`

14.

`"ulid": 01EXTEH5JA3QCQB1PXHAPP999D`

15.

`...`

16.

`}`

17.

`{`

18.

`"ulid": 01EXTEH6JA3QCQB1PXHAPP999D`

19.

`...`

20.

`}`

21.

`{`

22.

`"ulid": 01EXTEH5JA31CQB1PXHAPP999D`

23.

`...`

24.

`}`

25.

`]`

从中我们可以看到，该Block是由31个原始Block经历5次压缩而来。最后一次压缩的三个Block ulid记录在parents中。如下图所示:
![img](https://img-blog.csdnimg.cn/img_convert/7bc79567d4543911ac7a7398ba92d262.png)



## Chunks结构



### CUT文件切分

所有的Chunk文件在磁盘上都不会大于512M,对应的源码为:

```
 
```

1.

`func (w *Writer) WriteChunks(chks ...Meta) error {`

2.

`......`

3.

`for i, chk := range chks {`

4.

`cutNewBatch := (i != 0) && (batchSize+SegmentHeaderSize > w.segmentSize)`

5.

`......`

6.

`if cutNewBatch {`

7.

`......`

8.

`}`

9.

`......`

10.

`}`

11.

`}`

当写入磁盘单个文件超过512M的时候，就会自动切分一个新的文件。

一个Chunks文件包含了非常多的内存Chunk结构,如下图所示:
![img](https://img-blog.csdnimg.cn/img_convert/c7cc46b6293654e04d7c685c41d7d49c.png)
图中也标出了，我们是怎么寻找对应Chunk的。通过将文件名(000001，前32位)以及(offset,后32位)编码到一个int类型的refId中，使得我们可以轻松的通过这个id获取到对应的chunk数据。



### chunks文件通过mmap去访问

由于chunks文件大小基本固定(最大512M),所以我们很容易的可以通过mmap去访问对应的数据。直接将对应文件的读操作交给操作系统，既省心又省力。对应代码为:

```
 
```

1.

`func NewDirReader(dir string, pool chunkenc.Pool) (*Reader, error) {`

2.

`......`

3.

`for _, fn := range files {`

4.

`f, err := fileutil.OpenMmapFile(fn)`

5.

`......`

6.

`}`

7.

`......`

8.

`bs = append(bs, realByteSlice(f.Bytes()))`

9.

`}`

10.

`通过sgmBytes := s.bs[offset]就直接能获取对应的数据`

![img](https://img-blog.csdnimg.cn/img_convert/ccb698af66aeabb9cc30af2b8b20fd44.png)



## index索引结构

前面介绍完chunk文件，我们就可以开始阐述最复杂的索引结构了。



### 寻址过程

索引就是为了让我们快速的找到想要的内容，为了便于理解。笔者就通过一次数据的寻址来探究Prometheus的磁盘索引结构。考虑查询一个

```
 
```

1.

`拥有系列三个标签`

2.

`({__name__:http_requests}{job:api-server}{instance:0})`

3.

`且时间为start/end的所有序列数据`

我们先从选择Block开始,遍历所有Block的meta.json，找到具体的Block
![img](https://img-blog.csdnimg.cn/img_convert/44b81bc99fbbda939fb17be503332fc5.png)
前文说了，通过Labels找数据是通过倒排索引。我们的倒排索引是保存在index文件里面的。那么怎么在这个单一文件里找到倒排索引的位置呢？这就引入了TOC(Table Of Content)



### TOC(Table Of Content)

![img](https://img-blog.csdnimg.cn/img_convert/f1ee1011bbd1956f988f96e1ed66322d.png)
由于index文件一旦形成之后就不再会改变，所以Prometheus也依旧使用mmap来进行操作。采用mmap读取TOC非常容易:

```
 
```

1.

`func NewTOCFromByteSlice(bs ByteSlice) (*TOC, error) {`

2.

`......`

3.

`*// indexTOCLen = 6\*8+4 = 52*`

4.

`b := bs.Range(bs.Len()-indexTOCLen, bs.Len())`

5.

`......`

6.

`return &TOC{`

7.

`Symbols: d.Be64(),`

8.

`Series: d.Be64(),`

9.

`LabelIndices: d.Be64(),`

10.

`LabelIndicesTable: d.Be64(),`

11.

`Postings: d.Be64(),`

12.

`PostingsTable: d.Be64(),`

13.

`}, nil`

14.

`}`



### Posting offset table 以及 Posting倒排索引

首先我们访问的是Posting offset table。由于倒排索引按照不同的LabelPair(key/value)会有非常多的条目。所以Posing offset table就是决定到底访问哪一条Posting索引。offset就是指的这一Posting条目在文件中的偏移。
![img](https://img-blog.csdnimg.cn/img_convert/35c4376de22dc5708d0e8e2f857184ef.png)



### Series

我们通过三条Postings倒排索引索引取交集得出

```
 
```

1.

`{series1,Series2,Series3,Series4}`

2.

`∩`

3.

`{series1,Series2,Series3}`

4.

`∩`

5.

`{Series2,Series3}`

6.

`=`

7.

`{Series2,Series3}`

也就是要读取Series2和Serie3中的数据，而Posting中的Ref(Series2)和Ref(Series3)即为这两Series在index文件中的偏移。
![img](https://img-blog.csdnimg.cn/img_convert/28edecaa911dfff49ced2809b15584e2.png)
Series以Delta的形式记录了chunkId以及该chunk包含的时间范围。这样就可以很容易过滤出我们需要的chunk,然后再按照chunk文件的访问，即可找到最终的原始数据。



### SymbolTable

值得注意的是，为了尽量减少我们文件的大小，对于Label的Name和Value这些有限的数据，我们会按照字母序存在符号表中。由于是有序的，所以我们可以直接将符号表认为是一个
[]string切片。然后通过切片的下标去获取对应的sting。考虑如下符号表:
![img](https://img-blog.csdnimg.cn/img_convert/e730dda36c6af34790a78dda8437a78e.png)
读取index文件时候，会将SymbolTable全部加载到内存中，并组织成symbols []string这样的切片形式，这样一个Series中的所有标签值即可通过切片下标访问得到。



### Label Index以及Label Table

事实上，前面的介绍已经将一个普通数据寻址的过程全部讲完了。但是index文件中还包含label索引以及label Table，这两个是用来记录一个Label下面所有可能的值而存在的。
这样，在正则的时候就可以非常容易的找到我们需要哪些LabelPair。详情可以见前篇。
![img](https://img-blog.csdnimg.cn/img_convert/bcf521d3a3a3ca6e9e29440481bc9dae.png)

事实上，真正的Label Index比图中要复杂一点。它设计成一条LabelIndex可以表示(多个标签组合)的所有数据。不过在Prometheus代码中只会采用存储一个标签对应所有值的形式。



## 完整的index文件结构

这里直接给出完整的index文件结构，摘自Prometheus中index.md文档。

```
 
```

1.

`┌────────────────────────────┬─────────────────────┐`

2.

`│ magic(0xBAAAD700) <4b> │ version(1) <1 byte> │`

3.

`├────────────────────────────┴─────────────────────┤`

4.

`│ ┌──────────────────────────────────────────────┐ │`

5.

`│ │ Symbol Table │ │`

6.

`│ ├──────────────────────────────────────────────┤ │`

7.

`│ │ Series │ │`

8.

`│ ├──────────────────────────────────────────────┤ │`

9.

`│ │ Label Index 1 │ │`

10.

`│ ├──────────────────────────────────────────────┤ │`

11.

`│ │ ... │ │`

12.

`│ ├──────────────────────────────────────────────┤ │`

13.

`│ │ Label Index N │ │`

14.

`│ ├──────────────────────────────────────────────┤ │`

15.

`│ │ Postings 1 │ │`

16.

`│ ├──────────────────────────────────────────────┤ │`

17.

`│ │ ... │ │`

18.

`│ ├──────────────────────────────────────────────┤ │`

19.

`│ │ Postings N │ │`

20.

`│ ├──────────────────────────────────────────────┤ │`

21.

`│ │ Label Index Table │ │`

22.

`│ ├──────────────────────────────────────────────┤ │`

23.

`│ │ Postings Table │ │`

24.

`│ ├──────────────────────────────────────────────┤ │`

25.

`│ │ TOC │ │`

26.

`│ └──────────────────────────────────────────────┘ │`

27.

`└──────────────────────────────────────────────────┘`



## tombstones

由于Prometheus Block的数据一般在写完后就不会变动。如果要删除部分数据，就只能记录一下删除数据的范围，由下一次compactor组成新block的时候删除。而记录这些信息的文件即是tomstones。



## Prometheus入门书籍推荐



## 总结

Prometheus作为时序数据库，设计了各种文件结构来保存海量的监控数据，同时还兼顾了性能。只有彻底了解其存储结构，才能更好的指导我们应用它！
