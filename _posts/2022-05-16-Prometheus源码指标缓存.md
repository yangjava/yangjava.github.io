---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus源码指标缓存


## 指标缓存(scrapeCache)
Prometheus通过scrapeManager抓取的指标(metrics)可通过本地TSDB时序数据库存储，简单高效，但无法持久化数据．所以，可根据需求，选择本地存储或远端存储．本文不涉及存储的具体实现，而是分析指标(metrics)在存储前合法性校验，即指标缓存层(scrapeCache)．

由指标采集(scrapeManager)可知，scrapeLoop结构体包含scrapeCache，调用scrapeLoop.append方法处理指标(metrics)存储．在方法append中，把每个指标(metirics)放到结构体scrapeCache对应的方法中进行合法性验证，过滤和存储．

scrapeCache结构及实例化如下：
```
prometheus/scrape/scrape.go
 
// scrapeCache tracks mappings of exposed metric strings to label sets and
// storage references. Additionally, it tracks staleness of series between
// scrapes.
type scrapeCache struct {
	iter uint64 // Current scrape iteration. //被缓存的迭代次数
 
	// Parsed string to an entry with information about the actual label set
	// and its storage reference.
	series map[string]*cacheEntry  //map类型,key是metric,value是cacheEntry结构体
 
	// Cache of dropped metric strings and their iteration. The iteration must
	// be a pointer so we can update it without setting a new entry with an unsafe
	// string in addDropped().
	droppedSeries map[string]*uint64 //缓存不合法指标(metrics)
 
	// seriesCur and seriesPrev store the labels of series that were seen
	// in the current and previous scrape.
	// We hold two maps and swap them out to save allocations.
	seriesCur  map[uint64]labels.Labels //缓存本次scrape的指标(metrics)
	seriesPrev map[uint64]labels.Labels  //缓存上次scrape的指标(metrics)
 
	metaMtx  sync.Mutex //同步锁
	metadata map[string]*metaEntry //元数据
}
 
 
func newScrapeCache() *scrapeCache {
	return &scrapeCache{
		series:        map[string]*cacheEntry{},
		droppedSeries: map[string]*uint64{},
		seriesCur:     map[uint64]labels.Labels{},
		seriesPrev:    map[uint64]labels.Labels{},
		metadata:      map[string]*metaEntry{},
	}
}
```

scrapeCache结构体包含多种方法（按照在方法append出现的顺序列出）

判断指标(metrics)的合法性
```

prometheus/scrape/scrape.go
 
func (c *scrapeCache) getDropped(met string) bool {
    // 判断metric是否在非法的dropperSeries的map类型里, key是metric, value是迭代版本号
	iterp, ok := c.droppedSeries[met]
	if ok {
		*iterp = c.iter
	}
	return ok
}
```

根据指标(metrics)信息获取结构体cacheEntry
```

prometheus/scrape/scrape.go
 
func (c *scrapeCache) get(met string) (*cacheEntry, bool) {
    series是map类型,key是metric,value是结构体cachaEntry
	e, ok := c.series[met]
	if !ok {
		return nil, false
	}
	e.lastIter = c.iter
	return e, true
}
```
其中，cacheEntry结构体包含metric以下几个参数
```
prometheus/scrape/scrape.go
 
type cacheEntry struct {
        //添加到本地数据库或者远程数据库的一个返回值: 
        //A reference number is returned which can be 
        //used to add further samples in the same or later transactions.
	//Returned reference numbers are ephemeral and may be rejected in calls
	//to AddFast() at any point. Adding the sample via Add() returns a new
	//reference number.
	//If the reference is 0 it must not be used for caching.
	ref      uint64  
	lastIter uint64  //上一个版本号
	hash     uint64  // hash值
	lset     labels.Labels //包含的labels
}
```

添加不带时间戳的指标(metrics)到map类型seriesCur,以metric lset的hash值作为唯一标识
```
prometheus/scrape/scrape.go
 
func (c *scrapeCache) trackStaleness(hash uint64, lset labels.Labels) {
	c.seriesCur[hash] = lset
}
```

添加无效指标(metrics)到map类型droppedSeries
```
prometheus/scrape/scrape.go
 
func (c *scrapeCache) addDropped(met string) {
	iter := c.iter
    // droppedSeries是map类型,以metric作为key, 版本作为value
	c.droppedSeries[met] = &iter
}
```

根据指标(metircs)信息添加该meitric的结构体cacheEntry
```

prometheus/scrape/scrape.go
 
func (c *scrapeCache) addRef(met string, ref uint64, lset labels.Labels, hash uint64) {
	if ref == 0 {
		return
	}
    // series是map类型, key为metric, value是结构体cacheEntry
	c.series[met] = &cacheEntry{ref: ref, lastIter: c.iter, lset: lset, hash: hash}
}
```

