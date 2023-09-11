---
layout: post
categories: [JUC]
description: none
keywords: JUC
---
# 并发源码源码线程池ThreadPoolExecutor

## ThreadPoolExecutor简介
Executors其实是个工具类，里面提供了好多静态方法，这些方法根据用户选择返回不同的线程池实例。 ThreadPoolExecutor继承了AbstractExecutorService，成员变量ctl是一个Integer的原子变量，用来记录线程池状态和线程池中线程个数，类似于ReentrantReadWriteLock使用一个变量来保存两种信息。

```
public class ThreadPoolExecutor extends AbstractExecutorService {
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
    
    }
```
这里假设Integer类型是32位二进制表示，则其中高3位用来表示线程池状态，后面29位用来记录线程池线程个数。
```
//用来标记线程池状态（高3位），线程个数（低29位）
//默认是RUNNING状态，线程个数为0
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

//线程个数掩码位数，并不是所有平台int类型是32位，所以准确说是具体平台下Integer的二进制位数-3后的剩余位数才是线程的个数，
private static final int COUNT_BITS = Integer.SIZE - 3;

//线程最大个数(低29位)00011111111111111111111111111111
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
```

线程池状态：
```
//（高3位）：11100000000000000000000000000000
private static final int RUNNING    = -1 << COUNT_BITS;

//（高3位）：00000000000000000000000000000000
private static final int SHUTDOWN   =  0 << COUNT_BITS;

//（高3位）：00100000000000000000000000000000
private static final int STOP       =  1 << COUNT_BITS;

//（高3位）：01000000000000000000000000000000
private static final int TIDYING    =  2 << COUNT_BITS;

//（高3位）：01100000000000000000000000000000
private static final int TERMINATED =  3 << COUNT_BITS;
// 获取高三位 运行状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }

//获取低29位 线程个数
private static int workerCountOf(int c)  { return c & CAPACITY; }

//计算ctl新值，线程状态 与 线程个数
private static int ctlOf(int rs, int wc) { return rs | wc; }
```
线程池状态含义：
- RUNNING：接受新任务并且处理阻塞队列里的任务；
- SHUTDOWN：拒绝新任务但是处理阻塞队列里的任务；
- STOP：拒绝新任务并且抛弃阻塞队列里的任务，同时会中断正在处理的任务；
- TIDYING：所有任务都执行完（包含阻塞队列里面任务）当前线程池活动线程为 0，将要调用 terminated 方法；
- TERMINATED：终止状态，terminated方法调用完成以后的状态。

线程池状态转换：
- RUNNING -> SHUTDOWN：显式调用 shutdown() 方法，或者隐式调用了 finalize()，它里面调用了 shutdown() 方法。
- RUNNING or SHUTDOWN -> STOP：显式调用 shutdownNow() 方法时候。
- SHUTDOWN -> TIDYING：当线程池和任务队列都为空的时候。
- STOP -> TIDYING：当线程池为空的时候。
- TIDYING -> TERMINATED：当 terminated() hook 方法执行完成时候。

线程池参数：
- corePoolSize：线程池核心线程个数；
- workQueue：用于保存等待执行的任务的阻塞队列；比如基于数组的有界 ArrayBlockingQueue，基于链表的无界 LinkedBlockingQueue，最多只有一个元素的同步队列 SynchronousQueue，优先级队列 PriorityBlockingQueue 等。
- maximunPoolSize：线程池最大线程数量。
- ThreadFactory：创建线程的工厂。
- RejectedExecutionHandler：饱和策略，当队列满了并且线程个数达到 maximunPoolSize 后采取的策略，比如 AbortPolicy （抛出异常），CallerRunsPolicy（使用调用者所在线程来运行任务），DiscardOldestPolicy（调用 poll 丢弃一个任务，执行当前任务），DiscardPolicy（默默丢弃，不抛出异常）。
- keeyAliveTime：存活时间。如果当前线程池中的线程数量比核心线程数量要多，并且是闲置状态的话，这些闲置的线程能存活的最大时间。
- TimeUnit，存活时间的时间单位。

线程池类型：
- newFixedThreadPool
  创建一个核心线程个数和最大线程个数都为 nThreads 的线程池，并且阻塞队列长度为 Integer.MAX_VALUE，keeyAliveTime=0 说明只要线程个数比核心线程个数多并且当前空闲则回收。代码如下：
```
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
 }
 //使用自定义线程创建工厂
 public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
 }
```

