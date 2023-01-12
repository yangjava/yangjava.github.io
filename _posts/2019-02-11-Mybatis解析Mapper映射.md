---
layout: post
categories: Mybatis
description: none
keywords: Mybatis
---


# [Mybaits 源码解析 （三）----- Mapper映射的解析过程](https://www.cnblogs.com/java-chen-hao/p/11743442.html)

**正文**

上一篇我们讲解到mapperElement方法用来解析mapper，我们这篇文章具体来看看mapper.xml的解析过程

## mappers配置方式

mappers 标签下有许多 mapper 标签，每一个 mapper 标签中配置的都是一个独立的映射配置文件的路径，配置方式有以下几种。



### 接口信息进行配置



```
<mappers>
    <mapper class="org.mybatis.mappers.UserMapper"/>
    <mapper class="org.mybatis.mappers.ProductMapper"/>
    <mapper class="org.mybatis.mappers.ManagerMapper"/>
</mappers>
```

**注意：**这种方式必须保证接口名（例如UserMapper）和xml名（UserMapper.xml）相同，还必须在同一个包中。因为是通过获取mapper中的class属性，拼接上.xml来读取UserMapper.xml，如果xml文件名不同或者不在同一个包中是无法读取到xml的。



### 相对路径进行配置



```
<mappers>
    <mapper resource="org/mybatis/mappers/UserMapper.xml"/>
    <mapper resource="org/mybatis/mappers/ProductMapper.xml"/>
    <mapper resource="org/mybatis/mappers/ManagerMapper.xml"/>
</mappers>
```



**注意：**这种方式不用保证同接口同包同名。但是要保证xml中的namespase和对应的接口名相同。



### 绝对路径进行配置



```
<mappers>
    <mapper url="file:///var/mappers/UserMapper.xml"/>
    <mapper url="file:///var/mappers/ProductMapper.xml"/>
    <mapper url="file:///var/mappers/ManagerMapper.xml"/>
</mappers>
```



### 接口所在包进行配置

```
<mappers>
    <package name="org.mybatis.mappers"/>
</mappers>
```

这种方式和第一种方式要求一致，保证接口名（例如UserMapper）和xml名（UserMapper.xml）相同，还必须在同一个包中。

**注意：**以上所有的配置都要保证xml中的namespase和对应的接口名相同。

我们以packae属性为例详细分析一下:

## mappers解析入口方法

接上一篇文章最后部分，我们来看看mapperElement方法：

```
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            //包扫描的形式
            if ("package".equals(child.getName())) {
                // 获取 <package> 节点中的 name 属性
                String mapperPackage = child.getStringAttribute("name");
                // 从指定包中查找 所有的 mapper 接口，并根据 mapper 接口解析映射配置
                configuration.addMappers(mapperPackage);
            } else {
                // 获取 resource/url/class 等属性
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");

                // resource 不为空，且其他两者为空，则从指定路径中加载配置
                if (resource != null && url == null && mapperClass == null) {
                    ErrorContext.instance().resource(resource);
                    InputStream inputStream = Resources.getResourceAsStream(resource);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
                    // 解析映射文件
                    mapperParser.parse();
                // url 不为空，且其他两者为空，则通过 url 加载配置
                } else if (resource == null && url != null && mapperClass == null) {
                    ErrorContext.instance().resource(url);
                    InputStream inputStream = Resources.getUrlAsStream(url);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
                    // 解析映射文件
                    mapperParser.parse();
                // mapperClass 不为空，且其他两者为空，则通过 mapperClass 解析映射配置
                } else if (resource == null && url == null && mapperClass != null) {
                    Class<?> mapperInterface = Resources.classForName(mapperClass);
                    configuration.addMapper(mapperInterface);
                } else {
                    throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
                }
            }
        }
    }
}
```



在 MyBatis 中，共有四种加载映射文件或信息的方式。第一种是从文件系统中加载映射文件；第二种是通过 URL 的方式加载和解析映射文件；第三种是通过 mapper 接口加载映射信息，映射信息可以配置在注解中，也可以配置在映射文件中。最后一种是通过包扫描的方式获取到某个包下的所有类，并使用第三种方式为每个类解析映射信息。

我们先看下以packae扫描的形式，看下configuration.addMappers(mapperPackage)方法

```
public void addMappers(String packageName) {
    mapperRegistry.addMappers(packageName);
}
```

我们看一下MapperRegistry的addMappers方法:



```
 1 public void addMappers(String packageName) {
 2     //传入包名和Object.class类型
 3     this.addMappers(packageName, Object.class);
 4 }
 5 
 6 public void addMappers(String packageName, Class<?> superType) {
 7     ResolverUtil<Class<?>> resolverUtil = new ResolverUtil();
 8     /*
 9      * 查找包下的父类为 Object.class 的类。
10      * 查找完成后，查找结果将会被缓存到resolverUtil的内部集合中。上一篇文章我们已经看过这部分的源码，不再累述了
11      */ 
12     resolverUtil.find(new IsA(superType), packageName);
13     // 获取查找结果
14     Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
15     Iterator i$ = mapperSet.iterator();
16 
17     while(i$.hasNext()) {
18         Class<?> mapperClass = (Class)i$.next();
19         //我们具体看这个方法
20         this.addMapper(mapperClass);
21     }
22 
23 }
```



其实就是通过 VFS（虚拟文件系统）获取指定包下的所有文件的Class，也就是所有的Mapper接口，然后遍历每个Mapper接口进行解析，接下来就和第一种配置方式（接口信息进行配置）一样的流程了，接下来我们来看看 基于 XML 的映射文件的解析过程，可以看到先创建一个XMLMapperBuilder，再调用其parse()方法：



```
 1 public void parse() {
 2     // 检测映射文件是否已经被解析过
 3     if (!configuration.isResourceLoaded(resource)) {
 4         // 解析 mapper 节点
 5         configurationElement(parser.evalNode("/mapper"));
 6         // 添加资源路径到“已解析资源集合”中
 7         configuration.addLoadedResource(resource);
 8         // 通过命名空间绑定 Mapper 接口
 9         bindMapperForNamespace();
10     }
11 
12     parsePendingResultMaps();
13     parsePendingCacheRefs();
14     parsePendingStatements();
15 }
```



我们重点关注第5行和第9行的逻辑，也就是**configurationElement和****bindMapperForNamespace方法**



