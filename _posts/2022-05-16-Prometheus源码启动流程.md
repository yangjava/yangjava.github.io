---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus源码启动流程
Prometheus 启动过程中，主要包含cli参数解析。配置初始化。服务组件初始化。服务组件配置应用及启动各个服务组件。

## cli参数解析
prometheus采用Golang开发语言。服务启动代码入口：cmd/prometheus/main.go。

初始化flagConfig结构体
```
cfg := flagConfig{
  notifier: notifier.Options{
   // 默认注册器注册 cpu 和 go 指标收集器
   Registerer: prometheus.DefaultRegisterer,
  },
  web: web.Options{
   Registerer: prometheus.DefaultRegisterer,
   Gatherer:   prometheus.DefaultGatherer,
  },
  promlogConfig: promlog.Config{},
 }
```

flagConfig结构体定义如下，用于存放cli解析配置信息：
```
type flagConfig struct {
 configFile string

 localStoragePath    string
 notifier            notifier.Options
 forGracePeriod      model.Duration
 outageTolerance     model.Duration
 resendDelay         model.Duration
 web                 web.Options
 tsdb                tsdbOptions
 lookbackDelta       model.Duration
 webTimeout          model.Duration
 queryTimeout        model.Duration
 queryConcurrency    int
 queryMaxSamples     int
 RemoteFlushDeadline model.Duration

 featureList []string
 // These options are extracted from featureList
 // for ease of use.
 enablePromQLAtModifier     bool
 enablePromQLNegativeOffset bool
 enableExpandExternalLabels bool

 prometheusURL   string
 corsRegexString string

 promlogConfig promlog.Config
}
```
将cli参数绑定到flagConfig结构体的变量上：
```
a.Flag("config.file", "Prometheus configuration file path.").
  Default("prometheus.yml").StringVar(&cfg.configFile)

a.Flag("web.listen-address", "Address to listen on for UI, API, and telemetry.").
  Default("0.0.0.0:9090").StringVar(&cfg.web.ListenAddress)

a.Flag("web.read-timeout",
  "Maximum duration before timing out read of the request, and closing idle connections.").
  Default("5m").SetValue(&cfg.webTimeout)
...
```
prometheus支持的cli参数可以通过prometheus -h查看。比如这里最常见的几个参数：
```
--config.file:指定prometheus主配置文件路径
--web.listen-address:指定prometheus监听地址
--storage.tsdb.path:本地存储模式数据存放目录
--storage.tsdb.retention.time:本地存储模式存放时间，默认15天
--storage.tsdb.retention.size:数据保存的最大大小，支持的单位 B, KB, MB, GB, TB, PB, EB. Ex
```
解析cli参数
```
_, err := a.Parse(os.Args[1:])
```
`os.Args[1:]`获取到prometheus启动命令后所有参数信息，a.Parse()方法将命令行参数解析存放到上面初始化的flagConfig结构体的变量里，变量和命令行参数通过上面的a.Flag()方法进行绑定。

## 配置初始化
prometheus.yml合法性校验：config.LoadFile()方法用于解析校验prometheus.yml配置文件
```
if _, err := config.LoadFile(cfg.configFile, false, log.NewNopLogger()); err != nil {
 level.Error(logger).Log("msg", fmt.Sprintf("Error loading config (--config.file=%s)", cfg.configFile), "err", err)
 os.Exit(2)
}

// Now that the validity of the config is established, set the config
// success metrics accordingly, although the config isn't really loaded
// yet. This will happen later (including setting these metrics again),
// but if we don't do it now, the metrics will stay at zero until the
// startup procedure is complete, which might take long enough to
// trigger alerts about an invalid config.
configSuccess.Set(1)
configSuccessTime.SetToCurrentTime()
```
“注意，这里只解析prometheus.yml配置文件只用于校验配置文件合法性，并没有保留解析结果；校验成功后设置prometheus自身监控指标：prometheus_config_last_reload_successful和prometheus_config_last_reload_success_timestamp_seconds，避免prometheus_config_last_reload_successful为0的时间也就是启动的时间足够长可能触发配置无效的告警。

