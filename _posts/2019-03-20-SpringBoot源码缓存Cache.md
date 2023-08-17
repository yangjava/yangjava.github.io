---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot源码缓存Cache

## Spring Cache结构
缓存抽象不提供实际的存储，而是依赖于org.springframework.cache.Cache和org.springframework.cache.CacheManager接口实现抽象

## Spring Cache流程
项目启动加载bean时，解析缓存注解，创建代理对象。
调用方法时，直接调用代理对象，在代理方法中获取到指定的CacheManager对象。
根据注解的cacheNames属性，在CacheManager中获取到Cache对象，然后调用org.springframework.cache.Cache中的方法对缓存数据进行操作。

CacheManager：主要作用是存储某一数据结构（例如Redis）中所有Cache对象。
Cache：作用是操纵Cache对象中的K-V数据。

## 源码分析
这个得从@EnableCaching标签开始，在使用缓存功能时，在springboot的Application启动类上需要添加注解@EnableCaching
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;


@SpringBootApplication
@EnableCaching
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```
cache自动配置原理
```java
@Configuration
@ConditionalOnClass(CacheManager.class)
@ConditionalOnBean(CacheAspectSupport.class)
@ConditionalOnMissingBean(value = CacheManager.class, name = "cacheResolver")
@EnableConfigurationProperties(CacheProperties.class)
@AutoConfigureAfter({ CouchbaseAutoConfiguration.class, HazelcastAutoConfiguration.class,
		HibernateJpaAutoConfiguration.class, RedisAutoConfiguration.class })
//这里向容器中注入了一个组件，我们点进这个组件CacheConfigurationImportSelector
@Import(CacheConfigurationImportSelector.class)
public class CacheAutoConfiguration {
  
//这是一个静态的内部类
  static class CacheConfigurationImportSelector implements ImportSelector {

    //选择让哪些缓存配置类生效，通过debug的形式，看到imports为如下值
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
      CacheType[] types = CacheType.values();
      String[] imports = new String[types.length];
      for (int i = 0; i < types.length; i++) {
        imports[i] = CacheConfigurations.getConfigurationClass(types[i]);
      }
      //这里打个断点
      return imports;
      /**imports =
     * 0 = "org.springframework.boot.autoconfigure.cache.GenericCacheConfiguration"
     * 1 = "org.springframework.boot.autoconfigure.cache.JCacheCacheConfiguration"
     * 2 = "org.springframework.boot.autoconfigure.cache.EhCacheCacheConfiguration"
     * 3 = "org.springframework.boot.autoconfigure.cache.HazelcastCacheConfiguration"
     * 4 = "org.springframework.boot.autoconfigure.cache.InfinispanCacheConfiguration"
     * 5 = "org.springframework.boot.autoconfigure.cache.CouchbaseCacheConfiguration"
     * 6 = "org.springframework.boot.autoconfigure.cache.RedisCacheConfiguration"
     * 7 = "org.springframework.boot.autoconfigure.cache.CaffeineCacheConfiguration"
     * 8 = "org.springframework.boot.autoconfigure.cache.SimpleCacheConfiguration"
     * 9 = "org.springframework.boot.autoconfigure.cache.NoOpCacheConfiguration"
   默认是只有SimpleCacheConfiguration 这个配置类生效（debug=true）
   控制台打印如下：
   SimpleCacheConfiguration matched:
- Cache org.springframework.boot.autoconfigure.cache.SimpleCacheConfiguration automatic cache type (CacheCondition)
- @ConditionalOnMissingBean (types: org.springframework.cache.CacheManager; SearchStrategy: all) did not find any beans (OnBeanCondition)
      */
    }
  }
  
//点进SimpleCacheConfiguration，这是一个配置类
@Configuration
@ConditionalOnMissingBean(CacheManager.class)
@Conditional(CacheCondition.class)
class SimpleCacheConfiguration {
  //给容器中注入一个CacheManager缓存管理器
  //每个cacheManager都是一个ConcurrentMapCacheManager实例
  @Bean
	public ConcurrentMapCacheManager cacheManager() {
		ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager();
    //从配置文件中获取所有的cache的name
		List<String> cacheNames = this.cacheProperties.getCacheNames();
     //如果cache为空，cacheManager重新设置缓存名,并用一个Map将(cacheName，Cache)管理起来
		if (!cacheNames.isEmpty()) {
			cacheManager.setCacheNames(cacheNames);
		}
		return this.customizerInvoker.customize(cacheManager);
	}

