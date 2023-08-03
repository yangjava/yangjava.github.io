---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes源码存储核心
Kubernetes系统使用Etcd作为Kubernetes集群的唯一存储，Etcd在生产环境中一般以集群形式部署（称为Etcd集群）。

Etcd集群是分布式K/V存储集群，提供了可靠的强一致性服务发现。Etcd集群存储Kubernetes系统的集群状态和元数据，其中包括所有Kubernetes资源对象信息、资源对象状态、集群节点信息等。Kubernetes将所有数据存储至Etcd集群前缀为/registry的目录下。

## Etcd存储架构设计
Kubernetes系统对Etcd存储进行了大量封装，其架构是分层的，而每一层的封装设计又拥有高度的可扩展性。

Etcd存储架构设计说明如下。
- RESTStorage
实现了RESTful风格的对外资源存储服务的API接口。
- RegistryStore
实现了资源存储的通用操作，例如，在存储资源对象之前执行某个函数（即Before Func），在存储资源对象之后执行某个函数（即After Func）。
- Storage.Interface
通用存储接口，该接口定义了资源的操作方法（即Create、Delete、Watch、WatchList、Get、GetToList、List、GuaranteedUpdate、Count、Versioner等方法）。
- CacherStorage
带有缓存功能的资源存储对象，它是Storage.Interface通用存储接口的实现。CacherStorage缓存层的设计有利于Etcd集群中的数据能够获得快速的响应，并与Etcd集群数据保持一致。
- UnderlyingStorage
底层存储，也被称为BackendStorage（后端存储），是真正与Etcd集群交互的资源存储对象，CacherStorage相当于UnderlyingStorage的缓存层。UnderlyingStorage同样也是Storage.Interface通用存储接口的实现。

## RESTStorage存储服务通用接口
Kubernetes的每种资源（包括子资源）都提供了RESTful风格的对外资源存储服务API接口（即RESTStorage接口），所有通过RESTful API对外暴露的资源都必须实现RESTStorage接口。RESTStorage接口代码示例如下：

代码路径：vendor/k8s.io/apiserver/pkg/registry/rest/rest.go
```
type Storage interface {
    New() runtime.Object
}
```
Kubernetes的每种资源实现的RESTStorage接口一般定义在pkg/registry/<资源组>/<资源>/storage/storage.go中，它们通过NewStorage函数或NewREST函数实例化。以Deployment资源为例，代码示例如下：

代码路径：pkg/registry/apps/deployment/storage/storage.go
```

```
在以上代码中，Deployment资源定义了REST数据结构与StatusREST数据结构，其中REST数据结构用于实现deployment资源的RESTStorage接口，而StatusREST数据结构用于实现deployment/status子资源的RESTStorage接口。

每一个RESTStorage接口都对RegistryStore操作进行了封装，例如，对deployment/status子资源进行Get操作时，实际执行的是RegistryStore操作，代码示例如下：
```

```

## RegistryStore存储服务通用操作
RegistryStore实现了资源存储的通用操作，例如，在存储资源对象之前执行某个函数，在存储资源对象之后执行某个函数。

当通过RegistryStore存储了一个资源对象时，RegistryStore中定义了如下两种函数。
- Before Func ：也称Strategy预处理，它被定义为在创建资源对象之前调用，做一些预处理工作。
- After Func ：它被定义为在创建资源对象之后调用，做一些收尾工作。

每种资源的特殊化存储需求可以定义在Before Func和After Func中，但目前在Kubernetes系统中并未使用After Func功能。RegistryStore结构如下：

代码路径：vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go
```

```
RegistryStore定义了4种Strategy预处理方法，分别是CreateStrategy（创建资源对象时的预处理操作）、UpdateStrategy （更新资源对象时的预处理操作）、DeleteStrategy（删除资源对象时的预处理操作）、ExportStrategy（导出资源对象时的预处理操作）。更多关于Strategy预处理的内容，请参考6.8节“Strategy预处理”。

另外，RegistryStore定义了3种创建资源对象后的处理方法，分别是AfterCreate （创建资源对象后的处理操作）、AfterUpdate （更新资源对象后的处理操作）、AfterDelete（删除资源对象后的处理操作）。

