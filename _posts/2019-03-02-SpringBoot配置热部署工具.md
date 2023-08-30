---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot热部署工具

## 什么是热部署和热加载？

热部署和热加载是在应用正在运行的时候，自动更新（重新加载或者替换class等）应用的一种能力。（PS：spring-boot-devtools提供的方案也是要重启的，只是无需手动重启能实现自动加载而已。）

严格意义上，我们需要区分下热部署和热加载, 对于Java项目而言：

热部署

- 在服务器运行时重新部署项目
- 它是直接重新加载整个应用，这种方式会释放内存，比热加载更加干净彻底，但同时也更费时间。

热加载

- 在在运行时重新加载class，从而升级应用。
- 热加载的实现原理主要依赖java的类加载机制，在实现方式可以概括为在容器启动的时候起一条后台线程，定时的检测类文件的时间戳变化，如果类的时间戳变掉了，则将类重新载入。
- 对比反射机制，反射是在运行时获取类信息，通过动态的调用来改变程序行为； 热加载则是在运行时通过重新加载改变类信息，直接改变程序行为。


## 什么是LiveLoad？

LiveLoad是提供浏览器客户端自动加载更新的工具，分为LiveLoad服务器和Liveload浏览器插件两部分； devtools中已经集成了LiveLoad服务器，所以如果我们开发的是web应用，并且期望浏览器自动刷新， 这时候可以考虑LiveLoad.

同一时间只能运行一个LiveReload服务器。 开始应用程序之前，请确保没有其他LiveReload服务器正在运行。如果从IDE启动多个应用程序，则只有第一个应用程序将支持LiveReload。

## 实现热部署

### POM配置

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
        <!-- 可以防止将devtools依赖传递到其他模块中 -->
    </dependency>
</dependencies>
```

### IDEA配置

方式一： 无任何配置时，手动触发重启更新（Ctrl+F9）
方式二： IDEA需开启运行时编译，自动重启更新

File->Setting->Build,Execution,Deployment->Compile

勾选：Make project automatically


### application.yml配置

```yaml

spring:
  devtools:
    restart:
      enabled: true  #设置开启热部署
      additional-paths: src/main/java #重启目录
      exclude: WEB-INF/**
  thymeleaf:
    cache: false #使用Thymeleaf模板引擎，关闭缓存

```

## devtool的原理

为什么同样是重启应用，为什么不手动重启，而是建议使用spring-boot-devtools进行热部署重启？

spring-boot-devtools使用了两个类加载器ClassLoader，一个ClassLoader加载不会发生更改的类（第三方jar包），另一个ClassLoader（restart ClassLoader）加载会更改的类（自定义的类）。

后台启动一个文件监听线程（File Watcher），监测的目录中的文件发生变动时， 原来的restart ClassLoader被丢弃，将会重新加载新的restart ClassLoader。因为文件变动后，第三方jar包不再重新加载，只加载自定义的类，加载的类比较少，所以重启比较快。这也是为什么，同样是重启应用，为什么不手动重启，建议使用spring-boot-devtools进行热部署重启。


