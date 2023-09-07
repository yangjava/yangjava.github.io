---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码接收链路
Skywalking对Trace信息进行生成收集后，将TraceSegment对象转换为UpstreamSegment对象，通过GRPC发送给OAP服务端，服务端处理对应的模块是skywalking-trace-receiver-plugin模块。

## 模块结构
```text
org.apache.skywalking.oap.server.receiver.trace
├─module                      # 模块定义
├─├─TraceModule               # TraceModule模块定义

├─provider                    # 
├─├─TraceModuleProvider       # 
├─├─handler.v8.grpc             # 
├─├─├─├─TraceSegmentReportServiceHandler

├─├─handler.v8.rest            # 
├─├─├─├─TraceSegmentReportListServletHandler
```
## 协议定义
proto定义文件位于 apm-protocol/apm-network/src/main/java/proto/language-agent/Tracing.proto 中

定义的服务
```
service TraceSegmentReportService {
   
    rpc collect (stream SegmentObject) returns (Commands) {
    }

    // 异步批量收集
    rpc collectInSync (SegmentCollection) returns (Commands) {
    }
}
```

## TraceModule

## TraceModuleProvider
TraceModuleProvider向GRPCHandlerRegister中添加处理器TraceSegmentReportServiceHandler

协议 gpc 实现
服务实现注册在 TraceModuleProvider#start 中, 完成 grpc service 服务注册
```
@Override
public void start() {
        // 读取 grpc 注册器
        GRPCHandlerRegister grpcHandlerRegister = getManager().find(SharingServerModule.NAME)
        .provider()
        .getService(GRPCHandlerRegister.class);
        ...
        // 注册服务
        TraceSegmentReportServiceHandler traceSegmentReportServiceHandler = new TraceSegmentReportServiceHandler(getManager());
        grpcHandlerRegister.addHandler(traceSegmentReportServiceHandler);
        grpcHandlerRegister.addHandler(new TraceSegmentReportServiceHandlerCompat(traceSegmentReportServiceHandler));

        ...
        }
```
可以看到实际调用服务为 TraceSegmentReportServiceHandler, 调用 ISegmentParserService, 目前实现类只有 SegmentParserServiceImpl 进行解析

```
public class TraceSegmentReportServiceHandler extends TraceSegmentReportServiceGrpc.TraceSegmentReportServiceImplBase implements GRPCHandler {
    ...
    private ISegmentParserService segmentParserService;
    ...
     @Override
    public StreamObserver<SegmentObject> collect(StreamObserver<Commands> responseObserver) {
        return new StreamObserver<SegmentObject>() {
            @Override
            public void onNext(SegmentObject segment) {
                ...
                try {
                    segmentParserService.send(segment);
                } catch (Exception e) {
                    errorCounter.inc();
                    log.error(e.getMessage(), e);
                } finally {
                    timer.finish();
                }
            }

           ...
        };
    }
```
## 接收Agent数据
TraceSegmentReportServiceHandler的collect()方法接收Agent的数据，调用SegmentParseV2.Producer的send()方法发送

SegmentParseV2.Producer的send()方法：
```
public void send(UpstreamSegment segment, SegmentSource source) {
    SegmentParseV2 segmentParse = new SegmentParseV2(moduleManager, listenerManager, config);
    segmentParse.setStandardizationWorker(standardizationWorker);
    segmentParse.parse(new BufferData<>(segment), source);
}
```
- 创建SegmentParseV2解析器
- 调用SegmentParseV2的parse()方法

## SegmentParseV2的parse()方法：


- 创建SpanListener集合
- 获取UpstreamSegment对象
- 获取UpstreamSegment对象中关联的所有的TraceID
- 如果UpstreamSegment中的SegmentObject实例为空，就解析UpstreamSegment实例得到SegmentObject对象进行填充
- 重新检查段信息是否来自文件缓冲区，如果缓存中不存这个段信息对应的服务实例Id，然后返回true
- 把SegmentObject对象封装成SegmentDecorator对象，这里是装饰者模式的体现
- 调用preBuild()方法进行预构建操作，预构建不成功写入缓存文件中，构建成功会通知具体的监听器来进行构建


SegmentParserServiceImpl 只是个包装类, 实际调用 TranceAnalyzer 进行具体解析
```
@RequiredArgsConstructor
public class SegmentParserServiceImpl implements ISegmentParserService {
    private final ModuleManager moduleManager;
    private final AnalyzerModuleConfig config;
    // 
    @Setter
    private SegmentParserListenerManager listenerManager;

    @Override
    public void send(SegmentObject segment) {
        final TraceAnalyzer traceAnalyzer = new TraceAnalyzer(moduleManager, listenerManager, config);
        traceAnalyzer.doAnalysis(segment);
    }
}


public class TraceAnalyzer {
    ...
    private List<AnalysisListener> analysisListeners = new ArrayList<>();
    
    public void doAnalysis(SegmentObject segmentObject) {
        if (segmentObject.getSpansList().size() == 0) {
            return;
        }

        createSpanListeners();

        notifySegmentListener(segmentObject);
        // 根据 Segment 的 span 类型, 调用不同的 Listener 进行处理
        segmentObject.getSpansList().forEach(spanObject -> {
            // 判断 span 是否为首次进入, 
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

        // 通知 Listener 执行构建, 保存db 等操作
        notifyListenerToBuild();
    }
    
    private void notifyListenerToBuild() {
        // 调用 sourceReceiver#receive 进行消息处理
        analysisListeners.forEach(AnalysisListener::build);
    }
}
```
















