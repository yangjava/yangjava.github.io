---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码数据分片
日常开发过程中，几乎每个人都遇到过ES unassigned shard的情况，尤其是集群规模比较大的情况下，遇到这种问题处理还是比较麻烦。

Master节点负责Shard allocation，都调用allocationService.reroute方法执行Shard分配，这里做个总结。

分布式系统基本都会涉及Rebalance，例如Kafka的preferred leader选举，ES的shard reblance，这尽量让集群内部数据、分配分片，保证各个节点间尽量均衡。Kafka的leader replica负责数据的读写操作，ES 的primary shard负责写，replica shard能够承担读。如果集群内部node节点之间的负载不均衡，这极大影响集群的健壮性和读写性能。如果ES集群Primary shard/Replica shard都集中在某一节点，若节点异常，则分布式系统的高可用则不存在，这也是需要shard Rebalance。

allocation与rebalance的关系：rebalance需要依赖allocation，当allocation被关闭以后rebalance就被禁止。

分片分配（Shard Allocation）就是把一个分片指派到集群中某个节点的过程，分配决策由主节点完成。分配决策包含两部分：哪个分片应该分配到哪个节点，以及哪个分片作为主分片或者是副分片。

## Shard Allocation

### 重要参数
cluster.routing.allocation.enable能够动态调整，控制shard的recover和allocation

all - 默认值，允许为所有类型分片分配分片；
primaries - 仅允许分配主分片的分片；
new_primaries - 仅允许为新索引的主分片分配分片；
none - 任何索引都不允许任何类型的分片
重新启动节点时，此设置不会影响本地主分片的恢复。如果重新启动的节点具有未分配的主分片的副本，会立即恢复该主分片。

### 触发条件
index 增删
node 增删
replica数量改变
手工 reroute
集群重启

## Shard Rebalance

### 重要参数
动态调整参数，能够控制整个集群的shard rebalance

cluster.routing.rebalance.enable 特定的分片，启用或禁用rebalance

all - （默认值）允许各种分片的分片平衡；
primaries - 仅允许主分片的分片平衡；
replicas - 仅允许对副本分片进行分片平衡；
none - 任何索引都不允许任何类型的分片平衡
cluster.routing.allocation.allow_rebalance用来控制rebalance触发条件：

always - 始终允许重新平衡；
indices_primaries_active - 仅在所有主分片可用时；
indices_all_active - （默认）仅当所有分片都激活时；
cluster.routing.allocation.balance.threshold 执行操作的最小优化值

### 触发条件
当集群的分片数发生变化时，就有可能触发分片rebalance。重新分配分片重平衡是一个高IO和网络消耗的操作，会给集群带来很大的负担，一般遇到会触发集群重平衡的操作需要选择在业务低峰期进行。

具体分片是否执行重平衡是通过阈值判定，将在不同的node上的shard转移来平衡集群中每台node的shard数。个人理解，可以从逻辑角度和磁盘存储角度看待发分片rebalance：

一、逻辑角度：集群中每个Node的个数要尽量均衡，不能出现一个Node上shard过多或过少的情形；另一个从索引方面需要考虑，索引的shard也应该更加均衡的分配到node，而不是集中在某一个shard；

二、磁盘存储：ES的Docuement写入，是根据id进行路由（路由的算法：shard_num =hash(_routing)% num_primary_shards），如果用户不指定id使用ES默认的情况下，Node的磁盘利用率应该比较均衡（除非某些Docuement特别大，某些特别小）；如果用户指定id，则可能会存在某些Node的磁盘利用率比较高，其他的确比较低的情形。无论哪种原因造成磁盘利用率差别比较大的时候，确实需要分片的rebalance。

## 逻辑角度

rebalance策略的触发条件，主要由下面几个参数控制：

（a）定义分配在该节点的分片数的因子 阈值=因子*（当前节点的分片数-集群的总分片数/节点数，即每个节点的平均分片数）
cluster.routing.allocation.balance.shard: 0.45f
（b）定义分配在该节点某个索引的分片数的因子，阈值=因子*（保存当前节点的某个索引的分片数-索引的总分片数/节点数，即每个节点某个索引的平均分片数）
cluster.routing.allocation.balance.index: 0.55f
（c）超出这个阈值就会重新分配分片
cluster.routing.allocation.balance.threshold: 1.0f

## 磁盘存储角度

另一个方面看rebalance就是磁盘利用率

