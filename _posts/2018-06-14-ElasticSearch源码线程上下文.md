---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码线程上下文
EsThreadPoolExecutor类还有个重要属性ThreadContext（线程上下文）。

## ThreadLocal

### 什么是ThreadLocal

一句话理解ThreadLocal。Threadlocal是作为当前线程中属性ThreadLocalMap集合中的某一个Entry的key值Entry（threadlocal,value）。

虽然不同的线程之间threadlocal这个key值是一样，但是不同的线程所拥有的ThreadLocalMap是独一无二的，也就是不同的线程间同一个ThreadLocal（key）对应存储的值(value)不一样，从而到达了线程间变量隔离的目的，但是在同一个线程中这个value变量地址是一样的。

ThreadLocalMap即是Thread的一个属性，又是ThreadLocal的内部类。ThreadLocal 适用于如下两种场景

1、每个线程需要有自己单独的实例
2、实例需要在多个方法中共享，但不希望被多线程共享

## ThreadLocal在Spring中的应用
Spring使用ThreadLocal解决线程安全问题。有了ThreadLocal，用户可以定义出来仅在同一个线程共享的参数。例如一个Http请求，它是由同一个线程处理的，可以在这个请求里面通过ThreadLocal共享参数，不需要把参数在方法之间互相传递。

SpringMVC源码中RequestContextHolder顾名思义，持有上下文的Request容器，它所有方法都是static，主要维护了基于ThreadLocal两个全局容器。
```
//现成和request绑定的容器
private static final ThreadLocal<RequestAttributes> requestAttributesHolder =
		new NamedThreadLocal<>("Request attributes");
// 和上面比较，它是被子线程继承的request   Inheritable:可继承的
private static final ThreadLocal<RequestAttributes> inheritableRequestAttributesHolder =
		new NamedInheritableThreadLocal<>("Request context");
```

## ThreadContext
JDK和ElasticSearch有关ThreadContext的对比

在ES中，有关线程的操作绝大部分是基于线程池的，和日常业务开发中多线程基于线程类似。同时ThreadPool类包含了线程池的初始化、线程上下文、线程池信息等数据，如果把ES的线程池认为是JDK的线程，这里也可以把ES的ThreadPool类和JDK的Thread作对比。

### 初始化
初始化ThreadContext，保存的是一个请求头和一个Thread的对应关系。
```
/**
 * Creates a new ThreadContext instance
 * @param settings the settings to read the default request headers from
 */
public ThreadContext(Settings settings) {
    Settings headers = DEFAULT_HEADERS_SETTING.get(settings);
    if (headers == null) {
        this.defaultHeader = Collections.emptyMap();
    } else {
        Map<String, String> defaultHeader = new HashMap<>();
        for (String key : headers.names()) {
            defaultHeader.put(key, headers.get(key));
        }
        this.defaultHeader = Collections.unmodifiableMap(defaultHeader);
    }
    threadLocal = new ContextThreadLocal();
    this.maxWarningHeaderCount = SETTING_HTTP_MAX_WARNING_HEADER_COUNT.get(settings);
    this.maxWarningHeaderSize = SETTING_HTTP_MAX_WARNING_HEADER_SIZE.get(settings).getBytes();
}
```
ThreadContext非常灵活，它实现了以下三个实用的功能：

### 线程上下文暂存
```
/**1.下上下文暂存
 * Removes the current context and resets a default context. The removed context can be
 * restored by closing the returned {@link StoredContext}.
 */
public StoredContext stashContext() {
    final ThreadContextStruct context = threadLocal.get();
    /**
     * X-Opaque-ID should be preserved in a threadContext in order to propagate this across threads.
     * This is needed so the DeprecationLogger in another thread can see the value of X-Opaque-ID provided by a user.
     * Otherwise when context is stash, it should be empty.
     */
    if (context.requestHeaders.containsKey(Task.X_OPAQUE_ID)) {
        ThreadContextStruct threadContextStruct =
            DEFAULT_CONTEXT.putHeaders(MapBuilder.<String, String>newMapBuilder()
                .put(Task.X_OPAQUE_ID, context.requestHeaders.get(Task.X_OPAQUE_ID))
                .immutableMap());
        threadLocal.set(threadContextStruct);
    } else {
        threadLocal.set(null);
    }
    return () -> {
        // If the node and thus the threadLocal get closed while this task
        // is still executing, we don't want this runnable to fail with an
        // uncaught exception
        try {
            threadLocal.set(context);
        } catch (IllegalStateException e) {
}
```

