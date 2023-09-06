---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# 基于Dubbo的灰度发布

**前言**

保证系统的高可用和稳定性是互联网应用的基本要求。需求变化、版本迭代势必会影响系统的稳定性和可用性，如何兼顾需求变化和系统稳定呢？这个影响它的因素很多，发布是其中一个。我们要尝试尽可能让发布平滑、让新功能曝光、影响人群由少到多和由内部到外部、一旦有问题马上回滚等。

**灰度发布**

什么是灰度发布？看看百度百科的解释：灰度发布（又名金丝雀发布）是指在黑与白之间，能够平滑过渡的一种发布方式。在其上可以进行A/B testing，即让一部分用户继续用产品特性A，一部分用户开始用产品特性B，如果用户对B没有什么反对意见，那么逐步扩大范围，把所有用户都迁移到B上面来。灰度发布可以保证整体系统的稳定，在初始灰度的时候就可以发现、调整问题，以保证其影响度。灰度发布开始到结束期间的这一段时间，称为灰度期。

这个解释很到位。

**Dubbo对灰度发布的支持**

实现灰度发布，就要在请求到达服务提供者前，能将请求按预设的规则去访问目标服务提供者。要做到这一点，就要在消费端或者在服务集群之前的类似网关这样的中间层做好路由。我们知道Dubbo调用是端到端的，不存在一个中间层，所以，在Dubbo的消费端就要做好请求路由。我们先看一下Dubbo的调用链路，如下图

