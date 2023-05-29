---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes入门
Kubernetes是用于自动部署，扩展和管理容器化应用程序的开源系统。

**Kubernetes, also known as K8s, is an open-source system for automating deployment, scaling, and management of containerized applications.**

## 什么是Kubernetes
Kubernetes (通常称为K8s，K8s是将8个字母“ubernete”替换为“8”的缩写) 是用于自动部署、扩展和管理容器化（containerized）应用程序的开源系统。Google设计并捐赠给Cloud Native Computing Foundation（今属Linux基金会）来使用的。 它旨在提供“跨主机集群的自动部署、扩展以及运行应用程序容器的平台”。它支持一系列容器工具, 包括Docker等。

## Kubernetes发展史
Kubernetes (希腊语"舵手" 或 "飞行员") 由Joe Beda，Brendan Burns和Craig McLuckie创立，并由其他谷歌工程师，包括Brian Grant和Tim Hockin进行加盟创作，并由谷歌在2014年首次对外宣布 。它的开发和设计都深受谷歌的Borg系统的影响，它的许多顶级贡献者之前也是Borg系统的开发者。在谷歌内部，Kubernetes的原始代号曾经是Seven，即星际迷航中友好的Borg(博格人)角色。Kubernetes标识中舵轮有七个轮辐就是对该项目代号的致意。

Kubernetes v1.0于2015年7月21日发布。随着v1.0版本发布，谷歌与Linux 基金会合作组建了Cloud Native Computing Foundation (CNCF)并把Kubernetes作为种子技术来提供。

Rancher Labs在其Rancher容器管理平台中包含了Kubernetes的发布版。Kubernetes也在很多其他公司的产品中被使用，比如Red Hat在OpenShift产品中，CoreOS的Tectonic产品中， 以及IBM的IBM云私有产品中。

## Kubernetes是什么？
首先，它是一个全新的基于容器技术的分布式架构领先方案。这个方案虽然还很新，但它是谷歌十几年以来大规模应用容器技术的经验积累和升华的重要成果。它基于容器技术，目的是实现资源管理的自动化，以及跨多个数据中心的资源利用率的最大化。

其次，如果我们的系统设计遵循了Kubernetes的设计思想，那么传统系统架构中那些和业务没有多大关系的底层代码或功能模块，都可以立刻从我们的视线中消失，我们不必再费心于负载均衡器的选型和部署实施问题，不必再考虑引入或自己开发一个复杂的服务治理框架，不必再头疼于服务监控和故障处理模块的开发。

然后，Kubernetes是一个开放的开发平台。Kubernetes平台对现有的编程语言、编程框架、中间件没有任何侵入性，因此现有的系统也很容易改造升级并迁移到Kubernetes平台上。

最后，Kubernetes是一个完备的分布式系统支撑平台。Kubernetes具有完备的集群管理能力，包括多层次的安全防护和准入机制、多租户应用支撑能力、透明的服务注册和服务发现机制、内建的智能负载均衡器、强大的故障发现和自我修复能力、服务滚动升级和在线扩容能力、可扩展的资源自动调度机制，以及多粒度的资源配额管理能力。同时，Kubernetes提供了完善的管理工具，这些工具涵盖了包括开发、部署测试、运维监控在内的各个环节。因此，Kubernetes是一个全新的基于容器技术的分布式架构解决方案，并且是一个一站式的完备的分布式系统开发和支撑平台。

在正式开始Hello World之旅之前，我们首先要学习Kubernetes的一些基本知识，这样才能理解Kubernetes提供的解决方案。
在Kubernetes中，Service是分布式集群架构的核心，一个Service对象拥有如下关键特征。
- 拥有唯一指定的名称（比如mysql-server）。
- 拥有一个虚拟IP（Cluster IP、Service IP或VIP）和端口号。
- 能够提供某种远程服务能力。
- 被映射到提供这种服务能力的一组容器应用上。
Service的服务进程目前都基于Socket通信方式对外提供服务，比如Redis、Memcache、MySQL、Web Server，或者是实现了某个具体业务的特定TCP Server进程。虽然一个Service通常由多个相关的服务进程提供服务，每个服务进程都有一个独立的Endpoint（IP+Port）访问点，但Kubernetes能够让我们通过Service（虚拟Cluster IP +Service Port）连接到指定的Service。
容器提供了强大的隔离功能，所以有必要把为Service提供服务的这组进程放入容器中进行隔离。为此，Kubernetes设计了Pod对象，将每个服务进程都包装到相应的Pod中，使其成为在Pod中运行的一个容器（Container）。为了建立Service和Pod间的关联关系，Kubernetes首先给每个Pod都贴上一个标签（Label），给运行MySQL的Pod贴上name=mysql标签，给运行PHP的Pod贴上name=php标签，然后给相应的Service定义标签选择器（Label Selector），比如MySQL Service的标签选择器的选择条件为name=mysql，意为该Service要作用于所有包含name=mysql Label的Pod。这样一来，就巧妙解决了Service与Pod的关联问题。
这里先简单介绍Pod的概念。首先，Pod运行在一个被称为节点（Node）的环境中，这个节点既可以是物理机，也可以是私有云或者公有云中的一个虚拟机，通常在一个节点上运行几百个Pod；其次，在每个Pod中都运行着一个特殊的被称为Pause的容器，其他容器则为业务容器，这些业务容器共享Pause容器的网络栈和Volume挂载卷，因此它们之间的通信和数据交换更为高效，在设计时我们可以充分利用这一特性将一组密切相关的服务进程放入同一个Pod中；最后，需要注意的是，并不是每个Pod和它里面运行的容器都能被映射到一个Service上，只有提供服务（无论是对内还是对外）的那组Pod才会被映射为一个服务。
在集群管理方面，Kubernetes将集群中的机器划分为一个Master和一些Node。在Master上运行着集群管理相关的一组进程kube-apiserver、kube-controller-manager和kubescheduler，这些进程实现了整个集群的资源管理、Pod调度、弹性伸缩、安全控制、系统监控和纠错等管理功能，并且都是自动完成的。Node作为集群中的工作节点，运行真正的应用程序，在Node上Kubernetes管理的最小运行单元是Pod。在Node上运行着Kubernetes的kubelet、kube-proxy服务进程，这些服务进程负责Pod的创建、启动、监控、重启、销毁，以及实现软件模式的负载均衡器。

在Kubernetes集群中，只需为需要扩容的Service关联的Pod创建一个RC（Replication Controller），服务扩容以至服务升级等令人头疼的问题都迎刃而解。在一个RC定义文件中包括以下3个关键信息。
- 目标Pod的定义。
- 目标Pod需要运行的副本数量（Replicas）。
- 要监控的目标Pod的标签。
在创建好RC（系统将自动创建好Pod）后，Kubernetes会通过在RC中定义的Label筛选出对应的Pod实例并实时监控其状态和数量，如果实例数量少于定义的副本数量，则会根据在RC中定义的Pod模板创建一个新的Pod，然后将此Pod调度到合适的Node上启动运行，直到Pod实例的数量达到预定目标。这个过程完全是自动化的，无须人工干预。有了RC，服务扩容就变成一个纯粹的简单数字游戏了，只需修改RC中的副本数量即可。后续的服务升级也将通过修改RC来自动完成。

## 为什么要用Kubernetes
2015年，谷歌联合20多家公司一起建立了CNCF（Cloud Native Computing Foundation，云原生计算基金会）开源组织来推广Kubernetes，并由此开创了云原生应用（Cloud Native Application）的新时代。作为CNCF“钦定”的官方云原生平台，Kubernetes正在颠覆应用程序的开发方式。

使用Kubernetes会收获哪些好处呢？
首先，可以“轻装上阵”地开发复杂系统。以前需要很多人（其中不乏技术达人）一起分工协作才能设计、实现和运维的分布式系统，在采用Kubernetes解决方案之后，只需一个精悍的小团队就能轻松应对。
其次，可以全面拥抱微服务架构。微服务架构的核心是将一个巨大的单体应用分解为很多小的互相连接的微服务，一个微服务可能由多个实例副本支撑，副本的数量可以随着系统的负荷变化进行调整。微服务架构使得每个服务都可以独立开发、升级和扩展，因此系统具备很高的稳定性和快速迭代能力，开发者也可以自由选择开发技术。
再次，可以随时随地将系统整体“搬迁”到公有云上。未来会有更多的公有云及私有云支持Kubernetes。
然后，Kubernetes内在的服务弹性扩容机制可以让我们轻松应对突发流量。
最后，Kubernetes系统架构超强的横向扩容能力可以让我们的竞争力大大提升。对于互联网公司来说，用户规模等价于资产，因此横向扩容能力是衡量互联网业务系统竞争力的关键指标。我们利用Kubernetes提供的工具，不用修改代码，就能将一个Kubernetes集群从只包含几个Node的小集群平滑扩展到拥有上百个Node的大集群，甚至可以在线完成集群扩容。

