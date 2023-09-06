---
layout: post
categories: [Java]
description: none
keywords: Java
---
# Java集合HashMap
HashMap是Java中常用的一种数据结构，它基于哈希表实现，可以高效地存储和查找键值对。

## HashMap 的类结构
Java为数据结构中的映射定义了一个接口java.util.Map，此接口主要有四个常用的实现类，分别是HashMap、Hashtable、LinkedHashMap和TreeMap

HashMap是一个基于哈希表的Map接口实现类，它继承自AbstractMap类，实现了Map、Cloneable和Serializable接口。在Java中，哈希表是一种常用的数据结构，它可以快速地存储和查找数据。HashMap内部使用了一个数组来存储数据，每个数组元素都是一个链表，链表中存储了哈希值相同的键值对。

## HashMap存储结构
下文我们主要结合源码，从存储结构、常用方法分析、扩容以及安全性等方面深入讲解HashMap的工作原理。

HashMap的底层主要基于 数组+链表/红黑树 实现，数组优点就是 查询块 ，HashMap通过计算hash码获取到数组的下标来查询数据。同样也可以通过hash码得到数组下标，存放数据。

哈希表为了解决冲突，HashMap采用了 链表法 ,添加的数据存放在链表中，如果发送冲突，将数据放入链表尾部。

## 主要属性
```
// 储存元素数组  
Node<K,V>[] table;  
  
// 元素个数  
int size;  
  
// 数组扩容临界值，计算为：元素容量*装载因子  
int threshold  
  
// 装载因子，默认0.75  
float loadFactor;  
  
// 链表长度为 8 的时候会转为红黑树  
int TREEIFY_THRESHOLD = 8;  
  
// 长度为 6 的时候会从红黑树转为链表  
int UNTREEIFY_THRESHOLD = 6;  
```
- size记录元素个数
- threshold 扩容的临界值，等于元素容量*装载因子
- TREEIFY_THRESHOLD 8 链表个数增加到8会转成红黑树
- UNTREEIFY_THRESHOLD 6 链表个数减少到6会退化成链表
- loadFactor 装载因子，默认为0.75
loadFactor 装载因子等于扩容阈值/数组长度，表示元素被填满的程序，越高表示空间 利用率越高 ，但是hash冲突的概率增加，链表越长，查询的效率降低。越低hash冲突减少了，数据查询效率更高。但是 空间利用率越低 ，很多空间没用又继续扩容。为了均衡 查询时间 和 使用空间 ，系统默认装载因子为0.75。

### HashMap的重要属性：table 桶数组
从HashMap的源码中，我们可以发现，HashMap有一个非常重要的属性 —— table，

这是由一个Node类型的元素构成的数组：
```
transient Node<K,V>[] table;  
```
table 也叫 哈希数组 ， 哈希槽位 数组 ， table 桶数组 ， 散列表 ， 数组中的一个 元素，常常被称之为 一个 槽位 slot

Node类作为HashMap中的一个内部类，每个 Node 包含了一个 key-value 键值对。
```
static class Node<K,V> implements Map.Entry<K,V> {  
        final int hash;  
        final K key;  
        V value;  
        Node<K,V> next;  
  
        Node(int hash, K key, V value, Node<K,V> next) {  
        this.hash = hash;  
        this.key = key;  
        this.value = value;  
        this.next = next;  
    }  
  
    public final int hashCode() {  
        return Objects.hashCode(key) ^ Objects.hashCode(value);  
    }  
    ..........  
}  
```
Node 类作为 HashMap 中的一个内部类，除了 key、value 两个属性外，还定义了一个next 指针。next 指针的作用：链地址法解决哈希冲突。

当有哈希冲突时，HashMap 会用之前数组当中相同哈希值对应存储的 Node 对象，通过指针指向新增的相同哈希值的 Node 对象的引用。

从结构实现来讲，HashMap是数组+链表+红黑树（JDK1.8增加了红黑树部分）实现的

Java中HashMap采用了链地址法。链地址法，简单来说，就是 数组加链表 的结合。 在每个数组元素上都一个链表结构， 当数据被Hash后，首先得到 数组下标 ，然后 ， 把数据放在 对应下标元素的链表 上。

例如程序执行下面代码：
```
map.put("keyA","value1");  
map.put("keyB","value2");
```
对于 第一句， 系统将调用"keyA"的hashCode()方法得到其hashCode ，然后再通过Hash算法来定位该键值对的存储位置，然后将 构造 entry 后加入到 存储位置 指向 的 链表中

对于 第一句， 系统将调用"keyB"的hashCode()方法得到其hashCode ，然后再通过Hash算法来定位该键值对的存储位置，然后将 构造 entry 后加入到 存储位置 指向 的链表中

