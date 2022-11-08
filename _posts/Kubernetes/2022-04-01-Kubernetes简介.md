---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes简介
黑发不知勤学早，白首方悔读书迟。——颜真卿《劝学诗》
官网：Kubernetes是用于自动部署，扩展和管理容器化应用程序的开源系统。
**Kubernetes, also known as K8s, is an open-source system for automating deployment, scaling, and management of containerized applications.**

## 什么是Kubernetes
Kubernetes (通常称为K8s，K8s是将8个字母“ubernete”替换为“8”的缩写) 是用于自动部署、扩展和管理容器化（containerized）应用程序的开源系统。Google设计并捐赠给Cloud Native Computing Foundation（今属Linux基金会）来使用的。  

它旨在提供“跨主机集群的自动部署、扩展以及运行应用程序容器的平台”。它支持一系列容器工具, 包括Docker等。CNCF于2017年宣布首批Kubernetes认证服务提供商（KCSPs），包含IBM、MIRANTIS、华为、inwinSTACK迎栈科技等服务商。

## Kubernetes发展史

Kubernetes (希腊语"舵手" 或 "飞行员") 由Joe Beda，Brendan Burns和Craig McLuckie创立，并由其他谷歌工程师，包括Brian Grant和Tim Hockin进行加盟创作，并由谷歌在2014年首次对外宣布 。它的开发和设计都深受谷歌的Borg系统的影响，它的许多顶级贡献者之前也是Borg系统的开发者。在谷歌内部，Kubernetes的原始代号曾经是Seven，即星际迷航中友好的Borg(博格人)角色。Kubernetes标识中舵轮有七个轮辐就是对该项目代号的致意。

Kubernetes v1.0于2015年7月21日发布。随着v1.0版本发布，谷歌与Linux 基金会合作组建了Cloud Native Computing Foundation (CNCF)并把Kubernetes作为种子技术来提供。

Rancher Labs在其Rancher容器管理平台中包含了Kubernetes的发布版。Kubernetes也在很多其他公司的产品中被使用，比如Red Hat在OpenShift产品中，CoreOS的Tectonic产品中， 以及IBM的IBM云私有产品中。

## Kubernetes 特性

① 自我修复

在节点故障时，重新启动失败的容器，替换和重新部署，保证预期的副本数量；杀死健康检查失败的容器，并且在未准备好之前不会处理用户的请求，确保线上服务不中断。

② 弹性伸缩

使用命令、UI或者基于CPU使用情况自动快速扩容和缩容应用程序实例，保证应用业务高峰并发时的高可用性；业务低峰时回收资源，以最小成本运行服务。

③ 自动部署和回滚

K8S采用滚动更新策略更新应用，一次更新一个Pod，而不是同时删除所有Pod，如果更新过程中出现问题，将回滚更改，确保升级不影响业务。

④ 服务发现和负载均衡

K8S为多个容器提供一个统一访问入口（内部IP地址和一个DNS名称），并且负载均衡关联的所有容器，使得用户无需考虑容器IP问题。

⑤ 机密和配置管理

管理机密数据和应用程序配置，而不需要把敏感数据暴露在镜像里，提高敏感数据安全性。并可以将一些常用的配置存储在K8S中，方便应用程序使用。

⑥ 存储编排

挂载外部存储系统，无论是来自本地存储，公有云，还是网络存储，都作为集群资源的一部分使用，极大提高存储使用灵活性。

⑦ 批处理

提供一次性任务，定时任务；满足批量数据处理和分析的场景。


# 参考资料

Kubernetes 官方文档：[https://kubernetes.io/zh/](https://kubernetes.io/zh/)

