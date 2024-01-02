---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码节点状态
假设ElasticSearch集群有一个Master节点，两个Data节点，节点Node节点之间的通信是基于TCP协议，即任意一个Node都需要与其他节点建立连接，这肯定需要连接管理。

## 创建连接管理器
连接管理器ConnectionManager的初始化在TransportService的初始化中完成。
```
public TransportService(Settings settings, Transport transport, ThreadPool threadPool, TransportInterceptor transportInterceptor,
                        Function<BoundTransportAddress, DiscoveryNode> localNodeFactory, @Nullable ClusterSettings clusterSettings,
                        Set<String> taskHeaders) {
    this(settings, transport, threadPool, transportInterceptor, localNodeFactory, clusterSettings, taskHeaders,
        //创建连接管理器
        new ConnectionManager(settings, transport));
}
-----------------------------------------------------------------------
//创建连接管理器
public ConnectionManager(Settings settings, Transport transport) {
    //buildDefaultConnectionProfile创建5类共计13个连接
    this(ConnectionProfile.buildDefaultConnectionProfile(settings), transport);
}
```

## 参数初始化
buildDefaultConnectionProfile初始化ConnectionProfile的属性值。根据ConnectionProfile的配置初始化所有channels，默认创建5类共计13个连接，对应类型、个数、作用如下：

| 类型       | 个数 | 作用                       |
|----------|----|--------------------------|
| RECOVERY | 2  | 用于恢复                     |
| BULK     | 3  | 用于批量写入                   |
| REG      | 6  | 其他用途，例如，加入集群等            |
| STATE    | 1  | 传输集群状态                   |
| PING     | 1  | 用作nodeFD或masterFD的ping请求 |
```
/**
 * Builds a default connection profile based on the provided settings.
 *
 * @param settings to build the connection profile from
 * @return the connection profile
 */
public static ConnectionProfile buildDefaultConnectionProfile(Settings settings) {
    //默认值2 Type.RECOVERY
    int connectionsPerNodeRecovery = TransportSettings.CONNECTIONS_PER_NODE_RECOVERY.get(settings);
    //默认值3 Type.BULK
    int connectionsPerNodeBulk = TransportSettings.CONNECTIONS_PER_NODE_BULK.get(settings);
    //默认值6 Type.REG
    int connectionsPerNodeReg = TransportSettings.CONNECTIONS_PER_NODE_REG.get(settings);
    //默认值1 Type.STATE
    int connectionsPerNodeState = TransportSettings.CONNECTIONS_PER_NODE_STATE.get(settings);
    //默认值1 Type.PING
    int connectionsPerNodePing = TransportSettings.CONNECTIONS_PER_NODE_PING.get(settings);
    Builder builder = new Builder();
    builder.setConnectTimeout(TransportSettings.CONNECT_TIMEOUT.get(settings));
    builder.setHandshakeTimeout(TransportSettings.CONNECT_TIMEOUT.get(settings));
    builder.setPingInterval(TransportSettings.PING_SCHEDULE.get(settings));
    builder.setCompressionEnabled(TransportSettings.TRANSPORT_COMPRESS.get(settings));
    builder.addConnections(connectionsPerNodeBulk, TransportRequestOptions.Type.BULK);
    builder.addConnections(connectionsPerNodePing, TransportRequestOptions.Type.PING);
    // if we are not master eligible we don't need a dedicated channel to publish the state
    builder.addConnections(DiscoveryNode.isMasterNode(settings) ? connectionsPerNodeState : 0, TransportRequestOptions.Type.STATE);
    // if we are not a data-node we don't need any dedicated channels for recovery
    builder.addConnections(DiscoveryNode.isDataNode(settings) ? connectionsPerNodeRecovery : 0, TransportRequestOptions.Type.RECOVERY);
    builder.addConnections(connectionsPerNodeReg, TransportRequestOptions.Type.REG);
    return builder.build();
}
```

