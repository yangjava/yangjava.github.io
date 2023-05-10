---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes源码环境搭建
作为领先的容器编排引擎，Kubernetes提供了一个抽象层，使其可以在物理或虚拟环境中部署容器应用程序，提供以容器为中心的基础架构。

## Kubernetes源码
k8s源码[https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)

k8s源码结构
```
|--api  // 存放api规范相关的文档
|------api-rules  //已经存在的违反Api规范的api
|------openapi-spec  //OpenApi规范
|--build // 构建和测试脚本
|------run.sh  //在容器中运行该脚本，后面可接多个命令：make, make cross 等
|------copy-output.sh  //把容器中_output/dockerized/bin目录下的文件拷贝到本地目录
|------make-clean.sh  //清理容器中和本地的_output目录
|------shell.sh  // 容器中启动一个shell终端
|------......
|--cluster  // 自动创建和配置kubernetes集群的脚本，包括networking, DNS, nodes等
|--cmd  // 内部包含各个组件的入口，具体核心的实现部分在pkg目录下
|--hack  // 编译、构建及校验的工具类
|--logo // kubernetes的logo
|--pkg // 主要代码存放类，后面会详细补充该目录下内容
|------kubeapiserver
|------kubectl
|------kubelet
|------proxy
|------registry
|------scheduler
|------security
|------watch
|------......
|--plugin
|------pkg/admission  //认证
|------pkg/auth  //鉴权
|--staging  // 这里的代码都存放在独立的repo中，以引用包的方式添加到项目中
|------k8s.io/api
|------k8s.io/apiextensions-apiserver
|------k8s.io/apimachinery
|------k8s.io/apiserver
|------k8s.io/client-go
|------......
|--test  //测试代码
|--third_party  //第三方代码，protobuf、golang-reflect等
|--translations  //不同国家的语言包，使用poedit查看及编辑
```

## Kubernetes架构
Kubernetes系统用于管理分布式节点集群中的微服务或容器化应用程序，并且其提供了零停机时间部署、自动回滚、缩放和容器的自愈（其中包括自动配置、自动重启、自动复制的高弹性基础设施，以及容器的自动缩放等）等功能。

Kubernetes系统最重要的设计因素之一是能够横向扩展，即调整应用程序的副本数以提高可用性。设计一套大型系统，且保证其运行时健壮、可扩展、可移植和非常具有挑战性，尤其是在系统复杂度增加时，系统的体系结构会直接影响其运行方式、对环境的依赖程度及相关组件的耦合程度。

微服务是一种软件设计模式，适用于集群上的可扩展部署。开发人员使用这一模式能够创建小型、可组合的应用程序，通过定义良好的HTTP REST API接口进行通信。Kubernetes也是遵循微服务架构模式的程序，具有弹性、可观察性和管理功能，可以适应云平台的需求。

Kubernetes的系统架构设计与Borg的系统架构设计理念非常相似，如Scheduler调度器、Pod资源对象管理等。

Kubernetes系统架构遵循客户端/服务端（C/S）架构，系统架构分为Master和Node两部分，Master作为服务端，Node作为客户端。Kubernetes系统具有多个Master服务端，可以实现高可用。在默认的情况下，一个Master服务端即可完成所有工作。

Master服务端也被称为主控节点，它在集群中主要负责如下任务。
- 集群的“大脑”，负责管理所有节点（Node）。
- 负责调度Pod在哪些节点上运行。
- 负责控制集群运行过程中的所有状态。

Node客户端也被称为工作节点，它在集群中主要负责如下任务。
- 负责管理所有容器（Container）。
- 负责监控/上报所有Pod的运行状态。

Master服务端（主控节点）主要负责管理和控制整个Kubernetes集群，对集群做出全局性决策，相当于整个集群的“大脑”。集群所执行的所有控制命令都由Master服务端接收并处理。Master服务端主要包含如下组件。
- kube-apiserver组件 ：集群的HTTP REST API接口，是集群控制的入口。
- kube-controller-manager组件 ：集群中所有资源对象的自动化控制中心。
- kube-scheduler组件 ：集群中Pod资源对象的调度服务。

Node客户端（工作节点）是Kubernetes集群中的工作节点，Node节点上的工作由Master服务端进行分配，比如当某个Node节点宕机时，Master节点会将其上面的工作转移到其他Node节点上。Node节点主要包含如下组件。
- kubelet组件 ：负责管理节点上容器的创建、删除、启停等任务，与Master节点进行通信。
- kube-proxy组件 ：负责Kubernetes服务的通信及负载均衡服务。
- container组件 ：负责容器的基础管理服务，接收kubelet组件的指令。

