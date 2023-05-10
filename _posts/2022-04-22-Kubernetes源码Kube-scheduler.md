---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes源码调度
kube-scheduler组件是Kubernetes系统的核心组件之一，主要负责整个集群Pod资源对象的调度，根据内置或扩展的调度算法（预选与优选调度算法），将未调度的Pod资源对象调度到最优的工作节点上，从而更加合理、更加充分地利用集群的资源。

## kube-scheduler简介
kube-scheduler 负责分配调度 Pod 到集群内的节点上，它监听 kube-apiserver，查询还未分配 Node 的 Pod，然后根据调度策略为这些 Pod 分配节点（更新 Pod 的 NodeName 字段）。
调度器需要充分考虑诸多的因素：
- 公平调度
- 资源高效利用
- QoS
- affinity 和 anti-affinity
- 数据本地化（data locality）
- 内部负载干扰（inter-workload interference）
- deadlines

## kube-scheduler组件的启动流程
在kube-scheduler组件的启动流程中，分别介绍如下。
- 内置调度算法的注册。
- Cobra命令行参数解析。
- 实例化Scheduler对象。
- 运行EventBroadcaster事件管理器。
- 运行HTTP或HTTPS服务。
- 运行Informer同步资源。
- 领导者选举实例化。
- 运行sched.Run调度器。






-------
## 指定 Node 节点调度
有三种方式指定 Pod 只运行在指定的 Node 节点上
- nodeSelector：只调度到匹配指定 label 的 Node 上
- nodeAffinity：功能更丰富的 Node 选择器，比如支持集合操作
- podAffinity：调度到满足条件的 Pod 所在的 Node 上

### nodeSelector 示例

首先给 Node 打上标签
```shell
   kubectl label nodes node-01 disktype=ssd
```
然后在 daemonset 中指定 nodeSelector 为 disktype=ssd：
```shell
   spec:
    nodeSelector:
    disktype: ssd
```

### nodeAffinity 示例
nodeAffinity 目前支持两种：requiredDuringSchedulingIgnoredDuringExecution 和 preferredDuringSchedulingIgnoredDuringExecution，分别代表必须满足条件和优选条件。比如下面的例子代表调度到包含标签 kubernetes.io/e2e-az-name 并且值为 e2e-az1 或 e2e-az2 的 Node 上，并且优选还带有标签 another-node-label-key=another-node-label-value 的 Node。
```yaml
   apiVersion: v1
   kind: Pod
   metadata:
    name: with-node-affinity
   spec:
    affinity:
    nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchExpressions:
    - key: kubernetes.io/e2e-az-name
    operator: In
    values:
    - e2e-az1
    - e2e-az2
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 1
    preference:
    matchExpressions:
    - key: another-node-label-key
    operator: In
    values:
    - another-node-label-value
    containers:
    - name: with-node-affinity
    image: gcr.io/google_containers/pause:2.0
```

### podAffinity 示例

podAffinity 基于 Pod 的标签来选择 Node，仅调度到满足条件 Pod 所在的 Node 上，支持 podAffinity 和 podAntiAffinity。这个功能比较绕，以下面的例子为例：
- 如果一个 “Node 所在 Zone 中包含至少一个带有 security=S1 标签且运行中的 Pod”，那么可以调度到该 Node
- 不调度到 “包含至少一个带有 security=S2 标签且运行中 Pod” 的 Node 上

```yaml
   apiVersion: v1
   kind: Pod
   metadata:
    name: with-pod-affinity
   spec:
    affinity:
    podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
    matchExpressions:
    - key: security
    operator: In
    values:
    - S1
    topologyKey: failure-domain.beta.kubernetes.io/zone
    podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
    podAffinityTerm:
    labelSelector:
    matchExpressions:
    - key: security
    operator: In
    values:
    - S2
    topologyKey: kubernetes.io/hostname
    containers:
    - name: with-pod-affinity
    image: gcr.io/google_containers/pause:2.0
```

