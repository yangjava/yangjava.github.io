---
layout: post
categories: RocketMQ
description: none
keywords: RocketMQ
---

# ACL
这一节分析主要介绍了ACL的配置项、如何启用以及其源码分析,broker在启动时如何启用ACL,客户端向broker端发送请求时都有哪些ACL方面的操作以及broker收到客户端发送的请求后如何处理。

## ACL的配置项

什么是ACL？
ACL(权限控制)是RocketMQ提供topic资源级别的客户端访问控制，客户端在使用RocketMQ权限控制时，可以在客户端通过RPCHook注入AccessKey和SecretKey签名。

如何启用ACL

**broker端**
在broker端首先需要配置好plain_acl.yml，然后在broker.conf中添加配置项aclEnable=true(该配置项默认是false)

**客户端**
客户端通过RPCHook注入AccessKey和SecretKey签名

**plain_acl.yml文件**
在RocketMQ中权限控制属性的配置是在plain_acl.yml中，其配置项及其含义具体如下

|  字段   | 取值 |含义 |
|  ----  | ----  |----  |
| globalWhiteRemoteAddresses  | *;192.168.*.*;192.168.0.1 |全局IP白名单 |
| accessKey  | 字符串 |Access Key |
| secretKey  | 字符串 |Secret Key |
| whiteRemoteAddress  | *;192.168.*.*;192.168.0.1 |Access Key |
| admin  | true;false |是否管理员账户 |
| defaultTopicPerm  | DENY;PUB;SUB;PUB|SUB |默认的Topic权限 |
| defaultGroupPerm  | DENY;PUB;SUB;PUB|SUB |默认的ConsumerGroup权限 |
| topicPerms  | topic=权限 |各个Topic的权限 |
| groupPerms  | group=权限 |各个ConsumerGroup的权限 |


这里RocketMQ对资源访问控制权限的定义有以下四种：
- DENY	拒绝
- ANY	PUB 或者 SUB 权限
- PUB	发送权限
- SUB	订阅权限
注意：RocketMQ的权限控制存储的默认实现是基于yml配置文件。用户可以动态修改权限控制定义的属性，而不需重新启动Broker服务节点。


## ACL的使用

**broker端plain_acl.yml配置**
``` yaml
globalWhiteRemoteAddresses:

accounts:
- accessKey: RocketMQ
  secretKey: 12345678
  whiteRemoteAddress:
  admin: false
  defaultTopicPerm: DENY
  defaultGroupPerm: SUB
  topicPerms:
  - topic0607=PUB|SUB
  groupPerms:
  # the group should convert to retry topic
  - groupTest0607=DENY

- accessKey: rocketmq2
  secretKey: 12345678
  whiteRemoteAddress: 
  # if it is admin, it could access all resources
  admin: true

```

**Producer**
``` java

public class ACLProducerTest {
    private static final String ACL_ACCESS_KEY = "RocketMQ";

    private static final String ACL_SECRET_KEY = "12345678";

    public static void main(String[] args) throws MQClientException {
        DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName", getAclRPCHook());
        producer.setNamesrvAddr("127.0.0.1:9876");
        producer.start();

        for (int i = 0; i < 128; i++)
            try {
                {
                    Message msg = new Message("topic0607",
                            "TagA",
                            "OrderID188",
                            "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
                    SendResult sendResult = producer.send(msg);
                    System.out.printf("%s%n", sendResult);
                }

            } catch (Exception e) {
                e.printStackTrace();
            }

        producer.shutdown();
    }

    static RPCHook getAclRPCHook() {
        return new AclClientRPCHook(new SessionCredentials(ACL_ACCESS_KEY,ACL_SECRET_KEY));
    }
}


```

consumer

```java
public class ACLConsumerTest {
    private static final String ACL_ACCESS_KEY = "RocketMQ";

    private static final String ACL_SECRET_KEY = "12345678";

    public static void main(String[] args) throws MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("testGroup02", getAclRPCHook(), new AllocateMessageQueueAveragely());
        consumer.setNamesrvAddr("127.0.0.1:9876");
        consumer.subscribe("topic0607", "*");
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.registerMessageListener(new MessageListenerConcurrently() {

            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
        System.out.printf("Consumer Started.%n");

    }


    static RPCHook getAclRPCHook() {
        return new AclClientRPCHook(new SessionCredentials(ACL_ACCESS_KEY,ACL_SECRET_KEY));
    }

```

