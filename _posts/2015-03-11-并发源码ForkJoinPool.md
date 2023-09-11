---
layout: post
categories: [JUC]
description: none
keywords: JUC
---
# 并发源码ForkJoinPool
ForkJoinPool 是用于并行执行任务的框架， 是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。Fork就是把一个大任务切分为若干子任务并行的执行，Join就是合并这些子任务的执行结果，最后得到这个大任务的结果。

## ForkJoin框架
ForkJoinPool作为线程池，ForkJoinTask为任务，ForkJoinWorkerThread作为执行任务的线程，三者构成的任务调度机制在 Java 中通常被称为ForkJoin 框架

ForkJoin框架在 Java 7 的时候就加入到了 Java 并发包 java.util.concurrent，并且在 Java 8 的 lambda 并行流 中充当着底层框架的角色。其主要实现的功能是采用“分而治之”的算法将一个大型复杂任务 fork()分解成足够小的任务才使用多线程去并行处理这些小任务，处理完得到各个小任务的执行结果后再进行join()合并，将其汇集成一个大任务的结果，最终得到最初提交的那个大型复杂任务的执行结果。

JDK 7加入的一个线程池类。Fork/Join 技术是分治算法（Divide-and-Conquer）的并行实现，它是一项可以获得良好的并行性能的简单且高效的设计技术。目的是为了帮助我们更好地利用多处理器带来的好处，使用所有可用的运算能力来提升应用的性能。我们常用的数组工具类 Arrays 在JDK 8之后新增的并行排序方法（parallelSort）就运用了 ForkJoinPool 的特性，还有 ConcurrentHashMap 在JDK 8之后添加的函数式方法（如forEach等）也有运用。在整个JUC框架中，ForkJoinPool 相对其他类会复杂很多。

ForkJoin框架适用于非循环依赖的纯函数的计算或孤立对象的操作，如果存在 I/O，线程间同步，sleep() 等会造成线程长时间阻塞的情况时会因为任务依赖影响整体效率，此时可配合使用 ManagedBlocker。ForkJoinPool 线程池为了提高任务的并行度和吞吐量做了很多复杂的设计实现，目的是充分利用CPU，其中最著名的就是 work stealing 任务窃取机制

ManagedBlocker 相当于明确告诉 ForkJoinPool 有个任务要阻塞了，ForkJoinPool 就会启用另一个线程来运行任务，以最大化地利用CPU

## ForkJoinPool 的组件
Fork/Join 框架主要由 ForkJoinPool、ForkJoinWorkerThread 和 ForkJoinTask 来实现，它们之间有着很复杂的联系。ForkJoinPool 中只可以运行 ForkJoinTask 类型的任务（在实际使用中，也可以接收 Runnable/Callable 任务，但在真正运行时，也会把这些任务封装成 ForkJoinTask 类型的任务）；而 ForkJoinWorkerThread 是运行 ForkJoinTask 任务的工作线程。

ForkJoinPool 的另一个特性是它使用了work-stealing（工作窃取）算法：线程池内的所有工作线程都尝试找到并执行已经提交的任务，或者是被其他活动任务创建的子任务（如果不存在就阻塞等待）。这种特性使得 ForkJoinPool 在运行多个可以产生子任务的任务，或者是提交的许多小任务时效率更高。尤其是构建异步模型的 ForkJoinPool 时，对不需要合并（join）的事件类型任务也非常适用。

在 ForkJoinPool 中，线程池中每个工作线程（ForkJoinWorkerThread）都对应一个任务队列（WorkQueue），工作线程优先处理来自自身队列的任务（LIFO或FIFO顺序，参数 mode 决定），然后以FIFO的顺序随机窃取其他队列中的任务。

ForkJoinPool 作为最核心的组件，维护了所有任务队列的数组workQueues。这个数组的结构与 HashMap 底层数组一致，大小为 2 的次幂，每个工作线程用 ThreadLocalRandom.getProbe()作为 hash 值，经过计算即可得出工作线程私有的任务队列WorkQueue在任务队列数组workQueues中的数组下标

另外需注意，WorkQueue 任务队列其实也分为了两种类型，一种是 Submission Queue，也就是外部提交进来的任务所占用的队列，其在任务队列数组中的数组下标为偶数；另一种是Work Queue，属于工作线程私有的任务队列，保存大任务 fork 分解出来的任务，其在任务队列数组中的数组下标为奇数

Work Queue类型的WorkQueue对象通过变量 pool 持有 ForkJoinPool线程池 引用，用来获取其 workQueues 任务队列数组，从而窃取其他工作线程的任务来执行。WorkQueue中的ForkJoinTask<?>[] array数组保存了每一个具体的任务，数组的大小也为 2 的次幂，这个数据是为了高效定位数组中的元素。其初始化大小为 8192，任务在数组中的下标初始化为 4096，插入数组中的第一个任务通常是处理量最大的任务。WorkQueue对象中的owner指向其所属的ForkJoinWorkerThread工作线程，这样ForkJoinPool 就可以通过 WorkQueue 来操作工作线程

