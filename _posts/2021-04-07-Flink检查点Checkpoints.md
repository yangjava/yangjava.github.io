---
layout: post
categories: [Flink]
description: none
keywords: Flink
---
## Flink检查点Checkpoints

## Checkpoints检查点机制
Flink中基于异步轻量级的分布式快照技术提供了Checkpoints容错机制，分布式快照可以将同一时间点Task/Operator的状态数据全局统一快照处理，包括前面提到的Keyed State和Operator State。Flink会在输入的数据集上间隔性地生成checkpoint barrier，通过栅栏（barrier）将间隔时间段内的数据划分到相应的checkpoint中。当应用出现异常时，Operator就能够从上一次快照中恢复所有算子之前的状态，从而保证数据的一致性。

例如在KafkaConsumer算子中维护Offset状态，当系统出现问题无法从Kafka中消费数据时，可以将Offset记录在状态中，当任务重新恢复时就能够从指定的偏移量开始消费数据。对于状态占用空间比较小的应用，快照产生过程非常轻量，高频率创建且对Flink任务性能影响相对较小。checkpoint过程中状态数据一般被保存在一个可配置的环境中，通常是在JobManager节点或HDFS上。

## Flink Checkpoint 的运行机制
Checkpoint 整个流程如下：
- JM 定时调度 Checkpoint 的触发：JM CheckpointCoorinator 定时触发，CheckpointCoordinator 会去通过 RPC 接口调用 Source 算子的 TM 的 StreamTask 告诉 TM 可以开始执行 Checkpoint 了。
- Source 算子：接受到 JM 做 Checkpoint 的请求后，开始做本地 Checkpoint，本地执行完成之后，发 barrier 给下游算子。barrier 发送策略是随着 partition 策略走，将 barrier 发往连接到的所有下游算子（举例：keyby 就是广播，forward 就是直接送）。
- 剩余的算子：接收到上游所有 barrier 之后进行触发 Checkpoint。当一个算子接收到上游一个 channel 的 barrier 之后，就停止处理这个 input channel 来的数据（本质上就是不会再去影响状态了）

## Flink Checkpoint 的配置
默认情况下Flink不开启检查点的，用户需要在程序中通过调用enable-Checkpointing(n)方法配置和开启检查点，其中n为检查点执行的时间间隔，单位为毫秒。除了配置检查点时间间隔，针对检查点配置还可以调整其他相关参数：
```java

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 每 30 秒触发一次 checkpoint，checkpoint 时间应该远小于（该值 + MinPauseBetweenCheckpoints），否则程序会一直做 checkpoint，影响数据处理速度
env.enableCheckpointing(30000);
 
// Flink 框架内保证 EXACTLY_ONCE 
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
 
// 两个 checkpoints 之间最少有 30s 间隔（上一个 checkpoint 完成到下一个 checkpoint 开始，默认为 0，这里建议设置为非 0 值）
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(30000);
 
// checkpoint 超时时间（默认 600 s）
env.getCheckpointConfig().setCheckpointTimeout(600000);
 
// 同时只有一个checkpoint运行（默认）
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
 
// 取消作业时是否保留 checkpoint (默认不保留，非常建议配置为保留)
env.getCheckpointConfig().enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
 
// checkpoint 失败时 task 是否失败（默认 true， checkpoint 失败时，task 会失败）
env.getCheckpointConfig().setFailOnCheckpointingErrors(true);
 
// 对 FsStateBackend 刷出去的文件进行文件压缩，减小 checkpoint 体积
env.getConfig().setUseSnapshotCompression(true);

```

