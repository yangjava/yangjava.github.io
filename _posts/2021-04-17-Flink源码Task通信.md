---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
# Flink源码Task通信
flink数据的通信采用的netty框架，分为客户端和服务端，每个taskmanager即是客户端也是服务端，客户端用于向上游任务请求数据，服务端用于接收下游客户端请求，将数据发送给下游任务。数据处理的逻辑都是在ChannelHandler中完成，客户端和服务端有不同的ChannelHandler

## Netty初始化
在初始化TaskManager的时候，会创建网络服务NetworkEnvironment，同时会启动网络服务，NetworkEnvironment会启动NettyConnectionManager，NettyConnectionManager负责netty的连接，其中包含NettyServer和NettyClient，即netty的服务端和客户端，NettyConnectionManager的启动会初始化NettyServer和NettyClient
```
//NetworkEnvironment类
public void start() throws IOException {
   synchronized (lock) {
      Preconditions.checkState(!isShutdown, "The NetworkEnvironment has already been shut down.");

      LOG.info("Starting the network environment and its components.");

      try {
         LOG.debug("Starting network connection manager");
         connectionManager.start(resultPartitionManager, taskEventDispatcher);
      } catch (IOException t) {
         throw new IOException("Failed to instantiate network connection manager.", t);
      }

      ...
   }
}
```

```
//NettyConnectionManager类
public void start(ResultPartitionProvider partitionProvider, TaskEventDispatcher taskEventDispatcher) throws IOException {
   NettyProtocol partitionRequestProtocol = new NettyProtocol(
      partitionProvider,
      taskEventDispatcher,
      client.getConfig().isCreditBasedEnabled());
   //NettyClient初始化
   client.init(partitionRequestProtocol, bufferPool);
   //NettyServer初始化
   server.init(partitionRequestProtocol, bufferPool);
}
```

## NettyClient初始化
首先来看客户端NettyClient的初始化

客户端的初始化比较简单，只是创建了一个启动器bootstrap，设置了一些参数，此时还并未设置ChannelHandler，也并未去连接服务端，因为此时Task还未启动，也不知道要去连接哪个节点地址。
```
//NettyClient类
void init(final NettyProtocol protocol, NettyBufferPool nettyBufferPool) throws IOException {
   checkState(bootstrap == null, "Netty client has already been initialized.");

   this.protocol = protocol;

   final long start = System.nanoTime();
   //创建启动器bootstrap
   bootstrap = new Bootstrap();

   // --------------------------------------------------------------------
   // Transport-specific configuration
   // --------------------------------------------------------------------

   switch (config.getTransportType()) {
      case NIO: //默认传输类型是NIO
         //设置EventLoopGroup，包括NIO线程数量等
         initNioBootstrap();
         break;

      case EPOLL:
         initEpollBootstrap();
         break;

      case AUTO:
         if (Epoll.isAvailable()) {
            initEpollBootstrap();
            LOG.info("Transport type 'auto': using EPOLL.");
         }
         else {
            initNioBootstrap();
            LOG.info("Transport type 'auto': using NIO.");
         }
   }

   // --------------------------------------------------------------------
   // Configuration
   // --------------------------------------------------------------------

   bootstrap.option(ChannelOption.TCP_NODELAY, true);
   bootstrap.option(ChannelOption.SO_KEEPALIVE, true);

   // Timeout for new connections
   bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, config.getClientConnectTimeoutSeconds() * 1000);

   // Pooled allocator for Netty's ByteBuf instances
   bootstrap.option(ChannelOption.ALLOCATOR, nettyBufferPool);

   // Receive and send buffer size
   int receiveAndSendBufferSize = config.getSendAndReceiveBufferSize();
   if (receiveAndSendBufferSize > 0) {
      bootstrap.option(ChannelOption.SO_SNDBUF, receiveAndSendBufferSize);
      bootstrap.option(ChannelOption.SO_RCVBUF, receiveAndSendBufferSize);
   }

   try {
      clientSSLFactory = config.createClientSSLEngineFactory();
   } catch (Exception e) {
      throw new IOException("Failed to initialize SSL Context for the Netty client", e);
   }

   final long duration = (System.nanoTime() - start) / 1_000_000;
   LOG.info("Successful initialization (took {} ms).", duration);
}
```
NettyServer初始化
再看服务端NettyServer的初始化

