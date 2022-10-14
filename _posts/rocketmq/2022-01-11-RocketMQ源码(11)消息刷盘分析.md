在《[RocketMQ源码之Broker存储普通消息流程分析（二）](https://zhuanlan.zhihu.com/p/433140646)》和《[RocketMQ源码之Broker存储批量消息流程分析](https://zhuanlan.zhihu.com/p/433227003)》这两篇文章中，分析了消息是如何存储的。实际上，这两篇文章只分析到了将消息写到缓存中，并没有真正将消息写到文件中。这一篇文章就是分析消息是如何刷盘的。

在创建CommitLog对象的时候，会初始化刷盘服务：

```java
//代码位置：org.apache.rocketmq.store.CommitLog
public CommitLog(final DefaultMessageStore defaultMessageStore) {
    //省略代码
    //同步刷盘
    if (FlushDiskType.SYNC_FLUSH == defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
        this.flushCommitLogService = new GroupCommitService();
    } else {
        //异步刷盘
        this.flushCommitLogService = new FlushRealTimeService();
    }
    //实时提交服务
    this.commitLogService = new CommitRealTimeService();

    //省略代码
}
```

刷盘方式有同步和异步刷盘，RocketMQ默认的是异步刷盘，同步刷盘就是当消息追加到缓存以后，就立即进行刷盘到文件中保存起来，异步刷盘就是后台任务进行异步刷盘操作。刷盘服务的继承关系如下：

![img](https://pic4.zhimg.com/80/v2-b22c1e13ea2beaed5ff234f290df64b7_720w.jpg)

从上面类图可以看出，不管是同步刷盘还是异步刷盘都是继承了FlushCommitLogService类，**GroupCommitService类用于同步刷盘，FlushRealTimeService用于异步刷盘，CommitRealTimeService用于将缓存的消息写到file channel**。在CommitLog启动的时候，会启动刷盘服务，代码如下：

```java
//代码位置：org.apache.rocketmq.store.CommitLog#start
 public void start() {
        this.flushCommitLogService.start();

        //如果TransientStorePool开关打开，flush message to FileChannel at fixed periods
        if (defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
            this.commitLogService.start();
        }
}
```

在putMessage方法中，当消息写到缓存以后，就会调用handleDiskFlush方法进行刷盘，handleDiskFlush方法如下：

```java
//代码位置：org.apache.rocketmq.store.CommitLog#handleDiskFlush
public void handleDiskFlush(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
        // Synchronization flush
        //同步刷盘
        if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
            final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
            //等待消息存储成功
            if (messageExt.isWaitStoreMsgOK()) {
                //创建一个同步刷盘请求
                GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
                //添加同步刷盘请求
                service.putRequest(request);
                CompletableFuture<PutMessageStatus> flushOkFuture = request.future();
                PutMessageStatus flushStatus = null;
                try {
                    //阻塞获取刷盘状态
                    flushStatus = flushOkFuture.get(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout(),
                            TimeUnit.MILLISECONDS);
                } catch (InterruptedException | ExecutionException | TimeoutException e) {
                    //flushOK=false;
                }
                //刷盘的状态不成功
                if (flushStatus != PutMessageStatus.PUT_OK) {
                    log.error("do groupcommit, wait for flush failed, topic: " + messageExt.getTopic() + " tags: " + messageExt.getTags()
                        + " client address: " + messageExt.getBornHostString());
                    putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_DISK_TIMEOUT);
                }
            } else {
                //唤醒刷盘服务
                service.wakeup();
            }
        }
        // Asynchronous flush
        else {
            //TransientStorePool开关不可用
            if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
                //唤醒异步刷盘服务
                flushCommitLogService.wakeup();
            } else {
                //唤醒commitLogService服务
                commitLogService.wakeup();
            }
        }
 }
```

handleDiskFlush方法根据刷盘方式处理不同的逻辑：

- **同步刷盘**

如果消息存储到缓存中不成功，就调用wakeup方法唤醒同步刷盘服务，否则创建同步刷盘请求，putRequest方法添加到同步刷盘请求队列中，然后阻塞等待同步刷盘的状态，如果同步刷盘不成功，则打印等待刷盘的错误日志。

- **异步刷盘**

如果TransientStorePoolEnable开关不可用，则唤醒异步刷盘服务，否则唤醒提交服务。

### **同步刷盘服务GroupCommitService**

```java
//刷盘请求写队列
private volatile List<GroupCommitRequest> requestsWrite = new ArrayList<GroupCommitRequest>();
//刷盘请求读队列
private volatile List<GroupCommitRequest> requestsRead = new ArrayList<GroupCommitRequest>();
```

GroupCommitService类有两个属性，requestsWrite是刷盘请求写队列，当刷盘请求过来时，将刷盘请求添加到requestsWrite，当需要刷盘时，遍历requestsRead的刷盘请求将进行刷盘。在上述handleDiskFlush方法中，调用putRequest方法就是将刷盘请求写到requestsWrite中，代码如下：

```java
//代码位置：org.apache.rocketmq.store.CommitLog.GroupCommitService#putRequest
public synchronized void putRequest(final GroupCommitRequest request) {
     //添加到写请求list中
    synchronized (this.requestsWrite) {
        this.requestsWrite.add(request);
    }
    //如果更新成功，就唤醒waitPoint
    if (hasNotified.compareAndSet(false, true)) {
      waitPoint.countDown(); // notify
    }
}
```

putRequest方法用synchronized修饰requestsWrite，然后将同步刷盘请求写到requestsWrite中，如果更新通知标志成功，就唤醒GroupCommitService服务。当GroupCommitService服务被唤醒以后就会调用swapRequests方法，如下：

```java
//代码位置：org.apache.rocketmq.store.CommitLog.GroupCommitService#swapRequests
private void swapRequests() {
   List<GroupCommitRequest> tmp = this.requestsWrite;
   this.requestsWrite = this.requestsRead;
   this.requestsRead = tmp;
}
```

swapRequests方法的作用就是将 requestsWrite 中的请求交换到 requestsRead中，这样就处理刷盘的时候，也可以接受刷盘请求，避免产生竞争。

GroupCommitService服务是一个线程，启动以后就会在后台不断调用doCommit方法进行刷盘，doCommit方法遍历requestsRead的同步刷盘请求进行处理：

```java
//代码位置：org.apache.rocketmq.store.CommitLog.GroupCommitService#doCommit
private void doCommit() {
            //同步锁，防止刷盘请求写入队列与刷盘请求读队列互换
            synchronized (this.requestsRead) {
                if (!this.requestsRead.isEmpty()) {
                    for (GroupCommitRequest req : this.requestsRead) {
                        // There may be a message in the next file, so a maximum of
                        // two times the flush
                        boolean flushOK = false;
                        //两次刷盘，因为可能第一次刷盘并没有将消息全部刷进文件中，文件是固定大小的
                        for (int i = 0; i < 2 && !flushOK; i++) {

                            flushOK = CommitLog.this.mappedFileQueue.getFlushedWhere() >= req.getNextOffset();
                            //如果刷盘偏移量小于下一位移，说明还能继续刷盘
                            if (!flushOK) {
                                CommitLog.this.mappedFileQueue.flush(0);
                            }
                        }

                        req.wakeupCustomer(flushOK);
                    }

                    //设置检查点的写入时间
                    long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
                    if (storeTimestamp > 0) {
                        CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
                    }

                    //清空requestsRead
                    this.requestsRead.clear();
                } else {
                    // Because of individual messages is set to not sync flush, it
                    // will come to this process
                    CommitLog.this.mappedFileQueue.flush(0);
                }
            }
}
```

doCommit方法首先用synchronized对requestsRead进行同步锁上锁，防止同一时间其他线程操作requestsRead。如果requestsRead不为空，则遍历requestsRead，对同一同步刷盘请求，进行了两次刷盘，原因是可能第一次刷盘并没有把消息全部刷盘到文件中，文件是固定大小的，当文件差不多满了的时候，消息只能再一次刷到下一个文件中。调用方法getFlushedWhere判断当前刷盘偏移量是否大于等于该消息写入后的结束偏移量，如果否，则调用mappedFileQueue.flush方法进行刷盘，该方法等到分析完异步刷盘再一起分析。刷完盘以后，调用wakeupCustomer方法唤醒其他外部等待的线程。遍历完刷盘同步请求时，记录检查点的写入时间并清空requestsRead。

### 异步刷盘服务FlushRealTimeService

RocketMQ默认的是异步刷盘，异步刷盘服务FlushRealTimeService启动以后就在run方法不停的运行,run方法如下：

```java
//代码位置：org.apache.rocketmq.store.CommitLog.FlushRealTimeService#run
public void run() {
            CommitLog.log.info(this.getServiceName() + " service started");

            while (!this.isStopped()) {
                //是否固定时间周期进行刷盘
                boolean flushCommitLogTimed = CommitLog.this.defaultMessageStore.getMessageStoreConfig().isFlushCommitLogTimed();

                //刷盘的间隔
                int interval = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushIntervalCommitLog();
                //刷盘的最少页数
                int flushPhysicQueueLeastPages = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushCommitLogLeastPages();

                //最大刷盘间隔
                int flushPhysicQueueThoroughInterval =
                    CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushCommitLogThoroughInterval();

                boolean printFlushProgress = false;

                // Print flush progress
                long currentTimeMillis = System.currentTimeMillis();
                //当前时间大于等于最后一次刷盘的时间加上最大刷盘间隔
                if (currentTimeMillis >= (this.lastFlushTimestamp + flushPhysicQueueThoroughInterval)) {
                    //刷盘时间设置为最后一次刷盘时间
                    this.lastFlushTimestamp = currentTimeMillis;
                    //刷盘最少页数设置为0
                    flushPhysicQueueLeastPages = 0;
                    printFlushProgress = (printTimes++ % 10) == 0;
                }

                try {
                    //休眠
                    if (flushCommitLogTimed) {
                        Thread.sleep(interval);
                    } else {
                        this.waitForRunning(interval);
                    }

                    if (printFlushProgress) {
                        this.printFlushProgress();
                    }

                    //刷盘开始时间
                    long begin = System.currentTimeMillis();
                    //刷盘
                    CommitLog.this.mappedFileQueue.flush(flushPhysicQueueLeastPages);
                    long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
                    //设置检查点的写入时间
                    if (storeTimestamp > 0) {
                        CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
                    }
                    //刷盘花费时间
                    long past = System.currentTimeMillis() - begin;
                    if (past > 500) {
                        log.info("Flush data to disk costs {} ms", past);
                    }
                } catch (Throwable e) {
                    CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
                    this.printFlushProgress();
                }
            }

            // Normal shutdown, to ensure that all the flush before exit
            //关闭刷盘服务之前，首先进行刷盘，防止消息丢失
            boolean result = false;
            for (int i = 0; i < RETRY_TIMES_OVER && !result; i++) {
                result = CommitLog.this.mappedFileQueue.flush(0);
                CommitLog.log.info(this.getServiceName() + " service shutdown, retry " + (i + 1) + " times " + (result ? "OK" : "Not OK"));
            }

            this.printFlushProgress();

            CommitLog.log.info(this.getServiceName() + " service end");
}
```

run方法首先获取flushCommitLogTimed（是否固定时间周期进行刷盘）、interval（刷盘的间隔）、flushPhysicQueueLeastPages（刷盘的最少页数）、flushPhysicQueueThoroughInterval（最大刷盘间隔）这些属性值，然后判断当前时间是否大于等于最后一次刷盘的时间加上最大刷盘间隔，如果是，则设置最后一次刷盘时间以及刷盘最少页数，在该条件下，必须进行一次刷盘操作。

接着根据flushCommitLogTimed判断休眠的方式，如果flushCommitLogTimed为true，则进行线程休眠，否则等待休眠。当休眠完以后，调用mappedFileQueue.flush方法进行刷盘，同步刷盘也是调用mappedFileQueue.flush方法进行刷盘。最后记录检查点的写入时间以及打印刷盘花费时间。

当刷盘服务被关闭之前，就是run方法中while循环不再进行时，要做最后的刷盘操作，防止消息尽量少丢失，默认刷盘次数为10次（RETRY_TIMES_OVER）。

### **MappedFile的刷盘**

不管是同步刷盘还是异步刷盘，最终都是调用mappedFileQueue.flush方法进行刷盘，mappedFileQueue.flush方法如下：

```java
//代码位置：org.apache.rocketmq.store.MappedFileQueue#flush
public boolean flush(final int flushLeastPages) {
        boolean result = true;
        //根据刷盘位移获取mappedFile
        MappedFile mappedFile = this.findMappedFileByOffset(this.flushedWhere, this.flushedWhere == 0);
        if (mappedFile != null) {
            long tmpTimeStamp = mappedFile.getStoreTimestamp();
            //刷盘
            int offset = mappedFile.flush(flushLeastPages);
              //刷盘以后的位置
            long where = mappedFile.getFileFromOffset() + offset;
            //刷盘是否成功
            result = where == this.flushedWhere;
            this.flushedWhere = where;
            if (0 == flushLeastPages) {
                this.storeTimestamp = tmpTimeStamp;
            }
        }

        return result;
}
```

flush方法首先根据刷盘位移获取mappedFile，要知道将消息刷盘到哪一个MappedFile文件中，findMappedFileByOffset方法找到本次刷盘起始点的MappedFile文件。当获取的mappedFile文件不为null，则调用mappedFile.flush方法进行刷盘，刷盘盘以后更新刷盘以后的偏移量。我们接下来看看MappedFile文件是否进行刷盘的：

```java
//代码位置：org.apache.rocketmq.store.MappedFile#flush
public int flush(final int flushLeastPages) {

        //如果可以刷盘
        if (this.isAbleToFlush(flushLeastPages)) {
            if (this.hold()) {
                int value = getReadPosition();

                try {
                    //We only append data to fileChannel or mappedByteBuffer, never both.
                    //如果writeBuffer不为空，则表明消息是先提交到writeBuffer中，已经从writeBuffer提交到fileChannel，直接调用fileChannel.force（）
                    if (writeBuffer != null || this.fileChannel.position() != 0) {
                        this.fileChannel.force(false);
                    } else {
                        //消息是直接存储在文件内存映射缓冲区mappedByteBuffer中，直接调用它的force（）即可
                        this.mappedByteBuffer.force();
                    }
                } catch (Throwable e) {
                    log.error("Error occurred when force data to disk.", e);
                }

                //设置刷盘的偏移量
                this.flushedPosition.set(value);
                this.release();
            } else {
                log.warn("in flush, hold failed, flush offset = " + this.flushedPosition.get());
                this.flushedPosition.set(getReadPosition());
            }
        }
        //返回刷盘偏移量
        return this.getFlushedPosition();
}
```

flush方法首先判断用isAbleToFlush方法判断是否可以刷盘，当文件没有写满、刷盘内容的长度小于文件剩余的长度、如果满足就可以进行刷盘。

如果writeBuffer不为空，则表明消息是先提交到writeBuffer中，已经从writeBuffer提交到fileChannel，直接调用fileChannel.force方法将消息刷盘。否则消息是直接存储在文件内存映射缓冲区mappedByteBuffer中，直接调用它的force方法进行刷盘。最后设置下刷盘的偏移量以及返回刷盘的偏移量。

### CommitRealTimeService的作用

分析完同步刷盘和异步刷盘，接下来分析来CommitRealTimeService的作用，我们直接来看看CommitRealTimeService的run方法：

```java
//代码位置：org.apache.rocketmq.store.CommitLog.CommitRealTimeService#run
public void run() {
            CommitLog.log.info(this.getServiceName() + " service started");
            while (!this.isStopped()) {
                //提交间隔
                int interval = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitIntervalCommitLog();

                //最少一次提交的页数
                int commitDataLeastPages = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitCommitLogLeastPages();

                //最大的提交间隔
                int commitDataThoroughInterval =
                    CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitCommitLogThoroughInterval();

                //提交开始时间
                long begin = System.currentTimeMillis();
                //提交开始时间大于等最后一次提交时间加上最大的提交间隔
                if (begin >= (this.lastCommitTimestamp + commitDataThoroughInterval)) {
                    this.lastCommitTimestamp = begin;
                    commitDataLeastPages = 0;
                }

                try {
                    //提交
                    boolean result = CommitLog.this.mappedFileQueue.commit(commitDataLeastPages);
                    long end = System.currentTimeMillis();
                    if (!result) {
                        this.lastCommitTimestamp = end; // result = false means some data committed.
                        //now wake up flush thread.
                        flushCommitLogService.wakeup();
                    }

                    //提交花费时间大于500毫秒就打印日志
                    if (end - begin > 500) {
                        log.info("Commit data to file costs {} ms", end - begin);
                    }
                    //等待interval
                    this.waitForRunning(interval);
                } catch (Throwable e) {
                    CommitLog.log.error(this.getServiceName() + " service has exception. ", e);
                }
            }

            //提交服务停止后，需要刷盘，防止数据丢失
            boolean result = false;
            for (int i = 0; i < RETRY_TIMES_OVER && !result; i++) {
                result = CommitLog.this.mappedFileQueue.commit(0);
                CommitLog.log.info(this.getServiceName() + " service shutdown, retry " + (i + 1) + " times " + (result ? "OK" : "Not OK"));
            }
            CommitLog.log.info(this.getServiceName() + " service end");
        }
}
```

CommitRealTimeService的run方法有个while循环，当CommitRealTimeService启动以后没有停止，while循环就一直执行。while循环里面首先获取一些变量的值，如interval（提交间隔）、commitDataLeastPages（最少一次提交的页数）、commitDataThoroughInterval（最大的提交间隔）。当提交开始时间大于等最后一次提交时间加上最大的提交间隔时，设置最后提交的时间以及最少提交的页数为0，说明这时候最少也要提交一次了。然后调用mappedFileQueue.commit方法进行提交，提交完以后就等到interval毫秒以后再循环上述的流程。当CommitRealTimeService被停止时，默认进行十次刷盘，最大限度防止数据丢失。

接下来我们继续深入mappedFileQueue.commit方法：

```java
//代码位置：org.apache.rocketmq.store.MappedFileQueue#commit
public boolean commit(final int commitLeastPages) {
        boolean result = true;
        //查找提交的mappedFile
        MappedFile mappedFile = this.findMappedFileByOffset(this.committedWhere, this.committedWhere == 0);
        if (mappedFile != null) {
            //提交
            int offset = mappedFile.commit(commitLeastPages);
            //提交偏移量
            long where = mappedFile.getFileFromOffset() + offset;
            result = where == this.committedWhere;
            //提交的偏移量
            this.committedWhere = where;
        }

        return result;
}
```

mappedFileQueue.commit查找MappedFile，才能知道哪个MappedFile需要提交，当查找的mappedFile不等于null时，通过mappedFile.commit方法提交，提交完以后设置提交的偏移量。

mappedFile.commit方法首先就是判断是否满足提交的条件，如果满足的话，将缓存writeBuffer的数据提交到fileChannel，然后当所有的缓存writeBuffer的所有数据都提交了，就清空缓存writeBuffer。mappedFile.commit的代码就不展示，想了解的可以深入理解下。

### 总结

RocketMQ有同步刷盘与异步刷盘的方法，可以根据场景进行选择刷盘的方式，同步刷盘对性能影响比较大，但是可靠性比较高，因为要等到消息刷盘以后才给生产者返回确认消息；异步输盘方式只要写入到缓存以后就给生产者返回确认消息，同时采用后台异步的方式进行刷盘，提高了性能以及吞吐量。