---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot源码Redis序列化方式
使用Spring 提供的 Spring Data Redis 操作redis 必然要使用Spring提供的模板类 RedisTemplate， 今天我们好好的看看这个模板类 。

## RedisTemplate
```
public class RedisTemplate<K, V> extends RedisAccessor implements RedisOperations<K, V>, BeanClassLoaderAware {

	private boolean enableTransactionSupport = false;
	private boolean exposeConnection = false;
	private boolean initialized = false;
	private boolean enableDefaultSerializer = true;
	private @Nullable RedisSerializer<?> defaultSerializer;
	private @Nullable ClassLoader classLoader;

	@SuppressWarnings("rawtypes") private @Nullable RedisSerializer keySerializer = null;
	@SuppressWarnings("rawtypes") private @Nullable RedisSerializer valueSerializer = null;
	@SuppressWarnings("rawtypes") private @Nullable RedisSerializer hashKeySerializer = null;
	@SuppressWarnings("rawtypes") private @Nullable RedisSerializer hashValueSerializer = null;
	private RedisSerializer<String> stringSerializer = RedisSerializer.string();

	private @Nullable ScriptExecutor<K> scriptExecutor;

	private final ValueOperations<K, V> valueOps = new DefaultValueOperations<>(this);
	private final ListOperations<K, V> listOps = new DefaultListOperations<>(this);
	private final SetOperations<K, V> setOps = new DefaultSetOperations<>(this);
	private final StreamOperations<K, ?, ?> streamOps = new DefaultStreamOperations<>(this, new ObjectHashMapper());
	private final ZSetOperations<K, V> zSetOps = new DefaultZSetOperations<>(this);
	private final GeoOperations<K, V> geoOps = new DefaultGeoOperations<>(this);
	private final HyperLogLogOperations<K, V> hllOps = new DefaultHyperLogLogOperations<>(this);
	private final ClusterOperations<K, V> clusterOps = new DefaultClusterOperations<>(this);
```
看看4个序列化相关的属性 ，主要是 用于 KEY 和 VALUE 的序列化 。 举个例子，比如说我们经常会将POJO 对象存储到 Redis 中，一般情况下会使用 JSON 方式序列化成字符串，存储到 Redis 中 。

Spring提供的Redis数据结构的操作类
- ValueOperations 类，提供 Redis String API 操作
- ListOperations 类，提供 Redis List API 操作
- SetOperations 类，提供 Redis Set API 操作
- ZSetOperations 类，提供 Redis ZSet(Sorted Set) API 操作
- GeoOperations 类，提供 Redis Geo API 操作
- HyperLogLogOperations 类，提供 Redis HyperLogLog API 操作

## StringRedisTemplate
再看个常用的 StringRedisTemplate
```
public class StringRedisTemplate extends RedisTemplate<String, String> {
    
	public StringRedisTemplate() {
		setKeySerializer(RedisSerializer.string());
		setValueSerializer(RedisSerializer.string());
		setHashKeySerializer(RedisSerializer.string());
		setHashValueSerializer(RedisSerializer.string());
	}

```
RedisTemplate 支持泛型，StringRedisTemplate K V 均为String类型。

org.springframework.data.redis.core.StringRedisTemplate 继承 RedisTemplate 类，使用 org.springframework.data.redis.serializer.StringRedisSerializer 字符串序列化方式。

## RedisSerializer 序列化 接口
RedisSerializer接口 是 Redis 序列化接口，用于 Redis KEY 和 VALUE 的序列化

- OxmSerializer
XML 序列化方式。以xml格式存储（但还是String类型~），解析起来也比较复杂，效率也比较低。因此几乎没有人再使用此方式了
- JdkSerializationRedisSerializer
JDK 序列化方式 （默认）。从源码里可以看出，这是RestTemplate类默认的序列化方式。若你没有自定义，那就是它了。
- StringRedisSerializer
String 序列化方式。也是StringRedisTemplate默认的序列化方式，key和value都会采用此方式进行序列化，是被推荐使用的，对开发者友好，轻量级，效率也比较高。
- GenericToStringSerializer
他需要调用者给传一个对象到字符串互转的Converter（相当于转换为字符串的操作交给转换器去做），个人觉得使用起来其比较麻烦，还不如直接用字符串呢。所以不太推荐使用
- Jackson2JsonRedisSerializer
从名字可以看出来，这是把一个对象以Json的形式存储，效率高且对调用者友好。优点是速度快，序列化后的字符串短小精悍，不需要实现Serializable接口。
但缺点也非常致命：那就是此类的构造函数中有一个类型参数，必须提供要序列化对象的类型信息(.class对象)。 通过查看源代码，发现其在反序列化过程中用到了类型信息（必须根据此类型信息完成反序列化）。

- GenericJackson2JsonRedisSerializer
基本和上面的Jackson2JsonRedisSerializer功能差不多，使用方式也差不多，**但是是推荐使用的**

### JDK 序列化方式 （默认）
org.springframework.data.redis.serializer.JdkSerializationRedisSerializer ，默认情况下，RedisTemplate 使用该数据列化方式。

使用 JdkSerializationRedisSerializer 序列化对象可以方便地将 Java 对象转换为字节数组，并在 Redis 中进行存储和读取，但也存在对象版本兼容性问题和序列化性能差的风险。因此，在选择序列化方式时，需要根据实际情况和需求综合考虑。

