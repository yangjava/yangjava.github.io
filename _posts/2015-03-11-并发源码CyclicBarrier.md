---
layout: post
categories: [JUC]
description: none
keywords: JUC
---
## 公平锁和可重入锁的原理

### 原理介绍

最经典的分布式锁是可重入的公平锁。什么是可重入的公平锁呢？直接讲解的概念和原理，会比较抽象难懂，还是从具体的实例入手吧！这里用一个简单的故事来类比，估计就简单多了。

故事发生在一个没有自来水的古代，在一个村子有一口井，水质非常的好，村民们都抢着取井里的水。井就那么一口，村里的人很多，村民为争抢取水打架斗殴，甚至头破血流。

问题总是要解决，于是村长绞尽脑汁，最终想出了一个凭号取水的方案。井边安排一个看井人，维护取水的秩序。取水秩序很简单：

（1）取水之前，先取号；

（2）号排在前面的，就可以先取水；

（3）先到的排在前面，那些后到的，一个一个挨着，在井边排成一队。

这种排队取水模型，就是一种锁的模型。排在最前面的号，拥有取水权，就是一种典型的独占锁。另外，先到先得，号排在前面的人先取到水，取水之后就轮到下一个号取水，挺公平的，说明它是一种公平锁。

什么是可重入锁呢？
假定，取水时以家庭为单位，家庭的某人拿到号，其他的家庭成员过来打水，这时候不用再取号

排在1号的家庭，老公取号，假设其老婆来了，直接排第一个，正所谓妻凭夫贵。再看上图的2号，父亲正在打水，假设其儿子和女儿也到井边了，直接排第二个，所谓子凭父贵。总之，如果取水时以家庭为单位，则同一个家庭，可以直接复用排号，不用从后面排起重新取号。

以上这个故事模型中，取号一次，可以用来多次取水，其原理为可重入锁的模型。在重入锁模型中，一把独占锁，可以被多次锁定，这就叫做可重入锁。

## ReentrantLock重入锁

### synchronized的局限性

synchronized是java内置的关键字，它提供了一种独占的加锁方式。synchronized的获取和释放锁由jvm实现，用户不需要显示的释放锁，非常方便，然而synchronized也有一定的局限性，例如：

1. 当线程尝试获取锁的时候，如果获取不到锁会一直阻塞，这个阻塞的过程，用户无法控制
2. 如果获取锁的线程进入休眠或者阻塞，除非当前线程异常，否则其他线程尝试获取锁必须一直等待

JDK1.5之后发布，加入了Doug Lea实现的java.util.concurrent包。包内提供了Lock类，用来提供更多扩展的加锁功能。Lock弥补了synchronized的局限，提供了更加细粒度的加锁功能。

### ReentrantLock

ReentrantLock是Lock的默认实现，在聊ReentranLock之前，我们需要先弄清楚一些概念：

1. 可重入锁：可重入锁是指同一个线程可以多次获得同一把锁；ReentrantLock和关键字Synchronized都是可重入锁
2. 可中断锁：可中断锁时只线程在获取锁的过程中，是否可以相应线程中断操作。synchronized是不可中断的，ReentrantLock是可中断的
3. 公平锁和非公平锁：公平锁是指多个线程尝试获取同一把锁的时候，获取锁的顺序按照线程到达的先后顺序获取，而不是随机插队的方式获取。synchronized是非公平锁，而ReentrantLock是两种都可以实现，不过默认是非公平锁

### ReentrantLock基本使用

我们使用3个线程来对一个共享变量++操作，先使用**synchronized**实现，然后使用**ReentrantLock**实现。

**synchronized方式**：

```java
package com.itsoku.chat06;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo2 {

    private static int num = 0;

    private static synchronized void add() {
        num++;
    }

    public static class T extends Thread {
        @Override
        public void run() {
            for (int i = 0; i < 10000; i++) {
                Demo2.add();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        T t1 = new T();
        T t2 = new T();
        T t3 = new T();

        t1.start();
        t2.start();
        t3.start();

        t1.join();
        t2.join();
        t3.join();

        System.out.println(Demo2.num);
    }
}
```