## Kubernetes各组件的功能
架构中主要的组件有kubectl、kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy和container等。另外，作为开发者，还需要深入了解client-go库。不同组件之间是松耦合架构，各组件之间各司其职，保证整个集群的稳定运行。下面对各组件进行更细化的架构分析和功能阐述。

### kubectl
kubectl是Kubernetes官方提供的命令行工具（CLI），用户可以通过kubectl以命令行交互的方式对Kubernetes API Server进行操作，通信协议使用HTTP/JSON。

kubectl发送相应的HTTP请求，请求由Kubernetes API Server接收、处理并将结果反馈给kubectl。kubectl接收到响应并展示结果。至此，kubectl与kube-apiserver的一次请求周期结束。

### client-go
kubectl是通过命令行交互的方式与Kubernetes API Server进行交互的，Kubernetes还提供了通过编程的方式与Kubernetes API Server进行通信。client-go是从Kubernetes的代码中单独抽离出来的包，并作为官方提供的Go语言的客户端发挥作用。client-go简单、易用，Kubernetes系统的其他组件与Kubernetes API Server通信的方式也基于client-go实现。

在大部分基于Kubernetes做二次开发的程序中，建议通过client-go来实现与Kubernetes API Server的交互过程。这是因为client-go在Kubernetes系统上做了大量的优化，Kubernetes核心组件（如kube-scheduler、kube-controller-manager等）都通过client-go与Kubernetes API Server进行交互。

### kube-apiserver
kube-apiserver组件，也被称为Kubernetes API Server。它负责将Kubernetes“资源组/资源版本/资源”以RESTful风格的形式对外暴露并提供服务。Kubernetes集群中的所有组件都通过kube-apiserver组件操作资源对象。kube-apiserver组件也是集群中唯一与Etcd集群进行交互的核心组件。例如，开发者通过kubectl创建了一个Pod资源对象，请求通过kube-apiserver的HTTP接口将Pod资源对象存储至Etcd集群中。

Etcd集群是分布式键值存储集群，其提供了可靠的强一致性服务发现。Etcd集群存储Kubernetes系统集群的状态和元数据，其中包括所有Kubernetes资源对象信息、集群节点信息等。Kubernetes将所有数据存储至Etcd集群中前缀为/registry的目录下。

kube-apiserver属于核心组件，对于整个集群至关重要，它具有以下重要特性。
- 将Kubernetes系统中的所有资源对象都封装成RESTful风格的API接口进行管理。
- 可进行集群状态管理和数据管理，是唯一与Etcd集群交互的组件。
- 拥有丰富的集群安全访问机制，以及认证、授权及准入控制器。
- 提供了集群各组件的通信和交互功能。

### kube-controller-manager
kube-controller-manager组件，也被称为Controller Manager（管理控制器），它负责管理Kubernetes集群中的节点（Node）、Pod副本、服务、端点（Endpoint）、命名空间（Namespace）、服务账户（ServiceAccount）、资源定额（ResourceQuota）等。例如，当某个节点意外宕机时，Controller Manager会及时发现并执行自动化修复流程，确保集群始终处于预期的工作状态。

Controller Manager负责确保Kubernetes系统的实际状态收敛到所需状态，其默认提供了一些控制器（Controller），例如DeploymentControllers控制器、StatefulSet控制器、Namespace控制器及PersistentVolume控制器等，每个控制器通过kube-apiserver组件提供的接口实时监控整个集群每个资源对象的当前状态，当因发生各种故障而导致系统状态出现变化时，会尝试将系统状态修复到“期望状态”。

Controller Manager具备高可用性（即多实例同时运行），即基于Etcd集群上的分布式锁实现领导者选举机制，多实例同时运行，通过kube-apiserver提供的资源锁进行选举竞争。抢先获取锁的实例被称为Leader节点（即领导者节点），并运行kube-controller-manager组件的主逻辑；而未获取锁的实例被称为Candidate节点（即候选节点），运行时处于阻塞状态。在Leader节点因某些原因退出后，Candidate节点则通过领导者选举机制参与竞选，成为Leader节点后接替kube-controller-manager的工作。

