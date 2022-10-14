---
layout: post
title: RocketMQ源码(10)Broker存储批量消息流程分析
categories: RocketMQ
description: RocketMQ源码(10)Broker存储批量消息流程分析
keywords: RocketMQ，源码
---

我们已经在《[RocketMQ源码之Broker存储普通消息流程分析（一）](https://zhuanlan.zhihu.com/p/432299956)》和《[RocketMQ源码之Broker存储消息流程分析（二）](https://zhuanlan.zhihu.com/p/433140646)》分析了普通消息是如何存储的，这篇文章则分析如何存储批量消息。

当Broker收到消息时，将会将消息交给SendMessageProcessor处理器处理，SendMessageProcessor处理器根据消息的类型处理不同的消息，当消息是批量消息时，会将消息交给asyncSendBatchMessage方法进行处理。asyncSendBatchMessage方法如下：

```java
//代码位置：org.apache.rocketmq.broker.processor.SendMessageProcessor#asyncSendBatchMessage
private CompletableFuture<RemotingCommand> asyncSendBatchMessage(ChannelHandlerContext ctx, RemotingCommand request,
                                                                     SendMessageContext mqtraceContext,
                                                                     SendMessageRequestHeader requestHeader) {
        //创建消息响应
        final RemotingCommand response = preSend(ctx, request, requestHeader);
        final SendMessageResponseHeader responseHeader = (SendMessageResponseHeader)response.readCustomHeader();

        if (response.getCode() != -1) {
            return CompletableFuture.completedFuture(response);
        }

        int queueIdInt = requestHeader.getQueueId();
        //根据topic获取topic配置信息
        TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());

        if (queueIdInt < 0) {
            //随机一个写队列的id
            queueIdInt = randomQueueId(topicConfig.getWriteQueueNums());
        }

        //如果topic的长度大于127，则返回错误的响应
        if (requestHeader.getTopic().length() > Byte.MAX_VALUE) {
            response.setCode(ResponseCode.MESSAGE_ILLEGAL);
            response.setRemark("message topic length too long " + requestHeader.getTopic().length());
            return CompletableFuture.completedFuture(response);
        }

        //批量消息实体
        MessageExtBatch messageExtBatch = new MessageExtBatch();
        //设置topic和queueid
        messageExtBatch.setTopic(requestHeader.getTopic());
        messageExtBatch.setQueueId(queueIdInt);

        int sysFlag = requestHeader.getSysFlag();
        if (TopicFilterType.MULTI_TAG == topicConfig.getTopicFilterType()) {
            sysFlag |= MessageSysFlag.MULTI_TAGS_FLAG;
        }
        messageExtBatch.setSysFlag(sysFlag);

        messageExtBatch.setFlag(requestHeader.getFlag());
        MessageAccessor.setProperties(messageExtBatch, MessageDecoder.string2messageProperties(requestHeader.getProperties()));
        messageExtBatch.setBody(request.getBody());
        messageExtBatch.setBornTimestamp(requestHeader.getBornTimestamp());
        messageExtBatch.setBornHost(ctx.channel().remoteAddress());
        messageExtBatch.setStoreHost(this.getStoreHost());
        //消费次数
        messageExtBatch.setReconsumeTimes(requestHeader.getReconsumeTimes() == null ? 0 : requestHeader.getReconsumeTimes());
        String clusterName = this.brokerController.getBrokerConfig().getBrokerClusterName();
        MessageAccessor.putProperty(messageExtBatch, MessageConst.PROPERTY_CLUSTER, clusterName);

        //存储批量消息
        CompletableFuture<PutMessageResult> putMessageResult = this.brokerController.getMessageStore().asyncPutMessages(messageExtBatch);
        //处理消息存储结果
        return handlePutMessageResultFuture(putMessageResult, response, request, messageExtBatch, responseHeader, mqtraceContext, ctx, queueIdInt);
}
```

asyncSendBatchMessage方法跟普通消息的处理流程差不多，创建消息响应，根据topic查找topic配置信息、创建消息内容实体。然后调用asyncPutMessages方法进行存储批量消息，当批量消息存储完以后，就处理批量消息存储的结果。我们来深入asyncPutMessages方法，asyncPutMessages方法会调用putMessages方法存储消息，putMessages方法如下：

```java
//代码位置：org.apache.rocketmq.store.DefaultMessageStore#putMessages
public PutMessageResult putMessages(MessageExtBatch messageExtBatch) {
        //检查store服务器的状态
        PutMessageStatus checkStoreStatus = this.checkStoreStatus();
        //如果store状态不为ok时，直接返回结果
        if (checkStoreStatus != PutMessageStatus.PUT_OK) {
            return new PutMessageResult(checkStoreStatus, null);
        }

        //消息检查
        PutMessageStatus msgCheckStatus = this.checkMessages(messageExtBatch);
        if (msgCheckStatus == PutMessageStatus.MESSAGE_ILLEGAL) {
            return new PutMessageResult(msgCheckStatus, null);
        }

        //存储消息
        long beginTime = this.getSystemClock().now();
        PutMessageResult result = this.commitLog.putMessages(messageExtBatch);
        long elapsedTime = this.getSystemClock().now() - beginTime;
        if (elapsedTime > 500) {
            log.warn("not in lock elapsed time(ms)={}, bodyLength={}", elapsedTime, messageExtBatch.getBody().length);
        }

        //统计存储消息花费的时间
        this.storeStatsService.setPutMessageEntireTimeMax(elapsedTime);

        //如果失败，统计失败的次数
        if (null == result || !result.isOk()) {
            this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
        }

        return result;
}
```

putMessages方法会首先检查store服务器的状态和对消息进行检查，这个跟普通消息的处理是一样的，就不深入checkStoreStatus和checkMessages方法了。然后调用commitLog的putMessages存储消息，最后统计存储消息所花费的时间以及存储消息失败的次数。接下来，我们继续深入putMessages方法是如何存储批量消息的：

```java
public PutMessageResult putMessages(final MessageExtBatch messageExtBatch) {

        //代码省略


        MappedFile unlockMappedFile = null;
        //获取最后一个mappedFile
        MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();

        //fine-grained lock instead of the coarse-grained
        //消息编码类
        MessageExtBatchEncoder batchEncoder = batchEncoderThreadLocal.get();

        //设置已经编码好的消息
        messageExtBatch.setEncodedBuff(batchEncoder.encode(messageExtBatch));

        putMessageLock.lock();
        try {
            long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
            this.beginTimeInLock = beginLockTimestamp;

            // Here settings are stored timestamp, in order to ensure an orderly
            // global
            messageExtBatch.setStoreTimestamp(beginLockTimestamp);

            //mappedFile为空或者已经写满，重新创建mappedFile
            if (null == mappedFile || mappedFile.isFull()) {
                mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
            }
            //如果mappedFile为null，则返回创建mappedFile失败的结果
            if (null == mappedFile) {
                log.error("Create mapped file1 error, topic: {} clientAddr: {}", messageExtBatch.getTopic(), messageExtBatch.getBornHostString());
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
            }

            //存储消息
            result = mappedFile.appendMessages(messageExtBatch, this.appendMessageCallback);
            //对存储消息的结果进行处理
            switch (result.getStatus()) {
                case PUT_OK:
                    break;
                case END_OF_FILE:
                    unlockMappedFile = mappedFile;
                    // Create a new file, re-write the message
                    mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                    if (null == mappedFile) {
                        // XXX: warn and notify me
                        log.error("Create mapped file2 error, topic: {} clientAddr: {}", messageExtBatch.getTopic(), messageExtBatch.getBornHostString());
                        beginTimeInLock = 0;
                        return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
                    }
                    result = mappedFile.appendMessages(messageExtBatch, this.appendMessageCallback);
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

        if (elapsedTimeInLock > 500) {
            log.warn("[NOTIFYME]putMessages in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", elapsedTimeInLock, messageExtBatch.getBody().length, result);
        }

        if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
            this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
        }

        PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

        // Statistics
        //统计相关
        storeStatsService.getSinglePutMessageTopicTimesTotal(messageExtBatch.getTopic()).addAndGet(result.getMsgNum());
        storeStatsService.getSinglePutMessageTopicSizeTotal(messageExtBatch.getTopic()).addAndGet(result.getWroteBytes());

        //消息刷盘
        handleDiskFlush(result, putMessageResult, messageExtBatch);

        //消息同步
        handleHA(result, putMessageResult, messageExtBatch);

        return putMessageResult;
}
```

上述省略了一些无关紧要的代码，putMessages方法处理批量消息的存储与普通消息的存储的流程差不多，首先获取最后一个mappedFile，封装消息，这里的封装消息跟普通消息的存储有点不一样，就是这里专门用MessageExtBatchEncoder类处理批量消息，利用batchEncoder.encode对批量消息进行遍历写入内存。在批量消息写入之前，首先获取可重入锁，然后调用appendMessages方法对批量消息进行存储，当批量消息存储结果返回时，根据存储的结果状态返回不同处理结果。最后记录统计相关结果、消息进行刷盘、消息同步等。批量消息存储的流程跟普通消息存储的流程大体都相同。

appendMessages方法将消息追加到文件中，在批量消息存储和普通消息存储都用到了这个方法。将批量消息进行追加到文件中，跟普通消息追加到文件中，差别比较大的就是批量消息的追加是需要遍历追加文件的，看懂了普通消息的追加，批量消息的追加也是比较容易看懂的，这里就不继续分析，看懂《[RocketMQ源码之Broker存储消息流程分析（二）](https://zhuanlan.zhihu.com/p/433140646)》这篇文章的分析，批量消息的存储也是没有什么问题。