---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus源码服务发现k8s
prometheus与时俱进在现在各种容器管理平台流行的当下，能够对各种容器管理平台进行数据采集和处理，并且能够自动发现监控对象，这个就是今天要说的discovery。他是一个资源发现的组件，能够根据资源变化调整监控对象。目前已经支持的discovery主要有DNS、kubernetes、marathon、zk、consul、file等，我们先从这个kubernetes的服务发现开始讲解吧

## 服务发现k8s
Discoverer定义的接口类型，不同的服务发现协议基于该接口进行实现：
```
type Discoverer interface {
 // Run hands a channel to the discovery provider (Consul, DNS, etc.) through which
 // it can send updated target groups. It must return when the context is canceled.
 // It should not close the update channel on returning.
 Run(ctx context.Context, up chan<- []*targetgroup.Group)
}
```
## k8s协议配置
Prometheus本身就是作为云原生监控出现的，所以对云原生服务发现支持具有天然优势。kubernetes_sd_configs 服务发现协议核心原理就是利用API Server提供的Rest接口获取到云原生集群中的POD、Service、Node、Endpoints、Endpointslice、Ingress等对象的元数据，并基于这些信息生成Prometheus采集点，并且可以随着云原生集群状态变更进行动态实时刷新。

kubernetes云原生集群的POD、Service、Node、Ingress等对象元数据信息都被存储到etcd数据库中，并通过API Server组件暴露的Rest接口方式提供访问或操作这些对象数据信息。

kubernetes_sd_configs配置示例
```yaml
  - job_name: kubernetes-pod
    kubernetes_sd_configs:
    - role: pod
      namespaces:
        names:
        - 'test01'
      api_server: https://apiserver.simon:6443
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token 
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```
配置说明：
- api_server指定API Server地址，出于安全考虑，这些接口是带有安全认证的，bearer_token_file和ca_file则指定访问API Server使用到的认证信息；
- role指定基于云原生集群中哪种对象类型做服务发现，支持POD、Service、Node、Endpoints、Endpointslice、Ingress六种类型；
- namespaces指定作用于哪个云原生命名空间下的对象，不配置则对所有的云原生命名空间生效；

### 为什么没有配置api server信息也可以正常进行服务发现？

很多时候我们并不需要配置api server相关信息也可以进行服务发现，如我们将上面示例简化如下写法：
```yaml
  - job_name: kubernetes-pod
    kubernetes_sd_configs:
    - role: pod
      namespaces:
        names:
        - 'test01'
```
一般Prometheus部署在监控云原生集群上，从 Pod 使用 Kubernetes API 官方客户端库(client-go)提供了更为简便的方法：rest.InClusterConfig()。 API Server地址是从POD的环境变量KUBERNETES_SERVICE_HOST和KUBERNETES_SERVICE_PORT构建出来， token 以及 ca 信息从POD固定的文件中获取，因此这种场景下kubernetes_sd_configs中api_server和ca_file是不需要配置的。

client-go是kubernetes官方提供的go语言的客户端库，go应用使用该库可以访问kubernetes的API Server，这样我们就能通过编程来对kubernetes资源进行增删改查操作。

## Informer机制
从之前分析的服务发现协议接口设计得知，了解k8s服务发现协议入口在discovery/kubernetes.go的Run方法：

Run方法中switch罗列出不同role的处理逻辑，刚好和配置示例中role支持的六种云原生对象类型对应，只是基于不同的对象进行服务发现，基本原理都是一致的。

云原生服务发现基本原理是访问API Server获取到云原生集群资源对象，Prometheus与API Server进行交互这里使用到的是client-go官方客户端里的Informer核心工具包。Informer底层使用ListWatch机制，在Informer首次启动时，会调用List API获取所有最新版本的资源对象，缓存在内存中，然后再通过Watch API来监听这些对象的变化，去维护这份缓存，降低API Server的负载。除了ListWatch，Informer还可以注册自定义事件处理逻辑，之后如果监听到事件变化就会调用对应的用户自定义事件处理逻辑，这样就实现了用户业务逻辑扩展。

Informer机制本身比较复杂，这里先暂时不太具体说明，只需要理解Prometheus使用Informer机制获取和监听云原生资源对象。

