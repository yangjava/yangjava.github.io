---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot 集成Redis
Redis是最常用的KV数据库，Spring 通过模板方式（RedisTemplate）提供了对Redis的数据查询和操作功能。

## SpringBoot客户端
Jedis是Redis的Java客户端，在SpringBoot 1.x版本中也是默认的客户端。在SpringBoot 2.x版本中默认客户端是Luttuce。

## Spring中的Template和RedisTemplate

Spring 通过模板方式（RedisTemplate）提供了对Redis的数据查询和操作功能。

### 什么是模板模式？
模板方法模式(Template pattern): 在一个方法中定义一个算法的骨架, 而将一些步骤延迟到子类中. 模板方法使得子类可以在不改变算法结构的情况下, 重新定义算法中的某些步骤。

**Spring中有哪些模板模式的设计？**
比如：jdbcTemplate, mongodbTemplate, elasticsearchTemplate...等等
**RedisTemplate对于Redis5种基础类型的操作？**

```java
redisTemplate.opsForValue(); // 操作字符串
redisTemplate.opsForHash(); // 操作hash
redisTemplate.opsForList(); // 操作list
redisTemplate.opsForSet(); // 操作set
redisTemplate.opsForZSet(); // 操作zset
```

**对HyperLogLogs（基数统计）类型的操作?**
```java
redisTemplate.opsForHyperLogLog();
```

**对geospatial (地理位置)类型的操作?**
```java
redisTemplate.opsForGeo();
```

**对于BitMap的操作？也是在opsForValue()方法返回类型ValueOperations中**
```java
Boolean setBit(K key, long offset, boolean value);
Boolean getBit(K key, long offset);
```

**对于Stream的操作？**
```java
redisTemplate.opsForStream();
```

## 基于RedisTemplate+Jedis访问Redis数据

### Maven依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <exclusions>
        <exclusion>
            <artifactId>lettuce-core</artifactId>
            <groupId>io.lettuce</groupId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
    <version>2.9.0</version>
</dependency>
```

### yml配置
```yaml
spring:
  redis:
    database: 0
    host: 127.0.0.1
    port: 6379
    password: test
    jedis:
      pool:
        min-idle: 0
        max-active: 8
        max-idle: 8
        max-wait: -1ms
    connect-timeout: 30000ms
```

### RedisConfig配置
通过@Bean的方式配置RedisTemplate，主要是设置RedisConnectionFactory以及各种类型数据的Serializer。

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

/**
 * Redis configuration.
 *
 * @author yangjingjing
 */
@Configuration
public class RedisConfig {

    /**
     * redis template.
     *
     * @param factory factory
     * @return RedisTemplate
     */
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.afterPropertiesSet();
        return template;
    }
}

```

### RedisTemplate的使用

```java
import io.swagger.annotations.ApiOperation;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.web.bind.annotation.*;
import com.demo.springboot.redis.jedis.entity.User;
import com.demo.springboot.redis.jedis.entity.response.ResponseResult;

import javax.annotation.Resource;

/**
 * @author yangjingjing
 */
@RestController
@RequestMapping("/user")
public class UserController {

    // 注意：这里@Autowired是报错的，因为@Autowired按照类名注入的
    @Resource
    private RedisTemplate<String, User> redisTemplate;

    /**
     * @param user user param
     * @return user
     */
    @ApiOperation("Add")
    @PostMapping("add")
    public ResponseResult<User> add(User user) {
        redisTemplate.opsForValue().set(String.valueOf(user.getId()), user);
        return ResponseResult.success(redisTemplate.opsForValue().get(String.valueOf(user.getId())));
    }

    /**
     * @return user list
     */
    @ApiOperation("Find")
    @GetMapping("find/{userId}")
    public ResponseResult<User> edit(@PathVariable("userId") String userId) {
        return ResponseResult.success(redisTemplate.opsForValue().get(userId));
    }

}
```

## Lettuce简介

### 什么是Lettuce?
Lettuce 是一个可伸缩线程安全的 Redis 客户端。多个线程可以共享同一个 RedisConnection。它利用优秀 netty NIO 框架来高效地管理多个连接