存储时长和存储空间容量配置参数初始化：
```
{ // Time retention settings.
    // --storage.tsdb.retention参数已被废弃，不建议使用，如果使用会打印warn警告日志
 if oldFlagRetentionDuration != 0 {
  level.Warn(logger).Log("deprecation_notice", "'storage.tsdb.retention' flag is deprecated use 'storage.tsdb.retention.time' instead.")
  cfg.tsdb.RetentionDuration = oldFlagRetentionDuration
 }

 // --storage.tsdb.retention.time赋值给cfg.tsdb.RetentionDuration
 if newFlagRetentionDuration != 0 {
  cfg.tsdb.RetentionDuration = newFlagRetentionDuration
 }

    // --storage.tsdb.retention.time和--storage.tsdb.retention.size都为0，即既没有限制存储最大时长，也没有限制存储最大空间容量场景下，默认存储15d(天)
 if cfg.tsdb.RetentionDuration == 0 && cfg.tsdb.MaxBytes == 0 {
  cfg.tsdb.RetentionDuration = defaultRetentionDuration
  level.Info(logger).Log("msg", "No time or size retention was set so using the default time retention", "duration", defaultRetentionDuration)
 }

 // Check for overflows. This limits our max retention to 100y.
 // 如果最大存储时长过大溢出整数范围，则限制为100年
 if cfg.tsdb.RetentionDuration < 0 {
  y, err := model.ParseDuration("100y")
  if err != nil {
   panic(err)
  }
  cfg.tsdb.RetentionDuration = y
  level.Warn(logger).Log("msg", "Time retention value is too high. Limiting to: "+y.String())
 }
}
```
storage.tsdb.max-block-duration参数初始化：

prometheus tsdb数据文件最终会被存储到本地chunks文件中，storage.tsdb.max-block-duration就是限制单个chunks文件最大大小。默认为存储时长的 10%且不超过31d(天)。
```
{
 if cfg.tsdb.MaxBlockDuration == 0 {
  maxBlockDuration, err := model.ParseDuration("31d")
  if err != nil {
   panic(err)
  }
  // When the time retention is set and not too big use to define the max block duration.
  if cfg.tsdb.RetentionDuration != 0 && cfg.tsdb.RetentionDuration/10 < maxBlockDuration {
   maxBlockDuration = cfg.tsdb.RetentionDuration / 10
  }
  cfg.tsdb.MaxBlockDuration = maxBlockDuration
 }
}
```

## 服务组件初始化

### Storage组件初始化
Prometheus的Storage组件是时序数据库，包含两个：localStorage和remoteStorage．localStorage当前版本指TSDB，用于对metrics的本地存储存储，remoteStorage用于metrics的远程存储，

其中fanoutStorage作为localStorage和remoteStorage的读写代理服务器．初始化流程如下

prometheus/cmd/prometheus/main.go
```
/**
 Prometheus的Storage组件是时序数据库，包含两个：localStorage和remoteStorage．localStorage当前版本指TSDB，
 用于对metrics的本地存储存储，remoteStorage用于metrics的远程存储，其中fanoutStorage作为localStorage和remoteStorage的读写代理服务器
  */
var (
 // 本地存储
 localStorage  = &readyStorage{}
 scraper       = &readyScrapeManager{}
 // 远程存储
 remoteStorage = remote.NewStorage(log.With(logger, "component", "remote"), prometheus.DefaultRegisterer, localStorage.StartTime, cfg.localStoragePath, time.Duration(cfg.RemoteFlushDeadline), scraper)
 // 读写代理服务器
 fanoutStorage = storage.NewFanout(logger, localStorage, remoteStorage)
)
```

