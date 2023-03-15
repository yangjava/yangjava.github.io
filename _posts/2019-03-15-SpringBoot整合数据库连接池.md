---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot整合数据库连接池
在JDBC的基础上,我们发现来回的获得数据库连接返回数据库连接给数据库会大大降低数据库的执行效率。所以我们可以采用使用连接池的方式来放置连接、获取连接。当我们需要数据库连接的时候，我们不在向数据库获取，而是从连接池中获得数据库的连接，用完连接之后也不再返回给数据库，而是直接返回给连接池。这样数据库的效率的到很大的提升。

## 连接池介绍

在jdbc的基础上,我们发现来回的获得数据库连接返回数据库连接给数据库会大大降低数据库的执行效率。所以我们可以采用使用连接池的方式来放置连接、获取连接。当我们需要数据库连接的时候，我们不在向数据库获取，而是从连接池中获得数据库的连接，用完连接之后也不再返回给数据库，而是直接返回给连接池。这样数据库的效率的到很大的提升。

常用的数据库连接池有两种：DCBP、C3P0、HikariCP和Druid等。

### 什么是数据库连接池？
数据库连接池负责分配、管理和释放数据库连接，它允许应用程序重复使用一个现有的数据库连接，而不是再重新建立一个；释放空闲时间超过最大空闲时间的数据库连接来避免因为没有释放数据库连接而引起的数据库连接遗漏。这项技术能明显提高对数据库操作的性能。

### 数据库连接池基本原理？
连接池基本的思想是在系统初始化的时候，将数据库连接作为对象存储在内存中，当用户需要访问数据库时，并非建立一个新的连接，而是从连接池中取出一个已建立的空闲连接对象。使用完毕后，用户也并非将连接关闭，而是将连接放回连接池中，以供下一个请求访问使用。而连接的建立、断开都由连接池自身来管理。同时，还可以通过设置连接池的参数来控制连接池中的初始连接数、连接的上下限数以及每个连接的最大使用次数、最大空闲时间等等。也可以通过其自身的管理机制来监视数据库连接的数量、使用情况等。
数据库连接池的最小连接数和最大连接数的设置要考虑到下列几个因素：
- 最小连接数是连接池一直保持的数据库连接，所以如果应用程序对数据库连接的使用量不大，将会有大量的数据库连接资源被浪费。
- 最大连接数是连接池能申请的最大连接数，如果数据库连接请求超过此数，后面的数据库连接请求将被加入到等待队列中，这会影响之后的数据库操作。
- 最小连接数与最大连接数差距最小连接数与最大连接数相差太大，那么最先的连接请求将会获利，之后超过最小连接数量的连接请求等价于建立一个新的数据库连接。不过，这些大于最小连接数的数据库连接在使用完不会马上被释放，它将被放到连接池中等待重复使用或是空闲超时后被释放。

### 常见的数据库连接池？
开源的数据库连接池众多，这里我们需要了解曾经常用的开源数据库连接池及其被淘汰原因，并了解目前最常用的数据库连接池。
#### ①C3P0
开源的JDBC连接池，实现了数据源和JNDI绑定，支持JDBC3规范和JDBC2的标准扩展。目前使用它的开源项目有Hibernate、Spring等。单线程，性能较差，适用于小型系统，代码600KB左右。
是一个开放源代码的JDBC连接池，它在lib目录中与Hibernate一起发布，包括了实现jdbc3和jdbc2扩展规范说明的Connection 和Statement 池的DataSources 对象。由于一度是Hibernate内置的数据库连接池而被开发者熟知，但是由于性能和复杂度，官方已经放弃维护。
#### ②DBCP
全称(Database Connection Pool)，由Apache开发的一个Java数据库连接池项目， Jakarta commons-pool对象池机制，Tomcat使用的连接池组件就是DBCP。单独使用dbcp需要3个包：common-dbcp.jar,common-pool.jar,common-collections.jar，预先将数据库连接放在内存中，应用程序需要建立数据库连接时直接到连接池中申请一个就行，用完再放回。单线程，并发量低，性能不好，适用于小型系统。
DBCP（被淘汰：依赖Commons-Pool，性能差） DBCP（DataBase Connection Pool）属于Apache顶级项目Commons中的核心子项目。但DBCP并不是独立实现连接池功能的，它内部依赖于Commons-Pool项目，连接池最核心的“池”，就是由Commons-Pool组件提供的，因此，DBCP的性能实际上就是Pool的性能。
#### ③Tomcat Jdbc Pool
Tomcat在7.0以前都是使用common-dbcp做为连接池组件，但是dbcp是单线程，为保证线程安全会锁整个连接池，性能较差，dbcp有超过60个类，也相对复杂。Tomcat从7.0开始引入了新增连接池模块叫做Tomcat jdbc pool，基于Tomcat JULI，使用Tomcat日志框架，完全兼容dbcp，通过异步方式获取连接，支持高并发应用环境，超级简单核心文件只有8个，支持JMX，支持XA Connection。

