---
layout: post
categories: ElasticSearch
description: none
keywords: ElasticSearch
---
# ElasticSearch入门简介
Elasticsearch是一个分布式、可扩展、近实时的高性能搜索与数据分析引擎。Elasticsearch基于Apache Lucene构建，采用Java编写，并使用Lucene构建索引、提供搜索功能。Elasticsearch的目标是让全文搜索功能的落地变得简单。

## Elasticsearch
Lucene 是开源的搜索引擎工具包，Elasticsearch 充分利用Lucene，并对其进行了扩展，使存储、索引、搜索都变得更快、更容易， 而最重要的是， 正如名字中的“ elastic ”所示， 一切都是灵活、有弹性的。而且，应用代码也不是必须用Java 书写才可以和Elasticsearc兼容，完全可以通过JSON 格式的HTTP 请求来进行索引、搜索和管理Elasticsearch 集群。

Elasticsearch是一个基于Apache Lucene(TM)的开源搜索引擎，无论在开源还是专有领域，Lucene可以被认为是迄今为止最先进、性能最好的、功能最全的搜索引擎库。
但是，Lucene只是一个库。想要发挥其强大的作用，你需使用Java并要将其集成到你的应用中。Lucene非常复杂，你需要深入的了解检索相关知识来理解它是如何工作的。
Elasticsearch也是使用Java编写并使用Lucene来建立索引并实现搜索功能，但是它的目的是通过简单连贯的RESTful API让全文搜索变得简单并隐藏Lucene的复杂性。

不过，Elasticsearch不仅仅是Lucene和全文搜索引擎，它还提供：
- 分布式的实时文件存储，每个字段都被索引并可被搜索
- 实时分析的分布式搜索引擎
- 可以扩展到上百台服务器，处理PB级结构化或非结构化数据
而且，所有的这些功能被集成到一台服务器，你的应用可以通过简单的RESTful API、各种语言的客户端甚至命令行与之交互。上手Elasticsearch非常简单，它提供了许多合理的缺省值，并对初学者隐藏了复杂的搜索引擎理论。它开箱即用（安装即可使用），只需很少的学习既可在生产环境中使用。Elasticsearch在Apache 2 license下许可使用，可以免费下载、使用和修改。
随着知识的积累，你可以根据不同的问题领域定制Elasticsearch的高级特性，这一切都是可配置的，并且配置非常灵活。

## Elasticsearch的特点
- 分布式实时文件存储。Elasticsearch可将被索引文档中的每一个字段存入索引，以便字段可以被检索到。
- 实时分析的分布式搜索引擎。Elasticsearch的索引分拆成多个分片，每个分片可以有零个或多个副本。集群中的每个数据节点都可承载一个或多个分片，并且协调和处理各种操作；负载再平衡和路由会自动完成。
- 高可拓展性。大规模应用方面，Elasticsearch可以扩展到上百台服务器，处理PB级别的结构化或非结构化数据。当然，Elasticsearch也可以运行在单台PC上。
- 可插拔插件支持。Elasticsearch支持多种插件，如分词插件、同步插件、Hadoop插件、可视化插件等。

## 基本概念
Elasticsearch就被设计为能处理数以亿计的文档和每秒数以百计的搜索请求的分布式解决方案。这归功于几个重要的概念，我们现在将更详细地描述。

## 集群（Cluster）
集群是一个或多个节点的集合，它们共同保存您的整个数据，并提供跨所有节点的联合索引和搜索功能。群集由唯一的名称标识，默认情况下为“elasticsearch”。此名称很重要，因为如果节点设置为按名称加入群集，则该节点只能是群集的一部分。

确保不要在不同的环境中重用相同的群集名称，否则很可能会导致节点加入错误的群集。例如，您可以使用logging-dev，logging-stage以及logging-prod用于开发，预发和生产环境。

值得注意的是，如果群集中只有一个节点，那么它是完全正常的。此外，您还可以拥有多个独立的集群，每个集群都有自己唯一的集群名称。