### notifier 组件初始化
notifier组件用于发送告警信息给AlertManager，通过方法notifier.NewManager完成初始化

prometheus/cmd/prometheus/main.go
```
notifierManager = notifier.NewManager(&cfg.notifier, log.With(logger, "component", "notifier"))
```

### discoveryManagerScrape组件初始化
discoveryManagerScrape组件用于服务发现，该组件用于服务发现，当前版本支持多种服务发现系统，比如静态文件、eureka、kubertenes等，通过方法discovery.NewManager完成初始化，

prometheus/cmd/prometheus/main.go
```
discoveryManagerScrape  = discovery.NewManager(ctxScrape, log.With(logger, "component", "discovery manager scrape"), discovery.Name("scrape"))
```

### discoveryManagerNotify组件初始化
discoveryManagerNotify组件用于告警通知服务发现，比如AlertManager服务．也是通过方法discovery.NewManager完成初始化，不同的是，discoveryManagerNotify服务于notify，而discoveryManagerScrape服务与scrape

prometheus/cmd/prometheus/main.go
```
discoveryManagerNotify  = discovery.NewManager(ctxNotify, log.With(logger, "component", "discovery manager notify"), discovery.Name("notify")
```
discoveryManagerScrape组件和discoveryManagerNotify组件两个组件都是discovery.Manager类型组件，都是通过discovery.NewManager()方式创建，所以它们原理是一样的，因为它们都是用于服务发现。

### scrapeManager组件初始化
scrapeManager组件利用discoveryManagerScrape组件发现的targets，抓取对应targets的所有metrics，并将抓取的metrics存储到fanoutStorage中，通过方法scrape.NewManager完成初始化

prometheus/cmd/prometheus/main.go
```
scrapeManager = scrape.NewManager(log.With(logger, "component", "scrape manager"), fanoutStorage)
```

### queryEngine组件
queryEngine组件用于rules查询和计算，通过方法promql.NewEngine完成初始化

prometheus/cmd/prometheus/main.go
```
// 声明 promql 的引擎配置
opts = promql.EngineOpts{
 Logger:                   log.With(logger, "component", "query engine"),
 Reg:                      prometheus.DefaultRegisterer,
 MaxSamples:               cfg.queryMaxSamples,
 Timeout:                  time.Duration(cfg.queryTimeout), // 查询超时时
 ActiveQueryTracker:       promql.NewActiveQueryTracker(cfg.localStoragePath, cfg.queryConcurrency, log.With(logger, "component", "activeQueryTracker")),
 LookbackDelta:            time.Duration(cfg.lookbackDelta),
 NoStepSubqueryIntervalFn: noStepSubqueryInterval.Get,
 EnableAtModifier:         cfg.enablePromQLAtModifier,
 EnableNegativeOffset:     cfg.enablePromQLNegativeOffset,
}
// 初始化queryEngine
queryEngine = promql.NewEngine(opts)
```

### ruleManager组件初始化
ruleManager组件通过方法rules.NewManager完成初始化．其中rules.NewManager的参数涉及多个组件：存储，queryEngine和notifier，整个流程包含rule计算和发送告警

prometheus/cmd/prometheus/main.go
```
ruleManager = rules.NewManager(&rules.ManagerOptions{
    Appendable:      fanoutStorage,                        //存储器
    TSDB:            localStorage,　　　　　　　　　　　　　　//本地时序数据库TSDB
    QueryFunc:       rules.EngineQueryFunc(queryEngine, fanoutStorage), //rules计算
    NotifyFunc:      sendAlerts(notifierManager, cfg.web.ExternalURL.String()),　//告警通知
    Context:         ctxRule,　//用于控制ruleManager组件的协程
    ExternalURL:     cfg.web.ExternalURL,　//通过Web对外开放的URL
    Registerer:      prometheus.DefaultRegisterer, 
    Logger:          log.With(logger, "component", "rule manager"),
    OutageTolerance: time.Duration(cfg.outageTolerance), //当prometheus重启时，保持alert状态（https://ganeshvernekar.com/gsoc-2018/persist-for-state/）
    ForGracePeriod:  time.Duration(cfg.forGracePeriod),
    ResendDelay:     time.Duration(cfg.resendDelay),
}
```

