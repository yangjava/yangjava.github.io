---
layout: post
categories: [Zookeeper]
description: none
keywords: Zookeeper
---
# Zookeeper实战

## 数据发布/订阅

### 基本概念

（1）**数据发布/订阅系统即所谓的配置中心，也就是发布者将数据发布到ZooKeeper的一个节点或者一系列节点上，提供订阅者进行数据订阅，从而实现动态更新数据的目的，实现配置信息的集中式管理和数据的动态更新。**ZooKeeper采用的是推拉相结合的方式：客户端向服务器注册自己需要关注的节点，一旦该节点的数据发生改变，那么服务端就会向相应的客户端发送Wacher事件通知，客户端接收到消息通知后，需要主动到服务端获取最新的数据。

（2）实际系统开发过程中：**我们可以将初始化配置信息放到节点上集中管理，应用在启动时都会主动到ZooKeeper服务端进行一次配置读取，同时在指定节点注册Watcher监听，主要配置信息一旦变更，订阅者就可以获取读取最新的配置信息。**通常系统中需要使用一些通用的配置信息，比如机器列表信息、运行时的开关配置、数据库配置信息等全局配置信息，这些都会有以下3点特性：

1） 数据量通常比较小（通常是一些配置文件）

2） 数据内容在运行时会经常发生动态变化（比如数据库的临时切换等）

3） 集群中各机器共享，配置一致（比如数据库配置共享）。

（3）利用的ZooKeeper特性是：**ZooKeeper对任何节点（包括子节点）的变更，只要注册Wacther事件（使用Curator等客户端工具已经被封装好）都可以被其它客户端监听**

### 代码示例

```

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.cache.NodeCache;
import org.apache.curator.framework.recipes.cache.NodeCacheListener;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;

import java.util.concurrent.CountDownLatch;

public class ZooKeeper_Subsciption {
    private static final String ADDRESS = "xxx.xxx.xxx.xxx:2181";
    private static final int SESSION_TIMEOUT = 5000;
    private static final String PATH = "/configs";
    private static RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    private static String config = "jdbc_configuration";
    private static CountDownLatch countDownLatch = new CountDownLatch(4);

    public static void main(String[] args) throws Exception {
        // 订阅该配置信息的集群节点（客户端）:sub1-sub3
        for (int i = 0; i < 3; i++) {
            CuratorFramework consumerClient = getClient();
            subscribe(consumerClient, "sub" + String.valueOf(i));
        }
        // 更改配置信息的集群节点（客户端）:pub
        CuratorFramework publisherClient = getClient();
        publish(publisherClient, "pub");

    }
    private static void init() throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString(ADDRESS)
                .sessionTimeoutMs(SESSION_TIMEOUT)
                .retryPolicy(retryPolicy)
                .build();
        client.start();
        // 检查节点是否存在，不存在则初始化创建
        if (client.checkExists().forPath(PATH) == null) {
            client.create()
                    .creatingParentsIfNeeded()
                    .withMode(CreateMode.EPHEMERAL)
                    .forPath(PATH, config.getBytes());
        }
    }


    /**
     * 创建客户端并且初始化建立一个存储配置数据的节点
     *
     * @return
     * @throws Exception
     */
    private static CuratorFramework getClient() throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString(ADDRESS)
                .sessionTimeoutMs(SESSION_TIMEOUT)
                .retryPolicy(retryPolicy)
                .build();
        client.start();
        if (client.checkExists().forPath(PATH) == null) {
            client.create()
                    .creatingParentsIfNeeded()
                    .withMode(CreateMode.EPHEMERAL)
                    .forPath(PATH, config.getBytes());
        }
        return client;
    }

    /**
     * 集群中的某个节点机器更改了配置信息：即发布了更新了数据
     *
     * @param client
     * @throws Exception
     */
    private static void publish(CuratorFramework client, String znode) throws Exception {

        System.out.println("节点[" + znode + "]更改了配置数据...");
        client.setData().forPath(PATH, "configuration".getBytes());
        countDownLatch.await();
    }

    /**
     * 集群中订阅的节点客户端（机器）获得最新的配置数据
     *
     * @param client
     * @param znode
     * @throws Exception
     */
    private static void subscribe(CuratorFramework client, String znode) throws Exception {
        // NodeCache监听ZooKeeper数据节点本身的变化
        final NodeCache cache = new NodeCache(client, PATH);
        // 设置为true：NodeCache在第一次启动的时候就立刻从ZooKeeper上读取节点数据并保存到Cache中
        cache.start(true);
        System.out.println("节点["+ znode +"]已订阅当前配置数据：" + new String(cache.getCurrentData().getData()));
        // 节点监听
        countDownLatch.countDown();
        cache.getListenable().addListener(new NodeCacheListener() {
            @Override
            public void nodeChanged() {
                System.out.println("配置数据已发生改变, 节点[" + znode + "]读取当前新配置数据: " + new String(cache.getCurrentData().getData()));
            }
        });
    }
}
```