（a）是否启用基于磁盘的分发策略
cluster.routing.allocation.disk.threshold_enabled: true
（b）硬盘使用率高于这个值的节点，则不会分配分片
cluster.routing.allocation.disk.watermark.low: "85%"
（c）如果硬盘使用率高于这个值，则会重新分片该节点的分片到别的节点
cluster.routing.allocation.disk.watermark.high: "90%"
（d）当前硬盘使用率的查询频率
cluster.info.update.interval: "30s"
（e）计算硬盘使用率时，是否加上正在重新分配给其他节点的分片的大小
cluster.routing.allocation.disk.include_relocations: true

具体计算方法在WeightFunction类，其中计算规则如下：
```
weightindex(node, index) = indexBalance * (node.numShards(index) - avgShardsPerNode(index))
weightnode(node, index) = shardBalance * (node.numShards() - avgShardsPerNode)
weight(node, index) = weightindex(node, index) + weightnode(node, index)
```
除了上述情形，官方文档还给出其他重要参数：

A. 分片分配感知（shard allocation awareness），控制shard如何在不同的机架上进行分布。这个和集群Node物理分配有关。

Kafka在0.10版本也提供了类似功能，Kafka内置机架感知以便隔离副本，这使得Kafka保证副本可以跨越到多个机架或者是可用区域，显著提高了Kafka的弹性和可用性。
Ｂ. 分片分配过滤器（shard allocation filter），可以控制有些node不参与allocation的过程，这样的话，这些node就可以被安全的下线

## AllocationService
索引创建和索引删除的过程中，都调用allocationService.reroute方法执行Shard分配。在AllocationService类中，reroute方法有个重载方法，一个是针对ES系统内部模块执行，返回ClusterState；一个是用户手动执行POST /_cluster/reroute命令，返回执行结果。

Elasticsearch 主要通过两个基础组件来完成分片分配这个过程的: allocator 和 deciders;

（1）allocator 寻找最优的节点来分配分片;

allocator 负责找出拥有分片数量最少的节点列表, 按分片数量递增排序, 分片数量较少的会被优先选择;

对于新建索引, allocator 的目标是以更为均衡的方式把新索引的分片分配到集群的节点中;

（2）deciders 负责判断并决定是否要进行分配;

deciders 依次遍历 allocator 给出的节点列表, 判断是否要把分片分配给该节点, 比如是否满足分配过滤规则, 分片是否将超出节点磁盘容量阈值等等;

## 手动执行reroute
手动执行reroute命令的入口来自TransportClusterRerouteAction类的execute方法。

手动执行reroute命令，以官网给的示例如下：
```
POST /_cluster/reroute
{
    "commands" : [
        {
            "move" : {
                "index" : "test", "shard" : 0,
                "from_node" : "node1", "to_node" : "node2"
            }
        },
        {
          "allocate_replica" : {
                "index" : "test", "shard" : 1,
                "node" : "node3"
          }
        }
    ]
}
```
客户端发起请求，在服务端会解析成两个命令分别对应MoveAllocationCommand和AllocateReplicaAllocationCommand两个类。

手动执行reroute方法对应源码：
```
/**执行 relocation 命令,手动执行reroute*/
public CommandsResult reroute(final ClusterState clusterState, AllocationCommands commands, boolean explain, boolean retryFailed) {
    RoutingNodes routingNodes = getMutableRoutingNodes(clusterState);
    // we don't shuffle the unassigned shards here, to try and get as close as possible to
    // a consistent result of the effect the commands have on the routing
    // this allows systems to dry run the commands, see the resulting cluster state, and act on it
    RoutingAllocation allocation = new RoutingAllocation(allocationDeciders, routingNodes, clusterState,
        clusterInfoService.getClusterInfo(), currentNanoTime());
    // don't short circuit deciders, we want a full explanation
    allocation.debugDecision(true);
    // we ignore disable allocation, because commands are explicit
    allocation.ignoreDisable(true);

    if (retryFailed) {
        resetFailedAllocationCounter(allocation);
    }
    /**核心方法*/
    RoutingExplanations explanations = commands.execute(allocation, explain);
    // we revert the ignore disable flag, since when rerouting, we want the original setting to take place
    allocation.ignoreDisable(false);
    // the assumption is that commands will move / act on shards (or fail through exceptions)
    // so, there will always be shard "movements", so no need to check on reroute
    reroute(allocation);
    return new CommandsResult(explanations, buildResultAndLogHealthChange(clusterState, allocation, "reroute commands"));
}
```
commands.execute(allocation, explain)是执行的核心：
```
public RoutingExplanations execute(RoutingAllocation allocation, boolean explain) {
    RoutingExplanations explanations = new RoutingExplanations();
    for (AllocationCommand command : commands) {
        explanations.add(command.execute(allocation, explain));
    }
    return explanations;
}
```
command.execute会执行 MoveAllocationCommand和AllocateReplicaAllocationCommand两个类的实现方法。

