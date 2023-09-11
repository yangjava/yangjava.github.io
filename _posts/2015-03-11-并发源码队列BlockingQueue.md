---
layout: post
categories: [JUC]
description: none
keywords: JUC
---
# 并发源码BlockingQueue

## BlockingQueue（阻塞队列）
所谓Blocking Queue是指其中的元素数量存在界限，当队列已满时（队列元素数量达到了最大容量的临界值），对队列进行写入操作的线程将被阻塞挂起，当队列为空时（队列元素数量达到了为0的临界值），对队列进行读取的操作线程将被阻塞挂起。

实际上，BlockingQueue（LinkedTransferQueue除外）的内部实现主要依赖于显式锁Lock及其与之关联的Condition。BlockingQueue的实现都是线程安全的队列，在高并发的程序开发中，可以不用担心线程安全的问题而直接使用，另外，BlockingQueue在线程池服务（ExecutorService）中主要扮演着提供线程对任务存取容器的角色。

BlockingQueue接口为多线程操作同一个队列提供的4种处理方案为：
- 抛出异常。在通过add(e)方法向队列尾部添加元素时，如果队列已满，则抛出IllegalStateException。在通过remove()和element()方法，分别从队列头部删除或读取元素时，如果队列为空，则抛出NoSuchElementException。
- 返回特定值。在通过offer(e)方法向队列尾部添加元素时，如果队列已满，则返回false。在通过poll()和peek()方法，分别从队列头部删除或读取元素时，如果队列为空，则返回null。
- 线程阻塞。在通过put(e)方法向队列尾部添加元素时，如果队列已满，则当前线程进入阻塞状态，直到队列有剩余的容量来添加元素，才退出阻塞。在通过take()方法从队列头部删除并返回元素时，如果队列为空，则当前线程进入阻塞状态，直到从队列中成功删除并返回元素为止。
- 超时。超时和以上线程阻塞有一些共同之处。两者的区别在于当线程在特定条件下进入阻塞状态后，如果超过了offer(e,time,unit)和poll(time,unit)方法的参数所设置的时间限制，那么也会退出阻塞状态，分别返回false和null。例如poll(100, TimeUnit.MILLISECONDS)表示设置的时间限制为100毫秒。

在java.util.concurrent包中，BlockingQueue接口主要有以下实现类：
- LinkedBlockingQueue类：默认情况下，LinkedBlockingQueue的容量是没有上限的（确切地说，默认的容量为Integer.MAX_VALUE），但是也可以选择指定其最大容量。它是基于链表的队列，此队列按FIFO（先进先出）的原则存取元素。
- ArrayBlockingQueue类：它的ArrayBlockingQueue(int capacity, boolean fair)构造方法允许指定队列的容量，并可以选择是否采用公平策略。如果参数fair被设置为true，那么等待时间最长的线程会优先访问队列（其底层实现是通过将ReentrantLock设置为使用公平策略来达到这种公平性的）。通常，公平性以降低运行性能为代价，所以只有在确实非常需要的时候才使用这种公平机制。ArrayBlockingQueue是基于数组的队列，此队列按FIFO（先进先出）的原则存取元素。
- PriorityBlockingQueue是一个带优先级的队列，而不是先进先出队列。元素按优先级顺序来删除，该队列的容量没有上限。
- DelayQueue：在这个队列中存放的是延期元素，也就是说这些元素必须实现java.util.concurrent.Delayed接口。只有延迟期满的元素才能被取出或删除。当一个元素的getDelay(TimeUnit unit)方法返回一个小于或等于零的值时，则表示延迟期满。DelayQueue队列的容量没有上限。
- SynchronousQueue：基于 CAS 的阻塞队列。

## BlockingQueue说明       
BlockingQueue 通常用于一个线程生产对象，而另外一个线程消费这些对象的场景。一个线程往里边放，另外一个线程从里边取的一个 BlockingQueue。

阻塞队列是一个可以阻塞的先进先出集合，比如某个线程在空队列获取元素时、或者在已存满队列存储元素时，都会被阻塞。

## BlockingQueue 接口实现类
ArrayBlockingQueue ：基于数组的有界阻塞队列，必须指定大小。
LinkedBlockingQueue ：基于单链表的无界阻塞队列，不需指定大小。
PriorityBlockingQueue ：基于最小二叉堆的无界、优先级阻塞队列。
DelayQueue：基于延迟、优先级、无界阻塞队列。
SynchronousQueue ：基于 CAS 的阻塞队列。
