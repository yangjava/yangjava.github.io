---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus源码通知管理
Prometheus会在配置文件定义一些告警规则表达式,  当采集的metrics经过聚合, 满足告警表达式条件, 将触发告警, 发送给告警服务Alertmanager.

## 通知管理(notifierManager)
本文主要分析与Alertmanager交互的通知管理(notifierManager), 但会先梳理下规则管理(ruleManager)的部分内容.  因为告警规则的最终判断是由规则管理(ruleManager)完成.

### 规则管理(ruleManager)把告警信息发送给通知管理(notifierManager)
规则管理(ruleManager)调用方法NewManager完成实例化, 其中，参数是结构体ManagerOptions, 我们需要关注的是该结构体中类型为NotifyFunc的方法sendAlerts．
```
prometheus/cmd/prometheus/main.go
 
ruleManager = rules.NewManager(&rules.ManagerOptions{
    Appendable:      fanoutStorage,
    TSDB:            localStorage,
    QueryFunc:       rules.EngineQueryFunc(queryEngine, fanoutStorage),
    // 若触发告警规则,则通过sendAlerts发送告警信息
    NotifyFunc:      sendAlerts(notifierManager, cfg.web.ExternalURL.String()),
    Context:         ctxRule,
    ExternalURL:     cfg.web.ExternalURL,
    Registerer:      prometheus.DefaultRegisterer,
    Logger:          log.With(logger, "component", "rule manager"),
    OutageTolerance: time.Duration(cfg.outageTolerance),
    ForGracePeriod:  time.Duration(cfg.forGracePeriod),
    ResendDelay:     time.Duration(cfg.resendDelay),
})
 
prometheus/rules/manager.go
type ManagerOptions struct {
	ExternalURL     *url.URL
	QueryFunc       QueryFunc
	NotifyFunc      NotifyFunc
	Context         context.Context
	Appendable      Appendable
	TSDB            storage.Storage
	Logger          log.Logger
	Registerer      prometheus.Registerer
	OutageTolerance time.Duration
	ForGracePeriod  time.Duration
	ResendDelay     time.Duration
 
	Metrics *Metrics
}
```
sendAlerts方法主要作用是把规则管理(ruleManager)把告警信息转换成notifier.Alert类型.
```
prometheus/cmd/prometheus/main.go
 
//遍历告警信息,构造告警,告警信息大于0, 则发送告警
// sendAlerts implements the rules.NotifyFunc for a Notifier.
func sendAlerts(s sender, externalURL string) rules.NotifyFunc {
	return func(ctx context.Context, expr string, alerts ...*rules.Alert) {
		var res []*notifier.Alert
        // 遍历告警信息
		for _, alert := range alerts {
			a := &notifier.Alert{
				StartsAt:     alert.FiredAt,
				Labels:       alert.Labels,
				Annotations:  alert.Annotations,
				GeneratorURL: externalURL + strutil.TableLinkForExpression(expr),
			}
            // 若告警结束,设置告警结束时间为ResolverdAt时间
			if !alert.ResolvedAt.IsZero() {
				a.EndsAt = alert.ResolvedAt
			} else {
            // 若告警还是active转台,设置告警结束时间为当前时间
				a.EndsAt = alert.ValidUntil
			}
			res = append(res, a)
		}
 
        // 若是有告警信息, 则发送告警
		if len(alerts) > 0 {
			s.Send(res...)
		}
	}
}
```
接着由Send方法把notifier.Alert类型的告警信息添加到告警队列中n.queue，其中会涉及到一些队列大小限制，通过先进先出保存最新的告警信息．最后等待通知管理(notifierManager)从队列中取走．
```

prometheus/notifier/notifier.go
 
// Send queues the given notification requests for processing.
// Panics if called on a handler that is not running.
func (n *Manager) Send(alerts ...*Alert) {
    // 锁管理这里不多介绍
	n.mtx.Lock()
	defer n.mtx.Unlock()
 
	// Attach external labels before relabelling and sending.
    // 根据配置文件prometheus.yml的alert_relabel_configs下的relabel_config对告警的label进行重置
	for _, a := range alerts {
		lb := labels.NewBuilder(a.Labels)
 
		for ln, lv := range n.opts.ExternalLabels {
			if a.Labels.Get(string(ln)) == "" {
				lb.Set(string(ln), string(lv))
			}
		}
 
		a.Labels = lb.Labels()
	}
 
	alerts = n.relabelAlerts(alerts)
 
    // 若待告警信息的数量大于队列总容量,则移除待告警信息中最早的告警信息, 依据的规则是先进先移除
	// Queue capacity should be significantly larger than a single alert
	// batch could be.
	if d := len(alerts) - n.opts.QueueCapacity; d > 0 {
        // 和python的列表相似
		alerts = alerts[d:]
 
		level.Warn(n.logger).Log("msg", "Alert batch larger than queue capacity, dropping alerts", "num_dropped", d)
		n.metrics.dropped.Add(float64(d))
	}
    // 若队列中已有的告警信息和待发送的告警信息大于队列的总容量,则从队列中移除最早的告警信息, 依据是先进先移除
	// If the queue is full, remove the oldest alerts in favor
	// of newer ones.
	if d := (len(n.queue) + len(alerts)) - n.opts.QueueCapacity; d > 0 {
		n.queue = n.queue[d:]
 
		level.Warn(n.logger).Log("msg", "Alert notification queue full, dropping alerts", "num_dropped", d)
		n.metrics.dropped.Add(float64(d))
	}
    // 把待发送的告警信息加入队列
	n.queue = append(n.queue, alerts...)
 
    // 告知通知管理(notifierManager)有告警信息需要处理
	// Notify sending goroutine that there are alerts to be processed.
	n.setMore()
}
```
其中，setMore方法相当于一个触发器，向管道n.more发送触发信息, 告知通知管理(notifierManager)有告警信息需要处理，是连接规则管理(ruleManager)和通知管理(notifierManager)的桥梁. 
```

prometheus/notifier/notifier.go
 
// setMore signals that the alert queue has items.
func (n *Manager) setMore() {
	// If we cannot send on the channel, it means the signal already exists
	// and has not been consumed yet.
	select {
	case n.more <- struct{}{}:
	default:
	}
}
```
再分析通知管理(notifierManager)的Run方法可知，在for循环中，监听的管道正是n.more
```
prometheus/notifier/notifier.go
 
// Run dispatches notifications continuously.
func (n *Manager) Run(tsets <-chan map[string][]*targetgroup.Group) {
 
	for {
		select {
		case <-n.ctx.Done():
			return
		case ts := <-tsets:
			n.reload(ts)
        // 监听管道n.more
		case <-n.more:
		}
		alerts := n.nextBatch()
 
		if !n.sendAll(alerts...) {
			n.metrics.dropped.Add(float64(len(alerts)))
		}
		// If the queue still has items left, kick off the next iteration.
		if n.queueLen() > 0 {
			n.setMore()
		}
	}
}
```
以上即规则管理(ruleManager)和通知管理(notifierManager)之间的连接关系分析，接下来详细分析通知管理(notifierManager)

