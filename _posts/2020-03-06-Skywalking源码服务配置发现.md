---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码服务配置发现


## ConfigurationDiscoveryService

```
@DefaultImplementor
public class ConfigurationDiscoveryService implements BootService, GRPCChannelListener {

    private final Register register = new Register();
  
    /**
     * Register dynamic configuration watcher.
     *
     * @param watcher dynamic configuration watcher
     */
    public void registerAgentConfigChangeWatcher(AgentConfigChangeWatcher watcher) {
        WatcherHolder holder = new WatcherHolder(watcher);
        if (register.containsKey(holder.getKey())) {
            throw new IllegalStateException("Duplicate register, watcher=" + watcher);
        }
        register.put(holder.getKey(), holder);
    }

    /**
     * Local dynamic configuration center.
     */
    public static class Register {
        private final Map<String, WatcherHolder> register = new HashMap<>();

        private boolean containsKey(String key) {
            return register.containsKey(key);
        }

        private void put(String key, WatcherHolder holder) {
            register.put(key, holder);
        }

        public WatcherHolder get(String name) {
            return register.get(name);
        }

        public Set<String> keys() {
            return register.keySet();
        }

        @Override
        public String toString() {
            ArrayList<String> registerTableDescription = new ArrayList<>(register.size());
            register.forEach((key, holder) -> {
                AgentConfigChangeWatcher watcher = holder.getWatcher();
                registerTableDescription.add(new StringBuilder().append("key:")
                                                                .append(key)
                                                                .append("value(current):")
                                                                .append(watcher.value()).toString());
            });
            return registerTableDescription.stream().collect(Collectors.joining(",", "[", "]"));
        }
    }

    @Getter
    private static class WatcherHolder {
        private final AgentConfigChangeWatcher watcher;
        private final String key;

        public WatcherHolder(AgentConfigChangeWatcher watcher) {
            this.watcher = watcher;
            this.key = watcher.getPropertyKey();
        }
    }  

```
ConfigurationDiscoveryService中有一个内部类Register，实际上是一个Map，value类型是内部类WatcherHolder，WatcherHolder中包含了AgentConfigChangeWatcher属性，通过registerAgentConfigChangeWatcher()方法注册watcher

AgentConfigChangeWatcher用于监听SkyWalking Agent的某项配置的值的变化，代码如下：
```
/**
 * 监听agent的某项配置的值的变化
 */
@Getter
public abstract class AgentConfigChangeWatcher {
    // Config key, should match KEY in the Table of Agent Configuration Properties.
    // 这个key来源于agent.config,也就是说只有agent配置文件中合法的key才能在这里被使用
    private final String propertyKey;

    public AgentConfigChangeWatcher(String propertyKey) {
        this.propertyKey = propertyKey;
    }

    /**
     * 配置变更通知对应的watcher
     * Notify the watcher, the new value received.
     *
     * @param value of new.
     */
    public abstract void notify(ConfigChangeEvent value);

    /**
     * @return current value of current config.
     */
    public abstract String value();
  
    /**
     * 配置变更事件
     */
    @Getter
    @RequiredArgsConstructor
    public static class ConfigChangeEvent {
        // 新的配置值
        private final String newValue;
        // 事件类型
        private final EventType eventType;
    }

    public enum EventType {
        ADD, MODIFY, DELETE
    }  

```
AgentConfigChangeWatcher有以上四种实现，后面使用到对应的ChangeWatcher时会讲到