## ACL源码分析

### broker在启动时如何启用ACL

在broker启动过程中在对BrokerController进行初始化的过程中会调用initialAcl()

``` java
   /**
     * initialAcl方法主要是加载权限相关校验器，
     * RocketMQ的相关的管理的权限验证和安全就交给这里的加载的校验器了。
     * initialAcl方法也利用SPI原理加载接口的具体实现类，将所有加载的校验器缓存在map中，
     * 然后再注册RPC钩子，在请求之前调用校验器的validate的方法。
     */
    private void initialAcl() {
        // 首先判断Broker是否开启了acl，通过配置参数aclEnable指定，默认为false。
        if (!this.brokerConfig.isAclEnable()) {
            log.info("The broker dose not enable acl");
            return;
        }
        // 使用类似SPI机制，加载配置的AccessValidator,该方法返回一个列表，
        // 其实现逻辑时读取META-INF/service/org.apache.rocketmq.acl.AccessValidator文件中配置的访问验证器
        List<AccessValidator> accessValidators = ServiceProvider.load(ServiceProvider.ACL_VALIDATOR_ID, AccessValidator.class);
        if (accessValidators == null || accessValidators.isEmpty()) {
            log.info("The broker dose not load the AccessValidator");
            return;
        }
        // 将所有的权限校验器进行缓存以及注册
        // 遍历配置的访问验证器(AccessValidator),并向Broker处理服务器注册钩子函数，
        // RPCHook的doBeforeRequest方法会在服务端接收到请求，将其请求解码后，
        // 执行处理请求之前被调用;RPCHook的doAfterResponse方法会在处理完请求后，
        // 将结果返回之前被调用
        for (AccessValidator accessValidator: accessValidators) {
            final AccessValidator validator = accessValidator;
            this.registerServerRPCHook(new RPCHook() {

                @Override
                public void doBeforeRequest(String remoteAddr, RemotingCommand request) {
                    //Do not catch the exception
                    // 在RPCHook#doBeforeRequest方法中调用AccessValidator#validate,
                    // 在真实处理命令之前，先执行ACL的验证逻辑
                    // ，如果拥有该操作的执行权限，则放行，否则抛出AclException。
                    validator.validate(validator.parse(request, remoteAddr));
                }

                @Override
                public void doAfterResponse(String remoteAddr, RemotingCommand request, RemotingCommand response) {
                }
            });
        }
    }
```

首先判断broker端是否启用了ACL，如果broker端启用了ACL则会加载“META-INF/service/org.apache.rocketmq.acl.AccessValidator”路径下所有实现了 AccessValidator 接口的实现类，这里该文件的内容是“org.apache.rocketmq.acl.plain.PlainAccessValidator”，也就是说broker端采用PlainAccessValidator作为默认的权限访问校验器。注册RPCHook，这里注册分为两个一个是向remotingServer注册一个是向fastRemotingServer注册，其本质是将RPCHook添加到rpcHooks(是个List)

**上面注册的RPCHook是在什么时候调用的？**

broker端在接收到客户端的请求后会按照服务端启动过程中添加的handler来对请求进行处理，其中业务上的处理实现是NettyServerHandler，在NettyServerHandler中重写了channelRead0方法，顺着该方法往里面深入研究可以发现：在processRequestCommand方法中有调用上面注册的RPCHook，即doBeforeRpcHooks方法
```java
public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {
        final Pair<NettyRequestProcessor, ExecutorService> matched = this.processorTable.get(cmd.getCode());
        final Pair<NettyRequestProcessor, ExecutorService> pair = null == matched ? this.defaultRequestProcessor : matched;
        final int opaque = cmd.getOpaque();

        if (pair != null) {
            Runnable run = new Runnable() {
                @Override
                public void run() {
                    try {
                        doBeforeRpcHooks(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd);
                        final RemotingResponseCallback callback = new RemotingResponseCallback() {
                            @Override
                            public void callback(RemotingCommand response) {
                                doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd, response);
                                if (!cmd.isOnewayRPC()) {
                                    if (response != null) {
                                        response.setOpaque(opaque);
                                        response.markResponseType();
                                        try {
                                            ctx.writeAndFlush(response);
                                        } catch (Throwable e) {
                                            log.error("process request over, but response failed", e);
                                            log.error(cmd.toString());
                                            log.error(response.toString());
                                        }
                                    } else {
                                    }
                                }
                            }
                        };

```
这样，我们就知道broker在执行RequestTask时会先进行ACL相关的权限验证。

