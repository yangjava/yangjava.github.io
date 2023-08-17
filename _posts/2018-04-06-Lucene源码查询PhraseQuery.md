---
layout: post
categories: Lucene
description: none
keywords: Lucene
---
# Lucene源码查询PhraseQuery
Lucene的PhraseQuery匹配包含特定分词term的文档列表，PhraseQuery会使用存储在索引中的分词位置信息。

查询语句中词与词之间允许其他词的数量称之为Slop，使用~符号来结束查询语句。二个分词term之间距离越小，得分越高。

举个例子：使用“spicy food”这个短语搜索，如果待匹配的文档中包含了“spicy food”这个短语，这个文档就算是匹配成功。但如果待匹配的文档中包含了“spicy chinese food”，那么就会匹配失败。这个时候通过设定slop等于1，也能匹配成功。

slop是指查询语句中二个词之间允许的最大间隔距离。slop默认值为0，即表示查询语句中多个词条严格按照顺序在文档中进行检索，也就是不允许中间有任意其它的词。如果设置slop为1，表示“spicy food”这二个词之间可以“添加”或者“删除”至多1个词。

### 一个查询短语举例
如前所述，slop是指查询语句中二个词之间允许的最大间隔距离。

比如，一个文档内容“ abcba”，执行短语查询“ a b c”~ 1，此处的slop为1。文档中每个单词的位置以及短语查询中每个分词的偏移量如下:

执行短语查询“ a b c”~ 1中，它有三个term查询一起构成整个短语查询。其中term1=“a”，term2=“b”和term3=“c”。

phraseQuery(term1:offset1, term3:offset3, slop:1)中，term1在文档中的位置标记为doc_term1_pos， 而term3在文档中的位置标记为doc_term3_pos。在查询短语中，term1的偏移位置是offset1，term3的偏移位置是offset3。

文档中doc_term3_pos出现在doc_term1_pos后面，必须满足:
```
（doc_term3_pos - offset3）- （doc_term1_pos - offset1) <= slop
```

文档中doc_term3_pos出现在doc_term1_pos前面，必须满足:
```
（doc_term1_pos - offset1) -（doc_term3_pos - offset3) <= slop
```

小结
```
MAX(doc_term1_pos - offset1, doc_term3_pos - offset3) - MIN(doc_term1_pos - offset1, doc_term3_pos - offset3) <= slop
```

自然地，当slop为0时，查询短语中每个phrase_term需要满足如下条件:
```
doc_term1_pos - offset1 = doc_term3_pos - offset3
```

Lucene中设计了PhrasePositions类，查询短语中每个term会封装成对应的PhrasePositions实例。这样查询短语中第i个term的信息（doc_term_i_pos - offset_i）完全封装在PhrasePositions#nextPosition函数中。

类似：doc_term_pos - offset由PhrasePositions类负责计算，这个差值也称为phrase-position。

执行短语查询“ a b c”~ 1中会出现3个term，此时需要满足如下条件：
```
MAX(doc_term1_pos - offset1, doc_term3_pos - offset3, doc_term3_pos - offset3) - 
MIN(doc_term1_pos - offset1, doc_term3_pos - offset3, doc_term3_pos - offset3) 
                        <= slop
```

总结为：
```
max_phrase_position - min_phrase_position <= slop
```

### PhrasePositions类
距离的计算在 PhraseScorer#phrasefreq函数中实现。设计PhrasePositions类来统计每个term分别在文档与查询短语中的相对位置。

PhrasePositions#nextPosition方法转到term分词在当前文档的下一个位置，并设置PhrasePositions.position = doc_term_pos - offset。当所有的PhrasePositions.position都相等的话，目标匹配的文档就识别出来了。

```
final class PhrasePositions { 
  int doc;            // current doc
  int position;           // position in doc
  int count;            // remaining pos in this doc
  int offset;           // position in phrase
  TermPositions tp;         // stream of positions
  
  final boolean nextPosition() {
    if (count-- > 0) {              
      // read subsequent pos's
      position = tp.nextPosition() - offset;
      return true;
    } else
      return false;
  }
}
```

查询短语中每个term的phrase-position的计算
```
doc_term1_pos - offset1 = doc_term2_pos - offset2 = doc_term3_pos - offset3 
```

