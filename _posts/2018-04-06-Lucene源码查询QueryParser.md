---
layout: post
categories: [Lucene]
description: none
keywords: Lucene
---
# Lucene源码查询QueryParser
Lucene 的 QueryParser 是用于解析查询字符串并生成查询对象的类。

## 功能
QueryParser 类位于 org.apache.lucene.queryparser.classic 包中。它是一个用于解析用户查询字符串的工具类，将查询字符串转换成相应的 Lucene 查询对象。

QueryParser 是 Lucene 中的一个核心类，用于将用户输入的查询字符串转换成相应的查询对象，以便进行搜索操作。其作用主要包括以下几个方面：
- 解析查询字符串
QueryParser 可以解析用户输入的查询字符串，根据其中的关键词、短语、通配符、布尔运算符等语法规则进行分析，生成相应类型的查询对象。这些查询对象包括 TermQuery、PhraseQuery、WildcardQuery、BooleanQuery 等。
- 指定字段和分析器
QueryParser 可以指定要搜索的字段名和分析器，以便正确地对查询字符串进行分析处理。例如，如果使用 StandardAnalyzer 分析器，则会对查询字符串进行分词、停用词过滤、小写化等操作，以便生成准确的查询对象。
- 支持自定义查询
QueryParser 在生成查询对象时，支持用户自定义查询表达式，包括模糊查询、范围查询、前缀查询等，满足不同的搜索需求。
- 简化查询运算
QueryParser 还可以简化查询运算，设置默认的逻辑运算符（AND 或 OR），避免用户手动添加逻辑运算符带来的麻烦和错误。

以下是 QueryParser 类的关键方法：
- 构造函数：QueryParser 提供多个构造函数，用于初始化查询解析器的字段、分析器和搜索配置等参数。
- parse(String query) 方法：该方法接受一个查询字符串作为参数，并将其解析成相应的查询对象。
- setAllowLeadingWildcard(boolean allowLeadingWildcard) 方法：设置是否允许在通配符查询或前缀查询中使用通配符 * 或 ? 作为首字符，默认为禁止。
- setLowercaseExpandedTerms(boolean lowercaseExpandedTerms) 方法：设置是否将查询字符串转换为小写形式，默认为开启。
- setMultiTermRewriteMethod(MultiTermQuery.RewriteMethod method) 方法：设置多词项查询（如通配符查询、前缀查询等）的重写方法，默认为 constantScore_auto_rewrite。
- setDefaultOperator(Operator operator) 方法：设置默认的逻辑运算符（AND 或 OR）来组合用空格分隔的查询项，默认为 Operator.OR。
- setPhraseSlop(int phraseSlop) 方法：设置短语查询允许的词项位置之间的最大距离，默认为 0。
- setAutoGeneratePhraseQueries(boolean value) 方法：设置是否自动生成短语查询，默认为关闭。
- setFuzzyMinSim(float fuzzyMinSim) 和 setFuzzyPrefixLength(int fuzzyPrefixLength) 方法：用于设置模糊查询的最小相似度和前缀长度。 

QueryParser 使用了内部类 QueryParserConstants 和 QueryParserTokenManager 来进行词法分析和语法解析。在解析查询字符串时，它会根据查询字符串中的特殊符号和关键词进行相应的处理，生成对应类型的查询对象（如布尔查询、短语查询、通配符查询等）。

同时，QueryParser 还支持扩展和自定义的方式，你可以通过继承 QueryParserBase 类并重写相应的方法来实现自定义的查询解析逻辑。

## QueryParser源码
```
    public static void main(String[] args) throws Exception {
        QueryParser qp = new QueryParser("field",
                new org.apache.lucene.analysis.standard.StandardAnalyzer());
        Query q = qp.parse(args[0]);
        System.out.println(q.toString("field"));
    }
```

首先，我们来看一下QueryParser类的构造函数：
```
 public QueryParser(String f, Analyzer a) {
        this((CharStream)(new FastCharStream(new StringReader(""))));
        this.init(f, a);
    }
```
这段代码是 QueryParser 类的一个构造函数。让我们逐行来分析它：
- this((CharStream)(new FastCharStream(new StringReader(""))));
这是调用 QueryParser 类的另一个构造函数的方式，传入一个 CharStream 对象作为参数。CharStream 是用于提供字符流的抽象基类。

