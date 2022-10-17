# Prometheus

## 监控系统的前世今生

随着互联网的发展，监控系统也得到了发展。从最早期的网络监控、系统监控，发展到现在的业务监控、日志监控、性能监控、代码监控、全链路监控等，并在监控数据的基础上，逐步发展出了APM（应用性能管理）、AIOps（智能运维）等。

本文主要介绍监控系统的历史、流派及如何选型，希望对读者能有所帮助。

### 监控系统的历史

首先来看看监控系统的发展历程和常用工具软件，如下所示。

```
网络监控：OpenView  Tivoli Unicenter
系统监控：Ganglia Collected OpenNMS Cacti/RRDTool
服务应用监控：Zabbix Zenoss Nagios
业务监控： Graphite Grafana ElasticSearch StatsD TSDB Druid Hadoop/Spark
APM和AIOPS：
            终端用户体验监控
            应用拓扑的发现和可视化
            应用组件的深度监控
            异常检测。故障定位。大数据。机器学习
            自动化扩缩容。降级。封杀。流量调度
```

#### 早期的监控系统

互联网发展早期的监控系统，主要是指基于SNMP（简单网络管理协议）的网络监控和系统（主要指操作系统）监控。这个时候的互联网应用都很简单，只有网络设备和操作系统可以提供标准的SNMP服务，一些Web服务器、中间件也支持通过SNMP获取状态，但不是很完善。

而且在这一时期，开源还不流行，业界主流的商业监控系统（实际上监控只是这些商业管理软件的一小部分功能）有IBM的Tivoli、HP的OpenView、CA的UniCenter，主要客户是银行和电信，而弱小的互联网公司（特指那个时代）用不起。

#### 现在的监控系统

随着互联网公司的发展和强大，他们对业务、服务、应用也逐渐有了较强的监控需求，而基于前面的理由，互联网公司的监控系统一般都是走自主研发和开源软件相结合的路子。毕竟“昂贵”、“耗时”、“流程”这些词在互联网公司难以生存，而能发扬光大的系统一般具有“便宜”、“快速”、“简单”的特色。当时可用的开源监控软件包括Cacti、Zabbix、Nagios、RRDTool，这些软件今天仍然很活跃，像RRDTool这样的时序数据存储方式也是目前很多时序数据库参考的标准。

业务监控继续发展，并且更加细分，出现了性能监控、代码监控、日志监控、全链路跟踪（Trace）等方向。相应地有了全面的监控、日志分析等功能，有了告警的需求。随着告警功能的完善，出现了关联、收敛等技术，并能提供一定的建议，接着干预手段（降级、封禁、流量切换、扩缩容）也可以用上了。

#### 前沿方向

随着行业做到一定的程度，大家的应用水平都差不多，区别在于工程水平、产品化的能力，基于前面这些基础，又演化出了两个比较前沿的方向：APM和AIOps。

APM，即应用性能管理，定义了五个功能维度，分别为真实用户体验监控、运行时应用拓扑的发现和可视化、用户自定义业务分析、应用组件深度监控、运营分析。APM各大厂实施的程度也不太一样，或多或少都能靠上一部分。国外做的比较好的SaaS厂商有NewRelic 和AppDynamics，国内的读者可以自行搜索。

AIOps，原先指“AlgorithmicIT Operations”，也就是基于算法的IT运维，即利用数据和算法提高运维的自动化程度和效率，涵盖了数据的收集、存储、分析、可视化，以及通过API提供与第三方工具集成的能力，从这个角度来说，AIOps存在了很久，目前大多数公司努力达到的也是这个层次（但是国内除了少数初创公司，大部分公司内部各部门之间的运维、监控数据的互联互通都还做不到，别说在更高层次上统筹考虑运维方案了）。在这个基础上，再加上火热的大数据和机器学习，AIOps的内涵得到了发展，即我们现在所说的“智能运维”（Artificial Intelligencefor IT Operations），目前各个公司都在尝试使AIOps落地。

### 监控流派

说到流派，每个人都会有自己的喜好和一套理论，下面会对它们进行对比，读者自行评判选择。

Agent与Agentless

在我们的监控实践活动中，一般将必须要安装配置、对运行环境比较敏感的监控组件（一般完成信息采集和初步聚合）称为Agent，而相对应地，不需要安装、直接运行的脚本、远程SSH和基于SNMP服务、第三方管理API获取信息的方式称为Agentless（无代理）。

Agent与Agentless对比如图3所示。

|                  | Agentless                      | Agent                        |
| ---------------- | ------------------------------ | ---------------------------- |
| 部署方便性       | 部署简单，可以借助Puppet等工具 | 必须每台安装，对目标环境敏感 |
| 安全性           | 明文传输                       | 内部协议，安全性高           |
| 网络流量         | 原始数据，带宽占用大           | 只传输预处理和聚合后的数据   |
| 监控的深度和广度 | 广度取决于SNMP或者提供API      | 一般具有较好的深度和广度，   |
| 入门门槛         | 较高，需要充分理解监控需求     | 较低                         |

Total solution与自由组合

所谓“Total Solution”（整体解决方案）特指拥有特别多功能的、“大而全”的监控系统，能完成包括数据收集、聚合、存储、展示、告警等全套功能，Zabbix、Zenoss、Open-falcon、Prometheus等都是其代表。这一类功能比较完整的监控系统特点就是“完整”，除了必要的配置，一般你不需要考虑在其之上开发什么附加功能（当然二次开发也比较困难）。

“自由组合”是另外一种流派，核心思想就是“小步快跑”、“每次只做一件事”、“每个组件只完成一个功能”。具体说起来，就是通过组合各种小工具、循序渐进的实现一系列功能，为什么强调每次只做一件事呢？因为需求不明确，或者说需求变化太快，尤其互联网公司，业务更新变化太快，在这种环境下，不太适合规划一个需要较长开发周期、拥有很多功能的系统。

很难说哪一种方式较好，只能说哪一种方式比较适合。Total Solution的好处是可以快速搭建一套完整的监控系统，即使是默认配置，对于不太复杂的监控需求一般都能满足；小步快跑的好处是在一开始需求不明确的情况下，专注于矛盾最突出的地方，专注解决一个点，如有必要再扩展。

选型

选型的意思就是选择哪一种监控体系，是成熟的产品，还是自己研发，抑或基于开源软件来集成。当我们开始规划一个监控系统的时候，这问题就需要预先考虑和分析，列出竞品之间优缺点，结合需求来选择，而不是自己熟悉那个就用那个，也不是因为别人用了，所以自己也要用。

需要解决什么问题

选择监控系统，需要先问自己一些问题，明确自己的需求，下面是这些问题的范例。

我有很多服务器、数据库、网络设备，但可以知道它们的状态吗？

我给客户提供了一项服务，但服务是否有问题、服务质量如何？

我有一个（些）监控系统，但我对效果/成本/功能满意吗？（不满意是常态。）

这些问题的答案就对应着不同的解决方案：基础监控、业务服务质量监控、性能监控等，另外可以明确是需要重新建设还是在原有的基础上升级和补充。

分析自己的环境

“环境”包括了软硬件的运行环境，比如操作系统的版本、容器、框架、日志落地方式等，一般经过一段时间发展，环境基本上会变得“五花八门”，这时选取一个各种环境都容易集成的方案会比较好，也就是一个计算“较大公约数”的过程。

确定你的预算

有人觉得自己使用的是开源软件，应该没有预算问题，但是这背后还是会有很多成本的。首先就是学习和时间成本，你需要理解软件的理念和设计思想，判断是否能解决自己的问题；其次是部署和二次开发的成本，很多时候开源软件文档并不完善，需要自己探索，并且可能不能直接用于自己的环境，所以面临二次开发。一定要提前规划，看是否能够接受这些成本。 

本文的内容基本介绍完毕，简单来说，如果能预计自己的数据量，并且想尽快看到效果，那么直接用成熟的“Total Solution”比较好，前期成本较低，建设速度也比较快。另外一方面，如果需求不太明确，数据量无法估算，建议还是走“自由组合”的方式，利用一些小工具，先完成主要功能，再逐步迭代和演进。当然，现实中少有人会全盘推翻之前的遗留系统重新建设一套，一般大家都是从一个还“能用”的系统起步，再组合各种工具。

## 监控系统的目标

我们先来了解什么是监控，监控的重要性以及监控的目标，当然每个人所在的行业不同、公司不同、业务不同、岗位不同、对监控的理解也不同，但是我们需要注意，监控是需要站在公司的业务角度去考虑，而不是针对某个监控技术的使用。

监控目标

- 1.对系统不间断实时监控:实际上是对系统不间断的实时监控(这就是监控)
- 2.实时反馈系统当前状态:我们监控某个硬件、或者某个系统，都是需要能实时看到当前系统的状态，是正常、异常、或者故障
- 3.保证服务可靠性安全性:我们监控的目的就是要保证系统、服务、业务正常运行
- 4.保证业务持续稳定运行:如果我们的监控做得很完善，即使出现故障，能第一时间接收到故障报警，在第一时间处理解决，从而保证业务持续性的稳定运行。

监控方法

1.了解监控对象:我们要监控的对象你是否了解呢？比如CPU到底是如何工作的？ 
2.性能基准指标:我们要监控这个东西的什么属性？比如CPU的使用率、负载、用户态、内核态、上下文切换。 
3.报警阈值定义:怎么样才算是故障，要报警呢？比如CPU的负载到底多少算高，用户态、内核态分别跑多少算高？ 
4.故障处理流程:收到了故障报警，那么我们怎么处理呢？有什么更高效的处理流程吗？

监控核心

1.发现问题:当系统发生故障报警，我们会收到故障报警的信息 
2.定位问题:故障邮件一般都会写某某主机故障、具体故障的内容，我们需要对报警内容进行分析，比如一台服务器连不上:我们就需要考虑是网络问题、还是负载太高导致长时间无法连接，又或者某开发触发了防火墙禁止的相关策略等等，我们就需要去分析故障具体原因。 
3.解决问题:当然我们了解到故障的原因后，就需要通过故障解决的优先级去解决该故障。 
4.总结问题:当我们解决完重大故障后，需要对故障原因以及防范进行总结归纳，避免以后重复出现。

## 监控系统的设计

工欲善其事必先利其器，根据对一些监控产品的调研以及对监控的分层介绍、所需解决的问题，可以发现监控系统从收集到分析的流程架构：采集－存储－分析－展示－告警。

**数据采集（collector）:** 通过SNMP、Agent、ICMP、SSH、IPMI等协议对系统进行数据采集。

**数据存储（*storage*）：**主要存储在MySQL上，也可以存储在其他数据库服务。