有时两个key会定位到相同的位置，表示发生了Hash碰撞。Hash算法计算结果越分散均匀，Hash碰撞的概率就越小，map的存取效率就会越高。

### HashMap的重要属性：加载因子（loadFactor）和边界值（threshold）
HashMap还有两个重要的属性：
- 加载因子（loadFactor）
- 边界值（threshold）。

在初始化 HashMap时，就会涉及到这两个关键初始化参数。
- Node[] table的初始化长度length(默认值是16)，
- loadFactor 为负载因子(默认值是0.75)，
- threshold是HashMap所能容纳的最大数据量的Node 个数。

threshold 、length 、loadFactor 三者之间的关系：
```
threshold = length * Load factor。

默认情况下 threshold = 16 * 0.75 =12。
```
threshold就是允许的哈希数组 最大元素数目，超过这个数目就重新resize(扩容)，扩容后的哈希数组 容量length 是之前容量length 的两倍。

threshold是通过初始容量和LoadFactor计算所得，在初始HashMap不设置参数的情况下，默认边界值为12。

如果HashMap中Node的数量超过边界值，HashMap就会调用resize()方法重新分配table数组。

这将会导致HashMap的数组复制，迁移到另一块内存中去，从而影响HashMap的效率。

### HashMap的重要属性：loadFactor 属性
为什么loadFactor 默认是 0.75 这个值呢？

loadFactor 也是可以调整的，默认是0.75，但是，如果loadFactor 负载因子越大，在数组定义好 length 长度之后，所能容纳的键值对个数越多。

LoadFactor属性是用来间接设置Entry数组（哈希表）的内存空间大小，在初始HashMap不设置参数的情况下，默认LoadFactor值为0.75。

为什么loadFactor 默认是0.75这个值呢？

这是由于 加载因子的两面性导致的 加载因子越大，对空间的利用就越充分，碰撞的机会越高，这就意味着链表的长度越长，查找效率也就越低。

因为对于使用链表法的哈希表来说，查找一个元素的平均时间是O(1+n)，这里的n指的是遍 历链表的长度 ， 如果设置的加载因子太小，那么哈希表的数据将过于稀疏，对空间造成严重浪费。

当然，加载因子小，碰撞的机会越低， 查找的效率就搞，性能就越好。 默认的负载因子0.75是对空间和时间效率的一个平衡选择，建议大家不要修改，除非在时间和空间比较特殊的情况下。

分为两种情况：
- 如果内存空间很多而又对时间效率要求很高，可以降低负载因子Load factor的值；
- 相反，如果内存空间紧张而对时间效率要求不高，可以增加负载因子loadFactor的值，这个值可以大于1。

有什么办法可以来解决因链表过长而导致的查询时间复杂度高的问题呢？

在JDK1.8后就使用了将链表转换为红黑树来解决这个问题。

Entry 数组（哈希槽位数组）的 threshold 阈值 是通过初始容量和 loadFactor计算所得，

在初始 HashMap 不设置参数的情况下，默认边界值为12（16*0.75）。

如果我们在初始化时，设置的初始化容量较小，HashMap 中 Node 的数量超过边界值，HashMap 就会调用 resize() 方法重新分配 table 数组。

