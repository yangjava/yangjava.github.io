---
layout: post
categories: [Kubernetes]
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

## kube-scheduler命令行参数详解
kube-scheduler的命令行参数较多，它与其他组件的flags参数类似，下面进行分类介绍。

kube-scheduler的命令行参数可分为如下几类。
- Misc flags ：其他参数。
- Secure serving flags ：HTTPS服务相关参数。
- Authentication flags ：认证相关参数。
- Authorization flags ：授权相关参数。
- Leader election flags ：多节点领导者选举相关参数。
- Feature gate flags ：实验性功能相关参数。

kube-scheduler拥有很多Deprecated flags（已被弃用）参数，故不再对这些参数进行赘述。

## kube-scheduler架构设计详解
kube-scheduler是Kubernetes的默认调度器，其架构设计本身并不复杂，但Kubernetes系统在后期引入了优先级和抢占机制及亲和性调度等功能，kube-scheduler调度器的整体设计略微复杂。这些内容会在后面详解。

kube-scheduler调度器在为Pod资源对象选择合适节点时，有如下两种最优解。
- 全局最优解： 是指每个调度周期都会遍历Kubernetes集群中的所有节点，以便找出全局最优的节点。
- 局部最优解： 是指每个调度周期只会遍历部分Kubernetes集群中的节点，找出局部最优的节点。

全局最优解和局部最优解可以解决调度器在小型和大型Kubernetes集群规模上的性能问题。目前kube-scheduler调度器对两种最优解都支持。当集群中只有几百台主机时，例如100台主机，kube-scheduler使用全局最优解。当集群规模较大时，例如其中包含5000多台主机，kube-scheduler使用局部最优解。

kube-scheduler组件的主要逻辑在于，如何在Kubernetes集群中为一个Pod资源对象找到合适的节点。调度器每次只调度一个Pod资源对象，为每一个Pod资源对象寻找合适节点的过程就是一个调度周期。

## kube-scheduler组件的启动流程
在kube-scheduler组件的启动流程中，分为以下步骤
- 内置调度算法的注册。
- Cobra命令行参数解析。
- 实例化Scheduler对象。
- 运行EventBroadcaster事件管理器。
- 运行HTTP或HTTPS服务。
- 运行Informer同步资源。
- 领导者选举实例化。
- 运行sched.Run调度器。

### 内置调度算法的注册
kube-scheduler组件启动后的第一件事情是，将Kubernetes内置的调度算法注册到调度算法注册表中。调度算法注册表与Scheme资源注册表类似，都是通过map数据结构存放的，调度算法注册表代码示例如下：

代码路径：pkg/scheduler/factory/plugins.go
```

```
调度算法分为两类，第一类是预选调度算法，第二类是优选调度算法。两类调度算法都存储在调度算法注册表中，该表由3个map数据结构构成，分别介绍如下。
- fitPredicateMap ：存储所有的预选调度算法。
- priorityFunctionMap ：存储所有的优选调度算法。
- algorithmProviderMap ：存储所有类型的调度算法。

内置调度算法的注册过程与kube-apiserver资源的注册过程类似，它们都通过Go语言的导入和初始化机制触发。当引用k8s.io/kubernetes/pkg/scheduler/algorithmprovider包时，就会自动调用包下的init初始化函数，代码示例如下：
```
```
registerAlgorithmProvider函数负责预选调度算法集（defaultPredicates）和优选调度算法集（defaultPriorities）的注册。通过factory.RegisterAlgorithmProvider将两类调度算法注册至algorithmProviderMap中。

### Cobra命令行参数解析
Cobra工具的功能强大，是Kubernetes系统中统一的命令行参数解析库，所有的Kubernetes组件都使用Cobra来解析命令行参数。更多关于Cobra命令行参数解析的内容，请参考4.2节“Cobra命令行参数解析”。

kube-scheduler组件通过Cobra填充Options配置参数默认值并验证参数，代码示例如下：

代码路径：cmd/kube-scheduler/app/server.go
```

```
首先kube-scheduler组件通过options.NewOptions函数初始化各个模块的默认配置，例如HTTP或HTTPS服务等。然后通过Validate函数验证配置参数的合法性和可用性，并通过Complete函数填充默认的options配置参数。

最后将cc（kube-scheduler组件的运行配置）对象传入Run函数，Run函数定义了kube-scheduler组件启动的逻辑，它是一个运行不退出的常驻进程。至此，完成了kube-scheduler组件启动之前的环境配置。