### Web组件初始化
Web组件用于为Storage组件，queryEngine组件，scrapeManager组件， ruleManager组件和notifier 组件提供外部HTTP访问方式，

该组件用于web服务启动，这样可以通过http方式访问prometheus，比如调用promql语句。初始化代码如下

prometheus/cmd/prometheus/main.go
``` 
cfg.web.Context = ctxWeb
cfg.web.TSDB = localStorage.Get
cfg.web.Storage = fanoutStorage
cfg.web.QueryEngine = queryEngine
cfg.web.ScrapeManager = scrapeManager
cfg.web.RuleManager = ruleManager
cfg.web.Notifier = notifierManager
 
cfg.web.Version = &web.PrometheusVersion{
    Version:   version.Version,
    Revision:  version.Revision,
    Branch:    version.Branch,
    BuildUser: version.BuildUser,
    BuildDate: version.BuildDate,
    GoVersion: version.GoVersion,
}
 
cfg.web.Flags = map[string]string{}
 
// Depends on cfg.web.ScrapeManager so needs to be after cfg.web.ScrapeManager = scrapeManager
webHandler := web.New(log.With(logger, "component", "web"), &cfg.web)
```

## 服务组件配置应用
prometheus.yml是prometheus核心配置文件，待prometheus.yml配置信息解析到config.Config结构体中，然后调用各个组件定义的ApplyConfig(conf *config.Config)提取自己相关配置数据进行处理

通过以下代码，可以发现，除了服务组件ruleManager用的方法是Update，其他服务组件的在匿名函数中通过各自的ApplyConfig方法，实现配置的管理

prometheus/cmd/prometheus/main.go
```
reloaders := []func(cfg *config.Config) error{
    remoteStorage.ApplyConfig, //存储配置
    webHandler.ApplyConfig,    //web配置
    notifierManager.ApplyConfig, //notifier配置
    scrapeManager.ApplyConfig,　　//scrapeManger配置
　　//从配置文件中提取Section:scrape_configs
    func(cfg *config.Config) error {
        c := make(map[string]sd_config.ServiceDiscoveryConfig)
        for _, v := range cfg.ScrapeConfigs {
            c[v.JobName] = v.ServiceDiscoveryConfig
        }
        return discoveryManagerScrape.ApplyConfig(c)
    },
    //从配置文件中提取Section:alerting
    func(cfg *config.Config) error {
        c := make(map[string]sd_config.ServiceDiscoveryConfig)
        for _, v := range cfg.AlertingConfig.AlertmanagerConfigs {
            // AlertmanagerConfigs doesn't hold an unique identifier so we use the config hash as the identifier.
            b, err := json.Marshal(v)
            if err != nil {
                return err
            }
            c[fmt.Sprintf("%x", md5.Sum(b))] = v.ServiceDiscoveryConfig
        }
        return discoveryManagerNotify.ApplyConfig(c)
    },
    //从配置文件中提取Section:rule_files
    func(cfg *config.Config) error {
        // Get all rule files matching the configuration paths.
        var files []string
        for _, pat := range cfg.RuleFiles {
            fs, err := filepath.Glob(pat)
            if err != nil {
                // The only error can be a bad pattern.
                return fmt.Errorf("error retrieving rule files for %s: %s", pat, err)
            }
            files = append(files, fs...)
        }
        return ruleManager.Update(time.Duration(cfg.GlobalConfig.EvaluationInterval), files)
    },
}
```

