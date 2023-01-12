---
layout: post
categories: RocketMQ
description: none
keywords: RocketMQ
---

**关键字：Broker 、元数据同步、RocketMQ 源码**

Broker 有2种角色：Master和Slave。

- Master：主要用于处理生产者、消费者的请求和存储数据。
- Slave：从 Master 同步元数据和消息数据到Slave的磁盘保存。

这一节分析Broker Slave如何从Master同步元数据，同步的元数据包括：topic、消费者位移、延迟位移以及订阅组配置。

Broker 在启动的时候，会判断Broker的角色是什么，当Broker的角色是Slave时，会启动定时任务开始进行同步元数据。

```text
//代码位置：org.apache.rocketmq.broker.BrokerController
private void handleSlaveSynchronize(BrokerRole role) {
        //如果是Slave
        if (role == BrokerRole.SLAVE) {
            if (null != slaveSyncFuture) {
                slaveSyncFuture.cancel(false);
            }
            this.slaveSynchronize.setMasterAddr(null);
            //启动定时任务同步元数据
            slaveSyncFuture = this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                @Override
                public void run() {
                    try {
                        BrokerController.this.slaveSynchronize.syncAll();
                    }
                    catch (Throwable e) {
                        log.error("ScheduledTask SlaveSynchronize syncAll error.", e);
                    }
                }
            }, 1000 * 3, 1000 * 10, TimeUnit.MILLISECONDS);
        } else {
            //handle the slave synchronise
            if (null != slaveSyncFuture) {
                slaveSyncFuture.cancel(false);
            }
            this.slaveSynchronize.setMasterAddr(null);
        }
 }
```

handleSlaveSynchronize方法判断Broker是否是Slave，如果是Slave则启动定时任务，每10秒同步元数据信息。

```text
public void syncAll() {
        //同步topic
        this.syncTopicConfig();
        //同步消费者位移
        this.syncConsumerOffset();
        //同步延迟位移
        this.syncDelayOffset();
        //同步订阅组关系
        this.syncSubscriptionGroupConfig();
}
```

## 同步topic

```text
//代码位置：org.apache.rocketmq.broker.slave.SlaveSynchronize
private void syncTopicConfig() {
        String masterAddrBak = this.masterAddr;
        if (masterAddrBak != null && !masterAddrBak.equals(brokerController.getBrokerAddr())) {
            try {
                //通过客户端从远程获取topic配置信息
                TopicConfigSerializeWrapper topicWrapper =
                    this.brokerController.getBrokerOuterAPI().getAllTopicConfig(masterAddrBak);
                //如果topic配置的数据版本改变了，说明数据变化了，需要更新Slave的topic数据
                if (!this.brokerController.getTopicConfigManager().getDataVersion()
                    .equals(topicWrapper.getDataVersion())) {

                    //设置版本号
                    this.brokerController.getTopicConfigManager().getDataVersion()
                        .assignNewOne(topicWrapper.getDataVersion());
                    //清除旧的topic信息
                    this.brokerController.getTopicConfigManager().getTopicConfigTable().clear();
                    //添加新的topic信息
                    this.brokerController.getTopicConfigManager().getTopicConfigTable()
                        .putAll(topicWrapper.getTopicConfigTable());
                    //topic持久化
                    this.brokerController.getTopicConfigManager().persist();

                    log.info("Update slave topic config from master, {}", masterAddrBak);
                }
            } catch (Exception e) {
                log.error("SyncTopicConfig Exception, {}", masterAddrBak, e);
            }
        }
}
```

syncTopicConfig首先通过网络传输，从Broker master拉去所有的topic的信息。如果拉回来的topic的数据版本与Slave的topic的数据版本不一样，说明topic已经改变了，需要更新Slave的topic。更新Slave的topic的操作包括设置新的数据版本号、清除旧的topic信息、添加新的topic信息、topic持久化到本地。

