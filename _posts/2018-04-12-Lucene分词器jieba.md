---
layout: post
categories: [Lucene]
description: none
keywords: Lucene
---
# 中文分词器
在语言理解中，词是最小的能够独立活动的有意义的语言成分。将词确定下来是理解自然语言的第一步，只有跨越了这一步，中文才能像英文那样过渡到短语划分、概念抽取以及主题分析，以至自然语言理解，最终达到智能计算的最高境界。因此，每个NLP工作者都应掌握分词技术。

## 中文分词简介
“词”这个概念一直是汉语语言学界纠缠不清而又绕不开的问题。“词是什么”（词的抽象定义）和“什么是词”（词的具体界定），这两个基本问题迄今为止也未能有一个权威、明确的表述，更无法拿出令大众认同的词表来。主要难点在于汉语结构与印欧体系语种差异甚大，对词的构成边界方面很难进行界定。比如，在英语中，单词本身就是“词”的表达，一篇英文文章就是“单词”加分隔符（空格）来表示的，而在汉语中，词以字为基本单位的，但是一篇文章的语义表达却仍然是以词来划分的。因此，在处理中文文本时，需要进行分词处理，将句子转化为词的表示。这个切词处理过程就是中文分词，它通过计算机自动识别出句子的词，在词间加入边界标记符，分隔出各个词汇。整个过程看似简单，然而实践起来却很复杂，主要的困难在于分词歧义。以NLP分词的经典语句举例，“结婚的和尚未结婚的”，应该分词为“结婚/的/和/尚未/结婚/的”，还是“结婚/的/和尚/未/结婚/的”？这个由人来判定都是问题，机器就更难处理了。此外，像未登录词、分词粒度粗细等都是影响分词效果的重要因素。

自中文自动分词被提出以来，历经将近30年的探索，提出了很多方法，可主要归纳为“规则分词”“统计分词”和“混合分词（规则+统计）”这三个主要流派。规则分词是最早兴起的方法，主要是通过人工设立词库，按照一定方式进行匹配切分，其实现简单高效，但对新词很难进行处理。随后统计机器学习技术的兴起，应用于分词任务上后，就有了统计分词，能够较好应对新词发现等特殊场景。然而实践中，单纯的统计分词也有缺陷，那就是太过于依赖语料的质量，因此实践中多是采用这两种方法的结合，即混合分词。

下面将详细介绍这些方法的代表性算法。

## 规则分词
基于规则的分词是一种机械分词方法，主要是通过维护词典，在切分语句时，将语句的每个字符串与词表中的词进行逐一匹配，找到则切分，否则不予切分。

按照匹配切分的方式，主要有正向最大匹配法、逆向最大匹配法以及双向最大匹配法三种方法。

### 正向最大匹配法
正向最大匹配（Maximum Match Method，MM法）的基本思想为：假定分词词典中的最长词有i个汉字字符，则用被处理文档的当前字串中的前i个字作为匹配字段，查找字典。若字典中存在这样的一个i字词，则匹配成功，匹配字段被作为一个词切分出来。如果词典中找不到这样的一个i字词，则匹配失败，将匹配字段中的最后一个字去掉，对剩下的字串重新进行匹配处理。如此进行下去，直到匹配成功，即切分出一个词或剩余字串的长度为零为止。这样就完成了一轮匹配，然后取下一个i字字串进行匹配处理，直到文档被扫描完为止。

其算法描述如下：

1）从左向右取待切分汉语句的m个字符作为匹配字段，m为机器词典中最长词条的字符数。

2）查找机器词典并进行匹配。若匹配成功，则将这个匹配字段作为一个词切分出来。若匹配不成功，则将这个匹配字段的最后一个字去掉，剩下的字符串作为新的匹配字段，进行再次匹配，重复以上过程，直到切分出所有词为止。

比如我们现在有个词典，最长词的长度为5，词典中存在“南京市长”和“长江大桥”两个词。现采用正向最大匹配对句子“南京市长江大桥”进行分词，那么首先从句子中取出前五个字“南京市长江”，发现词典中没有该词，于是缩小长度，取前4个字“南京市长”，词典中存在该词，于是该词被确认切分。再将剩下的“江大桥”按照同样方式切分，得到“江”“大桥”，最终分为“南京市长”“江”“大桥”3个词。显然，这种结果还不是我们想要的。

