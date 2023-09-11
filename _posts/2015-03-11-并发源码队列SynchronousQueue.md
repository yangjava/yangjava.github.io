---
layout: post
categories: [JUC]
description: none
keywords: JUC
---
# 并发源码BlockingQueue

## ArrayBlockingQueue
ArrayBlockingQueue是一个基于数组结构实现的FIFO阻塞队列，在构造该阻塞队列时需要指定队列中最大元素的数量（容量）。当队列已满时，若再次进行数据写入操作，则线程将会进入阻塞，一直等待直到其他线程对元素进行消费。当队列为空时，对该队列的消费线程将会进入阻塞，直到有其他线程写入数据。该阻塞队列中提供了不同形式的读写方法。

### 阻塞式写方法
在ArrayBlockingQueue中提供了两个阻塞式写方法，分别如下（在该队列中，无论是阻塞式写方法还是非阻塞式写方法，都不允许写入null）。

- void put(E e)：向队列的尾部插入新的数据，当队列已满时调用该方法的线程会进入阻塞，直到有其他线程对该线程执行了中断操作，或者队列中的元素被其他线程消费。
```java
// 构造只有两个元素容量的ArrayBlockingQueue
ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<>(2);
        try
        {
        queue.put("first");
        queue.put("second");
        // 执行put将会使得当前线程进入阻塞
        queue.put("third");
        } catch (InterruptedException e)
        {
        e.printStackTrace();
        }
```
- boolean offer(E e, long timeout, TimeUnit unit)：向队列尾部写入新的数据，当队列已满时执行该方法的线程在指定的时间单位内将进入阻塞，直到到了指定的超时时间后，或者在此期间有其他线程对队列数据进行了消费。当然了，对由于执行该方法而进入阻塞的线程执行中断操作也可以使当前线程退出阻塞。该方法的返回值boolean为true时表示写入数据成功，为false时表示写入数据失败。
```java
ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<>(2);
try
{
    queue.offer("first", 10, TimeUnit.SECONDS);
    queue.offer("second", 10, TimeUnit.SECONDS);
    // 该方法会进入阻塞，10秒之后当前线程将会退出阻塞，并且对third数据的写入将会失败
    queue.offer("third", 10, TimeUnit.SECONDS);
} catch (InterruptedException e)
{
    e.printStackTrace();
}
```

### 非阻塞式写方法
当队列已满时写入数据，如果不想使得当前线程进入阻塞，那么就可以使用非阻塞式的写操作方法。

- boolean add(E e)：向队列尾部写入新的数据，当队列已满时不会进入阻塞，但是该方法会抛出队列已满的异常。
```java
ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<>(2);
// 写入元素成功
assert queue.add("first");
assert queue.add("second");
try
{
    // 写入失败，抛出异常
    queue.add("third");
} catch (Exception e)
{
    // 断言异常
    assert e instanceof IllegalStateException;
}
```
- boolean offer(E e)：向队列尾部写入新的数据，当队列已满时不会进入阻塞，并且会立即返回false。
```java
ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<>(2);
assert queue.offer("first");
assert queue.offer("second");
// 写入失败
assert !queue.offer("third");
// 第三次offer操作失败，此时队列的size为2
assert queue.size() == 2;
```

### 阻塞式读方法
ArrayBlockingQueue中提供了两个阻塞式读方法，分别如下。

- E take()：从队列头部获取数据，并且该数据会从队列头部移除，当队列为空时执行take方法的线程将进入阻塞，直到有其他线程写入新的数据，或者当前线程被执行了中断操作。
```java
ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<>(2);
assert queue.offer("first");
assert queue.offer("second");
try
{
    // 由于是队列，因此这里的断言语句也遵从FIFO，第一个被take出来的数据是first
    assert queue.take().equals("first");
    assert queue.take().equals("second");
    // 进入阻塞
    queue.take();
} catch (InterruptedException e)
{
    e.printStackTrace();
}
```