ForkJoinWorkerThread工作线程启动后就会扫描偷取任务执行，另外当其在 ForkJoinTask#join() 等待返回结果时如果被 ForkJoinPool 线程池发现其任务队列为空或者已经将当前任务执行完毕，也会通过工作窃取算法从其他任务队列中获取任务分配到其任务队列中并执行

## 任务的分类与分布情况
ForkJoinPool 中的任务分为两种：一种是本地提交的任务（Submission task，如 execute、submit 提交的任务）；另外一种是 fork 出的子任务（Worker task）。两种任务都会存放在 WorkQueue 数组中，但是这两种任务并不会混合在同一个队列里，ForkJoinPool 内部使用了一种随机哈希算法（有点类似 ConcurrentHashMap 的桶随机算法）将工作队列与对应的工作线程关联起来，Submission 任务存放在 WorkQueue 数组的偶数索引位置，Worker 任务存放在奇数索引位。实质上，Submission 与 Worker 一样，只不过他它们被限制只能执行它们提交的本地任务，在后面的源码解析中，我们统一称之为“Worker”。

## 线程池 ForkJoinPool
ForkJoinPool 的继承体系如下，可以看到它和 ThreadPoolExecutor 一样都是继承自AbstractExecutorService抽象类，所以它和 ThreadPoolExecutor 的使用几乎没什么区别，只是任务对象变成了 ForkJoinTask

ForkJoinPool 线程池的创建
与其他线程池类似，ForkJoinPool 线程池可使用以下方式创建

Executors.newWorkStealingPool()
```
public static ExecutorService newWorkStealingPool(int parallelism) {
     return new ForkJoinPool
         (parallelism,
          ForkJoinPool.defaultForkJoinWorkerThreadFactory,
          null, true);
 }
```

构造方法中各个参数的含义如下：

- parallelism: 并行度
即配置线程池线程个数，如果没有指定，则默认为Runtime.getRuntime().availableProcessors() - 1，最大值不能超过 MAX_CAP =32767，可通过以下方式自定义配置
```
System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "8");
// 或者启动参数指定
-Djava.util.concurrent.ForkJoinPool.common.parallelism=8
```

- factory: 工作线程 ForkJoinWorkerThread 的创建工厂
- handler: 未捕获异常的处理器类
多线程中子线程抛出的异常是无法被主线程 catch 的，因此可能需要使用 UncaughtExceptionHandler 对异常进行统一处理
- asyncMode: 任务队列处理任务模式
默认为 false，使用 LIFO(后入先出，类似栈结构)策略处理任务；为 true 时处理策略是 FIFO(先进先出)，更接近于一个消息队列，不适用于处理递归式的任务


### ForkJoinPool 线程池内部重要属性
```
    // 配合ctl在控制线程数量时使用
    private static final long ADD_WORKER = 0x0001L << (TC_SHIFT + 15); 
  
    // 控制ForkJoinPool创建线程数量，(ctl & ADD_WORKER) != 0L 时创建线程，也就是当ctl的第16位不为0时，可以继续创建线程
    volatile long ctl;                   // main pool control
    // 全局锁控制，全局运行状态
    volatile int runState;               // lockable status
    // 低16位表示并行数量，高16位表示 ForkJoinPool 处理任务的模式(异步/同步)
    final int config;                    // parallelism, mode
    // 工作任务队列数组
    volatile WorkQueue[] workQueues;     // main registry
    // 默认线程工厂，默认实现是DefaultForkJoinWorkerThreadFactory
    final ForkJoinWorkerThreadFactory factory;


```

### 工作线程 ForkJoinWorkerThread
ForkJoinWorkerThread的继承体系如下， 它继承了 Thread 类，主要在 Thread的基础上新增了变量，并重写run() 方法

ForkJoinWorkerThread 的创建
与其他线程池类似，ForkJoinWorkerThread 线程由线程池创建，创建后立即调用start()方法启动。 ForkJoinWorkerThread创建时会向ForkJoinPool线程池注册，此时线程池会将其设置为守护线程，并为其创建 ForkJoinPool.WorkQueue 队列后将引用返回，之后ForkJoinWorkerThread就可以从 workQueue 里面取任务出来处理了

守护线程
可以认为是属于 JVM 自身使用的线程，区别于用户线程。如果用户线程全部执行结束，意味着程序需要完成的业务操作已经结束，系统可以关闭退出。所以当系统中只剩下守护线程的时候，JVM 会自动退出

