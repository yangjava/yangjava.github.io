---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL源码MySQL Group Replication 的流控机制
Group Replication 是一种 Shared-Nothing 的架构，每个节点都会保留一份数据。

虽然支持多点写入，但实际上系统的吞吐量是由处理能力最弱的那个节点决定的。

如果各个节点的处理能力参差不齐，那处理能力慢的节点就会出现事务堆积。

在事务堆积的时候，如果处理能力快的节点出现了故障，这个时候能否让处理能力慢的节点（存在事务堆积）接受业务流量呢？

如果不等待堆积事务应用完，直接接受业务流量。

一方面会读到旧数据，另一方面也容易出现写冲突。

毕竟基于旧数据进行的写操作，它的 snapshot_version 肯定小于冲突检测数据库中对应记录的 snapshot_version。

如果等待堆积事务应用完才接受业务流量，又会影响数据库服务的可用性。

为了避免出现上述两难场景，Group Replication 引入了流控机制。

在实现上，Group Replication 的流控模块会定期检查各个节点的事务堆积情况，如果超过一定值，则会触发流控。

流控会基于上一周期各个节点的事务认证情况和事务应用情况，决定当前节点（注意是当前节点，不是其它节点）下个周期的写入配额。

超过写入配额的事务操作会被阻塞，等到下个周期才能执行。

接下来，我们通过源码分析下流控的实现原理。

本文主要包括以下几部分：

流控触发的条件。
配额的计算逻辑。
基于案例定量分析配额的计算逻辑。
配额作用的时机。
流控的相关参数。

## 流控触发的条件
默认情况下，节点的状态信息是每秒发送一次（节点的状态信息是在 flow_control_step 中发送的，发送周期由 group_replication_flow_control_period 决定）。

当接受到其它节点的状态信息时，会调用 Flow_control_module::handle_stats_data 来处理。
```
int Flow_control_module::handle_stats_data(const uchar *data, size_t len,
                                           const std::string &member_id) {
  DBUG_TRACE;
  int error = 0;
  Pipeline_stats_member_message message(data, len);

  m_flow_control_module_info_lock->wrlock();
  // m_info 是个字典，定义是 std::map<std::string, Pipeline_member_stats>
  // 其中，key 是节点的地址，value 是节点的状态信息。
  Flow_control_module_info::iterator it = m_info.find(member_id);
  // 如果 member_id 对应节点的状态信息在 m_info 中不存在，则插入。
  if (it == m_info.end()) {
    Pipeline_member_stats stats;

    std::pair<Flow_control_module_info::iterator, bool> ret = m_info.insert(
        std::pair<std::string, Pipeline_member_stats>(member_id, stats));
    error = !ret.second;
    it = ret.first;
  }
  // 更新节点的统计信息
  it->second.update_member_stats(message, m_stamp);

  // 检查是否需要流控
  if (it->second.is_flow_control_needed()) {
    ++m_holds_in_period;
#ifndef NDEBUG
    it->second.debug(it->first.c_str(), m_quota_size.load(),
                     m_quota_used.load());
#endif
  }

  m_flow_control_module_info_lock->unlock();
  return error;
}
```
首先判断节点的状态信息是否在 m_info 中存在。如果不存在，则插入。

接着通过 update_member_stats 更新节点的统计信息。

更新后的统计信息包括以下两部分：

当前数据：如 m_transactions_waiting_certification（当前等待认证的事务数），m_transactions_waiting_apply（当前等待应用的事务数）。

上一周期的增量数据：如 m_delta_transactions_certified（上一周期进行认证的事务数）。

m_delta_transactions_certified 等于 m_transactions_certified （这一次的采集数据） - previous_transactions_certified （上一次的采集数据）

最后会通过is_flow_control_needed判断是否需要流控。如果需要流控，则会将 m_holds_in_period 自增加 1。

如果是 Debug 版本，且将 log_error_verbosity 设置为 3。当需要流控时，会在错误日志中打印以下信息。
```
[Note] [MY-011726] [Repl] Plugin group_replication reported: 'Flow control - update member stats: 127.0.0.1:33071 stats certifier_queue 0, applier_queue 20 certified 387797 (308), applied 387786 (289), local 0 (0), quota 400 (274) mode=1'
```
什么时候会触发流控呢？