  //然后我们继续点进ConcurrentMapCacheManager这个类，这个类是用来管理Caches的，点进getCache（）方法
  @Override
	@Nullable
	public Cache getCache(String name) {
    // 通过传入的name在map中查找对应的Cache对象
		Cache cache = this.cacheMap.get(name);
    // 如果查找结果为空，或者这个name在ConcurrentMap中不存在
		if (cache == null && this.dynamic) {
			synchronized (this.cacheMap) {
				cache = this.cacheMap.get(name);
				if (cache == null) {
          // 那么就用线程安全的模式，按照传入的name创建一个ConcurrentMapCache实例
					cache = createConcurrentMapCache(name);
          // 然后通过Manager的Map属性将新建的cache与name管理起来
					this.cacheMap.put(name, cache);
				}
			}
		}
    // 最终获取到cache并返回出去
		return cache;
    
    
//ConcurrentMapCache是默认返回的Cache类型，我们点进去看看
//在Cache中，所有的缓存数据都是以(k,v)类型进行存储的，用一个Map store 属性进行维护
public class ConcurrentMapCache extends AbstractValueAdaptingCache {

  //获取缓存内容的方法
  protected Object lookup(Object key) {
		return this.store.get(key);
	}
  // 添加缓存内容的方法
 @Override
	public void put(Object key, @Nullable Object value) {
		this.store.put(key, toStoreValue(value));
	}
  // 删除缓存内容的方法
  @Override
  public void evict(Object key) {
    this.store.remove(key);
  }
	// 最后可以在各个方法内打个断点试一试了，用@Cacheable注解，然后在浏览器中访问2次 分别查看代码走向
```

，这个标签引入了
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({CachingConfigurationSelector.class})
public @interface EnableCaching {
    boolean proxyTargetClass() default false;
    AdviceMode mode() default AdviceMode.PROXY;
    int order() default 2147483647;
}
```
引入了CachingConfigurationSelector类，这个类便开启了缓存功能的配置。这个类添加了AutoProxyRegistrar.java,ProxyCachingConfiguration.java两个类。

- AutoProxyRegistrar : 实现了ImportBeanDefinitionRegistrar接口。这里看不懂，还需要继续学习。
- ProxyCachingConfiguration : 是一个配置类，生成了BeanFactoryCacheOperationSourceAdvisor，CacheOperationSource，和CacheInterceptor这三个bean。

CacheOperationSource封装了cache方法签名注解的解析工作，形成CacheOperation的集合。CacheInterceptor使用该集合过滤执行缓存处理。解析缓存注解的类是SpringCacheAnnotationParser，其主要方法如下
```
/**
由CacheOperationSourcePointcut作为注解切面，会解析
SpringCacheAnnotationParser.java
扫描方法签名，解析被缓存注解修饰的方法，将生成一个CacheOperation的子类并将其保存到一个数组中去
**/
protected Collection&lt;CacheOperation&gt; parseCacheAnnotations(SpringCacheAnnotationParser.DefaultCacheConfig cachingConfig, AnnotatedElement ae) {
        Collection&lt;CacheOperation&gt; ops = null;
        //找@cacheable注解方法
        Collection&lt;Cacheable&gt; cacheables = AnnotatedElementUtils.getAllMergedAnnotations(ae, Cacheable.class);
        if (!cacheables.isEmpty()) {
            ops = this.lazyInit(ops);
            Iterator var5 = cacheables.iterator();

            while(var5.hasNext()) {
                Cacheable cacheable = (Cacheable)var5.next();
                ops.add(this.parseCacheableAnnotation(ae, cachingConfig, cacheable));
            }
        }
        //找@cacheEvict注解的方法
        Collection&lt;CacheEvict&gt; evicts = AnnotatedElementUtils.getAllMergedAnnotations(ae, CacheEvict.class);
        if (!evicts.isEmpty()) {
            ops = this.lazyInit(ops);
            Iterator var12 = evicts.iterator();

            while(var12.hasNext()) {
                CacheEvict evict = (CacheEvict)var12.next();
                ops.add(this.parseEvictAnnotation(ae, cachingConfig, evict));
            }
        }
        //找@cachePut注解的方法
        Collection&lt;CachePut&gt; puts = AnnotatedElementUtils.getAllMergedAnnotations(ae, CachePut.class);
        if (!puts.isEmpty()) {
            ops = this.lazyInit(ops);
            Iterator var14 = puts.iterator();

            while(var14.hasNext()) {
                CachePut put = (CachePut)var14.next();
                ops.add(this.parsePutAnnotation(ae, cachingConfig, put));
            }
        }
        Collection&lt;Caching&gt; cachings = AnnotatedElementUtils.getAllMergedAnnotations(ae, Caching.class);
        if (!cachings.isEmpty()) {
            ops = this.lazyInit(ops);
            Iterator var16 = cachings.iterator();

            while(var16.hasNext()) {
                Caching caching = (Caching)var16.next();
                Collection&lt;CacheOperation&gt; cachingOps = this.parseCachingAnnotation(ae, cachingConfig, caching);
                if (cachingOps != null) {
                    ops.addAll(cachingOps);
                }
            }
        }
        return ops;
}
```
解析Cachable,Caching,CachePut,CachEevict 这四个注解对应的方法都保存到了Collection<CacheOperation> 集合中。

## 执行方法时做了什么
执行的时候，主要使用了CacheInterceptor类。
```
public class CacheInterceptor extends CacheAspectSupport implements MethodInterceptor, Serializable {
    public CacheInterceptor() {
    }

    public Object invoke(final MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();
        CacheOperationInvoker aopAllianceInvoker = new CacheOperationInvoker() {
            public Object invoke() {
                try {
                    return invocation.proceed();
                } catch (Throwable var2) {
                    throw new ThrowableWrapper(var2);
                }
            }
        };

        try {
            return this.execute(aopAllianceInvoker, invocation.getThis(), method, invocation.getArguments());
        } catch (ThrowableWrapper var5) {
            throw var5.getOriginal();
        }
    }
}
```

这个拦截器继承了CacheAspectSupport类和MethodInterceptor接口。其中CacheAspectSupport封装了主要的逻辑。比如下面这段。
```
/**
CacheAspectSupport.java
执行@CachaEvict @CachePut @Cacheable的主要逻辑代码
**/

private Object execute(final CacheOperationInvoker invoker, Method method, CacheAspectSupport.CacheOperationContexts contexts) {
        if (contexts.isSynchronized()) {
            CacheAspectSupport.CacheOperationContext context = (CacheAspectSupport.CacheOperationContext)contexts.get(CacheableOperation.class).iterator().next();
            if (this.isConditionPassing(context, CacheOperationExpressionEvaluator.NO_RESULT)) {
                Object key = this.generateKey(context, CacheOperationExpressionEvaluator.NO_RESULT);
                Cache cache = (Cache)context.getCaches().iterator().next();

                try {
                    return this.wrapCacheValue(method, cache.get(key, new Callable&lt;Object&gt;() {
                        public Object call() throws Exception {
                            return CacheAspectSupport.this.unwrapReturnValue(CacheAspectSupport.this.invokeOperation(invoker));
                        }
                    }));
                } catch (ValueRetrievalException var10) {
                    throw (ThrowableWrapper)var10.getCause();
                }
            } else {
                return this.invokeOperation(invoker);
            }
        } else {
            /**
            执行@CacheEvict的逻辑，这里是当beforeInvocation为true时清缓存
            **/
            this.processCacheEvicts(contexts.get(CacheEvictOperation.class), true, CacheOperationExpressionEvaluator.NO_RESULT);
            //获取命中的缓存对象
            ValueWrapper cacheHit = this.findCachedItem(contexts.get(CacheableOperation.class));
            List&lt;CacheAspectSupport.CachePutRequest&gt; cachePutRequests = new LinkedList();
            if (cacheHit == null) {
                //如果没有命中，则生成一个put的请求
                this.collectPutRequests(contexts.get(CacheableOperation.class), CacheOperationExpressionEvaluator.NO_RESULT, cachePutRequests);
            }


            Object cacheValue;
            Object returnValue;
            /**
                如果没有获得缓存对象，则调用业务方法获得返回对象，hasCachePut会检查exclude的情况
            **/
            if (cacheHit != null &amp;&amp; cachePutRequests.isEmpty() &amp;&amp; !this.hasCachePut(contexts)) {
                cacheValue = cacheHit.get();
                returnValue = this.wrapCacheValue(method, cacheValue);
            } else {
                
                returnValue = this.invokeOperation(invoker);
                cacheValue = this.unwrapReturnValue(returnValue);
            }

            this.collectPutRequests(contexts.get(CachePutOperation.class), cacheValue, cachePutRequests);
            Iterator var8 = cachePutRequests.iterator();

            while(var8.hasNext()) {
                CacheAspectSupport.CachePutRequest cachePutRequest = (CacheAspectSupport.CachePutRequest)var8.next();
                /**
                执行cachePut请求，将返回对象放到缓存中
                **/
                cachePutRequest.apply(cacheValue);
            }
            /**
            执行@CacheEvict的逻辑，这里是当beforeInvocation为false时清缓存
            **/
            this.processCacheEvicts(contexts.get(CacheEvictOperation.class), false, cacheValue);
            return returnValue;
        }
    }
```
上面的代码片段比较核心，均是cache的内容，对于aop的源码，这里不详细展开，应该单起一篇文章进行研究。主要的类和接口都在spring的context中，org.springframework.cache包中。

## 和第三方缓存组件的适配
通过以上的分析，知道了spring cache功能的来龙去脉，下面需要分析的是，为什么只需要maven声明一下依赖，spring boot 就可以自动就适配了.

在上面的执行方法中，我们看到了cachePutRequest.apply(cacheValue) ,这里会操作缓存，CachePutRequest是CacheAspectSupport的内部类。

```
private class CachePutRequest {
        private final CacheAspectSupport.CacheOperationContext context;
        private final Object key;
        public CachePutRequest(CacheAspectSupport.CacheOperationContext context, Object key) {
            this.context = context;
            this.key = key;
        }
        public void apply(Object result) {
            if (this.context.canPutToCache(result)) {
                //从context中获取cache实例，然后执行放入缓存的操作
                Iterator var2 = this.context.getCaches().iterator();
                while(var2.hasNext()) {
                    Cache cache = (Cache)var2.next();
                    CacheAspectSupport.this.doPut(cache, this.key, result);
                }
            }
        }
    }
```
Cache是一个标准接口，其中EhCacheCache就是EhCache的实现类。这里就是SpringBoot和Ehcache之间关联的部分，那么context中的cache列表是什么时候生成的呢。答案是CacheAspectSupport的getCaches方法

```
protected Collection&lt;? extends Cache&gt; getCaches(CacheOperationInvocationContext&lt;CacheOperation&gt; context, CacheResolver cacheResolver) {
        Collection&lt;? extends Cache&gt; caches = cacheResolver.resolveCaches(context);
        if (caches.isEmpty()) {
            throw new IllegalStateException("No cache could be resolved for '" + context.getOperation() + "' using resolver '" + cacheResolver + "'. At least one cache should be provided per cache operation.");
        } else {
            return caches;
        }
    }
```
而获取cache是在每一次进行进行缓存操作的时候执行。

在spring-boot-autoconfigure包里，有所有自动装配相关的类。这里有个EhcacheCacheConfiguration类 ，如下
```
@Configuration
@ConditionalOnClass({Cache.class, EhCacheCacheManager.class})
@ConditionalOnMissingBean({CacheManager.class})
@Conditional({CacheCondition.class, EhCacheCacheConfiguration.ConfigAvailableCondition.class})
class EhCacheCacheConfiguration {
 ......
 static class ConfigAvailableCondition extends ResourceCondition {
        ConfigAvailableCondition() {
            super("EhCache", "spring.cache.ehcache", "config", new String[]{"classpath:/ehcache.xml"});
        }
    }    
}
```
这里会直接判断类路径下是否有ehcache.xml文件
