这将导致 HashMap 的数组复制，迁移到另一块内存中去，从而影响 HashMap 的效率。
```
public HashMap() {//默认初始容量为16，加载因子为0.75  
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted  
    }  
    public HashMap(int initialCapacity) {//指定初始容量为initialCapacity  
        this(initialCapacity, DEFAULT_LOAD_FACTOR);  
    }  
    static final int MAXIMUM_CAPACITY = 1 << 30;//最大容量  
  
    //当size到达threshold这个阈值时会扩容，下一次扩容的值，根据capacity * load factor进行计算，  
    int threshold;  
    /**由于HashMap的capacity都是2的幂，因此这个方法用于找到大于等于initialCapacity的最小的2的幂（initialCapacity如果就是2的幂，则返回的还是这个数）  
     * 通过5次无符号移位运算以及或运算得到：  
     *    n第一次右移一位时，相当于将最高位的1右移一位，再和原来的n取或，就将最高位和次高位都变成1，也就是两个1；  
     *    第二次右移两位时，将最高的两个1向右移了两位，取或后得到四个1；  
     *    依次类推，右移16位再取或就能得到32个1；  
     *    最后通过加一进位得到2^n。  
     * 比如initialCapacity = 10 ，那就返回16， initialCapacity = 17，那么就返回32  
     *    10的二进制是1010，减1就是1001  
     *    第一次右移取或：1001 | 0100 = 1101 ；  
     *    第二次右移取或：1101 | 0011 = 1111 ；  
     *    第三次右移取或：1111 | 0000 = 1111 ；  
     *    第四次第五次同理  
     *    最后得到 n = 1111，返回值是 n+1 = 2 ^ 4 = 16 ;  
     * 让cap-1再赋值给n的目的是另找到的目标值大于或等于原值。这是为了防止，cap已经是2的幂。如果cap已经是2的幂，又没有执行这个减1操作，则执行完后面的几条无符号右移操作之后，返回的capacity将是这个cap的2倍。  
     * 例如十进制数值8，二进制为1000，如果不对它减1而直接操作，将得到答案10000，即16。显然不是结果。减1后二进制为111，再进行操作则会得到原来的数值1000，即8。  
     * 问题：tableSizeFor()最后赋值给threshold，但threshold是根据capacity * load factor进行计算的，这是不是有问题？  
     * 注意：在构造方法中，并没有对table这个成员变量进行初始化，table的初始化被推迟到了put方法中，在put方法中会对threshold重新计算。  
     * 问题：既然put会重新计算threshold，那么在构造初始化threshold的作用是什么？  
     * 答：在put时，会对table进行初始化，如果threshold大于0，会把threshold当作数组的长度进行table的初始化，否则创建的table的长度为16。  
     */  
    static final int tableSizeFor(int cap) {  
        int n = cap - 1;  
        n |= n >>> 1;  
        n |= n >>> 2;  
        n |= n >>> 4;  
        n |= n >>> 8;  
        n |= n >>> 16;  
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;  
    }  
    public HashMap(int initialCapacity, float loadFactor) {//指定初始容量和加载因子  
        if (initialCapacity < 0)  
            throw new IllegalArgumentException("Illegal initial capacity: " +  
                                               initialCapacity);  
        if (initialCapacity > MAXIMUM_CAPACITY)//大于最大容量，设置为最大容量  
            initialCapacity = MAXIMUM_CAPACITY;  
        if (loadFactor <= 0 || Float.isNaN(loadFactor))//加载因子小于等于0或为NaN抛出异常  
            throw new IllegalArgumentException("Illegal load factor: " +  
                                               loadFactor);  
        this.loadFactor = loadFactor;  
        this.threshold = tableSizeFor(initialCapacity);//边界值  
    } 
```

### HashMap的重要属性：size属性
size这个字段其实很好理解，就是HashMap中实际存在的键值对数量。 注意: size和table的长度length的区别 ，length是 哈希桶数组table的长度

在HashMap中，哈希桶数组table的长度length大小必须为2的n次方，这是一定是一个合数，这是一种反常规的设计.

常规的设计是把桶数组的大小设计为素数。相对来说素数导致冲突的概率要小于合数， 比如，Hashtable初始化桶大小为11，就是桶大小设计为素数的应用（Hashtable扩容后不能保证还是素数）。

HashMap采用这种非常规设计，主要是为了方便扩容。 而 HashMap为了减少冲突，采用另外的方法规避：计算哈希桶索引位置时，哈希值的高位参与运算。

### HashMap的重要属性：modCount属性
我们能够发现，在集合类的源码里，像HashMap、TreeMap、ArrayList、LinkedList等都有modCount属性，字面意思就是修改次数，

首先看一下源码里对此属性的注释

HashMap部分源码：
```
    /**  
     * The number of times this HashMap has been structurally modified  
     * Structural modifications are those that change the number of mappings in  
     * the HashMap or otherwise modify its internal structure (e.g.,  
     * rehash).  This field is used to make iterators on Collection-views of  
     * the HashMap fail-fast.  (See ConcurrentModificationException).  
     */  
    transient int modCount;  
```
此哈希表已被 结构性修改 的次数， 结构性修改 是指哈希表的内部结构被修改，比如桶数组被修改或者拉链被修改。 那些更改桶数组或者拉链的操作如，重新哈希。此字段用于HashMap集合迭代器的快速失败。

所以，modCount主要是为了防止在迭代过程中某些原因改变了原集合，导致出现不可预料的情况，从而抛出并发修改异常，

这可能也与Fail-Fast机制有关: 在可能出现错误的情况下提前抛出异常终止操作。

HashMap的remove方法源码(部分截取)：
```
if (node != null && (!matchValue || (v = node.value) == value ||  
                                 (value != null && value.equals(v)))) {  
                if (node instanceof TreeNode)  
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);  
                else if (node == p)  
                    tab[index] = node.next;  
                else  
                    p.next = node.next;  
                ++modCount;  //进行了modCount自增操作  
                --size;  
                afterNodeRemoval(node);  
                return node;  
```

## HashMap的构造方法
HashMap提供了多个构造方法，其中最常用的是无参构造方法和带初始容量和负载因子的构造方法。无参构造方法会创建一个默认容量为16、负载因子为0.75的HashMap对象，而带初始容量和负载因子的构造方法可以指定HashMap的容量和负载因子。