接下来我们看看 is_flow_control_needed 函数的处理逻辑。
```
bool Pipeline_member_stats::is_flow_control_needed() {
  return (m_flow_control_mode == FCM_QUOTA) &&
         (m_transactions_waiting_certification >
              get_flow_control_certifier_threshold_var() ||
          m_transactions_waiting_apply >
              get_flow_control_applier_threshold_var());
}
```
由此来看，触发流控需满足以下条件：

group_replication_flow_control_mode 设置为 QUOTA。

当前等待认证的事务数大于 group_replication_flow_control_certifier_threshold。

当前等待认证的事务数可通过 performance_schema.replication_group_member_stats 中的 COUNT_TRANSACTIONS_IN_QUEUE 查看。

当前等待应用的事务数大于 group_replication_flow_control_applier_threshold。

当前等待应用的事务数可通过 performance_schema.replication_group_member_stats 中的 COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE 查看。

除了条件 1，条件 2，3 满足其一即可。

当需要流控时，会将 m_holds_in_period 自增加 1。

m_holds_in_period 这个变量会在 Flow_control_module::flow_control_step 中使用。

而 Flow_control_module::flow_control_step 是在 Certifier_broadcast_thread::dispatcher() 中调用的，每秒执行一次。
```
void Certifier_broadcast_thread::dispatcher() {
  ...
  while (!aborted) {
    ...
    applier_module->run_flow_control_step();
    ...
    struct timespec abstime;
    // 定义超时时长 1s。
    set_timespec(&abstime, 1);
    mysql_cond_timedwait(&broadcast_dispatcher_cond, &broadcast_dispatcher_lock,
                         &abstime);
    mysql_mutex_unlock(&broadcast_dispatcher_lock);

    broadcast_counter++;
  }
}

void run_flow_control_step() override {
  flow_control_module.flow_control_step(&pipeline_stats_member_collector);
}
```
## 配额的计算逻辑
接下来我们重点分析下 flow_control_step 函数的处理逻辑。

这个函数非常关键，它是整个流控模块的核心。

它主要是用来计算 m_quota_size 和 m_quota_used。

其中，m_quota_size 决定了下个周期允许提交的事务数，即我们所说的配额。

