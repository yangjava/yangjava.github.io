---
layout: post
categories: ElasticSearch
description: none
keywords: ElasticSearch
---
# ElasticSearch源码模块

## gateway模块
gateway 模块负责集群元信息的存储和集群重启时的恢复。

## 元数据
ES 中存储的数据有以下几种：
- state 元数据信息
- index Lucene 生成的索引文件
- translog 事务日志

元数据信息又有以下几种：
- nodes/0/_state/*.st 集群层面元信息；
- nodes/0/indices/{index_uuid}/_state/*.st 索引层面元信息
- nodes/0/indices/{index_uuid}/0/_state/*.st ，分片层面元信息。

- 集群层面：/Data/apps/search/datas/nodes/0/_state/global-2231.st node-2.st
- 索引层面：/Data/apps/search/datas/nodes/0/indices/2A1sfea4RQmGiN3duyF5xw/_state/state-1.st
- 分片层面：/Data/apps/search/datas/nodes/0/indices/2A1sfea4RQmGiN3duyF5xw/0/_state/state-1.st

分别对应 ES 中的数据结构：
- MetaData（集群层）主要是 clusterUUID、settings、templates 等
- IndexMetaData（索引层）主要是 numberOfShards、mappings 等
- ShardStateMetaData（分片层）主要是 version、indexUUID、primary 等
上述信息被持久化到磁盘，需要注意的是：持久化的 state 不包括某个分片存在于哪个节点这种内容路由信息，集群完全重启时，依靠gateway的recovery过程重建RoutingTable。 当读取某个文档时，根据路由算法确定目的分片后，从RoutingTable中查找分片位于哪个节点，然后将请求转发到目的节点。

## 元数据的持久化
只有具备Master资格的节点和数据节点可以持久化集群状态。当收到主节点发布的集群状态时，节点判断元信息是否发生变化，如果发生变化，则将其持久化到磁盘中。

GatewayMetaState 类负责接收集群状态，它继承自 ClusterStateApplier， 并实现其中的applyClusterState 方法，当收到新的集群状态时，ClusterApplierService 通知全部 applier 应用该集群状态:
```
private void callClusterStateAppliers (ClusterChangedEvent clusterChangedEvent) {
    //遍历全部的applier,依次调用各模块对集群状态的处理
    clusterStateAppliers.forEach (applier -> {
        try {
            //调用各模块实现的applyClusterState
            applier. applyClusterState (clusterChangedEvent);
        } catch (Exception ex) {
            //某个模块应用集群状态出现异常时打印日志，但应用过程不会终止
            logger .warn("failed to notify ClusterStateApplier", ex);
        });
}
```
节点校验本身资格，判断元信是否发生变化，并将其持久化到磁盘中，全局元信息和索引级元信息都来自集群状态。
```
public void appl applyClusterState(ClusterChangedEvent event) {
    //只有具备Master资格的节点和数据节点才会持久化元数据
    if (state.nodes().getLocalNode().isMasterNode() || state.nodes().getLocalNode().isDataNode()) {
        //检查全局元信息是否发生变化，如果有变化，则将其写入磁盘
        if (previousMetaData == nulll || !MetaData.isGlobalStateEquals(previousMetaData, newMetaData)) {
            metaStateService.writeGlobalState ("changed", newMetaData);
        }
        //将发生变化的元信息写磁盘
        for (IndexMetaWriteinfo indexMetaWrite : writeinfo) { 
            metaStateServ.writeindex(indexMetaWrite.reason, indexMetaWrite.newMetaData);
        }
    }
}
```
执行文件写入的过程封装在MetaDataStateFormat类中，全局元信息和索引级元信息的写入都执行三个流程：写临时文件、刷盘、“move”成目标文件。
```
//临时文件名
final Path tmpStatePath = stateLocation. resolve (fileName + ". tmp");

//目标文件名
final Path finalStatePath = stateLocation. resolve (fileName);

//写临时文件
OutputStreamIndexOutput out = new OutputStreamIndexOutput (Files.newOutputStream(tmpStatePath), ...);
out.writeInt(format.index());

//从系统cache刷到磁盘中，保证持久化
IOUtils.fsync(tmpStatePath, false); // fsync the state file

//move为目标文件，move操作为系统原子操作
Files.move (tmpStatePath, finalStatePath, StandardCopyOption.ATOMIC_MOVE);
```

## 元数据的恢复
上述的三种元数据信息被持久化存储到集群的每个节点，当集群完全重启(full restart)时，由于分布式系统的复杂性，各个节点保存的元数据信息可能不同。此时需要选择正确的元数据作为权威元数据。gateway的recovery负责找到正确的元数据，应用到集群。

当集群完全重启，达到recovery条件时，进入元数据恢复流程，一般情况下，recovery 条件由以下三个配置控制。
- gateway.expected_nodes 预期的节点数。加入集群的节点数（数据节点或具备 Master 资格的节点）达到这个数量即开始 gateway 恢复。默认为 0；
- gateway.recover_after_time 如果没有达到预期节点数量，恢复程将等待配置的时间，再尝恢复。默认为 5min；
- gateway.recover_after_nodes只要配置数量的节点（数据节点或具备 Master 资格的节点）加入群就可以开始恢复；
假设取值10、5min、8，则启动时节点达 10个则即进入recover可；如果一直没达到10个， 5min 超时后如果节点达到8个也进入 recovery。

还有一些更细致配置，原理与上面三个配置类似：
- gateway.expected_master_nodes：预期的具备 Master 资格的节数，加入集群的具备 Master 资格的节点数达到这个数量后立即开始 gateway 恢复。默认为0；
- gateway.expected_data_nodes：预期的具备数据节点资格的节点数，加入集群的具备数据节点资格的节点数量达到这个数量后立即开始gatway的恢复。默认为0；
- gateway.recover_after_master_nodes：指定数量具备 Master 资格的节点加入集群后就可以开始恢复；
- gateway.recover_after_data_nodes：指定数量数据节点加入集群后就可以开始恢复；
当集群完全启动时，gateway模块负责集群层和索引层的元数据恢复，分片层的元数据恢复在allocation模块实现，但是由gateway模块在执行完上述两个层次恢复工作后触发。

当集群级、索引级元数据选举完毕后，执行submitStateUpdateTask提交一个source为local-gateway-elected-state的任务，触发获取shard级元数据的操作，这个Fetch过程是异步的，根据集群分片数量规模，Fetch过程可能比较长，然后submit任务就结束，gateway流程结束。

因此，三个层次的元数据恢复是由gateway模块和allocation 模块共同完成的，在Gateway将集群级、索引级元数据选举完毕后，在submitStateUpdateTask提交的任务中会执行allocation模块的reroute继续后面的流程。

## 元数据恢复流程分析
主要实现在GatewayService 类中，它继承自ClusterStateListener，在集群状态发生变化（clusterChanged）时触发，仅由Master节点执行。
- Master选举成功之后，判断其持有的集群状态中是否存在STATE_NOT_RECOVERED_BLOCK，如果不存在，则说明元数据已经恢复，跳过gateway恢复过程，否则等待。
- Master 从各个具备Master资格的节点主动获取元数据信息。
- 从获取的元数据信息中选择版本号最大的作为最新元数据，包括集群级、索引级。
- 两者确定之后，调用allocation模块的reroute，对未分配的分片执行分配，主分片分配过程中会异步获取各个shard级别元数据，默认超时为13s。

集群级和索引级元数据信息是根据存储在其中的版本号来选举的，而主分片位于哪个节点却是allocation模块动态计算出来的，先前主分片不一定还被选为新的主分片。

## 选举集群级和索引级别的元数据
判断是否满足进入recovery条件：实现位于GatewayService#clusterChanged，执行此流程的线程为clusterApplierService#updateTask。

此处省略相关的代码引用。当满足条件时，进入recovery 主要流程：实现位于Gateway#performStateRecovery；执行此流程的线程为generic。

首先向有Master资格的节点发起请求，获取它们存储的元数据：
```
//具有Master资格的节点列表
String[] nodesIds = clusterService.state().nodes().getMasterNodes().keys().toArray(String.class);
//发送获取请求并等待结果
TransportNodesListGatewayMetaState.NodesGatewayMetaState nodesState = listGatewayMetaState.list(nodesIds, null).actionGet();
```
等待回复时，必须收到所有节点的回复，无论回复成功还是失败(节点通信失败异常会被捕获，作为失败处理)，此处没有超时。

在收齐的这些回复中，有效元信息的总数必须达到指定数量。异常情况下，例如，某个节点上元信息读取失败，则回复信息中元数据为空。
```
int requiredAllocation = Math.max(1, minimumMasterNodesProvider.get());

minimumMasterNodesProvider的值由下面的配置项决定：discovery.zen.minimum_master_nodes
```
接下来就是通过版本号选取集群级和索引级元数据。
```
选举集群级元数据: 
public void performStateRecovery(...) {
    //遍历请求的所有节点
    for (TransportNodesListGatewayMetaState.NodeGatewayMetaState nodeState : nodesState.getNode ()) {
        //根据元信息中记录的版本号选举元信息
        if (electedGlobalState == null) {
            electedGlobalState = nodeState . metaData() ;
        } else if (nodeState.metaData().version() > electedGlobalState.version()) {
            electedGlobalState = nodeState .metaData() ;
        }
    }
}
```

```
选举索引级元数据:
public void performStateRecovery(. ..) {
    final Object[] keys = indices.keys;
    //遍历集群的全部索引
    for (int i = 0; i < keys. length; i++) {
        if (keys[i] != null) {
            Index index = (Index) keys[i];
            IndexMetaData electedIndexMetaData = null;
            //遍历请求的全部节点，对特定索引选择版本号最高的作为该索引元数据
            for (TransportNodesListGatewayMetaState.NodeGa tewayMetaStatenodeState : nodesState.getNodes()) {
                IndexMetaData indexMetaData = nodeState.metaData ().index(index);
                if (electedIndexMetaData == null) {
                    electedIndexMetaData = indexMetaData;
                } else if (indexMetaData.getVersion() > electedIndexMetaData.getVersion()) {
                    electedIndexMetaData = indexMetaData;
                }
            }
        }
    }
}
```

## 触发allocation
当上述两个层次的元信息选举完毕，调用clusterService.submitStateUpdateTask 提交一个集群任务，该任务在masterService#updateTask线程池中执行，实现位于GatewayRecoveryListener#onSuccess。

主要工作是构建集群状态(ClusterState)，其中的内容路由表依赖allocation模块协助完成，调用allocationService.reroute进入下一阶段：异步执行分片层元数据的恢复，以及分片分配。updateTask线程结束。至此，gateway恢复流程结束，集群级和索引级元数据选举完毕，如果存在未分配的主分片，则分片级元数据选举和分片分配正在进行中。

元数据信息是根据版本号选举出来的，而元数据写入成功的条件是“多数”，因此，保证进入recovery的条件为节点数量为“多数”，可以保证集群级和索引级的一致性。

获取各节点存储的元数据，然后根据版本号选举时，仅向具有Master资格的节点获取元数据。

## allocation模块分析

## 什么是 allocation
分片分配就是把一个分片指派到集群中某个节点的过程。分配决策由主节点完成，分配决策包含两方面：
- 哪些分片应该分配给哪些节点
- 哪个分片作为主分片，哪些作为副本分片

对于新建索引和已有索引, 分片分配过程也不尽相同，不过不管哪种场景，ElasticSearch 都通过两个基础组件完成工作：allocators 分配者和deciders 决定者
- Allocators尝试寻找最优的节点来分配分片，
- Deciders则负责判断并决定是否要进行这次分配.
对于新建索引和已有索引, 分片分配过程也不尽相同
- 对于新建索引，allocators负责找出拥有分片数最少的节点列表，并按分片数量升序排序，因此分片较少的节点会被优先选择。所以对于新建索引，allocators的目标就是以更均衡的方式把新索引的分片分配到集群的节点中。然后deciders依次遍历allocators给出的节点，并判断是否把分片分配到该节点。例如，如果分配过滤规则中禁止节点A持有索引idx中的任一分片，那么过滤器也阻止把索引idx分配到节点A中，即便A节点是allocators从集群负载均衡角度选出的最优节点。需要注意的是，allocators只关心每个节点上的分片数，而不管每个分片的具体大小。这恰好是 deciders 工作的一部分，即阻止把分片分配到将超出节点磁盘容量阈值的节点上。
- 对于已有索引，则要区分主分片还是副分片。对于主分片，allocators只允许把主分片指定在已经拥有该分片完整数据的节点上。而对于副分片，allocators则是先判断其他节点上是否已有该分片的数据的副本（即便数据不是最新的）。如果有这样的节点，则allocators优先把分片分配到其中一个节点。因为副分片一旦分配，就需要从主分片中进行数据同步，所以当一个节点只拥分片中的部分数据时，也就意味着那些未拥有的数据必须从主节点中复制得到。这样可以明显地提高副分片的数据恢复速度。

## 触发时机
触发分片分配有以下几种情况：
- index增删；
- node增删；
- 手工reroute；
- replica数量改变；
- 集群重启。

## allocation模块结构概述
这个复杂的分配过程在reroute函数中实现：
```
AllocationService.reroute
```
此函数对外有两种重载，一种是通过接口调用的手工 reroute，另一种是内部模块调用的reroute。本章以内部模块调用的reroute为例，手工reroute过程与此类似。

AllocationService.reroute对一个或多个主分片或副分片执行分配，分配以后产生新的集群状态。Master节点将新的集群状态广播下去，触发后续的流程。对于内部模块调用，返回值为新产生的集群状态，对于手工执行的reroute命令，返回命令执行结果。

## allocators
Allocator 负责为某个特定的 shard 分配目的节点。每个Allocator的主要工作是根据某种逻辑得到一个节点列表，然后调用 deciders 去决策，根据决策结果选择一个目的 node。

Allocators 分为 gatewayAllocator 和 shardsAllocator 两种。gatewayAllocator 是为了找到现有分片，shardsAllocator 是根据权重策略在集群的各节点间均衡分片分布。其中 gatewayAllocator 又分主分片和副分片的 allocator。 下面概述每个allocator的作用。
- primaryShardAllocator：找到那些拥有某分片最新数据的节点;
- replicaShardAllocator：找到磁盘上拥有这个分片数据的节点;
- BalancedShardsAllocator：找到拥有最少分片个数的节点。
对于这两类 Allocator，我的理解是 gatewayAllocator 是为了找到现有shard，shardsAllocator 是为了分配全新 shard

## Deciders
目前有下列类型的决策器：
```
public static Collection<AllocationDecider> createAllocationDeciders(...) {
    Map<Class, AllocationDecider> deciders = new LinkedHashMap<>();
    addAllocationDecider (deciders, new MaxRetryAllocationDecider(settings));
    addAllocationDecider (deciders, new ResizeAllocationDecider(settings));
    addAllocationDecider (deciders, new ReplicaAfterPrimaryActiveAllocation-Decider(settings));
    addAllocationDecider (deciders, new RebalanceOnlyWhenActiveAllocation-Decider(settings));
    addAllocationDecider (deciders, new ClusterRebalanceAllocationDecider(settings, clusterSettings));
    addAllocationDecider (deciders, new ConcurrentRebalanceAllocationDecider(settings, clusterSettings));
    addAllocationDecider (deciders, new EnableAllocationDecider(settings, clusterSettings));
    addAllocationDecider (deciders, new NodeVersionAllocationDecider(settings));
    addAllocationDecider (deciders, new SnapshotInProgressAllocationDecider(settings));
    addAllocationDecider (deciders, new RestoreInProgressAllocationDecider(settings));
    addAllocationDecider (deciders, new FilterAllocationDecider(settings, clusterSettings));
    addAllocationDecider (deciders, new SameShardAllocationDecider(settings, clusterSettings));
    addAllocationDecider (deciders, new DiskThresholdDecider(settings, clusterSettings));
    addAllocationDecider (deciders, new ThrottlingAllocationDecider(settings, clusterSettings));
    addAllocationDecider (deciders, new ShardsLimitAllocationDecider(settings, clusterSettings));
    addAllocationDecider (deciders, new AwarenessAllocationDecider(settings, clusterSettings));
    
    return deciders.values();
}
```
它们继承自AllocationDecider，需要实现的接口有：
- canRebalance：给定分片是否可以“re-balanced” 到给定allocation；
- canAllocate：给定分片是否可以分配到给定节点；
- canRemain：给定分片是否可以保留在给定节点；
- canForceAllocatePrimary：给定主分片是否可以强制分配在给定节点；
这些 deciders 在ClusterModule#createAllocationDeciders中全部添加进去，decider 运行之后可能产生的结果有以下几种：
ALWAYS、YES、NO、THROTTLE
这些 deciders 大致可以分为以下几类。

- 负载均衡类
```
SameShardAllocationDecider
避免主副分片分配到同一个节点

AwarenessAllocationDecider
感知分配器，感知服务器、机架等，尽量分散存储shard
有两种参数用于调整:
cluster.routing.allocation.awareness.attributes: rack_id
cluster.routing.allocation.awareness.attributes: zone

ShardsLimitAllocationDecider
同一个节点上允许存在的同一个index的shard数目
```

- 并发控制类
```
ThrottlingAllocationDecider
recovery阶段的限速配置，包括: 
    cluster.routing.allocation.node_concurrent_recoveries
    cluster.routing.allocation.node_initial_primaries_recoveries
    cluster.routing.allocation.node_concurrent_incoming_recoveries
    cluster.routing.allocation.node_concurrent_outgoing_recoveries

ConcurrentRebalanceAllocationDecider
rebalance并发控制，可以通过下面的参数配置:
    cluster.routing.allocation.cluster_concurrent_rebalance

DiskThresholdDecider
根据磁盘空间进行决策的分配器
```

- 条件限制类
```

RebalanceOnlyWhenActiveAllocationDecider
所有shard都处在active状态下，才可以执行rebalance操作

FilterAllocationDecider
可以调整的参数如下，可以通过接口动态设置:
    index.routing.allocation.require.* [ 必须 ]
    index.routing.allocation.include.* [ 允许 ]
    index.routing.allocation.exclude.* [ 排除 ]
    cluster.routing.allocation.require.*
    cluster.routing.allocation.include.*
    cluster.routing.allocation.exclude.*
    配置的目标为节点IP或节点名等。cluster 级别设置会覆盖index级别设置。

ReplicaAfterPrimaryActiveAllocationDecider
保证只会在主分片分配完毕后才开始分配分片副本。

ClusterRebalanceAllocationDecider
通过集群中active的shard状态来决定是否可以执行rebalance，通过下面的配置控制，可以动态生效:
    cluster.routing.allocation.allow_rebalance
```
可配置的值如下表所示：
- indices_ all_active	当集群所有的节点分配完毕，才认定集群rebalance完成(默认)
- indices_primaries_active	只要所有主分片分配完毕，就可以认定集群rebalance完成
- always	即使当主分片和分片副本都没有分配，也允许rebalance操作

## 核心reroute实现
reroute 中主要实现两种 allocator：
- gatewayAllocator：用于分配现实已存在的分片，从磁盘中找到它们；
- shardsAllocator：用于平衡分片在节点间的分布；
```
private void reroute (RoutingAllocation allocation) {
    if(allocation.routingNodes().unassigned().size() > 0) {
        removeDelayMarkers (allocation) ;
        //gateway分配器
        gatewayAllocator.allocateUnassigned (allocation);
    }
    //分片均衡分配器
    shardsAllocator.allocate(allocation);
}
```
reroute 流程全部运行于 masterService#updateTask线程。

## 集群启动时reroute的触发时机
gateway结束前： 调用submitStateUpdateTask提交任务，任务被clusterService 放入队列，在Master节点顺序执行。

执行到任务中的：allocationService.reroute

收集各个节点的shard元数据，待某个 shard 的 Response 从所有节点全部返回后，执行 finishHim()，然后对收集到的数据进行处理：AsyncShardFetch#processAsyncFetch

allocationService.reroute执行完毕返回新的集群状态。下面以集群启动时 gateway 之后的 reroute 为背景分析流程。

## 流程分析
gateway 阶段恢复的集群状态中，我们已经知道集群一共有多少个索引，每个索引的主副分片各有多少个，但是不知道它们位于哪个节点，现在需要找到它们都位于哪个节点。集群完全重启的初始状态，所有分片都被标记为未分配状态，此处也被称作分片分配过程。因此分片分配的概念不仅仅是分配一个全新分片。对于索引某个特定分片的分配过程中，先分配其主分片，后分配其副分片。

## GatewayAllocator
gatewayAllocator分为主分片和副分片分配器:

primaryShardAllocator.allocateUnassigned (allocation) ;
replicaShardAllocator.processExistingRecoveries (allocation) ;
replicaShardAllocator.allocateUnassigned (allocation) ;

### 主分片分配器
primaryShardAllocator#allocateUnassigned函数实现整个分配过程。

主分片分配器与副分片分配器都继承自BaseGatewayShardAllocator， 执行相同的allocateUnassigned函数，只是执行makeAllocationDecision时，主副分片分配器各自执行自己的策略。

allocateUnassigned的流程是：遍历所有unassigned shard, 依次处理，通过decider决策分配，期间可能需要fetchData获取这个shard对应的元数据。如果决策结果为YES，则将其初始化。
```
public void allocateUnassigned (RoutingAllocation allocation) {
    //遍历未分配的分片
    while (unassignedIterator.hasNext() ) {
        final ShardRouting shard = unassignedIterator.next() ;
        //调用主分片分配器的makeAllocationDecision进行决策
        final AllocateUnassignedDecision allocateUnassignedDecision = makeAllocationDecision (shard, allocation，logger);
        //根据决策器结果决定是否初始化分片
        if (allocateUnas signedDecision.getAllocationDecision() == AllocationDecision.YES) {
            unassignedIterator. initialize(...);
        } else {
            unassignedIterator . removeAndIgnore(. ..) ;
        }
    }
}
```


