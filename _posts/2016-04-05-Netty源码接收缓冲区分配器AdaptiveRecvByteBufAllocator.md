---
layout: post
categories: [Netty]
description: none
keywords: Netty
---
# Netty源码接收缓冲区分配器AdaptiveRecvByteBufAllocator

## 概述
我们知道为了提高性能，Netty使用内存池、ByteBuf对象池等技术优化了内存的使用，后面也会有文章对此进行介绍。这篇文章主要介绍Netty在客户端或者服务端进行read操作时是如何决定需要分配的ByteBuf大小的。通过ByteBuf的实现源码我们可以知道，Netty实现的ByteBuf和JDK的许多集合类一样，如果分配的初始容量小于写入数据的字节数的话，是会进行自动扩容的，比如下面AbstractByteBuf的代码：
```
//AbstractByteBuf
@Override
public int writeBytes(ScatteringByteChannel in, int length) throws IOException {
    //这里确保容量足够，如果不够的话会触发扩容动作
    ensureWritable(length);
    int writtenBytes = setBytes(writerIndex, in, length);
    if (writtenBytes > 0) {
        writerIndex += writtenBytes;
    }
    return writtenBytes;
}

@Override
public ByteBuf ensureWritable(int minWritableBytes) {
    ...
    ensureWritable0(minWritableBytes);
    return this;
}

final void ensureWritable0(int minWritableBytes) {
    ensureAccessible();
    if (minWritableBytes <= writableBytes()) {
        return;
    }

    ...

    //将新的容量大小进行规范化，规范化为2的指数
    // Normalize the current capacity to the power of 2.
    int newCapacity = alloc().calculateNewCapacity(writerIndex + minWritableBytes, maxCapacity);

    //调整ByteBuf为新的容量，如果是采用Netty内存池分配的ByteBuf则
    //会触发reallocate
    // Adjust to the new capacity.
    capacity(newCapacity);
}
```
可见，如果分配的ByteBuf容量小于实际所需（实际需要读取）的会触发ByteBuf的自动扩容操作，涉及到内存的拷贝、或者直接内存空间的分配，代价比较大，所以每次读取之前准备好足够大小的ByteBuf也能适当的提高性能，足够大小指的是准备的ByteBuf不能过大造成空间浪费；也不能太小，造成读取时的自动扩容。

为此，Netty默认在读取收到的数据时，使用了具有自适应的AdaptiveRecvByteBufAllocator决定每次分配的ByteBuf的大小，本文主要介绍AdaptiveRecvByteBufAllocator相关机制。

## RecvByteBufAllocator接口相关实现介绍
RecvByteBufAllocator从下面源码注释客户自，主要用于分配一个足够小且能容纳所有需要读取数据的接收缓冲区。
```
Allocates a new receive buffer whose capacity is probably large enough to read all inbound data and small enough not to waste its space.
```
RecvByteBufAllocator只有一个返回返回一个Handle接口，Handle是RecvByteBufAllocator内部声明的一个接口，进行实际的缓冲区分配工作，其实这里也不能说是实际的缓冲区分配工作，因为Handle.allocate方法入参为ByteBufAllocator，ByteBufAllocator才是实际分配缓冲区的对象，Handle仅仅告诉ByteBufAllocator应该分配多大的缓冲区，也就是决定缓冲区的尺寸。
```
//RecvByteBufAllocator
public interface RecvByteBufAllocator {
    /**
    * Creates a new handle.  The handle provides the actual operations and keeps the internal information which is
    * required for predicting an optimal buffer capacity.
    */
    Handle newHandle();
    /**
     * @deprecated Use {@link ExtendedHandle}.
     */
     //Handle中的一些方法主要用于对历史分配的缓冲区大小和实际读取
     //大小的统计,用于对下次分配大小进行指导，后面会具体介绍
    @Deprecated
    interface Handle {
        /**
         * Creates a new receive buffer whose capacity is probably large enough to read all inbound data and small
         * enough not to waste its space.
         */
        ByteBuf allocate(ByteBufAllocator alloc);

        /**
         * Similar to {@link #allocate(ByteBufAllocator)} except that it does not allocate anything but just tells the
         * capacity.
         */
        int guess();

        /**
         * Reset any counters that have accumulated and recommend how many messages/bytes should be read for the next
         * read loop.
         * <p>
         * This may be used by {@link #continueReading()} to determine if the read operation should complete.
         * </p>
         * This is only ever a hint and may be ignored by the implementation.
         * @param config The channel configuration which may impact this object's behavior.
         */
        void reset(ChannelConfig config);

        /**
         * Increment the number of messages that have been read for the current read loop.
         * @param numMessages The amount to increment by.
         */
        void incMessagesRead(int numMessages);

        /**
         * Set the bytes that have been read for the last read operation.
         * This may be used to increment the number of bytes that have been read.
         * @param bytes The number of bytes from the previous read operation. This may be negative if an read error
         * occurs. If a negative value is seen it is expected to be return on the next call to
         * {@link #lastBytesRead()}. A negative value will signal a termination condition enforced externally
         * to this class and is not required to be enforced in {@link #continueReading()}.
         */
        void lastBytesRead(int bytes);

        /**
         * Get the amount of bytes for the previous read operation.
         * @return The amount of bytes for the previous read operation.
         */
        int lastBytesRead();

        /**
         * Set how many bytes the read operation will (or did) attempt to read.
         * @param bytes How many bytes the read operation will (or did) attempt to read.
         */
        void attemptedBytesRead(int bytes);

        /**
         * Get how many bytes the read operation will (or did) attempt to read.
         * @return How many bytes the read operation will (or did) attempt to read.
         */
        int attemptedBytesRead();

        /**
         * Determine if the current read loop should should continue.
         * @return {@code true} if the read loop should continue reading. {@code false} if the read loop is complete.
         */
        boolean continueReading();

        /**
         * The read has completed.
         */
        void readComplete();
}
```
我们这里主要看

