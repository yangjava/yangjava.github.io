---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus源码服务发现
prometheus与时俱进在现在各种容器管理平台流行的当下，能够对各种容器管理平台进行数据采集和处理，并且能够自动发现监控对象，这个就是今天要说的discovery。

## 服务发现机制
Prometheus是基于Pull模式抓取监控数据，首先要能够发现需要监控的目标对象target，特别Prometheus最开始设计是一个面向云原生应用程序的，云原生、容器场景下按需的资源使用方式对于监控系统而言就意味着没有了一个固定的监控目标，

所有的监控对象(基础设施、应用、服务)都在动态的变化。而对于Prometheus而言其解决方案就是引入一个中间的代理人（服务注册中心），这个代理人掌握着当前所有监控目标的访问信息，

Prometheus只需要向这个代理人询问有哪些监控目标控即可， 这种模式被称为服务发现(service discovery)。

SD模块专门负责去发现需要监控的target信息，Prometheus去从SD模块订阅该信息，有target信息时会推送给Prometheus，然后Prometheus拿到target信息后通过pull http协议去拉取监控指标数据。

discovery是一个资源发现的组件，能够根据资源变化调整监控对象。目前已经支持的discovery主要有DNS、kubernetes、marathon、zk、consul、file等。discovery支持文件、http、consul等自动发现targets，targets会被发送到scrape模块进行拉取。

Prometheus支持的服务发现协议是非常丰富的，目前已支持多达二十多种服务发现协议：
```
<azure_sd_config>
<consul_sd_config>
<digitalocean_sd_config>
<docker_sd_config>
<dockerswarm_sd_config>
<dns_sd_config>
<ec2_sd_config>
<openstack_sd_config>
<file_sd_config>
<gce_sd_config>
<hetzner_sd_config>
<http_sd_config>
<kubernetes_sd_config>
<kuma_sd_config>
<lightsail_sd_config>
<linode_sd_config>
<marathon_sd_config>
<nerve_sd_config>
<serverset_sd_config>
<triton_sd_config>
<eureka_sd_config>
<scaleway_sd_config>
<static_config>
```

## 服务发现原理
Prometheus核心功能包括服务发现、数据采集和数据存储。服务发现模块专门负责发现需要监控的目标采集点(target)信息，数据采集模块从服务发现模块订阅该信息，获取到target信息后，其中就包含协议(scheme)、主机地址:端口(instance)、请求路径(metrics_path)、请求参数(params)等；

然后数据采集模块就可以基于这些信息构建出一个完整的Http Request请求，定时通过pull http协议不断的去目标采集点(target)拉取监控样本数据(sample)；最后，将采集到监控样本数据交由TSDB模块进行数据存储。

Prometheus服务发现实现原理大致如下:
- 配置处理模块解析的prometheus.yml配置中scrape_configs部分，将配置的job生成一个个Discoverer服务，不同的服务发现协议都会有各自的Discoverer实现方式，它们根据实现逻辑去发现target，并将其放入到targets容器中；

- discoveryManager组件内部有个定时周期触发任务，每5秒检查targets容器，如果有变更则将targets容器中target信息放入到syncCh通道中；

- scrape组件会监听syncCh通道，这样需要监控的targets信息就传递给scrape组件，然后reload将target纳入监控开始抓取监控指标。

配置处理部分会根据scrape_configs部分配置的不同协议类型生成不同Discoverer，然后根据它们内部不同的实现逻辑去发现target，discoveryManager组件则相当于一个搬运工，scrape组件则是一个使用者，这两个组件都无感知服务发现协议的差异。

## 整体框架
discovery组件通过Manager对象管理所有的逻辑，当有数据变化时，通过syncChannel将数据发送给scrape组件。

discovery组件会为每个Job_name创建一个provider对象，它包含Discover对象：
- Discover对象会自动发现target；
- 当有targets变化时：
  - 首先，通过updateGroup()更新Manager中的targets对象；
  - 然后，向Manager的triggerSend channel发送消息，告诉Manager要更新；
  - 最后，Manager收到triggerSend channel中的消息，将Manager中的所有targets发送给syncChannel；

scrape组件接收syncChannel中的数据，然后使用reload()进行抓取对象更新：
- 若有新job，则创建scrapePool并启动它；
- 若有新target，则创建scrapeLoop并启动它；
- 若有消失的target，则停止其scrapeLoop；