### 实例化Scheduler对象
Scheduler对象是运行kube-scheduler组件的主对象，它包含了kube-scheduler组件运行过程中的所有依赖模块对象。
Scheduler对象的实例化过程如下：
- 实例化所有的Informer；
- 实例化调度算法函数；
- 为所有Informer对象添加对资源事件的监控。

#### 实例化所有的Informer
代码路径：pkg/scheduler/scheduler.go
```

```
kube-scheduler组件依赖于多个资源的Informer对象，用于监控相应资源对象的事件。例如，通过PodInformer监控Pod资源对象，当某个Pod被创建时，kube-scheduler组件监控到该事件并为该Pod根据调度算法选择出合适的节点（Node）。

在Scheduler对象的实例化过程中，对NodeInformer、PodInformer、PersistentVolumeInformer、PersistentVolumeClaimInformer、ReplicationControllerInformer、ReplicaSetInformer、StatefulSetInformer、ServiceInformer、PodDisruptionBudgetInformer、StorageClassInformer资源通过Informer进行监控。

#### 实例化调度算法函数
在前面的章节中，内置调度算法的注册过程中只注册了调度算法的名称，在此处，为已经注册名称的调度算法实例化对应的调度算法函数，有两种方式实例化调度算法函数，它们被称为调度算法源（Scheduler Algorithm Source），代码示例如下：

代码路径：pkg/scheduler/apis/config/types.go
- Policy ：通过定义好的Policy（策略）资源的方式实例化调度算法函数。该方式可通过--policy-config-file参数指定调度策略文件。
- Provider ：通用调度器，通过名称的方式实例化调度算法函数，这也是kube-scheduler的默认方式。代码示例如下：


#### 为所有Informer对象添加对资源事件的监控
代码路径：pkg/scheduler/eventhandlers.go
```

```
AddAllEventHandlers函数为所有Informer对象添加对资源事件的监控并设置回调函数，以podInformer为例，代码示例如下：

代码路径：pkg/scheduler/eventhandlers.go
```

```
podInformer对象监控Pod资源对象，当该资源对象触发Add（添加）、Update （更新）、Delete（删除）事件时，触发对应的回调函数。例如，在触发Add事件后，podInformer将其放入SchedulingQueue调度队列中，等待kube-scheduler调度器为该Pod资源对象分配节点。

### 运行EventBroadcaster事件管理器
Kubernetes的事件（Event）是一种资源对象（Resource Object），用于展示集群内发生的情况，kube-scheduler组件会将运行时产生的各种事件上报给Kubernetes API Server。例如，调度器做了什么决定，为什么从节点中驱逐某些Pod资源对象等。可以通过kubectl get event或kubectl describe pod <podname>命令显示事件，用于查看Kubernetes集群中发生了哪些事件，这些命令只会显示最近（1小时内）发生的事件。更多关于EventBroadcaster事件管理器的内容，请参考5.5节“EventBroadcaster事件管理器”。代码示例如下：

代码路径：cmd/kube-scheduler/app/server.go
```

```
cc.Broadcaster通过StartLogging自定义函数将事件输出至klog stdout标准输出，通过StartRecordingToSink自定义函数将关键性事件上报给Kubernetes API Server。

### 运行HTTP或HTTPS服务
kube-scheduler组件也拥有自己的HTTP服务，但功能仅限于监控及监控检查等，其运行原理与kube-apiserver组件的类似，故不再赘述。kube-apiserver HTTP服务提供了如下几个重要接口。
- /healthz ：用于健康检查。
- /metrics ：用于监控指标，一般用于Prometheus指标采集。
- /debug/pprof ：用于pprof性能分析。

### 运行Informer同步资源
运行所有已经实例化的Informer对象，代码示例如下：

代码路径：cmd/kube-scheduler/app/server.go
```

```
通过Informer监控NodeInformer、PodInformer、PersistentVolumeInformer、PersistentVolumeClaimInformer、ReplicationControllerInformer、ReplicaSetInformer、StatefulSetInformer、ServiceInformer、PodDisruptionBudgetInformer、StorageClassInformer资源。

在正式启动Scheduler调度器之前，须通过cc.InformerFactory.WaitForCacheSync函数等待所有运行中的Informer的数据同步，使本地缓存数据与Etcd集群中的数据保持一致。

### 领导者选举实例化
领导者选举机制的目的是实现Kubernetes组件的高可用（High Availability）。在领导者选举实例化的过程中，会定义Callbacks函数，代码示例如下：

代码路径：cmd/kube-scheduler/app/server.go