ReentrantLock方式：

```java
package com.itsoku.chat06;

import java.util.concurrent.locks.ReentrantLock;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo3 {

    private static int num = 0;

    private static ReentrantLock lock = new ReentrantLock();

    private static void add() {
        lock.lock();
        try {
            num++;
        } finally {
            lock.unlock();
        }

    }

    public static class T extends Thread {
        @Override
        public void run() {
            for (int i = 0; i < 10000; i++) {
                Demo3.add();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        T t1 = new T();
        T t2 = new T();
        T t3 = new T();

        t1.start();
        t2.start();
        t3.start();

        t1.join();
        t2.join();
        t3.join();

        System.out.println(Demo3.num);
    }
}
```

**ReentrantLock的使用过程：**

1. **创建锁：ReentrantLock lock = new ReentrantLock();**
2. **获取锁：lock.lock()**
3. **释放锁：lock.unlock();**

对比上面的代码，与关键字synchronized相比，ReentrantLock锁有明显的操作过程，开发人员必须手动的指定何时加锁，何时释放锁，正是因为这样手动控制，ReentrantLock对逻辑控制的灵活度要远远胜于关键字synchronized，上面代码需要注意**lock.unlock()**一定要放在finally中，否则，若程序出现了异常，锁没有释放，那么其他线程就再也没有机会获取这个锁了。

### ReentrantLock是可重入锁

来验证一下ReentrantLock是可重入锁，实例代码：

```java
package com.itsoku.chat06;

import java.util.concurrent.locks.ReentrantLock;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo4 {
    private static int num = 0;
    private static ReentrantLock lock = new ReentrantLock();

    private static void add() {
        lock.lock();
        lock.lock();
        try {
            num++;
        } finally {
            lock.unlock();
            lock.unlock();
        }
    }

    public static class T extends Thread {
        @Override
        public void run() {
            for (int i = 0; i < 10000; i++) {
                Demo4.add();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        T t1 = new T();
        T t2 = new T();
        T t3 = new T();

        t1.start();
        t2.start();
        t3.start();

        t1.join();
        t2.join();
        t3.join();

        System.out.println(Demo4.num);
    }
}
```

上面代码中add()方法中，当一个线程进入的时候，会执行2次获取锁的操作，运行程序可以正常结束，并输出和期望值一样的30000，假如ReentrantLock是不可重入的锁，那么同一个线程第2次获取锁的时候由于前面的锁还未释放而导致死锁，程序是无法正常结束的。ReentrantLock命名也挺好的Reentrant Lock，和其名字一样，可重入锁。

代码中还有几点需要注意：

1. **lock()方法和unlock()方法需要成对出现，锁了几次，也要释放几次，否则后面的线程无法获取锁了；可以将add中的unlock删除一个事实，上面代码运行将无法结束**
2. **unlock()方法放在finally中执行，保证不管程序是否有异常，锁必定会释放**

## ReentrantLock实现公平锁

在大多数情况下，锁的申请都是非公平的，也就是说，线程1首先请求锁A，接着线程2也请求了锁A。那么当锁A可用时，是线程1可获得锁还是线程2可获得锁呢？这是不一定的，系统只是会从这个锁的等待队列中随机挑选一个，因此不能保证其公平性。这就好比买票不排队，大家都围在售票窗口前，售票员忙的焦头烂额，也顾及不上谁先谁后，随便找个人出票就完事了，最终导致的结果是，有些人可能一直买不到票。而公平锁，则不是这样，它会按照到达的先后顺序获得资源。公平锁的一大特点是：它不会产生饥饿现象，只要你排队，最终还是可以等到资源的；synchronized关键字默认是有jvm内部实现控制的，是非公平锁。而ReentrantLock运行开发者自己设置锁的公平性。

看一下jdk中ReentrantLock的源码，2个构造方法：

```java
public ReentrantLock() {
	sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
	sync = fair ? new FairSync() : new NonfairSync();
}
```

