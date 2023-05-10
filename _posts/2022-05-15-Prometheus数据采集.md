---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---
# Prometheus数据采集
Prometheus是通过pull从各个Target(目标)Exporter中去获取数据。

## 数据采集业务流程框架
数据采集主要采用scrape模块
- 由scrape.Manager管理所有的抓取对象；
- 所有的抓取对象按group分组，每个group是一个job_name；
- 每个group下含多个scrapeTarget，即具体的抓取目标endpoint；
- 对每个目标endpoint，启动一个抓取goroutine，按照interval间隔循环的抓取对象的指标；


首先，我们需要告诉Prometheus要通过什么样的方式去自动获取Targets。这就是我们的scrape_configs。

举例说明：假如prometheus.yaml中的抓取配置为：
```yaml
global:
  scrape_interval: 60s
  scrape_timeout: 40s
scrape_configs:
  - job_name: "monitor"
    static_configs:
      - targets: ['192.168.101.9:11504']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['10.21.1.74:9100', '192.168.101.9:9100']
```
那么，抓取对象将按如下结构分组：
```json
{
    "monitor": [
        {
            "Targets": [
                {
                    "__address__": "192.168.101.9:11504"
                }
            ],
            "Labels": null,
            "Source": "0"
        }
    ],
    "node-exporter": [
        {
            "Targets": [
                {
                    "__address__": "10.21.1.74:9100"
                },
                {
                    "__address__": "192.168.101.9:9100"
                }
            ],
            "Labels": null,
            "Source": "0"
        }
    ]
}
```

## scrape_configs的元数据解析
scrape_configs的元配置数据信息肯定是在启动的时候就已经加载好了。所以我们去观察main.go。

```
func main() {
		// 对scrape_configs配置建立providers，providers会提供最终所有的Target
		func(cfg *config.Config) error {
			c := make(map[string]sd_config.ServiceDiscoveryConfig)
			for _, v := range cfg.ScrapeConfigs {
				c[v.JobName] = v.ServiceDiscoveryConfig
			}
			return discoveryManagerScrape.ApplyConfig(c)
		},
}
```

上面的匿名函数中，将读取到的scrape_configs和其job名建立映射，然后调用discoveryManagerScrape.ApplyConfig去建立providers(Targets的提供者)。我们观察下代码`discoveryManagerScrape.ApplyConfig`:

```
func (m *Manager) ApplyConfig(cfg map[string]sd_config.ServiceDiscoveryConfig) error {
	......
	// 为每个scrape_config都建立一个provider并注册
	for name, scfg := range cfg {
		failedCount += m.registerProviders(scfg, name)
		discoveredTargets.WithLabelValues(m.name, name).Set(0)
	}	
	// 启动所有的providers，也就启动了不停的抓取过程
	for _, prov := range m.providers {
		m.startProvider(m.ctx, prov)
	}
}
```
其中注册过程如下，根据不同的scrapeConfigs类型生成不同的provider,然后放到一个scrapeManager的成员providers切片里面。

在我们解析出所有的provider之后，就可以启动provider来进行不停抓取所有Target的过程了。注意这里只是监控目标的抓取，并非抓取真正的数据。

```
func (m *Manager) startProvider(ctx context.Context, p *provider) {
	......
	// 抓取最新Targets的goroutine
	go p.d.Run(ctx, updates)
	// 将Prometheus现有Targets集合替换为新Targets(刚抓取到的)的携程
	go m.updater(ctx, p, updates)
}
```
Prometheus对每个provider分了两个goroutine来进行更新Targets(监控对象)集合的操作,第一个是不同的获取最新Targets的goroutine。在抓取到之后，会由updater goroutine来热更新当前抓取目标。更新完目标targets后，再触发triggerSend chan。 
这样做，虽然将逻辑变复杂了，但很好的将scrape的过程与更新target的过程隔离了开来。

## 防止更新过于频繁
为了防止更新过于频繁，Prometheus通过一个ticker去限制更新频率`manager.go`。

```
func (m *Manager) sender(){
	ticker := time.NewTicker(m.updatert)
	defer ticker.Stop()
	......
	for {
	.....
		select {
			......
			case <-ticker.C:
				......
				case <-m.triggerSend
				......
				case m.synch <- m.allGroups
				......
		}
	}
}
```