## 解析映射文件

在 MyBatis 映射文件中，可以配置多种节点。比如 <cache>，<resultMap>，<sql> 以及 <select | insert | update | delete> 等。下面我们来看一个映射文件配置示例。



```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="mapper.EmployeeMapper">
    <cache/>
    
    <resultMap id="baseMap" type="entity.Employee">
        <result property="id" column="id" jdbcType="INTEGER"></result>
        <result property="name" column="name" jdbcType="VARCHAR"></result>
    </resultMap>
    
    <sql id="table">
        employee
    </sql>
    
    <select id="getAll" resultMap="baseMap">
        select * from  <include refid="table"/>  WHERE id = #{id}
    </select>
    
    <!-- <insert|update|delete/> -->
</mapper>
```



接着来看看configurationElement解析mapper.xml中的内容。



```
 1 private void configurationElement(XNode context) {
 2     try {
 3         // 获取 mapper 命名空间，如 mapper.EmployeeMapper
 4         String namespace = context.getStringAttribute("namespace");
 5         if (namespace == null || namespace.equals("")) {
 6             throw new BuilderException("Mapper's namespace cannot be empty");
 7         }
 8 
 9         // 设置命名空间到 builderAssistant 中
10         builderAssistant.setCurrentNamespace(namespace);
11 
12         // 解析 <cache-ref> 节点
13         cacheRefElement(context.evalNode("cache-ref"));
14 
15         // 解析 <cache> 节点
16         cacheElement(context.evalNode("cache"));
17 
18         // 已废弃配置，这里不做分析
19         parameterMapElement(context.evalNodes("/mapper/parameterMap"));
20 
21         // 解析 <resultMap> 节点
22         resultMapElements(context.evalNodes("/mapper/resultMap"));
23 
24         // 解析 <sql> 节点
25         sqlElement(context.evalNodes("/mapper/sql"));
26 
27         // 解析 <select>、<insert>、<update>、<delete> 节点
28         buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
29     } catch (Exception e) {
30         throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
31     }
32 }
```



接下来我们就对其中关键的方法进行详细分析



### 解析 cache 节点

MyBatis 提供了一、二级缓存，其中一级缓存是 SqlSession 级别的，默认为开启状态。二级缓存配置在映射文件中，使用者需要显示配置才能开启。如下：

```
<cache/>
```

也可以使用第三方缓存

```
<cache type="org.mybatis.caches.redis.RedisCache"/>
```

其中有一些属性可以选择

```
<cache eviction="LRU"  flushInterval="60000"  size="512" readOnly="true"/>
```

1. 根据数据的历史访问记录来进行淘汰数据，其核心思想是“如果数据最近被访问过，那么将来被访问的几率也更高”
2. 缓存的容量为 512 个对象引用
3. 缓存每隔60秒刷新一次
4. 缓存返回的对象是写安全的，即在外部修改对象不会影响到缓存内部存储对象

下面我们来分析一下缓存配置的解析逻辑，如下：

```
private void cacheElement(XNode context) throws Exception {
    if (context != null) {
        // 获取type属性，如果type没有指定就用默认的PERPETUAL(早已经注册过的别名的PerpetualCache)
        String type = context.getStringAttribute("type", "PERPETUAL");
        // 根据type从早已经注册的别名中获取对应的Class,PERPETUAL对应的Class是PerpetualCache.class
        // 如果我们写了type属性，如type="org.mybatis.caches.redis.RedisCache"，这里将会得到RedisCache.class
        Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
        //获取淘汰方式，默认为LRU(早已经注册过的别名的LruCache)，最近最少使用到的先淘汰
        String eviction = context.getStringAttribute("eviction", "LRU");
        Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
        Long flushInterval = context.getLongAttribute("flushInterval");
        Integer size = context.getIntAttribute("size");
        boolean readWrite = !context.getBooleanAttribute("readOnly", false);
        boolean blocking = context.getBooleanAttribute("blocking", false);

        // 获取子节点配置
        Properties props = context.getChildrenAsProperties();

        // 构建缓存对象
        builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
}

public <T> Class<T> resolveAlias(String string) {
    try {
        if (string == null) {
            return null;
        } else {
            // 转换成小写
            String key = string.toLowerCase(Locale.ENGLISH);
            Class value;
            // 如果没有设置type属性，则这里传过来的是PERPETUAL，能从别名缓存中获取到PerpetualCache.class
            if (this.TYPE_ALIASES.containsKey(key)) {
                value = (Class)this.TYPE_ALIASES.get(key);
            } else {
                //如果是设置了自定义的type，则在别名缓存中是获取不到的，直接通过类加载，加载自定义的type，如RedisCache.class
                value = Resources.classForName(string);
            }

            return value;
        }
    } catch (ClassNotFoundException var4) {
        throw new TypeException("Could not resolve type alias '" + string + "'.  Cause: " + var4, var4);
    }
}
```



缓存的构建封装在 BuilderAssistant 类的 useNewCache 方法中,我们来看看

```
public Cache useNewCache(Class<? extends Cache> typeClass,
    Class<? extends Cache> evictionClass,Long flushInterval,
    Integer size,boolean readWrite,boolean blocking,Properties props) {

    // 使用建造模式构建缓存实例
    Cache cache = new CacheBuilder(currentNamespace)
        .implementation(valueOrDefault(typeClass, PerpetualCache.class))
        .addDecorator(valueOrDefault(evictionClass, LruCache.class))
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .blocking(blocking)
        .properties(props)
        .build();

    // 添加缓存到 Configuration 对象中
    configuration.addCache(cache);

    // 设置 currentCache 属性，即当前使用的缓存
    currentCache = cache;
    return cache;
}
```



上面使用了建造模式构建 Cache 实例，我们跟下去看看。



