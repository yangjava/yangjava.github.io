---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo集群LoadBalance



## LoadBalance
当服务提供方是集群时，为了避免大量请求一直集中在一个或者几个服务提供方机器上，从而使这些机器负载很高，甚至导致服务不可用，需要做一定的负载均衡策略。Dubbo提供了多种均衡策略，默认为random，也就是每次随机调用一台服务提供者的服务。
LoadBalance 接口是 负载均衡实现的基础 SPI 接口，其实现如下：
```java
// 标注是 SPI 接口，默认实现是 RandomLoadBalance
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {
	// @Adaptive("loadbalance") 表示会为 select 生成代理方法，并且通过 url 中的 loadbalance 参数值决定选择何种负载均衡实现策略。
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
}
```
而 org.apache.dubbo.rpc.cluster.LoadBalance 文件如下：
```java
random=org.apache.dubbo.rpc.cluster.loadbalance.RandomLoadBalance
roundrobin=org.apache.dubbo.rpc.cluster.loadbalance.RoundRobinLoadBalance
leastactive=org.apache.dubbo.rpc.cluster.loadbalance.LeastActiveLoadBalance
consistenthash=org.apache.dubbo.rpc.cluster.loadbalance.ConsistentHashLoadBalance
```
由此我们可知，Dubbo提供的负载均衡策略有如下几种：
- Random LoadBalance：随机策略。按照概率设置权重，比较均匀，并且可以动态调节提供者的权重。
- RoundRobin LoadBalance：轮循策略。轮循，按公约后的权重设置轮循比率。会存在执行比较慢的服务提供者堆积请求的情况，比如一个机器执行得非常慢，但是机器没有宕机（如果宕机了，那么当前机器会从ZooKeeper 的服务列表中删除），当很多新的请求到达该机器后，由于之前的请求还没处理完，会导致新的请求被堆积，久而久之，消费者调用这台机器上的所有请求都会被阻塞。
- LeastActive LoadBalance：最少活跃调用数。如果每个提供者的活跃数相同，则随机选择一个。在每个服务提供者里维护着一个活跃数计数器，用来记录当前同时处理请求的个数，也就是并发处理任务的个数。这个值越小，说明当前服务提供者处理的速度越快或者当前机器的负载比较低，所以路由选择时就选择该活跃度最底的机器。如果一个服务提供者处理速度很慢，由于堆积，那么同时处理的请求就比较多，也就是说活跃调用数目较大（活跃度较高），这时，处理速度慢的提供者将收到更少的请求。
- ConsistentHash LoadBalance：一致性Hash策略。一致性Hash，可以保证相同参数的请求总是发到同一提供者，当某一台提供者机器宕机时，原本发往该提供者的请求，将基于虚拟节点平摊给其他提供者，这样就不会引起剧烈变动。

