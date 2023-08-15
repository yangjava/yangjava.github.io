---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus源码规则


## rule
prometheus的rule有两类：

AlertingRule: 告警规则；
RecordingRule: 表达式规则，用于产生新的指标；