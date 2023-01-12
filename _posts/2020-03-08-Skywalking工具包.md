---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
# skywalking工具
skywalking提供工具包，用于提供链路追踪各种功能。

## 应用依赖

- 在pom.xml中增加Trace工具包的坐标：

```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-trace</artifactId>
    <version>8.3.0</version>
</dependency>
```

## 获取追踪的ID

- Skywalking提供了Trace工具包，用于在追踪链路的时候进行信息的打印或者获取对应的追踪ID。


- 修改Controller.java

```java
package com.example.springboot2.web;

import org.apache.skywalking.apm.toolkit.trace.ActiveSpan;
import org.apache.skywalking.apm.toolkit.trace.TraceContext;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;


@RestController
public class HelloController {

    @RequestMapping(value = "/hello")
    public String hello() {
        return "你好啊";
    }

    /**
     * TraceContext.traceId() 可以打印出当前追踪的ID，方便在Rocketbot中搜索
     * ActiveSpan提供了三个方法进行信息打印：
     *   error：会将本次调用转为失败状态，同时可以打印对应的堆栈信息和错误提示
     *   info：打印info级别额信息
     *   debug：打印debug级别的信息
     *
     * @return
     */
    @RequestMapping(value = "/exception")
    public String exception() {
        ActiveSpan.info("打印info信息");
        ActiveSpan.debug("打印debug信息");
        //使得当前的链路报错，并且提示报错信息
        try {
            int i = 10 / 0;
        } catch (Exception e) {
            ActiveSpan.error(new RuntimeException("报错了"));
            //返回trace id
            return TraceContext.traceId();
        }

        return "你怎么可以这个样子";
    }
}
```

- 将项目打包，部署，并使用如下的命令启动：

```shell
java -javaagent:/usr/local/skywalking/apache-skywalking-apm-bin-es7/agent/skywalking-agent.jar -Dserver.port=8082   -Dskywalking.agent.service_name=skywalking_mysql -jar springboot2-1.0.jar &
```

- 访问接口，获取追踪的ID：

- 可以搜索到对应的追踪记录，但是显示调用是失败的，这是因为使用了ActiveSpan.error方法，点开追踪的详细信息。

## 过滤指定的端点

- 在开发的过程中，有一些端点（接口）并不需要去进行监控，比如Swagger相关的端点。这个时候我们使用Skywalking提供的过滤插件来进行过滤。

### 应用示例

- 在上面的SpringBoot项目中增加如下的控制器：

```java
package com.example.springboot2.web;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;


@RestController
public class FilterController {

    /**
     * 此接口可以被追踪
     *
     * @return
     */
    @GetMapping(value = "/include")
    public String include() {
        return "include";
    }

    /**
     * 此接口不可以被追踪
     *
     * @return
     */
    @GetMapping(value = "/exclude")
    public String exclude() {
        return "exclude";
    }
}
```

- 将项目打包并上传到/usr/local/skywalking中。

- 将agent中的agent/optional-plugins/apm-trace-ignore-plugin-8.3.0.jar插件复制到plugins目录中。

```shell
cd /usr/local/skywalking/apache-skywalking-apm-bin-es7/agent
```

```shell
cp optional-plugins/apm-trace-ignore-plugin-8.3.0.jar plugins/
```

- 启动SpringBoot应用，并添加过滤参数。

```shell
java -javaagent:/usr/local/skywalking/apache-skywalking-apm-bin-es7/agent/skywalking-agent.jar -Dserver.port=8082 -Dskywalking.agent.service_name=skywalking_mysql -Dskywalking.trace.ignore_path=/exclude -jar springboot2-1.0.jar &
```

这里添加`-Dskywalking.trace.ignore_path`参数来标识需要过滤那些请求，支持`Ant Path`表达式：

- `/path/*`，`/path/**`，`/path/?`

- `?` 匹配任何单字符

- `*`匹配0或者任意数量的字符

- `**`匹配0或者更多的目录

- 调用接口，接口的地址为：

- - http://192.168.159.103:8082/include。

- - http://192.168.159.103:8082/exclude。

- 会发现exclude接口已经被过滤了，只有include接口能被看到。