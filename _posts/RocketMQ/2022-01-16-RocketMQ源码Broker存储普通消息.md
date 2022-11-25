---
layout: post
categories: RocketMQ
description: none
keywords: RocketMQ
---
# RocketMQ源码-Broker存储普通消息

Broker接收到消息请求以后，是如何将消息存储起来的？

Broker服务器接收到消息请求时，通过请求码找到SendMessageProcessor处理器，SendMessageProcessor处理器通过processRequest方法处理消息请求：

```java
//代码位置：org.apache.rocketmq.broker.processor.SendMessageProcessor#processRequest
public RemotingCommand processRequest(ChannelHandlerContext ctx,
                                          RemotingCommand request) throws RemotingCommandException {
        RemotingCommand response = null;
        try {
            response = asyncProcessRequest(ctx, request).get();
        } catch (InterruptedException | ExecutionException e) {
            log.error("process SendMessage error, request : " + request.toString(), e);
        }
        return response;
}
```

processRequest方法调用asyncProcessRequest方法处理请求，asyncProcessRequest方法是异步方法，通过get方法阻塞获取到最后的结果。asyncProcessRequest的代码如下：

```java
//代码位置：org.apache.rocketmq.broker.processor.SendMessageProcessor#asyncProcessRequest
public CompletableFuture<RemotingCommand> asyncProcessRequest(ChannelHandlerContext ctx,
                                                                  RemotingCommand request) throws RemotingCommandException {
        final SendMessageContext mqtraceContext;
        switch (request.getCode()) {
            //消费者发送返回消息
            case RequestCode.CONSUMER_SEND_MSG_BACK:
                return this.asyncConsumerSendMsgBack(ctx, request);
            default:
                //解析发送消息请求头
                SendMessageRequestHeader requestHeader = parseRequestHeader(request);
                if (requestHeader == null) {
                    return CompletableFuture.completedFuture(null);
                }
                //发送消息实体
                mqtraceContext = buildMsgContext(ctx, requestHeader);
                this.executeSendMessageHookBefore(ctx, request, mqtraceContext);
                //是否批量发送
                if (requestHeader.isBatch()) {
                    return this.asyncSendBatchMessage(ctx, request, mqtraceContext, requestHeader);
                } else {
                    return this.asyncSendMessage(ctx, request, mqtraceContext, requestHeader);
                }
        }
}
```

asyncProcessRequest方法逻辑比较简单，通过请求码选择不同的消息请求逻辑，当请求码是CONSUMER_SEND_MSG_BACK时，处理消费者发送返回的消息，默认处理生产者发送的消息请求，首先解析消息发送请求头，如果消息请求头为null，就直接返回处理结果了，然后创建消息内容实体，最后根据消是否是批量。调用不同方法处理消息请求。当是批量消息发送时，调用asyncSendBatchMessage方法处理批量消息，否则调用asyncSendMessage方法处理。

我们解析来看看parseRequestHeader方法，看方法的名字就知道是解析消息请求头，parseRequestHeader方法如下：

```java
//代码位置：org.apache.rocketmq.broker.processor.SendMessageProcessor#parseRequestHeader
protected SendMessageRequestHeader parseRequestHeader(RemotingCommand request)
        throws RemotingCommandException {

        //发送消息请求头，版本2
        SendMessageRequestHeaderV2 requestHeaderV2 = null;
        //发送消息请求头，版本1
        SendMessageRequestHeader requestHeader = null;
        switch (request.getCode()) {
                //批量消息，发送消息版本2
            case RequestCode.SEND_BATCH_MESSAGE:
            case RequestCode.SEND_MESSAGE_V2:
                requestHeaderV2 =
                    (SendMessageRequestHeaderV2) request
                        .decodeCommandCustomHeader(SendMessageRequestHeaderV2.class);
                //普通消息发送
            case RequestCode.SEND_MESSAGE:
                if (null == requestHeaderV2) {
                    requestHeader =
                        (SendMessageRequestHeader) request
                            .decodeCommandCustomHeader(SendMessageRequestHeader.class);
                } else {
                    requestHeader = SendMessageRequestHeaderV2.createSendMessageRequestHeaderV1(requestHeaderV2);
                }
            default:
                break;
        }
        return requestHeader;
}
```

RocketMQ为了发送消息的性能，除了会对消息进行压缩外，还会将消息转换为更轻量的消息进行发送，更轻量的消息就是将原来的消息换成不利于阅读的格式，这样消息就会变得更小，传输起来就更快。

