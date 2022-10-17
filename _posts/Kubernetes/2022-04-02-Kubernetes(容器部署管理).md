---
layout: post
title: Kubernetes简介
categories: Kubernetes
description: none
keywords: Kubernetes,K8S
---

# Kubernetes
Kubernetes是用于自动部署，扩展和管理容器化应用程序的开源系统。

**Kubernetes, also known as K8s, is an open-source system for automating deployment, scaling, and management of containerized applications.**

官网：Kubernetes是用于自动部署，扩展和管理容器化应用程序的开源系统。

## 简介

Kubernetes 是一个可移植的、可扩展的开源平台，用于管理容器化的工作负载和服务，可促进声明式配置和自动化。 Kubernetes 拥有一个庞大且快速增长的生态系统。Kubernetes 的服务、支持和工具广泛可用。

Kubernetes 这个名字源于希腊语，意为“舵手”或“飞行员”。k8s 这个缩写是因为 k 和 s 之间有八个字符的关系。 Google 在 2014 年开源了 Kubernetes 项目。Kubernetes 建立在 Google 在大规模运行生产工作负载方面拥有十几年的经验 的基础上，结合了社区中最好的想法和实践。

### Kubernetes 是什么

Kubernetes 是一个全新的基于容器技术的分布式架构解决方案，是 Google 开源的一个容器集群管理系统，Kubernetes 简称 K8S。

Kubernetes 是一个一站式的完备的分布式系统开发和支撑平台，更是一个开放平台，对现有的编程语言、编程框架、中间件没有任何侵入性。

Kubernetes 提供了完善的管理工具，这些工具涵盖了开发、部署测试、运维监控在内的各个环节。

Kubernetes 具有完备的集群管理能力，包括多层次的安全防护和准入机制、多租户应用支撑能力、透明的服务注册和服务发现机制、内建智能负载均衡器、强大的故障发现和自我修复能力、服务滚动升级和在线扩容能力、可扩展的资源自动调度机制、多粒度的资源配额管理能力。

Kubernetes 官方文档：https://kubernetes.io/zh/

### Kubernetes 特性

① 自我修复

在节点故障时，重新启动失败的容器，替换和重新部署，保证预期的副本数量；杀死健康检查失败的容器，并且在未准备好之前不会处理用户的请求，确保线上服务不中断。

② 弹性伸缩

使用命令、UI或者基于CPU使用情况自动快速扩容和缩容应用程序实例，保证应用业务高峰并发时的高可用性；业务低峰时回收资源，以最小成本运行服务。

③ 自动部署和回滚

K8S采用滚动更新策略更新应用，一次更新一个Pod，而不是同时删除所有Pod，如果更新过程中出现问题，将回滚更改，确保升级不影响业务。

④ 服务发现和负载均衡

K8S为多个容器提供一个统一访问入口（内部IP地址和一个DNS名称），并且负载均衡关联的所有容器，使得用户无需考虑容器IP问题。

⑤ 机密和配置管理

管理机密数据和应用程序配置，而不需要把敏感数据暴露在镜像里，提高敏感数据安全性。并可以将一些常用的配置存储在K8S中，方便应用程序使用。

⑥ 存储编排

挂载外部存储系统，无论是来自本地存储，公有云，还是网络存储，都作为集群资源的一部分使用，极大提高存储使用灵活性。

⑦ 批处理

提供一次性任务，定时任务；满足批量数据处理和分析的场景。

## 集群架构与组件

Kubernetes 集群架构以及相关的核心组件如下图所示：一个 Kubernetes 集群一般包含一个 Master 节点和多个 Node 节点，一个节点可以看成是一台物理机或虚拟机。

![img](png/Kubernetes/Kubernetes.png)

### Master

Master 是 K8S 的集群控制节点，每个 K8S 集群里需要有一个 Master 节点来负责整个集群的管理和控制，基本上 K8S 所有的控制命令都是发给它，它来负责具体的执行过程。Master 节点通常会占据一个独立的服务器，因为它太重要了，如果它不可用，那么所有的控制命令都将失效。

Master 节点上运行着以下关键组件：

##### ① **kube-apiserver**

是集群的统一入口，各组件协调者，以 HTTP Rest 提供接口服务，所有对象资源的增、删、改、查和监听操作都交给 apiserver 处理后再提交给 Etcd 存储。

##### ② **kube-controller-manager**

是 K8S 里所有资源对象的自动化控制中心，处理集群中常规后台任务，一个资源对应一个控制器，而 controller-manager 就是负责管理这些控制器的。

##### ③ **kube-scheduler**

根据调度算法为新创建的 Pod 选择一个 Node 节点，可以任意部署，可以部署在同一个节点上，也可以部署在不同的节点上。

##### ④ **etcd**

是一个分布式的，一致的 key-value 存储，主要用途是共享配置和服务发现，保存集群状态数据，比如 Pod、Service 等对象信息。

### Node

除了 Master，K8S 集群中的其它机器被称为 Node 节点，Node 节点是 K8S 集群中的工作负载节点，每个 Node 都会被 Master 分配一些工作负载，当某个 Node 宕机时，其上的工作负载会被 Master 自动转移到其它节点上去。

每个 Node 节点上都运行着以下关键组件：

##### ① **kubelet**

kubelet 是 Master 在 Node 节点上的 Agent(代理)，与 Master 密切协作，管理本机运行容器的生命周期，负责 Pod 对应的容器的创建、启停等任务，实现集群管理的基本功能。

##### ② **kube-proxy**

在 Node 节点上实现 Pod 网络代理，实现 Kubernetes Service 的通信，维护网络规则和四层负载均衡工作。

##### ③ **docker engine**

Docker 引擎，负责本机的容器创建和管理工作。

Node 节点可以在运行期间动态增加到 K8S 集群中，前提是这个节点上已经正确安装、配置和启动了上述关键组件。在默认情况下 kubelet 会向 Master 注册自己，一旦 Node 被纳入集群管理范围，kubelet 就会定时向 Master 节点汇报自身的情况，例如操作系统、Docker 版本、机器的 CPU 和内存情况，以及之前有哪些 Pod 在运行等，这样 Master 可以获知每个 Node 的资源使用情况，并实现高效均衡的资源调度策略。而某个 Node 超过指定时间不上报信息时，会被 Master 判定为“失联”，Node 的状态被标记为不可用（Not Ready），随后 Master 会触发“工作负载大转移”的自动流程。

## 核心概念

#### **Pod**

Pod 是 K8S 中最重要也是最基本的概念，Pod 是最小的部署单元，是一组容器的集合。每个 Pod 都由一个特殊的根容器 Pause 容器，以及一个或多个紧密相关的用户业务容器组成。

Pause 容器作为 Pod 的根容器，以它的状态代表整个容器组的状态。K8S 为每个 Pod 都分配了唯一的 IP 地址，称之为 Pod IP。Pod 里的多个业务容器共享 Pause 容器的IP，共享 Pause 容器挂载的 Volume。

Pod是一组紧密关联的容器集合，支持多个容器在一个Pod中共享网络和文件系统，可以通过进程间通信和文件共享这种简单高效的方式完成服务，是Kubernetes调度的基本单位。Pod的设计理念是每个Pod都有一个唯一的IP。

Pod具有如下特征：

- 包含多个共享IPC、Network和UTC namespace的容器，可直接通过localhost通信
- 所有Pod内容器都可以访问共享的Volume，可以访问共享数据
- 优雅终止:Pod删除的时候先给其内的进程发送SIGTERM，等待一段时间(grace period)后才强制停止依然还在运行的进程
- 特权容器(通过SecurityContext配置)具有改变系统配置的权限(在网络插件中大量应用)
- 支持三种重启策略（restartPolicy），分别是：Always、OnFailure、Never
- 支持三种镜像拉取策略（imagePullPolicy），分别是：Always、Never、IfNotPresent
- 资源限制，Kubernetes通过CGroup限制容器的CPU以及内存等资源，可以设置request以及limit值
- 健康检查，提供两种健康检查探针，分别是livenessProbe和redinessProbe，前者用于探测容器是否存活，如果探测失败，则根据重启策略进行重启操作，后者用于检查容器状态是否正常，如果检查容器状态不正常，则请求不会到达该Pod
- Init container在所有容器运行之前执行，常用来初始化配置
- 容器生命周期钩子函数，用于监听容器生命周期的特定事件，并在事件发生时执行已注册的回调函数，支持两种钩子函数：postStart和preStop，前者是在容器启动后执行，后者是在容器停止前执行

#### **Namespace**

命名空间，Namespace 多用于实现多租户的资源隔离。Namespace 通过将集群内部的资源对象“分配”到不同的Namespace中，形成逻辑上分组的不同项目、小组或用户组。

K8S 集群在启动后，会创建一个名为 default 的 Namespace，如果不特别指明 Namespace，创建的 Pod、RC、Service 都将被创建到 default 下。

当我们给每个租户创建一个 Namespace 来实现多租户的资源隔离时，还可以结合 K8S 的资源配额管理，限定不同租户能占用的资源，例如 CPU 使用量、内存使用量等。

Namespace（命名空间）是对一组资源和对象的抽象集合，比如可以用来将系统内部的对象划分为不同的项目组或者用户组。常见的pod、service、replicaSet和deployment等都是属于某一个namespace的(默认是default)，而node, persistentVolumes等则不属于任何namespace。

常用namespace操作：

- kubectl get namespace, 查询所有namespace
- kubectl create  namespace ns-name，创建namespace
- kubectl  delete  namespace  ns-name, 删除namespace

删除命名空间时，需注意以下几点：

1. 删除一个namespace会自动删除所有属于该namespace的资源。
2. default 和 kube-system 命名空间不可删除。
3. PersistentVolumes是不属于任何namespace的，但PersistentVolumeClaim是属于某个特定namespace的。
4. Events是否属于namespace取决于产生events的对象。

#### Node

Node是Pod真正运行的主机，可以是物理机也可以是虚拟机。Node本质上不是Kubernetes来创建的， Kubernetes只是管理Node上的资源。为了管理Pod，每个Node节点上至少需要运行container runtime（Docker）、kubelet和kube-proxy服务。

常用node操作：

- kubectl   get  nodes，查询所有node
- kubectl cordon $nodename, 将node标志为不可调度
- kubectl uncordon $nodename, 将node标志为可调度

**taint(污点)**

使用kubectl taint命令可以给某个Node节点设置污点，Node被设置上污点之后就和Pod之间存在了一种相斥的关系，可以让Node拒绝Pod的调度执行，甚至将Node已经存在的Pod驱逐出去。每个污点的组成：key=value:effect，当前taint effect支持如下三个选项：

- NoSchedule：表示k8s将不会将Pod调度到具有该污点的Node上
- PreferNoSchedule：表示k8s将尽量避免将Pod调度到具有该污点的Node上
- NoExecute：表示k8s将不会将Pod调度到具有该污点的Node上，同时会将Node上已经存在的Pod驱逐出去

常用命令如下：

- kubectl taint node node0 key1=value1:NoShedule，为node0设置不可调度污点
- kubectl taint node node0 key-，将node0上key值为key1的污点移除
- kubectl taint node node1 node-role.kubernetes.io/master=:NoSchedule，为kube-master节点设置不可调度污点
- kubectl taint node node1 node-role.kubernetes.io/master=PreferNoSchedule，为kube-master节点设置尽量不可调度污点

**容忍(Tolerations)**

设置了污点的Node将根据taint的effect：NoSchedule、PreferNoSchedule、NoExecute和Pod之间产生互斥的关系，Pod将在一定程度上不会被调度到Node上。 但我们可以在Pod上设置容忍(Toleration)，意思是设置了容忍的Pod将可以容忍污点的存在，可以被调度到存在污点的Node上。



#### **Service**

Service 定义了一个服务的访问入口，通过 Label Selector 与 Pod 副本集群之间“无缝对接”，定义了一组 Pod 的访问策略，防止 Pod 失联。

创建 Service 时，K8S会自动为它分配一个全局唯一的虚拟 IP 地址，即 Cluster IP。服务发现就是通过 Service 的 Name 和 Service 的 ClusterIP 地址做一个 DNS 域名映射来解决的。

Service是对一组提供相同功能的Pods的抽象，并为他们提供一个统一的入口，借助 Service 应用可以方便的实现服务发现与负载均衡，并实现应用的零宕机升级。Service通过标签(label)来选取后端Pod，一般配合ReplicaSet或者Deployment来保证后端容器的正常运行。

service 有如下四种类型，默认是ClusterIP：

- ClusterIP: 默认类型，自动分配一个仅集群内部可以访问的虚拟IP
- NodePort: 在ClusterIP基础上为Service在每台机器上绑定一个端口，这样就可以通过 NodeIP:NodePort 来访问该服务
- LoadBalancer: 在NodePort的基础上，借助cloud provider创建一个外部的负载均衡器，并将请求转发到 NodeIP:NodePort
- ExternalName: 将服务通过DNS CNAME记录方式转发到指定的域名

另外，也可以将已有的服务以Service的形式加入到Kubernetes集群中来，只需要在创建 Service 的时候不指定Label selector，而是在Service创建好后手动为其添加endpoint。



#### Volume 存储卷

默认情况下容器的数据是非持久化的，容器消亡以后数据也会跟着丢失，所以Docker提供了Volume机制以便将数据持久化存储。Kubernetes提供了更强大的Volume机制和插件，解决了容器数据持久化以及容器间共享数据的问题。

Kubernetes存储卷的生命周期与Pod绑定

- 容器挂掉后Kubelet再次重启容器时，Volume的数据依然还在
- Pod删除时，Volume才会清理。数据是否丢失取决于具体的Volume类型，比如emptyDir的数据会丢失，而PV的数据则不会丢

目前Kubernetes主要支持以下Volume类型：

- emptyDir：Pod存在，emptyDir就会存在，容器挂掉不会引起emptyDir目录下的数据丢失，但是pod被删除或者迁移，emptyDir也会被删除
- hostPath：hostPath允许挂载Node上的文件系统到Pod里面去
- NFS（Network File System）：网络文件系统，Kubernetes中通过简单地配置就可以挂载NFS到Pod中，而NFS中的数据是可以永久保存的，同时NFS支持同时写操作。
- glusterfs：同NFS一样是一种网络文件系统，Kubernetes可以将glusterfs挂载到Pod中，并进行永久保存
- cephfs：一种分布式网络文件系统，可以挂载到Pod中，并进行永久保存
- subpath：Pod的多个容器使用同一个Volume时，会经常用到
- secret：密钥管理，可以将敏感信息进行加密之后保存并挂载到Pod中
- persistentVolumeClaim：用于将持久化存储（PersistentVolume）挂载到Pod中
- ...



#### PersistentVolume(PV) 持久化存储卷

PersistentVolume(PV)是集群之中的一块网络存储。跟 Node 一样，也是集群的资源。PersistentVolume (PV)和PersistentVolumeClaim (PVC)提供了方便的持久化卷: PV提供网络存储资源，而PVC请求存储资源并将其挂载到Pod中。

PV的访问模式(accessModes)有三种:

- ReadWriteOnce(RWO):是最基本的方式，可读可写，但只支持被单个Pod挂载。
- ReadOnlyMany(ROX):可以以只读的方式被多个Pod挂载。
- ReadWriteMany(RWX):这种存储可以以读写的方式被多个Pod共享。

不是每一种存储都支持这三种方式，像共享方式，目前支持的还比较少，比较常用的是 NFS。在PVC绑定PV时通常根据两个条件来绑定，一个是存储的大小，另一个就是 访问模式。

PV的回收策略(persistentVolumeReclaimPolicy)也有三种

