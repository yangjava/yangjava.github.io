---
layout: post
categories: [Redis]
description: none
keywords: Redis
---
# Redis客户端Jedis

## Jedis对应Redis的四种工作模式
对应关系如下：

Jedis主要模块	Redis工作模式
Jedis	Redis Standalone（单节点模式）
JedisCluster	Redis Cluster（集群模式）
JedisSentinel	Redis Sentinel（哨兵模式）
ShardedJedis	Redis Sharding（分片模式）

## Jedis三种请求模式
Jedis实例有3种请求模式：Client、Pipeline和Transaction

Jedis实例通过Socket建立客户端与服务端的长连接，往outputStream发送命令，从inputStream读取回复

### Client模式
客户端发一个命令，阻塞等待服务端执行，然后读取返回结果

### Pipeline模式
一次性发送多个命令，最后一次取回所有的返回结果

这种模式通过减少网络的往返时间和IO的读写次数，大幅度提高通信性能

原生批命令（mset、mget）与Pipeline对比

原生批量命令是原子性，Pipeline是非原子性的
原生批量命令是一个命令对应多个key，Pipeline支持多个命令
原生批量命令是Redis服务端支持实现的，而Pipeline需要服务端与客户端的共同实现

### Transaction模式
Transaction模式即开启Redis的事务管理

Redis事务可以一次执行多个命令， 并且带有以下三个重要的保证：

批量操作在发送EXEC命令前被放入队列缓存
收到EXEC命令后进入事务执行，事务中任意命令执行失败，其余的命令依然被执行
在事务执行过程，其他客户端提交的命令请求不会插入到事务执行命令序列中
一个事务从开始到执行会经历以下三个阶段：

开始事务
命令入队
执行事务

### Redis事务相关命令：

multi：标记一个事务块的开始
exec：执行所有事务块的命令（一旦执行exec后，之前加的监控锁都会被取消掉）
discard：取消事务，放弃事务块中的所有命令
watch key1 key2 ...：监视一或多个key，如果在事务执行之前，被监视的key被其他命令改动，则事务被打断（类似乐观锁）
unwatch：取消watch对所有key的监控






































