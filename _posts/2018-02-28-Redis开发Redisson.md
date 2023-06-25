---
layout: post
categories: Redis
description: none
keywords: Redis
---
# Redis开发实战
Redisson是Redis官方推荐的Java版的Redis客户端。它提供了使用Redis的最简单和最便捷的方法。Redisson的宗旨是促进使用者对Redis的关注分离（Separation of Concern），从而让使用者能够将精力更集中地放在处理业务逻辑上。

## 概述
官网解释如下：Redisson是一个在Redis的基础上实现的Java驻内存数据网格（In-Memory Data Grid）。它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务。其中包括(BitSet, Set, Multimap, SortedSet, Map, List, Queue, BlockingQueue, Deque, BlockingDeque, Semaphore, Lock, AtomicLong, CountDownLatch, Publish / Subscribe, Bloom filter, Remote service, Spring cache, Executor service, Live Object service, Scheduler service) Redisson提供了使用Redis的最简单和最便捷的方法。Redisson的宗旨是促进使用者对Redis的关注分离（Separation of Concern），从而让使用者能够将精力更集中地放在处理业务逻辑上。

## 用法
官网：wiki地址    https://github.com/redisson/redisson/wiki/%E7%9B%AE%E5%BD%95

```xml
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson</artifactId>
            <version>3.15.0</version>
        </dependency>
```

## 简单的使用

## 限流器
基于Redis的分布式限流器RateLimiter可以用来在分布式环境下现在请求方的调用频率。既适用于不同Redisson实例下的多线程限流，也适用于相同Redisson实例下的多线程限流。

RateLimter主要作用就是可以限制调用接口的次数。主要原理就是调用接口之前，需要拥有指定个令牌。限流器每秒会产生X个令牌放入令牌桶，调用接口需要去令牌桶里面拿令牌。如果令牌被其它请求拿完了，那么自然而然，当前请求就调用不到指定的接口。
```
RRateLimiter rateLimiter = redisson.getRateLimiter("myRateLimiter");
// 初始化
// 最大流速 = 每10秒钟产生1个令牌
rateLimiter.trySetRate(RateType.OVERALL, 1, 10, RateIntervalUnit.SECONDS);
//需要1个令牌
if(rateLimiter.tryAcquire(1)){
    //TODO:Do something 
}
```

