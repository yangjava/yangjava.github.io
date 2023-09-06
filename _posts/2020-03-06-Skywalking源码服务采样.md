---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码服务采样


## 服务-SamplingService
SamplingService是来控制是否要采样该链路。每条链路都是被追踪到的，但是考虑到序列化/反序列化的CPU消耗以及网络带宽，如果开启采样，SkyWalking Agent并不会把所有的链路都发送给OAP。

默认采样是开启的，可以通过修改agent.config中的agent.sample_n_per_3_secs配置项控制每3秒最多采样多少条链路

```
/**
 * SamplingService是来控制是否要采样该链路
 * 每条链路都是被追踪到的,但是考虑到序列化/反序列化的CPU消耗以及网络带宽,如果开启采样,agent并不会把所有的链路都发送给OAP
 * The <code>SamplingService</code> take charge of how to sample the {@link TraceSegment}. Every {@link TraceSegment}s
 * have been traced, but, considering CPU cost of serialization/deserialization, and network bandwidth, the agent do NOT
 * send all of them to collector, if SAMPLING is on.
 * <p>
 * 默认采样是开启的
 * By default, SAMPLING is on, and  {@link Config.Agent#SAMPLE_N_PER_3_SECS }
 * SAMPLE_N_PER_3_SECS:每3秒最多采样多少条链路,如果配置的是负数或者0就表示采样机制关闭(即所有的链路都会被采集并发送)
 */
@DefaultImplementor
public class SamplingService implements BootService {

    private SamplingRateWatcher samplingRateWatcher;
  
    @Override
    public void boot() {
        samplingRateWatcher = new SamplingRateWatcher("agent.sample_n_per_3_secs", this);
        // 注册配置变更监听器
        ServiceManager.INSTANCE.findService(ConfigurationDiscoveryService.class)
                               .registerAgentConfigChangeWatcher(samplingRateWatcher);

        handleSamplingRateChanged();
    }
```
boot()方法中初始化SamplingRateWatcher，并向ConfigurationDiscoveryService注册配置变更监听器

SamplingRateWatcher是AgentConfigChangeWatcher的子类，监听的是agent.sample_n_per_3_secs这个动态配置
```
public class SamplingRateWatcher extends AgentConfigChangeWatcher {
    private static final ILog LOGGER = LogManager.getLogger(SamplingRateWatcher.class);

    private final AtomicInteger samplingRate; // 每3秒能够采集的最大链路数
    private final SamplingService samplingService;

    public SamplingRateWatcher(final String propertyKey, SamplingService samplingService) {
        super(propertyKey);
        this.samplingRate = new AtomicInteger(getDefaultValue());
        this.samplingService = samplingService;
    }

    private void activeSetting(String config) {
        if (LOGGER.isDebugEnable()) {
            LOGGER.debug("Updating using new static config: {}", config);
        }
        try {
            this.samplingRate.set(Integer.parseInt(config));

            /*
             * 通知samplingService每3秒能够采集的最大链路数的配置变更
             * We need to notify samplingService the samplingRate changed.
             */
            samplingService.handleSamplingRateChanged();
        } catch (NumberFormatException ex) {
            LOGGER.error(ex, "Cannot load {} from: {}", getPropertyKey(), config);
        }
    }

    @Override
    public void notify(final ConfigChangeEvent value) {
        if (EventType.DELETE.equals(value.getEventType())) {
            // 如果删除动态配置,使用agent.config中的配置
            activeSetting(String.valueOf(getDefaultValue()));
        } else {
            activeSetting(value.getNewValue());
        }
    }

    private int getDefaultValue() {
        return Config.Agent.SAMPLE_N_PER_3_SECS;
    }
```

如果该配置项发生变更，会调用SamplingRateWatcher的notify()方法，SamplingRateWatcher修改配置后，再调用SamplingService的handleSamplingRateChanged()方法，通知每3秒能够采集的最大链路数的配置变更
```
@DefaultImplementor
public class SamplingService implements BootService {
    private static final ILog LOGGER = LogManager.getLogger(SamplingService.class);

    private volatile boolean on = false;
    private volatile AtomicInteger samplingFactorHolder; // 用于累加3秒内已经采样的次数
    private volatile ScheduledFuture<?> scheduledFuture; // 每3秒重置一次samplingFactorHolder

    /**
     * Handle the samplingRate changed.
     */
    public void handleSamplingRateChanged() {
        if (samplingRateWatcher.getSamplingRate() > 0) {
            if (!on) {
                on = true;
                this.resetSamplingFactor();
                ScheduledExecutorService service = Executors.newSingleThreadScheduledExecutor(
                    new DefaultNamedThreadFactory("SamplingService"));
                scheduledFuture = service.scheduleAtFixedRate(new RunnableWithExceptionProtection(
                    this::resetSamplingFactor, t -> LOGGER.error("unexpected exception.", t)), 0, 3, TimeUnit.SECONDS);
                LOGGER.debug(
                    "Agent sampling mechanism started. Sample {} traces in 3 seconds.",
                    samplingRateWatcher.getSamplingRate()
                );
            }
        } else {
            if (on) {
                if (scheduledFuture != null) {
                    scheduledFuture.cancel(true);
                }
                on = false;
            }
        }
    }
  
    private void resetSamplingFactor() {
        samplingFactorHolder = new AtomicInteger(0);
    }  

```

handleSamplingRateChanged()方法中判断如果开启采样，初始化定时任务，每3秒重置一次samplingFactorHolder，samplingFactorHolder是用于累加3秒内已经采样的次数

每一条链路是否要采样取决于SamplingService的trySampling()方法的返回，如果返回值为true，则该链路会被采样，trySampling()方法代码如下：
```
@DefaultImplementor
public class SamplingService implements BootService {
  
    private volatile AtomicInteger samplingFactorHolder; // 用于累加3秒内已经采样的次数
  
    /**
     * 如果采样机制没有开启,即on=false,那么就表示每一条采集到的链路都会上报给OAP
     * @param operationName The first operation name of the new tracing context.
     * @return true, if sampling mechanism is on, and getDefault the sampling factor successfully.
     */
    public boolean trySampling(String operationName) {
        if (on) {
            int factor = samplingFactorHolder.get();
            // 3秒内已经采样的次数<每3秒能够采集的最大链路数
            if (factor < samplingRateWatcher.getSamplingRate()) {
                // 3秒内已经采样的次数+1,CAS修改成功则该链路会被采样
                return samplingFactorHolder.compareAndSet(factor, factor + 1);
            } else {
                return false;
            }
        }
        return true;
    }

```

SamplingService的`forceSampled()方法强制进行采样，代码如下：
```
@DefaultImplementor
public class SamplingService implements BootService {

    /**
     * Increase the sampling factor by force, to avoid sampling too many traces. If many distributed traces require
     * sampled, the trace beginning at local, has less chance to be sampled.
     */
    public void forceSampled() {
        if (on) {
            samplingFactorHolder.incrementAndGet();
        }
    }

```














