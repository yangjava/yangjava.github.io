---
layout: post
categories: [Linux]
description: none
keywords: Linux
---
# Netfilter是如何工作的(一)：HOOK点

## Netfilter 的基本概念
Netfilter是一套融入在Linux内核网络协议栈中的报文处理(过滤或者修改)框架。它在内核中报文的关键流动路径上定义了5个HOOK点(下图蓝色方框)，各个协议(如IPv4、IPv6、ARP)可以在这些HOOK点安装钩子函数，报文流经此地，内核会按照优先级调用这些钩子函数，这些钩子函数最终会决定报文是被NF_ACCEPT(放行)还是NF_DROP(丢弃)。

图中红色虚线表示内核最常见的报文流经的路径：本机接收、转发、本机发送。
5个HOOK点分别是：路由前、本地上送、转发、本地发送、路由后1

## 链(chain) & 表(table)
初次接触iptables的同学可能会被四表五链这个名字吓到,特别是链这个名字真的很容易令人困惑! 而当你了解了Netfilter的实现细节后，才会发现：噢，原来链就是HOOK点，HOOK点就是链，因为有5个HOOK点，所以有五链！

那么，为什么要叫链呢？

因为一个HOOK点可以上可以安装多个钩子, 内核用“链条”将这些钩子串起来！

相比之下，四表(table)就没那么神秘了: 起过滤作用的filter表、起NAT作用的nat表，用于修改报文的mangle表,用于取消连接跟踪的raw表。

Netfilter设计多个表的目的，一方面是方便分类管理，另一方面，更重要的是为了限定各个钩子(或者说用户规则)执行的顺序！

以PREROUTING这个HOOK点为例，用户使用iptables设置的NAT规则和mangle会分别挂到nat hook和mangle hook，NAT表的优先级天生比mangle表低，因此报文一定会先执行mangle表的规则。

这就是 四表五链 的概念。我个人认为链的比表重要多了. 因为就算Netfilter没有表的概念，那么通过小心翼翼地设置各个rule的顺序其实也可以达到相同的效果。但链(也就是HOOK点)的作用是独一无二的。换个角度，用户在配置iptables规则时，更多的精力也是放在"应该在哪个HOOK点进行操作"，至于用的是filter表、nat表还是其他表，其实都是顺理成章的事情。

## HOOK 点的位置
用户通过iptables配置的规则最终会记录在HOOK点。HOOK点定义在struct net结构中，即HOOK点是各个net namespace中独立的。所以，在使用容器的场景中，每个容器的防火墙规则是独立的。
```
struct net {
    /* code omitted */
    struct netns_nf        nf;
    /* code omitted */
}

struct netns_nf {
    /* code omitted */
    struct list_head hooks[NFPROTO_NUMPROTO][NF_MAX_HOOKS];
};
```
从上面的定义可以看到，HOOK点是一个二维数组，每个元素都是一个链表头。它的第一个维度是协议类型，其中最常用的NFPROTO_IPV4，我们使用的iptables命令都是将这个钩子安装到这个协议的hook，而使用ip6tables就是将钩子安装到NFPROTO_IPV6的hook；第二个维度是链,对于IPV4来说，它的取值范围如下：
```
enum nf_inet_hooks{
    NF_INET_PRE_ROUTING,
    NF_INET_LOCAL_IN,
    NF_INET_FORWARD,
    NF_INET_LOCAL_OUT,
    NF_INET_POST_ROUTING,
    NF_INET_NUMHOOKS,
}
```

## HOOK 点的元素
hooks的每个元素都是链表头，链表上挂的元素类型是struct nf_hook_ops，这些元素有两个来源，一类来自于Netfilter初始化时各个表(如filter)的初始化，另一类来自于如连接跟踪这样的内部模块。下图展示了第一类来源的元素的挂接情况，它们按优先级排列(数字越小优先级越高)，而.hook就是报文到达对应的路径时会执行的钩子函数。
```
iptable_filter_init 
  |--xt_hook_link
    |-- nf_register_hooks
       |-- nf_register_hook
```

## HOOK 点的调用
Netfilter框架已经完全融入内核协议栈了，所以在协议栈代码中常常可以看到NF_HOOK宏的调用，这个宏的参数指定了HOOK点。