```
public Cache build() {
    // 设置默认的缓存类型（PerpetualCache）和缓存装饰器（LruCache）
    setDefaultImplementations();

    // 通过反射创建缓存
    Cache cache = newBaseCacheInstance(implementation, id);
    setCacheProperties(cache);
    // 仅对内置缓存 PerpetualCache 应用装饰器
    if (PerpetualCache.class.equals(cache.getClass())) {
        // 遍历装饰器集合，应用装饰器
        for (Class<? extends Cache> decorator : decorators) {
            // 通过反射创建装饰器实例
            cache = newCacheDecoratorInstance(decorator, cache);
            // 设置属性值到缓存实例中
            setCacheProperties(cache);
        }
        // 应用标准的装饰器，比如 LoggingCache、SynchronizedCache
        cache = setStandardDecorators(cache);
    } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
        // 应用具有日志功能的缓存装饰器
        cache = new LoggingCache(cache);
    }
    return cache;
}

private void setDefaultImplementations() {
    if (this.implementation == null) {
        //设置默认缓存类型为PerpetualCache
        this.implementation = PerpetualCache.class;
        if (this.decorators.isEmpty()) {
            this.decorators.add(LruCache.class);
        }
    }
}

private Cache newBaseCacheInstance(Class<? extends Cache> cacheClass, String id) {
    //获取构造器
    Constructor cacheConstructor = this.getBaseCacheConstructor(cacheClass);

    try {
        //通过构造器实例化Cache
        return (Cache)cacheConstructor.newInstance(id);
    } catch (Exception var5) {
        throw new CacheException("Could not instantiate cache implementation (" + cacheClass + "). Cause: " + var5, var5);
    }
}
```



如上就创建好了一个Cache的实例，然后把它添加到Configuration中，并且设置到currentCache属性中，这个属性后面还要使用，也就是Cache实例后面还要使用，我们后面再看。



### 解析 resultMap 节点

resultMap 主要用于映射结果。通过 resultMap 和自动映射，可以让 MyBatis 帮助我们完成 ResultSet → Object 的映射。下面开始分析 resultMap 配置的解析过程。



```
private void resultMapElements(List<XNode> list) throws Exception {
    // 遍历 <resultMap> 节点列表
    for (XNode resultMapNode : list) {
        try {
            // 解析 resultMap 节点
            resultMapElement(resultMapNode);
        } catch (IncompleteElementException e) {
        }
    }
}

private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.<ResultMapping>emptyList());
}

private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings) throws Exception {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());

    // 获取 id 和 type 属性
    String id = resultMapNode.getStringAttribute("id", resultMapNode.getValueBasedIdentifier());
    String type = resultMapNode.getStringAttribute("type",
        resultMapNode.getStringAttribute("ofType",
            resultMapNode.getStringAttribute("resultType",
                resultMapNode.getStringAttribute("javaType"))));
    // 获取 extends 和 autoMapping
    String extend = resultMapNode.getStringAttribute("extends");
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");

    // 获取 type 属性对应的类型
    Class<?> typeClass = resolveClass(type);
    Discriminator discriminator = null;
    //创建ResultMapping集合，对应resultMap子节点的id和result节点
    List<ResultMapping> resultMappings = new ArrayList<ResultMapping>();
    resultMappings.addAll(additionalResultMappings);

    // 获取并遍历 <resultMap> 的子节点列表
    List<XNode> resultChildren = resultMapNode.getChildren();
    for (XNode resultChild : resultChildren) {
        if ("constructor".equals(resultChild.getName())) {
            processConstructorElement(resultChild, typeClass, resultMappings);
        } else if ("discriminator".equals(resultChild.getName())) {
            discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
        } else {

            List<ResultFlag> flags = new ArrayList<ResultFlag>();
            if ("id".equals(resultChild.getName())) {
                // 添加 ID 到 flags 集合中
                flags.add(ResultFlag.ID);
            }
            // 解析 id 和 result 节点，将id或result节点生成相应的 ResultMapping，将ResultMapping添加到resultMappings集合中
            resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
        }
    }
    //创建ResultMapResolver对象
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend,
        discriminator, resultMappings, autoMapping);
    try {
        // 根据前面获取到的信息构建 ResultMap 对象
        return resultMapResolver.resolve();
    } catch (IncompleteElementException e) {
        configuration.addIncompleteResultMap(resultMapResolver);
        throw e;
    }
}
```



**解析 id 和 result 节点**

在 <resultMap> 节点中，子节点 <id> 和 <result> 都是常规配置，比较常见。我们来看看解析过程



```
private ResultMapping buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags) throws Exception {
    String property;
    // 根据节点类型获取 name 或 property 属性
    if (flags.contains(ResultFlag.CONSTRUCTOR)) {
        property = context.getStringAttribute("name");
    } else {
        property = context.getStringAttribute("property");
    }

    // 获取其他各种属性
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String nestedSelect = context.getStringAttribute("select");
    
    /*
     * 解析 resultMap 属性，该属性出现在 <association> 和 <collection> 节点中。
     * 若这两个节点不包含 resultMap 属性，则调用 processNestedResultMappings 方法,递归调用resultMapElement解析<association> 和 <collection>的嵌套节点，生成resultMap，并返回resultMap.getId();
     * 如果包含resultMap属性，则直接获取其属性值，这个属性值对应一个resultMap节点
     */
    String nestedResultMap = context.getStringAttribute("resultMap", processNestedResultMappings(context, Collections.<ResultMapping>emptyList()));
    
    String notNullColumn = context.getStringAttribute("notNullColumn");
    String columnPrefix = context.getStringAttribute("columnPrefix");
    String typeHandler = context.getStringAttribute("typeHandler");
    String resultSet = context.getStringAttribute("resultSet");
    String foreignColumn = context.getStringAttribute("foreignColumn");
    boolean lazy = "lazy".equals(context.getStringAttribute("fetchType", configuration.isLazyLoadingEnabled() ? "lazy" : "eager"));

    Class<?> javaTypeClass = resolveClass(javaType);
    Class<? extends TypeHandler<?>> typeHandlerClass = (Class<? extends TypeHandler<?>>) resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);

    // 构建 ResultMapping 对象
    return builderAssistant.buildResultMapping(resultType, property, column, javaTypeClass, jdbcTypeEnum, nestedSelect,
        nestedResultMap, notNullColumn, columnPrefix, typeHandlerClass, flags, resultSet, foreignColumn, lazy);
}
```



看processNestedResultMappings解析<association> 和 <collection> 节点中的子节点，并返回ResultMap.id



```
private String processNestedResultMappings(XNode context, List<ResultMapping> resultMappings) throws Exception {
    if (("association".equals(context.getName()) || "collection".equals(context.getName()) || "case".equals(context.getName())) && context.getStringAttribute("select") == null) {
        ResultMap resultMap = this.resultMapElement(context, resultMappings);
        return resultMap.getId();
    } else {
        return null;
    }
}
```



