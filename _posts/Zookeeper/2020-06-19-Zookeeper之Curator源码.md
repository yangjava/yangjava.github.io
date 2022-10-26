
---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---

## Curator框架源码

Curator是Netflix公司开源的一套Zookeeper客户端框架。目前已经作为Apache的顶级项目出现，是最流行的Zookeeper客户端之一。

接着看下quick start中关于分布式锁相关的内容
地址为：http://curator.apache.org/getting-started.html

```
InterProcessMutex lock = new InterProcessMutex(client, lockPath);
if ( lock.acquire(maxWait, waitUnit) ) 
{
    try 
    {
        // do some work inside of the critical section here
    }
    finally
    {
        lock.release();
    }
}
```

使用很简单，使用`InterProcessMutex`类，使用其中的`acquire()`方法，就可以获取一个分布式锁了。

### 使用示例

启动两个线程t1和t2去争夺锁，拿到锁的线程会占用5秒。运行多次可以观察到，有时是t1先拿到锁而t2等待，有时又会反过来。Curator会用我们提供的lock路径的结点作为全局锁，这个结点的数据类似这种格式：**[_c_64e0811f-9475-44ca-aa36-c1db65ae5350-lock-00000000001]**，每次获得锁时会生成这种串，释放锁时清空数据。

接下来看看加锁的示例：

```
public class Application {
    private static final String ZK_ADDRESS = "192.20.38.58:2181";
    private static final String ZK_LOCK_PATH = "/locks/lock_01";

    public static void main(String[] args) throws InterruptedException {
        CuratorFramework client = CuratorFrameworkFactory.newClient(
                ZK_ADDRESS,
                new RetryNTimes(10, 5000)
        );
        client.start();
        System.out.println("zk client start successfully!");

        Thread t1 = new Thread(() -> {
            doWithLock(client);
        }, "t1");
        Thread t2 = new Thread(() -> {
            doWithLock(client);
        }, "t2");

        t1.start();
        t2.start();
    }

    private static void doWithLock(CuratorFramework client) {
        InterProcessMutex lock = new InterProcessMutex(client, ZK_LOCK_PATH);
        try {
            if (lock.acquire(10 * 1000, TimeUnit.SECONDS)) {
                System.out.println(Thread.currentThread().getName() + " hold lock");
                Thread.sleep(5000L);
                System.out.println(Thread.currentThread().getName() + " release lock");
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                lock.release();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

### Curator 加锁实现原理

直接看Curator加锁的代码：

```
public class InterProcessMutex implements InterProcessLock, Revocable<InterProcessMutex> {

    private final ConcurrentMap<Thread, LockData>   threadData = Maps.newConcurrentMap();

     private static class LockData
    {
        final Thread        owningThread;
        final String        lockPath;
        final AtomicInteger lockCount = new AtomicInteger(1);

        private LockData(Thread owningThread, String lockPath)
        {
            this.owningThread = owningThread;
            this.lockPath = lockPath;
        }
    }

    @Override
    public boolean acquire(long time, TimeUnit unit) throws Exception
    {
        return internalLock(time, unit);
    }


     private boolean internalLock(long time, TimeUnit unit) throws Exception
    {
        /*
           Note on concurrency: a given lockData instance
           can be only acted on by a single thread so locking isn't necessary
        */

        Thread          currentThread = Thread.currentThread();

        LockData        lockData = threadData.get(currentThread);
        if ( lockData != null )
        {
            // re-entering
            lockData.lockCount.incrementAndGet();
            return true;
        }

        String lockPath = internals.attemptLock(time, unit, getLockNodeBytes());
        if ( lockPath != null )
        {
            LockData        newLockData = new LockData(currentThread, lockPath);
            threadData.put(currentThread, newLockData);
            return true;
        }

        return false;
    }   
}
```

直接看`internalLock()`方法，首先是获取当前线程，然后查看当前线程是否在一个concurrentHashMap中，这里是`重入锁`的实现，如果当前已经已经获取了锁，那么这个线程获取锁的次数再+1

如果没有获取锁，那么就是用`attemptLock()`方法去尝试获取锁，如果`lockPath`不为空，说明获取锁成功，并将当前线程放入到map中。

接下来看看核心的加锁逻辑`attemptLock()`方法：

入参：
`time` : 获取锁等待的时间
`unit`：时间单位
`lockNodeBytes`：默认为null

```
public class LockInternals {    
    String attemptLock(long time, TimeUnit unit, byte[] lockNodeBytes) throws Exception
    {
        final long      startMillis = System.currentTimeMillis();
        final Long      millisToWait = (unit != null) ? unit.toMillis(time) : null;
        final byte[]    localLockNodeBytes = (revocable.get() != null) ? new byte[0] : lockNodeBytes;
        int             retryCount = 0;

        String          ourPath = null;
        boolean         hasTheLock = false;
        boolean         isDone = false;
        while ( !isDone )
        {
            isDone = true;

            try
            {
                if ( localLockNodeBytes != null )
                {
                    ourPath = client.create().creatingParentsIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path, localLockNodeBytes);
                }
                else
                {
                    ourPath = client.create().creatingParentsIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path);
                }
                hasTheLock = internalLockLoop(startMillis, millisToWait, ourPath);
            }
            catch ( KeeperException.NoNodeException e )
            {
                // gets thrown by StandardLockInternalsDriver when it can't find the lock node
                // this can happen when the session expires, etc. So, if the retry allows, just try it all again
                if ( client.getZookeeperClient().getRetryPolicy().allowRetry(retryCount++, System.currentTimeMillis() - startMillis, RetryLoop.getDefaultRetrySleeper()) )
                {
                    isDone = false;
                }
                else
                {
                    throw e;
                }
            }
        }

        if ( hasTheLock )
        {
            return ourPath;
        }

        return null;
    }
}
```



```
ourPath = client.create().creatingParentsIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path);
```

使用的临时顺序节点，首先他是临时节点，如果当前这台机器如果自己宕机的话，他创建的这个临时节点就会自动消失，如果有获取锁的客户端宕机了，zk可以保证锁会自动释放的