parseRequestHeader方法通过请求码的类型解析消息发送请求头，如果是批量消息和更轻量的消息，就解析为SendMessageRequestHeaderV2，否则就解析为SendMessageRequestHeader。

接下来，我们将会从下面两个方面继续分析消除的存储：

- 单笔消息存储处理
- 批量消息存储处理

### 单笔消息存储处理

```java
//代码位置：org.apache.rocketmq.broker.processor.SendMessageProcessor#asyncSendMessage
private CompletableFuture<RemotingCommand> asyncSendMessage(ChannelHandlerContext ctx, RemotingCommand request,
                                                                SendMessageContext mqtraceContext,
                                                                SendMessageRequestHeader requestHeader) {

        //创建响应结果
        final RemotingCommand response = preSend(ctx, request, requestHeader);
        //发送消息响应头
        final SendMessageResponseHeader responseHeader = (SendMessageResponseHeader)response.readCustomHeader();

        if (response.getCode() != -1) {
            return CompletableFuture.completedFuture(response);
        }

        //消息体
        final byte[] body = request.getBody();

        //队列id
        int queueIdInt = requestHeader.getQueueId();

        //根据topic获取topic配置
        TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());

        //队列小于0，随机给一个
        if (queueIdInt < 0) {
            queueIdInt = randomQueueId(topicConfig.getWriteQueueNums());
        }

        //消息
        MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
        //省略代码：为msgInner赋值

        CompletableFuture<PutMessageResult> putMessageResult = null;
        Map<String, String> origProps = MessageDecoder.string2messageProperties(requestHeader.getProperties());
        String transFlag = origProps.get(MessageConst.PROPERTY_TRANSACTION_PREPARED);
        if (transFlag != null && Boolean.parseBoolean(transFlag)) {
            //如果拒绝事务消息
            if (this.brokerController.getBrokerConfig().isRejectTransactionMessage()) {
                response.setCode(ResponseCode.NO_PERMISSION);
                response.setRemark(
                        "the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1()
                                + "] sending transaction message is forbidden");
                return CompletableFuture.completedFuture(response);
            }
            //处理事务消息
            putMessageResult = this.brokerController.getTransactionalMessageService().asyncPrepareMessage(msgInner);
        } else {
            //处理普通消息
            putMessageResult = this.brokerController.getMessageStore().asyncPutMessage(msgInner);
        }
        return handlePutMessageResultFuture(putMessageResult, response, request, msgInner, responseHeader, mqtraceContext, ctx, queueIdInt);
}
```

asyncSendMessage方法先创建发送消息响应头，从发送消息请求头获取消息内容，创建msgInner消息并设置消息的属性。如果消息是事务消息，则进行事务消息的处理，否则，进行普通消息的处理。接下来，我们先分析下普通消息的处理，然后在分析事务消息的处理。

### 处理普通消息

```java
//代码位置：org.apache.rocketmq.store.DefaultMessageStore#putMessage
public PutMessageResult putMessage(MessageExtBrokerInner msg) {
        //检查store状态
        PutMessageStatus checkStoreStatus = this.checkStoreStatus();
        if (checkStoreStatus != PutMessageStatus.PUT_OK) {
            return new PutMessageResult(checkStoreStatus, null);
        }

        //消息是否合法
        PutMessageStatus msgCheckStatus = this.checkMessage(msg);
        if (msgCheckStatus == PutMessageStatus.MESSAGE_ILLEGAL) {
            return new PutMessageResult(msgCheckStatus, null);
        }

        long beginTime = this.getSystemClock().now();
        //保存消息
        PutMessageResult result = this.commitLog.putMessage(msg);
        long elapsedTime = this.getSystemClock().now() - beginTime;
        if (elapsedTime > 500) {
            log.warn("not in lock elapsed time(ms)={}, bodyLength={}", elapsedTime, msg.getBody().length);
        }

        //统计保存消息花费的时间
        this.storeStatsService.setPutMessageEntireTimeMax(elapsedTime);

        if (null == result || !result.isOk()) {
            //统计保存消息失败次数
            this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
        }

        return result;
}
```

