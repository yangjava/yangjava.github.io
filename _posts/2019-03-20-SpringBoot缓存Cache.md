---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot使用缓存
Spring Boot 使用简单缓存非常简单，我们只需要导入所需要的包即可开箱即用，如果我们仅仅想使用缓存，则直接引入：org.springframework.boot:spring-boot-starter-cache starter 便可使用。

## 使用缓存
Spring 从 3.1 开始就引入了对 Cache 的支持。定义了 org.springframework.cache.Cache 和 org.springframework.cache.CacheManager 接口来统一不同的缓存技术。 并支持使用 JCache（JSR-107）注解简化我们的开发。

Spring Cache 是作用在方法上的，其核心思想是，当我们在调用一个缓存方法时会把**该方法参数和返回结果**作为一个键值对存在缓存中。

在配置类或启动类上加入 @EnableCaching 注解，该注解会触发一个后处理器（post processor）去检测每个 Spring Bean 上是否存在公共方法的缓存注解。如果找到这样的注解，则自动创建代理以拦截方法调用并相应地处理缓存行为。

此后处理器管理的注解是 Cacheable，CachePut 和 CacheEvict。Spring Boot 会自动配置合适的 CacheManager 作为相关缓存的供应商。如果只引入了该包，则默认只会使用 Spring 上下文 ConcurrentHashMap 结构来存储缓存，这完全符合 JCache 的标准。

如果当前上下文中存在 JSR-107 API，即 javax.cache:cache-api 该 jar 包，将额外的为 JSR-107 API 注解的 bean 创建代理，这些注解是 @CacheResult，@CachePut，@CacheRemove 和 @CacheRemoveAll。

每次调用需要缓存功能的方法时，Spring 会检查指定参数的指定目标方法是否已经被调用过，如果有就直接从缓存中获取方法调用后的结果，如果没有就调用方法并缓存结果后返回给用户。下次调用直接从缓存中获取。

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
开始使用前需要导入依赖 spring-boot-starter-cache 为基础依赖，其他依赖根据使用不同的缓存技术选择加入，

默认情况下使用 ConcurrentMapCache不需要引用任何依赖		
```
        <!-- 基础缓存依赖 -->　　　　
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        
        <!-- 使用 ehcache -->
        <dependency>
            <groupId>net.sf.ehcache</groupId>
            <artifactId>ehcache</artifactId>
        </dependency>
        
        <!-- 使用  caffeine -->
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

## 启用缓存@EnableCaching
现在大部分项目都是是SpringBoot项目，我们可以在启动类添加注解`@EnableCaching`来开启缓存功能。
　
```
@SpringBootApplication
@EnableCaching  //开启缓存
public class DemoApplication{

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}　
```
既然要能使用缓存，就需要有一个缓存管理器Bean，默认情况下，@EnableCaching 将注册一个ConcurrentMapCacheManager的Bean，不需要单独的 bean 声明。

ConcurrentMapCacheManager将值存储在ConcurrentHashMap的实例中，这是缓存机制的最简单的线程安全实现。

## @Cacheable
`@Cacheable`该注解可以将方法运行的结果进行缓存，在缓存时效内再次调用该方法时不会调用方法本身，而是直接从缓存获取结果并返回给调用方。

该注解主要有下面几个参数：
- value、cacheNames
两个等同的参数（cacheNames为Spring 4新增，作为value的别名），用于指定缓存存储的集合名。由于Spring 4中新增了@CacheConfig，因此在Spring 3中原本必须有的value属性，也成为非必需项了

- key
缓存对象存储在Map集合中的key值，非必需，缺省按照函数的所有参数组合作为key值，若自己配置需使用SpEL表达式，比如：@Cacheable(key = “#p0”)：使用函数第一个参数作为缓存的key值

- condition
缓存对象的条件，非必需，也需使用SpEL表达式，只有满足表达式条件的内容才会被缓存，比如：@Cacheable(key = “#p0”, condition = “#p0.length() < 3”)，表示只有当第一个参数的长度小于3的时候才会被缓存

- unless
另外一个缓存条件参数，非必需，需使用SpEL表达式。它不同于condition参数的地方在于它的判断时机，该条件是在函数被调用之后才做判断的，所以它可以通过对result进行判断。

- keyGenerator
用于指定key生成器，非必需。若需要指定一个自定义的key生成器，我们需要去实现org.springframework.cache.interceptor.KeyGenerator接口，并使用该参数来指定。需要注意的是：该参数与key是互斥的

- cacheManager
用于指定使用哪个缓存管理器，非必需。只有当有多个时才需要使用

- cacheResolver
用于指定使用那个缓存解析器，非必需。需通过org.springframework.cache.interceptor.CacheResolver接口来实现自己的缓存解析器，并用该参数指定。

### cacheNames & value
@Cacheable 提供两个参数来指定缓存名：value、cacheNames，二者选其一即可。这是 @Cacheable 最简单的用法示例：
```
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
```
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
与 `@Cacheable` 注解不同的是使用 `@CachePut` 注解标注的方法，在执行前不会去检查缓存中是否存在之前执行过的结果，而是每次都会执行该方法，并将执行结果以键值对的形式写入指定的缓存中。

@CachePut 注解一般用于更新缓存数据，相当于缓存使用的是写模式中的双写模式。

@CachePut：主要针对方法配置，能够根据方法的请求参数对其结果进行缓存，和 @Cacheable 不同的是，它每次都会触发真实方法的调用 。

简单来说就是用户更新缓存数据。但需要注意的是该注解的value 和 key 必须与要更新的缓存相同，也就是与@Cacheable 相同。

示例：
```
    @CachePut(value = "newJob", key = "#p0")  //按条件更新缓存
    public NewJob updata(NewJob job) {
        NewJob newJob = newJobDao.findAllById(job.getId());
        newJob.updata(job);
        return job;
    }