m_quota_used 用来统计下个周期已经提交的事务数，在该函数中会重置为 0。
```
void Flow_control_module::flow_control_step(
    Pipeline_stats_member_collector *member) {
  // 这里的 seconds_to_skip 实际上就是 group_replication_flow_control_period，后面会有定义。
  // 虽然 flow_control_step 是一秒调用一次，但实际起作用的还是 group_replication_flow_control_period。
  if (--seconds_to_skip > 0) return;
  
  // holds 即 m_holds_in_period
  int32 holds = m_holds_in_period.exchange(0);
  // get_flow_control_mode_var() 即 group_replication_flow_control_mode
  Flow_control_mode fcm =
      static_cast<Flow_control_mode>(get_flow_control_mode_var());
  // get_flow_control_period_var() 即 group_replication_flow_control_period
  seconds_to_skip = get_flow_control_period_var();
  // 计数器
  m_stamp++;
  // 发送当前节点的状态信息
  member->send_stats_member_message(fcm);

  switch (fcm) {
    case FCM_QUOTA: {
      // get_flow_control_hold_percent_var() 即 group_replication_flow_control_hold_percent，默认是 10
      // 所以 HOLD_FACTOR 默认是 0.9
      double HOLD_FACTOR =
          1.0 -
          static_cast<double>(get_flow_control_hold_percent_var()) / 100.0;
      // get_flow_control_release_percent_var() 即 group_replication_flow_control_release_percent，默认是 50
      // 所以 RELEASE_FACTOR 默认是 1.5
      double RELEASE_FACTOR =
          1.0 +
          static_cast<double>(get_flow_control_release_percent_var()) / 100.0;
      // get_flow_control_member_quota_percent_var() 即 group_replication_flow_control_member_quota_percent，默认是 0
      // 所以 TARGET_FACTOR 默认是 0
      double TARGET_FACTOR =
          static_cast<double>(get_flow_control_member_quota_percent_var()) /
          100.0;
      // get_flow_control_max_quota_var() 即 group_replication_flow_control_max_quota，默认是 0
      int64 max_quota = static_cast<int64>(get_flow_control_max_quota_var());

      // 将上一个周期的 m_quota_size，m_quota_used 赋值给 quota_size，quota_used，同时自身重置为 0
      int64 quota_size = m_quota_size.exchange(0);
      int64 quota_used = m_quota_used.exchange(0);
      int64 extra_quota = (quota_size > 0 && quota_used > quota_size)
                              ? quota_used - quota_size
                              : 0;

      if (extra_quota > 0) {
        mysql_mutex_lock(&m_flow_control_lock);
        // 发送一个信号，释放 do_wait() 处等待的事务
        mysql_cond_broadcast(&m_flow_control_cond);
        mysql_mutex_unlock(&m_flow_control_lock);
      }
      // m_holds_in_period 大于 0，则意味着需要进行流控
      if (holds > 0) {
        uint num_writing_members = 0, num_non_recovering_members = 0;
        // MAXTPS 是 INT 的最大值，即 2147483647
        int64 min_certifier_capacity = MAXTPS, min_applier_capacity = MAXTPS,
              safe_capacity = MAXTPS;

        m_flow_control_module_info_lock->rdlock();
        Flow_control_module_info::iterator it = m_info.begin();
        // 循环遍历所有节点的状态信息
        while (it != m_info.end()) {
            // 这一段源码中没有，加到这里可以直观的看到触发流控时，每个节点的状态信息。
#ifndef NDEBUG
            it->second.debug(it->first.c_str(), quota_size,
                     quota_used);
#endif
          if (it->second.get_stamp() < (m_stamp - 10)) {
            // 如果节点的状态信息在最近 10 个周期内都没有更新，则清掉
            m_info.erase(it++);
          } else {
            if (it->second.get_flow_control_mode() == FCM_QUOTA) {
              // 如果 group_replication_flow_control_certifier_threshold 大于 0，
              // 且上一个周期进行认证的事务数大于 0，
              // 且当前等待认证的事务数大于 group_replication_flow_control_certifier_threshold，
              // 且上一个周期进行认证的事务数小于 min_certifier_capacity
              // 则会将上一个周期进行认证的事务数赋予 min_certifier_capacity
              if (get_flow_control_certifier_threshold_var() > 0 &&
                  it->second.get_delta_transactions_certified() > 0 &&
                  it->second.get_transactions_waiting_certification() -
                          get_flow_control_certifier_threshold_var() >
                      0 &&
                  min_certifier_capacity >
                      it->second.get_delta_transactions_certified()) {
                min_certifier_capacity =
                    it->second.get_delta_transactions_certified();
              }

              if (it->second.get_delta_transactions_certified() > 0)
                // safe_capacity 取 safe_capacity 和 it->second.get_delta_transactions_certified() 中的较小值
                safe_capacity =
                    std::min(safe_capacity,
                             it->second.get_delta_transactions_certified());


              // 针对的是 applier，逻辑同 certifier 一样
              if (get_flow_control_applier_threshold_var() > 0 &&
                  it->second.get_delta_transactions_applied() > 0 &&
                  it->second.get_transactions_waiting_apply() -
                          get_flow_control_applier_threshold_var() >
                      0) {
                if (min_applier_capacity >
                    it->second.get_delta_transactions_applied())
                  min_applier_capacity =
                      it->second.get_delta_transactions_applied();

                if (it->second.get_delta_transactions_applied() > 0)
                  // 如果上一个周期有事务应用，说明该节点不是 recovering 节点
                  num_non_recovering_members++;
              }

              if (it->second.get_delta_transactions_applied() > 0)
                // safe_capacity 取 safe_capacity 和 it->second.get_delta_transactions_applied() 中的较小值
                safe_capacity = std::min(
                    safe_capacity, it->second.get_delta_transactions_applied());

              if (it->second.get_delta_transactions_local() > 0)
                // 如果上一个周期有本地事务，则意味着该节点存在写入
                num_writing_members++;
            }
            ++it;
          }
        }
        m_flow_control_module_info_lock->unlock();

        num_writing_members = num_writing_members > 0 ? num_writing_members : 1;
        // min_capacity 取 min_certifier_capacity 和 min_applier_capacity 的较小值
        int64 min_capacity = (min_certifier_capacity > 0 &&
                              min_certifier_capacity < min_applier_capacity)
                                 ? min_certifier_capacity
                                 : min_applier_capacity;

        // lim_throttle 是最小配额
        int64 lim_throttle = static_cast<int64>(
            0.05 * std::min(get_flow_control_certifier_threshold_var(),
                            get_flow_control_applier_threshold_var()));
        // get_flow_control_min_recovery_quota_var() 即 group_replication_flow_control_min_recovery_quota
        if (get_flow_control_min_recovery_quota_var() > 0 &&
            num_non_recovering_members == 0)
          lim_throttle = get_flow_control_min_recovery_quota_var();
        // get_flow_control_min_quota_var() 即 group_replication_flow_control_min_quota
        if (get_flow_control_min_quota_var() > 0)
          lim_throttle = get_flow_control_min_quota_var();

        // min_capacity 不能太小，不能低于 lim_throttle
        min_capacity =
            std::max(std::min(min_capacity, safe_capacity), lim_throttle);

        // HOLD_FACTOR 默认是 0.9
        quota_size = static_cast<int64>(min_capacity * HOLD_FACTOR);

        // max_quota 是由 group_replication_flow_control_max_quota 定义的，即 quota_size 不能超过 max_quota
        if (max_quota > 0) quota_size = std::min(quota_size, max_quota);
        
        // num_writing_members 是有实际写操作的节点数
        if (num_writing_members > 1) {
          // 如果没有设置 group_replication_flow_control_member_quota_percent，则按照节点数平分 quota_size
          if (get_flow_control_member_quota_percent_var() == 0)
            quota_size /= num_writing_members;
          else
          // 如果有设置，则当前节点的 quota_size 等于 quota_size * group_replication_flow_control_member_quota_percent / 100
            quota_size = static_cast<int64>(static_cast<double>(quota_size) *
                                            TARGET_FACTOR);
        }
        // quota_size 还会减去上个周期超额使用的 quota
        quota_size =
            (quota_size - extra_quota > 1) ? quota_size - extra_quota : 1;
#ifndef NDEBUG
        LogPluginErr(INFORMATION_LEVEL, ER_GRP_RPL_FLOW_CONTROL_STATS,
                     quota_size, get_flow_control_period_var(),
                     num_writing_members, num_non_recovering_members,
                     min_capacity, lim_throttle);
#endif
      } else {
        // 对应 m_holds_in_period = 0 的场景，RELEASE_FACTOR 默认是 1.5
        if (quota_size > 0 && get_flow_control_release_percent_var() > 0 &&
            (quota_size * RELEASE_FACTOR) < MAXTPS) {
          // 当流控结束后，quota_size = 上一个周期的 quota_size * 1.5
          int64 quota_size_next =
              static_cast<int64>(quota_size * RELEASE_FACTOR);
          quota_size =
              quota_size_next > quota_size ? quota_size_next : quota_size + 1;
        } else
          quota_size = 0;
      }

      if (max_quota > 0)
        // quota_size 会取 quota_size 和 max_quota 中的较小值
        quota_size =
            std::min(quota_size > 0 ? quota_size : max_quota, max_quota);
      // 最后，将 quota_size 赋值给 m_quota_size，m_quota_used 重置为 0
      m_quota_size.store(quota_size);
      m_quota_used.store(0);
      break;
    }

    // 如果 group_replication_flow_control_mode 为 DISABLED，
    // 则会将 m_quota_size 和 m_quota_used 置为 0，这个时候会禁用流控。
    case FCM_DISABLED:
      m_quota_size.store(0);
      m_quota_used.store(0);
      break;

    default:
      assert(0);
  }

  if (local_member_info->get_recovery_status() ==
      Group_member_info::MEMBER_IN_RECOVERY) {
    applier_module->get_pipeline_stats_member_collector()
        ->compute_transactions_deltas_during_recovery();
  }
}
```
代码的逻辑看上去有点复杂。

