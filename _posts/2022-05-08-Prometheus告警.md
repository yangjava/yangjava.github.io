---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---
# prometheus告警
Prometheus监控提供了告警规则模板功能，可以帮助用户快速为多个Prometheus实例创建告警规则，降低用户管理多个Prometheus实例告警规则的成本。

## 告警规则模板

- 监控应用是否可用

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