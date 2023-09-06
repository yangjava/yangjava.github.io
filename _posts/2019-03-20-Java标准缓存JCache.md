---
layout: post
categories: [Cache]
description: none
keywords: JCache
---
# Java标准缓存JCache
JCache是Java的标准缓存API。

## 概述JCache
Java临时缓存API(JSR-107)，也被称为JCache，它是一个规范在javax.cache.API中定义的。该规范是在Java Community Process下开发的，它的目的是为java应用程序提供一个标准的缓存概念和机制。

这套API用起来很方便，它被设计成缓存标准并且是平台无关的，它消除了过去不同供应商之间API是不同的，这导致开发人员坚持使用他们已经使用的专有API，而不是研究新的API，因为研究其他产品的门槛太高了。

作为应用程序开发人员，您可以很容易地使用来自某个供应商的JCache API开发应用程序，如果您愿意，可以尝试使用其他供应商的JCache实现，而不需要更改应用程序的任何一行代码。您所要做的就是使用所选供应商的JCache缓存库。这意味着您可以避免在应用程序中为了尝试新的缓存解决方案而重写大量与缓存相关的代码。

## JSR107 (JCache)
JCache 是 Java 的缓存 API。它由 JSR107 定义。它定义了供开发人员使用的标准 Java 缓存 API 和供实现者使用的标准 SPI（“服务提供者接口”）。

### Maven依赖
要使用JCache，我们需要在pom.xml中添加以下依赖项：
```xml
<dependency>
    <groupId>javax.cache</groupId>
    <artifactId>cache-api</artifactId>
    <version>1.0.0-PFD</version>
</dependency>
```
为了在应用程序中使用JCache的API，你需要下面的两个jar包：

JCache的jar包，它定义了JCache的API接口
Ehcache的jar包，它实现JCache中的接口，将JCache的调用转换成Ehcache的调用。
你可以使用JCacheAPI去开发一个完整的应用，不需要调用任何Ehcache的API。

## JCache 核心概念
Java 的缓存 API 定义了五个核心接口：CachingProvider，CacheManager，Cache，Entry 和 ExpiryPolicy。

### CachingProvider
定义了建立，配置，获取，管理和控制零个或多个 CacheManager 的机制。应用程序可以在运行期间访问和使用零个或多个 CacheProvider。

### CacheManager
定义在上下文中了建立，配置，获取，管理和控制零个或多个唯一命名的缓存的机制。 CacheManager 被单个 CachingProvider 拥有。

### Cache
是一个像 Map 一样的数据结构，它允许基于 Key 的临时储存。缓存被单个 CacheManager 拥有。

### Entry
是被 Cache 存储的单个 key-value 对，JCache 允许我们定义按值或按引用来存储条目。

### ExpiryPolicy
每一个被 Cache 存储的 entry 都定义了存活时间，被称作过期持续时间。**缓存过期时间是可以动态设置的，在执行某些缓存操作后，缓存条目将在设置的时间后过期。**一旦这个过期时间到达，该条目就被认为是过期了。一旦过期，就会从缓存中驱逐出去，不能再访问，更新和删除条目。

- getExpiryForCreation() - 创建条目时的持续时间
- getExpiryForAccess() - 条目被访问时的新持续时间
- getExpiryForUpdate() - 条目被更新时的新持续时间
另外，getExpiryForUpdate() 和 getExpiryForAccess() 也可能返回 null，这表示缓存实现应保留创建时这些操作的条目的有效期限不变

## 开始使用Ehcache和JCache
JCache规范除了定义Cache接口，还定义了CachingProvider和CacheManager规范。应用程序需要使用CacheManager去创建/获取一个cache，同样的使用CachingProvider去获取/访问CacheManager.