采用DefaultMessageStore类的putMessage方法进行普通消息的存储。checkStoreStatus方法先检查store服务器的状态，校验服务器的状态是否已经关闭、Broker的角色是否是slave，是否不可写以及是否繁忙？如果满足上述其中之一的条件，那么说明store服务器的状态都不是正常的。校验完store服务器的状态以后，校验消息是否正常，checkMessage方法主要是判断topic的长度不能超过topic最大的长度127、消息的Properties属性不能超过Properties最大的长度32767。消息正常以后就可以调用commitLog的putMessage方法保存消息了，最后统计保存消息花费的时间和统计保存消息失败次数，接下来深入commitLog的putMessage的方法，看看是如何保存消息的。

```java
//代码位置：org.apache.rocketmq.store.CommitLog#putMessage
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {

        //省略代码

        long elapsedTimeInLock = 0;

        MappedFile unlockMappedFile = null;
        //获取最后一个MappedFile
        MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();

        //可重入锁
        putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
        try {
            long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
            this.beginTimeInLock = beginLockTimestamp;

            // Here settings are stored timestamp, in order to ensure an orderly
            // global
            msg.setStoreTimestamp(beginLockTimestamp);

            if (null == mappedFile || mappedFile.isFull()) {
                mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
            }
            if (null == mappedFile) {
                log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
            }

            //保存消息
            result = mappedFile.appendMessage(msg, this.appendMessageCallback);
            //根据保存消息的状态，做不同的处理
            switch (result.getStatus()) {
                case PUT_OK:
                    break;
                //如果mappedFile文件已经写满了，那么就在创建一个新的，继续写
                case END_OF_FILE:
                    unlockMappedFile = mappedFile;
                    // Create a new file, re-write the message
                    mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                    if (null == mappedFile) {
                        // XXX: warn and notify me
                        log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                        beginTimeInLock = 0;
                        return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
                    }
                    result = mappedFile.appendMessage(msg, this.appendMessageCallback);
                    break;
                case MESSAGE_SIZE_EXCEEDED:
                case PROPERTIES_SIZE_EXCEEDED:
                    beginTimeInLock = 0;
                    return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);
                case UNKNOWN_ERROR:
                    beginTimeInLock = 0;
                    return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
                default:
                    beginTimeInLock = 0;
                    return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
            }

            elapsedTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
            beginTimeInLock = 0;
        } finally {
            putMessageLock.unlock();
        }

        //省略代码

        return putMessageResult;
}
```

上述代码省略了一些对此次分析不太重要的代码，不影响此次的分析。首先获取最后一个MappedFile文件，MappedFile文件是用来保存消息的。为了避免消息在写入的过程中被其他线程访问，用可重入锁进行上锁，当获取MappedFile文件不存在或者已经写满了，就新创建一个MappedFile文件。然后通过MappedFile的appendMessage方法将消息保存起来得到保存消息的结果，根据保存消息状态设置保存消息的结果，这里有一个状态需要注意下，**当状态时END_OF_FILE时，说明MappedFile文件已经写满了，所以新建MappedFile文件，再通过MappedFile的appendMessage的方法保存消息。**

我们先不分析MappedFile的appendMessage的方法，先分析下CommitLog和MappedFile是什么？

CommitLog可以看成是Broker服务器在逻辑上对应的一个大文件，一个Broker服务器对应一个CommitLog文件，Broker服务接收到的所有消息都通过CommitLog类进行存储，CommitLog类有一个MappedFileQueue，MappedFileQueue队列存储着多个MappedFile文件，每个MappedFile文件的默认大小为1G ，**文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。**（这句话从官方文档copy的）。MappedFile类就是实际存储消息的对应实体，采用**内存映射技术**来提高文件的访问速度与写入速度，以追加写的方式写入MappedFile文件。

在写入消息之前，首先要获取MappedFile文件，才能知道将消息写到哪个MappedFile文件中，putMessage方法在写入消息到MappedFile文件中，调用getLastMappedFile方法获取最后一个MappedFile文件，让我们看看getLastMappedFile方法：

```java
//代码位置：org.apache.rocketmq.store.MappedFileQueue#getLastMappedFile
public MappedFile getLastMappedFile() {
        MappedFile mappedFileLast = null;

        while (!this.mappedFiles.isEmpty()) {
            try {
                mappedFileLast = this.mappedFiles.get(this.mappedFiles.size() - 1);
                break;
            } catch (IndexOutOfBoundsException e) {
                //continue;
            } catch (Exception e) {
                log.error("getLastMappedFile has exception.", e);
                break;
            }
        }

        return mappedFileLast;
}
```

