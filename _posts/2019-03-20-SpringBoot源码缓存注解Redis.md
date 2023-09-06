---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot源码注解缓存Redis


## 找到CacheAutoConfiguration类
CacheAutoConfiguration
```
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(CacheManager.class)
@ConditionalOnBean(CacheAspectSupport.class)
@ConditionalOnMissingBean(value = CacheManager.class, name = "cacheResolver")
@EnableConfigurationProperties(CacheProperties.class)
@AutoConfigureAfter({ CouchbaseAutoConfiguration.class, HazelcastAutoConfiguration.class,
		HibernateJpaAutoConfiguration.class, RedisAutoConfiguration.class })
@Import({ CacheConfigurationImportSelector.class, CacheManagerEntityManagerFactoryDependsOnPostProcessor.class })
public class CacheAutoConfiguration {

}
```
发现 内部有个配置类加载选择器，决定了哪个配置类会被加载！
```
	static class CacheConfigurationImportSelector implements ImportSelector {

		@Override
		public String[] selectImports(AnnotationMetadata importingClassMetadata) {
			CacheType[] types = CacheType.values();
			String[] imports = new String[types.length];
			for (int i = 0; i < types.length; i++) {
				imports[i] = CacheConfigurations.getConfigurationClass(types[i]);
			}
			return imports;
		}

	}
```

RedisCacheConfiguration为redis的缓存配置类。
```
	static {
		Map<CacheType, Class<?>> mappings = new EnumMap<>(CacheType.class);
		mappings.put(CacheType.GENERIC, GenericCacheConfiguration.class);
		mappings.put(CacheType.EHCACHE, EhCacheCacheConfiguration.class);
		mappings.put(CacheType.HAZELCAST, HazelcastCacheConfiguration.class);
		mappings.put(CacheType.INFINISPAN, InfinispanCacheConfiguration.class);
		mappings.put(CacheType.JCACHE, JCacheCacheConfiguration.class);
		mappings.put(CacheType.COUCHBASE, CouchbaseCacheConfiguration.class);
		mappings.put(CacheType.REDIS, RedisCacheConfiguration.class);
		mappings.put(CacheType.CAFFEINE, CaffeineCacheConfiguration.class);
		mappings.put(CacheType.SIMPLE, SimpleCacheConfiguration.class);
		mappings.put(CacheType.NONE, NoOpCacheConfiguration.class);
		MAPPINGS = Collections.unmodifiableMap(mappings);
	}
```

找到RedisCacheConfiguration类
```
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisConnectionFactory.class)
@AutoConfigureAfter(RedisAutoConfiguration.class)
@ConditionalOnBean(RedisConnectionFactory.class)
@ConditionalOnMissingBean(CacheManager.class)
@Conditional(CacheCondition.class)
class RedisCacheConfiguration {

	@Bean
	RedisCacheManager cacheManager(CacheProperties cacheProperties, CacheManagerCustomizers cacheManagerCustomizers,
			ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration,
			ObjectProvider<RedisCacheManagerBuilderCustomizer> redisCacheManagerBuilderCustomizers,
			RedisConnectionFactory redisConnectionFactory, ResourceLoader resourceLoader) {
		RedisCacheManagerBuilder builder = RedisCacheManager.builder(redisConnectionFactory).cacheDefaults(
				determineConfiguration(cacheProperties, redisCacheConfiguration, resourceLoader.getClassLoader()));
		List<String> cacheNames = cacheProperties.getCacheNames();
		if (!cacheNames.isEmpty()) {
			builder.initialCacheNames(new LinkedHashSet<>(cacheNames));
		}
		redisCacheManagerBuilderCustomizers.orderedStream().forEach((customizer) -> customizer.customize(builder));
		return cacheManagerCustomizers.customize(builder.build());
	}
	
```
RedisCacheConfiguration向spring中放入了RedisCacheManager的bean，用于管理Redis的Cache。

其中有个方法：determineConfiguration（）决定哪个redis缓存配置类生效。

转到determineConfiguration（）方法
```
	private org.springframework.data.redis.cache.RedisCacheConfiguration determineConfiguration(
			CacheProperties cacheProperties,
			ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration,
			ClassLoader classLoader) {
		return redisCacheConfiguration.getIfAvailable(() -> createConfiguration(cacheProperties, classLoader));
	}
```
cache配置真相
- ObjectProvider表示是从spring的容器中取值
- determineConfiguration（）方法表示在spring容器中如果自己配置了RedisCacheConfiguration类，则取得并使用该类； 如果没有配置，则使用默认的RedisCacheConfiguration类！