下面有个代码样例显示了基本JCache配置的API：
```
CachingProvider provider = Caching.getCachingProvider();  
CacheManager cacheManager = provider.getCacheManager();   
MutableConfiguration<Long, String> configuration =
    new MutableConfiguration<Long, String>()  
        .setTypes(Long.class, String.class)  
        .setStoreByValue(false)  
        .setExpiryPolicyFactory(CreatedExpiryPolicy.factoryOf(Duration.ONE_MINUTE));  
Cache<Long, String> cache = cacheManager.createCache("jCache", configuration); 
cache.put(1L, "one"); 
String value = cache.get(1L);
```
- 从应用程序的classpath中获得一个默认的CachingProvider实现。当classpath中只有一个实现时这个方法才可以正常使用。如果classpath中有多个实现，那么使用全限定名org.ehcache.jsr107.EhcacheCachingProvider去获取Ehcache的实现。你可以通过使用静态方法Caching.getCachingProvider(String)来实现这一点。
-  通过Provider来获取默认的CacheManager
- 使用MutableConfiguration创建一个cache配置
- 设置key-type和value-type
- 设置通过引用存储cache entries
- 设置过期时间为从创建的那一刻起之后的一分钟
- 使用CacheManager和第三步创建的configuration，创建一个名为JCache的cache
- 向cache中存入一些数据
- 从cache中获取数据

## 集成JCache和Ehcache的配置
正如前面提到的，JCache提供了一组最小的配置，对于内存缓存非常理想。但是Ehcache原生api支持复杂得多的拓扑，并且提供了更多的特性。有时，应用程序开发人员可能希望配置比JCache MutableConfiguration允许的缓存复杂得多的缓存(就拓扑或特性而言)，同时仍然能够使用JCache缓存的api。所以Ehcache提供了几种实现此目的的方法，如下面的部分所述。

通过JCache配置访问底层的Ehcache配置

当你使用CacheManager和MutableConfiguration创建Cache，换句话说只是用JCache类型时，你仍然可以访问底层的Ehcache CacheRuntimeConfiguration:
```
MutableConfiguration<Long, String> configuration = new MutableConfiguration<>();
configuration.setTypes(Long.class, String.class);
Cache<Long, String> cache = cacheManager.createCache("someCache", configuration);

CompleteConfiguration<Long, String> completeConfiguration = cache.getConfiguration(CompleteConfiguration.class); 

Eh107Configuration<Long, String> eh107Configuration = cache.getConfiguration(Eh107Configuration.class); 

CacheRuntimeConfiguration<Long, String> runtimeConfiguration = eh107Configuration.unwrap(CacheRuntimeConfiguration.class); 
```
- 使用JCache规范中的MutableConfiuration接口创建JCache cache
- 获取CompleteConfiguration
- 获取连接Ehcache和JCache的配置桥
- 换成CacheRuntimeConfiguration类型

使用Ehcache API构建JCache配置

如果你需要在CacheManager级别配置，就像持久化目录那样，你需要使用特定的API，你可以像下面这么做：
```
CachingProvider cachingProvider = Caching.getCachingProvider();
EhcacheCachingProvider ehcacheProvider = (EhcacheCachingProvider) cachingProvider; 

DefaultConfiguration configuration = new DefaultConfiguration(ehcacheProvider.getDefaultClassLoader(),
  new DefaultPersistenceConfiguration(getPersistenceDirectory())); 

CacheManager cacheManager = ehcacheProvider.getCacheManager(ehcacheProvider.getDefaultURI(), configuration); 
```
- 转换CachingProvider为Ehcache指定的实现org.ehcache.jsr107.EhcacheCachingProvider
- 使用指定的Ehcache DefaultConfiguration创建一个configuration并把它传给CacheManager级别的配置中
- 使用参数为Ehcache configuration的方法创建CacheManager

缓存级别配置

还可以使用Ehcache CacheConfiguration创建JCache缓存。当使用此机制时，没有使用JCache complete teconfiguration，因此您不能使用它
```
CacheConfiguration<Long, String> cacheConfiguration = CacheConfigurationBuilder.newCacheConfigurationBuilder(Long.class, String.class,
    ResourcePoolsBuilder.heap(10)).build(); 

Cache<Long, String> cache = cacheManager.createCache("myCache",
    Eh107Configuration.fromEhcacheCacheConfiguration(cacheConfiguration)); 

Eh107Configuration<Long, String> configuration = cache.getConfiguration(Eh107Configuration.class);
configuration.unwrap(CacheConfiguration.class); 

configuration.unwrap(CacheRuntimeConfiguration.class); 

try {
  cache.getConfiguration(CompleteConfiguration.class); 
  throw new AssertionError("IllegalArgumentException expected");
} catch (IllegalArgumentException iaex) {
  // Expected
}
```
- 你可以使用象上面使用的构建器或者使用XML配置来创建Ehcache CacheConfiguration
- 通过包装Ehcache配置获得JCache配置
- 获取Ehcache CacheConfiguration
- 获取运行时配置
- 在这个上下文中没有JCache CompleteConfiguration可以获得

