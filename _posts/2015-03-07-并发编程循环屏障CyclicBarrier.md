---
layout: post
categories: [JUC]
description: none
keywords: JUC
---
# 并发循环屏障CyclicBarrier

## CyclicBarrier（循环屏障）
CyclicBarrier（循环屏障），它也是一个同步助手工具，它允许多个线程在执行完相应的操作之后彼此等待共同到达一个障点（barrier point）。CyclicBarrier也非常适合用于某个串行化任务被分拆成若干个并行执行的子任务，当所有的子任务都执行结束之后再继续接下来的工作。从这一点来看，Cyclic Barrier与CountDownLatch非常类似，但是它们之间的运行方式以及原理还是存在着比较大的差异的，并且CyclicBarrier所能支持的功能CountDownLatch是不具备的。比如，CyclicBarrier可以被重复使用，而CountDownLatch当计数器为0的时候就无法再次利用。

## CyclicBarrier使用示例
演示如何使用CyclicBarrier，相同的场景下使用不同的工具还可以有助于理解它们之间的相同点和不同之处。
```java
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.TimeUnit;
import java.util.stream.IntStream;

import static java.util.concurrent.ThreadLocalRandom.current;
import static java.util.stream.Collectors.toList;

public class CyclicBarrierExample1
{
    public static void main(String[] args)
                throws InterruptedException
    {
        // 根据商品品类获取一组商品ID
        final int[] products = getProductsByCategoryId();
       // 通过转换将商品编号转换为ProductPrice
        List<ProductPrice> list = Arrays.stream(products)
                            .mapToObj(ProductPrice::new)
                            .collect(toList());
       // ① 定义CyclicBarrier ，指定parties为子任务数量
        final CyclicBarrier barrier = new CyclicBarrier(list.size());
       // ② 用于存放线程任务的list
        final List<Thread> threadList = new ArrayList<>();
        list.forEach(pp ->
        {
             Thread thread = new Thread(() ->
             {
                 System.out.println(pp.getProdID() + "start calculate price.");
                 try
                 {
                     TimeUnit.SECONDS.sleep(current().nextInt(10));
                     if (pp.prodID % 2 == 0)
                     {
                         pp.setPrice(pp.prodID * 0.9D);
                     } else
                     {
                         pp.setPrice(pp.prodID * 0.71D);
                     }
                     System.out.println(pp.getProdID() + "->price calculate completed.");
                 } catch (InterruptedException e)
                 {
                    // ignore exception
                 } finally
                 {
                     try
                     {
                         // ③ 在此等待其他子线程到达barrier point
                         barrier.await();
                     } catch (InterruptedException
                              | BrokenBarrierException e)
                     {
                     }
                 }
             });
             threadList.add(thread);
             thread.start();
         }
        );
       // ④ 等待所有子任务线程结束
        threadList.forEach(t ->
        {
            try
            {
                t.join();
            } catch (InterruptedException e)
            {
                e.printStackTrace();
            }
        });
        System.out.println("all of prices calculate finished.");
        list.forEach(System.out::println);
}
// ...省略，其余代码与CountDownLatchExample1代码一致
```
虽然同样都是进行子任务并行化的执行并且等待所有子任务结束，但是它们的执行方式却存在着很大的差异。在子任务线程中，当执行结束后调用await方法使当前的子线程进入阻塞状态，直到其他所有的子线程都结束了任务的运行之后，它们才能退出阻塞，下面来解释一下代码注释中几个关键的地方。
- 在注释①处定义了一个CyclicBarrier，虽然要求传入大于0的int数字，但是它所代表的含义是“分片”而不再是计数器，虽然它的作用与计数器几乎类似。
- 在注释②处定义了一个Thread List，用于存放已经被启动的线程，其主要作用就是为了后面等待所有的任务结束而做准备。
- 在注释③处，子任务线程运行（正常/异常）结束后，调用await方法等待其他子线程也运行结束到达一个共同的barrier point，该await方法还会返回一个int的值，该值所代表的意思是当前任务到达的次序（说白了就是这个线程是第几个运行完相关逻辑单元的）。
- 在注释④处，逐一调用每一个子线程的join方法，使当前线程进入阻塞状态等待所有的子线程运行结束。
- 注释④处给出的等待子任务线程运行结束的方案虽然能够达到目的，但是这种方式不太优雅，我们可以通过一个小技巧使代码变得更加简洁。
```java
...省略
List<ProductPrice> list = Arrays.stream(products)
            .mapToObj(ProductPrice::new)
            .collect(toList());

// 在定义CyclicBarrier给定parties时，使parties的数量多一个
final CyclicBarrier barrier = new CyclicBarrier(list.size()+1);
...
// 在主线程中调用await方法，等待其他子任务线程也到达barrier point
barrier.await();
...省略
```
通过为barrier的数量多加一个分片的方式，将主线程也当成子任务线程，这个时候，主线程就可以调用await线程，等待其他线程运行结束并且到达barrier point，进而退出阻塞进入下一个运算逻辑中。