如代码中所见，只有在ticker每秒触发后才会去检测triggerSend信号，这样就不会频繁的触发重建scrape targets逻辑了。代码中，我们最终将allGroups,也就是m.targets(刚刚更新过的)的所有数据全部取出送入到m.synch 
通过m.synch触发更新manger.targetSets,最后触发reloader信号,如下代码所示:

```
scrape/manager.go

func (m *Manager) Run(tsets <-chan map[string][]*targetgroup.Group) error {
	go m.reloader()
	for {
		select {
		case ts := <-tsets:
			m.updateTsets(ts)

			select {
			case m.triggerReload <- struct{}{}:
			default:
			}

		case <-m.graceShut:
			return nil
		}
	}
}
```

## reloader操作
具体的初始化动作在reload()中，也就是真正改变Prometheus抓取目标动作的地方。 首先，我们对每一个provider(及其底下的subs)建立一个scrapePool，并对每个target建立一个goroutine去exporter定时抓取数据。

```
// scrape.manager.go
//遍历每个targetGroup：创建scrapePool，然后对scrapePool进行Sync
func (m *Manager) reload() {
    m.mtxScrape.Lock()
    var wg sync.WaitGroup
    for setName, groups := range m.targetSets {
        if _, ok := m.scrapePools[setName]; !ok {
            scrapeConfig, ok := m.scrapeConfigs[setName]
            ...
            //创建scrapePool
            sp, err := newScrapePool(scrapeConfig, m.append, m.jitterSeed, log.With(m.logger, "scrape_pool", setName))
            m.scrapePools[setName] = sp
        }

        wg.Add(1)
        // Run the sync in parallel as these take a while and at high load can't catch up.
        go func(sp *scrapePool, groups []*targetgroup.Group) {
            sp.Sync(groups)        //scrapePool内的Sync，这里是groups是数组，但是一般只有1个元素
            wg.Done()
        }(m.scrapePools[setName], groups)
    }
    m.mtxScrape.Unlock()
    wg.Wait()
}
```

对于一个targetGroup下的所有target对象，它们共享httpclient和bufferPool
```
// scrape/scrape.go
// 创建scrapePool
func newScrapePool(cfg *config.ScrapeConfig, app storage.Appendable, jitterSeed uint64, logger log.Logger) (*scrapePool, error) {
    client, err := config_util.NewClientFromConfig(cfg.HTTPClientConfig, cfg.JobName, false)
    // bufferPool
    buffers := pool.New(1e3, 100e6, 3, func(sz int) interface{} { return make([]byte, 0, sz) })

    ctx, cancel := context.WithCancel(context.Background())
    sp := &scrapePool{
        cancel:        cancel,
        appendable:    app,
        config:        cfg,
        client:        client,
        activeTargets: map[uint64]*Target{},
        loops:         map[uint64]loop{},
        logger:        logger,
    }
    sp.newLoop = func(opts scrapeLoopOptions) loop {
        ......
        return newScrapeLoop(
            ctx,
            opts.scraper,
            log.With(logger, "target", opts.target),
            buffers,
            func(l labels.Labels) labels.Labels {
                return mutateSampleLabels(l, opts.target, opts.honorLabels, opts.mrc)
            },
            func(l labels.Labels) labels.Labels { return mutateReportSampleLabels(l, opts.target) },
            func() storage.Appender { return appender(app.Appender(), opts.limit) },
            cache,
            jitterSeed,
            opts.honorTimestamps,
        )
    }
    return sp, nil
}
```

scrape.scrapePool的代码逻辑
scrapePool为targetGroup下的每个targets，创建1个scrapeLoop，然后让scrapeLoop干活
```
// scrape/scrape.go
func (sp *scrapePool) Sync(tgs []*targetgroup.Group) {
        //所有的targets
    var all []*Target
    sp.mtx.Lock()
    sp.droppedTargets = []*Target{}
    for _, tg := range tgs {    
        targets, err := targetsFromGroup(tg, sp.config)
        ......
        for _, t := range targets {
            if t.Labels().Len() > 0 {
                all = append(all, t)
            }
            ......
        }
    }
    sp.mtx.Unlock()
    //指挥target干活
    sp.sync(all)
    ......
}
```
对每个target，使用newLoop()创建targetLoop，然后启动1个goroutine，让targetLoop.run()循环拉取：
```
// scrape/scrape.go
//遍历Group下的每个target，对每个target: 创建targetScraper，创建scrapeLoop，然后scrapeLoop进行干活
func (sp *scrapePool) sync(targets []*Target) {
    for _, t := range targets {
        t := t
        hash := t.hash()
        uniqueTargets[hash] = struct{}{}

        if _, ok := sp.activeTargets[hash]; !ok {
            s := &targetScraper{Target: t, client: sp.client, timeout: timeout}    //创建targetScraper
            l := sp.newLoop(scrapeLoopOptions{    //创建scrapeLoop
                target:          t,
                scraper:         s,
                limit:           limit,
                honorLabels:     honorLabels,
                honorTimestamps: honorTimestamps,
                mrc:             mrc,
            })

            sp.activeTargets[hash] = t
            sp.loops[hash] = l

            go l.run(interval, timeout, nil)  //scrapeLoop循环拉取
        }
        ......
    }
    ...
    wg.Wait()
}
```

