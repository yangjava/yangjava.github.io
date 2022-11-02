---
layout: post
categories: Skywalking
description: none
keywords: Skywalking
---
# skywalking配置

## 配置覆盖

### 概述

- 我们每次部署应用都需要到agent中修改服务名，如果部署多个应用，那么只能复制agent以便产生隔离，但是这样非常麻烦。我们可以用Skywalking提供的配置覆盖功能通过启动命令动态的指定服务名，这样agent就只需要部署一份即可。

- Skywalking支持以下几种配置：

- - 系统配置。

- - 探针配置。

- - 系统环境变量。

- - 配置文件中的值（上面的案例中我们就是这样使用)。



### 系统配置（System Properties）

- 使用`skywalking.`+配置文件中的配置名作为系统配置项来进行覆盖。

- 例如：通过命令行启动的时候加上如下的参数可以进行`agent.service_name`的覆盖



```shell
-Dskywalking.agent.service_name=skywalking_mysql
```



###  探针配置（Agent Options）

- 可以指定探针的时候加上参数，如果配置中包含分隔符（,或=），就必须使用引号（‘’）包裹起来。

```shell
-javaagent:/usr/local/skywalking/apache-skywalking-apm-bin-es7/agent/skywalking-agent.jar=[option]=[value],[option]=[value]
```

- 例如：

```shell
-javaagent:/usr/local/skywalking/apache-skywalking-apm-bin-es7/agent/skywalking-agent.jar=agent.service_name=skywalking_mysql
```

###  系统环境变量

- 案例：

- 由于agent.service_name配置项如下所示

```properties
agent.service_name=${SW_AGENT_NAME:Your_ApplicationName}
```

- 可以在环境变量中设置SW_AGENT_NAME的值来指定服务名。

###  覆盖的优先级

- 探针配置>系统配置>系统环境变量>配置文件中的值。

- 简而言之，推荐使用`探针方式`或`系统配置`的方式。