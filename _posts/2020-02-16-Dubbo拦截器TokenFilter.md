---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo拦截器TokenFilter

## TokenFilter
TokenFilter 仅供服务提供者使用。实现了token 校验的功能。作用域提供者，通过 token 属性激活。

服务提供者接收到请求后执行到TokenFilter#invoke 会获取自身的token并与远端调用的token 比对，如果不一致则认为是非法调用。Token的可以防止跨注册中心调用的产生。

使用方式详参：
https://dubbo.apache.org/zh/docs/v2.7/user/examples/token-authorization/

TokenFilter#invoke 的实现如下：
```java
    @Override
    public Result invoke(Invoker<?> invoker, Invocation inv)
            throws RpcException {
        // 获取 服务提供者的 token 值
        String token = invoker.getUrl().getParameter(Constants.TOKEN_KEY);
        if (ConfigUtils.isNotEmpty(token)) {
        	// 获取消费者远程调用的token值
            Class<?> serviceType = invoker.getInterface();
            Map<String, String> attachments = inv.getAttachments();
            String remoteToken = attachments == null ? null : attachments.get(Constants.TOKEN_KEY);
            // 进行token比较。
            if (!token.equals(remoteToken)) {
                throw new RpcException("Invalid token! Forbid invoke remote service " + serviceType + " method " + inv.getMethodName() + "() from consumer " + RpcContext.getContext().getRemoteHost() + " to provider " + RpcContext.getContext().getLocalHost());
            }
        }
        return invoker.invoke(inv);
    }

```







