---
layout: post
categories: ElasticSearch
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

## 集群启动流程
让我们从启动流程开始，先在宏观上看看整个集群是如何启动的，集群状态如何从Red变成Green，不涉及代码，然后分析其他模块的流程。

本书中，集群启动过程指集群完全重启时的启动过程，期间要经历选举主节点、主分片、数据恢复等重要阶段，理解其中原理和细节，对于解决或避免集群维护过程中可能遇到的脑裂、无主、恢复慢、丢数据等问题有重要作用。

## 选举主节点
假设有若干节点正在启动，集群启动的第一件事是从已知的活跃机器列表中选择一个作为主节点，选主之后的流程由主节点触发。

ES的选主算法是基于Bully算法的改进，主要思路是对节点ID排序，取ID值最大的节点作为Master，每个节点都运行这个流程。是不是非常简单？选主的目的是确定唯一的主节点，初学者可能认为选举出的主节点应该持有最新的元数据信息，实际上这个问题在实现上被分解为两步：先确定唯一的、大家公认的主节点，再想办法把最新的机器元数据复制到选举出的主节点上。

基于节点ID排序的简单选举算法有三个附加约定条件：
- 参选人数需要过半，达到 quorum（多数）后就选出了临时的主。为什么是临时的？每个节点运行排序取最大值的算法，结果不一定相同。举个例子，集群有5台主机，节点ID分别是1、2、3、4、5。当产生网络分区或节点启动速度差异较大时，节点1看到的节点列表是1、2、3、4，选出4；节点2看到的节点列表是2、3、4、5，选出5。结果就不一致了，由此产生下面的第二条限制。
- 得票数需过半。某节点被选为主节点，必须判断加入它的节点数过半，才确认Master身份。解决第一个问题。
- 当探测到节点离开事件时，必须判断当前节点数是否过半。如果达不到 quorum，则放弃Master身份，重新加入集群。如果不这么做，则设想以下情况：假设5台机器组成的集群产生网络分区，2台一组，3台一组，产生分区前，Master位于2台中的一个，此时3台一组的节点会重新并成功选取Master，产生双主，俗称脑裂。

集群并不知道自己共有多少个节点，quorum值从配置中读取，我们需要设置配置项：
```
discovery.zen.minimum_master_nodes
```

## 选举集群元信息
被选出的 Master 和集群元信息的新旧程度没有关系。因此它的第一个任务是选举元信息，让各节点把各自存储的元信息发过来，根据版本号确定最新的元信息，然后把这个信息广播下去，这样集群的所有节点都有了最新的元信息。

集群元信息的选举包括两个级别：集群级和索引级。不包含哪个shard存于哪个节点这种信息。这种信息以节点磁盘存储的为准，需要上报。为什么呢？因为读写流程是不经过Master的，Master 不知道各 shard 副本直接的数据差异。HDFS 也有类似的机制，block 信息依赖于DataNode的上报。

为了集群一致性，参与选举的元信息数量需要过半，Master发布集群状态成功的规则也是等待发布成功的节点数过半。

在选举过程中，不接受新节点的加入请求。 集群元信息选举完毕后，Master发布首次集群状态，然后开始选举shard级元信息。

## allocation过程
选举shard级元信息，构建内容路由表，是在allocation模块完成的。在初始阶段，所有的shard都处于UNASSIGNED（未分配）状态。ES中通过分配过程决定哪个分片位于哪个节点，重构内容路由表。此时，首先要做的是分配主分片。

### 选主分片
现在看某个主分片`[website][0]`是怎么分配的。所有的分配工作都是 Master 来做的，此时， Master不知道主分片在哪，它向集群的所有节点询问：大家把`[website][0]`分片的元信息发给我。然后，Master 等待所有的请求返回，正常情况下它就有了这个 shard 的信息，然后根据某种策略选一个分片作为主分片。是不是效率有些低？这种询问量=shard 数×节点数。所以说我们最好控制shard的总规模别太大。

