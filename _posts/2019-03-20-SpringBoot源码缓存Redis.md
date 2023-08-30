---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot源码缓存Redis

## RedisAutoConfiguration
以spring-boot-starter-data-redis为例子，在spring-boot-autoconfigure中的spring.factories有引入RedisAutoConfiguration

RedisAutoConfiguration会有两个RedisTemplate，一个是默认的，一个是字符串序列化化的。

Jedis在实现上是直接连接的redis server，如果在多线程环境下是非线程安全的，这个时候只有使用连接池，为每个Jedis实例增加物理连接。

Lettuce的连接是基于Netty的，连接实例（StatefulRedisConnection）可以在多个线程间并发访问，应为StatefulRedisConnection是线程安全的，所以一个连接实例（StatefulRedisConnection）就可以满足多线程环境下的并发访问，当然这个也是可伸缩的设计，一个连接实例不够的情况也可以按需增加连接实例。lettuce主要利用netty实现与redis的同步和异步通信。
```
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(name = "redisTemplate")
	public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
			throws UnknownHostException {
		RedisTemplate<Object, Object> template = new RedisTemplate<>();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

	@Bean
	@ConditionalOnMissingBean
	public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory)
			throws UnknownHostException {
		StringRedisTemplate template = new StringRedisTemplate();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

}
```
我们可以看到RedisAutoConfiguration上有四个注解, 下面对这4个注解进行逐个分析.
- @Configuration(proxyBeanMethods = false)
如果配置类中的@Bean标识的方法之间不存在依赖调用的话，可以设置为false，可以避免拦截方法进行代理操作，提升性能。
- @ConditionalOnClass(RedisOperations.class)
@ConditionalOnClass表示在RedisOperations类型存在的时候启用, 因为我们导入了spring-boot-starter-data-redis中包含了RedisOperations类型, 因此条件满足启用配置类.
- @EnableConfigurationProperties(RedisProperties.class)
@EnableConfigurationProperties注解的作用是:让使用 @ConfigurationProperties 注解的类生效。 这里是让RedisProperties生效, 而RedisProperties类中属性就是接收yml文件里的配置
- @Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
@Import注解是向容器中注入bean, 首先注入的是LettuceConnectionConfiguration类型

LettuceConnectionConfiguration、JedisConnectionConfiguration有个公共功能是创建jedis连接工厂类之JedisConnectionFactory。由于先解析LettuceConnectionConfiguration类，所以两者配置类都满足的前提下，优先通过LettuceConnectionConfiguration创建JedisConnectionFactory。

### JedisConnectionConfiguration
```
@ConditionalOnClass({ GenericObjectPool.class, JedisConnection.class, Jedis.class })
class JedisConnectionConfiguration extends RedisConnectionConfiguration {
	@ConditionalOnMissingBean(RedisConnectionFactory.class)
	public JedisConnectionFactory redisConnectionFactory() throws UnknownHostException {
		return createJedisConnectionFactory();
	}
}
```
JedisConnectionConfiguration发挥作用的前提条件之一是存在Jedis类，该类是通过以下依赖引入的：
```xml
 <dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
```

### LettuceConnectionConfiguration
```
@Configuration
@ConditionalOnClass(RedisClient.class)
class LettuceConnectionConfiguration extends RedisConnectionConfiguration {

	@Bean
	@ConditionalOnMissingBean(RedisConnectionFactory.class)
	public LettuceConnectionFactory redisConnectionFactory(ClientResources clientResources)
			throws UnknownHostException {
		LettuceClientConfiguration clientConfig = getLettuceClientConfiguration(clientResources,
				this.properties.getLettuce().getPool());
		return createLettuceConnectionFactory(clientConfig);
	}
}
```
LettuceConnectionConfiguration配置类发挥作用的前提条件之一是存在RedisClient类，该类是通过以下依赖引入的：
```
<dependency>
   <groupId>io.lettuce</groupId>
   <artifactId>lettuce-core</artifactId>
   <version>5.1.7.RELEASE</version>
   <scope>compile</scope>
 </dependency>
```
在SpringBoot项目中spring-boot-starter-data-redis包下默认存在lettuce-core。

在构造器中将RedisProperties对象用properties属性接收.因为当前对象会被spring容器当做bean, 因此下面的源码将会执行.
```
@Bean
@ConditionalOnMissingBean(RedisConnectionFactory.class)
LettuceConnectionFactory redisConnectionFactory(
		ObjectProvider<LettuceClientConfigurationBuilderCustomizer> builderCustomizers,
		ClientResources clientResources) throws UnknownHostException {
		//获取lettuce客户端配置
	LettuceClientConfiguration clientConfig = getLettuceClientConfiguration(builderCustomizers, clientResources,
			getProperties().getLettuce().getPool());
			//创建connectionFactory对象
	return createLettuceConnectionFactory(clientConfig);
}
```

@ConditionalOnMissingBean(RedisConnectionFactory.class), 表示RedisConnectionFactory类型不存在的时候当前方法执行, 而在项目启动的时候这个类型确实不存在, 因此当前方法会执行.
```
private LettuceConnectionFactory createLettuceConnectionFactory(LettuceClientConfiguration clientConfiguration) {
	if (getSentinelConfig() != null) {
		return new LettuceConnectionFactory(getSentinelConfig(), clientConfiguration);
	}
	if (getClusterConfiguration() != null) {
		return new LettuceConnectionFactory(getClusterConfiguration(), clientConfiguration);
	}
	//单机redis
	return new LettuceConnectionFactory(getStandaloneConfig(), clientConfiguration);
}
```