ForkJoinPool#createWorker()
```
private boolean createWorker() {
     ForkJoinWorkerThreadFactory fac = factory;
     Throwable ex = null;
     ForkJoinWorkerThread wt = null;
     try {
         if (fac != null && (wt = fac.newThread(this)) != null) {
             wt.start();
             return true;
         }
     } catch (Throwable rex) {
         ex = rex;
     }
     deregisterWorker(wt, ex);
     return false;
 }

```

ForkJoinWorkerThread#ForkJoinWorkerThread()
```
protected ForkJoinWorkerThread(ForkJoinPool pool) {
     // Use a placeholder until a useful name can be set in registerWorker
     super("aForkJoinWorkerThread");
     this.pool = pool;
     this.workQueue = pool.registerWorker(this);
 }

```

ForkJoinWorkerThread 重要属性
```
   // 工作线程所属线程池
   final ForkJoinPool pool;                // the pool this thread works in
   // 私有的任务队列，在 ForkJoinWorkerThread 新建的时候由线程池为其创建
   final ForkJoinPool.WorkQueue workQueue; // work-stealing mechanics

```

线程任务 ForkJoinTask
ForkJoinTask 与FutureTask一样实现了Future接口，不过它是一个抽象类，使用时通常不会直接实现ForkJoinTask，而是实现其三个抽象子类。ForkJoinTask仅仅用于配合ForkJoinPool实现任务的调度执行，实现其抽象子类则侧重于提供任务的拆分与执行逻辑

- RecursiveAction
用于大多数不返回结果的计算
- RecursiveTask
用于返回结果的计算
- CountedCompleter
用于那些操作完成之后触发其他操作的操作，提供了外部回调方法， Java 8 新增，在 Stream 并行流中使用极多。CountedCompleter 在创建实例的时候还可以传入一个CountedCompleter实例，因此可以形成树状的任务结构，树上的所有任务是可以并行执行的，且每一个子任务完成后都可以通过tryComplete()辅助其父任务的完成


## ForkJoinTask 重要属性
status 字段将运行控制状态位打包到单个 int 中，以最小化占用空间并通过CAS确保原子性，其高16位存储任务执行状态例如 NORMAL，CANCELLED 或 EXCEPTIONAL，低16位预留用于用户自定义的标记

任务未完成之前status大于等于0，完成之后就是 NORMAL、CANCELLED 或 EXCEPTIONAL这几个小于 0 的值，这几个值也有大小顺序：0（初始状态） > NORMAL > CANCELLED > EXCEPTIONAL
```
    /*
     * The status field holds run control status bits packed into a
     * single int to minimize footprint and to ensure atomicity (via
     * CAS).  Status is initially zero, and takes on nonnegative
     * values until completed, upon which status (anded with
     * DONE_MASK) holds value NORMAL, CANCELLED, or EXCEPTIONAL. Tasks
     * undergoing blocking waits by other threads have the SIGNAL bit
     * set.  Completion of a stolen task with SIGNAL set awakens any
     * waiters via notifyAll. Even though suboptimal for some
     * purposes, we use basic builtin wait/notify to take advantage of
     * "monitor inflation" in JVMs that we would otherwise need to
     * emulate to avoid adding further per-task bookkeeping overhead.
     * We want these monitors to be "fat", i.e., not use biasing or
     * thin-lock techniques, so use some odd coding idioms that tend
     * to avoid them, mainly by arranging that every synchronized
     * block performs a wait, notifyAll or both.
     *
     * These control bits occupy only (some of) the upper half (16
     * bits) of status field. The lower bits are used for user-defined
     * tags.
     */

    /** The run status of this task */
    volatile int status; // accessed directly by pool and workers
    static final int DONE_MASK   = 0xf0000000;  // mask out non-completion bits
    static final int NORMAL      = 0xf0000000;  // must be negative
    static final int CANCELLED   = 0xc0000000;  // must be < NORMAL
    static final int EXCEPTIONAL = 0x80000000;  // must be < CANCELLED
    static final int SIGNAL      = 0x00010000;  // must be >= 1 << 16
    static final int SMASK       = 0x0000ffff;  // short bits for tags

```

### 外部任务提交的方法
ForkJoinPool 主要提供了以下三个方法来完成任务的提交。这三个方法在处理流程上并无明显差别，主要差异体现在其方法返回值上

- invoke()	返回任务执行的结果
- execute()	无返回值
- submit()	返回 ForkJoinTask 对象，可以通过该对象取消任务/获取结果

