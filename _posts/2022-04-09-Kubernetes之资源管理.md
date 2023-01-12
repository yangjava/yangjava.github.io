---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes Limits 和 Requests
      管理 Kubernetes 集群就像坐镇在大型战场前，指挥千军万马进行火拼。在我们面前有几乎压倒性的旋律和氛围来将斗士气势营造至完美境地，甚至知道从哪里开始似乎都具有挑战性，作为经验丰富的架构师，往往可能很清楚这种感觉。
     然而，一个没有资源限制的 Kubernetes 集群可能会导致诸多的不可见问题。因此，设置资源限制便是一个合乎逻辑的起点。

     话虽如此，然而，真正的挑战才刚刚开始。要正确设置 Kubernetes 资源限制，我们必须有条不紊并注意找到正确的值。将它们设置得太高，可能会对集群的节点产生负面影响。将该值设置得太低，会对应用程序性能产生负面影响。

     因此，有效地设置 Kubernetes 请求和限制对应用程序的性能、稳定性和成本有重大影响。

Resource Requests && Limits 即“资源请求和限制”是在容器级别指定的可选参数。Kubernetes 将 Pod 的请求和限制计算为其所有容器的请求和限制的总和。然后 Kubernetes 使用这些参数进行调度和资源分配决策。

     那么，什么是资源限制，它们为什么对我们来说如此重要？

     首先，在 Kubernetes 生态体系中，资源限制与资源请求往往以成对形式展现在大众面前：

     1、资源请求：分配给容器的 CPU 或内存量。Pod 资源请求等于其容器资源请求的总和。在调度 Pod 时，Kubernetes 将保证此数量的资源可供所支撑的 Pod 运行。

     2、资源限制：Kubernetes 将开始对超出限制的容器采取行动的级别。Kubernetes 会杀死一个消耗过多内存的容器或限制一个使用过多 CPU 的容器。

     如果设置了资源限制但没有资源请求，Kubernetes 会隐式设置内存和 CPU 请求等于限制。这种行为非常适合作为控制 Kubernetes 集群的第一步。这通常被称为保守方法，其中分配给容器的资源最多。


Resource Limits && Requests 解析

     Requests-请求

     Pod 将获得它们请求的内存量。如果他们超出了他们的内存请求，如果另一个 Pod 碰巧需要这个内存，他们可能会被杀死。只有当关键系统或高优先级工作负载需要内存时，Pod 才会在使用的内存少于请求的内存时被杀死。

     同样，Pod 中的每个容器都分配了它请求的 CPU 量（如果可用）。如果其他正在运行的 Pod/Jobs 不需要可用资源，它可能会被分配额外的 CPU 周期。因此，请求定义了容器需要的最小资源量。

     注意：如果 Pod 的总请求在单个节点上不可用，则 Pod 将保持在 Pending 状态（即未运行），直到这些资源可用。

     Limits-限制

     资源限制有助于 Kubernetes 调度程序更好地处理资源争用。当 Pod 使用的内存超过其限制时，其进程将被内核杀死以保护集群中的其他应用程序。当 Pod 超过其 CPU 限制时，它们将受到 CPU 限制。如果未设置限制，则 Pod 可以在可用时使用多余的内存和 CPU。限制决定了容器可以使用的最大资源量，防止资源短缺或由于资源消耗过多而导致机器崩溃。如果设置为 0，则表示容器没有资源限制。特别是如果你设置了 limits 而不指定 requests，Kubernetes 默认认为 requests 的值和 limits的值是一样的。

     Kubernetes 请求和限制适用于两种类型的资源 - 可压缩（例如 CPU）和不可压缩（例如内存）。对于不可压缩资源，适当的限制非常重要。

     为了充分利用 Kubernetes 集群中的资源，提高调度效率，Kubernetes 使用请求和限制来控制容器的资源分配。每个容器都可以有自己的请求和限制。这两个参数由 resources.requests 和 resources.limits 指定。一般来说，Requests-请求在调度中更重要，而 Limits-限制在运行中更重要。

