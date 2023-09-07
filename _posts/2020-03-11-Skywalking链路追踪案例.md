---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking链路追踪案例

## 链路追踪案例
SkyWalking UI中展示的一条链路，这条链路的流程如下：

- 入口是demo1的/api/demo1接口，demo1先调用MySQL，然后通过HttpClient调用demo2的/api/demo2接口
- 应用demo2的/api/demo2接口直接返回响应
- demo1收到demo2的/api/demo2接口的响应后返回，整条链路结束

下面来分析下SkyWalking Agent对这条链路的追踪过程：

## demo1入口接收请求
请求到达demo1后，走到Tomcat，Tomcat插件（TomcatInvokeInterceptor）创建EntrySpan（ContextManager.createEntrySpan()）。因为ThreadLocal中的TracingContext为空，会先创建TracingContext然后放到ThreadLocal中，然后使用TracingContext创建EntrySpan（TracingContext.createEntrySpan()）。

TracingContext中activeSpanStack为空，创建了第一个EntrySpan（spanId=0，parentSpanId=-1）并入栈到activeSpanStack中

Tomcat插件创建的EntrySpan入栈后：
```
       activeSpanStack
   _________________________
  |   spanId = 0            |
  |   parentSpanId = -1     |
  |   EntrySpan # Tomcat    |
  |_________________________|
```
请求走到SpringMVC后，SpringMVC插件（AbstractMethodInterceptor）使用ThreadLocal中的TracingContext创建EntrySpan。

这时TracingContext中activeSpanStack栈顶的Span是EntrySpan，所以直接复用，并覆盖了Tomcat插件记录的信息

SpringMVC插件复用Tomcat插件创建的EntrySpan：
```
       activeSpanStack
   _________________________
  |   spanId = 0            |
  |   parentSpanId = -1     |
  |   EntrySpan # SpringMVC |
  |_________________________|
```

## demo1调用MySQL
demo1调用MySQL，MySQL插件（PreparedStatementExecuteMethodsInterceptor）使用ThreadLocal中的TracingContext创建ExitSpan。

拿到TracingContext中activeSpanStack栈顶的Span（EntrySpan#SpringMVC）作为parentSpan，创建ExitSpan（spanId=1，parentSpanId=0）并入栈到activeSpanStack中
```
      activeSpanStack
   _________________________
  |   spanId = 1            |
  |   parentSpanId = 0      |
  |   ExitSpan # MySQL      |
  |_________________________|
   _________________________
  |   spanId = 0            |
  |   parentSpanId = -1     |
  |   EntrySpan # SpringMVC |
  |_________________________|
```

访问MySQL操作结束后，MySQL插件的后置处理使用ThreadLocal中的TracingContext stopSpan（TracingContext.stopSpan()）。TracingContext中activeSpanStack栈顶的Span出栈，放到TracingContext中TraceSegment的spans集合中（执行完的Span会放到TraceSegment的spans集合中，等待后续发送到OAP）

MySQL插件创建的ExitSpan出栈后：
```
   _________________________
  |   spanId = 0            |
  |   parentSpanId = -1     |
  |   EntrySpan # SpringMVC |
  |_________________________|
```


## demo1调用demo2接口
demo1通过HttpClient调用demo2接口，HttpClient插件（HttpClientExecuteInterceptor）使用ThreadLocal中的TracingContext创建ExitSpan。拿到TracingContext中activeSpanStack栈顶的Span（EntrySpan#SpringMVC）作为parentSpan，创建ExitSpan（spanId=2，parentSpanId=0）并入栈到activeSpanStack中

HttpClient插件创建的ExitSpan入栈后：

```
      activeSpanStack
   _________________________
  |   spanId = 2            |
  |   parentSpanId = 0      |
  |   ExitSpan # HttpClient |
  |_________________________|
   _________________________
  |   spanId = 0            |
  |   parentSpanId = -1     |
  |   EntrySpan # SpringMVC |
  |_________________________|
```

创建完ExitSpan后，调用TracingContext.inject()给ContextCarrier赋值，包括TraceId、TraceSegmentId、SpanId（当前ExitSpan的Id）、ParentService、ParentServiceInstance等信息。然后会把ContextCarrier中的数据放到Http请求头中，通过这种方式让链路信息传递下去

demo2接收到demo1的请求后，创建EntrySpan的流程和demo1入口接收请求一致，这里会多一步，就是从Http请求头中拿到demo1传递的链路信息赋值给ContextCarrier，调用TracingContext.extract()绑定当前TraceSegment的traceSegmentRef、traceId以及EntrySpan的ref

demo2的响应返回后，demo1中插件后置处理依次调用TracingContext.stopSpan()，TracingContext中activeSpanStack中的Span依次出栈，最后activeSpanStack栈为空时，TracingContext结束

## 如何正确地编写插件防止内存泄漏
在使用 SKywalking 的过程中，如果是同步调用的话，就比较简单，例如，在 before 方法中创建一个 span（就是向栈中推入一个 Span），在 after 方法中，执行 stop span（就是从栈中弹出一个 Span）。

当编写异步插件时，需要考虑的情况就比较复杂。有几个点需要注意：

- 当我们执行 capture 和 continued 时，栈顶一定要有 Span。这样才能将这两个 Span 进行链接。

- 当我们执行 prepareForAsync 异步时，一定要在其他线程执行 asyncFinish，否则这个 Segment 就会断开，因为如果不执行 asyncFinish，这个 Segment 就不会 finish，也就不会发送到后端 OAP。另外，对一个 Span 执行完 prepareForAsync 后，一定不要忘记执行这个 span 的 stop 方法。

- 一定要正确的调用 ContextManager.stopSpan()，否则，一定会出现内存泄漏。假设，Tomcat Span 是入口，在 Tomcat 插件的 after 方法里，执行了 stopSpan，但是栈却没有清空，那么 ThreadLocal里的对象就不会清除，当下次在这个线程里调用 continued时，continued 会将其他线程的对象继续添加到这个线程里的 Segment 列表里。导致内存无限增大（新版本限制了链表的大小，但没有从根本解决问题）。



















