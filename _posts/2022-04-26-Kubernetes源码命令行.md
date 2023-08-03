---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes源码kubectl
在维护Kubernetes系统集群时，kubectl应该是最常用的工具之一。从Kubernetes架构设计的角度来看，kubectl工具是Kubernetes API Server的客户端。

它的主要工作是向Kubernetes API Server发起HTTP请求。Kubernetes是一个完全以资源为中心的系统，而kubectl会通过发起HTTP请求来操纵这些资源（即对资源进行CRUD操作），以完全控制Kubernetes系统集群。

## kubectl
kubectl 是 Kubernetes 的命令行工具（CLI），是 Kubernetes 用户和管理员必备的管理工具。 kubectl 提供了大量的子命令，方便管理 Kubernetes 集群中的各种功能。
- kubectl -h 查看子命令列表
- kubectl options 查看全局选项
- kubectl <command> --help 查看子命令的帮助
- kubectl [command] [PARAMS] -o=<format> 设置输出格式（如 json、yaml、jsonpath 等）
- kubectl explain [RESOURCE] 查看资源的定义

## 配置
使用 kubectl 的第一步是配置 Kubernetes 集群以及认证方式，包括
- cluster 信息：Kubernetes server 地址
- 用户信息：用户名、密码或密钥
- Context：cluster、用户信息以及 Namespace 的组合

```yaml
   kubectl config set-credentials myself --username=admin --password=secret
   kubectl config set-cluster local-server --server=http://localhost:8080
   kubectl config set-context default-context --cluster=local-server --user=myself --namespace=default
   kubectl config use-context default-context
   kubectl config view
```

## 常用命令格式
- 创建：kubectl run <name> --image=<image> 或者 kubectl create -f manifest.yaml
- 查询：kubectl get <resource>
- 更新 kubectl set 或者 kubectl patch
- 删除：kubectl delete <resource> <name> 或者 kubectl delete -f manifest.yaml
- 查询 Pod IP：kubectl get pod <pod-name> -o jsonpath=''
- 容器内执行命令：kubectl exec -ti <pod-name> sh
- 容器日志：kubectl logs [-f] <pod-name>
- 导出服务：kubectl expose deploy <name> --port=80
- Base64 解码：kubectl get secret SECRET -o go-template=''

## kubectl命令行参数详解
Kubernetes官方提供了命令行工具（CLI），用户可以通过kubectl以命令行交互的方式与Kubernetes API Server进行通信，通信协议使用HTTP/JSON。kubectl的命令主要分为8个种类，分别介绍如下。
- Basic Commands （Beginner ）：基础命令（初级）。
- Basic Commands （Intermediate ）：基础命令（中级）。
- Deploy Commands ：部署命令。
- Cluster Management Commands ：集群管理命令。
- Troubleshooting and Debugging Commands ：故障排查和调试命令。
- Advanced Commands ：高级命令。
- Settings Commands ：设置命令。
- Other Commands ：其他命令。

## Cobra命令行参数解析
Cobra是一个创建强大的现代化CLI命令行应用程序的Go语言库，也可以用来生成应用程序的文件。很多知名的开源软件都使用Cobra实现其CLI部分，例如Istio、Docker、Etcd等。Cobra提供了如下功能。
- 支持子命令行（Subcommand）模式，如app server，其中server是app命令的子命令参数。
- 完全兼容posix命令行模式。
- 支持全局、局部、串联的命令行参数（Flag）。
- 轻松生成应用程序和命令。
- 如果命令输入错误，将提供智能建议，例如输入app srver，当srver参数不存在时，Cobra会智能提示用户是否应输入app server。
- 自动生成命令（Command）和参数（Flag）的帮助信息。
- 自动生成详细的命令行帮助（Help）信息，如app help。
- 自动识别-h、--help flag。
- 提供bash环境下的命令自动完成功能。
- 自动生成应用程序的帮助手册。
- 支持命令行别名。
- 自定义帮助和使用信息。
- 可与viper配置库紧密结合。