对于短语查询中每个term在文档中匹配的所有位置，可以计算出phrase-position，下面即是所有term对应的phrase-positions。
```
phrase-position => doc_term_pos - offset
```

比如，一个文档内容“ abcba”，执行短语查询“ a b c”~ 1，此处的slop为1。文档中每个单词的位置以及短语查询中每个分词的偏移量如下:

根据上面信息，计算每个位置组合的max_phrase_position - min_phrase_position，并将其作为距离。

假定一个文档内容“ abcba”，执行短语查询“ a b c”~ 1。计算每个term在文档出现位置的phrase_position。其中phrase_position_a和phrase_position_b和phrase_position_c分别表示三个term对应的phrase_position。

当前的三个箭头分别指向三个term在文档列表中各自的第一个位置时对应的phrase_position。显然它们之间距离是满足小于slop(1)。

第一次循环以phrase_position_a开始，停止条件是pharse_position_a大于phrase_position_b。使用以下公式计算match_length = max_phrase_position - min_phrase_position = 0，很明显是小于slop(1)。 继续遍历term_a在文档中下一个phrase_position_a位置。
```
max_phrase_position - min_phrase_position <= slop
```
此时，phrase_position_a指向了文档的第4个位置，它的值大于phrase_position_b。结束第一轮循环。当前最小的二个phrase_position是phrase_position_b和phrase_position_c。

开始第二轮查找，以phrase_position_b位置开始，结束条件是phrase_position_b大于phrase_position_c。使用以下公式计算match_length = max_phrase_position - min_phrase_position = 4，值大于slop(1)，并不满足匹配要求。继续遍历term_b在文档中下一个phrase_position_b位置。
```
max_phrase_position - min_phrase_position <= slop
```

现在，phrase_position_b指向了文档的第三个位置，其值等于2并大于phrase_position_c。结束第二轮循环。当前最小的二个phrase_position是phrase_position_c和phrase_position_b。

开始第三轮查找，以phrase_position_c位置开始，结束条件是phrase_position_c大于phrase_position_b。计算match_length = max_phrase_position - min_phrase_position = 4，值大于slop(1)，并不满足匹配要求。继续遍历term_c在文档中下一个phrase_position_c位置，因为这是term_c在文档中最后一个位置。循环退出。

## TermPositions与TermDocs类
TermDocs提供一个用于枚举索引中某个term中所有<doc, frequency>映射对的接口。其中doc包含了当前的分词term，并且term在doc中出现的频率次数是frequency。<doc, frequency>映射对是以文档ID有序存储。

TermDocs主要方法：

TermDocs#next方法用于移动下一对<doc, frequency>。 如果返回true，说明某个term在索引中存在更多的文档中。
TermDocs#skipTo(target)方法用于跳转到当前文档大于或等于target的位置。
继承自TermDocs的子类TermPositions，它提供一个用于枚举索引中某个term中所有<doc, frequency, <position>*>元组的接口。其中，position列表部分列出了文档中每个term分词出现位置的信息。

TermPositions主要方法：

TermPositions#nextPosition方法用于返回某个term分词在当前文档中的下一个位置。

## PhraseQuery类体系

```
query = new PhraseQuery();
query.add(new Term("field", "four"));
query.add(new Term("field", "five"));
Hits hits = searcher.search(query);
```

PhraseQuery类
```
public class PhraseQuery extends Query {
  private String field;
  private Vector terms = new Vector();
  private Vector positions = new Vector();
  private int slop = 0;
```
PhraseQuery查询用来匹配包含特定分词序列（短语）的文档。PhraseQuery定义了一个terms数组，表示支持对多个分词的组合查询。positions也是一个数组，元素类型是SegmentTermPositions，用来存储每个分词在索引中的位置信息。

SegmentTermPositions类
```
final class SegmentTermPositions
extends SegmentTermDocs implements TermPositions {
  private IndexInput proxStream;
  private int proxCount;
  private int position;
```
由于PhraseQuery涉及到统计不同term之间的间隔，所以将term和查询相关的信息<doc, frequency, <position> * > * 封装成TermPositions的子类。如上所术，TermPositions子类的nextPosition方法用于返回某个term分词在当前文档中的下一个位置。

