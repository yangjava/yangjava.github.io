---
layout: post
categories: [Hadoop]
description: none
keywords: Hadoop
---
# 开发MapReduce应用程序

## 系统参数的配置
1.通过API对相关组件的参数进行配置

Hadoop有很多自己的组件（例如Hbase和Chukwa等），每一种组件都可以实现不同的功能，并起着不同的作用，通过多种组件的配合使用，Hadoop就能够实现非常强大的功能。这些可以通过Hadoop的API对相关参数进行配置来实现。

先简单地介绍一下API [1] ，它被分成了以下几个部分（也就是几个不同的包）。

org. apache.hadoop.conf：定义了系统参数的配置文件处理API；

org. apache.hadoop.fs：定义了抽象的文件系统API；

org. apache.hadoop.dfs：Hadoop分布式文件系统（HDFS）模块的实现；

org. apache.hadoop.mapred：Hadoop分布式计算系统（MapReduce）模块的实现，包括任务的分发调度等；

org. apache.hadoop.ipc：用在网络服务端和客户端的工具，封装了网络异步I/O的基础模块；


org. apache.hadoop.io：定义了通用的I/O API，用于针对网络、数据库、文件等数据对象进行读写操作等。

在此我们需要用到org.apache.hadoop.conf，用它来定义系统参数的配置。Configurations类由源来设置，每个源包含以XML形式出现的一系列属性/值对。每个源以一个字符串或一个路径来命名。如果是以字符串命名，则通过类路径检查该字符串代表的路径是否存在；如果是以路径命名的，则直接通过本地文件系统进行检查，而不用类路径。

下面举一个配置文件的例子。

configuration-default. xml

＜?xml version="1.0"?＞

＜configuration＞

＜property＞

＜name＞hadoop.tmp.dir＜/name＞

＜value＞/tmp/hadoop-${usr.name}＜/value＞

＜description＞A base for other temporary directories.＜/description＞

＜/property＞

＜property＞

＜name＞io.file.buffer.size＜/name＞

＜value＞4096＜/value＞

＜description＞the size of buffer for use in sequence file.＜/description＞

＜/property＞

＜property＞

＜name＞height＜/name＞

＜value＞tall＜/value＞

＜final＞true＜/final＞

＜/property＞

＜/configuration＞

这个文件中的信息可以通过以下的方式进行抽取：

Configuration conf=new Configuration（）；

Conf.addResource（"configuration-default.xml"）；

aasertThat（conf.get（"hadoop.tmp.dir"），is（"/tmp/hadoop-${usr.name}"））；

assertThat（conf.get（"io.file.buffer.size"），is（"4096"））；

assertThat（conf.get（"height"），is（"tall"））；

2.多个配置文件的整合

假设还有另外一个配置文件configuration-site.xml，其中具体代码细节如下：

configuration-site. xml

＜?xml version="1.0"?＞

＜configuration＞

＜property＞

＜name＞io.file.buffer.size＜/name＞

＜value＞5000＜/value＞

＜description＞the size of buffer for use in sequence file.＜/description＞

＜/property＞

＜property＞

＜name＞height＜/name＞

＜value＞short＜/value＞

＜final＞true＜/final＞

＜/property＞

＜/configuration＞

使用两个资源configuation-default.xml和configuration-site.xml来定义配置。将资源按顺序添加到Configuration之中，代码如下：

Configuration conf=new Configuration（）；

conf.addResource（"configuration-default.xml"）；

conf.addResource（"|configuration-site.xml"）；

现在不同资源中有了相同属性，但是这些属性的取值却不一样。这时这些属性的取值应该如何确定呢？可以遵循这样一个原则：后添加进来的属性取值覆盖掉前面所添加资源中的属性取值。因此，此处的属性io.file.buffer.size取值应该是5000而不是先前的4096，即：

assertThat（conf.get（"io.file.buffer.size"），is（"5000"））；

但是，有一个特例，被标记为final的属性不能被后面定义的属性覆盖。Configuration-default.xml中的属性height被标记为final，因此在configuration-site.xml中重写height并不会成功，它依然会从configuration-default.xml中取值：

assertThat（conf.get（"height"），is（"tall"））；

重写标记为final的属性通常会报告配置错误，同时会有警告信息被记录下来以便为诊断所用。管理员将守护进程地址文件之中的属性标记为final，可防止用户在客户端配置文件中或作业提交参数中改变其取值。

Hadoop默认使用两个源进行配置，并按顺序加载core-default.xml和core-site.xml。在实际应用中可能会添加其他的源，应按照它们添加的顺序进行加载。其中core-default.xml用于定义系统默认的属性，core-site.xml用于定义在特定的地方重写。