## 从一个简单的例子开始

## 启动MySQL服务
```yaml
apiVersion: v1
kind: ReplicationController             # 类型是副本控制器
metadata:                               
  name: mysql                           # RC的名称全局是唯一的，这里name是mysql
spec:
  replicas: 1                           # 副本数量1
  selector:                             # RC管理 拥有label={app:mysql}这个标签的 pod
    app: mysql
  template:                             # template代表下面开始定义一个pod
    metadata:
      labels:
        app: mysql                      # 这个pod拥有{app:mysql}这样一个标签
    spec:                               
      containers:                       # pod中的容器定义部分
      - name: mysql 
        image: mysql:5.7
        ports:
        - containerPort: 3306           # 容器监听的端口
        env:                            # 注入容器里面的环境变量
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
```
以上YAML定义文件中的kind属性用来表明此资源对象的类型，比如这里的值为ReplicationController，表示这是一个RC；在spec一节中是RC的相关属性定义，比如spec.selector是RC的Pod标签选择器，即监控和管理拥有这些标签的Pod实例，确保在当前集群中始终有且仅有replicas个Pod实例在运行，这里设置replicas=1，表示只能运行一个MySQL Pod实例。
当在集群中运行的Pod数量少于replicas时，RC会根据在spec.template一节中定义的Pod模板来生成一个新的Pod实例，spec.template.metadata.labels指定了该Pod的标签，需要特别注意的是：这里的labels必须匹配之前的spec.selector，否则此RC每创建一个无法匹配Label的Pod，就会不停地尝试创建新的Pod，陷入恶性循环中。

在创建好mysql-rc.yaml文件后，为了将它发布到Kubernetes集群中，我们在Master上执行命令：
```
kubectl create -f mysql-rc.yaml
```
查看Pod的创建情况时，可以运行下面的命令：
```
kubectl get pods
```
我们看到一个名为mysql-xxxxx的Pod实例，这是Kubernetes根据mysql这个RC的定义自动创建的Pod。由于Pod的调度和创建需要花费一定的时间，比如需要一定的时间来确定调度到哪个节点上，以及下载Pod里的容器镜像需要一段时间，所以我们一开始看到Pod的状态显示为Pending。在Pod成功创建完成以后，状态最终会被更新为Running。

我们通过docker ps指令查看正在运行的容器，发现提供MySQL服务的Pod容器已经创建并正常运行了，此外会发现MySQL Pod对应的容器还多创建了一个来自谷歌的pause容器，这就是Pod的“根容器”

最后，创建一个与之关联的Kubernetes Service—MySQL的定义文件（文件名为mysql-svc.yaml），完整的内容和解释如下：
```yaml
apiVersion: v1
kind: Service              # 类型是service
metadata:                  
  name: mysql              # 这个service的全局唯一名称
spec:
  ports:
    - port: 3306           # service提供服务的端口号
  selector:
    app: mysql             # 把拥有{app:label}这个标签的pod应用到这个服务里面
```
其中，metadata.name是Service的服务名（ServiceName）；port属性则定义了Service的虚端口；spec.selector确定了哪些Pod副本（实例）对应本服务。类似地，我们通过kubectl create 命令创建Service对象。
```
kubectl create -f mysql-svc.yaml
```
运行kubectl命令查看刚刚创建的Service：
```yaml
kubectl get svc
```
可以发现，MySQL服务被分配了一个值为x.x.x.x的Cluster IP地址。随后，Kubernetes集群中其他新创建的Pod就可以通过Service的Cluster IP+端口号3306来连接和访问它了。

通常，Cluster IP是在Service创建后由Kubernetes系统自动分配的，其他Pod无法预先知道某个Service的Cluster IP地址，因此需要一个服务发现机制来找到这个服务。为此，最初时，Kubernetes巧妙地使用了Linux环境变量（Environment Variable）来解决这个问题，后面会详细说明其机制。现在只需知道，根据Service的唯一名称，容器可以从环境变量中获取Service对应的Cluster IP地址和端口，从而发起TCP/IP连接请求。

## Kubernetes的基本概念和术语
Kubernetes中的大部分概念如Node、Pod、Replication Controller、Service等都可以被看作一种资源对象，几乎所有资源对象都可以通过Kubernetes提供的kubectl工具（或者API编程调用）执行增、删、改、查等操作并将其保存在etcd中持久化存储。从这个角度来看，Kubernetes其实是一个高度自动化的资源控制系统，它通过跟踪对比etcd库里保存的“资源期望状态”与当前环境中的“实际资源状态”的差异来实现自动控制和自动纠错的高级功能。

在声明一个Kubernetes资源对象的时候，需要注意一个关键属性：apiVersion。以下面的Pod声明为例，可以看到Pod这种资源对象归属于v1这个核心API。
```yaml
apiVersion: v1
kind: ReplicationController             # 类型是副本控制器
metadata:                               
  name: mysql                           # RC的名称全局是唯一的，这里name是mysql
spec:
  replicas: 1                           # 副本数量1
  selector:                             # RC管理 拥有label={app:mysql}这个标签的 pod
    app: mysql
  template:                             # template代表下面开始定义一个pod
    metadata:
      labels:
        app: mysql                      # 这个pod拥有{app:mysql}这样一个标签
    spec:                               
      containers:                       # pod中的容器定义部分
      - name: mysql 
        image: mysql:5.7
        ports:
        - containerPort: 3306           # 容器监听的端口
        env:                            # 注入容器里面的环境变量
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
```
Kubernetes平台采用了“核心+外围扩展”的设计思路，在保持平台核心稳定的同时具备持续演进升级的优势。Kubernetes大部分常见的核心资源对象都归属于v1这个核心API，比如Node、Pod、Service、Endpoints、Namespace、RC、PersistentVolume等。在版本迭代过程中，Kubernetes先后扩展了extensions/v1beta1、apps/v1beta1、apps/v1beta2等API组，而在1.9版本之后引入了apps/v1这个正式的扩展API组，正式淘汰（deprecated）了extensions/v1beta1、apps/v1beta1、apps/v1beta2这三个API组。

我们可以采用YAML或JSON格式声明（定义或创建）一个Kubernetes资源对象，每个资源对象都有自己的特定语法格式（可以理解为数据库中一个特定的表），但随着Kubernetes版本的持续升级，一些资源对象会不断引入新的属性。为了在不影响当前功能的情况下引入对新特性的支持，我们通常会采用下面两种典型方法。
- 方法1，在设计数据库表的时候，在每个表中都增加一个很长的备注字段，之后扩展的数据以某种格式（如XML、JSON、简单字符串拼接等）放入备注字段。因为数据库表的结构没有发生变化，所以此时程序的改动范围是最小的，风险也更小，但看起来不太美观。
- 方法2，直接修改数据库表，增加一个或多个新的列，此时程序的改动范围较大，风险更大，但看起来比较美观。

显然，两种方法都不完美。更加优雅的做法是，先采用方法1实现这个新特性，经过几个版本的迭代，等新特性变得稳定成熟了以后，可以在后续版本中采用方法2升级到正式版。为此，Kubernetes为每个资源对象都增加了类似数据库表里备注字段的通用属性Annotations，以实现方法1的升级。以Kubernetes 1.3版本引入的Pod的Init Container新特性为例，一开始，Init Container的定义是在Annotations中声明的，如下面代码中粗体部分所示，是不是很不美观？
```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: myapp-pod
  labels: 
    app: myapp 
  annotations: 
    pod.beta.kubernetes.io/init-containers: '[
       {
         "name": "init-mydb",
         "image": "busybox",
         "command": [....]
    ]'
spec: 
  containers:
    .......
```
在Kubernetes 1.8版本以后，Init container特性完全成熟，其定义被放入Pod的spec.initContainers一节，看起来优雅了很多：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  # These containers are run during pod initialization
  jnitContainers:
  -name: init-mydb
   image: busybox
   command:
   -xxx