- newSingleThreadExecutor
  创建一个核心线程个数和最大线程个数都为1的线程池，并且阻塞队列长度为 Integer.MAX_VALUE，keeyAliveTime=0 说明只要线程个数比核心线程个数多并且当前空闲则回收。代码如下：
```
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }

    //使用自己的线程工厂
    public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));
    }
```

- newCachedThreadPool
  创建一个按需创建线程的线程池，初始线程个数为 0，最多线程个数为 Integer.MAX_VALUE，并且阻塞队列为同步队列，keeyAliveTime=60 说明只要当前线程 60s 内空闲则回收。这个特殊在于加入到同步队列的任务会被马上被执行，同步队列里面最多只有一个任务。代码如下：
```
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

    //使用自定义的线程工厂
    public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }
```
其中 mainLock 是独占锁，用来控制新增 Worker 线程时候的原子性，termination 是该锁对应的条件队列，在线程调用 awaitTermination 时候用来存放阻塞的线程。

Worker 继承 AQS 和 Runnable 接口，是具体承载任务的对象，Worker 继承了 AQS，自己实现了简单不可重入独占锁，其中 status=0 标示锁未被获取状态，state=1 标示锁已经被获取的状态，state=-1 是创建 Worker 时候默认的状态，创建时候状态设置为 -1 是为了避免在该线程在运行 runWorker() 方法前被中断，下面会具体讲解到。其中变量 firstTask 记录该工作线程执行的第一个任务，thread 是具体执行任务的线程。

DefaultThreadFactory 是线程工厂，newThread 方法是对线程的一个修饰，其中 poolNumber 是个静态的原子变量，用来统计线程工厂的个数，threadNumber 用来记录每个线程工厂创建了多少线程，这两个值也作为线程池和线程的名称的一部分。

## 源码分析
- corePoolSize
  the number of threads to keep in the pool, even if they are idle, unless {@code allowCoreThreadTimeOut} is set
  (核心线程数大小：不管它们创建以后是不是空闲的。线程池需要保持 corePoolSize 数量的线程，除非设置了 allowCoreThreadTimeOut。)
- maximumPoolSize
  the maximum number of threads to allow in the pool。
  (最大线程数：线程池中最多允许创建 maximumPoolSize 个线程。)
- keepAliveTime
  when the number of threads is greater than the core, this is the maximum time that excess idle threads will wait for new tasks before terminating。
  (存活时间：如果经过 keepAliveTime 时间后，超过核心线程数的线程还没有接受到新的任务，那就回收。)
- unit
  the time unit for the {@code keepAliveTime} argument
  (keepAliveTime 的时间单位。)
- workQueue
  the queue to use for holding tasks before they are executed.  This queue will hold only the {@code Runnable} tasks submitted by the {@code execute} method。
  (存放待执行任务的队列：当提交的任务数超过核心线程数大小后，再提交的任务就存放在这里。它仅仅用来存放被 execute 方法提交的 Runnable 任务。所以这里就不要翻译为工作队列了，好吗？不要自己给自己挖坑。)
- threadFactory
  the factory to use when the executor creates a new thread。
  (线程工程：用来创建线程工厂。比如这里面可以自定义线程名称，当进行虚拟机栈分析时，看着名字就知道这个线程是哪里来的，不会懵逼。)
- handler
  the handler to use when execution is blocked because the thread bounds and queue capacities are reached。
  (拒绝策略：当队列里面放满了任务、最大线程数的线程都在工作时，这时继续提交的任务线程池就处理不了，应该执行怎么样的拒绝策略。)

### public void execute(Runnable command)
execute方法的作用是提交任务command到线程池进行执行。ThreadPoolExecutor的实现实际是一个生产消费模型，当用户添加任务到线程池时相当于生产者生产元素，workers线程工作集中的线程直接执行任务或者从任务队列里面获取任务时则相当于消费者消费元素。

用户线程提交任务的execute方法的具体代码如下。
```
public void execute(Runnable command) {

    //(1) 如果任务为null，则抛出NPE异常
    if (command == null)
        throw new NullPointerException();

    //（2）获取当前线程池的状态+线程个数变量的组合值
    int c = ctl.get();

    //（3）当前线程池线程个数是否小于corePoolSize,小于则开启新线程运行
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    //（4）如果线程池处于RUNNING状态，则添加任务到阻塞队列
    if (isRunning(c) && workQueue.offer(command)) {

        //（4.1）二次检查
        int recheck = ctl.get();
        //（4.2）如果当前线程池状态不是RUNNING则从队列删除任务，并执行拒绝策略
        if (! isRunning(recheck) && remove(command))
            reject(command);

        //（4.3）否者如果当前线程池线程空，则添加一个线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    //（5）如果队列满了，则新增线程，新增失败则执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}
```

