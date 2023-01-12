---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes Pod
金鞍玉勒寻芳客，未信我庐别有春。——于谦《观书》    
Pod 是一组紧密关联的容器集合，它们共享 IPC、Network 和 UTS namespace，是 Kubernetes 调度的基本单位。Pod 的设计理念是支持多个容器在一个 Pod 中共享网络和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。

## Pod 的特征
- 包含多个共享 IPC、Network 和 UTC namespace 的容器，可直接通过 localhost 通信
- 所有 Pod 内容器都可以访问共享的 Volume，可以访问共享数据
- 无容错性：直接创建的 Pod 一旦被调度后就跟 Node 绑定，即使 Node 挂掉也不会被重新调度（而是被自动删除），因此推荐使用 Deployment、Daemonset 等控制器来容错
- 优雅终止：Pod 删除的时候先给其内的进程发送 SIGTERM，等待一段时间（grace period）后才强制停止依然还在运行的进程
- 特权容器（通过 SecurityContext 配置）具有改变系统配置的权限（在网络插件中大量应用）
- Kubernetes v1.8+ 还支持容器间共享 PID namespace，需要 docker >= 1.13.1，并配置 kubelet --docker-disable-shared-pid=false。

## Pod 定义
通过 yaml 或 json 描述 Pod 和其内容器的运行环境以及期望状态，比如一个最简单的 nginx pod 可以定义为
```yaml
  
   apiVersion: v1
   kind: Pod
   metadata:
    name: nginx
    labels:
    app: nginx
   spec:
    containers:
    - name: nginx
    image: nginx
    ports:
    - containerPort: 80

```
在生产环境中，推荐使用 Deployment、StatefulSet、Job 或者 CronJob 等控制器来创建 Pod，而不推荐直接创建 Pod。

## Docker 镜像支持

目前，Kubernetes 仅支持使用 Docker 镜像来创建容器，但并非支持 Dockerfile 定义的所有行为。如下表所示




## Pod 生命周期
Kubernetes 以 PodStatus.Phase 抽象 Pod 的状态（但并不直接反映所有容器的状态）。可能的 Phase 包括
- Pending: API Server已经创建该Pod，但一个或多个容器还没有被创建，包括通过网络下载镜像的过程。
- Running: Pod中的所有容器都已经被创建且已经调度到 Node 上面，但至少有一个容器还在运行或者正在启动。
- Succeeded: Pod 调度到 Node 上面后均成功运行结束，并且不会重启。
- Failed: Pod中的所有容器都被终止了，但至少有一个容器退出失败（即退出码不为 0 或者被系统终止）。
- Unknonwn: 状态未知，因为一些原因Pod无法被正常获取，通常是由于 apiserver 无法与 kubelet 通信导致。

可以用 kubectl 命令查询 Pod Phase：
```shell
$ kubectl get pod reviews-v1-5bdc544bbd-5qgxj -o jsonpath=""
   Running
```

PodSpec 中的 restartPolicy 可以用来设置是否对退出的 Pod 重启，可选项包括 Always、OnFailure、以及 Never。比如

- 单容器的 Pod，容器成功退出时，不同 restartPolicy 时的动作为
  - Always: 重启 Container; Pod phase 保持 Running.
  - OnFailure: Pod phase 变成 Succeeded.
  - Never: Pod phase 变成 Succeeded.
  
- 单容器的 Pod，容器失败退出时，不同 restartPolicy 时的动作为
  - Always: 重启 Container; Pod phase 保持 Running.
  - OnFailure: 重启 Container; Pod phase 保持 Running.
  - Never: Pod phase 变成 Failed.

- 2个容器的 Pod，其中一个容器在运行而另一个失败退出时，不同 restartPolicy 时的动作为
  - Always: 重启 Container; Pod phase 保持 Running.
  - OnFailure: 重启 Container; Pod phase 保持 Running.
  - Never: 不重启 Container; Pod phase 保持 Running.

- 2个容器的 Pod，其中一个容器停止而另一个失败退出时，不同 restartPolicy 时的动作为
  - Always: 重启 Container; Pod phase 保持 Running.
  - OnFailure: 重启 Container; Pod phase 保持 Running.
  - Never: Pod phase 变成 Failed.

- 单容器的 Pod，容器内存不足（OOM），不同 restartPolicy 时的动作为
  - Always: 重启 Container; Pod phase 保持 Running.
  - OnFailure: 重启 Container; Pod phase 保持 Running.
  - Never: 记录失败事件; Pod phase 变成 Failed.
  
- Pod 还在运行，但磁盘不可访问时
  - 终止所有容器
  - Pod phase 变成 Failed
  - 如果 Pod 是由某个控制器管理的，则重新创建一个 Pod 并调度到其他 Node 运行