LeaderCallbacks中定义了两个回调函数：OnStartedLeading函数是当前节点领导者选举成功后回调的函数，该函数定义了kube-scheduler组件的主逻辑；OnStoppedLeading函数是当前节点领导者被抢占后回调的函数，在领导者被抢占后，会退出当前的kube-scheduler进程。

通过leaderelection.NewLeaderElector函数实例化LeaderElector对象，通过leaderElector.Run函数参与领导者选举，该函数会一直尝试使节点成为领导者。

### 运行sched.Run调度器
在正式运行kube-scheduler组件的主逻辑之前，通过sched.config.WaitForCacheSync函数再次确认，所有运行中的Informer的数据是否已同步到本地，代码示例如下：

代码路径：pkg/scheduler/scheduler.go


sched.scheduleOne是kube-scheduler组件的调度主逻辑，它通过wait.Until定时器执行，内部会定时调用sched.scheduleOne函数，当sched.config.StopEverything Chan关闭时，该定时器才会停止并退出。


## 优先级与抢占机制
在当前Kubernetes系统中，Pod资源对象支持优先级（Priority）与抢占（Preempt）机制。当kube-scheduler调度器运行时，根据Pod资源对象的优先级进行调度，高优先级的Pod资源对象排在调度队列（SchedulingQueue）的前面，优先获得合适的节点（Node），然后为低优先级的Pod资源对象选择合适的节点。

当高优先级的Pod资源对象没有找到合适的节点时，调度器会尝试抢占低优先级的Pod资源对象的节点，抢占过程是将低优先级的Pod资源对象从所在的节点上驱逐走，使高优先级的Pod资源对象运行在该节点上，被驱逐走的低优先级的Pod资源对象会重新进入调度队列并等待再次选择合适的节点。

通过Pod资源对象的优先级可控制kube-scheduler的调度决策。在生产环境中，对不同业务进行分级，重要业务可拥有高优先级策略，以提升重要业务的可用性。

SchedulingQueue调度队列中拥有高优先级（HighPriority）、中优先级（MidPriority）、低优先级（LowPriority）3个Pod资源对象，它们等待被调度。调度队列中的Pod资源对象也被称为待调度Pod（Pending Pod）资源对象。

提示 ：当集群面对资源短缺的压力时，高优先级的Pod将依赖于调度程序抢占低优先级的Pod的方式进行调度，这样可以优先保证高优先级的业务运行，因此建议不要禁用抢占机制。

在默认的情况下，若不启用优先级功能，则现有Pod资源对象的优先级都为0，可以通过PriorityClass资源对象设置优先级，
PriorityClass Example代码示例如下：
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```
上述代码中定义了一个名为high-priority的PriorityClass，其value字段表示优先级，值越高则优先级越高；globalDefault字段表示此PriorityClass是否应用于没有PriorityClassName的Pod。需要注意的是，在Kubernetes系统中，只能存在一个将globalDefault字段设置为true的PriorityClass。

可通过kubectl get priorityclasses命令查看所有的PriorityClass资源对象。

为Pod资源对象设置优先级，在PodSpec中添加优先级对象名称，在 deployment、statefulset 或者 pod 中声明使用已有的 priorityClass 对象即可
在 pod 中使用：

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: nginx-a
  name: nginx-a
spec:
  containers:
  - image: nginx:1.7.9
    imagePullPolicy: IfNotPresent
    name: nginx-a
    ports:
    - containerPort: 80
      protocol: TCP
    resources:
      requests:
        memory: "64Mi"
        cpu: 5
      limits:
        memory: "128Mi"
        cpu: 5
  priorityClassName: high-priority
 ```

在 deployment 中使用：

```yaml
  template:
    spec:
      containers:
      - image: nginx
        name: nginx-deployment
        priorityClassName: high-priority
```

## 优先级与抢占机制源码
pod 优先级与抢占机制(Pod Priority and Preemption)，该功能是在 v1.8 中引入的，v1.11 中该功能为 beta 版本且默认启用了，v1.14 为 stable 版本。

正常情况下，当一个 pod 调度失败后，就会被暂时 “搁置” 处于 `pending` 状态，直到 pod 被更新或者集群状态发生变化，调度器才会对这个 pod 进行重新调度。

但在实际的业务场景中会存在在线与离线业务之分，若在线业务的 pod 因资源不足而调度失败时，此时就需要离线业务下掉一部分为在线业务提供资源，即在线业务要抢占离线业务的资源，此时就需要 scheduler 的优先级和抢占机制了，