## 获取客户端连接
```
class LettucePoolingConnectionProvider{
	
	private final Map<Class<?>, GenericObjectPool<StatefulConnection<?, ?>>> pools = new ConcurrentHashMap<>(32);
	
	public <T extends StatefulConnection<?, ?>> T getConnection(Class<T> connectionType) {

		GenericObjectPool<StatefulConnection<?, ?>> pool = pools.computeIfAbsent(connectionType, poolType -> {
			return ConnectionPoolSupport.createGenericObjectPool(() -> 
				  // connectionProvider：StandaloneConnectionProvider。工厂类PooledObjectFactory通过lambda表达式回调创建连接
				   connectionProvider.getConnection(connectionType),
				   poolConfig, false);
		});
		// pool：返回GenericObjectPool  调用ConnectionPoolSupport#createGenericObjectPool#borrowObject
		StatefulConnection<?, ?> connection = pool.borrowObject();
		poolRef.put(connection, pool);
		return connectionType.cast(connection);
	}
}
```
GenericObjectPool：管理连接核心类。也是实现池化技术常用手段。
```
public abstract class ConnectionPoolSupport {

	public static <T extends StatefulConnection<?, ?>> GenericObjectPool<T> createGenericObjectPool(
	        Supplier<T> connectionSupplier, GenericObjectPoolConfig config, boolean wrapConnections) {
	    
	    AtomicReference<Origin<T>> poolRef = new AtomicReference<>();
	    GenericObjectPool<T> pool = new GenericObjectPool<T>(new RedisPooledObjectFactory<T>(connectionSupplier), 	
	    				config) {
	        @Override
	        public T borrowObject() throws Exception {
	            return wrapConnections ? 
	            	   ConnectionWrapping.wrapConnection(super.borrowObject(), poolRef.get()) 
	            	   :super.borrowObject();//调用GenericObjectPool#borrowObject，最后是通过工厂类PooledObjectFactory的实现类RedisPooledObjectFactory创建连接
	        }
	
	        @Override
	        public void returnObject(T obj) {
	
	            if (wrapConnections && obj instanceof HasTargetConnection) {
	                super.returnObject((T) ((HasTargetConnection) obj).getTargetConnection());
	                return;
	            }
	            super.returnObject(obj);
	        }
	    };
	    poolRef.set(new ObjectPoolWrapper<>(pool));
	    return pool;
	}
}
```

在实例化LettuceConnectionFactory对象的时候会先判断配置的redis模式, 分别是通过sentinel, cluster属性是否配置的了值来进行判断的, 说明一下,sentinel指的是哨兵模式, cluster指的是集群模式, 本公司用的是单机模式, 因此会走单机方式的实例化方法, 其实大部分公司用redis都是单机版的, 只有数据量大的才会采用其他方式.

```
class StandaloneConnectionProvider implements LettuceConnectionProvider, TargetAware {
	private final RedisClient client;
	private final RedisCodec<?, ?> codec;
	private final Optional<ReadFrom> readFrom;
	private final Supplier<RedisURI> redisURISupplier;
	
	public <T extends StatefulConnection<?, ?>> T getConnection(Class<T> connectionType) {

		if (StatefulConnection.class.isAssignableFrom(connectionType)) {
			return connectionType.cast(readFrom.map(it -> this.masterReplicaConnection(redisURISupplier.get(), it))
					.orElseGet(() -> client.connect(codec)));//RedisClient#connect 章节4内容
		}
		throw new UnsupportedOperationException("Connection type " + connectionType + " not supported!");
	}
}
```

