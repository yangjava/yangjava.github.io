---
layout: post
categories: RocketMQ
description: none
keywords: RocketMQ
---
# RocketMQ源码-Broker处理消息请求

分析broker存储消息之前，首先回顾下生产者发送消息的流程。生产者首先通过查询缓存在本地的topic，如果本地没有缓存topic信息，就从Name Server服务器上拉取topic信息，默认轮询的方式选择topic的消息队列获取Broker Name，通过Broker Name找到Broker 地址，就知道消息应该发送到哪个Broker服务器了，RocketMQ的消息只能发送到Master Broker 服务器上。然后通过Broker 地址查找生产者与Broker服务器的channel（连接），如果连接不存在，则创建，将消息通过连接用不同的发送方式（单向、同步、异步）发送出去，这就是消息发送的大致流程了。

那消息发送到Broker服务器以后，Broker服务器是如何处理请求的？Broker是如何存储消息的呢？

Broker在启动的时候会注册各种处理器，当Broker服务器接收到请求时，先将请求进行处理，根据不同请求码将请求交给不同的处理器处理。Broker服务器会将消息交给SendMessageProcessor处理器，SendMessageProcessor接收到消息以后交给processRequest方法处理。这篇文章先分析下Broker如何处理请求，下一篇文章再分析Broker服务如何存储普通消息。

### Broker处理请求以及响应

我们先看看Broker服务器接收到消息以后是怎么交给SendMessageProcessor处理器的，Broker 是通过netty 处理各种请求通信的，当Broker启动时，就会启动netty，当消息发送给Broker服务器时，消息会被交给NettyServerHandler的channelRead0方法处理，代码如下：

```java
//代码位置：org.apache.rocketmq.remoting.netty.NettyRemotingServer.NettyServerHandler
@ChannelHandler.Sharable
class NettyServerHandler extends SimpleChannelInboundHandler<RemotingCommand> {

        @Override
        protected void channelRead0(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
            processMessageReceived(ctx, msg);
        }
}
```

channelRead0方法又将收到的请求或者响应交给processMessageReceived方法处理，processMessageReceived方法如下：

```java
//代码位置：org.apache.rocketmq.remoting.netty.NettyRemotingAbstract#processMessageReceived
public void processMessageReceived(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
        final RemotingCommand cmd = msg;
        if (cmd != null) {
            //根据类型处理不同的请求
            switch (cmd.getType()) {
                case REQUEST_COMMAND:
                    //请求
                    processRequestCommand(ctx, cmd);
                    break;
                    //响应
                case RESPONSE_COMMAND:
                    processResponseCommand(ctx, cmd);
                    break;
                default:
                    break;
            }
        }
}
```

processMessageReceived方法接收到请求以后，根据请求的类型做不同的逻辑处理，当类型是请求时，调用processRequestCommand方法处理，当类型是响应时，调用processResponseCommand方法处理。当生产者发送请求给Broker服务器时，会调用processRequestCommand方法处理，我们深入processRequestCommand方法：

### Broker处理请求

Broker服务器在启动的时候，会注册各种处理器，实际就是将请求码与请求码对应的处理器、执行线程保存在processorTable中。如下代码所示：

```java
//代码位置：org.apache.rocketmq.remoting.netty.NettyRemotingServer#registerProcessor
public void registerProcessor(int requestCode, NettyRequestProcessor processor, ExecutorService executor) {
        ExecutorService executorThis = executor;
        if (null == executor) {
            executorThis = this.publicExecutor;
        }

        //处理器、线程执行器
        Pair<NettyRequestProcessor, ExecutorService> pair = new Pair<NettyRequestProcessor, ExecutorService>(processor, executorThis);
        //缓存在map中
        this.processorTable.put(requestCode, pair);
}
```

processRequestCommand方法处理接收到的请求，代码如下：