- this.init(f, a);
这是一个私有方法 init(String f, Analyzer a) 的调用，用于初始化 QueryParser 对象的字段和分析器。f 参数是要搜索的字段名，a 参数是用于分析查询字符串的分析器。

总结起来，这个构造函数的目的是使用空的字符串作为输入，将字段和分析器设置为指定的值。它通过调用 init 方法完成了这些设置。

整个Lucene的搜索入口点在TopLevelQuery函数，它最后返回Query类型对象。
```
 public Query parse(String query) throws ParseException {
    ReInit(new FastCharStream(new StringReader(query)));
    try {
      // TopLevelQuery is a Query followed by the end-of-input (EOF)
      Query res = TopLevelQuery(field);
      return res!=null ? res : newBooleanQuery().build();
    }
    catch (ParseException | TokenMgrError tme) {
      // rethrow to include the original query:
      ParseException e = new ParseException("Cannot parse '" +query+ "': " + tme.getMessage());
      e.initCause(tme);
      throw e;
    } catch (BooleanQuery.TooManyClauses tmc) {
      ParseException e = new ParseException("Cannot parse '" +query+ "': too many boolean clauses");
      e.initCause(tmc);
      throw e;
    }
  }
```
这段代码是一个解析查询字符串的方法实现，它使用自定义的词法分析器和解析器来将查询字符串转换为相应的查询对象。在解析过程中，可能会捕获到异常，并将其封装成自定义的 ParseException 异常进行抛出，以提供更详细的错误信息。

## 查询语句解析
先做关键字识别、词条处理等等，最终生成语法树。我们创建QueryParser的时候，使用的是StandardAnalyzer分词器，analyzer主要完成分词、去除停用词、转换大小写等操作。

QueryParser在parse生成Query的时候，会根据不同的查询类型生成不同的Query，比如WildcardQuery、RegexpQuery、PrefixQuery等等。

假设现在已经有多个文档被索引成功，索引目录为：D:\index。我们要对name域（Field）进行查询，代码如下：
```
Path path = Paths.get("D:\\index");
Directory directory = FSDirectory.open(path);
IndexReader indexReader = DirectoryReader.open(directory);
IndexSearcher indexSearcher = new IndexSearcher(indexReader);
 
Analyzer nameanalyzer = new StandardAnalyzer();
QueryParser nameParser = new QueryParser("name", nameanalyzer);
Query nameQuery = nameParser.parse("詹姆斯");
 
TopDocs topDocs = indexSearcher.search(nameQuery, 2);
......
```
最终生成的是BooleanQuery，“詹姆斯”被分为三个词：詹  姆  斯。（当然也根据实际情况也可以不用分词，比如使用TermQuery）

BooleanQuery为布尔查询，支持四种条件字句：
- MUST("+")：表示必须匹配该子句，类似于sql中的AND。
- FILTER("#")：和MUST类似，但是它不参与评分。
- SHOULD("")：表示可能匹配该字句。类似于sql中的OR，对于没有MUST字句的布尔查询，匹配文档至少需要匹配一个SHOULD字句。
- MUST_NOT("-")：表示不匹配该字句。类似于sql中的!=。但是要注意的是，布尔查询不能仅仅只包含一个MUST_NOT字句。并且这些字句不会参与文档的评分。

使用这些条件，可以组成很复杂的复合查询。在我们的例子中，会根据分词结果生成三个查询子句，它们之间使用SHOULD关联：
```
name:詹 SHOULD name:姆 SHOULD name:斯
```
将上述查询语句按照语法树分析：
```
            Query
             |
         BooleanQuery
             |
    +--------+---------+--------+---------
    |                  |                 |
SHOULD                SHOULD         SHOULD
    |                  |                 |
name:詹               name:姆           name:斯   
```




## QueryParse.jj解析器
Lucene QueryParse.jj使用 JavaCC（Java Compiler Compiler）生成的解析器，根据预定义的语法规则将查询字符串分解为语法单元。