接下来，我们通过一个具体的示例看看 flow_control_step 函数的实现逻辑。

## 案例分析
测试集群有三个节点组成：127.0.0.1:33061，127.0.0.1:33071 和 127.0.0.1:33081。

运行在多主模式下。

使用 sysbench 对 127.0.0.1:33061 进行插入测试（oltp_insert）。

为了更容易触发流控，这里将 127.0.0.1:33061 节点的 group_replication_flow_control_applier_threshold 设置为了 10。

以下是触发流控时 127.0.0.1:33061 的日志信息。
```
[Note] [MY-011726] [Repl] Plugin group_replication reported: 'Flow control - update member stats: 127.0.0.1:33061 stats certifier_queue 0, applier_queue 0 certified 7841 (177), applied 0 (0), local 7851 (177), quota 146 (156) mode=1'
[Note] [MY-011726] [Repl] Plugin group_replication reported: 'Flow control - update member stats: 127.0.0.1:33071 stats certifier_queue 0, applier_queue 0 certified 7997 (186), applied 8000 (218), local 0 (0), quota 146 (156) mode=1'
[Note] [MY-011726] [Repl] Plugin group_replication reported: 'Flow control - update member stats: 127.0.0.1:33081 stats certifier_queue 0, applier_queue 15 certified 7911 (177), applied 7897 (195), local 0 (0), quota 146 (156) mode=1'
[Note] [MY-011727] [Repl] Plugin group_replication reported: 'Flow control: throttling to 149 commits per 1 sec, with 1 writing and 1 non-recovering members, min capacity 177, lim throttle 0'
```
以 127.0.0.1:33081 的状态数据为例，我们看看输出中各项的具体含义：