- E poll(long timeout, TimeUnit unit)：从队列头部获取数据并且该数据会从队列头部移除，如果队列中没有任何元素时则执行该方法，当前线程会阻塞指定的时间，直到在此期间有新的数据写入，或者阻塞的当前线程被其他线程中断，当线程由于超时退出阻塞时，返回值为null。
```java
ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<>(2);
assert queue.offer("first");
assert queue.offer("second");
try
{
    // FIFO
    assert queue.poll(10, TimeUnit.SECONDS).equals("first");
    assert queue.poll(10, TimeUnit.SECONDS).equals("second");
    // 10秒以后线程退出阻塞，并且返回null值。
    assert queue.poll(10, TimeUnit.SECONDS) == null;
} catch (InterruptedException e)
{
    e.printStackTrace();
}
```

### 非阻塞式读方法
当队列为空时读取数据，如果不想使得当前线程进入阻塞，那么就可以使用非阻塞式的读操作方法。

- E poll()：从队列头部获取数据并且该数据会从队列头部移除，当队列为空时，该方法不会使得当前线程进入阻塞，而是返回null值。
```java
ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<>(2);
assert queue.offer("first");
assert queue.offer("second");
// FIFO
assert queue.poll().equals("first");
assert queue.poll().equals("second");
// 队列为空，立即返回但是结果为null
assert queue.poll() == null;
```

- E peek()：peek的操作类似于debug操作（仅仅debug队列头部元素，本书的第6章将讲解针对Stream的操作，大家将从中学习到针对整个Stream数据元素的peek操作），它直接从队列头部获取一个数据，但是并不能从队列头部移除数据，当队列为空时，该方法不会使得当前线程进入阻塞，而是返回null值。
```java
ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<>(2);
assert queue.offer("first");
assert queue.offer("second");
// 第一次peek，从队列头部读取数据
assert queue.peek().equals("first");
// 第二次peek，从队列头部读取数据，同第一次
assert queue.peek().equals("first");
// 清除数据，队列为空
queue.clear();
// peek操作返回结果为null
assert queue.peek() == null;
```

## 生产者消费者
高并发多线程的环境下对共享资源的访问，在绝大多数情况下都可以通过生产者消费者模式进行理论化概括化，ArrayBlocking Queue在高并发的环境中同时读写的代码如下。
```java
// 定义阻塞队列
ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<>(10);
// 启动11个生产数据的线程，向队列的尾部写入数据
IntStream.rangeClosed(0, 10)
.boxed()
.map(i -> new Thread("P-Thread-" + i)
{
    @Override
    public void run()
    {
        while (true)
        {
            try
            {
                String data = String.valueOf(System.currentTimeMillis());
                queue.put(data);
                System.out.println(currentThread() + " produce data: " + data);
                TimeUnit.SECONDS.sleep(ThreadLocalRandom.current().nextInt(5));
            } catch (InterruptedException e)
            {
                System.out.println("Received the interrupt SIGNAL.");
                break;
            }
        }
    }
}).forEach(Thread::start);

// 定义11个消费线程，从队列的头部移除数据
IntStream.rangeClosed(0, 10)
.boxed()
.map(i -> new Thread("C-Thread-" + i)
{
    @Override
    public void run()
    {
        while (true)
        {
            try
            {
                String data = queue.take();
                System.out.println(currentThread() + " consume data: " + data);
                TimeUnit.SECONDS.sleep(ThreadLocalRandom.current().nextInt(5));
            } catch (InterruptedException e)
            {
                System.out.println("Received the interrupt SIGNAL.");
                break;
            }
        }
    }
}).forEach(Thread::start);
```
在上面的程序中，有22个针对queue的操作线程，我们并未提供对共享数据queue的线程安全保护措施，甚至没有进行任何临界值的判断与线程的挂起/唤醒动作，这一切都由该阻塞队列内部实现，因此开发者再也无需实现类似的队列，进行不同类型线程的数据交换和通信，运行上面的代码将会看到生产者与消费者在不断地交替输出。

除此之外，该阻塞队列还提供了一些其他方法，比如drainTo()排干队列中的数据到某个集合、remainingCapacity()获取剩余容量等，ArrayBlockingQueue除了实现了BlockingQueue定义的所有接口方法之外它还是Collection接口的实现类。
