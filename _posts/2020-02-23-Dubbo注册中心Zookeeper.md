---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo注册中心Zookeeper

## 
ZookeeperRegistryFactory：
```
public Registry createRegistry(URL url) {
    /* 创建ZookeeperRegistry实例 */
    return new ZookeeperRegistry(url, zookeeperTransporter);
}

```

ZookeeperRegistry
```
public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
    super(url); /* 调用父类构造方法 */
    if (url.isAnyHost()) {
        throw new IllegalStateException("registry address == null");
    }
    String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT);
    if (!group.startsWith(Constants.PATH_SEPARATOR)) {
        group = Constants.PATH_SEPARATOR + group;
    }
    this.root = group;
    /* 连接zookeeper */
    zkClient = zookeeperTransporter.connect(url);
    // 添加监听器
    zkClient.addStateListener(new StateListener() {
        public void stateChanged(int state) {
            if (state == RECONNECTED) {
                try {
                    /* 如果监听到的状态是重连，做恢复操作 */
                    recover();
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                }
            }
        }
    });
}

```

FailbackRegistry：
```
public FailbackRegistry(URL url) {
    /* 调用父类构造方法 */
    super(url);
    // 重试间隔，默认5000ms
    int retryPeriod = url.getParameter(Constants.REGISTRY_RETRY_PERIOD_KEY, Constants.DEFAULT_REGISTRY_RETRY_PERIOD);
    this.retryFuture = retryExecutor.scheduleWithFixedDelay(new Runnable() {
        public void run() {
            try {
                retry(); /* ScheduledExecutorService固定间隔重试 */
            } catch (Throwable t) {
                logger.error("Unexpected error occur at failed retry, cause: " + t.getMessage(), t);
            }
        }
    }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
}

```

AbstractRegistry：
```
public AbstractRegistry(URL url) {
    setUrl(url);
    // 是否同步保存文件，默认false
    syncSaveFile = url.getParameter(Constants.REGISTRY_FILESAVE_SYNC_KEY, false);
    // 配置缓存文件，默认在用户根目录/.duboo文件夹下
    String filename = url.getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + url.getParameter(Constants.APPLICATION_KEY) + "-" + url.getAddress() + ".cache");
    File file = null;
    if (ConfigUtils.isNotEmpty(filename)) {
        file = new File(filename);
        if (!file.exists() && file.getParentFile() != null && !file.getParentFile().exists()) {
            if (!file.getParentFile().mkdirs()) {
                throw new IllegalArgumentException("Invalid registry store file " + file + ", cause: Failed to create directory " + file.getParentFile() + "!");
            }
        }
    }
    this.file = file;
    loadProperties(); // 将缓存文件加载成为Properties
    notify(url.getBackupUrls()); // 通知更新配置
}

```
这里提到了缓存文件，简单来说它就是用来做容灾的，consumer从注册中心订阅了provider等信息后会缓存到本地文件中，这样当注册中心不可用时，consumer启动时就可以加载这个缓存文件里面的内容与provider进行交互，这样就可以不依赖注册中心，但是无法获取到新的provider变更通知，所以如果provider信息在注册中心不可用这段时间发生了很大变化，那就很可能会出现服务无法调用的情况，在2.5.7（记得是这个版本）版本之前，dubbo有一个bug，如果启动时注册中心连接不上，启动程序会hang住，无法启动，所以在2.5.7版本之前这个文件是没用的，在2.5.7版本进行了修复。下面我们来看FailbackRegistry用ScheduledExecutorService重试什么东西：


