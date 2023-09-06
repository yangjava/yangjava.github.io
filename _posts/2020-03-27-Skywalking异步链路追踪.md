---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---

# Skywalking的使用-异步链路追踪

### 1、异步链路追踪的概述

通过对 `Callable`,`Runnable`,`Supplier` 这3种接口的实现者进行增强拦截，将trace的上下文信息传递到子线程中，实现了异步链路追踪。

有非常多的方式来实现`Callable`,`Runnable`,`Supplier` 这3种接口，那么增强就面临以下问题：

1. 增强所有的实现类显然不可能，必须基于有限的约定
2. 不能让使用者大量修改代码，尽可能的基于现有的实现

可能基于以上问题的考虑，skywalking提供了一种即通用又快捷的方式来规范这一现象：

1. 只拦截增强带有`@TraceCrossThread` 注解的类：
2. 通过装饰的方式包装任务，避免大刀阔斧的修改

| 原始类      | 提供的包装类       | 拦截方法 | 使用技巧                        |
| ----------- | ------------------ | -------- | ------------------------------- |
| Callable<V> | CallableWrapper<V> | call     | CallableWrapper.of(xxxCallable) |
| Runnable    | RunnableWrapper    | run      | RunnableWrapper.of(xxxRunable)  |
| Supplier<V> | SupplierWrapper<V> | get      | SupplierWrapper.of(xxxSupplier) |

包装类 都有注解 `@TraceCrossThread` ，skywalking内部的拦截匹配逻辑是，标注了`@TraceCrossThread`的类，拦截 其名称为`call` 或`run`或 `get`  ，且没有入参的方法；对使用者来说大致分为2种方式：

1. 自定义类，实现接口 `Callable`、`Runnable`、`Supplier`，加`@TraceCrossThread`注解。当需要有更多的自定义属性时，考虑这种方式；参考 `CallableWrapper`、`RunnableWrapper`、`SupplierWrapper` 的实现方式。
2. 通过xxxWrapper.of 装饰的方式，即`CallableWrapper.of(xxxCallable)`、`RunnableWrapper.of(xxxRunable)`、`SupplierWrapper.of(xxxSupplier)`。大多情况下，通过这种包装模式即可。

### 2、异步链路追踪的使用

需引入如下依赖（版本限参考）：



```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-trace</artifactId>
    <version>8.5.0</version>
</dependency>
```

#### 2.1.  CallableWrapper

Skywalking 通过`CallableWrapper`包装`Callable`

