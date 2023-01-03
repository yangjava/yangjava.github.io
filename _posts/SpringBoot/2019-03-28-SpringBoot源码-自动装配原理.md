---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---

# 核心原理入门

springboot的有两大核心

- 依赖管理
- 自动装配

## **依赖管理**

### POM文件

#### 父项目

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.1.RELEASE</version>
    <relativePath/>
</parent>
```

其父项目是

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>2.0.1.RELEASE</version>
    <relativePath>../../spring-boot-dependencies</relativePath>
</parent>
```

该父项目是真正管理Spring Boot应用里面的所有依赖的版本：Spring Boot的版本仲裁中心，所以以后导入的依赖默认是不需要版本号。如下

```
  <properties>
    <activemq.version>5.16.3</activemq.version>
    <antlr2.version>2.7.7</antlr2.version>
    <appengine-sdk.version>1.9.91</appengine-sdk.version>
    <artemis.version>2.17.0</artemis.version>
    <aspectj.version>1.9.7</aspectj.version>
    <assertj.version>3.19.0</assertj.version>
    <atomikos.version>4.0.6</atomikos.version>
    <awaitility.version>4.0.3</awaitility.version>
    <build-helper-maven-plugin.version>3.2.0</build-helper-maven-plugin.version>
    <byte-buddy.version>1.10.22</byte-buddy.version>
    <caffeine.version>2.9.2</caffeine.version>
    <cassandra-driver.version>4.11.3</cassandra-driver.version>
    <classmate.version>1.5.1</classmate.version>
    <commons-codec.version>1.15</commons-codec.version>
    <commons-dbcp2.version>2.8.0</commons-dbcp2.version>
    <commons-lang3.version>3.12.0</commons-lang3.version>
    <commons-pool.version>1.6</commons-pool.version>
    <commons-pool2.version>2.9.0</commons-pool2.version>
    <couchbase-client.version>3.1.7</couchbase-client.version>
    <db2-jdbc.version>11.5.6.0</db2-jdbc.version>
    <dependency-management-plugin.version>1.0.11.RELEASE</dependency-management-plugin.version>
    <derby.version>10.14.2.0</derby.version>
    <dropwizard-metrics.version>4.1.25</dropwizard-metrics.version>
    <ehcache.version>2.10.9.2</ehcache.version>
    <ehcache3.version>3.9.5</ehcache3.version>
    <elasticsearch.version>7.12.1</elasticsearch.version>
    <embedded-mongo.version>3.0.0</embedded-mongo.version>
    <flyway.version>7.7.3</flyway.version>
    <freemarker.version>2.3.31</freemarker.version>
    <git-commit-id-plugin.version>4.0.5</git-commit-id-plugin.version>
    <glassfish-el.version>3.0.3</glassfish-el.version>
    <glassfish-jaxb.version>2.3.5</glassfish-jaxb.version>
    <groovy.version>3.0.8</groovy.version>
    <gson.version>2.8.7</gson.version>
    <h2.version>1.4.200</h2.version>
    <hamcrest.version>2.2</hamcrest.version>
    <hazelcast.version>4.1.5</hazelcast.version>
    <hazelcast-hibernate5.version>2.2.1</hazelcast-hibernate5.version>
    <hibernate.version>5.4.32.Final</hibernate.version>
    <hibernate-validator.version>6.2.0.Final</hibernate-validator.version>
    <hikaricp.version>4.0.3</hikaricp.version>
    <hsqldb.version>2.5.2</hsqldb.version>
    <htmlunit.version>2.49.1</htmlunit.version>
    <httpasyncclient.version>4.1.4</httpasyncclient.version>
    <httpclient.version>4.5.13</httpclient.version>
    <httpclient5.version>5.0.4</httpclient5.version>
    <httpcore.version>4.4.14</httpcore.version>
    <httpcore5.version>5.1.1</httpcore5.version>
    <infinispan.version>12.1.7.Final</infinispan.version>
    <influxdb-java.version>2.21</influxdb-java.version>
    <jackson-bom.version>2.12.4</jackson-bom.version>
    <jakarta-activation.version>1.2.2</jakarta-activation.version>
    <jakarta-annotation.version>1.3.5</jakarta-annotation.version>
    <jakarta-jms.version>2.0.3</jakarta-jms.version>
    <jakarta-json.version>1.1.6</jakarta-json.version>
    <jakarta-json-bind.version>1.0.2</jakarta-json-bind.version>
    <jakarta-mail.version>1.6.7</jakarta-mail.version>
    <jakarta-persistence.version>2.2.3</jakarta-persistence.version>
    <jakarta-servlet.version>4.0.4</jakarta-servlet.version>
    <jakarta-servlet-jsp-jstl.version>1.2.7</jakarta-servlet-jsp-jstl.version>
    <jakarta-transaction.version>1.3.3</jakarta-transaction.version>
    <jakarta-validation.version>2.0.2</jakarta-validation.version>
    <jakarta-websocket.version>1.1.2</jakarta-websocket.version>
    <jakarta-ws-rs.version>2.1.6</jakarta-ws-rs.version>
    <jakarta-xml-bind.version>2.3.3</jakarta-xml-bind.version>
    <jakarta-xml-soap.version>1.4.2</jakarta-xml-soap.version>
    <jakarta-xml-ws.version>2.3.3</jakarta-xml-ws.version>
    <janino.version>3.1.6</janino.version>
    <javax-activation.version>1.2.0</javax-activation.version>
    <javax-annotation.version>1.3.2</javax-annotation.version>
    <javax-cache.version>1.1.1</javax-cache.version>
    <javax-jaxb.version>2.3.1</javax-jaxb.version>
    <javax-jaxws.version>2.3.1</javax-jaxws.version>
    <javax-jms.version>2.0.1</javax-jms.version>
    <javax-json.version>1.1.4</javax-json.version>
    <javax-jsonb.version>1.0</javax-jsonb.version>
    <javax-mail.version>1.6.2</javax-mail.version>
    <javax-money.version>1.1</javax-money.version>
    <javax-persistence.version>2.2</javax-persistence.version>
    <javax-transaction.version>1.3</javax-transaction.version>
    <javax-validation.version>2.0.1.Final</javax-validation.version>
    <javax-websocket.version>1.1</javax-websocket.version>
    <jaxen.version>1.2.0</jaxen.version>
    <jaybird.version>4.0.3.java8</jaybird.version>
    <jboss-logging.version>3.4.2.Final</jboss-logging.version>
    <jboss-transaction-spi.version>7.6.1.Final</jboss-transaction-spi.version>
    <jdom2.version>2.0.6</jdom2.version>
    <jedis.version>3.6.3</jedis.version>
    <jersey.version>2.33</jersey.version>
    <jetty-el.version>9.0.48</jetty-el.version>
    <jetty-jsp.version>2.2.0.v201112011158</jetty-jsp.version>
    <jetty-reactive-httpclient.version>1.1.10</jetty-reactive-httpclient.version>
    <jetty.version>9.4.43.v20210629</jetty.version>
    <jmustache.version>1.15</jmustache.version>
    <johnzon.version>1.2.14</johnzon.version>
    <jolokia.version>1.6.2</jolokia.version>
    <jooq.version>3.14.13</jooq.version>
    <json-path.version>2.5.0</json-path.version>
    <json-smart.version>2.4.7</json-smart.version>
    <jsonassert.version>1.5.0</jsonassert.version>
    <jstl.version>1.2</jstl.version>
    <jtds.version>1.3.1</jtds.version>
    <junit.version>4.13.2</junit.version>
    <junit-jupiter.version>5.7.2</junit-jupiter.version>
    <kafka.version>2.7.1</kafka.version>
    <kotlin.version>1.5.21</kotlin.version>
    <kotlin-coroutines.version>1.5.1</kotlin-coroutines.version>
    <lettuce.version>6.1.4.RELEASE</lettuce.version>
    <liquibase.version>4.3.5</liquibase.version>
    <log4j2.version>2.14.1</log4j2.version>
    <logback.version>1.2.5</logback.version>
    <lombok.version>1.18.20</lombok.version>
    <mariadb.version>2.7.4</mariadb.version>
    .........
    
```

