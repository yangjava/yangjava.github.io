---
layout: post
categories: JUC
description: none
keywords: JUC
---

## volatile关键字

volatile是Java提供的一种轻量级的同步机制。同synchronized相比（synchronized通常称为重量级锁），volatile更轻量级。

一旦一个共享变量（类的成员变量、类的静态成员变量）被volatile修饰之后，那么就具备了两层语义：

**1）保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。**

**2）禁止进行指令重排序。**

### **1、共享变量的可见性**

```
public class TestVolatile {
    
    public static void main(String[] args) {
        ThreadDemo td = new ThreadDemo();
        new Thread(td).start();
        while(true){
            if(td.isFlag()){
                System.out.println("------------------");
                break;
            }
        }
    }

}

class ThreadDemo implements Runnable {
    private  boolean flag = false;
    @Override
    public void run() {
        try {
            Thread.sleep(200);
        } catch (InterruptedException e) {
        }
        flag = true;
        System.out.println("flag=" + isFlag());
    }

    public boolean isFlag() {
        return flag;
    }
}
```

上面这个例子，开启一个多线程去改变flag为true，main 主线程中可以输出"------------------"吗？

**答案是NO!**

这个结论会让人有些疑惑，可以理解。开启的线程虽然修改了flag 的值为true,但是还没来得及写入主存当中，此时main里面的 td.isFlag()还是false,但是由于 while(true) 是底层的指令来实现，速度非常之快，一直循环都没有时间去主存中更新td的值，所以这里会造成死循环！运行结果如下：

![img](https://img2018.cnblogs.com/blog/1168971/201811/1168971-20181116112911017-1818746206.png)

此时线程是没有停止的，一直在循环。

如何解决呢？只需将 flag 声明为volatile，即可保证在开启的线程A将其修改为true时，main主线程可以立刻得知：

第一：使用volatile关键字会强制将修改的值立即写入主存；

第二：使用volatile关键字的话，当开启的线程进行修改时，会导致main线程的工作内存中缓存变量flag的缓存行无效（反映到硬件层的话，就是CPU的L1缓存中对应的缓存行无效）；

第三：由于线程main的工作内存中缓存变量flag的缓存行无效，所以线程main再次读取变量flag的值时会去主存读取。

volatile具备两种特性，第一就是保证共享变量对所有线程的可见性。将一个共享变量声明为volatile后，会有以下效应：

**1.当写一个volatile变量时，JMM会把该线程对应的本地内存中的变量强制刷新到主内存中去；**

**2.这个写会操作会导致其他线程中的缓存无效。**



### 2、**禁止进行指令重排序**

**这里我们引用上篇文章单例里面的例子**

```
 1 class Singleton{
 2     private volatile static Singleton instance = null;
 3 
 4     private Singleton() {
 5     }
 6      
 7     public static Singleton getInstance() {
 8         if(instance==null) {
 9             synchronized (Singleton.class) {
10                 if(instance==null)
11                     instance = new Singleton();
12             }
13         }
14         return instance;
15     }
16 }
```

instance = new Singleton(); 这段代码可以分为三个步骤：
1、memory = allocate() 分配对象的内存空间
2、ctorInstance() 初始化对象
3、instance = memory 设置instance指向刚分配的内存

***但是此时有可能发生指令重排，CPU 的执行顺序可能为：***

1、memory = allocate() 分配对象的内存空间
3、instance = memory 设置instance指向刚分配的内存
2、ctorInstance() 初始化对象

在单线程的情况下，1->3->2这种顺序执行是没有问题的，但是如果是多线程的情况则有可能出现问题，线程A执行到11行代码，执行了指令1和3，此时instance已经有值了，值为第一步分配的内存空间地址，但是还没有进行对象的初始化；

此时线程B执行到了第8行代码处，此时instance已经有值了则return instance，线程B 使用instance的时候，就会出现异常。

这里可以**使用 volatile 来禁止指令重排序。**



**从上面知道volatile关键字保证了操作的可见性和有序性，但是volatile能保证对变量的操作是原子性吗？**

下面看一个例子：

```
package com.mmall.concurrency.example.count;
import java.util.concurrent.CountDownLatch;

/**
 * @author: ChenHao
 * @Description:
 * @Date: Created in 15:05 2018/11/16
 * @Modified by:
 */
public class CountTest {
    // 请求总数
    public static int clientTotal = 5000;
    public static volatile int count = 0;

    public static void main(String[] args) throws Exception {
        //使用CountDownLatch来等待计算线程执行完
        final CountDownLatch countDownLatch = new CountDownLatch(clientTotal);
        //开启clientTotal个线程进行累加操作
        for(int i=0;i<clientTotal;i++){
            new Thread(){
                public void run(){
                    count++;//自加操作
                    countDownLatch.countDown();
                }
            }.start();
        }
        //等待计算线程执行完
        countDownLatch.await();
        System.out.println(count);
    }
}
```

执行结果：

![img](https://img2018.cnblogs.com/blog/1168971/201811/1168971-20181116150856988-873369249.png)

针对这个示例，一些同学可能会觉得疑惑，如果用volatile修饰的共享变量可以保证可见性，那么结果不应该是5000么?

问题就出在count++这个操作上，**因为count++不是个原子性的操作，而是个复合操作**。我们可以简单讲这个操作理解为由这三步组成:

1.读取count

2.count 加 1

3.将count 写到主存

**所以，在多线程环境下，有可能线程A将count读取到本地内存中，此时其他线程可能已经将count增大了很多，线程A依然对过期的本地缓存count进行自加，重新写到主存中，最终导致了count的结果不合预期，而是小于5000。**

那么如何来解决这个问题呢？下面我们来看看