RecvByteBufAllocator

->MaxMessagesRecvByteBufAllocator

->DefaultMaxMessagesRecvByteBufAllocator

->AdaptiveRecvByteBufAllocator

这条继承线路。

MaxMessagesRecvByteBufAllocator接口主要功能可以见其注释，用于限制每次read函数被调用时从channel中读取的次数，因为预测的字节数可能较小，read函数使用循环从channel中读取数据，如果数据量较多的话则可能需要迭代多次，这里限制的读取次数就是迭代的次数，限制次数之后，可能进入read函数之后，并不会读取channel中的全部数据，剩下的数据会在下次进入read函数再次被读取。

read函数是在Unsafe中实现的，后面会列出其源码，这里先有个印象，就是read函数其实是通过循环读取channel中的数据的。

that limits the number of read operations that will be attempted when a read operation is attempted by the event loop.

DefaultMaxMessagesRecvByteBufAllocator是一个抽象类，是MaxMessagesRecvByteBufAllocator的一个部分实现，内部主要使用成员变量maxMessagesPerRead控制每次进入Unsafe.read函数的读取次数，Netty中默认的次数为1，可见DefaultMaxMessagesRecvByteBufAllocator构造函数：
```
//DefaultMaxMessagesRecvByteBufAllocator
public DefaultMaxMessagesRecvByteBufAllocator() {
    this(1);
}

public DefaultMaxMessagesRecvByteBufAllocator(int maxMessagesPerRead) {
    maxMessagesPerRead(maxMessagesPerRead);
}
```
子类AdaptiveRecvByteBufAllocator调用的就是DefaultMaxMessagesRecvByteBufAllocator的默认构造函数，所以每次进入'Unsafe.read'函数while循环只会进行一次，也就是只会根据分配的ByteBuf从channel中读取一次数据。

