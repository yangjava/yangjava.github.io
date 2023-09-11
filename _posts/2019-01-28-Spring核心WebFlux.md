---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring核心WebFlux
Spring 5发布了新一代响应式的Web框架，那便是Spring WebFlux。

## 基础概念
对于响应式编程，维基百科是这样定义的
```
In computing, reactive programming is an asynchronous programming paradigm concerned with data streams and the propagation of change.
```
这里的关键字是数据流（data streams）、异步（asynchronous ）和消息。此外，响应式编程还有宣言，具体的可以参考网址：
```
https://www.reactivemanifesto.org/zh-CN
```

## 什么是响应式系统
响应式系统的目标是灵敏度高，系统稳健一直有回复，松耦合和可扩展。响应式系统是一种架构，可以应用于任何地方，无论是一个小小的博客网页，还是复杂的网购系统，都可以使用响应式架构。

一般来说，响应式系统有四个显著的特点：
- 即时响应（responsive）
- 回弹性（resilience）
- 弹性（elastic）
- 消息驱动（message driven）

## Reactor模型
首先客户端会先向服务器注册其感兴趣的事件（Event），这样客户端就订阅了对应的事件，只是订阅事件并不会给服务器发送请求。当客户端发生一些已经注册的事件时，就会触发服务器的响应。当触发服务器响应时，服务器存在一个Selector线程，这个线程只是负责轮询客户端发送过来的事件，并不处理请求，当它接收到有客户端事件时，就会找到对应的请求处理器（Request Handler），然后启用另外一条线程运行处理器。因为Selector线程只是进行轮询，并不处理复杂的业务功能，所以它可以在轮询之后对请求做实时响应，速度十分快。由于事件存在很多种，所以请求处理器也存在多个，因此还需要进行区分事件的类型，所以Selector存在一个路由的问题。当请求处理器处理业务时，结果最终也会转换为数据流（data stream）发送到客户端。对于数据流的处理上还有一些细节，如后面将详细谈到的背压（Back Pressure）等。

从上述中可以看出，Reactor是一种基于事件的模型，对于服务器线程而言，它也是一种异步的，首先是Selector线程轮询到事件，然后通过路由找到处理器去运行对应的逻辑，处理器最后所返回的结果会转换为数据流。

## Spring WebFlux的概述
在Servlet 3.1之前，Web容器都是基于阻塞机制开发的，而在Servlet 3.1（包含）之后，就开始了非阻塞的规范。对于高并发网站，使用函数式的编程就显得更为直观和简易，所以它十分适合那些需要高并发和大量请求的互联网的应用。

在Java 8发布之后，引入了Lambda表达式和Functional接口等新特性，使得Java的语法更为丰富。此时Spring也有了开发响应式编程框架的想法，于是在Spring社区的支持下，Spring 5推出了Spring WebFlux这样新一代的Web响应式编程框架。

可以看到，对于响应式编程而言分为Router Functions、Spring WebFlux和HTTP/Reactive Streams共3层。
- Router Functions：是一个路由分发层，也就是它会根据请求的事件，决定采用什么类的什么方法处理客户端发送过来的事件请求。显然，在Reactor模式中，它就是Selector的作用。
- Spring-webflux：是一种控制层，类似Spring MVC框架的层级，它主要处理业务逻辑前进行的封装和控制数据流返回格式等。
- HTTP/Reactive Streams：是将结果转换为数据流的过程。对于数据流的处理还存在一些重要的细节，这是后续需要讨论的。
Spring WebFlux需要的是能够支持Servlet 3.1+的容器，如Tomcat、Jetty和Undertow等，而在Java异步编程的领域，使用得最多的却是Netty，所以在Spring Boot对Spring WebFlux的starter中默认是依赖于Netty库的。

