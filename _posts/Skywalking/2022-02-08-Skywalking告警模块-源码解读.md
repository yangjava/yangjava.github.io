1:告警规则定义
告警的核心是由’ alarm-setting.yml’文件中配置的一组规则驱动的。告警规则(Rules)的定义分为两部分

1.1:rules 定义了触发告警的度量和条件，该部分映射为AlarmRule类。
[Rule name] :规则名称，唯一标识，需以“_rule”结尾
Metrics-name :指标名称， oal脚本中度量名称，必须在oal脚本中定义过
include-names: 适用该规则的实体名称，可配置系统名称、服务名称。不配置默认所有实体均适用
Threshold: 阀值
op :操作符，和阀值一起使用
period :时间段，告警规则检查的时间段
Count:在period时间窗口内，达到op阀值的次数后即触发告警
Level:告警级别
silence-period:静默期，默认情况下与period相同，表示在静默期时间内，同样的告警只会触发一次
Message: 告警推送的消息
详细配置可参考github：

https://github.com/apache/skywalking/blob/master/docs/en/setup/backend/backend-alarm.md#entity-name

1.2:Webhooks触发告警规则后发送告警消息的服务端url列表


 2:告警处理类概述

告警模块中处理逻辑涉及到的类及其职责介绍

2.1:AlarmModule
org.apache.skywalking.oap.server.core.alarm.AlarmModule，继承 ModuleDefine 抽象类
Alarm管理器定义类。构造方法传入模块名name为 "alarm"
services()：返回MetricsNotify的实现类NotifyHandler
2.2:AlarmModuleProvider
org.apache.skywalking.oap.server.core.alarm.provider.AlarmModuleProvider,实现ModuleProvider 抽象类，Alarm模块的提供者Provider。在准备阶段ModuleDefine中调用，运行prepare()进行准备操作

name():实现方法，返回组件服务提供者名”default”
requiredModules():实现方法，返回组件”core”
prepare()：
调用RulesReader.readRules()方法读取alarm-setting.yml告警配置文件，将告警规则文件映射为Rules对象，其中Rules类由AlarmRule告警规则list和webhook发送地址list组成
新建NotifyHandler(Rules)对象，并调用其#init(AlarmCallback...)方法，传入参数为新建AlarmStandardPersistence对象
调用 #registerServiceImplementation(...) 方法，将NotifyHandler对象注册到 services
    

createConfigBeanIfAbsent()： 返回AlarmSettings
name()：提供名字 default
Module(): 实现方法 返回组件类AlarmModule
2.3:AlarmRulesWatcher
将Rules转换成RunningRule,构建Map<String, List> runningContext，metricsName对应RunningRule集合

和Map<AlarmRule, RunningRule> alarmRuleRunningRuleMap 规则和RunningRule对应MAP



2.4:NotifyHandler
告警服务实现类，处理告警逻辑

属性：

private final AlarmCore core; 告警核心处理
private final AlarmRulesWatcher alarmRulesWatcher
方法：

notify(Metrics metrics)：
1.接收Metrics，根据scope封装MetaInAlarm信息，获取MetricsName所有的RunningRule 集合

2.调用AlarmCore. #findRunningRule(String)，由入参MetaInAlarm指标名称获取匹配的 RunningRule实例List

3.若第一步返回的list不为空，对list<RunningRule>元素分别调用RunningRule.in()方法



2.5:RunningRule
告警规则执行类，分别对不同的告警规则的范围内的实例对象进行是否触发告警规则的判断并记录

方法：

in(MetaInAlarm meta, Metrics metrics)：添加metrics到window中
moveTo(LocalDateTime targetTime)：根据传入时间，调用Window内部类的moveTo方法移动windows属性所有元素的时间窗口
check()：检查条件，决定是否触发告警，告警阈值实际判断逻辑，调用Window内部类的checkAlarm()方法生成AlarmMessage对象，若checkAlarm()方法返回的不是Alarm对象，则将AlarmMessage对象加入到告警信息List<AlarmMessage>中，最后返回该告警信息List


2.6:Window
一个指标窗口，基于警报规则#period，这个窗口随时间滑动

属性：

endTime：当前window的结束时间
Period：告警规则rules的时间段
silenceCountdown：记录静默期剩余量，初始值为告警规则rules的silence-period 静默期。初始化为-1
Values：长度为period，记录endTime前period分钟的时间窗口内每分钟的metrics实例数据、
Lock：用于线程控制
方法：

add(Metrics metrics)：将metrics实例数据根据其时间戳加入到values中相应时间的位置上
moveTo(LocalDateTime current)：使用可重入锁进行线程同步控制，根据传入时间移动values时间窗口
若endTime为null，初始化values为长度period，值为null的list,并将传入的时间赋给endTime
若endTime不为null，若传入时间在endTime时间后则不做任何事，返回。再判断传入时间与endTime的分钟差，若分钟差大于period，则重新初始化values属性为长度period，值为null的list；否则移动values数据时间窗口到传入时间并设置window的结束时间endTime为传入时间
Ismatch：该方法主要进行是否触发告警规则的判断
2.7:AlarmEntrance
告警服务数据入口，判断有无初始化alarm模块，找到NotifyHandler，传送数据

forward(Metrics metrics): 转发数据到NotifyHandler
2.8:AlarmNotifyWorker
告警通知Worker，接受Metrics，

#(接告警数据来源，路由到告警核心。在MetricsAggregateWorker进行L1聚合之后执行告警工作)

