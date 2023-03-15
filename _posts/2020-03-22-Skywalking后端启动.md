---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
# Skywalking后端启动


启动OAP非常简单，OAP的代码是源码根目录下的oap-server，入口函数是 org.apache.skywalking.oap.server.starter 包下面的OAPServerStartUp类。直接启动即可。

Agent和OAP之间是通过gRPC来发送链路信息的。Agent端维护了一个队列（默认5个channel，每个channel大小为300）和一个线程池（默认1个线程，后面称为发送线程），链路数据采集后主线程（即业务线程）会写入这个队列，如果队列满了，主线程会直接把把数据丢掉（丢的时候会以debug级别打印日志）。发送线程会从队列取数据通过gRPC发送给后端OAP，OAP经过处理后写入存储。为了看得清楚，我把涉及的框架类画到了下面的图里面（格式是： {类名}#{方法名}({方法中调用的重要函数} )：

org.apache.skywalking.oap.server.receiver.trace.provider.handler.v8.grpc.TraceSegmentReportServiceHandler#collect

其内部功能很多，从是Segment这个数据流程来说就是：构建多个监听器，以监听器的模式来通过解析segmentObject各个属性，通过构建SourceBuilder对象来承载上下游的链路相关信息，并添加到entrySourceBuilders中；在build环节，进一步构建成各维度的souce数据，包括Trace(链路),Metrics(调用统计如调用次数，pxx，响应时长等) 信息都在这个环节创建。先大致看下其代码主体流程，接下来会分析内部更多的细节逻辑：

- SegmentAnalysisListener#parseSegment构建Segment（Source）,部分属性赋值
  1.1 赋值 起止时间
  1.2 赋值 是否error
  1.3 赋值 是否采样，这里是重点
- SegmentAnalysisListener#notifyFirstListener 更多的属性赋值
- 多个EntryAnalysisListener监听器处理Entry类型的span
  3.1 SegmentAnalysisListener#parseEntry赋值service和endpoint的Name和id
- 3.2 NetworkAddressAliasMappingListener#parseEntry 构造NetworkAddressAliasSetup完善ip_port地址与别名之间的映射关心，交给NetworkAddressAliasSetupDispatcher处理
- 3.3 MultiScopesAnalysisListener#parseEntry 遍历span列表
- 3.3.1 将每个span构建成SourceBuilder，设置上下游的游的Server、Instance、endpoint的name信息，这里mq和网关特殊处理，其上游保持ip端口，因为mq、网关通常没有搭载agent，没有相关的name信息。
- 3.3.2 setPublicAttrs：SourceBuilder中添加 tag信息，重点是时间bucket,setResponseCode，Status，type(http,rpc,db)
- 3.3.3 SourceBuilder添加到entrySourceBuilders,
- 3.3.4 parseLogicEndpoints//处理span的tag是LOGIC_ENDPOINT = "x-le"类型的，添加到 logicEndpointBuilders中（用途待梳理)
- MultiScopesAnalysisListener#parseExit监听器处理Exit类型的span
  4.1 将span构建成SourceBuilder，设置上下游的游的Server、Instance、Endpoint的name信息，尝试把下游的ip_port信息修改成别名。
  4.2 setPublicAttrs：SourceBuilder中添加 tag信息，重点是时间bucket,setResponseCode，Status，type(http,rpc,db)
  4.3 SourceBuilder添加到exitSourceBuilders，
  4.4 如果是db类型，构造slowStatementBuilder，判断时长设置慢查询标识，存入dbSlowStatementBuilders中。这里是全局的阈值 是个改造点。
- MultiScopesAnalysisListener#parseLocal监听器处理Local类型的span,通过parseLogicEndpoints方法处理span的tag是LOGIC_ENDPOINT = "x-le"类型的，添加到 logicEndpointBuilders中（用途待梳理)
- 6.1 SegmentAnalysisListener#build,设置endpoint的 id 和name，然后将Segment交给SourceReceiver#receive处理，而SourceReceiver#receive就是调用dispatcherManager#forward,最终交给SegmentDispatcher#dispatch处理了，
  6.2 MultiScopesAnalysisListener#build中根据以上流程中创建的数据，会再构造出多种Metric 类型的Source数据交给SourceReceiver处理；这些逻辑在这篇笔记中不展开，本篇已Segment流程为主


