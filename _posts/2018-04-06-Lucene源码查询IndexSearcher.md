---
layout: post
categories: [Lucene]
description: none
keywords: Lucene
---
# Lucene源码查询IndexSearcher
Lucene的搜索入口是通过创建一个IndexSearcher对象来实现的。IndexSearcher是Lucene用于执行搜索操作的主要组件。它负责在索引中执行查询，并返回与查询条件匹配的文档。

## IndexSearcher作用
- 执行搜索
  IndexSearcher提供了多种搜索方法，可以根据不同的查询条件执行搜索操作。你可以使用Query对象来构建查询条件，然后使用IndexSearcher的search()方法来执行搜索。
- 匹配文档
  IndexSearcher根据查询条件在索引中匹配文档。它会检查索引中的词项、文档频率和其他统计信息，以确定哪些文档与查询条件匹配。
- 返回搜索结果
  一旦搜索完成，IndexSearcher将返回一个包含与查询匹配的文档的结果集。每个匹配的文档通常表示为一个包含文档ID和得分的ScoreDoc对象。
- 计算文档得分
  IndexSearcher会为每个匹配的文档计算一个得分，该得分表示文档与查询的相关性。得分可以用于对搜索结果进行排序或评估文档的相关性。
- 支持多线程
  IndexSearcher可以在多个线程之间共享，并发执行搜索请求。它内部使用了一些线程安全的数据结构和算法，以提供高性能的并发搜索能力。

需要注意的是，IndexSearcher本身并不负责索引的创建或更新，它只用于搜索已经建立好的索引。如果需要创建索引或更新索引，请使用Lucene的IndexWriter组件。

总之，Lucene的IndexSearcher是负责执行搜索操作、匹配文档和返回搜索结果的重要组件。

## 源码实战

### IndexSearcher构造函数
IndexSearcher是搜索的入口，IndexSearcher类的源码位于Lucene的核心代码库中。首先，我们来看一下IndexSearcher类的构造函数：
```
  public IndexSearcher(IndexReader r) {
    this(r, null);
  }
  
  public IndexSearcher(IndexReader r, Executor executor) {
    this(r.getContext(), executor);
  }
```
在Lucene的IndexSearcher类中，存在一个构造函数 IndexSearcher(IndexReader r)，它会调用另一个构造函数 IndexSearcher(IndexReader r, IndexReaderContext context)。该构造函数主要用于创建一个IndexSearcher对象，并初始化相关的成员变量。

```
  public IndexSearcher(IndexReaderContext context, Executor executor) {
    assert context.isTopLevel: "IndexSearcher's ReaderContext must be topLevel for reader" + context.reader();
    reader = context.reader();
    this.executor = executor;
    this.readerContext = context;
    leafContexts = context.leaves();
    this.leafSlices = executor == null ? null : slices(leafContexts);
  }
```
这是Lucene中IndexSearcher类的一个构造函数。该构造函数接受两个参数：
- context：一个IndexReaderContext对象，表示索引的上下文信息。注意，在构造IndexSearcher时，必须提供一个顶层的ReaderContext。
- executor：一个Executor对象，用于执行多线程查询（可选参数）。

该构造函数的作用是创建IndexSearcher对象并初始化相关成员变量，以便进行索引搜索。通过提供合适的IndexReaderContext和Executor，可以实现多线程查询的功能。

### leafContexts作用
leafContexts是CompositeReaderContext中的leaves成员变量，是一个LeafReaderContext列表，每个LeafReaderContext封装了每个段的SegmentReader，SegmentReader可以读取每个段的所有信息和数据。

leafContexts 是一个列表，用于存储 LeafReaderContext 对象，即叶子阅读器上下文。在 Lucene 的索引读取过程中，leafContexts 用于保存所有的叶子阅读器上下文。

每个 LeafReaderContext 对象表示一个叶子阅读器（LeafReader）及其相关的信息，例如叶子阅读器在整个索引中的顺序、文档基数等。LeafReader 是 Lucene 中的最小读取单位，代表了索引的一部分。当索引是由多个段（segments）组成时，每个段通常对应一个 LeafReader。

