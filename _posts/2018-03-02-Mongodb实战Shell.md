---
layout: post
categories: [Mongodb]
description: none
keywords: Mongodb
---
# MongoDB实战Shell


## 访问MongoDB HTTP接口
MongoDB内置了一个HTTP接口，可向您提供有关MongoDB服务器的信息。HTTP接口提供了有关MongoDB服务器的状态信息，还提供了一个REST接口，让您能够通过REST调用来访问数据库。

在大多数情况下，您都将在应用程序中使用编程语言专用的驱动程序来访问MongoDB数据库。然而，使用HTTP接口通常有助于获悉如下信息。
- 版本。
- 数据库个数。
- 活动游标数。
- 复制信息。
- 客户端信息，包括锁和查询。
- DB日志视图。
  要访问MongoDB HTTP接口，可访问该接口的端口28017。

## 从MongoDB shell访问MongoDB
安装、配置并启动MongoDB后，便可通过MongoDB shell访问它了。MongoDB shell是MongoDB自带的一个交互式JavaScript shell，让您能够访问、配置和管理MongoDB数据库、用户等。使用这个shell可执行各种任务，从设置用户账户到创建数据库，再到查询数据库内容，无所不包。

### MongoDB shell命令
MongoDB shell提供了多个命令，您可在shell提示符下执行它们。
- help <option>  显示MongoDB shell命令的语法帮助。参数option让您能够指定需要哪方面的帮助，如db、collection或cursor
- use <database>  修改当前数据库句柄。数据库操作是在当前数据库句柄上进行的
- show <option>  根据参数option显示一个列表。参数option的可能取值如下。 dbs：显示数据库列表 collections：显示当前数据库中的集合列表 users：显示当前数据库中的用户列表 profile：显示system.profile中时间超过1毫秒的条目
- log [name]  显示内存中日志的最后一部分。如果没有指定日志名，则默认为global
- exit  退出MongoDB shell

### MongoDB shell原生方法和构造函数
- Date()   创建一个Date对象。默认情况下，创建一个包含当前日期的Date对象
- UUID(hex_string)     将32字节的十六进制字符串转换为BSON子类型UUID
- ObjectId.valueOf()  将一个ObjectId的属性str显示为十六进制字符串
- Mongo.getDB(database)  返回一个数据库对象，它表示指定的数据库
- Mongo(host:port)  创建一个连接对象，它连接到指定的主机和端口
- connect(string)  连接到指定MongoDB实例中的指定数据库。返回一个数据库对象。连接字符串的格式如下：host:port/database，如db = connect("localhost:28001/myDb")
- cat(path)  返回指定文件的内容
- version()  返回当前MongoDB shell实例的版本
- cd(path)  将工作目录切换到指定路径
- getMemInfo()  返回一个文档，指出了MongoDB shell当前占用的内存量
- hostname()  返回运行MongoDB shell的系统的主机名
- _isWindows()  如果MongoDB shell运行在Windows系统上，就返回true；如果运行在UNIX或Linux系统上，就返回false
- load(path)  在MongoDB shell中加载并运行参数path指定的JavaScript文件
- _rand()  返回一个0～1的随机数
