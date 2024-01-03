---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码RPC请求与处理
Node节点内部之间的通信称为RPC请求。Node节点之间内部通信流程与Rest请求类似，一个RPC请求需要有对应的处理器。

RPC （Remote Procedure Call）远程过程调用，在分布式应用中，RPC 是一个计算机通信协议，核心目的是实现进程间通信。例如：Mater节点收到创建、删除索引的请求，Master节点操作完成后，需要其他Data节点执行操作，这就需要RPC。在ElasticSearch中，无论是Http对应的Rest请求，还是TCP的RPC请求都是基于Netty。

## RPC的注册和映射
与REST Action 的注册类似，内部RPC也注册在ActionModule类中，描述某个Action应该被哪个类处理。一个Action可以理解为RPC调用的名称。一个RPC请求与处理模块的对应关系在两方面维护：

在ActionModule类中注册Action与处理类的映射;
通过TransportService#registerRequestHandler方法注册Action名称与对应处理器的映射。
这两种映射关系同时存在。RPC先在ActionModule 类中注册，再调用TransportService#registerRequestHandler在TransportService 类中注册。在大部分情况下，网络层收到请求后根据TransportService注册的Action信息获取对应的处理模块。

## ActionModule类注册
Node初始化：Node启动初始化操作中，包含ActionModule类的初始化。
```
//绑定Http的TransportActions与节点内部RPC的Actions关系
 ActionModule actionModule = new ActionModule(false, settings, clusterModule.getIndexNameExpressionResolver(),
                settingsModule.getIndexScopedSettings(), settingsModule.getClusterSettings(), settingsModule.getSettingsFilter(),
                threadPool, pluginsService.filterPlugins(ActionPlugin.class), client, circuitBreakerService, usageService, clusterService);
modules.add(actionModule);     
```
其中 new ActionModule 进行ActionModule初始化中，这里比较重要的是两个方法：
```
public ActionModule(......){
    .......................................................
    //RPC先在ActionModule 类中注册，再调用TransportService#registerRequestHandler在TransportService 类中注册
    actions = setupActions(actionPlugins);
}
```
setupActions方法把Node节点之间的请求与对应的Handler进行绑定。
```
//RPC先在ActionModule 类中注册，再调用TransportService#registerRequestHandler在TransportService 类中注册
//与REST Action 的注册类似，内部RPC也注册在ActionModule类中，描述某个Action应该被哪个类处理。
//一个Action可以理解为RPC调用的名称。Action 与处理类的映射关系如下:
static Map<String, ActionHandler<?, ?>> setupActions(List<ActionPlugin> actionPlugins) {
    // Subclass NamedRegistry for easy registration
    class ActionRegistry extends NamedRegistry<ActionHandler<?, ?>> {
        ActionRegistry() {
            super("action");
        }

        public void register(ActionHandler<?, ?> handler) {
            register(handler.getAction().name(), handler);
        }

        public <Request extends ActionRequest, Response extends ActionResponse> void register(
            ActionType<Response> action, Class<? extends TransportAction<Request, Response>> transportAction,
            Class<?>... supportTransportActions) {
            //action与handler的绑定
            register(new ActionHandler<>(action, transportAction, supportTransportActions));
        }
    }
    ActionRegistry actions = new ActionRegistry();

    actions.register(MainAction.INSTANCE, TransportMainAction.class);
.......................................................
   //索引创建
   actions.register(CreateIndexAction.INSTANCE, TransportCreateIndexAction.class);
.......................................................
   //索引创建
   actions.register(DeleteIndexAction.INSTANCE, TransportDeleteIndexAction.class);
.......................................................
```
Action与ActionHandler的绑定关系共有92个，setUpActions函数执行完成后如下图，key是Action的name，value是对应的处理器。

