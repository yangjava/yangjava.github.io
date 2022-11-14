---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
#SpringBoot日志管理
会当凌绝顶，一览众山小。——杜甫《望岳》
在学习这块时需要一些日志框架的发展和基础，同时了解日志配置时考虑的因素。

## 配置时考虑点

在配置日志时需要考虑哪些因素？
- 支持日志路径，日志level等配置 
- 日志控制配置通过application.yml下发 
- 按天生成日志，当天的日志>50MB回滚 
- 最多保存10天日志 生成的日志中Pattern自定义 
- Pattern中添加用户自定义的MDC字段，比如用户信息(当前日志是由哪个用户的请求产生)，request信息。此种方式可以通过AOP切面控制，在MDC中添加requestID，在spring-logback.xml中配置Pattern。 
- 根据不同的运行环境设置Profile - dev，test，product 
- 对控制台，Err和全量日志分别配置 
- 对第三方包路径日志控制