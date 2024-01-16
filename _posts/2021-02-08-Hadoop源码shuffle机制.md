---
layout: post
categories: [Hadoop]
description: none
keywords: Hadoop
---
# Hadoop源码shuffle机制

## shuffle机制
首先，shuffle是mapTask运行写出一个key，value键值对后，收集器收集，开始shuffle的工作。所以入口在MapTask的run()方法中的 runNewMapper(job, splitMetaInfo, umbilical, reporter);

在这里我主要聊shuffle的两个方面，一个是shuffle前的准备工作，即开启Collector收集器。读入一些Job的配置信息。另外一个，也是最重要的就是shuffle具体的工作机制和整体流程。

入口：
```
runNewMapper(job, splitMetaInfo, umbilical, reporter);
```
Shuffle的准备工作
以下为这个方法的具体实现：
```
private <INKEY,INVALUE,OUTKEY,OUTVALUE>
  void runNewMapper(final JobConf job,
                    final TaskSplitIndex splitIndex,
                    final TaskUmbilicalProtocol umbilical,
                    TaskReporter reporter
                    ) throws IOException, ClassNotFoundException,
                             InterruptedException {
    // make a task context so we can get the classes
    org.apache.hadoop.mapreduce.TaskAttemptContext taskContext =
      new org.apache.hadoop.mapreduce.task.TaskAttemptContextImpl(job, 
                                                                  getTaskID(),
                                                                  reporter);
    // make a mapper
    org.apache.hadoop.mapreduce.Mapper<INKEY,INVALUE,OUTKEY,OUTVALUE> mapper =
      (org.apache.hadoop.mapreduce.Mapper<INKEY,INVALUE,OUTKEY,OUTVALUE>)
        ReflectionUtils.newInstance(taskContext.getMapperClass(), job);
    // make the input format
    org.apache.hadoop.mapreduce.InputFormat<INKEY,INVALUE> inputFormat =
      (org.apache.hadoop.mapreduce.InputFormat<INKEY,INVALUE>)
        ReflectionUtils.newInstance(taskContext.getInputFormatClass(), job);
    // rebuild the input split
    org.apache.hadoop.mapreduce.InputSplit split = null;
    split = getSplitDetails(new Path(splitIndex.getSplitLocation()),
        splitIndex.getStartOffset());
    LOG.info("Processing split: " + split);

    org.apache.hadoop.mapreduce.RecordReader<INKEY,INVALUE> input =
      new NewTrackingRecordReader<INKEY,INVALUE>
        (split, inputFormat, reporter, taskContext);
    
    job.setBoolean(JobContext.SKIP_RECORDS, isSkipping());
    org.apache.hadoop.mapreduce.RecordWriter output = null;
    
    // get an output object
    if (job.getNumReduceTasks() == 0) {
      output = 
        new NewDirectOutputCollector(taskContext, job, umbilical, reporter);
    } else {
      output = new NewOutputCollector(taskContext, job, umbilical, reporter);
    }

    org.apache.hadoop.mapreduce.MapContext<INKEY, INVALUE, OUTKEY, OUTVALUE> 
    mapContext = 
      new MapContextImpl<INKEY, INVALUE, OUTKEY, OUTVALUE>(job, getTaskID(), 
          input, output, 
          committer, 
          reporter, split);

    org.apache.hadoop.mapreduce.Mapper<INKEY,INVALUE,OUTKEY,OUTVALUE>.Context 
        mapperContext = 
          new WrappedMapper<INKEY, INVALUE, OUTKEY, OUTVALUE>().getMapContext(
              mapContext);

    try {
      input.initialize(split, mapperContext);
      mapper.run(mapperContext);
      mapPhase.complete();
      setPhase(TaskStatus.Phase.SORT);
      statusUpdate(umbilical);
      input.close();
      input = null;
      output.close(mapperContext);
      output = null;
    } finally {
      closeQuietly(input);
      closeQuietly(output, mapperContext);
    }
  }

```
这部分代码中大多创建一些hadoop中的组件
```
org.apache.hadoop.mapreduce.TaskAttemptContext taskContext =
new org.apache.hadoop.mapreduce.task.TaskAttemptContextImpl(job,
getTaskID(), reporter);
创建taskContext

org.apache.hadoop.mapreduce.Mapper<INKEY,INVALUE,OUTKEY,OUTVALUE> mapper =
(org.apache.hadoop.mapreduce.Mapper<INKEY,INVALUE,OUTKEY,OUTVALUE>)
ReflectionUtils.newInstance(taskContext.getMapperClass(), job);

通过反射创建mapper

org.apache.hadoop.mapreduce.InputFormat<INKEY,INVALUE> inputFormat =
(org.apache.hadoop.mapreduce.InputFormat<INKEY,INVALUE>)
ReflectionUtils.newInstance(taskContext.getInputFormatClass(), job);

通过反射创建inputFormat，来读取数据

split = getSplitDetails(new Path(splitIndex.getSplitLocation()),
splitIndex.getStartOffset());

获取切片的详细信息

org.apache.hadoop.mapreduce.RecordReader<INKEY,INVALUE> input =
new NewTrackingRecordReader<INKEY,INVALUE>
(split, inputFormat, reporter, taskContext);

通过反射创建RecordReader。InputFormat是通过RecordReader来读取数据
```
重点来了。创建收集器，也就是OutputCollector对象
```
if (job.getNumReduceTasks() == 0) {
      output = 
        new NewDirectOutputCollector(taskContext, job, umbilical, reporter);
    } else {
      output = new NewOutputCollector(taskContext, job, umbilical, reporter);
    }

```
判断reduce的个数是否为0，如果为0，就没有shuffle过程，也就是直接将数据写到磁盘中。如果不为0，就帮你创建一个OutputCollector对象。具体创建的代码如下:
```
 NewOutputCollector(org.apache.hadoop.mapreduce.JobContext jobContext,
                       JobConf job,
                       TaskUmbilicalProtocol umbilical,
                       TaskReporter reporter
                       ) throws IOException, ClassNotFoundException {
      collector = createSortingCollector(job, reporter);
      partitions = jobContext.getNumReduceTasks();
      if (partitions > 1) {
        partitioner = (org.apache.hadoop.mapreduce.Partitioner<K,V>)
          ReflectionUtils.newInstance(jobContext.getPartitionerClass(), job);
      } else {
        partitioner = new org.apache.hadoop.mapreduce.Partitioner<K,V>() {
          @Override
          public int getPartition(K key, V value, int numPartitions) {
            return partitions - 1;
          }
        };
      }
    }

```
collector = createSortingCollector(job, reporter);创建收集器对象MapOutputCollector，创建的这个收集器对象只是一个宏观的收集器对象，真正核心的是他方法里面，创建的MapOutputBuffer对象