**数据分析（engine）：**当事后需要复盘分析故障时，监控系统能给我们提供图形和时间等相关信息，方面确定故障所在。

**数据展示（view）：**Web界面展示(移动APP、java_php开发一个web界面也可以)。

**监控报警（alert Manager）：**电话报警、邮件报警、微信报警、短信报警、报警升级机制等（无论什么报警都可以）。

**报警处理：**当接收到报警，我们需要根据故障的级别进行处理，比如：重要紧急、重要不紧急等。根据故障的级别，配合相关的人员进行快速处理。

![Prometheus1](Prometheus1.png)

## 监控系统的体系

评价一个监控系统的好坏最重要三要素是：监控粒度、监控指标完整性、监控实时性，从系统分层体系可以把监控系统分为三个层次：

#### 基础设施层

实时掌握服务器工作状态，留意性能、内存消耗、容量和整体系统健康状态，保证服务器稳定运行。监控指标：内存、磁盘、CPU、网络流量、系统进程等系统级性能指标。

**基础设施层的监控又分为状态、性能、质量、容量、架构等几个层面。举例说明:**

**状态监控**包括网络设备的软硬件状态（如设备存活状态、板卡、电源、风扇状态，设备温度、光功率、OSPF状态、生成树状态等）； 

**性能监控**包括设备CPU、设备内存大小、session数量、端口流量包量、内存溢出监控、内存使用率等； 

**质量监控**包括设备错包、丢包率，针对网络设备以及网络链路的探测延时、丢包率监控等；

 **容量监控**包括设备负载使用率、专线带宽使用率、出口流量分布等； 

**架构监控**包括路由跳变、缺失、绕行，流量穿越监控等。

#### 服务器层

服务器是业务部署运行起来的载体（早期服务器就是我们传统观念上的“物理机+操作系统”，现在已经扩大到虚拟机或者是容器等范畴）。服务器层的监控包括硬件层面和软件层面。

**硬件层面的监控**主要包括如下内容:

- **硬盘**：硬盘读写错误、读写超时、硬盘掉线、硬盘介质错误、[SSD硬盘]硬盘温度、硬盘寿命、硬盘坏块率；
- **内存**：内存缺失、内存配置错误、内存不可用、内存校验；网卡：网卡速率；
- **电源**：电源电压、电源模块是否失效；风扇：风扇转速；
- **Raid卡**：Raid卡电池状态、电池老化、电池和缓存是否在位、缓存策略。

**软件层面的监控**主要包括：CPU（CPU整体使用率、CPU各核使用率、CPU Load负载）、内存（应用内存、整体内存、Swap等）、磁盘IO（读写速率、IOPS、平均等待延时、平均服务延时等）、网络IO（流量、包量、错包、丢包）、连接（各种状态的TCP连接数等）、进程端口存活、文件句柄数、进程数、内网探测延时、丢包率等。

#### 平台层

对应用的整体运行状况进行了解、把控，如果将应用当成黑盒子，开发和运维就无从知晓应用当前状态，不能及时发现潜在故障。应用监控不应局限于业务系统，还包括各种中间件和计算引擎，如ClickHouse、ElasticSearch、redis、zookeeper、kafka等。常用监控数据：JVM堆内存、GC、CPU使用率、线程数、TPS、吞吐量等，一般通过抽象出的统一指标收集组件，收集应用级指标。

#### 业务层

业务系统本质目的是为了达成业务目标，因此监控业务系统是否正常最有效的方式是从数据上监控业务目标是否达成。对业务运营数据进行监控，可及时发现程序bug或业务逻辑设计缺陷，比如数据趋势、流量大小等。业务系统的多样性决定了应由各个业务系统实现监控指标开发。

## 著名的监控方法论

### **4个黄金指标**

Four Golden Signals是Google针对大量分布式监控的经验总结，4个黄金指标可以在服务级别帮助衡量终端用户体验、服务中断、业务影响等层面的问题。主要关注与以下四种类型的指标：延迟，通讯量，错误以及饱和度：

- 延迟：服务请求所需时间。

记录用户所有请求所需的时间，重点是要区分成功请求的延迟时间和失败请求的延迟时间。 例如在数据库或者其他关键祸端服务异常触发HTTP 500的情况下，用户也可能会很快得到请求失败的响应内容，如果不加区分计算这些请求的延迟，可能导致计算结果与实际结果产生巨大的差异。除此以外，在微服务中通常提倡“快速失败”，开发人员需要特别注意这些延迟较大的错误，因为这些缓慢的错误会明显影响系统的性能，因此追踪这些错误的延迟也是非常重要的。

- 通讯量：监控当前系统的流量，用于衡量服务的容量需求。

流量对于不同类型的系统而言可能代表不同的含义。例如，在HTTP REST API中, 流量通常是每秒HTTP请求数；

- 错误：监控当前系统所有发生的错误请求，衡量当前系统错误发生的速率。

对于失败而言有些是显式的(比如, HTTP 500错误)，而有些是隐式(比如，HTTP响应200，但实际业务流程依然是失败的)。

对于一些显式的错误如HTTP 500可以通过在负载均衡器(如Nginx)上进行捕获，而对于一些系统内部的异常，则可能需要直接从服务中添加钩子统计并进行获取。

- 饱和度：衡量当前服务的饱和度。

主要强调最能影响服务状态的受限制的资源。 例如，如果系统主要受内存影响，那就主要关注系统的内存状态，如果系统主要受限与磁盘I/O，那就主要观测磁盘I/O的状态。因为通常情况下，当这些资源达到饱和后，服务的性能会明显下降。同时还可以利用饱和度对系统做出预测，比如，“磁盘是否可能在4个小时候就满了”。

### **RED方法**

RED方法是Weave Cloud在基于Google的“4个黄金指标”的原则下结合Prometheus以及Kubernetes容器实践，细化和总结的方法论，特别适合于云原生应用以及微服务架构应用的监控和度量。主要关注以下三种关键指标：

- (请求)速率：服务每秒接收的请求数。
- (请求)错误：每秒失败的请求数。
- (请求)耗时：每个请求的耗时。

在“4大黄金信号”的原则下，RED方法可以有效的帮助用户衡量云原生以及微服务应用下的用户体验问题。

### **USE方法**

USE方法全称"Utilization Saturation and Errors Method"，主要用于分析系统性能问题，可以指导用户快速识别资源瓶颈以及错误的方法。正如USE方法的名字所表示的含义，USE方法主要关注与资源的：使用率(Utilization)、饱和度(Saturation)以及错误(Errors)。

- 使用率：关注系统资源的使用情况。 这里的资源主要包括但不限于：CPU，内存，网络，磁盘等等。100%的使用率通常是系统性能瓶颈的标志。
- 饱和度：例如CPU的平均运行排队长度，这里主要是针对资源的饱和度(注意，不同于4大黄金信号)。任何资源在某种程度上的饱和都可能导致系统性能的下降。
- 错误：错误计数。例如：“网卡在数据包传输过程中检测到的以太网网络冲突了14次”。

通过对资源以上指标持续观察，通过以下流程可以知道用户识别资源瓶颈：

##  **Prometheus简介**

**Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud. Since its inception in 2012, many companies and organizations have adopted Prometheus, and the project has a very active developer and user community. It is now a standalone open source project and maintained independently of any company. To emphasize this, and to clarify the project's governance structure, Prometheus joined the Cloud Native Computing Foundation in 2016 as the second hosted project, after Kubernetes.**

#### **什么是Prometheus？**

Prometheus是一个开源监控系统，它前身是SoundCloud的警告工具包。从2012年开始，许多公司和组织开始使用Prometheus。该项目的开发人员和用户社区非常活跃，越来越多的开发人员和用户参与到该项目中。目前它是一个独立的开源项目，且不依赖与任何公司。为了强调这点和明确该项目治理结构，Prometheus在2016年继Kurberntes之后，加入了Cloud Native Computing Foundation。

