---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码集群启动

## 集群启动流程
让我们从启动流程开始，先在宏观上看看整个集群是如何启动的，集群状态如何从Red变成Green，不涉及代码，然后分析其他模块的流程。

本书中，集群启动过程指集群完全重启时的启动过程，期间要经历选举主节点、主分片、数据恢复等重要阶段，理解其中原理和细节，对于解决或避免集群维护过程中可能遇到的脑裂、无主、恢复慢、丢数据等问题有重要作用。

## 选举主节点
假设有若干节点正在启动，集群启动的第一件事是从已知的活跃机器列表中选择一个作为主节点，选主之后的流程由主节点触发。

ES的选主算法是基于Bully算法的改进，主要思路是对节点ID排序，取ID值最大的节点作为Master，每个节点都运行这个流程。是不是非常简单？

选主的目的是确定唯一的主节点，初学者可能认为选举出的主节点应该持有最新的元数据信息，实际上这个问题在实现上被分解为两步：先确定唯一的、大家公认的主节点，再想办法把最新的机器元数据复制到选举出的主节点上。

基于节点ID排序的简单选举算法有三个附加约定条件：
- 参选人数需要过半，达到 quorum（多数）后就选出了临时的主。为什么是临时的？每个节点运行排序取最大值的算法，结果不一定相同。

举个例子，集群有5台主机，节点ID分别是1、2、3、4、5。当产生网络分区或节点启动速度差异较大时，节点1看到的节点列表是1、2、3、4，选出4；节点2看到的节点列表是2、3、4、5，选出5。结果就不一致了，由此产生下面的第二条限制。

- 得票数需过半。某节点被选为主节点，必须判断加入它的节点数过半，才确认Master身份。解决第一个问题。

- 当探测到节点离开事件时，必须判断当前节点数是否过半。如果达不到 quorum，则放弃Master身份，重新加入集群。如果不这么做，则设想以下情况：

假设5台机器组成的集群产生网络分区，2台一组，3台一组，产生分区前，Master位于2台中的一个，此时3台一组的节点会重新并成功选取Master，产生双主，俗称脑裂。

集群并不知道自己共有多少个节点，quorum值从配置中读取，我们需要设置配置项：
```
discovery.zen.minimum_master_nodes
```

## 选举集群元信息
被选出的Master 和集群元信息的新旧程度没有关系。因此它的第一个任务是选举元信息，让具有master资格的节点把各自存储的元信息发过来（等待回复，必须收到所有节点的回复无论成功还是失败，没有超时。在收齐的这些回复中，有效元信息的总数必须达到指定数量），

根据版本号（较大版本号）确定最新的元信息，然后把这个信息广播下去，这样集群的所有节点（master资格节点和数据节点）都有了最新的元信息。

集群元信息的选举包括两个级别：集群级和索引级。不包含哪个shard存于哪个节点这种信息。这种信息以节点磁盘存储的为准，需要上报。为什么呢？因为读写流程是不经过Master的，Master 不知道各 shard 副本直接的数据差异。HDFS 也有类似的机制，block 信息依赖于DataNode的上报。

为了集群一致性，参与选举的元信息数量需要过半，Master发布集群状态成功的规则也是等待发布成功的节点数过半。

在选举过程中，不接受新节点的加入请求。 集群元信息选举完毕后，Master发布首次集群状态，然后开始选举shard级元信息。

## allocation过程
选举shard级元信息，构建内容路由表，是在allocation模块完成的。在初始阶段，所有的shard都处于UNASSIGNED（未分配）状态。ES中通过分配过程决定哪个分片位于哪个节点，重构内容路由表。此时，首先要做的是分配主分片。

### 选主分片
现在看某个主分片`[website][0]`是怎么分配的。所有的分配工作都是 Master 来做的，此时， Master不知道主分片在哪，它向集群的所有节点询问：大家把`[website][0]`分片的元信息发给我。然后，Master 等待所有的请求返回，正常情况下它就有了这个 shard 的信息，然后根据某种策略选一个分片作为主分片。

是不是效率有些低？这种询问量=shard 数×节点数。所以说我们最好控制shard的总规模别太大。

现在有了`shard[website][0]`的分片的多份信息，具体数量取决于副本数设置了多少。现在考虑把哪个分片作为主分片。ES 5.x以下的版本，通过对比shard级元信息的版本号来决定。在多副本的情况下，考虑到如果只有一个 shard 信息汇报上来，则它一定会被选为主分片，但也许数据不是最新的，版本号比它大的那个shard所在节点还没启动。

在解决这个问题的时候，ES 5.x开始实施一种新的策略：给每个 shard 都设置一个 UUID，然后在集群级的元信息中记录哪个shard是最新的，因为ES是先写主分片，再由主分片节点转发请求去写副分片，所以主分片所在节点肯定是最新的，如果它转发失败了，则要求Master删除那个节点。

所以，从ES 5.x开始，主分片选举过程是通过集群级元信息中记录的“最新主分片的列表”来确定主分片的：汇报信息中存在，并且这个列表中也存在。

如果集群设置了：
```
＂cluster.routing.allocation.enable＂: ＂none＂
```
禁止分配分片，集群仍会强制分配主分片。因此，在设置了上述选项的情况下，集群重启后的状态为Yellow，而非Red。

### 选副分片
主分片选举完成后，从上一个过程汇总的 shard 信息中选择一个副本作为副分片。如果汇总信息中不存在，则分配一个全新副本的操作依赖于延迟配置项：
```
index.unassigned.node_left.delayed_timeout
```
我们的线上环境中最大的集群有100+节点，掉节点的情况并不罕见，很多时候不能第一时间处理，这个延迟我们一般配置为以天为单位。 最后，allocation过程中允许新启动的节点加入集群。

## index recovery
分片分配成功后进入recovery流程。主分片的recovery不会等待其副分片分配成功才开始recovery。它们是独立的流程，只是副分片的recovery需要主分片恢复完毕才开始。

为什么需要recovery？对于主分片来说，可能有一些数据没来得及刷盘；对于副分片来说，一是没刷盘，二是主分片写完了，副分片还没来得及写，主副分片数据不一致。

### 主分片recovery
由于每次写操作都会记录事务日志（translog），事务日志中记录了哪种操作，以及相关的数据。因此将最后一次提交（Lucene 的一次提交就是一次 fsync 刷盘的过程）之后的 translog中进行重放，建立Lucene索引，如此完成主分片的recovery。

### 副分片recovery
副分片的恢复是比较复杂的，在ES的版本迭代中，副分片恢复策略有过不少调整。

副分片需要恢复成与主分片一致，同时，恢复期间允许新的索引操作。在目前的6.0版本中，恢复分成两阶段执行。

- phase1：在主分片所在节点，获取translog保留锁，从获取保留锁开始，会保留translog不受其刷盘清空的影响。然后调用Lucene接口把shard做快照，这是已经刷磁盘中的分片数据。把这些shard数据复制到副本节点。在phase1完毕前，会向副分片节点发送告知对方启动engine，在phase2开始之前，副分片就可以正常处理写请求了。

- phase2：对translog做快照，这个快照里包含从phase1开始，到执行translog快照期间的新增索引。将这些translog发送到副分片所在节点进行重放。