这其中的关键就是注册自定义AddFunc、DeleteFunc和UpdateFunc三种事件处理器，分别对应增、删、改操作，当触发对应操作后，事件处理器就会被回调感知到。比如云原生集群新增一个POD资源对象，则会触发AddFunc处理器，该处理器并不做复杂的业务处理，只是将该对象的key放入到Workqueue队列中，然后Process Item组件作为消费端，不停从Workqueue中提取数据获取到新增POD的key，然后交由Handle Object组件，该组件通过Indexer组件提供的GetByKey()查询到该新增POD的所有元数据信息，然后基于该POD元数据就可以构建采集点信息，这样就实现kubernetes服务发现。

### 为什么需要Workqueue队列？
Resource Event Handlers组件注册自定义事件处理器，获取到事件时只是把对象key放入到Workerqueue中这种简单操作，而没有直接调用Handle Object进行事件处理，这里主要是避免阻塞影响整个informer框架运行。如果Handle Object比较耗时放到Resource Event Handlers组件中直接处理，可能就会影响到④⑤功能，所以这里引入Workqueue类似于MQ功能实现解耦。

## 源码分析
熟悉了上面Informer机制，下面以role=POD为例结合Prometheus源码梳理下上面流程。


```
func New(l log.Logger, conf *config.KubernetesSDConfig) (*Discovery, error) {
    var (
        kcfg *rest.Config
        err  error
    )
    if conf.APIServer.URL == nil {
        // Use the Kubernetes provided pod service account
        // as described in https://kubernetes.io/docs/admin/service-accounts-admin/
        kcfg, err = rest.InClusterConfig()
        if err != nil {
            return nil, err
        }
        // Because the handling of configuration parameters changes
        // we should inform the user when their currently configured values
        // will be ignored due to precedence of InClusterConfig
        l.Info("Using pod service account via in-cluster config")
        if conf.TLSConfig.CAFile != "" {
            l.Warn("Configured TLS CA file is ignored when using pod service account")
        }
        if conf.TLSConfig.CertFile != "" || conf.TLSConfig.KeyFile != "" {
            l.Warn("Configured TLS client certificate is ignored when using pod service account")
        }
        if conf.BearerToken != "" {
            l.Warn("Configured auth token is ignored when using pod service account")
        }
        if conf.BasicAuth != nil {
            l.Warn("Configured basic authentication credentials are ignored when using pod service account")
        }
    } else {
        kcfg = &rest.Config{
            Host: conf.APIServer.String(),
            TLSClientConfig: rest.TLSClientConfig{
                CAFile:   conf.TLSConfig.CAFile,
                CertFile: conf.TLSConfig.CertFile,
                KeyFile:  conf.TLSConfig.KeyFile,
            },
            Insecure: conf.TLSConfig.InsecureSkipVerify,
        }
        token := conf.BearerToken
        if conf.BearerTokenFile != "" {
            bf, err := ioutil.ReadFile(conf.BearerTokenFile)
            if err != nil {
                return nil, err
            }
            token = string(bf)
        }
        kcfg.BearerToken = token

        if conf.BasicAuth != nil {
            kcfg.Username = conf.BasicAuth.Username
            kcfg.Password = conf.BasicAuth.Password
        }
    }

    kcfg.UserAgent = "prometheus/discovery"

    c, err := kubernetes.NewForConfig(kcfg)
    if err != nil {
        return nil, err
    }
    return &Discovery{
        client: c,
        logger: l,
        role:   conf.Role,
    }, nil
}
```
创建的核心就是创建一个c，这个c就是一个kubernetes的client，负责调用kubernetes api的。那他是是怎么做到资源发现的呢？不用说大家应该都能想到之前ingress的设计，对，就是listwatch机制。

