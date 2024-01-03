---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码集群服务
Master节点操作的Action（继承TransportMasterNodeAction），会把任务提交到ClusterSerevice，并由MasterService类执行。

## ClusterSerevice
重要属性

ClusterSerevice最重要的两个属性如下：
```
//MasterService提供任务提交接口，内部维护一个线程池处理更新任务
private final MasterService masterService;
//ClusterApplierService则负责通知各个模块应用新生成的集群状态
private final ClusterApplierService clusterApplierService;
```
重要方法

（1）增加一个集群状态处理器，ClusterStateApplier是个接口，某一个服务修改集群状态，实现ClusterStateApplier接口即可。
```
public void addStateApplier(ClusterStateApplier applier) {
    clusterApplierService.addStateApplier(applier);
}
```
（2）增加一个集群状态监听器，ClusterStateListener是个接口，某一服务如果需要监听集群状态，实现ClusterStateListener接口即可。
```
public void addListener(ClusterStateListener listener) {
    clusterApplierService.addListener(listener);
}
```
同理，还有删除状态处理器或监听器的方法，这里不再描述。

（2）提交集群任务，提交任务有3个重载方法

1. void submitStateUpdateTask(String source, T updateTask)不支持批量提交
```
public <T extends ClusterStateTaskConfig & ClusterStateTaskExecutor<T> & ClusterStateTaskListener>
    void submitStateUpdateTask(String source, T updateTask) {
    //ClusterStateUpdateTask实现ClusterStateUpdateTask实现ClusterStateTaskConfig, ClusterStateTaskExecutor, ClusterStateTaskListener接口
    // 这里updateTask, updateTask, updateTask, updateTask分别获取对应的子类
    // T updateTask入参需要继承ClusterStateUpdateTask
    submitStateUpdateTask(source, updateTask, updateTask, updateTask, updateTask);
}
```
这里注意：T updateTask应该实现 ClusterStateTaskConfig ClusterStateTaskExecutor ClusterStateTaskListener三个接口。

2. 支持批量提交任务
```
submitStateUpdateTask(String source, T task,
                                          ClusterStateTaskConfig config,
                                          ClusterStateTaskExecutor<T> executor,
                                          ClusterStateTaskListener listener)
```

3. 所有提交任务最终都会调用如下方法：
```
/**
 * Submits a batch of cluster state update tasks; submitted updates are guaranteed to be processed together,
 * potentially with more tasks of the same executor.
 *
 * @param source   the source of the cluster state update task
 * @param tasks    a map of update tasks and their corresponding listeners
 * @param config   the cluster state update task configuration
 * @param executor the cluster state update task executor; tasks
 *                 that share the same executor will be executed
 *                 batches on this executor
 * @param <T>      the type of the cluster state update task state
 *
 */
public <T> void submitStateUpdateTasks(final String source,
                                       final Map<T, ClusterStateTaskListener> tasks, 
                                       final ClusterStateTaskConfig config,
                                       final ClusterStateTaskExecutor<T> executor) {
    masterService.submitStateUpdateTasks(source, tasks, config, executor);
}
```

## MasterService
MasterService提供任务提交接口，内部维护一个线程池处理更新任务。只有主节点会执行这个类中的方法。也就是说，只有主节点会提交集群任务到内部的队列，并运行队列中的任务。

### 线程池
线程池的属性是threadPoolExecutor
```
//运行集群任务的线程池
private volatile PrioritizedEsThreadPoolExecutor threadPoolExecutor;
```
线程池的初始化：
```
protected PrioritizedEsThreadPoolExecutor createThreadPoolExecutor() {
    //1.固定单线程的线程池
    //该线程池的名称为 (nodeName + "/"+ masterService#updateTask)
    return EsExecutors.newSinglePrioritizing(
            nodeName + "/" + MASTER_UPDATE_THREAD_NAME,
            daemonThreadFactory(nodeName, MASTER_UPDATE_THREAD_NAME),
            threadPool.getThreadContext(),
            threadPool.scheduler());
}
```
线程池类型提到，PrioritizedEsThreadPoolExecutor是固定单线程池，core pool size == max pool size ==1，说明该线程池里面只有一个工作线程。队列是阻塞优先队列PriorityBlockingQueue，能够区分优先级，如果两个任务优先级相同则按照先进先出（FIFO）方式执行。

