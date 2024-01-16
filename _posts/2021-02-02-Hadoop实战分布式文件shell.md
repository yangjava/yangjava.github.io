---
layout: post
categories: [Hadoop]
description: none
keywords: Hadoop
---
# Hadoop实战分布式文件Shell

## 确定命令是什么？ 确定命令的位置是是哪里？ 确定命令执行的主类是哪一个？ 按照执行流程查看
例子：
发现hdfs dfsadmin -report存储指标和hdfs dfs -du -h /结果不一致，需要查看两张的统计逻辑的区别

确定命令的位置，which is hdfs
[ops@m-onedata bin]$ which is hdfs
/usr/bin/hdfs

查看脚本，cat /usr/bin/hdfs
exec /usr/.../hadoop-hdfs/bin/hdfs.distro "$@"

发现实际执行的脚本位置，继续查看执行脚本

elif [ "$COMMAND" = "dfs" ] ; then
CLASS=org.apache.hadoop.fs.FsShell
HADOOP_OPTS="$HADOOP_OPTS $HADOOP_CLIENT_OPTS"
elif [ "$COMMAND" = "dfsadmin" ] ; then
CLASS=org.apache.hadoop.hdfs.tools.DFSAdmin
HADOOP_OPTS="$HADOOP_OPTS $HADOOP_CLIENT_OPTS"
elif [ "$COMMAND" = "haadmin" ] ; then
CLASS=org.apache.hadoop.hdfs.tools.DFSHAAdmin
CLASSPATH=${CLASSPATH}:${TOOL_PATH}
HADOOP_OPTS="$HADOOP_OPTS $HADOOP_CLIENT_OPTS"
elif [ "$COMMAND" = "fsck" ] ; then
CLASS=org.apache.hadoop.hdfs.tools.DFSck
HADOOP_OPTS="$HADOOP_OPTS $HADOOP_CLIENT_OPTS"
elif [ "$COMMAND" = "balancer" ] ; then
CLASS=org.apache.hadoop.hdfs.server.balancer.Balancer
HADOOP_OPTS="$HADOOP_OPTS $HADOOP_BALANCER_OPTS"

exec "$JAVA" -Dproc_$COMMAND $JAVA_HEAP_MAX $HADOOP_OPTS $CLASS "$@"

找到实际执行的主类FsShell和DFSAdmin 查看执行的流程
发现hdfs dfsadmin -report存储指标和hdfs dfs -du -h /结果不一致，需要查看两张的统计逻辑的区别

## 排查流程
找到实际执行的主类FsShell和DFSAdmin hdfs dfs -du