## AbstractLoadBalance
AbstractLoadBalance 类是 LoadBalance 实现类的父级抽象类，完成了一些基础的功能的实现。上面的各种负载均衡策略实际上就是继承了AbstractLoadBalance方法，但重写了其doSelect（）方法，所以下面我们重点看看每种负载均衡策略的doSelect（）方法
```java
/**
 * AbstractLoadBalance
 */
public abstract class AbstractLoadBalance implements LoadBalance {
 
    static int calculateWarmupWeight(int uptime, int warmup, int weight) {
        // 计算权重，下面代码逻辑上形似于 (uptime / warmup) * weight。
    	// 随着服务运行时间 uptime 增大，权重计算值 ww 会慢慢接近配置值 weight
        int ww = (int) ((float) uptime / ((float) warmup / (float) weight));
        return ww < 1 ? 1 : (ww > weight ? weight : ww);
    }

    @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    	// 服务校验，如果服务列表为空则返回null
        if (invokers == null || invokers.isEmpty()) {
            return null;
        }
        // 如果服务列表只有一个，则不需要负载均衡，直接返回
        if (invokers.size() == 1) {
            return invokers.get(0);
        }
        // 进行负载均衡，供子类实现
        return doSelect(invokers, url, invocation);
    }

    protected abstract <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation);

    //  服务提供者权重计算逻辑
    protected int getWeight(Invoker<?> invoker, Invocation invocation) {
    	// 从 url 中获取权重 weight 配置值。默认 100
        int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.WEIGHT_KEY, Constants.DEFAULT_WEIGHT);
        if (weight > 0) {
        	// 获取服务提供者启动时间戳
            long timestamp = invoker.getUrl().getParameter(Constants.REMOTE_TIMESTAMP_KEY, 0L);
            if (timestamp > 0L) {
            	// 计算服务提供者运行时长
                int uptime = (int) (System.currentTimeMillis() - timestamp);
                 // 获取服务预热时间，默认为10分钟
                int warmup = invoker.getUrl().getParameter(Constants.WARMUP_KEY, Constants.DEFAULT_WARMUP);
                // 如果服务运行时间小于预热时间，则重新计算服务权重，即降权
                if (uptime > 0 && uptime < warmup) {
                	 // 重新计算服务权重
                    weight = calculateWarmupWeight(uptime, warmup, weight);
                }
            }
        }
        return weight >= 0 ? weight : 0;
    }

}

```
上面是权重的计算过程，该过程主要用于保证当服务运行时长小于服务预热时间时，对服务进行降权，避免让服务在启动之初就处于高负载状态。服务预热是一个优化手段，与此类似的还有 JVM 预热。主要目的是让服务启动后“低功率”运行一段时间，使其效率慢慢提升至最佳状态。

## 策略实现

### RandomLoadBalance
RandomLoadBalance 是加权随机算法的具体实现。

它的算法思想很简单。假设我们有一组服务器 servers = [A, B, C]，他们对应的权重为 weights = [5, 3, 2]，权重总和为10。现在把这些权重值平铺在一维坐标值上，[0, 5) 区间属于服务器 A，[5, 8) 区间属于服务器 B，[8, 10) 区间属于服务器 C。接下来通过随机数生成器生成一个范围在 [0, 10) 之间的随机数，然后计算这个随机数会落到哪个区间上。比如数字3会落到服务器 A 对应的区间上，此时返回服务器 A 即可。权重越大的机器，在坐标轴上对应的区间范围就越大，因此随机数生成器生成的数字就会有更大的概率落到此区间内。只要随机数生成器产生的随机数分布性很好，在经过多次选择后，每个服务器被选中的次数比例接近其权重比例。比如，经过一万次选择后，服务器 A 被选中的次数大约为5000次，服务器 B 被选中的次数约为3000次，服务器 C 被选中的次数约为2000次。
RandomLoadBalance#doSelect 的实现如下：
```java
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // Number of invokers
        // 服务提供者数量
        int length = invokers.size();
        // Every invoker has the same weight?
        // 是否所有的服务提供者权重相等
        boolean sameWeight = true;
        // the weight of every invokers
        // 用来存储每个提供者的权重
        int[] weights = new int[length];
        // the first invoker's weight
        // 获取第一个提供者的权重
        int firstWeight = getWeight(invokers.get(0), invocation);
        weights[0] = firstWeight;
        // The sum of weights
        // 所有提供者的权重总和
        int totalWeight = firstWeight;
        // 计算出所有提供者的权重之和
        for (int i = 1; i < length; i++) {
        	// 获取权重
            int weight = getWeight(invokers.get(i), invocation);
            // save for later use
            weights[i] = weight;
            // Sum
            totalWeight += weight;
            if (sameWeight && weight != firstWeight) {
            	// 如果有权重不相等，则表明不是所有的权重都相等
                sameWeight = false;
            }
        }
        // 总权重大于0 && 权重不全相等，则根据权重进行分配调用
        if (totalWeight > 0 && !sameWeight) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
            // 生成一个 [0, totalWeight) 区间内的数字
            int offset = ThreadLocalRandom.current().nextInt(totalWeight);
            // Return a invoker based on the random value.
            // 按照上面说的规则判断这个随机数落入哪个提供者的区间，则返回该服务提供者
            for (int i = 0; i < length; i++) {
                offset -= weights[i];
                if (offset < 0) {
                    return invokers.get(i);
                }
            }
        }
        // If all invokers have the same weight value or totalWeight=0, return evenly.
        // 如果所有提供者权重相等，则随机返回一个
        return invokers.get(ThreadLocalRandom.current().nextInt(length));
    }

```
需要注意的是，这里没有使用Random而是使用了ThreadLocalRandom，这是出于性能上的考虑，因为Random在高并发下会导致大量线程竞争同一个原子变量，导致大量线程原地自旋，从而浪费CPU资源