下面分别来分析下配置处理、discoveryManager组件和scrape组件在服务发现方面的具体实现流程。

## 配置处理
Prometheus启动流程，有个配置加载组件通过reloadConfig加载解析prometheus配置文件后，

在reloader中循环调用各个组件的`ApplyConfig(cfg map[string]Configs)`方法处理配置，这其中就包括discovery/manager.go:

reloader中定义如下：
```
{
 name: "scrape_sd",
 //从配置文件中提取Section:scrape_configs
 reloader: func(cfg *config.Config) error {
  c := make(map[string]discovery.Configs)
  for _, v := range cfg.ScrapeConfigs {
   c[v.JobName] = v.ServiceDiscoveryConfigs
  }
  return discoveryManagerScrape.ApplyConfig(c)
 },
}
```
那下面就从discovery/manager.go中定义的ApplyConfig()方法分析。

根据配置注册provider：
```
for name, scfg := range cfg {
    //根据配置注册provider
 failedCount += m.registerProviders(scfg, name)
 discoveredTargets.WithLabelValues(m.name, name).Set(0)
}
```
其中关键的是m.registerProviders(scfg, name)，继续跟踪：
```
d, err := cfg.NewDiscoverer(DiscovererOptions{
 Logger: log.With(m.logger, "discovery", typ),
})
```
然后将所有注册到m.providers数组中的provider进行启动：
```
for _, prov := range m.providers {
 // 启动服务发现实例
 m.startProvider(m.ctx, prov)
}
```
跟踪到m.startProvider(m.ctx, prov)方法中：
```
updates := make(chan []*targetgroup.Group)
// 执行run  每个服务发现都有自己的run方法。
go p.d.Run(ctx, updates)
// 更新发现的服务
go m.updater(ctx, p, updates)
```
发现这里主要是启动两个协程，它们之间使用updates通道类型变量进行通信。

总结来说：
- 每个Config都会对应创建一个Discoverer实例，并被封装到provider存储在m.providers数组中；
- 然后遍历providers数组进行启动操作，启动操作启动了两个协程：
  - Discoverer.Run协程逻辑中主要根据发现协议发现targets；
  - 然后通过通道传递给discovery/Manager.updater协程中，将其存放到m.targets集合map中；

配置处理这里还有个比较关键的：Discoverer会根据不同协议实现发现target，它是如何实现的呢？

首先，我们来看下Discoverer实例创建：d, err := cfg.NewDiscoverer()，它是一个接口定义：
```
type Config interface {
 Name() string
 NewDiscoverer(DiscovererOptions) (Discoverer, error)
}
```
每种服务发现协议都在自己的SDConfig中实现了各自的NewDiscoverver()方法，这样就可以将服务发现逻辑封装到Discovererver实现中：

## discoveryManager组件
启动discoveryManagerScrape组件通过通道将targets数据信息传递给scrapeManager组件

discoveryManagerScrape组件启动入口：
```
g.Add(
 func() error {
  err := discoveryManagerScrape.Run()
  level.Info(logger).Log("msg", "Scrape discovery manager stopped")
  return err
 },
 func(err error) {
  level.Info(logger).Log("msg", "Stopping scrape discovery manager...")
  cancelScrape()
 },
)
```
一直跟踪会进入到sender()方法中，配置处理模块说过，有个协程会将Discoverer组件发现的targets信息存储到m.targets集合map中，然后给m.triggerSend发送信号，sender方法中就是启动定时周期触发器监听m.triggerSend信号：
```
func (m *Manager) sender() {
    // 周期性定时器定时触发任务，这里是5s触发一次
 ticker := time.NewTicker(m.updatert)
 defer ticker.Stop()

 for {
  select {
  case <-m.ctx.Done():
   return
  case <-ticker.C: // Some discoverers send updates too often so we throttle these with the ticker.
   select {
   case <-m.triggerSend:
    sentUpdates.WithLabelValues(m.name).Inc()
    select {
    case m.syncCh <- m.allGroups():
    default:
     delayedUpdates.WithLabelValues(m.name).Inc()
     level.Debug(m.logger).Log("msg", "Discovery receiver's channel was full so will retry the next cycle")
     select {
     case m.triggerSend <- struct{}{}:
     default:
     }
    }
   default:
   }
  }
 }
}
```
监听到m.triggerSend信号，则执行m.syncCh <- m.allGroups()，我们来看下m.allGroups()干了什么？
```
func (m *Manager) allGroups() map[string][]*targetgroup.Group {
 m.mtx.RLock()
 defer m.mtx.RUnlock()

 tSets := map[string][]*targetgroup.Group{}
 for pkey, tsets := range m.targets {
  var n int
  for _, tg := range tsets {
   // Even if the target group 'tg' is empty we still need to send it to the 'Scrape manager'
   // to signal that it needs to stop all scrape loops for this target set.
   tSets[pkey.setName] = append(tSets[pkey.setName], tg)
   n += len(tg.Targets)
  }
  discoveredTargets.WithLabelValues(m.name, pkey.setName).Set(float64(n))
 }
 return tSets
}
```
其实就是将m.targets数据发送到m.syncCh通道上，所以，discoveryManager组件比较简单，就是一个搬运工。