使用Ehcache XML配置构建JCache配置

JCache缓存上拥有完整的Ehcache配置选项的另一种方法是使用基于xml的配置，有关用XML配置缓存的更多细节，请参阅XML文档。下面是一个XML配置的例子:
```
<config
    xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'
    xmlns='http://www.ehcache.org/v3'
    xsi:schemaLocation="
        http://www.ehcache.org/v3 http://www.ehcache.org/schema/ehcache-core-3.0.xsd">

  <cache alias="ready-cache">
    <key-type>java.lang.Long</key-type>
    <value-type>com.pany.domain.Product</value-type>
    <loader-writer>
      <class>com.pany.ehcache.integration.ProductCacheLoaderWriter</class>
    </loader-writer>
    <heap unit="entries">100</heap>
  </cache>

</config>
```
下面是如何使用JCache访问XML配置的示例：
```
CachingProvider cachingProvider = Caching.getCachingProvider();
CacheManager manager = cachingProvider.getCacheManager( 
    getClass().getResource("/org/ehcache/docs/ehcache-jsr107-config.xml").toURI(),
    getClass().getClassLoader()); 
Cache<Long, Product> readyCache = manager.getCache("ready-cache", Long.class, Product.class);
```
- 调用javax.cache.spi.CachingProvider.getCacheManager(java.net.URI, java.lang.ClassLoader)
- 并且传入URI，它解析为Ehcache XML配置文件
- 第二个参数是ClassLoader，在需要时用于加载用户类型，Class实例存储在由CacheManager管理的Cache中
- 从CacheManager中获取Cache

使用/禁止 MBeans

当使用Ehcache XML配置时，您可能希望为JCache缓存启用管理或统计mbean。这使您可以控制以下内容:

javax.cache.configuration.CompleteConfiguration.isStatisticsEnabled

javax.cache.configuration.CompleteConfiguration.isManagementEnabled

您可以在两个不同的级别上这样做：
```
<config
    xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'
    xmlns='http://www.ehcache.org/v3'
    xmlns:jsr107='http://www.ehcache.org/v3/jsr107'
    xsi:schemaLocation="
        http://www.ehcache.org/v3 http://www.ehcache.org/schema/ehcache-core-3.0.xsd
        http://www.ehcache.org/v3/jsr107 http://www.ehcache.org/schema/ehcache-107-ext-3.0.xsd">


  <service>
    <jsr107:defaults enable-management="true" enable-statistics="true"/> (1)
  </service>

  <cache alias="stringCache"> (2)
    <key-type>java.lang.String</key-type>
    <value-type>java.lang.String</value-type>
    <heap unit="entries">2000</heap>
  </cache>

  <cache alias="overrideCache">
    <key-type>java.lang.String</key-type>
    <value-type>java.lang.String</value-type>
    <heap unit="entries">2000</heap>
    <jsr107:mbeans enable-management="false" enable-statistics="false"/> (3)
  </cache>

  <cache alias="overrideOneCache">
    <key-type>java.lang.String</key-type>
    <value-type>java.lang.String</value-type>
    <heap unit="entries">2000</heap>
    <jsr107:mbeans enable-statistics="false"/> (4)
  </cache>
</config>
```
- 使用JCache服务扩展，默认情况下可以启用MBean
- 根据服务配置，缓存stringCache将启用这两个MBean
- 缓存overrideCache将禁用两个MBean
- 缓存overrideOneCache将禁用统计MBean，而管理MBean将根据服务配置启用

使用EhcacheXML扩展来补充JCache的配置

您还可以创建cache-templates。有关详细信息，请参阅XML文档的缓存模板部分。Ehcache作为JCache的缓存的实现，提供了对常规XML配置的扩展，所以你可以:

配置一个默认模板，所有以编程方式创建的缓存实例都继承该模板
配置给定的命名缓存以从特定模板继承。
这个特性对于配置超出JCache规范范围的缓存特别有用，例如为缓存提供容量约束。为此在XML configura中添加一个jsr107服务
```
<config
    xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'
    xmlns='http://www.ehcache.org/v3'
    xmlns:jsr107='http://www.ehcache.org/v3/jsr107'
    xsi:schemaLocation="
        http://www.ehcache.org/v3 http://www.ehcache.org/schema/ehcache-core-3.0.xsd
        http://www.ehcache.org/v3/jsr107 http://www.ehcache.org/schema/ehcache-107-ext-3.0.xsd"> (1)

  <service> (2)
    <jsr107:defaults default-template="tinyCache"> (3)
      <jsr107:cache name="foos" template="clientCache"/> (4)
      <jsr107:cache name="byRefCache" template="byRefTemplate"/>
      <jsr107:cache name="byValCache" template="byValueTemplate"/>
      <jsr107:cache name="weirdCache1" template="mixedTemplate1"/>
      <jsr107:cache name="weirdCache2" template="mixedTemplate2"/>
    </jsr107:defaults>
  </service>

  <cache-template name="clientCache">
    <key-type>java.lang.String</key-type>
    <value-type>com.pany.domain.Client</value-type>
    <expiry>
      <ttl unit="minutes">2</ttl>
    </expiry>
    <heap unit="entries">2000</heap>
  </cache-template>

  <cache-template name="tinyCache">
    <heap unit="entries">20</heap>
  </cache-template>

  <cache-template name="byRefTemplate">
    <key-type copier="org.ehcache.impl.copy.IdentityCopier">java.lang.Long</key-type>
    <value-type copier="org.ehcache.impl.copy.IdentityCopier">com.pany.domain.Client</value-type>
    <heap unit="entries">10</heap>
  </cache-template>

  <cache-template name="byValueTemplate">
    <key-type copier="org.ehcache.impl.copy.SerializingCopier">java.lang.Long</key-type>
    <value-type copier="org.ehcache.impl.copy.SerializingCopier">com.pany.domain.Client</value-type>
    <heap unit="entries">10</heap>
  </cache-template>

  <cache-template name="mixedTemplate1">
    <key-type copier="org.ehcache.impl.copy.IdentityCopier">java.lang.Long</key-type>
    <value-type copier="org.ehcache.impl.copy.SerializingCopier">com.pany.domain.Client</value-type>
    <heap unit="entries">10</heap>
  </cache-template>

  <cache-template name="mixedTemplate2">
    <key-type copier="org.ehcache.impl.copy.SerializingCopier">java.lang.Long</key-type>
    <value-type copier="org.ehcache.impl.copy.IdentityCopier">com.pany.domain.Client</value-type>
    <heap unit="entries">10</heap>
  </cache-template>
</config>
```
- 首先，为JCache扩展声明一个命名空间，例如jsr107
- 在配置顶部的服务元素中，添加jsr107:defaults元素
- 元素接受一个可选属性default-template，该属性引用所有javax.cache的cache-template。使用javax.cache.CacheManager.createCache在运行时缓存应用程序创建的元素。在本例中，使用的缺省Cache -template将是tinyCache，这意味着除了它们的特定配置之外，任何通过编程创建的缓存实例的容量都将限制在20个条目。
- 嵌套在jsr107:defaults元素中，为给定名字的缓存添加指定的Cache -templates。例如，在运行时创建名为foos的缓存，Ehcache将增强其配置，使其容量为2000个条目，并确保键和值类型都是字符串。

