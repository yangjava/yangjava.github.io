---
layout: post
categories: Mybatis
description: none
keywords: Mybatis
---
# Mybatis源码创建Mapper代理


**正文**

刚开始使用Mybaits的同学有没有这样的疑惑，为什么我们没有编写Mapper的实现类，却能调用Mapper的方法呢？本篇文章我带大家一起来解决这个疑问

上一篇文章我们获取到了DefaultSqlSession，接着我们来看第一篇文章测试用例后面的代码

```
EmployeeMapper employeeMapper = sqlSession.getMapper(EmployeeMapper.class);
List<Employee> allEmployees = employeeMapper.getAll();
```



## 为 Mapper 接口创建代理对象

我们先从 DefaultSqlSession 的 getMapper 方法开始看起，如下：



```
 1 public <T> T getMapper(Class<T> type) {
 2     return configuration.<T>getMapper(type, this);
 3 }
 4 
 5 // Configuration
 6 public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
 7     return mapperRegistry.getMapper(type, sqlSession);
 8 }
 9 
10 // MapperRegistry
11 public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
12     // 从 knownMappers 中获取与 type 对应的 MapperProxyFactory
13     final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
14     if (mapperProxyFactory == null) {
15         throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
16     }
17     try {
18         // 创建代理对象
19         return mapperProxyFactory.newInstance(sqlSession);
20     } catch (Exception e) {
21         throw new BindingException("Error getting mapper instance. Cause: " + e, e);
22     }
23 }
```



这里最重要就是两行代码，第13行和第19行，我们接下来就分析这两行代码



### **获取MapperProxyFactory**

根据名称看，可以理解为Mapper代理的创建工厂，是不是Mapper的代理对象由它创建呢？我们先来回顾一下knownMappers 集合中的元素是何时存入的。这要在我前面的文章中找答案，MyBatis 在解析配置文件的 <mappers> 节点的过程中，会调用 MapperRegistry 的 addMapper 方法将 Class 到 MapperProxyFactory 对象的映射关系存入到 knownMappers。有兴趣的同学可以看看我之前的文章，我们来回顾一下源码：



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



在解析Mapper.xml的最后阶段，获取到Mapper.xml的namespace，然后利用反射，获取到namespace的Class,并创建一个MapperProxyFactory的实例，namespace的Class作为参数，最后将namespace的Class为key，MapperProxyFactory的实例为value存入knownMappers。

注意，我们这里是通过映射文件的命名空间的Class当做knownMappers的Key。然后我们看看getMapper方法的13行，是通过参数Employee.class也就是Mapper接口的Class来获取MapperProxyFactory，所以我们明白了为什么要求xml配置中的namespace要和和对应的Mapper接口的全限定名了



### 生成代理对象

我们看第19行代码 **return mapperProxyFactory.newInstance(sqlSession);，**很明显是调用了MapperProxyFactory的一个工厂方法，我们跟进去看看



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
         /*
         * 创建 MapperProxy 对象，MapperProxy 实现了 InvocationHandler 接口，代理逻辑封装在此类中
         * 将sqlSession传入MapperProxy对象中，第二个参数是Mapper的接口，并不是其实现类
         */
        MapperProxy<T> mapperProxy = new MapperProxy(sqlSession, this.mapperInterface, this.methodCache);
        return this.newInstance(mapperProxy);
    }
}
```



上面的代码首先创建了一个 MapperProxy 对象，该对象实现了 InvocationHandler 接口。然后将对象作为参数传给重载方法，并在重载方法中调用 JDK 动态代理接口为 Mapper接口 生成代理对象。

这里要注意一点，MapperProxy这个InvocationHandler 创建的时候，传入的参数并不是Mapper接口的实现类，我们以前是怎么创建JDK动态代理的？先创建一个接口，然后再创建一个接口的实现类，最后创建一个InvocationHandler并将实现类传入其中作为目标类，创建接口的代理类，然后调用代理类方法时会回调InvocationHandler的invoke方法，最后在invoke方法中调用目标类的方法，但是我们这里调用Mapper接口代理类的方法时，需要调用其实现类的方法吗？不需要，我们需要调用对应的配置文件的SQL，所以这里并不需要传入Mapper的实现类到MapperProxy中，那Mapper接口的代理对象是如何调用对应配置文件的SQL呢？下面我们来看看。



## Mapper代理类如何执行SQL？

上面一节中我们已经获取到了EmployeeMapper的代理类，并且其InvocationHandler为MapperProxy，那我们接着看Mapper接口方法的调用

```
List<Employee> allEmployees = employeeMapper.getAll();
```

知道JDK动态代理的同学都知道，调用代理类的方法，最后都会回调到InvocationHandler的Invoke方法，那我们来看看这个InvocationHandler（MapperProxy）



```
public class MapperProxy<T> implements InvocationHandler, Serializable {
    private final SqlSession sqlSession;
    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethod> methodCache;

    public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
        this.sqlSession = sqlSession;
        this.mapperInterface = mapperInterface;
        this.methodCache = methodCache;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 如果方法是定义在 Object 类中的，则直接调用
        if (Object.class.equals(method.getDeclaringClass())) {
            try {
                return method.invoke(this, args);
            } catch (Throwable var5) {
                throw ExceptionUtil.unwrapThrowable(var5);
            }
        } else {
            // 从缓存中获取 MapperMethod 对象，若缓存未命中，则创建 MapperMethod 对象
            MapperMethod mapperMethod = this.cachedMapperMethod(method);
            // 调用 execute 方法执行 SQL
            return mapperMethod.execute(this.sqlSession, args);
        }
    }

    private MapperMethod cachedMapperMethod(Method method) {
        MapperMethod mapperMethod = (MapperMethod)this.methodCache.get(method);
        if (mapperMethod == null) {
            //创建一个MapperMethod，参数为mapperInterface和method还有Configuration
            mapperMethod = new MapperMethod(this.mapperInterface, method, this.sqlSession.getConfiguration());
            this.methodCache.put(method, mapperMethod);
        }

        return mapperMethod;
    }
}
```

如上，回调函数**invoke**逻辑会首先检测被拦截的方法是不是定义在 Object 中的，比如 equals、hashCode 方法等。对于这类方法，直接执行即可。紧接着从缓存中获取或者创建 MapperMethod 对象，然后通过该对象中的 execute 方法执行 SQL。我们先来看看如何创建MapperMethod



### 创建 MapperMethod 对象

```
public class MapperMethod {

    //包含SQL相关信息，比喻MappedStatement的id属性，（mapper.EmployeeMapper.getAll）
    private final SqlCommand command;
    //包含了关于执行的Mapper方法的参数类型和返回类型。
    private final MethodSignature method;

    public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
        // 创建 SqlCommand 对象，该对象包含一些和 SQL 相关的信息
        this.command = new SqlCommand(config, mapperInterface, method);
        // 创建 MethodSignature 对象，从类名中可知，该对象包含了被拦截方法的一些信息
        this.method = new MethodSignature(config, mapperInterface, method);
    }
}
```

MapperMethod包含SqlCommand 和MethodSignature 对象，我们来看看其创建过程

**① 创建 SqlCommand 对象**

```
public static class SqlCommand {
    //name为MappedStatement的id，也就是namespace.methodName（mapper.EmployeeMapper.getAll）
    private final String name;
    //SQL的类型，如insert，delete，update
    private final SqlCommandType type;

    public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
        //拼接Mapper接口名和方法名，（mapper.EmployeeMapper.getAll）
        String statementName = mapperInterface.getName() + "." + method.getName();
        MappedStatement ms = null;
        //检测configuration是否有key为mapper.EmployeeMapper.getAll的MappedStatement
        if (configuration.hasStatement(statementName)) {
            //获取MappedStatement
            ms = configuration.getMappedStatement(statementName);
        } else if (!mapperInterface.equals(method.getDeclaringClass())) {
            String parentStatementName = method.getDeclaringClass().getName() + "." + method.getName();
            if (configuration.hasStatement(parentStatementName)) {
                ms = configuration.getMappedStatement(parentStatementName);
            }
        }
        
        // 检测当前方法是否有对应的 MappedStatement
        if (ms == null) {
            if (method.getAnnotation(Flush.class) != null) {
                name = null;
                type = SqlCommandType.FLUSH;
            } else {
                throw new BindingException("Invalid bound statement (not found): "
                    + mapperInterface.getName() + "." + methodName);
            }
        } else {
            // 设置 name 和 type 变量
            name = ms.getId();
            type = ms.getSqlCommandType();
            if (type == SqlCommandType.UNKNOWN) {
                throw new BindingException("Unknown execution method for: " + name);
            }
        }
    }
}

public boolean hasStatement(String statementName, boolean validateIncompleteStatements) {
    //检测configuration是否有key为statementName的MappedStatement
    return this.mappedStatements.containsKey(statementName);
}
```

通过拼接接口名和方法名，在configuration获取对应的MappedStatement，并设置设置 name 和 type 变量，代码很简单

**② 创建 MethodSignature 对象**

**MethodSignature** 包含了被拦截方法的一些信息，如目标方法的返回类型，目标方法的参数列表信息等。下面，我们来看一下 MethodSignature 的构造方法。

```
public static class MethodSignature {

    private final boolean returnsMany;
    private final boolean returnsMap;
    private final boolean returnsVoid;
    private final boolean returnsCursor;
    private final Class<?> returnType;
    private final String mapKey;
    private final Integer resultHandlerIndex;
    private final Integer rowBoundsIndex;
    private final ParamNameResolver paramNameResolver;

    public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {

        // 通过反射解析方法返回类型
        Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
        if (resolvedReturnType instanceof Class<?>) {
            this.returnType = (Class<?>) resolvedReturnType;
        } else if (resolvedReturnType instanceof ParameterizedType) {
            this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
        } else {
            this.returnType = method.getReturnType();
        }
        
        // 检测返回值类型是否是 void、集合或数组、Cursor、Map 等
        this.returnsVoid = void.class.equals(this.returnType);
        this.returnsMany = configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray();
        this.returnsCursor = Cursor.class.equals(this.returnType);
        // 解析 @MapKey 注解，获取注解内容
        this.mapKey = getMapKey(method);
        this.returnsMap = this.mapKey != null;
        /*
         * 获取 RowBounds 参数在参数列表中的位置，如果参数列表中
         * 包含多个 RowBounds 参数，此方法会抛出异常
         */ 
        this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
        // 获取 ResultHandler 参数在参数列表中的位置
        this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
        // 解析参数列表
        this.paramNameResolver = new ParamNameResolver(configuration, method);
    }
}
```





### 执行 execute 方法

前面已经分析了 MapperMethod 的初始化过程，现在 MapperMethod 创建好了。那么，接下来要做的事情是调用 MapperMethod 的 execute 方法，执行 SQL。传递参数sqlSession和method的运行参数args

```
return mapperMethod.execute(this.sqlSession, args);
```

我们去MapperMethod 的execute方法中看看

**MapperMethod**

```
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    
    // 根据 SQL 类型执行相应的数据库操作
    switch (command.getType()) {
        case INSERT: {
            // 对用户传入的参数进行转换，下同
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行插入操作，rowCountResult 方法用于处理返回值
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
        }
        case UPDATE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行更新操作
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
        }
        case DELETE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行删除操作
            result = rowCountResult(sqlSession.delete(command.getName(), param));
            break;
        }
        case SELECT:
            // 根据目标方法的返回类型进行相应的查询操作
            if (method.returnsVoid() && method.hasResultHandler()) {
                executeWithResultHandler(sqlSession, args);
                result = null;
            } else if (method.returnsMany()) {
                // 执行查询操作，并返回多个结果 
                result = executeForMany(sqlSession, args);
            } else if (method.returnsMap()) {
                // 执行查询操作，并将结果封装在 Map 中返回
                result = executeForMap(sqlSession, args);
            } else if (method.returnsCursor()) {
                // 执行查询操作，并返回一个 Cursor 对象
                result = executeForCursor(sqlSession, args);
            } else {
                Object param = method.convertArgsToSqlCommandParam(args);
                // 执行查询操作，并返回一个结果
                result = sqlSession.selectOne(command.getName(), param);
            }
            break;
        case FLUSH:
            // 执行刷新操作
            result = sqlSession.flushStatements();
            break;
        default:
            throw new BindingException("Unknown execution method for: " + command.getName());
    }
    return result;
}
```

如上，execute 方法主要由一个 switch 语句组成，用于根据 SQL 类型执行相应的数据库操作。我们先来看看是参数的处理方法convertArgsToSqlCommandParam是如何将方法参数数组转化成Map的

```
public Object convertArgsToSqlCommandParam(Object[] args) {
    return paramNameResolver.getNamedParams(args);
}