## Master选举

### 基本概念

（1）在一些读写分离的应用场景中，客户端写请求往往是由Master处理的，而另一些场景中，Master则常常负责处理一些复杂的逻辑，并将处理结果同步给集群中其它系统单元。比如一个广告投放系统后台与ZooKeeper交互，广告ID通常都是经过一系列海量数据处理中计算得到（非常消耗I/O和CPU资源的过程），那就可以只让集群中一台机器处理数据得到计算结果，之后就可以共享给整个集群中的其它所有客户端机器。

（2）利用ZooKeeper的特性：**利用ZooKeeper的强一致性，即能够很好地保证分布式高并发情况下节点的创建一定能够保证全局唯一性，ZooKeeper将会保证客户端无法重复创建一个已经存在的数据节点，也就是说如果多个客户端请求创建同一个节点，那么最终一定只有一个客户端请求能够创建成功，这个客户端就是Master，而其它客户端注在该节点上注册子节点Wacther，用于监控当前Master是否存活，如果当前Master挂了，那么其余客户端立马重新进行Master选举。**

（3）竞争成为Master角色之后，创建的子节点都是临时顺序节点，比如：_c_862cf0ce-6712-4aef-a91d-fc4c1044d104-lock-0000000001，并且序号是递增的。**需要注意的是这里有"lock"单词，这说明ZooKeeper这一特性，也可以运用于分布式锁。**

### 代码示例

 ```


import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.leader.LeaderSelector;
import org.apache.curator.framework.recipes.leader.LeaderSelectorListenerAdapter;
import org.apache.curator.retry.ExponentialBackoffRetry;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;

public class ZooKeeper_Master {

    private static final String ADDRESS="xxx.xxx.xxx.xxx:2181";
    private static final int SESSION_TIMEOUT=5000;
    private static final String MASTER_PATH = "/master_path";
    private static final int CLIENT_COUNT = 5;

    private static RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);


    public static void main(String[] args) throws InterruptedException {

        ExecutorService service = Executors.newFixedThreadPool(CLIENT_COUNT);
        for (int i = 0; i < CLIENT_COUNT; i++) {
            final String index = String.valueOf(i);
            service.submit(() -> {
                masterSelect(index);
            });
        }
    }

    private static void  masterSelect(final String znode){
        // client成为master的次数统计
        AtomicInteger leaderCount = new AtomicInteger(1);
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString(ADDRESS)
                .sessionTimeoutMs(SESSION_TIMEOUT)
                .retryPolicy(retryPolicy)
                .build();
        client.start();
        // 一旦执行完takeLeadership，就会重新进行选举
        LeaderSelector selector = new LeaderSelector(client, MASTER_PATH, new LeaderSelectorListenerAdapter() {
            @Override
            public void takeLeadership(CuratorFramework curatorFramework) throws Exception {
                System.out.println("节点["+ znode +"]成为master");
                System.out.println("节点["+ znode +"]已经成为master次数："+ leaderCount.getAndIncrement());
                // 睡眠5s模拟成为master后完成任务
                Thread.sleep(5000);
                System.out.println("节点["+ znode +"]释放master");
            }
        });
        // autoRequeue自动重新排队：使得上一次选举为master的节点还有可能再次成为master
        selector.autoRequeue();
        selector.start();
    }
}
 ```

## 分布式锁

### 基本概念

（1）对于排他锁：**ZooKeeper通过数据节点表示一个锁，例如/exclusive_lock/lock节点就可以定义一个锁，所有客户端都会调用create()接口，试图在/exclusive_lock下创建lock子节点，但是ZooKeeper的强一致性会保证所有客户端最终只有一个客户创建成功。也就可以认为获得了锁，其它线程Watcher监听子节点变化（等待释放锁，竞争获取资源）。**

（2）对于共享锁：ZooKeeper同样可以通过数据节点表示一个锁，类似于/shared_lock/[Hostname]-请求类型（读/写）-序号的临时节点，比如/shared_lock/192.168.0.1-R-0000000000

### 排他锁（X）

这里主要讲讲分布式锁中的排他锁。排他锁（Exclusive Locks，简称X锁），又称为写锁或独占锁，是一种基本的锁类型。如果事务T1对数据对象O1加上了排他锁，那么在整个加锁期间，只允许T1对O1进行数据的读取和更新操作，其它任何事务都不能对O1进行任何类型的操作，直道T1释放了排他锁。

