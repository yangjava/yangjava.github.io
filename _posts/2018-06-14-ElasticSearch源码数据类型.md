---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码数据类型
ES的底层是Lucene，数据的存储与查询依赖Lucene。有大佬举例，如果ElasticSearch是一个汽车，那么Lucene就是发动机。Lucene是个工具包提供了底层的存储和查询，ES在工具包上提供了分布式的功能，本文就数据类型进行简单分析。

## Lucene
### FieldType
Lucene的FieldType描述Field的属性，FieldType源码有16个属性。
```
//是否存储字段，默认为false。如果为false，则lucene不会保存这个字段的值，而搜索结果中返回的文档只会包含保存了的字段。
private boolean stored;
//是否analyze（分词），默认为true.在lucene中只有TextField这一个字段需要做分词。
private boolean tokenized = true;
//是否存储TermVector（如果是true，也不存储offset、position、payload信息），默认为false
// Term vector的用途主要有两个，一是关键词高亮，二是做文档间的相似度匹配（more-like-this）。
private boolean storeTermVectors;
//是否存储offset信息，默认为false。
private boolean storeTermVectorOffsets;
//是否存储position信息，默认为false。
private boolean storeTermVectorPositions;
//是否存储payload信息，默认为false
private boolean storeTermVectorPayloads;
// Norms是normalization的缩写，lucene允许每个文档的每个字段都存储一个normalization factor，
// 搜索时的相关性计算有关的一个系数。
// Norms的存储只占一个字节，但是每个文档的每个字段都会独立存储一份，且Norms数据会全部加载到内存。
// 所以若开启了Norms，会消耗额外的存储空间和内存。
private boolean omitNorms;
//Lucene提供倒排索引的5种可选参数,默认值为DOCS_AND_FREQS_AND_POSITIONS
// 用于选择该字段是否需要被索引，以及索引哪些内容。
private IndexOptions indexOptions = IndexOptions.NONE;
//该值设置为true之后，字段的各个属性就不允许再更改了，比如Field的TextField、StringField等子类都将该值设置为true了，因为他们已经将字段的各个属性定制好了
private boolean frozen;
//DocValue是Lucene 4.0引入的一个正向索引（docid到field的一个列存），大大优化了sorting、faceting或aggregation的效率。
// DocValues是一个强schema的存储结构，开启DocValues的字段必须拥有严格一致的类型
//指定字段的值指定以何种类型索引DocValue,正向索引就是DocValue，主要用于排序、聚集等操作。
private DocValuesType docValuesType = DocValuesType.NONE;
//dimensionCount: 维度（dimension）计数。Lucene支持多维数据的索引，采取特殊的索引来优化对多维数据的查询，
// 这类数据最典型的应用场景是地理位置索引，一般经纬度数据会采取这个索引方式
private int dimensionCount;
//Lucene 6.0开始，对于数值型都改用Point来组织。indexDimensionCount理解为是Point的维度，类似于数组的维度
private int indexDimensionCount;
//dimensionNumBytes是Point中每个值所使用的字节数，比如IntPoint和FloatPoint是4个字节，LongPoint和DoublePoint则是8个字节。
private int dimensionNumBytes;
//向量字段记录向量的维数
private int vectorDimension;
//向量相似度计算公式（欧式距离：EUCLIDEAN；点积：DOT_PRODUCT；余玄相似度：COSINE）
private VectorSimilarityFunction vectorSimilarityFunction = VectorSimilarityFunction.EUCLIDEAN;
//以key-value的形式给字段增加一些元数据信息，但注意这个key-value的map不是线程安全的
private Map<String, String> attributes;
```
其中倒排索引用于搜索和查询，存储的类型如下：
```
/** Not indexed */
NONE,
//倒排索引中只存储了包含该词的 文档id，没有词频、位置
DOCS,
//倒排索引中会存储 文档id、词频
DOCS_AND_FREQS,
//倒排索引中存储 文档id、词频、位置
DOCS_AND_FREQS_AND_POSITIONS,
//倒排索引中存储 文档id、词频、位置、偏移量
DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS,
```
DocValuesType核心作用是用于聚合排序功能，对应的类型如下：
```
//不开启docvalue时的状态
NONE,
//个数值类型的docvalue主要包括（int，long，float，double）
NUMERIC,
//二进制类型值对应不同的codes最大值可能超过32766字节
BINARY,
//有序增量字节存储，仅仅存储不同部分的值和偏移量指针，值必须小于等于32766字节
SORTED,
//存储数值类型的有序数组列表
SORTED_NUMERIC,
//可以存储多值域的docvalue值，但返回时，仅仅只能返回多值域的第一个docvalue 
SORTED_SET
```

