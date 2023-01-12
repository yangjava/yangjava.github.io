---
layout: post
categories: Mybatis
description: none
keywords: Mybatis
---


## 源码

### 源码--MyBatis HelloWorld

源码分析之前先搭一个mybatis的demo，这个在看源码的时候能起到了很大的作用，因为在看源码的时候，会恍然大悟，为什么要这么配置，为什么要这么写。（老鸟可以跳过这篇）

#### 创建maven项目

**pom.xml**

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.mybatis.chenhao</groupId>
    <artifactId>mybatisDemo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <!-- mybatis版本号 -->
        <mybatis.version>3.4.2</mybatis.version>
    </properties>
    <dependencies>

        <!--mybatis依赖 -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>${mybatis.version}</version>
        </dependency>
        <!-- mysql驱动包 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.44</version>
        </dependency>

    </dependencies>

</project>
```

#### 创建mybatis的配置文件

**mybatis-config.xml**

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <!-- 引入外部配置文件 -->
    <properties resource="db.properties"></properties>
    <environments default="default">
        <environment id="default">
            <transactionManager type="JDBC"></transactionManager>
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper class="mapper.DemoMapper"></mapper>
    </mappers>
</configuration>
```

**db.properties**

```
jdbc.driver=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/demo?useUnicode=true&characterEncoding=utf8
jdbc.username=chenhao
jdbc.password=123456
```



#### entity和mapper

**Employee**

```
package entity;

/***
 *
 *@Author ChenHao
 *@Description:
 *@Date: Created in 14:58 2019/10/26
 *@Modified By:
 *
 */
public class Employee {
    int id;
    String name;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Employee{" +
                "id=" + id +
                ", name='" + name + '\'' +
                '}';
    }
}
```

**EmployeeMapper**

```
package mapper;

import entity.Employee;
import java.util.List;

/***
 *
 *@Author ChenHao
 *@Description:
 *@Date: Created in 14:58 2019/10/26
 *@Modified By:
 *
 */
public interface EmployeeMapper {
    List<Employee> getAll();
}
```

**EmployeeMapper.xml**

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="mapper.EmployeeMapper">

    <resultMap id="baseMap" type="entity.Employee">
        <result property="id" column="id" jdbcType="INTEGER"></result>
        <result property="name" column="name" jdbcType="VARCHAR"></result>

    </resultMap>
    <select id="getAll" resultMap="baseMap">
        select * from employee
    </select>
</mapper>
```

#### 测试

```
public static void main(String[] args) throws IOException {
    String resource = "mybatis-config.xml";
    InputStream inputStream = Resources.getResourceAsStream(resource);
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
    SqlSession sqlSession = sqlSessionFactory.openSession();
    try {
         EmployeeMapper employeeMapper = sqlSession.getMapper(EmployeeMapper.class);
         List<Employee> all = employeeMapper.getAll();
         for (Employee item : all)
            System.out.println(item);
    } finally {
        sqlSession.close();
    }
}
```

测试结果：

```
Employee{id=1, name='name1'}
Employee{id=2, name='name2'}
Employee{id=3, name='name3'}
```

好了，MyBatis HelloWorld我们已经搭建完了，后面的源码分析文章我们将以这个为基础来分析

### 源码--Configuration的创建过程

我们使用mybatis操作数据库都是通过SqlSession的API调用，而创建SqlSession是通过SqlSessionFactory。下面我们就看看SqlSessionFactory的创建过程。

## 配置文件解析入口

我们看看第一篇文章中的测试方法

```
 1 public static void main(String[] args) throws IOException {
 2     String resource = "mybatis-config.xml";
 3     InputStream inputStream = Resources.getResourceAsStream(resource);
 4     SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
 5     SqlSession sqlSession = sqlSessionFactory.openSession();
 6     try {
 7         Employee employeeMapper = sqlSession.getMapper(Employee.class);
 8          List<Employee> all = employeeMapper.getAll();
 9          for (Employee item : all)
10             System.out.println(item);
11     } finally {
12         sqlSession.close();
13     }
14 }
```

首先，我们使用 MyBatis 提供的工具类 Resources 加载配置文件，得到一个输入流。然后再通过 SqlSessionFactoryBuilder 对象的`build`方法构建 SqlSessionFactory 对象。所以这里的 build 方法是我们分析配置文件解析过程的入口方法。我们看看build里面是代码：

```
public SqlSessionFactory build(InputStream inputStream) {
    // 调用重载方法
    return build(inputStream, null, null);
}

public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
        // 创建配置文件解析器
        XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
        // 调用 parser.parse() 方法解析配置文件，生成 Configuration 对象
        return build(parser.parse());
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
        ErrorContext.instance().reset();
        try {
        inputStream.close();
        } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
        }
    }
}

public SqlSessionFactory build(Configuration config) {
    // 创建 DefaultSqlSessionFactory，将解析配置文件后生成的Configuration传入
    return new DefaultSqlSessionFactory(config);
}
```

SqlSessionFactory是通过SqlSessionFactoryBuilder的build方法创建的，build方法内部是通过一个XMLConfigBuilder对象解析mybatis-config.xml文件生成一个Configuration对象。XMLConfigBuilder从名字可以看出是解析Mybatis配置文件的，其实它是继承了一个父类BaseBuilder，其每一个子类多是以XMLXXXXXBuilder命名的，也就是其子类都对应解析一种xml文件或xml文件中一种元素。

![img](https://img2018.cnblogs.com/blog/1168971/201910/1168971-20191026165910820-1306401062.png)

我们看看XMLConfigBuilder的构造方法：

```
private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    super(new Configuration());
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = parser;
}
```

可以看到调用了父类的构造方法，并传入一个new Configuration()对象，这个对象也就是最终的Mybatis配置对象

我们先来看看其基类BaseBuilder

```
public abstract class BaseBuilder {
  protected final Configuration configuration;
  protected final TypeAliasRegistry typeAliasRegistry;
  protected final TypeHandlerRegistry typeHandlerRegistry;
 