## MoveAllocationCommand

这里只发生在Master节点，并没有真正执行，真正的执行是在Data节点
```
public RerouteExplanation execute(RoutingAllocation allocation, boolean explain) {
    DiscoveryNode fromDiscoNode = allocation.nodes().resolveNode(fromNode);
    DiscoveryNode toDiscoNode = allocation.nodes().resolveNode(toNode);
    Decision decision = null;

    boolean found = false;
    //获取from的RoutingNode
    RoutingNode fromRoutingNode = allocation.routingNodes().node(fromDiscoNode.getId());
    if (fromRoutingNode == null && !fromDiscoNode.isDataNode()) {
        throw new IllegalArgumentException("[move_allocation] can't move [" + index + "][" + shardId + "] from "
            + fromDiscoNode + " to " + toDiscoNode + ": source [" +  fromDiscoNode.getName()
            + "] is not a data node.");
    }
    //获取to的RoutingNode
    RoutingNode toRoutingNode = allocation.routingNodes().node(toDiscoNode.getId());
    if (toRoutingNode == null && !toDiscoNode.isDataNode()) {
        throw new IllegalArgumentException("[move_allocation] can't move [" + index + "][" + shardId + "] from "
            + fromDiscoNode + " to " + toDiscoNode + ": source [" +  toDiscoNode.getName()
            + "] is not a data node.");
    }

    for (ShardRouting shardRouting : fromRoutingNode) {
        //从请求的index中查找from shard
        if (!shardRouting.shardId().getIndexName().equals(index)) {
            continue;
        }
        if (shardRouting.shardId().id() != shardId) {
            continue;
        }
        found = true;

        // TODO we can possibly support also relocating cases, where we cancel relocation and move...
        if (!shardRouting.started()) {
            
            throw new IllegalArgumentException("[move_allocation] can't move " + shardId +
                    ", shard is not started (state = " + shardRouting.state() + "]");
        }

        //判断是否能进行Allocate
        decision = allocation.deciders().canAllocate(shardRouting, toRoutingNode, allocation);
        if (decision.type() == Decision.Type.NO) {
            
            throw new IllegalArgumentException("[move_allocation] can't move " + shardId + ", from " + fromDiscoNode + ", to " +
                toDiscoNode + ", since its not allowed, reason: " + decision);
        }
        
        //执行分片分配，这里只发生在Master节点，并没有真正执行，真正的执行是在Data节点
        allocation.routingNodes().relocateShard(shardRouting, toRoutingNode.nodeId(),
            allocation.clusterInfo().getShardSize(shardRouting, ShardRouting.UNAVAILABLE_EXPECTED_SHARD_SIZE), allocation.changes());
    }

    //未找到shard报错，并返回
    if (!found) {
        throw new IllegalArgumentException("[move_allocation] can't move " + shardId + ", failed to find it on node " + fromDiscoNode);
    }
    return new RerouteExplanation(this, decision);
}
```
allocation.routingNodes().relocateShardMaster节点执行shard分配逻辑。

