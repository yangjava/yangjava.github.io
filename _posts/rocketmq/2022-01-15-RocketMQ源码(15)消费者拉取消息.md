## RocketMQ源码之消费者拉取消息（一）

RocketMQ消费者从Broker服务器拉取消息的过程分为两步：**消费者端发送拉取消息的请求、Broker端将消息返回给消费者**。

这篇文章主要讲解的是第一步的内容，消费者端发送拉取消息的请求。PullMessageService是一个线程，在消息者启动的时候，就启动PullMessageService线程，PullMessageService线程不断从队列中获取拉取消息的请求，然后拉取消息。

```java
//代码位置：org.apache.rocketmq.client.impl.consumer.PullMessageService
public void run() {
        log.info(this.getServiceName() + " service started");

        while (!this.isStopped()) {
            try {
                //从队列中获取到拉取消息请求
                PullRequest pullRequest = this.pullRequestQueue.take();
                this.pullMessage(pullRequest);
            } catch (InterruptedException ignored) {
            } catch (Exception e) {
                log.error("Pull Message Service Run Method exception", e);
            }
        }

        log.info(this.getServiceName() + " service end");
}
```

pullRequestQueue是 LinkedBlockingQueue阻塞队列，保存着拉取消息的请求实体。run方法一直运行，直到线程停止。首先从队列中获取拉取消息请求，然后交给pullMessage方法处理。pullMessage方法首先根据消费者组获取消息者，然后将拉取消息请求实体交给消费者的pullMessage方法，该方法的代码比较多，但是主要逻辑如下：

- 检测处理队列的状态、服务的状态以及消费者的状态
- 限流
- 发送拉取消息的请求

### 检测处理队列的状态、服务的状态以及消费者的状态

```java
//代码位置：org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#pullMessage
public void pullMessage(final PullRequest pullRequest) {
     //处理队列
        final ProcessQueue processQueue = pullRequest.getProcessQueue();
        if (processQueue.isDropped()) {
            log.info("the pull request[{}] is dropped.", pullRequest.toString());
            return;
        }

        //更新处理队列的最后拉取消息的时间
        pullRequest.getProcessQueue().setLastPullTimestamp(System.currentTimeMillis());

        try {
            //检测服务是否正常
            this.makeSureStateOK();
        } catch (MQClientException e) {
            log.warn("pullMessage exception, consumer state not ok", e);
            this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
            return;
        }

        //如果消费者是暂停的，延迟发送拉取消息请求
        if (this.isPause()) {
            log.warn("consumer was paused, execute pull request later. instanceName={}, group={}", this.defaultMQPushConsumer.getInstanceName(), this.defaultMQPushConsumer.getConsumerGroup());
            this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_SUSPEND);
            return;
        }

    //代码省略
}
```

如果处理队列的状态是被丢弃的，那么就直接返回。如果服务的状态正常，则进行下一步，否则调用executePullRequestLater方法将消息请求实体放进队列中，稍后再处理。如果消息者是暂停消息的，也是调用executePullRequestLater方法延迟发送拉取消息请求，将消息请求放入队列中。

### 限流

```java
//代码位置：org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#pullMessage
public void pullMessage(final PullRequest pullRequest) {

    //代码省略
     //缓存的消息数量
     long cachedMessageCount = processQueue.getMsgCount().get();
     //缓存的消息内存大小
     long cachedMessageSizeInMiB = processQueue.getMsgSize().get() / (1024 * 1024);

     //消息数量大于拉取消息队列的大小（1000）
     if (cachedMessageCount > this.defaultMQPushConsumer.getPullThresholdForQueue()) {
            //延迟发送
            this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
            //缓存的消息大小超过阈值，进行限流控制
            if ((queueFlowControlTimes++ % 1000) == 0) {
                log.warn(
                    "the cached message count exceeds the threshold {}, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
                    this.defaultMQPushConsumer.getPullThresholdForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
            }
            return;
        }
        //如果缓存的消息内存大于拉取消息队列的消息的阈值大小
        if (cachedMessageSizeInMiB > this.defaultMQPushConsumer.getPullThresholdSizeForQueue()) {
            //延迟发送消息
            this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
            //缓存的消息大小超过阈值，进行限流控制
            if ((queueFlowControlTimes++ % 1000) == 0) {
                log.warn(
                    "the cached message size exceeds the threshold {} MiB, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
                    this.defaultMQPushConsumer.getPullThresholdSizeForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
            }
            return;
        }

    //代码省略
}
```

当上述检测都正常时，接下来获取缓存消息的数量以及缓存消息的大小，当缓存消息的数量大于拉取消息队列的数量或者缓存消息的大小大于拉取消息队列的消息的阈值大小，那么就调用executePullRequestLater方法延迟发送拉取消息的请求，然后直接返回。这就限制了发送拉取消息请求，到达限流的目的。

### 发送拉取消息的请求

```java
//代码位置：org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#pullMessage
public void pullMessage(final PullRequest pullRequest) {

    //省略代码
     try {
            this.pullAPIWrapper.pullKernelImpl(
                pullRequest.getMessageQueue(),
                subExpression,
                subscriptionData.getExpressionType(),
                subscriptionData.getSubVersion(),
                pullRequest.getNextOffset(),
                this.defaultMQPushConsumer.getPullBatchSize(),
                sysFlag,
                commitOffsetValue,
                BROKER_SUSPEND_MAX_TIME_MILLIS,
                CONSUMER_TIMEOUT_MILLIS_WHEN_SUSPEND,
                CommunicationMode.ASYNC,
                pullCallback
            );
        } catch (Exception e) {
            log.error("pullKernelImpl exception", e);
            this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
    }
    //省略代码
}
```

