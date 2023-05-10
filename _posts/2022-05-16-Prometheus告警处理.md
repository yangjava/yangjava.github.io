---
layout: post
categories: Prometheus
description: none
keywords: Prometheus
---
# Prometheus告警处理
Prometheus只负责进行报警计算，而具体的报警触发则由AlertManager完成。

## 简单的报警规则
```yaml
rules:
	alert: HTTPRequestRateLow
	expr: http_requests < 100
	for: 60s
	labels:
		severity: warning
	annotations:
		description: "http request rate low"
	
```
这上面的规则即是http请求数量<100从持续1min,则我们开始报警，报警级别为warning

## 监控采集
```
# 数据采集间隔
scrape_interval: 15s 

# 评估告警周期
evaluation_interval: 15s 

# 数据采集超时时间默认10s
scrape_timeout: 10s
```

Prometheus以scrape_interval（默认为1m）规则周期，从监控目标上收集信息。其中scrape_interval可以基于全局或基于单个metric定义；然后将监控信息持久存储在其本地存储上。

Prometheus以evaluation_interval（默认为1m）另一个独立的规则周期，对告警规则做定期计算。其中evaluation_interval只有全局值；然后更新告警状态。

其中包含三种告警状态：
- inactive：没有触发阈值 
- pending：已触发阈值但未满足告警持续时间
- firing：已触发阈值且满足告警持续时间

举一个例子，阈值告警的配置如下：
```
groups:
  - name:example
    rules:
      - alert:mysql_uptime
      expr: mysql_server_status_uptime<30
      for:10s
      labels:
        level:"CRITICAL"
      annotations:
        detail:数据库运行时间
```
- 收集到的mysql_uptime>=30,告警状态为inactive
- 收集到的mysql_uptime<30,且持续时间小于10s，告警状态为pending
- 收集到的mysql_uptime<30,且持续时间大于10s，告警状态为firing
**注意**：配置中的for语法就是用来设置告警持续时间的；如果配置中不设置for或者设置为0，那么pending状态会被直接跳过。

那么怎么来计算告警阈值持续时间呢，需要回到上文的scrape_interval和evaluation_interval，假设scrape_interval为5s采集一次信息；evaluation_interval为10s；mysql_uptime告警阈值需要满足10s持续时间。
- Prometheus以5s（scrape_interval）一个采集周期采集状态；
- 然后根据采集到状态按照10s（evaluation_interval）一个计算周期，计算表达式；
- 表达式为真，告警状态切换到pending；
- 下个计算周期，表达式仍为真，且符合for持续10s，告警状态变更为active，并将告警从Prometheus发送给Altermanger；
- 下个计算周期，表达式仍为真，且符合for持续10s，持续告警给Altermanger；
- 直到某个计算周期，表达式为假，告警状态变更为inactive，发送一个resolve给Altermanger，说明此告警已解决。

## 告警分组、抑制、静默
Altermanger并不是把一条从Prometheus接收到的告警简简单单的直接发送出去；直接发送出去会导致告警信息过多，运维人员会被告警淹没；所以Altermanger需要对告警做合理的收敛。

Altermanger非常重要的告警收敛手段。
- 分组：group
- 抑制：inhibitor
- 静默：silencer

### 告警分组
告警分组的作用
- 同类告警的聚合帮助运维排查问题
- 通过告警邮件的合并，减少告警数量

举例来说：我们按照mysql的实例id对告警分组；如下图所示，告警信息会被拆分成两组。
```
mysql-A
mysql_cpu_high
mysql-B
mysql_uptime
mysql_slave_sql_thread_down
mysql_slave_io_thread_down
```
实例A分组下的告警会合并成一个告警邮件发送；
实例B分组下的告警会合并成一个告警邮件发送；

通过分组合并，能帮助运维降低告警数量，同时能有效聚合告警信息，帮助问题分析。

### 告警抑制
告警抑制的作用
- 消除冗余的告警

举例来说：同一台server-A的告警，如果有如下两条告警，并且配置了抑制规则。
- mysql_uptime
- server_uptime

最后只会收到一条server_uptime的告警。

A机器挂了，势必导致A服务器上的mysql也挂了；如配置了抑制规则，通过服务器down来抑制这台服务器上的其他告警；这样就能消除冗余的告警，帮助运维第一时间掌握最核心的告警信息。

### 告警静默
告警静默的作用
- 阻止发送可预期的告警

举例来说：夜间跑批时间，批量任务会导致实例A压力升高；我们配置了对实例A的静默规则。
- mysql-A
```
  qps_more_than_3000
  tps_more_than_2000
  thread_running_over_200
```

- mysql-B
```
  thread_running_over_200
```
最后我们只会收到一条实例B的告警。A压力高是可预期的，周期性的告警会影响运维判断；这种场景下，运维需要聚焦处理实例B的问题即可。

## 告警延时
分组的概念，分组势必会带来延时；合理的配置延时，才能避免告警不及时的问题，同时帮助我们避免告警轰炸的问题。

我们先来看告警延时的几个重要参数：
- group_by:分组参数，第二节已经介绍，比如按照`[mysql-id]`分组
- group_wait:分组等待时间，比如：5s
- group_interval:分组尝试再次发送告警的时间间隔，比如：5m
- Repeat_interval:分组内发送相同告警的时间间隔，比如：60m

## 什么时候触发这个计算
在加载完规则之后，Prometheus按照evaluation_interval这个全局配置去不停的计算Rules。代码逻辑如下所示:

```
rules/manager.go

func (g *Group) run(ctx context.Context) {
	iter := func() {
		......
		g.Eval(ctx,evalTimestamp)
		......
	}
	// g.interval = evaluation_interval
	tick := time.NewTicker(g.interval)
	defer tick.Stop()
	......
	for {
		......
		case <-tick.C:
			......
			iter()
	}
}
```

