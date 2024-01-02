---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码线程池

## JDK线程池
线程池解决的核心问题就是资源管理问题。线程池的创建方式总共包含以下 7 种（其中 6 种是通过 Executors 创建的，1 种是通过 ThreadPoolExecutor 创建的）：

Executors.newFixedThreadPool：创建一个固定大小的线程池，可控制并发的线程数，超出的线程会在队列中等待；
Executors.newCachedThreadPool：创建一个可缓存的线程池，若线程数超过处理所需，缓存一段时间后会回收，若线程数不够，则新建线程；
Executors.newSingleThreadExecutor：创建单个线程数的线程池，它可以保证先进先出的执行顺序；
Executors.newScheduledThreadPool：创建一个可以执行延迟任务的线程池；
Executors.newSingleThreadScheduledExecutor：创建一个单线程的可以执行延迟任务的线程池；
Executors.newWorkStealingPool：创建一个抢占式执行的线程池（任务执行顺序不确定）；
ThreadPoolExecutor：最原始的创建线程池的方式，它包含了 7 个参数可供设置；
JDK线程池初ThreadPoolExecutor始化代码如下：
```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
任务调度是线程池的主要入口，当用户提交了一个任务，接下来这个任务将如何执行都是由这个阶段决定的。所有任务的调度都是由execute方法完成的，这部分完成的工作是：检查现在线程池的运行状态、运行线程数、运行策略，决定接下来执行的流程，是直接申请线程执行，或是缓冲到队列中执行，亦或是直接拒绝该任务。其执行过程如下：

首先检测线程池运行状态，如果不是RUNNING，则直接拒绝，线程池要保证在RUNNING的状态下执行任务。
如果workerCount < corePoolSize，则创建并启动一个线程来执行新提交的任务。
如果workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中。
如果workerCount >= corePoolSize && workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务。
如果workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。

这里存在问题，阻塞队列如果是无界队列，那么最大线程数是没有用的。

## 线程池模块初始化
Node节点启动，初始化ThreadPool类创建线程池。
```
final ThreadPool threadPool = new ThreadPool(settings, executorBuilders.toArray(new ExecutorBuilder[0]));
```
ES根据不同的业务，在初始化线程池时设定不同的参数，创建不同的线程池类型的优势：

1.不同业务的线程池不一样，隔离了业务，避免业务之间的线程池相互影响；

2.提高线程的利用率。例如CPU密集型业务、IO密集型业务设置不同参数；

ThreadPool初始化的三个主要工作：

1.ES系统内部自定义线程池和用户自定义线程池参数进行合并；

2.全部线程池进行初始化，并保存每个线程池对应参数信息；

3.初始化任务调度；

```
public ThreadPool(final Settings settings, final ExecutorBuilder<?>... customBuilders) {
    assert Node.NODE_NAME_SETTING.exists(settings);

    final Map<String, ExecutorBuilder> builders = new HashMap<>();
    // 获取本机cpu核数，假设cpu核数为8
    final int availableProcessors = EsExecutors.numberOfProcessors(settings);
    // (cpu+1)/2 在区间 [1,5] 中的取值，此处(8+1)/2 = 4, 在[1,5]区间内取值为4
    final int halfProcMaxAt5 = halfNumberOfProcessorsMaxFive(availableProcessors);
    // (cpu+1)/2 在区间 [1,10] 中的取值，此处(8+1)/2 = 4, 在[1,5]区间内取值为4
    final int halfProcMaxAt10 = halfNumberOfProcessorsMaxTen(availableProcessors);
    // 4*8=32，genericThreadPoolMax在区间[128,512]中的取值为128
    final int genericThreadPoolMax = boundedBy(4 * availableProcessors, 128, 512);

    //各种线程池
    //generic：用于通用的请求（例如：后台节点发现），线程池类型为 scaling
    builders.put(Names.GENERIC, new ScalingExecutorBuilder(Names.GENERIC, 4, genericThreadPoolMax, TimeValue.timeValueSeconds(30)));
    //write：用于单个文档的 index/delete/update 请求以及bulk请求，线程池类型为fixed，大小的为处理器数量，队列大小为200，最大线程数为1 + 处理器数量。
    builders.put(Names.WRITE, new FixedExecutorBuilder(settings, Names.WRITE, availableProcessors, 200));
    //get：用于get请求。线程池类型为 fixed，大小的为处理器数量，队列大小为1000。
    builders.put(Names.GET, new FixedExecutorBuilder(settings, Names.GET, availableProcessors, 1000));
    //analyze：用于analyze请求。线程池类型为 fixed，大小的1，队列大小为16
    builders.put(Names.ANALYZE, new FixedExecutorBuilder(settings, Names.ANALYZE, 1, 16));
    //search：用于count/search/suggest请求。线程池类型为 fixed， 大小的为 int((处理器数量 3) / 2) +1，队列大小为1000。
    builders.put(Names.SEARCH, new AutoQueueAdjustingExecutorBuilder(settings,
                    Names.SEARCH, searchThreadPoolSize(availableProcessors), 1000, 1000, 1000, 2000));
    builders.put(Names.SEARCH_THROTTLED, new AutoQueueAdjustingExecutorBuilder(settings,
        Names.SEARCH_THROTTLED, 1, 100, 100, 100, 200));
    builders.put(Names.MANAGEMENT, new ScalingExecutorBuilder(Names.MANAGEMENT, 1, 5, TimeValue.timeValueMinutes(5)));
    // no queue as this means clients will need to handle rejections on listener queue even if the operation succeeded
    // the assumption here is that the listeners should be very lightweight on the listeners side
    //listener：主要用于Java客户端线程监听器被设置为true时执行动作。线程池类型为 scaling，最大线程数为min(10, (处理器数量)/2)
    builders.put(Names.LISTENER, new FixedExecutorBuilder(settings, Names.LISTENER, halfProcMaxAt10, -1));
    builders.put(Names.FLUSH, new ScalingExecutorBuilder(Names.FLUSH, 1, halfProcMaxAt5, TimeValue.timeValueMinutes(5)));
    //refresh：用于refresh请求。线程池类型为 scaling，线程空闲保持存活时间为5分钟，最大线程数为min(10, (处理器数量)/2)
    builders.put(Names.REFRESH, new ScalingExecutorBuilder(Names.REFRESH, 1, halfProcMaxAt10, TimeValue.timeValueMinutes(5)));
    //warmer：用于segment warm-up请求。线程池类型为 scaling，线程保持存活时间为5分钟，最大线程数为min(5, (处理器数量)/2)
    builders.put(Names.WARMER, new ScalingExecutorBuilder(Names.WARMER, 1, halfProcMaxAt5, TimeValue.timeValueMinutes(5)));
    //snapshot：用于snaphost/restore请求。线程池类型为 scaling，线程保持存活时间为5分钟，最大线程数为min(5, (处理器数量)/2)
    builders.put(Names.SNAPSHOT, new ScalingExecutorBuilder(Names.SNAPSHOT, 1, halfProcMaxAt5, TimeValue.timeValueMinutes(5)));
    builders.put(Names.FETCH_SHARD_STARTED,
            new ScalingExecutorBuilder(Names.FETCH_SHARD_STARTED, 1, 2 * availableProcessors, TimeValue.timeValueMinutes(5)));
    builders.put(Names.FORCE_MERGE, new FixedExecutorBuilder(settings, Names.FORCE_MERGE, 1, -1));
    builders.put(Names.FETCH_SHARD_STORE,
            new ScalingExecutorBuilder(Names.FETCH_SHARD_STORE, 1, 2 * availableProcessors, TimeValue.timeValueMinutes(5)));
    //builders包含ES系统内部定义线程池，customBuilders用户自定义线程池，这里将两者合并。
    for (final ExecutorBuilder<?> builder : customBuilders) {
        if (builders.containsKey(builder.name())) {
            throw new IllegalArgumentException("builder with name [" + builder.name() + "] already exists");
        }
        builders.put(builder.name(), builder);
    }
    this.builders = Collections.unmodifiableMap(builders);
    //线程上下文
    threadContext = new ThreadContext(settings);

    final Map<String, ExecutorHolder> executors = new HashMap<>();
    //真正创建线程池的代码，是在ThreadPool的构造方法中的for循环
    for (final Map.Entry<String, ExecutorBuilder> entry : builders.entrySet()) {
        final ExecutorBuilder.ExecutorSettings executorSettings = entry.getValue().getSettings(settings);
        final ExecutorHolder executorHolder = entry.getValue().build(executorSettings, threadContext);
        if (executors.containsKey(executorHolder.info.getName())) {
            throw new IllegalStateException("duplicate executors with name [" + executorHolder.info.getName() + "] registered");
        }
        logger.debug("created thread pool: {}", entry.getValue().formatInfo(executorHolder.info));
        executors.put(entry.getKey(), executorHolder);
    }

    executors.put(Names.SAME, new ExecutorHolder(DIRECT_EXECUTOR, new Info(Names.SAME, ThreadPoolType.DIRECT)));
    //构建出来的线程池被封装在ThreadPool.ExecutorHolder
    this.executors = unmodifiableMap(executors);

    //Info类主要是线程池名称、类型、队列大小、线程数量的max和min、keepAlive时间
    final List<Info> infos =
            executors
                    .values()
                    .stream()
                    .filter(holder -> holder.info.getName().equals("same") == false)
                    .map(holder -> holder.info)
                    .collect(Collectors.toList());
    //包含每一个线程池的信息（主要是线程池名称、类型、队列大小、线程数量的max和min、keepAlive时间）
    this.threadPoolInfo = new ThreadPoolInfo(infos);
    //Java任务调度
    this.scheduler = Scheduler.initScheduler(settings);
    TimeValue estimatedTimeInterval = ESTIMATED_TIME_INTERVAL_SETTING.get(settings);
    //缓存系统时间
    this.cachedTimeThread = new CachedTimeThread(EsExecutors.threadName(settings, "[timer]"), estimatedTimeInterval.millis());
    this.cachedTimeThread.start();
}
```

## ElasticSearch线程池
### 重要的线程池
ES内线程使用很多线程，Thread Pool官方及上一小节都列举出一些重要的线程池及作用如下：
```
//线程池的名称在内部类Names中，最好记住它们的名字，
//有时需要通过jstack查看堆栈，ES的堆栈非常长，这就需要通过线程池的名称去查找关注的内容
public static class Names {
    public static final String SAME = "same";
    //For generic operations (for example, background node discovery). Thread pool type is scaling
    public static final String GENERIC = "generic";
    //Mainly for java client executing of action when listener threaded is set to true.
    // Thread pool type is scaling with a default max of min(10, (# of available processors)/2).
    public static final String LISTENER = "listener";
    //For get operations. Thread pool type is fixed with a size of # of available processors, queue_size of 1000.
    public static final String GET = "get";
    //For analyze requests. Thread pool type is fixed with a size of 1, queue size of 16.
    public static final String ANALYZE = "analyze";
    //For single-document index/delete/update and bulk requests.
    // Thread pool type is fixed with a size of # of available processors, queue_size of 200.
    // The maximum size for this pool is 1 + # of available processors
    public static final String WRITE = "write";
    //For count/search/suggest operations.
    //Thread pool type is fixed_auto_queue_size with a size of int((# of available_processors * 3) / 2) + 1, and initial queue_size of 1000
    public static final String SEARCH = "search";
    //For count/search/suggest/get operations on search_throttled indices.
    // Thread pool type is fixed_auto_queue_size with a size of 1, and initial queue_size of 100.
    public static final String SEARCH_THROTTLED = "search_throttled";
    //For cluster management.
    // Thread pool type is scaling with a keep-alive of 5m and a default maximum size of 5.
    public static final String MANAGEMENT = "management";
    //For flush, synced flush, and translog fsync operations.
    //Thread pool type is scaling with a keep-alive of 5m and a default maximum size of min(5, (# of available processors)/2).
    public static final String FLUSH = "flush";
    //For refresh operations.
    // Thread pool type is scaling with a keep-alive of 5m and a max of min(10, (# of available processors)/2).
    public static final String REFRESH = "refresh";
    //For segment warm-up operations.
    // Thread pool type is scaling with a keep-alive of 5m and a max of min(5, (# of available processors)/2).
    public static final String WARMER = "warmer";
    //For snapshot/restore operations.
    // Thread pool type is scaling with a keep-alive of 5m and a max of min(5, (# of available processors)/2).
    public static final String SNAPSHOT = "snapshot";
    //For force merge operations.
    // Thread pool type is fixed with a size of 1 and an unbounded queue size
    public static final String FORCE_MERGE = "force_merge";
    //For listing shard states.
    // Thread pool type is scaling with keep-alive of 5m and a default maximum size of 2 * # of available processors.
    public static final String FETCH_SHARD_STARTED = "fetch_shard_started";
    //For listing shard stores.
    // Thread pool type is scaling with keep-alive of 5m and a default maximum size of 2 * # of available processors.
    public static final String FETCH_SHARD_STORE = "fetch_shard_store";
}
```

EsThreadPoolExecutor
线程池的创建在EsExecutors类调用EsThreadPoolExecutor进行初始化，EsThreadPoolExecutor类初始化代码如下。
```
EsThreadPoolExecutor(String name, int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit,
        BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, XRejectedExecutionHandler handler,
        ThreadContext contextHolder) {
    //corePoolSize：核心线程池大小;
    //maximumPoolSize：最大线程数量;
    //keepAliveTime：线程空闲回收时间;
    //BlockingQueue：任务队列;
    //handler： 队列满，拒绝请求时的回调函数
    super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
    this.name = name;
    this.contextHolder = contextHolder;
}
```
EsThreadPoolExecutor类继承了JDK的ThreadPoolExecutor类，并做了扩展，处理和ThreadPoolExecutor初始化需要的相同参数外，还有name和ThreadContext属性。name对应的是线程池名称，ThreadContext是线程上下文。EsThreadPoolExecutor类的关系图如下：

### 线程池类型
ES线程池模块，创建不同类型的线程池，本质上是对核心线程个数、最大线程个数、任务队列、拒绝策略设置不通的自定义参数。

newSinglePrioritizing
固定单线程的线程池，core pool size == max pool size ==1，说明该线程池里面只有一个工作线程。
```
public static PrioritizedEsThreadPoolExecutor newSinglePrioritizing(String name, ThreadFactory threadFactory,
                                                                    ThreadContext contextHolder, ScheduledExecutorService timer) {
    //core pool size == max pool size ==1，说明该线程池里面只有一个工作线程
    return new PrioritizedEsThreadPoolExecutor(name, 1, 1, 0L, TimeUnit.MILLISECONDS, threadFactory, contextHolder, timer);
}
```
队列是阻塞优先队列PriorityBlockingQueue，能够区分优先级，如果两个任务优先级相同则按照先进先出（FIFO）方式执行。

### fixed thread pool
线程数固定的线程池。corePoolSize和maximumPoolSize值相同，线程池正常执行时核心线程数和最大线程数固定的。
```
public static EsThreadPoolExecutor newFixed(String name, int size, int queueCapacity,
                                            ThreadFactory threadFactory, ThreadContext contextHolder) {
    BlockingQueue<Runnable> queue;
    //newFixed线程池队列分两种情况：
    if (queueCapacity < 0) {
        //By default, it is set to -1 which means its unbounded
        //当queueCapacity小于0时，队列是LinkedTransferQueue，为无界队列。
        queue = ConcurrentCollections.newBlockingQueue();
    } else {
        //当queueCapacity大于0时，队列是SizeBlockingQueue
        queue = new SizeBlockingQueue<>(ConcurrentCollections.<Runnable>newBlockingQueue(), queueCapacity);
    }
    //创建线程池
    return new EsThreadPoolExecutor(name, size, size, 0, TimeUnit.MILLISECONDS,
        queue, threadFactory, new EsAbortPolicy(), contextHolder);
}
```
队列是：SizeBlockingQueue，队列任务数不能超过队列初始化大小，否则拒绝写入队列。
```
@Override
public boolean offer(E e) {
    while (true) {
        final int current = size.get();
        // 当队列容量超过限制时，拒绝写入队列
        if (current >= capacity()) {
            return false;
        }
        if (size.compareAndSet(current, 1 + current)) {
            break;
        }
    }
    boolean offered = queue.offer(e);
    if (!offered) {
        size.decrementAndGet();
    }
    return offered;
}
```
拒绝策略：EsAbortPolicy。EsAbortPolicy和SizeBlockingQueue是相对应的，如果任务的isForceExecution方法返回true时，强行写入队列。
```
@Override
public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
    if (r instanceof AbstractRunnable) {
        //判断任务是否要强制执行
        if (((AbstractRunnable) r).isForceExecution()) {
            BlockingQueue<Runnable> queue = executor.getQueue();
            //创建ThreadPoolExecutor指定的 任务队列 类型是SizeBlockingQueue
            if (!(queue instanceof SizeBlockingQueue)) {
                throw new IllegalStateException("forced execution, but expected a size queue");
            }
            try {
                // 当任务的isForceExecution方法返回true时，强行写入队列
                //尽管任务执行失败了，还是再一次把它提交到任务队列，这样拒绝的任务又可以有执行机会了
                ((SizeBlockingQueue) queue).forcePut(r);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new IllegalStateException("forced execution, but got interrupted", e);
            }
            return;
        }
    }
    rejected.inc();
    throw new EsRejectedExecutionException("rejected execution of " + r + " on " + executor, executor.isShutdown());
}
```

### scaling thread pool

核心线程数、最大线程数、线程存活时间keep_alive是可以动态调整的。由于newScaling线程池的任务类型包含FLUSH，REFRESH这些IO密集型任务，这样处理的原因比较显而易见了————提前增大线程数量可以获得更高的吞吐量。
```
public static EsThreadPoolExecutor newScaling(String name, int min, int max, long keepAliveTime, TimeUnit unit,
                                              ThreadFactory threadFactory, ThreadContext contextHolder) {
    //ExecutorScalingQueue实际上是一个封装了的LinkedTransferQueue，为其加上了容量
    ExecutorScalingQueue<Runnable> queue = new ExecutorScalingQueue<>();
    EsThreadPoolExecutor executor =
        new EsThreadPoolExecutor(name, min, max, keepAliveTime, unit, queue, threadFactory, new ForceQueuePolicy(), contextHolder);
    queue.executor = executor;
    return executor;
}
```
队列是：ExecutorScalingQueue。第一小节 JDK线程池简单描述了JDK默认逻辑，只有当任务队列超过最大值且核心线程数小于最大线程数时，才会增加线程。这里重写方法，没有达到最大线程数，先创建线程，达到最大线程后再写入任务队列。

```
//JDK默认线程池的逻辑是：如果队列写入失败，并且当前线程数小于最大线程数时，才会增加线程并执行传入的任务。
// 重写这里的offer方法之后实际上将写入队列和增加线程的顺序颠倒过来了，传入一个新任务时，如果线程没有到最大线程，
// 直接增加线程，直到线程达到最大线程，任务再写入队列。
@Override
public boolean offer(E e) {
    // first try to transfer to a waiting worker thread
    if (!tryTransfer(e)) {
        // check if there might be spare capacity in the thread
        // pool executor
        int left = executor.getMaximumPoolSize() - executor.getCorePoolSize();
        if (left > 0) {
            // reject queuing the task to force the thread pool
            // executor to add a worker if it can; combined
            // with ForceQueuePolicy, this causes the thread
            // pool to always scale up to max pool size and we
            // only queue when there is no spare capacity
            return false;
        } else {
            return super.offer(e);
        }
    } else {
        return true;
    }
}
```
拒绝策略：ForceQueuePolicy。当达到最大线程数且ExecutorScalingQueue队列也达到最大，仍有新的任务时，等待空间可用时再插入队列。
```
/**会等待空间可用时再插入队列
 * A handler for rejected tasks that adds the specified element to this queue,
 * waiting if necessary for space to become available.
 */