  public BaseBuilder(Configuration configuration) {
    this.configuration = configuration;
    this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
  }
  ....
}
```

BaseBuilder中只有三个成员变量,而typeAliasRegistry和typeHandlerRegistry都是直接从Configuration的成员变量获得的，接着我们看看Configuration这个类

Configuration类位于mybatis包的org.apache.ibatis.session目录下，其属性就是对应于mybatis的全局配置文件mybatis-config.xml的配置，将XML配置中的内容解析赋值到Configuration对象中。

由于XML配置项有很多，所以Configuration类的属性也很多。先来看下Configuration对于的XML配置文件示例：

```
<?xml version="1.0" encoding="UTF-8"?>    

<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">    
<!-- 全局配置顶级节点 -->
<configuration>    
     
     <!-- 属性配置,读取properties中的配置文件 -->
    <properties resource="db.propertis">
       <property name="XXX" value="XXX"/>
    </properties>
    
    <!-- 类型别名 -->
    <typeAliases>
       <!-- 在用到User类型的时候,可以直接使用别名,不需要输入User类的全部路径 -->
       <typeAlias type="com.luck.codehelp.entity.User" alias="user"/>
    </typeAliases>

    <!-- 类型处理器 -->
    <typeHandlers>
        <!-- 类型处理器的作用是完成JDBC类型和java类型的转换,mybatis默认已经由了很多类型处理器,正常无需自定义-->
    </typeHandlers>
    
    <!-- 对象工厂 -->
    <!-- mybatis创建结果对象的新实例时,会通过对象工厂来完成，mybatis有默认的对象工厂，正常无需配置 -->
    <objectFactory type=""></objectFactory>
    
    <!-- 插件 -->
    <plugins>
        <!-- 可以自定义拦截器通过plugin标签加入 -->
       <plugin interceptor="com.lucky.interceptor.MyPlugin"></plugin>
    </plugins>
    
    <!-- 全局配置参数 -->
    <settings>   
        <setting name="cacheEnabled" value="false" />   
        <setting name="useGeneratedKeys" value="true" /><!-- 是否自动生成主键 -->
        <setting name="defaultExecutorType" value="REUSE" />   
        <setting name="lazyLoadingEnabled" value="true"/><!-- 延迟加载标识 -->
        <setting name="aggressiveLazyLoading" value="true"/><!--有延迟加载属性的对象是否延迟加载 -->
        <setting name="multipleResultSetsEnabled" value="true"/><!-- 是否允许单个语句返回多个结果集 -->
        <setting name="useColumnLabel" value="true"/><!-- 使用列标签而不是列名 -->
        <setting name="autoMappingBehavior" value="PARTIAL"/><!-- 指定mybatis如何自动映射列到字段属性;NONE:自动映射;PARTIAL:只会映射结果没有嵌套的结果;FULL:可以映射任何复杂的结果 -->
        <setting name="defaultExecutorType" value="SIMPLE"/><!-- 默认执行器类型 -->
        <setting name="defaultFetchSize" value=""/>
        <setting name="defaultStatementTimeout" value="5"/><!-- 驱动等待数据库相应的超时时间 ,单位是秒-->
        <setting name="safeRowBoundsEnabled" value="false"/><!-- 是否允许使用嵌套语句RowBounds -->
        <setting name="safeResultHandlerEnabled" value="true"/>
        <setting name="mapUnderscoreToCamelCase" value="false"/><!-- 下划线列名是否自动映射到驼峰属性：如user_id映射到userId -->
        <setting name="localCacheScope" value="SESSION"/><!-- 本地缓存（session是会话级别） -->
        <setting name="jdbcTypeForNull" value="OTHER"/><!-- 数据为空值时,没有特定的JDBC类型的参数的JDBC类型 -->
        <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString"/><!-- 指定触发延迟加载的对象的方法 -->
        <setting name="callSettersOnNulls" value="false"/><!--如果setter方法或map的put方法，如果检索到的值为null时，数据是否有用  -->
        <setting name="logPrefix" value="XXXX"/><!-- mybatis日志文件前缀字符串 -->
        <setting name="logImpl" value="SLF4J"/><!-- mybatis日志的实现类 -->
        <setting name="proxyFactory" value="CGLIB"/><!-- mybatis代理工具 -->
    </settings>  

    <!-- 环境配置集合 -->
    <environments default="development">    
        <environment id="development">    
            <transactionManager type="JDBC"/><!-- 事务管理器 -->
            <dataSource type="POOLED"><!-- 数据库连接池 -->    
                <property name="driver" value="com.mysql.jdbc.Driver" />    
                <property name="url" value="jdbc:mysql://localhost:3306/test?useUnicode=true&amp;characterEncoding=UTF-8" />    
                <property name="username" value="root" />    
                <property name="password" value="root" />    
            </dataSource>    
        </environment>    
    </environments>    
    
    <!-- mapper文件映射配置 -->
    <mappers>    
        <mapper resource="com/luck/codehelp/mapper/UserMapper.xml"/>    
    </mappers>    
