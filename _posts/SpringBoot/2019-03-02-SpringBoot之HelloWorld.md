---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot之HelloWorld  
  纸上得来终觉浅，绝知此事要躬行。下面就让我们开始一个Hello World级别的项目吧。

## Maven依赖

引入SpringBoot依赖版本  
```xml
<dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>2.5.4</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

```

启动器 引入spring-boot-starter-web依赖  

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
```

SpringBoot build插件
```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
```


## 配置文件application.yml

yml文件和properties配置文件具有同样的功能。二者的区别在于：  
- yml文件的层级更加清晰直观，但是书写时需要注意格式缩进对齐。yml格式配置文件更有利于表达复杂数据结构的配置。比如：列表，对象（后面章节会详细说明）。
- properties阅读上不如yml直观，好处在于书写时不用特别注意格式缩进对齐。

```yaml
server:
  port: 8888   # web应用服务端口
```

## HelloWorld类

```java

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class Application {

  public static void main(String[] args) {
    SpringApplication.run(Application.class);
  }
  /**
   * hello world.
   *
   * @return hello
   */
  @GetMapping("/hello")
  public ResponseEntity<String> hello() {
    return new ResponseEntity<>("hello world", HttpStatus.OK);
  }
}

```

启动即可启动一个WEB服务，通过浏览器访问http://localhost:8080/hello, 并返回Hello world

## 服务启动日志
```
[           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
[           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
[           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.35]
[           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
[           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1057 ms
[           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
[           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
[           main] com.demo.Application                     : Started Application in 1.835 seconds (JVM running for 3.032)
[nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
[nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
[nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 5 ms
```

## 项目详情

###  项目结构
项目结构目录整体上符合maven规范要求：

| 目录位置                                  | 功能                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| src/main/java                             | 项目java文件存放位置，初始化包含主程序入口 XxxApplication，可以通过直接运行该类来 启动 Spring Boot应用 |
| src/main/resources                        | 存放静态资源，图片、CSS、JavaScript、web页面模板文件等       |
| src/test                                  | 单元测试代码目录                                             |
| .gitignore                                | git版本管理排除文件                                          |
| target文件夹                              | 项目代码构建打包结果文件存放位置，不需要人为维护             |
| pom.xml                                   | maven项目配置文件                                            |
| application.properties（application.yml） | 用于存放程序的各种依赖模块的配置信息，比如服务端口，数据库连接配置等 |

### Maven依赖

```xml
<dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>2.3.0.RELEASE</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

```
spring-boot-dependencies来真正管理Spring Boot应用里面的所有依赖版本。以后我们导入依赖默认是不需要写版本；（没有在dependencies里面管理的依赖需要声明版本号）


### 启动器

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
```

**spring-boot-starter-web**:spring-boot场景启动器；帮我们导入了web模块正常运行所依赖的组件。  
Spring Boot将所有的功能场景都抽取出来，做成一个个的starters（启动器），只需要在项目里面引入这些starter相关场景的所有依赖都会导入进来。要用什么功能就导入什么场景的启动器。

### 主入口类

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

  public static void main(String[] args) {
    SpringApplication.run(Application.class);
  }
}
```

**`@SpringBootApplication`**: Spring Boot应用标注在某个类上说明这个类是SpringBoot的主配置类，SpringBoot就应该运行这个类的main方法来启动SpringBoot应用。

## SpringBoot还提供了哪些starter模块呢？

| 名称                                  | 说明                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| spring-boot-starter	                    | 核心 POM，包含自动配置支持、日志库和对 YAML 配置文件的支持。 |
| spring-boot-starter-amqp                  | 通过 spring-rabbit 支持 AMQP。      |
| spring-boot-starter-aop	                | 包含 spring-aop 和 AspectJ 来支持面向切面编程（AOP）。                                             |
| spring-boot-starter-batch	                | 支持 Spring Batch，包含 HSQLDB。                                          |
| spring-boot-starter-data-jpa	            | 包含 spring-data-jpa、spring-orm 和 Hibernate 来支持 JPA。          |
| spring-boot-starter-data-mongodb          | 包含 spring-data-mongodb 来支持 MongoDB。                                          |
| spring-boot-starter-data-rest	            | 通过 spring-data-rest-webmvc 支持以 REST 方式暴露 Spring Data 仓库。|
| spring-boot-starter-jdbc		            | 支持使用 JDBC 访问数据库。|
| spring-boot-starter-security		        | 包含 spring-security。|
| spring-boot-starter-test		            | 包含常用的测试所需的依赖，如 JUnit、Hamcrest、Mockito 和 spring-test 等。|
| spring-boot-starter-velocity		        | 支持使用 Velocity 作为模板引擎。|
| spring-boot-starter-web		            | 支持 Web 应用开发，包含 Tomcat 和 spring-mvc。|
| spring-boot-starter-websocket	            | 支持使用 Tomcat 开发 WebSocket 应用。|
| spring-boot-starter-ws		            | 支持 Spring Web Services。|
| spring-boot-starter-actuator	            | 添加适用于生产环境的功能，如性能指标和监测等功能。|
|spring-boot-starter-remote-shell		    | 添加远程 SSH 支持。|
| spring-boot-starter-jetty		            | 使用 Jetty 而不是默认的 Tomcat 作为应用服务器。|
| spring-boot-starter-log4j		            | 添加 Log4j 的支持。|
| spring-boot-starter-logging		        | 使用 Spring Boot 默认的日志框架 Logback。|
| spring-boot-starter-tomcat	            | 使用 Spring Boot 默认的 Tomcat 作为应用服务器。|


所有这些 POM 依赖的好处在于为开发 Spring 应用提供了一个良好的基础。Spring Boot 所选择的第三方库是经过考虑的，是比较适合产品开发的选择。但是 Spring Boot 也提供了不同的选项，比如日志框架可以用 Logback 或 Log4j，应用服务器可以用 Tomcat 或 Jetty。