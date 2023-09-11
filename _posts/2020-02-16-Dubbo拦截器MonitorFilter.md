---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo拦截器MonitorFilter

## MonitorFilter

## MonitorFilter
作用于消费者和提供者。
```java
	@Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    	// 如果指定了监控中心
        if (invoker.getUrl().hasParameter(Constants.MONITOR_KEY)) {
            RpcContext context = RpcContext.getContext(); // provider must fetch context before invoke() gets called
            String remoteHost = context.getRemoteHost();
            long start = System.currentTimeMillis(); // record start timestamp
            // 增加活跃调用次数
            getConcurrent(invoker, invocation).incrementAndGet(); // count up
            try {
            	// 服务调用
                Result result = invoker.invoke(invocation); // proceed invocation chain
                collect(invoker, invocation, result, remoteHost, start, false);
                return result;
            } catch (RpcException e) {
                collect(invoker, invocation, null, remoteHost, start, true);
                throw e;
            } finally {
            	// 减少活跃调用次数
                getConcurrent(invoker, invocation).decrementAndGet(); // count down
            }
        } else {
            return invoker.invoke(invocation);
        }
    }
	// 汇总监控指标
    private void collect(Invoker<?> invoker, Invocation invocation, Result result, String remoteHost, long start, boolean error) {
        try {
        	// 获取监控中心实例
            URL monitorUrl = invoker.getUrl().getUrlParameter(Constants.MONITOR_KEY);
            // 这里的监控中心实例为代理类。
            Monitor monitor = monitorFactory.getMonitor(monitorUrl);
            if (monitor == null) {
                return;
            }
            // 创建 统计 URL。URL作为信息承载
            URL statisticsURL = createStatisticsUrl(invoker, invocation, result, remoteHost, start, error);
            // 数据汇
            monitor.collect(statisticsURL);
        } catch (Throwable t) {
            logger.warn("Failed to monitor count service " + invoker.getUrl() + ", cause: " + t.getMessage(), t);
        }
    }
```






