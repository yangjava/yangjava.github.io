---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes组件
愿尔一祝后，读书日日忙。——杜牧《冬至日寄小侄阿宜诗》    

## 如何利用Kind安装K8S

Kind是Kubernetes In Docker的缩写，顾名思义，看起来是把k8s放到docker的意思。没错，kind创建k8s集群的基本原理就是：提前准备好k8s节点的镜像，通过docker启动容器，来模拟k8s的节点，从而组成完整的k8s集群。需要注意，kind创建的集群仅可用于开发、学习、测试等，不能用于生产环境。
Github 地址：https://github.com/kubernetes-sigs/kind
kind 能在 1 分钟之内就启动一个非常轻量级的 k8s 集群。之所以如此之快，得益于其基于其把整个 k8s 集群用到的组件都封装在了 Docker 容器里，构建一个 k8s 集群就是启动一个 Docker 容器

### kind启动流程
- 查看本地上是否存在一个基础的安装镜像，默认是 kindest/node:v1.13.4，这个镜像里面包含了需要安装的所有东西，包括了 kubectl、kubeadm、kubelet 二进制文件，以及安装对应版本 k8s 所需要的镜像，都以 tar 压缩包的形式放在镜像内的一个路径下
- 准备你的 node，这里就是做一些启动容器、解压镜像之类的工作
- 生成对应的 kubeadm 的配置，之后通过 kubeadm 安装，安装之后还会做另外的一些操作，比如像我刚才仅安装单节点的集群，会帮你删掉 master 节点上的污点，否则对于没有容忍的 pod 无法部署。
- 启动完毕

### Linux系统安装

Kind 的安装不包括 kubectl和docker，可以实现在服务器安装Kubectl和Docker
#### docker安装

```text
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum clean all && yum makecache fast
yum -y install docker-ce
systemctl start docker
```
#### kubelet安装

```text
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/linux/amd64/kubectl &&\
chmod +x ./kubectl &&\
mv ./kubectl /usr/bin/kubectl
```

#### Kind安装

```text
wget https://github.com/kubernetes-sigs/kind/releases/download/0.2.1/kind-linux-amd64
mv kind-linux-amd64 kind
chmod +x kind
mv kind /usr/local/bin
```

#### 使用
```text
kind creates and manages local Kubernetes clusters using Docker container 'nodes'

Usage:
  kind [command]

Available Commands:
  build       Build one of [node-image]
  completion  Output shell completion code for the specified shell (bash, zsh or fish)
  create      Creates one of [cluster]
  delete      Deletes one of [cluster]
  export      Exports one of [kubeconfig, logs]
  get         Gets one of [clusters, nodes, kubeconfig]
  help        Help about any command
  load        Loads images into nodes
  version     Prints the kind CLI version

Flags:
  -h, --help              help for kind
      --loglevel string   DEPRECATED: see -v instead
  -q, --quiet             silence all stderr output
  -v, --verbosity int32   info log verbosity
      --version           version for kind

Use "kind [command] --help" for more information about a command.
```
简单说下几个比较常用选项的含义：

build：用来从 Kubernetes 源代码构建一个新的镜像。

create：创建一个 Kubernetes 集群。

delete：删除一个 Kubernetes 集群。

get：可用来查看当前集群、节点信息以及 Kubectl 配置文件的地址。

load：从宿主机向 Kubernetes 节点内导入镜像。

#### 如何通过kind新建k8s集群？
kubectl是与k8s交互的客户端命令工具，因此需要先安装此工具。
```text
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```
然后通过一行命令就能够快速的创建k8s集群：
```text
kind create cluster --name k8s-1
```

创建K8S集群
```text
 kind create cluster --name k8s-1
Creating cluster "k8s-1" ...
 ✓ Ensuring node image (kindest/node:v1.18.2) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-k8s-1"
You can now use your cluster with:

kubectl cluster-info --context kind-k8s-1

Have a nice day! 👋
```
至此已得到一个可用的k8s集群了：

