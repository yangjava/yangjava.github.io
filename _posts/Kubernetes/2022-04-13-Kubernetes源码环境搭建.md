---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes源码环境搭建
最近需要学习k8s源码，为了阅读源码的方便，打算在Windows下使用Go配置k8s源码阅读环境。

## Kubernetes源码

下载k8s源码：[https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)

## 


```text
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