```

## @CacheEvict
标注了 @CacheEvict 注解的方法在被调用时，会从缓存中移除已存储的数据。@CacheEvict 注解一般用于删除缓存数据，相当于缓存使用的是写模式中的失效模式。

@CacheEvict：配置于函数上，通常用在删除方法上，用来从缓存中移除相应数据。除了同@Cacheable一样的参数之外，它还有下面两个参数：
- allEntries
非必需，默认为false。当为true时，会移除所有数据。如：@CachEvict(value=”testcache”,allEntries=true)

- beforeInvocation
非必需，默认为false，会在调用方法之后移除数据。当为true时，会在调用方法之前移除数据。 如：@CachEvict(value=”testcache”，beforeInvocation=true)

```
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

## @CacheConfig
@CacheConfig： 统一配置本类的缓存注解的属性，在类上面统一定义缓存的名字，方法上面就不用标注了，当标记在一个类上时则表示该类所有的方法都是支持缓存的 

```
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
```
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

## 最佳实践
通过Spring缓存注解可以快速优雅地在我们项目中实现缓存的操作，但是在双写模式或者失效模式下，可能会出现缓存数据一致性问题（读取到脏数据），Spring Cache 暂时没办法解决。

最后我们再总结下Spring Cache使用的一些最佳实践。

只缓存经常读取的数据。缓存可以显着提高性能，但只缓存经常访问的数据很重要。很少或从不访问的缓存数据会占用宝贵的内存资源，从而导致性能问题。

根据应用程序的特定需求选择合适的缓存提供程序和策略。SpringBoot 支持多种缓存提供程序，包括 Ehcache、Hazelcast 和 Redis。

使用缓存时请注意潜在的线程安全问题。对缓存的并发访问可能会导致数据不一致或不正确，因此选择线程安全的缓存提供程序并在必要时使用适当的同步机制非常重要。

避免过度缓存。缓存对于提高性能很有用，但过多的缓存实际上会消耗宝贵的内存资源，从而损害性能。在缓存频繁使用的数据和允许垃圾收集不常用的数据之间取得平衡很重要。

使用适当的缓存逐出策略。使用缓存时，重要的是定义适当的缓存逐出策略以确保在必要时从缓存中删除旧的或陈旧的数据。

使用适当的缓存键设计。缓存键对于每个数据项都应该是唯一的，并且应该考虑可能影响缓存数据的任何相关参数，例如用户 ID、时间或位置。

常规数据（读多写少、即时性与一致性要求不高的数据）完全可以使用 Spring Cache，至于写模式下缓存数据一致性问题的解决，只要缓存数据有设置过期时间就足够了。

特殊数据（读多写多、即时性与一致性要求非常高的数据），不能使用 Spring Cache，建议考虑特殊的设计（例如使用 Cancal 中间件等）。




