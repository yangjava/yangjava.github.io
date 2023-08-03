---
layout: post
categories: [SpringBoot,Cache,Caffeine,Ehcache]
description: none
keywords: SpringBoot
---
# SpringBoot 集成缓存
Spring 从 3.1 开始就引入了对 Cache 的支持。定义了 org.springframework.cache.Cache 和 org.springframework.cache.CacheManager 接口来统一不同的缓存技术。并支持使用 JCache（JSR-107）注解简化我们的开发。
Spring Cache 是作用在方法上的，其核心思想是，当我们在调用一个缓存方法时会把该方法参数和返回结果作为一个键值对存在缓存中。


## Cache 和 CacheManager 接口说明
- Cache 接口包含缓存的各种操作集合，你操作缓存就是通过这个接口来操作的。
- Cache 接口下 Spring 提供了各种 xxxCache 的实现，比如：RedisCache、EhCache、ConcurrentMapCache
- CacheManager 定义了创建、配置、获取、管理和控制多个唯一命名的 Cache。这些 Cache 存在于 CacheManager 的上下文中。


## 缓存技术类型与CacheManger

针对不同的缓存技术，需要实现不同的cacheManager，Spring定义了如下的cacheManger实现。

| CacheManger               | 描述                                                         |
|---------------------------|------------------------------------------------------------|
| SimpleCacheManager        | 使用简单的Collection来存储缓存，主要用于测试                                |
| ConcurrentMapCacheManager | 使用ConcurrentMap作为缓存技术（默认），需要显式的删除缓存，无过期机制                  |
| NoOpCacheManager          | 仅测试用途，不会实际存储缓存                                             |
| EhCacheCacheManager       | 使用EhCache作为缓存技术，以前在hibernate的时候经常用                         |
| GuavaCacheManager         | 使用google guava的GuavaCache作为缓存技术(1.5版本已不建议使用）               |
| CaffeineCacheManager      | 是使用Java8对Guava缓存的重写，spring5（springboot2）开始用Caffeine取代guava |
| HazelcastCacheManager     | 使用Hazelcast作为缓存技术                                          |
| JCacheCacheManager        | 使用JCache标准的实现作为缓存技术，如Apache Commons JCS                    |
| RedisCacheManager         | 使用Redis作为缓存技术                                              |

常规的SpringBoot已经为我们自动配置了EhCache、Collection、Guava、ConcurrentMap等缓存，默认使用ConcurrentMapCacheManager。SpringBoot的application.properties配置文件，使用spring.cache前缀的属性进行配置。

## 缓存依赖
开始使用前需要导入依赖

spring-boot-starter-cache 为基础依赖，其他依赖根据使用不同的缓存技术选择加入，默认情况下使用 ConcurrentMapCache不需要引用任何依赖	
	
```xml
        <!-- 基础依赖 -->　　　　
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        <!-- 使用 ehcache -->
        <dependency>
            <groupId>net.sf.ehcache</groupId>
            <artifactId>ehcache</artifactId>
        </dependency>
        <!-- 使用  caffeine https://mvnrepository.com/artifact/com.github.ben-manes.caffeine/caffeine -->
        <dependency>
            <groupId>com.github.ben-manes.caffeine</groupId>
            <artifactId>caffeine</artifactId>
            <version>2.6.0</version>
        </dependency>
        <!-- 使用  redis  -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        
```

## application配置	

```properties
spring.cache.type= ＃缓存的技术类型，可选 generic,ehcache,hazelcast,infinispan,jcache,redis,guava,simple,none
spring.cache.cache-names= ＃应用程序启动创建缓存的名称，必须将所有注释为@Cacheable缓存name（或value）罗列在这里，否者：Cannot find cache named 'xxx' for Builder[xx] caches=[sysItem] | key='' | keyGenerator='' | cacheManager='' | cacheResolver='' | condition='' | unless='' | sync='false'
#以下根据不同缓存技术选择配置
spring.cache.ehcache.config= ＃EHCache的配置文件位置
spring.caffeine.spec= ＃caffeine类型创建缓存的规范。查看CaffeineSpec了解更多关于规格格式的细节
spring.cache.infinispan.config= ＃infinispan的配置文件位置
spring.cache.jcache.config= ＃jcache配置文件位置
spring.cache.jcache.provider= ＃当多个jcache实现类时，指定选择jcache的实现类
```

## 缓存注解
下面是缓存公用接口注释，使用与任何缓存技术
## @EnableCaching
@EnableCaching：在启动类注解@EnableCaching开启缓存　　
```java
@SpringBootApplication
@EnableCaching  //开启缓存
public class DemoApplication{

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}　
```

## @Cacheable	
@Cacheable 注解就可以将运行结果缓存，以后查询相同的数据，直接从缓存中取，不需要调用方法。

该注解主要有下面几个参数：
- value、cacheNames：两个等同的参数（cacheNames为Spring 4新增，作为value的别名），用于指定缓存存储的集合名。由于Spring 4中新增了@CacheConfig，因此在Spring 3中原本必须有的value属性，也成为非必需项了
- key：缓存对象存储在Map集合中的key值，非必需，缺省按照函数的所有参数组合作为key值，若自己配置需使用SpEL表达式，比如：@Cacheable(key = “#p0”)：使用函数第一个参数作为缓存的key值
- condition：缓存对象的条件，非必需，也需使用SpEL表达式，只有满足表达式条件的内容才会被缓存，比如：@Cacheable(key = “#p0”, condition = “#p0.length() < 3”)，表示只有当第一个参数的长度小于3的时候才会被缓存
- unless：另外一个缓存条件参数，非必需，需使用SpEL表达式。它不同于condition参数的地方在于它的判断时机，该条件是在函数被调用之后才做判断的，所以它可以通过对result进行判断。
- keyGenerator：用于指定key生成器，非必需。若需要指定一个自定义的key生成器，我们需要去实现org.springframework.cache.interceptor.KeyGenerator接口，并使用该参数来指定。需要注意的是：该参数与key是互斥的
- cacheManager：用于指定使用哪个缓存管理器，非必需。只有当有多个时才需要使用
- cacheResolver：用于指定使用那个缓存解析器，非必需。需通过org.springframework.cache.interceptor.CacheResolver接口来实现自己的缓存解析器，并用该参数指定。

### cacheNames & value
@Cacheable 提供两个参数来指定缓存名：value、cacheNames，二者选其一即可。这是 @Cacheable 最简单的用法示例：
```java
@Override
@Cacheable("menu")
public Menu findById(String id) {
    Menu menu = this.getById(id);
    if (menu != null){
        System.out.println("menu.name = " + menu.getName());
    }
    return menu;
}

```
在这个例子中，findById 方法与一个名为 menu 的缓存关联起来了。调用该方法时，会检查 menu 缓存，如果缓存中有结果，就不会去执行方法了。
### 关联多个缓存名
其实，按照官方文档，@Cacheable 支持同一个方法关联多个缓存。这种情况下，当执行方法之前，这些关联的每一个缓存都会被检查，而且只要至少其中一个缓存命中了，那么这个缓存中的值就会被返回。示例：
```
@Override
    @Cacheable({"menu", "menuById"})
    public Menu findById(String id) {
        Menu menu = this.getById(id);
        if (menu != null){
            System.out.println("menu.name = " + menu.getName());
        }
        return menu;
    }
 
---------
@GetMapping("/findById/{id}")
public Menu findById(@PathVariable("id")String id){
    Menu menu0 = menuService.findById("fe278df654adf23cf6687f64d1549c0a");
    Menu menu2 = menuService.findById("fb6106721f289ebf0969565fa8361c75");
    return menu0;
}
```
### key & keyGenerator
一个缓存名对应一个被注解的方法，但是一个方法可能传入不同的参数，那么结果也就会不同，这应该如何区分呢？这就需要用到 key 。在 spring 中，key 的生成有两种方式：显式指定和使用 keyGenerator 自动生成。
#### KeyGenerator 自动生成
当我们在声明 @Cacheable 时不指定 key 参数，则该缓存名下的所有 key 会使用 KeyGenerator 根据参数 自动生成。spring 有一个默认的 SimpleKeyGenerator ，在 spring boot 自动化配置中，这个会被默认注入。生成规则如下：
- 如果该缓存方法没有参数，返回 SimpleKey.EMPTY ；
- 如果该缓存方法有一个参数，返回该参数的实例 ；
- 如果该缓存方法有多个参数，返回一个包含所有参数的 SimpleKey ；
默认的 key 生成器要求参数具有有效的 hashCode() 和 equals() 方法实现。另外，keyGenerator 也支持自定义， 并通过 keyGenerator 来指定。关于 KeyGenerator 这里不做详细介绍，有兴趣的话可以去看看源码，其实就是使用 hashCode 进行加乘运算。跟 String 和 ArrayList 的 hash 计算类似。
#### 显式指定 key
相较于使用 KeyGenerator 生成，spring 官方更推荐显式指定 key 的方式，即指定 @Cacheable 的 key 参数。
即便是显式指定，但是 key 的值还是需要根据参数的不同来生成，那么如何实现动态拼接呢？SpEL（Spring Expression Language，Spring 表达式语言） 能做到这一点。下面是一些使用 SpEL 生成 key 的例子。
```java
@Override
    @Cacheable(value = {"menuById"}, key = "#id")
    public Menu findById(String id) {
        Menu menu = this.getById(id);
        if (menu != null){
            System.out.println("menu.name = " + menu.getName());
        }
        return menu;
    }
 
    @Override
    @Cacheable(value = {"menuById"}, key = "'id-' + #menu.id")
    public Menu findById(Menu menu) {
        return menu;
    }
 
    @Override
    @Cacheable(value = {"menuById"}, key = "'hash' + #menu.hashCode()")
    public Menu findByHash(Menu menu) {
        return menu;
    }
```
注意：官方说 key 和 keyGenerator 参数是互斥的，同时指定两个会导致异常。

### cacheManager & cacheResolver
CacheManager，缓存管理器是用来管理（检索）一类缓存的。通常来讲，缓存管理器是与缓存组件类型相关联的。我们知道，spring 缓存抽象的目的是为使用不同缓存组件类型提供统一的访问接口，以向开发者屏蔽各种缓存组件的差异性。那么  CacheManager 就是承担了这种屏蔽的功能。spring 为其支持的每一种缓存的组件类型提供了一个默认的 manager，如：RedisCacheManager 管理 redis 相关的缓存的检索、EhCacheManager 管理 ehCache 相关的缓等。

CacheResolver，缓存解析器是用来管理缓存管理器的，CacheResolver 保持一个 cacheManager 的引用，并通过它来检索缓存。CacheResolver 与 CacheManager 的关系有点类似于 KeyGenerator 跟 key。spring 默认提供了一个 SimpleCacheResolver，开发者可以自定义并通过 @Bean 来注入自定义的解析器，以实现更灵活的检索。

大多数情况下，我们的系统只会配置一种缓存，所以我们并不需要显式指定 cacheManager 或者 cacheResolver。但是 spring 允许我们的系统同时配置多种缓存组件，这种情况下，我们需要指定。指定的方式是使用 @Cacheable 的 cacheManager 或者 cacheResolver 参数。

注意：按照官方文档，cacheManager 和 cacheResolver 是互斥参数，同时指定两个可能会导致异常。

### sync
是否同步，true/false。在一个多线程的环境中，某些操作可能被相同的参数并发地调用，这样同一个 value 值可能被多次计算（或多次访问 db），这样就达不到缓存的目的。针对这些可能高并发的操作，我们可以使用 sync 参数来告诉底层的缓存提供者将缓存的入口锁住，这样就只能有一个线程计算操作的结果值，而其它线程需要等待，这样就避免了 n-1 次数据库访问。
sync = true 可以有效的避免缓存击穿的问题。
### condition
调用前判断，缓存的条件。有时候，我们可能并不想对一个方法的所有调用情况进行缓存，我们可能想要根据调用方法时候的某些参数值，来确定是否需要将结果进行缓存或者从缓存中取结果。比如当我根据年龄查询用户的时候，我只想要缓存年龄大于 35 的查询结果。那么 condition 能实现这种效果。
condition 接收一个结果为 true 或 false 的表达式，表达式同样支持 SpEL 。如果表达式结果为 true，则调用方法时会执行正常的缓存逻辑（查缓存-有就返回-没有就执行方法-方法返回不空就添加缓存）；否则，调用方法时就好像该方法没有声明缓存一样（即无论传入了什么参数或者缓存中有些什么值，都会执行方法，并且结果不放入缓存）
### unless
执行后判断，不缓存的条件。unless 接收一个结果为 true 或 false 的表达式，表达式支持 SpEL。当结果为 true 时，不缓存。
可以看到，两次调用的结果都没有缓存。说明在这种情况下，condition 比 unless 的优先级高。总结起来就是：
condition 不指定相当于 true，unless 不指定相当于 false
当 condition = false，一定不会缓存；
当 condition = true，且 unless = true，不缓存；
当 condition = true，且 unless = false，缓存；

## @CachePut
@CachePut：主要针对方法配置，能够根据方法的请求参数对其结果进行缓存，和 @Cacheable 不同的是，它每次都会触发真实方法的调用 。简单来说就是用户更新缓存数据。但需要注意的是该注解的value 和 key 必须与要更新的缓存相同，也就是与@Cacheable 相同。示例：
```java
    @CachePut(value = "newJob", key = "#p0")  //按条件更新缓存
    public NewJob updata(NewJob job) {
        NewJob newJob = newJobDao.findAllById(job.getId());
        newJob.updata(job);
        return job;
    }
```

## @CacheEvict
@CacheEvict：配置于函数上，通常用在删除方法上，用来从缓存中移除相应数据。除了同@Cacheable一样的参数之外，它还有下面两个参数：
- allEntries：非必需，默认为false。当为true时，会移除所有数据。如：@CachEvict(value=”testcache”,allEntries=true)
- beforeInvocation：非必需，默认为false，会在调用方法之后移除数据。当为true时，会在调用方法之前移除数据。 如：@CachEvict(value=”testcache”，beforeInvocation=true)

```java
    @Cacheable(value = "emp",key = "#p0.id")
    public NewJob save(NewJob job) {
        newJobDao.save(job);
        return job;
    }

    //清除一条缓存，key为要清空的数据
    @CacheEvict(value="emp",key="#id")
    public void delect(int id) {
        newJobDao.deleteAllById(id);
    }

    //方法调用后清空所有缓存
    @CacheEvict(value="accountCache",allEntries=true)
    public void delectAll() {
        newJobDao.deleteAll();
    }

    //方法调用前清空所有缓存
    @CacheEvict(value="accountCache",beforeInvocation=true)
    public void delectAll() {
        newJobDao.deleteAll();
    }
```

### @CacheConfig
@CacheConfig： 统一配置本类的缓存注解的属性，在类上面统一定义缓存的名字，方法上面就不用标注了，当标记在一个类上时则表示该类所有的方法都是支持缓存的 

```java
@CacheConfig(cacheNames = {"myCache"})
public class BotRelationServiceImpl implements BotRelationService {
    @Override
    @Cacheable(key = "targetClass + methodName +#p0")//此处没写value
    public List<BotRelation> findAllLimit(int num) {
        return botRelationRepository.findAllLimit(num);
    }
    .....
}
```

## ConcurrentMap Cache
Spring boot默认使用的是SimpleCacheConfiguration，即使用ConcurrentMapCacheManager来实现缓存，ConcurrentMapCache实质是一个ConcurrentHashMap集合对象java内置，所以无需引入其他依赖，也没有额外的配置
ConcurrentMapCache的自动装配声明在SimpleCacheConfiguration中，如果需要也可对它进行额外的装配

```java
//注册1个id为cacheManager,类型为ConcurrentMapCacheManager的bean
@Bean
public ConcurrentMapCacheManager cacheManager() {
    ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager(); //实例化ConcurrentMapCacheManager
    List<String> cacheNames = this.cacheProperties.getCacheNames(); //读取配置文件，如果配置有spring.cache.cache-names=xx,xx,则进行配置cacheNames,默认是没有配置的
    if (!cacheNames.isEmpty()) {
        cacheManager.setCacheNames(cacheNames);
    }
    return this.customizerInvoker.customize(cacheManager); //调用CacheManagerCustomizers#customize 进行个性化设置,在该方法中是遍历其持有的List
}
```

## Caffeine Cache
Caffeine是使用Java8对Guava缓存的重写版本，在Spring Boot 2.0中将取代Guava。如果出现Caffeine，CaffeineCacheManager将会自动配置。

### Maven依赖管理
```xml
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
```java

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

## EHCache
EhCache 是一个纯Java的进程内缓存框架，具有快速、精干等特点，是Hibernate中默认CacheProvider。Ehcache是一种广泛使用的开源Java分布式缓存。主要面向通用缓存,Java EE和轻量级容器。它具有内存和磁盘存储，缓存加载器,缓存扩展,缓存异常处理程序,一个gzip缓存servlet过滤器,支持REST和SOAP api等特点。

### 导入依赖
引入springboot-cache和ehcache。需要注意，EhCache不需要配置version，SpringBoot的根pom已经集成了。
```xml
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

### 加入配置

```properties
spring.cache.type=ehcache # 配置ehcache缓存
spring.cache.ehcache.config=classpath:/ehcache.xml # 指定ehcache配置文件路径 ，可以不用写，因为默认就是这个路径，SpringBoot会自动扫描
```

### ehcache配置文件

EhCache的配置文件ehcache.xml只需要放到类路径下面，SpringBoot会自动扫描

```xml
<ehcache>

    <!--
        磁盘存储:指定一个文件目录，当EHCache把数据写到硬盘上时，将把数据写到这个文件目录下
        path:指定在硬盘上存储对象的路径
        path可以配置的目录有：
            user.home（用户的家目录）
            user.dir（用户当前的工作目录）
            java.io.tmpdir（默认的临时目录）
            ehcache.disk.store.dir（ehcache的配置目录）
            绝对路径（如：d:\\ehcache）
        查看路径方法：String tmpDir = System.getProperty("java.io.tmpdir");
     -->
    <diskStore path="java.io.tmpdir" />

    <!--
        defaultCache:默认的缓存配置信息,如果不加特殊说明,则所有对象按照此配置项处理
        maxElementsInMemory:设置了缓存的上限,最多存储多少个记录对象
        eternal:代表对象是否永不过期 (指定true则下面两项配置需为0无限期)
        timeToIdleSeconds:最大的发呆时间 /秒
        timeToLiveSeconds:最大的存活时间 /秒
        overflowToDisk:是否允许对象被写入到磁盘
        说明：下列配置自缓存建立起600秒(10分钟)有效 。
        在有效的600秒(10分钟)内，如果连续120秒(2分钟)未访问缓存，则缓存失效。
        就算有访问，也只会存活600秒。
     -->
    <defaultCache maxElementsInMemory="10000" eternal="false"
                  timeToIdleSeconds="600" timeToLiveSeconds="600" overflowToDisk="true" />
    <!-- 按缓存名称的不同管理策略 -->
    <cache name="myCache" maxElementsInMemory="10000" eternal="false"
                  timeToIdleSeconds="120" timeToLiveSeconds="600" overflowToDisk="true" />

</ehcache>
```

### 装配

SpringBoot会为我们自动配置 EhCacheCacheManager 这个Bean，如果想自定义设置一些个性化参数时，通过Java Config形式配置。
```java
@Configuration  
@EnableCaching
public class CacheConfig {  
  
    @Bean  
    public CacheManager cacheManager() {  
        return new EhCacheCacheManager(ehCacheCacheManager().getObject());  
    }  
  
    @Bean  
    public EhCacheManagerFactoryBean ehCacheCacheManager() {  
        EhCacheManagerFactoryBean cmfb = new EhCacheManagerFactoryBean();  
        cmfb.setConfigLocation(new ClassPathResource("ehcache.xml"));  
        cmfb.setShared(true);  
        return cmfb;  
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