## RedisCacheConfiguration
找到RedisCacheConfiguration类。这个类中制定了序列化的方式！因此，我们只需要将其改造成自己的MyRedisCacheConfiguration 配置类，将其放入容器中即可！

自定义RedisCacheConfiguration配置类指定序列化方式！
```
import com.alibaba.fastjson.support.spring.GenericFastJsonRedisSerializer;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.boot.autoconfigure.cache.CacheProperties;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;
 
/**
 */
@EnableConfigurationProperties(value = CacheProperties.class)//允许将配置属性配置类放入spring容器中
@Configuration
public class MyRedisCacheConfiguration {
    /**
     * 这里我们借鉴使用ObjectProvider，其实可以直接使用CacheProperties cacheProperties；
     * 因为@EnableConfigurationProperties(value = CacheProperties.class)，容器中放入了CacheProperties的Bean实例
     * @param cacheProperties
     * @return
     */
    @Bean
    public RedisCacheConfiguration redisCacheConfiguration(ObjectProvider<CacheProperties> cacheProperties){
        //在默认配置上修改，也可以new一个。指定序列化方式！
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig().
                serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer())).
                serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericFastJsonRedisSerializer()));
 
        //从容器中取得redisProperties并进行判断，不然在application.yml配置的文件无效。
        // 因为只有默认的RedisCacheConfiguration才会走下面的内容，自定义的需要cv大法后改造一下
        CacheProperties ifAvailable = cacheProperties.getIfAvailable();
        if (null != ifAvailable){
            CacheProperties.Redis redisProperties = ifAvailable.getRedis();
            if (redisProperties.getTimeToLive() != null) {
                config = config.entryTtl(redisProperties.getTimeToLive());
            }
            if (redisProperties.getKeyPrefix() != null) {
                config = config.prefixKeysWith(redisProperties.getKeyPrefix());
            }
            if (!redisProperties.isCacheNullValues()) {
                config = config.disableCachingNullValues();
            }
            if (!redisProperties.isUseKeyPrefix()) {
                config = config.disableKeyPrefix();
            }
        }
        return config;
    }
}
```

## 源码解析

## @EnableCaching
该类会注入@Import(CachingConfigurationSelector.class)，而CachingConfigurationSelector重写了selectImports方法。
```
	@Override
	public String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return getProxyImports();
			case ASPECTJ:
				return getAspectJImports();
			default:
				return null;
		}
	}

	private String[] getProxyImports() {
		List<String> result = new ArrayList<>(3);
		result.add(AutoProxyRegistrar.class.getName());
		result.add(ProxyCachingConfiguration.class.getName());
		if (jsr107Present && jcacheImplPresent) {
			result.add(PROXY_JCACHE_CONFIGURATION_CLASS);
		}
		return StringUtils.toStringArray(result);
	}
```

## ProxyCachingConfiguration
缓存代理配置类。拦截类CacheInterceptor，切面类CacheOperationSourcePointcut。
```
@Configuration(proxyBeanMethods = false)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyCachingConfiguration extends AbstractCachingConfiguration {

	@Bean(name = CacheManagementConfigUtils.CACHE_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryCacheOperationSourceAdvisor cacheAdvisor(
			CacheOperationSource cacheOperationSource, CacheInterceptor cacheInterceptor) {

		BeanFactoryCacheOperationSourceAdvisor advisor = new BeanFactoryCacheOperationSourceAdvisor();
		advisor.setCacheOperationSource(cacheOperationSource);
		advisor.setAdvice(cacheInterceptor);
		if (this.enableCaching != null) {
			advisor.setOrder(this.enableCaching.<Integer>getNumber("order"));
		}
		return advisor;
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheOperationSource cacheOperationSource() {
		return new AnnotationCacheOperationSource();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheInterceptor cacheInterceptor(CacheOperationSource cacheOperationSource) {
		CacheInterceptor interceptor = new CacheInterceptor();
		interceptor.configure(this.errorHandler, this.keyGenerator, this.cacheResolver, this.cacheManager);
		interceptor.setCacheOperationSource(cacheOperationSource);
		return interceptor;
	}

}
```