## Lucene数据类型
以Lucene官方定义的TextField为例分析，TYPE_NOT_STORED 需要索引，分词，并不需要存储。
```
/** Indexed, tokenized, not stored. */
public static final FieldType TYPE_NOT_STORED = new FieldType();

/** Indexed, tokenized, stored. */
public static final FieldType TYPE_STORED = new FieldType();

static {
  TYPE_NOT_STORED.setIndexOptions(IndexOptions.DOCS_AND_FREQS_AND_POSITIONS);
  TYPE_NOT_STORED.setTokenized(true);
  TYPE_NOT_STORED.freeze();

  TYPE_STORED.setIndexOptions(IndexOptions.DOCS_AND_FREQS_AND_POSITIONS);
  TYPE_STORED.setTokenized(true);
  TYPE_STORED.setStored(true);
  TYPE_STORED.freeze();
}
```
在Lucene内自定义字段类型，都需要设置FieldType，根据不同业务设置不同的属性。除了Lucene官方定义的数据类型，用户也可以自定义数据类型（ES的数据类型，全部是自定义的，并没有使用Luene定义好的数据类型）。

## Lucene文件格式
以Norms为例，在FieldType对应的属性是private boolean omitNorms; 如果一个Field设置omitNorms属性为true，则需要生成.nvd .nvm文件。

以Pre-Document Values为例，在FieldType对应的属性是DocValuesType ，如果一个Field设置docValuesType 属性不为NONE，则会生成.dvd dvm文件。

在 FieldType设置不同的属性，需要生成不同的对应的文件。Lucene使用了不同的压缩类型及存储数据结构来存储这些数据（好吧，这些内容太多了，我还没看）。

小结：

Lucene的Field设置不同的FieldType属性，可以自定义为不同的字段，在写入Document的时候Segment会包含不同的文件类型。高效的数据存储，同时支撑了多种类型的查询是设计Lucene的字段类型的核心。一般情况下，Lucene官方提供的字段类型（例如：TextField，IntPoint等等）已经基本满足日常开发，不建议通过设置不同的FieldType属性自定义字段类型。

但是ElasticSearch的开发者并没有使用任何Lucene官方定义的字段类型，而是自定义了全部字段，设置不同FieldType属性定义不同的Field，这进一步简化了开发者使用ElasticSearch，使用ES而不是Lucene则更没必要自定义字段。

## ElasticSearch数据类型
ES数据类型总体分为两类：Meta-fields和Fields or properties。

## Meta-fields
直观查看Metadata fields元数据，对ES发起查询请求，返回结果（如下）对应的_index _type _id等都属于Metadata fields。
```
{
  "_index" : "test",
  "_type" : "_doc",
  "_id" : "2",
  "_version" : 2,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 2,
  "_primary_term" : 1
}
```
在 7.6.0 版本之前，元数据直接存放在每个节点磁盘上，且在专有 master 节点上每个索引一个元数据目录。在 7.6.0 版本之后，ES 将元数据放到节点本地独立的 lucene 索引中保存。

在源码中，Metadata fields对应的属性，都需要继承MetadataFieldMapper类型。

## （1）fields元数据

_index ；_type（6.0版本已经废弃，7.0版本移除）； _id。这三个字段能够完整的查询ElasticSearch索引中一个Field的值。

_index元数据对应IndexFieldMapper类，对应的默认属性是不设置倒排索引，不分词，不存储，Norms（Document和Field的相关系数）设置为true，索引和搜索分析用Lucene.KEYWORD_ANALYZER类型（KeywordAnalyzer类），并且属性不允许修改属性。从属性看在当前ES版本，_index字段并没有存储在Lucene索引。
```
public static final String NAME = "_index";

public static final String CONTENT_TYPE = "_index";

public static class Defaults {
    public static final String NAME = IndexFieldMapper.NAME;

    public static final MappedFieldType FIELD_TYPE = new IndexFieldType();

    static {
        FIELD_TYPE.setIndexOptions(IndexOptions.NONE);
        FIELD_TYPE.setTokenized(false);
        FIELD_TYPE.setStored(false);
        FIELD_TYPE.setOmitNorms(true);
        FIELD_TYPE.setIndexAnalyzer(Lucene.KEYWORD_ANALYZER);
        FIELD_TYPE.setSearchAnalyzer(Lucene.KEYWORD_ANALYZER);
        FIELD_TYPE.setName(NAME);
        FIELD_TYPE.freeze();
    }
}
```
搜索类型支持：existsQuery/termQuery/termsQuery/prefixQuery/wildcardQuery(通配符)。如果在_index上作RangeQuery，肯定是要报错的。
```
public Query existsQuery(QueryShardContext context) {
    return new MatchAllDocsQuery();
}

@Override
public Query termQuery(Object value, @Nullable QueryShardContext context) {

}

@Override
public Query termsQuery(List values, QueryShardContext context) {

}

@Override
public Query prefixQuery(String value,
                         @Nullable MultiTermQuery.RewriteMethod method,
                         QueryShardContext context) {
}

@Override
public Query wildcardQuery(String value,
                           @Nullable MultiTermQuery.RewriteMethod method,
                           QueryShardContext context) {
}
```
（2）Document元数据

