---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus源码指标采集
discovery模块利用各种服务发现协议发现目标采集点，并通过channel管道将最新发现的目标采集点信息实时同步给scrape模块，scrape模块负责使用http协议从目标采集点上抓取监控指标数据。

## 数据抓取原理
discovery服务发现模块经过Discoverer组件 --> updater组件 --> sender组件，将服务发现采集点实时动态发送到syncCh通道上，而该通道的另一端就是scrape模块，这样discovery模块和scrape模块就构建起了关联。

scrape模块updateTsets组件通过协程方式运行实时监听syncCh通道，并将更新写入到scrapeManager结构体中targetSets字段对应的map中，同时触发triggerSend信号给reloader组件，告诉该组件采集点有更新，reloader组件就从scrapeManager中targetSets中拉取最新采集点进行加载。

reloader组件基于这些采集点信息生成一个个targetScraper组件，targetScraper组件组件主要负责按照job中配置的interval时间间隔不停轮训调用采集点的HTTP接口，这样就实现了采集点的指标数据采集。

## 指标采集(scrapeManager)简介
为了从服务发现(serviceDiscover)实时获取监控服务(targets)，指标采集(scrapeManager)通过协程把管道(chan)获取来的服务(targets)存进一个map类型：map[string][]*targetgroup.Group．其中，map的key是job_name，map的value是结构体targetgroup.Group，该结构体包含该job_name对应的Targets，Labels和Source．

指标采集(scrapeManager)获取服务(targets)的变动，可分为多种情况，以服务增加为例，若有新的job添加，指标采集(scrapeManager)会进行重载，为新的job创建一个scrapePool，并为job中的每个target创建一个scrapeLoop．若job没有变动，只增加了job下对应的targets，则只需创建新的targets对应的scrapeLoop．

## 指标采集(scrapeManager)实时获取监控服务
指标采集(scrapeManager)获取实时监控服务(targets)的入口函数：scrapeManager.Run(discoveryManagerScrape.SyncCh());

```
prometheus/cmd/prometheus/main.go
 
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
 
 
// ts即map[string][]*targetgroup.Group
(dlv) p ts["prometheus"]
[]*github.com/prometheus/prometheus/discovery/targetgroup.Group len: 1, cap: 1, [
	*{
		Targets: []github.com/prometheus/common/model.LabelSet len: 1, cap: 1, [
			[...],
		],
		Labels: github.com/prometheus/common/model.LabelSet nil,
		Source: "0",},
]
```
其中包含两个部分：scrapeManager的初始化和起一个协程监控服务(targets)的变化

- scrapeManager的初始化，调用NewManager方法实现：
```
prometheus/cmd/prometheus/main.go
 
//fanoutStorage是监控的存储的抽象
scrapeManager = scrape.NewManager(log.With(logger, "component", "scrape manager"), fanoutStorage)
```
NewManager方法了实例化结构体Manager：
```

prometheus/scrape/manager.go
 
// NewManager is the Manager constructor
func NewManager(logger log.Logger, app Appendable) *Manager {
	if logger == nil {
		logger = log.NewNopLogger()
	}
	return &Manager{
		append:        app,
		logger:        logger,
		scrapeConfigs: make(map[string]*config.ScrapeConfig),
		scrapePools:   make(map[string]*scrapePool),
		graceShut:     make(chan struct{}),
		triggerReload: make(chan struct{}, 1),
	}
}
```
scrapePool生成并初始化基础数据。

