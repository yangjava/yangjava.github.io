---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码自定义队列

## DataCarrier Buffer
Agent采集到的链路数据会先放到DataCarrier中，由消费者线程读取DataCarrier中的数据上报到OAP

## QueueBuffer
DataCarrier是使用Buffer作为数据存储，Buffer的底层接口是QueueBuffer，代码如下：
```
/**
 * Queue buffer interface.
 */
public interface QueueBuffer<T> {
    /**
     * 保存数据到队列中
     * Save data into the queue;
     *
     * @param data to add.
     * @return true if saved
     */
    boolean save(T data);

    /**
     * 设置队列满时的处理策略
     * Set different strategy when queue is full.
     */
    void setStrategy(BufferStrategy strategy);

    /**
     * 队列中的数据放到consumeList中并清空队列
     * Obtain the existing data from the queue
     */
    void obtain(List<T> consumeList);

    int getBufferSize();
}

```

BufferStrategy定义了队列满时的处理策略：
```
public enum BufferStrategy {
    /**
     * 阻塞,等待队列有空位置
     */
    BLOCKING,
    /**
     * 能放就放,不能放就算了
     */
    IF_POSSIBLE
}

```
QueueBuffer有两个实现Buffer和ArrayBlockingQueueBuffer

## Buffer
Buffer是一个环形队列，代码如下：
```
/**
 * 实现了环形队列
 * Self implementation ring queue.
 */
public class Buffer<T> implements QueueBuffer<T> {
    private final Object[] buffer; // 队列数据都存储到数组中
    private BufferStrategy strategy; // 队列满时的处理策略
    private AtomicRangeInteger index; // 索引

    Buffer(int bufferSize, BufferStrategy strategy) {
        buffer = new Object[bufferSize];
        this.strategy = strategy;
        index = new AtomicRangeInteger(0, bufferSize);
    }

    @Override
    public void setStrategy(BufferStrategy strategy) {
        this.strategy = strategy;
    }

    @Override
    public boolean save(T data) {
        // 拿到队列下一个位置的下标
        int i = index.getAndIncrement();
        if (buffer[i] != null) {
            switch (strategy) {
                case IF_POSSIBLE:
                    return false;
                default:
            }
        }
        buffer[i] = data;
        return true;
    }

    @Override
    public int getBufferSize() {
        return buffer.length;
    }

    @Override
    public void obtain(List<T> consumeList) {
        this.obtain(consumeList, 0, buffer.length);
    }

    void obtain(List<T> consumeList, int start, int end) {
        for (int i = start; i < end; i++) {
            if (buffer[i] != null) {
                consumeList.add((T) buffer[i]);
                buffer[i] = null;
            }
        }
    }

}

```

AtomicRangeInteger是队列的索引，代码如下：
```
public class AtomicRangeInteger extends Number implements Serializable {
    private static final long serialVersionUID = -4099792402691141643L;
    // 一个可以原子化操作数组某一个元素的数组封装
    private AtomicIntegerArray values;

    private static final int VALUE_OFFSET = 15;

    private int startValue;
    private int endValue;

    public AtomicRangeInteger(int startValue, int maxValue) {
        // 简单理解为,创建了一个长度为31的数组
        this.values = new AtomicIntegerArray(31);
        // 在values这个数组的下标为15(即第16个元素)的位置的值设置为指定值(默认为0)
        this.values.set(VALUE_OFFSET, startValue);
        this.startValue = startValue;
        this.endValue = maxValue - 1;
    }

    public final int getAndIncrement() {
        int next;
        do {
            next = this.values.incrementAndGet(VALUE_OFFSET);
            // 如果取到的next>endValue,就意味着下标越界了
            // 这时候需要通过CAS操作将values的第16个元素的值重置为startValue,即0
            if (next > endValue && this.values.compareAndSet(VALUE_OFFSET, next, startValue)) {
                return endValue;
            }
        }
        while (next > endValue);

        return next - 1;
    }

    public final int get() {
        return this.values.get(VALUE_OFFSET);
    }

    @Override
    public int intValue() {
        return this.values.get(VALUE_OFFSET);
    }

    @Override
    public long longValue() {
        return this.values.get(VALUE_OFFSET);
    }

    @Override
    public float floatValue() {
        return this.values.get(VALUE_OFFSET);
    }

    @Override
    public double doubleValue() {
        return this.values.get(VALUE_OFFSET);
    }
}

```
AtomicRangeInteger是使用JDK的AtomicIntegerArray实现的，AtomicRangeInteger初始化了一个长度为31的数组，使用数组最中间的元素（下标为15的元素）代表索引值，索引值初始值为0。getAndIncrement()方法中先对索引值+1，如果此时索引值>endValue就意味着下标越界了，这时候需要通过CAS操作将索引值重置为0，这样就实现了环形队列

