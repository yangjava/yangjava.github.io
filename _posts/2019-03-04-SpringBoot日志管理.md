---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot日志管理
会当凌绝顶，一览众山小。——杜甫《望岳》
在学习这块时需要一些日志框架的发展和基础，同时了解日志配置时考虑的因素。

## 配置时考虑点

在配置日志时需要考虑哪些因素？
- 支持日志路径，日志level等配置 
- 日志控制配置通过application.yml下发 
- 按天生成日志，当天的日志>50MB回滚 
- 最多保存10天日志 生成的日志中Pattern自定义 
- Pattern中添加用户自定义的MDC字段，比如用户信息(当前日志是由哪个用户的请求产生)，request信息。此种方式可以通过AOP切面控制，在MDC中添加requestID，在spring-logback.xml中配置Pattern。 
- 根据不同的运行环境设置Profile - dev，test，product 
- 对控制台，Err和全量日志分别配置 
- 对第三方包路径日志控制


## SpringBoot之logging
Spring Boot默认的日志组件为Logback。
日志配置参数
```yaml
logging:
    file:   # 日志文件,绝对路径或相对路径
    path:   # 保存日志文件目录路径
    config: # 日志配置文件,Spring Boot默认使用classpath路径下的日志配置文件,如:logback.xml
    level:  # 日志级别
        org.springframework.web: DEBUG # 配置spring web日志级别
```

更改Spring Boot日志组件为Log4j(注：Spring Boot仅仅支持Log4j 2.x版本):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

关于Spring Boot日志文件路径的疑惑?
同时配置了logging.path和logging.file属性，如下配置：
```yaml
logging:
    path: /var/log
    file: test.log
```
仅仅只会在项目根路径下产生test.log文件，不会在指定路径下产生日志文件(期望日志路径为：logging.path + logging.file)。

原因：Spring Boot中的logging.path和logging.file这2个属性，只需要配置其中之一即可，如果同时配置，则使用logging.file属性。

当配置了loggin.path属性时，将在该路径下生成spring.log文件，即：此时使用默认的日志文件名spring.log

当配置了loggin.file属性时，将在指定路径下生成指定名称的日志文件。默认为项目相对路径，可以为logging.file指定绝对路径。

```yaml
logging: 
    path: /var/logs          # 在/var/logs目录下生成spring.log文件
    file: /var/logs/test.log # 在/var/logs目录下生成test.log文件
```