所以进入到这个方法中。方法中的代码如下：
```
private <KEY, VALUE> MapOutputCollector<KEY, VALUE>
          createSortingCollector(JobConf job, TaskReporter reporter)
    throws IOException, ClassNotFoundException {
    MapOutputCollector.Context context =
      new MapOutputCollector.Context(this, job, reporter);

    Class<?>[] collectorClasses = job.getClasses(
      JobContext.MAP_OUTPUT_COLLECTOR_CLASS_ATTR, MapOutputBuffer.class);
    int remainingCollectors = collectorClasses.length;
    Exception lastException = null;
    for (Class clazz : collectorClasses) {
      try {
        if (!MapOutputCollector.class.isAssignableFrom(clazz)) {
          throw new IOException("Invalid output collector class: " + clazz.getName() +
            " (does not implement MapOutputCollector)");
        }
        Class<? extends MapOutputCollector> subclazz =
          clazz.asSubclass(MapOutputCollector.class);
        LOG.debug("Trying map output collector class: " + subclazz.getName());
        MapOutputCollector<KEY, VALUE> collector =
          ReflectionUtils.newInstance(subclazz, job);
        collector.init(context);
        LOG.info("Map output collector class = " + collector.getClass().getName());
        return collector;
      } catch (Exception e) {
        String msg = "Unable to initialize MapOutputCollector " + clazz.getName();
        if (--remainingCollectors > 0) {
          msg += " (" + remainingCollectors + " more collector(s) to try)";
        }
        lastException = e;
        LOG.warn(msg, e);
      }
    }
    throw new IOException("Initialization of all the collectors failed. " +
      "Error in last collector was :" + lastException.getMessage(), lastException);
  }

```
Class<?>[] collectorClasses = job.getClasses(
JobContext.MAP_OUTPUT_COLLECTOR_CLASS_ATTR, MapOutputBuffer.class);

获取到具体的收集器对象的类型 MapOutputBuffer

MapOutputCollector<KEY, VALUE> collector = ReflectionUtils.newInstance(subclazz, job);

创建MapOutputBuffer收集器对象

collector.init(context);初始化MapOutputBuffer收集器对象

具体的代码如下：
```
public void init(MapOutputCollector.Context context
                    ) throws IOException, ClassNotFoundException {
      job = context.getJobConf();
      reporter = context.getReporter();
      mapTask = context.getMapTask();
      mapOutputFile = mapTask.getMapOutputFile();
      sortPhase = mapTask.getSortPhase();
      spilledRecordsCounter = reporter.getCounter(TaskCounter.SPILLED_RECORDS);
      partitions = job.getNumReduceTasks();
      rfs = ((LocalFileSystem)FileSystem.getLocal(job)).getRaw();

      //sanity checks
      final float spillper =
        job.getFloat(JobContext.MAP_SORT_SPILL_PERCENT, (float)0.8);
      final int sortmb = job.getInt(JobContext.IO_SORT_MB, 100);
      indexCacheMemoryLimit = job.getInt(JobContext.INDEX_CACHE_MEMORY_LIMIT,
                                         INDEX_CACHE_MEMORY_LIMIT_DEFAULT);
      if (spillper > (float)1.0 || spillper <= (float)0.0) {
        throw new IOException("Invalid \"" + JobContext.MAP_SORT_SPILL_PERCENT +
            "\": " + spillper);
      }
      if ((sortmb & 0x7FF) != sortmb) {
        throw new IOException(
            "Invalid \"" + JobContext.IO_SORT_MB + "\": " + sortmb);
      }
      sorter = ReflectionUtils.newInstance(job.getClass("map.sort.class",
            QuickSort.class, IndexedSorter.class), job);
      // buffers and accounting
      int maxMemUsage = sortmb << 20;
      maxMemUsage -= maxMemUsage % METASIZE;
      kvbuffer = new byte[maxMemUsage];
      bufvoid = kvbuffer.length;
      kvmeta = ByteBuffer.wrap(kvbuffer)
         .order(ByteOrder.nativeOrder())
         .asIntBuffer();
      setEquator(0);
      bufstart = bufend = bufindex = equator;
      kvstart = kvend = kvindex;

      maxRec = kvmeta.capacity() / NMETA;
      softLimit = (int)(kvbuffer.length * spillper);
      bufferRemaining = softLimit;
      LOG.info(JobContext.IO_SORT_MB + ": " + sortmb);
      LOG.info("soft limit at " + softLimit);
      LOG.info("bufstart = " + bufstart + "; bufvoid = " + bufvoid);
      LOG.info("kvstart = " + kvstart + "; length = " + maxRec);

      // k/v serialization
      comparator = job.getOutputKeyComparator();
      keyClass = (Class<K>)job.getMapOutputKeyClass();
      valClass = (Class<V>)job.getMapOutputValueClass();
      serializationFactory = new SerializationFactory(job);
      keySerializer = serializationFactory.getSerializer(keyClass);
      keySerializer.open(bb);
      valSerializer = serializationFactory.getSerializer(valClass);
      valSerializer.open(bb);

      // output counters
      mapOutputByteCounter = reporter.getCounter(TaskCounter.MAP_OUTPUT_BYTES);
      mapOutputRecordCounter =
        reporter.getCounter(TaskCounter.MAP_OUTPUT_RECORDS);
      fileOutputByteCounter = reporter
          .getCounter(TaskCounter.MAP_OUTPUT_MATERIALIZED_BYTES);

      // compression
      if (job.getCompressMapOutput()) {
        Class<? extends CompressionCodec> codecClass =
          job.getMapOutputCompressorClass(DefaultCodec.class);
        codec = ReflectionUtils.newInstance(codecClass, job);
      } else {
        codec = null;
      }

      // combiner
      final Counters.Counter combineInputCounter =
        reporter.getCounter(TaskCounter.COMBINE_INPUT_RECORDS);
      combinerRunner = CombinerRunner.create(job, getTaskID(), 
                                             combineInputCounter,
                                             reporter, null);
      if (combinerRunner != null) {
        final Counters.Counter combineOutputCounter =
          reporter.getCounter(TaskCounter.COMBINE_OUTPUT_RECORDS);
        combineCollector= new CombineOutputCollector<K,V>(combineOutputCounter, reporter, job);
      } else {
        combineCollector = null;
      }
      spillInProgress = false;
      minSpillsForCombine = job.getInt(JobContext.MAP_COMBINE_MIN_SPILLS, 3);
      spillThread.setDaemon(true);
      spillThread.setName("SpillThread");
      spillLock.lock();
      try {
        spillThread.start();
        while (!spillThreadRunning) {
          spillDone.await();
        }
      } catch (InterruptedException e) {
        throw new IOException("Spill thread failed to initialize", e);
      } finally {
        spillLock.unlock();
      }
      if (sortSpillException != null) {
        throw new IOException("Spill thread failed to initialize",
            sortSpillException);
      }
    }

```
这个方法中，主要就是对收集器对象进行一些初始化，由于初始化的东西太多，在这里我就挑一部分比较重要的讲一讲。

final float spillper = job.getFloat(JobContext.MAP_SORT_SPILL_PERCENT, (float)0.8);

