---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---

### Watcher源码分析

## 客户端源码

Zookeeper 客户端主要有以下几个重要的组件。客户端会话创建可以分为三个阶段：一是初始化阶段、二是会话创建阶段、三是响应处理阶段。

| 类                    | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| Zookeeper             | Zookeeper 客户端入口                                         |
| ClientWatchManager    | 客户端 Watcher 管理器                                        |
| ClientCnxn            | 客户端核心线程，其内部又包含两个线程，即 SendThread 和 EventThread。前者是一个 IO 线程，主要负责 ZooKeeper 客户端和服务端之间的网络通信；后者是一个事件线程，主要负责对服务端事件进行处理。 |
| HostProvider          | 客户端地址列表管理器                                         |
| ClientCnxnSocketNetty | 最底层的通信 netty                                           |

客户端整体结构如下图：![Zookeeper客户端类图](png\zookeeper\Zookeeper客户端类图.png)

客户端在构造阶段创建 ClientCnxn 与服务端连接，后续命令都是通过 ClientCnxn 发送给服务端。ClientCnxn 是客户端与服务端通信的底层接口，它和 ClientCnxnSocketNetty 一起工作提供网络通信服务。服务端是 ZookeeperServer 类，收到 ClientCnxn 的请求处理后再通过 ClientCnxn 返回到客户端。

ClientCnxn 连接时可以同时指定多台服务器地址，根据指定的算法连接一台服务器，当某个服务器发生故障无法连接时，会自动连接到其他的服务器。实现这一机制的是 StaticHostProvider 类。

#### 创建阶段

**(1) ZooKeeper 创建** 【ZooKeeper】

```java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
        long sessionId, byte[] sessionPasswd, boolean canBeReadOnly,
        HostProvider aHostProvider) throws IOException {
    
    // 1. watcher 保存在 ZKWatchManager 的 defaultWatcher 中，作为整个会话的默认 watcher
    watchManager = defaultWatchManager();
    watchManager.defaultWatcher = watcher;
   
    // 2. 解析 server 获取 IP 以及 PORT
    ConnectStringParser connectStringParser = new ConnectStringParser(
            connectString);
    hostProvider = aHostProvider;

    // 3. 创建 ClientCnxn 实例
    cnxn = new ClientCnxn(connectStringParser.getChrootPath(),
            hostProvider, sessionTimeout, this, watchManager,
            getClientCnxnSocket(), sessionId, sessionPasswd, canBeReadOnly);
    cnxn.seenRwServerBefore = true; // since user has provided sessionId
    // 4. 启动 SendThread 和 EventThread 线程，这两个线程均为守护线程
    cnxn.start();
}
```

创建底层通信 ClientCnxnSocketNIO 或 ClientCnxnSocketNetty

```java
public static final String ZOOKEEPER_CLIENT_CNXN_SOCKET = "zookeeper.clientCnxnSocket";
private static ClientCnxnSocket getClientCnxnSocket() throws IOException {
    String clientCnxnSocketName = System
            .getProperty(ZOOKEEPER_CLIENT_CNXN_SOCKET);
    if (clientCnxnSocketName == null) {
        clientCnxnSocketName = ClientCnxnSocketNIO.class.getName();
    }
    try {
        return (ClientCnxnSocket) Class.forName(clientCnxnSocketName)
                .newInstance();
    } catch (Exception e) {
        IOException ioe = new IOException("Couldn't instantiate "
                + clientCnxnSocketName);
        ioe.initCause(e);
        throw ioe;
    }
}
```

**(2) ClientCnxn 创建** 【ClientCnxn】

| 属性          | 说明                        |
| ------------- | --------------------------- |
| Packet        | 所有的请求都会封装成 packet |
| SendThread    | 请求处理                    |
| EventThread   | 事件处理                    |
| outgoingQueue | 即将发送的请求 packets      |
| pendingQueue  | 已经发送等待响应的 packets  |

ClientCnxn 创建时创建了两个线程 SendThread 和 EventThread，这两个线程都是守护线程，主线程结束时即关闭线程。

