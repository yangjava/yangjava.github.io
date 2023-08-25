---
layout: post
categories: JDK
description: none
keywords: JDK
---
# JDK并发ForkJoinPool
ForkJoinPool就是JDK7提供的一种“分治算法”的多线程并行计算框架。Fork意为分叉，Join意为合并，一分一合，相互配合，形成分治算法。此外，也可以将ForkJoinPool看作一个单机版的Map/Reduce，只不过这里的并行不是多台机器并行计算，而是多个线程并行计算。

## ForkJoinPool
相比于ThreadPoolExecutor，ForkJoinPool可以更好地实现计算的负载均衡，提高资源利用率。假设有5个任务，在ThreadPoolExecutor中有5个线程并行执行，其中一个任务的计算量很大，其余4个任务的计算量很小，这会导致1个线程很忙，其他4个线程则处于空闲状态。

而利用ForkJoinPool，可以把大的任务拆分成很多小任务，然后这些小任务被所有的线程执行，从而实现任务计算的负载均衡。