由于需要支持恢复期间的新增写操作（让ES的可用性更强），这两个阶段中需要重点关注以下几个问题。

分片数据完整性：如何做到副分片不丢数据？第二阶段的 translog 快照包括第一阶段所有的新增操作。那么第一阶段执行期间如果发生“Lucene commit”（将文件系统写缓冲中的数据刷盘，并清空translog），清除translog怎么办？在ES 2.0之前，是阻止了刷新操作，以此让translog都保留下来。从2.0版本开始，为了避免这种做法产生过大的translog，引入了translog.view的概念，创建 view 可以获取后续的所有操作。从6.0版本开始，translog.view 被移除。引入TranslogDeletionPolicy的概念，它将translog做一个快照来保持translog不被清理。这样实现了在第一阶段允许Lucene commit。

数据一致性：在ES 2.0之前，副分片恢复过程有三个阶段，第三阶段会阻塞新的索引操作，传输第二阶段执行期间新增的translog，这个时间很短。自2.0版本之后，第三阶段被删除，恢复期间没有任何写阻塞过程。在副分片节点，重放translog时，phase1和phase2之间的写操作与phase2重放操作之间的时序错误和冲突，通过写流程中进行异常处理，对比版本号来过滤掉过期操作。

这样，时序上存在错误的操作被忽略，对于特定的 doc，只有最新一次操作生效，保证了主副分片一致。

第一阶段尤其漫长，因为它需要从主分片拉取全量的数据。在ES 6.x中，对第一阶段再次优化：标记每个操作。在正常的写操作中，每次写入成功的操作都分配一个序号，通过对比序号就可以计算出差异范围，在实现方式上，添加了global checkpoint和local checkpoint，主分片负责维护global checkpoint，代表所有分片都已写入这个序号的位置，local checkpoint代表当前分片已写入成功的最新位置，恢复时通过对比两个序列号，计算出缺失的数据范围，然后通过translog重放这部分数据，同时translog会为此保留更长的时间。

因此，有两个机会可以跳过副分片恢复的phase1：基于SequenceNumber，从主分片节点的translog恢复数据；主副两分片有相同的syncid且doc数相同，可以跳过phase1。

## 集群启动日志
日志是分布式系统中排查问题的重要手段，虽然 ES 提供了很多便于排查问题的接口，但重要日志仍然是不可或缺的。默认情况下，ES输出的INFO级别日志较少，许多重要模块的关键环节是DEBUG 或TRACE 级别的。

## 启动流程做了什么
总体来说，节点启动流程的任务是做下面几类工作：
- 解析配置，包括配置文件和命令行参数。
- 检查外部环境和内部环境，例如，JVM版本、操作系统内核参数等。
- 初始化内部资源，创建内部模块，初始化探测器。
- 启动各个子模块和keepalive线程。

### 启动脚本
当我们通过启动脚本bin/elasticsearch启动ES时，脚本通过exec加载Java程序。

ES_JAVA_OPTS变量保存了JVM参数，其内容来自对config/jvm.options配置文件的解析。

如果执行启动脚本时添加了-d参数：

bin/elasticsearch –d

则启动脚本会在exec中添加<&- &。<&-的作用是关闭标准输入，即进程中的0号fd。&的作用是让进程在后台运行。

### 解析命令行参数和配置文件
实际工程应用中建议在启动参数中添加-d和-p，例如：
```
bin/elasticsearch -d -p es.pid
```
此处解析的配置文件有下面两个，jvm.options是在启动脚本中解析的。
- elasticsearch.yml #主要配置文件
- log4j2.properties #日志配置文件

### 加载安全配置
什么是安全配置？本质上是配置信息，既然是配置信息，一般是写到配置文件中的。ES的几个配置文件在之前的章节提到过。此处的“安全配置”是为了解决有些敏感的信息不适合放到配置文件中的，因为配置文件是明文保存的，虽然文件系统有基于用户权限的保护，但这仍然不够。因此ES把这些敏感配置信息加密，单独放到一个文件中：config/elasticsearch.keystore。然后提供一些命令来查看、添加和删除配置。

哪种配置信息适合放到安全配置文件中？例如，X-Pack中的security相关配置，LDAP的base_dn等信息（相当于登录服务器的用户名和密码）。

### 检查内部环境
内部环境指ES软件包本身的完整性和正确性。包括：
- 检查 Lucene 版本，ES 各版本对使用的 Lucene 版本是有要求的，在这里检查 Lucene版本以防止有人替换不兼容的jar包。
- 检测jar冲突（JarHell），发现冲突则退出进程。

## 源码
打开 server 模块下的 `Elasticsearch` 类：`org.elasticsearch.bootstrap.Elasticsearch`，运行里面的 main 函数就可以启动 `ElasticSearch` 了

可以看到入口其实是一个 main 方法，方法里面先是检查权限，然后是一个错误日志监听器（确保在日志配置之前状态日志没有出现 error），然后是 Elasticsearch 对象的创建，然后调用了静态方法 main 方法，并把创建的对象和参数以及 Terminal 默认值传进去。

