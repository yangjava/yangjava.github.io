---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo拦截器TraceFilter

## TraceFilter


## TraceFilter
TraceFilter 完成了 Dubbo telent 的功能，作用于服务提供者，默认激活状态。

从 2.0.5 版本开始，dubbo 开始支持通过 telnet 命令来进行服务治理。

telnet 命令会被 dubbo handler 读取处理，将 trace 命令解析出来后交由 TraceTelnetHandler。TraceTelnetHandler 找到对应的 Invoker 后调用 TraceFilter#addTracer 方法添加追踪监控任务。之后当服务提供者被调用时，在 TraceFilter#invoke 方法中会判断是否是追踪方法，如果是，则进行追踪并将信息写回 telnet 通道。
```java
   public static void addTracer(Class<?> type, String method, Channel channel, int max) {
        channel.setAttribute(TRACE_MAX, max);
        channel.setAttribute(TRACE_COUNT, new AtomicInteger());
        String key = method != null && method.length() > 0 ? type.getName() + "." + method : type.getName();
        Set<Channel> channels = tracers.get(key);
        if (channels == null) {
            tracers.putIfAbsent(key, new ConcurrentHashSet<Channel>());
            channels = tracers.get(key);
        }
        channels.add(channel);
    }

    public static void removeTracer(Class<?> type, String method, Channel channel) {
        channel.removeAttribute(TRACE_MAX);
        channel.removeAttribute(TRACE_COUNT);
        String key = method != null && method.length() > 0 ? type.getName() + "." + method : type.getName();
        Set<Channel> channels = tracers.get(key);
        if (channels != null) {
            channels.remove(channel);
        }
    }

```
TraceFilter#invoke 实现如下：
```java
   @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        long start = System.currentTimeMillis();
        Result result = invoker.invoke(invocation);
        long end = System.currentTimeMillis();
        // 如果 tracers 不为空，则说明当前有服务跟踪
        if (tracers.size() > 0) {
        	// 获取当前接口 或方法的 telnet 通道
            String key = invoker.getInterface().getName() + "." + invocation.getMethodName();
            Set<Channel> channels = tracers.get(key);
            if (channels == null || channels.isEmpty()) {
                key = invoker.getInterface().getName();
                channels = tracers.get(key);
            }
            if (channels != null && !channels.isEmpty()) {
                for (Channel channel : new ArrayList<Channel>(channels)) {
                	// 当前通道保持连接
                    if (channel.isConnected()) {
                        try {
                            int max = 1;
                            // 获取追踪次数
                            Integer m = (Integer) channel.getAttribute(TRACE_MAX);
                            if (m != null) {
                                max = (int) m;
                            }
                            int count = 0;
                            // 获取已经追踪的次数
                            AtomicInteger c = (AtomicInteger) channel.getAttribute(TRACE_COUNT);
                            if (c == null) {
                                c = new AtomicInteger();
                                channel.setAttribute(TRACE_COUNT, c);
                            }
                            // i++
                            count = c.getAndIncrement();
                            // 如果追踪次数没达到上限，将调用信息写回 telnet 通道
                            if (count < max) {
                                String prompt = channel.getUrl().getParameter(Constants.PROMPT_KEY, Constants.DEFAULT_PROMPT);
                                channel.send("\r\n" + RpcContext.getContext().getRemoteAddress() + " -> "
                                        + invoker.getInterface().getName()
                                        + "." + invocation.getMethodName()
                                        + "(" + JSON.toJSONString(invocation.getArguments()) + ")" + " -> " + JSON.toJSONString(result.getValue())
                                        + "\r\nelapsed: " + (end - start) + " ms."
                                        + "\r\n\r\n" + prompt);
                            }
                            // 到达追踪次数上限，移除该通道
                            if (count >= max - 1) {
                                channels.remove(channel);
                            }
                        } catch (Throwable e) {
                            channels.remove(channel);
                            logger.warn(e.getMessage(), e);
                        }
                    } else {
                        channels.remove(channel);
                    }
                }
            }
        }
        return result;
    }
```






