---
layout: post
categories: [Prometheus]
description: none
keywords: Prometheus
---
# Prometheus指标监控
Prometheus是一个开放性的监控解决方案，用户可以非常方便的安装和使用Prometheus并且能够非常方便的对其进行扩展

## 我们需要什么指标
对于DDD、TDD等，大家比较熟悉了，但是对于MDD可能就比较陌生了。MDD是Metrics-Driven Development的缩写，主张开发过程由指标驱动，通过实用指标来驱动快速、精确和细粒度的软件迭代。MDD可使所有可以测量的东西都得到量化和优化，进而为整个开发过程带来可见性，帮助相关人员快速、准确地作出决策，并在发生错误时立即发现问题并修复。依照MDD的理念，在需求阶段就应该考虑关键指标，在应用上线后通过指标了解现状并持续优化。

有一些基于指标的方法论，建议大家了解一下：

Google的四大黄金指标：延迟Latency、流量Traffic、错误Errors、饱和度Saturation
Netflix的USE方法：使用率Utilization、饱和度Saturation、错误Error
WeaveCloud的RED方法：速率Rate、错误Errors、耗时Duration

## 在SrpingBoot中引入prometheus

### springboot 引入 prometheus相关jar包

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-core</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>
```

## 在application.yaml中将prometheus的endpoint放出来。

```yaml
management:
  endpoints:
    web:
      exposure:
        include: info,health,prometheus
```

application.prometheus文件配置
```properties
spring.application.name=actuator-prometheus
server.port=10001

# 管理端点的跟路径，默认就是/actuator
management.endpoints.web.base-path=/actuator
# 管理端点的端口
management.server.port=10002
# 暴露出 prometheus 端口
management.endpoints.web.exposure.include=prometheus
# 启用 prometheus 端口，默认就是true
management.metrics.export.prometheus.enabled=true