static class ForceQueuePolicy implements XRejectedExecutionHandler {

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        try {
            // force queue policy should only be used with a scaling queue
            assert executor.getQueue() instanceof ExecutorScalingQueue;
            executor.getQueue().put(r);
        } catch (final InterruptedException e) {
            // a scaling queue never blocks so a put to it can never be interrupted
            throw new AssertionError(e);
        }
    }
```

### fixed_auto_queue_size thread pool

7.7版本之前，专门给SEARCH任务使用的线程池，根据Little's Law实现的动态调整队列大小。
```
public static EsThreadPoolExecutor newAutoQueueFixed(String name, int size, int initialQueueCapacity, int minQueueSize,
                                                     int maxQueueSize, int frameSize, TimeValue targetedResponseTime,
                                                     ThreadFactory threadFactory, ThreadContext contextHolder) {
    //初始化大小必须大于等于1
    if (initialQueueCapacity <= 0) {
        throw new IllegalArgumentException("initial queue capacity for [" + name + "] executor must be positive, got: " +
                        initialQueueCapacity);
    }
    //初始化一个动态调整的队列
    ResizableBlockingQueue<Runnable> queue =
            new ResizableBlockingQueue<>(ConcurrentCollections.<Runnable>newBlockingQueue(), initialQueueCapacity);
    //创建线程池
    return new QueueResizingEsThreadPoolExecutor(name, size, size, 0, TimeUnit.MILLISECONDS,
            queue, minQueueSize, maxQueueSize, TimedRunnable::new, frameSize, targetedResponseTime, threadFactory,
            new EsAbortPolicy(), contextHolder);
}
```
队列是：ResizableBlockingQueue。SizeBlockingQueue的扩展，队列大小能够动态调整。
```
/** Resize the limit for the queue, returning the new size limit */
public synchronized int adjustCapacity(int optimalCapacity, int adjustmentAmount, int minCapacity, int maxCapacity) {
    if (optimalCapacity == capacity) {
        // Yahtzee!
        return this.capacity;
    }

    if (optimalCapacity > capacity + adjustmentAmount) {
        // adjust up
        final int newCapacity = Math.min(maxCapacity, capacity + adjustmentAmount);
        this.capacity = newCapacity;
        return newCapacity;
    } else if (optimalCapacity < capacity - adjustmentAmount) {
        // adjust down
        final int newCapacity = Math.max(minCapacity, capacity - adjustmentAmount);
        this.capacity = newCapacity;
        return newCapacity;
    } else {
        return this.capacity;
    }
}
```
拒绝策略：EsAbortPolicy。EsAbortPolicy和SizeBlockingQueue是相对应的，如果任务的isForceExecution方法返回true时，强行写入队列。
```
@Override
public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
    if (r instanceof AbstractRunnable) {
        //判断任务是否要强制执行
        if (((AbstractRunnable) r).isForceExecution()) {
            BlockingQueue<Runnable> queue = executor.getQueue();
            //创建ThreadPoolExecutor指定的 任务队列 类型是SizeBlockingQueue
            if (!(queue instanceof SizeBlockingQueue)) {
                throw new IllegalStateException("forced execution, but expected a size queue");
            }
            try {
                // 当任务的isForceExecution方法返回true时，强行写入队列
                //尽管任务执行失败了，还是再一次把它提交到任务队列，这样拒绝的任务又可以有执行机会了
                ((SizeBlockingQueue) queue).forcePut(r);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new IllegalStateException("forced execution, but got interrupted", e);
            }
            return;
        }
    }
    rejected.inc();
    throw new EsRejectedExecutionException("rejected execution of " + r + " on " + executor, executor.isShutdown());
}
```