静态的 main 方法里面调用 elasticsearch.main 方法。
```
public static void main(final String[] args) throws Exception {         //1、入口
    // we want the JVM to think there is a security manager installed so that if internal policy decisions that would be based on the
    // presence of a security manager or lack thereof act as if there is a security manager present (e.g., DNS cache policy)
    System.setSecurityManager(new SecurityManager() {
        @Override
        public void checkPermission(Permission perm) {
            // grant all permissions so that we can later set the security manager to the one that we want
        }
    });
    LogConfigurator.registerErrorListener();                            //
    final Elasticsearch elasticsearch = new Elasticsearch();
    int status = main(args, elasticsearch, Terminal.DEFAULT); //2、调用Elasticsearch.main方法
    if (status != ExitCodes.OK) {
        exit(status);
    }
}
static int main(final String[] args, final Elasticsearch elasticsearch, final Terminal terminal) throws Exception {
    return elasticsearch.main(args, terminal);  //3、command main
}
```
因为 Elasticsearch 类是继承了 EnvironmentAwareCommand 类，EnvironmentAwareCommand 类继承了 Command 类，但是 Elasticsearch 类并没有重写 main 方法，所以上面调用的 elasticsearch.main 其实是调用了 Command 的 main 方法，代码如下：
```
/** Parses options for this command from args and executes it. */
public final int main(String[] args, Terminal terminal) throws Exception {
    if (addShutdownHook()) {                                                //利用Runtime.getRuntime().addShutdownHook方法加入一个Hook，在程序退出时触发该Hook
        shutdownHookThread = new Thread(() -> {
            try {
                this.close();
            } catch (final IOException e) {
                try (
                    StringWriter sw = new StringWriter();
                    PrintWriter pw = new PrintWriter(sw)) {
                    e.printStackTrace(pw);
                    terminal.println(sw.toString());
                } catch (final IOException impossible) {
                    // StringWriter#close declares a checked IOException from the Closeable interface but the Javadocs for StringWriter
                    // say that an exception here is impossible
                    throw new AssertionError(impossible);
                }
            }
        });
        Runtime.getRuntime().addShutdownHook(shutdownHookThread);
    }
    beforeMain.run();
    try {
        mainWithoutErrorHandling(args, terminal);//4、mainWithoutErrorHandling
    } catch (OptionException e) {
        printHelp(terminal);
        terminal.println(Terminal.Verbosity.SILENT, "ERROR: " + e.getMessage());
        return ExitCodes.USAGE;
    } catch (UserException e) {
        if (e.exitCode == ExitCodes.USAGE) {
            printHelp(terminal);
        }
        terminal.println(Terminal.Verbosity.SILENT, "ERROR: " + e.getMessage());
        return e.exitCode;
    }
    return ExitCodes.OK;
}
```
上面代码一开始利用一个勾子函数，在程序退出时触发该 Hook，该方法主要代码是 mainWithoutErrorHandling() 方法，然后下面的是 catch 住方法抛出的异常，方法代码如下：
```
/*** Executes the command, but all errors are thrown. */
void mainWithoutErrorHandling(String[] args, Terminal terminal) throws Exception {
    final OptionSet options = parser.parse(args);
    if (options.has(helpOption)) {
        printHelp(terminal);
        return;
    }
    if (options.has(silentOption)) {
        terminal.setVerbosity(Terminal.Verbosity.SILENT);
    } else if (options.has(verboseOption)) {
        terminal.setVerbosity(Terminal.Verbosity.VERBOSE);
    } else {
        terminal.setVerbosity(Terminal.Verbosity.NORMAL);
    }
    execute(terminal, options);//5、执行 EnvironmentAwareCommand 中的 execute()，（重写了command里面抽象的execute方法）
}
```
解析传进来的参数并配置 terminal，重要的 execute() 方法，执行的是 EnvironmentAwareCommand 中的 execute() （重写了 Command 类里面的抽象 execute 方法），从上面那个继承图可以看到 EnvironmentAwareCommand 继承了 Command，重写的 execute 方法代码如下：
```
@Override
protected void execute(Terminal terminal, OptionSet options) throws Exception {
    final Map<String, String> settings = new HashMap<>();
    for (final KeyValuePair kvp : settingOption.values(options)) {
        if (kvp.value.isEmpty()) {
            throw new UserException(ExitCodes.USAGE, "setting [" + kvp.key + "] must not be empty");
        }
        if (settings.containsKey(kvp.key)) {
            final String message = String.format(
                Locale.ROOT, "setting [%s] already set, saw [%s] and [%s]",
                kvp.key, settings.get(kvp.key), kvp.value);
            throw new UserException(ExitCodes.USAGE, message);
        }
        settings.put(kvp.key, kvp.value);
    }
    //6、根据我们ide配置的 vm options 进行设置path.data、path.home、path.logs
    putSystemPropertyIfSettingIsMissing(settings, "path.data", "es.path.data");
    putSystemPropertyIfSettingIsMissing(settings, "path.home", "es.path.home");
    putSystemPropertyIfSettingIsMissing(settings, "path.logs", "es.path.logs");
    execute(terminal, options, createEnv(terminal, settings));//7、先调用 createEnv 创建环境
    //9、执行elasticsearch的execute方法，elasticsearch中重写了EnvironmentAwareCommand中的抽象execute方法
}
```
方法前面是根据传参去判断配置的，如果配置为空，就会直接跳到执行 putSystemPropertyIfSettingIsMissing 方法，这里会配置三个属性：path.data、path.home、path.logs 设置 es 的 data、home、logs 目录，它这里是根据我们 ide 配置的 vm options 进行设置的，如果不配置的话就会直接报错。

下面看看 putSystemPropertyIfSettingIsMissing 方法代码里面怎么做到的：
```
/** Ensure the given setting exists, reading it from system properties if not already set. */
private static void putSystemPropertyIfSettingIsMissing(final Map<String, String> settings, final String setting, final String key) {
    final String value = System.getProperty(key);//获取key（es.path.data）找系统设置
    if (value != null) {
        if (settings.containsKey(setting)) {
            final String message =
                String.format(
                Locale.ROOT,
                "duplicate setting [%s] found via command-line [%s] and system property [%s]",
                setting, settings.get(setting), value);
            throw new IllegalArgumentException(message);
        } else {
            settings.put(setting, value);
        }
    }
}
```
跳出此方法，继续看会发现 execute 方法调用了方法，
```
execute(terminal, options, createEnv(terminal, settings));
```
这里我们先看看 createEnv(terminal, settings) 方法：
```
protected Environment createEnv(final Terminal terminal, final Map<String, String> settings) throws UserException {
    final String esPathConf = System.getProperty("es.path.conf");//8、读取我们 vm options 中配置的 es.path.conf
    if (esPathConf == null) {
        throw new UserException(ExitCodes.CONFIG, "the system property [es.path.conf] must be set");
    }
    return InternalSettingsPreparer.prepareEnvironment(Settings.EMPTY, terminal, settings, getConfigPath(esPathConf));  //8、准备环境 prepareEnvironment
}
```
读取我们 ide vm options 中配置的 es.path.conf，因为 es 启动的时候会加载我们的配置和一些插件。
```
public static Environment prepareEnvironment(Settings input, Terminal terminal, Map<String, String> properties, Path configPath) {
    // just create enough settings to build the environment, to get the config dir
    Settings.Builder output = Settings.builder();
    initializeSettings(output, input, properties);
    Environment environment = new Environment(output.build(), configPath);
    //查看 es.path.conf 目录下的配置文件是不是 yml 格式的，如果不是则抛出一个异常
    if (Files.exists(environment.configFile().resolve("elasticsearch.yaml"))) {
        throw new SettingsException("elasticsearch.yaml was deprecated in 5.5.0 and must be renamed to elasticsearch.yml");
    }
    if (Files.exists(environment.configFile().resolve("elasticsearch.json"))) {
        throw new SettingsException("elasticsearch.json was deprecated in 5.5.0 and must be converted to elasticsearch.yml");
    }
    output = Settings.builder(); // start with a fresh output
    Path path = environment.configFile().resolve("elasticsearch.yml");
    if (Files.exists(path)) {
        try {
            output.loadFromPath(path);  //加载文件并读取配置文件内容
        } catch (IOException e) {
            throw new SettingsException("Failed to load settings from " + path.toString(), e);
        }
    }
    // re-initialize settings now that the config file has been loaded
    initializeSettings(output, input, properties);          //再一次初始化设置
    finalizeSettings(output, terminal);
    environment = new Environment(output.build(), configPath);
    // we put back the path.logs so we can use it in the logging configuration file
    output.put(Environment.PATH_LOGS_SETTING.getKey(), environment.logsFile().toAbsolutePath().normalize().toString());
    return new Environment(output.build(), configPath);
}
```
通过构建的环境查看配置文件 elasticsearch.yml 是不是以 yml 结尾，如果是 yaml 或者 json 结尾的则抛出异常（在 5.5.0 版本其他两种格式过期了，只能使用 yml 格式），然后加载该配置文件并读取里面的内容（KV结构）。