```
在Kubernetes 1.8中，资源对象中的很多Alpha、Beta版本的Annotations被取消，升级成了常规定义方式，在学习Kubernetes的过程中需要特别注意。

## Master
Kubernetes里的Master指的是集群控制节点，在每个Kubernetes集群里都需要有一个Master来负责整个集群的管理和控制，基本上Kubernetes的所有控制命令都发给它，它负责具体的执行过程，我们后面执行的所有命令基本都是在Master上运行的。Master通常会占据一个独立的服务器（高可用部署建议用3台服务器），主要原因是它太重要了，是整个集群的“首脑”，如果它宕机或者不可用，那么对集群内容器应用的管理都将失效。
在Master上运行着以下关键进程。
- Kubernetes API Server（kube-apiserver）：提供了HTTP Rest接口的关键服务进程，是Kubernetes里所有资源的增、删、改、查等操作的唯一入口，也是集群控制的入口进程。
- Kubernetes Controller Manager（kube-controller-manager）：Kubernetes里所有资源对象的自动化控制中心，可以将其理解为资源对象的“大总管”。
- Kubernetes Scheduler（kube-scheduler）：负责资源调度（Pod调度）的进程，相当于公交公司的“调度室”。
另外，在Master上通常还需要部署etcd服务，因为Kubernetes里的所有资源对象的数据都被保存在etcd中。

## Node
除了Master，Kubernetes集群中的其他机器被称为Node，在较早的版本中也被称为Minion。与Master一样，Node可以是一台物理主机，也可以是一台虚拟机。Node是Kubernetes集群中的工作负载节点，每个Node都会被Master分配一些工作负载（Docker容器），当某个Node宕机时，其上的工作负载会被Master自动转移到其他节点上。
在每个Node上都运行着以下关键进程。
- kubelet：负责Pod对应的容器的创建、启停等任务，同时与Master密切协作，实现集群管理的基本功能。
- kube-proxy：实现Kubernetes Service的通信与负载均衡机制的重要组件。
- Docker Engine（docker）：Docker引擎，负责本机的容器创建和管理工作。
Node可以在运行期间动态增加到Kubernetes集群中，前提是在这个节点上已经正确安装、配置和启动了上述关键进程，在默认情况下kubelet会向Master注册自己，这也是Kubernetes推荐的Node管理方式。一旦Node被纳入集群管理范围，kubelet进程就会定时向Master汇报自身的情报，例如操作系统、Docker版本、机器的CPU和内存情况，以及当前有哪些Pod在运行等，这样Master就可以获知每个Node的资源使用情况，并实现高效均衡的资源调度策略。而某个Node在超过指定时间不上报信息时，会被Master判定为“失联”，Node的状态被标记为不可用（Not Ready），随后Master会触发“工作负载大转移”的自动流程。
我们可以执行下述命令查看在集群中有多少个Node：
```
kubectl get nodes
```
然后，通过kubectl describe node <node_name>查看某个Node的详细信息：
```
kubectl describe node <node_name>
```
上述命令展示了Node的如下关键信息。
- Node的基本信息：名称、标签、创建时间等。
- Node当前的运行状态：Node启动后会做一系列的自检工作，比如磁盘空间是否不足（DiskPressure）、内存是否不足（MemoryPressure）、网络是否正常（NetworkUnavailable）、PID资源是否充足（PIDPressure）。在一切正常时设置Node为Ready状态（Ready=True），该状态表示Node处于健康状态，Master将可以在其上调度新的任务了（如启动Pod）。
- Node的主机地址与主机名。
- Node上的资源数量：描述Node可用的系统资源，包括CPU、内存数量、最大可调度Pod数量等。
- Node可分配的资源量：描述Node当前可用于分配的资源量。
- 主机系统信息：包括主机ID、系统UUID、Linux kernel版本号、操作系统类型与版本、Docker版本号、kubelet与kube-proxy的版本号等。
- 当前运行的Pod列表概要信息。
- 已分配的资源使用概要信息，例如资源申请的最低、最大允许使用量占系统总量的百分比。
- Node相关的Event信息。

## Pod
Pod是Kubernetes最重要的基本概念，Pod的组成，我们看到每个Pod都有一个特殊的被称为“根容器”的Pause容器。Pause容器对应的镜像属于Kubernetes平台的一部分，除了Pause容器，每个Pod还包含一个或多个紧密相关的用户业务容器。

为什么Kubernetes会设计出一个全新的Pod的概念并且Pod有这样特殊的组成结构？
- 原因之一：在一组容器作为一个单元的情况下，我们难以简单地对“整体”进行判断及有效地行动。比如，一个容器死亡了，此时算是整体死亡么？是N/M的死亡率么？引入业务无关并且不易死亡的Pause容器作为Pod的根容器，以它的状态代表整个容器组的状态，就简单、巧妙地解决了这个难题。
- 原因之二：Pod里的多个业务容器共享Pause容器的IP，共享Pause容器挂接的Volume，这样既简化了密切关联的业务容器之间的通信问题，也很好地解决了它们之间的文件共享问题。

Kubernetes为每个Pod都分配了唯一的IP地址，称之为Pod IP，一个Pod里的多个容器共享Pod IP地址。Kubernetes要求底层网络支持集群内任意两个Pod之间的TCP/IP直接通信，这通常采用虚拟二层网络技术来实现，例如Flannel、Open vSwitch等，因此我们需要牢记一点：在Kubernetes里，一个Pod里的容器与另外主机上的Pod容器能够直接通信。

Pod其实有两种类型：普通的Pod及静态Pod（Static Pod）。后者比较特殊，它并没被存放在Kubernetes的etcd存储里，而是被存放在某个具体的Node上的一个具体文件中，并且只在此Node上启动、运行。而普通的Pod一旦被创建，就会被放入etcd中存储，随后会被Kubernetes Master调度到某个具体的Node上并进行绑定（Binding），随后该Pod被对应的Node上的kubelet进程实例化成一组相关的Docker容器并启动。在默认情况下，当Pod里的某个容器停止时，Kubernetes会自动检测到这个问题并且重新启动这个Pod（重启Pod里的所有容器），如果Pod所在的Node宕机，就会将这个Node上的所有Pod重新调度到其他节点上。

Kubernetes里的所有资源对象都可以采用YAML或者JSON格式的文件来定义或描述，下面是我们用到的myweb这个Pod的资源定义文件：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myweb
  labels:
    name: myweb
spec:
  containers:
  - name: myweb
  image: kubeguide/tomcat-app: v1
  ports:
  - containerPort: 8080
```
Kind为Pod表明这是一个Pod的定义，metadata里的name属性为Pod的名称，在metadata里还能定义资源对象的标签，这里声明myweb拥有一个name=myweb的标签。在Pod里所包含的容器组的定义则在spec一节中声明，这里定义了一个名为myweb、对应镜像为kubeguide/tomcat-app:v1的容器，该容器注入了名为MYSQL_SERVICE_HOST='mysql'和MYSQL_SERVICE_PORT='3306'的环境变量（env关键字），并且在8080端口（containerPort）启动容器进程。Pod的IP加上这里的容器端口（containerPort），组成了一个新的概念—Endpoint，它代表此Pod里的一个服务进程的对外通信地址。一个Pod也存在具有多个Endpoint的情况，比如当我们把Tomcat定义为一个Pod时，可以对外暴露管理端口与服务端口这两个Endpoint。

我们所熟悉的Docker Volume在Kubernetes里也有对应的概念—Pod Volume，后者有一些扩展，比如可以用分布式文件系统GlusterFS实现后端存储功能；Pod Volume是被定义在Pod上，然后被各个容器挂载到自己的文件系统中的。

这里顺便提一下Kubernetes的Event概念。Event是一个事件的记录，记录了事件的最早产生时间、最后重现时间、重复次数、发起者、类型，以及导致此事件的原因等众多信息。Event通常会被关联到某个具体的资源对象上，是排查故障的重要参考信息，之前我们看到Node的描述信息包括了Event，而Pod同样有Event记录，当我们发现某个Pod迟迟无法创建时，可以用kubectl describe pod xxxx来查看它的描述信息，以定位问题的成因。

每个Pod都可以对其能使用的服务器上的计算资源设置限额，当前可以设置限额的计算资源有CPU与Memory两种，其中CPU的资源单位为CPU（Core）的数量，是一个绝对值而非相对值。

对于绝大多数容器来说，一个CPU的资源配额相当大，所以在Kubernetes里通常以千分之一的CPU配额为最小单位，用m来表示。通常一个容器的CPU配额被定义为100～300m，即占用0.1～0.3个CPU。由于CPU配额是一个绝对值，所以无论在拥有一个Core的机器上，还是在拥有48个Core的机器上，100m这个配额所代表的CPU的使用量都是一样的。与CPU配额类似，Memory配额也是一个绝对值，它的单位是内存字节数。
在Kubernetes里，一个计算资源进行配额限定时需要设定以下两个参数。
- Requests：该资源的最小申请量，系统必须满足要求。
- Limits：该资源最大允许使用的量，不能被突破，当容器试图使用超过这个量的资源时，可能会被Kubernetes“杀掉”并重启。
通常，我们会把Requests设置为一个较小的数值，符合容器平时的工作负载情况下的资源需求，而把Limit设置为峰值负载情况下资源占用的最大量。下面这段定义表明MySQL容器申请最少0.25个CPU及64MiB内存，在运行过程中MySQL容器所能使用的资源配额为0.5个CPU及128MiB内存：
```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"  
```

