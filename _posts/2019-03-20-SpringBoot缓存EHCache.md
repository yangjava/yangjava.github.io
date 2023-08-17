---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot缓存EHCache
## EHCache
EhCache 是一个纯Java的进程内缓存框架，具有快速、精干等特点，是Hibernate中默认CacheProvider。Ehcache是一种广泛使用的开源Java分布式缓存。主要面向通用缓存,Java EE和轻量级容器。它具有内存和磁盘存储，缓存加载器,缓存扩展,缓存异常处理程序,一个gzip缓存servlet过滤器,支持REST和SOAP api等特点。

### 导入依赖
引入springboot-cache和ehcache。需要注意，EhCache不需要配置version，SpringBoot的根pom已经集成了。
```xml
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

### 加入配置

```properties
spring.cache.type=ehcache # 配置ehcache缓存
spring.cache.ehcache.config=classpath:/ehcache.xml # 指定ehcache配置文件路径 ，可以不用写，因为默认就是这个路径，SpringBoot会自动扫描
```

### ehcache配置文件

EhCache的配置文件ehcache.xml只需要放到类路径下面，SpringBoot会自动扫描

```xml
<ehcache>

    <!--
        磁盘存储:指定一个文件目录，当EHCache把数据写到硬盘上时，将把数据写到这个文件目录下
        path:指定在硬盘上存储对象的路径
        path可以配置的目录有：
            user.home（用户的家目录）
            user.dir（用户当前的工作目录）
            java.io.tmpdir（默认的临时目录）
            ehcache.disk.store.dir（ehcache的配置目录）
            绝对路径（如：d:\\ehcache）
        查看路径方法：String tmpDir = System.getProperty("java.io.tmpdir");
     -->
    <diskStore path="java.io.tmpdir" />

    <!--
        defaultCache:默认的缓存配置信息,如果不加特殊说明,则所有对象按照此配置项处理
        maxElementsInMemory:设置了缓存的上限,最多存储多少个记录对象
        eternal:代表对象是否永不过期 (指定true则下面两项配置需为0无限期)
        timeToIdleSeconds:最大的发呆时间 /秒
        timeToLiveSeconds:最大的存活时间 /秒
        overflowToDisk:是否允许对象被写入到磁盘
        说明：下列配置自缓存建立起600秒(10分钟)有效 。
        在有效的600秒(10分钟)内，如果连续120秒(2分钟)未访问缓存，则缓存失效。
        就算有访问，也只会存活600秒。
     -->
    <defaultCache maxElementsInMemory="10000" eternal="false"
                  timeToIdleSeconds="600" timeToLiveSeconds="600" overflowToDisk="true" />
    <!-- 按缓存名称的不同管理策略 -->
    <cache name="myCache" maxElementsInMemory="10000" eternal="false"
                  timeToIdleSeconds="120" timeToLiveSeconds="600" overflowToDisk="true" />

</ehcache>
```

### 装配

SpringBoot会为我们自动配置 EhCacheCacheManager 这个Bean，如果想自定义设置一些个性化参数时，通过Java Config形式配置。
```java
@Configuration  
@EnableCaching
public class CacheConfig {  
  
    @Bean  
    public CacheManager cacheManager() {  
        return new EhCacheCacheManager(ehCacheCacheManager().getObject());  
    }  
  
    @Bean  
    public EhCacheManagerFactoryBean ehCacheCacheManager() {  
        EhCacheManagerFactoryBean cmfb = new EhCacheManagerFactoryBean();  
        cmfb.setConfigLocation(new ClassPathResource("ehcache.xml"));  
        cmfb.setShared(true);  
        return cmfb;  
    }  
  
}
```