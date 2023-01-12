---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---
# Prometheus配置



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