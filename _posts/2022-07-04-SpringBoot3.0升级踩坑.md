---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot3.0
---
# SpringBoot3.0升级指南
如果您当前运行的是 Spring Boot 的早期版本，官方强烈建议您在迁移到 Spring Boot 3.0 之前升级到 Spring Boot 2.7。

官方整理了一份专门的迁移指南(https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide)，以帮助您升级现有的 Spring Boot 2.7 应用程序。

## 功能变化
- 基于Java 17 和支持Java 19
Spring Boot 3.0 需要 Java 17 作为最低版本。 如果您当前使用的是 Java 8 或 Java 11，则需要先升级 JDK，然后才能开发 Spring Boot 3.0 应用程序。
- 第三方库升级
Spring Boot 3.0 建立在 Spring Framework 6 之上，并且需要 Spring Framework 6。



## 升级指南
Maven依赖管理
```xml
<dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>3.0.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```
### Spring Framework 6.0
Spring框架升级到6.0

### servlet升级
编译报错，import javax.servlet.*; 不存在。这个报错主要是Spring Boot3.0已经为所有依赖项从 Java EE 迁移到 Jakarta EE API，导致 servlet 包名的修改，Spring团队这样做的原因，主要是避免 Oracle 的版权问题
添加 jakarta.servlet 依赖
```xml
<dependency>
    <groupId>jakarta.servlet</groupId>
    <artifactId>jakarta.servlet-api</artifactId>
</dependency>
```
修改项目内所有代码的导入依赖修改`import javax.servlet.*`为`import jakarta.servlet.*`

### apollo配置升级
使用新的apollo版本号`2.0.0`
```xml
       <dependency>
            <groupId>com.ctrip.framework.apollo</groupId>
            <artifactId>apollo-client</artifactId>
            <version>${apollo-client.version}</version>
        </dependency>
```

### Druid数据源



二. 附带的众多依赖包升级，导致的部分代码写法过期报警
2.1 Thymeleaf升级到3.1.0.M2，日志打印的报警
14:40:39.936 [http-nio-84-exec-15] WARN  o.t.s.p.StandardIncludeTagProcessor - [doProcess,67] - [THYMELEAF][http-nio-84-exec-15][admin/goods/goods] Deprecated attribute {th:include,data-th-include} found in template admin/goods/goods, line 4, col 15. Please use {th:insert,data-th-insert} instead, this deprecated attribute will be removed in future versions of Thymeleaf.
14:40:39.936 [http-nio-84-exec-15] WARN  o.t.s.p.AbstractStandardFragmentInsertionTagProcessor - [computeFragment,385] - [THYMELEAF][http-nio-84-exec-15][admin/goods/goods] Deprecated unwrapped fragment expression "admin/header :: header-fragment" found in template admin/goods/goods, line 4, col 15. Please use the complete syntax of fragment expressions instead ("~{admin/header :: header-fragment}"). The old, unwrapped syntax for fragment expressions will be removed in future versions of Thymeleaf.
复制
可以看出作者很贴心，日志里已经给出了升级后的写法，修改如下：

修改前：
<th:block th:include="admin/header :: header-fragment"/>
修改后：
<th:block th:insert="~{admin/header :: header-fragment}"/>
复制
2.2 Thymeleaf升级到3.1.0.M2，后端使用 thymeleafViewResolver 手动渲染网页代码报错
// 修改前 Spring Boot2.7：
WebContext ctx = new (request, response,
request.getServletContext(), request.getLocale(), model.asMap());
html = thymeleafViewResolver.getTemplateEngine().process("mall/seckill-list", ctx);
复制
上述代码中针对 WebContext 对象的创建报错，这里直接给出新版写法

// 修改后 Spring Boot3.0：
JakartaServletWebApplication jakartaServletWebApplication = JakartaServletWebApplication.buildApplication(request.getServletContext());
WebContext ctx = new WebContext(jakartaServletWebApplication.buildExchange(request, response), request.getLocale(), model.asMap());
html = thymeleafViewResolver.getTemplateEngine().process("mall/seckill-list", ctx);
复制
三. 大量第三方库关于 Spring Boot 的 starter 依赖失效，导致项目启动报错
博主升级到3.0后，发现启动时，Druid 数据源开始报错，找不到数据源配置，便怀疑跟 Spring boot 3.0 更新有关