- Pod 还在运行，但由于网络分区故障导致 Node 无法访问
  - Node controller等待 Node 事件超时
  - Node controller 将 Pod phase 设置为 Failed.
  - 如果 Pod 是由某个控制器管理的，则重新创建一个 Pod 并调度到其他 Node 运行

## RestartPolicy

支持三种 RestartPolicy

- Always：当容器失效时，由Kubelet自动重启该容器。RestartPolicy的默认值。
- OnFailure：当容器终止运行且退出码不为0时由Kubelet重启。
- Never：无论何种情况下，Kubelet都不会重启该容器。
注意，这里的重启是指在 Pod 所在 Node 上面本地重启，并不会调度到其他 Node 上去。

## 环境变量
环境变量为容器提供了一些重要的资源，包括容器和 Pod 的基本信息以及集群中服务的信息等：
- hostname
HOSTNAME 环境变量保存了该 Pod 的 hostname。

- 容器和 Pod 的基本信息
  Pod 的名字、命名空间、IP 以及容器的计算资源限制等可以以 Downward API 的方式获取并存储到环境变量中。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-downwardapi-volume-example
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
  annotations:
    build: two
    builder: john-doe
spec:
  containers:
    - name: client-container
      image: registry.k8s.io/busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          if [[ -e /etc/podinfo/annotations ]]; then
            echo -en '\n\n'; cat /etc/podinfo/annotations; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
```
- 集群中服务的信息
  容器的环境变量中还可以引用容器运行前创建的所有服务的信息，比如默认的 kubernetes 服务对应以下环境变量：由于环境变量存在创建顺序的局限性（环境变量中不包含后来创建的服务），推荐使用 [DNS] 来解析服务。

## 镜像拉取策略

支持三种 ImagePullPolicy

- Always：不管本地镜像是否存在都会去仓库进行一次镜像拉取。校验如果镜像有变化则会覆盖本地镜像，否则不会覆盖。
- Never：只是用本地镜像，不会去仓库拉取镜像，如果本地镜像不存在则Pod运行失败。
- IfNotPresent：只有本地镜像不存在时，才会去仓库拉取镜像。ImagePullPolicy的默认值。

注意：

默认为 IfNotPresent，但 :latest 标签的镜像默认为 Always。
拉取镜像时 docker 会进行校验，如果镜像中的 MD5 码没有变，则不会拉取镜像数据。
生产环境中应该尽量避免使用 :latest 标签，而开发环境中可以借助 :latest 标签自动拉取最新的镜像。

## 资源限制
Kubernetes 通过 cgroups 限制容器的 CPU 和内存等计算资源，包括 requests（请求，调度器保证调度到资源充足的 Node 上，如果无法满足会调度失败）和 limits（上限）等：

- spec.containers[].resources.limits.cpu：CPU 上限，可以短暂超过，容器也不会被停止
- spec.containers[].resources.limits.memory：内存上限，不可以超过；如果超过，容器可能会被终止或调度到其他资源充足的机器上
- spec.containers[].resources.limits.ephemeral-storage：临时存储（容器可写层、日志以及 EmptyDir 等）的上限，超过后 Pod 会被驱逐
- spec.containers[].resources.requests.cpu：CPU 请求，也是调度 CPU 资源的依据，可以超过
- spec.containers[].resources.requests.memory：内存请求，也是调度内存资源的依据，可以超过；但如果超过，容器可能会在 Node 内存不足时清理
- spec.containers[].resources.requests.ephemeral-storage：临时存储（容器可写层、日志以及 EmptyDir 等）的请求，调度容器存储的依据

比如 nginx 容器请求 30% 的 CPU 和 56MB 的内存，但限制最多只用 50% 的 CPU 和 128MB 的内存：
```yaml
  
   apiVersion: v1
   kind: Pod
   metadata:
    labels:
    app: nginx
    name: nginx
   spec:
    containers:
    - image: nginx
    name: nginx
    resources:
      requests:
        cpu: "300m"
        memory: "56Mi"
      limits:
        cpu: "1"
        memory: "128Mi"