[administrator@JavaLangOutOfMemory ~ ]% less demo-service-ap.yaml
apiVersion: extensions/v1beta1
metadata:
name: redis
labels:
name: redis-deployment
app: demo-service-ap
spec:
replicas: 3
selector:
matchLabels:
name: redis
role: cachedb
app: demo-service-ap
template:
spec:
containers:
- name: redis
image: redis:6.2.6-alpine
resources:
limits:
memory: 600Mi
cpu: 1
requests:
memory: 300Mi
cpu: 500m
- name: busybox
image: busybox:1.31.1
resources:
limits:
memory: 200Mi
cpu: 300m
requests:
memory: 100Mi
cpu: 100m
复制
基于上述 Yaml 文件，在 Kubernetes 中，CPU 不是以百分比分配的，而是以千计（也称为 millicores 或 millicpu）。一个 CPU 等于 1000 毫核。如果希望分配三分之一的 CPU，我们应该为容器分配 333 Mi（毫核）。

     相对于 CPU ，内存往往更简单一些，其主要以字节为单位。

     Kubernetes 接受 SI 表示法 (K,M,G,T,P,E) 和二进制表示法 (Ki,Mi,Gi,Ti,Pi,Ei) 来定义内存。例如，要将内存限制在 256 MB，我们可以分配 268.4 M（SI 表示法）或 256 Mi（二进制表示法）。

     基于上述示例，我们可以看到针对 Resources 的定义涉及四个部分。其实，在实际的业务场景中，这些配置项中的每一个都是可选的。然而，在生产环境中，还是建议大家进行合理设置，以满足应用运行性能要求。

     requests.cpu 是命名空间中所有容器的最大组合 CPU 请求（以毫秒为单位）。在上面的例子中，你可以有 50 个 10m 请求的容器，5 个 100m 请求的容器，甚至一个 500m 请求的容器。只要在 Namespace 中请求的 CPU 总量小于 500m！

     requests.memory 是命名空间中所有容器的最大组合内存请求。在上面的示例参数中，可以拥有 50 个具有 2MiB 请求的容器、5 个具有 20MiB CPU 请求的容器，甚至是一个具有 100MiB 请求的容器。只要命名空间中请求的总内存小于 100MiB！

     limits.cpu 是命名空间中所有容器的最大组合 CPU 限制。它就像 requests.cpu 一样，但有限制。

     limits.memory 是命名空间中所有容器的最大组合内存限制。它就像 requests.memory 但有限制。

     请求和限制在实际的业务场景至关重要，因为它们在 Kubernetes 如何决定在需要释放资源时杀死哪些 Pod 中发挥着重要作用：

     1、没有限制或请求集的 Pod

     2、没有设置限制的 Pod

     3、超过内存请求但低于限制的 Pod

     4、Pod 使用的内存少于请求的内存

常见资源异常

     在实际的业务场景中，是否对所有容器设置了请求和限制？如果没有，Kubernetes 调度程序将随机分配任何没有请求和限制的 Pod。设置限制后，将避免以下大多数问题：

     1、内存不足 (OOM) 问题：节点可能死于内存不足，影响集群稳定性。例如，具有内存泄漏的应用程序可能会导致 OOM 问题。通常，在 Kubernetes 中看到两种主要的 OOMKilled 错误：OOMKilled：限制过度使用和 OOMKilled：已达到容器限制。

OOMKilled: Limit Overcommit - 限制过度使用

    当 Pod 限制的总和大于节点上的可用内存时，可能会发生 OOMKilled: Limit Overcommit 错误。例如，如果我们有一个具有 8 GB 可用内存节点，可能在前期的糟糕规划设计中会分配 8 个 Pod，每个 Pod 都需要 1 gig 内存。但是，即使其中一个 Pod 配置了 1.5 gigs 的限制，我们也会面临内存不足的风险。只需要一个 Pod 出现流量峰值或未知的内存泄漏，Kubernetes 将被迫开始杀死 Pod。

     此时，我们需要检查主机本身，看看是否有任何在 Kubernetes 之外运行的进程可能会占用内存，从而为 Pod 留下更少的内存。

OOMKilled: Container Limit Reached - 达到容器限制

     虽然 Limit Overcommit 错误与节点上的内存总量有关，但 Container Limit Reached 通常归结为单个 Pod。当 Kubernetes 检测到一个 Pod 使用的内存超过了设置的限制时，它会杀死该 Pod，并显示错误 OOMKilled—Container Limit Reached。

     发生这种情况时，请检查应用程序日志以尝试了解 Pod 使用的内存超过设置限制的原因。可能有多种原因，例如流量激增或长时间运行的 Kubernetes 作业导致它使用比平时更多的内存。

     如果在调查期间发现应用程序按预期运行并且它只需要更多内存来运行，则可能会考虑增加 request 和 limit 的值。

     2、CPU 饥饿：应用程序会变慢，因为它们必须共享有限数量的 CPU。消耗过多 CPU 的应用程序可能会影响同一节点上的所有应用程序。

     3、Pod 驱逐：当一个节点缺乏资源时，它会启动驱逐过程并终止 Pod，从没有资源请求的 Pod 开始。

     4、资源浪费：假设我们的集群在没有请求和限制的情况下运行状态很好，这意味着很可能过度配置。换句话说，你把钱花在了你从未使用过的资源上。

     因此，防止上述问题在业务运行中的发生，第一步，则是为所有容器设置资源限制操作。


