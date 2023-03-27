---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---
# Prometheus告警管理
Prometheus 包含一个 报警模块 ，就是我们的 AlertManager，Alertmanager 主要用于接收 Prometheus 发送的告警信息，它支持丰富的告警通知渠道，而且很容易做到告警信息进行去重，降噪，分组等，是一款前卫的告警通知系统。

- 如何构建良好的警报机制
- 如何安装和配置Alertmanager
- 如何使用它来路由通知和管理维护

## 警报
警报可以为我们提供一些指示，表明我们环境中的某些状态已发生变化，且通常会是比想象更糟的情况。一个好警报的关键是能够在正确的时间、以正确的理由和正确的速度发送，并在其中放入有用的信息。

警报方法中最常见的反模式是发送过多的警报。对于监控来说，过多的警报相当于“狼来了”这样的故事。收件人会对警告变得麻木，然后将其拒之门外，而重要的警报常常被淹没在无关紧要的更新中。

通常发送过多警报的原因可能包括：
- 警报缺少可操作性，它只是提供信息。你应关闭所有这些警报，或将其转换为计算速率的计数器，而不是发出警报。
- 故障的主机或服务上游会触发其下游的所有内容的警报。你应确保警报系统识别并抑制这些重复的相邻警报。
- 对原因而不是症状（symptom）进行警报。症状是应用程序停止工作的迹象，它们可能是由许多原因导致的各种问题的表现。API或网站的高延迟是一种症状，这种症状可能由许多问题导致：高数据库使用率、内存问题、磁盘性能等。对症状发送警报可以识别真正的问题。仅对原因（例如高数据库使用率）发出警报也可能识别出问题（但通常很可能不会）。对于这个应用程序，高数据库使用率可能是完全正常的，并且可能不会对最终用户或应用程序造成性能问题。作为一个内部状态，发送警报是没有意义的。这种警报可能会导致工程师错过更重要的问题，因为他们已经对大量不可操作且基于原因的警报变得麻木。你应该关注基于症状的警报，并依赖你的指标或其他诊断数据来确定原因。

第二种最常见的反模式是警报的错误分类。有时，这也意味着重要的警报会隐藏在其他警报中。但有时候，警报会被发送到错误的地方或者带有不正确的紧急情况说明。

第三种反模式是发送无用的警报，尤其当收件人通常是一个疲惫的、刚刚被叫醒的工程师（并且这可能是他在夜班中收到的第三个或第四个通知）时。

良好的警报应该具备以下几个关键特征：
- 适当数量的警报，关注症状而不是原因。噪声警报会导致警报疲劳，最终警报会被忽略。修复警报不足比修复过度警报更容易。
- 应设置正确的警报优先级。如果警报是紧急的，那么它应该快速路由到负责响应的一方。如果警报不紧急，那么我们应该以适当的速度发送警报，以便在需要时做出响应。
- 警报应包括适当的上下文，以便它们立即可以使用。

## Alertmanager如何工作
Alertmanager处理从客户端发来的警报，客户端通常是Prometheus服务器。Alertmanager对警报进行去重、分组，然后路由到不同的接收器，如电子邮件、短信或SaaS服务。你还可以使用Alertmanager管理维护。

我们将在Prometheus服务器上编写警报规则，这些规则将使用我们收集的指标并在指定的阈值或标准上触发警报。我们还将看到如何为警报添加一些上下文。当指标达到阈值或标准时，会生成一个警报并将其推送到Alertmanager。警报在Alertmanager上的HTTP端点上接收。一个或多个Prometheus服务器可以将警报定向到单个Alertmanager，或者你可以创建一个高可用的Alertmanager集群。

收到警报后，Alertmanager会处理警报并根据其标签进行路由。一旦路径确定，它们将由Alertmanager发送到外部目的地，如电子邮件、短信或聊天工具。

## AlertManager 架构
Alertmanager 由以下 6 部分组成：
- API 组件
用于接收 Prometheus Server 的 http 请求，主要是告警相关的内容。
- Alert Provider 组件 
API 层将来自 Prometheus Server 的告警信息存储到 Alert Provider 上。
- Dispatcher 组件 
不断的通过订阅的方式从 Alert Provider 获取新的告警，并根据 yaml 配置的 routing tree 将告警通过 label 路由到不同的分组中，以实现告警信息的分组处理。
- Notification Pipeline 组件 
这是一个责任链模式的组件，它通过一系列的逻辑（抑制、静默、去重）来优化告警质量。
- Silence Provider 组件 
API 层将来自 prometheus server 的告警信息存储到 silence provider 上，然后由这个组件实现去重逻辑处理。
- Notify Provider 组件 
是 Silence Provider 组件的下游，会在本地记录日志，并通过 peers 的方式将日志广播给集群中的其他节点，判断当前节点自身或者集群中其他节点是否已经发送过了，避免告警信息在集群中重复出现。

## AlertManager 部署

### 下载

```shell
wget https://github.com/prometheus/alertmanager/releases/download/v0.24.0/alertmanager-0.24.0.linux-amd64.tar.gz  
tar -xf alertmanager-0.24.0.linux-amd64.tar.gz  
```