SpringCacheAnnotationParser#isCandidateClass，判断是否会被拦截。
```
public class SpringCacheAnnotationParser implements CacheAnnotationParser, Serializable {

	private static final Set<Class<? extends Annotation>> CACHE_OPERATION_ANNOTATIONS = new LinkedHashSet<>(8);

	static {
		CACHE_OPERATION_ANNOTATIONS.add(Cacheable.class);
		CACHE_OPERATION_ANNOTATIONS.add(CacheEvict.class);
		CACHE_OPERATION_ANNOTATIONS.add(CachePut.class);
		CACHE_OPERATION_ANNOTATIONS.add(Caching.class);
	}


	@Override
	public boolean isCandidateClass(Class<?> targetClass) {
		return AnnotationUtils.isCandidateClass(targetClass, CACHE_OPERATION_ANNOTATIONS);
	}
}
```

## CacheAutoConfiguration
自动装配，在spring-boot-autoconfigure中的spring.factoires有引入CacheAutoConfiguration。该类会注入Spring的条件是CacheManger类存在，且Spring容器中存在CacheAspectSupport。因为@EnableCaching引入了CacheInterceptor，CacheInterceptor继承了CacheAspectSupport，所以CacheAutoConfiguration会被注入到Spring容器中。随着CacheAutoConfiguration的成功注入，会注入CacheConfigurationImportSelector和CacheManagerEntityManagerFactoryDependsOnPostProcessor。
```
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnClass({CacheManager.class})
@ConditionalOnBean({CacheAspectSupport.class})
@ConditionalOnMissingBean(
    value = {CacheManager.class},
    name = {"cacheResolver"}
)
@EnableConfigurationProperties({CacheProperties.class})
@AutoConfigureAfter({CouchbaseDataAutoConfiguration.class, HazelcastAutoConfiguration.class, HibernateJpaAutoConfiguration.class, RedisAutoConfiguration.class})
@Import({CacheAutoConfiguration.CacheConfigurationImportSelector.class, CacheAutoConfiguration.CacheManagerEntityManagerFactoryDependsOnPostProcessor.class})
public class CacheAutoConfiguration {
    public CacheAutoConfiguration() {
    }
}
```

## CacheAutoConfiguration.CacheConfigurationImportSelector
CacheAutoConfiguration.CacheConfigurationImportSelector#selectImports，该类重写了selectImports方法，引入了RedisCacheConfiguration等缓存类。
```
        public String[] selectImports(AnnotationMetadata importingClassMetadata) {
            CacheType[] types = CacheType.values();
            String[] imports = new String[types.length];

            for(int i = 0; i < types.length; ++i) {
                imports[i] = CacheConfigurations.getConfigurationClass(types[i]);
            }

            return imports;
        }
```

CacheType
```
public enum CacheType {
    GENERIC,
    JCACHE,
    EHCACHE,
    HAZELCAST,
    INFINISPAN,
    COUCHBASE,
    REDIS,
    CAFFEINE,
    SIMPLE,
    NONE;
}
```

## RedisCacheConfiguration
RedisCacheConfiguration会往spring中注入cacheManager的类。
```
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnClass({RedisConnectionFactory.class})
@AutoConfigureAfter({RedisAutoConfiguration.class})
@ConditionalOnBean({RedisConnectionFactory.class})
@ConditionalOnMissingBean({CacheManager.class})
@Conditional({CacheCondition.class})
class RedisCacheConfiguration {
    RedisCacheConfiguration() {
    }

    @Bean
    RedisCacheManager cacheManager(CacheProperties cacheProperties, CacheManagerCustomizers cacheManagerCustomizers, ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration, ObjectProvider<RedisCacheManagerBuilderCustomizer> redisCacheManagerBuilderCustomizers, RedisConnectionFactory redisConnectionFactory, ResourceLoader resourceLoader) {
        RedisCacheManagerBuilder builder = RedisCacheManager.builder(redisConnectionFactory).cacheDefaults(this.determineConfiguration(cacheProperties, redisCacheConfiguration, resourceLoader.getClassLoader()));
        List<String> cacheNames = cacheProperties.getCacheNames();
        if (!cacheNames.isEmpty()) {
            builder.initialCacheNames(new LinkedHashSet(cacheNames));
        }

        if (cacheProperties.getRedis().isEnableStatistics()) {
            builder.enableStatistics();
        }

        redisCacheManagerBuilderCustomizers.orderedStream().forEach((customizer) -> {
            customizer.customize(builder);
        });
        return (RedisCacheManager)cacheManagerCustomizers.customize(builder.build());
    }
}
```

