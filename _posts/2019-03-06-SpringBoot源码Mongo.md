---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot源码MongoDB

## 自动装配配置
```
org.springframework.boot.autoconfigure.mongo.embedded.EmbeddedMongoAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoReactiveAutoConfiguration,\
```
Mongo自动装配类
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(MongoClient.class)
@EnableConfigurationProperties(MongoProperties.class)
@ConditionalOnMissingBean(type = "org.springframework.data.mongodb.MongoDatabaseFactory")
public class MongoAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(MongoClient.class)
	public MongoClient mongo(MongoProperties properties, Environment environment,
			ObjectProvider<MongoClientSettingsBuilderCustomizer> builderCustomizers,
			ObjectProvider<MongoClientSettings> settings) {
		return new MongoClientFactory(properties, environment,
				builderCustomizers.orderedStream().collect(Collectors.toList()))
						.createMongoClient(settings.getIfAvailable());
	}

}
```