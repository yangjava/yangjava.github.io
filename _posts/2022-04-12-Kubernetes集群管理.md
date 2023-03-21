---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes集群管理

## Node的管理
## Node的隔离与恢复
在硬件升级、硬件维护等情况下，我们需要将某些Node隔离，使其脱离Kubernetes集群的调度范围。Kubernetes提供了一种机制，既可以将Node纳入调度范围，也可以将Node脱离调度范围。

创建配置文件unschedule_node.yaml，在spec部分指定unschedulable为true：
```yaml
apiVersion: v1
kind: Node
metadata:
	name: k8s-node-1
	labels:
		kubernetes.io/hostname:k8s-node-1
spec:
	unschedulable: true
```
通过kubectl replace命令完成对Node状态的修改：
```
kubectl replace -f unschedule_node.yaml
```
查看Node的状态，可以观察到在Node的状态中增加了一项SchedulingDisabled：
这样，对于后续创建的Pod，系统将不会再向该Node进行调度。

也可以不使用配置文件，直接使用kubectl patch命令完成：
```
kubectl patch node k8s-node-1 -p '{"spec":{"unschedulable":true}}'
```
需要注意的是，将某个Node脱离调度范围时，在其上运行的Pod并不会自动停止，管理员需要手动停止在该Node上运行的Pod。

同样，如果需要将某个Node重新纳入集群调度范围，则将unschedulable设置为false，再次执行kubectl replace或kubectl patch命令就能恢复系统对该Node的调度。

另外，使用kubectl的子命令cordon和uncordon也可以实现将Node进行隔离调度和恢复调度操作。

例如，使用kubectl cordon <node_name>对某个Node进行隔离调度操作：
```
kubectl cordon k8s-node-1
```
使用 kubectl uncordon <node_name>对某个Node进行恢复调度操作：
```
kubectl uncordon k8s-node-1
```

## Node的扩容
在实际生产系统中经常会出现服务器容量不足的情况，这时就需要购买新的服务器，然后将应用系统进行水平扩展来完成对系统的扩容。

在Kubernetes集群中，一个新Node的加入是非常简单的。在新的Node上安装Docker、kubelet和kube-proxy服务，然后配置kubelet和kube-proxy的启动参数，将Master URL指定为当前Kubernetes集群Master的地址，最后启动这些服务。通过kubelet默认的自动注册机制，新的Node将会自动加入现有的Kubernetes集群中。

Kubernetes Master在接受了新Node的注册之后，会自动将其纳入当前集群的调度范围，之后创建容器时，就可以对新的Node进行调度了。 通过这种机制，Kubernetes实现了集群中Node的扩容。

## 更新资源对象的Label
Label是用户可灵活定义的对象属性，对于正在运行的资源对象，我们随时可以通过kubectl label命令进行增加、修改、删除等操作。

例如，要给已创建的Pod“redis-master-bobr0”添加一个标签role=backend：
```
kubectl label pod redis-master-bobr0 role=backend
```
查看该Pod的Label：
```
kubectl get pods -Lrole
```
删除一个Label时，只需在命令行最后指定Label的key名并与一个减号相连即可：
```
kubectl label pod redis-master-bobr0 role-
```
在修改一个Label的值时，需要加上--overwrite参数：
```
kubectl label pod redis-master-bobr0 role=master --overwrite
```

## Namespace：集群环境共享与隔离
在一个组织内部，不同的工作组可以在同一个Kubernetes集群中工作，Kubernetes通过命名空间和Context的设置对不同的工作组进行区分，使得它们既可以共享同一个Kubernetes集群的服务，也能够互不干扰

假设在我们的组织中有两个工作组：开发组和生产运维组。开发组在Kubernetes集群中需要不断创建、修改、删除各种Pod、RC、Service等资源对象，以便实现敏捷开发。生产运维组则需要使用严格的权限设置来确保生产系统中的Pod、RC、Service处于正常运行状态且不会被误操作。

### 创建Namespace
为了在Kubernetes集群中实现这两个分组，首先需要创建两个命名空间：
namespace-development.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
name: development
```
namespace-production.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
name: production
```
使用kubectl create命令完成命名空间的创建：
```
kubectl create -f namespace-development.yaml
kubectl create -f namespace-production.yaml
```
查看系统中的命名空间：
```
kubectl get namespaces
```

## 定义Context（运行环境）
接下来，需要为这两个工作组分别定义一个 Context，即运行环境。这个运行环境将属于某个特定的命名空间。

