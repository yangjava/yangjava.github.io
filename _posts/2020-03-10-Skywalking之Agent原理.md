---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---

## Java Agent的原理

### 概述

- 我们知道，要使用Skywalking去监控服务，需要在其VM参数中添加“`-javaagent:/usr/local/skywalking/apache-skywalking-apm-bin-es7/agent/skywalking-agent.jar`”，这里就使用到了java agent技术。

### java agent是什么？

- java agent是java命令的一个参数，参数`javaagent`可以用于指定一个jar包。

- - 这个jar包的MANIFEST.MF文件必须指定Premain-Class项。

- - Premain-Class指定的那个类必须实现premain()方法。

- 当Java虚拟机启动时，在执行main函数之前，JVM会运行`-javaagent`所指定的jar包内Premain-Class这个类的premain方法。

### 如何使用java agent？

- 定义一个MANIFEST.MF文件，必须包含Premain-Class选项，通常也会加入Can-Redefine-Classes和Can-Retransform-Classes选项。

- 创建一个Premain-Class指定的类，类中包含premain方法，方法逻辑由用户自己确定。

- 将premain类和MANIFEST.MF文件打成jar包。

- 使用参数`-javaagent:jar包路径`启动要代理的方法。

### 搭建java agent工程

- 使用Maven搭建java_agent总工程，其中java_agent_demo子工程是真正的逻辑实现，而java_agent_user子工程是测试。

- 在java_agent_demo的工程中引入maven-assembly-plugin插件，用于将java_agent_demo工程打成符合java agent标准的jar包：

```xml
<build>
    <plugins>
        <plugin>
            <artifactId>maven-assembly-plugin</artifactId>
            <configuration>
                <appendAssemblyId>false</appendAssemblyId>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
                <archive>
                    <!--自动添加META-INF/MANIFEST.MF -->
                    <manifest>
                        <addClasspath>true</addClasspath>
                    </manifest>
                    <manifestEntries>
                        <Premain-Class>PreMainAgent</Premain-Class>
                        <Agent-Class>PreMainAgent</Agent-Class>
                        <Can-Redefine-Classes>true</Can-Redefine-Classes>
                        <Can-Retransform-Classes>true</Can-Retransform-Classes>
                    </manifestEntries>
                </archive>
            </configuration>
            <executions>
                <execution>
                    <id>make-assembly</id>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

- 在java_agent_demo工程创建PreMainAgent类。

```java
import java.lang.instrument.Instrumentation;

public class PreMainAgent {

    /**
     * 在这个 premain 函数中，开发者可以进行对类的各种操作。
     * 1、agentArgs 是 premain 函数得到的程序参数，随同 “– javaagent”一起传入。与 main 函数不同的是，
     * 这个参数是一个字符串而不是一个字符串数组，如果程序参数有多个，程序将自行解析这个字符串。
     * 2、Inst 是一个 java.lang.instrument.Instrumentation 的实例，由 JVM 自动传入。*
     * java.lang.instrument.Instrumentation 是 instrument 包中定义的一个接口，也是这个包的核心部分，
     * 集中了其中几乎所有的功能方法，例如类定义的转换和操作等等。
     *
     * @param agentArgs
     * @param inst
     */
    public static void premain(String agentArgs, Instrumentation inst) {
        System.out.println("=========premain方法执行1========");
        System.out.println(agentArgs);
    }

    /**
     * 如果不存在 premain(String agentArgs, Instrumentation inst)
     * 则会执行 premain(String agentArgs)
     *
     * @param agentArgs
     */
    public static void premain(String agentArgs) {
        System.out.println("=========premain方法执行2========");
        System.out.println(agentArgs);
    }

}
```

- 通过IDEA，进行打包。

- java_agent_user工程新建一个测试类。

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("你好 世界");
    }
}
```



- 先运行一次，然后点击编辑MAIN启动类：

- 在VM Options中添加代码：

```shell
-javaagent:D:/project/java_agent/java_agent_demo/target/java_agent_demo-1.0.jar
```