## 启动通知管理(notifierManager)
通知管理(notifierManager)实例化结构体Manager是在方法NewManagern中完成。
```

prometheus/notifier/notifier.go
 
// NewManager is the manager constructor.
func NewManager(o *Options, logger log.Logger) *Manager {
	ctx, cancel := context.WithCancel(context.Background())
 
    // 初始化http请求客户端
	if o.Do == nil {
		o.Do = do
	}
	if logger == nil {
		logger = log.NewNopLogger()
	}
 
    // 构建notifier的Manager结构体实例
	n := &Manager{
		queue:  make([]*Alert, 0, o.QueueCapacity), // 初始化告警队列
		ctx:    ctx, // 利用context协同处理
		cancel: cancel, // 级联式cancel ctx相关的协程
		more:   make(chan struct{}, 1), more是是否有告警信息的标记信号
		opts:   o,  // 其他一些参数,包含告警队列总容量, 告警label重置等
		logger: logger, // 日志相关
	}
 
    // 返回告警队列的长度
	queueLenFunc := func() float64 { return float64(n.queueLen()) }
     
    // 获取处于active的alertmanager地址个数, 用于notifier发送告警信息
	alertmanagersDiscoveredFunc := func() float64 { return float64(len(n.Alertmanagers())) }
 
    // 生成notifier相关的一些metric, 注册到prometheus
	n.metrics = newAlertMetrics(
		o.Registerer, // 注册的机制是: prometheus.Registerer
		o.QueueCapacity, //队列容量
		queueLenFunc, // 队列当前长度
		alertmanagersDiscoveredFunc, //处于active的alertmanager地址个数
	)
 
	return n
}
 
 
// Manager is responsible for dispatching alert notifications to an
// alert manager service.
type Manager struct {
	queue []*Alert
	opts  *Options
 
	metrics *alertMetrics
 
	more   chan struct{}
	mtx    sync.RWMutex
	ctx    context.Context
	cancel func()
 
	alertmanagers map[string]*alertmanagerSet
	logger        log.Logger
}
```
从配置文件中加载告警相关的配置信息，构建结构体alertmanagerSet
```
prometheus/notifier/notifier.go
 
// ApplyConfig updates the status state as the new config requires.
func (n *Manager) ApplyConfig(conf *config.Config) error {
    // 锁相关操作
	n.mtx.Lock()
	defer n.mtx.Unlock()
 
    // 配置文件prometheus.yml中global下的external_labels, 用于外部系统标签的，不是用于metrics数据
	n.opts.ExternalLabels = conf.GlobalConfig.ExternalLabels
    // 配置文件prometheus.yml中alertingl下alert_relabel_configs, 动态修改 alert 属性的规则配置
	n.opts.RelabelConfigs = conf.AlertingConfig.AlertRelabelConfigs
 
	amSets := make(map[string]*alertmanagerSet)
 
    // 遍历告警相关的配置,即配置文件prometheus.yml的alerting
	for _, cfg := range conf.AlertingConfig.AlertmanagerConfigs {
        把alerting下每个配置项, 转换成结构实例:alertmanagerSet
		ams, err := newAlertmanagerSet(cfg, n.logger)
		if err != nil {
			return err
		}
        //更新告警指标(metrics),即notifier注册到prometheus的指标(metrics), 参照方法newAlertMetrics
		ams.metrics = n.metrics
 
		// The config hash is used for the map lookup identifier.
        // 编码成json字符串
		b, err := json.Marshal(cfg)
		if err != nil {
			return err
		}
        // key为该配置项的md5值,可作为唯一识别码, value是该配置的实例化的alertmanagerSet
		amSets[fmt.Sprintf("%x", md5.Sum(b))] = ams
	}
 
	n.alertmanagers = amSets
 
	return nil
}
```
其中结构体alertmanagerSet定义如下，保存告警服务信息
```

prometheus/notifier/notifier.go
 
// alertmanagerSet contains a set of Alertmanagers discovered via a group of service
// discovery definitions that have a common configuration on how alerts should be sent.
type alertmanagerSet struct {
	cfg    *config.AlertmanagerConfig // prometheus.yml中告警相关的配置
	client *http.Client  // http客户端
 
	metrics *alertMetrics // 告警服务注册到prometheus的指标,参照方法: newAlertMetrics
 
	mtx        sync.RWMutex // 同步锁
	ams        []alertmanager // 处于active的alertmanager的地址
	droppedAms []alertmanager // 处于dropped状态的alertmanager, 下面给个示例
	logger     log.Logger    // 日志
}
```
active和dropped alertmanager的地址示例:
```

{
  "status": "success",
  "data": {
    "activeAlertmanagers": [
      {
        "url": "http://127.0.0.1:9094/api/v1/alerts"
      }
    ],
    "droppedAlertmanagers": [
      {
        "url": "http://127.0.0.1:9093/api/v1/alerts"
      }
    ]
  }
```
启动notifier服务的Run方法, 监听n.more管道. 若规则管理(ruleManager)向管道n.more发送信号, 告知通知管理(notifierManager)有告警信息需要发送，则触发接下来的告警信息处理
```
prometheus/notifier/notifier.go
 
// Run dispatches notifications continuously.
func (n *Manager) Run(tsets <-chan map[string][]*targetgroup.Group) {
 
	for {
		select {
        // 退出
		case <-n.ctx.Done():
			return
       // 发现告警服务有更新,重新加载配置. 参考服务发现(discoveryManager)
		case ts := <-tsets:
			n.reload(ts)
       // 告警信号
		case <-n.more:
		}
        // 利用告警队列, 获取告警信息
		alerts := n.nextBatch()
        // 根据active的alertmanager地址, 发送告警,并判断是否发送成功
		if !n.sendAll(alerts...) {
			n.metrics.dropped.Add(float64(len(alerts)))
		}
        // 若告警队列中还有告警信息,则再次想n.more传入信号,发送告警信息
		// If the queue still has items left, kick off the next iteration.
		if n.queueLen() > 0 {
			n.setMore()
		}
	}
}
```
此外, 以上的Run方法和的指标采集(scrapeManager)的Run方法共用服务发现 (serviceDiscover)的处理逻辑, 检测目标(targets)是否有变动, 不同的是通知管理(notifierManager)只监听告警服务的变动.