### AllocateReplicaAllocationCommand
```
public RerouteExplanation execute(RoutingAllocation allocation, boolean explain) {
    final DiscoveryNode discoNode;
    //根据客户端发起请求的node，查找对应的RoutingNode
    try {
        discoNode = allocation.nodes().resolveNode(node);
    } catch (IllegalArgumentException e) {
        return explainOrThrowRejectedCommand(explain, allocation, e);
    }
    final RoutingNodes routingNodes = allocation.routingNodes();
    RoutingNode routingNode = routingNodes.node(discoNode.getId());
    if (routingNode == null) {
        return explainOrThrowMissingRoutingNode(allocation, explain, discoNode);
    }

    try {
        //更新路由表信息
        allocation.routingTable().shardRoutingTable(index, shardId).primaryShard();
    } catch (IndexNotFoundException | ShardNotFoundException e) {
        return explainOrThrowRejectedCommand(explain, allocation, e);
    }

    ShardRouting primaryShardRouting = null;
    //遍历路由表，查找需要创建的primary shard
    for (RoutingNode node : allocation.routingNodes()) {
        for (ShardRouting shard : node) {
            if (shard.getIndexName().equals(index) && shard.getId() == shardId && shard.primary()) {
                primaryShardRouting = shard;
                break;
            }
        }
    }
    if (primaryShardRouting == null) {
        return explainOrThrowRejectedCommand(explain, allocation,
            "trying to allocate a replica shard [" + index + "][" + shardId +
                "], while corresponding primary shard is still unassigned");
    }

    //遍历路由表
    List<ShardRouting> replicaShardRoutings = new ArrayList<>();
    for (ShardRouting shard : allocation.routingNodes().unassigned()) {
        if (shard.getIndexName().equals(index) && shard.getId() == shardId && shard.primary() == false) {
            replicaShardRoutings.add(shard);
        }
    }

    ShardRouting shardRouting;
    //未找到replicaShard
    if (replicaShardRoutings.isEmpty()) {
        return explainOrThrowRejectedCommand(explain, allocation,
            "all copies of [" + index + "][" + shardId + "] are already assigned. Use the move allocation command instead");
    } else {
        shardRouting = replicaShardRoutings.get(0);
    }

    //Decider判断是否能够执行Allocate
    Decision decision = allocation.deciders().canAllocate(shardRouting, routingNode, allocation);
    if (decision.type() == Decision.Type.NO) {
        // don't use explainOrThrowRejectedCommand to keep the original "NO" decision
        throw new IllegalArgumentException("[" + name() + "] allocation of [" + index + "][" + shardId + "] on node " + discoNode +
            " is not allowed, reason: " + decision);
    }
    //初始化未分配的shard，并从UnassignedShard列表移除
    initializeUnassignedShard(allocation, routingNodes, routingNode, shardRouting);
    return new RerouteExplanation(this, decision);
}
```
canAllocate是Deciders判断是否能够进行Allocate。

## ES内部模块执行reroute
当触发Shard Allocation时，ES内部会调用AllocateService的reroute方法。
```
public ClusterState reroute(ClusterState clusterState, String reason) {
    // 1、检查复制副本是否具有需要调整的自动扩展功能。如果需要更改，则返回更新的群集状态；如果不需要更改，则返回相同的群集状态。
    ClusterState fixedClusterState = adaptAutoExpandReplicas(clusterState);
    // 2、创建一个{@link RoutingNodes}。这是一个开销很大的操作，因此只能调用一次！
    RoutingNodes routingNodes = getMutableRoutingNodes(fixedClusterState);

    // shuffle the unassigned nodes, just so we won't have things like poison failed shards
    // 3、重排序未分配的节点
    routingNodes.unassigned().shuffle();

    RoutingAllocation allocation = new RoutingAllocation(allocationDeciders, routingNodes, fixedClusterState,
        clusterInfoService.getClusterInfo(), currentNanoTime());
    //4.执行分配路由
    reroute(allocation);
    if (fixedClusterState == clusterState && allocation.routingNodesChanged() == false) {
        return clusterState;
    }
    //5.构建集群状态
    return buildResultAndLogHealthChange(clusterState, allocation, reason);
}
```
无论是手动执行retoute相关命令还是ES集群内部执行，最后都会调用reroute(allocation);方法执行分配。
```
//reroute方法核心
private void reroute(RoutingAllocation allocation) {
    removeDelayMarkers(allocation);
    // try to allocate existing shard copies first
    gatewayAllocator.allocateUnassigned(allocation);

    //分片均衡分配器:调用shardsAllocator 的allocate方法分配所有shard
    shardsAllocator.allocate(allocation);
    assert RoutingNodes.assertShardStats(allocation.routingNodes());
}
```
接下来会详细分析gatewayAllocator和shardsAllocator两种Allocator的实现。

## Allocators
Allocator首先分为已经存在shard和新建shard两种情形：

GatewayAllocator：shard已经存在，从集群内已经存在的shard进行复制；
ShardsAllocator：创建新的shard，用于平衡分片在节点间的分布；

### GatewayAllocator
GatewayAllocator有两个重要的属性 primaryShardAllocator和 replicaShardAllocator，对应的类图如下：

primaryShardAllocator显然是针对主分片的，对于主分片allocators只允许把主分片指定在已经拥有该分片完整数据的节点上。

replicaShardAllocator是针对副本分片，对于副本分片allocators则是先判断其他节点上是否已有该分片的数据的拷贝(即便数据不是最新的)。如果有这样的节点，allocators就优先把把分片分配到这其中一个节点。因为副本分片一旦分配，就需要从主分片中进行数据同步，所以当一个节点只拥分片中的部分时，也就意思着那些未拥有的数据必须从主节点中复制得到。这样可以明显的提高副本分片的数据恢复过程。