该机制解决的是 pod 调度失败时该怎么办的问题，若该 pod 的优先级比较高此时并不会被”搁置”，而是会”挤走”某个 node 上的一些低优先级的 pod，这样就可以保证高优先级的 pod 调度成功。

### 优先级与抢占机制源码分析
抢占发生的原因，一定是一个高优先级的 pod 调度失败，我们称这个 pod 为“抢占者”，称被抢占的 pod 为“牺牲者”(victims)。而 kubernetes 调度器实现抢占算法的一个最重要的设计，就是在调度队列的实现里，使用了两个不同的队列。

- 第一个队列叫作 activeQ，凡是在 activeQ 里的 pod，都是下一个调度周期需要调度的对象。所以，当你在 kubernetes 集群里新创建一个 pod 的时候，调度器会将这个 pod 入队到 activeQ 里面，调度器不断从队列里出队(pop)一个 pod 进行调度，实际上都是从 activeQ 里出队的。
- 第二个队列叫作 unschedulableQ，专门用来存放调度失败的 pod，当一个 unschedulableQ 里的 pod 被更新之后，调度器会自动把这个 pod 移动到 activeQ 里，从而给这些调度失败的 pod “重新做人”的机会。

当 pod 拥有了优先级之后，高优先级的 pod 就可能会比低优先级的 pod 提前出队，从而尽早完成调度过程。

代码路径： k8s.io/kubernetes/pkg/scheduler/internal/queue/scheduling_queue.go
```
// NewSchedulingQueue initializes a priority queue as a new scheduling queue.
func NewSchedulingQueue(stop <-chan struct{}, fwk framework.Framework) SchedulingQueue {
    return NewPriorityQueue(stop, fwk)
}
// NewPriorityQueue creates a PriorityQueue object.
func NewPriorityQueue(stop <-chan struct{}, fwk framework.Framework) *PriorityQueue {
    return NewPriorityQueueWithClock(stop, util.RealClock{}, fwk)
}

// NewPriorityQueueWithClock creates a PriorityQueue which uses the passed clock for time.
func NewPriorityQueueWithClock(stop <-chan struct{}, clock util.Clock, fwk framework.Framework) *PriorityQueue {
    comp := activeQComp
    if fwk != nil {
        if queueSortFunc := fwk.QueueSortFunc(); queueSortFunc != nil {
            comp = func(podInfo1, podInfo2 interface{}) bool {
                pInfo1 := podInfo1.(*framework.PodInfo)
                pInfo2 := podInfo2.(*framework.PodInfo)

                return queueSortFunc(pInfo1, pInfo2)
            }
        }
    }

    pq := &PriorityQueue{
        clock:            clock,
        stop:             stop,
        podBackoff:       NewPodBackoffMap(1*time.Second, 10*time.Second),
        activeQ:          util.NewHeapWithRecorder(podInfoKeyFunc, comp, metrics.NewActivePodsRecorder()),
        unschedulableQ:   newUnschedulablePodsMap(metrics.NewUnschedulablePodsRecorder()),
        nominatedPods:    newNominatedPodMap(),
        moveRequestCycle: -1,
    }
    pq.cond.L = &pq.lock
    pq.podBackoffQ = util.NewHeapWithRecorder(podInfoKeyFunc, pq.podsCompareBackoffCompleted, metrics.NewBackoffPodsRecorder())

    pq.run()

    return pq
}
```
`scheduleOne()` 是执行调度算法的主逻辑，其主要功能有：
调用 `sched.schedule()`，即执行 predicates 算法和 priorities 算法。若执行失败，会返回 `core.FitError`。若开启了抢占机制，则执行抢占机制

代码路径： k8s.io/kubernetes/pkg/scheduler/scheduler.go
```
func (sched *Scheduler) scheduleOne() {
    ......
    scheduleResult, err := sched.schedule(pod, pluginContext)
    // predicates 算法和 priorities 算法执行失败
    if err != nil {
        if fitError, ok := err.(*core.FitError); ok {
            // 是否开启抢占机制
            if sched.DisablePreemption {
                .......
            } else {
            	// 执行抢占机制
                preemptionStartTime := time.Now()
                sched.preempt(pluginContext, fwk, pod, fitError)
                ......
            }
            ......
        } else {
            ......
        }
        return
    }
    ......
}
```
我们主要来看其中的抢占机制，`sched.preempt()` 是执行抢占机制的主逻辑，主要功能有：
- 从 apiserver 获取 pod info
- 调用 `sched.Algorithm.Preempt()`执行抢占逻辑，该函数会返回抢占成功的 node、被抢占的 pods(victims) 以及需要被移除已提名的 pods
- 更新 scheduler 缓存，为抢占者绑定 nodeName，即设定 pod.Status.NominatedNodeName
- 将 pod info 提交到 apiserver
- 删除被抢占的 pods
- 删除被抢占 pods 的 NominatedNodeName 字段

