---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---
# Prometheus一条告警是怎么触发的

**第一节：监控采集、计算和告警**
**第二节：告警分组、抑制、静默**
告警分组
告警抑制
告警静默
收敛小结
**第三节：告警延时**
[延时](https://so.csdn.net/so/search?q=延时&spm=1001.2101.3001.7020)的三个参数
延时小结
**总结**

![图片描述](https://image-static.segmentfault.com/297/892/2978923381-5b8f66a363bca_articlex)

Prometheus+Grafana是监控告警解决方案里的后起之秀，比如大家熟悉的PMM，就是使用了这个方案；前不久罗老师在3306pi公众号上就写过完整的使用教程《构建狂拽炫酷屌的MySQL 监控平台》，所以我们在这里就不再赘述具体如何搭建使用。

今天我们聊一些Prometheus几个有意思的特性，这些特性能帮助大家更深入的了解Prometheus的一条告警是怎么触发的；本文提纲如下：

监控采集，计算和告警
告警分组，抑制和静默
告警延时

**第一节 监控采集、计算和告警**
·
Prometheus以scrape_interval（默认为1m）规则周期，从监控目标上收集信息。其中scrape_interval可以基于全局或基于单个metric定义；然后将监控信息持久存储在其本地存储上。

Prometheus以evaluation_interval（默认为1m）另一个独立的规则周期，对告警规则做定期计算。其中evaluation_interval只有全局值；然后更新告警状态。

其中包含三种告警状态：

·inactive：没有触发阈值
·pending：已触发阈值但未满足告警持续时间
·firing：已触发阈值且满足告警持续时间

举一个例子，阈值告警的配置如下：

![图片描述](https://image-static.segmentfault.com/415/762/4157621069-5b8f66d5df8f0_articlex)

·收集到的mysql_uptime>=30,告警状态为inactive
·收集到的mysql_uptime<30,且持续时间小于10s，告警状态为pending
·收集到的mysql_uptime<30,且持续时间大于10s，告警状态为firing

⚠ 注意：配置中的for语法就是用来设置告警持续时间的；如果配置中不设置for或者设置为0，那么pending状态会被直接跳过。

那么怎么来计算告警阈值持续时间呢，需要回到上文的scrape_interval和evaluation_interval，假设scrape_interval为5s采集一次信息；evaluation_interval为10s；mysql_uptime告警阈值需要满足10s持续时间。

![图片描述](https://image-static.segmentfault.com/400/296/4002961703-5b8f672074806_articlex)

如上图所示：
Prometheus以5s（scrape_interval）一个采集周期采集状态；
然后根据采集到状态按照10s（evaluation_interval）一个计算周期，计算表达式；
表达式为真，告警状态切换到pending；
下个计算周期，表达式仍为真，且符合for持续10s，告警状态变更为active，并将告警从Prometheus发送给Altermanger；
下个计算周期，表达式仍为真，且符合for持续10s，持续告警给Altermanger；
直到某个计算周期，表达式为假，告警状态变更为inactive，发送一个resolve给Altermanger，说明此告警已解决。

**第二节 告警分组、抑制、静默**

第一节我们成功的把一条mysql_uptime的告警发送给了Altermanger;但是Altermanger并不是把一条从Prometheus接收到的告警简简单单的直接发送出去；直接发送出去会导致告警信息过多，运维人员会被告警淹没；所以Altermanger需要对告警做合理的收敛。

![图片描述](https://image-static.segmentfault.com/327/084/3270844178-5b8f674284fac_articlex)

如上图，蓝色框标柱的分别为告警的接收端和发送端；这张Altermanger的架构图里，可以清晰的看到，中间还会有一系列复杂且重要的流程，等待着我们的mysql_uptime告警。

下面我们来讲Altermanger非常重要的告警收敛手段。

·分组：group
·抑制：inhibitor
·静默：silencer

![图片描述](https://image-static.segmentfault.com/186/892/1868925742-5b8f6765d9765_articlex)

**1.告警分组**

告警分组的作用
·同类告警的聚合帮助运维排查问题
·通过告警邮件的合并，减少告警数量

举例来说：我们按照mysql的实例id对告警分组；如下图所示，告警信息会被拆分成两组。

·mysql-A

```undefined
 mysql_cpu_high
```

·mysql-B

```undefined
 mysql_uptime



 mysql_slave_sql_thread_down



 mysql_slave_io_thread_down
```

实例A分组下的告警会合并成一个告警邮件发送；
实例B分组下的告警会合并成一个告警邮件发送；

通过分组合并，能帮助运维降低告警数量，同时能有效聚合告警信息，帮助问题分析。

![图片描述](https://image-static.segmentfault.com/338/103/3381032590-5b8f67b0abee6_articlex)

**2.告警抑制**

告警抑制的作用

·消除冗余的告警

举例来说：同一台server-A的告警，如果有如下两条告警，并且配置了抑制规则。

·mysql_uptime
·server_uptime

最后只会收到一条server_uptime的告警。

A机器挂了，势必导致A服务器上的mysql也挂了；如配置了抑制规则，通过服务器down来抑制这台服务器上的其他告警；这样就能消除冗余的告警，帮助运维第一时间掌握最核心的告警信息。

![图片描述](https://image-static.segmentfault.com/338/103/3381032590-5b8f67b0abee6_articlex)

**3.告警静默**

告警静默的作用

阻止发送可预期的告警

举例来说：夜间跑批时间，批量任务会导致实例A压力升高；我们配置了对实例A的静默规则。

·mysql-A

```undefined
  qps_more_than_3000



  tps_more_than_2000



  thread_running_over_200
```

·mysql-B

```undefined
  thread_running_over_200
```

最后我们只会收到一条实例B的告警。

A压力高是可预期的，周期性的告警会影响运维判断；这种场景下，运维需要聚焦处理实例B的问题即可。

![图片描述](https://image-static.segmentfault.com/151/804/1518040510-5b8f6897a92b5_articlex)

**收敛小结**
这一节，我们mysql_uptime同学从Prometheus被出发后，进入了Altermanger的内部流程，并没有如预期的被顺利告出警来；它会先被分组，被抑制掉，被静默掉；之所以这么做，是因为我们的运维同学很忙很忙，精力非常有限；只有mysql_uptime同学证明自己是非常重要的，我们才安排它和运维同学会面。

**第三节 告警延时**

第二节我们提到了分组的概念，分组势必会带来延时；合理的配置延时，才能避免告警不及时的问题，同时帮助我们避免告警轰炸的问题。

我们先来看告警延时的几个重要[参数](https://so.csdn.net/so/search?q=参数&spm=1001.2101.3001.7020)：

group_by:分组参数，第二节已经介绍，比如按照[mysql-id]分组
group_wait:分组等待时间，比如：5s
group_interval:分组尝试再次发送告警的时间间隔，比如：5m
Repeat_interval: 分组内发送相同告警的时间间隔，比如：60m

延时参数主要作用在Altermanger的Dedup阶段，如图：

![图片描述](https://image-static.segmentfault.com/339/558/3395585872-5b8f68df74036_articlex)

**1.延时的三个参数**

我们还是举例来说，假设如下：

·配置了延时参数：

```undefined
      group_wait:5s



      group_interval:5m
```

repeat_interval: 60m
·有同组告警集A，如下：

```undefined
      a1



      a2



      a3
```

·有同组告警集B，如下：

```undefined
      b1



      b2
```

**场景一：**

1.a1先到达告警系统，此时在group_wait:5s的作用下，a1不会立刻告出来，a1等待5s，下一刻a2在5s内也触发，a1,a2会在5s后合并为一个分组，通过一个告警消息发出来；
2.a1,a2持续未解决，它们会在repeat_interval: 60m的作用下，每隔一小时发送告警消息。

![图片描述](https://image-static.segmentfault.com/929/531/929531963-5b8f69924ce73_articlex)

**场景二：**

1.a1,a2持续未解决，中间又有新的同组告警a3出现，此时在group_interval:5m的作用下，由于同组的状态发生变化，a1,a2,a3会在5min中内快速的告知运维，不会被收敛60min（repeat_interval）的时间；
2.a1,a2,a3如持续无变化，它们会在repeat_interval: 60m的作用下，再次每隔一小时发送告警消息。

![图片描述](https://image-static.segmentfault.com/348/594/3485948370-5b8f69b80a442_articlex)

**场景三：**

1.a1,a2发生的过程中，发生了b1的告警，由于b1分组规则不在集合A中，所以b1遵循集合B的时间线；
2.b1发生后发生了b2，b1,b2按照类似集合A的延时规则收敛，但是时间线独立。

![图片描述](https://image-static.segmentfault.com/241/622/2416223728-5b8f69e2b593b_articlex)

**延时小结**

通过三个延时参数，告警实现了分组等待的合并发送（group_wait），未解决告警的重复提醒（repeat_interval），分组变化后快速提醒（group_interval）。

**总结**

本文通过监控信息的周期性采集、告警公式的周期性计算、合并同类告警的分组、减少冗余告警的抑制、降低可预期告警的静默、同时配合三个延时参数，讲解了Prometheus的一条告警是怎么触发的；当然对于Prometheus，还有很多特性可以实践；如您有兴趣，欢迎联系我们，我们是爱可生。