```

注意

- CPU 的单位是 CPU 个数，可以用 millicpu (m) 表示少于 1 个 CPU 的情况，如 500m = 500millicpu = 0.5cpu，而一个 CPU 相当于

  - AWS 上的一个 vCPU
  - GCP 上的一个 Core
  - Azure 上的一个 vCore
  - 物理机上开启超线程时的一个超线程
- 内存的单位则包括 E, P, T, G, M, K, Ei, Pi, Ti, Gi, Mi, Ki 等。
从 v1.10 开始，可以设置 kubelet ----cpu-manager-policy=static 为 Guaranteed（即 requests.cpu 与 limits.cpu 相等）Pod 绑定 CPU（通过 cpuset cgroups）。

## 健康检查

为了确保容器在部署后确实处在正常运行状态，Kubernetes 提供了两种探针（Probe）来探测容器的状态：

- LivenessProbe：探测应用是否处于健康状态，如果不健康则删除并重新创建容器。
- ReadinessProbe：探测应用是否启动完成并且处于正常服务状态，如果不正常则不会接收来自 Kubernetes Service 的流量，即将该Pod从Service的endpoint中移除。
Kubernetes 支持三种方式来执行探针：
- exec：在容器中执行一个命令，如果 命令退出码 返回 0 则表示探测成功，否则表示失败
- tcpSocket：对指定的容器 IP 及端口执行一个 TCP 检查，如果端口是开放的则表示探测成功，否则表示失败
- httpGet：对指定的容器 IP、端口及路径执行一个 HTTP Get 请求，如果返回的 状态码 在 [200,400) 之间则表示探测成功，否则表示失败

## Init Container

Pod 能够具有多个容器，应用运行在容器里面，但是它也可能有一个或多个先于应用容器启动的 Init 容器。Init 容器在所有容器运行之前执行（run-to-completion），常用来初始化配置。

如果为一个 Pod 指定了多个 Init 容器，那些容器会按顺序一次运行一个。 每个 Init 容器必须运行成功，下一个才能够运行。 当所有的 Init 容器运行完成时，Kubernetes 初始化 Pod 并像平常一样运行应用容器。

下面是一个 Init 容器的示例：

```yaml
  
   apiVersion: v1
   kind: Pod
   metadata:
    name: init-demo
   spec:
    containers:
    - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
    mountPath: /usr/share/nginx/html
     These containers are run during pod initialization
    initContainers:
    - name: install
    image: busybox
    command:
    - wget
    - "-O"
    - "/work-dir/index.html"
    - http://kubernetes.io
    volumeMounts:
    - name: workdir
    mountPath: "/work-dir"
    dnsPolicy: Default
    volumes:
    - name: workdir
    emptyDir: {}
```
因为 Init 容器具有与应用容器分离的单独镜像，使用 init 容器启动相关代码具有如下优势：

- 它们可以包含并运行实用工具，出于安全考虑，是不建议在应用容器镜像中包含这些实用工具的。
- 它们可以包含使用工具和定制化代码来安装，但是不能出现在应用镜像中。例如，创建镜像没必要 FROM 另一个镜像，只需要在安装过程中使用类似 sed、 awk、 python 或 dig 这样的工具。
- 应用镜像可以分离出创建和部署的角色，而没有必要联合它们构建一个单独的镜像。
- 它们使用 Linux Namespace，所以对应用容器具有不同的文件系统视图。因此，它们能够具有访问 Secret 的权限，而应用容器不能够访问。
- 它们在应用容器启动之前运行完成，然而应用容器并行运行，所以 Init 容器提供了一种简单的方式来阻塞或延迟应用容器的启动，直到满足了一组先决条件。
- Init 容器的资源计算，选择一下两者的较大值：

所有 Init 容器中的资源使用的最大值
- Pod 中所有容器资源使用的总和

Init 容器的重启策略：
- 如果 Init 容器执行失败，Pod 设置的 restartPolicy 为 Never，则 pod 将处于 fail 状态。否则 Pod 将一直重新执行每一个 Init 容器直到所有的 Init 容器都成功。
- 如果 Pod 异常退出，重新拉取 Pod 后，Init 容器也会被重新执行。所以在 Init 容器中执行的任务，需要保证是幂等的。

## 容器生命周期钩子
容器生命周期钩子（Container Lifecycle Hooks）监听容器生命周期的特定事件，并在事件发生时执行已注册的回调函数。支持两种钩子：

- postStart： 容器创建后立即执行，注意由于是异步执行，它无法保证一定在 ENTRYPOINT 之前运行。如果失败，容器会被杀死，并根据 RestartPolicy 决定是否重启
- preStop：容器终止前执行，常用于资源清理。如果失败，容器同样也会被杀死
而钩子的回调函数支持两种方式：
- exec：在容器内执行命令，如果命令的退出状态码是 0 表示执行成功，否则表示失败
- httpGet：向指定 URL 发起 GET 请求，如果返回的 HTTP 状态码在 [200, 400) 之间表示请求成功，否则表示失败
postStart 和 preStop 钩子示例：
```yaml
   apiVersion: v1
   kind: Pod
   metadata:
    name: lifecycle-demo
   spec:
    containers:
    - name: lifecycle-demo-container
    image: nginx
    lifecycle:
    postStart:
    httpGet:
    path: /
    port: 80
    preStop:
    exec:
    command: ["/usr/sbin/nginx","-s","quit"]
 

```

## 调度到指定的 Node 上