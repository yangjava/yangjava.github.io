---
layout: post
categories: [Netty]
description: none
keywords: Netty
---
# Netty源码Channel

## Channel
JDK的NIO类库的一个重要组成部分，就是java.nio.SocketChannel和java.nio.ServerSocketChannel，它们用于非阻塞的I/O操作。 类似于NIO的Channel，Netty提供了自己的Channel和其子类实现，用于异步I/O操作和其他相关的操作。

io.netty.channel.Channel是Netty网络操作抽象类，它聚合了一组功能，包括但不限于网路的读、写，客户端发起连接，主动关闭连接，链路关闭，获取通信双方的网络地址等。它也包含了Netty框架相关的一些功能，包括获取该Chanel的EventLoop，获取缓冲分配器ByteBufAllocator和pipeline等。

## Channel的工作原理
Channel是Netty抽象出来的网络I/O读写相关的接口，为什么不使用JDK NIO 原生的Channel而要另起炉灶呢，主要原因如下。
- JDK的SocketChannel和ServerSocketChannel没有统一的Channel接口供业务开发者使用，对于用户而言，没有统一的操作视图，使用起来并不方便。
- JDK的SocketChannel和ServerSocketChannel的主要职责就是网络I/O操作，由于它们是SPI类接口，由具体的虚拟机厂家来提供，所以通过继承SPI功能类来扩展其功能的难度很大；直接实现ServerSocketChannel和SocketChannel抽象类，其工作量和重新开发一个新的Channel功能类是差不多的。
- Netty的Channel需要能够跟Netty的整体架构融合在一起，例如I/O模型、基于ChannelPipeline的定制模型，以及基于元数据描述配置化的TCP参数等，这些JDK的SocketChannel和ServerSocketChannel都没有提供，需要重新封装。
- 自定义的Channel，功能实现更加灵活。

基于上述4个原因，Netty重新设计了Channel接口，并且给予了很多不同的实现。它的设计原理比较简单，但是功能却比较繁杂，主要的设计理念如下。
- 在Channel接口层，采用Facade模式进行统一封装，将网络I/O操作、网络I/O相关联的其他操作封装起来，统一对外提供。
- Channel接口的定义尽量大而全，为SocketChannel和ServerSocketChannel提供统一的视图，由不同子类实现不同的功能，公共功能在抽象父类中实现，最大程度地实现功能和接口的重用。
- 具体实现采用聚合而非包含的方式，将相关的功能类聚合在Channel中，由Channel统一负责分配和调度，功能实现更加灵活。