可以看到当上述抢占过程发生时，抢占者并不会立刻被调度到被抢占的 node 上，调度器只会将抢占者的 status.nominatedNodeName 字段设置为被抢占的 node 的名字。

然后，抢占者会重新进入下一个调度周期，在新的调度周期里来决定是不是要运行在被抢占的节点上，当然，即使在下一个调度周期，调度器也不会保证抢占者一定会运行在被抢占的节点上。

这样设计的一个重要原因是调度器只会通过标准的 DELETE API 来删除被抢占的 pod，所以，这些 pod 必然是有一定的“优雅退出”时间（默认是 30s）的。

而在这段时间里，其他的节点也是有可能变成可调度的，或者直接有新的节点被添加到这个集群中来。所以，鉴于优雅退出期间集群的可调度性可能会发生的变化，把抢占者交给下一个调度周期再处理，是一个非常合理的选择。

而在抢占者等待被调度的过程中，如果有其他更高优先级的 pod 也要抢占同一个节点，那么调度器就会清空原抢占者的 status.nominatedNodeName 字段，从而允许更高优先级的抢占者执行抢占，并且，这也使得原抢占者本身也有机会去重新抢占其他节点。以上这些都是设置 `nominatedNodeName` 字段的主要目的。

代码路径： k8s.io/kubernetes/pkg/scheduler/scheduler.go

```
func (sched *Scheduler) preempt(pluginContext *framework.PluginContext, fwk framework.Framework, preemptor *v1.Pod, scheduleErr error) (string, error) {
    // 获取 pod info
    preemptor, err := sched.PodPreemptor.GetUpdatedPod(preemptor)
    if err != nil {
        klog.Errorf("Error getting the updated preemptor pod object: %v", err)
        return "", err
    }

    // 执行抢占算法
    node, victims, nominatedPodsToClear, err := sched.Algorithm.Preempt(pluginContext, preemptor, scheduleErr)
    if err != nil {
        ......
    }
    var nodeName = ""
    if node != nil {
        nodeName = node.Name
        // 更新 scheduler 缓存，为抢占者绑定 nodename，即设定 pod.Status.NominatedNodeName
        sched.SchedulingQueue.UpdateNominatedPodForNode(preemptor, nodeName)

        // 将 pod info 提交到 apiserver
        err = sched.PodPreemptor.SetNominatedNodeName(preemptor, nodeName)
        if err != nil {
            sched.SchedulingQueue.DeleteNominatedPodIfExists(preemptor)
            return "", err
        }
        // 删除被抢占的 pods
        for _, victim := range victims {
            if err := sched.PodPreemptor.DeletePod(victim); err != nil {
                return "", err
            }
            ......
        }
    }

    // 删除被抢占 pods 的 NominatedNodeName 字段
    for _, p := range nominatedPodsToClear {
        rErr := sched.PodPreemptor.RemoveNominatedNodeName(p)
        if rErr != nil {
            ......
        }
    }
    return nodeName, err
}
```
`preempt()`中会调用 `sched.Algorithm.Preempt()`来执行实际抢占的算法，其主要功能有：
- 判断 err 是否为 `FitError`
- 调用`podEligibleToPreemptOthers()`确认 pod 是否有抢占其他 pod 的资格，若 pod 已经抢占了低优先级的 pod，被抢占的 pod 处于 terminating 状态中，则不会继续进行抢占
- 如果确定抢占可以发生，调度器会把自己缓存的所有节点信息复制一份，然后使用这个副本来模拟抢占过程
- 过滤预选失败的 node 列表，此处会检查 predicates 失败的原因，若存在 NodeSelectorNotMatch、PodNotMatchHostName 这些 error 则不能成为抢占者，如果过滤出的候选 node 为空则返回抢占者作为 nominatedPodsToClear
- 获取 `PodDisruptionBudget` 对象
- 从预选失败的 node 列表中并发计算可以被抢占的 nodes，得到 `nodeToVictims`
- 若声明了 extenders 则调用 extenders 再次过滤 `nodeToVictims`
- 调用  `pickOneNodeForPreemption()` 从 `nodeToVictims` 中选出一个节点作为最佳候选人
- 移除低优先级 pod 的 `Nominated`，更新这些 pod，移动到 activeQ 队列中，让调度器为这些 pod 重新 bind node