使用上面的配置，您不仅可以补充而且可以覆盖jcache创建的缓存的配置，而无需修改应用程序代码
```
MutableConfiguration<Long, Client> mutableConfiguration = new MutableConfiguration<>();
mutableConfiguration.setTypes(Long.class, Client.class); (1)

Cache<Long, Client> anyCache = manager.createCache("anyCache", mutableConfiguration); (2)

CacheRuntimeConfiguration<Long, Client> ehcacheConfig = (CacheRuntimeConfiguration<Long, Client>)anyCache.getConfiguration(
    Eh107Configuration.class).unwrap(CacheRuntimeConfiguration.class); (3)
ehcacheConfig.getResourcePools().getPoolForResource(ResourceType.Core.HEAP).getSize(); (4)

Cache<Long, Client> anotherCache = manager.createCache("byRefCache", mutableConfiguration);
assertFalse(anotherCache.getConfiguration(Configuration.class).isStoreByValue()); (5)

MutableConfiguration<String, Client> otherConfiguration = new MutableConfiguration<>();
otherConfiguration.setTypes(String.class, Client.class);
otherConfiguration.setExpiryPolicyFactory(CreatedExpiryPolicy.factoryOf(Duration.ONE_MINUTE)); (6)

Cache<String, Client> foosCache = manager.createCache("foos", otherConfiguration);(7)
CacheRuntimeConfiguration<Long, Client> foosEhcacheConfig = (CacheRuntimeConfiguration<Long, Client>)foosCache.getConfiguration(
    Eh107Configuration.class).unwrap(CacheRuntimeConfiguration.class);
Client client1 = new Client("client1", 1);
foosEhcacheConfig.getExpiryPolicy().getExpiryForCreation(42L, client1).toMinutes(); (8)

CompleteConfiguration<String, String> foosConfig = foosCache.getConfiguration(CompleteConfiguration.class);

try {
  final Factory<ExpiryPolicy> expiryPolicyFactory = foosConfig.getExpiryPolicyFactory();
  ExpiryPolicy expiryPolicy = expiryPolicyFactory.create(); (9)
  throw new AssertionError("Expected UnsupportedOperationException");
} catch (UnsupportedOperationException e) {
  // Expected
}
```
- 假设现有的JCache配置代码，默认情况下是按值存储的
- 创建JCache缓存
- 如果您要访问Ehcache RuntimeConfiguration
- 您可以验证配置的模板容量是否应用于缓存，并在这里返回20
- 缓存模板将覆盖JCache的按值存储配置到按引用存储，因为用于创建缓存的byRefTemplate是使用IdentityCopier显式配置的。
- 模板还将覆盖JCache配置，在本例中使用的配置是Time to Live (TTL) 1分钟
- 创建一个缓存，其中模板将TTL设置为2分钟
- 我们确实可以验证模板中提供的配置是否已被应用;持续时间为2分钟，而不是1分钟
- 这样做的一个缺点是，当获得CompleteConfiguration时，您不再能够从JCache访问工厂

Ehcache和通过JCache使用的Ehcache默认行为的差异

使用Ehcache和通过JCache使用的Ehcache在默认行为上并不总是一致的。虽然Ehcache可以按照JCache指定的方式运行，但这取决于使用的配置机制，您可能会看到缺省值的差异。

按引用调用或传递的

Ehcache和Ehcache通过JCache在仅支持堆缓存的默认模式上存在分歧。

使用JCache配置Ehcache

除非您调用mutableconimage.setstorebyvalue (boolean)，默认值为true。这意味着在使用Ehcache时，只能使用可序列化的键和值。

这将触发使用serializing copiers ，并从默认的序列化器中选择适当的序列化器。

使用本地XML或代码配置Ehcache

Heap only: 当仅使用堆缓存时，默认值为by-reference，除非配置了Copier。

其他分层配置: 当使用任何其他层时，由于序列化起作用，默认是按值进行的。

有关信息，请参阅序列化器和复印机部分。

Cache-through和比较交换操作

Ehcache和Ehcache通过JCache对缓存加载器在比较和交换操作中的作用存在不同

通过JCache使用Ehcache的行为

当使用比较和交换操作(如putIfAbsent(K, V))时，如果缓存没有映射，则不会使用缓存加载器。如果putIfAbsent(K, V)成功，那么缓存写入器将用于将更新传播到记录系统。这可能导致缓存的行为类似于INSERT，但实际上会导致底层记录系统的盲目更新。

使用Ehcache的行为

CacheLoaderWriter将始终用于加载缺少的映射并写更新。这使得cache-through中的putIfAbsent(K, V)可以作为记录系统上的INSERT操作。

如果您需要通过JCache行为使用Ehcache，下面显示了相关的XML配置:
```
<service>
  <jsr107:defaults jsr-107-compliant-atomics="true"/>
</service>
```