mappedFiles是CopyOnWriteArrayList数组，保存着MappedFile，getLastMappedFile方法获取mappedFiles数组的最后一个MappedFile返回。如果获取的MappedFile文件为null或者已经写满，putMessage方法会重新创建一个新的MappedFile文件返回。我们看看MappedFile类：

```java
public class MappedFile extends ReferenceResource {
    //系统page大小，4M
    public static final int OS_PAGE_SIZE = 1024 * 4;
    protected static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.STORE_LOGGER_NAME);

    //总的mapped虚拟内存
    private static final AtomicLong TOTAL_MAPPED_VIRTUAL_MEMORY = new AtomicLong(0);

    //总的mapped文件数
    private static final AtomicInteger TOTAL_MAPPED_FILES = new AtomicInteger(0);
    //写指针
    protected final AtomicInteger wrotePosition = new AtomicInteger(0);
    //提交指针
    protected final AtomicInteger committedPosition = new AtomicInteger(0);
    //刷盘指针
    private final AtomicInteger flushedPosition = new AtomicInteger(0);
    //文件大小
    protected int fileSize;
    protected FileChannel fileChannel;
    /**
     * Message will put to here first, and then reput to FileChannel if writeBuffer is not null.
     */
    //写缓存，消息会先写到这里，reput方法会将写到文件中去
    protected ByteBuffer writeBuffer = null;
    //临时的存储池
    protected TransientStorePool transientStorePool = null;
    //文件名字
    private String fileName;
    private long fileFromOffset;
    //文件
    private File file;
    //
    private MappedByteBuffer mappedByteBuffer;
    //存储时间戳
    private volatile long storeTimestamp = 0;
    //第一次在队列中创建
    private boolean firstCreateInQueue = false;

    //省略代码
}
```

MappedFile的有几个重要的属性，上面代码已经为这些属性已经注释了，如写指针、读指针、刷盘指针、文件大小、文件名、写缓存等属性。

获取到MappedFile文件以后，就可以调用appendMessage方法进行消息的存储，appendMessage方法又会调用appendMessagesInner方法真正进行消息的保存：

```java
//代码位置：org.apache.rocketmq.store.MappedFile.appendMessagesInner
 public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
        assert messageExt != null;
        assert cb != null;

        //当前写指针
        int currentPos = this.wrotePosition.get();

        //写指针小于文件大小
        if (currentPos < this.fileSize) {
            ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
            byteBuffer.position(currentPos);
            AppendMessageResult result;
            //broker接收到的消息
            if (messageExt instanceof MessageExtBrokerInner) {
                result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);
            } else if (messageExt instanceof MessageExtBatch) {
                //批量消息
                result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch) messageExt);
            } else {
                return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
            }
            //增加写指针
            this.wrotePosition.addAndGet(result.getWroteBytes());
            //存储的时间
            this.storeTimestamp = result.getStoreTimestamp();
            return result;
        }
        log.error("MappedFile.appendMessage return null, wrotePosition: {} fileSize: {}", currentPos, this.fileSize);
        return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
 }
```

appendMessagesInner方法有两个参数，MessageExt是消息，AppendMessageCallback是追加消息回调。appendMessagesInner方法首先获取当前写指针，如果当前写指针小于文件的大小，那么就对消息进行处理，否则，就返回不知道错误的结果。

当写指针小于文件大小，判断messageExt是Broker接收到的消息还是批量消息，然后调用AppendMessageCallback的doAppend方法进行消息的追加，最后增加写指针。接下来我们分析下doAppend方法是如何将消息追加到文件中：