因此在MasterService中，任务Task在线程池中是单线程执行。

## 任务Task
ClusterService支持提交批量任务，TaskBatcher是个抽象类（支持PrioritizedEsThreadPoolExecutor的批量任务），Batcher是TaskBatcher的具体实现类。批量提交的任务必须具有同一个batchingKey，保证任务按照顺序执行。

BatchedTask继承SourcePrioritizedRunnable，是抽象类TaskBatcher的子类，支持运行批量任务。UpdateTask是BatchedTask的子类，本质上是一个具有优先级特征的Runnable任务，是抽象类PrioritizedRunnable的实现，
能够在PrioritizedEsThreadPoolExecutor中运行。

## 任务提交
各个模块触发的集群元数据更新最终调用submitStateUpdateTasks方法，即无论是否为批量提交最终调用taskBatcher.submitTasks提交任务。 参数描述：

source：任务描述；
tasks：是一个Map，Key是具体任务，Value则是对应的任务监听器；
config：参数配置，这里核心是超时参数配置；
executor：执行任务的线程池；
```
public <T> void submitStateUpdateTasks(final String source,
                                       final Map<T, ClusterStateTaskListener> tasks, 
                                       final ClusterStateTaskConfig config,
                                       final ClusterStateTaskExecutor<T> executor) {
    if (!lifecycle.started()) {
        return;
    }
    final ThreadContext threadContext = threadPool.getThreadContext();
    //当返回的生产者被调用时，执行restores
    final Supplier<ThreadContext.StoredContext> supplier = threadContext.newRestorableContext(true);
    //下上下文暂存
    try (ThreadContext.StoredContext ignore = threadContext.stashContext()) {
        threadContext.markAsSystemContext();
        //封装为UpdateTask(抽象类PrioritizedRunnable的实现,能够在PrioritizedEsThreadPoolExecutor中运行)
        List<Batcher.UpdateTask> safeTasks = tasks.entrySet().stream()
            //safe封装新的Listener
            .map(e -> taskBatcher.new UpdateTask(config.priority(), source, e.getKey(), safe(e.getValue(), supplier), executor))
            .collect(Collectors.toList());
        //提交到TaskBatcher
        taskBatcher.submitTasks(safeTasks, config.timeout());
    }
```

## 集群状态发布
在MasterService类runTasks方法中，calculateTaskOutputs执行具体的任务，并根据previousClusterState及任务执行结果clusterTasksResult，计算出新的集群状态newClusterState。根据两个集群状态执行以下逻辑：

集群状态未发生变化，触发集群状态未发生变化的监听器
集群状态发生变化，通知其他节点

## ClusterApplierService
ClusterApplierService则负责通知各个模块应用新生成的集群状态

### 修改集群状态
如果某一模块需要修改集群状态，实现ClusterStateApplier接口的applyClusterState方法即可。实现ClusterStateApplier接口后，在子类中调用addStateApplier将类的实例添加到Applie列表。当应用集群状态时，会遍历这个列表通知各个模块执行应用。

以IndicesClusterStateService为例，实现ClusterStateApplier接口。
```
@Override
public synchronized void applyClusterState(final ClusterChangedEvent event) {
    if (!lifecycle.started()) {
        return;
    }

    final ClusterState state = event.state();
```
添加到Applier列表
```
protected void doStart() {
    // Doesn't make sense to manage shards on non-master and non-data nodes
    if (DiscoveryNode.isDataNode(settings) || DiscoveryNode.isMasterNode(settings)) {
        clusterService.addHighPriorityApplier(this);
    }
}
```
监听集群状态
若某一组件需要监听集群元数据发生变更，实现ClusterStateListener接口的clusterChanged方法即可。实现ClusterStateListener接口后，在实现类中调用addListener将类的实例添加到Listene列表。当集群状态应用完毕，会遍历这个列表通知各个模块集群状态已发生变化。