## 服务发现 (serviceDiscover) 简介
Prometheus采用pull方式拉取监控数据，需要实时感知被监控服务(Target)的变化．服务发现(serviceDiscover)支持多种服务发现系统，这些系统可以动态感知被监控的服务(Target)的变化，把变化的被监控服务(Target)转换为targetgroup.Group的结构，通过管道up发送个服务发现(serviceDiscover)．

以版本 v2.7.1为例，目前服务发现(serviceDiscover)支持的服务发现系统类型如下：
```

// ServiceDiscoveryConfig configures lists of different service discovery mechanisms.
type ServiceDiscoveryConfig struct {
	// List of labeled target groups for this job.
	StaticConfigs []*targetgroup.Group `yaml:"static_configs,omitempty"`
	// List of DNS service discovery configurations.
	DNSSDConfigs []*dns.SDConfig `yaml:"dns_sd_configs,omitempty"`
	// List of file service discovery configurations.
	FileSDConfigs []*file.SDConfig `yaml:"file_sd_configs,omitempty"`
	// List of Consul service discovery configurations.
	ConsulSDConfigs []*consul.SDConfig `yaml:"consul_sd_configs,omitempty"`
	// List of Serverset service discovery configurations.
	ServersetSDConfigs []*zookeeper.ServersetSDConfig `yaml:"serverset_sd_configs,omitempty"`
	// NerveSDConfigs is a list of Nerve service discovery configurations.
	NerveSDConfigs []*zookeeper.NerveSDConfig `yaml:"nerve_sd_configs,omitempty"`
	// MarathonSDConfigs is a list of Marathon service discovery configurations.
	MarathonSDConfigs []*marathon.SDConfig `yaml:"marathon_sd_configs,omitempty"`
	// List of Kubernetes service discovery configurations.
	KubernetesSDConfigs []*kubernetes.SDConfig `yaml:"kubernetes_sd_configs,omitempty"`
	// List of GCE service discovery configurations.
	GCESDConfigs []*gce.SDConfig `yaml:"gce_sd_configs,omitempty"`
	// List of EC2 service discovery configurations.
	EC2SDConfigs []*ec2.SDConfig `yaml:"ec2_sd_configs,omitempty"`
	// List of OpenStack service discovery configurations.
	OpenstackSDConfigs []*openstack.SDConfig `yaml:"openstack_sd_configs,omitempty"`
	// List of Azure service discovery configurations.
	AzureSDConfigs []*azure.SDConfig `yaml:"azure_sd_configs,omitempty"`
	// List of Triton service discovery configurations.
	TritonSDConfigs []*triton.SDConfig `yaml:"triton_sd_configs,omitempty"`
}
```

## 服务发现 (serviceDiscover) 管理多种服务发现系统
服务发现(serviceDiscover)为了实现对以上服务发现系统的统一管理，提供了一个Discoverer接口，由各个服务发现系统来实现，然后把上线的服务(Target)通过up管道发送给服务发现(serviceDiscover)

prometheus/discovery/manager.go
```
type Discoverer interface {
	// Run hands a channel to the discovery provider (Consul, DNS etc) through which it can send
	// updated target groups.
	// Must returns if the context gets canceled. It should not close the update
	// channel on returning.
	Run(ctx context.Context, up chan<- []*targetgroup.Group)
}
```