现在有了shard[website][0]的分片的多份信息，具体数量取决于副本数设置了多少。现在考虑把哪个分片作为主分片。ES 5.x以下的版本，通过对比shard级元信息的版本号来决定。在多副本的情况下，考虑到如果只有一个 shard 信息汇报上来，则它一定会被选为主分片，但也许数据不是最新的，版本号比它大的那个shard所在节点还没启动。在解决这个问题的时候，ES 5.x开始实施一种新的策略：给每个 shard 都设置一个 UUID，然后在集群级的元信息中记录哪个shard是最新的，因为ES是先写主分片，再由主分片节点转发请求去写副分片，所以主分片所在节点肯定是最新的，如果它转发失败了，则要求Master删除那个节点。所以，从ES 5.x开始，主分片选举过程是通过集群级元信息中记录的“最新主分片的列表”来确定主分片的：汇报信息中存在，并且这个列表中也存在。

如果集群设置了：
```
＂cluster.routing.allocation.enable＂: ＂none＂
```
禁止分配分片，集群仍会强制分配主分片。因此，在设置了上述选项的情况下，集群重启后的状态为Yellow，而非Red。

### 选副分片
主分片选举完成后，从上一个过程汇总的 shard 信息中选择一个副本作为副分片。如果汇总信息中不存在，则分配一个全新副本的操作依赖于延迟配置项：
```
index.unassigned.node_left.delayed_timeout
```
我们的线上环境中最大的集群有100+节点，掉节点的情况并不罕见，很多时候不能第一时间处理，这个延迟我们一般配置为以天为单位。 最后，allocation过程中允许新启动的节点加入集群。

## index recovery
分片分配成功后进入recovery流程。主分片的recovery不会等待其副分片分配成功才开始recovery。它们是独立的流程，只是副分片的recovery需要主分片恢复完毕才开始。

为什么需要recovery？对于主分片来说，可能有一些数据没来得及刷盘；对于副分片来说，一是没刷盘，二是主分片写完了，副分片还没来得及写，主副分片数据不一致。

### 主分片recovery
由于每次写操作都会记录事务日志（translog），事务日志中记录了哪种操作，以及相关的数据。因此将最后一次提交（Lucene 的一次提交就是一次 fsync 刷盘的过程）之后的 translog中进行重放，建立Lucene索引，如此完成主分片的recovery。

### 副分片recovery
副分片的恢复是比较复杂的，在ES的版本迭代中，副分片恢复策略有过不少调整。

副分片需要恢复成与主分片一致，同时，恢复期间允许新的索引操作。在目前的6.0版本中，恢复分成两阶段执行。

· phase1：在主分片所在节点，获取translog保留锁，从获取保留锁开始，会保留translog不受其刷盘清空的影响。然后调用Lucene接口把shard做快照，这是已经刷磁盘中的分片数据。把这些shard数据复制到副本节点。在phase1完毕前，会向副分片节点发送告知对方启动engine，在phase2开始之前，副分片就可以正常处理写请求了。

· phase2：对translog做快照，这个快照里包含从phase1开始，到执行translog快照期间的新增索引。将这些translog发送到副分片所在节点进行重放。

由于需要支持恢复期间的新增写操作（让ES的可用性更强），这两个阶段中需要重点关注以下几个问题。

分片数据完整性：如何做到副分片不丢数据？第二阶段的 translog 快照包括第一阶段所有的新增操作。那么第一阶段执行期间如果发生“Lucene commit”（将文件系统写缓冲中的数据刷盘，并清空translog），清除translog怎么办？在ES 2.0之前，是阻止了刷新操作，以此让translog都保留下来。从2.0版本开始，为了避免这种做法产生过大的translog，引入了translog.view的概念，创建 view 可以获取后续的所有操作。从6.0版本开始，translog.view 被移除。引入TranslogDeletionPolicy的概念，它将translog做一个快照来保持translog不被清理。这样实现了在第一阶段允许Lucene commit。

