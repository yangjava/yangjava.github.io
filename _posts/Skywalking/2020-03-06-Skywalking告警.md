---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
# 告警

Apache SkyWalking告警是由一组规则驱动，这些规则定义在config/alarm-settings.yml文件中。

告警规则的定义分为三部分。

1. **告警规则**：定义了触发告警所考虑的条件。
2. **webhook**：当告警触发时，被调用的服务端点列表。
3. **gRPCHook**：当告警触发时，被调用的远程gRPC方法的主机和端口。
4. **Slack Chat Hook**：当告警触发时，被调用的Slack Chat接口。
5. **微信 Hook**：当告警触发时，被调用的微信接口。
6. **钉钉 Hook**：当告警触发时，被调用的钉钉接口。

#### 告警规则

告警规则有两种类型，单独规则（Individual Rules）和复合规则（Composite Rules），复合规则是单独规则的组合。

##### 单独规则（Individual Rules）

单独规则主要有以下几点：

- **规则名称**：在告警信息中显示的唯一名称，必须以_rule结尾。
- **metrics-name**：度量名称，也是OAL脚本中的度量名。默认配置中可以用于告警的度量有：**服务**，**实例**，**端点**，**服务关系**，**实例关系**，**端点关系**。它只支持long,double和int类型。
- **include-names**：包含在此规则之内的实体名称列表。
- **exclude-names**：排除在此规则以外的实体名称列表。
- **include-names-regex**：提供一个正则表达式来包含实体名称。如果同时设置包含名称列表和包含名称的正则表达式，则两个规则都将生效。
- **exclude-names-regex**：提供一个正则表达式来排除实体名称。如果同时设置排除名称列表和排除名称的正则表达式，则两个规则都将生效。
- **include-labels**：包含在此规则之内的标签。
- **exclude-labels**：排除在此规则以外的标签。
- **include-labels-regex**：提供一个正则表达式来包含标签。如果同时设置包含标签列表和包含标签的正则表达式，则两个规则都将生效。
- **exclude-labels-regex**：提供一个正则表达式来排除标签。如果同时设置排除标签列表和排除标签的正则表达式，则两个规则都将生效。

标签的设置必须把数据存储在meter-system中，例如：Prometheus, Micrometer。以上四个标签设置必须实现LabeledValueHolder接口。

- **threshold**：阈值。

对于多个值指标，例如**percentile**，阈值是一个数组。像value1 value2 value3 value4 value5这样描述。
每个值可以作为度量中每个值的阈值。如果不想通过此值或某些值触发警报，则将值设置为 -。
例如在**percentile**中，value1是P50的阈值，value2是P75的阈值，那么-，-，value3, value4, value5的意思是，没有阈值的P50和P75的**percentile**告警规则。

- **op**：操作符，支持>, >=, <, <=, =。
- **period**：多久告警规则需要被检查一下。这是一个时间窗口，与后端部署环境时间相匹配。
- **count**：在一个周期窗口中，如果按**op**计算超过阈值的次数达到**count**，则发送告警。
- **only-as-condition**：true或者false，指定规则是否可以发送告警，或者仅作为复合规则的条件。
- **silence-period**：在时间N中触发报警后，在**N -> N + silence-period**这段时间内不告警。 默认情况下，它和**period**一样，这意味着相同的告警（同一个度量名称拥有相同的Id）在同一个周期内只会触发一次。
- **message**：该规则触发时，发送的通知消息。

举个例子：

```plain
rules:
  service_resp_time_rule:
    metrics-name: service_resp_time
    op: ">"
    threshold: 1000
    period: 10
    count: 2
    silence-period: 10
    message: 服务【{name}】的平均响应时间在最近10分钟内有2分钟超过1秒
  service_instance_resp_time_rule:
    metrics-name: service_instance_resp_time
    op: ">"
    threshold: 1000
    period: 10
    count: 2
    silence-period: 10
    message: 实例【{name}】的平均响应时间在最近10分钟内有2分钟超过1秒
  endpoint_resp_time_rule:
    metrics-name: endpoint_avg
    threshold: 1000
    op: ">"
    period: 10
    count: 2
    message: 端点【{name}】的平均响应时间在最近10分钟内有2分钟超过1秒
```

##### 复合规则（Composite Rules）

复合规则仅适用于针对相同实体级别的告警规则，例如都是服务级别的告警规则：service_percent_rule && service_resp_time_percentile_rule。
**不可以**编写不同实体级别的告警规则，例如服务级别的一个告警规则和端点级别的一个规则：service_percent_rule && endpoint_percent_rule。

复合规则主要有以下几点：

- **规则名称**：在告警信息中显示的唯一名称，必须以_rule结尾。
- **expression**：指定如何组成规则，支持&&, ||, ()操作符。
- **message**：该规则触发时，发送的通知消息。

举个例子：

```plain
rules:
  service_resp_time_rule:
    metrics-name: service_resp_time
    op: ">"
    threshold: 1000
    period: 10
    count: 2
    silence-period: 10
    message: 服务【{name}】的平均响应时间在最近10分钟内有2分钟超过1秒
  service_sla_rule:
    metrics-name: service_sla
    op: "<"
    threshold: 8000
    period: 10
    count: 2
    silence-period: 10
    message: 服务【{name}】的成功率在最近10分钟内有2分钟低于80％
composite-rules:
  comp_rule:
    expression: service_resp_time_rule && service_sla_rule
    message: 服务【{name}】在最近10分钟内有2分钟超过1秒平均响应时间超过1秒并且成功率低于80％
```