## 源码解读








#




## Skywalking OAP源码

#### OAP流程

启动了OAP。Agent和OAP之间是通过gRPC来发送链路信息的。Agent端维护了一个队列（默认5个channel，每个channel大小为300）和一个线程池（默认1个线程，后面称为发送线程），链路数据采集后主线程（即业务线程）会写入这个队列，如果队列满了，主线程会直接把把数据丢掉（丢的时候会以debug级别打印日志）。发送线程会从队列取数据通过gRPC发送给后端OAP，OAP经过处理后写入存储。

![Skywalking-OAP](F:/work/openGuide/Middleware/Skywalking-OAP.png)



#### OAPServer启动

由server-starter和server-starter-es7调用server-bootstrap
server-starter和server-starter-es7的区别在于maven中引入的存储模块Module不同

| 启动器             | 插件                          |
| ------------------ | ----------------------------- |
| server-starter     | storage-elasticsearch-plugin  |
| server-starter-es7 | storage-elasticsearch7-plugin |

ModuleDefine与maven模块关系

- ModuleDefine一般对应一个模块
- ModuleProvider一般对应一个模块的子模块
- 比如配置模块,server-configuration下多个模块对应configuration模块的不同provider实现

| ModuleDefine名称    | application.yml 名称 | maven模块         |
| ------------------- | -------------------- | ----------------- |
| ConfigurationModule | configuration: none: | configuration-api |

类名设计

| 名称                     | 作用                                                         |
| ------------------------ | ------------------------------------------------------------ |
| ApplicationConfigLoader  | 负责加载application.yml                                      |
| ApplicationConfiguration | 将application.yml转化为ApplicationConfiguration              |
| ModuleManager            | 管理所有的Module                                             |
| ModuleDefine             | 一个ModuleDefine代表一个模块,比如存储模块,UI查询query模块,JVM模块等等;不同的模块相互依赖或者不依赖构建整个oapServer功能 |
| ModuleProvider           | ModuleDefine的具体实现,比如StorageModule包含es存储实现,mysql实现等等 |
| Service                  | 多个service构成一个完整的ModuleProvider,也就是将module的具体实现拆分成多个serviceImpl |

启动架构图
两个starter模块依赖es7或者低版本es,启动时根据es版本决定启动who
调用server-bootstrap启动
解析application.yml生成ApplicationConfiguration
ModuleManager根据ApplicationConfiguration加载所有的ModuleDefine以及对应的loadedModuleProvider
执行prepare,start,notifyAfterCompleted完成所有模块的启动

OAPServerBootstrap

```
public class OAPServerBootstrap {
    public static void start() {
    // 初始化mode为init或者no-init,表示是否初始化[example:底层存储组件等]
        String mode = System.getProperty("mode");
        RunningMode.setMode(mode);
      // 初始化ApplicationConfigurationd的加载器和Module的加载管理器
        ApplicationConfigLoader configLoader = new ApplicationConfigLoader();
        ModuleManager manager = new ModuleManager();
        try {
        // 加载yml生成ApplicationConfiguration配置
        
            ApplicationConfiguration applicationConfiguration = configLoader.load();
            
            // 初始化模块 通过spi获取所有Module实现,基于yml配置加载spi中存在的相关实现
            manager.init(applicationConfiguration);

            manager.find(TelemetryModule.NAME)
                   .provider()
                   .getService(MetricsCreator.class)
                   .createGauge("uptime", "oap server start up time", MetricsTag.EMPTY_KEY, MetricsTag.EMPTY_VALUE)
                   // Set uptime to second
                   .setValue(System.currentTimeMillis() / 1000d);

            if (RunningMode.isInitMode()) {
                log.info("OAP starts up in init mode successfully, exit now...");
                System.exit(0);
            }
        } catch (Throwable t) {
            log.error(t.getMessage(), t);
            System.exit(1);
        }
    }
}
```

#### 源码分析-OAP数据接收

启动时CoreModuleProvider核心provider会初始化GRPCServer, 只留下GRPCServer相关的核心代码