#### ④BoneCP
官方说法BoneCP是一个高效、免费、开源的Java数据库连接池实现库。设计初衷就是为了提高数据库连接池性能，根据某些测试数据显示，BoneCP的速度是最快的，要比当时第二快速的连接池快25倍左右，完美集成到一些持久化产品如Hibernate和DataNucleus中。BoneCP特色：高度可扩展，快速；连接状态切换的回调机制；允许直接访问连接；自动化重置能力；JMX支持；懒加载能力；支持XML和属性文件配置方式；较好的Java代码组织，100%单元测试分支代码覆盖率；代码40KB左右。

#### ⑤Druid(德鲁伊)–推荐使用
Druid是Java语言中最好的数据库连接池，Druid能够提供强大的监控和扩展功能，是一个可用于大数据实时查询和分析的高容错、高性能的开源分布式系统，尤其是当发生代码部署、机器故障以及其他产品系统遇到宕机等情况时，Druid仍能够保持100%正常运行。主要特色：为分析监控设计；快速的交互式查询；高可用；可扩展；Druid是一个开源项目，源码托管在github上。

#### ⑥Hikari
HiKariCP是数据库连接池的一个后起之秀，号称性能最好，可以完美地PK掉其他连接池，是一个高性能的JDBC连接池，基于BoneCP做了不少的改进和优化。并且在springboot2.0之后，采用的默认数据库连接池就是Hikari。

## SpringBoot中的连接池及使用

### SpringBoot默认连接池(Hikari)使用
如下是常用的HikariCP的使用配置
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test?useSSL=false&autoReconnect=true&characterEncoding=utf8
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: root
    password: root
    # 指定为HikariDataSource
    type: com.zaxxer.hikari.HikariDataSource
    # hikari连接池配置
    hikari:
      #连接池名
      pool-name: HikariCP
      #最小空闲连接数
      minimum-idle: 5
      # 空闲连接存活最大时间，默认10分钟
      idle-timeout: 600000
      # 连接池最大连接数，默认是10
      maximum-pool-size: 10
      # 此属性控制从池返回的连接的默认自动提交行为,默认值：true
      auto-commit: true
      # 此属性控制池中连接的最长生命周期，值0表示无限生命周期，默认30分钟
      max-lifetime: 1800000
      # 数据库连接超时时间,默认30秒
      connection-timeout: 30000
      # 连接测试query
      connection-test-query: SELECT 1
```
配置文件中设置
```properties
#数据库配置
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/test?serverTimezone=UTC&useUnicode=true&characterEncoding=utf8
spring.datasource.username=root
spring.datasource.password=root