```java
// 已经发送等待响应的 packets
private final LinkedList<Packet> pendingQueue = new LinkedList<Packet>();
// 即将发送的请求 packets
private final LinkedBlockingDeque<Packet> outgoingQueue = new LinkedBlockingDeque<Packet>();

public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
        ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket,
        long sessionId, byte[] sessionPasswd, boolean canBeReadOnly) {
    this.zooKeeper = zooKeeper;
    this.watcher = watcher;
    this.sessionId = sessionId;
    this.sessionPasswd = sessionPasswd;
    this.sessionTimeout = sessionTimeout;
    this.hostProvider = hostProvider;
    this.chrootPath = chrootPath;

    connectTimeout = sessionTimeout / hostProvider.size();
    readTimeout = sessionTimeout * 2 / 3;
    readOnly = canBeReadOnly;

    sendThread = new SendThread(clientCnxnSocket);
    eventThread = new EventThread();
}

SendThread(ClientCnxnSocket clientCnxnSocket) {
    super(makeThreadName("-SendThread()"));
    state = States.CONNECTING;
    this.clientCnxnSocket = clientCnxnSocket;
    setDaemon(true);
}

EventThread() {
    super(makeThreadName("-EventThread"));
    setDaemon(true);
}
```

#### 会话创建

**(3) 启动两个守护线程** 【ClientCnxn】

```java
public void start() {
    sendThread.start();
    eventThread.start();
}
```

**(4) 开始创建连接** 【SendThread】

SendThread 的 run 方法启动时会调用 startConnect() 先创建连接。

```java
if (!clientCnxnSocket.isConnected()) {
    // don't re-establish connection if we are closing
    if (closing) {
        break;
    }
    startConnect();
    clientCnxnSocket.updateLastSendAndHeard();
}
```

startConnect() 会通过 hostProvider 获取 zookeeper 服务端地址，然后调用底层的 clientCnxnSocket 来创建一个连接。

```java
private void startConnect() throws IOException {
    if(!isFirstConnect){
        try {
            Thread.sleep(r.nextInt(1000));
        } catch (InterruptedException e) {
            LOG.warn("Unexpected exception", e);
        }
    }
    state = States.CONNECTING;

    InetSocketAddress addr;
    if (rwServerAddress != null) {
        addr = rwServerAddress;
        rwServerAddress = null;
    } else {
        addr = hostProvider.next(1000);
    }

    setName(getName().replaceAll("\\(.*\\)",
            "(" + addr.getHostString() + ":" + addr.getPort() + ")"));
    // ... 省略

    logStartConnect(addr);
    clientCnxnSocket.connect(addr);
}
```

**(5) 建立 TCP 连接** 【ClientCnxnSocketNetty】

ClientCnxnSocket 有两个实现类 ClientCnxnSocketNetty 和 ClientCnxnSocketNIO，负责底层的通信，以 ClientCnxnSocketNetty 为例通过 connect 方法创建了一个 TCP 连接。

```java
@Override
void connect(InetSocketAddress addr) throws IOException {
    firstConnect = new CountDownLatch(1);

    ClientBootstrap bootstrap = new ClientBootstrap(channelFactory);

    bootstrap.setPipelineFactory(new ZKClientPipelineFactory());
    bootstrap.setOption("soLinger", -1);
    bootstrap.setOption("tcpNoDelay", true);

    connectFuture = bootstrap.connect(addr);
    connectFuture.addListener(new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture channelFuture) throws Exception {
            // this lock guarantees that channel won't be assgined after cleanup().
            connectLock.lock();
            try {
                if (!channelFuture.isSuccess() || connectFuture == null) {
                    LOG.info("future isn't success, cause: {}", channelFuture.getCause());
                    return;
                }
                // 1. 建立 tcp 连接
                channel = channelFuture.getChannel();

                // 2. 初始化参数
                disconnected.set(false);
                initialized = false;
                lenBuffer.clear();
                incomingBuffer = lenBuffer;

                // 3. 发送 ConnectRequest 请求，会话创建完成
                sendThread.primeConnection();
                updateNow();
                updateLastSendAndHeard();

                if (sendThread.tunnelAuthInProgress()) {
                    waitSasl.drainPermits();
                    needSasl.set(true);
                    sendPrimePacket();
                } else {
                    needSasl.set(false);
                }

                // we need to wake up on first connect to avoid timeout.
                wakeupCnxn();
                firstConnect.countDown();
                LOG.info("channel is connected: {}", channelFuture.getChannel());
            } finally {
                connectLock.unlock();
            }
        }
    });
}
```

**(6) 发送 ConnectRequest 请求创建会话** 【SendThread】