## Label
Label（标签）是Kubernetes系统中另外一个核心概念。一个Label是一个key=value的键值对，其中key与value由用户自己指定。Label可以被附加到各种资源对象上，例如Node、Pod、Service、RC等，一个资源对象可以定义任意数量的Label，同一个Label也可以被添加到任意数量的资源对象上。Label通常在资源对象定义时确定，也可以在对象创建后动态添加或者删除。

我们可以通过给指定的资源对象捆绑一个或多个不同的Label来实现多维度的资源分组管理功能，以便灵活、方便地进行资源分配、调度、配置、部署等管理工作。例如，部署不同版本的应用到不同的环境中；监控和分析应用（日志记录、监控、告警）等。一些常用的Label示例如下。
- 版本标签："release":"stable"、"release":"canary"。
- 环境标签："environment":"dev"、"environment":"qa"、"environment":"production"。
- 架构标签："tier":"frontend"、"tier":"backend"、"tier":"middleware"。
- 分区标签："partition":"customerA"、"partition":"customerB"。
- 质量管控标签："track":"daily"、"track":"weekly"。
Label相当于我们熟悉的“标签”。给某个资源对象定义一个Label，就相当于给它打了一个标签，随后可以通过Label Selector（标签选择器）查询和筛选拥有某些Label的资源对象，Kubernetes通过这种方式实现了类似SQL的简单又通用的对象查询机制。
Label Selector可以被类比为SQL语句中的where查询条件，例如，name=redis-slave这个Label Selector作用于Pod时，可以被类比为select * from pod where pod’s name =‘redis-slave’这样的语句。当前有两种Label Selector表达式：基于等式的（Equality-based）和基于集合的（Set-based），前者采用等式类表达式匹配标签，下面是一些具体的例子。
- name=redis-slave：匹配所有具有标签name=redis-slave的资源对象。
- env!=production：匹配所有不具有标签env=production的资源对象，比如env=test就是满足此条件的标签之一。
后者则使用集合操作类表达式匹配标签，下面是一些具体的例子。
- name in（redis-master, redis-slave）：匹配所有具有标签name=redis-master或者name=redis-slave的资源对象。
- name not in（php-frontend）：匹配所有不具有标签name=php-frontend的资源对象。
可以通过多个Label Selector表达式的组合实现复杂的条件选择，多个表达式之间用“，”进行分隔即可，几个条件之间是“AND”的关系，即同时满足多个条件，比如下面的例子：
```

name=标签名
env != 标签名
name in (标签1，标签2)
name not in(标签1)
name in (redis-master, redis-slave):匹配所有具有标签`name=redis-master`或者`name=redis-slave`的资源对象。
name not in (phn-frontend):匹配所有不具有标签name=php-frontend的资源对象。
name=redis-slave, env!=production
name notin (php-frontend),env!=production

```
以myweb Pod为例，Label被定义在其metadata中：管理对象RC和Service则通过Selector字段设置需要关联Pod的Label：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myweb
  lables: 
    app: myweb
# 管理对象RC和Service 在 spec 中定义Selector 与 Pod 进行关联。
apiVersion: v1
kind: ReplicationController
metadata:
  name: myweb
spec: 
  replicas: 1
  selector:
    app: myweb
  template:
  ...略...
apiVersion" v1
kind: Service
metadata: 
  name: myweb
spec: 
  selector:
    app: myweb
  ports:
    port: 8080
```
其他管理对象如Deployment、ReplicaSet、DaemonSet和Job则可以在Selector中使用基于集合的筛选条件定义，例如：
```yaml
selector:
  matchLabels:
     app: myweb
  matchExpressions:
     - {key: tire,operator: In,values: [frontend]}
     - {key: environment, operator: NotIn, values: [dev]}
```
matchLabels用于定义一组Label，与直接写在Selector中的作用相同；matchExpressions用于定义一组基于集合的筛选条件，可用的条件运算符包括In、NotIn、Exists和DoesNotExist。

如果同时设置了matchLabels和matchExpressions，则两组条件为AND关系，即需要同时满足所有条件才能完成Selector的筛选。

Label Selector在Kubernetes中的重要使用场景如下。
- kube-controller进程通过在资源对象RC上定义的Label Selector来筛选要监控的Pod副本数量，使Pod副本数量始终符合预期设定的全自动控制流程。
- kube-proxy进程通过Service的Label Selector来选择对应的Pod，自动建立每个Service到对应Pod的请求转发路由表，从而实现Service的智能负载均衡机制。
- 通过对某些Node定义特定的Label，并且在Pod定义文件中使用NodeSelector这种标签调度策略，kube-scheduler进程可以实现Pod定向调度的特性。
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: podnodea
  name: podnodea
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: podnodea
    resources: {}
  affinity:
    nodeAffinity: #主机亲和性
      requiredDuringSchedulingIgnoredDuringExecution: #硬策略
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - vms85.liruilongs.github.io
            - vms84.liruilongs.github.io
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```
使用Label可以给对象创建多组标签，Label和Label Selector共同构成了Kubernetes系统中核心的应用模型，使得被管理对象能够被精细地分组管理，同时实现了整个集群的高可用性。

## Replication Controller
RC是Kubernetes系统中的核心概念之一，简单来说，它其实定义了一个期望的场景，即声明某种Pod的副本数量在任意时刻都符合某个预期值，所以RC的定义包括如下几个部分。
- Pod期待的副本数量。
- 用于筛选目标Pod的Label Selector。
- 当Pod的副本数量小于预期数量时，用于创建新Pod的Pod模板（template）。
下面是一个完整的RC定义的例子，即确保拥有tier=frontend标签的这个Pod（运行Tomcat容器）在整个Kubernetes集群中始终只有一个副本：
```yaml
apiVersion: v1
kind: ReplicationController
metadata: 
  name: frontend
spec:
  replicas: 1
  selector:
    tier: frontend
  template: 
    metadata:
      labels:
        app: app-demo
        tier: frontend
    spec:
      containers:
      - name: tomcat-demo
        image: tomcat
        imagePullPolicy: IfNotPresent
        env:
        - name: GET_HOSTS_FROM
          value: dns
        ports:
        - containerPort: 80

```
在我们定义了一个RC并将其提交到Kubernetes集群中后，Master上的Controller Manager组件就得到通知，定期巡检系统中当前存活的目标Pod，并确保目标Pod实例的数量刚好等于此RC的期望值，如果有过多的Pod副本在运行，系统就会停掉一些Pod，否则系统会再自动创建一些Pod。可以说，通过RC，Kubernetes实现了用户应用集群的高可用性，并且大大减少了系统管理员在传统IT环境中需要完成的许多手工运维工作（如主机监控脚本、应用监控脚本、故障恢复脚本等）。
此外，在运行时，我们可以通过修改RC的副本数量，来实现Pod的动态缩放（Scaling），这可以通过执行kubectl scale命令来一键完成：
```
kubectl scale rc redsi-slave --replicas=3
```
需要注意的是，删除RC并不会影响通过该RC已创建好的Pod。为了删除所有Pod，可以设置replicas的值为0，然后更新该RC。另外，kubectl提供了stop和delete命令来一次性删除RC和RC控制的全部Pod。

应用升级时，通常会使用一个新的容器镜像版本替代旧版本。我们希望系统平滑升级，比如在当前系统中有10个对应的旧版本的Pod，则最佳的系统升级方式是旧版本的Pod每停止一个，就同时创建一个新版本的Pod，在整个升级过程中此消彼长，而运行中的Pod数量始终是10个，几分钟以后，当所有的Pod都已经是新版本时，系统升级完成。通过RC机制，Kubernetes很容易就实现了这种高级实用的特性，被称为“滚动升级”（Rolling Update）