### 外部任务提交的流程
以 ForkJoinPool#invoke() 方法为触发起点，可以看到其首先调用 externalPush(task) 方法将任务提交， 然后调用task.join()等待获取任务执行结果
```
public <T> T invoke(ForkJoinTask<T> task) {
     if (task == null)
         throw new NullPointerException();
     externalPush(task);
     return task.join();
 }

```
ForkJoinPool#externalPush() 首先需要判断线程池中的任务队列数组 workQueues 以及当前线程所对应的Submission Queue是否已经创建，如果已经创建了，则将任务入队；如果没有创建，则调用 externalSubmit(task)
```
final void externalPush(ForkJoinTask<?> task) {
     WorkQueue[] ws; WorkQueue q; int m;
     int r = ThreadLocalRandom.getProbe();
     int rs = runState;
     if ((ws = workQueues) != null && (m = (ws.length - 1)) >= 0 &&
         (q = ws[m & r & SQMASK]) != null && r != 0 && rs > 0 &&
         U.compareAndSwapInt(q, QLOCK, 0, 1)) {
         ForkJoinTask<?>[] a; int am, n, s;
         if ((a = q.array) != null &&
             (am = a.length - 1) > (n = (s = q.top) - q.base)) {
             int j = ((am & s) << ASHIFT) + ABASE;
             U.putOrderedObject(a, j, task);
             U.putOrderedInt(q, QTOP, s + 1);
             U.putIntVolatile(q, QLOCK, 0);
             if (n <= 1)
                 signalWork(ws, q);
             return;
         }
         U.compareAndSwapInt(q, QLOCK, 1, 0);
     }
     externalSubmit(task);
 }

```

ForkJoinPool#externalSubmit() 内部是一个空循环，跳出逻辑主要结合runState来控制，内部工作分为3步，每个步骤都使用了锁的机制来处理并发事件，既有对 runState 使用 ForkJoinPool 的全局锁，也有对WorkQueue使用局部锁

- 完成任务队列数组 workQueues的初始化，数组长度为大于等于2倍并行数量的且是2的n次幂的数，然后扭转 runState
- 计算出一个workQueues数组偶数下标，如该位置没有任务队列，新建一个 Submission Queue 队列，并将其添加到 workQueues 数组对应下标。队列中的ForkJoinTask<?>[] array数组初始化大小为 8192，双端指针base/top初始化为 4096(top指针用来表示任务正常出队入队，每当有任务入队 top +1，每当有任务出列 top -1。base指针用来表示任务偷取状况，每当任务被偷取 base +1。当 base 小于top时，说明有数据可以取)
- 计算出的workQueues数组偶数下标位置已经有任务队列，将任务 CAS 入队，任务队列 top +1，标记任务为已提交，并调用 signalWork()，跳出空循环
```
private void externalSubmit(ForkJoinTask<?> task) {
    int r;                                    // initialize caller's probe
    if ((r = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();
        r = ThreadLocalRandom.getProbe();
    }
    for (;;) {
        WorkQueue[] ws; WorkQueue q; int rs, m, k;
        boolean move = false;
        if ((rs = runState) < 0) {
            tryTerminate(false, false);     // help terminate
            throw new RejectedExecutionException();
        }
        else if ((rs & STARTED) == 0 ||     // initialize
                 ((ws = workQueues) == null || (m = ws.length - 1) < 0)) {
            int ns = 0;
            rs = lockRunState();
            try {
                if ((rs & STARTED) == 0) {
                    U.compareAndSwapObject(this, STEALCOUNTER, null,
                                           new AtomicLong());
                    // create workQueues array with size a power of two
                    int p = config & SMASK; // ensure at least 2 slots
                    int n = (p > 1) ? p - 1 : 1;
                    n |= n >>> 1; n |= n >>> 2;  n |= n >>> 4;
                    n |= n >>> 8; n |= n >>> 16; n = (n + 1) << 1;
                    workQueues = new WorkQueue[n];
                    ns = STARTED;
                }
            } finally {
                unlockRunState(rs, (rs & ~RSLOCK) | ns);
            }
        }
        else if ((q = ws[k = r & m & SQMASK]) != null) {
            if (q.qlock == 0 && U.compareAndSwapInt(q, QLOCK, 0, 1)) {
                ForkJoinTask<?>[] a = q.array;
                int s = q.top;
                boolean submitted = false; // initial submission or resizing
                try {                      // locked version of push
                    if ((a != null && a.length > s + 1 - q.base) ||
                        (a = q.growArray()) != null) {
                        int j = (((a.length - 1) & s) << ASHIFT) + ABASE;
                        U.putOrderedObject(a, j, task);
                        U.putOrderedInt(q, QTOP, s + 1);
                        submitted = true;
                    }
                } finally {
                    U.compareAndSwapInt(q, QLOCK, 1, 0);
                }
                if (submitted) {
                    signalWork(ws, q);
                    return;
                }
            }
            move = true;                   // move on failure
        }
        else if (((rs = runState) & RSLOCK) == 0) { // create new queue
            q = new WorkQueue(this, null);
            q.hint = r;
            q.config = k | SHARED_QUEUE;
            q.scanState = INACTIVE;
            rs = lockRunState();           // publish index
            if (rs > 0 &&  (ws = workQueues) != null &&
                k < ws.length && ws[k] == null)
                ws[k] = q;                 // else terminated
            unlockRunState(rs, rs & ~RSLOCK);
        }
        else
            move = true;                   // move if busy
        if (move)
            r = ThreadLocalRandom.advanceProbe(r);
    }
}

```
ForkJoinPool#signalWork() 会根据ctl的值判断是否需要创建工作线程，当前工作线程太少会调用tryAddWorker()尝试创建新的工作线程；其他情况如果有工作线程在休眠，会 U.unpark(p)将其唤醒
```
final void signalWork(WorkQueue[] ws, WorkQueue q) {
   long c; int sp, i; WorkQueue v; Thread p;
   while ((c = ctl) < 0L) {                       // too few active
       if ((sp = (int)c) == 0) {                  // no idle workers
           if ((c & ADD_WORKER) != 0L)            // too few workers
               tryAddWorker(c);
           break;
       }
       if (ws == null)                            // unstarted/terminated
           break;
       if (ws.length <= (i = sp & SMASK))         // terminated
           break;
       if ((v = ws[i]) == null)                   // terminating
           break;
       int vs = (sp + SS_SEQ) & ~INACTIVE;        // next scanState
       int d = sp - v.scanState;                  // screen CAS
       long nc = (UC_MASK & (c + AC_UNIT)) | (SP_MASK & v.stackPred);
       if (d == 0 && U.compareAndSwapLong(this, CTL, c, nc)) {
           v.scanState = vs;                      // activate v
           if ((p = v.parker) != null)
               U.unpark(p);
           break;
       }
       if (q != null && q.base == q.top)          // no more work
           break;
   }
}

```