```java
void primeConnection() throws IOException {
    LOG.info("Socket connection established, initiating session, client: {}, server: {}",
            clientCnxnSocket.getLocalSocketAddress(),
            clientCnxnSocket.getRemoteSocketAddress());
    isFirstConnect = false;
    long sessId = (seenRwServerBefore) ? sessionId : 0;
    ConnectRequest conReq = new ConnectRequest(0, lastZxid,
            sessionTimeout, sessId, sessionPasswd);
    
    // 省略... 主要是对事件的处理

    // 1. 发送连接请求
    outgoingQueue.addFirst(new Packet(null, null, conReq,
            null, null, readOnly));
    clientCnxnSocket.connectionPrimed();
    if (LOG.isDebugEnabled()) {
        LOG.debug("Session establishment request sent on "
                + clientCnxnSocket.getRemoteSocketAddress());
    }
}
```

至此，一个完整的会话就创建完成。但还有个疑问，ConnectRequest 请求放到了 outgoingQueue 中到底是如何发送的，客户端又是如何处理请求的呢？

#### 发送请求

**(7) 发送请求** 【SendThread】

```java
@Override
public void run() {
    // 1. 初始化设置，如：心跳时间、上次发送请求时间、当前时间
    clientCnxnSocket.introduce(this, sessionId, outgoingQueue);
    clientCnxnSocket.updateNow();
    clientCnxnSocket.updateLastSendAndHeard();
    int to;
    long lastPingRwServer = Time.currentElapsedTime();
    final int MAX_SEND_PING_INTERVAL = 10000; //10 seconds
    while (state.isAlive()) {
        try {
            // 2. 建立会话连接
            if (!clientCnxnSocket.isConnected()) {
                // don't re-establish connection if we are closing
                if (closing) {
                    break;
                }
                startConnect();
                clientCnxnSocket.updateLastSendAndHeard();
            }

            // 3. 设置超时时间
            if (state.isConnected()) {
                // 省略...
                to = readTimeout - clientCnxnSocket.getIdleRecv();
            } else {
                to = connectTimeout - clientCnxnSocket.getIdleRecv();
            }
            
            if (to <= 0) {
                throw new SessionTimeoutException(
                        "Client session timed out, have not heard from server in "
                                + clientCnxnSocket.getIdleRecv() + "ms"
                                + " for sessionid 0x"
                                + Long.toHexString(sessionId));
            }
            if (state.isConnected()) {
                //1000(1 second) is to prevent race condition missing to send the second ping
                //also make sure not to send too many pings when readTimeout is small 
                int timeToNextPing = readTimeout / 2 - clientCnxnSocket.getIdleSend() - 
                        ((clientCnxnSocket.getIdleSend() > 1000) ? 1000 : 0);
                //send a ping request either time is due or no packet sent out within MAX_SEND_PING_INTERVAL
                if (timeToNextPing <= 0 || clientCnxnSocket.getIdleSend() > MAX_SEND_PING_INTERVAL) {
                    sendPing();
                    clientCnxnSocket.updateLastSend();
                } else {
                    if (timeToNextPing < to) {
                        to = timeToNextPing;
                    }
                }
            }

            // 省略...
            // 5. 处理请求，由底层的 ClientCnxnSocket 完成
            clientCnxnSocket.doTransport(to, pendingQueue, ClientCnxn.this);
        } catch (Throwable e) {
            // 6. closing=true 直接会话，跳出 while 循环
            if (closing) {
                break;
            } else {
                // 7. 清空 outgoingQueue 和 pendingQueue，等待一次 while 重新建立连接
                cleanup();
                if (state.isAlive()) {
                    eventThread.queueEvent(new WatchedEvent(
                            Event.EventType.None,
                            Event.KeeperState.Disconnected,
                            null));
                }
                clientCnxnSocket.updateNow();
                clientCnxnSocket.updateLastSendAndHeard();
            }
        }
    }

    // 8. 在第 6 步跳出 while 后会关闭连接，清空 outgoingQueue 和 pendingQueue
    synchronized (state) {
        cleanup();
    }
    clientCnxnSocket.close();
    if (state.isAlive()) {
        eventThread.queueEvent(new WatchedEvent(Event.EventType.None,
                Event.KeeperState.Disconnected, null));
    }
}
```

