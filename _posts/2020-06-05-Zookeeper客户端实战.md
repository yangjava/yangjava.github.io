---
layout: post
categories: [Zookeeper]
description: none
keywords: Zookeeper
---
# Zookeeper客户端实战
Zookeeper作为⼀个分布式框架，主要用来解决分布式⼀致性问题，它提供了简单的分布式原语，并且对多种编程语⾔提供了API，所以接下来重点来看下Zookeeper的java客户端API使用方式。

## Zookeeper客户端简介
Zookeeper API共包含五个包，分别为：
- org.apache.zookeeper
- org.apache.zookeeper.data
- org.apache.zookeeper.server
- org.apache.zookeeper.server.quorum
- org.apache.zookeeper.server.upgrade

其中org.apache.zookeeper，包含Zookeeper类，他是我们编程时最常⽤的类文件。这个类是Zookeeper客户端的主要类文件。如果要使用Zookeeper服务，应⽤程序⾸先必须创建⼀个Zookeeper 实例，这时就需要使用此类。⼀旦客户端和Zookeeper服务端建立起了连接，Zookeeper系统将会给本次连接会话分配⼀个ID值，并且客户端将会周期性的向服务器端发送心跳来维持会话连接。只要连接有效，客户端就可以使⽤Zookeeper API来做相应处理了

## 准备工作：导入依赖
```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.14</version>
</dependency>
```

### 建立会话
```java

import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
 
import java.io.IOException;
import java.util.concurrent.CountDownLatch;
 
public class CreateSession implements Watcher {
    private static CountDownLatch countDownLatch = new CountDownLatch(1);
 
    /**
     * 建立会话
     */
    public static void main(String[] args) throws IOException, InterruptedException {
        /*
            客户端可以通过创建⼀个zk实例来连接zk服务器
            new Zookeeper(connectString,sesssionTimeOut,Wather)
            connectString: 连接地址：IP：端⼝
            sesssionTimeOut：会话超时时间：单位毫秒
            Wather：监听器(当特定事件触发监听时，zk会通过watcher通知到客户端)
        */
        ZooKeeper zooKeeper = new ZooKeeper("127.0.0.1:2181", 5000, new CreateSession());
        System.out.println(zooKeeper.getState());
 
        // 计数工具类 CountDownLatch : 不让main方法结束，让线程处于等待阻塞
        countDownLatch.await();
        System.out.println("客户端与服务端会话真正建立了");
    }
 
    /**
     * 回调方法：处理来自服务器端的watcher通知
     */
    // 当前类实现了Watcher接⼝，重写了process⽅法，该⽅法负责处理来⾃Zookeeper服务端的watcher通知，在收到服务端发送过来的SyncConnected事件之后，解除主程序在CountDownLatch上的等待阻塞，⾄此，会话创建完毕
    public void process(WatchedEvent watchedEvent) {
        //当连接创建了，服务端发送给客户端SyncConnected事件
        if(watchedEvent.getState() == Event.KeeperState.SyncConnected) {
            // 解除主程序在CountDownLatch上的等待阻塞
            countDownLatch.countDown();
        }
    }

    /**
     * 创建会话（可复用sessionId的实例）
     *
     * @throws Exception Exception
     */
    public static void createSession2() throws Exception {
        createSession1();//完成基础会话实例
        long sessionId = zooKeeper.getSessionId();
        byte[] sessionPasswd = zooKeeper.getSessionPasswd();
        System.out.println(String.format("首次获取sessionId：%s，sessionPasswd：%s", sessionId, sessionPasswd));
        // 使用sessionId
        zooKeeper = new ZooKeeper(hosts, 5000, new ZooKeeperWatcher(), sessionId, sessionPasswd);
        System.out.println("ZooKeeper.state session：" + zooKeeper.getState());
        Thread.sleep(Integer.MAX_VALUE);
    }
    
}

```
注意，ZooKeeper 客户端和服务端会话的建立是⼀个异步的过程，也就是说在程序中，构造⽅法会在处理完客户端初始化工作后立即返回，在⼤多数情况下，此时并没有真正建立好⼀个可用的会话，在会话的生命周期中处于“CONNECTING”的状态。当该会话真正创建完毕后ZooKeeper服务端会向会话对应的客户端发送⼀个事件通知，以告知客户端，客户端只有在获取这个通知之后，才算真正建立了会话。