### 逆向最大匹配法

逆向最大匹配（Reverse Maximum Match Method，RMM法）的基本原理与MM法相同，不同的是分词切分的方向与MM法相反。逆向最大匹配法从被处理文档的末端开始匹配扫描，每次取最末端的i个字符（i为词典中最长词数）作为匹配字段，若匹配失败，则去掉匹配字段最前面的一个字，继续匹配。相应地，它使用的分词词典是逆序词典，其中的每个词条都将按逆序方式存放。在实际处理时，先将文档进行倒排处理，生成逆序文档。然后，根据逆序词典，对逆序文档用正向最大匹配法处理即可。

由于汉语中偏正结构较多，若从后向前匹配，可以适当提高精确度。所以，逆向最大匹配法比正向最大匹配法的误差要小。统计结果表明，单纯使用正向最大匹配的错误率为1/169，单纯使用逆向最大匹配的错误率为1/245。比如之前的“南京市长江大桥”，按照逆向最大匹配，最终得到“南京市”“长江大桥”。当然，如此切分并不代表完全正确，可能有个叫“江大桥”的“南京市长”也说不定。

### 双向最大匹配法

双向最大匹配法（Bi-directction Matching method）是将正向最大匹配法得到的分词结果和逆向最大匹配法得到的结果进行比较，然后按照最大匹配原则，选取词数切分最少的作为结果。据SunM.S.和Benjamin K.T.（1995）的研究表明，中文中90.0%左右的句子，正向最大匹配法和逆向最大匹配法完全重合且正确，只有大概9.0%的句子两种切分方法得到的结果不一样，但其中必有一个是正确的（歧义检测成功），只有不到1.0%的句子，使用正向最大匹配法和逆向最大匹配法的切分虽重合却是错的，或者正向最大匹配法和逆向最大匹配法切分不同但两个都不对（歧义检测失败）。这正是双向最大匹配法在实用中文信息处理系统中得以广泛使用的原因。

前面举例的“南京市长江大桥”，采用该方法，中间产生“南京市/长江/大桥”和“南京市/长江大桥”两种结果，最终选取词数较少的“南京市/长江大桥”这一结果。

## 统计分词
随着大规模语料库的建立，统计机器学习方法的研究和发展，基于统计的中文分词算法渐渐成为主流。

其主要思想是把每个词看做是由词的最小单位的各个字组成的，如果相连的字在不同的文本中出现的次数越多，就证明这相连的字很可能就是一个词。因此我们就可以利用字与字相邻出现的频率来反应成词的可靠度，统计语料中相邻共现的各个字的组合的频度，当组合频度高于某一个临界值时，我们便可认为此字组可能会构成一个词语。

基于统计的分词，一般要做如下两步操作：

1）建立统计语言模型。

2）对句子进行单词划分，然后对划分结果进行概率计算，获得概率最大的分词方式。这里就用到了统计学习算法，如隐含马尔可夫（HMM）、条件随机场（CRF）等。

### HMM模型
隐含马尔可夫模型（HMM）是将分词作为字在字串中的序列标注任务来实现的。其基本思路是：每个字在构造一个特定的词语时都占据着一个确定的构词位置（即词位），现规定每个字最多只有四个构词位置：即B（词首）、M（词中）、E（词尾）和S（单独成词），那么下面句子1）的分词结果就可以直接表示成如2）所示的逐字标注形式：

1）中文/分词/是/.文本处理/不可或缺/的/一步！

2）中/B文/E分/B词/E是/S文/B本/M处/M理/E不/B可/M或/M缺/E的/S一/B步/E！/S

用数学抽象表示如下：用λ=λ1λ2…λn代表输入的句子，n为句子长度，λi表示字，o=o1o2…on代表输出的标签，那么理想的输出即为：

max=maxP(o1o2…on|λ1λ2…λn) （3.4）

在分词任务上，o即为B、M、E、S这4种标记，λ为诸如“中”“文”等句子中的每个字（包括标点等非中文字符）。

需要注意的是，P(o|λ)是关于2n个变量的条件概率，且n不固定。因此，几乎无法对P(o|λ)进行精确计算。这里引入观测独立性假设，即每个字的输出仅仅与当前字有关，于是就能得到下式：

P(o1o2…on|λ1λ2…λn)=P(o1|λ1)P(o2|λ2)…P(on|λn) （3.5）