### 节点（Node）
节点是作为集群一部分的单个服务器，存储数据并参与集群的索引和搜索功能。就像集群一样，节点由名称标识，默认情况下，该名称是在启动时分配给节点的随机通用唯一标识符（UUID）。如果不需要默认值，可以定义所需的任何节点名称。此名称对于管理目的非常重要，您可以在其中识别网络中哪些服务器与Elasticsearch集群中的哪些节点相对应。

可以将节点配置为按集群名称加入特定群集。默认情况下，每个节点都设置为加入一个名为cluster的集群elasticsearch，这意味着如果您在网络上启动了许多节点并且假设它们可以相互发现，它们将自动形成并加入一个名为elasticsearch的集群。

在单个集群中，可以拥有任意数量的节点。此外，如果网络上当前没有其他Elasticsearch节点正在运行，则默认情况下启动单个节点将形成一个名为elasticsearch的新单节点集群。

### 索引（Index）
索引是具有某些类似特征的文档集合。 例如，可以给客户数据建立一个索引，给产品目录建立另一个索引或给订单数据的建立一个索引。

索引有一个名称标识（必须全部小写），此名称用于在对其中的文档执行索引，搜索，更新和删除操作时引用索引所用。 在单个群集中，您可以根据需要定义任意数量的索引。

索引 （index）是Elasticsearch对逻辑数据的逻辑存储，所以它可以分为更小的部分。你可以把索引看成关系型数据库的表。然而，索引的结构是为快速有效的全文索引准备的，特别是它不存储原始值。

Elasticsearch可以把索引存放在一台机器或者分散在多台服务器上，每个索引有一或多个分片 （shard），每个分片可以有多个副本 （replica）。

### 类型（Type）
type是一个逻辑意义上的分类或者叫分区，允许在同一索引中建立多个type。本质是相当于一个过滤条件，高版本将会废弃type概念。

不同_type下的字段不能冲突，删除整个_type也不会释放空间。在实际应用中，数据模型不同，有不同_type需求的时候，我们应该建立单独的索引，而不是在一个索引下使用不同的_type。删除过期老化的数据时，最好以索引为单位，而不是_type和_id。正由于_type在实际应用中容易引起概念混淆，以及允许索引存在多_type并没有什么实际意义，在ES 6.x版本中，一个索引只允许存在一个_type，未来的7.x版本将完全删除_type的概念。

### 文档（Document）
文档是可以编制索引的基本信息单元。例如，您可以为单个客户提供文档，为单个产品提供另一个文档，为单个订单提供另一个文档。该文档以JSON（JavaScript Object Notation）表示，JSON是一种普遍存在的互联网数据交换格式。

ES是面向文档的。各种文本内容以文档的形式存储到ES中，文档可以是一封邮件、一条日志，或者一个网页的内容。一般使用 JSON 作为文档的序列化格式，文档可以有很多字段，在创建索引的时候，我们需要描述文档中每个字段的数据类型，并且可能需要指定不同的分析器，就像在关系型数据中“CREATE TABLE”一样。

在索引/类型中，可以根据需要存储任意数量的文档。 值得注意的是，尽管文档实际上驻留在索引中，但实际上必须将文档分配给索引中的一个类型（type）中。

在存储结构上，由_index、_type和_id唯一标识一个文档。

_index指向一个或多个物理分片的逻辑命名空间，_type类型用于区分同一个集合中的不同细分，在不同的细分中，数据的整体模式是相同或相似的，不适合完全不同类型的数据。多个_type可以在相同的索引中存在，只要它们的字段不冲突即可（对于整个索引，映射在本质上被“扁平化”成一个单一的、全局的模式）。_id文档标记符由系统自动生成或使用者提供。

存储在Elasticsearch中的主要实体叫文档 （document）。用关系型数据库来类比的话，一个文档相当于数据库表中的一行记录。Elasticsearch的文档中，相同字段必须有相同类型。这意味着，所有包含title 字段的文档，title 字段类型都必须一样，比如string 。

