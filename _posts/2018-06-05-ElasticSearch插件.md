---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch插件
插件是用户以自定义方式增强Elasticsearch功能的一种方法。Elasticsearch插件包括添加自定义映射类型、自定义分析器、自定义脚本引擎和自定义发现等。

## 插件简介
Elasticsearch插件类型包含jar文件、脚本和配置文件，插件必须安装在集群中的每个节点上才能使用。安装插件后，必须重新启动每个节点，才能看到插件。

在Elasticsearch官网上，插件被归纳为两大类，分别是核心插件和社区贡献插件。

## 核心插件
核心插件属于Elasticsearch项目，插件与Elasticsearch安装包同时提供，插件的版本号始终与Elasticsearch安装包的版本号相同。这些插件是由Elasticsearch团队维护的。

## 社区贡献插件
社区贡献插件属于Elasticsearch项目外部的插件。这些插件由单个开发人员或私人公司提供，并拥有各自的许可证及各自的版本控制系统。

## 插件管理
Elasticsearch提供了用于安装、查看和删除插件相关的命令，这些命令默认位于$es_home/bin目录中。

用户可以运行以下命令获取插件命令的使用说明：
```shell
sudo bin/elasticsearch -plugin -h
```
### 插件位置指定

当在根目录中运行Elasticsearch时，如果使用DEB或RPM包安装了Elasticsearch，则以根目录运行/usr/share/Elasticsearch/bin/Elasticsearch-plugin，以便Elasticsearch可以写入磁盘的相应文件，否则需要以拥有所有Elasticsearch文件的用户身份运行bin/Elasticsearch插件。


## 分析插件
对分析器（Analyzer）而言，一般会接受一个字符串作为输入参数，分析器会将这个字符串拆分成独立的词或语汇单元（也称之为token）。当然，在处理过程中会丢弃一些标点符号等字符，处理后会输出一个语汇单元流（也称之为token stream）。
因此，一般分析器会包含三个部分：
- character filter：分词之前的预处理，过滤HTML标签、特殊符号转换等。
- tokenizer：用于分词。
- token filter：用于标准化输出。

Elasticsearch为很多语言提供了专用的分析器，特殊语言所需的分析器可以由用户根据需要以插件的形式提供。Elasticsearch内置的主要分析器有：
- Standard分析器：默认的分词器。Standard分析器会将词汇单元转换成小写形式，并且去除了停用词和标点符号，支持中文（采用的方法为单字切分）。停用词指语气助词等修饰性词语，如the、an、的、这等。
- Simple分析器：首先通过非字母字符分割文本信息，并去除数字类型的字符，然后将词汇单元统一为小写形式。
- Whitespace分析器：仅去除空格，不会将字符转换成小写形式，不支持中文；不对生成的词汇单元进行其他标准化处理。
- Stop分析器：与Simple分析器相比，增加了去除停用词的处理。
- Keyword分析器：该分析器不进行分词，而是直接将输入作为一个单词输出。
- Pattern分析器：该分析器通过正则表达式自定义分隔符，默认是“\W+”，即把非字词的符号作为分隔符。
- Language分析器：这是特定语言的分析器，不支持中文，支持如English、French和Spanish等语言。

通常来说，任何全文检索的字符串域都会默认使用Standard分析器，如果想要一个自定义分析器，则可以按照如下方式重新制作一个“标准”分析器：
```shell

```
在这个自定义分析器中，主要使用了Lowercase （小写字母）和Stop （停用词） 词汇单元过滤器。

### 什么是Standard分析器呢？
一般来说，分析器会接受一个字符串作为输入。在工作时，分析器会将这个字符串拆分成独立的词或语汇单元（称之为token），当然也会丢弃一些标点符号等字符，最终分析器输出一个语汇单元流。这就是典型的Standard分析器的工作模式。

分析器在识别词汇时有多种算法可供选择。最简单的是Whitespace分词算法，该算法按空白字符，如空格、Tab、换行符等，对语句进行简单的拆分，将连续的非空格字符组成一个语汇单元。例如，对下面的语句使用Whitespace分词算法分词时，会得到如下结果：
```
原文：You're the 1st runner home！

结果：You're、the、1st、runner、home！
```
而Standard分析器则使用Unicode文本分割算法寻找单词之间的界限，并输出所有界限之间的内容。Unicode内包含的知识使其可以成功地对包含混合语言的文本进行分词。

