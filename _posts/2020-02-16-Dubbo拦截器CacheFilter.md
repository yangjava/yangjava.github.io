---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo拦截器

## CacheFilter
CacheFilter 为 缓存过滤器，作用于消费者和提供者，通过 cache 属性激活。在启用的情况下，会缓存调用的结果，下次调用直接从缓存中获取。
可通过如下方式指定缓存策略
```java
@Service(cache = "lru")
或
@Reference(cache = "lru")
```
缓存方法有如下方案：
```java
org.apache.dubbo.cache.support.lru.LruCache : 默认使用该策略，最近最少使用
org.apache.dubbo.cache.support.jcache.JCache
org.apache.dubbo.cache.support.expiring.ExpiringCache
org.apache.dubbo.cache.support.threadlocal.ThreadLocalCache
```
CacheFilter#invoke 实现如下，逻辑比较清楚，这里不再赘述
```java

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        if (cacheFactory != null && ConfigUtils.isNotEmpty(invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.CACHE_KEY))) {
        	// 根据指定的缓存策略获取缓存类。cacheFactory 为 缓存策略工厂类，SPI 接口，可以通过实现该接口定义 缓存策略
            Cache cache = cacheFactory.getCache(invoker.getUrl(), invocation);
            if (cache != null) {
                String key = StringUtils.toArgumentString(invocation.getArguments());
                Object value = cache.get(key);
                if (value != null) {
                    if (value instanceof ValueWrapper) {
                        return new RpcResult(((ValueWrapper)value).get());
                    } else {
                        return new RpcResult(value);
                    }
                }
                Result result = invoker.invoke(invocation);
                if (!result.hasException()) {
                    cache.put(key, new ValueWrapper(result.getValue()));
                }
                return result;
            }
        }
        return invoker.invoke(invocation);
    }
```










