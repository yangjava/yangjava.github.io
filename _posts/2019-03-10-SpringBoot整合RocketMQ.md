---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot整合RocketMQ

## SpringBoot整合RocketMQ
在使用SpringBoot的starter集成包时，要特别注意版本。添加依赖
第一个是原生依赖，第二个是spring-boot-starter，这里我们添加第二个依赖。

```xml
<!-- https://mvnrepository.com/artifact/org.apache.rocketmq/rocketmq-client -->
        <dependency>
            <groupId>org.apache.rocketmq</groupId>
            <artifactId>rocketmq-client</artifactId>
            <version>4.9.0</version>
        </dependency>
        
 <!-- https://mvnrepository.com/artifact/org.apache.rocketmq/rocketmq-spring-boot-starter -->
        <dependency>
            <groupId>org.apache.rocketmq</groupId>
            <artifactId>rocketmq-spring-boot-starter</artifactId>
            <version>2.2.0</version>
        </dependency>
```

### 配置文件properties
```properties
rocketmq.name-server=127.0.0.1:9876
rocketmq.producer.group=provider
```
### 实现发送者
```java
@RequestMapping("redis")
@RestController
public class RedisKeyController {

    private static final Logger logger = LoggerFactory.getLogger(RedisKeyController.class);

    @Autowired
    RocketMQTemplate template;

    @Autowired
    StringRedisTemplate stringRedisTemplate;

    @GetMapping("put")
    public void putKey(String key, String value) {
        stringRedisTemplate.opsForValue().set(key, value);
    }

    @GetMapping("delete")
    public void delete(String key) {
        logger.info("key : {} is send to MQ ", key);
        try {
            template.convertAndSend(RedisConstant.TOPIC, key);
        } catch (MessagingException e) {
            e.printStackTrace();
        }
    }
}
```

### 实现消费者
写一个消费消息的实现类，这里我们接受消息来删除Redis中的一个key。

```java
@Service
@RocketMQMessageListener(consumerGroup = RedisKeyListener.GROUP,
        topic = RedisKeyListener.TOPIC,
        consumeMode = ConsumeMode.ORDERLY)
public class RedisKeyListener implements RocketMQListener<String> {

    public static final String GROUP = "redis_group";
    public static final String TOPIC = "redis_topic";

    private static final Logger logger = LoggerFactory.getLogger(RedisKeyListener.class);

    @Autowired
    StringRedisTemplate stringRedisTemplate;

    @Override
    public void onMessage(String key) {
        logger.info("redis consumer work for key : {} ", key);
        stringRedisTemplate.delete(key);
    }
}
```