Spring-logback.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- 日志根目录-->
    <springProperty scope="context" name="LOG_HOME" source="logging.path" defaultValue="/data/logs/springboot-logback-demo"/>

    <!-- 日志级别 -->
    <springProperty scope="context" name="LOG_ROOT_LEVEL" source="logging.level.root" defaultValue="DEBUG"/>

    <!--  标识这个"STDOUT" 将会添加到这个logger -->
    <springProperty scope="context" name="STDOUT" source="log.stdout" defaultValue="STDOUT"/>

    <!-- 日志文件名称-->
    <property name="LOG_PREFIX" value="spring-boot-logback" />

    <!-- 日志文件编码-->
    <property name="LOG_CHARSET" value="UTF-8" />

    <!-- 日志文件路径+日期-->
    <property name="LOG_DIR" value="${LOG_HOME}/%d{yyyyMMdd}" />

    <!--对日志进行格式化-->
    <property name="LOG_MSG" value="- | [%X{requestUUID}] | [%d{yyyyMMdd HH:mm:ss.SSS}] | [%level] | [${HOSTNAME}] | [%thread] | [%logger{36}] | --> %msg|%n "/>

    <!--文件大小，默认10MB-->
    <property name="MAX_FILE_SIZE" value="50MB" />

    <!-- 配置日志的滚动时间 ，表示只保留最近 10 天的日志-->
    <property name="MAX_HISTORY" value="10"/>

    <!--输出到控制台-->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 输出的日志内容格式化-->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>${LOG_MSG}</pattern>
        </layout>
    </appender>

    <!--输出到文件-->
    <appender name="0" class="ch.qos.logback.core.rolling.RollingFileAppender">
    </appender>

    <!-- 定义 ALL 日志的输出方式:-->
    <appender name="FILE_ALL" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--日志文件路径，日志文件名称-->
        <File>${LOG_HOME}/all_${LOG_PREFIX}.log</File>

        <!-- 设置滚动策略，当天的日志大小超过 ${MAX_FILE_SIZE} 文件大小时候，新的内容写入新的文件， 默认10MB -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">

            <!--日志文件路径，新的 ALL 日志文件名称，“ i ” 是个变量 -->
            <FileNamePattern>${LOG_DIR}/all_${LOG_PREFIX}%i.log</FileNamePattern>

            <!-- 配置日志的滚动时间 ，表示只保留最近 10 天的日志-->
            <MaxHistory>${MAX_HISTORY}</MaxHistory>

            <!--当天的日志大小超过 ${MAX_FILE_SIZE} 文件大小时候，新的内容写入新的文件， 默认10MB-->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>${MAX_FILE_SIZE}</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>

        </rollingPolicy>

        <!-- 输出的日志内容格式化-->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>${LOG_MSG}</pattern>
        </layout>
    </appender>

    <!-- 定义 ERROR 日志的输出方式:-->
    <appender name="FILE_ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 下面为配置只输出error级别的日志 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <OnMismatch>DENY</OnMismatch>
            <OnMatch>ACCEPT</OnMatch>
        </filter>
        <!--日志文件路径，日志文件名称-->
        <File>${LOG_HOME}/err_${LOG_PREFIX}.log</File>

        <!-- 设置滚动策略，当天的日志大小超过 ${MAX_FILE_SIZE} 文件大小时候，新的内容写入新的文件， 默认10MB -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">

            <!--日志文件路径，新的 ERR 日志文件名称，“ i ” 是个变量 -->
            <FileNamePattern>${LOG_DIR}/err_${LOG_PREFIX}%i.log</FileNamePattern>

            <!-- 配置日志的滚动时间 ，表示只保留最近 10 天的日志-->
            <MaxHistory>${MAX_HISTORY}</MaxHistory>

            <!--当天的日志大小超过 ${MAX_FILE_SIZE} 文件大小时候，新的内容写入新的文件， 默认10MB-->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>${MAX_FILE_SIZE}</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>

        <!-- 输出的日志内容格式化-->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <Pattern>${LOG_MSG}</Pattern>
        </layout>
    </appender>

    <!-- additivity 设为false,则logger内容不附加至root ，配置以配置包下的所有类的日志的打印，级别是 ERROR-->
    <logger name="org.springframework"     level="ERROR" />
    <logger name="org.apache.commons"      level="ERROR" />
    <logger name="org.apache.zookeeper"    level="ERROR"  />
    <logger name="com.alibaba.dubbo.monitor" level="ERROR"/>
    <logger name="com.alibaba.dubbo.remoting" level="ERROR" />

    <!-- ${LOG_ROOT_LEVEL} 日志级别 -->
    <root level="${LOG_ROOT_LEVEL}">

        <!-- 标识这个"${STDOUT}"将会添加到这个logger -->
        <appender-ref ref="${STDOUT}"/>

        <!-- FILE_ALL 日志输出添加到 logger -->
        <appender-ref ref="FILE_ALL"/>

        <!-- FILE_ERROR 日志输出添加到 logger -->
        <appender-ref ref="FILE_ERROR"/>
    </root>

</configuration>
```

Profile 相关的配置可以参考:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml" />
    
     <!-- roll by day -->
     <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">   
    	<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">   
      		<fileNamePattern>logs/springboot-logback-demo.%d{yyyy-MM-dd}.log</fileNamePattern>   
      		<maxHistory>30</maxHistory>  
    	</rollingPolicy>   
    	<encoder>   
      		<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{35} - %msg%n</pattern>   
    	</encoder>  
  	</appender> 
   
    <!-- dev -->
	<logger name="org.springframework.web" level="INFO"/>
		<root level="INFO">
		<appender-ref ref="FILE" />
	</root>

    <!-- test or production -->
    <springProfile name="test,prod">
        <logger name="org.springframework.web" level="INFO"/>
        <logger name="com.pdai.springboot" level="INFO"/>
        <root level="INFO">
        	<appender-ref ref="FILE" />
        </root>
    </springProfile>
  	 
</configuration>
```

## springboot日志源码解读
还记得springboot项目里我们只要在pom.xml引入spring-boot-starter-parent，父依赖也依赖于logging，所以在启动springboot时候，会自动打印info以上级别的日志，说明springboot已经办我们加载好了日志方面的配置等。

