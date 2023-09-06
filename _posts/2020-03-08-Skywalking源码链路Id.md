---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码链路Id

## 链路ID生成
定义了traceId的抽象类DistributedTraceId，代码如下：
```
/**
 * The <code>DistributedTraceId</code> presents a distributed call chain.
 * <p>
 * This call chain has a unique (service) entrance,
 * <p>
 * such as: Service : http://www.skywalking.com/cust/query, all the remote, called behind this service, rest remote, db
 * executions, are using the same <code>DistributedTraceId</code> even in different JVM.
 * 在一条链路中(Trace),无论请求分布于多少不同的进程中,这个TraceId都不会改变
 * <p>
 * The <code>DistributedTraceId</code> contains only one string, and can NOT be reset, creating a new instance is the
 * only option.
 */
@RequiredArgsConstructor
@ToString
@EqualsAndHashCode
public abstract class DistributedTraceId {
    @Getter
    private final String id;
}
```
DistributedTraceId有两个实现PropagatedTraceId和NewDistributedTraceId

### PropagatedTraceId
PropagatedTraceId构造函数需要传入traceId并赋值
```
public class PropagatedTraceId extends DistributedTraceId {
    public PropagatedTraceId(String id) {
        super(id);
    }
}
```

### NewDistributedTraceId
NewDistributedTraceId会通过GlobalIdGenerator的generate()方法生成traceId并赋值
```
public class NewDistributedTraceId extends DistributedTraceId {
    public NewDistributedTraceId() {
        super(GlobalIdGenerator.generate());
    }
}
```

GlobalIdGenerator的generate()方法，该方法用于生成TraceId和SegmentId，代码如下：
```
/**
 * 生成唯一ID
 * TraceId SegmentId
 */
public final class GlobalIdGenerator {
    private static final String PROCESS_ID = UUID.randomUUID().toString().replaceAll("-", "");
    private static final ThreadLocal<IDContext> THREAD_ID_SEQUENCE = ThreadLocal.withInitial(
        () -> new IDContext(System.currentTimeMillis(), (short) 0));

    private GlobalIdGenerator() {
    }

    /**
     * 生成一个新的id由三部分组成
     * Generate a new id, combined by three parts.
     * <p>
     * 第一部分表示应用实例的id
     * The first one represents application instance id.
     * <p>
     * 第二部分表示线程id
     * The second one represents thread id.
     * <p>
     * 第三部分包含两部分,一个毫秒级的时间戳+一个当前线程里的序号(0-9999不断循环)
     * The third one also has two parts, 1) a timestamp, measured in milliseconds 2) a seq, in current thread, between
     * 0(included) and 9999(included)
     *
     * @return unique id to represent a trace or segment
     */
    public static String generate() {
        return StringUtil.join(
            '.',
            PROCESS_ID,
            String.valueOf(Thread.currentThread().getId()),
            String.valueOf(THREAD_ID_SEQUENCE.get().nextSeq())
        );
    }
```

生成的traceId由三部分组成，第一部分表示应用实例的Id，是一个UUID；第二部分表示线程Id；第三部分是一个毫秒级的时间戳+一个当前线程里的序号，该序号的范围是0-9999

第三部分调用ThreadLocal中IDContext的nextSeq()方法生成，代码如下：
```
public final class GlobalIdGenerator {

    private static final ThreadLocal<IDContext> THREAD_ID_SEQUENCE = ThreadLocal.withInitial(
        () -> new IDContext(System.currentTimeMillis(), (short) 0));

    private static class IDContext {
        // 上次生成sequence的时间戳
        private long lastTimestamp;
        // 线程的序列号
        private short threadSeq;

        // Just for considering time-shift-back only.
        // 时钟回拨
        private long lastShiftTimestamp;
        private int lastShiftValue;

        private IDContext(long lastTimestamp, short threadSeq) {
            this.lastTimestamp = lastTimestamp;
            this.threadSeq = threadSeq;
        }

        private long nextSeq() {
            // 时间戳 * 10000 + 线程的序列号
            return timestamp() * 10000 + nextThreadSeq();
        }

        private long timestamp() {
            long currentTimeMillis = System.currentTimeMillis();

            if (currentTimeMillis < lastTimestamp) { // 发生了时钟回拨
                // Just for considering time-shift-back by Ops or OS. @hanahmily 's suggestion.
                if (lastShiftTimestamp != currentTimeMillis) {
                    lastShiftValue++;
                    lastShiftTimestamp = currentTimeMillis;
                }
                return lastShiftValue;
            } else { // 时钟正常
                lastTimestamp = currentTimeMillis;
                return lastTimestamp;
            }
        }

        private short nextThreadSeq() {
            if (threadSeq == 10000) {
                threadSeq = 0;
            }
            return threadSeq++;
        }
    }
```