一般来说，Standard分析器是大多数语言分词的一个合理的起点。事实上，它构成了大多数特定语言分析器的基础，如English分析器、French分析器和Spanish分析器。另外，它还支持亚洲语言，只是有些缺陷，因此读者可以考虑通过ICU分析插件的方式使用icu_tokenizer进行替换。

## Elasticsearch中的分析插件
分析插件是一类插件，我们可通过向Elasticsearch中添加新的分析器、标记化器、标记过滤器或字符过滤器等扩展Elasticsearch的分析功能。

Elasticsearch官方提供的核心分析插件如下。
- ICU库。
使用ICU库扩展对Unicode的支持，包括更好地分析亚洲语言、Unicode规范化、支持Unicode的大小写折叠、支持排序和音译。
- Kuromoji插件。
使用Kuromoji插件对日语进行高级分析。
- Lucene Nori插件。
使用Lucene Nori插件对韩语进行分析。
- Phonetic插件。
使用Soundex、Metaphone、Caverphone和其他编码器/解码器将标记分析为其语音等价物。
- SmartCN插件。
SmartCN插件可用于对中文或中英文混合文本进行分析。该插件利用概率知识对简化中文文本进行最优分词。首先文本被分割成句子，然后每个句子再被分割成单词。
- Stempel插件。
Stempel插件为波兰语提供了高质量的分析工具。
- Ukrainian插件。
Ukrainian插件可用于为乌克兰语提供词干分析。

除官方的分析插件外，Elasticsearch技术社区也贡献了不少分析插件，比较常用且著名的有：
- IK Analysis Plugin.
IK分析插件将Lucene IK Analyzer集成到Elasticsearch中，支持读者自定义字典。
- Pinyin Analysis Plugin.
Pinyin Analysis Plugin是一款拼音分析插件，该插件可对汉字和拼音进行相互转换。
- Vietnamese Analysis Plugin.
Vietnamese Analysis Plugin是一款用于对越南语进行分析的插件。
- Network Addresses Analysis Plugin.
Network Addresses Analysis Plugin可以用于分析网络地址。
- Dandelion Analysis Plugin.
Dandelion Analysis Plugin可译为蒲公英分析插件，该插件提供了一个分析器（称为“蒲公英-A”），该分析器会从输入文本中提取的实体进行语义搜索。
- STConvert Analysis Plugin.
STConvert Analysis Plugin可对中文简体和繁体进行相互转换。

## ICU分析插件
ICU分析插件将Lucene ICU模块集成到Elasticsearch中。
插件必须安装在群集中的每个节点上，并且必须在安装后重新启动节点。在Linux环境下，可以通过如下命令安装ICU分析插件：
```shell
bin/elasticsearch-plugin install analysis-icu
```
当需要删除ICU分析插件时，在Linux环境下，可以通过如下命令删除ICU分析插件：
```shell
bin/elasticsearch-plugin remove analysis-icu
```
### ICU插件的规范化字符过滤器
ICU插件的规范化字符过滤器主要用于规范化字符，可用于所有索引，无须进一步配置。规范化的类型可以由name参数指定，该参数接受NFC、NFKC和NFKC_CF方法，默认为NFKC_CF方法。模式参数还可以设置为分解，分别将NFC转换为NFD，或将NFKC转换为NFKD。

在使用时，可以通过指定unicode_set_filter参数控制规范化的字母，该参数接受unicodeset。
下面通过一个示例，展示字符过滤器的默认用法和自定义字符过滤器：


## 自带的几种分词器
- standard	标准标记器，标准过滤器，小写过滤器，停止过滤器
- simple	小写的分词器
- stop	小写标记器，停止过滤器
- keyword	不分词，内容整体作为一个值
- whitespace	以空格分词
- language	以语言分词
- snowball	标准标记器，标准过滤器，小写过滤器，停止过滤器，雪球过滤器
- custom	自定义分词。至少需要指定一个 Tokenizer, 零个或多个Token Filter, 零个或多个Char Filter
- pattern	正则分词