prometheus/discovery/targetgroup/targetgroup.go
```
// Group is a set of targets with a common label set(production , test, staging etc.).
type Group struct {
	// Targets is a list of targets identified by a label set. Each target is
	// uniquely identifiable in the group by its address label.
	Targets []model.LabelSet //服务(Target)主要标签，比如ip + port，示例："__address__": "localhost:9100"
	// Labels is a set of labels that is common across all targets in the group.
	Labels model.LabelSet　//服务(Target)其他标签，可以为空：
 
	// Source is an identifier that describes a group of targets.
	Source string //全局唯一个ID，示例：Source: "0"
}
```

Group的一个示例：
```
(dlv) p tg
*github.com/prometheus/prometheus/discovery/targetgroup.Group {
	Targets: []github.com/prometheus/common/model.LabelSet len: 1, cap: 1, [
	　　　　　　　[
		　　　　　　　"__address__": "localhost:9100", 
	　　　　　　　],
　　　　　　　　]
	],
	Labels: github.com/prometheus/common/model.LabelSet nil,
	Source: "0",}
```
除了静态服务发现系统(StaticConfigs)在prometheus/discovery/manager.go中实现了以上接口，其他动态服务发现系统，在prometheus/discovery/下都在有各自的目录下实现．

## 服务发现 (serviceDiscover)的配置初始化及应用
配置文件初始化

prometheus/cmd/prometheus/main.go
```
//discovery.Name("scrape")用于区分notify
discoveryManagerScrape  = discovery.NewManager(ctxScrape, log.With(logger, "component", "discovery manager scrape"), discovery.Name("scrape"))
```

调用NewManager方法，实例化Manager结构体：

prometheus/discovery/manager.go
```

// NewManager is the Discovery Manager constructor.
func NewManager(ctx context.Context, logger log.Logger, options ...func(*Manager)) *Manager {
	if logger == nil {
		logger = log.NewNopLogger()
	}
	mgr := &Manager{
		logger:         logger,
		syncCh:         make(chan map[string][]*targetgroup.Group),
		targets:        make(map[poolKey]map[string]*targetgroup.Group),
		discoverCancel: []context.CancelFunc{},
		ctx:            ctx,
		updatert:       5 * time.Second,
		triggerSend:    make(chan struct{}, 1),
	}
	for _, option := range options {
		option(mgr)
	}
	return mgr
}
```

结构体Manager定义如下：

prometheus/discovery/manager.go
```

// Manager maintains a set of discovery providers and sends each update to a map channel.
// Targets are grouped by the target set name.
type Manager struct {
	logger         log.Logger //日志
	name           string　　　// 用于区分srape和notify，因为他们用的同一个discovery/manager.go
	mtx            sync.RWMutex //同步读写锁
	ctx            context.Context //协同控制，比如系统退出
	discoverCancel []context.CancelFunc // 处理服务下线
 
	// Some Discoverers(eg. k8s) send only the updates for a given target group
	// so we use map[tg.Source]*targetgroup.Group to know which group to update.
	targets map[poolKey]map[string]*targetgroup.Group //发现的服务(Targets)
	// providers keeps track of SD providers.
	providers []*provider // providers的类型可分为kubernetes，DNS等
	// The sync channel sends the updates as a map where the key is the job value from the scrape config.
	syncCh chan map[string][]*targetgroup.Group //把发现的服务Targets)通过管道形式通知给scrapeManager
 
	// How long to wait before sending updates to the channel. The variable
	// should only be modified in unit tests.
	updatert time.Duration
 
	// The triggerSend channel signals to the manager that new updates have been received from providers.
	triggerSend chan struct{}
}
```

通过匿名函数加载prometheus.yml的scrape_configs下对应配置
```
prometheus/cmd/prometheus/main.go
 
func(cfg *config.Config) error {
    c := make(map[string]sd_config.ServiceDiscoveryConfig)
    for _, v := range cfg.ScrapeConfigs {
        c[v.JobName] = v.ServiceDiscoveryConfig
    }
    return discoveryManagerScrape.ApplyConfig(c)
},
```
以示例配置文件prometheus.yml为例，包含两个jobs，job_name分别是prometheus和node，每个job可以包含多个targets．以job_name：node为例，匿名函数变量v输出如下：