源码解析

     在上述的解析中，我们已经了解了一些配置请求和限制的最佳实践，让我们更深入地研究源代码。

     下面的代码展示了 Requests and Scheduling （请求与调度）中 Pod 的请求和 Pod 中容器的请求之间的关系。具体如下所示：

func computePodResourceRequest(pod *v1.Pod) *preFilterState {
result := &preFilterState{}
for _, container := range pod.Spec.Containers {
result.Add(container.Resources.Requests)
}

// take max_resource(sum_pod, any_init_container)
for _, container := range pod.Spec.InitContainers {
result.SetMaxResource(container.Resources.Requests)
}

// If Overhead is being utilized, add to the total requests for the pod
if pod.Spec.Overhead != nil && utilfeature.DefaultFeatureGate.Enabled(features.PodOverhead) {
result.Add(pod.Spec.Overhead)
}

return result
}
...
func (f *Fit) PreFilter(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod) *framework.Status {
cycleState.Write(preFilterStateKey, computePodResourceRequest(pod))
return nil
}
...
func getPreFilterState(cycleState *framework.CycleState) (*preFilterState, error) {
c, err := cycleState.Read(preFilterStateKey)
if err != nil {
// preFilterState doesn't exist, likely PreFilter wasn't invoked.
return nil, fmt.Errorf("error reading %q from cycleState: %v", preFilterStateKey, err)
}

s, ok := c.(*preFilterState)
if !ok {
return nil, fmt.Errorf("%+v  convert to NodeResourcesFit.preFilterState error", c)
}
return s, nil
}
...
func (f *Fit) Filter(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
s, err := getPreFilterState(cycleState)
if err != nil {
return framework.NewStatus(framework.Error, err.Error())
}

insufficientResources := fitsRequest(s, nodeInfo, f.ignoredResources, f.ignoredResourceGroups)

if len(insufficientResources) != 0 {
// We will keep all failure reasons.
failureReasons := make([]string, 0, len(insufficientResources))
for _, r := range insufficientResources {
failureReasons = append(failureReasons, r.Reason)
}
return framework.NewStatus(framework.Unschedulable, failureReasons...)
}
return nil
}
复制
基于上述代码定义，我们可以看到：调度器（调度线程）计算了待调度的 Pod 所需的资源。具体来说，它根据 Pod 规范分别计算 init 容器的总请求数和工作容器的总请求数。在接下来的 Filter 阶段，会检查所有节点是否满足条件。

     备注：对于轻量级虚拟机（例如 kata-container），它们自己的虚拟化资源消耗需要计入缓存中。 

     通常而言，调度过程需要不同的阶段，包括前置过滤器、过滤器、后置过滤器和评分。有关详细信息，可参阅过“滤器和评分节点”内容。具体可参考如下地址所示：

https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/#kube-scheduler-implementation

     过滤后，如果只有一个适用的节点，则将 Pod 调度到该节点上。如果有多个适用的 Pod，调度器会选择加权分数总和最高的节点。评分基于多种因素，因为调度插件实现了一个或多个扩展点。请注意，requests 的值和 limits 的值直接影响插件 NodeResourcesLeastAllocated 的最终结果。 其源代码实现如下所示：

func leastResourceScorer(resToWeightMap resourceToWeightMap) func(resourceToValueMap, resourceToValueMap, bool, int, int) int64 {
return func(requested, allocable resourceToValueMap, includeVolumes bool, requestedVolumes int, allocatableVolumes int) int64 {
var nodeScore, weightSum int64
for resource, weight := range resToWeightMap {
resourceScore := leastRequestedScore(requested[resource], allocable[resource])
nodeScore += resourceScore * weight
weightSum += weight
}
return nodeScore / weightSum
}
}
...
func leastRequestedScore(requested, capacity int64) int64 {
if capacity == 0 {
return 0
}
if requested > capacity {
return 0
}

return ((capacity - requested) * int64(framework.MaxNodeScore)) / capacity
}
复制
对于 NodeResourcesLeastAllocated，如果同一 Pod 拥有更多资源，则节点将获得更高的分数。 换句话说，一个 Pod 将更有可能被调度到资源充足的节点上。

     在创建 Pod 时，Kubernetes 需要分配不同的资源，包括 CPU 和内存。每种资源都有一个权重（源代码中的 resToWeightMap 结构）。作为一个整体，它们告诉 Kubernetes 调度程序什么是实现资源平衡的最佳决策。在 Score 阶段，调度器除了 

