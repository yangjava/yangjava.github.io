---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# Dubbo集群组件Cluster


## Cluster 的概念
Dubbo集群容错中存在两个概念，分别是集群接口 Cluster 和 Cluster Invoker，这两者是不同的。Cluster 是接口，而 Cluster Invoker 是一种 Invoker。服务提供者的选择逻辑，以及远程调用失败后的的处理逻辑均是封装在 Cluster Invoker 中。而 Cluster 接口和相关实现类的用途比较简单，仅用于生成 Cluster Invoker。简单来说，Cluster 就是用来创建 Cluster Invoker 的，并且一一对应。而Cluster 和 Cluster Invoker 的作用就是，在消费者进行服务调用时选择何种容错策略，如：服务调用失败后是重试、还是抛出异常亦或者返回一个空的结果集等。
Cluster 接口如下：
```java
@SPI(FailoverCluster.NAME)
public interface Cluster {
	// 
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory) throws RpcException;
}
```
配置方式
```xml
<dubbo:reference cluster="failsafe" />
```
### Cluster 的种类
Cluster 接口存在多个实现对应不同的容错策略。 如下是org.apache.dubbo.rpc.cluster.Cluster 文件中针对不同协议的具体实现类：
```properties
mock=org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterWrapper
failover=org.apache.dubbo.rpc.cluster.support.FailoverCluster
failfast=org.apache.dubbo.rpc.cluster.support.FailfastCluster
failsafe=org.apache.dubbo.rpc.cluster.support.FailsafeCluster
failback=org.apache.dubbo.rpc.cluster.support.FailbackCluster
forking=org.apache.dubbo.rpc.cluster.support.ForkingCluster
available=org.apache.dubbo.rpc.cluster.support.AvailableCluster
mergeable=org.apache.dubbo.rpc.cluster.support.MergeableCluster
broadcast=org.apache.dubbo.rpc.cluster.support.BroadcastCluster
registryaware=org.apache.dubbo.rpc.cluster.support.RegistryAwareCluster
```
上面的每个类都 对应一个 Cluster Invoker， 其中 MockClusterWrapper 是 扩展类，对应的 Cluster Invoker 为 MockClusterInvoker，其余都是容错策略，其作用如下：
- MockClusterInvoker : MockClusterWrapper 对应的 Cluster Invoker。完成了本地Mock 的功能。这里需要注意由于MockClusterWrapper 是 扩展类，所以 MockClusterInvoker 在最外层，即当服务调用时的顺序为 ： MockClusterInvoker#invoker -> XxxClusterInvoker#invoker。关于 MockClusterInvoker 的实现，详参： Dubbo衍生篇⑦ ：本地Mock 和服务降级
- Failover Cluster：失败重试。当服务消费方调用服务提供者失败后，会自动切换到其他服务提供者服务器进行重试，这通常用于读操作或者具有幂等的写操作。需要注意的是，重试会带来更长延迟。可以通过retries="2"来设置重试次数（不含第1次）。 可以使用＜dubbo：reference retries=“2”/＞来进行接口级别配置的重试次数，当服务消费方调用服务失败后，此例子会再重试两次，也就是说最多会做3次调用，这里的配置对该接口的所有方法生效。
- Failfast Cluster：快速失败。当服务消费方调用服务提供者失败后，立即报错，也就是只调用一次。通常，这种模式用于非幂等性的写操作。
- Failsafe Cluster：安全失败。当服务消费者调用服务出现异常时，直接忽略异常。这种模式通常用于写入审计日志等操作。
- Failback Cluster：失败自动恢复。当服务消费端调用服务出现异常后，在后台记录失败的请求，并按照一定的策略后期再进行重试。这种模式通常用于消息通知操作。
- Forking Cluster：并行调用。当消费方调用一个接口方法后，Dubbo Client会并行调用多个服务提供者的服务，只要其中有一个成功即返回。这种模式通常用于实时性要求较高的读操作，但需要浪费更多服务资源。如下代码可通过forks="4"来设置最大并行数：
- Available Cluster ：可用集群调用器。前面提到doInvoke的入参有远程服务提供者的列表invokers。AvailableClusterInvoker遍历invokers，当遍历到第一个服务可用的提供者时，便访问该提供者，成功返回结果，如果访问时失败抛出异常终止遍历。
- Mergeable Cluster ：该集群容错策略是对多个服务端返回结果合并，在消费者调多个分组下的同一个服务时会指定使用该 Cluster 来合并 多个分组执行的结果。
- Broadcast Cluster：广播调用。当消费者调用一个接口方法后，Dubbo Client会逐个调用所有服务提供者，任意一台服务器调用异常则这次调用就标志失败。这种模式通常用于通知所有提供者更新缓存或日志等本地资源信息。
- RegistryAware Cluster ：当消费者引用多个注册中心时会指定使用该策略。默认会首先引用默认的注册中心服务，如果默认注册中心服务没有提供该服务，则会从其他注册中心中寻找该服务。
  Dubbo本身提供了丰富的集群容错模式，但是如果你有定制化需求，可以根据Dubbo提供的扩展接口Cluster进行定制。

上述的每个Cluster 对应一个Cluster Invoker， 如下 FailoverCluster 对应 FailoverClusterInvoker，而 FailoverClusterInvoker 才是容错逻辑实现的地方，所以下文会直接分析 Cluster Invoker。
```java
public class FailoverCluster implements Cluster {

    public final static String NAME = "failover";

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new FailoverClusterInvoker<T>(directory);
    }

}
```
由于每个Cluster Invoker 都是 AbstractClusterInvoker 的子类。例如 MockClusterInvoker#invoker 后的调用为：
```
MockClusterInvoker#invoker => AbstractClusterInvoker#invoke => AbstractClusterInvoker#doInvoker (AbstractClusterInvoker并未实现该方法，供子类实现)
```
其中 MockClusterInvoker 为 MockClusterWrapper 对应 Cluster Invoker。完成了 本地mock功能。




