只要此节点是（association或者collection）并且select为空,就说明是嵌套查询，那如果select不为空呢？那说明是延迟加载此节点的信息，并不属于嵌套查询，但是有可能有多个association或者collection，有一个设置为延迟加载也就是select属性不为空，有一个没有设置延迟加载，那说明resultMap中有嵌套查询的ResultMapping，也有延迟加载的ResultMapping，这个在后面结果集映射时会用到。

下面以 <association> 节点为例，演示该节点的两种配置方式，分别如下：

第一种配置方式是通过 resultMap 属性引用其他的 <resultMap> 节点，配置如下：



```
<resultMap id="articleResult" type="Article">
    <id property="id" column="id"/>
    <result property="title" column="article_title"/>
    <!-- 引用 authorResult，此时为嵌套查询 -->
    <association property="article_author" column="article_author_id" javaType="Author" resultMap="authorResult"/>
    <!-- 引用 authorResult，此时为延迟查询 -->
    <association property="article_author" column="article_author_id" javaType="Author" select="authorResult"/>
</resultMap>

<resultMap id="authorResult" type="Author">
    <id property="id" column="author_id"/>
    <result property="name" column="author_name"/>
</resultMap>
```



第二种配置方式是采取 resultMap 嵌套的方式进行配置，如下：



```
<resultMap id="articleResult" type="Article">
    <id property="id" column="id"/>
    <result property="title" column="article_title"/>
    <!-- resultMap 嵌套 -->
    <association property="article_author" javaType="Author">
        <id property="id" column="author_id"/>
        <result property="name" column="author_name"/>
    </association>
</resultMap>
```



第二种配置，<association> 的子节点是一些结果映射配置，这些结果配置最终也会被解析成 ResultMap。

下面分析 ResultMapping 的构建过程。



```
public ResultMapping buildResultMapping(Class<?> resultType, String property, String column, Class<?> javaType,JdbcType jdbcType, 
    String nestedSelect, String nestedResultMap, String notNullColumn, String columnPrefix,Class<? extends TypeHandler<?>> typeHandler, 
    List<ResultFlag> flags, String resultSet, String foreignColumn, boolean lazy) {

    // resultType：即 <resultMap type="xxx"/> 中的 type 属性
    // property：即 <result property="xxx"/> 中的 property 属性
    Class<?> javaTypeClass = resolveResultJavaType(resultType, property, javaType);

    TypeHandler<?> typeHandlerInstance = resolveTypeHandler(javaTypeClass, typeHandler);

    List<ResultMapping> composites = parseCompositeColumnName(column);

    // 通过建造模式构建 ResultMapping
    return new ResultMapping.Builder(configuration, property, column, javaTypeClass)
        .jdbcType(jdbcType)
        .nestedQueryId(applyCurrentNamespace(nestedSelect, true))
        .nestedResultMapId(applyCurrentNamespace(nestedResultMap, true))
        .resultSet(resultSet)
        .typeHandler(typeHandlerInstance)
        .flags(flags == null ? new ArrayList<ResultFlag>() : flags)
        .composites(composites)
        .notNullColumns(parseMultipleColumnNames(notNullColumn))
        .columnPrefix(columnPrefix)
        .foreignColumn(foreignColumn)
        .lazy(lazy)
        .build();
}

private Class<?> resolveResultJavaType(Class<?> resultType, String property, Class<?> javaType) {
    if (javaType == null && property != null) {
        try {
            //获取ResultMap中的type属性的元类，如<resultMap id="user" type="java.model.User"/> 中User的元类
            MetaClass metaResultType = MetaClass.forClass(resultType, this.configuration.getReflectorFactory());
            //<result property="name" javaType="String"/>,如果result中没有设置javaType，则获取元类属性对那个的类型
            javaType = metaResultType.getSetterType(property);
        } catch (Exception var5) {
            ;
        }
    }

    if (javaType == null) {
        javaType = Object.class;
    }

    return javaType;
}
    
public ResultMapping build() {
    resultMapping.flags = Collections.unmodifiableList(resultMapping.flags);
    resultMapping.composites = Collections.unmodifiableList(resultMapping.composites);
    resolveTypeHandler();
    validate();
    return resultMapping;
}
```



我们来看看ResultMapping类



```
public class ResultMapping {
    private Configuration configuration;
    private String property;
    private String column;
    private Class<?> javaType;
    private JdbcType jdbcType;
    private TypeHandler<?> typeHandler;
    private String nestedResultMapId;
    private String nestedQueryId;
    private Set<String> notNullColumns;
    private String columnPrefix;
    private List<ResultFlag> flags;
    private List<ResultMapping> composites;
    private String resultSet;
    private String foreignColumn;
    private boolean lazy;

    ResultMapping() {
    }
    //略
}
```



我们看到ResultMapping中有属性**nestedResultMapId表示嵌套查询和****nestedQueryId表示延迟查询**

ResultMapping就是和ResultMap中子节点id和result对应

```
<id column="wi_id" jdbcType="INTEGER"  property="id" />
<result column="warrant_no" jdbcType="String"  jdbcType="CHAR" property="warrantNo" />
```

**ResultMap 对象构建**

前面的分析我们知道了<id>，<result> 等节点最终都被解析成了 ResultMapping。并且封装到了**resultMappings集合中，**紧接着要做的事情是构建 ResultMap，关键代码在**resultMapResolver.resolve()：**