### kube-scheduler
kube-scheduler组件，也被称为调度器，目前是Kubernetes集群的默认调度器。它负责在Kubernetes集群中为一个Pod资源对象找到合适的节点并在该节点上运行。调度器每次只调度一个Pod资源对象，为每一个Pod资源对象寻找合适节点的过程是一个调度周期。

kube-scheduler组件监控整个集群的Pod资源对象和Node资源对象，当监控到新的Pod资源对象时，会通过调度算法为其选择最优节点。调度算法分为两种，分别为预选调度算法和优选调度算法。除调度策略外，Kubernetes还支持优先级调度、抢占机制及亲和性调度等功能。

kube-scheduler组件支持高可用性（即多实例同时运行），即基于Etcd集群上的分布式锁实现领导者选举机制，多实例同时运行，通过kube-apiserver提供的资源锁进行选举竞争。抢先获取锁的实例被称为Leader节点（即领导者节点），并运行kube-scheduler组件的主逻辑；而未获取锁的实例被称为Candidate节点（即候选节点），运行时处于阻塞状态。在Leader节点因某些原因退出后，Candidate节点则通过领导者选举机制参与竞选，成为Leader节点后接替kube-scheduler的工作。

### kubelet
kubelet组件，用于管理节点，运行在每个Kubernetes节点上。kubelet组件用来接收、处理、上报kube-apiserver组件下发的任务。kubelet进程启动时会向kube-apiserver注册节点自身信息。它主要负责所在节点（Node）上的Pod资源对象的管理，例如Pod资源对象的创建、修改、监控、删除、驱逐及Pod生命周期管理等。

kubelet组件会定期监控所在节点的资源使用状态并上报给kube-apiserver组件，这些资源数据可以帮助kube-scheduler调度器为Pod资源对象预选节点。kubelet也会对所在节点的镜像和容器做清理工作，保证节点上的镜像不会占满磁盘空间、删除的容器释放相关资源。

kubelet组件实现了3种开放接口
- Container Runtime Interface ：简称CRI（容器运行时接口），提供容器运行时通用插件接口服务。CRI定义了容器和镜像服务的接口。CRI将kubelet组件与容器运行时进行解耦，将原来完全面向Pod级别的内部接口拆分成面向Sandbox和Container的gRPC接口，并将镜像管理和容器管理分离给不同的服务。
- Container Network Interface ：简称CNI（容器网络接口），提供网络通用插件接口服务。CNI定义了Kubernetes网络插件的基础，容器创建时通过CNI插件配置网络。
- Container Storage Interface ：简称CSI（容器存储接口），提供存储通用插件接口服务。CSI定义了容器存储卷标准规范，容器创建时通过CSI插件配置存储卷。

### kube-proxy
kube-proxy组件，作为节点上的网络代理，运行在每个Kubernetes节点上。它监控kube-apiserver的服务和端点资源变化，并通过iptables/ipvs等配置负载均衡器，为一组Pod提供统一的TCP/UDP流量转发和负载均衡功能。

kube-proxy组件是参与管理Pod-to-Service和External-to-Service网络的最重要的节点组件之一。kube-proxy组件相当于代理模型，对于某个IP：Port的请求，负责将其转发给专用网络上的相应服务或应用程序。但是，kube-proxy组件与其他负载均衡服务的区别在于，kube-proxy代理只向Kubernetes服务及其后端Pod发出请求。

## Kubernetes构建过程
构建过程是指“编译器”读取Go语言代码文件，经过大量的处理流程，最终产生一个二进制文件的过程；也就是将人类可读的代码转化成计算机可执行的二进制代码的过程。

手动构建Kubernetes二进制文件是一件非常麻烦的事情，尤其是对于较为复杂的Kubernetes大型程序来说。Kubernetes官方专门提供了一套编译工具，使构建过程变得更容易。

Kubernetes构建方式可以分为3种，分别是本地环境构建、容器环境构建、Bazel环境构建

### 本地环境构建
执行make或make all命令，会编译Kubernetes的所有组件，组件二进制文件输出的相对路径是_output/bin/。如果我们需要对Makefile的执行过程进行调试，可以在make命令后面加-n参数，输出但不执行所有执行命令，这样可以展示更详细的构建过程。
假设我们想单独构建某一个组件，如kubectl组件，则需要指定WHAT参数，命令示例如下：
```
$ make WHAT=cmd/kubectl
```










# 参考资料
Kubernetes源码剖析[https://www.ai2news.com/blog/2111249/]




