```java
//代码省略：org.apache.rocketmq.store.CommitLog#doAppend
public AppendMessageResult doAppend(final long fileFromOffset, final ByteBuffer byteBuffer, final int maxBlank,

            //代码省略
            /**
             * Serialize message
             */
            //序列化消息

            //消息的properties数据
            final byte[] propertiesData =
                msgInner.getPropertiesString() == null ? null : msgInner.getPropertiesString().getBytes(MessageDecoder.CHARSET_UTF8);

            //properties长度
            final int propertiesLength = propertiesData == null ? 0 : propertiesData.length;

            //消息properties太大
            if (propertiesLength > Short.MAX_VALUE) {
                log.warn("putMessage message properties length too long. length={}", propertiesData.length);
                return new AppendMessageResult(AppendMessageStatus.PROPERTIES_SIZE_EXCEEDED);
            }

            //topic数据和长度
            final byte[] topicData = msgInner.getTopic().getBytes(MessageDecoder.CHARSET_UTF8);
            final int topicLength = topicData.length;

            //消息体的长度
            final int bodyLength = msgInner.getBody() == null ? 0 : msgInner.getBody().length;

            //消息的长度
            final int msgLen = calMsgLength(msgInner.getSysFlag(), bodyLength, topicLength, propertiesLength);

            // Exceeds the maximum message 大于最大的消息长度4M
            if (msgLen > this.maxMessageSize) {
                CommitLog.log.warn("message size exceeded, msg total size: " + msgLen + ", msg body size: " + bodyLength
                    + ", maxMessageSize: " + this.maxMessageSize);
                return new AppendMessageResult(AppendMessageStatus.MESSAGE_SIZE_EXCEEDED);
            }

            // Determines whether there is sufficient free space
            //确定是否有足够的空间
            if ((msgLen + END_FILE_MIN_BLANK_LENGTH) > maxBlank) {
                this.resetByteBuffer(this.msgStoreItemMemory, maxBlank);
                // 1 TOTALSIZE
                this.msgStoreItemMemory.putInt(maxBlank);
                // 2 MAGICCODE
                this.msgStoreItemMemory.putInt(CommitLog.BLANK_MAGIC_CODE);
                // 3 The remaining space may be any value
                // Here the length of the specially set maxBlank
                final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
                //先写到byteBuffer缓存起来
                byteBuffer.put(this.msgStoreItemMemory.array(), 0, maxBlank);
                return new AppendMessageResult(AppendMessageStatus.END_OF_FILE, wroteOffset, maxBlank, msgId, msgInner.getStoreTimestamp(),
                    queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);
            }

            // Initialization of storage space
            //初始化内存
            this.resetByteBuffer(msgStoreItemMemory, msgLen);
             /**
             将消息添加到内存中
             **/
            // 1 TOTALSIZE
            this.msgStoreItemMemory.putInt(msgLen);
            // 2 MAGICCODE
            this.msgStoreItemMemory.putInt(CommitLog.MESSAGE_MAGIC_CODE);
            // 3 BODYCRC
            this.msgStoreItemMemory.putInt(msgInner.getBodyCRC());
            // 4 QUEUEID
            this.msgStoreItemMemory.putInt(msgInner.getQueueId());
            // 5 FLAG
            this.msgStoreItemMemory.putInt(msgInner.getFlag());
            // 6 QUEUEOFFSET
            this.msgStoreItemMemory.putLong(queueOffset);
            // 7 PHYSICALOFFSET
            this.msgStoreItemMemory.putLong(fileFromOffset + byteBuffer.position());
            // 8 SYSFLAG
            this.msgStoreItemMemory.putInt(msgInner.getSysFlag());
            // 9 BORNTIMESTAMP
            this.msgStoreItemMemory.putLong(msgInner.getBornTimestamp());
            // 10 BORNHOST
            this.resetByteBuffer(bornHostHolder, bornHostLength);
            this.msgStoreItemMemory.put(msgInner.getBornHostBytes(bornHostHolder));
            // 11 STORETIMESTAMP
            this.msgStoreItemMemory.putLong(msgInner.getStoreTimestamp());
            // 12 STOREHOSTADDRESS
            this.resetByteBuffer(storeHostHolder, storeHostLength);
            this.msgStoreItemMemory.put(msgInner.getStoreHostBytes(storeHostHolder));
            // 13 RECONSUMETIMES
            this.msgStoreItemMemory.putInt(msgInner.getReconsumeTimes());
            // 14 Prepared Transaction Offset
            this.msgStoreItemMemory.putLong(msgInner.getPreparedTransactionOffset());
            // 15 BODY
            this.msgStoreItemMemory.putInt(bodyLength);
            if (bodyLength > 0)
                this.msgStoreItemMemory.put(msgInner.getBody());
            // 16 TOPIC
            this.msgStoreItemMemory.put((byte) topicLength);
            this.msgStoreItemMemory.put(topicData);
            // 17 PROPERTIES
            this.msgStoreItemMemory.putShort((short) propertiesLength);
            if (propertiesLength > 0)
                this.msgStoreItemMemory.put(propertiesData);

            final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
            // Write messages to the queue buffer
             //将消息写到byteBuffer中
            byteBuffer.put(this.msgStoreItemMemory.array(), 0, msgLen);

            AppendMessageResult result = new AppendMessageResult(AppendMessageStatus.PUT_OK, wroteOffset, msgLen, msgId,
                msgInner.getStoreTimestamp(), queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);

            switch (tranType) {
                case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
                case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
                    break;
                case MessageSysFlag.TRANSACTION_NOT_TYPE:
                case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
                    // The next update ConsumeQueue information
                    CommitLog.this.topicQueueTable.put(key, ++queueOffset);
                    break;
                default:
                    break;
            }
            return result;
 }
```