![img](https://pic4.zhimg.com/80/v2-a7dceacc436f69355070f69e1c5af9d3_1440w.jpg)Dubbo调用链路——来自Dubbo官网

如上图所示，在Dubbo消费端，已经做了负载均衡了，但负载均衡不能满足我们对请求的路由需求。其实在负载均衡之前，invocation会先经过一个路由器链并返回一组合适的目标服务器地址列表，然后在这些备用的服务器地址中做负载均衡。源码如下：

![img](https://pic1.zhimg.com/80/v2-be17ba1ce29e18975b282a4f56f24230_1440w.jpg)AbstractClusterInvoker.invoke

所以只需要增加一个路由器，在路由器定义灰度的规则，就可以实现灰度发布了。

Dubbo原生已经按这种思路做了路由的支持了，它现在支持两种路由方式：条件路由、标签路由。

条件路由主要支持以服务或Consumer应用为粒度配置路由规则，源码对应的路由器是ConditionRouter.java。

标签路由主要是以Provider应用为粒度配置路由规则，通过将某一个或多个服务的提供者划分到同一个分组，约束流量只在指定分组中流转，从而实现流量隔离的目的，对应源码路由器是TagRouter.java。

具体的规则可以查阅文档：[http://dubbo.apache.org/zh-cn/docs/2.7/user/demos/routing-rule/](https://link.zhihu.com/?target=http%3A//dubbo.apache.org/zh-cn/docs/2.7/user/demos/routing-rule/)

Dubbo原生路由器支持的场景其实已经很丰富的了，比如条件路由器，可以设置排除预发布机器、指定Consumer应用访问指定的Provider应用，这其实就是蓝绿发布，还能设置黑白名单、按网段访问等等。但是这有一些问题，谁来管理这些规则，什么时候来管理。标签路由问题就更大一些，在配置了标签之后，还要侵入代码写入标签，至少也要在启动参数加上这个标签。使用是比较麻烦的。**现在很多公司已经在落地Devops，或者使用商业开发平台如EDAS，有的也会搭建自己的运维平台，发布流程都在追求自动化、半自动化，所以易用性很重要。**实现一个流畅的灰度发布流程，我们希望它能有一个统一的运维平台，能支持条件路由和权重路由，可以随时监控、回滚、或者继续发布。这样，我们有必要自定义一个路由器，并将其融入发布流程中。

**自定义路由器**

上篇文章我们介绍了Dubbo SPI的实现，这让Dubbo的扩展成本非常低，我们只需定义自己的路由器并把整合到应用就可以了，路由扩展文档请查阅文档：[http://dubbo.apache.org/zh-cn/docs/2.7/dev/impls/router/](https://link.zhihu.com/?target=http%3A//dubbo.apache.org/zh-cn/docs/2.7/dev/impls/router/)

自定义路由器就叫GrayRouter吧，规划一下它的功能，如下图

![img](https://pic3.zhimg.com/80/v2-baea1e01a0fa1c8a43b498ded1a6c0d2_1440w.jpg)GrayRouter功能

灰度路由器是以provider应用为粒度配置路由规则的，包含两个过滤器，条件过滤器和权重过滤器（不是Dubbo的过滤器），它们的主要任务根据配置过滤出备用provoider。配置信息放在分布式配置上，首次加载放入应用缓存，一旦有变更将自动更新。这样在发布时可以动态调整灰度参数，达到逐步扩大流量和影响人群的目的。下面是路由器的主要实现代码

![img](https://pic1.zhimg.com/80/v2-d6b318a59026823213cf671495e22dec_1440w.jpg)

![img](https://pic3.zhimg.com/80/v2-62ef19b25c5fd10db7d134f2d1eaa4b6_1440w.jpg)

![img](https://pic1.zhimg.com/80/v2-1dfdb59c59346d67b49ed6db9ea11448_1440w.jpg)

![img](https://pic2.zhimg.com/80/v2-9e279354e1ad55fa2fb5f10603a6ded9_1440w.jpg)

**发布流程可视化**

![img](https://pic4.zhimg.com/80/v2-5a000f6ee4b0e3c404ae0ca511e5736f_1440w.jpg)

![img](https://pic2.zhimg.com/80/v2-c2d21651ae49f0838e9233002ffb9a01_1440w.jpg)

![img](https://pic4.zhimg.com/80/v2-97ac73b13d3b3d2ef5f5e78502deb17b_1440w.jpg)

**监控**

我们使用了Skywalking做应用性能监控，下图是一个有两个节点的应用，灰度发布配置了权重发布，权重值10（全部是100），灰度流量分布效果如下图

![img](https://pic3.zhimg.com/80/v2-f3c67d8f30a3b6e5835733be47fecdd6_1440w.jpg)

**结束**

灰度发布在版本迭代中给系统稳定提供了有力的保证，应该将其纳入到发布流程中。

# [Springboot2.3+Dubbo2.7.3实现灰度跳转](https://www.cnblogs.com/penghq/p/13093812.html)

**1、jar包依赖**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.0.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
            <scope>provided</scope>
        </dependency>
        <!-- Aapche Dubbo相关 start  -->
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>2.7.3</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>2.13.0</version>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.zookeeper</groupId>
                    <artifactId>zookeeper</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.11</version>
            <exclusions>
                <exclusion>
                    <artifactId>slf4j-log4j12</artifactId>
                    <groupId>org.slf4j</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <!-- Aapche Dubbo相关 end -->
</dependencies>  
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

**2、自定义LoadBalance**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.pacmp.config.balance;



import lombok.extern.slf4j.Slf4j;
import org.apache.dubbo.common.URL;
import org.apache.dubbo.rpc.Invocation;
import org.apache.dubbo.rpc.Invoker;
import org.apache.dubbo.rpc.cluster.loadbalance.AbstractLoadBalance;
import org.springframework.stereotype.Component;

import java.util.*;
import java.util.concurrent.ThreadLocalRandom;

@Slf4j
@Component
public class GrayLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "gray";

    public GrayLoadBalance() {
        log.info("初始化GrayLoadBalance成功！");
    }

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        List<Invoker<T>> list = new ArrayList<>();
        for (Invoker invoker : invokers) {
            list.add(invoker);
        }
        Map<String, String> map = invocation.getAttachments();
        String ifGary = map.get("ifGary");
        String userId = map.get("userId");
        log.info("userId:"+userId+"=====ifGary:"+ifGary);
        Iterator<Invoker<T>> iterator = list.iterator();
        while (iterator.hasNext()) {
            Invoker<T> invoker = iterator.next();
            String providerStatus = invoker.getUrl().getParameter("status", "prod");
            if (Objects.equals(providerStatus, NAME)) {
                if ("1".equals(ifGary)) {
                    log.info("userId:"+userId+"=====ifGary:"+ifGary+"=====去灰度服务");
                    return invoker;
                } else {
                    log.info("userId:"+userId+"=====ifGary:"+ifGary+"=====去正常服务");
                    iterator.remove();
                }
            }
        }
        return this.randomSelect(list, url, invocation);
    }


    /**
     * 重写了一遍随机负载策略
     *
     * @param invokers
     * @param url
     * @param invocation
     * @param <T>
     * @return
     */
    private <T> Invoker<T> randomSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size();
        boolean sameWeight = true;
        int[] weights = new int[length];
        int firstWeight = this.getWeight((Invoker) invokers.get(0), invocation);
        weights[0] = firstWeight;
        int totalWeight = firstWeight;

        int offset;
        int i;
        for (offset = 1; offset < length; ++offset) {
            i = this.getWeight((Invoker) invokers.get(offset), invocation);
            weights[offset] = i;
            totalWeight += i;
            if (sameWeight && i != firstWeight) {
                sameWeight = false;
            }
        }

        if (totalWeight > 0 && !sameWeight) {
            offset = ThreadLocalRandom.current().nextInt(totalWeight);

            for (i = 0; i < length; ++i) {
                offset -= weights[i];
                if (offset < 0) {
                    return (Invoker) invokers.get(i);
                }
            }
        }
        return (Invoker) invokers.get(ThreadLocalRandom.current().nextInt(length));
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

**3、在resources加配置文件，路径如下图（路径必须一致）**

**文件名：org.apache.dubbo.rpc.cluster.LoadBalance**







**![img](https://img2020.cnblogs.com/blog/1179838/202006/1179838-20200611154418753-1649818507.png)**



```
添加GrayLoadBalance类的路径
```

![img](https://img2020.cnblogs.com/blog/1179838/202006/1179838-20200611154625135-738714939.png)

**4、application.yml配置**

**消费者配置：**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
# dubbo
dubbo:
  application:
    name: dubbo_consumer
  registry:
    address: zookeeper://127.0.0.1:2181
  scan:
    base-packages: com.pacmp.controller
  consumer:
    version: 2.0.0
  provider:
    loadbalance: gray
  protocol:
    port: 10000
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

**生产者配置：**

```
loadbalance: gray   表示加载自定义loadbalance：com.pacmp.config.balance.GrayLoadBalance。
parameters:
      status: gray  表示这个生产者（服务），是否为灰度。如果不是灰度可以不用配置。
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
## Dubbo配置
dubbo:
  application:
    name: dubbo_provider
  registry:
    address: zookeeper://127.0.0.1:2181
  protocol:
    name: dubbo
    port: -1
  scan:
    base-packages: com.pacmp
  provider:
    loadbalance: gray
    version: 2.0.0
    parameters:
      status: gray
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

**5、调用示例**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.pacmp.controller;


import com.pacmp.service.DemoService;
import lombok.extern.slf4j.Slf4j;


import org.apache.dubbo.config.annotation.Reference;
import org.apache.dubbo.rpc.RpcContext;
import org.springframework.web.bind.annotation.*;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;
import java.util.*;

/**
 * @Author xxx
 * @Date 2020/05/26 9:52
 * @Version 1.0
 * @Description 接入层
 */
@Slf4j
@RestController
@RequestMapping("/api")
public class ApiController {

    @Reference(check = false)
    private DemoService demoService;

    @GetMapping("/testUser")
    public String testUser(int userId, String version) {
        //ifGary=1代表灰度用户，0代表普通用户
        int ifGary = 0;
        if(userId<10){
            ifGary = 1;
        }
        RpcContext.getContext().setAttachment("ifGary", String.valueOf(ifGary));
        RpcContext.getContext().setAttachment("userId", String.valueOf(userId));
        return demoService.testUser(userId, version);
    }

}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

**6、测试**

启2个生产者服务，1个消费者服务。可根据userId的不同来调用生产服务/灰度服务。

3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
代码1)处调用pluginFinder的getBootstrapClassMatchDefine()方法，在初始化PluginFinder对象的时候，会对加载的所有插件做分类，要对JDK核心类库生效的插件都放入到一个List（bootstrapClassMatchDefine）中，这里调用getBootstrapClassMatchDefine()就是拿到所有要对JDK核心类库生效的插件

遍历所有要对JDK核心类库生效的插件，分别判断是否定义实例方法拦截点、是否定义构造器拦截点、是否定义静态方法拦截点，以实例方法拦截点为例，代码2)处再根据是否修改原方法入参走不同分支处理逻辑

以不重写原方法入参为例，代码3)处调用generateDelegator()方法生成一个代理器，这里会传入一个模板类名，对于实例方法拦截点且不修改原方法入参，模板类名为org.apache.skywalking.apm.agent.core.plugin.bootstrap.template.InstanceMethodInterTemplate

/**
* --------CLASS TEMPLATE---------
* <p>Author, Wu Sheng </p>
* <p>Comment, don't change this unless you are 100% sure the agent core mechanism for bootstrap class
* instrumentation.</p>
* <p>Date, 24th July 2019</p>
* -------------------------------
* <p>
* This class wouldn't be loaded in real env. This is a class template for dynamic class generation.
* 在真实环境下这个类是不会被加载的,这是一个类模板用于动态类生成
  */
  public class InstanceMethodInterTemplate {
  /**
    * This field is never set in the template, but has value in the runtime.
      */
      private static String TARGET_INTERCEPTOR;

  private static InstanceMethodsAroundInterceptor INTERCEPTOR;
  private static IBootstrapLog LOGGER;

  /**
    * Intercept the target instance method.
        *
    * @param obj          target class instance.
    * @param allArguments all method arguments
    * @param method       method description.
    * @param zuper        the origin call ref.
    * @return the return value of target instance method.
    * @throws Exception only throw exception because of zuper.call() or unexpected exception in sky-walking ( This is a
    *                   bug, if anything triggers this condition ).
      */
      @RuntimeType
      public static Object intercept(@This Object obj, @AllArguments Object[] allArguments, @SuperCall Callable<?> zuper,
      @Origin Method method) throws Throwable {
      EnhancedInstance targetObject = (EnhancedInstance) obj;

      prepare();

      MethodInterceptResult result = new MethodInterceptResult();
      try {
      if (INTERCEPTOR != null) {
      INTERCEPTOR.beforeMethod(targetObject, method, allArguments, method.getParameterTypes(), result);
      }
      } catch (Throwable t) {
      if (LOGGER != null) {
      LOGGER.error(t, "class[{}] before method[{}] intercept failure", obj.getClass(), method.getName());
      }
      }

      Object ret = null;
      try {
      if (!result.isContinue()) {
      ret = result._ret();
      } else {
      ret = zuper.call();
      }
      } catch (Throwable t) {
      try {
      if (INTERCEPTOR != null) {
      INTERCEPTOR.handleMethodException(targetObject, method, allArguments, method.getParameterTypes(), t);
      }
      } catch (Throwable t2) {
      if (LOGGER != null) {
      LOGGER.error(t2, "class[{}] handle method[{}] exception failure", obj.getClass(), method.getName());
      }
      }
      throw t;
      } finally {
      try {
      if (INTERCEPTOR != null) {
      ret = INTERCEPTOR.afterMethod(targetObject, method, allArguments, method.getParameterTypes(), ret);
      }
      } catch (Throwable t) {
      if (LOGGER != null) {
      LOGGER.error(t, "class[{}] after method[{}] intercept failure", obj.getClass(), method.getName());
      }
      }
      }

      return ret;
      }

  /**
    * Prepare the context. Link to the agent core in AppClassLoader.
        *
    * 1.打通BootstrapClassLoader和AgentClassLoader
    * 拿到ILog生成日志对象
    * 拿到插件自定义的拦截器实例
    * 2.代替非JDK核心类库插件运行逻辑里的
    * InterceptorInstanceLoader.load(instanceMethodsAroundInterceptorClassName, classLoader)
      */
      private static void prepare() {
      if (INTERCEPTOR == null) {
      ClassLoader loader = BootstrapInterRuntimeAssist.getAgentClassLoader();

           if (loader != null) {
               IBootstrapLog logger = BootstrapInterRuntimeAssist.getLogger(loader, TARGET_INTERCEPTOR);
               if (logger != null) {
                   LOGGER = logger;
           
                   INTERCEPTOR = BootstrapInterRuntimeAssist.createInterceptor(loader, TARGET_INTERCEPTOR, LOGGER);
               }
           } else {
               LOGGER.error("Runtime ClassLoader not found when create {}." + TARGET_INTERCEPTOR);
           }
      }
      }
      }
      1
      2
      3
      4
      5
      6
      7
      8
      9
      10
      11
      12
      13
      14
      15
      16
      17
      18
      19
      20
      21
      22
      23
      24
      25
      26
      27
      28
      29
      30
      31
      32
      33
      34
      35
      36
      37
      38
      39
      40
      41
      42
      43
      44
      45
      46
      47
      48
      49
      50
      51
      52
      53
      54
      55
      56
      57
      58
      59
      60
      61
      62
      63
      64
      65
      66
      67
      68
      69
      70
      71
      72
      73
      74
      75
      76
      77
      78
      79
      80
      81
      82
      83
      84
      85
      86
      87
      88
      89
      90
      91
      92
      93
      94
      95
      96
      97
      98
      99
      100
      101
      102
      103
      104
      105
      106
      107
      108
      模板类的intercept()方法和实例方法插桩的InstMethodsInter的intercept()方法逻辑基本相同

generateDelegator()方法源码如下：

public class BootstrapInstrumentBoost {

    /**
     * Generate the delegator class based on given template class. This is preparation stage level code generation.
     * 根据给定的模板类生成代理器类,这是准备阶段级别的代码生成
     * <p>
     * One key step to avoid class confliction between AppClassLoader and BootstrapClassLoader
     * 避免AppClassLoader和BootstrapClassLoader之间的类冲突的一个关键步骤
     *
     * @param classesTypeMap    hosts injected binary of generated class 所有要注入到BootStrapClassLoader中的类,key:全类名 value:字节码
     * @param typePool          to generate new class 加载BootstrapInstrumentBoost的ClassLoader的类型池
     * @param templateClassName represents the class as template in this generation process. The templates are
     *                          pre-defined in SkyWalking agent core. 模板类名
     * @param methodsInterceptor 插件拦截器全类名
     */
    private static void generateDelegator(Map<String, byte[]> classesTypeMap, TypePool typePool,
        String templateClassName, String methodsInterceptor) {
        // methodsInterceptor + "_internal"
        String internalInterceptorName = internalDelegate(methodsInterceptor);
        try {
            // ClassLoaderA 已经加载了100个类,但是在这个ClassLoader的classpath下有200个类,那么这里
            // typePool.describe可以拿到当前ClassLoader的classpath下还没有加载的类的定义(描述)
            TypeDescription templateTypeDescription = typePool.describe(templateClassName).resolve();
    
            DynamicType.Unloaded interceptorType = new ByteBuddy().redefine(templateTypeDescription, ClassFileLocator.ForClassLoader
                    .of(BootstrapInstrumentBoost.class.getClassLoader()))
                                                                  // 改名为methodsInterceptor + "_internal"
                                                                  .name(internalInterceptorName)
                                                                  // TARGET_INTERCEPTOR赋值为插件拦截器全类名
                                                                  .field(named("TARGET_INTERCEPTOR"))
                                                                  .value(methodsInterceptor)
                                                                  // 组装好字节码还未加载
                                                                  .make();
    
            classesTypeMap.put(internalInterceptorName, interceptorType.getBytes());
    
            InstrumentDebuggingClass.INSTANCE.log(interceptorType);
        } catch (Exception e) {
            throw new PluginException("Generate Dynamic plugin failure", e);
        }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
generateDelegator()方法就是将模板类交给ByteBuddy去编译成字节码，改了新的类名，并将TARGET_INTERCEPTOR属性赋值为插件拦截器全类名，然后就放入到classesTypeMap中（所有要注入到BootStrapClassLoader中的类）

再回到BootstrapInstrumentBoost的inject()方法：

public class BootstrapInstrumentBoost {

    public static AgentBuilder inject(PluginFinder pluginFinder, Instrumentation instrumentation,
        AgentBuilder agentBuilder, JDK9ModuleExporter.EdgeClasses edgeClasses) throws PluginException {
        // 所有要注入到Bootstrap ClassLoader里的类
        Map<String, byte[]> classesTypeMap = new HashMap<>();
    
        /**
         * 针对于目标类是JDK核心类库的插件,根据插件的拦截点的不同(实例方法、静态方法、构造方法)
         * 使用不同的模板(xxxTemplate)来定义新的拦截器的核心处理逻辑,并且将插件本身定义的拦截器的全类名
         * 赋值给模板的TARGET_INTERCEPTOR字段
         * 最终,这些新的拦截器的核心处理逻辑都会被放入到BootstrapClassLoader中
         */
      	// 1)
        if (!prepareJREInstrumentation(pluginFinder, classesTypeMap)) {
            return agentBuilder;
        }
    
        if (!prepareJREInstrumentationV2(pluginFinder, classesTypeMap)) {
            return agentBuilder;
        }
    
        for (String highPriorityClass : HIGH_PRIORITY_CLASSES) {
            loadHighPriorityClass(classesTypeMap, highPriorityClass);
        }
        for (String highPriorityClass : ByteBuddyCoreClasses.CLASSES) {
            loadHighPriorityClass(classesTypeMap, highPriorityClass);
        }
    
        /**
         * Prepare to open edge of necessary classes.
         */
        for (String generatedClass : classesTypeMap.keySet()) {
            edgeClasses.add(generatedClass);
        }
    
        /**
         * 将生成的类注入到Bootstrap ClassLoader
         * Inject the classes into bootstrap class loader by using Unsafe Strategy.
         * ByteBuddy adapts the sun.misc.Unsafe and jdk.internal.misc.Unsafe automatically.
         */
      	// 2)
        ClassInjector.UsingUnsafe.Factory factory = ClassInjector.UsingUnsafe.Factory.resolve(instrumentation);
        factory.make(null, null).injectRaw(classesTypeMap);
        agentBuilder = agentBuilder.with(new AgentBuilder.InjectionStrategy.UsingUnsafe.OfFactory(factory));
    
        return agentBuilder;
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
代码1)处调用prepareJREInstrumentation()方法，核心逻辑：针对于目标类是JDK核心类库的插件，根据插件的拦截点的不同（实例方法、静态方法、构造方法），使用不同的模板（xxxTemplate）来定义新的拦截器的核心处理逻辑，并且将插件本身定义的拦截器的全类名赋值给模板的TARGET_INTERCEPTOR字段

代码2)处将生成的类注入到Bootstrap ClassLoader

2）、JDK类库方法插桩
无论是静态方法插桩还是构造器和实例方法插桩都会判断是否是JDK类库中的类，如果是会调用BootstrapInstrumentBoost的forInternalDelegateClass()方法，以实例方法插桩为例：

public abstract class ClassEnhancePluginDefine extends AbstractClassEnhancePluginDefine {

    protected DynamicType.Builder<?> enhanceInstance(TypeDescription typeDescription,
        // ...
        /**
         * 3. enhance instance methods
         * 增强实例方法
         */
        if (existedMethodsInterceptPoints) {
            for (InstanceMethodsInterceptPoint instanceMethodsInterceptPoint : instanceMethodsInterceptPoints) {
                String interceptor = instanceMethodsInterceptPoint.getMethodsInterceptor();
                if (StringUtil.isEmpty(interceptor)) {
                    throw new EnhanceException("no InstanceMethodsAroundInterceptor define to enhance class " + enhanceOriginClassName);
                }
                ElementMatcher.Junction<MethodDescription> junction = not(isStatic()).and(instanceMethodsInterceptPoint.getMethodsMatcher());
                // 如果拦截点为DeclaredInstanceMethodsInterceptPoint
                if (instanceMethodsInterceptPoint instanceof DeclaredInstanceMethodsInterceptPoint) {
                    // 拿到的方法必须是当前类上的 通过注解匹配可能匹配到很多方法不是当前类上的
                    junction = junction.and(ElementMatchers.<MethodDescription>isDeclaredBy(typeDescription));
                }
                if (instanceMethodsInterceptPoint.isOverrideArgs()) {
                    if (isBootstrapInstrumentation()) {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .withBinders(Morph.Binder.install(OverrideCallable.class))
                                                                                    .to(BootstrapInstrumentBoost.forInternalDelegateClass(interceptor)));
                    } else {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .withBinders(Morph.Binder.install(OverrideCallable.class))
                                                                                    .to(new InstMethodsInterWithOverrideArgs(interceptor, classLoader)));
                    }
                } else {
                    if (isBootstrapInstrumentation()) {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .to(BootstrapInstrumentBoost.forInternalDelegateClass(interceptor)));
                    } else {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .to(new InstMethodsInter(interceptor, classLoader)));
                    }
                }
            }
        }
    
        return newClassBuilder;
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
BootstrapInstrumentBoost的forInternalDelegateClass()方法源码如下：

public class BootstrapInstrumentBoost {

    public static Class forInternalDelegateClass(String methodsInterceptor) {
        try {
            // methodsInterceptor + "_internal"
            return Class.forName(internalDelegate(methodsInterceptor));
        } catch (ClassNotFoundException e) {
            throw new PluginException(e.getMessage(), e);
        }
    }
1
2
3
4
5
6
7
8
9
10
通过Class.forName()加载插件拦截器全类名+_internal的类，这个类在Agent启动流程根据模板类生成并注入到Bootstrap ClassLoader中，所以这里是能加载到

/**
* --------CLASS TEMPLATE---------
* <p>Author, Wu Sheng </p>
* <p>Comment, don't change this unless you are 100% sure the agent core mechanism for bootstrap class
* instrumentation.</p>
* <p>Date, 24th July 2019</p>
* -------------------------------
* <p>
* This class wouldn't be loaded in real env. This is a class template for dynamic class generation.
* 在真实环境下这个类是不会被加载的,这是一个类模板用于动态类生成
  */
  public class InstanceMethodInterTemplate {
  /**
    * This field is never set in the template, but has value in the runtime.
      */
      private static String TARGET_INTERCEPTOR;

  private static InstanceMethodsAroundInterceptor INTERCEPTOR;
  private static IBootstrapLog LOGGER;

  /**
    * Intercept the target instance method.
        *
    * @param obj          target class instance.
    * @param allArguments all method arguments
    * @param method       method description.
    * @param zuper        the origin call ref.
    * @return the return value of target instance method.
    * @throws Exception only throw exception because of zuper.call() or unexpected exception in sky-walking ( This is a
    *                   bug, if anything triggers this condition ).
      */
      @RuntimeType
      public static Object intercept(@This Object obj, @AllArguments Object[] allArguments, @SuperCall Callable<?> zuper,
      @Origin Method method) throws Throwable {
      EnhancedInstance targetObject = (EnhancedInstance) obj;

      prepare();

      MethodInterceptResult result = new MethodInterceptResult();
      try {
      if (INTERCEPTOR != null) {
      INTERCEPTOR.beforeMethod(targetObject, method, allArguments, method.getParameterTypes(), result);
      }
      } catch (Throwable t) {
      if (LOGGER != null) {
      LOGGER.error(t, "class[{}] before method[{}] intercept failure", obj.getClass(), method.getName());
      }
      }

      Object ret = null;
      try {
      if (!result.isContinue()) {
      ret = result._ret();
      } else {
      ret = zuper.call();
      }
      } catch (Throwable t) {
      try {
      if (INTERCEPTOR != null) {
      INTERCEPTOR.handleMethodException(targetObject, method, allArguments, method.getParameterTypes(), t);
      }
      } catch (Throwable t2) {
      if (LOGGER != null) {
      LOGGER.error(t2, "class[{}] handle method[{}] exception failure", obj.getClass(), method.getName());
      }
      }
      throw t;
      } finally {
      try {
      if (INTERCEPTOR != null) {
      ret = INTERCEPTOR.afterMethod(targetObject, method, allArguments, method.getParameterTypes(), ret);
      }
      } catch (Throwable t) {
      if (LOGGER != null) {
      LOGGER.error(t, "class[{}] after method[{}] intercept failure", obj.getClass(), method.getName());
      }
      }
      }

      return ret;
      }

  /**
    * Prepare the context. Link to the agent core in AppClassLoader.
        *
    * 1.打通BootstrapClassLoader和AgentClassLoader
    * 拿到ILog生成日志对象
    * 拿到插件自定义的拦截器实例
    * 2.代替非JDK核心类库插件运行逻辑里的
    * InterceptorInstanceLoader.load(instanceMethodsAroundInterceptorClassName, classLoader)
      */
      private static void prepare() {
      if (INTERCEPTOR == null) {
      ClassLoader loader = BootstrapInterRuntimeAssist.getAgentClassLoader();

           if (loader != null) {
               IBootstrapLog logger = BootstrapInterRuntimeAssist.getLogger(loader, TARGET_INTERCEPTOR);
               if (logger != null) {
                   LOGGER = logger;
           
                   INTERCEPTOR = BootstrapInterRuntimeAssist.createInterceptor(loader, TARGET_INTERCEPTOR, LOGGER);
               }
           } else {
               LOGGER.error("Runtime ClassLoader not found when create {}." + TARGET_INTERCEPTOR);
           }
      }
      }
      }
      1
      2
      3
      4
      5
      6
      7
      8
      9
      10
      11
      12
      13
      14
      15
      16
      17
      18
      19
      20
      21
      22
      23
      24
      25
      26
      27
      28
      29
      30
      31
      32
      33
      34
      35
      36
      37
      38
      39
      40
      41
      42
      43
      44
      45
      46
      47
      48
      49
      50
      51
      52
      53
      54
      55
      56
      57
      58
      59
      60
      61
      62
      63
      64
      65
      66
      67
      68
      69
      70
      71
      72
      73
      74
      75
      76
      77
      78
      79
      80
      81
      82
      83
      84
      85
      86
      87
      88
      89
      90
      91
      92
      93
      94
      95
      96
      97
      98
      99
      100
      101
      102
      103
      104
      105
      106
      107
      108
      JDK类库中的类的实例方法插桩且不修改原方法入参会交给通过InstanceMethodInterTemplate生成的类去处理，实际也就是模板类InstanceMethodInterTemplate的TARGET_INTERCEPTOR赋值为插件拦截器全类名，和实例方法插桩的InstMethodsInter的intercept()方法相比这里多调用了一个prepare()方法

prepare()方法处理逻辑如下：

拿到AgentClassLoader
通过AgentClassLoader加载，拿到ILog生成日志对象
通过AgentClassLoader加载，拿到插件自定义的拦截器实例
InstanceMethodInterTemplate生成的类是由BootstrapClassLoader去加载的，而日志对象和插件自定义的拦截器都是通过AgentClassLoader去加载的，prepare()方法本质就是为了打通BootstrapClassLoader和AgentClassLoader


假设BootstrapClassLoader加载的由InstanceMethodInterTemplate生成的类是org.apache.skywalking.xxx.DubboInterceptor_internal，AgentClassLoader加载了日志用到的ILog和插件拦截器DubboInterceptor，AgentClassLoader的顶层父类加载器为BootstrapClassLoader，根据双亲委派模型，从下往上加载是可以拿到的，但是从上往下加载是拿不到的（BootstrapClassLoader中不能到ILog和DubboInterceptor），所以需要通过prepare()方法打通BootstrapClassLoader和AgentClassLoader
