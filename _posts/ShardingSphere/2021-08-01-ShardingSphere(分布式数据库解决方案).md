# **Apache ShardingSphere**

## 简介

Apache ShardingSphere 是一套开源的分布式数据库解决方案组成的生态圈，它由 JDBC、Proxy 和 Sidecar（规划中）这 3 款既能够独立部署，又支持混合部署配合使用的产品组成。 它们均提供标准化的数据水平扩展、分布式事务和分布式治理等功能，可适用于如 Java 同构、异构语言、云原生等各种多样化的应用场景。

Apache ShardingSphere 旨在充分合理地在分布式的场景下利用关系型数据库的计算和存储能力，而并非实现一个全新的关系型数据库。 关系型数据库当今依然占有巨大市场份额，是企业核心系统的基石，未来也难于撼动，我们更加注重在原有基础上提供增量，而非颠覆。

Apache ShardingSphere 5.x 版本开始致力于可插拔架构，项目的功能组件能够灵活的以可插拔的方式进行扩展。 目前，数据分片、读写分离、数据加密、影子库压测等功能，以及 MySQL、PostgreSQL、SQLServer、Oracle 等 SQL 与协议的支持，均通过插件的方式织入项目。 开发者能够像使用积木一样定制属于自己的独特系统。Apache ShardingSphere 目前已提供数十个 SPI 作为系统的扩展点，仍在不断增加中。

ShardingSphere 已于2020年4月16日成为 Apache 软件基金会的顶级项目。