- 代码（3）判断如果当前线程池线程个数小于 corePoolSize，会在 workers 里面新增一个核心线程（core 线程）执行该任务。

- 如果当前线程池线程个数大于等于 corePoolSize 执行代码（4），如果当前线程池处于 RUNNING 状态则添加当前任务到任务队列，这里需要判断线程池状态是因为有可能线程池已经处于非 RUNNING 状态，而非 RUNNING 状态下是抛弃新任务的。

- 如果任务添加任务队列成功，则代码（4.2）对线程池状态进行二次校验，这是因为添加任务到任务队列后，执行代码（4.2）前有可能线程池的状态已经变化了，这里进行二次校验，如果当前线程池状态不是 RUNNING 了则把任务从任务队列移除，移除后执行拒绝策略；如果二次校验通过，则执行代码（4.3）重新判断当前线程池里面是否还有线程，如果没有则新增一个线程。

- 如果代码（4）添加任务失败，则说明任务队列满了，则执行代码（5）尝试新开启线程来执行该任务，如果当前线程池线程个数 > maximumPoolSize 则执行拒绝策略。

接下来看新增线程的 addWorkder 方法的源码，如下：
```
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        //（6） 检查队列是否只在必要时为空
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        //（7）循环cas增加线程个数
        for (;;) {
            int wc = workerCountOf(c);

            //（7.1）如果线程个数超限则返回false
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //（7.2）cas增加线程个数，同时只有一个线程成功
            if (compareAndIncrementWorkerCount(c))
                break retry;
            //（7.3）cas失败了，则看线程池状态是否变化了，变化则跳到外层循环重试重新获取线程池状态，否者内层循环重新cas。
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
        }
    }

    //（8）到这里说明cas成功了
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        //（8.1）创建worker
        final ReentrantLock mainLock = this.mainLock;
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {

            //（8.2）加独占锁，为了workers同步，因为可能多个线程调用了线程池的execute方法。
            mainLock.lock();
            try {

                //（8.3）重新检查线程池状态，为了避免在获取锁前调用了shutdown接口
                int c = ctl.get();
                int rs = runStateOf(c);

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    //（8.4）添加任务
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            //（8.5）添加成功则启动任务
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
如上代码主要分两部分，第一部分的双重循环目的是通过 cas 操作增加线程池线程数，第二部分主要是并发安全的把任务添加到 workers 里面，并且启动任务执行。

先看第一部分的代码（6)，如下所示:
```
rs >= SHUTDOWN &&
               ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty())
```
这样看不好理解，我们展开！运算符后，相当于：
```
s >= SHUTDOWN &&
                (rs != SHUTDOWN ||//(1)
              firstTask != null ||//(2)
              workQueue.isEmpty())//(3)
