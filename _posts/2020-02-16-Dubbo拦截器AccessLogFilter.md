---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo拦截器AccessLogFilter

## AccessLogFilter

## AccessLogFilter
AccessLogFilter 的作用是将调用记录追加到日志中。作用于提供者，通过 accesslog 属性激活。
- 对于 accesslog = true || default 的情况，直接调用日志框架进行写日志
- 对于其他情况，则认为 accesslog 为日志文件路径。会将其保存到 logQueue中，其中key为 accesslog ，value 为日志内容。同时会启动一个定时任务，每隔 5s 将 logQueue 中的内容写入到日志文件中。
  AccessLogFilter#invoke 实现如下：
```java
    private final ConcurrentMap<String, Set<String>> logQueue = new ConcurrentHashMap<String, Set<String>>();

    @Override
    public Result invoke(Invoker<?> invoker, Invocation inv) throws RpcException {
        try {
            String accesslog = invoker.getUrl().getParameter(Constants.ACCESS_LOG_KEY);
            if (ConfigUtils.isNotEmpty(accesslog)) {
                RpcContext context = RpcContext.getContext();
                String serviceName = invoker.getInterface().getName();
                String version = invoker.getUrl().getParameter(Constants.VERSION_KEY);
                String group = invoker.getUrl().getParameter(Constants.GROUP_KEY);
                StringBuilder sn = new StringBuilder();
 				//  ... 拼接 sn 日志
                String msg = sn.toString();
                // 如果 accesslog 是默认情况 ： accesslog = true || default
                if (ConfigUtils.isDefault(accesslog)) {
                	// 默认情况 直接调用日志框架进行写日志
                    LoggerFactory.getLogger(ACCESS_LOG_KEY + "." + invoker.getInterface().getName()).info(msg);
                } else {
                	// 否则认为 accesslog  指定了写入的日志文件。将其保存到logQueue 中。
                	// 会开启定时任务将 logQueue 的内容写入到日志文件中
                    log(accesslog, msg);
                }
            }
        } catch (Throwable t) {
            logger.warn("Exception in AcessLogFilter of service(" + invoker + " -> " + inv + ")", t);
        }
        return invoker.invoke(inv);
    }

    private void init() {
        if (logFuture == null) {
            synchronized (logScheduled) {
                if (logFuture == null) {
                	// 开启定时任务，5s 执行一次。其中 LogTask 的作用即是将logQueue 中的内容写入到对应的日志文件中。这里篇幅所限，不再贴出LogTask  的实现。
                    logFuture = logScheduled.scheduleWithFixedDelay(new LogTask(), LOG_OUTPUT_INTERVAL, LOG_OUTPUT_INTERVAL, TimeUnit.MILLISECONDS);
                }
            }
        }
    }
	// 将日志保存到 logQueue 中
    private void log(String accesslog, String logmessage) {
        init();
        Set<String> logSet = logQueue.get(accesslog);
        if (logSet == null) {
            logQueue.putIfAbsent(accesslog, new ConcurrentHashSet<String>());
            logSet = logQueue.get(accesslog);
        }
        if (logSet.size() < LOG_MAX_BUFFER) {
            logSet.add(logmessage);
        }
    }

```