ForkJoinPool#tryAddWorker() 会一直尝试调用createWorker() 创建工作线程，直到控制工作线程数量的相关标记达到目标值
```
private void tryAddWorker(long c) {
   boolean add = false;
   do {
       long nc = ((AC_MASK & (c + AC_UNIT)) |
                  (TC_MASK & (c + TC_UNIT)));
       if (ctl == c) {
           int rs, stop;                 // check if terminating
           if ((stop = (rs = lockRunState()) & STOP) == 0)
               add = U.compareAndSwapLong(this, CTL, c, nc);
           unlockRunState(rs, rs & ~RSLOCK);
           if (stop != 0)
               break;
           if (add) {
               createWorker();
               break;
           }
       }
   } while (((c = ctl) & ADD_WORKER) != 0L && (int)c == 0);
}

```
ForkJoinPool#createWorker() 主要负责通过线程工厂创建工作线程，并调用其start() 方法启动线程
```
private boolean createWorker() {
    ForkJoinWorkerThreadFactory fac = factory;
    Throwable ex = null;
    ForkJoinWorkerThread wt = null;
    try {
        if (fac != null && (wt = fac.newThread(this)) != null) {
            wt.start();
            return true;
        }
    } catch (Throwable rex) {
        ex = rex;
    }
    deregisterWorker(wt, ex);
    return false;
}

```
线程工厂最终调用到ForkJoinWorkerThread#ForkJoinWorkerThread()来创建一个工作线程，并将其向ForkJoinPool线程池注册，之后线程池会为其创建私有的 Work Queue 任务队列后将引用返回
```
protected ForkJoinWorkerThread(ForkJoinPool pool) {
     // Use a placeholder until a useful name can be set in registerWorker
     super("aForkJoinWorkerThread");
     this.pool = pool;
     this.workQueue = pool.registerWorker(this);
 }

```
ForkJoinPool#registerWorker() 将工作线程设置为守护线程，并将新建的Work Queue 任务队列放入任务队列数组奇数下标中。至此任务队列数组workQueus中已经有了两种任务队列Submission Queue和Work Queue
```
final WorkQueue registerWorker(ForkJoinWorkerThread wt) {
     UncaughtExceptionHandler handler;
     wt.setDaemon(true);                           // configure thread
     if ((handler = ueh) != null)
         wt.setUncaughtExceptionHandler(handler);
     WorkQueue w = new WorkQueue(this, wt);
     int i = 0;                                    // assign a pool index
     int mode = config & MODE_MASK;
     int rs = lockRunState();
     try {
         WorkQueue[] ws; int n;                    // skip if no array
         if ((ws = workQueues) != null && (n = ws.length) > 0) {
             int s = indexSeed += SEED_INCREMENT;  // unlikely to collide
             int m = n - 1;
             i = ((s << 1) | 1) & m;               // odd-numbered indices
             if (ws[i] != null) {                  // collision
                 int probes = 0;                   // step by approx half n
                 int step = (n <= 4) ? 2 : ((n >>> 1) & EVENMASK) + 2;
                 while (ws[i = (i + step) & m] != null) {
                     if (++probes >= n) {
                         workQueues = ws = Arrays.copyOf(ws, n <<= 1);
                         m = n - 1;
                         probes = 0;
                     }
                 }
             }
             w.hint = s;                           // use as random seed
             w.config = i | mode;
             w.scanState = i;                      // publication fence
             ws[i] = w;
         }
     } finally {
         unlockRunState(rs, rs & ~RSLOCK);
     }
     wt.setName(workerNamePrefix.concat(Integer.toString(i >>> 1)));
     return w;
 }

```

