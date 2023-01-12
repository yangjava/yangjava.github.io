---
layout: post
categories: Redis
description: none
keywords: Redis
---

### String(字符串)

string 是Redis的最基本的数据类型，是简单的 key-value 类型，一个key 对应一个 value。value 不仅可以是 String，也可以是数字（当数字类型用 Long 可以表示的时候encoding 就是整型，其他都存储在 sdshdr 当做字符串）。string 类型是二进制安全的，意思是 Redis 的 string 可以包含任何数据，比如图片或者序列化的对象，一个 redis 中字符串 value 最多可以是 512M。

#### String命令

下面用表格来看一下String操作的相关命令：

在这之外，专门提两点：

- Redis的命令不区分大小写
- Redis的Key区分大小写

| **命令** | **描述**                                                     | **用法**                                              |
| -------- | ------------------------------------------------------------ | ----------------------------------------------------- |
| SET      | （1）将字符串值Value关联到Key（2）Key已关联则覆盖，无视类型（3）原本Key带有生存时间TTL，那么TTL被清除 | SET key value [EX seconds] [PX milliseconds] [NX\|XX] |
| GET      | （1）返回key关联的字符串值（2）Key不存在返回nil（3）Key存储的不是字符串，返回错误，因为GET只用于处理字符串 | GET key                                               |
| MSET     | （1）同时设置一个或多个Key-Value键值对（2）某个给定Key已经存在，那么MSET新值会覆盖旧值（3）如果上面的覆盖不是希望的，那么使用MSETNX命令，所有Key都不存在才会进行覆盖（4）**MSET是一个原子性操作**，所有Key都会在同一时间被设置，不会存在有些更新有些没更新的情况 | MSET key value [key value ...]                        |
| MGET     | （1）返回一个或多个给定Key对应的Value（2）某个Key不存在那么这个Key返回nil | MGET key [key ...]                                    |
| SETEX    | （1）将Value关联到Key（2）设置Key生存时间为seconds，单位为秒（3）如果Key对应的Value已经存在，则覆盖旧值（4）SET也可以设置失效时间，但是不同在于SETNX是一个原子操作，即关联值与设置生存时间同一时间完成 | SETEX key seconds value                               |
| SETNX    | （1）将Key的值设置为Value，当且仅当Key不存在（2）若给定的Key已经存在，SEXNX不做任何动作 | SETNX key value                                       |

①、上面的 ttl 命令是返回 key 的剩余过期时间，单位为秒。

②、mset和mget这种批量处理命令，能够极大的提高操作效率。因为一次命令执行所需要的时间=1次网络传输时间+1次命令执行时间，n个命令耗时=n次网络传输时间+n次命令执行时间，而批量处理命令会将n次网络时间缩减为1次网络时间，也就是1次网络传输时间+n次命令处理时间。

但是需要注意的是，Redis是单线程的，如果一次批量处理命令过多，会造成Redis阻塞或网络拥塞（传输数据量大）。

③、setnx可以用于实现分布式锁，具体实现方式后面会介绍。

#### **INCR/DECR**命令

前面介绍的是基本的Key-Value操作，下面介绍一种特殊的Key-Value操作即INCR/DECR，可以利用Redis自动帮助我们对一个Key对应的Value进行加减，用表格看一下相关命令：

| **命令** | **描述**                                                     | **用法**             |
| -------- | ------------------------------------------------------------ | -------------------- |
| INCR     | （1）Key中存储的数字值+1，返回增加之后的值（2）Key不存在，那么Key的值被初始化为0再执行INCR（3）如果值包含错误类型或者字符串不能被表示为数字，那么返回错误（4）值限制在64位有符号数字表示之内，即-9223372036854775808~9223372036854775807 | INCR key             |
| DECR     | （1）Key中存储的数字值-1（2）其余同INCR                      | DECR key             |
| INCRBY   | （1）将key所存储的值加上增量返回增加之后的值（2）其余同INCR  | INCRBY key increment |
| DECRBY   | （1）将key所存储的值减去减量decrement（2）其余同INCR         | DECRBY key decrement |

INCR/DECR在实际工作中还是非常管用的，

#### **典型使用场景**

- 计数：由于Redis单线程的特点，我们不用考虑并发造成计数不准的问题，通过 incrby 命令，我们可以正确的得到我们想要的结果。
- 限制次数：比如登录次数校验，错误超过三次5分钟内就不让登录了，每次登录设置key自增一次，并设置该key的过期时间为5分钟后，每次登录检查一下该key的值来进行限制登录。
- 秒杀库存：由于Redis本身极高的读写性能，一些秒杀的场景库存增减可以基于Redis来做而不是直接操作DB