</configuration>
```

而对于XML的配置，Configuration类的属性是和XML配置对应的。Configuration类属性如下：

```
public class Configuration {
  protected Environment environment;//运行环境

  protected boolean safeRowBoundsEnabled = false;
  protected boolean safeResultHandlerEnabled = true;
  protected boolean mapUnderscoreToCamelCase = false;
  protected boolean aggressiveLazyLoading = true; //true：有延迟加载属性的对象被调用时完全加载任意属性;false：每个属性按需要加载
  protected boolean multipleResultSetsEnabled = true;//是否允许多种结果集从一个单独的语句中返回
  protected boolean useGeneratedKeys = false;//是否支持自动生成主键
  protected boolean useColumnLabel = true;//是否使用列标签
  protected boolean cacheEnabled = true;//是否使用缓存标识
  protected boolean callSettersOnNulls = false;//
  protected boolean useActualParamName = true;

  protected String logPrefix;
  protected Class <? extends Log> logImpl;
  protected Class <? extends VFS> vfsImpl;
  protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
  protected JdbcType jdbcTypeForNull = JdbcType.OTHER;
  protected Set<String> lazyLoadTriggerMethods = new HashSet<String>(Arrays.asList(new String[] { "equals", "clone", "hashCode", "toString" }));
  protected Integer defaultStatementTimeout;
  protected Integer defaultFetchSize;
  protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
  protected AutoMappingBehavior autoMappingBehavior = AutoMappingBehavior.PARTIAL;//指定mybatis如果自动映射列到字段和属性,PARTIAL会自动映射简单的没有嵌套的结果,FULL会自动映射任意复杂的结果
  protected AutoMappingUnknownColumnBehavior autoMappingUnknownColumnBehavior = AutoMappingUnknownColumnBehavior.NONE;

  protected Properties variables = new Properties();
  protected ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
  protected ObjectFactory objectFactory = new DefaultObjectFactory();
  protected ObjectWrapperFactory objectWrapperFactory = new DefaultObjectWrapperFactory();

  protected boolean lazyLoadingEnabled = false;//是否延时加载，false则表示所有关联对象即使加载,true表示延时加载
  protected ProxyFactory proxyFactory = new JavassistProxyFactory(); // #224 Using internal Javassist instead of OGNL

  protected String databaseId;

  protected Class<?> configurationFactory;

  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
  protected final InterceptorChain interceptorChain = new InterceptorChain();
  protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry();
  protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
  protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();

  protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection");
  protected final Map<String, Cache> caches = new StrictMap<Cache>("Caches collection");
  protected final Map<String, ResultMap> resultMaps = new StrictMap<ResultMap>("Result Maps collection");
  protected final Map<String, ParameterMap> parameterMaps = new StrictMap<ParameterMap>("Parameter Maps collection");
  protected final Map<String, KeyGenerator> keyGenerators = new StrictMap<KeyGenerator>("Key Generators collection");

  protected final Set<String> loadedResources = new HashSet<String>(); //已经加载过的resource(mapper)
  protected final Map<String, XNode> sqlFragments = new StrictMap<XNode>("XML fragments parsed from previous mappers");

  protected final Collection<XMLStatementBuilder> incompleteStatements = new LinkedList<XMLStatementBuilder>();
  protected final Collection<CacheRefResolver> incompleteCacheRefs = new LinkedList<CacheRefResolver>();
  protected final Collection<ResultMapResolver> incompleteResultMaps = new LinkedList<ResultMapResolver>();
  protected final Collection<MethodResolver> incompleteMethods = new LinkedList<MethodResolver>();

  protected final Map<String, String> cacheRefMap = new HashMap<String, String>();
  