其中，服务组件remoteStorage，webHandler，notifierManager和ScrapeManager的ApplyConfig方法，参数cfg *config.Config中传递的配置文件，是整个文件prometheus.yml
```
prometheus/scrape/manager.go
 
func (m *Manager) ApplyConfig(cfg *config.Config) error {
   .......
}
```
而服务组件discoveryManagerScrape和discoveryManagerNotify的Appliconfig方法，参数中传递的配置文件，是文件中的一个Section
```
prometheus/discovery/manager.go
 
func (m *Manager) ApplyConfig(cfg map[string]sd_config.ServiceDiscoveryConfig) error {
     ......
}
```

所以，需要利用匿名函数提前处理下，取出对应的Section

prometheus/cmd/prometheus/main.go
``` 
//从配置文件中提取Section:scrape_configs
func(cfg *config.Config) error {
    c := make(map[string]sd_config.ServiceDiscoveryConfig)
    for _, v := range cfg.ScrapeConfigs {
        c[v.JobName] = v.ServiceDiscoveryConfig
    }
    return discoveryManagerScrape.ApplyConfig(c)
},
//从配置文件中提取Section:alerting
func(cfg *config.Config) error {
    c := make(map[string]sd_config.ServiceDiscoveryConfig)
    for _, v := range cfg.AlertingConfig.AlertmanagerConfigs {
        // AlertmanagerConfigs doesn't hold an unique identifier so we use the config hash as the identifier.
        b, err := json.Marshal(v)
        if err != nil {
            return err
        }
        c[fmt.Sprintf("%x", md5.Sum(b))] = v.ServiceDiscoveryConfig
    }
    return discoveryManagerNotify.ApplyConfig(c)
}
```

服务组件ruleManager，在匿名函数中提取出Section:rule_files
```
prometheus/cmd/prometheus/main.go
 
//从配置文件中提取Section:rule_files
func(cfg *config.Config) error {
    // Get all rule files matching the configuration paths.
    var files []string
    for _, pat := range cfg.RuleFiles {
        fs, err := filepath.Glob(pat)
        if err != nil {
            // The only error can be a bad pattern.
            return fmt.Errorf("error retrieving rule files for %s: %s", pat, err)
        }
        files = append(files, fs...)
    }
    return ruleManager.Update(time.Duration(cfg.GlobalConfig.EvaluationInterval), files)
},
```

利用该组件内置的Update方法完成配置管理
```
prometheus/rules/manager.go
 
func (m *Manager) Update(interval time.Duration, files []string) error {
  .......
}
```
最后，通过reloadConfig方法，加载各个服务组件的配置项
```

prometheus/cmd/prometheus/main.go
 
func reloadConfig(filename string, logger log.Logger, rls ...func(*config.Config) error) (err error) {
	level.Info(logger).Log("msg", "Loading configuration file", "filename", filename)
 
	defer func() {
		if err == nil {
			configSuccess.Set(1)
			configSuccessTime.SetToCurrentTime()
		} else {
			configSuccess.Set(0)
		}
	}()
 
	conf, err := config.LoadFile(filename)
	if err != nil {
		return fmt.Errorf("couldn't load configuration (--config.file=%q): %v", filename, err)
	}
 
	failed := false
　　//通过一个for循环，加载各个服务组件的配置项
	for _, rl := range rls {
		if err := rl(conf); err != nil {
			level.Error(logger).Log("msg", "Failed to apply configuration", "err", err)
			failed = true
		}
	}
	if failed {
		return fmt.Errorf("one or more errors occurred while applying the new configuration (--config.file=%q)", filename)
	}
	promql.SetDefaultEvaluationInterval(time.Duration(conf.GlobalConfig.EvaluationInterval))
	level.Info(logger).Log("msg", "Completed loading of configuration file", "filename", filename)
	return nil
}
```

## 启动各个服务组件
prometheus组件初始化完成，并且完成对prometheus.yml配置中信息处理，下面就开始启动这些组件。

prometheus组件间协调使用了oklog/run的goroutine编排工具。这里引用了github.com/oklog/oklog/pkg/group包，实例化一个对象g

