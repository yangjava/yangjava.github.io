---
layout: post
categories: [Netty]
description: none
keywords: Netty
---
# Netty源码ChannelPipeline
Netty的ChannelPipeline和ChannelHandler机制类似于Servlet和Filter过滤器，这类拦截器实际上是职责链模式的一种变形，主要是为了方便事件的拦截和用户业务逻辑的定制。

## ChannelPipeline功能说明
Netty的Channel过滤器实现原理与Servlet Filter机制一致，它将Channel的数据管道抽象为ChannelPipeline，消息在ChannelPipeline中流动和传递。ChannelPipeline持有I/O事件拦截器ChannelHandler的链表，由ChannelHandler对I/O事件进行拦截和处理，可以方便地通过新增和删除ChannelHandler来实现不同的业务逻辑定制，不需要对已有的ChannelHandler进行修改，能够实现对修改封闭和对扩展的支持。

下面我们对ChannelPipeline和ChannelHandler，以及与之相关的ChannelHandlerContext进行详细介绍和源码分析。

ChannelPipeline是ChannelHandler的容器，它负责ChannelHandler的管理和事件拦截与调度。

### ChannelPipeline的事件处理
一个消息被ChannelPipeline的ChannelHandler链拦截和处理的全过程，消息的读取和发送处理全流程描述如下。

（1）底层的SocketChannel read（）方法读取ByteBuf，触发ChannelRead事件，由I/O线程NioEventLoop调用ChannelPipeline的fireChannelRead（Object msg）方法，将消息（ByteBuf）传输到ChannelPipeline中。

（2）消息依次被HeadHandler、ChannelHandler1、ChannelHandler2……TailHandler拦截和处理，在这个过程中，任何ChannelHandler都可以中断当前的流程，结束消息的传递。

（3）调用ChannelHandlerContext的write方法发送消息，消息从TailHandler开始，途经ChannelHandlerN……ChannelHandler1、HeadHandler，最终被添加到消息发送缓冲区中等待刷新和发送，在此过程中也可以中断消息的传递，例如当编码失败时，就需要中断流程，构造异常的Future返回。

Netty中的事件分为inbound事件和outbound事件。inbound事件通常由I/O线程触发，例如TCP链路建立事件、链路关闭事件、读事件、异常通知事件等，它对应图17-1的左半部分。

触发inbound事件的方法如下。

（1）ChannelHandlerContext.fireChannelRegistered（）：Channel注册事件；

（2）ChannelHandlerContext.fireChannelActive（）：TCP链路建立成功，Channel激活事件；

（3）ChannelHandlerContext.fireChannelRead（Object）：读事件；

（4）ChannelHandlerContext.fireChannelReadComplete（）：读操作完成通知事件；

（5）ChannelHandlerContext.fireExceptionCaught（Throwable）：异常通知事件；

（6）ChannelHandlerContext.fireUserEventTriggered（Object）：用户自定义事件；

（7）ChannelHandlerContext.fireChannelWritabilityChanged（）：Channel的可写状态变化通知事件；

（8）ChannelHandlerContext.fireChannelInactive（）：TCP连接关闭，链路不可用通知事件。

Outbound事件通常是由用户主动发起的网络I/O操作，例如用户发起的连接操作、绑定操作、消息发送等操作，它对应图17-1的右半部分。

触发outbound事件的方法如下：

（1）ChannelHandlerContext.bind（SocketAddress，ChannelPromise）：绑定本地地址事件；

（2）ChannelHandlerContext.connect（SocketAddress，SocketAddress，ChannelPromise）：连接服务端事件；

（3）ChannelHandlerContext.write（Object，ChannelPromise）：发送事件；

（4）ChannelHandlerContext.flush（）：刷新事件；

（5）ChannelHandlerContext.read（）：读事件；

（6）ChannelHandlerContext.disconnect（ChannelPromise）：断开连接事件；

（7）ChannelHandlerContext.close（ChannelPromise）：关闭当前Channel事件。