AtomicRangeInteger为什么使用AtomicIntegerArray创建一个长度为31的数组？如果只是为了原子性操作完全可以使用AtomicInteger实现

SkyWalking之前也是使用AtomicInteger实现的，后面为了避免伪共享从而提高性能改为了AtomicIntegerArray

对应PR：https://github.com/apache/skywalking/pull/2930

伪共享相关文章：https://blog.csdn.net/qq_40378034/article/details/101383233

## ArrayBlockingQueueBuffer
ArrayBlockingQueueBuffer是使用JDK的ArrayBlockingQueue实现的，代码如下：
```
/**
 * The buffer implementation based on JDK ArrayBlockingQueue.
 * <p>
 * This implementation has better performance in server side. We are still trying to research whether this is suitable
 * for agent side, which is more sensitive about blocks.
 */
public class ArrayBlockingQueueBuffer<T> implements QueueBuffer<T> {
    private BufferStrategy strategy;
    private ArrayBlockingQueue<T> queue;
    private int bufferSize;

    ArrayBlockingQueueBuffer(int bufferSize, BufferStrategy strategy) {
        this.strategy = strategy;
        this.queue = new ArrayBlockingQueue<T>(bufferSize);
        this.bufferSize = bufferSize;
    }

    @Override
    public boolean save(T data) {
        //only BufferStrategy.BLOCKING
        try {
            queue.put(data);
        } catch (InterruptedException e) {
            // Ignore the error
            return false;
        }
        return true;
    }

    @Override
    public void setStrategy(BufferStrategy strategy) {
        this.strategy = strategy;
    }

    @Override
    public void obtain(List<T> consumeList) {
        queue.drainTo(consumeList);
    }

    @Override
    public int getBufferSize() {
        return bufferSize;
    }
}

```

## DataCarrier全解
Channels
Channels中管理了多个Buffer，代码如下：
```
/**
 * Channels of Buffer It contains all buffer data which belongs to this channel. It supports several strategy when
 * buffer is full. The Default is BLOCKING <p> Created by wusheng on 2016/10/25.
 */
public class Channels<T> {
    private final QueueBuffer<T>[] bufferChannels; // buffer数组
    private IDataPartitioner<T> dataPartitioner; // 数据分区器,确定每次操作哪个buffer
    private final BufferStrategy strategy;
    private final long size;

    public Channels(int channelSize, int bufferSize, IDataPartitioner<T> partitioner, BufferStrategy strategy) {
        this.dataPartitioner = partitioner;
        this.strategy = strategy;
        bufferChannels = new QueueBuffer[channelSize];
        for (int i = 0; i < channelSize; i++) {
            if (BufferStrategy.BLOCKING.equals(strategy)) {
                bufferChannels[i] = new ArrayBlockingQueueBuffer<>(bufferSize, strategy);
            } else {
                bufferChannels[i] = new Buffer<>(bufferSize, strategy);
            }
        }
        // noinspection PointlessArithmeticExpression
        size = 1L * channelSize * bufferSize; // it's not pointless, it prevents numeric overflow before assigning an integer to a long
    }

    public boolean save(T data) {
        // buffer的索引,即选择哪个buffer来存数据
        int index = dataPartitioner.partition(bufferChannels.length, data);
        int retryCountDown = 1;
        if (BufferStrategy.IF_POSSIBLE.equals(strategy)) {
            int maxRetryCount = dataPartitioner.maxRetryCount();
            if (maxRetryCount > 1) {
                retryCountDown = maxRetryCount;
            }
        }
        for (; retryCountDown > 0; retryCountDown--) {
            if (bufferChannels[index].save(data)) {
                return true;
            }
        }
        return false;
    }

```
一个Channels中包含多个Buffer

数据分区器IDataPartitioner接口代码如下：
```
public interface IDataPartitioner<T> {
    int partition(int total, T data);

    /**
     * @return an integer represents how many times should retry when {@link BufferStrategy#IF_POSSIBLE}.
     * <p>
     * Less or equal 1, means not support retry.
     */
    int maxRetryCount();
}
```

IDataPartitioner有两个实现SimpleRollingPartitioner和ProducerThreadPartitioner