EnvironmentAwareCommand 类的 execute 方法代码如下：
```
protected abstract void execute(Terminal terminal, OptionSet options, Environment env) throws Exception;
```
这是个抽象方法，那么它的实现方法在 Elasticsearch 类中，代码如下：
```
@Override
protected void execute(Terminal terminal, OptionSet options, Environment env) throws UserException {
    if (options.nonOptionArguments().isEmpty() == false) {
        throw new UserException(ExitCodes.USAGE, "Positional arguments not allowed, found " + options.nonOptionArguments());
    }
    if (options.has(versionOption)) {
        final String versionOutput = String.format(
            Locale.ROOT,
            "Version: %s, Build: %s/%s/%s/%s, JVM: %s",
            Version.displayVersion(Version.CURRENT, Build.CURRENT.isSnapshot()),
            Build.CURRENT.flavor().displayName(),
            Build.CURRENT.type().displayName(),
            Build.CURRENT.shortHash(),
            Build.CURRENT.date(),
            JvmInfo.jvmInfo().version());
        terminal.println(versionOutput);
        return;
    }
    final boolean daemonize = options.has(daemonizeOption);
    final Path pidFile = pidfileOption.value(options);
    final boolean quiet = options.has(quietOption);
    // a misconfigured java.io.tmpdir can cause hard-to-diagnose problems later, so reject it immediately
    try {
        env.validateTmpFile();
    } catch (IOException e) {
        throw new UserException(ExitCodes.CONFIG, e.getMessage());
    }
    try {
        init(daemonize, pidFile, quiet, env);    //10、初始化
    } catch (NodeValidationException e) {
        throw new UserException(ExitCodes.CONFIG, e.getMessage());
    }
}
```
上面代码里主要还是看看 init(daemonize, pidFile, quiet, env); 初始化方法吧。
```
void init(final boolean daemonize, final Path pidFile, final boolean quiet, Environment initialEnv)
    throws NodeValidationException, UserException {
    try {
        Bootstrap.init(!daemonize, pidFile, quiet, initialEnv); //11、执行 Bootstrap 中的 init 方法
    } catch (BootstrapException | RuntimeException e) {
        // format exceptions to the console in a special way
        // to avoid 2MB stacktraces from guice, etc.
        throw new StartupException(e);
    }
}
```
Bootstrap 中的静态 init 方法如下：
```
static void init(
    final boolean foreground,
    final Path pidFile,
    final boolean quiet,
    final Environment initialEnv) throws BootstrapException, NodeValidationException, UserException {
    // force the class initializer for BootstrapInfo to run before
    // the security manager is installed
    BootstrapInfo.init();
    INSTANCE = new Bootstrap();   //12、创建一个 Bootstrap 实例
    final SecureSettings keystore = loadSecureSettings(initialEnv);//如果注册了安全模块则将相关配置加载进来
    final Environment environment = createEnvironment(foreground, pidFile, keystore, initialEnv.settings(), initialEnv.configFile());   //干之前干过的事情
    try {
        LogConfigurator.configure(environment);   //13、log 配置环境
    } catch (IOException e) {
        throw new BootstrapException(e);
    }
    if (environment.pidFile() != null) {
        try {
            PidFile.create(environment.pidFile(), true);
        } catch (IOException e) {
            throw new BootstrapException(e);
        }
    }
    final boolean closeStandardStreams = (foreground == false) || quiet;
    try {
        if (closeStandardStreams) {
            final Logger rootLogger = ESLoggerFactory.getRootLogger();
            final Appender maybeConsoleAppender = Loggers.findAppender(rootLogger, ConsoleAppender.class);
            if (maybeConsoleAppender != null) {
                Loggers.removeAppender(rootLogger, maybeConsoleAppender);
            }
            closeSystOut();
        }
        // fail if somebody replaced the lucene jars
        checkLucene();             //14、检查Lucene版本
// install the default uncaught exception handler; must be done before security is initialized as we do not want to grant the runtime permission setDefaultUncaughtExceptionHandler
        Thread.setDefaultUncaughtExceptionHandler(
            new ElasticsearchUncaughtExceptionHandler(() -> Node.NODE_NAME_SETTING.get(environment.settings())));
        INSTANCE.setup(true, environment);      //15、调用 setup 方法
        try {
            // any secure settings must be read during node construction
            IOUtils.close(keystore);
        } catch (IOException e) {
            throw new BootstrapException(e);
        }
        INSTANCE.start();         //26、调用 start 方法
        if (closeStandardStreams) {
            closeSysError();
        }
    } catch (NodeValidationException | RuntimeException e) {
        // disable console logging, so user does not see the exception twice (jvm will show it already)
        final Logger rootLogger = ESLoggerFactory.getRootLogger();
        final Appender maybeConsoleAppender = Loggers.findAppender(rootLogger, ConsoleAppender.class);
        if (foreground && maybeConsoleAppender != null) {
            Loggers.removeAppender(rootLogger, maybeConsoleAppender);
        }
        Logger logger = Loggers.getLogger(Bootstrap.class);
        if (INSTANCE.node != null) {
            logger = Loggers.getLogger(Bootstrap.class, Node.NODE_NAME_SETTING.get(INSTANCE.node.settings()));
        }
        // HACK, it sucks to do this, but we will run users out of disk space otherwise
        if (e instanceof CreationException) {
            // guice: log the shortened exc to the log file
            ByteArrayOutputStream os = new ByteArrayOutputStream();
            PrintStream ps = null;
            try {
                ps = new PrintStream(os, false, "UTF-8");
            } catch (UnsupportedEncodingException uee) {
                assert false;
                e.addSuppressed(uee);
            }
            new StartupException(e).printStackTrace(ps);
            ps.flush();
            try {
                logger.error("Guice Exception: {}", os.toString("UTF-8"));
            } catch (UnsupportedEncodingException uee) {
                assert false;
                e.addSuppressed(uee);
            }
        } else if (e instanceof NodeValidationException) {
            logger.error("node validation exception\n{}", e.getMessage());
        } else {
            // full exception
            logger.error("Exception", e);
        }
        // re-enable it if appropriate, so they can see any logging during the shutdown process
        if (foreground && maybeConsoleAppender != null) {
            Loggers.addAppender(rootLogger, maybeConsoleAppender);
        }
        throw e;
    }
}
```
该方法主要有：
- 创建 Bootstrap 实例
- 如果注册了安全模块则将相关配置加载进来
- 创建 Elasticsearch 运行的必须环境以及相关配置, 如将 config、scripts、plugins、modules、logs、lib、bin 等配置目录加载到运行环境中
- log 配置环境，创建日志上下文
- 检查是否存在 PID 文件，如果不存在，创建 PID 文件
- 检查 Lucene 版本
- 调用 setup 方法（用当前环境来创建一个节点）