## CyclicBarrier的循环特性
CyclicBarrier的另一个很好的特性是可以被循环使用，也就是说当其内部的计数器为0之后还可以在接下来的使用中重置而无须重新定义一个新的。下面我们看一个简单的例子，想必每个人都是非常喜欢旅游的，旅游的时候不可避免地需要加入某些旅行团。在每一个旅行团中都至少会有一个导游为我们进行向导和解说，由于游客比较多，为了安全考虑导游经常会清点人数以防止个别旅客由于自由活动出现迷路、掉队的情况。
只有在所有的旅客都上了大巴之后司机才能将车开到下一个旅游景点，当大巴到达旅游景点之后，导游还会进行人数清点以确认车上没有旅客由于睡觉而逗留，车才能开去停车场，进而旅客在该景点游玩。由此我们可以看出，所有乘客全部上车和所有乘客在下一个景点全部下车才能开始进一步地统一行动，下面写一个程序简单模拟一下。
```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.TimeUnit;

import static java.util.concurrent.ThreadLocalRandom.current;

public class CyclicBarrierExample2
{
    public static void main(String[] args)
            throws BrokenBarrierException, InterruptedException
    {
        // 定义CyclicBarrier，注意这里的parties值为11
        final CyclicBarrier barrier = new CyclicBarrier(11);
        // 创建10个线程
        for (int i = 0; i < 10; i++)
        {
            // 定义游客线程，传入游客编号和barrier
            new Thread(new Tourist(i, barrier)).start();
        }
       // 主线程也进入阻塞，等待所有游客都上了旅游大巴
        barrier.await();
        System.out.println("Tour Guider:all of Tourist get on the bus.");
        // 主线程进入阻塞，等待所有游客都下了旅游大巴
        barrier.await();
        System.out.println("Tour Guider:all of Tourist get off the bus.");
    }

    private static class Tourist implements Runnable
    {
        private final int touristID;
        private final CyclicBarrier barrier;

        private Tourist(int touristID, CyclicBarrier barrier)
        {
            this.touristID = touristID;
            this.barrier = barrier;
        }

        @Override
        public void run()
        {
            System.out.printf("Tourist:%d by bus\n", touristID);
            // 模拟乘客上车的时间开销
            this.spendSeveralSeconds();
            // 上车后等待其他同伴上车
            this.waitAndPrint("Tourist:%d Get on the bus, and wait other people reached.\n");
            System.out.printf("Tourist:%d arrival the destination\n", touristID);
            // 模拟乘客下车的时间开销
            this.spendSeveralSeconds();
            // 下车后稍作等待，等待其他同伴全部下车
            this.waitAndPrint("Tourist:%d Get off the bus, and wait other people get off.\n");
        }

        private void waitAndPrint(String message)
        {
            System.out.printf(message, touristID);
            try
            {
                barrier.await();
            } catch (InterruptedException | BrokenBarrierException e)
            {
               // ignore
            }
        }

       // random sleep
        private void spendSeveralSeconds()
        {
            try
            {
                TimeUnit.SECONDS.sleep(current().nextInt(10));
            } catch (InterruptedException e)
            {
               // ignore
            }
        }
    }
}
```
在上面的程序中，我们根据前文描述对游客上车后的统一发车，以及到达目的地下车后的统一行动进行了控制。自始至终我们都是使用同一个CyclicBarrier来进行控制的，在这里需要注意的是，在主线程中的两次await中间为何没有对barrier进行reset的操作，那是因为在CyclicBarrier内部维护了一个count。当所有的await调用导致其值为0的时候，reset相关的操作会被默认执行。

## CyclicBarrier源码
下面来看一下CyclicBarrier的await方法调用的相关源码，代码如下。
```java
public int await()
    throws InterruptedException, BrokenBarrierException
{
    ...
        // 所有的await调用，事实上执行的是dowait方法
        return dowait(false, 0L);
    ...
}

private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
...省略
        int index = --count;
       // 当count为0的时候
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                // 生成新的Generation，并且直接返回
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }
...省略
    }
}

private void nextGeneration() {
    // 唤醒阻塞中的所有线程
    trip.signalAll();
    // set up next generation
    // 修改count的值使其等于构造CyclicBarrier转入的parties值
    count = parties;
    // 创建新的Generation
    generation = new Generation();
}
```
通过上面的代码片段，我们可以很清晰地看出，当count的值为0的时候，最后会重新生成新的Generation，并且将count的值设定为构造CyclicBarrier转入的parties值。

