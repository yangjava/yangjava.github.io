---
layout: post
categories: [Docker]
description: none
keywords: Docker
---
# Docker源码registry
内容简介：官网上介绍Registry是一个无状态，高度可扩展的服务器端应用程序，可存储并允许您分发Docker镜像。Registry是Apache开源的。我们暂且只要知道Registry是负责存储Docker镜像的就可以了。

## Registry是什么：

官网上介绍Registry是一个无状态，高度可扩展的服务器端应用程序，可存储并允许您分发 Docker 镜像。Registry是Apache开源的。

我们暂且只要知道Registry是负责存储Docker镜像的就可以了。

## 说明：

现在网络上有一些关于registry 启动的源码分析，由于感觉不太详细并且讲解不够清晰，所以自己重新记录一下，以便自己以后快速回顾，也希望对想要阅读源码的同学有所帮助。由于从未发过博客，如有发现错误或者有疑问，都欢迎随时联系我。

一、v1基本已经弃用了

1、镜像是以layer的形式存储，v1下载一个layer后回去找parent layer串行下载，速度当然上不去。 而v2增加了mainfest（manifest主要保存镜像layer信息，及一些校验值digest），先获取mainfest，然后并行下载各层。

2、v1以 python 编写，v2则为go。

## 准备知识

下面简单的说了一下Cobra和context ，对 go 不是特别熟的同学可以看下（大神跳过）。

Cobra：

Cobra既是一个用来创建强大的现代CLI命令行的golang库，也是一个生成程序应用和命令行文件的程序。regisry用Cobra管理应用程序。程序入口的RootCmd就是使用了Cobra提供的方法。

context：

①可以控制goroutine的生命期。实现了channel ＋ select不容易实现的场景，例如由一个请求衍生出的各个 goroutine 之间需要满足一定的约束关系，以实现一些诸如有效期，中止routine树②多goroutine 传值，Context 在多个goroutine 中是安全的。

## 整体启动流程

简单说registry启动过程就是读取配置文件，做相应配置（如认证方式，存储配置，缓存配置，通知方式配置等），配置路由函数，最后启动服务程序的过程。

1、启动命令

registry运行在容器中，查看Dockerfile可知启动命令为：
```
“registry serve /etc/docker/registry/config.yml”
```
serve ：启动命令参数，告知registry 执行启动流程。（registry 的另一个参数是garbage-collect，用于垃圾回收）

Docker registryV2-启动过程 源码分析


2、入口程序

程序路径：/cmd/registry/main.go。

Docker registryV2-启动过程 源码分析


3、根命令RootCmd

①registry以命令树的形式管理命令参数，通过init()函数为RootCmd添加了ServeCmd、GCCmd两个命令，分别对应server和garbage-collect两个子命令，server个garbage-collect。

②init函数是用于程序执行前自动执行，做一些初始化的操作，比如初始化包里的变量等。

③ 程序路径：registry/root.go

Docker registryV2-启动过程 源码分析


4、regisry启动命令ServeCmd

①Cobra的Execute方法会为我们通过agrs在命令树中寻找到要执行的命令，这里是ServeCmd。

②dcontext.WithVersion初始化一个goroutine上下文，并携带version信息。

③resolveConfiguration(args)方法读取配置文件放入config中。

④NewRegistry(ctx, config)方法实例化一个registry。

⑤对Prometheus提供支持

⑥ registry.ListenAndServe()方法，完成registry服务的启动。

⑦程序路径：/registry/registry.go

Docker registryV2-启动过程 源码分析
```
```

关键函数介绍

1、NewRegistry创建registry实例

①主要设置了日志规则、初始化app，返回registry实例

②程序路径：/registry/registry.go

Docker registryV2-启动过程 源码分析


2、NewApp方法

NewApp通过相应配置返回一个app实例，处理客户端的http请求，进行路由规则注册，后端存储配置，重定向配置，pull缓存配置等等。

程序路径：/registry/handlers/app.go

Docker registryV2-启动过程 源码分析
①v2.RouterWithPrefix 设置url(程序路径：/registry/app/v2/routes.go)

Docker registryV2-启动过程 源码分析


route详情在routeDescriptors中(程序路径：/registry/app/v2/descriptors.go)

Docker registryV2-启动过程 源码分析


②app.register register设置各个接口的处理函数

Docker registryV2-启动过程 源码分析


以manifest为例（其他的tag、blob思路相同）。manifestDispatcher实现了基础的manifest查询、下载规则，根据权限是否设置上传、删除规则。(程序路径：/registry/handlers/manifest.go)

Docker registryV2-启动过程 源码分析


③利用配置文件中配置的storage driver创建registry的storage driver对象

Docker registryV2-启动过程 源码分析


④接下来，为registry的App配置secret、events、 redis 和loghook

Docker registryV2-启动过程 源码分析


⑤根据配置文件中storage节点下delete的配置，确定否可以通过摘要删除图像blob和manifest。它默认为false

Docker registryV2-启动过程 源码分析


⑥根据配置文件中storage节点下redirect的配置，配置存储位置重定向，默认是启用的。当配置了远程存储的时候需要启用，以达到重定向到存储服务器。

Docker registryV2-启动过程 源码分析


⑦根据配置文件中storage节点下cache的配置，配置缓存，默认使用内存，可以配置成redis等。

Docker registryV2-启动过程 源码分析


五、总结

registry源码还是很值得读的，通过读源码可以学习go语言，能透彻的明白镜像是怎么上传及下载的。而且registry的源码还不是特别大。