## Netty建立连接的过程
```
public class RedisClient extends AbstractRedisClient {
	public <K, V> StatefulRedisConnection<K, V> connect(RedisCodec<K, V> codec) {
		return getConnection(connectStandaloneAsync(codec, this.redisURI, timeout));
    }
	
	private <K, V> ConnectionFuture<StatefulRedisConnection<K, V>> connectStandaloneAsync(RedisCodec<K, V> codec,
            RedisURI redisURI, Duration timeout) {
		// 该类是Redis通过通道channel写数据的核心类，内部notifyChannelActive方法是通过Netty建立连接后在ChannelActive方法中回调赋值channel的过程
        DefaultEndpoint endpoint = new DefaultEndpoint(clientOptions, clientResources);
        RedisChannelWriter writer = endpoint;
        if (CommandExpiryWriter.isSupported(clientOptions)) {
        	//CommandExpiryWriter对DefaultEndpoint包装了一层
            writer = new CommandExpiryWriter(writer, clientOptions, clientResources);
        }
		// 返回存在状态的连接 StatefulRedisConnectionImpl，其实是对DefaultEndpoint的抽象
        StatefulRedisConnectionImpl<K, V> connection = newStatefulRedisConnection(writer, codec, timeout);
        ConnectionFuture<StatefulRedisConnection<K, V>> future = connectStatefulAsync(connection, codec, endpoint, 
        // lambda表达式是为了初始化Netty涉及的ChannelHandler之CommandHandler，既是出栈也是入栈Handler。也就是在该Handler内部的ChannelActive方法负责对DefaultEndpoint赋值channel的操作
        redisURI,() -> new CommandHandler(clientOptions, clientResources, endpoint));
        future.whenComplete((channelHandler, throwable) -> {
            if (throwable != null) {
                connection.close();
            }
        });
        return future;
    }
	
	private <K, V, S> ConnectionFuture<S> connectStatefulAsync(StatefulRedisConnectionImpl<K, V> connection,
            RedisCodec<K, V> codec, Endpoint endpoint,RedisURI redisURI, Supplier<CommandHandler> 	
            		commandHandlerSupplier) {

        ConnectionBuilder connectionBuilder = ConnectionBuilder.connectionBuilder();
 		connectionBuilder.connection(connection);//StatefulRedisConnectionImpl
        connectionBuilder.clientOptions(clientOptions);
        connectionBuilder.clientResources(clientResources);
        connectionBuilder.commandHandler(commandHandlerSupplier).endpoint(endpoint);//DefaultEndpoint
        // 核心创建Netty 客户端的启动器之Bootstrap
        connectionBuilder(getSocketAddressSupplier(redisURI), connectionBuilder, redisURI);
        channelType(connectionBuilder, redisURI);

        if (clientOptions.isPingBeforeActivateConnection()) {
            if (hasPassword(redisURI)) {
                connectionBuilder.enableAuthPingBeforeConnect();
            } else {
                connectionBuilder.enablePingBeforeConnect();
            }
        }
        ConnectionFuture<RedisChannelHandler<K, V>> future = initializeChannelAsync(connectionBuilder);//调用父类
        ConnectionFuture<?> sync = future;
        if (!clientOptions.isPingBeforeActivateConnection() && hasPassword(redisURI)) {

            sync = sync.thenCompose(channelHandler -> {
                CommandArgs<K, V> args = new CommandArgs<>(codec).add(redisURI.getPassword());
                return connection.async().dispatch(CommandType.AUTH, new StatusOutput<>(codec), args);
            });
        }

        if (LettuceStrings.isNotEmpty(redisURI.getClientName())) {
            sync = sync.thenApply(channelHandler -> {
                connection.setClientName(redisURI.getClientName());
                return channelHandler;
            });
        }

        if (redisURI.getDatabase() != 0) {
            sync = sync.thenCompose(channelHandler -> {
                CommandArgs<K, V> args = new CommandArgs<>(codec).add(redisURI.getDatabase());
                return connection.async().dispatch(CommandType.SELECT, new StatusOutput<>(codec), args);
            });
        }
        return sync.thenApply(channelHandler -> (S) connection);
    }
}
```

```
public abstract class AbstractRedisClient {

	protected <K, V, T extends RedisChannelHandler<K, V>> ConnectionFuture<T> initializeChannelAsync(
	        ConnectionBuilder connectionBuilder) {
	
	    Mono<SocketAddress> socketAddressSupplier = connectionBuilder.socketAddress();//Redis配置的url地址
	    CompletableFuture<SocketAddress> socketAddressFuture = new CompletableFuture<>();
	    CompletableFuture<Channel> channelReadyFuture = new CompletableFuture<>();
		// 这种方式涉及 rtjava中响应式编程，
		socketAddressSupplier.doOnError(socketAddressFuture::completeExceptionally)
				.doOnNext(socketAddressFuture::complete)
	            .subscribe(redisAddress -> {// 启动发布者Mono的流式处理
	                if (channelReadyFuture.isCancelled()) {
	                    return;
	                }
	                initializeChannelAsync0(connectionBuilder, channelReadyFuture, redisAddress);
	            }, channelReadyFuture::completeExceptionally);
		// thenApply：当channelReadyFuture执行完complete方法触发。表明客户单连接正常建立。
		// DefaultConnectionFuture也是RedisClient返回连接的最终形式。其中connection是对DefaultEndpoint抽象化的 
		//StatefulRedisConnectionImpl
	    return new DefaultConnectionFuture<>(socketAddressFuture, channelReadyFuture.thenApply(channel -> (T) connectionBuilder.connection()));
	}
	
	private void initializeChannelAsync0(ConnectionBuilder connectionBuilder, CompletableFuture<Channel> 	
							channelReadyFuture,SocketAddress redisAddress) {
	    Bootstrap redisBootstrap = connectionBuilder.bootstrap();
		//ChannelInitializer的子类 PlainChannelInitializer，包裹客户端全部的channelHandler，其中就包括上述CommandHandler
	    RedisChannelInitializer initializer = connectionBuilder.build();
	    //初始化ChannelInitializer,最终会将全部channelHandler绑定到NioSocketChannel的管道pipeline中
	    redisBootstrap.handler(initializer);
	    clientResources.nettyCustomizer().afterBootstrapInitialized(redisBootstrap);
	    // 异步控制Redis客户端连接创建的结果
	    CompletableFuture<Boolean> initFuture = initializer.channelInitialized();
	    // 真正驱动Netty组件开始创建客户端连接
	    ChannelFuture connectFuture = redisBootstrap.connect(redisAddress);
	
	    channelReadyFuture.whenComplete((c, t) -> {
	
	        if (t instanceof CancellationException) {
	            connectFuture.cancel(true);
	            initFuture.cancel(true);
	        }
	    });
		// 此处的监听器涉及一个任务future。添加监听器的过程其实是将任务future添加到Netty普通队列中
	    connectFuture.addListener(future -> {
	        if (!future.isSuccess()) {
	            connectionBuilder.endpoint().initialState();
	            channelReadyFuture.completeExceptionally(future.cause());
	            return;
	        }
			//当Netty逐步处理普通任务队列中所有任务时，会涉及当前任务的执行。
			//RedisChannelInitializer中所有handler之channelInactive、userEventTriggered、channelActive、exceptionCaught方法执行都会涉及回调initFuture。执行完毕后继续执行initFuture回调方法whenComplete，根据handler执行结果判断连接建立是否顺利
	        initFuture.whenComplete((success, throwable) -> {
	            if (throwable == null) {// Netty建立连接过程中没有异常产生，说明连接建立顺利
	            	// StatefulRedisConnectionImpl
	                RedisChannelHandler<?, ?> connection = connectionBuilder.connection();
	                connection.registerCloseables(closeableResources, connection);
	                channelReadyFuture.complete(connectFuture.channel());//此处触发异步执行任务channelReadyFuture的执行
	                return;
	            }
	            connectionBuilder.endpoint().initialState();
	            Throwable failure;
	
	            if (throwable instanceof RedisConnectionException) {
	                failure = throwable;
	            } else if (throwable instanceof TimeoutException) {
	                failure = new RedisConnectionException("Could not initialize channel within "
	                        + connectionBuilder.getTimeout(), throwable);
	            } else {
	                failure = throwable;
	            }
	            channelReadyFuture.completeExceptionally(failure);
	        });
	    });
	}
}
```
返回的客户端连接实例为DefaultConnectionFuture，其实内部已经抽象化DefaultEndpoint。能获取到DefaultEndpoint就能获取到Netty从缓存区ByteBuf刷数据到通道channel对应的NioSocketChannel。

