---
layout: post
categories: [Lucene]
description: none
keywords: Lucene
---
# Lucene源码查询FuzzyQuery
FuzzyQuery模糊匹配允许用户非精确的匹配结果集。它是许多搜索引擎框架的基石，也是为什么即使用户在查询中有一个拼写错误时，仍旧获得搜索结果的主要原因之一。

实际上几乎所有搜索引擎框架背后主要使用Levenshtein (edit distance) algorithm来实现fuzzy模糊字符串匹配。

## Levenshtein Distance 一个简单例子
如何Levenshtein距离计算？简单说，你只需逐个比较二个字符串。比较过程中，如果二个字符不同，就可以将单词之间的距离加1.

比如，常见的拼写错误单词"acqurie"和正确的单词"acquire"之间的距离。在使用Levenshtein距离计算后，如下图所示，上述二个单词的距离为2。

## Edit distance原理
问题简化
输入：给定二个字符串s1与s2。

输出：计算出s1转换成s2所使用的最少步数。

允许对字符串有三种操作：插入，删除和替换一个字符。

补充：Lucene最终便是从索引中取出域值串与查询短语字符串逐个比较编辑距离，来判断是否保留域值。

## 思考框架
对于s1[i]与s2[j]每个字符，有如下4种情况：

```
if s1[i] == s2[j]
  i++; j++
else 
  insert | delete | replace
```
运用上面框架，问题就容易解决了，有else分支中究竟使用哪个，需要通过暴力枚举所有三种操作（insert, delete或者replace)，最后选择编辑距离最小的哪种情况。

## 递归框架
```
//求s1和s2字符串的编辑距离
func minEditDistance(i, j int) int {
  distances := [][]int{}
  for i := 0; i < len(s2); i++ {
      distances[0][i] = i
  }
  for i := 0; i < len(s1); i++ {
      distances[i][0] = i
  }
  
  if s1[i] == s2[j] {
    return distances[i-1][j-1]
  else {
    return min(
      minEditDistance(i, j - 1) + 1,
      minEditDistance(i -1, j) + 1,
      minEditDistance(i -1, j -1) + 1
    )
  }
}
```

当s1[i] ！= s2[j]，每层涉及到的三种递归调用：

- minEditDistance(i, j - 1)递归：表示在s1[i]后面插入一个s2[j]一样的字符。
- minEditDistance(i - 1, j)递归：表示将s1[i]字符删除。
- minEditDistance(i - 1, j - 1)递归：表示将s1[i]替换成s2[j]后，就匹配了。

## DP框架
动态规划与递归算法最大区别是，前者是自底向上，后者是自顶向下。显然DP算法效率会更加高效，可以充分避免子问题被重复计算。

```
int minEditDistance(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] editDist = new int[m + 1][n + 1];
  
    for (int i = 1; i <= m; i++)
        editDist[i][0] = i;
    for (int j = 1; j <= n; j++)
        editDist[0][j] = j;
    
    // 自底向上
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            if (s1.charAt(i-1) == s2.charAt(j-1))
                editDist[i][j] = editDist[i - 1][j - 1];
            else               
                editDist[i][j] = min(
                    editDist[i - 1][j] + 1,
                    editDist[i][j - 1] + 1,
                    editDist[i-1][j-1] + 1
                );
    return editDist[m][n];
}
```

minEditDistance的动态规划算法揭示出：editDist[i] [j] 只和它相邻的三个状态相关。

## FuzzyQuery工作流
简单梳理下面的搜索过程。

首先，使用indexReader打开索引文档，读取索引文件内容。 其次，将new FuzzyQuery(new Term("field", "aaaaa"))查询对象树转换成weight对象树，weight会考虑term的权重。

接着，使用weight对象树构建scorer对象树，这个过程会引入文档打分机制。scorer对象树的叶子节点，比如TermScorer会从索引中取出倒排表。

