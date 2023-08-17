---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot缓存caffeine

## Caffeine Cache
Caffeine是使用Java8对Guava缓存的重写版本，在Spring Boot 2.0中将取代Guava。如果出现Caffeine，CaffeineCacheManager将会自动配置。

### Maven依赖管理
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>2.6.0</version>
</dependency>
```

### 开启缓存的支持
使用@EnableCaching注解让Spring Boot开启对缓存的支持
```java

@SpringBootApplication
@EnableCaching// 开启缓存，需要显示的指定
public class SpringBootStudentCacheCaffeineApplication {
 
    public static void main(String[] args) {
        SpringApplication.run(SpringBootStudentCacheCaffeineApplication.class, args);
    }
}
```

### 配置文件
新增对缓存的特殊配置，如最大容量、过期时间等
```properties
spring.cache.cache-names=people
spring.cache.caffeine.spec=initialCapacity=50,maximumSize=500,expireAfterWrite=10s,refreshAfterWrite=5s
```

如果使用了refreshAfterWrite配置还必须指定一个CacheLoader，如：
```

/**
 * 必须要指定这个Bean，refreshAfterWrite=5s这个配置属性才生效
 *
 * @return
 */
@Bean
public CacheLoader<Object, Object> cacheLoader() {
 
    CacheLoader<Object, Object> cacheLoader = new CacheLoader<Object, Object>() {
 
        @Override
        public Object load(Object key) throws Exception {
            return null;
        }
 
        // 重写这个方法将oldValue值返回回去，进而刷新缓存
        @Override
        public Object reload(Object key, Object oldValue) throws Exception {
            return oldValue;
        }
    };
 
    return cacheLoader;
}
```

### Caffeine配置说明
```properties
initialCapacity=[integer]: 初始的缓存空间大小
maximumSize=[long]: 缓存的最大条数
maximumWeight=[long]: 缓存的最大权重
expireAfterAccess=[duration]: 最后一次写入或访问后经过固定时间过期
expireAfterWrite=[duration]: 最后一次写入后经过固定时间过期
refreshAfterWrite=[duration]: 创建缓存或者最近一次更新缓存后经过固定的时间间隔，刷新缓存
weakKeys: 打开key的弱引用
weakValues：打开value的弱引用
softValues：打开value的软引用
recordStats：开发统计功能
```
**注意：**
- refreshAfterWrite必须实现LoadingCache，跟expire的区别是，指定时间过后，expire是remove该key，下次访问是同步去获取返回新值，而refresh则是指定时间后，不会remove该key，下次访问会触发刷新，新值没有回来时返回旧值
- expireAfterWrite和expireAfterAccess同事存在时，以expireAfterWrite为准。
- maximumSize和maximumWeight不可以同时使用
- weakValues和softValues不可以同时使用

### SpringBoot 集成 Caffeine
直接引入 Caffeine 依赖，然后使用 Caffeine 方法实现缓存。
```java
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.concurrent.TimeUnit;

@Configuration
public class CacheConfig {

    @Bean
    public Cache caffeineCache() {
        return Caffeine.newBuilder()
                // 设置最后一次写入或访问后经过固定时间过期
                .expireAfterWrite(60, TimeUnit.SECONDS)
                // 初始的缓存空间大小
                .initialCapacity(100)
                // 缓存的最大条数
                .maximumSize(1000)
                .build();
    }

}
```
缓存的使用
```java

@Slf4j
@Service
public class UserInfoServiceImpl implements UserInfoService {

    /**
     * 模拟数据库存储数据
     */
    private HashMap userInfoMap = new HashMap<>();

    @Autowired
    Cache<String, Object> caffeineCache;

    @Override
    public void addUserInfo(UserInfo userInfo) {
        log.info("create");
        userInfoMap.put(userInfo.getId(), userInfo);
        // 加入缓存
        caffeineCache.put(String.valueOf(userInfo.getId()),userInfo);
    }

