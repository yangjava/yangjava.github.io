---
layout: post
categories: [JUC]
description: none
keywords: JUC
---
# 并发源码FutureTask

## FutureTask类源码重点
FutureTask是一种可取消的异步任务，实现了RunnableFuture接口也就相当于实现了Runnable接口，并重写run()方法，所以其任务是在线程中执行的，可以和线程池联用，RunnableFuture源码可以看我这篇文章 RunnableFuture
FutureTask实现RunnableFuture接口也就相当于实现了Future接口，自然就具有其接口的含义，Future源码可以看我这篇文章 Future
FutureTask的操作大致分为执行任务、获取任务执行结果和取消任务三个操作
FutureTask内部有一个等待链表，其节点为WaitNode，节点内部只有一个线程，用于获取任务执行结果时，将想要获取结果的线程放入节点并阻塞，而且放在WaitNode构成的链表中存储，因为当get获取结果时，该任务可能没有执行完成

FutureTask类的实现
FutureTask类属性
state是FutureTask的状态，它的值有NEW(新建)、COMPLETING(完成)、NORMAL(正常)、EXCEPTIONAL(异常)、CANCELLED(取消)、INTERRUPTING(中断中)、INTERRUPTED(已中断)，state表示着FutureTask的任务执行的状态
callable是FutureTask具体执行的任务，Callable接口是函数式接口只定义了一个带返回值的call()方法，是Runnable的改进，Callable源码可以看我这篇文章 Callable
outcome是一个Object，用来存放任务执行完的结果或执行过程发生异常产生的异常对象Throwable
runner 是执行该FutureTask的任务的线程对象
waiters是等待线程的链表头节点，
STATE、RUNNER、WAITERS是变量句柄，变量句柄的作用是代替Atomic类对变量指向CAS或原子操作，为什么JDK9要涌入变量句柄呢，因为以前如果我们想使用CAS或者原子操作，只能将我们变量定义为AtomicInteger、AtomicLong这些类，现在引入变量句柄我们仍然可以像平常一样定义我们的变量，但是还是可以进行CAS和原子操作

```
public class FutureTask<V> implements RunnableFuture<V> {
	private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;

	private Callable<V> callable;
	private Object outcome;
	private volatile Thread runner;
	private volatile WaitNode waiters;

	private static final VarHandle STATE;
    private static final VarHandle RUNNER;
    private static final VarHandle WAITERS;
```

等待线程的链表节点
可以看到每个等待节点内部都有一个线程
```
static final class WaitNode {
    volatile Thread thread;
    volatile WaitNode next;
    WaitNode() { thread = Thread.currentThread(); }
}
```