- Retain，不清理保留Volume(需要手动清理)
- Recycle，删除数据，即 rm -rf /thevolume/* (只有NFS和HostPath支持)
- Delete，删除存储资源



#### **Deployment**

Deployment 用于部署无状态应用，Deployment 为 Pod 和 ReplicaSet 提供声明式更新，只需要在 Deployment 描述想要的目标状态，Deployment 就会将 Pod 和 ReplicaSet 的实际状态改变到目标状态。

一般情况下我们不需要手动创建Pod实例，而是采用更高一层的抽象或定义来管理Pod，针对无状态类型的应用，Kubernetes使用Deloyment的Controller对象与之对应。其典型的应用场景包括：

- 定义Deployment来创建Pod和ReplicaSet
- 滚动升级和回滚应用
- 扩容和缩容
- 暂停和继续Deployment

常用的操作命令如下：

- kubectl run www--image=10.0.0.183:5000/hanker/www:0.0.1--port=8080 生成一个Deployment对象
- kubectlgetdeployment--all-namespaces 查找Deployment
- kubectl describe deployment www 查看某个Deployment
- kubectl edit deployment www 编辑Deployment定义
- kubectldeletedeployment www 删除某Deployment
- kubectl scale deployment/www--replicas=2 扩缩容操作，即修改Deployment下的Pod实例个数
- kubectlsetimage deployment/nginx-deployment nginx=nginx:1.9.1更新镜像
- kubectl rollout undo deployment/nginx-deployment 回滚操作
- kubectl rollout status deployment/nginx-deployment 查看回滚进度
- kubectl autoscale deployment nginx-deployment--min=10--max=15--cpu-percent=80 启用水平伸缩（HPA - horizontal pod autoscaling），设置最小、最大实例数量以及目标cpu使用率
- kubectl rollout pause deployment/nginx-deployment 暂停更新Deployment
- kubectl rollout resume deploy nginx 恢复更新Deployment

**更新策略**

.spec.strategy 指新的Pod替换旧的Pod的策略，有以下两种类型

- RollingUpdate 滚动升级，可以保证应用在升级期间，对外正常提供服务。
- Recreate 重建策略，在创建出新的Pod之前会先杀掉所有已存在的Pod。

Deployment和ReplicaSet两者之间的关系

- 使用Deployment来创建ReplicaSet。ReplicaSet在后台创建pod，检查启动状态，看它是成功还是失败。
- 当执行更新操作时，会创建一个新的ReplicaSet，Deployment会按照控制的速率将pod从旧的ReplicaSet移 动到新的ReplicaSet中



#### StatefulSet 有状态应用

Deployments和ReplicaSets是为无状态服务设计的，那么StatefulSet则是为了有状态服务而设计，其应用场景包括：

- 稳定的持久化存储，即Pod重新调度后还是能访问到相同的持久化数据，基于PVC来实现
- 稳定的网络标志，即Pod重新调度后其PodName和HostName不变，基于Headless Service(即没有Cluster IP的Service)来实现
- 有序部署，有序扩展，即Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次进行操作(即从0到N-1，在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态)，基于init containers来实现
- 有序收缩，有序删除(即从N-1到0)

支持两种更新策略：

- OnDelete:当 .spec.template更新时，并不立即删除旧的Pod，而是等待用户手动删除这些旧Pod后自动创建新Pod。这是默认的更新策略，兼容v1.6版本的行为
- RollingUpdate:当 .spec.template 更新时，自动删除旧的Pod并创建新Pod替换。在更新时这些Pod是按逆序的方式进行，依次删除、创建并等待Pod变成Ready状态才进行下一个Pod的更新。



#### DaemonSet 守护进程集

DaemonSet保证在特定或所有Node节点上都运行一个Pod实例，常用来部署一些集群的日志采集、监控或者其他系统管理应用。典型的应用包括:

- 日志收集，比如fluentd，logstash等
- 系统监控，比如Prometheus Node Exporter，collectd等
- 系统程序，比如kube-proxy, kube-dns, glusterd, ceph，ingress-controller等

指定Node节点

DaemonSet会忽略Node的unschedulable状态，有两种方式来指定Pod只运行在指定的Node节点上:

- nodeSelector:只调度到匹配指定label的Node上
- nodeAffinity:功能更丰富的Node选择器，比如支持集合操作
- podAffinity:调度到满足条件的Pod所在的Node上

目前支持两种策略

- OnDelete: 默认策略，更新模板后，只有手动删除了旧的Pod后才会创建新的Pod
- RollingUpdate: 更新DaemonSet模版后，自动删除旧的Pod并创建新的Pod



#### Ingress

Kubernetes中的负载均衡我们主要用到了以下两种机制：

- Service：使用Service提供集群内部的负载均衡，Kube-proxy负责将service请求负载均衡到后端的Pod中
- Ingress Controller：使用Ingress提供集群外部的负载均衡

Service和Pod的IP仅可在集群内部访问。集群外部的请求需要通过负载均衡转发到service所在节点暴露的端口上，然后再由kube-proxy通过边缘路由器将其转发到相关的Pod，Ingress可以给service提供集群外部访问的URL、负载均衡、HTTP路由等，为了配置这些Ingress规则，集群管理员需要部署一个Ingress Controller，它监听Ingress和service的变化，并根据规则配置负载均衡并提供访问入口。

常用的ingress controller：

- nginx
- traefik
- Kong
- Openresty



#### Job & CronJob 任务和定时任务

Job负责批量处理短暂的一次性任务 (short lived>CronJob即定时任务，就类似于Linux系统的crontab，在指定的时间周期运行指定的任务。





#### **Label**

标签，附加到某个资源上，用于关联对象、查询和筛选。一个 Label 是一个 key=value 的键值对，key 与 value 由用户自己指定。Label 可以附加到各种资源上，一个资源对象可以定义任意数量的 Label，同一个 Label 也可以被添加到任意数量的资源上。

我们可以通过给指定的资源对象捆绑一个或多个不同的 Label 来实现多维度的资源分组管理功能，以便于灵活、方便地进行资源分配、调度、配置、部署等工作。

K8S 通过 Label Selector（标签选择器）来查询和筛选拥有某些 Label 的资源对象。Label Selector 有基于等式（ name=label1 ）和基于集合（ name in (label1, label2) ）的两种方式。

#### **ReplicaSet（RC）**

ReplicaSet 用来确保预期的 Pod 副本数量，如果有过多的 Pod 副本在运行，系统就会停掉一些 Pod，否则系统就会再自动创建一些 Pod。

我们很少单独使用 ReplicaSet，它主要被 Deployment 这个更高层的资源对象使用，从而形成一整套 Pod 创建、删除、更新的编排机制。



#### **Horizontal Pod Autoscaler（HPA）**水平伸缩

HPA 为 Pod 横向自动扩容，也是 K8S 的一种资源对象。HPA 通过追踪分析 RC 的所有目标 Pod 的负载变化情况，来确定是否需要针对性调整目标 Pod 的副本数量。

Horizontal Pod Autoscaling可以根据CPU、内存使用率或应用自定义metrics自动扩展Pod数量 (支持replication controller、deployment和replica set)。

- 控制管理器默认每隔30s查询metrics的资源使用情况(可以通过 --horizontal-pod-autoscaler-sync-period 修改)
- 支持三种metrics类型
  - 预定义metrics(比如Pod的CPU)以利用率的方式计算
  - 自定义的Pod metrics，以原始值(raw value)的方式计算
  - 自定义的object metrics
- 支持两种metrics查询方式:Heapster和自定义的REST API
- 支持多metrics

可以通过如下命令创建HPA：kubectl autoscale deployment php-apache--cpu-percent=50--min=1--max=10



#### Service Account

Service account是为了方便Pod里面的进程调用Kubernetes API或其他外部服务而设计的授权

Service Account为服务提供了一种方便的认证机制，但它不关心授权的问题。可以配合RBAC(Role Based Access Control)来为Service Account鉴权，通过定义Role、RoleBinding、ClusterRole、ClusterRoleBinding来对sa进行授权。



#### Secret 密钥

Sercert-密钥解决了密码、token、密钥等敏感数据的配置问题，而不需要把这些敏感数据暴露到镜像或者Pod Spec中。Secret可以以Volume或者环境变量的方式使用。有如下三种类型：

- Service Account:用来访问Kubernetes API，由Kubernetes自动创建，并且会自动挂载到Pod的 /run/secrets/kubernetes.io/serviceaccount 目录中;
- Opaque:base64编码格式的Secret，用来存储密码、密钥等;
- kubernetes.io/dockerconfigjson: 用来存储私有docker registry的认证信息。



#### ConfigMap 配置中心

ConfigMap用于保存配置数据的键值对，可以用来保存单个属性，也可以用来保存配置文件。ConfigMap跟secret很类似，但它可以更方便地处理不包含敏感信息的字符串。ConfigMap可以通过三种方式在Pod中使用，三种分别方式为:设置环境变量、设置容器命令行参数以及在Volume中直接挂载文件或目录。

可以使用 kubectl create configmap从文件、目录或者key-value字符串创建等创建 ConfigMap。也可以通过 kubectl create-f value.yaml 创建。



Resource Quotas 资源配额**

资源配额(Resource Quotas)是用来限制用户资源用量的一种机制。

资源配额有如下类型：

- 计算资源，包括cpu和memory
  - cpu, limits.cpu, requests.cpu
  - memory, limits.memory, requests.memory
- 存储资源，包括存储资源的总量以及指定storage class的总量
  - requests.storage:存储资源总量，如500Gi
  - persistentvolumeclaims:pvc的个数
  - storageclass.storage.k8s.io/requests.storage
  - storageclass.storage.k8s.io/persistentvolumeclaims
- 对象数，即可创建的对象的个数
  - pods, replicationcontrollers, configmaps, secrets
  - resourcequotas, persistentvolumeclaims
  - services, services.loadbalancers, services.nodeports

它的工作原理为:

- 资源配额应用在Namespace上，并且每个Namespace最多只能有一个 ResourceQuota 对象
- 开启计算资源配额后，创建容器时必须配置计算资源请求或限制(也可以 用LimitRange设置默认值)，用户超额后禁止创建新的资源



## 集群搭建 —— 平台规划

### 生产环境 K8S 平台规划

K8S 环境有两种架构方式，单 Master 集群和多 Master 集群，将先搭建起单 Master 集群，再扩展为多 Master 集群。开发、测试环境可以部署单 Master 集群，生产环境为了保证高可用需部署多 Master 集群。

**① 单 Master 集群架构**

单 Master 集群架构相比于多 Master 集群架构无法保证集群的高可用，因为 master 节点一旦宕机就无法进行集群的管理工作了。单 master 集群主要包含一台 Master 节点，及多个 Node 工作节点、多个 Etcd 数据库节点。

Etcd 是 K8S 集群的数据库，可以安装在任何地方，也可以与 Master 节点在同一台机器上，只要 K8S 能连通 Etcd。

![img](png/Kubernetes/Kubernetes1.png)

**② 多 Master 集群架构**

多 Master 集群能保证集群的高可用，相比单 Master 架构，需要一个额外的负载均衡器来负载多个 Master 节点，Node 节点从连接 Master 改成连接 LB 负载均衡器。

![img](png/Kubernetes/Kubernetes2.png)

**③ 集群规划**

为了测试，我在本地使用 VMware 创建了几个虚拟机（可通过克隆快速创建虚拟机），一个虚拟机代表一台独立的服务器。

K8S 集群规划如下：

生产环境建议至少两台 Master 节点，LB 主备各一个节点；至少两台以上 Node 节点，根据实际运行的容器数量调整；Etcd 数据库可直接部署在 Master 和 Node 的节点，机器比较充足的话，可以部署在单独的节点上。

![img](https://img2018.cnblogs.com/blog/856154/201910/856154-20191031002330707-1803081307.png)

**④ 服务器硬件配置推荐**

测试环境与生产环境服务器配置推荐如下，本地虚拟机的配置将按照本地测试环境的配置来创建虚拟机。

![img](https://img2018.cnblogs.com/blog/856154/201910/856154-20191030010523449-1281148439.png)



### 操作系统初始化

接下来将基于二进制包的方式，手动部署每个组件，来组成 K8S 高可用集群。通过手动部署每个组件，一步步熟悉每个组件的配置、组件之间的通信等，深层次的理解和掌握 K8S。

首先做的是每台服务器的配置初始化，依次按如下步骤初始化。

① 关闭防火墙

```
# systemctl stop firewalld
# systemctl disable firewalld
```

② 关闭 selinux

```
# #临时生效
# setenforce 0
# #永久生效
# sed -i 's/enforcing/disabled/' /etc/selinux/config
```

③ 关闭 swap

```
# #临时关闭
# swapoff -a
# # 永久生效
# vim /etc/fstab# #将 [UUID=5b59fd54-eaad-41d6-90b2-ce28ac65dd81 swap                    swap    defaults        0 0] 这一行注释掉
```

④ 添加 hosts

```
# vim /etc/hosts

192.168.31.24 k8s-master-1
192.168.31.26 k8s-master-2
192.168.31.35 k8s-node-1
192.168.31.71 k8s-node-2
192.168.31.178 k8s-lb-master
192.168.31.224 k8s-lb-backup
```

⑤ 同步系统时间

各个节点之间需保持时间一致，因为自签证书是根据时间校验证书有效性，如果时间不一致，将校验不通过。

```
# #联网情况可使用如下命令
# ntpdate time.windows.com

# #如果不能联外网可使用 date 命令设置时间
```

## 集群搭建 —— 部署Etcd集群

### 自签证书

K8S 集群安装配置过程中，会使用各种证书，目的是为了加强集群安全性。K8S 提供了基于 CA 签名的双向数字证书认证方式和简单的基于 http base 或 token 的认证方式，其中 CA 证书方式的安全性最高。每个K8S集群都有一个集群根证书颁发机构（CA），集群中的组件通常使用CA来验证API server的证书，由API服务器验证kubelet客户端证书等。

证书生成操作可以在master节点上执行，证书只需要创建一次，以后在向集群中添加新节点时只要将证书拷贝到新节点上，并做一定的配置即可。下面就在 k8s-master-1 节点上来创建证书，详细的介绍也可以参考官方文档：分发自签名-CA-证书

① K8S 证书

如下是 K8S 各个组件需要使用的证书

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191112143250504-2083667283.png)

② 准备 cfssl 工具

我是使用 cfssl 工具来生成证书，首先下载 cfssl 工具。依次执行如下命令：

```
# curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /usr/local/bin/cfssl
# curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o /usr/local/bin/cfssljson
# curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o /usr/local/bin/cfssl-certinfo

# chmod +x /usr/local/bin/cfssl*
```

### 自签 Etcd SSL 证书

我们首先为 **etcd** 签发一套SSL证书，通过如下命令创建几个目录，/k8s/etcd/ssl 用于存放 etcd 自签证书，/k8s/etcd/cfg 用于存放 etcd 配置文件，/k8s/etcd/bin 用于存放 etcd 执行程序。

```
# cd /
# mkdir -p /k8s/etcd/{ssl,cfg,bin}
```

进入 etcd 目录：

```
# cd /k8s/etcd/ssl
```

① 创建 CA 配置文件：**ca-config.json**

执行如下命令创建 ca-config.json

```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "etcd": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```

**说明：**

- signing：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE；
- profiles：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile；
- expiry：证书过期时间
- server auth：表示client可以用该 CA 对server提供的证书进行验证；
- client auth：表示server可以用该CA对client提供的证书进行验证；

② 创建 CA 证书签名请求文件：**ca-csr.json**

执行如下命令创建 ca-csr.json：

```
cat > ca-csr.json <<EOF
{
  "CN": "etcd",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shanghai",
      "L": "Shanghai",
      "O": "etcd",
      "OU": "System"
    }
  ],
    "ca": {
       "expiry": "87600h"
    }
}
EOF
```

**说明：**

- CN：Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法；
- key：加密算法
- C：国家
- ST：地区
- L：城市
- O：组织，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)；
- OU：组织单位

③ 生成 CA 证书和私钥

```
# cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191114232231356-1578817713.png)

**说明：**

- **ca-key.pem**：CA 私钥
- **ca.pem**：CA 数字证书

④ 创建证书签名请求文件：**etcd-csr.json**

执行如下命令创建 etcd-csr.json：

```
cat > etcd-csr.json <<EOF
{
    "CN": "etcd",
    "hosts": [
      "192.168.31.24",
      "192.168.31.35",
      "192.168.31.71"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "etcd",
            "OU": "System"
        }
    ]
}
EOF
```

**说明：**

- **hosts**：需要指定授权使用该证书的 IP 或域名列表，这里配置所有 etcd 的IP地址。
- **key**：加密算法及长度

⑤ 为 etcd 生成证书和私钥

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=etcd etcd-csr.json | cfssljson -bare etcd
```

**说明：**

- -ca：指定 CA 数字证书
- -ca-key：指定 CA 私钥
- -config：CA 配置文件
- -profile：指定环境
- -bare：指定证书名前缀

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191114232316392-395705331.png)

**说明：**

- **ca-key.pem**：etcd 私钥
- **ca.pem**：etcd 数字证书

证书生成完成，后面部署 Etcd 时主要会用到如下几个证书：

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191114232356282-2007896330.png)



### Etcd 数据库集群部署

etcd 集群采用主从架构模式（一主多从）部署，集群通过选举产生 leader，因此需要部署奇数个节点（3/5/7）才能正常工作。etcd使用raft一致性算法保证每个节点的一致性。

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191203223130198-1515895263.png)

① 下载 etcd

从 github 上下载合适版本的 etcd

```
# cd /k8s/etcd
# wget https://github.com/etcd-io/etcd/releases/download/v3.2.28/etcd-v3.2.28-linux-amd64.tar.gz
```

解压：

```
# tar zxf etcd-v3.2.28-linux-amd64.tar.gz
```

将 etcd 复制到 /usr/local/bin 下：

```
# cp etcd-v3.2.28-linux-amd64/{etcd,etcdctl} /k8s/etcd/bin
# rm -rf etcd-v3.2.28-linux-amd64*
```

② 创建 etcd 配置文件：**etcd.conf**

```
cat > /k8s/etcd/cfg/etcd.conf <<EOF 
# [member]
ETCD_NAME=etcd-1
ETCD_DATA_DIR=/k8s/data/default.etcd
ETCD_LISTEN_PEER_URLS=https://192.168.31.24:2380
ETCD_LISTEN_CLIENT_URLS=https://192.168.31.24:2379

# [cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://192.168.31.24:2380
ETCD_ADVERTISE_CLIENT_URLS=https://192.168.31.24:2379
ETCD_INITIAL_CLUSTER=etcd-1=https://192.168.31.24:2380,etcd-2=https://192.168.31.35:2380,etcd-3=https://192.168.31.71:2380
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
ETCD_INITIAL_CLUSTER_STATE=new

# [security]
ETCD_CERT_FILE=/k8s/etcd/ssl/etcd.pem
ETCD_KEY_FILE=/k8s/etcd/ssl/etcd-key.pem
ETCD_TRUSTED_CA_FILE=/k8s/etcd/ssl/ca.pem
ETCD_PEER_CERT_FILE=/k8s/etcd/ssl/etcd.pem
ETCD_PEER_KEY_FILE=/k8s/etcd/ssl/etcd-key.pem
ETCD_PEER_TRUSTED_CA_FILE=/k8s/etcd/ssl/ca.pem
EOF
```

**说明：**

- ETCD_NAME：etcd在集群中的唯一名称
- ETCD_DATA_DIR：etcd数据存放目录
- ETCD_LISTEN_PEER_URLS：etcd集群间通讯的地址，设置为本机IP
- ETCD_LISTEN_CLIENT_URLS：客户端访问的地址，设置为本机IP
- 
- ETCD_INITIAL_ADVERTISE_PEER_URLS：初始集群通告地址，集群内部通讯地址，设置为本机IP
- ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址，设置为本机IP
- ETCD_INITIAL_CLUSTER：集群节点地址，以 key=value 的形式添加各个 etcd 的地址
- ETCD_INITIAL_CLUSTER_TOKEN：集群令牌，用于集群间做简单的认证
- ETCD_INITIAL_CLUSTER_STATE：集群状态
- 
- ETCD_CERT_FILE：客户端 etcd 数字证书路径
- ETCD_KEY_FILE：客户端 etcd 私钥路径
- ETCD_TRUSTED_CA_FILE：客户端 CA 证书路径
- ETCD_PEER_CERT_FILE：集群间通讯etcd数字证书路径
- ETCD_PEER_KEY_FILE：集群间通讯etcd私钥路径
- ETCD_PEER_TRUSTED_CA_FILE：集群间通讯CA证书路径

③ 创建 etcd 服务：**etcd.service**

通过EnvironmentFile指定 **etcd.conf** 作为环境配置文件

```
cat > /k8s/etcd/etcd.service <<'EOF'
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/k8s/etcd/cfg/etcd.conf
WorkingDirectory=${ETCD_DATA_DIR}

ExecStart=/k8s/etcd/bin/etcd \
  --name=${ETCD_NAME} \
  --data-dir=${ETCD_DATA_DIR} \
  --listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
  --listen-client-urls=${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
  --initial-advertise-peer-urls=${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
  --advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \
  --initial-cluster=${ETCD_INITIAL_CLUSTER} \
  --initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \
  --initial-cluster-state=${ETCD_INITIAL_CLUSTER_STATE} \
  --cert-file=${ETCD_CERT_FILE} \
  --key-file=${ETCD_KEY_FILE} \
  --trusted-ca-file=${ETCD_TRUSTED_CA_FILE} \
  --peer-cert-file=${ETCD_PEER_CERT_FILE} \
  --peer-key-file=${ETCD_PEER_KEY_FILE} \
  --peer-trusted-ca-file=${ETCD_PEER_TRUSTED_CA_FILE}

Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

etcd.service 更多的配置以及说明可以通过如下命令查看：

```
# /k8s/etcd/bin/etcd --help
```

④ 将 etcd 目录拷贝到另外两个节点

```
# scp -r /k8s root@k8s-node-1:/k8s
# scp -r /k8s root@k8s-node-2:/k8s
```

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191115222519900-1199048557.png)

⑤ 修改两个节点配置文件

修改 k8s-node-1 节点的 /k8s/etcd/cfg/etcd.conf：

```
# [member]
ETCD_NAME=etcd-2
ETCD_LISTEN_PEER_URLS=https://192.168.31.35:2380
ETCD_LISTEN_CLIENT_URLS=https://192.168.31.35:2379

# [cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://192.168.31.35:2380
ETCD_ADVERTISE_CLIENT_URLS=https://192.168.31.35:2379
```

修改 k8s-node-2 节点的 /k8s/etcd/cfg/etcd.conf：

```
# [member]
ETCD_NAME=etcd-2
ETCD_LISTEN_PEER_URLS=https://192.168.31.71:2380
ETCD_LISTEN_CLIENT_URLS=https://192.168.31.71:2379

# [cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://192.168.31.71:2380
ETCD_ADVERTISE_CLIENT_URLS=https://192.168.31.71:2379
```

⑥ 启动 etcd 服务

首先在三个节点将 **etcd.service** 拷贝到 /usr/lib/systemd/system/ 下

```
# cp /k8s/etcd/etcd.service /usr/lib/systemd/system/
# systemctl daemon-reload
```

在三个节点启动 etcd 服务

```
# systemctl start etcd
```

设置开机启动

```
# systemctl enable etcd
```

查看 etcd 的日志

```
# tail -f /var/log/messages
```

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191115232442732-2092355582.png)

注意：如果日志中出现连接异常信息，请确认所有节点防火墙是否开放2379,2380端口，或者直接关闭防火墙。

查看 etcd 集群状态

```
/k8s/etcd/bin/etcdctl \
    --ca-file=/k8s/etcd/ssl/ca.pem \
    --cert-file=/k8s/etcd/ssl/etcd.pem \
    --key-file=/k8s/etcd/ssl/etcd-key.pem \
    --endpoints=https://192.168.31.24:2379,https://192.168.31.35:2379,https://192.168.31.71:2379 \
cluster-health
```

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191115235034288-1984020155.png)

## 集群搭建 —— 部署Master组件

### 自签 ApiServer SSL 证书

K8S 集群中所有资源的访问和变更都是通过 kube-apiserver 的 REST API 来实现的，首先在 master 节点上部署 kube-apiserver 组件。

我们首先为 **apiserver** 签发一套SSL证书，过程与 etcd 自签SSL证书类似。通过如下命令创建几个目录，ssl 用于存放自签证书，cfg 用于存放配置文件，bin 用于存放执行程序，logs 存放日志文件。

```
# cd /
# mkdir -p /k8s/kubernetes/{ssl,cfg,bin,logs}
```

进入 kubernetes 目录：

```
# cd /k8s/kubernetes/ssl
```

① 创建 CA 配置文件：**ca-config.json**

执行如下命令创建 ca-config.json

```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```

② 创建 CA 证书签名请求文件：**ca-csr.json**

执行如下命令创建 ca-csr.json：

```
cat > ca-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shanghai",
      "L": "Shanghai",
      "O": "kubernetes",
      "OU": "System"
    }
  ],
    "ca": {
       "expiry": "87600h"
    }
}
EOF
```

③ 生成 CA 证书和私钥

```
# cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

④ 创建证书签名请求文件：**kubernetes-csr.json**

执行如下命令创建 kubernetes-csr.json：

```
cat > kubernetes-csr.json <<EOF
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "10.0.0.1",
      "192.168.31.24",
      "192.168.31.26",
      "192.168.31.35",
      "192.168.31.71",
      "192.168.31.26",
      "192.168.31.26",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "kubernetes",
            "OU": "System"
        }
    ]
}
EOF
```

**说明：**

- **hosts**：指定会直接访问 apiserver 的IP列表，一般需指定 etcd 集群、kubernetes master 集群的主机 IP 和 kubernetes 服务的服务 IP，Node 的IP一般不需要加入。

⑤ 为 kubernetes 生成证书和私钥

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
```

### 部署 kube-apiserver 组件

① 下载二进制包

通过 kubernetes Github 下载安装用的二进制包，我这里使用 **[v1.16.2](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.16.md#downloads-for-v1163)** 版本，server 二进制包已经包含了 master、node 上的各个组件，下载 server 二进制包即可。

考虑到网络问题，可以从百度网盘下载已经准备好的二进制包：链接: [下载离线安装包](https://pan.baidu.com/s/1LAaBajkPIWTU1Gd4Ez9ztw)。

将下载好的 kubernetes-v1.16.2-server-linux-amd64.tar.gz 上传到 /usr/local/src下，并解压：

```
# tar -zxf kubernetes-v1.16.2-server-linux-amd64.tar.gz
```

先将 master 节点上部署的组件拷贝到 /k8s/kubernetes/bin 目录下：

```
# cp -p /usr/local/src/kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler} /k8s/kubernetes/bin/

# cp -p /usr/local/src/kubernetes/server/bin/kubectl /usr/local/bin/
```

② 创建 Node 令牌文件：**token.csv**

Master apiserver 启用 TLS 认证后，Node节点 kubelet 组件想要加入集群，必须使用CA签发的有效证书才能与apiserver通信，当Node节点很多时，签署证书是一件很繁琐的事情，因此有了 TLS Bootstrap 机制，kubelet 会以一个低权限用户自动向 apiserver 申请证书，kubelet 的证书由 apiserver 动态签署。因此先为 apiserver 生成一个令牌文件，令牌之后会在 Node 中用到。

生成 token，一个随机字符串，可使用如下命令生成 token：apiserver 配置的 token 必须与 Node 节点 bootstrap.kubeconfig 配置保持一致。

```
# head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191120142640146-1408433933.png)

创建 token.csv，格式：token，用户，UID，用户组

```
cat > /k8s/kubernetes/cfg/token.csv <<'EOF'
bfa3cb7f6f21f87e5c0e5f25e6cfedad,kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF
```

③ 创建 kube-apiserver 配置文件：**kube-apiserver.conf**

kube-apiserver 有很多配置项，可以参考官方文档查看每个配置项的用途：[kube-apiserver](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/)

注意：踩的一个坑，“\” 后面不要有空格，不要有多余的换行，否则启动失败。

```
cat > /k8s/kubernetes/cfg/kube-apiserver.conf <<'EOF'
KUBE_APISERVER_OPTS="--etcd-servers=https://192.168.31.24:2379,https://192.168.31.35:2379,https://192.168.31.71:2379 \
  --bind-address=192.168.31.24 \
  --secure-port=6443 \
  --advertise-address=192.168.31.24 \
  --allow-privileged=true \
  --service-cluster-ip-range=10.0.0.0/24 \
  --service-node-port-range=30000-32767 \
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
  --authorization-mode=RBAC,Node \
  --enable-bootstrap-token-auth=true \
  --token-auth-file=/k8s/kubernetes/cfg/token.csv \
  --kubelet-client-certificate=/k8s/kubernetes/ssl/kubernetes.pem \
  --kubelet-client-key=/k8s/kubernetes/ssl/kubernetes-key.pem \
  --tls-cert-file=/k8s/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/k8s/kubernetes/ssl/kubernetes-key.pem \
  --client-ca-file=/k8s/kubernetes/ssl/ca.pem \
  --service-account-key-file=/k8s/kubernetes/ssl/ca-key.pem \
  --etcd-cafile=/k8s/etcd/ssl/ca.pem \
  --etcd-certfile=/k8s/etcd/ssl/etcd.pem \
  --etcd-keyfile=/k8s/etcd/ssl/etcd-key.pem \
  --v=2 \
  --logtostderr=false \
  --log-dir=/k8s/kubernetes/logs \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/k8s/kubernetes/logs/k8s-audit.log"
EOF
```

**重点配置说明：**

- --etcd-servers：etcd 集群地址
- --bind-address：apiserver 监听的地址，一般配主机IP
- --secure-port：监听的端口
- --advertise-address：集群通告地址，其它Node节点通过这个地址连接 apiserver，不配置则使用 --bind-address
- --service-cluster-ip-range：Service 的 虚拟IP范围，以CIDR格式标识，该IP范围不能与物理机的真实IP段有重合。
- --service-node-port-range：Service 可映射的物理机端口范围，默认30000-32767
- --admission-control：集群的准入控制设置，各控制模块以插件的形式依次生效，启用RBAC授权和节点自管理
- --authorization-mode：授权模式，包括：AlwaysAllow，AlwaysDeny，ABAC(基于属性的访问控制)，Webhook，RBAC(基于角色的访问控制)，Node(专门授权由 kubelet 发出的API请求)。（默认值"AlwaysAllow"）。
- --enable-bootstrap-token-auth：启用TLS bootstrap功能
- --token-auth-file：这个文件将被用于通过令牌认证来保护API服务的安全端口。
- --v：指定日志级别，0~8，越大日志越详细

④ 创建 apiserver 服务：**kube-apiserver.service**

```
cat > /usr/lib/systemd/system/kube-apiserver.service <<'EOF'
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/k8s/kubernetes/cfg/kube-apiserver.conf
ExecStart=/k8s/kubernetes/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

⑤ 启动 kube-apiserver 组件

启动组件

```
# systemctl daemon-reload
# systemctl start kube-apiserver
# systemctl enable kube-apiserver
```

检查启动状态

```
# systemctl status kube-apiserver.service
```

查看启动日志

```
# less /k8s/kubernetes/logs/kube-apiserver.INFO
```

⑥ 将 kubelet-bootstrap 用户绑定到系统集群角色，之后便于 Node 使用token请求证书

```
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```



### 部署 kube-controller-manager 组件

① 创建 kube-controller-manager 配置文件：**kube-controller-manager.conf**

详细的配置可参考官方文档：[kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)

```
cat > /k8s/kubernetes/cfg/kube-controller-manager.conf <<'EOF'
KUBE_CONTROLLER_MANAGER_OPTS="--leader-elect=true \
  --master=127.0.0.1:8080 \
  --address=127.0.0.1 \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.244.0.0/16 \
  --service-cluster-ip-range=10.0.0.0/24 \
  --cluster-signing-cert-file=/k8s/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/k8s/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/k8s/kubernetes/ssl/ca.pem \
  --service-account-private-key-file=/k8s/kubernetes/ssl/ca-key.pem \
  --experimental-cluster-signing-duration=87600h0m0s \
  --v=2 \
  --logtostderr=false \
  --log-dir=/k8s/kubernetes/logs"
EOF
```

**重点配置说明：**

- --leader-elect：当该组件启动多个时，自动选举，默认true
- --master：连接本地apiserver，apiserver 默认会监听本地8080端口
- --allocate-node-cidrs：是否分配和设置Pod的CDIR
- --service-cluster-ip-range：Service 集群IP段

② 创建 kube-controller-manager 服务：**kube-controller-manager.service**

```
cat > /usr/lib/systemd/system/kube-controller-manager.service <<'EOF'
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=/k8s/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/k8s/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

③ 启动 kube-controller-manager 组件

启动组件

```
# systemctl daemon-reload
# systemctl start kube-controller-manager
# systemctl enable kube-controller-manager
```

检查启动状态

```
# systemctl status kube-controller-manager.service
```

查看启动日志

```
# less /k8s/kubernetes/logs/kube-controller-manager.INFO
```



### 部署 kube-scheduler 组件

① 创建 kube-scheduler 配置文件：**kube-scheduler.conf**

```
cat > /k8s/kubernetes/cfg/kube-scheduler.conf <<'EOF'
KUBE_SCHEDULER_OPTS="--leader-elect=true \
  --master=127.0.0.1:8080 \
  --address=127.0.0.1 \
  --v=2 \
  --logtostderr=false \
  --log-dir=/k8s/kubernetes/logs"
EOF
```

② 创建 kube-scheduler 服务：**kube-scheduler.service**

```
cat > /usr/lib/systemd/system/kube-scheduler.service <<'EOF'
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=/k8s/kubernetes/cfg/kube-scheduler.conf
ExecStart=/k8s/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

③ 启动 kube-scheduler 组件

启动组件

```
# systemctl daemon-reload
# systemctl start kube-scheduler
# systemctl enable kube-scheduler
```

查看启动状态

```
# systemctl status kube-scheduler.service
```

查看启动日志

```
# less /k8s/kubernetes/logs/kube-scheduler.INFO
less /k8s/kubernetes/logs/kube-scheduler.INFO
```

### 查看集群状态

① 查看组件状态

```
# kubectl get cs
```

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191120235101404-1573446834.png)

## 集群搭建 —— 部署Node组件

### 安装 Docker

CentOS 安装参考官方文档：https://docs.docker.com/install/linux/docker-ce/centos/

① 卸载旧版本

```
# yum remove docker docker-common docker-selinux
```

② 安装依赖包

```
# yum install -y yum-utils device-mapper-persistent-data lvm2
```

③ 安装 Docker 软件包源

```
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

④ 安装 Docker CE

```
# yum install docker-ce
```

⑤ 启动 Docker 服务

```
# systemctl start docker
```

⑥ 设置开机启动

```
# systemctl enable docker
```

⑦ 验证安装是否成功

```
# docker -v
# docker info
```

![img](https://img2018.cnblogs.com/blog/856154/201908/856154-20190806005734464-186989792.png)



### Node 节点证书

① 创建 Node 节点的证书签名请求文件：**kube-proxy-csr.json**

首先在 k8s-master-1 节点上，通过颁发的 CA 证书先创建好 Node 节点要使用的证书，先创建证书签名请求文件：kube-proxy-csr.json：

```
cat > kube-proxy-csr.json <<EOF
{
    "CN": "system:kube-proxy",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "kubernetes",
            "OU": "System"
        }
    ]
}
EOF
```

② 为 kube-proxy 生成证书和私钥

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191127220706167-1455473162.png)

③ node 节点创建工作目录

在 k8s-node-1 节点上创建 k8s 目录

```
# mkdir -p /k8s/kubernetes/{bin,cfg,logs,ssl}
```

④ 将 k8s-master-1 节点的文件拷贝到 node 节点

将 kubelet、kube-proxy 拷贝到 node 节点上：

```
# scp -r /usr/local/src/kubernetes/server/bin/{kubelet,kube-proxy} root@k8s-node-1:/k8s/kubernetes/bin/
```

将证书拷贝到 k8s-node-1 节点上：

```
# scp -r /k8s/kubernetes/ssl/{ca.pem,kube-proxy.pem,kube-proxy-key.pem} root@k8s-node-1:/k8s/kubernetes/ssl/
```

### 安装 kubelet

① 创建请求证书的配置文件：**bootstrap.kubeconfig**

bootstrap.kubeconfig 将用于向 apiserver 请求证书，apiserver 会验证 token、证书 是否有效，验证通过则自动颁发证书。

```
cat > /k8s/kubernetes/cfg/bootstrap.kubeconfig <<'EOF'
apiVersion: v1
clusters:
- cluster: 
    certificate-authority: /k8s/kubernetes/ssl/ca.pem
    server: https://192.168.31.24:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubelet-bootstrap
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: kubelet-bootstrap
  user:
    token: bfa3cb7f6f21f87e5c0e5f25e6cfedad
EOF
```

**说明：**

- certificate-authority：CA 证书
- server：master 地址
- token：master 上 token.csv 中配置的 token

② 创建 kubelet 配置文件：**kubelet-config.yml**

为了安全性，kubelet 禁止匿名访问，必须授权才可以，通过 kubelet-config.yml 授权 apiserver 访问 kubelet。

```
cat > /k8s/kubernetes/cfg/kubelet-config.yml <<'EOF'
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2 
clusterDomain: cluster.local
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509: 
    clientCAFile: /k8s/kubernetes/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthroizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 100000
maxPods: 110
EOF
```

**说明：**

- address：kubelet 监听地址
- port：kubelet 的端口
- cgroupDriver：cgroup 驱动，与 docker 的 cgroup 驱动一致
- authentication：访问 kubelet 的授权信息
- authorization：认证相关信息
- evictionHard：垃圾回收策略
- maxPods：最大pod数

③ 创建 kubelet 服务配置文件：**kubelet.conf**

```
cat > /k8s/kubernetes/cfg/kubelet.conf <<'EOF'
KUBELET_OPTS="--hostname-override=k8s-node-1 \
  --network-plugin=cni \
  --cni-bin-dir=/opt/cni/bin \
  --cni-conf-dir=/etc/cni/net.d \
  --cgroups-per-qos=false \
  --enforce-node-allocatable="" \
  --kubeconfig=/k8s/kubernetes/cfg/kubelet.kubeconfig \
  --bootstrap-kubeconfig=/k8s/kubernetes/cfg/bootstrap.kubeconfig \
  --config=/k8s/kubernetes/cfg/kubelet-config.yml \
  --cert-dir=/k8s/kubernetes/ssl \
  --pod-infra-container-image=kubernetes/pause:latest \
  --v=2 \
  --logtostderr=false \
  --log-dir=/k8s/kubernetes/logs"
EOF
```

**说明：**

- --hostname-override：当前节点注册到K8S中显示的名称，默认为主机 hostname
- --network-plugin：启用 CNI 网络插件
- --cni-bin-dir：CNI 插件可执行文件位置，默认在 /opt/cni/bin 下
- --cni-conf-dir：CNI 插件配置文件位置，默认在 /etc/cni/net.d 下
- --cgroups-per-qos：必须加上这个参数和--enforce-node-allocatable，否则报错 [Failed to start ContainerManager failed to initialize top level QOS containers.......]
- --kubeconfig：会自动生成 kubelet.kubeconfig，用于连接 apiserver
- --bootstrap-kubeconfig：指定 bootstrap.kubeconfig 文件
- --config：kubelet 配置文件
- --cert-dir：证书目录
- --pod-infra-container-image：管理Pod网络的镜像，基础的 Pause 容器，默认是 k8s.gcr.io/pause:3.1

④ 创建 kubelet 服务：**kubelet.service**

```
cat > /usr/lib/systemd/system/kubelet.service <<'EOF'
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Before=docker.service

[Service]
EnvironmentFile=/k8s/kubernetes/cfg/kubelet.conf
ExecStart=/k8s/kubernetes/bin/kubelet $KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

⑤ 启动 kubelet

```
# systemctl daemon-reload
# systemctl start kubelet
```

开机启动：

```
# systemctl enable kubelet
```

查看启动日志： 

```
# tail -f /k8s/kubernetes/logs/kubelet.INFO
```

⑥ master 给 node 授权

kubelet 启动后，还没加入到集群中，会向 apiserver 请求证书，需手动在 k8s-master-1 上对 node 授权。

通过如下命令查看是否有新的客户端请求颁发证书：

```
# kubectl get csr
```

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191127220530728-425743917.png)

给客户端颁发证书，允许客户端加入集群：

```
# kubectl certificate approve node-csr-FoPLmv3Sr2XcYvNAineE6RpdARf2eKQzJsQyfhk-xf8
```

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191127220827044-1194859709.png)

⑦ 授权成功

查看 node 是否加入集群(此时的 node 还处于未就绪的状态，因为还没有安装 CNI 组件)：

```
# kubectl get node
```

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191127220909837-1664629101.png)

颁发证书后，可以在 /k8s/kubenetes/ssl 下看到 master 为 kubelet 颁发的证书：

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191127221220075-1579437290.png)

在 /k8s/kubenetes/cfg 下可以看到自动生成的 kubelet.kubeconfig 配置文件：

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191127221332515-613675599.png)



### 安装 kube-proxy

① 创建 kube-proxy 连接 apiserver 的配置文件：**kube-proxy.kubeconfig**

```
cat > /k8s/kubernetes/cfg/kube-proxy.kubeconfig <<'EOF'
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /k8s/kubernetes/ssl/ca.pem
    server: https://192.168.31.24:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kube-proxy
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: kube-proxy
  user:
    client-certificate: /k8s/kubernetes/ssl/kube-proxy.pem
    client-key: /k8s/kubernetes/ssl/kube-proxy-key.pem
EOF
```

② 创建 kube-proxy 配置文件：**kube-proxy-config.yml**

```
cat > /k8s/kubernetes/cfg/kube-proxy-config.yml <<'EOF'
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
address: 0.0.0.0
metrisBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /k8s/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-node-1
clusterCIDR: 10.0.0.0/24
mode: ipvs
ipvs:
  scheduler: "rr"
iptables:
  masqueradeAll: true
EOF
```

**说明：**

- metrisBindAddress：采集指标暴露的地址端口，便于监控系统，采集数据
- clusterCIDR：集群 Service 网段

③ 创建 kube-proxy 配置文件：**kube-proxy.conf**

```
cat > /k8s/kubernetes/cfg/kube-proxy.conf <<'EOF'
KUBE_PROXY_OPTS="--config=/k8s/kubernetes/cfg/kube-proxy-config.yml \
  --v=2 \
  --logtostderr=false \
  --log-dir=/k8s/kubernetes/logs"
EOF
```

④ 创建 kube-proxy 服务：**kube-proxy.service**

```
cat > /usr/lib/systemd/system/kube-proxy.service <<'EOF'
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
EnvironmentFile=/k8s/kubernetes/cfg/kube-proxy.conf
ExecStart=/k8s/kubernetes/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

⑤ 启动 kube-proxy

启动服务：

```
# systemctl daemon-reload
# systemctl start kube-proxy
```

开机启动：

```
# systemctl enable kube-proxy
```

查看启动日志： 

```
# tail -f /k8s/kubernetes/logs/kube-proxy.INFO
```

### 部署其它Node节点

部署其它 Node 节点基于与上述流程一致，只需将配置文件中 k8s-node-1 改为 k8s-node-2 即可。

```
# kubectl get node -o wide
```

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191128224015295-979089341.png)

### 部署K8S容器集群网络(Flannel)

① K8S 集群网络

Kubernetes 项目并没有使用 Docker 的网络模型，kubernetes 是通过一个 CNI 接口维护一个单独的网桥来代替 docker0，这个网桥默认叫 **cni0**。

CNI（Container Network Interface）是CNCF旗下的一个项目，由一组用于配置 Linux 容器的网络接口的规范和库组成，同时还包含了一些插件。CNI仅关心容器创建时的网络分配，和当容器被删除时释放网络资源。

Flannel 是 CNI 的一个插件，可以看做是 CNI 接口的一种实现。Flannel 是针对 Kubernetes 设计的一个网络规划服务，它的功能是让集群中的不同节点主机创建的Docker容器都具有全集群唯一的虚拟IP地址，并让属于不同节点上的容器能够直接通过内网IP通信。

Flannel 网络架构请参考：[flannel 网络架构](https://ggaaooppeenngg.github.io/zh-CN/2017/09/21/flannel-网络架构/)

② 创建 CNI 工作目录

通过给 kubelet 传递 **--network-plugin=cni** 命令行选项来启用 CNI 插件。 kubelet 从 **--cni-conf-dir** （默认是 /etc/cni/net.d）读取配置文件并使用该文件中的 CNI 配置来设置每个 pod 的网络。CNI 配置文件必须与 CNI 规约匹配，并且配置引用的任何所需的 CNI 插件都必须存在于 **--cni-bin-dir**（默认是 /opt/cni/bin）指定的目录。

由于前面部署 kubelet 服务时，指定了 **--cni-conf-dir=/etc/cni/net.d**，**--cni-bin-dir=/opt/cni/bin**，因此首先在node节点上创建这两个目录：

```
# mkdir -p /opt/cni/bin /etc/cni/net.d
```

③ 装 CNI 插件

可以从 github 上下载 CNI 插件：[下载 CNI 插件](https://github.com/containernetworking/plugins/releases) 。

解压到 /opt/cni/bin：

```
# tar zxf cni-plugins-linux-amd64-v0.8.2.tgz -C /opt/cni/bin/
```

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191128220554099-1684002659.png)

④ 部署 Flannel

可通过此地址下载 flannel 配置文件：[下载 kube-flannel.yml](https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml)

注意如下配置：Network 的地址需与 **kube-controller-manager.conf** 中的 **--cluster-cidr=10.244.0.0/16** 保持一致。

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191128112822714-1964053859.png)

在 k8s-master-1 节点上部署 Flannel：

```
# kubectl apply -f kube-flannel.yml
```

![img](https://img2018.cnblogs.com/blog/856154/201911/856154-20191128230506906-1309290806.png)

⑤ 检查部署状态

Flannel 会在 Node 上起一个 Flannel 的 Pod，可以查看 pod 的状态看 flannel 是否启动成功：

```
# kubectl get pods -n kube-system -o wide
```

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191205005818391-377110510.png)

 Flannel 部署成功后，就可以看 Node 是否就绪：

```
# kubectl get nodes -o wide
```

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191205010005973-37990797.png)

在 Node 上查看网络配置，可以看到多了一个 **flannel.1** 的虚拟网卡，这块网卡用于接收 Pod 的流量并转发出去。

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191205010303481-754515627.png)

⑥ 测试创建 Pod

例如创建一个 Nginx 服务：

```
# kubectl create deployment web --image=nginx
```

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191205010658719-540734499.png)

查看 Pod 状态：

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191205010735690-1937177430.png)

在对应的节点上可以看到部署的容器：

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191205012234493-297995248.png)

容器创建成功后，再在 Node 上查看网络配置，又多了一块 cni0 的虚拟网卡，cni0 用于 pod 本地通信使用。

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191205010844816-242752920.png)

暴露端口并访问 Nginx：

```
# kubectl expose deployment web --port=80 --type=NodePort
```

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191205012049946-519639325.png)



### 部署内部 DNS 服务

在Kubernetes集群推荐使用Service Name作为服务的访问地址，因此需要一个Kubernetes集群范围的DNS服务实现从Service Name到Cluster IP的解析，这就是Kubernetes基于DNS的服务发现功能。

① 部署 CoreDNS

下载CoreDNS配置文件：[coredns.yaml](https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/coredns/coredns.yaml.base)

注意如下 clusterIP 一定要与 **kube-config.yaml** 中的 **clusterDNS** 保持一致

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191206013327734-494461951.png)

部署 CoreDNS：

```
$ kubectl apply -f coredns.yaml
```

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191208235827818-71864912.png)

② 验证 CoreDNS

创建 busybox 服务：

```
# cat > busybox.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  dnsPolicy: ClusterFirst
  containers:
  - name: busybox
    image: busybox:1.28.4
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF# kubectl apply -f busybox.yaml
```

验证是否安装成功：

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191209011552538-431087460.png)

## 集群搭建 —— 部署 Dashboard

K8S 提供了一个 Web 版 Dashboard，用户可以用 dashboard 部署容器化的应用、监控应用的状态，能够创建和修改各种 K8S 资源，比如 Deployment、Job、DaemonSet 等。用户可以 Scale Up/Down Deployment、执行 Rolling Update、重启某个 Pod 或者通过向导部署新的应用。Dashboard 能显示集群中各种资源的状态以及日志信息。Kubernetes Dashboard 提供了 kubectl 的绝大部分功能。

### 部署 K8S Dashboard

通过此地址下载 dashboard yaml文件：[kubernetes-dashboard.yaml](http://mirror.faasx.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml)

下载下来之后，需更新如下内容：通过 Node 暴露端口访问 dashboard

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191206010304862-1366726165.png)

部署 dashboard：

```
# kubectl apply -f kubernetes-dashboard.yaml
```

查看是否部署成功：

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191206010403304-236700234.png)

通过 https 访问 dashboard：

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191206010519105-236777362.png)



### 登录授权

Dashboard 支持 Kubeconfig 和 Token 两种认证方式，为了简化配置，我们通过配置文件为 Dashboard 默认用户赋予 admin 权限。

```
cat > kubernetes-adminuser.yaml <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF
```

授权：

```
# kubectl apply -f kubernetes-adminuser.yaml
```

获取登录的 token：

```
# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk ' {print $1}')
```

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191206011211859-592094825.png)

通过token登录进 dashboard，就可以查看集群的信息：

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191206011456607-1992406368.png)

## 集群搭建 —— 多 Master 部署

### 部署Master2组件

① 将 k8s-master-1 上相关文件拷贝到 k8s-master-2 上

创建k8s工作目录：

```
# mkdir -p /k8s/kubernetes
# mkdir -p /k8s/etcd
```

拷贝 k8s 配置文件、执行文件、证书：

```
# scp -r /k8s/kubernetes/{cfg,ssl,bin} root@k8s-master-2:/k8s/kubernetes
# cp /k8s/kubernetes/bin/kubectl /usr/local/bin/
```

拷贝 etcd 证书：

```
# scp -r /k8s/etcd/ssl root@k8s-master-2:/k8s/etcd
```

拷贝 k8s 服务的service文件：

```
# scp /usr/lib/systemd/system/kube-* root@k8s-master-2:/usr/lib/systemd/system
```

② 修改 k8s-master-2 上的配置文件

修改 kube-apiserver.conf，修改IP为本机IP

 

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191210231658832-199791846.png)

③ 启动 k8s-master-2 组件

重新加载配置：

```
# systemctl daemon-reload
```

启动 kube-apiserver：

```
# systemctl start kube-apiserver
# systemctl enable kube-apiserver
```

启动 kube-controller-manager：

```
# systemctl start kube-controller-manager
# systemctl enable kube-controller-manager
```

部署 kube-scheduler：

```
# systemctl start kube-scheduler
# systemctl enable kube-scheduler
```

④ 验证

![img](https://img2018.cnblogs.com/blog/856154/201912/856154-20191210233157623-672470319.png)



### 部署 Nginx 负载均衡

为了保证 k8s master 的高可用，将使用 k8s-lb-master 和 k8s-lb-backup 这两台机器来部署负载均衡。这里使用 nginx 做负载均衡器，下面分别在 k8s-lb-master 和 k8s-lb-backup 这两台机器上部署 nginx。

① gcc等环境安装，后续有些软件安装需要这些基础环境

```
# gcc安装：
# yum install gcc-c++
# PCRE pcre-devel 安装：
# yum install -y pcre pcre-devel
# zlib 安装：
# yum install -y zlib zlib-devel
#OpenSSL 安装：
# yum install -y openssl openssl-devel
```

② 安装nginx

```
# rpm -ivh https://nginx.org/packages/rhel/7/x86_64/RPMS/nginx-1.16.1-1.el7.ngx.x86_64.rpm
```

③ apiserver 负载配置

```
# vim /etc/nginx/nginx.conf
```

增加如下配置：

```
stream {
    log_format main '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';
    access_log /var/log/nginx/k8s-access.log main;

    upstream k8s-apiserver {
        server 192.168.31.24:6443;
        server 192.168.31.26:6443;
    }

    server {
        listen 6443;
        proxy_pass k8s-apiserver;
    }
}
```

④ 启动 nginx

```
# systemctl start nginx
# systemctl enable nginx
```

### 部署 KeepAlive

为了保证 nginx 的高可用，还需要部署 keepalive，keepalive 主要负责 nginx 的健康检查和故障转移。

① 分别在 k8s-lb-master 和 k8s-lb-backup 这两台机器上安装 keepalive

```
# yum install keepalived -y
```

② master 启动 keepalived

修改 k8s-lb-master keepalived 配置文件

```
cat > /etc/keepalived/keepalived.conf <<'EOF'
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id NGINX_MASTER
}

vrrp_script check_nginx {
   script "/etc/keepalived/check_nginx.sh" 
}

# vrrp实例
vrrp_instance VI_1 {
    state MASTER 
    interface ens33 
    virtual_router_id 51 
    priority 100
    advert_int 1 
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.31.100/24 
    }
    track_script {
        check_nginx
    }
}
EOF
```

**配置说明：**

- vrrp_script：用于健康检查nginx状态，如果nginx没有正常工作，就会进行故障漂移，使备节点接管VIP，这里通过 shell 脚本来检查nginx状态
- state：keepalived 角色，主节点为 MASTER，备节点为 BACKUP
- interface：接口，配置本地网卡名，keepalived 会将虚拟IP绑定到这个网卡上
- virtual_router_id：#VRRP 路由ID实例，每个实例是唯一的
- priority：优先级，备服务器设置90
- advert_int：指定VRRP心跳包通告间隔时间，默认1秒
- virtual_ipaddress：VIP，要与当前机器在同一网段，keepalived 会在网卡上附加这个IP，之后通过这个IP来访问Nginx，当nginx不可用时，会将此虚拟IP漂移到备节点上。

增加 check_nginx.sh 脚本，通过此脚本判断 nginx 是否正常：

```
cat > /etc/keepalived/check_nginx.sh <<'EOF'
#!/bin/bash
count=$(ps -ef | grep nginx | egrep -cv "grep|$$")
if [ "$count" -eq 0 ];then
    exit 1;
else 
    exit 0;
fi
EOF
```

增加可执行权限：

```
# chmod +x /etc/keepalived/check_nginx.sh
```

启动 keepalived：

```
# systemctl start keepalived
# systemctl enable keepalived
```

③ backup 启动 keepalived

修改 k8s-lb-backup keepalived 配置文件

```
cat > /etc/keepalived/keepalived.conf <<'EOF'
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id NGINX_BACKUP
}

vrrp_script check_nginx {
   script "/etc/keepalived/check_nginx.sh"
}

# vrrp实例
vrrp_instance VI_1 {
    state BACKUP
    interface eno16777736
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.31.100/24
    }
    track_script {
        check_nginx
    }
}
EOF
```

增加 check_nginx.sh 脚本：

```
cat > /etc/keepalived/check_nginx.sh <<'EOF'
#!/bin/bash
count=$(ps -ef | grep nginx | egrep -cv "grep|$$")
if [ "$count" -eq 0 ];then
    exit 1;
else 
    exit 0;
fi
EOF
```

增加可执行权限：

```
# chmod +x /etc/keepalived/check_nginx.sh
```

启动 keepalived：

```
# systemctl start keepalived
# systemctl enable keepalived
```

④ 验证负载均衡

keepalived 已经将VIP附加到MASTER所在的网卡上

![img](https://img2018.cnblogs.com/blog/856154/202001/856154-20200105231349220-1660151152.png)

BACKUP节点上并没有

![img](https://img2018.cnblogs.com/blog/856154/202001/856154-20200105232120068-658127475.png)

关闭 k8s-lb-master 上的nginx，可看到VIP已经不在了

![img](https://img2018.cnblogs.com/blog/856154/202001/856154-20200105233741081-433366894.png)

可以看到已经漂移到备节点上了，如果再重启 MASTER 上的 Ngnix，VIP又会漂移到主节点上。

![img](https://img2018.cnblogs.com/blog/856154/202001/856154-20200105233815959-403369443.png)

访问虚拟IP还是可以访问的

![img](https://img2018.cnblogs.com/blog/856154/202001/856154-20200105233850964-1133520064.png)

4、Node节点连接VIP

① 修改 node 节点的配置文件中连接 k8s-master 的IP为VIP

![img](https://img2018.cnblogs.com/blog/856154/202001/856154-20200105234648106-1656311814.png)

② 重启 kubelet 和 kube-proxy

```
# systemctl restart kubelet
# systemctl restart kube-proxy
```

③ 在 node 节点上访问VIP调用 apiserver 验证

Authorization 的token 是前面生成 token.csv 中的令牌。

![img](https://img2018.cnblogs.com/blog/856154/202001/856154-20200106001143565-1533967509.png)

 

至此，通过二进制方式安装 K8S 集群就算完成了！！

Kubernetes是Google开源的容器集群管理系统，是Google多年⼤规模容器管理技术Borg的开源版本，主要功能包括:

- 基于容器的应用部署、维护和滚动升级
- 负载均衡和服务发现
- 跨机器和跨地区的集群调度
- 自动伸缩
- 无状态服务和有状态服务
- 广泛的Volume支持
- 插件机制保证扩展性

Kubernetes发展非常迅速，已经成为容器编排领域的领导者，接下来我们将讲解Kubernetes中涉及到的一些主要概念。

# K8S命令笔记

## 基本命令

### 操作 Node

#### 查看 Node

1.查看 master 节点初始化结果

````shell
kubectl get nodes -o wide
````

####  删除 Node

1.移除 worker 节点

①在需要移除的 worker 节点（如 NAME=worker）上执行命令

````shell
kubeadm reset
````

②在第一个 master 节点上执行命令

> 通过在 master 上执行 `kubectl get nodes -o wide` 来获取节点 NAME 等相关信息。

````shell
kubectl delete node [NODE NAME]

# 示例
kubectl delete node worker
````

### 操作 Pod

#### 查看 Pod

1.查看容器组中所有 Pod 信息

````shell
kubectl get pod -n kube-system -o wide
````

2.查看指定 Pod 完整信息

````shell
kubectl describe pod [Pod NAME] -n [Namespace NAME]

# 示例
kubectl describe pod coredns-66db54ff7f-ddz8k -n kube-system
````

3.查看指定 Pod 日志

````shell
# -f 表示实时刷新
# --tail=n，tail 表示查看日志尾部，n 表示显示n行
# -n 表示查看指定 Namespace
kubectl logs -f --tail=n [Pod NAME] -n [Namespace NAME]

# 示例
kubectl logs -f --tail=500 coredns-66db54ff7f-ddz8k -n kube-system
````

#### 删除 Pod

1.删除指定 Pod

````shell
kubectl delete pod [Pod NAME] -n kube-system

# 示例
kubectl delete pod kube-flannel-ds-amd64-8l25c -n kube-system
````

### 操作 Namespace

#### 查看 Namespace

1.查看命名空间

````shell
kubectl get ns
````

#### 删除 Namespace

1.删除指定 Namespace 命名空间

①方式一：

````shell
kubectl delete namespaces [Namespace NAME]

# 示例
kubectl delete namespaces kube-logs
````

②方式二：

````shell
kubectl delete ns [Namespace NAME]

# 示例
kubectl delete ns kube-logs
````

### 操作 StorageClass

#### 查看 StorageClass

1.查看命名空间

````shell
kubectl get sc
````

#### 删除 StorageClass

1.删除指定 StorageClass 命名空间

````shell
kubectl delete sc [StorageClass NAME]

# 示例
kubectl delete sc efk-nfs-client
````

### 操作 PersistentVolume

#### 查看 PersistentVolume

1.查看命名空间

````shell
kubectl get pv
````

#### 删除 PersistentVolume

1.删除指定 PersistentVolume 命名空间

````shell
kubectl delete pv [PersistentVolume NAME]

# 示例
kubectl delete pv efk-nfs-client-agkj9iow
````

## 高级命令

### 获取 token

1.获取 Master 节点的管理员 token

- 拥有 ClusterAdmin 的权限，可以执行所有操作

````shell
echo $(kubectl -n kube-system get secret $(kubectl -n kube-system get secret | grep kuboard-user | awk '{print $1}') -o go-template='{{.data.token}}' | base64 -d)
````

2.获取 Master 节点的只读 token

- view 可查看名称空间的内容
- system:node 可查看节点信息
- system:persistent-volume-provisioner 可查看存储类和存储卷声明的信息

````shell
echo $(kubectl -n kube-system get secret $(kubectl -n kube-system get secret | grep kuboard-viewer | awk '{print $1}') -o go-template='{{.data.token}}' | base64 -d)
````

### 创建和更新 K8S 对象

1.更新更新 k8s 对象，修改了 yaml 配置文件使其生效**（不推荐使用）**

````shell
kubectl replace -f [xxx.yaml 配置文件]

# 示例
kubectl replace -f nginx-deployment.yaml
````

2.创建或更新 k8s 对象，创建或修改了 yaml 配置文件使其生效**（官方推荐使用，便于版本管理）**

````shell
kubectl apply -f [xxx.yaml 本地或远程配置文件]

# 示例（本地）
kubectl apply -f nginx-deployment.yaml

# 示例（远程）
kubectl apply -f https://kuboard.cn/install-script/v1.16.2/nginx-ingress.yaml
````

## 搭建流程

### Kubeadm 搭建 K8S 单 Master 节点

可视化组件采用开源框架 Kuboard。

#### 环境要求

**1. 配置及要求**

| 配置项             | 要求                             |
| ------------------ | -------------------------------- |
| 服务器             | 2台及以上 2核4G内存 20G+固态更优 |
| 操作系统           | Cent OS 7.6 / 7.7 / 7.8          |
| Kubernetes 版本    | 1.18.x（当前为 1.18.8）          |
| Kuboard 版本       | 2.0.4.3                          |
| calico 版本        | 3.13.1                           |
| nginx-ingress 版本 | 1.5.5                            |
| Docker 版本        | 19.03.8                          |

**2. 环境检查**

1）检查 centos/hostname

````shell
# 在 master 节点和 worker 节点都要执行
cat /etc/redhat-release

# 此处 hostname 的输出将会是该机器在 Kubernetes 集群中的节点名字
# 不能使用 localhost 作为节点的名字，且不能包含下划线、小数点、大写字母
hostname

# 请使用 lscpu 命令，核对 CPU 信息
# Architecture: x86_64    本安装文档不支持 arm 架构
# CPU(s):       2         CPU 内核数量不能低于 2
lscpu
````

2）检查网络

所有节点上 Kubernetes 所使用的 IP 地址必须可以互通（无需 NAT 映射、无安全组或防火墙隔离），且为固定的内网 IP 地址。

````shell
# 查看服务器的默认网卡，通常是 eth0，如 default via 172.21.0.23 dev eth0
ip route show

# 显示默认网卡的 IP 地址，Kubernetes 将使用此 IP 地址与集群内的其他节点通信，如 172.17.216.80
ip addr
````

#### 配置环境

**所有服务器**都需要**执行**此配置环境的步骤。

**1. 修改 hostname**

````shell
# 修改 hostname（根据自己实际情况命名，如 master 节点命名为 master，worker 节点命名为 worker）
hostnamectl set-hostname master
# 查看修改结果
hostnamectl status
# 设置 hostname 解析
echo "127.0.0.1   $(hostname)" >> /etc/hosts
````

**2. 安装 docker、kubelete、kubeadm 和 kubectl**

方式一（推荐）：将下方 `#!/bin/bash` 开头的脚本保存为 install_env.sh 文件并执行命令即可。

````shell
# 创建执行脚本
vim install_env.sh

# 将下方脚本整个粘贴进来并保存，然后赋予其执行权限 
# 1.18.8 表示 K8S 的安装版本，可根据实际情况选择
chmod +x install_env.sh && ./install_env.sh 1.18.8
````

方式二：挨个命令执行。

````shell
#!/bin/bash

# 卸载旧版本
yum remove -y docker \
docker-client \
docker-client-latest \
docker-ce-cli \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-selinux \
docker-engine-selinux \
docker-engine

# 设置 yum repository
yum install -y yum-utils \
device-mapper-persistent-data \
lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装并启动 docker
yum install -y docker-ce-19.03.8 docker-ce-cli-19.03.8 containerd.io
systemctl enable docker
systemctl start docker

# 安装工具 vim
yum install -y vim

# 关闭 防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭 SeLinux
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

# 关闭 swap
swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak |grep -v swap > /etc/fstab

# 修改 /etc/sysctl.conf
# 如果有配置，则修改
sed -i "s#^net.ipv4.ip_forward.*#net.ipv4.ip_forward=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-ip6tables.*#net.bridge.bridge-nf-call-ip6tables=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-iptables.*#net.bridge.bridge-nf-call-iptables=1#g"  /etc/sysctl.conf
sed -i "s#^net.ipv6.conf.all.disable_ipv6.*#net.ipv6.conf.all.disable_ipv6=1#g"  /etc/sysctl.conf
sed -i "s#^net.ipv6.conf.default.disable_ipv6.*#net.ipv6.conf.default.disable_ipv6=1#g"  /etc/sysctl.conf
sed -i "s#^net.ipv6.conf.lo.disable_ipv6.*#net.ipv6.conf.lo.disable_ipv6=1#g"  /etc/sysctl.conf
sed -i "s#^net.ipv6.conf.all.forwarding.*#net.ipv6.conf.all.forwarding=1#g"  /etc/sysctl.conf
# 可能没有，追加
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.lo.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding = 1"  >> /etc/sysctl.conf
# 执行命令以应用
sysctl -p

# 配置 K8S 的 yum 源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 卸载 K8S 旧版本
yum remove -y kubelet kubeadm kubectl

# 安装kubelet、kubeadm、kubectl、kubernetes-cni（K8S 依赖）
# 将 ${1} 替换为 kubernetes 其他版本号，例如 1.18.8
yum install -y kubelet-${1} kubeadm-${1} kubectl-${1} kubernetes-cni-0.6.0

# 修改docker Cgroup Driver为systemd
# # 将/usr/lib/systemd/system/docker.service文件中的这一行 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
# # 修改为 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd
# 如果不修改，在添加 worker 节点时可能会碰到如下错误
# [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". 
# Please follow the guide at https://kubernetes.io/docs/setup/cri/
sed -i "s#^ExecStart=/usr/bin/dockerd.*#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd#g" /usr/lib/systemd/system/docker.service

# 配置 docker 的镜像源
cat <<EOF > /etc/docker/daemon.json
{
"registry-mirrors":["https://registry.docker-cn.com","https://pee6w651.mirror.aliyuncs.com","http://hub-mirror.c.163.com","https://docker.mirrors.ustc.edu.cn"]
}
EOF

# 重启 docker，并启动 kubelet
systemctl daemon-reload
systemctl restart docker
systemctl enable kubelet && systemctl start kubelet

docker version
````

#### 初始化 master 节点

只在 **master 节点执行**如下所有步骤。

**1. 设置环境变量**

- **APISERVER_NAME** 不能是 master 的 hostname
- **APISERVER_NAME** 必须全为小写字母、数字、小数点，不能包含减号
- **POD_SUBNET** 所使用的网段不能与 master节点/worker节点 所在的网段重叠。该字段的取值为一个 [CIDR](https://kuboard.cn/glossary/cidr.html) 值，如果您对 CIDR 这个概念还不熟悉，请仍然执行 export POD_SUBNET=10.100.0.1/16 命令，不做修改

````shell
# 替换 192.168.1.4 为 master 节点的内网 IP，根据实际情况填写
# export 命令只在当前 shell 会话中有效，开启新的 shell 窗口后，如果要继续安装过程，请重新执行此处的 export 命令
export MASTER_IP=192.168.1.4
# 替换 apiserver.demo 为自己的 dnsName
export APISERVER_NAME=apiserver.demo
# Kubernetes 容器组所在的网段，该网段安装完成后，由 kubernetes 创建，事先并不存在于您的物理网络中
export POD_SUBNET=10.100.0.1/16
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts
````

方式二：挨个命令执行。

**2. 初始化 master**

方式一（推荐）：将下方 `#!/bin/bash` 开头的脚本保存为 install_master.sh 文件并执行命令即可。

````shell
# 创建执行脚本
vim install_master.sh

# 将下方脚本整个粘贴进来并保存，然后赋予执行权限 
# 1.18.8 表示 K8S 的安装版本，可根据实际情况选择
chmod +x install_master.sh && ./install_master.sh 1.18.8
````

方式二：挨个命令执行。

````shell
#!/bin/bash

# 脚本出错时终止执行
set -e

if [ ${#POD_SUBNET} -eq 0 ] || [ ${#APISERVER_NAME} -eq 0 ]; then
  echo -e "\033[31;1m请确保您已经设置了环境变量 POD_SUBNET 和 APISERVER_NAME \033[0m"
  echo 当前POD_SUBNET=$POD_SUBNET
  echo 当前APISERVER_NAME=$APISERVER_NAME
  exit 1
fi

# 配置 K8S 的启动配置文件
# 查看完整配置选项 https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2
rm -f ./kubeadm-config.yaml
cat <<EOF > ./kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v${1}
imageRepository: registry.aliyuncs.com/k8sxio
controlPlaneEndpoint: "${APISERVER_NAME}:6443"
networking:
  serviceSubnet: "10.96.0.0/16"
  podSubnet: "${POD_SUBNET}"
  dnsDomain: "cluster.local"
EOF

# kubeadm init
# 根据您服务器网速的情况，您需要等候 3 - 10 分钟
kubeadm init --config=kubeadm-config.yaml --upload-certs

# 配置 kubectl
rm -rf /root/.kube/
mkdir /root/.kube/
cp -i /etc/kubernetes/admin.conf /root/.kube/config

# 安装 calico 网络插件
# 参考文档 https://docs.projectcalico.org/v3.13/getting-started/kubernetes/self-managed-onprem/onpremises
echo "安装calico-3.13.1"
rm -f calico-3.13.1.yaml
wget https://kuboard.cn/install-script/calico/calico-3.13.1.yaml
kubectl apply -f calico-3.13.1.yaml
````

**3. 查看初始化结果**

````shell
# 执行如下命令，等待 3-10 分钟，直到所有的容器组处于 Running 状态
watch kubectl get pod -n kube-system -o wide

# 查看 master 节点初始化结果
kubectl get nodes -o wide
````

#### 初始化 worker 节点

**1. 设置环境变量**

只在 **worker 节点执行**如下步骤。

- **APISERVER_NAME** 不能是 worker 的 hostname
- **APISERVER_NAME** 必须全为小写字母、数字、小数点，不能包含减号
- **POD_SUBNET** 所使用的网段不能与 master节点/worker节点 所在的网段重叠。该字段的取值为一个 [CIDR](https://kuboard.cn/glossary/cidr.html) 值，如果您对 CIDR 这个概念还不熟悉，请仍然执行 export POD_SUBNET=10.100.0.1/16 命令，不做修改

````shell
# 替换 192.168.1.5 为 worker 节点的内网 IP，根据实际情况填写
# export 命令只在当前 shell 会话中有效，开启新的 shell 窗口后，如果要继续安装过程，请重新执行此处的 export 命令
export MASTER_IP=192.168.1.5
# 替换 apiserver.demo 为自己的 dnsName
export APISERVER_NAME=apiserver.demo
# Kubernetes 容器组所在的网段，该网段安装完成后，由 kubernetes 创建，事先并不存在于您的物理网络中
export POD_SUBNET=10.100.0.1/16
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts
````

**2. 获取 master 节点的 join 命令参数**

此步骤在  **master 节点执行**。

````shell
# 创建 worker 节点的 join 命令
kubeadm token create --print-join-command

# kubeadm token create 命令的输出
kubeadm join apiserver.demo:6443 --token n1jsw9.0iqur09ootszdst7 \
    --discovery-token-ca-cert-hash sha256:e69288ca1cb8b40329b23e78dc6c8b4ec7e249cd0060f2ece27186b100866361
````

> 该 token 的有效时间为 2 个小时，2小时内，您可以使用此 token 初始化任意数量的 worker 节点。

**3. 初始化 worker**

只在 **worker 节点执行**如下步骤。

````shell
# 替换为 master 节点上 kubeadm token create 命令的输出（上个步骤中的 join 命令）
kubeadm join apiserver.demo:6443 --token n1jsw9.0iqur09ootszdst7 \
    --discovery-token-ca-cert-hash sha256:e69288ca1cb8b40329b23e78dc6c8b4ec7e249cd0060f2ece27186b100866361
````

**4. 查看初始化结果**

此步骤在  **master 节点执行**。

````shell
# 执行如下命令，等待 3-10 分钟，直到所有的容器组处于 Running 状态
watch kubectl get pod -n kube-system -o wide

# 查看 master 节点初始化结果
kubectl get nodes -o wide
````

#### 安装 Ingress Controller

在 **master 节点执行**。

````shell
kubectl apply -f https://kuboard.cn/install-script/v1.18.x/nginx-ingress.yaml
````

#### 安装 Kuboard

**1. 安装 Kuboard 可视化 UI**

````shell
# 安装稳定版 Kuboard
kubectl apply -f https://kuboard.cn/install-script/kuboard.yaml
kubectl apply -f https://addons.kuboard.cn/metrics-server/0.3.7/metrics-server.yaml
````

**2. 查看 Kuboard 运行状态**

````shell
kubectl get pods -l k8s.kuboard.cn/name=kuboard -n kube-system
````

**3. 获取 token**

查看本文档 `2 高级命令` 下的 `2.1 获取 token` 的获取 token 步骤。

**4. 访问 Kuboard web 界面**

Kuboard Service 使用了 NodePort 的方式暴露服务，NodePort 为 32567，访问地址为：

```
http://任意一个Worker节点的IP地址:32567/
```

输入上一步获取的 token 即可进入界面。

### Kubeadm 搭建 K8S 高可用集群

### K8S 搭建 nfs

### K8S 搭建 efk

## 卸载

### 一键卸载 Docker 及 K8S

````shell
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
docker rmi $(docker images -aq)
yum remove -y docker docker-client docker-client-latest docker-ce-cli docker-common docker-latest docker-latest-logrotate docker-logrotate docker-selinux docker-engine-selinux docker-engine
systemctl disable kubelet
yum remove -y kubelet kubeadm kubectl
echo 3 > /proc/sys/vm/drop_caches
ps -aux | grep kube
````

## yaml 配置文件详解

 yaml 格式的 pod 定义文件完整内容：

````yaml
apiVersion: v1       #必选，版本号，例如v1
kind: Pod       #必选，Pod
metadata:       #必选，元数据
  name: string       #必选，Pod名称
  namespace: string    #必选，Pod所属的命名空间
  labels:      #自定义标签
    - name: string     #自定义标签名字
  annotations:       #自定义注释列表
    - name: string
spec:         #必选，Pod中容器的详细定义
  replicas: int   #手动设置（初始）想要创建的副本数
  containers:      #必选，Pod中容器列表
  - name: string     #必选，容器名称
    image: string    #必选，容器的镜像名称
    imagePullPolicy: [Always | Never | IfNotPresent] #获取镜像的策略 Alawys表示下载镜像 IfnotPresent表示优先使用本地镜像，否则下载镜像，Nerver表示仅使用本地镜像
    command: [string]    #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]     #容器的启动命令参数列表
    workingDir: string     #容器的工作目录
    volumeMounts:    #挂载到容器内部的存储卷配置
    - name: string     #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string    #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean    #是否为只读模式
    ports:       #需要暴露的端口库号列表
    - name: string     #端口号名称
      containerPort: int   #容器需要监听的端口号
      hostPort: int    #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string     #端口协议，支持TCP和UDP，默认TCP
    env:       #容器运行前需设置的环境变量列表
    - name: string     #环境变量名称
      value: string    #环境变量的值
    resources:       #资源限制和请求的设置
      limits:      #资源限制的设置
        cpu: string    #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string     #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:      #资源请求的设置
        cpu: string    #Cpu请求，容器启动的初始可用数量
        memory: string     #内存清楚，容器启动的初始可用数量
    livenessProbe:     #对Pod内个容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
      exec:      #对Pod容器内检查方式设置为exec方式
        command: [string]  #exec方式需要制定的命令或脚本
      httpGet:       #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:     #对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0  #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0   #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged:false
    restartPolicy: [Always | Never | OnFailure] #Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod
    nodeSelector: obeject  #设置NodeSelector表示将该Pod调度到包含这个label的node上，以key：value的格式指定
    imagePullSecrets:    #Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string
    hostNetwork: false      #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    volumes:       #在该pod上定义共享存储卷列表
    - name: string     #共享存储卷名称 （volumes类型有很多种）
      emptyDir: {}     #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string     #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string     #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:      #类型为secret的存储卷，挂载集群与定义的secre对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:     #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string
          path: string
````

## 常见问题排查

# K8s 管理工具 kubectl 详解

## 一、陈述式管理

### 1. 陈述式资源管理方法

- kubernetes 集群管理集群资源的唯一入口是通过相应的方法调用 apiserver 的接口
- kubectl 是官方的 CLI 命令行工具，用于与 apiserver 进行通信，将用户在命令行输入的命令，组织并转化为 apiserver 能识别的信息，进而实现管理 k8s 各种资源的一种有效途径
- kubectl 的命令大全
  `kubectl --help`
  `k8s官方中文文档：`http://docs.kubernetes.org.cn/683.html
- 对资源的增、删、查操作比较容易，但对改的操作就不容易了

### 2. k8s相关信息查看

#### 2.1 查看版本信息

```
kubectl version
[root@master ~]# kubectl version
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.1", GitCommit:"4485c6f18cee9a5d3c3b4e523bd27972b1b53892", GitTreeState:"clean", BuildDate:"2019-07-18T09:18:22Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.1", GitCommit:"4485c6f18cee9a5d3c3b4e523bd27972b1b53892", GitTreeState:"clean", BuildDate:"2019-07-18T09:09:21Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
kubectl get nodes
[root@master ~]# kubectl get nodes 
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   20h   v1.15.1
node01   Ready    <none>   20h   v1.15.1
node02   Ready    <none>   20h   v1.15.1
```

#### 2.2 查看资源对象简写

kubectl api-resources

```
[root@master ~]# kubectl api-resources
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
bindings                                                                      true         Binding
componentstatuses                 cs                                          false        ComponentStatus
configmaps                        cm                                          true         ConfigMap
endpoints                         ep                                          true         Endpoints
events                            ev                                          true         Event
limitranges                       limits                                      true         LimitRange
namespaces                        ns                                          false        Namespace
nodes                             no                                          false        Node
persistentvolumeclaims            pvc                                         true         PersistentVolumeClaim
persistentvolumes                 pv                                          false        PersistentVolume
pods                              po                                          true         Pod
podtemplates                                                                  true         PodTemplate
replicationcontrollers            rc                                          true         ReplicationController
resourcequotas                    quota                                       true         ResourceQuota
secrets                                                                       true         Secret
serviceaccounts                   sa                                          true         ServiceAccount
services                          svc                                         true         Service
mutatingwebhookconfigurations                  admissionregistration.k8s.io   false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io   false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io           false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io         false        APIService
controllerrevisions                            apps                           true         ControllerRevision
daemonsets                        ds           apps                           true         DaemonSet
deployments                       deploy       apps                           true         Deployment
replicasets                       rs           apps                           true         ReplicaSet
statefulsets                      sts          apps                           true         StatefulSet
tokenreviews                                   authentication.k8s.io          false        TokenReview
localsubjectaccessreviews                      authorization.k8s.io           true         LocalSubjectAccessReview
selfsubjectaccessreviews                       authorization.k8s.io           false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io           false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io           false        SubjectAccessReview
horizontalpodautoscalers          hpa          autoscaling                    true         HorizontalPodAutoscaler
cronjobs                          cj           batch                          true         CronJob
jobs                                           batch                          true         Job
certificatesigningrequests        csr          certificates.k8s.io            false        CertificateSigningRequest
leases                                         coordination.k8s.io            true         Lease
events                            ev           events.k8s.io                  true         Event
daemonsets                        ds           extensions                     true         DaemonSet
deployments                       deploy       extensions                     true         Deployment
ingresses                         ing          extensions                     true         Ingress
networkpolicies                   netpol       extensions                     true         NetworkPolicy
podsecuritypolicies               psp          extensions                     false        PodSecurityPolicy
replicasets                       rs           extensions                     true         ReplicaSet
ingresses                         ing          networking.k8s.io              true         Ingress
networkpolicies                   netpol       networking.k8s.io              true         NetworkPolicy
runtimeclasses                                 node.k8s.io                    false        RuntimeClass
poddisruptionbudgets              pdb          policy                         true         PodDisruptionBudget
podsecuritypolicies               psp          policy                         false        PodSecurityPolicy
clusterrolebindings                            rbac.authorization.k8s.io      false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io      false        ClusterRole
rolebindings                                   rbac.authorization.k8s.io      true         RoleBinding
roles                                          rbac.authorization.k8s.io      true         Role
priorityclasses                   pc           scheduling.k8s.io              false        PriorityClass
csidrivers                                     storage.k8s.io                 false        CSIDriver
csinodes                                       storage.k8s.io                 false        CSINode
storageclasses                    sc           storage.k8s.io                 false        StorageClass
volumeattachments                              storage.k8s.io                 false        VolumeAttachment
```

#### 2.3 查看集群信息

kubectl cluster-info

```
[root@master ~]# kubectl cluster-info
Kubernetes master is running at https://192.168.122.10:6443
KubeDNS is running at https://192.168.122.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
 
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

#### 2.4 配置kubectl自动补全

source <(kubectl completion bash)

```
[root@master ~]# source <(kubectl completion bash)
```

可通过TAB键实现命令补全，建议将其写入etc/profile

#### 2.5 查看日志

journalctl -u kubelet -f

```
[root@master ~]# journalctl -u kubelet -f
-- Logs begin at 一 2021-11-01 18:58:09 CST. --
......
```

#### 2.6 基本信息查看

kubectl get [-o wide|json|yaml] [-n namespace]
获取资源的相关信息，-n指定命名空间，-o指定输出格式
resource可以是具体资源名称，如"pod nhinx-xxx"；也可以是资源类型，如“pod,node,svc,deploy”多种资源使用逗号间隔；或者all（仅展示几种核心资源，并不完整）
–all-namespaces或-A：表示显示所有命名空间
–show-labels：显示所有标签
-l app：仅显示标签为app的资源
-l app=nginx：仅显示包含app标签，且值为nginx的资源

##### 2.6.1 查看master节点状态

kubectl get componentstatuses
kubectl get cs

```
[root@master ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
[root@master ~]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
```

##### 2.6.2 查看命名空间

kubectl get namespace
kubectl get ns

```
[root@master ~]# kubectl get namespace
NAME              STATUS   AGE
default           Active   26h
kube-node-lease   Active   26h
kube-public       Active   26h
kube-system       Active   26h
[root@master ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   26h
kube-node-lease   Active   26h
kube-public       Active   26h
kube-system       Active   26h
```

#### 2.7 命名空间操作

##### 2.7.1 查看default命名空间的所有资源

kubectl get all [-n default]
由于deafult为缺省空间，当不指定命名空间时默认查看default命名空间

```bash
[root@node1 ~]# kubectl get all
NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.244.64.1   <none>        443/TCP   4h36m
```

##### 2.7.2 创建命名空间app

kubectl create ns app

```bash
[root@node1 ~]# kubectl create ns app
namespace/app created
[root@node1 ~]# kubectl get ns
NAME                   STATUS   AGE
app                    Active   3s
default                Active   4h37m
ingress-controller     Active   4h34m
kube-node-lease        Active   4h37m
kube-public            Active   4h37m
kube-system            Active   4h37m
kubernetes-dashboard   Active   4h33m
```

##### 2.7.3 删除命名空间app

kubectl delete ns app

```bash
[root@node1 ~]# kubectl delete ns app
namespace "app" deleted
[root@node1 ~]# kubectl get ns
NAME                   STATUS   AGE
default                Active   4h37m
ingress-controller     Active   4h35m
kube-node-lease        Active   4h37m
kube-public            Active   4h37m
kube-system            Active   4h37m
kubernetes-dashboard   Active   4h33m
```

#### 2.8 deployment/pod操作

##### 2.8.1 在命名空间kube-public创建副本控制器（deployment）来启动Pod（nginx-test）

kubectl create deployment nginx-test --image=nginx -n kube-public

```bash
[root@node1 ~]# kubectl get pod -n kube-public
No resources found in kube-public namespace.
[root@node1 ~]# kubectl create deployment nginx-test --image=nginx -n kube-public 
deployment.apps/nginx-test created
[root@node1 ~]# kubectl get deploy -n kube-public
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
nginx-test   0/1     1            0           23s
[root@node1 ~]# kubectl get deploy -n kube-public
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
nginx-test   1/1     1            1           2m24s
[root@node1 ~]# kubectl get pod -n kube-public
NAME                          READY   STATUS    RESTARTS   AGE
nginx-test-795d659f45-n4nks   1/1     Running   0          2m58s
```

##### 2.8.2 描述某个资源的详细信息

kubectl describe deployment nginx-test -n kube-public

```bash
[root@node1 ~]# kubectl describe deployment nginx-test -n kube-public 
Name:                   nginx-test
Namespace:              kube-public
CreationTimestamp:      Thu, 16 Dec 2021 20:35:23 +0800
Labels:                 app=nginx-test
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx-test
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx-test
  Containers:
   nginx:
    Image:        nginx
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-test-795d659f45 (1/1 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  4m26s  deployment-controller  Scaled up replica set nginx-test-795d659f45 to 1
```

kubectl describe pod nginx-test -n kube-public

```bash
[root@node1 ~]# kubectl describe pod nginx-test -n kube-public
......
```

##### 2.8.3 查看命名空间kube-public中pod信息

kubectl get pods -n kube-public

```bash
[root@node1 ~]# kubectl get pod -n kube-public 
NAME                          READY   STATUS    RESTARTS   AGE
nginx-test-795d659f45-n4nks   1/1     Running   0          6m4s
```

##### 2.8.4 登录容器

kubectl exec 可以跨主机登录容器，docker exec 只能在容器所在主机登录

```bash
[root@node1 ~]# kubectl exec -it nginx-test-795d659f45-n4nks bash -n kube-public 
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@nginx-test-795d659f45-n4nks:/# ls
bin   dev		   docker-entrypoint.sh  home  lib64  mnt  proc  run   srv  tmp  var
boot  docker-entrypoint.d  etc			 lib   media  opt  root  sbin  sys  usr
```

##### 2.8.5 删除（重启）pod资源

由于存在 deployment/rc 之类的副本控制器，删除 pod 也会重新拉起来

```bash
[root@node1 ~]# kubectl get pod -n kube-public 
NAME                          READY   STATUS    RESTARTS   AGE
nginx-test-795d659f45-n4nks   1/1     Running   0          12m
[root@node1 ~]# kubectl delete pod nginx-test-795d659f45-n4nks -n kube-public 
pod "nginx-test-795d659f45-n4nks" deleted
[root@node1 ~]# kubectl get pod -n kube-public 
NAME                          READY   STATUS    RESTARTS   AGE
nginx-test-795d659f45-65pwr   1/1     Running   0          11s
```

##### 2.8.6 若无法删除，总是处于terminate状态，则要强行删除pod

kubectl delete pod [] -n [] --force --grace-period=0
`grace-period表示过渡存活期，默认30s，在删除pod之前允许pod慢慢终止其上的容器进程，从而优雅的退出，0表示立即终止pod`

```bash
[root@node1 ~]# kubectl get pod -n kube-public 
NAME                          READY   STATUS    RESTARTS   AGE
nginx-test-795d659f45-r2jwz   1/1     Running   0          13s
[root@node1 ~]# kubectl delete pod nginx-test-795d659f45-r2jwz -n kube-public --force --grace-period=0
warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "nginx-test-795d659f45-r2jwz" force deleted
[root@node1 ~]# kubectl get pod -n kube-public 
NAME                          READY   STATUS              RESTARTS   AGE
nginx-test-795d659f45-6h9kj   0/1     ContainerCreating   0          9s
```

##### 2.8.7 扩缩容

###### 2.8.7.1 扩容

kubectl scale deployment nginx-test --replicas=3 -n kube-public

```bash
[root@node1 ~]# kubectl scale deployment nginx-test --replicas=3 -n kube-public
deployment.apps/nginx-test scaled
[root@node1 ~]# kubectl get pod -n kube-public 
NAME                          READY   STATUS              RESTARTS   AGE
nginx-test-795d659f45-6h9kj   1/1     Running             0          7m
nginx-test-795d659f45-gl6z2   0/1     ContainerCreating   0          101s
nginx-test-795d659f45-p2q9s   0/1     ContainerCreating   0          101s
```

###### 2.8.7.2 缩容

kubectl scale deployment nginx-test --replicas=1 -n kube-public

```bash
[root@node1 ~]# kubectl scale deployment nginx-test --replicas=1 -n kube-public 
deployment.apps/nginx-test scaled
[root@node1 ~]# kubectl get pod -n kube-public 
NAME                          READY   STATUS        RESTARTS   AGE
nginx-test-795d659f45-6h9kj   1/1     Running       0          8m4s
nginx-test-795d659f45-gl6z2   0/1     Terminating   0          2m45s
nginx-test-795d659f45-p2q9s   0/1     Terminating   0          2m45s
[root@node1 ~]# kubectl get pod -n kube-public 
NAME                          READY   STATUS    RESTARTS   AGE
nginx-test-795d659f45-6h9kj   1/1     Running   0          8m19s
```

##### 2.8.8 删除副本控制器

kubectl delete deployment nginx-test -n kube-public
kubectl delete deployment/nginx-test -n kube-public

```bash
[root@node1 ~]# kubectl delete deployment nginx-test -n kube-public
deployment.apps "nginx-test" deleted
[root@node1 ~]# kubectl get pod -n kube-public 
NAME                          READY   STATUS        RESTARTS   AGE
nginx-test-795d659f45-6h9kj   0/1     Terminating   0          8m46s
[root@node1 ~]# kubectl get pod -n kube-public 
No resources found in kube-public namespace.
```

#### 2.9 增加/删除label

##### 2.9.1 增加label

kubectl label deploy nginx-test version=nginx1.14

```bash
[root@master ~]# kubectl get deploy --show-labels
NAME         READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
nginx-test   3/3     3            3           19m   run=nginx-test
[root@master ~]# kubectl label deploy nginx-test version=nginx1.14
deployment.extensions/nginx-test labeled
[root@master ~]# kubectl get deploy --show-labels
NAME         READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
nginx-test   3/3     3            3           19m   run=nginx-test,version=nginx1.14
```

##### 2.9.2 删除label

kubectl label deploy nginx-test version-

```bash
[root@master ~]# kubectl get deploy --show-labels
NAME         READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
nginx-test   3/3     3            3           19m   run=nginx-test,version=nginx1.14
[root@master ~]# kubectl label deploy nginx-test version-
deployment.extensions/nginx-test labeled
[root@master ~]# kubectl get deploy --show-labels
NAME         READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
nginx-test   3/3     3            3           20m   run=nginx-test
```

### 3. K8S模拟项目

#### 3.1 项目的生命周期

创建–>发布–>更新–>回滚–>删除

#### 3.2 创建kubectl run命令

● 创建并运行一个或多个容器镜像
● 创建一个deployment或job来管理容器
kubectl run --help查看使用帮助

启动nginx实例，暴露容器端口80，设置副本数3
kubectl run nginx --image=nginx:1.14 --port=80 --replicas=3

```bash
[root@master ~]# kubectl run nginx --image=nginx:1.14 --port=80 --replicas=3
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx created
```

kubectl get pods

```bash
[root@master ~]# kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-65fc77987d-cwvwl   1/1     Running   0          7s
nginx-65fc77987d-m7cnn   1/1     Running   0          7s
nginx-65fc77987d-z7hvx   1/1     Running   0          7s
```

kubectl get all

```bash
[root@master ~]# kubectl get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-65fc77987d-cwvwl   1/1     Running   0          24s
pod/nginx-65fc77987d-m7cnn   1/1     Running   0          24s
pod/nginx-65fc77987d-z7hvx   1/1     Running   0          24s


NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.1.0.1     <none>        443/TCP   82s


NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   3/3     3            3           24s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-65fc77987d   3         3         3       24s
```

#### 3.3 发布kubectl expose命令

● 将资源暴露为新的Service
kubectl expose --help查看使用帮助

为Deployment的nginx创建Service，并通过Service的80端口转发至容器的80端口上，Service的名称为nginx-service，类型为NodePort
kubectl expose deployment nginx --port=80 --target-port=80 --name=nginx-service --type=NodePort

```bash
[root@master ~]# kubectl expose deployment nginx --port=80 --target-port=80 --name=nginx-service --type=NodePort
service/nginx-service exposed
```

##### 3.3.1 Service的作用

Kubernetes之所以需要Service，一方面是因为Pod的IP不是固定的（Pod可能会重建），另一方面是因为一组Pod实例之间总会有负载均衡的需求。
Service通过Label Selector实现的对一组的Pod的访问。
对于容器应用而言，Kubernetes提供了基于VIP（虚拟IP）的网桥的方式访问Service，再由Service重定向到相应的Pod。

##### 3.3.2 Service的类型

![img](https://img-blog.csdnimg.cn/img_convert/ccc985898fc438683c74d6d1cd2b5544.png)
● ClusterIP:提供一个集群内部的虚拟IP以供Pod访问（Service默认类型）
● NodePort:在每个Node上打开一个端口以供外部访问，Kubernetes将会在每个Node上打开一个端口并且每个Node的端口都是一样的，通过NodeIP:NodePort的方式
● LoadBalancer:通过外部的负载均衡器来访问，通常在云平台部署LoadBalancer还需要额外的费用。

##### 3.3.3 查看Pod网络状态详细信息和Service暴露的端口

kubectl get pods,svc -o wide

```bash
[root@master ~]# kubectl get pods,svc -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
pod/nginx-65fc77987d-cwvwl   1/1     Running   0          61s   10.244.1.25   node01   <none>           <none>
pod/nginx-65fc77987d-m7cnn   1/1     Running   0          61s   10.244.1.24   node01   <none>           <none>
pod/nginx-65fc77987d-z7hvx   1/1     Running   0          61s   10.244.2.15   node02   <none>           <none>

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE    SELECTOR
service/kubernetes      ClusterIP   10.1.0.1       <none>        443/TCP        119s   <none>
service/nginx-service   NodePort    10.1.155.154   <none>        80:32107/TCP   15s    run=nginx
```

##### 3.3.4 查看关联后端的节点

kubectl get endpoints

```bash
[root@master ~]# kubectl get endpoints
NAME            ENDPOINTS                                      AGE
kubernetes      192.168.122.10:6443                            15m
nginx-service   10.244.1.24:80,10.244.1.25:80,10.244.2.15:80   13m
```

##### 3.3.5 查看service的描述信息

kubectl describe svc nginx

```bash
[root@master ~]# kubectl describe svc nginx
Name:                     nginx-service
Namespace:                default
Labels:                   run=nginx
Annotations:              <none>
Selector:                 run=nginx
Type:                     NodePort
IP:                       10.1.155.154
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  32107/TCP
Endpoints:                10.244.1.24:80,10.244.1.25:80,10.244.2.15:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

##### 3.3.6 查看负载均衡端口

在node01节点上操作

```bash
[root@node01 ~]# yum install -y ipvsadm
[root@node01 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
......      
TCP  192.168.122.11:32107 rr
#外部访问的IP和端口
  -> 10.244.1.24:80               Masq    1      0          0         
  -> 10.244.1.25:80               Masq    1      0          0         
  -> 10.244.2.15:80               Masq    1      0          0         
......        
TCP  10.1.155.154:80 rr
#pod集群组内部访问的IP和端口
  -> 10.244.1.24:80               Masq    1      0          0         
  -> 10.244.1.25:80               Masq    1      0          0         
  -> 10.244.2.15:80               Masq    1      0          0                
......     
```

在node02节点上操作

```bash
[root@node02 ~]# yum install -y ipvsadm
[root@node02 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
......
TCP  192.168.122.12:32107 rr
#外部访问的IP和端口
  -> 10.244.1.24:80               Masq    1      0          0         
  -> 10.244.1.25:80               Masq    1      0          0         
  -> 10.244.2.15:80               Masq    1      0          0         
......
TCP  10.1.155.154:80 rr
#pod集群组内部访问的IP和端口
  -> 10.244.1.24:80               Masq    1      0          0         
  -> 10.244.1.25:80               Masq    1      0          0         
  -> 10.244.2.15:80               Masq    1      0          0         
......
```

##### 3.3.7 访问查看

curl 10.1.155.154

```bash
[root@master ~]# curl 10.1.155.154
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

curl 192.168.122.11:32107

```bash
[root@master ~]# curl 192.168.122.11:32107
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

##### 3.3.8 查看访问日志

```bash
[root@master ~]# kubectl logs nginx-65fc77987d-cwvwl
10.244.0.0 - - [02/Nov/2021:12:58:38 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
[root@master ~]# kubectl logs nginx-65fc77987d-m7cnn
10.244.1.1 - - [02/Nov/2021:13:00:06 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
[root@master ~]# kubectl logs nginx-65fc77987d-z7hvx
10.244.2.1 - - [02/Nov/2021:13:00:10 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
```

#### 3.4 更新kubectl set

● 更改现有应用资源一些信息。
kubectl set --help查看使用帮助

##### 3.4.1 获取修改模板

kubectl set image --help获取

```bash
[root@master ~]# kubectl set image --help
......
Examples:
  # Set a deployment's nginx container image to 'nginx:1.9.1', and its busybox container image to 'busybox'.
  kubectl set image deployment/nginx busybox=busybox nginx=nginx:1.9.1
......
```

##### 3.4.2 查看当前nginx的版本号

```bash
[root@master ~]# curl -I 192.168.122.11:32107
#通过报文头查看版本号
HTTP/1.1 200 OK
Server: nginx/1.14.2
#nginx版本号为1.14.2
Date: Tue, 02 Nov 2021 13:10:49 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes
```

##### 3.4.3 将nginx版本更新为1.15

kubectl set image deployment/nginx nginx=nginx:1.15

```bash
[root@master ~]# kubectl set image deployment/nginx nginx=nginx:1.15
deployment.extensions/nginx image updated
```

##### 3.4.4 监听pod状态

处于动态监听pod状态，由于使用的是滚动更新方式，所以会先生成一个新的pod，然后删除一个旧的pod，往后以此类推
kubectl get pods -w

```bash
[root@master ~]# kubectl get pods -w
NAME                     READY   STATUS              RESTARTS   AGE
nginx-65fc77987d-cwvwl   1/1     Running             0          62m
nginx-65fc77987d-m7cnn   1/1     Running             0          62m
nginx-65fc77987d-z7hvx   1/1     Running             0          62m
nginx-6cbd4b987c-65qtx   0/1     ContainerCreating   0          23s
#新建第一个pod
nginx-6cbd4b987c-65qtx   1/1     Running             0          24s
nginx-65fc77987d-cwvwl   1/1     Terminating         0          63m
#第一个新pod运行后，删除一个旧pod
nginx-6cbd4b987c-27qz7   0/1     Pending             0          0s
nginx-6cbd4b987c-27qz7   0/1     Pending             0          0s
nginx-6cbd4b987c-27qz7   0/1     ContainerCreating   0          0s
#新建第二个pod
nginx-65fc77987d-cwvwl   0/1     Terminating         0          63m
nginx-65fc77987d-cwvwl   0/1     Terminating         0          63m
nginx-65fc77987d-cwvwl   0/1     Terminating         0          63m
nginx-6cbd4b987c-27qz7   1/1     Running             0          8s
nginx-65fc77987d-m7cnn   1/1     Terminating         0          63m
#第二个新pod运行后，删除第二个旧pod
nginx-6cbd4b987c-m467f   0/1     Pending             0          0s
nginx-6cbd4b987c-m467f   0/1     Pending             0          0s
nginx-6cbd4b987c-m467f   0/1     ContainerCreating   0          0s
#新建第三个pod
nginx-65fc77987d-m7cnn   0/1     Terminating         0          63m
nginx-6cbd4b987c-m467f   1/1     Running             0          2s
nginx-65fc77987d-z7hvx   1/1     Terminating         0          63m
#第三个新pod运行后，删除第三个旧pod
nginx-65fc77987d-z7hvx   0/1     Terminating         0          63m
nginx-65fc77987d-m7cnn   0/1     Terminating         0          63m
nginx-65fc77987d-m7cnn   0/1     Terminating         0          63m
nginx-65fc77987d-z7hvx   0/1     Terminating         0          63m
nginx-65fc77987d-z7hvx   0/1     Terminating         0          63m
```

更新规则可通过“kubetl describe deployment nginx”的“RollingUpdateStrategy”查看，默认配置为“25% max unavailable, 25% max surge”，即按照25%的比例进行滚动更新。

##### 3.4.5 查看pod的ip变化

kubectl get pod -o wide

```bash
[root@master ~]# kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
nginx-6cbd4b987c-27qz7   1/1     Running   0          6m19s   10.244.2.16   node02   <none>           <none>
nginx-6cbd4b987c-65qtx   1/1     Running   0          6m43s   10.244.1.26   node01   <none>           <none>
nginx-6cbd4b987c-m467f   1/1     Running   0          6m11s   10.244.1.27   node01   <none>           <none>
```

pod更新后，ip改变

##### 3.4.6 重新查看nginx版本号

```bash
[root@master ~]# curl -I 192.168.122.11:32107
HTTP/1.1 200 OK
Server: nginx/1.15.12
#nginx版本更新为1.15.12
Date: Tue, 02 Nov 2021 13:22:29 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 16 Apr 2019 13:08:19 GMT
Connection: keep-alive
ETag: "5cb5d3c3-264"
Accept-Ranges: bytes
```

#### 3.5 回滚kubectl rollout

● 对资源进行回滚管理
kubectl rollout --help查看使用帮助

##### 3.5.1 查看历史版本

kubectl rollout history deployment/nginx

```bash
[root@master ~]# kubectl rollout history deployment/nginx
deployment.extensions/nginx 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

##### 3.5.2 执行回滚到上一个版本

kubectl rollout undo deployment/nginx

```bash
[root@master ~]# kubectl rollout undo deployment/nginx
deployment.extensions/nginx rolled back
```

查看pod的ip变化

```bash
[root@master ~]# kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
#回滚后ip再次改变
nginx-65fc77987d-2525z   1/1     Running   0          42s   10.244.2.17   node02   <none>           <none>
nginx-65fc77987d-9qkzp   1/1     Running   0          40s   10.244.2.18   node02   <none>           <none>
nginx-65fc77987d-qg75q   1/1     Running   0          41s   10.244.1.28   node01   <none>           <none>
```

查看当前nginx版本

```bash
[root@master ~]# curl -I 192.168.122.11:32107
HTTP/1.1 200 OK
Server: nginx/1.14.2
#nginx版本回到1.14.2
Date: Tue, 02 Nov 2021 13:31:35 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes
```

##### 3.5.3 执行回滚到指定版本

查看历史版本

```bash
[root@master ~]# kubectl rollout history deployment/nginx
deployment.extensions/nginx 
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

回到revison2，即1.15版本
kubectl rollout undo deployment/nginx --to-revision=2

```bash
[root@master ~]# kubectl rollout undo deployment/nginx --to-revision=2
deployment.extensions/nginx rolled back
```

查看pod的ip变化

```bash
[root@master ~]# kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
#回滚后ip再次改变
nginx-6cbd4b987c-mhqbc   1/1     Running   0          37s   10.244.1.29   node01   <none>           <none>
nginx-6cbd4b987c-tf462   1/1     Running   0          36s   10.244.2.19   node02   <none>           <none>
nginx-6cbd4b987c-z86d6   1/1     Running   0          35s   10.244.1.30   node01   <none>           <none>
```

查看当前nginx版本

```bash
[root@master ~]# curl -I 192.168.122.11:32107
HTTP/1.1 200 OK
Server: nginx/1.15.12
#nginx版本回到1.15.12
Date: Tue, 02 Nov 2021 13:36:42 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 16 Apr 2019 13:08:19 GMT
Connection: keep-alive
ETag: "5cb5d3c3-264"
Accept-Ranges: bytes
```

##### 3.5.4 检查回滚状态

kubectl rollout status deployment/nginx

```bash
[root@master ~]# kubectl rollout status deployment/nginx
deployment "nginx" successfully rolled out
```

#### 3.6 删除kubectl delete

##### 3.6.1 删除副本控制器

kubectl delete deployment/nginx

```bash
[root@master ~]# kubectl get deploy
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           88m
[root@master ~]# kubectl delete deployment/nginx
deployment.extensions "nginx" deleted
[root@master ~]# kubectl get deploy
No resources found.
```

##### 3.6.2 删除service

kubectl delete svc/nginx-service

```bash
[root@master ~]# kubectl get svc
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.1.0.1       <none>        443/TCP        91m
nginx-service   NodePort    10.1.155.154   <none>        80:32107/TCP   89m
[root@master ~]# kubectl delete svc/nginx-service
service "nginx-service" deleted
[root@master ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.1.0.1     <none>        443/TCP   91m
[root@master ~]# kubectl get all


NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.1.0.1     <none>        443/TCP   92m
```

### 4. 金丝雀发布/灰度发布（Canary Release）

#### 4.1 金丝雀发布简介

Deployment控制器支持自定义控制更新过程中的滚动节奏，如“暂停（pause）”或“继续（resume）”更新操作。比如等待第一批新的Pod资源创建完成后立即暂停更新过程，此时，仅存在一部分新版本的应用，主体部分还是旧的版本。然后，在筛选一小部分的用户请求路由到新版本的Pod应用，继续观察能否稳定地按期望的方式运行。确定没问题之后再继续完成余下的Pod资源滚动更新，否则立即回滚更新操作。这就是所谓的金丝雀发布。

#### 4.2 更新deployment的版本，并配置暂停deployment

##### 4.2.1 创建pods

kubectl run nginx-test --image=nginx:1.14 --replicas=3

```bash
[root@master ~]# kubectl run nginx-test --image=nginx:1.14 --replicas=3
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx-test created
[root@master ~]# kubectl get pods,deploy -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
pod/nginx-test-6cc7cd5547-4xqcb   1/1     Running   0          51s   10.244.2.23   node02   <none>           <none>
pod/nginx-test-6cc7cd5547-gcbh5   1/1     Running   0          51s   10.244.1.37   node01   <none>           <none>
pod/nginx-test-6cc7cd5547-nw6k6   1/1     Running   0          51s   10.244.1.38   node01   <none>           <none>

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES       SELECTOR
deployment.extensions/nginx-test   3/3     3            3           51s   nginx-test   nginx:1.14   run=nginx-test
```

##### 4.2.2 发布服务

kubectl expose deploy nginx-test --port=80 --target-port=80 --name=nginx-service --type=NodePort

```bash
[root@master ~]# kubectl expose deploy nginx-test --port=80 --target-port=80 --name=nginx-service --type=NodePort
service/nginx-service exposed
[root@master ~]# kubectl get svc -o wide
NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE   SELECTOR
kubernetes      ClusterIP   10.1.0.1      <none>        443/TCP        23h   <none>
nginx-service   NodePort    10.1.36.134   <none>        80:30191/TCP   9s    run=nginx-test
```

##### 4.2.3 查看nginx版本

```bash
[root@master ~]# curl -I 192.168.122.11:30191
HTTP/1.1 200 OK
Server: nginx/1.14.2
#版本号为1.14.2
Date: Wed, 03 Nov 2021 11:54:11 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes
```

##### 4.2.4 定义版本CHANGE-CAUSE

###### 4.2.4.1 查看历史版本

在不定义CHANGE-CAUSE的情况下，缺省值为，当历史版本较多时，不便于咱们回滚时辨认版本号。因此，建议定义CHANGE-CAUSE为服务版本以帮助咱们辨认当前服务。
kubectl rollout history deploy/nginx-test

```bash
[root@master ~]# kubectl rollout history deploy/nginx-test
deployment.extensions/nginx-test 
REVISION  CHANGE-CAUSE
1         <none>
```

###### 4.2.4.2 定义版本CHANGE-CAUSE

一般通过修改配置的方式定义change-cause

```bash
[root@master ~]# kubectl edit deploy/nginx-test

......
kind: Deployment
metadata:
  annotations:
#下行可定义历史版本revision
    deployment.kubernetes.io/revision: "1"
#在Deployment的matadata项下的annotations中如下行定义change-cause
    kubernetes.io/change-cause: "nginx1.14"
......
```

###### 4.2.4.3 再次查看历史版本

```bash
[root@master ~]# kubectl rollout history deploy/nginx-test
deployment.extensions/nginx-test 
REVISION  CHANGE-CAUSE
1         nginx1.14
```

##### 4.2.5 更新nginx版本为1.15并配置暂停

kubectl set image deploy/nginx-test nginx-test=nginx:1.15 && kubectl rollout pause deploy/nginx-test

```bash
[root@master ~]# kubectl set image deploy/nginx-test nginx-test=nginx:1.15 && kubectl rollout pause deploy/nginx-test
deployment.extensions/nginx-test image updated
deployment.extensions/nginx-test paused
```

##### 4.2.6 观察更新状态

kubectl rollout status deploy/nginx-test

```bash
[root@master ~]# kubectl rollout status deploy/nginx-test
Waiting for deployment "nginx-test" rollout to finish: 1 out of 3 new replicas have been updated...
```

##### 4.2.7 监控更新的过程

可以看到已经新增了一个pod，但是并未按照预期的状态去删除一个旧的资源，就是因为使用了pause暂停命令
kubectl get pods -w

```bash
[root@master ~]# kubectl get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
nginx-test-679dcbd68d-zw8w5   1/1     Running   0          8m28s   10.244.2.24   node02   <none>           <none>
nginx-test-6cc7cd5547-4xqcb   1/1     Running   0          48m     10.244.2.23   node02   <none>           <none>
nginx-test-6cc7cd5547-gcbh5   1/1     Running   0          48m     10.244.1.37   node01   <none>           <none>
nginx-test-6cc7cd5547-nw6k6   1/1     Running   0          48m     10.244.1.38   node01   <none>           <none>
```

##### 4.2.8 查看nginx版本

kubectl get pod -o wide

```bash
[root@master ~]# kubectl get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
nginx-test-679dcbd68d-zw8w5   1/1     Running   0          8m28s   10.244.2.24   node02   <none>           <none>
nginx-test-6cc7cd5547-4xqcb   1/1     Running   0          48m     10.244.2.23   node02   <none>           <none>
nginx-test-6cc7cd5547-gcbh5   1/1     Running   0          48m     10.244.1.37   node01   <none>           <none>
nginx-test-6cc7cd5547-nw6k6   1/1     Running   0          48m     10.244.1.38   node01   <none>           <none>
```

查看nginx版本

```bash
[root@master ~]# curl -I 192.168.122.12:30191
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Wed, 03 Nov 2021 12:45:37 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes

[root@master ~]# curl -I 192.168.122.12:30191
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Wed, 03 Nov 2021 12:46:59 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes

[root@master ~]# curl -I 192.168.122.12:30191
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Wed, 03 Nov 2021 12:47:09 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes

[root@master ~]# curl -I 192.168.122.12:30191
HTTP/1.1 200 OK
Server: nginx/1.15.12
#新pod为1.15
Date: Wed, 03 Nov 2021 12:47:12 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 16 Apr 2019 13:08:19 GMT
Connection: keep-alive
ETag: "5cb5d3c3-264"
Accept-Ranges: bytes
```

##### 4.2.9 查看并更新历史版本change-cause

kubectl rollout history deploy/nginx-test

```bash
[root@master ~]# kubectl rollout history deploy/nginx-test
deployment.extensions/nginx-test 
REVISION  CHANGE-CAUSE
1         nginx1.14
2         nginx1.14
```

kubectl edit deploy/nginx-test

```bash
[root@master ~]# kubectl edit deploy/nginx-test

kind: Deployment
metadata:
  annotations:
#下行的revison自动更新为2
    deployment.kubernetes.io/revision: "2"
#修改下行的change-cause为nginx1.15
    kubernetes.io/change-cause: nginx1.15
```

kubectl rollout history deploy/nginx-test

```bash
[root@master ~]# kubectl rollout history deploy/nginx-test
deployment.extensions/nginx-test 
REVISION  CHANGE-CAUSE
1         nginx1.14
2         nginx1.15
```

##### 4.2.10 resume继续更新

测试新版本没问题继续更新
kubectl rollout resume deploy/nginx-test

```bash
[root@master ~]# kubectl rollout resume deploy/nginx-test
deployment.extensions/nginx-test resumed
```

##### 4.2.11 查看最后的更新情况

```bash
[root@master ~]# kubectl get pods -w
NAME                          READY   STATUS    RESTARTS   AGE
nginx-test-679dcbd68d-zw8w5   1/1     Running   0          61s
nginx-test-6cc7cd5547-4xqcb   1/1     Running   0          41m
nginx-test-6cc7cd5547-gcbh5   1/1     Running   0          41m
nginx-test-6cc7cd5547-nw6k6   1/1     Running   0          41m
nginx-test-6cc7cd5547-gcbh5   1/1     Terminating   0          63m
nginx-test-679dcbd68d-cljzh   0/1     Pending       0          0s
nginx-test-679dcbd68d-cljzh   0/1     Pending       0          0s
nginx-test-679dcbd68d-cljzh   0/1     ContainerCreating   0          0s
nginx-test-6cc7cd5547-gcbh5   0/1     Terminating         0          63m
nginx-test-6cc7cd5547-gcbh5   0/1     Terminating         0          63m
nginx-test-6cc7cd5547-gcbh5   0/1     Terminating         0          63m
nginx-test-679dcbd68d-cljzh   1/1     Running             0          1s
nginx-test-6cc7cd5547-nw6k6   1/1     Terminating         0          63m
nginx-test-679dcbd68d-s2gck   0/1     Pending             0          0s
nginx-test-679dcbd68d-s2gck   0/1     Pending             0          0s
nginx-test-679dcbd68d-s2gck   0/1     ContainerCreating   0          0s
nginx-test-679dcbd68d-s2gck   1/1     Running             0          1s
nginx-test-6cc7cd5547-4xqcb   1/1     Terminating         0          63m
nginx-test-6cc7cd5547-nw6k6   0/1     Terminating         0          63m
nginx-test-6cc7cd5547-4xqcb   0/1     Terminating         0          63m
nginx-test-6cc7cd5547-nw6k6   0/1     Terminating         0          63m
nginx-test-6cc7cd5547-nw6k6   0/1     Terminating         0          63m
nginx-test-6cc7cd5547-4xqcb   0/1     Terminating         0          63m
nginx-test-6cc7cd5547-4xqcb   0/1     Terminating         0          63m
```

## 二、声明式管理

### 1. 声明式管理方法

1. 适合于对资源的修改操作
2. 声明式资源管理方法依赖于资源配置清明文件对资源进行管理
   资源配置清单文件有两种格式：yaml（人性化，易读），json（易于api接口解析）
3. 对资源的观念里，是通过实现定义在同一资源配置清单内，再通过陈述式命令应用到k8s集群里
4. 语法格式：kubectl create/apply/delete -f -o yaml

### 2. 查看资源配置清单

kubectl get deploy/nginx-test -o yaml

```bash
[root@master ~]# kubectl get deploy/nginx-test -o yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "3"
    kubernetes.io/change-cause: nginx1.14
  creationTimestamp: "2021-11-03T11:47:56Z"
  generation: 7
  labels:
    run: nginx-test
  name: nginx-test
  namespace: default
  resourceVersion: "105895"
  selfLink: /apis/extensions/v1beta1/namespaces/default/deployments/nginx-test
  uid: 634d471e-3907-4717-a02e-f5cce101d2f4
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      run: nginx-test
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: nginx-test
    spec:
      containers:
      - image: nginx:1.14
        imagePullPolicy: IfNotPresent
        name: nginx-test
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 3
  conditions:
  - lastTransitionTime: "2021-11-03T11:47:58Z"
    lastUpdateTime: "2021-11-03T11:47:58Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2021-11-03T12:50:56Z"
    lastUpdateTime: "2021-11-03T12:57:35Z"
    message: ReplicaSet "nginx-test-6cc7cd5547" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 7
  readyReplicas: 3
  replicas: 3
  updatedReplicas: 3
```

kubectl get service nginx-service -o yaml

```bash
[root@master ~]# kubectl get service nginx-service -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2021-11-03T11:52:40Z"
  labels:
    run: nginx-test
  name: nginx-service
  namespace: default
  resourceVersion: "100051"
  selfLink: /api/v1/namespaces/default/services/nginx-service
  uid: b2ec9834-864a-4146-9296-5420ac15451d
spec:
  clusterIP: 10.1.36.134
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30191
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx-test
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

### 3. 解释资源配置清单

kubectl explain deployment.metadata

```bash
[root@master ~]# kubectl explain deployment.metadata
KIND:     Deployment
VERSION:  extensions/v1beta1

RESOURCE: metadata <Object>

DESCRIPTION:
     Standard object metadata.

     ObjectMeta is metadata that all persisted resources must have, which
     includes all objects users must create.

FIELDS:
   annotations	<map[string]string>
     Annotations is an unstructured key value map stored with a resource that
     may be set by external tools to store and retrieve arbitrary metadata. They
     are not queryable and should be preserved when modifying objects. More
     info: http://kubernetes.io/docs/user-guide/annotations
......
```

kubectl explain service.metadata

```bash
[root@master ~]# kubectl explain service.metadata
KIND:     Service
VERSION:  v1

RESOURCE: metadata <Object>

DESCRIPTION:
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata

     ObjectMeta is metadata that all persisted resources must have, which
     includes all objects users must create.

FIELDS:
   annotations	<map[string]string>
     Annotations is an unstructured key value map stored with a resource that
     may be set by external tools to store and retrieve arbitrary metadata. They
     are not queryable and should be preserved when modifying objects. More
     info: http://kubernetes.io/docs/user-guide/annotations
......
```

### 4. 修改资源配置清单并应用

#### 4.1 离线修改

修改yaml文件：并用kubectl apply -f xxxx.yaml文件使之生效
注意：当apply不生效时，先使用delete清除资源，再apply创建资源
kubectl get service nginx-service -o yaml > nginx-svc.yaml

```bash
[root@master ~]# kubectl get service nginx-service -o yaml > nginx-svc.yaml
[root@master ~]# vim nginx-svc.yaml 

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2021-11-03T11:52:40Z"
  labels:
    run: nginx-test
  name: nginx-service
  namespace: default
  resourceVersion: "100051"
  selfLink: /api/v1/namespaces/default/services/nginx-service
  uid: b2ec9834-864a-4146-9296-5420ac15451d
spec:
  clusterIP: 10.1.36.134
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30191
#修改port为80
    port: 8080
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx-test
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

删除资源
kubectl delete -f nginx-svc.yaml

```bash
[root@master ~]# kubectl delete -f nginx-svc.yaml
service "nginx-service" deleted
```

新建资源
kubectl apply -f nginx-svc.yaml

```bash
[root@master ~]# kubectl apply -f nginx-svc.yaml
service/nginx-service created
```

查看service资源
kubectl get svc

```bash
[root@master ~]# kubectl get svc
NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
kubernetes      ClusterIP   10.1.0.1      <none>        443/TCP          25h
nginx-service   NodePort    10.1.36.134   <none>        8080:30191/TCP   56s
```

#### 4.2 在线修改

直接使用kubectl edit service nginx-service在线编辑配置资源清单并保存退出即时生效（如port: 888）
PS：此修改方式不会对yaml文件内容修改
kubectl edit service nginx-service

```bash
[root@master ~]# kubectl edit service nginx-service
 
......
  ports:
  - nodePort: 30191
#修改port为888
    port: 888
    protocol: TCP
    targetPort: 80
......
```

查看service资源
kubectl get svc

```bash
[root@master ~]# kubectl get svc
NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
kubernetes      ClusterIP   10.1.0.1      <none>        443/TCP         25h
nginx-service   NodePort    10.1.36.134   <none>        888:30191/TCP   6m50s
```

### 5. 删除资源配置清单

#### 5.1 陈述式删除

kubectl delete service nginx-service

```bash
[root@master ~]# kubectl delete service nginx-service
service "nginx-service" deleted
[root@master ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.1.0.1     <none>        443/TCP   25h
```

#### 5.2 声明式删除

kubectl delete -f nginx-svc.yaml

```bash
[root@master ~]# kubectl apply -f nginx-svc.yaml 
service/nginx-service created
[root@master ~]# kubectl get svc
NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
kubernetes      ClusterIP   10.1.0.1      <none>        443/TCP          25h
nginx-service   NodePort    10.1.36.134   <none>        8080:30191/TCP   1s
[root@master ~]# kubectl delete -f nginx-svc.yaml 
service "nginx-service" deleted
[root@master ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.1.0.1     <none>        443/TCP   25h
```