NodeResourcesLeastAllocated 外，还使用其他插件（InterPodAffinity）进行评分。

     接下来，我们顺带了解一下 QoS and Scheduling （QoS 与调度），QoS 作为Kubernetes 中的一种资源保护机制，主要用于控制内存等不可压缩资源。 它还会影响不同 Pod 和容器的 OOM 分数。 当节点内存不足时，内核（OOM Killer）会杀死低优先级的 Pod（分数越高，优先级越低）。其源代码实现如下所示：

func GetContainerOOMScoreAdjust(pod *v1.Pod, container *v1.Container, memoryCapacity int64) int {
if types.IsCriticalPod(pod) {
// Critical pods should be the last to get killed.
return guaranteedOOMScoreAdj
}

switch v1qos.GetPodQOS(pod) {
case v1.PodQOSGuaranteed:
// Guaranteed containers should be the last to get killed.
return guaranteedOOMScoreAdj
case v1.PodQOSBestEffort:
return besteffortOOMScoreAdj
}

// Burstable containers are a middle tier, between Guaranteed and Best-Effort. Ideally,
// we want to protect Burstable containers that consume less memory than requested.
// The formula below is a heuristic. A container requesting for 10% of a system's
// memory will have an OOM score adjust of 900. If a process in container Y
// uses over 10% of memory, its OOM score will be 1000. The idea is that containers
// which use more than their request will have an OOM score of 1000 and will be prime
// targets for OOM kills.
// Note that this is a heuristic, it won't work if a container has many small processes.
memoryRequest := container.Resources.Requests.Memory().Value()
oomScoreAdjust := 1000 - (1000*memoryRequest)/memoryCapacity
// A guaranteed pod using 100% of memory can have an OOM score of 10. Ensure
// that burstable pods have a higher OOM score adjustment.
if int(oomScoreAdjust) < (1000 + guaranteedOOMScoreAdj) {
return (1000 + guaranteedOOMScoreAdj)
}
// Give burstable pods a higher chance of survival over besteffort pods.
if int(oomScoreAdjust) == besteffortOOMScoreAdj {
return int(oomScoreAdjust - 1)
}
return int(oomScoreAdjust)
}
复制
资源管控

     要仔细分析指标，我们需要一步一步来。 通常，可概括为2个阶段组成，每个阶段都会导致不同的策略。从最激进的开始，挑战结果，并在必要时转向更保守的选择。


     通常，我们必须独立考虑 CPU 和内存，并根据每个阶段的结论应用不同的策略。

1、观测内存或 CPU

     在第一阶段，查看内存或 CPU 的百分之九十九。这种激进的方法旨在通过强制 Kubernetes 对异常值采取行动来减少问题。如果使用此值设置限制，我们的应用程序将有 1% 的时间受到影响。容器 CPU 将受到限制，并且永远不会再次达到该值。

激进的方法通常有利于 CPU 限制，因为其后果相对可以接受，并且可以帮助我们更好地管理资源。关于记忆，百分之九十九可能有问题；如果达到限制，容器将重新启动。


     此时，我们应该权衡后果并得出结论，如果 99 个百分位对有意义（作为旁注，我们应该更深入地调查为什么应用程序有时会达到设定的限制）。可能 99 个百分位数过于严格，因为我们的应用程序尚未达到最大利用率。在这种情况下，请继续使用第二个策略来设置限制。

     2、实时优化调整

     最后阶段是通过添加或减去一个系数（即，最大值 + 20%）来找到基于最大值的折衷方案。如果达到这一点，我们应该考虑执行负载测试以更好地描述我们的应用程序性能和资源使用情况。无限制地对每个应用程序重复此过程。

     以上为 Kubernetes 中资源限制与资源请求相关简要解析，更多内容欢迎大家深入沟通、探讨！

# 参考资料

https://medium.com/omio-engineering/cpu-limits-and-aggressive-throttling-in-kubernetes-c5b20bd8a718
https://learnk8s.io/setting-cpu-memory-limits-requests