Replication Controller由于与Kubernetes代码中的模块Replication Controller同名，同时“Replication Controller”无法准确表达它的本意，所以在Kubernetes 1.2中，升级为另外一个新概念—Replica Set，官方解释其为“下一代的RC”。Replica Set与RC当前的唯一区别是，Replica Sets支持基于集合的Label selector（Set-based selector），而RC只支持基于等式的Label Selector（equality-based selector），这使得Replica Set的功能更强。
kubectl命令行工具适用于RC的绝大部分命令同样适用于Replica Set。此外，我们当前很少单独使用Replica Set，它主要被Deployment这个更高层的资源对象所使用，从而形成一整套Pod创建、删除、更新的编排机制。我们在使用Deployment时，无须关心它是如何创建和维护Replica Set的，这一切都是自动发生的。
Replica Set与Deployment这两个重要的资源对象逐步替代了之前RC的作用，是Kubernetes 1.3里Pod自动扩容（伸缩）这个告警功能实现的基础，也将继续在Kubernetes未来的版本中发挥重要的作用。
最后总结一下RC（Replica Set）的一些特性与作用。
- 在大多数情况下，我们通过定义一个RC实现Pod的创建及副本数量的自动控制。
- 在RC里包括完整的Pod定义模板。
- RC通过Label Selector机制实现对Pod副本的自动控制。
- 通过改变RC里的Pod副本数量，可以实现Pod的扩容或缩容。
- 通过改变RC里Pod模板中的镜像版本，可以实现Pod的滚动升级。

## Deployment
Deployment是Kubernetes在1.2版本中引入的新概念，用于更好地解决Pod的编排问题。为此，Deployment在内部使用了Replica Set来实现目的，无论从Deployment的作用与目的、YAML定义，还是从它的具体命令行操作来看，我们都可以把它看作RC的一次升级，两者的相似度超过90%。

Deployment相对于RC的一个最大升级是我们可以随时知道当前Pod“部署”的进度。实际上由于一个Pod的创建、调度、绑定节点及在目标Node上启动对应的容器这一完整过程需要一定的时间，所以我们期待系统启动N个Pod副本的目标状态，实际上是一个连续变化的“部署过程”导致的最终状态。

Deployment的典型使用场景有以下几个。
- 创建一个Deployment对象来生成对应的Replica Set并完成Pod副本的创建。
- 检查Deployment的状态来看部署动作是否完成（Pod副本数量是否达到预期的值）。
- 更新Deployment以创建新的Pod（比如镜像升级）。
- 如果当前Deployment不稳定，则回滚到一个早先的Deployment版本。
- 暂停Deployment以便于一次性修改多个PodTemplateSpec的配置项，之后再恢复Deployment，进行新的发布。
- 扩展Deployment以应对高负载。
- 查看Deployment的状态，以此作为发布是否成功的指标。
- 清理不再需要的旧版本ReplicaSets。

除了API声明与Kind类型等有所区别，Deployment的定义与Replica Set的定义很类似：
```yaml
apiversion: extensions/vlbetal       apiversion: v1
kind: Deployment                     kind: ReplicaSet
metadata:                            metadata:
  name: nginx-deployment               name: nginx-repset
```
下面通过运行一些例子来直观地感受Deployment的概念。创建一个名为tomcat-deployment.yaml的Deployment描述文件，内容如下：
```yaml
apiVersion: extensions/v1betal
kind: Deployment
metadata: 
  name: frontend
spec: 
  replicas: 1
  selector: 
  matchLabels: 
    tier: frontend
  matchExpressions:
    - {key: tier, operator: In,value: [frontend]}
  template:
    metadata:
      labels:
        app: app-demo
        tier: frontend
    spec:
      containers:
      - name: tomcat-demo
        images: tomcat
        imagePullPolicy:  IfNotPresent
        ports:
        - containerPort: 8080 

```
运行下述命令创建Deployment：
```
kubectl create -f tomcat-deploment.yaml
```
运行下述命令查看Deployment的信息：
```
kubectl get deployments
```
对上述输出中涉及的数量解释如下。
- DESIRED：Pod副本数量的期望值，即在Deployment里定义的Replica。
- CURRENT：当前Replica的值，实际上是Deployment创建的Replica Set里的Replica值，这个值不断增加，直到达到DESIRED为止，表明整个部署过程完成。
- UP-TO-DATE：最新版本的Pod的副本数量，用于指示在滚动升级的过程中，有多少个Pod副本已经成功升级。
- AVAILABLE：当前集群中可用的Pod副本数量，即集群中当前存活的Pod数量。

运行下述命令查看对应的Replica Set，我们看到它的命名与Deployment的名称有关系：
```
kubectl get rs
```
运行下述命令查看创建的Pod，我们发现Pod的命名以Deployment对应的Replica Set的名称为前缀，这种命名很清晰地表明了一个Replica Set创建了哪些Pod，对于Pod滚动升级这种复杂的过程来说，很容易排查错误

运行kubectl describe deployments，可以清楚地看到Deployment控制的Pod的水平扩展过程

## Horizontal Pod Autoscaler
通过手工执行kubectl scale命令，我们可以实现Pod扩容或缩容。如果仅仅到此为止，显然不符合谷歌对Kubernetes的定位目标—自动化、智能化。在谷歌看来，分布式系统要能够根据当前负载的变化自动触发水平扩容或缩容，因为这一过程可能是频繁发生的、不可预料的，所以手动控制的方式是不现实的。

因此，在Kubernetes的1.0版本实现后，就有人在默默研究Pod智能扩容的特性了，并在Kubernetes 1.1中首次发布重量级新特性—Horizontal Pod Autoscaling（Pod横向自动扩容，HPA）。在Kubernetes 1.2中HPA被升级为稳定版本（apiVersion: autoscaling/v1），但仍然保留了旧版本（apiVersion: extensions/v1beta1）。Kubernetes从1.6版本开始，增强了根据应用自定义的指标进行自动扩容和缩容的功能，API版本为autoscaling/v2alpha1，并不断演进。

HPA与之前的RC、Deployment一样，也属于一种Kubernetes资源对象。通过追踪分析指定RC控制的所有目标Pod的负载变化情况，来确定是否需要有针对性地调整目标Pod的副本数量，这是HPA的实现原理。当前，HPA有以下两种方式作为Pod负载的度量指标。
- CPUUtilizationPercentage。
- 应用程序自定义的度量指标，比如服务在每秒内的相应请求数（TPS或QPS）。

CPUUtilizationPercentage是一个算术平均值，即目标Pod所有副本自身的CPU利用率的平均值。一个Pod自身的CPU利用率是该Pod当前CPU的使用量除以它的Pod Request的值，比如定义一个Pod的Pod Request为0.4，而当前Pod的CPU使用量为0.2，则它的CPU使用率为50%，这样就可以算出一个RC控制的所有Pod副本的CPU利用率的算术平均值了。如果某一时刻CPUUtilizationPercentage的值超过80%，则意味着当前Pod副本数量很可能不足以支撑接下来更多的请求，需要进行动态扩容，而在请求高峰时段过去后，Pod的CPU利用率又会降下来，此时对应的Pod副本数应该自动减少到一个合理的水平。如果目标Pod没有定义Pod Request的值，则无法使用CPUUtilizationPercentage实现Pod横向自动扩容。除了使用CPUUtilizationPercentage，Kubernetes从1.2版本开始也在尝试支持应用程序自定义的度量指标。

在CPUUtilizationPercentage计算过程中使用到的Pod的CPU使用量通常是1min内的平均值，通常通过查询Heapster监控子系统来得到这个值，所以需要安装部署Heapster，这样便增加了系统的复杂度和实施HPA特性的复杂度。因此，从1.7版本开始，Kubernetes自身孵化了一个基础性能数据采集监控框架——Kubernetes Monitoring Architecture，从而更好地支持HPA和其他需要用到基础性能数据的功能模块。在Kubernetes Monitoring Architecture中，Kubernetes定义了一套标准化的API接口Resource Metrics API，以方便客户端应用程序（如HPA）从Metrics Server中获取目标资源对象的性能数据，例如容器的CPU和内存使用数据。到了Kubernetes 1.8版本，Resource Metrics API被升级为metrics.k8s.io/v1beta1，已经接近生产环境中的可用目标了。

下面是HPA定义的一个具体例子：
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  maxReplicas: 10
  minReplicas: 4
  scaleTargetRef:
    kind: Deployment
    name: nginx-demo
  targetCPUUtilizationPercentage: 90