- Checkpoint开启和时间间隔指定
开启检查点并且指定检查点时间间隔为1000ms，根据实际情况自行选择，如果状态比较大，则建议适当增加该值。
```java
env.enableCheckpointing(1000);
```
- exactly-ance和at-least-once语义选择
可以选择exactly-once语义保证整个应用内端到端的数据一致性，这种情况比较适合于数据要求比较高，不允许出现丢数据或者数据重复，与此同时，Flink的性能也相对较弱，而at-least-once语义更适合于时廷和吞吐量要求非常高但对数据的一致性要求不高的场景。
如下通过setCheckpointingMode()方法来设定语义模式，默认情况下使用的是exactly-once模式。
```java
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
```
- Checkpoint超时时间
超时时间指定了每次Checkpoint执行过程中的上限时间范围，一旦Checkpoint执行时间超过该阈值，Flink将会中断Checkpoint过程，并按照超时处理。该指标可以通过setCheckpointTimeout方法设定，默认为10分钟。
```java
env.getCheckpointConfig().setCheckpointTimeout(60000);
```
- 检查点之间最小时间间隔
该参数主要目的是设定两个Checkpoint之间的最小时间间隔，防止出现例如状态数据过大而导致Checkpoint执行时间过长，从而导致Checkpoint积压过多，最终Flink应用密集地触发Checkpoint操作，会占用了大量计算资源而影响到整个应用的性能。
```java
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);
```
- 最大并行执行的检查点数量
通过setMaxConcurrentCheckpoints()方法设定能够最大同时执行的Checkpoint数量。在默认情况下只有一个检查点可以运行，根据用户指定的数量可以同时触发多个Checkpoint，进而提升Checkpoint整体的效率。
```java
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
```
- 外部检查点
设定周期性的外部检查点，然后将状态数据持久化到外部系统中，使用这种方式不会在任务正常停止的过程中清理掉检查点数据，而是会一直保存在外部系统介质中，另外也可以通过从外部检查点中对任务进行恢复。
```java
env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
```
- failOnCheckpointingErrors
failOnCheckpointingErrors参数决定了当Checkpoint执行过程中如果出现失败或者错误时，任务是否同时被关闭，默认值为True。
```java
env.getCheckpointConfig().setFailOnCheckpointingErrors (false);
```
建议大家的 Checkpoint 配置如下：
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 每 120 秒触发一次 checkpoint，不会特别频繁
env.enableCheckpointing(120000);
 
// Flink 框架内保证 EXACTLY_ONCE
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
 
// 两个 checkpoints 之间最少有 120s 间隔
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(120000);
 
// checkpoint 超时时间 600s
env.getCheckpointConfig().setCheckpointTimeout(600000);
 
// 同时只有一个 checkpoint 运行
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
 
// 取消作业时保留 checkpoint，因为有时候任务 savepoint 可能不可用，这时我们就可以直接从 checkpoint 重启任务
env.getCheckpointConfig().enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
 
// checkpoint 失败时 task 不失败，因为可能会有偶尔的写入 HDFS 失败，但是这并不会影响我们任务的运行
// 偶尔的由于网络抖动 checkpoint 失败可以接受，但是如果经常失败就要定位具体的问题！
env.getCheckpointConfig().setFailOnCheckpointingErrors(false);
```


## Savepoints机制
Savepoints是检查点的一种特殊实现，底层其实也是使用Checkpoints的机制。Savepoints是用户以手工命令的方式触发Checkpoint，并将结果持久化到指定的存储路径中，其主要目的是帮助用户在升级和维护集群过程中保存系统中的状态数据，避免因为停机运维或者升级应用等正常终止应用的操作而导致系统无法恢复到原有的计算状态的情况，从而无法实现端到端的Excatly-Once语义保证。

### Operator ID配置
当使用Savepoints对整个集群进行升级或运维操作的时候，需要停止整个Flink应用程序，此时用户可能会对应用的代码逻辑进行修改，即使Flink能够通过Savepoint将应用中的状态数据同步到磁盘然后恢复任务，但由于代码逻辑发生了变化，在升级过程中有可能导致算子的状态无法通过Savepoints中的数据恢复的情况，在这种情况下就需要通过唯一的ID标记算子。在Flink中默认支持自动生成Operator ID，但是这种方式不利于对代码层面的维护和升级，建议用户尽可能使用手工的方式对算子进行唯一ID标记，ID的应用范围在每个算子内部，
具体的使用方式可以通过使用Operator中提供的uid方法指定唯一ID，这样就能将算子唯一区分出来。
```java
DataStream<String> stream = env.
  // Stateful source (e.g. Kafka) with ID
  .addSource(new StatefulSource())
  .uid("source-id") // ID for the source operator
  .shuffle()
  // Stateful mapper with ID
  .map(new StatefulMapper())
  .uid("mapper-id") // ID for the mapper
  // Stateless printing sink
  .print(); // Auto-generated ID
