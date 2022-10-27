---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---

### 源码分析-DataCarrier 异步处理库

DataCarrier 是一个轻量级的生产者-消费者模式的实现库， Skywalking Agent 在收集到 Trace 数据之后，会先将 Trace 数据写入到 DataCarrier 中缓存，然后由后台线程定时发送到 Skywalking 服务端。呦吼，和前面介绍的 Dubbo MonitorFilter 的实现原理类似啊，emmm，其实多数涉及到大量网络传输的场景都会这么设计：先本地缓存聚合，然后定时发送。

DataCarrier 实现在 Skywalking 中是一个单独的模块

首先来看 Buffer 这个类，它是一个环形缓冲区，是整个 DataCarrier 最底层的一个类，其核心字段如下：

这里的 AtomicRangeInteger 是环形指针，我们可以指定其中的 value字段（AtomicInteger类型）从 start值（int类型）开始递增，当 value 递增到 end值（int类型）时，value字段会被重置为 start值，从而实现环形指正的效果。

Buffer.save() 方法负责向当前 Buffer 缓冲区中填充数据，实现如下：

**个人感觉：BLOCKING策略下，Buffer 在写满之后，index 发生循环，****可能会出现两个线程同时等待写入一个位置的情况，有并发问题啊，会丢数据~~~看官大佬们可以自己思考一下~~~**

生产者用的是 save() 方法，消费者用的是 Buffer.obtain() 方法：

Channels 是另一个比较基础的类，它底层封装了多个 Buffer 实例以及一个分区选择器，字段如下所示：

IDataPartitioner 提供了类似于 Kafka 的分区功能，当数据并行写入的时候，由分区选择器定将数据写入哪个分区，这样就可以有效降低并发导致的锁(或CAS)竞争，降低写入压力，可提高效率。IDataPartitioner 接口有两个实现，如下下图所示：

ProducerThreadPartitioner会根据写入线程的ID进行选择，这样可以保证相同的线程号写入的数据都在一个分区中。

SimpleRollingPartitioner简单循环自增选择器，使用无锁整型（volatile修饰）的自增，顺序选择线程号，在高负载时，会产生批量连续写入一个分区的效果，在中等负载情况下，提供较好性能。

下面来看 IConsumer 接口， DataCarrier 的消费者逻辑需要封装在其中。IConsumer 接口定义了消费者的基本行为：

后面即将介绍的TraceSegmentServiceClient 类就实现了 IConsumer 接口，后面展开说。

ConsumerThread 继承了Thread，表示的是消费者线程，其核心字段如下：

ConsumerThread.run() 方法会循环 consume() 方法：

ConsumerThread.consume() 方法是消费的核心：

另一个消费者线程是 MultipleChannelsConsumer，与 ConsumerThread 的区别在于:

ConsumerThread 处理的是一个确定的 IConsumer 消费一个 Channel 中的多个 Buffer。

MultipleChannelsConsumer 可以处理多组 Group，每个 Group 都是一个IConsumer+一个 Channels。

先来看 MultipleChannelsConsumer 的核心字段：

MultipleChannelsConsumer.consume()方法消费的依然是单个IConsumer，这里不再展开分析。在MultipleChannelsConsumer.run()方法中会循环每个 Group：

在MultipleChannelsConsumer.addNewTarget() 方法中会添加新的 Group。这里用了 Copy-on-Write，因为在添加的过程中，MultipleChannelsConsumer线程可能正在循环处理 consumeTargets 集合（这也是为什么consumeTargets用 volatile 修饰的原因）：

IDriver 负责将 ConsumerThread 和 IConsumer 集成到一起，其实现类如下：

先来看 ConsumeDriver，核心字段如下：

在 ConsumeDriver 的构造方法中，会初始化上述两个集合：

完成初始化之后，要调用 ConsumeDriver.begin() 方法，其中会根据 Buffer 的数量以及ConsumerThread 线程数进行分配：

如果 Buffer 个数较多，则一个 ConsumerThread 线程需要处理多个 Buffer。

如果 ConsumerThread 线程数较多，则一个 Buffer 被多个 ConsumerThread 线程处理（每个 ConsumerThread 线程负责消费这个 Buffer 的一段）。

如果两者数量正好相同，则是一对一关系。

逻辑虽然很简单，但是具体的分配代码比较长，这里就不粘贴了。

BulkConsumePool 是 IDriver 接口的另一个实现，其中的 allConsumers 字段（List类型）记录了当前启动的MultipleChannelsConsumer线程，在BulkConsumePool 的构造方法中会根据配置初始化该集合：

BulkConsumePool.add() 方法提供了添加新 Group的功能：

最后来看 DataCarrier ，它是整个 DataCarrier 库的入口和门面，其核心字段如下：

在 DataCarrier 的构造方法中会初始化 Channels 对象，默认使用 SimpleRollingPartitioner 以及 BLOCKING 策略，太简单，不展开了。

DataCarrier.produce() 方法实际上是调用 Channels.save()方法，实现数据写入，太简单，不展开了。

在 DataCarrier.consume()方法中，会初始化 ConsumeDriver 对象并调用 begin() 方法，启动 ConsumeThread 线程：