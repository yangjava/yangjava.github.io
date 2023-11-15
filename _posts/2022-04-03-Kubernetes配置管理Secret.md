---
layout: post
categories: [Kubernetes]
description: none
keywords: Kubernetes
---
# Kubernetes配置管理Secret
Secret 类似于 ConfigMap， 但专门用于保存敏感数据，例如密码、令牌或密钥等，其大小也不能超过1MBi。

## 创建Secret
使用kubectl命令可以创建不同类型的secret对象。下面是一个通用的kubectl create secret命令的示例：
```
kubectl create secret <type> <name> <data> <options>
```
其中，指定secret对象的类型，可以是generic、docker-registry、tls等。name指定secret对象的名称，data是secret对象的数据，options是secret对象的其他选项。

以下是一些常见的secret类型和创建命令的示例：
- generic类型的secret，用于存储任意类型的数据：
```
kubectl create secret generic <name> --from-literal=<key>=<value>
```
其中，name是secret对象的名称，key是数据的键名，value是数据的值。
- docker-registry类型的secret，用于存储Docker镜像仓库的认证信息
```
kubectl create secret docker-registry <name> --docker-username=<username> --docker-password=<password> --docker-email=<email> --docker-server=<server>
```
其中，name是secret对象的名称，username、password、email和server是Docker镜像仓库的认证信息。
- tls类型的secret，用于存储TLS证书和私钥：
```
kubectl create secret tls <name> --cert=<cert_file> --key=<key_file>
```
其中，是secret对象的名称，<cert_file>和<key_file>是TLS证书和私钥的文件路径。

### 使用原始数据
```
kubectl create secret generic easy-secret --from-literal=username=admin --from-literal=password=admin123
```
### 使用源文件
```
kubectl create secret generic file-secret --from-file=./username.txt --from-file=./password.txt
```
### 使用源文件的同时指定key
```
kubectl create secret generic file-secret --from-file=username=./username.txt --from-file=password=./password.txt
```

可以通过运行yaml文件创建：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: file-secret
# 默认的 Secret 类型是 Opaque
type: Opaque
data:
  username: WVdSdGFXND0K
  password: WVdSdGFXNHhNak09Cg==
```

## Secret内置类型

| 内置类型                                                 | 用法                  |
|------------------------------------------------------|---------------------|
| Opaque                                               | 用户定义的任意数据，默认类型      |
| kubernetes.io/service-account-token                  | 服务账号令牌              |
| kubernetes.io/dockercfg	~/.dockercfg                 | 文件的序列化形式            |
| kubernetes.io/dockerconfigjson ~/.docker/config.json | 文件的序列化形式            |
| kubernetes.io/basic-auth                             | 用于基本身份认证的凭据         |
| kubernetes.io/ssh-auth                               | 用于 SSH 身份认证的凭据      |
| kubernetes.io/tls                                    | 用于 TLS 客户端或者服务器端的数据 |
| bootstrap.kubernetes.io/token                        | 启动引导令牌数据            |

Kubernetes 并不对类型的名称作任何限制，因此可以自定义 Secret 类型。

## 查看Secret
使用get命令可以查到Secret 信息，其中Opaque 类型表示base64 编码格式的Secret
```
$ kubectl get secrets 
NAME           TYPE     DATA   AGE
db-user-pass   Opaque   2      15s
```

通过describe 命令可以发现Secret 与 ConfigMap不同，不会直接显示内容
```
$ kubectl describe secrets db-user-pass 
Name:         db-user-pass
Namespace:    default
Labels:       <none>
Annotations:  <none>
 
Type:  Opaque
 
Data
====
username.txt:  5 bytes
password.txt:  4 bytes
```

## 使用Secret
Pod有三种方式使用Secret：
- 作为挂载到一个或多个容器上的卷 中的文件
- 作为容器的环境变量
- 由 kubelet 在为 Pod 拉取镜像时使用

### 作为挂载到卷中的文件
通ConfigMap一样，Secret也可以挂载到卷中使用：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
    - name: secret-container
      image: busybox
      command: ["/bin/sh", "-c",  "sleep 120"]
      volumeMounts:
        - name: my-secret-volume
          mountPath: /etc/config
  volumes:
    - name: my-secret-volume
      secret:
        secretName: file-secret
  restartPolicy: Never
```

### 作为容器的环境变量
同ConfigMap一样，容器的环境变量值也可引用自Secret：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
    - name: secret-env-container
      image: busybox
      command: ["/bin/sh", "-c",  "echo usename: ${USERNAME}, password: ${PASSWORD}"]
      env:
        - name: USERNAME
          valueFrom:
            secretKeyRef:
              name: file-secret
              key: username
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: file-secret
              key: password
  restartPolicy: Never
```
运行kubectl logs secret-env-pod则会看到
```
usename: YWRtaW4= , password: YWRtaW4xMjM=
```