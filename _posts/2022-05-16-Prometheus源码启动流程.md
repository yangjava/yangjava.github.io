---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus源码启动流程
Prometheus 启动过程中，主要包含服务组件初始化，服务组件配置应用及启动各个服务组件三个部分。

## 服务组件初始化

### Storage组件初始化

Prometheus的Storage组件是时序数据库，包含两个：localStorage和remoteStorage．localStorage当前版本指TSDB，用于对metrics的本地存储存储，remoteStorage用于metrics的远程存储，其中fanoutStorage作为localStorage和remoteStorage的读写代理服务器．初始化流程如下

prometheus/cmd/prometheus/main.go
```
localStorage  = &tsdb.ReadyStorage{} //本地存储
remoteStorage = remote.NewStorage(log.With(logger, "component", "remote"), //远端存储 localStorage.StartTime, time.Duration(cfg.RemoteFlushDeadline))
fanoutStorage = storage.NewFanout(logger, localStorage, remoteStorage) //读写代理服务器
```

### notifier 组件初始化
notifier组件用于发送告警信息给AlertManager，通过方法notifier.NewManager完成初始化
```
prometheus/cmd/prometheus/main.go
 
notifierManager = notifier.NewManager(&cfg.notifier, log.With(logger, "component", "notifier"))
```

### discoveryManagerScrape组件初始化
discoveryManagerScrape组件用于服务发现，当前版本支持多种服务发现系统，比如kuberneters等，通过方法discovery.NewManager完成初始化，
```
prometheus/cmd/prometheus/main.go
 
discoveryManagerScrape  = discovery.NewManager(ctxScrape, log.With(logger, "component", "discovery manager scrape"), discovery.Name("scrape"))
```

### discoveryManagerNotify组件初始化
discoveryManagerNotify组件用于告警通知服务发现，比如AlertManager服务．也是通过方法discovery.NewManager完成初始化，不同的是，discoveryManagerNotify服务于notify，而discoveryManagerScrape服务与scrape
```
prometheus/cmd/prometheus/main.go
 
discoveryManagerNotify  = discovery.NewManager(ctxNotify, log.With(logger, "component", "discovery manager notify"), discovery.Name("notify")
```

### scrapeManager组件初始化
scrapeManager组件利用discoveryManagerScrape组件发现的targets，抓取对应targets的所有metrics，并将抓取的metrics存储到fanoutStorage中，通过方法scrape.NewManager完成初始化
```
prometheus/cmd/prometheus/main.go
 
scrapeManager = scrape.NewManager(log.With(logger, "component", "scrape manager"), fanoutStorage)
```

### queryEngine组件
queryEngine组件用于rules查询和计算，通过方法promql.NewEngine完成初始化
```
prometheus/cmd/prometheus/main.go
 
opts = promql.EngineOpts{
    Logger:        log.With(logger, "component", "query engine"),
    Reg:           prometheus.DefaultRegisterer,
    MaxConcurrent: cfg.queryConcurrency,　　　　　　　//最大并发查询个数
    MaxSamples:    cfg.queryMaxSamples,
    Timeout:       time.Duration(cfg.queryTimeout),　//查询超时时间
}
queryEngine = promql.NewEngine(opts)
```

### ruleManager组件初始化
ruleManager组件通过方法rules.NewManager完成初始化．其中rules.NewManager的参数涉及多个组件：存储，queryEngine和notifier，整个流程包含rule计算和发送告警
```

prometheus/cmd/prometheus/main.go
 
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
Web组件用于为Storage组件，queryEngine组件，scrapeManager组件， ruleManager组件和notifier 组件提供外部HTTP访问方式，初始化代码如下
```
prometheus/cmd/prometheus/main.go
 
 
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
通过以下代码，可以发现，除了服务组件ruleManager用的方法是Update，其他服务组件的在匿名函数中通过各自的ApplyConfig方法，实现配置的管理
```
prometheus/cmd/prometheus/main.go
 
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

其中，服务组件remoteStorage，webHandler，notifierManager和ScrapeManager的ApplyConfig方法，参数cfg *config.Config中传递的配置文件，是整个文件prometheus.yml，点击prometheus.yml查看一个完整的配置文件示例
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
```

prometheus/cmd/prometheus/main.go
 
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
这里引用了github.com/oklog/oklog/pkg/group包，实例化一个对象g
```
prometheus/cmd/prometheus/main.go
 
// "github.com/oklog/oklog/pkg/group"
var g group.Group
{
　　......
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