scrapeManager结构体中targetSets字段对应的map中存放了当前服务发现的最新采集点信息，key是job名称，遍历该targetSets中存放的采集点信息，为每个job对应生成一个scrapePool结构体的实例，即scrapePool是封装单个抓取job的工作单元：
```
ScrapePools 是单个的Job的抓取目标的工作单位：
type scrapePool struct {
 //存储指标
 appendable storage.Appendable
 //一个scrapePool对应一个job，config即为该job配置
 config *config.ScrapeConfig
 // 基于job配置生成http请求客户端工具，比如封装认证信息等
 client *http.Client
 //每个target都会生成一个loop
 loops          map[uint64]loop
 //target_limit检查
 targetLimitHit bool
 //relabe后有效的采集点
 activeTargets  map[uint64]*Target
 //relabel后无效采集点
 droppedTargets []*Target
 //生成scrapeLoop工厂函数
 newLoop func(scrapeLoopOptions) loop
}
```
每个抓取job生成的scrapePool存放在scrapeManager结构体中scrapePools这个map中：
```
scrapePools   map[string]*scrapePool
```

结构体Manager维护map类型的scrapePools和targetSets，两者key都是job_name，但scrapePools的value对应结构体scrapepool，而targetSets的value对应的结构体是Group，分别给出了两者的示例输出
```

prometheus/scrape/manager.go
 
// Manager maintains a set of scrape pools and manages start/stop cycles
// when receiving new target groups form the discovery manager.
type Manager struct {
	logger    log.Logger  //系统日志
	append    Appendable  //存储监控指标
	graceShut chan struct{}  //退出
 
	mtxScrape     sync.Mutex // Guards the fields below. 读写锁
	scrapeConfigs map[string]*config.ScrapeConfig  //prometheus.yml的srape_config配置部分，key对应job_name，value对应job_name的配置参数
	scrapePools   map[string]*scrapePool  //key对应job_name，value对应结构体scrapePool，包含该job_name下所有的targets
	targetSets    map[string][]*targetgroup.Group  //key对应job_name，value对应结构体Group，包含job_name对应的Targets，Labels和Source
 
	triggerReload chan struct{} //若有新的服务(targets)通过服务发现(serviceDisvoer)传过来，会向该管道传值，触发加载配置文件操作，后面会讲到
}
 
 
基于job_name：node的targetSets的示例输出：
(dlv) p m.targetSets["node"]
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
 
基于job_name：node的scrapePools示例输出：
(dlv) p m.scrapePools
map[string]*github.com/prometheus/prometheus/scrape.scrapePool [
	"node": *{
		appendable: github.com/prometheus/prometheus/scrape.Appendable(*github.com/prometheus/prometheus/storage.fanout) ...,
		logger: github.com/go-kit/kit/log.Logger(*github.com/go-kit/kit/log.context) ...,
		mtx: (*sync.RWMutex)(0xc001be0020),
		config: *(*"github.com/prometheus/prometheus/config.ScrapeConfig")(0xc00048ab40),
		client: *(*"net/http.Client")(0xc000d303c0),
		activeTargets: map[uint64]*github.com/prometheus/prometheus/scrape.Target [],
		droppedTargets: []*github.com/prometheus/prometheus/scrape.Target len: 0, cap: 0, nil,
		loops: map[uint64]github.com/prometheus/prometheus/scrape.loop [],
		cancel: context.WithCancel.func1,
		newLoop: github.com/prometheus/prometheus/scrape.newScrapePool.func2,}, 
]
```

在前面已经多次提到，指标采集(scrapeManager)在main.go启动时，会起一个协程运行Run方法，从服务发现(serviceDiscover)实时获取被监控服务(targets)，接下来看下Run方法的具体实现
```

prometheus/scrape/manager.go
 
// Run receives and saves target set updates and triggers the scraping loops reloading.
// Reloading happens in the background so that it doesn't block receiving targets updates.
func (m *Manager) Run(tsets <-chan map[string][]*targetgroup.Group) error {
     //定时(5s)更新服务(targets)，结合triggerReload一起使用，即每5s判断一次triggerReload是否更新．
	go m.reloader() 
	for {
		select {
        //通过管道获取被监控的服务(targets)
		case ts := <-tsets:  
			m.updateTsets(ts)
 
			select {
　　　　　　　//若从服务发现 (serviceDiscover)有服务(targets)变动，则给管道triggerReload传值，并触发reloader()方法更新服务．
			case m.triggerReload <- struct{}{}: 
			default:
			}
 
		case <-m.graceShut:
			return nil
		}
	}
}
```
以上流程还是比较清晰，若服务发现(serviceDiscovery)有服务(target)变动，Run方法就会向管道triggerReload注入值：m.triggerReload <- struct{}{}中，并起了一个协程，运行reloader方法．用于定时更新服务(targets)．启动这个协程应该是为了防止阻塞从服务发现(serviceDiscover)获取变动的服务(targets)