### 启动器(spring-boot-starter)

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

**spring-boot-starter** : spring boot场景启动器；帮助导入web模块正常运行所依赖的组件；

Spring Boot将所有的功能场景抽取出来，做成一个个的starter(启动器)，只需要在项目中引入这些starter，那么相关的场景的所有依赖都会导入进项目中。要用什么功能就导入什么场景的启动器。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
</dependency>
```

SpringBoot能够如此方便便捷，其实都是得益于这些“开箱即用”的依赖模块，那SpringBoot设计者约定这些“开箱即用”的依赖模块的命名都以`spring-boot-starter-`开始，并且这些模块都位于`org.springframework.boot`包或者命名空间下面。我们也可以模仿者来实现自己的自动配置依赖模块，也已`spring-boot-starter-`开头，是不是就很"正宗"呢？(虽然SpringBoot官方不建议我们这样做，以免跟官方提供的混淆，但是其实我们使用自己的groupId，这样命名应该不是啥问题)。

这些starter其实都有约定好的默认配置，但是它也允许我们调整这些默认配置，以便完成定制化的需求，我们可以改变默认配置的常见方式有以下几种：

- 命令行参数(Command Line Args)
- 系统环境变量(Environment Variables)
- 位于文件系统中的配置文件
- 位于classpath中的配置文件
- 固化到代码中的配置项

这几种方式从上到下优先级从高到低排列，高优先级的配置会覆盖优先级低的配置。还有就是不管位于文件系统还是classpath中的配置文件，SpringBoot应用默认的文件名称都是`application.properties`,可以放在当前项目的根目录下或者名称为config的子目录下。

SpringBoot其实提供了很多这样的模块，我们就挑几个我们常用的这样的模块来解析，其他的大家就举一反三。以达到在工作和开发中灵活运用这些spring-boot-starter模块的效果。

#### spring-boot-starter-logging(应用日志)

如果我们在maven依赖中添加了`spring-boot-starter-logging`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-logging</artifactId>
</dependency>
```

