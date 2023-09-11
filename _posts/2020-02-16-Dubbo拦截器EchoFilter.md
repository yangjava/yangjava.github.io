---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo拦截器EchoFilter

## EchoFilter
回声测试用于检测服务是否可用，回声测试按照正常请求流程执行，能够测试整个调用是否通畅，可用于监控。作用于提供者，默认激活状态

所有服务自动实现 EchoService 接口，只需将任意服务引用强制转型为 EchoService，即可使用。
Spring的配置
```xml
<dubbo:reference id="memberService" interface="com.xxx.MemberService" />
```

```java
// 远程服务引用
MemberService memberService = ctx.getBean("memberService"); 
 
EchoService echoService = (EchoService) memberService; // 强制转型为EchoService

// 回声测试可用性
String status = echoService.$echo("OK"); 
 
assert(status.equals("OK"));

```
这里解释一下 所有服务自动实现 EchoService 接口，只需将任意服务引用强制转型为 EchoService，即可使用。 说白了，就是所有的接口可以强转成 EchoService，并调用EchoService#$echo 方法来进行回声测试。

下面解释一下为什么接口可以直接强转成 EchoService，以上面代码为例：
- 对于消费者来说，消费者持有的 MemberService 并非真的是 MemberService 实例，而是一个代理类 ProxyService。当调用 ProxService#method 时，会通过网络将调用信息传递给服务提供者，由提供者找到对应的方法执行并返回，所以对消费者来说调用什么方法并没有什么区别。如：消费者调用MemberService#sayHello，发送给提供者时的信息是 类名 xxx.xxx.MemberService ，方法名 sayHello 等信息。至于是否MemberService#sayHello 方法，并不需要消费者端来判断。
- 对于提供者来说，当接收到请求后，在经历真正调用前，会先经过 Dubbo Filter 链。而在 EchoFilter 中当调用的方法满足 方法名为 $echo && 参数只有一个 的条件时就会认为是回声测试，则会在EchoFilter 中直接将结果返回，而并不会真正调用提供者的某个方法。

### EchoFilter#invoke 实现如下
```java
    @Override
    public Result invoke(Invoker<?> invoker, Invocation inv) throws RpcException {
    	// 方法名为 $echo && 参数只有一个
        if (inv.getMethodName().equals(Constants.$ECHO) && inv.getArguments() != null && inv.getArguments().length == 1) {
            return new RpcResult(inv.getArguments()[0]);
        }
        return invoker.invoke(inv);
    }
    
```