备注：以下所有的示例输出，都是基于以上示例配置文件
```

(dlv) p v
*github.com/prometheus/prometheus/config.ScrapeConfig {
	JobName: "node",
	HonorLabels: false,
	Params: net/url.Values nil,
	ScrapeInterval: 10000000000,
	ScrapeTimeout: 10000000000,
	MetricsPath: "/metrics",
	Scheme: "http",
	SampleLimit: 0,
	ServiceDiscoveryConfig: github.com/prometheus/prometheus/discovery/config.ServiceDiscoveryConfig {
		StaticConfigs: []*github.com/prometheus/prometheus/discovery/targetgroup.Group len: 1, cap: 1, [
			*(*"github.com/prometheus/prometheus/discovery/targetgroup.Group")(0xc0018a27b0),
		],
		DNSSDConfigs: []*github.com/prometheus/prometheus/discovery/dns.SDConfig len: 0, cap: 0, nil,
		FileSDConfigs: []*github.com/prometheus/prometheus/discovery/file.SDConfig len: 0, cap: 0, nil,
        ．．．．．．
        ．．．．．．
        ．．．．．．
		AzureSDConfigs: []*github.com/prometheus/prometheus/discovery/azure.SDConfig len: 0, cap: 0, nil,
		TritonSDConfigs: []*github.com/prometheus/prometheus/discovery/triton.SDConfig len: 0, cap: 0, nil,},
	HTTPClientConfig: github.com/prometheus/common/config.HTTPClientConfig {
		BasicAuth: *github.com/prometheus/common/config.BasicAuth nil,
		BearerToken: "",
		BearerTokenFile: "",
		ProxyURL: (*"github.com/prometheus/common/config.URL")(0xc000458cf8),
		TLSConfig: (*"github.com/prometheus/common/config.TLSConfig")(0xc000458d00),},
	RelabelConfigs: []*github.com/prometheus/prometheus/pkg/relabel.Config len: 0, cap: 0, nil,
	MetricRelabelConfigs: []*github.com/prometheus/prometheus/pkg/relabel.Config len: 0, cap: 0, nil,}
```
由以上结果可知，job_name：node对应静态服务发现系统(StaticConfigs)．其实，在配置文件prometheus.yml中的两个job_names都对应静态服务发现系统(StaticConfigs)．

ApplyConfig方法实现逻辑比较清晰：先实现每个job的Discoverer接口，然后启动该job对应的服务发现系统．
```
prometheus/discovery/manager.go
 
// ApplyConfig removes all running discovery providers and starts new ones using the provided config.
func (m *Manager) ApplyConfig(cfg map[string]sd_config.ServiceDiscoveryConfig) error {
	m.mtx.Lock()
	defer m.mtx.Unlock()
 
	for pk := range m.targets {
		if _, ok := cfg[pk.setName]; !ok {
			discoveredTargets.DeleteLabelValues(m.name, pk.setName)
		}
	}
	m.cancelDiscoverers()
    // name对应job_name，scfg是给出该job_name对应的服务发现系统类型，每个job_name下可以包含多种服务发现系统类型，但用的比较少
	for name, scfg := range cfg {
		m.registerProviders(scfg, name)
		discoveredTargets.WithLabelValues(m.name, name).Set(0)
	}
	for _, prov := range m.providers {
　　　　//启动每个job下对应的服务发现系统
		m.startProvider(m.ctx, prov)
	}
 
	return nil
}
```
ApplyConfig方法主要通过调用方法：registerProviders()和startProvider()实现以上功能， 接下来会详细分析这两个方法其实