public Object getNamedParams(Object[] args) {
    final int paramCount = names.size();
    if (args == null || paramCount == 0) {
        return null;
    } else if (!hasParamAnnotation && paramCount == 1) {
        return args[names.firstKey()];
    } else {
        //创建一个Map，key为method的参数名，值为method的运行时参数值
        final Map<String, Object> param = new ParamMap<Object>();
        int i = 0;
        for (Map.Entry<Integer, String> entry : names.entrySet()) {
            // 添加 <参数名, 参数值> 键值对到 param 中
            param.put(entry.getValue(), args[entry.getKey()]);
            final String genericParamName = GENERIC_NAME_PREFIX + String.valueOf(i + 1);
            if (!names.containsValue(genericParamName)) {
                param.put(genericParamName, args[entry.getKey()]);
            }
            i++;
        }
        return param;
    }
}
```

我们看到，将Object[] args转化成了一个Map<参数名, 参数值> ，接着我们就可以看查询过程分析了，如下

```
// 执行查询操作，并返回一个结果
result = sqlSession.selectOne(command.getName(), param);
```

我们看到是通过sqlSession来执行查询的，并且传入的参数为command.getName()和param，也就是**namespace.methodName（mapper.EmployeeMapper.getAll）和方法的运行参数。**

查询操作我们下一篇文章单独来讲

# [Mybaits 源码解析 （六）----- Select 语句的执行过程分析（上篇）](https://www.cnblogs.com/java-chen-hao/p/11754184.html)

**正文**

上一篇我们分析了Mapper接口代理类的生成，本篇接着分析是如何调用到XML中的SQL

我们回顾一下MapperMethod 的execute方法

```
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    
    // 根据 SQL 类型执行相应的数据库操作
    switch (command.getType()) {
        case INSERT: {
            // 对用户传入的参数进行转换，下同
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行插入操作，rowCountResult 方法用于处理返回值
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
        }
        case UPDATE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行更新操作
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
        }
        case DELETE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行删除操作
            result = rowCountResult(sqlSession.delete(command.getName(), param));
            break;
        }
        case SELECT:
            // 根据目标方法的返回类型进行相应的查询操作
            if (method.returnsVoid() && method.hasResultHandler()) {
                executeWithResultHandler(sqlSession, args);
                result = null;
            } else if (method.returnsMany()) {
                // 执行查询操作，并返回多个结果 
                result = executeForMany(sqlSession, args);
            } else if (method.returnsMap()) {
                // 执行查询操作，并将结果封装在 Map 中返回
                result = executeForMap(sqlSession, args);
            } else if (method.returnsCursor()) {
                // 执行查询操作，并返回一个 Cursor 对象
                result = executeForCursor(sqlSession, args);
            } else {
                Object param = method.convertArgsToSqlCommandParam(args);
                // 执行查询操作，并返回一个结果
                result = sqlSession.selectOne(command.getName(), param);
            }
            break;
        case FLUSH:
            // 执行刷新操作
            result = sqlSession.flushStatements();
            break;
        default:
            throw new BindingException("Unknown execution method for: " + command.getName());
    }
    return result;
}
```

## selectOne 方法分析

本节选择分析 selectOne 方法，主要是因为 selectOne 在内部会调用 selectList 方法。同时分析 selectOne 方法等同于分析 selectList 方法。代码如下

```
// 执行查询操作，并返回一个结果
result = sqlSession.selectOne(command.getName(), param);
```

我们看到是通过sqlSession来执行查询的，并且传入的参数为command.getName()和param，也就是namespace.methodName（mapper.EmployeeMapper.getAll）和方法的运行参数。我们知道了，所有的数据库操作都是交给sqlSession来执行的，那我们就来看看sqlSession的方法

**DefaultSqlSession**

```
public <T> T selectOne(String statement, Object parameter) {
    // 调用 selectList 获取结果
    List<T> list = this.<T>selectList(statement, parameter);
    if (list.size() == 1) {
        // 返回结果
        return list.get(0);
    } else if (list.size() > 1) {
        // 如果查询结果大于1则抛出异常
        throw new TooManyResultsException(
            "Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
        return null;
    }
}
```

如上，selectOne 方法在内部调用 selectList 了方法，并取 selectList 返回值的第1个元素作为自己的返回值。如果 selectList 返回的列表元素大于1，则抛出异常。下面我们来看看 selectList 方法的实现。

**DefaultSqlSession**

```
private final Executor executor;
public <E> List<E> selectList(String statement, Object parameter) {
    // 调用重载方法
    return this.selectList(statement, parameter, RowBounds.DEFAULT);
}

public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
        // 通过MappedStatement的Id获取 MappedStatement
        MappedStatement ms = configuration.getMappedStatement(statement);
        // 调用 Executor 实现类中的 query 方法
        return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

我们之前创建**DefaultSqlSession**的时候，是创建了一个Executor的实例作为其属性的，我们看到**通过MappedStatement的Id获取 MappedStatement后，就交由Executor去执行了**

我们回顾一下前面的文章，Executor的创建过程，代码如下

```
//创建一个执行器，默认是SIMPLE
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    //根据executorType来创建相应的执行器,Configuration默认是SIMPLE
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      //创建SimpleExecutor实例，并且包含Configuration和transaction属性
      executor = new SimpleExecutor(this, transaction);
    }
    
    //如果要求缓存，生成另一种CachingExecutor,装饰者模式,默认都是返回CachingExecutor
    /**
     * 二级缓存开关配置示例
     * <settings>
     *   <setting name="cacheEnabled" value="true"/>
     * </settings>
     */
    if (cacheEnabled) {
      //CachingExecutor使用装饰器模式，将executor的功能添加上了二级缓存的功能，二级缓存会单独文章来讲
      executor = new CachingExecutor(executor);
    }
    //此处调用插件,通过插件可以改变Executor行为，此处我们后面单独文章讲
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```

executor包含了Configuration和Transaction，默认的执行器为SimpleExecutor，如果开启了二级缓存(默认开启)，则CachingExecutor会包装SimpleExecutor，那么我们该看CachingExecutor的**query**方法了

**CachingExecutor**

```
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    // 获取 BoundSql
    BoundSql boundSql = ms.getBoundSql(parameterObject);
   // 创建 CacheKey
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    // 调用重载方法
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

上面的代码用于获取 BoundSql 对象，创建 CacheKey 对象，然后再将这两个对象传给重载方法。CacheKey 以及接下来即将出现的一二级缓存将会独立成文进行分析。



### 获取 BoundSql

我们先来看看获取BoundSql

```
// 获取 BoundSql
BoundSql boundSql = ms.getBoundSql(parameterObject);
```

调用了MappedStatement的getBoundSql方法，并将运行时参数传入其中，我们大概的猜一下，这里是不是拼接SQL语句呢，并将运行时参数设置到SQL语句中？

我们都知道 SQL 是配置在映射文件中的，但由于映射文件中的 SQL 可能会包含占位符 #{}，以及动态 SQL 标签，比如 <if>、<where> 等。因此，我们并不能直接使用映射文件中配置的 SQL。MyBatis 会将映射文件中的 SQL 解析成一组 SQL 片段。我们需要对这一组片段进行解析，从每个片段对象中获取相应的内容。然后将这些内容组合起来即可得到一个完成的 SQL 语句，这个完整的 SQL 以及其他的一些信息最终会存储在 BoundSql 对象中。下面我们来看一下 BoundSql 类的成员变量信息，如下：

```
private final String sql;
private final List<ParameterMapping> parameterMappings;
private final Object parameterObject;
private final Map<String, Object> additionalParameters;
private final MetaObject metaParameters;
```

下面用一个表格列举各个成员变量的含义。

| 变量名               | 类型       | 用途                                                         |
| -------------------- | ---------- | ------------------------------------------------------------ |
| sql                  | String     | 一个完整的 SQL 语句，可能会包含问号 ? 占位符                 |
| parameterMappings    | List       | 参数映射列表，SQL 中的每个 #{xxx} 占位符都会被解析成相应的 ParameterMapping 对象 |
| parameterObject      | Object     | 运行时参数，即用户传入的参数，比如 Article 对象，或是其他的参数 |
| additionalParameters | Map        | 附加参数集合，用于存储一些额外的信息，比如 datebaseId 等     |
| metaParameters       | MetaObject | additionalParameters 的元信息对象                            |

接下来我们接着MappedStatement 的 getBoundSql 方法，代码如下：

```
public BoundSql getBoundSql(Object parameterObject) {

    // 调用 sqlSource 的 getBoundSql 获取 BoundSql，把method运行时参数传进去
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);return boundSql;
}
```

MappedStatement 的 getBoundSql 在内部调用了 SqlSource 实现类的 getBoundSql 方法，并把method运行时参数传进去，SqlSource 是一个接口，它有如下几个实现类：

- DynamicSqlSource
- RawSqlSource
- StaticSqlSource
- ProviderSqlSource
- VelocitySqlSource

当 SQL 配置中包含 `${}`（不是 #{}）占位符，或者包含 <if>、<where> 等标签时，会被认为是动态 SQL，此时使用 DynamicSqlSource 存储 SQL 片段。否则，使用 RawSqlSource 存储 SQL 配置信息。我们来看看DynamicSqlSource的**getBoundSql**

**DynamicSqlSource**

```
public BoundSql getBoundSql(Object parameterObject) {
    // 创建 DynamicContext
    DynamicContext context = new DynamicContext(configuration, parameterObject);

    // 解析 SQL 片段，并将解析结果存储到 DynamicContext 中，这里会将${}替换成method对应的运行时参数，也会解析<if><where>等SqlNode
    rootSqlNode.apply(context);
    
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    /*
     * 构建 StaticSqlSource，在此过程中将 sql 语句中的占位符 #{} 替换为问号 ?，
     * 并为每个占位符构建相应的 ParameterMapping
     */
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    
 // 调用 StaticSqlSource 的 getBoundSql 获取 BoundSql
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);

    // 将 DynamicContext 的 ContextMap 中的内容拷贝到 BoundSql 中
    for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
        boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
    }
    return boundSql;
}
```

该方法由数个步骤组成，这里总结一下：

1. 创建 DynamicContext
2. 解析 SQL 片段，并将解析结果存储到 DynamicContext 中
3. 解析 SQL 语句，并构建 StaticSqlSource
4. 调用 StaticSqlSource 的 getBoundSql 获取 BoundSql
5. 将 DynamicContext 的 ContextMap 中的内容拷贝到 BoundSql

**DynamicContext**

DynamicContext 是 SQL 语句构建的上下文，每个 SQL 片段解析完成后，都会将解析结果存入 DynamicContext 中。待所有的 SQL 片段解析完毕后，一条完整的 SQL 语句就会出现在 DynamicContext 对象中。

```
public class DynamicContext {