## 连接管理
节点间通过openConnection和connectToNode来建立连接，区别是
(1)openConnection建立的连接不能通过ConnectionManager管理，需要发起连接的节点自己管理连接，(2)connectToNode方法建立的连接会通过ConectionManager管理。
```
public void openConnection(DiscoveryNode node, ConnectionProfile connectionProfile, ActionListener<Transport.Connection> listener) {
    ConnectionProfile resolvedProfile = ConnectionProfile.resolveConnectionProfile(connectionProfile, defaultProfile);
    //创建连接
    internalOpenConnection(node, resolvedProfile, listener);
}
```
connectToNode方法创建连接
```
public void connectToNode(DiscoveryNode node, ConnectionProfile connectionProfile,
                          ConnectionValidator connectionValidator,
                          ActionListener<Void> listener) throws ConnectTransportException {
    ConnectionProfile resolvedProfile = ConnectionProfile.resolveConnectionProfile(connectionProfile, defaultProfile);
    ........................
    if (connectingRefCounter.tryIncRef() == false) {
        listener.onFailure(new IllegalStateException("connection manager is closed"));
        return;
    }

    //连接已经存在
    if (connectedNodes.containsKey(node)) {
        connectingRefCounter.decRef();
        listener.onResponse(null);
        return;
    }

    final ListenableFuture<Void> currentListener = new ListenableFuture<>();
    final ListenableFuture<Void> existingListener = pendingConnections.putIfAbsent(node, currentListener);
    if (existingListener != null) {
        try {
            // wait on previous entry to complete connection attempt
            existingListener.addListener(listener, EsExecutors.newDirectExecutorService());
        } finally {
            connectingRefCounter.decRef();
        }
        return;
    }

    currentListener.addListener(listener, EsExecutors.newDirectExecutorService());

    final RunOnce releaseOnce = new RunOnce(connectingRefCounter::decRef);
    //创建连接
    internalOpenConnection(node, resolvedProfile, ActionListener.wrap(conn -> {
        connectionValidator.validate(conn, resolvedProfile, ActionListener.wrap(
            ignored -> {
                assert Transports.assertNotTransportThread("connection validator success");
                try {
                    //连接创建成功
                    if (connectedNodes.putIfAbsent(node, conn) != null) {
                        //相同连接创建重复
                        logger.debug("existing connection to node [{}], closing new redundant connection", node);
                        IOUtils.closeWhileHandlingException(conn);
                    } else {
                        //连接创建成功
                        logger.debug("connected to node [{}]", node);
                        try {
                            connectionListener.onNodeConnected(node, conn);
                        } finally {
                            final Transport.Connection finalConnection = conn;
                            //创建成功的连接增加监听器
                            conn.addCloseListener(ActionListener.wrap(() -> {
                                logger.trace("unregistering {} after connection close and marking as disconnected", node);
                                connectedNodes.remove(node, finalConnection);
                                connectionListener.onNodeDisconnected(node, conn);
                            }));
                        }
                    }
                } finally {
                    ListenableFuture<Void> future = pendingConnections.remove(node);
                    assert future == currentListener : "Listener in pending map is different than the expected listener";
                    releaseOnce.run();
                    future.onResponse(null);
                }
            }, e -> {
                assert Transports.assertNotTransportThread("connection validator failure");
                IOUtils.closeWhileHandlingException(conn);
                failConnectionListeners(node, releaseOnce, e, currentListener);
            }));
    }, e -> {
        assert Transports.assertNotTransportThread("internalOpenConnection failure");
        failConnectionListeners(node, releaseOnce, e, currentListener);
    }));
}
```
接下来详细分析创建连接的过程。					
					
## TcpTransport创建连接
TcpTransport类internalOpenConnection方法核心作用是创建连接，使用ReadWriteLock closeLock保证关闭连接过程中不能创建。
```
public void openConnection(DiscoveryNode node, ConnectionProfile profile, ActionListener<Transport.Connection> listener) {

    Objects.requireNonNull(profile, "connection profile cannot be null");
    if (node == null) {
        throw new ConnectTransportException(null, "can't open connection to a null node");
    }
    //maybeOverrideConnectionProfile：仅仅用于测试
    ConnectionProfile finalProfile = maybeOverrideConnectionProfile(profile);
    closeLock.readLock().lock(); // ensure we don't open connections while we are closing
    try {
        ensureOpen();
        initiateConnection(node, finalProfile, listener);
    } finally {
        closeLock.readLock().unlock();
    }
}
```