## Taints 和 tolerations
Taints 和 tolerations 用于保证 Pod 不被调度到不合适的 Node 上，其中 Taint 应用于 Node 上，而 toleration 则应用于 Pod 上。
目前支持的 taint 类型
- NoSchedule：新的 Pod 不调度到该 Node 上，不影响正在运行的 Pod
- PreferNoSchedule：soft 版的 NoSchedule，尽量不调度到该 Node 上
- NoExecute：新的 Pod 不调度到该 Node 上，并且删除（evict）已在运行的 Pod。Pod 可以增加一个时间（tolerationSeconds）
然而，当 Pod 的 Tolerations 匹配 Node 的所有 Taints 的时候可以调度到该 Node 上；当 Pod 是已经运行的时候，也不会被删除（evicted）。另外对于 NoExecute，如果 Pod 增加了一个 tolerationSeconds，则会在该时间之后才删除 Pod。

比如，假设 node1 上应用以下几个 taint
```shell
   kubectl taint nodes node1 key1=value1:NoSchedule
   kubectl taint nodes node1 key1=value1:NoExecute
   kubectl taint nodes node1 key2=value2:NoSchedule
 
```
下面的这个 Pod 由于没有 toleratekey2=value2:NoSchedule 无法调度到 node1 上
```yaml
   tolerations:
   - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
   - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoExecute"
```
而正在运行且带有 tolerationSeconds 的 Pod 则会在 600s 之后删除
```yaml
   tolerations:
   - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
   - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoExecute"
    tolerationSeconds: 600
   - key: "key2"
    operator: "Equal"
    value: "value2"
    effect: "NoSchedule"
```



### kube-scheduler 的设计

Kube-scheduler 是 kubernetes 的核心组件之一，也是所有核心组件之间功能比较单一的，其代码也相对容易理解。kube-scheduler 的目的就是为每一个 pod 选择一个合适的 node，整体流程可以概括为三步，获取未调度的 podList，通过执行一系列调度算法为 pod 选择一个合适的 node，提交数据到 apiserver，其核心则是一系列调度算法的设计与执行。


官方对 kube-scheduler 的调度流程描述 [The Kubernetes Scheduler](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduler.md)：

```
For given pod:

    +---------------------------------------------+
    |               Schedulable nodes:            |
    |                                             |
    | +--------+    +--------+      +--------+    |
    | | node 1 |    | node 2 |      | node 3 |    |
    | +--------+    +--------+      +--------+    |
    |                                             |
    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+

    Pred. filters: node 3 doesn't have enough resource

    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+
    |             remaining nodes:                |
    |   +--------+                 +--------+     |
    |   | node 1 |                 | node 2 |     |
    |   +--------+                 +--------+     |
    |                                             |
    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+

    Priority function:    node 1: p=2
                          node 2: p=5

    +-------------------+-------------------------+
                        |
                        |
                        v
            select max{node priority} = node 2
```

kube-scheduler 目前包含两部分调度算法 predicates 和 priorities，首先执行 predicates 算法过滤部分 node 然后执行  priorities 算法为所有 node 打分，最后从所有 node 中选出分数最高的最为最佳的 node。



### kube-scheduler 源码分析

> kubernetes 版本: v1.16


kubernetes 中所有组件的启动流程都是类似的，首先会解析命令行参数、添加默认值，kube-scheduler 的默认参数在 `k8s.io/kubernetes/pkg/scheduler/apis/config/v1alpha1/defaults.go` 中定义的。然后会执行  run 方法启动主逻辑，下面直接看 kube-scheduler 的主逻辑 run 方法执行过程。


`Run()` 方法主要做了以下工作：
- 初始化 scheduler 对象
- 启动 kube-scheduler server，kube-scheduler 监听 10251 和 10259 端口，10251 端口不需要认证，可以获取 healthz metrics 等信息，10259 为安全端口，需要认证
- 启动所有的 informer
- 执行 `sched.Run()` 方法，执行主调度逻辑