## RedisTemplate
在yml文件中配置的redis参数其实都赋值给了RedisStandaloneConfiguration对象中的对应属性.这些配置信息最终都被包装在了LettuceConnectionFactory实例中.通过看LettuceConnectionFactory的源码我们会发现LettuceConnectionFactory实际上最终是实现了RedisConnectionFactory, 因此在RedisAutoConfiguration源码中的redisTemplate方法中注入的RedisConnectionFactory对象实际上是LettuceConnectionFactory

RedisTemplate实现了InitializingBean接口，在系统启动的时候，会重写afterPropertiesSet方法，核心就是设置序列化器。默认的序列化器是JdkSerializationRedisSerializer。
```
	public void afterPropertiesSet() {

		super.afterPropertiesSet();

		boolean defaultUsed = false;

		if (defaultSerializer == null) {

			defaultSerializer = new JdkSerializationRedisSerializer(
					classLoader != null ? classLoader : this.getClass().getClassLoader());
		}

		if (enableDefaultSerializer) {

			if (keySerializer == null) {
				keySerializer = defaultSerializer;
				defaultUsed = true;
			}
			if (valueSerializer == null) {
				valueSerializer = defaultSerializer;
				defaultUsed = true;
			}
			if (hashKeySerializer == null) {
				hashKeySerializer = defaultSerializer;
				defaultUsed = true;
			}
			if (hashValueSerializer == null) {
				hashValueSerializer = defaultSerializer;
				defaultUsed = true;
			}
		}

		if (enableDefaultSerializer && defaultUsed) {
			Assert.notNull(defaultSerializer, "default serializer null and not all serializers initialized");
		}

		if (scriptExecutor == null) {
			this.scriptExecutor = new DefaultScriptExecutor<>(this);
		}

		initialized = true;
	}
```

```
@Bean
@ConditionalOnMissingBean(name = "redisTemplate")
public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
		throws UnknownHostException {
	RedisTemplate<Object, Object> template = new RedisTemplate<>();
	template.setConnectionFactory(redisConnectionFactory);
	return template;
}
```
有些读者可能会有疑问了,不是还导入了JedisConnectionConfiguration了吗,为什么不会是JedisConnectionFactory, 仔细看redisTemplate()方法, 他里面将RedisTemplate交给了spring管理, 而方法上同时加了注解@ConditionalOnMissingBean(name = “redisTemplate”), 

这就意味着当LettuceConnectionFactory注入后, RedisTemplate类型的bean也已经注入了容器中, redisTemplate方法不可能再次执行了, 因此我们说springboot集成redis默认用的是lettuce客户端方式

## StringRedisTemplate
```
@Bean
@ConditionalOnMissingBean
public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory)
		throws UnknownHostException {
	StringRedisTemplate template = new StringRedisTemplate();
	template.setConnectionFactory(redisConnectionFactory);
	return template;
}
```
当然springboot也默认将StringRedisTemplate 类型的bean交给了容器管理, 因此这两种bean都是自动装配的.

StringRedisTemplate指定序列化器，new StringRedisSerializer(StandardCharsets.UTF_8)。
```
	public StringRedisTemplate() {
		setKeySerializer(RedisSerializer.string());
		setValueSerializer(RedisSerializer.string());
		setHashKeySerializer(RedisSerializer.string());
		setHashValueSerializer(RedisSerializer.string());
	}
```
## 序列化器

