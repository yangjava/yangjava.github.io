---
layout: post
categories: [Kubernetes]
description: none
keywords: Kubernetes
---
# Kubernetes共享存储

## 共享存储机制概述
Kubernetes对于有状态的容器应用或者对数据需要持久化的应用，不仅需要将容器内的目录挂载到宿主机的目录或者emptyDir临时存储卷，而且需要更加可靠的存储来保存应用产生的重要数据，以便容器应用在重建之后仍然可以使用之前的数据。不过，存储资源和计算资源（CPU/内存）的管理方式完全不同。为了能够屏蔽底层存储实现的细节，让用户方便使用，同时让管理员方便管理，Kubernetes从1.0版本就引入PersistentVolume（PV）和PersistentVolumeClaim（PVC）两个资源对象来实现对存储的管理子系统。

PV是对底层网络共享存储的抽象，将共享存储定义为一种“资源”，比如Node也是一种容器应用可以“消费”的资源。PV由管理员创建和配置，它与共享存储的具体实现直接相关，例如GlusterFS、iSCSI、RBD或GCE或AWS公有云提供的共享存储，通过插件式的机制完成与共享存储的对接，以供应用访问和使用。

PVC则是用户对存储资源的一个“申请”。就像Pod“消费”Node的资源一样，PVC能够“消费”PV资源。PVC可以申请特定的存储空间和访问模式。

使用PVC“申请”到一定的存储空间仍然不能满足应用对存储设备的各种需求。通常应用程序都会对存储设备的特性和性能有不同的要求，包括读写速度、并发性能、数据冗余等更高的要求，Kubernetes从1.4版本开始引入了一个新的资源对象StorageClass，用于标记存储资源的特性和性能。到1.6版本时，StorageClass和动态资源供应的机制得到了完善，实现了存储卷的按需创建，在共享存储的自动化管理进程中实现了重要的一步。

通过StorageClass的定义，管理员可以将存储资源定义为某种类别（Class），正如存储设备对于自身的配置描述（Profile），例如“快速存储”“慢速存储”“有数据冗余”“无数据冗余”等。用户根据StorageClass的描述就能够直观地得知各种存储资源的特性，就可以根据应用对存储资源的需求去申请存储资源了。

Kubernetes从1.9版本开始引入容器存储接口Container Storage Interface（CSI）机制，目标是在Kubernetes和外部存储系统之间建立一套标准的存储管理接口，通过该接口为容器提供存储服务，类似于CRI（容器运行时接口）和CNI（容器网络接口）。

下面对Kubernetes的PV、PVC、StorageClass、动态资源供应和CSI等共享存储管理机制进行详细说明。

## PV详解
PV作为存储资源，主要包括存储能力、访问模式、存储类型、回收策略、后端存储类型等关键信息的设置。

下面的例子声明的PV具有如下属性：5GiB存储空间，访问模式为ReadWriteOnce，存储类型为slow（要求在系统中已存在名为slow的StorageClass），回收策略为Recycle，并且后端存储类型为nfs（设置了NFS Server的IP地址和路径）：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  nfs:
    path: /tmp
    server: 172.17.0.2