reloader方法启动了一个定时器，在无限循环中每5s判断一下管道triggerReload，若有值，则执行reload方法．
```
prometheus/scrape/manager.go
 
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
reload方法会根据job_name比较targetSets，scrapePools和scrapeConfigs的一致性，并把每个job_name下的类型为[]*targetgroup.Group的groups通过协程传给sp.Sync方法，增加并发．
```

prometheus/scrape/manager.go
 
func (m *Manager) reload() {
	m.mtxScrape.Lock()
	var wg sync.WaitGroup
　　//setName对应job_name，
　　//group的结构体包含job_name对应的Targets，Labels和source，这个在上篇文章已经详细介绍
	for setName, groups := range m.targetSets {
		var sp *scrapePool
		existing, ok := m.scrapePools[setName]
　　　　 //若该job_name不在scrapePools中，分为两种情况处理
        //(1)job_name不在scrapeConfigs中，则报错
　　　　 //(2)job_name在scrapeConfigs中，则需要把该job_name加到scrapePools中
		if !ok {
			scrapeConfig, ok := m.scrapeConfigs[setName]
			if !ok {
				level.Error(m.logger).Log("msg", "error reloading target set", "err", "invalid config id:"+setName)
				continue
			}
			sp = newScrapePool(scrapeConfig, m.append, log.With(m.logger, "scrape_pool", setName))
			m.scrapePools[setName] = sp
		} else {
			sp = existing
		}
 
		wg.Add(1)
		// Run the sync in parallel as these take a while and at high load can't catch up.
		go func(sp *scrapePool, groups []*targetgroup.Group) {
　　　　　　　//把groups转换为targets类型
			sp.Sync(groups)
			wg.Done()
		}(sp, groups)
	}
	m.mtxScrape.Unlock()
	wg.Wait()
}
```
sp.Sync方法引入了Target结构体，把[]*targetgroup.Group类型的groups转换为targets类型，其中每个groups对应一个job_name下多个targets．随后，调用sp.sync方法，同步scrape服务
```

prometheus/scrape/scrape.go
 
// Sync converts target groups into actual scrape targets and synchronizes
// the currently running scraper with the resulting set and returns all scraped and dropped targets.
func (sp *scrapePool) Sync(tgs []*targetgroup.Group) {
	start := time.Now()
 
	var all []*Target
	sp.mtx.Lock()
	sp.droppedTargets = []*Target{}
	for _, tg := range tgs {
　　　　//转换targetgroup.Group类型为Target
		targets, err := targetsFromGroup(tg, sp.config)
		if err != nil {
			level.Error(sp.logger).Log("msg", "creating targets failed", "err", err)
			continue
		}
        // 这里有个疑问，tg对应一个target，为什么返回回来的targets不是对应一个target相关参数，需要用for循环？
		for _, t := range targets {
            //判断Target的有效label是否大于0
			if t.Labels().Len() > 0 {
				all = append(all, t)
			} else if t.DiscoveredLabels().Len() > 0 {
                //若为无效Target，则加入scrapeLoop的droppedTargets中
				sp.droppedTargets = append(sp.droppedTargets, t)
			}
		}
	}
	sp.mtx.Unlock()
	sp.sync(all)
 
	targetSyncIntervalLength.WithLabelValues(sp.config.JobName).Observe(
		time.Since(start).Seconds(),
	)
	targetScrapePoolSyncsCounter.WithLabelValues(sp.config.JobName).Inc()
}
```
Target结构体定义：
```