服务端的初始化就比较完整了，不仅创建了server端的启动器bootstrap，也添加了server端的ChannelHandlers，同时也启动了server服务
```
void init(final NettyProtocol protocol, NettyBufferPool nettyBufferPool) throws IOException {
   checkState(bootstrap == null, "Netty server has already been initialized.");

   final long start = System.nanoTime();
   //创建启动器
   bootstrap = new ServerBootstrap();

   // --------------------------------------------------------------------
   // Transport-specific configuration
   // --------------------------------------------------------------------

   switch (config.getTransportType()) {
       //和客户端一样
      case NIO:
         initNioBootstrap();
         break;

      case EPOLL:
         initEpollBootstrap();
         break;

      case AUTO:
         if (Epoll.isAvailable()) {
            initEpollBootstrap();
            LOG.info("Transport type 'auto': using EPOLL.");
         }
         else {
            initNioBootstrap();
            LOG.info("Transport type 'auto': using NIO.");
         }
   }

   // --------------------------------------------------------------------
   // Configuration
   // --------------------------------------------------------------------

   // Server bind address
   bootstrap.localAddress(config.getServerAddress(), config.getServerPort());

   // Pooled allocators for Netty's ByteBuf instances
   bootstrap.option(ChannelOption.ALLOCATOR, nettyBufferPool);
   bootstrap.childOption(ChannelOption.ALLOCATOR, nettyBufferPool);

   if (config.getServerConnectBacklog() > 0) {
      bootstrap.option(ChannelOption.SO_BACKLOG, config.getServerConnectBacklog());
   }

   // Receive and send buffer size
   int receiveAndSendBufferSize = config.getSendAndReceiveBufferSize();
   if (receiveAndSendBufferSize > 0) {
      bootstrap.childOption(ChannelOption.SO_SNDBUF, receiveAndSendBufferSize);
      bootstrap.childOption(ChannelOption.SO_RCVBUF, receiveAndSendBufferSize);
   }

   // Low and high water marks for flow control
   // hack around the impossibility (in the current netty version) to set both watermarks at
   // the same time:
   final int defaultHighWaterMark = 64 * 1024; // from DefaultChannelConfig (not exposed)
   final int newLowWaterMark = config.getMemorySegmentSize() + 1;
   final int newHighWaterMark = 2 * config.getMemorySegmentSize();
   if (newLowWaterMark > defaultHighWaterMark) {
      bootstrap.childOption(ChannelOption.WRITE_BUFFER_HIGH_WATER_MARK, newHighWaterMark);
      bootstrap.childOption(ChannelOption.WRITE_BUFFER_LOW_WATER_MARK, newLowWaterMark);
   } else { // including (newHighWaterMark < defaultLowWaterMark)
      bootstrap.childOption(ChannelOption.WRITE_BUFFER_LOW_WATER_MARK, newLowWaterMark);
      bootstrap.childOption(ChannelOption.WRITE_BUFFER_HIGH_WATER_MARK, newHighWaterMark);
   }

   // SSL related configuration
   final SSLHandlerFactory sslHandlerFactory;
   try {
      sslHandlerFactory = config.createServerSSLEngineFactory();
   } catch (Exception e) {
      throw new IOException("Failed to initialize SSL Context for the Netty Server", e);
   }

   // --------------------------------------------------------------------
   // Child channel pipeline for accepted connections
   // --------------------------------------------------------------------

   bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
      @Override
      public void initChannel(SocketChannel channel) throws Exception {
         if (sslHandlerFactory != null) {
            channel.pipeline().addLast("ssl", sslHandlerFactory.createNettySSLHandler());
         }
         //在channel的pipeline添加数据处理的ChannelHandlers
         channel.pipeline().addLast(protocol.getServerChannelHandlers());
      }
   });

   // --------------------------------------------------------------------
   // Start Server
   // --------------------------------------------------------------------
   //启动服务端
   bindFuture = bootstrap.bind().syncUninterruptibly();

   localAddress = (InetSocketAddress) bindFuture.channel().localAddress();

   final long duration = (System.nanoTime() - start) / 1_000_000;
   LOG.info("Successful initialization (took {} ms). Listening on SocketAddress {}.", duration, localAddress);
}
```
我们来看看server端的ChannelHandler都是什么

通过源码看到有消息编码器、解码器、最重要的就是PartitionRequestServerHandler和PartitionRequestQueue，数据的核心处理逻辑都在这两个ChannelHandler里
```
//NettyProtocol类
public ChannelHandler[] getServerChannelHandlers() {
   PartitionRequestQueue queueOfPartitionQueues = new PartitionRequestQueue();
   PartitionRequestServerHandler serverHandler = new PartitionRequestServerHandler(
      partitionProvider, taskEventDispatcher, queueOfPartitionQueues, creditBasedEnabled);

   return new ChannelHandler[] {
      messageEncoder,
      new NettyMessage.NettyMessageDecoder(!creditBasedEnabled),
      serverHandler,
      queueOfPartitionQueues
   };
}
```
我们先不着急分析server端那两个ChannelHandler的实现，先看看客户端的逻辑，因为是客户端先发起请求，服务端才能有所响应，所以要先看客户端。

## 客户端请求数据
上述说道，在初始化阶段客户端只创建了bootstrap，并未有其他的动作。在《Task数据交互之数据读》中提到过，消费端任务线程是从InputGate中获取数据的，而InputGate会调用requestPartitions()来向上游节点发起数据请求，所以我们要从这个方法看起
```
//SingleInputGate类
public void requestPartitions() throws IOException, InterruptedException {
   synchronized (requestLock) {
      if (!requestedPartitionsFlag) {
         if (isReleased) {
            throw new IllegalStateException("Already released.");
         }

         // Sanity checks
         if (numberOfInputChannels != inputChannels.size()) {
            throw new IllegalStateException("Bug in input gate setup logic: mismatch between" +
                  "number of total input channels and the currently set number of input " +
                  "channels.");
         }
          //每个InputChannel也要发起数据请求
         for (InputChannel inputChannel : inputChannels.values()) {
            inputChannel.requestSubpartition(consumedSubpartitionIndex);
         }
      }

      requestedPartitionsFlag = true;
   }
}
```
可以看到，InputGate中有很多InputChannel，每个InputChannel也会发起数据请求，这里我们只看RemoteInputChannel，向远程节点发起数据请求的情况。这里我们也可以看到，InputGate只会发起一次数据请求，之后就不会再发请求了，之后就是基于credit的数据交互了。也可以看到同一个InputGate中的所有RemoteInputChannel请求的SubpartitionIndex都是一样的，也就是说，一个reduce任务会请求上游所有Map任务的相同index的ResultSubPartition，比如都是第二个ResultSubPartition

RemoteInputChannel会创建partitionRequestClient，通过partitionRequestClient向服务端发请求，通过这里我们也看到，每个RemoteInputChannel都会持有一个partitionRequestClient，因为每个RemoteInputChannel都对应了一个上游任务，这些上游任务会分布在不同的节点
```
//RemoteInputChannel类
public void requestSubpartition(int subpartitionIndex) throws IOException, InterruptedException {
   if (partitionRequestClient == null) {
      // Create a client and request the partition
      partitionRequestClient = connectionManager
         .createPartitionRequestClient(connectionId);

      partitionRequestClient.requestSubpartition(partitionId, subpartitionIndex, this, 0);
   }
}
```