gatewayAllocator.allocateUnassigned(allocation);是针对已经存在的shrad进行复制操作：
```
public void allocateUnassigned(final RoutingAllocation allocation) {
    assert primaryShardAllocator != null;
    assert replicaShardAllocator != null;
    ensureAsyncFetchStorePrimaryRecency(allocation);
    //关键流程走到GatewayAllocator类的innerAllocatedUnassigned()方法
    innerAllocatedUnassigned(allocation, primaryShardAllocator, replicaShardAllocator);
}
----------------------------------------------------------------------------
protected static void innerAllocatedUnassigned(RoutingAllocation allocation,
                                               PrimaryShardAllocator primaryShardAllocator,
                                               ReplicaShardAllocator replicaShardAllocator) {
    //找到unassigned shard
    RoutingNodes.UnassignedShards unassigned = allocation.routingNodes().unassigned();
    unassigned.sort(PriorityComparator.getAllocationComparator(allocation)); // sort for priority ordering
    //主分片分配器
    primaryShardAllocator.allocateUnassigned(allocation);
    if (allocation.routingNodes().hasInactiveShards()) {
        // cancel existing recoveries if we have a better match
        replicaShardAllocator.processExistingRecoveries(allocation);
    }
    //副分片分配器
    replicaShardAllocator.allocateUnassigned(allocation);
}
```
主分片分配器PrimaryShardAllocator和副本分片分配器ReplicaShardAllocator，执行分配过程中，都会调用父类BaseGatewayShardAllocator的allocateUnassigned方法。
```
public void allocateUnassigned(RoutingAllocation allocation) {
    final RoutingNodes routingNodes = allocation.routingNodes();
    final RoutingNodes.UnassignedShards.UnassignedIterator unassignedIterator = routingNodes.unassigned().iterator();
    //遍历未分配的分片
    while (unassignedIterator.hasNext()) {
        final ShardRouting shard = unassignedIterator.next();
        //调用主分片分配器的makeAllocationDecision进行决策
        final AllocateUnassignedDecision allocateUnassignedDecision = makeAllocationDecision(shard, allocation, logger);

        if (allocateUnassignedDecision.isDecisionTaken() == false) {
            // no decision was taken by this allocator
            continue;
        }

        //根据决策器结果决定是否初始化分片
        if (allocateUnassignedDecision.getAllocationDecision() == AllocationDecision.YES) {
            //主副分片执行同的 unassigned!terator.initialize 函数，将分片的 unassigned 状态改为 initialize 状态
            unassignedIterator.initialize(allocateUnassignedDecision.getTargetNode().getId(),
                allocateUnassignedDecision.getAllocationId(),
                shard.primary() ? ShardRouting.UNAVAILABLE_EXPECTED_SHARD_SIZE :
                                  allocation.clusterInfo().getShardSize(shard, ShardRouting.UNAVAILABLE_EXPECTED_SHARD_SIZE),
                allocation.changes());
        } else {
            unassignedIterator.removeAndIgnore(allocateUnassignedDecision.getAllocationStatus(), allocation.changes());
        }
    }
}
```
主分片分配器PrimaryShardAllocator和副本分片分配器ReplicaShardAllocator的区别是分别实现了不同的makeAllocationDecision方法。

## ShardsAllocator
ShardsAllocator对应的实现类是BalancedShardsAllocator，负责找出拥有分片数最少的节点列表，并按分片数量增序排序，因此分片较少的节点会被优先选择。
```
public void allocate(RoutingAllocation allocation) {
    if (allocation.routingNodes().size() == 0) {
        failAllocationOfNewPrimaries(allocation);
        return;
    }
    final Balancer balancer = new Balancer(logger, allocation, weightFunction, threshold);
    //根据权重算法和decider决定把shard分配到哪个节点。同样将决策后的分配信息更新到集群状态，由Master广播下去
    balancer.allocateUnassigned();
    //对状态为started的分片根据decider来判断是否需要“move"，
    // move过程中此shard的状态被设置为RELOCATING，在目标上创建这个shard时状态为INITIALIZING，同时版本号会加1
    balancer.moveShards();
    //根据权重函数平衡集群模型上的节点
    balancer.balance();
}
```
### （1）allocateUnassigned

