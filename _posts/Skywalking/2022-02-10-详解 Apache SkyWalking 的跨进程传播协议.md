简介
SkyWalking 跨进程传播协议是用于上下文的传播，本文介绍的版本是3.0，也被称为为sw8协议。

Header项
Header应该是上下文传播的最低要求。

Header名称：sw8.
Header值：由-分隔的8个字段组成。Header值的长度应该小于2KB。
Header值
Header值中具体包含以下8个字段：

采样（Sample），0 或 1，0 表示上下文存在, 但是可以（也很可能）被忽略；1 表示这个追踪需要采样并发送到后端。
追踪ID（Trace Id），是 BASE64 编码的字符串，其内容是由 . 分割的三个 long 类型值, 表示此追踪的唯一标识。
父追踪片段ID（Parent trace segment Id），是 BASE64 编码的字符串，其内容是字符串且全局唯一。
父跨度ID（Parent span Id），是一个从 0 开始的整数，这个跨度ID指向父追踪片段（segment）中的父跨度（span）。
父服务名称（Parent service），是 BASE64 编码的字符串，其内容是一个长度小于或等于50个UTF-8编码的字符串。
父服务实例标识（Parent service instance），是 BASE64 编码的字符串，其内容是一个长度小于或等于50个UTF-8编码的字符串。
父服务的端点（Parent endpoint），是 BASE64 编码的字符串，其内容是父追踪片段（segment）中第一个入口跨度（span）的操作名，由长度小于或等于50个UTF-8编码的字符组成。
本请求的目标地址（Peer），是 BASE64 编码的字符串，其内容是客户端用于访问目标服务的网络地址（不一定是 IP + 端口）。
Header值示例
上面的说明太干了，我们来举一个具体的例子，可以更好的理解。

有两个服务，分别叫onemore-a和 onemore-b，用户通过HTTP调用onemore-a的/onemore-a/get，然后onemore-a的/onemore-a/get又通过HTTP调用onemore-b的/onemore-b/get，流程图就是这样的：



那么，我们在onemore-b的/onemore-b/get的Header中就可以发现一个叫做sw8的key，其值为：

1-YTRlYzZmYzhjY2FiNGJiNGI2ODIwNjQ2OThjYzk3ZTYuNzQuMTYyMTgzODExMDQ1NTAwMDk=-YTRlYzZmYzhjY2FiNGJiNGI2ODIwNjQ2OThjYzk3ZTYuNzQuMTYyMTgzODExMDQ1NTAwMDg=-2-b25lbW9yZS1h-ZTFkMmZiYjYzYmJhNDMwNDk5YWY4OTVjMDQwZTMyZmVAMTkyLjE2OC4xLjEwMQ==-L29uZW1vcmUtYS9nZXQ=-MTkyLjE2OC4xLjEwMjo4MA==
1
以-字符进行分割，可以得到：

1，采样，表示这个追踪需要采样并发送到后端。
YTRlYzZmYzhjY2FiNGJiNGI2ODIwNjQ2OThjYzk3ZTYuNzQuMTYyMTgzODExMDQ1NTAwMDk=，追踪ID，解码后为：a4ec6fc8ccab4bb4b682064698cc97e6.74.16218381104550009
YTRlYzZmYzhjY2FiNGJiNGI2ODIwNjQ2OThjYzk3ZTYuNzQuMTYyMTgzODExMDQ1NTAwMDg=，父追踪片段ID，解码后为：a4ec6fc8ccab4bb4b682064698cc97e6.74.16218381104550009
2，父跨度ID。
b25lbW9yZS1h，父服务名称，解码后为：onemore-a
ZTFkMmZiYjYzYmJhNDMwNDk5YWY4OTVjMDQwZTMyZmVAMTkyLjE2OC4xLjEwMQ==，父服务实例标识，解码后为：e1d2fbb63bba430499af895c040e32fe@192.168.1.101
L29uZW1vcmUtYS9nZXQ=，父服务的端点，解码后为：/onemore-a/get
MTkyLjE2OC4xLjEwMjo4MA==，本请求的目标地址，解码后为：192.168.1.102:80
扩展Header项
扩展Header项是为高级特性设计的，它提供了部署在上游和下游服务中的探针之间的交互功能。

Header名称：sw8-x
Header值：由-分割，字段可扩展。
扩展Header值
当前值包括的字段：

追踪模式（Tracing Mode），空、0或1，默认为空或0。表示在这个上下文中生成的所有跨度（span）应该跳过分析。在默认情况下，这个应该在上下文中传播到服务端，除非它在跟踪过程中被更改。
