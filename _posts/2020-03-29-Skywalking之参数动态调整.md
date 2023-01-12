---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
# Skywalking之参数动态调正
如何才能做到通过后台，动态控制agent端的采样率、链路跨度等配置信息呢？正巧，Skywalking提供了动态更新功能

## 背景

按照Skywalking官网的客户端搭建方式，基本采取配置agent.properties文件，或者通过java -D 带参数方式（也可以直接使用环境变量进行配置），这些操作办法都属于静态配置。如果在业务高峰期，可能需要调整采样率 agent.sample_n_per_3_secs 的数值，只能通过重新启动agent方式更新配置信息。

那么如何才能做到通过后台，动态控制agent端的采样率、链路跨度等配置信息呢？正巧，Skywalking提供了动态更新功能，