    public static final String PARAMETER_OBJECT_KEY = "_parameter";
    public static final String DATABASE_ID_KEY = "_databaseId";

    //bindings 则用于存储一些额外的信息，比如运行时参数
    private final ContextMap bindings;
    //sqlBuilder 变量用于存放 SQL 片段的解析结果
    private final StringBuilder sqlBuilder = new StringBuilder();

    public DynamicContext(Configuration configuration, Object parameterObject) {
        // 创建 ContextMap,并将运行时参数放入ContextMap中
        if (parameterObject != null && !(parameterObject instanceof Map)) {
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            bindings = new ContextMap(metaObject);
        } else {
            bindings = new ContextMap(null);
        }

        // 存放运行时参数 parameterObject 以及 databaseId
        bindings.put(PARAMETER_OBJECT_KEY, parameterObject);
        bindings.put(DATABASE_ID_KEY, configuration.getDatabaseId());
    }

    
    public void bind(String name, Object value) {
        this.bindings.put(name, value);
    }

    //拼接Sql片段
    public void appendSql(String sql) {
        this.sqlBuilder.append(sql);
        this.sqlBuilder.append(" ");
    }
    
    //得到sql字符串
    public String getSql() {
        return this.sqlBuilder.toString().trim();
    }

    //继承HashMap
    static class ContextMap extends HashMap<String, Object> {

        private MetaObject parameterMetaObject;

        public ContextMap(MetaObject parameterMetaObject) {
            this.parameterMetaObject = parameterMetaObject;
        }

        @Override
        public Object get(Object key) {
            String strKey = (String) key;
            // 检查是否包含 strKey，若包含则直接返回
            if (super.containsKey(strKey)) {
                return super.get(strKey);
            }

            if (parameterMetaObject != null) {
                // 从运行时参数中查找结果，这里会在${name}解析时，通过name获取运行时参数值，替换掉${name}字符串
                return parameterMetaObject.getValue(strKey);
            }

            return null;
        }
    }
    // 省略部分代码
}
```



**解析 SQL 片段**

接着我们来看看解析SQL片段的逻辑

```
rootSqlNode.apply(context);
```

对于一个包含了 ${} 占位符，或 <if>、<where> 等标签的 SQL，在解析的过程中，会被分解成多个片段。每个片段都有对应的类型，每种类型的片段都有不同的解析逻辑。在源码中，片段这个概念等价于 sql 节点，即 SqlNode。

StaticTextSqlNode 用于存储静态文本，TextSqlNode 用于存储带有 ${} 占位符的文本，IfSqlNode 则用于存储 <if> 节点的内容。MixedSqlNode 内部维护了一个 SqlNode 集合，用于存储各种各样的 SqlNode。接下来，我将会对 MixedSqlNode 、StaticTextSqlNode、TextSqlNode、IfSqlNode、WhereSqlNode 以及 TrimSqlNode 等进行分析

```
public class MixedSqlNode implements SqlNode {
    private final List<SqlNode> contents;

    public MixedSqlNode(List<SqlNode> contents) {
        this.contents = contents;
    }

    @Override
    public boolean apply(DynamicContext context) {
        // 遍历 SqlNode 集合
        for (SqlNode sqlNode : contents) {
            // 调用 salNode 对象本身的 apply 方法解析 sql
            sqlNode.apply(context);
        }
        return true;
    }
}
```

MixedSqlNode 可以看做是 SqlNode 实现类对象的容器，凡是实现了 SqlNode 接口的类都可以存储到 MixedSqlNode 中，包括它自己。MixedSqlNode 解析方法 apply 逻辑比较简单，即遍历 SqlNode 集合，并调用其他 SqlNode实现类对象的 apply 方法解析 sql。

StaticTextSqlNode

```
public class StaticTextSqlNode implements SqlNode {

    private final String text;

    public StaticTextSqlNode(String text) {
        this.text = text;
    }

    @Override
    public boolean apply(DynamicContext context) {
        //直接拼接当前sql片段的文本到DynamicContext的sqlBuilder中
        context.appendSql(text);
        return true;
    }
}
```

StaticTextSqlNode 用于存储静态文本，直接将其存储的 SQL 的文本值拼接到 DynamicContext 的**sqlBuilder**中即可。下面分析一下 TextSqlNode。

TextSqlNode

```
public class TextSqlNode implements SqlNode {

    private final String text;
    private final Pattern injectionFilter;

    @Override
    public boolean apply(DynamicContext context) {
        // 创建 ${} 占位符解析器
        GenericTokenParser parser = createParser(new BindingTokenParser(context, injectionFilter));
        // 解析 ${} 占位符，通过ONGL 从用户传入的参数中获取结果，替换text中的${} 占位符
        // 并将解析结果的文本拼接到DynamicContext的sqlBuilder中
        context.appendSql(parser.parse(text));
        return true;
    }

    private GenericTokenParser createParser(TokenHandler handler) {
        // 创建占位符解析器
        return new GenericTokenParser("${", "}", handler);
    }

    private static class BindingTokenParser implements TokenHandler {

        private DynamicContext context;
        private Pattern injectionFilter;

        public BindingTokenParser(DynamicContext context, Pattern injectionFilter) {
            this.context = context;
            this.injectionFilter = injectionFilter;
        }

        @Override
        public String handleToken(String content) {
            Object parameter = context.getBindings().get("_parameter");
            if (parameter == null) {
                context.getBindings().put("value", null);
            } else if (SimpleTypeRegistry.isSimpleType(parameter.getClass())) {
                context.getBindings().put("value", parameter);
            }
            // 通过 ONGL 从用户传入的参数中获取结果
            Object value = OgnlCache.getValue(content, context.getBindings());
            String srtValue = (value == null ? "" : String.valueOf(value));
            // 通过正则表达式检测 srtValue 有效性
            checkInjection(srtValue);
            return srtValue;
        }
    }
}
```

GenericTokenParser 是一个通用的标记解析器，用于解析形如 ${name}，#{id} 等标记。此时是解析 ${name}的形式，从运行时参数的Map中获取到key为name的值，直接用运行时参数替换掉 ${name}字符串，将替换后的text字符串拼接到DynamicContext的sqlBuilder中

举个例子吧，比喻我们有如下SQL

```
SELECT * FROM user WHERE name = '${name}' and id= ${id}
```

假如我们传的参数 Map中name值为 chenhao,id为1，那么该 SQL 最终会被解析成如下的结果：

```
SELECT * FROM user WHERE name = 'chenhao' and id= 1
```

很明显这种直接拼接值很容易造成SQL注入，假如我们传入的参数为name值为 chenhao'; DROP TABLE user;# ，解析得到的结果为

```
SELECT * FROM user WHERE name = 'chenhao'; DROP TABLE user;#'
```

由于传入的参数没有经过转义，最终导致了一条 SQL 被恶意参数拼接成了两条 SQL。这就是为什么我们不应该在 SQL 语句中是用 ${} 占位符，风险太大。接着我们来看看IfSqlNode

IfSqlNode

```
public class IfSqlNode implements SqlNode {

    private final ExpressionEvaluator evaluator;
    private final String test;
    private final SqlNode contents;

    public IfSqlNode(SqlNode contents, String test) {
        this.test = test;
        this.contents = contents;
        this.evaluator = new ExpressionEvaluator();
    }

    @Override
    public boolean apply(DynamicContext context) {
        // 通过 ONGL 评估 test 表达式的结果
        if (evaluator.evaluateBoolean(test, context.getBindings())) {
            // 若 test 表达式中的条件成立，则调用其子节点节点的 apply 方法进行解析
            // 如果是静态SQL节点，则会直接拼接到DynamicContext中
            contents.apply(context);
            return true;
        }
        return false;
    }
}
```

IfSqlNode 对应的是 <if test='xxx'> 节点，首先是通过 ONGL 检测 test 表达式是否为 true，如果为 true，则调用其子节点的 apply 方法继续进行解析。如果子节点是静态SQL节点，则子节点的文本值会直接拼接到DynamicContext中

好了，其他的SqlNode我就不一一分析了，大家有兴趣的可以去看看

**解析 #{} 占位符**

经过前面的解析，我们已经能从 DynamicContext 获取到完整的 SQL 语句了。但这并不意味着解析过程就结束了，因为当前的 SQL 语句中还有一种占位符没有处理，即 #{}。与 ${} 占位符的处理方式不同，MyBatis 并不会直接将 #{} 占位符替换为相应的参数值，而是将其替换成**？**。其解析是在如下代码中实现的

```
SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
```

我们看到将前面解析过的sql字符串和运行时参数的Map作为参数，我们来看看parse方法

```
public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
    // 创建 #{} 占位符处理器
    ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
    // 创建 #{} 占位符解析器
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    // 解析 #{} 占位符，并返回解析结果字符串
    String sql = parser.parse(originalSql);
    // 封装解析结果到 StaticSqlSource 中，并返回,因为所有的动态参数都已经解析了，可以封装成一个静态的SqlSource
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
}