```
如上代码，也就是说代码（6）在下面几种情况下会返回 false：
- 当前线程池状态为 STOP，TIDYING，TERMINATED；
- 当前线程池状态为 SHUTDOWN 并且已经有了第一个任务；
- 当前线程池状态为 SHUTDOWN 并且任务队列为空。
  回到上面看新增线程的 addWorkder 方法，发现内层循环作用是使用 cas 增加线程，代码（7.1）如果线程个数超限则返回 false，否者执行代码（7.2）执行 CAS 操作设置线程个数，cas 成功则退出双循环，CAS 失败则执行代码（7.3）看当前线程池的状态是否变化了，如果变了，则重新进入外层循环重新获取线程池状态，否者进入内层循环继续进行 cas 尝试。

执行到第二部分的代码（8）说明使用 CAS 成功的增加了线程个数，但是现在任务还没开始执行，这里使用全局的独占锁来控制把新增的 Worker 添加到工作集 workers。代码（8.1）创建了一个工作线程 Worker。

代码（8.2）获取了独占锁，代码（8.3）重新检查线程池状态，这是为了避免在获取锁前其他线程调用了 shutdown 关闭了线程池，如果线程池已经被关闭，则释放锁，新增线程失败，否者执行代码（8.4）添加工作线程到线程工作集，然后释放锁，代码（8.5）如果判断如果工作线程新增成功，则启动工作线程。

## 工作线程 Worker 的执行
当用户线程提交任务到线程池后，具体是使用 worker 来执行的，先看下 Worker 的构造函数：
```
Worker(Runnable firstTask) {
    setState(-1); // 在调用runWorker前禁止中断
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);//创建一个线程
}
```
如上代码构造函数内首先设置 Worker 的状态为 -1，是为了避免当前 worker 在调用 runWorker 方法前被中断（当其它线程调用了线程池的 shutdownNow 时候，如果 worker 状态 >= 0 则会中断该线程）。这里设置了线程的状态为 -1，所以该线程就不会被中断了。如下代码运行 runWorker 的代码（9）时候会调用 unlock 方法，该方法把 status 变为了 0，所以这时候调用 shutdownNow 会中断 worker 线程了。

接着我们再看看runWorker方法，代码如下：
```
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); //(9)status设置为0，允许中断
        boolean completedAbruptly = true;
        try {
           //(10)
            while (task != null || (task = getTask()) != null) {

                 //(10.1)
                w.lock();
               ...
                try {
                    //(10.2)任务执行前干一些事情
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();//(10.3)执行任务
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        //(10.4)任务执行完毕后干一些事情
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    //(10.5)统计当前worker完成了多少个任务
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {

            //(11)执行清工作
            processWorkerExit(w, completedAbruptly);
        }
    }
```
如上代码（10）如果当前 task==null 或者调用 getTask 从任务队列获取的任务返回 null，则跳转到代码（11）执行。如果 task 不为 null 则执行代码（10.1）获取工作线程内部持有的独占锁，然后执行扩展接口代码（10.2）在具体任务执行前做一些事情，代码（10.3）具体执行任务，代码（10.4）在任务执行完毕后做一些事情，代码（10.5）统计当前 worker 完成了多少个任务，并释放锁。

这里在执行具体任务期间加锁，是为了避免任务运行期间，其他线程调用了 shutdown 或者 shutdownNow 命令关闭了线程池。

其中代码（11）执行清理任务，其代码如下:
```
private void processWorkerExit(Worker w, boolean completedAbruptly) {
     ...代码太长，这里就不展示了

    //(11.1)统计整个线程池完成的任务个数,并从工作集里面删除当前woker
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    //(11.2)尝试设置线程池状态为TERMINATED，如果当前是shutdonw状态并且工作队列为空
    //或者当前是stop状态当前线程池里面没有活动线程
    tryTerminate();

    //(11.3)如果当前线程个数小于核心个数，则增加
    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```
如上代码（11.1）统计线程池完成任务个数，可知在统计前加了全局锁，把当前工作线程中完成的任务累加到全局计数器，然后从工作集中删除当前 worker。

代码（11.2）判断如果当前线程池状态是 shutdonw 状态并且工作队列为空或者当前是 stop 状态当前线程池里面没有活动线程则设置线程池状态为 TERMINATED，如果设置为了 TERMINATED 状态还需要调用条件变量 termination 的 signalAll() 方法激活所有因为调用线程池的 awaitTermination 方法而被阻塞的线程

代码（11.3）则判断当前线程里面线程个数是否小于核心线程个数，如果是则新增一个线程。

shutdown 操作：调用 shutdown 后，线程池就不会在接受新的任务了，但是工作队列里面的任务还是要执行的，该方法立刻返回的，并不等待队列任务完成在返回。代码如下：
```
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //(12)权限检查
        checkShutdownAccess();

        //(13)设置当前线程池状态为SHUTDOWN，如果已经是SHUTDOWN则直接返回
        advanceRunState(SHUTDOWN);

        //(14)设置中断标志
        interruptIdleWorkers();
        onShutdown(); 
    } finally {
        mainLock.unlock();
    }
    //(15)尝试状态变为TERMINATED
    tryTerminate();
}
```
如上代码（12）检查如果设置了安全管理器，则看当前调用 shutdown 命令的线程是否有关闭线程的权限，如果有权限则还要看调用线程是否有中断工作线程的权限，如果没有权限则抛出 SecurityException 或者 NullPointerException 异常。

其中代码（13）内容如下，如果当前状态 >= SHUTDOWN 则直接返回，否者设置当前状态为 SHUTDOWN：
```
private void advanceRunState(int targetState) {
    for (;;) {
        int c = ctl.get();
        if (runStateAtLeast(c, targetState) ||
            ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
            break;
    }
}
```
代码（14）内容如下，设置所有空闲线程的中断标志，这里首先加了全局锁，同时只有一个线程可以调用 shutdown 设置中断标志，然后尝试获取 worker 自己的锁，获取成功则设置中断标识，由于正在执行的任务已经获取了锁，所以正在执行的任务没有被中断。这里中断的是阻塞到 getTask() 方法，企图从队列里面获取任务的线程，也就是空闲线程。

```
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            //如果工作线程没有被中断，并且没有正在运行则设置设置中断
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```
代码（15）判断如果当前线程池状态是 shutdonw 状态并且工作队列为空或者当前是 stop 状态当前线程池里面没有活动线程则设置线程池状态为 TERMINATED，如果设置为了 TERMINATED 状态还需要调用条件变量 termination 的 signalAll()方法激活所有因为调用线程池的 awaitTermination 方法而被阻塞的线程

### shutdownNow 操作
调用 shutdownNow 后，线程池就不会在接受新的任务了，并且丢弃工作队列里面里面的任务，正在执行的任务会被中断，该方法是立刻返回的，并不等待激活的任务执行完成在返回。返回值为这时候队列里面被丢弃的任务列表。代码如下：
```
public List<Runnable> shutdownNow() {


    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();//（16)权限检查
        advanceRunState(STOP);//(17) 设置线程池状态为stop
        interruptWorkers();//(18)中断所有线程
        tasks = drainQueue();//（19）移动队列任务到tasks
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}

```

如上代码首先调用代码（16）检查权限，然后调用代码（17）设置当前线程池状态为 stop，然后执行代码（18）中断所有的工作线程，这里需要注意的是中断所有的线程，包含空闲线程和正在执行任务的线程，代码如下：
```
  private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
```
然后代码（19）移动当前任务队列里面任务到 tasks 列表。

### awaitTermination 操作
当线程调用 awaitTermination 方法后，当前线程会被阻塞，知道线程池状态变为了 TERMINATED 才返回，或者等待时间超时才返回，整个过程独占锁,代码如下：
```
public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (;;) {
                if (runStateAtLeast(ctl.get(), TERMINATED))
                    return true;
                if (nanos <= 0)
                    return false;
                nanos = termination.awaitNanos(nanos);
            }
        } finally {
            mainLock.unlock();
        }
    }