### 创建节点
```java
import org.apache.zookeeper.*;
 
import java.io.IOException;
import java.util.concurrent.CountDownLatch;
 
public class CreateNote implements Watcher {
    private static ZooKeeper zooKeeper;
 
    /**
     * 建立会话
     */
    public static void main(String[] args) throws IOException, InterruptedException, KeeperException {
        /*
            客户端可以通过创建⼀个zk实例来连接zk服务器
            new Zookeeper(connectString,sesssionTimeOut,Wather)
            connectString: 连接地址：IP：端⼝
            sesssionTimeOut：会话超时时间：单位毫秒
            Wather：监听器(当特定事件触发监听时，zk会通过watcher通知到客户端)
        */
        zooKeeper = new ZooKeeper("127.0.0.1:2181", 5000, new CreateNote());
        System.out.println(zooKeeper.getState());
 
        Thread.sleep(Integer.MAX_VALUE);
 
    }
 
    // 创建节点的方法
    private static void createNoteSync() throws InterruptedException, KeeperException {
        /**
         *	path	：节点创建的路径
         *	data[]	：节点创建要保存的数据，是个byte类型的
         *	acl	：节点创建的权限信息(4种类型)
         *	    ANYONE_ID_UNSAFE	: 表示任何⼈
         *	    AUTH_IDS	：此ID仅可⽤于设置ACL。它将被客户机验证的ID替换。
         *	    OPEN_ACL_UNSAFE	：这是⼀个完全开放的ACL(常⽤)--> world:anyone
         *	    CREATOR_ALL_ACL	：此ACL授予创建者身份验证ID的所有权限
         *	createMode	：创建节点的类型(4种类型)
         *	    PERSISTENT：持久节点
         *	    PERSISTENT_SEQUENTIAL：持久顺序节点
         *	    EPHEMERAL：临时节点
         *	    EPHEMERAL_SEQUENTIAL：临时顺序节点
         String node = zookeeper.create(path,data,acl,createMode);
         */
        // 持久节点
        String note_persistent = zooKeeper.create("/lg-persistent", "持久节点内容".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
 
        // 临时节点
        String note_ephemeral = zooKeeper.create("/lg-ephemeral", "临时节点内容".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
 
        // 持久顺序节点
        String note_sequential = zooKeeper.create("/lg-sequential", "持久顺序节点内容".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL);
 
        System.out.println("创建的持久节点: " + note_persistent);
        System.out.println("创建的临时节点: " + note_ephemeral);
        System.out.println("创建的持久顺序节点: " + note_sequential);
 
    }
 
    /**
     * 回调方法：处理来自服务器端的watcher通知
     */
    public void process(WatchedEvent watchedEvent) {
        // SyncConnected
        if (watchedEvent.getState() == Event.KeeperState.SyncConnected) {
            // 创建节点
            try {
                createNoteSync();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
    }
}
```

### 获取节点数据
```java

import org.apache.zookeeper.*;
 
import java.io.IOException;
import java.util.List;
 
public class GetNoteData implements Watcher {
    private static ZooKeeper zooKeeper;
 
    /**
     * 建立会话
     */
    public static void main(String[] args) throws IOException, InterruptedException, KeeperException {
        /*
            客户端可以通过创建⼀个zk实例来连接zk服务器
            new Zookeeper(connectString,sesssionTimeOut,Wather)
            connectString: 连接地址：IP：端⼝
            sesssionTimeOut：会话超时时间：单位毫秒
            Wather：监听器(当特定事件触发监听时，zk会通过watcher通知到客户端)
        */
        zooKeeper = new ZooKeeper("127.0.0.1:2181", 5000, new GetNoteData());
        System.out.println(zooKeeper.getState());
 
        Thread.sleep(Integer.MAX_VALUE);
 
    }
 
 
    /**
     * 回调方法：处理来自服务器端的watcher通知
     */
    public void process(WatchedEvent watchedEvent) {
        /*
            子节点列表发生改变时，服务端会发送noteChildrenChanged事件通知
            要重新获取子节点列表，同时注意：通知是一次性的，需要反复注册监听
         */
        if (watchedEvent.getType() == Event.EventType.NodeChildrenChanged) {
            List<String> children = null;
            try {
                children = zooKeeper.getChildren("/lg-persistent", true);
            } catch (KeeperException | InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println(children);
        }
 
        // SyncConnected
        if (watchedEvent.getState() == Event.KeeperState.SyncConnected) {
 
            // 获取节点数据的方法
            try {
                getNoteData();
 
                // 获取节点的子节点列表方法
                getChildren();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
    }
 
    /*
        获取某个节点的内容
     */
    private void getNoteData() throws Exception {
        /**
         *	path	: 获取数据的路径
         *	watch	: 是否开启监听
         *	stat	: 节点状态信息
         *	    null: 表示获取最新版本的数据
         *	zk.getData(path, watch, stat);
         */
        byte[] data = zooKeeper.getData("/lg-persistent", false, null);
        System.out.println(new String(data));
    }
 
    /*
        获取某个节点的子节点列表方法
     */
    public static void getChildren() throws InterruptedException, KeeperException {
        /*
            path:路径
            watch:是否要启动监听，当⼦节点列表发⽣变化，会触发监听
            zooKeeper.getChildren(path, watch);
        */
        List<String> children = zooKeeper.getChildren("/lg-persistent", true);
        System.out.println(children);
    }
}
```