通过保存所有的叶子阅读器上下文到 leafContexts 列表中，Lucene 可以更方便地进行搜索操作。例如，当进行搜索时，可以通过遍历 leafContexts 列表，逐个对每个叶子阅读器进行搜索并合并结果。这种方式允许 Lucene 在并行或分布式环境下更高效地利用多个叶子阅读器进行搜索。

总结来说，leafContexts 列表的作用是保存所有的叶子阅读器上下文，以便于进行索引搜索和其他相关操作。

### IndexSearcher接口
IndexSearcher 是 Lucene 中用于执行搜索操作的主要类之一，它提供了一系列 API 接口来支持不同类型的搜索操作。

以下是 IndexSearcher 常用的 API 接口列表：
- search(Query query, int n)：执行搜索操作，返回前 n 个搜索结果。
- search(Query query, Filter filter, int n)：执行带有过滤器（Filter）的搜索操作，返回前 n 个搜索结果。
- searchAfter(ScoreDoc after, Query query, int n)：从指定的 ScoreDoc 开始，执行搜索操作，返回前 n 个搜索结果。可以使用该方法实现分页搜索。
- explain(Query query, int doc)：解释指定文档（doc）的搜索评分值（score）。
- rewrite(Query query)：对查询（Query）进行重写，以便更好地优化搜索操作。例如，将布尔运算符查询转换为 dismax 或者 edismax 查询等。
- doc(int docID)：获取指定文档（doc）的内容和元数据信息。
- getIndexReader()：获取当前 IndexSearcher 对象所使用的索引阅读器（IndexReader）。
- setSimilarity(Similarity similarity)：设置搜索评分模型（Similarity）。也可以在构建 IndexSearcher 对象时指定搜索评分模型。
- count(Query query)：计算满足查询条件的文档总数。
- count(Weight weight)：计算满足权重参数指定的查询条件的文档总数。
- search(Weight weight, int n)：执行带有权重参数（Weight）的搜索操作，返回前 n 个搜索结果。

除了以上列举的 API 接口外，IndexSearcher 还提供了许多其他有用的方法，例如：
- createNormalizedWeight(Query query)：创建规范化权重（Normalized Weight），用于评估查询的相关性。
- explain(Query query, int doc, Explanation explain)：解释并调试输入文档的得分。
通过这些 API 接口，可以在 Lucene 中实现各种基于索引的搜索操作，如基本搜索、短语搜索、加权搜索等。需要注意的是，不同类型的搜索操作需要使用不同的查询（Query）类型和搜索评分模型（Similarity）。为了获得更好的搜索性能和结果精度，建议仔细了解查询类型和搜索评分模型的选择和使用。

前两个search属于简单搜索一类的，接下来两个api是带Collector的，最后三个api是带排序的
```
public TopDocs search(Query query, int n)  throws IOException;
public TopDocs search(Query query, Filter filter, int n) throws IOException;
public void search(Query query, Filter filter, Collector results) throws IOException;
public void search(Query query, Collector results) throws IOException;
public TopFieldDocs search(Query query, Filter filter, int n,
                             Sort sort) throws IOException;
public TopFieldDocs search(Query query, Filter filter, int n,
                             Sort sort, boolean doDocScores, boolean doMaxScore) throws IOException;
public TopFieldDocs search(Query query, int n,
                             Sort sort) throws IOException;
```
以及一堆用于分页或深度查询用的searchAfter。
```
public TopDocs searchAfter(ScoreDoc after, Query query, int n) throws IOException;
public TopDocs searchAfter(ScoreDoc after, Query query, Filter filter, int n) throws IOException;
public TopDocs searchAfter(ScoreDoc after, Query query, Filter filter, int n, Sort sort) throws IOException;
public TopDocs searchAfter(ScoreDoc after, Query query, int n, Sort sort) throws IOException;
public TopDocs searchAfter(ScoreDoc after, Query query, Filter filter, int n, Sort sort,
                             boolean doDocScores, boolean doMaxScore) throws IOException;
```