// Target refers to a singular HTTP or HTTPS endpoint.
type Target struct {
	// Labels before any processing.
	discoveredLabels labels.Labels
	// Any labels that are added to this target and its metrics.
	labels labels.Labels
	// Additional URL parmeters that are part of the target URL.
	params url.Values
 
	mtx                sync.RWMutex
	lastError          error
	lastScrape         time.Time
	lastScrapeDuration time.Duration
	health             TargetHealth
	metadata           metricMetadataStore
}
```
targetgroup.Group构建Target。
scrapePool中主要初始化config、client等信息，并没有涉及到抓取采集点数据，然后对生成的scrapePool执行Sync方法，入参就是该抓取job当前所有采集点信息，这个方法就是对job的采集点信息进行处理
```
func (sp *scrapePool) Sync(tgs []*targetgroup.Group) 
```
遍历采集点，通过targetsFromGroup(tg, sp.config)解析采集点返回[]*Target，
```
var all []*Target
sp.droppedTargets = []*Target{}
for _, tg := range tgs {
 //基于targetgroup.Group构建target集合
 targets, err := targetsFromGroup(tg, sp.config)
 if err != nil {
  level.Error(sp.logger).Log("msg", "creating targets failed", "err", err)
  continue
 }
 for _, t := range targets {
  if t.Labels().Len() > 0 {//relabel后符合要求的采集点
   all = append(all, t)
  } else if t.DiscoveredLabels().Len() > 0 {//relabel后不符合要求的采集点：废弃
   sp.droppedTargets = append(sp.droppedTargets, t)
  }
 }
}
```
Target结构体主要字段如下，即将服务发现的采集点信息解析成scrape模块的Target信息，解析过程中会涉及relabel操作，从服务发现的目标采集点中过滤出符合要求的真实采集点，一个Target即代表一个将要真实触发Http请求对象：
```
type Target struct {
 //服务发现标签，即未经过relabel处理的标签
 discoveredLabels labels.Labels
 //经过relabel处理之后标签
 labels labels.Labels
 //http请求参数
 params url.Values
    //采集点状态：up、down、unknown
 health             TargetHealth
}
```
Target只是包含采集点信息，scrapeLoop实现loop接口，封装了发送http请求采集数据指标逻辑的Target执行单元
```
type loop interface {
 run(interval, timeout time.Duration, errc chan<- error)
 setForcedError(err error)
 stop()
 getCache() *scrapeCache
 disableEndOfRunStalenessMarkers()
}
```
其中run方法就是启动http数据抓取，入参interval指定循环抓取指标间隔；stop方法则是停止http数据采集。

我们来看下Target如何生成scrapeLoop：
```
if _, ok := sp.activeTargets[hash]; !ok {
    //生成targetScraper，其中封装了Target和client
 //Target封装了采集点请求IP、端口、请求参数等信息，通过这些信息构建HTTP请求Request
 //client是封装了认证信息的http请求客户端工具，用于将http请求request发送出去
 s := &targetScraper{Target: t, client: sp.client, timeout: timeout}
 l := sp.newLoop(scrapeLoopOptions{
  target:          t,
  scraper:         s,
  limit:           limit,
  honorLabels:     honorLabels,
  honorTimestamps: honorTimestamps,
  mrc:             mrc,
 })
   ...
}