## 分析
如下的Maven依赖将导致以依赖的形式引入 spring-boot-starter-logging-xxx.jar， 该JAR里面就一个 META-INF/spring.provides文件，内容是provides: logback-classic,jcl-over-slf4j,jul-to-slf4j——用以确保这些JAR的引入。
```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
  </dependency>
```
默认情况下，SpringBoot是提供了三种实现， JDK，logback，log4j2。因为笔者比较习惯log4j2，所以本次我们关注的中心主要在于SpringBoot默认的logback和log4j2上。
首先，在spring-boot-xxx.jar 的 META-INF/spring.factories文件中有如下一行：在jar里找到springboot-》META_INF-》spring.factories
可以发现LoggingApplicationListener这个类就是帮我们加载日志方面的的罪魁祸首。从名字不难猜到其实他就是监听了SpringbootApplication的启动，然后触发的。下面我们来看看LoggingApplicationListener

```java
public class LoggingApplicationListener implements GenericApplicationListener {

	private static final ConfigurationPropertyName LOGGING_LEVEL = ConfigurationPropertyName.of("logging.level");//默认日志级别level
	public static final String LOGGING_SYSTEM_BEAN_NAME = "springBootLoggingSystem";
private static final Map<String, List<String>> DEFAULT_GROUP_LOGGERS;
	static {//静态代码块表示可以配置输出的日志范围
		MultiValueMap<String, String> loggers = new LinkedMultiValueMap<>();
		loggers.add("web", "org.springframework.core.codec");
		loggers.add("web", "org.springframework.http");
		loggers.add("web", "org.springframework.web");
		loggers.add("web", "org.springframework.boot.actuate.endpoint.web");
		loggers.add("web", "org.springframework.boot.web.servlet.ServletContextInitializerBeans");
		loggers.add("sql", "org.springframework.jdbc.core");
		loggers.add("sql", "org.hibernate.SQL");
		loggers.add("sql", "org.jooq.tools.LoggerListener");
		DEFAULT_GROUP_LOGGERS = Collections.unmodifiableMap(loggers);
	}
	}
//进行一系列初始化环境加载操作，也就是把配置文件里配置的东西初始化
	protected void initialize(ConfigurableEnvironment environment, ClassLoader classLoader) {
		getLoggingSystemProperties(environment).apply();
		this.logFile = LogFile.get(environment);
		if (this.logFile != null) {
			this.logFile.applyToSystemProperties();
		}
		this.loggerGroups = new LoggerGroups(DEFAULT_GROUP_LOGGERS);
		initializeEarlyLoggingLevel(environment);
		initializeSystem(environment, this.loggingSystem, this.logFile);
		initializeFinalLoggingLevels(environment, this.loggingSystem);
		registerShutdownHookIfNecessary(environment, this.loggingSystem);
	}
	private void initializeSystem(ConfigurableEnvironment environment, LoggingSystem system, LogFile logFile) {
		String logConfig = StringUtils.trimWhitespace(environment.getProperty(CONFIG_PROPERTY));
		try {
			LoggingInitializationContext initializationContext = new LoggingInitializationContext(environment);
			if (ignoreLogConfig(logConfig)) {
				system.initialize(initializationContext, null, logFile);
			}
			else {
				system.initialize(initializationContext, logConfig, logFile);
			}
		}
		catch (Exception ex) {
			Throwable exceptionToReport = ex;
			while (exceptionToReport != null && !(exceptionToReport instanceof FileNotFoundException)) {
				exceptionToReport = exceptionToReport.getCause();
			}
			exceptionToReport = (exceptionToReport != null) ? exceptionToReport : ex;
			// NOTE: We can't use the logger here to report the problem
			System.err.println("Logging system failed to initialize using configuration from '" + logConfig + "'");
			exceptionToReport.printStackTrace(System.err);
			throw new IllegalStateException(ex);
		}
	}

	private boolean ignoreLogConfig(String logConfig) {
		return !StringUtils.hasLength(logConfig) || logConfig.startsWith("-D");
	}
//初始化日志级别
	private void initializeFinalLoggingLevels(ConfigurableEnvironment environment, LoggingSystem system) {
		bindLoggerGroups(environment);
		if (this.springBootLogging != null) {
			initializeSpringBootLogging(system, this.springBootLogging);
		}
		setLogLevels(system, environment);
	}

	private void bindLoggerGroups(ConfigurableEnvironment environment) {
		if (this.loggerGroups != null) {
			Binder binder = Binder.get(environment);
			binder.bind(LOGGING_GROUP, STRING_STRINGS_MAP).ifBound(this.loggerGroups::putAll);
		}
	}
	protected void initializeSpringBootLogging(LoggingSystem system, LogLevel springBootLogging) {
		BiConsumer<String, LogLevel> configurer = getLogLevelConfigurer(system);
		SPRING_BOOT_LOGGING_LOGGERS.getOrDefault(springBootLogging, Collections.emptyList())
				.forEach((name) -> configureLogLevel(name, springBootLogging, configurer));
	}

	/**
	 * Set logging levels based on relevant {@link Environment} properties.
	 * @param system the logging system
	 * @param environment the environment
	 */
	protected void setLogLevels(LoggingSystem system, ConfigurableEnvironment environment) {
		BiConsumer<String, LogLevel> customizer = getLogLevelConfigurer(system);
		Binder binder = Binder.get(environment);
		Map<String, LogLevel> levels = binder.bind(LOGGING_LEVEL, STRING_LOGLEVEL_MAP).orElseGet(Collections::emptyMap);
		levels.forEach((name, level) -> configureLogLevel(name, level, customizer));
	}

	private void configureLogLevel(String name, LogLevel level, BiConsumer<String, LogLevel> configurer) {
		if (this.loggerGroups != null) {
			LoggerGroup group = this.loggerGroups.get(name);
			if (group != null && group.hasMembers()) {
				group.configureLogLevel(level, configurer);
				return;
			}
		}
		configurer.accept(name, level);
	}

	private BiConsumer<String, LogLevel> getLogLevelConfigurer(LoggingSystem system) {
		return (name, level) -> {
			try {
				name = name.equalsIgnoreCase(LoggingSystem.ROOT_LOGGER_NAME) ? null : name;
				system.setLogLevel(name, level);
			}
			catch (RuntimeException ex) {
				this.logger.error(LogMessage.format("Cannot set level '%s' for '%s'", level, name));
			}
		};
	}

	private void registerShutdownHookIfNecessary(Environment environment, LoggingSystem loggingSystem) {
		boolean registerShutdownHook = environment.getProperty(REGISTER_SHUTDOWN_HOOK_PROPERTY, Boolean.class, false);
		if (registerShutdownHook) {
			Runnable shutdownHandler = loggingSystem.getShutdownHandler();
			if (shutdownHandler != null && shutdownHookRegistered.compareAndSet(false, true)) {
				registerShutdownHook(new Thread(shutdownHandler));
			}
		}
	}

	void registerShutdownHook(Thread shutdownHook) {
		Runtime.getRuntime().addShutdownHook(shutdownHook);
	}

	public void setOrder(int order) {
		this.order = order;
	}

	@Override
	public int getOrder() {
		return this.order;
	}

	/**
	 * Sets a custom logging level to be used for Spring Boot and related libraries.
	 * @param springBootLogging the logging level
	 */
	public void setSpringBootLogging(LogLevel springBootLogging) {
		this.springBootLogging = springBootLogging;
	}

	/**
	 * Sets if initialization arguments should be parsed for {@literal debug} and
	 * {@literal trace} properties (usually defined from {@literal --debug} or
	 * {@literal --trace} command line args). Defaults to {@code true}.
	 * @param parseArgs if arguments should be parsed
	 */
	public void setParseArgs(boolean parseArgs) {
		this.parseArgs = parseArgs;
	}

}
```
接下来说说LoggingApplicationListener类里面的重要属性
### LoggingSystem
LoggingSystem： 作用是根据引入的日志框架加载对应的日志文件 顶级基类，名字就能看出是日志系统，SpringBoot用来隔离外界变化用的抽象层
```java
private static final LoggingSystemFactory SYSTEM_FACTORY = LoggingSystemFactory.fromSpringFactories();

/**
	 * Detect and return the logging system in use. Supports Logback and Java Logging.
	 * @param classLoader the classloader
	 * @return the logging system
	 */
	public static LoggingSystem get(ClassLoader classLoader) {
        //SYSTEMS中存放了SpringBoot中Logback，Log4j2，Java Util Logging几个日志框架适配器的路径，检查这些类适配器是否存在以判断这些日志框架是否引入
		String loggingSystemClassName = System.getProperty(SYSTEM_PROPERTY);
		if (StringUtils.hasLength(loggingSystemClassName)) {
			if (NONE.equals(loggingSystemClassName)) {
				return new NoOpLoggingSystem();
			}
			return get(classLoader, loggingSystemClassName);
		}
		LoggingSystem loggingSystem = SYSTEM_FACTORY.getLoggingSystem(classLoader);
		Assert.state(loggingSystem != null, "No suitable logging system located");
		return loggingSystem;
	}

```