## RedisCacheManager
该类继承了AbstractCacheManager，AbstractCacheManager重写了接口InitializingBean的方法，loadCaches是模板方法，具体的业务由子类实现。
```
	@Override
	public void afterPropertiesSet() {
		initializeCaches();
	}

	public void initializeCaches() {
		Collection<? extends Cache> caches = loadCaches();

		synchronized (this.cacheMap) {
			this.cacheNames = Collections.emptySet();
			this.cacheMap.clear();
			Set<String> cacheNames = new LinkedHashSet<>(caches.size());
			for (Cache cache : caches) {
				String name = cache.getName();
				this.cacheMap.put(name, decorateCache(cache));
				cacheNames.add(name);
			}
			this.cacheNames = Collections.unmodifiableSet(cacheNames);
		}
	}
```

RedisCacheManager#loadCaches，会创建多个RedisCache的缓存。
```
	protected Collection<RedisCache> loadCaches() {

		List<RedisCache> caches = new LinkedList<>();

		for (Map.Entry<String, RedisCacheConfiguration> entry : initialCacheConfiguration.entrySet()) {
			caches.add(createRedisCache(entry.getKey(), entry.getValue()));
		}

		return caches;
	}
```
RedisCacheConfiguration#defaultCacheConfig()，加载默认配置，默认的缓存配置是0s，意味着没有缓存时间。

```
	public static RedisCacheConfiguration defaultCacheConfig() {
		return defaultCacheConfig(null);
	}

	public static RedisCacheConfiguration defaultCacheConfig(@Nullable ClassLoader classLoader) {

		DefaultFormattingConversionService conversionService = new DefaultFormattingConversionService();

		registerDefaultConverters(conversionService);

		return new RedisCacheConfiguration(Duration.ZERO, true, true, CacheKeyPrefix.simple(),
				SerializationPair.fromSerializer(RedisSerializer.string()),
				SerializationPair.fromSerializer(RedisSerializer.java(classLoader)), conversionService);
	}
```

## CacheInterceptor
CacheInterceptor重写invoke方法。
```
public class CacheInterceptor extends CacheAspectSupport implements MethodInterceptor, Serializable {

	@Override
	@Nullable
	public Object invoke(final MethodInvocation invocation) throws Throwable {
		Method method = invocation.getMethod();

		CacheOperationInvoker aopAllianceInvoker = () -> {
			try {
				return invocation.proceed();
			}
			catch (Throwable ex) {
				throw new CacheOperationInvoker.ThrowableWrapper(ex);
			}
		};

		Object target = invocation.getThis();
		Assert.state(target != null, "Target must not be null");
		try {
			return execute(aopAllianceInvoker, target, method, invocation.getArguments());
		}
		catch (CacheOperationInvoker.ThrowableWrapper th) {
			throw th.getOriginal();
		}
	}

}
```

CacheAspectSupport#execute()，获取CacheOperation，执行缓存的相关逻辑。
```
	@Nullable
	protected Object execute(CacheOperationInvoker invoker, Object target, Method method, Object[] args) {
		// Check whether aspect is enabled (to cope with cases where the AJ is pulled in automatically)
		if (this.initialized) {
			Class<?> targetClass = getTargetClass(target);
			CacheOperationSource cacheOperationSource = getCacheOperationSource();
			if (cacheOperationSource != null) {
				Collection<CacheOperation> operations = cacheOperationSource.getCacheOperations(method, targetClass);
				if (!CollectionUtils.isEmpty(operations)) {
					return execute(invoker, method,
							new CacheOperationContexts(operations, method, args, target, targetClass));
				}
			}
		}

		return invoker.invoke();
	}
```

AbstractFallbackCacheOperationSource#getCacheOperations获取注解信息，为了避免重复解析，进行了缓存优化。已经解析过的方法直接从缓存中获取。
```
	public Collection<CacheOperation> getCacheOperations(Method method, @Nullable Class<?> targetClass) {
		if (method.getDeclaringClass() == Object.class) {
			return null;
		}

		Object cacheKey = getCacheKey(method, targetClass);
		Collection<CacheOperation> cached = this.attributeCache.get(cacheKey);

		if (cached != null) {
			return (cached != NULL_CACHING_ATTRIBUTE ? cached : null);
		}
		else {
			Collection<CacheOperation> cacheOps = computeCacheOperations(method, targetClass);
			if (cacheOps != null) {
				if (logger.isTraceEnabled()) {
					logger.trace("Adding cacheable method '" + method.getName() + "' with attribute: " + cacheOps);
				}
				this.attributeCache.put(cacheKey, cacheOps);
			}
			else {
				this.attributeCache.put(cacheKey, NULL_CACHING_ATTRIBUTE);
			}
			return cacheOps;
		}
	}
```

