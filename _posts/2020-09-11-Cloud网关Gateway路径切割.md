---
layout: post
categories: [Gateway]
description: none
keywords: Gateway
---
# Cloud网关Gateway路径切割


## Spring Cloud Gateway 路径重写正则表达式的理解 (RewritePath GatewayFilter Factory)

首先
官网对于 RewritePath GatewayFilter Factory 的解释是这样的：
```
spring:
  cloud:
    gateway:
      routes:
      - id: rewritepath_route
        uri: https://example.org
        predicates:
        - Path=/red/**
        filters:
        - RewritePath=/red(?<segment>/?.*), $\{segment}

```
对于请求路径 /red/blue，当前的配置在请求到到达前会被重写为 /blue，由于YAML的语法问题，$符号后面应该加上\

然后
在解释正则表达式前，我们需要学习一下java正则表达式分组的两个概念：

命名分组：(?<name>capturing text)
将匹配的子字符串捕获到一个组名称中，后面可通过分组名获得匹配结果。例如这里的示例，就是将 capturing text 捕获到名称为 name 的组中

引用捕获文本：${name}
将名称为name的命名分组对应的内容替换到此处

那么就很好解释官网的这个例子了，
对于配置文件中的： - RewritePath=/red(?<segment>/?.*), $\{segment}详解：

(?<segment>/?.*)：
?<segment>
名称为 segment 的组
/?
字符/出现0次或1次
.*
任意字符出现0次或多次
合起来就是：将 /?.*匹配到的结果捕获到名称为segment的组中

$\{segment}：
将名称为 segment 的分组捕获到的文本置换到此处。
注意，\的出现是由于避免 yaml 语法认为这是一个变量（因为在 yaml 中变量的表示法为 ${variable}，而这里我们想表达的是字面含义），在 gateway 进行解析时，会被替换为 ${segment}

最后
业务举例：
将 https://spring.io/projects/** 这个路径重写为 https://spring.io/regexp/**
```
spring:
  cloud:
    gateway:
      routes:
      - id: rewritepath_route
        uri: https://spring.io
        predicates:
        - Path=/projects/**
        filters:
        - RewritePath=/projects(?<segment>/?.*), /regexp$\{segment}
```