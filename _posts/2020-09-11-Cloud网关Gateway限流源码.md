---
layout: post
categories: [Gateway]
description: none
keywords: Gateway
---
# Cloud网关Gateway限流源码

## 限流源码
通过 RequestRateLimiterGatewayFilterFactory ，可以创建一个GatewayFilter的匿名内部类实例，它的内部使用Redis+Lua实现限流。限流规则由KeyResolver接口的具体实现类来决定，比如通过IP、url等来进行限流。

RequestRateLimiterGatewayFilterFactory核心源码
```
public GatewayFilter apply(Config config) {
    // snipped... 意思是有些跟主要逻辑无关的代码略过了
    // 这里的resolver就是KeyResolver的具体实现，用于解析限流key
    // 默认为PrincipalNameKeyResolver，resolve()方法内容为：
    // return exchange.getPrincipal().flatMap(p -> Mono.justOrEmpty(p.getName()));
    // 如果用过Shiro做鉴权，应该是比较熟悉principal()这个词的，其实就是拿到当前登录用户
    // 所以这里是取用户名作为限流key
    return (exchange, chain) -> resolver.resolve(exchange).defaultIfEmpty(EMPTY_KEY)
        .flatMap(key -> {
            // snipped...
            // limiter这里默认只有RedisRateLimiter的实现
            return limiter.isAllowed(routeId, key).flatMap(response -> {
                // snipped...
                // 如果允许访问，再往下走过滤器链
                if (response.isAllowed()) {
                    return chain.filter(exchange);
                }
                // 被限流了，不允许访问，直接返回
                setResponseStatus(exchange, config.getStatusCode());
                return exchange.getResponse().setComplete();
            });
        });
}
```

## RedisRateLimiter核心源码
```
public Mono<Response> isAllowed(String routeId, String id) {
    // 1.加载Route的配置
    Config routeConfig = loadConfiguration(routeId);
    // 令牌补充速度
    // 官方注释此处直译为：一秒钟允许通过的请求数，为何？
    // 我每小时只充一格电，那这小时只能用一格电，尽管有时手机是满电
    // 我疯狂玩原神，掉电飞快，没过多久就搞没了，电量归零
    // 那我还是变成了从0开始，充一格用一格的状态
    // 由此得出一个结论，只要控制了产出就控制了消耗
    // 虽然听着很废话的感觉，但这样很好理解
    // 如果单纯的用补充速度这个词，不加解释，可能无法马上想到它造成的结果
    // 我觉得这点官方的注释还是非常好的，只是第一次读有点绕不过来，还以为是不是写错了地方呢
    int replenishRate = routeConfig.getReplenishRate();
    // 令牌桶容量
    int burstCapacity = routeConfig.getBurstCapacity();
    // 每个请求申请多少令牌，或者说消耗/取出多少令牌，默认是1个
    int requestedTokens = routeConfig.getRequestedTokens();

    try {
        // 构造tokenKey和时间戳key，都是通过前缀+用户ID拼出来的，用于到redis内去取令牌
        List<String> keys = getKeys(id);
        // 2.封装Lua脚本参数列表
        // Instant.now().getEpochSecond()获得的是从1970-01-01 00:00:00开始的秒数
        List<String> scriptArgs = Arrays.asList(replenishRate + "",
            burstCapacity + "", Instant.now().getEpochSecond() + "",
            requestedTokens + "");
	// 3.allowed, tokens_left = redis.eval(SCRIPT, keys, args)
	// 执行结果里有两个返回值：是否获取令牌成功（1-成功，0-失败）, 剩余令牌数
	Flux<List<Long>> flux = this.redisTemplate.execute(this.script, keys,
	    scriptArgs);

        // 4.返回执行结果
        return flux.onErrorResume(throwable -> {
            // Redis执行Lua脚本发生异常时，忽略异常，直接返回[成功，剩余令牌数：-1]
            // 避免Redis故障导致无法申请到令牌，所有请求直接挂了
            return Flux.just(Arrays.asList(1L, -1L));
	    // 将返回的Flux<List<Long>>类型转换成 Mono<List<Long>>类型
	    // 顺带回顾下Flux和Mono的基础知识:
	    // 1.在开发过程中，不再返回简单的POJO对象，而必须返回其他内容，在结果可用的时候返回。
	    // 2.在响应式流的规范中，被称为发布者（Publisher）。发布者有一个subscribe()方法，该方法允许使用者在POJO可用时获取它。
	    // 3.发布者可以通过以下两种形式返回结果：
	    //   - Flux返回0个或多个结果，可能是无限个
            //   - Mono返回0个或1个结果
	    // Redis执行lua脚本只会返回一次List<Long>，失败时填充默认值也是一次，所以转成Mono
	    }).reduce(new ArrayList<Long>(), (longs, l) -> {
                longs.addAll(l);
                return longs;
            }).map(results -> {
                // 取出Lua脚本的运行结果：是否获取令牌成功（1-成功，0-失败）, 剩余令牌数
                // 5.塞到response的headers里返回，这里result里一共塞了四个参数：
                // 剩余令牌数、每秒补充的令牌数、令牌桶容量、每个请求申请的令牌数
                boolean allowed = results.get(0) == 1L;
                Long tokensLeft = results.get(1);
                Response response = new Response(allowed,
                    getHeaders(routeConfig, tokensLeft));
                return response;
            });
        }
        catch (Exception e) {
            // 虽然redis不是天天抽风，但是万一真发生了这种事，还是留个日志告警
            log.error("Error determining if user allowed from redis", e);
        }
    // 6.最后的兜底尿布，碰到任何奇葩情况导致上面的执行失败了，都给个默认返回：申请令牌成功，剩余令牌数-1
    return Mono.just(new Response(true, getHeaders(routeConfig, -1L)));
}
```