public String handleToken(String content) {
    // 获取 content 的对应的 ParameterMapping
    parameterMappings.add(buildParameterMapping(content));
    // 返回 ?
    return "?";
}
```

我们看到将Sql中的 #{} 占位符替换成**"?"，并且将对应的参数转化成ParameterMapping 对象**，通过buildParameterMapping 完成,最后创建一个**StaticSqlSource，**将sql字符串和**ParameterMappings为参数传入，返回这个\**StaticSqlSource\****

```
private ParameterMapping buildParameterMapping(String content) {
    /*
     * 将#{xxx} 占位符中的内容解析成 Map。
     *   #{age,javaType=int,jdbcType=NUMERIC,typeHandler=MyTypeHandler}
     *      上面占位符中的内容最终会被解析成如下的结果：
     *  {
     *      "property": "age",
     *      "typeHandler": "MyTypeHandler", 
     *      "jdbcType": "NUMERIC", 
     *      "javaType": "int"
     *  }
     */
    Map<String, String> propertiesMap = parseParameterMapping(content);
    String property = propertiesMap.get("property");
    Class<?> propertyType;
    // metaParameters 为 DynamicContext 成员变量 bindings 的元信息对象
    if (metaParameters.hasGetter(property)) {
        propertyType = metaParameters.getGetterType(property);
    
    /*
     * parameterType 是运行时参数的类型。如果用户传入的是单个参数，比如 Employe 对象，此时 
     * parameterType 为 Employe.class。如果用户传入的多个参数，比如 [id = 1, author = "chenhao"]，
     * MyBatis 会使用 ParamMap 封装这些参数，此时 parameterType 为 ParamMap.class。
     */
    } else if (typeHandlerRegistry.hasTypeHandler(parameterType)) {
        propertyType = parameterType;
    } else if (JdbcType.CURSOR.name().equals(propertiesMap.get("jdbcType"))) {
        propertyType = java.sql.ResultSet.class;
    } else if (property == null || Map.class.isAssignableFrom(parameterType)) {
        propertyType = Object.class;
    } else {
        /*
         * 代码逻辑走到此分支中，表明 parameterType 是一个自定义的类，
         * 比如 Employe，此时为该类创建一个元信息对象
         */
        MetaClass metaClass = MetaClass.forClass(parameterType, configuration.getReflectorFactory());
        // 检测参数对象有没有与 property 想对应的 getter 方法
        if (metaClass.hasGetter(property)) {
            // 获取成员变量的类型
            propertyType = metaClass.getGetterType(property);
        } else {
            propertyType = Object.class;
        }
    }
    
    ParameterMapping.Builder builder = new ParameterMapping.Builder(configuration, property, propertyType);
    
    // 将 propertyType 赋值给 javaType
    Class<?> javaType = propertyType;
    String typeHandlerAlias = null;
    
    // 遍历 propertiesMap
    for (Map.Entry<String, String> entry : propertiesMap.entrySet()) {
        String name = entry.getKey();
        String value = entry.getValue();
        if ("javaType".equals(name)) {
            // 如果用户明确配置了 javaType，则以用户的配置为准
            javaType = resolveClass(value);
            builder.javaType(javaType);
        } else if ("jdbcType".equals(name)) {
            // 解析 jdbcType
            builder.jdbcType(resolveJdbcType(value));
        } else if ("mode".equals(name)) {...} 
        else if ("numericScale".equals(name)) {...} 
        else if ("resultMap".equals(name)) {...} 
        else if ("typeHandler".equals(name)) {
            typeHandlerAlias = value;    
        } 
        else if ("jdbcTypeName".equals(name)) {...} 
        else if ("property".equals(name)) {...} 
        else if ("expression".equals(name)) {
            throw new BuilderException("Expression based parameters are not supported yet");
        } else {
            throw new BuilderException("An invalid property '" + name + "' was found in mapping #{" + content
                + "}.  Valid properties are " + parameterProperties);
        }
    }
    if (typeHandlerAlias != null) {
        builder.typeHandler(resolveTypeHandler(javaType, typeHandlerAlias));
    }
    
    // 构建 ParameterMapping 对象
    return builder.build();
}
```

SQL 中的 #{name, ...} 占位符被替换成了问号 ?。#{name, ...} 也被解析成了一个 ParameterMapping 对象。我们再来看一下 StaticSqlSource 的创建过程。如下：

```
public class StaticSqlSource implements SqlSource {

    private final String sql;
    private final List<ParameterMapping> parameterMappings;
    private final Configuration configuration;

    public StaticSqlSource(Configuration configuration, String sql) {
        this(configuration, sql, null);
    }

    public StaticSqlSource(Configuration configuration, String sql, List<ParameterMapping> parameterMappings) {
        this.sql = sql;
        this.parameterMappings = parameterMappings;
        this.configuration = configuration;
    }

    @Override
    public BoundSql getBoundSql(Object parameterObject) {
        // 创建 BoundSql 对象
        return new BoundSql(configuration, sql, parameterMappings, parameterObject);
    }
}
```

最后我们通过创建的StaticSqlSource就可以获取BoundSql对象了，并传入运行时参数

```
BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
```

也就是调用上面创建的StaticSqlSource 中的getBoundSql方法，这是简单的 **return new** **BoundSql(configuration, sql, parameterMappings, parameterObject);** ，接着看看BoundSql

```
public class BoundSql {
    private String sql;
    private List<ParameterMapping> parameterMappings;
   private Object parameterObject;
    private Map<String, Object> additionalParameters;
    private MetaObject metaParameters;

    public BoundSql(Configuration configuration, String sql, List<ParameterMapping> parameterMappings, Object parameterObject) {
        this.sql = sql;
        this.parameterMappings = parameterMappings;
        this.parameterObject = parameterObject;
        this.additionalParameters = new HashMap();
        this.metaParameters = configuration.newMetaObject(this.additionalParameters);
    }

    public String getSql() {
        return this.sql;
    }
    //略
}
```

我们看到只是做简单的赋值。BoundSql中包含了sql，#{}解析成的parameterMappings，还有运行时参数parameterObject。好了，SQL解析我们就介绍这么多。我们先回顾一下我们代码是从哪里开始的

**CachingExecutor**

```
1 public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
2     // 获取 BoundSql
3     BoundSql boundSql = ms.getBoundSql(parameterObject);
4    // 创建 CacheKey
5     CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
6     // 调用重载方法
7     return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
8 }
```

如上，我们刚才都是分析的第三行代码，获取到了**BoundSql，**CacheKey 和二级缓存有关，我们留在下一篇文章单独来讲，接着我们看第七行重载方法 **query**

```
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    // 从 MappedStatement 中获取缓存
    Cache cache = ms.getCache();
    // 若映射文件中未配置缓存或参照缓存，此时 cache = null
    if (cache != null) {
        flushCacheIfRequired(ms);
        if (ms.isUseCache() && resultHandler == null) {
            ensureNoOutParams(ms, boundSql);
            List<E> list = (List<E>) tcm.getObject(cache, key);
            if (list == null) {
                // 若缓存未命中，则调用被装饰类的 query 方法，也就是SimpleExecutor的query方法
                list = delegate.<E>query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                tcm.putObject(cache, key, list); // issue #578 and #116
            }
            return list;
        }
    }
    // 调用被装饰类的 query 方法,这里的delegate我们知道应该是SimpleExecutor
    return delegate.<E>query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

上面的代码涉及到了二级缓存，若二级缓存为空，或未命中，则调用被装饰类的 query 方法。被装饰类为SimpleExecutor，而SimpleExecutor继承BaseExecutor，那我们来看看 BaseExecutor 的query方法

**BaseExecutor**

```
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();
    }
    List<E> list;
    try {
        queryStack++;
        // 从一级缓存中获取缓存项
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            // 一级缓存未命中，则从数据库中查询
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        deferredLoads.clear();
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            clearLocalCache();
        }
    }
    return list;
}
```

从一级缓存中查找查询结果。若缓存未命中，再向数据库进行查询。至此我们明白了一级二级缓存的大概思路，先从二级缓存中查找，若未命中二级缓存，再从一级缓存中查找，若未命中一级缓存，再从数据库查询数据，那我们来看看是怎么从数据库查询的

**BaseExecutor**

```
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds,
    ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    // 向缓存中存储一个占位符
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        // 调用 doQuery 进行查询
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        // 移除占位符
        localCache.removeObject(key);
    }
    // 缓存查询结果
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}
```

调用了**doQuery**方法进行查询，最后将查询结果放入一级缓存，我们来看看doQuery,在SimpleExecutor中

**SimpleExecutor**

```
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
        Configuration configuration = ms.getConfiguration();
        // 创建 StatementHandler
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
        // 创建 Statement
        stmt = prepareStatement(handler, ms.getStatementLog());
        // 执行查询操作
        return handler.<E>query(stmt, resultHandler);
    } finally {
        // 关闭 Statement
        closeStatement(stmt);
    }
}
```

我们先来看看第一步创建**StatementHandler**



### 创建StatementHandler

StatementHandler有什么作用呢？通过这个对象获取Statement对象，然后填充运行时参数，最后调用query完成查询。我们来看看其创建过程

```
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement,
    Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    // 创建具有路由功能的 StatementHandler
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    // 应用插件到 StatementHandler 上
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
}
```

我们看看RoutingStatementHandler的构造方法

```
public class RoutingStatementHandler implements StatementHandler {

    private final StatementHandler delegate;

    public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds,
        ResultHandler resultHandler, BoundSql boundSql) {

        // 根据 StatementType 创建不同的 StatementHandler 
        switch (ms.getStatementType()) {
            case STATEMENT:
                delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                break;
            case PREPARED:
                delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                break;
            case CALLABLE:
                delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                break;
            default:
                throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
        }
    }
    
}
```

RoutingStatementHandler 的构造方法会根据 MappedStatement 中的 statementType 变量创建不同的 StatementHandler 实现类。那statementType 是什么呢？我们还要回顾一下MappedStatement 的创建过程