## Channel的功能介绍
Channel网络I/O相关的方法定义
- Channel read（）：从当前的Channel中读取数据到第一个inbound缓冲区中，如果数据被成功读取，触发ChannelHandler.channelRead（ChannelHandlerContext，Object）事件。读取操作API调用完成之后，紧接着会触发ChannelHandler.channelReadComplete（Channel HandlerContext）事件，这样业务的ChannelHandler可以决定是否需要继续读取数据。如果已经有读操作请求被挂起，则后续的读操作会被忽略。
- ChannelFuture write（Object msg）：请求将当前的msg通过ChannelPipeline写入到目标Channel中。注意，write操作只是将消息存入到消息发送环形数组中，并没有真正被发送，只有调用flush操作才会被写入到Channel中，发送给对方。
- ChannelFuture write（Object msg，ChannelPromise promise）：功能与write（Object msg）相同，但是携带了ChannelPromise参数负责设置写入操作的结果。
- ChannelFuture writeAndFlush（Object msg，ChannelPromise promise）：与方法（3）功能类似，不同之处在于它会将消息写入Channel中发送，等价于单独调用write和flush操作的组合。
- ChannelFuture writeAndFlush（Object msg）：功能等同于方法（4），但是没有携带writeAndFlush（Object msg）参数。
- Channel flush（）：将之前写入到发送环形数组中的消息全部写入到目标Chanel中，发送给通信对方。
- ChannelFuture close（ChannelPromise promise）：主动关闭当前连接，通过Channel Promise设置操作结果并进行结果通知，无论操作是否成功，都可以通过ChannelPromise获取操作结果。该操作会级联触发ChannelPipeline中所有ChannelHandler的ChannelHandler.close（ChannelHandlerContext，ChannelPromise）事件。
- ChannelFuture disconnect（ChannelPromise promise）：请求断开与远程通信对端的连接并使用ChannelPromise来获取操作结果的通知消息。该方法会级联触发Channel Handler.disconnect（ChannelHandlerContext，ChannelPromise）事件。
- ChannelFuture connect（SocketAddress remoteAddress）：客户端使用指定的服务端地址remoteAddress发起连接请求，如果连接因为应答超时而失败，ChannelFuture中的操作结果就是ConnectTimeoutException异常；如果连接被拒绝，操作结果为ConnectException。该方法会级联触发ChannelHandler.connect（ChannelHandlerContext，SocketAddress，SocketAddress，ChannelPromise）事件。
- ChannelFuture connect（SocketAddress remoteAddress，SocketAddress localAddress）：与方法（9）功能类似，唯一不同的就是先绑定指定的本地地址localAddress，然后再连接服务端。
- ChannelFuture connect（SocketAddress remoteAddress，ChannelPromise promise）：与方法（9）功能类似，唯一不同的是携带了ChannelPromise参数用于写入操作结果。
- connect（SocketAddress remoteAddress，SocketAddress localAddress，ChannelPromise promise）：与方法（11）功能类似，唯一不同的就是绑定了本地地址。
- ChannelFuture bind（SocketAddress localAddress）：绑定指定的本地Socket地址localAddress，该方法会级联触发ChannelHandler.bind（ChannelHandlerContext，SocketAddress，ChannelPromise）事件。
- ChannelFuture bind（SocketAddress localAddress，ChannelPromise promise）：与方法（13）功能类似，多携带了了一个ChannelPromise用于写入操作结果。
- ChannelConfig config（）：获取当前Channel的配置信息，例如CONNECT_TIMEOUT_MILLIS。
- boolean isOpen（）：判断当前Channel是否已经打开。
- boolean isRegistered（）：判断当前Channel是否已经注册到EventLoop上。
- boolean isActive（）：判断当前Channel是否已经处于激活状态。
- ChannelMetadata metadata（）：获取当前Channel的元数据描述信息，包括TCP参数配置等。
- SocketAddress localAddress（）：获取当前Channel的本地绑定地址。
- SocketAddress remoteAddress（）：获取当前Channel通信的远程Socket地址。

他常用的API功能说明
- 第一个比较重要的方法是eventLoop（）。Channel需要注册到EventLoop的多路复用器上，用于处理I/O事件，通过eventLoop（）方法可以获取到Channel注册的EventLoop。EventLoop本质上就是处理网络读写事件的Reactor线程。在Netty中，它不仅仅用来处理网络事件，也可以用来执行定时任务和用户自定义NioTask等任务。
- 第二个比较常用的方法是metadata（）方法。熟悉TCP协议的读者可能知道，当创建Socket的时候需要指定TCP参数，例如接收和发送的TCP缓冲区大小、TCP的超时时间、是否重用地址等。在Netty中，每个Channel对应一个物理连接，每个连接都有自己的TCP参数配置。所以，Channel会聚合一个ChannelMetadata用来对TCP参数提供元数据描述信息，通过metadata（）方法就可以获取当前Channel的TCP参数配置。
- 第三个方法是parent（）。对于服务端Channel而言，它的父Channel为空；对于客户端Channel，它的父Channel就是创建它的ServerSocketChannel。
- 第四个方法是用户获取Channel标识的id（），它返回ChannelId对象，ChannelId是Channel的唯一标识，它的可能生成策略如下。
  （1）机器的MAC地址（EUI-48或者EUI-64）等可以代表全局唯一的信息；
  （2）当前的进程ID；
  （3）当前系统时间的毫秒——System.currentTimeMillis（）；
  （4）当前系统时间纳秒数——System.nanoTime（）；
  （5）32位的随机整型数；
  （6）32位自增的序列数。

## Channel源码分析