## ensureOpen
创建连接过程中，保证TcpTransport组件处于存活状态（TcpTransport继承了AbstractLifecycleComponent类）
```
/**
 * Ensures this transport is still started / open
 *
 * @throws IllegalStateException if the transport is not started / open
 */
private void ensureOpen() {
    if (lifecycle.started() == false) {
        throw new IllegalStateException("transport has been stopped");
    }
}
```

## initiateConnection
initiateConnection核心作用是 创建连接，为连接添加监听器，定时任务检测连接是否超时。

创建连接，建立连接过程如下，如果13个(默认值)连接中有一个连接失败，则整体认为失败，关闭已建立的连接。
```
private List<TcpChannel> initiateConnection(DiscoveryNode node, ConnectionProfile connectionProfile,
                                            ActionListener<Transport.Connection> listener) {
    //获取总连接数13个(默认值)
    int numConnections = connectionProfile.getNumConnections();
    assert numConnections > 0 : "A connection profile must be configured with at least one connection";

    final List<TcpChannel> channels = new ArrayList<>(numConnections);

    for (int i = 0; i < numConnections; ++i) {
        try {
            //1.建立一个连接.
            TcpChannel channel = initiateChannel(node);
            logger.trace(() -> new ParameterizedMessage("Tcp transport client channel opened: {}", channel));
            channels.add(channel);
        } catch (ConnectTransportException e) {
            //如果产生异常，则关闭所有连接(默认13个)
            CloseableChannel.closeChannels(channels, false);
            listener.onFailure(e);
            return channels;
        } catch (Exception e) {
            CloseableChannel.closeChannels(channels, false);
            listener.onFailure(new ConnectTransportException(node, "general node connection failure", e));
            return channels;
        }
    }

    //2.连接添加监听器
    ChannelsConnectedListener channelsConnectedListener = new ChannelsConnectedListener(node, connectionProfile, channels,
        new ThreadedActionListener<>(logger, threadPool, ThreadPool.Names.GENERIC, listener, false));

    for (TcpChannel channel : channels) {
        channel.addConnectListener(channelsConnectedListener);
    }

    TimeValue connectTimeout = connectionProfile.getConnectTimeout();
   //3.使用GENERIC类型线程池，定时任务检测连接是否超时
    threadPool.schedule(channelsConnectedListener::onTimeout, connectTimeout, ThreadPool.Names.GENERIC);
    return channels;
}
```
initiateChannel是建立一个连接的核心方法。createClientBootstrap，创建ClientBootstrap并没有初始化Handler，是在这里完成。
```
protected Netty4TcpChannel initiateChannel(DiscoveryNode node) throws IOException {
    InetSocketAddress address = node.getAddress().address();
    //根据模板Bootstrap克隆具体使用的对象实例
    Bootstrap bootstrapWithHandler = clientBootstrap.clone();
    //注册handler，这里注册的是一个ChannelInitializer对象，当客户端连接服务端成功后向其pipeline设置handler
    //注册的handler则涉及到编码、解码、粘包/拆包解决、报文处理等解决，与服务端handler相对应
    bootstrapWithHandler.handler(getClientChannelInitializer(node));
    bootstrapWithHandler.remoteAddress(address);
    ChannelFuture connectFuture = bootstrapWithHandler.connect();

    Channel channel = connectFuture.channel();
    if (channel == null) {
        ExceptionsHelper.maybeDieOnAnotherThread(connectFuture.cause());
        throw new IOException(connectFuture.cause());
    }
    addClosedExceptionLogger(channel);

    Netty4TcpChannel nettyChannel = new Netty4TcpChannel(channel, false, "default", connectFuture);
    channel.attr(CHANNEL_KEY).set(nettyChannel);

    return nettyChannel;
}
----------------------------------------------------------
protected ChannelHandler getClientChannelInitializer(DiscoveryNode node) {
    return new ClientChannelInitializer();
}
----------------------------------------------------------
protected void initChannel(Channel ch) throws Exception {
    //注册负责记录log的handler，但是进入ESLoggingHandler具体实现
    //可以看到其没有做日志记录操作，源码注释说明因为TcpTransport会做日志记录
    ch.pipeline().addLast("logging", new ESLoggingHandler());
    //注册解码器，这里没有注册编码器因为编码是在TcpTransport实现的，
    //需要发送的报文到达Channel已经是编码之后的格式了
    ch.pipeline().addLast("size", new Netty4SizeHeaderFrameDecoder());
    // using a dot as a prefix means this cannot come from any settings parsed
    //负责对报文进行处理，主要识别是request还是response，然后进行相应的处理
    ch.pipeline().addLast("dispatcher", new Netty4MessageChannelHandler(Netty4Transport.this));
}
```