```text
# Logging Systems
org.springframework.boot.logging.LoggingSystemFactory=\
org.springframework.boot.logging.logback.LogbackLoggingSystem.Factory,\
org.springframework.boot.logging.log4j2.Log4J2LoggingSystem.Factory,\
org.springframework.boot.logging.java.JavaLoggingSystem.Factory
```

可以看到SpringBoot采用了基于SLF4J的方式来统一log4j2和logback。默认提供了JDK，logback和log4j2三种日志实现方式。

## SpringBoot常用配置
SpringBoot 默认日志框架（SLF4J和logback）的基础配置。配置文件application.properties中
```properties
#配置全局输出级别 off<trace<debug<info<warn<error<fatal
logging.level.root=info
#配置包的输出级别
logging.level.com.demo=debug

# Appender pattern for output to the console. Supported only with the default Logback setup.
#有颜色输出
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} [%p] %highlight(%C:%L) : %m%n
#没有颜色的
#logging.pattern.console=%d %-5p %c:%line %m%n

# Appender pattern for output to a file. Supported only with the default Logback setup.
#配置日志文件格式
logging.pattern.file=%boldMagenta(%d{yyyy-MM-dd HH:mm:ss}) [%p] %highlight(%C:%L) : %m%n
#设置日志控制台编码
logging.charset.console=UTF-8

# 日志文件最大大小
logging.logback.rollingpolicy.max-file-size=10MB
# 设置日志目录
logging.file.path=logs/default/
logging.file.name=${logging.file.path}mylog.log
# 只支持默认的Logback设置，不能配合其他自定义日志项使用
logging.logback.rollingpolicy.file-name-pattern=${logging.file.name}.%d{yyyy-MMdd}.%i.gz
#是否开启定期清理 默认false
logging.logback.rollingpolicy.clean-history-on-start=true
#日志最多存储时间7天
logging.logback.rollingpolicy.max-history=7

# LOGGING
# Location of the logging configuration file. For instance, `classpath:logback.xml` for Logback.
logging.config= 
# Conversion word used when logging exceptions.
logging.exception-conversion-word=%wEx 
# Log file name (for instance, `myapp.log`). Names can be an exact location or relative to the current directory.
logging.file= 
# Maximum of archive log files to keep. Only supported with the default logback setup.
logging.file.max-history=0 
# Maximum log file size. Only supported with the default logback setup.
logging.file.max-size=10MB 
# Log levels severity mapping. For instance, `logging.level.org.springframework=DEBUG`.
logging.level.*= 
# Location of the log file. For instance, `/var/log`.
logging.path= 

logging.pattern.console=
# Appender pattern for log date format. Supported only with the default Logback setup. 
logging.pattern.dateformat=yyyy-MM-dd HH:mm:ss.SSS 

# Appender pattern for log level. Supported only with the default Logback setup.
logging.pattern.level=%5p 
# Register a shutdown hook for the logging system when it is initialized.
logging.register-shutdown-hook=false 

# 控制框架中的日志级别
logging.level.org.springframework=INFO
logging.level.sun=WARN

```

