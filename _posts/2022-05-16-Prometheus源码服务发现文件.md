---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus源码服务发现文件


## 服务发现文件
相对于静态配置方式，基于文件发现机制是一种更加常用的方式，通过监听JSON和YAML格式文件变更事件进行动态target加载，文件格式如下：
```
JSON json [ { "targets": [ "<host>", ... ], "labels": { "<labelname>": "<labelvalue>", ... } }, ... ]

YAML yaml - targets: [ - '<host>' ] labels: [ <labelname>: <labelvalue> ... ]
```
虽然，文件发现服务是基于文件事件监听实时感知的，但是作为补偿，Prometheus提供了定时周期刷新文件机制：
```yaml
- job_name: 'file_ds'
  file_sd_configs:
    - files:
      - d:/prometheus-data/targets/f1.json #支持文件后缀 .json、.yml和.yaml
      - d:/prometheus-data/targets/f2.json
      refresh_interval: 1m #自动发现间隔时间，默认5m
```
另外，每个target在relabel阶段会添加一个__meta_filepath标签，值就是提取target对应的文件路径。

files中配置的文件路径最后一级支持通配符*，比如`my/path/tg_*.json`。

## 协议分析
主要包括三部分之间协调联动完成主要功能，协议部分主要是在【配置处理】部分，通过配置中定义的服务发现类型会被解析成不同协议对应的SDConfig实现，然后创建Discovery，其内部封装了不同服务协议具体实现内容，最后调用Discovery.Run()启动服务开始进行服务发现。

### 基于文件服务发现配置解析

假如我们定义如下job:
```yaml
- job_name: 'file_ds'
  file_sd_configs:
    - files:
        - d:/prometheus-data/targets/f1.json
        - d:/prometheus-data/targets/f2.json
      refresh_interval: 30s
```
会被解析成file.SDConfig如下：

file.SDConfig定义如下：
```
type SDConfig struct {
Files           []string       `yaml:"files"`
RefreshInterval model.Duration `yaml:"refresh_interval,omitempty"`
}
```

该结构体定义比较简单：文件路径和周期刷新间隔。

### Discovery创建
```
func NewDiscovery(conf *SDConfig, logger log.Logger) *Discovery {
 if logger == nil {
  logger = log.NewNopLogger()
 }

 disc := &Discovery{
  paths:      conf.Files,
  interval:   time.Duration(conf.RefreshInterval),
  timestamps: make(map[string]float64),
  logger:     logger,
 }
 fileSDTimeStamp.addDiscoverer(disc)
 return disc
}
```

### Discovery创建完成，最后会调用Discovery.Run()启动服务发现：
```
func (d *Discovery) Run(ctx context.Context, ch chan<- []*targetgroup.Group) {
    //fsnotify 能监控指定文件夹内文件的修改情况，如 文件的增加、删除、修改、重命名等操作。
 watcher, err := fsnotify.NewWatcher()
 if err != nil {
  level.Error(d.logger).Log("msg", "Error adding file watcher", "err", err)
  return
 }
 d.watcher = watcher
 defer d.stop()

 d.refresh(ctx, ch)

 ticker := time.NewTicker(d.interval)
 defer ticker.Stop()

 for {
  select {
  case <-ctx.Done():
   return

  case event := <-d.watcher.Events:
   if len(event.Name) == 0 {
    break
   }
   if event.Op^fsnotify.Chmod == 0 {
    break
   }
   d.refresh(ctx, ch)

  case <-ticker.C:
   d.refresh(ctx, ch)

  case err := <-d.watcher.Errors:
   if err != nil {
    level.Error(d.logger).Log("msg", "Error watching file", "err", err)
   }
  }
 }
}
```
存在三种情况下会去调用d.refresh(ctx, ch)逻辑：
- Run方法调用时，会先调用一次d.refresh(ctx, ch)，然后进入无限for循环部分；
- case event := <-d.watcher.Events：监控到目录文件出现变更时；
- case <-ticker.C：定时周期执行调用，周期时间通过refresh_interval设置，默认5分钟；

上面分析了几种情况下都会去调用d.refresh(ctx, ch)逻辑，基于文件发现协议主要实现逻辑就在这里：
```
func (d *Discovery) refresh(ctx context.Context, ch chan<- []*targetgroup.Group) {
 t0 := time.Now()
 defer func() {
  fileSDScanDuration.Observe(time.Since(t0).Seconds())
 }()
 ref := map[string]int{}
 // listFiles()根据file_sd_configs配置中files配置查找出所有符合的文件
 for _, p := range d.listFiles() {
  // readFile()读取文件内容，解析出[]*targetgroup.Group
  tgroups, err := d.readFile(p)
  if err != nil {//如果出错，跳过当前文件继续处理下一个文件
   fileSDReadErrorsCount.Inc()

   level.Error(d.logger).Log("msg", "Error reading file", "path", p, "err", err)
   // Prevent deletion down below.
   ref[p] = d.lastRefresh[p]
   continue
  }
  select {
  // 将从文件中解析的[]*targetgroup.Group通过channel发送出去
  case ch <- tgroups:
  case <-ctx.Done():
   return
  }

  ref[p] = len(tgroups)
 }
 // Send empty updates for sources that disappeared.
 // d.lastRefresh记录上次刷新的，ref记录当次刷新的，p是文件名
 // 上次刷新文件存在，但是当前刷新该文件没有，则该文件被删除，则发送空target集合过去覆盖掉之前的target数据
 for f, n := range d.lastRefresh {
  m, ok := ref[f]
  if !ok || n > m {//文件被删，则发送空target
   level.Debug(d.logger).Log("msg", "file_sd refresh found file that should be removed", "file", f)
   d.deleteTimestamp(f)
   for i := m; i < n; i++ {
    select {
    // 发送[]*targetgroup.Group，但是targets是空，覆盖掉之前
    case ch <- []*targetgroup.Group{{Source: fileSource(f, i)}}:
    case <-ctx.Done():
     return
    }
   }
  }
 }
 d.lastRefresh = ref

 // 添加文件变更监听
 d.watchFiles()
}
```

基于文件方式的服务发现总结

- Prometheus配置中服务发现为file_sd_configs的job会被解析成file.SDConfig；
- 然后创建出Discovery，Discovery可以看成用于服务发现target核心组件；
- 然后调用Discovery.Run方法开始启动，这时Discovery组件就开始去发现target信息；
- 真正干活部分都被封装到refresh方法中，Discovery.Run会在三种场景下调用该方法：
  - Discovery.Run刚启动时；
  - file_sd_configs中files文件出现变更；
  - 定时周期刷新，补偿机制，刷新周期通过refresh_interval参数控制，默认5分钟；
- refresh方法中将发现的最新targets信息通过channel通道传递出去，最后一层层传递到scrape组件，具体参加上节分析流程。