### JdkSerializationRedisSerializer
生成字节数组，SerializingConverter#convert-->Serializer#serializeToByteArray
```
    default byte[] serializeToByteArray(T object) throws IOException {
        ByteArrayOutputStream out = new ByteArrayOutputStream(1024);
        this.serialize(object, out);
        return out.toByteArray();
    }
```
反序列化，DeserializingConverter#convert-->DefaultDeserializer#deserialize。
```
    public Object deserialize(InputStream inputStream) throws IOException {
        ConfigurableObjectInputStream objectInputStream = new ConfigurableObjectInputStream(inputStream, this.classLoader);

        try {
            return objectInputStream.readObject();
        } catch (ClassNotFoundException var4) {
            throw new NestedIOException("Failed to deserialize object type", var4);
        }
    }
```

### StringRedisSerializer
```
	@Override
	public String deserialize(@Nullable byte[] bytes) {
		return (bytes == null ? null : new String(bytes, charset));
	}

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.redis.serializer.RedisSerializer#serialize(java.lang.Object)
	 */
	@Override
	public byte[] serialize(@Nullable String string) {
		return (string == null ? null : string.getBytes(charset));
	}
```

案例：
```
       JdkSerializationRedisSerializer jdkSerializationRedisSerializer = new JdkSerializationRedisSerializer();
        byte[] bytes = jdkSerializationRedisSerializer.serialize("abc");
        Object res = jdkSerializationRedisSerializer.deserialize(bytes);
        System.out.println(new String(bytes) + "," + res);

        RedisSerializer<String> stringRedisSerializer = RedisSerializer.string();
        byte[] strBytes = stringRedisSerializer.serialize("abc");
        Object strRes = stringRedisSerializer.deserialize(strBytes);
        System.out.println(new String(strBytes) + "," + strRes);
```

## RedisTemplate获取数据
使用方法
```
   @Autowired
    private RedisTemplate redisTemplate;

    @GetMapping("operate")
    public void operate() {
        redisTemplate.opsForValue().set("ccc", "111");
        redisTemplate.opsForValue().get("ccc")
    }
```

DefaultValueOperations#get(java.lang.Object)
```
	@Override
	public V get(Object key) {

		return execute(new ValueDeserializingRedisCallback(key) {

			@Override
			protected byte[] inRedis(byte[] rawKey, RedisConnection connection) {
				return connection.get(rawKey);
			}
		});
	}
```

AbstractOperations#execute，使用template执行回调函数。
```
	@Nullable
	<T> T execute(RedisCallback<T> callback) {
		return template.execute(callback, true);
	}
```

RedisTemplate#execute()，获取连接，执行回调函数。
```
	@Nullable
	public <T> T execute(RedisCallback<T> action, boolean exposeConnection, boolean pipeline) {

		Assert.isTrue(initialized, "template not initialized; call afterPropertiesSet() before using it");
		Assert.notNull(action, "Callback object must not be null");

		RedisConnectionFactory factory = getRequiredConnectionFactory();
		RedisConnection conn = RedisConnectionUtils.getConnection(factory, enableTransactionSupport);

		try {

			boolean existingConnection = TransactionSynchronizationManager.hasResource(factory);
			RedisConnection connToUse = preProcessConnection(conn, existingConnection);

			boolean pipelineStatus = connToUse.isPipelined();
			if (pipeline && !pipelineStatus) {
				connToUse.openPipeline();
			}

			RedisConnection connToExpose = (exposeConnection ? connToUse : createRedisConnectionProxy(connToUse));
			T result = action.doInRedis(connToExpose);

			// close pipeline
			if (pipeline && !pipelineStatus) {
				connToUse.closePipeline();
			}

			return postProcessResult(result, connToUse, existingConnection);
		} finally {
			RedisConnectionUtils.releaseConnection(conn, factory, enableTransactionSupport);
		}
	}
```
最终会调用RedisTemplate中的execute()方法, 这里有个enableTransactionSupport属性,表示是否开启了事务, 默认关闭, 这里提到了redis事务。

我们往下看else中获取RedisConnection对象的方法,这里获取的RedisConnection类型的对象就是一个代理对象。

这里的核心是通过LettuceConnectionFactory获取LettuceConnection对象, 这个对象就相当于我们连接数据库的Connection对象,我们可以通过spring.redis.lettuce配置连接信息.

```
@Nullable
protected StatefulRedisConnection<byte[], byte[]> getSharedConnection() {
	return shareNativeConnection ? (StatefulRedisConnection) getOrCreateSharedConnection().getConnection() : null;
}

@Nullable
StatefulConnection<E, E> getConnection() {

	synchronized (this.connectionMonitor) {
		//获取StatefulRedisConnectionImpl类型的连接,这里比较重要的点是只有连接不存在的时候,才去获取, 所有整个过程只有一个connection对象.
		if (this.connection == null) {
			this.connection = getNativeConnection();
		}

		if (getValidateConnection()) {
			validateConnection();
		}

		return this.connection;
	}
}

private StatefulConnection<E, E> getNativeConnection() {
	return connectionProvider.getConnection(StatefulConnection.class);
}

@Override
public <T extends StatefulConnection<?, ?>> T getConnection(Class<T> connectionType) {

	if (connectionType.equals(StatefulRedisSentinelConnection.class)) {
		return connectionType.cast(client.connectSentinel());
	}
	
	//通过RedisClient创建连接对象
	if (connectionType.equals(StatefulRedisPubSubConnection.class)) {
		return connectionType.cast(client.connectPubSub(codec));
	}

	if (StatefulConnection.class.isAssignableFrom(connectionType)) {

		return connectionType.cast(readFrom.map(it -> this.masterReplicaConnection(redisURISupplier.get(), it))
				.orElseGet(() -> client.connect(codec)));
	}

	throw new UnsupportedOperationException("Connection type " + connectionType + " not supported!");
}
```
getConnection方法获取的是StatefulRedisConnectionImpl类型的连接,这里比较重要的点是只有连接不存在的时候,才去获取,所有整个过程只有一个connection对象。