```
如上代码首先获取了独占锁，然后无限循环内部首先判断当前线程池状态是否至少是 TERMINATED 状态，如果是则直接返回。否者说明当前线程池里面还有线程在执行，则看设置的超时时间 nanos 是否小于 0，小于 0 则说明不需要等待，则直接返回；如果大于0则调用条件变量 termination 的 awaitNanos 方法等待 nanos 时间，期望在这段时间内线程池状态内变为 TERMINATED 状态。

在讲解 shutdown 方法时候提到当线程池状态变为 TERMINATED 后，会调用 termination.signalAll() 用来激活调用条件变量 termination 的 await 系列方法被阻塞的所有线程，所以如果在调用了 awaitTermination 之后调用了 shutdown 方法，并且 shutdown 内部设置线程池状态为 TERMINATED 了，则 termination.awaitNanos 方法会返回。

另外在工作线程 Worker 的 runWorker 方法内当工作线程运行结束后，会调用 processWorkerExit 方法，processWorkerExit 方法内部也会调用 tryTerminate 方法测试当前是否应该把线程池设置为 TERMINATED 状态，如果是，则也会调用 termination.signalAll() 用来激活调用线程池的 awaitTermination 方法而被阻塞的线程

另外当等待时间超时后，termination.awaitNanos 也会返回，这时候会重新检查当前线程池状态是否为 TERMINATED，如果是则直接返回，否者继续阻塞挂起自己。

## 使用线程池需要注意的地方
创建线程池时候要指定与业务相关的名字，以便于追溯问题
日常开发中当一个应用中需要创建多个线程池时候最好给线程池根据业务类型设置具体的名字，以便在出现问题时候方便进行定位，下面就通过实例来说明不设置时候为何难以定位问题，以及如何进行设置。

下面通过简单的代码来说明不指定线程池名称为何难定位问题，代码如下：
```
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * 
 */
public class ThreadPoolExecutorTest {
    static ThreadPoolExecutor executorOne = new ThreadPoolExecutor(5, 5, 1, TimeUnit.MINUTES, new LinkedBlockingQueue<>());
    static ThreadPoolExecutor executorTwo = new ThreadPoolExecutor(5, 5, 1, TimeUnit.MINUTES, new LinkedBlockingQueue<>());