文档由多个字段 组成，每个字段可能多次出现在一个文档里，这样的字段叫多值字段 （multivalued）。每个字段有类型，如文本、数值、日期等。字段类型也可以是复杂类型，一个字段包含其他子文档或者数组。字段类型在Elasticsearch中很重要，因为它给出了各种操作（如分析或排序）如何被执行的信息。幸好，这可以自动确定，然而，我们仍然建议使用映射。与关系型数据库不同，文档不需要有固定的结构，每个文档可以有不同的字段，此外，在程序开发期间，不必确定有哪些字段。当然，可以用模式强行规定文档结构。

从客户端的角度看，文档是一个JSON对象。每个文档存储在一个索引中并有一个Elasticsearch自动生成的唯一标识符和文档类型 。文档需要有对应文档类型的唯一标识符，这意味着在一个索引中，两个不同类型的文档可以有相同的唯一标识符。

### 分片（Shards）
索引可能存储大量可能超过单个节点硬件限制的数据。例如，占用1TB磁盘空间的十亿个文档的单个索引可能不适合单个节点的磁盘，或者可能太慢而无法单独从单个节点提供搜索请求。

为了解决这个问题，Elasticsearch提供了将索引细分为多个称为分片的功能。创建索引时，只需定义所需的分片数即可。每个分片本身都是一个功能齐全且独立的“索引”，可以托管在集群中的任何节点上。

分片很重要，主要有两个原因：
- 它允许集群进行水平扩展；
- 它允许集群跨分片（可能在多个节点上）分布和并行化操作，从而提高性能和吞吐量。
分片的分布方式以及如何将其文档聚合回搜索请求的机制完全由Elasticsearch管理，对用户而言是透明的。

当有大量的文档时，由于内存的限制、硬盘能力、处理能力不足、无法足够快地响应客户端请求等，一个节点可能不够。在这种情况下，数据可以分为较小的称为分片 （shard）的部分（其中每个分片都是一个独立的Apache Lucene索引）。每个分片可以放在不同的服务器上，因此，数据可以在集群的节点中传播。当你查询的索引分布在多个分片上时，Elasticsearch会把查询发送给每个相关的分片，并将结果合并在一起，而应用程序并不知道分片的存在。此外，多个分片可以加快索引。

在分布式系统中，单机无法存储规模巨大的数据，要依靠大规模集群处理和存储这些数据，一般通过增加机器数量来提高系统水平扩展能力。因此，需要将数据分成若干小块分配到各个机器上。然后通过某种路由策略找到某个数据块所在的位置。

### 副本（Replicas）
为了提高查询吞吐量或实现高可用性，可以使用分片副本。副本 （replica）只是一个分片的精确复制，每个分片可以有零个或多个副本。换句话说，Elasticsearch可以有许多相同的分片，其中之一被自动选择去更改索引操作。这种特殊的分片称为主分片 （primary shard），其余称为副本分片 （replica shard）。在主分片丢失时，例如该分片数据所在服务器不可用，集群将副本提升为新的主分片。

除了将数据分片以提高水平扩展能力，分布式存储中还会把数据复制成多个副本，放置到不同的机器中，这样一来可以增加系统可用性，同时数据副本还可以使读操作并发执行，分担集群压力。但是多数据副本也带来了一致性的问题：部分副本写成功，部分副本写失败。

为了应对并发更新问题，ES将数据副本分为主从两部分，即主分片（primary shard）和副分片（replica shard）。主数据作为权威数据，写过程中先写主分片，成功后再写副分片，恢复阶段以主分片为准。

分片（shard）是底层的基本读写单元，分片的目的是分割巨大索引，让读写可以并行操作，由多台机器共同完成。读写请求最终落到某个分片上，分片可以独立执行读写工作。ES利用分片将数据分发到集群内各处。分片是数据的容器，文档保存在分片内，不会跨分片存储。分片又被分配到集群内的各个节点里。当集群规模扩大或缩小时，ES 会自动在各节点中迁移分片，使数据仍然均匀分布在集群里。