以IndicesStore为例，实现ClusterStateListener接口的clusterChanged方法：
```
@Override
public void clusterChanged(ClusterChangedEvent event) {
    if (!event.routingTableChanged()) {
        return;
    }

    if (event.state().blocks().disableStatePersistence()) {
        return;
    }

    RoutingTable routingTable = event.state().routingTable();
```
在实现类中调用addListener将类的实例添加到Listene列表：
```
// Doesn't make sense to delete shards on non-data nodes
if (DiscoveryNode.isDataNode(settings)) {
    // we double check nothing has changed when responses come back from other nodes.
    // it's easier to do that check when the current cluster state is visible.
    // also it's good in general to let things settle down
    clusterService.addListener(this);
}
```

## 线程池
修改集群状态有线程池属性threadPoolExecutor执行，对应的实现方法和2.1 线程池基本一致。
```
//1.固定单线程的线程池
protected PrioritizedEsThreadPoolExecutor createThreadPoolExecutor() {
    //线程池名称：nodeName + "/" +clusterApplierService#updateTask
    return EsExecutors.newSinglePrioritizing(
        nodeName + "/" + CLUSTER_UPDATE_THREAD_NAME,
        daemonThreadFactory(nodeName, CLUSTER_UPDATE_THREAD_NAME),
        threadPool.getThreadContext(),
        threadPool.scheduler());
}
```
固定单线程的线程池保证了修改集群状态的任务是单线程执行，这样简化很多操作。例如：一个请求要删除索引test，一个请求要修改索引test的settings，由于是单线程执行不用考虑并发情况。这样的思路在Kafka的设计中能看到，Controller监听Zookeeper的事件，放入队列，并由单线程从队列中获取任务并执行。