#### Webhook

Webhook 要求一个点对点的 Web 容器。告警的消息会通过 HTTP 请求进行发送，请求方法为 POST，Content-Type 为 application/json，JSON 格式包含以下信息：

- **scopeId**：目标 Scope 的 ID。
- **name**：目标 Scope 的实体名称。
- **id0**：Scope 实体的 ID。
- **id1**：未使用。
- **ruleName**：您在 alarm-settings.yml 中配置的规则名。
- **alarmMessage**. 告警消息内容。
- **startTime**. 告警时间戳，当前时间与 UTC 1970/1/1 相差的毫秒数。

举个例子：

```plain
[{
	"scopeId": 1, 
	"scope": "SERVICE",
	"name": "one-more-service", 
	"id0": "b3JkZXItY2VudGVyLXNlYXJjaC1hcGk=.1",  
	"id1": "",  
    "ruleName": "service_resp_time_rule",
	"alarmMessage": "服务【one-more-service】的平均响应时间在最近10分钟内有2分钟超过1秒",
	"startTime": 1617670815000
}, {
	"scopeId": 2,
	"scope": "SERVICE_INSTANCE",
	"name": "e4b31262acaa47ef92a22b6a2b8a7cb1@192.168.30.11 of one-more-service",
	"id0": "dWF0LWxib2Mtc2VydmljZQ==.1_ZTRiMzEyNjJhY2FhNDdlZjkyYTIyYjZhMmI4YTdjYjFAMTcyLjI0LjMwLjEzOA==",
	"id1": "",
    "ruleName": "instance_jvm_young_gc_count_rule",
	"alarmMessage": "实例【e4b31262acaa47ef92a22b6a2b8a7cb1@192.168.30.11 of one-more-service】的YoungGC次数在最近10分钟内有2分钟超过10次",
	"startTime": 1617670815000
}, {
	"scopeId": 3,
	"scope": "ENDPOINT",
	"name": "/one/more/endpoint in one-more-service",
	"id0": "b25lcGllY2UtYXBp.1_L3RlYWNoZXIvc3R1ZGVudC92aXBsZXNzb25z",
	"id1": "",
    "ruleName": "endpoint_resp_time_rule",
	"alarmMessage": "端点【/one/more/endpoint in one-more-service】的平均响应时间在最近10分钟内有2分钟超过1秒",
	"startTime": 1617670815000
}]
```

#### gRPCHook

告警消息将使用 Protobuf 类型通过gRPC远程方法发送。消息格式的关键信息定义如下：

```plain
syntax = "proto3";

option java_multiple_files = true;
option java_package = "org.apache.skywalking.oap.server.core.alarm.grpc";

service AlarmService {
    rpc doAlarm (stream AlarmMessage) returns (Response) {
    }
}

message AlarmMessage {
    int64 scopeId = 1;
    string scope = 2;
    string name = 3;
    string id0 = 4;
    string id1 = 5;
    string ruleName = 6;
    string alarmMessage = 7;
    int64 startTime = 8;
}

message Response {
}
```

#### Slack Chat Hook

您需要遵循[传入Webhooks入门指南](https://api.slack.com/messaging/webhooks)并创建新的Webhooks。

如果您按以下方式配置了Slack Incoming Webhooks，则告警消息将按 Content-Type 为 application/json 通过HTTP的 POST 方式发送。

举个例子：

```plain
slackHooks:
  textTemplate: |-
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": ":alarm_clock: *Apache Skywalking Alarm* \n **%s**."
      }
    }
  webhooks:
    - https://hooks.slack.com/services/x/y/z
```

#### 微信Hook

只有微信的企业版才支持 Webhooks ，如何使用微信的 Webhooks 可参见[如何配置群机器人](https://work.weixin.qq.com/help?doc_id=13376)。

如果您按以下方式配置了微信的 Webhooks ，则告警消息将按 Content-Type 为 application/json 通过HTTP的 POST 方式发送。

举个例子：

```plain
wechatHooks:
  textTemplate: |-
    {
      "msgtype": "text",
      "text": {
        "content": "Apache SkyWalking 告警: \n %s."
      }
    }
  webhooks:
    - https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=dummy_key
```

#### 钉钉 Hook

您需要遵循[自定义机器人开放](https://ding-doc.dingtalk.com/doc#/serverapi2/qf2nxq/uKPlK)并创建新的Webhooks。为了安全起见，您可以为Webhook网址配置可选的密钥。

如果您按以下方式配置了钉钉的 Webhooks ，则告警消息将按 Content-Type 为 application/json 通过HTTP的 POST 方式发送。

举个例子：

```plain
dingtalkHooks:
  textTemplate: |-
    {
      "msgtype": "text",
      "text": {
        "content": "Apache SkyWalking 告警: \n %s."
      }
    }
  webhooks:
    - url: https://oapi.dingtalk.com/robot/send?access_token=dummy_token
      secret: dummysecret
```