### 文档类型
在Elasticsearch中，一个索引对象可以存储很多不同用途的对象。例如，一个博客应用程序可以保存文章和评论。文档类型让我们轻易地区分单个索引中的不同对象。每个文档可以有不同的结构，但在实际部署中，将文件按类型区分对数据操作有很大帮助。当然，需要记住一个限制，不同的文档类型不能为相同的属性设置不同的类型。例如，在同一索引中的所有文档类型中，一个叫title 的字段必须具有相同的类型。

### 映射
文档中的每个字段都必须根据不同类型做相应的分析。举例来说，对数值字段和从网页抓取的文本字段有不同的分析，比如前者的数字不应该按字母顺序排序，后者的第一步是忽略HTML标签，因为它们是无用的信息噪音。Elasticsearch在映射中存储有关字段的信息。每一个文档类型都有自己的映射，即使我们没有明确定义。

## 分析器
对于字符串类型的字段，可以指定Elasticsearch应该使用哪个分析器。使用分析器时，只需在指定字段的正确属性上设置它的名字，就这么简单。

Elasticsearch允许我们使用众多默认定义的分析器中的一种。如下分析器可以开箱即用。
- standard ：方便大多数欧洲语言的标准分析器（关于参数的完整列表，请参阅http://www.lasticsearch.org/guide/en/elasticsearch/reference/current/analysis-standard-analyzer.html ）。
- simple ：这个分析器基于非字母字符来分离所提供的值，并将其转换为小写形式。
- whitespace ：这个分析器基于空格字符来分离所提供的值。
- stop ：这个分析器类似于simple 分析器，但除了simple 分析器的功能，它还能基于所提供的停用词（stop word）过滤数据（参数的完整列表，请参阅http://www.elasticsearch.rg/guide/en/elasticsearch/reference/current/analysis-stop-analyzer.html ）。
- keyword ：这是一个非常简单的分析器，只传入提供的值。你可以通过指定字段为not_analyzed 来达到相同的目的。
- pattern ：这个分析器通过使用正则表达式灵活地分离文本（参数的完整列表，请参阅http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/analysis-pattern-analyzer.html ）。
- language ：这个分析器旨在特定的语言环境下工作。该分析器所支持语言的完整列表可参考http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.tml 。
- snowball ：这个分析器类似于standard 分析器，但提供了词干提取算法（stemming algorithm，参数的完整列表请参阅http://www.elasticsearch.org/guide/en/elasticsearch/eference/current/analysis-snowball-analyzer.html ）。

## Elasticsearch的安装与配置
常言道：工欲善其事，必先利其器。因此在使用Elasticsearch之前，我们需要安装Elasticsearch。下面介绍Elasticsearch在Windows环境下和在Linux环境下的安装方法。

## 在Windows环境下安装
在Windows系统中，我们可以基于Windows下的zip安装包来构建Elasticsearch服务。该zip安装包附带了一个elasticsearch-service.bat命令文件，执行该命令文件，即可将Elasticsearch作为服务运行。

在elasticsearch-7.2.0文件夹中有bin、config、jdk、lib、logs、modules、plugins和文件夹。
- bin文件夹下存放的是二进制脚本，包括启动Elasticsearch节点和安装的Elasticsearch插件。
- config文件夹下存放的是包含elasticsearch.yml在内的配置文件。
- jdk文件夹下存放的是Java运行环境。
- lib文件夹下存放的是Elasticsearch自身所需的jar文件。
- logs文件夹下存放的是日志文件。
- modules文件夹下存放的是Elasticsearch的各个模块。
- plugins文件夹下存放的是配置插件，每个插件都包含在一个子目录中。
启动Elasticsearch服务。当看到节点started的输出后，说明Elasticsearch服务已经启动。节点已经启动，并且选举它自己作为单个集群中的Master主节点。
- Elasticsearch启动后，在默认情况下，Elasticsearch将在前台运行，并将其日志打印到标准输出（stdout）。可以按Ctrl+C组合键停止运行Elasticsearch。