### 配置
```yaml
global:  
  resolve_timeout: 5m  
  
route:  
  group_by: ['alertname']  
  group_wait: 30s  
  group_interval: 5m  
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
Alertmanager配置中一般会包含以下几个主要部分：
- 全局配置（global）：用于定义一些全局的公共参数，如全局的SMTP配置，Slack配置等内容。 
- 模板（templates）：用于定义告警通知时的模板，如HTML模板，邮件模板等。
- 告警路由（route）：根据标签匹配，确定当前告警应该如何处理。
- 接收人（receivers）：接收人是一个抽象的概念，它可以是一个邮箱也可以是 微信 ， Slack 或者 Webhook 等，接收人一般配合告警路由使用。
- 抑制规则（inhibit_rules）：合理设置抑制规则可以减少垃圾告警的产生。

具体字段解释：
- global 这个是全局设置
- resolve_timeout 当告警的状态有firing变为resolve的以后还要呆多长时间，才宣布告警解除。这个主要是解决某些监控指标在阀值边缘上波动，一会儿好一会儿不好。
- route 是个重点，告警内容从这里进入，寻找自己应该用那种策略发送出去。
- receiver 一级的receiver，也就是默认的receiver，当告警进来后没有找到任何子节点和自己匹配，就用这个receiver。
- group_by 告警应该根据那些标签进行分组。 
- group_wait 同一组的告警发出前要等待多少秒，这个是为了把更多的告警一个批次发出去。
- group_interval 同一组的多批次告警间隔多少秒后，才能发出。
- repeat_interval 重复的告警要等待多久后才能再次发出去。
- routes 也就是子节点了，配置项和上面一样。告警会一层层的找，如果匹配到一层，并且这层的continue选项为true，那么他会再往下找，如果下层节点不能匹配那么他就用区配的这一层的配置发送告警。如果匹配到一层，并且这层的continue选项为false，那么他会直接用这一层的配置发送告警，就不往下找了。
- match_re 用于匹配label。此处列出的所有label都匹配到才算匹配。
- inhibit_rules这个叫做抑制项，通过匹配源告警来抑制目的告警。比如说当我们的主机挂了，可能引起主机上的服务，数据库，中间件等一些告警，假如说后续的这些告警相对来说没有意义，我们可以用抑制项这个功能，让Prometheus只发出主机挂了的告警。
- source_match 根据label匹配源告警。
- target_match 根据label匹配目的告警。
- equal 此处的集合的label，在源和目的里的值必须相等。如果该集合的内的值再源和目的里都没有，那么目的告警也会被抑制。

## 启动服务
```shell
# 查看帮助  
./alertmanager -h  
# 可以直接启动，但是不提倡，最好配置alertmanager.service  
./alertmanager  
```

配置alertmanager.service启动脚本
```shell
# 默认端口9093  
cat >/usr/lib/systemd/system/alertmanager.service<<EOF  
[Unit]  
Description=alertmanager  
  
[Service]  
WorkingDirectory=/opt/prometheus/alertmanager/alertmanager-0.24.0.linux-amd64  
ExecStart=/opt/prometheus/alertmanager/alertmanager-0.24.0.linux-amd64/alertmanager --config.file=/opt/prometheus/alertmanager/alertmanager-0.24.0.linux-amd64/alertmanager.yml --storage.path=/opt/prometheus/alertmanager/alertmanager-0.24.0.linux-amd64/data --web.listen-address=:9093 --data.retention=120h  
Restart=on-failure  
  
[Install]  
WantedBy=multi-user.target  
EOF  
```

## 启动服务
```shell
# 执行 systemctl daemon-reload 命令重新加载systemd  
systemctl daemon-reload  
# 启动  
systemctl start alertmanager  
# 检查  
systemctl status alertmanager  
netstat -tnlp|grep :9093  
ps -ef|grep alertmanager  
```

web访问：http://ip:9093

## 在Prometheus中设置告警规则
在Prometheus 配置中添加报警规则配置，配置文件中 rule_files 就是用来指定报警规则文件的，如下配置即指定存放报警规则的目录为/etc/prometheus，规则文件为rules.yml：
```yaml
rule_files:  
- /etc/prometheus/rules.yml  
```

设置报警规则：
警报规则允许基于 Prometheus 表达式语言的表达式来定义报警报条件的，并在触发警报时发送通知给外部的接收者（Alertmanager），一条警报规则主要由以下几部分组成：
- alert——告警规则的名称。
- expr——是用于进行报警规则 PromQL 查询语句。
- for——评估告警的等待时间（Pending Duration）。
- labels——自定义标签，允许用户指定额外的标签列表，把它们附加在告警上。
- annotations——用于存储一些额外的信息，用于报警信息的展示之类的。

rules.yml如下所示：
```yaml
groups:  
- name: example  
  rules:  
  - alert: high_memory  
    # 当内存占有率超过10%，持续1min,则触发告警  
    expr: 100 - ((node_memory_MemAvailable_bytes{instance="192.168.182.110:9100",job="node_exporter"} * 100) / node_memory_MemTotal_bytes{instance="192.168.182.110:9100",job="node_exporter"}) > 90  
    for: 1m  
    labels:  
      severity: page  
    annotations:  
      summary: spike memeory  
