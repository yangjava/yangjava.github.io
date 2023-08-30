---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink流式编程
利用DataStream API开发流式应用。例如如何定义数据源、数据转换、数据输出等操作，以及每种操作在Flink中如何进行拓展。

## DataStream编程模型
Flink基于Google提出的DataFlow模型，实现了支持原生数据流处理的计算引擎。Flink中定义了DataStream API让用户灵活且高效地编写Flink流式应用。

DataStream API主要可为分为三个部分
- DataSource模块
Sources模块主要定义了数据接入功能，主要是将各种外部数据接入至Flink系统中，并将接入数据转换成对应的DataStream数据集。

- Transformation模块
Transformation模块定义了对DataStream数据集的各种转换操作，例如进行map、filter、windows等操作。

- DataSink模块
将结果数据通过DataSink模块写出到外部存储介质中，例如将数据输出到文件或Kafka消息中间件等。