Prometheus是由SoundCloud开发的开源监控报警系统和时序列数据库(TSDB)。Prometheus使用Go语言开发，是Google BorgMon监控系统的开源版本。
2016年由Google发起Linux基金会旗下的原生云基金会(Cloud Native Computing Foundation), 将Prometheus纳入其下第二大开源项目。
Prometheus目前在开源社区相当活跃。
Prometheus和Heapster(Heapster是K8S的一个子项目，用于获取集群的性能数据。相比功能更完善、更全面。Prometheus性能也足够支撑上万台规模的集群。

#### 特点

1）多维度数据模型

每一个时间序列数据都由metric度量指标名称和它的标签labels键值对集合唯一确定：这个metric度量指标名称指定监控目标系统的测量特征（如：http_requests_total- 接收http请求的总计数）。

labels开启了Prometheus的多维数据模型：对于相同的度量名称，通过不同标签列表的结合, 会形成特定的度量维度实例。(例如：所有包含度量名称为/api/tracks的http请求，打上method=POST的标签，则形成了具体的http请求)。这个查询语言在这些度量和标签列表的基础上进行过滤和聚合。改变任何度量上的任何标签值，则会形成新的时间序列图。

2）灵活的查询语言（PromQL）：可以对采集的metrics指标进行加法，乘法，连接等操作；

3）可以直接在本地部署，不依赖其他分布式存储；

4）通过基于HTTP的pull方式采集时序数据；

5）可以通过中间网关pushgateway的方式把时间序列数据推送到prometheus server端；

6）可通过服务发现或者静态配置来发现目标服务对象（targets）。

7）有多种可视化图像界面，如Grafana等。

8）高效的存储，每个采样数据占3.5 bytes左右，300万的时间序列，30s间隔，保留60天，消耗磁盘大概200G。

9）做高可用，可以对数据做异地备份，联邦集群，部署多套prometheus，pushgateway上报数据

#### 组件

1）`Prometheus Server`: 用于收集和存储时间序列数据。

2）`Client Library`: 客户端库，检测应用程序代码，当Prometheus抓取实例的HTTP端点时，客户端库会将所有跟踪的metrics指标的当前状态发送到prometheus server端。

3）`Exporters`: prometheus支持多种exporter，通过exporter可以采集metrics数据，然后发送到prometheus server端，所有向promtheus server提供监控数据的程序都可以被称为exporter

4）`Alertmanager`: 从 Prometheus server 端接收到 alerts 后，会进行去重，分组，并路由到相应的接收方，发出报警，常见的接收方式有：电子邮件，微信，钉钉, slack等。

5）`Grafana`：监控仪表盘，可视化监控数据

6）`pushgateway`: 各个目标主机可上报数据到pushgateway，然后prometheus server统一从pushgateway拉取数据。

#### **适用场景**

Prometheus在记录纯数字时间序列方面表现非常好。它既适用于面向服务器等硬件指标的监控，也适用于高动态的面向服务架构的监控。对于现在流行的微服务，Prometheus的多维度数据收集和数据筛选查询语言也是非常的强大。Prometheus是为服务的可靠性而设计的，当服务出现故障时，它可以使你快速定位和诊断问题。它的搭建过程对硬件和服务没有很强的依赖关系。

Prometheus，它的价值在于可靠性，甚至在很恶劣的环境下，你都可以随时访问它和查看系统服务各种指标的统计信息。 如果你对统计数据需要100%的精确，它并不适用，例如：它**不适用于实时计费系统**。



#### 什么是样本

**样本**：在时间序列中的每一个点称为一个样本（sample），样本由以下三部分组成：

- 指标（metric）：指标名称和描述当前样本特征的 labelsets；
- 时间戳（timestamp）：一个精确到毫秒的时间戳；
- 样本值（value）： 一个 folat64 的浮点型数据表示当前样本的值。

表示方式：通过如下表达方式表示指定指标名称和指定标签集合的时间序列：

```
<metric name>{<label name>=<label value>, ...}
```

例如，指标名称为api_http_requests_total，标签为 method="POST" 和 handler="/messages" 的时间序列可以表示为：

```
api_http_requests_total{method="POST", handler="/messages"}
```

## Prometheus架构

### 架构图

![Prometheus](Prometheus.png)

从上图可发现，Prometheus整个生态圈组成主要包括prometheus server，Exporter，pushgateway，alertmanager，grafana，Web ui界面。

Prometheus server由三个部分组成，Retrieval，Storage，PromQL

- `Retrieval`负责在活跃的target主机上抓取监控指标数据
- `Storage`存储主要是把采集到的数据存储到磁盘中
- `PromQL`是Prometheus提供的查询语言模块。

### 工作流程

1）Prometheus server可定期从活跃的（up）目标主机上（target）拉取监控指标数据，目标主机的监控数据可通过配置静态job或者服务发现的方式被prometheus server采集到，这种方式默认的pull方式拉取指标；也可通过pushgateway把采集的数据上报到prometheus server中；还可通过一些组件自带的exporter采集相应组件的数据；

2）Prometheus server把采集到的监控指标数据保存到本地磁盘或者数据库；

3）Prometheus采集的监控指标数据按时间序列存储，通过配置报警规则，把触发的报警发送到alertmanager

4）Alertmanager通过配置报警接收方，发送报警到邮件，微信或者钉钉等

5）Prometheus 自带的web ui界面提供PromQL查询语言，可查询监控数据

6）Grafana可接入prometheus数据源，把监控数据以图形化形式展示出

## Prometheus**基础概念**

### 数据模型

Prometheus 存储的是[时序数据](https://en.wikipedia.org/wiki/Time_series), 即按照相同时序(相同的名字和标签)，以时间维度存储连续的数据的集合。时序(time series) 是由名字(Metric)，以及一组 key/value 标签定义的，具有相同的名字以及标签属于相同时序。时序的名字由 ASCII 字符，数字，下划线，以及冒号组成，它必须满足正则表达式 `[a-zA-Z_:][a-zA-Z0-9_:]*`, 其名字应该具有语义化，一般表示一个可以度量的指标，例如 `http_requests_total`, 可以表示 http 请求的总数。

时序的标签可以使 Prometheus 的数据更加丰富，能够区分具体不同的实例，例如 `http_requests_total{method="POST"}` 可以表示所有 http 中的 POST 请求。标签名称由 ASCII 字符，数字，以及下划线组成， 其中 `__` 开头属于 Prometheus 保留，标签的值可以是任何 Unicode 字符，支持中文。

### **指标类型**（Metric Types）

Prometheus 时序数据分为 [Counter](https://prometheus.io/docs/concepts/metric_types/#counter), [Gauge](https://prometheus.io/docs/concepts/metric_types/#gauge), [Histogram](https://prometheus.io/docs/concepts/metric_types/#histogram), [Summary](https://prometheus.io/docs/concepts/metric_types/#summary) 四种类型。

1. Counter：表示收集的数据是按照某个趋势（增加／减少）一直变化的，我们往往用它记录服务请求总量，错误总数等。例如 Prometheus server 中 `http_requests_total`, 表示 Prometheus 处理的 http 请求总数，我们可以使用data, 很容易得到任意区间数据的增量。
2. Gauge：表示搜集的数据是一个瞬时的，与时间没有关系，可以任意变高变低，往往可以用来记录内存使用率、磁盘使用率等。
3. Histogram：Histogram 由 `<basename>_bucket{le="<upper inclusive bound>"}`，`<basename>_bucket{le="+Inf"}`, `<basename>_sum`，`<basename>_count` 组成，主要用于表示一段时间范围内对数据进行采样，（通常是请求持续时间或响应大小），并能够对其指定区间以及总数进行统计，通常我们用它计算分位数的直方图。
4. Summary：Summary 和 Histogram 类似，由 `<basename>{quantile="<φ>"}`，`<basename>_sum`，`<basename>_count`组成，主要用于表示一段时间内数据采样结果，（通常是请求持续时间或响应大小），它直接存储了 quantile 数据，而不是根据统计区间计算出来的。区别在于：

　　　　  a. 都包含 `<basename>_sum`，`<basename>_count。`

``        b. Histogram 需要通过 `<basename>_bucket` 计算 quantile, 而 Summary 直接存储了 quantile 的值。

### 作业（Job）和实例（Instance）



## PromQL

## Instrumentation（程序仪表）

## Exporters

## Alerts

## Prometheus的几种部署模式

### 基本高可用模式

基本的HA模式只能确保Promthues服务的可用性问题，但是**不解决Prometheus Server之间的数据一致性问题以及持久化问题(数据丢失后无法恢复)，也无法进行动态的扩展**。因此这种部署方式适合监控规模不大，Promthues Server也不会频繁发生迁移的情况，并且只需要保存短周期监控数据的场景。

### 基本高可用+远程存储

在解决了Promthues服务可用性的基础上，同时确保了数据的持久化，当Promthues Server发生宕机或者数据丢失的情况下，可以快速的恢复。 同时Promthues Server可能很好的进行迁移。因此，该方案适用于用户监控规模不大，但是希望能够将监控数据持久化，同时能够确保Promthues Server的可迁移性的场景。

### 基本HA + 远程存储 + 联邦集群方案

Promthues的性能瓶颈主要在于大量的采集任务，因此用户需要利用**Prometheus联邦集群**的特性，将不同类型的采集任务划分到不同的Promthues子服务中，从而实现功能分区。例如一个Promthues Server负责采集基础设施相关的监控指标，另外一个Prometheus Server负责采集应用监控指标。再有上层Prometheus Server实现对数据的汇聚。

## 三大套件

- Server 主要负责数据采集和存储，提供PromQL查询语言的支持。
- Alertmanager 警告管理器，用来进行报警。
- Push Gateway 支持临时性Job主动推送指标的中间网关。

## 数据模型

prometheus将所有数据存储为时间序列：属于相同 metric名称和相同标签组（键值对）的时间戳值流。

metric 和 标签：
每一个时间序列都是由其 metric名称和一组标签（键值对）组成唯一标识。

metric名称代表了被监控系统的一般特征（如 http_requests_total代表接收到的HTTP请求总数）。它可能包含ASCII字母和数字，以及下划线和冒号，它必须匹配正则表达式[a-zA-Z_:][a-zA-Z0-9_:]*。

注意：冒号是为用户定义的记录规则保留的，不应该被exporter使用。

标签给prometheus建立了多维度数据模型：对于相同的 metric名称，标签的任何组合都可以标识该 metric的特定维度实例（例如：所有使用POST方法到 /api/tracks 接口的HTTP请求）。查询语言会基于这些维度进行过滤和聚合。更改任何标签值，包括添加或删除标签，都会创建一个新的时间序列。

标签名称可能包含ASCII字母、数字和下划线，它必须匹配正则表达式[a-zA-Z_][a-zA-Z0-9_]*。另外，以双下划线__开头的标签名称仅供内部使用。

标签值可以包含任何Unicode字符。标签值为空的标签被认为是不存在的标签。

表示法：
给定 metric名称和一组标签，通常使用以下表示法标识时间序列：

<metric name>{<label name>=<label value>, ...}
1
例如，一个时间序列的 metric名称是 api_http_requests_total，标签是 method="POST" 和 handler="/messages"。可以这样写：

api_http_requests_total{method="POST", handler="/messages"}
1
这和OpenTSDB的表示法是一样的。

| 类型      | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| Counter   | 值只能单调增加或重启时归零，可以用来表示处理的请求数、完成的任务数、出现的错误数量等 |
| Gauge     | 值可以任意增加或减少，可以用来测量温度、当前内存使用等       |
| Histogram | 取样观测结果，一般用来请求持续时间或响应大小，并在一个可配置的分布区间（bucket）内计算这些结果，提供所有观测结果的总和<br/>                        <br/>                        累加的 counter，代表观测区间：<basename>_bucket{le="<upper inclusive bound>"}<br/>                        所有观测值的总数：<basename>_sum<br/>                        观测的事件数量：<basenmae>_count |
| Summary   | 取样观测结果，一般用来请求持续时间或响应大小，提供观测次数及所有观测结果的总和，还可以通过一个滑动的时间窗口计算可分配的分位数<br/>                        观测的事件流φ-quantiles (0 ≤ φ ≤ 1)：<basename>{quantile="φ"}<br/>                        所有观测值的总和：<basename>_sum<br/>                        观测的事件数量：<basename>_count |

实例与任务：
在prometheus中，一个可以拉取数据的端点叫做实例（instance），一般等同于一个进程。一组有着同样目标的实例（例如为弹性或可用性而复制的进程副本）叫做任务（job）。

当prometheus拉取目标时，它会自动添加一些标签到时间序列中，用于标识被拉取的目标：

job：目标所属的任务名称

instance：目标URL中的<host>:<port>部分
如果两个标签在被拉取的数据中已经存在，那么就要看配置选项 honor_labels 的值来决定行为了。

每次对实例的拉取，prometheus会在以下的时间序列中保存一个样本（样本指的是在一个时间序列中特定时间点的一个值）：

up{job="<job-name>", instance="<instance-id>"}：如果实例健康（可达），则为 1 ，否则为 0

scrape_duration_seconds{job="<job-name>", instance="<instance-id>"}：拉取的时长

scrape_samples_post_metric_relabeling{job="<job-name>", instance="<instance-id>"}：在 metric relabeling 之后，留存的样本数量

scrape_samples_scraped{job="<job-name>", instance="<instance-id>"}：目标暴露出的样本数量
up 时间序列对于实例的可用性监控来说非常有用。


### Counter

Counter是**计数器类型**：

- Counter 用于累计值，例如记录请求次数、任务完成数、错误发生次数。
- 一直增加，不会减少。
- 重启进程后，会被重置。

```bash
# Counter类型示例
http_response_total{method="GET",endpoint="/api/tracks"}  100
http_response_total{method="GET",endpoint="/api/tracks"}  160
```

Counter 类型数据可以让用户方便的了解事件产生的速率的变化，在PromQL内置的相关操作函数可以提供相应的分析，比如以HTTP应用请求量来进行说明

1）通过`rate()`函数获取HTTP请求量的增长率：`rate(http_requests_total[5m])`

2）查询当前系统中，访问量前10的HTTP地址：`topk(10, http_requests_total)`

### Gauge

Gauge是**测量器类型**：

- Gauge是常规数值，例如温度变化、内存使用变化。
- 可变大，可变小。
- 重启进程后，会被重置

```bash
# Gauge类型示例
memory_usage_bytes{host="master-01"}   100
memory_usage_bytes{host="master-01"}   30
memory_usage_bytes{host="master-01"}   50
memory_usage_bytes{host="master-01"}   80 
```

对于 Gauge 类型的监控指标，通过 PromQL 内置函数 `delta()` 可以获取样本在一段时间内的变化情况，例如，计算 CPU 温度在两小时内的差异：
`dalta(cpu_temp_celsius{host="zeus"}[2h])`

你还可以通过PromQL 内置函数 `predict_linear()` 基于简单线性回归的方式，对样本数据的变化趋势做出预测。例如，基于 2 小时的样本数据，来预测主机可用磁盘空间在 4 个小时之后的剩余情况：`predict_linear(node_filesystem_free{job="node"}[2h], 4 * 3600) < 0`

### Histogram

#### Histogram作用及特点

histogram是**柱状图**，在Prometheus系统的查询语言中，有三种作用：

1）在一段时间范围内对数据进行采样（通常是请求持续时间或响应大小等），并将其计入可配置的存储桶（`bucket`）中. 后续可通过指定区间筛选样本，也可以统计样本总数，最后一般将数据展示为直方图。

2）对每个采样点值累计和(`sum`)

3）对采样点的次数累计和(`count`)

**度量指标名称**: [basename]*上面三类的作用度量指标名称*

```bash
1）[basename]bucket{le="上边界"}, 这个值为小于等于上边界的所有采样点数量
2）[basename]_sum_
3）[basename]_count
```

小结：如果定义一个度量类型为Histogram，则Prometheus会自动生成三个对应的指标

#### 为什需要用histogram柱状图

 在大多数情况下人们都倾向于使用某些量化指标的平均值，例如 CPU 的平均使用率、页面的平均响应时间。这种方式的问题很明显，以系统 API 调用的平均响应时间为例：**如果大多数 API 请求都维持在 100ms 的响应时间范围内，而个别请求的响应时间需要 5s，那么就会导致某些 WEB 页面的响应时间落到中位数的情况，而这种现象被称为`长尾问题`**。
​ 为了区分是平均的慢还是长尾的慢，最简单的方式就是按照请求延迟的范围进行分组。例如，**统计延迟在 0~10ms 之间的请求数有多少，而 10~20ms 之间的请求数又有多少。通过这种方式可以快速分析系统慢的原因**。Histogram 和 Summary 都是为了能够解决这样问题的存在，通过 Histogram 和 Summary 类型的监控指标，我们可以快速了解监控样本的分布情况。

Histogram 类型的样本会提供三种指标（假设指标名称为 `<basename>`）：

1）样本的值分布在 bucket 中的数量，命名为 `<basename>_bucket{le="<上边界>"}`。解释的更通俗易懂一点，这个值表示指标值小于等于上边界的所有样本数量。

```bash
# 1、在总共2次请求当中。http 请求响应时间 <=0.005 秒 的请求次数为0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.005",} 0.0

# 2、在总共2次请求当中。http 请求响应时间 <=0.01 秒 的请求次数为0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.01",} 0.0

# 3、在总共2次请求当中。http 请求响应时间 <=0.025 秒 的请求次数为0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.025",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.05",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.075",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.1",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.25",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.5",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.75",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="1.0",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="2.5",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="5.0",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="7.5",} 2.0

# 4、在总共2次请求当中。http 请求响应时间 <=10 秒 的请求次数为 2
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="10.0",} 2.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="+Inf",} 2.0
```

2）所有样本值的大小总和，命名为 `<basename>_sum`

```bash
# 实际含义： 发生的2次 http 请求总的响应时间为 13.107670803000001 秒
io_namespace_http_requests_latency_seconds_histogram_sum{path="/",method="GET",code="200",} 13.107670803000001
```

3）样本总数，命名为 `<basename>_count`，值和 `<basename>_bucket{le="+Inf"}` 相同

```bash
# 实际含义： 当前一共发生了 2 次 http 请求
io_namespace_http_requests_latency_seconds_histogram_count{path="/",method="GET",code="200",} 2.0
```

**注意**：
1）bucket 可以理解为是对数据指标值域的一个划分，划分的依据应该基于数据值的分布。注意后面的采样点是包含前面的采样点的，假设 `xxx_bucket{...,le="0.01"}` 的值为 10，而 `xxx_bucket{...,le="0.05"}` 的值为 30，那么意味着这 30 个采样点中，有 10 个是小于 0.01s的，其余 20 个采样点的响应时间是介于0.01s 和 0.05s之间的。

2）可以通过 `histogram_quantile()` 函数来计算 Histogram 类型样本的分位数。分位数可能不太好理解，你可以理解为分割数据的点。我举个例子，假设样本的 9 分位数**（quantile=0.9）的值为 x，即表示小于 x 的采样值的数量占总体采样值的 90%**。Histogram 还可以用来计算应用性能指标值（Apdex score）。

### Summary

 与 Histogram 类型类似，用于表示一段时间内的数据采样结果（通常是请求持续时间或响应大小等），但它直接存储了分位数（通过客户端计算，然后展示出来），而不是通过区间来计算。它也有三种作用：

1）对于每个采样点进行统计，并形成分位图。（如：正态分布一样，统计低于60分不及格的同学比例，统计低于80分的同学比例，统计低于95分的同学比例）

2）统计班上所有同学的总成绩(sum)

3）统计班上同学的考试总人数(count)

带有度量指标的[basename]的summary 在抓取时间序列数据有如命名。

1、观察时间的`φ-quantiles (0 ≤ φ ≤ 1)`, 显示为`[basename]{分位数="[φ]"}`

2、`[basename]_sum`， 是指所有观察值的总和_

3、`[basename]_count`, 是指已观察到的事件计数值

样本值的分位数分布情况，命名为 `<basename>{quantile="<φ>"}`。

```bash
# 1、含义：这 12 次 http 请求中有 50% 的请求响应时间是 3.052404983s
io_namespace_http_requests_latency_seconds_summary{path="/",method="GET",code="200",quantile="0.5",} 3.052404983

# 2、含义：这 12 次 http 请求中有 90% 的请求响应时间是 8.003261666s
io_namespace_http_requests_latency_seconds_summary{path="/",method="GET",code="200",quantile="0.9",} 8.003261666
```

所有样本值的大小总和，命名为 `<basename>_sum`。

```bash
# 1、含义：这12次 http 请求的总响应时间为 51.029495508s
io_namespace_http_requests_latency_seconds_summary_sum{path="/",method="GET",code="200",} 51.029495508
```

样本总数，命名为 `<basename>_count`。

```bash
# 1、含义：当前一共发生了 12 次 http 请求
io_namespace_http_requests_latency_seconds_summary_count{path="/",method="GET",code="200",} 12.0
```

**Histogram 与 Summary 的异同**：

它们都包含了 `<basename>_sum` 和 `<basename>_count` 指标，Histogram 需要通过 `<basename>_bucket` 来计算分位数，而 Summary 则直接存储了分位数的值。

```bash
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.5"} 0.012352463
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.9"} 0.014458005
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.99"} 0.017316173
prometheus_tsdb_wal_fsync_duration_seconds_sum 2.888716127000002
prometheus_tsdb_wal_fsync_duration_seconds_count 216

# 从上面的样本中可以得知当前Promtheus Server进行wal_fsync操作的总次数为216次，耗时2.888716127000002s。其中中位数（quantile=0.5）的耗时为0.012352463，9分位数（quantile=0.9）的耗时为0.014458005s。
```

## Prometheus能监控什么

```
# Databases---数据库
    Aerospike exporter
    ClickHouse exporter
    Consul exporter (official)
    Couchbase exporter
    CouchDB exporter
    ElasticSearch exporter
    EventStore exporter
    Memcached exporter (official)
    MongoDB exporter
    MSSQL server exporter
    MySQL server exporter (official)
    OpenTSDB Exporter
    Oracle DB Exporter
    PgBouncer exporter
    PostgreSQL exporter
    ProxySQL exporter
    RavenDB exporter
    Redis exporter
    RethinkDB exporter
    SQL exporter
    Tarantool metric library
    Twemproxy
# Hardware related---硬件相关
    apcupsd exporter
    Collins exporter
    IBM Z HMC exporter
    IoT Edison exporter
    IPMI exporter
    knxd exporter
    Netgear Cable Modem Exporter
    Node/system metrics exporter (official)
    NVIDIA GPU exporter
    ProSAFE exporter
    Ubiquiti UniFi exporter
# Messaging systems---消息服务
    Beanstalkd exporter
    Gearman exporter
    Kafka exporter
    NATS exporter
    NSQ exporter
    Mirth Connect exporter
    MQTT blackbox exporter
    RabbitMQ exporter
    RabbitMQ Management Plugin exporter
# Storage---存储
    Ceph exporter
    Ceph RADOSGW exporter
    Gluster exporter
    Hadoop HDFS FSImage exporter
    Lustre exporter
    ScaleIO exporter
# HTTP---网站服务
    Apache exporter
    HAProxy exporter (official)
    Nginx metric library
    Nginx VTS exporter
    Passenger exporter
    Squid exporter
    Tinyproxy exporter
    Varnish exporter
    WebDriver exporter
# APIs
    AWS ECS exporter
    AWS Health exporter
    AWS SQS exporter
    Cloudflare exporter
    DigitalOcean exporter
    Docker Cloud exporter
    Docker Hub exporter
    GitHub exporter
    InstaClustr exporter
    Mozilla Observatory exporter
    OpenWeatherMap exporter
    Pagespeed exporter
    Rancher exporter
    Speedtest exporter
# Logging---日志
    Fluentd exporter
    Google's mtail log data extractor
    Grok exporter
# Other monitoring systems
    Akamai Cloudmonitor exporter
    Alibaba Cloudmonitor exporter
    AWS CloudWatch exporter (official)
    Cloud Foundry Firehose exporter
    Collectd exporter (official)
    Google Stackdriver exporter
    Graphite exporter (official)
    Heka dashboard exporter
    Heka exporter
    InfluxDB exporter (official)
    JavaMelody exporter
    JMX exporter (official)
    Munin exporter
    Nagios / Naemon exporter
    New Relic exporter
    NRPE exporter
    Osquery exporter
    OTC CloudEye exporter
    Pingdom exporter
    scollector exporter
    Sensu exporter
    SNMP exporter (official)
    StatsD exporter (official)
# Miscellaneous---其他
    ACT Fibernet Exporter
    Bamboo exporter
    BIG-IP exporter
    BIND exporter
    Bitbucket exporter
    Blackbox exporter (official)
    BOSH exporter
    cAdvisor
    Cachet exporter
    ccache exporter
    Confluence exporter
    Dovecot exporter
    eBPF exporter
    Ethereum Client exporter
    Jenkins exporter
    JIRA exporter
    Kannel exporter
    Kemp LoadBalancer exporter
    Kibana Exporter
    Meteor JS web framework exporter
    Minecraft exporter module
    PHP-FPM exporter
    PowerDNS exporter
    Presto exporter
    Process exporter
    rTorrent exporter
    SABnzbd exporter
    Script exporter
    Shield exporter
    SMTP/Maildir MDA blackbox prober
    SoftEther exporter
    Transmission exporter
    Unbound exporter
    Xen exporter
# Software exposing Prometheus metrics---Prometheus度量指标
    App Connect Enterprise
    Ballerina
    Ceph
    Collectd
    Concourse
    CRG Roller Derby Scoreboard (direct)
    Docker Daemon
    Doorman (direct)
    Etcd (direct)
    Flink
    FreeBSD Kernel
    Grafana
    JavaMelody
    Kubernetes (direct)
    Linkerd
```

## Prometheus对kubernetes的监控

对于Kubernetes而言，我们可以把当中所有的资源分为几类：

- **基础设施层（Node）**：集群节点，为整个集群和应用提供运行时资源
- **容器基础设施（Container）**：为应用提供运行时环境
- **用户应用（Pod）**：Pod中会包含一组容器，它们一起工作，并且对外提供一个（或者一组）功能
- **内部服务负载均衡（Service）**：在集群内，通过Service在集群暴露应用功能，集群内应用和应用之间访问时提供内部的负载均衡
- **外部访问入口（Ingress）**：通过Ingress提供集群外的访问入口，从而可以使外部客户端能够访问到部署在Kubernetes集群内的服务

因此，如果要构建一个完整的监控体系，我们应该考虑，以下5个方面：

- **集群节点状态监控**：从集群中各节点的kubelet服务获取节点的基本运行状态；
- **集群节点资源用量监控**：通过Daemonset的形式在集群中各个节点部署Node Exporter采集节点的资源使用情况；
- **节点中运行的容器监控**：通过各个节点中kubelet内置的cAdvisor中获取个节点中所有容器的运行状态和资源使用情况；
- 如果在集群中部署的应用程序本身内置了对Prometheus的监控支持，那么我们还应该找到相应的Pod实例，并从该Pod实例中获取其内部运行状态的监控指标。
- **对k8s本身的组件做监控**：apiserver、scheduler、controller-manager、kubelet、kube-proxy

## 配置

### Prometheus配置

prometheus的配置文件`prometheus.yml`，它主要分以下几个配置块：

```
全局配置        global

告警配置        alerting

规则文件配置    rule_files

拉取配置        scrape_configs

远程读写配置    remote_read、remote_write
```

全局配置 `global`

`global`指定在所有其他配置上下文中有效的参数。还可用作其他配置部分的默认设置。

```
global:
  # 默认拉取频率
  [ scrape_interval: <duration> | default = 1m ]

  # 拉取超时时间
  [ scrape_timeout: <duration> | default = 10s ]

  # 执行规则频率
  [ evaluation_interval: <duration> | default = 1m ]

  # 通信时添加到任何时间序列或告警的标签
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    [ <labelname>: <labelvalue> ... ]

  # 记录PromQL查询的日志文件
  [ query_log_file: <string> ]

```

告警配置 `alerting`

`alerting`指定与Alertmanager相关的设置。

```
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]
```

- 规则文件配置 `rule_files`：

`rule_files`指定prometheus加载的任何规则的位置，从所有匹配的文件中读取规则和告警。目前没有规则。

```
rule_files:
  [ - <filepath_glob> ... ]
```

- 拉取配置 `scrape_configs`：

scrape_configs指定prometheus监控哪些资源。默认会拉取prometheus本身的时间序列数据，通过http://localhost:9090/metrics进行拉取。

一个scrape_config指定一组目标和参数，描述如何拉取它们。在一般情况下，一个拉取配置指定一个作业。在高级配置中，这可能会改变。

可以通过static_configs参数静态配置目标，也可以使用支持的服务发现机制之一动态发现目标。

此外，relabel_configs在拉取之前，可以对任何目标及其标签进行修改。


```
scrape_configs:
job_name: <job_name>

# 拉取频率
[ scrape_interval: <duration> | default = <global_config.scrape_interval> ]

# 拉取超时时间
[ scrape_timeout: <duration> | default = <global_config.scrape_timeout> ]

# 拉取的http路径
[ metrics_path: <path> | default = /metrics ]

# honor_labels 控制prometheus处理已存在于收集数据中的标签与prometheus将附加在服务器端的标签("作业"和"实例"标签、手动配置的目标标签和由服务发现实现生成的标签)之间的冲突
# 如果 honor_labels 设置为 "true"，则通过保持从拉取数据获得的标签值并忽略冲突的服务器端标签来解决标签冲突
# 如果 honor_labels 设置为 "false"，则通过将拉取数据中冲突的标签重命名为"exported_<original-label>"来解决标签冲突(例如"exported_instance"、"exported_job")，然后附加服务器端标签
# 注意，任何全局配置的 "external_labels"都不受此设置的影响。在与外部系统的通信中，只有当时间序列还没有给定的标签时，它们才被应用，否则就会被忽略
[ honor_labels: <boolean> | default = false ]

# honor_timestamps 控制prometheus是否遵守拉取数据中的时间戳
# 如果 honor_timestamps 设置为 "true"，将使用目标公开的metrics的时间戳
# 如果 honor_timestamps 设置为 "false"，目标公开的metrics的时间戳将被忽略
[ honor_timestamps: <boolean> | default = true ]

# 配置用于请求的协议
[ scheme: <scheme> | default = http ]

# 可选的http url参数
params:
  [ <string>: [<string>, ...] ]

# 在每个拉取请求上配置 username 和 password 来设置 Authorization 头部，password 和 password_file 二选一
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# 在每个拉取请求上配置 bearer token 来设置 Authorization 头部，bearer_token 和 bearer_token_file 二选一
[ bearer_token: <secret> ]

# 在每个拉取请求上配置 bearer_token_file 来设置 Authorization 头部，bearer_token_file 和 bearer_token 二选一
[ bearer_token_file: /path/to/bearer/token/file ]

# 配置拉取请求的TLS设置
tls_config:
  [ <tls_config> ]

# 可选的代理URL
[ proxy_url: <string> ]

# Azure服务发现配置列表
azure_sd_configs:
  [ - <azure_sd_config> ... ]

# Consul服务发现配置列表
consul_sd_configs:
  [ - <consul_sd_config> ... ]

# DNS服务发现配置列表
dns_sd_configs:
  [ - <dns_sd_config> ... ]

# EC2服务发现配置列表
ec2_sd_configs:
  [ - <ec2_sd_config> ... ]

# OpenStack服务发现配置列表
openstack_sd_configs:
  [ - <openstack_sd_config> ... ]

# file服务发现配置列表
file_sd_configs:
  [ - <file_sd_config> ... ]

# GCE服务发现配置列表
gce_sd_configs:
  [ - <gce_sd_config> ... ]

# Kubernetes服务发现配置列表
kubernetes_sd_configs:
  [ - <kubernetes_sd_config> ... ]

# Marathon服务发现配置列表
marathon_sd_configs:
  [ - <marathon_sd_config> ... ]

# AirBnB's Nerve服务发现配置列表
nerve_sd_configs:
  [ - <nerve_sd_config> ... ]

# Zookeeper Serverset服务发现配置列表
serverset_sd_configs:
  [ - <serverset_sd_config> ... ]

# Triton服务发现配置列表
triton_sd_configs:
  [ - <triton_sd_config> ... ]

# 静态配置目标列表
static_configs:
  [ - <static_config> ... ]

# 目标relabel配置列表
relabel_configs:
  [ - <relabel_config> ... ]

# metric relabel配置列表
metric_relabel_configs:
  [ - <relabel_config> ... ]

# 每次拉取样品的数量限制
# metric relabelling之后，如果有超过这个数量的样品，整个拉取将被视为失效。0表示没有限制
[ sample_limit: <int> | default = 0 ]
```

- 远程读写配置 `remote_read`/`remote_write`：

`remote_read`/`remote_write`将数据源与prometheus分离，当前不做配置。

```
# 与远程写功能相关的设置
remote_write:
  [ - <remote_write> ... ]

# 与远程读功能相关的设置
remote_read:
  [ - <remote_read> ... ]
```

- 简单配置示例：

```
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
```

### AlertManager配置

alertmanager通过命令行标志和配置文件进行配置。命令行标志配置不可变的系统参数时，配置文件定义禁止规则，通知路由和通知接收器。

alertmanager的配置文件`alertmanager.yml`，它主要分以下几个配置块：

```
全局配置        global

通知模板        templates

路由配置        route

接收器配置      receivers

抑制配置        inhibit_rules
```

全局配置 `global`

`global`指定在所有其他配置上下文中有效的参数。还用作其他配置部分的默认设置。

```
global:
  # 默认的SMTP头字段
  [ smtp_from: <tmpl_string> ]
  
  # 默认的SMTP smarthost用于发送电子邮件，包括端口号
  # 端口号通常是25，对于TLS上的SMTP，端口号为587
  # Example: smtp.example.org:587
  [ smtp_smarthost: <string> ]
  
  # 要标识给SMTP服务器的默认主机名
  [ smtp_hello: <string> | default = "localhost" ]
  
  # SMTP认证使用CRAM-MD5，登录和普通。如果为空，Alertmanager不会对SMTP服务器进行身份验证
  [ smtp_auth_username: <string> ]
  
  # SMTP Auth using LOGIN and PLAIN.
  [ smtp_auth_password: <secret> ]
  
  # SMTP Auth using PLAIN.
  [ smtp_auth_identity: <string> ]
  
  # SMTP Auth using CRAM-MD5.
  [ smtp_auth_secret: <secret> ]
  
  # 默认的SMTP TLS要求
  # 注意，Go不支持到远程SMTP端点的未加密连接
  [ smtp_require_tls: <bool> | default = true ]

  # 用于Slack通知的API URL
  [ slack_api_url: <secret> ]
  [ victorops_api_key: <secret> ]
  [ victorops_api_url: <string> | default = "https://alert.victorops.com/integrations/generic/20131114/alert/" ]
  [ pagerduty_url: <string> | default = "https://events.pagerduty.com/v2/enqueue" ]
  [ opsgenie_api_key: <secret> ]
  [ opsgenie_api_url: <string> | default = "https://api.opsgenie.com/" ]
  [ wechat_api_url: <string> | default = "https://qyapi.weixin.qq.com/cgi-bin/" ]
  [ wechat_api_secret: <secret> ]
  [ wechat_api_corp_id: <string> ]

  # 默认HTTP客户端配置
  [ http_config: <http_config> ]

  # 如果告警不包括EndsAt，则ResolveTimeout是alertmanager使用的默认值，在此时间过后，如果告警没有更新，则可以声明警报已解除
  # 这对Prometheus的告警没有影响，它们包括EndsAt
  [ resolve_timeout: <duration> | default = 5m ]

```

- 通知模板 `templates`：

`templates`指定了从其中读取自定义通知模板定义的文件，最后一个文件可以使用一个通配符匹配器，如`templates/*.tmpl`。

```
templates:
  [ - <filepath> ... ]
```

- 路由配置 `route`：

route定义了路由树中的节点及其子节点。如果未设置，则其可选配置参数将从其父节点继承。

每个告警都会在已配置的顶级路由处进入路由树，该路由树必须与所有告警匹配（即没有任何已配置的匹配器），然后它会遍历子节点。如果continue设置为false，它将在第一个匹配的子项之后停止；如果continue设置为true，则告警将继续与后续的同级进行匹配。如果告警与节点的任何子节点都不匹配（不匹配的子节点或不存在子节点），则根据当前节点的配置参数来处理告警。

```
route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'
```

- 接收器配置 `receivers`：

`receivers`是一个或多个通知集成的命名配置。建议通过webhook接收器实现自定义通知集成。

```
receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://127.0.0.1:5001/'
```

- 抑制规则配置 `inhibit_rules`：

当存在与另一组匹配器匹配的告警（源）时，抑制规则会使与一组匹配器匹配的告警（目标）“静音”。目标和源告警的equal列表中的标签名称都必须具有相同的标签值。

在语义上，缺少标签和带有空值的标签是相同的。因此，如果equal源告警和目标告警都缺少列出的所有标签名称，则将应用抑制规则。

```
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```

- 默认配置示例：

```
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'
receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://127.0.0.1:5001/'
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```

## Grafana部署

grafana 是一款采用 go 语言编写的开源应用，主要用于大规模指标数据的可视化展现，是网络架构和应用分析中最流行的时序数据展示工具，目前已经支持绝大部分常用的时序数据库。

官网：https://grafana.com

# prometheus监控k8s中微服务JVM与服务的自动发现

需求：业务方需要看到每个服务实例的JVM资源使用情况

难点：

每个服务实例在k8s中都是一个pod且分布在不同的namespace中数量成百上千
prometheus监控需要服务提供metrics接口
prometheus需要在配置文件中添加每个实例的metrics地址，因为pod的ip一直在变，所以配置文件写死了无法完成，需要配置自动发现
解决方案：

搭建集成了Micrometer功能的Spring BootJVM监控，并配置Grafana监控大盘
配置prometheus自动发现
集成actuator与micrometer

```
        <!--监控系统健康情况的工具-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
 
		<!--桥接Prometheus-->
		<dependency>
			<groupId>io.micrometer</groupId>
			<artifactId>micrometer-registry-prometheus</artifactId>
			<version>1.6.0</version>
		</dependency>
 
		<!--micrometer核心包, 按需引入, 使用Meter注解或手动埋点时需要-->
		<dependency>
			<groupId>io.micrometer</groupId>
			<artifactId>micrometer-core</artifactId>
			<version>1.6.0</version>
		</dependency>
		<!--micrometer获取JVM相关信息, 并展示在Grafana上-->
		<dependency>
			<groupId>io.github.mweirauch</groupId>
			<artifactId>micrometer-jvm-extras</artifactId>
			<version>0.2.0</version>
		</dependency>
```

因为spring actuator因为安全原因默认只开启health和info接口，所以我们需要修改下application.yml，将prometheus接口放开

```

# metircs
management:
  endpoints:
    web:
      exposure:
        include: prometheus, health
  metrics:
    export:
      simple:
        enabled: false
    tags:
      application: ${spring.application.name}
```

## 配置prometheus自动发现

```
- job_name: spring metrics
  honor_timestamps: true
  scrape_interval: 10s
  scrape_timeout: 10s
  metrics_path: actuator/prometheus
  scheme: http
  relabel_configs:
  - source_labels: [__meta_kubernetes_namespace]
    separator: ;
    regex: online
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_pod_label_app]
    separator: ;
    regex: (.*(jar|tomcat).*)
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_pod_label_app]
    separator: ;
    regex: (.+)
    target_label: monitor_port
    replacement: "9876"
    action: replace
  - source_labels: [__meta_kubernetes_pod_label_app]
    separator: ;
    regex: ^(clms|account|ofs|tia|cmcs|rcc|uip)(?:-front)?-tomcat.*
    target_label: context_path
    replacement: /${1}
    action: replace
  - source_labels: [context_path]
    separator: ;
    regex: (.+)
    target_label: monitor_port
    replacement: "8080"
    action: replace
  - source_labels: [context_path, __metrics_path__]
    separator: /
    regex: (.+)
    target_label: __metrics_path__
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_ip, monitor_port]
    separator: ':'
    regex: (.+)
    target_label: __address__
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_namespace]
    separator: ;
    regex: (.*)
    target_label: spring_namespace
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_label_app]
    separator: ;
    regex: (.*)
    target_label: application
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_name]
    separator: ;
    regex: (.*)
    target_label: instance
    replacement: $1
    action: replace
  kubernetes_sd_configs:
  - role: pod
```

# 报警规则详解

这篇文章介绍prometheus和alertmanager的报警和通知规则，prometheus的配置文件名为prometheus.yml，alertmanager的配置文件名为alertmanager.yml

报警：指prometheus将监测到的异常事件发送给alertmanager，而不是指发送邮件通知
通知：指alertmanager发送异常事件的通知（邮件、[webhook](https://so.csdn.net/so/search?q=webhook&spm=1001.2101.3001.7020)等）

## 报警规则

在prometheus.yml中指定匹配报警规则的间隔

```php
# How frequently to evaluate rules.



[ evaluation_interval: <duration> | default = 1m ]
```

在prometheus.yml中指定规则文件（可使用通配符，如rules/*.rules）

```python
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.



rule_files:



 - "/etc/prometheus/alert.rules"
```

并基于以下模板：

```xml
ALERT <alert name>



  IF <expression>



  [ FOR <duration> ]



  [ LABELS <label set> ]



  [ ANNOTATIONS <label set> ]
```

其中：

Alert name是警报标识符。它不需要是唯一的。

Expression是为了触发警报而被评估的条件。它通常使用现有指标作为/metrics端点返回的指标。

Duration是规则必须有效的时间段。例如，5s表示5秒。

Label set是将在消息模板中使用的一组标签。

在prometheus-k8s-statefulset.yaml 文件创建ruleSelector，标记报警规则角色。在prometheus-k8s-rules.yaml 报警规则文件中引用

```undefined
  ruleSelector:



    matchLabels:



      role: prometheus-rulefiles



      prometheus: k8s
```

在prometheus-k8s-rules.yaml 使用configmap 方式引用prometheus-rulefiles

```bash
apiVersion: v1



kind: ConfigMap



metadata:



  name: prometheus-k8s-rules



  namespace: monitoring



  labels:



    role: prometheus-rulefiles



    prometheus: k8s



data:



  pod.rules.yaml: |+



    groups:



    - name: noah_pod.rules



      rules:



      - alert: Pod_all_cpu_usage



        expr: (sum by(name)(rate(container_cpu_usage_seconds_total{image!=""}[5m]))*100) > 10



        for: 5m



        labels:



          severity: critical



          service: pods



        annotations:



          description: 容器 {{ $labels.name }} CPU 资源利用率大于 75% , (current value is {{ $value }})



          summary: Dev CPU 负载告警



      - alert: Pod_all_memory_usage



        expr: sort_desc(avg by(name)(irate(container_memory_usage_bytes{name!=""}[5m]))*100) > 1024*10^3*2



        for: 10m



        labels:



          severity: critical



        annotations:



          description: 容器 {{ $labels.name }} Memory 资源利用率大于 2G , (current value is {{ $value }})



          summary: Dev Memory 负载告警



      - alert: Pod_all_network_receive_usage



        expr: sum by (name)(irate(container_network_receive_bytes_total{container_name="POD"}[1m])) > 1024*1024*50



        for: 10m



        labels:



          severity: critical



        annotations:



          description: 容器 {{ $labels.name }} network_receive 资源利用率大于 50M , (current value is {{ $value }})



          summary: network_receive 负载告警
```

配置文件设置好后，prometheus-opeartor自动重新读取配置。
如果二次修改comfigmap 内容只需要apply

```undefined
kubectl apply -f prometheus-k8s-rules.yaml
```

将邮件通知与rules对比一下（还需要配置alertmanager.yml才能收到邮件）

## 通知规则

设置alertmanager.yml的的route与receivers

```php
global:



  # ResolveTimeout is the time after which an alert is declared resolved



  # if it has not been updated.



  resolve_timeout: 5m



 



  # The smarthost and SMTP sender used for mail notifications.



  smtp_smarthost: 'xxxxx'



  smtp_from: 'xxxxxxx'



  smtp_auth_username: 'xxxxx'



  smtp_auth_password: 'xxxxxx'



  # The API URL to use for Slack notifications.



  slack_api_url: 'https://hooks.slack.com/services/some/api/token'



 



# # The directory from which notification templates are read.



templates:



- '*.tmpl'



 



# The root route on which each incoming alert enters.



route:



 



  # The labels by which incoming alerts are grouped together. For example,



  # multiple alerts coming in for cluster=A and alertname=LatencyHigh would



  # be batched into a single group.



 



  group_by: ['alertname', 'cluster', 'service']



 



  # When a new group of alerts is created by an incoming alert, wait at



  # least 'group_wait' to send the initial notification.



  # This way ensures that you get multiple alerts for the same group that start



  # firing shortly after another are batched together on the first



  # notification.



 



  group_wait: 30s



 



  # When the first notification was sent, wait 'group_interval' to send a batch



  # of new alerts that started firing for that group.



 



  group_interval: 5m



 



  # If an alert has successfully been sent, wait 'repeat_interval' to



  # resend them.



 



  #repeat_interval: 1m



  repeat_interval: 15m



 



  # A default receiver



 



  # If an alert isn't caught by a route, send it to default.



  receiver: default



 



  # All the above attributes are inherited by all child routes and can



  # overwritten on each.



 



  # The child route trees.



  routes:



  - match:



      severity: critical



    receiver: email_alert



 



receivers:



- name: 'default'



  email_configs:



  - to : 'yi.hu@dianrong.com'



    send_resolved: true



 



- name: 'email_alert'



  email_configs:



  - to : 'yi.hu@dianrong.com'



    send_resolved: true
```

### 名词解释

### Route

`route`属性用来设置报警的分发策略，它是一个树状结构，按照深度优先从左向右的顺序进行匹配。

```go
// Match does a depth-first left-to-right search through the route tree



// and returns the matching routing nodes.



func (r *Route) Match(lset model.LabelSet) []*Route {
```

### Alert

`Alert`是alertmanager接收到的报警，类型如下。

```go
// Alert is a generic representation of an alert in the Prometheus eco-system.



type Alert struct {



    // Label value pairs for purpose of aggregation, matching, and disposition



    // dispatching. This must minimally include an "alertname" label.



    Labels LabelSet `json:"labels"`



 



    // Extra key/value information which does not define alert identity.



    Annotations LabelSet `json:"annotations"`



 



    // The known time range for this alert. Both ends are optional.



    StartsAt     time.Time `json:"startsAt,omitempty"`



    EndsAt       time.Time `json:"endsAt,omitempty"`



    GeneratorURL string    `json:"generatorURL"`



}
```

> 具有相同Lables的Alert（key和value都相同）才会被认为是同一种。在prometheus rules文件配置的一条规则可能会产生多种报警

### Group

alertmanager会根据group_by配置将Alert分组。如下规则，当go_goroutines等于4时会收到三条报警，alertmanager会将这三条报警分成两组向receivers发出通知。

```puppet
ALERT test1



  IF go_goroutines > 1



  LABELS {label1="l1", label2="l2", status="test"}



ALERT test2



  IF go_goroutines > 2



  LABELS {label1="l2", label2="l2", status="test"}



ALERT test3



  IF go_goroutines > 3



  LABELS {label1="l2", label2="l1", status="test"}
```

### 主要处理流程

1. 接收到Alert，根据labels判断属于哪些Route（可存在多个Route，一个Route有多个Group，一个Group有多个Alert）
2. 将Alert分配到Group中，没有则新建Group
3. 新的Group等待group_wait指定的时间（等待时可能收到同一Group的Alert），根据resolve_timeout判断Alert是否解决，然后发送通知
4. 已有的Group等待group_interval指定的时间，判断Alert是否解决，当上次发送通知到现在的间隔大于repeat_interval或者Group有更新时会发送通知

## Alertmanager

Alertmanager是警报的缓冲区，它具有以下特征：

可以通过特定端点（不是特定于Prometheus）接收警报。

可以将警报重定向到接收者，如hipchat、邮件或其他人。

足够智能，可以确定已经发送了类似的通知。所以，如果出现问题，你不会被成千上万的电子邮件淹没。

Alertmanager客户端（在这种情况下是Prometheus）首先发送POST消息，并将所有要处理的警报发送到/ api / v1 / alerts。例如：

```json
[



 {



  "labels": {



     "alertname": "low_connected_users",



     "severity": "warning"



   },



   "annotations": {



      "description": "Instance play-app:9000 under lower load",



      "summary": "play-app:9000 of job playframework-app is under lower load"



    }



 }]
```

### alert工作流程

一旦这些警报存储在Alertmanager，它们可能处于以下任何状态：

![alert 报警流程](https://box.kancloud.cn/1aa4400491ac3e8202fad57c6859d622_215x300.png)

- Inactive：这里什么都没有发生。
- Pending：客户端告诉我们这个警报必须被触发。然而，警报可以被分组、压抑/抑制或者静默/静音。一旦所有的验证都通过了，我们就转到Firing。
- Firing：警报发送到Notification Pipeline，它将联系警报的所有接收者。然后客户端告诉我们警报解除，所以转换到状Inactive状态。

Prometheus有一个专门的端点，允许我们列出所有的警报，并遵循状态转换。Prometheus所示的每个状态以及导致过渡的条件如下所示：

规则不符合。警报没有激活。

![img](https://box.kancloud.cn/0f044c1fb0e4bf236269249a83b65787_634x223.png)

规则符合。警报现在处于活动状态。 执行一些验证是为了避免淹没接收器的消息。

![img](https://box.kancloud.cn/cb47caff504e873ace45116b0dded829_1874x625.png)

警报发送到接收者

![img](https://box.kancloud.cn/8d8c30577028613304be57b4750cae67_1895x446.png)

###  

接收器 receiver

顾名思义，警报接收的配置。
通用配置格式

\# The unique name of the receiver.
name: <string>

\# Configurations for several notification integrations.
email_configs:
 [ - <email_config>, ... ]
pagerduty_configs:
 [ - <pagerduty_config>, ... ]
slack_config:
 [ - <slack_config>, ... ]
opsgenie_configs:
 [ - <opsgenie_config>, ... ]
webhook_configs:
 [ - <webhook_config>, ... ]

邮件接收器 email_config

\# Whether or not to notify about resolved alerts.
[ send_resolved: <boolean> | default = false ]

\# The email address to send notifications to.
to: <tmpl_string>
\# The sender address.
[ from: <tmpl_string> | default = global.smtp_from ]
\# The SMTP host through which emails are sent.
[ smarthost: <string> | default = global.smtp_smarthost ]

\# The HTML body of the email notification.
[ html: <tmpl_string> | default = '{undefined{ template "email.default.html" . }}' ]

\# Further headers email header key/value pairs. Overrides any headers
\# previously set by the notification implementation.
[ headers: { <string>: <tmpl_string>, ... } ]


Slack接收器 slack_config

\# Whether or not to notify about resolved alerts.
[ send_resolved: <boolean> | default = true ]

\# The Slack webhook URL.
[ api_url: <string> | default = global.slack_api_url ]

\# The channel or user to send notifications to.
channel: <tmpl_string>

\# API request data as defined by the Slack webhook API.
[ color: <tmpl_string> | default = '{undefined{ if eq .Status "firing" }}danger{undefined{ else }}good{undefined{ end }}' ]
[ username: <tmpl_string> | default = '{undefined{ template "slack.default.username" . }}'
[ title: <tmpl_string> | default = '{undefined{ template "slack.default.title" . }}' ]
[ title_link: <tmpl_string> | default = '{undefined{ template "slack.default.titlelink" . }}' ]
[ pretext: <tmpl_string> | default = '{undefined{ template "slack.default.pretext" . }}' ]
[ text: <tmpl_string> | default = '{undefined{ template "slack.default.text" . }}' ]
[ fallback: <tmpl_string> | default = '{undefined{ template "slack.default.fallback" . }}' ]

Webhook接收器 webhook_config

\# Whether or not to notify about resolved alerts.
[ send_resolved: <boolean> | default = true ]

\# The endpoint to send HTTP POST requests to.
url: <string>

Alertmanager会使用以下的格式向配置端点发送HTTP POST请求：

{undefined
 "version": "2",
 "status": "<resolved|firing>",
 "alerts": [
  {undefined
   "labels": <object>,
   "annotations": <object>,
   "startsAt": "<rfc3339>",
   "endsAt": "<rfc3339>"
  },
  ...
 ]
}



### Inhibition

抑制是指当警报发出后，停止重复发送由此警报引发其他错误的警报的机制。

例如，当警报被触发，通知整个集群不可达，可以配置Alertmanager忽略由该警报触发而产生的所有其他警报，这可以防止通知数百或数千与此问题不相关的其他警报。
抑制机制可以通过Alertmanager的配置文件来配置。

Inhibition允许在其他警报处于触发状态时，抑制一些警报的通知。例如，如果同一警报（基于警报名称）已经非常紧急，那么我们可以配置一个抑制来使任何警告级别的通知静音。 alertmanager.yml文件的相关部分如下所示：

```vbnet
inhibit_rules:- source_match:



    severity: 'critical'



  target_match:



    severity: 'warning'



  equal: ['low_connected_users']
```

配置抑制规则，是存在另一组匹配器匹配的情况下，静音其他被引发警报的规则。这两个警报，必须有一组相同的标签。

```python
# Matchers that have to be fulfilled in the alerts to be muted.



target_match:



  [ <labelname>: <labelvalue>, ... ]



target_match_re:



  [ <labelname>: <regex>, ... ]



 



# Matchers for which one or more alerts have to exist for the



# inhibition to take effect.



source_match:



  [ <labelname>: <labelvalue>, ... ]



source_match_re:



  [ <labelname>: <regex>, ... ]



 



# Labels that must have an equal value in the source and target



# alert for the inhibition to take effect.



[ equal: '[' <labelname>, ... ']' ]
```

### Silences

Silences是快速地使警报暂时静音的一种方法。 我们直接通过Alertmanager管理控制台中的专用页面来配置它们。在尝试解决严重的生产问题时，这对避免收到垃圾邮件很有用。

![img](https://box.kancloud.cn/827627fe7d1652bda41c0e688a3a091f_1187x830.png)
[alertmanager 参考资料](https://mp.weixin.qq.com/s/eqgfd5_D0aH8dOGWUddEjg)
[抑制规则 inhibit_rule参考资料](http://blog.csdn.net/y_xiao_/article/details/50818451)


https://www.kancloud.cn/huyipow/prometheus/527563

转载于:https://www.cnblogs.com/yx88/p/11555431.html

**prometheus 告警指标**

https://blog.51cto.com/shoufu/2561993

# **常用prometheus告警规则模板（三）**



### 应用类相关

1.监控应用是否可用

规则模板 :

```
up=${value} 
```

规则描述:

监测应用是否可用

参数说明:

```
value : 0表示宕机  1 表示可用
```

具体应用

```
groups:
- name: example   #报警规则组的名字
  rules:
  - alert: InstanceDown     #检测job的状态，持续1分钟metrices不能访问会发给altermanager进行报警
    expr: up == 0
    for: 1m    #持续时间 ， 表示持续一分钟获取不到信息，则触发报警
    labels:
      serverity: page   # 自定义标签
    annotations:
      summary: "Instance {{ $labels.instance }} down"     # 自定义摘要 
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than                1 minutes."   # 自定义具体描述
```

本文理出的规则模板主要用于告警规则中的 “expr” 表达式使用。

labels参数说明

```
env : 数据源（通常用于区分环境）
instance : 实例名称
job : 应用名
```

2.接口请求异常（job，method，uri）

规则模板 :

```
http_server_requests_seconds_count{exception!="None",job="${app}",method="${method}",uri="${uri}"}  > ${value} 
```

规则描述 :

请求接口的异常信息不为空， 使用的时候需要动态传入 app , method ,uri , value 这四个参数，然后设置规则。

参数详解：

```
tex app : 应用名 method : POST 或 GET uri : 接口地址 value ： 检测指标 ,取值为 int整数
```

labels参数说明

```
env : 数据源（通常用于区分环境）
exception : 异常信息
instance ： 实例名
job : 应用名
method ： POST 或GET
status ：http请求状态 200为成功
uri : 接口地址
```

3.接口请求异常（job，method，uri），正则表达式(job,uri)

规则模板:

```
http_server_requests_seconds_count{exception!="None",job=~"${app}",method="${method}",uri=~"${uri}"}  > ${value} 
```

规则描述 :

请求接口的异常信息不为空， 使用的时候需要动态传入 app , method ,uri , value 这四个参数 ，这四个参数中**Job和uri可以为正则表达式**，然后设置规则。

参数解释:

```
app : 应用名 ， 可使用正则表达式，例： .*MSG.* 
method : POST 或 GET ，需大写
uri : 接口地址 , 可使用正则表达式
value ： 检测指标 ,取值为 int整数
```

labels参数说明

```
env : 数据源（通常用于区分环境）
exception : 异常信息
instance ： 实例名
job : 应用名
method ： POST 或GET
status ：http请求状态 200为成功
uri : 接口地址
```

4.应用CPU占比

规则模板:

```
process_cpu_usage{job="${app}"} * 100 > ${value}
```

规则描述 :

监测应用使用的百分比 ， 此处仅需传入 app 名称，就可以监测某个应用了

参数解释 :

```
app : 应用名 
value ： 检测指标, 百分比
```

labels参数说明

```
env : 数据源（通常用于区分环境）
instance : 实例名称
job : 应用名
```

5.Hystrix接口调用熔断次数监控

规则模板:

```
increase(hystrix_errors_total{job="${app}"}[${timeRange}]) > ${value}
```

规则描述 :

监测在指定的时间范围内，应用调用其他接口被Hystrix熔断的次数，

参数解释:

```
app : 应用名
timeRange : 指定时间范围内的熔断次数，取值单位可以为  s (秒) , m(分钟) , h(小时) ,d(天)
value : 熔断次数，int整数
```

labels参数说明

```
env : 数据源（通常用于区分环境）
group : 我们通过fegin调用其他应用的应用名
instance : 实例名称
job : 应用名
key : 具体的类名以及调用的方法 例： AcsClient#checkUserLogin(String)
```

6.Hystrix接口调用失败次数监控

规则模板:

```
increase(hystrix_fallback_total{job="${app}"}[${timeRange}]) > ${value}
```

规则描述 :

监测在指定的时间范围内，应用调用其他接口failback的次数

参数解释:

```
app : 应用名
timeRange : 指定时间范围内的熔断次数，取值单位可以为  s (秒) , m(分钟) , h(小时) ,d(天)
value : failback次数，int整数
```

labels参数说明

```
env : 数据源（通常用于区分环境）
group : 我们通过fegin调用其他应用的应用名
instance : 实例名称
job : 应用名
key : 具体的类名以及调用的方法 例： AcsClient#checkUserLogin(String)
```

7.JVM堆内存使用率监控

规则模板

```
sum(jvm_memory_used_bytes{job="${app}", instance="${instance}", area="heap"})*100/sum(jvm_memory_max_bytes{job="${app}",instance="${instance}", area="heap"}) >${value}
```

规则描述

监测JVM的堆内存的使用率， 前提是一定要指定应用名和实例名，否则prometheus不知道监控的那个JVM，这里是以JVM为单位的

参数解释

```
app : 应用名
instance : 实例名，默认为 IP:PORT
value : 监控指标，int整数，百分比
```

8.JVM非堆内存使用率监控

规则模板

```
sum(jvm_memory_used_bytes{job="${app}", instance="${instance}", area="nonheap"})*100/sum(jvm_memory_max_bytes{job="${app}",instance="${instance}", area="nonheap"})  > ${value}
```

规则描述

监测JVM的非堆内存的使用率（也就是通常意义上的栈内存，JIT编译代码缓存，永久代（jdk1.8为元空间））， 前提是一定要指定应用名和实例名，否则prometheus不知道监控的那个JVM，这里是以JVM为单位的

参数解释

```
app : 应用名
instance : 实例名，默认为 IP:PORT
value : 监控指标，int整数，百分比
```

9.接口某个时间段内平均响应时间监控

规则模板

```
increase(http_server_requests_seconds_sum{job="${app}",exception="None", uri="${uri}"}[${timeRange}])/
increase(http_server_requests_seconds_count{job="${app}",exception="None", uri="${uri}"}[${timeRange}]) >${value}
```

规则描述

监控某个接口在指定时间范围内的相应时间

参数解释

```
app : 应用名
instance : 实例名，默认为 IP:PORT
uri : 接口地址
timeRange : 时间范围
value :监控指标，long类型，毫秒级别。 
```

labels参数说明

```
env : 数据源（通常用于区分环境）
exception : 异常信息
instance ： 实例名
job : 应用名
method ： POST 或GET
status ：http请求状态 200为成功
uri : 接口地址
```

10.接口某个时间段内平均响应时间监控（正则表达式）

规则模板

```
increase(http_server_requests_seconds_sum{job=~"${app}",exception="None", uri=~"${uri}"}[${timeRange}])/increase(http_server_requests_seconds_count{job="${app}",exception="None", uri=~"${uri}"}[${timeRange}]) >${value}
```

规则描述

监控某个接口在指定时间范围内的响应时间，比如在某些场景下，有些接口的请求时间过于慢了， 这样我们可以及时收到通知，以便后续优化。

参数解释

```
app : 应用名, 正则表达式匹配
uri : 接口地址 , 正则表达式匹配
timeRange : 时间范围
value :监控指标，long类型，毫秒级别。 
```

labels参数说明

```
env : 数据源（通常用于区分环境）
exception : 异常信息
instance ： 实例名
job : 应用名
method ： POST 或GET
status ：http请求状态 200为成功
uri : 接口地址
```



### 服务器相关

11.全局CPU使用率监测

规则模板

```
100 - ((avg by (instance,job,env)(irate(node_cpu_seconds_total{mode="idle"}[30s]))) *100) > ${value}
```

规则描述

监测CPU的平均使用率

参数解释

```
value :监控指标，百分比，int整数 
```

labels参数说明

```
env : 数据源（通常用于区分环境）
instance ： 实例名
job : 应用名
```

12.监测指定服务器的CPU使用率

规则模板

```
100 - ((avg by (instance,job,env)(irate(node_cpu_seconds_total{mode="idle",job="${app}"}[30s]))) *100) > ${value}
```

规则描述

监测某个应用的CPU的平均使用率

参数解释

```
app : 服务器IP 
value :监控指标，百分比，int整数 
```

labels参数说明

```
env : 数据源（通常用于区分环境）
instance ： 实例名
job : 应用名
```

13.内存使用率

规则模板

```
((node_memory_MemTotal_bytes -(node_memory_MemFree_bytes+node_memory_Buffers_bytes+node_memory_Cached_bytes) )/node_memory_MemTotal_bytes ) * 100 > ${value}
```

规则描述

监测内存使用率

参数解释

```
value :监控指标，百分比，int整数 
```

labels参数说明

```
env : 数据源（通常用于区分环境）
instance ： 实例名
job : 应用名
```

14.磁盘使用率

规则模板

```
(node_filesystem_avail_bytes{fstype !~ "nfs|rpc_pipefs|rootfs|tmpfs",device!~"/etc/auto.misc|/dev/mapper/centos-home",mountpoint !~ "/boot|/net|/selinux"} /node_filesystem_size_bytes{fstype !~ "nfs|rpc_pipefs|rootfs|tmpfs",device!~"/etc/auto.misc|/dev/mapper/centos-home",mountpoint !~ "/boot|/net|/selinux"} ) * 100 > ${value}
```

规则描述

监测磁盘使用的比率，可以自定义当使用率大于多少的时候进行报警

参数解释

```
value :监控指标，百分比，int整数 
```

labels参数说明

```
env : 数据源（通常用于区分环境）
instance ： 实例名
job : 应用名
device : 系统路径
fstype : 文件系统类型
mountpoint : /
```

15.网卡流出速率

规则模板

```
(irate(node_network_transmit_bytes_total{device!~"lo"}[1m]) / 1000) > ${value}
```

规则描述

监控网卡的流出速率

参数解释

```
value :监控指标,单位为 kb
```

labels参数说明

```
env : 数据源（通常用于区分环境）
instance ： 实例名
job : 应用名
device : 网卡名称 ，例： eth0 , eth1
```

16.系统负载率1分钟

规则模板

```
node_load1 > ${value}
```

规则描述

监测系统一分钟内的负载率。

参数解释

```
value :监控指标，dubble小数
```

labels参数说明

```
env : 数据源（通常用于区分环境）
instance ： 实例名
job : 应用名
```

17.系统负载率5分钟

规则模板

```
node_load5 > ${value}
```

规则描述

监测系统5分钟内的负载率。

参数解释

```
value :监控指标，dubble小数
```

labels参数说明

```
env : 数据源（通常用于区分环境）
instance ： 实例名
job : 应用名
```

18.系统负载率15分钟

规则模板

```
node_load15 > ${value}
```

规则描述

监测系统15分钟内的负载率。

参数解释

```
value :监控指标，dubble小数
```

labels参数说明

```
env : 数据源（通常用于区分环境）
instance ： 实例名
job : 应用名
```

# Prometheus一条告警是怎么触发的

**文章来源：爱可生云数据库**
**作者：张沈波**

大纲

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