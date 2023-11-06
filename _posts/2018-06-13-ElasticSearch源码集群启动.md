---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码集群启动
让我们从启动流程开始，先在宏观上看看整个集群是如何启动的，集群状态如何从Red变成Green，不涉及代码，然后分析其他模块的流程。

## 集群启动
今天我们将以 ES 7.13.0 为基础来走一遍节点的启动过程。节点的启动过程比较长，我们只关注大概的流程，至于具体的细节，大家可以后面再慢慢研究。

集群启动过程指集群完全重启时的启动过程，期间要经历选举主节点、主分片、数据恢复等重要阶段，理解其中原理和细节，对于解决或避免集群维护过程中可能遇到的脑裂、无主、恢复慢、丢数据等问题有重要作用。

## 启动核心类
ES 的启动主要涉及 3 个类：

- Elasticsearch:
其中 Elasticsearch 继承了 EnvironmentAwareCommand，而 EnvironmentAwareCommand 继承了 Command，所以 Elasticsearch 可以解析命令行参数，同时 Elasticsearch 还负责加载配置等工作。

- Bootstrap
Bootstrap 主要负责资源检查、本地资源初始化。

- Node，主要负责启动节点，包括加载各个模块和插件、创建线程池、创建 keepalive 线程等工作。

所以我们今天会分析节点整体的启动流程，最后我们还会从宏观的角度来看看集群的启动流程。

## 集群启动源码
打开 server 模块下的 `Elasticsearch` 类：`org.elasticsearch.bootstrap.Elasticsearch`，运行里面的 main 函数就可以启动 `ElasticSearch` 了

### 命令行参数解析与配置加载
```
    /**
     * Main entry point for starting elasticsearch
     */
    public static void main(final String[] args) throws Exception {
        overrideDnsCachePolicyProperties();
        /*
         * We want the JVM to think there is a security manager installed so that if internal policy decisions that would be based on the
         * presence of a security manager or lack thereof act as if there is a security manager present (e.g., DNS cache policy). This
         * forces such policies to take effect immediately.
         */
        System.setSecurityManager(new SecurityManager() {

            @Override
            public void checkPermission(Permission perm) {
                // grant all permissions so that we can later set the security manager to the one that we want
            }

        });
        LogConfigurator.registerErrorListener();
        final Elasticsearch elasticsearch = new Elasticsearch();
        int status = main(args, elasticsearch, Terminal.DEFAULT);
        if (status != ExitCodes.OK) {
            final String basePath = System.getProperty("es.logs.base_path");
            // It's possible to fail before logging has been configured, in which case there's no point
            // suggesting that the user look in the log file.
            if (basePath != null) {
                Terminal.DEFAULT.errorPrintln(
                    "ERROR: Elasticsearch did not exit normally - check the logs at "
                        + basePath
                        + System.getProperty("file.separator")
                        + System.getProperty("es.logs.cluster_name") + ".log"
                );
            }
            exit(status);
        }
    }
```
ES 的启动入口在 org.elasticsearch.bootstrap.Elasticsearch.java 中的 main 函数。main 主要做了以下几件事：

- 重写 DNS 缓存时间
如果启动时传入 es.networkaddress.cache.ttl 或 es.networkaddress.cache.negative.ttl VM 参数的话，将会在这里被解析。你可以在 IDEA 启动 ES 时加入 -Des.networkaddress.cache.ttl=15 进行调试跟踪这个函数的工作。更多关于 DNS 缓存的信息可以参考官方文档。

- 创建空的 SecurityManager 安全管理器
为了保证 Java 应用程序的安全性，在应用层，Java 为用户提供了一套安全管理机制：SecurityManager 安全管理器。Java 应用可以使用自己的安全管理器在运行阶段检查资源的访问和操作权限，从而保护系统不受恶意的攻击。

在这里，ES 先简单创建了一个安全管理器，并且这个安全管理器是授予了所有权限的，在后续执行中可以为这个安全管理器设置所需要的权限。