那也就意味着我们的SpringBoot应用自动使用logback作为日志框架，在启动的时候，由`org.springframework.boot.logging.LoggingApplicationListener`根据情况初始化并使用。默认情况下，SpringBoot已经给我们提供好了很多默认的日志配置，我们只需要将`spring-boot-starter-logging`作为依赖加入到你的SpringBoot应用就可以了，但是如果我们要对这些默认配置进行定制，可以有两种方式进行：

- 遵守logback的约定，在classpath中使用定制化的logback.xml配置文件。

- 在文件系统中任意一个地方提供自己的logback.xml配置文件，然后通过如下配置来`application.properties`中指定我们日志系统配置文件位置：

  ```properties
  logging.config=/{your config file location}}/logback.xml
  ```

如果我们已经习惯了log4j或log4j2,那我们只需要把`spring-boot-starter-logging`换成如下的starter就好。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j</artifactId>
</dependency>
或
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

#### spring-boot-starter-web(快速构建web应用)

现如今，我们在工作中大部分实际用的还是SpringMVC开发的web应用，SpringBoot当然贴心的为我们开发了一个web项目模块，让我们更加方便的开发web应用。maven依赖如下：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

这样我们就可以得到一个可以直接执行的Web应用，然后我们运行`mvn spring-boot:run`，就能直接启动一个基于嵌入式tomcat容器的Web应用了，然后就可以像这篇文章中定义controller来供用户访问了。但是呢，这简单的表象之下，其实却隐藏着很多约定，我们要把这些潜规则了解清楚才能更好地应用`spring-boot-starter-web`。

##### 项目结构的“潜规则”

传统的Java Web项目中，我们的静态文件以及页面模板都是放在`src/main/webapp`目录下，但是在SpringBoot应用中，这些文件被统一放在`src/main/resources`相应的子目录下：

- `src/main/resources/static`目录用于存放各种静态资源，如：js、css、image等。
- `src/main/resources/template`目录用于存放模板文件。

> 细心地我们会发现SpringBoot的web应用已经变成了jar包而再是war包，如果我们还是希望以war包的形式发布也是可以的。

##### SpringMVC框架层面的约定及定制

`spring-boot-starter-web`默认将为我们自动配置如下一些SpringMVC必要的组件：

- ViewResolver，如：`ContentNegotiatingViewResolver`和`BeanNameViewResolver`。
- Converter，如：`GenericConverter`和`Formatter`等bean被注册到IoC容器。
- 默认添加一系列`HttpMessageConverter`用于支持对Web请求和相应的类型转换。
- 自动配置和注册`MessageCodesResolver`。
- 其他必要组件…