## scrapeLoop的代码逻辑
每个scrapeLoop按抓取周期循环执行。我们通过scrapeLoop去定时从不同的Target获取数据并写入到TSDB(时序数据库里面)。
- scrape抓取指标数据；
- append写入底层存储；
- 最后更新scrapeLoop的状态(主要是指标值)；

```
// scrape/scrape.go
func (sl *scrapeLoop) run(interval, timeout time.Duration, errc chan<- error) {
    ......
    ticker := time.NewTicker(interval)    //定时器，定时执行
    defer ticker.Stop()

    for {
        ......
        var (
            start             = time.Now()
            scrapeCtx, cancel = context.WithTimeout(sl.ctx, timeout)
        )
        ......
        contentType, scrapeErr := sl.scraper.scrape(scrapeCtx, buf)    //scrape进行抓取
        ......

        //写scrape的数据写入底层存储
        total, added, seriesAdded, appErr := sl.append(b, contentType, start)
        ......

        sl.buffers.Put(b)    //写入buffer
        //更新scrapeLoop的状态
        if err := sl.report(start, time.Since(start), total, added, seriesAdded, scrapeErr); err != nil {
            level.Warn(sl.l).Log("msg", "Appending scrape report failed", "err", err)
        }

        select {
        ......
        case <-ticker.C:       //循环执行
        }
    }
    ......
}
```

## targetScrape的抓取逻辑
最终达到HTTP抓取的逻辑：

首先，向target的url发送HTTP Get；然后，将写入io.writer(即上文中的buffers)中，待后面解析出指标：

```
//抓取逻辑
func (s *targetScraper) scrape(ctx context.Context, w io.Writer) (string, error) {
    if s.req == nil {
        req, err := http.NewRequest("GET", s.URL().String(), nil)    //Get /metrics接口
        if err != nil {
            return "", err
        }
        req.Header.Add("Accept", acceptHeader)
        req.Header.Add("Accept-Encoding", "gzip")
        req.Header.Set("User-Agent", userAgentHeader)
        req.Header.Set("X-Prometheus-Scrape-Timeout-Seconds", fmt.Sprintf("%f", s.timeout.Seconds()))

        s.req = req
    }

    resp, err := s.client.Do(s.req.WithContext(ctx))    //发送http GET请求
    if err != nil {
        return "", err
    }
    .......

    if resp.Header.Get("Content-Encoding") != "gzip" {
        _, err = io.Copy(w, resp.Body)      //将response body写入参数w
        if err != nil {
            return "", err
        }
        return resp.Header.Get("Content-Type"), nil
    }
    if s.gzipr == nil {
        s.buf = bufio.NewReader(resp.Body)
        s.gzipr, err = gzip.NewReader(s.buf)
        if err != nil {
            return "", err
        }
    }
    // 写入io.writer
    _, err = io.Copy(w, s.gzipr)
    s.gzipr.Close()
    return resp.Header.Get("Content-Type"), nil
}
```


解析http Response并插入TSDB的函数为:

```
scrape.go

func (sl *scrapeLoop) append(b []byte, contentType string, ts time.Time) {
......
loop:
	for {
		var et textparse.Entry
		// 解析parse
		......
		// 解析后的数据写入TSDB
		app.AddFast(ce.lset, ce.ref, t, v);
	}
	......
	// 没有错误的话，再提交
	if err := app.Commit(); err != nil {
		return total, added, seriesAdded, err
	}
	......
}
```