```
# CoreModuleProvider
public void prepare() {
    if (moduleConfig.isGRPCSslEnabled()) {
        grpcServer = new GRPCServer(moduleConfig.getGRPCHost(), moduleConfig.getGRPCPort(),
                                    moduleConfig.getGRPCSslCertChainPath(),
                                    moduleConfig.getGRPCSslKeyPath()
        );
    } else {
        grpcServer = new GRPCServer(moduleConfig.getGRPCHost(), moduleConfig.getGRPCPort());
    }
    if (moduleConfig.getMaxConcurrentCallsPerConnection() > 0) {
        grpcServer.setMaxConcurrentCallsPerConnection(moduleConfig.getMaxConcurrentCallsPerConnection());
    }
    if (moduleConfig.getMaxMessageSize() > 0) {
        grpcServer.setMaxMessageSize(moduleConfig.getMaxMessageSize());
    }
    if (moduleConfig.getGRPCThreadPoolQueueSize() > 0) {
        grpcServer.setThreadPoolQueueSize(moduleConfig.getGRPCThreadPoolQueueSize());
    }
    if (moduleConfig.getGRPCThreadPoolSize() > 0) {
        grpcServer.setThreadPoolSize(moduleConfig.getGRPCThreadPoolSize());
    }
    grpcServer.initialize();

}
```

GRPCServer 用于接收 SkyWalking Agent 发送的 gRPC 请求，SkyWalking Agent 会切分OAP服务列表配置项，得到 OAP 服务列表，然后从其中随机选择一个 OAP 服务创建长连接，实现后续的数据上报

GRPCServer 处理 gRPC 请求的逻辑，封装在了 ServerHandler 实现之中的。我们可以通过两者的 addHandler() 方法，为指定请求添加相应的 ServerHandler 实现, 比如常见的TraceSegmentReportServiceHandler上报trace数据，就是利用对应的handler处理的

##### 默认GRPC接收

TraceSegmentReportServiceHandler接收的agent的方法，只留核心的代码

**TraceSegmentReportServiceHandler**

```
    @Override
    public StreamObserver<SegmentObject> collect(StreamObserver<Commands> responseObserver) {
        return new StreamObserver<SegmentObject>() {
            @Override
            public void onNext(SegmentObject segment) {
                if (log.isDebugEnabled()) {
                    log.debug("received segment in streaming");
                }

                HistogramMetrics.Timer timer = histogram.createTimer();
                try {
                    segmentParserService.send(segment);
                } catch (Exception e) {
                    errorCounter.inc();
                    log.error(e.getMessage(), e);
                } finally {
                    timer.finish();
                }
            }

            @Override
            public void onError(Throwable throwable) {
                log.error(throwable.getMessage(), throwable);
                responseObserver.onCompleted();
            }

            @Override
            public void onCompleted() {
                responseObserver.onNext(Commands.newBuilder().build());
                responseObserver.onCompleted();
            }
        };
    }
```

通过GRPC接受发送过来的数据，调用 `ISegmentParserService#send#send(UpstreamSegment)` 方法，处理**一条** TraceSegment 。

**SegmentParserServiceImpl**`

```
@RequiredArgsConstructor
public class SegmentParserServiceImpl implements ISegmentParserService {
    private final ModuleManager moduleManager;
    private final AnalyzerModuleConfig config;
    @Setter
    private SegmentParserListenerManager listenerManager;

    @Override
    public void send(SegmentObject segment) {
        final TraceAnalyzer traceAnalyzer = new TraceAnalyzer(moduleManager, listenerManager, config);
        traceAnalyzer.doAnalysis(segment);
    }
}
```

TraceAnalyzer中doAnalysis方法: 各个listen是构建存储model，notifyListenerToBuild才触发存储，比如es

```
public void doAnalysis(SegmentObject segmentObject) {
    if (segmentObject.getSpansList().size() == 0) {
        return;
    }
    // 创建span的listener
    createSpanListeners();
    // 通知segment监听，构建存储model
    notifySegmentListener(segmentObject);
    // 通知 first exit local的各个listen，构建存储model
    segmentObject.getSpansList().forEach(spanObject -> {
        if (spanObject.getSpanId() == 0) {
            notifyFirstListener(spanObject, segmentObject);
        }

        if (SpanType.Exit.equals(spanObject.getSpanType())) {
            notifyExitListener(spanObject, segmentObject);
        } else if (SpanType.Entry.equals(spanObject.getSpanType())) {
            notifyEntryListener(spanObject, segmentObject);
        } else if (SpanType.Local.equals(spanObject.getSpanType())) {
            notifyLocalListener(spanObject, segmentObject);
        } else {
            log.error("span type value was unexpected, span type name: {}", spanObject.getSpanType()
                                                                                      .name());
        }
    });
    // 触发存储动作
    notifyListenerToBuild();
}

