---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码缓存
毫无疑问，ES是一个内存大户。作为一款使用 Java 开发的应用程序，Elasticsearch 需要从系统的物理内存中分配部分逻辑内存（堆）。这部分内存最大不超过物理 RAM 的一半，上限为 32GB，建议设置31G。建议的 Elasticsearch 内存通常不超过总可用内存的一半，这样另一半就可用于页缓存了。

JVM 在内存小于 32 GB 的时候会采用一个内存对象指针压缩技术，超过32G对象指针压缩技术就会切回普通对象，指针压缩会失效。超过 32 GB。浪费了内存，降低了 CPU 的性能，还要让 GC 应对大内存。
这个不仅仅是针对ES而言，任何基于JVM的语言（Java Scala等）都可以基于此考虑内存分配。

关于操作系统的页缓存（Page Cache），几乎是每个大数据应用都会使用的缓存，包括ElasticSearch/Kafka等等。基于Page Cache可以大大提高读写速度，缺点就是服务器断电，在Page Cache的数据就会丢失。

## Node查询缓存
Node级别的filter过滤器结果缓存，集群中的每个节点包含一个 Node Query Cache，作用域是Node实例，由该节点的所有 shard 共享，Cache 采用 LRU 算法进行置换。

### 什么情况下会产生NodeQueryCache
1）只有Filter下的子Query才能参与Cache。

2）不能参与Cache的Query有TermQuery/MatchAllDocsQuery/MatchNoDocsQuery/BooleanQuery/DisjunnctionMaxQuery，对应源码在UsageTrackingQueryCachingPolicy类。
```
  private static boolean shouldNeverCache(Query query) {
    if (query instanceof TermQuery) {
      // We do not bother caching term queries since they are already plenty fast.
      return true;
    }

    if (query instanceof MatchAllDocsQuery) {
      // MatchAllDocsQuery has an iterator that is faster than what a bit set could do.
      return true;
    }
    if (query instanceof MatchNoDocsQuery) {
      return true;
    }

    if (query instanceof BooleanQuery) {
      BooleanQuery bq = (BooleanQuery) query;
      if (bq.clauses().isEmpty()) {
        return true;
      }
    }

    if (query instanceof DisjunctionMaxQuery) {
      DisjunctionMaxQuery dmq = (DisjunctionMaxQuery) query;
      if (dmq.getDisjuncts().isEmpty()) {
        return true;
      }
    }
    return false;
}
```
3）MultiTermQuery/MultiTermQueryConstantScoreWrapper/TermInSetQuery/Point*Query的Query查询超过2次会被Cache，其它Query要5次，对应源码在UsageTrackingQueryCachingPolicy类。
```
 /**
   * For a given filter, return how many times it should appear in the history
   * before being cached. The default implementation returns 2 for filters that
   * need to evaluate against the entire index to build a {@link DocIdSetIterator},
   * like {@link MultiTermQuery}, point-based queries or {@link TermInSetQuery},
   * and 5 for other filters.
   */
  protected int minFrequencyToCache(Query query) {
    if (isCostly(query)) {
      return 2;
    } else {
      // default: cache after the filter has been seen 5 times
      int minFrequency = 5;
      if (query instanceof BooleanQuery
          || query instanceof DisjunctionMaxQuery) {
        // Say you keep reusing a boolean query that looks like "A OR B" and
        // never use the A and B queries out of that context. 5 times after it
        // has been used, we would cache both A, B and A OR B, which is
        // wasteful. So instead we cache compound queries a bit earlier so that
        // we would only cache "A OR B" in that case.
        minFrequency--;
      }
      return minFrequency;
    }
  }
--------------------------------------------
  static boolean isCostly(Query query) {
    // This does not measure the cost of iterating over the filter (for this we
    // already have the DocIdSetIterator#cost API) but the cost to build the
    // DocIdSet in the first place
    return query instanceof MultiTermQuery ||
        query instanceof MultiTermQueryConstantScoreWrapper ||
        query instanceof TermInSetQuery ||
        isPointQuery(query);
  }
```
4）默认每个Segment大于10000个doc或每个段的doc数大于总doc数的30%时才允许参与cache。

5）结果集比较大的Query在Cache时尽量增加使用周期以免频繁Cache构建DocIdset。

6）Segment被合并或者删除，那么也会清理掉对应的缓存。

7） 内存无法被GC。

## Cache相关配置
1）indices.queries.cache.size：为每个节点配置缓存的内存大小，默认是10%，支持两种格式，一种是百分数，占节点heap的百分比，另一种是精确的值，如512mb，这个参数是静态的配置后需要重启节点生效。