CacheAspectSupport#execute()，缓存中存在数据，则不会进行方法调用，缓存中不存在数据，进行缓存的更新。
```
	@Nullable
	private Object execute(final CacheOperationInvoker invoker, Method method, CacheOperationContexts contexts) {
		// Special handling of synchronized invocation
		if (contexts.isSynchronized()) {
			CacheOperationContext context = contexts.get(CacheableOperation.class).iterator().next();
			if (isConditionPassing(context, CacheOperationExpressionEvaluator.NO_RESULT)) {
				Object key = generateKey(context, CacheOperationExpressionEvaluator.NO_RESULT);
				Cache cache = context.getCaches().iterator().next();
				try {
					return wrapCacheValue(method, handleSynchronizedGet(invoker, key, cache));
				}
				catch (Cache.ValueRetrievalException ex) {
					// Directly propagate ThrowableWrapper from the invoker,
					// or potentially also an IllegalArgumentException etc.
					ReflectionUtils.rethrowRuntimeException(ex.getCause());
				}
			}
			else {
				// No caching required, only call the underlying method
				return invokeOperation(invoker);
			}
		}


		// Process any early evictions
		processCacheEvicts(contexts.get(CacheEvictOperation.class), true,
				CacheOperationExpressionEvaluator.NO_RESULT);

		// Check if we have a cached item matching the conditions
		Cache.ValueWrapper cacheHit = findCachedItem(contexts.get(CacheableOperation.class));

		// Collect puts from any @Cacheable miss, if no cached item is found
		List<CachePutRequest> cachePutRequests = new ArrayList<>();
		if (cacheHit == null) {
			collectPutRequests(contexts.get(CacheableOperation.class),
					CacheOperationExpressionEvaluator.NO_RESULT, cachePutRequests);
		}

		Object cacheValue;
		Object returnValue;

		if (cacheHit != null && !hasCachePut(contexts)) {
			// If there are no put requests, just use the cache hit
			cacheValue = cacheHit.get();
			returnValue = wrapCacheValue(method, cacheValue);
		}
		else {
			// Invoke the method if we don't have a cache hit
			returnValue = invokeOperation(invoker);
			cacheValue = unwrapReturnValue(returnValue);
		}

		// Collect any explicit @CachePuts
		collectPutRequests(contexts.get(CachePutOperation.class), cacheValue, cachePutRequests);

		// Process any collected put requests, either from @CachePut or a @Cacheable miss
		for (CachePutRequest cachePutRequest : cachePutRequests) {
			cachePutRequest.apply(cacheValue);
		}

		// Process any late evictions
		processCacheEvicts(contexts.get(CacheEvictOperation.class), false, cacheValue);

		return returnValue;
	}
```
CacheAspectSupport#findCachedItem，判断缓存中是否有数据。
```
	private Cache.ValueWrapper findCachedItem(Collection<CacheOperationContext> contexts) {
		Object result = CacheOperationExpressionEvaluator.NO_RESULT;
		for (CacheOperationContext context : contexts) {
			if (isConditionPassing(context, result)) {
				Object key = generateKey(context, result);
				Cache.ValueWrapper cached = findInCaches(context, key);
				if (cached != null) {
					return cached;
				}
				else {
					if (logger.isTraceEnabled()) {
						logger.trace("No cache entry for key '" + key + "' in cache(s) " + context.getCacheNames());
					}
				}
			}
		}
		return null;
	}
```
CacheAspectSupport.CacheOperationContext#generateKey，获取key，这里用到了Spring的表达式。
```
		protected Object generateKey(@Nullable Object result) {
			if (StringUtils.hasText(this.metadata.operation.getKey())) {
				EvaluationContext evaluationContext = createEvaluationContext(result);
				return evaluator.key(this.metadata.operation.getKey(), this.metadata.methodKey, evaluationContext);
			}
			return this.metadata.keyGenerator.generate(this.target, this.metadata.method, this.args);
		}
```