通过kubectl config set-context命令定义Context，并将Context置于之前创建的命名空间中：
```
kubectl config set-cluster kubernetes-cluster --server=https://192.168.1.128:8080
kubectl config set-context ctx-dev --namespace=development --cluster=kubernetes-cluster --user=dev
kubectl config set-context ctx-dev --namespace=production --cluster=kubernetes-cluster --user=prod
```
使用kubectl config view命令查看已定义的Context：
```
kubectl config view
```
注意，通过kubectl config命令在${HOME}/.kube目录下生成了一个名为config的文件，文件的内容为以kubectl config view命令查看到的内容。所以，也可以通过手工编辑该文件的方式来设置Context。

## 设置工作组在特定Context环境下工作
使用kubectl config use-context <context_name>命令设置当前运行环境。

下面的命令将把当前运行环境设置为ctx-dev：
```
kubectl config use-context ctx-dev
```
运行这个命令后，当前的运行环境被设置为开发组所需的环境。之后的所有操作都将在名为development的命名空间中完成。

现在，以redis-slave RC为例创建两个Pod：
```yaml

apiVersion: v1
kind: ReplicationController
metadata:
  name: redis-slave
  labels:
    name: redis-slave
spec:
  replicas: 2
  selector: 
    name: redis-slave
  template:
    metadata:
      labels:
        name: redis-slave
    spec:
      containser:
      - name: slave
        image: kubeguide/guestbook-redis-slave
        ports:

```
查看创建好的Pod：
```
kubectl get pods
```
可以看到容器被正确创建并运行起来了。而且，由于当前运行环境是ctx-dev，所以不会影响生产运维组的工作。

切换到生产运维组的运行环境查看RC和Pod：
```
kubectl config use-context ctx-prod
kubectl get rc
kubectl get pods
```
结果为空，说明看不到开发组创建的RC和Pod。
现在也为生产运维组创建两个redis-slave的Pod，查看创建好的Pod。可以看到容器被正确创建并运行起来了，并且当前运行环境是ctx-prod，也不会影响开发组的工作。

至此，我们为两个工作组分别设置了两个运行环境，设置好当前运行环境时，各工作组之间的工作将不会相互干扰，并且都能在同一个Kubernetes集群中同时工作。

## Kubernetes资源管理
讲解Pod的两个重要参数：CPU Request与Memory Request。在大多数情况下，我们在定义Pod时并没有定义这两个参数，此时Kubernetes会认为该Pod所需的资源很少，并可以将其调度到任何可用的Node上。这样一来，当集群中的计算资源不很充足时，如果集群中的Pod负载突然加大，就会使某个Node的资源严重不足。

为了避免系统挂掉，该Node会选择“清理”某些Pod来释放资源，此时每个Pod都可能成为牺牲品。但有些Pod担负着更重要的职责，比其他Pod更重要，比如与数据存储相关的、与登录相关的、与查询余额相关的，即使系统资源严重不足，也需要保障这些Pod的存活，Kubernetes中该保障机制的核心如下。
- 通过资源限额来确保不同的Pod只能占用指定的资源。
- 允许集群的资源被超额分配，以提高集群的资源利用率。
- 为Pod划分等级，确保不同等级的Pod有不同的服务质量（QoS），资源不足时，低等级的Pod会被清理，以确保高等级的Pod稳定运行。

Kubernetes集群里的节点提供的资源主要是计算资源，计算资源是可计量的能被申请、分配和使用的基础资源，这使之区别于API资源（API Resources，例如Pod和Services等）。当前Kubernetes集群中的计算资源主要包括CPU、GPU及Memory，绝大多数常规应用是用不到GPU的，因此这里重点介绍CPU与Memory的资源管理问题。

CPU与Memory是被Pod使用的，因此在配置Pod时可以通过参数CPU Request及Memory Request为其中的每个容器指定所需使用的CPU与Memory量，Kubernetes会根据Request的值去查找有足够资源的Node来调度此Pod，如果没有，则调度失败。

我们知道，一个程序所使用的CPU与Memory是一个动态的量，确切地说，是一个范围，跟它的负载密切相关：负载增加时，CPU和Memory的使用量也会增加。因此最准确的说法是，某个进程的CPU使用量为0.1个CPU～1个CPU，内存占用则为500MB～1GB。对应到Kubernetes的Pod容器上，就是下面这4个参数：
- spec.container[].resources.requests.cpu；
- spec.container[].resources.limits.cpu；
- spec.container[].resources.requests.memory；
- spec.container[].resources.limits.memory。