设置环形缓冲区溢写百分比为80%。大家知道，收集器对象将map阶段输出的数据收集起来，并将其收集到环形缓冲区中。环形缓冲区，分为两片，一片为专门存放数据的区域，一片为存放这个数据的一些元数据信息的区域。这两片区域共同组成一个环形。但是这个环形缓冲区的大小不会太大，否则会很占内存。所以当环形缓冲区内数据的大小到达整体的80%时，就会将环形缓冲区中的数据溢写到磁盘中。

final int sortmb = job.getInt(JobContext.IO_SORT_MB, 100);

设置环形缓冲区的大小为100M。

sorter = ReflectionUtils.newInstance(job.getClass(“map.sort.class”,

QuickSort.class, IndexedSorter.class), job);

获取到排序对象，在数据由环形缓冲区溢写到磁盘中前，数据需要排序。所以需要排序器。并且排序是针对索引的，并非对数据进行排序。

comparator = job.getOutputKeyComparator();

获取分组比较器

获取压缩器
```
if (job.getCompressMapOutput()) {
        Class<? extends CompressionCodec> codecClass =
          job.getMapOutputCompressorClass(DefaultCodec.class);
        codec = ReflectionUtils.newInstance(codecClass, job);
      } else {
        codec = null;
      }

```
可能shuffle中需要对数据进行压缩，这样能提高shuffle的效率

获取Combiner
```
if (combinerRunner != null) {
        final Counters.Counter combineOutputCounter =
          reporter.getCounter(TaskCounter.COMBINE_OUTPUT_RECORDS);
        combineCollector= new CombineOutputCollector<K,V>(combineOutputCounter, reporter, job);
      } else {
        combineCollector = null;
      }

```
可能shuffle中设置了combiner，如果设置了，就创建combiner对象

spillThread.start();

在一系列准备工作做完之后，就开启溢写线程进行工作

## Shuffle的工作流程
入口：在map往外写的时候，就开启了shuffle

一路debug，一直到
```
public void write(K key, V value) throws IOException, InterruptedException {
      collector.collect(key, value,
                        partitioner.getPartition(key, value, partitions));
    }

```
Collector收集器开始工作

具体的代码实现如下：
```
public synchronized void collect(K key, V value, final int partition
                                     ) throws IOException {
      reporter.progress();
      if (key.getClass() != keyClass) {
        throw new IOException("Type mismatch in key from map: expected "
                              + keyClass.getName() + ", received "
                              + key.getClass().getName());
      }
      if (value.getClass() != valClass) {
        throw new IOException("Type mismatch in value from map: expected "
                              + valClass.getName() + ", received "
                              + value.getClass().getName());
      }
      if (partition < 0 || partition >= partitions) {
        throw new IOException("Illegal partition for " + key + " (" +
            partition + ")");
      }
      checkSpillException();
      bufferRemaining -= METASIZE;
      if (bufferRemaining <= 0) {
        // start spill if the thread is not running and the soft limit has been
        // reached
        spillLock.lock();
        try {
          do {
            if (!spillInProgress) {
              final int kvbidx = 4 * kvindex;
              final int kvbend = 4 * kvend;
              // serialized, unspilled bytes always lie between kvindex and
              // bufindex, crossing the equator. Note that any void space
              // created by a reset must be included in "used" bytes
              final int bUsed = distanceTo(kvbidx, bufindex);
              final boolean bufsoftlimit = bUsed >= softLimit;
              if ((kvbend + METASIZE) % kvbuffer.length !=
                  equator - (equator % METASIZE)) {
                // spill finished, reclaim space
                resetSpill();
                bufferRemaining = Math.min(
                    distanceTo(bufindex, kvbidx) - 2 * METASIZE,
                    softLimit - bUsed) - METASIZE;
                continue;
              } else if (bufsoftlimit && kvindex != kvend) {
                // spill records, if any collected; check latter, as it may
                // be possible for metadata alignment to hit spill pcnt
                startSpill();
                final int avgRec = (int)
                  (mapOutputByteCounter.getCounter() /
                  mapOutputRecordCounter.getCounter());
                // leave at least half the split buffer for serialization data
                // ensure that kvindex >= bufindex
                final int distkvi = distanceTo(bufindex, kvbidx);
                final int newPos = (bufindex +
                  Math.max(2 * METASIZE - 1,
                          Math.min(distkvi / 2,
                                   distkvi / (METASIZE + avgRec) * METASIZE)))
                  % kvbuffer.length;
                setEquator(newPos);
                bufmark = bufindex = newPos;
                final int serBound = 4 * kvend;
                // bytes remaining before the lock must be held and limits
                // checked is the minimum of three arcs: the metadata space, the
                // serialization space, and the soft limit
                bufferRemaining = Math.min(
                    // metadata max
                    distanceTo(bufend, newPos),
                    Math.min(
                      // serialization max
                      distanceTo(newPos, serBound),
                      // soft limit
                      softLimit)) - 2 * METASIZE;
              }
            }
          } while (false);
        } finally {
          spillLock.unlock();
        }
      }

      try {
        // serialize key bytes into buffer
        int keystart = bufindex;
        keySerializer.serialize(key);
        if (bufindex < keystart) {
          // wrapped the key; must make contiguous
          bb.shiftBufferedKey();
          keystart = 0;
        }
        // serialize value bytes into buffer
        final int valstart = bufindex;
        valSerializer.serialize(value);
        // It's possible for records to have zero length, i.e. the serializer
        // will perform no writes. To ensure that the boundary conditions are
        // checked and that the kvindex invariant is maintained, perform a
        // zero-length write into the buffer. The logic monitoring this could be
        // moved into collect, but this is cleaner and inexpensive. For now, it
        // is acceptable.
        bb.write(b0, 0, 0);

        // the record must be marked after the preceding write, as the metadata
        // for this record are not yet written
        int valend = bb.markRecord();

        mapOutputRecordCounter.increment(1);
        mapOutputByteCounter.increment(
            distanceTo(keystart, valend, bufvoid));

        // write accounting info
        kvmeta.put(kvindex + PARTITION, partition);
        kvmeta.put(kvindex + KEYSTART, keystart);
        kvmeta.put(kvindex + VALSTART, valstart);
        kvmeta.put(kvindex + VALLEN, distanceTo(valstart, valend));
        // advance kvindex
        kvindex = (kvindex - NMETA + kvmeta.capacity()) % kvmeta.capacity();
      } catch (MapBufferTooSmallException e) {
        LOG.info("Record too large for in-memory buffer: " + e.getMessage());
        spillSingleRecord(key, value, partition);
        mapOutputRecordCounter.increment(1);
        return;
      }
    }

```
什么时候开始进行溢写？

当你的环形缓冲区里面的数据大小达到整个环形缓冲区大小的80%时，开始进行溢写。溢写时要上锁。spillLock.lock();