## 握手
连接管理提到过，节点之前初始化连接之后，节点会向另一个节点发出握手请求。所以其实握手很直接的作用就是，发送一个实际的连接确定是否两节点之间在应用层面上可以通信。换句话说就是，发送请求的节点要确认收到请求的节点是否理解。这个的前提肯定是发送的节点能收到响应，其次就是确认一下是否两个节点的ES版本是否兼容。TCP协议建立连接需要三次握手，对应的初始化在Netty4Transport类初始化调用父类TcpTransport的初始化接口：
```
//TCP 协议握手,发送一个实际的连接确定是否两节点之间在应用层面上可以通信
this.handshaker = new TransportHandshaker(version, threadPool,
    //HandshakeRequestSender
    (node, channel, requestId, v) -> outboundHandler.sendRequest(node, channel, requestId,
        TransportHandshaker.HANDSHAKE_ACTION_NAME, new TransportHandshaker.HandshakeRequest(version),
        TransportRequestOptions.EMPTY, v, false, true),
    //HandshakeResponseSender
    (v, features1, channel, response, requestId) -> outboundHandler.sendResponse(v, features1, channel, requestId,
        TransportHandshaker.HANDSHAKE_ACTION_NAME, response, false, true));
```
## 发送方发送请求
发送方发送请求，把requestId请求的唯一标识和HandshakeResponseHandler响应解析处理器放入pendingHandshakes，等收到接收方的响应是根据requestId获取对应的HandshakeResponseHandler，进行业务处理。
```
/**1. 发送请求
 */
void sendHandshake(long requestId, DiscoveryNode node, TcpChannel channel, TimeValue timeout, ActionListener<Version> listener) {
    numHandshakes.inc();
    // 在发送节点存储将来收到响应时的handler，根据requestId确认
    final HandshakeResponseHandler handler = new HandshakeResponseHandler(requestId, version, listener);
    //把requestId，handler放入pendingHandshakes，方便发送方解析响应
    pendingHandshakes.put(requestId, handler);
    channel.addCloseListener(ActionListener.wrap(
        () -> handler.handleLocalException(new TransportException("handshake failed because connection reset"))));
    boolean success = false;
    try {
        // for the request we use the minCompatVersion since we don't know what's the version of the node we talk to
        // we also have no payload on the request but the response will contain the actual version of the node we talk
        // to as the payload.
        // 将发送方version带入请求并发出
        final Version minCompatVersion = version.minimumCompatibilityVersion();
        handshakeRequestSender.sendRequest(node, channel, requestId, minCompatVersion);
        //处理超时情况
        threadPool.schedule(
            () -> handler.handleLocalException(new ConnectTransportException(node, "handshake_timeout[" + timeout + "]")),
            timeout,
            ThreadPool.Names.GENERIC);
        success = true;
    }
-----------------------------------------------------
//HandshakeRequestSender
(node, channel, requestId, v) -> outboundHandler.sendRequest(node, channel, requestId,
    TransportHandshaker.HANDSHAKE_ACTION_NAME, new TransportHandshaker.HandshakeRequest(version),
    TransportRequestOptions.EMPTY, v, false, true),
```
handshakeRequestSender.sendRequest执行发送请求， 对应的HandshakeRequestSender在TcpTransport初始化时完成。