  //其他方法略
}
```

加载的过程是SqlSessionFactoryBuilder根据xml配置的文件流，通过XMLConfigBuilder的parse方法进行解析得到一个Configuration对象，我们再看看其构造函数

```
 1 public Configuration() {
 2     this.safeRowBoundsEnabled = false;
 3     this.safeResultHandlerEnabled = true;
 4     this.mapUnderscoreToCamelCase = false;
 5     this.aggressiveLazyLoading = true;
 6     this.multipleResultSetsEnabled = true;
 7     this.useGeneratedKeys = false;
 8     this.useColumnLabel = true;
 9     this.cacheEnabled = true;
10     this.callSettersOnNulls = false;
11     this.localCacheScope = LocalCacheScope.SESSION;
12     this.jdbcTypeForNull = JdbcType.OTHER;
13     this.lazyLoadTriggerMethods = new HashSet(Arrays.asList("equals", "clone", "hashCode", "toString"));
14     this.defaultExecutorType = ExecutorType.SIMPLE;
15     this.autoMappingBehavior = AutoMappingBehavior.PARTIAL;
16     this.autoMappingUnknownColumnBehavior = AutoMappingUnknownColumnBehavior.NONE;
17     this.variables = new Properties();
18     this.reflectorFactory = new DefaultReflectorFactory();
19     this.objectFactory = new DefaultObjectFactory();
20     this.objectWrapperFactory = new DefaultObjectWrapperFactory();
21     this.mapperRegistry = new MapperRegistry(this);
22     this.lazyLoadingEnabled = false;
23     this.proxyFactory = new JavassistProxyFactory();
24     this.interceptorChain = new InterceptorChain();
25     this.typeHandlerRegistry = new TypeHandlerRegistry();
26     this.typeAliasRegistry = new TypeAliasRegistry();
27     this.languageRegistry = new LanguageDriverRegistry();
28     this.mappedStatements = new Configuration.StrictMap("Mapped Statements collection");
29     this.caches = new Configuration.StrictMap("Caches collection");
30     this.resultMaps = new Configuration.StrictMap("Result Maps collection");
31     this.parameterMaps = new Configuration.StrictMap("Parameter Maps collection");
32     this.keyGenerators = new Configuration.StrictMap("Key Generators collection");
33     this.loadedResources = new HashSet();
34     this.sqlFragments = new Configuration.StrictMap("XML fragments parsed from previous mappers");
35     this.incompleteStatements = new LinkedList();
36     this.incompleteCacheRefs = new LinkedList();
37     this.incompleteResultMaps = new LinkedList();
38     this.incompleteMethods = new LinkedList();
39     this.cacheRefMap = new HashMap();
40     this.typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
41     this.typeAliasRegistry.registerAlias("MANAGED", ManagedTransactionFactory.class);
42     this.typeAliasRegistry.registerAlias("JNDI", JndiDataSourceFactory.class);
43     this.typeAliasRegistry.registerAlias("POOLED", PooledDataSourceFactory.class);
44     this.typeAliasRegistry.registerAlias("UNPOOLED", UnpooledDataSourceFactory.class);
45     this.typeAliasRegistry.registerAlias("PERPETUAL", PerpetualCache.class);
46     this.typeAliasRegistry.registerAlias("FIFO", FifoCache.class);
47     this.typeAliasRegistry.registerAlias("LRU", LruCache.class);
48     this.typeAliasRegistry.registerAlias("SOFT", SoftCache.class);
49     this.typeAliasRegistry.registerAlias("WEAK", WeakCache.class);
50     this.typeAliasRegistry.registerAlias("DB_VENDOR", VendorDatabaseIdProvider.class);
51     this.typeAliasRegistry.registerAlias("XML", XMLLanguageDriver.class);
52     this.typeAliasRegistry.registerAlias("RAW", RawLanguageDriver.class);
53     this.typeAliasRegistry.registerAlias("SLF4J", Slf4jImpl.class);
54     this.typeAliasRegistry.registerAlias("COMMONS_LOGGING", JakartaCommonsLoggingImpl.class);
55     this.typeAliasRegistry.registerAlias("LOG4J", Log4jImpl.class);
56     this.typeAliasRegistry.registerAlias("LOG4J2", Log4j2Impl.class);
57     this.typeAliasRegistry.registerAlias("JDK_LOGGING", Jdk14LoggingImpl.class);
58     this.typeAliasRegistry.registerAlias("STDOUT_LOGGING", StdOutImpl.class);
59     this.typeAliasRegistry.registerAlias("NO_LOGGING", NoLoggingImpl.class);
60     this.typeAliasRegistry.registerAlias("CGLIB", CglibProxyFactory.class);
61     this.typeAliasRegistry.registerAlias("JAVASSIST", JavassistProxyFactory.class);
62     this.languageRegistry.setDefaultDriverClass(XMLLanguageDriver.class);
63     this.languageRegistry.register(RawLanguageDriver.class);
64 }
```

我们看到第26行**this.typeAliasRegistry = new TypeAliasRegistry();，并且第40到61行向** **typeAliasRegistry 注册了很多别名，我们看看\**TypeAliasRegistry\****

```
public class TypeAliasRegistry {
    private final Map<String, Class<?>> TYPE_ALIASES = new HashMap();

    public TypeAliasRegistry() {
        this.registerAlias("string", String.class);
        this.registerAlias("byte", Byte.class);
        this.registerAlias("long", Long.class);
        this.registerAlias("short", Short.class);
        this.registerAlias("int", Integer.class);
        this.registerAlias("integer", Integer.class);
        this.registerAlias("double", Double.class);
        this.registerAlias("float", Float.class);
        this.registerAlias("boolean", Boolean.class);
        this.registerAlias("byte[]", Byte[].class);
        this.registerAlias("long[]", Long[].class);
        this.registerAlias("short[]", Short[].class);
        this.registerAlias("int[]", Integer[].class);
        this.registerAlias("integer[]", Integer[].class);
        this.registerAlias("double[]", Double[].class);
        this.registerAlias("float[]", Float[].class);
        this.registerAlias("boolean[]", Boolean[].class);
        this.registerAlias("_byte", Byte.TYPE);
        this.registerAlias("_long", Long.TYPE);
        this.registerAlias("_short", Short.TYPE);
        this.registerAlias("_int", Integer.TYPE);
        this.registerAlias("_integer", Integer.TYPE);
        this.registerAlias("_double", Double.TYPE);
        this.registerAlias("_float", Float.TYPE);
        this.registerAlias("_boolean", Boolean.TYPE);
        this.registerAlias("_byte[]", byte[].class);
        this.registerAlias("_long[]", long[].class);
        this.registerAlias("_short[]", short[].class);
        this.registerAlias("_int[]", int[].class);
        this.registerAlias("_integer[]", int[].class);
        this.registerAlias("_double[]", double[].class);
        this.registerAlias("_float[]", float[].class);
        this.registerAlias("_boolean[]", boolean[].class);
        this.registerAlias("date", Date.class);
        this.registerAlias("decimal", BigDecimal.class);
        this.registerAlias("bigdecimal", BigDecimal.class);
        this.registerAlias("biginteger", BigInteger.class);
        this.registerAlias("object", Object.class);
        this.registerAlias("date[]", Date[].class);
        this.registerAlias("decimal[]", BigDecimal[].class);
        this.registerAlias("bigdecimal[]", BigDecimal[].class);
        this.registerAlias("biginteger[]", BigInteger[].class);
        this.registerAlias("object[]", Object[].class);
        this.registerAlias("map", Map.class);
        this.registerAlias("hashmap", HashMap.class);
        this.registerAlias("list", List.class);
        this.registerAlias("arraylist", ArrayList.class);
        this.registerAlias("collection", Collection.class);
        this.registerAlias("iterator", Iterator.class);
        this.registerAlias("ResultSet", ResultSet.class);
    }