`k8s.io/kubernetes/cmd/kube-scheduler/app/server.go:160`

```
func Run(cc schedulerserverconfig.CompletedConfig, stopCh <-chan struct{}, registryOptions ...Option) error {
    ......
    // 1、初始化 scheduler 对象
    sched, err := scheduler.New(......）
    if err != nil {
        return err
    }

    // 2、启动事件广播
    if cc.Broadcaster != nil && cc.EventClient != nil {
        cc.Broadcaster.StartRecordingToSink(stopCh)
    }
    if cc.LeaderElectionBroadcaster != nil && cc.CoreEventClient != nil {
        cc.LeaderElectionBroadcaster.StartRecordingToSink(&corev1.EventSinkImpl{Interface: cc.CoreEventClient.Events("")})
    }

    ......
    // 3、启动 http server
    if cc.InsecureServing != nil {
        separateMetrics := cc.InsecureMetricsServing != nil
        handler := buildHandlerChain(newHealthzHandler(&cc.ComponentConfig, separateMetrics, checks...), nil, nil)
        if err := cc.InsecureServing.Serve(handler, 0, stopCh); err != nil {
            return fmt.Errorf("failed to start healthz server: %v", err)
        }
    }
    ......
    // 4、启动所有 informer
    go cc.PodInformer.Informer().Run(stopCh)
    cc.InformerFactory.Start(stopCh)

    cc.InformerFactory.WaitForCacheSync(stopCh)

    run := func(ctx context.Context) {
        sched.Run()
        <-ctx.Done()
    }

    ctx, cancel := context.WithCancel(context.TODO()) // TODO once Run() accepts a context, it should be used here
    defer cancel()
    go func() {
        select {
        case <-stopCh:
            cancel()
        case <-ctx.Done():
        }
    }()

    // 5、选举 leader
    if cc.LeaderElection != nil {
        ......
    }
    // 6、执行 sched.Run() 方法
    run(ctx)
    return fmt.Errorf("finished without leader elect")
}
```



下面看一下 `scheduler.New()` 方法是如何初始化 scheduler 结构体的，该方法主要的功能是初始化默认的调度算法以及默认的调度器 GenericScheduler。

- 创建 scheduler 配置文件
- 根据默认的 DefaultProvider 初始化  schedulerAlgorithmSource 然后加载默认的预选及优选算法，然后初始化 GenericScheduler
- 若启动参数提供了 policy config 则使用其覆盖默认的预选及优选算法并初始化 GenericScheduler，不过该参数现已被弃用

`k8s.io/kubernetes/pkg/scheduler/scheduler.go:166`
```
func New(......) (*Scheduler, error) {
	......
    // 1、创建 scheduler 的配置文件
    configurator := factory.NewConfigFactory(&factory.ConfigFactoryArgs{
  	    ......
  	})
    var config *factory.Config
    source := schedulerAlgorithmSource
    // 2、加载默认的调度算法
    switch {
    case source.Provider != nil:
        // 使用默认的 ”DefaultProvider“ 初始化 config
        sc, err := configurator.CreateFromProvider(*source.Provider)
        if err != nil {
            return nil, fmt.Errorf("couldn't create scheduler using provider %q: %v", *source.Provider, err)
        }
        config = sc
    case source.Policy != nil:
        // 通过启动时指定的 policy source 加载 config
	......
        config = sc
    default:
        return nil, fmt.Errorf("unsupported algorithm source: %v", source)
    }
    // Additional tweaks to the config produced by the configurator.
    config.Recorder = recorder
    config.DisablePreemption = options.disablePreemption
    config.StopEverything = stopCh

    // 3.创建 scheduler 对象
    sched := NewFromConfig(config)
	......
    return sched, nil
}
```



下面是 pod informer 的启动逻辑，只监听 status.phase 不为 succeeded 以及 failed 状态的 pod，即非 terminating 的 pod。