事实上，P(ok|λk)的计算要容易得多。通过观测独立性假设，目标问题得到极大简化。然而该方法完全没有考虑上下文，且会出现不合理的情况。比如按照之前设定的B、M、E和S标记，正常来说B后面只能是M或者E，然而基于观测独立性假设，我们很可能得到诸如BBB、BEM等的输出，显然是不合理的。

### 其他统计分词算法
条件随机场（CRF）也是一种基于马尔可夫思想的统计模型。在隐含马尔可夫中，有个很经典的假设，那就是每个状态只与它前面的状态有关。这样的假设显然是有偏差的，于是学者们提出了条件随机场算法，使得每个状态不止与他前面的状态有关，还与他后面的状态有关。该算法在本节将不会重点介绍，会在后续章节详细介绍。

神经网络分词算法是深度学习方法在NLP上的应用。通常采用CNN、LSTM等深度学习网络自动发现一些模式和特征，然后结合CRF、softmax等分类算法进行分词预测。基于深度学习的分词方法，我们将在后续介绍完深度学习相关知识后，再做拓展。

## 中文分词工具——Jieba
近年来，随着NLP技术的日益成熟，开源实现的分词工具越来越多，如Ansj、盘古分词等。在本书中，我们选取了Jieba进行介绍和案例展示，主要基于以下考虑：
- 社区活跃。在本书写作的时候，Jieba在Github上已经有将近10000的star数目。社区活跃度高，代表着该项目会持续更新，实际生产实践中遇到的问题能够在社区反馈并得到解决，适合长期使用。
- 功能丰富。Jieba其实并不是只有分词这一个功能，其是一个开源框架，提供了很多在分词之上的算法，如关键词提取、词性标注等。
- 提供多种编程语言实现。Jieba官方提供了Python、C++、Go、R、iOS等多平台多语言支持，不仅如此，还提供了很多热门社区项目的扩展插件，如ElasticSearch、solr、lucene等。在实际项目中，进行扩展十分容易。
- 使用简单。Jieba的API总体来说并不多，且需要进行的配置并不复杂，方便上手。

Jieba分词官网地址是：https://github.com/fxsjy/jieba，可以采用如下方式进行安装。

Jieba分词结合了基于规则和基于统计这两类方法。首先基于前缀词典进行词图扫描，前缀词典是指词典中的词按照前缀包含的顺序排列，例如词典中出现了“上”，之后以“上”开头的词都会出现在这一部分，例如“上海”，进而会出现“上海市”，从而形成一种层级包含结构。如果将词看作节点，词和词之间的分词符看作边，那么一种分词方案则对应着从第一个字到最后一个字的一条分词路径。因此，基于前缀词典可以快速构建包含全部可能分词结果的有向无环图，这个图中包含多条分词路径，有向是指全部的路径都始于第一个字、止于最后一个字，无环是指节点之间不构成闭环。基于标注语料，使用动态规划的方法可以找出最大概率路径，并将其作为最终的分词结果。对于未登录词，Jieba使用了基于汉字成词的HMM模型，采用了Viterbi算法进行推导。

Jieba的三种分词模式

Jieba提供了三种分词模式：
- 精确模式：试图将句子最精确地切开，适合文本分析。
- 全模式：把句子中所有可以成词的词语都扫描出来，速度非常快，但是不能解决歧义。
- 搜索引擎模式：在精确模式的基础上，对长词再次切分，提高召回率，适合用于搜索引擎分词。

## 快速分词利器Jieba的实现思想
从实现特点上看：
- jieba基于前缀词典实现高效的词图扫描，生成句子中汉字所有可能成词情况所构成的有向无环图 (DAG)；
- 采用了动态规划查找最大概率路径, 找出基于词频的最大切分组合；
- 对于未登录词，采用了基于汉字成词能力的 HMM 模型，使用了 Viterbi 算法。

### 基本底库词典

基本底库包括词典，dict.txt用于分词，idf.txt用于关键词提取，stop_words.txt用于停用词过滤。

其中，dict.txt默认有349046个词，每个词指定了词的名称、频次和词性。在源码中，还给出了dict.txt.big共584429个词，dict.txt.small，共109749个词。

### 建立有向无环图DAG

DAG是jieba分词的核心。