Lucene的Token正规表达式定义
```

/* ***************** */
/* Token Definitions */
/* ***************** */

<*> TOKEN : {
// 数字
  <#_NUM_CHAR:   ["0"-"9"] >
// every character that follows a backslash is considered as an escaped character
// 特殊字符
| <#_ESCAPED_CHAR: "\\" ~[] >
// Term的起始字符，除了下面列出的其它字符都可以。
| <#_TERM_START_CHAR: ( ~[ " ", "\t", "\n", "\r", "+", "-", "!", "(", ")", ":", "^",
                           "[", "]", "\"", "{", "}", "~", "*", "?", "\\" ]
                       | <_ESCAPED_CHAR> ) >
// term                       
| <#_TERM_CHAR: ( <_TERM_START_CHAR> | <_ESCAPED_CHAR> | "-" | "+" ) >
// 空格和回车
| <#_WHITESPACE: ( " " | "\t" | "\n" | "\r") >
}

<DEFAULT, RangeIn, RangeEx> SKIP : {
  < <_WHITESPACE>>
}

<DEFAULT> TOKEN : {
  <AND:       ("AND" | "&&") >
| <OR:        ("OR" | "||") >
| <NOT:       ("NOT" | "!") >
| <PLUS:      "+" >
| <MINUS:     "-" >
| <LPAREN:    "(" >
| <RPAREN:    ")" >
| <COLON:     ":" >
| <STAR:      "*" >
| <CARAT:     "^" > : Boost
| <QUOTED:     "\"" (~["\""] | "\\\"")* "\"">
// term由二部分TERM_START_CHAR + (TERM_CHAR)*来组成
| <TERM:      <_TERM_START_CHAR> (<_TERM_CHAR>)*  >
// 词权重 字符串必须以字符~开始，而后是数字。模糊字符串的正规表达式必须以字符~开始。
| <FUZZY_SLOP:     "~" ( (<_NUM_CHAR>)+ ( "." (<_NUM_CHAR>)+ )? )? >
// 前缀字符串的正规表达式必须以*结尾。
| <PREFIXTERM:  ("*") | ( <_TERM_START_CHAR> (<_TERM_CHAR>)* "*" ) >
// 通配字符串的正规表达式必须包含*或者？字符。
| <WILDTERM:  (<_TERM_START_CHAR> | [ "*", "?" ]) (<_TERM_CHAR> | ( [ "*", "?" ] ))* >
| <RANGEIN_START: "[" > : RangeIn
| <RANGEEX_START: "{" > : RangeEx
}

<Boost> TOKEN : {
<NUMBER:    (<_NUM_CHAR>)+ ( "." (<_NUM_CHAR>)+ )? > : DEFAULT
}

<RangeIn> TOKEN : {
<RANGEIN_TO: "TO">
| <RANGEIN_END: "]"> : DEFAULT
// 表示用"包起来的字符串，字符"开始，中间由不是"的符号或者连着的这两个符号"组成，字符"结束，
| <RANGEIN_QUOTED: "\"" (~["\""] | "\\\"")+ "\"">
| <RANGEIN_GOOP: (~[ " ", "]" ])+ >
}

// 包含边界的rnage查询
// 不包含边界的range查询
<RangeEx> TOKEN : {
<RANGEEX_TO: "TO">
| <RANGEEX_END: "}"> : DEFAULT
| <RANGEEX_QUOTED: "\"" (~["\""] | "\\\"")+ "\"">
| <RANGEEX_GOOP: (~[ " ", "}" ])+ >
}
```

一个Query查询语句，是由多个clause组成，每个cluas有对应的modifier (+/-)连接符。
```
Query::=Modifiers Clause (Conjunction Modifiers Clause)*

// This makes sure that there is no garbage after the query string
Query TopLevelQuery(String field) : 
{
  Query q;
}
{
  q=Query(field) <EOF>
  {
    return q;
  }
}

Query Query(String field) :
{
  Vector clauses = new Vector();
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
```
查询方法，接收一个字段名 field 作为参数，并返回一个查询对象。

它使用 Modifiers 方法获取修饰符（如 "+"、"-"）并调用 Clause 方法解析主要查询子句。然后，它使用 Conjunction 方法获取连词（如 "AND"、"OR"）和 Modifiers 方法再次获取修饰符，并调用 Clause 方法解析其他查询子句。最后，根据解析得到的子句和连词，构建一个复合查询对象（如 BooleanQuery）。