if _, ok := sp.activeTargets[hash]; !ok {
    //sp.activeTargets不存在则表示新发现的采集点，则创建scrapeLoop
 
    //生成targetScraper，其中封装了Target和client
 //Target封装了采集点请求IP、端口、请求参数等信息，通过这些信息构建HTTP请求Request
 //client是封装了认证信息的http请求客户端工具，用于将http请求request发送出去
 s := &targetScraper{Target: t, client: sp.client, timeout: timeout}
 l := sp.newLoop(scrapeLoopOptions{
  target:          t,
  scraper:         s,
  limit:           limit,
  honorLabels:     honorLabels,
  honorTimestamps: honorTimestamps,
  mrc:             mrc,
 })

 sp.activeTargets[hash] = t
 sp.loops[hash] = l

 uniqueLoops[hash] = l
} else {
    //sp.activeTargets存在则可能：
    //1、重复的采集点：直接忽略即可
    //2、之前发现并启动的采集点：设置uniqueLoops[hash] = nil，则后续启动loop时不用启动
    
 //target在sp.activeTargets已存在，但是uniqueLoops不存在，说明该采集点之前就被发现过并被启动，当前发现的和之前一致未变
 //uniqueLoops[hash] = nil表示当前还是存在，但是不需要启动，后面对于sp.activeTargets存在但是uniqueLoops中不存在的采集点，则为采集点消失，需要停止loop并移除掉
 if _, ok := uniqueLoops[hash]; !ok {
  uniqueLoops[hash] = nil
 }
 sp.activeTargets[hash].SetDiscoveredLabels(t.DiscoveredLabels())
}
```
uniqueLoops存储当前抓取job所有有效采集点，不在该集合中的采集点需要停止并移除，如之前存在的采集点，但是当前又消失不见的采集点：
```
for hash := range sp.activeTargets {
 //uniqueLoops存储当前抓取job所有有效采集点，不在该集合中的采集点需要停止并移除，如之前存在的采集点，但是当前又消失不见的采集点
 //uniqueLoops中value=nil的是不需要启动，之前服务发现过并被启动的；value不是nil则表示需要启动
 if _, ok := uniqueLoops[hash]; !ok {
  //移除
  wg.Add(1)
  go func(l loop) {
   l.stop()
   wg.Done()
  }(sp.loops[hash])

  delete(sp.loops, hash)
  delete(sp.activeTargets, hash)
 }
}
```
scrapeLoop中还有个关键的类型targetScraper，它才是真正执行http请求组件，其实现scraper接口(如下)，其中scrape就是一次http请求逻辑封装：
```
type scraper interface {
 scrape(ctx context.Context, w io.Writer) (string, error)
 Report(start time.Time, dur time.Duration, err error)
 offset(interval time.Duration, jitterSeed uint64) time.Duration
}
```


sp.sync方法对比新的Target列表和原来的Target列表，若发现不在原来的Target列表中，则新建该targets的scrapeLoop，通过协程启动scrapeLoop的run方法，并发采集存储指标．然后判断原来的Target列表是否存在失效的Target，若存在，则移除
```
prometheus/scrape/scrape.go
 
// sync takes a list of potentially duplicated targets, deduplicates them, starts
// scrape loops for new targets, and stops scrape loops for disappeared targets.
// It returns after all stopped scrape loops terminated.
func (sp *scrapePool) sync(targets []*Target) {
	sp.mtx.Lock()
	defer sp.mtx.Unlock()
 
	var (
		uniqueTargets = map[uint64]struct{}{} 
		interval      = time.Duration(sp.config.ScrapeInterval) //指标采集周期
		timeout       = time.Duration(sp.config.ScrapeTimeout)  //指标采集超时时间
		limit         = int(sp.config.SampleLimit) //指标采集的限额
		honor         = sp.config.HonorLabels  //
		mrc           = sp.config.MetricRelabelConfigs
	)
 
	for _, t := range targets {
		t := t
		hash := t.hash()
		uniqueTargets[hash] = struct{}{}
　　　　 //若发现不在原来的Target列表中，则新建该target的scrapeLoop．
		if _, ok := sp.activeTargets[hash]; !ok {
			s := &targetScraper{Target: t, client: sp.client, timeout: timeout}
			l := sp.newLoop(t, s, limit, honor, mrc)
 
			sp.activeTargets[hash] = t
			sp.loops[hash] = l
            //通过协程启动scrapeLoop的run方法，采集存储指标
			go l.run(interval, timeout, nil)
		} else {
			// Need to keep the most updated labels information
			// for displaying it in the Service Discovery web page.
			sp.activeTargets[hash].SetDiscoveredLabels(t.DiscoveredLabels())
		}
	}
 
	var wg sync.WaitGroup
 
	// Stop and remove old targets and scraper loops.
    //判断原来的Target列表是否存在失效的Target，若存在则移除
	for hash := range sp.activeTargets {
		if _, ok := uniqueTargets[hash]; !ok {
			wg.Add(1)
			go func(l loop) {
 
				l.stop()
 
				wg.Done()
			}(sp.loops[hash])
 
			delete(sp.loops, hash)
			delete(sp.activeTargets, hash)
		}
	}
 
	// Wait for all potentially stopped scrapers to terminate.
	// This covers the case of flapping targets. If the server is under high load, a new scraper
	// may be active and tries to insert. The old scraper that didn't terminate yet could still
	// be inserting a previous sample set.
	wg.Wait()
}
```

sp.sync方法起了一个协程运行scrapePool的run方法去采集并存储监控指标(metrics)，run方法实现如下：
```
prometheus/scrape/scrape.go
 
