---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo拦截器ExceptionFilter

## ExceptionFilter

## ExceptionFilter
ExceptionFilter 作用即是对服务端的调用异常做规范化处理。作用于服务提供者，默认激活状态。将所有不满足条件的异常封装成 RuntimeException 返回给客户端。
ExceptionFilter#invoke 实现如下：
```java
 @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        try {
            return invoker.invoke(invocation);
        } catch (RuntimeException e) {
        	// 运行时异常打印 error 日志后抛出
            logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost()
                    + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName()
                    + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
            throw e;
        }
    }

    @Override
    public Result onResponse(Result result, Invoker<?> invoker, Invocation invocation) {
    	// 非泛化调用时出现了异常
        if (result.hasException() && GenericService.class != invoker.getInterface()) {
            try {
                Throwable exception = result.getException();

                // directly throw if it's checked exception
                // 受检异常(编译时异常)直接抛出
                if (!(exception instanceof RuntimeException) && (exception instanceof Exception)) {
                    return result;
                }
                // directly throw if the exception appears in the signature
                try {
                	// 如果该方法声明了该异常，则直接抛出
                    Method method = invoker.getInterface().getMethod(invocation.getMethodName(), invocation.getParameterTypes());
                    Class<?>[] exceptionClassses = method.getExceptionTypes();
                    for (Class<?> exceptionClass : exceptionClassses) {
                        if (exception.getClass().equals(exceptionClass)) {
                            return result;
                        }
                    }
                } catch (NoSuchMethodException e) {
                    return result;
                }
				// 未在方法签名上定义的异常，在服务器端打印ERROR日志
                // for the exception not found in method's signature, print ERROR message in server's log.
                logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost()
                        + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName()
                        + ", exception: " + exception.getClass().getName() + ": " + exception.getMessage(), exception);

                // directly throw if exception class and interface class are in the same jar file.
                // 如果异常类和接口类在同一个jar文件中，则直接抛出。
                String serviceFile = ReflectUtils.getCodeBase(invoker.getInterface());
                String exceptionFile = ReflectUtils.getCodeBase(exception.getClass());
                if (serviceFile == null || exceptionFile == null || serviceFile.equals(exceptionFile)) {
                    return result;
                }
                // directly throw if it's JDK exception
                // 如果是 JDK 异常直接抛出
                String className = exception.getClass().getName();
                if (className.startsWith("java.") || className.startsWith("javax.")) {
                    return result;
                }
                // directly throw if it's dubbo exception
               	// 如果是 Dubbo的  RpcException 异常直接抛出
                if (exception instanceof RpcException) {
                    return result;
                }

                // otherwise, wrap with RuntimeException and throw back to the client
                // 否则封装成 RuntimeException 异常返回给客户端
                return new RpcResult(new RuntimeException(StringUtils.toString(exception)));
            } catch (Throwable e) {
                logger.warn("Fail to ExceptionFilter when called by " + RpcContext.getContext().getRemoteHost()
                        + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName()
                        + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
                return result;
            }
        }
        return result;
    }
```