RandomLoadBalance 的算法思想比较简单，在经过多次请求后，能够将调用请求按照权重值进行“均匀”分配。当然 RandomLoadBalance 也存在一定的缺点，当调用次数比较少时，Random 产生的随机数可能会比较集中，此时多数请求会落到同一台服务器上。这个缺点并不是很严重，多数情况下可以忽略。RandomLoadBalance 是一个简单，高效的负载均衡实现，因此 Dubbo 选择它作为缺省实现。

### RoundRobinLoadBalance
RoundRobinLoadBalance 即加权轮询负载均衡的实现。

在详细分析源码前，我们先来了解一下什么是加权轮询。这里从最简单的轮询开始讲起，所谓轮询是指将请求轮流分配给每台服务器。举个例子，我们有三台服务器 A、B、C。我们将第一个请求分配给服务器 A，第二个请求分配给服务器 B，第三个请求分配给服务器 C，第四个请求再次分配给服务器 A。这个过程就叫做轮询。轮询是一种无状态负载均衡算法，实现简单，适用于每台服务器性能相近的场景下。但现实情况下，我们并不能保证每台服务器性能均相近。如果我们将等量的请求分配给性能较差的服务器，这显然是不合理的。因此，这个时候我们需要对轮询过程进行加权，以调控每台服务器的负载。经过加权后，每台服务器能够得到的请求数比例，接近或等于他们的权重比。比如服务器 A、B、C 权重比为 5:2:1。那么在8次请求中，服务器 A 将收到其中的5次请求，服务器 B 会收到其中的2次请求，服务器 C 则收到其中的1次请求。

在上述基础上，此处的轮询算法还需要避免 类似 [A, A, A, A, A, B, B, C] 的情况产生，这样会使得 A 服务在短时间内接收了大量请求，而是需要实现 类似[A, A, B, A, A ,C, A, B] 的效果以均匀的对服务进行访问。
RoundRobinLoadBalance#doSelect 实现如下：

