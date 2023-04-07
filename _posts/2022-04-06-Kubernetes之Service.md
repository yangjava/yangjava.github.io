---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes组件
身病多思虑，亦读神农经。——张籍《卧疾》    
Kubernetes 在设计之初就充分考虑了针对容器的服务发现与负载均衡机制，提供了 Service 资源，并通过 kube-proxy 配合 cloud provider 来适应不同的应用场景。随着 kubernetes 用户的激增，用户场景的不断丰富，又产生了一些新的负载均衡机制。

## 服务发现与负载均衡
目前，kubernetes 中的负载均衡大致可以分为以下几种机制，每种机制都有其特定的应用场景：
- Service：直接用 Service 提供 cluster 内部的负载均衡，并借助 cloud provider 提供的 LB 提供外部访问
- Ingress Controller：还是用 Service 提供 cluster 内部的负载均衡，但是通过自定义 Ingress Controller 提供外部访问
- Service Load Balancer：把 load balancer 直接跑在容器中，实现 Bare Metal 的 Service Load Balancer
- Custom Load Balancer：自定义负载均衡，并替代 kube-proxy，一般在物理部署 Kubernetes 时使用，方便接入公司已有的外部服务

## Service
Service 是对一组提供相同功能的 Pods 的抽象，并为它们提供一个统一的入口。借助 Service，应用可以方便的实现服务发现与负载均衡，并实现应用的零宕机升级。Service 通过标签来选取服务后端，一般配合 Replication Controller 或者 Deployment 来保证后端容器的正常运行。这些匹配标签的 Pod IP 和端口列表组成 endpoints，由 kube-proxy 负责将服务 IP 负载均衡到这些 endpoints 上。
Service 有四种类型：

- ClusterIP：默认类型，自动分配一个仅 cluster 内部可以访问的虚拟 IP
- NodePort：在 ClusterIP 基础上为 Service 在每台机器上绑定一个端口，这样就可以通过 <NodeIP>:NodePort 来访问该服务。如果 kube-proxy 设置了 --nodeport-addresses=10.240.0.0/16（v1.10 支持），那么仅该 NodePort 仅对设置在范围内的 IP 有效。
- LoadBalancer：在 NodePort 的基础上，借助 cloud provider 创建一个外部的负载均衡器，并将请求转发到 <NodeIP>:NodePort
- ExternalName：将服务通过 DNS CNAME 记录方式转发到指定的域名（通过 spec.externlName 设定）。需要 kube-dns 版本在 1.7 以上。

另外，也可以将已有的服务以 Service 的形式加入到 Kubernetes 集群中来，只需要在创建 Service 的时候不指定 Label selector，而是在 Service 创建好后手动为其添加 endpoint。

## Service 定义
Service 的定义也是通过 yaml 或 json，比如下面定义了一个名为 nginx 的服务，将服务的 80 端口转发到 default namespace 中带有标签 run=nginx 的 Pod 的 80 端口
```yaml
 apiVersion: v1
   kind: Service
   metadata:
    labels:
    run: nginx
    name: nginx
    namespace: default
   spec:
    ports:
    - port: 80
    protocol: TCP
    targetPort: 80
    selector:
    run: nginx
    sessionAffinity: None
    type: ClusterIP
```

```shell
   service 自动分配了 Cluster IP 10.0.0.108
   $ kubectl get service nginx
   NAME CLUSTER-IP EXTERNAL-IP PORT(S) AGE
   nginx 10.0.0.108 <none> 80/TCP 18m
    自动创建的 endpoint
   $ kubectl get endpoints nginx
   NAME ENDPOINTS AGE
   nginx 172.17.0.5:80 18m
    Service 自动关联 endpoint
   $ kubectl describe service nginx
   Name: nginx
   Namespace: default
   Labels: run=nginx
   Annotations: <none>
   Selector: run=nginx
   Type: ClusterIP
   IP: 10.0.0.108
   Port: <unset> 80/TCP
   Endpoints: 172.17.0.5:80
   Session Affinity: None
   Events: <none>
```

