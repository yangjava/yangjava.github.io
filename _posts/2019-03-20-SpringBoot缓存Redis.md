---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot缓存Redis

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

```
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
```
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
```
/**
 * 自定义缓存配置文件，继承 CachingConfigurerSupport
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

## 模板编程
除了使用注解，Spring boot集成 Redis 客户端jedis。封装Redis 连接池，以及操作模板，可以方便的显示的在方法的代码中处理缓存对象

```
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