```
## Savepoints操作
Savepoint操作可以通过命令行的方式进行触发，命令行提供了取消任务、从Savepoints中恢复任务、撤销Savepoints等操作，在Flink1.2版本以后也可以通过Flink Web页面从Savepoints中恢复应用。

### 手动触发Savepoints
通过在Flink命令中指定“savepoint”关键字来触发Savepoints操作，同时需要在命令中指定jobId和targetDirectory（两个参数），其中jobId是需要触发Savepoints操作的Job Id编号，targetDirectory指定Savepoint数据存储路径，所有Savepoint存储的数据都会放置在该配置路径中。
```java
bin/flink savepoint :jobId [:targetDirectory]
```
在Hadoop Yarn上提交的应用，需要指定Flink jobId的同时也需要通过使用yid指定YarnAppId，其他参数和普通模式一样。
```java
bin/flink savepoint :jobId [:targetDirectory] -yid :yarnAppId
```
### 取消任务并触发Savepoints
通过cancel命令将停止Flink任务的同时将自动触发Savepoints操作，并把中间状态数据写入磁盘，用以后续的任务恢复。

```java
bin/flink cancel -s [:targetDirectory] :jobId
```
### 通过从Savepoints中恢复任务
```java
bin/flink run -s :savepointPath [:runArgs]
```
通过使用run命令将任务从保存的Savepoint中恢复，其中-s参数指定了Savepoint数据存储路径。通常情况下Flink通过使用savepoint可以恢复应用中的状态数据，但在某些情况下如果应用中的算子和Savepoint中的算子状态可能不一致，例如用户在新的代码中删除了某个算子，这时就会出现任务不能恢复的情况，此时可以通过--allowNonRestoredState（--n）参数来设置忽略状态无法匹配的问题，让程序能够正常启动和运行。

### 释放Savepoints数据
```java
bin/flink savepoint -d :savepointPath
```
可以通过以上--dispose（-d）命令释放已经存储的Savepoint数据，这样存储在指定路径中的savepointPath将会被清除掉。

## TargetDirectory配置
在前面的内容中我们已经知道Flink在执行Savepoints过程中需要指定目标存储路径，但对目标路径配置除了可以在每次生成Savepoint，通过在命令行中指定之外，也可以在系统环境中配置默认的TargetDirectory，这样就不需要每次在命令行中指定。需要注意，默认路径和命令行中必须至少指定一个，否则无法正常执行Savepoint过程。
### 默认TargetDirecy配置
在flink-conf.yaml配置文件中配置state.savepoints.dir参数，配置的路径参数需要是TaskManager和JobManager都能够访问到的路径，例如分布式文件系统Hdfs的路径。
```java
state.savepoints.dir: hdfs:///flink/savepoints
```
### TargetDirectoy文件路径结构
TargetDirectoy文件路径结构如下，其中包括Savepoint directory以及metadata路径信息。需要注意，即便Flink任务的Savepoints数据存储在一个路径中，但目前还不支持将Savepoint路径拷贝到另外的环境中，然后通过Savepoint恢复Flink任务的操作，这主要是因为在_metadata文件中使用到了绝对路径，这点会在Flink未来的版本中得到改善，也就是支持不同环境中的Flink任务的恢复和迁移。
```
# Savepoint target directory
/savepoints/
# Savepoint directory
/savepoints/savepoint-:shortjobid-:savepointid/
# Savepoint file contains the checkpoint meta data
/savepoints/savepoint-:shortjobid-:savepointid/_metadata
# Savepoint state
/savepoints/savepoint-:shortjobid-:savepointid/...
```

## 状态管理器
在Flink中提供了StateBackend来存储和管理Checkpoints过程中的状态数据。

## StateBackend类别
Flink中一共实现了三种类型的状态管理器，包括基于内存的MemoryStateBackend、基于文件系统的FsStateBackend，以及基于RockDB作为存储介质的RocksDBState-Backend。这三种类型的StateBackend都能够有效地存储Flink流式计算过程中产生的状态数据，在默认情况下Flink使用的是内存作为状态管理器，下面分别对每种状态管理器的特点进行说明。

### MemoryStateBackend
原理：运行时所需的 State 数据全部保存在 TaskManager JVM 堆上内存中，执行 Checkpoint 的时候，会把 State 的快照数据保存到 JobManager 进程 的内存中。执行 Savepoint 时，可以把 State 存储到文件系统中。

基于内存的状态管理器将状态数据全部存储在JVM堆内存中，包括用户在使用DataStream API中创建的Key/Value State，窗口中缓存的状态数据，以及触发器等数据。基于内存的状态管理具有非常快速和高效的特点，但也具有非常多的限制，最主要的就是内存的容量限制，一旦存储的状态数据过多就会导致系统内存溢出等问题，从而影响整个应用的正常运行。同时如果机器出现问题，整个主机内存中的状态数据都会丢失，进而无法恢复任务中的状态数据。因此从数据安全的角度建议用户尽可能地避免在生产环境中使用MemoryStateBackend。

Flink将MemoryStateBackend作为默认状态后端管理器，也可以通过如下参数配置初始化MemoryStateBackend，其中“MAX_MEM_STATE_SIZE”指定每个状态值最大的内存使用大小。
```java
new MemoryStateBackend(MAX_MEM_STATE_SIZE, false);
```
在Flink中MemoryStateBackend具有如下特点，需要用户在选择使用中注意：
- 聚合类算子的状态会存储在JobManager内存中，因此对于聚合类算子比较多的应用会对JobManager的内存有一定压力，进而对整个集群会造成较大负担。
- 尽管在创建MemoryStateBackend时可以指定状态初始化内存大小，但是状态数据传输大小也会受限于Akka框架通信的“akka.framesize”大小限制（默认：10485760bit），该指标表示在JobManage和TaskManager之间传输数据的最大消息容量。
- JVM内存容量受限于主机内存大小，也就是说不管是JobManager内存还是在TaskManager的内存中维护状态数据都有内存的限制，因此对于非常大的状态数据则不适合使用MemoryStateBackend存储。

适用场景：
- 基于内存的 StateBackend 在生产环境下不建议使用，因为 State 大小超过 JobManager 内存就 OOM 了，此种状态后端适合在本地开发调试测试，生产环境基本不用。
- State 存储在 JobManager 的内存中。受限于 JobManager 的内存大小。
- 每个 State 默认 5MB，可通过 MemoryStateBackend 构造函数调整。
- 每个 Stale 不能超过 Akka Frame 大小。

因此综上可以得出，MemoryStateBackend比较适合用于测试环境中，并用于本地调试和验证，不建议在生产环境中使用。但如果应用状态数据量不是很大，例如使用了大量的非状态计算算子，也可以在生产环境中使用MemoryStateBackend，否则应该改用其他更加稳定的StateBackend作为状态管理器，例如后面讲到的FsStateBackend和RockDbStateBackend等。

### FsStateBackend
原理： 运行时所需的 State 数据全部保存在 TaskManager 的内存中，执行 Checkpoint 的时候，会把 State 的快照数据保存到配置的文件系统中。TM 是异步将 State 数据写入外部存储。
适用场景：
- 适用于处理小状态、短窗口、或者小键值状态的有状态处理任务，不建议在大状态的任务下使用 FSStateBackend。
- 适用的场景比如明细层 ETL 任务，小时间间隔的 TUMBLE 窗口 
- State 大小不能超过 TM 内存。

和MemoryStateBackend有所不同，FsStateBackend是基于文件系统的一种状态管理器，这里的文件系统可以是本地文件系统，也可以是HDFS分布式文件系统。
```java
new FsStateBackend(path, false);
```
如以上创建FsStateBackend的实例代码，其中path如果为本地路径，其格式为“file:///data/flink/checkpoints”，如果path为HDFS路径，其格式为“hdfs://nameservice/flink/checkpoints”。FsStateBackend中第二个Boolean类型的参数指定是否以同步的方式进行状态数据记录，默认采用异步的方式将状态数据同步到文件系统中，异步方式能够尽可能避免在Checkpoint的过程中影响流式计算任务。如果用户想采用同步的方式进行状态数据的检查点数据，则将第二个参数指定为True即可。
相比于MemoryStateBackend，FsStateBackend更适合任务状态非常大的情况，例如应用中含有时间范围非常长的窗口计算，或Key/value State状态数据量非常大的场景，这时系统内存不足以支撑状态数据的存储。同时基于文件系统存储最大的好处是相对比较稳定，同时借助于像HDFS分布式文件系统中具有三副本备份的策略，能最大程度保证状态数据的安全性，不会出现因为外部故障而导致任务无法恢复等问题。

### RocksDBStateBackend
原理：使用嵌入式的本地数据库 RocksDB 将流计算数据状态存储在本地磁盘中。在执行 Checkpoint 的时候，会将整个 RocksDB 中保存的 State 数据全量或者增量持久化到配置的文件系统中。
适用场景：
- 最适合用于处理大状态、长窗口，或大键值状态的有状态处理任务。
- RocksDBStateBackend 是目前唯一支持增量检查点的后端。
- 增量检查点非常适用于超大状态的场景。比如计算 DAU 这种大数据量去重，大状态的任务都建议直接使用 RocksDB 状态后端。

RocksDBStateBackend是Flink中内置的第三方状态管理器，和前面的状态管理器不同，RocksDBStateBackend需要单独引入相关的依赖包到工程中。通过初始化RockDBStateBackend类，使可以得到RockDBStateBackend实例类。
```java
//创建RocksDBStateBackend实例类
new RocksDBStateBackend(path);
```
RocksDBStateBackend采用异步的方式进行状态数据的Snapshot，任务中的状态数据首先被写入RockDB中，然后再异步地将状态数据写入文件系统中，这样在RockDB仅会存储正在进行计算的热数据，对于长时间才更新的数据则写入磁盘中进行存储。而对于体量比较小的元数据状态，则直接存储在JobManager的内存中。

与FsStateBackend相比，RocksDBStateBackend在性能上要比FsStateBackend高一些，主要是因为借助于RocksDB存储了最新热数据，然后通过异步的方式再同步到文件系统中，但RocksDBStateBackend和MemoryStateBackend相比性能就会较弱一些。

需要注意的是RocksDB通过JNI的方式进行数据的交互，而JNI构建在byte[]数据结构之上，因此每次能够传输的最大数据量为2^31字节，也就是说每次在RocksDBStateBackend合并的状态数据量大小不能超过2^31字节限制，否则将会导致状态数据无法同步，这是RocksDB采用JNI方式的限制，用户在使用过程中应当注意。

综上可以看出，RocksDBStateBackend和FsStateBackend一样，适合于任务状态数据非常大的场景。在Flink最新版本中，已经提供了基于RocksDBStateBackend实现的增量Checkpoints功能，极大地提高了状态数据同步到介质中的效率和性能，在后续的社区发展中，RocksDBStateBackend也会作为状态管理器重点使用的方式之一。

## Flink 中状态的能力扩展 - TTL
Flink 对状态做了能力扩展，即 TTL。它的能力其实和 redis 的过期策略类似，举例：
- 支持 TTL 更新类型：更新 TTL 的时机
- 访问到已过期数据的时的数据可见性 
- 过期时间语义：目前只支持处理时间
- 具体过期实现：lazy，后台线程
那么首先我们看下什么场景需要用到 TTL 机制呢？举例：
比如计算 DAU 使用 Flink MapState 进行去重，到第二天的时候，第一天的 MapState 就可以删除了，就可以用 Flink State TTL 进行自动删除（当然你也可以通过代码逻辑进行手动删除）。
其实在 Flink DataStream API 中，TTL 功能还是比较少用的。Flink State TTL 在 Flink SQL 中是被大规模应用的，几乎除了窗口类、ETL（DWD 明细处理任务）类的任务之外，SQL 任务基本都会用到 State TTL。
那么我们在要怎么开启 TTL 呢？这里分 DataStream API 和 SQL API：
DataStream API：
```java
private final MapStateDescriptor<String, List<Item>> mapStateDesc =
        new MapStateDescriptor<>(
                "itemsMap",
                BasicTypeInfo.STRING_TYPE_INFO,
                new ListTypeInfo<>(Item.class));

