---
layout: post
categories: [Kubernetes]
description: none
keywords: Kubernetes
---
# Kubernetes配置管理ConfigMap
应用部署的一个最佳实践是将应用所需的配置信息与程序进行分离，这样可以使应用程序被更好地复用，通过不同的配置也能实现更灵活的功能。将应用打包为容器镜像后，可以通过环境变量或者外挂文件的方式在创建容器时进行配置注入，但在大规模容器集群的环境中，对多个容器进行不同的配置将变得非常复杂。从Kubernetes 1.2开始提供了一种统一的应用配置管理方案 ConfigMap。

## ConfigMap（配置地图）介绍
当开发人员开发完成一个应用程序，比如一个Web程序，在正式上线之前需要在各种环境中运行，例如开发时的开发环境，测试环节的测试环境，直到最终的线上环境。Web程序在各种不同的环境中都需要对接不同的数据库、中间件等服务，在传统方式中我们通过配置文件来定义这些配置，而kubernetes中如果需要进入一个Pod来配置，那将会是一个巨大的麻烦

## ConfigMap 的功能
- ConfigMap 用于容器的配置文件管理。它作为多个properties文件的应用，类似一个专门存储配置文件的目录，里面存放着各种配置文件
- ConfigMap 实现了 image 和应用程序的配置文件、命令行参数和环境变量等信息解耦（一般用于不敏感的信息，因为是明文）
- ConfigMap 和 Secrets 类似，但ConfigMap用于处理不含敏感信息的配置文件

ConfigMap 是 Kubernetes 用来向应用 Pod 中注入非敏感键值对配置数据的方法。在容器中使用时，Pods 可以将其用作环境变量、命令行参数或者存储卷中的配置文件，但其不是用来保存大量数据的，保存的数据大小不可超过 1 MiB。

## 创建ConfigMap资源对象
使用命令行创建 ConfigMap
```shell
kubectl create configmap ${configMapName}
```
其中dataSource数据源是要从中提取数据的目录、 文件或者字面值。 目录和文件使用--from-file参数指定，字面值即字符串，使用--from-literal指定。

### 基于目录创建 ConfigMap
可以基于目录创建ConfigMap，其结果是将这个目录下的所有文件都保存在一个ConfigMap中，其key为文件的basename，value为UTF-8编码的整个文件内容。

例如，在目录/opt/properties下有两个配置文件game.properties和ui.properties：
```
# game.properties
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
```

```
# ui.properties
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice
```
指定目录/opt/properties创建ConfigMap：
```shell
kubectl create configmap game-config --from-file=/opt/properties
```
这会创建一个名为game-config-dir，key分别为game.properties和ui.properties，value分别为这两个文件中的配置数据（UTF-8 编码字符串），运行kubectl get cm game-config -o yaml可以查看：
```yaml
apiVersion: v1
data:
  game.properties: |+
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30

  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  name: game-config-dir
  namespace: default
```

### 基于文件创建 ConfigMap
可以指定单个或多个文件创建ConfigMap，--from-file的值为文件路径，允许同时指定多个文件。例如分别指定上述/opt/properties目录下的game.properties和ui.properties文件创建名为game-config-fie的ConfigMap：
```shell
kubectl create cm game-config-file --from-file=/opt/properties/game.properties --from-file=/opt/properties/ui.properties
```
其结果和上述使用目录创建一样。

### 基于文件创建指定Key的ConfigMap
基于文件创建时，默认的Key名是文件的basename，为方便使用，可以指定Key以覆盖默认名。例如上述基于/opt/properties目录下的game.properties和ui.properties文件创建时，将前者指定为game，后者的Key指定为ui。
```shell
kubectl create configmap game-config --from-file=game=/opt/properties/game.properties --from-file=ui=/opt/properties/ui.properties
```
查看结果为：
```yaml
apiVersion: v1
data:
  game: |+
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30

  ui: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  name: game-config
  namespace: default
```

### 使用字面量创建ConfigMap
可以直接通过--from-literal定义ConfigMap中的键值对：
```shell
kubectl create configmap me --from-literal=name=liuweibing --from-literal=lang=Java
```
创建的ConfigMap结果如下（kubectl get cm me -o yaml）
```yaml
apiVersion: v1
data:
  lang: Java
  name: liuweibing
kind: ConfigMap
metadata:
  name: me
```
## 查看ConfigMap
创建完成后依然可以通过get 或 describe 命令查看ConfigMap
```shell
kubectl get configmaps     #确认资源是否存在
```
或者
```shell
kubectl describe configmaps game-config
```