FsShell
```
  /**
   * main() has some simple utility methods
   * @param argv the command and its arguments
   * @throws Exception upon error
   */
  public static void main(String argv[]) throws Exception {
    FsShell shell = newShellInstance();// 创建一个FsShell对象
    Configuration conf = new Configuration();
    conf.setQuietMode(false);
    shell.setConf(conf);
    int res;
    try {
      res = ToolRunner.run(shell, argv);// 执行FsShell的run方法
    } finally {
      shell.close();
    }
    System.exit(res);
  }
// -------------------------------分割线---------------------------------------
  /**
   * run
   */
  @Override
  public int run(String[] argv) {
    // initialize FsShell
    init();
    Tracer tracer = new Tracer.Builder("FsShell").
        conf(TraceUtils.wrapHadoopConf(SHELL_HTRACE_PREFIX, getConf())).
        build();
    int exitCode = -1;
    if (argv.length < 1) {
      printUsage(System.err);
    } else {
      String cmd = argv[0];
      Command instance = null;
      try {
        instance = commandFactory.getInstance(cmd);// 根据参数创建对应的Command对象
        if (instance == null) {
          throw new UnknownCommandException();
        }
        TraceScope scope = tracer.newScope(instance.getCommandName());
        if (scope.getSpan() != null) {
          String args = StringUtils.join(" ", argv);
          if (args.length() > 2048) {
            args = args.substring(0, 2048);
          }
          scope.getSpan().addKVAnnotation("args", args);
        }
        try {
          exitCode = instance.run(Arrays.copyOfRange(argv, 1, argv.length));// 执行命令
        } finally {
          scope.close();
        }
      } catch (IllegalArgumentException e) {
        if (e.getMessage() == null) {
          displayError(cmd, "Null exception message");
          e.printStackTrace(System.err);
        } else {
          displayError(cmd, e.getLocalizedMessage());
        }
        printUsage(System.err);
        if (instance != null) {
          printInstanceUsage(System.err, instance);
        }
      } catch (Exception e) {
        // instance.run catches IOE, so something is REALLY wrong if here
        LOG.debug("Error", e);
        displayError(cmd, "Fatal internal error");
        e.printStackTrace(System.err);
      }
    }
    tracer.close();
    return exitCode;
  }
```
CommandFactory
```
  /**
   * Get an instance of the requested command
   * @param cmdName name of the command to lookup
   * @param conf the hadoop configuration
   * @return the {@link Command} or null if the command is unknown
   */
  public Command getInstance(String cmdName, Configuration conf) {
    if (conf == null) throw new NullPointerException("configuration is null");
    
    Command instance = objectMap.get(cmdName);// 执行的时候添加的命令
    if (instance == null) {
      Class<? extends Command> cmdClass = classMap.get(cmdName);// 如果没有那么按照命令执行反射去创建命令的对象
      if (cmdClass != null) {
        instance = ReflectionUtils.newInstance(cmdClass, conf);
        instance.setName(cmdName);
        instance.setCommandFactory(this);
      }
    }
    return instance;
  }
```
既然是执行对应命令的command对象，那么就找到对应的对象即可，查看Command的继承树如下：
```
DFSAdminCommand in DFSAdmin (org.apache.hadoop.hdfs.tools)
    SetSpaceQuotaCommand in DFSAdmin (org.apache.hadoop.hdfs.tools)
    ClearSpaceQuotaCommand in DFSAdmin (org.apache.hadoop.hdfs.tools)
    SetQuotaCommand in DFSAdmin (org.apache.hadoop.hdfs.tools)
    ClearQuotaCommand in DFSAdmin (org.apache.hadoop.hdfs.tools)
FsCommand (org.apache.hadoop.fs.shell)
    Stat (org.apache.hadoop.fs.shell)
    SetfaclCommand in AclCommands (org.apache.hadoop.fs.shell)
    Mkdir (org.apache.hadoop.fs.shell)
    Display (org.apache.hadoop.fs.shell)
    FsShellPermissions (org.apache.hadoop.fs)
    Help in FsShell (org.apache.hadoop.fs)
    Truncate (org.apache.hadoop.fs.shell)
    Ls (org.apache.hadoop.fs.shell)
    Usage in FsShell (org.apache.hadoop.fs)
    Rmdir in Delete (org.apache.hadoop.fs.shell)
    DeleteSnapshot in SnapshotCommands (org.apache.hadoop.fs.shell)
    Find (org.apache.hadoop.fs.shell.find)
    InterruptCommand in TestFsShellReturnCode (org.apache.hadoop.fs)
    SetReplication (org.apache.hadoop.fs.shell)
    Count (org.apache.hadoop.fs.shell)
    SnapshotCommands (org.apache.hadoop.fs.shell)
    AclCommands (org.apache.hadoop.fs.shell)
    TouchCommands (org.apache.hadoop.fs.shell)
    RenameSnapshot in SnapshotCommands (org.apache.hadoop.fs.shell)
    FsUsage (org.apache.hadoop.fs.shell)
    XAttrCommands (org.apache.hadoop.fs.shell)
    Merge in CopyCommands (org.apache.hadoop.fs.shell)
    MoveToLocal in MoveCommands (org.apache.hadoop.fs.shell)
    GetfattrCommand in XAttrCommands (org.apache.hadoop.fs.shell)
    CommandWithDestination (org.apache.hadoop.fs.shell)
    Rm in Delete (org.apache.hadoop.fs.shell)
    Expunge in Delete (org.apache.hadoop.fs.shell)
    CreateSnapshot in SnapshotCommands (org.apache.hadoop.fs.shell)
    GetfaclCommand in AclCommands (org.apache.hadoop.fs.shell)
    SetfattrCommand in XAttrCommands (org.apache.hadoop.fs.shell)
    Head (org.apache.hadoop.fs.shell)
    Tail (org.apache.hadoop.fs.shell)
    Test (org.apache.hadoop.fs.shell)
```
我们可以很容易找到du对应的Command，使用IDEA的类搜索功能即可