@Override
public void open(Configuration parameters) throws Exception {
    super.open(parameters);

    // 使用 StateTtlConfig 开启 State TTL
    mapStateDesc.enableTimeToLive(StateTtlConfig
            .newBuilder(Time.milliseconds(1))
            .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
            .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
            .cleanupInRocksdbCompactFilter(10)
            .build());
}
```
SQL API：
```java
StreamTableEnvironment
    .getConfig()
    .getConfiguration()
    .setString("table.exec.state.ttl", "180 s");
```
## Flink 中状态 TTL 的原理机制？
首先我们来想想，要做到 TTL 的话，要具备什么条件呢？
想想 Redis 的 TTL 设置，如果我们要设置 TTL 则必然需要给一条数据给一个时间戳，只有这样才能判断这条数据是否过期了。
在 Flink 中设置 State TTL，就会有这样一个时间戳，具体实现时，Flink 会把时间戳字段和具体数据字段存储作为同级存储到 State 中。
举个例子，我要将一个 String 存储到 State 中时：
- 没有设置 State TTL 时，则直接将 String 存储在 State 中
- 如果设置 State TTL 时，则 Flink 会将 <String, Long> 存储在 State 中，其中 Long 为时间戳，用于判断是否过期。
了解了基础数据结构之后，我们再来看看 Flink 提供的 State 过期的 4 种删除策略：
- lazy 删除策略：就是在访问 State 的时候根据时间戳判断是否过期，如果过期则主动删除 State 数据
- full snapshot cleanup 删除策略：从状态恢复（checkpoint、savepoint）的时候采取做过期删除，但是不支持 rocksdb 增量 ck
- incremental cleanup 删除策略：访问 state 的时候，主动去遍历一些 state 数据判断是否过期，如果过期则主动删除 State 数据
- rocksdb compaction cleanup 删除策略：rockdb 做 compaction 的时候遍历进行删除。仅仅支持 rocksdb

## 状态管理器配置
在StateBackend应用过程中，除了MemoryStateBackend不需要显示配置之外，其他状态管理器都需要进行相关的配置。在Flink中包含了两种级别的StateBackend配置：一种是应用层面配置，配置的状态管理器只会针对当前应用有效；另外一种是整个集群的默认配置，一旦配置就会对整个Flink集群上的所有应用有效。

## 应用级别配置
在Flink应用中通过StreamExecutionEnvironment提供的setStateBackend()方法配置状态管理器，代码清单5-8通过实例化FsStateBackend，然后在setStateBackend方法中指定相应的状态管理器，这样后续应用的状态管理都会基于HDFS文件系统进行。
```java
StreamExecutionEnvironment env =
StreamExecutionEnvironment.getExecutionEnvironment();
env.setStateBackend(new     FsStateBackend("hdfs://namenode:40010/flink/checkpoints"));
```
如果使用RocksDBStateBackend则需要单独引入rockdb依赖库，将相关的Maven依赖配置引入到本地工程中。
```java
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-statebackend-rocksdb_2.11</artifactId>
  <version>1.7.0</version>