```
// 无参构造方法  
public HashMap() {  
    this.loadFactor = DEFAULT_LOAD_FACTOR; // 负载因子默认为0.75  
}  
// 带初始容量和负载因子的构造方法  
public HashMap(int initialCapacity, float loadFactor) {  
    if (initialCapacity < 0)  
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);  
    if (initialCapacity > MAXIMUM_CAPACITY)  
        initialCapacity = MAXIMUM_CAPACITY;  
    if (loadFactor <= 0 || Float.isNaN(loadFactor))  
        throw new IllegalArgumentException("Illegal load factor: " + loadFactor);  
    this.loadFactor = loadFactor;  
    this.threshold = tableSizeFor(initialCapacity);  
}  
```

### put方法源码：
当将一个 key-value 对添加到 HashMap 中，

首先会根据该 key 的 hashCode() 返回值，再通过 hash() 方法计算出 hash 值，

再 除留余数法 ，取得余数，这里通过位运算来完成。putVal 方法中的 (n-1) & hash 就是 hash值除以n留余数， n 代表哈希表的长度。余数 (n-1) & hash 决定该 Node 的存储位置，哈希表习惯将长度设置为2的 n 次方，这样可以恰好保证 (n-1)&hash 计算得出的索引值总是位于 table 数组的索引之内。
```
public V put(K key, V value) {
return putVal(hash(key), key, value, false, true);
}
```

hash计算：
key的hash值 高16位不变 ， 低16位与高16位异或 ，作为key的最终hash值。
```
static final int hash(Object key) {  
        int h;  
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);  
    }  
  
// 要点1：h >>>  16，表示无符号右移16位，高位补0，任何数跟0异或都是其本身，因此key的hash值高16位不变。  
  
//  要点2：异或的运算法则为：0⊕0=0，1⊕0=1，0⊕1=1，1⊕1=0（同为0，异为1）
```
即取 int 类型的一半，刚好可以将该二进制数对半切开，

利用异或运算（如果两个数对应的位置相反，则结果为1，反之为0），这样可以避免哈希冲突。

底16位与高16位异或，其目标：

尽量打乱 hashCode 真正参与运算的低16位，减少hash 碰撞 。

之所以要无符号右移16位，是跟table的下标有关，位置计算方式是：

（n-1)&hash 计算 Node 的存储位置

假如n=16， 从下图可以看出：

table的下标仅与hash值的低n位有关，hash值的高位都被与操作置为0了，只有hash值的低4位参与了运算。

### putVal方法源码
putVal：

而当链表长度太长（默认超过 8）时，链表就进行转换红黑树的操作。

这里利用 红黑树快速增删改查 的特点，提高 HashMap 的性能。

当红黑树结点个数少于 6 个的时候，又会将红黑树转化为链表。

因为在数据量较小的情况下，红黑树要维护平衡，比起链表来，性能上的优势并不明显。
```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,  
                   boolean evict) {  
        Node<K,V>[] tab; Node<K,V> p; int n, i;  
        //此时 table 尚未初始化，通过 resize 方法得到初始化的table  
        if ((tab = table) == null || (n = tab.length) == 0)  
            n = (tab = resize()).length;  
        // （n-1)&hash 计算 Node 的存储位置，如果判断 Node 不在哈希表中(链表的第一个节点位置），新增一个 Node，并加入到哈希表中  
        if ((p = tab[i = (n - 1) & hash]) == null)  
            tab[i] = newNode(hash, key, value, null);  
        else {//hash冲突了  
            Node<K,V> e; K k;  
            if (p.hash == hash &&  
                ((k = p.key) == key || (key != null && key.equals(k))))  
                e = p;//判断key的条件是key的hash相同和eqauls方法符合，p.key等于插入的key，将p的引用赋给e  
            else if (p instanceof TreeNode)// p是红黑树节点，插入后仍然是红黑树节点，所以直接强制转型p后调用putTreeVal，返回的引用赋给e  
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);  
            else {//链表  
                // 循环，直到链表中的某个节点为null，或者某个节点hash值和给定的hash值一致且key也相同，则停止循环。  
                for (int binCount = 0; ; ++binCount) {//binCount是一个计数器，来计算当前链表的元素个数  
                    if ((e = p.next) == null) {//next为空，将添加的元素置为next  
                        p.next = newNode(hash, key, value, null);  
                        //插入成功后，要判断是否需要转换为红黑树，因为插入后链表长度+1，而binCount并不包含新节点，所以判断时要将临界阀值-1.【链表长度达到了阀值TREEIFY_THRESHOLD=8，即链表长度达到了7】  
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st  
                            // 如果链表长度达到了8，且数组长度小于64，那么就重新散列resize()，如果大于64，则创建红黑树，将链表转换为红黑树  
                            treeifyBin(tab, hash);  
                        //结束循环  
                        break;  
                    }  
                    //节点hash值和给定的hash值一致且key也相同，停止循环  
                    if (e.hash == hash &&  
                        ((k = e.key) == key || (key != null && key.equals(k))))  
                        break;  
                    //如果给定的hash值不同或者key不同。将next值赋给p，为下次循环做铺垫。即结束当前节点，对下一节点进行判断  
                    p = e;  
                }  
            }  
            //如果e不是null，该元素存在了(也就是key相等)  
            if (e != null) { // existing mapping for key  
                // 取出该元素的值  
                V oldValue = e.value;  
                // 如果 onlyIfAbsent 是 true，就不用改变已有的值；如果是false(默认)，或者value是null，将新的值替换老的值  
                if (!onlyIfAbsent || oldValue == null)  
                    e.value = value;  
                //什么都不做  
                afterNodeAccess(e);  
                //返回旧值  
                return oldValue;  
            }  
        }  
        //修改计数器+1，为迭代服务  
        ++modCount;  
        //达到了边界值，需要扩容  
        if (++size > threshold)  
            resize();  
        //什么都不做  
        afterNodeInsertion(evict);  
        //返回null  
        return null;  
    }  
```

