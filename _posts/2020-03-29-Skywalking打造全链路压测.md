---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking打造全链路压测

##
SKywalking 是基于 java agent 技术打造的，而 java agent 又非常的适合开发全链路压测产品，那么，是否可以借助 SKywalking 的现有能力开发出全链路压测呢？答案是可以的。

全链路压测的核心问题是压测的过程中不能有脏数据，当影子流量进入容器，这些流量不能进入正式的数据库。通常的做法是，例如在执行 SQL 的时候，判断是否是影子流量，如果是，则更换 SQL 数据源，即不能在正式库中执行影子 SQL。

基于 SKywalking 的目前的实现，我们只需要对一个类实现多个插件即可，并将这些插件进行包装，基于过滤器模式进行串联，实现对一个类的 压测增强 和 全链路Trace增强。