2）index.queries.cache.enabled：属于index级别的配置，用来控制是否启用缓存，默认是开启的。

缓存策略初始化在Node初始化的时候完成，Query Cache在Node层面，如果想把Cache基于每个Shard层面，需要把属性index.queries.cache.everything设置为true(默认false)，对应源码在Node类。
```
if (IndexModule.INDEX_QUERY_CACHE_EVERYTHING_SETTING.get(settings)) {
    cachingPolicy = new QueryCachingPolicy() {
        @Override
        public void onUse(Query query) {

        }
        @Override
        public boolean shouldCache(Query query) {
            return true;
        }
    };
} else {
    //Lucene类缓存策略
    cachingPolicy = new UsageTrackingQueryCachingPolicy();
}
```
## Shard查询缓存
针对 ES 的 query 请求，缓存各分片的查询结果，主要用于缓存聚合结果，采用LRU机制，当分片 refresh 后会失效。Shard Request Cache是分片级别的查询缓存，每个分片有自己的缓存。该缓存采用 LRU 机制，缓存的 key 是整个客户端请求，缓存内容为单个分片的查询结果。如果一个客户端请求被数据节点缓存了，下次查询的时候可以直接从缓存获取，无需对分片进行查询。Shard查询缓存的理念是对请求的完整响应进行缓存，不需要执行任何搜索，并且基本上可以立即返回响应 — 只要数据没有更改，以确保您不会返回任何过时数据！
```
final Key key =  new Key(cacheEntity, reader.getReaderCacheHelper().getKey(), cacheKey);
```
cacheEntity， 主要是 shard信息，代表该缓存是哪个 shard上的查询结果。
readerCacheKey， 主要用于区分不同的 IndexReader。 cacheKey， 主要是整个客户端请求的请求体（source）和请求参数（preference、indexRoutings、requestCache等）。由于客户端请求信息直接序列化为二进制作为缓存 key 的一部分，所以客户端请求的 json 顺序，聚合名称等变化都会导致 cache 无法命中。

## Cache失效策略
Request Cache是非常智能的，它能够保证和在近实时搜索中的非缓存查询结果一致。

Request Cache缓存失效是自动的，当索引refresh时就会失效，所以其生命周期是一个refresh_interval，也就是说在默认情况下Request Cache是每1秒钟失效一次（注意：分片在这段时间内确实有改变才会失效）。当一个文档被索引到该文档变成Searchable之前的这段时间内，不管是否有请求命中缓存该文档都不会被返回，正是因为如此ES才能保证在使用Request Cache的情况下执行的搜索和在非缓存近实时搜索的结果一致。

## 配置参数
index.requests.cache.enable：默认为true，启动RequestCache配置。
indices.requests.cache.size：RequestCache占用JVM的百分比，默认情况下是JVM堆的1%大小
indices.requests.cache.expire：配置过期时间，单位为分钟。

每次请求的时候使用query-string参数可以指定是否使用Cache（request_cache=true），这个会覆盖Index级别的设置。
```
GET /my_index/_search?request_cache=true
{
  "size": 0,
  "aggs": {
    "popular_colors": {
      "terms": {
        "field": "colors"
      }
    }
  }
}
```

## Request Cache作用
Request Cache 的主要作用是对聚合的缓存，聚合过程是实时计算，通常会消耗很多资源，缓存对聚合来说意义重大。

只有客户端查询请求中 size=0的情况下才会被缓存，其他不被缓存的条件还包括 scroll、设置了 profile属性，查询类型不是 QUERY_THEN_FETCH，以及设置了 requestCache=false等。另外一些存在不确定性的查询例如：范围查询带有now，由于它是毫秒级别的，缓存下来没有意义，类似的还有在脚本查询中使用了 Math.random() 等函数的查询也不会进行缓存。

该缓存使用LRU淘汰策略，内存无法被GC。

1）默认被开启，ES6.71版本。

2）RequestCache作用域为Node，在Node中的Shard共享这个Cache空间。。

3）RequestCache 是以查询的DSL整个串为key的，修改一个字符和顺序都需要重新生成Cache。

4）缓存失效是索引的refresh操作，也可以设置失效时间。

5）缓存的默认大小是JVM堆内存的1%，可以通过手动设置。

## Indexing Buffer
索引缓冲区，用于存储新索引的文档，当其达到阈值时，会触发 ES flush 将段落到磁盘上。Lucene文件写入Indexing Buffer中，这时文件不能搜索，单处罚refresh后，进入Page Cache后形成Segment就可以被搜索，这个也是ES NRT近实时搜索特性的原因。