代码路径： k8s.io/kubernetes/pkg/scheduler/core/generic_scheduler.go
```
func (g *genericScheduler) Preempt(pluginContext *framework.PluginContext, pod *v1.Pod, scheduleErr error) (*v1.Node, []*v1.Pod, []*v1.Pod, error) {
    fitError, ok := scheduleErr.(*FitError)
    if !ok || fitError == nil {
        return nil, nil, nil, nil
    }
    // 判断 pod 是否支持抢占，若 pod 已经抢占了低优先级的 pod，被抢占的 pod 处于 terminating 状态中，则不会继续进行抢占
    if !podEligibleToPreemptOthers(pod, g.nodeInfoSnapshot.NodeInfoMap, g.enableNonPreempting) {
        return nil, nil, nil, nil
    }
    // 从缓存中获取 node list
    allNodes := g.cache.ListNodes()
    if len(allNodes) == 0 {
        return nil, nil, nil, ErrNoNodesAvailable
    }
    // 过滤 predicates 算法执行失败的 node 作为抢占的候选 node
    potentialNodes := nodesWherePreemptionMightHelp(allNodes, fitError)
    // 如果过滤出的候选 node 为空则返回抢占者作为 nominatedPodsToClear
    if len(potentialNodes) == 0 {
        return nil, nil, []*v1.Pod{pod}, nil
    }
    // 获取 PodDisruptionBudget objects
    pdbs, err := g.pdbLister.List(labels.Everything())
    if err != nil {
        return nil, nil, nil, err
    }
    // 过滤出可以抢占的 node 列表
    nodeToVictims, err := g.selectNodesForPreemption(pluginContext, pod, g.nodeInfoSnapshot.NodeInfoMap, potentialNodes, g.predicates,
        g.predicateMetaProducer, g.schedulingQueue, pdbs)
    if err != nil {
        return nil, nil, nil, err
    }

    // 若有 extender 则执行
    nodeToVictims, err = g.processPreemptionWithExtenders(pod, nodeToVictims)
    if err != nil {
        return nil, nil, nil, err
    }

    // 选出最佳的 node
    candidateNode := pickOneNodeForPreemption(nodeToVictims)
    if candidateNode == nil {
        return nil, nil, nil, nil
    }

    // 移除低优先级 pod 的 Nominated，更新这些 pod，移动到 activeQ 队列中，让调度器
    // 为这些 pod 重新 bind node
    nominatedPods := g.getLowerPriorityNominatedPods(pod, candidateNode.Name)
    if nodeInfo, ok := g.nodeInfoSnapshot.NodeInfoMap[candidateNode.Name]; ok {
        return nodeInfo.Node(), nodeToVictims[candidateNode].Pods, nominatedPods, nil
    }

    return nil, nil, nil, fmt.Errorf(
        "preemption failed: the target node %s has been deleted from scheduler cache",
        candidateNode.Name)
}
```
该函数中调用了多个函数：
- `nodesWherePreemptionMightHelp()`：过滤 predicates 算法执行失败的 node
- `selectNodesForPreemption()`：过滤出可以抢占的 node 列表
- `pickOneNodeForPreemption()`：选出最佳的 node
- `getLowerPriorityNominatedPods()`：移除低优先级 pod 的 Nominated


`selectNodesForPreemption()` 从 prediacates 算法执行失败的 node 列表中来寻找可以被抢占的 node，通过`workqueue.ParallelizeUntil()`并发执行`checkNode()`函数检查 node。

`k8s.io/kubernetes/pkg/scheduler/core/generic_scheduler.go:996`
```
func (g *genericScheduler) selectNodesForPreemption(
   ......
   ) (map[*v1.Node]*schedulerapi.Victims, error) {
    nodeToVictims := map[*v1.Node]*schedulerapi.Victims{}
    var resultLock sync.Mutex

    meta := metadataProducer(pod, nodeNameToInfo)
    // checkNode 函数
    checkNode := func(i int) {
        nodeName := potentialNodes[i].Name
        var metaCopy predicates.PredicateMetadata
        if meta != nil {
            metaCopy = meta.ShallowCopy()
        }
        // 调用 selectVictimsOnNode 函数进行检查
        pods, numPDBViolations, fits := g.selectVictimsOnNode(pluginContext, pod, metaCopy, nodeNameToInfo[nodeName], fitPredicates, queue, pdbs)
        if fits {
            resultLock.Lock()
            victims := schedulerapi.Victims{
                Pods:             pods,
                NumPDBViolations: numPDBViolations,
            }
            nodeToVictims[potentialNodes[i]] = &victims
            resultLock.Unlock()
        }
    }
    // 启动 16 个 goroutine 并发执行
    workqueue.ParallelizeUntil(context.TODO(), 16, len(potentialNodes), checkNode)
    return nodeToVictims, nil
}
```