通过TSDB事务的概念，Prometheus使得我们的数据写入是原子的，即不会出现只看到部分数据的现象。

## 数据查询
Prometheus提供了强大的Promql来满足我们千变万化的查询需求。

## Promql

一个Promql表达式可以计算为下面四种类型:

```
瞬时向量(Instant Vector) - 一组同样时间戳的时间序列(取自不同的时间序列，例如不同机器同一时间的CPU idle)
区间向量(Range vector) - 一组在一段时间范围内的时间序列
标量(Scalar) - 一个浮点型的数据值
字符串(String) - 一个简单的字符串
```

我们还可以在Promql中使用svm/avg等集合表达式，不过只能用在瞬时向量(Instant Vector)上面。为了阐述Prometheus的聚合计算以及篇幅原因，笔者在本篇文章只详细分析瞬时向量(Instant Vector)的执行过程。

## 瞬时向量(Instant Vector)

前面说到，瞬时向量是一组拥有同样时间戳的时间序列。但是实际过程中，我们对不同Endpoint采样的时间是不可能精确一致的。所以，Prometheus采取了距离指定时间戳之前最近的数据(Sample)。
当然，如果是距离当前时间戳1个小时的数据直观看来肯定不能纳入到我们的返回结果里面。 所以Prometheus通过一个指定的时间窗口来过滤数据(通过启动参数--query.lookback-delta指定，默认5min)。



## 对一条简单的Promql进行分析

好了，解释完Instant Vector概念之后，我们可以着手进行分析了。直接上一条带有聚合函数的Promql把。

```
SUM BY (group) (http_requests{job="api-server",group="production"})
```

首先,对于这种有语法结构的语句肯定是将其Parse一把，构造成AST树了。调用

```
promql.ParseExpr
```

由于Promql较为简单，所以Prometheus直接采用了LL语法分析。在这里直接给出上述Promql的AST树结构。 
Prometheus对于语法树的遍历过程都是通过vistor模式,具体到代码为:

```
ast.go vistor设计模式
func Walk(v Visitor, node Node, path []Node) error {
	var err error
	if v, err = v.Visit(node, path); v == nil || err != nil {
		return err
	}
	path = append(path, node)

	for _, e := range Children(node) {
		if err := Walk(v, e, path); err != nil {
			return err
		}
	}

	_, err = v.Visit(nil, nil)
	return err
}
func (f inspector) Visit(node Node, path []Node) (Visitor, error) {
	if err := f(node, path); err != nil {
		return nil, err
	}

	return f, nil
}
```

通过golang里非常方便的函数式功能，直接传递求值函数inspector进行不同情况下的求值。

```
type inspector func(Node, []Node) error
```



## 求值过程

具体的求值过程核心函数为:

```
func (ng *Engine) execEvalStmt(ctx context.Context, query *query, s *EvalStmt) (Value, storage.Warnings, error) {
	......
	querier, warnings, err := ng.populateSeries(ctxPrepare, query.queryable, s) 	// 这边拿到对应序列的数据
	......
	val, err := evaluator.Eval(s.Expr) // here 聚合计算
	......

}
```



### populateSeries

首先通过populateSeries的计算出VectorSelector Node所对应的series(时间序列)。这里直接给出求值函数

```
 func(node Node, path []Node) error {
 	......
 	querier, err := q.Querier(ctx, timestamp.FromTime(mint), timestamp.FromTime(s.End))
 	......
 	case *VectorSelector:
 		.......
 		set, wrn, err = querier.Select(params, n.LabelMatchers...)
 		......
 		n.unexpandedSeriesSet = set
 	......
 	case *MatrixSelector:
 		......
 }
 return nil
```

可以看到这个求值函数，只对VectorSelector/MatrixSelector进行操作，针对我们的Promql也就是只对叶子节点VectorSelector有效。 



#### select

获取对应数据的核心函数就在querier.Select。我们先来看下qurier是如何得到的.

```
querier, err := q.Querier(ctx, timestamp.FromTime(mint), timestamp.FromTime(s.End))
```

根据时间戳范围去生成querier,里面最重要的就是计算出哪些block在这个时间范围内，并将他们附着到querier里面。具体见函数