这部分内存，节点上所有 shard 共享，空间是可以通过GC被反复利用的。

### indexing buffer配置
以下设置是静态的，必须在群集中的每个数据节点上进行配置：

indices.memory.index_buffer_size 接受百分比或字节大小的值。 它默认为10％，这意味着分配给一个节点的总堆栈的10％将用作所有分片共享的索引缓冲区大小。
indices.memory.min_index_buffer_size 如果将index_buffer_size指定为百分比，则可以使用此设置指定绝对最小值。 默认值为48mb。
ndices.memory.max_index_buffer_size 如果index_buffer_size被指定为百分比，则可以使用此设置来指定绝对最大值。 默认为限制

### indexing buffer注意事项
由于可以GC，有flush操作，不需要特殊的关注。Indexing Buffer是用来缓存新数据，当其满了或者refresh/flush interval到了，就会以segment file的形式写入到磁盘。 这个参数的默认值是10% heap size。根据经验，这个默认值也能够很好的工作，应对很大的索引吞吐量。 但有些用户认为这个buffer越大吞吐量越高，因此见过有用户将其设置为40%的。到了极端的情况，写入速度很高的时候，40%都被占用，导致OOM

### Fielddata
用于 text 字段分词或排序时获取字段值，基于 lucene 的 segment 构建，伴随 segment 生命周期常驻在堆内存中。

Fielddata Cache做了解即可，使用它的也非常的少了，基本可以用doc_value代替了，doc_value使用不需要全部载入内存。

### 页缓存
ES的读写操作是基于Lucene完成的，Lucene的读写操作是基于Segnemt。Segment具有不变性，一旦生成，无法修改。这样，把Segment加载到Page Cache进行读操作是很高效的，写操作也是先写入Page Cache，然后flush到磁盘，推荐Page Cache占整个内存的50%。ES的写操作只有增加和删除，并没有修改（修改API是先删除再新增），有写操作的时候Page Cache会有脏页，

Linux除了通过对read，write函数的调用实现数据的读写，还提供了一种方式，对文件数据进行读写，即利用mmap函数。

关于mmap和read的性能比较，这个stackoverflow高赞回答讲的很清晰。mmap与read相比的优势在于：少了一次内存从内核空间到用户空间的拷贝，对于需要常驻内存的文件和随机读取场景更适用；而反过来read的优势在于，调用开销少，不需要构建内存映射的页表等操作（包括unmap），对于少量顺序读取或者读取完就丢弃的场景更适用。

ES存储类型对应官网描述：fs simplefs niofs mmapfs hybridfs，默认FsDirectoryFactory。
```
//https://www.elastic.co/guide/en/elasticsearch/reference/7.5/index-modules-store.html
private static IndexStorePlugin.DirectoryFactory getDirectoryFactory(
        final IndexSettings indexSettings, final Map<String, IndexStorePlugin.DirectoryFactory> indexStoreFactories) {
    //通过indexSettings查找index.store.type
    final String storeType = indexSettings.getValue(INDEX_STORE_TYPE_SETTING);
    final Type type;
    //MMap FS 类型通过将文件映射到内存（mmap）将分片索引存储在文件系统上（映射到 Lucene MMapDirectory ）
    final Boolean allowMmap = NODE_STORE_ALLOW_MMAP.get(indexSettings.getNodeSettings());
    if (storeType.isEmpty() || Type.FS.getSettingsKey().equals(storeType)) {
        //type类型：hybridfs mmapfs  niofs simplefs fs
        type = defaultStoreType(allowMmap);
    } else {
        if (isBuiltinType(storeType)) {
            type = Type.fromSettingsKey(storeType);
        } else {
            type = null;
        }
    }
    if (allowMmap == false && (type == Type.MMAPFS || type == Type.HYBRIDFS)) {
        throw new IllegalArgumentException("store type [" + storeType + "] is not allowed because mmap is disabled");
    }
    final IndexStorePlugin.DirectoryFactory factory;
    if (storeType.isEmpty() || isBuiltinType(storeType)) {
        //默认FsDirectoryFactory
        factory = DEFAULT_DIRECTORY_FACTORY;
    } else {
        factory = indexStoreFactories.get(storeType);
        if (factory == null) {
            throw new IllegalArgumentException("Unknown store type [" + storeType + "]");
        }
    }
    return factory;
}
------------------------------------------------------
//mmapfs使用在Windows的64bit系统上，simplefs使用在windows的32bit系统上，
// 除此之外默认是用(hybrid niofs 和 mmapfs)
public static Type defaultStoreType(final boolean allowMmap) {
    if (allowMmap && Constants.JRE_IS_64BIT && MMapDirectory.UNMAP_SUPPORTED) {
        return Type.HYBRIDFS;
    } else if (Constants.WINDOWS) {
        return Type.SIMPLEFS;
    } else {
        return Type.NIOFS;
    }
}
```
niofs：通过 NIO 的方式读取文件内容，基于 Java 提供的 FileChannel::read 方法读取数据，支持多线程读取同一个文件，按需读取，需要注意的是 niofs 模式下读取到的内容在系统层面也会进 page cache。
mmapfs：通过mmap读写文件，会比常规文件读写方式减少一次内存拷贝的过程，因此对于命中 page cache 的文件读取会非常快，该模式即零拷贝的一种实现（可减少内核态和用户态间的数据拷贝）。不过 mmap 系统调用在内核层面会产生预读，对于 .fdt 这类文件，预读读到的内容后续命中的概率极低，还容易引起page cache的争用，进而产生频繁的缺页中断，相关问题可参考https://github.com/elastic/elas