这里直接给出原因：Spring Boot 3.0 中自动配置注册的 spring.factories 写法已废弃，改为了 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 写法，导致大量第三方 starter 依赖失效

在吐槽一下，这么重要的更改在Spring官方的 Spring-Boot-3.0-发布说明 中竟然没有，被放在了 Spring-Boot-3.0.0-M5-发布说明 中

这里给出两个解决方案：

等待第三方库适配 Spring Boot 3.0
按照 Spring Boot 3.0要求，在项目resources 下新建 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 文件，手动将第三方库的 spring.factories 加到 imports 中，这样可以手动修复第三方库 spring boot starter 依赖失效问题
四. Mybatis Plus 依赖问题
Mybatis plus 最新版本还是3.5.2，其依赖的 mybatis-spring 版本是2.2.2（mybatis-spring 已经发布了3.0.0版本适配 Spring Boot 3.0），这会导致项目中的sql查询直接报错，这里主要是因 Spring Boot 3.0中删除 NestedIOException 这个类，在 Spring boot 2.7中这个类还存在，给出类说明截图


image.png
这个类在2.7中已经被标记为废弃，建议替换为 IOException， 而 Mybatis plus 3.5.2中还在使用。这里给出问题截图 MybatisSqlSessionFactoryBean 这个类还在使用 NestedIOException


image.png
查看 Mybatis plus 官方issue也已经有人提到了这个问题，官方的说法是 mybatis-plus-spring-boot-starter 还在验证尚未推送maven官方仓库，这里我就不得不动用我的小聪明，给出解决方案：

