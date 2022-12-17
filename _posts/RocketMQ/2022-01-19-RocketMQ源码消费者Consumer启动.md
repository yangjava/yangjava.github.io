---
layout: post
categories: RocketMQ
description: none
keywords: RocketMQ
---
# RocketMQ源码消费者启动流程分析

RocketMQ有push（推模式）和pull（拉模式）两种消费消息的模式，推模式就是Broker主动将消息推送给消费者，拉模式就是消费者主动从Broker将消息拉回来。推模式本质实际上是拉模式，是基于拉模式实现的。这篇文章分析推模式的消息者启动流程，实际上拉模式的消费者启动流程大致也是一样的。我们先看看消费者消费消息的案例。

### 消费者消费消息案例

```java
public class Consumer {

    public static void main(String[] args) throws InterruptedException, MQClientException {


        //创建默认的push消费者
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_4");

        //设置从哪里开始消费
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        //设置订阅的topic
        consumer.subscribe("TopicTest", "*");

        //注册消息监听器
        consumer.registerMessageListener(new MessageListenerConcurrently() {

            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                ConsumeConcurrentlyContext context) {
                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        //消费者启动
        consumer.start();

        System.out.printf("Consumer Started.%n");
    }
}
```

默认创建DefaultMQPushConsumer实例，设置消费者的消费位移，即从哪里开始消费，设置订阅哪一个topic，注册消息监听器，当消息到达时，则执行消费监听器消费消息。RocketMQ消息监听器有两种：MessageListenerConcurrently（并发消息监听器）、MessageListenerOrderly（顺序消息监听器）。MessageListenerConcurrently不管消息的顺序并发的消息消息，MessageListenerOrderly顺序消息。最后调用consumer.start()方法启动消费者。

```java
//代码位置：org.apache.rocketmq.client.consumer.DefaultMQPushConsumer#start
public void start() throws MQClientException {
        //设置消费组
        setConsumerGroup(NamespaceUtil.wrapNamespace(this.getNamespace(), this.consumerGroup));
        //启动消费者
        this.defaultMQPushConsumerImpl.start();
        //省略代码
}
```

consumer.start()方法首先设置消费组，然后调用defaultMQPushConsumerImpl.start()启动消费者。

defaultMQPushConsumerImpl.start()方法的逻辑比较重，接下来会分成几部分来讲：

```java
//代码位置：org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#start
public synchronized void start() throws MQClientException {
      switch (this.serviceState) {
            case CREATE_JUST:  
                //省略代码
              break;
            case RUNNING:
            case START_FAILED:
            case SHUTDOWN_ALREADY:
                throw new MQClientException("The PushConsumer service state not OK, maybe started once, "
                    + this.serviceState
                    + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                    null);
            default:
                break;
      }

     //省略代码
}
```

defaultMQPushConsumerImpl.start()方法方法首先根据服务启动的状态来不同的逻辑，这里分成两个分支，CREATE_JUST（刚创建）和其他的状态，CREATE_JUST状态的逻辑代码被省略了，将在下面进行讲解，RUNNING（运行中）、START_FAILED（启动失败）、SHUTDOWN_ALREADY（已经关闭）状态下，则抛出异常。

```java
//代码位置：org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#start
public synchronized void start() throws MQClientException {
      switch (this.serviceState) {
            case CREATE_JUST:  
                //设置服务状态为START_FAILED
                this.serviceState = ServiceState.START_FAILED;
                //检查配置
                this.checkConfig();
                //复制订阅信息
                this.copySubscription();
                //如果是集群模式，如果MQ实例是DEFAULT，改变成pid
                if (this.defaultMQPushConsumer.getMessageModel() == MessageModel.CLUSTERING) {
                    this.defaultMQPushConsumer.changeInstanceNameToPID();
                }

                //获取或者创建MQ客户端实例
                this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);
               //代码省略
              break;
      }

    //省略代码
}
```

当服务状态时CREATE_JUST时，首先设置服务状态serviceState为START_FAILED，checkConfig方法检查下一些配置信息，如消费组名称的正确性、消费偏移量、消费时间戳、消费消息的大小阈值等。copySubscription方法是将defaultMQPushConsumerImpl中的订阅信息拷贝到RebalancePushImpl中去。getOrCreateMQClientInstance方法就是创建MQ客户端实例，用于与Broker、NameServer进行通信。getOrCreateMQClientInstance方法首先根据客户端id从缓存factoryTable（clientId与MQClientInstance对应的关系map）中获取MQ客户端实例，如果MQ客户端实例等于空，则创建MQ客户端实例并保存在缓存factoryTable中。