Lettuce的特性：
- 支持 同步、异步、响应式 的方式
- 支持 Redis Sentinel
- 支持 Redis Cluster
- 支持 SSL 和 Unix Domain Socket 连接
- 支持 Streaming API
- 支持 CDI 和 Spring 的集成
- 支持 Command Interfaces兼容 Java 8+ 以上版本

### 为何SpringBoot2.x中Lettuce会成为默认的客户端
**线程共享**
Jedis 是直连模式，在多个线程间共享一个 Jedis 实例时是线程不安全的，如果想要在多线程环境下使用 Jedis，需要使用连接池，每个线程都去拿自己的 Jedis 实例，当连接数量增多时，物理连接成本就较高了。

Lettuce 是基于 netty 的，连接实例可以在多个线程间共享，所以，一个多线程的应用可以使用一个连接实例，而不用担心并发线程的数量。

**异步和反应式**
Lettuce 从一开始就按照非阻塞式 IO 进行设计，是一个纯异步客户端，对异步和反应式 API 的支持都很全面。即使是同步命令，底层的通信过程仍然是异步模型，只是通过阻塞调用线程来模拟出同步效果而已。

## Lettuce的基本的API方式

### 依赖POM
```xml
<dependency>
  <groupId>io.lettuce</groupId>
  <artifactId>lettuce-core</artifactId>
  <version>x.y.z.BUILD-SNAPSHOT</version>
</dependency>

```

### 基础用法
```java
RedisClient client = RedisClient.create("redis://localhost");
StatefulRedisConnection<String, String> connection = client.connect();
RedisStringCommands sync = connection.sync();
String value = sync.get("key");
```

### 异步方式
```java
StatefulRedisConnection<String, String> connection = client.connect();
RedisStringAsyncCommands<String, String> async = connection.async();
RedisFuture<String> set = async.set("key", "value")
RedisFuture<String> get = async.get("key")

async.awaitAll(set, get) == true

set.get() == "OK"
get.get() == "value"

```

### 响应式
```java
StatefulRedisConnection<String, String> connection = client.connect();
RedisStringReactiveCommands<String, String> reactive = connection.reactive();
Mono<String> set = reactive.set("key", "value");
Mono<String> get = reactive.get("key");

set.subscribe();

get.block() == "value"

```

## 基于RedisTemplate+Lettuce访问Redis数据
引入spring-boot-starter-data-redis包，SpringBoot2中默认的客户端是Lettuce。
### Lettuce Maven依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
    <version>2.9.0</version>
</dependency>
```
### yml配置
常用的Lettuce的使用配置
```yaml
spring:
  redis:
    database: 0
    host: 127.0.0.1
    port: 6379
    password: test
    lettuce:
      pool:
        min-idle: 0
        max-active: 8
        max-idle: 8
        max-wait: -1ms
    connect-timeout: 30000ms
```

## RedisConfig配置
通过@Bean的方式配置RedisTemplate，主要是设置RedisConnectionFactory以及各种类型数据的Serializer。
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

/**
 * Redis configuration.
 *
 * @author yangjingjing
 */
@Configuration
public class RedisConfig {

    /**
     * redis template.
     *
     * @param factory factory
     * @return RedisTemplate
     */
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.afterPropertiesSet();
        return template;
    }
}

```

### RedisTemplate的使用
```java
import io.swagger.annotations.ApiOperation;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.web.bind.annotation.*;
import com.demo.springboot.redis.lettuce.entity.User;
import com.demo.springboot.redis.lettuce.entity.response.ResponseResult;

import javax.annotation.Resource;

/**
 * @author yangjingjing
 */
@RestController
@RequestMapping("/user")
public class UserController {

    // 注意：这里@Autowired是报错的，因为@Autowired按照类名注入的
    @Resource
    private RedisTemplate<String, User> redisTemplate;

    /**
     * @param user user param
     * @return user
     */
    @ApiOperation("Add")
    @PostMapping("add")
    public ResponseResult<User> add(User user) {
        redisTemplate.opsForValue().set(String.valueOf(user.getId()), user);
        return ResponseResult.success(redisTemplate.opsForValue().get(String.valueOf(user.getId())));
    }

    /**
     * @return user list
     */
    @ApiOperation("Find")
    @GetMapping("find/{userId}")
    public ResponseResult<User> edit(@PathVariable("userId") String userId) {
        return ResponseResult.success(redisTemplate.opsForValue().get(userId));
    }

}

```