- 启动的时候加载javaagent，指向java_agent_demo工程编译出来的javaagent的jar包地址。

### 统计方法的调用时间

- Skywalking中对每个调用的时长都进行了统计，我们要使用ByteBuddy和java agent技术来统计方法的调用时长。

- Byte Buddy是开源的、基于Apache2.0许可证的库，它致力于解决字节码操作和instrumentation API的复杂性。Byte Buddy所声称的目标是将显示的字节码操作隐藏在一个类型安全的领域特定语言背后，通过Byte Buddy，任何熟悉Java编程语言的人都有望非常容易的进行字节码的操作。Byte Buddy提供了额外的依赖API来生成java agent，可以轻松的增加我们已有的代码。

- 需要在java_agent_demo工程中添加如下的依赖：



```xml
<dependency>
    <groupId>net.bytebuddy</groupId>
    <artifactId>byte-buddy</artifactId>
    <version>1.9.2</version>
</dependency>
<dependency>
    <groupId>net.bytebuddy</groupId>
    <artifactId>byte-buddy-agent</artifactId>
    <version>1.9.2</version>
</dependency>
```

- 新建一个MyInterceptor的类，用来统计调用的时长。

```java
import net.bytebuddy.implementation.bind.annotation.Origin;
import net.bytebuddy.implementation.bind.annotation.RuntimeType;
import net.bytebuddy.implementation.bind.annotation.SuperCall;

import java.lang.reflect.Method;
import java.util.concurrent.Callable;

public class MyInterceptor {
    @RuntimeType
    public static Object intercept(@Origin Method method,
                                   @SuperCall Callable<?> callable)
            throws Exception {
        long start = System.currentTimeMillis();
        try {
            //执行原方法
            return callable.call();
        } finally {
            //打印调用时长
            System.out.println(method.getName() + ":" + (System.currentTimeMillis() - start)  + "ms");
        }
    }
}
```

- 修改PreMainAgent的代码：

```java
import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.description.method.MethodDescription;
import net.bytebuddy.description.type.TypeDescription;
import net.bytebuddy.dynamic.DynamicType;
import net.bytebuddy.implementation.MethodDelegation;
import net.bytebuddy.matcher.ElementMatchers;
import net.bytebuddy.utility.JavaModule;

import java.lang.instrument.Instrumentation;

public class PreMainAgent {

    public static void premain(String agentArgs, Instrumentation inst) {
        //创建一个转换器，转换器可以修改类的实现
        //ByteBuddy对java agent提供了转换器的实现，直接使用即可
        AgentBuilder.Transformer transformer = new AgentBuilder.Transformer() {
            public DynamicType.Builder<?> transform(DynamicType.Builder<?> builder, TypeDescription typeDescription, ClassLoader classLoader, JavaModule javaModule) {
                return builder
                        // 拦截任意方法
                        .method(ElementMatchers.<MethodDescription>any())
                        // 拦截到的方法委托给TimeInterceptor
                        .intercept(MethodDelegation.to(MyInterceptor.class));
            }
        };
        new AgentBuilder // Byte Buddy专门有个AgentBuilder来处理Java Agent的场景
                .Default()
                // 根据包名前缀拦截类
                .type(ElementMatchers.nameStartsWith("com.agent"))
                // 拦截到的类由transformer处理
                .transform(transformer)
                .installOn(inst);
    }
}
```

- 对java_agent_demo工程重新打包。

- 将java_agent_user工程中的Main类方法com.agent包下，修改代码的内容为：

```java
package com.agent;

/**
 * @author 许大仙
 * @version 1.0
 * @since 2021-01-21 08:46
 */
public class Main {
    public static void main(String[] args) {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("你好 世界");
    }
}
```

- 执行Main方法之后的显示结果为

- 我们在没有修改代码的情况下，利用了java agent和Byte Buddy统计出了方法的时长，Skywalking的agent也是基于这些技术来实现统计时长的调用的。