执行任务
run操作
可以看到只有当state状态为NEW，且用变量句柄RUNNER对变量runner执行CAS操作将执行任务的线程runner设为当前线程成功，才会执行以后的操作，不满足这两个条件中的一个直接返回，这样可以防止多个线程重复执行同一任务
执行任务就是取出Callable执行其call()方法，获取执行结果，如果没有发生异常则调用set方法设置执行结果，如果发生了异常则调用setException方法设置返回结果为异常对象
最终的finally块要将runner即执行任务的线程设为空，而且判断当前状态state是不是中断中和已中断状态，如果是则调用handlePossibleCancellationInterrupt，让出CPU等待任务被中断
最后，FutureTask实现了Runnable，所以这个run方法实际是在线程中执行的
```
public void run() {
	// 防止多个线程执行同一任务
    if (state != NEW ||
        !RUNNER.compareAndSet(this, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        runner = null;
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

set操作
使用CAS尝试把state设为COMPLETING完成状态，如果成功，则将上面call的执行结果也就是传入进来的v赋给outcome，outcome是用来容纳执行结果或执行中的异常对象的
setRelease会把state的值设为NORMAL正常状态，setRelease能确保在此次访问state变量之后和之前的读和写不会重排序，读是从主存上读取变量，写是把写缓冲区的数据刷盘至主存
最后调用finishCompletion方法将waiters即等待节点链表头节点置空，且把等待链表上所有节点的线程置空，并唤醒该阻塞线程
```
protected void set(V v) {
    if (STATE.compareAndSet(this, NEW, COMPLETING)) {
        outcome = v;
        STATE.setRelease(this, NORMAL);
        finishCompletion();
    }
}
```
finishCompletion操作
如果waiters等待节点不为空则进入循环，先使用CAS将waiters设为空，如果成功则进行后续操作，不成功则在外层for循环自旋直至操作成功
CAS成功后，则把q的内部线程置为空，并调用LockSupport.unpark接触该线程的阻塞，一直循环把q链表后面移动，对每个等待节点都进行释放操作并唤醒其内部线程，然后才跳出循环，第一个break跳出内存循环，第二break虽然在if语句里面但是仍然可以跳出第一层循环
最后执行done()操作，done操作是要子类重写的方法，默认为空，用来执行类似任务完成后一个回调方法的操作，再将callable即该FutureTask的任务对象置空
为什么要有这些等待节点呢，因为当runner线程执行任务过程，还会有其他线程调用get()方法想获取任务执行结果，此时其他线程每个都会创建一个WaitNode，放入waiters链表中阻塞等待，直到runner执行完任务后调用finishCompletion把它们全部唤醒，去获取结果
```
private void finishCompletion() {
    for (WaitNode q; (q = waiters) != null;) {
        if (WAITERS.weakCompareAndSet(this, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; 
                q = next;
            }
            break;
        }
    }
    done();
    callable = null; 
}
```

setException操作
可以看到和上面的set方法几乎一模一样，只是最后用setRelease将state设为EXCEPTIONAL异常状态
```
protected void setException(Throwable t) {
    if (STATE.compareAndSet(this, NEW, COMPLETING)) {
        outcome = t;
        STATE.setRelease(this, EXCEPTIONAL); 
        finishCompletion();
    }
}
```

handlePossibleCancellationInterrupt操作
Thread.yield() 方法，使当前线程由执行状态，变成为就绪状态，让出cpu时间，等待其他线程的中断
```
 private void handlePossibleCancellationInterrupt(int s) {
    if (s == INTERRUPTING)
        while (state == INTERRUPTING)
            Thread.yield();
}
```

runAndReset操作
可以看到和run操作几乎一模一样，就是没有执行set方法，set方法作用如下

将Callable的call方法返回的返回值赋值给outcome
把state设为COMPLETING再设为NORMAL
将waiters置空，并遍历等待链表将所有节点的线程置空，并解锁阻塞线程
由于没有执行set操作改变state状态，所以runAndReset()方法是可以多次执行的，runAndReset()设计用于本质上执行多次的任务
```
protected boolean runAndReset() {
    if (state != NEW ||
        !RUNNER.compareAndSet(this, null, Thread.currentThread()))
        return false;
    boolean ran = false;
    int s = state;
    try {
        Callable<V> c = callable;
        if (c != null && s == NEW) {
            try {
                c.call(); 
                ran = true;
            } catch (Throwable ex) {
                setException(ex);
            }
        }
    } finally {
        runner = null;
        s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
    return ran && s == NEW;
}
```

获取任务执行的结果
get操作
获取当前任务执行状态state，如果state为NEW或COMPLETING，则执行awaitDone方法阻塞等待任务执行完成，COMPLETING不算正常完成任务最终状态，NORMAL才是最终状态
然后调用report返回已完成任务的结果或引发异常
```
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}
```
awaitDone操作
startTime是任务开始时间，q是一个等待节点包含当前线程，queued是q节点是否在等待节点链表上面排队，queued为true，说明q节点在等待链表上排队
for循环自旋，首先获取state状态，如果状态大于COMPLETING，则可能是NORMAL(正常完成)、EXCEPTIONAL(异常)、CANCELLED(取消)、INTERRUPTING(中断中)、INTERRUPTED(已中断)这些状态，这些状态都应当直接结束等待，将p的线程置为空,返回当前的state
如果当前状态等于COMPLETING说明已经在NORMAL的中间状态了，任务差一点就做完了，则调用Thread.yield()向调度程序发出的提示，表示当前线程愿意使用处理器，加快任务执行（调度程序可以随意忽略此提示）
如果使用Thread.interrupted方法检测到当前线程被打上中断标志，则从等待链表中移除等待节点p，并抛出InterruptedException中断异常以中断程序
如果q等于null，先判断函数是否设置了超时返回以及是否超时，再创建以当前线程为内部遍历的WaitNode节点p
如果q节点没有在waiters等待链表中排队，则使用CAS设置waiters为q，在这之前把q.next指向waiters，相当于在队头插入q节点，再使用CAS把队头设位置q
如果函数设置了超时返回，则进入判断超时返回的区域，第一次进入时startTime为0，则把startTime设为系统当前的纳秒时间， parkNanos阻塞时长设为函数传参进来的nanos，然后直接判断state是否小于COMPLETING，也就是处于new状态时，用 LockSupport.parkNanos(this, parkNanos)方法阻塞当前线程parkNanos纳秒
第二次及以后进入这个超时判断块时，则计算当前纳秒时间和开始纳秒时间的差值，如果大于给定的超时时间nanos则直接先移除等待链表中的q节点，再返回状态，如果没有大于则计算剩余要阻塞的时间nanos - elapsed，继续调用LockSupport.parkNanos阻塞当前线程
上面有两个地方要注意的，为什么parkNanos定义成final还能在两处地方更改，因为这是一个for循环，而且那两处地方是if和else结果，for循环每次重新进入时会重新定义一次，而if和else每次都只会执行其中一个
另一个要注意的地方是 LockSupport.parkNanos阻塞当前线程那任务怎么执行呢，这就知道调用get()方法的线程和执行任务的线程是不一样的，执行任务的线程是runner，而调用get方法的线程是其他线程，在等runner执行完任务之前会在等待链表中阻塞，直到runner执行完任务后调用finishCompletion把它们全部唤醒，去获取结果
最后一个else，如果前面都不满足，即不满足状态大于等于COMPLETING，不满足当前线程被打上中断标志，不满足还没排队，不满足设置超时时间，即当任务状态为NEW且线程没有被打上中断标志，且在等待链表中排队，没有设置超时的线程会被调用 LockSupport.park一直阻塞，直到runner执行完任务后调用finishCompletion把它们全部唤醒，去获取结果
```
private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        long startTime = 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING)
                Thread.yield();
            else if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }
            else if (q == null) {
                if (timed && nanos <= 0L)
                    return s;
                q = new WaitNode();
            }
            else if (!queued)
                queued = WAITERS.weakCompareAndSet(this, q.next = waiters, q);
            else if (timed) {
                final long parkNanos;
                if (startTime == 0L) { 
                    startTime = System.nanoTime();
                    if (startTime == 0L)
                        startTime = 1L;
                    parkNanos = nanos;
                } 
                else {
                    long elapsed = System.nanoTime() - startTime;
                    if (elapsed >= nanos) {
                        removeWaiter(q);
                        return state;
                    }
                    parkNanos = nanos - elapsed;
                }
                if (state < COMPLETING)
                    LockSupport.parkNanos(this, parkNanos);
            }
            else
                LockSupport.park(this);
        }
    }