以本机收到IPv4报文为例
```
int ip_rcv(struct sk_buff* skb,...)
{
   // code omitted
   return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, net, NULL, skb, dev, NULL, ip_rcv_finish); 
   // code omitted
}
```
它指定要遍历的钩子函数是net namespace为net的hooks[NFPROTO_IPV4][NF_INET_PRE_ROUTING]链表上的元素,也就是上面图中的第一行的链表。如果三个钩子函数执行的结果(verdict)都是NF_ACCEPT，那么NF_HOOK指定的ip_rcv_finish就会被执行。

## 表(table)
内核使用struct xt_table这个结构来表示一个表(table)。结构中记录了这个表在哪些HOOK点有效，它的private指针保存了这个表包含的所有规则。

每个新的net namespace创建时，Netfitler会创建所有属于当前net namespace的表，并将其记录到自己的net namespace中。

从图中可以看出，创建一个包含默认rule的table分两步

根据已有的模板，生成默认rule
注册table到Netfilter框架中

## 生成默认rule
如果你使用过iptables,可能会知道每张table的每条chain都有一个默认POLICY,它表示chain中的所有的rule都不匹配，则执行默认的行为。这个默认行为可以通过iptables配置，在系统启动时默认行为都被设置为NF_ACCEPT.而这里的默认rule就是这个意思，Netfilter会为每个chain创建一条默认的rule，并将它们加入到table

ipt_replace
上图创建默认rule的时候使用了一个struct ipt_replace的结构，这个结构的来历和iptables的配置方法是紧密相连的：iptables采用的是读-修改-写的方式进行配置的！

举个例子，当你使用iptables向Netfilter的filter表添加一条新的rule时，iptables会将filter整个表读取到用户空间，在用户空间修改完成后，再将其重新设置到Netfilter。所以，这是一个替换(Replace)的过程。

因此，ipt_replace一般就是指用户使用iptables下发的表。典型的结构如下：hook_entry和underflow分别表示各个HOOK上的第一条rule和最后一条rule的偏移(具体的规则在最后一个ipt_entry的柔性数组里！)

但在这里，由于还处于初始化阶段，所以这里的repl是内核自己由packet_filter模板生成的，它会为filter所在的LOCAL_IN、FORWARD、LOCAL_OUT这几个HOOK点各创建一条默认rule.

ipt_entry
内核使用struct ipt_entry表示一条用户使用iptables配置的rule. 这个结构后面会紧跟若干个ipt_entry_match结构和1个ipt_standard_target结构。它与一条用户配置的iptables的关系如下：

其中

ipt_entry:表示标准匹配结构。包括报文的源IP、目的IP、入接口、出接口等条件
ipt_entry_match:表示扩展匹配。一条完整的rule包含零个到多个该结构
ipt_standard_target:表示匹配后的动作
如果对iptables的匹配规则不太熟悉，建议点此扩展匹配了解一下

## 注册 table 到 Netfilter 框架
在完成了默认规则的创建之后(保存在ipt_replace)，接下来就是应该注册新的表了。

xt_table中保存规则的private指针类型是struct xt_table_info，因此这里第一步就是把repl转换为newinfo。接下来就是调用xt_register_table创建新的表并注册到框架了

注册完成之后，Netfilter可以通过net->xt.tables[af]链表遍历到所有注册的表，也可以通过net->ipv4.快速访问到具体的表。

每一条iptables配置的rule都包含了匹配条件(match)部分和动作(target)。当报文途径HOOK点时，Netfilter会逐个遍历挂在该钩子点上的表的rule,若报文满足rule的匹配条件，内核就会执行动作(target)。

扩展match的表示
而match又可以分为标准match和扩展match两部分，其中前者有且只有一个，而后者有零到多个。在Netfilter中，标准match条件用ipt_entry表示(这个在上一篇文章中已经说过了)，其中包括源地址和目的地址等基本信息，而扩展match用ipt_entry_match表示：

这个结构上面是一个union，下面是一个柔性数组。union的元素有三个，抛开最后一个用于对齐的match_size，前两个元素表示它在用户空间和内核空间的有不同的解释(这个结构在linux内核代码和iptables应用程序代码中定义一样)