我们来看下源码 RedisTemplate#afterPropertiesSet()
```
	@Override
	public void afterPropertiesSet() {

		super.afterPropertiesSet();

		boolean defaultUsed = false;

		if (defaultSerializer == null) {

			defaultSerializer = new JdkSerializationRedisSerializer(
					classLoader != null ? classLoader : this.getClass().getClassLoader());
		}
```
Spring Boot 自动化配置 RedisTemplate Bean 对象时，就未设置默认的序列化方式。

绝大多数情况下，不推荐使用 JdkSerializationRedisSerializer 进行序列化。主要是不方便人工排查数据。

KEY 前面带着奇怪的 16 进制字符 ， VALUE 也是一串奇怪的 16 进制字符 。。。。。

为什么是这样一串奇怪的 16 进制？ ObjectOutputStream#writeString(String str, boolean unshared) 实际就是标志位 + 字符串长度 + 字符串内容

KEY 被序列化成这样，线上通过 KEY 去查询对应的 VALUE非常不方便，所以 KEY 肯定是不能被这样序列化的。

VALUE 被序列化成这样，除了阅读可能困难一点，不支持跨语言外，实际上也没还OK。不过，实际线上场景，还是使用 JSON 序列化居多。

## String 序列化方式
org.springframework.data.redis.serializer.StringRedisSerializer ，字符串和二进制数组的直接转换

绝大多数情况下，我们 KEY 和 VALUE 都会使用这种序列化方案。

## JSON 序列化方式
org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer 使用 Jackson 实现 JSON 的序列化方式，并且从 Generic 单词可以看出，是支持所有类。
```
public GenericJackson2JsonRedisSerializer(@Nullable String classPropertyTypeName) {

			.....
			..... 
		if (StringUtils.hasText(classPropertyTypeName)) {
			mapper.enableDefaultTypingAsProperty(DefaultTyping.NON_FINAL, classPropertyTypeName);
		} else {
			mapper.enableDefaultTyping(DefaultTyping.NON_FINAL, As.PROPERTY);
		}
	}
```
classPropertyTypeName 不为空的话，使用传入对象的 classPropertyTypeName 属性对应的值，作为默认类型（Default Typing） ，否则使用传入对象的类全名，作为默认类型（Default Typing）。

我们来思考下，在将一个对象序列化成一个字符串，怎么保证字符串反序列化成对象的类型呢？Jackson 通过 Default Typing ，会在字符串多冗余一个类型，这样反序列化就知道具体的类型了

先说个结论

标准JSON
```
{
  "id": 100,
  "name": "小工匠",
  "sex": "Male"
}
```
使用 Jackson Default Typing 机制序列化
```
{
  "@class": "com.artisan.domain.Artisan",
  "id": 100,
  "name": "小工匠",
  "sex": "Male"
}
```

XML 序列化方式
org.springframework.data.redis.serializer.OxmSerializer使用 Spring OXM 实现将对象和 String 的转换，从而 String 和二进制数组的转换。 没见过哪个项目用过。

## Jackson2JsonRedisSerializer和GenericJackson2JsonRedisSerializer的异同
Jackson2JsonRedisSerializer：为我们提供了两个构造方法，一个需要传入序列化对象Class，一个需要传入对象的JavaType:
```
    public Jackson2JsonRedisSerializer(Class<T> type) {
        this.javaType = getJavaType(type);
    }
 
    public Jackson2JsonRedisSerializer(JavaType javaType) {
        this.javaType = javaType;
    }
```
这种的坏处，很显然，我们就不能全局使用统一的序列化方式了，而是每次调用RedisTemplate前，都需要类似这么处理：
```
redisTemplate.setKeySerializer(RedisSerializerType.StringSerializer.getRedisSerializer());
        redisTemplate.setValueSerializer(new Jackson2JsonRedisSerializer<>(Person.class));
```
但因为redisTemplate我们都是单例的，所以这样设置显然是非常不可取的行为。虽然它有好处 这种序列化方式的好处：他能实现不同的Project之间数据互通（因为没有@class信息，所以只要字段名相同即可），因为其实就是Json的返序列化，只要你指定了类型，就能反序列化成功（因为它和包名无关）

使用这种Json序列化方式果然是可以成功的在不同project中进行序列化和反序列化的。但是，但是，但是：在实际的使用中，我们希望职责单一和高内聚的，所以并不希望我们存在的对象，其它服务可以直接访问，那样就非常不好控制了，因此此种方式也不建议使用。

GenericJackson2JsonRedisSerializer：这种序列化方式不用自己手动指定对象的Class。所以其实我们就可以使用一个全局通用的序列化方式了。使用起来和JdkSerializationRedisSerializer基本一样。

同样的JdkSerializationRedisSerializer不能序列化和反序列化不同包路径对象的毛病它也有。因为它序列化之后的内容，是存储了对象的class信息的

### Jackson2JsonRedisSerializer的坑：
存储普通对象的时候没有问题，但是当我们存储带泛型的List的时候，反序化就会报错了：
```
    @Test
    public void contextLoads() {
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new Jackson2JsonRedisSerializer<>(List.class));
        ValueOperations<String, List<Person>> valueOperations = redisTemplate.opsForValue();
        valueOperations.set("aaa", Arrays.asList(new Person("fsx", 24), new Person("fff", 30)));
 
        List<Person> p = valueOperations.get("aaa");
        System.out.println(p); //[{name=fsx, age=24}, {name=fff, age=30}]
 
        List<Person> aaa = (List<Person>) redisTemplate.opsForValue().get("aaa");
        System.out.println(aaa); //[{name=fsx, age=24}, {name=fff, age=30}]
    
```




