![img](https:////upload-images.jianshu.io/upload_images/4642883-6dfd54c30fd15cb6.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image.png

###### 2.1.1 thread+callable



```dart
private String async_thread_callable(String way,long time11,long time22 ) throws ExecutionException, InterruptedException {
        FutureTask<String> futureTask = new FutureTask<String>(CallableWrapper.of(()->{
            ActiveSpan.debug("async_Thread_Callable");
            String str1 = service.sendMessage(way, time11, time22);
            return str1;
        }));
        new Thread(futureTask).start();
        return futureTask.get();
    }
```

###### 2.1.2 threadPool+callable



```dart
    private String async_executorService_callable(String way,long time11,long time22 ) throws ExecutionException, InterruptedException {
        Future<String> callableResult = executorService.submit(CallableWrapper.of(() -> {
            String str1 = service.sendMessage(way, time11, time22);
            return str1;
        }));
        return (String) callableResult.get();
    }
```

#### 2.2.  RunnableWrapper

Skywalking 通过`RunnableWrapper`包装`Runnable`

![img](https:////upload-images.jianshu.io/upload_images/4642883-b78fbb8216c1f347.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image.png



###### 2.2.1 thread+runnable



```csharp
    private String async_thread_runnable(String way,long time11,long time22 ) throws ExecutionException, InterruptedException {
        //忽略返回值
        FutureTask futureTask = new FutureTask(RunnableWrapper.of(() -> {
            String str1 = service.sendMessage(way, time11, time22);
        }), "mockRunnableResult");
        new Thread(futureTask).start();
        return (String) futureTask.get();
    }
```

###### 2.2.2 threadPool+runnable



```dart
        private String async_executorService_runnable(String way,long time11,long time22 ) throws ExecutionException, InterruptedException {
        //忽略真实返回值，mock固定返回值
        Future<String> mockRunnableResult = executorService.submit(RunnableWrapper.of(() -> {
            String str1 = service.sendMessage(way, time11, time22);
        }), "mockRunnableResult");
        return (String) mockRunnableResult.get();
    }
```

###### 2.2.3 completableFuture + runAsync

通过RunnableWrapper.of(xxx)包装rannable即可。

#### 2.3.  SupplierWrapper

Skywalking 通过`SupplierWrapper<V>`包装`Supplier<V>`

![img](https:////upload-images.jianshu.io/upload_images/4642883-a229457310526e40.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image.png



###### 2.3.1 completableFuture + supplyAsync



```csharp
    private String async_completableFuture_supplyAsync(String way,long time11,long time22 ) throws ExecutionException, InterruptedException {
        CompletableFuture<String> stringCompletableFuture = CompletableFuture.supplyAsync(SupplierWrapper.of(() -> {
            String str1 = service.sendMessage(way, time11, time22);
            return str1;
        }));
        return stringCompletableFuture.get();
    }
```

### 3、异步链路追踪的内部原理

需要将trace信息，在线程之间传递，比如 线程A -调用-> 线程B 的场景：

- 线程A
    1. 调用`ContextManager.capture()`,将trace的上下文信息保存到一个`ContextSnapshot`的实例，并返回，此处命名为：contextSnapshot。
    2. 通过某种方式将contextSnapshot传递给线程B。
- 线程B
    1. 在任务执行前，线程中B获取到contextSnapshot对象，并将其作为入参调用`ContextManager.continued(contextSnapshot)`。
    2. 此方法中解析出trace的信息后，存储到线程B的线程上下文中。



我们都知道 ThreadLocal 作为一种多线程处理手段，将数据限制在当前线程中，避免多线程情况下出现错误。

一般的使用场景大多会是服务上下文、分布式日志跟踪。

但是在业务代码中，为了提高响应速度，将多个复杂、长时间的计算或调用过程异步进行，让主线程可以先进行其他操作。像我们项目中最常用的就是 CompletableFuture 了，默认会使用预设的 ForkJoin ThreadPool 执行。

这也就引入了一个问题，如果保证 ThreadLocal 的信息能够传递异步线程？通过 ThreadLocal？通过线程池？通过 Runnable 或者 Callable？

有些场景丢了就丢了，比如目前我们的服务上下文传递，一般都没有很严谨的处理 ......

但是，如果是分布式追踪的场景，丢了就要累惨了。

注：以下代码仅保留关键代码，其余无关紧要则忽略

# InheritableThreadLocal

InheritableThreadLocal 是 JDK 本身自带的一种线程传递解决方案。顾名思义，由当前线程创建的线程，将会继承当前线程里 ThreadLocal 保存的值。

其本质上是 ThreadLocal 的一个子类，通过覆写父类中创建初始化的相关方法来实现的。我们知道，ThreadLocal 实际上是 Thread 中保存的一个 ThreadLocalMap 类型的属性搭配使用才能让广大 Javaer 直呼真香的，所以 InheritableThreadLocal 也是如此。



```java
public class Thread implements Runnable {
    // 如果单纯使用 ThreadLocal，则 Thread 使用该属性值保存 ThreadLocalMap
    ThreadLocal.ThreadLocalMap threadLocals = null;
        // 否则使用该属性值
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
  
    private void init(ThreadGroup g, Runnable target, String name, long stackSize, AccessControlContext acc) {
          Thread parent = currentThread();

          if (parent.inheritableThreadLocals != null)
              this.inheritableThreadLocals =
                  ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
      }
}
```

init 方法作为 Thread 初始化的核心方法，相关 ThreadLocal 代码已经全部摘出。如我们所见，仅仅就只是这一点改动。在创建线程时，如果当前线程的 inheritableThreadLocals 不为空，则根据它创建出新的 InheritableThreadLocals 保存到新线程中。

Ps : ThreadLocal 作为老牌选手，默认都是使用时，直接初始化 Thread 的 threadLocals 属性。

只有像是 InheritableThreadLocal 这样的后辈，需要特殊处理一下。



```cpp
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    
    protected T childValue(T parentValue) {
        return parentValue;
    }
  
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

   // Thread 中 ThreadLocalMap 不存在时的初始化动作，需要改为初始化 inheritableThreadLocals
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

因此，原先 ThreadLocal 会从 Thread 的 threadLocals 获取 Map，那么 InheritableThreadLocal 就要从 inheritableThreadLocals 拿了。 childValue 方法用作从父线程中获取值，可以看到，这边是直接返回的，如果是复杂对象，就直接传引用了。当然，继承覆写该方法，可以实现浅拷贝、深拷贝等等方式。

# 缺点

这样的方式解决了创建线程时的 ThreadLocal 传值的问题，但不可能一直创建新的线程，那实在耗费资源。因此通用做法是线程复用，比如线程池呗。但是，递交异步任务是相应的 ThreadLocal 的值就无法传递过去了。

我们希望的是，异步线程执行任务的所使用的 ThreadLocal 值，是将任务提交给线程时主线程持有的。即从任务创建时传递到任务执行时。

想想，如果我们在创建异步任务时，在任务代码外获取当前线程的值临时保存，再传递给执行线程，在真正的任务执行前保存到当前线程即可。对，确实可以，但是麻烦不？每个创建异步任务的地方都要写。

那就把它封装到递交任务的方法中。

# RunnableWrapper & CallableWrapper

假设按照服务上下文的场景举例，目前项目中的执行异步操作的方案是定义一个 AsyncExecutor ，并声明执行 Supplier 返回 CompletableFuture 的方法。

既然这样就可以对方法做一些改造，保证上下文的传递。



```dart
private static ThreadLocal<String> contextHolder = new ThreadLocal<>();

public static <T> CompletableFuture<T> invokeToCompletableFuture(Supplier<T> supplier, String errorMessage) {
    // 第一步
    String context = contextHolder.get();
    Supplier<T> newSupplier = () -> {
         // 第二步
        String origin = contextHolder.get();
        try {
            contextHolder.set(context);
            // 第三步
            return supplier.get();
        } finally {
            // 第四步
            contextHolder.set(origin);
            log.info(origin);
        }
    };
    return CompletableFuture.supplyAsync(newSupplier).exceptionally(e -> {
        throw new ServerErrorException(errorMessage, e);
    });
}
// test code
public static void main(String[] args) throws ExecutionException, InterruptedException {
    contextHolder.set("main");
    log.info(contextHolder.get());
    CompletableFuture<String> context = invokeToCompletableFuture(() -> test.contextHolder.get(), "error");
    log.info(context.get());
}
```

总得来说，就是在将异步任务派发给线程池时，对其做一下上下文传递的处理。

第一步：主线程获取上下文，传递给任务暂存。

1 之后的操作都将是异步执行线程操作的。

第二步：异步执行线程将原有上下文取出，暂时保存。并将主线程传递过来的上下文设置。

第三步：执行异步任务

第四步：将原有上下文设置回去。

可以看到一般并不会在异步线程执行完任务之后直接进行 remove 。而是一开始取出原上下文（可能为 NULL，也可能是线程创建时 InheritableThreadLocal 继承过来的值。当然后续也会被清除的），并在任务执行结束重新放回。这样的方式可以说是异步 ThreadLocal 传递的标准范式（大佬说的）。

这样子既起到了显式清除主线程带来的上下文，也避免了如果线程池的拒绝策略为 CallerRunsPolicy ，后续处理时上下文丢失的问题。

Supplier 不算是典型例子，更为典型的应该是 Runnable 和 Callable。不过举一推三，都是修饰一下，再丢给线程池。



```csharp
public final class DelegatingContextRunnable implements Runnable {

    private final Runnable delegate;

    private final Optional<String> delegateContext;

    public DelegatingContextRunnable(Runnable delegate,
                                       Optional<String> context) {
        assert delegate != null;
        assert context != null;

        this.delegate = delegate;
        this.delegateContext = context;
    }

    public DelegatingContextRunnable(Runnable delegate) {
        // 修饰原有的任务，并保存当前线程的值
        this(delegate, ContextHolder.get());
    }

    public void run() {
        Optional<String> originalContext = ContextHolder.get();

        try {
            ContextHolder.set(delegateContext);
            delegate.run();
        } finally {
            ContextHolder.set(originalContext);
        }
    }
}

public final void execute(Runnable task) {
  // 递交给真正的执行线程池前，对任务进行修饰
  executor.execute(wrap(task));
}

protected final Runnable wrap(Runnable task) {
  return new DelegatingContextRunnable(task);
}
```

后续，使用线程池执行异步任务的时候，事先对任务进行封装代理即可。

不过，还是比较麻烦。自定义的线程池，需要显式处理任务。而且更严谨的做法，不同业务场景之间的线程池应该是隔离的，以免受到影响，就比如 Hystrix 的线程池。

每一个线程池都要处理就麻烦了。所以换个思路，代理线程池。

# DelegaingExecutor

这个就不多说了，实际很简单，就照搬我们上下文相关类库。



```java
public class DelegatingContextExecutor implements Executor  {

    private final Executor delegate;


    public DelegatingContextExecutor(Executor delegateExecutor) {
        this.delegate = delegateExecutor;
    }

    public final void execute(Runnable task) {
        delegate.execute(wrap(task));
    }

    protected final Runnable wrap(Runnable task) {
        return new DelegatingContextRunnable(task);
    }

    protected final Executor getDelegateExecutor() {
        return delegate;
    }
}
// 自定义的线程池，用于执行项目中的异步任务
public Executor queryExecutor() {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor();
    // 封装服务上下文的线程池修饰
    return new DelegatingContextExecutorService(threadPoolExecutor);
}
```

问题似乎都解决了，那还有什么？

对，适用场景不够通用。上面的做法只针对于指定的 ThreadLocal，其他场景例如链路追踪、应用容器或上层框架跨应用代码给下层 SDK 传递信息（像是契约包 Feign 的执行线程）。

那么 TransmittableThreadLocal 就是为了解决通用化场景而设计的。

# TransmittableThreadLocal

作为一个核心代码不超过一千行的工具框架，实际使用和架构设计都十分简单。

其使用方法本质上与上述提到的 CallableWrapper 和 DelegatingExecutor 是一样的，并且为了方便使用，对外提供了静态工厂方法或工具类。



```csharp
public final void execute(Runnable task) {
  executor.execute(TtlCallable.get(task));
}
// 或者
public Executor queryExecutor() {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor();
    // 封装服务上下文的线程池修饰
    return TtlExecutors.getTtlExecutorService(threadPoolExecutor);
}
```

当然，前提是 ThreadLocal 必须使用 TransmittableThreadLocal。至于为什么，我们源码分析时再细细说来。

先看看核心实现类的结构，以 Callable 和 ExecutorService 为例。

![img](https:////upload-images.jianshu.io/upload_images/22943445-5e9526ea84028f20?imageMogr2/auto-orient/strip|imageView2/2/w/550/format/webp)

全链路追踪必备组件之 TransmittableThreadLocal 详解

整体主要是三个部分：任务（ TtlCallable ）、线程池（ ExecutorServiceTtlWrapper ）、ThreadLocal（ TransmittableThreadLocal ）。其实对应上述讲到的 CallableWrapper、DelegatingExecutor、InheritableThreadLocal。

但是无论是任务和线程池，本身还是依赖于 TransmittableThreadLocal 对于存储值的管理。

用官方的时序图直观展示一下，框架是如何起作用的：

![img](https:////upload-images.jianshu.io/upload_images/22943445-48b08f9ef0f4e387?imageMogr2/auto-orient/strip|imageView2/2/w/550/format/webp)

全链路追踪必备组件之 TransmittableThreadLocal 详解

可以看到，从第三步创建完任务，第四步修饰完任务，后续大部分过程都依赖于 TransmittableThreadLocal 或 TransmittableThreadLocal 中声明的静态工具类 Transmitter 。Transmitter 主要负责 ThreadLocal 的管理和值的传递。

首先看看 TtlCallable。

# TtlCallable

该类实际上是 JDK Callable 的一个修饰。类比于，上文讲到的 RunnableWrapper，只是为了临时保存父线程 ThreadLocal 的值，以便在执行任务之前，赋值到子线程中。

因此，TtlCallable 和 TtlExecutorService 都实现了 TtlWrapper 接口。也许你以为，该接口是实现修饰的语义，但是它只提供了一个方法，表达了拆修饰的语义：



```java
public interface TtlWrapper<T> extends TtlEnhanced {
    @NonNull
    T unwrap();
}
```

毕竟核心是修饰，所以该类主要为了提供修饰的核心抽象，便于框架对其进行判断和管理。

该方法语义要求，必须返回修饰的源对象或下层对象（毕竟可能修饰了很多层），因此也是空值安全的。null 进来，null 出去。



```java
public final class TtlCallable<V> implements Callable<V>, TtlWrapper<Callable<V>>, TtlEnhanced, TtlAttachments {
    // 保存父线程的 ThreadLocal 快照
    private final AtomicReference<Object> capturedRef;
    // 实际执行任务
    private final Callable<V> callable;
    // 判断是否执行完，清除任务所保存的 ThreadLocal 快照
    private final boolean releaseTtlValueReferenceAfterCall;

    private TtlCallable(@NonNull Callable<V> callable, boolean releaseTtlValueReferenceAfterCall) {
        // 1.创建时， 从 Transmitter 抓取快照
        this.capturedRef = new AtomicReference<Object>(capture());
        this.callable = callable;
        this.releaseTtlValueReferenceAfterCall = releaseTtlValueReferenceAfterCall;
    }

    @Override
    public V call() throws Exception {
        Object captured = capturedRef.get();
        // 如果 releaseTtlValueReferenceAfterCall 为 true，则在执行线程取出快照后清除。
        if (captured == null || releaseTtlValueReferenceAfterCall && !capturedRef.compareAndSet(captured, null)) {
            throw new IllegalStateException("TTL value reference is released after call!");
        }
                // 2.使用 Transmitter 将快照重做到当前执行线程，并将原来的值取出
        Object backup = replay(captured);
        try {
            // 3.执行任务
            return callable.call();
        } finally {
            // 4.Transmitter 重新将原值放回执行线程
            restore(backup);
        }
    }
}
```

可以看到，从实例化到任务执行的顺序，和上文讲到的 CallableWrapper 是完全一致的。但是在其之上，提供了更为完整的特性和线程安全性。

- releaseTtlValueReferenceAfterCall 的可控，保证了任务执行完，依然被业务代码持有的场景下，避免 ThreadLocal 快照继续持有而造成的内存泄漏。毕竟，对于业务方来说，这个东西是我不关心的，无需跟随任务本身的生命周期。
- 快照使用 AtomicReference 保存，保证任务误重用下，清除快照动作的多线程安全性。

上面两者的合用，相当于期望一个任务只能被执行一次，尽量避免任务重用和继续持有。

任务重用的间隔之间，可能出现 ThreadLocal 值被修改的情况，那么后一次任务执行时，快照实际是不准确的。业务场景应该尽量避免这种情况出现才对。

该类提供了静态工厂方法，方便业务方创建。



```tsx
public static <T> TtlCallable<T> get(@Nullable Callable<T> callable) {
    return get(callable, false);
}

@Nullable
public static <T> TtlCallable<T> get(@Nullable Callable<T> callable, boolean releaseTtlValueReferenceAfterCall) {
    return get(callable, releaseTtlValueReferenceAfterCall, false);
}

@Nullable
public static <T> TtlCallable<T> get(@Nullable Callable<T> callable, boolean releaseTtlValueReferenceAfterCall, boolean idempotent) {
    if (null == callable) return null;

    if (callable instanceof TtlEnhanced) {
        // avoid redundant decoration, and ensure idempotency
        if (idempotent) return (TtlCallable<T>) callable;
        else throw new IllegalStateException("Already TtlCallable!");
    }
    return new TtlCallable<T>(callable, releaseTtlValueReferenceAfterCall);
}
```

可以看到，默认工厂方法的
releaseTtlValueReferenceAfterCall 是 false。如果想要使用执行完清除，就要注意方法的使用。

其次，这里还有一个幂等的参数控制： idempotent 。如果传入的 Callable 已经是修饰过的，那么根据 idempotent 的值，要么返回原 Callable，要么报错。

我觉得这里有个两难的点。

我们调用静态工厂方法期望得到的是调用该方法时 ThreadLocal 的快照。所以理论上，应该无论传入什么 Callable，都应该返回一个保存当前本地线程值快照的 TtlCallable。

但是，如果这样的逻辑下，传入的是已修饰的类，那么最后结果就是在任务执行时，会造成外层修饰的快照被内层修饰的覆盖。实际使用的是之前保存的快照了。

因此默认情况就只能 FastFail 。

官方并不建议设置 idempotent 为 true，因为直接返回原修饰类，本身也就违反静态工厂方法的语义。所以官方建议： <b>DO NOT</b> set, only when you know why.

# ExecutorServiceTtlWrapper

该类并不需要多讲，本身与上文的 DelegatingExecutor 一样。



```java
class ExecutorServiceTtlWrapper extends ExecutorTtlWrapper implements ExecutorService, TtlEnhanced {
    private final ExecutorService executorService;

    ExecutorServiceTtlWrapper(@NonNull ExecutorService executorService) {
        super(executorService);
        this.executorService = executorService;
    }

    @NonNull
    @Override
    public <T> Future<T> submit(@NonNull Callable<T> task) {
        return executorService.submit(TtlCallable.get(task));
    }
}
```

其余方法都是一样的做法。

从上文看到，实际 ThreadLocal 的线程传递的核心在于 TransmittableThreadLocal 和 Transmitter。

# TransmittableThreadLocal

TransmittableThreadLocal 只继承了 InheritableThreadLocal 和实现了该框架提供的函数接口 TtlCopier。

因此 TransmittableThreadLocal 自身是一个 InheritableThreadLocal，同样具备了线程创建时传递的特性。

其次，从类体系上看，TransmittableThreadLocal 自身是比较简单的，本质上只是为了让框架能够进行线程传递，做了一些小动作而已。

![img](https:////upload-images.jianshu.io/upload_images/22943445-f201b5562f57a635?imageMogr2/auto-orient/strip|imageView2/2/w/550/format/webp)

全链路追踪必备组件之 TransmittableThreadLocal 详解

可以看到提供的方法是十分少的，源码行数总共也才不超过200行。

首先说一下构造函数。



```java
private final boolean disableIgnoreNullValueSemantics;

public TransmittableThreadLocal() {
    this(false);
}

public TransmittableThreadLocal(boolean disableIgnoreNullValueSemantics) {
    this.disableIgnoreNullValueSemantics = disableIgnoreNullValueSemantics;
}
```

一共两个构造函数，有参构造函数允许设置 “是否禁用忽略空值语义”。默认是开启的，表现行为是如果是 null 值，那么 TransmittableThreadLocal 是不会传递这个值，并且如果 set null，同时执行 remove 操作。表达的意思就是，“我不要 null，不归我管。你敢给我，我就再也不管你了“。

这样设计可能是因为一开始设计服务于业务，是希望业务不要通过 NULL 来表达任何含义，同时避免 NPE 和优化 GC。但是后来官方考虑到作为一个基础服务框架，应该尽量保证完整的语义。毕竟这样的特性是 JDK 的 ThreadLocal 不兼容的。因此后来，官方为了保证兼容性，加了控制参数，允许禁用该特性。

# TtlCopier

TransmittableThreadLocal 实现了一个类，TtlCopier。顾名思义，该类定义了线程传递时，值复制的抽象语义。



```java
public interface TtlCopier<T> {
    T copy(T parentValue);
}
```

而 TransmittableThreadLocal 的默认实现是与 InheritableThreadLocal 相同的，返回值的引用。



```cpp
public T copy(T parentValue) {
    return parentValue;
}
```

同时，该接口也为业务方留下了扩展点。开发者可以重写该方法，来定义线程传递时，如何进行值的复制。

TransmittableThreadLocal 内部维护了一个非常关键的属性，用来注册项目中维护的 TransmittableThreadLocal，从而保证 Transmitter 去正确传递 ThreadLocal 的值。



```tsx
private static InheritableThreadLocal<WeakHashMap<TransmittableThreadLocal<Object>, ?>> holder =
        new InheritableThreadLocal<WeakHashMap<TransmittableThreadLocal<Object>, ?>>() {
            @Override
            protected WeakHashMap<TransmittableThreadLocal<Object>, ?> initialValue() {
                return new WeakHashMap<TransmittableThreadLocal<Object>, Object>();
            }

            @Override
            protected WeakHashMap<TransmittableThreadLocal<Object>, ?> childValue(WeakHashMap<TransmittableThreadLocal<Object>, ?> parentValue) {
                return new WeakHashMap<TransmittableThreadLocal<Object>, Object>(parentValue);
            }
        };
```

holder 是一个 InheritableThreadLocal，用来保存所有注册的 TransmittableThreadLocal。父子线程传递时，可以直接将父线程的注册表传递过来。使用 InheritableThreadLocal，主要保证了嵌套线程场景下，注册表的正确传递。官方有个 issue 以及为其 fix 的 release 版本，从 ThreadLocal 改成了 InheritableThreadLocal。嵌入Thread调用的bug

其次，存储的是 WeakHashMap ，value 都是无意义的 null，并且永远不会被使用。这样一来，保证项目使用 TransmittableThreadLocal 的话，不会引入新的内存泄漏问题。其内存泄漏的可能风险，就只完全来自于 InheritableThreadLocal 本身。



```csharp
@Override
public final T get() {
    T value = super.get();
    if (disableIgnoreNullValueSemantics || null != value) addThisToHolder();
    return value;
}

@Override
public final void set(T value) {
    if (!disableIgnoreNullValueSemantics && null == value) {
        // may set null to remove value
        remove();
    } else {
        super.set(value);
        addThisToHolder();
    }
}

@Override
public final void remove() {
    removeThisFromHolder();
    super.remove();
}

@SuppressWarnings("unchecked")
private void addThisToHolder() {
    if (!holder.get().containsKey(this)) {
        holder.get().put((TransmittableThreadLocal<Object>) this, null); // WeakHashMap supports null value.
    }
}

private void removeThisFromHolder() {
    holder.get().remove(this);
}
```

get & set 会将当前的 TransmittableThreadLocal 注册到 holder 中， remove 时，会删除对应注册。

可以看到，前文说到的
disableIgnoreNullValueSemantics 的值在 get 和 set 时使用到。默认为 false 时，ThreadLocal 不会保存 null，holder 不会注册对应的 TransmittableThreadLocal。

TransmittableThreadLocal 就这样没了，可以看到就很简单。但是，线程传递的内容呢，为什么没有？

这是因为，TransmittableThreadLocal 将线程传递的所有工作全部委托给了其静态内部类 Transmitter。

# Transmitter

我们讲到 TransmittableThreadLocal 会将有值的对象，注册到 holder 中，以便 Transmitter 去知道传递哪一些实例的值。但是如果这样，那不是都要修改代码，将项目中的 ThreadLocal 都改掉吗？

这当然不可能，因此 Transmitter 承担了这个任务，允许业务代码将原有的 ThreadLocal 注册进来，以方便 Transmitter 来识别和传递。



```tsx
// 注册 ThreadLocal 的 threadLocalHolder 依然是 WeakHashMap
private static volatile WeakHashMap<ThreadLocal<Object>, TtlCopier<Object>> threadLocalHolder = new WeakHashMap<ThreadLocal<Object>, TtlCopier<Object>>();
// ThreadLocal 手动注册时用的锁
private static final Object threadLocalHolderUpdateLock = new Object();
// 标记 ThreadLocal 的值已清除，类似于设置一个 null
private static final Object threadLocalClearMark = new Object();
// 传递 TtlCopier，来确定 threadLocal 传递值的方式。默认是 引用传递，与 TransmittableThreadLocal 的 copy 一致。
public static <T> boolean registerThreadLocal(@NonNull ThreadLocal<T> threadLocal, @NonNull TtlCopier<T> copier) {
    return registerThreadLocal(threadLocal, copier, false);
}

@SuppressWarnings("unchecked")
public static <T> boolean registerThreadLocalWithShadowCopier(@NonNull ThreadLocal<T> threadLocal) {
    // 默认是内部定义个 shadowCopier
    return registerThreadLocal(threadLocal, (TtlCopier<T>) shadowCopier, false);
}

public static <T> boolean registerThreadLocal(@NonNull ThreadLocal<T> threadLocal, @NonNull TtlCopier<T> copier, boolean force) {
    // 如果是 TransmittableThreadLocal，则没有必要再维护了。默认就实现了其的传递。
    if (threadLocal instanceof TransmittableThreadLocal) {
        logger.warning("register a TransmittableThreadLocal instance, this is unnecessary!");
        return true;
    }
        
    synchronized (threadLocalHolderUpdateLock) {
        // force 为 false，则不会更新对应的 copier
        if (!force && threadLocalHolder.containsKey(threadLocal)) return false;
                // copy on write
        WeakHashMap<ThreadLocal<Object>, TtlCopier<Object>> newHolder = new WeakHashMap<ThreadLocal<Object>, TtlCopier<Object>>(threadLocalHolder);
        newHolder.put((ThreadLocal<Object>) threadLocal, (TtlCopier<Object>) copier);
        threadLocalHolder = newHolder;
        return true;
    }
}

public static <T> boolean registerThreadLocalWithShadowCopier(@NonNull ThreadLocal<T> threadLocal, boolean force) {
    return registerThreadLocal(threadLocal, (TtlCopier<T>) shadowCopier, force);
}
// 清除 ThreadLocal 的注册
public static <T> boolean unregisterThreadLocal(@NonNull ThreadLocal<T> threadLocal) {
    if (threadLocal instanceof TransmittableThreadLocal) {
        logger.warning("unregister a TransmittableThreadLocal instance, this is unnecessary!");
        return true;
    }

    synchronized (threadLocalHolderUpdateLock) {
        if (!threadLocalHolder.containsKey(threadLocal)) return false;

        WeakHashMap<ThreadLocal<Object>, TtlCopier<Object>> newHolder = new WeakHashMap<ThreadLocal<Object>, TtlCopier<Object>>(threadLocalHolder);
        newHolder.remove(threadLocal);
        threadLocalHolder = newHolder;
        return true;
    }
}
// 默认实现的 TtlCopier，直接引用传递
private static final TtlCopier<Object> shadowCopier = new TtlCopier<Object>() {
    @Override
    public Object copy(Object parentValue) {
        return parentValue;
    }
};
```

其实我自己有个想不明白的，既然已经用了
threadLocalHolderUpdateLock 做锁，为什么还要用 copy on write？GC 友好？mark 一下。

剩下的部分，就是 Transmitter 怎么传递 ThreadLocal 的值了。

实际就是三个步骤，capture -> reply -> restore，crr。

# 1.抓取当前线程的值快照



```dart
// 快照类，用来保存当前线程的 TtlThreadLocal 和 ThreadLocal 的快照
private static class Snapshot {
    final WeakHashMap<TransmittableThreadLocal<Object>, Object> ttl2Value;
    final WeakHashMap<ThreadLocal<Object>, Object> threadLocal2Value;

    private Snapshot(WeakHashMap<TransmittableThreadLocal<Object>, Object> ttl2Value, WeakHashMap<ThreadLocal<Object>, Object> threadLocal2Value) {
        this.ttl2Value = ttl2Value;
        this.threadLocal2Value = threadLocal2Value;
    }
}

public static Object capture() {
    // 抓取快照
    return new Snapshot(captureTtlValues(), captureThreadLocalValues());
}
// 抓取 TransmittableThreadLocal 的快照
private static WeakHashMap<TransmittableThreadLocal<Object>, Object> captureTtlValues() {
    WeakHashMap<TransmittableThreadLocal<Object>, Object> ttl2Value = new WeakHashMap<TransmittableThreadLocal<Object>, Object>();
    // 从 TransmittableThreadLocal 的 holder 中，遍历所有有值的 TransmittableThreadLocal，将 TransmittableThreadLocal 取出和值复制到 Map 中。
    for (TransmittableThreadLocal<Object> threadLocal : holder.get().keySet()) {
        ttl2Value.put(threadLocal, threadLocal.copyValue());
    }
    return ttl2Value;
}

//  抓取注册的 ThreadLocal。
private static WeakHashMap<ThreadLocal<Object>, Object> captureThreadLocalValues() {
    final WeakHashMap<ThreadLocal<Object>, Object> threadLocal2Value = new WeakHashMap<ThreadLocal<Object>, Object>();
    // 从 threadLocalHolder 中，遍历注册的 ThreadLocal，将 ThreadLocal 和 TtlCopier 取出，将值复制到 Map 中。
    for (Map.Entry<ThreadLocal<Object>, TtlCopier<Object>> entry : threadLocalHolder.entrySet()) {
        final ThreadLocal<Object> threadLocal = entry.getKey();
        final TtlCopier<Object> copier = entry.getValue();

        threadLocal2Value.put(threadLocal, copier.copy(threadLocal.get()));
    }
    return threadLocal2Value;
}
```

# 2.将快照重做到执行线程



```dart
@NonNull
public static Object replay(@NonNull Object captured) {
    final Snapshot capturedSnapshot = (Snapshot) captured;
    return new Snapshot(replayTtlValues(capturedSnapshot.ttl2Value), replayThreadLocalValues(capturedSnapshot.threadLocal2Value));
}

// 重播 TransmittableThreadLocal，并保存执行线程的原值
@NonNull
private static WeakHashMap<TransmittableThreadLocal<Object>, Object> replayTtlValues(@NonNull WeakHashMap<TransmittableThreadLocal<Object>, Object> captured) {
    WeakHashMap<TransmittableThreadLocal<Object>, Object> backup = new WeakHashMap<TransmittableThreadLocal<Object>, Object>();
  
    for (final Iterator<TransmittableThreadLocal<Object>> iterator = holder.get().keySet().iterator(); iterator.hasNext(); ) {
        TransmittableThreadLocal<Object> threadLocal = iterator.next();

        // 遍历 holder，从 父线程继承过来的,或者之前注册进来的
        backup.put(threadLocal, threadLocal.get());

        // clear the TTL values that is not in captured
        // avoid the extra TTL values after replay when run task
        // 清除本次没有传递过来的 ThreadLocal，和对应值。毕竟一是可能会有因为 InheritableThreadLocal 而传递并保留的值。二来保证主线程 set 过的 ThreadLocal，不应该被传递过来。明确，其传递是由业务代码控制的，就是明确 set 过值的。
        if (!captured.containsKey(threadLocal)) {
            iterator.remove();
            threadLocal.superRemove();
        }
    }

    // 将 map 中的值，设置到 ThreadLocal 中。
    setTtlValuesTo(captured);

    // TransmittableThreadLocal 的回调方法，在任务执行前执行。
    doExecuteCallback(true);

    return backup;
}

private static void setTtlValuesTo(@NonNull WeakHashMap<TransmittableThreadLocal<Object>, Object> ttlValues) {
    for (Map.Entry<TransmittableThreadLocal<Object>, Object> entry : ttlValues.entrySet()) {
        TransmittableThreadLocal<Object> threadLocal = entry.getKey();
        // set 的同时，也就将 TransmittableThreadLocal 注册到当前线程的注册表了。
        threadLocal.set(entry.getValue());
    }
}

private static WeakHashMap<ThreadLocal<Object>, Object> replayThreadLocalValues(@NonNull WeakHashMap<ThreadLocal<Object>, Object> captured) {
    final WeakHashMap<ThreadLocal<Object>, Object> backup = new WeakHashMap<ThreadLocal<Object>, Object>();

    for (Map.Entry<ThreadLocal<Object>, Object> entry : captured.entrySet()) {
        final ThreadLocal<Object> threadLocal = entry.getKey();
        backup.put(threadLocal, threadLocal.get());

        final Object value = entry.getValue();
        // 如果值是标记已删除，则清除
        if (value == threadLocalClearMark) threadLocal.remove();
        else threadLocal.set(value);
    }

    return backup;
}
```

doExecuteCallback 是 TransmittableThreadLocal 定义的回调方法，保证任务执行前和执行后的回调动作。

isBefore 控制是执行前还是执行后。

内部调用了 beforeExecute 和 afterExecute 方法。默认是不做任何动作。



```cpp
private static void doExecuteCallback(boolean isBefore) {
    for (TransmittableThreadLocal<Object> threadLocal : holder.get().keySet()) {
        try {
            if (isBefore) threadLocal.beforeExecute();
            else threadLocal.afterExecute();
        } catch (Throwable t) {
            // 忽略所有异常，保证任务的执行
            if (logger.isLoggable(Level.WARNING)) {
                logger.log(Level.WARNING, "TTL exception when " + (isBefore ? "beforeExecute" : "afterExecute") + ", cause: " + t.toString(), t);
            }
        }
    }
}
protected void beforeExecute() {
}

protected void afterExecute() {
}
```

# 3.恢复备份的原快照



```dart
public static void restore(@NonNull Object backup) {
    final Snapshot backupSnapshot = (Snapshot) backup;
    restoreTtlValues(backupSnapshot.ttl2Value);
    restoreThreadLocalValues(backupSnapshot.threadLocal2Value);
}

private static void restoreTtlValues(@NonNull WeakHashMap<TransmittableThreadLocal<Object>, Object> backup) {
    // call afterExecute callback 任务执行完回调
    doExecuteCallback(false);

    for (final Iterator<TransmittableThreadLocal<Object>> iterator = holder.get().keySet().iterator(); iterator.hasNext(); ) {
        TransmittableThreadLocal<Object> threadLocal = iterator.next();

        // clear the TTL values that is not in backup
        // avoid the extra TTL values after restore
        // 恢复快照时，清除本次传递注册进来，但是原先不存在的 TransmittableThreadLocal
        if (!backup.containsKey(threadLocal)) {
            iterator.remove();
            threadLocal.superRemove();
        }
    }

    // restore TTL values
    // 恢复快照中的 value 到 TransmittableThreadLocal 中
    setTtlValuesTo(backup);
}

private static void setTtlValuesTo(@NonNull WeakHashMap<TransmittableThreadLocal<Object>, Object> ttlValues) {
    for (Map.Entry<TransmittableThreadLocal<Object>, Object> entry : ttlValues.entrySet()) {
        TransmittableThreadLocal<Object> threadLocal = entry.getKey();
        threadLocal.set(entry.getValue());
    }
}

private static void restoreThreadLocalValues(@NonNull WeakHashMap<ThreadLocal<Object>, Object> backup) {
    for (Map.Entry<ThreadLocal<Object>, Object> entry : backup.entrySet()) {
        final ThreadLocal<Object> threadLocal = entry.getKey();
        threadLocal.set(entry.getValue());
    }
}
```

# 对特殊场景以及 Lambda 的支持

Transmitter 定义了几个特殊场景下以及 Java 8 lambda 表达式的使用。

特殊场景就是指，执行前，清除当前执行线程 ThreadLocal 的值，包括 TtlThreadLocal 和注册 ThreadLocal 。

像一开始讲到的业务代码喜欢使用 Supplier，所以也对其做了支持。本质是为了简化工作。

不过，注意的是，快照的捕获则需要业务代码自己完成并传递。



```tsx
public static <R> R runSupplierWithCaptured(@NonNull Object captured, @NonNull Supplier<R> bizLogic) {
    Object backup = replay(captured);
    try {
        return bizLogic.get();
    } finally {
        restore(backup);
    }
}

public static <R> R runSupplierWithClear(@NonNull Supplier<R> bizLogic) {
    Object backup = clear();
    try {
        return bizLogic.get();
    } finally {
        restore(backup);
    }
}

public static <R> R runCallableWithCaptured(@NonNull Object captured, @NonNull Callable<R> bizLogic) throws Exception {
    Object backup = replay(captured);
    try {
        return bizLogic.call();
    } finally {
        restore(backup);
    }
}

public static <R> R runCallableWithClear(@NonNull Callable<R> bizLogic) throws Exception {
    Object backup = clear();
    try {
        return bizLogic.call();
    } finally {
        restore(backup);
    }
}
```

简化方法，使用起来也就是：



```dart
// 线程A
Object captured = Transmitter.capture();

// 线程B
@Async
String result = runSupplierWithCaptured(captured, () -> {
  
     System.out.println("Hello");
     ...
     return "World";
});
```

否则只能按照全套流程了：



```kotlin
// 线程A
Object captured = Transmitter.capture();

// 线程B
@Async
String result = runSupplierWithCaptured(captured, () -> {
  
     System.out.println("Hello");
     ...
     return "World";
});
Object backup = Transmitter.replay(captured); // (2)
try {
    System.out.println("Hello");
    // ...
    return "World";
} finally {
    // restore the TransmittableThreadLocal of thread B when replay
    Transmitter.restore(backup); (3)
```

# Clear

上面可以看到，一些方法是做了 clear 操作。

就是不依赖快照的捕获，将空值的快照信息，传递给重做方法执行，就能清除当前执行线程的值，并得到返回原值备份。



```dart
public static Object clear() {
    final WeakHashMap<TransmittableThreadLocal<Object>, Object> ttl2Value = new WeakHashMap<TransmittableThreadLocal<Object>, Object>();

    final WeakHashMap<ThreadLocal<Object>, Object> threadLocal2Value = new WeakHashMap<ThreadLocal<Object>, Object>();
    for (Map.Entry<ThreadLocal<Object>, TtlCopier<Object>> entry : threadLocalHolder.entrySet()) {
        final ThreadLocal<Object> threadLocal = entry.getKey();
        // threadLocalClearMark 标记为未被传递和注册，更为合适，从而避免和 null 混淆。否则无法区分原有就是 null，还是未被注册
        threadLocal2Value.put(threadLocal, threadLocalClearMark);
    }

    return replay(new Snapshot(ttl2Value, threadLocal2Value));
}
```

# 注意

如果注意到 TransmittableThreadLocal 是继承 InheritableThreadLocal，就应该知道，子线程创建时，值还是会被传递过去。这也就可能带来内存泄漏问题。

所以，同时提供
DisableInheritableThreadFactoryWrapper，以方便业务代码自定义线程池，禁止值的继承传递。



```java
class DisableInheritableThreadFactoryWrapper implements DisableInheritableThreadFactory {
    private final ThreadFactory threadFactory;

    DisableInheritableThreadFactoryWrapper(@NonNull ThreadFactory threadFactory) {
        this.threadFactory = threadFactory;
    }

    @Override
    public Thread newThread(@NonNull Runnable r) {
        // 调用了 Transmitter 的 clear 方法，在创建子线程前，清除当前线程的值，并保存下来
        final Object backup = clear();
        try {
            return threadFactory.newThread(r);
        } finally {
            // 创建完，再重新恢复。以此，避免了值的继承传递。
            restore(backup);
        }
    }

    @NonNull
    @Override
    public ThreadFactory unwrap() {
        return threadFactory;
    }
}
```

对于 1.8 特性，还提供了
ForkJoinWorkerThreadFactory 和 TtlForkJoinPoolHelper 等类的支持。

# Java Agent 支持

避免代码改动的话，可以使用 Java Agent，来隐式替换 JDK 的相应类。对于 1.8 的 CompletableFuture 和 Stream，在底层通过对 ForkJoinPool 的支持，也做了透明支持。

# 总结

到此，TransmittableThreadLocal 的源码解析就结束了。核心源码是不是很简单？但是某些思想和考量还是很值得学习的。

ThreadLocal 的使用，本身类似于全局变量，而且是可修改的。一旦中间过程被修改，就无法保证整体流程的前后一致性。它将是一个隐藏的强依赖，一个可能被忽略、意想不到的坑。（我不承认，我在还原大佬的话。）

应该尽量避免在业务代码中使用的。 **DO NOT** use, only when you know why .

嗯，还有加上一句，让其他人也明白，文档务必齐全。（说实话，我挺想用英文的，想想算了）。


# skywalking线程池插件,解决lambda使用问题

1. 简介
   分布式链路追踪其实原理都很简单，每一个请求过来的时候，把一个context存到ThreadLocal里面，然后随着线程一路传递下去就行，一个请求大多数时候都是一个线程去执行下去所以并没有什么问题，但是如果在请求中使用了ThreadPool那么ThreadLocal中的context就会失联(如果使用的是Thread，ThreadLocal是可以继续传递下去的，具体原因可以去看ThreadLocal和ThreadPool的实现这里就不过多解释了)，所以需要插件的支持去把这个失联的context给找回来

2. 官方插件
   skywalking 的跨线程池 其实是已经有官方插件的支持了。实现的原理也是非常简单，他会去织入业务方自己实现的Runnable类，在执行run()方法之前去传递context。

使用方法：
修改 skywalking-agent.jar所在目录的 config/agent.config文件，在里面修改/添加配置
#他将会去把指定包下所有`Runnable`的实现类给织入，可以前缀匹配
jdkthreading.threading_class_prefixes=com.chy
1
2
然后在plugins文件夹里加入插件apm-jdk-threading-plugin.jar就能够使用了(如果是下载官方编译好的插件，apm-jdk-threading-plugin.jar被放在了bootstrap-plugins文件夹中，移动到plugins即可)

缺点：
虽然简单但是却有一个缺点，无法使用lambda表达式，比如下面的代码，就无法正常的把链路给连接起来

@GetMapping("/ajax/{setId}/async")
public List<DoctorListVO> getDoctorListasync(@PathVariable Integer setId) throws ExecutionException, InterruptedException {


        //线程池访问
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Future<List<DoctorListVO>> submit = executor.submit(() -> {
            //远程调用了别的服务
            return doctorSetAdminService.queryDoctorList(setId);
        });
        Future<List<DoctorListVO>> submit2 = executor.submit(() -> {
            return doctorSetAdminService.queryDoctorList(setId);
        });
    
        //和上面 queryDoctorList 一样只是queryDoctorListAsy() 方法都是打了 `@Async` 注解
        doctorSetAdminService.queryDoctorListAsy(setId);
        doctorSetAdminService.queryDoctorListAsy(setId);


        List<DoctorListVO> doctorListVOS = submit.get();
        List<DoctorListVO> doctorListVOS2 = submit2.get();
        return doctorListVOS;
    }

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
简单说明一下原因，上面提到过这个插件的原理是去修改所有Runnable的实现类，但是如果使用lambda去实现的Runnable类，虽然会生成一个匿名内部类，但是这个匿名内部类的和skywalking-agent使用的不同的类加载器，导致skywalking-agent无法去修改lambda表达式生成的Runnable的实现类.

3. 非官方插件
   为了解决lambda的问题，博主自己写了一个插件去解决这个问题，思路也很简单，既然我不能去修改lambda生成的类，那么我就换个地方去修改他，比如在调用的时候再去修改。
   所以我把切入点放在了ThreadPoolExecutor的execute(Runnable r)方法上面，也就是每当执行execute(Runnable r)方法的时候，我都会拦截下这个方法，然后把他的入参的Runnable给换成我的代理对象，然后继续去执行execute(Runnable r)方法

话不多说，直接上插件的地址，使用方法在readme里有写

github
gitee
这里需要注意的是，除了去把插件放入plugins文件夹外，还需要替换 skywalking-agent.jar 文件,原因在于博主的切入点是ThreadPoolExecutor 需要在字节码框架上做一些额外的配置。

还是执行这个代码

@GetMapping("/ajax/{setId}/async")
public List<DoctorListVO> getDoctorListasync(@PathVariable Integer setId) throws ExecutionException, InterruptedException {


        //线程池访问
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Future<List<DoctorListVO>> submit = executor.submit(() -> {
            //远程调用了别的服务
            return doctorSetAdminService.queryDoctorList(setId);
        });
        Future<List<DoctorListVO>> submit2 = executor.submit(() -> {
            return doctorSetAdminService.queryDoctorList(setId);
        });
    
        //和上面 queryDoctorList 一样只是queryDoctorListAsy() 方法都是打了 `@Async` 注解
        doctorSetAdminService.queryDoctorListAsy(setId);
        doctorSetAdminService.queryDoctorListAsy(setId);


        List<DoctorListVO> doctorListVOS = submit.get();
        List<DoctorListVO> doctorListVOS2 = submit2.get();
        return doctorListVOS;
    }

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
结果如下