    public void registerAliases(String packageName) {
        this.registerAliases(packageName, Object.class);
    }
    //略
}
```

其实TypeAliasRegistry里面有一个**HashMap，**并且在TypeAliasRegistry的构造器中注册很多别名到这个hashMap中，好了，到现在我们只是创建了一个 XMLConfigBuilder，在其构造器中我们创建了一个 Configuration 对象，接下来我们看看将mybatis-config.xml解析成Configuration中对应的属性，也就是**parser.parse()**方法：

**XMLConfigBuilder**

```
1 public Configuration parse() {
2     if (parsed) {
3         throw new BuilderException("Each XMLConfigBuilder can only be used once.");
4     }
5     parsed = true;
6     // 解析配置
7     parseConfiguration(parser.evalNode("/configuration"));
8     return configuration;
9 }
```

我们看看第7行，注意一个 xpath 表达式 - `/configuration`。这个表达式代表的是 MyBatis 的`<configuration/>`标签，这里选中这个标签，并传递给`parseConfiguration`方法。我们继续跟下去。

```
private void parseConfiguration(XNode root) {
    try {
        // 解析 properties 配置
        propertiesElement(root.evalNode("properties"));

        // 解析 settings 配置，并将其转换为 Properties 对象
        Properties settings = settingsAsProperties(root.evalNode("settings"));

        // 加载 vfs
        loadCustomVfs(settings);

        // 解析 typeAliases 配置
        typeAliasesElement(root.evalNode("typeAliases"));

        // 解析 plugins 配置
        pluginElement(root.evalNode("plugins"));

        // 解析 objectFactory 配置
        objectFactoryElement(root.evalNode("objectFactory"));

        // 解析 objectWrapperFactory 配置
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));

        // 解析 reflectorFactory 配置
        reflectorFactoryElement(root.evalNode("reflectorFactory"));

        // settings 中的信息设置到 Configuration 对象中
        settingsElement(settings);

        // 解析 environments 配置
        environmentsElement(root.evalNode("environments"));

        // 解析 databaseIdProvider，获取并设置 databaseId 到 Configuration 对象
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));

        // 解析 typeHandlers 配置
        typeHandlerElement(root.evalNode("typeHandlers"));

        // 解析 mappers 配置
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```

## 解析 properties 配置

先来看一下 properties 节点的配置内容。如下：

```
<properties resource="db.properties">
    <property name="username" value="root"/>
    <property name="password" value="123456"/>
</properties>
```

我为 properties 节点配置了一个 resource 属性，以及两个子节点。接着我们看看propertiesElement的逻辑

```
private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
        // 解析 propertis 的子节点，并将这些节点内容转换为属性对象 Properties
        Properties defaults = context.getChildrenAsProperties();
        // 获取 propertis 节点中的 resource 和 url 属性值
        String resource = context.getStringAttribute("resource");
        String url = context.getStringAttribute("url");

        // 两者都不用空，则抛出异常
        if (resource != null && url != null) {
            throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
        }
        if (resource != null) {
            // 从文件系统中加载并解析属性文件
            defaults.putAll(Resources.getResourceAsProperties(resource));
        } else if (url != null) {
            // 通过 url 加载并解析属性文件
            defaults.putAll(Resources.getUrlAsProperties(url));
        }
        Properties vars = configuration.getVariables();
        if (vars != null) {
            defaults.putAll(vars);
        }
        parser.setVariables(defaults);
        // 将属性值设置到 configuration 中
        configuration.setVariables(defaults);
    }
}

public Properties getChildrenAsProperties() {
    //创建一个Properties对象
    Properties properties = new Properties();
    // 获取并遍历子节点
    for (XNode child : getChildren()) {
        // 获取 property 节点的 name 和 value 属性
        String name = child.getStringAttribute("name");
        String value = child.getStringAttribute("value");
        if (name != null && value != null) {
            // 设置属性到属性对象中
            properties.setProperty(name, value);
        }
    }
    return properties;
}

// -☆- XNode
public List<XNode> getChildren() {
    List<XNode> children = new ArrayList<XNode>();
    // 获取子节点列表
    NodeList nodeList = node.getChildNodes();
    if (nodeList != null) {
        for (int i = 0, n = nodeList.getLength(); i < n; i++) {
            Node node = nodeList.item(i);
            if (node.getNodeType() == Node.ELEMENT_NODE) {
                children.add(new XNode(xpathParser, node, variables));
            }
        }
    }
    return children;
}
```

解析properties主要分三个步骤：

1. 解析 properties 节点的子节点，并将解析结果设置到 Properties 对象中。
2. 从文件系统或通过网络读取属性配置，这取决于 properties 节点的 resource 和 url 是否为空。
3. 将解析出的属性对象设置到 XPathParser 和 Configuration 对象中。

需要注意的是，propertiesElement 方法是先解析 properties 节点的子节点内容，后再从文件系统或者网络读取属性配置，并将所有的属性及属性值都放入到 defaults 属性对象中。这就会存在同名属性覆盖的问题，也就是从文件系统，或者网络上读取到的属性及属性值会覆盖掉 properties 子节点中同名的属性和及值。

## 解析 settings 配置



### settings 节点的解析过程

下面先来看一个settings比较简单的配置，如下：

```
<settings>
    <setting name="cacheEnabled" value="true"/>
    <setting name="lazyLoadingEnabled" value="true"/>
    <setting name="autoMappingBehavior" value="PARTIAL"/>