# 增加每个指标的全局的tag，及给每个指标一个 application的 tag,值是 spring.application.name的值
management.metrics.tags.application=${spring.application.name}
```

## 指标项的添加

添加指标API
```java
Metrics.counter("", tags).increment();
```
SpringBoot2.0的metrics支持多tag

web通过可以http://localhost:8080/actuator/prometheus访问

```text
# HELP jvm_threads_live_threads The current number of live threads including both daemon and non-daemon threads
# TYPE jvm_threads_live_threads gauge
jvm_threads_live_threads 21.0
# HELP process_uptime_seconds The uptime of the Java virtual machine
# TYPE process_uptime_seconds gauge
process_uptime_seconds 1947.48
# HELP process_start_time_seconds Start time of the process since unix epoch.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1.673243010006E9
# HELP executor_queue_remaining_tasks The number of additional elements that this queue can ideally accept without blocking
# TYPE executor_queue_remaining_tasks gauge
executor_queue_remaining_tasks{name="applicationTaskExecutor",} 2.147483647E9
# HELP tomcat_sessions_rejected_sessions_total  
# TYPE tomcat_sessions_rejected_sessions_total counter
tomcat_sessions_rejected_sessions_total 0.0
......
```

## 指标项详解

### 进程
- process_uptime_seconds 表示进程运行时间
```text
# HELP process_uptime_seconds The uptime of the Java virtual machine
# TYPE process_uptime_seconds gauge
process_uptime_seconds 35.901
```
-  process_start_time_seconds 表示进程启动时刻
```text
# HELP process_start_time_seconds Start time of the process since unix epoch.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1.673313858767E9
```
- process_files_max_files 最大文件数
- process_files_open_files 打开文件数
- process_cpu_usage cpu使用率
```text
# HELP process_cpu_usage The "recent cpu usage" for the Java Virtual Machine process
# TYPE process_cpu_usage gauge
process_cpu_usage 0.009232165838729317
```

### 系统
- system_cpu_count cpu个数
```text
# HELP system_cpu_count The number of processors available to the Java virtual machine
# TYPE system_cpu_count gauge
system_cpu_count 16.0
```
- system_cpu_usage cpu使用情况
```text
# HELP system_cpu_usage The "recent cpu usage" of the system the application is running in
# TYPE system_cpu_usage gauge
system_cpu_usage 0.09081119051122843
```
- system_load_average_1m 系统平均负载

### http请求
- http_server_requests_seconds 每秒http请求数
- http_server_requests_seconds_max http请求数峰值
```text
# HELP http_server_requests_seconds Duration of HTTP server request handling
# TYPE http_server_requests_seconds summary
http_server_requests_seconds_count{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",} 14.0
http_server_requests_seconds_sum{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",} 1.6270625

通过上面的数据可以发现一共请求14（http_server_requests_seconds_count）次，总时间为（http_server_requests_seconds_sum）

# HELP http_server_requests_seconds_max Duration of HTTP server request handling
# TYPE http_server_requests_seconds_max gauge
http_server_requests_seconds_max{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",} 0.0
最大请求时间为（http_server_requests_seconds_max）
```
**qps统计**
```text
sum(rate(http_server_requests_seconds_count{application="prometheus-example"}[10s]))
sum(rate(http_server_requests_seconds_count{instance="$instance", application="$application", uri!~".*actuator.*"}[5m]))
```
rate: 用于统计增长趋势，要求上报的Metric为Counter类型（只增不减）
irate: 与rate相似，区别在于rate统计的是一段时间内的平均增长速率，无法反应这个时间窗口内的突发情况（即瞬时高峰），irate通过区间向量中最后两个样本数据来计算增长速率，但是当选用的区间范围较大时，可能造成不小的偏差
sum: 求和，适用于统计场景
**耗时统计**
除了qps，另外一个经常关注的指标就是rt了，如上面接口的平均rt，通过两个Metric的组合来实现
```text
sum(rate(http_server_requests_seconds_sum{application="prometheus-example"}[10s])) / sum(rate(http_server_requests_seconds_count{application="prometheus-example"}[10s]))
```


### JVM监控
缓冲区
- jvm_buffer_count_buffers 计数缓冲
- jvm_buffer_memory_used_bytes 缓冲内存使用大小
- jvm_buffer_total_capacity_bytes 缓冲容量大小
```text
# HELP jvm_buffer_total_capacity_bytes An estimate of the total capacity of the buffers in this pool
# TYPE jvm_buffer_total_capacity_bytes gauge
jvm_buffer_total_capacity_bytes{id="direct",} 82632.0
jvm_buffer_total_capacity_bytes{id="mapped",} 0.0

# HELP jvm_buffer_memory_used_bytes An estimate of the memory that the Java virtual machine is using for this buffer pool
# TYPE jvm_buffer_memory_used_bytes gauge
jvm_buffer_memory_used_bytes{id="direct",} 82632.0
jvm_buffer_memory_used_bytes{id="mapped",} 0.0

# HELP jvm_buffer_count_buffers An estimate of the number of buffers in the pool
# TYPE jvm_buffer_count_buffers gauge
jvm_buffer_count_buffers{id="direct",} 11.0
jvm_buffer_count_buffers{id="mapped",} 0.0

```
类信息
- jvm_classes_loaded_classes 已加载类个数
- jvm_classes_unloaded_classes_total 已卸载类总数
```text
# HELP jvm_classes_loaded_classes The number of classes that are currently loaded in the Java virtual machine
# TYPE jvm_classes_loaded_classes gauge
jvm_classes_loaded_classes 8976.0

# HELP jvm_classes_unloaded_classes_total The total number of classes unloaded since the Java virtual machine has started execution
# TYPE jvm_classes_unloaded_classes_total counter
jvm_classes_unloaded_classes_total 0.0
```
gc信息
- jvm_gc_live_data_size_bytes gc存活数据大小
- jvm_gc_max_data_size_bytes gc最大数据大小
- jvm_gc_memory_allocated_bytes_total gc分配的内存大小
- jvm_gc_memory_promoted_bytes_total gc晋升到下一代的内存大小
- jvm_gc_pause_seconds gc等待的时间
- jvm_gc_pause_seconds_max gc等待的最大时间
```text
# HELP jvm_gc_memory_allocated_bytes_total Incremented for an increase in the size of the (young) heap memory pool after one GC to before the next
# TYPE jvm_gc_memory_allocated_bytes_total counter
jvm_gc_memory_allocated_bytes_total 2.41696768E8

# HELP jvm_gc_memory_promoted_bytes_total Count of positive increases in the size of the old generation memory pool before GC to after GC
# TYPE jvm_gc_memory_promoted_bytes_total counter
jvm_gc_memory_promoted_bytes_total 1.229668E7

# HELP jvm_gc_live_data_size_bytes Size of long-lived heap memory pool after reclamation
# TYPE jvm_gc_live_data_size_bytes gauge
jvm_gc_live_data_size_bytes 1.7648168E7

# HELP jvm_gc_max_data_size_bytes Max size of long-lived heap memory pool
# TYPE jvm_gc_max_data_size_bytes gauge
jvm_gc_max_data_size_bytes 2.751463424E9

# HELP jvm_gc_overhead_percent An approximation of the percent of CPU time used by GC activities over the last lookback period or since monitoring began, whichever is shorter, in the range [0..1]
# TYPE jvm_gc_overhead_percent gauge
jvm_gc_overhead_percent 0.0

# HELP jvm_gc_pause_seconds Time spent in GC pause
# TYPE jvm_gc_pause_seconds summary
jvm_gc_pause_seconds_count{action="end of major GC",cause="Metadata GC Threshold",} 1.0
jvm_gc_pause_seconds_sum{action="end of major GC",cause="Metadata GC Threshold",} 0.057
jvm_gc_pause_seconds_count{action="end of minor GC",cause="Metadata GC Threshold",} 1.0
jvm_gc_pause_seconds_sum{action="end of minor GC",cause="Metadata GC Threshold",} 0.009
jvm_gc_pause_seconds_count{action="end of minor GC",cause="Allocation Failure",} 1.0
jvm_gc_pause_seconds_sum{action="end of minor GC",cause="Allocation Failure",} 0.011
# HELP jvm_gc_pause_seconds_max Time spent in GC pause
# TYPE jvm_gc_pause_seconds_max gauge
jvm_gc_pause_seconds_max{action="end of major GC",cause="Metadata GC Threshold",} 0.0
jvm_gc_pause_seconds_max{action="end of minor GC",cause="Metadata GC Threshold",} 0.0
jvm_gc_pause_seconds_max{action="end of minor GC",cause="Allocation Failure",} 0.0

```

内存信息
- 已提交内存 jvm_memory_committed_bytes
- 最大内存 jvm_memory_max_bytes
- 已使用内存 jvm_memory_used_bytes
```text
# HELP jvm_memory_used_bytes The amount of used memory
# TYPE jvm_memory_used_bytes gauge
jvm_memory_used_bytes{area="heap",id="PS Survivor Space",} 1.3478848E7
jvm_memory_used_bytes{area="heap",id="PS Old Gen",} 1.765636E7
jvm_memory_used_bytes{area="heap",id="PS Eden Space",} 4.5597968E7
jvm_memory_used_bytes{area="nonheap",id="Metaspace",} 4.7203528E7
jvm_memory_used_bytes{area="nonheap",id="Code Cache",} 1.7431232E7
jvm_memory_used_bytes{area="nonheap",id="Compressed Class Space",} 6161240.0

# HELP jvm_memory_committed_bytes The amount of memory in bytes that is committed for the Java virtual machine to use
# TYPE jvm_memory_committed_bytes gauge
jvm_memory_committed_bytes{area="heap",id="PS Survivor Space",} 1.4680064E7
jvm_memory_committed_bytes{area="heap",id="PS Old Gen",} 1.31596288E8
jvm_memory_committed_bytes{area="heap",id="PS Eden Space",} 1.95035136E8
jvm_memory_committed_bytes{area="nonheap",id="Metaspace",} 5.0552832E7
jvm_memory_committed_bytes{area="nonheap",id="Code Cache",} 1.8284544E7
jvm_memory_committed_bytes{area="nonheap",id="Compressed Class Space",} 6774784.0

# HELP jvm_memory_max_bytes The maximum amount of memory in bytes that can be used for memory management
# TYPE jvm_memory_max_bytes gauge
jvm_memory_max_bytes{area="heap",id="PS Survivor Space",} 1.4680064E7
jvm_memory_max_bytes{area="heap",id="PS Old Gen",} 2.751463424E9
jvm_memory_max_bytes{area="heap",id="PS Eden Space",} 1.343225856E9
jvm_memory_max_bytes{area="nonheap",id="Metaspace",} -1.0
jvm_memory_max_bytes{area="nonheap",id="Code Cache",} 2.5165824E8
jvm_memory_max_bytes{area="nonheap",id="Compressed Class Space",} 1.073741824E9

# HELP jvm_memory_used_bytes The amount of used memory
# TYPE jvm_memory_used_bytes gauge
jvm_memory_used_bytes{area="heap",id="PS Survivor Space",} 1.3478848E7
jvm_memory_used_bytes{area="heap",id="PS Old Gen",} 1.765636E7
jvm_memory_used_bytes{area="heap",id="PS Eden Space",} 4.5597968E7
jvm_memory_used_bytes{area="nonheap",id="Metaspace",} 4.7203528E7
jvm_memory_used_bytes{area="nonheap",id="Code Cache",} 1.7431232E7
jvm_memory_used_bytes{area="nonheap",id="Compressed Class Space",} 6161240.0

# HELP jvm_memory_usage_after_gc_percent The percentage of long-lived heap pool used after the last GC event, in the range [0..1]
# TYPE jvm_memory_usage_after_gc_percent gauge
jvm_memory_usage_after_gc_percent{area="heap",pool="long-lived",} 0.006417079669673268
```

线程信息
- 守护线程 jvm_threads_daemon_threads
- 存活线程 jvm_threads_live_threads
- 线程峰值 jvm_threads_peak_threads
- 不同状态的线程 jvm_threads_states_threads
```text
# HELP jvm_threads_states_threads The current number of threads
# TYPE jvm_threads_states_threads gauge
jvm_threads_states_threads{state="runnable",} 6.0
jvm_threads_states_threads{state="blocked",} 0.0
jvm_threads_states_threads{state="waiting",} 12.0
jvm_threads_states_threads{state="timed-waiting",} 3.0
jvm_threads_states_threads{state="new",} 0.0
jvm_threads_states_threads{state="terminated",} 0.0

# HELP jvm_threads_live_threads The current number of live threads including both daemon and non-daemon threads
# TYPE jvm_threads_live_threads gauge
jvm_threads_live_threads 21.0

# HELP jvm_threads_peak_threads The peak live thread count since the Java virtual machine started or peak was reset
# TYPE jvm_threads_peak_threads gauge
jvm_threads_peak_threads 21.0

# HELP jvm_threads_daemon_threads The current number of live daemon threads
# TYPE jvm_threads_daemon_threads gauge
jvm_threads_daemon_threads 17.0
```

### 日志
- 打印日志个数 logback_events_total
```text
# HELP logback_events_total Number of events that made it to the logs
# TYPE logback_events_total counter
logback_events_total{level="warn",} 0.0
logback_events_total{level="debug",} 0.0
logback_events_total{level="error",} 0.0
logback_events_total{level="trace",} 0.0
logback_events_total{level="info",} 6.0


```
### rabbitmq
- 已发布消息数 rabbitmq_published_total
- 已消费消息数 rabbitmq_consumed_total
- 已拒绝消息数 rabbitmq_rejected_total
- 已确认消息数 rabbitmq_acknowledged_total
- 通道数 rabbitmq_channels
- 连接数 rabbitmq_connections
### integration
- 通道数 spring_integration_channels
- 处理器数 spring_integration_handlers
- 发送消息数 spring_integration_send_seconds
- 单位时间发送消息最大值 spring_integration_send_seconds_max


### tomcat信息
全局信息
- 总体报错数 tomcat_global_error_total
- 接收的字节总数 tomcat_global_received_bytes_total
- 发出的字节总数 tomcat_global_sent_bytes_total
- 每秒最大请求数 tomcat_global_request_max_seconds
- 每秒请求数 tomcat_global_request_seconds
会话信息
- 目前活跃会话数 tomcat_sessions_active_current_sessions
- 活跃最大会话数 tomcat_sessions_active_max_sessions
- 会话活跃的最长时间 tomcat_sessions_alive_max_seconds
- 累计创建的会话数 tomcat_sessions_created_sessions_total
- 累计失效的会话数 tomcat_sessions_expired_sessions_total
- 累计拒绝的会话数 tomcat_sessions_rejected_sessions_total
线程信息
- 繁忙的线程数 tomcat_threads_busy_threads
- 配置的最大线程数 tomcat_threads_config_max_threads
- 当前线程数 tomcat_threads_current_threads