## RedisCache
获取数据
根据key获取缓存数据，AbstractValueAdaptingCache#get(java.lang.Object)
```
	public ValueWrapper get(Object key) {
		return toValueWrapper(lookup(key));
	}
```
RedisCache#lookup，这里默认使用字符串序列化器对key进行序列化，用JDK序列器对value进行序列化。
```
	@Override
	protected Object lookup(Object key) {

		byte[] value = cacheWriter.get(name, createAndConvertCacheKey(key));

		if (value == null) {
			return null;
		}

		return deserializeCacheValue(value);
	}
```
RedisCache#prefixCacheKey，会将缓存的name和key拼接在一起。格式name::key。
```
	private String prefixCacheKey(String key) {

		// allow contextual cache names by computing the key prefix on every call.
		return cacheConfig.getKeyPrefixFor(name) + key;
	}
```
RedisCache#serializeCacheKey，根据生成的key进行序列化。这里有用到ByteBuff。
```
protected byte[] serializeCacheKey(String cacheKey) {
   return ByteUtils.getBytes(cacheConfig.getKeySerializationPair().write(cacheKey));
}
```
DefaultRedisCacheWriter#get，根据生成的key的字节数组来获取缓存中的数据。
```
	@Override
	public byte[] get(String name, byte[] key) {

		Assert.notNull(name, "Name must not be null!");
		Assert.notNull(key, "Key must not be null!");

		byte[] result = execute(name, connection -> connection.get(key));

		statistics.incGets(name);

		if (result != null) {
			statistics.incHits(name);
		} else {
			statistics.incMisses(name);
		}

		return result;
	}
```

RedisCache#deserializeCacheValue，反序列化转成对象。
```
	@Nullable
	protected Object deserializeCacheValue(byte[] value) {

		if (isAllowNullValues() && ObjectUtils.nullSafeEquals(value, BINARY_NULL_VALUE)) {
			return NullValue.INSTANCE;
		}

		return cacheConfig.getValueSerializationPair().read(ByteBuffer.wrap(value));
	}
```

存储数据
RedisCache#put
```
	@Override
	public void put(Object key, @Nullable Object value) {

		Object cacheValue = preProcessCacheValue(value);

		if (!isAllowNullValues() && cacheValue == null) {

			throw new IllegalArgumentException(String.format(
					"Cache '%s' does not allow 'null' values. Avoid storing null via '@Cacheable(unless=\"#result == null\")' or configure RedisCache to allow 'null' via RedisCacheConfiguration.",
					name));
		}

		cacheWriter.put(name, createAndConvertCacheKey(key), serializeCacheValue(cacheValue), cacheConfig.getTtl());
	}
```
DefaultRedisCacheWriter#put，判断是否需要过期时间。RedisCache的缓存配置和CacheManager的缓存配置是一致的，默认的缓存配置ttl就是0s，所以没有过期时间。
```
	@Override
	public void put(String name, byte[] key, byte[] value, @Nullable Duration ttl) {

		Assert.notNull(name, "Name must not be null!");
		Assert.notNull(key, "Key must not be null!");
		Assert.notNull(value, "Value must not be null!");

		execute(name, connection -> {

			if (shouldExpireWithin(ttl)) {
				connection.set(key, value, Expiration.from(ttl.toMillis(), TimeUnit.MILLISECONDS), SetOption.upsert());
			} else {
				connection.set(key, value);
			}

			return "OK";
		});

		statistics.incPuts(name);
	}

	private static boolean shouldExpireWithin(@Nullable Duration ttl) {
		return ttl != null && !ttl.isZero() && !ttl.isNegative();
	}
```

设置过期时间
```
    @Bean
    public CacheManager cacheManager(@Autowired RedisConnectionFactory connectionFactory) {
        return RedisCacheManager
                .builder(connectionFactory)
                .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(5)))
                .transactionAware()
                .build();
    }
```

为每个缓存设置不同的过期时间
```
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration defaultCacheConfig = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10)).
                serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer())).
                serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

        // 为每个缓存设置不同的过期时间
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        cacheConfigurations.put("test", defaultCacheConfig.entryTtl(Duration.ofMinutes(10)));

        return RedisCacheManager.builder(redisConnectionFactory)
                .cacheDefaults(defaultCacheConfig)
                .withInitialCacheConfigurations(cacheConfigurations)
                .build();
    }

```