## 创建PartitionRequestClient
partitionRequestClient的创建过程最终是调用了NettyClient.connect()方法，在这个过程中，给NettyClient设置了ChannelHandler，并且进行连接服务端。因为存在多个RemoteInputChannel，所以会对上游多个taskmanager都进行连接，每个taskmanager连接都会有一个ChannelHandler，这里我们看到客户端的ChannelHandler默认情况下是CreditBasedPartitionRequestClientHandler，就是基于Credit的处理器。后面的flink版本就只有这一种ChannelHandler了
```
//PartitionRequestClientFactory类
PartitionRequestClient createPartitionRequestClient(ConnectionID connectionId) throws IOException, InterruptedException {
   Object entry;
   PartitionRequestClient client = null;

   while (client == null) {
      entry = clients.get(connectionId);

      if (entry != null) {
         // Existing channel or connecting channel
         //如果通道已经连接过，就直接取。
         //例如有两个map任务在同一个taskmanager的情况，这时只需要连接一次就行，partitionRequestClient共用一个
        if (entry instanceof PartitionRequestClient) {
           client = (PartitionRequestClient) entry;
        }
        else {
           ConnectingChannel future = (ConnectingChannel) entry;
           client = future.waitForChannel();

           clients.replace(connectionId, future, client);
        }
      }
      else {
         // No channel yet. Create one, but watch out for a race.
         ConnectingChannel connectingChannel = new ConnectingChannel(connectionId, this);
         Object old = clients.putIfAbsent(connectionId, connectingChannel);

         if (old == null) {
             //使用NettyClient进行连接服务端
            nettyClient.connect(connectionId.getAddress()).addListener(connectingChannel);
            //等待连接成功
            client = connectingChannel.waitForChannel();

            clients.replace(connectionId, connectingChannel, client);
         }
         ...
   }

   return client;
}
```

```
//NettyClient类
ChannelFuture connect(final InetSocketAddress serverSocketAddress) {
   checkState(bootstrap != null, "Client has not been initialized yet.");

   // --------------------------------------------------------------------
   // Child channel pipeline for accepted connections
   // --------------------------------------------------------------------

   bootstrap.handler(new ChannelInitializer<SocketChannel>() {
      @Override
      public void initChannel(SocketChannel channel) throws Exception {

         // SSL handler should be added first in the pipeline
         if (clientSSLFactory != null) {
            SslHandler sslHandler = clientSSLFactory.createNettySSLHandler(
                  serverSocketAddress.getAddress().getCanonicalHostName(),
                  serverSocketAddress.getPort());
            channel.pipeline().addLast("ssl", sslHandler);
         }
         //在channel的pipeline添加数据处理的ChannelHandlers
         channel.pipeline().addLast(protocol.getClientChannelHandlers());
      }
   });

   try {
       //连接服务端NettyServer
      return bootstrap.connect(serverSocketAddress);
   }
   ...
}
```

```
//NettyProtocol类
public ChannelHandler[] getClientChannelHandlers() {
    //默认情况是CreditBasedPartitionRequestClientHandler
   NetworkClientHandler networkClientHandler =
      creditBasedEnabled ? new CreditBasedPartitionRequestClientHandler() :
         new PartitionRequestClientHandler();
   return new ChannelHandler[] {
      messageEncoder,
      new NettyMessage.NettyMessageDecoder(!creditBasedEnabled),
      networkClientHandler};
}
```

## 发起partition数据请求
创建完partitionRequestClient之后，就会发起数据请求，实现方法在PartitionRequestClient.requestSubpartition()中。

通过源码可以看到，首先创建一个请求实例PartitionRequest，包含了请求的是哪个ResultSubPartition，和当前的RemoteInputChannel初始Credit。Credit是信任消费凭证，具体的介绍可以参考《Flink基于Credit的数据传输和背压》，简单来说就是消费者有多个credit，生产端就能给消费端发送多少个数据buffer，消费端的credit值等于可用于接收数据的空闲buffer数。
```
//PartitionRequestClient类
public ChannelFuture requestSubpartition(
      final ResultPartitionID partitionId,
      final int subpartitionIndex,
      final RemoteInputChannel inputChannel,
      int delayMs) throws IOException {

    ...
    //将InputChannel添加到CreditBasedPartitionRequestClientHandler
   clientHandler.addInputChannel(inputChannel);
   //创建请求体PartitionRequest
   final PartitionRequest request = new PartitionRequest(
         partitionId, subpartitionIndex, inputChannel.getInputChannelId(), inputChannel.getInitialCredit());

   ...

   if (delayMs == 0) {
      ChannelFuture f = tcpChannel.writeAndFlush(request);
      f.addListener(listener);
      return f;
   } 
   ...
}
```
创建完请求实例request以后，将这个请求实例，也可以称之为消息，发送给服务端。接下来就是服务端接收到请求消息之后的逻辑了。

## 服务端处理数据请求
服务端会调用ChannelHander.channelRead()来读取接收到的消息，PartitionRequestServerHandler.channelRead()调用了channelRead0()方法，实现也在channelRead0()中，实现大致如下：

1、接收到客户端的PartitionRequest之后，会给这个请求创建一个reader，在这里我们只看基于Credit的CreditBasedSequenceNumberingViewReader，每个reader都有一个初始凭据credit，值等于消费端RemoteInputChannel的独占buffer数。