```
根据上面的定义，可以知道这个 HPA 控制的目标对象为一个名叫 nginx-demo 的 Deployment 里的 Pod 副本，当这些 Pod 副本的 CPUUtilizationPercentage 的值超过 90% 时会触发自动动态扩容行为，扩容或缩容时必须满足一个约束条件是 Pod 的副本数要介于4与 10 之间。

除了可以通过之间定义 YAML 文件并且调用 kubectl create 的命令来创建一个 HPA 资源对象的方式，还能通过下面的简单命令直接创建等价的 HPA 对象：
```
kubectl autoscale deployment php-apache --cpu-percent=90 --min=1 --max=10
```

## StatefulSet
在Kubernetes系统中，Pod的管理对象RC、Deployment、DaemonSet和Job都面向无状态的服务。但现实中有很多服务是有状态的，特别是一些复杂的中间件集群，例如MySQL集群、MongoDB集群、Akka集群、ZooKeeper集群等，这些应用集群有4个共同点。
- 每个节点都有固定的身份ID，通过这个ID，集群中的成员可以相互发现并通信。
- 集群的规模是比较固定的，集群规模不能随意变动。
- 集群中的每个节点都是有状态的，通常会持久化数据到永久存储中。
- 如果磁盘损坏，则集群里的某个节点无法正常运行，集群功能受损。

如果通过RC或Deployment控制Pod副本数量来实现上述有状态的集群，就会发现第1点是无法满足的，因为Pod的名称是随机产生的，Pod的IP地址也是在运行期才确定且可能有变动的，我们事先无法为每个Pod都确定唯一不变的ID。另外，为了能够在其他节点上恢复某个失败的节点，这种集群中的Pod需要挂接某种共享存储，为了解决这个问题，Kubernetes从1.4版本开始引入了PetSet这个新的资源对象，并且在1.5版本时更名为StatefulSet，StatefulSet从本质上来说，可以看作Deployment/RC的一个特殊变种，它有如下特性。
- StatefulSet里的每个Pod都有稳定、唯一的网络标识，可以用来发现集群内的其他成员。假设StatefulSet的名称为kafka，那么第1个Pod叫kafka-0，第2个叫kafka-1，以此类推。
- StatefulSet控制的Pod副本的启停顺序是受控的，操作第n个Pod时，前n-1个Pod已经是运行且准备好的状态。
- StatefulSet里的Pod采用稳定的持久化存储卷，通过PV或PVC来实现，删除Pod时默认不会删除与StatefulSet相关的存储卷（为了保证数据的安全）。

StatefulSet除了要与PV卷捆绑使用以存储Pod的状态数据，还要与Headless Service配合使用，即在每个StatefulSet定义中都要声明它属于哪个Headless Service。Headless Service与普通Service的关键区别在于，它没有Cluster IP，如果解析Headless Service的DNS域名，则返回的是该Service对应的全部Pod的Endpoint列表。StatefulSet在Headless Service的基础上又为StatefulSet控制的每个Pod实例都创建了一个DNS域名，这个域名的格式为：
```
$(podname).$(headless service name) 
```

## Service
Service服务也是Kubernetes里的核心资源对象之一，Kubernetes里的每个Service其实就是我们经常提起的微服务架构中的一个微服务，之前讲解Pod、RC等资源对象其实都是为讲解Kubernetes Service做铺垫的。

既然每个Pod都会被分配一个单独的IP地址，而且每个Pod都提供了一个独立的Endpoint（Pod IP+ContainerPort）以被客户端访问，现在多个Pod副本组成了一个集群来提供服务，那么客户端如何来访问它们呢？一般的做法是部署一个负载均衡器（软件或硬件），为这组Pod开启一个对外的服务端口如8000端口，并且将这些Pod的Endpoint列表加入8000端口的转发列表，客户端就可以通过负载均衡器的对外IP地址+服务端口来访问此服务。客户端的请求最后会被转发到哪个Pod，由负载均衡器的算法所决定。
Kubernetes也遵循上述常规做法，运行在每个Node上的kube-proxy进程其实就是一个智能的软件负载均衡器，负责把对Service的请求转发到后端的某个Pod实例上，并在内部实现服务的负载均衡与会话保持机制。但Kubernetes发明了一种很巧妙又影响深远的设计：Service没有共用一个负载均衡器的IP地址，每个Service都被分配了一个全局唯一的虚拟IP地址，这个虚拟IP被称为Cluster IP。这样一来，每个服务就变成了具备唯一IP地址的通信节点，服务调用就变成了最基础的TCP网络通信问题。
我们知道，Pod的Endpoint地址会随着Pod的销毁和重新创建而发生改变，因为新Pod的IP地址与之前旧Pod的不同。而Service一旦被创建，Kubernetes就会自动为它分配一个可用的Cluster IP，而且在Service的整个生命周期内，它的Cluster IP不会发生改变。于是，服务发现这个棘手的问题在Kubernetes的架构里也得以轻松解决：只要用Service的Name与Service的Cluster IP地址做一个DNS域名映射即可完美解决问题。现在想想，这真是一个很棒的设计。
说了这么久，下面动手创建一个Service来加深对它的理解。创建一个名为tomcat-service.yaml的定义文件，内容如下：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
  ports:
  - port: 18080
  selector:
    tier: frontend
```
上述内容定义了一个名为tomcat-service的Service，它的服务端口为8080，拥有“tier=frontend”这个Label的所有Pod实例都属于它，运行下面的命令进行创建：
```yaml
kubectl create -f tomcat-server.yaml
```
我们之前在tomcat-deployment.yaml里定义的Tomcat的Pod刚好拥有这个标签，所以刚才创建的tomcat-service已经对应一个Pod实例，运行下面的命令可以查看tomcatservice的Endpoint列表，其中172.17.1.3是Pod的IP地址，端口8080是Container暴露的端口：
```yaml
kubectl get endpoints
```
你可能有疑问：“说好的Service的Cluster IP呢？怎么没有看到？”运行下面的命令即可看到tomct-service被分配的Cluster IP及更多的信息：
```yaml
kubectl get svc tomcat-service -o yaml
```
在spec.ports的定义中，targetPort属性用来确定提供该服务的容器所暴露（EXPOSE）的端口号，即具体业务进程在容器内的targetPort上提供TCP/IP接入；port属性则定义了Service的虚端口。前面定义Tomcat服务时没有指定targetPort，则默认targetPort与port相同。

接下来看看Service的多端口问题。
很多服务都存在多个端口的问题，通常一个端口提供业务服务，另外一个端口提供管理服务，比如Mycat、Codis等常见中间件。Kubernetes Service支持多个Endpoint，在存在多个Endpoint的情况下，要求每个Endpoint都定义一个名称来区分。下面是Tomcat多端口的Service定义样例：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
  ports:
  - port: 8080
   name: service-port
  - port: 8005
   name: shutdown-port
  selector:
    tier: frontend
```
多端口为什么需要给每个端口都命名呢？这就涉及Kubernetes的服务发现机制了，接下来进行讲解。

## Kubernetes的服务发现机制
任何分布式系统都会涉及“服务发现”这个基础问题，大部分分布式系统都通过提供特定的API接口来实现服务发现功能，但这样做会导致平台的侵入性比较强，也增加了开发、测试的难度。Kubernetes则采用了直观朴素的思路去解决这个棘手的问题。

首先，每个Kubernetes中的Service都有唯一的Cluster IP及唯一的名称，而名称是由开发者自己定义的，部署时也没必要改变，所以完全可以被固定在配置中。接下来的问题就是如何通过Service的名称找到对应的Cluster IP。

最早时Kubernetes采用了Linux环境变量解决这个问题，即每个Service都生成一些对应的Linux环境变量（ENV），并在每个Pod的容器启动时自动注入这些环境变量。

考虑到通过环境变量获取Service地址的方式仍然不太方便、不够直观，后来Kubernetes通过Add-On增值包引入了DNS系统，把服务名作为DNS域名，这样程序就可以直接使用服务名来建立通信连接了。目前，Kubernetes上的大部分应用都已经采用了DNS这种新兴的服务发现机制，后面会讲解如何部署DNS系统。

## 外部系统访问Service的问题
为了更深入地理解和掌握Kubernetes，我们需要弄明白Kubernetes里的3种IP，这3种IP分别如下。
- Node IP：Node的IP地址。
- Pod IP：Pod的IP地址。
- Cluster IP：Service的IP地址。
首先，Node IP是Kubernetes集群中每个节点的物理网卡的IP地址，是一个真实存在的物理网络，所有属于这个网络的服务器都能通过这个网络直接通信，不管其中是否有部分节点不属于这个Kubernetes集群。这也表明在Kubernetes集群之外的节点访问Kubernetes集群之内的某个节点或者TCP/IP服务时，都必须通过Node IP通信。

其次，Pod IP是每个Pod的IP地址，它是Docker Engine根据docker0网桥的IP地址段进行分配的，通常是一个虚拟的二层网络，前面说过，Kubernetes要求位于不同Node上的Pod都能够彼此直接通信，所以Kubernetes里一个Pod里的容器访问另外一个Pod里的容器时，就是通过Pod IP所在的虚拟二层网络进行通信的，而真实的TCP/IP流量是通过Node IP所在的物理网卡流出的。

最后说说Service的Cluster IP，它也是一种虚拟的IP，但更像一个“伪造”的IP网络，原因有以下几点。
- Cluster IP仅仅作用于Kubernetes Service这个对象，并由Kubernetes管理和分配IP地址（来源于Cluster IP地址池）。
- Cluster IP无法被Ping，因为没有一个“实体网络对象”来响应。
- Cluster IP只能结合Service Port组成一个具体的通信端口，单独的Cluster IP不具备TCP/IP通信的基础，并且它们属于Kubernetes集群这样一个封闭的空间，集群外的节点如果要访问这个通信端口，则需要做一些额外的工作。
在Kubernetes集群内，Node IP网、Pod IP网与Cluster IP网之间的通信，采用的是Kubernetes自己设计的一种编程方式的特殊路由规则，与我们熟知的IP路由有很大的不同。
根据上面的分析和总结，我们基本明白了：Service的Cluster IP属于Kubernetes集群内部的地址，无法在集群外部直接使用这个地址。那么矛盾来了：实际上在我们开发的业务系统中肯定多少有一部分服务是要提供给Kubernetes集群外部的应用或者用户来使用的，典型的例子就是Web端的服务模块，比如上面的tomcat-service，那么用户怎么访问它？
采用NodePort是解决上述问题的最直接、有效的常见做法。以tomcat-service为例，在Service的定义里做如下扩展即可
```yaml
apiVersion: v1
kind: Service
metadata:
  name: tomcat-servie
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 31002
  selector:
    tier: frontend