针对输入的文本，通过词表匹配的方式找到所有词语，然后根据头尾，构建领阶链表存储方式，每个词都会有一个连接的有向边，字本身都会有一个自我连接的边。
```python
    # 有向无环图构建主函数  
    def get_DAG(self, sentence):  
        # 检查系统是否已经初始化  
        self.check_initialized()  
        # DAG存储向无环图的数据，数据结构是dict  
        DAG = {}  
        N = len(sentence)  
        # 依次遍历文本中的每个位置  
        for k in xrange(N):  
            tmplist = []  
            i = k  
            # 位置k形成的片段  
            frag = sentence[k]  
            # 判断片段是否在前缀词典中  
            # 如果片段不在前缀词典中，则跳出本循环  
            # 也即该片段已经超出统计词典中该词的长度  
            while i < N and frag in self.FREQ:  
                # 如果该片段的词频大于0  
                # 将该片段加入到有向无环图中  
                # 否则，继续循环  
                if self.FREQ[frag]:  
                    tmplist.append(i)  
                # 片段末尾位置加1  
                i += 1  
                # 新的片段较旧的片段右边新增一个字  
                frag = sentence[k:i + 1]  
            if not tmplist:  
                tmplist.append(k)  
            DAG[k] = tmplist  
        return DAG  
```
援引文献4的总结，如上图jieba源码所述，其实现思想在于：

从前往后依次遍历文本的每个位置，对于位置k，首先形成一个片段，这个片段只包含位置k的字，然后就判断该片段是否在前缀词典中，如果这个片段在前缀词典中，如果词频大于0，就将这个位置i追加到以k为key的一个列表中；

如果词频等于0，则表明前缀词典存在这个前缀，但是统计词典并没有这个词，继续循环；

如果这个片段不在前缀词典中，则表明这个片段已经超出统计词典中该词的范围，则终止循环；

然后该位置加1，然后就形成一个新的片段，该片段在文本的索引为[k:i+1]，继续判断这个片段是否在前缀词典中。

例如，以“春雨医生你好全身红疹子”为例，给定词表：
```
春雨 100 n  
医生 100 n  
春雨医生 100 n  
你好 100 n  
全身 100 n  
红疹 100 n  
疹子 100 n  
```
用索引位置代表字符，邻接链表存取表示如下。0:[0, 1, 3]代表春可以组成词[春, 春雨, 春雨医生]
```
{0: [0, 1, 3], 1: [1], 2: [2, 3], 3: [3], 4: [4, 5], 5: [5]}  
```
形成有向无环图如下

路径得分排序

在构建好图之后，给每条路径加上权重，分词转换成了求最大路径的问题，将最大路径转换成分词的目标函数就是最大联合概率(路径相乘)。

实现:
```
    def calc(self, sentence, DAG, route):  
        N = len(sentence)  
        route[N] = (0, 0)  
        logtotal = log(self.total)  
        for idx in xrange(N - 1, -1, -1):  
            route[idx] = max((log(self.FREQ.get(sentence[idx:x + 1]) or 1) -logtotal + route[x + 1][0], x) for x in DAG[idx])  
```
如上面代码所示，每条边的权重即这个词的概率为: p=freq/total，freq代表这个词的频数，也就是词表里面设定的100，total为所有词频数之和，由于freq小于total，那么p是小于1的。null

以春雨医生为例，共存在四条路径:
```
[春，雨，医，生]、[春雨，医，生]、[春雨，医生]、[春雨医生]。
```
这春雨医生四个字没有与其他字的路径，也就是说这四个字的分词是独立，他们的最大路径必定也是整个句子的最大路径的子路径。

通过计算，可以发现：
```
p[春，雨，医，生]<p[春雨，医，生]<p[春雨，医生]<p[春雨医生],  
```
即jieba倾向于分出更长的词，这是因为词权重小于1，相乘越多越小。

### 未登录词再切分
前面说到，构建前缀词典后，可以构建有向无环图，计算最大概率路径，基于最大概率路径进行分词，可以得到一版结果，但未登录的问题十分明显。

为了解决这一问题，jieba设定了一个处理逻辑：遇到未登录词，则调用HMM模型进行切分，其实也就是再对词进行切分。具体实现上，对于未登录词（注意：未登录词不是词典没有的词，而是单个字的词）组合成buf，对buf使用HMM分词。