默认构造方法创建的是非公平锁。

第2个构造方法，有个fair参数，当fair为true的时候创建的是公平锁，公平锁看起来很不错，不过要实现公平锁，系统内部肯定需要维护一个有序队列，因此公平锁的实现成本比较高，性能相对于非公平锁来说相对低一些。因此，在默认情况下，锁是非公平的，如果没有特别要求，则不建议使用公平锁。

公平锁和非公平锁在程序调度上是很不一样，来一个公平锁示例看一下：

```java
package com.itsoku.chat06;

import java.util.concurrent.locks.ReentrantLock;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo5 {
    private static int num = 0;
    private static ReentrantLock fairLock = new ReentrantLock(true);

    public static class T extends Thread {
        public T(String name) {
            super(name);
        }

        @Override
        public void run() {
            for (int i = 0; i < 5; i++) {
                fairLock.lock();
                try {
                    System.out.println(this.getName() + "获得锁!");
                } finally {
                    fairLock.unlock();
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        T t1 = new T("t1");
        T t2 = new T("t2");
        T t3 = new T("t3");

        t1.start();
        t2.start();
        t3.start();

        t1.join();
        t2.join();
        t3.join();
    }
}
```

运行结果输出：

```txt
t1获得锁!
t2获得锁!
t3获得锁!
t1获得锁!
t2获得锁!
t3获得锁!
t1获得锁!
t2获得锁!
t3获得锁!
t1获得锁!
t2获得锁!
t3获得锁!
t1获得锁!
t2获得锁!
t3获得锁!
```

看一下输出的结果，锁时按照先后顺序获得的。

修改一下上面代码，改为非公平锁试试，如下：

```java
ReentrantLock fairLock = new ReentrantLock(false);
```

运行结果如下：

```java
t1获得锁!
t3获得锁!
t3获得锁!
t3获得锁!
t3获得锁!
t1获得锁!
t1获得锁!
t1获得锁!
t1获得锁!
t2获得锁!
t2获得锁!
t2获得锁!
t2获得锁!
t2获得锁!
t3获得锁!
```

可以看到t3可能会连续获得锁，结果是比较随机的，不公平的。

### ReentrantLock获取锁的过程是可中断的

对于synchronized关键字，如果一个线程在等待获取锁，最终只有2种结果：

1. 要么获取到锁然后继续后面的操作
2. 要么一直等待，直到其他线程释放锁为止

而ReentrantLock提供了另外一种可能，就是在等的获取锁的过程中（**发起获取锁请求到还未获取到锁这段时间内**）是可以被中断的，也就是说在等待锁的过程中，程序可以根据需要取消获取锁的请求。有些使用这个操作是非常有必要的。比如：你和好朋友越好一起去打球，如果你等了半小时朋友还没到，突然你接到一个电话，朋友由于突发状况，不能来了，那么你一定达到回府。中断操作正是提供了一套类似的机制，如果一个线程正在等待获取锁，那么它依然可以收到一个通知，被告知无需等待，可以停止工作了。

示例代码：

```java
package com.itsoku.chat06;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo6 {
    private static ReentrantLock lock1 = new ReentrantLock(false);
    private static ReentrantLock lock2 = new ReentrantLock(false);

    public static class T extends Thread {
        int lock;

        public T(String name, int lock) {
            super(name);
            this.lock = lock;
        }

        @Override
        public void run() {
            try {
                if (this.lock == 1) {
                    lock1.lockInterruptibly();
                    TimeUnit.SECONDS.sleep(1);
                    lock2.lockInterruptibly();
                } else {
                    lock2.lockInterruptibly();
                    TimeUnit.SECONDS.sleep(1);
                    lock1.lockInterruptibly();
                }
            } catch (InterruptedException e) {
                System.out.println("中断标志:" + this.isInterrupted());
                e.printStackTrace();
            } finally {
                if (lock1.isHeldByCurrentThread()) {
                    lock1.unlock();
                }
                if (lock2.isHeldByCurrentThread()) {
                    lock2.unlock();
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        T t1 = new T("t1", 1);
        T t2 = new T("t2", 2);

        t1.start();
        t2.start();
    }
}
```