```
private void setup(boolean addShutdownHook, Environment environment) throws BootstrapException {
    Settings settings = environment.settings();//根据环境得到配置
    try {
        spawner.spawnNativeControllers(environment);
    } catch (IOException e) {
        throw new BootstrapException(e);
    }
    initializeNatives(
        environment.tmpFile(),
        BootstrapSettings.MEMORY_LOCK_SETTING.get(settings),
        BootstrapSettings.SYSTEM_CALL_FILTER_SETTING.get(settings),
        BootstrapSettings.CTRLHANDLER_SETTING.get(settings));
    // initialize probes before the security manager is installed
    initializeProbes();
    if (addShutdownHook) {
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                try {
                    IOUtils.close(node, spawner);
                    LoggerContext context = (LoggerContext) LogManager.getContext(false);
                    Configurator.shutdown(context);
                } catch (IOException ex) {
                    throw new ElasticsearchException("failed to stop node", ex);
                }
            }
        });
    }
    try {
        // look for jar hell
        final Logger logger = ESLoggerFactory.getLogger(JarHell.class);
        JarHell.checkJarHell(logger::debug);
    } catch (IOException | URISyntaxException e) {
        throw new BootstrapException(e);
    }
    // Log ifconfig output before SecurityManager is installed
    IfConfig.logIfNecessary();
    // install SM after natives, shutdown hooks, etc.
    try {
        Security.configure(environment, BootstrapSettings.SECURITY_FILTER_BAD_DEFAULTS_SETTING.get(settings));
    } catch (IOException | NoSuchAlgorithmException e) {
        throw new BootstrapException(e);
    }
    node = new Node(environment) {              //16、新建节点
        @Override
        protected void validateNodeBeforeAcceptingRequests(
            final BootstrapContext context,
            final BoundTransportAddress boundTransportAddress, List<BootstrapCheck> checks) throws NodeValidationException {
            BootstrapChecks.check(context, boundTransportAddress, checks);
        }
    };
}
```
Node 的创建
```
public Node(Environment environment) {

        this(environment, Collections.emptyList()); //执行下面的代码

    }

protected Node(final Environment environment, Collection<Class<? extends Plugin>> classpathPlugins) {

    final List<Closeable> resourcesToClose = new ArrayList<>(); // register everything we need to release in the case of an error

    boolean success = false;

    {

// use temp logger just to say we are starting. we can't use it later on because the node name might not be set

        Logger logger = Loggers.getLogger(Node.class, NODE_NAME_SETTING.get(environment.settings()));

        logger.info("initializing ...");

    }

    try {

        originalSettings = environment.settings();

        Settings tmpSettings = Settings.builder().put(environment.settings())

            .put(Client.CLIENT_TYPE_SETTING_S.getKey(), CLIENT_TYPE).build();

// create the node environment as soon as possible, to recover the node id and enable logging

        try {

            nodeEnvironment = new NodeEnvironment(tmpSettings, environment); //1、创建节点环境,比如节点名称,节点ID,分片信息,存储元,以及分配内存准备给节点使用

            resourcesToClose.add(nodeEnvironment);

        } catch (IOException ex) {

        throw new IllegalStateException("Failed to create node environment", ex);

        }

        final boolean hadPredefinedNodeName = NODE_NAME_SETTING.exists(tmpSettings);

        final String nodeId = nodeEnvironment.nodeId();

        tmpSettings = addNodeNameIfNeeded(tmpSettings, nodeId);

        final Logger logger = Loggers.getLogger(Node.class, tmpSettings);

// this must be captured after the node name is possibly added to the settings

        final String nodeName = NODE_NAME_SETTING.get(tmpSettings);

        if (hadPredefinedNodeName == false) {

            logger.info("node name derived from node ID [{}]; set [{}] to override", nodeId, NODE_NAME_SETTING.getKey());

        } else {

            logger.info("node name [{}], node ID [{}]", nodeName, nodeId);

        }

        //2、打印出JVM相关信息

        final JvmInfo jvmInfo = JvmInfo.jvmInfo();

        logger.info(

"version[{}], pid[{}], build[{}/{}/{}/{}], OS[{}/{}/{}], JVM[{}/{}/{}/{}]",

            Version.displayVersion(Version.CURRENT, Build.CURRENT.isSnapshot()),

            jvmInfo.pid(), Build.CURRENT.flavor().displayName(),

            Build.CURRENT.type().displayName(), Build.CURRENT.shortHash(),

            Build.CURRENT.date(), Constants.OS_NAME, Constants.OS_VERSION,

            Constants.OS_ARCH,Constants.JVM_VENDOR,Constants.JVM_NAME,

            Constants.JAVA_VERSION,Constants.JVM_VERSION);

        logger.info("JVM arguments {}", Arrays.toString(jvmInfo.getInputArguments()));

        //检查当前版本是不是 pre-release 版本（Snapshot），

        warnIfPreRelease(Version.CURRENT, Build.CURRENT.isSnapshot(), logger);

		。。。

        this.pluginsService = new PluginsService(tmpSettings, environment.configFile(), environment.modulesFile(), environment.pluginsFile(), classpathPlugins);   //3、利用PluginsService加载相应的模块和插件

        this.settings = pluginsService.updatedSettings();

        localNodeFactory = new LocalNodeFactory(settings, nodeEnvironment.nodeId());

// create the environment based on the finalized (processed) view of the settings

// this is just to makes sure that people get the same settings, no matter where they ask them from

        this.environment = new Environment(this.settings, environment.configFile());

        Environment.assertEquivalent(environment, this.environment);

        final List<ExecutorBuilder<?>> executorBuilders = pluginsService.getExecutorBuilders(settings);        //线程池

        final ThreadPool threadPool = new ThreadPool(settings, executorBuilders.toArray(new ExecutorBuilder[0]));

        resourcesToClose.add(() -> ThreadPool.terminate(threadPool, 10, TimeUnit.SECONDS));

        // adds the context to the DeprecationLogger so that it does not need to be injected everywhere

        DeprecationLogger.setThreadContext(threadPool.getThreadContext());

        resourcesToClose.add(() -> DeprecationLogger.removeThreadContext(threadPool.getThreadContext()));

        final List<Setting<?>> additionalSettings = new ArrayList<>(pluginsService.getPluginSettings());       //额外配置

        final List<String> additionalSettingsFilter = new ArrayList<>(pluginsService.getPluginSettingsFilter());

        for (final ExecutorBuilder<?> builder : threadPool.builders()) {

            //4、加载一些额外配置

            additionalSettings.addAll(builder.getRegisteredSettings());

        }

        client = new NodeClient(settings, threadPool);//5、创建一个节点客户端

        //6、缓存一系列模块,如NodeModule,ClusterModule,IndicesModule,ActionModule,GatewayModule,SettingsModule,RepositioriesModule，scriptModule，analysisModule

        final ResourceWatcherService resourceWatcherService = new ResourceWatcherService(settings, threadPool);

        final ScriptModule scriptModule = new ScriptModule(settings, pluginsService.filterPlugins(ScriptPlugin.class));

        AnalysisModule analysisModule = new AnalysisModule(this.environment, pluginsService.filterPlugins(AnalysisPlugin.class));

        // this is as early as we can validate settings at this point. we already pass them to ScriptModule as well as ThreadPool so we might be late here already

        final SettingsModule settingsModule = new SettingsModule(this.settings, additionalSettings, additionalSettingsFilter);

scriptModule.registerClusterSettingsListeners(settingsModule.getClusterSettings());

        resourcesToClose.add(resourceWatcherService);

        final NetworkService networkService = new NetworkService(

  getCustomNameResolvers(pluginsService.filterPlugins(DiscoveryPlugin.class)));

        List<ClusterPlugin> clusterPlugins = pluginsService.filterPlugins(ClusterPlugin.class);

        final ClusterService clusterService = new ClusterService(settings, settingsModule.getClusterSettings(), threadPool,                                                      ClusterModule.getClusterStateCustomSuppliers(clusterPlugins));

        clusterService.addStateApplier(scriptModule.getScriptService());

        resourcesToClose.add(clusterService);

        final IngestService ingestService = new IngestService(settings, threadPool, this.environment,                                                  scriptModule.getScriptService(), analysisModule.getAnalysisRegistry(), pluginsService.filterPlugins(IngestPlugin.class));

        final DiskThresholdMonitor listener = new DiskThresholdMonitor(settings, clusterService::state, clusterService.getClusterSettings(), client);

        final ClusterInfoService clusterInfoService = newClusterInfoService(settings, clusterService, threadPool, client,

listener::onNewInfo);

        final UsageService usageService = new UsageService(settings);

        ModulesBuilder modules = new ModulesBuilder();

// plugin modules must be added here, before others or we can get crazy injection errors...

        for (Module pluginModule : pluginsService.createGuiceModules()) {

            modules.add(pluginModule);

        }

        final MonitorService monitorService = new MonitorService(settings, nodeEnvironment, threadPool, clusterInfoService);

        ClusterModule clusterModule = new ClusterModule(settings, clusterService, clusterPlugins, clusterInfoService);

        modules.add(clusterModule);

        IndicesModule indicesModule = new IndicesModule(pluginsService.filterPlugins(MapperPlugin.class));

        modules.add(indicesModule);

        SearchModule searchModule = new SearchModule(settings, false, pluginsService.filterPlugins(SearchPlugin.class));

        CircuitBreakerService circuitBreakerService = createCircuitBreakerService(settingsModule.getSettings(),

                                                                                  settingsModule.getClusterSettings());

        resourcesToClose.add(circuitBreakerService);

        modules.add(new GatewayModule());

        PageCacheRecycler pageCacheRecycler = createPageCacheRecycler(settings);

        BigArrays bigArrays = createBigArrays(pageCacheRecycler, circuitBreakerService);

        resourcesToClose.add(bigArrays);

        modules.add(settingsModule);

        List<NamedWriteableRegistry.Entry> namedWriteables = Stream.of(

            NetworkModule.getNamedWriteables().stream(),

            indicesModule.getNamedWriteables().stream(),

            searchModule.getNamedWriteables().stream(),

            pluginsService.filterPlugins(Plugin.class).stream()

            .flatMap(p -> p.getNamedWriteables().stream()),

            ClusterModule.getNamedWriteables().stream())

            .flatMap(Function.identity()).collect(Collectors.toList());

        final NamedWriteableRegistry namedWriteableRegistry = new NamedWriteableRegistry(namedWriteables);

        NamedXContentRegistry xContentRegistry = new NamedXContentRegistry(Stream.of(

            NetworkModule.getNamedXContents().stream(),

            searchModule.getNamedXContents().stream(),

            pluginsService.filterPlugins(Plugin.class).stream()

            .flatMap(p -> p.getNamedXContent().stream()),

            ClusterModule.getNamedXWriteables().stream())

.flatMap(Function.identity()).collect(toList()));

        modules.add(new RepositoriesModule(this.environment, pluginsService.filterPlugins(RepositoryPlugin.class), xContentRegistry));

        final MetaStateService metaStateService = new MetaStateService(settings, nodeEnvironment, xContentRegistry);

        final IndicesService indicesService = new IndicesService(settings, pluginsService, nodeEnvironment, xContentRegistry,

analysisModule.getAnalysisRegistry(),                                                                clusterModule.getIndexNameExpressionResolver(), indicesModule.getMapperRegistry(), namedWriteableRegistry,threadPool, settingsModule.getIndexScopedSettings(), circuitBreakerService, bigArrays, scriptModule.getScriptService(),client, metaStateService);

        Collection<Object> pluginComponents = pluginsService.filterPlugins(Plugin.class).stream()

            .flatMap(p -> p.createComponents(client, clusterService, threadPool, resourceWatcherService,scriptModule.getScriptService(), xContentRegistry, environment, nodeEnvironment,namedWriteableRegistry).stream())

.collect(Collectors.toList());

        ActionModule actionModule = new ActionModule(false, settings, clusterModule.getIndexNameExpressionResolver(),

                                                     settingsModule.getIndexScopedSettings(), settingsModule.getClusterSettings(), settingsModule.getSettingsFilter(),threadPool, pluginsService.filterPlugins(ActionPlugin.class), client, circuitBreakerService, usageService);

        modules.add(actionModule);

        //7、获取RestController,用于处理各种Elasticsearch的rest命令,如_cat,_all,_cat/health,_clusters等rest命令(Elasticsearch称之为action)

        final RestController restController = actionModule.getRestController();

        final NetworkModule networkModule = new NetworkModule(settings, false, pluginsService.filterPlugins(NetworkPlugin.class),threadPool, bigArrays, pageCacheRecycler, circuitBreakerService, namedWriteableRegistry, xContentRegistry,networkService, restController);

        Collection<UnaryOperator<Map<String, MetaData.Custom>>> customMetaDataUpgraders =

            pluginsService.filterPlugins(Plugin.class).stream()

            .map(Plugin::getCustomMetaDataUpgrader)

            .collect(Collectors.toList());

        Collection<UnaryOperator<Map<String, IndexTemplateMetaData>>> indexTemplateMetaDataUpgraders =

            pluginsService.filterPlugins(Plugin.class).stream()

            .map(Plugin::getIndexTemplateMetaDataUpgrader)

            .collect(Collectors.toList());

        Collection<UnaryOperator<IndexMetaData>> indexMetaDataUpgraders = pluginsService.filterPlugins(Plugin.class).stream()

            .map(Plugin::getIndexMetaDataUpgrader).collect(Collectors.toList());

        final MetaDataUpgrader metaDataUpgrader = new MetaDataUpgrader(customMetaDataUpgraders, indexTemplateMetaDataUpgraders);

        final MetaDataIndexUpgradeService metaDataIndexUpgradeService = new MetaDataIndexUpgradeService(settings, xContentRegistry,                                                                                            indicesModule.getMapperRegistry(), settingsModule.getIndexScopedSettings(), indexMetaDataUpgraders);

        final GatewayMetaState gatewayMetaState = new GatewayMetaState(settings, nodeEnvironment, metaStateService,                                                      metaDataIndexUpgradeService, metaDataUpgrader);

        new TemplateUpgradeService(settings, client, clusterService, threadPool, indexTemplateMetaDataUpgraders);

        final Transport transport = networkModule.getTransportSupplier().get();

        Set<String> taskHeaders = Stream.concat(

            pluginsService.filterPlugins(ActionPlugin.class).stream().flatMap(p -> p.getTaskHeaders().stream()),

            Stream.of("X-Opaque-Id")

        ).collect(Collectors.toSet());

        final TransportService transportService = newTransportService(settings, transport, threadPool,

                                                                      networkModule.getTransportInterceptor(), localNodeFactory, settingsModule.getClusterSettings(), taskHeaders);

        final ResponseCollectorService responseCollectorService = new ResponseCollectorService(this.settings, clusterService);

        final SearchTransportService searchTransportService =  new SearchTransportService(settings, transportService,

                                                                                          SearchExecutionStatsCollector.makeWrapper(responseCollectorService));

        final Consumer<Binder> httpBind;

        final HttpServerTransport httpServerTransport;

        if (networkModule.isHttpEnabled()) {

            httpServerTransport = networkModule.getHttpServerTransportSupplier().get();

            httpBind = b -> {

b.bind(HttpServerTransport.class).toInstance(httpServerTransport);

            };

        } else {

            httpBind = b -> {

                b.bind(HttpServerTransport.class).toProvider(Providers.of(null));

            };

            httpServerTransport = null;

        }

        final DiscoveryModule discoveryModule = new DiscoveryModule(this.settings, threadPool, transportService, namedWriteableRegistry,networkService, clusterService.getMasterService(), clusterService.getClusterApplierService(),clusterService.getClusterSettings(), pluginsService.filterPlugins(DiscoveryPlugin.class),clusterModule.getAllocationService());

        this.nodeService = new NodeService(settings, threadPool, monitorService, discoveryModule.getDiscovery(),transportService, indicesService, pluginsService, circuitBreakerService, scriptModule.getScriptService(),httpServerTransport, ingestService, clusterService, settingsModule.getSettingsFilter(), responseCollectorService,searchTransportService);

        final SearchService searchService = newSearchService(clusterService, indicesService, threadPool, scriptModule.getScriptService(), bigArrays, searchModule.getFetchPhase(),responseCollectorService);

        final List<PersistentTasksExecutor<?>> tasksExecutors = pluginsService

            .filterPlugins(PersistentTaskPlugin.class).stream()

     .map(p -> p.getPersistentTasksExecutor(clusterService, threadPool, client))

            .flatMap(List::stream)

            .collect(toList());

        final PersistentTasksExecutorRegistry registry = new PersistentTasksExecutorRegistry(settings, tasksExecutors);

        final PersistentTasksClusterService persistentTasksClusterService =

            new PersistentTasksClusterService(settings, registry, clusterService);

        final PersistentTasksService persistentTasksService = new PersistentTasksService(settings, clusterService, threadPool, client);

//8、绑定处理各种服务的实例,这里是最核心的地方,也是Elasticsearch能处理各种服务的核心.

        modules.add(b -> {

            b.bind(Node.class).toInstance(this);

            b.bind(NodeService.class).toInstance(nodeService);

            b.bind(NamedXContentRegistry.class).toInstance(xContentRegistry);

            b.bind(PluginsService.class).toInstance(pluginsService);

            b.bind(Client.class).toInstance(client);

            b.bind(NodeClient.class).toInstance(client);

            b.bind(Environment.class).toInstance(this.environment);

            b.bind(ThreadPool.class).toInstance(threadPool);

            b.bind(NodeEnvironment.class).toInstance(nodeEnvironment);

 b.bind(ResourceWatcherService.class).toInstance(resourceWatcherService);

b.bind(CircuitBreakerService.class).toInstance(circuitBreakerService);

            b.bind(BigArrays.class).toInstance(bigArrays);

      b.bind(ScriptService.class).toInstance(scriptModule.getScriptService());

 b.bind(AnalysisRegistry.class).toInstance(analysisModule.getAnalysisRegistry());

            b.bind(IngestService.class).toInstance(ingestService);

            b.bind(UsageService.class).toInstance(usageService);

 b.bind(NamedWriteableRegistry.class).toInstance(namedWriteableRegistry);

            b.bind(MetaDataUpgrader.class).toInstance(metaDataUpgrader);

            b.bind(MetaStateService.class).toInstance(metaStateService);

            b.bind(IndicesService.class).toInstance(indicesService);

            b.bind(SearchService.class).toInstance(searchService);            b.bind(SearchTransportService.class).toInstance(searchTransportService);

b.bind(SearchPhaseController.class).toInstance(new SearchPhaseController(settings, searchService::createReduceContext));

            b.bind(Transport.class).toInstance(transport);

            b.bind(TransportService.class).toInstance(transportService);

            b.bind(NetworkService.class).toInstance(networkService);

            b.bind(UpdateHelper.class).toInstance(new UpdateHelper(settings, scriptModule.getScriptService()));

b.bind(MetaDataIndexUpgradeService.class).toInstance(metaDataIndexUpgradeService);

            b.bind(ClusterInfoService.class).toInstance(clusterInfoService);

            b.bind(GatewayMetaState.class).toInstance(gatewayMetaState);

            b.bind(Discovery.class).toInstance(discoveryModule.getDiscovery());

            {

                RecoverySettings recoverySettings = new RecoverySettings(settings, settingsModule.getClusterSettings());

                processRecoverySettings(settingsModule.getClusterSettings(), recoverySettings);

                b.bind(PeerRecoverySourceService.class).toInstance(new PeerRecoverySourceService(settings, transportService,

indicesService, recoverySettings));

                b.bind(PeerRecoveryTargetService.class).toInstance(new PeerRecoveryTargetService(settings, threadPool,

transportService, recoverySettings, clusterService));

            }

            httpBind.accept(b);

            pluginComponents.stream().forEach(p -> b.bind((Class) p.getClass()).toInstance(p));

b.bind(PersistentTasksService.class).toInstance(persistentTasksService);       b.bind(PersistentTasksClusterService.class).toInstance(persistentTasksClusterService);

b.bind(PersistentTasksExecutorRegistry.class).toInstance(registry); });

        injector = modules.createInjector();

        // TODO hack around circular dependencies problems in AllocationService

clusterModule.getAllocationService().setGatewayAllocator(injector.getInstance(GatewayAllocator.class));

        List<LifecycleComponent> pluginLifecycleComponents = pluginComponents.stream()

            .filter(p -> p instanceof LifecycleComponent)

            .map(p -> (LifecycleComponent) p).collect(Collectors.toList());

        //9、利用Guice将各种模块以及服务(xxxService)注入到Elasticsearch环境中

pluginLifecycleComponents.addAll(pluginsService.getGuiceServiceClasses().stream()                                     .map(injector::getInstance).collect(Collectors.toList()));

        resourcesToClose.addAll(pluginLifecycleComponents);

        this.pluginLifecycleComponents = Collections.unmodifiableList(pluginLifecycleComponents);

        client.initialize(injector.getInstance(new Key<Map<GenericAction, TransportAction>>() {}), () -> clusterService.localNode().getId(), transportService.getRemoteClusterService());

        if (NetworkModule.HTTP_ENABLED.get(settings)) { //如果elasticsearch.yml文件中配置了http.enabled参数(默认为true),则会初始化RestHandlers

            logger.debug("initializing HTTP handlers ...");

            actionModule.initRestHandlers(() -> clusterService.state().nodes()); //初始化RestHandlers, 解析集群命令,如_cat/,_cat/health

        }

        //10、初始化工作完成

        logger.info("initialized");

        success = true;

    } catch (IOException ex) {

        throw new ElasticsearchException("failed to bind service", ex);

    } finally {

        if (!success) {

            IOUtils.closeWhileHandlingException(resourcesToClose);

        }

    }

}

```
这里再说下上面这么多代码主要干了什么吧
- 创建节点环境,比如节点名称,节点 ID,分片信息,存储元,以及分配内存准备给节点使用
- 打印出 JVM 相关信息
- 利用 PluginsService 加载相应的模块和插件，具体哪些模块可以去 modules 目录下查看
- 加载一些额外的配置参数
- 创建一个节点客户端
- 缓存一系列模块,如NodeModule,ClusterModule,IndicesModule,ActionModule,GatewayModule,SettingsModule,RepositioriesModule，scriptModule，analysisModule
- 获取 RestController，用于处理各种 Elasticsearch 的 rest 命令,如 _cat, _all, _cat/health, _clusters 等 rest命令
- 绑定处理各种服务的实例
- 利用 Guice 将各种模块以及服务(xxxService)注入到 Elasticsearch 环境中
- 初始化工作完成（打印日志）