在Elasticsearch运行过程中，如果需要将Elasticsearch作为守护进程运行，则需要在命令行上指定命令参数“-d”，并使用 “-p”选项将Elasticsearch的进程ID记录在文件中。此时的启动命令如下：
```shell
./bin/elasticsearch -d -p pid 
```
此时Elasticsearch的日志消息可以在$ES_HOME/logs/目录中找到。
在启动Elasticsearch的过程中，我们可以通过命令行对Elasticsearch进行配置。一般来说，在默认情况下，Elasticsearch会从$ ES_HOME/config/elasticsearch.yml文件加载其配置内容。我们还可以在命令行上指定配置，此时需要使用“-e”语法。在命令行配置Elasticsearch参数时，启动命令如下：
```shell
./bin/elasticsearch -d -ECluster.name=my_cluster  
```
在Elasticsearch启动后，我们可以在浏览器的地址栏输入http://localhost:9200/来验证Elasticsearch的启动情况。
此外，我们还可以设置Elasticsearch是否自动创建x-pack索引。x-pack将尝试在Elasticsearch中自动创建多个索引。在默认情况下，Elasticsearch是允许自动创建索引的，且不需要其他步骤。

如果需要在Elasticsearch中禁用自动创建索引，则必须在Elasticsearch.yml中配置action.auto_create_index。

## Elasticsearch的配置
与近年来很多流行的框架和中间件一样，Elasticsearch的配置同样遵循“约定大于配置”的设计原则。Elasticsearch具有极好的默认值设置，用户仅需要很少的配置即可使用Elasticsearch。用户既可以使用群集更新设置API在正在运行的群集上更改大多数设置，也可以通过配置文件对Elasticsearch进行配置。

一般来说，配置文件应包含特定节点的设置，如node.name和paths路径等信息，还会包含节点为了能够加入Elasticsearch群集而需要做出的设置，如cluster.name和network.host等。

### 配置文件位置信息

在Elasticsearch中有三个配置文件，分别是elasticsearch.yml、jvm.options和log4j2.properties，这些文件位于config目录

其中，elasticsearch.yml用于配置Elasticsearch，jvm.options用于配置Elasticsearch依赖的JVM信息，log4j2.properties用于配置Elasticsearch日志记录中的各个属性。

### 配置文件的格式
Elasticsearch的配置文件格式为yaml。 如果需要在配置文件中引用环境变量的值，则可以在配置文件中使用 ${...}符号。

### 设置JVM选项
在Elasticsearch中，用户很少需要更改Java虚拟机（JVM）选项。一般来说，最可能的更改是设置堆大小。在默认情况下，Elasticsearch设置JVM使用最小堆空间和最大堆空间的大小均为1GB。

设置JVM选项（包括系统属性和jvm标志）的首选方法是通过jvm.options配置文件设置。此文件的默认位置为config/jvm.options。

在Elasticsearch中，我们通过xms（最小堆大小）和xmx（最大堆大小）这两个参数设置jvm.options配置文件指定的整个堆大小，一般应将这两个参数设置为相等。

### 安全设置
在Elasticsearch中，有些设置信息是敏感且需要保密的，此时单纯依赖文件系统权限来保护这些信息是不够的，因此需要配置安全维度的信息。Elasticsearch提供了一个密钥库和相应的密钥库工具来管理密钥库中的设置。这里的所有命令都适用于Elasticsearch用户。

需要指出的是，对密钥库所做的所有修改，都必须在重新启动Elasticsearch之后才会生效。

此外，在当前Elasticsearch密钥库中只提供模糊处理，以后会增加密码保护。

安全设置就像elasticsearch.yml配置文件中的常规设置一样，需要在集群中的每个节点上指定。当前，所有安全设置都是特定于节点的设置，每个节点上必须有相同的值。

安全设置的常规操作有创建密钥库、查看密钥库中的设置列表、添加字符串设置、添加文件设置、删除密钥设置和可重新加载的安全设置等。

## 日志记录配置
在Elasticsearch中，使用log4j2来记录日志。用户可以使用log4j2.properties文件配置log4j2。

Elasticsearch公开了三个属性信息，分别是$sys：es.logs.base_path、$sys：es.logs.cluster_name和$sys：es.logs.node_name，用户可以在配置文件中引用这些属性来确定日志文件的位置。