```

## AlertManager 告警通道配置
```yaml
## Alertmanager 配置文件  
global:  
  resolve_timeout: 5m  
  # smtp配置  
  smtp_from: "xxx@qq.com"  
  smtp_smarthost: 'smtp.qq.com:465'  
  smtp_auth_username: "xxx@qq.com"  
  smtp_auth_password: "auth_pass"  
  smtp_require_tls: true  
  
# email、企业微信的模板配置存放位置，钉钉的模板会单独讲如果配置。  
templates:  
  - '/data/alertmanager/templates/*.tmpl'  
# 路由分组  
route:  
  receiver: ops  
  group_wait: 30s # 在组内等待所配置的时间，如果同组内，30秒内出现相同报警，在一个组内出现。  
  group_interval: 5m # 如果组内内容不变化，合并为一条警报信息，5m后发送。  
  repeat_interval: 24h # 发送报警间隔，如果指定时间内没有修复，则重新发送报警。  
  group_by: [alertname]  # 报警分组  
  routes:  
      - match:  
          team: operations  
        group_by: [env,dc]  
        receiver: 'ops'  
      - match_re:  
          service: nginx|apache  
        receiver: 'web'  
      - match_re:  
          service: hbase|spark  
        receiver: 'hadoop'  
      - match_re:  
          service: mysql|mongodb  
        receiver: 'db'  
# 接收器  
# 抑制测试配置  
      - receiver: ops  
        group_wait: 10s  
        match:  
          status: 'High'  
# ops  
      - receiver: ops # 路由和标签，根据match来指定发送目标，如果 rule的lable 包含 alertname， 使用 ops 来发送  
        group_wait: 10s  
        match:  
          team: operations  
# web  
      - receiver: db # 路由和标签，根据match来指定发送目标，如果 rule的lable 包含 alertname， 使用 db 来发送  
        group_wait: 10s  
        match:  
          team: db  
# 接收器指定发送人以及发送渠道  
receivers:  
# ops分组的定义  
- name: ops  
  email_configs:  
  - to: '9935226@qq.com,10000@qq.com'  
    send_resolved: true  
    headers:  
      subject: "[operations] 报警邮件"  
      from: "警报中心"  
      to: "小煜狼皇"  
  # 钉钉配置  
  webhook_configs:  
  - url: http://localhost:8070/dingtalk/ops/send  
    # 企业微信配置  
  wechat_configs:  
  - corp_id: 'ww5421dksajhdasjkhj'  
    api_url: 'https://qyapi.weixin.qq.com/cgi-bin/'  
    send_resolved: true  
    to_party: '2'  
    agent_id: '1000002'  
    api_secret: 'Tm1kkEE3RGqVhv5hO-khdakjsdkjsahjkdksahjkdsahkj'  
  
# web  
- name: web  
  email_configs:  
  - to: '9935226@qq.com'  
    send_resolved: true  
    headers: { Subject: "[web] 报警邮件"} # 接收邮件的标题  
  webhook_configs:  
  - url: http://localhost:8070/dingtalk/web/send  
  - url: http://localhost:8070/dingtalk/ops/send  
# db  
- name: db  
  email_configs:  
  - to: '9935226@qq.com'  
    send_resolved: true  
    headers: { Subject: "[db] 报警邮件"} # 接收邮件的标题  
  webhook_configs:  
  - url: http://localhost:8070/dingtalk/db/send  
  - url: http://localhost:8070/dingtalk/ops/send  
# hadoop  
- name: hadoop  
  email_configs:  
  - to: '9935226@qq.com'  
    send_resolved: true  
    headers: { Subject: "[hadoop] 报警邮件"} # 接收邮件的标题  
  webhook_configs:  
  - url: http://localhost:8070/dingtalk/hadoop/send  
  - url: http://localhost:8070/dingtalk/ops/send  
  
# 抑制器配置  
inhibit_rules: # 抑制规则  
  - source_match: # 源标签警报触发时抑制含有目标标签的警报，在当前警报匹配 status: 'High'  
      status: 'High'  # 此处的抑制匹配一定在最上面的route中配置不然，会提示找不key。  
    target_match:  
      status: 'Warning' # 目标标签值正则匹配，可以是正则表达式如: ".*MySQL.*"  
    equal: ['alertname','operations', 'instance'] # 确保这个配置下的标签内容相同才会抑制，也就是说警报中必须有这三个标签值才会被抑制。 
```
一般企业钉钉、邮件、webhook告警通道比较常用，尤其是webhook，一般都会通过webhook对接公司内部的统一告警平台。









---------------------------------------------
# 数据模型

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


# 参考资料
GitHub 地址：https://github.com/prometheus/alertmanager/[1]
官方文档：https://prometheus.io/docs/alerting/latest/alertmanager/[2]
关于 Prometheus 整体介绍，可以参考这篇文章：Prometheus 原理详解[3]
Prometheus 和 Pushgetway 的安装，可以参考这篇文章：Prometheus Pushgetway 讲解与实战操作[4]