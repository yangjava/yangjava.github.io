---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus源码指标采集

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