### 通过 ForkJoinTask 的内部提交

ForkJoinTask#fork()提交，返回 ForkJoinTask对象，通过该对象可以对任务进行控制，比如获取、取消等操作。该方法首先判断当前线程是否是 ForkJoinWorkerThread 线程，如果是说明是内部大任务分解的小任务，直接将任务插入其私有队列，如果不是说明是外部提交，需要走外部任务提交流程

入队时将新任务始终 push 到队列一端的方式可以保证较大的任务在队列的头部，越小的任务越在尾部。这时候拥有该任务队列的工作线程如果按照LIFO(栈结构) 的方式弹出任务执行，将会优先从小任务开始，逐渐往大任务进行，而窃取任务的其他工作线程从队列头部(FIFO方式)开始窃取将会帮助它完成大任务
```
public final ForkJoinTask<V> fork() {
        Thread t;
        if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
            ((ForkJoinWorkerThread)t).workQueue.push(this);
        else
            ForkJoinPool.common.externalPush(this);
        return this;
    }

```

## 任务的处理
ForkJoinWorkerThread 的任务处理循环
ForkJoinWorkerThread工作线程启动后，其 run()方法就会被调用。方法中首先是判断当前工作线程的 workQueue.array是否为 null，毫无疑问，这个判断只在工作线程创建的时候成立一次，之后就会调用线程池的 runWorker() 方法
```
public void run() {
     if (workQueue.array == null) { // only run once
         Throwable exception = null;
         try {
             onStart();
             pool.runWorker(workQueue);
         } catch (Throwable ex) {
             exception = ex;
         } finally {
             try {
                 onTermination(exception);
             } catch (Throwable ex) {
                 if (exception == null)
                     exception = ex;
             } finally {
                 pool.deregisterWorker(this, exception);
             }
         }
     }
 }

```
ForkJoinPool#runWorker() 开始初始化 WorkQueue 工作队列中的 array 数组，并开启了空循环一直调用 scan() 方法扫描偷取任务，获取到任务后即执行
```
final void runWorker(WorkQueue w) {
    // 1. 初始化或者两倍扩容 array 数组
     w.growArray();                   // allocate queue
     int seed = w.hint;               // initially holds randomization hint
     int r = (seed == 0) ? 1 : seed;  // avoid 0 for xorShift
     // 2. 空循环
     for (ForkJoinTask<?> t;;) {
     // 3. scan()方法扫描偷取任务，working stealing 算法
         if ((t = scan(w, r)) != null)
         // 4. 偷取到任务即执行
             w.runTask(t);
          // 5. 未偷取到任务则 await 等待
         else if (!awaitWork(w, r))
             break;
         r ^= r << 13; r ^= r >>> 17; r ^= r << 5; // xorshift
     }
 }

```
ForkJoinPool#scan() 为 work stealing 算法的重要实现，此处暂不分析，接下去看偷取到任务后的执行逻辑 WorkQueue#runTask()，该方法的主要逻辑先调用 (currentSteal = task).doExec() 方法执行偷取到的任务，再调用 execLocalTasks() 方法执行当前任务队列中的任务
```
final void runTask(ForkJoinTask<?> task) {
         if (task != null) {
             scanState &= ~SCANNING; // mark as busy
             (currentSteal = task).doExec();
             U.putOrderedObject(this, QCURRENTSTEAL, null); // release for GC
             execLocalTasks();
             ForkJoinWorkerThread thread = owner;
             if (++nsteals < 0)      // collect on overflow
                 transferStealCount(pool);
             scanState |= SCANNING;
             if (thread != null)
                 thread.afterTopLevelExec();
         }
     }

```
ForkJoinTask#doExec() 方法首先判断任务状态是否大于等于 0 (大于等于 0 说明任务还没有执行完成)，因为这个任务可能被其它线程窃取过去处理完了。方法内主要是执行 exec() 方法，再根据方法返回值设置任务的完成状态
```
final int doExec() {
     int s; boolean completed;
     if ((s = status) >= 0) {
         try {
             completed = exec();
         } catch (Throwable rex) {
             return setExceptionalCompletion(rex);
         }
         if (completed)
             s = setCompletion(NORMAL);
     }
     return s;
 }

```
ForkJoinTask#exec() 是抽象方法，主要由子类重写。以CountedCompleter#exec()方法为例，其内部其实就是调用 compute() 方法，而这个方法正是 CountedCompleter 子类重写实现，内部主要为任务分解与计算的逻辑。任务分解会调用 ForkJoinTask#fork() 方法，根据当前线程实例决定将其提交到线程池Submission Queue 任务队列还是工作线程 Work Queue 私有任务队列
```
protected final boolean exec() {
    compute();
    return false;
}

```

