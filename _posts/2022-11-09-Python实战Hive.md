---
layout: post
categories: [Python,Hive]
description: none
keywords: Python
---
# Python实战Hive
PyHive 是 Python 语言编写的用于操作 Hive 的简便工具库。

## PyHive安装
```
# Liunx系统
pip install sasl
pip install thrift
pip install thrift-sasl
pip install PyHive

# Windows系统会出现莫名其妙的报错,sasl需要选择对应的版本号
pip install sasl‑0.3.1‑cp310‑cp310‑win_amd64.whl
或者下载到本地手动导入包
pip install D:\sasl-0.3.1-cp310-cp310-win_amd64.whl

优化后导入如下：
pip install sasl-0.2.1-cp36-cp36m-win_amd64.whl
pip install thrift -i https://pypi.tuna.tsinghua.edu.cn/simple
pip install thrift_sasl==0.3.0 -i https://pypi.tuna.tsinghua.edu.cn/simple
pip install pyhive -i https://pypi.tuna.tsinghua.edu.cn/simple
```
下载一个对应你所使用的Python版本和Windows版本的sasl文件：https://www.lfd.uci.edu/~gohlke/pythonlibs/#sasl。
例如， sasl‑0.2.1‑cp36‑cp36m‑win_amd64.whl，它对应的Python 版本为3.6，对应的Windows系统为64位。 安装执行pip install sasl-0.2.1-cp37-cp37m-win_amd64.whl

## 简介
pyhive通过与HiveServer2通讯来操作Hive数据。当hiveserver2服务启动后，会开启10000的端口，对外提供服务，此时pyhive客户端通过JDBC连接hiveserver2进行Hive sql操作。

Pyhive Client通过JDBC与hiveserver2建立通信，hiveserver2服务端发送HQL语句到Driver端，Driver端将HQL发送至Compiler组件进行语法树解析，此时需在metastore获取HQL相关的database和table等信息，在对HQL完成解析后，Compiler组件发送执行计划至Driver端等待处理，Driver端发送执行计划至Executor端，再由Executor端发送MapReduce任务至Hadoop集群执行Job，Job完成后最终将HQL查询数据发送Driver端，再由hive server2返回数据至pyhive Client。

## 访问
PyHive 连接 Hive 一般流程：
- 创建连接
- 获取游标
- 执行SQL语句
- 获取结果
- 关闭连接

```python
# 加载包
from pyhive import hive

# 建立连接
conn = hive.connect(host = '10.8.1.2',      # 主机
                    port = 10000,                  # 端口 
                    username = 'hdfs',                  # 用户
                    )
                    
# 查询
cursor = conn.cursor()  # 获取一个游标
cursor.execute('show databases')  # 执行sql语句
# sql = 'show tables'  # 操作语句
for result in cursor.fetchall():  # 输出获取结果的所有行
    print(result)
 
# 关闭连接
cursor.close()
conn.close()

```
其中，cursor.fetchall() 返回的是一个 list 对象，并且每一个元素都是一个 tuple 对象。 需要对其进行一定的处理，才能转换为建模需要的 DataFrame。

## 函数封装
```python
# 函数封装
def get_data(params, sql_text, is_all_col=1):
    '''
    is_all_col: 是否选取所有的列 select *
    '''
    # 建立连接
    con = hive.connect(host = params.get('ip'),
                       port = params.get('port'),
                       auth = params.get('auth'),
                       kerberos_service_name = params.get('kerberos_service_name'),
                       database = params.get('database'),
                       password = params.get('password'))
    cursor = con.cursor()
    cursor.execute(sql_text)
    # 列名
    col_tmp = cursor.description
    col = list()
    if is_all_col == 1:
        for i in range(len(col_tmp)):
            col.append(col_tmp[i][0].split('.')[1])
    else:
        for i in range(len(col_tmp)):
            col.append(col_tmp[i][0])
    # 数据
    data = cursor.fetchall()
    result = pd.DataFrame(list(data), columns=col)
    # 关闭连接 释放资源
    cursor.close()
    con.close()
    
    return result

if __name__ == '__main__':

    import pandas as pd
    import numpy as np
    
    params = {
        'ip': 'xxx.xxx.xxx.xxx',
        'port': '1000',
        'auth': 'hider',
        'kerberos_service_name': 'hive',
        'database': 'hive',
        'password': '100',
    }
    
    sql_text1 = 'select * from table limit 5'
    sql_text2 = 'select a, b, c from table limit 5'
    
    # 所有列
    data1 = get_data(params, sql_text1, is_all_col=1)
    # 指定列
    data2 = get_data(params, sql_text2, is_all_col=0)
```

## Hive配置
Hive 有许多必要的参数设置，通过 Connection 类的 configuration 参数可进行配置。
```
hive_config = {
    'mapreduce.job.queuename': 'my_hive',
    'hive.exec.compress.output': 'false',
    'hive.exec.compress.intermediate': 'true',
    'mapred.min.split.size.per.node': '1',
    'mapred.min.split.size.per.rack': '1',
    'hive.map.aggr': 'true',
    'hive.groupby.skewindata': 'true',
    'hive.exec.dynamic.partition.mode': 'nonstrict'
}

conn = hive.connect(host = '',
                    port = '',
                    ...
                    configuration = hive_config)
```

## 列名
Cursor 类中有一个 description 方法，可以获取数据表中的列名、数据类型等信息。
```
col = cursor.description
col_names = list()
for column in col:
	col_names.append(column[0]) # 提取第一个元素：列名
```

## 执行脚本带参数
游标所执行的脚本可以不写死，通过参数的方式进行配置。
```
month = 202205

# %占位符
sql_text = 'select * from table where month = %d limit 5' % month
print(sql_text)

# format
sql_text = 'select * from table where month = {} limit 5'.format(month)

# f-string
sql_text = f'select * from table where month = {month} limit 5'
print(sql_text)
```