### get方法源码：
当 HashMap 只存在数组，而数组中没有 Node 链表时，是 HashMap 查询数据性能最好的时候。

一旦发生大量的哈希冲突，就会产生 Node 链表，这个时候每次查询元素都可能遍历 Node 链表，从而降低查询数据的性能。

特别是在链表长度过长的情况下，性能明显下降， 使用红黑树 就很好地解决了这个问题，

红黑树使得查询的平均复杂度降低到了 O(log(n)) ，链表越长，使用红黑树替换后的查询效率提升就越明显。
```
public V get(Object key) {  
        Node<K,V> e;  
        return (e = getNode(hash(key), key)) == null ? null : e.value;  
    }  
    final Node<K,V> getNode(int hash, Object key) {  
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;  
        //数组不为null，数组长度大于0，根据hash计算出来的槽位的元素不为null  
        if ((tab = table) != null && (n = tab.length) > 0 &&  
            (first = tab[(n - 1) & hash]) != null) {  
            //查找的元素在数组中，返回该元素  
            if (first.hash == hash && // always check first node  
                ((k = first.key) == key || (key != null && key.equals(k))))  
                return first;  
            if ((e = first.next) != null) {//查找的元素在链表或红黑树中  
                if (first instanceof TreeNode)//元素在红黑树中，返回该元素  
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);  
                do {//遍历链表，元素在链表中，返回该元素  
                    if (e.hash == hash &&  
                        ((k = e.key) == key || (key != null && key.equals(k))))  
                        return e;  
                } while ((e = e.next) != null);  
            }  
        }  
        //找不到返回null  
        return null;  
    }  
```

### remove方法源码：
```
public V remove(Object key) {  
        Node<K,V> e;  
        return (e = removeNode(hash(key), key, null, false, true)) == null ?  
            null : e.value;  
    }  
    final Node<K,V> removeNode(int hash, Object key, Object value,  
                               boolean matchValue, boolean movable) {  
        Node<K,V>[] tab; Node<K,V> p; int n, index;  
        //数组不为null，数组长度大于0，要删除的元素计算的槽位有元素  
        if ((tab = table) != null && (n = tab.length) > 0 &&  
            (p = tab[index = (n - 1) & hash]) != null) {  
            Node<K,V> node = null, e; K k; V v;  
            //当前元素在数组中  
            if (p.hash == hash &&  
                ((k = p.key) == key || (key != null && key.equals(k))))  
                node = p;  
            //元素在红黑树或链表中  
            else if ((e = p.next) != null) {  
                if (p instanceof TreeNode)//是树节点，从树种查找节点  
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);  
                else {  
                    do {  
                        //hash相同，并且key相同，找到节点并结束  
                        if (e.hash == hash &&  
                            ((k = e.key) == key ||  
                             (key != null && key.equals(k)))) {  
                            node = e;  
                            break;  
                        }  
                        p = e;  
                    } while ((e = e.next) != null);//遍历链表  
                }  
            }  
            //找到节点了，并且值也相同  
            if (node != null && (!matchValue || (v = node.value) == value ||  
                                 (value != null && value.equals(v)))) {  
                if (node instanceof TreeNode)//是树节点，从树中移除  
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);  
                else if (node == p)//节点在数组中，  
                    tab[index] = node.next;//当前槽位置为null，node.next为null  
                else//节点在链表中  
                    p.next = node.next;//将节点删除  
                ++modCount;//修改计数器+1，为迭代服务  
                --size;//数量-1  
                afterNodeRemoval(node);//什么都不做  
                return node;//返回删除的节点  
            }  
        }  
        return null;  
    }  
```