- 注册错误日志监听器
通过调用 LogConfigurator.registerErrorListener() 注册错误日志监听器。在此处安装监听器可以记录在一些在加载 log4j 配置前的启动错误信息，如果启动有错误将会停止启动并且显示出来。

- 创建一个 Elasticsearch 对象实例
可以看到，其实 Elasticsearch 类继承了 EnvironmentAwareCommand，而 EnvironmentAwareCommand 继承了 Command，所以 Elasticsearch 可以解析命令行参数，同时 Elasticsearch 还负责加载配置等工作。

调用 Elasticsearch 构造方法时，主要解析了一些 命令行传入的参数，如 V（版本）、d（后台运行）、p（pid文件）、q（退出）等，需要注意的是此处的 beforeMain 函数是啥都没有干的。而在 super（EnvironmentAwareCommand构造函数）里，主要是设置 settingOption，所以命令行的 ES 参数需要以 -Ees.path.home=/es/home 这样子来设定。
```
    // visible for testing
    Elasticsearch() {
        super("Starts Elasticsearch", () -> {}); // we configure logging later so we override the base class from configuring logging
        versionOption = parser.acceptsAll(Arrays.asList("V", "version"),
            "Prints Elasticsearch version information and exits");
        daemonizeOption = parser.acceptsAll(Arrays.asList("d", "daemonize"),
            "Starts Elasticsearch in the background")
            .availableUnless(versionOption);
        pidfileOption = parser.acceptsAll(Arrays.asList("p", "pidfile"),
            "Creates a pid file in the specified path on start")
            .availableUnless(versionOption)
            .withRequiredArg()
            .withValuesConvertedBy(new PathConverter());
        quietOption = parser.acceptsAll(Arrays.asList("q", "quiet"),
            "Turns off standard output/error streams logging in console")
            .availableUnless(versionOption)
            .availableUnless(daemonizeOption);
    }
```
通过跟踪 Elasticsearch.main 可以发现，其最终调用了 Command.main（这个函数太长了，不截图啦），其主要执行内容为以下几步：

注册一个 ShutdownHook
```
    public final int main(String[] args, Terminal terminal) throws Exception {
        if (addShutdownHook()) {

            shutdownHookThread = new Thread(() -> {
                try {
                    this.close();
                } catch (final IOException e) {
                    try (
                        StringWriter sw = new StringWriter();
                        PrintWriter pw = new PrintWriter(sw)) {
                        e.printStackTrace(pw);
                        terminal.errorPrintln(sw.toString());
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
            mainWithoutErrorHandling(args, terminal);
        } catch (OptionException e) {
            // print help to stderr on exceptions
            printHelp(terminal, true);
            terminal.errorPrintln(Terminal.Verbosity.SILENT, "ERROR: " + e.getMessage());
            return ExitCodes.USAGE;
        } catch (UserException e) {
            if (e.exitCode == ExitCodes.USAGE) {
                printHelp(terminal, true);
            }
            if (e.getMessage() != null) {
                terminal.errorPrintln(Terminal.Verbosity.SILENT, "ERROR: " + e.getMessage());
            }
            return e.exitCode;
        }
        return ExitCodes.OK;
    }
```
Command.main 函数首先注册了一个 ShutdownHook，其作用是在系统关闭的时候捕获 IOException 并且进行输出。

这里需要注意，beforeMain 其实是之前 Elasticsearch 构造函数传进来的，是一个空函数，此处啥都没干。

- 解析命令行参数
```
    /**
     * Executes the command, but all errors are thrown.
     */
    void mainWithoutErrorHandling(String[] args, Terminal terminal) throws Exception {
        final OptionSet options = parser.parse(args);

        if (options.has(helpOption)) {
            printHelp(terminal, false);
            return;
        }

        if (options.has(silentOption)) {
            terminal.setVerbosity(Terminal.Verbosity.SILENT);
        } else if (options.has(verboseOption)) {
            terminal.setVerbosity(Terminal.Verbosity.VERBOSE);
        } else {
            terminal.setVerbosity(Terminal.Verbosity.NORMAL);
        }

        execute(terminal, options);
    }

```
在注册完 ShutdownHook 后，调用 Command.mainWithoutErrorHandling 函数进行命令行参数解析，最终这个函数在解析完命令行参数后调用了 EnvironmentAwareCommand.execute 函数。