## 高级配置
SpringBoot默认扫描类路径下的logback-spring.xml对日志配置进行自定义的处理。
在resources下添加 logback-spring.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 日志级别从低到高分为TRACE < DEBUG < INFO < WARN < ERROR < FATAL，
如果设置为WARN，则低于WARN的信息都不会输出 -->
<!-- scan:当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true -->
<!-- scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当
scan为true时，此属性生效。默认的时间间隔为1分钟。 -->
<!-- ！！！必须修改 target 内部的才能生效，修改 resources的 logback.xml 不生效。-->
<!-- debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认
值为false。 -->
<configuration scan="true" scanPeriod="5 seconds" debug="false"><contextName>logback</contextName>
    <!-- name的值是变量的名称，value的值时变量定义的值。通过定义的值会被插入到logger上下文
    中。定义变量后，可以使“${}”来使用变量。 -->
    <property name="log.path" value="logs/logback-spring/"/>
    <!-- 彩色日志 -->
    <!-- 配置格式变量：CONSOLE_LOG_PATTERN 彩色日志格式 -->
    <!-- 颜色定义在 ch.qos.logback.core.pattern.color 包下
    black 黑色
    blue 蓝色
    boldBlue 蓝色加粗
    boldCyan 青色加粗
    boldGreen 绿色加粗
    boldMagenta 洋红加粗
    boldRed 红色加粗
    boldWhite 白色加粗
    boldYellow 黄色加粗
    cyan 青色
    gray 灰色
    green 绿色
    magenta 洋红
    red 红色
    white 白色
    yellow 黄色
    -->
    <property name="CONSOLE_LOG_PATTERN"
              value="%black(%date{yyyy-MM-dd HH:mm:ss}) [%blue(%thread)]