Reactor提供的Flux和Mono，它们都是封装数据流的类。其中Flux是存放0～N个数据流序列，响应式框架会一个接一个地（请注意不是一次性）将它们发送到客户端；而对于Mono则是存放0～1个数据流序列，这就是它们之间的区别，而它们是可以进行相互转换的。这里还存在一个背压（Backpressure）的概念，只是这个概念只对Flux有意义。对于客户端，有时候响应能力距离服务端有很大的差距，如果在很短的时间内服务端将大量的数据流传输给客户端，那么客户端就可能被压垮。为了处理这个问题，一般会考虑使用响应式拉取，也就是将服务端的数据流划分为多个序列，一次仅发送一个数据流序列给客户端，当客户端处理完这个序列后，再给服务端发送消息，然后再拉取第二个序列进行处理，处理完后，再给服务端发送消息，以此类推，直至Flux中的0～N个数据流被完全处理，这样客户端就可以根据自己响应的速度来获取数据流。

## WebHandler接口和运行流程
与Spring MVC使用DispactcherServlet不同的是Spring WebFlux使用的是WebHandler。它与DispatcherServlet有异曲同工之妙，它们十分相似，所以就不需要像DispatcherServlet那样再详细地介绍它。只是WebHandler是一个接口，为此Spring WebFlux为其提供了几个实现类。

DispatcherHandler是我们关注的核心；WebHandlerDecorator则是一个装饰者，采用了装饰者模式去装饰WebHandler，而实际上并没有改变WebHandler的执行本质，所以不再详细讨论；而ResourceWebHandler则是资源的管理器，主要是处理文件和其他资源的，也属于次要的内容。因此，接下来以DispatcherHandler作为核心内容进行讨论，而实际上它与Spring MVC的DispatcherServlet是十分接近的，为了更好地研究WebFlux的流程，这里对DispatcherHandler的handle方法源码进行探讨

DispatcherHandler的handle方法
```
@Override
public Mono<Void> handle(ServerWebExchange exchange) {
    // 日志
    if (logger.isDebugEnabled()) {
        ServerHttpRequest request = exchange.getRequest();
        logger.debug("Processing " + request.getMethodValue() 
            + " request for [" + request.getURI() + "]");
    }
    return 
        Flux // Reactive框架封装数据流的类Flux
             // 循环HandlerMapping
             .fromIterable(this.handlerMappings)
             // 找到合适的处理器
             .concatMap(mapping -> mapping.getHandler(exchange))
             // 处理第一条合适的记录
             .next()
             // 如果出现找不到处理器的情况
             .switchIfEmpty(Mono.error(HANDLER_NOT_FOUND_EXCEPTION))
             // 通过反射运行处理器
             .flatMap(handler -> invokeHandler(exchange, handler))
             // 解析结果，将其转换为对应的数据流序列
             .flatMap(result -> handleResult(exchange, result));
}
```
在源码中我加入了中文注释来帮助读者了解整个运行的流程。与Spring MVC一样，都是从HandlerMapping找到对应的处理器，这也是为什么Spring WebFlux也沿用@Controller、@RequestMapping、@GetMapping、@PostMapping等注解的原因，通过这些配置路径就能够通过getHandler方法找到对应的处理器（与Spring MVC一样，处理器包含控制器的逻辑）。找到处理器后，就会通过invokeHandler方法运行处理器，在这个方法里也是找到合适的HandlerAdapter去运行处理器的，这些都可以参考Spring MVC的原理，最后就到了处理结果的handleResult方法，通过它将结果转变为对应的数据流序列。

## Backpressure
Backpressure 在国内被翻译成背压。Backpressure 是一种现象：当数据流从上游生产者向下游消费者传输的过程中，上游生产速度大于下游消费速度，导致下游的 Buffer 溢出，这种现象就叫做 Backpressure。

换句话说，上游生产数据，生产完成后通过管道将数据传到下游，下游消费数据，当下游消费速度小于上游数据生产速度时，数据在管道中积压会对上游形成一个压力，这就是 Backpressure，从这个角度来说，Backpressure 翻译成反压、回压似乎更合理一些。

## Flow API
JDK9 中推出了 Flow API，用以支持 Reactive Programming，即响应式编程。