HMM模型主要是将分词问题视为一个序列标注（sequence labeling）问题，其中，句子为观测序列，分词结果为状态序列。首先通过语料训练出HMM相关的模型，然后利用Viterbi算法进行求解，最终得到最优的状态序列，然后再根据状态序列，输出分词结果。
```
def __cut_DAG(self, sentence):  
        DAG = self.get_DAG(sentence)  # 构建有向无环图  
        route = {}  
        self.calc(sentence, DAG, route)  # 动态规划计算最大概率路径  
        x = 0  
        buf = ''  
        #句子长度  
        N = len(sentence)  
        #遍历句子  
        while x < N:  
            y = route[x][1] + 1  #确定词语AB中的B坐标y  
            l_word = sentence[x:y] #提取词语AB  
            if y - x == 1:  #如果词语的长度为1  
                buf += l_word  #buf中放进词语  
            else:  
                if buf:  
                 #如果buf不为空，长度为1，代表里面已经放入过词，使用迭代器yield返回当前值  
                    if len(buf) == 1:  
                        yield buf  
                        buf = ''  
                    else:  
                     #buf使用HMM的分词，对这些未识别成功的词进行标注  
                        if not self.FREQ.get(buf):  
                            recognized = finalseg.cut(buf)  
                            for t in recognized:  
                                yield t  
                        else:  
                            for elem in buf:  
                                yield elem  
                        buf = ''  
                #使用迭代器yield返回当前词语  
                yield l_word  
            #下次开始寻词从y处开始，x到y便是分的词  
            x = y  
        if buf:  
            if len(buf) == 1:  
                yield buf  
            elif not self.FREQ.get(buf):  
                recognized = finalseg.cut(buf)  
                for t in recognized:  
                    yield t  
            else:  
                for elem in buf:  
                    yield elem
```
其中，HMM模型主要是将分词问题视为一个序列标注问题，其中，句子为观测序列，分词结果为状态序列。首先通过语料训练出HMM相关的模型，然后利用Viterbi算法进行求解，最终得到最优的状态序列，然后再根据状态序列，输出分词结果。

### cut精确切分模式
在整个jieba分词组件中，默认的就是精确分词模式。

如【精确模式】：
我/来到/北京/清华大学

如下面的源代码所示，默认调用HMM来执行未登录词识别。
```
    def cut(self, sentence, cut_all=False, HMM=True):  
   
        # 解码为unicode  
        sentence = strdecode(sentence)  
        # 不同模式下的正则  
        if cut_all:  
            re_han = re_han_cut_all  #re.compile("([\u4E00-\u9FD5]+)", re.U)  
            re_skip = re_skip_cut_all #re.compile("[^a-zA-Z0-9+#\n]", re.U)  
        else:  
            re_han = re_han_default #re.compile("([\u4E00-\u9FD5a-zA-Z0-9+#&\._]+)", re.U)  
            re_skip = re_skip_default #re.compile("(\r\n|\s)", re.U)  
        #设置不同模式下的cut_block分词方法  
        if cut_all:  
            cut_block = self.__cut_all  #全模式  
        elif HMM:  
            cut_block = self.__cut_DAG #精确模式，使用隐马模型  
        else:  
            cut_block = self.__cut_DAG_NO_HMM #精确模式，不使用隐马模型  
        #先用正则对句子进行切分  
        blocks = re_han.split(sentence)  
        for blk in blocks:  
            if not blk:  
                continue  
            if re_han.match(blk): # re_han匹配的串  
                for word in cut_block(blk): #根据不同模式的方法进行分词  
                    yield word  
            else: # 按照re_skip正则表对blk进行重新切分  
                tmp = re_skip.split(blk) # 返回list  
                for x in tmp:  
                    if re_skip.match(x):  
                        yield x  
                    elif not cut_all: # 精准模式下逐个字符输出  
                        for xx in x:  
                            yield xx  
                    else:  
                        yield x  
```
我们可以来看下具体的操作流程：

首先，初始化。加载词典文件，获取每个词语和它出现的词数。给定待分词的句子, 使用正则(re_han)获取匹配的中文字符(和英文字符)切分成的短语列表；
```
re_eng = re.compile('[a-zA-Z0-9]', re.U)

re_han_default = re.compile("([\u4E00-\u9FD5a-zA-Z0-9+#&\._%\-]+)", re.U)

re_skip_default = re.compile("(\r\n|\s)", re.U)  
```

