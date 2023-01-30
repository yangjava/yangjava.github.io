---
layout: post
categories: [SpringBoot,Mybatis]
description: none
keywords: SpringBoot
---
# SpringBoot整合Mybatis
MyBatis之前，先搭建一个基本的Spring Boot项目然后引入mybatis-spring-boot-starter和数据库连接驱动

## Maven依赖
```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.3.1</version>
</dependency>

<!-- MySQL驱动 -->
<dependency>
<groupId>mysql</groupId>
<artifactId>mysql-connector-java</artifactId>
</dependency>
```
不同版本的Spring Boot和MyBatis版本对应不一样，具体可查看官方文档：
[http://www.mybatis.org/spring-boot-starter/mybatis-spring-boot-autoconfigure/](http://www.mybatis.org/spring-boot-starter/mybatis-spring-boot-autoconfigure/)

通过dependency:tree命令查看mybatis-spring-boot-starter都有哪些隐性依赖：
```xml
+- org.mybatis.spring.boot:mybatis-spring-boot-starter:jar:1.3.1:compile
|  +- org.springframework.boot:spring-boot-starter-jdbc:jar:1.5.9.RELEASE:compile
|  |  +- org.apache.tomcat:tomcat-jdbc:jar:8.5.23:compile
|  |  |  \- org.apache.tomcat:tomcat-juli:jar:8.5.23:compile
|  |  \- org.springframework:spring-jdbc:jar:4.3.13.RELEASE:compile
|  |     \- org.springframework:spring-tx:jar:4.3.13.RELEASE:compile
|  +- org.mybatis.spring.boot:mybatis-spring-boot-autoconfigure:jar:1.3.1:compile
|  +- org.mybatis:mybatis:jar:3.4.5:compile
|  \- org.mybatis:mybatis-spring:jar:1.3.1:compile
```

## 配置数据源



## mybatis.mapper-locations配置问题详解
springboot或者spring项目经常会引用其它项目，把其它项目的Jar包加进来，因为每个项目的包路径不一样,mapper.xml的路径也不一样，这个时候就需要引入多个路径。

### *.xml文件路径在resources包下时，可根据路径配置如下
```properties
# 方法一：只有一个路径
mybatis.mapper-locations= classpath:mapper/*.xml
# 方法二：有多个路径
mybatis.mapper-locations= classpath:mapper/*.xml,classpath:mapper/user*.xml
# 方法三：通配符 ** 表示任意级的目录
mybatis.mapper-locations= classpath:**/*.xml
```
### *.xml文件路径在java包下时，不可使用mybatis.mapper-locations配置，可根据路径配置如下

在pom.xml的<build>标签中添加如下
```xml
   <build>
        <resources>
            <resource>
                <directory>src/main/java</directory>
                <includes>
                    <include>**/*.xml</include>
                </includes>
            </resource>
            <resource>
                <directory>src/main/resources</directory>
            </resource>
        </resources>
    </build>

```



# 参考资料