### containsKey方法：
```
public boolean containsKey(Object key) {  
        return getNode(hash(key), key) != null;//查看上面的get的getNode  
    }  
```

### containsValue方法：
```
public boolean containsValue(Object value) {  
        Node<K,V>[] tab; V v;  
        //数组不为null并且长度大于0  
        if ((tab = table) != null && size > 0) {  
            for (int i = 0; i < tab.length; ++i) {//对数组进行遍历  
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {  
                    //当前节点的值等价查找的值，返回true  
                    if ((v = e.value) == value ||  
                        (value != null && value.equals(v)))  
                        return true;  
                }  
            }  
        }  
        return false;//找不到返回false  
    }  
```

### putAll方法
```
public void putAll(Map<? extends K, ? extends V> m) {  
        putMapEntries(m, true);  
    }  
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {  
        int s = m.size();//获得插入整个m的元素数量  
        if (s > 0) {  
            if (table == null) { // pre-size，当前map还没有初始化数组  
                float ft = ((float)s / loadFactor) + 1.0F;//m的容量  
                //判断容量是否大于最大值MAXIMUM_CAPACITY  
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?  
                         (int)ft : MAXIMUM_CAPACITY);  
                //容量达到了边界值，比如插入的m的定义容量是16，但当前map的边界值是12，需要对当前map进行重新计算边界值  
                if (t > threshold)  
                    threshold = tableSizeFor(t);//重新计算边界值  
            }  
            else if (s > threshold)//存放的数量达到了边界值，扩容  
                resize();  
            //对m进行遍历，放到当前map中  
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {  
                K key = e.getKey();  
                V value = e.getValue();  
                putVal(hash(key), key, value, false, evict);  
            }  
        }  
    }  
```

### clear方法
```
public void clear() {  
        Node<K,V>[] tab;  
        modCount++;//修改计数器+1，为迭代服务  
        if ((tab = table) != null && size > 0) {  
            size = 0;//将数组的元素格式置为0，然后遍历数组，将每个槽位的元素置为null  
            for (int i = 0; i < tab.length; ++i)  
                tab[i] = null;  
        }  
    }  
```

### replace方法
```
public boolean replace(K key, V oldValue, V newValue) {  
        Node<K,V> e; V v;  
        //根据hash计算得到槽位的节点不为null，并且节点的值等于旧值  
        if ((e = getNode(hash(key), key)) != null &&  
            ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {  
            e.value = newValue;//覆盖旧值  
            afterNodeAccess(e);  
            return true;  
        }  
        return false;  
    }  
  
    public V replace(K key, V value) {  
        Node<K,V> e;  
        //根据hash计算得到槽位的节点不为null  
        if ((e = getNode(hash(key), key)) != null) {  
            V oldValue = e.value;//节点的旧值  
            e.value = value;//覆盖旧值  
            afterNodeAccess(e);  
            return oldValue;//返回旧值  
        }  
        return null;//找不到key对应的节点  
    }  
```

## 扩容
在 JDK1.7 中，HashMap 整个扩容过程就是：

分别取出数组元素，一般该元素是最后一个放入链表中的元素，然后遍历以该元素为头的单向链表元素，依据每个被遍历元素的 hash 值计算其在新数组中的下标，然后进行交换。

这样的扩容方式，会将原来哈希冲突的单向链表尾部，变成扩容后单向链表的头部。

而在 JDK1.8 后，HashMap 对扩容操作做了优化。

由于扩容数组的长度是2倍关系，

所以对于假设初始 tableSize=4 要扩容到8来说就是 0100 到 1000 的变化（左移一位就是2倍），

在扩容中只用判断原来的 hash 值和 oldCap（旧数组容量）按位与操作是 0 或 1 就行:

- 0的话索引不变，
- 1的话索引变成原索引加扩容前数组。

之所以能通过这种“与”运算来重新分配索引，

是因为 hash 值本来是随机的，而 hash 按位与上 oldCap 得到的 0 和 1 也是随机的，