2、这个reader随后会创建一个ResultSubpartitionView，reader就是通过这个ResultSubpartitionView来从对应的ResultSubpartition里读取数据，在实时计算里，这个ResultSubpartitionView是PipelinedSubpartitionView的实例。
```
//PartitionRequestServerHandler类
protected void channelRead0(ChannelHandlerContext ctx, NettyMessage msg) throws Exception {
   try {
      Class<?> msgClazz = msg.getClass();
      
      if (msgClazz == PartitionRequest.class) {
         PartitionRequest request = (PartitionRequest) msg;

         LOG.debug("Read channel on {}: {}.", ctx.channel().localAddress(), request);

         try {
            NetworkSequenceViewReader reader;
            if (creditBasedEnabled) {
               reader = new CreditBasedSequenceNumberingViewReader(
                  request.receiverId,
                  request.credit,
                  outboundQueue);
            } else {
               reader = new SequenceNumberingViewReader(
                  request.receiverId,
                  outboundQueue);
            }

            reader.requestSubpartitionView(
               partitionProvider,
               request.partitionId,
               request.queueIndex);

            outboundQueue.notifyReaderCreated(reader);
         } catch (PartitionNotFoundException notFound) {
            respondWithError(ctx, notFound, request.receiverId);
         }
      }
      ...
}

//CreditBasedSequenceNumberingViewReader类
public void requestSubpartitionView(
   ResultPartitionProvider partitionProvider,
   ResultPartitionID resultPartitionId,
   int subPartitionIndex) throws IOException {

   synchronized (requestLock) {
      if (subpartitionView == null) {
         this.subpartitionView = partitionProvider.createSubpartitionView(
            resultPartitionId,
            subPartitionIndex,
            this);
      } else {
         throw new IllegalStateException("Subpartition already requested");
      }
   }
}
```
我们可以分析一下，一个上游（Map）任务对应一个ResultPartition，每个ResultPartition有多个ResultSubPartition，每个ResultSubPartition对应一个下游（Reduce）任务，每个下游任务都会来请求ResultPartition里的一个ResultSubPartition。比如有10个ResultSubPartition，就有10个Reduce任务，每个Reduce发起一个PartitionRequest，Map端就会创建10个reader，每个reader读取一个ResultSubPartition。

## 创建ResultSubpartitionView
ResultSubpartitionView的创建最终是通过PipelinedSubpartitionView.createReadView()来实现， ResultSubpartitionView创建之后会立刻触发数据发送，调用的是PipelinedSubpartition.notifyDataAvailable()方法，这个方法最终会回调到CreditBasedSequenceNumberingViewReader.notifyDataAvailable()方法，触发PartitionRequestQueue.userEventTriggered()，上述说了PartitionRequestQueue也是一个ChannelHandler。
```
//PipelinedSubpartition类
public PipelinedSubpartitionView createReadView(BufferAvailabilityListener availabilityListener) throws IOException {
   final boolean notifyDataAvailable;
   synchronized (buffers) {
      ...
      readView = new PipelinedSubpartitionView(this, availabilityListener);
      notifyDataAvailable = !buffers.isEmpty();
   }
   if (notifyDataAvailable) {
      notifyDataAvailable();
   }

   return readView;
}

//PipelinedSubpartition类
private void notifyDataAvailable() {
   if (readView != null) {
      readView.notifyDataAvailable();
   }
}

//PipelinedSubpartitionView类
public void notifyDataAvailable() {
   availabilityListener.notifyDataAvailable();
}

//CreditBasedSequenceNumberingViewReader类
public void notifyDataAvailable() {
   requestQueue.notifyReaderNonEmpty(this);
}

//PartitionRequestQueue类
void notifyReaderNonEmpty(final NetworkSequenceViewReader reader) {
   ctx.executor().execute(() -> ctx.pipeline().fireUserEventTriggered(reader));
}
```
PartitionRequestQueue.userEventTriggered()会将这个reader（CreditBasedSequenceNumberingViewReader）添加到可用reader队列中，同时触发数据读取和写出，具体实现在writeAndFlushNextMessageIfPossible()中
```
//PartitionRequestQueue类
public void userEventTriggered(ChannelHandlerContext ctx, Object msg) throws Exception {
    
   if (msg instanceof NetworkSequenceViewReader) {
      enqueueAvailableReader((NetworkSequenceViewReader) msg);
   } 
   ...
}

private void enqueueAvailableReader(final NetworkSequenceViewReader reader) throws Exception {
   if (reader.isRegisteredAsAvailable() || !reader.isAvailable()) {
      return;
   }
   // Queue an available reader for consumption. If the queue is empty,
   // we try trigger the actual write. Otherwise this will be handled by
   // the writeAndFlushNextMessageIfPossible calls.
   boolean triggerWrite = availableReaders.isEmpty();
   //将reader添加到可读取队列
   registerAvailableReader(reader);

   if (triggerWrite) {
       //触发数据发送
      writeAndFlushNextMessageIfPossible(ctx.channel());
   }
}

private void registerAvailableReader(NetworkSequenceViewReader reader) {
   availableReaders.add(reader);
   reader.setRegisteredAsAvailable(true);
}
```

发送buffer
下面就是关键的writeAndFlushNextMessageIfPossible()方法，方法大致逻辑如下：

1、从可用reader队列里拿一个reader，reader从ResultSubPartition里读取一个数据buffer，每读一个buffer，凭据credit就减1

2、如果ResultSubPartition里还有可消费的数据，就再次将这个reader添加到可用reader队列里，以便继续读数据。

3、将刚才获取到的buffer进行封装，写出到socket，也就是发送给下游。这里有一个变量很重要，就是buffersInBacklog，这个值是ResultSubPartition中的buffer积压量，就是有多少个buffer积压了，这对于flink的背压判断很重要，这个积压量决定了Reduce应该使用多少个buffer来接收数据。在上述步骤2中，如果说消费端已经没有足够的buffer来接收数据了，那么也就不会再给消费端发送数据了