数据一致性：在ES 2.0之前，副分片恢复过程有三个阶段，第三阶段会阻塞新的索引操作，传输第二阶段执行期间新增的translog，这个时间很短。自2.0版本之后，第三阶段被删除，恢复期间没有任何写阻塞过程。在副分片节点，重放translog时，phase1和phase2之间的写操作与phase2重放操作之间的时序错误和冲突，通过写流程中进行异常处理，对比版本号来过滤掉过期操作。

这样，时序上存在错误的操作被忽略，对于特定的 doc，只有最新一次操作生效，保证了主副分片一致。

第一阶段尤其漫长，因为它需要从主分片拉取全量的数据。在ES 6.x中，对第一阶段再次优化：标记每个操作。在正常的写操作中，每次写入成功的操作都分配一个序号，通过对比序号就可以计算出差异范围，在实现方式上，添加了global checkpoint和local checkpoint，主分片负责维护global checkpoint，代表所有分片都已写入这个序号的位置，local checkpoint代表当前分片已写入成功的最新位置，恢复时通过对比两个序列号，计算出缺失的数据范围，然后通过translog重放这部分数据，同时translog会为此保留更长的时间。

因此，有两个机会可以跳过副分片恢复的phase1：基于SequenceNumber，从主分片节点的translog恢复数据；主副两分片有相同的syncid且doc数相同，可以跳过phase1。

## 集群启动日志
日志是分布式系统中排查问题的重要手段，虽然 ES 提供了很多便于排查问题的接口，但重要日志仍然是不可或缺的。默认情况下，ES输出的INFO级别日志较少，许多重要模块的关键环节是DEBUG 或TRACE 级别的。

## 启动流程做了什么
总体来说，节点启动流程的任务是做下面几类工作：
- 解析配置，包括配置文件和命令行参数。
- 检查外部环境和内部环境，例如，JVM版本、操作系统内核参数等。
- 初始化内部资源，创建内部模块，初始化探测器。
- 启动各个子模块和keepalive线程。

### 启动脚本
当我们通过启动脚本bin/elasticsearch启动ES时，脚本通过exec加载Java程序。

ES_JAVA_OPTS变量保存了JVM参数，其内容来自对config/jvm.options配置文件的解析。

如果执行启动脚本时添加了-d参数：

bin/elasticsearch –d

则启动脚本会在exec中添加<&- &。<&-的作用是关闭标准输入，即进程中的0号fd。&的作用是让进程在后台运行。

### 解析命令行参数和配置文件
实际工程应用中建议在启动参数中添加-d和-p，例如：
```
bin/elasticsearch -d -p es.pid
```
此处解析的配置文件有下面两个，jvm.options是在启动脚本中解析的。
- elasticsearch.yml #主要配置文件
- log4j2.properties #日志配置文件

### 加载安全配置
什么是安全配置？本质上是配置信息，既然是配置信息，一般是写到配置文件中的。ES的几个配置文件在之前的章节提到过。此处的“安全配置”是为了解决有些敏感的信息不适合放到配置文件中的，因为配置文件是明文保存的，虽然文件系统有基于用户权限的保护，但这仍然不够。因此ES把这些敏感配置信息加密，单独放到一个文件中：config/elasticsearch.keystore。然后提供一些命令来查看、添加和删除配置。

哪种配置信息适合放到安全配置文件中？例如，X-Pack中的security相关配置，LDAP的base_dn等信息（相当于登录服务器的用户名和密码）。

### 检查内部环境
内部环境指ES软件包本身的完整性和正确性。包括：
- 检查 Lucene 版本，ES 各版本对使用的 Lucene 版本是有要求的，在这里检查 Lucene版本以防止有人替换不兼容的jar包。
- 检测jar冲突（JarHell），发现冲突则退出进程。

# 参考资料
Elasticsearch 源码解析与优化实战





