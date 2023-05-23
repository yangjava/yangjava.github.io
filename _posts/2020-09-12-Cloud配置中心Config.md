---
layout: post
categories: [SpringCloud]
description: none
keywords: SpringCloud
---
# SpringCloud配置中心
使用Spring Cloud Config作为SpringBoot微服务体系结构的配置中心，让您轻松管理配置信息并独立部署。

## 快速启动
引入一个新的配置服务器，并将ProductService和UserService微服务中的配置统一存放到Git仓库中，最后对这两个微服务进行改造，升级为配置客户端，从而实现配置的统一管理。

### 构建配置服务器
配置服务器是一个标准的Spring Boot应用，其可以嵌入在其他服务中以提供配置服务，可以不作为一个独立的服务进行部署。这里为了方便讲解，还是将配置服务器独立成一个单独的服务器，并进行部署。

### 编写依赖
对于一个单独部署的服务，首先需要编写一个Maven配置文件（pom.xml），在配置文件中加入对spring-cloud-config-server的依赖。
```xml
<artifactId>config-server</artifactId>
<name>SpringCloud Demo Projects(Config) -- Config Server</name>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>    
</dependencies>
```

### 编写启动类（Application）
在Application类的注解中增加@EnableConfigServer注解，用以启动配置服务。
```java
// 最重要的就是增加该注解，启动配置服务
@EnableConfigServer
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 编写配置文件（application.properties）
```properties
server.port=8888

spring.application.name=config-server

spring.cloud.config.server.git.uri=https://github.com/SpringcloudConfigDemo
spring.cloud.config.server.git.username=your git username
spring.cloud.config.server.git.password=your git password
```

这里将配置服务器端口定为8888，服务器名称设置为config-server。然后是配置仓库地址https://github.com/SpringcloudConfigDemo，及访问该仓库的用户名和密码。这个配置仓库是在GitHub上创建的一个Git仓库。

### 创建应用配置文件
application.properties文件如下：
```properties
# 默认日志级别
logging.level.org.springframework=INFO

# JPA相关配置
spring.jpa.open-in-view=true
spring.jpa.hibernate.ddl-auto=none
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.physical_naming_strategy=com.cd826dong.
springcloud.util.HibernatePhysicalNamingNamingStrategy
```

productservice.properties文件如下：
```properties
eureka.client.service-url.defaultZone=http://localhost:8260/eureka

# ProductService的日志级别配置
logging.level.org.springframework=INFO
logging.level.com.cd826dong=INFO

# 正式环境所使用MySQL数据库配置
spring.jpa.properties.hibernate.connection.charSet=UTF-8
spring.jpa.hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect
spring.datasource.driverClassName=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/spring-product?useUni
code=true&autoReconnect=true&rewriteBatchedStatements=TRUE&zeroDateTime
Behavior=convertToNull
spring.datasource.username=springdb
spring.datasource.password=2qH68gs!296m

# 其他属性值配置
foo=I'm default value
```

productservice-dev.properties文件如下：
```properties
# dev环境下日志级别
logging.level.com.cd826dong=DEBUG

# dev环境下将使用H2数据库
spring.jpa.database=H2
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.url=jdbc\:h2\:mem\:testdb;DB_CLOSE_DELAY=-1;
spring.datasource.username=sa
spring.datasource.password=

# 覆盖默认的属性值配置
foo=I'm development value
```

### 微服务读取配置
需要在项目中引入spring-cloud-starter-config的依赖

### 引入依赖
```xml
<dependencies>
    // 增加spring-cloud-starter-config的依赖
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    //……省略了其他的依赖
</dependencies>
```
然后，修改配置文件。对于Spring Boot应用来说有两个配置文件，一个是前面所使用的application.properties配置文件，另一个是bootstrap.properties。Spring Boot应用在启动时会根据bootstrap.properties配置文件创建一个引导上下文（Bootstrap Context）的文件，引导上下文将负责从外部加载配置属性并进行相应的解析，并作为Spring应用上下文（Application Context）的父上下文。此外，这两个上下文将共享一个从外部获取的Environment。默认情况，引导上下文的配置属性具有高优先级，不会被本地配置覆盖。由于引导上下文和应用上下文具有不同的含义，所以这里所要增加的配置属性信息就不是在application.properties配置文件中进行配置，而是在bootstrap.properties配置文件中进行配置。

bootstrap.properties配置文件内容如下：
```properties
# productservice微服务默认端口
server.port=2200

# productservice微服务的服务名称，配置服务器后续会根据该名称查找相应的配置文件
spring.application.name=productservice