## TransportService注册
以TransportCreateIndexAction为例，在初始化阶段，最终调用 HandledTransportAction的HandledTransportAction方法，把Action注册到transportService中。
```
protected HandledTransportAction(String actionName, boolean canTripCircuitBreaker,
                                 TransportService transportService, ActionFilters actionFilters,
                                 Writeable.Reader<Request> requestReader, String executor) {
    super(actionName, actionFilters, transportService.getTaskManager());
    transportService.registerRequestHandler(actionName, executor, false, canTripCircuitBreaker, requestReader,
        new TransportHandler());
}
```
registerRequestHandler注册Action和对应的处理类TransportRequest-Handler，在ActionModule类中注册的RPC信息会自动在TransportService中添加映射关系。
```
/**
 * Registers a new request handler
 *
 * @param action         The action the request handler is associated with
 * @param requestReader  a callable to be used construct new instances for streaming
 * @param executor       The executor the request handling will be executed on
 * @param handler        The handler itself that implements the request handling
 */
public <Request extends TransportRequest> void registerRequestHandler(String action, String executor,
                                                                      Writeable.Reader<Request> requestReader,
                                                                      TransportRequestHandler<Request> handler) {
    validateActionName(action);
    handler = interceptor.interceptHandler(action, executor, false, handler);
    RequestHandlerRegistry<Request> reg = new RequestHandlerRegistry<>(
        action, requestReader, taskManager, handler, executor, false, true);
    transport.registerRequestHandler(reg);
}

/**registerRequestHandler注册Action和对应的处理类TransportRequest-Handler
 * 在ActionModule类中注册的RPC信息会自动在TransportService中添加映射关系。
 *
 * Registers a new request handler
 *
 * @param action                The action the request handler is associated with
 * @param requestReader         The request class that will be used to construct new instances for streaming
 * @param executor              The executor the request handling will be executed on
 * @param forceExecution        Force execution on the executor queue and never reject it
 * @param canTripCircuitBreaker Check the request size and raise an exception in case the limit is breached.
 * @param handler               The handler itself that implements the request handling
 */
public <Request extends TransportRequest> void registerRequestHandler(String action, //action对应handler的名字
                                                                      String executor,//executor对应了线程的类型，执行时会从线程池中拿相应类型的线程来处理request
                                                                      boolean forceExecution,
                                                                      boolean canTripCircuitBreaker,
                                                                      Writeable.Reader<Request> requestReader,//requestReader对应请求的reader，服务端根据reader来从网络IO流中反序列化出相应对象
                                                                      TransportRequestHandler<Request> handler) {//handler对应实际的handler，做实际的处理工作
    validateActionName(action);
    handler = interceptor.interceptHandler(action, executor, forceExecution, handler);
    RequestHandlerRegistry<Request> reg = new RequestHandlerRegistry<>(
        action, requestReader, taskManager, handler, executor, forceExecution, canTripCircuitBreaker);
    transport.registerRequestHandler(reg);
}
```
这里的重点是RequestHandlerRegistry类的构造。

action: action对应handler的名字;
taskManager: 保持当前Node运行的task;
executor: executor对应了线程的类型，执行时会从线程池中拿相应类型的线程来处理request;
requestReader: requestReader对应请求的reader，服务端根据reader来从网络IO流中反序列化出相应对象;
handler: 发给远方节点执行后返回的结果的回调函数，主要功能是整合这些返回信息，返回给用户或是再分发到其他节点上

## RPC调用
TransportService类是在网络层之上，属于服务层。Netty4Transport继承了TcpTransport属于传输层。

### TransportService#sendRequest
当需要发送到网络时，TransportService#sendRequest调用asyncSender.sendRequest 执行发送，最后调用sendRequestInternal当前节点向其他节点发送请求。在发送请求时注册了一个对应requestId的responseHandler，然后在接收请求时拿出来requestId对应handler。
```
private <T extends TransportResponse> void sendRequestInternal(final Transport.Connection connection, final String action,
                                                               final TransportRequest request,
                                                               final TransportRequestOptions options,
                                                               TransportResponseHandler<T> handler) {

    DiscoveryNode node = connection.getNode();
    Supplier<ThreadContext.StoredContext> storedContextSupplier = threadPool.getThreadContext().newRestorableContext(true);
    ContextRestoreResponseHandler<T> responseHandler = new ContextRestoreResponseHandler<>(storedContextSupplier, handler);
    // TODO we can probably fold this entire request ID dance into connection.sendReqeust but it will be a bigger refactoring
    //在发送请求时注册了一个对应requestId的responseHandler，然后在接收请求时拿出来requestId对应handler。
    final long requestId = responseHandlers.add(new Transport.ResponseContext<>(responseHandler, connection, action));
    final TimeoutHandler timeoutHandler;
    //Request指定超时
    if (options.timeout() != null) {
        timeoutHandler = new TimeoutHandler(requestId, connection.getNode(), action);
        responseHandler.setTimeoutHandler(timeoutHandler);
    } else {
        timeoutHandler = null;
    }
    try {
        if (lifecycle.stoppedOrClosed()) {
            /*
             * If we are not started the exception handling will remove the request holder again and calls the handler to notify the
             * caller. It will only notify if toStop hasn't done the work yet.
             *
             * Do not edit this exception message, it is currently relied upon in production code!
             */
            // TODO: make a dedicated exception for a stopped transport service? cf. ExceptionsHelper#isTransportStoppedForAction
            throw new TransportException("TransportService is closed stopped can't send request");
        }
        if (timeoutHandler != null) {
            assert options.timeout() != null;
            //定时检测是否超时
            timeoutHandler.scheduleTimeout(options.timeout());
        }
        // TcpTransport#sendRequest
        connection.sendRequest(requestId, action, request, options); // local node optimization happens upstream
    }
```