上述将doAppend方法中一些代码省略掉，但是不影响主要流程，将消息的properties数据、topic数据转换为 byte数组，并计算properties数据的长度、topic数据的长度以及消息的长度。然后判断消息大小是否大于消息最大大小，如果是直接返回超过消息大小的结果。

接着确定是否有足够的空间存下消息，有充足的空间，则将消息写到byteBuffer缓存起来，返回文件已经写满的结果。如果没有充足的空间，初始化足够的空间，并将消息加入到msgStoreItemMemory中，最后将消息写入到byteBuffer中。doAppend方法并没有将消息直接写入到文件中，而是将消息写入到byteBuffer中缓存起来。

### 处理事务消息

当Broker接收到事务消息以后，根据消息的类型将消息交给TransactionalMessageService类的asyncPrepareMessage方法处理，这个asyncPrepareMessage的具体由TransactionalMessageServiceImpl实现。如下所示：

```java
//代码位置：org.apache.rocketmq.broker.transaction.queue.TransactionalMessageServiceImpl#asyncPrepareMessage
public CompletableFuture<PutMessageResult> asyncPrepareMessage(MessageExtBrokerInner messageInner) {
        return transactionalMessageBridge.asyncPutHalfMessage(messageInner);
}
```

asyncPrepareMessage方法调用asyncPutHalfMessage方法，如下：

```java
//代码位置：org.apache.rocketmq.broker.transaction.queue.TransactionalMessageBridge#asyncPutHalfMessage
public CompletableFuture<PutMessageResult> asyncPutHalfMessage(MessageExtBrokerInner messageInner) {
        return store.asyncPutMessage(parseHalfMessageInner(messageInner));
}
```

asyncPutHalfMessage方法首先调用parseHalfMessageInner方法先处理下半消息，处理完以后调用store的asyncPutMessage方法保存消息。我们先看看parseHalfMessageInner方法是如何处理半消息的。（RocketMQ事务消息不熟悉的可以参考阅读《RocketMQ源码分析之事务消息分析》）

```java
//代码位置：org.apache.rocketmq.broker.transaction.queue.TransactionalMessageBridge#parseHalfMessageInner
private MessageExtBrokerInner parseHalfMessageInner(MessageExtBrokerInner msgInner) {

        //将真实的topic和queueId保存在Property中
        //REAL_TOPIC
        MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_TOPIC, msgInner.getTopic());
        //REAL_QID
        MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_QUEUE_ID,
            String.valueOf(msgInner.getQueueId()));
        msgInner.setSysFlag(
            MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), MessageSysFlag.TRANSACTION_NOT_TYPE));
        //重新设置topic，设置的topic为RMQ_SYS_TRANS_HALF_TOPIC
        msgInner.setTopic(TransactionalMessageUtil.buildHalfTopic());
        //设置queueId为0
        msgInner.setQueueId(0);
        msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgInner.getProperties()));
        return msgInner;
}
```

parseHalfMessageInner方法首先将消息真实的topic和queueId存放到Property中，并重新设置topic为RMQ_SYS_TRANS_HALF_TOPIC，queueId为0，这样保存起来的消息就不能被消费者消费了。我们返回到asyncPutMessage方法中，asyncPutMessage方法最终也是会调用putMessage存储消息的，putMessage方法就是上述讲述的方法，跟上面普通消息处理是一样的，共用putMessage方法。

### 总结

Broker接收到消息时，会将消息交给SendMessageProcessor处理器处理，SendMessageProcessor处理器首先解析发送消息请求头，根据消息的类型交给不同方法处理消息。CommitLog类具体处理消息，首先获取最后一个MappedFile文件，然后将消息以追加的方式写入到MappedFile中的byteBuffer缓存起来。