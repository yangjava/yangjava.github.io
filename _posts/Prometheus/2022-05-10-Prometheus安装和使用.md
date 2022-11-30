---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---
# Prometheus安装和使用
Prometheus云原生时代监控领域的现象级产品，常与Grafana搭配使用，是当前互联网企业的首选监控解决方案。
## Docker下搭建Prometheus




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