先运行一下上面代码，发现程序无法结束，使用jstack查看线程堆栈信息，发现2个线程死锁了。

```java
Found one Java-level deadlock:
=============================
"t2":
  waiting for ownable synchronizer 0x0000000717380c20, (a java.util.concurrent.locks.ReentrantLock$NonfairSync),
  which is held by "t1"
"t1":
  waiting for ownable synchronizer 0x0000000717380c50, (a java.util.concurrent.locks.ReentrantLock$NonfairSync),
  which is held by "t2"
```

lock1被线程t1占用，lock2倍线程t2占用，线程t1在等待获取lock2，线程t2在等待获取lock1，都在相互等待获取对方持有的锁，最终产生了死锁，如果是在synchronized关键字情况下发生了死锁现象，程序是无法结束的。

我们队上面代码改造一下，线程t2一直无法获取到lock1，那么等待5秒之后，我们中断获取锁的操作。主要修改一下main方法，如下：

```java
T t1 = new T("t1", 1);
T t2 = new T("t2", 2);

t1.start();
t2.start();

TimeUnit.SECONDS.sleep(5);
t2.interrupt();
```

新增了2行代码`TimeUnit.SECONDS.sleep(5);t2.interrupt();`，程序可以结束了，运行结果：

```java
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
	at com.itsoku.chat06.Demo6$T.run(Demo6.java:31)
中断标志:false
```

从上面信息中可以看出，代码的31行触发了异常，**中断标志输出：false**