SimpleRollingPartitioner分区是每次+1和total取模：
```
/**
 * use normal int to rolling.
 */
public class SimpleRollingPartitioner<T> implements IDataPartitioner<T> {
    @SuppressWarnings("NonAtomicVolatileUpdate")
    private volatile int i = 0;

    @Override
    public int partition(int total, T data) {
        return Math.abs(i++ % total);
    }

    @Override
    public int maxRetryCount() {
        return 3;
    }
}

```

ProducerThreadPartitioner分区是使用当前线程ID和total取模：
```
/**
 * use threadid % total to partition
 */
public class ProducerThreadPartitioner<T> implements IDataPartitioner<T> {
    public ProducerThreadPartitioner() {
    }

    @Override
    public int partition(int total, T data) {
        return (int) Thread.currentThread().getId() % total;
    }

    @Override
    public int maxRetryCount() {
        return 1;
    }
}

```

消费者

消费者读取DataCarrier中的数据上报到OAP，IConsumer是消费者的顶层接口：
```
public interface IConsumer<T> {
    void init();

    void consume(List<T> data);

    void onError(List<T> data, Throwable t);

    void onExit();

    /**
     * Notify the implementation, if there is nothing fetched from the queue. This could be used as a timer to trigger
     * reaction if the queue has no element.
     */
    default void nothingToConsume() {
        return;
    }
}

```

ConsumerThread代码如下：
```
public class ConsumerThread<T> extends Thread {
    private volatile boolean running;
    private IConsumer<T> consumer;
    private List<DataSource> dataSources;
    // 本次消费没有取到数据时,线程sleep的时间
    private long consumeCycle;

    ConsumerThread(String threadName, IConsumer<T> consumer, long consumeCycle) {
        super(threadName);
        this.consumer = consumer;
        running = false;
        dataSources = new ArrayList<DataSource>(1);
        this.consumeCycle = consumeCycle;
    }

    /**
     * add whole buffer to consume
     */
    void addDataSource(QueueBuffer<T> sourceBuffer) {
        this.dataSources.add(new DataSource(sourceBuffer));
    }

    @Override
    public void run() {
        running = true;

        final List<T> consumeList = new ArrayList<T>(1500);
        while (running) {
            if (!consume(consumeList)) {
                try {
                    // 没有消费到数据,线程sleep
                    Thread.sleep(consumeCycle);
                } catch (InterruptedException e) {
                }
            }
        }

        // consumer thread is going to stop
        // consume the last time
        consume(consumeList);

        consumer.onExit();
    }

    private boolean consume(List<T> consumeList) {
        for (DataSource dataSource : dataSources) {
            // 将buffer中的数据放到consumeList中,并清空buffer
            dataSource.obtain(consumeList);
        }

        if (!consumeList.isEmpty()) {
            try {
                // 调用消费者的消费逻辑
                consumer.consume(consumeList);
            } catch (Throwable t) {
                consumer.onError(consumeList, t);
            } finally {
                consumeList.clear();
            }
            return true;
        }
        consumer.nothingToConsume();
        return false;
    }

    void shutdown() {
        running = false;
    }

    /**
     * DataSource is a refer to {@link Buffer}.
     */
    class DataSource {
        private QueueBuffer<T> sourceBuffer;

        DataSource(QueueBuffer<T> sourceBuffer) {
            this.sourceBuffer = sourceBuffer;
        }

        void obtain(List<T> consumeList) {
            sourceBuffer.obtain(consumeList);
        }
    }
}

```

一个ConsumerThread中包含多个DataSource，DataSource里包装了Buffer。同时一个ConsumerThread绑定了一个Consumer，Consumer会消费ConsumerThread中的DataSource

