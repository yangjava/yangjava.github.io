---
layout: post
categories: [JUC]
description: none
keywords: JUC
---
# 并发源码BlockingQueue

## DelayQueue
DelayQueue也是一个实现了BlockingQueue接口的“无边界”阻塞队列，但是该队列却是非常有意思和特殊的一个队列（存入DelayQueue中的数据元素会被延迟单位时间后才能消费），在DelayQueue中，元素也会根据优先级进行排序，这种排序可以是基于数据元素过期时间而进行的（比如，你可以将最快过期的数据元素排到队列头部，最晚过期的数据元素排到队尾）。

对于存入DelayQueue中的元素是有一定要求的：元素类型必须是Delayed接口的子类，存入DelayQueue中的元素需要重写getDelay(TimeUnit unit)方法用于计算该元素距离过期的剩余时间，如果在消费DelayQueue时发现并没有任何一个元素到达过期时间，那么对该队列的读取操作会立即返回null值，或者使得消费线程进入阻塞。

### DelayQueue的基本使用
从前文的描述中我们可以得知，DelayQueue中的元素都必须是Delayed接口的子类，该接口继承自Comparable<Delayed>接口，并且定义了一个唯一的接口方法getDelay，如下所示。
```java
public interface Delayed extends Comparable<Delayed> {
    long getDelay(TimeUnit unit);
}
```
所以，首先需要实现Delayed接口，并且重写getDelay方法和compareTo方法。
```java
// 继承自Delayed接口
class DelayedEntry implements Delayed
{
    // 元素数据内容
    private final String value;
    // 用于计算失效时间
    private final long time;

    private DelayedEntry(String value, long delayTime)
    {
        this.value = value;
    // 该元素可在（当前时间+delayTime）毫秒后消费，也就是说延迟消费delayTime毫秒
        this.time = delayTime + System.currentTimeMillis();
    }
    // 重写getDelay方法，返回当前元素的延迟时间还剩余（remaining）多少个时间单位
    @Override
    public long getDelay(TimeUnit unit)
    {
        long delta = time - System.currentTimeMillis();
        return unit.convert(delta, TimeUnit.MILLISECONDS);
    }

    public String getValue()
    {
        return value;
    }
    // 重写compareTo方法，根据我们所实现的代码可以看出，队列头部的元素是最早即将失效的数据元素
    @Override
    public int compareTo(Delayed o)
    {
        if (this.time < ((DelayedEntry) o).time)
        {
            return -1;
        } else if (this.time > ((DelayedEntry) o).time)
        {
            return 1;
        } else
            return 0;
    }
    @Override
    public String toString()
    {
        return "DelayedEntry{" +
                "value='" + value + '\" +
                ", time=" + time +
                '}';
    }
}
```
在DelayQueue中，每一个元素都必须是Delayed接口的子类，在上面的代码中，我们实现的DelayedEntry就是Delayed的子类，现在我们可以在DelayQueue中正常地存取DelayedEntry了。
```java
// 定义DelayQueue，无需指定容量，因为DelayQueue是一个"无边界"的阻塞队列
DelayQueue<DelayedEntry> delayQueue = new DelayQueue<>();
// 存入数据A，数据A将在10000毫秒后过期，或者说会被延期10000毫秒后处理
delayQueue.put(new DelayedEntry("A", 10 * 1000L));
// 存入数据A，数据B将在5000毫秒后过期，或者说会被延期5000毫秒后处理
delayQueue.put(new DelayedEntry("B", 5 * 1000L));
// 记录时间戳
final long timestamp = System.currentTimeMillis();
// 非阻塞读方法，立即返回null，原因是当前AB元素不会有一个到达过期时间
assert delayQueue.poll() == null;


// take方法会阻塞5000毫秒左右，因为此刻队列中最快达到过期条件的数据B只能在5000毫秒以后
DelayedEntry value = delayQueue.take();
// 断言队列头部的元素为B
assert value.getValue().equals("B");
// 耗时5000毫秒或以上
assert (System.currentTimeMillis() - timestamp) >= 5_000L;

// 再次执行take操作
value = delayQueue.take();
// 断言队列头部的元素为A
assert value.getValue().equals("A");
// 耗时在10000毫秒或以上
assert (System.currentTimeMillis() - timestamp) >= 10_000L;
```