```
Kubernetes支持的PV类型如下。
- AWSElasticBlockStore：AWS公有云提供的ElasticBlockStore。
- AzureFile：Azure公有云提供的File。
- AzureDisk：Azure公有云提供的Disk。
- CephFS：一种开源共享存储系统。
- FC（Fibre Channel）：光纤存储设备。
- FlexVolume：一种插件式的存储机制。
- Flocker：一种开源共享存储系统。
- GCEPersistentDisk：GCE公有云提供的PersistentDisk。
- Glusterfs：一种开源共享存储系统。
- HostPath：宿主机目录，仅用于单机测试。
- iSCSI：iSCSI存储设备。
- Local：本地存储设备，从Kubernetes 1.7版本引入，到1.14版本时更新为稳定版，目前可以通过指定块（Block）设备提供Local PV，或通过社区开发的sig-storage-local-static-provisioner插件（https://github.com/kubernetes-sigs/sigstorage-local-static-provisioner）来管理Local PV的生命周期。
- NFS：网络文件系统。
- Portworx Volumes：Portworx提供的存储服务。
- Quobyte Volumes：Quobyte提供的存储服务。
- RBD（Ceph Block Device）：Ceph块存储。
- ScaleIO Volumes：DellEMC的存储设备。
- StorageOS：StorageOS提供的存储服务。
- VsphereVolume：VMWare提供的存储系统。

每种存储类型都有各自的特点，在使用时需要根据它们各自的参数进行设置。

## PV的关键配置参数

### 存储能力（Capacity）
描述存储设备具备的能力，目前仅支持对存储空间的设置（storage=xx），未来可能加入IOPS、吞吐率等指标的设置。

### 存储卷模式（Volume Mode）
Kubernetes从1.13版本开始引入存储卷类型的设置（volumeMode=xxx），可选项包括Filesystem（文件系统）和Block（块设备），默认值为Filesystem。

目前有以下PV类型支持块设备类型：
- AWSElasticBlockStore
- AzureDisk
- FC
- GCEPersistentDisk
- iSCSI
- Local volume
- RBD（Ceph Block Device）
- VsphereVolume（alpha）

下面的例子为使用块设备的PV定义：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block_pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: slow
  PersistentVolumeReclaimPolicy: Retain
  volumeMode: block
  fc:
    targetWWNs: ["xxxxxx"]
    lun: 0
    readOnly: false
```
### 访问模式（Access Modes）
对PV进行访问模式的设置，用于描述用户的应用对存储资源的访问权限。访问模式如下。
- ReadWriteOnce（RWO）：读写权限，并且只能被单个Node挂载。
- ReadOnlyMany（ROX）：只读权限，允许被多个Node挂载。
- ReadWriteMany（RWX）：读写权限，允许被多个Node挂载。

某些PV可能支持多种访问模式，但PV在挂载时只能使用一种访问模式，多种访问模式不能同时生效。

### 存储类别（Class）
PV可以设定其存储的类别，通过storageClassName参数指定一个StorageClass资源对象的名称。具有特定类别的PV只能与请求了该类别的PVC进行绑定。未设定类别的PV则只能与不请求任何类别的PVC进行绑定。

