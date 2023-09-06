---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码命令服务


## 服务-CommandService
CommandService
```
/**
 * Command Scheduler命令的调度器
 * 收集OAP返回的Command,然后分发给不同的处理器去处理
 */
@DefaultImplementor
public class CommandService implements BootService, Runnable {
  
    private ExecutorService executorService = Executors.newSingleThreadExecutor(
        new DefaultNamedThreadFactory("CommandService")
      
    @Override
    public void boot() throws Throwable {
        executorService.submit(
            new RunnableWithExceptionProtection(this, t -> LOGGER.error(t, "CommandService failed to execute commands"))
        );
    }      

```

boot()方法中使用executorService提交任务，Runnable传入的是this，实际上执行的是CommandService的run()方法，代码如下：
```
@DefaultImplementor
public class CommandService implements BootService, Runnable {

    private volatile boolean isRunning = true; // 命令的处理流程是否在运行
    private LinkedBlockingQueue<BaseCommand> commands = new LinkedBlockingQueue<>(64); // 待处理命令列表
    private CommandSerialNumberCache serialNumberCache = new CommandSerialNumberCache(); // 命令的序列号缓存

    /**
     * 不断从命令队列(任务队列)里取出任务,交给执行器去执行
     */
    @Override
    public void run() {
        final CommandExecutorService commandExecutorService = ServiceManager.INSTANCE.findService(CommandExecutorService.class);

        while (isRunning) {
            try {
                BaseCommand command = commands.take();
                // 同一个命令不要重复执行
                if (isCommandExecuted(command)) {
                    continue;
                }

                // 执行command
                commandExecutorService.execute(command);
                serialNumberCache.add(command.getSerialNumber());
            } catch (InterruptedException e) {
                LOGGER.error(e, "Failed to take commands.");
            } catch (CommandExecutionException e) {
                LOGGER.error(e, "Failed to execute command[{}].", e.command().getCommand());
            } catch (Throwable e) {
                LOGGER.error(e, "There is unexpected exception");
            }
        }
    }

    private boolean isCommandExecuted(BaseCommand command) {
        return serialNumberCache.contain(command.getSerialNumber());
    }

```
run()方法中不断从命令队列里取出任务，交给执行器去执行，使用CommandSerialNumberCache作为命令的序列号缓存，执行完命令后在序列号缓存中添加该命令的序列号，保证同一个命令不会重复执行

CommandSerialNumberCache代码如下：
```
/**
 * 命令的序列号缓存,序列号被放到一个队列里面,并且做了容量控制
 */
public class CommandSerialNumberCache {
    private static final int DEFAULT_MAX_CAPACITY = 64;
    private final Deque<String> queue;
    private final int maxCapacity;

    public CommandSerialNumberCache() {
        this(DEFAULT_MAX_CAPACITY);
    }

    public CommandSerialNumberCache(int maxCapacity) {
        queue = new LinkedBlockingDeque<String>(maxCapacity);
        this.maxCapacity = maxCapacity;
    }

    public void add(String number) {
        if (queue.size() >= maxCapacity) {
            queue.pollFirst();
        }

        queue.add(number);
    }

    public boolean contain(String command) {
        return queue.contains(command);
    }
}

```
ServiceManagementClient的run()方法中拿到服务端给的命令，会调用CommandService的receiveCommand()方法去处理
```
                        // 服务端给到的响应交给CommandService去处理
                        final Commands commands = managementServiceBlockingStub.withDeadlineAfter(
                            GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS
                        ).keepAlive(InstancePingPkg.newBuilder()
                                                   .setService(Config.Agent.SERVICE_NAME)
                                                   .setServiceInstance(Config.Agent.INSTANCE_NAME)
                                                   .build());

                        ServiceManager.INSTANCE.findService(CommandService.class).receiveCommand(commands);

```

CommandService的receiveCommand()方法代码如下：
```
@DefaultImplementor
public class CommandService implements BootService, Runnable {

    private LinkedBlockingQueue<BaseCommand> commands = new LinkedBlockingQueue<>(64); // 待处理命令列表

    public void receiveCommand(Commands commands) {
        for (Command command : commands.getCommandsList()) {
            try {
                // 反序列化
                BaseCommand baseCommand = CommandDeserializer.deserialize(command);

                if (isCommandExecuted(baseCommand)) {
                    LOGGER.warn("Command[{}] is executed, ignored", baseCommand.getCommand());
                    continue;
                }

                // 添加到待处理命令列表
                boolean success = this.commands.offer(baseCommand);

                if (!success && LOGGER.isWarnEnable()) {
                    LOGGER.warn(
                        "Command[{}, {}] cannot add to command list. because the command list is full.",
                        baseCommand.getCommand(), baseCommand.getSerialNumber()
                    );
                }
            } catch (UnsupportedCommandException e) {
                if (LOGGER.isWarnEnable()) {
                    LOGGER.warn("Received unsupported command[{}].", e.getCommand().getCommand());
                }
            }
        }
    }

```

CommandDeserializer的deserialize()方法进行反序列化，代码如下：
```
public class CommandDeserializer {

    public static BaseCommand deserialize(final Command command) {
        final String commandName = command.getCommand();
        if (ProfileTaskCommand.NAME.equals(commandName)) { // 性能追踪
            return ProfileTaskCommand.DESERIALIZER.deserialize(command);
        } else if (ConfigurationDiscoveryCommand.NAME.equals(commandName)) { // 配置更改
            return ConfigurationDiscoveryCommand.DESERIALIZER.deserialize(command);
        }
        throw new UnsupportedCommandException(command);
    }

}

```
OAP下发的命令包含两类：

