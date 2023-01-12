---
layout: post
categories: Redis
description: none
keywords: Redis
---
### List(列表)

![Redis-List](png\Redis\Redis-List.png)

list 列表，它是简单的字符串列表，链表（redis 使用双端链表实现的 List）。按照插入顺序排序，你可以添加一个元素到列表的头部（左边）或者尾部（右边），它的底层实际上是个链表。

列表有两个特点：

**一、有序**

**二、可以重复**

#### **List命令**

接着我们看一下Redis中的List，相关命令有：

| **命令**  | **描述**                                                     | **用法**                              |
| --------- | ------------------------------------------------------------ | ------------------------------------- |
| LPUSH     | （1）将一个或多个值value插入到列表key的表头（2）如果有多个value值，那么各个value值按从左到右的顺序依次插入表头（3）key不存在，一个空列表会被创建并执行LPUSH操作（4）key存在但不是列表类型，返回错误 | LPUSH key value [value ...]           |
| LPUSHX    | （1）将值value插入到列表key的表头，当且晋档key存在且为一个列表（2）key不存在时，LPUSHX命令什么都不做 | LPUSHX key value                      |
| LPOP      | （1）移除并返回列表key的头元素                               | LPOP key                              |
| LRANGE    | （1）返回列表key中指定区间内的元素，区间以偏移量start和stop指定（2）start和stop都以0位底（3）可使用负数下标，-1表示列表最后一个元素，-2表示列表倒数第二个元素，以此类推（4）start大于列表最大下标，返回空列表（5）stop大于列表最大下标，stop=列表最大下标 | LRANGE key start stop                 |
| LREM      | （1）根据count的值，移除列表中与value相等的元素（2）count>0表示从头到尾搜索，移除与value相等的元素，数量为count（3）count<0表示从从尾到头搜索，移除与value相等的元素，数量为count（4）count=0表示移除表中所有与value相等的元素 | LREM key count value                  |
| LSET      | （1）将列表key下标为index的元素值设为value（2）index参数超出范围，或对一个空列表进行LSET时，返回错误 | LSET key index value                  |
| LINDEX    | （1）返回列表key中，下标为index的元素                        | LINDEX key index                      |
| LINSERT   | （1）将值value插入列表key中，位于pivot前面或者后面（2）pivot不存在于列表key时，不执行任何操作（3）key不存在，不执行任何操作 | LINSERT key BEFORE\|AFTER pivot value |
| LLEN      | （1）返回列表key的长度（2）key不存在，返回0                  | LLEN key                              |
| LTRIM     | （1）对一个列表进行修剪，让列表只返回指定区间内的元素，不存在指定区间内的都将被移除 | LTRIM key start stop                  |
| RPOP      | （1）移除并返回列表key的尾元素                               | RPOP key                              |
| RPOPLPUSH | 在一个原子时间内，执行两个动作：（1）将列表source中最后一个元素弹出并返回给客户端（2）将source弹出的元素插入到列表desination，作为destination列表的头元素 | RPOPLPUSH source destination          |
| RPUSH     | （1）将一个或多个值value插入到列表key的表尾                  | RPUSH key value [value ...]           |
| RPUSHX    | （1）将value插入到列表key的表尾，当且仅当key存在并且是一个列表（2）key不存在，RPUSHX什么都不做 | RPUSHX key value                      |

理解Redis的List了，**操作List千万注意区分LPUSH、RPUSH两个命令，把数据添加到表头和把数据添加到表尾是完全不一样的两种结果**。

另外List还有BLPOP、BRPOP、BRPOPLPUSH三个命令没有说，它们是几个POP的阻塞版本，**即没有数据可以弹出的时候将阻塞客户端直到超时或者发现有可以弹出的元素为止**。

#### **典型使用场景**

- Stack(栈)：通过命令 lpush+lpop
- Queue（队列）：命令 lpush+rpop
- Capped Collection（有限集合）：命令 lpush+ltrim
- Message Queue（消息队列）：命令 lpush+brpop 消息队列，可以利用 List 的 *PUSH 操作，将任务存在 List 中，然后工作线程再用 POP 操作将任务取出进行执行。
-  TimeLine：使用 List 结构，我们可以轻松地实现最新消息排行等功能（比如新浪微博的 TimeLine ）。例如微博的时间轴，有人发布微博，用lpush加入时间轴，展示新的列表信息。