手动将原有的 MybatisSqlSessionFactoryBean 类代码复制到一个我们自己代码目录下新建的 MybatisSqlSessionFactoryBean 类，去掉 NestedIOException 依赖
数据源自动配置代码修改
@Slf4j
@EnableConfigurationProperties(MybatisPlusProperties.class)
@EnableTransactionManagement
@EnableAspectJAutoProxy
@Configuration
@MapperScan(basePackages = "ltd.newbee.mall.core.dao", sqlSessionFactoryRef = "masterSqlSessionFactory")
public class HikariCpConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }


    @Bean(name = "masterDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public DataSource masterDataSource() {
        return new HikariDataSource();
    }

    /**
     * @param datasource 数据源
     * @return SqlSessionFactory
     * @Primary 默认SqlSessionFactory
     */
    @Bean(name = "masterSqlSessionFactory")
    public SqlSessionFactory masterSqlSessionFactory(@Qualifier("masterDataSource") DataSource datasource,
                                                     Interceptor interceptor,
                                                     MybatisPlusProperties properties) throws Exception {
        MybatisSqlSessionFactoryBean bean = new MybatisSqlSessionFactoryBean();
        bean.setDataSource(datasource);
        // 兼容mybatis plus的自动配置写法
        bean.setMapperLocations(properties.resolveMapperLocations());
        if (properties.getConfigurationProperties() != null) {
            bean.setConfigurationProperties(properties.getConfigurationProperties());
        }
        if (StringUtils.hasLength(properties.getTypeAliasesPackage())) {
            bean.setTypeAliasesPackage(properties.getTypeAliasesPackage());
        }
        bean.setPlugins(interceptor);
        GlobalConfig globalConfig = properties.getGlobalConfig();
        bean.setGlobalConfig(globalConfig);
        log.info("------------------------------------------masterDataSource 配置成功");
        return bean.getObject();
    }

    @Bean("masterSessionTemplate")
    public SqlSessionTemplate masterSessionTemplate(@Qualifier("masterSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

}
复制
到这里，项目就能够正常跑起来了

总结
Spring Boot 3.0 升级带来了很多破坏性更改，把众多依赖升级到了最新，算是解决了一部分历史问题，也为了云原型需求，逐步适配 graalvm ，不管怎么样作为技术开发者，希望有更多的开发者来尝试 Spring Boot 3.0 带来的新变化。


书接上文再 Spring Boot3.0升级，踩坑之旅，附解决方案 第一篇中我们介绍了大部分 Spring Boot3.0  升级所带来的破坏性修改，这篇文章将介绍剩下的修改部分，并针对Jdk17升级带来的优化写法进行案例展示。

本文基于 newbeemall 项目升级Spring Boot3.0踩坑总结而来

一。Jdk8中内置的JavaScript引擎 nashorn 被移除，导致验证码使用报错Cannot invoke "javax.script.ScriptEngine.eval(String)" because "engine" is null
项目中使用了 com.github.whvcse包的easy-captcha 验证码依赖，升级至Jdk17后，验证码接口报错：Cannot invoke "javax.script.ScriptEngine.eval(String)" because "engine" is null，错误原因很明显脚本引擎执行脚本语句报错，因为执行引擎为空。查询相关资料Jdk8自带的JavaScript引擎 nashorn 再升级到Jdk9后就被移除了，从而导致报错

解决办法：添加JavaScript引擎 nashorn依赖

<dependency>
    <groupId>org.openjdk.nashorn</groupId>
    <artifactId>nashorn-core</artifactId>
    <version>15.4</version>
</dependency>
复制
二. Spring data redis 配置前缀被修改
在 Spring Boot2.0 中 redis的配置前缀为 spring.redis


image.png
但是在最新 Spring Boot3.0 中redis的配置前缀被修改为 spring.data.redis，导致老项目中redis配置写法需要修改，如下图


image.png
这里我们猜测一波，Spring 对redis配置的破坏性修改可能是为了统一 Spring data 配置把。

解决办法，修改redis配置前缀为 spring.data.redis

三. 升级Jdk17的优化一些写法
3.1 文本块语法。再很多其他语言中早就支持的文本块写法，现在在Jdk17中也可以通过 """ 语法使用啦，如下，针对一段 lua 脚本代码，我们再也不用通过字符串拼接了
private String buildLuaScript() {
return "local c" +
"\nc = redis.call('get',KEYS[1])" +
"\nif c and tonumber(c) > tonumber(ARGV[1]) then" +
"\nreturn c;" +
"\nend" +
"\nc = redis.call('incr',KEYS[1])" +
"\nif tonumber(c) == 1 then" +
"\nredis.call('expire',KEYS[1],ARGV[2])" +
"\nend" +
"\nreturn c;";
}
复制
文本块写法，代码可读性提高了一个档次

private String buildLuaScript() {
return """
local c
c = redis.call('get',KEYS[1])
if c and tonumber(c) > tonumber(ARGV[1]) then
return c;
end
c = redis.call('incr',KEYS[1])
if tonumber(c) == 1 then
redis.call('expire',KEYS[1],ARGV[2])
end
return c;""";
}
复制
3.2 instanceof 模式匹配
Jdk17中针对 instanceof 关键字支持模式变量定义，可以减少不必要的强制转换逻辑，如下

if (handler instanceof HandlerMethod) {
HandlerMethod handlerMethod = (HandlerMethod) handler;
Method method = handlerMethod.getMethod();
RepeatSubmit annotation = method.getAnnotation(RepeatSubmit.class);
if (annotation != null) {
if (this.isRepeatSubmit(request)) {
R error = R.error("不允许重复提交，请稍后再试");
ServletUtil.renderString(response, JSON.toJSONString(error));
return false;
}
}
}
复制
使用模式变量，可以消除代码中不必要的类型转换

if (handler instanceof HandlerMethod handlerMethod) {
Method method = handlerMethod.getMethod();
RepeatSubmit annotation = method.getAnnotation(RepeatSubmit.class);
if (annotation != null) {
if (this.isRepeatSubmit(request)) {
R error = R.error("不允许重复提交，请稍后再试");
ServletUtil.renderString(response, JSON.toJSONString(error));
return false;
}
}
}
复制
3.3 switch 表达式扩展
升级到Jdk17后支持 switch 表达式扩展写法，优化前的写法

public static String getExtension(String prefix) {
switch (prefix) {
case IMAGE_PNG:
return "png";
case IMAGE_JPG:
return "jpg";
case IMAGE_JPEG:
return "jpeg";
case IMAGE_BMP:
return "bmp";
case IMAGE_GIF:
return "gif";
default:
return "";
}
}
复制
使用 switch 表达式扩展

public static String getExtension(String prefix) {
return switch (prefix) {
case IMAGE_PNG -> "png";
case IMAGE_JPG -> "jpg";
case IMAGE_JPEG -> "jpeg";
case IMAGE_BMP -> "bmp";
case IMAGE_GIF -> "gif";
default -> "";
};
}
复制
总结
本文介绍了Spring Boot 3.0 升级带来了破坏性更改第二部分介绍，也展示了一些新版Jdk的优化写法，希望更多的朋友能够尝试升级到最新 Spring Boot 3.0，紧跟时代潮流

Spring Boot3.0升级，踩坑之旅，附解决方案