4、继续循环上面的步骤，直到没有可用的reader了，这时也说明Map端暂时没有可消费的数据了，比如数据还未填满一个buffer，或者还没有到定时flush的时间，或者消费端没有空闲的buffer接收数据（对应的就是生产端的凭据credit等于0了）
```
private void writeAndFlushNextMessageIfPossible(final Channel channel) throws IOException {
   ...

   BufferAndAvailability next = null;
   try {
      while (true) {
          //从可读取reader队列取出一个reader
         NetworkSequenceViewReader reader = pollAvailableReader();

         // No queue with available data. We allow this here, because
         // of the write callbacks that are executed after each write.
         if (reader == null) {
            return;
         }
         //reader从ResultSubPartition里读取一个数据buffer
         next = reader.getNextBuffer();
         if (next == null) {
            ... //没取到数的异常情况
         } else {
            // This channel was now removed from the available reader queue.
            // We re-add it into the queue if it is still available
            if (next.moreAvailable()) {
                //再次将这个reader添加到可用reader队列里
               registerAvailableReader(reader);
            }

            BufferResponse msg = new BufferResponse(
               next.buffer(),
               reader.getSequenceNumber(),
               reader.getReceiverId(),
               next.buffersInBacklog());

            ...

            // Write and flush and wait until this is done before
            // trying to continue with the next buffer.
            //将buffer数据发送出去
            channel.writeAndFlush(msg).addListener(writeListener);

            return;
         }
      }
   } 
   ...
}
```
来看一下reader从ResultSubPartition中读数据的过程，通过subpartitionView从PipelinedSubpartition中获取一个buffer，获取一个就将credit值减1。获取buffer的逻辑简单来说就是从ResultSubPartition的buffers队列里拿一个buffer
```
//CreditBasedSequenceNumberingViewReader类
public BufferAndAvailability getNextBuffer() throws IOException, InterruptedException {
   //通过subpartitionView获取一个buffer
   BufferAndBacklog next = subpartitionView.getNextBuffer();
   if (next != null) {
      sequenceNumber++;
      //获取一个buffer，信任值credit就减1
      if (next.buffer().isBuffer() && --numCreditsAvailable < 0) {
         throw new IllegalStateException("no credit available");
      }
      //对buffer进行封装
      return new BufferAndAvailability(
         next.buffer(), isAvailable(next), next.buffersInBacklog());
   } else {
      return null;
   }
}
```

```
//PipelinedSubpartitionView类
public BufferAndBacklog getNextBuffer() {
   return parent.pollBuffer();
}

//PipelinedSubpartition类
BufferAndBacklog pollBuffer() {
   synchronized (buffers) {
      Buffer buffer = null;

      if (buffers.isEmpty()) {
         flushRequested = false;
      }

      while (!buffers.isEmpty()) {
          //从PipelinedSubpartition的buffers数据队列取队头的buffer
         BufferConsumer bufferConsumer = buffers.peek();
         buffer = bufferConsumer.build();

         checkState(bufferConsumer.isFinished() || buffers.size() == 1,
            "When there are multiple buffers, an unfinished bufferConsumer can not be at the head of the buffers queue.");

         if (buffers.size() == 1) {
            // turn off flushRequested flag if we drained all of the available data
            flushRequested = false;
         }
        //如果buffer是已经被写满的，不是写了一半数据的那种，就可以从buffers队列里删掉了
         if (bufferConsumer.isFinished()) {
            buffers.pop().close();
            decreaseBuffersInBacklogUnsafe(bufferConsumer.isBuffer());
         }

         if (buffer.readableBytes() > 0) {
            break;
         }
         buffer.recycleBuffer();
         buffer = null;
         if (!bufferConsumer.isFinished()) {
            break;
         }
      }

      if (buffer == null) {
         return null;
      }
      //更新PipelinedSubpartition的数据状态
      updateStatistics(buffer);
      // Do not report last remaining buffer on buffers as available to read (assuming it's unfinished).
      // It will be reported for reading either on flush or when the number of buffers in the queue
      // will be 2 or more.
      return new BufferAndBacklog(
         buffer,
         isAvailableUnsafe(),
         getBuffersInBacklog(),
         nextBufferIsEventUnsafe());
   }
}
```

## 生成者主动触发数据发送
上述的过程是下游消费者在开始阶段向上游请求数据，然后触发生产者向下游发送数据。如果在某个时间点，比如上述步骤4中说了，生产者暂时没有数据生产了。那么在有数据生产之后，又是如何触发数据发送的呢？

答案就是每当数据写满一个buffer之后，或者当数据定时flush的时候，都可能会触发PipelinedSubpartition.notifyDataAvailable()，触发之后就又是上述writeAndFlushNextMessageIfPossible()方法的逻辑
```
//PipelinedSubpartition类
private boolean add(BufferConsumer bufferConsumer, boolean finish) {
   checkNotNull(bufferConsumer);

   final boolean notifyDataAvailable;
   synchronized (buffers) {
      if (isFinished || isReleased) {
         bufferConsumer.close();
         return false;
      }

      // Add the bufferConsumer and update the stats
      buffers.add(bufferConsumer);
      updateStatistics(bufferConsumer);
      increaseBuffersInBacklog(bufferConsumer);
      notifyDataAvailable = shouldNotifyDataAvailable() || finish;

      isFinished |= finish;
   }
    //判断是否要触发数据消费，发送给消费者
   if (notifyDataAvailable) {
      notifyDataAvailable();
   }

   return true;
}
```

```
//PipelinedSubpartition类
public void flush() {
   final boolean notifyDataAvailable;
   synchronized (buffers) {
      if (buffers.isEmpty()) {
         return;
      }
      // if there is more then 1 buffer, we already notified the reader
      // (at the latest when adding the second buffer)
      notifyDataAvailable = !flushRequested && buffers.size() == 1;
      flushRequested = true;
   }
   //判断是否要触发数据消费，发送给消费者
   if (notifyDataAvailable) {
      notifyDataAvailable();
   }
}
```

## 客户端接收数据
下面来看客户端（消费端）的逻辑