什么意思呢？ 还是以本文开头的那条rule为例，这条rule使用了tcp扩展模块，匹配条件为--dport 80,所以用户态的iptables程序会这样理解这个结构：

当内核收到这条rule时，会根据名字"tcp"去从Netfilter框架中"搜索"已注册的扩展match，若找到，就设置其match指针.这样，从内核Netfilter的角度来看，扩展匹配条件就变成了这样:

## 注册扩展 match
我们需要将扩展match预先注册到Netfilter框架中，才能在之后的配置中使用这个匹配条件。就拿本文最初的那条规则来说，需要一个隐含的前提就是tcp这个xt_match事先被注册到Netfilter框架了。这部分工作是在xt_tcpudp.c定义的内核模块中完成的。

除了tcp, 通过xt_register_match接口，其他各个模块都可以注册自己的扩展匹配条件。

每个xt_match有三个函数指针，其中

match：这个函数指针会在扩展匹配报文时被调用，用来判断skb是否满足条件，如果满足则返回非0值。
checkentry：这个函数在用户配置规则时被调用，如果返回0，表示配置失败。
destroy：这个函数在使用该match的规则被删除时调用。
使用扩展match
当数据包真的到达时，Netfilter会调用各个表安装的钩子函数，这些钩子函数用表中的每条rule去尝试匹配数据包

每一条iptables配置的规则(rule)都包含了匹配条件(match)部分和动作(target)。当报文途径HOOK点时，Netfilter会逐个遍历挂在该钩子点上的表的rule,若报文满足rule的匹配条件，内核就会执行动作(target)。

上面是一条普通iptables规则，如果报文匹配前面的条件，就会执行最后的SNAT，它就是这条规则的动作(target)

## 普通动作 & 扩展动作
动作又可分为普通target和扩展target两类，其中普通动作是指ACCEPT(允许报文通过)、DROP(拒绝报文通过)这类简单明了的行为，而扩展target是指包含了其他操作的动作，比如REJECT动作会产生ICMP差错信息、LOG动作记录日志、SNAT和DNAT用于进行地址转换。

## 表示 target
Netfilter使用xt_standard_target表示一个动作：

该结构由两部分组成，其中verdict是动作的编码, 取值有NF_DROP、NF_ACCEPT、NF_QUEUE等等。对于普通动作来说，有verdict就够了，但对于扩展动作来说，还需要有地方存储额外的信息(eg. 如何进行NAT),这里的target就是存储这些额外信息的地方。与本系列上一篇中说的xt_entry_match一样，xt_entry_target结构同样是一个union。它们的设计思路是一样的：内核代码和iptables用户态代码定义这样一个同样的数据类型，用户态使用的是user部分，设置要使用的扩展动作的name(普通动作的name为"")，内核收到该结构后，根据name查询到注册过的动作，将该信息挂到xt_entry_target的target指针上。而data字段表示的柔性数组区域就是各种扩展模块各显神通的地方了，对NAT来说，这里存储转换后的地址。

注册 target
我们需要将target预先注册到Netfilter框架中，才能在之后的配置中使用这个target。就拿本文最初的那条规则来说，需要一个隐含的前提就是SNAT这个xt_target事先被注册到Netfilter框架了。这部分工作在xt_nat.c定义的内核模块中完成：

除了SNAT, 通过xt_register_target接口，其他各个模块都可以注册自己的动作。根据名字进行区分，所有的target会挂到xf链表上。

每个target上有三个函数指针，其中

target：这个函数将决定skb的后续处理结果，如果为NULL,那么这条规则的动作就是普通target，处理结果从外面的verdict就可以得出。如果不为NULL,那么就执行这个函数，这个函数返回NF_DROP, NF_ACCEPT, NF_STOLEN这类动作
checkentry：这个函数在用户配置规则时被调用，如果返回0，表示配置失败。
destroy：这个函数在使用该target的规则被删除时调用。
查找 target
当用户通过iptables下发一条规则时，Netfilter会从xf链表上查找是否已有这样的target

执行 target
当数据包经过HOOK点，如果某条rule的匹配条件与报文一致，就会执行该rule包含的动作。