- registerProviders
方法先判断每个job_name下包含所有的服务发现系统类型，接着由其对应的服务发现系统实现Discoverer接口，并构建provider和TargetGroups ，代码如下：
```

prometheus/discovery/manager.go
 
func (m *Manager) registerProviders(cfg sd_config.ServiceDiscoveryConfig, setName string) {
	var added bool
	add := func(cfg interface{}, newDiscoverer func() (Discoverer, error)) {
		t := reflect.TypeOf(cfg).String()
		for _, p := range m.providers {
			if reflect.DeepEqual(cfg, p.config) {
				p.subs = append(p.subs, setName)
				added = true
				return
			}
		}
 
		d, err := newDiscoverer()
		if err != nil {
			level.Error(m.logger).Log("msg", "Cannot create service discovery", "err", err, "type", t)
			failedConfigs.WithLabelValues(m.name).Inc()
			return
		}
 
		provider := provider{
			name:   fmt.Sprintf("%s/%d", t, len(m.providers)),
			d:      d,
			config: cfg,SyncCh
			subs:   []string{setName},
		}
		m.providers = append(m.providers, &provider)
		added = true
	}
　　//若是DNS服务发现，构造DNS的Discoverer
	for _, c := range cfg.DNSSDConfigs {
		add(c, func() (Discoverer, error) {
			return dns.NewDiscovery(*c, log.With(m.logger, "discovery", "dns")), nil
		})
	}
 ．．．．．．
 ．．．．．．
　　　
　　//若是静态服务发现，基于示例配置文件，构造静态服务发现的Discoverer
	if len(cfg.StaticConfigs) > 0 {
		add(setName, func() (Discoverer, error) {
			return &StaticProvider{TargetGroups: cfg.StaticConfigs}, nil
		})
	}
	if !added {
		// Add an empty target group to force the refresh of the corresponding
		// scrape pool and to notify the receiver that this target set has no
		// current targets.
		// It can happen because the combined set of SD configurations is empty
		// or because we fail to instantiate all the SD configurations.
		add(setName, func() (Discoverer, error) {
			return &StaticProvider{TargetGroups: []*targetgroup.Group{{}}}, nil
		})
	}
}
```

其中，cfg.StaticConfigs对应TargetGroups， 以job_name：node为例，TargetGroups对应输出如下：
```

(dlv) p setName
"node"
(dlv) p cfg.StaticConfigs
[]*github.com/prometheus/prometheus/discovery/targetgroup.Group len: 1, cap: 1, [
	*{
		Targets: []github.com/prometheus/common/model.LabelSet len: 1, cap: 1, [
	　　　　　　　　　　[
		　　　　　　　　　　"__address__": "localhost:9100", 
	　　　　　　　　　　],
		],
		Labels: github.com/prometheus/common/model.LabelSet nil,
		Source: "0",},
]
```

每个job_name对应一个TargetGroups，而每个TargetGroups可以包含多个provider，每个provider包含实现对应的Discoverer接口和job_name等．所以，对应关系：job_name －> TargetGroups －> 多个targets －> 多个provider －> 多个Discover．部分示例如下：
```

(dlv) p m.providers
[]*github.com/prometheus/prometheus/discovery.provider len: 2, cap: 2, [
	*{
		name: "string/0",
		d: github.com/prometheus/prometheus/discovery.Discoverer(*github.com/prometheus/prometheus/discovery.StaticProvider) ...,
		subs: []string len: 1, cap: 1, [
			"prometheus",
		],
		config: interface {}(string) *(*interface {})(0xc000536268),},
	*{
		name: "string/1",
		d: github.com/prometheus/prometheus/discovery.Discoverer(*github.com/prometheus/prometheus/discovery.StaticProvider) ...,
		subs: []string len: 1, cap: 1, ["node"],
		config: interface {}(string) *(*interface {})(0xc000518b78),},
]
(dlv) p m.providers[0].d
github.com/prometheus/prometheus/discovery.Discoverer(*github.com/prometheus/prometheus/discovery.StaticProvider) *{
	TargetGroups: []*github.com/prometheus/prometheus/discovery/targetgroup.Group len: 1, cap: 1, [
		*(*"github.com/prometheus/prometheus/discovery/targetgroup.Group")(0xc000ce09f0),
	],}
```

startProvider方法逐一启动job_name对应的所有服务发现系统
```

prometheus/discovery/manager.go
 
func (m *Manager) startProvider(ctx context.Context, p *provider) {
	level.Debug(m.logger).Log("msg", "Starting provider", "provider", p.name, "subs", fmt.Sprintf("%v", p.subs))
	ctx, cancel := context.WithCancel(ctx)
	updates := make(chan []*targetgroup.Group)
 
	m.discoverCancel = append(m.discoverCancel, cancel)
    
    // 第一个协程启动具体的发现的服务，作为[]*targetgroup.Group的生产者
	go p.d.Run(ctx, updates)
    // 第二个协程是[]*targetgroup.Group的消费者
	go m.updater(ctx, p, updates)
}
 
备注：Run方法调用位置是实现Discoverer的服务发现系统中．若是静态服务发现，Run方法在prometheus/discovery/manager.go中实现，若是动态服务发现系统，则在对应系统的目录下实现．
```