#### 定义锁

在ZooKeeper中，可以通过在ZooKeeper中创建一个数据节点来表示一个锁。比如，/exclusive_lock/lock节点（znode）就可以表示为一个锁。

#### 获取锁

在需要获取排他锁时，所有的客户端都会试图通过create()接口，在/exclusive_lock节点下创建临时的子节点/exclusive_lock/lock，但ZooKeeper的强一致性最终只会保证仅有一个客户端能创建成功，那么就认为该客户端获取了锁。同时，所有没有获取锁的客户端事务只能处于等待状态，这些处于等待状态的客户端事先可以在/exclusive_lock节点上注册一个子节点变更的Watcher监听，以便实时监听到子节点的变更情况。

#### 释放锁

在“定义锁”部分，我们已经提到/exclusive_lock/lock是一个临时节点，因此在以下两种情况下可能释放锁。

- 当前获取锁的客户端发生宕机，那么ZooKeeper服务器上保存的临时性节点就会被删除；
- 正常执行完业务逻辑后，由客户端主动来将自己创建的临时节点删除。

无论什么情况下，临时节点/exclusive_lock/lock被移除，ZooKeeper都会通知在/exclusive_lock注册了子节点变更Watcher监听的客户端。这些客户端在接收到通知以后就会再次发起获取锁的操作，即重复“获取锁”过程。

主要流程图如下：

- 查看目标Node是否已经创建，已经创建，那么等待锁。
- 如果未创建，创建一个瞬时Node，表示已经占有锁。
- 如果创建失败，那么证明锁已经被其他线程占有了，那么同样等待锁。
- 当释放锁，或者当前Session超时的时候，节点被删除，唤醒之前等待锁的线程去争抢锁。

其实上面的实现有优点也有缺点：
优点：
实现比较简单，有通知机制，能提供较快的响应，有点类似reentrantlock的思想，对于节点删除失败的场景由Session超时保证节点能够删除掉。
缺点：
重量级，同时在大量锁的情况下会有“惊群”的问题。

“惊群”就是在一个节点删除的时候，大量对这个节点的删除动作有订阅Watcher的线程会进行回调，这对Zk集群是十分不利的。所以需要避免这种现象的发生。

解决“惊群”：

为了解决“惊群“问题，我们需要放弃订阅一个节点的策略，那么怎么做呢？

1. 我们将锁抽象成目录，多个线程在此目录下创建瞬时的顺序节点，因为Zk会为我们保证节点的顺序性，所以可以利用节点的顺序进行锁的判断。
2. 首先创建顺序节点，然后获取当前目录下最小的节点，判断最小节点是不是当前节点，如果是那么获取锁成功，如果不是那么获取锁失败。
3. 获取锁失败的节点获取当前节点上一个顺序节点，对此节点注册监听，当节点删除的时候通知当前节点。
4. 当unlock的时候删除节点之后会通知下一个节点。

### 代码示例

Curator提供的有四种锁，分别如下：

（1）InterProcessMutex：分布式可重入排它锁

（2）InterProcessSemaphoreMutex：分布式排它锁

（3）InterProcessReadWriteLock：分布式读写锁

（4）InterProcessMultiLock：将多个锁作为单个实体管理的容器

​                  InterProcessSemaphoreV2 信号量

主要是以InterProcessMutex为例，编写示例：