```

prometheus/cmd/prometheus/main.go
 
g.Add(
    func() error {
        // When the notifier manager receives a new targets list
        // it needs to read a valid config for each job.
        // It depends on the config being in sync with the discovery manager
        // so we wait until the config is fully loaded.
        <-reloadReady.C
        // 入口
        notifierManager.Run(discoveryManagerNotify.SyncCh())
        level.Info(logger).Log("msg", "Notifier manager stopped")
        return nil
    },
    func(err error) {
        notifierManager.Stop()
    },
)
```

指标采集(scrapeManager)的tset为map类型, key为job_name, value为targetgroup.Group．而通知管理(notifierManager)的tset也为map类型，不同的是key为AlertmanagerConfig, value为targetgroup.Group． 其中，AlertmanagerConfig对应prometheus.yml中的alertmanagers对应的配置, 可参考Prometheus.yml配置文件示例

若检测到发现告警服务有变动,则会调用reload方法，同步告警服务
```

prometheus/notifier/notifier.go
 
func (n *Manager) reload(tgs map[string][]*targetgroup.Group) {
    // 锁操作, 不做解释
	n.mtx.Lock()
	defer n.mtx.Unlock()
 
    // 遍历发现的告警服务
	for id, tgroup := range tgs {
        // 判断每个告警服务的md5sum值是否已经在prometheus注册过, 可参考本文前面分析的ApplyConfig方法
		am, ok := n.alertmanagers[id]
		if !ok {
			level.Error(n.logger).Log("msg", "couldn't sync alert manager set", "err", fmt.Sprintf("invalid id:%v", id))
			continue
		}
        // 把告警信息同步给该告警服务
		am.sync(tgroup)
	}
}
```