FailbackRegistry：
```
protected void retry() {
    // 失败注册的重试
    if (!failedRegistered.isEmpty()) {
        Set<URL> failed = new HashSet<URL>(failedRegistered);
        if (failed.size() > 0) {
            if (logger.isInfoEnabled()) {
                logger.info("Retry register " + failed);
            }
            try {
                for (URL url : failed) {
                    try {
                        doRegister(url);
                        failedRegistered.remove(url);
                    } catch (Throwable t) {
                        logger.warn("Failed to retry register " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                    }
                }
            } catch (Throwable t) {
                logger.warn("Failed to retry register " + failed + ", waiting for again, cause: " + t.getMessage(), t);
            }
        }
    }
    // 失败注销的重试
    if (!failedUnregistered.isEmpty()) {
        Set<URL> failed = new HashSet<URL>(failedUnregistered);
        if (failed.size() > 0) {
            if (logger.isInfoEnabled()) {
                logger.info("Retry unregister " + failed);
            }
            try {
                for (URL url : failed) {
                    try {
                        doUnregister(url);
                        failedUnregistered.remove(url);
                    } catch (Throwable t) {
                        logger.warn("Failed to retry unregister  " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                    }
                }
            } catch (Throwable t) {
                logger.warn("Failed to retry unregister  " + failed + ", waiting for again, cause: " + t.getMessage(), t);
            }
        }
    }
    // 失败订阅的重试
    if (!failedSubscribed.isEmpty()) {
        Map<URL, Set<NotifyListener>> failed = new HashMap<URL, Set<NotifyListener>>(failedSubscribed);
        for (Map.Entry<URL, Set<NotifyListener>> entry : new HashMap<URL, Set<NotifyListener>>(failed).entrySet()) {
            if (entry.getValue() == null || entry.getValue().size() == 0) {
                failed.remove(entry.getKey());
            }
        }
        if (failed.size() > 0) {
            if (logger.isInfoEnabled()) {
                logger.info("Retry subscribe " + failed);
            }
            try {
                for (Map.Entry<URL, Set<NotifyListener>> entry : failed.entrySet()) {
                    URL url = entry.getKey();
                    Set<NotifyListener> listeners = entry.getValue();
                    for (NotifyListener listener : listeners) {
                        try {
                            doSubscribe(url, listener);
                            listeners.remove(listener);
                        } catch (Throwable t) {
                            logger.warn("Failed to retry subscribe " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                        }
                    }
                }
            } catch (Throwable t) {
                logger.warn("Failed to retry subscribe " + failed + ", waiting for again, cause: " + t.getMessage(), t);
            }
        }
    }
    // 失败退订的重试
    if (!failedUnsubscribed.isEmpty()) {
        Map<URL, Set<NotifyListener>> failed = new HashMap<URL, Set<NotifyListener>>(failedUnsubscribed);
        for (Map.Entry<URL, Set<NotifyListener>> entry : new HashMap<URL, Set<NotifyListener>>(failed).entrySet()) {
            if (entry.getValue() == null || entry.getValue().size() == 0) {
                failed.remove(entry.getKey());
            }
        }
        if (failed.size() > 0) {
            if (logger.isInfoEnabled()) {
                logger.info("Retry unsubscribe " + failed);
            }
            try {
                for (Map.Entry<URL, Set<NotifyListener>> entry : failed.entrySet()) {
                    URL url = entry.getKey();
                    Set<NotifyListener> listeners = entry.getValue();
                    for (NotifyListener listener : listeners) {
                        try {
                            doUnsubscribe(url, listener);
                            listeners.remove(listener);
                        } catch (Throwable t) {
                            logger.warn("Failed to retry unsubscribe " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                        }
                    }
                }
            } catch (Throwable t) {
                logger.warn("Failed to retry unsubscribe " + failed + ", waiting for again, cause: " + t.getMessage(), t);
            }
        }
    }
    // 失败通知的重试
    if (!failedNotified.isEmpty()) {
        Map<URL, Map<NotifyListener, List<URL>>> failed = new HashMap<URL, Map<NotifyListener, List<URL>>>(failedNotified);
        for (Map.Entry<URL, Map<NotifyListener, List<URL>>> entry : new HashMap<URL, Map<NotifyListener, List<URL>>>(failed).entrySet()) {
            if (entry.getValue() == null || entry.getValue().size() == 0) {
                failed.remove(entry.getKey());
            }
        }
        if (failed.size() > 0) {
            if (logger.isInfoEnabled()) {
                logger.info("Retry notify " + failed);
            }
            try {
                for (Map<NotifyListener, List<URL>> values : failed.values()) {
                    for (Map.Entry<NotifyListener, List<URL>> entry : values.entrySet()) {
                        try {
                            NotifyListener listener = entry.getKey();
                            List<URL> urls = entry.getValue();
                            listener.notify(urls);
                            values.remove(listener);
                        } catch (Throwable t) {
                            logger.warn("Failed to retry notify " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                        }
                    }
                }
            } catch (Throwable t) {
                logger.warn("Failed to retry notify " + failed + ", waiting for again, cause: " + t.getMessage(), t);
            }
        }
    }
}

```

我们看到，重试就是对注册、订阅等各个失败的操作进行重试，dubbo会在这些动作失败时将失败的记录存入集合或map中，这里会取出这些记录进行重试。下面我们来看zookeeper的链接，可以选择使用ZkClient或curator来连接zookeeper，默认为ZkClient：
ZkclientZookeeperTransporter：
```
public ZookeeperClient connect(URL url) {
    /* 构建ZkclientZookeeperClient */
    return new ZkclientZookeeperClient(url);
}

```