private void notifyListenerToBuild() {
    analysisListeners.forEach(AnalysisListener::build);
}
```

SegmentAnalysisListener.build方法存储

```
// 去掉非核心代码
public void build() {
    segment.setEndpointId(endpointId);
    segment.setEndpointName(endpointName);
    // 存储核心
    sourceReceiver.receive(segment);
}
```

SourceReceiverImpl.receive方法

```
public void receive(ISource source) {
     dispatcherManager.forward(source);
}
```

DispatcherManager.forward方法，只留下核心代码

```
public void forward(ISource source) {
    source.prepare();
    for (SourceDispatcher dispatcher : dispatchers) {
        dispatcher.dispatch(source);
    }   
}
```

SegmentDispatcher.dispatch方法，存储数据，比如es

```
public void dispatch(Segment source) {
    SegmentRecord segment = new SegmentRecord();
    segment.setSegmentId(source.getSegmentId());
    segment.setTraceId(source.getTraceId());
    segment.setServiceId(source.getServiceId());
    segment.setServiceInstanceId(source.getServiceInstanceId());
    segment.setEndpointName(source.getEndpointName());
    segment.setEndpointId(source.getEndpointId());
    segment.setStartTime(source.getStartTime());
    segment.setEndTime(source.getEndTime());
    segment.setLatency(source.getLatency());
    segment.setIsError(source.getIsError());
    segment.setDataBinary(source.getDataBinary());
    segment.setTimeBucket(source.getTimeBucket());
    segment.setVersion(source.getVersion());
    segment.setTagsRawData(source.getTags());
    segment.setTags(Tag.Util.toStringList(source.getTags()));

    RecordStreamProcessor.getInstance().in(segment);
}
```

最终会调RecordPersistentWorker.in存储

```
public void in(Record record) {
    InsertRequest insertRequest = recordDAO.prepareBatchInsert(model, record);
    batchDAO.asynchronous(insertRequest);    
}
```

最后到es存储BatchProcessEsDAO, 调es的bulk

```
public void asynchronous(InsertRequest insertRequest) {
    if (bulkProcessor == null) {
        this.bulkProcessor = getClient().createBulkProcessor(bulkActions, flushInterval, concurrentRequests);
    }

    this.bulkProcessor.add((IndexRequest) insertRequest);
}
```



#### TraceSegmentServiceHandler

调用 `ITraceSegmentService#send(UpstreamSegment)` 方法，处理**一条** TraceSegment 。

## TraceSegmentService

`org.skywalking.apm.collector.agent.stream.service.trace.ITraceSegmentService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，TraceSegment 服务接口。

- 定义了 [`#send(UpstreamSegment)`](https://github.com/YunaiV/skywalking/blob/c15cf5e1356c7b44a23f2146b8209ab78c2009ac/apm-collector/apm-collector-agent-stream/collector-agent-stream-define/src/main/java/org/skywalking/apm/collector/agent/stream/service/trace/ITraceSegmentService.java#L31) **接口**方法，处理**一条** TraceSegment 。

`org.skywalking.apm.collector.agent.stream.worker.trace.ApplicationIDService` ，实现 IApplicationIDService 接口，TraceSegment 服务实现类。