_source表示 Document的原文（Json形式）；

_size表示_source对应的size ;

以_source字段为例分析对应的类是SourceFieldMapper，对应的属性是不进行索引，但是进行存储。
```
public static final MappedFieldType FIELD_TYPE = new SourceFieldType();

static {
    FIELD_TYPE.setIndexOptions(IndexOptions.NONE); // not indexed
    FIELD_TYPE.setStored(true);
    FIELD_TYPE.setOmitNorms(true);
    FIELD_TYPE.setIndexAnalyzer(Lucene.KEYWORD_ANALYZER);
    FIELD_TYPE.setSearchAnalyzer(Lucene.KEYWORD_ANALYZER);
    FIELD_TYPE.setName(NAME);
    FIELD_TYPE.freeze();
}
```
每个Document都存储原文，其实是很浪费空间的，但是_source不进行存储，会造成update,update_by_query, andreindex等API；highLight(高亮)等特性无法使用。如果担心存储浪费空间，官方建议考虑compression level，而不是禁用_source存储。

SourceFieldMapper对Source字段不支持搜索，因此在_source字段搜索都会报错。
```
//不支持搜索
@Override
public Query existsQuery(QueryShardContext context) {
    throw new QueryShardException(context, "The _source field is not searchable");
}

//不支持搜索
@Override
public Query termQuery(Object value, QueryShardContext context) {
    throw new QueryShardException(context, "The _source field is not searchable");
}
```
（3）Indexing元数据

_field_names：从名称上能够理解应该是索引中Filed的name，该字段已经弃用，下个大版本会移除。

（4）Routing元数据

在索引中增加一个Doucument，具体写入shard的公式是：

shard_num = hash(_routing) % num_primary_shards

其中_routing是document对应的_id值，可以自定义设置值替代_id，但是在日常开发中这种场景并不常见。

（5）其他元数据

自定义特殊的数据类型，这种在日常开发中直接忽略。

## Fields or properties
ES存在多种定义的Field，是日常开发的核心也是本文的重点。

## Fields数据类型

### 核心数据类型
（1）String类型：textandkeyword

text和keyword最大的区别是分词，text：需要分词支持全文索引；keyword不分词，支持精确查找；

text对应TextFieldMapper类，属性是设置分词。ES默认的分词对于英文来说相当友好，但是对应中文的支持还是比较若，建议使用IK，jieba等针对中文优化的分词器。全文索引使用优化的中文分词器是十分重要的，这是搜索的基础步骤。
```
public static final MappedFieldType FIELD_TYPE = new TextFieldType();

static {
    FIELD_TYPE.freeze();
}

public TextFieldType() {
    setTokenized(true);
    fielddata = false;
    fielddataMinFrequency = Defaults.FIELDDATA_MIN_FREQUENCY;
    fielddataMaxFrequency = Defaults.FIELDDATA_MAX_FREQUENCY;
    fielddataMinSegmentSize = Defaults.FIELDDATA_MIN_SEGMENT_SIZE;
}
```
支持的查询包含prefixQuery/spanPrefixQuery/existsQuery等待。
```
public Query prefixQuery(String value, MultiTermQuery.RewriteMethod method, QueryShardContext context) {   
}

public SpanQuery spanPrefixQuery(String value, SpanMultiTermQueryWrapper.SpanRewriteMethod method, QueryShardContext context) { 
}

public Query existsQuery(QueryShardContext context) { 
}

public Query phraseQuery(TokenStream stream, int slop, boolean enablePosIncrements) throws IOException {

}
public Query multiPhraseQuery(TokenStream stream, int slop, boolean enablePositionIncrements) throws IOException {
}
public Query phrasePrefixQuery(TokenStream stream, int slop, int maxExpansions) throws IOException {
    return analyzePhrasePrefix(stream, slop, maxExpansions);
}
```
keyword对应KeywordFieldMapper类，属性不分词，设置norms，索引只存储文档，不存储词频/offset/position位置等信息。Lucene.KEYWORD_ANALYZER对应的是Lucend的KeywordAnalyzer，即把词语只分为一个token。
```
public static final MappedFieldType FIELD_TYPE = new KeywordFieldType();

static {
    FIELD_TYPE.setTokenized(false);
    FIELD_TYPE.setOmitNorms(true);
    FIELD_TYPE.setIndexOptions(IndexOptions.DOCS);
    FIELD_TYPE.freeze();
}

public KeywordFieldType() {
    setIndexAnalyzer(Lucene.KEYWORD_ANALYZER);
    setSearchAnalyzer(Lucene.KEYWORD_ANALYZER);
}
```
支持的查询是DocValuesFieldExistsQuery/TermQuery/NormsFieldExistsQuery三种。
```
public Query existsQuery(QueryShardContext context) {
    if (hasDocValues()) {
        return new DocValuesFieldExistsQuery(name());
    } else if (omitNorms()) {
        return new TermQuery(new Term(FieldNamesFieldMapper.NAME, name()));
    } else {
        return new NormsFieldExistsQuery(name());
    }
}
```
text和keyword类型，在Lucene对应的是TextField和StringField，属性设置区别最大的也是分词，感兴趣的可以看下Lucene的实现。