其次，利用get_DAG(sentence)函数获得待切分句子的DAG。通过字符串匹配，构建所有可能的分词情况的有向无环图，也就是DAG。

接着，构建节点最大路径概率，以及结束位置。计算每个汉字节点到语句结尾的所有路径中的最大概率，并记下最大概率时在DAG中对应的该汉字成词的结束位置。

然后，构建切分组合。根据节点路径，得到词语切分的结果，也就是分词结果。

其次，HMM新词处理。对于新词，也就是jieba词典中没有的字，采用了HMM组合一个buf单元进行处理。

最后，返回分词结果：通过yield将上面步骤中切分好的词语逐个返回。

### cut_all全切分模式
与精确匹配这种常规分词，一个句子只会得到一种分词结果不同。jieba还提供了全模式分词模式，即把句子中所有的可以成词的词语都扫描出来。

这种场景可以直接用于候选生成全集场景。

如【全模式】：
我/来到/北京/清华/清华大学/华大/大学

在具体实现上，先构建DAG有向无环图，然后根据头尾位置进行匹配输出。

实现如下：
```
 def __cut_all(self, sentence):  
        dag = self.get_DAG(sentence)  
        old_j = -1  
        eng_scan = 0  
        eng_buf = u''  
        for k, L in iteritems(dag):  
            if eng_scan == 1 and not re_eng.match(sentence[k]):  
                eng_scan = 0  
                yield eng_buf  
            if len(L) == 1 and k > old_j:  
                word = sentence[k:L[0] + 1]  
                if re_eng.match(word):  
                    if eng_scan == 0:  
                        eng_scan = 1  
                        eng_buf = word  
                    else:  
                        eng_buf += word  
                if eng_scan == 0:  
                    yield word  
                old_j = L[0]  
            else:  
                for j in L:  
                    if j > k:  
                        yield sentence[k:j + 1]  
                        old_j = j  
        if eng_scan == 1:  
            yield eng_buf  
```

## cut_for_search搜索引擎模式
此外，针对搜索引擎分词，得到所有有意义的ngram成分，jieba还提供了搜索引擎模式。

如【搜索引擎模式】：我/来到/北京/清华/华大/大学/清华大学

思想很简单：在精确模式的基础上，对长词再次切分，提高召回率。即针对切分好的长词，如果词语长度大于2，那么从词表中匹配出所有二字词，如果词语长度大于3，那么则进一步地从词表中匹配出所有三字词。

```
    def cut_for_search(self, sentence, HMM=True):  
        """  
        Finer segmentation for search engines.  
        """  
        words = self.cut(sentence, HMM=HMM)  
        for w in words:  
            if len(w) > 2:  
                for i in xrange(len(w) - 1):  
                    gram2 = w[i:i + 2]  
                    if self.FREQ.get(gram2):  
                        yield gram2  
            if len(w) > 3:  
                for i in xrange(len(w) - 2):  
                    gram3 = w[i:i + 3]  
                    if self.FREQ.get(gram3):  
                        yield gram3  
            yield w  

```

## 实战
引入 pom 依赖
```xml
<dependency>
   	<groupId>com.huaban</groupId>
    <artifactId>jieba-analysis</artifactId>
     <version>1.0.2</version>
 </dependency>
```
普通分词实现代码
```
import com.huaban.analysis.jieba.JiebaSegmenter;

import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

public class JiebaDemo {

    public static void main(String[] args) {
        String content = "《开端》《镜双城》《淘金》三部热播剧均有她，你发现了吗？";
        List<String> stop_words = Arrays.asList("，","？","《","》","》《");
        JiebaSegmenter segmenter = new JiebaSegmenter();
        List<String> result = segmenter.sentenceProcess(content);
        System.out.println("没有过滤停用词======" + result);
        result = result.stream().map(o -> o.trim()).filter(o -> !stop_words.contains(o)).collect(Collectors.toList());
        System.out.println("过滤停用词=========" + result);
    }
}
```

加载自定义词典