使用 kind create cluster 安装，是没有指定任何配置文件的安装方式。从安装打印出的输出来看，分为4步：

- 查看本地上是否存在一个基础的安装镜像，默认是 kindest/node:v1.13.4，这个镜像里面包含了需要安装的所有东西，包括了 kubectl、kubeadm、kubelet 二进制文件，以及安装对应版本 k8s 所需要的镜像，都以 tar 压缩包的形式放在镜像内的一个路径下
- 准备你的 node，这里就是做一些启动容器、解压镜像之类的工作
- 生成对应的 kubeadm 的配置，之后通过 kubeadm 安装，安装之后还会做另外的一些操作，比如像我刚才仅安装单节点的集群，会帮你删掉 master 节点上的污点，否则对于没有容忍的 pod 无法部署。
- 启动完毕

```text
kubectl get nodes
NAME                  STATUS   ROLES    AGE   VERSION
k8s-1-control-plane   Ready    master   64s   v1.18.2
```
先查看多出来的docker镜像和容器：
```text
docker images
REPOSITORY     TAG       IMAGE ID       CREATED         SIZE
kindest/node   <none>    de6eb7df13da   2 years ago     1.25GB
```
查看当前集群的运行情况
```text
kubectl get po -n kube-system

NAME                                          READY   STATUS    RESTARTS   AGE
coredns-66bff467f8-448sh                      1/1     Running   5          8d
coredns-66bff467f8-47vlh                      1/1     Running   4          8d
etcd-k8s-1-control-plane                      1/1     Running   3          8d
kindnet-4fghr                                 1/1     Running   5          8d
kube-apiserver-k8s-1-control-plane            1/1     Running   3          8d
kube-controller-manager-k8s-1-control-plane   1/1     Running   3          8d
kube-proxy-rb5bn                              1/1     Running   3          8d
kube-scheduler-k8s-1-control-plane            1/1     Running   3          8d
```
默认方式启动的节点类型是 control-plane 类型，包含了所有的组件。包括2 * coredns、etcd、api-server、controller-manager、kube-proxy、sheduler。

基本上，kind 的所有秘密都在那个基础镜像中。下面是基础容器内部的 /kind 目录，在 bin 目录下安装了 kubelet、kubeadm、kubectl 这些二进制文件，images 下面是镜像的 tar 包，kind 在启动基础镜像后会执行一遍 docker load 操作将这些 tar 包导入。


### 创建过程

先获取镜像kindest/node:v1.21.1，然后启动容器myk8s-01-control-plane，启动的容器就是这个k8s集群的master节点，显然此集群只有master节点。
同时，kind create cluster命令还是将此新建的k8s集群的连接信息写入当前用户(root)的kubectl配置文件中，从而让kubectl能够与集群交互。

### kind程序的完整用法

. kind create cluster
. –image 指定node镜像名称，默认是kindest/node
. –name 指定创建集群的名称
. –config kind-example-config.yaml
. –kubeconfig string 指定生成的kubeconfig的文件路径。默认在$KUBECONFIG or $HOME/.kube/config
. 常用：kind create cluster --config=xxxcfg --name=xxxname
. kind delete cluster xxxx
. kind get clusters：查看kind创建所有的k8s集群
. kind get kubeconfig --name 集群name【必须填写–name参数，默认name是kind】
. kind completion
. kind export kubeconfig --name=xxx --kubeconfig=kubeconfigpath   导出kind中的k8s集群的kubeconfig连接配置文件

### 实践创建多节点的集群


官网相关：
https://blog.csdn.net/easylife206/article/details/101091751
官方文档：https://kind.sigs.k8s.io/
GitHub主页：https://github.com/kubernetes-sigs/kind/
https://blog.csdn.net/pythonxxoo/article/details/123321193
https://blog.csdn.net/qianghaohao/article/details/108020937