### 修改节点数据
```java
import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;
 
import java.io.IOException;
 
public class UpdateNoteData implements Watcher {
    private static ZooKeeper zooKeeper;
 
    /**
     * 建立会话
     */
    public static void main(String[] args) throws IOException, InterruptedException, KeeperException {
        /*
            客户端可以通过创建⼀个zk实例来连接zk服务器
            new Zookeeper(connectString,sesssionTimeOut,Wather)
            connectString: 连接地址：IP：端⼝
            sesssionTimeOut：会话超时时间：单位毫秒
            Wather：监听器(当特定事件触发监听时，zk会通过watcher通知到客户端)
        */
        zooKeeper = new ZooKeeper("127.0.0.1:2181", 5000, new UpdateNoteData());
        System.out.println(zooKeeper.getState());
 
        Thread.sleep(Integer.MAX_VALUE);
 
    }
 
 
    /**
     * 回调方法：处理来自服务器端的watcher通知
     */
    public void process(WatchedEvent watchedEvent) {
        // SyncConnected
        if (watchedEvent.getState() == Event.KeeperState.SyncConnected) {
            // 更新数据节点内容的方法
            try {
                updateNoteSync();
            } catch (InterruptedException | KeeperException e) {
                throw new RuntimeException(e);
            }
        }
    }
 
    /*
        更新数据节点内容的方法
     */
    private void updateNoteSync() throws InterruptedException, KeeperException {
        /*
            path:路径
            data:要修改的内容 byte[]
            version:为-1，表示对最新版本的数据进⾏修改
            zooKeeper.setData(path, data,version);
        */
        byte[] data = zooKeeper.getData("/lg-persistent", false, null);
        System.out.println("修改前的值：" + new String(data));
 
        // 修改 /lg-persistent 的数据    stat: 状态信息对象
        Stat stat = zooKeeper.setData("/lg-persistent", "客户端修改了节点数据".getBytes(), -1);
        byte[] data2 = zooKeeper.getData("/lg-persistent", false, null);
        System.out.println("修改后的值：" + new String(data2));
    }
}

```

### 删除节点
```java
import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;
 
import java.io.IOException;
 
public class DeleteNote implements Watcher {
    private static ZooKeeper zooKeeper;
 
    /**
     * 建立会话
     */
    public static void main(String[] args) throws IOException, InterruptedException, KeeperException {
        /*
            客户端可以通过创建⼀个zk实例来连接zk服务器
            new Zookeeper(connectString,sesssionTimeOut,Wather)
            connectString: 连接地址：IP：端⼝
            sesssionTimeOut：会话超时时间：单位毫秒
            Wather：监听器(当特定事件触发监听时，zk会通过watcher通知到客户端)
        */
        zooKeeper = new ZooKeeper("127.0.0.1:2181", 5000, new DeleteNote());
        System.out.println(zooKeeper.getState());
 
        Thread.sleep(Integer.MAX_VALUE);
 
    }
 
 
    /**
     * 回调方法：处理来自服务器端的watcher通知
     */
    public void process(WatchedEvent watchedEvent) {
        // SyncConnected
        if (watchedEvent.getState() == Event.KeeperState.SyncConnected) {
            // 删除节点
            try {
                deleteNoteSync();
            } catch (InterruptedException | KeeperException e) {
                throw new RuntimeException(e);
            }
        }
    }
 
    /*
        删除节点方法
     */
    private void deleteNoteSync() throws InterruptedException, KeeperException {
        /*
            zooKeeper.exists(path,watch) :判断节点是否存在
            zookeeper.delete(path,version) : 删除节点
        */
        Stat stat = zooKeeper.exists("/lg-persistent/c1", false);
        System.out.println(stat == null ? "该节点不存在" : "该节点存在");
        if (stat != null) {
            zooKeeper.delete("/lg-persistent/c1", -1);
        }
        Stat stat2 = zooKeeper.exists("/lg-persistent/c1", false);
        System.out.println(stat2 == null ? "该节点不存在" : "该节点存在");
    }
}
```

