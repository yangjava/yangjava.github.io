---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL源码Buffer Pool
Buffer Pool是InnoDB中非常重要的组成部分，也是数据库用户最关心的组件之一。

## 背景
Buffer Pool的基本功能并不复杂，设计实现也比较清晰，但作为一个有几十年历史的工业级数据库产品，不可避免的在代码上融合了越来越多的功能，以及很多细节的优化，从而显得有些臃肿和晦涩。

本文希望聚焦在Buffer Pool的本职功能上，从其提供的接口、内存组织方式、Page获取、刷脏等方面进行介绍，其中会穿插一些重要的优化手段，之后用单独的一节介绍其中稍显复杂的并发控制，也就是各种mutex的设计及实现。

而除此之外，像Change Buffer、压缩Page、Double Write Buffer等功能虽然大量的穿插在Buffer Pool的实现之中，但其本身并不属于Buffer Pool的核心逻辑。

传统数据库中的数据是完整的保存在磁盘上的，但计算却只能发生在内存中，因此需要有良好的机制来协调内存及磁盘的数据交互，这就是Buffer Pool存在的意义。

也因此Buffer Pool通常按固定长度的Page来管理内存，从而方便的进行跟磁盘的数据换入换出。除此之外，磁盘和内存在访问性能上有着巨大的差距，如何最小化磁盘的IO就成了Buffer Pool的设计核心目标。

主流的数据库会采用REDO LOG加UNDO LOG，而不是限制刷脏顺序的方式，来保证数据库ACID特性。这种做法也保证了Buffer Pool可以更专注地实现高效的Cache策略。 

Buffer Pool作为一个整体，其对外部使用者提供的其实是非常简单的接口，我们称之为FIX-UNFIX接口，之所以需要FIX和UNFIX，是因为对Buffer Pool来说，上层对Page的使用时长是未知的，这个过程中需要保证Page被正确的维护在Buffer Pool中：

上层调用者先通过索引获得要访问的Page Number；

之后用这个Page Number调用Buffer Pool的FIX接口，获得Page并对其进行访问或修改，被FIX的Page不会被换出Buffer Pool；

之后调用者通过UNFIX释放Page的锁定状态。

不同事务、不同线程并发的调用Buffer Pool的FIX-UNFIX接口的序列，我们称为Page 访问序列（Page Reference String），这个序列本身是Buffer Pool无关的，只取决于数据库上面的负载类型、负载并发度、上层的索引实现以及数据模型。而通用数据库的Buffer Pool设计就是希望能在大多数的Page 访问序列下，尽可能的实现最小化磁盘IO以及高效访问的目标。 为了实现这个目标，Buffer Pool内部做了大量的工作，而替换算法是其中最至关重要的部分，由于内存容量通常是远小于磁盘容量的，替换算法需要在内存容量达到上限时，选择将现有的内存Page踢出，替换成新的被访问的Page，好的替换算法可以在给定的Buffer Size下尽量少的出现Buffer Miss。理想的情况下， 我们每次替换未来的访问序列中最远的那个Page，这也是OPT算法的思路，但显然获得未来的Page序列是不切实际的，因此OPT算法只是一个理想模型，作为评判替换算法的一个最优边界。与之相反的是作为最劣边界的Random算法，其思路是完全随机的替换。大多数的情况下， Page的访问其实是有热度区分的，这也就给替换算法一个通过历史序列判断未来序列的可能，参考的指标通常有两个：

访问距离（Age）：在Page访问序列上，某个Page上一次访问到现在的距离；
引用次数（References）：某个Page历史上或者一段时间的历史上被访问的次数。
只考虑访问距离的FIFO（First In First Out）算法和只考虑引用次数的LFU（Least Frequently Used）算法都被证明在特定序列下会有巨大的缺陷。而好的实用的替换算法会同时考虑这两个因素，其中有我们熟悉的LRU(Least Recently Used)算法以及Clocks算法。本文接下来会详细的介绍InnoDB中的LRU替换算法的实现，除此之外，还会包括如何实现高效的Page查找、内存管理、刷脏策略以及Page的并发访问。


























































