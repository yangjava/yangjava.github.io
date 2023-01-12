---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---
## Apache Curator（ZooKeeper客户端框架）

Curator是Netflix公司开源的一套zookeeper客户端框架，解决了很多Zookeeper客户端非常底层的细节开发工作，包括连接重连、反复注册Watcher和NodeExistsException异常等等。

Apache Curator是一个比较完善的ZooKeeper客户端框架，通过封装的一套高级API 简化了ZooKeeper的操作。通过查看官方文档，可以发现Curator主要解决了三类问题：

- 封装ZooKeeper client与ZooKeeper server之间的连接处理
- 提供了一套Fluent风格的操作API
- 提供ZooKeeper各种应用场景(recipe， 比如：分布式锁服务、集群领导选举、共享计数器、缓存机制、分布式队列等)的抽象封装

**Curator主要从以下几个方面降低了zk使用的复杂性：**

- 重试机制:提供可插拔的重试机制, 它将给捕获所有可恢复的异常配置一个重试策略，并且内部也提供了几种标准的重试策略(比如指数补偿)
- 连接状态监控: Curator初始化之后会一直对zk连接进行监听，一旦发现连接状态发生变化将会作出相应的处理
- zk客户端实例管理:Curator会对zk客户端到server集群的连接进行管理，并在需要的时候重建zk实例，保证与zk集群连接的可靠性
- 各种使用场景支持:Curator实现了zk支持的大部分使用场景（甚至包括zk自身不支持的场景），这些实现都遵循了zk的最佳实践，并考虑了各种极端情况

### Maven依赖

```
       <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>2.4.2</version>
            <exclusions>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-api</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
```

### Curator基本用法

```
package com.zookeeper;

import lombok.extern.slf4j.Slf4j;
import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.api.transaction.CuratorTransactionBridge;
import org.apache.curator.framework.api.transaction.CuratorTransactionResult;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;

import java.util.List;

@Slf4j
public class CuratorDemo {

    //会话超时时间
    private static final int SESSION_TIMEOUT = 50 * 1000;

    //连接超时时间
    private static final int CONNECTION_TIMEOUT = 3 * 1000;

    //ZooKeeper服务地址
    private static final String CONNECT_ADDR = "127.0.0.1:2181";

    /**
     * Curator除了使用一般方法创建会话外，还可以使用fluent风格进行创建。
     * 值得注意的是session2会话含有隔离命名空间，
     * 即客户端对Zookeeper上数据节点的任何操作都是相对/base目录进行的，
     * 这有利于实现不同的Zookeeper的业务之间的隔离。
     */
    public static void main(String[] args) throws Exception {
        //1 重试策略：初试时间为1s 重试10次
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 10);
        //2 通过工厂创建连接
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString(CONNECT_ADDR).connectionTimeoutMs(CONNECTION_TIMEOUT)
                .sessionTimeoutMs(SESSION_TIMEOUT)
                .retryPolicy(retryPolicy)
//命名空间           .namespace("super")
                .build();
        //3 开启连接
        client.start();

        System.out.println(ZooKeeper.States.CONNECTED);
        System.out.println(client.getState());

        //创建永久节点
        client.create().forPath("/curator","/curator data".getBytes());

        //创建永久有序节点
        client.create().withMode(CreateMode.PERSISTENT_SEQUENTIAL).forPath("/curator_sequential","/curator_sequential data".getBytes());

        //创建临时节点
        client.create().withMode(CreateMode.EPHEMERAL)
                .forPath("/curator/ephemeral","/curator/ephemeral data".getBytes());

        //创建临时有序节点
        client.create().withMode(CreateMode.EPHEMERAL_SEQUENTIAL) .forPath("/curator/ephemeral_path1","/curator/ephemeral_path1 data".getBytes());

        client.create().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath("/curator/ephemeral_path2","/curator/ephemeral_path2 data".getBytes());

        //测试检查某个节点是否存在
        Stat stat1 = client.checkExists().forPath("/curator");
        Stat stat2 = client.checkExists().forPath("/curator2");

        System.out.println("'/curator'是否存在： " + (stat1 != null ? true : false));
        System.out.println("'/curator2'是否存在： " + (stat2 != null ? true : false));

        //获取某个节点的所有子节点
        System.out.println(client.getChildren().forPath("/"));

        //获取某个节点数据
        System.out.println(new String(client.getData().forPath("/curator")));

        //设置某个节点数据
        client.setData().forPath("/curator","/curator modified data".getBytes());

        //创建测试节点
        client.create().creatingParentsIfNeeded()
                .forPath("/curator/del_key1","/curator/del_key1 data".getBytes());

        client.create().creatingParentsIfNeeded()
                .forPath("/curator/del_key2","/curator/del_key2 data".getBytes());

        client.create().forPath("/curator/del_key2/test_key","test_key data".getBytes());

        //删除该节点
        client.delete().forPath("/curator/del_key1");

        //级联删除子节点
        client.delete().guaranteed().deletingChildrenIfNeeded().forPath("/curator/del_key2");
    }
    
}


```