ZkclientZookeeperClient：
```
public ZkclientZookeeperClient(URL url) {
    super(url);
    /* 构建ZkClientWrapper，可以在连接超时后自动监控连接的状态 */
    client = new ZkClientWrapper(url.getBackupAddress(), 30000);
    // 添加监听器，用于连接状态变更通知监听器
    client.addListener(new IZkStateListener() {
        public void handleStateChanged(KeeperState state) throws Exception {
            ZkclientZookeeperClient.this.state = state;
            if (state == KeeperState.Disconnected) {
                stateChanged(StateListener.DISCONNECTED);
            } else if (state == KeeperState.SyncConnected) {
                stateChanged(StateListener.CONNECTED);
            }
        }
        public void handleNewSession() throws Exception {
            stateChanged(StateListener.RECONNECTED);
        }
    });
    client.start(); /* 开启线程，连接zookeeper */
}

```


ZkClientWrapper：
```
public ZkClientWrapper(final String serverAddr, long timeout) {
    this.timeout = timeout;
    // 创建任务创建ZkClient
    listenableFutureTask = ListenableFutureTask.create(new Callable<ZkClient>() {
        @Override
        public ZkClient call() throws Exception {
            return new ZkClient(serverAddr, Integer.MAX_VALUE);
        }
    });
}

```

ZkClientWrapper：
```
public void start() {
    if (!started) {
        Thread connectThread = new Thread(listenableFutureTask);
        connectThread.setName("DubboZkclientConnector");
        connectThread.setDaemon(true);
        connectThread.start(); // 创建线程执行创建ZkClient任务
        try {
            client = listenableFutureTask.get(timeout, TimeUnit.MILLISECONDS);
        } catch (Throwable t) {
            logger.error("Timeout! zookeeper server can not be connected in : " + timeout + "ms!", t);
        }
        started = true;
    } else {
        logger.warn("Zkclient has already been started!");
    }
}

```

创建ZkClient之后接下来添加添加了一个连接状态变更监听器，目的是在重连时做恢复操作：

FailbackRegistry：
```
protected void recover() throws Exception {
    Set<URL> recoverRegistered = new HashSet<URL>(getRegistered());
    if (!recoverRegistered.isEmpty()) {
        if (logger.isInfoEnabled()) {
            logger.info("Recover register url " + recoverRegistered);
        }
        for (URL url : recoverRegistered) {
            failedRegistered.add(url); // 将注册相关信息添加到失败注册集合中
        }
    }
    Map<URL, Set<NotifyListener>> recoverSubscribed = new HashMap<URL, Set<NotifyListener>>(getSubscribed());
    if (!recoverSubscribed.isEmpty()) {
        if (logger.isInfoEnabled()) {
            logger.info("Recover subscribe url " + recoverSubscribed.keySet());
        }
        for (Map.Entry<URL, Set<NotifyListener>> entry : recoverSubscribed.entrySet()) {
            URL url = entry.getKey();
            for (NotifyListener listener : entry.getValue()) {
                // 将订阅相关信息添加到失败订阅map中
                addFailedSubscribed(url, listener);
            }
        }
    }
}

```

我们看到，恢复操作其实就是将注册和订阅的信息保存起来，我们之前看到的重试流程会拉取这些信息进行重试。接下来就是将服务信息注册到注册中心：
FailbackRegistry：
```
public void register(URL url) {
    if (destroyed.get()){
        return;
    }
    super.register(url); // 添加到已注册服务集合中
    failedRegistered.remove(url); // 从失败的注册集合中移除
    failedUnregistered.remove(url); // 从失败的注销集合中移除
    try {
        doRegister(url); /* 发起注册请求 */
    } catch (Exception e) {
        Throwable t = e;
        // 如果启动检测被打开，则直接抛出异常
        boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
        && url.getParameter(Constants.CHECK_KEY, true)
        && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
        boolean skipFailback = t instanceof SkipFailbackWrapperException;
        if (check || skipFailback) {
            if (skipFailback) {
                t = t.getCause();
            }
            throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
        } else {
            logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
        }
        // 将失败的注册请求记录到失败的列表中，定期重试
        failedRegistered.add(url);
    }
}

```


ZookeeperRegistry：
```
protected void doRegister(URL url) {
    try {
        /* 创建zookeeper节点 */
        zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
    } catch (Throwable e) {
        throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}

```


AbstractZookeeperClient
```
public void create(String path, boolean ephemeral) {
    int i = path.lastIndexOf('/');
    if (i > 0) {
        String parentPath = path.substring(0, i);
        if (!checkExists(parentPath)) {
            // 递归创建父节点
            create(parentPath, false);
        }
    }
    if (ephemeral) {
        createEphemeral(path); /* 创建临时节点 */
    } else {
        createPersistent(path); /* 创建持久节点 */
    }
}

```

ZkclientZookeeperClient：
```
public void createEphemeral(String path) {
    try {
        /* 创建临时节点 */
        client.createEphemeral(path);
    } catch (ZkNodeExistsException e) {
    }
}

```