- 加载多个路径：data、home、logs
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
                        Locale.ROOT,
                        "setting [%s] already set, saw [%s] and [%s]",
                        kvp.key,
                        settings.get(kvp.key),
                        kvp.value);
                throw new UserException(ExitCodes.USAGE, message);
            }
            settings.put(kvp.key, kvp.value);
        }

        putSystemPropertyIfSettingIsMissing(settings, "path.data", "es.path.data");
        putSystemPropertyIfSettingIsMissing(settings, "path.home", "es.path.home");
        putSystemPropertyIfSettingIsMissing(settings, "path.logs", "es.path.logs");

        execute(terminal, options, createEnv(settings));
    }
```
EnvironmentAwareCommand.execute 函数主要将命令行参数解析为 HashMap 的形式，然后确保 es.path.data、es.path.home、es.path.logs 这几个路径设置的存在。最后调用 createEnv 函数加载 elasticsearch.yml 配置文件，再调用 Elasticsearch.execute 函数。

- 加载 elasticsearch.yml 配置文件
createEnv 函数最终调用了 InternalSettingsPreparer.prepareEnvironment 来加载 elasticsearch.yml 配置文件，并且创建了 command 运行的环境：Environment 对象。这部分感兴趣的可以进行 debug 跟踪其实现细节，这里就不展开了。

验证配置
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
                Build.CURRENT.getQualifiedVersion(),
                Build.CURRENT.flavor().displayName(),
                Build.CURRENT.type().displayName(),
                Build.CURRENT.hash(),
                Build.CURRENT.date(),
                JvmInfo.jvmInfo().version()
            );
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
            init(daemonize, pidFile, quiet, env);
        } catch (NodeValidationException e) {
            throw new UserException(ExitCodes.CONFIG, e.getMessage());
        }
    }
```
EnvironmentAwareCommand.execute 函数最终调用了 Elasticsearch.execute 函数，Elasticsearch.execute 主要做配置验证的操作，并且调用 init 函数进入到 Bootstrap.init，也就进入到第二阶段了。

所以总的来说，启动的第一阶段做了命令行参数解析与配置加载验证的工作。

## 资源检查与本地资源初始化
阶段二主要在 Bootstrap 类中进行，我们进入到 Bootstrap.init 中进行跟踪，其详细流程如下：

创建 Bootstrap 实例 通过 INSTANCE = new Bootstrap() 创建了 Bootstrap 实例：
```
/**
 * Internal startup code.
 */
final class Bootstrap {

    private static volatile Bootstrap INSTANCE;
    private volatile Node node;
    private final CountDownLatch keepAliveLatch = new CountDownLatch(1);
    private final Thread keepAliveThread;
    private final Spawner spawner = new Spawner();

    /** creates a new instance */
    Bootstrap() {
        keepAliveThread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    keepAliveLatch.await();
                } catch (InterruptedException e) {
                    // bail out
                }
            }
        }, "elasticsearch[keepAlive/" + Version.CURRENT + "]");
        keepAliveThread.setDaemon(false);
        // keep this thread alive (non daemon thread) until we shutdown
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                keepAliveLatch.countDown();
            }
        });
    }

```
如上代码，Bootstrap 实例是一个单例，但其又不是严格按照我们常用的那几种创建单例的模式实现的，这里的实现简单粗暴，因为外面只调用了一次。

Bootstrap 构造函数创建了 keepAlive 线程，并且将这个线程设置为非守护线程，因为 JVM 中必须存在一个非守护线程，否则 JVM 进程会退出。

这个 keepAlive 线程啥都没干，只执行了 keepAliveLatch.await()，然后就是一直等待 countDown 操作。CountDownLatch 在 Java 里是一个线程同步工具，keepAliveLatch 是一个 CountDownLatch 实例，并且 count 为 1。所以 keepAliveLatch.await() 的作用是一直等待直到有一个线程执行了一次 keepAliveLatch.countDown() 操作。