MultipleChannelsConsumer代表一个单消费者线程，但支持多个Channels和它们的消费者，代码如下：
```
/**
 * MultipleChannelsConsumer代表一个单消费者线程,但支持多个channels和它们的消费者
 * MultipleChannelsConsumer represent a single consumer thread, but support multiple channels with their {@link
 * IConsumer}s
 */
public class MultipleChannelsConsumer extends Thread {
    private volatile boolean running;
    private volatile ArrayList<Group> consumeTargets;
    @SuppressWarnings("NonAtomicVolatileUpdate")
    private volatile long size;
    private final long consumeCycle;

    public MultipleChannelsConsumer(String threadName, long consumeCycle) {
        super(threadName);
        this.consumeTargets = new ArrayList<Group>();
        this.consumeCycle = consumeCycle;
    }

    @Override
    public void run() {
        running = true;

        final List consumeList = new ArrayList(2000);
        while (running) {
            boolean hasData = false;
            for (Group target : consumeTargets) {
                boolean consume = consume(target, consumeList);
                hasData = hasData || consume;
            }

            if (!hasData) {
                try {
                    Thread.sleep(consumeCycle);
                } catch (InterruptedException e) {
                }
            }
        }

        // consumer thread is going to stop
        // consume the last time
        for (Group target : consumeTargets) {
            consume(target, consumeList);

            target.consumer.onExit();
        }
    }

    private boolean consume(Group target, List consumeList) {
        // 遍历channels中的buffer,将buffer中的数据放到consumeList中,并清空buffer
        for (int i = 0; i < target.channels.getChannelSize(); i++) {
            QueueBuffer buffer = target.channels.getBuffer(i);
            buffer.obtain(consumeList);
        }

        if (!consumeList.isEmpty()) {
            try {
                // 调用消费者的消费逻辑
                target.consumer.consume(consumeList);
            } catch (Throwable t) {
                target.consumer.onError(consumeList, t);
            } finally {
                consumeList.clear();
            }
            return true;
        }
        target.consumer.nothingToConsume();
        return false;
    }

    /**
     * Add a new target channels.
     */
    public void addNewTarget(Channels channels, IConsumer consumer) {
        Group group = new Group(channels, consumer);
        // Recreate the new list to avoid change list while the list is used in consuming.
        ArrayList<Group> newList = new ArrayList<Group>();
        for (Group target : consumeTargets) {
            newList.add(target);
        }
        newList.add(group);
        consumeTargets = newList;
        size += channels.size();
    }

    public long size() {
        return size;
    }

    void shutdown() {
        running = false;
    }

    private static class Group {
        private Channels channels; // 一个channels对应多个buffer
        private IConsumer consumer; // consumer会消费channels中所有的buffer

        public Group(Channels channels, IConsumer consumer) {
            this.channels = channels;
            this.consumer = consumer;
        }
    }
}

```

一个Group中包含一个Consumer和一个Channels，一个Channels包含多个Buffer，Consumer会消费Channels中所有的Buffer

一个MultipleChannelsConsumer包含多个Group，实际上是管理多个Consumer以及它们对应的Buffer


## 消费者驱动
IDriver代码如下：
```
/**
 * The driver of consumer.
 */
public interface IDriver {
    boolean isRunning(Channels channels);

    void close(Channels channels);

    void begin(Channels channels);
}

```

ConsumeDriver代码如下：
```
/**
 * Pool of consumers <p> Created by wusheng on 2016/10/25.
 * 一堆消费者线程拿着一堆buffer,按allocateBuffer2Thread()的策略进行分配消费
 */
public class ConsumeDriver<T> implements IDriver {
    private boolean running;
    private ConsumerThread[] consumerThreads;
    private Channels<T> channels;
    private ReentrantLock lock;

    public ConsumeDriver(String name, Channels<T> channels, Class<? extends IConsumer<T>> consumerClass, int num,
        long consumeCycle) {
        this(channels, num);
        for (int i = 0; i < num; i++) {
            consumerThreads[i] = new ConsumerThread("DataCarrier." + name + ".Consumer." + i + ".Thread", getNewConsumerInstance(consumerClass), consumeCycle);
            consumerThreads[i].setDaemon(true);
        }
    }

    public ConsumeDriver(String name, Channels<T> channels, IConsumer<T> prototype, int num, long consumeCycle) {
        this(channels, num);
        prototype.init();
        for (int i = 0; i < num; i++) {
            consumerThreads[i] = new ConsumerThread("DataCarrier." + name + ".Consumer." + i + ".Thread", prototype, consumeCycle);
            consumerThreads[i].setDaemon(true);
        }

    }

    private ConsumeDriver(Channels<T> channels, int num) {
        running = false;
        this.channels = channels;
        consumerThreads = new ConsumerThread[num];
        lock = new ReentrantLock();
    }

    private IConsumer<T> getNewConsumerInstance(Class<? extends IConsumer<T>> consumerClass) {
        try {
            IConsumer<T> inst = consumerClass.getDeclaredConstructor().newInstance();
            inst.init();
            return inst;
        } catch (InstantiationException e) {
            throw new ConsumerCannotBeCreatedException(e);
        } catch (IllegalAccessException e) {
            throw new ConsumerCannotBeCreatedException(e);
        } catch (NoSuchMethodException e) {
            throw new ConsumerCannotBeCreatedException(e);
        } catch (InvocationTargetException e) {
            throw new ConsumerCannotBeCreatedException(e);
        }
    }

    @Override
    public void begin(Channels channels) {
        // begin只能调用一次
        if (running) {
            return;
        }
        lock.lock();
        try {
            this.allocateBuffer2Thread();
            for (ConsumerThread consumerThread : consumerThreads) {
                consumerThread.start();
            }
            running = true;
        } finally {
            lock.unlock();
        }
    }

    @Override
    public boolean isRunning(Channels channels) {
        return running;
    }

    private void allocateBuffer2Thread() {
        // buffer的数量
        int channelSize = this.channels.getChannelSize();
        /**
         * 因为channels里面有多个buffer,同时这里也有多个消费者线程
         * 这一步的操作就是将这些buffer分配给不同的消费者线程去消费
         *
         * if consumerThreads.length < channelSize
         * each consumer will process several channels.
         *
         * if consumerThreads.length == channelSize
         * each consumer will process one channel.
         *
         * if consumerThreads.length > channelSize
         * there will be some threads do nothing.
         */
        for (int channelIndex = 0; channelIndex < channelSize; channelIndex++) {
            // 消费者线程索引 = buffer的下标和消费者线程数取模
            int consumerIndex = channelIndex % consumerThreads.length;
            consumerThreads[consumerIndex].addDataSource(channels.getBuffer(channelIndex));
        }

    }

    @Override
    public void close(Channels channels) {
        lock.lock();
        try {
            this.running = false;
            for (ConsumerThread consumerThread : consumerThreads) {
                consumerThread.shutdown();
            }
        } finally {
            lock.unlock();
        }
    }
}

```

