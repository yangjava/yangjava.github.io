---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo拦截器ActiveLimitFilter

## ActiveLimitFilter

## ActiveLimitFilter & ExecuteLimitFilter
ActiveLimitFilter & ExecuteLimitFilter 完成了Dubbo 并发调用的次数限制。
- ActiveLimitFilter 控制了消费者端最大并发，作用于消费者， 通过 actives 属性激活。ActiveLimitFilter 限制的次数作用的是同一个提供者的同一个接口的同一个方法。
- ExecuteLimitFilter 控制了提供者端最大并发，作用于提供者，通过 executes 属性激活。ExecuteLimitFilter 限制的次数作用的是当前提供者的某个方法最大可以支持多少并发访问。

## ActiveLimitFilter
ActiveLimitFilter#invoke 实现如下：
```java
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        URL url = invoker.getUrl();
        String methodName = invocation.getMethodName();
        // 获取最大调用次数
        int max = invoker.getUrl().getMethodParameter(methodName, Constants.ACTIVES_KEY, 0);
        // 获取当前消费者对 当前方法的调用统计状态 RpcStatus 
        RpcStatus count = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName());
        // 判断是否达到最大限制，如果到达次数限制则进入 if分支
        if (!RpcStatus.beginCount(url, methodName, max)) {
        	// 获取超时时间
            long timeout = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.TIMEOUT_KEY, 0);
            // 获取当前时间作为开始时间
            long start = System.currentTimeMillis();
            // 最大等待时间。为 超时时间
            long remain = timeout;
            synchronized (count) {
            	// while 循环，直到活跃调用次数小于 max时才会跳出
                while (!RpcStatus.beginCount(url, methodName, max)) {
                    try {
                    	// 等待，最大等待时间为 remain
                        count.wait(remain);
                    } catch (InterruptedException e) {
                        // ignore
                    }
                    // 获取等待时间差
                    long elapsed = System.currentTimeMillis() - start;
                    // 如果等待阶段已经超时则抛出异常
                    remain = timeout - elapsed;
                    if (remain <= 0) {
						//... 抛出异常
                    }
                }
            }
        }
		// 到达这里说明当前活跃访问数量小于 最大限制数 max，可以进行正常调用
        boolean isSuccess = true;
        long begin = System.currentTimeMillis();
        try {
        	// 进行服务调用
            return invoker.invoke(invocation);
        } catch (RuntimeException t) {
            isSuccess = false;
            throw t;
        } finally {
        	// 减少一个活跃调用次数
            RpcStatus.endCount(url, methodName, System.currentTimeMillis() - begin, isSuccess);
            if (max > 0) {
                synchronized (count) {
                	// 唤醒其他等待者
                    count.notifyAll();
                }
            }
        }
    }
```

## ExecuteLimitFilter
ExecuteLimitFilter#invoke 提供端的活跃调用限制 。逻辑和 ActiveLimitFilter 基本相同。其实现如下：
```java
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        URL url = invoker.getUrl();
        String methodName = invocation.getMethodName();
        // 获取服务端针对 methodName 的最大并发调用次数
        int max = url.getMethodParameter(methodName, Constants.EXECUTES_KEY, 0);
        // 如果当前并发超过了最大并发次数限制。直接抛出异常
        if (!RpcStatus.beginCount(url, methodName, max)) {
            throw new RpcException("Failed to invoke method " + invocation.getMethodName() + " in provider " +
                    url + ", cause: The service using threads greater than <dubbo:service executes=\"" + max +
                    "\" /> limited.");
        }
		// 开始常规调用
        long begin = System.currentTimeMillis();
        boolean isSuccess = true;
        try {
            return invoker.invoke(invocation);
        } catch (Throwable t) {
            isSuccess = false;
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new RpcException("unexpected exception when ExecuteLimitFilter", t);
            }
        } finally {
        	// 调用结束，并发量减一
            RpcStatus.endCount(url, methodName, System.currentTimeMillis() - begin, isSuccess);
        }

```
注意：
- ExecuteLimitFilter在超过限制的数量后会直接返回，而ActiveLimitFilter会等待，直到超时。
- RpcStatus 中缓存了每个方法的正在调用的次数等信息。并且通过原子类（AtomicXxx） 来控制线程安全。