（2）数字类型

整型对应字段取值范围如下：

数据类型NumberFieldType需要实现IndexNumericFieldData接口（ScaledFloat除外）。
```
public NumberFieldType(NumberType type) {
    super();
    this.type = Objects.requireNonNull(type);
    setTokenized(false);
    setHasDocValues(true);
    setOmitNorms(true);
}
```
支持的查询：existsQuery/termQuery/termsQuery/rangeQuery.

（3）Range类型

Range类型包含integer_range,float_range,long_range,double_range,date_range

Range类型对应源码RangeFieldMapper类，属性是不分词，HasDocValues属性不是Lucene内部设定而是ES设置。
```
RangeFieldType(RangeType type) {
    super();
    this.rangeType = Objects.requireNonNull(type);
    setTokenized(false);
    setHasDocValues(true);
    setOmitNorms(true);
    if (rangeType == RangeType.DATE) {
        setDateTimeFormatter(DateFieldMapper.DEFAULT_DATE_TIME_FORMATTER);
    }
}
```
支持的查询：existsQuery/termQuery/rangeQuery/withinQuery/containsQuery/intersectsQuery等。

以integer_range为例，ES的integer_range字段是对Lucene字段IntRange的封装，其他依次类推。

（4）时间类型

date 精读是毫秒，格式以ISO8601为标准构建，几乎涵盖了所有日期类型。源码可以查看StrictISODateTimeFormat类。

date_nanos精读是纳秒，使用long类型存储，时间范围大于是1970 到 2262。这种场景在日常开发中很少见，可以忽略。

（5）Boolean类型

取值可以说false, "false"或true, "true"。（个人理解，几乎就是Keyword类型的变身，只是取值固定true||false）

（6）Binary

存储Binary类型需要Base64编码为String类型，不索引也不存储（Binary类型，感觉像是凑的一样，在ES内存储Binary类型实际开发意义不大）

## 复杂数据类型
（1）Object类型针对JSON存在嵌套的情况，官网给了个示例：

其实存储信息前缀是叠加的，这里就有个需要注意的地方，嵌套层数一定不能太多，不然Key值会很大。

（2）Nested 类型是特殊的Object类型，支持数组。

### 空间数据类型
Geo-point和Geo-shape类型主要用于地理位置信息，经纬度数据，应用场景比较特殊。

### 特殊数据类型
IPipfor IPv4 and IPv6 IP地址；

completion自动补全类型 ；

Join父子索引类型；

Alias 不止索引可以设置别名，Filed也可以设置别名；

特殊数据类型比较多，也是在特定场景下使用，还是开发的时候再仔细看吧。

### 数组
Array类型要求同一个数组内的字段类型必须相同。

### Multi-fields
如果一个String，需要keyword的同时还需要全文搜索，这时候就可以用Multi-fields；

如果一个String，需要多种分词器，例如需要英语也需要标准分词器（这种情况不适合中文，中文分词器本身据就需要定制），也可以用Multi-fields；

ES的字段类型是大而全，但是不是适合所有实际开发的场景，实际应用是还是看官网和代码比较准确。从源码看ES的字段类型大多是基于Lucene字段类型的封装，并提供了多种查询方式，对开发者更加友好。