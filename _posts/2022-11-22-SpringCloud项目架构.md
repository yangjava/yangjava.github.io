---
layout: post
categories: [SpringCloud]
description: none
keywords: SpringCloud
---
# SpringCloud项目架构

# Maven依赖管理
```xml
<properties>
    <java.version>8</java.version>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>8</maven.compiler.source>
    <maven.compiler.target>8</maven.compiler.target>
    <spring-cloud.version>Hoxton.SR9</spring-cloud.version>
    <spring-cloud-alibaba.version>2.2.6.RELEASE</spring-cloud-alibaba.version>
    <spring-boot.version>2.3.2.RELEASE</spring-boot.version>
</properties>

    <dependencyManagement>
        <dependencies>
            <!-- SpringCloud -->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- SpringCloud Alibaba -->
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!--Springboot -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-parent</artifactId>
                <version>${spring-boot.version}</version>
                <!--以下两个配置为引用程序版本将有springBoot来进行管理-->
                <type>pom</type>
                <scope>import</scope>
            </dependency>

        </dependencies>
    </dependencyManagement>
```