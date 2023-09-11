---
layout: post
categories: [JUC]
description: none
keywords: JUC
---
# 并发源码BlockingQueue

## SynchronousQueue
SynchronousQueue也是实现自BlockingQueue的一个阻塞队列，每一次对其的写入操作必须等待（阻塞）其他线程进行对应的移除操作，SynchronousQueue的内部并不会涉及容量、获取size，就连peek方法的返回值永远都将会是null，除此之外还有更多的方法在SynchronousQueue中也都未提供对应的支持（列举如下），因此在使用的过程中需要引起注意，否则会使得程序的运行出现不符合预期的错误。
- clear()：清空队列的方法在SynchronousQueue中不起任何作用。
- contains(Object o)：永远返回false。
- containsAll(Collection<?> c)：等价于c是否为空的判断。
- isEmpty()：永远返回true。
- iterator()：返回一个空的迭代器。
- peek()：永远返回null。
- remainingCapacity()：始终返回0。
- remove(Object o)：不做任何删除，并且始终返回false。
- removeAll(Collection<?> c)：不做任何删除，始终返回false。
- retainAll(Collection<?> c)：始终返回false。
- size()：返回值始终为0。
- spliterator()：返回一个空的Spliterator（关于Spliterator，我们会在本书的第6章的6.3.2节“Spliterator详解”一节进行详细介绍）。
- toArray()及toArray(T[] a)方法同样也不支持。

看起来好多方法在SynchronousQueue中都不提供对应的支持，那么SynchronousQueue是一个怎样的队列呢？简单来说，我们可以借助于SynchronousQueue在两个线程间进行线程安全的数据交换。

尽管SynchronousQueue是一个队列，但是它的主要作用在于在两个线程之间进行数据交换，区别于Exchanger的主要地方在于（站在使用的角度）SynchronousQueue所涉及的一对线程一个更加专注于数据的生产，另一个更加专注于数据的消费（各司其职），而Exchanger则更加强调一对线程数据的交换。打开Exchanger的官方文档，可以看到如下的一句话：
```java
An Exchanger may be viewed as a bidirectional form of a {@link SynchronousQueue}. Exchanger 可以看作一个双向的SynchronousQueue
```

SynchronousQueue在日常的开发使用中并不是很常见，即使在JDK内部，该队列也仅用于ExecutorService中的Cache Thread Pool创建，本节只是简单了解一下SynchronousQueue的基本使用方法即可。
```java
// 定义String类型的SynchronousQueue
SynchronousQueue<String> queue = new SynchronousQueue<>();

// 启动两个线程，向queue中写入数据
IntStream.rangeClosed(0, 1).forEach(i ->
        new Thread(() ->
        {
            try
            {
        // 若没有对应的数据消费线程，则put方法将会导致当前线程进入阻塞
                queue.put(currentThread().getName());
                System.out.println(currentThread() + " put element " + currentThread().getName());
            } catch (InterruptedException e)
            {
                e.printStackTrace();
            }
        }).start()
);

// 启动两个线程从queue中消费数据
IntStream.rangeClosed(0, 1).forEach(i ->
        new Thread(() ->
        {
            try
            {
        // 若没有对应的数据生产线程，则take方法将会导致当前线程进入阻塞
                String value = queue.take();
                System.out.println(currentThread() + " take " + value);
            } catch (InterruptedException e)
            {
                e.printStackTrace();
            }
        }).start()
);
// 运行上面的程序将得到如下所示的输出结果
Thread[Thread-2,5,main] take Thread-0
Thread[Thread-0,5,main] put element Thread-0
Thread[Thread-3,5,main] take Thread-1
Thread[Thread-1,5,main] put element Thread-1
```