### RocketMQMessageListener
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RocketMQMessageListener {
	//配置文件中读取
    String NAME_SERVER_PLACEHOLDER = "${rocketmq.name-server:}";
    String ACCESS_KEY_PLACEHOLDER = "${rocketmq.consumer.access-key:}";
    String SECRET_KEY_PLACEHOLDER = "${rocketmq.consumer.secret-key:}";
    String TRACE_TOPIC_PLACEHOLDER = "${rocketmq.consumer.customized-trace-topic:}";
    String ACCESS_CHANNEL_PLACEHOLDER = "${rocketmq.access-channel:}";
	//消费组
    String consumerGroup();
	//指定topic
    String topic();
	//如何选择消息
    SelectorType selectorType() default SelectorType.TAG;
	//根据表达式选择消息
    String selectorExpression() default "*";
	//消费模式
    ConsumeMode consumeMode() default ConsumeMode.CONCURRENTLY;
	//消息模式
    MessageModel messageModel() default MessageModel.CLUSTERING;
    
    @Deprecated
    int consumeThreadMax() default 64;
	//消费线程数
    int consumeThreadNumber() default 20;
	//最大重复消费次数
    int maxReconsumeTimes() default -1;
	//阻塞线程最大时间
    long consumeTimeout() default 15L;
	//回复时间
    int replyTimeout() default 3000;
	//替换配置文件的ACCESS_KEY_PLACEHOLDER
    String accessKey() default ACCESS_KEY_PLACEHOLDER;
	//替换配置文件的SECRET_KEY_PLACEHOLDER
    String secretKey() default SECRET_KEY_PLACEHOLDER;
    //是否开启消息追踪
    boolean enableMsgTrace() default false;
	//替换配置文件的TRACE_TOPIC_PLACEHOLDER
    String customizedTraceTopic() default TRACE_TOPIC_PLACEHOLDER;
	//替换配置文件的NAME_SERVER_PLACEHOLDER
    String nameServer() default NAME_SERVER_PLACEHOLDER;
	//替换配置文件的ACCESS_CHANNEL_PLACEHOLDER;
    String accessChannel() default ACCESS_CHANNEL_PLACEHOLDER;
 	//是否开启tls
    String tlsEnable() default "false";
	//命名空间
    String namespace() default "";
	//并发模式重试策略
    int delayLevelWhenNextConsume() default 0;
	//暂停时间间隔
    int suspendCurrentQueueTimeMillis() default 1000;
	//关闭等待时间
    int awaitTerminationMillisWhenShutdown() default 1000;
}
```

### ROCKETMQ-SPRING-BOOT-STARTER源码解析
spring-boot-starter 是模块化编程的方式，添加依赖就可以实现自动装配，自动装配需要实现接口配在配置文件中，这就是模块化编程中提倡的面向接口编程。在 rocketmq-spring-boot-starter源码中 的rocketmq-spring-boot模块下，找到resources文件夹，下的META-INF文件夹下有一个spring.factories的文件。这里配置了自动装配的类，也就是这个模块被整合时候的程序入口。如下图，这是spring标准的配置整合方式

SpringBoot会默认开启自动配置模式, 扫描各jar包下的spring.factories文件, 并对文件中指定的类注入到容器中

```text
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.apache.rocketmq.spring.autoconfigure.RocketMQAutoConfiguration
```
RocketMQAutoConfiguration 通过Import方式, 向容器中导入了ListenerContainerConfiguration类

```java
@Configuration
@EnableConfigurationProperties(RocketMQProperties.class)
@ConditionalOnClass({ MQAdmin.class, ObjectMapper.class })
@ConditionalOnProperty(prefix = "rocketmq", value = "name-server")
@Import({ JacksonFallbackConfiguration.class, ListenerContainerConfiguration.class })
@AutoConfigureAfter(JacksonAutoConfiguration.class)
public class RocketMQAutoConfiguration {
	...
}
```
@EnableConfigurationProperties(RocketMQProperties.class) 这个注解触发自动导入项目配置文件中RocketMQ的配置，封装到 RocketMQProperties这个类中注入到spring中，这个类中可以查看nameServer等配置项和默认配置项等参数。

@ConditionalOnClass({MQAdmin.class})
@ConditionalOnProperty(prefix = “rocketmq”, value = “name-server”, matchIfMissing = true)
这两个注解标注了触发这个模块自动装配的条件。

@Import({MessageConverterConfiguration.class, ListenerContainerConfiguration.class, ExtProducerResetConfiguration.class, RocketMQTransactionConfiguration.class})
@AutoConfigureAfter({MessageConverterConfiguration.class})
@AutoConfigureBefore({RocketMQTransactionConfiguration.class})
这三个注解标识了自动装配的时候需要初始化导入的类和，初始化的导入顺序。

### 整合生产者的源码分析
```java
@Bean
@ConditionalOnMissingBean(DefaultMQProducer.class)
@ConditionalOnProperty(prefix = "rocketmq", value = {"name-server", "producer.group"})
public DefaultMQProducer defaultMQProducer(RocketMQProperties rocketMQProperties) {
    RocketMQProperties.Producer producerConfig = rocketMQProperties.getProducer();
    String nameServer = rocketMQProperties.getNameServer();
    String groupName = producerConfig.getGroup();
    Assert.hasText(nameServer, "[rocketmq.name-server] must not be null");
    Assert.hasText(groupName, "[rocketmq.producer.group] must not be null");

    String accessChannel = rocketMQProperties.getAccessChannel();

    String ak = rocketMQProperties.getProducer().getAccessKey();
    String sk = rocketMQProperties.getProducer().getSecretKey();
    boolean isEnableMsgTrace = rocketMQProperties.getProducer().isEnableMsgTrace();
    String customizedTraceTopic = rocketMQProperties.getProducer().getCustomizedTraceTopic();

    DefaultMQProducer producer = RocketMQUtil.createDefaultMQProducer(groupName, ak, sk, isEnableMsgTrace, customizedTraceTopic);

    producer.setNamesrvAddr(nameServer);
    if (!StringUtils.isEmpty(accessChannel)) {
        producer.setAccessChannel(AccessChannel.valueOf(accessChannel));
    }
    producer.setSendMsgTimeout(producerConfig.getSendMessageTimeout());
    producer.setRetryTimesWhenSendFailed(producerConfig.getRetryTimesWhenSendFailed());
    producer.setRetryTimesWhenSendAsyncFailed(producerConfig.getRetryTimesWhenSendAsyncFailed());
    producer.setMaxMessageSize(producerConfig.getMaxMessageSize());
    producer.setCompressMsgBodyOverHowmuch(producerConfig.getCompressMessageBodyThreshold());
    producer.setRetryAnotherBrokerWhenNotStoreOK(producerConfig.isRetryNextServer());

    return producer;
}
```

```java
@Bean(destroyMethod = "destroy")
@ConditionalOnBean(DefaultMQProducer.class)
@ConditionalOnMissingBean(name = ROCKETMQ_TEMPLATE_DEFAULT_GLOBAL_NAME)
public RocketMQTemplate rocketMQTemplate(DefaultMQProducer mqProducer,
    RocketMQMessageConverter rocketMQMessageConverter) {
    RocketMQTemplate rocketMQTemplate = new RocketMQTemplate();
    rocketMQTemplate.setProducer(mqProducer);
    rocketMQTemplate.setMessageConverter(rocketMQMessageConverter.getMessageConverter());
    return rocketMQTemplate;
}
```
从上面两个方法中可以看到，public DefaultMQProducer defaultMQProducer(RocketMQProperties rocketMQProperties)这个方法先是创建了生产者对象注册到了spring容器中，这个方法又使用了@EnableConfigurationProperties(RocketMQProperties.class)注解引入的配置文件中的配置项来创建生产者。

public RocketMQTemplate rocketMQTemplate(DefaultMQProducer mqProducer, RocketMQMessageConverter rocketMQMessageConverter)方法则是使用上一个方法创建的 DefaultMQProducer 生产者对象和@Import注解引入的MessageConverterConfiguration对象。创建 RocketMQTemplate 对象封装 DefaultMQProducer 生产者对象 和 MessageConverterConfiguration对象，然后注册到容器中。这也就是为什么使用@Autowired注入RocketMQTemplate对象就能够发送消息的原因，其实就是进行简单封装后底层还是调用DefaultMQProducer 生产者对象发送消息。

看到这里，你是不是有点疑惑，创建RocketMQTemplate 的时候指定了销毁的时候@Bean(destroyMethod = “destroy”)指定了要调用的方法，实际还是调用producer的shutdown()方法释放资源，可是创建生产者的时候并没有指明调用 producer.start()方法？

### 整合消费者的源码解析

使用过 rocketmq-spring-boot-starter应该了解，实现消费者需要类实现RocketMQListener接口 并且要使用@RocketMQMessageListener标准消费者的信息 和@Service向Spring注册服务，就实现了消息消费的监听。

上面开始介绍入口的时候@Import 注解中有导入ListenerContainerConfiguration.class这个类，看类名就知道这个应该跟消息消费监听有关。查看这个类，空参构造方法没有进行相关初始化操作，但是类实现了两个接口如下：

org.apache.rocketmq.spring.autoconfigure.ListenerContainerConfiguration实现了SmartInitializingSingleton接口, 在容器实例化bean之后会调用此bean的afterSingletonsInstantiated()

```java
public class ListenerContainerConfiguration implements ApplicationContextAware, SmartInitializingSingleton {
	...
	