sync方法实现具体的同步操作，先遍历服务发现的告警服务，结合配置文件中alerting中relabel_configs信息，为每个激活状态的alertmanager构建实例，并根据得到的告警服务，再利用url进行去重处理．

其中sync方法中的alertmanagerFromGroup处理逻辑比较简单，对配置文件中的每个告警服务的label整理成如下示例：
```

(dlv) p lset
github.com/prometheus/prometheus/pkg/labels.Labels len: 3, cap: 3, [
	{
		Name: "__address__",
		Value: "alertmanager:9093",},
	{
		Name: "__alerts_path__",
		Value: "/api/v1/alerts",},
	{
		Name: "__scheme__",
		Value: "http",},
]
```
然后根据配置文件中alerting中relabel_configs的SourceLabels和Action，每个告警服务中进行匹配，若匹配成功，则标记该告警服务为Active，否则标记为Dropped．
```

prometheus/notifier/notifier.go
 
// sync extracts a deduplicated set of Alertmanager endpoints from a list
// of target groups definitions.
func (s *alertmanagerSet) sync(tgs []*targetgroup.Group) {
    // 处于激活状态的告警服务
	allAms := []alertmanager{}
    // 处于已丢弃状态的告警服务, 该部分是后加的保存项
	allDroppedAms := []alertmanager{}
 
    // 遍历服务发现的告警服务
	for _, tg := range tgs {
		ams, droppedAms, err := alertmanagerFromGroup(tg, s.cfg)
		if err != nil {
			level.Error(s.logger).Log("msg", "Creating discovered Alertmanagers failed", "err", err)
			continue
		}
		allAms = append(allAms, ams...)
		allDroppedAms = append(allDroppedAms, droppedAms...)
	}
 
	s.mtx.Lock()
	defer s.mtx.Unlock()
　　// 清空之前保存的告警服务，并通过每个告警的URL, 实现去重
	// Set new Alertmanagers and deduplicate them along their unique URL.
	s.ams = []alertmanager{}
	s.droppedAms = []alertmanager{}
	s.droppedAms = append(s.droppedAms, allDroppedAms...)
	seen := map[string]struct{}{}7
　　//进行去重处理
	for _, am := range allAms {
		us := am.url().String()
		if _, ok := seen[us]; ok {
			continue
		}
 
		// This will initialize the Counters for the AM to 0.
		s.metrics.sent.WithLabelValues(us)
		s.metrics.errors.WithLabelValues(us)
 
		seen[us] = struct{}{}
		s.ams = append(s.ams, am)
	}
}
```
获得了最新的告警服务列表，再回到之前的方法Run()，把待发送的告警信息信息，通过方法sendAll发送个最新的告警服务列表．若至少成功发送个一个告警服务，则返回true．

