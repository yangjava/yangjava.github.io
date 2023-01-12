---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
应用上了K8S，wss内存持续上升，比rss内存高出几倍，哪位大神遇到过类似的情况吗？

首先要了解这些指标具体含义，以我遇到的为例，我们使用的性能监控是k8s原生的kube-prometheus技术。（这个可以从运维同学获取）
这里的rss约等于linux的rss，可以认为你程序常驻内存使用。
这里的wss约等于 rss+cache，里面还有核心内存以及脏内存。具体查询官方文档。
那么可以推测，rss没有增高或者说rss会自己内存回收，wss却一直增高并且与rss分离度高的原因是cache导致。
在容器里可以 free查看cache变化，不过如果没有运维协助看到的是宿主机的cache，我是用压测脚本短时间内推高wss的。发现缺失cache增高了，然后结合常见的缓存增长原因，定位到我的原因是日志等级低了，请求时大量io导致了缓存急速飙升。调整日志等级后重新发布果然解决了问题。