getAllTopicConfig方法是从通过网络从master获取所有的topic信息，客户端和服务端之间的通信在《RocketMQ的通信机制设计源码分析》中已经分析过，具体细节可以参考该文章。客户端通过请求码将消息发送给服务端，服务端接收到消息，通过请求码将请求交给不同处理器处理，然后将结果返回。

## 其他元数据的同步

消费者位移、延迟位移、订阅组关系的同步，与topic的同步都是大同小异。这里只放出源码，源码中都有详情的注释。

**消费者位移同步**

```text
private void syncConsumerOffset() {
        String masterAddrBak = this.masterAddr;
        if (masterAddrBak != null && !masterAddrBak.equals(brokerController.getBrokerAddr())) {
            try {
                //获取所有的消费者位移
                ConsumerOffsetSerializeWrapper offsetWrapper =
                    this.brokerController.getBrokerOuterAPI().getAllConsumerOffset(masterAddrBak);
                //将所有消费者位移添加到位移table中
                this.brokerController.getConsumerOffsetManager().getOffsetTable()
                    .putAll(offsetWrapper.getOffsetTable());
                //持久化
                this.brokerController.getConsumerOffsetManager().persist();
                log.info("Update slave consumer offset from master, {}", masterAddrBak);
            } catch (Exception e) {
                log.error("SyncConsumerOffset Exception, {}", masterAddrBak, e);
            }
        }
}
```

**延迟位移同步**

```text
private void syncDelayOffset() {
        String masterAddrBak = this.masterAddr;
        if (masterAddrBak != null && !masterAddrBak.equals(brokerController.getBrokerAddr())) {
            try {
                //获取所有的延迟位移
                String delayOffset =
                    this.brokerController.getBrokerOuterAPI().getAllDelayOffset(masterAddrBak);
                if (delayOffset != null) {

                    //消息保存配置文件路径
                    String fileName =
                        StorePathConfigHelper.getDelayOffsetStorePath(this.brokerController
                            .getMessageStoreConfig().getStorePathRootDir());
                    try {
                        //将延迟位移进行持久化
                        MixAll.string2File(delayOffset, fileName);
                    } catch (IOException e) {
                        log.error("Persist file Exception, {}", fileName, e);
                    }
                }
                log.info("Update slave delay offset from master, {}", masterAddrBak);
            } catch (Exception e) {
                log.error("SyncDelayOffset Exception, {}", masterAddrBak, e);
            }
        }
}
```

**订阅组关系的同步**

```text
private void syncSubscriptionGroupConfig() {
        String masterAddrBak = this.masterAddr;
        if (masterAddrBak != null  && !masterAddrBak.equals(brokerController.getBrokerAddr())) {
            try {
                //获取订阅组信息（消费者组）
                SubscriptionGroupWrapper subscriptionWrapper =
                    this.brokerController.getBrokerOuterAPI()
                        .getAllSubscriptionGroupConfig(masterAddrBak);

                if (!this.brokerController.getSubscriptionGroupManager().getDataVersion()
                    .equals(subscriptionWrapper.getDataVersion())) {
                    SubscriptionGroupManager subscriptionGroupManager =
                        this.brokerController.getSubscriptionGroupManager();
                    //设置数据版本
                    subscriptionGroupManager.getDataVersion().assignNewOne(
                        subscriptionWrapper.getDataVersion());
                    //删除旧的订阅组信息
                    subscriptionGroupManager.getSubscriptionGroupTable().clear();
                    //添加所有消费者信息
                    subscriptionGroupManager.getSubscriptionGroupTable().putAll(
                        subscriptionWrapper.getSubscriptionGroupTable());
                    //持久化信息
                    subscriptionGroupManager.persist();
                    log.info("Update slave Subscription Group from master, {}", masterAddrBak);
                }
            } catch (Exception e) {
                log.error("SyncSubscriptionGroup Exception, {}", masterAddrBak, e);
            }
        }
}
```