## 接收方处理请求
接收方处理，并响应发送方的请求，响应内容包含当前节点的ES对应的Version。
```
/**2.接收请求并响应
 *
 */
void handleHandshake(Version version, Set<String> features, TcpChannel channel, long requestId, StreamInput stream) throws IOException {
    // Must read the handshake request to exhaust the stream
    HandshakeRequest handshakeRequest = new HandshakeRequest(stream);
    final int nextByte = stream.read();
    if (nextByte != -1) {
        throw new IllegalStateException("Handshake request not fully read for requestId [" + requestId + "], action ["
            + TransportHandshaker.HANDSHAKE_ACTION_NAME + "], available [" + stream.available() + "]; resetting");
    }
    // 将version响应给发送方
    HandshakeResponse response = new HandshakeResponse(this.version);
    handshakeResponseSender.sendResponse(version, features, channel, response, requestId);
}
-----------------------------------------------------
//HandshakeResponseSender
(v, features1, channel, response, requestId) -> outboundHandler.sendResponse(v, features1, channel, requestId,
    TransportHandshaker.HANDSHAKE_ACTION_NAME, response, false, true));
-----------------------------------------------------
private void sendMessage(TcpChannel channel, OutboundMessage networkMessage, ActionListener<Void> listener) throws IOException {
    MessageSerializer serializer = new MessageSerializer(networkMessage, bigArrays);
    SendContext sendContext = new SendContext(channel, serializer, listener, serializer);
    internalSend(channel, sendContext);
}

//发送请求
private void internalSend(TcpChannel channel, SendContext sendContext) throws IOException {
    channel.getChannelStats().markAccessed(threadPool.relativeTimeInMillis());
    BytesReference reference = sendContext.get();
    try {
        channel.sendMessage(reference, sendContext);
    } catch (RuntimeException ex) {
}
```
handshakeResponseSender.sendResponse响应发送方的请求，对应的HandshakeResponseSender在TcpTransport初始化时完成。

## 发送方解析响应
发送方解析请求，核心时判断Version是否兼容。
```
//3.发送方解析响应
@Override
public void handleResponse(HandshakeResponse response) {
    if (isDone.compareAndSet(false, true)) {
        Version version = response.responseVersion;
        // 判断version是否适配
        if (currentVersion.isCompatible(version) == false) {
            listener.onFailure(new IllegalStateException("Received message from unsupported version: [" + version
                + "] minimal compatible version is: [" + currentVersion.minimumCompatibilityVersion() + "]"));
        } else {
            listener.onResponse(version);
        }
    }
}
```

## 保活机制
握手之后，节点间就需要通过保活来保持连接。虽然Netty中有SO_KEEPALIVE开关来保持TCP的长连接，但是应用层面的保活还是必要的。毕竟TCP是传输层，并不能保证应用的连通性。保活机制对应的是TransportKeepAlive类，对应的初始化在Netty4Transport类初始化调用父类TcpTransport的初始化接口：
```
this.keepAlive = new TransportKeepAlive(threadPool, this.outboundHandler::sendBytes);
```
保持长连接发送的内容是ES的字节形式加上-1：
```
// 发送的内容已经静态初始化好了，实际上就是ES的字节形式加上-1的size
try (BytesStreamOutput out = new BytesStreamOutput()) {
    out.writeByte((byte) 'E');
    out.writeByte((byte) 'S');
    out.writeInt(PING_DATA_SIZE);
    pingMessage = out.bytes();
```

## 初始化
Node节点之间长连接初始化，启动定时任务执行ScheduledPing。
```
//Node节点之间长连接初始化
void registerNodeConnection(List<TcpChannel> nodeChannels, ConnectionProfile connectionProfile) {
    // 获取到ping的间隔
    TimeValue pingInterval = connectionProfile.getPingInterval();
    if (pingInterval.millis() < 0) {
        return;
    }
    // 获取到一个定时ping的工具类
    final ScheduledPing scheduledPing = pingIntervals.computeIfAbsent(pingInterval, ScheduledPing::new);
    //使用GENERIC类型线程池，定时任务启动
    scheduledPing.ensureStarted();

    // 每个channel都需要定时ping
    for (TcpChannel channel : nodeChannels) {
        scheduledPing.addChannel(channel);
        channel.addCloseListener(ActionListener.wrap(() -> scheduledPing.removeChannel(channel)));
    }
}
-----------------------------------------------------
void ensureStarted() {
    if (isStarted.get() == false && isStarted.compareAndSet(false, true)) {
        threadPool.schedule(this, pingInterval, ThreadPool.Names.GENERIC);
    }
}
-----------------------------------------------------
protected void doRunInLifecycle() {
    for (TcpChannel channel : channels) {
        // In the future it is possible that we may want to kill a channel if we have not read from
        // the channel since the last ping. However, this will need to be backwards compatible with
        // pre-6.6 nodes that DO NOT respond to pings
        //先判断需不需要发送
        if (needsKeepAlivePing(channel)) {
            //发送Ping请求
            sendPing(channel);
        }
    }
    this.lastPingRelativeMillis = threadPool.relativeTimeInMillis();
}
```

