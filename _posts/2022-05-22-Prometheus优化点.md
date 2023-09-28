---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus优化点
Prometheus 用良好的用户接口掩盖了内部复杂实现。深入其中，我们会看到时序数据存储、大规模数据采集、时间线高基数、数据搅动、长周期查询、历史数据归档等等。像潜藏在宁静湖面下磨牙吮血的鳄鱼，静待不小心掉进坑中的 SRE 们。

## 概念解读

### 时间线（Time Series）
时间线的概念在Prometheus中非常重要，它表示一组不相同的label/value 的集合。比如temperature{city="BJ",country="CN"} 和 temperature{city="SH",country="CN"}就是两条不同的时间线。

因为其中的 city这个label对应的值不同；temperature{city="BJ",country="CN"}和 humidity{city="BJ",country="CN"} 也是两条不相同的时间线，因为在Prometheus中，指标名称也对应一个特殊的label __name__。时间线中的label会被索引以加速查询，所以时间线越多存储压力越大。

### 时间线高基数（High Cardinality）
如果指标设计不合理，label 取值范围宽泛，如 URL，订单号，哈希值等，会造成写入的时间线数量难以预估，会发生爆炸式增长，为采集、存储带来巨大挑战。对于这种情况，我们称之为时间线高基数或者时间线爆炸。时间线基数过高带来的影响是全方位的，如采集压力大，网络和磁盘 IO 压力大，存储成本上升，查询响应迟缓等。

高基数场景常见应对思路有两种：依靠时序存储本身的能力，或更优秀的架构来硬抗压力；或使用预聚合/预计算等手段来主动降低基数。

### 高搅动率（High Churn Rate）
高搅动率是高基数的特殊场景之一，如果时间线的“寿命”很短，经常被汰换，这种场景我们称之为高搅动率。比如 k8s 场景下，deployment 每次创建新的 pod 时，都会产生新 pod name，即旧 pod 相关时间线被淘汰，新 pod 相关时间线被产生出来。如果经常性发生重启（或类似场景），那么可能从某个时间点来看，时间线并不多，但在较长时间范围来看，时间线总量可能就很庞大。换句话说，在高搅动率场景下，“活跃”的时间线不一定很多，但累积时间线总数非常庞大，所以对时间跨度较大的查询影响尤其严重。