- orSetData()方法：如果节点存在则Curator将会使用给出的数据设置这个节点的值，相当于 setData() 方法
- creatingParentContainersIfNeeded()方法：如果指定节点的父节点不存在，则Curator将会自动级联创建父节点
- guaranteed()方法：如果服务端可能删除成功，但是client没有接收到删除成功的提示，Curator将会在后台持续尝试删除该节点
- deletingChildrenIfNeeded()方法：如果待删除节点存在子节点，则Curator将会级联删除该节点的子节点

### 事务管理

```
    /* 事务管理：碰到异常，事务会回滚
     * @throws Exception
     */
    @Test
    public void testTransaction() throws Exception{
    
    client.inTransaction().check().forPath("/nodeA")
                .and()
                .create().withMode(CreateMode.EPHEMERAL).forPath("/nodeB", "init".getBytes())
                .and()
                .create().withMode(CreateMode.EPHEMERAL).forPath("/nodeC", "init".getBytes())
                .and()
                .commit();
                
        //定义几个基本操作
        CuratorOp createOp = client.transactionOp().create()
                .forPath("/curator/one_path","some data".getBytes());
        
        CuratorOp setDataOp = client.transactionOp().setData()
                .forPath("/curator","other data".getBytes());
        
        CuratorOp deleteOp = client.transactionOp().delete()
                .forPath("/curator");
        
        //事务执行结果
        List<CuratorTransactionResult> results = client.transaction()
                .forOperations(createOp,setDataOp,deleteOp);
        
        //遍历输出结果
        for(CuratorTransactionResult result : results){
            System.out.println("执行结果是： " + result.getForPath() + "--" + result.getType());
        }
    }
//因为节点“/curator”存在子节点，所以在删除的时候将会报错，事务回滚
```

### 监听器

Curator提供了三种Watcher(Cache)来监听结点的变化：

- **Path Cache**：监视一个路径下1）孩子结点的创建、2）删除，3）以及结点数据的更新。产生的事件会传递给注册的PathChildrenCacheListener。
- **Node Cache**：监视一个结点的创建、更新、删除，并将结点的数据缓存在本地。
- **Tree Cache**：Path Cache和Node Cache的“合体”，监视路径下的创建、更新、删除事件，并缓存路径下所有孩子结点的数据。

```
/**
         * 在注册监听器的时候，如果传入此参数，当事件触发时，逻辑由线程池处理
         */
        ExecutorService pool = Executors.newFixedThreadPool(2);
        
        /**
         * 监听数据节点的变化情况
         */
        final NodeCache nodeCache = new NodeCache(client, "/zk-huey/cnode", false);
        nodeCache.start(true);
        nodeCache.getListenable().addListener(
            new NodeCacheListener() {
                @Override
                public void nodeChanged() throws Exception {
                    System.out.println("Node data is changed, new data: " + 
                        new String(nodeCache.getCurrentData().getData()));
                }
            }, 
            pool
        );
        
        /**
         * 监听子节点的变化情况
         */
        final PathChildrenCache childrenCache = new PathChildrenCache(client, "/zk-huey", true);
        childrenCache.start(StartMode.POST_INITIALIZED_EVENT);
        childrenCache.getListenable().addListener(
            new PathChildrenCacheListener() {
                @Override
                public void childEvent(CuratorFramework client, PathChildrenCacheEvent event)
                        throws Exception {
                        switch (event.getType()) {
                        case CHILD_ADDED:
                            System.out.println("CHILD_ADDED: " + event.getData().getPath());
                            break;
                        case CHILD_REMOVED:
                            System.out.println("CHILD_REMOVED: " + event.getData().getPath());
                            break;
                        case CHILD_UPDATED:
                            System.out.println("CHILD_UPDATED: " + event.getData().getPath());
                            break;
                        default:
                            break;
                    }
                }
            },
            pool
        );
        
        client.setData().forPath("/zk-huey/cnode", "world".getBytes());
        
        Thread.sleep(10 * 1000);
        pool.shutdown();
        client.close();
```