开启了溢写，溢写主要工作顺序是先排序后溢写，所以执行的是sortAndSpill方法，具体代码如下：
```
private void sortAndSpill() throws IOException, ClassNotFoundException,
                                       InterruptedException {
      //approximate the length of the output file to be the length of the
      //buffer + header lengths for the partitions
      final long size = distanceTo(bufstart, bufend, bufvoid) +
                  partitions * APPROX_HEADER_LENGTH;
      FSDataOutputStream out = null;
      try {
        // create spill file
        final SpillRecord spillRec = new SpillRecord(partitions);
        final Path filename =
            mapOutputFile.getSpillFileForWrite(numSpills, size);
        out = rfs.create(filename);

        final int mstart = kvend / NMETA;
        final int mend = 1 + // kvend is a valid record
          (kvstart >= kvend
          ? kvstart
          : kvmeta.capacity() + kvstart) / NMETA;
        sorter.sort(MapOutputBuffer.this, mstart, mend, reporter);
        int spindex = mstart;
        final IndexRecord rec = new IndexRecord();
        final InMemValBytes value = new InMemValBytes();
        for (int i = 0; i < partitions; ++i) {
          IFile.Writer<K, V> writer = null;
          try {
            long segmentStart = out.getPos();
            FSDataOutputStream partitionOut = CryptoUtils.wrapIfNecessary(job, out);
            writer = new Writer<K, V>(job, partitionOut, keyClass, valClass, codec,
                                      spilledRecordsCounter);
            if (combinerRunner == null) {
              // spill directly
              DataInputBuffer key = new DataInputBuffer();
              while (spindex < mend &&
                  kvmeta.get(offsetFor(spindex % maxRec) + PARTITION) == i) {
                final int kvoff = offsetFor(spindex % maxRec);
                int keystart = kvmeta.get(kvoff + KEYSTART);
                int valstart = kvmeta.get(kvoff + VALSTART);
                key.reset(kvbuffer, keystart, valstart - keystart);
                getVBytesForOffset(kvoff, value);
                writer.append(key, value);
                ++spindex;
              }
            } else {
              int spstart = spindex;
              while (spindex < mend &&
                  kvmeta.get(offsetFor(spindex % maxRec)
                            + PARTITION) == i) {
                ++spindex;
              }
              // Note: we would like to avoid the combiner if we've fewer
              // than some threshold of records for a partition
              if (spstart != spindex) {
                combineCollector.setWriter(writer);
                RawKeyValueIterator kvIter =
                  new MRResultIterator(spstart, spindex);
                combinerRunner.combine(kvIter, combineCollector);
              }
            }

            // close the writer
            writer.close();

            // record offsets
            rec.startOffset = segmentStart;
            rec.rawLength = writer.getRawLength() + CryptoUtils.cryptoPadding(job);
            rec.partLength = writer.getCompressedLength() + CryptoUtils.cryptoPadding(job);
            spillRec.putIndex(rec, i);

            writer = null;
          } finally {
            if (null != writer) writer.close();
          }
        }

        if (totalIndexCacheMemory >= indexCacheMemoryLimit) {
          // create spill index file
          Path indexFilename =
              mapOutputFile.getSpillIndexFileForWrite(numSpills, partitions
                  * MAP_OUTPUT_INDEX_RECORD_LENGTH);
          spillRec.writeToFile(indexFilename, job);
        } else {
          indexCacheList.add(spillRec);
          totalIndexCacheMemory +=
            spillRec.size() * MAP_OUTPUT_INDEX_RECORD_LENGTH;
        }
        LOG.info("Finished spill " + numSpills);
        ++numSpills;
      } finally {
        if (out != null) out.close();
      }
    }

```
final Path filename = mapOutputFile.getSpillFileForWrite(numSpills, size);

获取溢写文件名

out = rfs.create(filename);

创建溢写文件

创建结果：

sorter.sort(MapOutputBuffer.this, mstart, mend, reporter);

排序。对每个数据对应的索引进行排序

下面的for循环中，就是将每一个排序好的分区中的数据溢写到spill0.out文件中，由于可能要溢写多次，所以最终溢写来的文件名可能会spilln。

判断存放排好序的索引占用的内存是否大于等于阈值，如果满足，也需要在溢写到磁盘.

判断条件为：
```
if (totalIndexCacheMemory >= indexCacheMemoryLimit) {
          // create spill index file
          Path indexFilename =
              mapOutputFile.getSpillIndexFileForWrite(numSpills, partitions
                  * MAP_OUTPUT_INDEX_RECORD_LENGTH);
          spillRec.writeToFile(indexFilename, job);
        } else {
          indexCacheList.add(spillRec);
          totalIndexCacheMemory +=
            spillRec.size() * MAP_OUTPUT_INDEX_RECORD_LENGTH;
        }

```
在Mapper中所有的kv全部写出去以后，会执行 runNewMapper中的 output.close(mapperContext);
```
output.close(mapperContext);
```
在最后一次map的写之后，需要将环形缓冲区的剩余没有被溢写的文件，通过flush的方式，全部溢写到文件中。

调用flush方法

collector.flush();

同样需要将最后溢写的数据进行排序和溢写，调用的依然是sortAndSpill()方法

sortAndSpill()

在sortAndSpill()方法完成后，调用mergeParts();进行溢写文件的合并

溢写文件的合并也是一个很关键的步骤，这里详细讲解下