所以扩容的过程就能把之前哈希冲突的元素再随机分布到不同的索引中去。
```
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16，默认大小  
    //元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置  
    final Node<K,V>[] resize() {  
        Node<K,V>[] oldTab = table;//原先的数组，旧数组  
        int oldCap = (oldTab == null) ? 0 : oldTab.length;//旧数组长度  
        int oldThr = threshold;//阀值  
        int newCap, newThr = 0;  
        if (oldCap > 0) {//数组已经存在不需要进行初始化  
            if (oldCap >= MAXIMUM_CAPACITY) {//旧数组容量超过最大容量限制，不扩容直接返回旧数组  
                threshold = Integer.MAX_VALUE;  
                return oldTab;  
            }  
            //进行2倍扩容后的新数组容量小于最大容量和旧数组长度大于等于16  
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&  
                     oldCap >= DEFAULT_INITIAL_CAPACITY)  
                newThr = oldThr << 1; // double threshold，重新计算阀值为原来的2倍  
        }  
        //初始化数组  
        else if (oldThr > 0) // initial capacity was placed in threshold，有阀值，初始容量的值为阀值  
            newCap = oldThr;  
        else {               // zero initial threshold signifies using defaults，没有阀值  
            newCap = DEFAULT_INITIAL_CAPACITY;//初始化的默认容量  
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//重新计算阀值  
        }  
        //有阀值，定义了新数组的容量，重新计算阀值  
        if (newThr == 0) {  
            float ft = (float)newCap * loadFactor;  
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?  
                      (int)ft : Integer.MAX_VALUE);  
        }  
        threshold = newThr;//赋予新阀值  
        @SuppressWarnings({"rawtypes","unchecked"})  
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];//创建新数组  
        table = newTab;  
        if (oldTab != null) {//如果旧数组有数据，进行数据移动，如果没有数据，返回一个空数组  
            for (int j = 0; j < oldCap; ++j) {//对旧数组进行遍历  
                Node<K,V> e;  
                if ((e = oldTab[j]) != null) {  
                    oldTab[j] = null;//将旧数组的所属位置的旧元素清空  
                    if (e.next == null)//当前节点是在数组上，后面没有链表，重新计算槽位  
                        newTab[e.hash & (newCap - 1)] = e;  
                    else if (e instanceof TreeNode)//当前节点是红黑树，红黑树重定位  
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);  
                    else { // preserve order，当前节点是链表  
                        Node<K,V> loHead = null, loTail = null;  
                        Node<K,V> hiHead = null, hiTail = null;  
                        Node<K,V> next;  
                        //遍历链表  
                        do {  
                            next = e.next;  
                            if ((e.hash & oldCap) == 0) {//不需要移位  
                                if (loTail == null)//头节点是空的  
                                    loHead = e;//头节点放置当前遍历到的元素  
                                else  
                                    loTail.next = e;//当前元素放到尾节点的后面  
                                loTail = e;//尾节点重置为当前元素  
                            }  
                            else {//需要移位  
                                if (hiTail == null)//头节点是空的  
                                    hiHead = e;//头节点放置当前遍历到的元素  
                                else  
                                    hiTail.next = e;//当前元素放到尾节点的后面  
                                hiTail = e;//尾节点重置为当前元素  
                            }  
                        } while ((e = next) != null);  
                        if (loTail != null) {//不需要移位  
                            loTail.next = null;  
                            newTab[j] = loHead;//原位置  
                        }  
                        if (hiTail != null) {  
                            hiTail.next = null;  
                            newTab[j + oldCap] = hiHead;//移动到当前hash槽位 + oldCap的位置，即在原位置再移动2次幂的位置  
                        }  
                    }  
                }  
            }  
        }  
        return newTab;  
    }  
```

当前节点是数组，后面没有链表，重新计算槽位:位与操作的效率比效率高
```
    定位槽位：e.hash & (newCap - 1)  
     我们用长度16, 待插入节点的hash值为21举例:  
     (1)取余: 21 % 16 = 5  
     (2)位与:  
     21: 0001 0101  
             &  
     15: 0000 1111  
     5:  0000 0101  
```

遍历链表，对链表节点进行移位判断：(e.hash & oldCap) == 0
```
 比如oldCap=8,hash是3，11，19，27时，  
     （1）JDK1.8中(e.hash & oldCap)的结果是0，8，0，8，这样3，19组成新的链表，index为3；而11，27组成新的链表，新分配的index为3+8；  
     （2）JDK1.7中是(e.hash & newCap-1)，newCap是oldCap的两倍，也就是3，11，19，27对(16-1)与计算，也是0，8，0，8，但由于是使用了单链表的头插入方式，即同一位置上新元素总会被放在链表的头部位置；这样先放在一个索引上的元素终会被放到Entry链的尾部(如果发生了hash冲突的话），这样index为3的链表是19，3，index为3+8的链表是 27，11。  
     也就是说1.7中经过resize后数据的顺序变成了倒叙，而1.8没有改变顺序。  
```

## 问题
- hash冲突可以避免么？

理论上是没有办法避免的，就类比“抽屉原理”， 比如说一共有10个苹果，但是咱一共有9个抽屉，最终一定会有一个抽屉里的数量是大于1的， 所以hash冲突没有办法避免，只能尽量避免。

- 好的hash算法考虑的点，应该是哪些呢？

首先这个hash算法，它一定效率得高，要做到长文本也能高效计算出hash值， 这二点就是hash值不能让它逆推出原文吧； 两次输入，只要有一点不同，它也得保证这个hash值是不同的。

其次，就是尽可能的要分散吧，因为，在table中slot中slot大部分都处于空闲状的，要尽可能降低hash冲突。