### Clause语义
clause产生式使用的LL(2)提前查询了二个token,避免了函数的回朔。多补充一句，JavaCC是自顶而下的非递归算法

驱动表 (first + follow集)
stack

```
Clause::=[(<TERM> <COLON>|<STAR> <COLON>)]


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
    {
      if (boost != null) {
        float f = (float)1.0;
  try {
    f = Float.valueOf(boost.image).floatValue();
          q.setBoost(f);
  } catch (Exception ignored) { }
      }
      return q;
    }
}
```
Clause产生式分析如下：

- 当遇到“field : abc"时，确定是field而不是term查询，<term><colon>是输入的特征。通过执行动作：field=discardEscapeChar(fieldToken.image))来获得field的值。
- 当遇到"abc and (xyz or efg)"时，确定是grouping分组情况，支持使用小括号对每个子句进行分组，从而支持更加复杂的嵌套查询逻辑。

分组递归定义:
```
| <LPAREN> q=Query(field) <RPAREN> (<CARAT> boost=<NUMBER>)?
```

Term语义
```
(Term|<LPAREN> Query <RPAREN> (<CARAT> <NUMBER>)?) 

Term::=(
  (<TERM>|<STAR>|<PREFIXTERM>|<WILDTERM>|<NUMBER>) [<FUZZY_SLOP>] [<CARAT><NUMBER>[<FUZZY_SLOP>]]

| ( <RANGEIN_START> (<RANGEIN_GOOP>|<RANGEIN_QUOTED>) [ <RANGEIN_TO> ] (<RANGEIN_GOOP>|<RANGEIN_QUOTED> <RANGEIN_END> ) [ <CARAT> boost=<NUMBER> ] 

| ( <RANGEEX_START> <RANGEEX_GOOP>|<RANGEEX_QUOTED> [ <RANGEEX_TO> ] <RANGEEX_GOOP>|<RANGEEX_QUOTED> <RANGEEX_END> )[ <CARAT> boost=<NUMBER> ] 

| <QUOTED> [ <FUZZY_SLOP> ] [ <CARAT> boost=<NUMBER> ] 
)
```

Query产生式分析如下

第一阶段
普通normal-term
通配wildcard-term
对应的解析动作：wilcard变量进行了初始化。
前缀prefix-term
对应的解析动作：prefix变量进行了初始化。
模糊词fuzzy-term
对应的解析动作：fuzzy变量进行了初始化。
它必须以~开头，后面接数字。
数字term

类型的token的term变量正确初始化,并且根据token来初始化wildcard/prefix/fuzzy这样的bool值。
```
Query Term(String field) : {
  Token term, boost=null, fuzzySlop=null, goop1, goop2;
  boolean prefix = false;
  boolean wildcard = false;
  boolean fuzzy = false;
  boolean rangein = false;
  Query q;
}
{
  (
     (
       term=<TERM>
       | term=<STAR> { wildcard=true; }
       | term=<PREFIXTERM> { prefix=true; }
       | term=<WILDTERM> { wildcard=true; }
       | term=<NUMBER>
     )
     [ fuzzySlop=<FUZZY_SLOP> { fuzzy=true; } ]
     [ <CARAT> boost=<NUMBER> [ fuzzySlop=<FUZZY_SLOP> { fuzzy=true; } ] ]
     {
       String termImage=discardEscapeChar(term.image);
       if (wildcard) {
       q = getWildcardQuery(field, termImage);
       } else if (prefix) {
         q = getPrefixQuery(field,
           discardEscapeChar(term.image.substring
          (0, term.image.length()-1)));
       } else if (fuzzy) {
             float fms = fuzzyMinSim;
             try {
            fms = Float.valueOf(fuzzySlop.image.substring(1)).floatValue();
             } catch (Exception ignored) { }
            if(fms < 0.0f || fms > 1.0f){
              throw new ParseException("Minimum similarity for a FuzzyQuery has to be between 0.0f and 1.0f !");
            }
            q = getFuzzyQuery(field, termImage,fms);
       } else {
         q = getFieldQuery(field, termImage);
       }
     }
```