在响应式编程中，会有一个数据发布者 Publisher 和数据订阅者 Subscriber，Subscriber 接收 Publisher 发布的数据并进行消费，在 Subscriber 和 Publisher 之间还存在一个 Processor，类似于一个过滤器，可以对数据进行中间处理。

JDK9 中提供了 Flow API 用以支持响应式编程，另外 RxJava 和 Reactor 等框架也提供了相关的实现。

### Publisher
Publisher 为数据发布者，这是一个函数式接口，里边只有一个方法，通过这个方法将数据发布出去，Publisher 的定义如下：
```
@FunctionalInterface
public static interface Publisher<T> {
    public void subscribe(Subscriber<? super T> subscriber);
}
```

### Subscriber
Subscriber 为数据订阅者，这个里边有四个方法，如下：
```
public static interface Subscriber<T> {
    public void onSubscribe(Subscription subscription);
    public void onNext(T item);
    public void onError(Throwable throwable);
    public void onComplete();
}
```
- onSubscribe：这个是订阅成功的回调方法，用于初始化 Subscription，并且表明可以开始接收订阅数据了。
- onNext：接收下一项订阅数据的回调方法。
- onError：在 Publisher 或 Subcriber 遇到不可恢复的错误时调用此方法，之后 Subscription 不会再调用 Subscriber 其他的方法。
- onComplete：当接收完所有订阅数据，并且发布者已经关闭后会回调这个方法。

### Subscription
Subscription 为发布者和订阅者之间的订阅关系，用来控制消息的消费，这个里边有两个方法：
```
public static interface Subscription {
    public void request(long n);
    public void cancel();
}
```
- request：这个方法用来向数据发布者请求 n 个数据。
- cancel：取消消息订阅，订阅者将不再接收数据。

### Processor
Processor 是一个空接口，不过它同时继承了 Publisher 和 Subscriber，所以它既能发布数据也能订阅数据，因此我们可以通过 Processor 来完成一些数据转换的功能，先接收数据进行处理，处理完成后再将数据发布出去，这个也有点类似于我们 JavaEE 中的过滤器。
```
public static interface Processor<T,R> extends Subscriber<T>, Publisher<R> {
}
```

我们通过如下一段代码体验一下消息的订阅与发布：
```
public class FlowDemo {
    public static void main(String[] args) {
        SubmissionPublisher<String> publisher = new SubmissionPublisher<>();
        Flow.Subscriber<String> subscriber = new Flow.Subscriber<String>() {
            private Flow.Subscription subscription;
            @Override
            public void onSubscribe(Flow.Subscription subscription) {
                this.subscription = subscription;
                //向数据发布者请求一个数据
                this.subscription.request(1);
            }
            @Override
            public void onNext(String item) {
                System.out.println("接收到 publisher 发来的消息了：" + item);
                //接收完成后，可以继续接收或者不接收
                //this.subscription.cancel();
                this.subscription.request(1);
            }
            @Override
            public void onError(Throwable throwable) {
                //出现异常，就会来到这个方法，此时直接取消订阅即可
                this.subscription.cancel();
            }
            @Override
            public void onComplete() {
                //发布者的所有数据都被接收，并且发布者已经关闭
                System.out.println("数据接收完毕");
            }
        };
        //配置发布者和订阅者
        publisher.subscribe(subscriber);
        for (int i = 0; i < 5; i++) {
            //发送数据
            publisher.submit("hello:" + i);
        }
        //关闭发布者
        publisher.close();
        new Scanner(System.in).next();
    }
}
```
- 首先创建一个 SubmissionPublisher 对象作为消息发布者。
- 接下来创建 Flow.Subscriber 对象作为消息订阅者，实现消息订阅者里边的四个方法，分别进行处理。
- 为 publisher 配置上 subscriber。
- 发送消息。
- 消息发送完成后关闭 publisher。
- 最后是让程序不要停止，观察消息订阅者打印情况。
