根据权重算法WeightFunction和decider决定把shard分配到哪个节点
```
private void allocateUnassigned() {
    //找到未分配分片 UnassignedShards
    RoutingNodes.UnassignedShards unassigned = routingNodes.unassigned();
    if (unassigned.isEmpty()) {
        return;
    }

    /*
     * TODO: We could be smarter here and group the shards by index and then
     * use the sorter to save some iterations.
     */
    //获取Deciders
    final AllocationDeciders deciders = allocation.deciders();
    //按恢复等级排序
    final PriorityComparator secondaryComparator = PriorityComparator.getAllocationComparator(allocation);
    final Comparator<ShardRouting> comparator = (o1, o2) -> {
        if (o1.primary() ^ o2.primary()) {
            return o1.primary() ? -1 : 1;
        }
        final int indexCmp;
        if ((indexCmp = o1.getIndexName().compareTo(o2.getIndexName())) == 0) {
            return o1.getId() - o2.getId();
        }
        final int secondary = secondaryComparator.compare(o1, o2);
        return secondary == 0 ? indexCmp : secondary;
    };
    ShardRouting[] primary = unassigned.drain();
    ShardRouting[] secondary = new ShardRouting[primary.length];
    int secondaryLength = 0;
    int primaryLength = primary.length;
    //把ShardRouting[] primary按照comparator进行排序
    ArrayUtil.timSort(primary, comparator);
    final Set<ModelNode> throttledNodes = Collections.newSetFromMap(new IdentityHashMap<>());

    //每个主分片都要和其他主分片进行对比
    do {
        for (int i = 0; i < primaryLength; i++) {
            ShardRouting shard = primary[i];
            //判断是否能执行分配
            AllocateUnassignedDecision allocationDecision = decideAllocateUnassigned(shard, throttledNodes);
            final String assignedNodeId = allocationDecision.getTargetNode() != null ?
                                              allocationDecision.getTargetNode().getId() : null;
            final ModelNode minNode = assignedNodeId != null ? nodes.get(assignedNodeId) : null;

            if (allocationDecision.getAllocationDecision() == AllocationDecision.YES) {
                //计算shardSize
                final long shardSize = DiskThresholdDecider.getExpectedShardSize(shard, allocation,
                    ShardRouting.UNAVAILABLE_EXPECTED_SHARD_SIZE);
                //能够执行分配，进行shard初始化
                shard = routingNodes.initializeShard(shard, minNode.getNodeId(), null, shardSize, allocation.changes());
                minNode.addShard(shard);
                if (!shard.primary()) {
                    // copy over the same replica shards to the secondary array so they will get allocated
                    // in a subsequent iteration, allowing replicas of other shards to be allocated first
                    while(i < primaryLength-1 && comparator.compare(primary[i], primary[i+1]) == 0) {
                        secondary[secondaryLength++] = primary[++i];
                    }
                }
            } else {
                // did *not* receive a YES decision
                if (minNode != null) {
                    // throttle decision scenario
                    assert allocationDecision.getAllocationStatus() == AllocationStatus.DECIDERS_THROTTLED;
                    final long shardSize = DiskThresholdDecider.getExpectedShardSize(shard, allocation,
                        ShardRouting.UNAVAILABLE_EXPECTED_SHARD_SIZE);
                    minNode.addShard(shard.initialize(minNode.getNodeId(), null, shardSize));
                    final RoutingNode node = minNode.getRoutingNode();
                    //Deciders全部执行，是否满足分配条件
                    final Decision.Type nodeLevelDecision = deciders.canAllocate(node, allocation).type();
                    if (nodeLevelDecision != Type.YES) {
                        assert nodeLevelDecision == Type.NO;
                        throttledNodes.add(minNode);
                    }
                } else {
                }

                unassigned.ignoreShard(shard, allocationDecision.getAllocationStatus(), allocation.changes());
                if (!shard.primary()) { // we could not allocate it and we are a replica - check if we can ignore the other replicas
                    while(i < primaryLength-1 && comparator.compare(primary[i], primary[i+1]) == 0) {
                        unassigned.ignoreShard(primary[++i], allocationDecision.getAllocationStatus(), allocation.changes());
                    }
                }
            }
        }
        primaryLength = secondaryLength;
        ShardRouting[] tmp = primary;
        primary = secondary;
        secondary = tmp;
        secondaryLength = 0;
    } while (primaryLength > 0);
    // clear everything we have either added it or moved to ignoreUnassigned
}
```

（2）moveShards