in(Metrics metrics)： 转发数据给NotifyHandler处理 ---> 遍历所有scope的rule， rule列表通过解析alarm-setting,yaml得到–-> rule.in 判断metricsNAME是否相同，添加到windows里
2.9:AlarmCallback
告警信息持久化或推送的接口，只有一个方法doAlarm(List<AlarmMessage> alarmMessage)用于处理告警信息，AlarmStandardPersistence 类用于持久化告警信息；WebhookCallback类用于通过Webhook推送告警信息

# 在此强调一点，如果想要在skywalking中实现自己的通知钩子，需要实现AlarmCallback接口，实现doAlarm方法，参数就是AlarmMessage集合

在Rules中加入自己的配置


在NotifyHandler的init方法加入勾子


2.10:AlarmCore
告警核心处理，根据告警配置，包括某些时间窗口的指标值。通过使用它的内部定时器
触发器和告警规则来决定是否发送告警到数据库和webhook(s)

findRunningRule: 由指标名称(metricsName)获取List<RunningRule>
start(List allCallbacks)：在NotifyHandler初始化时调用，启动告警定时任务10s执行一次
1:初始化最后执行时间(lastExecuteTime属性)为当前系统时间

2:调用Executors.newSingleThreadScheduledExecutor().scheduleAtFixedRate方法新建每10秒定期执行的单线程、延迟10秒的线程池任务，任务流程如下

  2.1:计算任务最后执行时间(lastExecuteTime属性)与当前任务执行时间(checkTime取当前系统时间)之间相差的分钟数

  2.2:若分钟数不大于0，则本次任务不做任何事

      2.3:若分钟数大于0，则循环操作runningContext里的值List<RunningRule>的每个元素

首先调用RunningRule类的moveTo方法移动其时间窗口到当前任务执行时间(checkTime)，然后判断当前任务执行时间(checkTime)的秒数是否超过15，若超过15秒，则调用RunningRule类的check()方法生成告警信息放入List<AlarmMessage>，再判断生成的告警信息是否为空，不为空则对入参List<AlarmCallback>循环调用AlarmCallback.doAlarm方法推送告警信息并将任务最后执行时间(lastExecuteTime属性)更新为当前任务执行时间(checkTime，秒数设为00)



2.11:告警处理逻辑图


3:告警规则动态配置
告警模块中规则动态读取涉及到的类及其职责介绍
告警模块在准备阶段会初始化告警规则，AlarmModuleProvider加载alarm-setting.yml
当我们使用动态配置alarm-setting.yml的时候，处理流程，涉及到的类及其职责介绍如下

3.1:ConfigWatcherRegister
配置监听默认实现者，实现 DynamicConfigurationService

start()：启动定时任务读取配置
configSync()： 同步配置


readConfig() 读取配置，根据不同的数据源使用不同的实现类
4:告警数据来源
前面介绍了告警处理和告警规则动态配置，告警处理的NotifyHandler.notify(Metrics metrics)： 接收Metrics进行告警处理。这部分主要说下告警的metrics是怎么传递过去的

4.1:BootstrapFlow
流程启动类

start(): 加载启动执行的ModuleProvider，为其注入service和调用provider.start()。其中AnalyzerModuleProvider就是分析器提供者
4.2:AnalyzerModuleProvider
分析器提供者，加载oal/core.oal 解析成 OALDefine

start()：


4.3:OALEngineLoaderService
OAL加载处理类

load(): 解析OALDefine


4.4:OALRuntime
OAL Runtime是类生成引擎，它从OAL脚本定义加载生成的类，这个运行时动态加载

start(): 解析oal,生成运行时类
generateClassAtRuntime(): 根据oal动态生成告警指标实体类，生成metricsClasses 和 dispatcherClasses，并将其注入类加载器，其中dispatcher有调用MetricsStreamProcessor.getInstance().in(metrics);
notifyAllListeners(): 遍历生成的metricsClasses,启动监听，监听@Stream注解


4.5:StreamAnnotationListener
@Stream注解处理类，根据Processor类型处理，以上生成的metricsClasses注解@Sream中processor=MetricsStreamProcessor

notify()


4.6:MetricsStreamProcessor
指标Stream聚合处理器

属性:

entryWorkers： Metrics处理器 MetricsAggregateWorker集合，(@Stream注解类里的processor和metrics实体封装得到MetricsAggregateWorker)
方法:

create(): 为每个指标(上述生成的实体)创建工作者和工作流程 (包括AlarmNotifyWorker和MetricsAggregateWorker)
in(Metrics metrics): 接收Dispatcher来的数据，执行MetricsAggregateWorker.in() 进行L1聚合处理，之后传递给下一个Worker
4.7:MetricsAggregateWorker
提供了内存中的指标合并功能。指标聚合类（L1聚合）

onWork(): 处理数据


4.8:DispatcherManager
forward(Source source) 接收数据转发所有的本scope的Dispatcher，例如sourceaDispatcher把数据给MetricsStreamProcessor，MetricsStreamProcessor在交给具体MetricsAggregateWorker处理和AlarmNotifyWorker， MetricsRemoteWorker等处理器处理(当有数据上报时会调用此方法)
MetricsRemoteWorker: 将L1聚合后的数据通过grpc发送到远程
AlarmNotifyWorker:将聚合后的数据进行告警处理，(#接图一告警处理，则此告警处理流程即可串联起来了)
4.9:告警数据来源流程图