具体的代码实现如下：
```
private void mergeParts() throws IOException, InterruptedException, 
                                     ClassNotFoundException {
      // get the approximate size of the final output/index files
      long finalOutFileSize = 0;
      long finalIndexFileSize = 0;
      final Path[] filename = new Path[numSpills];
      final TaskAttemptID mapId = getTaskID();

      for(int i = 0; i < numSpills; i++) {
        filename[i] = mapOutputFile.getSpillFile(i);
        finalOutFileSize += rfs.getFileStatus(filename[i]).getLen();
      }
      if (numSpills == 1) { //the spill is the final output
        sameVolRename(filename[0],
            mapOutputFile.getOutputFileForWriteInVolume(filename[0]));
        if (indexCacheList.size() == 0) {
          sameVolRename(mapOutputFile.getSpillIndexFile(0),
            mapOutputFile.getOutputIndexFileForWriteInVolume(filename[0]));
        } else {
          indexCacheList.get(0).writeToFile(
            mapOutputFile.getOutputIndexFileForWriteInVolume(filename[0]), job);
        }
        sortPhase.complete();
        return;
      }

      // read in paged indices
      for (int i = indexCacheList.size(); i < numSpills; ++i) {
        Path indexFileName = mapOutputFile.getSpillIndexFile(i);
        indexCacheList.add(new SpillRecord(indexFileName, job));
      }

      //make correction in the length to include the sequence file header
      //lengths for each partition
      finalOutFileSize += partitions * APPROX_HEADER_LENGTH;
      finalIndexFileSize = partitions * MAP_OUTPUT_INDEX_RECORD_LENGTH;
      Path finalOutputFile =
          mapOutputFile.getOutputFileForWrite(finalOutFileSize);
      Path finalIndexFile =
          mapOutputFile.getOutputIndexFileForWrite(finalIndexFileSize);

      //The output stream for the final single output file
      FSDataOutputStream finalOut = rfs.create(finalOutputFile, true, 4096);

      if (numSpills == 0) {
        //create dummy files
        IndexRecord rec = new IndexRecord();
        SpillRecord sr = new SpillRecord(partitions);
        try {
          for (int i = 0; i < partitions; i++) {
            long segmentStart = finalOut.getPos();
            FSDataOutputStream finalPartitionOut = CryptoUtils.wrapIfNecessary(job, finalOut);
            Writer<K, V> writer =
              new Writer<K, V>(job, finalPartitionOut, keyClass, valClass, codec, null);
            writer.close();
            rec.startOffset = segmentStart;
            rec.rawLength = writer.getRawLength() + CryptoUtils.cryptoPadding(job);
            rec.partLength = writer.getCompressedLength() + CryptoUtils.cryptoPadding(job);
            sr.putIndex(rec, i);
          }
          sr.writeToFile(finalIndexFile, job);
        } finally {
          finalOut.close();
        }
        sortPhase.complete();
        return;
      }
      {
        sortPhase.addPhases(partitions); // Divide sort phase into sub-phases
        
        IndexRecord rec = new IndexRecord();
        final SpillRecord spillRec = new SpillRecord(partitions);
        for (int parts = 0; parts < partitions; parts++) {
          //create the segments to be merged
          List<Segment<K,V>> segmentList =
            new ArrayList<Segment<K, V>>(numSpills);
          for(int i = 0; i < numSpills; i++) {
            IndexRecord indexRecord = indexCacheList.get(i).getIndex(parts);

            Segment<K,V> s =
              new Segment<K,V>(job, rfs, filename[i], indexRecord.startOffset,
                               indexRecord.partLength, codec, true);
            segmentList.add(i, s);

            if (LOG.isDebugEnabled()) {
              LOG.debug("MapId=" + mapId + " Reducer=" + parts +
                  "Spill =" + i + "(" + indexRecord.startOffset + "," +
                  indexRecord.rawLength + ", " + indexRecord.partLength + ")");
            }
          }

          int mergeFactor = job.getInt(JobContext.IO_SORT_FACTOR, 100);
          // sort the segments only if there are intermediate merges
          boolean sortSegments = segmentList.size() > mergeFactor;
          //merge
          @SuppressWarnings("unchecked")
          RawKeyValueIterator kvIter = Merger.merge(job, rfs,
                         keyClass, valClass, codec,
                         segmentList, mergeFactor,
                         new Path(mapId.toString()),
                         job.getOutputKeyComparator(), reporter, sortSegments,
                         null, spilledRecordsCounter, sortPhase.phase(),
                         TaskType.MAP);

          //write merged output to disk
          long segmentStart = finalOut.getPos();
          FSDataOutputStream finalPartitionOut = CryptoUtils.wrapIfNecessary(job, finalOut);
          Writer<K, V> writer =
              new Writer<K, V>(job, finalPartitionOut, keyClass, valClass, codec,
                               spilledRecordsCounter);
          if (combinerRunner == null || numSpills < minSpillsForCombine) {
            Merger.writeFile(kvIter, writer, reporter, job);
          } else {
            combineCollector.setWriter(writer);
            combinerRunner.combine(kvIter, combineCollector);
          }

          //close
          writer.close();

          sortPhase.startNextPhase();
          
          // record offsets
          rec.startOffset = segmentStart;
          rec.rawLength = writer.getRawLength() + CryptoUtils.cryptoPadding(job);
          rec.partLength = writer.getCompressedLength() + CryptoUtils.cryptoPadding(job);
          spillRec.putIndex(rec, parts);
        }
        spillRec.writeToFile(finalIndexFile, job);
        finalOut.close();
        for(int i = 0; i < numSpills; i++) {
          rfs.delete(filename[i],true);
        }
      }
    }

```
Path finalOutputFile =mapOutputFile.getOutputFileForWrite(finalOutFileSize);
获取最终合并完的溢写文件名 file.out

Path finalIndexFile =mapOutputFile.getOutputIndexFileForWrite(finalIndexFileSize);

获取最终溢写到磁盘的索引文件 file.out.index

对于溢写到磁盘中的索引文件。所有的partition对应的数据都放在file.out文件里，虽然是顺序存放的，但是怎么直接知道某个partition在这个文件中存放的起始位置呢？强大的索引又出场了。有一个三元组记录某个partition对应的数据在这个文件中的索引：起始位置、原始数据长度、压缩之后的数据长度，一个partition对应一个三元组。然后把这些索引信息存放在内存中，如果内存中放不下了，后续的索引信息就需要写到磁盘文件中了：从所有的本地目录中轮训查找能存储这么大空间的目录，找到之后在其中创建一个类似于“file.out.index”的文件，文件中不光存储了索引数据，还存储了crc32的校验数据。这样方便reduce从多个file.out文件中来拿取同一个分区的数据。

FSDataOutputStream finalOut = rfs.create(finalOutputFile, true, 4096);

获取到hdfs文件系统，能够在hdfs上进行文件的操作

Merger.writeFile(kvIter, writer, reporter, job);

将合并好的数据写到最终的文件中

spillRec.writeToFile(finalIndexFile, job);
将索引文件也写到文件中

rfs.delete(filename[i],true);
e);
获取最终合并完的溢写文件名 file.out

Path finalIndexFile =mapOutputFile.getOutputIndexFileForWrite(finalIndexFileSize);

获取最终溢写到磁盘的索引文件 file.out.index

对于溢写到磁盘中的索引文件。所有的partition对应的数据都放在file.out文件里，虽然是顺序存放的，但是怎么直接知道某个partition在这个文件中存放的起始位置呢？强大的索引又出场了。有一个三元组记录某个partition对应的数据在这个文件中的索引：起始位置、原始数据长度、压缩之后的数据长度，一个partition对应一个三元组。然后把这些索引信息存放在内存中，如果内存中放不下了，后续的索引信息就需要写到磁盘文件中了：从所有的本地目录中轮训查找能存储这么大空间的目录，找到之后在其中创建一个类似于“file.out.index”的文件，文件中不光存储了索引数据，还存储了crc32的校验数据。这样方便reduce从多个file.out文件中来拿取同一个分区的数据。

FSDataOutputStream finalOut = rfs.create(finalOutputFile, true, 4096);

获取到hdfs文件系统，能够在hdfs上进行文件的操作

Merger.writeFile(kvIter, writer, reporter, job);

将合并好的数据写到最终的文件中

spillRec.writeToFile(finalIndexFile, job);
将索引文件也写到文件中

rfs.delete(filename[i],true);
将溢写文件删除

## Shuffle过程源码剖析