```java
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    	// 1. 获取调用方法的key，格式 key = 全限定类名 + "." + 方法名，比如 com.xxx.DemoService.sayHello
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        // 2. 获取该调用方法对应的每个服务提供者的 WeightedRoundRobin对象组成的map
        ConcurrentMap<String, WeightedRoundRobin> map = methodWeightMap.get(key);
        if (map == null) {
            methodWeightMap.putIfAbsent(key, new ConcurrentHashMap<String, WeightedRoundRobin>());
            map = methodWeightMap.get(key);
        }
        // 3. 遍历所有的提供者，计算总权重和权重最大的提供者
        int totalWeight = 0;
        long maxCurrent = Long.MIN_VALUE;
        long now = System.currentTimeMillis();
        Invoker<T> selectedInvoker = null;
        WeightedRoundRobin selectedWRR = null;
        for (Invoker<T> invoker : invokers) {
        	// 获取提供者唯一id ：格式为 protocol://ip:port/接口全限定类名。
        	//  如 dubbo：//30.10.75.231：20880/com.books.dubbo.demo.api.GreetingService
            String identifyString = invoker.getUrl().toIdentityString();
            WeightedRoundRobin weightedRoundRobin = map.get(identifyString);
            // 获取当前提供者的 权重
            int weight = getWeight(invoker, invocation);
			// 如果 weightedRoundRobin  为空，则说明 map 中还没有针对此提供者的权重信息，则创建后保存
            if (weightedRoundRobin == null) {
            	// 创建提供者的权重信息，并保存到map中
                weightedRoundRobin = new WeightedRoundRobin();
                weightedRoundRobin.setWeight(weight);
                map.putIfAbsent(identifyString, weightedRoundRobin);
            }
            // 如果权重有变化，更新权重信息
            if (weight != weightedRoundRobin.getWeight()) {
                //weight changed
                // 3.1 权重变化
                weightedRoundRobin.setWeight(weight);
            }
            // 这里 cur = cur + weight
            long cur = weightedRoundRobin.increaseCurrent();
            // 设置权重更新时间
            weightedRoundRobin.setLastUpdate(now);
            // 记录提供者中的最大权重的提供者
            if (cur > maxCurrent) {
            	// 最大权重
                maxCurrent = cur;
                // 最大权重的提供者
                selectedInvoker = invoker;
                selectedWRR = weightedRoundRobin;
            }
            // 记录总权重
            totalWeight += weight;
        }
        // 4. 更新 Map
        // CAS 确保线程安全: updateLock为true 时表示有线程在更新 methodWeightMap
        // 如果 invokers.size() != map.size() 则说明提供者列表有变化
        if (!updateLock.get() && invokers.size() != map.size()) {
            if (updateLock.compareAndSet(false, true)) {
                try {
                    // copy -> modify -> update reference
                    // 4.1 拷贝新值
                    ConcurrentMap<String, WeightedRoundRobin> newMap = new ConcurrentHashMap<String, WeightedRoundRobin>();
                    newMap.putAll(map);
                    // 4.2 更新map，移除过期值
                    Iterator<Entry<String, WeightedRoundRobin>> it = newMap.entrySet().iterator();
                    while (it.hasNext()) {
                        Entry<String, WeightedRoundRobin> item = it.next();
                        if (now - item.getValue().getLastUpdate() > RECYCLE_PERIOD) {
                            it.remove();
                        }
                    }
                    // 4.3 更新methodWeightMap
                    methodWeightMap.put(key, newMap);
                } finally {
                    updateLock.set(false);
                }
            }
        }
        // 5. 返回选择的 服务提供者
        if (selectedInvoker != null) {
        	// 更新selectedInvoker 权重 ： selectedInvoker 权重 = selectedInvoker 权重 - totalWeight
            selectedWRR.sel(totalWeight);
            return selectedInvoker;
        }
        // should not happen here
        // 返回第一个服务提供者，不会走到这里
        return invokers.get(0);
    }

```
其中 WeightedRoundRobin 为 RoundRobinLoadBalance 内部类，其实现如下：
```java
    protected static class WeightedRoundRobin {
        private int weight;
        private AtomicLong current = new AtomicLong(0);
        private long lastUpdate;
        public int getWeight() {
            return weight;
        }
        public void setWeight(int weight) {
            this.weight = weight;
            current.set(0);
        }
        public long increaseCurrent() {
            return current.addAndGet(weight);
        }
        public void sel(int total) {
            current.addAndGet(-1 * total);
        }
        public long getLastUpdate() {
            return lastUpdate;
        }
        public void setLastUpdate(long lastUpdate) {
            this.lastUpdate = lastUpdate;
        }
    }
```
我们按照代码来整理整个过程：
- 获取调用方法的 methodKey，格式 key = 全限定类名 + “.” + 方法名，比如 com.xxx.DemoService.sayHello
- 获取该调用方法对应的每个服务提供者的 WeightedRoundRobin对象组成的map。
这里 methodWeightMap 的定义为 ConcurrentMap<String, ConcurrentMap<String, WeightedRoundRobin>> ，其格式如下：
```java
|-- methodWeightMap 
 |    |-- key : 获取调用方法的 methodKey，格式 key = 全限定类名 + "." + 方法名。即第一步中的key 
 |		   |-- key : invoker.getUrl().toIdentityString(), 格式为 protocol://ip:port/接口全限定类名。
 					 如 dubbo：//30.10.75.231：20880/com.books.dubbo.demo.api.GreetingService
 |	 	   |-- value : WeightedRoundRobin对象，其中记录了当前提供者的权重和最后一次更新的时间

```
- 遍历所有的提供者，计算总权重和权重最大的提供者。需要注意经过这一步，每一个WeightedRoundRobin对象中的current 都经历了如下运算 : current = current + weight
- 更新 methodWeightMap ：当前没有线程更新 methodWeightMap && 提供者列表有变化。updateLock用来确保更新methodWeightMap的线程安全，这里使用了原子变量的CAS操作，减少了因为使用锁带来的开销。
- 创建 newMap 保存新的 提供者权重信息。 更新map，移除过期值。这里 RECYCLE_PERIOD 是清理周期，如果服务提供者耗时RECYCLE_PERIOD还没有更新自己的WeightedRoundRobin对象，则会被自动回收； 更新 methodWeightMap
- 返回权重最大的提供者。这里 selectedWRR.sel(totalWeight); 的作用是更新selectedInvoker 权重 ： selectedInvoker 权重 = selectedInvoker 权重 - totalWeight