```
public ResultMap resolve() {
    return assistant.addResultMap(this.id, this.type, this.extend, this.discriminator, this.resultMappings, this.autoMapping);
}

public ResultMap addResultMap(
    String id, Class<?> type, String extend, Discriminator discriminator,
    List<ResultMapping> resultMappings, Boolean autoMapping) {
    
    // 为 ResultMap 的 id 和 extend 属性值拼接命名空间
    id = applyCurrentNamespace(id, false);
    extend = applyCurrentNamespace(extend, true);

    if (extend != null) {
        if (!configuration.hasResultMap(extend)) {
            throw new IncompleteElementException("Could not find a parent resultmap with id '" + extend + "'");
        }
        ResultMap resultMap = configuration.getResultMap(extend);
        List<ResultMapping> extendedResultMappings = new ArrayList<ResultMapping>(resultMap.getResultMappings());
        extendedResultMappings.removeAll(resultMappings);
        
        boolean declaresConstructor = false;
        for (ResultMapping resultMapping : resultMappings) {
            if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
                declaresConstructor = true;
                break;
            }
        }
        
        if (declaresConstructor) {
            Iterator<ResultMapping> extendedResultMappingsIter = extendedResultMappings.iterator();
            while (extendedResultMappingsIter.hasNext()) {
                if (extendedResultMappingsIter.next().getFlags().contains(ResultFlag.CONSTRUCTOR)) {
                    extendedResultMappingsIter.remove();
                }
            }
        }
        resultMappings.addAll(extendedResultMappings);
    }

    // 构建 ResultMap
    ResultMap resultMap = new ResultMap.Builder(configuration, id, type, resultMappings, autoMapping)
        .discriminator(discriminator)
        .build();
    // 将创建好的ResultMap加入configuration中
    configuration.addResultMap(resultMap);
    return resultMap;
}
```



我们先看看**ResultMap**



```
public class ResultMap {
    private String id;
    private Class<?> type;
    private List<ResultMapping> resultMappings;
    //用于存储 <id> 节点对应的 ResultMapping 对象
    private List<ResultMapping> idResultMappings;
    private List<ResultMapping> constructorResultMappings;
    //用于存储 <id> 和 <result> 节点对应的 ResultMapping 对象
    private List<ResultMapping> propertyResultMappings;
    //用于存储 所有<id>、<result> 节点 column 属性
    private Set<String> mappedColumns;
    private Discriminator discriminator;
    private boolean hasNestedResultMaps;
    private boolean hasNestedQueries;
    private Boolean autoMapping;

    private ResultMap() {
    }
    //略
}
```



再来看看通过建造模式构建 ResultMap 实例



```
public ResultMap build() {
    if (resultMap.id == null) {
        throw new IllegalArgumentException("ResultMaps must have an id");
    }
    resultMap.mappedColumns = new HashSet<String>();
    resultMap.mappedProperties = new HashSet<String>();
    resultMap.idResultMappings = new ArrayList<ResultMapping>();
    resultMap.constructorResultMappings = new ArrayList<ResultMapping>();
    resultMap.propertyResultMappings = new ArrayList<ResultMapping>();
    final List<String> constructorArgNames = new ArrayList<String>();

    for (ResultMapping resultMapping : resultMap.resultMappings) {
        /*
         * resultMapping.getNestedQueryId()不为空，表示当前resultMap是中有需要延迟查询的属性
         * resultMapping.getNestedResultMapId()不为空，表示当前resultMap是一个嵌套查询
         * 有可能当前ResultMapp既是一个嵌套查询，又存在延迟查询的属性
         */
        resultMap.hasNestedQueries = resultMap.hasNestedQueries || resultMapping.getNestedQueryId() != null;
        resultMap.hasNestedResultMaps =  resultMap.hasNestedResultMaps || (resultMapping.getNestedResultMapId() != null && resultMapping.getResultSet() == null);

        final String column = resultMapping.getColumn();
        if (column != null) {
            // 将 colum 转换成大写，并添加到 mappedColumns 集合中
            resultMap.mappedColumns.add(column.toUpperCase(Locale.ENGLISH));
        } else if (resultMapping.isCompositeResult()) {
            for (ResultMapping compositeResultMapping : resultMapping.getComposites()) {
                final String compositeColumn = compositeResultMapping.getColumn();
                if (compositeColumn != null) {
                    resultMap.mappedColumns.add(compositeColumn.toUpperCase(Locale.ENGLISH));
                }
            }
        }

        // 添加属性 property 到 mappedProperties 集合中
        final String property = resultMapping.getProperty();
        if (property != null) {
            resultMap.mappedProperties.add(property);
        }

        if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
            resultMap.constructorResultMappings.add(resultMapping);
            if (resultMapping.getProperty() != null) {
                constructorArgNames.add(resultMapping.getProperty());
            }
        } else {
            // 添加 resultMapping 到 propertyResultMappings 中
            resultMap.propertyResultMappings.add(resultMapping);
        }

        if (resultMapping.getFlags().contains(ResultFlag.ID)) {
            // 添加 resultMapping 到 idResultMappings 中
            resultMap.idResultMappings.add(resultMapping);
        }
    }
    if (resultMap.idResultMappings.isEmpty()) {
        resultMap.idResultMappings.addAll(resultMap.resultMappings);
    }
    if (!constructorArgNames.isEmpty()) {
        final List<String> actualArgNames = argNamesOfMatchingConstructor(constructorArgNames);
        if (actualArgNames == null) {
            throw new BuilderException("Error in result map '" + resultMap.id
                + "'. Failed to find a constructor in '"
                + resultMap.getType().getName() + "' by arg names " + constructorArgNames
                + ". There might be more info in debug log.");
        }
        Collections.sort(resultMap.constructorResultMappings, new Comparator<ResultMapping>() {
            @Override
            public int compare(ResultMapping o1, ResultMapping o2) {
                int paramIdx1 = actualArgNames.indexOf(o1.getProperty());
                int paramIdx2 = actualArgNames.indexOf(o2.getProperty());
                return paramIdx1 - paramIdx2;
            }
        });
    }

    // 将以下这些集合变为不可修改集合
    resultMap.resultMappings = Collections.unmodifiableList(resultMap.resultMappings);
    resultMap.idResultMappings = Collections.unmodifiableList(resultMap.idResultMappings);
    resultMap.constructorResultMappings = Collections.unmodifiableList(resultMap.constructorResultMappings);
    resultMap.propertyResultMappings = Collections.unmodifiableList(resultMap.propertyResultMappings);
    resultMap.mappedColumns = Collections.unmodifiableSet(resultMap.mappedColumns);
    return resultMap;
}
```



主要做的事情就是将 ResultMapping 实例及属性分别存储到不同的集合中。



### 解析 sql 节点

<sql> 节点用来定义一些可重用的 SQL 语句片段，比如表名，或表的列名等。在映射文件中，我们可以通过 <include> 节点引用 <sql> 节点定义的内容。



```
<sql id="table">
    user
</sql>

<select id="findOne" resultType="Article">
    SELECT * FROM <include refid="table"/> WHERE id = #{id}
</select>
```