再回来看ConfigurationDiscoveryService的相关方法：
```
@DefaultImplementor
public class ConfigurationDiscoveryService implements BootService, GRPCChannelListener {

    private volatile ScheduledFuture<?> getDynamicConfigurationFuture; // 获取动态配置器

    @Override
    public void boot() throws Throwable {
        getDynamicConfigurationFuture = Executors.newSingleThreadScheduledExecutor(
            new DefaultNamedThreadFactory("ConfigurationDiscoveryService")
        ).scheduleAtFixedRate(
            new RunnableWithExceptionProtection(
                this::getAgentDynamicConfig,
                t -> LOGGER.error("Sync config from OAP error.", t)
            ),
            Config.Collector.GET_AGENT_DYNAMIC_CONFIG_INTERVAL,
            Config.Collector.GET_AGENT_DYNAMIC_CONFIG_INTERVAL,
            TimeUnit.SECONDS
        );
    }

```
boot()方法中初始化获取动态配置器getDynamicConfigurationFuture，实际上执行的是ConfigurationDiscoveryService的getAgentDynamicConfig()方法，代码如下：
```
@DefaultImplementor
public class ConfigurationDiscoveryService implements BootService, GRPCChannelListener {
  
    /**
     * UUID of the last return value.
     */
    private String uuid;
    private final Register register = new Register();

    private volatile int lastRegisterWatcherSize; // 上一次计算的watcher的数量
  
    private volatile ConfigurationDiscoveryServiceGrpc.ConfigurationDiscoveryServiceBlockingStub configurationDiscoveryServiceBlockingStub;  

		/**
     * 通过grpc获取Agent动态配置
     * get agent dynamic config through gRPC.
     */
    private void getAgentDynamicConfig() {
        LOGGER.debug("ConfigurationDiscoveryService running, status:{}.", status);

        if (GRPCChannelStatus.CONNECTED.equals(status)) {
            try {
                ConfigurationSyncRequest.Builder builder = ConfigurationSyncRequest.newBuilder();
                builder.setService(Config.Agent.SERVICE_NAME);

                // 有些插件会延迟注册watcher
                // Some plugin will register watcher later.
                final int size = register.keys().size();
                // 如果本次计算的watcher的数量和上一次不相同
                if (lastRegisterWatcherSize != size) {
                    // 当watcher的数量发生了变动,代表有新的配置项需要监听
                    // 重置uuid,避免同样的uuid导致配置信息没有被更新
                    // reset uuid, avoid the same uuid causing the configuration not to be updated.
                    uuid = null;
                    lastRegisterWatcherSize = size;
                }

                if (null != uuid) {
                    builder.setUuid(uuid);
                }

                if (configurationDiscoveryServiceBlockingStub != null) {
                    // 拉取配置 反序列化后为ConfigurationDiscoveryCommand类型
                    final Commands commands = configurationDiscoveryServiceBlockingStub.withDeadlineAfter(
                        GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS
                    ).fetchConfigurations(builder.build());
                    ServiceManager.INSTANCE.findService(CommandService.class).receiveCommand(commands);
                }
            } catch (Throwable t) {
                LOGGER.error(t, "ConfigurationDiscoveryService execute fail.");
                ServiceManager.INSTANCE.findService(GRPCChannelManager.class).reportError(t);
            }
        }
    }
```
getAgentDynamicConfig()方法处理逻辑如下：

- 因为有些插件会延迟注册watcher，当watcher的数量发生了变动，代表有新的配置项需要监听，需要重置uuid，避免同样的uuid导致配置信息没有被更新
- 调用ConfigurationDiscoveryServiceBlockingStub的fetchConfigurations()方法拉取配置，调用CommandService的receiveCommand()将命令添加到待处理命令列表，该方法拿到的command序列化后为ConfigurationDiscoveryCommand类型，走执行command中讲的流程，最终由ConfigurationDiscoveryService的handleConfigurationDiscoveryCommand()来处理配置变更的命令

ConfigurationDiscoveryService的handleConfigurationDiscoveryCommand()方法代码如下：
```
@DefaultImplementor
public class ConfigurationDiscoveryService implements BootService, GRPCChannelListener {
  
    /**
     * Process ConfigurationDiscoveryCommand and notify each configuration watcher.
     *
     * @param configurationDiscoveryCommand Describe dynamic configuration information
     */
    public void handleConfigurationDiscoveryCommand(ConfigurationDiscoveryCommand configurationDiscoveryCommand) {
        final String responseUuid = configurationDiscoveryCommand.getUuid();

        // uuid相同说明配置没有变化
        if (responseUuid != null && Objects.equals(this.uuid, responseUuid)) {
            return;
        }

        // configurationDiscoveryCommand可能返回了10个配置变更
        // 但是register里面只有5个配置项的监听器
        // 这里就要过滤出有监听器的配置项
        List<KeyStringValuePair> config = readConfig(configurationDiscoveryCommand);

        config.forEach(property -> {
            String propertyKey = property.getKey();
            WatcherHolder holder = register.get(propertyKey);
            if (holder != null) {
                AgentConfigChangeWatcher watcher = holder.getWatcher();
                String newPropertyValue = property.getValue();
                if (StringUtil.isBlank(newPropertyValue)) {
                    if (watcher.value() != null) {
                        // 通知watcher,删除该配置
                        // Notify watcher, the new value is null with delete event type.
                        watcher.notify(
                            new AgentConfigChangeWatcher.ConfigChangeEvent(
                                null, AgentConfigChangeWatcher.EventType.DELETE
                            ));
                    } else {
                        // Don't need to notify, stay in null.
                    }
                } else {
                    if (!newPropertyValue.equals(watcher.value())) {
                        // 通知watcher,配置变更
                        watcher.notify(new AgentConfigChangeWatcher.ConfigChangeEvent(
                            newPropertyValue, AgentConfigChangeWatcher.EventType.MODIFY
                        ));
                    } else {
                        // Don't need to notify, stay in the same config value.
                    }
                }
            } else {
                LOGGER.warn("Config {} from OAP, doesn't match any watcher, ignore.", propertyKey);
            }
        });
      	// 更新uuid
        this.uuid = responseUuid;

        LOGGER.trace("Current configurations after the sync, configurations:{}", register.toString());
    }  
```
handleConfigurationDiscoveryCommand()方法处理逻辑如下：

- 如果uuid相同，说明配置没有变化，直接返回
- 过滤出有监听器的配置项，判断如果配置变更或者被删除，通知watcher，最后更新uuid