</dependency>
```
RocksDBStateBackend应用配置
```java
StreamExecutionEnvironment env = 
StreamExecutionEnvironment.getExecutionEnvironment();
env.setStateBackend(
new RocksDBStateBackend ("hdfs://namenode:40010/flink/checkpoints"));
```

## 集群级别配置
前面已经提到除了能够在应用层面对StateBackend进行配置，应用独立使用自己的StateBackend之外，Flink同时支持在集群中配置默认的StateBackend。具体的配置项在flink-conf.yaml文件中，如下代码所示，参数state.backend指明StateBackend类型，state.checkpoints.dir配置具体的状态存储路径，代码中使用filesystem作为StateBackend，然后指定相应的HDFS文件路径作为state的checkpoint文件夹。
```java
state.backend: filesystem
# Directory for storing checkpoints
state.checkpoints.dir: hdfs://namenode:40010/flink/checkpoints
```
如果在集群默认使用RocksDBStateBackend作为状态管理器，则对应在flink-conf.yaml中的配置参数如下：
```java
state.backend.rocksdb.checkpoint.transfer.thread.num：1
state.backend.rocksdb.localdir：/var/rockdb/flink/checkpoints
state.backend.rocksdb.timer-service.factory： HEAP
```
- state.backend.rocksdb.checkpoint.transfer.thread.num：用于指定同时可以操作RocksDBStateBackend的线程数量，默认值为1，用户可以根据实际应用场景进行调整，如果状态量比较大则可以将此参数适当增大。
- state.backend.rocksdb.localdir：用于指定RocksDB存储状态数据的本地文件路径，在每个TaskManager提供该路径存储节点中的状态数据。
- state.backend.rocksdb.timer-service.factory：用于指定定时器服务的工厂类实现类，默认为“HEAP”，也可以指定为“RocksDB”。

## Querable State
在Flink中将算子的状态视为头等公民，状态作为系统流式数据计算的重要数据支撑，借助于状态可以完成了相比于无状态计算更加复杂的场景。而在通常情况下流式系统中基于状态统计出的结果数据必须输出到外部系统中才能被其他系统使用，业务系统无法与流系统直接对接并获取中间状态结果。在Flink新的版本中提出可查询的状态服务，也就是说业务系统可以通过Flink提供的RestfulAPI接口直接查询Flink系统内部的状态数据。

Flink可查询状态架构中包含三个重要组件：
- QueryableStateClient：用于外部应用中，作为客户端提交查询请求并收集状态查询结果。

- QueryableStateClientProxy：用于接收和处理客户端的请求，每个TaskManager上运行一个客户端代理。状态数据分布在算子所有并发的实例中，Client Proxy需要通过从JobManager中获取Key Group的分布，然后判断哪一个TaskManager实例维护了Client中传过来的Key对应的状态数据，并且向所在的TaskManager的Server发出访问请求并将查询到的状态值返回给客户端。

- QueryableStateServer：用于接收Client Proxy的请求，每个TaskManager上会运行一个State Server，该Server用于通过从本地的状态后台管理器中查询状态结果，然后返回给客户端代理。

### 激活可查询状态服务
为了能够开启可查询状态服务，需要在Flink中引入flink-queryable-state-runtime.jar文件，可以通过将flink-queryable-state-runtime.jar从安装路径中./opt拷贝到./lib路径下引入，每次Flink集群启动时就会将flink-queryable-state-runtime.jar加载到TaskManager的环境变量中。在Flink集群启动时，Queryable State的服务就会在TaskManager中被拉起，此时就能够处理由Client发送的请求。可以通过检查TaskManager日志的方式来确认Queryable State服务是否成功启动，如果日志中出现：“Started the Queryable State Proxy Server”，则表明Queryable State服务被正常启动并可以使用。Queryable State Proxy和Server端口以及其他相关的参数均可以在flink-conf.yaml文件中配置。

### 可查询状态应用配置
除了在集群层面激活Queryable State服务，还需要在Flink应用中修改应用程序的代码，将需要暴露的可查询状态通过配置开放出来。在代码中增加Queryable State功能相对比较简单，在创建状态的StateDescriptor中调用setQueryable(String)方法就能够将需要暴露的状态开发出来。
```java
override def open(parameters: Configuration): Unit = {
  //创建ValueStateDescriptor,定义状态名称为leastValue,并指定数据类型
  val leastValueStateDescriptor = new ValueStateDescriptor[Long]("leastValue", classOf[Long])
  //打开可查询状态功能，让状态可以被外部应用检索
  leastValueStateDescriptor.setQueryable("leastQueryValue")
  //通过getRuntimeContext.getState获取State
  leastValueState = getRuntimeContext.getState(leastValueStateDescriptor)
}
```
通过在KeyedStream上使用asQueryableState方法来设定可查询状态，其中返回的QueryableStateStream数据集将被当作DataSink算子，因此后面不能再接入其他算子。根据状态类型的不同，可以在asQueryableState()方法中指定不同的StatDesciptor来设定相应的可查询状态：

## Flink 状态的误用之痛？
一定要分清楚 operator-state 和 keyed-state 的区别以及使用方式。KeyedStream 后面错用 operator-state，operator-state 大 State 导致 OOM。建议 KeyedStream 上还是使用 keyed-state。
一定要学会分场景使用 ValueState 和 MapState。在ValueState 中存储一个大 Map，并且使用 RocksDB，导致 State 访问非常慢（因为 RocksDB 访问 State 经过序列化），拖慢任务处理速度。
两者的具体区别如下：
- ValueState
应用场景：简单的一个变量存储，比如 Long\String 等。如果状态后端为 RocksDB，极其不建议在 ValueState 中存储一个大 Map，这种场景下序列化和反序列化的成本非常高，拖慢任务处理速度，这种常见适合使用 MapState。其实这种场景也是很多小伙伴一开始使用 State 的误用之痛，一定要避免。
TTL：针对整个 Value 起作用
- MapState
应用场景：和 Map 使用方式一样一样的
TTL：针对 Map 的 key 生效，每个 key 一个 TTL
keyed-state 不能在 open 方法中访问、更新 state，这是不行的，因为 open 方法在执行时，还没有到正式的数据处理环节，上下文中是没有 key 的
operator-state 中的 ListState 进行以下操作会有问题。因为当实例化的 state 为 PartitionableListState 时，会先把 list clear，然后再 add，你会发现 state 一致为空。