### Map端Shuffle
在开始之前，我们注意到，Shuffle是Map端输出——>Reduce端接收过程。在这个过程中我们很容易想到一个问题，那就是，我们该怎么知道Map端输出是在源码哪个部分，我们这样想：既然要进行Map输出，那么Map任务一定已经开启了，所以我们找到MapTask类

首先贴出runNewMapper的源码：
```

 private <INKEY, INVALUE, OUTKEY, OUTVALUE> void runNewMapper(JobConf job, TaskSplitIndex splitIndex, TaskUmbilicalProtocol umbilical, TaskReporter reporter) throws IOException, ClassNotFoundException, InterruptedException {
        TaskAttemptContext taskContext = new TaskAttemptContextImpl(job, this.getTaskID(), reporter);//通过Driver类中的job和任务ID反射出·TaskAttemptContext·这个类在上篇文章中有出现，大家可以去回顾一下。这个类几乎贯穿了MR程序的一生。。。。
        Mapper<INKEY, INVALUE, OUTKEY, OUTVALUE> mapper = (Mapper)ReflectionUtils.newInstance(taskContext.getMapperClass(), job);//获取到我们自定义的Mapper类
        InputFormat<INKEY, INVALUE> inputFormat = (InputFormat)ReflectionUtils.newInstance(taskContext.getInputFormatClass(), job);//获取输入类型。默认的是`TextFileInputFormat`
        org.apache.hadoop.mapreduce.InputSplit split = null;
        split = (org.apache.hadoop.mapreduce.InputSplit)this.getSplitDetails(new Path(splitIndex.getSplitLocation()), splitIndex.getStartOffset());//获取逻辑分区
        LOG.info("Processing split: " + split);
        org.apache.hadoop.mapreduce.RecordReader<INKEY, INVALUE> input = new MapTask.NewTrackingRecordReader(split, inputFormat, reporter, taskContext);//获取Recordeader类。
        job.setBoolean("mapreduce.job.skiprecords", this.isSkipping());
        RecordWriter output = null;
        if (job.getNumReduceTasks() == 0) {
            output = new MapTask.NewDirectOutputCollector(taskContext, job, umbilical, reporter);
        } else {
            output = new MapTask.NewOutputCollector(taskContext, job, umbilical, reporter);
        }//这一段算是核心代码。看代码意思我们都能看出来它是Mapp端输出的主要类。也就是`OutputCollectoer`类

        MapContext<INKEY, INVALUE, OUTKEY, OUTVALUE> mapContext = new MapContextImpl(job, this.getTaskID(), input, (RecordWriter)output, this.committer, reporter, split);
        org.apache.hadoop.mapreduce.Mapper.Context mapperContext = (new WrappedMapper()).getMapContext(mapContext);

        try {
            input.initialize(split, mapperContext);
            mapper.run(mapperContext);//核心代码，该步骤会调到我们自己写的Mapper类中。
            this.mapPhase.complete();
            this.setPhase(Phase.SORT);
            this.statusUpdate(umbilical);
            input.close();
            input = null;
            ((RecordWriter)output).close(mapperContext);
            output = null;
        } finally {
            this.closeQuietly((org.apache.hadoop.mapreduce.RecordReader)input);
            this.closeQuietly((RecordWriter)output, mapperContext);
        }

    }
```
output = new MapTask.NewOutputCollector(taskContext, job, umbilical, reporter);这时我们点F5进入该方法内部，发现：
```

private class NewOutputCollector<K, V> extends RecordWriter<K, V> {
        private final MapOutputCollector<K, V> collector;
        private final Partitioner<K, V> partitioner;
        private final int partitions;

        NewOutputCollector(JobContext jobContext, JobConf job, TaskUmbilicalProtocol umbilical, TaskReporter reporter) throws IOException, ClassNotFoundException {
            this.collector = MapTask.this.createSortingCollector(job, reporter);
            this.partitions = jobContext.getNumReduceTasks();//获取到分区数。一会我们看一下它是怎么获取分区的。
            if (this.partitions > 1) {
                this.partitioner = (Partitioner)ReflectionUtils.newInstance(jobContext.getPartitionerClass(), job);//如果分区大于1，那么会根据分区来获取到分区类型。《这个会在之后的文章中详细讲解》
            } else {
                this.partitioner = new Partitioner<K, V>() {
                    public int getPartition(K key, V value, int numPartitions) {
                        return NewOutputCollector.this.partitions - 1;//如果没有设置，分区书是1，那么此时返回的值为0
                    }
                };
            }

        }

        public void write(K key, V value) throws IOException, InterruptedException {
            this.collector.collect(key, value, this.partitioner.getPartition(key, value, this.partitions));//核心方法，该方法的K-V值来自于Map端输出，也就是我们上篇文章中写到的context.writer(K,V)。那么这里我们可以看到，它会往对应的分区中写入该K，V值。
        }

        public void close(TaskAttemptContext context) throws IOException, InterruptedException {
            try {
                this.collector.flush();//核心方法，将数据刷出去。
            } catch (ClassNotFoundException var3) {
                throw new IOException("can't find class ", var3);
            }

            this.collector.close();
        }
    }
```
我们会进入MapTask类中的NewOutputCollector中，也就是上面的那个源码中的write方法。因为是核心方法，所以我们在这一步也点F5进入collect来完成源码刨析。这时我们进入了MapOutputBuffer类中，该类有collect方法。还有很多其他核心方法。我们先看collect方法：
```

  public synchronized void collect(K key, V value, int partition) throws IOException {
            this.reporter.progress();//汇报进程状态
            if (key.getClass() != this.keyClass) {
                throw new IOException("Type mismatch in key from map: expected " + this.keyClass.getName() + ", received " + key.getClass().getName());
            } else if (value.getClass() != this.valClass) {
                throw new IOException("Type mismatch in value from map: expected " + this.valClass.getName() + ", received " + value.getClass().getName());
            } else if (partition >= 0 && partition < this.partitions) {
                this.checkSpillException();
                this.bufferRemaining -= 16;
                int kvbidx;
                int kvbend;
                int bUsed;
                if (this.bufferRemaining <= 0) {
                    this.spillLock.lock();

                    try {
                        if (!this.spillInProgress) {
                            kvbidx = 4 * this.kvindex;
                            kvbend = 4 * this.kvend;
                            bUsed = this.distanceTo(kvbidx, this.bufindex);
                            boolean bufsoftlimit = bUsed >= this.softLimit;
                            if ((kvbend + 16) % this.kvbuffer.length != this.equator - this.equator % 16) {
                                this.resetSpill();//核心方法，重新开始溢写
                                this.bufferRemaining = Math.min(this.distanceTo(this.bufindex, kvbidx) - 32, this.softLimit - bUsed) - 16;
                            } else if (bufsoftlimit && this.kvindex != this.kvend) {
                                this.startSpill();//核心方法，开始溢写(因为溢写会有触发条件，所以我们如果一步一步Debug很难触发该操作，所以一会我们单独触发它。)
                                int avgRec = (int)(this.mapOutputByteCounter.getCounter() / this.mapOutputRecordCounter.getCounter());
                                int distkvi = this.distanceTo(this.bufindex, kvbidx);
                                int newPos = (this.bufindex + Math.max(31, Math.min(distkvi / 2, distkvi / (16 + avgRec) * 16))) % this.kvbuffer.length;
                                this.setEquator(newPos);
                                this.bufmark = this.bufindex = newPos;
                                int serBound = 4 * this.kvend;
                                this.bufferRemaining = Math.min(this.distanceTo(this.bufend, newPos), Math.min(this.distanceTo(newPos, serBound), this.softLimit)) - 32;
                            }
                        }
                    } finally {
                        this.spillLock.unlock();
                    }
                }

                try {
                    kvbidx = this.bufindex;
                    this.keySerializer.serialize(key);//序列化key
                    if (this.bufindex < kvbidx) {
                        this.bb.shiftBufferedKey();//写入keyBuffer
                        kvbidx = 0;
                    }

                    kvbend = this.bufindex;
                    this.valSerializer.serialize(value);//序列化value值
                    this.bb.write(this.b0, 0, 0);//写入值
                    bUsed = this.bb.markRecord();
                    this.mapOutputRecordCounter.increment(1L);//计数器+1
                    this.mapOutputByteCounter.increment((long)this.distanceTo(kvbidx, bUsed, this.bufvoid));
                    this.kvmeta.put(this.kvindex + 2, partition);//维护K-V元数据，分区相关
                    this.kvmeta.put(this.kvindex + 1, kvbidx);//
                    this.kvmeta.put(this.kvindex + 0, kvbend);//维护K-V元数据
                    this.kvmeta.put(this.kvindex + 3, this.distanceTo(kvbend, bUsed));//维护K-V元数据
                    this.kvindex = (this.kvindex - 4 + this.kvmeta.capacity()) % this.kvmeta.capacity();
                } catch (MapTask.MapBufferTooSmallException var15) {
                    MapTask.LOG.info("Record too large for in-memory buffer: " + var15.getMessage());
                    this.spillSingleRecord(key, value, partition);
                    this.mapOutputRecordCounter.increment(1L);
                }
            } else {
                throw new IOException("Illegal partition for " + key + " (" + partition + ")");
            }
        }
```
上面源码我们看到，在我们自定义的Mapper类中，会循环调用我们写的map方法，而在map方法内，我们使用context.write()将K-V值通过MapOutputBuffer类中的collect方法不停的往内存缓存区中写数据，这些数据的元数据包含了分区信息等，在内存缓存区到达一定的大小时，他就开始往外溢写数据，也就是collect方法中的this.startSpill();那么现在我们需要看的是Spill都干了什么事情。我们停止Debug，重新打断点，这次我们只在collect方法的this.startSpill();处打上断点。

