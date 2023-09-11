---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo拦截器DeprecatedFilter

## DeprecatedFilter


## DeprecatedFilter
DeprecatedFilter 作用是日志记录调用废弃方法的过程，但并非阻断调用，作用于服务消费者，通过 deprecated 属性激活。

DeprecatedFilter#invoke 实现如下：
```java
  @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        String key = invoker.getInterface().getName() + "." + invocation.getMethodName();
        if (!logged.contains(key)) {
            logged.add(key);
            // 如果方法标记已经废弃，则打印 error 日志。但并不阻止本地调用
            if (invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.DEPRECATED_KEY, false)) {
                LOGGER.error("The service method " + invoker.getInterface().getName() + "." + getMethodSignature(invocation) + " is DEPRECATED! Declare from " + invoker.getUrl());
            }
        }
        return invoker.invoke(invocation);
    }
```




