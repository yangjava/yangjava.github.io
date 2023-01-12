---
layout: post
categories: Redis
description: none
keywords: Redis
---

### Sorted Set(有序集合)

![Redis-SortedSet](png/Redis/Redis-SortedSet.png)

有序集合（sorted set ），和上面的set 数据类型一样，也是 string 类型元素的集合，但是它是有序的。和Sets相比，Sorted Sets是将 Set 中的元素增加了一个权重参数 score，使得集合中的元素能够按 score 进行有序排列，比如一个存储全班同学成绩的 Sorted Sets，其集合 value 可以是同学的学号，而 score 就可以是其考试得分，这样在数据插入集合的时候，就已经进行了天然的排序。另外还可以用 Sorted Sets 来做带权重的队列，比如普通消息的 score 为1，重要消息的 score 为2，然后工作线程可以选择按 score 的倒序来获取工作任务。让重要的任务优先执行。

#### SortedSet命令

SortedSet顾名思义，即有序的Set，看下相关命令：

| **命令**         | **描述**                                                     | **用法**                                                  |
| ---------------- | ------------------------------------------------------------ | --------------------------------------------------------- |
| ZADD             | （1）将一个或多个member元素及其score值加入有序集key中（2）如果member已经是有序集的成员，那么更新member对应的score并重新插入member保证member在正确的位置上（3）score可以是整数值或双精度浮点数 | ZADD key score member [[score member] [score member] ...] |
| ZCARD            | （1）返回有序集key的元素个数                                 | ZCARD key                                                 |
| ZCOUNT           | （1） 返回有序集key中，score值>=min且<=max的成员的数量       | ZCOUNT key min max                                        |
| ZRANGE           | （1）返回有序集key中指定区间内的成员，成员位置按score从小到大排序（2）具有相同score值的成员按字典序排列（3）需要成员按score从大到小排列，使用ZREVRANGE命令（4）下标参数start和stop都以0为底，也可以用负数，-1表示最后一个成员，-2表示倒数第二个成员（5）可通过WITHSCORES选项让成员和它的score值一并返回 | ZRANGE key start stop [WITHSCORES]                        |
| ZRANK            | （1）返回有序集key中成员member的排名，有序集成员按score值从小到大排列（2）排名以0为底，即score最小的成员排名为0（3）ZREVRANK命令可将成员按score值从大到小排名 | ZRANK key number                                          |
| ZREM             | （1）移除有序集key中的一个或多个成员，不存在的成员将被忽略（2）当key存在但不是有序集时，返回错误 | ZREM key member [member ...]                              |
| ZREMRANGEBYRANK  | （1）移除有序集key中指定排名区间内的所有成员                 | ZREMRANGEBYRANK key start stop                            |
| ZREMRANGEBYSCORE | （1）移除有序集key中，所有score值>=min且<=max之间的成员      | ZREMRANGEBYSCORE key min max                              |

#### **典型使用场景**

- 带有权重的元素，比如一个游戏的用户得分排行榜。和set数据结构一样，zset也可以用于社交领域的相关业务，并且还可以利用zset 的有序特性，还可以做类似排行榜的业务。
- 比较复杂的数据结构，一般用到的场景不算太多
- 排行榜：有序集合经典使用场景。例如小说视频等网站需要对用户上传的小说视频做排行榜，榜单可以按照用户关注数，更新时间，字数等打分，做排行。