- HashMap中存储数据的结构，长什么样啊？

JDK1.7 是 数组 + 链表；
JDK1.8是 数组 + 链表 + 红黑树，每个数据单元都是一个Node结构，Node结构中有key字段、有value字段、还有next字段、还有hash字段。

Node结构next字段就是发生hash冲突的时候，当前桶位中node与冲突的node连成一个链表要用的字段。

- 那这个散列表，new HashMap() 的时候就创建了，还是说在什么时候创建的？
散列表是懒加载机制， 只有第一次put数据的时候，它才创建的

- 链表它转化为这个红黑树需在达到什么条件？
链表转红黑树，主要是有两个指标，其中一个就是链表长度达到8，还有一个指标就是当前散列表数组长度它已经达到64。

如果前散列表数组长度它已经达到64，就算slot内部链表长度到了8，它也不会链转树， 它仅仅会发生一次resize，散列表扩容。

- Node对象hash值与key对象的hashcode() 有什么关系？
Node对象hash值是key.hashcode二次加工得到的。 加工原则是： key的hashcode 高16位 ^ 低16位，得到的一个新值。

主要为了优化hash算法，近可能的分散得比较均匀，尽可能的减少 碰撞 因为hashmap内部散列表，它大多数场景下，它不会特别大。

hashmap内部散列表的长度，也就是说 length - 1 对应的 二进制数，实际有效位很有限，一般都在（低）16位以内，

- hashmap Put写数据的具体流程，尽可能的详细点去说

主要为4种情况：
前面这个，寻址算法是一样的，都是根据key的hashcode 经过 高低位 异或 之后的值，然后再 按位与 & （table.length -1)，得到一个槽位下标，然后根据这个槽内状况，状况不同，情况也不同，大概就是4种状态，

第一种是slot == null，直接占用slot就可以了，然后把当前put方法传进来的key和value包状成一个Node 对象，放到这个slot中就可以了

第二种是slot != null 并且 它引用的node 还没有链化；需要对比一下，node的key 与当前put 对象的key 是否完全相等；

如果完全相等的话，这个操作就是replace操作，就是替换操作，把那个新的value替换当前slot -> node.value 就可以了；

否则的话，这次put操作就是一个正儿八经的hash冲突了，slot->node 后面追加一个node就可以了，采用尾插法。

第三种就是slot 内的node已经链化了；

这种情况和第二种情况处理很相似，首先也是迭代查找node，看看链表上的元素的key，与当前传来的key是不是完全一致。如果一致的话，还是repleace操作，替换当前node.value，否则的话就是我们迭代到链表尾节点也没有匹配到完全一致的node，把put数据包装成node追加到链表尾部；

这块还没完，还需要再检查一下当前链表长度，有没有达到树化阈值，如果达到阈值的话，就调用一个树化方法，树化操作都在这个方法里完成

第四种就是冲突很严重的情况下，就是那个链已经转化成红黑树了

- jdk8 HashMap为什么要引入红黑树呢？
其实主要就是解决hash冲突导致链化严重的问题，如果链表过长，查找时间复杂度为O(n)，效率变慢。

本身散列表最理想的查询效率为O(1)，但是链化特别严重，就会导致查询退化为O(n)。

严重影响查询性能了，为了解决这个问题，JDK1.8它才引入的红黑树。红黑树其实就是一颗特殊的二叉排序树，这个时间复杂度是log(N)

- 那为什么链化之后性能就变低了呀？
因为链表它毕竟不是数组，它从内存角度来看，它没有连续着。

如果我们要往后查询的话，要查询的数据它在链表末尾，那只能从链表一个节点一个节点Next跳跃过去，非常耗费性能。

- 再聊聊hashmap的扩容机制吧？你说一下，什么情况下会触发这个扩容呢？
在写数据之后会触发扩容，可能会触发扩容。hashmap结构内，我记得有个记录当前数据量的字段，这个数据量字段达到扩容阈值的话，下一个写入的对象是在列表才会触发扩容

- 扩容后会扩容多大呢？这块算法是咋样的呢？
因为table 数组长度必须是2的次方数嘛，扩容其实，每次都是按照上一次的tableSize位移运算得到的。就是做一次左移1位运算，假设当前tableSize是16的话，16 << 1 == 32

- 这里为什么要采用位移运算呢？咋不直接tableSize乘以2呢？
主要是因为性能，因为cpu毕竟它不支持乘法运算，所有乘法运算它最终都是在指令层面转化为加法实现的。 效率很低，如果用位运算的话对cpu来说就非常简洁高效

- 创建新的扩容数组，老数组中的这个数据怎么迁移呢？
迁移其实就是，每个桶位推进迁移，就是一个桶位一个桶位的处理； 主要还是看当前处理桶位的数据状态吧