消费端接收处理数据的起点在CreditBasedPartitionRequestClientHandler.channelRead()方法，该方法接收到消息之后会先将消息进行解码。CreditBasedPartitionRequestClientHandler先拿到这个消息的所属的InputChannel，然后从InputChannel中获取一个buffer，将接收到的消息数据拷贝到buffer中。
```
//CreditBasedPartitionRequestClientHandler类
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
   try {
      decodeMsg(msg);
   } catch (Throwable t) {
      notifyAllChannelsOfErrorAndClose(t);
   }
}

private void decodeMsg(Object msg) throws Throwable {
   final Class<?> msgClazz = msg.getClass();

   // ---- Buffer --------------------------------------------------------
   if (msgClazz == NettyMessage.BufferResponse.class) {
      NettyMessage.BufferResponse bufferOrEvent = (NettyMessage.BufferResponse) msg;

      RemoteInputChannel inputChannel = inputChannels.get(bufferOrEvent.receiverId);
      ...

      decodeBufferOrEvent(inputChannel, bufferOrEvent);

   } 
    ...
}

private void decodeBufferOrEvent(RemoteInputChannel inputChannel, NettyMessage.BufferResponse bufferOrEvent) throws Throwable {
   try {
      ByteBuf nettyBuffer = bufferOrEvent.getNettyBuffer();
      final int receivedSize = nettyBuffer.readableBytes();
      if (bufferOrEvent.isBuffer()) {
         // ---- Buffer ------------------------------------------------

         // Early return for empty buffers. Otherwise Netty's readBytes() throws an
         // IndexOutOfBoundsException.
         if (receivedSize == 0) {
            inputChannel.onEmptyBuffer(bufferOrEvent.sequenceNumber, bufferOrEvent.backlog);
            return;
         }
         //从InputChannel中获取一个buffer
         Buffer buffer = inputChannel.requestBuffer();
         if (buffer != null) {
             //将数据拷贝到buffer中
            nettyBuffer.readBytes(buffer.asByteBuf(), receivedSize);
            //onBuffer()核心处理逻辑
            inputChannel.onBuffer(buffer, bufferOrEvent.sequenceNumber, bufferOrEvent.backlog);
         } else if (inputChannel.isReleased()) {
            cancelRequestFor(bufferOrEvent.receiverId);
         } else {
            throw new IllegalStateException("No buffer available in credit-based input channel.");
         }
      } 
      ...
}
```

## 使用空闲buffer接收数据
RemoteInputChannel获取buffer是从RemoteInputChannel的bufferQueue中获取的，在《Task数据交互之数据读》中我们分析到RemoteInputChannel中有两种类型的buffer，一种是独占buffer，一种是浮动buffer，浮动buffer是从LocalBufferPool中申请的，所有RemoteInputChannel可以共享。这两种类型的buffer都在bufferQueue中。
```
//RemoteInputChannel类
public Buffer requestBuffer() {
   synchronized (bufferQueue) {
      return bufferQueue.takeBuffer();
   }
}

//AvailableBufferQueue类
Buffer takeBuffer() {
    //优化获取浮动buffer
   if (floatingBuffers.size() > 0) {
      return floatingBuffers.poll();
   } else {
      return exclusiveBuffers.poll();
   }
}
```
解码消息之后，就会调用inputChannel.onBuffer()方法来进行核心数据处理了，该方法的处理逻辑大致如下：

1、将上述解码之后的buffer添加到RemoteInputChannel的buffer数据列表中

2、仅添加到buffer数据列表中还不行，在《Task数据交互之数据读》中我们分析到，如果receivedBuffers是空的，这个InputChannel就会被移除出InputGate的inputChannelsWithData队列里，无法再被InputGate轮询，所以当这个InputChannel有数据了之后要再次将自己入队到InputGate中，以便可以让自己继续被InputGate所消费。

3、判断生产端的数据积压量，决定要向LocalBufferPool申请多少个空闲buffer，同时新申请的buffer数要反馈给生产者，增加生产者的credit信任值，以便生产者可以继续往消费者发送数据。
```
public void onBuffer(Buffer buffer, int sequenceNumber, int backlog) throws IOException {
   boolean recycleBuffer = true;

   try {

      final boolean wasEmpty;
      synchronized (receivedBuffers) {
         ...

         wasEmpty = receivedBuffers.isEmpty();
         //将这个buffer添加到自己的buffer数据队列里
         receivedBuffers.add(buffer);
         recycleBuffer = false;
      }

      ++expectedSequenceNumber;

      if (wasEmpty) {
          //让自己可以被InputGate消费
         notifyChannelNonEmpty();
      }

      if (backlog >= 0) {
          //向生产者反馈credit信任值
         onSenderBacklog(backlog);
      }
   } finally {
      if (recycleBuffer) {
         buffer.recycleBuffer();
      }
   }
}
```
简单看下上述2的逻辑，最终会调用SingleInputGate.queueChannel()方法

```
//SingleInputGate类
private void queueChannel(InputChannel channel) {
   int availableChannels;

   synchronized (inputChannelsWithData) {
      if (enqueuedInputChannelsWithData.get(channel.getChannelIndex())) {
         return;
      }
      availableChannels = inputChannelsWithData.size();
       //将自己添加到inputChannelsWithData，可以继续被InputGate轮询
      inputChannelsWithData.add(channel);
      enqueuedInputChannelsWithData.set(channel.getChannelIndex());

      if (availableChannels == 0) {
          //唤醒被阻塞的InputGate
         inputChannelsWithData.notifyAll();
      }
   }

  ...
}
```

## 反馈Credit信任值
这里我们重点看看上述3中的逻辑，实现在RemoteInputChannel.onSenderBacklog()方法中，实现如下：

1、消费者需要的buffer=生产者的数据积压量+初始信任值，可以看到消费者需求的buffer数实际上要略大于生产者的数据积压量的

2、如果InputChannel中没有足够的空闲buffer，会向LocalBufferPool中去申请，申请的这部分属于浮动buffer，申请到了多少buffer，就 增加多少credit信任值。如果申请不到，就暂时给LocalBufferPool添加一个listener，当LocalBufferPool有空闲的buffer的时候，就会把buffer分配给这个InputChannel