##### 嵌入式Web容器的约定和定制

我们知道`spring-boot-starter-web`默认把嵌入式tomcat作为web容器来对外提供HTTP服务，默认使用8080端口对外监听和提供服务。这里我们可能会有两个疑问：

- 我们不想使用默认的嵌入式tomcat容器怎么办？

  很简单，我们只需要引入`spring-boot-starter-jetty`或`spring-boot-starter-undertow`依赖就能替代默认嵌入式tomcat容器了。

- 我们想要把启动后提供服务的端口改掉怎么办？

  我们可以通过在配置文件中修改启动端口就可以了，如：

  ```properties
  server.port=9000
  ```

其实，`spring-boot-starter-web`提供了很多以`server.`作为前缀的配置以用来修改嵌入式容器的配置，如：

```properties
server.port
server.address
server.ssl.*
server.tomcat.*
```

那若这些还满足不了你，SpringBoot甚至都允许我们直接对嵌入式Web容器实例进行定制化，我们通过向IoC容器中注册一个`EmbeddedServletContainerCustomizer`类型的组件来实现：

```java
package com.springbootdemo;

import org.springframework.boot.context.embedded.ConfigurableEmbeddedServletContainer;
import org.springframework.boot.context.embedded.EmbeddedServletContainerCustomizer;

/**
 * @description: 自定义内嵌容器配置
 */
public class DemoEmbeddedTomcatCustomizer implements EmbeddedServletContainerCustomizer {
    @Override
    public void customize(ConfigurableEmbeddedServletContainer container) {
        container.setPort(9111);
        container.setContextPath("/demo");
        // ...
    }
}
```

如果还要再深入的定制，那就需要实现对应内嵌容器的Factory并注册到IoC容器：

- TomcatEmbeddedServletContainerFactory
- JettyEmbeddedServletContainerFactory
- UndertowEmbeddedServletContainerFactory

但是，我们几乎没有可能需要这样的定制化，也不建议这样的定制化，使用SpringBoot默认的`spring-boot-starter-web`提供的配置项列表已经很简单、很完整了。

#### spring-boot-starter-jdbc(数据访问)

我们知道，现实中大多数的Java应用都需要访问数据库，那SpringBoot肯定不会放过这个组件，它会很贴心的为我们自动配置好相应的数据访问工具。我们只需要在`pom.xml`中添加以下依赖就好了：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

这样，在我们没有配置任何DataSource的情况下，SpringBoot会默认为我们自动配置一个基于嵌入式数据的DataSource，这种自动配置适合于测试场景，生产环境不适合。大多数情况下，我们都会自己配置DataSource实例，或通过自动配置模块提供的配置参数对DataSource实例配置自定义的参数。

若我们的SpringBoot应用只依赖一个数据库，那我们直接使用自动配置模块提供的配置参数最方便快捷：

```properties
spring.datasource.url=jdbc:mysql://{db host}:{db port}/{db name}
spring.datasource.username={db user name}
spring.datasource.password={db password}
```

有的小伙伴说了：那我自己配置一个DataSource行不行？答案是当然可以，SpringBoot会很智能的优先选择使用我们自己配置的这个DataSource，但是感觉多此一举！你要知道，SpringBoot除了自动帮我们配置DataSource以外，还自动帮我们配置了相应的`JdbcTemplate`以及`DataSourceTransactionManager`等相关的组件，我们只需要在需要使用的地方直接使用`@Autowired`注解引用就好了。

那SpringBoot是不是一直贴心呢？很明显不是的，如果我们的单个项目需要依赖和访问多个数据库，这个时候就不行了，就算是我们在ApplicationContext中配置了多个DataSource实例来访问多个数据库：

```java
@Bean
public DataSource dataSource1() throws Throwable {
    DruidDataSource ds = new DruidDataSource();
    ds.setUrl(...);
    ds.setUsername(...);
    ds.setPassword(...);
    // set other db setting
    return ds;
}
@Bean
public DataSource dataSource2() throws Throwable {
    DruidDataSource ds = new DruidDataSource();
    ds.setUrl(...);
    ds.setUsername(...);
    ds.setPassword(...);
    // set other db setting
    return ds;
}
```

