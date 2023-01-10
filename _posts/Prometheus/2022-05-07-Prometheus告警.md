---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---
# Prometheus告警

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