```java
//代码位置：org.apache.rocketmq.remoting.netty.NettyRemotingAbstract#processRequestCommand
public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {
        //通过请求码找到处理器与执行线程
        final Pair<NettyRequestProcessor, ExecutorService> matched = this.processorTable.get(cmd.getCode());
        final Pair<NettyRequestProcessor, ExecutorService> pair = null == matched ? this.defaultRequestProcessor : matched;
        //请求id，根据请求id可以找到对应的响应
        final int opaque = cmd.getOpaque();

        if (pair != null) {
            //代码省略：处理器处理消息
        } else {
            //没有找到请求处理器，返回不支持该请求的响应
            String error = " request type " + cmd.getCode() + " not supported";
            final RemotingCommand response =
                RemotingCommand.createResponseCommand(RemotingSysResponseCode.REQUEST_CODE_NOT_SUPPORTED, error);
            response.setOpaque(opaque);
            ctx.writeAndFlush(response);
            log.error(RemotingHelper.parseChannelRemoteAddr(ctx.channel()) + error);
        }
}
```

processRequestCommand方法首先通过请求码获取处理器和执行线程的成对关系pair，当pair不等于null，就将请求交给获取的处理器和执行线程进行处理，这里的代码省略了，代码比较长。当pair等于null，返回不支持该请求的响应。找到处理器和执行线程时，就将请求交给处理器和执行线程处理，代码如下：

```java
//代码位置：org.apache.rocketmq.remoting.netty.NettyRemotingAbstract#processRequestCommand
public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {

        //省略代码

        if (pair != null) {
            Runnable run = new Runnable() {
                @Override
                public void run() {
                    try {
                        //执行钩子方法
                        doBeforeRpcHooks(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd);
                        //响应回调
                        final RemotingResponseCallback callback = new RemotingResponseCallback() {
                            @Override
                            public void callback(RemotingCommand response) {、
                                //执行钩子方法
                                doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd, response);
                                //如果不是单向的请求，则将结果返回
                                if (!cmd.isOnewayRPC()) {
                                    if (response != null) {
                                        response.setOpaque(opaque);
                                        response.markResponseType();
                                        try {
                                            ctx.writeAndFlush(response);
                                        } catch (Throwable e) {
                                            log.error("process request over, but response failed", e);
                                            log.error(cmd.toString());
                                            log.error(response.toString());
                                        }
                                    } else {
                                    }
                                }
                            }
                        };
                        //异步请求处理器
                        if (pair.getObject1() instanceof AsyncNettyRequestProcessor) {
                            AsyncNettyRequestProcessor processor = (AsyncNettyRequestProcessor)pair.getObject1();
                            processor.asyncProcessRequest(ctx, cmd, callback);
                        } else {
                            //同步请求处理，等到处理结果以后在回调
                            NettyRequestProcessor processor = pair.getObject1();
                            //调用处理器的processRequest方法
                            RemotingCommand response = processor.processRequest(ctx, cmd);、
                             //处理器处理以后，调用钩子方法
                            doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd, response);
                            //回调方法
                            callback.callback(response);
                        }
                    } catch (Throwable e) {
                        log.error("process request exception", e);
                        log.error(cmd.toString());
                        //如果不是单向的请求，则将返回系统异常的响应
                        if (!cmd.isOnewayRPC()) {
                            final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_ERROR,
                                RemotingHelper.exceptionSimpleDesc(e));
                            response.setOpaque(opaque);
                            ctx.writeAndFlush(response);
                        }
                    }
                }
            };

            //省略代码
        } else {
            //没有找到请求处理器，返回不支持该请求的响应
           //代码省略
        }
}
```

上述代码首先创建了Runnable类，Runnable类的run方法在处理器处理请求之前，首先执行doBeforeRpcHooks钩子方法，然后创建了响应回调callback，当处理器是异步处理器，则采用异步处理器的asyncProcessRequest方法处理请求，得到响应以后，在调用回调响应callback的callback方法。当处理器是同步处理器时，调用处理器的processRequest方法处理，然后处理器处理完请求以后，调用doAfterRpcHooks钩子方法，最后才调用回调响应callback的callback方法。当run方法发生异常时，如果不是单向的请求，则将返回系统异常的响应。

在run方法中创建了回调响应RemotingResponseCallback，我们看看RemotingResponseCallback的callback方法到底做了什么？

callback方法首先会调用doAfterRpcHooks钩子方法，如果不是单向的请求，并且响应结果response不等null，则将响应结果response写回给发送请求的客户端，这个客户端可以是生产者。也可以是消费者。