# 数据库连接池配置
#最小空闲连接，默认值10，小于0或大于maximum-pool-size，都会重置为maximum-pool-size
spring.datasource.hikari.minimum-idle=10
#最大连接数，小于等于0会被重置为默认值10；大于零小于1会被重置为minimum-idle的值
spring.datasource.hikari.maximum-pool-size=20
#空闲连接超时时间，默认值600000（10分钟），大于等于max-lifetime且max-lifetime>0，会被重置为0；不等于0且小于10秒，会被重置为10秒
spring.datasource.hikari.idle-timeout=500000
#连接最大存活时间，不等于0且小于30秒，会被重置为默认值30分钟.设置应该比mysql设置的超时时间短
spring.datasource.hikari.max-lifetime=540000
#连接超时时间：毫秒，小于250毫秒，否则被重置为默认值30秒
spring.datasource.hikari.connection-timeout=60000
```
Hikari连接池的使用是很简单的，因为是默认的连接池，因此我们也不需要做过多的配置，拿来既可以使用。

优势
- 字节码精简：优化代码，直到编译后的字节码最少，这样，CPU缓存可以加载更多的程序代码；
- 优化代理和拦截器：减少代码，例如HikariCP的Statement proxy只有100行代码，只有BoneCP的十分之一；
- 自定义数组类型（FastStatementList）代替ArrayList：避免每次get()调用都要进行range check，避免调用remove()时的从头到尾的扫描；
- 自定义集合类型（ConcurrentBag）：提高并发读写的效率；
- 其他针对BoneCP缺陷的优化，比如对于耗时超过一个CPU时间片的方法调用的研究（但没说具体怎么优化）。


## Hikari最佳实践
HikariCP连接池配置详解
对应的配置属性、解释、注意点都来源于官方：[https://github.com/brettwooldridge/HikariCP]

注意点：
1）所有时间单位的属性都是毫秒。
2）属性分必须；频繁使用；非频繁使用三个级别，重要度由高到低

必须属性(Essentials)
1. dataSourceClassName or jdbcUrl
   配置由 jdbc驱动提供的对应数据库 DataSource 类名，常见的如下表：

注意点：
⚠ 1）Note: Spring Boot auto-configuration users, you need to use jdbcUrl-based configuration. （使用spring boot自动化配置数据源的需要配置jdbcUrl，不能只配置dataSourceClassName）
⚠ 2）The MySQL DataSource is known to be broken with respect to network timeout support. Use jdbcUrl configuration instead.（MySQL数据库数据源在面对网络超时时有故障，所以也需要用jdbcUrl替代，所有上表中MySQL的行注释掉了）

jdbcUrl用来配置jdbc连接串，很多人习惯配置jdbcUrl 。注意：一些老的驱动还要同时配置 driverClassName，否则会报错（推荐不用配置它，可以先不配置尝试运行看报不报错）

dataSourceClassName 和 jdbcUrl缺省都是空。

最佳实践：鉴于使用spring-boot自动装配使用HikariCP的只配置jdbcUrl即可，driverClassName根据测试情况可选配置

2. username
   很容易理解，就是当前用户名，缺省是空。 必须配置

3. password
   指用户名对应的密码，缺省是空。必须配置

频繁使用的属性(Frequently used)
1. autoCommit
   这个属性控制连接返回池中前auto-commit是否自动进行。缺省：true。
   最佳实践：不需要配置，保持缺省即可。

2. connectionTimeout
   控制一个客户端等待从池中获取连接的最大时间。超过该时间还获取不到连接则抛出SQLException，最低可设置的时间是250ms，缺省：30000ms

最佳实践：非特殊业务场景保持缺省30s连接超时即可。

3. idleTimeout
   控制空闲连接的在池中最大的空闲时间。注意：这个配置只有当配置了minimumIdle属性(表示允许的最小空闲连接数)，且比maximumPoolSize（表示池中允许的最大连接数）更小时才生效。 这个比较好理解：当前池中有空闲连接且比允许的最小空闲连接多时，根据空闲超时时间来逐出。

当配置为0时表示空闲连接永远不逐出。
缺省：600000ms
最小生效值：10000ms

连接池会定时轮询检测哪些连接是空闲，并且空闲达到了idleTimeout配置的时间，但轮询间隔有时间差，一个空闲连接最大可空闲idleTimeout + 30s会逐出，平均是：idleTimeout + 15s。

最佳实践：不设置该属性和minimumIdle属性，保持连接池固定的连接

4. keepaliveTime
   用于跟数据库保持心跳连接，防止底层网络基础设施超时断开，定期验证连接的有效性。这个参数需要设置的比maxLifetime(连接最大生存时间，下文会介绍)小。只会对池中空闲连接发生keeplive，具体执行细节是：
   从池中取出一个连接，执行“ping”，成功后再放回。
   ping:
   如果当前支持JDBC4 , 通过调用isValid()方法实现
   否则执行connectionTestQuery属性(下文会介绍)指定的sql

这个参数配置后连接池会定期检测连接有效性，检测过程中连接短暂从池中移出，不过不用担心，耗时非常短(几ms甚至更短)

缺省：0， 即不开启
最小可配置：30000ms ，即30秒

最佳实践： 需要设置， 可设置为60000, 即1分钟心跳一次

5. maxLifetime
   该属性用于控制连接在池中的最大生存时间，超过该时间强制逐出，连接池向数据申请新的连接进行补充。注意：当前正在使用的连接不会强制逐出，哪怕它的累计时间已经到了maxLifetime。

强烈建议设置该属性，可设置的比数据库或网络基础设施允许的最大连接时间小一些。 如数据库连接最大失效时间是8小时，可设置为4小时。

缺省：1800000， 即30min
最小可配置：30000，即30s

最佳实践：需要设置，根据数据库或网络基础设施的情况，比它们小一些

7. connectionTestQuery
   如果当前连接驱动支持JDBC4, 强烈不建议设置此属性。因为该属性的设置是为了支持keepalive检测，只有当JDBC4的isValid不可用时才使用connectionTestQuery配置的sql进行检测；或者当从池中获取连接时检测连接是否有效

缺省：none

最佳实践：驱动支持JDBC4不需要设置，反之需要配置，MYSQL: select 1; Oracle: select 1 from dual

8. minimumIdle
   配置连接池最小空闲连接数。为了性能最优化和为了应对高峰请求的快速响应强烈不建议设置该属性，让HikariCP连接池保持固定大小。
   缺省：跟maximumPoolSize相同

最佳实践：保持缺省，让连接池固定大小，避免扩缩容带来的性能影响

9. maximumPoolSize
   配置允许连接池达到的最大连接数（包括空闲和正在使用的），当池中连接达到maximumPoolSize，且都不空闲，当有新请求从池中申请连接池会阻塞等待可用连接，达到connectionTimeout还不能申请成功，则抛出SQLException。

缺省：10

最佳实践：根据实际环境配置，通常设置为核心数的2倍较优

10. metricRegistry
    配置该属性，能让连接池将度量信息收集起来供分析。配置的属性是Codahale/Dropwizard的实例，具体可参考wiki页面：https://github.com/brettwooldridge/HikariCP/wiki/Dropwizard-Metrics

11. healthCheckRegistry
    同样是收集信息用，这个属性收集的是连接池的健康状况。配置的属性是Codahale/Dropwizard的实例，具体可参考wiki页面：https://github.com/brettwooldridge/HikariCP/wiki/Dropwizard-HealthChecks



### SpringBoot整合druid(德鲁伊)连接池

官网：[https://druid.apache.org/](https://druid.apache.org/)

①导入druid-spring-boot-starter包(推荐1.2.21版本)
注意：此包1.1.10后的版本数据监控中心做了调整需要自己新增配置类
```xml
<!--引入druid数据源 1.1.10 此版本的数据监控中心可以直接使用-->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid-spring-boot-starter</artifactId>
        <version>1.1.10</version>
    </dependency>