### 读取DelayQueue中的数据
DelayQueue队列区别于我们之前学习过的队列，其中之一就是存入该队列的元素必须是Delayed的子类，除此之外队列中的数据元素会被延迟（Delay）消费，这也正是延迟队列名称的由来。与PriorityBlockingQueue一样，DelayQueue中有关增加元素的所有方法都等价于offer(E e)，并不存在针对队列临界值上限的控制，因此也不存在阻塞写的情况（多线程争抢导致的线程阻塞另当别论）但是对该队列中数据元素的消费（延迟消费）则有别于本节中接触过的其他阻塞队列。

- remainingCapacity()方法始终返回Integer.MAX_VALUE
```java
DelayQueue<DelayedEntry> delayQueue = new DelayQueue<>();
assert delayQueue.size() == 0;
assert delayQueue.remainingCapacity() == Integer.MAX_VALUE;
delayQueue.put(new DelayedEntry("A", 10 * 1000L));
delayQueue.put(new DelayedEntry("B", 5 * 1000L));
assert delayQueue.size() == 2;
assert delayQueue.remainingCapacity() == Integer.MAX_VALUE;
```
- peek()：非阻塞读方法，立即返回但并不移除DelayQueue的头部元素，当队列为空时返回null。
```java
DelayQueue<DelayedEntry> delayQueue = new DelayQueue<>();
// 队列为空时，peek方法立即返回null
assert delayQueue.peek()==null;
delayQueue.put(new DelayedEntry("A", 10 * 1000L));
delayQueue.put(new DelayedEntry("B", 5 * 1000L));

// 队列不为空时，peek方法不会出现延迟，而且立即返回队列头部的元素，但不移除
assert delayQueue.peek().getValue().equals("B");
```

- poll():非阻塞读方法，当队列为空或者队列头部元素还未到达过期时间时返回值为null，否则将会从队列头部立即将元素移除并返回。
```java
DelayQueue<DelayedEntry> delayQueue = new DelayQueue<>();
// 队列为空，立即返回null
assert delayQueue.poll() == null;

// 队列中存入数据
delayQueue.put(new DelayedEntry("A", 10 * 1000L));
delayQueue.put(new DelayedEntry("B", 5 * 1000L));
// 队列不为空，但是队头元素并未达到超时时间，立即返回null
assert delayQueue.poll() == null;

// 休眠5秒，使得头部元素到达超时时间
TimeUnit.SECONDS.sleep(5);
// 立即返回元素B
assert delayQueue.poll().getValue().equals("B");
```

- poll(long timeout, TimeUnit unit)：最大阻塞单位时间，当达到阻塞时间后，此刻为空或者队列头部元素还未达到过期时间时返回值为null，否则将会立即从队列头部将元素移除并返回。
```java
DelayQueue<DelayedEntry> delayQueue = new DelayQueue<>();
// 队列为空，该方法会阻塞10毫秒并且返回null
assert delayQueue.poll(10, TimeUnit.MILLISECONDS) == null;

delayQueue.put(new DelayedEntry("A", 10 * 1000L));
delayQueue.put(new DelayedEntry("B", 5 * 1000L));

// 队列不为空，但是队头元素在10毫秒内不会达到过期时间
assert delayQueue.poll(10, TimeUnit.MILLISECONDS) == null;
// 休眠5秒
TimeUnit.SECONDS.sleep(5);

// 移除并返回B
assert delayQueue.poll(10, TimeUnit.MILLISECONDS)
.getValue().equals("B");
```

- take()：阻塞式的读取方法，该方法会一直阻塞到队列中有元素，并且队列中的头部元素已达到过期时间，然后将其从队列中移除并返回。