而这个StatefulRedisConnectionImpl是一个线程安全的对象,这就是为什么说lettuce模式是线程安全的。

因为我们用的是单机版的redis, 因此最终进入StandaloneConnectionProvider类中的getConnection()方法,并通过RedisClient创建连接对象
```
public <K, V> StatefulRedisConnection<K, V> connect(RedisCodec<K, V> codec) {
    checkForRedisURI();
	//先创建连接对象, 通过ConnectionFuture的获取线程的返回值
    return getConnection(connectStandaloneAsync(codec, this.redisURI, timeout));
}

  private <K, V> ConnectionFuture<StatefulRedisConnection<K, V>> connectStandaloneAsync(RedisCodec<K, V> codec,
           RedisURI redisURI, Duration timeout) {

       assertNotNull(codec);
       checkValidRedisURI(redisURI);

       logger.debug("Trying to get a Redis connection for: " + redisURI);

       DefaultEndpoint endpoint = new DefaultEndpoint(clientOptions, clientResources);
       RedisChannelWriter writer = endpoint;

       if (CommandExpiryWriter.isSupported(clientOptions)) {
           writer = new CommandExpiryWriter(writer, clientOptions, clientResources);
       }
		
		//实例化StatefulRedisConnectionImpl对象, 这个对象包装了代理对象
       StatefulRedisConnectionImpl<K, V> connection = newStatefulRedisConnection(writer, codec, timeout);
       ConnectionFuture<StatefulRedisConnection<K, V>> future = connectStatefulAsync(connection, codec, endpoint, redisURI,
               () -> new CommandHandler(clientOptions, clientResources, endpoint));

       future.whenComplete((channelHandler, throwable) -> {

           if (throwable != null) {
               connection.close();
           }
       });

       return future;
   }
```
先创建连接对象, 通过ConnectionFuture的获取线程的返回值,实例化StatefulRedisConnectionImpl对象, 这个对象包装了代理对象.
```
   protected <K, V> StatefulRedisConnectionImpl<K, V> newStatefulRedisConnection(RedisChannelWriter channelWriter,
           RedisCodec<K, V> codec, Duration timeout) {
       return new StatefulRedisConnectionImpl<>(channelWriter, codec, timeout);
   }

   public StatefulRedisConnectionImpl(RedisChannelWriter writer, RedisCodec<K, V> codec, Duration timeout) {

       super(writer, timeout);

       this.codec = codec;
       this.async = newRedisAsyncCommandsImpl();
       this.sync = newRedisSyncCommandsImpl();
       this.reactive = newRedisReactiveCommandsImpl();
   }
```
在StatefulRedisConnectionImpl的构造器中会实例化3种类型的commands实例, RedisAsyncCommandsImpl是一个异步API, RedisSyncCommandsImpl是一个同步API, RedisTemplate默认用的就是这个api, 但底层其实还是用的异步, 这个等分析执行API的时候再说, RedisReactiveCommandsImpl是一个响应式API, 在使用ReactiveRedisTemplate时执行该API, 因为RedisTemplate用的是同步API, 因此我们特别分析RedisSyncCommandsImpl构造器.
```
protected RedisCommands<K, V> newRedisSyncCommandsImpl() {
    return syncHandler(async(), RedisCommands.class, RedisClusterCommands.class);
}

//这个方法在newRedisSyncCommandsImpl方法中被调用, 这里返回一个异步对象, 而这个对象就是上面的RedisAsyncCommandsImpl
@Override
public RedisAsyncCommands<K, V> async() {
    return async;
}

//创建RedisCommands,RedisClusterCommands接口的代理
protected <T> T syncHandler(Object asyncApi, Class<?>... interfaces) {
    FutureSyncInvocationHandler h = new FutureSyncInvocationHandler((StatefulConnection<?, ?>) this, asyncApi, interfaces);
    return (T) Proxy.newProxyInstance(AbstractRedisClient.class.getClassLoader(), interfaces, h);
}
```
async()方法在newRedisSyncCommandsImpl方法中被调用, 这里返回一个异步对象, 而这个对象就是上面的RedisAsyncCommandsImpl, 这点很重要, 这就是为什么说同步执行其实执行的是异步命令, 因为在执行同步命令的时候,会通过反射执行RedisAsyncCommandsImpl的method, syncHandler是创建RedisCommands,RedisClusterCommands接口的代理, 而对应的InvocationHandler是FutureSyncInvocationHandler, 所以他们的delegate最终都会执行FutureSyncInvocationHandler.到这里代理对象就被创建完成了.

