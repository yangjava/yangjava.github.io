---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot项目依赖管理
通过spring initializr，我们可以快速构建一个springboot应用，如果你选择的是Maven来管理项目，在默认的pom文件中有这么一个部分：
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.6</version>
</parent>
```
它表示当前pom文件从spring-boot-starter-parent继承下来，在spring-boot-starter-parent中提供了很多默认的配置，这些配置可以大大简化我们的开发。

## spring-boot-dependencies
一般用来放在父项目中，来声明依赖，子项目引入相关依赖而不需要指定版本号，好处就是解决依赖冲突，统一管理依赖版本号

利用pom的继承，一处声明，处处使用。在最顶级的spring-boot-dependencies中，使用dependencyManagement让所有子项目引用一个依赖而不用显式的列出版本号，将结构信息，部署信息，共同的依赖信息放置在统一的位置。dependencyManagement只声明依赖，并不真正引入，因此子项目需要通过dependencies引入相关依赖。

Maven 父子项目结构和 Java继承一样，都是单继承，一个子项目只能制定一个父 pom

但是如果项目中已经有了其他父pom， 又想用 spring-boot 怎么办呢？

```xml
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>2.2.6.RELEASE</version>
        <type>pom</type>
        <scope>import</scope>  <!-- import scope只能用在dependencyManagement里面，且仅用于type=pom的dependency。解决单继承问题-->  
      </dependency>
```

## SpringBoot2 依赖管理

springboot2-dependencies管理SpringBoot2的所有依赖，管理版本号等。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.springboot2</groupId>
    <artifactId>springboot2-dependencies</artifactId>
    <version>${revision}</version>
    <packaging>pom</packaging>
    <name>${project.artifactId}</name>
    <description>基础 bom 文件，管理整个项目的依赖版本</description>


    <properties>
        <revision>1.0.0-SNAPSHOT</revision>
        <!-- SpringBoot依赖管理 -->
        <spring.boot.version>2.7.6</spring.boot.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!-- SpringBoot依赖管理 -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring.boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

在springboot2父类中管理所有的依赖项

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.springboot2</groupId>
    <artifactId>springboot2</artifactId>
    <version>${revision}</version>
    <packaging>pom</packaging>

    <modules>
        <module>springboot2-dependencies</module>
        <module>springboot2-spring-boot-starter-web</module>
    </modules>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <revision>1.0.0-SNAPSHOT</revision>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.springboot2</groupId>
                <artifactId>springboot2-dependencies</artifactId>
                <version>${revision}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

在springboot2-spring-boot-starter-web中的web中管理web模块的依赖，不需要相关的依赖管理了。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.springboot2</groupId>
        <artifactId>springboot2</artifactId>
        <version>${revision}</version>
    </parent>

    <artifactId>springboot2-spring-boot-starter-web</artifactId>

    <dependencies>

        <!-- Web 相关 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

    </dependencies>

</project>
```

## SpringBoot3 依赖管理
要管理SpringBoot3依赖，只需要将springboot2-dependencies这个依赖文件进行升级即可。


## 数据库连接池
SpringBoot默认连接池(Hikari)配置如下：
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


## web通用处理器