现根据decideMove方法结果判断是否能够执行move，详细的Move执行过程查看decideMove方法。
```
public void moveShards() {
    // Iterate over the started shards interleaving between nodes, and check if they can remain. In the presence of throttling
    // shard movements, the goal of this iteration order is to achieve a fairer movement of shards from the nodes that are
    // offloading the shards.
    for (Iterator<ShardRouting> it = allocation.routingNodes().nodeInterleavedShardIterator(); it.hasNext(); ) {
        ShardRouting shardRouting = it.next();
        //判断是否能够Move
        final MoveDecision moveDecision = decideMove(shardRouting);
        if (moveDecision.isDecisionTaken() && moveDecision.forceMove()) {
            final ModelNode sourceNode = nodes.get(shardRouting.currentNodeId());
            final ModelNode targetNode = nodes.get(moveDecision.getTargetNode().getId());
            //sourceNode删除
            sourceNode.removeShard(shardRouting);
            //targetNode对shard分配，并进行初始化
            Tuple<ShardRouting, ShardRouting> relocatingShards = routingNodes.relocateShard(shardRouting, targetNode.getNodeId(),
                allocation.clusterInfo().getShardSize(shardRouting,
                                                      ShardRouting.UNAVAILABLE_EXPECTED_SHARD_SIZE), allocation.changes());
            //targetNode新增
            targetNode.addShard(relocatingShards.v2());
}
```
（3）balance
```
private void balance() {
   
    //shard真正fetch数据，直接返回，则禁止rebalance
    if (allocation.hasPendingAsyncFetch()) {
        logger.debug("skipping rebalance due to in-flight shard/store fetches");
        return;
    }
    //根据Deciders判断是否能够执行
    if (allocation.deciders().canRebalance(allocation).type() != Type.YES) {
        logger.trace("skipping rebalance as it is disabled");
        return;
    }
    //nodes是同一个节点直接返回，例如执行：allocate_replica命令
    if (nodes.size() < 2) { /* skip if we only have one node */
        logger.trace("skipping rebalance as single node only");
        return;
    }
    //按照WeightFunction的结果进行balance
    balanceByWeights();
}
```
无论是GatewayAllocator还是ShardsAllocator，在执行过程中判断能否进行move或relocate等操作时，都需要决策器Decider来判断，例如决策器的方法：canRebalance 、canAllocate、canRemain、canForceAllocatePrimary。

## Decider
Decider决策器，在执行Allocate过程中，判断是否满足条件，AllocationDecider是所有Decider的父类。Decider加载的过程在ClusterModule初始化中完成，当前有16个决策器及插件内实现的决策器。
```
//Decider决策器,当前版本有16个+插件包含决策器
public static Collection<AllocationDecider> createAllocationDeciders(Settings settings, ClusterSettings clusterSettings,
                                                                     List<ClusterPlugin> clusterPlugins) {
    // collect deciders by class so that we can detect duplicates
    Map<Class, AllocationDecider> deciders = new LinkedHashMap<>();
    addAllocationDecider(deciders, new MaxRetryAllocationDecider());
    addAllocationDecider(deciders, new ResizeAllocationDecider());
    addAllocationDecider(deciders, new ReplicaAfterPrimaryActiveAllocationDecider());
    addAllocationDecider(deciders, new RebalanceOnlyWhenActiveAllocationDecider());
    addAllocationDecider(deciders, new ClusterRebalanceAllocationDecider(settings, clusterSettings));
    addAllocationDecider(deciders, new ConcurrentRebalanceAllocationDecider(settings, clusterSettings));
    addAllocationDecider(deciders, new EnableAllocationDecider(settings, clusterSettings));
    addAllocationDecider(deciders, new NodeVersionAllocationDecider());
    addAllocationDecider(deciders, new SnapshotInProgressAllocationDecider());
    addAllocationDecider(deciders, new RestoreInProgressAllocationDecider());
    addAllocationDecider(deciders, new FilterAllocationDecider(settings, clusterSettings));
    addAllocationDecider(deciders, new SameShardAllocationDecider(settings, clusterSettings));
    addAllocationDecider(deciders, new DiskThresholdDecider(settings, clusterSettings));
    addAllocationDecider(deciders, new ThrottlingAllocationDecider(settings, clusterSettings));
    addAllocationDecider(deciders, new ShardsLimitAllocationDecider(settings, clusterSettings));
    addAllocationDecider(deciders, new AwarenessAllocationDecider(settings, clusterSettings));

    //插件中包含的Decider决策器
    clusterPlugins.stream()
        .flatMap(p -> p.createAllocationDeciders(settings, clusterSettings).stream())
        .forEach(d -> addAllocationDecider(deciders, d));

    return deciders.values();
}
```
它们继承自AllocationDecider，需要实现的接口有：

canRebalance：当前shard routing是否允许rebalance, 默认是ALWAYS始终允许的；
canAllocate：当前shard routing是否允许分配到目标Node, 默认是ALWAYS始终允许的；
canRemain：在rebalance过程中，当前Shard是否允许留在当前Node,默认是ALWAYS始终允许的；
canForceAllocatePrimary：给定主分片是否可以强制分配在给定节点；