下面分析一下 sql 节点的解析过程，如下：



```
private void sqlElement(List<XNode> list) throws Exception {
    if (configuration.getDatabaseId() != null) {
        // 调用 sqlElement 解析 <sql> 节点
        sqlElement(list, configuration.getDatabaseId());
    }

    // 再次调用 sqlElement，不同的是，这次调用，该方法的第二个参数为 null
    sqlElement(list, null);
}

private void sqlElement(List<XNode> list, String requiredDatabaseId) throws Exception {
    for (XNode context : list) {
        // 获取 id 和 databaseId 属性
        String databaseId = context.getStringAttribute("databaseId");
        String id = context.getStringAttribute("id");

        // id = currentNamespace + "." + id
        id = builderAssistant.applyCurrentNamespace(id, false);

        // 检测当前 databaseId 和 requiredDatabaseId 是否一致
        if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
            // 将 <id, XNode> 键值对缓存到XMLMapperBuilder对象的 sqlFragments 属性中，以供后面的sql语句使用
            sqlFragments.put(id, context);
        }
    }
}
```





### 解析select|insert|update|delete节点

<select>、<insert>、<update> 以及 <delete> 等节点统称为 SQL 语句节点，其解析过程在buildStatementFromContext方法中：




```
private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
        // 调用重载方法构建 Statement
        buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
}

private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
        // 创建 XMLStatementBuilder 建造类
        final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
        try {
            /*
             * 解析sql节点，将其封装到 Statement 对象中，并将解析结果存储到 configuration 的 mappedStatements 集合中
             */
            statementParser.parseStatementNode();
        } catch (IncompleteElementException e) {
            configuration.addIncompleteStatement(statementParser);
        }
    }
}
```



我们继续看 statementParser.parseStatementNode();



```
public void parseStatementNode() {
    // 获取 id 和 databaseId 属性
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");

    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
        return;
    }

    // 获取各种属性
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultType = context.getStringAttribute("resultType");
    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);

    // 通过别名解析 resultType 对应的类型
    Class<?> resultTypeClass = resolveClass(resultType);
    String resultSetType = context.getStringAttribute("resultSetType");
    
    // 解析 Statement 类型，默认为 PREPARED
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    
    // 解析 ResultSetType
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);

    // 获取节点的名称，比如 <select> 节点名称为 select
    String nodeName = context.getNode().getNodeName();
    // 根据节点名称解析 SqlCommandType
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // 解析 <include> 节点
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());

    processSelectKeyNodes(id, parameterTypeClass, langDriver);

    // 解析 SQL 语句
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    String resultSets = context.getStringAttribute("resultSets");
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");

    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    if (configuration.hasKeyGenerator(keyStatementId)) {
        keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
        keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
            configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType)) ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    }

    /*
     * 构建 MappedStatement 对象，并将该对象存储到 Configuration 的 mappedStatements 集合中
     */
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered,
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
}
```

我们主要来分析下面几个重要的方法：

1. 解析 <include> 节点
2. 解析 SQL，获取 SqlSource
3. 构建 MappedStatement 实例

**解析 <include> 节点**

先来看一个include的例子



```
<mapper namespace="java.mybaits.dao.UserMapper">
    <sql id="table">
        user
    </sql>

    <select id="findOne" resultType="User">
        SELECT  * FROM  <include refid="table"/> WHERE id = #{id}
    </select>
</mapper>
```



<include> 节点的解析逻辑封装在 applyIncludes 中，该方法的代码如下：



```
public void applyIncludes(Node source) {
    Properties variablesContext = new Properties();
    Properties configurationVariables = configuration.getVariables();
    if (configurationVariables != null) {
        // 将 configurationVariables 中的数据添加到 variablesContext 中
        variablesContext.putAll(configurationVariables);
    }

    // 调用重载方法处理 <include> 节点
    applyIncludes(source, variablesContext, false);
}
```



继续看 applyIncludes 方法



```
private void applyIncludes(Node source, final Properties variablesContext, boolean included) {

    // 第一个条件分支
    if (source.getNodeName().equals("include")) {

        //获取 <sql> 节点。
        Node toInclude = findSqlFragment(getStringAttribute(source, "refid"), variablesContext);

        Properties toIncludeContext = getVariablesContext(source, variablesContext);

        applyIncludes(toInclude, toIncludeContext, true);

        if (toInclude.getOwnerDocument() != source.getOwnerDocument()) {
            toInclude = source.getOwnerDocument().importNode(toInclude, true);
        }
        // 将 <select>节点中的 <include> 节点替换为 <sql> 节点
        source.getParentNode().replaceChild(toInclude, source);
        while (toInclude.hasChildNodes()) {
            // 将 <sql> 中的内容插入到 <sql> 节点之前
            toInclude.getParentNode().insertBefore(toInclude.getFirstChild(), toInclude);
        }

        /*
         * 前面已经将 <sql> 节点的内容插入到 dom 中了，
         * 现在不需要 <sql> 节点了，这里将该节点从 dom 中移除
         */
        toInclude.getParentNode().removeChild(toInclude);

    // 第二个条件分支
    } else if (source.getNodeType() == Node.ELEMENT_NODE) {
        if (included && !variablesContext.isEmpty()) {
            NamedNodeMap attributes = source.getAttributes();
            for (int i = 0; i < attributes.getLength(); i++) {
                Node attr = attributes.item(i);
                // 将 source 节点属性中的占位符 ${} 替换成具体的属性值
                attr.setNodeValue(PropertyParser.parse(attr.getNodeValue(), variablesContext));
            }
        }
        
        NodeList children = source.getChildNodes();
        for (int i = 0; i < children.getLength(); i++) {
            // 递归调用
            applyIncludes(children.item(i), variablesContext, included);
        }
        
    // 第三个条件分支
    } else if (included && source.getNodeType() == Node.TEXT_NODE && !variablesContext.isEmpty()) {
        // 将文本（text）节点中的属性占位符 ${} 替换成具体的属性值
        source.setNodeValue(PropertyParser.parse(source.getNodeValue(), variablesContext));
    }
}
```



我们先来看一下 applyIncludes 方法第一次被调用时的状态，source为<select> 节点，节点类型：ELEMENT_NODE，此时会进入第二个分支，获取到获取 <select> 子节点列表，遍历子节点列表，将子节点作为参数，进行递归调用applyIncludes ，此时可获取到的子节点如下：