最后，Storage字段是RegistryStore对Storage.Interface通用存储接口进行的封装，实现了对Etcd集群的读/写操作。以RegistryStore的Create方法（创建资源对象的方法）为例，代码示例如下：

代码路径：vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go
```

```
Create方法创建资源对象的过程可分为3步：第1步，通过rest.BeforeCreate函数执行预处理操作；第2步，通过s.Storage.Create函数创建资源对象；第3步，通过e.AfterCreate函数执行收尾操作。

## Storage.Interface通用存储接口
Storage.Interface通用存储接口定义了资源的操作方法，代码示例如下：

代码路径：vendor/k8s.io/apiserver/pkg/storage/interfaces.go
```

```
Storage.Interface接口字段说明如下。
- Versioner ：资源版本管理器，用于管理Etcd集群中的数据版本对象。
- Create ：创建资源对象的方法。
- Delete ：删除资源对象的方法。
- Watch ：通过Watch机制监控资源对象变化方法，只应用于单个key。
- WatchList ：通过Watch机制监控资源对象变化方法，应用于多个key（当前目录及目录下所有的key）。
- Get ：获取资源对象的方法。
- GetToList ：获取资源对象的方法，以列表（List）的形式返回。
- List ：获取资源对象的方法，以列表（List）的形式返回。
- GuaranteedUpdate ：保证传入的tryUpdate函数运行成功。
- Count ：获取指定key下的条目数量。

Storage.Interface是通用存储接口，实现通用存储接口的分别是CacherStorage资源存储对象和UnderlyingStorage资源存储对象，分别介绍如下。
- CacherStorage ：带有缓存功能的资源存储对象，它定义在vendor/k8s.io/apiserver/pkg/storage/cacher/cacher.go中。
- UnderlyingStorage ：底层存储对象，真正与Etcd集群交互的资源存储对象，它定义在vendor/k8s.io/apiserver/pkg/storage/etcd3/store.go中。

下面介绍CacherStorage与UnderlyingStorage的实例化过程，代码示例如下：

代码路径：vendor/k8s.io/apiserver/pkg/server/options/etcd.go
```

```
如果不启用WatchCache功能，Kubernetes API Server通过generic.UndecoratedStorage函数直接创建UnderlyingStorage底层存储对象并返回。在默认的情况下，Kubernetes API Server的缓存功能是开启的，可通过--watch-cache参数设置，如果该参数为true，则通过genericregistry.StorageWithCacher函数创建带有缓存功能的资源存储对象。

CacherStorage实际上是在UnderlyingStorage之上封装了一层缓存层，在genericregistry.StorageWithCacher函数实例化的过程中，也会创建UnderlyingStorage底层存储对象。

CacherStorage的实例化过程基于装饰器模式，下面介绍一下装饰器模式。装饰器模式将函数装饰到另一些函数上，这样可以让代码看起来更简练，也可以让一些函数的代码复用性更高。

在Go语言中，装饰器模式的代码形式如下：
```

```
首先，装饰器基本上是一个函数，它将特定类型的函数作为参数，并返回相同类型的函数。ToLower函数是一个装饰器函数，ToLower的函数返回值与其内部函数返回值是相同的类型。

在CacherStorage的实例化过程中，在装饰器函数里实现了UnderlyingStorage和CacherStorage的实例化过程，代码示例如下：

代码路径：vendor/k8s.io/apiserver/pkg/registry/generic/registry/storage_factory.go
```

```

## CacherStorage缓存层
缓存（Cache）的应用场景非常广泛，可以使用缓存来降低数据库服务器的负载、减少连接数等。例如，将某些业务需要读/写的数据库服务器中的一些数据存储到缓存服务器中，然后这部分业务就可以直接通过缓存服务器进行数据读/写，这样可以加快数据库的请求响应时间。

在生产环境下，我们常将Memcached或Redis等作为MySQL的缓存层对外提供服务，缓存层中的数据存储于内存中，所以大大提升了数据库的I/O响应能力。

CacherStorage缓存层为DB数据层提供缓存服务，目的是快速响应并减轻DB数据层的压力。图6-4中提供了缓存的两种应用场景，左边为缓存命中，右边为缓存回源。
- 缓存命中 ：客户端发起请求，请求中的数据存于缓存层中，则从缓存层直接返回数据。
- 缓存回源 ：客户端发起请求，请求中的数据未存于缓存层中，此时缓存层向DB数据层获取数据（该过程被称为回源），DB数据层将数据返回给缓存层，缓存层收到数据并将数据更新到自身缓存中（以便下次客户端请求时提升缓存命中率），最后将数据返回给客户端。

