---
layout: post
categories: JUC
description: none
keywords: JUC
---
**正文**

今天我们一起探讨下ThreadLocal的实现原理和源码分析。首先，本文先谈一下对ThreadLocal的理解，然后根据ThreadLocal类的源码分析了其实现原理和使用需要注意的地方，最后给出了两个应用场景。相信本文一定能让大家完全了解ThreadLocal。

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10059735.html#_labelTop)

## ThreadLocal是什么？

ThreadLocal是啥？以前面试别人时就喜欢问这个，有些伙伴喜欢把它和线程同步机制混为一谈，事实上ThreadLocal与线程同步无关。ThreadLocal虽然提供了一种解决多线程环境下成员变量的问题，但是它并不是解决多线程共享变量的问题。那么ThreadLocal到底是什么呢？

ThreadLocal很容易让人望文生义，想当然地认为是一个“本地线程”。其实，ThreadLocal并不是一个Thread，而是Thread的**局部变量**，也许把它命名为ThreadLocalVariable更容易让人理解一些。线程局部变量(ThreadLocal)其实的功用非常简单，就是为每一个使用该变量的线程都提供一个变量值的副本，是Java中一种较为特殊的线程绑定机制，是每一个线程都可以独立地改变自己的副本，而不会和其它线程的副本冲突。

通过ThreadLocal存取的数据，总是与当前线程相关，也就是说，JVM 为每个运行的线程，绑定了私有的本地实例存取空间，从而为多线程环境常出现的并发访问问题提供了一种隔离机制。ThreadLocal是如何做到为每一个线程维护变量的副本的呢？其实实现的思路很简单，在ThreadLocal类中有一个Map，用于存储每一个线程的变量的副本。概括起来说，ThreadLocal为每一个线程都提供了一份变量，因此可以同时访问而互不影响。

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10059735.html#_labelTop)

## API说明



### **1、ThreadLocal()**

创建一个线程本地变量。



### **2、T get()**

返回此线程局部变量的当前线程副本中的值，如果这是线程第一次调用该方法，则创建并初始化此副本。



### 3、protected T initialValue()

返回此线程局部变量的当前线程的初始值。最多在每次访问线程来获得每个线程局部变量时调用此方法一次，即线程第一次使用 get() 方法访问变量的时候。如果线程先于 get 方法调用 set(T) 方法，则不会在线程中再调用 initialValue 方法。

若该实现只返回 null；如果程序员希望将线程局部变量初始化为 null 以外的某个值，则必须为 ThreadLocal 创建子类，并重写此方法。通常，将使用匿名内部类。initialValue 的典型实现将调用一个适当的构造方法，并返回新构造的对象。



### 4、void remove()

移除此线程局部变量的值。这可能有助于减少线程局部变量的存储需求。



### 5、void set(T value)

将此线程局部变量的当前线程副本中的值设置为指定值。

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10059735.html#_labelTop)

## ThreadLocal使用示例