`k8s.io/kubernetes/pkg/scheduler/factory/factory.go:527`
````
func NewPodInformer(client clientset.Interface, resyncPeriod time.Duration) coreinformers.PodInformer {
    selector := fields.ParseSelectorOrDie(
        "status.phase!=" + string(v1.PodSucceeded) +
            ",status.phase!=" + string(v1.PodFailed))
    lw := cache.NewListWatchFromClient(client.CoreV1().RESTClient(), string(v1.ResourcePods), metav1.NamespaceAll, selector)
    return &podInformer{
        informer: cache.NewSharedIndexInformer(lw, &v1.Pod{}, resyncPeriod, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}),
    }
}
````



然后继续看 `Run()`  方法中最后执行的 `sched.Run()` 调度循环逻辑，若 informer 中的 cache 同步完成后会启动一个循环逻辑执行 `sched.scheduleOne` 方法。

`k8s.io/kubernetes/pkg/scheduler/scheduler.go:313`
```
func (sched *Scheduler) Run() {
    if !sched.config.WaitForCacheSync() {
        return
    }

    go wait.Until(sched.scheduleOne, 0, sched.config.StopEverything)
}
```



`scheduleOne()` 每次对一个 pod 进行调度，主要有以下步骤：

- 从 scheduler 调度队列中取出一个 pod，如果该 pod 处于删除状态则跳过
- 执行调度逻辑 `sched.schedule()` 返回通过预算及优选算法过滤后选出的最佳 node
- 如果过滤算法没有选出合适的 node，则返回 core.FitError
- 若没有合适的 node 会判断是否启用了抢占策略，若启用了则执行抢占机制
- 判断是否需要 VolumeScheduling 特性
- 执行 reserve plugin
- pod 对应的 spec.NodeName 写上 scheduler 最终选择的 node，更新 scheduler cache
- 请求 apiserver 异步处理最终的绑定操作，写入到 etcd
- 执行 permit plugin
- 执行 prebind plugin
- 执行 postbind plugin

`k8s.io/kubernetes/pkg/scheduler/scheduler.go:515`
```
func (sched *Scheduler) scheduleOne() {
    fwk := sched.Framework

    pod := sched.NextPod()
    if pod == nil {
        return
    }
    // 1.判断 pod 是否处于删除状态
    if pod.DeletionTimestamp != nil {
    	......
    }
		
    // 2.执行调度策略选择 node
    start := time.Now()
    pluginContext := framework.NewPluginContext()
    scheduleResult, err := sched.schedule(pod, pluginContext)
    if err != nil {
        if fitError, ok := err.(*core.FitError); ok {
            // 3.若启用抢占机制则执行
            if sched.DisablePreemption {
            	......
            } else {
                preemptionStartTime := time.Now()
                sched.preempt(pluginContext, fwk, pod, fitError)
                ......
            }
            ......
            metrics.PodScheduleFailures.Inc()
        } else {
            klog.Errorf("error selecting node for pod: %v", err)
            metrics.PodScheduleErrors.Inc()
        }
        return
    }
    ......
    assumedPod := pod.DeepCopy()

    // 4.判断是否需要 VolumeScheduling 特性
    allBound, err := sched.assumeVolumes(assumedPod, scheduleResult.SuggestedHost)
    if err != nil {
        klog.Errorf("error assuming volumes: %v", err)
        metrics.PodScheduleErrors.Inc()
        return
    }

    // 5.执行 "reserve" plugins
    if sts := fwk.RunReservePlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost); !sts.IsSuccess() {
        .....
    }

    // 6.为 pod 设置 NodeName 字段，更新 scheduler 缓存
    err = sched.assume(assumedPod, scheduleResult.SuggestedHost)
    if err != nil {
        ......
    }

    // 7.异步请求 apiserver
    go func() {
        // Bind volumes first before Pod
        if !allBound {
            err := sched.bindVolumes(assumedPod)
            if err != nil {
				......
                return
            }
        }

        // 8.执行 "permit" plugins
        permitStatus := fwk.RunPermitPlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost)
        if !permitStatus.IsSuccess() {
			......
        }
        // 9.执行 "prebind" plugins
        preBindStatus := fwk.RunPreBindPlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost)
        if !preBindStatus.IsSuccess() {
            ......
        }
        err := sched.bind(assumedPod, scheduleResult.SuggestedHost, pluginContext)
        ......
        if err != nil {
            ......
        } else {
            ......
            // 10.执行 "postbind" plugins
            fwk.RunPostBindPlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost)
        }
    }()
}
```



