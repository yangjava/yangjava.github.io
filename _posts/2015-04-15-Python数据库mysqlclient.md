---
layout: post
categories: [Python]
description: none
keywords: Python
---
# Python数据库mysqlclient

## 
使用pip命令安装mysqlclient。
```
pip install mysqlclient
```
如果使用pip安装出错，可以转而使用conda install mysqlclient命令安装，或者下载mysqlclient库，在定位到mysqlclient库所在目录后，重新使用pip命令安装即可。

mysqlclient库的下载地址为https://www.lfd.uci.edu/~gohlke/pythonlibs/#mysql-python。

在Python中，有了mysqlclient库，操作MySQL数据库就变得非常简单了。

## 操作MySQL数据库
下面来看一下流程中各个功能的实现方法。
连接MySQL数据库服务器

调用方法MySQLdb.connect(db,host,user,password,charset)。对应的参数有：

- db：数据库名；
- host：主机；
- user：用户名；
- password：密码；
- charset：编码格式。

以下代码实现了连接MySQL的名为qidian的数据库。
```
import MySQLdb                                          #导入MySQLdb模块
db_conn = MySQLdb.connect(db="qidian",host="localhost",user="root",
password="1234",charset="utf8")
```
db_conn是返回的Connection对象。


获取操作游标

调用Connection对象的cursor()方法，获取操作游标，实现代码如下：
```
db_cursor = db_conn.cursor()
```
db_cursor是返回的Cursor对象，用于执行SQL语句。


执行SQL语句

调用Cursor对象的execute()方法，执行SQL语句，实现对数据库的增、删、改、查操作，代码如下：
```
#新增数据
sql='insert into hot(name,author,type,form)values("道君","未知","仙侠","连载")'
db_cursor.execute(sql)
#修改数据
sql='update hot set author = "跃千愁" where name="道君"'
db_cursor.execute(sql)
#查询表hot中type为仙侠的数据
sql='select * from hot where type="仙侠"'
db_cursor.execute(sql)
#删除表中type为仙侠的数据
sql='delete from hot where type="仙侠"'
db_cursor.execute(sql)
```

回滚

在对数据库执行更新操作（update、insert和delete）的过程中，如果遇到错误，可以使用rollback()方法将数据恢复到更新前的状态。这就是所谓的原子性，即要么完整地被执行，要么完全不执行，实现代码如下：
```
db_conn.rollback()             #回滚操作
```
需要注意的是，回滚操作一定要在commit()方法之前执行，否则就无法恢复了。


提交数据

调用Connection对象的commit()方法实现数据的提交。前面虽然通过execute()方法执行SQL语句完成了对数据库的更新操作，但是并未真正更新到数据库中，需要通过commit()方法实现对数据库的永久修改，实现代码如下：
```
db_conn.commit()
```

关闭游标及数据库

当执行完对数据库的所有操作后，不要忘了关闭游标和数据库对象，实现代码如下：
```
db_cursor.close()       #关闭游标
db_conn.close()         #关闭数据库
```