prometheus/cmd/prometheus/main.go
```
// "github.com/oklog/oklog/pkg/group"

var g run.Group

//加入goroutine（或者称为 actor）
g.Add(...)
g.Add(...)

//调用Run方法后会启动所有goroutine（或者称为 actor）
if err := g.Run(); err != nil {
 level.Error(logger).Log("err", err)
 os.Exit(1)
}
```

对象g中包含各个服务组件的入口，通过调用Add方法把把这些入口添加到对象g中，以组件scrapeManager为例：
```
prometheus/cmd/prometheus/main.go
 
{
    // Scrape manager.
　　//通过方法Add，把ScrapeManager组件添加到g中
    g.Add(
        func() error {
            // When the scrape manager receives a new targets list
            // it needs to read a valid config for each job.
            // It depends on the config being in sync with the discovery manager so
            // we wait until the config is fully loaded.
            <-reloadReady.C
　　　　　　　//ScrapeManager组件的启动函数
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

通过对象g，调用方法run，启动所有服务组件
```
prometheus/cmd/prometheus/main.go
 
if err := g.Run(); err != nil {
    level.Error(logger).Log("err", err)
    os.Exit(1)
}
level.Info(logger).Log("msg", "See you next time!")
```

## prometheus启动以下协程

优雅退出
```
// 创建监听退出chan
term := make(chan os.Signal, 1)
// pkill信号syscall.SIGTERM
// ctrl+c信号os.Interrupt
// 首先我们创建一个os.Signal channel，然后使用signal.Notify注册要接收的信号。
signal.Notify(term, os.Interrupt, syscall.SIGTERM)
cancel := make(chan struct{})

g.Add(
 func() error {
  // Don't forget to release the reloadReady channel so that waiting blocks can exit normally.
  select {
  case <-term: //监听到系统ctrl+c或kill等程序退出信号
   level.Warn(logger).Log("msg", "Received SIGTERM, exiting gracefully...")
   reloadReady.Close()
  case <-webHandler.Quit()://监听到web服务停止
   level.Warn(logger).Log("msg", "Received termination request via web service, exiting gracefully...")
  case <-cancel://其它需要退出程序信号
   reloadReady.Close()
  }
  return nil
 },
 func(err error) {
  close(cancel)
 },
)
```

discoveryManagerScrape组件启动：抓取scrape组件自动发现targets
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

discoveryManagerNotify组件启动：告警AlertManager服务自动发现
```
g.Add(
 func() error {
  err := discoveryManagerNotify.Run()
  level.Info(logger).Log("msg", "Notify discovery manager stopped")
  return err
 },
 func(err error) {
  level.Info(logger).Log("msg", "Stopping notify discovery manager...")
  cancelNotify()
 },
)
```
scrapeManager组件启动，用于监控指标抓取
```
g.Add(
 func() error {
  // When the scrape manager receives a new targets list
  // it needs to read a valid config for each job.
  // It depends on the config being in sync with the discovery manager so
  // we wait until the config is fully loaded.
  // 当所有配置都准备好
  // scrape manager 获取到新的抓取目标列表时，它需要读取每个 job 的合法的配置。
  // 这依赖于正在被 discovery manager 同步的配置文件，所以要等到配置加载完成。
  <-reloadReady.C
  level.Info(logger).Log("cus_msg", "--->ScrapeManager reloadReady")
  // 启动scrapeManager
  //ScrapeManager组件的启动函数
  err := scrapeManager.Run(discoveryManagerScrape.SyncCh())
  level.Info(logger).Log("msg", "Scrape manager stopped")
  return err
 },
 func(err error) {
  // 失败处理
  // Scrape manager needs to be stopped before closing the local TSDB
  // so that it doesn't try to write samples to a closed storage.
  level.Info(logger).Log("msg", "Stopping scrape manager...")
  scrapeManager.Stop()
 },
)
```

配置动态加载：通过监听kill -HUP pid信号和curl -XPOST `http://ip:9090/-/reload` 方式实现配置动态加载；
```
// Make sure that sighup handler is registered with a redirect to the channel before the potentially
// long and synchronous tsdb init.
// tsdb 初始化时间可能很长，确保 sighup 处理函数在这之前注册完成。
hup := make(chan os.Signal, 1)
signal.Notify(hup, syscall.SIGHUP)
cancel := make(chan struct{})
g.Add(
 func() error {
  <-reloadReady.C
  for {
   select {
   case <-hup:
    if err := reloadConfig(cfg.configFile, cfg.enableExpandExternalLabels, logger, noStepSubqueryInterval, reloaders...); err != nil {
     level.Error(logger).Log("msg", "Error reloading config", "err", err)
    }
   case rc := <-webHandler.Reload():
    if err := reloadConfig(cfg.configFile, cfg.enableExpandExternalLabels, logger, noStepSubqueryInterval, reloaders...); err != nil {
     level.Error(logger).Log("msg", "Error reloading config", "err", err)
     rc <- err
    } else {
     rc <- nil
    }
   case <-cancel:
    return nil
   }
  }
 },
 func(err error) {
  // Wait for any in-progress reloads to complete to avoid
  // reloading things after they have been shutdown.
  cancel <- struct{}{}
 },
)
```

