---
layout: post
categories: [Python]
description: none
keywords: Python
---
# Python访问Redis

## 
Python想要访问Redis数据库，必须要安装访问Redis数据库所需要的第三方库redis-py。可以使用pip命令安装Redis库。
```
pip install -U redis==2.10.6
```

连接Redis数据库服务器

redis-py库提供了两个类Redis和StrictRedis来实现Redis的命令操作。StrictRedis实现了大部分官方的语法和命令，而Redis是StrictRedis的子类，用于向后兼容旧版本的redis-py。这里使用官方推荐的StrictRedis类实现相关操作。

已知Redis服务器已安装在本地，端口是6379，密码为foobared。使用StrictRedis类连接Redis数据库的实现代码如下：
```
import redis                                            #导入redis模块
#host是redis主机，端口是6379，数据库索引为0，密码为foobared
r = redis.StrictRedis(host='localhost', port=6379,db=0,password="foobared")
#将键值对存入redis缓存，key是"name"，value是"cathy"
r.set('name', "cathy")
#取出键name对应的值
print(r['name'])
print(r.get('name'))
```

首先，导入redis模块，再通过redis.StrictRedis方法生成一个StrictRedis对象，传递的参数有：
- host：Redis服务器地址。
- port：端口，默认为6379。
- db：数据库索引，默认为0。
- password：密码。

然后，调用set()方法设置一个键值对数据。最后，使用[key]或者get方法获取数据。

运行后的结果如下：
```
b'cathy'
b'cathy'
```

在Python 3中，通过get()方法获取的值是一个字节类型。如果想设置为字符串类型，可在初始化StrictRedis对象时，设置参数decode_responses=True（默认为False，即Bytes型）。

还可以使用连接池（connection pool）来连接Redis数据库。连接池用来管理对Redis服务器的所有连接，避免每次建立、释放连接的开销。

下面是使用连接池连接Redis数据库的代码：
```
import redis                                                    #导入redis模块
pool = redis.ConnectionPool(host='localhost',port=6379,password="foobared",
decode_responses=True)
r = redis.Redis(connection_pool=pool)
r.set('name', 'cathy')
print(r.get('name'))
```

运行后的结果如下：
```
cathy
```

字符串（String）操作

字符串是Redis中最基本的键值对存储形式。它在Redis中是二进制安全的，这意味着它可以接受任何格式的数据，如JPEG图像数据或JSON信息等。在Redis中，字符串最大可容纳的数据长度为512MB。下面使用字符串来存储cathy的个人信息，实现代码如下：
```
import redis                                                                    #导入redis模块
#生成StrictRedis对象
r = redis.StrictRedis(host='localhost',                 #主机
                    port=6379,                                  #端口
                    db=0,                                       #数据库索引
                    password="foobared",                #密码
                    decode_responses=True)              #设置解码
r.set('name', "cathy")                          #将值为"cathy"的字符串赋给键name
r.set("age",10)                                                         #将10赋给age键
r.setnx("height",1.50)                          #如果键height不存在，则赋给值1.50
r.mset({"score1":100,"score2":98})                              #批量设置
r.get("name")                                                           #获取键为name的值
r.mget(["name","age"])                                                  #批量获取键为name和age的值
r.append("name","good")                                                 #向键为name的值后追加good
print(r.mget(["name","age","height","score1","score2"]))
```

运行结果如下：
```
['cathygood', '10', '1.5', '100', '98']
```
对键的赋值不仅限于字符串，也可以是整型、浮点型等其他类型的数据，最后都会转换为字符串形式。

列表（List）操作

Redis中的列表是一个双向链表，可以在链表左右分别操作，即支持双向存储。有时也把列表看成一个队列，实现先进先出的功能，所以很多时候将Redis用作消息队列。后面章节介绍的分布式爬虫框架，默认就是使用Redis的列表存储爬虫数据的。下面使用列表来存储cathy的个人信息，实现代码如下：
```
import redis                                                            #导入redis模块
#生成StrictRedis对象
r = redis.StrictRedis(host='localhost',         #主机
                    port=6379,                          #端口
                    db=0,                               #数据库索引
                    password="foobared",        #密码
                    decode_responses=True)      #解析形式：字符串
r.lpush("student","cathy",10)   #向键为student的列表头部添加值"cathy"和10
r.rpush("student",1.50, "女")    #向键为student的列表尾部添加值身高和性别
print(r.lrange("student",0,3))  #获取列表student中索引范围是0~3的列表
r.lset("student",1,9)                   #向键为student中索引为1的位置赋值9
r.lpop("student")                       #返回并删除列表student中的首元素
r.rpop("student")                       #返回并删除列表student中的尾元素
r.llen("student")                       #获取student列表长度
print(r.lrange("student",0,-1))                         #获取列表student中的所有数据
```

在使用上述方法向列表中添加数据时，如果列表不存在，则会创建一个空列表。添加的数据类型可以是bytes、string和number。运行结果如下：
```
['10', 'cathy', '1.5', '女']
['9', '1.5']
```

无序集合（Set）操作