其中调用的`selectVictimsOnNode()`是来获取每个 node 上 victims pod 的，首先移除所有低优先级的 pod 尝试抢占者是否可以调度成功，如果能够调度成功，然后基于 pod 是否有 PDB 被分为两组 `violatingVictims` 和 `nonViolatingVictims`，再对每一组的  pod 按优先级进行排序。PDB(pod 中断预算)是 kubernetes 保证副本高可用的一个对象。



然后开始逐一”删除“ pod 即要删掉最少的 pod 数来完成这次抢占即可，先从 `violatingVictims`(有PDB)的一组中进行”删除“ pod，并且记录删除有 PDB  pod 的数量，然后再“删除” `nonViolatingVictims` 组中的 pod，每次”删除“一个 pod 都要检查一下抢占者是否能够运行在该 node 上即执行一次预选策略，若执行预选策略失败则该 node 当前不满足抢占需要继续”删除“ pod 并将该 pod 加入到 victims 中，直到”删除“足够多的 pod 可以满足抢占，最后返回 victims 以及删除有 PDB  pod 的数量。

`k8s.io/kubernetes/pkg/scheduler/core/generic_scheduler.go:1086`
```
func (g *genericScheduler) selectVictimsOnNode(
		......
) ([]*v1.Pod, int, bool) {
    if nodeInfo == nil {
        return nil, 0, false
    }

    potentialVictims := util.SortableList{CompFunc: util.MoreImportantPod}
    nodeInfoCopy := nodeInfo.Clone()

    removePod := func(rp *v1.Pod) {
        nodeInfoCopy.RemovePod(rp)
        if meta != nil {
            meta.RemovePod(rp, nodeInfoCopy.Node())
        }
    }
    addPod := func(ap *v1.Pod) {
        nodeInfoCopy.AddPod(ap)
        if meta != nil {
            meta.AddPod(ap, nodeInfoCopy)
        }
    }
    // 先删除所有的低优先级 pod 检查是否能满足抢占 pod 的调度需求
    podPriority := util.GetPodPriority(pod)
    for _, p := range nodeInfoCopy.Pods() {
        if util.GetPodPriority(p) < podPriority {
            potentialVictims.Items = append(potentialVictims.Items, p)
            removePod(p)
        }
    }
    // 如果删除所有低优先级的 pod 不符合要求则直接过滤掉该 node
    // podFitsOnNode 就是前文讲过用来执行预选函数的
    if fits, _, _, err := g.podFitsOnNode(pluginContext, pod, meta, nodeInfoCopy, fitPredicates, queue, false); !fits {
        if err != nil {
						......
        }
        return nil, 0, false
    }
    var victims []*v1.Pod
    numViolatingVictim := 0
    potentialVictims.Sort()

    // 尝试尽量多地“删除”这些 pods，先从 PDB violating victims 中“删除”，再从 PDB non-violating victims 中“删除”
    violatingVictims, nonViolatingVictims := filterPodsWithPDBViolation(potentialVictims.Items, pdbs)

    // reprievePod 是“删除” pods 的函数
    reprievePod := func(p *v1.Pod) bool {
        addPod(p)
        // 同样也会调用 podFitsOnNode 再次执行 predicates 算法
        fits, _, _, _ := g.podFitsOnNode(pluginContext, pod, meta, nodeInfoCopy, fitPredicates, queue, false)
        if !fits {
            removePod(p)
            // 加入到 victims 中
            victims = append(victims, p)
        }
        return fits
    }
     // 删除 violatingVictims 中的 pod，同时也记录删除了多少个
    for _, p := range violatingVictims {
        if !reprievePod(p) {
            numViolatingVictim++
        }
    }
    // 删除 nonViolatingVictims 中的 pod
    for _, p := range nonViolatingVictims {
        reprievePod(p)
    }
    return victims, numViolatingVictim, true
}
```