最后通过 addShutdownHook，在系统关闭的时候执行 keepAliveLatch.countDown() 操作。

加载 elasticsearch.keystore 安全配置 细心的你会发现，在运行了 ES 后在，在 config 目录会生成一个 elasticsearch.keystore 文件，这个文件是用来保存一些敏感配置的。因为 ES 大多数配置都是明文保存的，但是像 X-Pack 中的 security 配置需要进行加密保存，所以这些配置信息就是保存在 elasticsearch.keystore 中。
```
    static SecureSettings loadSecureSettings(Environment initialEnv, InputStream stdin) throws BootstrapException {
        final KeyStoreWrapper keystore;
        try {
            keystore = KeyStoreWrapper.load(initialEnv.configFile());
        } catch (IOException e) {
            throw new BootstrapException(e);
        }

        SecureString password;
        try {
            if (keystore != null && keystore.hasPassword()) {
                password = readPassphrase(stdin, KeyStoreAwareCommand.MAX_PASSPHRASE_LENGTH);
            } else {
                password = new SecureString(new char[0]);
            }
        } catch (IOException e) {
            throw new BootstrapException(e);
        }

        try{
            if (keystore == null) {
                final KeyStoreWrapper keyStoreWrapper = KeyStoreWrapper.create();
                keyStoreWrapper.save(initialEnv.configFile(), new char[0]);
                return keyStoreWrapper;
            } else {
                keystore.decrypt(password.getChars());
                KeyStoreWrapper.upgrade(keystore, initialEnv.configFile(), password.getChars());
            }
        } catch (Exception e) {
            throw new BootstrapException(e);
        } finally {
            password.close();
        }
        return keystore;
    }

```
如上代码，在 loadSecureSettings 函数中进行加载 elasticsearch.keystore 中的安全配置，如果 elasticsearch.keystore 不存在，则进行创建并且保存相关信息，如果 elasticsearch.keystore 存在，则更新配置信息。

- 创建一个新的 Environment
```
    private static Environment createEnvironment(
            final Path pidFile,
            final SecureSettings secureSettings,
            final Settings initialSettings,
            final Path configPath) {
        Settings.Builder builder = Settings.builder();
        if (pidFile != null) {
            builder.put(Environment.NODE_PIDFILE_SETTING.getKey(), pidFile);
        }
        builder.put(initialSettings);
        if (secureSettings != null) {
            builder.setSecureSettings(secureSettings);
        }
        return InternalSettingsPreparer.prepareEnvironment(builder.build(), Collections.emptyMap(), configPath,
                // HOSTNAME is set by elasticsearch-env and elasticsearch-env.bat so it is always available
                () -> System.getenv("HOSTNAME"));
    }
```

- 设置节点名称
```
    public static void setNodeName(String nodeName) {
        NodeNamePatternConverter.setNodeName(nodeName);
    }
```
如上代码，调用 LogConfigurator.setNodeName 设置节点的名字，这里设置节点的名字，可以后续的日志输出中使用，否则只要节点 ID 可用就会使用节点 ID（节点 ID 可读性不好！）

- 加载 log4j2 配置
```
    public static void configure(final Environment environment) throws IOException, UserException {
        Objects.requireNonNull(environment);
        try {
            // we are about to configure logging, check that the status logger did not log any error-level messages
            checkErrorListener();
        } finally {
            // whether or not the error listener check failed we can remove the listener now
            StatusLogger.getLogger().removeListener(ERROR_LISTENER);
        }
        configure(environment.settings(), environment.configFile(), environment.logsFile());
    }
```
如上代码，调用 LogConfigurator.configure 加载 log4j2.properties 文件中的相关配置，然后配置 log4j 的属性。注意这里的 checkErrorListener()，不知道你是否还记得 Elasticsearch.main 中注册的错误日志监听器，在这里检查这个日志监听器是否有记录错误等级的日志。