    @Override
    public UserInfo getByName(Integer id) {
        // 先从缓存读取
        caffeineCache.getIfPresent(id);
        UserInfo userInfo = (UserInfo) caffeineCache.asMap().get(String.valueOf(id));
        if (userInfo != null){
            return userInfo;
        }
        // 如果缓存中不存在，则从库中查找
        log.info("get");
        userInfo = userInfoMap.get(id);
        // 如果用户信息不为空，则加入缓存
        if (userInfo != null){
            caffeineCache.put(String.valueOf(userInfo.getId()),userInfo);
        }
        return userInfo;
    }

    @Override
    public UserInfo updateUserInfo(UserInfo userInfo) {
        log.info("update");
        if (!userInfoMap.containsKey(userInfo.getId())) {
            return null;
        }
        // 取旧的值
        UserInfo oldUserInfo = userInfoMap.get(userInfo.getId());
        // 替换内容
        if (!StringUtils.isEmpty(oldUserInfo.getAge())) {
            oldUserInfo.setAge(userInfo.getAge());
        }
        if (!StringUtils.isEmpty(oldUserInfo.getName())) {
            oldUserInfo.setName(userInfo.getName());
        }
        if (!StringUtils.isEmpty(oldUserInfo.getSex())) {
            oldUserInfo.setSex(userInfo.getSex());
        }
        // 将新的对象存储，更新旧对象信息
        userInfoMap.put(oldUserInfo.getId(), oldUserInfo);
        // 替换缓存中的值
        caffeineCache.put(String.valueOf(oldUserInfo.getId()),oldUserInfo);
        return oldUserInfo;
    }

    @Override
    public void deleteById(Integer id) {
        log.info("delete");
        userInfoMap.remove(id);
        // 从缓存中删除
        caffeineCache.asMap().remove(String.valueOf(id));
    }

}
```

### SpringBoot 集成 Caffeine
引入 Caffeine 和 Spring Cache 依赖，使用 SpringCache 注解方法实现缓存。
```java
@Configuration
public class CacheConfig {

    /**
     * 配置缓存管理器
     *
     * @return 缓存管理器
     */
    @Bean("caffeineCacheManager")
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
                // 设置最后一次写入或访问后经过固定时间过期
                .expireAfterAccess(60, TimeUnit.SECONDS)
                // 初始的缓存空间大小
                .initialCapacity(100)
                // 缓存的最大条数
                .maximumSize(1000));
        return cacheManager;
    }

}
```

```java

@Slf4j
@Service
@CacheConfig(cacheNames = "caffeineCacheManager")
public class UserInfoServiceImpl implements UserInfoService {

    /**
     * 模拟数据库存储数据
     */
    private HashMap userInfoMap = new HashMap<>();

    @Override
    @CachePut(key = "#userInfo.id")
    public void addUserInfo(UserInfo userInfo) {
        log.info("create");
        userInfoMap.put(userInfo.getId(), userInfo);
    }

    @Override
    @Cacheable(key = "#id")
    public UserInfo getByName(Integer id) {
        log.info("get");
        return userInfoMap.get(id);
    }

    @Override
    @CachePut(key = "#userInfo.id")
    public UserInfo updateUserInfo(UserInfo userInfo) {
        log.info("update");
        if (!userInfoMap.containsKey(userInfo.getId())) {
            return null;
        }
        // 取旧的值
        UserInfo oldUserInfo = userInfoMap.get(userInfo.getId());
        // 替换内容
        if (!StringUtils.isEmpty(oldUserInfo.getAge())) {
            oldUserInfo.setAge(userInfo.getAge());
        }
        if (!StringUtils.isEmpty(oldUserInfo.getName())) {
            oldUserInfo.setName(userInfo.getName());
        }
        if (!StringUtils.isEmpty(oldUserInfo.getSex())) {
            oldUserInfo.setSex(userInfo.getSex());
        }
        // 将新的对象存储，更新旧对象信息
        userInfoMap.put(oldUserInfo.getId(), oldUserInfo);
        // 返回新对象信息
        return oldUserInfo;
    }

    @Override
    @CacheEvict(key = "#id")
    public void deleteById(Integer id) {
        log.info("delete");
        userInfoMap.remove(id);
    }

} 
```

## 参考资料
https://mvnrepository.com/artifact/com.github.ben-manes.caffeine/caffeine 