启动项目时，你就会发现如下的异常:

```java
No qualifying bean of type [javax.sql.DataSource] is defined: expected single matching bean but found 2...
```

那怎么解决这个问题呢？有两种方式：

- 在SpringBoot的启动类上“动手脚”

  ```java
  @SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    DataSourceTransactionManagerAutoConfiguration.class
  })
  public class DemoSpringBootApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoSpringBootApplication.class, args);
    }
  }
  ```

  这也就是说我们需要排除掉SpringBoot默认的DataSource的相关的自动配置。

- 使用`@primary`注解

  那我们既要配置两个数据源，又要使用SpringBoot默认的DataSource，这时我们就可以为我们配置的两个DataSource中的任意一个使用`@primary`注解就可以了。

  ```java
  @Bean
  @Primary
  public DataSource dataSource1() throws Throwable {
    DruidDataSource ds = new DruidDataSource();
    ds.setUrl(...);
    ds.setUsername(...);
    ds.setPassword(...);
    // set other db setting
    return ds;
  }
  @Bean
  public DataSource dataSource2() throws Throwable {
    DruidDataSource ds = new DruidDataSource();
    ds.setUrl(...);
    ds.setUsername(...);
    ds.setPassword(...);
    // set other db setting
    return ds;
  }
  ```

  除此之外，SpringBoot还提供了很多其他数据源访问相关的自动配置模块，如：`spring-boot-starter-jpa`、`spring-boot-starter-mongodb`等。

#### 自定义starter

首先定义一个配置类模块：

```
@Configuration
@ConditionalOnProperty(name = "enabled.autoConfituration", matchIfMissing = true)
public class MyAutoConfiguration {

    static {
        System.out.println("myAutoConfiguration init...");
    }

    @Bean
    public SimpleBean simpleBean(){
        return new SimpleBean();
    }

}
```

然后定义一个starter模块，里面无需任何代码，pom也无需任何依赖，只需在META-INF下面建一个 `spring.factories`文件，添加如下配置：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.springdemo.MyAutoConfiguration
```

最后只需在启动类项目的pom中引入我们的 starter 模块即可。

**springBoot为我们提供的配置类有多个，但是我们不可能会全部引入。按条件注解 @Conditional或者@ConditionalOnProperty等相关注解进行判断，决定是否需要装配。**

我们自定义的配置类也是以相同的逻辑进行装配，我们指定了以下注解：

```
@ConditionalOnProperty(name = "enabled.autoConfituration", matchIfMissing = true)
```

默认为 true，所以自定义的starter成功执行。

## 自动配置

springboot是基于spring的新型的轻量级框架，最厉害的地方当属**自动配置**。那我们就可以根据启动流程和相关原理来看看，如何实现传奇的自动配置。

### springboot的启动类入口

springboot有自己独立的启动类（独立程序）

```
@SpringBootApplication
public class HelloWorldApplication {

