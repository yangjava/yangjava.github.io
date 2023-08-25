---
layout: post
categories: [JUC]
description: none
keywords: JUC
---
# Java并发编程synchronized


### 什么是线程安全？

如果你的代码所在的进程中有多个线程在同时运行，而这些线程可能会同时运行这段代码。如果每次运行结果和单线程运行的结果是一样的，而且其他的变量的值也和预期的是一样的，就是线程安全的。

或者说当多个线程去访问同一个类（对象或方法）的时候，该类都能表现出正常的行为（与自己预想的结果一致），那我们就可以所这个类是线程安全的。

线程安全问题都是由全局变量及静态变量引起的。
若每个线程中对全局变量、静态变量只有读操作，而无写操作，一般来说，这个全局变量是线程安全的；若有多个线程同时执行写操作，一般都需要考虑线程同步，否则就可能影响线程安全。

看一段代码：

```java
package com.itsoku.chat04;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo1 {
    static int num = 0;

    public static void m1() {
        for (int i = 0; i < 10000; i++) {
            num++;
        }
    }

    public static class T1 extends Thread {
        @Override
        public void run() {
            Demo1.m1();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        T1 t1 = new T1();
        T1 t2 = new T1();
        T1 t3 = new T1();
        t1.start();
        t2.start();
        t3.start();

        //等待3个线程结束打印num
        t1.join();
        t2.join();
        t3.join();

        System.out.println(Demo1.num);
        /**
         * 打印结果：
         * 25572
         */
    }
}
```

Demo1中有个静态变量num，默认值是0，m1()方法中对num++执行10000次，main方法中创建了3个线程用来调用m1()方法，然后调用3个线程的join()方法，用来等待3个线程执行完毕之后，打印num的值。我们期望的结果是30000，运行一下，但真实的结果却不是30000。上面的程序在多线程中表现出来的结果和预想的结果不一致，说明上面的程序不是线程安全的。

线程安全是并发编程中的重要关注点，应该注意到的是，造成线程安全问题的主要诱因有两点：

1. 一是存在共享数据(也称临界资源)
2. 二是存在多条线程共同操作共享数据

因此为了解决这个问题，我们可能需要这样一个方案，当存在多个线程操作共享数据时，**需要保证同一时刻有且只有一个线程在操作共享数据**，其他线程必须等到该线程处理完数据后再进行，这种方式有个高尚的名称叫**互斥锁**，即能达到互斥访问目的的锁，也就是说当一个共享数据被当前正在访问的线程加上互斥锁后，在同一个时刻，其他线程只能处于等待的状态，直到当前线程处理完毕释放该锁。在 Java 中，**关键字 synchronized可以保证在同一个时刻，只有一个线程可以执行某个方法或者某个代码块(主要是对方法或者代码块中存在共享数据的操作)**，**同时我们还应该注意到synchronized另外一个重要的作用，synchronized可保证一个线程的变化(主要是共享数据的变化)被其他线程所看到（保证可见性，完全可以替代volatile功能）**，这点确实也是很重要的。

那么我们把上面的程序做一下调整，在m1()方法上面使用关键字synchronized，如下：

```java
public static synchronized void m1() {
    for (int i = 0; i < 10000; i++) {
        num++;
    }
}
```

然后执行代码，输出30000，和期望结果一致。

### synchronized使用方式

1. 修饰实例方法，作用于当前实例，进入同步代码前需要先获取实例的锁
2. 修饰静态方法，作用于类的Class对象，进入修饰的静态方法前需要先获取类的Class对象的锁
3. 修饰代码块，需要指定加锁对象(记做lockobj)，在进入同步代码块前需要先获取lockobj的锁

#### synchronized作用于实例对象

所谓实例对象锁就是用synchronized修饰实例对象的实例方法，注意是**实例方法**，不是**静态方法**，如：

```java
package com.itsoku.chat04;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo2 {
    int num = 0;

    public synchronized void add() {
        num++;
    }

    public static class T extends Thread {
        private Demo2 demo2;

        public T(Demo2 demo2) {
            this.demo2 = demo2;
        }

        @Override
        public void run() {
            for (int i = 0; i < 10000; i++) {
                this.demo2.add();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Demo2 demo2 = new Demo2();
        T t1 = new T(demo2);
        T t2 = new T(demo2);
        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println(demo2.num);
    }
}
```