进入详细的flush流程，我们发现我们又一次进入了MapOutputBuffer类中，该类中是有flush方法，那么到了这里我们可以这么任务，MapTask类的核心类为MapOutputBuffer类，它里面有MapTask输出的K-V值在内存中干的事。那么我们来看详细源码:
```

public void flush() throws IOException, ClassNotFoundException, InterruptedException {
            MapTask.LOG.info("Starting flush of map output");
            if (this.kvbuffer == null) {
                MapTask.LOG.info("kvbuffer is null. Skipping flush.");
            } else {
                this.spillLock.lock();//溢出加锁

                try {
                    while(this.spillInProgress) {//判断当前进度是否事溢写进度，如果是
                        this.reporter.progress();//汇报进度
                        this.spillDone.await();//等待当前溢写完成。
                    }

                    this.checkSpillException();
                    int kvbend = 4 * this.kvend;
                    if ((kvbend + 16) % this.kvbuffer.length != this.equator - this.equator % 16) {
                        this.resetSpill();//重置溢写的条件。
                    }

                    if (this.kvindex != this.kvend) {//如果当前kvindex不是kv的最后下标
                        this.kvend = (this.kvindex + 4) % this.kvmeta.capacity();
                        this.bufend = this.bufmark;
                        MapTask.LOG.info("Spilling map output");
                        MapTask.LOG.info("bufstart = " + this.bufstart + "; bufend = " + this.bufmark + "; bufvoid = " + this.bufvoid);
                        MapTask.LOG.info("kvstart = " + this.kvstart + "(" + this.kvstart * 4 + "); kvend = " + this.kvend + "(" + this.kvend * 4 + "); length = " + (this.distanceTo(this.kvend, this.kvstart, this.kvmeta.capacity()) + 1) + "/" + this.maxRec);
                        this.sortAndSpill();//核心代码，排序并溢出。一会我们详细看一下方法
                    }
                } catch (InterruptedException var7) {
                    throw new IOException("Interrupted while waiting for the writer", var7);
                } finally {
                    this.spillLock.unlock();
                }

                assert !this.spillLock.isHeldByCurrentThread();

                try {
                    this.spillThread.interrupt();
                    this.spillThread.join();
                } catch (InterruptedException var6) {
                    throw new IOException("Spill failed", var6);
                }

                this.kvbuffer = null;
                this.mergeParts();//核心代码，合并操作
                Path outputPath = this.mapOutputFile.getOutputFile();
                this.fileOutputByteCounter.increment(this.rfs.getFileStatus(outputPath).getLen());
            }
        }
```
虽然因为我们的输入文件太小没有触发spill操作，但是shullfe阶段总得将数据溢出，所以我们看到在flush阶段，它会触发sortAndSpill也就是排序并溢出操作。下面我们看一下sortAndSpill得源码:
```
       private void sortAndSpill() throws IOException, ClassNotFoundException, InterruptedException {
            long size = (long)(this.distanceTo(this.bufstart, this.bufend, this.bufvoid) + this.partitions * 150);//获取写出长度
            FSDataOutputStream out = null;//新建输出流

            try {
                SpillRecord spillRec = new SpillRecord(this.partitions);
                Path filename = this.mapOutputFile.getSpillFileForWrite(this.numSpills, size);//确认将数据溢出到那个文件中
                out = this.rfs.create(filename);
                int mstart = this.kvend / 4;
                int mend = 1 + (this.kvstart >= this.kvend ? this.kvstart : this.kvmeta.capacity() + this.kvstart) / 4;
                this.sorter.sort(this, mstart, mend, this.reporter);//核心方法，排序方法，默认为快速排序。(这里就是对key的排序)
                int spindex = mstart;
                IndexRecord rec = new IndexRecord();
                MapTask.MapOutputBuffer<K, V>.InMemValBytes value = new MapTask.MapOutputBuffer.InMemValBytes();

                for(int i = 0; i < this.partitions; ++i) //循环分区{
                    Writer writer = null;

                    try {
                        long segmentStart = out.getPos();
                        FSDataOutputStream partitionOut = CryptoUtils.wrapIfNecessary(this.job, out);
                        writer = new Writer(this.job, partitionOut, this.keyClass, this.valClass, this.codec, this.spilledRecordsCounter);
                        if (this.combinerRunner == null) {
                            for(DataInputBuffer key = new DataInputBuffer(); spindex < mend && this.kvmeta.get(this.offsetFor(spindex % this.maxRec) + 2) == i; ++spindex)//循环写入k-v {
                                int kvoff = this.offsetFor(spindex % this.maxRec);
                                int keystart = this.kvmeta.get(kvoff + 1);
                                int valstart = this.kvmeta.get(kvoff + 0);
                                key.reset(this.kvbuffer, keystart, valstart - keystart);
                                this.getVBytesForOffset(kvoff, value);
                                writer.append(key, value);
                            }
                        } else {
                            int spstart;
                            for(spstart = spindex; spindex < mend && this.kvmeta.get(this.offsetFor(spindex % this.maxRec) + 2) == i; ++spindex) {
                                ;
                            }

                            if (spstart != spindex) {
                                this.combineCollector.setWriter(writer);
                                RawKeyValueIterator kvIter = new MapTask.MapOutputBuffer.MRResultIterator(spstart, spindex);
                                this.combinerRunner.combine(kvIter, this.combineCollector);
                            }
                        }

                        writer.close();
                        rec.startOffset = segmentStart;
                        rec.rawLength = writer.getRawLength() + (long)CryptoUtils.cryptoPadding(this.job);
                        rec.partLength = writer.getCompressedLength() + (long)CryptoUtils.cryptoPadding(this.job);
                        spillRec.putIndex(rec, i);
                        writer = null;
                    } finally {
                        if (null != writer) {
                            writer.close();
                        }

                    }
                }

                if (this.totalIndexCacheMemory >= this.indexCacheMemoryLimit) {
                    Path indexFilename = this.mapOutputFile.getSpillIndexFileForWrite(this.numSpills, (long)(this.partitions * 24));
                    spillRec.writeToFile(indexFilename, this.job);//写如文件，从这里看，排序是在内存中完成的。
                } else {
                    this.indexCacheList.add(spillRec);
                    this.totalIndexCacheMemory += spillRec.size() * 24;
                }

                MapTask.LOG.info("Finished spill " + this.numSpills);
                ++this.numSpills;
            } finally {
                if (out != null) {
                    out.close();
                }

            }
        }
```
上面我们看完了排序的过程，那么排序完成后，我们会干什么呢？没错，合并相同Key的小文件mergeParts，这部分代码就不看了。合并完成就代表这map端结束。我们大概总结一下：