certifier_queue 0：认证队列的长度。
applier_queue 15：应用队列的长度。
certified 7911 (177)：7911 是已经认证的总事务数，177 是上一周期进行认证的事务数（m_delta_transactions_certified）。
applied 7897 (195)：7897 是已经应用的总事务数，195 是上一周期应用的事务数（m_delta_transactions_applied）。
local 0 (0)：本地事务数。括号中的 0 是上一周期的本地事务数（m_delta_transactions_local）。
quota 146 (156)：146 是上一周期的 quota_size，156 是上一周期的 quota_used。
mode=1：mode 等于 1 是开启流控。
因为 127.0.0.1:33081 中 applier_queue 的长度（15）超过 127.0.0.1:33061 中的 group_replication_flow_control_applier_threshold（10），所以会触发流控。

触发流控后，会调用 flow_control_step 计算下一周期的 m_quota_size。

循环遍历各节点的状态信息。集群的吞吐量（min_capacity）取各个节点 m_delta_transactions_certified 和 m_delta_transactions_applied 的最小值。具体在本例中， min_capacity = min(177, 186, 218, 177, 195) = 177。

min_capacity 不能太小，不能低于 lim_throttle。im_throttle 的取值逻辑如下：

初始值是 0.05 * min (group_replication_flow_control_applier_threshold, group_replication_flow_control_certifier_threshold)。

具体在本例中，min_capacity = 0.05 * min(10, 25000) = 0.5。

如果设置了 group_replication_flow_control_min_recovery_quota 且 num_non_recovering_members 为 0，则会将 group_replication_flow_control_min_recovery_quota 赋值给 min_capacity。

num_non_recovering_members 什么时候会为 0 呢？在新节点加入时，因为认证队列中积压的事务超过阈值而触发的流控。

如果设置了 group_replication_flow_control_min_quota，则会将 group_replication_flow_control_min_quota 赋值给 min_capacity。

quota_size = min_capacity * 0.9 = 177 * 0.9 = 159。这里的 0.9 是 1 - group_replication_flow_control_hold_percent /100。之所以要预留部分配额，主要是为了处理积压事务。

quota_size 不能太大，不能超过 group_replication_flow_control_max_quota。

注意，这里计算的 quota_size 是集群的吞吐量，不是单个节点的吞吐量。如果要计算当前节点的吞吐量，最简单的办法是将 quota_size / 有实际写操作的节点数（num_writing_members）。怎么判断一个节点是否进行了实际的写操作呢？很简单，上一周期有本地事务提交，即 m_delta_transactions_local > 0。

具体在本例中，只有一个写节点，所以，当前节点的 quota_size 就等于集群的 quota_size，即 159。除了均分这个简单粗暴的方法，如果希望某些节点比其它节点承担更多的写操作，也可通过 group_replication_flow_control_member_quota_percent 设置权重。这个时候，当前节点的吞吐量就等于 quota_size * group_replication_flow_control_member_quota_percent / 100。

最后，当前节点的 quota_size 还会减去上个周期超额使用的 quota（extra_quota）。上个周期的 extra_quota 等于上个周期的 quota_used - quota_size = 156 - 146 = 10。所以，当前节点的 quota_size 就等于 159 - 10 = 149，和日志中的输出完全一致。为什么会出现 quota 超额使用的情况呢？这个后面会提到。

当 m_holds_in_period 又恢复为 0 时，就意味着流控结束。这个时候，MGR 不会完全放开 quota 的限制，否则写入量太大，容易出现突刺。MGR 采取的是一种渐进式的恢复策略，即下一周期的 quota_size = 上一周期的 quota_size * （1 + group_replication_flow_control_release_percent / 100）。

group_replication_flow_control_mode 是 DISABLED ，则会将 m_quota_size 和 m_quota_used 置为 0。m_quota_size 置为 0，实际上会禁用流控。为什么会禁用流控，这个后面会提到。