func (sl *scrapeLoop) run(interval, timeout time.Duration, errc chan<- error) {
	select {
    //检测超时
	case <-time.After(sl.scraper.offset(interval)):
		// Continue after a scraping offset.
　　//停止，退出
	case <-sl.scrapeCtx.Done():
		close(sl.stopped)
		return
	}
 
	var last time.Time
　　//设置定时器
	ticker := time.NewTicker(interval)
	defer ticker.Stop()
 
mainLoop:
	for {
		select {
		case <-sl.ctx.Done():
			close(sl.stopped)
			return
		case <-sl.scrapeCtx.Done():
			break mainLoop
		default:
		}
 
		var (
			start             = time.Now()
			scrapeCtx, cancel = context.WithTimeout(sl.ctx, timeout)
		)
 
		// Only record after the first scrape.
		if !last.IsZero() {
			targetIntervalLength.WithLabelValues(interval.String()).Observe(
				time.Since(last).Seconds(),
			)
		}
        //获取上次scrape(拉取)指标(metric)占用空间
		b := sl.buffers.Get(sl.lastScrapeSize).([]byte)
　      //根据上次的占用的空间申请存储空间
		buf := bytes.NewBuffer(b)
 
        //开始scrape(拉取)指标
		contentType, scrapeErr := sl.scraper.scrape(scrapeCtx, buf)
		cancel()
 
		if scrapeErr == nil {
			b = buf.Bytes()
			// NOTE: There were issues with misbehaving clients in the past
			// that occasionally returned empty results. We don't want those
			// to falsely reset our buffer size.
			if len(b) > 0 {
　　　　　　　　　//存储本次scrape拉取磁盘占用的空间，留待下次scrape(拉取)使用
				sl.lastScrapeSize = len(b)
			}
		} else {
			level.Debug(sl.l).Log("msg", "Scrape failed", "err", scrapeErr.Error())
			if errc != nil {
				errc <- scrapeErr
			}
		}
 
		// A failed scrape is the same as an empty scrape,
		// we still call sl.append to trigger stale markers.
        //存储指标
		total, added, appErr := sl.append(b, contentType, start)
		if appErr != nil {
			level.Warn(sl.l).Log("msg", "append failed", "err", appErr)
			// The append failed, probably due to a parse error or sample limit.
			// Call sl.append again with an empty scrape to trigger stale markers.
			if _, _, err := sl.append([]byte{}, "", start); err != nil {
				level.Warn(sl.l).Log("msg", "append failed", "err", err)
			}
		}
 
		sl.buffers.Put(b)
 
		if scrapeErr == nil {
			scrapeErr = appErr
		}
 
		if err := sl.report(start, time.Since(start), total, added, scrapeErr); err != nil {
			level.Warn(sl.l).Log("msg", "appending scrape report failed", "err", err)
		}
		last = start
 
        //停止scrapeLoop
		select {
		case <-sl.ctx.Done():
			close(sl.stopped)
			return
		case <-sl.scrapeCtx.Done():
			break mainLoop
		case <-ticker.C:
		}
	}
 
	close(sl.stopped)
 
	sl.endOfRunStaleness(last, ticker, interval)
}