## PhraseScorer类
PhraseScorer主要方法：

int[] getpositions()
返回查询语句（短语）中分词的相对位置。
Term[] getTerms()
返回查询语句（短语）中分词的列表。
Weight createWeight(Searcher searcher)
为当前查询PhraseQuery构建一个PhraseWeight对象。PhraseWeight#score方法内部根据slop的值，决定构建ExactPhraseScorer或者SloppyPhraseScorer对象，它们都PhraseScorer的子类。

```
public Scorer scorer(IndexReader reader) {
  if (slop == 0)              // optimize exact case
    return new ExactPhraseScorer(this, tps, getPositions(),
                                 similarity,
                                 reader.norms(field));
  else
    return new SloppyPhraseScorer(this, tps, getPositions(), 
                             similarity, slop,
                             reader.norms(field));
}
```
在PhraseQuery构建一个PhraseWeight对象后，PhraseWeight#scorer方法使用 IndexReader.termPositions(term)返回SegmentTermPositions对象，该对象被传递到Scorer的PhrasePosition对象中，PhrasePosition是对TermPosition的一个封装，在ExactPhraseScorer和SloppyPhraseScorer类中被用来计算phrase-position信息。

## PhrasePosition
为什么需要PhrasePosition？用来统计查询短语中每个term的phrase-position数据。
```
final class PhrasePositions {
  int doc;            // current doc
  int position;           // position in doc
  int count;            // remaining pos in this doc
  int offset;           // position in phrase
  TermPositions tp;         // stream of positions
  PhrasePositions next;         // used to make lists
  boolean repeats;  

  PhrasePositions(TermPositions t, int o) {
    tp = t;
    offset = o;
  }
```

## PhraseScorer工作流
PhraseQuery对查询短语的搜索，入口依然是从indexSearch.search方法开始，内部基于PhraseQuery实例对象会生成PhraseWeight对象，weight对象提供的scorer方法根据查询短语是否设置slop值，来创建计分器对象。如果slop为0，构建ExactPhraseScorer子类对象，如果slop非0，创建SloppyPhraseScorer子类对象。

紧接着，PhraseScorer子类对象会调用score方法，通过next方法循环遍历短语在索引中的文档集来收集结果。其中score#next方法是一个虚方法，PhraseScorer负责具体的实现。PhraseScorer#next内部又会进一步调用 phraseFreq虚方法找到一个同时包含所有分词的文档 。phraseFreq在二个子类ExactPhraseScorer和SloppyPhraseScorer中有具体的实现。

整个过程，是一个典型的模板设计模式，在基类Scorer#next定义了通用的流程，每个子类根据各自定义扩展自己的逻辑实现。
```
public abstract class Scorer {
  public void score(HitCollector hc) throws IOException {
    while (next()) {
      hc.collect(doc(), score());
    }
  }
}
```

二个基本条件
满足所有term都指向同一个文档。
查询短语中每个term在同一个文档中的距离不能大于slop。
我特别想对比下SpanNearQuery与PhraseQuery二者的实现。本质上，对每个子查询（每个子查询可理解成一个term的查询）都需要满足二个基本条件：第一是子查询的结果都指向同一个文档；第二是保证子查询在文档出现的位置距离不能大于预置的slop值。
```
class NearSpansOrdered implements Spans {
  private boolean advanceAfterOrdered() throws IOException {
    while (more && (inSameDoc || toSameDoc())) {
      if (stretchToOrder() && shrinkToAfterShortestMatch()) {
        return true;
      }
    }
    return false; // no more matches
  }
}

public class SpanNearQuery extends SpanQuery {
    public Spans getSpans(final IndexReader reader) {
    return inOrder
            ? (Spans) new NearSpansOrdered(this, reader)
            : (Spans) new NearSpansUnordered(this, reader);
  }
}
```