## 发送请求
发送请求，依旧是ES源码非常常见的函数式编程生产者-消费者模式，最终调用outboundHandler::sendBytes方法。
```
private void sendPing(TcpChannel channel) {
    // 发送ping请求，实际调用的是OutboundHandler#sendBytes，发送了PING_MESSAGE
    //apply: this.outboundHandler::sendBytes
    pingSender.apply(channel, pingMessage, new ActionListener<Void>() {

        @Override
        public void onResponse(Void v) {
            successfulPings.inc();
        }
       ........................................
}
-----------------------------------------------------
void sendBytes(TcpChannel channel, BytesReference bytes, ActionListener<Void> listener) {
    SendContext sendContext = new SendContext(channel, () -> bytes, listener);
    try {
        internalSend(channel, sendContext);
    } catch (IOException e) {
     
}
-----------------------------------------------------
private void internalSend(TcpChannel channel, SendContext sendContext) throws IOException {
    channel.getChannelStats().markAccessed(threadPool.relativeTimeInMillis());
    BytesReference reference = sendContext.get();
    try {
        channel.sendMessage(reference, sendContext);
    } catch (RuntimeException ex) {
}
```
最后通过channel发送。

## 响应
Node收到发送方的Ping请求，开始响应，响应也很简单就是再发一个Ping请求即可。
```
/**响应长连接的定时请求
 * Called when a keep alive ping is received. If the channel that received the keep alive ping is a
 * server channel, a ping is sent back. If the channel that received the keep alive is a client channel,
 * this method does nothing as the client initiated the ping in the first place.
 *
 * @param channel that received the keep alive ping
 */
void receiveKeepAlive(TcpChannel channel) {
    // The client-side initiates pings and the server-side responds. So if this is a client channel, this
    // method is a no-op.
    // 只有当节点是channel的server端时才返回ping请求
    // 因为是client端开始的ping请求流程是这样的：client --ping--> server，server --ping--> client
    if (channel.isServerChannel()) {
        sendPing(channel);
    }
}
```

## 握手与保活机制
在握手确认两个节点之间兼容之后，channel注册到保活服务中去定时发送ping请求。我们看回连接管理器初始化channel完成后的代码：
```
public void onResponse(Void v) {
    // Returns true if all connections have completed successfully
    if (countDown.countDown()) {
        final TcpChannel handshakeChannel = channels.get(0);
        try {
            // 在TransportHandshaker处理器中，执行TCP协议的三次握手
            executeHandshake(node, handshakeChannel, connectionProfile, ActionListener.wrap(version -> {
                NodeChannels nodeChannels = new NodeChannels(node, channels, connectionProfile, version);
                long relativeMillisTime = threadPool.relativeTimeInMillis();
                // 这里可以看到当任何一个channel close时，整个NodeChannels就会close，也就是两节点Connection断掉了，
                // 接下来就需要NodeConnectionsService去重试连接
                nodeChannels.channels.forEach(ch -> {
                    // Mark the channel init time
                    ch.getChannelStats().markAccessed(relativeMillisTime);
                    ch.addCloseListener(ActionListener.wrap(nodeChannels::close));
                });
                /// 所有channel注册到长连接服务中，Node节点之间长连接初始化
                keepAlive.registerNodeConnection(nodeChannels.channels, connectionProfile);
                // 这里是连接管理器connectToNode的回调，返回NodeChannels作为Connection给连接管理器
                listener.onResponse(nodeChannels);
            }, e -> closeAndFail(e instanceof ConnectTransportException ?
                e : new ConnectTransportException(node, "general node connection failure", e))));

        } catch (Exception ex) {
            closeAndFail(ex);
        }
    }
}
```