# 配置服务器地址及所启用的profile
spring.cloud.config.profile=dev
spring.cloud.config.uri=http://localhost:8888/
```

application.properties配置文件的精简部分如下：
```properties
# 仅保留一个日志默认级别，其实也可以不配置该文件
logging.level.org.springframework=INFO
```

### @Value注解


## Spring配置加载顺序
对于一个标准的Spring Boot应用，可以通过多种方式进行配置。比如配置文件（properties或yml）、命令行参数，此外还有系统环境变量、JVM的参数等。下面我们来了解一下这些配置方式。

- 命令行参数：命令行参数使用--xxx=xxx格式在启动时传递，比如：--server.port=2300，就是将服务的端口设置为2300。这里的参考可以是Spring Boot框架的参数，也可以是我们自定义的参数或属性配置。
- 从java：comp/env加载的JNDI属性。
- Java系统属性：通过-Dxxx=xxx格式设置，只是优先级比上面的配置低。
- 操作系统环境变量：这里需要注意的一点是，有些操作系统配置时不支持使用点“.”进行分割，则需要将点替换成下画线。例如，server.port需要替换成server_port。
- RandomValuePropertySource：使用random.*属性进行配置，主要是在系统配置中需要使用随机数的地方使用，如foo.securityid=$ {random.value}。
- 特定应用的properties或yml配置文件：这些文件名称的命名格式为application-{profile}.properties或者yml，通过指定所要使用的profile来加载，例如前面使用的application-dev.properties配置文件。
- 应用配置文件application.properties或yml文件：为Spring Boot应用所默认加载的配置文件，可以通过上面的配置进行全部或部分配置属性的覆写。
- @Configuration、@PropertySource或@ConfigurationProperties所指向的配置文件，其中@ConfigurationProperties可以批量按照一定规范将配置注入到一个Bean中，但这些配置的优先级较低。

## 配置中心源码
Spring 提供的扩展机制中的`ApplicationContextInitializer`，该扩展是在上下文准备阶段（prepareContext），容器刷新之前做一些初始化工作，比如我们常用的配置中心 client 基本都是继承该初始化器，在容器刷新前将配置从远程拉到本地，然后封装成 PropertySource 放到 Environment 中供使用。

在 SpringCloud 场景下，SpringCloud 规范中提供了 PropertySourceBootstrapConfiguration 继承 ApplicationContextInitializer，另外还提供了个 PropertySourceLocator，二者配合完成配置中心的接入。
```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(PropertySourceBootstrapProperties.class)
public class PropertySourceBootstrapConfiguration implements
		ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {

	/**
	 * Bootstrap property source name.
	 */
	public static final String BOOTSTRAP_PROPERTY_SOURCE_NAME = BootstrapApplicationListener.BOOTSTRAP_PROPERTY_SOURCE_NAME
			+ "Properties";

	private static Log logger = LogFactory
			.getLog(PropertySourceBootstrapConfiguration.class);

	private int order = Ordered.HIGHEST_PRECEDENCE + 10;

	@Autowired(required = false)
	private List<PropertySourceLocator> propertySourceLocators = new ArrayList<>();

	@Override
	public int getOrder() {
		return this.order;
	}

	public void setPropertySourceLocators(
			Collection<PropertySourceLocator> propertySourceLocators) {
		this.propertySourceLocators = new ArrayList<>(propertySourceLocators);
	}

	@Override
	public void initialize(ConfigurableApplicationContext applicationContext) {
		List<PropertySource<?>> composite = new ArrayList<>();
		AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
		boolean empty = true;
		ConfigurableEnvironment environment = applicationContext.getEnvironment();
		for (PropertySourceLocator locator : this.propertySourceLocators) {
			Collection<PropertySource<?>> source = locator.locateCollection(environment);
			if (source == null || source.size() == 0) {
				continue;
			}
			List<PropertySource<?>> sourceList = new ArrayList<>();
			for (PropertySource<?> p : source) {
				if (p instanceof EnumerablePropertySource) {
					EnumerablePropertySource<?> enumerable = (EnumerablePropertySource<?>) p;
					sourceList.add(new BootstrapPropertySource<>(enumerable));
				}
				else {
					sourceList.add(new SimpleBootstrapPropertySource(p));
				}
			}
			logger.info("Located property source: " + sourceList);
			composite.addAll(sourceList);
			empty = false;
		}
		if (!empty) {
			MutablePropertySources propertySources = environment.getPropertySources();
			String logConfig = environment.resolvePlaceholders("${logging.config:}");
			LogFile logFile = LogFile.get(environment);
			for (PropertySource<?> p : environment.getPropertySources()) {
				if (p.getName().startsWith(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
					propertySources.remove(p.getName());
				}
			}
			insertPropertySources(propertySources, composite);
			reinitializeLoggingSystem(environment, logConfig, logFile);
			setLogLevels(applicationContext, environment);
			handleIncludedProfiles(environment);
		}
	}

	private void reinitializeLoggingSystem(ConfigurableEnvironment environment,
			String oldLogConfig, LogFile oldLogFile) {
		Map<String, Object> props = Binder.get(environment)
				.bind("logging", Bindable.mapOf(String.class, Object.class))
				.orElseGet(Collections::emptyMap);
		if (!props.isEmpty()) {
			String logConfig = environment.resolvePlaceholders("${logging.config:}");
			LogFile logFile = LogFile.get(environment);
			LoggingSystem system = LoggingSystem
					.get(LoggingSystem.class.getClassLoader());
			try {
				ResourceUtils.getURL(logConfig).openStream().close();
				// Three step initialization that accounts for the clean up of the logging
				// context before initialization. Spring Boot doesn't initialize a logging
				// system that hasn't had this sequence applied (since 1.4.1).
				system.cleanUp();
				system.beforeInitialize();
				system.initialize(new LoggingInitializationContext(environment),
						logConfig, logFile);
			}
			catch (Exception ex) {
				PropertySourceBootstrapConfiguration.logger
						.warn("Error opening logging config file " + logConfig, ex);
			}
		}
	}
```
在 PropertySourceBootstrapConfiguration 这个单例对象初始化的时候会将 Spring 容器中所有的 PropertySourceLocator 实现注入进来。然后在 initialize 方法中循环所有的 PropertySourceLocator 进行配置的获取，从这儿可以看出 SpringCloud 应用是支持我们引入多个配置中心实现的，获取到配置后调用 insertPropertySources 方法将所有的 PropertySource（封装的一个个配置文件）添加到 Spring 的环境变量 environment 中。

