当服务需要多个端口时，每个端口都必须设置一个名字
```yaml
  kind: Service
   apiVersion: v1
   metadata:
    name: my-service
   spec:
    selector:
    app: MyApp
    ports:
    - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
    - name: https
    protocol: TCP
    port: 443
    targetPort: 9377

```
协议
Service、Endpoints 和 Pod 支持三种类型的协议：
- TCP（Transmission Control Protocol，传输控制协议）是一种面向连接的、可靠的、基于字节流的传输层通信协议。
- UDP（User Datagram Protocol，用户数据报协议）是一种无连接的传输层协议，用于不可靠信息传送服务。
- SCTP（Stream Control Transmission Protocol，流控制传输协议），用于通过IP网传输SCN（Signaling Communication Network，信令通信网）窄带信令消息。

## 不指定 Selectors 的服务
在创建 Service 的时候，也可以不指定 Selectors，用来将 service 转发到 kubernetes 集群外部的服务（而不是 Pod）。目前支持两种方法
自定义 endpoint，即创建同名的 service 和 endpoint，在 endpoint 中设置外部服务的 IP 和端口
```yaml
  
   kind: Service
   apiVersion: v1
   metadata:
    name: my-service
   spec:
    ports:
    - protocol: TCP
    port: 80
 
    targetPort: 9376
   ---
   kind: Endpoints
   apiVersion: v1
   metadata:
    name: my-service
   subsets:
    - addresses:
    - ip: 1.2.3.4
    ports:
    - port: 9376
```
通过 DNS 转发，在 service 定义中指定 externalName。此时 DNS 服务会给 <service-name>.<namespace>.svc.cluster.local 创建一个 CNAME 记录，其值为 my.database.example.com。并且，该服务不会自动分配 Cluster IP，需要通过 service 的 DNS 来访问。
```yaml
   kind: Service
   apiVersion: v1
   metadata:
    name: my-service
    namespace: default
   spec:
    type: ExternalName
    externalName: my.database.example.com
 

```
注意：Endpoints 的 IP 地址不能是 127.0.0.0/8、169.254.0.0/16 和 224.0.0.0/24，也不能是 Kubernetes 中其他服务的 clusterIP。

# Kubernetes组件
力学如力耕，勤惰尔自知。——刘过《书院》   
Ingress 是 k8s 资源对象，用于对外暴露服务，该资源对象定义了不同主机名（域名）及 URL 和对应后端 Service（k8s Service）的绑定，根据不同的路径路由 http 和 https 流量。

## nodePort，LoadBalancer 和 Ingress的关系
向 k8s 集群外部暴露服务的方式有三种： nodePort，LoadBalancer 和 Ingress。

nodePort 方式在服务变多的情况下会导致节点要开的端口越来越多，不好管理。

LoadBalancer 更适合结合云提供商的 LB 来使用，但是在 LB 越来越多的情况下对成本的花费也是不可小觑。

Ingress 是 k8s 官方提供的用于对外暴露服务的方式，也是在生产环境用的比较多的方式，一般在云环境下是 LB + Ingress Ctroller 方式对外提供服务，可以使用 Ingress 来使内部服务暴露到集群外部去，它为你节省了宝贵的静态 IP，因为你不需要声明多个 LoadBalancer 服务了，此次，它还可以进行更多的额外配置。

## ingress Controller
Ingress Contoller 是一个 pod 服务，封装了一个 web 前端负载均衡器，同时在其基础上实现了动态感知 Ingress 并根据 Ingress 的定义动态生成 前端 web 负载均衡器的配置文件，比如 Nginx Ingress Controller 本质上就是一个 Nginx，只不过它能根据 Ingress 资源的定义动态生成 Nginx 的配置文件，然后动态 Reload。
所以，总的来说要使用 Ingress，得先部署 Ingress Controller 实体（相当于前端 Nginx），然后再创建 Ingress （相当于 Nginx 配置的 k8s 资源体现），Ingress Controller 部署好后会动态检测 Ingress 的创建情况生成相应配置。Ingress Controller 的实现有很多种：有基于 Nginx 的，也有基于 HAProxy的，还有基于 OpenResty 的 Kong Ingress Controller 等，更多 Controller 见：https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/，本文使用基于 Nginx 的 Ingress Controller：ingress-nginx。

## Pod与Ingress的关系