```
func (d *Discovery) Run(ctx context.Context, ch chan<- []*config.TargetGroup) {
    rclient := d.client.Core().GetRESTClient()

    switch d.role {
    case "endpoints":
        elw := cache.NewListWatchFromClient(rclient, "endpoints", api.NamespaceAll, nil)
        slw := cache.NewListWatchFromClient(rclient, "services", api.NamespaceAll, nil)
        plw := cache.NewListWatchFromClient(rclient, "pods", api.NamespaceAll, nil)
        eps := NewEndpoints(
            d.logger.With("kubernetes_sd", "endpoint"),
            cache.NewSharedInformer(slw, &apiv1.Service{}, resyncPeriod),
            cache.NewSharedInformer(elw, &apiv1.Endpoints{}, resyncPeriod),
            cache.NewSharedInformer(plw, &apiv1.Pod{}, resyncPeriod),
        )
        go eps.endpointsInf.Run(ctx.Done())
        go eps.serviceInf.Run(ctx.Done())
        go eps.podInf.Run(ctx.Done())

        for !eps.serviceInf.HasSynced() {
            time.Sleep(100 * time.Millisecond)
        }
        for !eps.endpointsInf.HasSynced() {
            time.Sleep(100 * time.Millisecond)
        }
        for !eps.podInf.HasSynced() {
            time.Sleep(100 * time.Millisecond)
        }
        eps.Run(ctx, ch)

    case "pod":
        plw := cache.NewListWatchFromClient(rclient, "pods", api.NamespaceAll, nil)
        pod := NewPod(
            d.logger.With("kubernetes_sd", "pod"),
            cache.NewSharedInformer(plw, &apiv1.Pod{}, resyncPeriod),
        )
        go pod.informer.Run(ctx.Done())

        for !pod.informer.HasSynced() {
            time.Sleep(100 * time.Millisecond)
        }
        pod.Run(ctx, ch)

    case "service":
        slw := cache.NewListWatchFromClient(rclient, "services", api.NamespaceAll, nil)
        svc := NewService(
            d.logger.With("kubernetes_sd", "service"),
            cache.NewSharedInformer(slw, &apiv1.Service{}, resyncPeriod),
        )
        go svc.informer.Run(ctx.Done())

        for !svc.informer.HasSynced() {
            time.Sleep(100 * time.Millisecond)
        }
        svc.Run(ctx, ch)

    case "node":
        nlw := cache.NewListWatchFromClient(rclient, "nodes", api.NamespaceAll, nil)
        node := NewNode(
            d.logger.With("kubernetes_sd", "node"),
            cache.NewSharedInformer(nlw, &apiv1.Node{}, resyncPeriod),
        )
        go node.informer.Run(ctx.Done())

        for !node.informer.HasSynced() {
            time.Sleep(100 * time.Millisecond)
        }
        node.Run(ctx, ch)

    default:
        d.logger.Errorf("unknown Kubernetes discovery kind %q", d.role)
    }

    <-ctx.Done()
}
```

这个里面根据role去启动不同的资源监听，下面以node为例，就是节点事件，譬如添加节点或者删除节点。
```
    n.informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(o interface{}) {
            eventCount.WithLabelValues("node", "add").Inc()

            node, err := convertToNode(o)
            if err != nil {
                n.logger.With("err", err).Errorln("converting to Node object failed")
                return
            }
            send(n.buildNode(node))
        },
        DeleteFunc: func(o interface{}) {
            eventCount.WithLabelValues("node", "delete").Inc()

            node, err := convertToNode(o)
            if err != nil {
                n.logger.With("err", err).Errorln("converting to Node object failed")
                return
            }
            send(&config.TargetGroup{Source: nodeSource(node)})
        },
        UpdateFunc: func(_, o interface{}) {
            eventCount.WithLabelValues("node", "update").Inc()

            node, err := convertToNode(o)
            if err != nil {
                n.logger.With("err", err).Errorln("converting to Node object failed")
                return
            }
            send(n.buildNode(node))
        },
```
针对不同事件的方法。当添加节点的时候，通过buildNode创建targetgroup如下：
```
func (n *Node) buildNode(node *apiv1.Node) *config.TargetGroup {
    tg := &config.TargetGroup{
        Source: nodeSource(node),
    }
    tg.Labels = nodeLabels(node)

    addr, addrMap, err := nodeAddress(node)
    if err != nil {
        n.logger.With("err", err).Debugf("No node address found")
        return nil
    }
    addr = net.JoinHostPort(addr, strconv.FormatInt(int64(node.Status.DaemonEndpoints.KubeletEndpoint.Port), 10))

    t := model.LabelSet{
        model.AddressLabel:  lv(addr),
        model.InstanceLabel: lv(node.Name),
    }

    for ty, a := range addrMap {
        ln := strutil.SanitizeLabelName(nodeAddressPrefix + string(ty))
        t[model.LabelName(ln)] = lv(a[0])
    }
    tg.Targets = append(tg.Targets, t)

    return tg
}
```
这个里面先是创建一个targetgroup，他是target的组，每个组都有自己标签tg.Labels，每个组都有自己的成员tg.Targets。再把这个tg返回到discovery，这样discovery就可以感知资源变化，那么谁来调用呢？