join() 触发的处理流程
ForkJoinTask#join() 会等待任务执行完成，并返回执行结果。若任务被取消抛出CancellationException异常，若是其他异常导致异常结束则抛出相关RuntimeException或Error信息，这些异常可能包括由于内部资源耗尽而导致的RejectedExecutionException，比如分配内部任务队列失败

ForkJoinTask#join() 方法的内部逻辑很简单，就是调用 doJoin() 方法，然后根据任务的状态决定是否要reportException()报告异常，如果任务正常结束，就调用 getRawResult()返回执行结果
```
public final V join() {
     int s;
     if ((s = doJoin() & DONE_MASK) != NORMAL)
         reportException(s);
     return getRawResult();
 }

```
ForkJoinTask#doJoin() 方法内部逻辑很复杂，主要分为了以下几步：

doJoin() 首先判断staus是不是已完成，如果完成了(status < 0)就直接返回，因为这个任务可能被其它线程窃取过去处理掉了
其次判断当前线程是否是 ForkJoinWorkerThread：
是的话直接尝试 tryUnpush(this) 出队然后doExec()执行任务处理。如果没有出队成功并且处理成功，则执行wt.pool.awaitJoin(w, this, 0L)，等待任务执行完成
不是的话执行 externalAwaitDone() 等待外部任务执行完成
```
private int doJoin() {
     int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
     return (s = status) < 0 ? s :
         ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
         (w = (wt = (ForkJoinWorkerThread)t).workQueue).
         tryUnpush(this) && (s = doExec()) < 0 ? s : //调用pool的执行逻辑，并等待返回执行结果状态
         wt.pool.awaitJoin(w, this, 0L) : //调用pool的等待机制
         externalAwaitDone();
 }

```

wt.pool.awaitJoin(w, this, 0L)执行了线程池的方法 ForkJoinPool#awaitJoin()，其内部逻辑如下

- 更新 WorkQueue 的 currentJoin
- 空循环开启，如果任务已经结束（(s = task.status) < 0），则 break
- 如果任务为 CountedCompleter 类型，则调用 helpComplete() 帮助父任务完成
- 队列为空或者任务已经执行成功(可能被其他线程偷取)，则帮助偷取了自己任务的工作线程执行任务（互相帮助）helpStealer()
- 如果任务已经结束（(s = task.status) < 0），则 break
- 如果超时结束，则 break
- 执行失败的情况下，执行补偿操作 tryCompensate()
- 当前任务完成后，替换 currentJoin 为以前的值
```
final int awaitJoin(WorkQueue w, ForkJoinTask<?> task, long deadline) {
     int s = 0;
     if (task != null && w != null) {
         ForkJoinTask<?> prevJoin = w.currentJoin;
         U.putOrderedObject(w, QCURRENTJOIN, task);
         CountedCompleter<?> cc = (task instanceof CountedCompleter) ?
             (CountedCompleter<?>)task : null;
         for (;;) {
             if ((s = task.status) < 0)
                 break;
             if (cc != null)
                 helpComplete(w, cc, 0);
             else if (w.base == w.top || w.tryRemoveAndExec(task))
                 helpStealer(w, task);
             if ((s = task.status) < 0)
                 break;
             long ms, ns;
             if (deadline == 0L)
                 ms = 0L;
             else if ((ns = deadline - System.nanoTime()) <= 0L)
                 break;
             else if ((ms = TimeUnit.NANOSECONDS.toMillis(ns)) <= 0L)
                 ms = 1L;
             if (tryCompensate(w)) {
                 task.internalWait(ms);
                 U.getAndAddLong(this, CTL, AC_UNIT);
             }
         }
         U.putOrderedObject(w, QCURRENTJOIN, prevJoin);
     }
     return s;
 }

```

ForkJoinTask#externalAwaitDone() 的处理逻辑也比较简单，步骤如下：