一个ConsumeDriver包含多个ConsumerThread
```
/**
 * BulkConsumePool works for consuming data from multiple channels(DataCarrier instances), with multiple {@link
 * MultipleChannelsConsumer}s.
 * <p>
 * In typical case, the number of {@link MultipleChannelsConsumer} should be less than the number of channels.
 */
public class BulkConsumePool implements ConsumerPool {
    private List<MultipleChannelsConsumer> allConsumers;
    private volatile boolean isStarted = false;

    public BulkConsumePool(String name, int size, long consumeCycle) {
        size = EnvUtil.getInt(name + "_THREAD", size);
        allConsumers = new ArrayList<MultipleChannelsConsumer>(size);
        for (int i = 0; i < size; i++) {
            MultipleChannelsConsumer multipleChannelsConsumer = new MultipleChannelsConsumer("DataCarrier." + name + ".BulkConsumePool." + i + ".Thread", consumeCycle);
            multipleChannelsConsumer.setDaemon(true);
            allConsumers.add(multipleChannelsConsumer);
        }
    }

    @Override
    synchronized public void add(String name, Channels channels, IConsumer consumer) {
        MultipleChannelsConsumer multipleChannelsConsumer = getLowestPayload();
        multipleChannelsConsumer.addNewTarget(channels, consumer);
    }

    /**
     * 拿到负载最低的消费者线程
     * Get the lowest payload consumer thread based on current allocate status.
     *
     * @return the lowest consumer.
     */
    private MultipleChannelsConsumer getLowestPayload() {
        MultipleChannelsConsumer winner = allConsumers.get(0);
        for (int i = 1; i < allConsumers.size(); i++) {
            MultipleChannelsConsumer option = allConsumers.get(i);
            // 比较consumer的size(consumer中buffer的总数)
            if (option.size() < winner.size()) {
                winner = option;
            }
        }
        return winner;
    }

    /**
     *
     */
    @Override
    public boolean isRunning(Channels channels) {
        return isStarted;
    }

    @Override
    public void close(Channels channels) {
        for (MultipleChannelsConsumer consumer : allConsumers) {
            consumer.shutdown();
        }
    }

    @Override
    public void begin(Channels channels) {
        if (isStarted) {
            return;
        }
        for (MultipleChannelsConsumer consumer : allConsumers) {
            consumer.start();
        }
        isStarted = true;
    }

    /**
     * The creator for {@link BulkConsumePool}.
     */
    public static class Creator implements Callable<ConsumerPool> {
        private String name;
        private int size;
        private long consumeCycle;

        public Creator(String name, int poolSize, long consumeCycle) {
            this.name = name;
            this.size = poolSize;
            this.consumeCycle = consumeCycle;
        }

        @Override
        public ConsumerPool call() {
            return new BulkConsumePool(name, size, consumeCycle);
        }

        public static int recommendMaxSize() {
            return Runtime.getRuntime().availableProcessors() * 2;
        }
    }
}

```
一个BulkConsumePool包含多个MultipleChannelsConsumer