main()方法中创建了一个对象demo2和2个线程t1、t2，t1、t2中调用demo2的add()方法10000次，add()方法中执行了num++，num++实际上是分3步，获取num，然后将num+1，然后将结果赋值给num，如果t2在t1读取num和num+1之间获取了num的值，那么t1和t2会读取到同样的值，然后执行num++，两次操作之后num是相同的值，最终和期望的结果不一致，造成了线程安全失败，因此我们对add方法加了synchronized来保证线程安全。

注意：m1()方法是实例方法，两个线程操作m1()时，需要先获取demo2的锁，没有获取到锁的，将等待，直到其他线程释放锁为止。

synchronize作用于实例方法需要注意：

1. 实例方法上加synchronized，线程安全的前提是，多个线程操作的是**同一个实例**，如果多个线程作用于不同的实例，那么线程安全是无法保证的
2. 同一个实例的多个实例方法上有synchronized，这些方法都是互斥的，同一时间只允许一个线程操作**同一个实例的其中的一个synchronized方法**

#### synchronized作用于静态方法

当synchronized作用于静态方法时，锁的对象就是当前类的Class对象。如：

```java
package com.itsoku.chat04;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo3 {
    static int num = 0;

    public static synchronized void m1() {
        for (int i = 0; i < 10000; i++) {
            num++;
        }
    }

    public static class T1 extends Thread {
        @Override
        public void run() {
            Demo3.m1();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        T1 t1 = new T1();
        T1 t2 = new T1();
        T1 t3 = new T1();
        t1.start();
        t2.start();
        t3.start();

        //等待3个线程结束打印num
        t1.join();
        t2.join();
        t3.join();

        System.out.println(Demo3.num);
        /**
         * 打印结果：
         * 30000
         */
    }
}
```

上面代码打印30000，和期望结果一致。m1()方法是静态方法，有synchronized修饰，锁用于与Demo3.class对象，和下面的写法类似：

```java
public static void m1() {
    synchronized (Demo4.class) {
        for (int i = 0; i < 10000; i++) {
            num++;
        }
    }
}
```

#### synchronized同步代码块

除了使用关键字修饰实例方法和静态方法外，还可以使用同步代码块，在某些情况下，我们编写的方法体可能比较大，同时存在一些比较耗时的操作，而需要同步的代码又只有一小部分，如果直接对整个方法进行同步操作，可能会得不偿失，此时我们可以使用同步代码块的方式对需要同步的代码进行包裹，这样就无需对整个方法进行同步操作了，同步代码块的使用示例如下：

```java
package com.itsoku.chat04;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo5 implements Runnable {
    static Demo5 instance = new Demo5();
    static int i = 0;

    @Override
    public void run() {
        //省略其他耗时操作....
        //使用同步代码块对变量i进行同步操作,锁对象为instance
        synchronized (instance) {
            for (int j = 0; j < 10000; j++) {
                i++;
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(instance);
        Thread t2 = new Thread(instance);
        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println(i);
    }
}
```

从代码看出，将synchronized作用于一个给定的实例对象instance，即当前实例对象就是锁对象，每次当线程进入synchronized包裹的代码块时就会要求当前线程持有instance实例对象锁，如果当前有其他线程正持有该对象锁，那么新到的线程就必须等待，这样也就保证了每次只有一个线程执行i++;操作。当然除了instance作为对象外，我们还可以使用this对象(代表当前实例)或者当前类的class对象作为锁，如下代码：

```java
//this,当前实例对象锁
synchronized(this){
    for(int j=0;j<1000000;j++){
        i++;
    }
}

//class对象锁
synchronized(Demo5.class){
    for(int j=0;j<1000000;j++){
        i++;
    }
}
```

分析代码是否互斥的方法，先找出synchronized作用的对象是谁，如果多个线程操作的方法中synchronized作用的锁对象一样，那么这些线程同时异步执行这些方法就是互斥的。如下代码:

```java
package com.itsoku.chat04;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo6 {
    //作用于当前类的实例对象
    public synchronized void m1() {
    }

    //作用于当前类的实例对象
    public synchronized void m2() {
    }

    //作用于当前类的实例对象
    public void m3() {
        synchronized (this) {
        }
    }

    //作用于当前类Class对象
    public static synchronized void m4() {
    }

    //作用于当前类Class对象
    public static void m5() {
        synchronized (Demo6.class) {
        }
    }

    public static class T extends Thread{
        Demo6 demo6;

        public T(Demo6 demo6) {
            this.demo6 = demo6;
        }

        @Override
        public void run() {
            super.run();
        }
    }

    public static void main(String[] args) {
        Demo6 d1 = new Demo6();
        Thread t1 = new Thread(() -> {
            d1.m1();
        });
        t1.start();
        Thread t2 = new Thread(() -> {
            d1.m2();
        });
        t2.start();

        Thread t3 = new Thread(() -> {
            d1.m2();
        });
        t3.start();

        Demo6 d2 = new Demo6();
        Thread t4 = new Thread(() -> {
            d2.m2();
        });
        t4.start();

        Thread t5 = new Thread(() -> {
            Demo6.m4();
        });
        t5.start();

        Thread t6 = new Thread(() -> {
            Demo6.m5();
        });
        t6.start();
    }

}
```

分析上面代码：

1. 线程t1、t2、t3中调用的方法都需要获取d1的锁，所以他们是互斥的
2. t1/t2/t3这3个线程和t4不互斥，他们可以同时运行，因为前面三个线程依赖于d1的锁，t4依赖于d2的锁
3. t5、t6都作用于当前类的Class对象锁，所以这两个线程是互斥的，和其他几个线程不互斥

### synchronized可以确保变量的可见性

synchronized除了用于线程同步、确保线程安全外，还可以保证线程间的可见性和有序性。从可见性的角度上将，关键字synchronized可以完全替代关键字volatile的功能，只是使用上没有那么方便。就有序性而言，由于关键字synchronized限制每次只有一个线程可以访问同步块，因此，无论同步块内的代码如何被乱序执行，只要保证串行语义一致，那么执行结果总是一样的。而其他访问线程，又必须在获得锁后方能进入代码块读取数据，因此，他们看到的最终结果并不取决于代码的执行过程，有序性问题自然得到了解决（换言之，被关键字synchronized限制的多个线程是串行执行的）。

线程进入synchronized修饰的代码中时，synchronized代码块内部使用到的共享变量在当前线程的工作内存中都会被清空，会从主内存中获取，当synchronized代码块结束的时候，代码块内部修改的共享变量都会强制刷新到主存储中，所以是可见的。

关于synchronized可以保证可见性的，上个例子：

```java
package com.itsoku.chat05;

import java.util.concurrent.TimeUnit;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo4 {
    static boolean flag = false;

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread() {
            @Override
            public void run() {
                System.out.println(this.getName() + " start");
                while (true) {
                    synchronized (this) {
                        if (flag) {
                            break;
                        }
                    }
                }
                System.out.println(this.getName() + " exit");
            }
        };
        t1.setName("t1");
        t1.start();
        Thread t2 = new Thread() {
            @Override
            public void run() {
                System.out.println(this.getName() + " start");
                synchronized (this) {
                    while (true) {
                        if (flag) {
                            break;
                        }
                    }
                }
                System.out.println(this.getName() + " exit");
            }
        };
        t2.setName("t2");
        t2.start();
        TimeUnit.SECONDS.sleep(2);
        flag = true;
    }
}
```