算法1： 确定所有term都出现在同一个文档
PhraseScorer
```
abstract class PhraseScorer extends Scorer {
  private Weight weight;
  protected byte[] norms;
  protected float value;
  protected PhraseQueue pq;
  protected PhrasePositions first, last;
  
  PhraseScorer(Weight weight, TermPositions[] tps, 
               int[] offsets, Similarity similarity,
               byte[] norms) { 
    for (int i = 0; i < tps.length; i++) {
      PhrasePositions pp = new PhrasePositions(tps[i], offsets[i]);
      if (last != null) {       
        // add next to end of list
        last.next = pp;
      } else
        first = pp;
      last = pp;
    }

    pq = new PhraseQueue(tps.length);             
    // construct empty pq
  }
}
```
PhraseScorer类的构造函数中创建一个链表，由first和last分别指向链表的首尾二端。链表元素是PhrasePositions，表示查询短语中出现的每个term的phrase-position信息。

PhraseScorer#next
```
 public boolean next() throws IOException {
    if (firstTime) {
      init();
      firstTime = false;
    } else if (more) {
      more = last.next();                         
      // trigger further scanning
    }
    return doNext();
  }
```
PhraseScorer#next函数中，init用来遍历PhrasePositions链表并初始化时优先队列。PhrasePositions链表的每一个元素代表查询短语中的某一个term查询。我们知道term可能在一个文档出现多次，也可能在多个文档出现多次。比如，Lucene索引中类似<term, <doc, docFreq, <position>* > *>组织。

```
private void init() throws IOException {
  for (PhrasePositions pp = first; more && pp != null; pp = pp.next) 
    more = pp.next();
  if(more)
    sort();
}

private void sort() {
  pq.clear();
  for (PhrasePositions pp = first; pp != null; pp = pp.next)
    pq.put(pp);
  pqToList();
}
```
对链表每个元素进行循环，对每个PhrasePositions元素调用next方法，间接将每个 term在各自doc的位置进行定位。

```
final class PhrasePositions {
  int doc;               // current doc
  int position;                  // position in doc
  int offset;                // position in phrase
  TermPositions tp;               // stream of positions
  PhrasePositions next;    
  
   final boolean next(){    
     // increments to next doc
    if (!tp.next()) {
      return false;
    }
    doc = tp.doc();
    position = 0;
    return true;
  }
}
```
此处，关键函数是PhraseScorer#doNext，第一部分是找出查询短语中term共同出现的文档列表；第二部分使用PhraseScorer#phraseFreq虚函数，允许二个子类重写这个逻辑。 ExactPhraseScorer#phrasefreq函数分析见下文。
```
private boolean doNext() {
  while (more) {
    while (more && first.doc < last.doc) {      
      // find doc w/ all the terms
      more = first.skipTo(last.doc);            
      // skip first upto last
      firstToLast();                            
      // and move it to the end
    }

    if (more) {
      // found a doc with all of the terms
      freq = phraseFreq();                     
      // check for phrase
      if (freq == 0.0f)                         
        // no match
        more = last.next();                     
      // trigger further scanning
      else
        return true;                           
      // found a match
    }
  }
  return false;                                 
  // no more matches
}
```
我们先聚焦PhraseScorer#doNext方法第一部分逻辑，它用来对查询短语中每个term对应的倒排表求交集运算。
```
while (more && first.doc < last.doc) {      
  // find doc w/ all the terms
  more = first.skipTo(last.doc);            
  // skip first upto last
  firstToLast();                            
  // and move it to the end
}
```

Lucene Search深入分析出现过一个ConjunctionScorer#doNext方法，用来实现完全一致的算法：求解多个倒排表的交集。
```
class ConjunctionScorer extends Scorer {
  private final Scorer[] scorers;
  
  private boolean doNext() {
    int first=0;
    Scorer lastScorer = scorers[scorers.length-1];
    Scorer firstScorer;
    while (more && (firstScorer=scorers[first]).doc() 
           < (lastDoc=lastScorer.doc())) {
      more = firstScorer.skipTo(lastDoc);
      lastScorer = firstScorer;
      first = (first == (scorers.length-1)) ? 0 : first+1;
    }
    return more;
  }
}
```