```java
//代码位置：org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#start
public synchronized void start() throws MQClientException {
      switch (this.serviceState) {
            case CREATE_JUST:  
               //代码省略
              //设置负载均衡服务的属性
                this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
                this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
                this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
                this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

                //创建pullAPI
                this.pullAPIWrapper = new PullAPIWrapper(
                    mQClientFactory,
                    this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
                //注册过滤消息的钩子
                this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);
              break;
      }

    //省略代码
}
```

上述代码设置了负载均衡服务的一些属性，如消费组、消息模式、消息队列的策略模式、MQ客户端工厂。然后创建pullAPIWrapper实例，pullAPIWrapper是拉取消息的一些API封装，最后并为pullAPIWrapper注册过滤消息的钩子。

```java
//代码位置：org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#start
public synchronized void start() throws MQClientException {
      switch (this.serviceState) {
            case CREATE_JUST:  
               //代码省略
              //消费位移存储加载
                if (this.defaultMQPushConsumer.getOffsetStore() != null) {
                    this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
                } else {
                    switch (this.defaultMQPushConsumer.getMessageModel()) {
                        //广播模式
                        case BROADCASTING:
                            //本地文件偏移量存贮
                            this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                            break;
                        case CLUSTERING:
                            //远程broker偏移量存贮
                            this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                            break;
                        default:
                            break;
                    }
                    this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
                }
                //偏移量加载
                this.offsetStore.load();
              break;
      }

    //省略代码
}
```

如果消费位移存储不为空，就设置offsetStore，否则判断消息模式是集群还是广播模式。OffsetStore接口有两个实现：LocalFileOffsetStore、RemoteBrokerOffsetStore。LocalFileOffsetStore是本地文件的位移存储，RemoteBrokerOffsetStore是远程Broker位移存储。如果是广播模式，则设置offsetStore为LocalFileOffsetStore，否则设置为RemoteBrokerOffsetStore。最后offsetStore.load()方法加载消费位移。LocalFileOffsetStore的load方法是从本地文件中加载消费者位移，RemoteBrokerOffsetStore的load方法是一个空的实现，什么也没有做。

```java
//代码位置：org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#start
public synchronized void start() throws MQClientException {
      switch (this.serviceState) {
            case CREATE_JUST:  
                //代码省略
                 //判断消息监听器的类型
                 if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
                    this.consumeOrderly = true;
                    //顺序消费服务
                    this.consumeMessageService =
                        new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
                } else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
                    this.consumeOrderly = false;
                    //并发消费服务
                    this.consumeMessageService =
                        new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
                }

                //启动清消费者服务
                this.consumeMessageService.start();

                //注册消费者，将消费者保存到缓存中
                boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this);
                if (!registerOK) {
                    this.serviceState = ServiceState.CREATE_JUST;
                    this.consumeMessageService.shutdown();
                    throw new MQClientException("The consumer group[" + this.defaultMQPushConsumer.getConsumerGroup()
                        + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                        null);
                }

                //MQ客户端启动
                mQClientFactory.start();
                log.info("the consumer [{}] start OK.", this.defaultMQPushConsumer.getConsumerGroup());
                //服务器状态设置为运行中
                this.serviceState = ServiceState.RUNNING;
              break;
      }

    //省略代码
}
```

上述代码的逻辑如下：

- 判断消息监听器的类型，如果是MessageListenerOrderly（顺序消息监听器），则设置消息者服务consumeMessageService为ConsumeMessageOrderlyService（顺序消费者服务），否则为ConsumeMessageConcurrentlyService（并发消费者服务）。
- 启动清消费者服务，如果是ConsumeMessageOrderlyService，start()方法是每15分钟定时清理过期的消息。如果是ConsumeMessageOrderlyService则秒调用ConsumeMessageOrderlyService.this.lockMQPeriodically()方法锁定所有的Broker服务器。
- 注册消费者，将消费者保存到缓存中。
- MQ客户端启动，MQ客户端启动在《RocketMQ生产者启动过程分析》已经分析过了，这里就不分析了。
- 设置服务器的状态为运行中。

```java
//代码位置：org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#start
public synchronized void start() throws MQClientException {
      switch (this.serviceState) {
           //省略代码
      }

         //更新topic订阅信息
        this.updateTopicSubscribeInfoWhenSubscriptionChanged();
        //检测tag配置是否准确
        this.mQClientFactory.checkClientInBroker();
        //发送心跳给所有的btoker
        this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
        //唤醒负载均衡服务
        this.mQClientFactory.rebalanceImmediately();
}
```

消费者启动的最后一部分的逻辑如下：

- 更新topic订阅信息，从NameServer服务器拉取所有的路由信息，将已经改变的路由信息挑选出来，如果路由已经改变，将更新本地的路由信息，这部分具体的分析在《RocketMQ生产者启动过程分析》已经讲过了。
- 检查客户端配置信息
- 发送心跳信息给所有的Broker，这部分内容已经在《RocketMQ生产者启动过程分析》讲过了。
- 唤醒负载均衡服务。