其中，limits对应资源量的上限，即最多允许使用这个上限的资源量。由于CPU资源是可压缩的，进程无论如何也不可能突破上限，因此设置起来比较容易。对于Memory这种不可压缩资源来说，它的Limit设置就是一个问题了，如果设置得小了，当进程在业务繁忙期试图请求超过Limit限制的Memory时，此进程就会被Kubernetes杀掉。因此，Memory的Request与Limit的值需要结合进程的实际需求谨慎设置。如果不设置CPU或Memory的Limit值，会怎样呢？在这种情况下，该Pod的资源使用量有一个弹性范围，我们不用绞尽脑汁去思考这两个Limit的合理值，但问题也来了，考虑下面的例子：

Pod A的 Memory Request被设置为1GB，Node A当时空闲的Memory为1.2GB，符合Pod A的需求，因此Pod A被调度到Node A上。运行3天后，Pod A的访问请求大增，内存需要增加到1.5GB，此时Node A的剩余内存只有200MB，由于Pod A新增的内存已经超出系统资源，所以在这种情况下，Pod A就会被Kubernetes杀掉。

没有设置Limit的Pod，或者只设置了CPU Limit或者Memory Limit两者之一的Pod，表面看都是很有弹性的，但实际上，相对于4个参数都被设置的Pod，是处于一种相对不稳定的状态的，它们与4个参数都没设置的Pod相比，只是稳定一点而已。理解了这一点，就很容易理解Resource QoS问题了。

如果我们有成百上千个不同的Pod，那么先手动设置每个Pod的这4个参数，再检查并确保这些参数的设置，都是合理的。比如不能出现内存超过2GB或者CPU占据2个核心的Pod。最后还得手工检查不同租户（Namespace）下的Pod的资源使用量是否超过限额。为此，Kubernetes提供了另外两个相关对象：LimitRange及ResourceQuota，前者解决request与limit参数的默认值和合法取值范围等问题，后者则解决约束租户的资源配额问题。

本章从计算资源管理（Compute Resources）、服务质量管理（QoS）、资源配额管理（LimitRange、ResourceQuota）等方面，对Kubernetes集群内的资源管理进行详细说明，并结合实践操作、常见问题分析和一个完整的示例，力求对Kubernetes集群资源管理相关的运维工作提供指导。

## 计算资源管理
### 详解Requests和Limits参数
尽管Requests和Limits只能被设置到容器上，但是设置Pod级别的Requests和Limits能大大提高管理Pod的便利性和灵活性，因此在Kubernetes中提供了对Pod级别的Requests和Limits的配置。对于CPU和内存而言，Pod的Requests或Limits是指该Pod中所有容器的Requests或Limits的总和（对于Pod中没有设置Requests或Limits的容器，该项的值被当作0或者按照集群配置的默认值来计算）。下面对CPU和内存这两种计算资源的特点进行说明。
### CPU
CPU的Requests和Limits是通过CPU数（cpus）来度量的。CPU的资源值是绝对值，而不是相对值，比如0.1CPU在单核或多核机器上是一样的，都严格等于0.1 CPU core。

### Memory
内存的Requests和Limits计量单位是字节数。使用整数或者定点整数加上国际单位制（International System of Units）来表示内存值。国际单位制包括十进制的E、P、T、G、M、K、m，或二进制的Ei、Pi、Ti、Gi、Mi、Ki。KiB与MiB是以二进制表示的字节单位，常见的KB与MB则是以十进制表示的字节单位，比如：
- 1 KB（KiloByte）= 1000 Bytes = 8000 Bits；
- 1 KiB（KibiByte）= 210 Bytes = 1024 Bytes = 8192 Bits。
因此，128974848、129e6、129M、123Mi的内存配置是一样的。

Kubernetes的计算资源单位是大小写敏感的，因为m可以表示千分之一单位（milli unit），而M可以表示十进制的1000，二者的含义不同；同理，小写的k不是一个合法的资源单位。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
 continers:
 - name: db
   image: mysql
   resources:
     requests:
       memory: "64Mi"
       cpu: "250m"
     limits:
       memory: "128Mi"
       cpu: "500m"
 - name: wp
   image: wordpress
   resources:
     requests:
       memory: "64Mi"
       cpu: "250m"
     limits:
       memory: "128Mi"
       cpu: "500m"