我们可以看到 while 循环中最终调用 clientCnxnSocket.doTransport(to, pendingQueue, ClientCnxn.this) 处理请求，以 ClientCnxnSocketNetty 为例。

**(8) 处理请求** 【ClientCnxnSocketNetty】

```java
@Override
void doTransport(int waitTimeOut,
                 List<Packet> pendingQueue,
                 ClientCnxn cnxn)
        throws IOException, InterruptedException {
    try {
        if (!firstConnect.await(waitTimeOut, TimeUnit.MILLISECONDS)) {
            return;
        }
        Packet head = null;
        if (needSasl.get()) {
            if (!waitSasl.tryAcquire(waitTimeOut, TimeUnit.MILLISECONDS)) {
                return;
            }
        } else {
            if ((head = outgoingQueue.poll(waitTimeOut, TimeUnit.MILLISECONDS)) == null) {
                return;
            }
        }
        // check if being waken up on closing.
        if (!sendThread.getZkState().isAlive()) {
            // adding back the patck to notify of failure in conLossPacket().
            addBack(head);
            return;
        }
        // channel disconnection happened
        if (disconnected.get()) {
            addBack(head);
            throw new EndOfStreamException("channel for sessionid 0x"
                    + Long.toHexString(sessionId)
                    + " is lost");
        }
        if (head != null) {
            doWrite(pendingQueue, head, cnxn);
        }
    } finally {
        updateNow();
    }
}
```

doTransport 中将处理请求的委托给了 doWrite 完成，在 doWrite 中会遍历所有的 outgoingQueue 发送请求。

```java
private void doWrite(List<Packet> pendingQueue, Packet p, ClientCnxn cnxn) {
    updateNow();
    while (true) {
        if (p != WakeupPacket.getInstance()) {
            if ((p.requestHeader != null) &&
                    (p.requestHeader.getType() != ZooDefs.OpCode.ping) &&
                    (p.requestHeader.getType() != ZooDefs.OpCode.auth)) {
                p.requestHeader.setXid(cnxn.getXid());
                synchronized (pendingQueue) {
                    pendingQueue.add(p);
                }
            }
            sendPkt(p);
        }
        if (outgoingQueue.isEmpty()) {
            break;
        }
        p = outgoingQueue.remove();
    }
}

private void sendPkt(Packet p) {
    // Assuming the packet will be sent out successfully. Because if it fails,
    // the channel will close and clean up queues.
    p.createBB();
    updateLastSend();
    sentCount++;
    channel.write(ChannelBuffers.wrappedBuffer(p.bb));
}
```

#### 响应阶段

**(9) 接收响应数据，处理业务** 【ZKClientHandler】

```java
@Override
public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
    updateNow();
    ChannelBuffer buf = (ChannelBuffer) e.getMessage();
    while (buf.readable()) {
        // incomingBuffer 的长度大小 buf 时会抛出异常
        if (incomingBuffer.remaining() > buf.readableBytes()) {
            int newLimit = incomingBuffer.position()
                    + buf.readableBytes();
            incomingBuffer.limit(newLimit);
        }
        buf.readBytes(incomingBuffer);
        incomingBuffer.limit(incomingBuffer.capacity());

        if (!incomingBuffer.hasRemaining()) {
            incomingBuffer.flip();
            if (incomingBuffer == lenBuffer) {
                // 1. 读取长度
                recvCount++;
                readLength();
            } else if (!initialized) {
                // 2. 连接信息
                readConnectResult();
                lenBuffer.clear();
                incomingBuffer = lenBuffer;
                initialized = true;
                updateLastHeard();
            } else {
                // 3. 请求信息
                sendThread.readResponse(incomingBuffer);
                lenBuffer.clear();
                incomingBuffer = lenBuffer;
                updateLastHeard();
            }
        }
    }
    wakeupCnxn();
}
```

重点关注一下 sendThread.readResponse(incomingBuffer) 是如何处理响应的。

**(10) 数据处理** 【SendThread】

