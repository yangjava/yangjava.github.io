---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo拦截器CompatibleFilter

## CompatibleFilter

## CompatibleFilter
兼容过滤器， 用于兼容不同的Dubbo之间的相互调用
CompatibleFilter#invoke 实现如下：
```java
@Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        Result result = invoker.invoke(invocation);
        // 如果方法不以 $ 开头 && 调用没有出现异常
        if (!invocation.getMethodName().startsWith("$") && !result.hasException()) {
        	// 获取返回值
            Object value = result.getValue();
            if (value != null) {
                try {
                	// 获取方法实例
                    Method method = invoker.getInterface().getMethod(invocation.getMethodName(), invocation.getParameterTypes());
                    // 获取f昂发返回值
                    Class<?> type = method.getReturnType();
                    Object newValue;
                    // 获取序列化方式
                    String serialization = invoker.getUrl().getParameter(Constants.SERIALIZATION_KEY);
                    // 如果是 json 或 fastjson 序列化方式则
                    if ("json".equals(serialization)
                            || "fastjson".equals(serialization)) {
                        // If the serialization key is json or fastjson
                        Type gtype = method.getGenericReturnType();
                        newValue = PojoUtils.realize(value, type, gtype);
                    } else if (!type.isInstance(value)) {
                        //if local service interface's method's return type is not instance of return value
                        // 如果本地服务接口的方法的返回类型不是返回值的实例。进行类型装换
                        newValue = PojoUtils.isPojo(type)
                                ? PojoUtils.realize(value, type)
                                : CompatibleTypeUtils.compatibleTypeConvert(value, type);

                    } else {
                        newValue = value;
                    }
                    if (newValue != value) {
                        result = new RpcResult(newValue);
                    }
                } catch (Throwable t) {
                    logger.warn(t.getMessage(), t);
                }
            }
        }
        return result;
    }
```