    @Override
    public void afterPropertiesSet() throws Exception {
    	// 默认初始化了push模式的consumer
        initRocketMQPushConsumer();

        this.messageType = getMessageType();
        log.debug("RocketMQ messageType: {}", messageType.getName());
    }

    @Override
    public void afterSingletonsInstantiated() {
    	// 根据注解从容器中获取bean
        Map<String, Object> beans = this.applicationContext.getBeansWithAnnotation(RocketMQMessageListener.class);

        if (Objects.nonNull(beans)) {
        	// 逐个处理
            beans.forEach(this::registerContainer);
        }
    }

    private void registerContainer(String beanName, Object bean) {
        Class<?> clazz = AopProxyUtils.ultimateTargetClass(bean);
		// 判断类型是否为RocketMQListener
        if (!RocketMQListener.class.isAssignableFrom(bean.getClass())) {
            throw new IllegalStateException(clazz + " is not instance of " + RocketMQListener.class.getName());
        }
		// 获取注解信息 并校验
        RocketMQMessageListener annotation = clazz.getAnnotation(RocketMQMessageListener.class);
        validate(annotation);

		// 生成beanName -> org.apache.rocketmq.spring.support.DefaultRocketMQListenerContainer_1
        String containerBeanName = String.format("%s_%s", DefaultRocketMQListenerContainer.class.getName(),
            counter.incrementAndGet());
        GenericApplicationContext genericApplicationContext = (GenericApplicationContext) applicationContext;
		
		// 注册bean 类型为DefaultRocketMQListenerContainer 重点  
		// Spring在bean创建对象时会回调supplier, 即createRocketMQListenerContainer()创建对象
        genericApplicationContext.registerBean(containerBeanName, DefaultRocketMQListenerContainer.class,
            () -> createRocketMQListenerContainer(bean, annotation));
        // 从容器中获取对应的bean, 容器会对刚注入的bean创建对象, 会调用createRocketMQListenerContainer()
        DefaultRocketMQListenerContainer container = genericApplicationContext.getBean(containerBeanName,
            DefaultRocketMQListenerContainer.class);
        // 校验状态 初始不是running状态
        if (!container.isRunning()) {
            try {
            	// 执行start方法
                container.start();
            } catch (Exception e) {
                log.error("Started container failed. {}", container, e);
                throw new RuntimeException(e);
            }
        }

        log.info("Register the listener to container, listenerBeanName:{}, containerBeanName:{}", beanName, containerBeanName);
    }