## 配额的作用时机
既然我们已经计算出下一周期的 m_quota_size，什么时候使用它呢？事务提交之后，GCS 广播事务消息之前。
```
int group_replication_trans_before_commit(Trans_param *param) {
  ...
  // 判断事务是否需要等待
  applier_module->get_flow_control_module()->do_wait();

  // 广播事务消息
  send_error = gcs_module->send_transaction_message(*transaction_msg);
  ...
}
```
接下来，我们看看 do_wait 函数的处理逻辑。
```
int32 Flow_control_module::do_wait() {
  DBUG_TRACE;
  // 首先加载 m_quota_size
  int64 quota_size = m_quota_size.load();
  // m_quota_used 自增加 1。
  int64 quota_used = ++m_quota_used;

  if (quota_used > quota_size && quota_size != 0) {
    struct timespec delay;
    set_timespec(&delay, 1);

    mysql_mutex_lock(&m_flow_control_lock);
    mysql_cond_timedwait(&m_flow_control_cond, &m_flow_control_lock, &delay);
    mysql_mutex_unlock(&m_flow_control_lock);
  }

  return 0;
}
```
如果 quota_size 等于 0，do_wait 会直接返回，不会执行任何等待操作。这也就是为什么当 m_quota_size 等于 0 时，会禁用流控操作。

如果 quota_used 大于 quota_size 且 quota_size 不等于 0，则意味着当前周期的配额用完了。这个时候，会调用 mysql_cond_timedwait 触发等待。

这里的 mysql_cond_timedwait 会在两种情况下退出：

收到 m_flow_control_cond 信号（该信号会在 flow_control_step 函数中发出 ）。
超时。这里的超时时间是 1s。
需要注意的是，m_quota_used 是自增在前，然后才进行判断，这也就是为什么 quota 会出现超额使用的情况。

在等待的过程中，如果客户端是多线程并发写入，这里会等待多个事务，并且超额使用的事务数不会多于客户端并发线程数。

所以，在上面的示例中，为什么 quota_used（156） 比 quota_size（146）多 10，这个实际上是 sysbench 并发线程数的数量。

## 流控的相关参数
group_replication_flow_control_mode

是否开启流控。默认是 QUOTA，基于配额进行流控。如果设置为 DISABLED ，则关闭流控。

group_replication_flow_control_period

流控周期。有效值 1 - 60，单位秒。默认是 1。注意，各个节点的流控周期应保持一致，否则的话，就会将周期较短的节点配额作为集群配额。

看下面这个示例，127.0.0.1:33061 这个节点的 group_replication_flow_control_period 是 10，而其它两个节点的 group_replication_flow_control_period 是 1。

group_replication_flow_control_applier_threshold

待应用的事务数如果超过 group_replication_flow_control_applier_threshold 的设置，则会触发流控，该参数默认是 25000。

group_replication_flow_control_certifier_threshold

待认证的事务数如果超过 group_replication_flow_control_certifier_threshold 的设置，则会触发流控，该参数默认是 25000。

group_replication_flow_control_min_quota

group_replication_flow_control_min_recovery_quota

两个参数都会决定当前节点下个周期的最小配额，只不过 group_replication_flow_control_min_recovery_quota 适用于新节点加入时的分布式恢复阶段。group_replication_flow_control_min_quota 则适用于所有场景。如果两者同时设置了， group_replication_flow_control_min_quota 的优先级更高。两者默认都为 0，即不限制。

group_replication_flow_control_max_quota

当前节点下个周期的最大配额。默认是 0，即不限制。

group_replication_flow_control_member_quota_percent

分配给当前成员的配额比例。有效值 0 - 100。默认为 0，此时，节点配额 = 集群配额 / 上个周期写节点的数量。

注意，这里的写节点指的是有实际写操作的节点，不是仅指 PRIMARY 节点。

group_replication_flow_control_hold_percent

预留配额的比例。有效值 0 - 100，默认是 10。预留的配额可用来处理落后节点积压的事务。

group_replication_flow_control_release_percent

当流控结束后，会逐渐增加吞吐量以避免出现突刺。

下一周期的 quota_size = 上一周期的 quota_size * （1 + group_replication_flow_control_release_percent / 100）。有效值 0 - 1000，默认是 50。

总结
从可用性的角度出发，不建议线上关闭流控。虽然主节点出现故障的概率很小，但按照墨菲定律，任何有可能发生的事情最后一定会发生。在线上还是不要心存侥幸。

流控限制的是当前节点的流量，不是其它节点的。

流控参数在各节点应保持一致，尤其是 group_replication_flow_control_period。