第二阶段
根据第一阶段识别出term的类型，调用不同API生成 不同子类的term对象

或者只包含普通字符的term ( abc)
调用getFieldQuery
或者包含wildcard通配符的term (abc*xyz 或者abc ?xyz)
调用getWildcardQuery
或者包含prefix的term (abc*). [其中必须出现在未尾]
调用getPrefixQuery
或者包含fuzzy模糊匹配的term (abc~3)
调用getFuzzyQuery
或者包含数字的term (333)
调用getFieldQuery

```
 {
       String termImage=discardEscapeChar(term.image);
       if (wildcard) {
       q = getWildcardQuery(field, termImage);
       } else if (prefix) {
         q = getPrefixQuery(field,
           discardEscapeChar(term.image.substring
          (0, term.image.length()-1)));
       } else if (fuzzy) {
          float fms = fuzzyMinSim;
          try {
            fms = Float.valueOf(fuzzySlop.image.substring(1)).floatValue();
          } catch (Exception ignored) { }
         if(fms < 0.0f || fms > 1.0f){
           throw new ParseException("Minimum similarity for a FuzzyQuery has to be between 0.0f and 1.0f !");
         }
         q = getFuzzyQuery(field, termImage,fms);
       } else {
         q = getFieldQuery(field, termImage);
       }
     }
```

第三阶段
支持有界与无界范围查询。

[date1 to date2]
[name1 to name2]
比如，range匹配: title: {leon to sammi}

<pangein_start>匹配 {
通过goop = <pangein_goop>取"león"
<pangein_to>匹配 to
通过goop = <pangein_goop>取"sammi"
<pangein_end>匹配 }
其对应的执行动作是调用getRangeQuery函数，创建RangeQuery对象。
```
| ( <RANGEIN_START> ( goop1=<RANGEIN_GOOP>|goop1=<RANGEIN_QUOTED> )
         [ <RANGEIN_TO> ] ( goop2=<RANGEIN_GOOP>|goop2=<RANGEIN_QUOTED> )
         <RANGEIN_END> )
       [ <CARAT> boost=<NUMBER> ]
        {
          if (goop1.kind == RANGEIN_QUOTED) {
            goop1.image = goop1.image.substring(1, goop1.image.length()-1);
          }
          if (goop2.kind == RANGEIN_QUOTED) {
            goop2.image = goop2.image.substring(1, goop2.image.length()-1);
          }
          q = getRangeQuery(field, discardEscapeChar(goop1.image), discardEscapeChar(goop2.image), true);
        }
     | ( <RANGEEX_START> ( goop1=<RANGEEX_GOOP>|goop1=<RANGEEX_QUOTED> )
         [ <RANGEEX_TO> ] ( goop2=<RANGEEX_GOOP>|goop2=<RANGEEX_QUOTED> )
         <RANGEEX_END> )
       [ <CARAT> boost=<NUMBER> ]
        {
          if (goop1.kind == RANGEEX_QUOTED) {
            goop1.image = goop1.image.substring(1, goop1.image.length()-1);
          }
          if (goop2.kind == RANGEEX_QUOTED) {
            goop2.image = goop2.image.substring(1, goop2.image.length()-1);
          }

          q = getRangeQuery(field, discardEscapeChar(goop1.image), discardEscapeChar(goop2.image), false);
        }
     | 
```

第四阶段
Lucene支持给不同的查询词设置不同的权重，设置权重使用"^"符号，将"^"符号放置于查询词term的尾部，同时紧跟权重值。权重值越大，这个词term就越重要。在搜索过程中具备更高的相关性。

term查询的加权匹配: field: xyz^4

<cart>匹配 ^
通过boost =<number> 来初始化boost=4
```
{
    if (boost != null) {
      float f = (float) 1.0;
      try {
        f = Float.valueOf(boost.image).floatValue();
      }
      catch (Exception ignored) {
    /* Should this be handled somehow? (defaults to "no boost", if
     * boost number is invalid)
     */
      }

      // avoid boosting null queries, such as those caused by stop words
      if (q != null) {
        q.setBoost(f);
      }
    }
    return q;
  }
```

