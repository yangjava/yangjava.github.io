---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---

## Zookeeper操作

### Maven依赖

```
<dependencies>
       <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.4.14</version>
        </dependency>
</dependencies>
```

### 单机启动Zookeeper服务

```
package com.zookeeper;

import org.apache.zookeeper.server.ZooKeeperServerMain;

import java.util.Arrays;

public class Main {
    /**
     * 单机版ZooKeeper,启动类是ZooKeeperServerMain。
     * 最终调用ZooKeeperServer的startup()方法来处理request。
     * 可以通过配置选择使用JAVA自带NIO或者netty作为异步连接服务端。
     */
    public static void main(String[] args) {
        SingServer();
    }
    /**
     * 启动流程
     * ZooKeeper启动参数有两种配置方式：
     * 方式1：
     * main方法中传入四个参数，其中前两参数必填，后两个参数可选。
     * 分别为：对客户端暴露的端口clientPortAddress,
     * 存放事务记录、内存树快照记录的dataDir,
     * 用于指定seesion检查时间间隔的tickTime,
     * 控制最大客户端连接数的maxClientCnxns。
     *方式2：
     * 给出启动参数配置文件路径，
     * 当args启动参数只有一个时，ZooKeeperServerMain中main方法，
     * 会认为传入了配置文件路径，
     * 默认情况下该参数是传conf目录中的zoo.cfg。
     */
    public static void SingServer() {
        String clientPortAddress="2181";
        String dataDir="F:/data";
        String[] args= Arrays.asList(clientPortAddress,dataDir).toArray(new String[]{});
        ZooKeeperServerMain zooKeeperServerMain=new ZooKeeperServerMain();
        zooKeeperServerMain.main(args);
    }
}

```

### zookeeper节点操作

```
package com.zookeeper;

import lombok.extern.slf4j.Slf4j;
import org.apache.zookeeper.*;
import org.apache.zookeeper.ZooDefs.Ids;
import org.apache.zookeeper.data.ACL;
import org.apache.zookeeper.data.Stat;

import java.io.IOException;
import java.util.ArrayList;
@Slf4j
public class ZookeeperDemo {
    private static String connectString = "127.0.0.1:2181";
    private static int sessionTimeout = 50 * 1000;

    public static void main(String[] args) throws Exception{
        ZookeeperDemo zookeeperDemo=new ZookeeperDemo();
        ZooKeeper zooKeeper = zookeeperDemo.connectionZooKeeper();
        String result = zookeeperDemo.createZnode(zooKeeper, "/user", "zhangsan");
        log.info("创建zookeeper znode:{}",result);
        String znodeData = zookeeperDemo.getZnodeData(zooKeeper, "/user");
        log.info("获取zookeeper znode:{}",znodeData);
    }

    public ZooKeeper connectionZooKeeper() throws IOException {
        log.info("client连接Zookeeper");
        ZooKeeper zooKeeper = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
            public void process(WatchedEvent event) {
                //可做其他操作（设置监听或观察者）
                log.info("监听机制");
            }
        });
        return zooKeeper;
    }


    /**
     * 创建节点
     * 1. CreateMode.PERSISTENT ：持久节点，一旦创建就保存到硬盘上面
     　　　  2.  CreateMode.SEQUENTIAL ： 顺序持久节点
     　　　  3.  CreateMode.EPHEMERAL ：临时节点，创建以后如果断开连接则该节点自动删除
     　　　  4.  CreateMode.EPHEMERAL_SEQUENTIAL ：顺序临时节点
     * @param zooKeeper Zookeeper已经建立连接的对象
     * @param path 要创建节点的路径
     * @param data 该节点上的数据
     * @return 返回创建的节点的路径
     * @throws KeeperException
     * @throws InterruptedException
     */
    public String createZnode(ZooKeeper zooKeeper, String path, String data) throws KeeperException, InterruptedException {
        byte[] bytesData = data.getBytes();
        //访问控制列表
        ArrayList<ACL> openAclUnsafe = Ids.OPEN_ACL_UNSAFE;
        //创建模式
        CreateMode mode = CreateMode.PERSISTENT;
        String result = zooKeeper.create(path, bytesData, openAclUnsafe, mode);
        return result;
    }

    /**
     * 获取节点上的数据
     * @param zooKeeper Zookeeper已经建立连接的对象
     * @param path 节点路径
     * @return 返回节点上的数据
     * @throws KeeperException
     * @throws InterruptedException
     */
    public String getZnodeData(ZooKeeper zooKeeper, String path) throws KeeperException, InterruptedException {
        byte[] data = zooKeeper.getData(path, false, new Stat());
        return new String(data);
    }

    /**
     * 设置节点上的数据
     * @param zooKeeper Zookeeper已经建立连接的对象
     * @param path 节点路径
     * @param data
     * @return
     * @throws KeeperException
     * @throws InterruptedException
     */
    public Stat setZnodeData(ZooKeeper zooKeeper, String path, String data) throws KeeperException, InterruptedException {
        return zooKeeper.setData(path, data.getBytes(), -1);
    }

    /**
     * 判断节点是否存在
     * @param zooKeeper
     * @param path 节点路径
     * @return
     * @throws KeeperException
     * @throws InterruptedException
     */
    public Stat isExitZKPath(ZooKeeper zooKeeper, String path) throws KeeperException, InterruptedException {
        Stat stat = zooKeeper.exists(path, false);
        return stat;
    }


}

```