![img](https://img2018.cnblogs.com/blog/1168971/201910/1168971-20191029105527512-170019357.png)



我们看到statementType 的默认类型为PREPARED，这里将会创建PreparedStatementHandler。

接着我们看下面一行代码prepareStatement,



### 创建 Statement

创建 Statement 在 stmt = prepareStatement(handler, ms.getStatementLog()); 这句代码，那我们跟进去看看

```
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    // 获取数据库连接
    Connection connection = getConnection(statementLog);
    // 创建 Statement，
    stmt = handler.prepare(connection, transaction.getTimeout());
  // 为 Statement 设置参数
    handler.parameterize(stmt);
    return stmt;
}
```

在上面的代码中我们终于看到了和jdbc相关的内容了，创建完Statement，最后就可以执行查询操作了。由于篇幅的原因，我们留在下一篇文章再来详细讲解

# [Mybaits 源码解析 （七）----- Select 语句的执行过程分析（下篇）](https://www.cnblogs.com/java-chen-hao/p/11758412.html)

**正文**

我们上篇文章讲到了查询方法里面的doQuery方法，这里面就是调用JDBC的API了，其中的逻辑比较复杂，我们这边文章来讲，先看看我们上篇文章分析的地方

**SimpleExecutor**

```
 1 public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
 2     Statement stmt = null;
 3     try {
 4         Configuration configuration = ms.getConfiguration();
 5         // 创建 StatementHandler
 6         StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
 7         // 创建 Statement
 8         stmt = prepareStatement(handler, ms.getStatementLog());
 9         // 执行查询操作
10         return handler.<E>query(stmt, resultHandler);
11     } finally {
12         // 关闭 Statement
13         closeStatement(stmt);
14     }
15 }
```

上篇文章我们分析完了第6行代码，在第6行处我们创建了一个**PreparedStatementHandler，**我们要接着第8行代码开始分析，也就是创建 Statement，先不忙着分析，我们先来回顾一下 ，我们以前是怎么使用jdbc的

**jdbc**

```
public class Login {
    /**
     *    第一步，加载驱动，创建数据库的连接
     *    第二步，编写sql
     *    第三步，需要对sql进行预编译
     *    第四步，向sql里面设置参数
     *    第五步，执行sql
     *    第六步，释放资源 
     * @throws Exception 
     */
     
    public static final String URL = "jdbc:mysql://localhost:3306/chenhao";
    public static final String USER = "liulx";
    public static final String PASSWORD = "123456";
    public static void main(String[] args) throws Exception {
        login("lucy","123");
    }
    
    public static void login(String username , String password) throws Exception{
        Connection conn = null; 
        PreparedStatement psmt = null;
        ResultSet rs = null;
        try {
            //加载驱动程序
            Class.forName("com.mysql.jdbc.Driver");
            //获得数据库连接
            conn = DriverManager.getConnection(URL, USER, PASSWORD);
            //编写sql
            String sql = "select * from user where name =? and password = ?";//问号相当于一个占位符
            //对sql进行预编译
            psmt = conn.prepareStatement(sql);
            //设置参数
            psmt.setString(1, username);
            psmt.setString(2, password);
            //执行sql ,返回一个结果集
            rs = psmt.executeQuery();
            //输出结果
            while(rs.next()){
                System.out.println(rs.getString("user_name")+" 年龄："+rs.getInt("age"));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }finally{
            //释放资源
            conn.close();
            psmt.close();
            rs.close();
        }
    }
}
```

上面代码中注释已经很清楚了，我们来看看mybatis中是怎么和数据库打交道的。

**SimpleExecutor**

```
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    // 获取数据库连接
    Connection connection = getConnection(statementLog);
   // 创建 Statement，
    stmt = handler.prepare(connection, transaction.getTimeout());
   // 为 Statement 设置参数
    handler.parameterize(stmt);
    return stmt;
}
```

在上面的代码中我们终于看到了和jdbc相关的内容了，大概分为下面三个步骤：

1. 获取数据库连接
2. 创建PreparedStatement
3. 为PreparedStatement设置运行时参数

我们先来看看获取数据库连接，跟进代码看看

**BaseExecutor**

```
protected Connection getConnection(Log statementLog) throws SQLException {
    //通过transaction来获取Connection
    Connection connection = this.transaction.getConnection();
    return statementLog.isDebugEnabled() ? ConnectionLogger.newInstance(connection, statementLog, this.queryStack) : connection;
}
```

我们看到是通过**Executor中的**transaction属性来获取Connection，那我们就先来看看transaction，根据前面的文章中的配置**` ``<``transactionManager` `type="jdbc"/>，`**则MyBatis会创建一个**JdbcTransactionFactory.class** 实例，Executor中的transaction是一个**JdbcTransaction.class** 实例，其实现Transaction接口，那我们先来看看Transaction

## **JdbcTransaction**

我们先来看看其接口Transaction

### **Transaction**

```
public interface Transaction {
    //获取数据库连接
    Connection getConnection() throws SQLException;
    //提交事务
    void commit() throws SQLException;
    //回滚事务
    void rollback() throws SQLException;
    //关闭事务
    void close() throws SQLException;
    //获取超时时间
    Integer getTimeout() throws SQLException;
}
```

接着我们看看其实现类**JdbcTransaction**



### **JdbcTransaction**

```
public class JdbcTransaction implements Transaction {
  
  private static final Log log = LogFactory.getLog(JdbcTransaction.class);
  
  //数据库连接
  protected Connection connection;
  //数据源信息
  protected DataSource dataSource;
  //隔离级别
  protected TransactionIsolationLevel level;
  //是否为自动提交
  protected boolean autoCommmit;
  
  public JdbcTransaction(DataSource ds, TransactionIsolationLevel desiredLevel, boolean desiredAutoCommit) {
    dataSource = ds;
    level = desiredLevel;
    autoCommmit = desiredAutoCommit;
  }
  
  public JdbcTransaction(Connection connection) {
    this.connection = connection;
  }
  
  public Connection getConnection() throws SQLException {
    //如果事务中不存在connection，则获取一个connection并放入connection属性中
    //第一次肯定为空
    if (connection == null) {
      openConnection();
    }
    //如果事务中已经存在connection，则直接返回这个connection
    return connection;
  }
  
    /**
     * commit()功能 
     * @throws SQLException
     */
  public void commit() throws SQLException {
    if (connection != null && !connection.getAutoCommit()) {
      if (log.isDebugEnabled()) {
        log.debug("Committing JDBC Connection [" + connection + "]");
      }
      //使用connection的commit()
      connection.commit();
    }
  }
  
    /**
     * rollback()功能 
     * @throws SQLException
     */
  public void rollback() throws SQLException {
    if (connection != null && !connection.getAutoCommit()) {
      if (log.isDebugEnabled()) {
        log.debug("Rolling back JDBC Connection [" + connection + "]");
      }
      //使用connection的rollback()
      connection.rollback();
    }
  }
  
    /**
     * close()功能 
     * @throws SQLException
     */
  public void close() throws SQLException {
    if (connection != null) {
      resetAutoCommit();
      if (log.isDebugEnabled()) {
        log.debug("Closing JDBC Connection [" + connection + "]");
      }
      //使用connection的close()
      connection.close();
    }
  }
  
  protected void openConnection() throws SQLException {
    if (log.isDebugEnabled()) {
      log.debug("Opening JDBC Connection");
    }
    //通过dataSource来获取connection，并设置到transaction的connection属性中
    connection = dataSource.getConnection();
   if (level != null) {
      //通过connection设置事务的隔离级别
      connection.setTransactionIsolation(level.getLevel());
    }
    //设置事务是否自动提交
    setDesiredAutoCommit(autoCommmit);
  }
  
  protected void setDesiredAutoCommit(boolean desiredAutoCommit) {
    try {
        if (this.connection.getAutoCommit() != desiredAutoCommit) {
            if (log.isDebugEnabled()) {
                log.debug("Setting autocommit to " + desiredAutoCommit + " on JDBC Connection [" + this.connection + "]");
            }
            //通过connection设置事务是否自动提交
            this.connection.setAutoCommit(desiredAutoCommit);
        }

    } catch (SQLException var3) {
        throw new TransactionException("Error configuring AutoCommit.  Your driver may not support getAutoCommit() or setAutoCommit(). Requested setting: " + desiredAutoCommit + ".  Cause: " + var3, var3);
    }
  }
  
}
```

我们看到JdbcTransaction中有一个**Connection属性和dataSource属性，使用****connection来进行提交、回滚、关闭等操作，也就是说JdbcTransaction其实只是在jdbc的connection上面封装了一下，实际使用的其实还是jdbc的事务。**我们看看getConnection()方法

```
//数据库连接
protected Connection connection;
//数据源信息
protected DataSource dataSource;

public Connection getConnection() throws SQLException {
//如果事务中不存在connection，则获取一个connection并放入connection属性中
//第一次肯定为空
if (connection == null) {
  openConnection();
}
//如果事务中已经存在connection，则直接返回这个connection
return connection;
}

protected void openConnection() throws SQLException {
if (log.isDebugEnabled()) {
  log.debug("Opening JDBC Connection");
}
//通过dataSource来获取connection，并设置到transaction的connection属性中
connection = dataSource.getConnection();
if (level != null) {
  //通过connection设置事务的隔离级别
  connection.setTransactionIsolation(level.getLevel());
}
//设置事务是否自动提交
setDesiredAutoCommit(autoCommmit);
}
```

先是判断当前事务中是否存在connection，如果存在，则直接返回connection，如果不存在则通过dataSource来获取connection，这里我们明白了一点，如果当前事务没有关闭，也就是没有释放connection，那么在同一个Transaction中使用的是同一个connection,我们再来想想，transaction是SimpleExecutor中的属性，*SimpleExecutor又是SqlSession中的属性，那我们可以这样说，同一个SqlSession中只有一个SimpleExecutor，SimpleExecutor中有一个Transaction，Transaction有一个connection。我们来看看如下例子


```
public static void main(String[] args) throws IOException {
    String resource = "mybatis-config.xml";
    InputStream inputStream = Resources.getResourceAsStream(resource);
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
    //创建一个SqlSession
    SqlSession sqlSession = sqlSessionFactory.openSession();
    try {
         EmployeeMapper employeeMapper = sqlSession.getMapper(Employee.class);
         UserMapper userMapper = sqlSession.getMapper(User.class);
         List<Employee> allEmployee = employeeMapper.getAll();
         List<User> allUser = userMapper.getAll();
         Employee employee = employeeMapper.getOne();
    } finally {
        sqlSession.close();
    }
}
```



我们看到同一个sqlSession可以获取多个Mapper代理对象，则多个Mapper代理对象中的sqlSession引用应该是同一个，那么多个Mapper代理对象调用方法应该是同一个Connection，直到调用close(),所以说我们的sqlSession是线程不安全的，如果所有的业务都使用一个sqlSession，那Connection也是同一个，一个业务执行完了就将其关闭，那其他的业务还没执行完呢。大家明白了吗？我们回归到源码，connection = dataSource.getConnection();，最终还是调用dataSource来获取连接，那我们是不是要来看看dataSource呢？

我们还是从前面的配置文件来看<dataSource type="UNPOOLED|POOLED">，这里有UNPOOLED和POOLED两种DataSource，一种是使用连接池，一种是普通的DataSource，UNPOOLED将会创将new UnpooledDataSource()实例，POOLED将会new pooledDataSource()实例，都实现DataSource接口，那我们先来看看DataSource接口

**DataSource**

```
public interface DataSource  extends CommonDataSource,Wrapper {
  //获取数据库连接
  Connection getConnection() throws SQLException;

  Connection getConnection(String username, String password)
    throws SQLException;

}
```

很简单，只有一个获取数据库连接的接口，那我们来看看其实现类

## UnpooledDataSource

UnpooledDataSource，从名称上即可知道，该种数据源不具有池化特性。该种数据源每次会返回一个新的数据库连接，而非复用旧的连接。其核心的方法有三个，分别如下：

1. initializeDriver - 初始化数据库驱动
2. doGetConnection - 获取数据连接
3. configureConnection - 配置数据库连接



### 初始化数据库驱动

看下我们上面使用JDBC的例子，在执行 SQL 之前，通常都是先获取数据库连接。一般步骤都是加载数据库驱动，然后通过 DriverManager 获取数据库连接。UnpooledDataSource 也是使用 JDBC 访问数据库的，因此它获取数据库连接的过程一样

**UnpooledDataSource**

```
public class UnpooledDataSource implements DataSource {
    private ClassLoader driverClassLoader;
    private Properties driverProperties;
    private static Map<String, Driver> registeredDrivers = new ConcurrentHashMap();
    private String driver;
    private String url;
    private String username;
    private String password;
    private Boolean autoCommit;
    private Integer defaultTransactionIsolationLevel;

    public UnpooledDataSource() {
    }

    public UnpooledDataSource(String driver, String url, String username, String password) {
        this.driver = driver;
        this.url = url;
        this.username = username;
        this.password = password;
    }

    private synchronized void initializeDriver() throws SQLException {
        // 检测当前 driver 对应的驱动实例是否已经注册
        if (!registeredDrivers.containsKey(driver)) {
            Class<?> driverType;
            try {
                // 加载驱动类型
                if (driverClassLoader != null) {
                    // 使用 driverClassLoader 加载驱动
                    driverType = Class.forName(driver, true, driverClassLoader);
                } else {
                    // 通过其他 ClassLoader 加载驱动
                    driverType = Resources.classForName(driver);
                }

                // 通过反射创建驱动实例
                Driver driverInstance = (Driver) driverType.newInstance();
                /*
                 * 注册驱动，注意这里是将 Driver 代理类 DriverProxy 对象注册到 DriverManager 中的，而非 Driver 对象本身。
                 */
                DriverManager.registerDriver(new DriverProxy(driverInstance));
                // 缓存驱动类名和实例，防止多次注册
                registeredDrivers.put(driver, driverInstance);
            } catch (Exception e) {
                throw new SQLException("Error setting driver on UnpooledDataSource. Cause: " + e);
            }
        }
    }
    //略...
}

//DriverManager
private final static CopyOnWriteArrayList<DriverInfo> registeredDrivers = new CopyOnWriteArrayList<DriverInfo>();
public static synchronized void registerDriver(java.sql.Driver driver)
    throws SQLException {

    if(driver != null) {
        registeredDrivers.addIfAbsent(new DriverInfo(driver));
    } else {
        // This is for compatibility with the original DriverManager
        throw new NullPointerException();
    }
}
```



```
通过反射机制加载驱动Driver，并将其注册到DriverManager中的一个常量集合中，供后面获取连接时使用，为什么这里是一个List呢？我们实际开发中有可能使用到了多种数据库类型，如Mysql、Oracle等，其驱动都是不同的，不同的数据源获取连接时使用的是不同的驱动。
在我们使用JDBC的时候，也没有通过DriverManager.registerDriver(new DriverProxy(driverInstance));去注册Driver啊，如果我们使用的是Mysql数据源，那我们来看Class.forName("com.mysql.jdbc.Driver");这句代码发生了什么
Class.forName主要是做了什么呢？它主要是要求JVM查找并装载指定的类。这样我们的类com.mysql.jdbc.Driver就被装载进来了。而且在类被装载进JVM的时候，它的静态方法就会被执行。我们来看com.mysql.jdbc.Driver的实现代码。在它的实现里有这么一段代码：
```



```
static {  
    try {  
        java.sql.DriverManager.registerDriver(new Driver());  
    } catch (SQLException E) {  
        throw new RuntimeException("Can't register driver!");  
    }  
}
```



很明显，这里使用了DriverManager并将该类给注册上去了。所以，对于任何实现前面Driver接口的类，只要在他们被装载进JVM的时候注册DriverManager就可以实现被后续程序使用。

作为那些被加载的Driver实现，他们本身在被装载时会在执行的static代码段里通过调用DriverManager.registerDriver()来把自身注册到DriverManager的registeredDrivers列表中。这样后面就可以通过得到的Driver来取得连接了。



### 获取数据库连接

在上面例子中使用 JDBC 时，我们都是通过 DriverManager 的接口方法获取数据库连接。我们来看看UnpooledDataSource是如何获取的。

**UnpooledDataSource**



```
public Connection getConnection() throws SQLException {
    return doGetConnection(username, password);
}
    
private Connection doGetConnection(String username, String password) throws SQLException {
    Properties props = new Properties();
    if (driverProperties != null) {
        props.putAll(driverProperties);
    }
    if (username != null) {
        // 存储 user 配置
        props.setProperty("user", username);
    }
    if (password != null) {
        // 存储 password 配置
        props.setProperty("password", password);
    }
    // 调用重载方法
    return doGetConnection(props);
}

private Connection doGetConnection(Properties properties) throws SQLException {
    // 初始化驱动,我们上一节已经讲过了，只用初始化一次
    initializeDriver();
    // 获取连接
    Connection connection = DriverManager.getConnection(url, properties);
    // 配置连接，包括自动提交以及事务等级
    configureConnection(connection);
    return connection;
}

private void configureConnection(Connection conn) throws SQLException {
    if (autoCommit != null && autoCommit != conn.getAutoCommit()) {
        // 设置自动提交
        conn.setAutoCommit(autoCommit);
    }
    if (defaultTransactionIsolationLevel != null) {
        // 设置事务隔离级别
        conn.setTransactionIsolation(defaultTransactionIsolationLevel);
    }
}
```



上面方法将一些配置信息放入到 Properties 对象中，然后将数据库连接和 Properties 对象传给 DriverManager 的 getConnection 方法即可获取到数据库连接。我们来看看是怎么获取数据库连接的



```
private static Connection getConnection(String url, java.util.Properties info, Class<?> caller) throws SQLException {
    // 获取类加载器
    ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
    synchronized(DriverManager.class) {
      if (callerCL == null) {
        callerCL = Thread.currentThread().getContextClassLoader();
      }
    }
    // 此处省略部分代码 
    // 这里遍历的是在registerDriver(Driver driver)方法中注册的驱动对象
    // 每个DriverInfo包含了驱动对象和其信息
    for(DriverInfo aDriver : registeredDrivers) {

      // 判断是否为当前线程类加载器加载的驱动类
      if(isDriverAllowed(aDriver.driver, callerCL)) {
        try {
          println("trying " + aDriver.driver.getClass().getName());

          // 获取连接对象，这里调用了Driver的父类的方法
          // 如果这里有多个DriverInfo，比喻Mysql和Oracle的Driver都注册registeredDrivers了
          // 这里所有的Driver都会尝试使用url和info去连接，哪个连接上了就返回
          // 会不会所有的都会连接上呢？不会，因为url的写法不同，不同的Driver会判断url是否适合当前驱动
          Connection con = aDriver.driver.connect(url, info);
          if (con != null) {
            // 打印连接成功信息
            println("getConnection returning " + aDriver.driver.getClass().getName());
            // 返回连接对像
            return (con);
          }
        } catch (SQLException ex) {
          if (reason == null) {
            reason = ex;
          }
        }
      } else {
        println("    skipping: " + aDriver.getClass().getName());
      }
    }  
}
```



代码中循环所有注册的驱动，然后通过驱动进行连接，所有的驱动都会尝试连接，但是不同的驱动，连接的URL是不同的，如Mysql的url是**jdbc:mysql://localhost:3306/chenhao，以\**jdbc:mysql://开头，则其Mysql的驱动肯定会判断获取连接的url符合，Oracle的也类似，我们来看看Mysql的驱动获取连接\****

## PooledDataSource

PooledDataSource 内部实现了连接池功能，用于复用数据库连接。因此，从效率上来说，PooledDataSource 要高于 UnpooledDataSource。但是最终获取Connection还是通过UnpooledDataSource，只不过PooledDataSource 提供一个存储Connection的功能。



### 辅助类介绍

PooledDataSource 需要借助两个辅助类帮其完成功能，这两个辅助类分别是 PoolState 和 PooledConnection。PoolState 用于记录连接池运行时的状态，比如连接获取次数，无效连接数量等。同时 PoolState 内部定义了两个 PooledConnection 集合，用于存储空闲连接和活跃连接。PooledConnection 内部定义了一个 Connection 类型的变量，用于指向真实的数据库连接。以及一个 Connection 的代理类，用于对部分方法调用进行拦截。至于为什么要拦截，随后将进行分析。除此之外，PooledConnection 内部也定义了一些字段，用于记录数据库连接的一些运行时状态。接下来，我们来看一下 PooledConnection 的定义。

**PooledConnection**



```
class PooledConnection implements InvocationHandler {

    private static final String CLOSE = "close";
    private static final Class<?>[] IFACES = new Class<?>[]{Connection.class};

    private final int hashCode;
    private final PooledDataSource dataSource;
    // 真实的数据库连接
    private final Connection realConnection;
    // 数据库连接代理
    private final Connection proxyConnection;
    
    // 从连接池中取出连接时的时间戳
    private long checkoutTimestamp;
    // 数据库连接创建时间
    private long createdTimestamp;
    // 数据库连接最后使用时间
    private long lastUsedTimestamp;
    // connectionTypeCode = (url + username + password).hashCode()
    private int connectionTypeCode;
    // 表示连接是否有效
    private boolean valid;

    public PooledConnection(Connection connection, PooledDataSource dataSource) {
        this.hashCode = connection.hashCode();
        this.realConnection = connection;
        this.dataSource = dataSource;
        this.createdTimestamp = System.currentTimeMillis();
        this.lastUsedTimestamp = System.currentTimeMillis();
        this.valid = true;
        // 创建 Connection 的代理类对象
        this.proxyConnection = (Connection) Proxy.newProxyInstance(Connection.class.getClassLoader(), IFACES, this);
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {...}
    
    // 省略部分代码
}
```



下面再来看看 PoolState 的定义。

**PoolState**



```
public class PoolState {

    protected PooledDataSource dataSource;

    // 空闲连接列表
    protected final List<PooledConnection> idleConnections = new ArrayList<PooledConnection>();
    // 活跃连接列表
    protected final List<PooledConnection> activeConnections = new ArrayList<PooledConnection>();
    // 从连接池中获取连接的次数
    protected long requestCount = 0;
    // 请求连接总耗时（单位：毫秒）
    protected long accumulatedRequestTime = 0;
    // 连接执行时间总耗时
    protected long accumulatedCheckoutTime = 0;
    // 执行时间超时的连接数
    protected long claimedOverdueConnectionCount = 0;
    // 超时时间累加值
    protected long accumulatedCheckoutTimeOfOverdueConnections = 0;
    // 等待时间累加值
    protected long accumulatedWaitTime = 0;
    // 等待次数
    protected long hadToWaitCount = 0;
    // 无效连接数
    protected long badConnectionCount = 0;
}
```



大家记住上面的空闲连接列表和活跃连接列表



### 获取连接

前面已经说过，PooledDataSource 会将用过的连接进行回收，以便可以复用连接。因此从 PooledDataSource 获取连接时，如果空闲链接列表里有连接时，可直接取用。那如果没有空闲连接怎么办呢？此时有两种解决办法，要么创建新连接，要么等待其他连接完成任务。

**PooledDataSource**



```
public class PooledDataSource implements DataSource {
    private static final Log log = LogFactory.getLog(PooledDataSource.class);
    //这里有辅助类PoolState
    private final PoolState state = new PoolState(this);
    //还有一个UnpooledDataSource属性，其实真正获取Connection是由UnpooledDataSource来完成的
    private final UnpooledDataSource dataSource;
    protected int poolMaximumActiveConnections = 10;
    protected int poolMaximumIdleConnections = 5;
    protected int poolMaximumCheckoutTime = 20000;
    protected int poolTimeToWait = 20000;
    protected String poolPingQuery = "NO PING QUERY SET";
    protected boolean poolPingEnabled = false;
    protected int poolPingConnectionsNotUsedFor = 0;
    private int expectedConnectionTypeCode;
    
    public PooledDataSource() {
        this.dataSource = new UnpooledDataSource();
    }
    
    public PooledDataSource(String driver, String url, String username, String password) {
        //构造器中创建UnpooledDataSource对象
        this.dataSource = new UnpooledDataSource(driver, url, username, password);
    }
    
    public Connection getConnection() throws SQLException {
        return this.popConnection(this.dataSource.getUsername(), this.dataSource.getPassword()).getProxyConnection();
    }
    
    private PooledConnection popConnection(String username, String password) throws SQLException {
        boolean countedWait = false;
        PooledConnection conn = null;
        long t = System.currentTimeMillis();
        int localBadConnectionCount = 0;

        while (conn == null) {
            synchronized (state) {
                // 检测空闲连接集合（idleConnections）是否为空
                if (!state.idleConnections.isEmpty()) {
                    // idleConnections 不为空，表示有空闲连接可以使用，直接从空闲连接集合中取出一个连接
                    conn = state.idleConnections.remove(0);
                } else {
                    /*
                     * 暂无空闲连接可用，但如果活跃连接数还未超出限制
                     *（poolMaximumActiveConnections），则可创建新的连接
                     */
                    if (state.activeConnections.size() < poolMaximumActiveConnections) {
                        // 创建新连接，看到没，还是通过dataSource获取连接，也就是UnpooledDataSource获取连接
                        conn = new PooledConnection(dataSource.getConnection(), this);
                    } else {    // 连接池已满，不能创建新连接
                        // 取出运行时间最长的连接
                        PooledConnection oldestActiveConnection = state.activeConnections.get(0);
                        // 获取运行时长
                        long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
                        // 检测运行时长是否超出限制，即超时
                        if (longestCheckoutTime > poolMaximumCheckoutTime) {
                            // 累加超时相关的统计字段
                            state.claimedOverdueConnectionCount++;
                            state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
                            state.accumulatedCheckoutTime += longestCheckoutTime;

                            // 从活跃连接集合中移除超时连接
                            state.activeConnections.remove(oldestActiveConnection);
                            // 若连接未设置自动提交，此处进行回滚操作
                            if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
                                try {
                                    oldestActiveConnection.getRealConnection().rollback();
                                } catch (SQLException e) {...}
                            }
                            /*
                             * 创建一个新的 PooledConnection，注意，
                             * 此处复用 oldestActiveConnection 的 realConnection 变量
                             */
                            conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
                            /*
                             * 复用 oldestActiveConnection 的一些信息，注意 PooledConnection 中的 
                             * createdTimestamp 用于记录 Connection 的创建时间，而非 PooledConnection 
                             * 的创建时间。所以这里要复用原连接的时间信息。
                             */
                            conn.setCreatedTimestamp(oldestActiveConnection.getCreatedTimestamp());
                            conn.setLastUsedTimestamp(oldestActiveConnection.getLastUsedTimestamp());

                            // 设置连接为无效状态
                            oldestActiveConnection.invalidate();
                            
                        } else {// 运行时间最长的连接并未超时
                            try {
                                if (!countedWait) {
                                    state.hadToWaitCount++;
                                    countedWait = true;
                                }
                                long wt = System.currentTimeMillis();
                                // 当前线程进入等待状态
                                state.wait(poolTimeToWait);
                                state.accumulatedWaitTime += System.currentTimeMillis() - wt;
                            } catch (InterruptedException e) {
                                break;
                            }
                        }
                    }
                }
                if (conn != null) {
                    if (conn.isValid()) {
                        if (!conn.getRealConnection().getAutoCommit()) {
                            // 进行回滚操作
                            conn.getRealConnection().rollback();
                        }
                        conn.setConnectionTypeCode(assembleConnectionTypeCode(dataSource.getUrl(), username, password));
                        // 设置统计字段
                        conn.setCheckoutTimestamp(System.currentTimeMillis());
                        conn.setLastUsedTimestamp(System.currentTimeMillis());
                        state.activeConnections.add(conn);
                        state.requestCount++;
                        state.accumulatedRequestTime += System.currentTimeMillis() - t;
                    } else {
                        // 连接无效，此时累加无效连接相关的统计字段
                        state.badConnectionCount++;
                        localBadConnectionCount++;
                        conn = null;
                        if (localBadConnectionCount > (poolMaximumIdleConnections
                            + poolMaximumLocalBadConnectionTolerance)) {
                            throw new SQLException(...);
                        }
                    }
                }
            }

        }
        if (conn == null) {
            throw new SQLException(...);
        }

        return conn;
    }
}
```



从连接池中获取连接首先会遇到两种情况：

1. 连接池中有空闲连接
2. 连接池中无空闲连接

对于第一种情况，把连接取出返回即可。对于第二种情况，则要进行细分，会有如下的情况。

1. 活跃连接数没有超出最大活跃连接数
2. 活跃连接数超出最大活跃连接数

对于上面两种情况，第一种情况比较好处理，直接创建新的连接即可。至于第二种情况，需要再次进行细分。

1. 活跃连接的运行时间超出限制，即超时了
2. 活跃连接未超时

对于第一种情况，我们直接将超时连接强行中断，并进行回滚，然后复用部分字段重新创建 PooledConnection 即可。对于第二种情况，目前没有更好的处理方式了，只能等待了。



### 回收连接

相比于获取连接，回收连接的逻辑要简单的多。回收连接成功与否只取决于空闲连接集合的状态，所需处理情况很少，因此比较简单。

我们还是来看看

```
public Connection getConnection() throws SQLException {
    return this.popConnection(this.dataSource.getUsername(), this.dataSource.getPassword()).getProxyConnection();
}
```

返回的是PooledConnection的一个代理类，为什么不直接使用PooledConnection的realConnection呢？我们可以看下PooledConnection这个类

```
class PooledConnection implements InvocationHandler {
```

很熟悉是吧，标准的代理类用法，看下其invoke方法

**PooledConnection**

```
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    String methodName = method.getName();
    // 重点在这里，如果调用了其close方法，则实际执行的是将连接放回连接池的操作
    if (CLOSE.hashCode() == methodName.hashCode() && CLOSE.equals(methodName)) {
        dataSource.pushConnection(this);
        return null;
    } else {
        try {
            if (!Object.class.equals(method.getDeclaringClass())) {
                // issue #579 toString() should never fail
                // throw an SQLException instead of a Runtime
                checkConnection();
            }
            // 其他的操作都交给realConnection执行
            return method.invoke(realConnection, args);
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
    }
}
```

那我们来看看**pushConnection**做了什么

```
protected void pushConnection(PooledConnection conn) throws SQLException {
    synchronized (state) {
        // 从活跃连接池中移除连接
        state.activeConnections.remove(conn);
        if (conn.isValid()) {
            // 空闲连接集合未满
            if (state.idleConnections.size() < poolMaximumIdleConnections
                && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
                state.accumulatedCheckoutTime += conn.getCheckoutTime();

                // 回滚未提交的事务
                if (!conn.getRealConnection().getAutoCommit()) {
                    conn.getRealConnection().rollback();
                }

                // 创建新的 PooledConnection
                PooledConnection newConn = new PooledConnection(conn.getRealConnection(), this);
                state.idleConnections.add(newConn);
                // 复用时间信息
                newConn.setCreatedTimestamp(conn.getCreatedTimestamp());
                newConn.setLastUsedTimestamp(conn.getLastUsedTimestamp());

                // 将原连接置为无效状态
                conn.invalidate();

                // 通知等待的线程
                state.notifyAll();
                
            } else {// 空闲连接集合已满
                state.accumulatedCheckoutTime += conn.getCheckoutTime();
                // 回滚未提交的事务
                if (!conn.getRealConnection().getAutoCommit()) {
                    conn.getRealConnection().rollback();
                }

                // 关闭数据库连接
                conn.getRealConnection().close();
                conn.invalidate();
            }
        } else {
            state.badConnectionCount++;
        }
    }
}
```

先将连接从活跃连接集合中移除，如果空闲集合未满，此时复用原连接的字段信息创建新的连接，并将其放入空闲集合中即可；若空闲集合已满，此时无需回收连接，直接关闭即可。

连接池总觉得很神秘，但仔细分析完其代码之后，也就没那么神秘了，就是将连接使用完之后放到一个集合中，下面再获取连接的时候首先从这个集合中获取。 还有PooledConnection的代理模式的使用，值得我们学习

好了，我们已经获取到了数据库连接，接下来要创建**PrepareStatement**了，我们上面JDBC的例子是怎么获取的？ **psmt = conn.prepareStatement(sql);，直接通过Connection来获取，并且把sql传进去了**，我们看看Mybaits中是怎么创建PrepareStatement的

## **创建PreparedStatement**

**PreparedStatementHandler**

```
stmt = handler.prepare(connection, transaction.getTimeout());

public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    Statement statement = null;
    try {
        // 创建 Statement
        statement = instantiateStatement(connection);
        // 设置超时和 FetchSize
        setStatementTimeout(statement, transactionTimeout);
        setFetchSize(statement);
        return statement;
    } catch (SQLException e) {
        closeStatement(statement);
        throw e;
    } catch (Exception e) {
        closeStatement(statement);
        throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
}

protected Statement instantiateStatement(Connection connection) throws SQLException {
    //获取sql字符串，比如"select * from user where id= ?"
    String sql = boundSql.getSql();
    // 根据条件调用不同的 prepareStatement 方法创建 PreparedStatement
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
        String[] keyColumnNames = mappedStatement.getKeyColumns();
        if (keyColumnNames == null) {
            //通过connection获取Statement，将sql语句传进去
            return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
        } else {
            return connection.prepareStatement(sql, keyColumnNames);
        }
    } else if (mappedStatement.getResultSetType() != null) {
        return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
        return connection.prepareStatement(sql);
    }
}
```

看到没和jdbc的形式一模一样，我们具体来看看**connection.prepareStatement**做了什么

```
 1 public PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency) throws SQLException {
 2        
 3     boolean canServerPrepare = true;
 4 
 5     String nativeSql = getProcessEscapeCodesForPrepStmts() ? nativeSQL(sql) : sql;
 6 
 7     if (this.useServerPreparedStmts && getEmulateUnsupportedPstmts()) {
 8         canServerPrepare = canHandleAsServerPreparedStatement(nativeSql);
 9     }
10 
11     if (this.useServerPreparedStmts && getEmulateUnsupportedPstmts()) {
12         canServerPrepare = canHandleAsServerPreparedStatement(nativeSql);
13     }
14 
15     if (this.useServerPreparedStmts && canServerPrepare) {
16         if (this.getCachePreparedStatements()) {
17             ......
18         } else {
19             try {
20                 //这里使用的是ServerPreparedStatement创建PreparedStatement
21                 pStmt = ServerPreparedStatement.getInstance(getMultiHostSafeProxy(), nativeSql, this.database, resultSetType, resultSetConcurrency);
22 
23                 pStmt.setResultSetType(resultSetType);
24                 pStmt.setResultSetConcurrency(resultSetConcurrency);
25             } catch (SQLException sqlEx) {
26                 // Punt, if necessary
27                 if (getEmulateUnsupportedPstmts()) {
28                     pStmt = (PreparedStatement) clientPrepareStatement(nativeSql, resultSetType, resultSetConcurrency, false);
29                 } else {
30                     throw sqlEx;
31                 }
32             }
33         }
34     } else {
35         pStmt = (PreparedStatement) clientPrepareStatement(nativeSql, resultSetType, resultSetConcurrency, false);
36     }
37 }
```

我们只用看最关键的第21行代码，使用**ServerPreparedStatement的getInstance返回一个**PreparedStatement，其实本质上**ServerPreparedStatement继承了PreparedStatement对象**，我们看看其构造方法

```
protected ServerPreparedStatement(ConnectionImpl conn, String sql, String catalog, int resultSetType, int resultSetConcurrency) throws SQLException {
    //略...

    try {
        this.serverPrepare(sql);
    } catch (SQLException var10) {
        this.realClose(false, true);
        throw var10;
    } catch (Exception var11) {
        this.realClose(false, true);
        SQLException sqlEx = SQLError.createSQLException(var11.toString(), "S1000", this.getExceptionInterceptor());
        sqlEx.initCause(var11);
        throw sqlEx;
    }
    //略...

}
```

继续调用this.serverPrepare(sql);

```
public class ServerPreparedStatement extends PreparedStatement {
    //存放运行时参数的数组
    private ServerPreparedStatement.BindValue[] parameterBindings;
    //服务器预编译好的sql语句返回的serverStatementId
    private long serverStatementId;
    private void serverPrepare(String sql) throws SQLException {
        synchronized(this.connection.getMutex()) {
            MysqlIO mysql = this.connection.getIO();
            try {
                //向sql服务器发送了一条PREPARE指令
                Buffer prepareResultPacket = mysql.sendCommand(MysqlDefs.COM_PREPARE, sql, (Buffer)null, false, characterEncoding, 0);
                //记录下了预编译好的sql语句所对应的serverStatementId
                this.serverStatementId = prepareResultPacket.readLong();
                this.fieldCount = prepareResultPacket.readInt();
                //获取参数个数，比喻 select * from user where id= ？and name = ？，其中有两个？，则这里返回的参数个数应该为2
                this.parameterCount = prepareResultPacket.readInt();
                this.parameterBindings = new ServerPreparedStatement.BindValue[this.parameterCount];

                for(int i = 0; i < this.parameterCount; ++i) {
                    //根据参数个数，初始化数组
                    this.parameterBindings[i] = new ServerPreparedStatement.BindValue();
                }

            } catch (SQLException var16) {
                throw sqlEx;
            } finally {
                this.connection.getIO().clearInputStream();
            }

        }
    }
}
```



```
ServerPreparedStatement继承PreparedStatement，ServerPreparedStatement初始化的时候就向sql服务器发送了一条PREPARE指令，把SQL语句传到mysql服务器，如select * from user where id= ？and name = ？，mysql服务器会对sql进行编译，并保存在服务器，返回预编译语句对应的id，并保存在
ServerPreparedStatement中，同时创建BindValue[] parameterBindings数组，后面设置参数就直接添加到此数组中。好了，此时我们创建了一个ServerPreparedStatement并返回，下面就是设置运行时参数了
```



## 设置运行时参数到 SQL 中



### 我们已经获取到了PreparedStatement，接下来就是将运行时参数设置到PreparedStatement中，如下代码

```
handler.parameterize(stmt);
```

JDBC是怎么设置的呢？我们看看上面的例子，很简单吧

```
psmt = conn.prepareStatement(sql);
//设置参数
psmt.setString(1, username);
psmt.setString(2, password);
```

我们来看看parameterize方法



```
public void parameterize(Statement statement) throws SQLException {
    // 通过参数处理器 ParameterHandler 设置运行时参数到 PreparedStatement 中
    parameterHandler.setParameters((PreparedStatement) statement);
}

public class DefaultParameterHandler implements ParameterHandler {
    private final TypeHandlerRegistry typeHandlerRegistry;
    private final MappedStatement mappedStatement;
    private final Object parameterObject;
    private final BoundSql boundSql;
    private final Configuration configuration;

    public void setParameters(PreparedStatement ps) {
        /*
         * 从 BoundSql 中获取 ParameterMapping 列表，每个 ParameterMapping 与原始 SQL 中的 #{xxx} 占位符一一对应
         */
        List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
        if (parameterMappings != null) {
            for (int i = 0; i < parameterMappings.size(); i++) {
                ParameterMapping parameterMapping = parameterMappings.get(i);
                if (parameterMapping.getMode() != ParameterMode.OUT) {
                    Object value;
                    // 获取属性名
                    String propertyName = parameterMapping.getProperty();
                    if (boundSql.hasAdditionalParameter(propertyName)) {
                        value = boundSql.getAdditionalParameter(propertyName);
                    } else if (parameterObject == null) {
                        value = null;
                    } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                        value = parameterObject;
                    } else {
                        // 为用户传入的参数 parameterObject 创建元信息对象
                        MetaObject metaObject = configuration.newMetaObject(parameterObject);
                        // 从用户传入的参数中获取 propertyName 对应的值
                        value = metaObject.getValue(propertyName);
                    }

                    TypeHandler typeHandler = parameterMapping.getTypeHandler();
                    JdbcType jdbcType = parameterMapping.getJdbcType();
                    if (value == null && jdbcType == null) {
                        jdbcType = configuration.getJdbcTypeForNull();
                    }
                    try {
                        // 由类型处理器 typeHandler 向 ParameterHandler 设置参数
                        typeHandler.setParameter(ps, i + 1, value, jdbcType);
                    } catch (TypeException e) {
                        throw new TypeException(...);
                    } catch (SQLException e) {
                        throw new TypeException(...);
                    }
                }
            }
        }
    }
}
```



首先从**boundSql中获取****parameterMappings 集合，这块大家可以看看我前面的文章，然后遍历获取** **parameterMapping中的****propertyName ，如**#{name} 中的name,然后从运行时参数**parameterObject中获取name对应的参数值，最后设置到**PreparedStatement 中，我们主要来看是如何设置参数的。也就是

typeHandler.setParameter(ps, i + 1, value, jdbcType);，这句代码最终会向我们例子中一样执行，如下

```
public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
    ps.setString(i, parameter);
}
```

还记得我们的PreparedStatement是什么吗？是**ServerPreparedStatement，那我们就来看看\**ServerPreparedStatement的\******setString方法**



```
public void setString(int parameterIndex, String x) throws SQLException {
    this.checkClosed();
    if (x == null) {
        this.setNull(parameterIndex, 1);
    } else {
        //根据参数下标从parameterBindings数组总获取BindValue
        ServerPreparedStatement.BindValue binding = this.getBinding(parameterIndex, false);
        this.setType(binding, this.stringTypeCode);
        //设置参数值
        binding.value = x;
        binding.isNull = false;
        binding.isLongData = false;
    }

}

protected ServerPreparedStatement.BindValue getBinding(int parameterIndex, boolean forLongData) throws SQLException {
    this.checkClosed();
    if (this.parameterBindings.length == 0) {
        throw SQLError.createSQLException(Messages.getString("ServerPreparedStatement.8"), "S1009", this.getExceptionInterceptor());
    } else {
        --parameterIndex;
        if (parameterIndex >= 0 && parameterIndex < this.parameterBindings.length) {
            if (this.parameterBindings[parameterIndex] == null) {
                this.parameterBindings[parameterIndex] = new ServerPreparedStatement.BindValue();
            } else if (this.parameterBindings[parameterIndex].isLongData && !forLongData) {
                this.detectedLongParameterSwitch = true;
            }

            this.parameterBindings[parameterIndex].isSet = true;
            this.parameterBindings[parameterIndex].boundBeforeExecutionNum = (long)this.numberOfExecutions;
            //根据参数下标从parameterBindings数组总获取BindValue
            return this.parameterBindings[parameterIndex];
        } else {
            throw SQLError.createSQLException(Messages.getString("ServerPreparedStatement.9") + (parameterIndex + 1) + Messages.getString("ServerPreparedStatement.10") + this.parameterBindings.length, "S1009", this.getExceptionInterceptor());
        }
    }
}
```



```
就是根据参数下标从ServerPreparedStatement的参数数组parameterBindings中获取BindValue对象，然后设置值，好了现在ServerPreparedStatement包含了预编译SQL语句的Id和参数数组，最后一步便是执行SQL了。
```



## 执行查询

执行查询操作就是我们文章开头的最后一行代码，如下

```
return handler.<E>query(stmt, resultHandler);
```

我们来看看query是怎么做的



```
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement)statement;
    //直接执行ServerPreparedStatement的execute方法
    ps.execute();
    return this.resultSetHandler.handleResultSets(ps);
}

public boolean execute() throws SQLException {
    this.checkClosed();
    ConnectionImpl locallyScopedConn = this.connection;
    if (!this.checkReadOnlySafeStatement()) {
        throw SQLError.createSQLException(Messages.getString("PreparedStatement.20") + Messages.getString("PreparedStatement.21"), "S1009", this.getExceptionInterceptor());
    } else {
        ResultSetInternalMethods rs = null;
        CachedResultSetMetaData cachedMetadata = null;
        synchronized(locallyScopedConn.getMutex()) {
            //略....
            rs = this.executeInternal(rowLimit, sendPacket, doStreaming, this.firstCharOfStmt == 'S', metadataFromCache, false);
            //略....
        }

        return rs != null && rs.reallyResult();
    }
}
```



省略了很多代码，只看最关键的**executeInternal**

**ServerPreparedStatement**



```
protected ResultSetInternalMethods executeInternal(int maxRowsToRetrieve, Buffer sendPacket, boolean createStreamingResultSet, boolean queryIsSelectOnly, Field[] metadataFromCache, boolean isBatch) throws SQLException {
    try {
        return this.serverExecute(maxRowsToRetrieve, createStreamingResultSet, metadataFromCache);
    } catch (SQLException var11) {
        throw sqlEx;
    } 
}

private ResultSetInternalMethods serverExecute(int maxRowsToRetrieve, boolean createStreamingResultSet, Field[] metadataFromCache) throws SQLException {
    synchronized(this.connection.getMutex()) {
        //略....
        MysqlIO mysql = this.connection.getIO();
        Buffer packet = mysql.getSharedSendPacket();
        packet.clear();
        packet.writeByte((byte)MysqlDefs.COM_EXECUTE);
        //将该语句对应的id写入数据包
        packet.writeLong(this.serverStatementId);

        int i;
        //将对应的参数写入数据包
        for(i = 0; i < this.parameterCount; ++i) {
            if (!this.parameterBindings[i].isLongData) {
                if (!this.parameterBindings[i].isNull) {
                    this.storeBinding(packet, this.parameterBindings[i], mysql);
                } else {
                    nullBitsBuffer[i / 8] = (byte)(nullBitsBuffer[i / 8] | 1 << (i & 7));
                }
            }
        }
        //发送数据包,表示执行id对应的预编译sql
        Buffer resultPacket = mysql.sendCommand(MysqlDefs.COM_EXECUTE, (String)null, packet, false, (String)null, 0);
        //略....
        ResultSetImpl rs = mysql.readAllResults(this,  this.resultSetType,  resultPacket, true, (long)this.fieldCount, metadataFromCache);
        //返回结果
        return rs;
    }
}
```



ServerPreparedStatement在记录下serverStatementId后，对于相同SQL模板的操作，每次只是发送serverStatementId和对应的参数，省去了编译sql的过程。 至此我们的已经从数据库拿到了查询结果，但是结果是ResultSetImpl类型，我们还需要将返回结果转化成我们的java对象呢，留在下一篇来讲吧