自定义 词典 dict.txt
```
杨京京 3 n
张三 3 n
李四 3 nr
王五 3 nr
```
加载自定义词典
也就是通过 WordDictionary加载自定义的词典
```
import com.huaban.analysis.jieba.JiebaSegmenter;
import com.huaban.analysis.jieba.WordDictionary;

import java.nio.file.FileSystems;
import java.nio.file.Path;
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

public class JiebaDemo {

    public static void main(String[] args) {
        // 加载自定义的词典
        Path path = FileSystems.getDefault().getPath("D:/files", "dict.txt");
        WordDictionary.getInstance().loadUserDict(path);

        String content = "《开端》《镜双城》《淘金》三部热播剧均有她，你发现了吗？";
        List<String> stop_words = Arrays.asList("，","？","《","》","》《");
        JiebaSegmenter segmenter = new JiebaSegmenter();
        List<String> result = segmenter.sentenceProcess(content);
        System.out.println("没有过滤停用词======" + result);
        result = result.stream().map(o -> o.trim()).filter(o -> !stop_words.contains(o)).collect(Collectors.toList());
        System.out.println("过滤停用词=========" + result);
    }
}
```

## 评测方法

### 黄金标准/Golden standard
所谓的黄金标准是指：评价一个分词器分词结果的好坏，必然要有一份“公认正确”的分词结果数据来作为参照。 通常，我们使用一份人工标注的数据作为黄金标准。但是，就算是人工标注的数据，每个人对同一句话的分词结果恐怕也持有不同的意见，例如，有一句话“科学技术是第一生产力”，有人说应该这样分词：“科学技术 是 第一 生产力”，又有人说应该这样分词：“科学 技术 是 第一 生产力”。那么，到底哪种才是对的呢？

因此，要找有权威的分词数据来做为黄金标准。

大家可以使用SIGHAN（国际计算语言学会（ACL）中文语言处理小组）举办的国际中文语言处理竞赛Second International Chinese Word Segmentation Bakeoff（
http://sighan.cs.uchicago.edu/bakeoff2005/） 所提供的公开数据来评测，它包含了多个测试集以及对应的黄金标准分词结果。

评价指标

精度（Precision）、召回率（Recall）、F值（F-mesure）是用于评价一个信息检索系统的质量的3个主要指标，以下分别简记为P，R和F。同时，还可以把错误率（Error Rate）作为分词效果的评价标准之一（以下简记为ER）。

直观地说，精度表明了分词器分词的准确程度；召回率也可认为是“查全率”，表明了分词器切分正确的词有多么全；F值综合反映整体的指标；错误率表明了分词器分词的错误程度。

P、R、F越大越好，ER越小越好。一个完美的分词器的P、R、F值均为1，ER值为0。

通常，召回率和精度这两个指标会相互制约。

例如，还是拿上面那句话作为例子：“科学技术是第一生产力”（黄金标准为“科学技术 是 第一 生产力”），假设有一个分词器很极端，把几乎所有前后相连的词的组合都作为分词结果，就像这个样子：“科学 技术 科学技术 是 是第一 第一生产力 生产力”，那么毫无疑问，它切分正确的词已经覆蓋了黄金标准中的所有词，即它的召回率（Recall）很高。但是由于它分错了很多词，因此，它的精度（Precision）很低。

因此，召回率和精度这二者有一个平衡点，我们希望它们都是越大越好，但通常不容易做到都大。
文章来源：http://www.codelast.com/
为了陈述上述指标的计算方法，先定义如下数据：
N：黄金标准分割的单词数
e：分词器错误标注的单词数
c：分词器正确标注的单词数

## 源码
分词过程如下
根据默认词典进行前缀词典初始化
基于前缀词典实现高效的词图扫描，生成句子中汉字所有可能成词情况所构成的有向无环图
采用了动态规划查找最大概率路径, 找出基于词频的最大切分组合
对于未登录词，采用了基于汉字成词能力的 HMM 模型。

初始化
WordDictionary数据字典。 用于获得词频和词频总数。其中FREQ是一个dict,存储词频。