比较两个map：seriesCur和seriesPrev，查找过期指标
```

prometheus/scrape/scrape.go
 
func (c *scrapeCache) forEachStale(f func(labels.Labels) bool) {
	for h, lset := range c.seriesPrev {
        // 判断之前的metric是否在当前的seriesCur里.
		if _, ok := c.seriesCur[h]; !ok {
			if !f(lset) {
				break
			}
		}
	}
}
```

整理scrapeCache结构体中包含的几个map类型的缓存
```
prometheus/scrape/scrape.go
 
func (c *scrapeCache) iterDone() {
	// All caches may grow over time through series churn
	// or multiple string representations of the same metric. Clean up entries
	// that haven't appeared in the last scrape.
    // 只保存最近两个版本的合法的series
	for s, e := range c.series {
		if c.iter-e.lastIter > 2 {
			delete(c.series, s)
		}
	}
    // 只保留最近两个版本的droppedSeries
	for s, iter := range c.droppedSeries {
		if c.iter-*iter > 2 {
			delete(c.droppedSeries, s)
		}
	}
	c.metaMtx.Lock()
    // 保留最近十个版本的metadata
	for m, e := range c.metadata {
		// Keep metadata around for 10 scrapes after its metric disappeared.
		if c.iter-e.lastIter > 10 {
			delete(c.metadata, m)
		}
	}
	c.metaMtx.Unlock()
 
	// Swap current and previous series.
    // 把上次采集的指标(metircs)集合和本次采集的指标集(metrics)互换
	c.seriesPrev, c.seriesCur = c.seriesCur, c.seriesPrev
 
	// We have to delete every single key in the map.
    // 删除本地获取的指标集(metrics)
	for k := range c.seriesCur {
		delete(c.seriesCur, k)
	}
 
    //迭代版本号自增
	c.iter++
}
```
append方法利用scrapeCache的以上几个方法, 可实现对指标(metircs)的合法性验证,过滤及存储.

其中,存储路径分两条
- 如果能够在scrapeCache中找到该metric的cacheEntry, 说明之前添加过,则做AddFast存储路径
- 如果在scrapeCache中不能找到该metric的cacheEntry, 需要生成指标的lables,hash及ref等,通过Add方法jinxing存储, 然后把该指标(metric)的cacheEntry加到scrapeCache中.