### 线程上下文网络传输
ES节点之前涉及TCP通信，有时候需要用到上下文（比如增加一些请求级别的与安全相关的信息，opendistro security插件就用到了很多），ES就实现了ThreadContext的序列化和反序列化，使得线程上下文可以跨节点传输。

### 线程池上下文共享
线程池上下文共享，由于ES实现了自己的线程池，所以如果要保留当前线程的上下文到线程池中，需要做以下五件事：

(1)暂存当前线程上下文

(2)等待线程池中有空闲线程

(3)暂存线程池线程的上下文

(4)恢复线程池线程上下文开始执行

(5)执行完成后恢复线程池线程上下文

这几件事看起来很复杂，其实就是一个线程上下文切换的问题，实现起来也很简单
```
/**
 * Wraps a Runnable to preserve the thread context.
 */
private class ContextPreservingRunnable implements WrappedRunnable {
    private final Runnable in;
    private final ThreadContext.StoredContext ctx;

    private ContextPreservingRunnable(Runnable in) {
        // 将任务放入线程池之前暂存当前线程上下文
        ctx = newStoredContext(false);
        this.in = in;
    }

    @Override
    public void run() {
        boolean whileRunning = false;
        // 暂存线程池线程上下文
        try (ThreadContext.StoredContext ignore = stashContext()){
            // 恢复当前线程上下文
            ctx.restore();
            whileRunning = true;
            in.run();
            // 结束后再恢复线程池线程上下文
            whileRunning = false;
        } catch (IllegalStateException ex) {
```

## Master节点处理
问题：Master节点的操作，大部分是通过clusterService.submitStateUpdateTask提交任务，并由其他线程去执行。这里会存在一个问题，如果一个请求要创建索引“test”，另一个请求要删除索引“test”,提交任务并由其他线程会出现异常吗？答案是不会。

线程池提到newSinglePrioritizing线程池类型，只在MasterService中用于更新集群元数据，其中的任务队列使用了优先级队列PriorityBlockingQueue。把core pool size 和 max pool size 都设置成1，保证了ES集群的任务更新状态是有序的，这里不存在一个任务被多个线程执行的场景。

### MasterService启动
MasterService启动时，创建固定单线程的线程池PrioritizedEsThreadPoolExecutor，并传入创建固定单线程的线程池作参数创建Batcher。这里说明下，创建Batcher传入线程池参数，就是让该线程池执行Batcher这个任务。
```
protected synchronized void doStart() {
    //1.固定单线程的线程池
    threadPoolExecutor = createThreadPoolExecutor();
    taskBatcher = new Batcher(logger, threadPoolExecutor);
}

protected PrioritizedEsThreadPoolExecutor createThreadPoolExecutor() {
    //1.固定单线程的线程池
    return EsExecutors.newSinglePrioritizing(
            nodeName + "/" + MASTER_UPDATE_THREAD_NAME,
            daemonThreadFactory(nodeName, MASTER_UPDATE_THREAD_NAME),
            threadPool.getThreadContext(),
            threadPool.scheduler());
}
```