最后，基于查询语句特征，对多个倒排表进行取交集或者并集后，将文档结果集返回给用户。
```
public void testFuzziness() throws Exception {
  RAMDirectory directory = new RAMDirectory();
  IndexWriter writer = new IndexWriter(directory
                         , new WhitespaceAnalyzer(), true);
  addDoc("aaaaa", writer);
  addDoc("aaaab", writer);
  addDoc("aaabb", writer);
  addDoc("aabbb", writer);
  addDoc("abbbb", writer);
  addDoc("bbbbb", writer);
  addDoc("ddddd", writer);
  writer.optimize();
  writer.close();
  IndexSearcher searcher = new IndexSearcher(directory);

  query = new FuzzyQuery(new Term("field", "aaaaa"), 
                         FuzzyQuery.defaultMinSimilarity, 3);   
  hits = searcher.search(query)
  assertEquals(3, hits.length());
```
FuzzyQuery#search的执行流程与普通的TermQuery大致相同。创建Hits对象，将FuzzyQuery查询对象转换成FuzzyQuery#weight对象前，会对FuzzyQuery的查询进行重写。

### Hits & weight
```
public Hits search(Query query, Filter filter)  {
  return new Hits(this, query, filter);
}
  
Hits(Searcher s, Query q, Filter f) {
  weight = q.weight(s);
}

public Weight weight(Searcher searcher) {
  Query query = searcher.rewrite(this);
  Weight weight = query.createWeight(searcher);
}
```

### rewrite
查询重写主要涉及multiTermQuery，即一个query查询语句代表了多个term参与查询，比如PrefixQuery和当前分析的FuzzyQuery。对于此类的查询，Lucene无法直接查询，必须进行重写。有二种情况：

一种情况是：查询语句使用fuzzyQuery或者prefixQuery前缀查询，需要用前缀查询获取索引中具体包含的域值，然后将一个term的多个域值，使用booleanClause对象，组装成booleanQuery对象，使用OR逻辑关系。
另一种情况是：查询语句使用多个term。转换也很直观，将多个term通过booleanCluase对象，组装成booleanQuery对象，使用OR逻辑关系。
fuzzyQuery#rewrite重写流程与逻辑与以前系统文章中的介绍（比如，prefixQuery）高度相似，唯一不同的是在查询重写中引入了Levenshtein distance算法来计算二个term的编辑距离。

```
public Query rewrite(Query original) {
  Query query = original;
  for (Query rewrittenQuery = query.rewrite(reader); 
       rewrittenQuery != query;
       rewrittenQuery = query.rewrite(reader)) {
    query = rewrittenQuery;
  }
  return query;
}
```

## FuzzyQuery#rewrite
FuzzyQuery是基于levenshtein distance算法。它提供了一个定义前缀长度的选项，我思考主要的原因是允许在检查模糊匹配之前，对可能的结果进行预先搜索，从而提高性能。defaultPrefixLength的缺省前缀长度为0，可伸缩性并不很好。建议自行覆盖这个值。

FuzzyQuery查询重写逻辑有二部分：第一部分是基于prefix-query从索引的倒排表中取出候选term分词项。对应第一个循环中next方法取出term列表，它会间接调用FuzzyQuery#termCompare方法完成候选分词term与查询短语之间编辑距离的计算，满足条件的分词会留下来放入优先队列，并进入到第二部分处理。

第二部分，确定找到满足前缀查询和编辑距离的term分词项，有了这些term后组装成booleanQuery布尔查询对象，同样使用OR逻辑关系。

```
public class FuzzyQuery extends MultiTermQuery {
  private int prefixLength;
  private float minimumSimilarity;
  
  public Query rewrite(IndexReader reader) {
    FilteredTermEnum enumerator = getEnum(reader);
    int maxClauseCount = BooleanQuery.getMaxClauseCount();
    ScoreTermQueue stQueue = 
      new ScoreTermQueue(maxClauseCount);

    try {
      do {
        float score = 0.0f;
        Term t = enumerator.term();
        if (t != null) {
          score = enumerator.difference();
          reusableST = new ScoreTerm(t, score);
          reusableST = (ScoreTerm) stQueue.insertWithOverflow(reusableST);
        }
      } while (enumerator.next());
    } 

    BooleanQuery query = new BooleanQuery(true);
    int size = stQueue.size();
    for(int i = 0; i < size; i++){
      ScoreTerm st = (ScoreTerm) stQueue.pop();
      TermQuery tq = new TermQuery(st.term);      
      // found a match
      tq.setBoost(getBoost() * st.score); 
      // set the boost
      query.add(tq, BooleanClause.Occur.SHOULD);          
      // add to query
    }
    return query;
  }
}
```

