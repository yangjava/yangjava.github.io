---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot3源码环境搭建
在maven仓库中搜索SpringBoot3的Maven依赖
https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-parent
maven依赖
```xml
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>3.0.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```
SpringBoot3启动类
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

日志信息
```text
[           main] com.demo.springboot.Application          : Starting Application using Java 17.0.5 with PID 9664 (D:\Qjd\springboot3\target\classes started by Admin in D:\Qjd\springboot3)
[           main] com.demo.springboot.Application          : No active profile set, falling back to 1 default profile: "default"
[           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
[           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
[           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.1]
[           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
[           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 1633 ms
[           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
[           main] com.demo.springboot.Application          : Started Application in 2.828 seconds (process running for 3.531)
```
注意点：
Spring Boot 3的是Spring 6，Spring 6需要JDK17