### 客户端向broker端发送请求时都有哪些ACL方面的操作

客户端向broker端发送请求从通信层面上看在底层是在调用invokeSync、invokeAsync和invokeOneway方法，这些方法都会在发送请求前先执行doBeforeRpcHooks方法，该方法会执行客户端注册的AclClientRPCHook中的doBeforeRequest方法，下面详细看看这个方法都做了哪些操作。
```java
public RemotingCommand invokeSync(String addr, final RemotingCommand request, long timeoutMillis)
        throws InterruptedException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException {
        long beginStartTime = System.currentTimeMillis();
        final Channel channel = this.getAndCreateChannel(addr);
        if (channel != null && channel.isActive()) {
            try {
                doBeforeRpcHooks(addr, request);
                long costTime = System.currentTimeMillis() - beginStartTime;
                if (timeoutMillis < costTime) {
                    throw new RemotingTimeoutException("invokeSync call timeout");
                }
                RemotingCommand response = this.invokeSyncImpl(channel, request, timeoutMillis - costTime);
                doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(channel), request, response);
                return response;
            } catch (RemotingSendRequestException e) {
                log.warn("invokeSync: send request exception, so close the channel[{}]", addr);
                this.closeChannel(addr, channel);
                throw e;
            } catch (RemotingTimeoutException e) {
                if (nettyClientConfig.isClientCloseSocketIfTimeout()) {
                    this.closeChannel(addr, channel);
                    log.warn("invokeSync: close socket because of timeout, {}ms, {}", timeoutMillis, addr);
                }
                log.warn("invokeSync: wait response timeout exception, the channel[{}]", addr);
                throw e;
            }
        } else {
            this.closeChannel(addr, channel);
            throw new RemotingConnectException(addr);
        }
    }

public void invokeAsync(String addr, RemotingCommand request, long timeoutMillis, InvokeCallback invokeCallback)
        throws InterruptedException, RemotingConnectException, RemotingTooMuchRequestException, RemotingTimeoutException,
        RemotingSendRequestException {
        long beginStartTime = System.currentTimeMillis();
        final Channel channel = this.getAndCreateChannel(addr);
        if (channel != null && channel.isActive()) {
            try {
                doBeforeRpcHooks(addr, request);
                long costTime = System.currentTimeMillis() - beginStartTime;
                if (timeoutMillis < costTime) {
                    throw new RemotingTooMuchRequestException("invokeAsync call timeout");
                }
                this.invokeAsyncImpl(channel, request, timeoutMillis - costTime, invokeCallback);
            } catch (RemotingSendRequestException e) {
                log.warn("invokeAsync: send request exception, so close the channel[{}]", addr);
                this.closeChannel(addr, channel);
                throw e;
            }
        } else {
            this.closeChannel(addr, channel);
            throw new RemotingConnectException(addr);
        }
    }

public void invokeOneway(String addr, RemotingCommand request, long timeoutMillis) throws InterruptedException,
        RemotingConnectException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
        final Channel channel = this.getAndCreateChannel(addr);
        if (channel != null && channel.isActive()) {
            try {
                doBeforeRpcHooks(addr, request);
                this.invokeOnewayImpl(channel, request, timeoutMillis);
            } catch (RemotingSendRequestException e) {
                log.warn("invokeOneway: send request exception, so close the channel[{}]", addr);
                this.closeChannel(addr, channel);
                throw e;
            }
        } else {
            this.closeChannel(addr, channel);
            throw new RemotingConnectException(addr);
        }
    }

```