### Netty4Transport.NodeChannels#sendRequest
Netty4Transport继承了TcpTransport类，是基于Netty封装的传输层，为业务TransportService业务层提供传输。

NodeChannels#sendRequest
sendRequest方法，channel(options.type())首先根据type获取对应的Channel。Type是5种类型，共计13个（默认值）连接，根据Type获取后，以round-robin（路由方式）随机从中抽取一个。
```
public void sendRequest(long requestId, String action, TransportRequest request, TransportRequestOptions options)
    throws IOException, TransportException {
    //判断了一下传入的options参数属于channel的哪一个类型，从连接池中选择对应类型的连接使用
    //通过请求类型获取13个连接中的相应连接
    TcpChannel channel = channel(options.type());
    //发送请求
    outboundHandler.sendRequest(node, channel, requestId, action, request, options, getVersion(), compress, false);
}
```

### OutboundHandler#sendRequest

OutboundHandler的初始化，在Netty4Transport类初始化时调用父类TcpTransport初始化时完成。OutboundHandler#sendRequest对应的方法及调用链如下：
```
void sendRequest(final DiscoveryNode node, final TcpChannel channel, final long requestId, final String action,
                 final TransportRequest request, final TransportRequestOptions options, final Version channelVersion,
                 final boolean compressRequest, final boolean isHandshake) throws IOException, TransportException {
    Version version = Version.min(this.version, channelVersion);
    OutboundMessage.Request message = new OutboundMessage.Request(threadPool.getThreadContext(), features, request, version, action,
        requestId, isHandshake, compressRequest);
    ActionListener<Void> listener = ActionListener.wrap(() ->
        messageListener.onRequestSent(node, requestId, action, request, options));
    sendMessage(channel, message, listener);
}
-----------------------------------------------
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
这里有几个封装过程，涉及OutboundMessage.Request-->SendContext-->BytesReference。

### Netty4TcpChannel#sendMessage
最后调用channel.writeAndFlush通过Netty把消息发送到另一个Node。Netty4Utils.toByteBuf(reference)把 Reference转换为Netty上下文传输的ByteBuf类型。
```
public void sendMessage(BytesReference reference, ActionListener<Void> listener) {
    //Netty4Utils.toByteBuf(reference)把对BytrBuf（Netty）Byte的封装转换为BytrBuf（Netty）
    // addPromise(listener, channel),为Channel增加监听器
    channel.writeAndFlush(Netty4Utils.toByteBuf(reference), addPromise(listener, channel));

    if (channel.eventLoop().isShutdown()) {
        listener.onFailure(new TransportException("Cannot send message, event loop is shutting down."));
    }
}
```

### Netty4MessageChannelHandler#channelRead
当远程节点发来RPC请求，Netty传输对应的Handler是Netty4MessageChannelHandler，通过channelRead方法接收消息。Netty上下文传递的类型是自己封装的ByteBuf，和JDK的ByteBuffer相比有更好的性能。
```
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    assert Transports.assertTransportThread();
    assert msg instanceof ByteBuf : "Expected message type ByteBuf, found: " + msg.getClass();
    //Netty上下文传递
    final ByteBuf buffer = (ByteBuf) msg;
    try {
        Channel channel = ctx.channel();
        //AttributeKey.newInstance("es-channel")
        Attribute<Netty4TcpChannel> channelAttribute = channel.attr(Netty4Transport.CHANNEL_KEY);
        //调用TcpTransport.inboundMessage进行消息入站处理
        transport.inboundMessage(channelAttribute.get(), Netty4Utils.toBytesReference(buffer));
    } finally {
        buffer.release();
    }
}
```
接下来进入传输层，处理入站消息。Netty4Utils.toBytesReference(buffer)把Netty网络传输的数据转换为BytesReference类型，和Netty4Utils.toByteBuf(reference)相对应。

### Netty4Transport#inboundMessage
InboundHandler的初始化在TcpTransport的初始化中完成，在InboundHandler中完成逻辑的处理。
```
public void inboundMessage(TcpChannel channel, BytesReference message) {
    try {
        inboundHandler.inboundMessage(channel, message);
    } catch (Exception e) {
        onException(channel, e);
    }
}
-----------------------------------------------------------------------
//inboundHandler，处理请求的方法是inboundMessage
//client和server两部分都注册了这个handler，因为客户端和服务端都需要处理入方向的请求
void inboundMessage(TcpChannel channel, BytesReference message) throws Exception {
    //时间戳标记
    channel.getChannelStats().markAccessed(threadPool.relativeTimeInMillis());
    TransportLogger.logInboundMessage(channel, message);
    readBytesMetric.inc(message.length() + TcpHeader.MARKER_BYTES_SIZE + TcpHeader.MESSAGE_LENGTH_SIZE);
    // Message length of 0 is a ping
    if (message.length() != 0) {
        //入站处理请求
        messageReceived(message, channel);
    } else {
        //message长度为0表示是ping连接
        keepAlive.receiveKeepAlive(channel);
    }
}
-----------------------------------------------------------------------
private void messageReceived(BytesReference reference, TcpChannel channel) throws IOException {
    InetSocketAddress remoteAddress = channel.getRemoteAddress();

    ThreadContext threadContext = threadPool.getThreadContext();
    //Try-catch不用重新写一个finally块，这里会自动释放
    try (ThreadContext.StoredContext existing = threadContext.stashContext();
         //reference是对ByteBuf(Netty)数据的封装
         //把ByteBuf数据反序列化转换为InboundMessage
         InboundMessage message = reader.deserialize(reference)) {
        // Place the context with the headers from the message
        message.getStoredContext().restore();
        threadContext.putTransient("_remote_address", remoteAddress);
        //Netty4Transport 在Client 和Server 中共同使用了这个Netty4MessageChannelHandler 回调函数。
        // Client 在发送请求给远方的Node 的时候会把在信息的header里面标注为request 这个状态，
        // 所以这个回调函数判断到底是远方Node 发来的请求还是返回执行的结果是就是根据这个 TransportStatus.isRequest(status) 状态
        if (message.isRequest()) {
            //Node远程节点处理，Request请求
            handleRequest(channel, (InboundMessage.Request) message, reference.length());
        }
-----------------------------------------------------------------------
//接收请求并响应
private void handleRequest(TcpChannel channel, InboundMessage.Request message, int messageLengthBytes) {
    final Set<String> features = message.getFeatures();
    //action对应handler的名字
    final String action = message.getActionName();
    //请求唯一标识
    final long requestId = message.getRequestId();
    //真正的数据
    final StreamInput stream = message.getStreamInput();
    final Version version = message.getVersion();
    TransportChannel transportChannel = null;
    try {
        messageListener.onRequestReceived(requestId, action);
        if (message.isHandshake()) {
            //TCP三次握手请求处理
            //Handshake状态，调用TransportHandshaker的匿名类HandshakeResponseSender，直接通过Netty出站返回Response
            handshaker.handleHandshake(version, features, channel, requestId, stream);
        } else {
            //通过action获取RequestHandlerRegistry（包含Handler等信息）
            final RequestHandlerRegistry reg = getRequestHandler(action);
            if (reg == null) {
                throw new ActionNotFoundTransportException(action);
            }
            CircuitBreaker breaker = circuitBreakerService.getBreaker(CircuitBreaker.IN_FLIGHT_REQUESTS);
            if (reg.canTripCircuitBreaker()) {
                breaker.addEstimateBytesAndMaybeBreak(messageLengthBytes, "<transport_request>");
            } else {
                breaker.addWithoutBreaking(messageLengthBytes);
            }
            transportChannel = new TcpTransportChannel(outboundHandler, channel, action, requestId, version, features,
                circuitBreakerService, messageLengthBytes, message.isCompress());
            final TransportRequest request = reg.newRequest(stream);
            request.remoteAddress(new TransportAddress(channel.getRemoteAddress()));
            // in case we throw an exception, i.e. when the limit is hit, we don't want to verify
            final int nextByte = stream.read();
            .........................................
            threadPool.executor(reg.getExecutor()).execute(new RequestHandler(reg, request, transportChannel));
        }
```
getRequestHandler(action)根据action获取对应的Handler，最终由线程池执行对应的任务，线程池是Handler注册时指定的。

### RequestHandler
taskManager.register把本地的任务暂时存储，handle是根据getRequestHandler(action)获取的，然后根据不同的业务逻辑处理不同的任务。
```
protected void doRun() throws Exception {
    reg.processMessageReceived(request, transportChannel);
}
-----------------------------------------------------------------------
public void processMessageReceived(Request request, TransportChannel channel) throws Exception {
    //本地Node任务暂存
    final Task task = taskManager.register(channel.getChannelType(), action, request);
    boolean success = false;
    try {
        handler.messageReceived(request, new TaskTransportChannel(taskManager, task, channel), task);
        success = true;
    } finally {
        if (success == false) {
            taskManager.unregister(task);
        }
    }
}
```