```java
//代码位置：org.apache.rocketmq.remoting.netty.NettyRemotingAbstract#processRequestCommand
public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {
       //省略代码

        if (pair != null) {

            //省略代码：创建Runnable对象

            //如果拒绝服务，那么将返回系统繁忙的响应
            if (pair.getObject1().rejectRequest()) {
                final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
                    "[REJECTREQUEST]system busy, start flow control for a while");
                response.setOpaque(opaque);
                ctx.writeAndFlush(response);
                return;
            }

            try {
                //提交给线程处理
                final RequestTask requestTask = new RequestTask(run, ctx.channel(), cmd);
                pair.getObject2().submit(requestTask);
            } catch (RejectedExecutionException e) {
                if ((System.currentTimeMillis() % 10000) == 0) {
                    log.warn(RemotingHelper.parseChannelRemoteAddr(ctx.channel())
                        + ", too many requests and system thread pool busy, RejectedExecutionException "
                        + pair.getObject2().toString()
                        + " request code: " + cmd.getCode());
                }

                //如果不是单向的请求
                if (!cmd.isOnewayRPC()) {
                    final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
                        "[OVERLOAD]system busy, start flow control for a while");
                    response.setOpaque(opaque);
                    ctx.writeAndFlush(response);
                }
            }
        } else {
            //没有找到请求处理器，返回不支持该请求的响应
           //代码省略
        }
}
```

创建完Runnable类后，先判断下处理器是否拒绝请求，如果拒绝请求，就返回系统繁忙的响应。然后将请求封装为RequestTask任务交给执行线程执行，当执行线程执行过程发生了异常，如果是单向的请求，就返回系统繁忙的响应。分析到这里，Broker处理请求的过程就分析完，接下来，分析下Broker处理响应的流程。

### Broker处理响应

客户端会发送请求给Broker服务器，同样地，Broker服务器也会收到请求返回的响应结果。processResponseCommand方法就是处理请求的响应结果的，代码如下：

```java
//代码位置：org.apache.rocketmq.remoting.netty.NettyRemotingAbstract#processResponseCommand
public void processResponseCommand(ChannelHandlerContext ctx, RemotingCommand cmd) {
        //获取请求id
        final int opaque = cmd.getOpaque();
        //获取结果响应
        final ResponseFuture responseFuture = responseTable.get(opaque);
        if (responseFuture != null) {
            responseFuture.setResponseCommand(cmd);

            //将缓存的结果响应删除
            responseTable.remove(opaque);

            //如果执行回调不为空，则执行回调
            if (responseFuture.getInvokeCallback() != null) {
                executeInvokeCallback(responseFuture);
            } else {
                //否则设置响应和释放信号量
                responseFuture.putResponse(cmd);
                responseFuture.release();
            }
        } else {
            log.warn("receive response, but not matched any request, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()));
            log.warn(cmd.toString());
        }

}
```

当发送请求时，RocketMQ怎么通过请求找到响应结果？发送请求时，会生成一个唯一的id，这个id叫做请求id，然后将请求id与响应结果Future保存在map中，这样通过请求id就可以找到对应的响应结果了。processResponseCommand方法首先从响应命令cmd中获取到请求id，然后通过请求id从responseTable获取到响应结果，当响应结果responseFuture等于null时，说明接收的响应结果没有找到请求，只打印警告日志，当responseFuture不等于null时，从responseTable删除请求id对应的响应。如果responseFuture有回调方法，则执行回调方法executeInvokeCallback，否则设置响应结果和释放信号量，释放信号量是因为在请求的时候，首先需要获取信号量才能继续处理请求，当没有获取的信号量，说明RocketMQ比较繁忙，不会处理请求，所以在处理响应结果的时候，需要释放在请求时获取的信号量，这样下一个请求过来的时候，就可以获取信号量进行请求了。

Broker服务器处理请求和响应的逻辑不复杂，就是利用netty接收请求和响应，然后根据不同的类型进行请求和响应的处理。处理请求时，根据请求码获取对应处理器以及线程执行器。然后将请求封装任务，交给线程执行器进行执行，处理器负责具体请求的具体逻辑处理。当请求响应返回时，Broker服务器根据请求id‘找到对应响应结果Future，然后判断Future是否回调方法，如果有的话，则调用回调方法，最后，需要将请求-响应关系从缓存中删除，并且释放请求过程中获取的信号量，供下个请求使用。