### CacherStorage缓存层设计
CacherStorage缓存层的设计有利于快速响应请求并返回所需的数据，这样可以减少Etcd集群的连接数，返回的数据也与Etcd集群中的数据保持一致。

CacherStoraeg缓存层并非会为所有操作都缓存数据。对于某些操作，为保证数据一致性，没有必要在其上再封装一层缓存层，例如Create、Delete、Count等操作，通过UnderlyingStorage直接向Etcd集群发出请求即可。只有Get、GetToList、List、GuaranteedUpdate、Watch、WatchList等操作是基于缓存设计的。其中Watch操作的事件缓存机制（即watchCache）使用缓存滑动窗口来保证历史事件不会丢失，设计较为巧妙。

Actor（它可以是kubectl、Kubernetes组件等）作为客户端向CacherStorage缓存层发送Watch请求。CacherStorage缓存层可分为如下部分。
- cacheWatcher ：Watcher观察者管理。
- watchCache ：通过Reflector框架与UnderlyingStorage底层存储对象交互，UnderlyingStorage与Etcd集群进行交互，并将回调事件分别存储至w.onEvent、w.cache、cache.Store中。
- Cacher ：用于分发给目前所有已连接的观察者，分发过程通过非阻塞（Non-Blocking）机制实现。

1.cacheWatcher

每一个发送Watch请求的客户端都会分配一个cacheWatcher，用于客户端接收Watch事件，代码示例如下：

代码路径：vendor/k8s.io/apiserver/pkg/storage/cacher/cacher.go

## ResourceVersion资源版本号
所有Kubernetes资源都有一个资源版本号（ResourceVersion），其用于表示资源存储版本，一般定义于资源的元数据中。

每次在修改Etcd集群存储的资源对象时，Kubernetes API Server都会更改ResourceVersion，使得client-go执行Watch操作时可以根据ResourceVersion来确定资源对象是否发生变化。当client-go断开时，只要从上一次的ResourceVersion继续监控（Watch操作），就能够获取历史事件，这样可以防止事件遗漏或丢失。

Kubernetes API Server对ResourceVersion资源版本号并没有实现一套管理机制，而是依赖于Etcd集群中的全局Index机制来进行管理的。在Etcd集群中，有两个比较关键的Index，分别是createdIndex和modifiedIndex，它们用于跟踪Etcd集群中的数据发生了什么。
- createdIndex ：全局唯一且递增的正整数。每次在Etcd集群中创建key时其会递增。
- modifiedIndex ：与createdIndex功能类似，每次在Etcd集群中修改key时其会递增。

createdIndex和modifiedIndex都是原子操作，其中modifiedIndex机制被Kubernetes系统用于获取资源版本号（ResourceVersion）。Kubernetes系统通过资源版本号的概念来实现乐观并发控制，也称乐观锁（Optimistic Concurrency Control）。它是一种并发控制的方法，当处理多客户端并发的事务时，事物之间不会互相影响，各事务能够在不产生锁的情况下处理各自影响的那部分数据。在提交更新数据之前，每个事务都会先检查在自己读取数据之后，有没有其他事务又修改了该数据。如果其他事务有更新过数据，那么正在提交数据的事务会进行回滚。

## watchCache缓存滑动窗口
前面介绍了watchCache，它接收Reflector框架的事件回调，并将事件存储至3个地方，其中有一个地方就是w.cache（即缓存滑动窗口），其提供了对Watch操作的缓存数据（事件的历史数据），防止客户端因网络或其他原因连接中断，导致事件丢失。在介绍缓存滑动窗口之前，先了解一下目前常用的一些缓存算法。

在生产环境中，缓存算法的实现有很多种方案，常用的缓存算法有FIFO、LRU、LFU等。不同的缓存算法有不同的特点和应用场景，分别介绍如下。

1.FIFO(First Input First Output)

● 特点 ：即先进先出，实现简单。

● 数据结构 ：队列。

● 淘汰原则 ：当缓存满的时候，将最先进入缓存的数据淘汰。

