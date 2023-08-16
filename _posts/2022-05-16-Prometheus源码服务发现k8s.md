---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus源码服务发现k8s
prometheus与时俱进在现在各种容器管理平台流行的当下，能够对各种容器管理平台进行数据采集和处理，并且能够自动发现监控对象，这个就是今天要说的discovery。他是一个资源发现的组件，能够根据资源变化调整监控对象。目前已经支持的discovery主要有DNS、kubernetes、marathon、zk、consul、file等，我们先从这个kubernetes的服务发现开始讲解吧

## 服务发现k8s
先是创建
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