```java
void readResponse(ByteBuffer incomingBuffer) throws IOException {
    ByteBufferInputStream bbis = new ByteBufferInputStream(
            incomingBuffer);
    BinaryInputArchive bbia = BinaryInputArchive.getArchive(bbis);
    ReplyHeader replyHdr = new ReplyHeader();

    replyHdr.deserialize(bbia, "header");
    if (replyHdr.getXid() == -2) {
        // 1. -2 pings
        return;
    }
    if (replyHdr.getXid() == -4) {
        // 2. -4 认证             
        if(replyHdr.getErr() == KeeperException.Code.AUTHFAILED.intValue()) {
            state = States.AUTH_FAILED;                    
            eventThread.queueEvent( new WatchedEvent(Watcher.Event.EventType.None, 
                    Watcher.Event.KeeperState.AuthFailed, null) );                                      
        }
        return;
    }
    if (replyHdr.getXid() == -1) {
        // 3. -1 事件
        WatcherEvent event = new WatcherEvent();
        event.deserialize(bbia, "response");

        // convert from a server path to a client path
        if (chrootPath != null) {
            String serverPath = event.getPath();
            if(serverPath.compareTo(chrootPath)==0)
                event.setPath("/");
            else if (serverPath.length() > chrootPath.length())
                event.setPath(serverPath.substring(chrootPath.length()));
            else {
                LOG.warn("Got server path " + event.getPath()
                        + " which is too short for chroot path "
                        + chrootPath);
            }
        }

        WatchedEvent we = new WatchedEvent(event);
        if (LOG.isDebugEnabled()) {
            LOG.debug("Got " + we + " for sessionid 0x"
                    + Long.toHexString(sessionId));
        }

        eventThread.queueEvent( we );
        return;
    }

    // 省略...
    // 4. 处理请求响应
    Packet packet;
    synchronized (pendingQueue) {
        if (pendingQueue.size() == 0) {
            throw new IOException("Nothing in the queue, but got "
                    + replyHdr.getXid());
        }
        packet = pendingQueue.remove();
    }
    
    try {
        // 4.1 服务端一定是顺序处理请求的，但最好验证一下请求和响应的 xid 是否一致
        if (packet.requestHeader.getXid() != replyHdr.getXid()) {
            packet.replyHeader.setErr(
                    KeeperException.Code.CONNECTIONLOSS.intValue());
            throw new IOException("Xid out of order. Got Xid "
                    + replyHdr.getXid() + " with err " +
                    + replyHdr.getErr() +
                    " expected Xid "
                    + packet.requestHeader.getXid()
                    + " for a packet with details: "
                    + packet );
        }

        packet.replyHeader.setXid(replyHdr.getXid());
        packet.replyHeader.setErr(replyHdr.getErr());
        packet.replyHeader.setZxid(replyHdr.getZxid());
        if (replyHdr.getZxid() > 0) {
            lastZxid = replyHdr.getZxid();
        }
        if (packet.response != null && replyHdr.getErr() == 0) {
            packet.response.deserialize(bbia, "response");
        }
    } finally {
        // 4.2 此至 packet 封装了请求和响应的消息，还需要根据业务进行具体的处理。如：同步 
        finishPacket(packet);
    }
}
```

#### 请求处理的全过程

**(1) create** 【Zookeeper】

```java
public String create(final String path, byte data[], List<ACL> acl,CreateMode createMode)
    throws KeeperException, InterruptedException {
    final String clientPath = path;
    PathUtils.validatePath(clientPath, createMode.isSequential());

    final String serverPath = prependChroot(clientPath);

    RequestHeader h = new RequestHeader();
    h.setType(createMode.isContainer() ? ZooDefs.OpCode.createContainer : ZooDefs.OpCode.create);
    CreateRequest request = new CreateRequest();
    CreateResponse response = new CreateResponse();
    request.setData(data);
    request.setFlags(createMode.toFlag());
    request.setPath(serverPath);
    if (acl != null && acl.size() == 0) {
        throw new KeeperException.InvalidACLException();
    }
    request.setAcl(acl);
    // 请求委托 cnxn 完成
    ReplyHeader r = cnxn.submitRequest(h, request, response, null);
    if (r.getErr() != 0) {
        throw KeeperException.create(KeeperException.Code.get(r.getErr()),
                clientPath);
    }
    if (cnxn.chrootPath == null) {
        return response.getPath();
    } else {
        return response.getPath().substring(cnxn.chrootPath.length());
    }
}
```

**(2) submitRequest** 【ClientCnxn】