getFieldQuery公共函数
```
  protected Query getFieldQuery(String field, String queryText)  throws ParseException {
 
    TokenStream source = analyzer.tokenStream(field, new StringReader(queryText));
    Vector v = new Vector();
    org.apache.lucene.analysis.Token t;
    int positionCount = 0;
    boolean severalTokensAtSamePosition = false;
​
    while (true) {
      try {
        t = source.next();
      }
      catch (IOException e) {
        t = null;
      }
      if (t == null)
        break;
      v.addElement(t);
      if (t.getPositionIncrement() != 0)
        positionCount += t.getPositionIncrement();
      else
        severalTokensAtSamePosition = true;
    }
    try {
      source.close();
    } 
​
    if (v.size() == 0)
      return null;
    else if (v.size() == 1) {
      t = (org.apache.lucene.analysis.Token) v.elementAt(0);
      return new TermQuery(new Term(field, t.termText()));
    } else {
      if (severalTokensAtSamePosition) {
        if (positionCount == 1) {
          // no phrase query:
          BooleanQuery q = new BooleanQuery(true);
          for (int i = 0; i < v.size(); i++) {
            t = (org.apache.lucene.analysis.Token) v.elementAt(i);
            TermQuery currentQuery = new TermQuery(
                new Term(field, t.termText()));
            q.add(currentQuery, BooleanClause.Occur.SHOULD);
          }
          return q;
        }
        else {
          // phrase query:
          MultiPhraseQuery mpq = new MultiPhraseQuery();
          mpq.setSlop(phraseSlop);          
          List multiTerms = new ArrayList();
          int position = -1;
          for (int i = 0; i < v.size(); i++) {
            t = (org.apache.lucene.analysis.Token) v.elementAt(i);
            if (t.getPositionIncrement() > 0 && multiTerms.size() > 0) {
              if (enablePositionIncrements) {
                mpq.add((Term[])multiTerms.toArray(new Term[0]),position);
              } else {
                mpq.add((Term[])multiTerms.toArray(new Term[0]));
              }
              multiTerms.clear();
            }
            position += t.getPositionIncrement();
            multiTerms.add(new Term(field, t.termText()));
          }
          if (enablePositionIncrements) {
            mpq.add((Term[])multiTerms.toArray(new Term[0]),position);
          } else {
            mpq.add((Term[])multiTerms.toArray(new Term[0]));
          }
          return mpq;
        }
      }
      else {
        PhraseQuery pq = new PhraseQuery();
        pq.setSlop(phraseSlop);
        int position = -1;
        for (int i = 0; i < v.size(); i++) {
          t = (org.apache.lucene.analysis.Token) v.elementAt(i);
          if (enablePositionIncrements) {
            position += t.getPositionIncrement();
            pq.add(new Term(field, t.termText()),position);
          } else {
            pq.add(new Term(field, t.termText()));
          }
        }
        return pq;
      }
    }
  }
```

TermQuery
```
 if (v.size() == 0)
      return null;
    else if (v.size() == 1) {
      t = (org.apache.lucene.analysis.Token) v.elementAt(0);
      return new TermQuery(new Term(field, t.termText()));
```
比如，输入"field: Beijing"，搜索内容"Beijing"只会被解析成一个分词，对应一个term：term1("Beijing")。如果分词后只有一个Token，则生成new TermQuery(new Term(field, t.termText()))对象。

BoolQuery
```
BooleanQuery q = new BooleanQuery(true);
          for (int i = 0; i < v.size(); i++) {
            t = (org.apache.lucene.analysis.Token) v.elementAt(i);
            TermQuery currentQuery = new TermQuery(
                new Term(field, t.termText()));
            q.add(currentQuery, BooleanClause.Occur.SHOULD);
          }
          return q;
```
比如，输入field:"Beijing China"时，搜索内容"Beijing China"会被解析成二个分词，对应二个term：term1("Beijing")和term("China")。最后会实例化PhraseQuery pq = new PhraseQuery()对象。它背后语义是：所有的分词在搜索中必须同时匹配与满足。

Demo: "+(foo bar) +(baz boo)" 表达成 (foo OR bar) AND (baz OR boo)
```
assertQueryEquals("(foo OR bar) AND (baz OR boo)", null,
            "+(foo bar) +(baz boo)");
```