| 编号 | 子节点                   | 类型         | 描述     |
| ---- | ------------------------ | ------------ | -------- |
| 1    | SELECT * FROM            | TEXT_NODE    | 文本节点 |
| 2    | <include refid="table"/> | ELEMENT_NODE | 普通节点 |
| 3    | WHERE id = #{id}         | TEXT_NODE    | 文本节点 |

接下来要做的事情是遍历列表，然后将子节点作为参数进行递归调用。第一个子节点调用applyIncludes方法，source为 SELECT * FROM 节点，节点类型：TEXT_NODE，进入分支三，没有${}，不会替换，则节点一结束返回，什么都没有做。第二个节点调用applyIncludes方法，此时source为 <include refid="table"/>节点，节点类型：ELEMENT_NODE，进入分支一，通过refid找到 sql 节点，也就是toInclude节点，然后执行source.getParentNode().replaceChild(toInclude, source);，直接将<include refid="table"/>节点的父节点，也就是<select> 节点中的当前<include >节点替换成 <sql> 节点，然后调用toInclude.getParentNode().insertBefore(toInclude.getFirstChild(), toInclude);，将 <sql> 中的内容插入到 <sql> 节点之前，也就是将user插入到 <sql> 节点之前，现在不需要 <sql> 节点了，最后将该节点从 dom 中移除

**创建SqlSource**

创建SqlSource在createSqlSource方法中



```
public SqlSource createSqlSource(Configuration configuration, XNode script, Class<?> parameterType) {
    XMLScriptBuilder builder = new XMLScriptBuilder(configuration, script, parameterType);
    return builder.parseScriptNode();
}

// -☆- XMLScriptBuilder
public SqlSource parseScriptNode() {
    // 解析 SQL 语句节点
    MixedSqlNode rootSqlNode = parseDynamicTags(context);
    SqlSource sqlSource = null;
    // 根据 isDynamic 状态创建不同的 SqlSource
    if (isDynamic) {
        sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    } else {
        sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
    }
    return sqlSource;
}
```



继续跟进parseDynamicTags



```
/** 该方法用于初始化 nodeHandlerMap 集合，该集合后面会用到 */
private void initNodeHandlerMap() {
    nodeHandlerMap.put("trim", new TrimHandler());
    nodeHandlerMap.put("where", new WhereHandler());
    nodeHandlerMap.put("set", new SetHandler());
    nodeHandlerMap.put("foreach", new ForEachHandler());
    nodeHandlerMap.put("if", new IfHandler());
    nodeHandlerMap.put("choose", new ChooseHandler());
    nodeHandlerMap.put("when", new IfHandler());
    nodeHandlerMap.put("otherwise", new OtherwiseHandler());
    nodeHandlerMap.put("bind", new BindHandler());
}
    
protected MixedSqlNode parseDynamicTags(XNode node) {
    List<SqlNode> contents = new ArrayList<SqlNode>();
    NodeList children = node.getNode().getChildNodes();
    // 遍历子节点
    for (int i = 0; i < children.getLength(); i++) {
        XNode child = node.newXNode(children.item(i));
        //如果节点是TEXT_NODE类型
        if (child.getNode().getNodeType() == Node.CDATA_SECTION_NODE || child.getNode().getNodeType() == Node.TEXT_NODE) {
            // 获取文本内容
            String data = child.getStringBody("");
            TextSqlNode textSqlNode = new TextSqlNode(data);
            // 若文本中包含 ${} 占位符，会被认为是动态节点
            if (textSqlNode.isDynamic()) {
                contents.add(textSqlNode);
                // 设置 isDynamic 为 true
                isDynamic = true;
            } else {
                // 创建 StaticTextSqlNode
                contents.add(new StaticTextSqlNode(data));
            }

        // child 节点是 ELEMENT_NODE 类型，比如 <if>、<where> 等
        } else if (child.getNode().getNodeType() == Node.ELEMENT_NODE) {
            // 获取节点名称，比如 if、where、trim 等
            String nodeName = child.getNode().getNodeName();
            // 根据节点名称获取 NodeHandler，也就是上面注册的nodeHandlerMap
            NodeHandler handler = nodeHandlerMap.get(nodeName);
            if (handler == null) {
                throw new BuilderException("Unknown element <" + nodeName + "> in SQL statement.");
            }
            // 处理 child 节点，生成相应的 SqlNode
            handler.handleNode(child, contents);

            // 设置 isDynamic 为 true
            isDynamic = true;
        }
    }
    return new MixedSqlNode(contents);
}
```



对于if、trim、where等这些动态节点，是通过对应的handler来解析的，如下

```
handler.handleNode(child, contents);
```

该代码用于处理动态 SQL 节点，并生成相应的 SqlNode。下面来简单分析一下 WhereHandler 的代码。



```
/** 定义在 XMLScriptBuilder 中 */
private class WhereHandler implements NodeHandler {

    public WhereHandler() {
    }

    @Override
    public void handleNode(XNode nodeToHandle, List<SqlNode> targetContents) {
        // 调用 parseDynamicTags 解析 <where> 节点
        MixedSqlNode mixedSqlNode = parseDynamicTags(nodeToHandle);
        // 创建 WhereSqlNode
        WhereSqlNode where = new WhereSqlNode(configuration, mixedSqlNode);
        // 添加到 targetContents
        targetContents.add(where);
    }
}
```



我们已经将 XML 配置解析了 SqlSource，下面我们看看MappedStatement的构建。

**构建MappedStatement**

SQL 语句节点可以定义很多属性，这些属性和属性值最终存储在 MappedStatement 中。



