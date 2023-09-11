---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo拦截器ValidationFilter

## ValidationFilter
ValidationFilter 完成了参数校验的功能，作用于提供者和消费者, 通过 validation 属性激活。对于 Dubbo 的Rpc 请求来说，有时候也需要参数验证，而ValidationFilter则完成了对 JSR303 标准的验证 annotation，并通过声明 filter 来实现验证。

详细使用参考https://dubbo.apache.org/zh/docs/v2.7/user/examples/parameter-validation/

ValidationFilter#invoke 实现如下：
```java
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
	    //	validation  为  SPI 接口 Validation。默认实现为 JValidation
        if (validation != null && !invocation.getMethodName().startsWith("$")
                && ConfigUtils.isNotEmpty(invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.VALIDATION_KEY))) {
            try {
            	// 获取 Validation 实例
                Validator validator = validation.getValidator(invoker.getUrl());
                if (validator != null) {
                	// 进行参数校验。在该方法中将参数校验委托给 javax.validation.Validator 来完成
                    validator.validate(invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments());
                }
            } catch (RpcException e) {
                throw e;
            } catch (Throwable t) {
                return new RpcResult(t);
            }
        }
        return invoker.invoke(invocation);
    }
```