### LeastActiveLoadBalance
LeastActiveLoadBalance 翻译过来是最小活跃数负载均衡。活跃调用数越小，表明该服务提供者效率越高，单位时间内可处理更多的请求。此时应优先将请求分配给该服务提供者。

在具体实现中，每个服务提供者对应一个活跃数 active。初始情况下，所有服务提供者活跃数均为0。每收到一个请求，活跃数加1，完成请求后则将活跃数减1。在服务运行一段时间后，性能好的服务提供者处理请求的速度更快，因此活跃数下降的也越快，此时这样的服务提供者能够优先获取到新的服务请求、这就是最小活跃数负载均衡算法的基本思想。除了最小活跃数，LeastActiveLoadBalance 在实现上还引入了权重值。所以准确的来说，LeastActiveLoadBalance 是基于加权最小活跃数算法实现的。

举个例子说明一下，在一个服务提供者集群中，有两个性能优异的服务提供者。某一时刻它们的活跃数相同，此时 Dubbo 会根据它们的权重去分配请求，权重越大，获取到新请求的概率就越大。如果两个服务提供者权重相同，此时随机选择一个即可。
LeastActiveLoadBalance#doSelect 的实现如下：
```java
  	@Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    	/*********** 1. 属性初始化 ************/
        // Number of invokers
        // 获取服务提供者个数
        int length = invokers.size();
        // The least active value of all invokers
        // 临时变量，用于暂存当前最小的活跃调用次数
        int leastActive = -1;
        // The number of invokers having the same least active value (leastActive)
        // 具有相同“最小活跃数”的服务者提供者（以下用 Invoker 代称）数量
        int leastCount = 0;
        // The index of invokers having the same least active value (leastActive)
        // 记录具有最小活跃调用的提供者在 invokers 中的下标位置
        int[] leastIndexes = new int[length];
        // the weight of every invokers
        // 记录每个服务提供者的权重
        int[] weights = new int[length];
        // The sum of the warmup weights of all the least active invokes
        // 记录活跃调用次数最小的服务提供者的权重和
        int totalWeight = 0;
        // The weight of the first least active invoke
        // 记录第一个调用次数等于最小调用次数的服务提供者的权重。用于与其他具有相同最小活跃数的 Invoker 的权重进行对比，
        // 以检测是否“所有具有相同最小活跃数的 Invoker 的权重”均相等
        int firstWeight = 0;
        // Every least active invoker has the same weight value?
        // 所有的调用次数等于最小调用次数的服务提供者的权重是否一样
        boolean sameWeight = true;
		/*********** 2. 挑选最小的活跃调用的服务提供者 ************/
        // Filter out all the least active invokers
        // 过滤出所有调用次数等于最小调用次数的服务提供者
        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            // Get the active number of the invoke
            // 获取当前服务提供者的被调用次数
            int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive();
            // Get the weight of the invoke configuration. The default value is 100.
            // 获取当前服务提供者的权重
            int afterWarmup = getWeight(invoker, invocation);
            // save for later use
            // 记录下当前服务提供者的权重
            weights[i] = afterWarmup;
            // If it is the first invoker or the active number of the invoker is less than the current least active number
            // 如果是第一个服务提供者(leastActive  == -1) 或者 当前服务提供者的活跃调用次数小于当前最小的活跃调用次数。
            // 满足上述条件则认为最小活跃调用相关信息需要更新，进行更性
            if (leastActive == -1 || active < leastActive) {
                // Reset the active number of the current invoker to the least active number
                // 记录下最新的最小活跃调用次数
                leastActive = active;
                // Reset the number of least active invokers
                // 最小活跃调用的 提供者数量为1
                leastCount = 1;
                // Put the first least active invoker first in leastIndexs
                // 记录最小活跃调用次数的提供者在  invokers 中的下标
                leastIndexes[0] = i;
                // Reset totalWeight
                // 重置最小活跃调用次数的提供者的权重和
                totalWeight = afterWarmup;
                // Record the weight the first least active invoker
                // 记录当前最小活跃调用的权重
                firstWeight = afterWarmup;
                // Each invoke has the same weight (only one invoker here)
                // 此时处于重置最小活跃调用次数信息，目前只有一个提供者，所以所有的调用次数等于最小调用次数的服务提供者的权重一样
                sameWeight = true;
                // If current invoker's active value equals with leaseActive, then accumulating.
            }
			// 如果当前提供者的活跃调用次数等于当前最小活跃调用次数
			 else if (active == leastActive) {
                // Record the index of the least active invoker in leastIndexs order
                // 记录当前服务提供者的下标
                leastIndexes[leastCount++] = i;
                // Accumulate the total weight of the least active invoker
                // 累加最小活跃调用者的权重
                totalWeight += afterWarmup;
                // If every invoker has the same weight?
                // 是否每个最小活跃调用者的权重都相等
                if (sameWeight && i > 0
                        && afterWarmup != firstWeight) {
                    sameWeight = false;
                }
            }
        }
		/*********** 3. 对最小活跃调用的服务提供者的处理 ************/
        // Choose an invoker from all the least active invokers
        // 如果只有一个最小调用者次数的 Invoker 则直接返回
        if (leastCount == 1) {
            // If we got exactly one invoker having the least active value, return this invoker directly.
            return invokers.get(leastIndexes[0]);
        }
        //  如果最小调用次数者的 Invoker 有多个且权重不同
        if (!sameWeight && totalWeight > 0) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on 
            // totalWeight.
            // 按照 RandomLoadBalance 的算法按照权重随机一个服务提供者。
            int offsetWeight = ThreadLocalRandom.current().nextInt(totalWeight);
            // Return a invoker based on the random value.
            for (int i = 0; i < leastCount; i++) {
                int leastIndex = leastIndexes[i];
                offsetWeight -= weights[leastIndex];
                if (offsetWeight < 0) {
                    return invokers.get(leastIndex);
                }
            }
        }
        // If all invokers have the same weight value or totalWeight=0, return evenly.
        // 如果最小调用此数者的invoker有多个并且权重相同，则随机拿一个使用。
        return invokers.get(leastIndexes[ThreadLocalRandom.current().nextInt(leastCount)]);
    }
```
简述思想：
这里的代码虽然看起来很复杂，但是其思路很简单：首先从所有服务提供者中获取 活跃调用最小的 提供者。但是因为活跃调用最小的提供者可能有多个。如果只有一个，则直接返回。如果存在多个，则从中按照 RandomLoadBalance 的权重算法挑选。