- 创建 pid 文件
```
        if (environment.pidFile() != null) {
            try {
                PidFile.create(environment.pidFile(), true);
            } catch (IOException e) {
                throw new BootstrapException(e);
            }
        }
```
如上代码，创建 pid 文件（如果存在先进行删除），并且将进程的 pid 写入其中。

检查 Lucene jar checkLucene() 函数通过版本号来检查 lucene 是否被替换了，如果 lucene 被替换将无法启动。

- 安装未捕获异常的处理程序
```
            // install the default uncaught exception handler; must be done before security is
            // initialized as we do not want to grant the runtime permission
            // setDefaultUncaughtExceptionHandler
            Thread.setDefaultUncaughtExceptionHandler(new ElasticsearchUncaughtExceptionHandler());

            INSTANCE.setup(true, environment);
```
如上代码，通过 Thread.setDefaultUncaughtExceptionHandler 设置了一个 ElasticsearchUncaughtExceptionHandler 未捕获异常处理程序。Thread.UncaughtExceptionHandler 是当线程由于未捕获的异常而突然终止时调用的处理程序接口。在多线程的环境下，主线程无法捕捉其他线程产生的异常，这时需要通过实现 UncaughtExceptionHandler 来捕获其他线程产生但又未被捕获的异常。

为创建 Node 对象实例做准备工作 通过调用 INSTANCE.setup(true, environment) 为创建 Node 对象实例做一些准备工作，下面几步我们进入到 INSTANCE.setup 中看看其实现。

为给定模块生成控制器守护程序 在 INSTANCE.setup 里调用了 spawner.spawnNativeControllers，通过跟踪，其实现如下：
```
    void spawnNativeControllers(final Environment environment, final boolean inheritIo) throws IOException {
        if (spawned.compareAndSet(false, true) == false) {
            throw new IllegalStateException("native controllers already spawned");
        }
        if (Files.exists(environment.modulesFile()) == false) {
            throw new IllegalStateException("modules directory [" + environment.modulesFile() + "] not found");
        }
        /*
         * For each module, attempt to spawn the controller daemon. Silently ignore any module that doesn't include a controller for the
         * correct platform.
         */
        List<Path> paths = PluginsService.findPluginDirs(environment.modulesFile());
        for (final Path modules : paths) {
            final PluginInfo info = PluginInfo.readFromProperties(modules);
            final Path spawnPath = Platforms.nativeControllerPath(modules);
            if (Files.isRegularFile(spawnPath) == false) {
                continue;
            }
            if (info.hasNativeController() == false) {
                final String message = String.format(
                    Locale.ROOT,
                    "module [%s] does not have permission to fork native controller",
                    modules.getFileName());
                throw new IllegalArgumentException(message);
            }
            final Process process = spawnNativeController(spawnPath, environment.tmpFile(), inheritIo);
            processes.add(process);
        }
    }
```
如上代码，通过注释可以知道，spawnNativeControllers 的作用主要是尝试为每个模块（modules 目录下的模块）生成 native 控制器守护进程的。生成的进程将通过其 stdin、stdout 和 stderr 流保持与此 JVM 的连接，这个进程不应该写入任何数据到其 stdout 和 stderr，否则如果没有其他线程读取这些 output 数据的话，这个进程将会被阻塞，为了避免这种情况发生，可以继承 JVM 的 stdout 和 stderr（在标准安装中，它们会被重定向到文件）。此处的实现如下：
```
        // The process _shouldn't_ write any output via its stdout or stderr, but if it does then
        // it will block if nothing is reading that output. To avoid this we can inherit the
        // JVM's stdout and stderr (which are redirected to files in standard installations).
        if (inheritIo) {
            pb.redirectOutput(ProcessBuilder.Redirect.INHERIT);
            pb.redirectError(ProcessBuilder.Redirect.INHERIT);
        }
```
怎么说呢，这部分要明白的话基础知识要非常扎实才行，这里涉及到很多进程、线程、子进程、守护进程、进程标准输入输出等操作系统相关的知识。遗憾的是这里无法展开了，感兴趣的可以自行了解。