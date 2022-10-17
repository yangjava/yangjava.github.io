# lucene

# 一、Lucene简介

## 1.1 Lucene是什么？

- Lucene是Apache基金会jakarta项目组的一个子项目；
- Lucene是一个开放源码的全文检索引擎工具包，**提供了完整的查询引擎和索引引擎，部分语种文本分析引擎**；
- Lucene并不是一个完整的全文检索引擎，仅提供了全文检索引擎架构，但仍可以作为一个工具包结合各类插件为项目**提供部分高性能的全文检索功能**；
- 现在常用的ElasticSearch、Solr等全文搜索引擎均是基于Lucene实现的。

## 1.2 Lucene的使用场景

适用于需要数据索引量不大的场景，当索引量过大时需要使用ES、Solr等全文搜索服务器实现搜索功能。

## 1.3 通过本文你能了解到哪些内容？

- Lucene如此繁杂的索引如何生成并写入，索引中的各个文件又在起着什么样的作用？
- Lucene全文索引如何进行高效搜索？
- Lucene如何优化搜索结果，使用户根据关键词搜索到想要的内容？

本文旨在分享Lucene搜索引擎的源码阅读和功能开发中的经验，Lucene采用**7.3.1**版本。

# 二、Lucene基础工作流程

索引的生成分为两个部分：

\1. 创建阶段：

- 添加文档阶段，通过IndexWriter调用addDocument方法生成正向索引文件；
- 文档添加后，通过flush或merge操作生成倒排索引文件。

\2. 搜索阶段：

- 用户通过查询语句向Lucene发送查询请求；
- 通过IndexSearch下的IndexReader读取索引库内容，获取文档索引；
- 得到搜索结果后，基于搜索算法对结果进行排序后返回。

索引创建及搜索流程如下图所示：

![img](https://static001.geekbang.org/infoq/8b/8bccfe031d2eb2d5e5577f237eb7e76d.png)

# 三、Lucene索引构成

## 3.1 正向索引

Lucene的基础层次结构由**索引、段、文档、域、词**五个部分组成。正向索引的生成即为基于Lucene的基础层次结构一级一级处理文档并分解域存储词的过程。

![img](https://static001.geekbang.org/infoq/9f/9f1990da40aac221261cb7e2fb441a7a.png)

索引文件层级关系如图1所示：

- **索引**：Lucene索引库包含了搜索文本的所有内容，可以通过文件或文件流的方式存储在不同的数据库或文件目录下。
- **段**：一个索引中包含多个段，段与段之间相互独立。由于Lucene进行关键词检索时需要加载索引段进行下一步搜索，如果索引段较多会增加较大的I/O开销，减慢检索速度，因此写入时会通过段合并策略对不同的段进行合并。
- **文档**：Lucene会将文档写入段中，一个段中包含多个文档。
- **域**：一篇文档会包含多种不同的字段，不同的字段保存在不同的域中。
- **词**：Lucene会通过分词器将域中的字符串通过词法分析和语言处理后拆分成词，Lucene通过这些关键词进行全文检索。

## 3.2 倒排索引

Lucene全文索引的核心是基于倒排索引实现的快速索引机制。

倒排索引原理如图2所示，倒排索引简单来说就是基于分析器将文本内容进行分词后，记录每个词出现在哪篇文章中，从而通过用户输入的搜索词查询出包含该词的文章。

![img](https://static001.geekbang.org/infoq/bf/bf8ae4f7631aeb53b97c1c9f7362bd3e.png)

**问题：**上述倒排索引使用时每次都需要将索引词加载到内存中，当文章数量较多，篇幅较长时，索引词可能会占用大量的存储空间，加载到内存后内存损耗较大。

**解决方案**：从Lucene4开始，Lucene采用了FST来减少索引词带来的空间消耗。

FST(Finite StateTransducers)，中文名有限状态机转换器。其主要特点在于以下四点：

- 查找词的时间复杂度为O(len(str))；
- 通过将前缀和后缀分开存储的方式，减少了存放词所需的空间；
- 加载时仅将前缀放入内存索引，后缀词在磁盘中进行存放，减少了内存索引使用空间的损耗；
- FST结构在对PrefixQuery、FuzzyQuery、RegexpQuery等查询条件查询时，查询效率高。

具体存储方式如图3所示：

![img](https://static001.geekbang.org/infoq/f3/f3399da17fa9605ba6da90b798130415.png)

倒排索引相关文件包含.tip、.tim和.doc这三个文件，其中：

- tip：用于保存倒排索引Term的前缀，来快速定位.tim文件中属于这个Field的Term的位置，即上图中的aab、abd、bdc。
- tim：保存了不同前缀对应的相应的Term及相应的倒排表信息，倒排表通过跳表实现快速查找，通过跳表能够跳过一些元素的方式对多条件查询交集、并集、差集之类的集合运算也提高了性能。
- doc：包含了文档号及词频信息，根据倒排表中的内容返回该文件中保存的文本信息。

## 3.3 索引查询及文档搜索过程

Lucene利用倒排索引定位需要查询的文档号，通过文档号搜索出文件后，再利用词权重等信息对文档排序后返回。

- 内存加载tip文件，根据FST匹配到后缀词块在tim文件中的位置；
- 根据查询到的后缀词块位置查询到后缀及倒排表的相关信息；
- 根据tim中查询到的倒排表信息从doc文件中定位出文档号及词频信息，完成搜索；
- 文件定位完成后Lucene将去.fdx文件目录索引及.fdt中根据正向索引查找出目标文件。

文件格式如图4所示：

![img](https://static001.geekbang.org/infoq/c5/c5c021286013b7a8d1b23d35a211e7b6.png)

上文主要讲解Lucene的工作原理，下文将阐述Java中Lucene执行索引、查询等操作的相关代码。

# 四、Lucene的增删改操作

Lucene项目中文本的解析，存储等操作均由IndexWriter类实现，IndexWriter文件主要由Directory和IndexWriterConfig两个类构成，其中：

> **Directory**：用于指定存放索引文件的目录类型。既然要对文本内容进行搜索，自然需要先将这些文本内容及索引信息写入到目录里。Directory是一个抽象类，针对索引的存储允许有多种不同的实现。常见的存储方式一般包括存储有本地（FSDirectory），内存（RAMDirectory）等。
>
> **IndexWriterConfig**：用于指定IndexWriter在文件内容写入时的相关配置，包括OpenMode索引构建模式、Similarity相关性算法等。

IndexWriter具体是如何操作索引的呢？让我们来简单分析一下IndexWriter索引操作的相关源码。

## 4.1. 文档的新增

a. Lucene会为每个文档创建ThreadState对象，对象持有DocumentWriterPerThread来执行文件的增删改操作；

```java
ThreadState getAndLock(Thread requestingThread, DocumentsWriter documentsWriter) {
  ThreadState threadState = null;
  synchronized (this) {
    if (freeList.isEmpty()) {
      // 如果不存在已创建的空闲ThreadState，则新创建一个
      return newThreadState();
    } else {
      // freeList后进先出，仅使用有限的ThreadState操作索引
      threadState = freeList.remove(freeList.size()-1);

      // 优先使用已经初始化过DocumentWriterPerThread的ThreadState，并将其与当前
      // ThreadState换位，将其移到队尾优先使用
      if (threadState.dwpt == null) {
        for(int i=0;i<freeList.size();i++) {
          ThreadState ts = freeList.get(i);
          if (ts.dwpt != null) {
            freeList.set(i, threadState);
            threadState = ts;
            break;
          }
        }
      }
    }
  }
  threadState.lock();
  
  return threadState;
}
```

b. 索引文件的插入：DocumentWriterPerThread调用DefaultIndexChain下的processField来处理文档中的每个域，**processField方法是索引链的核心执行逻辑**。通过用户对每个域设置的不同的FieldType进行相应的索引、分词、存储等操作。FieldType中比较重要的是indexOptions：

- NONE：域信息不会写入倒排表，索引阶段无法通过该域名进行搜索；
- DOCS：文档写入倒排表，但由于不记录词频信息，因此出现多次也仅当一次处理；
- DOCS_AND_FREQS：文档和词频写入倒排表；
- DOCS_AND_FREQS_AND_POSITIONS：文档、词频及位置写入倒排表；
- DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS：文档、词频、位置及偏移写入倒排表。

```java
// 构建倒排表

if (fieldType.indexOptions() != IndexOptions.NONE) {
    fp = getOrAddField(fieldName, fieldType, true);
    boolean first = fp.fieldGen != fieldGen;
    // field具体的索引、分词操作
    fp.invert(field, first);

    if (first) {
      fields[fieldCount++] = fp;
      fp.fieldGen = fieldGen;
    }
} else {
  verifyUnIndexedFieldType(fieldName, fieldType);
}

// 存储该field的storeField
if (fieldType.stored()) {
  if (fp == null) {
    fp = getOrAddField(fieldName, fieldType, false);
  }
  if (fieldType.stored()) {
    String value = field.stringValue();
    if (value != null && value.length() > IndexWriter.MAX_STORED_STRING_LENGTH) {
      throw new IllegalArgumentException("stored field \"" + field.name() + "\" is too large (" + value.length() + " characters) to store");
    }
    try {
      storedFieldsConsumer.writeField(fp.fieldInfo, field);
    } catch (Throwable th) {
      throw AbortingException.wrap(th);
    }
  }
}

// 建立DocValue（通过文档查询文档下包含了哪些词）
DocValuesType dvType = fieldType.docValuesType();
if (dvType == null) {
  throw new NullPointerException("docValuesType must not be null (field: \"" + fieldName + "\")");
}
if (dvType != DocValuesType.NONE) {
  if (fp == null) {
    fp = getOrAddField(fieldName, fieldType, false);
  }
  indexDocValue(fp, dvType, field);
}
if (fieldType.pointDimensionCount() != 0) {
  if (fp == null) {
    fp = getOrAddField(fieldName, fieldType, false);
  }
  indexPoint(fp, field);
}
```

c. 解析Field首先需要构造TokenStream类，用于产生和转换token流，TokenStream有两个重要的派生类Tokenizer和TokenFilter，其中Tokenizer用于通过java.io.Reader类读取字符，产生Token流，然后通过任意数量的TokenFilter来处理这些输入的Token流，具体源码如下：

```java
// invert：对Field进行分词处理首先需要将Field转化为TokenStream
try (TokenStream stream = tokenStream = field.tokenStream(docState.analyzer, tokenStream))
// TokenStream在不同分词器下实现不同，根据不同分词器返回相应的TokenStream
if (tokenStream != null) {
  return tokenStream;
} else if (readerValue() != null) {
  return analyzer.tokenStream(name(), readerValue());
} else if (stringValue() != null) {
  return analyzer.tokenStream(name(), stringValue());
}

public final TokenStream tokenStream(final String fieldName, final Reader reader) {
  // 通过复用策略，如果TokenStreamComponents中已经存在Component则复用。
  TokenStreamComponents components = reuseStrategy.getReusableComponents(this, fieldName);
  final Reader r = initReader(fieldName, reader);
  // 如果Component不存在，则根据分词器创建对应的Components。
  if (components == null) {
    components = createComponents(fieldName);
    reuseStrategy.setReusableComponents(this, fieldName, components);
  }
  // 将java.io.Reader输入流传入Component中。
  components.setReader(r);
  return components.getTokenStream();
}
```

d. 根据IndexWriterConfig中配置的分词器，通过策略模式返回分词器对应的分词组件，针对不同的语言及不同的分词需求，分词组件存在很多不同的实现。

- StopAnalyzer：停用词分词器，用于过滤词汇中特定字符串或单词。
- StandardAnalyzer：标准分词器，能够根据数字、字母等进行分词，支持词表过滤替代StopAnalyzer功能，支持中文简单分词。
- CJKAnalyzer：能够根据中文语言习惯对中文分词提供了比较好的支持。

 以StandardAnalyzer（标准分词器）为例：

```java
// 标准分词器创建Component过程，涵盖了标准分词处理器、Term转化小写、常用词过滤三个功能
protected TokenStreamComponents createComponents(final String fieldName) {
  final StandardTokenizer src = new StandardTokenizer();
  src.setMaxTokenLength(maxTokenLength);
  TokenStream tok = new StandardFilter(src);
  tok = new LowerCaseFilter(tok);
  tok = new StopFilter(tok, stopwords);
  return new TokenStreamComponents(src, tok) {
    @Override
    protected void setReader(final Reader reader) {
      src.setMaxTokenLength(StandardAnalyzer.this.maxTokenLength);
      super.setReader(reader);
    }
  };
}
```

e. 在获取TokenStream之后通过TokenStream中的incrementToken方法分析并获取属性，再通过TermsHashPerField下的add方法构建倒排表，最终将Field的相关数据存储到类型为FreqProxPostingsArray的freqProxPostingsArray中，以及TermVectorsPostingsArray的termVectorsPostingsArray中，构成倒排表;

```java
// 以LowerCaseFilter为例，通过其下的increamentToken将Token中的字符转化为小写
public final boolean incrementToken() throws IOException {
  if (input.incrementToken()) {
    CharacterUtils.toLowerCase(termAtt.buffer(), 0, termAtt.length());
    return true;
  } else
    return false;
}
  try (TokenStream stream = tokenStream = field.tokenStream(docState.analyzer, tokenStream)) {
    // reset TokenStream
    stream.reset();
    invertState.setAttributeSource(stream);
    termsHashPerField.start(field, first);
    // 分析并获取Token属性
    while (stream.incrementToken()) {
      ……
      try {
        // 构建倒排表
        termsHashPerField.add();
      } catch (MaxBytesLengthExceededException e) {
        ……
      } catch (Throwable th) {
        throw AbortingException.wrap(th);
      }
    }
    ……
}
```

## 4.2 文档的删除

a. Lucene下文档的删除，首先将要删除的Term或Query添加到删除队列中；

```java
synchronized long deleteTerms(final Term... terms) throws IOException {
  // TODO why is this synchronized?
  final DocumentsWriterDeleteQueue deleteQueue = this.deleteQueue;
  // 文档删除操作是将删除的词信息添加到删除队列中，根据flush策略进行删除
  long seqNo = deleteQueue.addDelete(terms);
  flushControl.doOnDelete();
  lastSeqNo = Math.max(lastSeqNo, seqNo);
  if (applyAllDeletes(deleteQueue)) {
    seqNo = -seqNo;
  }
  return seqNo;
}
```

b. 根据Flush策略触发删除操作;

```java
private boolean applyAllDeletes(DocumentsWriterDeleteQueue deleteQueue) throws IOException {
  // 判断是否满足删除条件 --> onDelete
  if (flushControl.getAndResetApplyAllDeletes()) {
    if (deleteQueue != null) {
      ticketQueue.addDeletes(deleteQueue);
    }
    // 指定执行删除操作的event
    putEvent(ApplyDeletesEvent.INSTANCE); // apply deletes event forces a purge
    return true;
  }
  return false;
}
public void onDelete(DocumentsWriterFlushControl control, ThreadState state) {
  // 判断并设置是否满足删除条件
  if ((flushOnRAM() && control.getDeleteBytesUsed() > 1024*1024*indexWriterConfig.getRAMBufferSizeMB())) {
    control.setApplyAllDeletes();
    if (infoStream.isEnabled("FP")) {
      infoStream.message("FP", "force apply deletes bytesUsed=" + control.getDeleteBytesUsed() + " vs ramBufferMB=" + indexWriterConfig.getRAMBufferSizeMB());
    }
  }
}
```

## 4.3 文档的更新

文档的更新就是一个先删除后插入的过程，本文就不再做更多赘述。

## 4.4 索引Flush

文档写入到一定数量后，会由某一线程触发IndexWriter的Flush操作，生成段并将内存中的Document信息写到硬盘上。Flush操作目前仅有一种策略：FlushByRamOrCountsPolicy。FlushByRamOrCountsPolicy主要基于两种策略自动执行Flush操作：

- maxBufferedDocs：文档收集到一定数量时触发Flush操作。
- ramBufferSizeMB：文档内容达到限定值时触发Flush操作。

其中 activeBytes() 为dwpt收集的索引所占的内存量，deleteByteUsed为删除的索引量。

```java
@Override
public void onInsert(DocumentsWriterFlushControl control, ThreadState state) {
  // 根据文档数进行Flush
  if (flushOnDocCount()
      && state.dwpt.getNumDocsInRAM() >= indexWriterConfig
          .getMaxBufferedDocs()) {
    // Flush this state by num docs
    control.setFlushPending(state);
  // 根据内存使用量进行Flush
  } else if (flushOnRAM()) {// flush by RAM
    final long limit = (long) (indexWriterConfig.getRAMBufferSizeMB() * 1024.d * 1024.d);
    final long totalRam = control.activeBytes() + control.getDeleteBytesUsed();
    if (totalRam >= limit) {
      if (infoStream.isEnabled("FP")) {
        infoStream.message("FP", "trigger flush: activeBytes=" + control.activeBytes() + " deleteBytes=" + control.getDeleteBytesUsed() + " vs limit=" + limit);
      }
      markLargestWriterPending(control, state, totalRam);
    }
  }
}
```

将内存信息写入索引库。

![img](https://static001.geekbang.org/infoq/b6/b6a0adabd4711a8f23e0a52502309c44.png)

索引的Flush分为主动Flush和自动Flush，根据策略触发的Flush操作为自动Flush，主动Flush的执行与自动Flush有较大区别，关于主动Flush本文暂不多做赘述。需要了解的话可以跳转[链接](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fwww.amazingkoala.com.cn%2FLucene%2FIndex%2Flist_18_7.html)。

## 4.5 索引段Merge

索引Flush时每个dwpt会单独生成一个segment，当segment过多时进行全文检索可能会跨多个segment，产生多次加载的情况，因此需要对过多的segment进行合并。

段合并的执行通过MergeScheduler进行管理。mergeScheduler也包含了多种管理策略，包括NoMergeScheduler、SerialMergeScheduler和ConcurrentMergeScheduler。

1. merge操作首先需要通过updatePendingMerges方法根据段的合并策略查询需要合并的段。段合并策略分为很多种，本文仅介绍两种Lucene默认使用的段合并策略：TieredMergePolicy和LogMergePolicy。

- TieredMergePolicy：先通过OneMerge打分机制对IndexWriter提供的段集进行排序，然后在排序后的段集中选取部分（可能不连续）段来生成一个待合并段集，即非相邻的段文件（Non-adjacent Segment）。
- LogMergePolicy：定长的合并方式，通过maxLevel、LEVEL_LOG_SPAN、levelBottom参数将连续的段分为不同的层级，再通过mergeFactor从每个层级中选取段进行合并。

```java
private synchronized boolean updatePendingMerges(MergePolicy mergePolicy, MergeTrigger trigger, int maxNumSegments)
  throws IOException {

  final MergePolicy.MergeSpecification spec;
  // 查询需要合并的段
  if (maxNumSegments != UNBOUNDED_MAX_MERGE_SEGMENTS) {
    assert trigger == MergeTrigger.EXPLICIT || trigger == MergeTrigger.MERGE_FINISHED :
    "Expected EXPLICT or MERGE_FINISHED as trigger even with maxNumSegments set but was: " + trigger.name();

    spec = mergePolicy.findForcedMerges(segmentInfos, maxNumSegments, Collections.unmodifiableMap(segmentsToMerge), this);
    newMergesFound = spec != null;
    if (newMergesFound) {
      final int numMerges = spec.merges.size();
      for(int i=0;i<numMerges;i++) {
        final MergePolicy.OneMerge merge = spec.merges.get(i);
        merge.maxNumSegments = maxNumSegments;
      }
    }
  } else {
    spec = mergePolicy.findMerges(trigger, segmentInfos, this);
  }
  // 注册所有需要合并的段
  newMergesFound = spec != null;
  if (newMergesFound) {
    final int numMerges = spec.merges.size();
    for(int i=0;i<numMerges;i++) {
      registerMerge(spec.merges.get(i));
    }
  }
  return newMergesFound;
}
```

2）通过ConcurrentMergeScheduler类中的merge方法创建用户合并的线程MergeThread并启动。

```java
@Override
public synchronized void merge(IndexWriter writer, MergeTrigger trigger, boolean newMergesFound) throws IOException {
  ……
  while (true) {
    ……
    // 取出注册的后选段
    OneMerge merge = writer.getNextMerge();
    boolean success = false;
    try {
      // 构建用于合并的线程MergeThread 
      final MergeThread newMergeThread = getMergeThread(writer, merge);
      mergeThreads.add(newMergeThread);

      updateIOThrottle(newMergeThread.merge, newMergeThread.rateLimiter);

      if (verbose()) {
        message("    launch new thread [" + newMergeThread.getName() + "]");
      }
      // 启用线程
      newMergeThread.start();
      updateMergeThreads();

      success = true;
    } finally {
      if (!success) {
        writer.mergeFinish(merge);
      }
    }
  }
}
```

3）通过doMerge方法执行merge操作；

```java
public void merge(MergePolicy.OneMerge merge) throws IOException {
  ……
      try {
        // 用于处理merge前缓存任务及新段相关信息生成
        mergeInit(merge);
        // 执行段之间的merge操作
        mergeMiddle(merge, mergePolicy);
        mergeSuccess(merge);
        success = true;
      } catch (Throwable t) {
        handleMergeException(t, merge);
      } finally {
        // merge完成后的收尾工作
        mergeFinish(merge)
      }
……
}
```

# 五、Lucene搜索功能实现

## 5.1 加载索引库

Lucene想要执行搜索首先需要将索引段加载到内存中，由于加载索引库的操作非常耗时，因此仅有当索引库产生变化时需要重新加载索引库。

![img](https://static001.geekbang.org/infoq/c1/c114e13721c7873c218aa8f3b62f918d.png)

加载索引库分为加载段信息和加载文档信息两个部分：

1）加载段信息：

- 通过segments.gen文件获取段中最大的generation，获取段整体信息；
- 读取.si文件，构造SegmentInfo对象，最后汇总得到SegmentInfos对象。

2）加载文档信息：

- 读取段信息，并从.fnm文件中获取相应的FieldInfo，构造FieldInfos；
- 打开倒排表的相关文件和词典文件；
- 读取索引的统计信息和相关norms信息；
- 读取文档文件。

![img](https://static001.geekbang.org/infoq/05/052917347dd93a817e0cf57cf5dc0542.png)

## 5.2 封装

索引库加载完成后需要IndexReader封装进IndexSearch，IndexSearch通过用户构造的Query语句和指定的Similarity文本相似度算法（默认BM25）返回用户需要的结果。通过IndexSearch.search方法实现搜索功能。

**搜索**：Query包含多种实现，包括BooleanQuery、PhraseQuery、TermQuery、PrefixQuery等多种查询方法，使用者可根据项目需求构造查询语句

**排序**：IndexSearch除了通过Similarity计算文档相关性分值排序外，也提供了BoostQuery的方式让用户指定关键词分值，定制排序。Similarity相关性算法也包含很多种不同的相关性分值计算实现，此处暂不做赘述，读者有需要可自行网上查阅。

# 六、总结

Lucene作为全文索引工具包，为中小型项目提供了强大的全文检索功能支持，但Lucene在使用的过程中存在诸多问题：

- 由于Lucene需要将检索的索引库通过IndexReader读取索引信息并加载到内存中以实现其检索能力，当索引量过大时，会消耗服务部署机器的过多内存。
- 搜索实现比较复杂，需要对每个Field的索引、分词、存储等信息一一设置，使用复杂。
- Lucene不支持集群。

Lucene使用时存在诸多限制，使用起来也不那么方便，当数据量增大时还是尽量选择ElasticSearch等分布式搜索服务器作为搜索功能的实现方案。





## 如何高效地阅读lucene源码？

搜索最主要的解决下面的两个问题——

- 跟查询比配的文档是哪些？
- [比配](https://www.zhihu.com/search?q=比配&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2300524075})的文档顺序是什么，哪一个应该排在最前面，哪一个排在后面？

并且以高效的方式来解决这两个问题。

如果你花个一个小时来解决这个问题，估计就没什么人喜欢用你的搜索功能了。

很多人听过Solor或Elastic Search，而它们建立在Lucene的基础上。所以本系列文章探索探索这个基础的原理和实现。

接下来分几篇文章讲讲搜索的原理，主要围绕Lucene 搜索库源码，可能的话去看看Lucene对应的Rust实现。

网上已经有许多文章介绍搜索和Lucene，我写这些文章主要是使用可以运行和调试的例子，用来加深自己对Lucene的理解。

（目前写了两个系列，一、[CrackingOysters：如何debug crash 系列](https://zhuanlan.zhihu.com/p/369006512), 二、[Rust 系列：正确编程的思考模型](https://zhuanlan.zhihu.com/p/365845688))

Talk is cheap, let's start with code.

我有时候喜欢以调试的方式去研究源码，这样可以专注某一块，而且看着代码运行，更有感觉。

所以要先搭建一个可以debug Lucene代码的环境。后续代码会上传到这个[github repo](https://link.zhihu.com/?target=https%3A//github.com/Celthi/lucene-explorer)

使用的工具是Docker + VSCode + Lucene 8.10 版本 + Java 11.

这个开发环境是根据[CrackingOysters：被VSCode圈粉了!](https://zhuanlan.zhihu.com/p/261470007)创建Ubuntu bionic的image

注意：这个[脚本](https://link.zhihu.com/?target=https%3A//github.com/Celthi/lucene-explorer/blob/main/.devcontainer/prepare-env.sh)安装的Java是带源码和debug symbol，这样有兴趣的时候还可以debug JVM看看（我有一篇debug Java jhsdb来调试JVM的core dump的文章在草稿箱中）
通过VScode进入container以后，我们就可以开始按F5 debug了，

比如下面这个简单的代码，

```java
public class App {
    public static void main(String[] args) throws Exception {
        System.out.println("Hello, World!");

		Searcher searcher = new Searcher();
		searcher.AddDocuments();
		searcher.Search();
		//searcher.Query();
       // testAnalyzer();
    }
}
```

## 开始调试

我们可以一步一步的调试代码，下面是Field Type设置的调用栈，

下面是score计算的调用栈

如果你希望修改Lucene的源码，可以通过下面的命令来编译生成Lucene的jar，

```bash
ant jar 
ant jar-src 
```

## High Level的概念梳理

正如前言所说，搜索要高效地解决那两个问题，那么背后是什么数据结构和算法支撑呢？

这里简短地，以通俗地语言梳理一下，更详细和准确地讲解后续再结合代码补充。

以Lucene为例，一个文档会不会匹配是看这个文档有没有我们要搜索的字符，而Lucene将这个字符的概念进行了更详细和细致的定义。

这个定义就是term。

一个文档会不会匹配，是看它没有我们要搜索的terms。我们搜索的字符串会被当成一个term或者拆分成好多个terms。

当我们添加一个文档的时候，Lucene会先把文档变成好多个terms，然后标记这些terms存在这个文档中。

这个就是我们常说的inverted index(倒排索引）。（我们这里会假设这个文档只有一个field，field后续会提）。

terms，决定了文档是否匹配，解决了第一个问题。第二个问题，哪个文档排在前面，又是怎么实现的呢？

Lucene的方法是，根据某个算法对每个文档计算一个匹配值，匹配值高的排在前面。

算法我们后续会再详细地分析。

后续也会涉及排序怎么排，boost field怎么boost，模糊匹配是怎么模糊匹配，高亮的实现，删除一个文档的具体过程等等。

## 后记

这是第一篇，后续文章会慢慢整理发出，每篇文章会经过多次整理，修正一些错误（我可以预料到随着学习的深入，一开始可能会有理解不到位的地方），并整理到我自己的号里。



# 第一小节 Lucene 常见查询的使用

  从本篇文章开始介绍 Lucene 查询阶段的内容，由于 Lucene 提供了几十种不同方式的查询，但其核心的查询逻辑是一致的，该系列的文章通过 Query 的其中的一个子类 BooleanQuery，同时也是作者在实际业务中最常使用的，来介绍 Lucene 的查询原理。

# 查询方式

  下文中先介绍几种常用的查询方式的简单应用：

- TermQuery
- BooleanQuery
- WildcardQuery
- PrefixQuery
- FuzzyQuery
- RegexpQuery
- PhraseQuery
- TermRangeQuery
- PointRangeQuery

## TermQuery

图 1：

![1.png](https://img.6aiq.com/2020/04/1-68857fc2.png)

  图 1 中的 TermQuery 描述的是，我们想要找出包含**域名（FieldName）为“content”，域值（FieldValue）中包含“a”的域（Field）**的文档。

  该查询方式的 demo 见：[https://github.com/LuXugang/Lucene-7.5.0/blob/master/LuceneDemo/src/main/java/lucene/query/TermQueryTest.java](https://www.6aiq.com/forward?goto=https%3A%2F%2Fgithub.com%2FLuXugang%2FLucene-7.5.0%2Fblob%2Fmaster%2FLuceneDemo%2Fsrc%2Fmain%2Fjava%2Flucene%2Fquery%2FTermQueryTest.java)。

## BooleanQuery

图 2：

![2.png](https://img.6aiq.com/2020/04/2-20485a38.png)

  BooleanQuery 为组合查询，图 2 中给出了最简单的多个 TermQuery 的组合（允许其他查询方式的组合），上图中描述的是，我们期望的文档必须至少（根据 BooleanClause.Occur.SHOULD）满足两个 TermQuery 中的一个，如果都满足，那么打分更高。

  该查询方式的 demo 见：[https://github.com/LuXugang/Lucene-7.5.0/blob/master/LuceneDemo/src/main/java/lucene/query/BooleanQueryTest.java](https://www.6aiq.com/forward?goto=https%3A%2F%2Fgithub.com%2FLuXugang%2FLucene-7.5.0%2Fblob%2Fmaster%2FLuceneDemo%2Fsrc%2Fmain%2Fjava%2Flucene%2Fquery%2FBooleanQueryTest.java)。

## WildcardQuery

  该查询方式为通配符查询，支持两种通配符：

```java
    // 星号通配符 *
    public static final char WILDCARD_STRING = '*';
    // 问号通配符 ?
    public static final char WILDCARD_CHAR = '?';
    // 转义符号（escape character）
    public static final char WILDCARD_ESCAPE = '\\';
```

  星号通配符描述的是匹配零个或多个字符，问号通配符描述的是匹配一个字符，转义符号用来对星号跟问号进行转移，表示这两个作为字符使用，而不是通配符。

图 3：

![3.png](https://img.6aiq.com/2020/04/3-bb6bbb44.png)

  问号通配符的查询：

图 4：

![4.png](https://img.6aiq.com/2020/04/4-7d6355b7.png)

  图 4 中的查询会匹配**文档 3，文档 1**。

  星号通配符的查询：

图 5：

![5.png](https://img.6aiq.com/2020/04/5-3a9dd9e1.png)

  图 4 中的查询会匹配**文档 0、文档 1、文档 2、文档 3**。

  转义符号的使用：

图 6：

![6.png](https://img.6aiq.com/2020/04/6-9dd0912b.png)

  图 4 中的查询会匹配**文档 3**。

  该查询方式的 demo 见：[https://github.com/LuXugang/Lucene-7.5.0/blob/master/LuceneDemo/src/main/java/lucene/query/WildcardQueryTest.java](https://www.6aiq.com/forward?goto=https%3A%2F%2Fgithub.com%2FLuXugang%2FLucene-7.5.0%2Fblob%2Fmaster%2FLuceneDemo%2Fsrc%2Fmain%2Fjava%2Flucene%2Fquery%2FWildcardQueryTest.java)。

## PrefixQuery

  该查询方式为前缀查询：

图 7：

![7.png](https://img.6aiq.com/2020/04/7-42e920ad.png)

  图 7 中的 PrefixQuery 描述的是，我们想要找出包含**域名为“content”，域值的前缀值为"go"的域**的文档。

  以图 3 为例子，图 7 的查询会匹配**文档 0、文档 1**。

  该查询方式的 demo 见：[https://github.com/LuXugang/Lucene-7.5.0/blob/master/LuceneDemo/src/main/java/lucene/query/PrefixQueryTest.java](https://www.6aiq.com/forward?goto=https%3A%2F%2Fgithub.com%2FLuXugang%2FLucene-7.5.0%2Fblob%2Fmaster%2FLuceneDemo%2Fsrc%2Fmain%2Fjava%2Flucene%2Fquery%2FPrefixQueryTest.java)。

## FuzzyQuery

  该查询方式为模糊查询，使用编辑距离来实现模糊匹配，**下面的查询都是以图 3 作为例子**：

图 8：

![8.png](https://img.6aiq.com/2020/04/8-27f35586.png)

  图 8 中的各个参数介绍如下：

- maxEdits：编辑距离的最大编辑值
- prefixLength：模糊匹配到的 term 的至少跟图 8 中的域值"god"有两个相同的前缀值，即 term 的前缀要以"go"开头
- maxExpansions：在 maxEidts 跟 prefixLength 条件下，可能匹配到很多个 term，但是只允许处理最多 20 个 term
- transpositions：该值在本篇文档中不做介绍，需要了解[确定型有穷自动机](https://www.6aiq.com/forward?goto=https%3A%2F%2Fwww.amazingkoala.com.cn%2FLucene%2Fgongjulei%2F2019%2F0417%2F51.html)的知识

  图 8 中的查询会匹配**文档 0、文档 1**。

图 9：

![9.png](https://img.6aiq.com/2020/04/9-6b40c3c4.png)

  图 9 中的方法最终会调用图 8 的构造方法，即 maxExpansions 跟 transpositions 的值会使用默认值：

- maxExpansions：默认值为 50
- transpositions：默认值为 true

  图 9 中的查询会匹配**文档 0、文档 1**。

图 10：

![10.png](https://img.6aiq.com/2020/04/10-09c81ee7.png)

  图 10 中的方法最终会调用图 8 的构造方法，即 prefixLength、maxExpansions 跟 transpositions 的值会使用默认值：

- prefixLength：默认值为 0
- maxExpansions：默认值为 50
- transpositions：默认值为 true

  图 10 中的查询会匹配**文档 0、文档 1、文档 2、文档 3**。

图 11：

![11.png](https://img.6aiq.com/2020/04/11-f42e5434.png)

  图 11 中的方法最终会调用图 8 的构造方法，即 maxEdits、maxEprefixLength、maxExpansions 跟 transpositions 的值会使用默认值：

- maxEdits：默认值为 2
- prefixLength：默认值为 0
- maxExpansions：默认值为 50
- transpositions：默认值为 true

  图 10 中的查询会匹配**文档 0、文档 1、文档 2、文档 3**。

  该查询方式的 demo 见：[https://github.com/LuXugang/Lucene-7.5.0/blob/master/LuceneDemo/src/main/java/lucene/query/FuzzyQueryTest.java](https://www.6aiq.com/forward?goto=https%3A%2F%2Fgithub.com%2FLuXugang%2FLucene-7.5.0%2Fblob%2Fmaster%2FLuceneDemo%2Fsrc%2Fmain%2Fjava%2Flucene%2Fquery%2FFuzzyQueryTest.java)。

## RegexpQuery

  该查询方式为正则表达式查询，使用正则表达式来匹配域的域值：

图 12：

![12.png](https://img.6aiq.com/2020/04/12-5d47ed12.png)

  图 12 中的 RegexpQuery 描述的是，我们想要找出包含**域名（FieldName）为“content”，域值（FieldValue）中包含以"g"开头，以"d"结尾，中间包含零个或多个"o"的域（Field）**的文档。

  图 12 中的查询会匹配**文档 0、文档 1、文档 2**。

  该查询方式的 demo 见：[https://github.com/LuXugang/Lucene-7.5.0/blob/master/LuceneDemo/src/main/java/lucene/query/RegexpQueryTest.java](https://www.6aiq.com/forward?goto=https%3A%2F%2Fgithub.com%2FLuXugang%2FLucene-7.5.0%2Fblob%2Fmaster%2FLuceneDemo%2Fsrc%2Fmain%2Fjava%2Flucene%2Fquery%2FRegexpQueryTest.java)。

## PhraseQuery

图 13：

![13.png](https://img.6aiq.com/2020/04/13-00d3f54f.png)

  该查询方式为短语查询：

图 14：

![14.png](https://img.6aiq.com/2020/04/14-cb2f952c.png)

  图 14 中，我们定义了两个 Term，域值分别为"quick"、"fox"，期望获得的这样文档：文档中必须包含这两个 term，同时两个 term 之间的相对位置为 2 （3 - 1），并且允许编辑距离最大为 1，编辑距离用来**调整**两个 term 的相对位置（必须满足）。

  故根据图 13 的例子，图 14 中的查询会匹配**文档 0、文档 1、文档 2**。

图 15：

![15.png](https://img.6aiq.com/2020/04/15-c7407734.png)

  图 15 中，我们另编辑距离为 0，那么改查询只会匹配**文档 0、文档 1**。

图 16：

![16.png](https://img.6aiq.com/2020/04/16-b49b6b5b.png)

  图 16 中，我们另编辑距离为 4，此时查询会匹配**文档 0、文档 1、文档 2、文档 3**。

  这里简单说下短语查询的匹配逻辑：

- 步骤一：找出同时包含"quick"跟"fox"的文档
- 步骤二：计算"quick"跟"fox"之间的相对位置能否在经过编辑距离调整后达到查询的条件

  该查询方式的 demo 见：[https://github.com/LuXugang/Lucene-7.5.0/blob/master/LuceneDemo/src/main/java/lucene/query/PhraseQueryTest.java](https://www.6aiq.com/forward?goto=https%3A%2F%2Fgithub.com%2FLuXugang%2FLucene-7.5.0%2Fblob%2Fmaster%2FLuceneDemo%2Fsrc%2Fmain%2Fjava%2Flucene%2Fquery%2FPhraseQueryTest.java)。

## TermRangeQuery

图 17：

![17.png](https://img.6aiq.com/2020/04/17-e8dd37e2.png)

  该查询方式为范围查询：

图 18：

![18.png](https://img.6aiq.com/2020/04/18-22921d95.png)

  图 18 中的查询会匹配**文档 1、文档 2、文档 3**。

  在后面的文章中会详细介绍 TermRangeQuery，对这个查询方法感兴趣的同学可以先看 [Automaton](https://www.6aiq.com/forward?goto=https%3A%2F%2Fwww.amazingkoala.com.cn%2FLucene%2Fgongjulei%2F2019%2F0417%2F51.html)，它通过确定型有穷自动机的机制来找到查询条件范围内的所有 term。

  该查询方式的 demo 见：[https://github.com/LuXugang/Lucene-7.5.0/blob/master/LuceneDemo/src/main/java/lucene/query/TermRangeQueryTest.java](https://www.6aiq.com/forward?goto=https%3A%2F%2Fgithub.com%2FLuXugang%2FLucene-7.5.0%2Fblob%2Fmaster%2FLuceneDemo%2Fsrc%2Fmain%2Fjava%2Flucene%2Fquery%2FTermRangeQueryTest.java)。

## PointRangeQuery

图 19：

![19.png](https://img.6aiq.com/2020/04/19-7d4ca1fe.png)

  该查询方式为域值是数值类型的范围查询（多维度查询）：

图 20：

![20.png](https://img.6aiq.com/2020/04/20-2d54386a.png)

  PointRangeQuery 用来实现多维度查询，在图 19 中，文档 0 中域名为"coordinate"，域值为"2, 8"的 IntPoint 域，可以把该域的域值看做是直角坐标系中一个 x 轴值为 2，y 轴值为 8 的一个坐标点。

  故文档 1 中域名为"coordinate"的域，它的域值的个数描述的是维度的维数值。

  在图 20 中，lowValue 描述的是 x 轴的值在[1, 5]的区间，upValue 描述的 y 轴的值在[4, 7]的区间，我们期望找出由 lowValue 和 upValue 组成的一个矩形内的点对应的文档。

图 21：

![21.png](https://img.6aiq.com/2020/04/21-7190abf7.png)

  图 21 中红框描述的是 lowValue 跟 upValue 组成的矩形。

  故图 20 中的查询会匹配**文档 1**。



- 本文地址：[Lucene 源码系列——查询原理（上）](https://www.6aiq.com/article/1586725343175)
- 本文版权归作者和[AIQ](https://www.6aiq.com/)共有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出



  在后面的文章中会详细介绍 PointRangeQuery 的查询过程，对这个查询方法感兴趣的同学可以先看 [Bkd-Tree](https://www.6aiq.com/forward?goto=https%3A%2F%2Fwww.amazingkoala.com.cn%2FLucene%2Fgongjulei%2F2019%2F0422%2F52.html) 以及[索引文件之 dim&&dii](https://www.6aiq.com/forward?goto=https%3A%2F%2Fwww.amazingkoala.com.cn%2FLucene%2Fsuoyinwenjian%2F2019%2F0424%2F53.html)，这两篇文章介绍了在索引阶段如何存储数值类型的索引信息。

  该查询方式的 demo 见：[https://github.com/LuXugang/Lucene-7.5.0/blob/master/LuceneDemo/src/main/java/lucene/query/PointRangeQueryTest.java](https://www.6aiq.com/forward?goto=https%3A%2F%2Fgithub.com%2FLuXugang%2FLucene-7.5.0%2Fblob%2Fmaster%2FLuceneDemo%2Fsrc%2Fmain%2Fjava%2Flucene%2Fquery%2FPointRangeQueryTest.java)。

# 第二小节

  在第一小节的文章中，我们介绍了几种常用查询方式的使用方法，从本篇文章开始，通过 BooleanQuery 来介绍查询原理。

# 查询原理流程图

图 1：
![1.png](https://img.6aiq.com/2020/04/1-dcefe20a.png)

## 执行 IndexSearcher 的 search()方法

  根据用户提供的不同参数 IndexSearcher 类提供了多种 search( )方法：

- 分页参数：当用户提供了上一次查询的 ScoreDoc 对象就可以实现分页查询的功能，该内容的实现方式已经在 [Collector（二）](https://www.6aiq.com/forward?goto=https%3A%2F%2Fwww.amazingkoala.com.cn%2FLucene%2FSearch%2F2019%2F0813%2F83.html)中介绍，不赘述
- 排序参数：用户通过提供 `Sort` 参数使得查询结果按照自定义的规则进行排序，默认使用 [TopFieldCollector](https://www.6aiq.com/article/1586704007975) 对满足查询条件的结果进行排序。
- Collector 参数：用户通过提供 Collector 收集器来自定义处理那些满足查询条件的文档的方法，如果不提供，那么默认使用 [TopFieldCollector](https://www.6aiq.com/article/1586704007975) 或者 [TopScoreDocCollector](https://www.6aiq.com/article/1586704007975)，如果用户提供了 Sort 参数，那么使用前者，反之使用后者
- TopN 参数：用户通过提供该参数用来描述期望获得的查询结果的最大个数

  至于使用上述不同的参数对应哪些不同的 search( )就不详细展开了，感兴趣的同学可以结合上述的参数介绍及源码中的几个 [search( )](https://www.6aiq.com/forward?goto=https%3A%2F%2Fgithub.com%2FLuXugang%2FLucene-7.5.0%2Fblob%2Fmaster%2Fsolr-7.5.0%2Flucene%2Fcore%2Fsrc%2Fjava%2Forg%2Fapache%2Flucene%2Fsearch%2FIndexSearcher.java)方法，相信很容易能理解。

## 重写 Query

  每一种 Query 都需要重写（rewrite）才能使得较为友好的接口层面（API level）的 Query **完善**成一个"最终的（final）"Query，如果这个 Query 已经是"最终的"，就不需要重写，这个最终的 Query 在源码注释中被称为 primitive query。

  图 2 中定义了一个 TermQuery（接口层面（API level）的 Query），它**显示的（explicit）**描述了满足查询条件的文档必须包含一个域名（FieldName）为"content"，域值（FIeldValue）中包含"a"的 term（对域值分词后的每一个结果称为 term）的域，我们称这种 TermQuery 是"最终"的 Query。在 TermQuery 中，我们能显示的知道，查询的 term 为"a"。

图 2：

![2.png](https://img.6aiq.com/2020/04/2-36392735.png)

  图 3 中定义了一个 PrefixQuery（前缀查询，见第一小节），它描述了满足条件的文档必须包含一个域名为"content"，域值中包含前缀为"go"的 term 的域，相比较 TermQuery，这个查询没有**显示的** 在用户使用的接口层面（API level）描述我们要查询具体哪个 term，我们称之为这不是一个“最终”的 Query，故需要通过重写 Query 来完善成一个新的 Query，先找到以"go"为前缀的所有 term 集合，然后根据这些 term 重新生成一个 Query 对象，具体过程在下文中展开。

图 3：

![3.png](https://img.6aiq.com/2020/04/3-0b5ff91c.png)

  **注意的是上述的介绍只是描述了重写 Query 的其中一个目的。**

  根据第一小节中介绍的 9 种 Query，我们分别来讲解这些 Query 的重写过程。

### TermQuery

  TermQuery 不需要重写。

### PointRangeQuery

  数值类型的查询，它没有重写的必要。

### BooleanQuery

  BooleanQuery 的重写过程在 [BooleanQuery](https://www.6aiq.com/article/1586423952263) 的文章中介绍，不赘述。

### PhraseQuery

  PhraseQuery 的重写会生成以下两种新的 Query：

- TermQuery：图 4 中的 PhraseQuery，它只有一个域名为“content”，域值为"quick"的 term，这种 PhraseQuery 可以被重写为 TermQuery，TermQuery 是所有的查询性能最好的查询方式（性能好到 Lucene 认为这种查询方式都不需要使用缓存机制，见 [LRUQueryCache](https://www.6aiq.com/article/1586695917701)，可见这次的重写是一个完善的过程

图 4：

![4.png](https://img.6aiq.com/2020/04/4-e789e59b.png)

- PhraseQuery：图 5 中的 PhraseQuery 跟图 6 中的 PhraseQuery，他们的查询结果实际是一致的，因为对于图 5 的 PhraseQuery，它会在重写 PhraseQuery 后变成图 6 中的 PhraseQuery，也就是这种查询方式只关心 term 之间的相对位置，对于图 5 中的 PhraseQuery，在重写的过程中，"quick"的 position 参数会被改为 0，"fox"的 position 参数会被改为 2，由于本篇文章只是描述 PhraseQuery 的重写过程，对于为什么要做出这样的重写逻辑，在后面的文章中会展开介绍

图 5：

![5.png](https://img.6aiq.com/2020/04/5-d27aa10f.png)

图 6：

![6.png](https://img.6aiq.com/2020/04/6-4f632627.png)

### FuzzyQuery、WildcardQuery、PrefixQuery、RegexpQuery、TermRangeQuery

  这几种 Query 的重写逻辑是一致的，在重写的过程中，找到所有的 term，每一个生成对应的 TermQuery，并用 BooleanQuery 封装。

  他们的差异在于不同的 Query 还会对 BooleanQuery 进行再次封装，不过这不是我们本篇文章关心的。

  下面用一个例子来说明上面的描述：

图 7：

![7.png](https://img.6aiq.com/2020/04/7-e0142c4e.png)

图 8：

![8.png](https://img.6aiq.com/2020/04/8-024812f0.png)

  图 8 中我们使用 TermRangeQuery 对图 7 中的内容进行查询。

图 9：

![9.png](https://img.6aiq.com/2020/04/9-18ffdaad.png)

  图 9 中我们使用 BooleanQuery 对图 7 中的内容进行查询。

  图 8 中 TermRangeQuery 在重写的过程中，会先找到"bc" ~ "gc"之间的所有 term（查找方式见 [Automaton](https://www.6aiq.com/forward?goto=https%3A%2F%2Fwww.amazingkoala.com.cn%2FLucene%2Fgongjulei%2F2019%2F0417%2F51.html)），这些 term 即"bcd"、"ga"、"gc"，然后将他们生成对应的 TermQuery，并用 BooleanQuery 进行封装，所以图 8 中的 TermRangeQuery 相当于图 9 中的 BooleanQuery。

  不得不提的是，TermRangeQuery 最终重写后的 Query 对象不仅仅如此，生成 BooleanQuery 只是其中最重要，最关键的一步，本篇文章中我们只需要了解到这个程度即可，因为在后面的文章会详细介绍 TermRangeQuery。

  所有的 Query 在查询的过程中都会执行该流程点，但不是重写 Query 唯一执行的地方，在构建 Weight 的过程中，可能还会执行重写 Query 的操作。

## 生成 Weight

  不同的 Query 生成 Weight 的逻辑各不相同，由于无法介绍所有的情况，故挑选了最最常用的一个查询 BooleanQuery 来作介绍。

图 10：

![10.png](https://img.6aiq.com/2020/04/10-96f0b588.png)

图 11：

![11.png](https://img.6aiq.com/2020/04/11-447add82.png)

  图 10 跟图 11 分别是索引阶段跟查询阶段的内容，我们在查询阶段定义了一个 BooleanQuery，封装了 3 个 TermQuery，该查询条件描述的是：我们期望获得的文档中至少包含三个 term，"h"、"f"、"a"中的一个。

### BooleanWeight

  对于上述的例子中，该 BooleanQuery 生成的 Weight 对象如下所示：

图 12：

![12.png](https://img.6aiq.com/2020/04/12-15194f96.png)

  BooleanQuery 生成的 Weight 对象即 BooleanWeight 对象，它由三个 TermWeight 对象组合而成，TermWeight 即图 11 中封装的三个 **TermQuery** 对应生成的 Weight 对象。

### TermWeight

  图 13 中列出了 TermWeight 中至少包含的几个关键对象：

图 13：

![13.png](https://img.6aiq.com/2020/04/13-230246f2.png)

#### Similarity

  Similarity 描述的是当前查询使用的文档打分规则，当前 Lucene7.5.0 中默认使用 BM25Similarity。用户可以使用自定义的打分规则，可以在构造 IndexSearcher 后，执行 IndexSearcher 的 search()方法前，调用 [IndexSearcher.setSimilarity(Similarity)](https://www.6aiq.com/forward?goto=https%3A%2F%2Fgithub.com%2FLuXugang%2FLucene-7.5.0%2Fblob%2Fmaster%2Fsolr-7.5.0%2Flucene%2Fcore%2Fsrc%2Fjava%2Forg%2Fapache%2Flucene%2Fsearch%2FIndexSearcher.java)的方法设置。Lucene 的文档打分规则在后面的文章中会展开介绍。

#### SimWeight

图 14：

![14.png](https://img.6aiq.com/2020/04/14-734f1e28.png)

  图 14 中描述的是 SimWeight 中包含的几个重要的信息，这些信息在后面的流程中用来作为文档打分的参数，由于 SimWeight 是一个抽象类，在使用 BM25Similarity 的情况下，SimWeight 类的具体实现是 BM25Stats 类。

  我们以下图中红框标识的 TermQuery 为例子来介绍 SimWeight 中的各个信息

图 15：

![15.png](https://img.6aiq.com/2020/04/15-77ccec49.png)

##### field

  该值描述的是 TermQuery 中的域名，在图 15 中，field 的值即“content”。

##### idf

  idf 即逆文档频率因子，它的计算公式如下：

```java
    (float) Math.log(1 + (docCount - docFreq + 0.5D)/(docFreq + 0.5D))
```

- docCount：该值描述是包含域名为“content”的域的文档的数量，从图 10 中可以看出，文档 0~文档 9 都包含，故 docCount 的值为 10
- docFreq：该值描述的是包含域值"h"的文档的数量，从图 10 中可以看出，只有文档 0、文档 8 包含，故 docFreq 的值为 2
- 0.5D：平滑值

##### boost

  该值描述的是查询权值（即图 17 中打分公式的第三部分），boost 值越高，通过该查询获得的文档的打分会更高。

  默认情况下 boost 的值为 1，如果我们期望查询返回的文档尽量是通过某个查询获得的，那么我们就可以在查询（搜索）阶段指定这个查询的权重，如下图所示：

图 16：

![16.png](https://img.6aiq.com/2020/04/16-f919d292.png)

  相比较图 15，在图 16 中，我们使用 BoostQuery 封装了 TermQuery，并显示的指定这个查询的 boost 值为 100。

  图 16 中的查询条件表达了这么一个意愿：我们更期待在执行搜索后，能获得包含"h"的文档。

##### avgdl

  avgdl（average document length，即图 17 中打分公式第二部分中参数 K 中的 avgdl 变量）描述的是平均每篇文档（一个段中的文档）的长度，并且使用域名为"content"的 term 的个数来描述平均每篇文档的长度。

  例如图 7 中的文档 3，在使用空格分词器（WhitespaceAnalyzer）的情况下，域名为"content"，域值为"a c e"的域，在分词后，文档 3 中就包含了 3 个域名为"content"的 term，这三个 term 分别是"a"、"c"、"e"。

  avgdl 的计算公式如下：

```java
    (float) (sumTotalTermFreq / (double) docCount)
```

- sumTotalTermFreq：域名为"content"的 term 的总数，图 7 中，文档 0 中有 1 个、文档 1 中有 1 个，文档 2 中有 2 个，文档 3 中有 3 个，文档 4 中有 1 个，文档 5 中有 2 个，文档 6 中有 3 个，文档 7 中有 1 个，文档 8 中有 8 个，文档 9 中有 6 个，故 sumTotalTermFreq 的值为（1 + 1 + 2 + 3 + 1 + 2 + 3 + 1 + 8 + 6）= 28
- docCount：同 idf 中的 docCount，不赘述，该值为 10

##### cache

  cache 是一个数组，数组中的元素会作为 BM25Similarity 打分公式中的一个参数 K（图 17 打分公式第二部分的参数 K），具体 cache 的含义会在介绍 BM25Similarity 的文章中展开，在这里我们只需要了解 cache 这个数组是在生成 Weight 时生成的。

##### weight

  该值计算公式如下：

```java
    idf * boost
```

  图 17 是 BM25Similarity 的打分公式，它由三部分组成，在 Lucene 的实现中，第一部分即 idf，第三部分即 boost，**至此我们发现，在生成 Weight 的阶段，除了文档号跟 term 在文档中的词频这两个参数，我们已经获得了计算文档分数的其他条件**，至于为什么需要文档号，不是本篇文章关系的部分，再介绍打分公式的文章中会展开介绍，另外 idf 跟 boost 即 SimWeight 中的信息，不赘述。

图 17：

![17.png](https://img.6aiq.com/2020/04/17-117833f1.png)

  **图 17 源自于 << 这就是搜索引擎 >>，作者：张俊林。**

#### TermContext

图 18：

![18.png](https://img.6aiq.com/2020/04/18-c70ce285.png)

  图 18 中描述的是 TermContext 中包含的几个重要的信息，**其中红框标注表示生成 Weight 阶段需要用到的值**，这些信息通过读取索引文件[.tip、tim](https://www.6aiq.com/forward?goto=https%3A%2F%2Fwww.amazingkoala.com.cn%2FLucene%2Fsuoyinwenjian%2F2019%2F0401%2F43.html) 中的内容获得，其读取过程不再这里赘述（因为太复杂~），不过会在以后的文章中介绍，而每个变量的含义都在[.tip、tim](https://www.6aiq.com/forward?goto=https%3A%2F%2Fwww.amazingkoala.com.cn%2FLucene%2Fsuoyinwenjian%2F2019%2F0401%2F43.html) 中详细介绍了，不赘述。

  图 19 中是索引文件。tim 的数据结构：

图 19：

![19.png](https://img.6aiq.com/2020/04/19-1b65497a.png)

  **上文中，计算 idf（DocCount、DocFreq）跟 avgdl（SumTotalTemFreq、DocCount）需要用到的信息在图 19 中用红框标注。**

  最后给出完整的 BooleanWeight 包含的主要信息：

图 20：

![20.png](https://img.6aiq.com/2020/04/20-be71f827.png)

### 关于 Weight 的一些其他介绍

  生成 Weight 的目的是为了不更改 Query 的属性，使得 Query 可以复用。

  从 Weight 包含的主要信息可以看出，生成这些信息的目的就是为了文档打分，那如果我们不关心文档的打分，生成 Weight 的过程又是如何呢？

  这个问题包含了两个子问题：

  **问题一：如何设置不对文档进行打分：**

- 我们在执行 IndexSearcher 的 search()方法时，需要提供自定义的 Collector，并且通过 [Collector.needsScores( )](https://www.6aiq.com/article/1586704007975)来设置为不对文档进行打分

  **问题二：生成的 Weight 有什么不同：**

- 由于不需要对文档进行打分，所以不需要用到 TermContext，即 TermContext 为 null，同时也不需要 SimWeight，这两个信息都是为文档打分准备的
- 如果设置了查询缓存（queryCache，默认开启），那么在不对文档打分的前提下，我们还可以使用查询缓存机制，当然使用缓存机制的前提是有要求的，感兴趣可以看 [LRUQueryCache](https://www.6aiq.com/article/1586695917701)

## 小结

  基于篇幅，本篇只介绍了图 1 中的三个流程点 `执行IndexSearcher的search()方法`、`重写Query`、`生成Weight`，从本文的内容可以看出，想要深入了解查询逻辑的前提是熟悉索引文件的数据结构。

### lucene源码分析—实例

本章开始分析lucene的源码，版本为目前最新的6.1.0，下面先看一段常见的lucene建立索引和进行搜索的实例，

建立索引实例：

            String filePath = ...//文件路径
            String indexPath = ...//索引路径
            File fileDir = new File(filePath);    
            Directory dir = FSDirectory.open(Paths.get(indexPath));  
    
            Analyzer luceneAnalyzer = new StandardAnalyzer();
            IndexWriterConfig iwc = new IndexWriterConfig(luceneAnalyzer);  
            iwc.setOpenMode(OpenMode.CREATE);  
            IndexWriter indexWriter = new IndexWriter(dir,iwc);    
            File[] textFiles = fileDir.listFiles();    
    
            for (int i = 0; i < textFiles.length; i++) {    
                if (textFiles[i].isFile()) {     
                    String temp = FileReaderAll(textFiles[i].getCanonicalPath(),    
                            "GBK");    
                    Document document = new Document();    
                    Field FieldPath = new StringField("path", textFiles[i].getPath(), Field.Store.YES);
                    Field FieldBody = new TextField("body", temp, Field.Store.YES);    
                    document.add(FieldPath);    
                    document.add(FieldBody);    
                    indexWriter.addDocument(document);    
                }    
            }    
            indexWriter.close();
    其中，FileReaderAll函数用来从文件中读取字符串。
搜索实例：

            String indexPath=...//索引路径  
            IndexReader reader = DirectoryReader.open(FSDirectory.open(Paths.get(indexPath)));
            IndexSearcher searcher=new IndexSearcher(reader);
            ScoreDoc[] hits=null;  
            String queryString=...//关键字符串
            Query query=null;  
            Analyzer analyzer= new StandardAnalyzer();  
            try {  
                QueryParser qp=new QueryParser("body",analyzer);
                query=qp.parse(queryString);  
            } catch (ParseException e) {  
    
            }  
            if (searcher!=null) {  
                TopDocs results=searcher.search(query, 10);
                hits=results.scoreDocs;  
                Document document=null;  
                for (int i = 0; i < hits.length; i++) {  
                    document=searcher.doc(hits[i].doc);  
                    String body=document.get("body");  
                    String path=document.get("path");  
                    String modifiedtime=document.get("modifiField");  
                }  
                reader.close();  
            }  
    后面的章节就会开始分析这两个实例究竟做了哪些工作，以及探究lucene背后的原理。
### lucene源码分析—lucene创建索引之准备工作

为了方便分析，这里再贴一次在上一章中lucene关于建立索引的实例的源代码，

            String filePath = ...//文件路径
            String indexPath = ...//索引路径
            File fileDir = new File(filePath);    
            Directory dir = FSDirectory.open(Paths.get(indexPath));  
    
            Analyzer luceneAnalyzer = new SimpleAnalyzer();
            IndexWriterConfig iwc = new IndexWriterConfig(luceneAnalyzer);  
            iwc.setOpenMode(OpenMode.CREATE);  
            IndexWriter indexWriter = new IndexWriter(dir,iwc);    
            File[] textFiles = fileDir.listFiles();    
    
            for (int i = 0; i < textFiles.length; i++) {    
                if (textFiles[i].isFile()) {     
                    String temp = FileReaderAll(textFiles[i].getCanonicalPath(),    
                            "GBK");    
                    Document document = new Document();    
                    Field FieldPath = new StringField("path", textFiles[i].getPath(), Field.Store.YES);
                    Field FieldBody = new TextField("body", temp, Field.Store.YES);    
                    document.add(FieldPath);    
                    document.add(FieldBody);    
                    indexWriter.addDocument(document);    
                }    
            }    
            indexWriter.close();

首先，FSDirectory的open函数用来打开索引文件夹，用来存放后面生成的索引文件，代码如下，

```
  public static FSDirectory open(Path path) throws IOException {
    return open(path, FSLockFactory.getDefault());
  }

  public static FSDirectory open(Path path, LockFactory lockFactory) throws IOException {
    if (Constants.JRE_IS_64BIT && MMapDirectory.UNMAP_SUPPORTED) {
      return new MMapDirectory(path, lockFactory);
    } else if (Constants.WINDOWS) {
      return new SimpleFSDirectory(path, lockFactory);
    } else {
      return new NIOFSDirectory(path, lockFactory);
    }
  }
```


FSLockFactory获得的默认LockFactory是NativeFSLockFactory，该工厂可以获得文件锁NativeFSLock，后面如果分析到再来细看这方面代码。这里假设FSDirectory的open函数创建了一个NIOFSDirectory，NIOFSDirectory继承自FSDirectory，并且直接调用了其父类FSDirectory的构造函数，

```
  protected FSDirectory(Path path, LockFactory lockFactory) throws IOException {
    super(lockFactory);
    if (!Files.isDirectory(path)) {
      Files.createDirectories(path);
    }
    directory = path.toRealPath();
  }
```


FSDirectory的构造函数根据Path创建了一个目录或者文件，并且保存了对应的路径。FSDirectory继承自BaseDirectory，其构造函数只是简单保存了LockFactory，这里就不要往下看了。

回到最上面的例子中，接下来构造了SimpleAnalyzer，然后根据构造的SimpleAnalyzer创建一个IndexWriterConfig，其构造函数直接调用了其父类LiveIndexWriterConfig的构造函数，

```
LiveIndexWriterConfig(Analyzer analyzer) {
    this.analyzer = analyzer;
    ramBufferSizeMB = IndexWriterConfig.DEFAULT_RAM_BUFFER_SIZE_MB;
    maxBufferedDocs = IndexWriterConfig.DEFAULT_MAX_BUFFERED_DOCS;
    maxBufferedDeleteTerms = IndexWriterConfig.DEFAULT_MAX_BUFFERED_DELETE_TERMS;
    mergedSegmentWarmer = null;
    delPolicy = new KeepOnlyLastCommitDeletionPolicy();
    commit = null;
    useCompoundFile = IndexWriterConfig.DEFAULT_USE_COMPOUND_FILE_SYSTEM;
    openMode = OpenMode.CREATE_OR_APPEND;
    similarity = IndexSearcher.getDefaultSimilarity();
    mergeScheduler = new ConcurrentMergeScheduler();
    indexingChain = DocumentsWriterPerThread.defaultIndexingChain;
    codec = Codec.getDefault();
    infoStream = InfoStream.getDefault();
    mergePolicy = new TieredMergePolicy();
    flushPolicy = new FlushByRamOrCountsPolicy();
    readerPooling = IndexWriterConfig.DEFAULT_READER_POOLING;
    indexerThreadPool = new DocumentsWriterPerThreadPool();
    perThreadHardLimitMB = IndexWriterConfig.DEFAULT_RAM_PER_THREAD_HARD_LIMIT_MB;
  }
```


LiveIndexWriterConfig构造函数又创建并保存了一系列组件，在后面的代码分析中如果碰到会一一分析，这里就不往下看了。

回到lucene实例中，接下来根据刚刚创建的LiveIndexWriterConfig创建一个IndexWriter，IndexWriter时lucene创建索引最为核心的类，其构造函数比较长，下面一一来看，

```
 public IndexWriter(Directory d, IndexWriterConfig conf) throws IOException {
    if (d instanceof FSDirectory && ((FSDirectory) d).checkPendingDeletions()) {
      throw new IllegalArgumentException();
    }
    conf.setIndexWriter(this);
config = conf;
infoStream = config.getInfoStream();
writeLock = d.obtainLock(WRITE_LOCK_NAME);

boolean success = false;
try {
  directoryOrig = d;
  directory = new LockValidatingDirectoryWrapper(d, writeLock);
  mergeDirectory = addMergeRateLimiters(directory);

  analyzer = config.getAnalyzer();
  mergeScheduler = config.getMergeScheduler();
  mergeScheduler.setInfoStream(infoStream);
  codec = config.getCodec();

  bufferedUpdatesStream = new BufferedUpdatesStream(infoStream);
  poolReaders = config.getReaderPooling();

  OpenMode mode = config.getOpenMode();
  boolean create;
  if (mode == OpenMode.CREATE) {
    create = true;
  } else if (mode == OpenMode.APPEND) {
    create = false;
  } else {
    create = !DirectoryReader.indexExists(directory);
  }
  boolean initialIndexExists = true;
  String[] files = directory.listAll();
  IndexCommit commit = config.getIndexCommit();

  StandardDirectoryReader reader;
  if (commit == null) {
    reader = null;
  } else {
    reader = commit.getReader();
  }

  if (create) {

    if (config.getIndexCommit() != null) {
      if (mode == OpenMode.CREATE) {
        throw new IllegalArgumentException();
      } else {
        throw new IllegalArgumentException();
      }
    }

    SegmentInfos sis = null;
    try {
      sis = SegmentInfos.readLatestCommit(directory);
      sis.clear();
    } catch (IOException e) {
      initialIndexExists = false;
      sis = new SegmentInfos();
    }
    segmentInfos = sis;
    rollbackSegments = segmentInfos.createBackupSegmentInfos();
    changed();

  } else if (reader != null) {
    ...
  } else {
    ...
  }

  pendingNumDocs.set(segmentInfos.totalMaxDoc());
  globalFieldNumberMap = getFieldNumberMap();
  config.getFlushPolicy().init(config);
  docWriter = new DocumentsWriter(this, config, directoryOrig, directory);
  eventQueue = docWriter.eventQueue();
  synchronized(this) {
    deleter = new IndexFileDeleter(files, directoryOrig, directory,
                                   config.getIndexDeletionPolicy(),
                                   segmentInfos, infoStream, this,
                                   initialIndexExists, reader != null);
    assert create || filesExist(segmentInfos);
  }

  if (deleter.startingCommitDeleted) {
    changed();
  }

  if (reader != null) {
    ...
  }

  success = true;

} finally {
  if (!success) {
    IOUtils.closeWhileHandlingException(writeLock);
    writeLock = null;
  }
}
}
```


IndexWriter构造函数首先通过checkPendingDeletions函数删除被标记的文件，checkPendingDeletions函数定义在FSDirectory中，如下所示

```
  public boolean checkPendingDeletions() throws IOException {
    deletePendingFiles();
    return pendingDeletes.isEmpty() == false;
  }

  public synchronized void deletePendingFiles() throws IOException {
    if (pendingDeletes.isEmpty() == false) {
      for(String name : new HashSet<>(pendingDeletes)) {
        privateDeleteFile(name, true);
      }
    }
  }

  private void privateDeleteFile(String name, boolean isPendingDelete) throws IOException {
    try {
      Files.delete(directory.resolve(name));
      pendingDeletes.remove(name);
    } catch (NoSuchFileException | FileNotFoundException e) {

    } catch (IOException ioe) {

    }
  }

```


checkPendingDeletions函数最后调用Files的delete函数删除保存在pendingDeletes的文件。

回到IndexWriter的构造函数中，接下来通过infoStream获得在LiveIndexWriterConfig构造函数中创建的NoOutput，该infoStream用来显示信息，然后调用FSDirectory的obtainLock函数获得文件的写锁，这里就不往下分析了。

回到IndexWriter的构造函数中，接下来会经过一系列的创建和赋值操作，假设create为true，即表示第一次创建或者重新创建索引，然后会通过SegmentInfos的readLatestCommit函数读取段信息，

```
 public static final SegmentInfos readLatestCommit(Directory directory) throws IOException {
    return new FindSegmentsFile<SegmentInfos>(directory) {
      @Override
      protected SegmentInfos doBody(String segmentFileName) throws IOException {
        return readCommit(directory, segmentFileName);
      }
    }.run();
  }

```


SegmentInfos的readLatestCommit函数创建了一个FindSegmentsFile并调用其run函数，定义如下，

    public T run() throws IOException {
      return run(null);
    }
    
    public T run(IndexCommit commit) throws IOException {
      long lastGen = -1;
      long gen = -1;
      IOException exc = null;
    
      for (;;) {
        lastGen = gen;
        String files[] = directory.listAll();
        String files2[] = directory.listAll();
        Arrays.sort(files);
        Arrays.sort(files2);
        if (!Arrays.equals(files, files2)) {
          continue;
        }
        gen = getLastCommitGeneration(files);
        if (gen == -1) {
          throw new IndexNotFoundException();
        } else if (gen > lastGen) {
          String segmentFileName = IndexFileNames.fileNameFromGeneration(IndexFileNames.SEGMENTS, "", gen);
    
          try {
            T t = doBody(segmentFileName);
            return t;
          } catch (IOException err) {
    
          }
        } else {
          throw exc;
        }
      }
    }
这里的泛型T就是SegmentInfos，run函数首先调用getLastCommitGeneration函数获得gen信息，假设索引文件夹下有一个文件名为segments_6的文件，则getLastCommitGeneration最后会返回6赋值到gen中，接下来，如果gen大于lastGen，就表示段信息有更新了，这时候就要通过doBody函数读取该segments_6文件的信息，并返回一个SegmentInfos。
根据前面readLatestCommit的代码，doBody函数最后会调用readCommit函数，定义在SegmentInfos中，代码如下

```
  public static final SegmentInfos readCommit(Directory directory, String segmentFileName) throws IOException {
    long generation = generationFromSegmentsFileName(segmentFileName);
    try (ChecksumIndexInput input = directory.openChecksumInput(segmentFileName, IOContext.READ)) {
      return readCommit(directory, input, generation);
    }
  }
```


readCommit函数首先创建一个ChecksumIndexInput，然后通过readCommit函数读取段信息并返回一个SegmentInfos，这里的readCommit函数和具体的segments_*文件格式和协议相关，这里就不往下看了。最后返回的SegmentInfos保存了段信息。

回到IndexWriter的构造函数中，如果readLatestCommit函数返回的SegmentInfos不为空，就调用其clear清空，如果是第一次创建索引，就会构造一个SegmentInfos，SegmentInfos的构造函数为空函数。接下来调用SegmentInfos的createBackupSegmentInfos函数备份其中的SegmentCommitInfo信息列表，该备份主要是为了回滚rollback操作使用。IndexWriter然后调用changed表示段信息发生了变化。

继续往下看IndexWriter的构造函数，pendingNumDocs函数记录了索引记录的文档总数，globalFieldNumberMap记录了该段中Field的相关信息，getFlushPolicy返回在LiveIndexWriterConfig构造函数中创建的FlushByRamOrCountsPolicy，然后通过FlushByRamOrCountsPolicy的init函数进行简单的赋值。再往下创建了一个DocumentsWriter，并获得其事件队列保存在eventQueue中。IndexWriter的构造函数接下来会创建一个IndexFileDeleter，IndexFileDeleter用来管理索引文件，例如添加引用计数，在多线程环境下操作索引文件时可以保持同步性。

下一章继续分析lucene创建索引的实例的源代码。


### lucene源码分析—添加文档

为了方便分析，这里继续贴出第一章中给出的lucene中关于创建索引的实例，

            String filePath = ...//文件路径
            String indexPath = ...//索引路径
            File fileDir = new File(filePath);    
            Directory dir = FSDirectory.open(Paths.get(indexPath));  
    
            Analyzer luceneAnalyzer = new SimpleAnalyzer();
            IndexWriterConfig iwc = new IndexWriterConfig(luceneAnalyzer);  
            iwc.setOpenMode(OpenMode.CREATE);  
            IndexWriter indexWriter = new IndexWriter(dir,iwc);    
            File[] textFiles = fileDir.listFiles();    
    
            for (int i = 0; i < textFiles.length; i++) {    
                if (textFiles[i].isFile()) {     
                    String temp = FileReaderAll(textFiles[i].getCanonicalPath(),    
                            "GBK");    
                    Document document = new Document();    
                    Field FieldPath = new StringField("path", textFiles[i].getPath(), Field.Store.YES);
                    Field FieldBody = new TextField("body", temp, Field.Store.YES);    
                    document.add(FieldPath);    
                    document.add(FieldBody);    
                    indexWriter.addDocument(document);    
                }    
            }    
            indexWriter.close();

这段代码中，FileReaderAll函数用来从文件中读取字符串，默认编码为“GBK”。在创建完最重要的IndexWriter之后，就开始遍历需要索引的文件，构造对应的Document和Filed类，最终通过IndexWriter的addDocument函数开始索引。
Document的构造函数为空，StringField、TextField和Field的构造函数也很简单。下面重点分析IndexWriter的addDocument函数，代码如下，

addDocument
addDocument函数不仅添加文档数据，而且创建了索引，下面来看。
IndexWriter::addDocument

```
 public void addDocument(Iterable<? extends IndexableField> doc) throws IOException {
    updateDocument(null, doc);
  }
  public void updateDocuments(Term delTerm, Iterable<? extends Iterable<? extends IndexableField>> docs) throws IOException {
    ensureOpen();
    try {
      boolean success = false;
      try {
        if (docWriter.updateDocuments(docs, analyzer, delTerm)) {
          processEvents(true, false);
        }
        success = true;
      } finally {
       }
} catch (AbortingException | VirtualMachineError tragedy) {

} 
}
```



addDocument进而调用updateDocuments函数完成索引的添加。传入的参数delTerm与更新索引和删除操作有关，这里为null，不管它。updateDocuments函数首先通过ensureOpen函数确保IndexWriter未被关闭，然后就调用DocumentsWriter的updateDocuments函数，代码如下，
IndexWriter::updateDocuments->DocumentsWriter::updateDocuments

  boolean updateDocuments(final Iterable<? extends Iterable<? extends IndexableField>> docs, final Analyzer analyzer,final Term delTerm) throws IOException, AbortingException {

    boolean hasEvents = preUpdate();
    
    final ThreadState perThread = flushControl.obtainAndLock();
    final DocumentsWriterPerThread flushingDWPT;
    
    try {
      ensureOpen();
      ensureInitialized(perThread);
      assert perThread.isInitialized();
      final DocumentsWriterPerThread dwpt = perThread.dwpt;
      final int dwptNumDocs = dwpt.getNumDocsInRAM();
      try {
        dwpt.updateDocuments(docs, analyzer, delTerm);
      } catch (AbortingException ae) {
    
      } finally {
        numDocsInRAM.addAndGet(dwpt.getNumDocsInRAM() - dwptNumDocs);
      }
      final boolean isUpdate = delTerm != null;
      flushingDWPT = flushControl.doAfterDocument(perThread, isUpdate);
    } finally {
      perThreadPool.release(perThread);
    }
    
    return postUpdate(flushingDWPT, hasEvents);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
updateDocuments首先调用preUpdate函数处理没有写入硬盘的数据，代码如下。
IndexWriter::updateDocuments->DocumentsWriter::updateDocuments->preUpdate

  private boolean preUpdate() throws IOException, AbortingException {
    ensureOpen();
    boolean hasEvents = false;
    if (flushControl.anyStalledThreads() || flushControl.numQueuedFlushes() > 0) {
      do {
        DocumentsWriterPerThread flushingDWPT;
        while ((flushingDWPT = flushControl.nextPendingFlush()) != null) {
          hasEvents |= doFlush(flushingDWPT);
        }

        flushControl.waitIfStalled();
      } while (flushControl.numQueuedFlushes() != 0);
    
    }
    return hasEvents;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
flushControl是在DocumentsWriter构造函数中创建的DocumentsWriterFlushControl。preUpdate函数从DocumentsWriterFlushControl中逐个取出DocumentsWriterPerThread，因为在lucene中只能有一个IndexWriter获得文件锁并操作索引文件，但是实际中对文档的索引需要多线程进行，DocumentsWriterPerThread就代表一个索引文档的线程。获取到DocumentsWriterPerThread之后，就通过doFlush将DocumentsWriterPerThread内存中的索引数据写入硬盘文件里。关于doFlush函数的分析，留在后面的章节。

回到DocumentsWriter的updateDocuments函数中，接下来通过DocumentsWriterFlushControl的obtainAndLock函数获得一个DocumentsWriterPerThread，DocumentsWriterPerThread被封装在ThreadState中，obtainAndLock函数的代码如下，

IndexWriter::updateDocuments->DocumentsWriter::updateDocuments->DocumentsWriterFlushControl::obtainAndLock

  ThreadState obtainAndLock() {
    final ThreadState perThread = perThreadPool.getAndLock(Thread
        .currentThread(), documentsWriter);
    boolean success = false;
    try {
      if (perThread.isInitialized()
          && perThread.dwpt.deleteQueue != documentsWriter.deleteQueue) {
        addFlushableState(perThread);
      }
      success = true;
      return perThread;
    } finally {

    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
obtainAndLock函数中的perThreadPool是在LiveIndexWriterConfig中创建的DocumentsWriterPerThreadPool，其对应的getAndLock函数如下，
IndexWriter::updateDocuments->DocumentsWriter::updateDocuments->DocumentsWriterFlushControl::obtainAndLock->DocumentsWriterPerThreadPool::getAndLock

  ThreadState getAndLock(Thread requestingThread, DocumentsWriter documentsWriter) {
    ThreadState threadState = null;
    synchronized (this) {
      if (freeList.isEmpty()) {
        return newThreadState();
      } else {
        threadState = freeList.remove(freeList.size()-1);
        if (threadState.dwpt == null) {
          for(int i=0;i<freeList.size();i++) {
            ThreadState ts = freeList.get(i);
            if (ts.dwpt != null) {
              freeList.set(i, threadState);
              threadState = ts;
              break;
            }
          }
        }
      }
    }
    threadState.lock();
    return threadState;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
getAndLock函数概括来说，就是如果freeList不为空，就从中取出成员变量dwpt不为空的ThreadState，否则就创建一个新的ThreadState，创建ThreadState对应的newThreadState函数如下，
IndexWriter::updateDocuments->DocumentsWriter::updateDocuments->DocumentsWriterFlushControl::obtainAndLock->DocumentsWriterPerThreadPool::getAndLock->newThreadState

  private synchronized ThreadState newThreadState() {
    while (aborted) {
      try {
        wait();
      } catch (InterruptedException ie) {
        throw new ThreadInterruptedException(ie);        
      }
    }
    ThreadState threadState = new ThreadState(null);
    threadState.lock();
    threadStates.add(threadState);
    return threadState;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
newThreadState函数创建一个新的ThreadState，并将其添加threadStates中。ThreadState的构造函数很简单，这里就不往下看了。

回到DocumentsWriterFlushControl的obtainAndLock函数中，如果新创建的ThreadState中的dwpt为空，因此isInitialized返回false，obtainAndLock直接返回刚刚创建的ThreadState。

再回到DocumentsWriter的updateDocuments函数中，接下来通过ensureInitialized函数初始化刚刚创建的ThreadState中的dwpt成员变量，ensureInitialized函数的定义如下
IndexWriter::updateDocuments->DocumentsWriter::updateDocuments->ensureInitialized

  private void ensureInitialized(ThreadState state) throws IOException {
    if (state.dwpt == null) {
      final FieldInfos.Builder infos = new FieldInfos.Builder(writer.globalFieldNumberMap);
      state.dwpt = new DocumentsWriterPerThread(writer, writer.newSegmentName(), directoryOrig,
                                                directory, config, infoStream, deleteQueue, infos,
                                                writer.pendingNumDocs, writer.enableTestPoints);
    }
  }
1
2
3
4
5
6
7
8
ensureInitialized函数会创建一个DocumentsWriterPerThread并赋值给ThreadState的dwpt成员变量，DocumentsWriterPerThread的构造函数很简单，这里就不往下看了。

初始化完ThreadState的dwpt后，updateDocuments函数继续调用刚刚创建的DocumentsWriterPerThread的updateDocuments函数来索引文档，定义如下，
IndexWriter::updateDocuments->DocumentsWriter::updateDocuments->DocumentsWriterPerThread::updateDocuments

  public int updateDocuments(Iterable<? extends Iterable<? extends IndexableField>> docs, Analyzer analyzer, Term delTerm) throws IOException, AbortingException {
    docState.analyzer = analyzer;
    int docCount = 0;
    boolean allDocsIndexed = false;
    try {

      for(Iterable<? extends IndexableField> doc : docs) {
        reserveOneDoc();
        docState.doc = doc;
        docState.docID = numDocsInRAM;
        docCount++;
    
        boolean success = false;
        try {
          consumer.processDocument();
          success = true;
        } finally {
    
        }
        finishDocument(null);
      }
      allDocsIndexed = true;
    
    } finally {
    
    }
    
    return docCount;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
DocumentsWriterPerThread的updateDocuments函数首先调用reserveOneDoc查看索引中的文档数是否超过限制，文档的信息被封装在成员变量DocState中，然后调用consumer的processDocument函数继续处理，consumer被定义为DefaultIndexingChain。

DefaultIndexingChain的processDocument函数
DefaultIndexingChain是一个默认的索引处理链，下面来看它的processDocument函数。
IndexWriter::updateDocuments->DocumentsWriter::updateDocuments->DocumentsWriterPerThread::updateDocuments->DefaultIndexingChain::processDocument

  public void processDocument() throws IOException, AbortingException {

    int fieldCount = 0;
    long fieldGen = nextFieldGen++;
    termsHash.startDocument();
    fillStoredFields(docState.docID);
    startStoredFields();
    
    boolean aborting = false;
    try {
      for (IndexableField field : docState.doc) {
        fieldCount = processField(field, fieldGen, fieldCount);
      }
    } catch (AbortingException ae) {
    
    } finally {
      if (aborting == false) {
        for (int i=0;i<fieldCount;i++) {
          fields[i].finish();
        }
        finishStoredFields();
      }
    }
    
    try {
      termsHash.finishDocument();
    } catch (Throwable th) {
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
DefaultIndexingChain中的termsHash在DefaultIndexingChain的构造函数中被定义为FreqProxTermsWriter，其startDocument最终会调用FreqProxTermsWriter以及TermVectorsConsumer的startDocument函数作一些简单的初始化工作，FreqProxTermsWriter和TermVectorsConsumer分别在内存中保存了词和词向量的信息。DefaultIndexingChain的processDocument函数接下来通过fillStoredFields继续完成一些初始化工作。
DefaultIndexingChain::processDocument->fillStoredFields

  private void fillStoredFields(int docID) throws IOException, AbortingException {
    while (lastStoredDocID < docID) {
      startStoredFields();
      finishStoredFields();
    }
  }

  private void startStoredFields() throws IOException, AbortingException {
    try {
      initStoredFieldsWriter();
      storedFieldsWriter.startDocument();
    } catch (Throwable th) {
      throw AbortingException.wrap(th);
    }
    lastStoredDocID++;
  }

  private void finishStoredFields() throws IOException, AbortingException {
    try {
      storedFieldsWriter.finishDocument();
    } catch (Throwable th) {

    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
startStoredFields函数中的initStoredFieldsWriter函数用来创建一个StoredFieldsWriter，用来存储Field域中的值，定义如下，
DefaultIndexingChain::processDocument->fillStoredFields->startStoredFields->initStoredFieldsWriter

  private void initStoredFieldsWriter() throws IOException {
    if (storedFieldsWriter == null) {
      storedFieldsWriter = docWriter.codec.storedFieldsFormat().fieldsWriter(docWriter.directory, docWriter.getSegmentInfo(), IOContext.DEFAULT);
    }
  }
1
2
3
4
5
这里的docWriter是DocumentsWriterPerThread，其成员变量codec会被初始化为Lucene60Codec，其storedFieldsFormat函数返回一个Lucene50StoredFieldsFormat，这些类的名称都是为了兼容使用的。Lucene50StoredFieldsFormat的fieldsWriter函数如下，
DefaultIndexingChain::processDocument->fillStoredFields->startStoredFields->initStoredFieldsWriter->Lucene50StoredFieldsFormat::fieldsWriter

  public StoredFieldsWriter fieldsWriter(Directory directory, SegmentInfo si, IOContext context) throws IOException {
    String previous = si.putAttribute(MODE_KEY, mode.name());
    return impl(mode).fieldsWriter(directory, si, context);
  }
1
2
3
4
mode的默认值为BEST_SPEED，impl根据该mode会返回一个CompressingStoredFieldsFormat实例，CompressingStoredFieldsFormat的fieldsWriter函数最后会创建一个CompressingStoredFieldsWriter并返回，下面简单看一下CompressingStoredFieldsWriter的构造函数，

  public CompressingStoredFieldsWriter(Directory directory, SegmentInfo si, String segmentSuffix, IOContext context,
      String formatName, CompressionMode compressionMode, int chunkSize, int maxDocsPerChunk, int blockSize) throws IOException {
    assert directory != null;
    this.segment = si.name;
    this.compressionMode = compressionMode;
    this.compressor = compressionMode.newCompressor();
    this.chunkSize = chunkSize;
    this.maxDocsPerChunk = maxDocsPerChunk;
    this.docBase = 0;
    this.bufferedDocs = new GrowableByteArrayDataOutput(chunkSize);
    this.numStoredFields = new int[16];
    this.endOffsets = new int[16];
    this.numBufferedDocs = 0;

    boolean success = false;
    IndexOutput indexStream = directory.createOutput(IndexFileNames.segmentFileName(segment, segmentSuffix, FIELDS_INDEX_EXTENSION), 
                                                                     context);
    try {
      fieldsStream = directory.createOutput(IndexFileNames.segmentFileName(segment, segmentSuffix, FIELDS_EXTENSION),context);
    
      final String codecNameIdx = formatName + CODEC_SFX_IDX;
      final String codecNameDat = formatName + CODEC_SFX_DAT;
      CodecUtil.writeIndexHeader(indexStream, codecNameIdx, VERSION_CURRENT, si.getId(), segmentSuffix);
      CodecUtil.writeIndexHeader(fieldsStream, codecNameDat, VERSION_CURRENT, si.getId(), segmentSuffix);
    
      indexWriter = new CompressingStoredFieldsIndexWriter(indexStream, blockSize);
      indexStream = null;
    
      fieldsStream.writeVInt(chunkSize);
      fieldsStream.writeVInt(PackedInts.VERSION_CURRENT);
    
      success = true;
    } finally {
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
简单地说，CompressingStoredFieldsWriter构造函数会创建对应的.fdt和.fdx文件，并写入相应的头信息。

回到DefaultIndexingChain的startStoredFields函数中，构造完CompressingStoredFieldsWriter后，会调用其startDocument函数，该函数为空。再回头看finishStoredFields函数，该函数会通过刚刚构造的CompressingStoredFieldsWriter的finishDocument函数进行一些统计工作，在满足条件时，通过flush函数将内存中的索引信息写入到硬盘文件中，flush函数的源码留在后面的章节分析。
DefaultIndexingChain::processDocument->fillStoredFields->finishStoredFields->CompressingStoredFieldsWriter::finishDocument

  public void finishDocument() throws IOException {
    if (numBufferedDocs == this.numStoredFields.length) {
      final int newLength = ArrayUtil.oversize(numBufferedDocs + 1, 4);
      this.numStoredFields = Arrays.copyOf(this.numStoredFields, newLength);
      endOffsets = Arrays.copyOf(endOffsets, newLength);
    }
    this.numStoredFields[numBufferedDocs] = numStoredFieldsInDoc;
    numStoredFieldsInDoc = 0;
    endOffsets[numBufferedDocs] = bufferedDocs.length;
    ++numBufferedDocs;
    if (triggerFlush()) {
      flush();
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
再回到DefaultIndexingChain的processDocument函数中，接下来遍历文档的Field域，并通过processField函数逐个处理每个域，processField函数的定义如下，
DefaultIndexingChain::processDocument->processField

  private int processField(IndexableField field, long fieldGen, int fieldCount) throws IOException, AbortingException {
    String fieldName = field.name();
    IndexableFieldType fieldType = field.fieldType();

    PerField fp = null;
    if (fieldType.indexOptions() != IndexOptions.NONE) {
    
      fp = getOrAddField(fieldName, fieldType, true);
      boolean first = fp.fieldGen != fieldGen;
      fp.invert(field, first);
    
      if (first) {
        fields[fieldCount++] = fp;
        fp.fieldGen = fieldGen;
      }
    } else {
      verifyUnIndexedFieldType(fieldName, fieldType);
    }
    
    if (fieldType.stored()) {
      if (fp == null) {
        fp = getOrAddField(fieldName, fieldType, false);
      }
      if (fieldType.stored()) {
        try {
          storedFieldsWriter.writeField(fp.fieldInfo, field);
        } catch (Throwable th) {
          throw AbortingException.wrap(th);
        }
      }
    }
    
    return fieldCount;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
processField函数中的getOrAddField函数用来根据Field的name信息创建或获得一个PerField，代码如下，
DefaultIndexingChain::processDocument->processField->getOrAddField

  private PerField getOrAddField(String name, IndexableFieldType fieldType, boolean invert) {

    final int hashPos = name.hashCode() & hashMask;
    PerField fp = fieldHash[hashPos];
    while (fp != null && !fp.fieldInfo.name.equals(name)) {
      fp = fp.next;
    }
    
    if (fp == null) {
      FieldInfo fi = fieldInfos.getOrAdd(name);
      fi.setIndexOptions(fieldType.indexOptions());
    
      fp = new PerField(fi, invert);
      fp.next = fieldHash[hashPos];
      fieldHash[hashPos] = fp;
      totalFieldCount++;
    
      if (totalFieldCount >= fieldHash.length/2) {
        rehash();
      }
    
      if (totalFieldCount > fields.length) {
        PerField[] newFields = new PerField[ArrayUtil.oversize(totalFieldCount, RamUsageEstimator.NUM_BYTES_OBJECT_REF)];
        System.arraycopy(fields, 0, newFields, 0, fields.length);
        fields = newFields;
      }
    
    } else if (invert && fp.invertState == null) {
      fp.fieldInfo.setIndexOptions(fieldType.indexOptions());
      fp.setInvertState();
    }
    
    return fp;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
成员变量fieldHash是一个hash桶，用来存储PerField，PerField中的FieldInfo保存了Field的名称name等相应信息，并被选择适当的位置插入到fieldHash中，getOrAddField函数也会在适当的时候扩充hash桶，最后返回一个hash桶中指向新创建的PerField的指针。
回到processField函数中，接下来通过invert方法调用analyzer解析Field，这部分代码比较复杂，留到下一章分析。

回到processField函数中，假设fieldType为TYPE_STORED，而TYPE_STORED对应的stored函数返回true，表示需要对该域的值进行存储，因此接下来调用StoredFieldsWriter的writeField函数保存对应Field的值。根据本章前面的分析，这里的storedFieldsWriter为CompressingStoredFieldsWriter，其writeField函数如下，
DefaultIndexingChain::processDocument->processField->CompressingStoredFieldsWriter::writeField

  public void writeField(FieldInfo info, IndexableField field)
      throws IOException {

    ...
    
    string = field.stringValue();
    
    ...
    
    if (bytes != null) {
      bufferedDocs.writeVInt(bytes.length);
      bufferedDocs.writeBytes(bytes.bytes, bytes.offset, bytes.length);
    } else if (string != null) {
      bufferedDocs.writeString(string);
    } else {
      if (number instanceof Byte || number instanceof Short || number instanceof Integer) {
        bufferedDocs.writeZInt(number.intValue());
      } else if (number instanceof Long) {
        writeTLong(bufferedDocs, number.longValue());
      } else if (number instanceof Float) {
        writeZFloat(bufferedDocs, number.floatValue());
      } else if (number instanceof Double) {
        writeZDouble(bufferedDocs, number.doubleValue());
      } else {
        throw new AssertionError("Cannot get here");
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
假设需要存储的Field的值为String类型，因此这里最终会调用bufferedDocs的writeString函数，bufferedDocs在CompressingStoredFieldsWriter的构造函数中被设置为GrowableByteArrayDataOutput，其writeString函数就是将对应的String值缓存在内存的某个结构中。在特定的时刻通过flush函数将这些数据存入.fdt文件中。

回到DefaultIndexingChain的processDocument函数中，接下来遍历fields，取出前面在processField函数中创建的各个PerField，并调用其finish函数，该函数深入看较为复杂，但没有特别重要的内容，这里就不往下看了。processDocument最后会调用finishStoredFields函数，该函数前面已经分析过了，主要是在必要的时候触发一次flush操作。

回到DocumentsWriterPerThread的updateDocuments函数中，接下来清空刚刚创建的DocState，并调用finishDocument处理一些需要删除的数据，这里先不管该函数。

再向上回到DocumentsWriter的updateDocuments函数中，接下来的numDocsInRAM保存了当前有多少文档被索引了，然后再调用DocumentsWriterFlushControl的doAfterDocument函数继续处理，，本章暂时不管它。然后updateDocuments函数调用release函数释放开始创建的ThreadState，即将其放入一个freeList中用来给后面的程序调用。最后通过postUpdate函数选择相应的DocumentsWriterPerThread并调用其doFlush函数将索引数据写入硬盘文件中。
IndexWriter::updateDocuments->DocumentsWriter::updateDocuments->

  private boolean postUpdate(DocumentsWriterPerThread flushingDWPT, boolean hasEvents) throws IOException, AbortingException {
    hasEvents |= applyAllDeletes(deleteQueue);
    if (flushingDWPT != null) {
      hasEvents |= doFlush(flushingDWPT);
    } else {
      final DocumentsWriterPerThread nextPendingFlush = flushControl.nextPendingFlush();
      if (nextPendingFlush != null) {
        hasEvents |= doFlush(nextPendingFlush);
      }
    }

    return hasEvents;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
再向上回到IndexWriter的updateDocument中，接下来调用processEvents触发相应的事件，代码如下，
IndexWriter::updateDocuments->processEvents

  private boolean processEvents(boolean triggerMerge, boolean forcePurge) throws IOException {
    return processEvents(eventQueue, triggerMerge, forcePurge);
  }

  private boolean processEvents(Queue<Event> queue, boolean triggerMerge, boolean forcePurge) throws IOException {
    boolean processed = false;
    if (tragedy == null) {
      Event event;
      while((event = queue.poll()) != null)  {
        processed = true;
        event.process(this, triggerMerge, forcePurge);
      }
    }
    return processed;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
processEvents函数会一次从Queue队列中取出Event事件，并调用其process函数进行处理。
————————————————
版权声明：本文为CSDN博主「二侠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/conansonic/article/details/51886014

lucene源码分析—倒排表存储
根据《lucene源码分析—3》中的分析，倒排表的存储函数是DefaultIndexingChain的processField函数中的invert函数，invert函数定义在PerField中，代码如下，
IndexWriter::updateDocuments->DocumentsWriter::updateDocuments->DefaultIndexingChain::processDocument->processField->PerField::invert

    public void invert(IndexableField field, boolean first) throws IOException, AbortingException {
      if (first) {
        invertState.reset();
      }
    
      IndexableFieldType fieldType = field.fieldType();
    
      IndexOptions indexOptions = fieldType.indexOptions();
      fieldInfo.setIndexOptions(indexOptions);
    
      if (fieldType.omitNorms()) {
        fieldInfo.setOmitsNorms();
      }
    
      final boolean analyzed = fieldType.tokenized() && docState.analyzer != null;
      final boolean checkOffsets = indexOptions == IndexOptions.DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS;
    
      boolean succeededInProcessingField = false;
      try (TokenStream stream = tokenStream = field.tokenStream(docState.analyzer, tokenStream)) {
        stream.reset();
        invertState.setAttributeSource(stream);
        termsHashPerField.start(field, first);
    
        while (stream.incrementToken()) {
          int posIncr = invertState.posIncrAttribute.getPositionIncrement();
          invertState.position += posIncr;
          invertState.lastPosition = invertState.position;
          if (posIncr == 0) {
            invertState.numOverlap++;
          }
    
          if (checkOffsets) {
            int startOffset = invertState.offset + invertState.offsetAttribute.startOffset();
            int endOffset = invertState.offset + invertState.offsetAttribute.endOffset();
            invertState.lastStartOffset = startOffset;
          }
    
          invertState.length++;
          try {
            termsHashPerField.add();
          } catch (MaxBytesLengthExceededException e) {
    
          } catch (Throwable th) {
    
          }
        }
        stream.end();
        invertState.position += invertState.posIncrAttribute.getPositionIncrement();
        invertState.offset += invertState.offsetAttribute.endOffset();
        succeededInProcessingField = true;
      }
    
      if (analyzed) {
        invertState.position += docState.analyzer.getPositionIncrementGap(fieldInfo.name);
        invertState.offset += docState.analyzer.getOffsetGap(fieldInfo.name);
      }
    
      invertState.boost *= field.boost();
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
传入的参数Field假设为TextField，布尔类型first表示是否是某个文档中对应Field名字name的第一个Field。invertState存储了本次建立索引的各个状态，如果是第一次，这要通过reset函数对invertState进行初始化。invert函数接下来通过tokenStream函数获得TokenStream，TokenStream代表一个分词器。tokenStream函数定义在Field中，代码如下，
PerField::invert->Field::tokenStream

  public TokenStream tokenStream(Analyzer analyzer, TokenStream reuse) {
    if (fieldType().indexOptions() == IndexOptions.NONE) {
      return null;
    }

    ...
    
    if (tokenStream != null) {
      return tokenStream;
    } else if (readerValue() != null) {
      return analyzer.tokenStream(name(), readerValue());
    } else if (stringValue() != null) {
      return analyzer.tokenStream(name(), stringValue());
    }

  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
这段tokenStream代码进行了简化，假设Field的域值是String类型，Analyzer设定为SimpleAnalyzer，因此接下来看SimpleAnalyzer的tokenStream函数，
PerField::invert->Field::tokenStream->SimpleAnalyzer::tokenStream

  public final TokenStream tokenStream(final String fieldName,
                                       final Reader reader) {
    TokenStreamComponents components = reuseStrategy.getReusableComponents(this, fieldName);
    final Reader r = initReader(fieldName, reader);
    if (components == null) {
      components = createComponents(fieldName);
      reuseStrategy.setReusableComponents(this, fieldName, components);
    }
    components.setReader(r);
    return components.getTokenStream();
  }
1
2
3
4
5
6
7
8
9
10
11
Analyzer的默认reuseStrategy为ReuseStrategy，其getReusableComponents函数如下，
PerField::invert->Field::tokenStream->SimpleAnalyzer::tokenStream->ReuseStrategy::getReusableComponents

    public TokenStreamComponents getReusableComponents(Analyzer analyzer, String fieldName) {
      return (TokenStreamComponents) getStoredValue(analyzer);
    }
    
    protected final Object getStoredValue(Analyzer analyzer) {
      return analyzer.storedValue.get();
    }
1
2
3
4
5
6
7
storedValue是线程安全的Object变量。
假设通过getReusableComponents函数获取到的TokenStreamComponents为空，因此tokenStream函数接下来会创建一个ReusableStringReader用来封装String类型的域值，然后通过createComponents函数创建一个TokenStreamComponents，createComponents函数被SimpleAnalyzer重载，定义如下，
PerField::invert->Field::tokenStream->SimpleAnalyzer::tokenStream->SimpleAnalyzer::createComponents

  protected TokenStreamComponents createComponents(final String fieldName) {
    return new TokenStreamComponents(new LowerCaseTokenizer());
  }
1
2
3
createComponents创建一个LowerCaseTokenizer，并将其作为参数构造一个TokenStreamComponents。

回到Analyzer的tokenStream函数中，接下来将刚刚创建的TokenStreamComponents缓存到ReuseStrategy中。最后设置Field的域值Reader，并返回TokenStream。从createComponents可以知道，这里返回的其实是LowerCaseTokenizer。

再回到PerField的invert函数中，接下来调用TokenStream的reset函数进行初始化，该reset函数初始化一些值，并设置其成员变量input为前面封装Field域值的Reader。

invert函数接下来通过setAttributeSource设置invertState的各个属性，然后依次调用TermsHashPerField的各个start函数进行一些初始化工作，该start函数最终会调用FreqProxTermsWriterPerField以及TermVectorsConsumerPerField的start函数。再往下，invert函数通过while循环，调用TokenStream的incrementToken函数，下面看LowerCaseTokenizer的incrementToken函数，定义如下
PerField::invert->LowerCaseTokenizer::incrementToken

  public final boolean incrementToken() throws IOException {
    clearAttributes();
    int length = 0;
    int start = -1;
    int end = -1;
    char[] buffer = termAtt.buffer();
    while (true) {
      if (bufferIndex >= dataLen) {
        offset += dataLen;
        charUtils.fill(ioBuffer, input);
        if (ioBuffer.getLength() == 0) {
          dataLen = 0;
          if (length > 0) {
            break;
          } else {
            finalOffset = correctOffset(offset);
            return false;
          }
        }
        dataLen = ioBuffer.getLength();
        bufferIndex = 0;
      }
      final int c = charUtils.codePointAt(ioBuffer.getBuffer(), bufferIndex, ioBuffer.getLength());
      final int charCount = Character.charCount(c);
      bufferIndex += charCount;

      if (isTokenChar(c)) {
        if (length == 0) {
          start = offset + bufferIndex - charCount;
          end = start;
        } else if (length >= buffer.length-1) {
          buffer = termAtt.resizeBuffer(2+length);
        }
        end += charCount;
        length += Character.toChars(normalize(c), buffer, length);
        if (length >= MAX_WORD_LEN)
          break;
      } else if (length > 0)
        break;
    }
    
    termAtt.setLength(length);
    offsetAtt.setOffset(correctOffset(start), finalOffset = correctOffset(end));
    return true;

  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
这段代码虽然长，但这里只看关键的几个函数，成员变量input的类型为Reader，存放了Field域中的值，charUtils实际上是Java5CharacterUtils。首先通过Java5CharacterUtils的fill函数将input中的值读取到类型为CharacterBuffer的ioBuffer中，然后通过isTokenChar和normalize函数判断并且过滤字符c，isTokenChar用来判断字符c是否为字母，而normalize将c转化为小写字母，最终将字符c放入buffer，也即termAtt.buffer()中。

回到DefaultIndexingChain的invert函数中，接下来对成员变量invertState进行一些设置后就调用termsHashPerField的add函数，termsHashPerField的类型为FreqProxTermsWriterPerField，add函数定义在其父类TermsHashPerField中，
PerField::invert->FreqProxTermsWriterPerField::add

  void add() throws IOException {
    int termID = bytesHash.add(termAtt.getBytesRef());
    if (termID >= 0) {
      bytesHash.byteStart(termID);
      if (numPostingInt + intPool.intUpto > IntBlockPool.INT_BLOCK_SIZE) {
        intPool.nextBuffer();
      }

      if (ByteBlockPool.BYTE_BLOCK_SIZE - bytePool.byteUpto < numPostingInt*ByteBlockPool.FIRST_LEVEL_SIZE) {
        bytePool.nextBuffer();
      }
    
      intUptos = intPool.buffer;
      intUptoStart = intPool.intUpto;
      intPool.intUpto += streamCount;
    
      postingsArray.intStarts[termID] = intUptoStart + intPool.intOffset;
    
      for(int i=0;i<streamCount;i++) {
        final int upto = bytePool.newSlice(ByteBlockPool.FIRST_LEVEL_SIZE);
        intUptos[intUptoStart+i] = upto + bytePool.byteOffset;
      }
      postingsArray.byteStarts[termID] = intUptos[intUptoStart];
    
      newTerm(termID);
    
    } else {
      termID = (-termID)-1;
      int intStart = postingsArray.intStarts[termID];
      intUptos = intPool.buffers[intStart >> IntBlockPool.INT_BLOCK_SHIFT];
      intUptoStart = intStart & IntBlockPool.INT_BLOCK_MASK;
      addTerm(termID);
    }
    
    if (doNextCall) {
      nextPerField.add(postingsArray.textStarts[termID]);
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
add函数中的bytePool用于存放词的频率和位置信息，intPool用于存放每个词在bytePool中的偏移量，当空间不足时，两者都会通过nextBuffer函数分配新的空间。再深入，跟踪newTerm和addTerm函数可知，最终将Field的相关数据存储到类型为FreqProxPostingsArray的freqProxPostingsArray中，以及TermVectorsPostingsArray的termVectorsPostingsArray中，这两者就是最终的倒排序表，由于后面的代码涉及到太多的内存存储细节以及数据结构，这里就不往下看了，等有时间了再补充。

总结一下invert函数，该函数会通过分词器依次取出处理后的每个词，对于SimpleAnalyzer而言取出的就是一个个字符，然后将其存放在TermsHashPerField对象的IntBlockPool、ByteBlockPool以及倒排序表FreqProxPostingsArray和TermVectorsPostingsArray中。
————————————————
版权声明：本文为CSDN博主「二侠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/conansonic/article/details/51910145

lucene源码分析—flush
在前几章的分析中经常遇到flush操作，即当索引的相关数据存入内存中的某些数据结构后，再适当的实际就会通过flush函数将这些数据写入文件中，本章就开始分析flush函数，从DocumentsWriter的doflush函数开始分析，下面来看。
DocumentsWriter::doflush

  private boolean doFlush(DocumentsWriterPerThread flushingDWPT) throws IOException, AbortingException {
    boolean hasEvents = false;
    while (flushingDWPT != null) {
      hasEvents = true;
      boolean success = false;
      SegmentFlushTicket ticket = null;
      try {
        try {
          ticket = ticketQueue.addFlushTicket(flushingDWPT);

          final int flushingDocsInRam = flushingDWPT.getNumDocsInRAM();
          boolean dwptSuccess = false;
          try {
            final FlushedSegment newSegment = flushingDWPT.flush();
            ticketQueue.addSegment(ticket, newSegment);
            dwptSuccess = true;
          } finally {
            subtractFlushedNumDocs(flushingDocsInRam);
            if (!flushingDWPT.pendingFilesToDelete().isEmpty()) {
              putEvent(new DeleteNewFilesEvent(flushingDWPT.pendingFilesToDelete()));
              hasEvents = true;
            }
            if (!dwptSuccess) {
              putEvent(new FlushFailedEvent(flushingDWPT.getSegmentInfo()));
              hasEvents = true;
            }
          }
          success = true;
        } finally {
          if (!success && ticket != null) {
            ticketQueue.markTicketFailed(ticket);
          }
        }
    
        if (ticketQueue.getTicketCount() >= perThreadPool.getActiveThreadStateCount()) {
          putEvent(ForcedPurgeEvent.INSTANCE);
          break;
        }
      } finally {
        flushControl.doAfterFlush(flushingDWPT);
      }
    
      flushingDWPT = flushControl.nextPendingFlush();
    }
    if (hasEvents) {
      putEvent(MergePendingEvent.INSTANCE);
    }
    final double ramBufferSizeMB = config.getRAMBufferSizeMB();
    if (ramBufferSizeMB != IndexWriterConfig.DISABLE_AUTO_FLUSH &&
        flushControl.getDeleteBytesUsed() > (1024*1024*ramBufferSizeMB/2)) {
      hasEvents = true;
      if (!this.applyAllDeletes(deleteQueue)) {
        putEvent(ApplyDeletesEvent.INSTANCE);
      }
    }
    
    return hasEvents;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
传入的参数flushingDWPT是DocumentsWriterPerThread类型代表一个索引文档的线程。
ticketQueue被定义为DocumentsWriterFlushQueue，用来同步多个flush线程，其addFlushTicket定义如下，
DocumentsWriter::doflush->DocumentsWriterFlushQueue::addFlushTicket

  synchronized SegmentFlushTicket addFlushTicket(DocumentsWriterPerThread dwpt) {
    incTickets();
    boolean success = false;
    try {
      final SegmentFlushTicket ticket = new SegmentFlushTicket(dwpt.prepareFlush());
      queue.add(ticket);
      success = true;
      return ticket;
    } finally {
      if (!success) {
        decTickets();
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
addFlushTicket函数首先通过incTickets增加计数。prepareFlush操作在flush还没开始前将一些被标记的文档删除。该函数主要创建一个SegmentFlushTicket并添加进内部队列queue中。。

回到DocumentsWriter的doflush函数中，该函数继续通过getNumDocsInRAM获得在内存中的文档数，然后调用DocumentsWriterPerThread的flush函数继续进行。
DocumentsWriter::doflush->DocumentsWriterPerThread::flush

  FlushedSegment flush() throws IOException, AbortingException {
    segmentInfo.setMaxDoc(numDocsInRAM);
    final SegmentWriteState flushState = new SegmentWriteState(infoStream, directory, segmentInfo, fieldInfos.finish(), pendingUpdates, new IOContext(new FlushInfo(numDocsInRAM, bytesUsed())));
    final double startMBUsed = bytesUsed() / 1024. / 1024.;

    if (pendingUpdates.docIDs.size() > 0) {
      flushState.liveDocs = codec.liveDocsFormat().newLiveDocs(numDocsInRAM);
      for(int delDocID : pendingUpdates.docIDs) {
        flushState.liveDocs.clear(delDocID);
      }
      flushState.delCountOnFlush = pendingUpdates.docIDs.size();
      pendingUpdates.bytesUsed.addAndGet(-pendingUpdates.docIDs.size() * BufferedUpdates.BYTES_PER_DEL_DOCID);
      pendingUpdates.docIDs.clear();
    }
    
    if (aborted) {
      return null;
    }
    
    long t0 = System.nanoTime();
    
    try {
      consumer.flush(flushState);
      pendingUpdates.terms.clear();
      segmentInfo.setFiles(new HashSet<>(directory.getCreatedFiles()));
    
      final SegmentCommitInfo segmentInfoPerCommit = new SegmentCommitInfo(segmentInfo, 0, -1L, -1L, -1L);
    
      final BufferedUpdates segmentDeletes;
      if (pendingUpdates.queries.isEmpty() && pendingUpdates.numericUpdates.isEmpty() && pendingUpdates.binaryUpdates.isEmpty()) {
        pendingUpdates.clear();
        segmentDeletes = null;
      } else {
        segmentDeletes = pendingUpdates;
      }
    
      FlushedSegment fs = new FlushedSegment(segmentInfoPerCommit, flushState.fieldInfos, segmentDeletes, flushState.liveDocs, flushState.delCountOnFlush);
      sealFlushedSegment(fs);
    
      return fs;
    } catch (Throwable th) {
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
flush函数的pendingUpdates保存了待删除或更新的文档ID。假设待删除或更新的文档数大于0，就要标记处这些文档，接下来的codec被定义为Lucene60Codec，往下跟踪可知liveDocsFormat函数返回Lucene50LiveDocsFormat，Lucene50LiveDocsFormat的newLiveDocs函数创建FixedBitSet用来标记待删除或更新的文档ID。再往下的consumer在创建函数中被定义为DefaultIndexingChain，下面开始重点看DefaultIndexingChain的flush函数。

DefaultIndexingChain的flush函数
DefaultIndexingChain的flush函数代码如下所示，
DocumentsWriter::doflush->DocumentsWriterPerThread::flush->DefaultIndexingChain::flush

  public void flush(SegmentWriteState state) throws IOException, AbortingException {

    int maxDoc = state.segmentInfo.maxDoc();
    long t0 = System.nanoTime();
    writeNorms(state);
    
    t0 = System.nanoTime();
    writeDocValues(state);
    
    t0 = System.nanoTime();
    writePoints(state);
    
    t0 = System.nanoTime();
    initStoredFieldsWriter();
    fillStoredFields(maxDoc);
    storedFieldsWriter.finish(state.fieldInfos, maxDoc);
    storedFieldsWriter.close();
    
    t0 = System.nanoTime();
    Map<String,TermsHashPerField> fieldsToFlush = new HashMap<>();
    for (int i=0;i<fieldHash.length;i++) {
      PerField perField = fieldHash[i];
      while (perField != null) {
        if (perField.invertState != null) {
          fieldsToFlush.put(perField.fieldInfo.name, perField.termsHashPerField);
        }
        perField = perField.next;
      }
    }
    
    termsHash.flush(fieldsToFlush, state);
    
    t0 = System.nanoTime();
    docWriter.codec.fieldInfosFormat().write(state.directory, state.segmentInfo, "", state.fieldInfos, IOContext.DEFAULT);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
参数state中的segmentInfo是DocumentsWriterPerThread构造函数中创建的SegmentInfo，保存了相应的段信息，maxDoc函数返回目前在内存中的文档树。DefaultIndexingChain的flush函数接下来通过writeNorms函数将norm信息写入.nvm和.nvd文件中。

DefaultIndexingChain的writeNorms函数
DefaultIndexingChain::flush->DefaultIndexingChain::writeNorms

  private void writeNorms(SegmentWriteState state) throws IOException {
    boolean success = false;
    NormsConsumer normsConsumer = null;
    try {
      if (state.fieldInfos.hasNorms()) {
        NormsFormat normsFormat = state.segmentInfo.getCodec().normsFormat();
        normsConsumer = normsFormat.normsConsumer(state);

        for (FieldInfo fi : state.fieldInfos) {
          PerField perField = getPerField(fi.name);
          assert perField != null;
    
          if (fi.omitsNorms() == false && fi.getIndexOptions() != IndexOptions.NONE) {
            perField.norms.finish(state.segmentInfo.maxDoc());
            perField.norms.flush(state, normsConsumer);
          }
        }
      }
      success = true;
    } finally {
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
Lucene60Codec的normsFormat函数最终返回Lucene53NormsFormat，对应的normsConsumer函数返回一个Lucene53NormsConsumer。
DefaultIndexingChain::flush->writeNorms->Lucene53NormsFormat::normsConsumer

  public NormsConsumer normsConsumer(SegmentWriteState state) throws IOException {
    return new Lucene53NormsConsumer(state, DATA_CODEC, DATA_EXTENSION, METADATA_CODEC, METADATA_EXTENSION);
  }

  Lucene53NormsConsumer(SegmentWriteState state, String dataCodec, String dataExtension, String metaCodec, String metaExtension) throws IOException {
    boolean success = false;
    try {
      String dataName = IndexFileNames.segmentFileName(state.segmentInfo.name, state.segmentSuffix, dataExtension);
      data = state.directory.createOutput(dataName, state.context);
      CodecUtil.writeIndexHeader(data, dataCodec, VERSION_CURRENT, state.segmentInfo.getId(), state.segmentSuffix);
      String metaName = IndexFileNames.segmentFileName(state.segmentInfo.name, state.segmentSuffix, metaExtension);
      meta = state.directory.createOutput(metaName, state.context);
      CodecUtil.writeIndexHeader(meta, metaCodec, VERSION_CURRENT, state.segmentInfo.getId(), state.segmentSuffix);
      maxDoc = state.segmentInfo.maxDoc();
      success = true;
    } finally {

    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
IndexFileNames的segmentFileName函数会根据传入的参数段名(例如_0)和拓展名(例如.nvd)构造文件名_0.nvd。接着通过FSDirectory的createOutput创建输出流，代码如下，
DefaultIndexingChain::flush->writeNorms->Lucene53NormsFormat::normsConsumer->TrackingDirectoryWrapper::createOutput

  public IndexOutput createOutput(String name, IOContext context) throws IOException {
    IndexOutput output = in.createOutput(name, context);
    createdFileNames.add(name);
    return output;
  }

  public IndexOutput createOutput(String name, IOContext context) throws IOException {
    ensureOpen();

    pendingDeletes.remove(name);
    maybeDeletePendingFiles();
    return new FSIndexOutput(name);
  }

  public FSIndexOutput(String name) throws IOException {
    this(name, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING, StandardOpenOption.WRITE);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
createOutput函数内部会根据文件名创建一个FSIndexOutput并返回。

回到Lucene53NormsFormat的normsConsumer函数中，接下来就通过writeIndexHeader向文件写入头信息。
DefaultIndexingChain::flush->writeNorms->Lucene53NormsFormat::normsConsumer->CodecUtil::writeIndexHeader

  public static void writeIndexHeader(DataOutput out, String codec, int version, byte[] id, String suffix) throws IOException {
    writeHeader(out, codec, version);
    out.writeBytes(id, 0, id.length);
    BytesRef suffixBytes = new BytesRef(suffix);
    out.writeByte((byte) suffixBytes.length);
    out.writeBytes(suffixBytes.bytes, suffixBytes.offset, suffixBytes.length);
  }

  public static void writeHeader(DataOutput out, String codec, int version) throws IOException {
    BytesRef bytes = new BytesRef(codec);
    out.writeInt(CODEC_MAGIC);
    out.writeString(codec);
    out.writeInt(version);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
回到DefaultIndexingChain的writeNorms函数中，接下来通过getPerField获取PerField，其中的FieldInfo保存了域信息，代码如下
DefaultIndexingChain:flush->writeNorms->getPerField

  private PerField getPerField(String name) {
    final int hashPos = name.hashCode() & hashMask;
    PerField fp = fieldHash[hashPos];
    while (fp != null && !fp.fieldInfo.name.equals(name)) {
      fp = fp.next;
    }
    return fp;
  }
1
2
3
4
5
6
7
8
继续往下看，PerField中的成员变量norms在其构造函数中被定义为NormValuesWriter，对应的finish为空函数，而flush函数如下，
DefaultIndexingChain::flush->writeNorms->NormValuesWriter::flush

  public void flush(SegmentWriteState state, NormsConsumer normsConsumer) throws IOException {

    final int maxDoc = state.segmentInfo.maxDoc();
    final PackedLongValues values = pending.build();
    
    normsConsumer.addNormsField(fieldInfo,
                               new Iterable<Number>() {
                                 @Override
                                 public Iterator<Number> iterator() {
                                   return new NumericIterator(maxDoc, values);
                                 }
                               });
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
这里的normsConsumer就是Lucene53NormsConsumer，对应的addNormsField函数如下所示，
DefaultIndexingChain::flush->writeNorms->NormValuesWriter::flush->Lucene53NormsConsumer::addNormsField

  public void addNormsField(FieldInfo field, Iterable<Number> values) throws IOException {
    meta.writeVInt(field.number);
    long minValue = Long.MAX_VALUE;
    long maxValue = Long.MIN_VALUE;
    int count = 0;

    for (Number nv : values) {
      final long v = nv.longValue();
      minValue = Math.min(minValue, v);
      maxValue = Math.max(maxValue, v);
      count++;
    }
    
    if (minValue == maxValue) {
      addConstant(minValue);
    } else if (minValue >= Byte.MIN_VALUE && maxValue <= Byte.MAX_VALUE) {
      addByte1(values);
    } else if (minValue >= Short.MIN_VALUE && maxValue <= Short.MAX_VALUE) {
      addByte2(values);
    } else if (minValue >= Integer.MIN_VALUE && maxValue <= Integer.MAX_VALUE) {
      addByte4(values);
    } else {
      addByte8(values);
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
该函数再往下看就是将FieldInfo中的数据通过刚刚创建的FSIndexOutput写入到.nvd和.nvm文件中。

DefaultIndexingChain的writeDocValues函数
看完了writeNorms函数，接下来看writeDocValues函数，
DefaultIndexingChain::flush->writeDocValues

  private void writeDocValues(SegmentWriteState state) throws IOException {
    int maxDoc = state.segmentInfo.maxDoc();
    DocValuesConsumer dvConsumer = null;
    boolean success = false;
    try {
      for (int i=0;i<fieldHash.length;i++) {
        PerField perField = fieldHash[i];
        while (perField != null) {
          if (perField.docValuesWriter != null) {
            if (dvConsumer == null) {
              DocValuesFormat fmt = state.segmentInfo.getCodec().docValuesFormat();
              dvConsumer = fmt.fieldsConsumer(state);
            }
            perField.docValuesWriter.finish(maxDoc);
            perField.docValuesWriter.flush(state, dvConsumer);
            perField.docValuesWriter = null;
          } 
          perField = perField.next;
        }
      }
      success = true;
    } finally {

    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
和前面writeNorms函数中的分析类似，writeDocValues函数遍历得到每个PerField，PerField中的docValuesWriter根据不同的Field值域类型被定义为NumericDocValuesWriter、BinaryDocValuesWriter、SortedDocValuesWriter、SortedNumericDocValuesWriter和SortedSetDocValuesWriter，代码如下，
DefaultIndexingChain::flush->indexDocValue

  private void indexDocValue(PerField fp, DocValuesType dvType, IndexableField field) throws IOException {

    if (fp.fieldInfo.getDocValuesType() == DocValuesType.NONE) {
      fieldInfos.globalFieldNumbers.setDocValuesType(fp.fieldInfo.number, fp.fieldInfo.name, dvType);
    }
    fp.fieldInfo.setDocValuesType(dvType);
    int docID = docState.docID;
    
    switch(dvType) {
    
      case NUMERIC:
        if (fp.docValuesWriter == null) {
          fp.docValuesWriter = new NumericDocValuesWriter(fp.fieldInfo, bytesUsed);
        }
        ((NumericDocValuesWriter) fp.docValuesWriter).addValue(docID, field.numericValue().longValue());
        break;
    
      case BINARY:
        if (fp.docValuesWriter == null) {
          fp.docValuesWriter = new BinaryDocValuesWriter(fp.fieldInfo, bytesUsed);
        }
        ((BinaryDocValuesWriter) fp.docValuesWriter).addValue(docID, field.binaryValue());
        break;
    
      case SORTED:
        if (fp.docValuesWriter == null) {
          fp.docValuesWriter = new SortedDocValuesWriter(fp.fieldInfo, bytesUsed);
        }
        ((SortedDocValuesWriter) fp.docValuesWriter).addValue(docID, field.binaryValue());
        break;
    
      case SORTED_NUMERIC:
        if (fp.docValuesWriter == null) {
          fp.docValuesWriter = new SortedNumericDocValuesWriter(fp.fieldInfo, bytesUsed);
        }
        ((SortedNumericDocValuesWriter) fp.docValuesWriter).addValue(docID, field.numericValue().longValue());
        break;
    
      case SORTED_SET:
        if (fp.docValuesWriter == null) {
          fp.docValuesWriter = new SortedSetDocValuesWriter(fp.fieldInfo, bytesUsed);
        }
        ((SortedSetDocValuesWriter) fp.docValuesWriter).addValue(docID, field.binaryValue());
        break;
    
      default:
        throw new AssertionError();
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
为了方便分析，下面假设PerField中的docValuesWriter被定义为BinaryDocValuesWriter。

回到writeDocValues函数中，再往下通过docValuesFormat函数返回一个PerFieldDocValuesFormat，并通过PerFieldDocValuesFormat的fieldsConsumer获得一个DocValuesConsumer。
DefaultIndexingChain::flush->writeDocValues->PerFieldDocValuesFormat::fieldsConsumer

  public final DocValuesConsumer fieldsConsumer(SegmentWriteState state) throws IOException {
    return new FieldsWriter(state);
  }
1
2
3
fieldsConsumer最后返回的其实是一个FieldsWriter。

回到DefaultIndexingChain的writeDocValues函数中，接下来继续调用docValuesWriter也即前面假设的BinaryDocValuesWriter的flush函数。
DefaultIndexingChain::flush->writeDocValues->BinaryDocValuesWriter::flush

  public void flush(SegmentWriteState state, DocValuesConsumer dvConsumer) throws IOException {
    final int maxDoc = state.segmentInfo.maxDoc();
    bytes.freeze(false);
    final PackedLongValues lengths = this.lengths.build();
    dvConsumer.addBinaryField(fieldInfo,
                              new Iterable<BytesRef>() {
                                @Override
                                public Iterator<BytesRef> iterator() {
                                   return new BytesIterator(maxDoc, lengths);
                                }
                              });
  }
1
2
3
4
5
6
7
8
9
10
11
12
BinaryDocValuesWriter的flush函数主要调用了FieldsWriter的addBinaryField函数添加FieldInfo中的数据。
DefaultIndexingChain::flush->writeDocValues->BinaryDocValuesWriter::flush->FieldsWriter::addBinaryField

    public void addBinaryField(FieldInfo field, Iterable<BytesRef> values) throws IOException {
      getInstance(field).addBinaryField(field, values);
    }
1
2
3
addBinaryField首先通过getInstance函数最终获得一个Lucene54DocValuesConsumer。
DefaultIndexingChain::flush->writeDocValues->BinaryDocValuesWriter::flush->FieldsWriter::addBinaryField->getInstance

    private DocValuesConsumer getInstance(FieldInfo field) throws IOException {
      DocValuesFormat format = null;
      if (field.getDocValuesGen() != -1) {
        final String formatName = field.getAttribute(PER_FIELD_FORMAT_KEY);
        if (formatName != null) {
          format = DocValuesFormat.forName(formatName);
        }
      }
      if (format == null) {
        format = getDocValuesFormatForField(field.name);
      }
    
      final String formatName = format.getName();
    
      String previousValue = field.putAttribute(PER_FIELD_FORMAT_KEY, formatName);
    
      Integer suffix = null;
    
      ConsumerAndSuffix consumer = formats.get(format);
      if (consumer == null) {
        if (field.getDocValuesGen() != -1) {
          final String suffixAtt = field.getAttribute(PER_FIELD_SUFFIX_KEY);
          if (suffixAtt != null) {
            suffix = Integer.valueOf(suffixAtt);
          }
        }
    
        if (suffix == null) {
          suffix = suffixes.get(formatName);
          if (suffix == null) {
            suffix = 0;
          } else {
            suffix = suffix + 1;
          }
        }
        suffixes.put(formatName, suffix);
    
        final String segmentSuffix = getFullSegmentSuffix(segmentWriteState.segmentSuffix,
                                                          getSuffix(formatName, Integer.toString(suffix)));
        consumer = new ConsumerAndSuffix();
        consumer.consumer = format.fieldsConsumer(new SegmentWriteState(segmentWriteState, segmentSuffix));
        consumer.suffix = suffix;
        formats.put(format, consumer);
      } else {
        suffix = consumer.suffix;
      }
    
      previousValue = field.putAttribute(PER_FIELD_SUFFIX_KEY, Integer.toString(suffix));
    
      return consumer.consumer;
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
假设是第一次进入该函数，format会通过getDocValuesFormatForField函数被定义为Lucene54DocValuesFormat，然后通过Lucene54DocValuesFormat的fieldsConsumer函数构造一个Lucene54DocValuesConsumer并返回。
DefaultIndexingChain::flush->writeDocValues->BinaryDocValuesWriter::flush->FieldsWriter::addBinaryField->getInstance->Lucene54DocValuesFormat::fieldsConsumer

  public DocValuesConsumer fieldsConsumer(SegmentWriteState state) throws IOException {
    return new Lucene54DocValuesConsumer(state, DATA_CODEC, DATA_EXTENSION, META_CODEC, META_EXTENSION);
  }

  public Lucene54DocValuesConsumer(SegmentWriteState state, String dataCodec, String dataExtension, String metaCodec, String metaExtension) throws IOException {
    boolean success = false;
    try {
      String dataName = IndexFileNames.segmentFileName(state.segmentInfo.name, state.segmentSuffix, dataExtension);
      data = state.directory.createOutput(dataName, state.context);
      CodecUtil.writeIndexHeader(data, dataCodec, Lucene54DocValuesFormat.VERSION_CURRENT, state.segmentInfo.getId(), state.segmentSuffix);
      String metaName = IndexFileNames.segmentFileName(state.segmentInfo.name, state.segmentSuffix, metaExtension);
      meta = state.directory.createOutput(metaName, state.context);
      CodecUtil.writeIndexHeader(meta, metaCodec, Lucene54DocValuesFormat.VERSION_CURRENT, state.segmentInfo.getId(), state.segmentSuffix);
      maxDoc = state.segmentInfo.maxDoc();
      success = true;
    } finally {

    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
和前面的分析类似，这里创建了.dvd和.dvm的文件输出流并写入相应的头信息。
Lucene54DocValuesConsumer的addBinaryField函数就不往下看了，就是调用文件输出流写入相应的数据。

DefaultIndexingChain的writePoints函数
DefaultIndexingChain::flush->writePoints

  private void writePoints(SegmentWriteState state) throws IOException {
    PointsWriter pointsWriter = null;
    boolean success = false;
    try {
      for (int i=0;i<fieldHash.length;i++) {
        PerField perField = fieldHash[i];
        while (perField != null) {
          if (perField.pointValuesWriter != null) {
            if (pointsWriter == null) {
              PointsFormat fmt = state.segmentInfo.getCodec().pointsFormat();
              pointsWriter = fmt.fieldsWriter(state);
            }

            perField.pointValuesWriter.flush(state, pointsWriter);
            perField.pointValuesWriter = null;
          } 
          perField = perField.next;
        }
      }
      if (pointsWriter != null) {
        pointsWriter.finish();
      }
      success = true;
    } finally {
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
和前面的分析类似，writePoints函数中的pointsFormat最终返回Lucene60PointsFormat，然后通过其fieldsWriter函数获得一个Lucene60PointsWriter。

DefaultIndexingChain::writePoints->Lucene60PointsFormat::fieldsWriter

  public PointsWriter fieldsWriter(SegmentWriteState state) throws IOException {
    return new Lucene60PointsWriter(state);
  }
1
2
3
PerField中的成员变量pointValuesWriter被设置为PointValuesWriter，对应的flush函数如下所示，
DefaultIndexingChain::writePoints->PointValuesWriter::flush

  public void flush(SegmentWriteState state, PointsWriter writer) throws IOException {

    writer.writeField(fieldInfo,
                      new PointsReader() {
                        @Override
                        public void intersect(String fieldName, IntersectVisitor visitor) throws IOException {
                          if (fieldName.equals(fieldInfo.name) == false) {
                            throw new IllegalArgumentException();
                          }
                          for(int i=0;i<numPoints;i++) {
                            bytes.readBytes(packedValue.length * i, packedValue, 0, packedValue.length);
                            visitor.visit(docIDs[i], packedValue);
                          }
                        }
    
                        @Override
                        public void checkIntegrity() {
                          throw new UnsupportedOperationException();
                        }
    
                        @Override
                        public long ramBytesUsed() {
                          return 0L;
                        }
    
                        @Override
                        public void close() {
                        }
    
                        @Override
                        public byte[] getMinPackedValue(String fieldName) {
                          throw new UnsupportedOperationException();
                        }
    
                        @Override
                        public byte[] getMaxPackedValue(String fieldName) {
                          throw new UnsupportedOperationException();
                        }
    
                        @Override
                        public int getNumDimensions(String fieldName) {
                          throw new UnsupportedOperationException();
                        }
    
                        @Override
                        public int getBytesPerDimension(String fieldName) {
                          throw new UnsupportedOperationException();
                        }
    
                        @Override
                        public long size(String fieldName) {
                          return numPoints;
                        }
    
                        @Override
                        public int getDocCount(String fieldName) {
                          return numDocs;
                        }
                      });
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
PointValuesWriter的flush函数继而会调用Lucene60PointsWriter的writeField函数，如下所示，
DefaultIndexingChain::writePoints->PointValuesWriter::flush->Lucene60PointsWriter::writeField

  public void writeField(FieldInfo fieldInfo, PointsReader values) throws IOException {

    boolean singleValuePerDoc = values.size(fieldInfo.name) == values.getDocCount(fieldInfo.name);
    
    try (BKDWriter writer = new BKDWriter(writeState.segmentInfo.maxDoc(),
                                          writeState.directory,
                                          writeState.segmentInfo.name,
                                          fieldInfo.getPointDimensionCount(),
                                          fieldInfo.getPointNumBytes(),
                                          maxPointsInLeafNode,
                                          maxMBSortInHeap,
                                          values.size(fieldInfo.name),
                                          singleValuePerDoc)) {
    
      values.intersect(fieldInfo.name, new IntersectVisitor() {
          @Override
          public void visit(int docID) {
            throw new IllegalStateException();
          }
    
          public void visit(int docID, byte[] packedValue) throws IOException {
            writer.add(packedValue, docID);
          }
    
          @Override
          public Relation compare(byte[] minPackedValue, byte[] maxPackedValue) {
            return Relation.CELL_CROSSES_QUERY;
          }
        });
    
      if (writer.getPointCount() > 0) {
        indexFPs.put(fieldInfo.name, writer.finish(dataOut));
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
结合PointValuesWriter的flush函数中PointsReader的定义，以及Lucene60PointsWriter中writeField函数中visit函数的定义，writeField函数最终会调用BKDWriter的add函数，BKD是一种数据结构，add函数定义如下，

  public void add(byte[] packedValue, int docID) throws IOException {

    if (pointCount >= maxPointsSortInHeap) {
      if (offlinePointWriter == null) {
        spillToOffline();
      }
      offlinePointWriter.append(packedValue, pointCount, docID);
    } else {
      heapPointWriter.append(packedValue, pointCount, docID);
    }
    
    if (pointCount == 0) {
      System.arraycopy(packedValue, 0, minPackedValue, 0, packedBytesLength);
      System.arraycopy(packedValue, 0, maxPackedValue, 0, packedBytesLength);
    } else {
      for(int dim=0;dim<numDims;dim++) {
        int offset = dim*bytesPerDim;
        if (StringHelper.compare(bytesPerDim, packedValue, offset, minPackedValue, offset) < 0) {
          System.arraycopy(packedValue, offset, minPackedValue, offset, bytesPerDim);
        }
        if (StringHelper.compare(bytesPerDim, packedValue, offset, maxPackedValue, offset) > 0) {
          System.arraycopy(packedValue, offset, maxPackedValue, offset, bytesPerDim);
        }
      }
    }
    
    pointCount++;
    docsSeen.set(docID);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
成员变量heapPointWriter的类型为HeapPointWriter，用来将数据写入内存；offlinePointWriter的类型为OfflinePointWriter，用来将数据写入硬盘。一开始，数据将会通过HeapPointWriter被写入内存，当内存中的数据超过maxPointsSortInHeap时，就调用spillToOffline函数进行切换。

  private void spillToOffline() throws IOException {

    offlinePointWriter = new OfflinePointWriter(tempDir, tempFileNamePrefix, packedBytesLength, longOrds, "spill", 0, singleValuePerDoc);
    tempInput = offlinePointWriter.out;
    PointReader reader = heapPointWriter.getReader(0, pointCount);
    for(int i=0;i<pointCount;i++) {
      boolean hasNext = reader.next();
      offlinePointWriter.append(reader.packedValue(), i, heapPointWriter.docIDs[i]);
    }
    
    heapPointWriter = null;
  }
1
2
3
4
5
6
7
8
9
10
11
12
OfflinePointWriter的构造函数会创建类似”段名bkd_spill临时文件数量.tmp”的文件名对应的输出流，然后通过append函数复制HeapPointWriter中的数据。

回到DefaultIndexingChain的writePoints函数中，接下来通过finish函数将数据写入最终的.dim文件中，代码如下，

  public void finish() throws IOException {
    finished = true;
    CodecUtil.writeFooter(dataOut);

    String indexFileName = IndexFileNames.segmentFileName(writeState.segmentInfo.name,
                                                          writeState.segmentSuffix,
                                                          Lucene60PointsFormat.INDEX_EXTENSION);
    try (IndexOutput indexOut = writeState.directory.createOutput(indexFileName, writeState.context)) {
      CodecUtil.writeIndexHeader(indexOut,
                                 Lucene60PointsFormat.META_CODEC_NAME,
                                 Lucene60PointsFormat.INDEX_VERSION_CURRENT,
                                 writeState.segmentInfo.getId(),
                                 writeState.segmentSuffix);
      int count = indexFPs.size();
      indexOut.writeVInt(count);
      for(Map.Entry<String,Long> ent : indexFPs.entrySet()) {
        FieldInfo fieldInfo = writeState.fieldInfos.fieldInfo(ent.getKey());
        indexOut.writeVInt(fieldInfo.number);
        indexOut.writeVLong(ent.getValue());
      }
      CodecUtil.writeFooter(indexOut);
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
数据写入.fdt以及.fdx文件
继续看DefaultIndexingChain的flush函数，接下来通过initStoredFieldsWriter函数初始化一个StoredFieldsWriter，代码如下
DefaultIndexingChain::flush->initStoredFieldsWriter

  private void initStoredFieldsWriter() throws IOException {
    if (storedFieldsWriter == null) {
      storedFieldsWriter = docWriter.codec.storedFieldsFormat().fieldsWriter(docWriter.directory, docWriter.getSegmentInfo(), IOContext.DEFAULT);
    }
  }
1
2
3
4
5
storedFieldsFormat函数返回Lucene50StoredFieldsFormat，其fieldsWriter函数会接着调用CompressingStoredFieldsFormat的fieldsWriter函数，最后返回CompressingStoredFieldsWriter，
DefaultIndexingChain::flush->initStoredFieldsWriter->Lucene60Codec::storedFieldsFormat->Lucene50StoredFieldsFormat::fieldsWriter->CompressingStoredFieldsFormat::fieldsWriter

  public StoredFieldsWriter fieldsWriter(Directory directory, SegmentInfo si,
      IOContext context) throws IOException {
    return new CompressingStoredFieldsWriter(directory, si, segmentSuffix, context,
        formatName, compressionMode, chunkSize, maxDocsPerChunk, blockSize);
  }
1
2
3
4
5
CompressingStoredFieldsWriter的构造函数如下所示，
DefaultIndexingChain::flush->initStoredFieldsWriter->Lucene60Codec::storedFieldsFormat->Lucene50StoredFieldsFormat::fieldsWriter->CompressingStoredFieldsFormat::fieldsWriter->CompressingStoredFieldsWriter::CompressingStoredFieldsWriter

  public CompressingStoredFieldsWriter(Directory directory, SegmentInfo si, String segmentSuffix, IOContext context, String formatName, CompressionMode compressionMode, int chunkSize, int maxDocsPerChunk, int blockSize) throws IOException {
    assert directory != null;
    this.segment = si.name;
    this.compressionMode = compressionMode;
    this.compressor = compressionMode.newCompressor();
    this.chunkSize = chunkSize;
    this.maxDocsPerChunk = maxDocsPerChunk;
    this.docBase = 0;
    this.bufferedDocs = new GrowableByteArrayDataOutput(chunkSize);
    this.numStoredFields = new int[16];
    this.endOffsets = new int[16];
    this.numBufferedDocs = 0;

    boolean success = false;
    IndexOutput indexStream = directory.createOutput(IndexFileNames.segmentFileName(segment, segmentSuffix, FIELDS_INDEX_EXTENSION), context);
    try {
      fieldsStream = directory.createOutput(IndexFileNames.segmentFileName(segment, segmentSuffix, FIELDS_EXTENSION),context);
    
      final String codecNameIdx = formatName + CODEC_SFX_IDX;
      final String codecNameDat = formatName + CODEC_SFX_DAT;
      CodecUtil.writeIndexHeader(indexStream, codecNameIdx, VERSION_CURRENT, si.getId(), segmentSuffix);
      CodecUtil.writeIndexHeader(fieldsStream, codecNameDat, VERSION_CURRENT, si.getId(), segmentSuffix);
    
      indexWriter = new CompressingStoredFieldsIndexWriter(indexStream, blockSize);
      indexStream = null;
    
      fieldsStream.writeVInt(chunkSize);
      fieldsStream.writeVInt(PackedInts.VERSION_CURRENT);
    
      success = true;
    } finally {
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
CompressingStoredFieldsWriter的构造函数创建了.fdt和.fdx两个文件并建立输出流。

回到DefaultIndexingChain的flush函数中，接下来调用fillStoredFields，进而调用startStoredFields以及finishStoredFields函数，startStoredFields函数会调用刚刚上面构造的CompressingStoredFieldsWriter的startDocument函数，该函数为空，finishStoredFields函数会调用CompressingStoredFieldsWriter的finishDocument函数，代码如下
DefaultIndexingChain::flush->fillStoredFields->startStoredFields->CompressingStoredFieldsWriter::finishDocument

  public void finishDocument() throws IOException {
    if (numBufferedDocs == this.numStoredFields.length) {
      final int newLength = ArrayUtil.oversize(numBufferedDocs + 1, 4);
      this.numStoredFields = Arrays.copyOf(this.numStoredFields, newLength);
      endOffsets = Arrays.copyOf(endOffsets, newLength);
    }
    this.numStoredFields[numBufferedDocs] = numStoredFieldsInDoc;
    numStoredFieldsInDoc = 0;
    endOffsets[numBufferedDocs] = bufferedDocs.length;
    ++numBufferedDocs;
    if (triggerFlush()) {
      flush();
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
经过一些值得计算以及设置后，finishDocument通过triggerFlush函数判断是否需要进行flush操作，该flush函数定义在CompressingStoredFieldsWriter中，
DefaultIndexingChain::flush->fillStoredFields->startStoredFields->CompressingStoredFieldsWriter::finishDocument->flush

  private void flush() throws IOException {
    indexWriter.writeIndex(numBufferedDocs, fieldsStream.getFilePointer());

    final int[] lengths = endOffsets;
    for (int i = numBufferedDocs - 1; i > 0; --i) {
      lengths[i] = endOffsets[i] - endOffsets[i - 1];
    }
    final boolean sliced = bufferedDocs.length >= 2 * chunkSize;
    writeHeader(docBase, numBufferedDocs, numStoredFields, lengths, sliced);
    
    if (sliced) {
      for (int compressed = 0; compressed < bufferedDocs.length; compressed += chunkSize) {
        compressor.compress(bufferedDocs.bytes, compressed, Math.min(chunkSize, bufferedDocs.length - compressed), fieldsStream);
      }
    } else {
      compressor.compress(bufferedDocs.bytes, 0, bufferedDocs.length, fieldsStream);
    }
    
    docBase += numBufferedDocs;
    numBufferedDocs = 0;
    bufferedDocs.length = 0;
    numChunks++;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
fieldsStream是根据fdt文件创建的FSIndexOutput，对应的函数getFilePointer返回当前可以插入数据的位置。indexWriter被定义为CompressingStoredFieldsIndexWriter，是在CompressingStoredFieldsWriter的构造函数中创建的，其writeIndex函数代码如下，
DefaultIndexingChain::flush->fillStoredFields->startStoredFields->CompressingStoredFieldsWriter::finishDocument->flush->CompressingStoredFieldsIndexWriter::writeIndex

  void writeIndex(int numDocs, long startPointer) throws IOException {
    if (blockChunks == blockSize) {
      writeBlock();
      reset();
    }

    if (firstStartPointer == -1) {
      firstStartPointer = maxStartPointer = startPointer;
    }
    
    docBaseDeltas[blockChunks] = numDocs;
    startPointerDeltas[blockChunks] = startPointer - maxStartPointer;
    
    ++blockChunks;
    blockDocs += numDocs;
    totalDocs += numDocs;
    maxStartPointer = startPointer;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
当blockChunks增大到blockSize时，启动writeBlock函数写入索引，writeBlock函数最终向.fdx中写入索引信息，这里就不往下看了。

回到CompressingStoredFieldsWriter的flush函数中，接下来通过writeHeader函数向.fdt文件中写入头信息。再往下的compressor被定义为LZ4FastCompressor，其compress函数将缓存bufferedDocs.bytes中的数据写入到fieldStream，也即.fdt文件对应的输出流中。

再回到DefaultIndexingChain的flush函数中，接下来看finish函数，
DefaultIndexingChain::flush->CompressingStoredFieldsWriter::finish

  public void finish(FieldInfos fis, int numDocs) throws IOException {
    if (numBufferedDocs > 0) {
      flush();
      numDirtyChunks++;
    } else {

    }
    
    indexWriter.finish(numDocs, fieldsStream.getFilePointer());
    fieldsStream.writeVLong(numChunks);
    fieldsStream.writeVLong(numDirtyChunks);
    CodecUtil.writeFooter(fieldsStream);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
CompressingStoredFieldsWriter的flush函数前面已经分析过了，这里的fieldsStream代表.fdt文件的输出流，最终将索引数据写入到fdt文件中。
DefaultIndexingChain::flush->CompressingStoredFieldsWriter::finish->CompressingStoredFieldsIndexWriter::finish

  void finish(int numDocs, long maxPointer) throws IOException {
    if (blockChunks > 0) {
      writeBlock();
    }
    fieldsIndexOut.writeVInt(0);
    fieldsIndexOut.writeVLong(maxPointer);
    CodecUtil.writeFooter(fieldsIndexOut);
  }
1
2
3
4
5
6
7
8
这里最重要的是writeBlock函数，其最终也是将索引数据写入.fdx文件中。

数据写入.tvd以及.tvx文件
回到DefaultIndexingChain的flush函数中，接下来使用fieldsToFlush封装了fieldHash函数中的域信息，然后调用termsHash的flush函数，termsHash在DefaultIndexingChain的构造函数中被定义为FreqProxTermsWriter，其flush函数代码如下，
DefaultIndexingChain::flush->FreqProxTermsWriter::flush

  public void flush(Map<String,TermsHashPerField> fieldsToFlush, final SegmentWriteState state) throws IOException {
    super.flush(fieldsToFlush, state);
    List<FreqProxTermsWriterPerField> allFields = new ArrayList<>();

    for (TermsHashPerField f : fieldsToFlush.values()) {
      final FreqProxTermsWriterPerField perField = (FreqProxTermsWriterPerField) f;
      if (perField.bytesHash.size() > 0) {
        perField.sortPostings();
        allFields.add(perField);
      }
    }
    
    CollectionUtil.introSort(allFields);
    
    Fields fields = new FreqProxFields(allFields);
    applyDeletes(state, fields);
    
    FieldsConsumer consumer = state.segmentInfo.getCodec().postingsFormat().fieldsConsumer(state);
    boolean success = false;
    try {
      consumer.write(fields);
      success = true;
    } finally {
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
首先，FreqProxTermsWriter的父类的flush函数最终会调用TermVectorsConsumer的flush函数，定义如下，
DefaultIndexingChain::flush->TermVectorsConsumer::flush

  void flush(Map<String, TermsHashPerField> fieldsToFlush, final SegmentWriteState state) throws IOException {
    if (writer != null) {
      int numDocs = state.segmentInfo.maxDoc();
      try {
        fill(numDocs);
        writer.finish(state.fieldInfos, numDocs);
      } finally {

      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
TermVectorsConsumer的flush函数中的writer为Lucene50TermVectorsFormat中创建的CompressingTermVectorsWriter，下面主要看其的finish函数，
DefaultIndexingChain::flush->TermVectorsConsumer::flush->CompressingTermVectorsWriter::finish

  public void finish(FieldInfos fis, int numDocs) throws IOException {
    if (!pendingDocs.isEmpty()) {
      flush();
      numDirtyChunks++;
    }
    indexWriter.finish(numDocs, vectorsStream.getFilePointer());
    vectorsStream.writeVLong(numChunks);
    vectorsStream.writeVLong(numDirtyChunks);
    CodecUtil.writeFooter(vectorsStream);
  }
1
2
3
4
5
6
7
8
9
10
这里的indexWriter是CompressingStoredFieldsIndexWriter，其finish前面分析过了，该函数最终将索引数据写入.tvx文件中，下面来看flush函数，

DefaultIndexingChain::flush->TermVectorsConsumer::flush->CompressingTermVectorsWriter::finish->flush

  private void flush() throws IOException {
    final int chunkDocs = pendingDocs.size();
    assert chunkDocs > 0 : chunkDocs;

    indexWriter.writeIndex(chunkDocs, vectorsStream.getFilePointer());
    
    final int docBase = numDocs - chunkDocs;
    vectorsStream.writeVInt(docBase);
    vectorsStream.writeVInt(chunkDocs);
    
    final int totalFields = flushNumFields(chunkDocs);
    if (totalFields > 0) {
      final int[] fieldNums = flushFieldNums();
      flushFields(totalFields, fieldNums);
      flushFlags(totalFields, fieldNums);
      flushNumTerms(totalFields);
      flushTermLengths();
      flushTermFreqs();
      flushPositions();
      flushOffsets(fieldNums);
      flushPayloadLengths();
    
      compressor.compress(termSuffixes.bytes, 0, termSuffixes.length, vectorsStream);
    }
    
    pendingDocs.clear();
    curDoc = null;
    curField = null;
    termSuffixes.length = 0;
    numChunks++;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
indexWriter是CompressingStoredFieldsIndexWriter，writeIndex前面分析过了，最终将数据写入到.tvx文件中。
CompressingTermVectorsWriter的flush函数接下来调用flushNumFields向.tvd文件中写入索引信息，代码如下，
DefaultIndexingChain::flush->TermVectorsConsumer::flush->CompressingTermVectorsWriter::finish->flush->flushNumFields

  private int flushNumFields(int chunkDocs) throws IOException {
    if (chunkDocs == 1) {
      final int numFields = pendingDocs.getFirst().numFields;
      vectorsStream.writeVInt(numFields);
      return numFields;
    } else {
      writer.reset(vectorsStream);
      int totalFields = 0;
      for (DocData dd : pendingDocs) {
        writer.add(dd.numFields);
        totalFields += dd.numFields;
      }
      writer.finish();
      return totalFields;
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
writer和vectorsStream都是.tvd文件对应的输出流的封装，最终都是将索引写入该文件中。
再看CompressingTermVectorsWriter的flush函数，类似flushFieldNums函数的源码，后面的flushFields、flushFlags、flushNumTerms、flushTermLengths、flushTermFreqs、flushPositions、flushOffsets、flushPayloadLengths以及LZ4FastCompressor的compress函数都是最终将索引信息写入.tvd文件中，这里就不往下看了。

分析完了TermVectorsConsumer的flush函数后，在回头看FreqProxTermsWriter的flush函数，接下来的consumer被设置为FieldsWriter，下面来看其write函数，
DefaultIndexingChain::flush->FreqProxTermsWriter::flush->FieldsWriter::write

    public void write(Fields fields) throws IOException {
    
      Map<PostingsFormat,FieldsGroup> formatToGroups = new HashMap<>();
      Map<String,Integer> suffixes = new HashMap<>();
    
      for(String field : fields) {
        FieldInfo fieldInfo = writeState.fieldInfos.fieldInfo(field);
        final PostingsFormat format = getPostingsFormatForField(field);
    
        String formatName = format.getName();
    
        FieldsGroup group = formatToGroups.get(format);
        if (group == null) {
    
          Integer suffix = suffixes.get(formatName);
          if (suffix == null) {
            suffix = 0;
          } else {
            suffix = suffix + 1;
          }
          suffixes.put(formatName, suffix);
    
          String segmentSuffix = getFullSegmentSuffix(field,
                                                      writeState.segmentSuffix,
                                                      getSuffix(formatName, Integer.toString(suffix)));
          group = new FieldsGroup();
          group.state = new SegmentWriteState(writeState, segmentSuffix);
          group.suffix = suffix;
          formatToGroups.put(format, group);
        } else {
    
        }
        group.fields.add(field);
        String previousValue = fieldInfo.putAttribute(PER_FIELD_FORMAT_KEY, formatName);
    
        previousValue = fieldInfo.putAttribute(PER_FIELD_SUFFIX_KEY, Integer.toString(group.suffix));
      }
    
      boolean success = false;
      try {
        for(Map.Entry<PostingsFormat,FieldsGroup> ent : formatToGroups.entrySet()) {
          PostingsFormat format = ent.getKey();
          final FieldsGroup group = ent.getValue();
    
          Fields maskedFields = new FilterFields(fields) {
              @Override
              public Iterator<String> iterator() {
                return group.fields.iterator();
              }
            };
    
          FieldsConsumer consumer = format.fieldsConsumer(group.state);
          toClose.add(consumer);
          consumer.write(maskedFields);
        }
        success = true;
      } finally {
    
      }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
简单分析一下FieldsWriter的write函数，该函数最重要的部分是继续调用consumer的write函数。最后的format是通过getPostingsFormatForField设置为Lucene50PostingsFormat，其fieldsConsumer函数代码如下，
DefaultIndexingChain::flush->FreqProxTermsWriter::flush->FieldsWriter::write->Lucene50PostingsFormat::fieldsConsumer

  public FieldsConsumer fieldsConsumer(SegmentWriteState state) throws IOException {
    PostingsWriterBase postingsWriter = new Lucene50PostingsWriter(state);

    boolean success = false;
    try {
      FieldsConsumer ret = new BlockTreeTermsWriter(state, 
                                                    postingsWriter,
                                                    minTermBlockSize, 
                                                    maxTermBlockSize);
      success = true;
      return ret;
    } finally {
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
fieldsConsumer函数最终返回BlockTreeTermsWriter，然后调用其write函数，定义如下，
DefaultIndexingChain::flush->FreqProxTermsWriter::flush->FieldsWriter::write->BlockTreeTermsWriter::write

  public void write(Fields fields) throws IOException {
    String lastField = null;
    for(String field : fields) {
      lastField = field;

      Terms terms = fields.terms(field);
      FieldInfo fieldInfo = fieldInfos.fieldInfo(field);
    
      List<PrefixTerm> prefixTerms;
      if (minItemsInAutoPrefix != 0) {
        prefixTerms = new AutoPrefixTermsWriter(terms, minItemsInAutoPrefix, maxItemsInAutoPrefix).prefixes;
      } else {
        prefixTerms = null;
      }
    
      TermsEnum termsEnum = terms.iterator();
      TermsWriter termsWriter = new TermsWriter(fieldInfos.fieldInfo(field));
      int prefixTermUpto = 0;
      while (true) {
        BytesRef term = termsEnum.next();
        if (prefixTerms != null) {
          while (prefixTermUpto < prefixTerms.size() && (term == null || prefixTerms.get(prefixTermUpto).compareTo(term) <= 0)) {
            PrefixTerm prefixTerm = prefixTerms.get(prefixTermUpto);
            termsWriter.write(prefixTerm.term, getAutoPrefixTermsEnum(terms, prefixTerm), prefixTerm);
            prefixTermUpto++;
          }
        }
    
        if (term == null) {
          break;
        }
        termsWriter.write(term, termsEnum, null);
      }
      termsWriter.finish();
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
BlockTreeTermsWriter的write函数中的termsWriter被定义为TermsWriter。
TermsWriter的write函数内部会通过Lucene50PostingsWriter将数据信息写入.doc，.pos、.pay三个文件中。
TermsWriter的finish函数会通过其内部的writeBlocks函数将索引信息写入.tim、.tip中。

将数据写入.fnm文件
继续往下看DefaultIndexingChain的flush函数，fieldInfosFormat返回Lucene60FieldInfosFormat，
DefaultIndexingChain::flush->Lucene60FieldInfosFormat::write

  public void write(Directory directory, SegmentInfo segmentInfo, String segmentSuffix, FieldInfos infos, IOContext context) throws IOException {
    final String fileName = IndexFileNames.segmentFileName(segmentInfo.name, segmentSuffix, EXTENSION);
    try (IndexOutput output = directory.createOutput(fileName, context)) {
      CodecUtil.writeIndexHeader(output, Lucene60FieldInfosFormat.CODEC_NAME, Lucene60FieldInfosFormat.FORMAT_CURRENT, segmentInfo.getId(), segmentSuffix);
      output.writeVInt(infos.size());
      for (FieldInfo fi : infos) {
        fi.checkConsistency();

        output.writeString(fi.name);
        output.writeVInt(fi.number);
    
        byte bits = 0x0;
        if (fi.hasVectors()) bits |= STORE_TERMVECTOR;
        if (fi.omitsNorms()) bits |= OMIT_NORMS;
        if (fi.hasPayloads()) bits |= STORE_PAYLOADS;
        output.writeByte(bits);
        output.writeByte(indexOptionsByte(fi.getIndexOptions()));
        output.writeByte(docValuesByte(fi.getDocValuesType()));
        output.writeLong(fi.getDocValuesGen());
        output.writeMapOfStrings(fi.attributes());
        int pointDimensionCount = fi.getPointDimensionCount();
        output.writeVInt(pointDimensionCount);
        if (pointDimensionCount != 0) {
          output.writeVInt(fi.getPointNumBytes());
        }
      }
      CodecUtil.writeFooter(output);
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
Lucene60FieldInfosFormat的write函数会创建.fnm文件，并将Field域的相关信息写入该文件中。

合并索引文件
向上回到DocumentsWriterPerThread的flush中，接下来创建FlushedSegment，然后调用sealFlushedSegment合并索引文件夹中的文件，代码如下，

DocumentsWriter::doflush->DocumentsWriterPerThread::flush->sealFlushedSegment

  void sealFlushedSegment(FlushedSegment flushedSegment) throws IOException {

    SegmentCommitInfo newSegment = flushedSegment.segmentInfo;
    IndexWriter.setDiagnostics(newSegment.info, IndexWriter.SOURCE_FLUSH);
    IOContext context = new IOContext(new FlushInfo(newSegment.info.maxDoc(), newSegment.sizeInBytes()));
    
    boolean success = false;
    try {
    
      if (indexWriterConfig.getUseCompoundFile()) {
        Set<String> originalFiles = newSegment.info.files();
        indexWriter.createCompoundFile(infoStream, new TrackingDirectoryWrapper(directory), newSegment.info, context);
        filesToDelete.addAll(originalFiles);
        newSegment.info.setUseCompoundFile(true);
      }
    
      codec.segmentInfoFormat().write(directory, newSegment.info, context);
    
      if (flushedSegment.liveDocs != null) {
        final int delCount = flushedSegment.delCount;
        SegmentCommitInfo info = flushedSegment.segmentInfo;
        Codec codec = info.info.getCodec();
        codec.liveDocsFormat().writeLiveDocs(flushedSegment.liveDocs, directory, info, delCount, context);
        newSegment.setDelCount(delCount);
        newSegment.advanceDelGen();
      }
    
      success = true;
    } finally {
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
sealFlushedSegment函数中的indexWriter被定义为IndexWriter，其createCompoundFile函数用来合并索引文件夹中的文件，代码定义如下，
DocumentsWriter::doflush->DocumentsWriterPerThread::flush->sealFlushedSegment->IndexWriter::createCompoundFile

  final void createCompoundFile(InfoStream infoStream, TrackingDirectoryWrapper directory, final SegmentInfo info, IOContext context) throws IOException {

    boolean success = false;
    try {
      info.getCodec().compoundFormat().write(directory, info, context);
      success = true;
    } finally {
    
    }
    
    info.setFiles(new HashSet<>(directory.getCreatedFiles()));
  }
1
2
3
4
5
6
7
8
9
10
11
12
compoundFormat函数返回Lucene50CompoundFormat，其write函数定义如下，
DocumentsWriter::doflush->DocumentsWriterPerThread::flush->sealFlushedSegment->IndexWriter::createCompoundFile->Lucene50CompoundFormat::write

  public void write(Directory dir, SegmentInfo si, IOContext context) throws IOException {
    String dataFile = IndexFileNames.segmentFileName(si.name, "", DATA_EXTENSION);
    String entriesFile = IndexFileNames.segmentFileName(si.name, "", ENTRIES_EXTENSION);

    try (IndexOutput data =    dir.createOutput(dataFile, context);
         IndexOutput entries = dir.createOutput(entriesFile, context)) {
      CodecUtil.writeIndexHeader(data,    DATA_CODEC, VERSION_CURRENT, si.getId(), "");
      CodecUtil.writeIndexHeader(entries, ENTRY_CODEC, VERSION_CURRENT, si.getId(), "");
    
      entries.writeVInt(si.files().size());
      for (String file : si.files()) {
    
        long startOffset = data.getFilePointer();
        try (IndexInput in = dir.openInput(file, IOContext.READONCE)) {
          data.copyBytes(in, in.length());
        }
        long endOffset = data.getFilePointer();
    
        long length = endOffset - startOffset;
    
        entries.writeString(IndexFileNames.stripSegmentName(file));
        entries.writeLong(startOffset);
        entries.writeLong(length);
      }
    
      CodecUtil.writeFooter(data);
      CodecUtil.writeFooter(entries);
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
Lucene50CompoundFormat的write函数会创建.cfs、.cfe文件，以及对应的输出流，其中，.cfs保存各个文件的数据，.cfe保存各个文件的位置信息。

再回到DocumentsWriterPerThread的sealFlushedSegment函数中，接下来通过segmentInfoFormat返回一个Lucene50SegmentInfoFormat，其代码如下，
DocumentsWriter::doflush->DocumentsWriterPerThread::flush->sealFlushedSegment->Lucene50SegmentInfoFormat::write

  public void write(Directory dir, SegmentInfo si, IOContext ioContext) throws IOException {
    final String fileName = IndexFileNames.segmentFileName(si.name, "", Lucene50SegmentInfoFormat.SI_EXTENSION);

    try (IndexOutput output = dir.createOutput(fileName, ioContext)) {
      si.addFile(fileName);
      CodecUtil.writeIndexHeader(output, 
                                   Lucene50SegmentInfoFormat.CODEC_NAME, 
                                   Lucene50SegmentInfoFormat.VERSION_CURRENT,
                                   si.getId(),
                                   "");
      Version version = si.getVersion();
      output.writeInt(version.major);
      output.writeInt(version.minor);
      output.writeInt(version.bugfix);
      assert version.prerelease == 0;
      output.writeInt(si.maxDoc());
    
      output.writeByte((byte) (si.getUseCompoundFile() ? SegmentInfo.YES : SegmentInfo.NO));
      output.writeMapOfStrings(si.getDiagnostics());
      Set<String> files = si.files();
      for (String file : files) {
        if (!IndexFileNames.parseSegmentName(file).equals(si.name)) {
          throw new IllegalArgumentException();
        }
      }
      output.writeSetOfStrings(files);
      output.writeMapOfStrings(si.getAttributes());
      CodecUtil.writeFooter(output);
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
Lucene50SegmentInfoFormat的write函数会创建.si文件以及对应的输出流，然后向该文件写入相应的段信息。
sealFlushedSegment函数的最后会调用writeLiveDocs，创建.liv文件并写入相应的索引信息，代码如下，
DocumentsWriter::doflush->DocumentsWriterPerThread::flush->sealFlushedSegment->Lucene50LiveDocsFormat::writeLiveDocs

  public void writeLiveDocs(MutableBits bits, Directory dir, SegmentCommitInfo info, int newDelCount, IOContext context) throws IOException {
    long gen = info.getNextDelGen();
    String name = IndexFileNames.fileNameFromGeneration(info.info.name, EXTENSION, gen);
    FixedBitSet fbs = (FixedBitSet) bits;
    long data[] = fbs.getBits();
    try (IndexOutput output = dir.createOutput(name, context)) {
      CodecUtil.writeIndexHeader(output, CODEC_NAME, VERSION_CURRENT, info.info.getId(), Long.toString(gen, Character.MAX_RADIX));
      for (int i = 0; i < data.length; i++) {
        output.writeLong(data[i]);
      }
      CodecUtil.writeFooter(output);
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
回到DocumentsWriter的doflush函数，接下来就会根据flush函数的返回或者异常生成相应的事件，最后添加到事件队列中，例如DeleteNewFilesEvent、FlushFailedEvent、ForcedPurgeEvent、MergePendingEvent、ApplyDeletesEvent。这里就不往下再看了。
————————————————
版权声明：本文为CSDN博主「二侠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/conansonic/article/details/51966713

lucene源码分析—创建IndexReader
本章开始分析lucene的查询过程，下面先看一段lucene6版本下常用的查询代码，

        String indexPath;
        IndexReader reader = DirectoryReader.open(FSDirectory.open(Paths.get(indexPath)));
        IndexSearcher searcher = new IndexSearcher(reader);
        ScoreDoc[] hits = null;
        Query query = null;
        Analyzer analyzer = new SimpleAnalyzer();
        try {
            QueryParser qp = new QueryParser("body", analyzer);
            query = qp.parse(words);
        } catch (ParseException e) {
            return null;
        }
        if (searcher != null) {
            TopDocs results = searcher.search(query, 20);
            hits = results.scoreDocs;
            Document document = null;
            for (int i = 0; i < hits.length; i++) {
                document = searcher.doc(hits[i].doc);
            }
            reader.close();
        }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
indexPath表示索引文件夹的路径。FSDirectory的open函数前面几章已经分析过了，最后返回MMapDirectory、SimpleFSDirectory以及NIOFSDirectory其中之一，本章后面都假设为NIOFSDirectory。然后调用DirectoryReader的open函数创建一个IndexReader，如下所示，
DirectoryReader::open

  public static DirectoryReader open(final Directory directory) throws IOException {
    return StandardDirectoryReader.open(directory, null);
  }
  static DirectoryReader open(final Directory directory, final IndexCommit commit) throws IOException {
    return new SegmentInfos.FindSegmentsFile<DirectoryReader>(directory) {
        ...
    }.run(commit);
  }
1
2
3
4
5
6
7
8
DirectoryReader的open函数调用StandardDirectoryReader的open函数，进而调用FindSegmentsFile的run函数，最后其实返回一个StandardDirectoryReader。
DirectoryReader::open->FindSegmentsFile::run

    public T run() throws IOException {
      return run(null);
    }
    
    public T run(IndexCommit commit) throws IOException {
      if (commit != null) {
        ...
      }
    
      long lastGen = -1;
      long gen = -1;
      IOException exc = null;
    
      for (;;) {
        lastGen = gen;
        String files[] = directory.listAll();
        String files2[] = directory.listAll();
        Arrays.sort(files);
        Arrays.sort(files2);
        if (!Arrays.equals(files, files2)) {
          continue;
        }
        gen = getLastCommitGeneration(files);
    
        if (gen == -1) {
          throw new IndexNotFoundException();
        } else if (gen > lastGen) {
          String segmentFileName = IndexFileNames.fileNameFromGeneration(IndexFileNames.SEGMENTS, "", gen);
          try {
            T t = doBody(segmentFileName);
            return t;
          } catch (IOException err) {
    
          }
        } else {
          throw exc;
        }
      }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
假设索引文件夹下有文件segments_0，segments_1，segments.gen，上面这段代码中的getLastCommitGeneration返回1，即以”segments”开头的文件名里结尾最大的数字，fileNameFromGeneration返回segments_1。最重要的是doBody函数，用来将文件中的段以及域信息读入内存数据结构中，doBody在DirectoryReader的open中被重载，定义如下，
DirectoryReader::open->FindSegmentsFile::run->doBody

      protected DirectoryReader doBody(String segmentFileName) throws IOException {
        SegmentInfos sis = SegmentInfos.readCommit(directory, segmentFileName);
        final SegmentReader[] readers = new SegmentReader[sis.size()];
        boolean success = false;
        try {
          for (int i = sis.size()-1; i >= 0; i--) {
            readers[i] = new SegmentReader(sis.info(i), IOContext.READ);
          }
          DirectoryReader reader = new StandardDirectoryReader(directory, readers, null, sis, false, false);
          success = true;
    
          return reader;
        } finally {
    
        }
      }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
doBody首先通过SegmentInfos的readCommit函数读取段信息存入SegmentInfos，然后根据该段信息创建SegmentReader，SegmentReader的构造函数会读取每个段中的域信息并存储在SegmentReader的成员变量里。先来看SegmentInfos的readCommit函数，
DirectoryReader::open->FindSegmentsFile::run->doBody->SegmentInfos::readCommit

  public static final SegmentInfos readCommit(Directory directory, String segmentFileName) throws IOException {

    long generation = generationFromSegmentsFileName(segmentFileName);
    try (ChecksumIndexInput input = directory.openChecksumInput(segmentFileName, IOContext.READ)) {
      return readCommit(directory, input, generation);
    }
  }

  public ChecksumIndexInput openChecksumInput(String name, IOContext context) throws IOException {
    return new BufferedChecksumIndexInput(openInput(name, context));
  }

  public IndexInput openInput(String name, IOContext context) throws IOException {
    ensureOpen();
    ensureCanRead(name);
    Path path = getDirectory().resolve(name);
    FileChannel fc = FileChannel.open(path, StandardOpenOption.READ);
    return new NIOFSIndexInput("NIOFSIndexInput(path=\"" + path + "\")", fc, context);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
假设传入的段文件名为segments_1，上面代码中的generationFromSegmentsFileName函数返回1。readCommit函数首先通过openChecksumInput创建BufferedChecksumIndexInput，代表文件的输入流，其中的openInput函数用来创建NIOFSIndexInput，然后根据该输入流通过readCommit函数读取文件内容，
DirectoryReader::open->FindSegmentsFile::run->doBody->SegmentInfos::readCommit

  public static final SegmentInfos readCommit(Directory directory, ChecksumIndexInput input, long generation) throws IOException {

    int magic = input.readInt();
    if (magic != CodecUtil.CODEC_MAGIC) {
      throw new IndexFormatTooOldException();
    }
    int format = CodecUtil.checkHeaderNoMagic(input, "segments", VERSION_50, VERSION_CURRENT);
    byte id[] = new byte[StringHelper.ID_LENGTH];
    input.readBytes(id, 0, id.length);
    CodecUtil.checkIndexHeaderSuffix(input, Long.toString(generation, Character.MAX_RADIX));
    
    SegmentInfos infos = new SegmentInfos();
    infos.id = id;
    infos.generation = generation;
    infos.lastGeneration = generation;
    if (format >= VERSION_53) {
      infos.luceneVersion = Version.fromBits(input.readVInt(), input.readVInt(), input.readVInt());
    } else {
    
    }
    
    infos.version = input.readLong();
    infos.counter = input.readInt();
    int numSegments = input.readInt();
    
    if (format >= VERSION_53) {
      if (numSegments > 0) {
        infos.minSegmentLuceneVersion = Version.fromBits(input.readVInt(), input.readVInt(), input.readVInt());
      } else {
    
      }
    } else {
    
    }
    
    long totalDocs = 0;
    for (int seg = 0; seg < numSegments; seg++) {
      String segName = input.readString();
      final byte segmentID[];
      byte hasID = input.readByte();
      if (hasID == 1) {
        segmentID = new byte[StringHelper.ID_LENGTH];
        input.readBytes(segmentID, 0, segmentID.length);
      } else if (hasID == 0) {
    
      } else {
    
      }
      Codec codec = readCodec(input, format < VERSION_53);
      SegmentInfo info = codec.segmentInfoFormat().read(directory, segName, segmentID, IOContext.READ);
      info.setCodec(codec);
      totalDocs += info.maxDoc();
      long delGen = input.readLong();
      int delCount = input.readInt();
    
      long fieldInfosGen = input.readLong();
      long dvGen = input.readLong();
      SegmentCommitInfo siPerCommit = new SegmentCommitInfo(info, delCount, delGen, fieldInfosGen, dvGen);
      if (format >= VERSION_51) {
        siPerCommit.setFieldInfosFiles(input.readSetOfStrings());
      } else {
        siPerCommit.setFieldInfosFiles(Collections.unmodifiableSet(input.readStringSet()));
      }
      final Map<Integer,Set<String>> dvUpdateFiles;
      final int numDVFields = input.readInt();
      if (numDVFields == 0) {
        dvUpdateFiles = Collections.emptyMap();
      } else {
        Map<Integer,Set<String>> map = new HashMap<>(numDVFields);
        for (int i = 0; i < numDVFields; i++) {
          if (format >= VERSION_51) {
            map.put(input.readInt(), input.readSetOfStrings());
          } else {
            map.put(input.readInt(), Collections.unmodifiableSet(input.readStringSet()));
          }
        }
        dvUpdateFiles = Collections.unmodifiableMap(map);
      }
      siPerCommit.setDocValuesUpdatesFiles(dvUpdateFiles);
      infos.add(siPerCommit);
    
      Version segmentVersion = info.getVersion();
      if (format < VERSION_53) {
        if (infos.minSegmentLuceneVersion == null || segmentVersion.onOrAfter(infos.minSegmentLuceneVersion) == false) {
          infos.minSegmentLuceneVersion = segmentVersion;
        }
      }
    }
    
    if (format >= VERSION_51) {
      infos.userData = input.readMapOfStrings();
    } else {
      infos.userData = Collections.unmodifiableMap(input.readStringStringMap());
    }
    CodecUtil.checkFooter(input);
    return infos;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
readCommit函数较长，归纳起来，就是针对所有的段信息，读取并设置id、generation、lastGeneration、luceneVersion、version、counter、minSegmentLuceneVersion、userData等信息；
并且针对每个段，读取或设置段名、段ID、该段删除的文档数、删除文档的gen数字，域文件的gen数字，更新的文档的gen数字、该段域信息文件名、该段更新的文件名，最后将这些信息封装成SegmentInfos并返回。
其中，针对每个段，通过segmentInfoFormat函数获得Lucene50SegmentInfoFormat，调用其read函数读取各个信息封装成SegmentInfo，代码如下，
DirectoryReader::open->FindSegmentsFile::run->doBody->SegmentInfos::readCommit->Lucene50SegmentInfoFormat::read

  public SegmentInfo read(Directory dir, String segment, byte[] segmentID, IOContext context) throws IOException {
    final String fileName = IndexFileNames.segmentFileName(segment, "", Lucene50SegmentInfoFormat.SI_EXTENSION);
    try (ChecksumIndexInput input = dir.openChecksumInput(fileName, context)) {
      Throwable priorE = null;
      SegmentInfo si = null;
      try {
        int format = CodecUtil.checkIndexHeader(input, Lucene50SegmentInfoFormat.CODEC_NAME,
                                          Lucene50SegmentInfoFormat.VERSION_START,
                                          Lucene50SegmentInfoFormat.VERSION_CURRENT,
                                          segmentID, "");
        final Version version = Version.fromBits(input.readInt(), input.readInt(), input.readInt());

        final int docCount = input.readInt();
        final boolean isCompoundFile = input.readByte() == SegmentInfo.YES;
    
        final Map<String,String> diagnostics;
        final Set<String> files;
        final Map<String,String> attributes;
    
        if (format >= VERSION_SAFE_MAPS) {
          diagnostics = input.readMapOfStrings();
          files = input.readSetOfStrings();
          attributes = input.readMapOfStrings();
        } else {
          diagnostics = Collections.unmodifiableMap(input.readStringStringMap());
          files = Collections.unmodifiableSet(input.readStringSet());
          attributes = Collections.unmodifiableMap(input.readStringStringMap());
        }
    
        si = new SegmentInfo(dir, version, segment, docCount, isCompoundFile, null, diagnostics, segmentID, attributes);
        si.setFiles(files);
      } catch (Throwable exception) {
        priorE = exception;
      } finally {
        CodecUtil.checkFooter(input, priorE);
      }
      return si;
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
该read函数打开.si文件，并从中读取version、docCount、isCompoundFile、diagnostics、attributes、files信息，然后创建SegmentInfo封装这些信息并返回。

回到FindSegmentsFile的doBody函数中，从文件中所有的段信息通过readCommit函数封装成SegmentInfos，然后针对每个段，创建SegmentReader，在其构造函数中读取域信息。
DirectoryReader::open->FindSegmentsFile::run->doBody->SegmentReader::SegmentReader

  public SegmentReader(SegmentCommitInfo si, IOContext context) throws IOException {
    this.si = si;
    core = new SegmentCoreReaders(si.info.dir, si, context);
    segDocValues = new SegmentDocValues();

    boolean success = false;
    final Codec codec = si.info.getCodec();
    try {
      if (si.hasDeletions()) {
        liveDocs = codec.liveDocsFormat().readLiveDocs(directory(), si, IOContext.READONCE);
      } else {
        liveDocs = null;
      }
      numDocs = si.info.maxDoc() - si.getDelCount();
      fieldInfos = initFieldInfos();
      docValuesProducer = initDocValuesProducer();
    
      success = true;
    } finally {
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
si.info.dir就是索引文件所在的文件夹，先来看SegmentCoreReaders的构造函数，SegmentCoreReaders的构造函数中会读取域信息，
DirectoryReader::open->FindSegmentsFile::run->doBody->SegmentReader::SegmentReader->SegmentCoreReaders::SegmentCoreReaders

  SegmentCoreReaders(Directory dir, SegmentCommitInfo si, IOContext context) throws IOException {

    final Codec codec = si.info.getCodec();
    final Directory cfsDir;
    boolean success = false;
    
    try {
      if (si.info.getUseCompoundFile()) {
        cfsDir = cfsReader = codec.compoundFormat().getCompoundReader(dir, si.info, context);
      } else {
        cfsReader = null;
        cfsDir = dir;
      }
    
      coreFieldInfos = codec.fieldInfosFormat().read(cfsDir, si.info, "", context);
    
      final SegmentReadState segmentReadState = new SegmentReadState(cfsDir, si.info, coreFieldInfos, context);
      final PostingsFormat format = codec.postingsFormat();
      fields = format.fieldsProducer(segmentReadState);
    
      if (coreFieldInfos.hasNorms()) {
        normsProducer = codec.normsFormat().normsProducer(segmentReadState);
        assert normsProducer != null;
      } else {
        normsProducer = null;
      }
    
      fieldsReaderOrig = si.info.getCodec().storedFieldsFormat().fieldsReader(cfsDir, si.info, coreFieldInfos, context);
    
      if (coreFieldInfos.hasVectors()) {
        termVectorsReaderOrig = si.info.getCodec().termVectorsFormat().vectorsReader(cfsDir, si.info, coreFieldInfos, context);
      } else {
        termVectorsReaderOrig = null;
      }
    
      if (coreFieldInfos.hasPointValues()) {
        pointsReader = codec.pointsFormat().fieldsReader(segmentReadState);
      } else {
        pointsReader = null;
      }
      success = true;
    } finally {
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
getUseCompoundFile表示是否会封装成.cfs、.cfe文件，如果封装，就通过compoundFormat函数获得Lucene50CompoundFormat，然后调用其getCompoundReader函数，
DirectoryReader::open->FindSegmentsFile::run->doBody->SegmentReader::SegmentReader->SegmentCoreReaders::SegmentCoreReaders->Lucene50CompoundFormat::getCompoundReader

  public Directory getCompoundReader(Directory dir, SegmentInfo si, IOContext context) throws IOException {
    return new Lucene50CompoundReader(dir, si, context);
  }

  public Lucene50CompoundReader(Directory directory, SegmentInfo si, IOContext context) throws IOException {
    this.directory = directory;
    this.segmentName = si.name;
    String dataFileName = IndexFileNames.segmentFileName(segmentName, "", Lucene50CompoundFormat.DATA_EXTENSION);
    String entriesFileName = IndexFileNames.segmentFileName(segmentName, "", Lucene50CompoundFormat.ENTRIES_EXTENSION);
    this.entries = readEntries(si.getId(), directory, entriesFileName);
    boolean success = false;

    long expectedLength = CodecUtil.indexHeaderLength(Lucene50CompoundFormat.DATA_CODEC, "");
    for(Map.Entry<String,FileEntry> ent : entries.entrySet()) {
      expectedLength += ent.getValue().length;
    }
    expectedLength += CodecUtil.footerLength(); 
    
    handle = directory.openInput(dataFileName, context);
    try {
      CodecUtil.checkIndexHeader(handle, Lucene50CompoundFormat.DATA_CODEC, version, version, si.getId(), "");
      CodecUtil.retrieveChecksum(handle);
      success = true;
    } finally {
      if (!success) {
        IOUtils.closeWhileHandlingException(handle);
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
getCompoundReader用来创建Lucene50CompoundReader。Lucene50CompoundReader的构造函数打开.cfs以及.cfe文件，然后通过readEntries函数将其中包含的文件读取出来，存入entries中。

回到SegmentCoreReaders的构造函数。fieldInfosFormat返回Lucene60FieldInfosFormat，其read函数用来读取域信息，
DirectoryReader::open->FindSegmentsFile::run->doBody->SegmentReader::SegmentReader->SegmentCoreReaders::SegmentCoreReaders->Lucene60FieldInfosFormat::read

  public FieldInfos read(Directory directory, SegmentInfo segmentInfo, String segmentSuffix, IOContext context) throws IOException {
    final String fileName = IndexFileNames.segmentFileName(segmentInfo.name, segmentSuffix, EXTENSION);
    try (ChecksumIndexInput input = directory.openChecksumInput(fileName, context)) {
      Throwable priorE = null;
      FieldInfo infos[] = null;
      try {
        CodecUtil.checkIndexHeader(input,
                                   Lucene60FieldInfosFormat.CODEC_NAME, 
                                   Lucene60FieldInfosFormat.FORMAT_START, 
                                   Lucene60FieldInfosFormat.FORMAT_CURRENT,
                                   segmentInfo.getId(), segmentSuffix);

        final int size = input.readVInt();
        infos = new FieldInfo[size];
    
        Map<String,String> lastAttributes = Collections.emptyMap();
    
        for (int i = 0; i < size; i++) {
          String name = input.readString();
          final int fieldNumber = input.readVInt();
          byte bits = input.readByte();
          boolean storeTermVector = (bits & STORE_TERMVECTOR) != 0;
          boolean omitNorms = (bits & OMIT_NORMS) != 0;
          boolean storePayloads = (bits & STORE_PAYLOADS) != 0;
    
          final IndexOptions indexOptions = getIndexOptions(input, input.readByte());
          final DocValuesType docValuesType = getDocValuesType(input, input.readByte());
          final long dvGen = input.readLong();
          Map<String,String> attributes = input.readMapOfStrings();
    
          if (attributes.equals(lastAttributes)) {
            attributes = lastAttributes;
          }
          lastAttributes = attributes;
          int pointDimensionCount = input.readVInt();
          int pointNumBytes;
          if (pointDimensionCount != 0) {
            pointNumBytes = input.readVInt();
          } else {
            pointNumBytes = 0;
          }
    
          try {
            infos[i] = new FieldInfo(name, fieldNumber, storeTermVector, omitNorms, storePayloads, 
                                     indexOptions, docValuesType, dvGen, attributes,
                                     pointDimensionCount, pointNumBytes);
            infos[i].checkConsistency();
          } catch (IllegalStateException e) {
    
          }
        }
      } catch (Throwable exception) {
        priorE = exception;
      } finally {
        CodecUtil.checkFooter(input, priorE);
      }
      return new FieldInfos(infos);
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
该read函数打开.fnm文件，读取Field域的基本信息。然后遍历所有域，读取name域名、fieldNumber文档数量，storeTermVector是否存储词向量、omitNorms是否存储norm、storePayloads是否存储payload、indexOptions域存储方式、docValuesType文档内容类型、文档的gen、attributes、pointDimensionCount、pointNumBytes，最后封装成FieldInfo，再封装成FieldInfos。

回到SegmentCoreReaders构造函数。接下来的postingsFormat函数返回PerFieldPostingsFormat，其fieldsProducer函数最终设置fields为FieldsReader。
DirectoryReader::open->FindSegmentsFile::run->doBody->SegmentReader::SegmentReader->SegmentCoreReaders::SegmentCoreReaders->PerFieldPostingsFormat::fieldsProducer

  public final FieldsProducer fieldsProducer(SegmentReadState state)
      throws IOException {
    return new FieldsReader(state);
  }
1
2
3
4
normsFormat函数返回Lucene53NormsFormat，Lucene53NormsFormat的normsProducer函数返回Lucene53NormsProducer，赋值给normsProducer。

  public NormsProducer normsProducer(SegmentReadState state) throws IOException {
    return new Lucene53NormsProducer(state, DATA_CODEC, DATA_EXTENSION, METADATA_CODEC, METADATA_EXTENSION);
  }
1
2
3
再往下，依次分析，fieldsReaderOrig最终被赋值为CompressingStoredFieldsReader。termVectorsReaderOrig最终被赋值为CompressingTermVectorsReader。pointsReader最终被赋值为Lucene60PointsReader。

回到SegmentReader构造函数，现在已经读取了所有的段信息和域信息了，接下来如果段中有删除信息，就通过liveDocsFormat函数获得Lucene50LiveDocsFormat，并调用其readLiveDocs函数，
DirectoryReader::open->FindSegmentsFile::run->doBody->SegmentReader::SegmentReader->Lucene50LiveDocsFormat::readLiveDocs

  public Bits readLiveDocs(Directory dir, SegmentCommitInfo info, IOContext context) throws IOException {
    long gen = info.getDelGen();
    String name = IndexFileNames.fileNameFromGeneration(info.info.name, EXTENSION, gen);
    final int length = info.info.maxDoc();
    try (ChecksumIndexInput input = dir.openChecksumInput(name, context)) {
      Throwable priorE = null;
      try {
        CodecUtil.checkIndexHeader(input, CODEC_NAME, VERSION_START, VERSION_CURRENT, 
                                     info.info.getId(), Long.toString(gen, Character.MAX_RADIX));
        long data[] = new long[FixedBitSet.bits2words(length)];
        for (int i = 0; i < data.length; i++) {
          data[i] = input.readLong();
        }
        FixedBitSet fbs = new FixedBitSet(data, length);
        return fbs;
      } catch (Throwable exception) {
        priorE = exception;
      } finally {
        CodecUtil.checkFooter(input, priorE);
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
readLiveDocs函数打开.liv文件，创建输入流，然后读取并创建FixedBitSet用来标识哪些文件被删除。

回到SegmentReader构造函数。接下来的initFieldInfos函数将SegmentCoreReaders中的coreFieldInfos赋值给fieldInfos，如果段有更新，就重新读取一次。docValuesProducer函数最后会返回FieldsReader。

再回到FindSegmentsFile的doBody函数中，最后创建StandardDirectoryReader并返回。StandardDirectoryReader本身的构造函数较为简单，值得注意的是StandardDirectoryReader的父类CompositeReader的

回到实例中，接下来创建IndexSearcher以及QueryParser，这两个类的构造函数都没有关键内容，这里就不往下看了。
值得注意的是IndexSearcher的构造函数会调用StandardDirectoryReader的getContext函数，进而调用leaves函数，首先是getContext函数，定义在StandardDirectoryReader的父类CompositeReader中，
StandardDirectoryReader::getContext

  public final CompositeReaderContext getContext() {
    ensureOpen();
    if (readerContext == null) {
      readerContext = CompositeReaderContext.create(this);
    }
    return readerContext;
  }
1
2
3
4
5
6
7
ensureOpen用来确保IndexWriter未关闭，接下来通过create函数创建CompositeReaderContext，
CompositeReaderContext::create

    static CompositeReaderContext create(CompositeReader reader) {
      return new Builder(reader).build();
    }
    
    public CompositeReaderContext build() {
      return (CompositeReaderContext) build(null, reader, 0, 0);
    }
    
    private IndexReaderContext build(CompositeReaderContext parent, IndexReader reader, int ord, int docBase) {
      if (reader instanceof LeafReader) {
        final LeafReader ar = (LeafReader) reader;
        final LeafReaderContext atomic = new LeafReaderContext(parent, ar, ord, docBase, leaves.size(), leafDocBase);
        leaves.add(atomic);
        leafDocBase += reader.maxDoc();
        return atomic;
      } else {
        final CompositeReader cr = (CompositeReader) reader;
        final List<? extends IndexReader> sequentialSubReaders = cr.getSequentialSubReaders();
        final List<IndexReaderContext> children = Arrays.asList(new IndexReaderContext[sequentialSubReaders.size()]);
        final CompositeReaderContext newParent;
        if (parent == null) {
          newParent = new CompositeReaderContext(cr, children, leaves);
        } else {
          newParent = new CompositeReaderContext(parent, cr, ord, docBase, children);
        }
        int newDocBase = 0;
        for (int i = 0, c = sequentialSubReaders.size(); i < c; i++) {
          final IndexReader r = sequentialSubReaders.get(i);
          children.set(i, build(newParent, r, i, newDocBase));
          newDocBase += r.maxDoc();
        }
        assert newDocBase == cr.maxDoc();
        return newParent;
      }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
首先，getSequentialSubReaders函数返回的正是在FindSegmentsFile的doBody函数中为每个段创建的SegmentReader列表，接下来创建CompositeReaderContext，接下来为每个SegmentReader嵌套调用build函数并设置进children中，而SegmentReader继承自LeafReader，因此在嵌套调用的build函数中，会将每个SegmentReader封装为LeafReaderContext并设置进leaves列表中。

因此最后的leaves函数返回封装了SegmentReader的LeafReaderContext列表。

下一章开始分析QueryParser的parse函数。
————————————————
版权声明：本文为CSDN博主「二侠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/conansonic/article/details/52006663

lucene源码分析—QueryParser的parse函数
本章主要分析QueryParser类的parse函数，定义在其父类QueryParserBase中，
QueryParserBase::parse

  public Query parse(String query) throws ParseException {
    ReInit(new FastCharStream(new StringReader(query)));
    try {
      Query res = TopLevelQuery(field);
      return res!=null ? res : newBooleanQuery().build();
    } catch (ParseException | TokenMgrError tme) {

    } catch (BooleanQuery.TooManyClauses tmc) {
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
parse首先将需要搜索的字符串query封装成FastCharStream，FastCharStream实现了Java的CharStream接口，内部使用了一个缓存，并且可以方便读取并且改变读写指针。然后调用ReInit进行初始化，ReInit以及整个QueryParser都是由JavaCC根据org.apache.lucene.queryparse.classic.QueryParser.jj文件自动生成，设计到的JavaCC的知识可以从网上或者别的书上查找，本博文不会重点分析这块内容。
parse最重要的函数是TopLevelQuery，即返回顶层Query，TopLevelQuery会根据用来搜索的字符串query创建一个树形的Query结构，传入的参数field在QueryParserBase的构造函数中赋值，用来标识对哪个域进行搜索。
QueryParserBase::parse->QueryParser::TopLevelQuery

  final public Query TopLevelQuery(String field) throws ParseException {
    Query q;
    q = Query(field);
    jj_consume_token(0);
    {
      if (true) return q;
    }
    throw new Error();
  }
1
2
3
4
5
6
7
8
9
TopLevelQuery函数中最关键的是Query函数，由于QueryParser由JavaCC生成，这里只看QueryParser.jj文件。

QueryParser.jj::Query

Query Query(String field) :
{
  List<BooleanClause> clauses = new ArrayList<BooleanClause>();
  Query q, firstQuery=null;
  int conj, mods;
}
{
  mods=Modifiers() q=Clause(field)
  {
    addClause(clauses, CONJ_NONE, mods, q);
    if (mods == MOD_NONE)
        firstQuery=q;
  }
  (
    conj=Conjunction() mods=Modifiers() q=Clause(field)
    { addClause(clauses, conj, mods, q); }
  )*
    {
      if (clauses.size() == 1 && firstQuery != null)
        return firstQuery;
      else {
  return getBooleanQuery(clauses);
      }
    }
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
Modifiers返回搜索字符串中的”+”或”-“，Conjunction返回连接字符串。Query首先通过Clause函数返回一个子查询，然后调用addClause函数添加该子查询，

QueryParserBase::addClause

  protected void addClause(List<BooleanClause> clauses, int conj, int mods, Query q) {
    boolean required, prohibited;

    ...
    
    if (required && !prohibited)
      clauses.add(newBooleanClause(q, BooleanClause.Occur.MUST));
    else if (!required && !prohibited)
      clauses.add(newBooleanClause(q, BooleanClause.Occur.SHOULD));
    else if (!required && prohibited)
      clauses.add(newBooleanClause(q, BooleanClause.Occur.MUST_NOT));
    else
      throw new RuntimeException("Clause cannot be both required and prohibited");
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
addClause函数中省略的部分是根据参数连接符conj和mods计算required和prohibited的值，然后将Query封装成BooleanClause并添加到clauses列表中。

回到Query函数中，如果子查询clauses列表只有一个子查询，就直接返回，否则通过getBooleanQuery函数封装所有的子查询并最终返回一个BooleanClause。

下面来看Clause函数，即创建一个子查询，
QueryParser.jj::Clause

Query Clause(String field) : {
  Query q;
  Token fieldToken=null, boost=null;
}
{
  [
    LOOKAHEAD(2)
    (
    fieldToken=<TERM> <COLON> {field=discardEscapeChar(fieldToken.image);}
    | <STAR> <COLON> {field="*";}
    )
  ]

  (
   q=Term(field)
   | <LPAREN> q=Query(field) <RPAREN> (<CARAT> boost=<NUMBER>)?

  )
    {  return handleBoost(q, boost); }
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
LOOKAHEAD(2)表示要看两个符号，如果是Field，则要重新调整搜索的域。Clause函数最重要的是Term函数，该函数返回最终的Query，当然Clause函数也可以嵌套调用Query函数生成子查询。

QueryParser.jj::Term

Query Term(String field) : {
  Token term, boost=null, fuzzySlop=null, goop1, goop2;
  boolean prefix = false;
  boolean wildcard = false;
  boolean fuzzy = false;
  boolean regexp = false;
  boolean startInc=false;
  boolean endInc=false;
  Query q;
}
{
  (
     (
       term=<TERM>
       | term=<STAR> { wildcard=true; }
       | term=<PREFIXTERM> { prefix=true; }
       | term=<WILDTERM> { wildcard=true; }
       | term=<REGEXPTERM> { regexp=true; }
       | term=<NUMBER>
       | term=<BAREOPER> { term.image = term.image.substring(0,1); }
     )
     [ fuzzySlop=<FUZZY_SLOP> { fuzzy=true; } ]
     [ <CARAT> boost=<NUMBER> [ fuzzySlop=<FUZZY_SLOP> { fuzzy=true; } ] ]
     {
       q = handleBareTokenQuery(field, term, fuzzySlop, prefix, wildcard, fuzzy, regexp);
     }
     | ( ( <RANGEIN_START> {startInc=true;} | <RANGEEX_START> )
         ( goop1=<RANGE_GOOP>|goop1=<RANGE_QUOTED> )
         [ <RANGE_TO> ]
         ( goop2=<RANGE_GOOP>|goop2=<RANGE_QUOTED> )
         ( <RANGEIN_END> {endInc=true;} | <RANGEEX_END>))
       [ <CARAT> boost=<NUMBER> ]
        {
          boolean startOpen=false;
          boolean endOpen=false;
          if (goop1.kind == RANGE_QUOTED) {
            goop1.image = goop1.image.substring(1, goop1.image.length()-1);
          } else if ("*".equals(goop1.image)) {
            startOpen=true;
          }
          if (goop2.kind == RANGE_QUOTED) {
            goop2.image = goop2.image.substring(1, goop2.image.length()-1);
          } else if ("*".equals(goop2.image)) {
            endOpen=true;
          }
          q = getRangeQuery(field, startOpen ? null : discardEscapeChar(goop1.image), endOpen ? null : discardEscapeChar(goop2.image), startInc, endInc);
        }
     | term=<QUOTED>
       [ fuzzySlop=<FUZZY_SLOP> ]
       [ <CARAT> boost=<NUMBER> ]
       { q = handleQuotedTerm(field, term, fuzzySlop); }
  )
  { return handleBoost(q, boost); }
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
如果一个查询不包括引号(QUOTED)，边界符号(RANGE，例如小括号、中括号等)，大部分情况下最终会通过handleBareTokenQuery函数生成一个Term，代表一个词，然后被封装成一个子查询Clause，最后被封装成一个Query，Clause和Query互相嵌套，即一个Query里可以包含多个Clause，一个Clause里又可以从一个Query开始，最终的叶子节点就是Term对应的Query。

QueryParserBase::handleBareTokenQuery

  Query handleBareTokenQuery(String qfield, Token term, Token fuzzySlop, boolean prefix, boolean wildcard, boolean fuzzy, boolean regexp) throws ParseException {
    Query q;

    String termImage=discardEscapeChar(term.image);
    if (wildcard) {
      q = getWildcardQuery(qfield, term.image);
    } else if (prefix) {
      q = getPrefixQuery(qfield,
          discardEscapeChar(term.image.substring
              (0, term.image.length()-1)));
    } else if (regexp) {
      q = getRegexpQuery(qfield, term.image.substring(1, term.image.length()-1));
    } else if (fuzzy) {
      q = handleBareFuzzy(qfield, fuzzySlop, termImage);
    } else {
      q = getFieldQuery(qfield, termImage, false);
    }
    return q;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
举例来说，查询字符串AAA*代表prefix查询，此时参数prefix为真，A*A代表wildcard查询，此时参数wildcard为真，AA~代表fuzzy模糊查询，此时参数fuzzy为真。这里假设三个都不为真，就是一串平常的单词，最后会通过getFieldQuery生成一个Query，本文重点分析该函数。

QueryParserBase::handleBareTokenQuery->getFieldQuery

  protected Query getFieldQuery(String field, String queryText, boolean quoted) throws ParseException {
    return newFieldQuery(getAnalyzer(), field, queryText, quoted);
  }

  protected Query newFieldQuery(Analyzer analyzer, String field, String queryText, boolean quoted)  throws ParseException {
    BooleanClause.Occur occur = operator == Operator.AND ? BooleanClause.Occur.MUST : BooleanClause.Occur.SHOULD;
    return createFieldQuery(analyzer, occur, field, queryText, quoted || autoGeneratePhraseQueries, phraseSlop);
  }
1
2
3
4
5
6
7
8
getAnalyzer返回QueryParserBase的init函数中设置的分词器，这里为了方便分析，假设为SimpleAnalyzer。quoted以及autoGeneratePhraseQueries表示是否创建PhraseQuery，phraseSlop为位置因子，只有PhraseQuery用得到，这里不管它。下面来看createFieldQuery函数。
QueryParserBase::handleBareTokenQuery->getFieldQuery->newFieldQuery->QueryBuilder::createFieldQuery

  protected final Query createFieldQuery(Analyzer analyzer, BooleanClause.Occur operator, String field, String queryText, boolean quoted, int phraseSlop) {

    try (TokenStream source = analyzer.tokenStream(field, queryText);
         CachingTokenFilter stream = new CachingTokenFilter(source)) {
    
      TermToBytesRefAttribute termAtt = stream.getAttribute(TermToBytesRefAttribute.class);
      PositionIncrementAttribute posIncAtt = stream.addAttribute(PositionIncrementAttribute.class);
    
      int numTokens = 0;
      int positionCount = 0;
      boolean hasSynonyms = false;
    
      stream.reset();
      while (stream.incrementToken()) {
        numTokens++;
        int positionIncrement = posIncAtt.getPositionIncrement();
        if (positionIncrement != 0) {
          positionCount += positionIncrement;
        } else {
          hasSynonyms = true;
        }
      }
    
      if (numTokens == 0) {
        return null;
      } else if (numTokens == 1) {
        return analyzeTerm(field, stream);
      } else if (quoted && positionCount > 1) {
        ...
      } else {
        if (positionCount == 1) {
          return analyzeBoolean(field, stream);
        } else {
          return analyzeMultiBoolean(field, stream, operator);
        }
      }
    } catch (IOException e) {
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
关于分词器的tokenStream以及incrementToken函数在《lucene源码分析—4》中分析过了。直接看最后的结果，假设numTokens==1，则分词器的输出结果只有一个词，则使用analyzeTerm创建最终的Query；
假设positionCount == 1，则表示结果中多个词出现在同一个位置，此时使用analyzeBoolean创建Query；剩下情况表示有多个词，至少两个词出现在不同位置，使用analyzeMultiBoolean创建Query。本文只分析analyzeMultiBoolean函数，
QueryParserBase::handleBareTokenQuery->getFieldQuery->newFieldQuery->QueryBuilder::createFieldQuery->analyzeMultiBoolean

  private Query analyzeMultiBoolean(String field, TokenStream stream, BooleanClause.Occur operator) throws IOException {
    BooleanQuery.Builder q = newBooleanQuery();
    List<Term> currentQuery = new ArrayList<>();

    TermToBytesRefAttribute termAtt = stream.getAttribute(TermToBytesRefAttribute.class);
    PositionIncrementAttribute posIncrAtt = stream.getAttribute(PositionIncrementAttribute.class);
    
    stream.reset();
    while (stream.incrementToken()) {
      if (posIncrAtt.getPositionIncrement() != 0) {
        add(q, currentQuery, operator);
        currentQuery.clear();
      }
      currentQuery.add(new Term(field, termAtt.getBytesRef()));
    }
    add(q, currentQuery, operator);
    
    return q.build();
  }

  private void add(BooleanQuery.Builder q, List<Term> current, BooleanClause.Occur operator) {
    if (current.isEmpty()) {
      return;
    }
    if (current.size() == 1) {
      q.add(newTermQuery(current.get(0)), operator);
    } else {
      q.add(newSynonymQuery(current.toArray(new Term[current.size()])), operator);
    }
  }

  public Builder add(Query query, Occur occur) {
    clauses.add(new BooleanClause(query, occur));
    return this;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
分词器的输出结果保存在TermToBytesRefAttribute中，analyzeMultiBoolean函数将同一个起始位置不同的Term添加到列表currentQuery中，如果同一个位置只有一个Term，则将其封装成TermQuery，如果有多个Term，就封装成SynonymQuery，TermQuery和SynonymQuery最后被封装成BooleanClause，添加到BooleanQuery.Builder中的一个BooleanClause列表中。最后通过BooleanQuery.Builder的build函数根据内置的BooleanClause列表创建一个最终的BooleanClause。
————————————————
版权声明：本文为CSDN博主「二侠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/conansonic/article/details/52021137

lucene源码分析—查询过程
本章开始介绍lucene的查询过程，即IndexSearcher的search函数，

IndexSearcher::search

  public TopDocs search(Query query, int n)
    throws IOException {
    return searchAfter(null, query, n);
  }
  public TopDocs searchAfter(ScoreDoc after, Query query, int numHits) throws IOException {
    final int limit = Math.max(1, reader.maxDoc());
    numHits = Math.min(numHits, limit);
    final int cappedNumHits = Math.min(numHits, limit);

    final CollectorManager<TopScoreDocCollector, TopDocs> manager = new CollectorManager<TopScoreDocCollector, TopDocs>() {
    
      @Override
      public TopScoreDocCollector newCollector() throws IOException {
        ...
      }
    
      @Override
      public TopDocs reduce(Collection<TopScoreDocCollector> collectors) throws IOException {
        ...
      }
    
    };
    
    return search(query, manager);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
传入的参数query封装了查询语句，n代表取前n个结果。searchAfter函数前面的计算保证最后的文档数量n不会超过所有文档的数量。接下来创建CollectorManager，并调用重载的search继续执行。

IndexSearch::search->searchAfter->search

  public <C extends Collector, T> T search(Query query, CollectorManager<C, T> collectorManager) throws IOException {
    if (executor == null) {
      final C collector = collectorManager.newCollector();
      search(query, collector);
      return collectorManager.reduce(Collections.singletonList(collector));
    } else {

      ...
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
假设查询过程为单线程，此时executor为空。首先通过CollectorManager的newCollector创建TopScoreDocCollector，每个TopScoreDocCollector封装了最后的查询结果，如果是多线程查询，则最后要对多个TopScoreDocCollector进行合并。

IndexSearch::search->searchAfter->search->CollectorManager::newCollector

   public TopScoreDocCollector newCollector() throws IOException {
     return TopScoreDocCollector.create(cappedNumHits, after);
   }

   public static TopScoreDocCollector create(int numHits, ScoreDoc after) {
    if (after == null) {
      return new SimpleTopScoreDocCollector(numHits);
    } else {
      return new PagingTopScoreDocCollector(numHits, after);
    }
  }
1
2
3
4
5
6
7
8
9
10
11
参数after用来实现类似分页的效果，这里假设为null。newCollector函数最终返回SimpleTopScoreDocCollector。创建完TopScoreDocCollector后，接下来调用重载的search函数继续执行。

IndexSearch::search->searchAfter->search->search

  public void search(Query query, Collector results)
    throws IOException {
    search(leafContexts, createNormalizedWeight(query, results.needsScores()), results);
  }
1
2
3
4
leafContexts是CompositeReaderContext中的leaves成员变量，是一个LeafReaderContext列表，每个LeafReaderContext封装了每个段的SegmentReader，SegmentReader可以读取每个段的所有信息和数据。接下来通过createNormalizedWeight函数进行查询匹配，并计算一些基本的权重用来给后面的打分过程使用。

  public Weight createNormalizedWeight(Query query, boolean needsScores) throws IOException {
    query = rewrite(query);
    Weight weight = createWeight(query, needsScores);
    float v = weight.getValueForNormalization();
    float norm = getSimilarity(needsScores).queryNorm(v);
    if (Float.isInfinite(norm) || Float.isNaN(norm)) {
      norm = 1.0f;
    }
    weight.normalize(norm, 1.0f);
    return weight;
  }
1
2
3
4
5
6
7
8
9
10
11
首先通过rewrite函数对Query进行重写，例如删除一些不必要的项，将非原子查询转化为原子查询。

rewrite
IndexSearch::search->searchAfter->search->search->createNormalizedWeight->rewrite

  public Query rewrite(Query original) throws IOException {
    Query query = original;
    for (Query rewrittenQuery = query.rewrite(reader); rewrittenQuery != query;
         rewrittenQuery = query.rewrite(reader)) {
      query = rewrittenQuery;
    }
    return query;
  }
1
2
3
4
5
6
7
8
这里循环调用每个Query的rewrite函数进行重写，之所以循环是因为可能一次重写改变Query结构后又产生了可以被重写的部分，下面假设这里的query为BooleanQuery，BooleanQuery并不包含真正的查询语句，而是包含多个子查询，每个子查询可以是TermQuery这样不可再分的Query，也可以是另一个BooleanQuery。
由于BooleanQuery的rewrite函数较长，下面分段来看。
IndexSearch::search->searchAfter->search->search->createNormalizedWeight->BooleanQuery::rewrite
第一部分

  public Query rewrite(IndexReader reader) throws IOException {

    if (clauses.size() == 1) {
      BooleanClause c = clauses.get(0);
      Query query = c.getQuery();
      if (minimumNumberShouldMatch == 1 && c.getOccur() == Occur.SHOULD) {
        return query;
      } else if (minimumNumberShouldMatch == 0) {
        switch (c.getOccur()) {
          case SHOULD:
          case MUST:
            return query;
          case FILTER:
            return new BoostQuery(new ConstantScoreQuery(query), 0);
          case MUST_NOT:
            return new MatchNoDocsQuery();
          default:
            throw new AssertionError();
        }
      }
    }
    
    ...

  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
如果BooleanQuery中只有一个子查询，则没必要对其封装，直接取出该子查询中的Query即可。
minimumNumberShouldMatch成员变量表示至少需要匹配多少项，如果唯一的子查询条件为SHOULD，并且匹配1项就行了，则直接返回对应的Query，如果条件为MUST或者SHOULD，也是直接返回子查询中的Query，如果条件为FILTER，则直接通过BoostQuery封装并返回，如果条件为MUST_NOT，则说明唯一的查询不需要查询任何文档，直接创建MatchNODocsQuery即可。

IndexSearch::search->searchAfter->search->search->createNormalizedWeight->BooleanQuery::rewrite
第二部分

  public Query rewrite(IndexReader reader) throws IOException {

    ...
    
    {
      BooleanQuery.Builder builder = new BooleanQuery.Builder();
      builder.setDisableCoord(isCoordDisabled());
      builder.setMinimumNumberShouldMatch(getMinimumNumberShouldMatch());
      boolean actuallyRewritten = false;
      for (BooleanClause clause : this) {
        Query query = clause.getQuery();
        Query rewritten = query.rewrite(reader);
        if (rewritten != query) {
          actuallyRewritten = true;
        }
        builder.add(rewritten, clause.getOccur());
      }
      if (actuallyRewritten) {
        return builder.build();
      }
    }
    
    ...

  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
这部分rewrite函数遍历BooleanQuery下的所有的子查询列表，嵌套调用rewrite函数，如果某次rewrite函数返回的Query和原来的Query不一样，则说明某个子查询被重写了，此时通过BooleanQuery.Builder的build函数重新生成BooleanQuery。

IndexSearch::search->searchAfter->search->search->createNormalizedWeight->BooleanQuery::rewrite
第三部分

  public Query rewrite(IndexReader reader) throws IOException {

    ...
    
    {
      int clauseCount = 0;
      for (Collection<Query> queries : clauseSets.values()) {
        clauseCount += queries.size();
      }
      if (clauseCount != clauses.size()) {
        BooleanQuery.Builder rewritten = new BooleanQuery.Builder();
        rewritten.setDisableCoord(disableCoord);
        rewritten.setMinimumNumberShouldMatch(minimumNumberShouldMatch);
        for (Map.Entry<Occur, Collection<Query>> entry : clauseSets.entrySet()) {
          final Occur occur = entry.getKey();
          for (Query query : entry.getValue()) {
            rewritten.add(query, occur);
          }
        }
        return rewritten.build();
      }
    }
    
    ...

  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
clauseSets中保存了MUST_NOT和FILTER对应的子查询Clause，并使用HashSet进行存储。利用HashSet的结构可以去除重复的条件为MUST_NOT和FILTER的子查询。

IndexSearch::search->searchAfter->search->search->createNormalizedWeight->BooleanQuery::rewrite
第四部分

  public Query rewrite(IndexReader reader) throws IOException {

    ...
    
    if (clauseSets.get(Occur.MUST).size() > 0 && clauseSets.get(Occur.FILTER).size() > 0) {
      final Set<Query> filters = new HashSet<Query>(clauseSets.get(Occur.FILTER));
      boolean modified = filters.remove(new MatchAllDocsQuery());
      modified |= filters.removeAll(clauseSets.get(Occur.MUST));
      if (modified) {
        BooleanQuery.Builder builder = new BooleanQuery.Builder();
        builder.setDisableCoord(isCoordDisabled());
        builder.setMinimumNumberShouldMatch(getMinimumNumberShouldMatch());
        for (BooleanClause clause : clauses) {
          if (clause.getOccur() != Occur.FILTER) {
            builder.add(clause);
          }
        }
        for (Query filter : filters) {
          builder.add(filter, Occur.FILTER);
        }
        return builder.build();
      }
    }
    
    ...

  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
删除条件为FILTER又同时为MUST的子查询，同时删除查询所有文档的子查询（因为此时子查询的数量肯定大于1），查询所有文档的结果集里包含了任何其他查询的结果集。

IndexSearch::search->searchAfter->search->search->createNormalizedWeight->BooleanQuery::rewrite
第五部分

    {
      final Collection<Query> musts = clauseSets.get(Occur.MUST);
      final Collection<Query> filters = clauseSets.get(Occur.FILTER);
      if (musts.size() == 1
          && filters.size() > 0) {
        Query must = musts.iterator().next();
        float boost = 1f;
        if (must instanceof BoostQuery) {
          BoostQuery boostQuery = (BoostQuery) must;
          must = boostQuery.getQuery();
          boost = boostQuery.getBoost();
        }
        if (must.getClass() == MatchAllDocsQuery.class) {
          BooleanQuery.Builder builder = new BooleanQuery.Builder();
          for (BooleanClause clause : clauses) {
            switch (clause.getOccur()) {
              case FILTER:
              case MUST_NOT:
                builder.add(clause);
                break;
              default:
                break;
            }
          }
          Query rewritten = builder.build();
          rewritten = new ConstantScoreQuery(rewritten);
    
          builder = new BooleanQuery.Builder()
            .setDisableCoord(isCoordDisabled())
            .setMinimumNumberShouldMatch(getMinimumNumberShouldMatch())
            .add(rewritten, Occur.MUST);
          for (Query query : clauseSets.get(Occur.SHOULD)) {
            builder.add(query, Occur.SHOULD);
          }
          rewritten = builder.build();
          return rewritten;
        }
      }
    }
    
    return super.rewrite(reader);
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
如果某个MatchAllDocsQuery是唯一的类型为MUST的Query，则对其进行重写。最后如果没有重写，就调用父类Query的rewrite直接返回其自身。

看完了BooleanQuery的rewrite函数，下面简单介绍一下其他类型Query的rewrite函数。
TermQuery的rewrite函数，直接返回自身。SynonymQuery的rewrite函数检测是否只包含一个Query，如果只有一个Query，则将其转化为TermQuery。WildcardQuery、PrefixQuery、RegexpQuery以及FuzzyQuery都继承自MultiTermQuery。WildcardQuery的rewrite函数返回一个封装了原来Query的MultiTermQueryConstantScoreWrapper。PrefixQuery的rewrite函数返回一个MultiTermQueryConstantScoreWrapper。RegexpQuery类似PrefixQuery。FuzzyQuery最后根据情况返回一个BlendedTermQuery。

回到createNormalizedWeight函数中，重写完Query之后，接下来通过createWeight函数进行匹配并计算权重。

createWeight
IndexSearch::search->searchAfter->search->search->createNormalizedWeight->createWeight

  public Weight createWeight(Query query, boolean needsScores) throws IOException {
    final QueryCache queryCache = this.queryCache;
    Weight weight = query.createWeight(this, needsScores);
    if (needsScores == false && queryCache != null) {
      weight = queryCache.doCache(weight, queryCachingPolicy);
    }
    return weight;
  }
1
2
3
4
5
6
7
8
IndexSearch中的成员变量queryCache被初始化为LRUQueryCache。createWeight函数会调用各个Query中的createWeight函数，假设为BooleanQuery。

IndexSearch::search->searchAfter->search->search->createNormalizedWeight->createWeight->BooleanQuery::createWeight

  public Weight createWeight(IndexSearcher searcher, boolean needsScores) throws IOException {
    BooleanQuery query = this;
    if (needsScores == false) {
      query = rewriteNoScoring();
    }
    return new BooleanWeight(query, searcher, needsScores, disableCoord);
  }
1
2
3
4
5
6
7
needsScores在SimpleTopScoreDocCollector中默认返回true。createWeight函数创建BooleanWeight并返回。
IndexSearch::search->searchAfter->search->search->createNormalizedWeight->createWeight->BooleanWeight::createWeight->BooleanWeight::BooleanWeight

  BooleanWeight(BooleanQuery query, IndexSearcher searcher, boolean needsScores, boolean disableCoord) throws IOException {
    super(query);
    this.query = query;
    this.needsScores = needsScores;
    this.similarity = searcher.getSimilarity(needsScores);
    weights = new ArrayList<>();
    int i = 0;
    int maxCoord = 0;
    for (BooleanClause c : query) {
      Weight w = searcher.createWeight(c.getQuery(), needsScores && c.isScoring());
      weights.add(w);
      if (c.isScoring()) {
        maxCoord++;
      }
      i += 1;
    }
    this.maxCoord = maxCoord;

    coords = new float[maxCoord+1];
    Arrays.fill(coords, 1F);
    coords[0] = 0f;
    if (maxCoord > 0 && needsScores && disableCoord == false) {
      boolean seenActualCoord = false;
      for (i = 1; i < coords.length; i++) {
        coords[i] = coord(i, maxCoord);
        seenActualCoord |= (coords[i] != 1F);
      }
      this.disableCoord = seenActualCoord == false;
    } else {
      this.disableCoord = true;
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
getSimilarity函数默认返回IndexSearcher中的BM25Similarity。BooleanWeight函数嵌套调用createWeight获取子查询的Weight，假设子查询为TermQuery，后面来看TermQuery的createWeight函数。maxCoord用来表示有多少个子查询，最后面的coords数组能够影响检索文档的得分，计算公式为coord(q,d) = q/d。

IndexSearch::search->searchAfter->search->search->createNormalizedWeight->createWeight->TermQuery::createWeight

  public Weight createWeight(IndexSearcher searcher, boolean needsScores) throws IOException {
    final IndexReaderContext context = searcher.getTopReaderContext();
    final TermContext termState;
    if (perReaderTermState == null
        || perReaderTermState.topReaderContext != context) {
      termState = TermContext.build(context, term);
    } else {
      termState = this.perReaderTermState;
    }

    return new TermWeight(searcher, needsScores, termState);
  }
1
2
3
4
5
6
7
8
9
10
11
12
getTopReaderContext返回CompositeReaderContext，封装了SegmentReader。
perReaderTermState默认为null，因此接下来通过TermContext的build函数进行匹配并获取对应的Term在索引表中的相应信息，最后根据得到的信息TermContext创建TermWeight并返回。

IndexSearch::search->searchAfter->search->search->createNormalizedWeight->createWeight->TermQuery::createWeight->TermContext::build

  public static TermContext build(IndexReaderContext context, Term term)
      throws IOException {
    final String field = term.field();
    final BytesRef bytes = term.bytes();
    final TermContext perReaderTermState = new TermContext(context);
    for (final LeafReaderContext ctx : context.leaves()) {
      final Terms terms = ctx.reader().terms(field);
      if (terms != null) {
        final TermsEnum termsEnum = terms.iterator();
        if (termsEnum.seekExact(bytes)) { 
          final TermState termState = termsEnum.termState();
          perReaderTermState.register(termState, ctx.ord, termsEnum.docFreq(), termsEnum.totalTermFreq());
        }
      }
    }
    return perReaderTermState;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
Term的bytes函数返回查询的字节，默认的是UTF-8编码。LeafReaderContext的reader函数返回SegmentReader，对应的terms函数返回FieldReader用来读取文件中的信息。

IndexSearcher::search->searchAfter->search->search->createNormalizedWeight->createWeight->TermQuery::createWeight->TermContext::build->SegmentReader::terms

  public final Terms terms(String field) throws IOException {
    return fields().terms(field);
  }

  public final Fields fields() {
    return getPostingsReader();
  }

  public FieldsProducer getPostingsReader() {
    ensureOpen();
    return core.fields;
  }

  public Terms terms(String field) throws IOException {
    FieldsProducer fieldsProducer = fields.get(field);
    return fieldsProducer == null ? null : fieldsProducer.terms(field);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
core在SegmentReader构造函数中创建为SegmentCoreReaders，对应fields为PerFieldPostingsFormat。fields.get最终返回BlockTreeTermsReader，在创建索引时设置的。
BlockTreeTermsReader的terms最终返回对应域的FieldReader。

回到TermContext的build函数中，接下来的iterator函数返回SegmentTermsEnum，然后通过seekExact函数查询匹配，如果匹配，通过SegmentTermsEnum的termState函数返回一个IntBlockTermState，里面封装该Term的各个信息，seekExact函数在下一章分析。build函数最后通过TermContext的register函数保存计算获得的IntBlockTermState。

IndexSearch::search->searchAfter->search->search->createNormalizedWeight->createWeight->TermQuery::createWeight->TermContext::build->register

  public void register(TermState state, final int ord, final int docFreq, final long totalTermFreq) {
    register(state, ord);
    accumulateStatistics(docFreq, totalTermFreq);
  }

  public void register(TermState state, final int ord) {
    states[ord] = state;
  }

  public void accumulateStatistics(final int docFreq, final long totalTermFreq) {
    this.docFreq += docFreq;
    if (this.totalTermFreq >= 0 && totalTermFreq >= 0)
      this.totalTermFreq += totalTermFreq;
    else
      this.totalTermFreq = -1;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
传入的参数ord用于标识一个唯一的IndexReaderContext，即一个段。register函数将TermState，其实是IntBlockTermState存储进数组states中，然后通过accumulateStatistics更新统计信息。
回到TermQuery的createWeight函数中，最后创建一个TermWeight并返回。

IndexSearch::search->searchAfter->search->search->createNormalizedWeight->createWeight->TermQuery::createWeight->TermWeight::TermWeight

    public TermWeight(IndexSearcher searcher, boolean needsScores, TermContext termStates)
        throws IOException {
      super(TermQuery.this);
      this.needsScores = needsScores;
      this.termStates = termStates;
      this.similarity = searcher.getSimilarity(needsScores);
    
      final CollectionStatistics collectionStats;
      final TermStatistics termStats;
      if (needsScores) {
        collectionStats = searcher.collectionStatistics(term.field());
        termStats = searcher.termStatistics(term, termStates);
      } else {
        ...
      }
    
      this.stats = similarity.computeWeight(collectionStats, termStats);
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
整体上看，collectionStatistics函数用来统计某个域中的信息，termStatistics函数用来统计某个词的信息。
最后通过这两个信息调用computeWeight函数计算权重。下面分别来看。

IndexSearcher::search->searchAfter->search->search->createNormalizedWeight->createWeight->TermQuery::createWeight->TermWeight::TermWeight->IndexSearcher::collectionStatistics

  public CollectionStatistics collectionStatistics(String field) throws IOException {
    final int docCount;
    final long sumTotalTermFreq;
    final long sumDocFreq;

    Terms terms = MultiFields.getTerms(reader, field);
    if (terms == null) {
      docCount = 0;
      sumTotalTermFreq = 0;
      sumDocFreq = 0;
    } else {
      docCount = terms.getDocCount();
      sumTotalTermFreq = terms.getSumTotalTermFreq();
      sumDocFreq = terms.getSumDocFreq();
    }
    return new CollectionStatistics(field, reader.maxDoc(), docCount, sumTotalTermFreq, sumDocFreq);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
getTerms函数和前面的分析类似，最后返回一个FieldReader，然后获取docCount文档数、sumTotalTermFreq所有termFreq（每篇文档有多少个Term）的总和、sumDocFreq所有docFreq（多少篇文档包含Term）的总和，最后创建CollectionStatistics封装这些信息并返回。

IndexSearcher::search->searchAfter->search->search->createNormalizedWeight->createWeight->TermQuery::createWeight->TermWeight::TermWeight->IndexSearcher::termStatistics

  public TermStatistics termStatistics(Term term, TermContext context) throws IOException {
    return new TermStatistics(term.bytes(), context.docFreq(), context.totalTermFreq());
  }
1
2
3
docFreq返回有多少篇文档包含该词，totalTermFreq返回文档中包含多少个该词，最后创建一个TermStatistics并返回，构造函数简单。

回到TermWeight的构造函数中，similarity默认为BM25Similarity，computeWeight函数如下。

IndexSearcher::search->searchAfter->search->search->createNormalizedWeight->createWeight->TermQuery::createWeight->TermWeight::TermWeight->BM25Similarity::computeWeight

  public final SimWeight computeWeight(CollectionStatistics collectionStats, TermStatistics... termStats) {
    Explanation idf = termStats.length == 1 ? idfExplain(collectionStats, termStats[0]) : idfExplain(collectionStats, termStats);

    float avgdl = avgFieldLength(collectionStats);
    
    float cache[] = new float[256];
    for (int i = 0; i < cache.length; i++) {
      cache[i] = k1 * ((1 - b) + b * decodeNormValue((byte)i) / avgdl);
    }
    return new BM25Stats(collectionStats.field(), idf, avgdl, cache);
  }
1
2
3
4
5
6
7
8
9
10
11
idfExplain函数用来计算idf，即反转文档频率。
IndexSearcher::search->searchAfter->search->search->createNormalizedWeight->createWeight->TermQuery::createWeight->TermWeight::TermWeight->BM25Similarity::computeWeight->idfExplain

  public Explanation idfExplain(CollectionStatistics collectionStats, TermStatistics termStats) {
    final long df = termStats.docFreq();
    final long docCount = collectionStats.docCount() == -1 ? collectionStats.maxDoc() : collectionStats.docCount();
    final float idf = idf(df, docCount);
    return Explanation.match(idf, "idf(docFreq=" + df + ", docCount=" + docCount + ")");
  }
1
2
3
4
5
6
df为多少篇文档包含该词，docCount为文档总数，idf函数的计算公式如下，
1 + log(numDocs/(docFreq+1))，含义是如果文档中出现Term的频率越高显得文档越不重要。
回到computeWeight中，avgFieldLength函数用来计算每篇文档包含词的平均数。

IndexSearcher::search->searchAfter->search->search->createNormalizedWeight->createWeight->TermQuery::createWeight->TermWeight::TermWeight->BM25Similarity::computeWeight->avgFieldLength

  protected float avgFieldLength(CollectionStatistics collectionStats) {
    final long sumTotalTermFreq = collectionStats.sumTotalTermFreq();
    if (sumTotalTermFreq <= 0) {
      return 1f;
    } else {
      final long docCount = collectionStats.docCount() == -1 ? collectionStats.maxDoc() : collectionStats.docCount();
      return (float) (sumTotalTermFreq / (double) docCount);
    }
  }
1
2
3
4
5
6
7
8
9
avgFieldLength函数将词频总数除以文档数，得到每篇文档的平均词数。回到computeWeight中，接下来计算BM25的相关系数，BM25是lucene进行排序的算法，最后创建BM25Stats并返回。

回到createNormalizedWeight中，接下来通过getValueForNormalization函数计算权重。

IndexSearch::search->searchAfter->search->search->createNormalizedWeight->BooleanWeight::getValueForNormalization

  public float getValueForNormalization() throws IOException {
    float sum = 0.0f;
    int i = 0;
    for (BooleanClause clause : query) {
      float s = weights.get(i).getValueForNormalization();
      if (clause.isScoring()) {
        sum += s;
      }
      i += 1;
    }

    return sum ;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
BooleanWeight的getValueForNormalization函数用来累积子查询中getValueForNormalization函数返回的值。假设子查询为TermQuery，对应的Weight为TermWeight，其getValueForNormalization函数如下。

IndexSearch::search->searchAfter->search->search->createNormalizedWeight->BooleanWeight::getValueForNormalization->TermWeight::getValueForNormalization

    public float getValueForNormalization() {
      return stats.getValueForNormalization();
    }
    
    public float getValueForNormalization() {
      return weight * weight;
    }
    
    public void normalize(float queryNorm, float boost) {
      this.boost = boost;
      this.weight = idf.getValue() * boost;
    } 
1
2
3
4
5
6
7
8
9
10
11
12
stats就是是BM25Stats，其getValueForNormalization函数最终返回idf值乘以boost后的平方。

回到createNormalizedWeight中，queryNorm函数直接返回1，normalize函数根据norm重新计算权重。首先看BooleanWeight的normalize函数，

IndexSearch::search->searchAfter->search->search->createNormalizedWeight->BooleanWeight::normalize

  public void normalize(float norm, float boost) {
    for (Weight w : weights) {
      w.normalize(norm, boost);
    }
  }
1
2
3
4
5
假设子查询对应的Weight为TermWeight。

IndexSearch::search->searchAfter->search->search->createNormalizedWeight->TermWeight::normalize

    public void normalize(float queryNorm, float boost) {
      stats.normalize(queryNorm, boost);
    }
    
    public void normalize(float queryNorm, float boost) {
      this.boost = boost;
      this.weight = idf.getValue() * boost;
    }
1
2
3
4
5
6
7
8
回到IndexSearcher的search函数中，createNormalizedWeight返回Weight后，继续调用重载的search函数，定义如下，

IndexSearch::search->searchAfter->search->search->search

  protected void search(List<LeafReaderContext> leaves, Weight weight, Collector collector)
      throws IOException {

    for (LeafReaderContext ctx : leaves) {
      final LeafCollector leafCollector;
      try {
        leafCollector = collector.getLeafCollector(ctx);
      } catch (CollectionTerminatedException e) {
    
      }
      BulkScorer scorer = weight.bulkScorer(ctx);
      if (scorer != null) {
        try {
          scorer.score(leafCollector, ctx.reader().getLiveDocs());
        } catch (CollectionTerminatedException e) {
    
        }
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
根据《lucence源码分析—6》leaves是封装了SegmentReader的LeafReaderContext列表，collector是SimpleTopScoreDocCollector。

IndexSearch::search->searchAfter->search->search->search->SimpleTopScoreDocCollector::getLeafCollector

    public LeafCollector getLeafCollector(LeafReaderContext context)
        throws IOException {
      final int docBase = context.docBase;
      return new ScorerLeafCollector() {
    
        @Override
        public void collect(int doc) throws IOException {
          float score = scorer.score();
          totalHits++;
          if (score <= pqTop.score) {
            return;
          }
          pqTop.doc = doc + docBase;
          pqTop.score = score;
          pqTop = pq.updateTop();
        }
    
      };
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
getLeafCollector函数创建ScorerLeafCollector并返回。

回到search函数中，接下来通过Weight的bulkScorer函数获得BulkScorer，用来计算得分。

bulkScorer
假设通过createNormalizedWeight函数创建的Weight为BooleanWeight，下面来看其bulkScorer函数，

IndexSearcher::search->searchAfter->search->search->search->BooleanWeight::bulkScorer

  public BulkScorer bulkScorer(LeafReaderContext context) throws IOException {
    final BulkScorer bulkScorer = booleanScorer(context);
    if (bulkScorer != null) {
      return bulkScorer;
    } else {
      return super.bulkScorer(context);
    }
  }
1
2
3
4
5
6
7
8
bulkScorer函数首先创建一个booleanScorer，假设为null，下面调用其父类Weight的bulkScorer函数并返回。

IndexSearcher::search->searchAfter->search->search->search->Weight::bulkScorer

  public BulkScorer bulkScorer(LeafReaderContext context) throws IOException {

    Scorer scorer = scorer(context);
    if (scorer == null) {
      return null;
    }
    
    return new DefaultBulkScorer(scorer);
  }
1
2
3
4
5
6
7
8
9
scorer函数重定义在BooleanWeight中，

IndexSearcher::search->searchAfter->search->search->search->Weight::bulkScorer->BooleanWeight::scorer

  public Scorer scorer(LeafReaderContext context) throws IOException {
    int minShouldMatch = query.getMinimumNumberShouldMatch();

    List<Scorer> required = new ArrayList<>();
    List<Scorer> requiredScoring = new ArrayList<>();
    List<Scorer> prohibited = new ArrayList<>();
    List<Scorer> optional = new ArrayList<>();
    Iterator<BooleanClause> cIter = query.iterator();
    for (Weight w  : weights) {
      BooleanClause c =  cIter.next();
      Scorer subScorer = w.scorer(context);
      if (subScorer == null) {
        if (c.isRequired()) {
          return null;
        }
      } else if (c.isRequired()) {
        required.add(subScorer);
        if (c.isScoring()) {
          requiredScoring.add(subScorer);
        }
      } else if (c.isProhibited()) {
        prohibited.add(subScorer);
      } else {
        optional.add(subScorer);
      }
    }
    
    if (optional.size() == minShouldMatch) {
      required.addAll(optional);
      requiredScoring.addAll(optional);
      optional.clear();
      minShouldMatch = 0;
    }
    
    if (required.isEmpty() && optional.isEmpty()) {
      return null;
    } else if (optional.size() < minShouldMatch) {
      return null;
    }
    
    if (!needsScores && minShouldMatch == 0 && required.size() > 0) {
      optional.clear();
    }
    
    if (optional.isEmpty()) {
      return excl(req(required, requiredScoring, disableCoord), prohibited);
    }
    
    if (required.isEmpty()) {
      return excl(opt(optional, minShouldMatch, disableCoord), prohibited);
    }
    
    Scorer req = excl(req(required, requiredScoring, true), prohibited);
    Scorer opt = opt(optional, minShouldMatch, true);
    
    if (disableCoord) {
      if (minShouldMatch > 0) {
        return new ConjunctionScorer(this, Arrays.asList(req, opt), Arrays.asList(req, opt), 1F);
      } else {
        return new ReqOptSumScorer(req, opt);          
      }
    } else if (optional.size() == 1) {
      if (minShouldMatch > 0) {
        return new ConjunctionScorer(this, Arrays.asList(req, opt), Arrays.asList(req, opt), coord(requiredScoring.size()+1, maxCoord));
      } else {
        float coordReq = coord(requiredScoring.size(), maxCoord);
        float coordBoth = coord(requiredScoring.size() + 1, maxCoord);
        return new BooleanTopLevelScorers.ReqSingleOptScorer(req, opt, coordReq, coordBoth);
      }
    } else {
      if (minShouldMatch > 0) {
        return new BooleanTopLevelScorers.CoordinatingConjunctionScorer(this, coords, req, requiredScoring.size(), opt);
      } else {
        return new BooleanTopLevelScorers.ReqMultiOptScorer(req, opt, requiredScoring.size(), coords); 
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
BooleanWeight的scorer函数会循环调用每个子查询对应的Weight的scorer函数，假设为TermWeight。

IndexSearcher::search->searchAfter->search->search->search->Weight::bulkScorer->BooleanWeight::scorer->TermWeight::scorer

    public Scorer scorer(LeafReaderContext context) throws IOException {
      final TermsEnum termsEnum = getTermsEnum(context);
      PostingsEnum docs = termsEnum.postings(null, needsScores ? PostingsEnum.FREQS : PostingsEnum.NONE);
      return new TermScorer(this, docs, similarity.simScorer(stats, context));
    }
1
2
3
4
5
IndexSearcher::search->searchAfter->search->search->search->Weight::bulkScorer->BooleanWeight::scorer->TermWeight::scorer->getTermsEnum

    private TermsEnum getTermsEnum(LeafReaderContext context) throws IOException {
      final TermState state = termStates.get(context.ord);
      final TermsEnum termsEnum = context.reader().terms(term.field())
          .iterator();
      termsEnum.seekExact(term.bytes(), state);
      return termsEnum;
    }
1
2
3
4
5
6
7
首先获得前面查询的结果TermState，iterator函数返回SegmentTermsEnum。SegmentTermsEnum的seekExact函数主要是封装前面的查询结果TermState，具体的细节下一章再研究。

回到TermWeight的scorer函数中，接下来调用SegmentTermsEnum的postings函数，最终调用Lucene50PostingsReader的postings函数。

IndexSearcher::search->searchAfter->search->search->search->Weight::bulkScorer->BooleanWeight::scorer->TermWeight::scorer->Lucene50PostingsReader::postings

  public PostingsEnum postings(FieldInfo fieldInfo, BlockTermState termState, PostingsEnum reuse, int flags) throws IOException {

    boolean indexHasPositions = fieldInfo.getIndexOptions().compareTo(IndexOptions.DOCS_AND_FREQS_AND_POSITIONS) >= 0;
    boolean indexHasOffsets = fieldInfo.getIndexOptions().compareTo(IndexOptions.DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS) >= 0;
    boolean indexHasPayloads = fieldInfo.hasPayloads();
    
    if (indexHasPositions == false || PostingsEnum.featureRequested(flags, PostingsEnum.POSITIONS) == false) {
      BlockDocsEnum docsEnum;
      if (reuse instanceof BlockDocsEnum) {
        ...
      } else {
        docsEnum = new BlockDocsEnum(fieldInfo);
      }
      return docsEnum.reset((IntBlockTermState) termState, flags);
    } else if ((indexHasOffsets == false || PostingsEnum.featureRequested(flags, PostingsEnum.OFFSETS) == false) &&
               (indexHasPayloads == false || PostingsEnum.featureRequested(flags, PostingsEnum.PAYLOADS) == false)) {
        ...
    } else {
        ...
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
首先取出索引文件中的存储类型。假设进入第一个if语句，reuse参数默认为null，因此接下来创建一个BlockDocsEnum，并通过reset函数初始化。

IndexSearcher::search->searchAfter->search->search->search->Weight::bulkScorer->BooleanWeight::scorer->TermWeight::scorer->BM25Similarity::simScorer

  public final SimScorer simScorer(SimWeight stats, LeafReaderContext context) throws IOException {
    BM25Stats bm25stats = (BM25Stats) stats;
    return new BM25DocScorer(bm25stats, context.reader().getNormValues(bm25stats.field));
  }

  public final NumericDocValues getNormValues(String field) throws IOException {
    ensureOpen();
    Map<String,NumericDocValues> normFields = normsLocal.get();

    NumericDocValues norms = normFields.get(field);
    if (norms != null) {
      return norms;
    } else {
      FieldInfo fi = getFieldInfos().fieldInfo(field);
      if (fi == null || !fi.hasNorms()) {
        return null;
      }
      norms = getNormsReader().getNorms(fi);
      normFields.put(field, norms);
      return norms;
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
getNormsReader返回Lucene53NormsProducer，Lucene53NormsProducer的getNorms函数根据域信息以及.nvd、.nvm文件读取信息创建NumericDocValues并返回。simScorer最终创建BM25DocScorer并返回。

回到TermWeight的scorer函数中，最后创建TermScorer并返回。

再回到BooleanWeight的scorer函数中，如果SHOULD条件的Scorer等于minShouldMatch，则表明及时条件为SHOULD但也必须得到满足，此时将其归入MUST条件中。再往下，如果SHOULD以及MUST对应的Scorer都为空，则表明没有任何查询条件，返回空，如果SHOULD条件的Scorer小于minShouldMatch，则表明SHOULD条件下查询到的匹配字符太少，也返回空。再往下，如果optional为空，则没有SHOULD条件的Scorer，此时通过req封装MUST条件的Scorer，并通过excl排除MUST_NOT条件的Scorer；相反，如果required为空，则没有MUST条件的Scorer，此时通过opt封装SHOULD条件的Scorer。

IndexSearcher::search->searchAfter->search->search->search->Weight::bulkScorer->BooleanWeight::scorer->req

  private Scorer req(List<Scorer> required, List<Scorer> requiredScoring, boolean disableCoord) {
    if (required.size() == 1) {
      Scorer req = required.get(0);

      if (needsScores == false) {
        return req;
      }
    
      if (requiredScoring.isEmpty()) {
        return new FilterScorer(req) {
          @Override
          public float score() throws IOException {
            return 0f;
          }
          @Override
          public int freq() throws IOException {
            return 0;
          }
        };
      }
    
      float boost = 1f;
      if (disableCoord == false) {
        boost = coord(1, maxCoord);
      }
      if (boost == 1f) {
        return req;
      }
      return new BooleanTopLevelScorers.BoostedScorer(req, boost);
    } else {
      return new ConjunctionScorer(this, required, requiredScoring,
                                   disableCoord ? 1.0F : coord(requiredScoring.size(), maxCoord));
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
如果MUST条件下的Scorer数量大于1，则直接创建ConjunctionScorer并返回，如果requiredScoring为空，则对应唯一的MUST条件下的Scorer并不需要评分，此时直接返回FilterScorer，否则通过计算返回BoostedScorer，其他情况下直接返回对应的Scorer。

IndexSearcher::search->searchAfter->search->search->search->Weight::bulkScorer->BooleanWeight::scorer->excl

  private Scorer excl(Scorer main, List<Scorer> prohibited) throws IOException {
    if (prohibited.isEmpty()) {
      return main;
    } else if (prohibited.size() == 1) {
      return new ReqExclScorer(main, prohibited.get(0));
    } else {
      float coords[] = new float[prohibited.size()+1];
      Arrays.fill(coords, 1F);
      return new ReqExclScorer(main, new DisjunctionSumScorer(this, prohibited, coords, false));
    }
  }
1
2
3
4
5
6
7
8
9
10
11
excl根据是否有MUST_NOT条件的Scorer将Scorer进一步封装成ReqExclScorer（表示需要排除prohibited中的Scorer）或者直接返回。

IndexSearcher::search->searchAfter->search->search->search->Weight::bulkScorer->BooleanWeight::scorer->opt

  private Scorer opt(List<Scorer> optional, int minShouldMatch, boolean disableCoord) throws IOException {
    if (optional.size() == 1) {
      Scorer opt = optional.get(0);
      if (!disableCoord && maxCoord > 1) {
        return new BooleanTopLevelScorers.BoostedScorer(opt, coord(1, maxCoord));
      } else {
        return opt;
      }
    } else {
      float coords[];
      if (disableCoord) {
        coords = new float[optional.size()+1];
        Arrays.fill(coords, 1F);
      } else {
        coords = this.coords;
      }
      if (minShouldMatch > 1) {
        return new MinShouldMatchSumScorer(this, optional, minShouldMatch, coords);
      } else {
        return new DisjunctionSumScorer(this, optional, coords, needsScores);
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
和req函数类似，opt函数根据不同情况将Scorer封装成BoostedScorer、MinShouldMatchSumScorer、DisjunctionSumScorer或者直接返回。

回到BooleanWeight的scorer函数中，接下来根据disableCoord、minShouldMatch和optional的值将Scorer进一步封装成ConjunctionScorer、ReqOptSumScorer、ReqSingleOptScorer、CoordinatingConjunctionScorer或者ReqMultiOptScorer。

回到Weight的bulkScorer和BooleanWeight的bulkScorer函数中，接下来将scorer函数返回的结果进一步封装成DefaultBulkScorer并返回。

再向上回到IndexSearcher的search函数中，接下来调用刚刚创建的DefaultBulkScorer的score函数，
IndexSearch::search->searchAfter->search->search->search->DefaultBulkScorer::score

  public void score(LeafCollector collector, Bits acceptDocs) throws IOException {
    final int next = score(collector, acceptDocs, 0, DocIdSetIterator.NO_MORE_DOCS);
  }

  public int score(LeafCollector collector, Bits acceptDocs, int min, int max) throws IOException {
    collector.setScorer(scorer);
    if (scorer.docID() == -1 && min == 0 && max == DocIdSetIterator.NO_MORE_DOCS) {
      scoreAll(collector, iterator, twoPhase, acceptDocs);
      return DocIdSetIterator.NO_MORE_DOCS;
    } else {
      ...
    }
  }

  static void scoreAll(LeafCollector collector, DocIdSetIterator iterator, TwoPhaseIterator twoPhase, Bits acceptDocs) throws IOException {
    if (twoPhase == null) {
      for (int doc = iterator.nextDoc(); doc != DocIdSetIterator.NO_MORE_DOCS; doc = iterator.nextDoc()) {
        if (acceptDocs == null || acceptDocs.get(doc)) {
          collector.collect(doc);
        }
      }
    } else {
      ...
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
默认情况下，需要对所有命中文档进行评分，因此这里往下看scoreAll函数，首先遍历iterator，然后依次调用ScorerLeafCollector的collect函数进行打分。

IndexSearch::search->searchAfter->search->search->search->DefaultBulkScorer::score->score->scoreAll->ScorerLeafCollector::collect

        public void collect(int doc) throws IOException {
          float score = scorer.score();
    
          totalHits++;
          if (score <= pqTop.score) {
            return;
          }
          pqTop.doc = doc + docBase;
          pqTop.score = score;
          pqTop = pq.updateTop();
        }
1
2
3
4
5
6
7
8
9
10
11
成员变量scorer假设为ReqOptSumScorer，对应的score函数计算文档的分数，因为本博文不涉及lucene的打分算法，因此就不往下看了。最后设置刚刚计算的分数到队列pqTop中，并调用updateTop更新队列指针。

回到再上一层的search函数中，下面来看CollectorManager的reduce函数。

IndexSearch::search->searchAfter->search->CollectorManager::reduce

      public TopDocs reduce(Collection<TopScoreDocCollector> collectors) throws IOException {
        final TopDocs[] topDocs = new TopDocs[collectors.size()];
        int i = 0;
        for (TopScoreDocCollector collector : collectors) {
          topDocs[i++] = collector.topDocs();
        }
        return TopDocs.merge(cappedNumHits, topDocs);
      }
1
2
3
4
5
6
7
8
这里遍历collectors，依次取出SimpleTopScoreDoc并调用其topDocs函数取出前面的文档信息。

IndexSearch::search->searchAfter->search->CollectorManager::reduce->SimpleTopScoreDoc::topDocs

  public TopDocs topDocs() {
    return topDocs(0, topDocsSize());
  }

  public TopDocs topDocs(int start, int howMany) {

    int size = topDocsSize();
    if (start < 0 || start >= size || howMany <= 0) {
      return newTopDocs(null, start);
    }
    
    howMany = Math.min(size - start, howMany);
    ScoreDoc[] results = new ScoreDoc[howMany];
    
    for (int i = pq.size() - start - howMany; i > 0; i--) { pq.pop(); }
    
    populateResults(results, howMany);
    
    return newTopDocs(results, start);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
topDocsSize返回命中的文档数，pop函数删除队列中没有用的ScoreDoc，populateResults函数复制最后剩下的ScoreDoc至results中，最后通过newTopDocs函数创建TopDocs封装ScoreDoc数组。

回到CollectorManager的reduce函数中，最后通过merge函数合并不同TopScoreDocCollector产生的TopDocs结果。

IndexSearch::search->searchAfter->search->CollectorManager::reduce->TopDocs::merge

  public static TopDocs merge(int topN, TopDocs[] shardHits) throws IOException {
    return merge(0, topN, shardHits);
  }

  public static TopDocs merge(int start, int topN, TopDocs[] shardHits) throws IOException {
    return mergeAux(null, start, topN, shardHits);
  }

  private static TopDocs mergeAux(Sort sort, int start, int size, TopDocs[] shardHits) throws IOException {
    final PriorityQueue<ShardRef> queue;
    if (sort == null) {
      queue = new ScoreMergeSortQueue(shardHits);
    } else {
      queue = new MergeSortQueue(sort, shardHits);
    }

    int totalHitCount = 0;
    int availHitCount = 0;
    float maxScore = Float.MIN_VALUE;
    for(int shardIDX=0;shardIDX<shardHits.length;shardIDX++) {
      final TopDocs shard = shardHits[shardIDX];
      totalHitCount += shard.totalHits;
      if (shard.scoreDocs != null && shard.scoreDocs.length > 0) {
        availHitCount += shard.scoreDocs.length;
        queue.add(new ShardRef(shardIDX));
        maxScore = Math.max(maxScore, shard.getMaxScore());
      }
    }
    
    if (availHitCount == 0) {
      maxScore = Float.NaN;
    }
    
    final ScoreDoc[] hits;
    if (availHitCount <= start) {
      hits = new ScoreDoc[0];
    } else {
      hits = new ScoreDoc[Math.min(size, availHitCount - start)];
      int requestedResultWindow = start + size;
      int numIterOnHits = Math.min(availHitCount, requestedResultWindow);
      int hitUpto = 0;
      while (hitUpto < numIterOnHits) {
        ShardRef ref = queue.top();
        final ScoreDoc hit = shardHits[ref.shardIndex].scoreDocs[ref.hitIndex++];
        hit.shardIndex = ref.shardIndex;
        if (hitUpto >= start) {
          hits[hitUpto - start] = hit;
        }
        hitUpto++;
        if (ref.hitIndex < shardHits[ref.shardIndex].scoreDocs.length) {
          queue.updateTop();
        } else {
          queue.pop();
        }
      }
    }
    
    if (sort == null) {
      return new TopDocs(totalHitCount, hits, maxScore);
    } else {
      return new TopFieldDocs(totalHitCount, hits, sort.getSort(), maxScore);
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
mergeAux简单来说就是合并多个TopDocs为一个TopDocs并返回。

到此IndexSearcher的search函数的大体过程就分析到这了，下一章开始分析lucene读取索引文件的具体函数。
————————————————
版权声明：本文为CSDN博主「二侠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/conansonic/article/details/52041923

lucene源码分析—倒排索引的写过程
本章介绍倒排索引的写过程，下一章再介绍其读过程，和前几章相似，本章所有代码会基于原有代码进行少量的改写，方便阅读，省略了一些不重要的部分。
lucene将倒排索引的信息写入.tim和.tip文件，这部分代码也是lucene最核心的一部分。倒排索引的写过程从BlockTreeTermsWriter的write函数开始，

BlockTreeTermsWriter::write

  public void write(Fields fields) throws IOException {

    String lastField = null;
    for(String field : fields) {
      lastField = field;
    
      Terms terms = fields.terms(field);
      if (terms == null) {
        continue;
      }
      List<PrefixTerm> prefixTerms = null;
    
      TermsEnum termsEnum = terms.iterator();
      TermsWriter termsWriter = new TermsWriter(fieldInfos.fieldInfo(field));
      int prefixTermUpto = 0;
    
      while (true) {
        BytesRef term = termsEnum.next();
        termsWriter.write(term, termsEnum, null);
      }
      termsWriter.finish();
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
遍历每个域，首先通过terms函数根据field名返回一个FreqProxTerms，包含了该域的所有Term；接下来fieldInfo根据域名返回域信息，并以此创建一个TermsWriter，TermsWriter是倒排索引写的主要类，接下来依次取出FreqProxTerms中的每个term，并调用TermsWriter的write函数写入.tim文件，并创建对应的索引信息，最后通过TermsWriter的finish函数将索引信息写入.tip文件中，下面依次来看。

BlockTreeTermsWriter::write->TermsWriter::write

    public void write(BytesRef text, TermsEnum termsEnum, PrefixTerm prefixTerm) throws IOException {
    
      BlockTermState state = postingsWriter.writeTerm(text, termsEnum, docsSeen);
      if (state != null) {
    
        pushTerm(text);
    
        PendingTerm term = new PendingTerm(text, state, prefixTerm);
        pending.add(term);
    
        if (prefixTerm == null) {
          sumDocFreq += state.docFreq;
          sumTotalTermFreq += state.totalTermFreq;
          numTerms++;
          if (firstPendingTerm == null) {
            firstPendingTerm = term;
          }
          lastPendingTerm = term;
        }
      }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
TermsWriter的write函数一次处理一个Term。postingsWriter是Lucene50PostingsWriter。write函数首先通过Lucene50PostingsWriter的writeTerm函数记录每个Term以及对应文档的相应信息。
成员变量pending是一个PendingEntry列表，PendingEntry用来保存一个Term或者是一个Block，pending列表用来保存多个待处理的Term。
pushTerm是write里的核心函数，用于具体处理一个Term，后面详细来看。write函数的最后统计文档频和词频信息并记录到sumDocFreq和sumTotalTermFreq两个成员变量中。

BlockTreeTermsWriter::write->TermsWriter::write->Lucene50PostingsWriter::writeTerm

  public final BlockTermState writeTerm(BytesRef term, TermsEnum termsEnum, FixedBitSet docsSeen) throws IOException {
    startTerm();
    postingsEnum = termsEnum.postings(postingsEnum, enumFlags);

    int docFreq = 0;
    long totalTermFreq = 0;
    while (true) {
      int docID = postingsEnum.nextDoc();
      if (docID == PostingsEnum.NO_MORE_DOCS) {
        break;
      }
      docFreq++;
      docsSeen.set(docID);
      int freq;
      if (writeFreqs) {
        freq = postingsEnum.freq();
        totalTermFreq += freq;
      } else {
        freq = -1;
      }
      startDoc(docID, freq);
    
      if (writePositions) {
        for(int i=0;i<freq;i++) {
          int pos = postingsEnum.nextPosition();
          BytesRef payload = writePayloads ? postingsEnum.getPayload() : null;
          int startOffset;
          int endOffset;
          if (writeOffsets) {
            startOffset = postingsEnum.startOffset();
            endOffset = postingsEnum.endOffset();
          } else {
            startOffset = -1;
            endOffset = -1;
          }
          addPosition(pos, payload, startOffset, endOffset);
        }
      }
    
      finishDoc();
    }
    
    if (docFreq == 0) {
      return null;
    } else {
      BlockTermState state = newTermState();
      state.docFreq = docFreq;
      state.totalTermFreq = writeFreqs ? totalTermFreq : -1;
      finishTerm(state);
      return state;
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
startTerm设置.doc、.pos和.pay三个文件的指针。postings函数创建FreqProxPostingsEnum或者FreqProxDocsEnum，内部封装了FreqProxTermsWriterPerField，即第五章中每个PerField的termsHashPerField成员变量，termsHashPerField的内部保存了对应Field的所有Terms信息。
writeTerm函数接下来通过nextDoc获得下一个文档ID，获得freq词频，并累加到totalTermFreq（总词频）中。再调用startDoc记录文档的信息。addPosition函数记录词的位置、偏移和payload信息，必要时写入文件中。finishDoc记录文件指针等信息。然后创建BlockTermState，设置相应词频和文档频信息以最终返回。
writeTerm函数最后通过finishTerm写入文档信息至.doc文件，写入位置信息至.pos文件。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm

    private void pushTerm(BytesRef text) throws IOException {
    
      int limit = Math.min(lastTerm.length(), text.length);
      int pos = 0;
    
      while (pos < limit && lastTerm.byteAt(pos) == text.bytes[text.offset+pos]) {
        pos++;
      }
    
      for(int i=lastTerm.length()-1;i>=pos;i--) {
    
        int prefixTopSize = pending.size() - prefixStarts[i];
        if (prefixTopSize >= minItemsInBlock) {
          writeBlocks(i+1, prefixTopSize);
          prefixStarts[i] -= prefixTopSize-1;
        }
      }
    
      if (prefixStarts.length < text.length) {
        prefixStarts = ArrayUtil.grow(prefixStarts, text.length);
      }
    
      for(int i=pos;i<text.length;i++) {
        prefixStarts[i] = pending.size();
      }
    
      lastTerm.copyBytes(text);
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
lastTerm保存了上一次处理的Term。pushTerm函数的核心功能是计算一定的条件，当满足一定条件时，就表示pending列表中待处理的一个或者多个Term，需要保存为一个block，此时调用writeBlocks函数进行保存。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm->writeBlocks

    void writeBlocks(int prefixLength, int count) throws IOException {
      int lastSuffixLeadLabel = -1;
    
      boolean hasTerms = false;
      boolean hasPrefixTerms = false;
      boolean hasSubBlocks = false;
    
      int start = pending.size()-count;
      int end = pending.size();
      int nextBlockStart = start;
      int nextFloorLeadLabel = -1;
    
      for (int i=start; i<end; i++) {
    
        PendingEntry ent = pending.get(i);
        int suffixLeadLabel;
    
        if (ent.isTerm) {
          PendingTerm term = (PendingTerm) ent;
          if (term.termBytes.length == prefixLength) {
            suffixLeadLabel = -1;
          } else {
            suffixLeadLabel = term.termBytes[prefixLength] & 0xff;
          }
        } else {
          PendingBlock block = (PendingBlock) ent;
          suffixLeadLabel = block.prefix.bytes[block.prefix.offset + prefixLength] & 0xff;
        }
    
        if (suffixLeadLabel != lastSuffixLeadLabel) {
          int itemsInBlock = i - nextBlockStart;
          if (itemsInBlock >= minItemsInBlock && end-nextBlockStart > maxItemsInBlock) {
            boolean isFloor = itemsInBlock < count;
            newBlocks.add(writeBlock(prefixLength, isFloor, nextFloorLeadLabel, nextBlockStart, i, hasTerms, hasPrefixTerms, hasSubBlocks));
    
            hasTerms = false;
            hasSubBlocks = false;
            hasPrefixTerms = false;
            nextFloorLeadLabel = suffixLeadLabel;
            nextBlockStart = i;
          }
    
          lastSuffixLeadLabel = suffixLeadLabel;
        }
    
        if (ent.isTerm) {
          hasTerms = true;
          hasPrefixTerms |= ((PendingTerm) ent).prefixTerm != null;
        } else {
          hasSubBlocks = true;
        }
      }
    
      if (nextBlockStart < end) {
        int itemsInBlock = end - nextBlockStart;
        boolean isFloor = itemsInBlock < count;
        newBlocks.add(writeBlock(prefixLength, isFloor, nextFloorLeadLabel, nextBlockStart, end, hasTerms, hasPrefixTerms, hasSubBlocks));
      }
      PendingBlock firstBlock = newBlocks.get(0);
      firstBlock.compileIndex(newBlocks, scratchBytes, scratchIntsRef);
    
      pending.subList(pending.size()-count, pending.size()).clear();
      pending.add(firstBlock);
      newBlocks.clear();
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
hasTerms表示将要合并的项中是否含有Term（因为特殊情况下，合并的项只有子block）。
hasPrefixTerms表示是否有词的前缀，假设一直为false。
hasSubBlocks和hasTerms对应，表示将要合并的项中是否含有子block。
start和end的规定了需要合并的Term或Block在待处理的pending列表中的范围。
writeBlocks函数接下来遍历pending列表中每个待处理的Term或者Block，suffixLeadLabel保存了树中某个节点下的各个Term的byte，lastSuffixLeadLabel则是对应的最后一个不同的byte，检查所有项中是否有Term和子block，并对hasTerms和hasSubBlocks进行相应的设置。如果pending中的Term或block太多，大于minItemsInBlock和maxItemsInBlock计算出来的阈值，就会调用writeBlock写成一个block，最后也会写一次。
writeBlocks函数接下来通过compileIndex函数将一个block的信息写入FST结构中（保存在其成员变量index中），FST是有限状态机的缩写，其实就是将一棵树的信息保存在其自身的结构中，而这颗树是由所有Term的每个byte形成的，后面来看。
writeBlocks函数最后清空被保存的一部分pending列表，并添加刚刚创建的block到pending列表中。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm->writeBlocks->writeBlock
第一种情况

    private PendingBlock writeBlock(int prefixLength, boolean isFloor, int floorLeadLabel, int start, int end, boolean hasTerms, boolean hasPrefixTerms, boolean hasSubBlocks) throws IOException {
    
      long startFP = termsOut.getFilePointer();
    
      boolean hasFloorLeadLabel = isFloor && floorLeadLabel != -1;
    
      final BytesRef prefix = new BytesRef(prefixLength + (hasFloorLeadLabel ? 1 : 0));
      System.arraycopy(lastTerm.get().bytes, 0, prefix.bytes, 0, prefixLength);
      prefix.length = prefixLength;
    
      int numEntries = end - start;
      int code = numEntries << 1;
      if (end == pending.size()) {
        code |= 1;
      }
      termsOut.writeVInt(code);
    
      boolean isLeafBlock = hasSubBlocks == false && hasPrefixTerms == false;
      final List<FST<BytesRef>> subIndices;
      boolean absolute = true;
    
      if (isLeafBlock) {
        subIndices = null;
        for (int i=start;i<end;i++) {
          PendingEntry ent = pending.get(i);
    
          PendingTerm term = (PendingTerm) ent;
          BlockTermState state = term.state;
          final int suffix = term.termBytes.length - prefixLength;
    
          suffixWriter.writeVInt(suffix);
          suffixWriter.writeBytes(term.termBytes, prefixLength, suffix);
          statsWriter.writeVInt(state.docFreq);
          if (fieldInfo.getIndexOptions() != IndexOptions.DOCS) {
            statsWriter.writeVLong(state.totalTermFreq - state.docFreq);
          }
    
          postingsWriter.encodeTerm(longs, bytesWriter, fieldInfo, state, absolute);
          for (int pos = 0; pos < longsSize; pos++) {
            metaWriter.writeVLong(longs[pos]);
          }
          bytesWriter.writeTo(metaWriter);
          bytesWriter.reset();
          absolute = false;
        }
      } else {
        ...
      }
    
      termsOut.writeVInt((int) (suffixWriter.getFilePointer() << 1) | (isLeafBlock ? 1:0));
      suffixWriter.writeTo(termsOut);
      suffixWriter.reset();
    
      termsOut.writeVInt((int) statsWriter.getFilePointer());
      statsWriter.writeTo(termsOut);
      statsWriter.reset();
    
      termsOut.writeVInt((int) metaWriter.getFilePointer());
      metaWriter.writeTo(termsOut);
      metaWriter.reset();
    
      if (hasFloorLeadLabel) {
        prefix.bytes[prefix.length++] = (byte) floorLeadLabel;
      }
    
      return new PendingBlock(prefix, startFP, hasTerms, isFloor, floorLeadLabel, subIndices);
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
termsOut封装了.tim文件的输出流，其实是FSIndexOutput，其getFilePointer函数返回的startFP保存了该文件可以插入的指针。
writeBlock函数首先提取相同的前缀，例如需要写为一个block的Term有aaa，aab，aac，则相同的前缀为aa，保存在类型为BytesRef的prefix中，BytesRef用于封装一个byte数组。
numEntries保存了本次需要写入多少个Term或者Block，code封装了numEntries的信息，并在最后一个bit表示后面是否还有。然后将code写入.tim文件中。
isLeafBlock表示是否是叶子节点。bytesWriter、suffixWriter、statsWriter、metaWriter在内存中模拟文件。
writeBlock函数接下来遍历需要写入的Term或者Block，suffix表示最后取出的不同字幕的长度，例如aaa，aab，aac则suffix为1，首先写入该长度suffix，最终写入suffixWriter中的为a、b、c。再往下往statsWriter中写入词频和文档频率。
再往下postingsWriter是Lucene50PostingsWriter，encodeTerm函数在longs中保存了.doc、.pos和.pay中文件指针的偏移，然后singletonDocID、lastPosBlockOffset、skipOffset等信息保存在bytesWriter中，再将longs的指针写入metaWriter中，最后把其余信息写入bytesWriter中。
再往下调用bytesWriter、suffixWriter、statsWriter、metaWriter的writeTo函数将内存中的数据写入.tim文件中。
writeBlock函数最后创建PendingBlock并返回，PendingBlock封装了本次写入的各个Term或者子Block的信息。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm->writeBlocks->writeBlock
第二种情况

    private PendingBlock writeBlock(int prefixLength, boolean isFloor, int floorLeadLabel, int start, int end, boolean hasTerms, boolean hasPrefixTerms, boolean hasSubBlocks) throws IOException {
    
      long startFP = termsOut.getFilePointer();
    
      boolean hasFloorLeadLabel = isFloor && floorLeadLabel != -1;
    
      final BytesRef prefix = new BytesRef(prefixLength + (hasFloorLeadLabel ? 1 : 0));
      System.arraycopy(lastTerm.get().bytes, 0, prefix.bytes, 0, prefixLength);
      prefix.length = prefixLength;
    
      int numEntries = end - start;
      int code = numEntries << 1;
      if (end == pending.size()) {
        code |= 1;
      }
      termsOut.writeVInt(code);
    
      boolean isLeafBlock = hasSubBlocks == false && hasPrefixTerms == false;
      final List<FST<BytesRef>> subIndices;
      boolean absolute = true;
    
      if (isLeafBlock) {
        ...
      } else {
        subIndices = new ArrayList<>();
        boolean sawAutoPrefixTerm = false;
        for (int i=start;i<end;i++) {
          PendingEntry ent = pending.get(i);
          if (ent.isTerm) {
            PendingTerm term = (PendingTerm) ent;
            BlockTermState state = term.state;
            final int suffix = term.termBytes.length - prefixLength;
            if (minItemsInAutoPrefix == 0) {
              suffixWriter.writeVInt(suffix << 1);
              suffixWriter.writeBytes(term.termBytes, prefixLength, suffix);
            } else {
              code = suffix<<2;
              int floorLeadEnd = -1;
              if (term.prefixTerm != null) {
                sawAutoPrefixTerm = true;
                PrefixTerm prefixTerm = term.prefixTerm;
                floorLeadEnd = prefixTerm.floorLeadEnd;
    
                if (prefixTerm.floorLeadStart == -2) {
                  code |= 2;
                } else {
                  code |= 3;
                }
              }
              suffixWriter.writeVInt(code);
              suffixWriter.writeBytes(term.termBytes, prefixLength, suffix);
              if (floorLeadEnd != -1) {
                suffixWriter.writeByte((byte) floorLeadEnd);
              }
            }
    
            statsWriter.writeVInt(state.docFreq);
            if (fieldInfo.getIndexOptions() != IndexOptions.DOCS) {
              statsWriter.writeVLong(state.totalTermFreq - state.docFreq);
            }
            postingsWriter.encodeTerm(longs, bytesWriter, fieldInfo, state, absolute);
            for (int pos = 0; pos < longsSize; pos++) {
              metaWriter.writeVLong(longs[pos]);
            }
            bytesWriter.writeTo(metaWriter);
            bytesWriter.reset();
            absolute = false;
          } else {
            PendingBlock block = (PendingBlock) ent;
            final int suffix = block.prefix.length - prefixLength;
            if (minItemsInAutoPrefix == 0) {
              suffixWriter.writeVInt((suffix<<1)|1);
            } else {
              suffixWriter.writeVInt((suffix<<2)|1);
            }
            suffixWriter.writeBytes(block.prefix.bytes, prefixLength, suffix);
            suffixWriter.writeVLong(startFP - block.fp);
            subIndices.add(block.index);
          }
        }
      }
    
      termsOut.writeVInt((int) (suffixWriter.getFilePointer() << 1) | (isLeafBlock ? 1:0));
      suffixWriter.writeTo(termsOut);
      suffixWriter.reset();
    
      termsOut.writeVInt((int) statsWriter.getFilePointer());
      statsWriter.writeTo(termsOut);
      statsWriter.reset();
    
      termsOut.writeVInt((int) metaWriter.getFilePointer());
      metaWriter.writeTo(termsOut);
      metaWriter.reset();
    
      if (hasFloorLeadLabel) {
        prefix.bytes[prefix.length++] = (byte) floorLeadLabel;
      }
    
      return new PendingBlock(prefix, startFP, hasTerms, isFloor, floorLeadLabel, subIndices);
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
第二种情况表示要写入的不是叶子节点，如果是Term，和第一部分一样，如果是一个子block，写入子block的相应信息，最后创建的PendingBlock需要封装每个Block对应的FST结构，即subIndices。

writeBlocks函数调用完writeBlock函数后将pending列表中的Term或者Block写入.tim文件中，接下来要通过PendingBlock的compileIndex函数针对刚刚写入.tim文件中的Term创建索引信息，最后要将这些信息写入.tip文件中，用于查找。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm->writeBlocks->PendingBlock::compileIndex

    public void compileIndex(List<PendingBlock> blocks, RAMOutputStream scratchBytes, IntsRefBuilder scratchIntsRef) throws IOException {
    
      scratchBytes.writeVLong(encodeOutput(fp, hasTerms, isFloor));
      if (isFloor) {
        scratchBytes.writeVInt(blocks.size()-1);
        for (int i=1;i<blocks.size();i++) {
          PendingBlock sub = blocks.get(i);
          scratchBytes.writeByte((byte) sub.floorLeadByte);
          scratchBytes.writeVLong((sub.fp - fp) << 1 | (sub.hasTerms ? 1 : 0));
        }
      }
    
      final ByteSequenceOutputs outputs = ByteSequenceOutputs.getSingleton();
      final Builder<BytesRef> indexBuilder = new Builder<>(FST.INPUT_TYPE.BYTE1,
                                                           0, 0, true, false, Integer.MAX_VALUE,
                                                           outputs, false,
                                                           PackedInts.COMPACT, true, 15);
    
      final byte[] bytes = new byte[(int) scratchBytes.getFilePointer()];
      scratchBytes.writeTo(bytes, 0);
      indexBuilder.add(Util.toIntsRef(prefix, scratchIntsRef), new BytesRef(bytes, 0, bytes.length));
      scratchBytes.reset();
    
      for(PendingBlock block : blocks) {
        if (block.subIndices != null) {
          for(FST<BytesRef> subIndex : block.subIndices) {
            append(indexBuilder, subIndex, scratchIntsRef);
          }
          block.subIndices = null;
        }
      }
      index = indexBuilder.finish();
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
fp是对应.tim文件的指针，encodeOutput函数将fp、hasTerms和isFloor信息封装到一个长整型中，然后将该长整型存入scratchBytes中。compileIndex函数接下来创建Builder，用于构造索引树，再往下将scratchBytes中的数据存入byte数组bytes中。
compileIndex最核心的部分是通过Builder的add函数依次将Term或者Term的部分前缀添加到一颗树中，由frontier数组维护，进而添加到FST中。compileIndex最后通过Builder的finish函数将add添加后的FST树中的信息写入缓存中，后续添加到.tip文件里。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm->writeBlocks->PendingBlock::compileIndex->Builder::Builder

  public Builder(FST.INPUT_TYPE inputType, int minSuffixCount1, int minSuffixCount2, boolean doShareSuffix, boolean doShareNonSingletonNodes, int shareMaxTailLength, Outputs<T> outputs, boolean doPackFST, float acceptableOverheadRatio, boolean allowArrayArcs, int bytesPageBits) {

    this.minSuffixCount1 = minSuffixCount1;
    this.minSuffixCount2 = minSuffixCount2;
    this.doShareNonSingletonNodes = doShareNonSingletonNodes;
    this.shareMaxTailLength = shareMaxTailLength;
    this.doPackFST = doPackFST;
    this.acceptableOverheadRatio = acceptableOverheadRatio;
    this.allowArrayArcs = allowArrayArcs;
    fst = new FST<>(inputType, outputs, doPackFST, acceptableOverheadRatio, bytesPageBits);
    bytes = fst.bytes;
    if (doShareSuffix) {
      dedupHash = new NodeHash<>(fst, bytes.getReverseReader(false));
    } else {
      dedupHash = null;
    }
    NO_OUTPUT = outputs.getNoOutput();
    
    final UnCompiledNode<T>[] f = (UnCompiledNode<T>[]) new UnCompiledNode[10];
    frontier = f;
    for(int idx=0;idx<frontier.length;idx++) {
      frontier[idx] = new UnCompiledNode<>(this, idx);
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
Builder的构造函数主要是创建了一个FST，并初始化frontier数组，frontier数组中的每个元素UnCompiledNode代表树中的每个节点。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm->writeBlocks->PendingBlock::compileIndex->Builder::add

  public void add(IntsRef input, T output) throws IOException {

    ...
    
    int pos1 = 0;
    int pos2 = input.offset;
    final int pos1Stop = Math.min(lastInput.length(), input.length);
    while(true) {
      frontier[pos1].inputCount++;
      if (pos1 >= pos1Stop || lastInput.intAt(pos1) != input.ints[pos2]) {
        break;
      }
      pos1++;
      pos2++;
    }
    final int prefixLenPlus1 = pos1+1;
    
    if (frontier.length < input.length+1) {
      final UnCompiledNode<T>[] next = ArrayUtil.grow(frontier, input.length+1);
      for(int idx=frontier.length;idx<next.length;idx++) {
        next[idx] = new UnCompiledNode<>(this, idx);
      }
      frontier = next;
    }
    
    freezeTail(prefixLenPlus1);
    
    for(int idx=prefixLenPlus1;idx<=input.length;idx++) {
      frontier[idx-1].addArc(input.ints[input.offset + idx - 1],
                             frontier[idx]);
      frontier[idx].inputCount++;
    }
    
    final UnCompiledNode<T> lastNode = frontier[input.length];
    if (lastInput.length() != input.length || prefixLenPlus1 != input.length + 1) {
      lastNode.isFinal = true;
      lastNode.output = NO_OUTPUT;
    }
    
    for(int idx=1;idx<prefixLenPlus1;idx++) {
      final UnCompiledNode<T> node = frontier[idx];
      final UnCompiledNode<T> parentNode = frontier[idx-1];
    
      final T lastOutput = parentNode.getLastOutput(input.ints[input.offset + idx - 1]);
    
      final T commonOutputPrefix;
      final T wordSuffix;
    
      if (lastOutput != NO_OUTPUT) {
        commonOutputPrefix = fst.outputs.common(output, lastOutput);
        wordSuffix = fst.outputs.subtract(lastOutput, commonOutputPrefix);
        parentNode.setLastOutput(input.ints[input.offset + idx - 1], commonOutputPrefix);
        node.prependOutput(wordSuffix);
      } else {
        commonOutputPrefix = wordSuffix = NO_OUTPUT;
      }
    
      output = fst.outputs.subtract(output, commonOutputPrefix);
    }
    
    if (lastInput.length() == input.length && prefixLenPlus1 == 1+input.length) {
      lastNode.output = fst.outputs.merge(lastNode.output, output);
    } else {
      frontier[prefixLenPlus1-1].setLastOutput(input.ints[input.offset + prefixLenPlus1-1], output);
    }
    lastInput.copyInts(input);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
add函数首先计算和上一个字符串的共同前缀，prefixLenPlus1表示FST数中的相同前缀的长度，如果存在，后面就需要进行相应的合并。接下来通过for循环调用addArc函数依次添加input即Term中的每个byte至frontier中，形成一个FST树，由frontier数组维护，然后设置frontier数组中的最后一个UnCompiledNode，将isFinal标志位设为true。add函数最后将output中的数据（文件指针等信息）存入本次frontier数组中最前面的一个UnCompiledNode中，并设置lastInput为本次的input。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm->writeBlocks->PendingBlock::compileIndex->Builder::add->freezeTail

  private void freezeTail(int prefixLenPlus1) throws IOException {
    final int downTo = Math.max(1, prefixLenPlus1);
    for(int idx=lastInput.length(); idx >= downTo; idx--) {

      boolean doPrune = false;
      boolean doCompile = false;
    
      final UnCompiledNode<T> node = frontier[idx];
      final UnCompiledNode<T> parent = frontier[idx-1];
    
      if (node.inputCount < minSuffixCount1) {
        doPrune = true;
        doCompile = true;
      } else if (idx > prefixLenPlus1) {
        if (parent.inputCount < minSuffixCount2 || (minSuffixCount2 == 1 && parent.inputCount == 1 && idx > 1)) {
          doPrune = true;
        } else {
          doPrune = false;
        }
        doCompile = true;
      } else {
        doCompile = minSuffixCount2 == 0;
      }
    
      if (node.inputCount < minSuffixCount2 || (minSuffixCount2 == 1 && node.inputCount == 1 && idx > 1)) {
        for(int arcIdx=0;arcIdx<node.numArcs;arcIdx++) {
          final UnCompiledNode<T> target = (UnCompiledNode<T>) node.arcs[arcIdx].target;
          target.clear();
        }
        node.numArcs = 0;
      }
    
      if (doPrune) {
        node.clear();
        parent.deleteLast(lastInput.intAt(idx-1), node);
      } else {
    
        if (minSuffixCount2 != 0) {
          compileAllTargets(node, lastInput.length()-idx);
        }
        final T nextFinalOutput = node.output;
    
        final boolean isFinal = node.isFinal || node.numArcs == 0;
    
        if (doCompile) {
          parent.replaceLast(lastInput.intAt(idx-1),
                             compileNode(node, 1+lastInput.length()-idx),
                             nextFinalOutput,
                             isFinal);
        } else {
          parent.replaceLast(lastInput.intAt(idx-1),
                             node,
                             nextFinalOutput,
                             isFinal);
          frontier[idx] = new UnCompiledNode<>(this, idx);
        }
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
freezeTail函数的核心功能是将不会再变化的节点通过compileNode函数添加到FST结构中。
replaceLast函数设置父节点对应的参数，例如其子节点在bytes中的位置target，是否为最后一个节点isFinal等等。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm->writeBlocks->PendingBlock::compileIndex->Builder::add->freezeTail->compileNode

  private CompiledNode compileNode(UnCompiledNode<T> nodeIn, int tailLength) throws IOException {
    final long node;
    long bytesPosStart = bytes.getPosition();
    if (dedupHash != null && (doShareNonSingletonNodes || nodeIn.numArcs <= 1) && tailLength <= shareMaxTailLength) {
      if (nodeIn.numArcs == 0) {
        node = fst.addNode(this, nodeIn);
        lastFrozenNode = node;
      } else {
        node = dedupHash.add(this, nodeIn);
      }
    } else {
      node = fst.addNode(this, nodeIn);
    }

    long bytesPosEnd = bytes.getPosition();
    if (bytesPosEnd != bytesPosStart) {
      lastFrozenNode = node;
    }
    
    nodeIn.clear();
    
    final CompiledNode fn = new CompiledNode();
    fn.node = node;
    return fn;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
compileNode的核心部分是调用FST的addNode函数添加节点。dedupHash是一个hash缓存，这里不管它。如果bytesPosEnd不等于bytesPosStart，表示有节点写入bytes中了，设置lastFrozenNode为当前node（其实是bytes中的缓存指针位置）。compileNode函数最后创建CompiledNode，设置其中的node并返回。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm->writeBlocks->PendingBlock::compileIndex->Builder::add->freezeTail->compileNode->FST::addNode

  long addNode(Builder<T> builder, Builder.UnCompiledNode<T> nodeIn) throws IOException {
    T NO_OUTPUT = outputs.getNoOutput();

    if (nodeIn.numArcs == 0) {
      if (nodeIn.isFinal) {
        return FINAL_END_NODE;
      } else {
        return NON_FINAL_END_NODE;
      }
    }
    
    final long startAddress = builder.bytes.getPosition();
    
    final boolean doFixedArray = shouldExpand(builder, nodeIn);
    if (doFixedArray) {
      if (builder.reusedBytesPerArc.length < nodeIn.numArcs) {
        builder.reusedBytesPerArc = new int[ArrayUtil.oversize(nodeIn.numArcs, 1)];
      }
    }
    
    builder.arcCount += nodeIn.numArcs;
    
    final int lastArc = nodeIn.numArcs-1;
    
    long lastArcStart = builder.bytes.getPosition();
    int maxBytesPerArc = 0;
    for(int arcIdx=0;arcIdx<nodeIn.numArcs;arcIdx++) {
      final Builder.Arc<T> arc = nodeIn.arcs[arcIdx];
      final Builder.CompiledNode target = (Builder.CompiledNode) arc.target;
      int flags = 0;
    
      if (arcIdx == lastArc) {
        flags += BIT_LAST_ARC;
      }
    
      if (builder.lastFrozenNode == target.node && !doFixedArray) {
        flags += BIT_TARGET_NEXT;
      }
    
      if (arc.isFinal) {
        flags += BIT_FINAL_ARC;
        if (arc.nextFinalOutput != NO_OUTPUT) {
          flags += BIT_ARC_HAS_FINAL_OUTPUT;
        }
      } else {
    
      }
    
      boolean targetHasArcs = target.node > 0;
    
      if (!targetHasArcs) {
        flags += BIT_STOP_NODE;
      } else if (inCounts != null) {
        inCounts.set((int) target.node, inCounts.get((int) target.node) + 1);
      }
    
      if (arc.output != NO_OUTPUT) {
        flags += BIT_ARC_HAS_OUTPUT;
      }
    
      builder.bytes.writeByte((byte) flags);
      writeLabel(builder.bytes, arc.label);
    
      if (arc.output != NO_OUTPUT) {
        outputs.write(arc.output, builder.bytes);
      }
    
      if (arc.nextFinalOutput != NO_OUTPUT) {
        outputs.writeFinalOutput(arc.nextFinalOutput, builder.bytes);
      }
    
      if (targetHasArcs && (flags & BIT_TARGET_NEXT) == 0) {
        builder.bytes.writeVLong(target.node);
      }
    
    }
    
    final long thisNodeAddress = builder.bytes.getPosition()-1;
    builder.bytes.reverse(startAddress, thisNodeAddress);
    
    builder.nodeCount++;
    final long node;
    node = thisNodeAddress;
    
    return node;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
首先判断如果是最后的节点，直接返回。接下来累加numArcs至arcCount中，统计节点arc个数。addNode函数接下来计算并设置标志位flags，然后将flags和label写入bytes中，label就是Term中的某个字母或者byte。addNode函数最后返回bytes即BytesStore中的位置。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm->writeBlocks->PendingBlock::compileIndex->Builder::add->freezeTail->compileNode->NodeHash::addNode

  public long add(Builder<T> builder, Builder.UnCompiledNode<T> nodeIn) throws IOException {
    final long h = hash(nodeIn);
    long pos = h & mask;
    int c = 0;
    while(true) {
      final long v = table.get(pos);
      if (v == 0) {
        final long node = fst.addNode(builder, nodeIn);
        count++;
        table.set(pos, node);
        if (count > 2*table.size()/3) {
          rehash();
        }
        return node;
      } else if (nodesEqual(nodeIn, v)) {
        return v;
      }
      pos = (pos + (++c)) & mask;
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
dedupHash的add函数首先通过hash函数获得该node的hash值，遍历node内的每个arc，计算hash值。
该函数内部也是使用了FST的addNode函数添加节点，并在必要的时候通过rehash扩展hash数组。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm->writeBlocks->PendingBlock::compileIndex->Builder::add->UnCompiledNode::addArc

    public void addArc(int label, Node target) {
      if (numArcs == arcs.length) {
        final Arc<T>[] newArcs = ArrayUtil.grow(arcs, numArcs+1);
        for(int arcIdx=numArcs;arcIdx<newArcs.length;arcIdx++) {
          newArcs[arcIdx] = new Arc<>();
        }
        arcs = newArcs;
      }
      final Arc<T> arc = arcs[numArcs++];
      arc.label = label;
      arc.target = target;
      arc.output = arc.nextFinalOutput = owner.NO_OUTPUT;
      arc.isFinal = false;
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
addArc用来将一个Term里的字母或者byte添加到该节点UnCompiledNode的arcs数组中，开头的if语句用来扩充arcs数组，然后按照顺序获取arcs数组中的Arc，并存入label，传入的参数target指向下一个UnCompiledNode节点。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm->writeBlocks->PendingBlock::compileIndex->Builder::finish

  public FST<T> finish() throws IOException {

    final UnCompiledNode<T> root = frontier[0];
    
    freezeTail(0);
    
    if (root.inputCount < minSuffixCount1 || root.inputCount < minSuffixCount2 || root.numArcs == 0) {
      if (fst.emptyOutput == null) {
        return null;
      } else if (minSuffixCount1 > 0 || minSuffixCount2 > 0) {
        return null;
      }
    } else {
      if (minSuffixCount2 != 0) {
        compileAllTargets(root, lastInput.length());
      }
    }
    fst.finish(compileNode(root, lastInput.length()).node);
    
    if (doPackFST) {
      return fst.pack(this, 3, Math.max(10, (int) (getNodeCount()/4)), acceptableOverheadRatio);
    } else {
      return fst;
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
finish函数开头的freezeTail函数传入的参数0，代表要处理frontier数组维护的所有节点，compileNode函数最后向bytes中写入根节点。最后的finish函数将FST的信息缓存到成员变量blocks中去，blocks是一个byte数组列表。

BlockTreeTermsWriter::write->TermsWriter::write->pushTerm->writeBlocks->PendingBlock::compileIndex->Builder::finish->FST::finish

  void finish(long newStartNode) throws IOException {
    startNode = newStartNode;
    bytes.finish();
    cacheRootArcs();
  }

  public void finish() {
    if (current != null) {
      byte[] lastBuffer = new byte[nextWrite];
      System.arraycopy(current, 0, lastBuffer, 0, nextWrite);
      blocks.set(blocks.size()-1, lastBuffer);
      current = null;
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
回到BlockTreeTermsWriter的write函数中，接下来通过TermsWriter的finish函数将FST中的信息写入.tip文件中。

BlockTreeTermsWriter::write->TermsWriter::write->finish

    public void finish() throws IOException {
      if (numTerms > 0) {
        pushTerm(new BytesRef());
        pushTerm(new BytesRef());
        writeBlocks(0, pending.size());
    
        final PendingBlock root = (PendingBlock) pending.get(0);
        indexStartFP = indexOut.getFilePointer();
        root.index.save(indexOut);
    
        BytesRef minTerm = new BytesRef(firstPendingTerm.termBytes);
        BytesRef maxTerm = new BytesRef(lastPendingTerm.termBytes);
    
        fields.add(new FieldMetaData(fieldInfo,
                                     ((PendingBlock) pending.get(0)).index.getEmptyOutput(),
                                     numTerms,
                                     indexStartFP,
                                     sumTotalTermFreq,
                                     sumDocFreq,
                                     docsSeen.cardinality(),
                                     longsSize,
                                     minTerm, maxTerm));
      } else {
    
      }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
root.index.save(indexOut)就是将信息写入.tip文件中。

总结
总接一下本章的大体流程。
BlockTreeTermWrite的调用TermsWriter的write函数处理每个域中的每个Term，然后通过finish函数将信息写入.tip文件。
TermsWriter的write函数针对每个Term，调用pushTerm函数将Term的信息写入.tim文件和FST中，然后将每个Term添加到待处理列表pending中。
pushTerm函数通过计算选择适当的时候调用writeBlocks函数将pending中多个Term写成一个Block。
writeBlocks在pending列表中选择相应的Term或者子Block，然后调用writeBlock函数写入相应的信息，然后调用compileIndex函数建立索引，最后删除在pending列表中已被处理的Term或者Block。
writeBlock函数向各个文件.doc、.pos和.pay写入对应Term或者Block的信息。
compileIndex函数通过Builder的add函数添加节点（每个Term的每个字母或者byte）到frontier数组中，frontier数组维护了UnCompiledNode节点，构成一棵树，compileIndex内部通过freezeTail函数将树中不会变动的节点通过compileNode函数写入FST结构中。
BlockTreeTermWrite最后在finish函数中将FST中的信息写入.tip文件中。
————————————————
版权声明：本文为CSDN博主「二侠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/conansonic/article/details/52133356

lucene源码分析—倒排索引的读过程
上一章中分析了lucene倒排索引的写过程，本章开始分析其读过程，重点分析SegmentTermsEnum的seekExact函数。
首先看几个构造函数，先看SegmentCoreReaders的构造函数，在Lucene50PostingFormat的fieldsProducer函数中创建。

BlockTreeTermsReader::BlockTreeTermsReader

  public BlockTreeTermsReader(PostingsReaderBase postingsReader, SegmentReadState state) throws IOException {
    boolean success = false;
    IndexInput indexIn = null;

    this.postingsReader = postingsReader;
    this.segment = state.segmentInfo.name;
    
    String termsName = IndexFileNames.segmentFileName(segment, state.segmentSuffix, TERMS_EXTENSION);
    try {
      termsIn = state.directory.openInput(termsName, state.context);
      version = CodecUtil.checkIndexHeader(termsIn, TERMS_CODEC_NAME, VERSION_START, VERSION_CURRENT, state.segmentInfo.getId(), state.segmentSuffix);
      ...
      String indexName = IndexFileNames.segmentFileName(segment, state.segmentSuffix, TERMS_INDEX_EXTENSION);
      indexIn = state.directory.openInput(indexName, state.context);
      CodecUtil.checkIndexHeader(indexIn, TERMS_INDEX_CODEC_NAME, version, version, state.segmentInfo.getId(), state.segmentSuffix);
      CodecUtil.checksumEntireFile(indexIn);
    
      postingsReader.init(termsIn, state);
      CodecUtil.retrieveChecksum(termsIn);
    
      seekDir(termsIn, dirOffset);
      seekDir(indexIn, indexDirOffset);
    
      final int numFields = termsIn.readVInt();
    
      for (int i = 0; i < numFields; ++i) {
        final int field = termsIn.readVInt();
        final long numTerms = termsIn.readVLong();
        final int numBytes = termsIn.readVInt();
        final BytesRef rootCode = new BytesRef(new byte[numBytes]);
        termsIn.readBytes(rootCode.bytes, 0, numBytes);
        rootCode.length = numBytes;
        final FieldInfo fieldInfo = state.fieldInfos.fieldInfo(field);
        final long sumTotalTermFreq = fieldInfo.getIndexOptions() == IndexOptions.DOCS ? -1 : termsIn.readVLong();
        final long sumDocFreq = termsIn.readVLong();
        final int docCount = termsIn.readVInt();
        final int longsSize = termsIn.readVInt();
        BytesRef minTerm = readBytesRef(termsIn);
        BytesRef maxTerm = readBytesRef(termsIn);
        final long indexStartFP = indexIn.readVLong();
        FieldReader previous = fields.put(fieldInfo.name, new FieldReader(this, fieldInfo, numTerms, rootCode, sumTotalTermFreq, sumDocFreq, docCount, indexStartFP, longsSize, indexIn, minTerm, maxTerm));
      }
    
      indexIn.close();
      success = true;
    } finally {
    
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
BlockTreeTermsReader的核心功能是打开.tim和.tip文件并创建输出流，然后创建FiledReader用于读取数据。
函数中的segment为段名，例如”_0”，state.segmentSuffix假设返回Lucene50_0，TERMS_EXTENSION默认为tim，因此segmentFileName构造文件名_0_Lucene50_0.tim。
directory对于cfs文件，返回Lucene50CompoundReader。
openInput函数返回SingleBufferImpl或者MultiBufferImpl，下面假设为SingleBufferImpl，termsIn封装了_0_Lucene50_0.tim文件的输出流。
checkIndexHeader检查头信息，和写过程的writeIndexHeader函数对应。
和.tim文件的打开过程类似，BlockTreeTermsReader的构造函数接下来打开_0_Lucene50_0.tip文件，检查头信息，同样调用openInput返回的indexIn封装了_0_Lucene50_0.tip文件的输出流。
seekDir最终调用SingleBufferImpl的父类ByteBufferIndexInput的seek函数，改变DirectByteBufferR的position指针的位置，用于略过一些头信息。然后从tim文件中读取并设置域的相应信息。最后创建FieldReader并返回。

BlockTreeTermsReader::BlockTreeTermsReader->FieldReader::FieldReader

  FieldReader(BlockTreeTermsReader parent, FieldInfo fieldInfo, long numTerms, BytesRef rootCode, long sumTotalTermFreq, long sumDocFreq, int docCount, long indexStartFP, int longsSize, IndexInput indexIn, BytesRef minTerm, BytesRef maxTerm) throws IOException {
    this.fieldInfo = fieldInfo;
    this.parent = parent;
    this.numTerms = numTerms;
    this.sumTotalTermFreq = sumTotalTermFreq;
    this.sumDocFreq = sumDocFreq;
    this.docCount = docCount;
    this.indexStartFP = indexStartFP;
    this.rootCode = rootCode;
    this.longsSize = longsSize;
    this.minTerm = minTerm;
    this.maxTerm = maxTerm;

    rootBlockFP = (new ByteArrayDataInput(rootCode.bytes, rootCode.offset, rootCode.length)).readVLong() >>> BlockTreeTermsReader.OUTPUT_FLAGS_NUM_BITS;
    
    if (indexIn != null) {
      final IndexInput clone = indexIn.clone();
      clone.seek(indexStartFP);
      index = new FST<>(clone, ByteSequenceOutputs.getSingleton());
    
    } else {
      index = null;
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
FieldReader函数的核心部分是创建一个FST，FST，全称Finite State Transducer，用有限状态机实现对词典中单词前缀和后缀的重复利用，压缩存储空间，在上一章已经介绍了如何将FST中的信息写入.tip文件，这一章后面介绍的过程相反，要将.tip文件中的数据读取出来。
rootBlockFP被创建为ByteArrayDataInput，ByteArrayDataInput对应的每个存储结构的最高位bit用来表示是否后面的位置信息有用。例如10000001（高位1表示后面的数据和前面的数据组成一个数据）+00000001最终其实为10000001。
seek函数调整ByteBufferIndexInput中当前ByteBuffer中的position位置为indexStartFP。
最后创建FST赋值给成员变量index。

BlockTreeTermsReader::BlockTreeTermsReader->FieldReader::FieldReader->FST::FST

  public FST(DataInput in, Outputs<T> outputs) throws IOException {
    this(in, outputs, DEFAULT_MAX_BLOCK_BITS);
  }

  public FST(DataInput in, Outputs<T> outputs, int maxBlockBits) throws IOException {
    this.outputs = outputs;

    version = CodecUtil.checkHeader(in, FILE_FORMAT_NAME, VERSION_PACKED, VERSION_NO_NODE_ARC_COUNTS);
    packed = in.readByte() == 1;
    if (in.readByte() == 1) {
      BytesStore emptyBytes = new BytesStore(10);
      int numBytes = in.readVInt();
      emptyBytes.copyBytes(in, numBytes);
    
      BytesReader reader;
      if (packed) {
        reader = emptyBytes.getForwardReader();
      } else {
        reader = emptyBytes.getReverseReader();
        if (numBytes > 0) {
          reader.setPosition(numBytes-1);
        }
      }
      emptyOutput = outputs.readFinalOutput(reader);
    } else {
      emptyOutput = null;
    }
    final byte t = in.readByte();
    switch(t) {
      case 0:
        inputType = INPUT_TYPE.BYTE1;
        break;
      case 1:
        inputType = INPUT_TYPE.BYTE2;
        break;
      case 2:
        inputType = INPUT_TYPE.BYTE4;
        break;
    default:
      throw new IllegalStateException("invalid input type " + t);
    }
    if (packed) {
      nodeRefToAddress = PackedInts.getReader(in);
    } else {
      nodeRefToAddress = null;
    }
    startNode = in.readVLong();
    if (version < VERSION_NO_NODE_ARC_COUNTS) {
      in.readVLong();
      in.readVLong();
      in.readVLong();
    }
    
    long numBytes = in.readVLong();
    if (numBytes > 1 << maxBlockBits) {
      bytes = new BytesStore(in, numBytes, 1<<maxBlockBits);
      bytesArray = null;
    } else {
      bytes = null;
      bytesArray = new byte[(int) numBytes];
      in.readBytes(bytesArray, 0, bytesArray.length);
    }
    
    cacheRootArcs();
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
FST的构造函数简而言之就是从.tip文件中读取写入的各个索引，并进行初始化。

传入的参数DEFAULT_MAX_BLOCK_BITS表示读取文件时每个块的大小，默认为30个bit。
checkHeader检查.tip文件的合法性。getForwardReader和getReverseReader返回FST.BytesReader。getForwardReader返回的BytesReader从缓存中向前读取数据，getReverseReader向后读取数据。
读取数据类型至inputType，即一个Term中的每个元素占多少字节。
最后读取了.tip文件最核心的内容并存储至bytesArray中，即倒排索引写过程中写入树的每个节点的信息。
cacheRootArcs函数对bytesArray中的数据进行解析并缓存根节点。

BlockTreeTermsReader::BlockTreeTermsReader->FieldReader::FieldReader->FST::FST->cacheRootArcs

  private void cacheRootArcs() throws IOException {
    final Arc<T> arc = new Arc<>();
    getFirstArc(arc);
    if (targetHasArcs(arc)) {
      final BytesReader in = getBytesReader();
      Arc<T>[] arcs = (Arc<T>[]) new Arc[0x80];
      readFirstRealTargetArc(arc.target, arc, in);
      int count = 0;
      while(true) {
        if (arc.label < arcs.length) {
          arcs[arc.label] = new Arc<T>().copyFrom(arc);
        } else {
          break;
        }
        if (arc.isLast()) {
          break;
        }
        readNextRealArc(arc, in);
        count++;
      }

      int cacheRAM = (int) ramBytesUsed(arcs);
    
      if (count >= FIXED_ARRAY_NUM_ARCS_SHALLOW && cacheRAM < ramBytesUsed()/5) {
        cachedRootArcs = arcs;
        cachedArcsBytesUsed = cacheRAM;
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
cacheRootArcs函数首先创建Arc，并调用getFirstArc对第一个节点进行初始化。targetHasArcs函数判断是否有可读信息，即在.tip文件中，一个节点是否有下一个节点。接着调用readFirstRealTargetArc读取第一个节点也即根节点的信息，这里就不往下看了，其中最重要的是读取该节点的内容label和下一个节点在bytesArray缓存中的位置。
再往下看cacheRootArcs函数，接下来通过一个while循环读取其他的根节点，如果读取的内容label大于128或者已经读取到最后的一个叶子节点，就退出循环，否则将读取到的节点信息存入arcs中，最后根据条件缓存到cachedRootArcs和cachedArcsBytesUsed成员变量里。

BlockTreeTermsReader::BlockTreeTermsReader->FieldReader::FieldReader->FST::FST->cacheRootArcs->getFirstArc

  public Arc<T> getFirstArc(Arc<T> arc) {
    T NO_OUTPUT = outputs.getNoOutput();

    if (emptyOutput != null) {
      arc.flags = BIT_FINAL_ARC | BIT_LAST_ARC;
      arc.nextFinalOutput = emptyOutput;
      if (emptyOutput != NO_OUTPUT) {
        arc.flags |= BIT_ARC_HAS_FINAL_OUTPUT;
      }
    } else {
      arc.flags = BIT_LAST_ARC;
      arc.nextFinalOutput = NO_OUTPUT;
    }
    arc.output = NO_OUTPUT;
    
    arc.target = startNode;
    return arc;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
getFirstArc函数用来初始化第一个节点，最重要的是设置了最后的arc.target，标识了一会从.tip核心内容的缓存bytesArray的哪个位置开始读。

下面开始分析SegmentTermsEnum的seekExact函数，先看一下SegmentTermsEnum的构造函数。

SegmentTermsEnum::SegmentTermsEnum

  public SegmentTermsEnum(FieldReader fr) throws IOException {
    this.fr = fr;

    stack = new SegmentTermsEnumFrame[0];
    staticFrame = new SegmentTermsEnumFrame(this, -1);
    
    if (fr.index == null) {
      fstReader = null;
    } else {
      fstReader = fr.index.getBytesReader();
    }
    
    for(int arcIdx=0;arcIdx<arcs.length;arcIdx++) {
      arcs[arcIdx] = new FST.Arc<>();
    }
    
    currentFrame = staticFrame;
    validIndexPrefix = 0;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
根据前面的分析，FieldReader的成员变量index是前面构造的FST，其构造函数读取了.tip文件，缓存了其核心内容到bytesArray中，并标记了起始位置为startNode。如果该index不为null，接下来的getBytesReader返回的就是bytesArray。

SegmentTermsEnum::seekExact
第一部分

  public boolean seekExact(BytesRef target) throws IOException {

    term.grow(1 + target.length);
    
    FST.Arc<BytesRef> arc;
    int targetUpto;
    BytesRef output;
    
    targetBeforeCurrentLength = currentFrame.ord;
    
    if (currentFrame != staticFrame) {
    
      ...
    
    } else {
    
      targetBeforeCurrentLength = -1;
      arc = fr.index.getFirstArc(arcs[0]);
      output = arc.output;
      currentFrame = staticFrame;
    
      targetUpto = 0;
      currentFrame = pushFrame(arc, BlockTreeTermsReader.FST_OUTPUTS.add(output, arc.nextFinalOutput), 0);
    }
    
    ...

  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
这里的fr是前面创建的FieldReader，index是FST，内部分装了从.tip文件读取的信息，FST_OUTPUTS是ByteSequenceOutputs。ByteSequenceOutputs的add函数合并arc.output和arc.nextFinalOutput两个BytesRef。
currentFrame和staticFrame不相等的情况不是第一次调用seekExact，if里省略的代码会利用之前的查找结果，本章不分析这种情况。
如果currentFrame和staticFrame相等，就调用getFirstArc初始化第一个Arc，最后pushFrame获得对应位置上（这里是第一个）的SegmentTermsEnumFrame并进行相应的设置。一个SegmentTermsEnumFrame代表的是一层节点，并不是一个节点，一层节点表示树中大于1个以上叶子节点到下一个该种节点间的所有节点。

SegmentTermsEnum::seekExact->SegmentTermsEnumFrame::pushFrame

  SegmentTermsEnumFrame pushFrame(FST.Arc<BytesRef> arc, BytesRef frameData, int length) throws IOException {
    scratchReader.reset(frameData.bytes, frameData.offset, frameData.length);
    final long code = scratchReader.readVLong();
    final long fpSeek = code >>> BlockTreeTermsReader.OUTPUT_FLAGS_NUM_BITS;
    final SegmentTermsEnumFrame f = getFrame(1+currentFrame.ord);
    f.hasTerms = (code & BlockTreeTermsReader.OUTPUT_FLAG_HAS_TERMS) != 0;
    f.hasTermsOrig = f.hasTerms;
    f.isFloor = (code & BlockTreeTermsReader.OUTPUT_FLAG_IS_FLOOR) != 0;
    if (f.isFloor) {
      f.setFloorData(scratchReader, frameData);
    }
    pushFrame(arc, fpSeek, length);

    return f;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
frameData保存了从.tip文件中读取的该节点对应的下一层节点的所有信息，即Arc结构中的nextFinalOutput。getFrame函数从SegmentTermsEnumFrame数组stack中获取对应位置上的SegmentTermsEnumFrame结构，然后调用pushFrame对其设置记录信息。

继续看seekExact函数的后一部分。

SegmentTermsEnum::seekExact
第二部分

  public boolean seekExact(BytesRef target) throws IOException {

    ...
    
    while (targetUpto < target.length) {
    
      final int targetLabel = target.bytes[target.offset + targetUpto] & 0xFF;
    
      final FST.Arc<BytesRef> nextArc = fr.index.findTargetArc(targetLabel, arc, getArc(1+targetUpto), fstReader);
    
      if (nextArc == null) {
    
        validIndexPrefix = currentFrame.prefix;
    
        currentFrame.scanToFloorFrame(target);
    
        if (!currentFrame.hasTerms) {
          termExists = false;
          term.setByteAt(targetUpto, (byte) targetLabel);
          term.setLength(1+targetUpto);
          return false;
        }
    
        currentFrame.loadBlock();
    
        final SeekStatus result = currentFrame.scanToTerm(target, true);            
        if (result == SeekStatus.FOUND) {
          return true;
        } else {
          return false;
        }
      } else {
        arc = nextArc;
        term.setByteAt(targetUpto, (byte) targetLabel);
        if (arc.output != BlockTreeTermsReader.NO_OUTPUT) {
          output = BlockTreeTermsReader.FST_OUTPUTS.add(output, arc.output);
        }
    
        targetUpto++;
    
        if (arc.isFinal()) {
          currentFrame = pushFrame(arc, BlockTreeTermsReader.FST_OUTPUTS.add(output, arc.nextFinalOutput), targetUpto);
        }
      }
    }
    
    validIndexPrefix = currentFrame.prefix;
    currentFrame.scanToFloorFrame(target);
    if (!currentFrame.hasTerms) {
      termExists = false;
      term.setLength(targetUpto);
      return false;
    }
    
    currentFrame.loadBlock();
    
    final SeekStatus result = currentFrame.scanToTerm(target, true);            
    if (result == SeekStatus.FOUND) {
      return true;
    } else {
      return false;
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
getArc函数每次在SegmentTermsEnum的成员变量Arc数组arcs中分配一个Arc结构，用于存放下一个节点信息，例如查询“abc”，如果当前查找“a”，有可能下一个节点即为“b”。
findTargetArc查找byte对应节点。
if部分表示找到了最后一层节点，或者没找到节点，scanToFloorFrame函数首先从.tip文件读取的结果中获取.tim文件中的指针。如果currentFrame.hasTerms为false，则表示没有找到Term，此时就直接返回了。如果找到了，则首先通过loadBlock函数从.tim文件中读取余下的信息，再调用scanToTerm进行比较，返回最终的结果。
这个举个例子，假设lucene索引中存储了“aab”、“aac”两个Term，在调用loadBlock前，已经找到了“aa”在.tip文件中信息，loadBlock函数就是根据“aa”在.tip文件中提供的指针位置，在.tim文件中获取到了b、c。
else部分表示找到了节点，则将查找到的label缓存到term中，如果到达了该层的最后一个节点，就调用pushFrame函数创建一个SegmentTermsEnumFrame记录下一层节点的信息。

SegmentTermsEnum::seekExact->getArc->findTargetArc

  public Arc<T> findTargetArc(int labelToMatch, Arc<T> follow, Arc<T> arc, BytesReader in) throws IOException {
    return findTargetArc(labelToMatch, follow, arc, in, true);
  }

  private Arc<T> findTargetArc(int labelToMatch, Arc<T> follow, Arc<T> arc, BytesReader in, boolean useRootArcCache) throws IOException {

    ...
    
    in.setPosition(getNodeAddress(follow.target));
    arc.node = follow.target;
    
    ...
    
    readFirstRealTargetArc(follow.target, arc, in);
    
    while(true) {
      if (arc.label == labelToMatch) {
        return arc;
      } else if (arc.label > labelToMatch) {
        return null;
      } else if (arc.isLast()) {
        return null;
      } else {
        readNextRealArc(arc, in);
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
省略的部分代码处理两种情况，一种情况是要查询的byte是个结束字符-1，另一种是直接从缓存cachedRootArcs查找。
第二部分省略的代码是当节点数量相同时采用二分法查找。
剩下的代码就是线性搜索了，传入的参数in就是对应.tip文件核心内容的缓存，即前面读取到的bytesArray，follow的target变量存储了第一个节点在.tip文件缓存中的指针位置，调用setPosition调整指针位置。
如果是线性搜索，则首先调用readFirstRealTargetArc读取根节点信息到arc，读取的信息最重要的一是根节点的label，二是根节点的下一个节点。如果匹配到要查找的labelToMatch就直接返回该节点，否则继续读取下一个节点直到匹配到或返回。

进入seekExact函数的if部分，scanToFloorFrame根据.tip文件中的信息获取最后的叶子节点在.tim文件中的指针，loadBlock则从.tim文件中读取最后的信息。

SegmentTermsEnum::seekExact->SegmentTermsEnumFrame::loadBlock

  void loadBlock() throws IOException {

    ste.initIndexInput();
    
    ste.in.seek(fp);
    int code = ste.in.readVInt();
    entCount = code >>> 1;
    isLastInFloor = (code & 1) != 0;
    
    code = ste.in.readVInt();
    isLeafBlock = (code & 1) != 0;
    int numBytes = code >>> 1;
    if (suffixBytes.length < numBytes) {
      suffixBytes = new byte[ArrayUtil.oversize(numBytes, 1)];
    }
    ste.in.readBytes(suffixBytes, 0, numBytes);
    suffixesReader.reset(suffixBytes, 0, numBytes);
    
    numBytes = ste.in.readVInt();
    if (statBytes.length < numBytes) {
      statBytes = new byte[ArrayUtil.oversize(numBytes, 1)];
    }
    ste.in.readBytes(statBytes, 0, numBytes);
    statsReader.reset(statBytes, 0, numBytes);
    metaDataUpto = 0;
    
    state.termBlockOrd = 0;
    nextEnt = 0;
    lastSubFP = -1;
    
    numBytes = ste.in.readVInt();
    if (bytes.length < numBytes) {
      bytes = new byte[ArrayUtil.oversize(numBytes, 1)];
    }
    ste.in.readBytes(bytes, 0, numBytes);
    bytesReader.reset(bytes, 0, numBytes);
    
    fpEnd = ste.in.getFilePointer();

  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
ste为SegmentTermsEnum。initIndexInput函数设置SegmentTermsEnum的成员变量in为BlockTreeTermsReader中的termsIn，对应.tim文件的输出流。fp为文件中的指针位置，在.tip文件中读取出来的。seek调整termsIn的读取位置。然后从tim文件读取数据到suffixBytes中，在封装到suffixBytes中。读取数据到statBytes中，封装到statsReader中。读取数据到bytes中，封装到bytesReader中。其中suffixBytes中封装的是待比较的数据。

SegmentTermsEnum::seekExact->SegmentTermsEnumFrame::scanToTerm

  public SeekStatus scanToTerm(BytesRef target, boolean exactOnly) throws IOException {
    return isLeafBlock ? scanToTermLeaf(target, exactOnly) : scanToTermNonLeaf(target, exactOnly);
  }

  public SeekStatus scanToTermLeaf(BytesRef target, boolean exactOnly) throws IOException {

    ste.termExists = true;
    subCode = 0;
    
    if (nextEnt == entCount) {
      if (exactOnly) {
        fillTerm();
      }
      return SeekStatus.END;
    }


    nextTerm: while (true) {
      nextEnt++;
    
      suffix = suffixesReader.readVInt();
    
      final int termLen = prefix + suffix;
      startBytePos = suffixesReader.getPosition();
      suffixesReader.skipBytes(suffix);
    
      final int targetLimit = target.offset + (target.length < termLen ? target.length : termLen);
      int targetPos = target.offset + prefix;
    
      int bytePos = startBytePos;
      while(true) {
        final int cmp;
        final boolean stop;
        if (targetPos < targetLimit) {
          cmp = (suffixBytes[bytePos++]&0xFF) - (target.bytes[targetPos++]&0xFF);
          stop = false;
        } else {
          cmp = termLen - target.length;
          stop = true;
        }
    
        if (cmp < 0) {
          if (nextEnt == entCount) {
            break nextTerm;
          } else {
            continue nextTerm;
          }
        } else if (cmp > 0) {
          fillTerm();
          return SeekStatus.NOT_FOUND;
        } else if (stop) {
          fillTerm();
          return SeekStatus.FOUND;
        }
      }
    }
    
    if (exactOnly) {
      fillTerm();
    }
    
    return SeekStatus.END;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
suffixesReader中有多个可以匹配的term，外层的while循环依次取出每个term，其中prefix是已经匹配的长度，不需要再匹配的，因为该长度已经对应到一个block中了（block下面包含多个suffix）。suffix保存了term的长度，startBytePos保存了该term在suffixesReader也即在suffixBytes中的偏移。内层的while循环依次比对每个字节，直到每个字节都相等，targetPos会等于targetLimit，stop被设为true。其他情况下，例如遍历了所有suffix都没找到，或者cmp大于0（suffix中的字节按顺序排序），意味着该block中找不到匹配的term，则也返回。
最后如果找到了，就返回SeekStatus.FOUND。

总结
下面通过一个例子总结一下lucene倒排索引的读过程。
假设索引文件中存储了“abc”“abd”两个字符串，待查找的字符串为“abc”，首先从.tip文件中按层次按节点查找“a”节点、再查找“b”节点（findTargetArc函数），获得“b”节点后继续从.tip文件中读取剩下的部分在.tim文件中的指针（scanToFloorFrame函数），然后从.tim文件中读取了“c”和“d”（loadBlock函数），最后比较获得最终结果（scanToTerm函数）。
————————————————
版权声明：本文为CSDN博主「二侠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/conansonic/article/details/52091301

lucene源码分析—BooleanQuery的评分过程
前面的章节分析过BooleanQuery的查询过，评分的过程只是简单介绍了下，本章回头再看一下BooleanQuery的评分过程，从其score函数开始。

BooleanScorer::score

  public int score(LeafCollector collector, Bits acceptDocs, int min, int max) throws IOException {

    ...
    
    BulkScorerAndDoc top = advance(min);
    while (top.next < max) {
      top = scoreWindow(top, collector, singleClauseCollector, acceptDocs, min, max);
    }
    
    return top.next;
  }
1
2
3
4
5
6
7
8
9
10
11
advance函数首先获得第一个匹配文档对应的BulkScorerAndDoc结构，其next成员变量就是文档号，然后通过scoreWindow函数循环处理匹配到的文档，scoreWindow函数默认一次处理最多2048个文档。

BooleanScorer::score->advance

  private BulkScorerAndDoc advance(int min) throws IOException {
    final HeadPriorityQueue head = this.head;
    final TailPriorityQueue tail = this.tail;
    BulkScorerAndDoc headTop = head.top();
    BulkScorerAndDoc tailTop = tail.top();
    while (headTop.next < min) {

        ...
    
        headTop.advance(min);
        headTop = head.updateTop();
    
        ...
    
    }
    return headTop;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
head为HeadPriorityQueue，对应的top函数返回BulkScorerAndDoc，updateTop函数将文档数量小的BulkScorerAndDoc排在前面并返回。

BooleanScorer::score->advance->BulkScorerAndDoc::advance

    void advance(int min) throws IOException {
      score(orCollector, null, min, min);
    }
    
    void score(LeafCollector collector, Bits acceptDocs, int min, int max) throws IOException {
      next = scorer.score(collector, acceptDocs, min, max);
    }
    
    public int score(LeafCollector collector, Bits acceptDocs, int min, int max) throws IOException {
      collector.setScorer(scorer);
      if (scorer.docID() == -1 && min == 0 && max == DocIdSetIterator.NO_MORE_DOCS) {
        ...
      } else {
        int doc = scorer.docID();
        if (doc < min) {
            doc = iterator.advance(min);
        }
        return scoreRange(collector, iterator, twoPhase, acceptDocs, doc, max);
      }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
如果是第一次获得文档ID，则docID函数返回-1，min为0，因此此时会调用iterator的advance函数获得文档ID，iterator的类型为BlockDocsEnum，其advance函数从对应的.doc文件中读取文档信息。

BooleanScorer::score->advance->BulkScorerAndDoc::advance->score->DefaultBulkScorer::score->BlockDocsEnum::advance

    public int advance(int target) throws IOException {
    
      if (docFreq > BLOCK_SIZE && target > nextSkipDoc) {
        ...
      }
    
      if (docUpto == docFreq) {
        return doc = NO_MORE_DOCS;
      }
    
      if (docBufferUpto == BLOCK_SIZE) {
        refillDocs();
      }
    
      while (true) {
        accum += docDeltaBuffer[docBufferUpto];
        docUpto++;
    
        if (accum >= target) {
          break;
        }
        docBufferUpto++;
        if (docUpto == docFreq) {
          return doc = NO_MORE_DOCS;
        }
      }
    
      freq = freqBuffer[docBufferUpto];
      docBufferUpto++;
      return doc = accum;
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
docUpto表示处理的文档指针，docBufferUpto是当前处理的文档指针，BLOCK_SIZE表示缓存大小，如果缓存已满，则调用refillDocs从.doc文件中读取数据到缓存。docDeltaBuffer和freqBuffer缓存分别存储了文档ID和词频，存储方式为差值存储，最后返回需要的文档ID。

获得第一个文档ID后，BooleanScorer的score函数接下来通过scoreWindow函数处理匹配到的文档。

BooleanScorer::score->scoreWindow

  private BulkScorerAndDoc scoreWindow(BulkScorerAndDoc top, LeafCollector collector,
      LeafCollector singleClauseCollector, Bits acceptDocs, int min, int max) throws IOException {
    final int windowBase = top.next & ~MASK;
    final int windowMin = Math.max(min, windowBase);
    final int windowMax = Math.min(max, windowBase + SIZE);

    leads[0] = head.pop();
    int maxFreq = 1;
    while (head.size() > 0 && head.top().next < windowMax) {
      leads[maxFreq++] = head.pop();
    }
    
    if (minShouldMatch == 1 && maxFreq == 1) {
    
      ...
    
    } else {
      scoreWindowMultipleScorers(collector, acceptDocs, windowBase, windowMin, windowMax, maxFreq);
      return head.top();
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
scoreWindow函数一次只处理最多SIZE大小的文档，windowMin和windowMax分别表示当前处理的文档号的最小值和最大值。接下来获得对应的BulkScorerAndDoc保存在leads数组中，最后调用scoreWindowMultipleScorers函数继续处理。

BooleanScorer::score->scoreWindow->scoreWindowMultipleScorers

  private void scoreWindowMultipleScorers(LeafCollector collector, Bits acceptDocs, int windowBase, int windowMin, int windowMax, int maxFreq) throws IOException {

    ...
    
    if (maxFreq >= minShouldMatch) {
    
      ...
    
      scoreWindowIntoBitSetAndReplay(collector, acceptDocs, windowBase, windowMin, windowMax, leads, maxFreq);
    }
    
    ...
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
scoreWindowMultipleScorers函数会继续调用scoreWindowIntoBitSetAndReplay进行处理。

BooleanScorer::score->scoreWindow->scoreWindowMultipleScorers->scoreWindowIntoBitSetAndReplay

  private void scoreWindowIntoBitSetAndReplay(LeafCollector collector, Bits acceptDocs,
      int base, int min, int max, BulkScorerAndDoc[] scorers, int numScorers) throws IOException {
    for (int i = 0; i < numScorers; ++i) {
      final BulkScorerAndDoc scorer = scorers[i];
      scorer.score(orCollector, acceptDocs, min, max);
    }

    scoreMatches(collector, base);
    Arrays.fill(matching, 0L);
  }
1
2
3
4
5
6
7
8
9
10
scoreWindowIntoBitSetAndReplay函数遍历当前的BulkScorerAndDoc数组，调用其score函数计算评分。BulkScorerAndDoc的score函数最终会调用到OrCollector的collect函数。scoreMatches对本次的处理结果进行最终处理。最终清空matching数组，以便后续2048个文档的分析。

BooleanScorer::score->scoreWindow->scoreWindowMultipleScorers->scoreWindowIntoBitSetAndReplay->BulkScorerAndDoc::score->DefaultBulkScorer::score->scoreRange->OrCollector::collect

    public void collect(int doc) throws IOException {
      final int i = doc & MASK;
      final int idx = i >>> 6;
      matching[idx] |= 1L << i;
      final Bucket bucket = buckets[i];
      bucket.freq++;
      bucket.score += scorer.score();
    }
1
2
3
4
5
6
7
8
collect函数一次最多处理2048个文档，成员变量matching用比特位记录匹配到了哪些文档，buckets存储当前处理的最多2048个文档的得分，分别调用score函数计算得到。其中，2048个文档被分成32个组，每组64个比特位记录哪些文档匹配。

BooleanScorer::score->scoreWindow->scoreWindowMultipleScorers->scoreWindowIntoBitSetAndReplay->scoreMatches

  private void scoreMatches(LeafCollector collector, int base) throws IOException {
    long matching[] = this.matching;
    for (int idx = 0; idx < matching.length; idx++) {
      long bits = matching[idx];
      while (bits != 0L) {
        int ntz = Long.numberOfTrailingZeros(bits);
        int doc = idx << 6 | ntz;
        scoreDocument(collector, base, doc);
        bits ^= 1L << ntz;
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
scoreMatches根据比特位查看匹配到的文档号是多少，然后调用scoreDocument函数计算最终得分并排序。

BooleanScorer::score->scoreWindow->scoreWindowMultipleScorers->scoreWindowIntoBitSetAndReplay->scoreMatches->scoreDocument

  private void scoreDocument(LeafCollector collector, int base, int i) throws IOException {
    final FakeScorer fakeScorer = this.fakeScorer;
    final Bucket bucket = buckets[i];
    if (bucket.freq >= minShouldMatch) {
      fakeScorer.freq = bucket.freq;
      fakeScorer.score = (float) bucket.score * coordFactors[bucket.freq];
      final int doc = base | i;
      fakeScorer.doc = doc;
      collector.collect(doc);
    }
    bucket.freq = 0;
    bucket.score = 0;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
这里计算文档号，并从buckets数组中取出前面的计算结果，然后调用collect函数处理最终结果。这里的collector是最先创建的SimpleTopScoreDocCollector，其collect函数就是比较分数，对最终要返回的文档进行排序。
————————————————
版权声明：本文为CSDN博主「二侠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/conansonic/article/details/52950534

lucene源码分析—PhraseQuery
本章开始查看PhraseQuery的源码，PhraseQuery顾名思义是短语查询，先来看PhraseQuery是如何构造的，

        PhraseQuery.Builder queryBuilder = new PhraseQuery.Builder();
        queryBuilder.add(new Term("body","hello"));
        queryBuilder.add(new Term("body","world"));
        queryBuilder.setSlop(100);
        PhraseQuery query = queryBuilder.build();
1
2
3
4
5
首先创建PhraseQuery.Builder，然后向其添加短语中的每个词Term，PhraseQuery要求Term的数量至少为2个，这很容易理解，例子中的“body”表示域名，“hello”和“world”代表词的内容。然后通过setSlop设置词的最大间距，最后通过build函数创建PhraseQuery 。

创建完PhraseQuery后，就调用IndexSearcher的search函数进行查询，最终会执行到如下代码，

  public void search(Query query, Collector results)
    throws IOException {
    search(leafContexts, createNormalizedWeight(query, results.needsScores()), results);
  }
1
2
3
4
createNormalizedWeight函数找出短语所在文档的整体信息，然后通过search函数对每个文档进行评分，筛选出最后的文档。

IndexSearcher::createNormalizedWeight

  public Weight createNormalizedWeight(Query query, boolean needsScores) throws IOException {
    query = rewrite(query);
    Weight weight = createWeight(query, needsScores);

    ...
    
    return weight;
  }
1
2
3
4
5
6
7
8
对PhraseQuery而言，rewrite函数什么也不做，即PhraseQuery不需要改写。createNormalizedWeight函数接下来通过createWeight函数创建Weight，封装短语的整体信息。

IndexSearcher::createNormalizedWeight->createWeight

  public Weight createWeight(Query query, boolean needsScores) throws IOException {

    ...
    
    Weight weight = query.createWeight(this, needsScores);
    
    ...
    
    return weight;
  }

  public Weight createWeight(IndexSearcher searcher, boolean needsScores) throws IOException {
    return new PhraseWeight(searcher, needsScores);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
createWeight函数最终会创建PhraseWeight。

IndexSearcher::createNormalizedWeight->createWeight->PhraseQuery::createWeight->PhraseWeight::PhraseWeight

  public PhraseWeight(IndexSearcher searcher, boolean needsScores)
    throws IOException {
    final int[] positions = PhraseQuery.this.getPositions();
    this.needsScores = needsScores;
    this.similarity = searcher.getSimilarity(needsScores);
    final IndexReaderContext context = searcher.getTopReaderContext();
    states = new TermContext[terms.length];
    TermStatistics termStats[] = new TermStatistics[terms.length];
    for (int i = 0; i < terms.length; i++) {
      final Term term = terms[i];
      states[i] = TermContext.build(context, term);
      termStats[i] = searcher.termStatistics(term, states[i]);
    }
    stats = similarity.computeWeight(searcher.collectionStatistics(field), termStats);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
getPositions函数返回查询的每个词在查询短语中的位置，例如查询“hello world”，则getPositions返回[0，1]。
getSimilarity默认返回BM25Similarity，用于评分，开发者可以定义自己的Similarity并设置进IndexerSearch中，这里就会返回自定义的Similarity。
getTopReaderContext默认返回CompositeReaderContext。再往下，遍历每个查询的词Term，通过TermContext的build函数获取每个词的整体信息，然后通过termStatistics函数将部分信息封装成TermStatistics。
最后通过Similarity的computeWeight函数计算所有词的整体信息保存在stats中。
这些函数在前面的章节已经分析过了，和PhraseQuery并无关系，这里就不往下看了。

回到一开始的search函数中，接下来继续通过search函数进行搜索。

IndexSearcher::search

  protected void search(List<LeafReaderContext> leaves, Weight weight, Collector collector)
      throws IOException {
    for (LeafReaderContext ctx : leaves) {
      final LeafCollector leafCollector = collector.getLeafCollector(ctx);
      BulkScorer scorer = weight.bulkScorer(ctx);
      scorer.score(leafCollector, ctx.reader().getLiveDocs());
    }
  }
1
2
3
4
5
6
7
8
getLeafCollector返回的LeafCollector用来打分并对评分后的文档进行排序。
bulkScorer返回的BulkScorer用来为多个文档打分。
最终通过BulkScorer的scorer函数进行打分并获得最终的结果。

IndexSearcher::search->PhraseWeight::bulkScorer

  public BulkScorer bulkScorer(LeafReaderContext context) throws IOException {
    Scorer scorer = scorer(context);
    return new DefaultBulkScorer(scorer);
  }
1
2
3
4
scorer函数最终会调用PhraseWeight的scorer函数获得Scorer，然后通过DefaultBulkScorer进行封装和处理。

IndexSearcher::search->PhraseWeight::bulkScorer->scorer

    public Scorer scorer(LeafReaderContext context) throws IOException {
      final LeafReader reader = context.reader();
      PostingsAndFreq[] postingsFreqs = new PostingsAndFreq[terms.length];
    
      final Terms fieldTerms = reader.terms(field);
      final TermsEnum te = fieldTerms.iterator();
      float totalMatchCost = 0;
    
      for (int i = 0; i < terms.length; i++) {
        final Term t = terms[i];
        final TermState state = states[i].get(context.ord);
        te.seekExact(t.bytes(), state);
        PostingsEnum postingsEnum = te.postings(null, PostingsEnum.POSITIONS);
        postingsFreqs[i] = new PostingsAndFreq(postingsEnum, positions[i], t);
        totalMatchCost += termPositionsCost(te);
      }
    
      return new SloppyPhraseScorer(this, postingsFreqs, slop,
                                        similarity.simScorer(stats, context),
                                        needsScores, totalMatchCost);
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
terms获得FieldReader，并通过其iterator函数创建SegmentTermsEnum，该过程在前面的章节有叙述。
接下来遍历待搜索的词Term，针对每个Term，通过SegmentTermsEnum的postings函数返回倒排列表对应的.doc、.pos和.pay文件的读取信息，并根据.tip文件中的索引数据设置这三个文件的输出流指针。封装成PostingsAndFreq。
假设slop>0，scorer函数最后创建SloppyPhraseScorer并返回，SloppyPhraseScorer用于计算短语查询的最终得分。

IndexSearcher::search->PhraseWeight::bulkScorer->DefaultBulkScorer

    public DefaultBulkScorer(Scorer scorer) {
      this.scorer = scorer;
      this.iterator = scorer.iterator();
      this.twoPhase = scorer.twoPhaseIterator();
    }
1
2
3
4
5
DefaultBulkScorer构造函数中的iterator和twoPhaseIterator函数都是创建迭代器，用于获取到下一个文档ID，还可以计算词位置登信息。

回到IndexSearcher函数中，接下来通过刚刚创建的DefaultBulkScorer的score函数计算最终的文档得分并排序，DefaultBulkScorer的score函数最终会调用到scoreAll函数。

IndexSearcher::search->DefaultBulkScorer::score->score->scoreAll

    static void scoreAll(LeafCollector collector, DocIdSetIterator iterator, TwoPhaseIterator twoPhase, Bits acceptDocs) throws IOException {
      final DocIdSetIterator approximation = twoPhase.approximation();
      for (int doc = approximation.nextDoc(); doc != DocIdSetIterator.NO_MORE_DOCS; doc = approximation.nextDoc()) {
        if ((acceptDocs == null || acceptDocs.get(doc)) && twoPhase.matches()) {
          collector.collect(doc);
        }
      }
    }
1
2
3
4
5
6
7
8
approximation是前面在DefaultBulkScorer的构造函数中创建的继承自DocIdSetIterator的ConjunctionDISI，通过其nextDoc函数获取下一个文档ID，然后调用TwoPhaseIterator的matches信息获取词的位置信息，最后通过LeafCollector的collect函数完成最终的打分并排序。

IndexSearcher::search->DefaultBulkScorer::score->score->scoreAll->ConjunctionDISI::nextDoc

  public int nextDoc() throws IOException {
    return doNext(lead.nextDoc());
  }
1
2
3
这里的变量lead实质上未BlockPostingsEnum，其nextDoc获得.doc文件中对应的倒排列表中的下一个文档ID。然后通过doNext函数查找下一个文档ID，对应的文档包含了查询短语中的所有词，如果不包含，则要继续查找下一个文档。

IndexSearcher::search->DefaultBulkScorer::score->score->scoreAll->ConjunctionDISI::nextDoc->doNext

  private int doNext(int doc) throws IOException {
    for(;;) {
      advanceHead: for(;;) {
        for (DocIdSetIterator other : others) {
          if (other.docID() < doc) {
            final int next = other.advance(doc);

            if (next > doc) {
              doc = lead.advance(next);
              break advanceHead;
            }
          }
        }
        return doc;
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
others是除了lead外的所有词对应的DocIdSetIterator列表，例如查找“hello world champion”，则假设此时lead为“hello”，others对应的DocIdSetIterator列表则分别包含“world”和“champion”，advance函数从other中找到一个等于或大于当前文档ID的文档，这里有一个默认条件，就是lead中的文档ID是最小的，因此当others中的所有文档ID不大于当前文档ID，就说明该文档ID包含了所有的词，返回当前文档ID。如果某个other的文档ID大于了当前的文档ID，则要重新计算当前文档ID，并继续循环直到找到为止。

获取到文档ID后，就要通过phraseFreq计算词的位置信息了。

IndexSearcher::search->DefaultBulkScorer::score->score->scoreAll->TwoPhaseIterator::matches->phraseFreq

  private float phraseFreq() throws IOException {
    if (!initPhrasePositions()) {
      return 0.0f;
    }
    float freq = 0.0f;
    numMatches = 0;
    PhrasePositions pp = pq.pop();
    int matchLength = end - pp.position;
    int next = pq.top().position; 
    while (advancePP(pp)) {
      if (pp.position > next) {
        if (matchLength <= slop) {
          freq += docScorer.computeSlopFactor(matchLength);
          numMatches++;
        }      
        pq.add(pp);
        pp = pq.pop();
        next = pq.top().position;
        matchLength = end - pp.position;
      } else {
        int matchLength2 = end - pp.position;
        if (matchLength2 < matchLength) {
          matchLength = matchLength2;
        }
      }
    }
    if (matchLength <= slop) {
      freq += docScorer.computeSlopFactor(matchLength);
      numMatches++;
    }    
    return freq;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
省略的部分代码用于处理重复的词，例如当查询“hello hello world”时，就会出现两个重复的“hello”。initPhrasePositions用来初始化。matchLength表示计算得到的两个词的距离，当matchLength小于slop时，调用computeSlopFactor计算得分并累加，公式为1/(distance+1)。
这里不逐行逐字地分析这段代码了，举个例子就明白了，假设一篇文档如下
“…hello(5)…world(10)…world(13)…hello(21)…”
查询短语为“hello world”。
则首先找到第一个hello和第一个world，计算得到matchLength为10-5=5，

IndexSearcher::search->DefaultBulkScorer::score->score->scoreAll->TwoPhaseIterator::matches->phraseFreq->initPhrasePositions

  private boolean initPhrasePositions() throws IOException {
    end = Integer.MIN_VALUE;
    if (!checkedRpts) {
      return initFirstTime();
    }
    if (!hasRpts) {
      initSimple();
      return true;
    }
    return initComplex();
  }
1
2
3
4
5
6
7
8
9
10
11
DefaultBulkScorer::score->score->scoreAll->TwoPhaseIterator::matches->phraseFreq->initPhrasePositions->initFirstTime

  private boolean initFirstTime() throws IOException {
    checkedRpts = true;
    placeFirstPositions();

    LinkedHashMap<Term,Integer> rptTerms = repeatingTerms(); 
    hasRpts = !rptTerms.isEmpty();
    
    if (hasRpts) {
      rptStack = new PhrasePositions[numPostings];
      ArrayList<ArrayList<PhrasePositions>> rgs = gatherRptGroups(rptTerms);
      sortRptGroups(rgs);
      if (!advanceRepeatGroups()) {
        return false;
      }
    }
    
    fillQueue();
    return true;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
checkedRpts设置为true，表示初始化过了。
placeFirstPositions函数初始化位置信息。
repeatingTerms函数获取重复的词。
如果存在重复的词，则此时hasRpts为true，则要进行相应的处理，本章不考虑这种情况。
最后调用fillQueue函数设置end成员变量，end表示当前最大的位置。

DefaultBulkScorer::score->score->scoreAll->TwoPhaseIterator::matches->phraseFreq->initPhrasePositions->initFirstTime->placeFirstPositions

  private void placeFirstPositions() throws IOException {
    for (PhrasePositions pp : phrasePositions) {
      pp.firstPosition();
    }
  }
1
2
3
4
5
phrasePositions对应每个Term的PhrasePositions，内部封装了倒排列表信息，firstPosition函数初始化位置信息，内部会读取.pos文件。

DefaultBulkScorer::score->score->scoreAll->TwoPhaseIterator::matches->phraseFreq->initPhrasePositions->initFirstTime->placeFirstPositions->PhrasePositions::firstPosition

  final void firstPosition() throws IOException {
    count = postings.freq();
    nextPosition();
  }

  final boolean nextPosition() throws IOException {
    if (count-- > 0) {
      position = postings.nextPosition() - offset;
      return true;
    } else
      return false;
  }
1
2
3
4
5
6
7
8
9
10
11
12
freq表示当前有多少可以读取的位置信息。
然后调用nextPosition获得下一个位置信息，进而调用BlockPostingsEnum的nextPosition函数获取下一个位置信息，offset是起始的位移，例如查询“abc def”，当获得abc所在文档的位置时，offset为0，当获得def所在文档的位置时，offset为1。

DefaultBulkScorer::score->score->scoreAll->TwoPhaseIterator::matches->phraseFreq->initPhrasePositions->initFirstTime->placeFirstPositions->PhrasePositions::firstPosition->nextPosition->BlockPostingsEnum::nextPosition

    public int nextPosition() throws IOException {
    
      if (posPendingFP != -1) {
        posIn.seek(posPendingFP);
        posPendingFP = -1;
        posBufferUpto = BLOCK_SIZE;
      }
    
      if (posPendingCount > freq) {
        skipPositions();
        posPendingCount = freq;
      }
    
      if (posBufferUpto == BLOCK_SIZE) {
        refillPositions();
        posBufferUpto = 0;
      }
      position += posDeltaBuffer[posBufferUpto++];
      posPendingCount--;
      return position;
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
posPendingFP是从.tip文件中读取得到的.pos文件的指针。
通过seek函数移动到读取位置posPendingFP，然后将posPendingFP指针重置。
设置posBufferUpto为BLOCK_SIZE后面会调用refillPositions将.pos文件中的信息读入缓存中。
posPendingCount变量表示待读取的词位置的个数，假设单词“abc”在文档1中的频率为2，在文档2中的频率为3，当读取完文档1中的频率2而没有处理，再读取文档2中的频率3，此时posPendingCount为2+3=5。
当posPendingCount大于当前文档的频率freq时，则调用skipPositions跳过之前读取的信息。
posBufferUpto等于BLOCK_SIZE表示缓存已经读到末尾，或者表示首次读取缓存，此时通过refillPositions函数将.pos文件中的信息读取到缓存中，并将缓存的指针posBufferUpto设置为0。
最后返回的position是从缓存posDeltaBuffer读取到的最终的位置，并将待读取的个数posPendingCount减1。

DefaultBulkScorer::score->score->scoreAll->TwoPhaseIterator::matches->phraseFreq->initPhrasePositions->initFirstTime->placeFirstPositions->PhrasePositions::firstPosition->nextPosition->BlockPostingsEnum::nextPosition->skipPositions

    private void skipPositions() throws IOException {
      int toSkip = posPendingCount - freq;
    
      final int leftInBlock = BLOCK_SIZE - posBufferUpto;
      if (toSkip < leftInBlock) {
        posBufferUpto += toSkip;
      } else {
        toSkip -= leftInBlock;
        while(toSkip >= BLOCK_SIZE) {
          forUtil.skipBlock(posIn);
          toSkip -= BLOCK_SIZE;
        }
        refillPositions();
        posBufferUpto = toSkip;
      }
    
      position = 0;
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
toSkip计算需要跳过多少。
leftInBlock表示当前缓存中的剩余空间。
如果toSkip小于leftInBlock则表示缓存中还有富余空间，此时直接更新缓存的指针posBufferUpto即可。
如果toSkip大于等于leftInBlock，则先减去当前缓存的剩余空间，然后判断剩余的toSkip是否大于缓存的最大空间BLOCK_SIZE，如果大于，就循环将文件指针向前移动BLOCK_SIZE，再将toSkip减去BLOCK_SIZE，知道toSkip小于BLOCK_SIZE，此时通过refillPositions函数再向缓存填充BLOCK_SIZE大小的数据，设置文件指针为toSkip。
因为位置信息存储的是差值，因此最后将位置position重置为0。

DefaultBulkScorer::score->score->scoreAll->TwoPhaseIterator::matches->phraseFreq->initPhrasePositions->initFirstTime->repeatingTerms

  private LinkedHashMap<Term,Integer> repeatingTerms() {
    LinkedHashMap<Term,Integer> tord = new LinkedHashMap<>();
    HashMap<Term,Integer> tcnt = new HashMap<>();
    for (PhrasePositions pp : phrasePositions) {
      for (Term t : pp.terms) {
        Integer cnt0 = tcnt.get(t);
        Integer cnt = cnt0==null ? new Integer(1) : new Integer(1+cnt0.intValue());
        tcnt.put(t, cnt);
        if (cnt==2) {
          tord.put(t,tord.size());
        }
      }
    }
    return tord;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
PhrasePositions的terms成员变量保存了该PhrasePositions的词，这里其实是遍历搜索的所有词，将重复的词放入最终返回的HashMap中。

DefaultBulkScorer::score->score->scoreAll->TwoPhaseIterator::matches->phraseFreq->initPhrasePositions->initFirstTime->fillQueue

  private void fillQueue() {
    pq.clear();
    for (PhrasePositions pp : phrasePositions) {
      if (pp.position > end) {
        end = pp.position;
      }
      pq.add(pp);
    }
  }
1
2
3
4
5
6
7
8
9
fillQueue依次读取每个PhrasePositions，将第一个Term在文档中出现位置的最大值保存在end中。
————————————————
版权声明：本文为CSDN博主「二侠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/conansonic/article/details/53138111

lucene源码分析—高亮
本章分析lucene的highlighter高亮部分的代码，例子的代码如下，

    Analyzer analyzer = new StandardAnalyzer();
    QueryScorer scorer = new QueryScorer(query);
    Highlighter highlight = new Highlighter(scorer);
    TokenStream tokenStream = analyzer.tokenStream("fieldName", new StringReader(fieldValue));
    String highlightStr = highlight.getBestFragment(tokenStream, fieldValue);
1
2
3
4
5
该例子首先创建相应的类，例如分词器StandardAnalyzer，用于匹配词并打分的QueryScorer，以及高亮的主体Highlighter，然后调用分词器的tokenStream函数进行初始化，最后调用Highlighter的getBestFragment函数获取片段。
这些类的构造函数都相对简单，本章就不往下看了，下面重点看一下Highlighter的getBestFragment函数。

Highlighter::getBestFragment

  public final String getBestFragment(TokenStream tokenStream, String text)
    throws IOException, InvalidTokenOffsetsException
  {
    String[] results = getBestFragments(tokenStream,text, 1);
    return results[0];
  }

  public final String[] getBestFragments(
    TokenStream tokenStream,
    String text,
    int maxNumFragments)
    throws IOException, InvalidTokenOffsetsException
  {
    maxNumFragments = Math.max(1, maxNumFragments);
    TextFragment[] frag =getBestTextFragments(tokenStream,text, true,maxNumFragments);
    ArrayList<String> fragTexts = new ArrayList<>();
    for (int i = 0; i < frag.length; i++)
    {
      if ((frag[i] != null) && (frag[i].getScore() > 0))
      {
        fragTexts.add(frag[i].toString());
      }
    }
    return fragTexts.toArray(new String[0]);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
getBestFragment函数进而调用getBestFragments函数，maxNumFragments代表返回的TextFragment的最大数量。getBestTextFragments返回处理得到的TextFragment数组，里面封装了高亮的文本片段。

Highlighter::getBestFragment->getBestFragments->getBestTextFragments

  public final TextFragment[] getBestTextFragments(
    TokenStream tokenStream,
    String text,
    boolean mergeContiguousFragments,
    int maxNumFragments)
    throws IOException, InvalidTokenOffsetsException
  {
    ArrayList<TextFragment> docFrags = new ArrayList<>();
    StringBuilder newText=new StringBuilder();

    CharTermAttribute termAtt = tokenStream.addAttribute(CharTermAttribute.class);
    OffsetAttribute offsetAtt = tokenStream.addAttribute(OffsetAttribute.class);
    TextFragment currentFrag =  new TextFragment(newText,newText.length(), docFrags.size());
    fragmentScorer.setMaxDocCharsToAnalyze(maxDocCharsToAnalyze);  
    
    TokenStream tokenStream = fragmentScorer.init(tokenStream);   
    fragmentScorer.startFragment(currentFrag);
    docFrags.add(currentFrag);
    FragmentQueue fragQueue = new FragmentQueue(maxNumFragments);
    
    String tokenText;
    int startOffset;
    int endOffset;
    int lastEndOffset = 0;
    textFragmenter.start(text, tokenStream);
    
    TokenGroup tokenGroup=new TokenGroup(tokenStream);
    tokenStream.reset();
      for (boolean next = tokenStream.incrementToken(); next && (offsetAtt.startOffset()< maxDocCharsToAnalyze); next = tokenStream.incrementToken())
      {
        if((tokenGroup.getNumTokens() >0)&&(tokenGroup.isDistinct()))
        {
          startOffset = tokenGroup.getStartOffset();
          endOffset = tokenGroup.getEndOffset();
          tokenText = text.substring(startOffset, endOffset);
          String markedUpText=formatter.highlightTerm(encoder.encodeText(tokenText), tokenGroup);
          if (startOffset > lastEndOffset)
            newText.append(encoder.encodeText(text.substring(lastEndOffset, startOffset)));
          newText.append(markedUpText);
          lastEndOffset=Math.max(endOffset, lastEndOffset);
          tokenGroup.clear();
          if(textFragmenter.isNewFragment())
          {
            currentFrag.setScore(fragmentScorer.getFragmentScore());
            currentFrag.textEndPos = newText.length();
            currentFrag =new TextFragment(newText, newText.length(), docFrags.size());
            fragmentScorer.startFragment(currentFrag);
            docFrags.add(currentFrag);
          }
        }
    
        tokenGroup.addToken(fragmentScorer.getTokenScore());
      }
    
      ...
    
      for (Iterator<TextFragment> i = docFrags.iterator(); i.hasNext();){
        currentFrag = i.next();
        fragQueue.insertWithOverflow(currentFrag);
      }
    
      TextFragment frag[] = new TextFragment[fragQueue.size()];
      for (int i = frag.length - 1; i >= 0; i--){
        frag[i] = fragQueue.pop();
      }
    
      ArrayList<TextFragment> fragTexts = new ArrayList<>();
      for (int i = 0; i < frag.length; i++){
        fragTexts.add(frag[i]);
      }
      return fragTexts.toArray(new TextFragment[0]);
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
首先进行初始化工作，获取TokenStream中的CharTermAttribute和OffsetAttribute，分别保存了每个词的byte数组和位置信息。
创建TextFragment，表示一个文档中的一个片段。
newText表示最终被修改的文档，例如在一些关键字上加上标签后的文档。
docFrags是一个TextFragment列表，内部保存了所有处理后的文档片段TextFragment。
fragmentScorer是前面创建的QueryScorer，其setMaxDocCharsToAnalyze函数设置该QueryScorer针对每篇文档能够处理的最大字节数。
接下来调用QueryScorer的init函数进行初始化，该初始化的主要任务是将待处理的文档分词，并保存查询的词。
然后调用fragmentScorer的startFragment函数对第一个待处理的片段进行初始化，并添加第一个片段currentFrag到docFrags列表中，注意这里只是保存了引用，currentFrag还未处理过。
创建FragmentQueue，也用于保存TextFragment，最后会将docFrags中所有处理完的TextFragment再次保存在FragmentQueue中，FragmentQueue继承自PriorityQueue，PriorityQueue会对添加进的元素进行自动排序。maxNumFragments表示该FragmentQueue可以保存的元素个数。
再往下的textFragmenter在Highlighter的构造函数中被初始化为SimpleFragmenter，其作用是用于判断是否需要重新构造一个TextFragment，默认一个TextFragment能够包含的文档字节数为100，即对于一篇文档而言，每处理100个字节，就创建一个新的TextFragment。其start函数获取当前文档处理的位置引用，并将处理的TextFragment总数设置为1。
接下来创建TokenGroup，TokenGroup保存了多组有互相重叠的词，例如“abc”被分词为“ab”和“bc”，则该两个Term的处理结果会保存在一个TokenGroup中。

初始化工作完成后，接下来就要对一个个片段进行处理了，首先通过一个for循环遍历文档的所有分词，TokenGroup的getNumTokens函数返回TokenGroup中已经处理的词的数量，isDistinct表示下一个词和TokenGroup中已经处理的词是否有位置上的重叠，如果重叠则返回true，因此进入if语句，表示要对之前处理的词进行处理。
进入if语句后，首先通过TokenGroup的getStartOffset和getEndOffset函数获取该TokenGroup处理的所有词的起始位置和结束位置，然后根据该位置从text中截取片段tokenText。
接下来的encoder的encodeText对获取到的tokenText进行第一次处理，encoder默认为DefaultEncoder，其encodeText函数默认原样返回tokenText。
formatter默认为SimpleHTMLFormatter，其highlightTerm函数会对tokenText的前后加上默认的标签。
lastEndOffset表示上一次处理词在文档中的结束位置，如果startOffset大于lastEndOffset则表示上一次处理的词到当前处理的词有一段未处理的文本，需要把这段未处理的文本添加到最终的结果newText中。这里举个列子，假设待处理的文本为“abcdddabc”，分词器的字典里只包含“abc”这一个词，则当处理完第二个abc时，startOffset为5，lastEndOffset为2（假设起始位置是0），此时要把中间的“ddd”添加到newText中去。
接下来再添加刚刚SimpleHTMLFormatter修改后的文档片段markedUpText，并重新设置lastEndOffset，然后就要清空TokenGroup，用于存储下一个不重叠的词的处理结果了。
接下来通过isNewFragment函数判断是否要重新创建一个新的TextFragment了，如果需要，则设置当前TextFragment的分数，设置已经处理过的文档的结束位置，也即newText长度（注意前面已经将currentFrag添加进docFrags列表中了），然后重新创建一个新的TextFragment，初始化并添加到docFrags中。
退出if语句，QueryScorer的getTokenScore函数会获取文档中的下一个词，匹配成功后返回该次对应的分数，然后通过TokenGroup的addToken函数记录到当前TokenGroup中。

退出for循环，此时docFrags中保存了所有经过处理后的片段，接下来遍历这些片段，并通过FragmentQueue的insertWithOverflow函数将这些片段插入FragmentQueue中，FragmentQueue默认会对这些插入的片段进行排序，最后通过其pop函数，返回得分最高的片段，封装，并返回。

初始化
Highlighter::getBestFragment->getBestFragments->getBestTextFragments->QueryScore::init

  public TokenStream init(TokenStream tokenStream) throws IOException {
    position = -1;
    termAtt = tokenStream.addAttribute(CharTermAttribute.class);
    posIncAtt = tokenStream.addAttribute(PositionIncrementAttribute.class);
    return initExtractor(tokenStream);
  }
1
2
3
4
5
6
init函数主要获取其中的CharTermAttribute和PositionIncrementAttribute，分别保存了词的byte数组和位置信息，然后调用initExtractor继续初始化。

Highlighter::getBestFragment->getBestFragments->getBestTextFragments->QueryScore::init->initExtractor

  private TokenStream initExtractor(TokenStream tokenStream) throws IOException {
    WeightedSpanTermExtractor qse = newTermExtractor(defaultField);
    qse.setMaxDocCharsToAnalyze(maxCharsToAnalyze);
    qse.setExpandMultiTermQuery(expandMultiTermQuery);
    qse.setWrapIfNotCachingTokenFilter(wrapToCaching);
    qse.setUsePayloads(usePayloads);
    this.fieldWeightedSpanTerms = qse.getWeightedSpanTermsWithScores(query, 1f,
      tokenStream, field, reader);

    return qse.getTokenStream();
  }
1
2
3
4
5
6
7
8
9
10
11
newTermExtractor用于创建WeightedSpanTermExtractor，进行相应的设置后，通过WeightedSpanTermExtractor的getWeightedSpanTermsWithScores函数创建的fieldWeightedSpanTerms的结构为Map，其中Key为查询的Term，Value为WeightedSpanTerm对应Term的信息，用于评分。
getWeightedSpanTermsWithScores还会对待处理的文档进行分词，最后结果保存在WeightedSpanTermExtractor的tokenStream成员变量中，最后返回。

Highlighter::getBestFragment->getBestFragments->getBestTextFragments->QueryScore::init->initExtractor->WeightedSpanTermExtractor::getWeightedSpanTerms

  public Map<String,WeightedSpanTerm> getWeightedSpanTerms(Query query, float boost, TokenStream tokenStream, String fieldName) throws IOException {

    Map<String,WeightedSpanTerm> terms = new PositionCheckingMap<>();
    this.tokenStream = tokenStream;
    try {
      extract(query, boost, terms);
    } finally {
      IOUtils.close(internalReader);
    }
    
    return terms;
  }
1
2
3
4
5
6
7
8
9
10
11
12
getWeightedSpanTerms函数主要通过extract函数，最后将查询语句分词后保存在Map结构terms中并返回。

Highlighter::getBestFragment->getBestFragments->getBestTextFragments->QueryScore::init->initExtractor->WeightedSpanTermExtractor::getWeightedSpanTerms->extract

  protected void extract(Query query, float boost, Map<String,WeightedSpanTerm> terms) throws IOException {

    ...
    
    else if (query instanceof TermQuery || query instanceof SynonymQuery) {
      extractWeightedTerms(terms, query, boost);
    }
    
    ...

  }
1
2
3
4
5
6
7
8
9
10
11
extract会根据不同的Query类型调用不同的函数进行处理，本章假设Query的类型为TermQuery，因此接下来调用extractWeightedTerms函数。

Highlighter::getBestFragment->getBestFragments->getBestTextFragments->QueryScore::init->initExtractor->WeightedSpanTermExtractor::getWeightedSpanTerms->extract->extractWeightedTerms

  protected void extractWeightedTerms(Map<String,WeightedSpanTerm> terms, Query query, float boost) throws IOException {
    Set<Term> nonWeightedTerms = new HashSet<>();
    final IndexSearcher searcher = new IndexSearcher(getLeafContext());
    searcher.createNormalizedWeight(query, false).extractTerms(nonWeightedTerms);

    for (final Term queryTerm : nonWeightedTerms) {
    
      if (fieldNameComparator(queryTerm.field())) {
        WeightedSpanTerm weightedSpanTerm = new WeightedSpanTerm(boost, queryTerm.text());
        terms.put(queryTerm.text(), weightedSpanTerm);
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
getLeafContext函数会对待高亮的片段进行分词，并将分词结果保存在TokenSteam中。
createNormalizedWeight和前面章节的分析类似，用于获得查询词的整体信息。
extractTerms将TermQuery中的词添加到nonWeightedTerms集合中。
最后创建WeightedSpanTerm封装了每个词的信息并添加到terms中。

Highlighter::getBestFragment->getBestFragments->getBestTextFragments->QueryScore::init->initExtractor->WeightedSpanTermExtractor::getWeightedSpanTerms->extract->extractWeightedTerms->getLeafContext

  protected LeafReaderContext getLeafContext() throws IOException {

    ...
    
    final MemoryIndex indexer = new MemoryIndex(true, usePayloads);
    tokenStream = new CachingTokenFilter(new OffsetLimitTokenFilter(tokenStream, maxDocCharsToAnalyze));
    indexer.addField(DelegatingLeafReader.FIELD_NAME, tokenStream);
    final IndexSearcher searcher = indexer.createSearcher();
    internalReader = ((LeafReaderContext) searcher.getTopReaderContext()).reader();
    this.internalReader = new DelegatingLeafReader(internalReader);


    return internalReader.getContext();
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
getLeafContext创建MemoryIndex，即将索引信息保存在内存中，而不是硬盘文件中。
接下来创建CachingTokenFilter，通过MemoryIndex的addField对文档进行分词并保存。
IndexSearcher的add函数内部会对文档进行分词，该函数较为复杂，在之前的章节已经分析过了。
最后创建DelegatingLeafReader。

处理过程
Highlighter::getBestFragment->getBestFragments->getBestTextFragments->TokenGroup::isDistinct

  boolean isDistinct() {
    return offsetAtt.startOffset() >= endOffset;
  }
1
2
3
offsetAtt的startOffset函数表示当前处理的词在文档中的起始位置，endOffset则表示之前处理的词在文档中最大的结束位置，如果startOffset大于等于endOffset，则表示当前处理的词和之前处理的词在位置上并不重叠，此时返回true，否则返回false。

Highlighter::getBestFragment->getBestFragments->getBestTextFragments->DefaultEncoder::encodeText

  public String encodeText(String originalText) {
    return originalText;
  }
1
2
3
默认的DefaultEncoder直接原样返回originalText字符串。

Highlighter::getBestFragment->getBestFragments->getBestTextFragments->SimpleHTMLFormatter::highlightTerm

  public String highlightTerm(String originalText, TokenGroup tokenGroup) {
    if (tokenGroup.getTotalScore() <= 0) {
      return originalText;
    }

    StringBuilder returnBuffer = new StringBuilder(preTag.length() + originalText.length() + postTag.length());
    returnBuffer.append(preTag);
    returnBuffer.append(originalText);
    returnBuffer.append(postTag);
    return returnBuffer.toString();
  }
1
2
3
4
5
6
7
8
9
10
11
TokenGroup的getTotalScore函数返回内部的成员变量tot，表示是否有打分高于0的文档，假设有，就对originalText的前后分别加上preTag和postTag标签，默认的标签为B标签。

Highlighter::getBestFragment->getBestFragments->getBestTextFragments->SimpleFragmenter::isNewFragment

  public boolean isNewFragment() {
    boolean isNewFrag = offsetAtt.endOffset() >= (fragmentSize * currentNumFrags);
    if (isNewFrag) {
      currentNumFrags++;
    }
    return isNewFrag;
  }
1
2
3
4
5
6
7
fragmentSize是当前段的大小，默认为100个字符，currentNumFrags表示一共处理的TextFragment的数量，fragmentSize*currentNumFrags当前的TextFragment在文档中最大的结束位置，endOffset表示如果当前处理的词在文档中的位置，如果大于fragmentSize*currentNumFrags，则超过了一个段的大小，此时返回true，并递增currentNumFrags。

Highlighter::getBestFragment->getBestFragments->getBestTextFragments->QueryScorer::getTokenScore

  public float getTokenScore() {
    String termText = termAtt.toString();
    WeightedSpanTerm weightedSpanTerm;
    if ((weightedSpanTerm = fieldWeightedSpanTerms.get(
              termText)) == null) {
      return 0;
    }
    float score = weightedSpanTerm.getWeight();
    if (!foundTerms.contains(termText)) {
      totalScore += score;
      foundTerms.add(termText);
    }
    return score;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
首先获取当前文档中待处理的词至termText中，然后从fieldWeightedSpanTerms中查看该词是否为查询的某个词，fieldWeightedSpanTerms是一个Map结构，前面分析过了，里面Key值保存了查询短语经过分词后的所有词，其Value值为WeightedSpanTerm，保存了该词的信息，如果get返回null，则说明当前文档中的词termText并不在查询短语中，直接返回0，如果不为null，则获取该词对应的WeightedSpanTerm结构，并通过getWeight函数获取该词的得分。再往下计算总得分，并把已经匹配到的词添加到foundTerms后，返回当前得分。

Highlighter::getBestFragment->getBestFragments->getBestTextFragments->TokenGroup::addToken

  void addToken(float score) {
    if (numTokens < MAX_NUM_TOKENS_PER_GROUP) {
      final int termStartOffset = offsetAtt.startOffset();
      final int termEndOffset = offsetAtt.endOffset();
      if (numTokens == 0) {
        startOffset = matchStartOffset = termStartOffset;
        endOffset = matchEndOffset = termEndOffset;
        tot += score;
      } else {
        startOffset = Math.min(startOffset, termStartOffset);
        endOffset = Math.max(endOffset, termEndOffset);
        if (score > 0) {
          if (tot == 0) {
            matchStartOffset = termStartOffset;
            matchEndOffset = termEndOffset;
          } else {
            matchStartOffset = Math.min(matchStartOffset, termStartOffset);
            matchEndOffset = Math.max(matchEndOffset, termEndOffset);
          }
          tot += score;
        }
      }
      Token token = new Token();
      token.setOffset(termStartOffset, termEndOffset);
      token.setEmpty().append(termAtt);
      tokens[numTokens] = token;
      scores[numTokens] = score;
      numTokens++;
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
MAX_NUM_TOKENS_PER_GROUP表示每个TokenGroup能够存储词的最大数量。如果不超过这个值，则首先获取当前词的起始位置termStartOffset和结束位置termEndOffset。
numTokens为0表示当前TokenGroup还没有记录任何词，此时简单的赋值就行，如果不为0，则要重新计算TokenGroup包含所有词的最小位置startOffset和最大位置endOffset，以及有效词（假设有些词的评分为0，则跳过该词）的最小位置matchStartOffset和最大位置matchEndOffset。
最后创建Token，保存相应的信息，并存储到TokenGroup的成员变量数组tokens和scores中。

最后看一下FragmentQueue是如何对插入的TextFragment进行排序的，FragmentQueue继承自PriorityQueue，重载了lessThan函数，每当插入一个元素时，都会调用lessThan函数进行判断。

FragmentQueue::lessThan

  public final boolean lessThan(TextFragment fragA, TextFragment fragB){
    if (fragA.getScore() == fragB.getScore())
      return fragA.fragNum > fragB.fragNum;
    else
      return fragA.getScore() < fragB.getScore();
  }
1
2
3
4
5
6
lessThan函数重点比较片段TextFragment的得分，在得分相等的情况下，比较fragNum，表示片段的生成次序。
————————————————
版权声明：本文为CSDN博主「二侠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/conansonic/article/details/53162302

lucene源码分析—删除索引
本章介绍lucene中索引的删除，主要介绍IndexWriter的deleteDocuments函数，该函数可以介绍两种参数，一种是Term，将所有包含该Term的文档都删除，另一种是Query，删除所有根据该Query查询得到的文档。本章只介绍第一种情况，以下面的一个例子开始。再次申明一下，博文里的基本上所有代码都在不影响整体功能的情况下进行了或多或少的修改或删除以方便阅读。

    indexWriter.deleteDocuments(new Term("id", "value"));
    indexWriter.commit();
1
2
该例子首先创建一个Term，域名为“id”，值为“value”，然后通过IndexWriter的deleteDocuments函数添加删除操作，最后通过commit函数真正执行删除。

IndexWriter::deleteDocuments

  public void deleteDocuments(Term... terms) throws IOException {
    if (docWriter.deleteTerms(terms)) {
      processEvents(true, false);
    }
  }
1
2
3
4
5
IndexWriter的成员变量docWriter是在其构造函数中创建的DocumentsWriter。deleteDocuments函数首先通过DocumentsWriter的deleteTerms函数执行主要的删除工作，然后调用processEvents函数处理删除过程中产生的事件，例如ApplyDeletesEvent、MergePendingEvent、ForcedPurgeEvent。

1. DocumentsWriter::deleteTerms
IndexWriter::deleteDocuments->DocumentsWriter::deleteTerms

  synchronized boolean deleteTerms(final Term... terms) throws IOException {
    final DocumentsWriterDeleteQueue deleteQueue = this.deleteQueue;
    deleteQueue.addDelete(terms);
    flushControl.doOnDelete();
    return applyAllDeletes(deleteQueue);
  }
1
2
3
4
5
6
DocumentsWriter的成员变量deleteQueue被初始化为DocumentsWriterDeleteQueue队列，接下来通过addDelete函数将待删除的Term列表添加到该队列中。doOnDelete和applyAllDeletes函数会根据条件将队列中添加的删除信息直接添加到缓存中，本文不考虑这种情况，即如果是先删除的词，再添加的文档，则不会对后添加的文档进行操作操作。

IndexWriter::deleteDocuments->DocumentsWriter::deleteTerms->DocumentsWriterDeleteQueue::addDelete

  void addDelete(Term... terms) {
    add(new TermArrayNode(terms));
    tryApplyGlobalSlice();
  }
1
2
3
4
首先将待删除的Term数组封装成TermArrayNode，TermArrayNode继承自Node实现链表操作。
add函数最终将Term数组添加到链表中，其函数内部实现了原子添加操作。tryApplyGlobalSlice函数用于将添加的节点写入DocumentsWriterDeleteQueue的globalBufferedUpdates缓存中。

IndexWriter::deleteDocuments->DocumentsWriter::deleteTerms->DocumentsWriterDeleteQueue::addDelete->add

  void add(Node<?> item) {
      final Node<?> currentTail = this.tail;
      if (currentTail.casNext(null, item)) {
        tailUpdater.compareAndSet(this, currentTail, item);
        return;
      }
  }
1
2
3
4
5
6
7
tail成员变量是链表中的尾节点，也即最新节点currentTail。首先通过casNext函数在尾节点currentTail的下一个节点位置上插入item，然后设置DocumentsWriterDeleteQueue的tail指向最新插入的节点item。

IndexWriter::deleteDocuments->DocumentsWriter::deleteTerms->DocumentsWriterDeleteQueue::addDelete->tryApplyGlobalSlice

  void tryApplyGlobalSlice() {
     if (updateSlice(globalSlice)) {
       globalSlice.apply(globalBufferedUpdates, BufferedUpdates.MAX_INT);
     } 
  }

  boolean updateSlice(DeleteSlice slice) {
    if (slice.sliceTail != tail) {
      slice.sliceTail = tail;
      return true;
    }
    return false;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
DocumentsWriterDeleteQueue的成员变量globalSlice被设置为DeleteSlice，其内部的成员变量sliceHead和sliceTail分别指向当前Slice的头节点和尾节点，updateSlice函数将sliceTail更新为最新插入的节点，也即上面add函数中最后插入的item。然后通过DeleteSlice的apply函数将数据写入globalBufferedUpdates中。

IndexWriter::deleteDocuments->DocumentsWriter::deleteTerms->DocumentsWriterDeleteQueue::addDelete->tryApplyGlobalSlice->DeleteSlice::apply

    void apply(BufferedUpdates del, int docIDUpto) {
      Node<?> current = sliceHead;
      do {
        current = current.next;
        current.apply(del, docIDUpto);
      } while (current != sliceTail);
      reset();
    }
    
    void reset() {
      sliceHead = sliceTail;
    }
    
    void apply(BufferedUpdates bufferedUpdates, int docIDUpto) {
      for (Term term : item) {
        bufferedUpdates.addTerm(term, docIDUpto);  
      }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
首先获得整个Slice的头节点，即sliceHead，然后依次遍历直至尾节点sliceTail，依次调用每个node的apply函数将要删除的Term信息设置进del即globalBufferedUpdates中，最后通过reset函数重置slice。

2. IndexWriter::commit
根据deleteTerms函数的分析，删除的Term信息最终会被保存在DocumentsWriter的DocumentsWriterDeleteQueue的globalBufferedUpdates中，接下来通过IndexWriter的commit的函数要取出这部分信息执行删除操作了，下面来看。

IndexWriter::commit->commitInternal->prepareCommitInternal

  public final void commit() throws IOException {
    commitInternal(config.getMergePolicy());
  }

  private final void commitInternal(MergePolicy mergePolicy) throws IOException {
    prepareCommitInternal(mergePolicy);
    finishCommit(); 
  }
1
2
3
4
5
6
7
8
IndexWriter的成员变量config在其构造函数中被创建为IndexWriterConfig，getMergePolicy返回默认的合并策略TieredMergePolicy，后面会介绍该类的函数。
commitInternal函数首先通过prepareCommitInternal函数执行文档删除的最主要操作，再调用finishCommit函数执行一些收尾工作，下面一一来看。

IndexWriter::commit->commitInternal->prepareCommitInternal

  private void prepareCommitInternal(MergePolicy mergePolicy) throws IOException {

    boolean anySegmentsFlushed = docWriter.flushAllThreads();
    processEvents(false, true);
    maybeApplyDeletes(true);
    SegmentInfos toCommit = segmentInfos.clone();
    
    ...
    
    if (anySegmentsFlushed) {
      maybeMerge(mergePolicy, MergeTrigger.FULL_FLUSH, UNBOUNDED_MAX_MERGE_SEGMENTS);
    }
    startCommit(toCommit);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
flushAllThreads函数执行最主要的flush操作，processEvents函数监听flush产生的事件，maybeApplyDeletes进行具体的删除操作，最终会在有删除的段创建.liv文件，省略的部分是进行一些初始化工作。
anySegmentsFlushed标志位表示在flush时有一些新的段产生，此时调用maybeMerge操作找出需要合并的段并对其进行合并，lucene的合并操作将在下一章介绍。

2.1 flushAllThreads
IndexWriter::commit->commitInternal->prepareCommitInternal->DocumentsWriter::flushAllThreads

  boolean flushAllThreads() {  
    flushControl.markForFullFlush(); 
    boolean anythingFlushed = false;
    DocumentsWriterPerThread flushingDWPT;
    while ((flushingDWPT = flushControl.nextPendingFlush()) != null) {
      anythingFlushed |= doFlush(flushingDWPT);
    }
    ticketQueue.forcePurge(writer);
    return anythingFlushed;
  }
1
2
3
4
5
6
7
8
9
10
DocumentsWriter的成员变量flushControl被创建为DocumentsWriterFlushControl，其markForFullFlush函数从线程池中选择对应的DocumentsWriterPerThread添加到DocumentsWriterFlushControl的flushQueue中等待flush。
flushAllThreads函数接下来通过nextPendingFlush函数遍历DocumentsWriterFlushControl中的flushQueue，依次取出在markForFullFlush函数中添加的DocumentsWriterPerThread，然后调用doFlush函数进行处理。
doFlush函数是执行flush的主要函数，该函数将添加的文档写入lucene的各个文件中，doFlush函数还将DocumentsWriterPerThread中关联的删除信息封装成SegmentFlushTicket再添加到DocumentsWriterFlushQueue队列中等待处理。
最后的forcePurge函数从将DocumentsWriterFlushQueue队列中依次取出SegmentFlushTicket并添加到IndexWriter的BufferedUpdatesStream缓存中等待最后的处理。
这里值得注意的是删除信息再各个类的各个结构之间传递来传递去是有意义的，因为flush操作会产生段的增加和更新操作，容易和删除操作产生冲突。

2.1.1 markForFullFlush
markForFullFlush函数的主要任务是从线程池DocumentsWriterPerThreadPool中选择和当前删除队列DocumentsWriterDeleteQueue一致的ThreadState，将其中的DocumentsWriterPerThread添加到flushQueue中等待处理。

IndexWriter::commit->commitInternal->prepareCommitInternal->DocumentsWriter::flushAllThreads->DocumentsWriterFlushControl::markForFullFlush

  void markForFullFlush() {
    final DocumentsWriterDeleteQueue flushingQueue = documentsWriter.deleteQueue;;
    DocumentsWriterDeleteQueue newQueue = new DocumentsWriterDeleteQueue(flushingQueue.generation+1);
    documentsWriter.deleteQueue = newQueue;

    final int limit = perThreadPool.getActiveThreadStateCount();
    for (int i = 0; i < limit; i++) {
      final ThreadState next = perThreadPool.getThreadState(i);
      if (next.dwpt.deleteQueue != flushingQueue) {
        continue;
      }
      addFlushableState(next);
    }
    
    flushQueue.addAll(fullFlushBuffer);
    fullFlushBuffer.clear();
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
markForFullFlush函数首先获取前面创建的DocumentsWriterDeleteQueue，其内部保存了删除的Term信息，然后创建一个新的DocumentsWriterDeleteQueue用于重置DocumentsWriter中原来的deleteQueue。
成员变量perThreadPool默认为DocumentsWriterPerThreadPool线程池，getActiveThreadStateCount函数获取线程池中可用线程的数量，然后通过getThreadState从中获取ThreadState，通过其成员变量dwpt即DocumentsWriterPerThread的deleteQueue是否与前面创建的DocumentsWriterDeleteQueue一致来判断是否为对应的线程，如果不是，就继续遍历线程池寻找，如果找到，就通过addFlushableState函数将其中的DocumentsWriterPerThread添加到DocumentsWriter的成员变量fullFlushBuffer和flushingWriters中，并重置该ThreadState。
最后将fullFlushBuffer中刚刚添加的所有DocumentsWriterPerThread添加到flushQueue中，并清空fullFlushBuffer。

IndexWriter::commit->commitInternal->prepareCommitInternal->DocumentsWriter::flushAllThreads->DocumentsWriterFlushControl::markForFullFlush->addFlushableState

  void addFlushableState(ThreadState perThread) {
    final DocumentsWriterPerThread dwpt = perThread.dwpt;
    if (dwpt.getNumDocsInRAM() > 0) {
      if (!perThread.flushPending) {
        setFlushPending(perThread);
      }
      final DocumentsWriterPerThread flushingDWPT = internalTryCheckOutForFlush(perThread);
      fullFlushBuffer.add(flushingDWPT);
    }
  }
1
2
3
4
5
6
7
8
9
10
如果当前DocumentsWriterPerThread在内存中的文档数量getNumDocsInRAM()大于0，就通过setFlushPending将该ThreadState的flushPending设置为true，表示该DocumentsWriterPerThread需要flush。
internalTryCheckOutForFlush函数将ThreadState中的DocumentsWriterPerThread保存在成员变量flushingWriters中并返回，同时重置该ThreadState。
最后将该DocumentsWriterPerThread添加到fullFlushBuffer中，fullFlushBuffer是一个DocumentsWriterPerThread列表。

IndexWriter::commit->commitInternal->prepareCommitInternal->DocumentsWriter::flushAllThreads->DocumentsWriterFlushControl::markForFullFlush->addFlushableState->internalTryCheckOutForFlush

  private DocumentsWriterPerThread internalTryCheckOutForFlush(ThreadState perThread) {
    final long bytes = perThread.bytesUsed;
    DocumentsWriterPerThread dwpt = perThreadPool.reset(perThread);
    flushingWriters.put(dwpt, Long.valueOf(bytes));
    return dwpt;
  }

  DocumentsWriterPerThread reset(ThreadState threadState) {
    final DocumentsWriterPerThread dwpt = threadState.dwpt;
    threadState.reset();
    return dwpt;
  }
1
2
3
4
5
6
7
8
9
10
11
12
internalTryCheckOutForFlush函数通过DocumentsWriterPerThreadPool的reset函数重置DocumentsWriterPerThread，并获取其中的DocumentsWriterPerThread。然后将该DocumentsWriterPerThread添加到flushingWriters中，flushingWriters是一个map。

2.1.2 doFlush
doFlush函数在《lucene源码分析—5》中已经重点介绍过了，该函数和删除操作并没有直接联系，这里为了完整性，只看其中和本章相关的部分代码。

IndexWriter::commit->commitInternal->prepareCommitInternal->DocumentsWriter::flushAllThreads->doFlush

  private boolean doFlush(DocumentsWriterPerThread flushingDWPT) throws IOException, AbortingException {
    boolean hasEvents = false;
    while (flushingDWPT != null) {
      hasEvents = true;
      SegmentFlushTicket ticket = ticketQueue.addFlushTicket(flushingDWPT);
      final int flushingDocsInRam = flushingDWPT.getNumDocsInRAM();
      final FlushedSegment newSegment = flushingDWPT.flush();
      ticketQueue.addSegment(ticket, newSegment);
      subtractFlushedNumDocs(flushingDocsInRam);
      flushControl.doAfterFlush(flushingDWPT);    
      flushingDWPT = flushControl.nextPendingFlush();
    }
    return hasEvents;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
ticketQueue被创建为DocumentsWriterFlushQueue，代表flush的队列，其addFlushTicket函数将DocumentsWriterPerThread中的待删除信息封装成一个SegmentFlushTicket，保存在queue列表中并返回。
getNumDocsInRAM函数返回DocumentsWriterPerThread中的文档数，该数量是新添加的文档数量，在其finishDocument函数中递增。
DocumentsWriterPerThread的flush函数完成最主要的flush操作，该函数向lucene的各个文件中保存更新的文档信息，并返回新创建的段信息FlushedSegment。
addSegment函数将flush后新的段信息FlushedSegment添加到SegmentFlushTicket中。
subtractFlushedNumDocs函数将待flush的文档数减去刚刚通过flush函数保存到文件中的文档数，并更新numDocsInRAM变量。

IndexWriter::commit->commitInternal->prepareCommitInternal->DocumentsWriter::flushAllThreads->doFlush->DocumentsWriterFlushQueue::addFlushTicket

  synchronized SegmentFlushTicket addFlushTicket(DocumentsWriterPerThread dwpt) {
    final SegmentFlushTicket ticket = new SegmentFlushTicket(dwpt.prepareFlush());
    queue.add(ticket);
    return ticket;
  }
1
2
3
4
5
DocumentsWriterPerThread的prepareFlush函数将待删除的信息封装成FrozenBufferedUpdates并返回。然后再将该FrozenBufferedUpdates封装成SegmentFlushTicket，最后添加到queue列表中并返回。
FrozenBufferedUpdates是对删除信息的一种更高效的封装。

IndexWriter::commit->commitInternal->prepareCommitInternal->DocumentsWriter::flushAllThreads->doFlush->DocumentsWriterFlushQueue::addFlushTicket->DocumentsWriterPerThread::prepareFlush

  FrozenBufferedUpdates prepareFlush() {
    final FrozenBufferedUpdates globalUpdates = deleteQueue.freezeGlobalBuffer(deleteSlice);
    return globalUpdates;
  }

  FrozenBufferedUpdates freezeGlobalBuffer(DeleteSlice callerSlice) {

    ...
    
    final FrozenBufferedUpdates packet = new FrozenBufferedUpdates(globalBufferedUpdates, false);
    globalBufferedUpdates.clear();
    return packet;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
freezeGlobalBuffer函数的主要任务是将DocumentsWriterDeleteQueue删除队列中的删除信息globalBufferedUpdates封装成FrozenBufferedUpdates并返回，然后清空globalBufferedUpdates。

IndexWriter::commit->commitInternal->prepareCommitInternal->DocumentsWriter::flushAllThreads->doFlush->DocumentsWriterPerThread::flush

  FlushedSegment flush() throws IOException, AbortingException {
    segmentInfo.setMaxDoc(numDocsInRAM);
    final SegmentWriteState flushState = new SegmentWriteState(infoStream, directory, segmentInfo, fieldInfos.finish(), pendingUpdates, new IOContext(new FlushInfo(numDocsInRAM, bytesUsed())));
    final double startMBUsed = bytesUsed() / 1024. / 1024.;

    consumer.flush(flushState);
    
    pendingUpdates.terms.clear();
    segmentInfo.setFiles(new HashSet<>(directory.getCreatedFiles()));
    final SegmentCommitInfo segmentInfoPerCommit = new SegmentCommitInfo(segmentInfo, 0, -1L, -1L, -1L);
    pendingUpdates.clear();
    BufferedUpdates segmentDeletes= null;
    
    FlushedSegment fs = new FlushedSegment(segmentInfoPerCommit, flushState.fieldInfos, segmentDeletes, flushState.liveDocs, flushState.delCountOnFlush);
    sealFlushedSegment(fs);
    
    return fs;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
flush函数首先向segmentInfo中添加更新的文档数量信息。FieldInfos.Builder的finish函数返回FieldInfos，内部封装了所有域Field的信息。接下来创建FlushInfo，包含了文档数和内存字节信息，然后创建IOContext，进而创建SegmentWriteState。
成员变量consumer为DefaultIndexingChain，其flush函数最终向.doc、.tim、.nvd、.pos、.fdx、.nvm、.fnm、.fdt、.tip文件写入信息。
完成信息的写入后，接下来清空pendingUpdates中的terms信息，并向段segmentInfo中设置新创建的文件名，然后清空pendingUpdates。
flush最后创建FlushedSegment，然后通过sealFlushedSegment函数创建新的.si文件并向其中写入段信息。

IndexWriter::commit->commitInternal->prepareCommitInternal->DocumentsWriter::flushAllThreads->doFlush->DocumentsWriterFlushControl::doAfterFlush

  synchronized void doAfterFlush(DocumentsWriterPerThread dwpt) {
    Long bytes = flushingWriters.remove(dwpt);
    flushBytes -= bytes.longValue();
    perThreadPool.recycle(dwpt);
  }
1
2
3
4
5
doAfterFlush函数从flushingWriters中移除刚刚处理过的DocumentsWriterPerThread，然后将flushBytes减去刚刚flush的字节数，recycle函数回收DocumentsWriterPerThread，便于重复利用。

2.1.3 forcePurge
回到flushAllThreads函数中，markForFullFlush函数将线程池中的DocumentsWriterPerThread添加到flushQueue中等待处理；doFlush函数依次处理flushQueue中的每个DocumentsWriterPerThread，将更新的文档flush到各个文件中，并将DocumentsWriterDeleteQueue中的删除信息封装成SegmentFlushTicket再添加到DocumentsWriterFlushQueue队列中等待处理；本小节分析的forcePurge函数最终将DocumentsWriterFlushQueue中的删除信息添加到IndexWriter的BufferedUpdatesStream中等待最终的处理，下面来看。

IndexWriter::commit->commitInternal->prepareCommitInternal->DocumentsWriter::flushAllThreads->DocumentsWriterFlushQueue::forcePurge

  int forcePurge(IndexWriter writer) throws IOException {
    return innerPurge(writer);
  }

  private int innerPurge(IndexWriter writer) throws IOException {
    int numPurged = 0;
    while (true) {
      final FlushTicket head = queue.peek();
      numPurged++;
      head.publish(writer);
      queue.poll();
      ticketCount.decrementAndGet();
    }
    return numPurged;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
doFlush函数将待删除的Term信息封装成SegmentFlushTicket添加到DocumentsWriterFlushQueue的queue成员变量，innerPurge函数依次从该队列中获取SegmentFlushTicket，调用其publish函数将其写入IndexWriter的BufferedUpdatesStream中。操作成功后通过poll函数从队列中删除该SegmentFlushTicket。

IndexWriter::commit->commitInternal->prepareCommitInternal->DocumentsWriter::flushAllThreads->DocumentsWriterFlushQueue::forcePurge->innerPurge->SegmentFlushTicket::publish

    protected void publish(IndexWriter writer) throws IOException {
      finishFlush(writer, segment, frozenUpdates);
    }
    
    protected final void finishFlush(IndexWriter indexWriter, FlushedSegment newSegment, FrozenBufferedUpdates bufferedUpdates) throws IOException {
      publishFlushedSegment(indexWriter, newSegment, bufferedUpdates);
    }
    
    protected final void publishFlushedSegment(IndexWriter indexWriter, FlushedSegment newSegment, FrozenBufferedUpdates globalPacket) throws IOException {
      indexWriter.publishFlushedSegment(newSegment.segmentInfo, segmentUpdates, globalPacket);
    }
    
    void publishFlushedSegment(SegmentCommitInfo newSegment, FrozenBufferedUpdates packet, FrozenBufferedUpdates globalPacket) throws IOException {
      bufferedUpdatesStream.push(globalPacket);
      nextGen = bufferedUpdatesStream.getNextGen();
      newSegment.setBufferedDeletesGen(nextGen);
      segmentInfos.add(newSegment);
      checkpoint();
    }

  public synchronized long push(FrozenBufferedUpdates packet) {
    packet.setDelGen(nextGen++);
    updates.add(packet);
    numTerms.addAndGet(packet.numTermDeletes);
    bytesUsed.addAndGet(packet.bytesUsed);
    return packet.delGen();
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
SegmentFlushTicket的publish函数最终会调用IndexWriter的publishFlushedSegment函数，传入的第一个参数为新增的段信息，第二个参数在Term删除时为null，不管它，最后一个参数就是前面创建的FrozenBufferedUpdates ，封装了Term的删除信息。
publishFlushedSegment通过BufferedUpdatesStream的push函数添加packet即FrozenBufferedUpdates，最终添加到BufferedUpdatesStream的updates列表中并更新相应信息。
publishFlushedSegment函数接下来将新增的段newSegment添加到SegmentInfos中，最后通过checkpoint函数更新文件的引用次数，在必要时删除文件，本章最后会分析该函数。

2.2 processEvents
回到prepareCommitInternal函数中，下面简单介绍一下processEvents函数，flushAllThreads函数操作过后会产生各类事件，processEvents函数监听这些事件并执行相应的操作。

IndexWriter::commit->commitInternal->prepareCommitInternal->processEvents

  private boolean processEvents(boolean triggerMerge, boolean forcePurge) throws IOException {
    return processEvents(eventQueue, triggerMerge, forcePurge);
  }

  private boolean processEvents(Queue<Event> queue, boolean triggerMerge, boolean forcePurge) throws IOException {
    boolean processed = false;
    if (tragedy == null) {
      Event event;
      while((event = queue.poll()) != null)  {
        processed = true;
        event.process(this, triggerMerge, forcePurge);
      }
    }
    return processed;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
lucene默认实现的event包括ApplyDeletesEvent、MergePendingEvent、ForcedPurgeEvent，processEvents函数从事件队列queue中依次取出这些事件，并调用process函数执行操作。这些事件和本章介绍的内容没有直接关系，这里就不往下看了。

2.3 maybeApplyDeletes
maybeApplyDeletes是lucene执行删除的最主要函数，下面重点分析该函数。

IndexWriter::commit->commitInternal->prepareCommitInternal->maybeApplyDeletes

  final synchronized boolean maybeApplyDeletes(boolean applyAllDeletes) throws IOException {
      return applyAllDeletesAndUpdates();
  }

  final synchronized boolean applyAllDeletesAndUpdates() throws IOException {
    final BufferedUpdatesStream.ApplyDeletesResult result;

    result = bufferedUpdatesStream.applyDeletesAndUpdates(readerPool, segmentInfos.asList());
    if (result.anyDeletes) {
      checkpoint();
    }
    if (!keepFullyDeletedSegments && result.allDeleted != null) {
      for (SegmentCommitInfo info : result.allDeleted) {
        if (!mergingSegments.contains(info)) {
          segmentInfos.remove(info);
          pendingNumDocs.addAndGet(-info.info.maxDoc());
          readerPool.drop(info);
        }
      }
      checkpoint();
    }
    bufferedUpdatesStream.prune(segmentInfos);
    return result.anyDeletes;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
BufferedUpdatesStream的applyDeletesAndUpdates执行主要的删除操作，最终将删除的文档ID标记在对应段的.liv文件中。
如果有文档被删除，则调用checkpoint函数递减对应段的引用字数，如果引用计数到达0，则删除该文件。
keepFullyDeletedSegments标记表示当一个段的文档被全部删除时，是否要删除对应的段，如果此时有的段文档被全部删除了，则遍历对应的段，从segmentInfos中删除该段，pendingNumDocs删除对应段的所有文档数，再从ReaderPool中删除该段。
最后的BufferedUpdatesStream的prune函数继续做一些收尾工作，删除前面创建的FrozenBufferedUpdates。

IndexWriter::commit->commitInternal->prepareCommitInternal->applyAllDeletesAndUpdates->maybeApplyDeletes->BufferedUpdatesStream::applyDeletesAndUpdates

  public synchronized ApplyDeletesResult applyDeletesAndUpdates(IndexWriter.ReaderPool pool, List<SegmentCommitInfo> infos) throws IOException {
    SegmentState[] segStates = null;
    long totDelCount = 0;
    long totTermVisitedCount = 0;
    boolean success = false;
    ApplyDeletesResult result = null;
    infos = sortByDelGen(infos);

    CoalescedUpdates coalescedUpdates = null;
    int infosIDX = infos.size()-1;
    int delIDX = updates.size()-1;
    
    while (infosIDX >= 0) {
      final FrozenBufferedUpdates packet = delIDX >= 0 ? updates.get(delIDX) : null;
      final SegmentCommitInfo info = infos.get(infosIDX);
      final long segGen = info.getBufferedDeletesGen();
    
      if (packet != null && segGen < packet.delGen()) {
        if (!packet.isSegmentPrivate && packet.any()) {
          if (coalescedUpdates == null) {
            coalescedUpdates = new CoalescedUpdates();
          }
          coalescedUpdates.update(packet);
        }
        delIDX--;
      } else if (packet != null && segGen == packet.delGen()) {
        ...
      } else {
        if (coalescedUpdates != null) {
          segStates = openSegmentStates(pool, infos);
          SegmentState segState = segStates[infosIDX];
    
          int delCount = 0;
          delCount += applyQueryDeletes(coalescedUpdates.queriesIterable(), segState);
          DocValuesFieldUpdates.Container dvUpdates = new DocValuesFieldUpdates.Container();
          applyDocValuesUpdatesList(coalescedUpdates.numericDVUpdates, segState, dvUpdates);
          applyDocValuesUpdatesList(coalescedUpdates.binaryDVUpdates, segState, dvUpdates);
          if (dvUpdates.any()) {
            segState.rld.writeFieldUpdates(info.info.dir, dvUpdates);
          }
          totDelCount += delCount;
        }
        infosIDX--;
      }
    }
    
    if (coalescedUpdates != null && coalescedUpdates.totalTermCount != 0) {
      if (segStates == null) {
        segStates = openSegmentStates(pool, infos);
      }
      totTermVisitedCount += applyTermDeletes(coalescedUpdates, segStates);
    }
    
    result = closeSegmentStates(pool, segStates, success, gen);
    return result;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
applyDeletesAndUpdates函数首先进入循环，遍历所有的段，然后从成员变量updates列表中获取在forcePurge函数中添加的FrozenBufferedUpdates，并获取段信息SegmentCommitInfo，再通过getBufferedDeletesGen函数获取该段的bufferedDeletesGen变量，用来表示操作的时间顺序，这里暂时叫做更新度。
下面的三个条件语句，第一个if表示要删除的FrozenBufferedUpdates存在，并且段的更新度小于要删除数据的更新度，表示可以删除，此时创建CoalescedUpdates，用来封装删除的信息，例如有些删除是通过Term，有些删除通过Query，这里全部封装起来。
第二个if语句表示更新度相同，这里不考虑这种情况。
第三个if语句表示对应的段有需要删除的数据，首先通过openSegmentStates函数将段信息封装成SegmentState，再通过applyQueryDeletes删除Query指定的删除信息，然后调用applyDocValuesUpdatesList函数检查是否有更新，如果有，则通过writeFieldUpdates进行更新，这里假设没有更新。
退出循环后，coalescedUpdates封装了待删除的Term信息，如果不为null，则通过applyTermDeletes执行删除操作。
删除完成后通过closeSegmentStates函数获取是否某段中的所有文件都被删除了，将该结果封装成ApplyDeletesResult并返回，该函数还会调用SegmentState的finish函数将applyTermDeletes函数中的标记写入到.liv文件中去，这里就不往下看了。

IndexWriter::commit->commitInternal->prepareCommitInternal->maybeApplyDeletes->BufferedUpdatesStream::applyAllDeletesAndUpdates->CoalescedUpdates::update

  void update(FrozenBufferedUpdates in) {
    totalTermCount += in.terms.size();
    terms.add(in.terms);

    ...

  }
1
2
3
4
5
6
7
CoalescedUpdates的update函数用于封装删除信息，如果是通过Term删除，则直接添加到成员变量terms列表中。

IndexWriter::commit->commitInternal->prepareCommitInternal->maybeApplyDeletes->BufferedUpdatesStream::applyAllDeletesAndUpdates->applyTermDeletes

  private synchronized long applyTermDeletes(CoalescedUpdates updates, SegmentState[] segStates) throws IOException {

    long startNS = System.nanoTime();
    int numReaders = segStates.length;
    long delTermVisitedCount = 0;
    long segTermVisitedCount = 0;
    FieldTermIterator iter = updates.termIterator();
    
    String field = null;
    SegmentQueue queue = null;
    BytesRef term;
    
    while ((term = iter.next()) != null) {
    
      if (iter.field() != field) {
        field = iter.field();
    
        queue = new SegmentQueue(numReaders);
    
        long segTermCount = 0;
        for(int i=0;i<numReaders;i++) {
          SegmentState state = segStates[i];
          Terms terms = state.reader.fields().terms(field);
          if (terms != null) {
            segTermCount += terms.size();
            state.termsEnum = terms.iterator();
            state.term = state.termsEnum.next();
            if (state.term != null) {
              queue.add(state);
            }
          }
        }
      }
    
      delTermVisitedCount++;
    
      long delGen = iter.delGen();
    
      while (queue.size() != 0) {
        SegmentState state = queue.top();
        segTermVisitedCount++;
    
        int cmp = term.compareTo(state.term);
        if (cmp < 0) {
          break;
        } else if (cmp == 0) {
        } else {
          TermsEnum.SeekStatus status = state.termsEnum.seekCeil(term);
          if (status == TermsEnum.SeekStatus.FOUND) {
          } else {
            if (status == TermsEnum.SeekStatus.NOT_FOUND) {
              state.term = state.termsEnum.term();
              queue.updateTop();
            } else {
              queue.pop();
            }
            continue;
          }
        }
    
        if (state.delGen < delGen) {
          final Bits acceptDocs = state.rld.getLiveDocs();
          state.postingsEnum = state.termsEnum.postings(state.postingsEnum, PostingsEnum.NONE);
          while (true) {
            final int docID = state.postingsEnum.nextDoc();
            if (docID == DocIdSetIterator.NO_MORE_DOCS) {
              break;
            }
            if (acceptDocs != null && acceptDocs.get(docID) == false) {
              continue;
            }
            if (!state.any) {
              state.rld.initWritableLiveDocs();
              state.any = true;
            }
            state.rld.delete(docID);
          }
        }
    
        state.term = state.termsEnum.next();
        if (state.term == null) {
          queue.pop();
        } else {
          queue.updateTop();
        }
      }
    }
    
    return delTermVisitedCount;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
applyTermDeletes函数进行具体的Term删除操作，首先通过termIterator函数获得Term的迭代器TermIterator。
第一个if语句表示field域有变化，首先通过field函数获得要删除的Term所在的域，然后创建SegmentQueue用来排序，再针对每个段，获取其中的FieldReader，并添加到SegmentQueue中。

接下来通过compareTo函数比较待删除的Term(term)和该段中存储的Term(state.term)，因为段中的词是经过排序的，因此比较的结果cmp小于0代表该段没有要找的词，直接break返回。如果相等，则什么也不做，进行下一步，如果大于0，则需要通过seekCeil函数继续在该段寻找词，如果找到了，则也继续进行下一步，如果没找到，则要通过updateTop函数更新SegmentQueue队列，等待查找下一个词，如果是其他情况，则直接退出该段的查找过程。

函数到达第二个if循环表示在该段找到了待删除的词，如果delGen表示更新度小于删除Term的更新度，则表示该词建立的时间要早于删除词的时间，此时要进行删除操作，进入while循环，循环读取包含该次的下一个文档id，获得文档id后，就要将其记录在.liv文件中，如果还未初始化，就先要调用initWritableLiveDocs函数初始化对应段的.liv文件。初始化完成后就通过delete函数在.liv文件对应的缓存中标记删除。

再往下再检查是否段没有词了，没有就通过pop函数从queue中删除该段，如果还有，则调用updateTop函数更新队列，等待下一个词的咋找。最后如果两个段都从队列queue删除了，则退出while循环。

最后返回删除的词的数量delTermVisitedCount。

IndexWriter::commit->commitInternal->prepareCommitInternal->maybeApplyDeletes->BufferedUpdatesStream::applyAllDeletesAndUpdates->applyTermDeletes ->ReadersAndUpdates::initWritableLiveDocs

  public synchronized void initWritableLiveDocs() throws IOException {
    if (liveDocsShared) {
      LiveDocsFormat liveDocsFormat = info.info.getCodec().liveDocsFormat();
      if (liveDocs == null) {
        liveDocs = liveDocsFormat.newLiveDocs(info.info.maxDoc());
      } else {
        liveDocs = liveDocsFormat.newLiveDocs(liveDocs);
      }
      liveDocsShared = false;
    }
  }
1
2
3
4
5
6
7
8
9
10
11
liveDocsFormat函数最后返回Lucene50LiveDocsFormat，然后调用其newLiveDocs函数进行初始化，返回一个FixedBitSet，该结构用bit位记录文档id用于判断该文档是否未被删除。

IndexWriter::commit->commitInternal->prepareCommitInternal->maybeApplyDeletes->BufferedUpdatesStream::applyAllDeletesAndUpdates->applyTermDeletes ->ReadersAndUpdates::delete

  public synchronized boolean delete(int docID) {
    final boolean didDelete = liveDocs.get(docID);
    if (didDelete) {
      ((MutableBits) liveDocs).clear(docID);
      pendingDeleteCount++;
    }
    return didDelete;
  }
1
2
3
4
5
6
7
8
delete函数根据文档id，在前面创建的FixedBitSet里标记位置表示删除，注意这里是将删除后保留的文档id对应的bit位置上标记为1。

IndexWriter::commit->commitInternal->prepareCommitInternal->maybeApplyDeletes->BufferedUpdatesStream::applyAllDeletesAndUpdates->closeSegmentStates

  private ApplyDeletesResult closeSegmentStates(IndexWriter.ReaderPool pool, SegmentState[] segStates, boolean success, long gen) throws IOException {
    int numReaders = segStates.length;
    Throwable firstExc = null;
    List<SegmentCommitInfo> allDeleted = new ArrayList<>();
    long totDelCount = 0;
    for (int j=0;j<numReaders;j++) {
      SegmentState segState = segStates[j];
      totDelCount += segState.rld.getPendingDeleteCount() - segState.startDelCount;
      segState.reader.getSegmentInfo().setBufferedDeletesGen(gen);
      int fullDelCount = segState.rld.info.getDelCount() + segState.rld.getPendingDeleteCount();
      if (fullDelCount == segState.rld.info.info.maxDoc()) {
        allDeleted.add(segState.reader.getSegmentInfo());
      }
      segStates[j].finish(pool);
    }

    return new ApplyDeletesResult(totDelCount > 0, gen, allDeleted);      
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
closeSegmentStates函数在删除操作执行后检查是否删除了某个段的所有文档，如果有，就将其添加到allDeleted列表中，最终在返回时封装成ApplyDeletesResult。closeSegmentStates函数也会遍历每个段，对前面创建的SegmentState执行finish函数，将对应的FixedBitSet结构写入.liv文件中去，因为涉及到文件格式，这里就不往下看了。

2.4 startCommit
IndexWriter::commit->commitInternal->prepareCommitInternal->startCommit

  private void startCommit(final SegmentInfos toSync) throws IOException {

    ...
    
    toSync.prepareCommit(directory);
    
    ...
    
    filesToSync = toSync.files(false);
    directory.sync(filesToSync);
    
    ...

  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
执行到这里时，有一些段由于flush操作新生成，有一些段有数据生成，有一些段进行了合并操作，执行到这里，需要对segments段文件执行更新操作。
首先调用SegmentInfos的prepareCommit函数，创建pending_segments文件，向其写入基本信息。
然后通过files函数获得创建后索引目录下最终的所有文件，再通过sync函数将文件同步到硬盘中去。

2.5 finishCommit
执行到这里，主要的删除任务已经结束，最终删除的文档ID会被标记在对应段的.liv文件中，finishCommit函数完成接下来的收尾工作。

IndexWriter::commit->commitInternal->finishCommit

  private final void finishCommit() throws IOException {
    pendingCommit.finishCommit(directory);
    deleter.checkpoint(pendingCommit, true);
    segmentInfos.updateGeneration(pendingCommit);
    rollbackSegments = pendingCommit.createBackupSegmentInfos();
  }
1
2
3
4
5
6
finishCommit函数重命名前面创建的临时段文件。成员变量deleter被初始化为IndexFileDeleter，其checkpoint函数检查是否有待删除的文件并将其删除。
updateGeneration函数更新段的generation信息。createBackupSegmentInfos函数备份当前段的最新信息保存在rollbackSegments中。

IndexWriter::commit->commitInternal->finishCommit->SegmentInfos::finishCommit

  final String finishCommit(Directory dir) throws IOException {
    final String src = IndexFileNames.fileNameFromGeneration(IndexFileNames.PENDING_SEGMENTS, "", generation);
    String dest = IndexFileNames.fileNameFromGeneration(IndexFileNames.SEGMENTS, "", generation);
    dir.renameFile(src, dest);
    pendingCommit = false;
    lastGeneration = generation;
    return dest;
  }
1
2
3
4
5
6
7
8
finishCommit函数主要完成将索引目录下的临时段文件重命名为正式的段文件，例如将pending_segments_1文件重命名为segments_1文件。

IndexWriter::commit->commitInternal->finishCommit->IndexFileDeleter::checkpoint

  public void checkpoint(SegmentInfos segmentInfos, boolean isCommit) throws IOException {
    incRef(segmentInfos, isCommit);
    commits.add(new CommitPoint(commitsToDelete, directoryOrig, segmentInfos));
    policy.onCommit(commits);
    deleteCommits(); 
  }
1
2
3
4
5
6
checkpoint函数首先通过incRef递增段中每个文件的引用次数，然后将待删除文件的信息封装成CommitPoint并添加到commits列表中。
policy是在LiveIndexWriterConfig中默认的KeepOnlyLastCommitDeletionPolicy，onCommit函数用来保存最新的CommitPoint。deleteCommits函数降低文件的引用次数，可能执行最终的删除操作。

IndexWriter::commit->commitInternal->finishCommit->IndexFileDeleter::checkpoint->incRef

  void incRef(SegmentInfos segmentInfos, boolean isCommit) throws IOException {
    for(final String fileName: segmentInfos.files(isCommit)) {
      incRef(fileName);
    }
  }

  public Collection<String> files(boolean includeSegmentsFile) throws IOException {
    HashSet<String> files = new HashSet<>();
    if (includeSegmentsFile) {
      final String segmentFileName = getSegmentsFileName();
      if (segmentFileName != null) {
        files.add(segmentFileName);
      }
    }
    final int size = size();
    for(int i=0;i<size;i++) {
      final SegmentCommitInfo info = info(i);
      files.addAll(info.files());
    }

    return files;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
getSegmentsFileName函数获得对应段的文件名，例如segments_1，然后将段文件名添加到files集合中。
size返回段信息SegmentCommitInfo的数量。info函数从SegmentCommitInfo列表中获取对应的SegmentCommitInfo，然后调用其files函数获取该段对应的诸如.doc、.tim、.si、.nvd、.pos、.fdx、.nvm、.fnm、.fdt、.tip等文件名，然后将这些文件名添加到files集合中并返回。

IndexWriter::commit->commitInternal->finishCommit->IndexFileDeleter::checkpoint->incRef->incRef

  void incRef(String fileName) {
    RefCount rc = getRefCount(fileName);
    rc.IncRef();
  }

  private RefCount getRefCount(String fileName) {
    RefCount rc;
    if (!refCounts.containsKey(fileName)) {
      rc = new RefCount(fileName);
      refCounts.put(fileName, rc);
    } else {
      rc = refCounts.get(fileName);
    }
    return rc;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
getRefCount从IndexFileDeleter的成员变量refCounts中获得当前每个文件的引用次数，然后将其加1。

IndexWriter::commit->commitInternal->finishCommit->IndexFileDeleter::checkpoint->KeepOnlyLastCommitDeletionPolicy::onCommit

  public void onCommit(List<? extends IndexCommit> commits) {
    int size = commits.size();
    for(int i=0;i<size-1;i++) {
      commits.get(i).delete();
    }
  }

  public void delete() {
    if (!deleted) {
      deleted = true;
      commitsToDelete.add(this);
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
onCommit会将最新添加的IndexCommit继续保存在commits列表中，并将其余的IndexCommit添加到commitsToDelete列表中。

IndexWriter::commit->commitInternal->finishCommit->IndexFileDeleter::checkpoint->deleteCommits

  private void deleteCommits() {
    int size = commitsToDelete.size();

    for(int i=0;i<size;i++) {
      CommitPoint commit = commitsToDelete.get(i);
      decRef(commit.files);
    }
    commitsToDelete.clear();
    
    size = commits.size();
    int readFrom = 0;
    int writeTo = 0;
    while(readFrom < size) {
      CommitPoint commit = commits.get(readFrom);
      if (!commit.deleted) {
        if (writeTo != readFrom) {
          commits.set(writeTo, commits.get(readFrom));
        }
        writeTo++;
      }
      readFrom++;
    }
    
    while(size > writeTo) {
      commits.remove(size-1);
      size--;
    }

  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
遍历commitsToDelete中的CommitPoint，降低其对应文件的引用次数，如果等于0，就将其删除。
删除完成后，再通过一个while循环更新commits列表，取出删除的CommitPoint。

IndexWriter::commit->commitInternal->finishCommit->IndexFileDeleter::checkpoint->deleteCommits->decRef

  void decRef(Collection<String> files) throws IOException {
    Set<String> toDelete = new HashSet<>();
    for(final String file : files) {
      if (decRef(file)) {
        toDelete.add(file);
      }
    }
    deleteFiles(toDelete);
  }
1
2
3
4
5
6
7
8
9
decRef函数和前面介绍的incRef类似，用于降低该文件的引用次数，如果等于0，就返回true，表示要删除该文件，将其添加到待删除的文件列表toDelete中，最后调用deleteFiles删除这些文件。
————————————————
版权声明：本文为CSDN博主「二侠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/conansonic/article/details/53235528

lucene源码分析—合并段
本章分析lucene中段的合并，承接上一章的分析，IndexWriter的prepareCommitInternal函数最后会调用maybeMerge检查是否需要对段进行合并，如果需要，则进行合并，下面来看。

maybeMerge(mergePolicy, MergeTrigger.FULL_FLUSH, UNBOUNDED_MAX_MERGE_SEGMENTS);
1
IndexWriter::maybeMerge

  private final void maybeMerge(MergePolicy mergePolicy, MergeTrigger trigger, int maxNumSegments) throws IOException {
    boolean newMergesFound = updatePendingMerges(mergePolicy, trigger, maxNumSegments);
    mergeScheduler.merge(this, trigger, newMergesFound);
  }
1
2
3
4
传入的参数mergePolicy为TieredMergePolicy，在LiveIndexWriterConfig配置中创建，TieredMergePolicy类主要用于决定在什么情况下触发一次段的合并。maybeMerge函数首先通过updatePendingMerges函数查找需要合并的段，然后通过merge函数执行合并操作。

1. updatePendingMerges
IndexWriter::maybeMerge->updatePendingMerges

  private synchronized boolean updatePendingMerges(MergePolicy mergePolicy, MergeTrigger trigger, int maxNumSegments) throws IOException {

    boolean newMergesFound = false;
    final MergePolicy.MergeSpecification spec;
    
    spec = mergePolicy.findMerges(trigger, segmentInfos, this);
    
    newMergesFound = spec != null;
    if (newMergesFound) {
      final int numMerges = spec.merges.size();
      for(int i=0;i<numMerges;i++) {
        registerMerge(spec.merges.get(i));
      }
    }
    return newMergesFound;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
updatePendingMerges函数首先通过TieredMergePolicy的findMerges函数计算符合合并的段，保存在结果MergeSpecification中。接下来通过registerMerge函数对即将合并的段进行注册。

IndexWriter::maybeMerge->updatePendingMerges->TieredMergePolicy::findMerges

  public MergeSpecification findMerges(MergeTrigger mergeTrigger, SegmentInfos infos, IndexWriter writer) throws IOException {
    final Collection<SegmentCommitInfo> merging = writer.getMergingSegments();
    final Collection<SegmentCommitInfo> toBeMerged = new HashSet<>();

    final List<SegmentCommitInfo> infosSorted = new ArrayList<>(infos.asList());
    Collections.sort(infosSorted, new SegmentByteSizeDescending(writer));
    
    long totIndexBytes = 0;
    long minSegmentBytes = Long.MAX_VALUE;
    
    for(SegmentCommitInfo info : infosSorted) {
      final long segBytes = size(info, writer);
      minSegmentBytes = Math.min(segBytes, minSegmentBytes);
      totIndexBytes += segBytes;
    }
    
    int tooBigCount = 0;
    while (tooBigCount < infosSorted.size()) {
      long segBytes = size(infosSorted.get(tooBigCount), writer);
      if (segBytes < maxMergedSegmentBytes/2.0) {
        break;
      }
      totIndexBytes -= segBytes;
      tooBigCount++;
    }
    
    minSegmentBytes = floorSize(minSegmentBytes);
    
    long levelSize = minSegmentBytes;
    long bytesLeft = totIndexBytes;
    double allowedSegCount = 0;
    while(true) {
      final double segCountLevel = bytesLeft / (double) levelSize;
      if (segCountLevel < segsPerTier) {
        allowedSegCount += Math.ceil(segCountLevel);
        break;
      }
      allowedSegCount += segsPerTier;
      bytesLeft -= segsPerTier * levelSize;
      levelSize *= maxMergeAtOnce;
    }
    int allowedSegCountInt = (int) allowedSegCount;
    MergeSpecification spec = null;
    
    while(true) {
      long mergingBytes = 0;
      final List<SegmentCommitInfo> eligible = new ArrayList<>();
      for(int idx = tooBigCount; idx<infosSorted.size(); idx++) {
        final SegmentCommitInfo info = infosSorted.get(idx);
        if (merging.contains(info)) {
          mergingBytes += size(info, writer);
        } else if (!toBeMerged.contains(info)) {
          eligible.add(info);
        }
      }
      final boolean maxMergeIsRunning = mergingBytes >= maxMergedSegmentBytes;
      if (eligible.size() == 0) {
        return spec;
      }
    
      if (eligible.size() > allowedSegCountInt) {
        MergeScore bestScore = null;
        List<SegmentCommitInfo> best = null;
        boolean bestTooLarge = false;
        long bestMergeBytes = 0;
    
        for(int startIdx = 0;startIdx <= eligible.size()-maxMergeAtOnce; startIdx++) {
    
          long totAfterMergeBytes = 0;
    
          final List<SegmentCommitInfo> candidate = new ArrayList<>();
          boolean hitTooLarge = false;
          for(int idx = startIdx;idx<eligible.size() && candidate.size() < maxMergeAtOnce;idx++) {
            final SegmentCommitInfo info = eligible.get(idx);
            final long segBytes = size(info, writer);
    
            if (totAfterMergeBytes + segBytes > maxMergedSegmentBytes) {
              hitTooLarge = true;
              continue;
            }
            candidate.add(info);
            totAfterMergeBytes += segBytes;
          }
    
          final MergeScore score = score(candidate, hitTooLarge, mergingBytes, writer);
          if ((bestScore == null || score.getScore() < bestScore.getScore()) && (!hitTooLarge || !maxMergeIsRunning)) {
            best = candidate;
            bestScore = score;
            bestTooLarge = hitTooLarge;
            bestMergeBytes = totAfterMergeBytes;
          }
        }
    
        if (best != null) {
          if (spec == null) {
            spec = new MergeSpecification();
          }
          final OneMerge merge = new OneMerge(best);
          spec.add(merge);
          for(SegmentCommitInfo info : merge.segments) {
            toBeMerged.add(info);
          }
        } else {
          return spec;
        }
      } else {
        return spec;
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
109
110
findMerges函数首先通过sort对当前所有段按照其占用的字节数从大到小排序。
第一个for语句遍历所有的段，计算其中占用字节的最小值，保存在minSegmentBytes中，再计算所有段占用的总字节数，保存在totIndexBytes中。
接下来while语句判断是否有段的大小大于maxMergedSegmentBytes的一半，如果有，则记录最大段在排序完的infosSorted中的位置为tooBigCount，后续要略过这些段。
再往下通过floorSize函数对段大小的最小值minSegmentBytes进行调整，以避免后面阈值的计算。
再往下的while循环计算段数量的阈值allowedSegCount，当正常段的数量大于这个阈值时，就需要进行一次合并。计算公式为所有段的总字节数除以最小段的字节数，如果大于segsPerTier（默认为10），则将阈值递增segsPerTier，进行一次记录，并将最小段的字节数乘以maxMergeAtOnce（默认为10），再循环计算。

进入最大的while循环，第一个for循环根据前面计算的tooBigCount值忽略掉较大段的SegmentCommitInfo，将剩余的段添加到eligible列表中，如果段已经被添加到merging列表中，则表示该段正在进行合并，此时只统计该段的大小mergingBytes，并不添加到eligible列表中。
maxMergeAtOnce变量也是一次可以合并的最大段的数量，默认10，因为存在该限制，如果此时lucene索引目录中段的数量大于maxMergeAtOnce，就需要从所有段中选择最优的maxMergeAtOnce个段，但是这里并不是遍历所有的段的组合，而是一种次优的选择，例如lucene索引目录中有0~11个段，该for循环只是循环比较0~9、1~10、2~11这三个组合，并选出其中最优的组合。
如果正常段的数量大于allowedSegCountInt和maxMergeAtOnce，这种情况下需要进行合并，否则，直接返回一个空的MergeSpecification，表示不要进行合并。
totAfterMergeBytes表示即将要合并的段的大小，如果totAfterMergeBytes加上待处理的段的大小大于maxMergedSegmentBytes，则设定为不满足合并条件，继续循环，否则加入候选段列表candidate中，并递增totAfterMergeBytes。
再往下通过score函数，对候选段评分，具体的评分公式涉及到公式问题，这里就不看了。接下来要比较上一次候选段的评分和本次选出的段的评分，如果本次选出的段由于上一次的候选段，则进行替换。

退出for循环后，再往下创建MergeSpecification，将所有要封装的段封装成OneMerge，并添加到MergeSpecification中。

IndexWriter::maybeMerge->updatePendingMerges->registerMerge

  final synchronized boolean registerMerge(MergePolicy.OneMerge merge) throws IOException {

    boolean isExternal = false;
    for(SegmentCommitInfo info : merge.segments) {
      if (mergingSegments.contains(info)) {
        return false;
      }
      if (!segmentInfos.contains(info)) {
        return false;
      }
      if (info.info.dir != directoryOrig) {
        isExternal = true;
      }
      if (segmentsToMerge.containsKey(info)) {
        merge.maxNumSegments = mergeMaxNumSegments;
      }
    }
    
    ensureValidMerge(merge);
    
    pendingMerges.add(merge);


    merge.mergeGen = mergeGen;
    merge.isExternal = isExternal;
    
    for(SegmentCommitInfo info : merge.segments) {
      mergingSegments.add(info);
    }
    
    for(SegmentCommitInfo info : merge.segments) {
      if (info.info.maxDoc() > 0) {
        final int delCount = numDeletedDocs(info);
        final double delRatio = ((double) delCount)/info.info.maxDoc();
        merge.estimatedMergeBytes += info.sizeInBytes() * (1.0 - delRatio);
        merge.totalMergeBytes += info.sizeInBytes();
      }
    }
    
    merge.registerDone = true;
    
    return true;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
registerMerge函数遍历所有的候选段，通过集合mergingSegments判断即将合并的段是否正在被合并中，如果是则直接返回。如果即将合并的段已经被合并或删除，即不在segmentInfos中，也直接返回。如果是外部的目录，设置isExternal为true。
接下来将待合并的候选段集合添加到pendingMerges中。
然后遍历所有段，依次添加到mergingSegments中，表示正在合并。

再往下再次遍历所有段，numDeletedDocs函数获得该段需要删除的文档数，统计estimatedMergeBytes和totalMergeBytes，其中estimatedMergeBytes包含删除的文档数。

2. merge
updatePendingMerges函数找出了需要合并的段集合，接下来通过merge函数执行合并操作，下面来看。

IndexWriter::maybeMerge->ConcurrentMergeScheduler::merge

  public synchronized void merge(IndexWriter writer, MergeTrigger trigger, boolean newMergesFound) throws IOException {
    while (true) {
      OneMerge merge = writer.getNextMerge();
      if (merge == null) {
        return;
      }

      final MergeThread merger = getMergeThread(writer, merge);
      mergeThreads.add(merger);
      merger.start();
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
merge函数首先循环调用getNextMerge函数依次取出updatePendingMerges函数注册的候选段集合OneMerge，然后通过getMergeThread函数创建用于合并的线程MergeThread，然后通过其start函数执行合并操作。

IndexWriter::maybeMerge->ConcurrentMergeScheduler::merge->IndexWriter::getNextMerge

  public synchronized MergePolicy.OneMerge getNextMerge() {
    if (pendingMerges.size() == 0) {
      return null;
    } else {
      MergePolicy.OneMerge merge = pendingMerges.removeFirst();
      runningMerges.add(merge);
      return merge;
    }
  }
1
2
3
4
5
6
7
8
9
getNextMerge函数返回前面注册的候选段集合OneMerge，并将其添加到runningMerges中。

IndexWriter::maybeMerge->ConcurrentMergeScheduler::merge->getMergeThread

  protected synchronized MergeThread getMergeThread(IndexWriter writer, OneMerge merge) throws IOException {
    final MergeThread thread = new MergeThread(writer, merge);
    thread.setDaemon(true);
    thread.setName("Lucene Merge Thread #" + mergeThreadCount++);
    return thread;
  }
1
2
3
4
5
6
getMergeThread函数创建MergeThread，进行相应的设置并返回。

MergeThread::run

    public void run() {
      doMerge(writer, merge);
      merge(writer, MergeTrigger.MERGE_FINISHED, true);
      removeMergeThread();
      ...
    }
1
2
3
4
5
6
MergeThread线程通过doMerge函数完成段的合并，merge函数检查是否还有merge操作需要执行，如果需要则开启一个线程继续执行，removeMergeThread函数从mergeThreads中移除当前线程。

MergeThread::run->doMerge

  protected void doMerge(IndexWriter writer, OneMerge merge) throws IOException {
    writer.merge(merge);
  }

  public void merge(MergePolicy.OneMerge merge) throws IOException {
    rateLimiters.set(merge.rateLimiter);
    final long t0 = System.currentTimeMillis();
    final MergePolicy mergePolicy = config.getMergePolicy();
    mergeInit(merge);
    mergeMiddle(merge, mergePolicy);
    mergeFinish(merge);
  }
1
2
3
4
5
6
7
8
9
10
11
12
mergeInit函数进行合并的初始化，并创建合并后的新段，mergeMiddle是合并的主要函数，内部完成合并操作，最后的mergeFinish函数执行一些收尾工作，下面一一分析。

2.1 mergeInit
MergeThread::run->doMerge->IndexWriter::merge->mergeInit

  final synchronized void mergeInit(MergePolicy.OneMerge merge) throws IOException {
      _mergeInit(merge);
  }
  synchronized private void _mergeInit(MergePolicy.OneMerge merge) throws IOException {

    final BufferedUpdatesStream.ApplyDeletesResult result = bufferedUpdatesStream.applyDeletesAndUpdates(readerPool, merge.segments);
    
    ...
    
    final String mergeSegmentName = newSegmentName();
    SegmentInfo si = new SegmentInfo(directoryOrig, Version.LATEST, mergeSegmentName, -1, false, codec, Collections.emptyMap(), StringHelper.randomId(), new HashMap<>());
    Map<String,String> details = new HashMap<>();
    details.put("mergeMaxNumSegments", "" + merge.maxNumSegments);
    details.put("mergeFactor", Integer.toString(merge.segments.size()));
    setDiagnostics(si, SOURCE_MERGE, details);
    merge.setMergeInfo(new SegmentCommitInfo(si, 0, -1L, -1L, -1L));
    
    ...
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
_mergeInit函数首先通过BufferedUpdatesStream的applyDeletesAndUpdates函数执行缓存中还没删除和更新的任务，接下来通过newSegmentName函数获得新的段名（例如“_3”），然后创建对应该段的SegmentInfo并进行相应的设置，最后将其设置到OneMerge中。

2.2 mergeMiddle
MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle

  private int mergeMiddle(MergePolicy.OneMerge merge, MergePolicy mergePolicy) throws IOException {

    List<SegmentCommitInfo> sourceSegments = merge.segments;
    IOContext context = new IOContext(merge.getStoreMergeInfo());
    final TrackingDirectoryWrapper dirWrapper = new TrackingDirectoryWrapper(mergeDirectory);
    merge.readers = new ArrayList<>(sourceSegments.size());
    
    boolean success = false;
    
    int segUpto = 0;
    while(segUpto < sourceSegments.size()) {
    
      final SegmentCommitInfo info = sourceSegments.get(segUpto);
      final ReadersAndUpdates rld = readerPool.get(info, true);
    
      SegmentReader reader;
      final Bits liveDocs;
      final int delCount;
    
      reader = rld.getReaderForMerge(context);
      liveDocs = rld.getReadOnlyLiveDocs();
      delCount = rld.getPendingDeleteCount() + info.getDelCount();
    
      if (reader.numDeletedDocs() != delCount) {
        SegmentReader newReader = new SegmentReader(info, reader, liveDocs, info.info.maxDoc() - delCount);
        rld.release(reader);
        reader = newReader;
      }
    
      merge.readers.add(reader);
      segUpto++;
    }
    
    final SegmentMerger merger = new SegmentMerger(merge.getMergeReaders(), merge.info.info, infoStream, dirWrapper, globalFieldNumberMap, context);
    if (merger.shouldMerge()) {
      merger.merge();
    }
    
    MergeState mergeState = merger.mergeState;
    merge.info.info.setFiles(new HashSet<>(dirWrapper.getCreatedFiles()));
    
    ...
    
    codec.segmentInfoFormat().write(directory, merge.info.info, context);
    commitMerge(merge, mergeState)
    return merge.info.info.maxDoc();
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
getStoreMergeInfo返回MergeInfo，封装成IOContext。MergeInfo包含了本次合并的信息，例如合并的文档数、字节数、段数等等。
然后将合并的目录mergeDirectory（默认就是lucene的索引目录）封装成TrackingDirectoryWrapper。mergeDirectory在IndexWriter的构造函数中创建。
接着遍历即将合并的段，ReaderPool的get函数内部会创建并保存一个ReadersAndUpdates，最后返回该ReadersAndUpdates，ReadersAndUpdates内部封装了SegmentReader，用于删除、更新或合并操作。
再往下，getReaderForMerge函数创建并返回对应段SegmentReader，其中SegmentReader的构造函数会去读取.liv文件，根据文件中的创建Bits，内部存储了该段文档的删除信息，然后存储在成员变量liveDocs中。接着getReadOnlyLiveDocs函数直接返回刚刚读取的liveDocs。delCount记录了段删除的文档数，其中包括等待删除和已经删除的文档数的和。
如果delCount和numDeletedDocs函数返回的文档数不等，表示期间别的线程已经删除了部分文档，此时要重新创建SegmentReader，这部分代码省略。最后将新的SegmentReader添加到OneMerge的readers列表中。

接下来创建SegmentMerger，SegmentMerger封装了基本上一次合并的所有信息，其内部创建的MergeState封装了所有段的所有文件的各个读取器Reader，然后调用SegmentMerger的merge函数执行合并操作，该函数是整个合并过程的核心函数。
合并完成后，调用TrackingDirectoryWrapper的getCreatedFiles函数获得刚刚合并过程中创建的所有新文件，然后设置进OneMerge中。
接下来省略的一大段代码用来判断是否要合并到cfs文件中，如果需要则进行合并。
再往下调用segmentInfoFormat函数获得Lucene50SegmentInfoFormat，调用其write函数在新段中创建.si文件并向其中添加段的信息。
commitMerge完成收尾工作，包括未删除的文档和更新操作，关于lucene的更新将在后面的章节介绍，这里就不仔细往下看了。
最后返回合并后新段的最大文档数。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->ReaderPool::get

    public synchronized ReadersAndUpdates get(SegmentCommitInfo info, boolean create) {
      ReadersAndUpdates rld = readerMap.get(info);
      if (rld == null) {
        rld = new ReadersAndUpdates(IndexWriter.this, info);
        readerMap.put(info, rld);
      }
      return rld;
    }
1
2
3
4
5
6
7
8
将IndexWriter和段的信息SegmentCommitInfo封装成ReadersAndUpdates，并添加到readerMap中并返回。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->ReadersAndUpdates::getReaderForMerge

  synchronized SegmentReader getReaderForMerge(IOContext context) throws IOException {
    isMerging = true;
    return getReader(context);
  }

  public SegmentReader getReader(IOContext context) throws IOException {
    if (reader == null) {
      reader = new SegmentReader(info, context);
      if (liveDocs == null) {
        liveDocs = reader.getLiveDocs();
      }
    }
    return reader;
  }

  public SegmentReader(SegmentCommitInfo si, IOContext context) throws IOException {
    this.si = si;
    core = new SegmentCoreReaders(si.info.dir, si, context);
    segDocValues = new SegmentDocValues();
    final Codec codec = si.info.getCodec();
    if (si.hasDeletions()) {
        liveDocs = codec.liveDocsFormat().readLiveDocs(directory(), si, IOContext.READONCE);
    }

    numDocs = si.info.maxDoc() - si.getDelCount();
    fieldInfos = initFieldInfos();
    docValuesProducer = initDocValuesProducer();
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
ReadersAndUpdates的getReaderForMerge函数会创建SegmentReader，SegmentReader在之前的文章介绍过了，其封装了对该段各个文件的读取类，用于读取各个文件。hasDeletions返回true表示该段包含删除标记，此时通过readLiveDocs读取.liv文件，将删除的标记信息保存在liveDocs中。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::SegmentMerger->MergeState::MergeState

  MergeState(List<CodecReader> readers, SegmentInfo segmentInfo, InfoStream infoStream){
    int numReaders = readers.size();
    docMaps = new DocMap[numReaders];
    docBase = new int[numReaders];
    maxDocs = new int[numReaders];
    fieldsProducers = new FieldsProducer[numReaders];
    normsProducers = new NormsProducer[numReaders];
    storedFieldsReaders = new StoredFieldsReader[numReaders];
    termVectorsReaders = new TermVectorsReader[numReaders];
    docValuesProducers = new DocValuesProducer[numReaders];
    pointsReaders = new PointsReader[numReaders];
    fieldInfos = new FieldInfos[numReaders];
    liveDocs = new Bits[numReaders];

    for(int i=0;i<numReaders;i++) {
      final CodecReader reader = readers.get(i);
    
      maxDocs[i] = reader.maxDoc();
      liveDocs[i] = reader.getLiveDocs();
      fieldInfos[i] = reader.getFieldInfos();
    
      normsProducers[i] = reader.getNormsReader();
      if (normsProducers[i] != null) {
        normsProducers[i] = normsProducers[i].getMergeInstance();
      }
    
      docValuesProducers[i] = reader.getDocValuesReader();
      if (docValuesProducers[i] != null) {
        docValuesProducers[i] = docValuesProducers[i].getMergeInstance();
      }
    
      storedFieldsReaders[i] = reader.getFieldsReader();
      if (storedFieldsReaders[i] != null) {
        storedFieldsReaders[i] = storedFieldsReaders[i].getMergeInstance();
      }
    
      termVectorsReaders[i] = reader.getTermVectorsReader();
      if (termVectorsReaders[i] != null) {
        termVectorsReaders[i] = termVectorsReaders[i].getMergeInstance();
      }
    
      fieldsProducers[i] = reader.getPostingsReader().getMergeInstance();
      pointsReaders[i] = reader.getPointsReader();
      if (pointsReaders[i] != null) {
        pointsReaders[i] = pointsReaders[i].getMergeInstance();
      }
    }
    
    this.segmentInfo = segmentInfo;
    setDocMaps(readers);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
MergeState的构造函数主要封装了每个段的每个文件的Reader，例如fieldsProducers数组中包含了用于读取每个段.doc、.pos文件的BlockTreeTermsReader。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge

  MergeState merge() throws IOException {
    mergeFieldInfos();
    int numMerged = mergeFields();

    final SegmentWriteState segmentWriteState = new SegmentWriteState(mergeState.infoStream, directory, mergeState.segmentInfo, mergeState.mergeFieldInfos, null, context);
    mergeTerms(segmentWriteState);
    mergeNorms(segmentWriteState);
    codec.fieldInfosFormat().write(directory, mergeState.segmentInfo, "", mergeState.mergeFieldInfos, context);
    return mergeState;
  }
1
2
3
4
5
6
7
8
9
10
首先通过mergeFieldInfos函数对域进行合并。然后根据最新合并的域信息，通过mergeFields函数对所有段的.fdt和.fdx文件进行合并。
再往下创建SegmentWriteState，接着调用mergeTerms函数，用于合并每个段的.doc、.pos、.pay、.tip、.tim文件。
然后继续通过mergeNorms函数合并.nvd、.nvm文件。
最后通过fieldInfosFormat函数获得Lucene60FieldInfosFormat，调用write向新的.fnm文件写入最新段的域信息。

2.2.1 mergeFieldInfos
MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeFieldInfos

  public void mergeFieldInfos() throws IOException {
    for (FieldInfos readerFieldInfos : mergeState.fieldInfos) {
      for (FieldInfo fi : readerFieldInfos) {
        fieldInfosBuilder.add(fi);
      }
    }
    mergeState.mergeFieldInfos = fieldInfosBuilder.finish();
  }
1
2
3
4
5
6
7
8
mergeFieldInfos函数遍历每个段的每个域，将每个段的域信息FieldInfo通过add函数添加到fieldInfosBuilder中，最后调用FieldInfos.Builder的finish函数整合所有域的信息，为新的段生成一个新的域信息FieldInfos。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeFieldInfos->FieldInfos.Builder::add

    public FieldInfo add(FieldInfo fi) {
      return addOrUpdateInternal(fi.name, fi.number, fi.hasVectors(),
                                 fi.omitsNorms(), fi.hasPayloads(),
                                 fi.getIndexOptions(), fi.getDocValuesType(),
                                 fi.getPointDimensionCount(), fi.getPointNumBytes());
    }
    
    private FieldInfo addOrUpdateInternal(String name, int preferredFieldNumber, boolean storeTermVector, boolean omitNorms, boolean storePayloads,IndexOptions indexOptions, DocValuesType docValues, int dimensionCount, int dimensionNumBytes) {
    
      FieldInfo fi = fieldInfo(name);
      if (fi == null) {
        final int fieldNumber = globalFieldNumbers.addOrGet(name, preferredFieldNumber, docValues, dimensionCount, dimensionNumBytes);
        fi = new FieldInfo(name, fieldNumber, storeTermVector, omitNorms, storePayloads, indexOptions, docValues, -1, new HashMap<>(), dimensionCount, dimensionNumBytes);
        globalFieldNumbers.verifyConsistent(Integer.valueOf(fi.number), fi.name, fi.getDocValuesType());
        byName.put(fi.name, fi);
      } else {
        fi.update(storeTermVector, omitNorms, storePayloads, indexOptions, dimensionCount, dimensionNumBytes);
        if (docValues != DocValuesType.NONE) {
          boolean updateGlobal = fi.getDocValuesType() == DocValuesType.NONE;
          if (updateGlobal) {
            globalFieldNumbers.setDocValuesType(fi.number, name, docValues);
          }
          fi.setDocValuesType(docValues);
        }
      }
      return fi;
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
fieldInfo会从成员变量byName中获取是否已经添加了对应域的FieldInfo，如果没有添加，则通过addOrGet函数从全局获得域对应的number，然后创建FieldInfo添加到byName中，如果已经添加过了，就调用FieldInfo的update函数和原来添加的FieldInfo的值进行合并。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeFieldInfos->FieldInfos.Builder::add->addOrUpdateInternal->FieldInfo::update

  void update(boolean storeTermVector, boolean omitNorms, boolean storePayloads, IndexOptions indexOptions,
              int dimensionCount, int dimensionNumBytes) {
    if (this.indexOptions != indexOptions) {
      if (this.indexOptions == IndexOptions.NONE) {
        this.indexOptions = indexOptions;
      } else if (indexOptions != IndexOptions.NONE) {
        this.indexOptions = this.indexOptions.compareTo(indexOptions) < 0 ? this.indexOptions : indexOptions;
      }
    }

    if (this.pointDimensionCount == 0 && dimensionCount != 0) {
      this.pointDimensionCount = dimensionCount;
      this.pointNumBytes = dimensionNumBytes;
    }
    
    if (this.indexOptions != IndexOptions.NONE) {
      this.storeTermVector |= storeTermVector;
      this.storePayloads |= storePayloads;
    
      if (indexOptions != IndexOptions.NONE && this.omitNorms != omitNorms) {
        this.omitNorms = true;
      }
    }
    if (this.indexOptions == IndexOptions.NONE || this.indexOptions.compareTo(IndexOptions.DOCS_AND_FREQS_AND_POSITIONS) < 0) {
      this.storePayloads = false;
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
update函数可以看到域相互合并的规则，这里就不详细介绍了。

2.2.2 mergeFields
mergeFieldInfos函数创建了新段的域信息，mergeFields函数根据该域信息对所有段的.fdt和.fdx文件进行合并。
MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeFields

  private int mergeFields() throws IOException {
    try (StoredFieldsWriter fieldsWriter = codec.storedFieldsFormat().fieldsWriter(directory, mergeState.segmentInfo, context)) {
      return fieldsWriter.merge(mergeState);
    }
  }
1
2
3
4
5
fieldsWriter函数最终返回CompressingStoredFieldsWriter，然后调用其merge函数开始合并。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeFields->CompressingStoredFieldsWriter::merge

  public int merge(MergeState mergeState) throws IOException {
    int docCount = 0;
    int numReaders = mergeState.maxDocs.length;

    MatchingReaders matching = new MatchingReaders(mergeState);
    
    for (int readerIndex=0;readerIndex<numReaders;readerIndex++) {
      MergeVisitor visitor = new MergeVisitor(mergeState, readerIndex);
      CompressingStoredFieldsReader matchingFieldsReader = null;
      if (matching.matchingReaders[readerIndex]) {
        final StoredFieldsReader fieldsReader = mergeState.storedFieldsReaders[readerIndex];
        matchingFieldsReader = (CompressingStoredFieldsReader) fieldsReader;
      }
    
      final int maxDoc = mergeState.maxDocs[readerIndex];
      final Bits liveDocs = mergeState.liveDocs[readerIndex];
    
      if (matchingFieldsReader == null || matchingFieldsReader.getVersion() != VERSION_CURRENT || BULK_MERGE_ENABLED == false) {
    
        ...
    
      } else if (matchingFieldsReader.getCompressionMode() == compressionMode && 
                 matchingFieldsReader.getChunkSize() == chunkSize && 
                 matchingFieldsReader.getPackedIntsVersion() == PackedInts.VERSION_CURRENT &&
                 liveDocs == null &&
                 !tooDirty(matchingFieldsReader)) {  
        matchingFieldsReader.checkIntegrity();
        IndexInput rawDocs = matchingFieldsReader.getFieldsStream();
        CompressingStoredFieldsIndexReader index = matchingFieldsReader.getIndexReader();
        rawDocs.seek(index.getStartPointer(0));
        int docID = 0;
        while (docID < maxDoc) {
          int base = rawDocs.readVInt();
          int code = rawDocs.readVInt();
          int bufferedDocs = code >>> 1;
    
          indexWriter.writeIndex(bufferedDocs, fieldsStream.getFilePointer());
          fieldsStream.writeVInt(docBase);
          fieldsStream.writeVInt(code);
          docID += bufferedDocs;
          docBase += bufferedDocs;
          docCount += bufferedDocs;
          final long end;
          if (docID == maxDoc) {
            end = matchingFieldsReader.getMaxPointer();
          } else {
            end = index.getStartPointer(docID);
          }
          fieldsStream.copyBytes(rawDocs, end - rawDocs.getFilePointer());
        }               
        numChunks += matchingFieldsReader.getNumChunks();
        numDirtyChunks += matchingFieldsReader.getNumDirtyChunks();
      } else {
        matchingFieldsReader.checkIntegrity();
        for (int docID = 0; docID < maxDoc; docID++) {
          if (liveDocs != null && liveDocs.get(docID) == false) {
            continue;
          }
          SerializedDocument doc = matchingFieldsReader.document(docID);
          bufferedDocs.copyBytes(doc.in, doc.length);
          numStoredFieldsInDoc = doc.numStoredFields;
          finishDocument();
          ++docCount;
        }
      }
    }
    finish(mergeState.mergeFieldInfos, docCount);
    return docCount;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
首先创建MatchingReaders，MatchingReaders的构造函数标记了合并后段的域是否和现有段的域相同。然后遍历所有段，根据MatchingReaders的标记matchingReaders，如果一致，就直接从MergeState的成员变量storedFieldsReaders里获得对应段的CompressingStoredFieldsReader。
接下来从MergeState中获得最大的文档数maxDoc和删除的信息liveDocs。
再往下的第一个if语句表示要合并的段和最终的段在域上结构不一致，此时要改写document文档，这部分代码省略。
第二个if语句表示该段没有要删除的数据，即liveDocs为null。此时，首先通过checkIntegrity函数确认对应段的.fdt文件的合法性，然后获得.fdt文件的数据流rawDocs和索引信息index，初始化数据流的指针。成员变量indexWriter用于向新段的.fdx文件写入数据，fieldsStream用于向新段的.fdt文件的写入数据。接着根据rawDocs的原有数据向新的.fdx和.fdt文件写入数据。其中bufferedDocs代表一次性读取的文档数，end是计算的下一组文档的文件起始指针，也就是上一组文档的结束指针，然后更新.fdx索引文件，更新.fdt文件中的文档号docBase，最后通过copyBytes函数拷贝.fdt文件的文档数据。
第三个if语句表示合并的段和最终的段在结构上一直，但是有要删除的数据。此时首先通过checkIntegrity函数继续检查要合并的段的.fdt文件的合法性，然后遍历该段所有文档，如果该文档的文档号docID在liveDocs的对应比特上为0，则表示该文档已经被标记为删除，继续循环，如果找到未被删除的文档，就通过CompressingStoredFieldsReader的document函数获得文档对象SerializedDocument，将数据拷贝到bufferedDocs中，存储该文档的域的数量写入numStoredFieldsInDoc中，然后调用finishDocument函数开始合并。

当处理完所有的段后，调用finish函数向.fdt和.fdx的文件尾部写入相应信息，至此，最新段的.fdt和.fdx文件就诞生了。最后返回新段的文件数。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeFields->CompressingStoredFieldsWriter::merge->MatchingReaders::MatchingReaders

  MatchingReaders(MergeState mergeState) {
    int numReaders = mergeState.maxDocs.length;
    int matchedCount = 0;
    matchingReaders = new boolean[numReaders];

    nextReader:
    for (int i = 0; i < numReaders; i++) {
      for (FieldInfo fi : mergeState.fieldInfos[i]) {
        FieldInfo other = mergeState.mergeFieldInfos.fieldInfo(fi.number);
        if (other == null || !other.name.equals(fi.name)) {
          continue nextReader;
        }
      }
      matchingReaders[i] = true;
      matchedCount++;
    }
    this.count = matchedCount;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
MatchingReaders的构造函数遍历所有的段，再遍历该段的所有域，通过FieldInfo的number判断该域和即将合并后的域是否一致，如果某个段的所有域都和最新段的域一致，就将matchingReaders的相应位置标记为true。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeFields->CompressingStoredFieldsWriter::merge->finishDocument

  public void finishDocument() throws IOException {
    if (numBufferedDocs == this.numStoredFields.length) {
      final int newLength = ArrayUtil.oversize(numBufferedDocs + 1, 4);
      this.numStoredFields = Arrays.copyOf(this.numStoredFields, newLength);
      endOffsets = Arrays.copyOf(endOffsets, newLength);
    }
    this.numStoredFields[numBufferedDocs] = numStoredFieldsInDoc;
    numStoredFieldsInDoc = 0;
    endOffsets[numBufferedDocs] = bufferedDocs.length;
    ++numBufferedDocs;
    if (triggerFlush()) {
      flush();
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
finishDocument函数将当前文档的域的数量存储在numStoredFields中，将缓存中的指针存储在endOffsets中，在必要的时候通过flush函数将数据写入新的.fdx和.fdt文件。

2.2.3 mergeTerms
mergeTerms主要用于生成合并后段的.doc、.pos、.pay、.tim、.tip文件。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeTerms

  private void mergeTerms(SegmentWriteState segmentWriteState) throws IOException {
    try (FieldsConsumer consumer = codec.postingsFormat().fieldsConsumer(segmentWriteState)) {
      consumer.merge(mergeState);
    }
  }
1
2
3
4
5
consumer最终为PerFieldPostingsFormat.FieldsWriter，然后调用其merge函数执行合并操作。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeTerms->FieldsWriter::merge

  public void merge(MergeState mergeState) throws IOException {
    final List<Fields> fields = new ArrayList<>();
    final List<ReaderSlice> slices = new ArrayList<>();
    int docBase = 0;

    for(int readerIndex=0;readerIndex<mergeState.fieldsProducers.length;readerIndex++) {
      final FieldsProducer f = mergeState.fieldsProducers[readerIndex];
    
      final int maxDoc = mergeState.maxDocs[readerIndex];
      f.checkIntegrity();
      slices.add(new ReaderSlice(docBase, maxDoc, readerIndex));
      fields.add(f);
      docBase += maxDoc;
    }
    
    Fields mergedFields = new MappedMultiFields(mergeState, new MultiFields(fields.toArray(Fields.EMPTY_ARRAY), slices.toArray(ReaderSlice.EMPTY_ARRAY)));
    write(mergedFields);
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
merge函数遍历每个段的FieldsProducer，通过checkIntegrity函数检查.tim、.pos、.doc、.pay文件的完整性，创建的ReaderSlice封装了每个段的基本信息，最后创建MappedMultiFields，调用write函数继续执行。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeTerms->FieldsWriter::merge->write

    public void write(Fields fields) throws IOException {
    
      Map<PostingsFormat,FieldsGroup> formatToGroups = new HashMap<>();
      Map<String,Integer> suffixes = new HashMap<>();
    
      for(String field : fields) {
        FieldInfo fieldInfo = writeState.fieldInfos.fieldInfo(field);
        final PostingsFormat format = getPostingsFormatForField(field);
        String formatName = format.getName();
    
        FieldsGroup group = formatToGroups.get(format);
        if (group == null) {
          Integer suffix = suffixes.get(formatName);
          if (suffix == null) {
            suffix = 0;
          } else {
            suffix = suffix + 1;
          }
          suffixes.put(formatName, suffix);
          String segmentSuffix = getFullSegmentSuffix(field, writeState.segmentSuffix, getSuffix(formatName, Integer.toString(suffix)));
          group = new FieldsGroup();
          group.state = new SegmentWriteState(writeState, segmentSuffix);
          group.suffix = suffix;
          formatToGroups.put(format, group);
        }
    
        group.fields.add(field);
        String previousValue = fieldInfo.putAttribute(PER_FIELD_FORMAT_KEY, formatName);
        previousValue = fieldInfo.putAttribute(PER_FIELD_SUFFIX_KEY, Integer.toString(group.suffix));
      }
    
      for(Map.Entry<PostingsFormat,FieldsGroup> ent : formatToGroups.entrySet()) {
        PostingsFormat format = ent.getKey();
        final FieldsGroup group = ent.getValue();
    
        Fields maskedFields = new FilterFields(fields) {
            @Override
            public Iterator<String> iterator() {
              return group.fields.iterator();
            }
        };
    
        FieldsConsumer consumer = format.fieldsConsumer(group.state);
        consumer.write(maskedFields);
      }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
write函数首先遍历所有的域，根据域名，获取新段对应的FieldInfo，getPostingsFormatForField函数默认返回Lucene50PostingsFormat，再往下创建的FieldsGroup是对不同的域相同的存储格式的封装，然后将其添加到formatToGroups集合中。
再往下遍历formatToGroups，fieldsConsumer函数根据SegmentWriteState的信息和存储格式format，生成对应的.doc、.pos、.pay、.tim、.tip文件，创建输入流并初始化，最后调用返回的BlockTreeTermsWriter的write函数将原有数据写入这些新的文件。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeTerms->FieldsWriter::merge->write->BlockTreeTermsWriter::write

  public void write(Fields fields) throws IOException {

    String lastField = null;
    for(String field : fields) {
      lastField = field;
      Terms terms = fields.terms(field);
      FieldInfo fieldInfo = fieldInfos.fieldInfo(field);
    
      TermsEnum termsEnum = terms.iterator();
      TermsWriter termsWriter = new TermsWriter(fieldInfos.fieldInfo(field));
      while (true) {
        BytesRef term = termsEnum.next();
        if (term == null) break;
        termsWriter.write(term, termsEnum, null);
      }
      termsWriter.finish();
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
write函数遍历所有的域，对每个域通过terms函数返回一个MultiTerms包含之前所有域的FieldReader。
接下来获得对应域的FieldInfo信息。再往下通过一个MappedMultiTerms的iterator函数最终返回一个MappedMultiTermsEnum，内部封装了每个段的FieldReader。
然后创建TermsWriter，进入循环，获得倒排列表中的每个Term，然后调用TermsWriter的write和finish函数将这些信息逐个写入.doc、.pos、.pay、.tip和.tim文件中，这部分代码和《lucene源码分析—9》中分析的类似，不同点是要对所有段的FieldReader读出的字节进行排序，这里就不往下看了。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeTerms->FieldsWriter::merge->write->BlockTreeTermsWriter::write->FilterFields::terms

  public Terms terms(String field) throws IOException {
    return in.terms(field);
  }

  public Terms terms(String field) throws IOException {
    MultiTerms terms = (MultiTerms) in.terms(field);
    if (terms == null) {
      return null;
    } else {
      return new MappedMultiTerms(field, mergeState, terms);
    }
  }

  public Terms terms(String field) throws IOException {
    Terms result = terms.get(field);
    if (result != null)
      return result;

    final List<Terms> subs2 = new ArrayList<>();
    final List<ReaderSlice> slices2 = new ArrayList<>();
    
    for(int i=0;i<subs.length;i++) {
      final Terms terms = subs[i].terms(field);
      if (terms != null) {
        subs2.add(terms);
        slices2.add(subSlices[i]);
      }
    }
    
    result = new MultiTerms(subs2.toArray(Terms.EMPTY_ARRAY), slices2.toArray(ReaderSlice.EMPTY_ARRAY));
    terms.put(field, result);
    
    return result;
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
terms函数遍历所有段subs，获得每个段对应域field的FieldReader，添加到subs2中，再将每个段对应的基本信息添加到slices2中，封装成MultiTerms，添加到缓存terms中，最后返回MultiTerms。
然后将返回的MultiTerms和合并信息mergeState封装成MappedMultiTerms并返回。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeTerms->FieldsWriter::merge->write->BlockTreeTermsWriter::write->MappedMultiTerms::iterator

  public TermsEnum iterator() throws IOException {
    TermsEnum iterator = in.iterator();
    return new MappedMultiTermsEnum(field, mergeState, (MultiTermsEnum) iterator);
  }

  public TermsEnum iterator() throws IOException {

    final List<MultiTermsEnum.TermsEnumIndex> termsEnums = new ArrayList<>();
    for(int i=0;i<subs.length;i++) {
      final TermsEnum termsEnum = subs[i].iterator();
      if (termsEnum != null) {
        termsEnums.add(new MultiTermsEnum.TermsEnumIndex(termsEnum, i));
      }
    }
    
    return new MultiTermsEnum(subSlices).reset(termsEnums.toArray(MultiTermsEnum.TermsEnumIndex.EMPTY_ARRAY));
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
MappedMultiTerms的iterator函数进而调用MultiTerms的iterator函数，该函数遍历所有段，调用每个段的FieldReader的iterator函数创建SegmentTermsEnum并返回，然后封装成TermsEnumIndex添加到termsEnums列表中，最后创建MultiTermsEnum并通过reset函数封装这些信息。

2.2.4 mergeNorms
mergeNorms相对前几个文件的合并操作要简单得多，其最终向.nvd和.nvm文件写入数据。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeNorms

  private void mergeNorms(SegmentWriteState segmentWriteState) throws IOException {
    try (NormsConsumer consumer = codec.normsFormat().normsConsumer(segmentWriteState)) {
      consumer.merge(mergeState);
    }
  }
1
2
3
4
5
consumer最终获得Lucene53NormsConsumer，调用其merge函数继续执行。

MergeThread::run->doMerge->IndexWriter::merge->mergeMiddle->SegmentMerger::merge->mergeNorms->Lucene53NormsConsumer::merge

  public void merge(MergeState mergeState) throws IOException {
    for(NormsProducer normsProducer : mergeState.normsProducers) {
        normsProducer.checkIntegrity();
    }
    for (FieldInfo mergeFieldInfo : mergeState.mergeFieldInfos) {
      if (mergeFieldInfo.hasNorms()) {
        List<NumericDocValues> toMerge = new ArrayList<>();
        for (int i=0;i<mergeState.normsProducers.length;i++) {
          NormsProducer normsProducer = mergeState.normsProducers[i];
          NumericDocValues norms = null;
          if (normsProducer != null) {
            FieldInfo fieldInfo = mergeState.fieldInfos[i].fieldInfo(mergeFieldInfo.name);
            if (fieldInfo != null && fieldInfo.hasNorms()) {
              norms = normsProducer.getNorms(fieldInfo);
            }
          }
          toMerge.add(norms);
        }
        mergeNormsField(mergeFieldInfo, mergeState, toMerge);
      }
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
这里首先对每个段的Lucene53NormsProducer做文件检查，然后外循环遍历所有域，内循环遍历所有段，将每个段的norm信息添加到toMerge列表中，最后调用mergeNormsField写入新段的.nvd和.nvm文件中去，具体的写这里就不往下看了。

2.3 mergeFinish
mergeFinish完成最后的收尾工作。

MergeThread::run->doMerge->IndexWriter::merge->mergeFinish

  final synchronized void mergeFinish(MergePolicy.OneMerge merge) {

    if (merge.registerDone) {
      final List<SegmentCommitInfo> sourceSegments = merge.segments;
      for (SegmentCommitInfo info : sourceSegments) {
        mergingSegments.remove(info);
      }
      merge.registerDone = false;
    }
    runningMerges.remove(merge);
  }
1
2
3
4
5
6
7
8
9
10
11
mergeFinish函数将每个段对应的SegmentCommitInfo从mergingSegments集合中删除，再从runningMerges移除前面初始化构造的OneMerge，表示合并完成。
————————————————
版权声明：本文为CSDN博主「二侠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/conansonic/article/details/53325533