    public static void main(String[] args) {

        //接受用户链接模块
        executorOne.execute(new  Runnable() {
            public void run() {
                System.out.println("接受用户链接线程");
                throw new NullPointerException();
            }
        });
        //具体处理用户请求模块
        executorTwo.execute(new  Runnable() {
            public void run() {
                System.out.println("具体处理业务请求线程");
            }
        });

        executorOne.shutdown();
        executorTwo.shutdown();
    }
}
```

同理我们并不知道是那个模块的线程池抛出了这个异常，那么我们看下这个 pool-1-thread-1 是如何来的。其实是使用了线程池默认的 ThreadFactory，翻看线程池创建的源码如下：
```
 public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }

   public static ThreadFactory defaultThreadFactory() {
        return new DefaultThreadFactory();
   }

static class DefaultThreadFactory implements ThreadFactory {
        //(1)
        private static final AtomicInteger poolNumber = new AtomicInteger(1);
        private final ThreadGroup group;
        //(2)
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        //(3)
        private final String namePrefix;

        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
        }

        public Thread newThread(Runnable r) {
           //(4)
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
            if (t.isDaemon())
                t.setDaemon(false);
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }

```
如上代码 DefaultThreadFactory 的实现可知:

- 代码（1）poolNumber 是 static 的原子变量用来记录当前线程池的编号，它是应用级别的，所有线程池公用一个，比如创建第一个线程池时候线程池编号为1，创建第二个线程池时候线程池的编号为2，这里 pool-1-thread-1 里面的 pool-1 中的 1 就是这个值。

- 代码（2）threadNumber 是线程池级别的，每个线程池有一个该变量用来记录该线程池中线程的编号，这里 pool-1-thread-1 里面的 thread - 1 中的 1 就是这个值。

- 代码（3）namePrefix是线程池中线程的前缀，默认固定为pool。

- 代码（4）具体创建线程，可知线程的名称使用 namePrefix + threadNumber.getAndIncrement() 拼接的。

### 线程池中使用 ThreadLocal 导致的内存泄露
下面先看线程池中使用 ThreadLocal 的例子：
```
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;

/**
 */
public class ThreadPoolTest {

    static class LocalVariable {
        private Long[] a = new Long[1024 * 1024];
    }

    // (1)
    final static ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(5, 5, 1, TimeUnit.MINUTES,
            new LinkedBlockingQueue<>());
    // (2)
    final static ThreadLocal<LocalVariable> localVariable = new ThreadLocal<LocalVariable>();

    public static void main(String[] args) throws InterruptedException {
        // (3)
        for (int i = 0; i < 50; ++i) {
            poolExecutor.execute(new Runnable() {
                public void run() {
                    // (4)
                    localVariable.set(new LocalVariable());
                    // (5)
                    System.out.println("use local varaible");
                    //localVariable.remove();

                }
            });

            Thread.sleep(1000);
        }
        // (6)
        System.out.println("pool execute over");
    }
}
```
代码（1）创建了一个核心线程数和最大线程数为 5 的线程池，这个保证了线程池里面随时都有 5 个线程在运行。

代码（2）创建了一个 ThreadLocal 的变量，泛型参数为 LocalVariable，LocalVariable 内部是一个 Long 数组。

代码（3）向线程池里面放入 50 个任务

代码（4）设置当前线程的 localVariable 变量，也就是把 new 的 LocalVariable 变量放入当前线程的 threadLocals 变量。

由于没有调用线程池的 shutdown 或者 shutdownNow 方法所以线程池里面的用户线程不会退出，进而 JVM 进程也不会退出。

从运行结果一可知，当主线程处于休眠时候进程占用了大概 77M 内存，运行结果二则占用了大概 25M 内存，可知运行代码一时候内存发生了泄露，下面分析下泄露的原因。

运行结果一的代码，在设置线程的 localVariable 变量后没有调用 localVariable.remove()方法，导致线程池里面的 5 个线程的 threadLocals 变量里面的 new LocalVariable() 实例没有被释放，虽然线程池里面的任务执行完毕了，但是线程池里面的 5 个线程会一直存在直到 JVM 进程被杀死。

这里需要注意的是由于 localVariable 被声明了 static，虽然线程的 ThreadLocalMap 里面是对localVariable的弱引用，localVariable也不会被回收。

运行结果二的代码由于线程在设置 localVariable 变量后及时调用了 localVariable.remove() 方法进行了清理，所以不会存在内存泄露。

总结：线程池里面设置了 ThreadLocal 变量一定要记得及时清理，因为线程池里面的核心线程是一直存在的，如果不清理，那么线程池的核心线程的 threadLocals 变量一直会持有 ThreadLocal 变量。