```java
public ReplyHeader submitRequest(RequestHeader h, Record request,
        Record response, WatchRegistration watchRegistration)
        throws InterruptedException {
    return submitRequest(h, request, response, watchRegistration, null);
}

// 同步等待请求处理完成
public ReplyHeader submitRequest(RequestHeader h, Record request,
        Record response, WatchRegistration watchRegistration,
        WatchDeregistration watchDeregistration)
        throws InterruptedException {
    ReplyHeader r = new ReplyHeader();
    Packet packet = queuePacket(h, r, request, response, null, null, null,
            null, watchRegistration, watchDeregistration);
    synchronized (packet) {
        while (!packet.finished) {
            packet.wait();
        }
    }
    return r;
}
```

**(3) queuePacket** 【ClientCnxn】

```java
Packet queuePacket(RequestHeader h, ReplyHeader r, Record request,
        Record response, AsyncCallback cb, String clientPath,
        String serverPath, Object ctx, WatchRegistration watchRegistration,
        WatchDeregistration watchDeregistration) {
    Packet packet = null;

    // 封装成 Packet
    packet = new Packet(h, r, request, response, watchRegistration);
    packet.cb = cb;
    packet.ctx = ctx;
    packet.clientPath = clientPath;
    packet.serverPath = serverPath;
    packet.watchDeregistration = watchDeregistration;
    // The synchronized block here is for two purpose:
    // 1. synchronize with the final cleanup() in SendThread.run() to avoid race
    // 2. synchronized against each packet. So if a closeSession packet is added,
    // later packet will be notified.
    synchronized (state) {
        if (!state.isAlive() || closing) {
            conLossPacket(packet);
        } else {
            if (h.getType() == OpCode.closeSession) {
                closing = true;
            }
            // 加入到 outgoingQueue 等待发送请求
            outgoingQueue.add(packet);
        }
    }
    sendThread.getClientCnxnSocket().packetAdded();
    return packet;
}
```

**(4) finishPacket** 【ClientCnxn】

```java
private void finishPacket(Packet p) {
    int err = p.replyHeader.getErr();
    if (p.watchRegistration != null) {
        p.watchRegistration.register(err);
    }
    // 处理事件
    if (p.watchDeregistration != null) {
        Map<EventType, Set<Watcher>> materializedWatchers = null;
        try {
            materializedWatchers = p.watchDeregistration.unregister(err);
            for (Entry<EventType, Set<Watcher>> entry : materializedWatchers
                    .entrySet()) {
                Set<Watcher> watchers = entry.getValue();
                if (watchers.size() > 0) {
                    queueEvent(p.watchDeregistration.getClientPath(), err,
                            watchers, entry.getKey());
                    p.replyHeader.setErr(Code.OK.intValue());
                }
            }
        } catch (KeeperException.NoWatcherException nwe) {
            LOG.error("Failed to find watcher!", nwe);
            p.replyHeader.setErr(nwe.code().intValue());
        } catch (KeeperException ke) {
            LOG.error("Exception when removing watcher", ke);
            p.replyHeader.setErr(ke.code().intValue());
        }
    }
    
    // 响应请求内容 notifyAll
    if (p.cb == null) {
        synchronized (p) {
            p.finished = true;
            p.notifyAll();
        }
    } else {
        p.finished = true;
        eventThread.queuePacket(p);
    }
}
```

**(5) cleanup** 【ClientCnxn】

注意到另一种情况，当会话断开时有可能出现死锁的情况，如何解决这个问题呢？

```java
// 会话失效时清空所有的 pendingQueue 和 outgoingQueue
private void cleanup() {
    clientCnxnSocket.cleanup();
    synchronized (pendingQueue) {
        for (Packet p : pendingQueue) {
            conLossPacket(p);
        }
        pendingQueue.clear();
    }
    // We can't call outgoingQueue.clear() here because
    // between iterating and clear up there might be new
    // packets added in queuePacket().
    Iterator<Packet> iter = outgoingQueue.iterator();
    while (iter.hasNext()) {
        Packet p = iter.next();
        conLossPacket(p);
        iter.remove();
    }
}

// 手动结束请求
private void conLossPacket(Packet p) {
    if (p.replyHeader == null) {
        return;
    }
    switch (state) {
    case AUTH_FAILED:
        p.replyHeader.setErr(KeeperException.Code.AUTHFAILED.intValue());
        break;
    case CLOSED:
        p.replyHeader.setErr(KeeperException.Code.SESSIONEXPIRED.intValue());
        break;
    default:
        p.replyHeader.setErr(KeeperException.Code.CONNECTIONLOSS.intValue());
    }
    finishPacket(p);
}
```