- ProfileTaskCommand：在SkyWalking UI性能剖析功能中，新建任务，会下发给Agent性能追踪任务
- ConfigurationDiscoveryCommand：当前版本SkyWalking Agent支持运行时动态调整配置

ConfigurationDiscoveryCommand反序列化代码如下：
```
public class ConfigurationDiscoveryCommand extends BaseCommand implements Serializable, Deserializable<ConfigurationDiscoveryCommand> {

    public static final String NAME = "ConfigurationDiscoveryCommand";

    public static final String UUID_CONST_NAME = "UUID";
    public static final String SERIAL_NUMBER_CONST_NAME = "SerialNumber";
  
    /*
     * 如果配置没有变,那么OAP返回的UUID就是一样的
     * If config is unchanged, then could response the same uuid, and config is not required.
     */
    private String uuid;
    /*
     * The configuration of service.
     */
    private List<KeyStringValuePair> config;

    public ConfigurationDiscoveryCommand(String serialNumber,
                                         String uuid,
                                         List<KeyStringValuePair> config) {
        super(NAME, serialNumber);
        this.uuid = uuid;
        this.config = config;
    }

    @Override
    public ConfigurationDiscoveryCommand deserialize(Command command) {
        String serialNumber = null;
        String uuid = null;
        List<KeyStringValuePair> config = new ArrayList<>();

        for (final KeyStringValuePair pair : command.getArgsList()) {
            if (SERIAL_NUMBER_CONST_NAME.equals(pair.getKey())) {
                serialNumber = pair.getValue();
            } else if (UUID_CONST_NAME.equals(pair.getKey())) {
                uuid = pair.getValue();
            } else {
                config.add(pair);
            }
        }
        return new ConfigurationDiscoveryCommand(serialNumber, uuid, config);
    }
```

执行command
```
@DefaultImplementor
public class CommandService implements BootService, Runnable {

    @Override
    public void run() {
        final CommandExecutorService commandExecutorService = ServiceManager.INSTANCE.findService(CommandExecutorService.class);

        while (isRunning) {
            try {
                BaseCommand command = commands.take();
                // 同一个命令不要重复执行
                if (isCommandExecuted(command)) {
                    continue;
                }

                // 执行command
                commandExecutorService.execute(command);
                serialNumberCache.add(command.getSerialNumber());
            } catch (InterruptedException e) {
                LOGGER.error(e, "Failed to take commands.");
            } catch (CommandExecutionException e) {
                LOGGER.error(e, "Failed to execute command[{}].", e.command().getCommand());
            } catch (Throwable e) {
                LOGGER.error(e, "There is unexpected exception");
            }
        }
    }

```

CommandService的run()方法中调用CommandExecutorService的execute()方法执行command，CommandExecutorService代码如下：
```
@DefaultImplementor
public class CommandExecutorService implements BootService, CommandExecutor {
    /**
     * key: 命令的名字 value:对应的命令执行器
     */
    private Map<String, CommandExecutor> commandExecutorMap;

    @Override
    public void prepare() throws Throwable {
        commandExecutorMap = new HashMap<String, CommandExecutor>();

        // 性能追踪命令执行器
        // Profile task executor
        commandExecutorMap.put(ProfileTaskCommand.NAME, new ProfileTaskCommandExecutor());

        // 配置变更命令执行器
        //Get ConfigurationDiscoveryCommand executor.
        commandExecutorMap.put(ConfigurationDiscoveryCommand.NAME, new ConfigurationDiscoveryCommandExecutor());
    }

    @Override
    public void boot() throws Throwable {

    }

    @Override
    public void onComplete() throws Throwable {

    }

    @Override
    public void shutdown() throws Throwable {

    }

    @Override
    public void execute(final BaseCommand command) throws CommandExecutionException {
        executorForCommand(command).execute(command);
    }

    private CommandExecutor executorForCommand(final BaseCommand command) {
      	// 根据command类型找到对应的命令执行器
        final CommandExecutor executor = commandExecutorMap.get(command.getCommand());
        if (executor != null) {
            return executor;
        }
        return NoopCommandExecutor.INSTANCE;
    }
}

```
execute()方法就是根据command类型找到对应的命令执行器，再调用对应的execute()实现

接下来来看下配置变更命令执行器的实现，ConfigurationDiscoveryCommandExecutor代码如下：
```
public class ConfigurationDiscoveryCommandExecutor implements CommandExecutor {

    private static final ILog LOGGER = LogManager.getLogger(ConfigurationDiscoveryCommandExecutor.class);

    @Override
    public void execute(BaseCommand command) throws CommandExecutionException {
        try {
            ConfigurationDiscoveryCommand agentDynamicConfigurationCommand = (ConfigurationDiscoveryCommand) command;

            ServiceManager.INSTANCE.findService(ConfigurationDiscoveryService.class)
                                   .handleConfigurationDiscoveryCommand(agentDynamicConfigurationCommand);
        } catch (Exception e) {
            LOGGER.error(e, "Handle ConfigurationDiscoveryCommand error, command:{}", command.toString());
        }
    }
}

```
execute()方法中调用ConfigurationDiscoveryService的handleConfigurationDiscoveryCommand()方法真正来处理配置变更的命令