```
解读：如上所示，该Pod包含两个容器，每个容器配置的Requests都是0.25CPU和64MiB（226 Bytes）内存，而配置的Limits都是0.5CPU和
128MiB（227 Bytes）内存。
这个Pod的Requests和Limits等于Pod中所有容器对应配置的总和，所以Pod的Requests是0.5CPU和128MiB（227 Bytes）内存，Limits是1CPU和256MiB（228 Bytes）内存。

### 基于Requests和Limits的Pod调度机制
当一个Pod创建成功时，Kubernetes调度器（Scheduler）会为该Pod选择一个节点来执行。对于每种计算资源（CPU和Memory）而言，每个节点都有一个能用于运行Pod的最大容量值。调度器在调度时，首先要确保调度后该节点上所有Pod的CPU和内存的Requests总和，不超过该节点能提供给Pod使用的CPU和Memory的最大容量值。

例如，某个节点上的CPU资源充足，而内存为4GB，其中3GB可以运行Pod，而某Pod的Memory Requests为1GB、Limits为2GB，那么在这个节点上最多可以运行3个这样的Pod。

这里需要注意：可能某节点上的实际资源使用量非常低，但是已运行Pod配置的Requests值的总和非常高，再加上需要调度的Pod的Requests值，会超过该节点提供给Pod的资源容量上限，这时Kubernetes仍然不会将Pod调度到该节点上。如果Kubernetes将Pod调度到该节点上，之后该节点上运行的Pod又面临服务峰值等情况，就可能导致Pod资源短缺。

接着上面的例子，假设该节点已经启动3个Pod实例，而这3个Pod的实际内存使用都不足500MB，那么理论上该节点的可用内存应该大于 1.5GB。但是由于该节点的Pod Requests总和已经达到节点的可用内存上限，因此Kubernetes不会再将任何Pod实例调度到该节点上。

### Requests和Limits的背后机制
kubelet在启动Pod的某个容器时，会将容器的Requests和Limits值转化为相应的容器启动参数传递给容器执行器（Docker或者rkt）。
如果容器的执行环境是Docker，那么容器的如下4个参数是这样传递给Docker的。
- spec.container[].resources.requests.cpu
这个参数会转化为core数（比如配置的100m会转化为0.1），然后乘以1024，再将这个结果作为--cpu-shares参数的值传递给docker run命令。在docker run命令中，--cpu-share参数是一个相对权重值（Relative Weight），这个相对权重值会决定Docker在资源竞争时分配给容器的资源比例。

举例说明--cpu-shares参数在Docker中的含义：比如将两个容器的CPU Requests分别设置为1和2，那么容器在docker run启动时对应的--cpu-shares参数值分别为1024和2048，在主机CPU资源产生竞争时，Docker会尝试按照1∶2的配比将CPU资源分配给这两个容器使用。
这里需要区分清楚的是：这个参数对于Kubernetes而言是绝对值，主要用于Kubernetes调度和管理；同时Kubernetes会将这个参数的值传递给docker run的--cpu-shares参数。--cpu-shares参数对于Docker而言是相对值，主要用于资源分配比例。

- spec.container[].resources.limits.cpu
这个参数会转化为millicore数（比如配置的1被转化为1000，而配置的100m被转化为100），将此值乘以100000，再除以1000，然后将结果值作为--cpu-quota参数的值传递给docker run命令。docker run命令中另外一个参数--cpu-period默认被设置为100000，表示Docker重新计量和分配CPU的使用时间间隔为100000μs（100ms）。
Docker的--cpu-quota参数和--cpu-period参数一起配合完成对容器CPU的使用限制：比如Kubernetes中配置容器的CPU Limits为0.1，那么计算后--cpu-quota为10000，而--cpu-period为100000，这意味着Docker在100ms内最多给该容器分配10ms×core的计算资源用量，10/100=0.1 core的结果与Kubernetes配置的意义是一致的。
注意：如果kubelet的启动参数--cpu-cfs-quota被设置为true，那么kubelet会强制要求所有Pod都必须配置CPU Limits（如果Pod没有配置，则集群提供了默认配置也可以）。从Kubernetes 1.2版本开始，这个--cpu-cfs-quota启动参数的默认值就是true。

- spec.container[].resources.requests.memory
这个参数值只提供给Kubernetes调度器作为调度和管理的依据，不会作为任何参数传递给Docker。

- spec.container[].resources.limits.memory
这个参数值会转化为单位为Bytes的整数，数值会作为--memory参数传递给docker run命令。

如果一个容器在运行过程中使用了超出了其内存Limits配置的内存限制值，那么它可能会被杀掉，如果这个容器是一个可重启的容器，那么之后它会被kubelet重新启动。因此对容器的Limits配置需要进行准确测试和评估。

与内存Limits不同的是，CPU在容器技术中属于可压缩资源，因此对CPU的Limits配置一般不会因为偶然超标使用而导致容器被系统杀掉。

### 计算资源使用情况监控
Pod的资源用量会作为Pod的状态信息一同上报给Master。如果在集群中配置了Heapster来监控集群的性能数据，那么还可以从Heapster中查看Pod的资源用量信息。

### 计算资源相关常见问题分析
Pod状态为Pending，错误信息为FailedScheduling。如果Kubernetes调度器在集群中找不到合适的节点来运行Pod，那么这个Pod会一直处于未调度状态，直到调度器找到合适的节点为止。每次调度器尝试调度失败时，Kubernetes都会产生一个事件，我们可以通过下面这种方式来查看事件的信息：
```
kubectl describe pod <podname>| grep -A 3 Events
```
在上面这个例子中，名为frontend的Pod由于节点的CPU资源不足而调度失败（Pod ExceedsFreeCPU），同样，如果内存不足，则也可能导致调度失败（PodExceedsFreeMemory）。
如果一个或者多个Pod调度失败且有这类错误，那么可以尝试以下几种解决方法。
- 添加更多的节点到集群中。
- 停止一些不必要的运行中的Pod，释放资源。
- 检查Pod的配置，错误的配置可能导致该Pod永远无法被调度执行。比如整个集群中所有节点都只有1 CPU，而Pod配置的CPU Requests为2，该Pod就不会被调度执行。

我们可以使用kubectl describe nodes命令来查看集群中节点的计算资源容量和已使用量：
```
kubectl describe nodes k8snode01
```
超过可用资源容量上限（Capacity）和已分配资源量（Allocated resources）差额的Pod无法运行在该Node上。在这个例子中，如果一个Pod的Requests超过90 millicpus或者超过1341MiB内存，就无法运行在这个节点上。在10.4.4节中，我们还可以配置针对一组Pod的Requests和Limits总量的限制，这种限制可以作用于命名空间，通过这种方式可以防止一个命名空间下的用户将所有资源据为己有。

容器被强行终止（Terminated）。如果容器使用的资源超过了它配置的Limits，那么该容器可能会被强制终止。我们可以通过kubectl describe pod命令来确认容器是否因为这个原因被终止：
```
kubectl describe pod
```
Restart Count: 5说明这个名为simmemleak的容器被强制终止并重启了5次。

我们可以在使用kubectl get pod命令时添加-o go-template=...格式参数来读取已终止容器之前的状态信息：
```
kubectl get pod -o go-template='{{range.status.containerStatuses}}{{"Container Name:"}}{{.name}}{{"\r\nLastate:"}}{{.lastState}}{{end}}'
```
可以看到这个容器因为reason:OOM Killed而被强制终止，说明这个容器的内存超过了限制（Out of Memory）。

## 对大内存页（Huge Page）资源的支持


## 资源配置范围管理（LimitRange）
在默认情况下，Kubernetes不会对Pod加上CPU和内存限制，这意味着Kubernetes系统中任何Pod都可以使用其所在节点的所有可用的CPU和内存。通过配置Pod的计算资源Requests和Limits，我们可以限制Pod的资源使用，但对于Kubernetes集群管理员而言，配置每一个Pod的Requests和Limits是烦琐的，而且很受限制。更多时候，我们需要对集群内Requests和Limits的配置做一个全局限制。常见的配置场景如下。
- 集群中的每个节点都有2GB内存，集群管理员不希望任何Pod申请超过2GB的内存：因为在整个集群中都没有任何节点能满足超过2GB内存的请求。如果某个Pod的内存配置超过2GB，那么该Pod将永远都无法被调度到任何节点上执行。为了防止这种情况的发生，集群管理员希望能在系统管理功能中设置禁止Pod申请超过2GB内存。
- 集群由同一个组织中的两个团队共享，分别运行生产环境和开发环境。生产环境最多可以使用8GB内存，而开发环境最多可以使用512MB内存。集群管理员希望通过为这两个环境创建不同的命名空间，并为每个命名空间设置不同的限制来满足这个需求。
- 用户创建Pod时使用的资源可能会刚好比整个机器资源的上限稍小，而恰好剩下的资源大小非常尴尬：不足以运行其他任务但整个集群加起来又非常浪费。因此，集群管理员希望设置每个Pod都必须至少使用集群平均资源值（CPU和内存）的20%，这样集群能够提供更好的资源一致性的调度，从而减少了资源浪费。

针对这些需求，Kubernetes提供了LimitRange机制对Pod和容器的Requests和Limits配置进一步做出限制。在下面的示例中，将说明如何将LimitsRange应用到一个Kubernetes的命名空间中，然后说明LimitRange的几种限制方式，比如最大及最小范围、Requests和Limits的默认值、Limits与Requests的最大比例上限等。
- 创建一个Namespace

