```

```xml
 <!--druid数据源 1.1.21 此版本的数据监控中心增加了登录界面需要增加配置类-->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.21</version>
</dependency>
```

②修改配置文件

```properties
#数据库连接中修改数据源类型
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource

# druid参数调优（可选）,若配置如下参数则必须手动添加配置类
# 初始化大小，最小，最大
spring.datasource.initialSize=5
spring.datasource.minIdle=5
spring.datasource.maxActive=20
# 配置获取连接等待超时的时间
spring.datasource.maxWait=60000
# 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
spring.datasource.timeBetweenEvictionRunsMillis=60000
# 配置一个连接在池中最小生存的时间，单位是毫秒
spring.datasource.minEvictableIdleTimeMillis=300000
# 测试连接
spring.datasource.testWhileIdle=true
spring.datasource.testOnBorrow=false
spring.datasource.testOnReturn=false
# 打开PSCache，并且指定每个连接上PSCache的大小
spring.datasource.poolPreparedStatements=true
# 配置监控统计拦截的filters
# asyncInit是1.1.4中新增加的配置，如果有initialSize数量较多时，打开会加快应用启动时间
spring.datasource.asyncInit=true
# druid监控配置信息
spring.datasource.filters=stat,config
spring.datasource.maxPoolPreparedStatementPerConnectionSize=20
spring.datasource.useGlobalDataSourceStat=true
spring.datasource.connectionProperties=druid.stat.mergeSql=true;druid.stat.slowSqlMillis=500
```

配置文件信息
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test?useSSL=false&autoReconnect=true&characterEncoding=utf8
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: root
    password: test
    # Druid datasource
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      # 初始化大小
      initial-size: 5
      # 最小连接数
      min-idle: 10
      # 最大连接数
      max-active: 20
      # 获取连接时的最大等待时间
      max-wait: 60000
      # 一个连接在池中最小生存的时间，单位是毫秒
      min-evictable-idle-time-millis: 300000
      # 多久才进行一次检测需要关闭的空闲连接，单位是毫秒
      time-between-eviction-runs-millis: 60000
      # 配置扩展插件：stat-监控统计，log4j-日志，wall-防火墙（防止SQL注入），去掉后，监控界面的sql无法统计
      filters: stat,wall
      # 检测连接是否有效的 SQL语句，为空时以下三个配置均无效
      validation-query: SELECT 1
      # 申请连接时执行validationQuery检测连接是否有效，默认true，开启后会降低性能
      test-on-borrow: true
      # 归还连接时执行validationQuery检测连接是否有效，默认false，开启后会降低性能
      test-on-return: true
      # 申请连接时如果空闲时间大于timeBetweenEvictionRunsMillis，执行validationQuery检测连接是否有效，默认false，建议开启，不影响性能
      test-while-idle: true
      # 是否开启 StatViewServlet
      stat-view-servlet:
        enabled: true
        # 访问监控页面 白名单，默认127.0.0.1
        allow: 127.0.0.1
        login-username: admin
        login-password: admin
      # FilterStat
      filter:
        stat:
          # 是否开启 FilterStat，默认true
          enabled: true
          # 是否开启 慢SQL 记录，默认false
          log-slow-sql: true
          # 慢 SQL 的标准，默认 3000，单位：毫秒
          slow-sql-millis: 5000
          # 合并多个连接池的监控数据，默认false
          merge-sql: false
  jpa:
    open-in-view: false
    generate-ddl: false
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
        format_sql: true
        use-new-id-generator-mappings: false
```