### ConsistentHashLoadBalance
ConsistentHashLoadBalance 实现的是 一致性Hash负载均衡策略。

ConsistentHashLoadBalance#doSelect 的实现如下：
```java
	// 每个 methodKey 对应一个 “hash环”
    private final ConcurrentMap<String, ConsistentHashSelector<?>> selectors = new ConcurrentHashMap<String, ConsistentHashSelector<?>>();
    
    @SuppressWarnings("unchecked")
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String methodName = RpcUtils.getMethodName(invocation);
        // 获取服务调用方法 的key
        String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName;
        // 获取 invokers 的hash 值
        int identityHashCode = System.identityHashCode(invokers);
        // 获取当前方法 key 对应的 一致性hash选择器 ConsistentHashSelector
        ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
         // 如果 invokers 是一个新的 List 对象，意味着服务提供者数量发生了变化，可能新增也可能减少了。
        // 此时 selector.identityHashCode != identityHashCode 条件成立
        if (selector == null || selector.identityHashCode != identityHashCode) {
        	// 创建新的 ConsistentHashSelector
            selectors.put(key, new ConsistentHashSelector<T>(invokers, methodName, identityHashCode));
            selector = (ConsistentHashSelector<T>) selectors.get(key);
        }
         // 调用 ConsistentHashSelector 的 select 方法选择 Invoker
        return selector.select(invocation);
    }
```
如上，doSelect 方法主要做了一些前置工作，比如检测 invokers 列表是不是变动过，以及创建 ConsistentHashSelector。
而重点则在于 ConsistentHashSelector 中。
- ConsistentHashSelector 在创建时完成了 Hash环的创建。
- ConsistentHashSelector#select 方法中完成了节点的查找。