配置加载：通过reloadConfig()方法将prometheus.yml配置文件加载进来
```
cancel := make(chan struct{})
g.Add(
 func() error {
  select {
  case <-dbOpen:
  // In case a shutdown is initiated before the dbOpen is released
  case <-cancel:
   reloadReady.Close()
   return nil
  }

  //加载解析prometheus.yml配置文件，并调用各个组件ApplyConfig()方法将配置传入
  if err := reloadConfig(cfg.configFile, cfg.enableExpandExternalLabels, logger, noStepSubqueryInterval, reloaders...); err != nil {
   return errors.Wrapf(err, "error loading config from %q", cfg.configFile)
  }
  //配置加载完毕，执行reloadReady.Close()关闭reloadReady.C通道，这样  <-reloadReady.C 阻塞地方可以继续向下执行
  reloadReady.Close()
  webHandler.Ready()
  level.Info(logger).Log("msg", "Server is ready to receive web requests.")
  <-cancel
  return nil
 },
 func(err error) {
  close(cancel)
 },
)
```

ruleManager组件启动，用于进行rule规则计算：
```
g.Add(
 func() error {
  <-reloadReady.C
  ruleManager.Run()
  return nil
 },
 func(err error) {
  ruleManager.Stop()
 },
)
```

tsdb模块启动，用户监控数据存储：
```
opts := cfg.tsdb.ToTSDBOptions()
cancel := make(chan struct{})
g.Add(
 func() error {
  level.Info(logger).Log("msg", "Starting TSDB ...")
  if cfg.tsdb.WALSegmentSize != 0 {
   if cfg.tsdb.WALSegmentSize < 10*1024*1024 || cfg.tsdb.WALSegmentSize > 256*1024*1024 {
    return errors.New("flag 'storage.tsdb.wal-segment-size' must be set between 10MB and 256MB")
   }
  }
  if cfg.tsdb.MaxBlockChunkSegmentSize != 0 {
   if cfg.tsdb.MaxBlockChunkSegmentSize < 1024*1024 {
    return errors.New("flag 'storage.tsdb.max-block-chunk-segment-size' must be set over 1MB")
   }
  }
  db, err := openDBWithMetrics(
   cfg.localStoragePath,
   logger,
   prometheus.DefaultRegisterer,
   &opts,
  )

  if err != nil {
   return errors.Wrapf(err, "opening storage failed")
  }

  switch fsType := prom_runtime.Statfs(cfg.localStoragePath); fsType {
  case "NFS_SUPER_MAGIC":
   level.Warn(logger).Log("fs_type", fsType, "msg", "This filesystem is not supported and may lead to data corruption and data loss. Please carefully read https://prometheus.io/docs/prometheus/latest/storage/ to learn more about supported filesystems.")
  default:
   level.Info(logger).Log("fs_type", fsType)
  }

  level.Info(logger).Log("msg", "TSDB started")
  level.Debug(logger).Log("msg", "TSDB options",
   "MinBlockDuration", cfg.tsdb.MinBlockDuration,
   "MaxBlockDuration", cfg.tsdb.MaxBlockDuration,
   "MaxBytes", cfg.tsdb.MaxBytes,
   "NoLockfile", cfg.tsdb.NoLockfile,
   "RetentionDuration", cfg.tsdb.RetentionDuration,
   "WALSegmentSize", cfg.tsdb.WALSegmentSize,
   "AllowOverlappingBlocks", cfg.tsdb.AllowOverlappingBlocks,
   "WALCompression", cfg.tsdb.WALCompression,
  )

  startTimeMargin := int64(2 * time.Duration(cfg.tsdb.MinBlockDuration).Seconds() * 1000)
  localStorage.Set(db, startTimeMargin)
  time.Sleep(time.Duration(10)*time.Second)
  close(dbOpen)
  <-cancel
  return nil
 },
 func(err error) {
  if err := fanoutStorage.Close(); err != nil {
   level.Error(logger).Log("msg", "Error stopping storage", "err", err)
  }
  close(cancel)
 },
)
```