③新增数据源配置类DruidConfig
若没有指定连接池参数，则无需此配置类

````java
import com.alibaba.druid.pool.DruidDataSource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import java.sql.SQLException;

/**
 * Druid连接池调优配置信息：只有配置数据库连接池调优信息才需要该类
 *
 *
 */
@Configuration
public class DruidConfig {
    private Logger logger = LoggerFactory.getLogger(DruidConfig.class);

    @Value("${spring.datasource.url}")
    private String dbUrl;

    @Value("${spring.datasource.username}")
    private String username;

    @Value("${spring.datasource.password}")
    private String password;

    @Value("${spring.datasource.driver-class-name}")
    private String driverClassName;

    @Value("${spring.datasource.initial-size}")
    private int initialSize;

    @Value("${spring.datasource.min-idle}")
    private int minIdle;

    @Value("${spring.datasource.max-active}")
    private int maxActive;

    @Value("${spring.datasource.max-wait}")
    private int maxWait;

    @Value("${spring.datasource.time-between-eviction-runs-millis}")
    private int timeBetweenEvictionRunsMillis;

    @Value("${spring.datasource.min-evictable-idle-time-millis}")
    private int minEvictableIdleTimeMillis;

    @Value("${spring.datasource.test-while-idle}")
    private boolean testWhileIdle;

    @Value("${spring.datasource.test-on-borrow}")
    private boolean testOnBorrow;

    @Value("${spring.datasource.test-on-return}")
    private boolean testOnReturn;

    @Value("${spring.datasource.pool-prepared-statements}")
    private boolean poolPreparedStatements;

    @Value("${spring.datasource.max-pool-prepared-statement-per-connection-size}")
    private int maxPoolPreparedStatementPerConnectionSize;

    @Value("${spring.datasource.filters}")
    private String filters;

    @Primary  //在同样的DataSource中，首先使用被标注的DataSource
    @Bean
    public DruidDataSource dataSourceDefault(){
        DruidDataSource datasource = new DruidDataSource();

        datasource.setUrl(this.dbUrl);
        datasource.setUsername(username);
        datasource.setPassword(password);
        datasource.setDriverClassName(driverClassName);

        //configuration
        datasource.setInitialSize(initialSize);
        datasource.setMinIdle(minIdle);
        datasource.setMaxActive(maxActive);
        datasource.setMaxWait(maxWait);
        datasource.setTimeBetweenEvictionRunsMillis(timeBetweenEvictionRunsMillis);
        datasource.setMinEvictableIdleTimeMillis(minEvictableIdleTimeMillis);
        datasource.setTestWhileIdle(testWhileIdle);
        datasource.setTestOnBorrow(testOnBorrow);
        datasource.setTestOnReturn(testOnReturn);
        datasource.setPoolPreparedStatements(poolPreparedStatements);
        datasource.setMaxPoolPreparedStatementPerConnectionSize(maxPoolPreparedStatementPerConnectionSize);
        try {
            datasource.setFilters(filters);
        } catch (SQLException e) {
            logger.error("druid configuration initialization filter", e);
        }
        return datasource;
    }
}
````