### Hash环 的初始化
ConsistentHashSelector 的构造函数如下：
```java
       ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
            this.virtualInvokers = new TreeMap<Long, Invoker<T>>();
            this.identityHashCode = identityHashCode;
            URL url = invokers.get(0).getUrl();
            // 1. 获取虚拟节点数，默认为160。
            this.replicaNumber = url.getMethodParameter(methodName, "hash.nodes", 160);
            // 2. 获取参与 hash 计算的参数下标值，默认对第一个参数进行 hash 运算。
            String[] index = Constants.COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, "hash.arguments", "0"));
            argumentIndex = new int[index.length];
            // String 转 Integer
            for (int i = 0; i < index.length; i++) {
                argumentIndex[i] = Integer.parseInt(index[i]);
            }
            // 3.  根据所有服务提供者的invoker列表，生成从Hash环上的节点到服务提供者机器的映射关系，并存放到virtualInvokers中
            for (Invoker<T> invoker : invokers) {
                String address = invoker.getUrl().getAddress();
                for (int i = 0; i < replicaNumber / 4; i++) {
                	// 对 address + i 进行 md5 运算，得到一个长度为16的字节数组
                    byte[] digest = md5(address + i);
                    // 对 digest 部分字节进行4次 hash 运算，得到四个不同的 long 型正整数
                    for (int h = 0; h < 4; h++) {
                     	// h = 0 时，取 digest 中下标为 0 ~ 3 的4个字节进行位运算
	                    // h = 1 时，取 digest 中下标为 4 ~ 7 的4个字节进行位运算
	                    // h = 2, h = 3 时过程同上
                        long m = hash(digest, h);
                        // 将 hash 到 invoker 的映射关系存储到 virtualInvokers 中，
                    	// virtualInvokers 需要提供高效的查询操作，因此选用 TreeMap 作为存储结构
                        virtualInvokers.put(m, invoker);
                    }
                }
            }
        }
```
我们按照上面注释中的步骤：
- 获取虚拟节点数，默认为160。虚拟节点的数量代表每个服务提供者在 Hash环上有多少个虚拟节点。
- 获取参与 hash 计算的参数下标值，默认对第一个参数进行 hash 运算。即当消费者进行调用该方法时，会使用调用的第一个参数进行hash运算得到hash值并依据此hash值在hash环上找到对应的Invoker。后面调用的时候会详解。
- 根据所有服务提供者的invoker列表，生成从Hash环上的节点到服务提供者机器的映射关系，并存放到virtualInvokers中。简单来说，就是生成Hash环上的虚拟节点，并保存到 Hash环中。需要注意的是这里的 Hash环实现的数据结构是 TreeMap。

