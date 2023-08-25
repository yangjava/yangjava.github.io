---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码

## 编译调试
将代码克隆到本地
```
git clone https://github.com/cr7258/elasticsearch
```
切换到指定发布版本，这里我们基于 7.14.1 版本进行学习。
```
git checkout v7.14.1
```
我们编译的是 Elasticsearch 7.14.1 版本，在源码根目录下的 CONTRIBUTING.md 文件中说明了 IntelliJ 和 JDK 的版本要求，Gradle 我们可以不必自行安装，在编译的时候会自动使用源码根目录下 gradlew 脚本进行安装。
- IntelliJ 2020.1 以上
- JDK 16
- Gradle 7.1

### 源码导入 IntelliJ Idea
- 打开 IntelliJ Idea，点击 Open or Import。
- 选中 Elasticsearch 源码根目录中的 build.gradle 文件，点击 Open。
- 点击 OPEN AS PROJECT。
- 点击 Event Log，然后点击 Import Gradle Project。

配置参数：
- Gradle user home：选择 Elasticsearch 源码包中的 gradle 目录。
- Gradle JVM：选择安装的 JDK 16。

Elasticsearch启动类`org.elasticsearch.bootstrap.Elasticsearch`

## ElasticSearch源码组织形式
elasticsearch采用google的guice框架进行依赖注入。与spring相比较，guice省去了xml形式的配置文件，而是采用编码到module中进行管理，同时采用标注的形式进行注入。个人觉得这种代码组织方式没有xml文件方式直观，因此给阅读源代码带来一定困难。那么elasticsearch是如何使用的guice进行代码管理的呢？

elasticsearch定义了很多module类来管理不同功能的代码，在实例化InternalNode的过程中，完成了所有代码的初始化、依赖注入等过程。具体每个module类对应的功能可以参考其命名。

## 数据结构
Elasticsearch的底层搜索是以lucene来实现的。其主要是提供了一个分布式的框架来扩展了lucene，从而实现大数据量的，分布式搜索功能。其实现思想很简单，将大数据量分而治之，哈希分成多份，然后对每一份进行“lucene处理”——用lucene索引、检索，最后将每份结果合并返回。

Elasticsearch中的路由信息是分这么四层结构的：
- RoutingTable：是整个集群的总体路由信息。每个Elasticsearch都可以建立多个索引，这里存储的是各个索引的信息。
- IndexRoutingTable：是针对一个索引的路由信息。每个索引可以被拆分成不同的shard，这里存储的是各个shard的信息。
- IndexShardRoutingTable：这个概念不好解释。Elasticsearch的shard是分为主副本的。这里的副本shard可以简单理解为对一个主本shard的备份。他们的数据是相同的，id号是相同的，所不同的是主副本标志位以及所处node等信息。IndexShardRoutingTable中存储针对同一个id号的，所有主副本shard的信息。（可以理解为shard的一个集合。）
- ShardRouting：分片信息，是路由表信息的最小单元，主要保存了shardId，所属index，所属node节点，是否主本等信息。

```
data |
    elasticsearch |
                 node |
                    0 |
                      indices
                        test
                        twitter
                        0 /  1 / 2 
                    1 |
                      indices
                        test
                        twitter  
                        0 /2 /3 /4                  
                    2 |
                      indices
                        test
                        twitter  
                        1 /3 /4                      
```
本集群是3个节点，因此在nodes文件夹下，会有0、1、2文件夹对应不同的node。我建立了两个索引，分别叫做test和twitter。针对twitter索引，分成5个shard，每个shard只有一个副本。因此从图中可以看到，twitter文件夹下的子文件夹应该是：0 0 1 1 2 2 3 3 4 4.他们均衡的分布在三个node上。并且因为其采用了这种文件夹结构，所以针对同一个shard的主副本是不能在同一个node上的。
结合上面的图片，就会有比较清晰的认识。
Elasticsearch同时还提供了另外一种纬度的路由视图是：RoutingNodes——RoutingNode——MutableShardRouting。

## 内部模块简介
在分析内部模块流程之前，我们先了解一下ES中几个基础模块的功能。

- Cluster
Cluster模块是主节点执行集群管理的封装实现，管理集群状态，维护集群层面的配置信息。主要功能如下：
  - 管理集群状态，将新生成的集群状态发布到集群所有节点。
  - 调用allocation模块执行分片分配，决策哪些分片应该分配到哪个节点
  - 在集群各节点中直接迁移分片，保持数据平衡。
  
- allocation
封装了分片分配相关的功能和策略，包括主分片的分配和副分片的分配，本模块由主节点调用。创建新索引、集群完全重启都需要分片分配的过程。

- Discovery
发现模块负责发现集群中的节点，以及选举主节点。当节点加入或退出集群时，主节点会采取相应的行动。从某种角度来说，发现模块起到类似ZooKeeper的作用，选主并管理集群拓扑。

- gateway
负责对收到Master广播下来的集群状态（cluster state）数据的持久化存储，并在集群完全重启时恢复它们。

- Indices
索引模块管理全局级的索引设置，不包括索引级的（索引设置分为全局级和每个索引级）。它还封装了索引数据恢复功能。集群启动阶段需要的主分片恢复和副分片恢复就是在这个模块实现的。

- HTTP
HTTP模块允许通过JSON over HTTP的方式访问ES的API,HTTP模块本质上是完全异步的，这意味着没有阻塞线程等待响应。使用异步通信进行 HTTP 的好处是解决了 C10k 问题（10k量级的并发连接）。 在部分场景下，可考虑使用HTTP keepalive以提升性能。注意：不要在客户端使用HTTP chunking。

- Transport
传输模块用于集群内节点之间的内部通信。从一个节点到另一个节点的每个请求都使用传输模块。 如同HTTP模块，传输模块本质上也是完全异步的。
传输模块使用 TCP 通信，每个节点都与其他节点维持若干 TCP 长连接。内部节点间的所有通信都是本模块承载的。

- Engine
Engine模块封装了对Lucene的操作及translog的调用，它是对一个分片读写操作的最终提供者。 ES使用Guice框架进行模块化管理。Guice是Google开发的轻量级依赖注入框架（IoC）。
软件设计中经常说要依赖于抽象而不是具象，IoC 就是这种理念的实现方式，并且在内部实现了对象的创建和管理。

## 模块结构
在Guice框架下，一个典型的模块由Service和Module类（类名可以自由定义）组成，Service用于实现业务功能，Module类中配置绑定信息。

AbstractModule是Guice提供的基类，模块需要从这个类继承。Module类的主要作用是定义绑定关系，例如：
```
protected void configure（） {
//绑定实现类
bind（ClusterService.class）.toInstance（clusterService）；
}
```

## 模块管理
定义好的模块由ModulesBuilder类统一管理，ModulesBuilder是ES对Guice的封装，内部调用Guice接口，主要对外提供两个方法。
- add方法：添加创建好的模块
- createInjector方法：调用Guice.createInjector创建并返回Injector，后续通过Injector获取相应Service类的实例。

使用ModulesBuilder进行模块管理的伪代码示例：
```
ModulesBuilder modules = new ModulesBuilder（）；
//以Cluster模块为例
ClusterModule clusterModule = new ClusterModule（）；
modules.add（clusterModule）；
//省略其他模块的创建和添加
...
//创建Injector，并获取相应类的实例
injector = modules.createInjector（）；
setGatewayAllocator（injector.getInstance（GatewayAllocator.class）
```
模块化的封装让 ES 易于扩展，插件本身也是一个模块，节点启动时被模块管理器添加进来。






