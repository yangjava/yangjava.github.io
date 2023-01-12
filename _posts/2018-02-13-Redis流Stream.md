---
layout: post
categories: Redis
description: none
keywords: Redis
---
### Redis5.0新数据结构-Stream

Redis的作者在Redis5.0中，放出一个新的数据结构，Stream。Redis Stream 的内部，其实也是一个队列，每一个不同的key，对应的是不同的队列，每个队列的元素，也就是消息，都有一个msgid，并且需要保证msgid是严格递增的。在Stream当中，消息是默认持久化的，即便是Redis重启，也能够读取到消息。那么，stream是如何做到多播的呢？其实非常的简单，与其他队列系统相似，Redis对不同的消费者，也有消费者Group这样的概念，不同的消费组，可以消费同一个消息，对于不同的消费组，都维护一个Idx下标，表示这一个消费群组消费到了哪里，每次进行消费，都会更新一下这个下标，往后面一位进行偏移。