```

report操作
s是上面传入的任务执行状态，如果任务执行状态为NORMAL正常，则返回执行结果 outcome
如果 s 大于等于 CANCELLED，说明s >= 4，则s可能是CANCELLED、INTERRUPTING、INTERRUPTED ，这三个状态，则抛出CancellationException异常
如果 s 为 EXCEPTIONAL，也就是 s 等于 3，则outcome就是一个Throwable对象，将其传入ExecutionException中，并抛出ExecutionException异常
```
private V report(int s) throws ExecutionException {
     Object x = outcome;
     if (s == NORMAL)
         return (V)x;
     if (s >= CANCELLED)
         throw new CancellationException();
     throw new ExecutionException((Throwable)x);
 }
```
get(long timeout, TimeUnit unit)操作
可以看出和get差别不大，只是判断当awaitDone方法返回时，如果此时状态仍然小于等于COMPLETING，是任务没完成的状态，则抛出超时异常
```
 public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
    if (unit == null)
        throw new NullPointerException();
    int s = state;
    if (s <= COMPLETING &&
        (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
        throw new TimeoutException();
    return report(s);
}
```
取消任务
cancel操作
第一个if语句有点复杂，我们可以先不看外部的非号!，里面的意思是状态为NEW 且 使用CAS设置state的值成功，而里面的三元运算符只是决定把state设成什么状态而已，加了个非号就是如果状态不为NEW或使用CAS将state设置为其他状态不成功，直接返回false，就是取消失败的意思
只有什么情况才会继续执行下面代码进行取消呢，只有当state为NEW且使用CAS将该state设为INTERRUPTING 或CANCELLED这两个状态成功时，才执行后续操作，这是为了防止多线程并发取消，因为只有一个线程能够CAS成功，则该线程可以执行后续操作
mayInterruptIfRunning是中断的意思，如果设为true，则使用CAS将state设为 INTERRUPTING 中断中状态
try处如果 mayInterruptIfRunning为true，则调用该线程的t.interrupt方法，打上中断标志，并调用setRelease将state设为INTERRUPTED已中断状态
不管是直接取消，还是中断，都会执行finishCompletion，清空等待链表，唤醒所有等待获取结果的线程
```
public boolean cancel(boolean mayInterruptIfRunning) {
   if (!(state == NEW && STATE.compareAndSet
          (this, NEW, mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally {
                STATE.setRelease(this, INTERRUPTED);
            }
        }
    } finally {
        finishCompletion();
    }
    return true;
}
```





