而g.Eval的调用为:

```
func (g *Group) Eval(ctx context.Context, ts time.Time) {
	// 对所有的rule
	for i, rule := range g.rules {
		......
		// 先计算出是否有符合rule的数据
		vector, err := rule.Eval(ctx, ts, g.opts.QueryFunc, g.opts.ExternalURL)
		......
		// 然后发送
		ar.sendAlerts(ctx, ts, g.opts.ResendDelay, g.interval, g.opts.NotifyFunc)
	}
	......
}
```

整个过程如下图所示: 



## 对单个rule的计算

我们可以看到，最重要的就是rule.Eval这个函数。代码如下所示:

```
func (r *AlertingRule) Eval(ctx context.Context, ts time.Time, query QueryFunc, externalURL *url.URL) (promql.Vector, error) {
	// 最终调用了NewInstantQuery
	res, err = query(ctx,r.vector.String(),ts)
	......
	// 报警组装逻辑
	......
	// active 报警状态变迁
}
```

这个Eval包含了报警的计算/组装/发送的所有逻辑。我们先聚焦于最重要的计算逻辑。也就是其中的query。其实，这个query是对NewInstantQuery的一个简单封装。

```
func EngineQueryFunc(engine *promql.Engine, q storage.Queryable) QueryFunc {
	return func(ctx context.Context, qs string, t time.Time) (promql.Vector, error) {
		q, err := engine.NewInstantQuery(q, qs, t)
		......
		res := q.Exec(ctx)
	}
}
```

也就是说它执行了一个瞬时向量的查询。而其查询的表达式按照我们之前给出的报警规则，即是

```
http_requests < 100 
```

既然要计算表达式，那么第一步，肯定是将其构造成一颗AST。其树形结构如下图所示: 
解析出左节点是个VectorSelect而且知道了其lablelMatcher是

```
__name__:http_requests
```

那么我们就可以左节点VectorSelector进行求值。直接利用倒排索引在head中查询即可(因为instant query的是当前时间，所以肯定在内存中)。 
想知道具体的计算流程，可以见笔者之前的博客《Prometheus时序数据库-数据的查询》 计算出左节点的数据之后，我们就可以和右节点进行比较以计算出最终结果了。具体代码为:

```
func (ev *evaluator) eval(expr Expr) Value {
	......
	case *BinaryExpr:
	......
		case lt == ValueTypeVector && rt == ValueTypeScalar:
			return ev.rangeEval(func(v []Value, enh *EvalNodeHelper) Vector {
				return ev.VectorscalarBinop(e.Op, v[0].(Vector), Scalar{V: v[1].(Vector)[0].Point.V}, false, e.ReturnBool, enh)
			}, e.LHS, e.RHS)
	.......
}
```

最后调用的函数即为:

```
func (ev *evaluator) VectorBinop(op ItemType, lhs, rhs Vector, matching *VectorMatching, returnBool bool, enh *EvalNodeHelper) Vector {
	// 对左节点计算出来的所有的数据sample
	for _, lhsSample := range lhs {
		......
		// 由于左边lv = 75 < 右边rv = 100，且op为less
		/**
			vectorElemBinop(){
				case LESS
					return lhs, lhs < rhs
			}
		**/
		// 这边得到的结果value=75,keep = true
		value, keep := vectorElemBinop(op, lv, rv)
		......
		if keep {
			......
			// 这边就讲75放到了输出里面，也就是说我们最后的计算确实得到了数据。
			enh.out = append(enh.out.sample)
		}
	}
}
```

最后我们的expr输出即为

```
sample {
	Point {t:0,V:75}
	Metric {__name__:http_requests,instance:0,job:api-server}
		
}
```



## 报警状态变迁

计算过程讲完了，笔者还稍微讲一下报警的状态变迁，也就是最开始报警规则中的rule中的for,也即报警持续for(规则中为1min)，我们才真正报警。为了实现这种功能，这就需要一个状态机了。笔者这里只阐述下从Pending(报警出现)->firing(真正发送)的逻辑。

在之前的Eval方法里面，有下面这段

```
func (r *AlertingRule) Eval(ctx context.Context, ts time.Time, query QueryFunc, externalURL *url.URL) (promql.Vector, error) {
	for _, smpl := range res {
	......
			if alert, ok := r.active[h]; ok && alert.State != StateInactive {
			alert.Value = smpl.V
			alert.Annotations = annotations
			continue
		}
		// 如果这个告警不在active map里面，则将其放入
		// 注意，这里的hash依旧没有拉链法，有极小概率hash冲突
r.active[h] = &Alert{
			Labels:      lbs,
			Annotations: annotations,
			ActiveAt:    ts,
			State:       StatePending,
			Value:       smpl.V,
		}
	}
	......
	// 报警状态的变迁逻辑
	for fp, a := range r.active {
		// 如果当前r.active的告警已经不在刚刚计算的result里面了		if _, ok := resultFPs[fp]; !ok {
			// 如果状态是Pending待发送
			if a.State == StatePending || (!a.ResolvedAt.IsZero() && ts.Sub(a.ResolvedAt) > resolvedRetention) {
				delete(r.active, fp)
			}
			......
			continue
		}
		// 对于已有的Active报警，如果其Active的时间>r.holdDuration，也就是for指定的
		if a.State == StatePending && ts.Sub(a.ActiveAt) >= r.holdDuration {
			// 我们将报警置为需要发送
			a.State = StateFiring
			a.FiredAt = ts
		}
		......
	
	}
}
```
## 总结

Prometheus作为一个监控神器，给我们提供了各种各样的遍历。其强大的报警计算功能就是其中之一。了解其中告警的计算原理，才能让我们更好的运用它。