DefaultMaxMessagesRecvByteBufAllocator也实现了RecvByteBufAllocator内部声明的接口Handle，因为源码不是特别长，列在下面，并对相关的字段和方法进行简单介绍：
```
/**
* Focuses on enforcing the maximum messages per read condition for {@link #continueReading()}.
*/
public abstract class MaxMessageHandle implements ExtendedHandle {
    private ChannelConfig config;
    //每次调用Unsafe.read从channel中进行读取的次数
    //这个字段默认为1，在reset方法中会调用外部类
    //DefaultMaxMessagesRecvByteBufAllocator.maxMessagesPerRead
    //方法进行初始化，DefaultMaxMessagesRecvByteBufAllocator上面
    //介绍过，该字段默认为1.
    private int maxMessagePerRead;
    //Unsafe.read方法中，循环读取的次数
    //每次进入Unsafe.read进行循环读取前会调用reset进行置0操作。
    private int totalMessages;
    //Unsafe.read方法中，进行循环读取的总字节数，同样
    //会在读取前调用reset函数被置0
    private int totalBytesRead;
    //在进行实际读取时，会调用如下方法
    //allocHandle.attemptedBytesRead(byteBuf.writableBytes());
    //获取该ByteBuf可以读取的字节数，并使用该数值对attemptedBytesRead
    //进行赋值，也就是此次循环迭代ByteBuf剩下可用于读取字节的数量
    private int attemptedBytesRead;
    //记录上次迭代或者最后一次迭代实际读取的字节数
    private int lastBytesRead;
    //这个字段看了源码没有实际使用，不介绍
    private final boolean respectMaybeMoreData = DefaultMaxMessagesRecvByteBufAllocator.this.respectMaybeMoreData;
    private final UncheckedBooleanSupplier defaultMaybeMoreSupplier = new UncheckedBooleanSupplier() {
        @Override
        public boolean get() {
            return attemptedBytesRead == lastBytesRead;
        }
    };

    /**
    * Only {@link ChannelConfig#getMaxMessagesPerRead()} is used.
    */
    //每次进入Unsafe.read函数进行循环读取前，会调用reset函数，对
    //Handle中的一些统计数值进行复位重置
    @Override
    public void reset(ChannelConfig config) {
        this.config = config;
        maxMessagePerRead = maxMessagesPerRead();
        totalMessages = totalBytesRead = 0;
    }

    //分配根据统计数据预测大小的ByteBuf
    @Override
    public ByteBuf allocate(ByteBufAllocator alloc) {
        return alloc.ioBuffer(guess());
    }

    //在Unsafe.read函数的每次循环迭代中，每读取一次
    //都会调用incMessagesRead(1)增加已经读取的次数
    @Override
    public final void incMessagesRead(int amt) {
        totalMessages += amt;
    }

    //在Unsafe.read函数的每次循环迭代从channel进行读取
    //之后，都会记录本次读取的字节数并累计总共读取的字节数
    @Override
    public void lastBytesRead(int bytes) {
        lastBytesRead = bytes;
        if (bytes > 0) {
            totalBytesRead += bytes;
        }
    }

    @Override
    public final int lastBytesRead() {
        return lastBytesRead;
    }

    //判断是否继续读取，这里主要是根据配置限制每次Unsafe.read函数
    //中循环迭代的次数。
    @Override
    public boolean continueReading() {
        return continueReading(defaultMaybeMoreSupplier);
    }

    //次数限制的具体实现，这里主要看
    //totalMessages < maxMessagePerRead && totalBytesRead > 0
    //也就是循环迭代次数小于指定的最大次数并且总共读取的字节数大于0
    //因为maxMessagePerRead默认为1，所以Unsafe.read函数中循环只会迭代
    //一次。
    @Override
    public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {
        return config.isAutoRead() &&
                (!respectMaybeMoreData || maybeMoreDataSupplier.get()) &&
                totalMessages < maxMessagePerRead &&
                totalBytesRead > 0;
    }

    @Override
    public void readComplete() {
    }

    @Override
    public int attemptedBytesRead() {
        return attemptedBytesRead;
    }

    @Override
    public void attemptedBytesRead(int bytes) {
        attemptedBytesRead = bytes;
    }

    protected final int totalBytesRead() {
        return totalBytesRead < 0 ? Integer.MAX_VALUE : totalBytesRead;
    }
}
```
## AdaptiveRecvByteBufAllocator实现
AdaptiveRecvByteBufAllocator没有什么具体的工作，主要定义了一个缓冲区分配大小的序列、最大、最小以及初始化（没有任何统计数据时的第一次分配）应该分配的缓冲区大小。

源码注释说明如下：
```
The {@link RecvByteBufAllocator} that automatically increases and decreases the predicted buffer size on feed back. It gradually increases the expected number of readable bytes if the previous read fully filled the allocated buffer. It gradually decreases the expected number of readable bytes if the read operation was not able to fill a certain amount of the allocated buffer two times consecutively. Otherwise, it keeps returning the same prediction.
```
大意为根据每次实际读取的反馈动态的增大或缩小下次预测所需分配的缓冲区大小数值。如果读取的实际字节数正好等于本次预测的缓冲区可写入字节数，则扩大预测下次需要分配的缓冲区大小；如果连续两次实际读取的字节数没有填满本次预测的缓冲区可写入字节数，则缩小预测值。其他情况则不改变预测值。