Run方法从结构体StaticProvider中取值，传递给[]*targetgroup.Group，作为服务发现的生产者
```

prometheus/discovery/manager.go
 
// StaticProvider holds a list of target groups that never change.
type StaticProvider struct {
	TargetGroups []*targetgroup.Group
}
 
// Run implements the Worker interface.
func (sd *StaticProvider) Run(ctx context.Context, ch chan<- []*targetgroup.Group) {
	// We still have to consider that the consumer exits right away in which case
	// the context will be canceled.
	select {
	case ch <- sd.TargetGroups:
	case <-ctx.Done():
	}
	close(ch)
}
```
updater方法从[]*targetgroup.Group获取TargetGroups，并把它发传给结构体Manager中对应的Targets，Manager中对应的Targets是map类型．
```
prometheus/discovery/manager.go
 
func (m *Manager) updater(ctx context.Context, p *provider, updates chan []*targetgroup.Group) {
	for {
		select {
		case <-ctx.Done(): //退出
			return
		case tgs, ok := <-updates: // 从[]*targetgroup.Group取TargetGroups
			receivedUpdates.WithLabelValues(m.name).Inc()
			if !ok {
				level.Debug(m.logger).Log("msg", "discoverer channel closed", "provider", p.name)
				return
			}
            // subs对应job_names，p.name对应系统名/索引值，比如：string/0
			for _, s := range p.subs {
				m.updateGroup(poolKey{setName: s, provider: p.name}, tgs)
			}
 
			select {
			case m.triggerSend <- struct{}{}:
			default:
			}
		}
	}
}
```
更新结构体Manager对应的targets，key是结构体poolKey，value是传递过来的TargetGroups，其中包含targets．
```
prometheus/discovery/manager.go
 
func (m *Manager) updateGroup(poolKey poolKey, tgs []*targetgroup.Group) {
	m.mtx.Lock()
	defer m.mtx.Unlock()
 
	for _, tg := range tgs {
		if tg != nil { // Some Discoverers send nil target group so need to check for it to avoid panics.
			if _, ok := m.targets[poolKey]; !ok {
				m.targets[poolKey] = make(map[string]*targetgroup.Group)
			}
			m.targets[poolKey][tg.Source] = tg //一个tg对应一个job，在map类型targets中，结构体poolkey和tg.Source可以确定一个tg，即job
		}
	}
}
```

```
prometheus/discovery/manager.go
 
// poolKey定义了每个发现的服务的来源
type poolKey struct {
	setName  string //对应系统名/索引值，比如：string/0(静态服务发现)，DNS/1(动态服务发现)
	provider string //对应job_name
}
```

服务发现 (serviceDiscover)起一个协程从服务发现系统获取数据

在main.go方法中启一个协程，运行Run()方法
```
prometheus/cmd/prometheus/main.go
 
{
    // Scrape discovery manager.
    g.Add(
        func() error {
            err := discoveryManagerScrape.Run()
            level.Info(logger).Log("msg", "Scrape discovery manager stopped")
            return err
        },
        func(err error) {
            level.Info(logger).Log("msg", "Stopping scrape discovery manager...")
            cancelScrape()
        },
    )
}
```

Run方法再起一个协程，运行sender()方法
```
// Run starts the background processing
func (m *Manager) Run() error {
	go m.sender()
	for range m.ctx.Done() {
		m.cancelDiscoverers()
		return m.ctx.Err()
	}
	return nil
}
```
sender方法的主要功能是处理结构体Manager中map类型的targets，然后传给结构体Manager中的map类型syncCh：syncCh chan map[string][]*targetgroup.Group．