    public static void main(String[] args) {
        SpringApplication.run(HelloWorldApplication.class, args);
    }
    
}
```

**@SpringBootApplication**

- Spring Boot应用标注在某个类上，说明这个类是SpringBoot的主配置类，SpringBoot就应该运行这个类的main方法来启动SpringBoot应用。

### SpringBootApplication注解

```
@Target(ElementType.TYPE) // 注解的适用范围，其中TYPE用于描述类、接口（包括包注解类型）或enum声明
@Retention(RetentionPolicy.RUNTIME) // 注解的生命周期，保留到class文件中（三个生命周期）
@Documented // 表明这个注解应该被javadoc记录
@Inherited // 子类可以继承该注解
@SpringBootConfiguration // 继承了Configuration，表示当前是注解类
@EnableAutoConfiguration // 开启springboot的注解功能，springboot的四大神器之一，其借助@import的帮助
@ComponentScan(excludeFilters = { // 扫描路径设置
@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
...
}　
```

在其中比较重要的有三个注解，分别是：

- @SpringBootConfiguration // 继承了Configuration，表示当前是注解类
- @EnableAutoConfiguration // 开启springboot的注解功能，springboot的四大神器之一，其借助@import的帮助
- @ComponentScan(excludeFilters = { // 扫描路径设置（具体使用待确认）

#### @SpringBootConfiguration注解　　

**@SpringBootConfiguration**

- Spring Boot的配置类
- 标注在某个类上，表示这是一个Spring Boot的配置类

注解定义如下：

```
@Configuration
public @interface SpringBootConfiguration {}
```

其实就是一个**Configuration配置类**，意思是HelloWorldMainApplication最终会被注册到Spring容器中

**配置bean方式的不同**　

- xml配置文件的形式配置bean

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd"
default-lazy-init="true">
<!--bean定义-->
</beans>
```

- java configuration的配置形式配置bean

```
@Configuration
public class MockConfiguration{
    //bean定义
}
```

**注入bean方式的不同**

- xml配置文件的形式注入bean

```
<bean id="mockService" class="..MockServiceImpl">
...
</bean>
```

- java configuration的配置形式注入bean

```
@Configuration
public class MockConfiguration{
    @Bean
    public MockService mockService(){
        return new MockServiceImpl();
    }
}
```

任何一个标注了@Bean的方法，其返回值将作为一个bean定义注册到Spring的IoC容器，方法名将默认成该bean定义的id。

**表达bean之间依赖关系的不同**

- xml配置文件的形式表达依赖关系

```
<bean id="mockService" class="..MockServiceImpl">
　　<propery name ="dependencyService" ref="dependencyService" />
</bean>
<bean id="dependencyService" class="DependencyServiceImpl"></bean>
```

- java configuration配置的形式表达**依赖关系（重点）**

**如果一个bean A的定义依赖其他bean B,则直接调用对应的JavaConfig类中依赖bean B的创建方法就可以了。**

```
@Configuration
public class MockConfiguration{
　　@Bean
　　public MockService mockService(){
    　　return new MockServiceImpl(dependencyService());
　　}
　　@Bean
　　public DependencyService dependencyService(){
    　　return new DependencyServiceImpl();
　　}
}
```

#### @ComponentScan注解

ComponentScan注解作用

- 对应xml配置中的元素
- **（重点）**ComponentScan的功能其实就是自动扫描并加载符合条件的组件（比如@Component和@Repository等）或者bean定义
- 将这些bean定义加载到**IoC**容器中

我们可以通过basePackages等属性来**细粒度**的定制@ComponentScan自动扫描的范围，如果不指定，则**默认**Spring框架实现会从声明@ComponentScan所在类的package进行扫描。

注：所以SpringBoot的启动类最好是放在root ``package``下，因为默认不指定basePackages

#### **@EnableAutoConfiguration**注解

- 开启自动配置功能
- 以前使用Spring需要配置的信息，Spring Boot帮助自动配置；
- **@EnableAutoConfiguration**通知SpringBoot开启自动配置功能，这样自动配置才能生效。

此注解顾名思义是可以自动配置，所以应该是springboot中最为重要的注解。在spring框架中就提供了各种以@Enable开头的注解，例如： @EnableScheduling、@EnableCaching、@EnableMBeanExport等； @EnableAutoConfiguration的理念和做事方式其实一脉相承简单概括一下就是，借助@Import的支持，收集和注册特定场景相关的bean定义。　　

- @EnableScheduling是通过@Import将Spring调度框架相关的bean定义都加载到IoC容器【定时任务、时间调度任务】
- @EnableMBeanExport是通过@Import将JMX相关的bean定义加载到IoC容器【监控JVM运行时状态】
- @EnableAutoConfiguration也是借助@Import的帮助，将所有符合自动配置条件的bean定义加载到IoC容器。

@EnableAutoConfiguration作为一个复合Annotation,其自身定义关键信息如下：

```
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage【重点注解】
@Import(AutoConfigurationImportSelector.class)【重点注解】
public @interface EnableAutoConfiguration {
...
}
```

### EnableAutoConfiguration注解

最重要的两个注解

- @AutoConfigurationPackage
- @Import(AutoConfigurationImportSelector.class)

#### AutoConfigurationPackage注解

- 自动配置包注解

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
 
}
```

**@Import(AutoConfigurationPackages.Registrar.class)**：默认将主配置类(**@SpringBootApplication**)所在的包及其子包里面的所有组件扫描到Spring容器中。如下

```
@Order(Ordered.HIGHEST_PRECEDENCE)
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
            BeanDefinitionRegistry registry) {
          //默认将会扫描@SpringBootApplication标注的主配置类所在的包及其子包下所有组件
        register(registry, new PackageImport(metadata).getPackageName());
    }

    @Override
    public Set<Object> determineImports(AnnotationMetadata metadata) {
        return Collections.<Object>singleton(new PackageImport(metadata));
    }
}
```

- 注册当前启动类的根package；
- 注册org.springframework.boot.autoconfigure.AutoConfigurationPackages的BeanDefinition。

#### Import(AutoConfigurationImportSelector.class)注解

**AutoConfigurationImportSelector**： 导入哪些组件的选择器，将所有需要导入的组件以全类名的方式返回，这些组件就会被添加到容器中。

**AutoConfigurationImportSelector 实现**了 DeferredImportSelector 从 ImportSelector继承的方法：**selectImports**。

```
@Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        }
        AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
                .loadMetadata(this.beanClassLoader);
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        List<String> configurations = getCandidateConfigurations(annotationMetadata,
                attributes);
        configurations = removeDuplicates(configurations);
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);
        configurations = filter(configurations, autoConfigurationMetadata);
        fireAutoConfigurationImportEvents(configurations, exclusions);
        return StringUtils.toStringArray(configurations);
    }