首先parseRequestContent方法会将客户端的AccessKey、SecurityToken以及请求中header的属性进行排序，然后combineRequestContent方法会将上一步返回的map以及请求合并成一个byte数组，calSignature方法会根据客户端的AccessKey以及上一步返回的byte数组生成一个签名，将生成的签名添加到请求的扩展属性中，将AccessKey添加到请求的扩展属性中，
所以在doBeforeRpcHooks完成后客户端的请求中多了两个扩展属性分别是SIGNATURE和ACCESS_KEY，接着客户端通过底层的网络通信将请求发送给broker
```java
public void doBeforeRequest(String remoteAddr, RemotingCommand request) {
        byte[] total = AclUtils.combineRequestContent(request,
            parseRequestContent(request, sessionCredentials.getAccessKey(), sessionCredentials.getSecurityToken()));
        String signature = AclUtils.calSignature(total, sessionCredentials.getSecretKey());
        request.addExtField(SIGNATURE, signature);
        request.addExtField(ACCESS_KEY, sessionCredentials.getAccessKey());
        
        // The SecurityToken value is unneccessary,user can choose this one.
        if (sessionCredentials.getSecurityToken() != null) {
            request.addExtField(SECURITY_TOKEN, sessionCredentials.getSecurityToken());
        }
    }
```
### broker收到客户端发送的请求后如何处理
现在分析broker在收到producer端发送消息后的处理步骤（这里接着上面1中的doBeforeRpcHooks），在broker端会执行在启用ACL后注册的RPCHook中的doBeforeRequest方法：
```java
public void doBeforeRequest(String remoteAddr, RemotingCommand request) {
	//Do not catch the exception
	validator.validate(validator.parse(request, remoteAddr));
}

public void validate(PlainAccessResource plainAccessResource) {

	// Check the global white remote addr
	for (RemoteAddressStrategy remoteAddressStrategy : globalWhiteRemoteAddressStrategy) {
		if (remoteAddressStrategy.match(plainAccessResource)) {
			return;
		}
	}

	if (plainAccessResource.getAccessKey() == null) {
		throw new AclException(String.format("No accessKey is configured"));
	}

	if (!plainAccessResourceMap.containsKey(plainAccessResource.getAccessKey())) {
		throw new AclException(String.format("No acl config for %s", plainAccessResource.getAccessKey()));
	}

	// Check the white addr for accesskey
	PlainAccessResource ownedAccess = plainAccessResourceMap.get(plainAccessResource.getAccessKey());
	if (ownedAccess.getRemoteAddressStrategy().match(plainAccessResource)) {
		return;
	}

	// Check the signature
	String signature = AclUtils.calSignature(plainAccessResource.getContent(), ownedAccess.getSecretKey());
	if (!signature.equals(plainAccessResource.getSignature())) {
		throw new AclException(String.format("Check signature failed for accessKey=%s", plainAccessResource.getAccessKey()));
	}
	// Check perm of each resource

	checkPerm(plainAccessResource, ownedAccess);
}

```
首先调用parse方法从请求命令中解析出该请求访问所需要的访问权限并构建PlainAccessResource对象，调用validate方法进行验证，即将上一步构建的PlainAccessResource对象与当前用户在broker端配置的权限进行对比，如果与配置文件中定义的权限不一致则会抛出AclException

