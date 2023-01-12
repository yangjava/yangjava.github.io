## 手把手教你编写Skywalking插件

## 前置知识

在正式进入编写环节之前，建议先花一点时间了解下javaagent（这是JDK 5引入的一个玩意儿，最好了解下其工作原理）；另外，Skywalking用到了byte-buddy（一个动态操作二进制码的库），所以最好也熟悉下。

当然不了解关系也不大，一般不影响你玩转Skywalking。

- [javaagent](https://link.segmentfault.com/?enc=%2B%2FD%2FVP6HHVJ4el9DaVjZNw%3D%3D.lx4ydyf0tEq5oKtQPtYgZOWikMOWE4mhV7NWdRzbGHdnU3NdKIDw2AS%2FQVp5%2Fn1GjzZ1YRJI40L9qZ5fn%2F%2BY%2FA%3D%3D)
- [byte-buddy 1.9.6 简述及原理1](https://link.segmentfault.com/?enc=c5izOPDlNaCoUUGV%2F89E4A%3D%3D.UWXceWWKJBI45oP1W%2BYy%2Befi3cr3cxSxclLSjr5UdQRFscYeyY44QVIpp0VjIAaDo4XYgN0sAU4OmKv1iYYzaw%3D%3D)

## 术语

Span:可理解为一次方法调用，一个程序块的调用，或一次RPC/数据库访问。只要是一个具有完整时间周期的程序访问，都可以被认为是一个span。SkyWalking `Span` 对象中的重要属性

| 属性          | 名称     | 备注                                                       |
| ------------- | -------- | ---------------------------------------------------------- |
| component     | 组件     | 插件的组件名称，如：Lettuce，详见:ComponentsDefine.Class。 |
| tag           | 标签     | k-v结构，关键标签，key详见：Tags.Class。                   |
| peer          | 对端资源 | 用于拓扑图，若DB组件，需记录集群信息。                     |
| operationName | 操作名称 | 若span=0，operationName将会搜索的下拉列表。                |
| layer         | 显示     | 在链路页显示，详见SpanLayer.Class。                        |

Trace:调用链，通过归属于其的Span来隐性的定义。一条Trace可被认为是一个由多个Span组成的有向无环图（DAG图），在SkyWalking链路模块你可以看到，Trace又由多个归属于其的trace segment组成。

Trace segment:Segment是SkyWalking中的一个概念，它应该包括单个OS进程中每个请求的所有范围，通常是基于语言的单线程。由多个归属于本线程操作的Span组成。

> **TIPS**
> Skywalking的这几个术语和Spring Cloud Sleuth类似，借鉴自谷歌的Dapper。我在 [《Spring Cloud Alibaba微服务从入门到进阶》](https://link.segmentfault.com/?enc=7TNjSh16NKm4l8SB0OWmwg%3D%3D.I3ThFmv4hAmBWJ2GEBboW4%2BBJAbcNE0DxxW8eyZxHA8mU3xYb6AZ0Y2JQSVyNg7D) 课程中有详细剖析调用链的实现原理，并且用数据库做了通俗的类比，本文不再赘述。

## 核心API

详见 [http://www.itmuch.com/books/skywalking/guides/Java-Plugin-Development-Guide.html](https://link.segmentfault.com/?enc=nfEe3JoFhIpqOU%2BWTsEEnA%3D%3D.9N0X5D1vPkXRLQQw2rIM8lXU12eaWmKtzKB%2BN5PG2sU5U%2B6OlrhGGASmmQYIaMz3yZms86YhVTuUDGfSxNjEk43hFe3Tc8fhEHEwuQlET01Dw5BzW3RC2NPwWCAQcqy1) 文章有非常详细的描述。

## 实战

本文以监控 `org.apache.commons.lang3.StringUtils.replace` 为例，手把手教你编写Skywalking插件。

### 依赖

首先，创建一个Maven项目，Pom.xml如下。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.itmuch.skywalking</groupId>
    <artifactId>apm-string-replace-plugin</artifactId>
    <packaging>jar</packaging>
    <version>1.0.0-SNAPSHOT</version>

    <properties>
        <skywalking.version>6.6.0</skywalking.version>
        <shade.package>org.apache.skywalking.apm.dependencies</shade.package>
        <shade.net.bytebuddy.source>net.bytebuddy</shade.net.bytebuddy.source>
        <shade.net.bytebuddy.target>${shade.package}.${shade.net.bytebuddy.source}</shade.net.bytebuddy.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.skywalking</groupId>
            <artifactId>apm-agent-core</artifactId>
            <version>${skywalking.version}</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.skywalking</groupId>
            <artifactId>apm-util</artifactId>
            <version>${skywalking.version}</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.4</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <artifactId>maven-shade-plugin</artifactId>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <shadedArtifactAttached>false</shadedArtifactAttached>
                            <createDependencyReducedPom>true</createDependencyReducedPom>
                            <createSourcesJar>true</createSourcesJar>
                            <shadeSourcesContent>true</shadeSourcesContent>
                            <relocations>
                                <relocation>
                                    <pattern>${shade.net.bytebuddy.source}</pattern>
                                    <shadedPattern>${shade.net.bytebuddy.target}</shadedPattern>
                                </relocation>
                            </relocations>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>6</source>
                    <target>6</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

这个Pom.xml中，除如下依赖以外，其他都得照抄，不管你开发什么框架的Skywalking插件，**否则无法正常构建插件**！！

```xml
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-lang3</artifactId>
  <version>3.4</version>
  <scope>provided</scope>
</dependency>
```

### 编写Instrumentation类

```java
public class StringReplaceInstrumentation extends ClassInstanceMethodsEnhancePluginDefine {
    @Override
    protected ClassMatch enhanceClass() {
        // 指定想要监控的类
        return NameMatch.byName("org.apache.commons.lang3.StringUtils");
    }

    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return new ConstructorInterceptPoint[0];
    }

    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        // 指定想要监控的实例方法，每个实例方法对应一个InstanceMethodsInterceptPoint
        return new InstanceMethodsInterceptPoint[0];
    }

    @Override
    public StaticMethodsInterceptPoint[] getStaticMethodsInterceptPoints() {
        // 指定想要监控的静态方法，每一个方法对应一个StaticMethodsInterceptPoint
        return new StaticMethodsInterceptPoint[]{
            new StaticMethodsInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    // 静态方法名称
                    return ElementMatchers.named("replace");
                }

                @Override
                public String getMethodsInterceptor() {
                    // 该静态方法的监控拦截器类名全路径
                    return "com.itmuch.skywalking.plugin.stringreplace.StringReplaceInterceptor";
                }

                @Override
                public boolean isOverrideArgs() {
                    return false;
                }
            }
        };
    }
}
```

### 编写拦截器

```java
public class StringReplaceInterceptor implements StaticMethodsAroundInterceptor {
    @Override
    public void beforeMethod(Class aClass, Method method, Object[] argumentsTypes, Class<?>[] classes, MethodInterceptResult methodInterceptResult) {
        // 创建span(监控的开始)，本质上是往ThreadLocal对象里面设值
        AbstractSpan span = ContextManager.createLocalSpan("replace");

        /*
         * 可用ComponentsDefine工具类指定Skywalking官方支持的组件
         * 也可自己new OfficialComponent或者Component
         * 不过在Skywalking的控制台上不会被识别，只会显示N/A
         */
        span.setComponent(ComponentsDefine.TOMCAT);

        span.tag(new StringTag(1000, "params"), argumentsTypes[0].toString());
        // 指定该调用的layer，layer是个枚举
        span.setLayer(SpanLayer.CACHE);
    }

    @Override
    public Object afterMethod(Class aClass, Method method, Object[] objects, Class<?>[] classes, Object o) {
        String retString = (String) o;
        // 激活span，本质上是读取ThreadLocal对象
        AbstractSpan span = ContextManager.activeSpan();
        // 状态码，任意写，Tags也是个Skywalking的工具类，用来比较方便地操作tag
        Tags.STATUS_CODE.set(span, "20000");

        // 停止span(监控的结束)，本质上是清理ThreadLocal对象
        ContextManager.stopSpan();
        return retString;
    }

    @Override
    public void handleMethodException(Class aClass, Method method, Object[] objects, Class<?>[] classes, Throwable throwable) {
        AbstractSpan activeSpan = ContextManager.activeSpan();
        
        // 记录日志
        activeSpan.log(throwable);
        activeSpan.errorOccurred();
    }
}
```

### 编写配置文件

创建`resources/skywalking-plugin.def` ，内容如下：

```applescript
# Key=value的形式
# key随便写；value是Instrumentation类的包名类名全路径
my-string-replace-
plugin=org.apache.skywalking.apm.plugin.stringreplace.define.StringReplaceInstrumentation
```

### 构建

```armasm
mvn clean install
```

构建完成后，到target目录中，找到JAR包(非 `origin-xxx.jar` 、非`xxx-source.jar` )，扔到 `agent/plugins` 目录里面去，即可启动。

### 插件调试

插件的编写可能不是一步到位的，有时候可能会报点错什么的。如果想要Debug自己的插件，那么需要将你的插件代码和接入Java Agent的项目（也就是你配置了-javaagent启动的项目）扔到同一个工作空间内，可以这么玩：

- 使用IDEA，打开接入Java Agent的项目
- 找到File->New->Module from Exisiting Sources…，引入你的插件源码即可。

### 测试与监控效果

想办法让你接入Java Agent的项目，调用到如下代码

```java
String replace = StringUtils.replace("oldString", "old","replaced");
System.out.println(replace);
```

之后，就可以看到类似如下的图啦：

## 写在最后

本文只是弄了一个简单的例子，讲解插件编写的套路。总的来说，插件编写还是非常顺利的，单纯代码的层面，很少会遇到坑；但搜遍各种搜索引擎，竟然没有一篇手把手的文章…而且也没有文章讲解依赖该如何引入、Maven插件如何引入。于是只好参考Skywalking官方插件的写法，引入依赖和Maven插件了，这块反倒是费了点时间。

此外，如果你想真正掌握乃至精通skywalking插件编写，最好的办法，还是阅读官方的插件代码，详见：`https://github.com/apache/skywalking/tree/master/apm-sniffer/apm-sdk-plugin` ，随便挑两款看一下就知道怎么玩了。我在学习过程中参考的插件代码有：

- apm-feign-default-http-9.x-plugin
- apm-jdbc-commons