正式启动 ES 节点

回到上面 Bootstrap 中的静态 init 方法中，接下来就是正式启动 elasticsearch 节点了：
```
INSTANCE.start();  //调用下面的 start 方法
private void start() throws NodeValidationException {
    node.start();                                       //正式启动 Elasticsearch 节点
    keepAliveThread.start();
}
```
接下来看看这个 start 方法里面的 node.start() 方法源码：
```
public Node start() throws NodeValidationException {

    if (!lifecycle.moveToStarted()) {

        return this;

    }

    Logger logger = Loggers.getLogger(Node.class, NODE_NAME_SETTING.get(settings));

    logger.info("starting ...");

    pluginLifecycleComponents.forEach(LifecycleComponent::start);

    //1、利用Guice获取上述注册的各种模块以及服务

    //Node 的启动其实就是 node 里每个组件的启动，同样的，分别调用不同的实例的 start 方法来启动这个组件, 如下：

    injector.getInstance(MappingUpdatedAction.class).setClient(client);

    injector.getInstance(IndicesService.class).start();

    injector.getInstance(IndicesClusterStateService.class).start();

    injector.getInstance(SnapshotsService.class).start();

    injector.getInstance(SnapshotShardsService.class).start();

    injector.getInstance(RoutingService.class).start();

    injector.getInstance(SearchService.class).start();

    nodeService.getMonitorService().start();

    final ClusterService clusterService = injector.getInstance(ClusterService.class);

    final NodeConnectionsService nodeConnectionsService = injector.getInstance(NodeConnectionsService.class);

    nodeConnectionsService.start();

    clusterService.setNodeConnectionsService(nodeConnectionsService);

    injector.getInstance(ResourceWatcherService.class).start();

    injector.getInstance(GatewayService.class).start();

    Discovery discovery = injector.getInstance(Discovery.class);

    clusterService.getMasterService().setClusterStatePublisher(discovery::publish);

    // Start the transport service now so the publish address will be added to the local disco node in ClusterService

    TransportService transportService = injector.getInstance(TransportService.class);

    transportService.getTaskManager().setTaskResultsService(injector.getInstance(TaskResultsService.class));

    transportService.start();

    assert localNodeFactory.getNode() != null;

    assert transportService.getLocalNode().equals(localNodeFactory.getNode())

        : "transportService has a different local node than the factory provided";

    final MetaData onDiskMetadata;

    try {

        // we load the global state here (the persistent part of the cluster state stored on disk) to

        // pass it to the bootstrap checks to allow plugins to enforce certain preconditions based on the recovered state.

        if (DiscoveryNode.isMasterNode(settings) || DiscoveryNode.isDataNode(settings)) {//根据配置文件看当前节点是master还是data节点

            onDiskMetadata = injector.getInstance(GatewayMetaState.class).loadMetaState();

        } else {

            onDiskMetadata = MetaData.EMPTY_META_DATA;

        }

        assert onDiskMetadata != null : "metadata is null but shouldn't"; // this is never null

    } catch (IOException e) {

        throw new UncheckedIOException(e);

    }

    validateNodeBeforeAcceptingRequests(new BootstrapContext(settings, onDiskMetadata), transportService.boundAddress(), pluginsService

        .filterPlugins(Plugin

        .class)

        .stream()

        .flatMap(p -> p.getBootstrapChecks().stream()).collect(Collectors.toList()));

    //2、将当前节点加入到一个集群簇中去,并启动当前节点

    clusterService.addStateApplier(transportService.getTaskManager());

    // start after transport service so the local disco is known

    discovery.start(); // start before cluster service so that it can set initial state on ClusterApplierService

    clusterService.start();

    assert clusterService.localNode().equals(localNodeFactory.getNode())

        : "clusterService has a different local node than the factory provided";

    transportService.acceptIncomingRequests();

    discovery.startInitialJoin();

    // tribe nodes don't have a master so we shouldn't register an observer         s

    final TimeValue initialStateTimeout = DiscoverySettings.INITIAL_STATE_TIMEOUT_SETTING.get(settings);

    if (initialStateTimeout.millis() > 0) {

        final ThreadPool thread = injector.getInstance(ThreadPool.class);

        ClusterState clusterState = clusterService.state();

        ClusterStateObserver observer = new ClusterStateObserver(clusterState, clusterService, null, logger, thread.getThreadContext());

        if (clusterState.nodes().getMasterNodeId() == null) {

            logger.debug("waiting to join the cluster. timeout [{}]", initialStateTimeout);

            final CountDownLatch latch = new CountDownLatch(1);

            observer.waitForNextChange(new ClusterStateObserver.Listener() {

                @Override

                public void onNewClusterState(ClusterState state) { latch.countDown(); }

                @Override

                public void onClusterServiceClose() {

                    latch.countDown();

                }

                @Override

                public void onTimeout(TimeValue timeout) {

                    logger.warn("timed out while waiting for initial discovery state - timeout: {}",

                        initialStateTimeout);

                    latch.countDown();

                }

            }, state -> state.nodes().getMasterNodeId() != null, initialStateTimeout);

            try {

                latch.await();

            } catch (InterruptedException e) {

                throw new ElasticsearchTimeoutException("Interrupted while waiting for initial discovery state");

            }

        }

    }

    if (NetworkModule.HTTP_ENABLED.get(settings)) {

        injector.getInstance(HttpServerTransport.class).start();

    }

    if (WRITE_PORTS_FILE_SETTING.get(settings)) {

        if (NetworkModule.HTTP_ENABLED.get(settings)) {

            HttpServerTransport http = injector.getInstance(HttpServerTransport.class);

            writePortsFile("http", http.boundAddress());

        }

        TransportService transport = injector.getInstance(TransportService.class);

        writePortsFile("transport", transport.boundAddress());

    }

    logger.info("started");

    pluginsService.filterPlugins(ClusterPlugin.class).forEach(ClusterPlugin::onNodeStarted);

    return this;

}

```
# 参考资料
Elasticsearch 源码解析与优化实战





