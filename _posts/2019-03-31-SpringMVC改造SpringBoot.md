---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringMVC改造SpringBoot
老项目使用SpringMVC，现在需要改造程SpringBoot。

## Maven依赖管理
将SpringMVC依赖修改为SpringBoot依赖
```xml
   <properties>
    <spring-boot-starter-parent.version>2.2.0.RELEASE</spring-boot-starter-parent.version>
    </properties>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-parent</artifactId>
                <version>${spring-boot-starter-parent.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

## XML配置文件
因为springmvc项目里都是通过xml文件配置的，配置文件有很多，不可能全部都用java配置类来重写，所以需要想办法通过springboot的注解来加载配置文件
关键注解：@ImportResource
新建一个配置类ShiroConfiguration，加上@Configuration注解，通过@ImportResource加载spring-shiro.xml配置文件，配置shiro相关信息
```java
@Configuration
@ImportResource(locations={"classpath:spring-shiro.xml"})
public class ShiroConfiguration {
}
```

## SpringBoot启动类
在web的模块下新建springboot启动主程序WebsiteApplication
```java

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```



## 定时器问题
跟shiro和定时器相关：

报错：

java.lang.InstantiationError: org.quartz.SimpleTrigger

或者：

java.lang.IncompatibleClassChangeError: Found interface org.quartz.JobExecutionContext, but class was expected

这些应该都跟shiro和org.quartz版本有关，各种尝试之后都没有解决，最后解决方案：

引入依赖2.2.1版本的quartz必须是2.*版本以上：

```xml
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.2.1</version>
    <exclusions>
        <exclusion>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-quartz</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```
然后在spring-shiro.xml里指定id为sessionValidationScheduler的bean的配置改为如下：

```xml
  <bean id="sessionValidationScheduler" class="org.apache.shiro.session.mgt.ExecutorServiceSessionValidationScheduler">
<!--  <bean id="sessionValidationScheduler" class="shenji.report.analysis.bind.QuartzSessionValidationScheduler">-->
    <!-- 定时清理失效会话, 清理用户直接关闭浏览器造成的孤立会话，单位为毫秒，即默认为3600000 -->
<!--    <property name="sessionValidationInterval" value="3600000"/>-->
    <property name="interval" value="3600000"/>
    <property name="sessionManager" ref="sessionManager"/>
  </bean>
```
原来是自己去实现指定的class，都出现了问题，后来指定为shiro自己的ExecutorServiceSessionValidationScheduler问题解决