```

我们主要看List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes)会给容器中注入众多的自动配置类（xxxAutoConfiguration）。

```
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
            AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
        getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
    //...
    return configurations;
}

protected Class<?> getSpringFactoriesLoaderFactoryClass() {
    return EnableAutoConfiguration.class;
}

public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
    String factoryClassName = factoryClass.getName();
    try {
        //从类路径的META-INF/spring.factories中加载所有默认的自动配置类
        Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                                 ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        List<String> result = new ArrayList<String>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
            //获取EnableAutoConfiguration指定的所有值,也就是EnableAutoConfiguration.class的值
            String factoryClassNames = properties.getProperty(factoryClassName);
            result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
        }
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() + "] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```

就是给容器中导入这个场景需要的所有组件，并配置好这些组件。其实是去加载各个组件jar下的  public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"外部文件。

SpringBoot启动的时候从类路径下的 **META-INF/spring.factories**中获取EnableAutoConfiguration指定的值，并将这些值作为自动配置类导入到容器中，自动配置类就会生效，最后完成自动配置工作。EnableAutoConfiguration默认在spring-boot-autoconfigure这个包中，如下图

```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.boot.autoconfigure.integration.IntegrationPropertiesEnvironmentPostProcessor

... ...
```

该方法在springboot启动流程——bean实例化前被执行，返回要实例化的类信息列表；

如果获取到类信息，spring可以通过类加载器将类加载到jvm中，现在我们已经通过spring-boot的starter依赖方式依赖了我们需要的组件，那么这些组件的类信息在select方法中就可以被获取到。

```
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
 List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
 Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
 return configurations;
 }
```

其返回一个自动配置类的类名列表，方法调用了loadFactoryNames方法，查看该方法

```
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
 String factoryClassName = factoryClass.getName();
 return (List)loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
 }
```

自动配置器会跟根据传入的factoryClass.getName()到项目系统路径下所有的spring.factories文件中找到相应的key，从而加载里面的类。

**（重点）**其中，最关键的要属@Import(AutoConfigurationImportSelector.class)，借助AutoConfigurationImportSelector，@EnableAutoConfiguration可以帮助SpringBoot应用将所有符合条件(spring.factories)的**bean定义**（如Java Config@Configuration配置）都加载到当前SpringBoot创建并使用的IoC容器。

### SpringFactoriesLoader详解

借助于Spring框架**原有**的一个工具类：**SpringFactoriesLoader**的支持，@EnableAutoConfiguration可以智能的自动配置功效才得以大功告成！

SpringFactoriesLoader属于Spring框架私有的一种扩展方案，其主要功能就是从**指定的配置文件META-INF/spring.factories**加载配置,**加载工厂类**。

SpringFactoriesLoader为Spring工厂加载器，该对象提供了loadFactoryNames方法，入参为factoryClass和classLoader即需要传入**工厂类**名称和对应的类加载器，方法会根据指定的classLoader，加载该类加器搜索路径下的指定文件，即spring.factories文件；