[%highlight(%-5level)] %blue(%file:%line) | %green(%logger) | %cyan(%msg%n)"/>
    <!--输出到控制台-->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <!--此日志appender是为开发使用，只配置最底级别，控制台输出的日志级别是大于或等于此级
        别的日志信息-->
        <!-- 例如：如果此处配置了INFO级别，则后面其他位置即使配置了DEBUG级别的日志，也不会被
        输出 -->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>INFO</level>
        </filter>
        <encoder>
            <Pattern>${CONSOLE_LOG_PATTERN}</Pattern>
            <!-- 设置字符集 -->
            <charset>UTF-8</charset>
        </encoder>
    </appender>
    <!--输出到文件-->
    <!-- 时间滚动输出 level为 INFO 日志 -->
    <appender name="INFO_FILE"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${log.path}/log_info.log</file>
        <!--日志文件输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50}
                - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy
                class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 每天日志归档路径以及格式 -->
            <fileNamePattern>${log.path}/info/log-info-%d{yyyy-MMdd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy
                    class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文件只记录info级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
    <!-- 时间滚动输出 level为 WARN 日志 -->
    <appender name="WARN_FILE"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${log.path}/log_warn.log</file>
        <!--日志文件输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50}
                - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy
                class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/warn/log-warn-%d{yyyy-MMdd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy
                    class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文件只记录warn级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>warn</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
    <!-- 时间滚动输出 level为 ERROR 日志 -->
    <appender name="ERROR_FILE"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${log.path}/log_error.log</file>
        <!--日志文件输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50}
                - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy
                class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/error/log-error-%d{yyyy-MMdd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy
                    class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文件只记录ERROR级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
    <!--
    <logger>用来设置某一个包或者具体的某一个类的日志打印级别、以及指定<appender>。
    <logger>仅有一个name属性，
    一个可选的level和一个可选的addtivity属性。
    name:用来指定受此logger约束的某一个包或者具体的某一个类。
    level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和
    OFF，
    如果未设置此属性，那么当前logger将会继承上级的级别。
    -->
    <!--
    使用mybatis的时候，sql语句是debug下才会打印，而这里我们只配置了info，所以想要查看
    sql语句的话，有以下两种操作：
    第一种把<root level="INFO">改成<root level="DEBUG">这样就会打印sql，不过这样日
    志那边会出现很多其他消息
    第二种就是单独给mapper下目录配置DEBUG模式，代码如下，这样配置sql语句会打印，其他还是
    正常INFO级别：
    -->
    <!--开发环境:打印控制台-->
    <springProfile name="dev">
        <!--可以输出项目中的debug日志，包括mybatis的sql日志-->
        <!-- <logger name="com.alen" level="INFO" />-->
        <!--
        root节点是必选节点，用来指定最基础的日志输出级别，只有一个level属性
        level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL
        和 OFF，默认是DEBUG
        可以包含零个或多个appender元素。
        -->
        <!--设置 输出级别 -->
        <root level="INFO">
            <!--下面设置输出到控制台和文件-->
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="INFO_FILE"/>
            <appender-ref ref="WARN_FILE"/><appender-ref ref="ERROR_FILE"/>
        </root>
    </springProfile>
    <!--生产环境:输出到文件-->
    <springProfile name="pro">
        <!-- 设置 输出级别 -->
        <root level="INFO">
            <!-- 下面设置输出到控制台和文件-->
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="INFO_FILE"/>
            <appender-ref ref="ERROR_FILE"/>
            <appender-ref ref="WARN_FILE"/>
        </root>
    </springProfile>
</configuration>
```