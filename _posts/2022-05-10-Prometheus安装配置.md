---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---
# Prometheus安装配置
Prometheus云原生时代监控领域的现象级产品，常与Grafana搭配使用，是当前互联网企业的首选监控解决方案。

## Docker下搭建Prometheus
Prometheus安装方式可以通过二进制文件安装，也可以通过docker方式安装。

### 安装 Prometheus Server
- 获取prometheus镜像  

```shell
docker pull prom/prometheus
```

- 新建prometheus目录，并编辑prometheus.yml文件

```shell
mkdir /usr/local/prometheus
mkdir /usr/local/prometheus/data
cd /usr/local/prometheus/data
vim prometheus.yml
```
prometheus.yml内容如下
```yaml

# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
 
# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093
 
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
 
# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
 
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
 
    static_configs:
    - targets: ['localhost:9090']
```

- 启动prometheus
```shell
sudo docker run -d -p 9090:9090 --name prom --net=host -v /usr/local/prometheus/data:/data prom/prometheus --config.file=/data/prometheus.yml
```

- 访问url
```text
http://localhost:9090/
```

### 安装Node Exporter
所有服务器安装  
Node Exporter 收集系统信息，用于监控CPU、内存、磁盘使用率、磁盘读写等系统信息  
–net=host，这样 Prometheus Server 可以直接与 Node Exporter 通信  
```shell
docker run -d -p 9100:9100 \
-v "/proc:/host/proc" \
-v "/sys:/host/sys" \
-v "/:/rootfs" \
-v "/etc/localtime:/etc/localtime" \
--net=host \
prom/node-exporter \
--path.procfs /host/proc \
--path.sysfs /host/sys \
--collector.filesystem.ignored-mount-points "^/(sys|proc|dev|host|etc)($|/)"
```
验证
```shell
http://localhost:9100/metrics
```

添加job
```yaml
  - job_name: 'os_1'
    static_configs:
    - targets: ['localhost:9100']
      labels:
         instance: os_1
```
执行
```shell
docker restart prom/prometheus
```



### 安装cAdvisor

所有服务器安装
cAdvisor 收集docker信息，用于展示docker的cpu、内存、上传下载等信息
–net=host，这样 Prometheus Server 可以直接与 cAdvisor 通信
```yaml
docker run -d \
-v "/etc/localtime:/etc/localtime" \
--volume=/:/rootfs:ro \
--volume=/var/run:/var/run:rw \
--volume=/sys:/sys:ro \
--volume=/var/lib/docker/:/var/lib/docker:ro \
--volume=/dev/disk/:/dev/disk:ro \
--publish=18104:8080 \
--detach=true \
--name=cadvisor \
--privileged=true \
google/cadvisor:latest
```
完整的配置如下
```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090','localhost:9100','localhost:18104']                                                                     
```



## k8s集群下搭建Prometheus

安装主要有YAML、Operater两种，先从YAML开始可以更好的理解细节（Operater最终也是生成的yml文件）。需要考虑几个点：
-访问权限
-配置文件
- 存储卷

**首先在k8s集群创建命名空间monitoring**
```shell
kubectl create namespace monitoring
```

**访问权限相关的配置**
创建prometheus访问权限配置prometheus-sa.yaml
```yaml
## 服务账户
apiVersion: v1  
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
#集群角色
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources:
      - configmaps
    verbs: ["get"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
---
#服务账户与集群角色绑定，完成授权
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring

```
创建相关内容
```shell
kubectl apply -f prometheus-sa.yaml
```
**创建prometheus需要用到的配置文件，以configmap形式挂载到pod内**
创建prometheus需要用到的配置文件，以configmap形式挂载到pod内
```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s
  external_labels:
    cluster: k8s-cluster # 添加公共标签，在被其他prometheus采集时标识数据来源
alerting:
  alertmanagers:
  - static_configs:
    - targets:
       - alertmanager.monitoring.svc:9093 # 在monitoring命名空间下创建alertmanager后访问地址
 
rule_files:
  - etc/prometheus-rules/*_rules.yaml # rules配置文件挂载位置
  - etc/prometheus-rules/*_alerts.yaml
 
scrape_configs:
   config.
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
    - targets: ['localhost:9090']

#remote_write:
  #- url: "http://localhost:8086/api/v1/prom/write?db=prometheus&u=username&p=password"
```

/rules/example_rules.yaml

```yaml
groups:
  - name: example
    rules:
    - record: up:count
      expr: sum(up) by (job)
```

/rules/example_alerts.yaml

```yaml
groups:
- name: example
  rules:
  - alert: InstanceDown
    expr: up == 0.5
    for: 10m
    labels:
      severity: page
    annotations:
      summary: Instance {{ $labels.instance }} down
```

分别创建configmap

```shell
kubectl create configmap prometheus-rules --from-file=/rules -n monitoring
kubectl create configmap prometheus-config --from-file=prometheus.yaml -n monitoring
```

**创建pod，如果需要将监控数据做持久化存储，需要挂载pv，同时最好使用statefuset类型创建pod**

prometheus-deploy.yaml

