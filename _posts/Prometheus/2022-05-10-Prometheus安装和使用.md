---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---
# Prometheus安装和使用
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