### PrioritizedEsThreadPoolExecutor
PrioritizedEsThreadPoolExecutor类型的EsThreadPoolExecutor提交的任务必须是PrioritizedRunnable and/or PrioritizedCallable的实现类。
```
public <T> void submitStateUpdateTasks(final String source,
                                       final Map<T, ClusterStateTaskListener> tasks, final ClusterStateTaskConfig config,
                                       final ClusterStateTaskExecutor<T> executor) {
        .......................................
        //封装为UpdateTask(抽象类PrioritizedRunnable的实现,能够在PrioritizedEsThreadPoolExecutor中运行)
        List<Batcher.UpdateTask> safeTasks = tasks.entrySet().stream()
            .map(e -> taskBatcher.new UpdateTask(config.priority(), source, e.getKey(), safe(e.getValue(), supplier), executor))
            .collect(Collectors.toList());
        //提交到TaskBatcher
        taskBatcher.submitTasks(safeTasks, config.timeout());
    }
```
taskBatcher.submitTasks提交任务类型UpdateTask（PrioritizedRunnable 实现类）

提交后的任务，由MasterService初始化时创建的PrioritizedEsThreadPoolExecutor执行，如下代码。
```
//threadExecutor：MasterService对应的是PrioritizedEsThreadPoolExecutor
if (timeout != null) {
    threadExecutor.execute(firstTask, timeout, () -> onTimeoutInternal(tasks, timeout));
} else {
    threadExecutor.execute(firstTask);
}
```

### 监听器与线程上下文
创建索引提交任务的逻辑如下，threadContext.newRestorableContext(true)
```
public <T> void submitStateUpdateTasks(final String source,
                                       final Map<T, ClusterStateTaskListener> tasks, final ClusterStateTaskConfig config,
                                       final ClusterStateTaskExecutor<T> executor) {
    final ThreadContext threadContext = threadPool.getThreadContext();
    //当返回的生产者被调用时，执行restores
    final Supplier<ThreadContext.StoredContext> supplier = threadContext.newRestorableContext(true);
    //下上下文暂存
    try (ThreadContext.StoredContext ignore = threadContext.stashContext()) {
        threadContext.markAsSystemContext();
        //封装为UpdateTask(抽象类PrioritizedRunnable的实现,能够在PrioritizedEsThreadPoolExecutor中运行)
        List<Batcher.UpdateTask> safeTasks = tasks.entrySet().stream()
            .map(e -> taskBatcher.new UpdateTask(config.priority(), source, e.getKey(), safe(e.getValue(), supplier), executor))
            .collect(Collectors.toList());
```
safe(e.getValue(), supplier)封装监听器SafeClusterStateTaskListener对应代码如下：
```
private SafeClusterStateTaskListener safe(ClusterStateTaskListener listener, Supplier<ThreadContext.StoredContext> contextSupplier) {
    //创建索引提交任务IndexCreationTask是AckedClusterStateUpdateTask的子类
    if (listener instanceof AckedClusterStateTaskListener) {
        return new SafeAckedClusterStateTaskListener((AckedClusterStateTaskListener) listener, contextSupplier, logger);
    } else {
        return new SafeClusterStateTaskListener(listener, contextSupplier, logger);
    }
}
-----------------------------------------------------------------------
private static class SafeClusterStateTaskListener implements ClusterStateTaskListener {
    @Override
    public void onFailure(String source, Exception e) {
        //当生产者Supplier<ThreadContext.StoredContext> context.get被调用，StoredContext执行restores
        try (ThreadContext.StoredContext ignore = context.get()) {
            listener.onFailure(source, e);
        } 
```
当任务执行若出现失败会触发SafeClusterStateTaskListener对应的操作：

1.这时生产者Supplier<ThreadContext.StoredContext> context.get被调用，StoredContext执行restores；

2.调用提交任务的监听器。在创建索引对应的是IndexCreationTask类实现ClusterStateTaskListener的接口；

Transport**Action对应的MetaData**Service，提交到ClusterService的任务，在MasterService内封装新的Task和Listener；
新封装的Task由MasterService初始化创建的Executor执行，执行结果由新封装的监听，此时触发ThreadContext的暂存；
之后调用MetaData**Service内AckedClusterStateUpdateTask或者ClusterStateUpdateTask的实现类的监听器，再响应给Client；
个人理解：ES每个任务都会对应一个监听器，监听器会把任务的执行结果向上层反馈，不同线程之间通过ThreadContext进行数据传递。





