生成所有成词情况的有向无环图DAG
```
    private Map<Integer, List<Integer>> createDAG(String sentence) {
    // #用字典存储有向无环图，key表示当前节点，values为list,存储所有指向点如{1:[1,2,3]}
        Map<Integer, List<Integer>> dag = new HashMap();
        DictSegment trie = wordDict.getTrie();
        char[] chars = sentence.toCharArray();
        int N = chars.length;
        int i = 0;
        int j = 0;

        while(true) {
            while(i < N) {
                Hit hit = trie.match(chars, i, j - i + 1);
                if (!hit.isPrefix() && !hit.isMatch()) {
                    ++i;
                    j = i;
                } else {
                    if (hit.isMatch()) {
                        if (!dag.containsKey(i)) {
                            List<Integer> value = new ArrayList();
                            dag.put(i, value);
                            value.add(j);
                        } else {
                            ((List)dag.get(i)).add(j);
                        }
                    }

                    ++j;
                    if (j >= N) {
                        ++i;
                        j = i;
                    }
                }
            }

            for(i = 0; i < N; ++i) {
                if (!dag.containsKey(i)) {
                    List<Integer> value = new ArrayList();
                    value.add(i);
                    dag.put(i, value);
                }
            }

            return dag;
        }
    }
```
该函数传入一个句子，并返回这个句子所有可能分词情况的DAG。用dict存储的图，key表示字符串起点下标，value代表字符串末尾的下表。对“我爱重庆”计算DAG,得到图中结果，分词的所有情况有：我、爱、爱重、重、重庆、庆。然后要做的是从所有可能的分词结果中找到最好的一个。

计算最优概率路径
目标：最大化句子的概率
```

    private Map<Integer, Pair<Integer>> calc(String sentence, Map<Integer, List<Integer>> dag) {
        int N = sentence.length();
        HashMap<Integer, Pair<Integer>> route = new HashMap<Integer, Pair<Integer>>();
        route.put(N, new Pair<Integer>(0, 0.0));
        for (int i = N - 1; i > -1; i--) {
            Pair<Integer> candidate = null;
            for (Integer x : dag.get(i)) {
                double freq = wordDict.getFreq(sentence.substring(i, x + 1)) + route.get(x + 1).freq;
                if (null == candidate) {
                    candidate = new Pair<Integer>(x, freq);
                }
                else if (candidate.freq < freq) {
                    candidate.freq = freq;
                    candidate.key = x;
                }
            }
            route.put(i, candidate);
        }
        return route;
    }

```
根据代码，我们知道作者使用了基于unigram的语言模型来评估分词结果的好坏。这里为了防止溢出，方便计算，对概率取对数log(FREQ[word]/total)。

对于较长的句子复杂度是比较高的，所以这里使用了维特比算法（动态规划）计算最优路径，从右往左计算。直接对route进行修改，无需return。route[N]存储以N开头，最优分词的结果末尾字符的下标，即sentence[N:route[N][1]]即为单词。

分词函数
```
   public List<String> sentenceProcess(String sentence) {
        List<String> tokens = new ArrayList<String>();
        int N = sentence.length();
        // 计算DAG
        Map<Integer, List<Integer>> dag = createDAG(sentence);
        // 计算最优概率路径
        Map<Integer, Pair<Integer>> route = calc(sentence, dag);

        // 切词 
        int x = 0;
        int y = 0;
        String buf;
        StringBuilder sb = new StringBuilder();
        while (x < N) {
            y = route.get(x).key + 1;
            String lWord = sentence.substring(x, y);
            if (y - x == 1)
                sb.append(lWord);
            else {
                if (sb.length() > 0) {
                    buf = sb.toString();
                    sb = new StringBuilder();
                    if (buf.length() == 1) {
                        tokens.add(buf);
                    }
                    else {
                        if (wordDict.containsWord(buf)) {
                            tokens.add(buf);
                        }
                        else {
                            finalSeg.cut(buf, tokens);
                        }
                    }
                }
                tokens.add(lWord);
            }
            x = y;
        }
        buf = sb.toString();
        if (buf.length() > 0) {
            if (buf.length() == 1) {
                tokens.add(buf);
            }
            else {
                if (wordDict.containsWord(buf)) {
                    tokens.add(buf);
                }
                else {
                    finalSeg.cut(buf, tokens);
                }
            }

        }
        return tokens;
    }
```
总结一下，得到的最优分词路径中，主要有两种情况，单个字和单词。
- 对于单词或单个字+单词，直接返回结果。
- 对于连续多个字单独成词的情况，将他们全部存入buf。若buf不是词典中的词，使用HMM模型处理。若buf是词典中的词，证明通过计算的确单独成词效果更好，返回每个字作为词。
- _cut_DAG是属于精准模式+HMM,jieba还有另外的模式