报文过滤和连接跟踪可以说是Netfilter提供的两大基本功能。前者被大多数人熟知，因为我们对防火墙的第一印象就是可以阻止有害的报文伤害计算机；而后者就没这么有名了，很多人甚至不知道Netfilter有这项功能。

Why 使用连接跟踪
顾名思义，连接跟踪是保存连接状态的一种机制。为什么要保存连接状态呢？ 举个例子，当你通过浏览器访问一个网站(连接网站的80端口)时，预期会收到服务器发送的源端口为80的报文回应，防火墙自然应该放行这些回应报文。那是不是所有源端口为80端口的报文都应该放行呢？显然不是，我们只应该放行源IP为服务器地址，源端口为80的报文，而应该阻止源地址不符的报文，即使它的源端口也是80。总结一下这种情况就是，我们只应该让主动发起的连接产生的双向报文通过。

另一个例子是NAT。我们可以使用iptables配置nat表进行地址或者端口转换的规则。如果每一个报文都去查询规则，这样效率太低了，因为同一个连接的转换方式是不变的！连接跟踪提供了一种缓存解决方案：当一条连接的第一个数据包通过时查询nat表时，连接跟踪将转换方法保存下来，后续的报文只需要根据连接跟踪里保存的转换方法就可以了。

连接跟踪发生在哪里
Connection tracking hooks into high-priority NF_IP_LOCAL_OUT and NF_IP_PRE_ROUTING hooks, in order to see packets before they enter the system.
连接跟踪需要拿到报文的第一手资料，因此它们的入口是以高优先级存在于LOCAL_OUT(本机发送)和PRE_ROUTING(报文接收)这两个链。

既然有入口，自然就有出口。连接跟踪采用的方案是在入口记录，在出口确认(confirm)。以IPv4为例：

当连接的第一个skb通过入口时，连接跟踪会将连接跟踪信息保存在skb->nfctinfo，而在出口处，连接跟踪会从skb上取下连接跟踪信息，保存在自己的hash表中。当然，如果这个数据包在中途其他HOOK点被丢弃了，也就不存在最后的confirm过程了。

连接跟踪信息是什么
连接跟踪信息会在入口处进行计算，保存在skb上，信息具体包括tuple信息(地址、端口、协议号等)、扩展信息以及各协议的私有信息。

tuple信息包括发送和接收两个方向，对TCP和UDP来说，是IP加Port；对ICMP来说是IP加Type和Code,等等；
扩展信息比较复杂，本文暂时略过；
各协议的私有信息，比如对TCP就是序号、重传次数、缩放因子等。
报文的连接跟踪状态
途径Netfilter框架的每一个报文总是会在入口处(PRE ROUTING或者LOCAL OUT)被赋予一个连接跟踪状态。这个状态存储在skb->nfctinfo，有以下常见的取值：

IP_CT_ESTABLISHED：这是一个属于已经建立连接的报文，Netfilter目击过两个方向都互通过报文了
IP_CT_RELATED：这个状态的报文所处的连接与另一个IP_CT_ESTABLISHED状态的连接是有联系的。比如典型的ftp，ftp-data的连接就是ftp-control派生出来的，它就是RELATED状态
IP_CT_NEW：这是连接的第一个包，常见的就是TCP中的SYN包，UDP、ICMP中第一个包，
IP_CT_ESTABLISHED + IP_CT_IS_REPLY：与IP_CT_ESTABLISHED类似，但是是在回复方向
IP_CT_RELATED + IP_CT_IS_REPLY：与IP_CT_RELATED类似，但是是在回复方向
总结
连接跟踪是Netfilter提供的一项基本功能，它可以保存连接的状态。用户可以为不同状态的连接的报文制定不同的策略；
连接跟踪在报文进入Netfilter的入口将信息记录在报文上，在出口进行confirm.确认后的连接信息可以影响之后的报文；
连接跟踪的信息主要包括基本的描述连接的tuple以及各协议的私有信息。
附录
目前一个连接跟踪的五元组为源目的IP，传输层协议，源目的端口。多租户环境下，租户的私有地址网络可能存在重叠，如果只用这五个元素来区分一个CT的话，无法满足多租户的需求。所以引入zone的概念，zone是一个16bit的整型数，不同用户使用不同的id，从而保证租户之间的隔离。