`scheduleOne()`  中通过调用 `sched.schedule()` 来执行预选与优选算法处理：

`k8s.io/kubernetes/pkg/scheduler/scheduler.go:337`
```
func (sched *Scheduler) schedule(pod *v1.Pod, pluginContext *framework.PluginContext) (core.ScheduleResult, error) {
    result, err := sched.Algorithm.Schedule(pod, pluginContext)
    if err != nil {
	......
    }
    return result, err
}
```



`sched.Algorithm` 是一个 interface，主要包含四个方法，GenericScheduler 是其具体的实现：

`k8s.io/kubernetes/pkg/scheduler/core/generic_scheduler.go:131`
```
type ScheduleAlgorithm interface {
    Schedule(*v1.Pod, *framework.PluginContext) (scheduleResult ScheduleResult, err error)
    Preempt(*framework.PluginContext, *v1.Pod, error) (selectedNode *v1.Node, preemptedPods []*v1.Pod, cleanupNominatedPods []*v1.Pod, err error)
    Predicates() map[string]predicates.FitPredicate
    Prioritizers() []priorities.PriorityConfig
}
```

- `Schedule()`：正常调度逻辑，包含预算与优选算法的执行
- `Preempt()`：抢占策略，在 pod 调度发生失败的时候尝试抢占低优先级的 pod，函数返回发生抢占的 node，被 抢占的 pods 列表，nominated node name 需要被移除的 pods 列表以及 error
- `Predicates()`：predicates 算法列表
- `Prioritizers()`：prioritizers 算法列表



kube-scheduler 提供的默认调度为 DefaultProvider，DefaultProvider 配置的 predicates 和 priorities policies 在 `k8s.io/kubernetes/pkg/scheduler/algorithmprovider/defaults/defaults.go` 中定义，算法具体实现是在 `k8s.io/kubernetes/pkg/scheduler/algorithm/predicates/` 和`k8s.io/kubernetes/pkg/scheduler/algorithm/priorities/` 中，默认的算法如下所示：

`pkg/scheduler/algorithmprovider/defaults/defaults.go`
```
func defaultPredicates() sets.String {
    return sets.NewString(
        predicates.NoVolumeZoneConflictPred,
        predicates.MaxEBSVolumeCountPred,
        predicates.MaxGCEPDVolumeCountPred,
        predicates.MaxAzureDiskVolumeCountPred,
        predicates.MaxCSIVolumeCountPred,
        predicates.MatchInterPodAffinityPred,
        predicates.NoDiskConflictPred,
        predicates.GeneralPred,
        predicates.CheckNodeMemoryPressurePred,
        predicates.CheckNodeDiskPressurePred,
        predicates.CheckNodePIDPressurePred,
        predicates.CheckNodeConditionPred,
        predicates.PodToleratesNodeTaintsPred,
        predicates.CheckVolumeBindingPred,
    )
}

func defaultPriorities() sets.String {
    return sets.NewString(
        priorities.SelectorSpreadPriority,
        priorities.InterPodAffinityPriority,
        priorities.LeastRequestedPriority,
        priorities.BalancedResourceAllocation,
        priorities.NodePreferAvoidPodsPriority,
        priorities.NodeAffinityPriority,
        priorities.TaintTolerationPriority,
        priorities.ImageLocalityPriority,
    )
}
```



