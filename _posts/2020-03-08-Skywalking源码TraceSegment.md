---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码TraceSegment


## TraceSegment
先来看下TraceSegment中定义的属性：
```
/**
 * {@link TraceSegment} is a segment or fragment of the distributed trace. See https://github.com/opentracing/specification/blob/master/specification.md#the-opentracing-data-model
 * A {@link TraceSegment} means the segment, which exists in current {@link Thread}. And the distributed trace is formed
 * by multi {@link TraceSegment}s, because the distributed trace crosses multi-processes, multi-threads. <p>
 *
 * Trace不是一个具体的数据模型,而是多个Segment串起来表示的逻辑对象
 */
public class TraceSegment {
    /**
     * 每个segment都有一个全局唯一的id
     * The id of this trace segment. Every segment has its unique-global-id.
     */
    private String traceSegmentId;

    /**
     * 指向当前segment的parent segment的指针
     * 对于大部分RPC调用,ref只会包含一个元素.但如果是批处理或者是消息队列,就会有多个parents,这里时候只会保存第一个引用
     * The refs of parent trace segments, except the primary one. For most RPC call, {@link #ref} contains only one
     * element, but if this segment is a start span of batch process, the segment faces multi parents, at this moment,
     * we only cache the first parent segment reference.
     * <p>
     * 这个字段不会被序列化.为了快速访问整条链路保存了parent的引用
     * This field will not be serialized. Keeping this field is only for quick accessing.
     */
    private TraceSegmentRef ref;

```

- traceSegmentId：每个Segment都有一个全局唯一的Id
- ref：指向当前Segment的Parent Segment的指针

TraceSegmentRef的代码如下：
```
@Getter
public class TraceSegmentRef {
    private SegmentRefType type; // segment类型:跨进程或跨线程
    private String traceId;
    private String traceSegmentId; // parent segmentId
    private int spanId; // parent segment spanId
    private String parentService; // Mall -> Order 对于Order服务来讲,parentService就是Mall
    private String parentServiceInstance; // parentService的具体一个实例
    private String parentEndpoint; // 进入parentService的那个请求
    private String addressUsedAtClient;
  
    public enum SegmentRefType {
        CROSS_PROCESS, CROSS_THREAD
    }  

```

```
public class TraceSegment {
  
    /**
     * The spans belong to this trace segment. They all have finished. All active spans are hold and controlled by
     * "skywalking-api" module.
     * 保存segment中所有的span
     */
    private List<AbstractTracingSpan> spans;
  
    /**
     * The <code>relatedGlobalTraceId</code> represent the related trace. Most time it related only one
     * element, because only one parent {@link TraceSegment} exists, but, in batch scenario, the num becomes greater
     * than 1, also meaning multi-parents {@link TraceSegment}. But we only related the first parent TraceSegment.
     * 当前segment所在trace的id
     */
    private DistributedTraceId relatedGlobalTraceId;

```

- spans：保存Segment中所有的Span
- relatedGlobalTraceId：当前Segment所在Trace的Id

TraceSegment中的定义方法如下：
```
public class TraceSegment {
  
    /**
     * 创建一个空的trace segment,生成一个新的segment id
     * Create a default/empty trace segment, with current time as start time, and generate a new segment id.
     */
    public TraceSegment() {
        this.traceSegmentId = GlobalIdGenerator.generate();
        this.spans = new LinkedList<>();
        this.relatedGlobalTraceId = new NewDistributedTraceId();
        this.createTime = System.currentTimeMillis();
    }

    /**
     * 设置当前segment的parent segment
     * Establish the link between this segment and its parents.
     *
     * @param refSegment {@link TraceSegmentRef}
     */
    public void ref(TraceSegmentRef refSegment) {
        if (null == ref) {
            this.ref = refSegment;
        }
    }

    /**
     * 将当前segment关联到某一条trace上
     * Establish the line between this segment and the relative global trace id.
     */
    public void relatedGlobalTrace(DistributedTraceId distributedTraceId) {
        if (relatedGlobalTraceId instanceof NewDistributedTraceId) {
            this.relatedGlobalTraceId = distributedTraceId;
        }
    }

    /**
     * 添加span
     * After {@link AbstractSpan} is finished, as be controller by "skywalking-api" module, notify the {@link
     * TraceSegment} to archive it.
     */
    public void archive(AbstractTracingSpan finishedSpan) {
        spans.add(finishedSpan);
    }

    /**
     * Finish this {@link TraceSegment}. <p> return this, for chaining
     */
    public TraceSegment finish(boolean isSizeLimited) {
        this.isSizeLimited = isSizeLimited;
        return this;
    }

```
finish()需要传入isSizeLimited，SkyWalking中限制了Segment中Span的数量，默认是最大是300，上层调用finish()方法时会判断此时该Segment中的Span是否到达上限

```
public class Config {

    public static class Agent {

        /**
         * The max number of spans in a single segment. Through this config item, SkyWalking keep your application
         * memory cost estimated.
         */
        public static int SPAN_LIMIT_PER_SEGMENT = 300;

```