算法2： 查询短语中每个term在一个文档中出现的距离不大于 slop
继续看PhraseScorer#doNext方法第二部分逻辑：已知某个文档包含查询短语中所有的term分词后，如何判断所有term在当前文档中出现的距离不大于slop？主要的逻辑由子类ExactPhraseScorer#phrasefreq方法重载实现。
```
private boolean doNext() {
  while (more) {
    // First Part: 对多个term的倒排表取交集
    // Second Part:
    if (more) {
      // found a doc with all of the terms
      freq = phraseFreq();                     
      if (freq == 0.0f)                         
        more = last.next();                     
      else
        return true;                           
      // found a match
    }
  }
  return false;                                 
}
```

## ExactPhraseScorer#phrasefreq
如前所述，当slop为0时，查询短语中每个phrase_term需要满足如下条件:
```
doc_term1_pos - offset1 = doc_term2_pos - offset2 = doc_term3_pos - offset3
```
一个文档内容“ abcba”，执行查询短语“ a b c”~ 1，

我们还是以这个例子来说明，下面需要考虑如何计算查询短语中每一个term的phrase-position位置信息。考虑到一个term（比如term_a)在文档中会出现多次，所以需要通过firstPosition，nextPosition函数取出term在文档出现的所有位置，并计算相应的phrase-position。

只有计算机所有的term在某一文档中的phrase-position值完全相同，则说明该文档匹配。

```
protected final float phraseFreq() throws IOException {
    pq.clear();
    for (PhrasePositions pp = first; pp != null; pp = pp.next{
      pp.firstPosition();
      pq.put(pp);         // build pq from list
    }
    pqToList();           // rebuild list from pq
  
    int freq = 0;
    do {            // find position w/ all terms
      while (first.position < last.position) {    
        // scan forward in first
      do {
        if (!first.nextPosition())
          return (float)freq;
      } while (first.position < last.position);
        firstToLast();
      }
      freq++;           
      // all equal: a match
    } while (last.nextPosition());
  
    return (float)freq;
  }
```
对PhrasePositions链表再次遍历，next函数通过firstPoistion间接调用了nextPosition函数，position = tp.nextPosition() - offset用来在遍历term在文档中不同位置的同时，也计算出term对应的term-phrase值。遍历链表的同时，也将计算出phrase-position值的链表元素插入优先队列PhraseQueue。
```
final void firstPosition() {
  count = tp.freq();              // read first pos
  nextPosition();
}

 final boolean nextPosition() {
    if (count-- > 0) {          // read subsequent pos's
      position = tp.nextPosition() - offset;
      return true;
    } else
      return false;
  }
```

pqToList
紧接着，pqToList函数是基于优先队列PhraseQueue重建PhrasePositions链表。
```
protected final void pqToList() {
  last = first = null;
  while (pq.top() != null) {
    PhrasePositions pp = (PhrasePositions) pq.pop();
    if (last != null) {            // add next to end of list
      last.next = pp;
    } else
      first = pp;
    last = pp;
    pp.next = null;
  }
}
```
为什么需要从pqToList重建PhrasePositions链表？

即不断从priorityQueue弹出到PhrasePositions对象并插入到有序链表中来完成转换。priorityQueue通过lessThan重新定义对phrase_position值进行排序的规则。

## PhraseQueue
比较规则是基于phrase_position值进行排序。
```
final class PhraseQueue extends PriorityQueue {
  PhraseQueue(int size) {
    initialize(size);
  }

  protected final boolean lessThan(Object o1, Object o2) {
    PhrasePositions pp1 = (PhrasePositions)o1;
    PhrasePositions pp2 = (PhrasePositions)o2;
    if (pp1.doc == pp2.doc) 
      if (pp1.position == pp2.position)
        return pp1.offset < pp2.offset;
      else
        return pp1.position < pp2.position;
    else
      return pp1.doc < pp2.doc;
  }
}
```
SloppyPhraseScorer

## MultiPhraseQuery

MultiPhraseQuery支持多短语查询，可能通过多个查询短语的拼接来实现复杂的查询。

比如：使用StandardAnalyzer分词器建立索引，希望找出同时包含“spicy food”和“spicy ingredients”二个查询短语的文档。 其中一个解决方案是：指定一个前缀“spicy”，一个后缀数组Term[] (new Term(“food”), new Term(“ingredients”))，则查询的结果即满足要求。

当然，也可以指定一个后缀，多个前缀，并且允许设定slop值，指定前缀与后缀之间最多可以相距的间隔。






