```
func (db *DB) Querier(mint, maxt int64) (Querier, error) {
	for _, b := range db.blocks {
		......
		// 遍历blocks挑选block
	}
	// 如果maxt>head.mint(即内存中的block),那么也加入到里面querier里面。
	if maxt >= db.head.MinTime() {
		blocks = append(blocks, &rangeHead{
			head: db.head,
			mint: mint,
			maxt: maxt,
		})
	}
	......
}
```

知道数据在哪些block里面，我们就可以着手进行计算VectorSelector的数据了。

```
 // labelMatchers {job:api-server} {__name__:http_requests} {group:production}
 querier.Select(params, n.LabelMatchers...)
```

有了matchers我们很容易的就能够通过倒排索引取到对应的series。为了篇幅起见，我们假设数据都在headBlock(也就是内存里面)。那么我们对于倒排的计算就如下图所示: 
这样，我们的VectorSelector节点就已经有了最终的数据存储地址信息了，例如图中的memSeries refId=3和4。

```
<<Prometheus时序数据库-磁盘中的存储结构>>
```



### evaluator.Eval

通过populateSeries找到对应的数据，那么我们就可以通过evaluator.Eval获取最终的结果了。计算采用后序遍历，等下层节点返回数据后才开始上层节点的计算。那么很自然的，我们先计算VectorSelector。

```
func (ev *evaluator) eval(expr Expr) Value {
	......
	case *VectorSelector:
	// 通过refId拿到对应的Series
	checkForSeriesSetExpansion(ev.ctx, e)
	// 遍历所有的series
	for i, s := range e.series {
		// 由于我们这边考虑的是instant query,所以只循环一次
		for ts := ev.startTimestamp; ts <= ev.endTimestamp; ts += ev.interval {
			// 获取距离ts最近且小于ts的最近的sample
			_, v, ok := ev.vectorSelectorSingle(it, e, ts)
			if ok {
					if ev.currentSamples < ev.maxSamples {
						// 注意，这边的v对应的原始t被替换成了ts,也就是instant query timeStamp
						ss.Points = append(ss.Points, Point{V: v, T: ts})
						ev.currentSamples++
					} else {
						ev.error(ErrTooManySamples(env))
					}
				}
			......
		}
	}
}
```

如代码注释中看到，当我们找到一个距离ts最近切小于ts的sample时候，只用这个sample的value,其时间戳则用ts(Instant Query指定的时间戳)代替。

其中vectorSelectorSingle值得我们观察一下:

```
func (ev *evaluator) vectorSelectorSingle(it *storage.BufferedSeriesIterator, node *VectorSelector, ts int64) (int64, float64, bool){
	......
	// 这一步是获取>=refTime的数据，也就是我们instant query传入的
	ok := it.Seek(refTime)
	......
		if !ok || t > refTime { 
		// 由于我们需要的是<=refTime的数据，所以这边回退一格，由于同一memSeries同一时间的数据只有一条，所以回退的数据肯定是<=refTime的
		t, v, ok = it.PeekBack(1)
		if !ok || t < refTime-durationMilliseconds(LookbackDelta) {
			return 0, 0, false
		}
	}
}
```

就这样，我们找到了series 3和4距离Instant Query时间最近且小于这个时间的两条记录，并保留了记录的标签。这样，我们就可以在上层进行聚合。 



## SUM by聚合

叶子节点VectorSelector得到了对应的数据后，我们就可以对上层节点AggregateExpr进行聚合计算了。代码栈为:

```
evaluator.rangeEval
	|->evaluate.eval.func2
		|->evelator.aggregation grouping key为group
```

具体的函数如下图所示:

```
func (ev *evaluator) aggregation(op ItemType, grouping []string, without bool, param interface{}, vec Vector, enh *EvalNodeHelper) Vector {
	......
	// 对所有的sample
	for _, s := range vec {
		metric := s.Metric
		......
		group, ok := result[groupingKey] 
		// 如果此group不存在，则新加一个group
		if !ok {
			......
			result[groupingKey] = &groupedAggregation{
				labels:     m, // 在这里我们的m=[group:production]
				value:      s.V,
				mean:       s.V,
				groupCount: 1,
			}
			......
		}
		switch op {
		// 这边就是对SUM的最终处理
		case SUM:
			group.value += s.V
		.....
		}
	}
	.....
	for _, aggr := range result {
		enh.out = append(enh.out, Sample{
		Metric: aggr.labels,
		Point:  Point{V: aggr.value},
		})
	}
	......
	return enh.out
}
```
