Du
```
    @Override
    protected void processPath(PathData item) throws IOException {
      // 通过循环目录的子目录统计所有文件的大小信息进行累积
      ContentSummary contentSummary = item.fs.getContentSummary(item.path);
      long length = contentSummary.getLength();
      long spaceConsumed = contentSummary.getSpaceConsumed();
      if (excludeSnapshots) {
        length -= contentSummary.getSnapshotLength();
        spaceConsumed -= contentSummary.getSnapshotSpaceConsumed();
      }
      getUsagesTable().addRow(formatSize(length),
          formatSize(spaceConsumed), item);
    }
```

FileSystem
```
  /** Return the {@link ContentSummary} of a given {@link Path}.
   * @param f path to use
   * @throws FileNotFoundException if the path does not resolve
   * @throws IOException IO failure
   */
  public ContentSummary getContentSummary(Path f) throws IOException {
    FileStatus status = getFileStatus(f);
    if (status.isFile()) {
      // f is a file
      long length = status.getLen();
      return new ContentSummary.Builder().length(length).
          fileCount(1).directoryCount(0).spaceConsumed(length).build();
    }
    // f is a directory
    long[] summary = {0, 0, 1};
    for(FileStatus s : listStatus(f)) {
      long length = s.getLen();
      ContentSummary c = s.isDirectory() ? getContentSummary(s.getPath()) :
          new ContentSummary.Builder().length(length).
          fileCount(1).directoryCount(0).spaceConsumed(length).build();
      summary[0] += c.getLength();
      summary[1] += c.getFileCount();
      summary[2] += c.getDirectoryCount();
    }
    // 实际上我们可以看到能拿到的目录总个数，文件总大小，文件总个数
    return new ContentSummary.Builder().length(summary[0]).
        fileCount(summary[1]).directoryCount(summary[2]).
        spaceConsumed(summary[0]).build();
  }
```