首先看其内部变量：
```
//AdaptiveRecvByteBufAllocator
//默认的最小缓冲区大小
static final int DEFAULT_MINIMUM = 64;
//第一次分配的缓冲区大小
static final int DEFAULT_INITIAL = 1024;
//最大缓冲区大小
static final int DEFAULT_MAXIMUM = 65536;

//如果上次预测的缓冲区太小，造成实际读取字节数大于上次预测的缓冲区
//大小的话，这里用于指定应该增加的步进大小
private static final int INDEX_INCREMENT = 4;
//这个和上面相反，如果上次预测的缓冲区太大，造成实际读取字节数小于上次预测的缓冲区大小的话，这里用于指定应该减少的步进大小
private static final int INDEX_DECREMENT = 1;

//缓冲区大小序列，SIZE_TABLE数组从小到大保存了所有可以分配的缓冲区
//大小，上面说的步进其实就是SIZE_TABLE数组下标，下标越大对应的缓冲区越大
private static final int[] SIZE_TABLE;

//下面静态初始化块对SIZE_TABLE进行初始化，
//初始化之后为：
//[16, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224,
// 240, 256, 272, 288, 304, 320, 336, 352, 368, 384, 400, 416, 
// 432, 448, 464, 480, 496, 512, 1024, 2048, 4096, 8192, 16384, 
// 32768, 65536, 131072, 262144, 524288, 1048576, 2097152, 
// 4194304, 8388608, 16777216, 33554432, 67108864, 134217728, 
// 268435456, 536870912, 1073741824]
//小于512之前因为缓冲区容量较小，降低步进值，采用每次增加16字节进行分配，
//大于512之后则进行加倍分配每次分配缓冲区容量较大，为了减少动态扩张的频率
//采用加倍的快速步进分配。

static {
    List<Integer> sizeTable = new ArrayList<Integer>();
    for (int i = 16; i < 512; i += 16) {
        sizeTable.add(i);
    }

    for (int i = 512; i > 0; i <<= 1) {
        sizeTable.add(i);
    }

    SIZE_TABLE = new int[sizeTable.size()];
    for (int i = 0; i < SIZE_TABLE.length; i ++) {
        SIZE_TABLE[i] = sizeTable.get(i);
    }
}
```
AdaptiveRecvByteBufAllocator还实现了如下方法，根据预测的大小，使用二分查找法从SIZE_TABLE中获取预测缓冲区大小对应的最近接的一个值，用于缓冲区的实际分配：
```
//AdaptiveRecvByteBufAllocator
private static int getSizeTableIndex(final int size) {
    for (int low = 0, high = SIZE_TABLE.length - 1;;) {
        if (high < low) {
            return low;
        }
        if (high == low) {
            return high;
        }

        int mid = low + high >>> 1;
        int a = SIZE_TABLE[mid];
        int b = SIZE_TABLE[mid + 1];
        if (size > b) {
            low = mid + 1;
        } else if (size < a) {
            high = mid - 1;
        } else if (size == a) {
            return mid;
        } else {
            return mid + 1;
        }
    }
}
```
AdaptiveRecvByteBufAllocator构造函数如下：
```
//三个字段
//最小缓冲区内存规范为SIZE_TABLE中数值的对应的下标
private final int minIndex;
//最大缓冲区内存规范为SIZE_TABLE中数值的对应的下标
private final int maxIndex;
//默认的分配的缓冲区大小
private final int initial;
public AdaptiveRecvByteBufAllocator() {
    //上面列出的默认值，分别为64 1024 65536
    this(DEFAULT_MINIMUM, DEFAULT_INITIAL, DEFAULT_MAXIMUM);
}

//根据默认值对minIndex/maxIndex/initial进行初始化
public AdaptiveRecvByteBufAllocator(int minimum, int initial, int maximum) {
    if (minimum <= 0) {
        throw new IllegalArgumentException("minimum: " + minimum);
    }
    if (initial < minimum) {
        throw new IllegalArgumentException("initial: " + initial);
    }
    if (maximum < initial) {
        throw new IllegalArgumentException("maximum: " + maximum);
    }

    int minIndex = getSizeTableIndex(minimum);
    if (SIZE_TABLE[minIndex] < minimum) {
        this.minIndex = minIndex + 1;
    } else {
        this.minIndex = minIndex;
    }

    int maxIndex = getSizeTableIndex(maximum);
    if (SIZE_TABLE[maxIndex] > maximum) {
        this.maxIndex = maxIndex - 1;
    } else {
        this.maxIndex = maxIndex;
    }

    this.initial = initial;
}
```
除此之外，AdaptiveRecvByteBufAllocator内部类HandleImpl还实现了DefaultMaxMessagesRecvByteBufAllocator.MaxMessageHandle类如下：
```
//AdaptiveRecvByteBufAllocator
private final class HandleImpl extends MaxMessageHandle {
    private final int minIndex;
    private final int maxIndex;
    private int index;
    private int nextReceiveBufferSize;
    private boolean decreaseNow;

    //构造函数的三个参数分别对应上面AdaptiveRecvByteBufAllocator
    //最小缓冲区内存规范为SIZE_TABLE中数值的对应的下标、
    //最大缓冲区内存规范为SIZE_TABLE中数值的对应的下标
    //以及默认的分配的缓冲区大小
    public HandleImpl(int minIndex, int maxIndex, int initial) {
        this.minIndex = minIndex;
        this.maxIndex = maxIndex;
        //获取默认缓冲大小规范为SIZE_TABLE中数值对应的小标
        index = getSizeTableIndex(initial);
        nextReceiveBufferSize = SIZE_TABLE[index];
    }

    //Unsafe.read函数每次循环迭代调用该方法，传入每次循环迭代实际读
    //的字节数量
    @Override
    public void lastBytesRead(int bytes) {
        // If we read as much as we asked for we should check if we need to ramp up the size of our next guess.
        // This helps adjust more quickly when large amounts of data is pending and can avoid going back to
        // the selector to check for more data. Going back to the selector can add significant latency for large
        // data transfers.
        //如果读取字节数正好等于读取之前缓冲区可供写入的字节数的话
        //调用record方法调整下次分配的缓冲区大小
        if (bytes == attemptedBytesRead()) {
            record(bytes);
        }
        //调用父类方法进行统计计数，即MaxMessageHandle.lastBytesRead
        super.lastBytesRead(bytes);
    }

    //返回此次预测的下次需分配的缓冲区大小
    @Override
    public int guess() {
        return nextReceiveBufferSize;
    }

    //根据实际读取的字节数，预测下次应该分配的缓冲区大小
    private void record(int actualReadBytes) {
        //尝试减小下次分配的缓冲区大小，如果此次实际读取的字节数
        //小于减小之后的值则减小下标，下次分配的缓冲区将减少
        if (actualReadBytes <= SIZE_TABLE[max(0, index - INDEX_DECREMENT - 1)]) {
            //decreaseNow则实现上面注释说明的
            //要求连续两次读取实际字节数恰好等于当前分配
            //缓冲区剩余可用大小才进行所缩小调整
            if (decreaseNow) {
                index = max(index - INDEX_DECREMENT, minIndex);
                nextReceiveBufferSize = SIZE_TABLE[index];
                decreaseNow = false;
            } else {
                decreaseNow = true;
            }
        //下面的else if则表明实际读取的大小大于上次预测的缓冲区大小，
        //需要扩大预测的数值，扩大不要求满足梁旭两次都大于预测值。
        } else if (actualReadBytes >= nextReceiveBufferSize) {
            index = min(index + INDEX_INCREMENT, maxIndex);
            nextReceiveBufferSize = SIZE_TABLE[index];
            decreaseNow = false;
        }
    }

    //Unsafe.read方法所有循环迭代结束，调用测方法进行一次预测，为下次
    //进入Unsafe.read方法分配缓冲区做准备
    //下次进入Unsafe.read方法会对Handle的一些统计数值进行复位，但是
    //不会修改预测的nextReceiveBufferSize的值。
    @Override
    public void readComplete() {
        record(totalBytesRead());
    }
}
```
HandleImpl的父类MaxMessageHandle中allocate方法通过调用guess方法获取上次预测的大小进行缓冲区分配，源码上面已经列出，这里再次列在下面：
```
//MaxMessageHandle
@Override
public ByteBuf allocate(ByteBufAllocator alloc) {
    return alloc.ioBuffer(guess());
}
```