## 负载均衡类
SameShardAllocationDecider: 避免将shard的不同类型(主shard，副本shard)分配到同一个node上，先检查已分配shard的NodeId是否和目标Node相同，相同肯定是不能分配。除了检查NodeId，为了避免分配到同一台机器的不同Node，会检查已分配shard的Node ip和hostname是否和目标Node相同，相同的话也是不允许分配的;
AwarenessAllocationDecider: 感知分配器, 感知服务器, 机架等, 尽量分散存储 Shard;
对应的配置参数有:
cluster.routing.allocation.awareness.attributes: rack_id
cluster.routing.allocation.awareness.attributes: zone
ShardsLimitAllocationDecider: 同一个节点上允许存在同一个 index 的 shard 数目;
index.routing.allocation.total_shards_per_node: 表示该索引每个节点上允许最多的 shard 数量; 默认值=-1, 表示无限制;
cluster.routing.allocation.total_shards_per_node: cluster 级别, 表示集群范围内每个节点上允许最多的 shard 数量, 默认值=-1, 表示无限制;
index 级别会覆盖 cluster 级别;

## 并发控制类
ThrottlingAllocationDecider: recovery 阶段的限速配置, 避免过多的 recovering allocation 导致该节点的负载过高;
cluster.routing.allocation.node_initial_primaries_recoveries: 当前节点在进行主分片恢复时的数量, 默认值=4;
cluster.routing.allocation.node_concurrent_incoming_recoveries: 默认值=2, 通常是其他节点上的副本 shard 恢复到该节点上;
cluster.routing.allocation.node_concurrent_outgoing_recoveries: 默认值=2, 通常是当前节点上的主分片 shard 恢复副本分片到其他节点上;
cluster.routing.allocation.node_concurrent_recoveries: 统一配置上面两个配置项;
ConcurrentRebalanceAllocationDecider: rebalace 并发控制, 表示集群同时允许进行 rebalance 操作的并发数量;
cluster.routing.allocation.cluster_concurrent_rebalance, 默认值=2
通过检查 RoutingNodes 类中维护的 reloadingShard 计数器, 看是否超过配置的并发数;
DiskThresholdDecider: 根据节点的磁盘剩余量来决定是否分配到该节点上;
cluster.routing.allocation.disk.threshold_enabled, 默认值=true;
cluster.routing.allocation.disk.watermark.low: 默认值=85%, 达到这个值后, 新索引的分片不会分配到该节点上;
cluster.routing.allocation.disk.watermark.high: 默认值=90%, 达到这个值后, 会触发已分配到该节点上的 Shard 会 rebalance 到其他节点上去;

## 条件限制类
RebalanceOnlyWhenActiveAllocationDecider: 所有 Shard 都处于 active 状态下才可以执行 rebalance 操作;
FilterAllocationDecider: 通过接口动态设置的过滤器; cluster 级别会覆盖 index 级别;
index.routing.allocation.require.{attribute}
index.routing.allocation.include.{attribute}
index.routing.allocation.exclude.{attribute}
cluster.routing.allocation.require.{attribute}
cluster.routing.allocation.include.{attribute}
cluster.routing.allocation.exclude.{attribute}
require 表示必须满足, include 表示可以分配到指定节点, exclude 表示不允许分配到指定节点;
{attribute} 还有 ES 内置的几个选择, _name, _ip, _host;
ReplicaAfterPrimaryActiveAllocationDecider: 保证只在主分片分配完成后(active 状态)才开始分配副本分片;
ClusterRebalanceAllocationDecider: 通过集群中 active 的 shard 状态来决定是否可以执行 rebalance;
cluster.routing.allocation.allow_rebalance
indices_all_active(默认): 当集群所有的节点分配完成, 才可以执行 rebalance 操作;
indices_primaries_active: 只要所有主分片分配完成, 才可以执行 rebalance 操作;
always: 任何情况下都允许 rebalance 操作;
MaxRetryAllocationDecider: 防止 shard 在失败次数达到上限后继续分配;
index.allocation.max_retries: 设置分配的最大失败重试次数, 默认值=5;

## 其他决策类
EnableAllocationDecider: 设置允许分配的分片类型; index 级别配置会覆盖 cluster 级别配置;
all(默认): 允许所有类型的分片;
primaries: 仅允许主分片;
new_primaries: 仅允许新建索引的主分片;
none: 禁止分片分配操作;
NodeVersionAllocationDecider: 检查分片所在 Node 的版本是否高于目标 Node 的 ES 版本;
SnapshotInProgressAllocationDecider: 决定 snapshot 期间是否允许 allocation, 因为 snapshot 只会发生在主分片上, 所以该配置只会限制主分片的 allocation;
cluster.routing.allocation.snapshot.relocation_enabled