### 节点的查找
ConsistentHashSelector#select 的实现如下：
```java
		
 		public Invoker<T> select(Invocation invocation) {
 			// 1. 获取参与一致性Hash 的key。默认是方法的第一个参数
            String key = toKey(invocation.getArguments());
            // 2. 根据具体算法获取 key对应的md5值
            byte[] digest = md5(key);
            // 3. 计算  key 对应 hash 环上的哪个节点，并返回对应的Invoker。
            return selectForKey(hash(digest, 0));
        }

		// 根据 argumentIndex 指定的下标来获取调用方法时的入参并拼接后返回。
        private String toKey(Object[] args) {
            StringBuilder buf = new StringBuilder();
            for (int i : argumentIndex) {
                if (i >= 0 && i < args.length) {
                    buf.append(args[i]);
                }
            }
            return buf.toString();
        }
        // 从 hash 环 上获取对应的节点
        private Invoker<T> selectForKey(long hash) {
            Map.Entry<Long, Invoker<T>> entry = virtualInvokers.ceilingEntry(hash);
            if (entry == null) {
                entry = virtualInvokers.firstEntry();
            }
            return entry.getValue();
        }
		// 进行hash 算法
        private long hash(byte[] digest, int number) {
            return (((long) (digest[3 + number * 4] & 0xFF) << 24)
                    | ((long) (digest[2 + number * 4] & 0xFF) << 16)
                    | ((long) (digest[1 + number * 4] & 0xFF) << 8)
                    | (digest[number * 4] & 0xFF))
                    & 0xFFFFFFFFL;
        }
		// md5 计算
        private byte[] md5(String value) {
            MessageDigest md5;
            try {
                md5 = MessageDigest.getInstance("MD5");
            } catch (NoSuchAlgorithmException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            md5.reset();
            byte[] bytes;
            try {
                bytes = value.getBytes("UTF-8");
            } catch (UnsupportedEncodingException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            md5.update(bytes);
            return md5.digest();
        }

```
这里我们需要注意一下 toKey(invocation.getArguments()) 的实现。在上面 Hash环的初始化中，我们知道了在初始化Hash环时会获取 hash.arguments 属性，并转换为argumentIndex。其作用在于此，当消费者调用时，会使用前 argumentIndex+1 个入参值作为key进行 hash，依次来选择合适的服务提供者。
```java
		// 根据 argumentIndex 指定的下标来获取调用方法时的入参并拼接后返回。
        private String toKey(Object[] args) {
            StringBuilder buf = new StringBuilder();
            for (int i : argumentIndex) {
                if (i >= 0 && i < args.length) {
                    buf.append(args[i]);
                }
            }
            return buf.toString();
        }

```

## 自定义 LoadBalance
创建自定义的负载均衡类，实现LoadBalance 接口
```java
public class SimpleLoadBalance implements LoadBalance {
    @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException {
        return invokers.get(0);
    }
}
```
在META-INF/dubbo 目录下创建创建 org.apache.dubbo.rpc.cluster.LoadBalance 文件，并添加如下内容用以指定 simple 协议指定使用 SimpleLoadBalance 作为负载均衡策略 ：
```java
simple=com.kingfish.main.simple.SimpleLoadBalance
```
调用时指定使用 simple 协议的负载均衡策略
```
  @Reference(loadbalance = "simple")
    private DemoService demoService;
```


















