④新增数据监控中心配置类DruidMonitorConfig
若jar包版本高于1.1.10时才需要配置该类
```java
import com.alibaba.druid.support.http.StatViewServlet;
import com.alibaba.druid.support.http.WebStatFilter;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.HashMap;
import java.util.Map;

/**
 * Druid连接池监控配置信息
 * 提示：druid-spring-boot-starter jar包的版本高于1.1.10时才需要配置该类
 * 1.若低于1.1.10版本时直接访问：IP:端口/druid/index.html即可
 * 2.若高于1.0.10版本时访问:IP:端口/druid/login.html即可 账号密码根据自己设置的来
 *
 * @author wyj
 * @date 2022/8/16 15:54
 */
@Configuration
public class DruidMonitorConfig {

    //因为Springboot内置了servlet容器，所以没有web.xml，替代方法就是将ServletRegistrationBean注册进去
    //加入后台监控
    @Bean  //这里其实就相当于servlet的web.xml
    public ServletRegistrationBean<StatViewServlet> statViewServlet() {
        ServletRegistrationBean<StatViewServlet> bean =
                new ServletRegistrationBean<>(new StatViewServlet(), "/druid/*");

        //后台需要有人登录，进行配置
        //bean.addUrlMappings(); 这个可以添加映射，我们在构造里已经写了
        //设置一些初始化参数
        Map<String, String> initParas = new HashMap<>();
        initParas.put("loginUsername", "admin");//它这个账户密码是固定的
        initParas.put("loginPassword", "123456");
        //允许谁能防伪
        initParas.put("allow", "");//这个值为空或没有就允许所有人访问，ip白名单
        //initParas.put("allow","localhost");//只允许本机访问，多个ip用逗号,隔开
        //initParas.put("deny","");//ip黑名单，拒绝谁访问 deny和allow同时存在优先deny
        initParas.put("resetEnable", "false");//禁用HTML页面的Reset按钮
        bean.setInitParameters(initParas);
        return bean;
    }

    //再配置一个过滤器，Servlet按上面的方式注册Filter也只能这样
    @Bean
    public FilterRegistrationBean<WebStatFilter> webStatFilter() {
        FilterRegistrationBean<WebStatFilter> bean = new FilterRegistrationBean<>();
        //可以设置也可以获取,设置一个阿里巴巴的过滤器
        bean.setFilter(new WebStatFilter());
        bean.addUrlPatterns("/*");
        //可以过滤和排除哪些东西
        Map<String, String> initParams = new HashMap<>();
        //把不需要监控的过滤掉,这些不进行统计
        initParams.put("exclusions", "*.js,*.css,/druid/*");
        bean.setInitParameters(initParams);
        return bean;
    }
}
```
数据监控中心访问地址：
1.若jar包版本低于1.1.10版本时访问：127.0.0.1:端口/druid/index.html
2.若jar包版本高于1.0.10版本时访问：127.0.0.1:端口/druid/login.html(账号密码根据自己设置的来)


优势
Druid 相对于其他数据库连接池的优点：

- 强大的监控特性，通过Druid提供的监控功能，可以清楚知道连接池和SQL的工作情况
- 监控SQL的执行时间、ResultSet持有时间、返回行数、更新行数、错误次数、错误堆栈信息。
- SQL执行的耗时区间分布,通过耗时区间分布，能够非常清楚知道SQL的执行耗时情况。
- 监控连接池的物理连接创建和销毁次数、逻辑连接的申请和关闭次数、非空等待次数、PSCache命中率等
- 方便扩展，Druid提供了Filter-Chain模式的扩展API，可以自己编写Filter拦截JDBC中的任何方法，可以在上面做任何事情，比如说性能监控、SQL审计、用户名密码加密、日志等等。

Druid集合了开源和商业数据库连接池的优秀特性，并结合阿里巴巴大规模苛刻生产环境的使用经验进行优化。