### 回收策略（Reclaim Policy）
通过PV定义中的persistentVolumeReclaimPolicy字段进行设置，可选项如下。
- 保留(Retain):
保留数据，需要手工处理。
- 回收空间( Recycle)
简单清除文件的操作（例如执行rm -rf /thevolume/*命令）。
- 删除(Delete)
与PV相连的后端存储完成Volume的删除操作（如AWS EBS、GCE PD、Azure Disk、OpenStack Cinder等设备的内部Volume清理）。

目前，只有NFS和HostPath两种类型的存储支持Recycle策略；AWS EBS、GCE PD、Azure Disk和Cinder volumes支持Delete策略。

### 挂载参数（Mount Options）
在将PV挂载到一个Node上时，根据后端存储的特点，可能需要设置额外的挂载参数，可以根据PV定义中的mountOptions字段进行设置。

下面的例子为对一个类型为gcePersistentDisk的PV设置挂载参数：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
name: gce-disk-1
spec:
  capacity:
  storage: 5Gi
  accessModes:
    - ReadWriteOnce
  mountOptions:
    - hard
    - nolock
    - nfsvers=3
  gcePersistentDisk:
  fsType: "ext4"
  pdName: "gce-disk-1"
```

目前，以下PV类型支持设置挂载参数：
- AWSElasticBlockStore
- AzureDisk
- AzureFile
- CephFS
- Cinder (OpenStack block storage)
- GCEPersistentDisk
- Glusterfs
- NFS
- Quobyte Volumes
- RBD (Ceph Block Device)
- StorageOS
- VsphereVolume
- iSCSI

### 节点亲和性（Node Affinity）
PV可以设置节点亲和性来限制只能通过某些Node访问Volume，可以在PV定义中的nodeAffinity字段进行设置。使用这些Volume的Pod将被调度到满足条件的Node上。

这个参数仅用于Local存储卷，例如：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-local-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /test/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kuberneter.io/hostname
          operator: In
          values:
          - master
```
公有云提供的存储卷（如AWS EBS、GCE PD、Azure Disk等）都由公有云自动完成节点亲和性设置，无须用户手工设置。

## PV生命周期的各个阶段
某个PV在生命周期中可能处于以下4个阶段（Phaes）之一。
- Available
可用状态，还未与某个PVC绑定。
- Bound
已与某个PVC绑定。
- Released
绑定的PVC已经删除，资源已释放，但没有被集群回收。
- Failed
自动资源回收失败。

定义了PV以后如何使用呢？这时就需要用到PVC了。

## PVC详解
PVC作为用户对存储资源的需求申请，主要包括存储空间请求、访问模式、PV选择条件和存储类别等信息的设置。

下例声明的PVC具有如下属性：申请8GiB存储空间，访问模式为ReadWriteOnce，PV 选择条件为包含标签“release=stable”并且包含条件为“environment In　[dev]”的标签，存储类别为“slow”（要求在系统中已存在名为slow的StorageClass）：
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
     - {key: environment, operator: In, values: [dev]}
```
PVC的关键配置参数说明如下。
- 资源请求（Resources）：描述对存储资源的请求，目前仅支持request.storage的设置，即存储空间大小。
- 访问模式（Access Modes）：PVC也可以设置访问模式，用于描述用户应用对存储资源的访问权限。其三种访问模式的设置与PV的设置相同。
- 存储卷模式（Volume Modes）：PVC也可以设置存储卷模式，用于描述希望使用的PV存储卷模式，包括文件系统和块设备。
- PV选择条件（Selector）：通过对Label Selector的设置，可使PVC对于系统中已存在的各种PV进行筛选。系统将根据标签选出合适的PV与该PVC进行绑定。选择条件可以使用matchLabels和matchExpressions进行设置，如果两个字段都设置了，则Selector的逻辑将是两组条件同时满足才能完成匹配。
- 存储类别（Class）：PVC 在定义时可以设定需要的后端存储的类别（通过storageClassName字段指定），以减少对后端存储特性的详细信息的依赖。只有设置了该Class的PV才能被系统选出，并与该PVC进行绑定。

PVC也可以不设置Class需求。如果storageClassName字段的值被设置为空（storageClassName=""），则表示该PVC不要求特定的Class，系统将只选择未设定Class的PV与之匹配和绑定。PVC也可以完全不设置storageClassName字段，此时将根据系统是否启用了名为DefaultStorageClass的admission controller进行相应的操作。
- 未启用DefaultStorageClass：等效于PVC设置storageClassName的值为空（storageClassName=""），即只能选择未设定Class的PV与之匹配和绑定。
- 启用DefaultStorageClass：要求集群管理员已定义默认的StorageClass。如果在系统中不存在默认的StorageClass，则等效于不启用DefaultStorageClass的情况。如果存在默认的StorageClass，则系统将自动为PVC创建一个PV（使用默认StorageClass的后端存储），并将它们进行绑定。集群管理员设置默认StorageClass的方法为，在StorageClass的定义中加上一个annotation“storageclass.kubernetes.io/is-default-class= true”。如果管理员将多个StorageClass都定义为default，则由于不唯一，系统将无法为PVC创建相应的PV。

注意，PVC和PV都受限于Namespace，PVC在选择PV时受到Namespace的限制，只有相同Namespace中的PV才可能与PVC绑定。Pod在引用PVC时同样受Namespace的限制，只有相同Namespace中的PVC才能挂载到Pod内。

当Selector和Class都进行了设置时，系统将选择两个条件同时满足的PV与之匹配。

另外，如果资源供应使用的是动态模式，即管理员没有预先定义PV，仅通过StorageClass交给系统自动完成PV的动态创建，那么PVC再设定Selector时，系统将无法为其供应任何存储资源。

在启用动态供应模式的情况下，一旦用户删除了PVC，与之绑定的PV也将根据其默认的回收策略“Delete”被删除。如果需要保留PV（用户数据），则在动态绑定成功后，用户需要将系统自动生成PV的回收策略从“Delete”改成“Retain”。

## PV和PVC的生命周期
我们可以将PV看作可用的存储资源，PVC则是对存储资源的需求。

## 资源供应
Kubernetes支持两种资源的供应模式：静态模式（Static）和动态模式（Dynamic）。资源供应的结果就是创建好的PV。
- 静态模式： 集群管理员手工创建许多PV，在定义PV时需要将后端存储的特性进行设置。
- 动态模式： 集群管理员无须手工创建PV，而是通过StorageClass的设置对后端存储进行描述，标记为某种类型。此时要求PVC对存储的类型进行声明，系统将自动完成PV的创建及与PVC的绑定。PVC可以声明Class为""，说明该PVC禁止使用动态模式。

## 资源绑定
在用户定义好PVC之后，系统将根据PVC对存储资源的请求（存储空间和访问模式）在已存在的PV中选择一个满足PVC要求的PV，一旦找到，就将该PV与用户定义的PVC进行绑定，用户的应用就可以使用这个PVC了。如果在系统中没有满足PVC要求的PV，PVC则会无限期处于Pending状态，直到等到系统管理员创建了一个符合其要求的PV。PV一旦绑定到某个PVC上，就会被这个PVC独占，不能再与其他PVC进行绑定了。在这种情况下，当PVC申请的存储空间比PV的少时，整个PV的空间就都能够为PVC所用，可能会造成资源的浪费。如果资源供应使用的是动态模式，则系统在为PVC找到合适的StorageClass后，将自动创建一个PV并完成与PVC的绑定。

## 资源使用
Pod使用Volume的定义，将PVC挂载到容器内的某个路径进行使用。Volume的类型为persistentVolumeClaim，在后面的示例中再进行详细说明。在容器应用挂载了一个PVC后，就能被持续独占使用。不过，多个Pod可以挂载同一个PVC，应用程序需要考虑多个实例共同访问一块存储空间的问题。

## 资源释放
当用户对存储资源使用完毕后，用户可以删除PVC，与该PVC绑定的PV将会被标记为“已释放”，但还不能立刻与其他PVC进行绑定。通过之前PVC写入的数据可能还被留在存储设备上，只有在清除之后该PV才能再次使用。

## 资源回收
对于PV，管理员可以设定回收策略，用于设置与之绑定的PVC释放资源之后如何处理遗留数据的问题。只有PV的存储空间完成回收，才能供新的PVC绑定和使用。回收策略详见下节的说明。

下面通过两张图分别对在静态资源供应模式和动态资源供应模式下，PV、PVC、StorageClass及Pod使用PVC的原理进行说明。

## StorageClass详解
StorageClass作为对存储资源的抽象定义，对用户设置的PVC申请屏蔽后端存储的细节，一方面减少了用户对于存储资源细节的关注，另一方面减轻了管理员手工管理PV的工作，由系统自动完成PV的创建和绑定，实现了动态的资源供应。基于StorageClass的动态资源供应模式将逐步成为云平台的标准存储配置模式。

StorageClass的定义主要包括名称、后端存储的提供者（provisioner）和后端存储的相关参数配置。StorageClass一旦被创建出来，则将无法修改。如需更改，则只能删除原StorageClass的定义重建。下例定义了一个名为standard的StorageClass，提供者为aws-ebs，其参数设置了一个type，值为gp2：
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

## StorageClass的关键配置参数

### 提供者（Provisioner）
描述存储资源的提供者，也可以看作后端存储驱动。目前Kubernetes支持的Provisioner都以“kubernetes.io/”为开头，用户也可以使用自定义的后端存储提供者。为了符合StorageClass的用法，自定义Provisioner需要符合存储卷的开发规范，详见https://github.com/kubernetes/community/blob/master/contributors/design-proposals/volume-provisioning.md的说明。

### 参数（Parameters）
后端存储资源提供者的参数设置，不同的Provisioner包括不同的参数设置。某些参数可以不显示设定，Provisioner将使用其默认值。

接下来通过几种常见的Provisioner对StorageClass的定义进行详细说明。