（1）文件读取

对于nvd、dvd、tim等索引文件，由于查询频率很高，因此使用mmap常驻内存，对于fdx、fdm、nvm等索引元数据文件，读完需要的内容就可以丢弃，则应该使用read来读，另外一种上面没有提到的场景是fdt文件，由于存放原始数据，磁盘占用较大，全部使用mmap加载到内存中会导致页缓存抖动，真正需要常驻内存的索引文件会被换出页缓存，会导致性能劣化，因此也需要使用read。

ES默认存储类型HybridDirectory类，使用了代理模式，被代理的就是MmapDirectory，父类是NIOFSDirectory。当以下方法返回true时使用mmap，返回false时使用read：
```
boolean useDelegate(String name) {
    String extension = FileSwitchDirectory.getExtension(name);
    switch(extension) {
        //定义文件后缀都使用mmap，其他情况下使用read
        // We are mmapping norms, docvalues as well as term dictionaries, all other files are served through NIOFS
        // this provides good random access performance and does not lead to page cache thrashing.
        case "nvd":
        case "dvd":
        case "tim":
        case "tip":
        case "cfs":
            return true;
        default:
            return false;
    }
}
```
（2）文件写入

假设使用mmap来写入，首先要指定好索引文件做映射，需要知道索引的大小，但是在实际写入之前，是没办法知道写入文件大小的（Lucene一般都是通过try-with-resources打开一个IndexOutput，然后开始写入）。使用mmap很难完成这样的操作，而write调用则不受这个限制。

Lucene操作对Lucene文件的写入有具体描述，写入使用IndexWriter类完成。实际上Lucene的write是带buffer的，类似于fwrite，因此也不会触发过多的中断。mmap和fwrite相比，一个是预先映射好内存空间，另一个是一个buffer一个buffer写入Page Cache。对于已知大小的大文件写入，mmap应该会更快一些，但是Lucene的segment是比较小的，从性能上讲，mmap也没有优势。

ES对Lucene文件的读取，nvd、dvd、tim等索引文件，由于查询频率很高，因此使用mmap常驻内存，对于fdx、fdm、nvm等索引元数据文件，读完需要的内容就可以丢弃，则应该使用read来读。

ES对Lucene文件的写入，使用的是Linux系统的write方法，并没有基于mmap实现零拷贝。

## 其他
（1）Nested Cache

深入理解Elasticsearch中的缓存——Nested Cache提到Nested嵌套类型的Cache。不建议在ES中使用嵌套类型。Nested类型查询会慢几倍，而Parent-child类型的查询会慢百倍！
```
In particular, joins should be avoided.nested can make queries several times slower andparent-childrelations can make queries hundreds of times slower. So if the same questions can be answered without joins by denormalizing documents, significant speedups can be expected.
```
nested无形中增加了索引量，如果不了解具体实现，将无法很好的进行文档划分和预估。ES限制了Field个数和nested对象的size，避免无限制的扩大
nested Query 整体性能慢，但比parent/child Query稍快。应从业务上尽可能的避免使用NestedQuery, 对于性能要求高的场景，应该直接禁止使用。

## （2）缓存预加载

缓存预加载到PageCache，这是一项需要专家来完成的设置，请小心使用这项设置，以确保页缓存不会持续出现“抖动”现象。

官网给的示例：
```
PUT /my_index
{
  "settings": {
    "index.store.preload": ["nvd", "dvd"]
  }
}
```
这里"nvd","dvd"是Lucene定义的文件格式，如果Luene文件格式及其内存读写不是很熟悉，这个属性建议不要设置。