```yaml
apiVersion: app/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - image: prom/prometheus
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus" # 使用自身Pod进行存储
        - "--storage.tsdb.retention=30d" # 数据保留30天
        - "--web.enable-admin-api"  # 控制对admin HTTP API的访问，其中包括删除时间序列等功能
        - "--web.enable-lifecycle"  # 支持热更新，直接执行localhost:9090/-/reload立即生效
        ports:
        - containerPort: 9090
          protocol: TCP
          name: http
        volumeMounts:
        - mountPath: "/etc/prometheus"
          name: config-volume
        - mountPath: "/etc/prometheus-rules"
          name: rule-volume
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 4000m
            memory: 10Gi
      securityContext:
        runAsUser: 0
      volumes:
      - configMap:
          name: prometheus-config
        name: config-volume
      - name: rule-volume
        configMap:
          name: prometheus-rules
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
spec:
  selector:
    app: prometheus
  type: NodePort
  ports:
    - name: web
      nodePort: 9090
      port: 9090
      targetPort: http
```
创建node-port类型svc，指定9090端口供外部访问

执行 prometheus-deploy.yaml
```shell
kubectl apply -f prometheus-deploy.yaml
```

到这里，prometheus在k8s环境下的搭建就完成


## Prometheus监控k8s集群资源

### 监控流程
- 容器监控：
Prometheus使用cadvisor采集容器监控指标，而cadvisor集成在K8S的kubelet中所以无需部署，通过Prometheus进程存储，使用grafana进行展示
- node节点监控：
node端的监控通过node_exporter采集当前主机的资源，通过Prometheus进程存储，最后使用grafana进行展示
- master节点监控：
master的监控通过kube-state-metrics插件从K8S获取到apiserver的相关数据并通过网页页面暴露出来，然后通过Prometheus进程存储，最后使用grafana进行展示

### Kubernetes监控指标
- K8S本身的监控指标

node的资源利用率：监控node节点上的cpu、内存、硬盘等硬件资源
node的数量：监控node数量与资源利用率、业务负载的比例情况，对成本、资源扩展进行评估
pod的数量：监控当负载到一定程度时，node与pod的数量。评估负载到哪个阶段，大约需要多少服务器，以及每个pod的资源占用率，然后进行整体评估
资源对象状态：在K8S运行过程中，会创建很多pod、控制器、任务等，这些内容都是由K8S中的资源对象进行维护，所以我们可以对资源对象进行监控，获取资源对象的状态

- Pod的监控

每个项目中pod的数量：分别监控正常、有问题的pod数量
容器资源利用率：统计当前pod的资源利用率，统计pod中的容器资源利用率，结合cpu、网络、内存进行评估
应用程序：监控项目中程序的自身情况，例如并发、请求响应等

### 服务发现

从k8s的api中发现抓取的目标，并且始终与k8s集群状态保持一致。动态获取被抓取的目标，实时从api中获取当前状态是否存在

## 采用daemonset方式部署node-exporter

node-exporter.yaml

```yaml
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: kube-system
  labels:
    k8s-app: node-exporter
spec:
  selector:
    matchLabels:
      k8s-app: node-exporter
  template:
    metadata:
      labels:
        k8s-app: node-exporter
    spec:
      containers:
      - image: prom/node-exporter
        name: node-exporter
        ports:
        - containerPort: 9100
          protocol: TCP
          name: http
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: node-exporter
  name: node-exporter
  namespace: kube-system
spec:
  ports:
  - name: http
    port: 9100
    nodePort: 31672
    protocol: TCP
  type: NodePort
  selector:
    k8s-app: node-exporter
```


## prometheus.yml

```yaml
    global:
      scrape_interval:     15s
      evaluation_interval: 15s
    scrape_configs:

    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

    - job_name: 'kubernetes-cadvisor'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name

    - job_name: 'kubernetes-services'
      kubernetes_sd_configs:
      - role: service
      metrics_path: /probe
      params:
        module: [http_2xx]
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter.example.com:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_name

    - job_name: 'kubernetes-ingresses'
      kubernetes_sd_configs:
      - role: ingress
      relabel_configs:
      - source_labels: [__meta_kubernetes_ingress_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_ingress_scheme,__address__,__meta_kubernetes_ingress_path]
        regex: (.+);(.+);(.+)
        replacement: ${1}://${2}${3}
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter.example.com:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_ingress_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_ingress_name]
        target_label: kubernetes_name

    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

```

## Prometheus配置
prometheus的配置文件`prometheus.yml`，它主要分以下几个配置块：

- global          全局配置        
- alerting        告警配置
- rule_files      规则文件配置
- scrape_configs  拉取配置
- remote_read、remote_write 远程读写配置  

### global(全局配置)
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

### alerting(告警配置)
`alerting`指定与Alertmanager相关的设置。

```
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]
```

### rule_files(规则文件配置)
`rule_files`指定prometheus加载的任何规则的位置，从所有匹配的文件中读取规则和告警。目前没有规则。

```
rule_files:
  [ - <filepath_glob> ... ]
```

### scrape_configs(拉取配置)
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

### remote_read`/`remote_write(远程读写配置)

`remote_read`/`remote_write`将数据源与prometheus分离，当前不做配置。

```
# 与远程写功能相关的设置
remote_write:
  [ - <remote_write> ... ]

# 与远程读功能相关的设置
remote_read:
  [ - <remote_read> ... ]
```

## 简单配置示例
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