```
run方法主要实现两个功能：指标采集(scrape)和指标存储．此外，为了实现对象的复用，在采集(scrape)过程中，使用了sync.Pool机制提高性能，即每次采集(scrape)完成后，都会申请和本次采集(scrape)指标存储空间一样的大小的bytes，加入到buffer中，以备下次指标采集(scrape)直接使用．

## 指标采集(scrapeManager)配置初始化和应用
指标采集(scrapeManager)调用scrapeManager.ApplyConfig方法，完成配置初始化及应用，具体方法如下：
```

prometheus/scrape/manager.go
 
// ApplyConfig resets the manager's target providers and job configurations as defined by the new cfg.
func (m *Manager) ApplyConfig(cfg *config.Config) error {
   //操作前加锁
	m.mtxScrape.Lock()
   //完成后解锁
	defer m.mtxScrape.Unlock()ApplyConfig
 
　　// 创建一个map，key是job_name，value是结构体config.ScrapeConfig
	c := make(map[string]*config.ScrapeConfig)
	for _, scfg := range cfg.ScrapeConfigs {
		c[scfg.JobName] = scfg
	}
	m.scrapeConfigs = c
 
    //首次启动不执行
	// Cleanup and reload pool if config has changed.
	for name, sp := range m.scrapePools {
        // 若job_name在scrapePools中，不在scrapeConfigs中，则说明已经更新，停止该job_name对应的scrapePool
		if cfg, ok := m.scrapeConfigs[name]; !ok {
			sp.stop()
			delete(m.scrapePools, name)
		} else if !reflect.DeepEqual(sp.config, cfg) {
            // 若job_name在scrapePools中，也在scrapeConfigs中，但配置有变化，比如target增加或减少，需要重新加载
			sp.reload(cfg)
		}
	}
 
	return nil
}
```
调用reload方法重新加载配置文件：
```

prometheus/scrape/scrape.go
 
// reload the scrape pool with the given scrape configuration. The target state is preserved
// but all scrape loops are restarted with the new scrape configuration.
// This method returns after all scrape loops that were stopped have stopped scraping.
func (sp *scrapePool) reload(cfg *config.ScrapeConfig) {
	start := time.Now()
 
    // 操作前加锁
	sp.mtx.Lock()
　　// 完成后解锁
	defer sp.mtx.Unlock()
 
    // 生成client，用于获取指标(metircs)
	client, err := config_util.NewClientFromConfig(cfg.HTTPClientConfig, cfg.JobName)
	if err != nil {
		// Any errors that could occur here should be caught during config validation.
		level.Error(sp.logger).Log("msg", "Error creating HTTP client", "err", err)
	}
	sp.config = cfg
	sp.client = client
 
	var (
		wg       sync.WaitGroup
		interval = time.Duration(sp.config.ScrapeInterval)
		timeout  = time.Duration(sp.config.ScrapeTimeout)
		limit    = int(sp.config.SampleLimit)
		honor    = sp.config.HonorLabels
		mrc      = sp.config.MetricRelabelConfigs
	)
 
　　// 停止该scrapePool下对应的所有的oldLoop，更具配置创建所有的newLoop，并通过协程启动．
	for fp, oldLoop := range sp.loops {
		var (
			t       = sp.activeTargets[fp]
			s       = &targetScraper{Target: t, client: sp.client, timeout: timeout}
			newLoop = sp.newLoop(t, s, limit, honor, mrc)
		)
		wg.Add(1)
 
		go func(oldLoop, newLoop loop) {
			oldLoop.stop()
			wg.Done()
 
			go newLoop.run(interval, timeout, nil)
		}(oldLoop, newLoop)
 
		sp.loops[fp] = newLoop
	}
 
	wg.Wait()
	targetReloadIntervalLength.WithLabelValues(interval.String()).Observe(
		time.Since(start).Seconds(),
	)
}
```
指标采集(scrapeManager)功能分析结束