```
其中，nodePort:31002这个属性表明手动指定tomcat-service的NodePort为31002，否则Kubernetes会自动分配一个可用的端口。接下来在浏览器里访问http://<nodePortIP>:31002/，就可以看到Tomcat的欢迎界面了
NodePort的实现方式是在Kubernetes集群里的每个Node上都为需要外部访问的Service开启一个对应的TCP监听端口，外部系统只要用任意一个Node的IP地址+具体的NodePort端口号即可访问此服务，在任意Node上运行netstat命令，就可以看到有NodePort端口被监听：
但NodePort还没有完全解决外部访问Service的所有问题，比如负载均衡问题。假如在我们的集群中有10个Node，则此时最好有一个负载均衡器，外部的请求只需访问此负载均衡器的IP地址，由负载均衡器负责转发流量到后面某个Node的NodePort上
Load balancer组件独立于Kubernetes集群之外，通常是一个硬件的负载均衡器，或者是以软件方式实现的，例如HAProxy或者Nginx。对于每个Service，我们通常需要配置一个对应的Load balancer实例来转发流量到后端的Node上，这的确增加了工作量及出错的概率。于是Kubernetes提供了自动化的解决方案，如果我们的集群运行在谷歌的公有云GCE上，那么只要把Service的type=NodePort改为type=LoadBalancer，Kubernetes就会自动创建一个对应的Load balancer实例并返回它的IP地址供外部客户端使用。其他公有云提供商只要实现了支持此特性的驱动，则也可以达到上述目的。此外，裸机上的类似机制（Bare Metal Service Load Balancers）也在被开发。

## Job
批处理任务通常并行（或者串行）启动多个计算进程去处理一批工作项（work item），在处理完成后，整个批处理任务结束。从1.2版本开始，Kubernetes支持批处理类型的应用，我们可以通过Kubernetes Job这种新的资源对象定义并启动一个批处理任务Job。与RC、Deployment、ReplicaSet、DaemonSet类似，Job也控制一组Pod容器。从这个角度来看，Job也是一种特殊的Pod副本自动控制器，同时Job控制Pod副本与RC等控制器的工作机制有以下重要差别。

Job所控制的Pod副本是短暂运行的，可以将其视为一组Docker容器，其中的每个Docker容器都仅仅运行一次。当Job控制的所有Pod副本都运行结束时，对应的Job也就结束了。Job在实现方式上与RC等副本控制器不同，Job生成的Pod副本是不能自动重启的，对应Pod副本的RestartPoliy都被设置为Never。因此，当对应的Pod副本都执行完成时，相应的Job也就完成了控制使命，即Job生成的Pod在Kubernetes中是短暂存在的。Kubernetes在1.5版本之后又提供了类似crontab的定时任务——CronJob，解决了某些批处理任务需要定时反复执行的问题。

Job所控制的Pod副本的工作模式能够多实例并行计算，以TensorFlow框架为例，可以将一个机器学习的计算任务分布到10台机器上，在每台机器上都运行一个worker执行计算任务，这很适合通过Job生成10个Pod副本同时启动运算。

## Volume
Volume（存储卷）是Pod中能够被多个容器访问的共享目录。Kubernetes的Volume概念、用途和目的与Docker的Volume比较类似，但两者不能等价。首先，Kubernetes中的Volume被定义在Pod上，然后被一个Pod里的多个容器挂载到具体的文件目录下；其次，Kubernetes中的Volume与Pod的生命周期相同，但与容器的生命周期不相关，当容器终止或者重启时，Volume中的数据也不会丢失。最后，Kubernetes支持多种类型的Volume，例如GlusterFS、Ceph等先进的分布式文件系统。

Volume的使用也比较简单，在大多数情况下，我们先在Pod上声明一个Volume，然后在容器里引用该Volume并挂载（Mount）到容器里的某个目录上。举例来说，我们要给之前的Tomcat Pod增加一个名为datavol的Volume，并且挂载到容器的/mydata-data目录上，则只要对Pod的定义文件做如下修正即可
```yaml
template:
  metadata:
    labels:
      app: app-demo
      tier: frontend
  spec:
    volumes:
      - name: datavol
        emptyDir: {}
    containers:
    - name: tomcat-demo
      image: tomcat
      volumeMounts:
        - mountPath: /myddata-data
          name: datavol
      imagePullPolicy: IfNotPresent

```
除了可以让一个Pod里的多个容器共享文件、让容器的数据写到宿主机的磁盘上或者写文件到网络存储中，Kubernetes的Volume还扩展出了一种非常有实用价值的功能，即容器配置文件集中化定义与管理，这是通过ConfigMap这种新的资源对象来实现的

Kubernetes提供了非常丰富的Volume类型，下面逐一进行说明。
### emptyDir
一个emptyDir Volume是在Pod分配到Node时创建的。从它的名称就可以看出，它的初始内容为空，并且无须指定宿主机上对应的目录文件，因为这是Kubernetes自动分配的一个目录，当Pod从Node上移除时，emptyDir中的数据也会被永久删除。emptyDir的一些用途如下。
- 临时空间，例如用于某些应用程序运行时所需的临时目录，且无须永久保留。
- 长时间任务的中间过程CheckPoint的临时保存目录。
- 一个容器需要从另一个容器中获取数据的目录（多容器共享目录）。
目前，用户无法控制emptyDir使用的介质种类。如果kubelet的配置是使用硬盘，那么所有emptyDir都将被创建在该硬盘上。Pod在将来可以设置emptyDir是位于硬盘、固态硬盘上还是基于内存的tmpfs上，上面的例子便采用了emptyDir类的Volume。

### hostPath
hostPath为在Pod上挂载宿主机上的文件或目录，它通常可以用于以下几方面。
- 容器应用程序生成的日志文件需要永久保存时，可以使用宿主机的高速文件系统进行存储。
- 需要访问宿主机上Docker引擎内部数据结构的容器应用时，可以通过定义hostPath为宿主机/var/lib/docker目录，使容器内部应用可以直接访问Docker的文件系统。
在使用这种类型的Volume时，需要注意以下几点。
- 在不同的Node上具有相同配置的Pod，可能会因为宿主机上的目录和文件不同而导致对Volume上目录和文件的访问结果不一致。
- 如果使用了资源配额管理，则Kubernetes无法将hostPath在宿主机上使用的资源纳入管理。
在下面的例子中使用宿主机的/data目录定义了一个hostPath类型的Volume：
```yaml
volumes:
  - name: "persistent-storage"
    hostPath:
      path: "/data"
```

### gcePersistentDisk
使用这种类型的Volume表示使用谷歌公有云提供的永久磁盘（Persistent Disk，PD）存放Volume的数据，它与emptyDir不同，PD上的内容会被永久保存，当Pod被删除时，PD只是被卸载（Unmount），但不会被删除。需要注意的是，你需要先创建一个PD，才能使用gcePersistentDisk。
使用gcePersistentDisk时有以下一些限制条件。
- Node（运行kubelet的节点）需要是GCE虚拟机。
- 这些虚拟机需要与PD存在于相同的GCE项目和Zone中。

### awsElasticBlockStore
与GCE类似，该类型的Volume使用亚马逊公有云提供的EBS Volume存储数据，需要先创建一个EBS Volume才能使用awsElasticBlockStore。
使用awsElasticBlockStore的一些限制条件如下。
- Node（运行kubelet的节点）需要是AWS EC2实例。
- 这些AWS EC2实例需要与EBS Volume存在于相同的region和availability-zone中。
- EBS只支持单个EC2实例挂载一个Volume。

### NFS
使用NFS网络文件系统提供的共享目录存储数据时，我们需要在系统中部署一个NFS Server。定义NFS类型的Volume的示例如下：
```yaml