传入的工厂类为接口，而文件中对应的类则是接口的实现类，或最终作为实现类。

```
public abstract class SpringFactoriesLoader {
//...
　　public static <T> List<T> loadFactories(Class<T> factoryClass, ClassLoader classLoader) {
　　　　...
　　}
   
   
　　public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
　　　　....
　　}
}
```

配合@EnableAutoConfiguration使用的话，它更多是提供一种配置查找的功能支持，即根据@EnableAutoConfiguration的完整类名org.springframework.boot.autoconfigure.EnableAutoConfiguration作为查找的Key,获取对应的一组@Configuration类　　

上图就是从SpringBoot的autoconfigure依赖包中的META-INF/spring.factories配置文件中摘录的一段内容，可以很好地说明问题。

（重点）所以，@EnableAutoConfiguration自动配置的魔法其实就变成了：

从classpath中搜寻所有的META-INF/spring.factories配置文件，并将其中org.springframework.boot.autoconfigure.EnableAutoConfiguration对应的**配置项**通过**反射（Java Refletion）**实例化为对应的标注了**@Configuration**的JavaConfig形式的IoC容器配置类，然后汇总为一个并加载到IoC容器。

### 自动配置奥秘

`@EnableAutoConfiguration`借助SpringFactoriesLoader可以将标注了`@Configuration`这个注解的JavaConfig类一并汇总并加载到最终的ApplicationContext，这么说只是很简单的解释，其实基于`@EnableAutoConfiguration`的自动配置功能拥有非常强大的调控能力。比如我们可以通过配合基于条件的配置能力或定制化加载顺序，对自动化配置进行更加细粒度的调整和控制。

#### 基于条件的自动配置

这个基于条件的自动配置来源于Spring框架中的"基于条件的配置"特性。在Spring框架中，我们可以使用`@Conditional`这个注解配合`@Configuration`或`@Bean`等注解来干预一个配置或bean定义是否能够生效，它最终实现的效果或者语义类如下伪代码：

```
if (复合@Conditional规定的条件) {
    加载当前配置（Enable Current Configuration）或者注册当前bean定义;
}
```

要实现基于条件的配置，我们需要通过`@Conditional`注解指定自己Condition实现类就可以了(可以应用于类型Type的注解或者方法Method的注解)

```
@Conditional({DemoCondition1.class, DemoCondition2.class})
```

最重要的是，`@Conditional`注解可以作为一个Meta Annotaion用来标注其他注解实现类，从而构建各种复合注解，比如SpringBoot的autoconfigre模块就基于这一优良的革命传统，实现了一批这样的注解(在org.springframework.boot.autoconfigure.condition包下):

- @ConditionalOnClass
- @ConditionalOnBean
- @CondtionalOnMissingClass
- @CondtionalOnMissingBean
- @CondtionalOnProperty
- ……

有了这些复合Annotation的配合，我们就可以结合@EnableAutoConfiguration实现基于条件的自动配置了。其实说白了，SpringBoot能够如此的盛行，很重要的一部分就是它默认提供了一系列自动配置的依赖模块，而这些依赖模块都是基于以上的@Conditional复合注解实现的，这也就说明这些所有的依赖模块都是按需加载的，只有复合某些特定的条件，这些依赖模块才会生效，这也解释了为什么自动配置是“智能”的。

#### 定制化自动配置的顺序

在实现自动配置的过程中，我们除了可以提供基于条件的配置之外，我们还能对当前要提供的配置或组件的加载顺序进行个性化调整，以便让这些配置或者组件之间的依赖分析和组装能够顺利完成。

最经典的是我们可以通过使用`@org.springframework.boot.autoconfigure.AutoConfigureBefore`或者`@org.springframework.boot.autoconfigure.AutoConfigureAfter`让当前配置或者组件在某个其他组件之前或者之后进行配置。例如，假如我们希望某些JMX操作相关的bean定义在MBeanServer配置完成以后在进行配置，那我们就可以提供如下配置：

```
@Configuration
@AutoConfigureAfter(JmxAutoConfiguration.class)
public class AfterMBeanServerReadyConfiguration {
    @Autowired
    MBeanServer mBeanServer;

    // 通过@Bean添加其他必要的bean定义
}
```