3、向生产者发送credit信任值
```
void onSenderBacklog(int backlog) throws IOException {
   int numRequestedBuffers = 0;

   synchronized (bufferQueue) {
      // Similar to notifyBufferAvailable(), make sure that we never add a buffer
      // after releaseAllResources() released all buffers (see above for details).
      if (isReleased.get()) {
         return;
      }

      numRequiredBuffers = backlog + initialCredit;
      while (bufferQueue.getAvailableBufferSize() < numRequiredBuffers && !isWaitingForFloatingBuffers) {
         Buffer buffer = inputGate.getBufferPool().requestBuffer();
         if (buffer != null) {
             //申请到buffer的情况，将buffer作为浮动buffer
            bufferQueue.addFloatingBuffer(buffer);
            numRequestedBuffers++;
         } else if (inputGate.getBufferProvider().addBufferListener(this)) {
            // If the channel has not got enough buffers, register it as listener to wait for more floating buffers.
            //申请不到buffer时暂时给LocalBufferPool添加一个listener，等待分配
            isWaitingForFloatingBuffers = true;
            break;
         }
      }
   }

   if (numRequestedBuffers > 0 && unannouncedCredit.getAndAdd(numRequestedBuffers) == 0) {
      //向生产者发送credit信任值
      notifyCreditAvailable();
   }
}
```

## 其他反馈Credit信任值的时机
上述只是说了RemoteInputChannel.onSenderBacklog()方法中将目前申请到的buffer添加到credit信任值，那其他什么时刻会再次增加信任值呢？

有两个地方：一个就是上述说的当LocalBufferPool有空闲的buffer的时候，就会把buffer分配给这个InputChannel，这个时候会增加信任值；
```
//LocalBufferPool类
public void recycle(MemorySegment segment) {
   BufferListener listener;
   NotificationResult notificationResult = NotificationResult.BUFFER_NOT_USED;
   while (!notificationResult.isBufferUsed()) {
      synchronized (availableMemorySegments) {
         if (isDestroyed || numberOfRequestedMemorySegments > currentPoolSize) {
            returnMemorySegment(segment);
            return;
         } else {
             //有超额的buffer请求，将回收的buffer分配给对应的RemoteInputChannel
            listener = registeredListeners.poll();
            if (listener == null) {
               availableMemorySegments.add(segment);
               availableMemorySegments.notify();
               return;
            }
         }
      }
      notificationResult = fireBufferAvailableNotification(listener, segment);
   }
}

//RemoteInputChannel类
public NotificationResult notifyBufferAvailable(Buffer buffer) {
   NotificationResult notificationResult = NotificationResult.BUFFER_NOT_USED;
   try {
      synchronized (bufferQueue) {
         ...
         //将buffer添加到浮动buffer里
         bufferQueue.addFloatingBuffer(buffer);

         ...
      //同时增加credit信任值
      if (unannouncedCredit.getAndAdd(1) == 0) {
         notifyCreditAvailable();
      }
   } catch (Throwable t) {
      setError(t);
   }
   return notificationResult;
}
```
另一个地方就是当RemoteInputChannel的独占buffer数据消费完进行回收的时候，同样也会增加信任值。也就是说，只要RemoteInputChannel有新的空闲buffer加入的时候，都会增加信任值
```
//RemoteInputChannel类
public void recycle(MemorySegment segment) {
   int numAddedBuffers;

   synchronized (bufferQueue) {
      ...
      numAddedBuffers = bufferQueue.addExclusiveBuffer(new NetworkBuffer(segment, this), numRequiredBuffers);
   }

   if (numAddedBuffers > 0 && unannouncedCredit.getAndAdd(numAddedBuffers) == 0) {
      notifyCreditAvailable();
   }
}
```
notifyCreditAvailable()会调用CreditBasedPartitionRequestClientHandler.userEventTriggered()，最终在writeAndFlushNextMessageIfPossible()方法中将credit以AddCredit消息类型发送给生产者。
```
//CreditBasedPartitionRequestClientHandler类
public void userEventTriggered(ChannelHandlerContext ctx, Object msg) throws Exception {
   if (msg instanceof RemoteInputChannel) {
      boolean triggerWrite = inputChannelsWithCredit.isEmpty();

      inputChannelsWithCredit.add((RemoteInputChannel) msg);

      if (triggerWrite) {
         writeAndFlushNextMessageIfPossible(ctx.channel());
      }
   } else {
      ctx.fireUserEventTriggered(msg);
   }
}

private void writeAndFlushNextMessageIfPossible(Channel channel) {
   if (channelError.get() != null || !channel.isWritable()) {
      return;
   }

   while (true) {
      RemoteInputChannel inputChannel = inputChannelsWithCredit.poll();

      // The input channel may be null because of the write callbacks
      // that are executed after each write.
      if (inputChannel == null) {
         return;
      }

      //It is no need to notify credit for the released channel.
      if (!inputChannel.isReleased()) {
         AddCredit msg = new AddCredit(
            inputChannel.getPartitionId(),
            inputChannel.getAndResetUnannouncedCredit(),
            inputChannel.getInputChannelId());

         // Write and flush and wait until this is done before
         // trying to continue with the next input channel.
         //将Credit消息发送给生产者
         channel.writeAndFlush(msg).addListener(writeListener);

         return;
      }
   }
}
```