### RRateLimiter的实现
接下来我们顺着tryAcquire()方法来看下它的实现方式，在RedissonRateLimiter类中，我们可以看到最底层的tryAcquireAsync()方法。
```
    private <T> RFuture<T> tryAcquireAsync(RedisCommand<T> command, Long value) {
        byte[] random = new byte[8];
        ThreadLocalRandom.current().nextBytes(random);
 
        return commandExecutor.evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
                "——————————————————————————————————————"
                + "这里是一大段lua代码"
                + "____________________________________",
                Arrays.asList(getRawName(), getValueName(), getClientValueName(), getPermitsName(), getClientPermitsName()),
                value, System.currentTimeMillis(), random);
    }
```
映入眼帘的就是一大段lua代码，其实这段Lua代码就是限流实现的核心，我把这段lua代码摘出来，并加了一些注释，我们来详细看下。
```
local rate = redis.call("hget", KEYS[1], "rate")  # 100 
local interval = redis.call("hget", KEYS[1], "interval")  # 3600000
local type = redis.call("hget", KEYS[1], "type")  # 0
assert(rate ~= false and interval ~= false and type ~= false, "RateLimiter is not initialized")
local valueName = KEYS[2]      # {xindoo.limiter}:value 用来存储剩余许可数量
local permitsName = KEYS[4]    # {xindoo.limiter}:permits 记录了所有许可发出的时间戳  
# 如果是单实例模式，name信息后面就需要拼接上clientId来区分出来了
if type == "1" then
    valueName = KEYS[3]        # {xindoo.limiter}:value:b474c7d5-862c-4be2-9656-f4011c269d54
    permitsName = KEYS[5]      # {xindoo.limiter}:permits:b474c7d5-862c-4be2-9656-f4011c269d54
end
# 对参数校验 
assert(tonumber(rate) >= tonumber(ARGV[1]), "Requested permits amount could not exceed defined rate")
# 获取当前还有多少许可 
local currentValue = redis.call("get", valueName)   
local res
# 如果有记录当前还剩余多少许可 
if currentValue ~= false then
    # 回收已过期的许可数量
    local expiredValues = redis.call("zrangebyscore", permitsName, 0, tonumber(ARGV[2]) - interval)
    local released = 0
    for i, v in ipairs(expiredValues) do
        local random, permits = struct.unpack("Bc0I", v)
        released = released + permits
    end
    # 清理已过期的许可记录
    if released > 0 then
        redis.call("zremrangebyscore", permitsName, 0, tonumber(ARGV[2]) - interval)
        if tonumber(currentValue) + released > tonumber(rate) then
            currentValue = tonumber(rate) - redis.call("zcard", permitsName)
        else
            currentValue = tonumber(currentValue) + released
        end
        redis.call("set", valueName, currentValue)
    end
    # ARGV  permit  timestamp  random， random是一个随机的8字节
    # 如果剩余许可不够，需要在res中返回下个许可需要等待多长时间 
    if tonumber(currentValue) < tonumber(ARGV[1]) then
        local firstValue = redis.call("zrange", permitsName, 0, 0, "withscores")
        res = 3 + interval - (tonumber(ARGV[2]) - tonumber(firstValue[2]))
    else
        redis.call("zadd", permitsName, ARGV[2], struct.pack("Bc0I", string.len(ARGV[3]), ARGV[3], ARGV[1]))
        # 减小可用许可量 
        redis.call("decrby", valueName, ARGV[1])
        res = nil
    end
else # 反之，记录到还有多少许可，说明是初次使用或者之前已记录的信息已经过期了，就将配置rate写进去，并减少许可数 
    redis.call("set", valueName, rate)
    redis.call("zadd", permitsName, ARGV[2], struct.pack("Bc0I", string.len(ARGV[3]), ARGV[3], ARGV[1]))
    redis.call("decrby", valueName, ARGV[1])
    res = nil
end
local ttl = redis.call("pttl", KEYS[1])
# 重置
if ttl > 0 then
    redis.call("pexpire", valueName, ttl)
    redis.call("pexpire", permitsName, ttl)
end
return res
```
首先用RRateLimiter有个name，在我代码中就是xx.limiter，用这个作为KEY你就可以在Redis中找到一个map，里面存储了limiter的工作模式(type)、可数量(rate)、时间窗口大小(interval)，这些都是在limiter创建时写入到的redis中的，在上面的lua代码中也使用到了。

其次还俩很重要的key，valueName和permitsName，其中在我的代码实现中valueName是{xx.limiter}:value ，它存储的是当前可用的许可数量。我代码中permitsName的具体值是{xx.limiter}:permits，它是一个zset，其中存储了当前所有的许可授权记录（含有许可授权时间戳），其中SCORE直接使用了时间戳，而VALUE中包含了8字节的随机值和许可的数量

{xx.limiter}:permits这个zset中存储了所有的历史授权记录。

那Redis是如何保证在分布式下这些限流信息数据的一致性的？答案是不需要保证，在这个场景下，信息天然就是一致性的。原因是Redis的单进程数据处理模型，在同一个Key下，所有的eval请求都是串行的，所有不需要考虑数据并发操作的问题。在这里，Redisson也使用了HashTag，保证所有的限流信息都存储在同一个Redis实例上。


## RedisClient
```
@Configuration
public class MyRedissonConfig {

    //注册RedissonClient对象
    @Bean(destroyMethod="shutdown")
    RedissonClient redisson() throws IOException {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://127.0.0.1:6379");
        RedissonClient redissonClient = Redisson.create(config);
        return redissonClient;
    }
}
```




