那么在调用了reset方法之后呢？我们同样也可以看一下CyclicBarrier reset的源码片段。
```java
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 调用break barrier方法
        breakBarrier();   // break the current generation
        // 重新生成新的generation
        nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}

private void breakBarrier() {
    // generation的broken设置为true，标识该barrier已经被broken了
    generation.broken = true;
    // 重置count的值
    count = parties;
    // 唤醒阻塞的其他线程
    trip.signalAll();
}
```
由于所有的子任务线程都已经顺利完成，虽然在reset方法中调用了breakBarrier方法和唤醒其他新阻塞线程，但是它们都会被忽略掉，根本不会影响到dowait方法中的线程（因为执行该方法的线程已经没有了），紧接着generation又会被重新创建，因此在本节的例子中，主线程的两次await方法调用之间完全可以不用调用reset方法，当然你加入了reset方法也不会有什么影响。

## CyclicBarrier总结
通过前面两个章节的学习，读者应该已经掌握了CyclicBarrier的基本用法，当然它还提供了一些其他的方法和构造方式。
- CyclicBarrier(int parties)构造器：构造CyclicBarrier并且传入parties。
- CyclicBarrier(int parties, Runnable barrierAction)构造器：构造CyclicBarrier不仅传入parties，而且指定一个Runnable接口，当所有的线程到达barrier point的时候，该Runnable接口会被调用，有时我们需要在所有任务执行结束之后执行某个动作，这时就可以使用这种CyclicBarrier的构造方式了。
- int getParties()方法：获取CyclicBarrier在构造时的parties，该值一经CyclicBarrier创建将不会被改变。
- await()方法：我们使用最多的一个方法，调用该方法之后，当前线程将会进入阻塞状态，等待其他线程执行await()方法进入barrier point，进而全部退出阻塞状态，当CyclicBarrier内部的count为0时，调用await()方法将会直接返回而不会进入阻塞状态。
```java
final CyclicBarrier barrier = new CyclicBarrier(1);
barrier.await(); // barrier的count为0
barrier.await(); // 直接返回
barrier.await(); // 直接返回
```
- await(long timeout, TimeUnit unit)方法：该方法与无参的await方法类似，只不过增加了超时的功能，当其他线程在设定的时间内没有到达barrier point时，当前线程也会退出阻塞状态。
- isBroken()：返回barrier的broken状态，某个线程由于执行await方法而进入阻塞状态，如果该线程被执行了中断操作，那么isBroken()方法将会返回true。
```java
final CyclicBarrier barrier = new CyclicBarrier(2);
Thread thread = new Thread(() ->
{
    try
    {
        // thread将会进入阻塞状态
        barrier.await();
    } catch (InterruptedException | BrokenBarrierException e)
    {
        e.printStackTrace();
    }
});
thread.start();
// 两秒后在main线程中执行thread的中断操作
TimeUnit.SECONDS.sleep(2);
// 调用中断
thread.interrupt();
// 短暂休眠，确保thread的执行动作发生在main线程读取broken状态之前
TimeUnit.SECONDS.sleep(2);
// 输出barrier的broken状态，这种情况下该返回值肯定为true
System.out.println(barrier.isBroken());
```
当一个线程在执行CyclicBarrier的await方法进入阻塞而被中断时，CyclicBarrier会被broken这一点我们已经通过上面的代码证明过了，但是需要注意如下几点（非常重要）。
 1）当一个线程由于在执行CyclicBarrier的await方法而进入阻塞状态时，这个时候对该线程执行中断操作会导致CyclicBarrier被broken。
 2）被broken的CyclicBarrier此时已经不能再直接使用了，如果想要使用就必须使用reset方法对其重置。
 3）如果有其他线程此时也由于执行了await方法而进入阻塞状态，那么该线程会被唤醒并且抛出BrokenBarrierException异常。
- getNumberWaiting()方法： 该方法返回当前barrier有多少个线程执行了await方法而不是还有多少个线程未到达barrier point，这一点需要注意。
- reset()方法：前面已经详细地介绍过这个方法，其主要作用是中断当前barrier，并且重新生成一个generation，还有将barrier内部的计数器count设置为parties值，但是需要注意的是，如果还有未到达barrier point的线程，则所有的线程将会被中断并且退出阻塞，此时isBroken()方法将返回false而不是true。
```java
final CyclicBarrier barrier = new CyclicBarrier(3);

Thread thread = new Thread(() ->
{
    try
    {
        barrier.await();
    } catch (InterruptedException | BrokenBarrierException e)
    {
        e.printStackTrace();
    }
});
thread.start();
TimeUnit.SECONDS.sleep(2);
// 执行reset方法，thread线程将会被中断
barrier.reset();
// 此时isBroken()为false而不是true
assert !barrier.isBroken() : "broken state must false.";
```

## CyclicBarrier VS CountDownLatch
- CoundDownLatch的await方法会等待计数器被count down到0，而执行CyclicBarrier的await方法的线程将会等待其他线程到达barrier point。
- CyclicBarrier内部的计数器count是可被重置的，进而使得CyclicBarrier也可被重复使用，而CoundDownLatch则不能。
- CyclicBarrier是由Lock和Condition实现的，而CountDownLatch则是由同步控制器AQS（AbstractQueuedSynchronizer）来实现的。
- 在构造CyclicBarrier时不允许parties为0，而CountDownLatch则允许count为0。