2.LRU(Least Recently Used)

● 特点 ：即最近最少使用，优先移除最久未使用的数据，按时间维度衡量。

● 数据结构 ：链表和HashMap。

● 淘汰原则 ：根据缓存数据使用的时间，将最不经常使用的缓存数据优先淘汰。如果一个数据在最近一段时间内都没有被访问，那么在将来它被访问的可能性也很小。

3.LFU(Least Frequently Used)

● 特点 ：即最近最不常用，优先移除访问次数最少的数据，按统计维度衡量。

● 数据结构 ：数组、HashMap和堆。

● 淘汰原则 ：根据缓存数据使用的次数，将访问次数最少的缓存数据优先淘汰。如果一个数据在最近一段时间内使用次数很少，那么在将来被使用的可能性也很小。

watchCache使用缓存滑动窗口（即数组）保存事件，其功能有些类似于FIFO队列，但实现方式不同，其数据结构如下：

代码路径：vendor/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go

## UnderlyingStorage底层存储对象
UnderlyingStorage（底层存储），也称为BackendStorage（后端存储），是真正与Etcd集群交互的资源存储对象，CacherStorage相当于UnderlyingStorage的缓存层。数据回源操作最终还是由UnderlyingStorage代替执行的。

UnderlyingStorage对Etcd的官方库进行了封装，早期版本的Kubernetes系统中同时支持Etcd v2与Etcd v3。但当前Kubernetes系统已经将Etcd v2抛弃了，只支持Etcd v3，因为Etcd v3的性能和通信方式都优于Etcd v2。UnderlyingStorage通过newETCD3Storage函数进行实例化，代码示例如下：

代码路径：vendor/k8s.io/apiserver/pkg/storage/storagebackend/factory/factory.go

## Codec编解码数据
Kubernetes系统将资源对象以二进制形式存储在Etcd集群中。在Kubernetes系统外部，通过Codec Example代码示例直接编/解码Etcd集群中的资源对象数据，理解会更深刻。
Codec Example代码示例如下：
```

```
在Codec Example代码示例中
- 实例化Scheme资源注册表及Codecs编解码器，并通过init函数将corev1资源组下的资源注册至Scheme资源注册表中，这是因为要对Pod资源数据进行解码操作。inMediaType定义了编码类型（即Protobuf格式），outMediaType定义了解码类型（即JSON格式）。
- 通过clientv3.New函数实例化Etcd Client对象，并设置一些参数，例如将Endpoints参数连接至Etcd集群的地址，将DialTimeout参数连接至集群的超时时间等。通过cli.Get函数获取Etcd集群中/registry/pods/default/centos-59db99c6bc-m6tsz下的Pod资源对象数据。
- 通过newCodec函数实例化runtime.Codec编解码器，分别实例化inCodec编码器对象、outCodec解码器对象。
- 通过runtime.Decode解码器（即protobufSerializer）解码资源对象数据并通过fmt.Println函数输出。
- 通过runtime.Encode编码器（即jsonSerializer）解码资源对象数据并通过fmt.Println函数输出。

## Strategy预处理
在Kubernetes系统中，每种资源都有自己的预处理（Strategy）操作，其用于资源存储对象创建（Create）、更新（Update）、删除（Delete）、导出（Export）资源对象之前对资源执行预处理操作，例如在存储资源对象之前验证或者修改该对象。每个资源的特殊需求都可以在自己的Strategy预处理接口中实现。每个资源的Strategy预处理接口代码实现一般定义在pkg/registry/<资源组>/<资源>/strategy.go中。

Strategy预处理接口定义如下：

代码路径：vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go
```

```

Strategy预处理接口字段说明如下。
- CreateStrategy ：创建资源对象时的预处理操作。
- UpdateStrategy ：更新资源对象时的预处理操作。
- DeleteStrategy ：删除资源对象时的预处理操作。
- ExportStrategy ：导出资源对象时的预处理操作。

## 创建资源对象时的预处理操作
CreateStrategy是创建资源对象时的预处理操作，它提供了BeforeCreate函数，用于在创建资源对象之前执行该操作。CreateStrategy操作的接口定义如下：

代码路径：vendor/k8s.io/apiserver/pkg/registry/rest/create.go