## 使用ConfigMap
使用 ConfigMap 配置 Pod 中的容器：
- 在容器命令和参数内
- 容器的环境变量
- 在只读卷里面添加一个文件，让应用来读取
- 编写代码在 Pod 中运行，使用 Kubernetes API 来读取 ConfigMap
前两种的定义方式相同，都是将ConfigMap的数据作为容器的环境变量，kubelet 使用 ConfigMap 中的数据启动 Pod 中的容器；第三种需要添加只读卷；第四种方式是直接使用 Kubernetes API，只要 ConfigMap 发生更改， 应用就能够通过订阅来获取更新。

### 作为容器的环境变量
可以使用spec.containers.env指定环境变量名，然后通过valueFrom声明变量值引用自哪个ConfigMap的哪一个key，则这个key对应的value值会被赋给这个环境变量，容器命令及容器应用都可使用这个环境变量。

例如，上面创建的名为me的ConfigMap，可以在容器启动的命令中使用。在Pod的模板配置中可以通过spec.containers.command指定容器在启动时运行的命令。
```yaml
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
    - name: env-pod-container
      image: busybox
      command: ["/bin/sh", "-c",  "echo I am ${NAME}, I use ${LANG}"]
      env:  #环境变量
        - name: NAME
          valueFrom: #值从哪或得
            configMapKeyRef:
              name: me #从此configmap中指定变量获得
              key: name #此变量从哪获得
        - name: LANG
          valueFrom:
            configMapKeyRef:
              name: me
              key: lang
  restartPolicy: Never
```
如需将ConfigMap中的全部配置都作为环境变量，则只需要指定configMapRef.name，而无需指定key。

### 在只读卷里面添加一个文件，让应用来读取
基于文件创建的ConfigMap其数据是文件内容本身，这样的ConfigMap可以以Volume卷的形式挂载到Pod的指定目录下，然后被Pod中运行的容器使用，即ConfigMap卷。spec.volumes声明Volume名称及ConfigMap类型和要添加的ConfigMap名称，spec.containers.volumeMounts指定挂载的卷及挂载路径。
```yaml
# configMapVolumePod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: config-map-volume-pod
spec:
  containers:
    - name: volume-pod-container
      image: busybox
      command: ["/bin/sh", "-c",  "ls /etc/config"]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: game-config
  restartPolicy: Never

```
该Pod中的容器在启动时会列举中/etc/config下的文件，由于已经将game-config ConfigMap的数据填充到config-volume卷，且挂载到了/etc/config目录下，所以在容器启动时会看到ls /etc/config命令执行的结果为game-config对应的game和ui。

为了验证，可以将command修改为
```
command: [“/bin/sh”, “-c”, “cat /etc/config/game”]
```
或者，可以修改command让其处于等待状态：
```
command: [“/bin/sh”, “-c”, “sleep 120”]
```
然后可以通过kubectl exec进入容器执行命令查看文件内容。

### 编写代码在 Pod 中运行，使用 Kubernetes API 来读取 ConfigMap

## 使用ConfigMap的限制条件
使用ConfigMap的限制条件如下。
- ConfigMap必须在Pod之前创建。如果Pod调用ConfigMap失败，则无法创建
- ConfigMap受Namespace限制，只有处于相同Namespace中的Pod才可以引用它。ConfigMap 创建方式通常使用文件方式
- ConfigMap中的配额管理还未能实现。
- kubelet只支持可以被API Server管理的Pod使用ConfigMap。kubelet在本Node上通过 --manifest-url或--config自动创建的静态Pod将无法引用ConfigMap。
- 在Pod对ConfigMap进行挂载（volumeMount）操作时，在容器内部只能挂载为“目录”，无法挂载为“文件”。在挂载到容器内部后，在目录下将包含ConfigMap定义的每个item，如果在该目录下原来还有其他文件，则容器内的该目录将被挂载的ConfigMap覆盖。如果应用程序需要保留原来的其他文件，则需要进行额外的处理。可以将ConfigMap挂载到容器内部的临时目录，再通过启动脚本将配置文件复制或者链接到（cp或link命令）应用所用的实际配置目录下。
- 以volume 方式挂载 ConfigMap，更新ConfigMap 或删除重建 ConfigMap，Pod内挂载的配置信息会热更新
