---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot整合Kafka


## SpringBoot整合Kafka
springboot提供自动配置整合kafka的方式，需要做一下步骤：

引入kafka依赖包：
```
   <dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>2.2.7.RELEASE</version>
   </dependency>
```

在springboot配置中加入kafka相关配置，springboot启动时候会自动加载这些配置，完成链接kafka,创建producer,consumer等。
```
spring:
  kafka:
    bootstrap-servers: 127.0.0.1:9092,127.0.0.1:9093,127.0.0.1:9094   # kafka集群信息
    # 消费者配置
    consumer:
      bootstrap-servers: 127.0.0.1:9092
      group-id: myGroup    # 消费者组
      # 消费者的偏移量是自动提交还是手动提交，此处自动提交偏移量
      enable-auto-commit: true # 关闭自动提交
      # 如果在kafka中找不到当前消费者的偏移量，则直接将偏移量重置为最早的
      auto-offset-reset: earliest # 当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费
      # 消费者偏移量自动提交的时间间隔
      auto-commit-interval: 1000
      max-poll-records: 10
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    # 生产者配置
    producer:
      retries: 5  # 设置大于0的值，则客户端会将发送失败的记录重新发送
      batch-size: 16384  #16K
      buffer-memory: 33554432 #32M
      acks: 1
      # 指定消息key和消息体的编解码方式
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
    listener:
      # 当每一条记录被消费者监听器（ListenerConsumer）处理之后提交
      # RECORD
      # 当每一批poll()的数据被消费者监听器（ListenerConsumer）处理之后提交
      # BATCH
      # 当每一批poll()的数据被消费者监听器（ListenerConsumer）处理之后，距离上次提交时间大于TIME时提交
      # TIME
      # 当每一批poll()的数据被消费者监听器（ListenerConsumer）处理之后，被处理record数量大于等于COUNT时提交
      # COUNT
      # TIME |　COUNT　有一个条件满足时提交
      # COUNT_TIME
      # 当每一批poll()的数据被消费者监听器（ListenerConsumer）处理之后, 手动调用Acknowledgment.acknowledge()后提交
      # MANUAL
      # 手动调用Acknowledgment.acknowledge()后立即提交，一般使用这种
      # MANUAL_IMMEDIATE
      ack-mode: manual_immediate
```

## 消息发送端
```java
@Service
public class KafkaProducer {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendMessage(String topic, String message) {
        kafkaTemplate.send(topic, message);
    }
}
```

## 消息消费端
```java
/**
 * kafka消费者
 */
@Component
public class KafkaConsumer {

    //kafka的监听器，topic为"zhTest"，消费者组为"zhTestGroup"
    @KafkaListener(topics = "test_topic", groupId = "zhTestGroup")
    public void receiveSkMessageInfo(ConsumerRecord<String, String> record, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic, Acknowledgment ack) {
        log.info(record.value());
        //手动提交offset
        ack.acknowledge();
    }
    
    //配置多个消费组
    @KafkaListener(topics = "test_topic",groupId = "zhTestGroup2")
    public void receiveSkMessageInfo(ConsumerRecord<String, String> record, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic, Acknowledgment ack) {
        log.info(record.value());
        //手动提交offset
        ack.acknowledge();
    }
}

```
以上实现是最简单的方式，但使用springboot自动配置的方式，所有配置项必须事先写好在在applicantion.yml的spring.kafka下面

springboot自动配置kafka是在KafkaAutoConfiguration这个类中实现的，它有一个成员KafkaProperties，这个properties中保存所有关于kafka的配置。
```
// 自动配置是在KafkaAutoConfiguration类实现的
@Configuration
@ConditionalOnClass(KafkaTemplate.class)
@EnableConfigurationProperties(KafkaProperties.class)
@Import({ KafkaAnnotationDrivenConfiguration.class,
        KafkaStreamsAnnotationDrivenConfiguration.class })
public class KafkaAutoConfiguration {

    private final KafkaProperties properties;
```
KafkaProperties类的注解可以看出，配置都是从yml里的spring.kafka配置读出来的
```
@ConfigurationProperties(prefix = "spring.kafka")
public class KafkaProperties {
```




