- 首先判断当前任务是否是 CountedCompleter，是的话调用线程池 externalHelpComplete() 尝试将其入Submission Queue 队列，由某个工作线程偷取帮助执行；否则调用ForkJoinPool.common.tryExternalUnpush(this) ? doExec()尝试把自己出队然后执行掉
- 如果任务没有执行成功，利用 object/wait 的方法去监听任务 status 的状态变更
```
private int externalAwaitDone() {
     int s = ((this instanceof CountedCompleter) ? // try helping
              ForkJoinPool.common.externalHelpComplete(
                  (CountedCompleter<?>)this, 0) :
              ForkJoinPool.common.tryExternalUnpush(this) ? doExec() : 0);
     if (s >= 0 && (s = status) >= 0) {
         boolean interrupted = false;
         do {
             if (U.compareAndSwapInt(this, STATUS, s, s | SIGNAL)) {
                 synchronized (this) {
                     if (status >= 0) {
                         try {
                             wait(0L);
                         } catch (InterruptedException ie) {
                             interrupted = true;
                         }
                     }
                     else
                         notifyAll();
                 }
             }
         } while ((s = status) >= 0);
         if (interrupted)
             Thread.currentThread().interrupt();
     }
     return s;
 }

```
work stealing 算法 的实现主要体现在 ForkJoinPool#scan() 方法，其主要机制如下：

- ForkJoinPool 的每个工作线程都维护着一个私有的双端任务队列 Work Queue，每个工作线程在运行中产生的新任务（通常是因为调用了 fork()），会放入工作队列的队尾，并且工作线程在处理自己的工作队列时，使用的是 LIFO 方式，其实就是栈结构
- 每个工作线程在处理自己的工作队列同时，会尝试窃取一个任务（刚刚提交到线程池的 Submission Queue 队列中的任务，或是来自于其他工作线程的工作队列），窃取的任务位于其他线程的工作队列的队首，也就是使用的 FIFO 方式，目的是减少竞争
- 在遇到 join() 时，如果需要 join 的任务尚未完成，则会先处理其他任务，并等待其完成
- 在既没有自己的任务，也没有可以窃取的任务时，进入休眠

ForkJoinPool除了工作窃取机制，还有一种互助机制，体现在ForkJoinPool#helpStealer()方法 。假设工作线程2窃取了工作线程1的任务之后，通过 fork又分解产生了子任务，这些子任务会进入工作线程2的工作队列。这时候如果工作线程1把剩余的任务都完成了，当它发现自己的任务被工作线程2窃取了，那它也会试着去窃取工作线程2的任务 (你偷了我的，我有空就要偷你的)，这就是互助机制
```
private ForkJoinTask<?> scan(WorkQueue w, int r) {
        WorkQueue[] ws; int m;
        if ((ws = workQueues) != null && (m = ws.length - 1) > 0 && w != null) {
            int ss = w.scanState;                     // initially non-negative
            for (int origin = r & m, k = origin, oldSum = 0, checkSum = 0;;) {
                WorkQueue q; ForkJoinTask<?>[] a; ForkJoinTask<?> t;
                int b, n; long c;
                if ((q = ws[k]) != null) {
                    if ((n = (b = q.base) - q.top) < 0 &&
                        (a = q.array) != null) {      // non-empty
                        long i = (((a.length - 1) & b) << ASHIFT) + ABASE;
                        if ((t = ((ForkJoinTask<?>)
                                  U.getObjectVolatile(a, i))) != null &&
                            q.base == b) {
                            if (ss >= 0) {
                                if (U.compareAndSwapObject(a, i, t, null)) {
                                    q.base = b + 1;
                                    if (n < -1)       // signal others
                                        signalWork(ws, q);
                                    return t;
                                }
                            }
                            else if (oldSum == 0 &&   // try to activate
                                     w.scanState < 0)
                                tryRelease(c = ctl, ws[m & (int)c], AC_UNIT);
                        }
                        if (ss < 0)                   // refresh
                            ss = w.scanState;
                        r ^= r << 1; r ^= r >>> 3; r ^= r << 10;
                        origin = k = r & m;           // move and rescan
                        oldSum = checkSum = 0;
                        continue;
                    }
                    checkSum += b;
                }
                if ((k = (k + 1) & m) == origin) {    // continue until stable
                    if ((ss >= 0 || (ss == (ss = w.scanState))) &&
                        oldSum == (oldSum = checkSum)) {
                        if (ss < 0 || w.qlock < 0)    // already inactive
                            break;
                        int ns = ss | INACTIVE;       // try to inactivate
                        long nc = ((SP_MASK & ns) |
                                   (UC_MASK & ((c = ctl) - AC_UNIT)));
                        w.stackPred = (int)c;         // hold prev stack top
                        U.putInt(w, QSCANSTATE, ns);
                        if (U.compareAndSwapLong(this, CTL, c, nc))
                            ss = ns;
                        else
                            w.scanState = ss;         // back out
                    }
                    checkSum = 0;
                }
            }
        }
        return null;
    }

```
















