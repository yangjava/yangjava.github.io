---
layout: post
categories: Redis
description: none
keywords: Redis
---

### linkedList(链表)

![quicklist](png/Redis/Redis-quicklist.png)



链表是一种常用的数据结构，C 语言内部是没有内置这种数据结构的实现，所以Redis自己构建了链表的实现。

链表定义：

```
typedef  struct listNode{
       //前置节点
       struct listNode *prev;
       //后置节点
       struct listNode *next;
       //节点的值
       void *value;  
}listNode
```

通过多个 listNode 结构就可以组成链表，这是一个双向链表，Redis还提供了操作链表的数据结构：

```
typedef struct list{
     //表头节点
     listNode *head;
     //表尾节点
     listNode *tail;
     //链表所包含的节点数量
     unsigned long len;
     //节点值复制函数
     void (*free) (void *ptr);
     //节点值释放函数
     void (*free) (void *ptr);
     //节点值对比函数
     int (*match) (void *ptr,void *key);
}list;
```

![Redis-链表](png\Redis\Redis-链表.png)

Redis链表特性：

①、双端：链表具有前置节点和后置节点的引用，获取这两个节点时间复杂度都为O(1)。

②、无环：表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL,对链表的访问都是以 NULL 结束。　　

　　③、带链表长度计数器：通过 len 属性获取链表长度的时间复杂度为 O(1)。

④、多态：链表节点使用 void* 指针来保存节点值，可以保存各种不同类型的值。

后续版本对列表数据结构进行了改造，使用 quicklist 代替了 ziplist 和 linkedlist。quicklist 是 ziplist 和 linkedlist 的混合体，它将 linkedlist 按段切分，每一段使用 ziplist 来紧凑存储，多个 ziplist 之间使用双向指针串接起来。这也是为何 Redis 快的原因，不放过任何一个可以提升性能的细节。

