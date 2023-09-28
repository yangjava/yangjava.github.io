---
layout: post
categories: [Mongodb]
description: none
keywords: MongoDB
---
# Spring源码MongoDB

## MongoDB常见的问题
Invalid mongo configuration, either uri or host/port/credentials/replicaSet must be specified

查看springframework\boot\autoconfigure\mongo\MongoClientFactorySupport中发现其中的validateConfiguration方法
```java
private void validateConfiguration() {
	if (hasCustomAddress() || hasCustomCredentials() || hasReplicaSet()) {
			Assert.state(this.properties.getUri() == null,
					"Invalid mongo configuration, either uri or host/port/credentials/replicaSet must be specified");
	}
}

private boolean hasCustomAddress() {
		return this.properties.getHost() != null || this.properties.getPort() != null;
}
private boolean hasCustomCredentials() {
		return this.properties.getUsername() != null && this.properties.getPassword() != null;
}
private boolean hasReplicaSet() {
	return this.properties.getReplicaSetName() != null;
}

```
会发现，如果使用uri，就不能使用host,port,username等字段，所以如果使用uri，就只使用uri，否则就使用后者。


## SpringBoot中MongoTemplate源码
```java
package org.springframework.boot.autoconfigure.data.mongo;

import com.mongodb.client.MongoClient;

import org.springframework.boot.autoconfigure.AutoConfigureAfter;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration;
import org.springframework.boot.autoconfigure.mongo.MongoProperties;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.gridfs.GridFsTemplate;

/**
 * {@link EnableAutoConfiguration Auto-configuration} for Spring Data's mongo support.
 * <p>
 * Registers a {@link MongoTemplate} and {@link GridFsTemplate} beans if no other beans of
 * the same type are configured.
 * <p>
 * Honors the {@literal spring.data.mongodb.database} property if set, otherwise connects
 * to the {@literal test} database.
 */
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ MongoClient.class, MongoTemplate.class })
@EnableConfigurationProperties(MongoProperties.class)
@Import({ MongoDataConfiguration.class, MongoDatabaseFactoryConfiguration.class,
		MongoDatabaseFactoryDependentConfiguration.class })
@AutoConfigureAfter(MongoAutoConfiguration.class)
public class MongoDataAutoConfiguration {

}
```

创建MongoDatabaseFactory,MongoDatabaseFactory工厂类
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(MongoDatabaseFactory.class)
@ConditionalOnSingleCandidate(MongoClient.class)
class MongoDatabaseFactoryConfiguration {

	@Bean
	MongoDatabaseFactorySupport<?> mongoDatabaseFactory(MongoClient mongoClient, MongoProperties properties) {
		return new SimpleMongoClientDatabaseFactory(mongoClient, properties.getMongoClientDatabase());
	}

}
```

## 如何创建索引
```java
@Configuration(proxyBeanMethods = false)
class MongoDataConfiguration {

	@Bean
	@ConditionalOnMissingBean
	MongoMappingContext mongoMappingContext(ApplicationContext applicationContext, MongoProperties properties,
			MongoCustomConversions conversions) throws ClassNotFoundException {
		PropertyMapper mapper = PropertyMapper.get().alwaysApplyingWhenNonNull();
		MongoMappingContext context = new MongoMappingContext();
		mapper.from(properties.isAutoIndexCreation()).to(context::setAutoIndexCreation);
		context.setInitialEntitySet(new EntityScanner(applicationContext).scan(Document.class));
		Class<?> strategyClass = properties.getFieldNamingStrategy();
		if (strategyClass != null) {
			context.setFieldNamingStrategy((FieldNamingStrategy) BeanUtils.instantiateClass(strategyClass));
		}
		context.setSimpleTypeHolder(conversions.getSimpleTypeHolder());
		return context;
	}

	@Bean
	@ConditionalOnMissingBean
	MongoCustomConversions mongoCustomConversions() {
		return new MongoCustomConversions(Collections.emptyList());
	}

}
```
会扫描所有的@Document注解的类，放入到MongoMappingContext
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnBean(MongoDatabaseFactory.class)
class MongoDatabaseFactoryDependentConfiguration {

	private final MongoProperties properties;

	MongoDatabaseFactoryDependentConfiguration(MongoProperties properties) {
		this.properties = properties;
	}

	@Bean
	@ConditionalOnMissingBean(MongoOperations.class)
	MongoTemplate mongoTemplate(MongoDatabaseFactory factory, MongoConverter converter) {
		return new MongoTemplate(factory, converter);
	}
    ......
```
创建MongoTemplate
```java
   public MongoTemplate(MongoDatabaseFactory mongoDbFactory, @Nullable MongoConverter mongoConverter) {
        this.writeConcernResolver = DefaultWriteConcernResolver.INSTANCE;
        this.writeResultChecking = WriteResultChecking.NONE;
        this.sessionSynchronization = SessionSynchronization.ON_ACTUAL_TRANSACTION;
        Assert.notNull(mongoDbFactory, "MongoDbFactory must not be null!");
        this.mongoDbFactory = mongoDbFactory;
        this.exceptionTranslator = mongoDbFactory.getExceptionTranslator();
        this.mongoConverter = mongoConverter == null ? getDefaultMongoConverter(mongoDbFactory) : mongoConverter;
        this.queryMapper = new QueryMapper(this.mongoConverter);
        this.updateMapper = new UpdateMapper(this.mongoConverter);
        this.schemaMapper = new MongoJsonSchemaMapper(this.mongoConverter);
        this.projectionFactory = new SpelAwareProxyProjectionFactory();
        this.operations = new EntityOperations(this.mongoConverter.getMappingContext());
        this.propertyOperations = new PropertyOperations(this.mongoConverter.getMappingContext());
        this.queryOperations = new QueryOperations(this.queryMapper, this.updateMapper, this.operations, this.propertyOperations, mongoDbFactory);
        this.mappingContext = this.mongoConverter.getMappingContext();
        if (this.mappingContext instanceof MongoMappingContext) {
            MongoMappingContext mappingContext = (MongoMappingContext)this.mappingContext;
            if (mappingContext.isAutoIndexCreation()) {
                this.indexCreator = new MongoPersistentEntityIndexCreator(mappingContext, this);
                this.eventPublisher = new MongoMappingEventPublisher(this.indexCreator);
                mappingContext.setApplicationEventPublisher(this.eventPublisher);
            }
        }

    }
```
大致的逻辑梳理一下：系统启动的时候，spring-boot-autoconfigure根据spring.factories找那个配置的MongoDataAutoConfiguration去加载mongodb相关的配置，MongoDataAutoConfiguration这个类又会继续加载MongoDataConfiguration和MongoDbFactoryDependentConfiguration。MongoDataConfiguration在构建MongoMappingContext的时候回去扫描系统中所有的@Document和@Persistent，这就找到了所有的domain对象，MongoDbFactoryDependentConfiguration在构建mongoTemplate的时候，会遍历MongoMappingContext中的domain对象，然后查找对象是否需要创建索引并创建。如果创建索引的过程中出现DataIntegrityViolationException数据完整性约束的异常，直接忽略，因为有可能被另一个应用先创建出来了。