![img](https://img2018.cnblogs.com/blog/687624/201907/687624-20190717191731394-781205156.png)

t2在31行一直获取不到lock1的锁，主线程中等待了5秒之后，t2线程调用了`interrupt()`方法，将线程的中断标志置为true，此时31行会触发`InterruptedException`异常，然后线程t2可以继续向下执行，释放了lock2的锁，然后线程t1可以正常获取锁，程序得以继续进行。线程发送中断信号触发InterruptedException异常之后，中断标志将被清空。

关于获取锁的过程中被中断，注意几点:

1. **ReentrankLock中必须使用实例方法`lockInterruptibly()`获取锁时，在线程调用interrupt()方法之后，才会引发`InterruptedException`异常**
2. **线程调用interrupt()之后，线程的中断标志会被置为true**
3. **触发InterruptedException异常之后，线程的中断标志有会被清空，即置为false**
4. **所以当线程调用interrupt()引发InterruptedException异常，中断标志的变化是:false->true->false**

### ReentrantLock锁申请等待限时

申请锁等待限时是什么意思？一般情况下，获取锁的时间我们是不知道的，synchronized关键字获取锁的过程中，只能等待其他线程把锁释放之后才能够有机会获取到锁。所以获取锁的时间有长有短。如果获取锁的时间能够设置超时时间，那就非常好了。

ReentrantLock刚好提供了这样功能，给我们提供了获取锁限时等待的方法`tryLock()`，可以选择传入时间参数，表示等待指定的时间，无参则表示立即返回锁申请的结果：true表示获取锁成功，false表示获取锁失败。

### tryLock无参方法

看一下源码中tryLock方法：

```java
public boolean tryLock()
```

返回boolean类型的值，此方法会立即返回，结果表示获取锁是否成功，示例：

```java
package com.itsoku.chat06;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo8 {
    private static ReentrantLock lock1 = new ReentrantLock(false);

    public static class T extends Thread {

        public T(String name) {
            super(name);
        }

        @Override
        public void run() {
            try {
                System.out.println(System.currentTimeMillis() + ":" + this.getName() + "开始获取锁!");
                //获取锁超时时间设置为3秒，3秒内是否能否获取锁都会返回
                if (lock1.tryLock()) {
                    System.out.println(System.currentTimeMillis() + ":" + this.getName() + "获取到了锁!");
                    //获取到锁之后，休眠5秒
                    TimeUnit.SECONDS.sleep(5);
                } else {
                    System.out.println(System.currentTimeMillis() + ":" + this.getName() + "未能获取到锁!");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                if (lock1.isHeldByCurrentThread()) {
                    lock1.unlock();
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        T t1 = new T("t1");
        T t2 = new T("t2");

        t1.start();
        t2.start();
    }
}
```

代码中获取锁成功之后，休眠5秒，会导致另外一个线程获取锁失败，运行代码，输出：

```java
1563356291081:t2开始获取锁!
1563356291081:t2获取到了锁!
1563356291081:t1开始获取锁!
1563356291081:t1未能获取到锁!
```

可以看到t2获取成功，t1获取失败了，tryLock()是立即响应的，中间不会有阻塞。

### tryLock有参方法

可以明确设置获取锁的超时时间，该方法签名：

```java
public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException
```

该方法在指定的时间内不管是否可以获取锁，都会返回结果，返回true，表示获取锁成功，返回false表示获取失败。此方法由2个参数，第一个参数是时间类型，是一个枚举，可以表示时、分、秒、毫秒等待，使用比较方便，第1个参数表示在时间类型上的时间长短。此方法在执行的过程中，如果调用了线程的中断interrupt()方法，会触发InterruptedException异常。

示例：

```java
package com.itsoku.chat06;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo7 {
    private static ReentrantLock lock1 = new ReentrantLock(false);

    public static class T extends Thread {

        public T(String name) {
            super(name);
        }

        @Override
        public void run() {
            try {
                System.out.println(System.currentTimeMillis() + ":" + this.getName() + "开始获取锁!");
                //获取锁超时时间设置为3秒，3秒内是否能否获取锁都会返回
                if (lock1.tryLock(3, TimeUnit.SECONDS)) {
                    System.out.println(System.currentTimeMillis() + ":" + this.getName() + "获取到了锁!");
                    //获取到锁之后，休眠5秒
                    TimeUnit.SECONDS.sleep(5);
                } else {
                    System.out.println(System.currentTimeMillis() + ":" + this.getName() + "未能获取到锁!");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                if (lock1.isHeldByCurrentThread()) {
                    lock1.unlock();
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        T t1 = new T("t1");
        T t2 = new T("t2");

        t1.start();
        t2.start();
    }
}
```

程序中调用了ReentrantLock的实例方法`tryLock(3, TimeUnit.SECONDS)`，表示获取锁的超时时间是3秒，3秒后不管是否能否获取锁，该方法都会有返回值，获取到锁之后，内部休眠了5秒，会导致另外一个线程获取锁失败。

运行程序，输出：

```java
1563355512901:t2开始获取锁!
1563355512901:t1开始获取锁!
1563355512902:t2获取到了锁!
1563355515904:t1未能获取到锁!
```

输出结果中分析，t2获取到锁了，然后休眠了5秒，t1获取锁失败，t1打印了2条信息，时间相差3秒左右。

**关于tryLock()方法和tryLock(long timeout, TimeUnit unit)方法，说明一下：**

1. 都会返回boolean值，结果表示获取锁是否成功
2. tryLock()方法，不管是否获取成功，都会立即返回；而有参的tryLock方法会尝试在指定的时间内去获取锁，中间会阻塞的现象，在指定的时间之后会不管是否能够获取锁都会返回结果
3. tryLock()方法不会响应线程的中断方法；而有参的tryLock方法会响应线程的中断方法，而出发`InterruptedException`异常，这个从2个方法的声明上可以可以看出来

## ReentrantLock其他常用的方法

1. isHeldByCurrentThread：实例方法，判断当前线程是否持有ReentrantLock的锁，上面代码中有使用过。

## 获取锁的4中方法对比

| 获取锁的方法                         | 是否立即响应(不会阻塞) | 是否响应中断 |
| ------------------------------------ | ---------------------- | ------------ |
| lock()                               | ×                      | ×            |
| lockInterruptibly()                  | ×                      | √            |
| tryLock()                            | √                      | ×            |
| tryLock(long timeout, TimeUnit unit) | ×                      | √            |

## 总结

1. ReentrantLock可以实现公平锁和非公平锁
2. ReentrantLock默认实现的是非公平锁
3. ReentrantLock的获取锁和释放锁必须成对出现，锁了几次，也要释放几次
4. 释放锁的操作必须放在finally中执行
5. lockInterruptibly()实例方法可以相应线程的中断方法，调用线程的interrupt()方法时，lockInterruptibly()方法会触发`InterruptedException`异常
6. 关于`InterruptedException`异常说一下，看到方法声明上带有 `throws InterruptedException`，表示该方法可以相应线程中断，调用线程的interrupt()方法时，这些方法会触发`InterruptedException`异常，触发InterruptedException时，线程的中断中断状态会被清除。所以如果程序由于调用`interrupt()`方法而触发`InterruptedException`异常，线程的标志由默认的false变为ture，然后又变为false
7. 实例方法tryLock()获会尝试获取锁，会立即返回，返回值表示是否获取成功
8. 实例方法tryLock(long timeout, TimeUnit unit)会在指定的时间内尝试获取锁，指定的时间内是否能够获取锁，都会返回，返回值表示是否获取锁成功，该方法会响应线程的中断

# Condition条件对象

本文目标：

1. synchronized中实现线程等待和唤醒
2. Condition简介及常用方法介绍及相关示例
3. 使用Condition实现生产者消费者
4. 使用Condition实现同步阻塞队列

## synchronized中等待和唤醒线程示例

```java
package com.itsoku.chat09;

import java.util.concurrent.TimeUnit;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo1 {
    static Object lock = new Object();

    public static class T1 extends Thread {
        @Override
        public void run() {
            System.out.println(System.currentTimeMillis() + "," + this.getName() + "准备获取锁!");
            synchronized (lock) {
                System.out.println(System.currentTimeMillis() + "," + this.getName() + "获取锁成功!");
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            System.out.println(System.currentTimeMillis() + "," + this.getName() + "释放锁成功!");
        }
    }

    public static class T2 extends Thread {
        @Override
        public void run() {
            System.out.println(System.currentTimeMillis() + "," + this.getName() + "准备获取锁!");
            synchronized (lock) {
                System.out.println(System.currentTimeMillis() + "," + this.getName() + "获取锁成功!");
                lock.notify();
                System.out.println(System.currentTimeMillis() + "," + this.getName() + " notify!");
                try {
                    TimeUnit.SECONDS.sleep(5);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(System.currentTimeMillis() + "," + this.getName() + "准备释放锁!");
            }
            System.out.println(System.currentTimeMillis() + "," + this.getName() + "释放锁成功!");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        T1 t1 = new T1();
        t1.setName("t1");
        t1.start();
        TimeUnit.SECONDS.sleep(5);
        T2 t2 = new T2();
        t2.setName("t2");
        t2.start();
    }
}
```

输出：

```txt
1：1563530109234,t1准备获取锁!
2：1563530109234,t1获取锁成功!
3：1563530114236,t2准备获取锁!
4：1563530114236,t2获取锁成功!
5：1563530114236,t2 notify!
6：1563530119237,t2准备释放锁!
7：1563530119237,t2释放锁成功!
8：1563530119237,t1释放锁成功!
```

代码结合输出的结果我们分析一下：

1. 线程t1先获取锁，然后调用了wait()方法将线程置为等待状态，然后会释放lock的锁
2. 主线程等待5秒之后，启动线程t2，t2获取到了锁，结果中1、3行时间相差5秒左右
3. t2调用lock.notify()方法，准备将等待在lock上的线程t1唤醒，notify()方法之后又休眠了5秒，看一下输出的5、8可知，notify()方法之后，t1并不能立即被唤醒，需要等到t2将synchronized块执行完毕，释放锁之后，t1才被唤醒
4. wait()方法和notify()方法必须放在同步块内调用（synchronized块内），否则会报错