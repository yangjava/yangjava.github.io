---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
## Open Tracing介绍

### 概述

- Open Tracing通过提供平台无关、厂商无关的API，使得开发人员能够方便的添加（或更换）追踪系统的实现。Open Tracing最核心的概念就是Trace。

### Trace

- 在广义上，一个trace代表了一个事务或者流程（在分布式）系统中的执行过程。在Open Tracing标准中，trace是多个span的一个有向无环图（DAG），每一个span代表trace中被命名并计时的连续性的执行片段。

- 例如客户端发起一次请求，就可以认为是一次Trace。将上面的图通过Open Tracing的语义修改完之后做可视化，得到下面的图：

- 图中的每一个色块其实就是一个span。

### Span的概念

- 一个Span代表系统中具有开始时间和执行时长的逻辑运行单元。span之间通过嵌套或者顺序排列建立逻辑因果关系。

- Span里面的信息包含：操作的名字，开始时间和结束时间，可以携带多个`key:value`构成的Tags（key必须是String，value可以是String、Boolean或者数字等），还可以携带Logs信息（不一定所有的实现都支持），也必须是`key:value`形式。

- 下面的例子是一个Trace，里面有8个Span：

```latex
       [Span A]  ←←←(the root span)        
           |
    +------+------+   
    |             |
[Span B]      [Span C] ←←←(Span C 是    Span A 的孩子节点, ChildOf)   
    |            |
[Span D]     +---+-------+
             |           |
          [Span E]    [Span F] >>> [Span G] >>> [Span H]                    
                                        ↑
                                        ↑
                                        ↑
                        (Span G 在 Span F 后被调用, FollowsFrom)
```

- 一个Span可以和一个或者多个Span间存在因果关系。Open Tracing定义了两种关系：ChildOf和FollowsFrom。这两种引用类型代表了子节点和父节点间的直接因果关系。未来，OpenTracing将支持非因果关系的span引用关系。（例如：多个span被批量处理，span在同一个队列中，等等）

- ChildOf 很好理解，就是父亲 Span 依赖另一个孩子 Span。比如函数调用，被调者是调用者的孩子，比如说 RPC 调用，服务端那边的Span，就是 ChildOf 客户端的。很多并发的调用，然后将结果聚合起来的操作，就构成了 ChildOf 关系。

- 如果父亲 Span 并不依赖于孩子 Span 的返回结果，这时可以说它他构成 FollowsFrom 关系。

### Log的概念

- 每个Span可以进行多次Logs操作，每一个Logs操作，都需要一个带时间戳的时间名称、以及可选的任意大小的存储结果。

###  Tags的概念

- 每个Span可以有多个键值对（`key:value`）形式的Tags，Tags是没有时间戳的，支持简单的对Span进行注解和补充。