## 服务端接收AddCredit消息
我们再回到生产者，也就是Netty的服务端，同样是PartitionRequestServerHandler.channelRead0()方法，生产者再接收到credit信任值后，会给对应的reader增加信任值，意味着可以继续往消费者发送credit个数量的buffer了。PartitionRequestQueue会将这个reader重新添加到可读取reader列表中，使这个reader可以被继续轮询读取ResultSubPartition的数据。如果这个reader已经在可读取的reader中，因为这个reader增加了credit信任值，这使得它可以多读取新增的credit个数据buffer。
```
//PartitionRequestServerHandler类
protected void channelRead0(ChannelHandlerContext ctx, NettyMessage msg) throws Exception {
   try {
      Class<?> msgClazz = msg.getClass();

      ...
      } else if (msgClazz == AddCredit.class) {
         AddCredit request = (AddCredit) msg;

         outboundQueue.addCredit(request.receiverId, request.credit);
      } 
      ...
}

//PartitionRequestQueue类
void addCredit(InputChannelID receiverId, int credit) throws Exception {
   if (fatalError) {
      return;
   }

   NetworkSequenceViewReader reader = allReaders.get(receiverId);
   if (reader != null) {
       //给reader增加信任值
      reader.addCredit(credit);
      //重新入队reader
      enqueueAvailableReader(reader);
   } else {
      throw new IllegalStateException("No reader for receiverId = " + receiverId + " exists.");
   }
}
```
credit的用处在生产者中的体现是在reader.isAvailable()方法，只有当credit信任值>0时，reader才可以继续读取ResultSubPartition的数据

```
//CreditBasedSequenceNumberingViewReader类
public boolean isAvailable() {
   // BEWARE: this must be in sync with #isAvailable(BufferAndBacklog)!
   return hasBuffersAvailable() &&
      (numCreditsAvailable > 0 || subpartitionView.nextBufferIsEvent());
}
```
在此之后，生产者、消费者之间将循环进行这个数据交互过程，生产者将数据发送给消费者，消费者反馈credit给生产者，使得数据可以进行持续的生产、消费。到此，这个数据交互过程也就分析完毕了！

总结：

1、flink数据的通信采用的netty框架，分为客户端和服务端，每个taskmanager即是客户端也是服务端，客户端用于向上游任务请求数据，服务端用于接收下游客户端请求，将数据发送给下游任务，数据处理的逻辑都是在ChannelHandler中完成

2、在初始化TaskManager的时候，就会初始化NettyServer和NettyClient，客户端NettyClient的初始化比较简单，只是创建了一个启动器bootstrap，设置了一些参数，此时还并未设置ChannelHandler，也并未去连接服务端。服务端NettyServer的初始化就比较完整了，不仅创建了server端的启动器bootstrap，设置EventLoopGroup，也添加了server端的ChannelHandlers，最重要的就是PartitionRequestServerHandler和PartitionRequestQueue；同时也启动了server服务。

3、消费端任务线程从InputGate中获取数据，而InputGate会调用requestPartitions()来向上游节点发起数据请求，InputGate中有很多RemoteInputChannel，每个RemoteInputChannel会创建partitionRequestClient，通过partitionRequestClient向服务端发请求，partitionRequestClient的创建过程会调用NettyClient.connect()方法，在这个过程中，给NettyClient设置了ChannelHandler，默认情况下是CreditBasedPartitionRequestClientHandler，就是基于Credit的处理器，并且进行连接服务端。

4、创建完partitionRequestClient之后，就会发起数据请求，首先创建一个请求实例PartitionRequest，包含了请求的是哪个ResultSubPartition，和当前的RemoteInputChannel初始Credit。Credit是信任消费凭证，简单来说就是消费者有多个credit，生产端就能给消费端发送多少个数据buffer

5、服务端会调用ChannelHander.channelRead()来读取接收到的消息，接收到客户端的PartitionRequest之后，会给这个请求创建一个reader，即CreditBasedSequenceNumberingViewReader，这个reader随后会创建一个ResultSubpartitionView，reader就是通过这个ResultSubpartitionView来从对应的ResultSubpartition里读取数据。生产者的每个ResultSubPartition都会对应一个下游任务的请求（每个RemoteInputChannel消费一个ResultSubPartition），同时都会有一个reader。

6、ResultSubpartitionView创建之后会立刻触发数据发送，触发PartitionRequestQueue.writeAndFlushNextMessageIfPossible()方法，简单来说数据的处理逻辑就是reader从ResultSubPartition中读取一个buffer，同时信任值credit减1；如果ResultSubPartition中还有可用的数据会继续入队被轮询读数据，如果reader中的credit信任值等于0了，也不能再消费了；最后会将buffer进行封装发送给消费者，同时还封装了ResultSubPartition中的积压量信息

7、消费者同样调用CreditBasedPartitionRequestClientHandler.channelRead()来处理接收到的数据，先拿到这个消息的所属的InputChannel，然后从InputChannel中获取一个buffer，RemoteInputChannel中有两种类型的buffer，一种是独占buffer，一种是浮动buffer，浮动buffer是从LocalBufferPool中申请的。将接收到的消息数据拷贝到buffer中，再将buffer添加到RemoteInputChannel的buffer数据列表中，同时将自己入队到InputGate中，以便可以让自己继续被InputGate所轮询消费。

8、消费端RemoteInputChannel判断生产端的数据积压量，决定要向LocalBufferPool申请多少个空闲buffer，同时新申请的buffer数要反馈给生产者，增加生产者的credit信任值，以便生产者可以继续往消费者发送数据。当RemoteInputChannel有新的空闲buffer加入的时候，都会增加信任值，反馈给生产者。最终调用CreditBasedPartitionRequestClientHandler.writeAndFlushNextMessageIfPossible()发送AddCredit消息给生产者，也即Netty的服务端

9、生产者（服务端）再接收到credit信任值后，会给对应的reader增加信任值，意味着可以继续往消费者发送credit个数量的buffer了。如果当前的reader因为没有credit而无法继续读取数据，PartitionRequestQueue会将这个reader重新添加到可读取reader列表中，使这个reader可以被继续轮询读取ResultSubPartition的数据。然后继续发送数据给消费者（客户端）

10、在此之后，生产者、消费者之间将循环进行这个数据交互过程，生产者将数据发送给消费者，消费者反馈credit给生产者，使得数据可以进行持续的生产、消费。