## request_rate_limiter.lua
```
-- 令牌桶需要两个Redis密钥
local tokens_key = KEYS[1]
local timestamp_key = KEYS[2]
--redis.log(redis.LOG_WARNING, "tokens_key " .. tokens_key)

-- 每秒产生多少个令牌
local rate = tonumber(ARGV[1])

-- 令牌桶的容量
local capacity = tonumber(ARGV[2])

-- 时间戳 当前时间的秒数
local now = tonumber(ARGV[3])

-- 需要获取多少个令牌
local requested = tonumber(ARGV[4])

-- 容量/速率 即从无到填满令牌桶需花费的时间，单位：秒
local fill_time = capacity/rate

-- 过期时间
local ttl = math.floor(fill_time*2)

--redis.log(redis.LOG_WARNING, "rate " .. ARGV[1])
--redis.log(redis.LOG_WARNING, "capacity " .. ARGV[2])
--redis.log(redis.LOG_WARNING, "now " .. ARGV[3])
--redis.log(redis.LOG_WARNING, "requested " .. ARGV[4])
--redis.log(redis.LOG_WARNING, "filltime " .. fill_time)
--redis.log(redis.LOG_WARNING, "ttl " .. ttl)

-- 剩余的令牌数量
local last_tokens = tonumber(redis.call("get", tokens_key))
if last_tokens == nil then
  last_tokens = capacity
end
--redis.log(redis.LOG_WARNING, "last_tokens " .. last_tokens)

-- 上一次的剩余过期时间
local last_refreshed = tonumber(redis.call("get", timestamp_key))
if last_refreshed == nil then
  last_refreshed = 0
end
--redis.log(redis.LOG_WARNING, "last_refreshed " .. last_refreshed)

-- 时间间隔 当前时间-上一次的剩余过期时间
local delta = math.max(0, now-last_refreshed)

-- 桶内可用令牌数 = 之前剩余的 + 在delta时间间隔内产生的,最大capacity个
local filled_tokens = math.min(capacity, last_tokens+(delta*rate))

-- 是否允许本次令牌申请,依据是,桶内可用令牌数>=需要获取的令牌数
local allowed = filled_tokens >= requested

-- 需要更新到redis的剩余令牌数
local new_tokens = filled_tokens

-- 本次令牌申请是否成功, 0表示false, 1表示true, 即在java代码中只有当该值为1时, 申请令牌才成功
local allowed_num = 0
if allowed then
-- 如果允许本次令牌的申请, 则剩余令牌数需要减去本次申请的令牌数, 并且设置allowed_num = 1
  new_tokens = filled_tokens - requested
  allowed_num = 1
end

--redis.log(redis.LOG_WARNING, "delta " .. delta)
--redis.log(redis.LOG_WARNING, "filled_tokens " .. filled_tokens)
--redis.log(redis.LOG_WARNING, "allowed_num " .. allowed_num)
--redis.log(redis.LOG_WARNING, "new_tokens " .. new_tokens)

-- ttl必须>0,(即令牌桶容量)capacity>=rate(每秒产生多少个令牌)
if ttl
 > 0 then
  -- 保存剩余令牌数
  redis.call("setex", tokens_key, ttl, new_tokens)
  -- 保存当前时间戳
  redis.call("setex", timestamp_key, ttl, now)
end

-- 返回本次申请成功标志allowed_num和剩余令牌数new_tokens (注意区分剩余令牌数和本次申请的令牌数)
-- return { allowed_num, new_tokens, capacity, filled_tokens, requested, new_tokens }
return { allowed_num, new_tokens }
```

## 场景
在使用SCG限流功能时，默认情况下是按秒限流，即一秒允许多少个请求，现需要根据不同时间频率进行限流，即限制每分钟、每小时或者每天限流。

### 分析
SCG的限流使用的guava的ratelimiter工具，令牌桶模式，参数包括以下3个：
- replenishRate: 每次补充令牌数量
- burstCapacity: 令牌桶最大容量，突发请求数量
- requestedTokens: 每次请求消耗令牌的数量