</settings>
```

接着来看看settingsAsProperties

```
private Properties settingsAsProperties(XNode context) {
    if (context == null) {
        return new Properties();
    }
    // 获取 settings 子节点中的内容，解析成Properties，getChildrenAsProperties 方法前面已分析过
    Properties props = context.getChildrenAsProperties();

    // 创建 Configuration 类的“元信息”对象
    MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
    for (Object key : props.keySet()) {
        // 检测 Configuration 中是否存在相关属性，不存在则抛出异常
        if (!metaConfig.hasSetter(String.valueOf(key))) {
            throw new BuilderException("The setting " + key + " is not known.  Make sure you spelled it correctly (case sensitive).");
        }
    }
    return props;
}
```



### 设置 settings 配置到 Configuration 中

接着我们看看将 settings 配置设置到 Configuration 对象中的过程。如下：

```
private void settingsElement(Properties props) throws Exception {
    // 设置 autoMappingBehavior 属性，默认值为 PARTIAL
    configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
    configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
    // 设置 cacheEnabled 属性，默认值为 true
    configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));

    // 省略部分代码

    // 解析默认的枚举处理器
    Class<? extends TypeHandler> typeHandler = (Class<? extends TypeHandler>)resolveClass(props.getProperty("defaultEnumTypeHandler"));
    // 设置默认枚举处理器
    configuration.setDefaultEnumTypeHandler(typeHandler);
    configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
    configuration.setUseActualParamName(booleanValueOf(props.getProperty("useActualParamName"), true));
    
    // 省略部分代码
}
```

上面代码处理调用 Configuration 的 setter 方法

## 解析 typeAliases 配置

在 MyBatis 中，可以为我们自己写的有些类定义一个别名。这样在使用的时候，我们只需要输入别名即可，无需再把全限定的类名写出来。在 MyBatis 中，我们有两种方式进行别名配置。第一种是仅配置包名，让 MyBatis 去扫描包中的类型，并根据类型得到相应的别名

```
<typeAliases>
    <package name="com.mybatis.model"/>
</typeAliases>
```

第二种方式是通过手动的方式，明确为某个类型配置别名。这种方式的配置如下：

```
<typeAliases>
    <typeAlias alias="employe" type="com.mybatis.model.Employe" />
    <typeAlias type="com.mybatis.model.User" />
</typeAliases>
```

下面我们来看一下两种不同的别名配置是怎样解析的。代码如下：

**XMLConfigBuilder**

```
private void typeAliasesElement(XNode parent) {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            // 从指定的包中解析别名和类型的映射
            if ("package".equals(child.getName())) {
                String typeAliasPackage = child.getStringAttribute("name");
                configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
                
            // 从 typeAlias 节点中解析别名和类型的映射
            } else {
                // 获取 alias 和 type 属性值，alias 不是必填项，可为空
                String alias = child.getStringAttribute("alias");
                String type = child.getStringAttribute("type");
                try {
                    // 加载 type 对应的类型
                    Class<?> clazz = Resources.classForName(type);

                    // 注册别名到类型的映射
                    if (alias == null) {
                        typeAliasRegistry.registerAlias(clazz);
                    } else {
                        typeAliasRegistry.registerAlias(alias, clazz);
                    }
                } catch (ClassNotFoundException e) {
                    throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
                }
            }
        }
    }
}
```

我们看到通过包扫描和手动注册时通过子节点名称是否package来判断的



### 从 typeAlias 节点中解析并注册别名

在别名的配置中，`type`属性是必须要配置的，而`alias`属性则不是必须的。

```
private final Map<String, Class<?>> TYPE_ALIASES = new HashMap<String, Class<?>>();

public void registerAlias(Class<?> type) {
    // 获取全路径类名的简称
    String alias = type.getSimpleName();
    Alias aliasAnnotation = type.getAnnotation(Alias.class);
    if (aliasAnnotation != null) {
        // 从注解中取出别名
        alias = aliasAnnotation.value();
    }
    // 调用重载方法注册别名和类型映射
    registerAlias(alias, type);
}

public void registerAlias(String alias, Class<?> value) {
    if (alias == null) {
        throw new TypeException("The parameter alias cannot be null");
    }
    // 将别名转成小写
    String key = alias.toLowerCase(Locale.ENGLISH);
    /*
     * 如果 TYPE_ALIASES 中存在了某个类型映射，这里判断当前类型与映射中的类型是否一致，
     * 不一致则抛出异常，不允许一个别名对应两种类型
     */
    if (TYPE_ALIASES.containsKey(key) && TYPE_ALIASES.get(key) != null && !TYPE_ALIASES.get(key).equals(value)) {
        throw new TypeException(
            "The alias '" + alias + "' is already mapped to the value '" + TYPE_ALIASES.get(key).getName() + "'.");
    }
    // 缓存别名到类型映射
    TYPE_ALIASES.put(key, value);
}
```

若用户为明确配置 alias 属性，MyBatis 会使用类名的小写形式作为别名。比如，全限定类名com.mybatis.model.User的别名为user。若类中有@Alias注解，则从注解中取值作为别名。



### 从指定的包中解析并注册别名

```
public void registerAliases(String packageName) {
    registerAliases(packageName, Object.class);
}