我们回到 RedisConnection conn = factory.getConnection()方法, 这个RedisConnection实际上是LettuceConnection类型, 上面说到StatefulRedisConnectionImpl是单实例的, 但是lettuce每次创建的LettuceConnection是多实例的, 每个LettuceConnection包装同一个StatefulRedisConnectionImpl实例, 这点很重要,所以整个项目的代理也只创建了一个。
```
LettuceConnection(@Nullable StatefulConnection<byte[], byte[]> sharedConnection,
		LettuceConnectionProvider connectionProvider, long timeout, int defaultDbIndex) {

	Assert.notNull(connectionProvider, "LettuceConnectionProvider must not be null.");

	this.asyncSharedConn = sharedConnection;
	this.connectionProvider = connectionProvider;
	this.timeout = timeout;
	this.defaultDbIndex = defaultDbIndex;
	this.dbIndex = this.defaultDbIndex;
}

	public final V doInRedis(RedisConnection connection) {
		byte[] result = inRedis(rawKey(key), connection);
		return deserializeValue(result);
	}
```
获取到连接对象后, 会new 一个LettuceConnection实例,同时将StatefulRedisConnectionImpl通过构造器赋值给asyncSharedConn属性, 这点很重要. 之后会执行T result = action.doInRedis(connToExpose); 因为这个action其实是ValueDeserializingRedisCallback, 我们可以看到会执行inRedis(),根据函数式编程的特性,实际上就是执行下面这个方法里的inRedis()方法

```
@Override
	public void set(K key, V value) {

		byte[] rawValue = rawValue(value);
		execute(new ValueDeserializingRedisCallback(key) {

			@Override
			protected byte[] inRedis(byte[] rawKey, RedisConnection connection) {	
			//实际上会执行这个方法,这里的connection就是LettuceConnection对象
				connection.set(rawKey, rawValue);
				return null;
			}
		}, true);
	}
```
实际会执行connection.set()方法,这里的connection就是LettuceConnection对象.

因为LettuceConnection继承了AbstractRedisConnection,而AbstractRedisConnection又实现了DefaultedRedisConnection, 因此这个set方法实现上执行的是DefaultedRedisConnection的set方法, 如果用的是StringRedisTemplate那么执行的就是DefaultedStringRedisConnection的set方法, 不管怎样,看到最后我们会发现,他们底层执行的方法是一样的, 这里我们以DefaultedRedisConnection为例分析
```
default Boolean set(byte[] key, byte[] value) {
	return stringCommands().set(key, value);
}
	//stringCommands方法调用的实际上是这个,获取LettuceStringCommands对象
	@Override
	public RedisStringCommands stringCommands() {
		return new LettuceStringCommands(this);
	}
public Boolean set(byte[] key, byte[] value) {

		Assert.notNull(key, "Key must not be null!");
		Assert.notNull(value, "Value must not be null!");

		try {
			if (isPipelined()) {
				pipeline(
						connection.newLettuceResult(getAsyncConnection().set(key, value), Converters.stringToBooleanConverter()));
				return null;
			}
			if (isQueueing()) {
				transaction(
						connection.newLettuceResult(getAsyncConnection().set(key, value), Converters.stringToBooleanConverter()));
				return null;
			}
			return Converters.stringToBoolean(getConnection().set(key, value));
		} catch (Exception ex) {
			throw convertLettuceAccessException(ex);
		}
	}
	
	//这里的connection就是LettuceConnetion
	public RedisClusterCommands<byte[], byte[]> getConnection() {
		return connection.getConnection();
	}

	protected RedisClusterCommands<byte[], byte[]> getConnection() {

		if (isQueueing()) {
			return getDedicatedConnection();
		}
		if (asyncSharedConn != null) {
			//在实例化LettuceConnection时已经将asyncSharedConn属性赋值StatefulRedisConnectionImpl,因此会执行当前方法, 默认获取的是同步对象,也就是代理对象
			if (asyncSharedConn instanceof StatefulRedisConnection) {
				return ((StatefulRedisConnection<byte[], byte[]>) asyncSharedConn).sync();
			}
			if (asyncSharedConn instanceof StatefulRedisClusterConnection) {
				return ((StatefulRedisClusterConnection<byte[], byte[]>) asyncSharedConn).sync();
			}
		}
		return getDedicatedConnection();
	}
```
这里是获取LettuceStringCommands对象, 最终调用LettuceStringCommands的getConnect(), 因为在返回LettuceCommands实例的时候,在构造器中将asyncSharedConn 属性赋值StatefulRedisConnectionImpl, 因为此通过sync()方法获取就是第二部分创建的代理对象,因为实际上会执行FutureSyncInvocationHandler的handleInvocation方法.
```
   FutureSyncInvocationHandler(StatefulConnection<?, ?> connection, Object asyncApi, Class<?>[] interfaces) {
        this.connection = connection;
        this.timeoutProvider = new TimeoutProvider(() -> connection.getOptions().getTimeoutOptions(), () -> connection
                .getTimeout().toNanos());
        this.asyncApi = asyncApi;
		//translator内部维护了一个map, 这个map的key是RedisAsyncCommandsImpl的命令方法, value是AbstractRedisAsyncCommands的命令方法, 而RedisAsyncCommandsImpl又继承了AbstractRedisAsyncCommands, 因此所有操作redis的命令都能获取到对应的执行命令
        this.translator = MethodTranslator.of(asyncApi.getClass(), interfaces);
    }
protected Object handleInvocation(Object proxy, Method method, Object[] args) throws Throwable {

        try {
			
			//实际会执行AbstractRedisAsyncCommands中的命令方法
            Method targetMethod = this.translator.get(method);
            Object result = targetMethod.invoke(asyncApi, args);

            if (result instanceof RedisFuture<?>) {

                RedisFuture<?> command = (RedisFuture<?>) result;

                if (isNonTxControlMethod(method.getName()) && isTransactionActive(connection)) {
                    return null;
                }

                long timeout = getTimeoutNs(command);

                return LettuceFutures.awaitOrCancel(command, timeout, TimeUnit.NANOSECONDS);
            }

            return result;
        } catch (InvocationTargetException e) {
            throw e.getTargetException();
        }
    }

//AbstractRedisAsyncCommands中的set()方法
    @Override
    public RedisFuture<String> set(K key, V value) {
        return dispatch(commandBuilder.set(key, value));
    }

    Command<K, V, String> set(K key, V value) {
        notNullKey(key);

        return createCommand(SET, new StatusOutput<>(codec), key, value);
    }

    protected <T> Command<K, V, T> createCommand(CommandType type, CommandOutput<K, V, T> output, K key, V value) {
        CommandArgs<K, V> args = new CommandArgs<K, V>(codec).addKey(key).addValue(value);
        return createCommand(type, output, args);
    }
    
    protected <T> Command<K, V, T> createCommand(CommandType type, CommandOutput<K, V, T> output, CommandArgs<K, V> args) {
        return new Command<K, V, T>(type, output, args);
    }
	
	//命令每次都实例化一个
    public Command(ProtocolKeyword type, CommandOutput<K, V, T> output, CommandArgs<K, V> args) {
        LettuceAssert.notNull(type, "Command type must not be null");
        this.type = type;
        this.output = output;
        this.args = args;
    }

//这里的connection就是StatefulRedisConnectionImpl, 在底层维护一个 tcp 连接,多个线程共享一个连接对象。同时会有一个ConnectionWatchdog[ChannelInboundHandlerAdapter继承netty]来维护连接,实现断连重连。因此是一个线程安全的对象
public <T> AsyncCommand<K, V, T> dispatch(RedisCommand<K, V, T> cmd) {
        AsyncCommand<K, V, T> asyncCommand = new AsyncCommand<>(cmd);
        RedisCommand<K, V, T> dispatched = connection.dispatch(asyncCommand);
        if (dispatched instanceof AsyncCommand) {
            return (AsyncCommand<K, V, T>) dispatched;
        }
        return asyncCommand;
    }
    @Override
    public <T> RedisCommand<K, V, T> dispatch(RedisCommand<K, V, T> command) {

        RedisCommand<K, V, T> toSend = preProcessCommand(command);

        try {
            return super.dispatch(toSend);
        } finally {
            if (command.getType().name().equals(MULTI.name())) {
                multi = (multi == null ? new MultiOutput<>(codec) : multi);
            }
        }
    }

//底层是通过netty通信发送命令
protected <T> RedisCommand<K, V, T> dispatch(RedisCommand<K, V, T> cmd) {

        if (debugEnabled) {
            logger.debug("dispatching command {}", cmd);
        }

        if (tracingEnabled) {

            RedisCommand<K, V, T> commandToSend = cmd;
            TraceContextProvider provider = CommandWrapper.unwrap(cmd, TraceContextProvider.class);

            if (provider == null) {
                commandToSend = new TracedCommand<>(cmd, clientResources.tracing()
                        .initialTraceContextProvider().getTraceContext());
            }

            return channelWriter.write(commandToSend);
        }

        return channelWriter.write(cmd);
    }
```
在 FutureSyncInvocationHandler的属性translator内部维护了一个map, 这个map的key是RedisAsyncCommandsImpl的命令方法, value是AbstractRedisAsyncCommands的命令方法,

