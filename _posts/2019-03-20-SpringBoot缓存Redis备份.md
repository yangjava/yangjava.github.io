---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot缓存Redis
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



## Redis

### Redis 优势
- 分布式
- 性能极高 – Redis能读的速度是110000次/s,写的速度是81000次/s 。
- 丰富的数据类型 – Redis支持二进制案例的 Strings, Lists, Hashes, Sets 及 Ordered Sets 数据类型操作。
- 原子 – Redis的所有操作都是原子性的，意思就是要么成功执行要么失败完全不执行。单个操作是原子性的。多个操作也支持事务，即原子性，通过MULTI和EXEC指令包起来。
- 丰富的特性 – Redis还支持 publish/subscribe, 通知, key 过期等等特性

### 导入依赖

就只需要这一个依赖！不需要spring-boot-starter-cache
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
  </dependency>
```
当你导入这一个依赖时，SpringBoot的CacheManager就会使用RedisCache。

Redis使用模式使用pool2连接池，在需要时引用下面的依赖
```xml
<!-- https://mvnrepository.com/artifact/org.apache.commons/commons-pool2 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
    <version>2.6.2</version>
</dependency>
```
### 配置Redis

```properties
spring.redis.database=1 # Redis数据库索引（默认为0）
spring.redis.host=127.0.0.1 # Redis服务器地址
spring.redis.port=6379 # Redis服务器连接端口
spring.redis.password= # Redis服务器连接密码（默认为空）
spring.redis.pool.max-active=1000 # 连接池最大连接数（使用负值表示没有限制）
spring.redis.pool.max-wait=-1 # 连接池最大阻塞等待时间（使用负值表示没有限制）
spring.redis.pool.max-idle=10 # 连接池中的最大空闲连接
spring.redis.pool.min-idle=2 # 连接池中的最小空闲连接
spring.redis.timeout=0 # 连接超时时间（毫秒）
```
如果你的Redis这时候已经可以启动程序了。

### 装配

如果需要自定义缓存配置可以通过，继承CachingConfigurerSupport类，手动装配，如果一切使用默认配置可不必

装配序列化类型

```java
@Bean
public RedisTemplate<String, Object> redisTemplate(LettuceConnectionFactory connectionFactory) {
    // 配置redisTemplate
    RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
    redisTemplate.setConnectionFactory(connectionFactory);
    redisTemplate.setKeySerializer(new StringRedisSerializer());//key序列化
    redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());//value序列化
    redisTemplate.afterPropertiesSet();
    return redisTemplate;
}
```

装配过期时间
```java
 /**
     * 通过RedisCacheManager配置过期时间
     *
     * @param redisConnectionFactory
     * @return
     */
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1)); // 设置缓存有效期一小时
        return RedisCacheManager
                .builder(RedisCacheWriter.nonLockingRedisCacheWriter(redisConnectionFactory))
                .cacheDefaults(redisCacheConfiguration).build();
    }
```

一个比较完整的装配类 demo
```java
/**
 * 自定义缓存配置文件，继承 CachingConfigurerSupport
 * Created by huanl on 2017/8/22.
 */
@Configuration
@EnableCaching
public class RedisConfig extends CachingConfigurerSupport{
    public RedisConfig() {
        super();
    }

    /**
     * 指定使用哪一种缓存
     * @param redisTemplate
     * @return
     */
    @Bean
    public CacheManager cacheManager(RedisTemplate<?,?> redisTemplate) {
        RedisCacheManager rcm = new RedisCacheManager(redisTemplate);
        return rcm;
    }

    /**
     * 指定默认的key生成方式
     * @return
     */
    @Override
    public KeyGenerator keyGenerator() {
       KeyGenerator keyGenerator = new KeyGenerator() {
           @Override
           public Object generate(Object o, Method method, Object... objects) {
               StringBuilder sb = new StringBuilder();
               sb.append(o.getClass().getName());
               sb.append(method.getName());
               for (Object obj : objects) {
                   sb.append(obj.toString());
               }
               return sb.toString();
           }
       };
       return keyGenerator;
    }

    @Override
    public CacheResolver cacheResolver() {
        return super.cacheResolver();
    }

    @Override
    public CacheErrorHandler errorHandler() {
        return super.errorHandler();
    }

    /**
     * redis 序列化策略 ，通常情况下key值采用String序列化策略
     * StringRedisTemplate默认采用的是String的序列化策略，保存的key和value都是采用此策略序列化保存的。StringRedisSerializer
     * RedisTemplate默认采用的是JDK的序列化策略，保存的key和value都是采用此策略序列化保存的。JdkSerializationRedisSerializer
     * @param factory
     * @return
     */
    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory factory){
        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(factory);

//        // 使用Jackson2JsonRedisSerialize 替换默认序列化
//        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
//        ObjectMapper om = new ObjectMapper();
//        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
//        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
//        jackson2JsonRedisSerializer.setObjectMapper(om);
//
//
//        //设置value的序列化方式
//        redisTemplate.setValueSerializer(jackson2JsonRedisSerializer);
//        //设置key的序列化方式
//        redisTemplate.setKeySerializer(new StringRedisSerializer());
//        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
//        redisTemplate.setHashValueSerializer(jackson2JsonRedisSerializer);

        //使用fastJson作为默认的序列化方式
        GenericFastJsonRedisSerializer genericFastJsonRedisSerializer = new GenericFastJsonRedisSerializer();
        redisTemplate.setDefaultSerializer(genericFastJsonRedisSerializer);
        redisTemplate.setValueSerializer(genericFastJsonRedisSerializer);
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashValueSerializer(genericFastJsonRedisSerializer);
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.afterPropertiesSet();

        return redisTemplate;

    }

    /**
     * 转换返回的object为json
     * @return
     */
    @Bean
    public HttpMessageConverters fastJsonHttpMessageConverters(){
        // 1、需要先定义一个converter 转换器
        FastJsonHttpMessageConverter fastConverter = new FastJsonHttpMessageConverter();
        // 2、添加fastJson 的配置信息，比如：是否要格式化返回的json数据
        FastJsonConfig fastJsonConfig = new FastJsonConfig();
        fastJsonConfig.setSerializerFeatures(SerializerFeature.PrettyFormat);
        // 3、在convert 中添加配置信息
        fastConverter.setFastJsonConfig(fastJsonConfig);
        // 4、将convert 添加到converters当中
        HttpMessageConverter<?> converter = fastConverter;
        return new HttpMessageConverters(converter);
    }


}
```

模板编程
除了使用注解，Spring boot集成 Redis 客户端jedis。封装Redis 连接池，以及操作模板，可以方便的显示的在方法的代码中处理缓存对象

```java
@Autowired
private StringRedisTemplate stringRedisTemplate;//操作key-value都是字符串

@Autowired
private RedisTemplate redisTemplate;//操作key-value都是对象

@Autowired
private RedisCacheManager redisCacheManager;
/**
 *  Redis常见的五大数据类型：
 *  stringRedisTemplate.opsForValue();[String(字符串)]
 *  stringRedisTemplate.opsForList();[List(列表)]
 *  stringRedisTemplate.opsForSet();[Set(集合)]
 *  stringRedisTemplate.opsForHash();[Hash(散列)]
 *  stringRedisTemplate.opsForZSet();[ZSet(有序集合)]
 */
public void test(){
    stringRedisTemplate.opsForValue().append("msg","hello");
    Cache emp = redisCacheManager.getCache("emp");
    emp.put("111", "222");
}
```