运行结果：
![img](https://img2018.cnblogs.com/blog/687624/201907/687624-20190717094128450-1132515455.png)

t1线程可以正常结束，t2线程无法结束，说明主线程中flag修改之后已经被刷新到了主内存了，t1可以看到主内存中中flag最新的值。
t1线程中有个while循环，循环内部有个synchronized块，前面提到过，进入synchronized时，块内部用到的变量在当前线程的工作内存中都会被清空，所以每次进入块中第一次访问flag的时候，都会从主内存中获取，然后复制到工作内存中，所以t1可以正常结束。
t2线程中while循环在synchronized内部，循环内部第一次访问flag的时候会从主内存中获取最新的值，后面再次访问的时候会从工作内存中获取，所以获取到flag一直未false，程序无法结束。

## 线程中断的方式

本文主要探讨一下中断线程的几种方式。

### 通过一个变量控制线程中断

代码：

```java
package com.itsoku.chat05;

import java.util.concurrent.TimeUnit;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo1 {

    public volatile static boolean exit = false;

    public static class T extends Thread {
        @Override
        public void run() {
            while (true) {
                //循环处理业务
                if (exit) {
                    break;
                }
            }
        }
    }

    public static void setExit() {
        exit = true;
    }

    public static void main(String[] args) throws InterruptedException {
        T t = new T();
        t.start();
        TimeUnit.SECONDS.sleep(3);
        setExit();
    }
}
```

代码中启动了一个线程，线程的run方法中有个死循环，内部通过exit变量的值来控制是否退出。`TimeUnit.SECONDS.sleep(3);`让主线程休眠3秒，此处为什么使用TimeUnit？TimeUnit使用更方便一些，能够很清晰的控制休眠时间，底层还是转换为Thread.sleep实现的。程序有个重点：**volatile**关键字，exit变量必须通过这个修饰，如果把这个去掉，程序无法正常退出。volatile控制了变量在多线程中的可见性，关于volatile前面的文章中有介绍，此处就不再说了。

### 通过线程自带的中断标志控制

示例代码：

```java
package com.itsoku.chat05;

import java.util.concurrent.TimeUnit;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo2 {

    public static class T extends Thread {
        @Override
        public void run() {
            while (true) {
                //循环处理业务
                if (this.isInterrupted()) {
                    break;
                }
            }
        }
    }


    public static void main(String[] args) throws InterruptedException {
        T t = new T();
        t.start();
        TimeUnit.SECONDS.sleep(3);
        t.interrupt();
    }
}
```

运行上面的程序，程序可以正常结束。线程内部有个中断标志，当调用线程的interrupt()实例方法之后，线程的中断标志会被置为true，可以通过线程的实例方法isInterrupted()获取线程的中断标志。

### 线程阻塞状态中如何中断

示例代码：

```java
package com.itsoku.chat05;

import java.util.concurrent.TimeUnit;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo3 {

    public static class T extends Thread {
        @Override
        public void run() {
            while (true) {
                //循环处理业务
                //下面模拟阻塞代码
                try {
                    TimeUnit.SECONDS.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }


    public static void main(String[] args) throws InterruptedException {
        T t = new T();
        t.start();
    }
}
```

运行上面代码，发现程序无法结束。

在此先补充几点知识：

1. **调用线程的interrupt()实例方法，线程的中断标志会被置为true**
2. **当线程处于阻塞状态时，调用线程的interrupt()实例方法，线程内部会触发InterruptedException异常，并且会清除线程内部的中断标志（即将中断标志置为false）**

那么上面代码可以调用线程的interrupt()方法来引发InterruptedException异常，来中断sleep方法导致的阻塞，调整一下代码，如下：

```java
package com.itsoku.chat05;

import java.util.concurrent.TimeUnit;

/**
 * 微信公众号：路人甲Java，专注于java技术分享（带你玩转 爬虫、分布式事务、异步消息服务、任务调度、分库分表、大数据等），喜欢请关注！
 */
public class Demo3 {

    public static class T extends Thread {
        @Override
        public void run() {
            while (true) {
                //循环处理业务
                //下面模拟阻塞代码
                try {
                    TimeUnit.SECONDS.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    this.interrupt();
                }
                if (this.isInterrupted()) {
                    break;
                }
            }
        }
    }


    public static void main(String[] args) throws InterruptedException {
        T t = new T();
        t.start();
        TimeUnit.SECONDS.sleep(3);
        t.interrupt();
    }
}
```

运行结果：

```java
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at java.lang.Thread.sleep(Thread.java:340)
	at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
	at com.itsoku.chat05.Demo3$T.run(Demo3.java:17)
```

程序可以正常结束了，分析一下上面代码，注意几点：

1. main方法中调用了t.interrupt()方法，此时线程t内部的中断标志会置为true
2. 然后会触发run()方法内部的InterruptedException异常，所以运行结果中有异常输出，上面说了，当触发InterruptedException异常时候，线程内部的中断标志又会被清除（变为false），**所以在catch中又调用了this.interrupt();一次**，将中断标志置为false
3. run()方法中通过this.isInterrupted()来获取线程的中断标志，退出循环（break）

### 总结

1. 当一个线程处于被阻塞状态或者试图执行一个阻塞操作时，可以使用`Thread.interrupt()`方式中断该线程，注意此时将会抛出一个**InterruptedException**的异常，同时中断状态将会被复位(由中断状态改为非中断状态)
2. 内部有循环体，可以通过一个变量来作为一个信号控制线程是否中断，注意变量需要volatile修饰
3. 文中的几种方式可以结合起来灵活使用控制线程的中断