```

// sendAll sends the alerts to all configured Alertmanagers concurrently.
// It returns true if the alerts could be sent successfully to at least one Alertmanager.
func (n *Manager) sendAll(alerts ...*Alert) bool {
	begin := time.Now()
　　// 转换告警信息为json格式
	b, err := json.Marshal(alerts)
	if err != nil {
		level.Error(n.logger).Log("msg", "Encoding alerts failed", "err", err)
		return false
	}
 
	n.mtx.RLock()
　　// 获取当前最新的告警服务列表
	amSets := n.alertmanagers
	n.mtx.RUnlock()
 
	var (
		wg         sync.WaitGroup
		numSuccess uint64
	)
　　// 遍历告警服务列表
	for _, ams := range amSets {
		ams.mtx.RLock()
 
		for _, am := range ams.ams {
			wg.Add(1)
 
			ctx, cancel := context.WithTimeout(n.ctx, time.Duration(ams.cfg.Timeout))
			defer cancel()
 
　　　　　　　// 起一个协程发送告警信息
			go func(ams *alertmanagerSet, am alertmanager) {
				u := am.url().String()
 
				if err := n.sendOne(ctx, ams.client, u, b); err != nil {
					level.Error(n.logger).Log("alertmanager", u, "count", len(alerts), "msg", "Error sending alert", "err", err)
					n.metrics.errors.WithLabelValues(u).Inc()
				} else {
					atomic.AddUint64(&numSuccess, 1)
				}
				n.metrics.latency.WithLabelValues(u).Observe(time.Since(begin).Seconds())
				n.metrics.sent.WithLabelValues(u).Add(float64(len(alerts)))
 
				wg.Done()
			}(ams, am)
		}
		ams.mtx.RUnlock()
	}
	wg.Wait()
 
　　// 若至少成功发送给一个告警服务，则返回true
	return numSuccess > 0
}
```
至此，通知管理(notifierManager)分析完成