其实很容易理解，这个discovery是干嘛的，就是动态更新target嘛，prometheus谁来维护呢？答案就是TargetManager
```
type TargetManager struct {
    appender      storage.SampleAppender
    scrapeConfigs []*config.ScrapeConfig

    mtx    sync.RWMutex
    ctx    context.Context
    cancel func()
    wg     sync.WaitGroup

    // Set of unqiue targets by scrape configuration.
    targetSets map[string]*targetSet
}
```
这个里面有一个targetSets的属性，它的key是job的命名，value是targetSet，
```
func (ts *TargetSet) Run(ctx context.Context) {
Loop:
    for {
        // Throttle syncing to once per five seconds.
        select {
        case <-ctx.Done():
            break Loop
        case p := <-ts.providerCh:
            ts.updateProviders(ctx, p)
        case <-time.After(5 * time.Second):
        }

        select {
        case <-ctx.Done():
            break Loop
        case <-ts.syncCh:
            ts.sync()
        case p := <-ts.providerCh:
            ts.updateProviders(ctx, p)
        }
    }
}
```
这个TargetSet启动死循环调用updateProviders，而这个updateProviders
```
for name, prov := range providers {
        wg.Add(1)

        updates := make(chan []*config.TargetGroup)
        go prov.Run(ctx, updates)

        go func(name string, prov TargetProvider) {
            select {
            case <-ctx.Done():
            case initial, ok := <-updates:
                // Handle the case that a target provider exits and closes the channel
                // before the context is done.
                if !ok {
                    break
                }
                // First set of all targets the provider knows.
                for _, tgroup := range initial {
                    ts.setTargetGroup(name, tgroup)
                }
            case <-time.After(5 * time.Second):
                // Initial set didn't arrive. Act as if it was empty
                // and wait for updates later on.
            }
            wg.Done()

            // Start listening for further updates.
            for {
                select {
                case <-ctx.Done():
                    return
                case tgs, ok := <-updates:
                    // Handle the case that a target provider exits and closes the channel
                    // before the context is done.
                    if !ok {
                        return
                    }
                    for _, tg := range tgs {
                        ts.update(name, tg)
                    }
                }
            }
        }(name, prov)
    }
```
变量所有的providers，这里的每个provider都是一个资源发现的提供者，通过provider的Run方法去加载更新，这个Run就是discovery的Run。回到了上面说的方法了，这样target就能够更新了。

上面演示了kubernetes的资源发现，下面还补充一个marathon的，其实原理是一样的discovery/marathon/marathon.go，
```
func (d *Discovery) updateServices(ctx context.Context, ch chan<- []*config.TargetGroup) (err error) {
    t0 := time.Now()
    defer func() {
        refreshDuration.Observe(time.Since(t0).Seconds())
        if err != nil {
            refreshFailuresCount.Inc()
        }
    }()

    targetMap, err := d.fetchTargetGroups()
    if err != nil {
        return err
    }

    all := make([]*config.TargetGroup, 0, len(targetMap))
    for _, tg := range targetMap {
        all = append(all, tg)
    }

    select {
    case <-ctx.Done():
        return ctx.Err()
    case ch <- all:
    }

    // Remove services which did disappear.
    for source := range d.lastRefresh {
        _, ok := targetMap[source]
        if !ok {
            select {
            case <-ctx.Done():
                return ctx.Err()
            case ch <- []*config.TargetGroup{{Source: source}}:
                log.Debugf("Removing group for %s", source)
            }
        }
    }

    d.lastRefresh = targetMap
    return nil
}
```
这个里面定时执行，通过fetchTargetGroups获取TargetGroups，
```
func (d *Discovery) fetchTargetGroups() (map[string]*config.TargetGroup, error) {
    url := RandomAppsURL(d.servers)
    apps, err := d.appsClient(d.client, url, d.token)
    if err != nil {
        return nil, err
    }

    groups := AppsToTargetGroups(apps)
    return groups, nil
}
```
这个方法就是调用marathon的/v2/apps接口去获取APP、task（container）信息。这个就是discovery的基本工作原理。