### 分布式锁

分布式编程时，比如最容易碰到的情况就是应用程序在线上多机部署，于是当多个应用同时访问某一资源时，就需要某种机制去协调它们。例如，现在一台应用正在rebuild缓存内容，要临时锁住某个区域暂时不让访问；又比如调度程序每次只想一个任务被一台应用执行等等。

下面的程序会启动两个线程t1和t2去争夺锁，拿到锁的线程会占用5秒。运行多次可以观察到，有时是t1先拿到锁而t2等待，有时又会反过来。Curator会用我们提供的lock路径的结点作为全局锁，这个结点的数据类似这种格式：[_c_64e0811f-9475-44ca-aa36-c1db65ae5350-lock-0000000005]，每次获得锁时会生成这种串，释放锁时清空数据。

```
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.RetryNTimes;

import java.util.concurrent.TimeUnit;

/**
 * Curator framework's distributed lock test.
 */
public class CuratorDistrLockTest {

    /** Zookeeper info */
    private static final String ZK_ADDRESS = "192.168.1.100:2181";
    private static final String ZK_LOCK_PATH = "/zktest";

    public static void main(String[] args) throws InterruptedException {
        // 1.Connect to zk
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

### Leader选举

当集群里的某个服务down机时，我们可能要从slave结点里选出一个作为新的master，这时就需要一套能在分布式环境中自动协调的Leader选举方法。Curator提供了LeaderSelector监听器实现Leader选举功能。同一时刻，只有一个Listener会进入takeLeadership()方法，说明它是当前的Leader。注意：**当Listener从takeLeadership()退出时就说明它放弃了“Leader身份”**，这时Curator会利用Zookeeper再从剩余的Listener中选出一个新的Leader。autoRequeue()方法使放弃Leadership的Listener有机会重新获得Leadership，如果不设置的话放弃了的Listener是不会再变成Leader的。

```
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.leader.LeaderSelector;
import org.apache.curator.framework.recipes.leader.LeaderSelectorListener;
import org.apache.curator.framework.state.ConnectionState;
import org.apache.curator.retry.RetryNTimes;
import org.apache.curator.utils.EnsurePath;

/**
 * Curator framework's leader election test.
 * Output:
 *  LeaderSelector-2 take leadership!
 *  LeaderSelector-2 relinquish leadership!
 *  LeaderSelector-1 take leadership!
 *  LeaderSelector-1 relinquish leadership!
 *  LeaderSelector-0 take leadership!
 *  LeaderSelector-0 relinquish leadership! 
 *      ...
 */
public class CuratorLeaderTest {

    /** Zookeeper info */
    private static final String ZK_ADDRESS = "192.168.1.100:2181";
    private static final String ZK_PATH = "/zktest";

    public static void main(String[] args) throws InterruptedException {
        LeaderSelectorListener listener = new LeaderSelectorListener() {
            @Override
            public void takeLeadership(CuratorFramework client) throws Exception {
                System.out.println(Thread.currentThread().getName() + " take leadership!");

                // takeLeadership() method should only return when leadership is being relinquished.
                Thread.sleep(5000L);

                System.out.println(Thread.currentThread().getName() + " relinquish leadership!");
            }

            @Override
            public void stateChanged(CuratorFramework client, ConnectionState state) {
            }
        };

        new Thread(() -> {
            registerListener(listener);
        }).start();

        new Thread(() -> {
            registerListener(listener);
        }).start();

        new Thread(() -> {
            registerListener(listener);
        }).start();

        Thread.sleep(Integer.MAX_VALUE);
    }

    private static void registerListener(LeaderSelectorListener listener) {
        // 1.Connect to zk
        CuratorFramework client = CuratorFrameworkFactory.newClient(
                ZK_ADDRESS,
                new RetryNTimes(10, 5000)
        );
        client.start();

        // 2.Ensure path
        try {
            new EnsurePath(ZK_PATH).ensure(client.getZookeeperClient());
        } catch (Exception e) {
            e.printStackTrace();
        }

        // 3.Register listener
        LeaderSelector selector = new LeaderSelector(client, ZK_PATH, listener);
        selector.autoRequeue();
        selector.start();
    }
}
```