- 实现了 [`#send(UpstreamSegment)`](https://github.com/YunaiV/skywalking/blob/c15cf5e1356c7b44a23f2146b8209ab78c2009ac/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/TraceSegmentService.java#L39) 方法，代码如下：
    - 第 40 至 41 行：创建 SegmentParse 对象，后调用 `SegmentParse#parse(UpstreamSegment, Source)` 方法，解析并处理 TraceSegment 。

## SegmentParse

[`org.skywalking.apm.collector.agent.stream.parser.SegmentParse`](https://github.com/YunaiV/skywalking/blob/c15cf5e1356c7b44a23f2146b8209ab78c2009ac/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/SegmentParse.java) ，Segment 解析器。属性如下：

- `spanListeners` 属性，Span 监听器集合。**通过不同的监听器，对 TraceSegment 进行构建，生成不同的数据**。在 [`#SegmentParse(ModuleManager)` 构造方法](https://github.com/YunaiV/skywalking/blob/c15cf5e1356c7b44a23f2146b8209ab78c2009ac/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/SegmentParse.java#L72) ，会看到它的初始化。
- `segmentId` 属性，TraceSegment 编号，即 [`TraceSegment.traceSegmentId`](https://github.com/YunaiV/skywalking/blob/c15cf5e1356c7b44a23f2146b8209ab78c2009ac/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/TraceSegment.java#L47) 。
- `timeBucket` 属性，第一个 Span 的开始时间。

[`#parse(UpstreamSegment, Source)`](https://github.com/YunaiV/skywalking/blob/c15cf5e1356c7b44a23f2146b8209ab78c2009ac/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/SegmentParse.java#L86) 方法，解析并处理 TraceSegment 。在该方法里，我们会看到，本文开头提到的【**构造**】。整个构造的过程，实际分成**两步**：1）预构建；2）执行构建。代码如下：

- 第 88 至 89 行：从



  ```
  segment
  ```



参数中，解析出 ：

- `traceIds` ，关联的链路追踪**全局编号**。
- `segmentObject` ，TraceSegmentObject 对象。

- 第 91 行：创建 SegmentDecorator 对象。该对象的用途，在 [「2.3 Standardization 标准化」](https://www.iocoder.cn/SkyWalking/collector-receive-trace/#) 统一解析。

- ——– 构建失败 ——–

- 第 94 行：调用 `#preBuild(List<UniqueId>, SegmentDecorator)` 方法，**预构建**。

- 第 97 至 99 行：调用



  ```
  #writeToBufferFile()
  ```



方法，将 TraceSegment 写入 Buffer 文件

暂存

。为什么会判断



  ```
  source == Source.Agent
  ```



呢？

  ```
  #parse(UpstreamSegment, Source)
  ```



方法的调用，共有

两个



Source



：

- 目前我们看到 TraceSegmentService 的调用使用的是 `Source.Agent` 。
- 而后台线程，定时调用该方法重新构建使用的是 `Source.Buffer` ，如果不加盖判断，会预构建失败**重复**写入。

- 第 100 行：返回 `false` ，表示构建失败。

- ——– 构建成功 ——–

- 第 106 行：调用 [`#notifyListenerToBuild()`](https://github.com/YunaiV/skywalking/blob/c15cf5e1356c7b44a23f2146b8209ab78c2009ac/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/SegmentParse.java#L199) 方法，通知 Span 监听器们，**执行构建**各自的数据。在 [《SkyWalking 源码解析 —— Collector 存储 Trace 数据》](http://www.iocoder.cn/SkyWalking/collector-store-trace/?self) 详细解析。

- 第 109 行：调用 [`buildSegment(id, dataBinary)`](https://github.com/YunaiV/skywalking/blob/c15cf5e1356c7b44a23f2146b8209ab78c2009ac/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/SegmentParse.java#L177) 方法，**执行构建** TraceSegment 。

- 第 110 行：返回 `true` ，表示构建成功。

- 第 112 至 115 行：发生 InvalidProtocolBufferException 异常，返回 `false` ，表示构建失败。

# [Collector 存储 Trace 数据](https://www.iocoder.cn/SkyWalking/collector-store-trace/)

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [2. SpanListener](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [3. GlobalTrace](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [4. InstPerformance](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [5. SegmentCost](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [6. NodeComponent](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [7. NodeMapping](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [8. NodeReference](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [9. ServiceEntry](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [10. ServiceReference](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [11. Segment](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/collector-store-trace/)

------