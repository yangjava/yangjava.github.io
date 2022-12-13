---
layout: post
categories: Maven
description: none
keywords: Maven
---
# Maven之Nexus

## Nexus介绍
Nexus 是Maven仓库管理器，如果你使用Maven，你可以从Maven中央仓库 下载所需要的构件（artifact），但这通常不是一个好的做法，你应该在本地架设一个Maven仓库服务器，在代理远程仓库的同时维护本地仓库，以节省带宽和时间，Nexus就可以满足这样的需要。此外，他还提供了强大的仓库管理功能，构件搜索功能，它基于REST，友好的UI是一个extjs的REST客户端，它占用较少的内存，基于简单文件系统而非数据库。这些优点使其日趋成为最流行的Maven仓库管理器。

## Nexus下载

下载地址：http://www.sonatype.org/nexus/，下载开源版本

NEXUS OSS [OSS = Open Source Software，开源软件——免费]

NEXUS PROFESSIONAL -FREE TRIAL [专业版本——收费]。

## Nexus简单说明
用途：指定私服的中央地址、将自己的Maven项目指定到私服地址、从私服下载中央库的项目索引、从私服仓库下载依赖组件、将第三方项目jar上传到私服供其他项目组使用

仓库：
- hosted   类型的仓库，内部项目的发布仓库
- releases 内部的模块中release模块的发布仓库
- snapshots 发布内部的SNAPSHOT模块的仓库
- 3rd party 第三方依赖的仓库，这个数据通常是由内部人员自行下载之后发布上去
- proxy   类型的仓库，从远程中央仓库中寻找数据的仓库
- group   类型的仓库，组仓库用来方便我们开发人员进行设置的仓库


## Nexus配置
nexus配置大部分使用默认配置即可，主要是配置一个项目索引

选择Central仓库，设置Download Remote Indexes：True

## Nexus使用
项目使用nexus私服的jar包，在项目的pom.xml文件中指定私服仓库
```xml
 <repositories>
     <repository>
         <id>nexus</id>
         <name>nexus</name>
         <url>http://xxxx:8081/nexus/content/groups/public/</url>
         <releases>
             <enabled>true</enabled>
         </releases>
         <snapshots>
             <enabled>true</enabled>
         </snapshots>
     </repository>
 </repositories>
```

项目使用nexus私服的插件，在项目的pom.xml文件中指定插件仓库
```xml
 <pluginRepositories>
     <pluginRepository>
         <id>nexus</id>
         <name>nexus</name>
         <url>http://xxxx:8081/nexus/content/groups/public/</url>
         <releases>
             <enabled>true</enabled>
         </releases>
         <snapshots>
             <enabled>true</enabled>
         </snapshots>
     </pluginRepository>
 </pluginRepositories>
```

如果想本机所有的maven项目都使用私服的组件，可以在maven的设置文件settings.xml中添加属性，并激活
```xml
 <profiles>
     <profile>
         <id>nexusProfile</id>
         <repositories>
             <repository>
                 <id>nexus</id>
                 <name>nexus</name>
                 <url>http://xxxx:8081/nexus/content/groups/public/</url>
                 <releases>
                     <enabled>true</enabled>
                 </releases>
                 <snapshots>
                     <enabled>true</enabled>
                 </snapshots>
             </repository>
         </repositories>
     </profile>
 </profiles>
 <!-- 激活 -->
 <activeProfiles>
     <activeProfile>nexusProfile</activeProfile>
 </activeProfiles>
```

项目发布到私服，maven项目使用命令：mvn clean deploy；需要在pom文件中配置一下代码；
```xml
 <distributionManagement>
         <repository>
             <id>user-release</id>
             <name>User Project Release</name>
             <url>http://xxxx:8081/nexus/content/repositories/releases/</url>
         </repository>
 
         <snapshotRepository>
             <id>user-snapshots</id>
             <name>User Project SNAPSHOTS</name>
             <url>http://xxxx:8081/nexus/content/repositories/snapshots/</url>
         </snapshotRepository>
     </distributionManagement>
```

注意还需要配置mvn发布的权限，否则会报401错误，在settings.xml中配置权限，其中id要与pom文件中的id一致
```xml
<server>
     <id>user-release</id>
     <username>admin</username>
     <password>admin123</password>
 </server>
 <server>
     <id>user-snapshots</id>
     <username>admin</username>
     <password>admin123</password>
 </server>
```