Redis的Set是由非重复的字符串元素组成的无序集合。后面章节介绍的分布式爬虫框架，默认就是使用Redis的无序集合存储网站请求的指纹（经过加密形成的唯一识别码），实现爬虫的去重功能。下面使用集合来存储cathy的个人信息，实现代码如下：
```
import redis #导入redis模块
#生成StrictRedis对象
r = redis.StrictRedis(host='localhost',         #主机
                    port=6379,                          #端口
                    db=0,                               #数据库索引
                    password="foobared",        #密码
                    decode_responses=True)      #解析形式：字符串
#将"cathy"，"tom"，"terry"，"lili"，"tom"5个元素添加到键为names的集合中
r.sadd("names","cathy","tom","terry","lili","tom") 
r.scard("names")                                #获取键为names的集合中的元素个数，结果为4
r.srem("names","tom")                           #从键为names的集合中删除"tom"
r.spop("names")                                 #从键为names的集合中随机删除并返回该元素
#将"terry"从键为names的集合中转移到键为names1的集合中
r.smove("names","names1","terry") 
r.sismember("names","cathy")            #判断"cathy"是否是键为names的集合中的元
                                                                 素，结果为True
r.srandmember("names")                          #随机获取键为names的集合中的一个元素
print(r.smembers("names"))              #获取键为names的集合中的所有元素
```

首先使用sadd()方法添加了5个元素到键为names的集合中，但是通过scard()方法发现集合中元素的个数只有4个，这是因为添加的5个元素中，有两个是重复的，而集合是不允许有重复元素的。运行结果如下：
```
{'lili', 'cathy'}
```
如果运行多次，会发现显示的结果可能会不一样，这是因为spop()方法会随机删除元素，再加上集合的无序性，每次显示的顺序也未必相同。

散列表（Hash）操作

Redis的散列表可以看成是具有key-value键值对的map容器。Redis的key-value结构中，value也可以存储散列表，而key可以理解为散列表的表名。散列表特别适合存储对象信息。例如，使用散列表存储同学cathy的个人信息
```
import redis                                                                    #导入redis模块
#生成StrictRedis对象
r = redis.StrictRedis(host='localhost',         #主机
                    port=6379,                                  #端口
                    db=0,                                       #数据库索引
                    password="foobared",                #密码
                    decode_responses=True)              #解析形式：字符串
#将key为name，value为cathy的键值对添加到键为stu散列表中
r.hset("stu","name","cathy")
r.hmset("stu",{"age":10,"height":1.50})                 #批量添加键值对
r.hsetnx("stu","score",100)             #如果score=100的键值对不存在，则添加
r.hget("stu","name")                            #获取散列表中key为name的值
r.hmget("stu",["name","age"])           #获取散列表中多个key对应的值
r.hexists("stu","name")                 #判断key为name的值是否存在，此处为True
r.hdel("stu","score")                           #删除key为score的键值对
r.hlen("stu")                           #获取散列表中键值对个数
r.hkeys("stu")                                  #获取散列表中所有的key
```

有序集合（Sorted Set）操作

与无序集合（Set）一样，有序集合也是由非重复的字符串元素组成的。为了实现对集合中元素的排序，有序集合中每个元素都有一个与其关联的浮点值，称为“分数”。有序集合中的元素按照以下规则进行排序。

（1）如果元素A和元素B的“分数”不同，则按“分数”的大小排序。

（2）如果元素A和元素B的“分数”相同，则按元素A和元素B在字典中的排序排列。

下面来看一个例子，将一些黑客名字添加到有序集合中，出生年份为其“分数”。
```
import redis                                                            #导入redis模块
#生成StrictRedis对象
r = redis.StrictRedis(host='localhost',         #主机
                    port=6379,                          #端口
                    db=0,                               #数据库索引
                    password="foobared",        #密码
                    decode_responses=True)      #解析形式：字符串
#将"Alan Kay"（分数为1940）添加到键为hackers的有序集合中
r.zadd("hackers",{"Alan Kay":1940})
r.zadd("hackers",{"Sophie Wilson":1957,"Richard Stallman":1953}) #批量添加
r.zadd("hackers",{"Anita Borg":1953})
r.zadd("hackers",{"Hedy Lamarr":1914})
print(r.zrank("hackers","Alan Kay"))            #获取"Alan Kay"在有序集合中的位
                                                                         置（从0开始）
print(r.zcard("hackers"))                               #获取有序集合中元素个数
print(r.zrange("hackers",0,-1))                         #获取有序集合中所有元素，默认
                                                                         按score从小到大排序
print(r.zrevrange("hackers",0,-1))              #按score从大到小顺序获取所有元素
print(r.zrangebyscore("hackers",1900,1950))     #获取score为1900~1950之间的所
                                                                         有元素
```
在redis-py3.0之前，添加有序元素的方法是zadd(REDIS_KEY,score,value)，而在3.0之后，添加有序元素的方法就变为了zadd(REDIS_KEY,{value:score})。

运行结果如下：
```
1
5
['Hedy Lamarr', 'Alan Kay', 'Anita Borg', 'Richard Stallman', 'Sophie
Wilson']
['Sophie Wilson', 'Richard Stallman', 'Anita Borg', 'Alan Kay', 'Hedy
Lamarr']
['Hedy Lamarr', 'Alan Kay']
```