## 基于RedisTemplate+Lettuce数据类封装
RedisTemplate中的操作和方法众多，为了程序保持方法使用的一致性，屏蔽一些无关的方法以及对使用的方法进一步封装。

### RedisService封装
```java
import org.springframework.data.redis.core.RedisCallback;

import java.util.Collection;
import java.util.Set;

/**
 * Redis Service.
 *
 * @author pdai
 */
public interface IRedisService<T> {

    void set(String key, T value);

    void set(String key, T value, long time);

    T get(String key);

    void delete(String key);

    void delete(Collection<String> keys);

    boolean expire(String key, long time);

    Long getExpire(String key);

    boolean hasKey(String key);

    Long increment(String key, long delta);

    Long decrement(String key, long delta);

    void addSet(String key, T value);

    Set<T> getSet(String key);

    void deleteSet(String key, T value);

    T execute(RedisCallback<T> redisCallback);
}
```

### RedisService的实现类

```java
import org.springframework.data.redis.core.RedisCallback;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import tech.pdai.springboot.redis.lettuce.enclosure.service.IRedisService;

import javax.annotation.Resource;
import java.util.Collection;
import java.util.Set;
import java.util.concurrent.TimeUnit;

/**
 * @author pdai
 */
@Service
public class RedisServiceImpl<T> implements IRedisService<T> {

    @Resource
    private RedisTemplate<String, T> redisTemplate;

    @Override
    public void set(String key, T value, long time) {
        redisTemplate.opsForValue().set(key, value, time, TimeUnit.SECONDS);
    }

    @Override
    public void set(String key, T value) {
        redisTemplate.opsForValue().set(key, value);
    }

    @Override
    public T get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    @Override
    public void delete(String key) {
        redisTemplate.delete(key);
    }

    @Override
    public void delete(Collection<String> keys) {
        redisTemplate.delete(keys);
    }

    @Override
    public boolean expire(String key, long time) {
        return redisTemplate.expire(key, time, TimeUnit.SECONDS);
    }

    @Override
    public Long getExpire(String key) {
        return redisTemplate.getExpire(key, TimeUnit.SECONDS);
    }

    @Override
    public boolean hasKey(String key) {
        return redisTemplate.hasKey(key);
    }

    @Override
    public Long increment(String key, long delta) {
        return redisTemplate.opsForValue().increment(key, delta);
    }

    @Override
    public Long decrement(String key, long delta) {
        return redisTemplate.opsForValue().increment(key, -delta);
    }

    @Override
    public void addSet(String key, T value) {
        redisTemplate.opsForSet().add(key, value);
    }

    @Override
    public Set<T> getSet(String key) {
        return redisTemplate.opsForSet().members(key);
    }

    @Override
    public void deleteSet(String key, T value) {
        redisTemplate.opsForSet().remove(key, value);
    }

    @Override
    public T execute(RedisCallback<T> redisCallback) {
        return redisTemplate.execute(redisCallback);
    }

}

```

### RedisService的调用
```java

import io.swagger.annotations.ApiOperation;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import com.demo.springboot.redis.lettuce.enclosure.entity.User;
import com.demo.springboot.redis.lettuce.enclosure.entity.response.ResponseResult;
import com.demo.springboot.redis.lettuce.enclosure.service.IRedisService;

/**
 * @author pdai
 */
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private IRedisService<User> redisService;

    /**
     * @param user user param
     * @return user
     */
    @ApiOperation("Add")
    @PostMapping("add")
    public ResponseResult<User> add(User user) {
        redisService.set(String.valueOf(user.getId()), user);
        return ResponseResult.success(redisService.get(String.valueOf(user.getId())));
    }

    /**
     * @return user list
     */
    @ApiOperation("Find")
    @GetMapping("find/{userId}")
    public ResponseResult<User> edit(@PathVariable("userId") String userId) {
        return ResponseResult.success(redisService.get(userId));
    }

}

```