```


import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ZooKeeper_Lock {
    private static final String ADDRESS = "xxx.xxx.xxx.xxx:2181";
    private static final int SESSION_TIMEOUT = 5000;
    private static final String LOCK_PATH = "/lock_path";
    private static final int CLIENT_COUNT = 10;

    private static RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    private static int resource = 0;

    public static void main(String[] args){
        ExecutorService service = Executors.newFixedThreadPool(CLIENT_COUNT);
        for (int i = 0; i < CLIENT_COUNT; i++) {
            final String index = String.valueOf(i);
            service.submit(() -> {
                distributedLock(index);
            });
        }
    }

    private static void distributedLock(final String znode) {
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString(ADDRESS)
                .sessionTimeoutMs(SESSION_TIMEOUT)
                .retryPolicy(retryPolicy)
                .build();
        client.start();
        final InterProcessMutex lock = new InterProcessMutex(client, LOCK_PATH);
        try {
//            lock.acquire();
            System.out.println("客户端节点[" + znode + "]获取lock");
            System.out.println("客户端节点[" + znode + "]读取的资源为：" + String.valueOf(resource));
            resource ++;
//            lock.release();
            System.out.println("客户端节点[" + znode + "]释放lock");

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### 分布式可重入排它锁(InterProcessMutex)

此锁可以重入，但是重入几次需要释放几次。

```
@Test
    public void sharedReentrantLock() throws Exception {
        // 创建共享锁
        final InterProcessLock lock = new InterProcessMutex(client, lockPath);
        // lock2 用于模拟其他客户端
        final InterProcessLock lock2 = new InterProcessMutex(client2, lockPath);

        final CountDownLatch countDownLatch = new CountDownLatch(2);

        new Thread(new Runnable() {
            @Override
            public void run() {
                // 获取锁对象
                try {
                    lock.acquire();
                    System.out.println("1获取锁===============");
                    // 测试锁重入
                    lock.acquire();
                    System.out.println("1再次获取锁===============");
                    Thread.sleep(5 * 1000);
                    lock.release();
                    System.out.println("1释放锁===============");
                    lock.release();
                    System.out.println("1再次释放锁===============");

                    countDownLatch.countDown();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                // 获取锁对象
                try {
                    lock2.acquire();
                    System.out.println("2获取锁===============");
                    // 测试锁重入
                    lock2.acquire();
                    System.out.println("2再次获取锁===============");
                    Thread.sleep(5 * 1000);
                    lock2.release();
                    System.out.println("2释放锁===============");
                    lock2.release();
                    System.out.println("2再次释放锁===============");

                    countDownLatch.countDown();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        countDownLatch.await();
    }
```

**原理:**

InterProcessMutex通过在zookeeper的某路径节点下创建临时序列节点来实现分布式锁，即每个线程（跨进程的线程）获取同一把锁前，都需要在同样的路径下创建一个节点，节点名字由uuid + 递增序列组成。而通过对比自身的序列数是否在所有子节点的第一位，来判断是否成功获取到了锁。当获取锁失败时，它会添加watcher来监听前一个节点的变动情况，然后进行等待状态。直到watcher的事件生效将自己唤醒，或者超时时间异常返回。

#### 分布式排它锁(InterProcessSemaphoreMutex)

InterProcessSemaphoreMutex是一种不可重入的互斥锁，也就意味着即使是同一个线程也无法在持有锁的情况下再次获得锁，所以需要注意，不可重入的锁很容易在一些情况导致死锁。

```
@Test
    public void sharedLock() throws Exception {
        // 创建共享锁
        final InterProcessLock lock = new InterProcessSemaphoreMutex(client, lockPath);
        // lock2 用于模拟其他客户端
        final InterProcessLock lock2 = new InterProcessSemaphoreMutex(client2, lockPath);

        new Thread(new Runnable() {
            @Override
            public void run() {
                // 获取锁对象
                try {
                    lock.acquire();
                    System.out.println("1获取锁===============");
                    // 测试锁重入
                    Thread.sleep(5 * 1000);
                    lock.release();
                    System.out.println("1释放锁===============");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                // 获取锁对象
                try {
                    lock2.acquire();
                    System.out.println("2获取锁===============");
                    Thread.sleep(5 * 1000);
                    lock2.release();
                    System.out.println("2释放锁===============");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        Thread.sleep(20 * 1000);
    }
```

#### 分布式读写锁(InterProcessReadWriteLock)

读锁和读锁不互斥，只要有写锁就互斥。

```
@Test
    public void sharedReentrantReadWriteLock() throws Exception {
        // 创建共享可重入读写锁
        final InterProcessReadWriteLock locl1 = new InterProcessReadWriteLock(client, lockPath);
        // lock2 用于模拟其他客户端
        final InterProcessReadWriteLock lock2 = new InterProcessReadWriteLock(client2, lockPath);

        // 获取读写锁(使用 InterProcessMutex 实现, 所以是可以重入的)
        final InterProcessLock readLock = locl1.readLock();
        final InterProcessLock readLockw = lock2.readLock();

        final CountDownLatch countDownLatch = new CountDownLatch(2);

        new Thread(new Runnable() {
            @Override
            public void run() {
                // 获取锁对象
                try {
                    readLock.acquire();
                    System.out.println("1获取读锁===============");
                    // 测试锁重入
                    readLock.acquire();
                    System.out.println("1再次获取读锁===============");
                    Thread.sleep(5 * 1000);
                    readLock.release();
                    System.out.println("1释放读锁===============");
                    readLock.release();
                    System.out.println("1再次释放读锁===============");

                    countDownLatch.countDown();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                // 获取锁对象
                try {
                    Thread.sleep(500);
                    readLockw.acquire();
                    System.out.println("2获取读锁===============");
                    // 测试锁重入
                    readLockw.acquire();
                    System.out.println("2再次获取读锁==============");
                    Thread.sleep(5 * 1000);
                    readLockw.release();
                    System.out.println("2释放读锁===============");
                    readLockw.release();
                    System.out.println("2再次释放读锁===============");

                    countDownLatch.countDown();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        countDownLatch.await();
    }
```

#### 共享信号量（InterProcessSemaphoreV2）

```
@Test
    public void semaphore() throws Exception {
        // 创建一个信号量, Curator 以公平锁的方式进行实现
        final InterProcessSemaphoreV2 semaphore = new InterProcessSemaphoreV2(client, lockPath, 1);

        final CountDownLatch countDownLatch = new CountDownLatch(2);

        new Thread(new Runnable() {
            @Override
            public void run() {
                // 获取锁对象
                try {
                    // 获取一个许可
                    Lease lease = semaphore.acquire();
                    logger.info("1获取读信号量===============");
                    Thread.sleep(5 * 1000);
                    semaphore.returnLease(lease);
                    logger.info("1释放读信号量===============");

                    countDownLatch.countDown();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                // 获取锁对象
                try {
                    // 获取一个许可
                    Lease lease = semaphore.acquire();
                    logger.info("2获取读信号量===============");
                    Thread.sleep(5 * 1000);
                    semaphore.returnLease(lease);
                    logger.info("2释放读信号量===============");

                    countDownLatch.countDown();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        countDownLatch.await();
    }
```

**当然可以一次获取多个信号量:**

```
@Test
    public void semaphore() throws Exception {
        // 创建一个信号量, Curator 以公平锁的方式进行实现
        final InterProcessSemaphoreV2 semaphore = new InterProcessSemaphoreV2(client, lockPath, 3);

        final CountDownLatch countDownLatch = new CountDownLatch(2);

        new Thread(new Runnable() {
            @Override
            public void run() {
                // 获取锁对象
                try {
                    // 获取2个许可
                    Collection<Lease> acquire = semaphore.acquire(2);
                    logger.info("1获取读信号量===============");
                    Thread.sleep(5 * 1000);
                    semaphore.returnAll(acquire);
                    logger.info("1释放读信号量===============");

                    countDownLatch.countDown();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                // 获取锁对象
                try {
                    // 获取1个许可
                    Collection<Lease> acquire = semaphore.acquire(1);
                    logger.info("2获取读信号量===============");
                    Thread.sleep(5 * 1000);
                    semaphore.returnAll(acquire);
                    logger.info("2释放读信号量===============");

                    countDownLatch.countDown();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        countDownLatch.await();
    }
```

#### 多重共享锁（InterProcessMultiLock）

```
@Test
    public void multiLock() throws Exception {
        // 可重入锁
        final InterProcessLock interProcessLock1 = new InterProcessMutex(client, lockPath);
        // 不可重入锁
        final InterProcessLock interProcessLock2 = new InterProcessSemaphoreMutex(client2, lockPath);
        // 创建多重锁对象
        final InterProcessLock lock = new InterProcessMultiLock(Arrays.asList(interProcessLock1, interProcessLock2));

        final CountDownLatch countDownLatch = new CountDownLatch(1);

        new Thread(new Runnable() {
            @Override
            public void run() {
                // 获取锁对象
                try {
                    // 获取参数集合中的所有锁
                    lock.acquire();
                    // 因为存在一个不可重入锁, 所以整个 InterProcessMultiLock 不可重入
                    System.out.println(lock.acquire(2, TimeUnit.SECONDS));
                    // interProcessLock1 是可重入锁, 所以可以继续获取锁
                    System.out.println(interProcessLock1.acquire(2, TimeUnit.SECONDS));
                    // interProcessLock2 是不可重入锁, 所以获取锁失败
                    System.out.println(interProcessLock2.acquire(2, TimeUnit.SECONDS));

                    countDownLatch.countDown();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        countDownLatch.await();
    }
```

## 命名服务

在分布式系统中，通常需要一个全局唯一的名字，如生成全局唯一的订单号等，ZooKeeper 可以通过顺序节点的特性来生成全局唯一 ID，从而可以对分布式系统提供命名服务。

```
   private String createSeqNode(String pathPefix) {
      try {
            // 创建一个 ZNode 顺序节点
            String destPath = client.create()
                    .creatingParentsIfNeeded()
                    .withMode(CreateMode.*EPHEMERAL_SEQUENTIAL*)
//避免zookeeper的顺序节点暴增，可以删除创建的顺序节点
                    .forPath(pathPefix);
            return destPath;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
```

节点创建完成后，会返回节点的完整的层次路径，生成的序号，放置在路径的末尾。一般为10位数字字符。

通过截取路径末尾的数字，作为新生成的ID。截取数字的代码如下：

```
public String makeId(String nodeName) {
    String str = createSeqNode(nodeName);
    if (null == str) {
        return null;
    }
    int index = str.lastIndexOf(nodeName);
    if (index >= 0) {
        index += nodeName.length();
        return index <= str.length() ? str.substring(index) : "";
    }
    return str;
}
```

## 集群管理

ZooKeeper 还能解决大多数分布式系统中的问题：

- 如可以通过创建临时节点来建立心跳检测机制。如果分布式系统的某个服务节点宕机了，则其持有的会话会超时，此时该临时节点会被删除，相应的监听事件就会被触发。
- 分布式系统的每个服务节点还可以将自己的节点状态写入临时节点，从而完成状态报告或节点工作进度汇报。
- 通过数据的订阅和发布功能，ZooKeeper 还能对分布式系统进行模块的解耦和任务的调度。
- 通过监听机制，还能对分布式系统的服务节点进行动态上下线，从而实现服务的动态扩容。

## 队列管理

ZooKeeper 可以处理两种类型的队列：

- 当一个队列的成员都聚齐时，这个队列才可用，否则一直等待所有成员到达，这种是同步队列。，在约定目录下创建临时目录节点，监听节点数目是否是我们要求的数目。创建一个父目录 /synchronizing，每个成员都监控标志（Set Watch）位目录 /synchronizing/start 是否存在，然后每个成员都加入这个队列，加入队列的方式就是创建 /synchronizing/member_i 的临时目录节点，然后每个成员获取 / synchronizing 目录的所有目录节点，也就是 member_i。判断 i 的值是否已经是成员的个数，如果小于成员个数等待 /synchronizing/start 的出现，如果已经相等就创建 /synchronizing/start。
- 队列按照 FIFO 方式进行入队和出队操作，例如实现生产者和消费者模型。分布式锁服务中的控制时序场景基本原理一致，入列有编号，出列按编号。在特定的目录下创建PERSISTENT_SEQUENTIAL节点，创建成功时Watcher通知等待的队列，队列删除序列号最小的节点用以消费。此场景下Zookeeper的znode用于消息存储，znode存储的数据就是消息队列中的消息内容，SEQUENTIAL序列号就是消息的编号，按序取出即可。由于创建的节点是持久化的，所以不必担心队列消息的丢失问题。

Curator也提供ZK Recipe的分布式队列实现。利用ZK的 PERSISTENTSEQUENTIAL节点，可以保证放入到队列中的项目是按照顺序排队的。如果单一的消费者从队列中取数据，那么它是先入先出的，这也是队列的特点。如果你严格要求顺序，你就得使用单一的消费者，可以使用leader选举只让leader作为唯一的消费者。但是，根据Netflix的Curator作者所说，ZooKeeper真心不适合做Queue，或者说ZK没有实现一个好的Queue，详细内容可以看 Tech Note 4，原因有五：

- ZK有1MB 的传输限制。实践中ZNode必须相对较小，而队列包含成千上万的消息，非常的大
- 如果有很多节点，ZK启动时相当的慢。而使用queue会导致好多ZNode。你需要显著增大 initLimit 和 syncLimit
- ZNode很大的时候很难清理。Netflix不得不创建了一个专门的程序做这事
- 当很大量的包含成千上万的子节点的ZNode时，ZK的性能变得不好
- ZK的数据库完全放在内存中。大量的Queue意味着会占用很多的内存空间

尽管如此，Curator还是创建了各种Queue的实现。如果Queue的数据量不太多，数据量不太大的情况下，酌情考虑，还是可以使用的。

### DistributedQueue

**DistributedQueue介绍**

DistributedQueue是最普通的一种队列。 它设计以下四个类：

- QueueBuilder - 创建队列使用QueueBuilder,它也是其它队列的创建类
- QueueConsumer - 队列中的消息消费者接口
- QueueSerializer - 队列消息序列化和反序列化接口，提供了对队列中的对象的序列化和反序列化
- DistributedQueue - 队列实现类

  QueueConsumer是消费者，它可以接收队列的数据。处理队列中的数据的代码逻辑可以放在QueueConsumer.consumeMessage()中。

  正常情况下先将消息从队列中移除，再交给消费者消费。但这是两个步骤，不是原子的。可以调用Builder的lockPath()消费者加锁，当消费者消费数据时持有锁，这样其它消费者不能消费此消息。如果消费失败或者进程死掉，消息可以交给其它进程。这会带来一点性能的损失。最好还是单消费者模式使用队列。

**示例程序**

```typescript
public class DistributedQueueExample
{
    private static final String PATH = "/example/queue";
    public static void main(String[] args) throws Exception
    {
        CuratorFramework clientA = CuratorFrameworkFactory.newClient("127.0.0.1:2181", new ExponentialBackoffRetry(1000, 3));
        clientA.start();
        CuratorFramework clientB = CuratorFrameworkFactory.newClient("127.0.0.1:2181", new ExponentialBackoffRetry(1000, 3));
        clientB.start();

        DistributedQueue<String> queueA = null;
        QueueBuilder<String> builderA = QueueBuilder.builder(clientA, createQueueConsumer("A"), createQueueSerializer(), PATH);
        queueA = builderA.buildQueue();
        queueA.start();
        
        DistributedQueue<String> queueB = null;
        QueueBuilder<String> builderB = QueueBuilder.builder(clientB, createQueueConsumer("B"), createQueueSerializer(), PATH);
        queueB = builderB.buildQueue();
        queueB.start();

        for (int i = 0; i < 100; i++)
        {
            queueA.put(" test-A-" + i);
            Thread.sleep(10);
            queueB.put(" test-B-" + i);
        }

        Thread.sleep(1000 * 10);// 等待消息消费完成
        queueB.close();
        queueA.close();
        clientB.close();
        clientA.close();
        System.out.println("OK!");
    }

    /** 队列消息序列化实现类 */
    private static QueueSerializer<String> createQueueSerializer()
    {
        return new QueueSerializer<String>()
        {
            @Override
            public byte[] serialize(String item)
            {
                return item.getBytes();
            }
            @Override
            public String deserialize(byte[] bytes)
            {
                return new String(bytes);
            }
        };
    }

    /** 定义队列消费者 */
    private static QueueConsumer<String> createQueueConsumer(final String name)
    {
        return new QueueConsumer<String>()
        {
            @Override
            public void stateChanged(CuratorFramework client, ConnectionState newState)
            {
                System.out.println("连接状态改变: " + newState.name());
            }
            @Override
            public void consumeMessage(String message) throws Exception
            {
                System.out.println("消费消息(" + name + "): " + message);
            }
        };
    }
}
```

以上程序创建两个client(A和B)，它们在同一路径上创建队列(A和B)，同时发消息、同时消费消息。

### DistributedIdQueue

DistributedIdQueue和上面的队列类似，但是可以为队列中的每一个元素设置一个ID。可以通过ID把队列中任意的元素移除。

**DistributedIdQueue结束**

DistributedIdQueue的使用与上面队列的区别是：

- 通过下面方法创建：builder.buildIdQueue()
- 放入元素时：queue.put(aMessage, messageId);
- 移除元素时：int numberRemoved = queue.remove(messageId);

**示例程序**

在这个例子中，有些元素还没有被消费者消费时就移除了，这样消费者不会收到删除的消息。(此示例是根据上面例子修改而来)

```cpp
public class DistributedIdQueueExample
{
    private static final String PATH = "/example/queue";

    public static void main(String[] args) throws Exception
    {
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", new ExponentialBackoffRetry(1000, 3));
        client.start();
        
        DistributedIdQueue<String> queue = null;
        QueueConsumer<String> consumer = createQueueConsumer("A");
        QueueBuilder<String> builder = QueueBuilder.builder(client, consumer, createQueueSerializer(), PATH);
        queue = builder.buildIdQueue();
        queue.start();

        for (int i = 0; i < 10; i++)
        {
            queue.put(" test-" + i, "Id" + i);
            Thread.sleep((long) (50 * Math.random()));
            queue.remove("Id" + i);
        }

        Thread.sleep(1000 * 3);
        queue.close();
        client.close();
        System.out.println("OK!");
    }
......
}
```

### DistributedPriorityQueue

优先级队列对队列中的元素按照优先级进行排序。Priority越小，元素月靠前，越先被消费掉。

**DistributedPriorityQueue介绍**

通过builder.buildPriorityQueue(minItemsBeforeRefresh)方法创建。

当优先级队列得到元素增删消息时，它会暂停处理当前的元素队列，然后刷新队列。minItemsBeforeRefresh指定刷新前当前活动的队列的最小数量。主要设置你的程序可以容忍的不排序的最小值。

放入队列时需要指定优先级：queue.put(aMessage, priority);

**示例程序**

```cpp
public class DistributedPriorityQueueExample
{
    private static final String PATH = "/example/queue";
    public static void main(String[] args) throws Exception
    {
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", new ExponentialBackoffRetry(1000, 3));
        client.start();
        DistributedPriorityQueue<String> queue = null;
        QueueConsumer<String> consumer = createQueueConsumer("A");
        QueueBuilder<String> builder = QueueBuilder.builder(client, consumer, createQueueSerializer(), PATH);
        queue = builder.buildPriorityQueue(0);
        queue.start();
        for (int i = 0; i < 5; i++)
        {
            int priority = (int) (Math.random() * 100);
            System.out.println("test-" + i + " 优先级:" + priority);
            queue.put("test-" + i, priority);
            Thread.sleep((long) (50 * Math.random()));
        }
        Thread.sleep(1000 * 2);
        queue.close();
        client.close();
    }
......
}
```

### DistributedDelayQueue

JDK中也有DelayQueue，不知道你是否熟悉。DistributedDelayQueue也提供了类似的功能，元素有个delay值，消费者隔一段时间才能收到元素。

**DistributedDelayQueue介绍**

放入元素时可以指定delayUntilEpoch：queue.put(aMessage, delayUntilEpoch);

**注意：**delayUntilEpoch不是离现在的一个时间间隔，比如20毫秒，而是未来的一个时间戳，如 System.currentTimeMillis() + 10秒。如果delayUntilEpoch的时间已经过去，消息会立刻被消费者接收。

**示例程序**

```cpp
public class DistributedDelayQueueExample
{
    private static final String PATH = "/example/queue";
    public static void main(String[] args) throws Exception
    {
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", new ExponentialBackoffRetry(1000, 3));
        client.start();
        DistributedDelayQueue<String> queue = null;
        QueueConsumer<String> consumer = createQueueConsumer("A");
        QueueBuilder<String> builder = QueueBuilder.builder(client, consumer, createQueueSerializer(), PATH);
        queue = builder.buildDelayQueue();
        queue.start();
        for (int i = 0; i < 10; i++)
        {
            queue.put("test-" + i, System.currentTimeMillis() + 3000);
        }
        System.out.println("put 完成！");
        Thread.sleep(1000 * 5);
        queue.close();
        client.close();
        System.out.println("OK!");
    }
......
}
```

### SimpleDistributedQueue

前面虽然实现了各种队列，但是你注意到没有，这些队列并没有实现类似JDK一样的接口。SimpleDistributedQueue提供了和JDK一致性的接口(但是没有实现Queue接口)。

**SimpleDistributedQueue介绍**

SimpleDistributedQueue常用方法：

```java
// 创建
public SimpleDistributedQueue(CuratorFramework client, String path)

// 增加元素
public boolean offer(byte[] data) throws Exception

// 删除元素
public byte[] take() throws Exception

// 另外还提供了其它方法
public byte[] peek() throws Exception
public byte[] poll(long timeout, TimeUnit unit) throws Exception
public byte[] poll() throws Exception
public byte[] remove() throws Exception
public byte[] element() throws Exception
```

没有add方法，多了take方法。take方法在成功返回之前会被阻塞。而poll在队列为空时直接返回null。

# 参考资料

**官方**

- [ZooKeeper 官网](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fzookeeper.apache.org%2F)
- [ZooKeeper 官方文档](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FZOOKEEPER)
- [ZooKeeper Github](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fgithub.com%2Fapache%2Fzookeeper)

**书籍**

- [《Hadoop 权威指南（第四版）》](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fitem.jd.com%2F12109713.html)
- [《从 Paxos 到 Zookeeper 分布式一致性原理与实践》](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fitem.jd.com%2F11622772.html)

**文章**

- [分布式服务框架 ZooKeeper -- 管理分布式环境中的数据](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fwww.ibm.com%2Fdeveloperworks%2Fcn%2Fopensource%2Fos-cn-zookeeper%2Findex.html)
- [ZooKeeper 的功能以及工作原理](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fwww.cnblogs.com%2Ffelixzh%2Fp%2F5869212.html)
- [ZooKeeper 简介及核心概念](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fgithub.com%2Fheibaiying%2FBigData-Notes%2Fblob%2Fmaster%2Fnotes%2FZooKeeper%E7%AE%80%E4%BB%8B%E5%8F%8A%E6%A0%B8%E5%BF%83%E6%A6%82%E5%BF%B5.md)
- [详解分布式协调服务 ZooKeeper](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fdraveness.me%2Fzookeeper-chubby)
- [深入浅出 Zookeeper（一） Zookeeper 架构及 FastLeaderElection 机制](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fwww.jasongj.com%2Fzookeeper%2Ffastleaderelection%2F)