下面继续看 `sched.Algorithm.Schedule()` 调用具体调度算法的过程：

- 检查 pod pvc 信息
- 执行 prefilter plugins
- 获取 scheduler cache 的快照，每次调度 pod 时都会获取一次快照
- 执行 `g.findNodesThatFit()` 预选算法
- 执行 postfilter plugin
- 若 node  为 0 直接返回失败的 error，若 node 数为1 直接返回该 node
- 执行 `g.priorityMetaProducer()` 获取 metaPrioritiesInterface，计算 pod 的metadata，检查该 node 上是否有相同 meta 的 pod
- 执行 `PrioritizeNodes()` 算法
- 执行 `g.selectHost()` 通过得分选择一个最佳的 node

`k8s.io/kubernetes/pkg/scheduler/core/generic_scheduler.go:186`
```
func (g *genericScheduler) Schedule(pod *v1.Pod, pluginContext *framework.PluginContext) (result ScheduleResult, err error) {
    ......
    // 1.检查 pod pvc 
    if err := podPassesBasicChecks(pod, g.pvcLister); err != nil {
        return result, err
    }

    // 2.执行 "prefilter" plugins
    preFilterStatus := g.framework.RunPreFilterPlugins(pluginContext, pod)
    if !preFilterStatus.IsSuccess() {
        return result, preFilterStatus.AsError()
    }

    // 3.获取 node 数量
    numNodes := g.cache.NodeTree().NumNodes()
    if numNodes == 0 {
        return result, ErrNoNodesAvailable
    }

    // 4.快照 node 信息
    if err := g.snapshot(); err != nil {
        return result, err
    }
		
    // 5.执行预选算法
    startPredicateEvalTime := time.Now()
    filteredNodes, failedPredicateMap, filteredNodesStatuses, err := g.findNodesThatFit(pluginContext, pod)
    if err != nil {
        return result, err
    }
    // 6.执行 "postfilter" plugins
    postfilterStatus := g.framework.RunPostFilterPlugins(pluginContext, pod, filteredNodes, filteredNodesStatuses)
    if !postfilterStatus.IsSuccess() {
        return result, postfilterStatus.AsError()
    }

    // 7.预选后没有合适的 node 直接返回
    if len(filteredNodes) == 0 {
        ......
    }

    startPriorityEvalTime := time.Now()
    // 8.若只有一个 node 则直接返回该 node
    if len(filteredNodes) == 1 {
        return ScheduleResult{
            SuggestedHost:  filteredNodes[0].Name,
            EvaluatedNodes: 1 + len(failedPredicateMap),
            FeasibleNodes:  1,
        }, nil
    }
    
    // 9.获取 pod meta 信息，执行优选算法
    metaPrioritiesInterface := g.priorityMetaProducer(pod, g.nodeInfoSnapshot.NodeInfoMap)
    priorityList, err := PrioritizeNodes(pod, g.nodeInfoSnapshot.NodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders, g.framework,      pluginContext)
    if err != nil {
        return result, err
    }

    // 10.根据打分选择最佳的 node
    host, err := g.selectHost(priorityList)
    trace.Step("Selecting host done")
    return ScheduleResult{
        SuggestedHost:  host,
        EvaluatedNodes: len(filteredNodes) + len(failedPredicateMap),
        FeasibleNodes:  len(filteredNodes),
    }, err
}
```

至此，scheduler 的整个过程分析完毕。



### 总结

本文主要对于 kube-scheduler v1.16 的调度流程进行了分析，但其中有大量的细节都暂未提及，包括预选算法以及优选算法的具体实现、优先级与抢占调度的实现、framework 的使用及实现，因篇幅有限，部分内容会在后文继续说明。



参考：

[The Kubernetes Scheduler](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduler.md)

[scheduling design proposals](https://github.com/kubernetes/community/tree/master/contributors/design-proposals/scheduling)