broker端是如何加载plain_acl.yml？
broker是如何watch plain_acl.yml 文件的变化，即热感知如何实现？
```java
public PlainPermissionManager() {
	load();
	watch();
}

public void load() {

	Map<String, PlainAccessResource> plainAccessResourceMap = new HashMap<>();
    List<RemoteAddressStrategy> globalWhiteRemoteAddressStrategy = new ArrayList<>();

    JSONObject plainAclConfData = AclUtils.getYamlDataObject(fileHome + File.separator + fileName,
            JSONObject.class);
    if (plainAclConfData == null || plainAclConfData.isEmpty()) {
            throw new AclException(String.format("%s file is not data", fileHome + File.separator + fileName));
    }
    log.info("Broker plain acl conf data is : ", plainAclConfData.toString());
    JSONArray globalWhiteRemoteAddressesList = plainAclConfData.getJSONArray("globalWhiteRemoteAddresses");
    if (globalWhiteRemoteAddressesList != null && !globalWhiteRemoteAddressesList.isEmpty()) {
		for (int i = 0; i < globalWhiteRemoteAddressesList.size(); i++) {
			globalWhiteRemoteAddressStrategy.add(remoteAddressStrategyFactory.
                        getRemoteAddressStrategy(globalWhiteRemoteAddressesList.getString(i)));
        }
     }

	JSONArray accounts = plainAclConfData.getJSONArray(AclConstants.CONFIG_ACCOUNTS);
    if (accounts != null && !accounts.isEmpty()) {
		List<PlainAccessConfig> plainAccessConfigList = accounts.toJavaList(PlainAccessConfig.class);
        for (PlainAccessConfig plainAccessConfig : plainAccessConfigList) {
			PlainAccessResource plainAccessResource = buildPlainAccessResource(plainAccessConfig);
                plainAccessResourceMap.put(plainAccessResource.getAccessKey(),plainAccessResource);
        }
     }

        // For loading dataversion part just
	JSONArray tempDataVersion = plainAclConfData.getJSONArray(AclConstants.CONFIG_DATA_VERSION);
    if (tempDataVersion != null && !tempDataVersion.isEmpty()) {
		List<DataVersion> dataVersion = tempDataVersion.toJavaList(DataVersion.class);
        DataVersion firstElement = dataVersion.get(0);
        this.dataVersion.assignNewOne(firstElement);
     }

	this.globalWhiteRemoteAddressStrategy = globalWhiteRemoteAddressStrategy;
    this.plainAccessResourceMap = plainAccessResourceMap;
}

private void watch() {
	try {
		String watchFilePath = fileHome + fileName;
        FileWatchService fileWatchService = new FileWatchService(new String[] {watchFilePath}, new FileWatchService.Listener() {
			@Override
            public void onChanged(String path) {
				log.info("The plain acl yml changed, reload the context");
                load();
            }
         });
		fileWatchService.start();
		log.info("Succeed to start AclWatcherService");
		this.isWatchStart = true;
	} catch (Exception e) {
		log.error("Failed to start AclWatcherService", e);
    }
}

public void run() {
        log.info(this.getServiceName() + " service started");

        while (!this.isStopped()) {
            try {
                this.waitForRunning(WATCH_INTERVAL);

                for (int i = 0; i < watchFiles.size(); i++) {
                    String newHash;
                    try {
                        newHash = hash(watchFiles.get(i));
                    } catch (Exception ignored) {
                        log.warn(this.getServiceName() + " service has exception when calculate the file hash. ", ignored);
                        continue;
                    }
                    if (!newHash.equals(fileCurrentHash.get(i))) {
                        fileCurrentHash.set(i, newHash);
                        listener.onChanged(watchFiles.get(i));
                    }
                }
            } catch (Exception e) {
                log.warn(this.getServiceName() + " service has exception. ", e);
            }
        }
        log.info(this.getServiceName() + " service end");
    }
    
private String hash(String filePath) throws IOException, NoSuchAlgorithmException {
        Path path = Paths.get(filePath);
        md.update(Files.readAllBytes(path));
        byte[] hash = md.digest();
        return UtilAll.bytes2string(hash);
    }

```

这两个问题可以在PlainPermissionManager中找到答案，在其构造函数中分别调用了load方法和watch方法。其中load方法是用来加载plain_acl.yml，通过分析可以看出其主要是读取plain_acl.yml中的配置项并将权限相关的配置加载到内存中的plainAccessResourceMap、将全局白名单对应的访问策略添加到globalWhiteRemoteAddressStrategy中；watch方法是用来监听plain_acl.yml文件是否发生变化，这里将相关实现封装成FileWatchService，它继承了ServiceThread，所以其核心实现是在其run方法中，该方法实现的功能是每500ms通过比较plain_acl.yml文件的md5值来判断其内容是否发生变化，如果发生变化则会调用其监听器重新加载该文件（注意：在初始化FileWatchService时会对plain_acl.yml调用hash方法计算其md5值并存储在fileCurrentHash中）。
