---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
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