而RedisAsyncCommandsImpl又继承了AbstractRedisAsyncCommands, 因此所有操作redis的命令都能获取到对应的执行命令, 因此实际会执行AbstractRedisAsyncCommands中对应的命令方法.

每个命令都会用Command包装, 实际发送命令会通过StatefulRedisConnectionImpl, 在整个项目中只有这一个实例, 它是线程安全的,在底层维护一个 tcp

连接,多个线程共享一个连接对象。同时会有一个`ConnectionWatchdog[ChannelInboundHandlerAdapter继承netty]`来维护连接,实现断连重连. 底层是通过netty通信发送命令.

ValueDeserializingRedisCallback#doInRedis，根据key进行序列化，获取到的value进行反序列化。
```
	public final V doInRedis(RedisConnection connection) {
		byte[] result = inRedis(rawKey(key), connection);
		return deserializeValue(result);
	}

	byte[] rawKey(Object key) {
		Assert.notNull(key, "non null key required");
		if (keySerializer() == null && key instanceof byte[]) {
			return (byte[]) key;
		}

		return keySerializer().serialize(key);
	}
```

## RedisTemplate存储数据
DefaultValueOperations#set(K, V)，对value进行序列化
```
	@Override
	public void set(K key, V value) {

		byte[] rawValue = rawValue(value);
		execute(new ValueDeserializingRedisCallback(key) {

			@Override
			protected byte[] inRedis(byte[] rawKey, RedisConnection connection) {
				connection.set(rawKey, rawValue);
				return null;
			}
		});
	}
```
ValueDeserializingRedisCallback#doInRedis，将key进行序列化，执行回调方法。
```
		public final V doInRedis(RedisConnection connection) {
			byte[] result = inRedis(rawKey(key), connection);
			return deserializeValue(result);
		}
```





















