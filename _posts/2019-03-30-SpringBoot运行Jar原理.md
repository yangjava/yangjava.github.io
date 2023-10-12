---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot运行原理
为什么SpringBoot的 jar 可以直接运行
SpringBoot提供了一个插件spring-boot-maven-plugin用于把程序打包成一个可执行的jar包。

## Maven依赖
```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>2.5.4</version>
            </plugin>
        </plugins>
    </build>
```
打包完生成的executable-jar-1.0-SNAPSHOT.jar内部的结构如下：
```text
|--META-INF
   |--
```
然后可以直接执行jar包就能启动程序了：
```shell
java -jar executable-jar-1.0-SNAPSHOT.jar
```
打包出来fat jar内部有4种文件类型：
- META-INF文件夹：程序入口，其中MANIFEST.MF用于描述jar包的信息
- lib目录：放置第三方依赖的jar包，比如springboot的一些jar包
- spring boot loader相关的代码
- 模块自身的代码

MANIFEST.MF文件的内容：
```text
Manifest-Version: 1.0
Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx
Implementation-Title: projectmanagerservice
Implementation-Version: 1.0.0
Spring-Boot-Layers-Index: BOOT-INF/layers.idx
Start-Class: com.auto.Application
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Build-Jdk-Spec: 1.8
Spring-Boot-Version: 2.5.4
Created-By: Maven Archiver 3.4.0
Main-Class: org.springframework.boot.loader.JarLauncher
```
我们看到，它的Main-Class是org.springframework.boot.loader.JarLauncher，当我们使用java -jar执行jar包的时候会调用JarLauncher的main方法，而不是我们编写的SpringApplication。

## 那么JarLauncher这个类是的作用是什么的？
它是SpringBoot内部提供的工具Spring Boot Loader提供的一个用于执行Application类的工具类(fat jar内部有spring loader相关的代码就是因为这里用到了)。相当于Spring Boot Loader提供了一套标准用于执行SpringBoot打包出来的jar。

## JarLauncher的执行过程

## Spring Boot Loader的作用
SpringBoot在可执行jar包中定义了自己的一套规则，比如第三方依赖jar包在/lib目录下，jar包的URL路径使用自定义的规则并且这个规则需要使用org.springframework.boot.loader.jar.Handler处理器处理。

它的Main-Class使用JarLauncher，如果是war包，使用WarLauncher执行。这些Launcher内部都会另起一个线程启动自定义的SpringApplication类。这些特性通过spring-boot-maven-plugin插件打包完成。

## 解决 SpringBoot 没有主清单属性
首先我们项目要依赖springboot的parent或者引入spring-boot-dependencies
```xml

```
当使用继承spring-boot-starter-parent时，就会出现标志，表示是继承自父类的，然后我们点到spring-boot-starter-parent的pom文件中，查看插件部分：
```xml
<plugin>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-maven-plugin</artifactId>
          <executions>
            <execution>
              <id>repackage</id>
              <goals>
                <goal>repackage</goal>
              </goals>
            </execution>
          </executions>
          <configuration>
            <mainClass>${start-class}</mainClass>
          </configuration>
        </plugin>
        <plugin>
          <artifactId>maven-shade-plugin</artifactId>
          <executions>
            <execution>
              <phase>package</phase>
              <goals>
                <goal>shade</goal>
              </goals>
              <configuration>
                <transformers>
                  <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                    <resource>META-INF/spring.handlers</resource>
                  </transformer>
                  <transformer implementation="org.springframework.boot.maven.PropertiesMergingResourceTransformer">
                    <resource>META-INF/spring.factories</resource>
                  </transformer>
                  <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                    <resource>META-INF/spring.schemas</resource>
                  </transformer>
                  <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                  <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                    <mainClass>${start-class}</mainClass>
                  </transformer>
                </transformers>
              </configuration>
            </execution>
          </executions>
          <dependencies>
            <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-maven-plugin</artifactId>
              <version>2.1.12.RELEASE</version>
            </dependency>
          </dependencies>
          <configuration>
            <keepDependenciesWithProvidedScope>true</keepDependenciesWithProvidedScope>
            <createDependencyReducedPom>true</createDependencyReducedPom>
            <filters>
              <filter>
                <artifact>*:*</artifact>
                <excludes>
                  <exclude>META-INF/*.SF</exclude>
                  <exclude>META-INF/*.DSA</exclude>
                  <exclude>META-INF/*.RSA</exclude>
                </excludes>
              </filter>
            </filters>
          </configuration>
        </plugin>
```
注意到里面有一个${start-class}变量，这个变量在parent的pom文件中并没有定义，那么我们就在自己要打jar包运行的模块定义这个变量：
如果不是继承自parnetxml，而是选择第一种，导入dependencies的方式：
```xml
     <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>${spring-boot-starter-parent.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
```
那么就要改一下前面的spring-boot-maven-plugin插件
```xml
<plugin>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-maven-plugin</artifactId>
          <version>${spring.version}</version>
          <executions>
            <execution>
              <goals>
                <goal>repackage</goal>
              </goals>
            </execution>
          </executions>
</plugin>
```
使用spring-boot-dependencies这个BOM代替了spring-boot-starter-parent这个parent POM导致spring-boot-maven-plugin的配置项丢失，使得打包后的jar中的MANIFEST.MF文件缺少Main-Class。


















