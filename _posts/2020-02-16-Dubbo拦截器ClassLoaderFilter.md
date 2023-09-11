---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo拦截器ClassLoaderFilter

## ClassLoaderFilter


## ClassLoaderFilter
切换线程上下文类加载器，作用于消费者。默认激活状态。作用是在设计目的中，切换到加载了接口定义的类加载器，以便实现与相同的类加载器上下文一起工作。
```java
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        ClassLoader ocl = Thread.currentThread().getContextClassLoader();
        // 切换类加载器为借口的类加载器
        Thread.currentThread().setContextClassLoader(invoker.getInterface().getClassLoader());
        try {
            return invoker.invoke(invocation);
        } finally {
        	// 切换回类加载器
            Thread.currentThread().setContextClassLoader(ocl);
        }
    }
```




