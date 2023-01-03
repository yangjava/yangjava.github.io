---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes源码kubectl
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