[简介](https://shardingsphere.apache.org/document/5.0.0/cn/overview/#简介)[ShardingSphere-JDBC](https://shardingsphere.apache.org/document/5.0.0/cn/overview/#shardingsphere-jdbc)[ShardingSphere-Proxy](https://shardingsphere.apache.org/document/5.0.0/cn/overview/#shardingsphere-proxy)[ShardingSphere-Sidecar（TODO）](https://shardingsphere.apache.org/document/5.0.0/cn/overview/#shardingsphere-sidecartodo)[混合架构](https://shardingsphere.apache.org/document/5.0.0/cn/overview/#混合架构)[解决方案](https://shardingsphere.apache.org/document/5.0.0/cn/overview/#解决方案)[线路规划](https://shardingsphere.apache.org/document/5.0.0/cn/overview/#线路规划)

[![GitHub release](https://img.shields.io/github/release/apache/shardingsphere.svg?style=social&label=Release)](https://github.com/apache/shardingsphere/releases) [![GitHub stars](https://img.shields.io/github/stars/apache/shardingsphere.svg?style=social&label=Star)](https://github.com/apache/shardingsphere/stargazers) [![GitHub forks](https://img.shields.io/github/forks/apache/shardingsphere.svg?style=social&label=Fork)](https://github.com/apache/shardingsphere/fork) [![GitHub watchers](https://img.shields.io/github/watchers/apache/shardingsphere.svg?style=social&label=Watch)](https://github.com/apache/shardingsphere/watchers)

**星评增长时间线**

[![Stargazers over time](https://starchart.cc/apache/shardingsphere.svg)](https://starchart.cc/apache/shardingsphere)

**贡献者增长时间线**

[![Contributor over time](https://contributor-graph-api.apiseven.com/contributors-svg?chart=contributorOverTime&repo=apache/shardingsphere)](https://www.apiseven.com/en/contributor-graph?chart=contributorOverTime&repo=apache/shardingsphere)

Apache ShardingSphere 产品定位为 `Database Plus`，旨在构建多模数据库上层的标准和生态。 它关注如何充分合理地利用数据库的计算和存储能力，而并非实现一个全新的数据库。ShardingSphere 站在数据库的上层视角，关注他们之间的协作多于数据库自身。

`连接`、`增量`和`可插拔`是 Apache ShardingSphere 的核心概念。

- `连接`：通过对数据库协议、SQL 方言以及数据库存储的灵活适配，快速的连接应用与多模式的异构数据库；
- `增量`：获取数据库的访问流量，并提供流量重定向（数据分片、读写分离、影子库）、流量变形（数据加密、数据脱敏）、流量鉴权（安全、审计、权限）、流量治理（熔断、限流）以及流量分析（服务质量分析、可观察性）等透明化增量功能；
- `可插拔`：项目采用微内核 + 三层可插拔模型，使内核、功能组件以及生态对接完全能够灵活的方式进行插拔式扩展，开发者能够像使用积木一样定制属于自己的独特系统。

ShardingSphere 已于2020年4月16日成为 [Apache 软件基金会](https://apache.org/index.html#projects-list)的顶级项目。 欢迎通过[邮件列表](mailto:dev@shardingsphere.apache.org)参与讨论。

[![License](https://img.shields.io/badge/license-Apache%202-4EB1BA.svg)](https://www.apache.org/licenses/LICENSE-2.0.html) [![GitHub release](https://img.shields.io/github/release/apache/shardingsphere.svg)](https://github.com/apache/shardingsphere/releases)

[![Twitter](https://img.shields.io/twitter/url/https/twitter.com/ShardingSphere.svg?style=social&label=Follow%20%40ShardingSphere)](https://twitter.com/ShardingSphere) [![Slack](https://img.shields.io/badge/%20Slack-ShardingSphere%20Channel-blueviolet)](https://join.slack.com/t/apacheshardingsphere/shared_invite/zt-sbdde7ie-SjDqo9~I4rYcR18bq0SYTg) [![Gitter](https://badges.gitter.im/shardingsphere/shardingsphere.svg)](https://gitter.im/shardingsphere/Lobby)

[![Build Status](https://api.travis-ci.org/apache/shardingsphere.svg?branch=master&status=created)](https://travis-ci.org/apache/shardingsphere) [![codecov](https://codecov.io/gh/apache/shardingsphere/branch/master/graph/badge.svg)](https://codecov.io/gh/apache/shardingsphere) [![snyk](https://snyk.io/test/github/apache/shardingsphere/badge.svg?targetFile=pom.xml)](https://snyk.io/test/github/apache/shardingsphere?targetFile=pom.xml) [![Maintainability](https://cloud.quality-gate.com/dashboard/api/badge?projectName=apache_shardingsphere&branchName=master)](https://cloud.quality-gate.com/dashboard/branches/30#overview)

[![OpenTracing-1.0 Badge](https://img.shields.io/badge/OpenTracing--1.0-enabled-blue.svg)](http://opentracing.io/) [![Skywalking Tracing](https://img.shields.io/badge/Skywalking%20Tracing-enable-brightgreen.svg)](https://github.com/apache/skywalking)

[![Overview](https://shardingsphere.apache.org/document/current/img/overview.cn_v2.png)](https://shardingsphere.apache.org/document/current/img/overview.cn_v2.png)

## 简介

Apache ShardingSphere 由 JDBC、Proxy 和 Sidecar（规划中）这 3 款既能够独立部署，又支持混合部署配合使用的产品组成。 它们均提供标准化的基于数据库作为存储节点的增量功能，可适用于如 Java 同构、异构语言、云原生等各种多样化的应用场景。

关系型数据库当今依然占有巨大市场份额，是企业核心系统的基石，未来也难于撼动，我们更加注重在原有基础上提供增量，而非颠覆。

### ShardingSphere-JDBC

[![Maven Status](https://maven-badges.herokuapp.com/maven-central/org.apache.shardingsphere/shardingsphere-jdbc/badge.svg)](https://mvnrepository.com/artifact/org.apache.shardingsphere/shardingsphere-jdbc)

定位为轻量级 Java 框架，在 Java 的 JDBC 层提供的额外服务。 它使用客户端直连数据库，以 jar 包形式提供服务，无需额外部署和依赖，可理解为增强版的 JDBC 驱动，完全兼容 JDBC 和各种 ORM 框架。

- 适用于任何基于 JDBC 的 ORM 框架，如：JPA, Hibernate, Mybatis, Spring JDBC Template 或直接使用 JDBC。
- 支持任何第三方的数据库连接池，如：DBCP, C3P0, BoneCP, Druid, HikariCP 等。
- 支持任意实现 JDBC 规范的数据库，目前支持 MySQL，Oracle，SQLServer，PostgreSQL 以及任何遵循 SQL92 标准的数据库。

[![ShardingSphere-JDBC Architecture](https://shardingsphere.apache.org/document/current/img/shardingsphere-jdbc_v3.png)](https://shardingsphere.apache.org/document/current/img/shardingsphere-jdbc_v3.png)

### ShardingSphere-Proxy

[![Download](https://img.shields.io/badge/release-download-orange.svg)](https://apache.org/dyn/closer.cgi?path=shardingsphere/5.0.0-beta/apache-shardingsphere-5.0.0-beta-shardingsphere-proxy-bin.tar.gz) [![Docker Pulls](https://img.shields.io/docker/pulls/apache/sharding-proxy.svg)](https://store.docker.com/community/images/apache/sharding-proxy)

定位为透明化的数据库代理端，提供封装了数据库二进制协议的服务端版本，用于完成对异构语言的支持。 目前提供 MySQL 和 PostgreSQL（兼容 openGauss 等基于 PostgreSQL 的数据库）版本，它可以使用任何兼容 MySQL/PostgreSQL 协议的访问客户端（如：MySQL Command Client, MySQL Workbench, Navicat 等）操作数据，对 DBA 更加友好。

- 向应用程序完全透明，可直接当做 MySQL/PostgreSQL 使用。
- 适用于任何兼容 MySQL/PostgreSQL 协议的的客户端。

[![ShardingSphere-Proxy Architecture](https://shardingsphere.apache.org/document/current/img/shardingsphere-proxy_v2.png)](https://shardingsphere.apache.org/document/current/img/shardingsphere-proxy_v2.png)

### ShardingSphere-Sidecar（TODO）

定位为 Kubernetes 的云原生数据库代理，以 Sidecar 的形式代理所有对数据库的访问。 通过无中心、零侵入的方案提供与数据库交互的啮合层，即 `Database Mesh`，又可称数据库网格。

Database Mesh 的关注重点在于如何将分布式的数据访问应用与数据库有机串联起来，它更加关注的是交互，是将杂乱无章的应用与数据库之间的交互进行有效地梳理。 使用 Database Mesh，访问数据库的应用和数据库终将形成一个巨大的网格体系，应用和数据库只需在网格体系中对号入座即可，它们都是被啮合层所治理的对象。

[![ShardingSphere-Sidecar Architecture](https://shardingsphere.apache.org/document/current/img/shardingsphere-sidecar-brief.png)](https://shardingsphere.apache.org/document/current/img/shardingsphere-sidecar-brief.png)

|            | *ShardingSphere-JDBC* | *ShardingSphere-Proxy* | *ShardingSphere-Sidecar* |
| :--------- | :-------------------- | :--------------------- | :----------------------- |
| 数据库     | 任意                  | MySQL/PostgreSQL       | MySQL/PostgreSQL         |
| 连接消耗数 | 高                    | 低                     | 高                       |
| 异构语言   | 仅 Java               | 任意                   | 任意                     |
| 性能       | 损耗低                | 损耗略高               | 损耗低                   |
| 无中心化   | 是                    | 否                     | 是                       |
| 静态入口   | 无                    | 有                     | 无                       |

### 混合架构

ShardingSphere-JDBC 采用无中心化架构，与应用程序共享资源，适用于 Java 开发的高性能的轻量级 OLTP 应用； ShardingSphere-Proxy 提供静态入口以及异构语言的支持，独立于应用程序部署，适用于 OLAP 应用以及对分片数据库进行管理和运维的场景。

Apache ShardingSphere 是多接入端共同组成的生态圈。 通过混合使用 ShardingSphere-JDBC 和 ShardingSphere-Proxy，并采用同一注册中心统一配置分片策略，能够灵活的搭建适用于各种场景的应用系统，使得架构师更加自由地调整适合与当前业务的最佳系统架构。

[![ShardingSphere Hybrid Architecture](https://shardingsphere.apache.org/document/current/img/shardingsphere-hybrid-architecture_v2.png)](https://shardingsphere.apache.org/document/current/img/shardingsphere-hybrid-architecture_v2.png)

## 解决方案

| *解决方案/功能* | *分布式数据库* | *数据安全*         | *数据库网关*         | *全链路压测* |
| :-------------- | :------------- | :----------------- | :------------------- | :----------- |
|                 | 数据分片       | 数据加密           | 异构数据库支持       | 影子库       |
|                 | 读写分离       | 行级权限（TODO）   | SQL 方言转换（TODO） | 可观测性     |
|                 | 分布式事务     | SQL 审计（TODO）   |                      |              |
|                 | 弹性伸缩       | SQL 防火墙（TODO） |                      |              |
|                 | 高可用         |                    |                      |              |

## 线路规划

# SHARDINGSPHERE-JDBC

[1. 引入 maven 依赖](https://shardingsphere.apache.org/document/5.0.0/cn/quick-start/shardingsphere-jdbc-quick-start/#1-引入-maven-依赖)[2. 规则配置](https://shardingsphere.apache.org/document/5.0.0/cn/quick-start/shardingsphere-jdbc-quick-start/#2-规则配置)[3. 创建数据源](https://shardingsphere.apache.org/document/5.0.0/cn/quick-start/shardingsphere-jdbc-quick-start/#3-创建数据源)

## 1. 引入 maven 依赖

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core</artifactId>
    <version>${latest.release.version}</version>
</dependency>
```

> 注意：请将 `${latest.release.version}` 更改为实际的版本号。

## 2. 规则配置

ShardingSphere-JDBC 可以通过 `Java`，`YAML`，`Spring 命名空间`和 `Spring Boot Starter` 这 4 种方式进行配置，开发者可根据场景选择适合的配置方式。 详情请参见[配置手册](https://shardingsphere.apache.org/document/5.0.0/cn/user-manual/shardingsphere-jdbc/configuration/)。

## 3. 创建数据源

通过 `ShardingSphereDataSourceFactory` 工厂和规则配置对象获取 `ShardingSphereDataSource`。 该对象实现自 JDBC 的标准 DataSource 接口，可用于原生 JDBC 开发，或使用 JPA, MyBatis 等 ORM 类库。

```java
DataSource dataSource = ShardingSphereDataSourceFactory.createDataSource(dataSourceMap, configurations, properties);
```

# SHARDINGSPHERE-PROXY

[1. 规则配置](https://shardingsphere.apache.org/document/5.0.0/cn/quick-start/shardingsphere-proxy-quick-start/#1-规则配置)[2. 引入依赖](https://shardingsphere.apache.org/document/5.0.0/cn/quick-start/shardingsphere-proxy-quick-start/#2-引入依赖)[3. 启动服务](https://shardingsphere.apache.org/document/5.0.0/cn/quick-start/shardingsphere-proxy-quick-start/#3-启动服务)[4. 使用ShardingSphere-Proxy](https://shardingsphere.apache.org/document/5.0.0/cn/quick-start/shardingsphere-proxy-quick-start/#4-使用shardingsphere-proxy)

## 1. 规则配置

编辑`%SHARDINGSPHERE_PROXY_HOME%/conf/config-xxx.yaml`。详情请参见[配置手册](https://shardingsphere.apache.org/document/5.0.0/cn/user-manual/shardingsphere-proxy/configuration/)。

编辑`%SHARDINGSPHERE_PROXY_HOME%/conf/server.yaml`。详情请参见[配置手册](https://shardingsphere.apache.org/document/5.0.0/cn/user-manual/shardingsphere-proxy/configuration/)。

> %SHARDINGSPHERE_PROXY_HOME% 为 Proxy 解压后的路径，例：/Users/ss/shardingsphere-proxy-bin/

## 2. 引入依赖

如果后端连接 PostgreSQL 数据库，不需要引入额外依赖。

如果后端连接 MySQL 数据库，请下载 [mysql-connector-java-5.1.47.jar](https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.47/mysql-connector-java-5.1.47.jar) 或者 [mysql-connector-java-8.0.11.jar](https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.11/mysql-connector-java-8.0.11.jar)，并将其放入 `%SHARDINGSPHERE_PROXY_HOME%/lib` 目录。

## 3. 启动服务

- 使用默认配置项

```bash
sh %SHARDINGSPHERE_PROXY_HOME%/bin/start.sh
```

默认启动端口为 `3307`，默认配置文件目录为：`%SHARDINGSPHERE_PROXY_HOME%/conf/`。

- 自定义端口和配置文件目录

```bash
sh %SHARDINGSPHERE_PROXY_HOME%/bin/start.sh ${proxy_port} ${proxy_conf_directory}
```

## 4. 使用ShardingSphere-Proxy

执行 MySQL 或 PostgreSQL的客户端命令直接操作 ShardingSphere-Proxy 即可。以 MySQL 举例：

```bash
mysql -u${proxy_username} -p${proxy_password} -h${proxy_host} -P${proxy_port}
```

# SHARDINGSPHERE-SCALING (EXPERIMENTAL)

[1. 规则配置](https://shardingsphere.apache.org/document/5.0.0/cn/quick-start/shardingsphere-scaling-quick-start/#1-规则配置)[2. 引入依赖](https://shardingsphere.apache.org/document/5.0.0/cn/quick-start/shardingsphere-scaling-quick-start/#2-引入依赖)[3. 启动服务](https://shardingsphere.apache.org/document/5.0.0/cn/quick-start/shardingsphere-scaling-quick-start/#3-启动服务)[4. 任务管理](https://shardingsphere.apache.org/document/5.0.0/cn/quick-start/shardingsphere-scaling-quick-start/#4-任务管理)[5. 相关文档](https://shardingsphere.apache.org/document/5.0.0/cn/quick-start/shardingsphere-scaling-quick-start/#5-相关文档)

## 1. 规则配置

编辑`%SHARDINGSPHERE_PROXY_HOME%/conf/server.yaml`。详情请参见[运行部署](https://shardingsphere.apache.org/document/5.0.0/cn/user-manual/shardingsphere-scaling/build/)。

> %SHARDINGSPHERE_PROXY_HOME% 为 Proxy 解压后的路径，例：/Users/ss/shardingsphere-proxy-bin/

## 2. 引入依赖

如果后端连接 PostgreSQL 数据库，不需要引入额外依赖。

如果后端连接 MySQL 数据库，请下载 [mysql-connector-java-5.1.47.jar](https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.47/mysql-connector-java-5.1.47.jar)，并将其放入 `%SHARDINGSPHERE_PROXY_HOME%/lib` 目录。

## 3. 启动服务

```bash
sh %SHARDINGSPHERE_PROXY_HOME%/bin/start.sh
```

## 4. 任务管理

通过相应的 DistSQL 接口管理迁移任务。

详情请参见[使用手册](https://shardingsphere.apache.org/document/5.0.0/cn/user-manual/shardingsphere-scaling/usage/)。

## 5. 相关文档

- [功能#弹性伸缩](https://shardingsphere.apache.org/document/5.0.0/cn/features/scaling/)：核心概念、使用规范
- [用户手册#弹性伸缩](https://shardingsphere.apache.org/document/5.0.0/cn/user-manual/shardingsphere-scaling/)：运行部署、使用手册
- [RAL#弹性伸缩](https://shardingsphere.apache.org/document/5.0.0/cn/user-manual/shardingsphere-proxy/usage/distsql/syntax/ral/ral/#弹性伸缩)：弹性伸缩的DistSQL
- [开发者手册#弹性伸缩](https://shardingsphere.apache.org/document/5.0.0/cn/dev-manual/scaling/)：SPI接口及实现类

[ShardingSphere](https://shardingsphere.apache.org/document/5.0.0/cn/) > [概念](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/) > 接入端

[ShardingSphere-JDBC](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/adaptor/#shardingsphere-jdbc)[ShardingSphere-Proxy](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/adaptor/#shardingsphere-proxy)[混合架构](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/adaptor/#混合架构)

Apache ShardingSphere 由 ShardingSphere-JDBC 和 ShardingSphere-Proxy 这 2 款既能够独立部署，又支持混合部署配合使用的产品组成。 它们均提供标准化的基于数据库作为存储节点的增量功能，可适用于如 Java 同构、异构语言、云原生等各种多样化的应用场景。

## ShardingSphere-JDBC

ShardingSphere-JDBC 是 Apache ShardingSphere 的第一个产品，也是 Apache ShardingSphere 的前身。 定位为轻量级 Java 框架，在 Java 的 JDBC 层提供的额外服务。 它使用客户端直连数据库，以 jar 包形式提供服务，无需额外部署和依赖，可理解为增强版的 JDBC 驱动，完全兼容 JDBC 和各种 ORM 框架。

- 适用于任何基于 JDBC 的 ORM 框架，如：JPA, Hibernate, Mybatis, Spring JDBC Template 或直接使用 JDBC。
- 支持任何第三方的数据库连接池，如：DBCP, C3P0, BoneCP, Druid, HikariCP 等。
- 支持任意实现 JDBC 规范的数据库，目前支持 MySQL，Oracle，SQLServer，PostgreSQL 以及任何遵循 SQL92 标准的数据库。

[![ShardingSphere-JDBC Architecture](https://shardingsphere.apache.org/document/current/img/shardingsphere-jdbc_v3.png)](https://shardingsphere.apache.org/document/current/img/shardingsphere-jdbc_v3.png)

|            | *ShardingSphere-JDBC* | *ShardingSphere-Proxy* |
| :--------- | :-------------------- | :--------------------- |
| 数据库     | `任意`                | MySQL/PostgreSQL       |
| 连接消耗数 | `高`                  | 低                     |
| 异构语言   | `仅Java`              | 任意                   |
| 性能       | `损耗低`              | 损耗略高               |
| 无中心化   | `是`                  | 否                     |
| 静态入口   | `无`                  | 有                     |

ShardingSphere-JDBC 的优势在于对 Java 应用的友好度。

## ShardingSphere-Proxy

ShardingSphere-Proxy 是 Apache ShardingSphere 的第二个产品。 它定位为透明化的数据库代理端，提供封装了数据库二进制协议的服务端版本，用于完成对异构语言的支持。 目前提供 MySQL 和 PostgreSQL（兼容 openGauss 等基于 PostgreSQL 的数据库）版本，它可以使用任何兼容 MySQL/PostgreSQL 协议的访问客户端（如：MySQL Command Client, MySQL Workbench, Navicat 等）操作数据，对 DBA 更加友好。

- 向应用程序完全透明，可直接当做 MySQL/PostgreSQL 使用。
- 适用于任何兼容 MySQL/PostgreSQL 协议的的客户端。

[![ShardingSphere-Proxy Architecture](https://shardingsphere.apache.org/document/current/img/shardingsphere-proxy_v2.png)](https://shardingsphere.apache.org/document/current/img/shardingsphere-proxy_v2.png)

|            | *ShardingSphere-JDBC* | *ShardingSphere-Proxy* |
| :--------- | :-------------------- | :--------------------- |
| 数据库     | 任意                  | `MySQL/PostgreSQL`     |
| 连接消耗数 | 高                    | `低`                   |
| 异构语言   | 仅Java                | `任意`                 |
| 性能       | 损耗低                | `损耗略高`             |
| 无中心化   | 是                    | `否`                   |
| 静态入口   | 无                    | `有`                   |

ShardingSphere-Proxy 的优势在于对异构语言的支持，以及为 DBA 提供可操作入口。

## 混合架构

ShardingSphere-JDBC 采用无中心化架构，与应用程序共享资源，适用于 Java 开发的高性能的轻量级 OLTP 应用； ShardingSphere-Proxy 提供静态入口以及异构语言的支持，独立于应用程序部署，适用于 OLAP 应用以及对分片数据库进行管理和运维的场景。

Apache ShardingSphere 是多接入端共同组成的生态圈。 通过混合使用 ShardingSphere-JDBC 和 ShardingSphere-Proxy，并采用同一注册中心统一配置分片策略，能够灵活的搭建适用于各种场景的应用系统，使得架构师更加自由地调整适合与当前业务的最佳系统架构。

[![Hybrid Architecture](https://shardingsphere.apache.org/document/current/img/shardingsphere-hybrid-architecture_v2.png)](https://shardingsphere.apache.org/document/current/img/shardingsphere-hybrid-architecture_v2.png)

[背景](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/mode/#背景)[内存模式](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/mode/#内存模式)[单机模式](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/mode/#单机模式)[集群模式](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/mode/#集群模式)

## 背景

为满足用户快速测试启动、单机运行以及集群运行等不同的需求，Apache ShardingSphere 提供了内存模式、单机模式和集群模式。

## 内存模式

适用于做快速集成测试的环境启动，方便开发人员在整合功能测试中集成 ShardingSphere。 该模式也是 Apache ShardingSphere 的默认模式。

## 单机模式

适用于单机启动 ShardingSphere，通过该模式可将数据源和规则等元数据信息持久化。 默认在根目录创建 `.shardingsphere` 文件用于存储配置信息。

## 集群模式

适用于分布式场景，它提供了多个计算节点之间的元数据共享和状态协调。 需要提供用于分布式协调的注册中心组件，如：ZooKeeper、Etcd 等。

[背景](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/distsql/#背景)[挑战](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/distsql/#挑战)[目标](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/distsql/#目标)[注意事项](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/distsql/#注意事项)

## 背景

DistSQL（Distributed SQL）是 Apache ShardingSphere 特有的内置 SQL 语言，提供了标准 SQL 之外的增量功能操作能力。

## 挑战

灵活的规则配置和资源管控能力是 Apache ShardingSphere 的特点之一。 在使用 ShardingSphere-Proxy 时，开发者虽然可以像使用数据库一样操作数据，但却需要通过 YAML 文件（或注册中心）配置资源和规则。 然而，YAML 格式的展现形式，以及注册中心动态修改带来的操作习惯变更，对于运维工程师并不友好。

DistSQL 让用户可以像操作数据库一样操作 Apache ShardingSphere，使其从面向开发人员的框架和中间件转变为面向运维人员的数据库产品。

DistSQL 细分为 RDL、RQL 和 RAL 三种类型。

- RDL（Resource & Rule Definition Language）负责资源和规则的创建、修改和删除；
- RQL（Resource & Rule Query Language）负责资源和规则的查询和展现；
- RAL（Resource & Rule Administration Language）负责 Hint、事务类型切换、分片执行计划查询等管理功能。

## 目标

**打破中间件和数据库之间的界限，让开发者像使用数据库一样使用 Apache ShardingSphere，是 DistSQL 的设计目标。**

## 注意事项

DistSQL 只能用于 ShardingSphere-Proxy，ShardingSphere-JDBC 暂不提供。

[背景](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/pluggable/#背景)[挑战](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/pluggable/#挑战)[目标](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/pluggable/#目标)[实现](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/pluggable/#实现)[L1 内核层](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/pluggable/#l1-内核层)[L2 功能层](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/pluggable/#l2-功能层)[L3 生态层](https://shardingsphere.apache.org/document/5.0.0/cn/concepts/pluggable/#l3-生态层)

## 背景

在 Apache ShardingSphere 中，很多功能实现类的加载方式是通过 [SPI（Service Provider Interface）](https://docs.oracle.com/javase/tutorial/sound/SPI-intro.html) 注入的方式完成的。 SPI 是一种为了被第三方实现或扩展的 API，它可以用于实现框架扩展或组件替换。

## 挑战

可插拔架构对程序架构设计的要求非常高，需要将各个模块相互独立，互不感知，并且通过一个可插拔内核，以叠加的方式将各种功能组合使用。 设计一套将功能开发完全隔离的架构体系，既可以最大限度的将开源社区的活力激发出来，也能够保障项目的质量。

Apache ShardingSphere 5.x 版本开始致力于可插拔架构，项目的功能组件能够灵活的以可插拔的方式进行扩展。 目前，数据分片、读写分离、数据库高可用、数据加密、影子库压测等功能，以及对 MySQL、PostgreSQL、SQLServer、Oracle 等 SQL 与协议的支持，均通过插件的方式织入项目。 Apache ShardingSphere 目前已提供数十个 SPI 作为系统的扩展点，而且仍在不断增加中。

## 目标

**让开发者能够像使用积木一样定制属于自己的独特系统，是 Apache ShardingSphere 可插拔架构的设计目标。**

[![Pluggable Platform](https://shardingsphere.apache.org/document/current/img/pluggable_platform.png)](https://shardingsphere.apache.org/document/current/img/pluggable_platform.png)

## 实现

Apache ShardingSphere 的可插拔架构划分为 3 层，它们是：L1 内核层、L2 功能层、L3 生态层。

### L1 内核层

是数据库基本能力的抽象，其所有组件均必须存在，但具体实现方式可通过可插拔的方式更换。 主要包括查询优化器、分布式事务引擎、分布式执行引擎、权限引擎和调度引擎等。

### L2 功能层

用于提供增量能力，其所有组件均是可选的，可以包含零至多个组件。组件之间完全隔离，互无感知，多组件可通过叠加的方式相互配合使用。 主要包括数据分片、读写分离、数据库高可用、数据加密、影子库等。用户自定义功能可完全面向 Apache ShardingSphere 定义的顶层接口进行定制化扩展，而无需改动内核代码。

### L3 生态层

用于对接和融入现有数据库生态，包括数据库协议、SQL 解析器和存储适配器，分别对应于 Apache ShardingSphere 以数据库协议提供服务的方式、SQL 方言操作数据的方式以及对接存储节点的数据库类型。

[背景](https://shardingsphere.apache.org/document/5.0.0/cn/features/db-compatibility/#背景)[挑战](https://shardingsphere.apache.org/document/5.0.0/cn/features/db-compatibility/#挑战)[目标](https://shardingsphere.apache.org/document/5.0.0/cn/features/db-compatibility/#目标)

## 背景

随着通信技术的革新，全新领域的应用层出不穷，推动和颠覆整个人类社会协作模式的革新。 数据存量随着应用的探索不断增加，数据的存储和计算模式无时无刻面临着创新。

面向交易、大数据、关联分析、物联网等场景越来越细分，单一数据库再也无法适用于所有的应用场景。 与此同时，场景内部也愈加细化，相似场景使用不同数据库已成为常态。 由此可见，数据库碎片化的趋势已经不可逆转。

## 挑战

并无统一标准的数据库的访问协议和 SQL 方言，以及各种数据库带来的不同运维方法和监控工具的异同，让开发者的学习成本和 DBA 的运维成本不断增加。 提升与原有数据库兼容度，是在其之上提供增量服务的前提。

SQL 方言和数据库协议的兼容，是数据库兼容度提升的关键点。

## 目标

**尽量多的兼容各种数据库，让用户零使用成本，是 Apache ShardingSphere 数据库兼容度希望达成的主要目标。**

[背景](https://shardingsphere.apache.org/document/5.0.0/cn/features/sharding/#背景)[垂直分片](https://shardingsphere.apache.org/document/5.0.0/cn/features/sharding/#垂直分片)[水平分片](https://shardingsphere.apache.org/document/5.0.0/cn/features/sharding/#水平分片)[挑战](https://shardingsphere.apache.org/document/5.0.0/cn/features/sharding/#挑战)[目标](https://shardingsphere.apache.org/document/5.0.0/cn/features/sharding/#目标)

## 背景

传统的将数据集中存储至单一节点的解决方案，在性能、可用性和运维成本这三方面已经难于满足海量数据的场景。

从性能方面来说，由于关系型数据库大多采用 B+ 树类型的索引，在数据量超过阈值的情况下，索引深度的增加也将使得磁盘访问的 IO 次数增加，进而导致查询性能的下降； 同时，高并发访问请求也使得集中式数据库成为系统的最大瓶颈。

从可用性的方面来讲，服务化的无状态型，能够达到较小成本的随意扩容，这必然导致系统的最终压力都落在数据库之上。 而单一的数据节点，或者简单的主从架构，已经越来越难以承担。数据库的可用性，已成为整个系统的关键。

从运维成本方面考虑，当一个数据库实例中的数据达到阈值以上，对于 DBA 的运维压力就会增大。 数据备份和恢复的时间成本都将随着数据量的大小而愈发不可控。一般来讲，单一数据库实例的数据的阈值在 1TB 之内，是比较合理的范围。

在传统的关系型数据库无法满足互联网场景需要的情况下，将数据存储至原生支持分布式的 NoSQL 的尝试越来越多。 但 NoSQL 对 SQL 的不兼容性以及生态圈的不完善，使得它们在与关系型数据库的博弈中始终无法完成致命一击，而关系型数据库的地位却依然不可撼动。

数据分片指按照某个维度将存放在单一数据库中的数据分散地存放至多个数据库或表中以达到提升性能瓶颈以及可用性的效果。 数据分片的有效手段是对关系型数据库进行分库和分表。分库和分表均可以有效的避免由数据量超过可承受阈值而产生的查询瓶颈。 除此之外，分库还能够用于有效的分散对数据库单点的访问量；分表虽然无法缓解数据库压力，但却能够提供尽量将分布式事务转化为本地事务的可能，一旦涉及到跨库的更新操作，分布式事务往往会使问题变得复杂。 使用多主多从的分片方式，可以有效的避免数据单点，从而提升数据架构的可用性。

通过分库和分表进行数据的拆分来使得各个表的数据量保持在阈值以下，以及对流量进行疏导应对高访问量，是应对高并发和海量数据系统的有效手段。 数据分片的拆分方式又分为垂直分片和水平分片。

### 垂直分片

按照业务拆分的方式称为垂直分片，又称为纵向拆分，它的核心理念是专库专用。 在拆分之前，一个数据库由多个数据表构成，每个表对应着不同的业务。而拆分之后，则是按照业务将表进行归类，分布到不同的数据库中，从而将压力分散至不同的数据库。 下图展示了根据业务需要，将用户表和订单表垂直分片到不同的数据库的方案。

[![垂直分片](https://shardingsphere.apache.org/document/current/img/sharding/vertical_sharding.png)](https://shardingsphere.apache.org/document/current/img/sharding/vertical_sharding.png)

垂直分片往往需要对架构和设计进行调整。通常来讲，是来不及应对互联网业务需求快速变化的；而且，它也并无法真正的解决单点瓶颈。 垂直拆分可以缓解数据量和访问量带来的问题，但无法根治。如果垂直拆分之后，表中的数据量依然超过单节点所能承载的阈值，则需要水平分片来进一步处理。

### 水平分片

水平分片又称为横向拆分。 相对于垂直分片，它不再将数据根据业务逻辑分类，而是通过某个字段（或某几个字段），根据某种规则将数据分散至多个库或表中，每个分片仅包含数据的一部分。 例如：根据主键分片，偶数主键的记录放入 0 库（或表），奇数主键的记录放入 1 库（或表），如下图所示。

[![水平分片](https://shardingsphere.apache.org/document/current/img/sharding/horizontal_sharding.png)](https://shardingsphere.apache.org/document/current/img/sharding/horizontal_sharding.png)

水平分片从理论上突破了单机数据量处理的瓶颈，并且扩展相对自由，是数据分片的标准解决方案。

## 挑战

虽然数据分片解决了性能、可用性以及单点备份恢复等问题，但分布式的架构在获得了收益的同时，也引入了新的问题。

面对如此散乱的分片之后的数据，应用开发工程师和数据库管理员对数据库的操作变得异常繁重就是其中的重要挑战之一。 他们需要知道数据需要从哪个具体的数据库的子表中获取。

另一个挑战则是，能够正确的运行在单节点数据库中的 SQL，在分片之后的数据库中并不一定能够正确运行。 例如，分表导致表名称的修改，或者分页、排序、聚合分组等操作的不正确处理。

跨库事务也是分布式的数据库集群要面对的棘手事情。 合理采用分表，可以在降低单表数据量的情况下，尽量使用本地事务，善于使用同库不同表可有效避免分布式事务带来的麻烦。 在不能避免跨库事务的场景，有些业务仍然需要保持事务的一致性。 而基于 XA 的分布式事务由于在并发度高的场景中性能无法满足需要，并未被互联网巨头大规模使用，他们大多采用最终一致性的柔性事务代替强一致事务。

## 目标

**尽量透明化分库分表所带来的影响，让使用方尽量像使用一个数据库一样使用水平分片之后的数据库集群，是 Apache ShardingSphere 数据分片模块的主要设计目标。**

# 表

[逻辑表](https://shardingsphere.apache.org/document/5.0.0/cn/features/sharding/concept/table/#逻辑表)[真实表](https://shardingsphere.apache.org/document/5.0.0/cn/features/sharding/concept/table/#真实表)[绑定表](https://shardingsphere.apache.org/document/5.0.0/cn/features/sharding/concept/table/#绑定表)[广播表](https://shardingsphere.apache.org/document/5.0.0/cn/features/sharding/concept/table/#广播表)[单表](https://shardingsphere.apache.org/document/5.0.0/cn/features/sharding/concept/table/#单表)

表是透明化数据分片的关键概念。 Apache ShardingSphere 通过提供多样化的表类型，适配不同场景下的数据分片需求。

## 逻辑表

相同结构的水平拆分数据库（表）的逻辑名称，是 SQL 中表的逻辑标识。 例：订单数据根据主键尾数拆分为 10 张表，分别是 `t_order_0` 到 `t_order_9`，他们的逻辑表名为 `t_order`。

## 真实表

在水平拆分的数据库中真实存在的物理表。 即上个示例中的 `t_order_0` 到 `t_order_9`。

## 绑定表

指分片规则一致的主表和子表。 例如：`t_order` 表和 `t_order_item` 表，均按照 `order_id` 分片，则此两张表互为绑定表关系。 绑定表之间的多表关联查询不会出现笛卡尔积关联，关联查询效率将大大提升。 举例说明，如果 SQL 为：

```sql
SELECT i.* FROM t_order o JOIN t_order_item i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);
```

在不配置绑定表关系时，假设分片键 `order_id` 将数值 10 路由至第 0 片，将数值 11 路由至第 1 片，那么路由后的 SQL 应该为 4 条，它们呈现为笛卡尔积：

```sql
SELECT i.* FROM t_order_0 o JOIN t_order_item_0 i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);

SELECT i.* FROM t_order_0 o JOIN t_order_item_1 i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);

SELECT i.* FROM t_order_1 o JOIN t_order_item_0 i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);

SELECT i.* FROM t_order_1 o JOIN t_order_item_1 i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);
```

在配置绑定表关系后，路由的 SQL 应该为 2 条：

```sql
SELECT i.* FROM t_order_0 o JOIN t_order_item_0 i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);

SELECT i.* FROM t_order_1 o JOIN t_order_item_1 i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);
```

其中 `t_order` 在 FROM 的最左侧，ShardingSphere 将会以它作为整个绑定表的主表。 所有路由计算将会只使用主表的策略，那么 `t_order_item` 表的分片计算将会使用 `t_order` 的条件。 因此，绑定表间的分区键需要完全相同。

## 广播表

指所有的分片数据源中都存在的表，表结构及其数据在每个数据库中均完全一致。 适用于数据量不大且需要与海量数据的表进行关联查询的场景，例如：字典表。

## 单表

指所有的分片数据源中仅唯一存在的表。 适用于数据量不大且无需分片的表。