volumes:
- name: test-volume
  nfs:
    server: nfs.server.locathost
    path: "/"

```
### 其他类型的Volume
- iscsi：使用iSCSI存储设备上的目录挂载到Pod中。
- flocker：使用Flocker管理存储卷。
- glusterfs：使用开源GlusterFS网络文件系统的目录挂载到Pod中。
- rbd：使用Ceph块设备共享存储（Rados Block Device）挂载到Pod中。
- gitRepo：通过挂载一个空目录，并从Git库clone一个git repository以供Pod使用。
- secret：一个Secret Volume用于为Pod提供加密的信息，你可以将定义在Kubernetes中的Secret直接挂载为文件让Pod访问。Secret Volume是通过TMFS（内存文件系统）实现的，这种类型的Volume总是不会被持久化的。

## Persistent Volume
之前提到的Volume是被定义在Pod上的，属于计算资源的一部分，而实际上，网络存储是相对独立于计算资源而存在的一种实体资源。比如在使用虚拟机的情况下，我们通常会先定义一个网络存储，然后从中划出一个“网盘”并挂接到虚拟机上。Persistent Volume（PV）和与之相关联的Persistent Volume Claim（PVC）也起到了类似的作用。
PV可以被理解成Kubernetes集群中的某个网络存储对应的一块存储，它与Volume类似，但有以下区别。
- PV只能是网络存储，不属于任何Node，但可以在每个Node上访问。
- PV并不是被定义在Pod上的，而是独立于Pod之外定义的。
- PV目前支持的类型包括：gcePersistentDisk、AWSElasticBlockStore、AzureFile、AzureDisk、FC（Fibre Channel）、Flocker、NFS、iSCSI、RBD（Rados Block Device）、CephFS、Cinder、GlusterFS、VsphereVolume、Quobyte Volumes、VMware Photon、Portworx Volumes、ScaleIO Volumes和HostPath（仅供单机测试）。
下面给出了NFS类型的PV的一个YAML定义文件，声明了需要5Gi的存储空间：
```yaml
apiversion: v1
kind: PersistentVolume 
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce 
  nfs: 
    path: /somepath 
    server: 172.17.0.2

```
比较重要的是PV的accessModes属性，目前有以下类型。
- ReadWriteOnce：读写权限，并且只能被单个Node挂载。
- ReadOnlyMany：只读权限，允许被多个Node挂载。
- ReadWriteMany：读写权限，允许被多个Node挂载。
如果某个Pod想申请某种类型的PV，则首先需要定义一个PersistentVolumeClaim对象：
```yaml
kind: Persistentvolumeclaim 
apiversion: v1
metadata: 
  name: myclaim 
spec: 
  accessModes:
  - Readwriteonce
  resources:
    requests:
      storage: BGi 

```
然后，在Pod的Volume定义中引用上述PVC即可：
```yaml
volumes:
  - name: mypd 
    persistentvolumeclaim: 
      claimName: myclaim
```
最后说说PV的状态。PV是有状态的对象，它的状态有以下几种。
- Available：空闲状态。
- Bound：已经绑定到某个PVC上。
- Released：对应的PVC已经被删除，但资源还没有被集群收回。
- Failed：PV自动回收失败。

## Namespace
Namespace（命名空间）是Kubernetes系统中的另一个非常重要的概念，Namespace在很多情况下用于实现多租户的资源隔离。Namespace通过将集群内部的资源对象“分配”到不同的Namespace中，形成逻辑上分组的不同项目、小组或用户组，便于不同的分组在共享使用整个集群的资源的同时还能被分别管理。
Kubernetes集群在启动后会创建一个名为default的Namespace，通过kubectl可以查看：
```yaml
kubectl get namespaces
```
接下来，如果不特别指明Namespace，则用户创建的Pod、RC、Service都将被系统创建到这个默认的名为default的Namespace中。
Namespace的定义很简单。如下所示的YAML定义了名为development的Namespace。
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
```
一旦创建了Namespace，我们在创建资源对象时就可以指定这个资源对象属于哪个Namespace。比如在下面的例子中定义了一个名为busybox的Pod，并将其放入development这个Namespace里：
```yaml
apiVersion: v1
kind: Pod
metadata:
    name: busybox
    namespace: development
```
此时使用kubectl get命令查看，将无法显示。这是因为如果不加参数，则kubectl get命令将仅显示属于default命名空间的资源对象。
可以在kubectl命令中加入--namespace参数来查看某个命名空间中的对象：
```
kubectl get pods -n development
```
当给每个租户创建一个Namespace来实现多租户的资源隔离时，还能结合Kubernetes的资源配额管理，限定不同租户能占用的资源，例如CPU使用量、内存使用量等。

## Annotation
Annotation（注解）与Label类似，也使用key/value键值对的形式进行定义。不同的是Label具有严格的命名规则，它定义的是Kubernetes对象的元数据（Metadata），并且用于Label Selector。Annotation则是用户任意定义的附加信息，以便于外部工具查找。在很多时候，Kubernetes的模块自身会通过Annotation标记资源对象的一些特殊信息。
通常来说，用Annotation来记录的信息如下。
- build信息、release信息、Docker镜像信息等，例如时间戳、release id号、PR号、镜像Hash值、Docker Registry地址等。
- 日志库、监控库、分析库等资源库的地址信息。
- 程序调试工具信息，例如工具名称、版本号等。
- 团队的联系信息，例如电话号码、负责人名称、网址等。

## ConfigMap
为了能够准确和深刻理解Kubernetes ConfigMap的功能和价值，我们需要从Docker说起。我们知道，Docker通过将程序、依赖库、数据及配置文件“打包固化”到一个不变的镜像文件中的做法，解决了应用的部署的难题，但这同时带来了棘手的问题，即配置文件中的参数在运行期如何修改的问题。我们不可能在启动Docker容器后再修改容器里的配置文件，然后用新的配置文件重启容器里的用户主进程。为了解决这个问题，Docker提供了两种方式：
- 在运行时通过容器的环境变量来传递参数；
- 通过Docker Volume将容器外的配置文件映射到容器内。
这两种方式都有其优势和缺点，在大多数情况下，后一种方式更合适我们的系统，因为大多数应用通常从一个或多个配置文件中读取参数。但这种方式也有明显的缺陷：我们必须在目标主机上先创建好对应的配置文件，然后才能映射到容器里。

上述缺陷在分布式情况下变得更为严重，因为无论采用哪种方式，写入（修改）多台服务器上的某个指定文件，并确保这些文件保持一致，都是一个很难完成的目标。此外，在大多数情况下，我们都希望能集中管理系统的配置参数，而不是管理一堆配置文件。针对上述问题，Kubernetes给出了一个很巧妙的设计实现，如下所述。

首先，把所有的配置项都当作key-value字符串，当然value可以来自某个文本文件，比如配置项password=123456、user=root、host=192.168.8.4用于表示连接FTP服务器的配置参数。这些配置项可以作为Map表中的一个项，整个Map的数据可以被持久化存储在Kubernetes的Etcd数据库中，然后提供API以方便Kubernetes相关组件或客户应用CRUD操作这些数据，上述专门用来保存配置参数的Map就是Kubernetes ConfigMap资源对象。

接下来，Kubernetes提供了一种内建机制，将存储在etcd中的ConfigMap通过Volume映射的方式变成目标Pod内的配置文件，不管目标Pod被调度到哪台服务器上，都会完成自动映射。进一步地，如果ConfigMap中的key-value数据被修改，则映射到Pod中的“配置文件”也会随之自动更新。于是，Kubernetes ConfigMap就成了分布式系统中最为简单（使用方法简单，但背后实现比较复杂）且对应用无侵入的配置中心。

## 小结
上述这些组件是Kubernetes系统的核心组件，它们共同构成了Kubernetes系统的框架和计算模型。通过对它们进行灵活组合，用户就可以快速、方便地对容器集群进行配置、创建和管理。除了本章所介绍的核心组件，在Kubernetes系统中还有许多辅助配置的资源对象，例如LimitRange、ResourceQuota。另外，一些系统内部使用的对象Binding、Event等请参考Kubernetes的API文档。

# 参考资料

Kubernetes 官方文档：[https://kubernetes.io/zh/](https://kubernetes.io/zh/)

Kubernetes权威指南