假设我们要为每个线程关联一个唯一的序号，在每个线程周期内，我们需要多次访问这个序号，这时我们就可以使用ThreadLocal了

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 package concurrent;
 2 
 3 import java.util.concurrent.atomic.AtomicInteger;
 4 
 5 /**
 6  * Created by chenhao on 2018/12/03.
 7  */
 8 public class ThreadLocalDemo {
 9     public static void main(String []args){
10         for(int i=0;i<5;i++){
11             final Thread t = new Thread(){
12                 @Override
13                 public void run(){
14                     System.out.println("当前线程:"+Thread.currentThread().getName()+",已分配ID:"+ThreadId.get());
15                 }
16             };
17             t.start();
18         }
19     }
20     static   class ThreadId{
21         //一个递增的序列，使用AtomicInger原子变量保证线程安全
22         private static final AtomicInteger nextId = new AtomicInteger(0);
23         //线程本地变量，为每个线程关联一个唯一的序号
24         private static final ThreadLocal<Integer> threadId =
25                 new ThreadLocal<Integer>() {
26                     @Override
27                     protected Integer initialValue() {
28                         return nextId.getAndIncrement();//相当于nextId++,由于nextId++这种操作是个复合操作而非原子操作，会有线程安全问题(可能在初始化时就获取到相同的ID，所以使用原子变量
29                     }
30                 };
31 
32        //返回当前线程的唯一的序列，如果第一次get，会先调用initialValue，后面看源码就了解了
33         public static int get() {
34             return threadId.get();
35         }
36     }
37 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

运行结果：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
当前线程:Thread-4,已分配ID:1
当前线程:Thread-0,已分配ID:0
当前线程:Thread-2,已分配ID:3
当前线程:Thread-1,已分配ID:4
当前线程:Thread-3,已分配ID:2
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10059735.html#_labelTop)

## ThreadLocal源码分析

ThreadLocal最常见的操作就是set、get、remove三个动作，下面来看看这三个动作到底做了什么事情。首先看set操作，源码片段

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
1 public void set(T value) {
2     Thread t = Thread.currentThread();
3     ThreadLocalMap map = getMap(t);
4     if (map != null)
5         map.set(this, value);
6     else
7         createMap(t, value);
8 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

第 2 行代码取出了当前线程 t，然后调用getMap(t)方法时传入了当前线程，换句话说，该方法返回的ThreadLocalMap和当前线程有点关系，我们先记录下来。进一步判定如果这个map不为空，那么设置到Map中的Key就是this，值就是外部传入的参数。这个this是什么呢？就是定义的ThreadLocal对象。
代码中有两条路径需要追踪，分别是getMap(Thread)和createMap(Thread , T)。首先来看看getMap(t)操作

```
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

在这里，我们看到ThreadLocalMap其实就是线程里面的一个属性，它在Thread类中的定义是：

```
ThreadLocal.ThreadLocalMap threadLocals = null;
```

即:每个Thread对象都有一个ThreadLocal.ThreadLocalMap成员变量,ThreadLocal.ThreadLocalMap是一个ThreadLocal类的静态内部类(如下所示),所以Thread类可以进行引用.所以每个线程都会有一个ThreadLocal.ThreadLocalMap对象的引用

```
static class ThreadLocalMap {
```



首先获取当前线程的引用,然后获取当前线程的ThreadLocal.ThreadLocalMap对象,如果该对象为空就创建一个,如下所示:

```
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

这个this变量就是ThreadLocal的引用,对于同一个ThreadLocal对象每个线程都是相同的,但是每个线程各自有一个ThreadLocal.ThreadLocalMap对象保存着各自ThreadLocal引用为key的值,所以互不影响,而且:如果你新建一个ThreadLocal的对象,这个对象还是保存在每个线程同一个ThreadLocal.ThreadLocalMap对象之中,因为一个线程只有一个ThreadLocal.ThreadLocalMap对象,这个对象是在第一个ThreadLocal第一次设值的时候进行创建,如上所述的createMap方法.

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

至此，ThreadLocal的原理我们应该已经清楚了，简单来讲，就是每个Thread里面有一个ThreadLocal.ThreadLocalMap threadLocals作为私有的变量而存在，所以是线程安全的。ThreadLocal通过Thread.currentThread()获取当前的线程就能得到这个Map对象，同时将自身（ThreadLocal对象）作为Key发起写入和读取，由于将自身作为Key，所以一个ThreadLocal对象就能存放一个线程中对应的Java对象，通过get也自然能找到这个对象。

最后来看看get()、remove()代码，或许看到这里就可以认定我们的理论是正确的

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 public T get() {
 2     Thread t = Thread.currentThread();
 3     ThreadLocalMap map = getMap(t);
 4     if (map != null) {
 5         ThreadLocalMap.Entry e = map.getEntry(this);
 6         if (e != null) {
 7             @SuppressWarnings("unchecked")
 8             T result = (T)e.value;
 9             return result;
10         }
11     }
12     return setInitialValue();
13 }
14 
15 public void remove() {
16      ThreadLocalMap m = getMap(Thread.currentThread());
17      if (m != null)
18          m.remove(this);
19 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

第一句是取得当前线程，然后通过getMap(t)方法获取到一个map，map的类型为ThreadLocalMap。然后接着下面获取到<key,value>键值对，注意这里获取键值对传进去的是 this，而不是当前线程t。

如果获取成功，则返回value值。

如果map为空，则调用setInitialValue方法返回value。

可以看出第12行处的方法setInitialValue()只有在线程第一次使用 get() 方法访问变量的时候调用。如果线程先于 get 方法调用 set(T) 方法，则不会在线程中再调用 initialValue 方法。

```
protected T initialValue() {
    return null;
}
```

该方法定义为protected级别且返回为null，很明显是要子类实现它的，所以我们在使用ThreadLocal的时候一般都应该覆盖该方法,创建匿名内部类重写此方法。该方法不能显示调用，只有在第一次调用get()或者set()方法时才会被执行，并且仅执行1次。



**对于ThreadLocal需要注意的有两点：**
\1. ThreadLocal实例本身是不存储值，它只是提供了一个在当前线程中找到副本值得key。
\2. 是ThreadLocal包含在Thread中，而不是Thread包含在ThreadLocal中，有些小伙伴会弄错他们的关系。

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10059735.html#_labelTop)

## ThreadLocal的应用场景

最常见的ThreadLocal使用场景为 用来解决 数据库连接、Session管理等。如：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
/**
 * 数据库连接管理类
 */
public class ConnectionManager {
 
    /** 线程内共享Connection，ThreadLocal通常是全局的，支持泛型 */
    private static ThreadLocal<Connection> threadLocal = new ThreadLocal<Connection>();
    
    public static Connection getCurrConnection() {
        // 获取当前线程内共享的Connection
        Connection conn = threadLocal.get();
        try {
            // 判断连接是否可用
            if(conn == null || conn.isClosed()) {
                // 创建新的Connection赋值给conn(略)
                // 保存Connection
                threadLocal.set(conn);
            }
        } catch (SQLException e) {
            // 异常处理
        }
        return conn;
    }
    
    /**
     * 关闭当前数据库连接
     */
    public static void close() {
        // 获取当前线程内共享的Connection
        Connection conn = threadLocal.get();
        try {
            // 判断是否已经关闭
            if(conn != null && !conn.isClosed()) {
                // 关闭资源
                conn.close();
                // 移除Connection
                threadLocal.remove();
                conn = null;
            }
        } catch (SQLException e) {
            // 异常处理
        }
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

也可以重写initialValue方法

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
private static ThreadLocal<Connection> connectionHolder= new ThreadLocal<Connection>() {
    public Connection initialValue() {
        return DriverManager.getConnection(DB_URL);
    }
};
 
public static Connection getConnection() {
    return connectionHolder.get();
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

Hiberante的Session 工具类HibernateUtil

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public class HibernateUtil {
private static Log log = LogFactory.getLog(HibernateUtil.class);
private static final SessionFactory sessionFactory; //定义SessionFactory
static {
try {
// 通过默认配置文件hibernate.cfg.xml创建SessionFactory
sessionFactory = new Configuration().configure().buildSessionFactory();
} catch (Throwable ex) {
log.error("初始化SessionFactory失败！", ex);
throw new ExceptionInInitializerError(ex);
}
}
//创建线程局部变量session，用来保存Hibernate的Session
public static final ThreadLocal session = new ThreadLocal();
/**
* 获取当前线程中的Session
* @return Session
* @throws HibernateException
*/
public static Session currentSession() throws HibernateException {
Session s = (Session) session.get();
// 如果Session还没有打开，则新开一个Session
if (s == null) {
s = sessionFactory.openSession();
session.set(s); //将新开的Session保存到线程局部变量中
}
return s;
}
public static void closeSession() throws HibernateException {
//获取线程局部变量，并强制转换为Session类型
Session s = (Session) session.get();
session.set(null);
if (s != null)
s.close();
}
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

在这个类中，由于没有重写ThreadLocal的initialValue()方法，则首次创建线程局部变量session其初始值为null，第一次调用currentSession()的时候，线程局部变量的get()方法也为null。因此，对session做了判断，如果为null，则新开一个Session，并保存到线程局部变量session中

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10059735.html#_labelTop)

##  ThreadLocal使用的一般步骤

1、在多线程的类（如ThreadDemo类）中，创建一个ThreadLocal对象threadXxx，用来保存线程间需要隔离处理的对象xxx。

2、在ThreadDemo类中，创建一个获取要隔离访问的数据的方法getXxx()，在方法中判断，若ThreadLocal对象为null时候，应该new()一个隔离访问类型的对象，并强制转换为要应用的类型。

3、在ThreadDemo类的run()方法中，通过getXxx()方法获取要操作的数据，这样可以保证每个线程对应一个数据对象，在任何时刻都操作的是这个对象。

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10059735.html#_labelTop)

## ThreadLocal为什么会内存泄漏

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

上面代码中Entry 继承了WeakReference，说明该map的key为一个弱引用，我们知道弱引用有利于GC回收。

![img](https://img2018.cnblogs.com/blog/1168971/201812/1168971-20181204130755546-2031692808.png)

`　　ThreadLocalMap`使用`ThreadLocal`的弱引用作为`key`，如果一个`ThreadLocal`没有外部强引用来引用它，那么系统 GC 的时候，这个`ThreadLocal`势必会被回收，这样一来，`ThreadLocalMap`中就会出现`key`为`null`的`Entry`，就没有办法访问这些`key`为`null`的`Entry`的`value`，如果当前线程再迟迟不结束的话，这些`key`为`null`的`Entry`的`value`就会一直存在一条强引用链：`Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value`永远无法回收，造成内存泄漏。其实，`ThreadLocalMap`的设计中已经考虑到这种情况，也加上了一些防护措施：在`ThreadLocal`的`get()`,`set()`,`remove()`的时候都会清除线程`ThreadLocalMap`里所有`key`为`null`的`value`。但是这些被动的预防措施并不能保证不会内存泄漏：

- 使用`static`的`ThreadLocal`，延长了`ThreadLocal`的生命周期，可能导致的内存泄漏。
- 分配使用了`ThreadLocal`又不再调用`get()`,`set()`,`remove()`方法，那么就会导致内存泄漏。

[![img](https://img-blog.csdn.net/20171005154856389?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbnNzeQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](https://img-blog.csdn.net/20171005154856389?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbnNzeQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



### 为什么使用弱引用

- **key 使用强引用**：引用的`ThreadLocal`的对象被回收了，但是`ThreadLocalMap`还持有`ThreadLocal`的强引用，如果没有手动删除，`ThreadLocal`不会被回收，导致`Entry`内存泄漏。
- **key 使用弱引用**：引用的`ThreadLocal`的对象被回收了，由于`ThreadLocalMap`持有`ThreadLocal`的弱引用，即使没有手动删除，`ThreadLocal`也会被回收。`value`在下一次`ThreadLocalMap`调用`set`,`get`，`remove`的时候会被清除。

**1、可以知道使用弱引用可以多一层保障：理论上\**弱引用`ThreadLocal`不会内存泄漏，对应的`value`在下一次`ThreadLocalMap`调用`set`,`get`,`remove`的时候会被清除；但是如果分配使用了`ThreadLocal`又不再调用`get()`,`set()`,`remove()`方法，那么就有可能导致内存泄漏\**
**

**2、通常，我们需要\*保证作为key的\*ThreadLocal\*类型能够被全局访问到\*，同时也必须\*保证其为单例\*，因此，在一个类中将其设为static类型便成为了惯用做法，如上面例子中都是用了Static修饰。使用static修饰\**ThreadLocal对象的引用后，\*\*\*\*ThreadLocal\*\*\*\*\**的生命周期跟`Thread`一样长，因此ThreadLocalMap的Key也不会被GC回收，弱引用形同虚设，此时就极容易造成\**\*\*`ThreadLocalMap内存泄露。`\*\**\***

**关键在于threadLocal如果用Static修饰，如果是多线程操作threadlocal，当前线程结束后，ThreadLocal对象作为GCRoot还在其他线程中，这是弱引用就不能被回收，也就是当前Thread中的Map中的key还不会被回收，也就是很多线程中都有threadlocal为key的map不会被回收，那就会出现内存泄露。**



### ThreadLocal 最佳实践

综合上面的分析，我们可以理解`ThreadLocal`内存泄漏的前因后果，那么怎么避免内存泄漏呢？

- 每次使用完`ThreadLocal`，都调用它的`remove()`方法，清除数据。

在使用线程池的情况下，没有及时清理`ThreadLocal`，不仅是内存泄漏的问题，更严重的是可能导致业务逻辑出现问题。所以，使用`ThreadLocal`就跟加锁完要解锁一样，用完就清理。

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10059735.html#_labelTop)

## **总结**

- ThreadLocal 不是用于解决共享变量的问题的，也不是为了协调线程同步而存在，而是为了方便每个线程处理自己的状态而引入的一个机制。这点至关重要。
- 每个Thread内部都有一个ThreadLocal.ThreadLocalMap类型的成员变量，该成员变量用来存储实际的ThreadLocal变量副本。
- ThreadLocal并不是为线程保存对象的副本，它仅仅只起到一个索引的作用。它的主要木得视为每一个线程隔离一个类的实例，这个实例的作用范围仅限于线程内部。
- 每次使用完`ThreadLocal`，都调用它的`remove()`方法，清除数据，避免造成内存泄露。