    private DefaultRocketMQListenerContainer createRocketMQListenerContainer(Object bean, RocketMQMessageListener annotation) {
    	// 此处只是简单的对注解进行解析封装成一个ListenerContainer
        DefaultRocketMQListenerContainer container = new DefaultRocketMQListenerContainer();

        container.setNameServer(rocketMQProperties.getNameServer());
        container.setTopic(environment.resolvePlaceholders(annotation.topic()));
        container.setConsumerGroup(environment.resolvePlaceholders(annotation.consumerGroup()));
        container.setRocketMQMessageListener(annotation);
        container.setRocketMQListener((RocketMQListener) bean);
        container.setObjectMapper(objectMapper);

        return container;
    }
}

```
ApplicationContextAware 接口可以获取到spring容器的上下文对象，SmartInitializingSingleton接口实现的方法在是会在这个类对象实例化完成后调用这个接口的方法的。ListenerContainerConfiguration这个类是@Configuration修饰的配置类，是由spring进行实例会操作管理的，所以会自动调用到SmartInitializingSingleton接口实现的方法:
```java
public void afterSingletonsInstantiated() {
   Map<String, Object> beans = this.applicationContext.getBeansWithAnnotation(RocketMQMessageListener.class)
       .entrySet().stream().filter(entry -> !ScopedProxyUtils.isScopedTarget(entry.getKey()))
       .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

   beans.forEach(this::registerContainer);
}
```
这个方法中我们可以看到applicationContext.getBeansWithAnnotation(RocketMQMessageListener.class)获取到了Spring管理对象中被RocketMQMessageListener注解标注的所有对象。这也是我们上面说的实现消费者监听需要满足的三个条件中的两个：使用@RocketMQMessageListener注解 和使用@Service注解向Spring注册服务。
上面方法流程总结如下：
- 获取到RocketMQMessageListener注解
- 生成注入到Spring容器中的Bean的名字
- 向spring注册DefaultRocketMQListenerContainer类的Bean
- 获取到容器中的Bean,开启消费者监听
  重点要注意的代码如下, 注解标注的类（我们实现的消费者接口的类）被复制给了DefaultRocketMQListenerContainer 中的引用。
```java
if (RocketMQListener.class.isAssignableFrom(bean.getClass())) {
	container.setRocketMQListener((RocketMQListener) bean);
} else if (RocketMQReplyListener.class.isAssignableFrom(bean.getClass())) {
    container.setRocketMQReplyListener((RocketMQReplyListener) bean);
}
```
这时候查看DefaultRocketMQListenerContainer 对象，发现它实现了我们上面提到的InitializingBean接口，DefaultRocketMQListenerContainer 对象所以初始化设置完成后会调用到afterPropertiesSet()方法，方法实现如下:
```java
@Override
public void afterPropertiesSet() throws Exception {
    initRocketMQPushConsumer();
    this.messageType = getMessageType();
    this.methodParameter = getMethodParameter();
    log.debug("RocketMQ messageType: {}", messageType);
}
```
看方法名就知道第一行代码对消费者进行初始化，到这里终于找到消费者的影子了。initRocketMQPushConsumer（）方法很长，我不贴出来了，方法里面创建消费者对象，并根据上面获取到的注解信息，设置订阅的topic 、过滤条件、广播消费还是并发消费等等。


接下来会执行start方法, 开启监听 org.apache.rocketmq.client.consumer.DefaultMQPushConsumer#start
```java
public void start() throws MQClientException {
	// 此处重点 开启监听
    this.defaultMQPushConsumerImpl.start();
    // 校验traceDispatcher是否为空 不为空也开启start 默认不是空为AsyncTraceDispatcher, 但是会因为循环执行start 抛出异常 打印log
    if (null != traceDispatcher) {
        try {
            traceDispatcher.start(this.getNamesrvAddr());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}
```

### RocketMQMessageListener参数问题
- consumerGroup 消费者分组
- topic 主题
- selectorType 消息选择器类型
   默认值 SelectorType.TAG 根据TAG选择
   仅支持表达式格式如：“tag1 || tag2 || tag3”，如果表达式为null或者“*”标识订阅所有消息
   SelectorType.SQL92 根据SQL92表达式选择
- selectorExpression 选择器表达式
   默认值 ”*“
- consumeMode 消费模式
   默认值 ConsumeMode.CONCURRENTLY 并行处理
   ConsumeMode.ORDERLY 按顺序处理
- messageModel 消息模型
   默认值 MessageModel.CLUSTERING 集群
   MessageModel.BROADCASTING 广播
- consumeThreadMax 最大线程数
   默认值 64
- consumeTimeout 超时时间
   默认值 30000ms
- accessKey
   默认值 ${rocketmq.consumer.access-key:}
- secretKey
    默认值 ${rocketmq.consumer.secret-key:}
- enableMsgTrace 启用消息轨迹
    默认值 true
- customizedTraceTopic 自定义的消息轨迹主题
    默认值 ${rocketmq.consumer.customized-trace-topic:}
    没有配置此配置项则使用默认的主题
- nameServer 命名服务器地址
    默认值 ${rocketmq.name-server:}
- accessChannel
    默认值 ${rocketmq.access-channel:}