属性$sys：es.logs.base_path将解析为日志文件目录地址，$sys：es.logs.cluster_name将解析为群集名称（在默认配置中，用作日志文件名的前缀），$sys：es.logs.node_name_将解析为节点名称（如果显式地设置了节点名称）。

例如，假设用户的日志目录（path.logs）是/var/log/elasticsearch，集群命名为production，那么$sys：es.logs.base_path_将解析为/var/log/elasticsearch，$sys：es.logs.base_path/sys：file.Separator/$sys：es.logs.cluster_name.log将解析为/var/log/elasticsearch/production.log。

## docker安装ES
首先拉取镜像：`docker pull elasticsearch:7.12.0`
创建docker容器挂在的目录：
```shell
mkdir -p /opt/elasticsearch/config
mkdir -p /opt/elasticsearch/data
mkdir -p /opt/elasticsearch/plugins
```
配置文件:
```shell
echo "http.host: 0.0.0.0" >> /opt/elasticsearch/config/elasticsearch.yml
```
创建容器:
```yaml
sudo docker run --name elasticsearch -p 9200:9200  -p 9300:9300 \
-e "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms84m -Xmx512m" \
-v /opt/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /opt/elasticsearch/data:/usr/share/elasticsearch/data \
-v /opt/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-d elasticsearch:7.12.0
```
说明:
- -p 端口映射
- -e discovery.type=single-node 单点模式启动
- -e ES_JAVA_OPTS=“-Xms84m -Xmx512m”：设置启动占用的内存范围
- -v 目录挂载
- -d 后台运行

可能会出现的安装异常

异常一：文件夹未设置所有用户读写执行权限，处理：sudo chmod -R 777 /opt/elasticsearch/
```
"stacktrace": ["org.elasticsearch.bootstrap.StartupException: ElasticsearchException[failed to bind service]; nested: AccessDeniedException[/usr/share/elasticsearch/data/nodes];",
"at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:163) ~[elasticsearch-7.12.0.jar:7.12.0]",
"at org.elasticsearch.bootstrap.Elasticsearch.execute(Elasticsearch.java:150) ~[elasticsearch-7.12.0.jar:7.12.0]",
"at org.elasticsearch.cli.EnvironmentAwareCommand.execute(EnvironmentAwareCommand.java:75) ~[elasticsearch-7.12.0.jar:7.12.0]",
"at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:116) ~[elasticsearch-cli-7.12.0.jar:7.12.0]",
"at org.elasticsearch.cli.Command.main(Command.java:79) ~[elasticsearch-cli-7.12.0.jar:7.12.0]",
"at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:115) ~[elasticsearch-7.12.0.jar:7.12.0]",
"at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:81) ~[elasticsearch-7.12.0.jar:7.12.0]",
"Caused by: org.elasticsearch.ElasticsearchException: failed to bind service",
"at org.elasticsearch.node.Node.<init>(Node.java:744) ~[elasticsearch-7.12.0.jar:7.12.0]",
"at org.elasticsearch.node.Node.<init>(Node.java:278) ~[elasticsearch-7.12.0.jar:7.12.0]",
```
异常二：因虚拟内存太少导致，处理：sudo sysctl -w vm.max_map_count=262144

在Elasticsearch启动后，我们可以在浏览器的地址栏输入http://localhost:9200/来验证Elasticsearch的启动情况。
```json
{
  "name" : "0636895502a8",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "rdkOf3_CTQ-7XejmweLToQ",
  "version" : {
    "number" : "7.12.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "78722783c38caa25a70982b5b042074cde5d3b3a",
    "build_date" : "2021-03-18T06:17:15.410153305Z",
    "build_snapshot" : false,
    "lucene_version" : "8.8.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```


## 参考资料
深入理解Elasticsearch

Elasticsearch服务器开发

Elasticsearch 源码解析与优化实战

Elasticsearch实战与原理解析

Elasticsearch: 权威指南 https://www.elastic.co/guide/cn/elasticsearch/guide/current/foreword_id.html

Elasticsearch 基础教程 https://www.knowledgedict.com/tutorial/elasticsearch-intro.html


















