### 使用方案
- 每秒限制请求1次
```
- name: RequestRateLimiter #基于redis漏斗限流
  args:
    key-resolver: "#{@myResolver}"
    redis-rate-limiter:
      replenishRate: 1
      burstCapacity: 1
      requestedTokens: 1

```

- 每秒限制请求10次
```
- name: RequestRateLimiter #基于redis漏斗限流
  args:
    key-resolver: "#{@myResolver}"
    redis-rate-limiter:
      replenishRate: 10
      burstCapacity: 10
      requestedTokens: 1
```

- 每分钟限制请求1次
```
- name: RequestRateLimiter #基于redis漏斗限流
  args:
    key-resolver: "#{@myResolver}"
    redis-rate-limiter:
      replenishRate: 1
      burstCapacity: 60
      requestedTokens: 60
```

- 每分钟限制请求10次
```
- name: RequestRateLimiter #基于redis漏斗限流
  args:
    key-resolver: "#{@myResolver}"
    redis-rate-limiter:
      replenishRate: 1
      burstCapacity: 60
      requestedTokens: 6
```

- 每小时限制请求1次
```
- name: RequestRateLimiter #基于redis漏斗限流
  args:
    key-resolver: "#{@myResolver}"
    redis-rate-limiter:
      replenishRate: 1
      burstCapacity: 3600
      requestedTokens: 3600
```

- 每小时限制请求10次
```
- name: RequestRateLimiter #基于redis漏斗限流
  args:
    key-resolver: "#{@myResolver}"
    redis-rate-limiter:
      replenishRate: 1
      burstCapacity: 3600
      requestedTokens: 360
```
其他频率以此类推，调整三个参数即可。

### 其他
当触发限流过滤时，在SCG会在redis插入2个key，分别是
- request_rate_limiter.{key}.tokens：当前令牌数，访问时根据令牌数判断是否有资源访问
- request_rate_limiter.{key}.timestamp：上一次访问时间，用于访问时计算上一次到这一次可增长的令牌数，并增加到tokens中。
- TTL：redis中限流key过期时间，规则为burstCapacity/replenishRate*2s， 如1分钟限流key过期时间为60/1*2s=120s

## 自定义限流
```
package com.juzhun.kqc.gateway.service;

import cn.hutool.core.collection.CollectionUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.io.ClassPathResource;
import org.springframework.data.redis.core.ReactiveStringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.scripting.support.ResourceScriptSource;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import javax.annotation.Resource;
import java.time.Instant;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

@Component
@Slf4j
public class RedisLimitService {

    @Resource
    private ReactiveStringRedisTemplate reactiveStringRedisTemplate;

    private static final String PREFIX = "qps_limit_";

    public Boolean isBlack(String key, List<String> blackList){
        if (CollectionUtil.isNotEmpty(blackList)) {
            return blackList.stream().anyMatch(e -> e.contains(key));
        }
        return false;
    }

    // 限流实现
    public Boolean limitQps(String uidKey, long rate, long rateInterval) {
        String key = PREFIX + uidKey;

        DefaultRedisScript redisScript = new DefaultRedisScript<>();
        redisScript.setScriptSource(new ResourceScriptSource(
                new ClassPathResource("META-INF/scripts/request_rate_limiter.lua")));
        redisScript.setResultType(List.class);

        // The arguments to the LUA script. time() returns unixtime in seconds.
        List<String> scriptArgs = Arrays.asList(rate + "",
                rateInterval * rate + "", Instant.now().getEpochSecond() + "",
                1 + "");
        // allowed, tokens_left = redis.eval(SCRIPT, keys, args)
        Flux<List<Long>> flux = this.reactiveStringRedisTemplate.execute(redisScript, getKeys(key),
                scriptArgs);

        Mono<Boolean> mono = flux.onErrorResume(throwable -> {
            if (log.isDebugEnabled()) {
                log.debug("Error calling rate limiter lua", throwable);
            }
            return Flux.just(Arrays.asList(1L, -1L));
        }).reduce(new ArrayList<Long>(), (longs, l) -> {
            longs.addAll(l);
            return longs;
        }).map(results -> {
            boolean allowed = results.get(0) == 1L;
            Long tokensLeft = results.get(1);
            log.info("限流器Key:{},流量限制:{},可用流量:{},是否通过:{}", key, rate, tokensLeft, allowed);
            return allowed;
        });
        return mono.block();
    }

    static List<String> getKeys(String id) {
        // use `{}` around keys to use Redis Key hash tags
        // this allows for using redis cluster

        // Make a unique key per user.
        String prefix = "request_rate_limiter.{" + id;

        // You need two Redis keys for Token Bucket.
        String tokenKey = prefix + "}.tokens";
        String timestampKey = prefix + "}.timestamp";
        return Arrays.asList(tokenKey, timestampKey);
    }
}

```
