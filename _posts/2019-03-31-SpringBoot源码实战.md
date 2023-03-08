---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot源码实战
背景：在公司的开发的过程中，偶然发现了一个很有用的spring的工具类-stopWatch.今天我们来聊聊spring一个比较有用的工具类背景：在公司的开发的过程中，偶然发现了一个很有用的spring的工具类-stopWatch.

之前的我们写代码的时候，如果想知道一个方法的执行耗时时长，代码的方式如下：

Long startTime = System.currentTimeMillis();
// 方法体
Long endTime = System.currentTimeMillis();
Long runTime = endTime - startTime;
上面的代码是不是很简单，但是这种写法看上去不美观，下面我们来研究下一个spring强大的工具类：stopWatch.

1 什么是stopWatch

stopWatch 是spring工具包org.springframework.util下的一个工具类，他主要就是计算同步单线程执行时间。我们先看看例子：

    public void TestStopWatch() throws InterruptedException {
        StopWatch stopWatch = new StopWatch("snail");
        stopWatch.start("snail_task1");
        Thread.sleep(1000);
        stopWatch.stop();
        stopWatch.start("snail_task2");
        Thread.sleep(2000);
        stopWatch.stop();
        System.out.println("stopWatch.prettyPrint()-------");
        System.out.println(stopWatch.prettyPrint());
    }
运行结果：


start开始记录，stop结束记录，然后通过prettyPrint方法打印出来，可以直观显示每段代码的执行时间以及百分比，比之间用系统时间来比瞬间高大上了。

这个工具类还有其他两个方法，计算这个stopWatch实例的所有任务的总的执行时间shortSummary,getTotalTimeMillis，如下图：


2 源码探索

其实这个工具类的源码比较简单，其实就是把我们之前的写法封装了一下，我们一起看下源码。

我们先来看看这个类的一些属性：

public class StopWatch {
// 这个类的实例的id值
private final String id;
// 是否保留任务，默认为True是保留任务信息
private boolean keepTaskList;
// 任务计略列表
private final List<StopWatch.TaskInfo> taskList;
// 开始时间
private long startTimeMillis;
// 当前任务名称
@Nullable
private String currentTaskName;
// 最后一个任务信息
@Nullable
private StopWatch.TaskInfo lastTaskInfo;
// 任务总数
private int taskCount;
// 总花费时间
private long totalTimeMillis;
下面我们来看看start方法和stop方法：

public void start(String taskName) throws IllegalStateException {
if (this.currentTaskName != null) {
throw new IllegalStateException("Can't start StopWatch: it's already running");
} else {
// 看这里，是不是很熟悉，跟我们之前写的是一样的
this.currentTaskName = taskName;
this.startTimeMillis = System.currentTimeMillis();
}
}

    public void stop() throws IllegalStateException {
        if (this.currentTaskName == null) {
            throw new IllegalStateException("Can't stop StopWatch: it's not running");
        } else {
        // 这里结束也是，直接相减
            long lastTime = System.currentTimeMillis() - this.startTimeMillis;
            this.totalTimeMillis += lastTime;
            this.lastTaskInfo = new StopWatch.TaskInfo(this.currentTaskName, lastTime);
            if (this.keepTaskList) {
            // 这里分装保留每个任务的运行时间，可以随时获取的到
                this.taskList.add(this.lastTaskInfo);
            }
            // 统计任务数量
            ++this.taskCount;
            this.currentTaskName = null;
        }
    }
一个任务信息的内部类：

public static final class TaskInfo {
private final String taskName;
private final long timeMillis;

        TaskInfo(String taskName, long timeMillis) {
            this.taskName = taskName;
            this.timeMillis = timeMillis;
        }

        public String getTaskName() {
            return this.taskName;
        }

        public long getTimeMillis() {
            return this.timeMillis;
        }

        public double getTimeSeconds() {
            return (double)this.timeMillis / 1000.0D;
        }
    }
任务信息打印方法：

public String prettyPrint() {
StringBuilder sb = new StringBuilder(this.shortSummary());
sb.append('\n');
if (!this.keepTaskList) {
sb.append("No task info kept");
} else {
sb.append("-----------------------------------------\n");
sb.append("ms     %     Task name\n");
sb.append("-----------------------------------------\n");
NumberFormat nf = NumberFormat.getNumberInstance();
nf.setMinimumIntegerDigits(5);
nf.setGroupingUsed(false);
NumberFormat pf = NumberFormat.getPercentInstance();
pf.setMinimumIntegerDigits(3);
pf.setGroupingUsed(false);
StopWatch.TaskInfo[] var4 = this.getTaskInfo();
int var5 = var4.length;

            for(int var6 = 0; var6 < var5; ++var6) {
                StopWatch.TaskInfo task = var4[var6];
                sb.append(nf.format(task.getTimeMillis())).append("  ");
                sb.append(pf.format(task.getTimeSeconds() / this.getTotalTimeSeconds())).append("  ");
                sb.append(task.getTaskName()).append("\n");
            }
        }

        return sb.toString();
    }
上面是比较常用的方法，看起来实现比较简单，只是帮我封装好了，直接调用就行。

3 优缺点

优点：

spring自带工具类，可直接使用
代码实现简单，使用更简单
统一归纳，展示每项任务耗时与占用总时间的百分比，展示结果直观，性能消耗相对较小，并且最大程度的保证了start与stop之间的时间记录的准确性
可在start时直接指定任务名字，从而更加直观的显示记录结果
缺点：

一个StopWatch实例一次只能开启一个task，不能同时start多个task，并且在该task未stop之前不能start一个新的task，必须在该task stop之后才能开启新的task，若要一次开启多个，需要new不同的StopWatch实例
代码侵入式使用，需要改动多处代码
不单单只有spring提供了这个工具类，apache，google也提供了，有兴趣可以去研究下，源码是挺简单的，大同小异 ：

import com.google.common.base.Stopwatch;

import org.apache.commons.lang3.time.StopWatch;

但是以springframework.util.StopWatch效果最好，推荐使用。

本次就记录到此，我们下期再会！ 你知道的越多，你不知道的越多！



## Spring常用工具类
利用Spring将Object转为Map
```properties
import org.springframework.beans.BeanUtils;
import org.springframework.util.ReflectionUtils;

    private static Map<String, String> convertObjectToMap(Object obj){
        return Arrays.stream(BeanUtils.getPropertyDescriptors(obj.getClass()))
                .filter(pd -> !"class".equals(pd.getName()))
                .collect(HashMap::new,
                        (map, pd) -> {
                            if(ReflectionUtils.invokeMethod(pd.getReadMethod(), obj) != null) {
                                map.put(pd.getName(), String.valueOf(ReflectionUtils.invokeMethod(pd.getReadMethod(), obj)));
                            }
                        },
                        HashMap::putAll);
    }
```