public void registerAliases(String packageName, Class<?> superType) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    /*
     * 查找包下的父类为 Object.class 的类。
     * 查找完成后，查找结果将会被缓存到resolverUtil的内部集合中。
     */ 
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    // 获取查找结果
    Set<Class<? extends Class<?>>> typeSet = resolverUtil.getClasses();
    for (Class<?> type : typeSet) {
        // 忽略匿名类，接口，内部类
        if (!type.isAnonymousClass() && !type.isInterface() && !type.isMemberClass()) {
            // 为类型注册别名 
            registerAlias(type);
        }
    }
}
```

主要分为两个步骤：

1. 查找指定包下的所有类
2. 遍历查找到的类型集合，为每个类型注册别名

我们看看查找指定包下的所有类

```
private Set<Class<? extends T>> matches = new HashSet();

public ResolverUtil<T> find(ResolverUtil.Test test, String packageName) {
    //将包名转换成文件路径
    String path = this.getPackagePath(packageName);

    try {
        //通过 VFS（虚拟文件系统）获取指定包下的所有文件的路径名，比如com/mybatis/model/Employe.class
        List<String> children = VFS.getInstance().list(path);
        Iterator i$ = children.iterator();

        while(i$.hasNext()) {
            String child = (String)i$.next();
            //以.class结尾的文件就加入到Set集合中
            if (child.endsWith(".class")) {
                this.addIfMatching(test, child);
            }
        }
    } catch (IOException var7) {
        log.error("Could not read package: " + packageName, var7);
    }

    return this;
}

protected String getPackagePath(String packageName) {
    //将包名转换成文件路径
    return packageName == null ? null : packageName.replace('.', '/');
}

protected void addIfMatching(ResolverUtil.Test test, String fqn) {
    try {
        //将路径名转成全限定的类名，通过类加载器加载类名，比如com.mybatis.model.Employe.class
        String externalName = fqn.substring(0, fqn.indexOf(46)).replace('/', '.');
        ClassLoader loader = this.getClassLoader();
        if (log.isDebugEnabled()) {
            log.debug("Checking to see if class " + externalName + " matches criteria [" + test + "]");
        }

        Class<?> type = loader.loadClass(externalName);
        if (test.matches(type)) {
            //加入到matches集合中
            this.matches.add(type);
        }
    } catch (Throwable var6) {
        log.warn("Could not examine class '" + fqn + "'" + " due to a " + var6.getClass().getName() + " with message: " + var6.getMessage());
    }

}
```

主要有以下几步：

1. 通过 VFS（虚拟文件系统）获取指定包下的所有文件的路径名，比如 **com/mybatis/model/Employe.class**
2. 筛选以`.class`结尾的文件名
3. 将路径名转成全限定的类名，通过类加载器加载类名
4. 对类型进行匹配，若符合匹配规则，则将其放入内部集合中

这里我们要注意，在前面我们分析Configuration对象的创建时，就已经默认注册了很多别名，可以回到文章开头看看。

## 解析 plugins 配置

插件是 MyBatis 提供的一个拓展机制，通过插件机制我们可在 SQL 执行过程中的某些点上做一些自定义操作。比喻分页插件，在SQL执行之前动态拼接语句，我们后面会单独来讲插件机制，先来了解插件的配置。如下：

```
<plugins>
    <plugin interceptor="com.github.pagehelper.PageInterceptor">
        <property name="helperDialect" value="mysql"/>
    </plugin>
</plugins>
```

解析过程分析如下：

```
private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            String interceptor = child.getStringAttribute("interceptor");
            // 获取配置信息
            Properties properties = child.getChildrenAsProperties();
            // 解析拦截器的类型，并创建拦截器
            Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
            // 设置属性
            interceptorInstance.setProperties(properties);
            // 添加拦截器到 Configuration 中
            configuration.addInterceptor(interceptorInstance);
        }
    }
}
```

首先是获取配置，然后再解析拦截器类型，并实例化拦截器。最后向拦截器中设置属性，并将拦截器添加到 Configuration 中。

```
private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            String interceptor = child.getStringAttribute("interceptor");
            // 获取配置信息
            Properties properties = child.getChildrenAsProperties();
            // 解析拦截器的类型，并实例化拦截器
            Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
            // 设置属性
            interceptorInstance.setProperties(properties);
            // 添加拦截器到 Configuration 中
            configuration.addInterceptor(interceptorInstance);
        }
    }
}

public void addInterceptor(Interceptor interceptor) {
    //添加到Configuration的interceptorChain属性中
    this.interceptorChain.addInterceptor(interceptor);
}
```

我们来看看InterceptorChain

```
public class InterceptorChain {
    private final List<Interceptor> interceptors = new ArrayList();

    public InterceptorChain() {
    }

    public Object pluginAll(Object target) {
        Interceptor interceptor;
        for(Iterator i$ = this.interceptors.iterator(); i$.hasNext(); target = interceptor.plugin(target)) {
            interceptor = (Interceptor)i$.next();
        }

        return target;
    }

    public void addInterceptor(Interceptor interceptor) {
        this.interceptors.add(interceptor);
    }

