---
layout: post
categories: Redis
description: none
keywords: Redis
---

### Hash(字典)

![Redis-Hash](png/Redis/Redis-Hash.png)

Hash 是一个键值对集合，是一个 string 类型的 key和 value 的映射表，key 还是key，但是value是一个键值对（key-value）。类比于 Java里面的 Map<String,Map<String,Object>> 集合。

#### Hash命令

接着我们用表格看一下Hash数据结构的相关命令：

| **命令** | **描述**                                                     | **用法**                                |
| -------- | ------------------------------------------------------------ | --------------------------------------- |
| HSET     | （1）将哈希表Key中的域field的值设为value（2）key不存在，一个新的Hash表被创建（3）field已经存在，旧的值被覆盖 | HSET key field value                    |
| HGET     | （1）返回哈希表key中给定域field的值                          | HGET key field                          |
| HDEL     | （1）删除哈希表key中的一个或多个指定域（2）不存在的域将被忽略 | HDEL key filed [field ...]              |
| HEXISTS  | （1）查看哈希表key中，给定域field是否存在，存在返回1，不存在返回0 | HEXISTS key field                       |
| HGETALL  | （1）返回哈希表key中，所有的域和值                           | HGETALL key                             |
| HINCRBY  | （1）为哈希表key中的域field加上增量increment（2）其余同INCR命令 | HINCRYBY key filed increment            |
| HKEYS    | （1）返回哈希表key中的所有域                                 | HKEYS key                               |
| HLEN     | （1）返回哈希表key中域的数量                                 | HLEN key                                |
| HMGET    | （1）返回哈希表key中，一个或多个给定域的值（2）如果给定的域不存在于哈希表，那么返回一个nil值 | HMGET key field [field ...]             |
| HMSET    | （1）同时将多个field-value对设置到哈希表key中（2）会覆盖哈希表中已存在的域（3）key不存在，那么一个空哈希表会被创建并执行HMSET操作 | HMSET key field value [field value ...] |
| HVALS    | （1）返回哈希表key中所有的域和值                             | HVALS key                               |

#### **典型使用场景**

- 缓存： 经典使用场景，把常用信息，字符串，图片或者视频等信息放到redis中，redis作为缓存层，mysql做持久化层，降低mysql的读写压力
- 计数器：redis是单线程模型，一个命令执行完才会执行下一个，同时数据可以一步落地到其他的数据源
- session：常见方案spring session + redis实现session共享