`pickOneNodeForPreemption()` 用来选出最佳的 node 作为抢占者的 node，该函数主要基于 6 个原则：
- PDB violations 值最小的 node
- 挑选具有高优先级较少的 node
- 对每个 node 上所有 victims 的优先级进项累加，选取最小的
- 如果多个 node 优先级总和相等，选择具有最小 victims  数量的 node
- 如果多个 node 优先级总和相等，选择具有高优先级且 pod 运行时间最短的
- 如果依据以上策略仍然选出了多个 node 则直接返回第一个 node


`k8s.io/kubernetes/pkg/scheduler/core/generic_scheduler.go:867`
```
func pickOneNodeForPreemption(nodesToVictims map[*v1.Node]*schedulerapi.Victims) *v1.Node {
    if len(nodesToVictims) == 0 {
        return nil
    }
    minNumPDBViolatingPods := math.MaxInt32
    var minNodes1 []*v1.Node
    lenNodes1 := 0
    for node, victims := range nodesToVictims {
        if len(victims.Pods) == 0 {
	        // 若该 node 没有 victims 则返回
            return node
        }
        numPDBViolatingPods := victims.NumPDBViolations
        if numPDBViolatingPods < minNumPDBViolatingPods {
            minNumPDBViolatingPods = numPDBViolatingPods
            minNodes1 = nil
            lenNodes1 = 0
        }
        if numPDBViolatingPods == minNumPDBViolatingPods {
            minNodes1 = append(minNodes1, node)
            lenNodes1++
        }
    }
    if lenNodes1 == 1 {
        return minNodes1[0]
    }

    // 选出 PDB violating pods 数量最少的或者高优先级 victim 数量少的
    minHighestPriority := int32(math.MaxInt32)
    var minNodes2 = make([]*v1.Node, lenNodes1)
    lenNodes2 := 0
    for i := 0; i < lenNodes1; i++ {
        node := minNodes1[i]
        victims := nodesToVictims[node]
        highestPodPriority := util.GetPodPriority(victims.Pods[0])
        if highestPodPriority < minHighestPriority {
            minHighestPriority = highestPodPriority
            lenNodes2 = 0
        }
        if highestPodPriority == minHighestPriority {
            minNodes2[lenNodes2] = node
            lenNodes2++
        }
    }
    if lenNodes2 == 1 {
        return minNodes2[0]
    }
    // 若多个 node 高优先级的 pod 同样少，则选出加权得分最小的
    minSumPriorities := int64(math.MaxInt64)
    lenNodes1 = 0
    for i := 0; i < lenNodes2; i++ {
        var sumPriorities int64
        node := minNodes2[i]
        for _, pod := range nodesToVictims[node].Pods {
            sumPriorities += int64(util.GetPodPriority(pod)) + int64(math.MaxInt32+1)
        }
        if sumPriorities < minSumPriorities {
            minSumPriorities = sumPriorities
            lenNodes1 = 0
        }
        if sumPriorities == minSumPriorities {
            minNodes1[lenNodes1] = node
            lenNodes1++
        }
    }
    if lenNodes1 == 1 {
        return minNodes1[0]
    }
    // 若多个 node 高优先级的 pod 数量同等且加权分数相等，则选出 pod 数量最少的
    minNumPods := math.MaxInt32
    lenNodes2 = 0
    for i := 0; i < lenNodes1; i++ {
        node := minNodes1[i]
        numPods := len(nodesToVictims[node].Pods)
        if numPods < minNumPods {
            minNumPods = numPods
            lenNodes2 = 0
        }
        if numPods == minNumPods {
            minNodes2[lenNodes2] = node
            lenNodes2++
        }
    }
    if lenNodes2 == 1 {
        return minNodes2[0]
    }
    // 若多个 node 的 pod 数量相等，则选出高优先级 pod 启动时间最短的
    latestStartTime := util.GetEarliestPodStartTime(nodesToVictims[minNodes2[0]])
    if latestStartTime == nil {
        return minNodes2[0]
    }
    nodeToReturn := minNodes2[0]
    for i := 1; i < lenNodes2; i++ {
        node := minNodes2[i]
        earliestStartTimeOnNode := util.GetEarliestPodStartTime(nodesToVictims[node])
        if earliestStartTimeOnNode == nil {
            klog.Errorf("earliestStartTime is nil for node %s. Should not reach here.", node)
            continue
        }
        if earliestStartTimeOnNode.After(latestStartTime.Time) {
            latestStartTime = earliestStartTimeOnNode
            nodeToReturn = node
        }
    }

    return nodeToReturn
}
```

以上就是对抢占机制代码的一个通读。











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