    public List<Interceptor> getInterceptors() {
        return Collections.unmodifiableList(this.interceptors);
    }
}
```

实际上是一个 interceptors 集合，关于插件的原理我们后面再讲。

## 解析 environments 配置

在 MyBatis 中，事务管理器和数据源是配置在 environments 中的。它们的配置大致如下：

```
<environments default="development">
    <environment id="development">
        <transactionManager type="JDBC"/>
        <dataSource type="POOLED">
            <property name="driver" value="${jdbc.driver}"/>
            <property name="url" value="${jdbc.url}"/>
            <property name="username" value="${jdbc.username}"/>
            <property name="password" value="${jdbc.password}"/>
        </dataSource>
    </environment>
</environments>
```

我们来看看environmentsElement方法

```
private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
        if (environment == null) {
            // 获取 default 属性
            environment = context.getStringAttribute("default");
        }
        for (XNode child : context.getChildren()) {
            // 获取 id 属性
            String id = child.getStringAttribute("id");
            /*
             * 检测当前 environment 节点的 id 与其父节点 environments 的属性 default 
             * 内容是否一致，一致则返回 true，否则返回 false
             * 将其default属性值与子元素environment的id属性值相等的子元素设置为当前使用的Environment对象
             */
            if (isSpecifiedEnvironment(id)) {
                // 将environment中的transactionManager标签转换为TransactionFactory对象
                TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
                // 将environment中的dataSource标签转换为DataSourceFactory对象
                DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
                // 创建 DataSource 对象
                DataSource dataSource = dsFactory.getDataSource();
                Environment.Builder environmentBuilder = new Environment.Builder(id)
                    .transactionFactory(txFactory)
                    .dataSource(dataSource);
                // 构建 Environment 对象，并设置到 configuration 中
                configuration.setEnvironment(environmentBuilder.build());
            }
        }
    }
}
```

看看TransactionFactory和 DataSourceFactory的获取

```
private TransactionFactory transactionManagerElement(XNode context) throws Exception {
    if (context != null) {
        String type = context.getStringAttribute("type");
        Properties props = context.getChildrenAsProperties();
        //通过别名获取Class,并实例化
        TransactionFactory factory = (TransactionFactory)this.resolveClass(type).newInstance();
        factory.setProperties(props);
        return factory;
    } else {
        throw new BuilderException("Environment declaration requires a TransactionFactory.");
    }
}

private DataSourceFactory dataSourceElement(XNode context) throws Exception {
    if (context != null) {
        String type = context.getStringAttribute("type");
        //通过别名获取Class,并实例化
        Properties props = context.getChildrenAsProperties();
        DataSourceFactory factory = (DataSourceFactory)this.resolveClass(type).newInstance();
        factory.setProperties(props);
        return factory;
    } else {
        throw new BuilderException("Environment declaration requires a DataSourceFactory.");
    }
}
```

**<transactionManager type="JDBC"/>**中type有"JDBC"、"MANAGED"这两种配置，而我们前面Configuration中默认注册的别名中有对应的JdbcTransactionFactory.class、ManagedTransactionFactory.class这两个TransactionFactory

**<dataSource type="POOLED">**中type有"JNDI"、"POOLED"、"UNPOOLED"这三种配置，默认注册的别名中有对应的JndiDataSourceFactory.class、PooledDataSourceFactory.class、UnpooledDataSourceFactory.class这三个DataSourceFactory

而我们的environment配置中**transactionManager type="JDBC"和\**dataSource type="POOLED"，则生成的\*\*transactionManager为JdbcTransactionFactory，DataSourceFactory为PooledDataSourceFactory\*\**\***

我们来看看PooledDataSourceFactory和UnpooledDataSourceFactory

```
public class UnpooledDataSourceFactory implements DataSourceFactory {
    private static final String DRIVER_PROPERTY_PREFIX = "driver.";
    private static final int DRIVER_PROPERTY_PREFIX_LENGTH = "driver.".length();
    //创建UnpooledDataSource实例
    protected DataSource dataSource = new UnpooledDataSource();
    
    public DataSource getDataSource() {
        return this.dataSource;
    }
    //略
}
 
//继承UnpooledDataSourceFactory
public class PooledDataSourceFactory extends UnpooledDataSourceFactory {
    public PooledDataSourceFactory() {
        //创建PooledDataSource实例
        this.dataSource = new PooledDataSource();
    }
}
```

我们发现 UnpooledDataSourceFactory 创建的dataSource是 UnpooledDataSource，PooledDataSourceFactory创建的 dataSource是PooledDataSource

## 解析 mappers 配置

mapperElement方法会将mapper标签内的元素转换成MapperProxyFactory产生的代理类，和与mapper.xml文件的绑定，我们下一篇文章会详解介绍这个方法

```
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
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

## 创建DefaultSqlSessionFactory

到此为止XMLConfigBuilder的parse方法中的重要步骤都过了一遍了，然后返回的就是一个完整的Configuration对象了，最后通过SqlSessionFactoryBuilder的build的重载方法创建了一个SqlSessionFactory实例DefaultSqlSessionFactory，我们来看看

```
public SqlSessionFactory build(Configuration config) {
    //创建DefaultSqlSessionFactory实例
    return new DefaultSqlSessionFactory(config);
}

public class DefaultSqlSessionFactory implements SqlSessionFactory {
    private final Configuration configuration;

    //只是将configuration设置为其属性
    public DefaultSqlSessionFactory(Configuration configuration) {
        this.configuration = configuration;
    }
    
    //略
}
```