hdfs dfsadmin -report
DFSAdmin
```
  /**
   * main() has some simple utility methods.
   * @param argv Command line parameters.
   * @exception Exception if the filesystem does not exist.
   */
  public static void main(String[] argv) throws Exception {
    int res = ToolRunner.run(new DFSAdmin(), argv);// 实际执行命令对象的run方法
    System.exit(res);
  }

  /**
   * @param argv The parameters passed to this program.
   * @exception Exception if the filesystem does not exist.
   * @return 0 on success, non zero on error.
   */
  @Override
  public int run(String[] argv) {

    if (argv.length < 1) {
      printUsage("");
      return -1;
    }
    // ... 省略部分判断代码
    try {
      if ("-report".equals(cmd)) {
        report(argv, i);// 和明显执行的是report方法
      } else if ("-safemode".equals(cmd)) {
          
      }
    }
    // ... 省略部分判断代码

   }
    
 /**
   * Gives a report on how the FileSystem is doing. fs的报告信息，也就是我们执行命令打印信息的地方
   * @exception IOException if the filesystem does not exist.
   */
  public void report(String[] argv, int i) throws IOException {
    DistributedFileSystem dfs = getDFS();
    FsStatus ds = dfs.getStatus();
    long capacity = ds.getCapacity();
    long used = ds.getUsed();// 重点看一下已经使用的容量是怎么统计的，这里逻辑是一定的，所以需要查看FsStatus是如何创建的
    long remaining = ds.getRemaining();
    long bytesInFuture = dfs.getBytesWithFutureGenerationStamps();
    long presentCapacity = used + remaining;
    boolean mode = dfs.setSafeMode(HdfsConstants.SafeModeAction.SAFEMODE_GET);
    if (mode) {
      System.out.println("Safe mode is ON");
      if (bytesInFuture > 0) {
        System.out.println("\nWARNING: ");
        System.out.println("Name node has detected blocks with generation " +
            "stamps in future.");
        System.out.println("Forcing exit from safemode will cause " +
            bytesInFuture + " byte(s) to be deleted.");
        System.out.println("If you are sure that the NameNode was started with"
            + " the correct metadata files then you may proceed with " +
            "'-safemode forceExit'\n");
      }
    }
    System.out.println("Configured Capacity: " + capacity
                       + " (" + StringUtils.byteDesc(capacity) + ")");
    System.out.println("Present Capacity: " + presentCapacity
        + " (" + StringUtils.byteDesc(presentCapacity) + ")");
    System.out.println("DFS Remaining: " + remaining
        + " (" + StringUtils.byteDesc(remaining) + ")");
    System.out.println("DFS Used: " + used
                       + " (" + StringUtils.byteDesc(used) + ")");
    double dfsUsedPercent = 0;
    if (presentCapacity != 0) {
      dfsUsedPercent = used/(double)presentCapacity;
    }
    System.out.println("DFS Used%: "
        + StringUtils.formatPercent(dfsUsedPercent, 2));

    /* These counts are not always upto date. They are updated after  
     * iteration of an internal list. Should be updated in a few seconds to 
     * minutes. Use "-metaSave" to list of all such blocks and accurate 
     * counts.
     */
    ReplicatedBlockStats replicatedBlockStats =
        dfs.getClient().getNamenode().getReplicatedBlockStats();
    System.out.println("Replicated Blocks:");
    System.out.println("\tUnder replicated blocks: " +
        replicatedBlockStats.getLowRedundancyBlocks());
    System.out.println("\tBlocks with corrupt replicas: " +
        replicatedBlockStats.getCorruptBlocks());
    System.out.println("\tMissing blocks: " +
        replicatedBlockStats.getMissingReplicaBlocks());
    System.out.println("\tMissing blocks (with replication factor 1): " +
        replicatedBlockStats.getMissingReplicationOneBlocks());
    if (replicatedBlockStats.hasHighestPriorityLowRedundancyBlocks()) {
      System.out.println("\tLow redundancy blocks with highest priority " +
          "to recover: " +
          replicatedBlockStats.getHighestPriorityLowRedundancyBlocks());
    }
// 此处省略部分无用代码
  }
```
DistributedFileSystem
```
  // 由FileSystem.getStatus()调用
    @Override 
  public FsStatus getStatus(Path p) throws IOException {
    statistics.incrementReadOps(1);
    storageStatistics.incrementOpCounter(OpType.GET_STATUS);
    return dfs.getDiskStatus();
  }
```
DFSClient
```
  /**
   * @see ClientProtocol#getStats()
   */
  public FsStatus getDiskStatus() throws IOException {
    return new FsStatus(getStateByIndex(0),
        getStateByIndex(1), getStateByIndex(2));
  }
```
NameNodeRpcServer
```
  @Override // ClientProtocol
  public long[] getStats() throws IOException {
    checkNNStartup();
    namesystem.checkOperation(OperationCategory.READ);
    return namesystem.getStats();
  }
```
FSNamesystem
```
  /** @see ClientProtocol#getStats() */
  long[] getStats() {
    final long[] stats = datanodeStatistics.getStats();// 可以看到这里是去收集datanode上的统计信息进行的汇总
    stats[ClientProtocol.GET_STATS_LOW_REDUNDANCY_IDX] =
        getLowRedundancyBlocks();
    stats[ClientProtocol.GET_STATS_CORRUPT_BLOCKS_IDX] =
        getCorruptReplicaBlocks();
    stats[ClientProtocol.GET_STATS_MISSING_BLOCKS_IDX] =
        getMissingBlocksCount();
    stats[ClientProtocol.GET_STATS_MISSING_REPL_ONE_BLOCKS_IDX] =
        getMissingReplOneBlocksCount();
    stats[ClientProtocol.GET_STATS_BYTES_IN_FUTURE_BLOCKS_IDX] =
        blockManager.getBytesInFuture();
    stats[ClientProtocol.GET_STATS_PENDING_DELETION_BLOCKS_IDX] =
        blockManager.getPendingDeletionBlocksCount();
    return stats;
  }
```
