```
public MappedStatement addMappedStatement(
    String id, SqlSource sqlSource, StatementType statementType, 
    SqlCommandType sqlCommandType,Integer fetchSize, Integer timeout, 
    String parameterMap, Class<?> parameterType,String resultMap, 
    Class<?> resultType, ResultSetType resultSetType, boolean flushCache,
    boolean useCache, boolean resultOrdered, KeyGenerator keyGenerator, 
    String keyProperty,String keyColumn, String databaseId, 
    LanguageDriver lang, String resultSets) {

    if (unresolvedCacheRef) {
        throw new IncompleteElementException("Cache-ref not yet resolved");
    }
　　// 拼接上命名空间，如 <select id="findOne" resultType="User">，则id=java.mybaits.dao.UserMapper.findOne
    id = applyCurrentNamespace(id, false);
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;

    // 创建建造器，设置各种属性
    MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
        .resource(resource).fetchSize(fetchSize).timeout(timeout)
        .statementType(statementType).keyGenerator(keyGenerator)
        .keyProperty(keyProperty).keyColumn(keyColumn).databaseId(databaseId)
        .lang(lang).resultOrdered(resultOrdered).resultSets(resultSets)
        .resultMaps(getStatementResultMaps(resultMap, resultType, id))
        .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
        .resultSetType(resultSetType).useCache(valueOrDefault(useCache, isSelect))
        .cache(currentCache);//这里用到了前面解析<cache>节点时创建的Cache对象，设置到MappedStatement对象里面的cache属性中

    // 获取或创建 ParameterMap
    ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
    if (statementParameterMap != null) {
        statementBuilder.parameterMap(statementParameterMap);
    }

    // 构建 MappedStatement
    MappedStatement statement = statementBuilder.build();
    // 添加 MappedStatement 到 configuration 的 mappedStatements 集合中
    // 通过UserMapper代理对象调用findOne方法时，就可以拼接UserMapper接口名java.mybaits.dao.UserMapper和findOne方法找到id=java.mybaits.dao.UserMapper的MappedStatement，然后执行对应的sql语句
    configuration.addMappedStatement(statement);
    return statement;
}
```



这里我们要注意，MappedStatement对象中有一个cache属性，将前面解析<cache>节点时创建的Cache对象，设置到MappedStatement对象里面的cache属性中，以备后面二级缓存使用，我们后面专门来讲这一块。

我们还要注意一个地方，.resultMaps(getStatementResultMaps(resultMap, resultType, id))，设置MappedStatement的resultMaps，我们来看看是怎么获取resultMap的



```
private List<ResultMap> getStatementResultMaps(String resultMap, Class<?> resultType, String statementId) {
    //拼接上当前nameSpace
    resultMap = this.applyCurrentNamespace(resultMap, true);
    //创建一个集合
    List<ResultMap> resultMaps = new ArrayList();
    if (resultMap != null) {
        //通过,分隔字符串，一般resultMap只会是一个，不会使用逗号
        String[] resultMapNames = resultMap.split(",");
        String[] arr$ = resultMapNames;
        int len$ = resultMapNames.length;

        for(int i$ = 0; i$ < len$; ++i$) {
            String resultMapName = arr$[i$];

            try {
                //从configuration中通过resultMapName获取ResultMap对象加入到resultMaps中
                resultMaps.add(this.configuration.getResultMap(resultMapName.trim()));
            } catch (IllegalArgumentException var11) {
                throw new IncompleteElementException("Could not find result map " + resultMapName, var11);
            }
        }
    } else if (resultType != null) {
        ResultMap inlineResultMap = (new org.apache.ibatis.mapping.ResultMap.Builder(this.configuration, statementId + "-Inline", resultType, new ArrayList(), (Boolean)null)).build();
        resultMaps.add(inlineResultMap);
    }

    return resultMaps;
}
```



从configuration中获取到ResultMap并设置到MappedStatement中，当查询结束后，就可以拿到ResultMap进行结果映射，这个在后面讲



### Mapper 接口绑定

映射文件解析完成后，我们需要通过命名空间将绑定 mapper 接口，看看具体绑定的啥



```
private void bindMapperForNamespace() {
    // 获取映射文件的命名空间
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
        Class<?> boundType = null;
        try {
            // 根据命名空间解析 mapper 类型
            boundType = Resources.classForName(namespace);
        } catch (ClassNotFoundException e) {
        }
        if (boundType != null) {
            // 检测当前 mapper 类是否被绑定过
            if (!configuration.hasMapper(boundType)) {
                configuration.addLoadedResource("namespace:" + namespace);
                // 绑定 mapper 类
                configuration.addMapper(boundType);
            }
        }
    }
}

// Configuration
public <T> void addMapper(Class<T> type) {
    // 通过 MapperRegistry 绑定 mapper 类
    mapperRegistry.addMapper(type);
}

// MapperRegistry
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
        if (hasMapper(type)) {
            throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
        }
        boolean loadCompleted = false;
        try {
            /*
             * 将 type 和 MapperProxyFactory 进行绑定，MapperProxyFactory 可为 mapper 接口生成代理类
             */
            knownMappers.put(type, new MapperProxyFactory<T>(type));
            
            MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
            // 解析注解中的信息
            parser.parse();
            loadCompleted = true;
        } finally {
            if (!loadCompleted) {
                knownMappers.remove(type);
            }
        }
    }
}
```



其实就是获取当前映射文件的命名空间，并获取其Class，也就是获取每个Mapper接口，然后为每个Mapper接口创建一个代理类工厂，new MapperProxyFactory<T>(type)，并放进 knownMappers 这个HashMap中，我们来看看这个MapperProxyFactory



```
public class MapperProxyFactory<T> {
    //存放Mapper接口Class
    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap();

    public MapperProxyFactory(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    public Class<T> getMapperInterface() {
        return this.mapperInterface;
    }

    public Map<Method, MapperMethod> getMethodCache() {
        return this.methodCache;
    }

    protected T newInstance(MapperProxy<T> mapperProxy) {
        //生成mapperInterface的代理类
        return Proxy.newProxyInstance(this.mapperInterface.getClassLoader(), new Class[]{this.mapperInterface}, mapperProxy);
    }

    public T newInstance(SqlSession sqlSession) {
        MapperProxy<T> mapperProxy = new MapperProxy(sqlSession, this.mapperInterface, this.methodCache);
        return this.newInstance(mapperProxy);
    }
}
```



这一块我们后面文章再来看是如何调用的。

# [Mybaits 源码解析 （四）----- SqlSession的创建过程](https://www.cnblogs.com/java-chen-hao/p/11743506.html)

**正文**

SqlSession是mybatis的核心接口之一，是myabtis接口层的主要组成部分，对外提供了mybatis常用的api。myabtis提供了两个SqlSesion接口的实现，常用的实现类是DefaultSqlSession。它相当于一个数据库连接对象，在一个SqlSession中可以执行多条SQL语句。
