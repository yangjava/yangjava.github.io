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























