webHandler组件启动，prometheus可以接收http请求：
```
g.Add(
 func() error {
  if err := webHandler.Run(ctxWeb, listener, *webConfig); err != nil {
   return errors.Wrapf(err, "error starting web server")
  }
  return nil
 },
 func(err error) {
  cancelWeb()
 },
)
```

notifierManager组件启动，将告警数据发送给AlertManager服务：
```
g.Add(
 func() error {
  // When the notifier manager receives a new targets list
  // it needs to read a valid config for each job.
  // It depends on the config being in sync with the discovery manager
  // so we wait until the config is fully loaded.
  <-reloadReady.C
  notifierManager.Run(discoveryManagerNotify.SyncCh())
  level.Info(logger).Log("msg", "Notifier manager stopped")
  return nil
 },
 func(err error) {
  notifierManager.Stop()
 },
)
```

上面分析的Prometheus启动流程中最为关键的是：通过oklog/run的协程编排工具启动10个协程组件，每个协程组件都有各自功能

- 优雅退出组件：主要用于监听系统发出的kill和Ctrl+C信号，用于Prometheus优雅退出；
- discoveryManagerScrape和discoveryManagerNotify这两个是服务发现组件，分别用于发现targets和alertmanager服务，通过通道传递给scrapeManager和notifierManager组件，scrapeManager组件拿到targets开始抓取监控指标，notifierManager拿到alertmanager服务发送告警数据；
- 配置加载组件：主要用于加载prometheus.yml配置并初始化到Config结构体中，然后遍历执行reloader，将解析的配置数据Config传递给相关组件进行处理，reloader信息见前面分析的ApplyConfig一节；
- 配置加载完成后会向reloadReady.C通道发送信号，scrapeManager组件、配置动态加载组件、ruleManager组件和notifierManager这四个组件会监听该信号才能执行，即这四个组件依赖配置加载完成，reloader执行完成；
- TSDB组件：主要用于scrapeManager抓取的监控指标存储时序数据库；
- webHandler组件：prometheus启动http服务器，这样可以查询到prometheus相关数据，如执行promql语句；
- 动态配置加载：该组件主要在配置加载完成后启动监听kill -HUP或curl -XPOST http://ip:port/-/reload信号，动态重新加载prometheus.yml配置文件；
- ruleManager组件：ruleManager组件主要用于rule规则文件计算，包括record rule和alert rule规则文件。