发送拉取消息的请求就是调用pullKernelImpl方法进行发送，这里使用异步的发送的发送方式。pullKernelImpl方法首先通过brokerName获取broker的服务器的相关信息，这样就知道从哪一个broker服务器拉取消息了，然后构造拉取消息的请求头，最后发送拉取消息的请求，发送请求是通过netty进行发送的。

### 总结

消费者端发送拉取消息的请求逻辑也是比较简单，PullMessageService线程服务在消息者启动的时候启动，不断从阻塞队列中获取拉取消息的请求，然后给broker服务器发送拉取消息的请求，发送拉取消息请求的过程中，首先进行一些校验，如线程服务的状态是否正常、消息者对否停止消费、是否需要限流的等，如果费正常情况下会进行延迟发送拉取消息的请求，实际就是将拉取消息的请求放进阻塞队列中。最后通过brokerName获取broker服务器的地址，构造拉取消息的请求头，将拉取消息的请求通过netty发送给broker服务器。

我们发现其实他还是通过DefaultMQPushConsumerImpl类的pullMessage方法来进行消息的逻辑处理.

#### pullRequest拉取方式

PullRequest这里说明一下，上面我们已经提了一下rocketmq的push模式其实是通过pull模式封装实现的，pullrequest这里是通过长轮询的方式达到push效果。

长轮询方式既有pull的优点又有push模式的实时性有点。

- push方式是server端接收到消息后，主动把消息推送给client端，实时性高。弊端是server端工作量大，影响性能，其次是client端处理能力不同且client端的状态不受server端的控制，如果client端不能及时处理消息容易导致消息堆积已经影响正常业务等。
- pull方式是client循环从server端拉取消息，主动权在client端，自己处理完一个消息再去拉取下一个，缺点是循环的时间不好设定，时间太短容易忙等，浪费CPU资源，时间间隔太长client的处理能力会下降，有时候有些消息会处理不及时。

##### 长轮询的方式可以结合两者优点

1. 检查PullRequest对象中的ProcessQueue对象的dropped是否为true（在RebalanceService线程中为topic下的MessageQueue创建拉取消息请求时要维护对应的ProcessQueue对象，若Consumer不再订阅该topic则会将该对象的dropped置为true）；若是则认为该请求是已经取消的，则直接跳出该方法；
2. 更新PullRequest对象中的ProcessQueue对象的时间戳（ProcessQueue.lastPullTimestamp）为当前时间戳；
3. 检查该Consumer是否运行中，即DefaultMQPushConsumerImpl.serviceState是否为RUNNING;若不是运行状态或者是暂停状态（DefaultMQPushConsumerImpl.pause=true），则调用PullMessageService.executePullRequestLater(PullRequest pullRequest, long timeDelay)方法延迟再拉取消息，其中timeDelay=3000；该方法的目的是在3秒之后再次将该PullRequest对象放入PullMessageService. pullRequestQueue队列中；并跳出该方法；
4. 进行流控。若ProcessQueue对象的msgCount大于了消费端的流控阈值（DefaultMQPushConsumer.pullThresholdForQueue，默认值为1000），则调用PullMessageService.executePullRequestLater方法，在50毫秒之后重新该PullRequest请求放入PullMessageService.pullRequestQueue队列中；并跳出该方法；
5. 若不是顺序消费（即DefaultMQPushConsumerImpl.consumeOrderly等于false），则检查ProcessQueue对象的msgTreeMap:TreeMap<Long,MessageExt>变量的第一个key值与最后一个key值之间的差额，该key值表示查询的队列偏移量queueoffset；若差额大于阈值（由DefaultMQPushConsumer. consumeConcurrentlyMaxSpan指定，默认是2000），则调用PullMessageService.executePullRequestLater方法，在50毫秒之后重新将该PullRequest请求放入PullMessageService.pullRequestQueue队列中；并跳出该方法；
6. 以PullRequest.messageQueue对象的topic值为参数从RebalanceImpl.subscriptionInner: ConcurrentHashMap, SubscriptionData>中获取对应的SubscriptionData对象，若该对象为null，考虑到并发的关系，调用executePullRequestLater方法，稍后重试；并跳出该方法；
7. 若消息模型为集群模式（RebalanceImpl.messageModel等于CLUSTERING），则以PullRequest对象的MessageQueue变量值、type =READ_FROM_MEMORY（从内存中获取消费进度offset值）为参数调用DefaultMQPushConsumerImpl. offsetStore对象（初始化为RemoteBrokerOffsetStore对象）的readOffset(MessageQueue mq, ReadOffsetType type)方法从本地内存中获取消费进度offset值。若该offset值大于0 则置临时变量commitOffsetEnable等于true否则为false；该offset值作为pullKernelImpl方法中的commitOffset参数，在Broker端拉取消息之后根据commitOffsetEnable参数值决定是否用该offset更新消息进度。该readOffset方法的逻辑是：以入参MessageQueue对象从RemoteBrokerOffsetStore.offsetTable:ConcurrentHashMap <MessageQueue,AtomicLong>变量中获取消费进度偏移量；若该偏移量不为null则返回该值，否则返回-1；
8. 当每次拉取消息之后需要更新订阅关系（由DefaultMQPushConsumer. postSubscriptionWhenPull参数表示，默认为false）并且以topic值参数从RebalanceImpl.subscriptionInner获取的SubscriptionData对象的classFilterMode等于false（默认为false），则将sysFlag标记的第3个字节置为1，否则该字节置为0；
9. 该sysFlag标记的第1个字节置为commitOffsetEnable的值；第2个字节（suspend标记）置为1；第4个字节置为classFilterMode的值；
10. 初始化匿名内部类PullCallback，实现了onSucess/onException方法； 该方法只有在异步请求的情况下才会回调；
11. 调用底层的拉取消息API接口：