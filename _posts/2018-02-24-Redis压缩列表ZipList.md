---
layout: post
categories: Redis
description: none
keywords: Redis
---
### zipList(压缩列表)

![ZipList压缩列表](png/Redis/Redis-ZipList压缩列表.png)

压缩列表是 List 、hash、 sorted Set 三种数据类型底层实现之一。当一个列表只有少量数据的时候，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么 Redis 就会使用压缩列表来做列表键的底层实现。

ziplist 是由一系列特殊编码的连续内存块组成的顺序型的数据结构，ziplist 中可以包含多个 entry 节点，每个节点可以存放整数或者字符串。ziplist 在表头有三个字段 zlbytes、zltail 和 zllen，分别表示列表占用字节数、列表尾的偏移量和列表中的 entry 个数；压缩列表在表尾还有一个 zlend，表示列表结束。

压缩列表（ziplist）是Redis为了节省内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型数据结构，一个压缩列表可以包含任意多个节点（entry），每个节点可以保存一个字节数组或者一个整数值。

**压缩列表的原理：压缩列表并不是对数据利用某种算法进行压缩，而是将数据按照一定规则编码在一块连续的内存区域，目的是节省内存。**

```go
struct ziplist<T> {
    int32 zlbytes;           // 整个压缩列表占用字节数
    int32 zltail_offset;  // 最后一个元素距离压缩列表起始位置的偏移量，用于快速定位到最后一个节点
    int16 zllength;          // 元素个数
    T[] entries;              // 元素内容列表，挨个挨个紧凑存储
    int8 zlend;               // 标志压缩列表的结束，值恒为 0xFF
}
```

如果我们要查找定位第一个元素和最后一个元素，可以通过表头三个字段的长度直接定位，复杂度是 O(1)。而查找其他元素时，就没有这么高效了，只能逐个查找，此时的复杂度就是 O(N)。

​		①、previous_entry_ength：记录压缩列表前一个字节的长度。previous_entry_ength的长度可能是1个字节或者是5个字节，如果上一个节点的长度小于254，则该节点只需要一个字节就可以表示前一个节点的长度了，如果前一个节点的长度大于等于254，则previous length的第一个字节为254，后面用四个字节表示当前节点前一个节点的长度。利用此原理即当前节点位置减去上一个节点的长度即得到上一个节点的起始位置，压缩列表可以从尾部向头部遍历。这么做很有效地减少了内存的浪费。

②、encoding：节点的encoding保存的是节点的content的内容类型以及长度，encoding类型一共有两种，一种字节数组一种是整数，encoding区域长度为1字节、2字节或者5字节长。

③、content：content区域用于保存节点的内容，节点内容类型和长度由encoding决定。

### 快速列表(quicklist)

一个由ziplist组成的双向链表。但是一个quicklist可以有多个quicklist节点，它很像B树的存储方式。是在redis3.2版本中新加的数据结构，用在列表的底层实现。

```
typedef struct quicklist {
    //指向头部(最左边)quicklist节点的指针
    quicklistNode *head;
 
    //指向尾部(最右边)quicklist节点的指针
    quicklistNode *tail;
 
    //ziplist中的entry节点计数器
    unsigned long count;        /* total count of all entries in all ziplists */
 
    //quicklist的quicklistNode节点计数器
    unsigned int len;           /* number of quicklistNodes */
 
    //保存ziplist的大小，配置文件设定，占16bits
    int fill : 16;              /* fill factor for individual nodes */
 
    //保存压缩程度值，配置文件设定，占16bits，0表示不压缩
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off */
} quicklist;
```

quicklist节点结构:

```
typedef struct quicklistNode {
    struct quicklistNode *prev;     //前驱节点指针
    struct quicklistNode *next;     //后继节点指针
 
    //不设置压缩数据参数recompress时指向一个ziplist结构
    //设置压缩数据参数recompress指向quicklistLZF结构
    unsigned char *zl;
 
    //压缩列表ziplist的总长度
    unsigned int sz;                  /* ziplist size in bytes */
 
    //ziplist中包的节点数，占16 bits长度
    unsigned int count : 16;          /* count of items in ziplist */
 
    //表示是否采用了LZF压缩算法压缩quicklist节点，1表示压缩过，2表示没压缩，占2 bits长度
    unsigned int encoding : 2;        /* RAW==1 or LZF==2 */
 
    //表示一个quicklistNode节点是否采用ziplist结构保存数据，2表示压缩了，1表示没压缩，默认是2，占2bits长度
    unsigned int container : 2;       /* NONE==1 or ZIPLIST==2 */
 
    //标记quicklist节点的ziplist之前是否被解压缩过，占1bit长度
    //如果recompress为1，则等待被再次压缩
    unsigned int recompress : 1; /* was this node previous compressed? */
 
    //测试时使用
    unsigned int attempted_compress : 1; /* node can't compress; too small */
 
    //额外扩展位，占10bits长度
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```

**相关配置**

在redis.conf中的ADVANCED CONFIG部分：

```
list-max-ziplist-size -2
list-compress-depth 0
```

**list-max-ziplist-size参数**

我们来详细解释一下`list-max-ziplist-size`这个参数的含义。它可以取正值，也可以取负值。

当取正值的时候，表示按照数据项个数来限定每个quicklist节点上的ziplist长度。比如，当这个参数配置成5的时候，表示每个quicklist节点的ziplist最多包含5个数据项。

当取负值的时候，表示按照占用字节数来限定每个quicklist节点上的ziplist长度。这时，它只能取-1到-5这五个值，每个值含义如下：

-5: 每个quicklist节点上的ziplist大小不能超过64 Kb。（注：1kb => 1024 bytes）

-4: 每个quicklist节点上的ziplist大小不能超过32 Kb。

-3: 每个quicklist节点上的ziplist大小不能超过16 Kb。

-2: 每个quicklist节点上的ziplist大小不能超过8 Kb。（-2是Redis给出的默认值）

**list-compress-depth参数**

这个参数表示一个quicklist两端不被压缩的节点个数。注：这里的节点个数是指quicklist双向链表的节点个数，而不是指ziplist里面的数据项个数。实际上，一个quicklist节点上的ziplist，如果被压缩，就是整体被压缩的。

参数list-compress-depth的取值含义如下：

0: 是个特殊值，表示都不压缩。这是Redis的默认值。 1: 表示quicklist两端各有1个节点不压缩，中间的节点压缩。 2: 表示quicklist两端各有2个节点不压缩，中间的节点压缩。 3: 表示quicklist两端各有3个节点不压缩，中间的节点压缩。 依此类推…

Redis对于quicklist内部节点的压缩算法，采用的LZF——一种无损压缩算法。

### 总结

大多数情况下，Redis使用简单字符串SDS作为字符串的表示，相对于C语言字符串，SDS具有常数复杂度获取字符串长度，杜绝了缓存区的溢出，减少了修改字符串长度时所需的内存重分配次数，以及二进制安全能存储各种类型的文件，并且还兼容部分C函数。

通过为链表设置不同类型的特定函数，Redis链表可以保存各种不同类型的值，除了用作列表键，还在发布与订阅、慢查询、监视器等方面发挥作用（后面会介绍）。

Redis的字典底层使用哈希表实现，每个字典通常有两个哈希表，一个平时使用，另一个用于rehash时使用，使用链地址法解决哈希冲突。

跳跃表通常是有序集合的底层实现之一，表中的节点按照分值大小进行排序。

整数集合是集合键的底层实现之一，底层由数组构成，升级特性能尽可能的节省内存。

压缩列表是Redis为节省内存而开发的顺序型数据结构，通常作为列表键和哈希键的底层实现之一。

以上介绍的简单字符串、链表、字典、跳跃表、整数集合、压缩列表等数据结构就是Redis底层的一些数据结构，用来实现上一篇博客介绍的Redis五大数据类型，那么每种数据类型是由哪些数据结构实现的呢？