具体实现如下:
```
func (sl *scrapeLoop) append(b []byte, contentType string, ts time.Time) (total, added int, err error) {
	var (
		app            = sl.appender() // 获取指标(metrics)存储器(本地或远端),在服务启动的时候已经指定
		p              = textparse.New(b, contentType) // 创建指标(metrics)解析器,其中b包含所有一个target的所有指标(metrics)
		defTime        = timestamp.FromTime(ts)
		numOutOfOrder  = 0
		numDuplicates  = 0
		numOutOfBounds = 0
	)
	var sampleLimitErr error
 
loop:
	for {
		var et textparse.Entry
		if et, err = p.Next(); err != nil {
			if err == io.EOF {
				err = nil
			}
			break // 退出的条件是该target获取的所有指标(metrics)存储结束
		}
		switch et {
		case textparse.EntryType:
			sl.cache.setType(p.Type())
			continue
		case textparse.EntryHelp:
			sl.cache.setHelp(p.Help())
			continue
		case textparse.EntryUnit:
			sl.cache.setUnit(p.Unit())
			continue
		case textparse.EntryComment:
			continue
		default:
		}
		total++
 
		t := defTime
		met, tp, v := p.Series() //获取一个指标(metric)的信息
		if tp != nil {
			t = *tp
		}
 
        // 判断指标(metric)是否合法
		if sl.cache.getDropped(yoloString(met)) {
			continue
		}
        // 根据指标(metric)判断是否在scrapeCache中存在该指标的cacheEntry
		ce, ok := sl.cache.get(yoloString(met))
		if ok {
            // 若存在,则存储指标(metric)到数据库
			switch err = app.AddFast(ce.lset, ce.ref, t, v); err {
			case nil:
				if tp == nil {
					sl.cache.trackStaleness(ce.hash, ce.lset)
				}
           // 错误处理
			case storage.ErrNotFound:
				ok = false
			case storage.ErrOutOfOrderSample:
				numOutOfOrder++
				level.Debug(sl.l).Log("msg", "Out of order sample", "series", string(met))
				targetScrapeSampleOutOfOrder.Inc()
				continue
			case storage.ErrDuplicateSampleForTimestamp:
				numDuplicates++
				level.Debug(sl.l).Log("msg", "Duplicate sample for timestamp", "series", string(met))
				targetScrapeSampleDuplicate.Inc()
				continue
			case storage.ErrOutOfBounds:
				numOutOfBounds++
				level.Debug(sl.l).Log("msg", "Out of bounds metric", "series", string(met))
				targetScrapeSampleOutOfBounds.Inc()
				continue
			case errSampleLimit:
				// Keep on parsing output if we hit the limit, so we report the correct
				// total number of samples scraped.
				sampleLimitErr = err
				added++
				continue
			default:
				break loop
			}
		}
		if !ok {
			var lset labels.Labels
            // 若不存在, 把指标(metric)格式从ASCII转换labels字典:先把ASCII转换成字串川,然后把字符串分割成label的name和value,具体的不再细讲
			mets := p.Metric(&lset)
            // 对指标(metirc)的labels做hash值:先在每个label的key和value后面分别加上255,然后根据golang的库函数xxhash,计算hash值
			hash := lset.Hash()
 
			// Hash label set as it is seen local to the target. Then add target labels
			// and relabeling and store the final label set.
            // 根据sp.config.HonorLabels和sp.config.MetricRelabelConfigs规则,对指标(metric)的lables进行重置
			lset = sl.sampleMutator(lset)
 
			// The label set may be set to nil to indicate dropping.
            // 若重置后的labels为空,则把该指标(metirc)加到droppedSeries
			if lset == nil {
				sl.cache.addDropped(mets)
				continue
			}
 
			var ref uint64
            // 添加到本地或远程存储中,返回值ref用于后期AddFast函数的判断
			ref, err = app.Add(lset, t, v)
			// TODO(fabxc): also add a dropped-cache?
			switch err {
			case nil:
			case storage.ErrOutOfOrderSample:
				err = nil
				numOutOfOrder++
				level.Debug(sl.l).Log("msg", "Out of order sample", "series", string(met))
				targetScrapeSampleOutOfOrder.Inc()
				continue
			case storage.ErrDuplicateSampleForTimestamp:
				err = nil
				numDuplicates++
				level.Debug(sl.l).Log("msg", "Duplicate sample for timestamp", "series", string(met))
				targetScrapeSampleDuplicate.Inc()
				continue
			case storage.ErrOutOfBounds:
				err = nil
				numOutOfBounds++
				level.Debug(sl.l).Log("msg", "Out of bounds metric", "series", string(met))
				targetScrapeSampleOutOfBounds.Inc()
				continue
			case errSampleLimit:
				sampleLimitErr = err
				added++
				continue
			default:
				level.Debug(sl.l).Log("msg", "unexpected error", "series", string(met), "err", err)
				break loop
			}
			if tp == nil {
				// Bypass staleness logic if there is an explicit timestamp.
				sl.cache.trackStaleness(hash, lset)
			}
            // 根据指标(metircs)信息添加该meitric的结构体cacheEntry
			sl.cache.addRef(mets, ref, lset, hash)
		}
        // 个数加1
		added++
	}
    // 错误处理
	if sampleLimitErr != nil {
		if err == nil {
			err = sampleLimitErr
		}
		// We only want to increment this once per scrape, so this is Inc'd outside the loop.
		targetScrapeSampleLimit.Inc()
	}
	if numOutOfOrder > 0 {
		level.Warn(sl.l).Log("msg", "Error on ingesting out-of-order samples", "num_dropped", numOutOfOrder)
	}
	if numDuplicates > 0 {
		level.Warn(sl.l).Log("msg", "Error on ingesting samples with different value but same timestamp", "num_dropped", numDuplicates)
	}
	if numOutOfBounds > 0 {
		level.Warn(sl.l).Log("msg", "Error on ingesting samples that are too old or are too far into the future", "num_dropped", numOutOfBounds)
	}
	if err == nil {
		sl.cache.forEachStale(func(lset labels.Labels) bool {
			// Series no longer exposed, mark it stale.
			_, err = app.Add(lset, defTime, math.Float64frombits(value.StaleNaN))
			switch err {
			case storage.ErrOutOfOrderSample, storage.ErrDuplicateSampleForTimestamp:
				// Do not count these in logging, as this is expected if a target
				// goes away and comes back again with a new scrape loop.
				err = nil
			}
			return err == nil
		})
	}
    // 若出错,则做回滚操作
	if err != nil {
		app.Rollback()
		return total, added, err
	}
    // 否则,数据库做提交处理
	if err := app.Commit(); err != nil {
		return total, added, err
	}
    // 整理scrapeCache结构体中包含的几个map类型的缓存
	sl.cache.iterDone()
	return total, added, nil
}
示例:
(dlv) p lset
github.com/prometheus/prometheus/pkg/labels.Labels len: 4, cap: 4, [
	{
		Name: "__name__",
		Value: "go_gc_duration_seconds",},
	{
		Name: "instance",
		Value: "localhost:9090",},
	{
		Name: "job",
		Value: "prometheus",},
	{
		Name: "quantile",
		Value: "0.25",},
]
(dlv) p mets
"go_gc_duration_seconds{quantile=\"0.25\"}"
(dlv) p hash
10509196582921338824
(dlv) p ref
2
```
至此，指标缓存(scrapeCache)功能分析结束




