FilteredTermEnum#getEnum
```
protected FilteredTermEnum getEnum(IndexReader reader) {
  return new FuzzyTermEnum(reader, getTerm(), 
                           minimumSimilarity, prefixLength);
}

 public FuzzyTermEnum(IndexReader reader, Term term, 
                      final float minSimilarity, 
                      final int prefixLength)  {
    super();
    setEnum(reader.terms(
      new Term(searchTerm.field(), prefix)));
  }
```

## FilteredTermEnum#setEnum 和termCompare函数
termCompare 方法使用莱文斯坦距离:计算给定项和基于prefix从索引中取出的term项之间的距离。

FuzzyQuery提供了minimumSimilarity字段，用来代表最小相似度。通过指定一个相似度来决定模糊匹配的程度。默认值 为0.5，当这个值越小，通过模糊查询出的文档匹配的程序越低，文档数量也就越多。当这个值越大，说明要匹配的程度越高，匹配的文档数目也就越少。当minimumSimilarity设为1时，那么退化为TermQuery查询。

similarity方法用来计算二个输入字符串（查询短语串与基于prefix前缀从索引中取出的候选项term）的编辑距离，然后返回1-(编辑距离/长度) 。只有当这个计算的值大于FuzzyQuery预定义minimumSimilarity相似度值，当前的term才会被留下并放入优先队列。similarity作为查询重写逻辑的核心，留在优先队列的多个term会转换成一个booleanQuery对象，用来做最后的索引查询。

```
protected void setEnum(TermEnum actualEnum) throws IOException {
    this.actualEnum = actualEnum;
    // Find the first term that matches
    Term term = actualEnum.term();
    if (term != null && termCompare(term)) 
        currentTerm = term;
    else next();
}

protected final boolean termCompare(Term term) {
    if (field == term.field() && 
        term.text().startsWith(prefix)) {
        final String target = 
          term.text().substring(prefix.length());
        this.similarity = similarity(target);
        return (similarity > minimumSimilarity);
    }
    endEnum = true;
    return false;
  }
```

FuzzyTermEnum#similarity：本质是编辑距离算法

Edit distance原理一节已经详细分析了算法的原理与实现。下面是Lucene源码中对edit distance算法动态规划的实现。了解了原理，实现并没有太多的惊喜。

值得说明的是，此算法被广泛应用于拼写检查，语句识别，DNA分析等场景。
```
public final class FuzzyTermEnum extends FilteredTermEnum {
  
  private synchronized final float similarity(final String target) {
    final int m = target.length();
    final int n = text.length();

    final int maxDistance = getMaxDistance(m);
    if (d[0].length <= m) {
      growDistanceArray(m);
    }

    // init matrix d
    for (int i = 0; i <= n; i++) d[i][0] = i;
    for (int j = 0; j <= m; j++) d[0][j] = j;

    // start computing edit distance
    for (int i = 1; i <= n; i++) {
      int bestPossibleEditDistance = m;
      final char s_i = text.charAt(i - 1);
      for (int j = 1; j <= m; j++) {
        if (s_i != target.charAt(j-1)) {
            d[i][j] = min(d[i-1][j], 
                          d[i][j-1], 
                          d[i-1][j-1])+1;
        }
        else {
          d[i][j] = min(d[i-1][j]+1, 
                        d[i][j-1]+1, 
                        d[i-1][j-1]);
        }
        bestPossibleEditDistance = 
          Math.min(bestPossibleEditDistance, d[i][j]);
      }
 
      if (i > maxDistance && bestPossibleEditDistance > maxDistance) { 
        return 0.0f;
      }
    }
 
    return 1.0f - ((float)d[n][m] / (float) (prefix.length() + Math.min(n, m)));
  }
}
```



























