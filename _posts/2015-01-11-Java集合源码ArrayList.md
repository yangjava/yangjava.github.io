---
layout: post
categories: [Java]
description: none
keywords: Java
---
# Java集合源码ArrayList


## ArrayList
ArrayList就是动态数组,就是Array的复杂版本，它提供了动态的增加和减少元素。它的底层就是数组队列，相对于Java中的数组来说，它的容量可以动态增长，因此被称为动态数组。正因为自动扩容机制，ArrayList已经成为平时最常用的集合类（以下的讲解基于jdk1.8版本）。

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    
}
```
可以看到ArrayList实现了四个接口
- java.util.List：提供数组的添加、删除、修改、迭代遍历等操作
- java.util.RandomAccess ：表示 ArrayList 支持快速的随机访问
- java.io.Serializable ：表示 ArrayList 支持序列化的功能
- java.lang.Cloneable ：表示 ArrayList 支持克隆

ArrayList同时继承了一个抽象类
- java.util.AbstractList：提供了List接口的骨架实现，大幅度的减少了实现迭代遍历相关操作的代码（实际上ArrayList大量重写了AbstractList的提供的方法，所以，AbstractList 对于 ArrayList 意义不大，其内部方法更多用于 AbstractList 其它子类）。

### 属性和构造方法
属性
```
    /**
     * 序列化版本号
     */
	private static final long serialVersionUID = 8683452581122892189L;

    /**
     * 默认初始容量大小
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 空数组（用于空实例）
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * 用于默认大小空实例的共享空数组实例
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 保存ArrayList数据的数组
     */
    transient Object[] elementData;

    /**
     * ArrayList 所包含的元素个数
     */
    private int size;

	/**
     * 最大数组容量
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```
其中核心的两个属性
- elementData：存放元素的数组，可以动态扩容。
- size：元素数量，这里指的数组中被使用的元素的个数

### 构造方法
ArrayList一共有三个构造方法
- 无参构造方法
```
/**
     * Constructs an empty list with an initial capacity of ten.
     */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```
初始化的时候默认为DEFAULTCAPACITY_EMPTY_ELEMENTDATA这个空数组，即初始容量为0，只有首次添加元素时才真正初始化为容量10，这可能是考虑节省内存。

注意：为啥不使用直接使用EMPTY_ELEMENTDATA ？这是因为EMPTY_ELEMENTDATA 按照1.5倍扩容而非从10开始，即起点不同，下面的扩容机制会详细说明。

- 带初始容量的有参构造方法
```
/**
     * Constructs an empty list with the specified initial capacity.
     *
     * @param  initialCapacity  the initial capacity of the list
     * @throws IllegalArgumentException if the specified initial capacity
     *         is negative
     */
public ArrayList(int initialCapacity) {
    //初始容量大于0时，创建指定大小的Object数组
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    }
    //初始容量等于0时，直接引用到EMPTY_ELEMENTDATA属性
    else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } 
    //初始容量小于0时，抛出IllegalArgumentException异常
    else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```
注意：初始容量指定为0时，使用EMPTY_ELEMENTDATA空数组，而无参构造方法时使用DEFAULTCAPACITY_EMPTY_ELEMENTDATA共享空数组，后面的扩容机制会讲解二者的不同。

- 带集合的有参构造方法
```
/**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection whose elements are to be placed into this list
     * @throws NullPointerException if the specified collection is null
     */
public ArrayList(Collection<? extends E> c) {
    //将传进来的集合c转换成Object数组并赋值给elementData
    elementData = c.toArray();
    //如果数组长度大于0，即数组不为空
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        // 如果集合元素不是 Object[] 类型，则会创建新的 Object[] 数组，并将 elementData 赋值到其中，最后赋值给 elementData
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } 
    //如果数组为空，则引用指向EMPTY_ELEMENTDATA，类似带初始容量为0的有参构造方法
    else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```
该方法不常用。

### 扩容机制
ArrayList提供两种扩容机制，一种自动扩容，用户无法显示调用；另一种手动扩容，即提供给用户直接调用的。

- 自动扩容
自动扩容主要发生在添加元素的方法中，下面以add方法为例一步一步剖析扩容

先看add方法
```
/**
     * 添加指定元素到列表末尾
     */
public boolean add(E e) {
    //添加元素前先调用ensureCapacityInternal方法
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //为数组赋值，很简单的操作
    elementData[size++] = e;
    return true;
}
```
可以看到添加的时候预先需要进行扩容判断，扩容结束后直接添加元素到数组末尾即可。可以看到ensureCapacityInternal()方法即为我们想要的扩容逻辑。

ensureCapacityInternal()
```
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

//计算最小扩容量
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    //如果elementData引用指向DEFAULTCAPACITY_EMPTY_ELEMENTDATA属性，即默认大小空实例的共享空数组实例，也就是无参构造函数初始化的空实例，会取指定大小和默认大小（10）的较大值。也就是无参构造方法那一块所说的，初始时刻数组大小为空，首次添加元素时会初始化为默认大小（10）
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
```
这里的计算最小扩容量实质就是判断是否是无参构造函数首次添加元素，如果是则初始化为默认大小10，否则不调整指定的扩容量。

ensureExplicitCapacity()
```
private void ensureExplicitCapacity(int minCapacity) {
    //修改的次数自增
    modCount++;

    //如果需要扩容的大小比当前数组大小大，则进行扩容
    //这里分析一下为啥要加这个判断。无参构造函数首次添加元素的时候，elementData的大小还是0，根据上面的分析，首次新增结束后数组长度会变为10，这样第2次、第3次...到第10次都不用去扩容，所以这里加了个判断。
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```
这里终于到了我们想要的真正的扩容方法——grow()，接下来重点分析该方法的扩容逻辑

grow()
```
/**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
private void grow(int minCapacity) {
    // overflow-conscious code
    //数组旧容量
    int oldCapacity = elementData.length;
    //准新容量=旧容量的1.5倍（这里使用移位计算提供计算速度）
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    //如果准新容量比最小需要容量还要小，则最小需要容量当做准新容量
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    //如果准新容量大于最大的数组大小，需要进一步进行计算（后面讲解）
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
扩容的主要逻辑是先计算预计的新容量（原来的1.5倍大小），然后根据最小所需容量和最大数组大小进行调整。