Key–>Partion–>[触发溢出]–>SortAndSpill–>Merge–>Reduce

我们看Map端在MatTask中，collect收集，根据分区规则将Key分区，然后触发Spill进而排序并溢出文件,在通过mergeParts合并相同Key的小文件，等待Reduce端拉去。OK，Map端源码刨析完毕！

### Reduce端shuffle
和Map端一样，我们需要找到ReduceTask类来分析，我们看一下它的run()方法：
```
 public void run(JobConf job, TaskUmbilicalProtocol umbilical) throws IOException, InterruptedException, ClassNotFoundException {
        job.setBoolean("mapreduce.job.skiprecords", this.isSkipping());
        if (this.isMapOrReduce()) {
            this.copyPhase = this.getProgress().addPhase("copy");
            this.sortPhase = this.getProgress().addPhase("sort");
            this.reducePhase = this.getProgress().addPhase("reduce");
        }

        TaskReporter reporter = this.startReporter(umbilical);
        boolean useNewApi = job.getUseNewReducer();
        this.initialize(job, this.getJobID(), reporter, useNewApi);//核心代码，初始化任务
        if (this.jobCleanup) {
            this.runJobCleanupTask(umbilical, reporter);
        } else if (this.jobSetup) {
            this.runJobSetupTask(umbilical, reporter);
        } else if (this.taskCleanup) {
            this.runTaskCleanupTask(umbilical, reporter);
        } else {
            this.codec = this.initCodec();
            RawKeyValueIterator rIter = null;
            ShuffleConsumerPlugin shuffleConsumerPlugin = null;//使用的shuffle插件
            Class combinerClass = this.conf.getCombinerClass();
            CombineOutputCollector combineCollector = null != combinerClass ? new CombineOutputCollector(this.reduceCombineOutputCounter, reporter, this.conf) : null;
            Class<? extends ShuffleConsumerPlugin> clazz = job.getClass("mapreduce.job.reduce.shuffle.consumer.plugin.class", Shuffle.class, ShuffleConsumerPlugin.class);
            shuffleConsumerPlugin = (ShuffleConsumerPlugin)ReflectionUtils.newInstance(clazz, job);
            LOG.info("Using ShuffleConsumerPlugin: " + shuffleConsumerPlugin);
            Context shuffleContext = new Context(this.getTaskID(), job, FileSystem.getLocal(job), umbilical, super.lDirAlloc, reporter, this.codec, combinerClass, combineCollector, this.spilledRecordsCounter, this.reduceCombineInputCounter, this.shuffledMapsCounter, this.reduceShuffleBytes, this.failedShuffleCounter, this.mergedMapOutputsCounter, this.taskStatus, this.copyPhase, this.sortPhase, this, this.mapOutputFile, this.localMapFiles);
            shuffleConsumerPlugin.init(shuffleContext);//初始化shuffle插件，核心代码
            rIter = shuffleConsumerPlugin.run();//跑shuflle核心代码,此步骤，会通过网络IO将Map端的输出给拉过来，并且进行合并操作~~~
            this.mapOutputFilesOnDisk.clear();
            this.sortPhase.complete();
            this.setPhase(Phase.REDUCE);
            this.statusUpdate(umbilical);
            Class keyClass = job.getMapOutputKeyClass();
            Class valueClass = job.getMapOutputValueClass();
            RawComparator comparator = job.getOutputValueGroupingComparator();
            if (useNewApi) {
                this.runNewReducer(job, umbilical, reporter, rIter, comparator, keyClass, valueClass);//肯定跑的是新API，核心代码
            } else {
                this.runOldReducer(job, umbilical, reporter, rIter, comparator, keyClass, valueClass);
            }

            shuffleConsumerPlugin.close();
            this.done(umbilical, reporter);
        }
    }
```
其实Reduce端的逻辑并不复杂，我们看到上面会产生shuffle插件，该插件中有一个MergeManager类的实例，在该类中，有这几行代码：
```
 this.inMemoryMerger = this.createInMemoryMerger();
                    this.inMemoryMerger.start();//内存中合并
                    this.onDiskMerger = new MergeManagerImpl.OnDiskMerger(this);//
                    this.onDiskMerger.start();//磁盘中合并
                    this.mergePhase = mergePhase;
```
也就是说在Reduce真正处理数据之前，会经历一次合并。合并完成之后，shuffle阶段算是结束了，Reduce开始真正执行处理逻辑。











