## Apply集群状态
```
public void onNewClusterState(final String source, final Supplier<ClusterState> clusterStateSupplier,
                              final ClusterApplyListener listener) {
    Function<ClusterState, ClusterState> applyFunction = currentState -> {
        ClusterState nextState = clusterStateSupplier.get();
        if (nextState != null) {
            return nextState;
        } else {
            return currentState;
        }
    };
    submitStateUpdateTask(source, ClusterStateTaskConfig.build(Priority.HIGH), applyFunction, listener);
}
---------------------------------------------
//在新的线程池应用集群状态
private void submitStateUpdateTask(final String source, final ClusterStateTaskConfig config,
                                   final Function<ClusterState, ClusterState> executor,
                                   final ClusterApplyListener listener) {
    if (!lifecycle.started()) {
        return;
    }
    try {
        UpdateTask updateTask = new UpdateTask(config.priority(), source, new SafeClusterApplyListener(listener, logger), executor);
        if (config.timeout() != null) {
            threadPoolExecutor.execute(updateTask, config.timeout(),
                () -> threadPool.generic().execute(
                    () -> listener.onFailure(source, new ProcessClusterEventTimeoutException(config.timeout(), source))));
        } else {
            threadPoolExecutor.execute(updateTask);
        }
    }
```
线程池执行UpdateTask任务
```
class UpdateTask extends SourcePrioritizedRunnable implements Function<ClusterState, ClusterState> {
    final ClusterApplyListener listener;
    final Function<ClusterState, ClusterState> updateFunction;

    UpdateTask(Priority priority, String source, ClusterApplyListener listener,
               Function<ClusterState, ClusterState> updateFunction) {
        super(priority, source);
        this.listener = listener;
        this.updateFunction = updateFunction;
    }

    @Override
    public ClusterState apply(ClusterState clusterState) {
        return updateFunction.apply(clusterState);
    }
```
updateFunction的定义在onNewClusterState方法
```
    Function<ClusterState, ClusterState> applyFunction = currentState -> {
        ClusterState nextState = clusterStateSupplier.get();
        if (nextState != null) {
            return nextState;
        } else {
            return currentState;
        }
    };
```
UpdateTask的run方法对应代码：
```
private void runTask(UpdateTask task) {
   ..................
    final ClusterState previousClusterState = state.get();

    long startTimeMS = currentTimeInMillis();
    final StopWatch stopWatch = new StopWatch();
    final ClusterState newClusterState;
    try {
        try (Releasable ignored = stopWatch.timing("running task [" + task.source + ']')) {
            //apply对应applyFunction
            newClusterState = task.apply(previousClusterState);
        }
    }
    
    //检查应用的集群状态是否与上一个相同，如果相同则不应用
    if (previousClusterState == newClusterState) {
        //集群状态ClusterState元数据未发生变化
        TimeValue executionTime = TimeValue.timeValueMillis(Math.max(0, currentTimeInMillis() - startTimeMS));
        logger.debug("processing [{}]: took [{}] no change in cluster state", task.source, executionTime);
        warnAboutSlowTaskIfNeeded(executionTime, task.source, stopWatch);
        task.listener.onSuccess(task.source);
    } else {
        
        //集群状态ClusterState元数据发生变化
        try {
            //在正常情况下，进入applyChanges应用集群状态，该方法中正式调用各个模块的Applier和Listener，并更新本节点存储的集群状态。
            applyChanges(task, previousClusterState, newClusterState, stopWatch);
            TimeValue executionTime = TimeValue.timeValueMillis(Math.max(0, currentTimeInMillis() - startTimeMS));
            logger.debug("processing [{}]: took [{}] done applying updated cluster state (version: {}, uuid: {})", task.source,
                executionTime, newClusterState.version(),
                newClusterState.stateUUID());
            warnAboutSlowTaskIfNeeded(executionTime, task.source, stopWatch);
            task.listener.onSuccess(task.source);
        } catch (Exception e) {
    }
```
在正常情况下，进入applyChanges应用集群状态，该方法中正式调用各个模块的Applier和Listener，并更新本节点存储的集群状态。
```
//在正常情况下，进入applyChanges应用集群状态，该方法中正式调用各个模块的Applier和Listener，并更新本节点存储的集群状态。
private void applyChanges(UpdateTask task, ClusterState previousClusterState, ClusterState newClusterState, StopWatch stopWatch) {
    ClusterChangedEvent clusterChangedEvent = new ClusterChangedEvent(task.source, newClusterState, previousClusterState);
    // new cluster state, notify all listeners
    final DiscoveryNodes.Delta nodesDelta = clusterChangedEvent.nodesDelta();

    logger.trace("connecting to nodes of cluster state with version {}", newClusterState.version());
    try (Releasable ignored = stopWatch.timing("connecting to new nodes")) {
        connectToNodesAndWait(newClusterState);
    }

    // nothing to do until we actually recover from the gateway or any other block indicates we need to disable persistency
    if (clusterChangedEvent.state().blocks().disableStatePersistence() == false && clusterChangedEvent.metaDataChanged()) {
        logger.debug("applying settings from cluster state with version {}", newClusterState.version());
        final Settings incomingSettings = clusterChangedEvent.state().metaData().settings();
        try (Releasable ignored = stopWatch.timing("applying settings")) {
            clusterSettings.applySettings(incomingSettings);
        }
    }

    //通知所有Appliers
    logger.debug("apply cluster state with version {}", newClusterState.version());
    // 进行元数据应用，会按优先级分为 low/normal/high 来应用，例如创建索引属于 high
    callClusterStateAppliers(clusterChangedEvent, stopWatch);

    nodeConnectionsService.disconnectFromNodesExcept(newClusterState.nodes());

    //更新本 类存储的集群状态信息
    logger.debug("set locally applied cluster state to version {}", newClusterState.version());
    // 更新本地最新 commited 的 clusterState
    state.set(newClusterState);

    //通知所有Listeners
    // 这里一般是 master 节点发起元数据 commit 结束后，再回调相应的 listener
    callClusterStateListeners(clusterChangedEvent, stopWatch);
}
```
callClusterStateAppliers是应用新的集群状态，不同的业务实现不同功能。创建索引对应的服务是IndicesClusterStateService类实现的applyClusterState方法。