```
func (m *Manager) sender() {
	ticker := time.NewTicker(m.updatert)
	defer ticker.Stop()
 
	for {
		select {
		case <-m.ctx.Done():
			return
		case <-ticker.C: // Some discoverers send updates too often so we throttle these with the ticker.
			select {
			case <-m.triggerSend:
				sentUpdates.WithLabelValues(m.name).Inc()
				select {
　　　　　　　　　//方法allGroups负责类型转换，并传给syncCh
				case m.syncCh <- m.allGroups(): 
				default:
					delayedUpdates.WithLabelValues(m.name).Inc()
					level.Debug(m.logger).Log("msg", "discovery receiver's channel was full so will retry the next cycle")
					select {
					case m.triggerSend <- struct{}{}:
					default:
					}
				}
			default:
			}
		}
	}
}
```
负责转换的allGroups()方法
```
func (m *Manager) allGroups() map[string][]*targetgroup.Group {
	m.mtx.Lock()
	defer m.mtx.Unlock()
 
	tSets := map[string][]*targetgroup.Group{}
	for pkey, tsets := range m.targets {
		var n int
		for _, tg := range tsets {
			// Even if the target group 'tg' is empty we still need to send it to the 'Scrape manager'
			// to signal that it needs to stop all scrape loops for this target set.
			tSets[pkey.setName] = append(tSets[pkey.setName], tg)
			n += len(tg.Targets)
		}
		discoveredTargets.WithLabelValues(m.name, pkey.setName).Set(float64(n))
	}
	return tSets
}
```

服务发现 (serviceDiscover)和指标采集 (scrapeManager)通信

scrapeManager通过一个协程启动，监听的正是结构体Manager中的syncCh，实现两者的通信
```

prometheus/cmd/prometheus/main.go
 
	{
		// Scrape manager.
		g.Add(
			func() error {
				// When the scrape manager receives a new targets list
				// it needs to read a valid config for each job.
				// It depends on the config being in sync with the discovery manager so
				// we wait until the config is fully loaded.
				<-reloadReady.C
 
				err := scrapeManager.Run(discoveryManagerScrape.SyncCh())
				level.Info(logger).Log("msg", "Scrape manager stopped")
				return err
			},
			func(err error) {
				// Scrape manager needs to be stopped before closing the local TSDB
				// so that it doesn't try to write samples to a closed storage.
				level.Info(logger).Log("msg", "Stopping scrape manager...")
				scrapeManager.Stop()
			},
		)
	}
```

```
prometheus/discovery/manager.go
 
// SyncCh returns a read only channel used by all the clients to receive target updates.
func (m *Manager) SyncCh() <-chan map[string][]*targetgroup.Group {
    //结构体Manager中的syncCh
	return m.syncCh
}
```

## scrape组件
scrapeManager组件启动：scrapeManager.Run(discoveryManagerScrape.SyncCh())，通道syncCh是被scrapeManager组件持有的，跟踪进入Run方法中：

```
func (m *Manager) Run(tsets <-chan map[string][]*targetgroup.Group) error {
 go m.reloader()
 for {
  select {
  //通过管道获取被监控的服务(targets)
  case ts := <-tsets:
   m.updateTsets(ts)

   select {
   // 关闭 Scrape Manager 处理信号
   //若从服务发现 (serviceDiscover)有服务(targets)变动，则给管道triggerReload传值，并触发reloader()方法更新服务
   case m.triggerReload <- struct{}{}:
   default:
   }

  case <-m.graceShut:
   return nil
  }
 }
}
```
通过case ts := <-tsets获取到syncCh通道上传递过来的targets数据，然后调用m.updateTsets(ts)将targets数据存储到scrapeManager.targetSets中，然后给m.triggerReload发送信号。

这个方法中go m.reloader()启动了一个协程，进入reloader()方法中：
```
func (m *Manager) reloader() {
 //定时器5s
 ticker := time.NewTicker(5 * time.Second)
 defer ticker.Stop()

 for {
  select {
  case <-m.graceShut:
   return
   // 若服务发现(serviceDiscovery)有服务(targets)变动，就会向管道triggerReload写入值，定时器每5s判断一次triggerReload管道是否有值，若有值，则触发reload方法
  case <-ticker.C:
   select {
   case <-m.triggerReload:
    m.reload()
   case <-m.graceShut:
    return
   }
  }
 }
}
```

也是通过定时周期触发任务监听m.triggerReload信号，执行m.reload()将targets加载进来。

至此，服务发现 (serviceDiscover)功能分析结束











