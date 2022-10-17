# ElasticSearch

## 概述

官网：https://www.elastic.co/cn/products/elasticsearch

Elasticsearch 是一个采用Restful API标准的**分布式**、**可扩展**、**近实时的**搜索与数据分析引擎。

**elasticsearch与关系型数据库的对比**

| 关系型数据库（MySQL） | 非关系型数据库 |
| --------------------- | -------------- |
| 数据库database        | 索引 index     |
| 表 table              | 类型 type      |
| 数据行 row            | 文档 document  |
| 数据列 column         | 字段 field     |

### 基本概念

#### Index

类似于mysql数据库中的database

#### Type

类似于mysql数据库中的table表，es中可以在Index中建立type（table），通过mapping进行映射。

#### Document

由于es存储的数据是文档型的，一条数据对应一篇文档即相当于mysql数据库中的一行数据row， 一个文档中可以有多个字段也就是mysql数据库一行可以有多列。

#### Field

es中一个文档中对应的多个列与mysql数据库中每一列对应

#### Mapping

可以理解为mysql或者solr中对应的schema，只不过有些时候es中的mapping增加了动态识别功能，感觉很强大的样子， 其实实际生产环境上不建议使用，最好还是开始制定好了对应的schema为主。

#### indexed

就是名义上的建立索引。mysql中一般会对经常使用的列增加相应的索引用于提高查询速度，而在es中默认都是会加 上索引的，除非你特殊制定不建立索引只是进行存储用于展示，这个需要看你具体的需求和业务进行设定了。

#### Query DSL

类似于mysql的sql语句，只不过在es中是使用的json格式的查询语句，专业术语就叫：QueryDSL

#### GET/PUT/POST/DELETE

分别类似与mysql中的select/update/delete......

### 1.3 Elasticsearch的架构

![es](es.png)

#### Gateway层

es用来存储索引文件的一个文件系统且它支持很多类型，例如：本地磁盘、共享存储（做snapshot的时候需要用到）、hadoop 的hdfs分布式存储、亚马逊的S3。它的主要职责是用来对数据进行长持久化以及整个集群重启之后可以通过gateway重新恢复数据。

#### Distributed Lucene Directory

Gateway上层就是一个lucene的分布式框架，lucene是做检索的，但是它是一个单机的搜索引擎，像这种es分布式搜索引擎系 统，虽然底层用lucene，但是需要在每个节点上都运行lucene进行相应的索引、查询以及更新，所以需要做成一个分布式的运 行框架来满足业务的需要。

#### 四大模块组件

districted lucene directory之上就是一些es的模块

1.**Index Module**是索引模块，就是对数据建立索引也就是通常所说的建立一些倒排索引等；

2.**Search Module**是搜索模块，就是对数据进行查询搜索；

3.**Mapping**模块是数据映射与解析模块，就是你的数据的每个字段可以根据你建立的表结构        通过mapping进行映射解析，如果你没有建立表结构，es就会根据你的数据类型推测你 的数据结构之后自己生成一个mapping，然后都是根据这个mapping进行解析你的数据；

4.**River**模块在es2.0之后应该是被取消了，它的意思表示是第三方插件，例如可以通过一 些自定义的脚本将传统的数据库（mysql）等数据源通过格式化转换后直接同步到es集群里， 这个river大部分是自己写的，写出来的东西质量参差不齐，将这些东西集成到es中会引发 很多内部bug，严重影响了es的正常应用，所以在es2.0之后考虑将其去掉。

#### Discovery、Script

es4大模块组件之上有 Discovery模块：es是一个集群包含很多节点，很多节点需要互相发现对方，然后组成一个集群包括选 主的，这些es都是用的discovery模块，默认使用的是Zen，也可是使用EC2；es查询还可以支撑多种script即脚本语言，包括 mvel、js、python等等。

#### Transport协议层

再上一层就是es的通讯接口Transport，支持的也比较多：Thrift、Memcached以及Http，默认的是http，JMX就是java的一个 远程监控管理框架，因为es是通过java实现的。

#### RESTful接口层

最上层就是es暴露给我们的访问接口，官方推荐的方案就是这种Restful接口，直接发送http请求，方便后续使用nginx做代理、 分发包括可能后续会做权限的管理，通过http很容易做这方面的管理。如果使用java客户端它是直接调用api，在做负载均衡以及权限管理还是不太好做。

### 部署

### CentOS7下安装ElasticSearch6.2.4

#### 配置JDK环境

```
配置环境变量
复制代码
export JAVA_HOME="/opt/jdk1.8.0_144"

export PATH="$JAVA_HOME/bin:$PATH"

export CLASSPATH=".:$JAVA_HOME/lib"
复制代码
```

#### 安装ElasticSearch6.2.4

下载地址：[www.elastic.co/cn/download…](https://link.juejin.cn?target=https%3A%2F%2Fwww.elastic.co%2Fcn%2Fdownloads%2Felasticsearch)

启动报错：

Caused by ： can not run elasticsearch as root

解决方式： bin/elasticsearch -Des.insecure.allow.root=true

或者修改bin/elasticsearch，加上ES_JAVA_OPTS属性： ES_JAVA_OPTS="-Des.insecure.allow.root=true"

### Head插件

Head是elasticsearch的集群管理工具，可以用于数据的浏览和查询

(1)elasticsearch-head是一款开源软件，被托管在github上面，所以如果我们要使用它，必须先安装git，通过git获取elasticsearch-head

(2)运行elasticsearch-head会用到grunt，而grunt需要npm包管理器，所以nodejs是必须要安装的

(3)elasticsearch5.0之后，elasticsearch-head不做为插件放在其plugins目录下了。 使用git拷贝elasticsearch-head到本地

cd /usr/local/

git clone git://github.com/mobz/elasticsearch-head.git

(4)安装elasticsearch-head依赖包

[root@localhost local]# npm install -g grunt-cli

[root@localhost _site]# cd /usr/local/elasticsearch-head/

[root@localhost elasticsearch-head]# cnpm install

(5)修改Gruntfile.js

[root@localhost _site]# cd /usr/local/elasticsearch-head/

[root@localhost elasticsearch-head]# vi Gruntfile.js

在connect-->server-->options下面添加：hostname:’*’，允许所有IP可以访问

(6)修改elasticsearch-head默认连接地址 [root@localhost elasticsearch-head]# cd /usr/local/elasticsearch-head/_site/

[root@localhost _site]# vi app.js

将this.base_uri = this.config.base_uri || this.prefs.get("app-base_uri") || "[http://localhost:9200](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A9200)";中的localhost修改成你es的服务器地址

(7)配置elasticsearch允许跨域访问

打开elasticsearch的配置文件elasticsearch.yml，在文件末尾追加下面两行代码即可：

http.cors.enabled: true

http.cors.allow-origin: "*"

(8)打开9100端口

[root@localhost elasticsearch-head]# firewall-cmd --zone=public --add-port=9100/tcp --permanent

重启防火墙

[root@localhost elasticsearch-head]# firewall-cmd --reload

(9)启动elasticsearch

(10)启动elasticsearch-head

[root@localhost _site]# cd /usr/local/elasticsearch-head/

[root@localhost elasticsearch-head]# node_modules/grunt/bin/grunt server

(11)访问elasticsearch-head

关闭防火墙：systemctl stop firewalld.service

浏览器输入网址：[http://192.168.25.131:9100/](https://link.juejin.cn?target=http%3A%2F%2F192.168.25.131%3A9100%2F)

### 安装Kibana

Kibana是一个针对Elasticsearch的开源分析及可视化平台，使用Kibana可以查询、查看并与存储在ES索引的数据进行交互操作，使用Kibana能执行高级的数据分析，并能以图表、表格和地图的形式查看数据

(1)下载Kibana [www.elastic.co/downloads/k…](https://link.juejin.cn?target=https%3A%2F%2Fwww.elastic.co%2Fdownloads%2Fkibana)

(2)把下载好的压缩包拷贝到/soft目录下

(3)解压缩，并把解压后的目录移动到/user/local/kibana

(4)编辑kibana配置文件

[root@localhost /]# vi /usr/local/kibana/config/kibana.yml



![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/9/19/16d4750c94d90034~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



将server.host,elasticsearch.url修改成所在服务器的ip地址

(5)开启5601端口

Kibana的默认端口是5601

开启防火墙:systemctl start firewalld.service

开启5601端口:firewall-cmd --permanent --zone=public --add-port=5601/tcp

重启防火墙：firewall-cmd –reload

(6)启动Kibana

[root@localhost /]# /usr/local/kibana/bin/kibana

浏览器访问：[http://192.168.25.131:5601](https://link.juejin.cn?target=http%3A%2F%2F192.168.25.131%3A5601)

### 1.9安装中文分词器

(1)下载中文分词器 [github.com/medcl/elast…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fmedcl%2Felasticsearch-analysis-ik)

```
下载elasticsearch-analysis-ik-master.zip
复制代码
```

(2)解压elasticsearch-analysis-ik-master.zip

unzip elasticsearch-analysis-ik-master.zip

(3)进入elasticsearch-analysis-ik-master，编译源码

mvn clean install -Dmaven.test.skip=true

(4)在es的plugins文件夹下创建目录ik

(5)将编译后生成的elasticsearch-analysis-ik-版本.zip移动到ik下，并解压

(6)解压后的内容移动到ik目录下

## 基本操作

### 倒排索引

Elasticsearch 使用一种称为 倒排索引 的结构，它适用于快速的全文搜索。一个倒排索引由文档中所有不重复词的列表构成，对于其中每个词，有一个包含它的文档列表。

示例：

(1)：假设文档集合包含五个文档，每个文档内容如图所示，在图中最左端一栏是每个文档对应的文档编号。我们的任务就是对这个文档集合建立倒排索引。



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/9/20/16d4ccc7d2e9e209~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



(2):中文和英文等语言不同，单词之间没有明确分隔符号，所以首先要用分词系统将文档自动切分成单词序列。这样每个文档就转换为由单词序列构成的数据流，为了系统后续处理方便，需要对每个不同的单词赋予唯一的单词编号，同时记录下哪些文档包含这个单词，在如此处理结束后，我们可以得到最简单的倒排索引



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/9/20/16d4ccd312187b3f~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

“单词ID”一栏记录了每个单词的单词编号，第二栏是对应的单词，第三栏即每个单词对应的倒排列表



(3):索引系统还可以记录除此之外的更多信息,下图还记载了单词频率信息（TF）即这个单词在某个文档中的出现次数，之所以要记录这个信息，是因为词频信息在搜索结果排序时，计算查询和文档相似度是很重要的一个计算因子，所以将其记录在倒排列表中，以方便后续排序时进行分值计算。



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/9/20/16d4cce3ac5f1792~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



(4):倒排列表中还可以记录单词在某个文档出现的位置信息

(1,<11>,1),(2,<7>,1),(3,<3,9>,2)

有了这个索引系统，搜索引擎可以很方便地响应用户的查询，比如用户输入查询词“Facebook”，搜索系统查找倒排索引，从中可以读出包含这个单词的文档，这些文档就是提供给用户的搜索结果，而利用单词频率信息、文档频率信息即可以对这些候选搜索结果进行排序，计算文档和查询的相似性，按照相似性得分由高到低排序输出，此即为搜索系统的部分内部流程。

#### 2.1.2 倒排索引原理

1.The quick brown fox jumped over the lazy dog

2.Quick brown foxes leap over lazy dogs in summer

倒排索引：

| Term   | Doc_1 | Doc_2 |
| ------ | ----- | ----- |
| Quick  |       | X     |
| The    | X     |       |
| brown  | X     | X     |
| dog    | X     |       |
| dogs   |       | X     |
| fox    | X     |       |
| foxes  |       | X     |
| in     |       | X     |
| jumped | X     |       |
| lazy   | X     | X     |
| leap   |       | X     |
| over   | X     | X     |
| quick  | X     |       |
| summer |       | X     |
| the    | X     |       |

搜索quick brown ：

| Term  | Doc_1 | Doc_2 |
| ----- | ----- | ----- |
| brown | X     | X     |
| quick | X     |       |
| Total | 2     | 1     |

计算相关度分数时，文档1的匹配度高，分数会比文档2高

问题：

Quick 和 quick 以独立的词条出现，然而用户可能认为它们是相同的词。

fox 和 foxes 非常相似, 就像 dog 和 dogs ；他们有相同的词根。

jumped 和 leap, 尽管没有相同的词根，但他们的意思很相近。他们是同义词。

搜索含有 Quick fox的文档是搜索不到的

使用标准化规则(normalization)： 建立倒排索引的时候，会对拆分出的各个单词进行相应的处理，以提升后面搜索的时候能够搜索到相关联的文档的概率

| Term   | Doc_1 | Doc_2 |
| ------ | ----- | ----- |
| brown  | X     | X     |
| dog    | X     | X     |
| fox    | X     | X     |
| in     |       | X     |
| jump   | X     | X     |
| lazy   | X     | X     |
| over   | X     | X     |
| quick  | X     | X     |
| summer |       | X     |
| the    | X     | X     |

#### 2.1.3 分词器介绍及内置分词器

分词器：从一串文本中切分出一个一个的词条，并对每个词条进行标准化

包括三部分：

character filter：分词之前的预处理，过滤掉HTML标签，特殊符号转换等

tokenizer：分词

token filter：标准化

内置分词器：

standard 分词器：(默认的)他会将词汇单元转换成小写形式，并去除停用词和标点符号，支持中文采用的方法为单字切分

simple 分词器：首先会通过非字母字符来分割文本信息，然后将词汇单元统一为小写形式。该分析器会去掉数字类型的字符。

Whitespace 分词器：仅仅是去除空格，对字符没有lowcase化,不支持中文； 并且不对生成的词汇单元进行其他的标准化处理。

language 分词器：特定语言的分词器，不支持中文

### 2.2 使用ElasticSearch API 实现CRUD

添加索引：

```
PUT /lib/

{

  "settings":{
  
      "index":{
      
        "number_of_shards": 5,
        
        "number_of_replicas": 1
        
        }
        
      }
}

PUT  lib
复制代码
```

查看索引信息:

```
GET /lib/_settings

GET _all/_settings
复制代码
```

添加文档:

```
PUT /lib/user/1

{
    "first_name" :  "Jane",
    
    "last_name" :   "Smith",
    
    "age" :         32,
    
    "about" :       "I like to collect rock albums",
    
    "interests":  [ "music" ]
}

POST /lib/user/

{
    "first_name" :  "Douglas",
    
    "last_name" :   "Fir",
    
    "age" :         23,
    
    "about":        "I like to build cabinets",
    
    "interests":  [ "forestry" ]
    
}
复制代码
```

查看文档:

```
GET /lib/user/1

GET /lib/user/

GET /lib/user/1?_source=age,interests
复制代码
```

更新文档:

```
PUT /lib/user/1

{
    "first_name" :  "Jane",
    
    "last_name" :   "Smith",
    
    "age" :         36,
    
    "about" :       "I like to collect rock albums",
    
    "interests":  [ "music" ]
}

POST /lib/user/1/_update

{

  "doc":{
  
      "age":33
      
      }
}
复制代码
```

删除一个文档

```
DELETE /lib/user/1
复制代码
```

删除一个索引

```
DELETE /lib
复制代码
```

### 2.3 批量获取文档

使用es提供的Multi Get API：

使用Multi Get API可以通过索引名、类型名、文档id一次得到一个文档集合，文档可以来自同一个索引库，也可以来自不同索引库

使用curl命令：

```
curl 'http://192.168.25.131:9200/_mget' -d '{

"docs"：[

   {
   
    "_index": "lib",
    
    "_type": "user",
    
    "_id": 1
    
   },
   
   {
   
     "_index": "lib",
     
     "_type": "user",
     
     "_id": 2
     
   }

  ]
}'
复制代码
```

在客户端工具中：

```
GET /_mget

{
   
    "docs":[
       
       {
           "_index": "lib",
           "_type": "user",
           "_id": 1
       },
       {
           "_index": "lib",
           "_type": "user",
           "_id": 2
       },
       {
           "_index": "lib",
           "_type": "user",
           "_id": 3
       }
       
     ]
}
复制代码
```

可以指定具体的字段：

```
GET /_mget

{
   
    "docs":[
       
       {
           "_index": "lib",
           "_type": "user",
           "_id": 1,
           "_source": "interests"
       },
       {
           "_index": "lib",
           "_type": "user",
           "_id": 2,
           "_source": ["age","interests"]
       }
       
     ]
}
复制代码
```

获取同索引同类型下的不同文档：

```
GET /lib/user/_mget

{
   
    "docs":[
       
       {
           "_id": 1
       },
       {
           "_type": "user",
           "_id": 2,
       }
       
     ]
}

GET /lib/user/_mget

{
   
   "ids": ["1","2"]
   
}
复制代码
```

### 2.4 使用Bulk API 实现批量操作

bulk的格式：

```
{action:{metadata}}\n

{requstbody}\n

action:(行为)

  create：文档不存在时创建
  
  update:更新文档
  
  index:创建新文档或替换已有文档
  
  delete:删除一个文档
  
metadata：_index,_type,_id
复制代码
```

create 和index的区别

如果数据存在，使用create操作失败，会提示文档已经存在，使用index则可以成功执行。

示例：

```
{"delete":{"_index":"lib","_type":"user","_id":"1"}}
复制代码
```

批量添加:

```
POST /lib2/books/_bulk

{"index":{"_id":1}}

{"title":"Java","price":55}

{"index":{"_id":2}}

{"title":"Html5","price":45}

{"index":{"_id":3}}

{"title":"Php","price":35}

{"index":{"_id":4}}

{"title":"Python","price":50}
复制代码
```

批量获取:

```
GET /lib2/books/_mget
{

"ids": ["1","2","3","4"]
}
复制代码
```

删除：没有请求体

```
POST /lib2/books/_bulk

{"delete":{"_index":"lib2","_type":"books","_id":4}}

{"create":{"_index":"tt","_type":"ttt","_id":"100"}}

{"name":"lisi"}

{"index":{"_index":"tt","_type":"ttt"}}

{"name":"zhaosi"}

{"update":{"_index":"lib2","_type":"books","_id":"4"}}

{"doc":{"price":58}}
复制代码
```

bulk一次最大处理多少数据量:

  bulk会把将要处理的数据载入内存中，所以数据量是有限制的，最佳的数据量不是一个确定的数值，它取决于你的硬件，你的文档大小以及复杂性，你的索引以及搜索的负载。

  一般建议是1000-5000个文档，大小建议是5-15MB，默认不能超过100M，可以在es的配置文件（即$ES_HOME下的config下的elasticsearch.yml）中。  

### 2.5 版本控制

ElasticSearch采用了乐观锁来保证数据的一致性，也就是说，当用户对document进行操作时，并不需要对该document作加锁和解锁的操作，只需要指定要操作的版本即可。当版本号一致时，ElasticSearch会允许该操作顺利执行，而当版本号存在冲突时，ElasticSearch会提示冲突并抛出异常（VersionConflictEngineException异常）。

ElasticSearch的版本号的取值范围为1到2^63-1。

内部版本控制：使用的是_version

外部版本控制：elasticsearch在处理外部版本号时会与对内部版本号的处理有些不同。它不再是检查_version是否与请求中指定的数值_相同_,而是检查当前的_version是否比指定的数值小。如果请求成功，那么外部的版本号就会被存储到文档中的_version中。

为了保持_version与外部版本控制的数据一致 使用version_type=external

### 2.6 什么是Mapping

```
PUT /myindex/article/1 
{ 
  "post_date": "2018-05-10", 
  "title": "Java", 
  "content": "java is the best language", 
  "author_id": 119
}

PUT /myindex/article/2
{ 
  "post_date": "2018-05-12", 
  "title": "html", 
  "content": "I like html", 
  "author_id": 120
}

PUT /myindex/article/3
{ 
  "post_date": "2018-05-16", 
  "title": "es", 
  "content": "Es is distributed document store", 
  "author_id": 110
}


GET /myindex/article/_search?q=2018-05

GET /myindex/article/_search?q=2018-05-10

GET /myindex/article/_search?q=html

GET /myindex/article/_search?q=java

#查看es自动创建的mapping

GET /myindex/article/_mapping
复制代码
```

es自动创建了index，type，以及type对应的mapping(dynamic mapping)

什么是映射：mapping定义了type中的每个字段的数据类型以及这些字段如何分词等相关属性

```
{
  "myindex": {
    "mappings": {
      "article": {
        "properties": {
          "author_id": {
            "type": "long"
          },
          "content": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          },
          "post_date": {
            "type": "date"
          },
          "title": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          }
        }
      }
    }
  }
}
复制代码
```

创建索引的时候,可以预先定义字段的类型以及相关属性，这样就能够把日期字段处理成日期，把数字字段处理成数字，把字符串字段处理字符串值等

支持的数据类型：

(1)核心数据类型（Core datatypes）

```
字符型：string，string类型包括
text 和 keyword

text类型被用来索引长文本，在建立索引前会将这些文本进行分词，转化为词的组合，建立索引。允许es来检索这些词语。text类型不能用来排序和聚合。

Keyword类型不需要进行分词，可以被用来检索过滤、排序和聚合。keyword 类型字段只能用本身来进行检索

数字型：long, integer, short, byte, double, float
日期型：date
布尔型：boolean
二进制型：binary
复制代码
```

(2)复杂数据类型（Complex datatypes）

```
数组类型（Array datatype）：数组类型不需要专门指定数组元素的type，例如：
    字符型数组: [ "one", "two" ]
    整型数组：[ 1, 2 ]
    数组型数组：[ 1, [ 2, 3 ]] 等价于[ 1, 2, 3 ]
    对象数组：[ { "name": "Mary", "age": 12 }, { "name": "John", "age": 10 }]
对象类型（Object datatype）：_ object _ 用于单个JSON对象；
嵌套类型（Nested datatype）：_ nested _ 用于JSON数组；
复制代码
```

(3)地理位置类型（Geo datatypes）

```
地理坐标类型（Geo-point datatype）：_ geo_point _ 用于经纬度坐标；
地理形状类型（Geo-Shape datatype）：_ geo_shape _ 用于类似于多边形的复杂形状；
复制代码
```

(4)特定类型（Specialised datatypes）

```
IPv4 类型（IPv4 datatype）：_ ip _ 用于IPv4 地址；
Completion 类型（Completion datatype）：_ completion _提供自动补全建议；
Token count 类型（Token count datatype）：_ token_count _ 用于统计做了标记的字段的index数目，该值会一直增加，不会因为过滤条件而减少。
mapper-murmur3
类型：通过插件，可以通过 _ murmur3 _ 来计算 index 的 hash 值；
附加类型（Attachment datatype）：采用 mapper-attachments
插件，可支持_ attachments _ 索引，例如 Microsoft Office 格式，Open Document 格式，ePub, HTML 等。
复制代码
```

支持的属性：

"store":false//是否单独设置此字段的是否存储而从_source字段中分离，默认是false，只能搜索，不能获取值

"index": true//分词，不分词是：false ，设置成false，字段将不会被索引

"analyzer":"ik"//指定分词器,默认分词器为standard analyzer

"boost":1.23//字段级别的分数加权，默认值是1.0

"doc_values":false//对not_analyzed字段，默认都是开启，分词字段不能使用，对排序和聚合能提升较大性能，节约内存

"fielddata":{"format":"disabled"}//针对分词字段，参与排序或聚合时能提高性能，不分词字段统一建议使用doc_value

"fields":{"raw":{"type":"string","index":"not_analyzed"}} //可以对一个字段提供多种索引模式，同一个字段的值，一个分词，一个不分词

"ignore_above":100 //超过100个字符的文本，将会被忽略，不被索引

"include_in_all":ture//设置是否此字段包含在_all字段中，默认是true，除非index设置成no选项

"index_options":"docs"//4个可选参数docs（索引文档号） ,freqs（文档号+词频），positions（文档号+词频+位置，通常用来距离查询），offsets（文档号+词频+位置+偏移量，通常被使用在高亮字段）分词字段默认是position，其他的默认是docs

"norms":{"enable":true,"loading":"lazy"}//分词字段默认配置，不分词字段：默认{"enable":false}，存储长度因子和索引时boost，建议对需要参与评分字段使用 ，会额外增加内存消耗量

"null_value":"NULL"//设置一些缺失字段的初始化值，只有string可以使用，分词字段的null值也会被分词

"position_increament_gap":0//影响距离查询或近似查询，可以设置在多值字段的数据上火分词字段上，查询时可指定slop间隔，默认值是100

"search_analyzer":"ik"//设置搜索时的分词器，默认跟ananlyzer是一致的，比如index时用standard+ngram，搜索时用standard用来完成自动提示功能

"similarity":"BM25"//默认是TF/IDF算法，指定一个字段评分策略，仅仅对字符串型和分词类型有效

"term_vector":"no"//默认不存储向量信息，支持参数yes（term存储），with_positions（term+位置）,with_offsets（term+偏移量），with_positions_offsets(term+位置+偏移量) 对快速高亮fast vector highlighter能提升性能，但开启又会加大索引体积，不适合大数据量用

映射的分类：

(1)动态映射：

当ES在文档中碰到一个以前没见过的字段时，它会利用动态映射来决定该字段的类型，并自动地对该字段添加映射。

可以通过dynamic设置来控制这一行为，它能够接受以下的选项：

```
true：默认值。动态添加字段
false：忽略新字段
strict：如果碰到陌生字段，抛出异常
复制代码
```

dynamic设置可以适用在根对象上或者object类型的任意字段上。

POST /lib2

\#给索引lib2创建映射类型

```
{

    "settings":{
    
    "number_of_shards" : 3,
    
    "number_of_replicas" : 0
    
    },
    
     "mappings":{
     
      "books":{
      
        "properties":{
        
            "title":{"type":"text"},
            "name":{"type":"text","index":false},
            "publish_date":{"type":"date","index":false},
            
            "price":{"type":"double"},
            
            "number":{"type":"integer"}
        }
      }
     }
}
复制代码
```

POST /lib2

\#给索引lib2创建映射类型

```
{

    "settings":{
    
    "number_of_shards" : 3,
    
    "number_of_replicas" : 0
    
    },
    
     "mappings":{
     
      "books":{
      
        "properties":{
        
            "title":{"type":"text"},
            "name":{"type":"text","index":false},
            "publish_date":{"type":"date","index":false},
            
            "price":{"type":"double"},
            
            "number":{
                "type":"object",
                "dynamic":true
            }
        }
      }
     }
}
复制代码
```

### 2.7 基本查询(Query查询)

#### 2.7.1 数据准备

```
PUT /lib3
{
    "settings":{
    "number_of_shards" : 3,
    "number_of_replicas" : 0
    },
     "mappings":{
      "user":{
        "properties":{
            "name": {"type":"text"},
            "address": {"type":"text"},
            "age": {"type":"integer"},
            "interests": {"type":"text"},
            "birthday": {"type":"date"}
        }
      }
     }
}

GET /lib3/user/_search?q=name:lisi

GET /lib3/user/_search?q=name:zhaoliu&sort=age:desc
复制代码
```

#### 2.7.2 term查询和terms查询

term query会去倒排索引中寻找确切的term，它并不知道分词器的存在。这种查询适合keyword 、numeric、date。

term:查询某个字段里含有某个关键词的文档

```
GET /lib3/user/_search/
{
  "query": {
      "term": {"interests": "changge"}
  }
}
复制代码
```

terms:查询某个字段里含有多个关键词的文档

```
GET /lib3/user/_search
{
    "query":{
        "terms":{
            "interests": ["hejiu","changge"]
        }
    }
}
复制代码
```

#### 2.7.3 控制查询返回的数量

from：从哪一个文档开始 size：需要的个数

```
GET /lib3/user/_search
{
    "from":0,
    "size":2,
    "query":{
        "terms":{
            "interests": ["hejiu","changge"]
        }
    }
}
复制代码
```

#### 2.7.4 返回版本号

```
GET /lib3/user/_search
{
    "version":true,
    "query":{
        "terms":{
            "interests": ["hejiu","changge"]
        }
    }
}
复制代码
```

#### 2.7.5 match查询

match query知道分词器的存在，会对filed进行分词操作，然后再查询

```
GET /lib3/user/_search
{
    "query":{
        "match":{
            "name": "zhaoliu"
        }
    }
}
GET /lib3/user/_search
{
    "query":{
        "match":{
            "age": 20
        }
    }
}
复制代码
```

match_all:查询所有文档

```
GET /lib3/user/_search
{
  "query": {
    "match_all": {}
  }
}
复制代码
```

multi_match:可以指定多个字段

```
GET /lib3/user/_search
{
    "query":{
        "multi_match": {
            "query": "lvyou",
            "fields": ["interests","name"]
         }
    }
}
复制代码
```

match_phrase:短语匹配查询

ElasticSearch引擎首先分析（analyze）查询字符串，从分析后的文本中构建短语查询，这意味着必须匹配短语中的所有分词，并且保证各个分词的相对位置不变：

```
GET lib3/user/_search
{
  "query":{  
      "match_phrase":{  
         "interests": "duanlian，shuoxiangsheng"
      }
   }
}
复制代码
```

#### 2.7.6 指定返回的字段

```
GET /lib3/user/_search
{
    "_source": ["address","name"],
    "query": {
        "match": {
            "interests": "changge"
        }
    }
}
复制代码
```

#### 2.7.7 控制加载的字段

```
GET /lib3/user/_search
{
    "query": {
        "match_all": {}
    },
    
    "_source": {
          "includes": ["name","address"],
          "excludes": ["age","birthday"]
      }
}
复制代码
```

使用通配符*

```
GET /lib3/user/_search
{
    "_source": {
          "includes": "addr*",
          "excludes": ["name","bir*"]
        
    },
    "query": {
        "match_all": {}
    }
}
复制代码
```

#### 2.7.8 排序

使用sort实现排序： desc:降序，asc升序

```
GET /lib3/user/_search
{
    "query": {
        "match_all": {}
    },
    "sort": [
        {
           "age": {
               "order":"asc"
           }
        }
    ]
        
}

GET /lib3/user/_search
{
    "query": {
        "match_all": {}
    },
    "sort": [
        {
           "age": {
               "order":"desc"
           }
        }
    ]
        
}
复制代码
```

#### 2.7.9 前缀匹配查询

```
GET /lib3/user/_search
{
  "query": {
    "match_phrase_prefix": {
        "name": {
            "query": "zhao"
        }
    }
  }
}
复制代码
```

#### 2.7.10 范围查询

range:实现范围查询

参数：from,to,include_lower,include_upper,boost

include_lower:是否包含范围的左边界，默认是true

include_upper:是否包含范围的右边界，默认是true

```
GET /lib3/user/_search
{
    "query": {
        "range": {
            "birthday": {
                "from": "1990-10-10",
                "to": "2018-05-01"
            }
        }
    }
}


GET /lib3/user/_search
{
    "query": {
        "range": {
            "age": {
                "from": 20,
                "to": 25,
                "include_lower": true,
                "include_upper": false
            }
        }
    }
}
复制代码
```

#### 2.7.11 wildcard查询

允许使用通配符* 和 ?来进行查询

*代表0个或多个字符

？代表任意一个字符

```
GET /lib3/user/_search
{
    "query": {
        "wildcard": {
             "name": "zhao*"
        }
    }
}


GET /lib3/user/_search
{
    "query": {
        "wildcard": {
             "name": "li?i"
        }
    }
}
复制代码
```

#### 2.7.12 fuzzy实现模糊查询

value：查询的关键字

boost：查询的权值，默认值是1.0

min_similarity:设置匹配的最小相似度，默认值为0.5，对于字符串，取值为0-1(包括0和1);对于数值，取值可能大于1;对于日期型取值为1d,1m等，1d就代表1天

prefix_length:指明区分词项的共同前缀长度，默认是0

max_expansions:查询中的词项可以扩展的数目，默认可以无限大

```
GET /lib3/user/_search
{
    "query": {
        "fuzzy": {
             "interests": "chagge"
        }
    }
}

GET /lib3/user/_search
{
    "query": {
        "fuzzy": {
             "interests": {
                 "value": "chagge"
             }
        }
    }
}
复制代码
```

#### 2.7.13 高亮搜索结果

```
GET /lib3/user/_search
{
    "query":{
        "match":{
            "interests": "changge"
        }
    },
    "highlight": {
        "fields": {
             "interests": {}
        }
    }
}
复制代码
```

### 2.8 Filter查询

filter是不计算相关性的，同时可以cache。因此，filter速度要快于query。

```
POST /lib4/items/_bulk
{"index": {"_id": 1}}

{"price": 40,"itemID": "ID100123"}

{"index": {"_id": 2}}

{"price": 50,"itemID": "ID100124"}

{"index": {"_id": 3}}

{"price": 25,"itemID": "ID100124"}

{"index": {"_id": 4}}

{"price": 30,"itemID": "ID100125"}

{"index": {"_id": 5}}

{"price": null,"itemID": "ID100127"}
复制代码
```

\####2.8.1 简单的过滤查询

```
GET /lib4/items/_search
{ 
       "post_filter": {
             "term": {
                 "price": 40
             }
       }
}


GET /lib4/items/_search
{
      "post_filter": {
          "terms": {
                 "price": [25,40]
              }
        }
}

GET /lib4/items/_search
{
    "post_filter": {
        "term": {
            "itemID": "ID100123"
          }
      }
}
复制代码
```

查看分词器分析的结果：

```
GET /lib4/_mapping
复制代码
```

不希望商品id字段被分词，则重新创建映射

```
DELETE lib4

PUT /lib4
{
    "mappings": {
        "items": {
            "properties": {
                "itemID": {
                    "type": "text",
                    "index": false
                }
            }
        }
    }
}
复制代码
```

#### 2.8.2 bool过滤查询

可以实现组合过滤查询

格式：

```
{
    "bool": {
        "must": [],
        "should": [],
        "must_not": []
    }
}
复制代码
```

must:必须满足的条件---and

should：可以满足也可以不满足的条件--or

must_not:不需要满足的条件--not

```
GET /lib4/items/_search
{
    "post_filter": {
          "bool": {
               "should": [
                    {"term": {"price":25}},
                    {"term": {"itemID": "id100123"}}
                   
                  ],
                "must_not": {
                    "term":{"price": 30}
                   }
                       
                }
             }
}
复制代码
```

嵌套使用bool：

```
GET /lib4/items/_search
{
    "post_filter": {
          "bool": {
                "should": [
                    {"term": {"itemID": "id100123"}},
                    {
                      "bool": {
                          "must": [
                              {"term": {"itemID": "id100124"}},
                              {"term": {"price": 40}}
                            ]
                          }
                    }
                  ]
                }
            }
}
复制代码
```

#### 2.8.3 范围过滤

gt: >

lt: <

gte: >=

lte: <=

```
GET /lib4/items/_search
{
     "post_filter": {
          "range": {
              "price": {
                   "gt": 25,
                   "lt": 50
                }
            }
      }
}
复制代码
```

#### 2.8.5 过滤非空

```
GET /lib4/items/_search
{
  "query": {
    "bool": {
      "filter": {
          "exists":{
             "field":"price"
         }
      }
    }
  }
}

GET /lib4/items/_search
{
    "query" : {
        "constant_score" : {
            "filter": {
                "exists" : { "field" : "price" }
            }
        }
    }
}
复制代码
```

#### 2.8.6 过滤器缓存

ElasticSearch提供了一种特殊的缓存，即过滤器缓存（filter cache），用来存储过滤器的结果，被缓存的过滤器并不需要消耗过多的内存（因为它们只存储了哪些文档能与过滤器相匹配的相关信息），而且可供后续所有与之相关的查询重复使用，从而极大地提高了查询性能。

注意：ElasticSearch并不是默认缓存所有过滤器， 以下过滤器默认不缓存：

```
numeric_range
script
geo_bbox
geo_distance
geo_distance_range
geo_polygon
geo_shape
and
or
not
复制代码
```

exists,missing,range,term,terms默认是开启缓存的

开启方式：在filter查询语句后边加上 "_catch":true

### 2.9 聚合查询

(1)sum

```
GET /lib4/items/_search
{
  "size":0,
  "aggs": {
     "price_of_sum": {
         "sum": {
           "field": "price"
         }
     }
  }
}
复制代码
```

(2)min

```
GET /lib4/items/_search
{
  "size": 0, 
  "aggs": {
     "price_of_min": {
         "min": {
           "field": "price"
         }
     }
  }
}
复制代码
```

(3)max

```
GET /lib4/items/_search
{
  "size": 0, 
  "aggs": {
     "price_of_max": {
         "max": {
           "field": "price"
         }
     }
  }
}
复制代码
```

(4)avg

```
GET /lib4/items/_search
{
  "size":0,
  "aggs": {
     "price_of_avg": {
         "avg": {
           "field": "price"
         }
     }
  }
}
复制代码
```

(5)cardinality:求基数

```
GET /lib4/items/_search
{
  "size":0,
  "aggs": {
     "price_of_cardi": {
         "cardinality": {
           "field": "price"
         }
     }
  }
}
复制代码
```

(6)terms:分组

```
GET /lib4/items/_search
{
  "size":0,
  "aggs": {
     "price_group_by": {
         "terms": {
           "field": "price"
         }
     }
  }
}
复制代码
```

对那些有唱歌兴趣的用户按年龄分组

```
GET /lib3/user/_search
{
  "query": {
      "match": {
        "interests": "changge"
      }
   },
   "size": 0, 
   "aggs":{
       "age_group_by":{
           "terms": {
             "field": "age",
             "order": {
               "avg_of_age": "desc"
             }
           },
           "aggs": {
             "avg_of_age": {
               "avg": {
                 "field": "age"
               }
             }
           }
       }
   }
}
复制代码
```

### 2.10 复合查询

将多个基本查询组合成单一查询的查询

#### 2.10.1 使用bool查询

接收以下参数：

must： 文档 必须匹配这些条件才能被包含进来。

must_not： 文档 必须不匹配这些条件才能被包含进来。

should： 如果满足这些语句中的任意语句，将增加 _score，否则，无任何影响。它们主要用于修正每个文档的相关性得分。

filter： 必须 匹配，但它以不评分、过滤模式来进行。这些语句对评分没有贡献，只是根据过滤标准来排除或包含文档。

相关性得分是如何组合的。每一个子查询都独自地计算文档的相关性得分。一旦他们的得分被计算出来， bool 查询就将这些得分进行合并并且返回一个代表整个布尔操作的得分。

下面的查询用于查找 title 字段匹配 how to make millions 并且不被标识为 spam 的文档。那些被标识为 starred 或在2014之后的文档，将比另外那些文档拥有更高的排名。如果 *两者* 都满足，那么它排名将更高：

```
{
    "bool": {
        "must": { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }},
            { "range": { "date": { "gte": "2014-01-01" }}}
        ]
    }
}
复制代码
```

如果没有 must 语句，那么至少需要能够匹配其中的一条 should 语句。但，如果存在至少一条 must 语句，则对 should 语句的匹配没有要求。 如果我们不想因为文档的时间而影响得分，可以用 filter 语句来重写前面的例子：

```
{
    "bool": {
        "must": { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }}
        ],
        "filter": {
          "range": { "date": { "gte": "2014-01-01" }} 
        }
    }
}
复制代码
```

通过将 range 查询移到 filter 语句中，我们将它转成不评分的查询，将不再影响文档的相关性排名。由于它现在是一个不评分的查询，可以使用各种对 filter 查询有效的优化手段来提升性能。

bool 查询本身也可以被用做不评分的查询。简单地将它放置到 filter 语句中并在内部构建布尔逻辑：

```
{
    "bool": {
        "must": { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }}
        ],
        "filter": {
          "bool": { 
              "must": [
                  { "range": { "date": { "gte": "2014-01-01" }}},
                  { "range": { "price": { "lte": 29.99 }}}
              ],
              "must_not": [
                  { "term": { "category": "ebooks" }}
              ]
          }
        }
    }
}
复制代码
```

#### 2.10.2 constant_score查询

它将一个不变的常量评分应用于所有匹配的文档。它被经常用于你只需要执行一个 filter 而没有其它查询（例如，评分查询）的情况下。

```
{
    "constant_score":   {
        "filter": {
            "term": { "category": "ebooks" } 
        }
    }
}
复制代码
```

term 查询被放置在 constant_score 中，转成不评分的filter。这种方式可以用来取代只有 filter 语句的 bool 查询。

## 第三节 ElasticSearch原理

### 3.1 解析es的分布式架构

#### 3.1.1 分布式架构的透明隐藏特性

ElasticSearch是一个分布式系统，隐藏了复杂的处理机制

分片机制：我们不用关心数据是按照什么机制分片的、最后放入到哪个分片中

分片的副本：

集群发现机制(cluster discovery)：比如当前我们启动了一个es进程，当启动了第二个es进程时，这个进程作为一个node自动就发现了集群，并且加入了进去

shard负载均衡：比如现在有10shard，集群中有3个节点，es会进行均衡的进行分配，以保持每个节点均衡的负载请求

请求路由

#### 3.1.2 扩容机制

垂直扩容：购置新的机器，替换已有的机器

水平扩容：直接增加机器

#### 3.1.3 rebalance

增加或减少节点时会自动均衡

#### 3.1.4 master节点

主节点的主要职责是和集群操作相关的内容，如创建或删除索引，跟踪哪些节点是群集的一部分，并决定哪些分片分配给相关的节点。稳定的主节点对集群的健康是非常重要的。

#### 3.1.5 节点对等

每个节点都能接收请求 每个节点接收到请求后都能把该请求路由到有相关数据的其它节点上 接收原始请求的节点负责采集数据并返回给客户端

### 3.2 分片和副本机制

1.index包含多个shard

2.每个shard都是一个最小工作单元，承载部分数据；每个shard都是一个lucene实例，有完整的建立索引和处理请求的能力

3.增减节点时，shard会自动在nodes中负载均衡

4.primary shard和replica shard，每个document肯定只存在于某一个primary shard以及其对应的replica shard中，不可能存在于多个primary shard

5.replica shard是primary shard的副本，负责容错，以及承担读请求负载

6.primary shard的数量在创建索引的时候就固定了，replica shard的数量可以随时修改

7.primary shard的默认数量是5，replica默认是1，默认有10个shard，5个primary shard，5个replica shard

8.primary shard不能和自己的replica shard放在同一个节点上（否则节点宕机，primary shard和副本都丢失，起不到容错的作用），但是可以和其他primary shard的replica shard放在同一个节点上

### 3.3 单节点环境下创建索引分析

```
PUT /myindex
{
   "settings" : {
      "number_of_shards" : 3,
      "number_of_replicas" : 1
   }
}
复制代码
```

这个时候，只会将3个primary shard分配到仅有的一个node上去，另外3个replica shard是无法分配的（一个shard的副本replica，他们两个是不能在同一个节点的）。集群可以正常工作，但是一旦出现节点宕机，数据全部丢失，而且集群不可用，无法接收任何请求。

### 3.4 两个节点环境下创建索引分析

将3个primary shard分配到一个node上去，另外3个replica shard分配到另一个节点上

primary shard 和replica shard 保持同步

primary shard 和replica shard 都可以处理客户端的读请求

### 3.5 水平扩容的过程

1.扩容后primary shard和replica shard会自动的负载均衡

2.扩容后每个节点上的shard会减少，那么分配给每个shard的CPU，内存，IO资源会更多，性能提高

3.扩容的极限，如果有6个shard，扩容的极限就是6个节点，每个节点上一个shard，如果想超出扩容的极限，比如说扩容到9个节点，那么可以增加replica shard的个数

4.6个shard，3个节点，最多能承受几个节点所在的服务器宕机？(容错性) 任何一台服务器宕机都会丢失部分数据

为了提高容错性，增加shard的个数： 9个shard，(3个primary shard，6个replicashard)，这样就能容忍最多两台服务器宕机了

总结：扩容是为了提高系统的吞吐量，同时也要考虑容错性，也就是让尽可能多的服务器宕机还能保证数据不丢失

### 3.6 ElasticSearch的容错机制

以9个shard，3个节点为例：

1.如果master node 宕机，此时不是所有的primary shard都是Active status，所以此时的集群状态是red。

容错处理的第一步:是选举一台服务器作为master 容错处理的第二步:新选举出的master会把挂掉的primary shard的某个replica shard 提升为primary shard,此时集群的状态为yellow，因为少了一个replica shard，并不是所有的replica shard都是active status

容错处理的第三步：重启故障机，新master会把所有的副本都复制一份到该节点上，（同步一下宕机后发生的修改），此时集群的状态为green，因为所有的primary shard和replica shard都是Active status

### 3.7 文档的核心元数据

1._index:

说明了一个文档存储在哪个索引中

同一个索引下存放的是相似的文档(文档的field多数是相同的)

索引名必须是小写的，不能以下划线开头，不能包括逗号

2._type:

表示文档属于索引中的哪个类型

一个索引下只能有一个type

类型名可以是大写也可以是小写的，不能以下划线开头，不能包括逗号

3._id:

文档的唯一标识，和索引，类型组合在一起唯一标识了一个文档

可以手动指定值，也可以由es来生成这个值

### 3.8 文档id生成方式

1.手动指定

```
put /index/type/66
复制代码
```

通常是把其它系统的已有数据导入到es时

2.由es生成id值

```
post /index/type
复制代码
```

es生成的id长度为20个字符，使用的是base64编码，URL安全，使用的是GUID算法，分布式下并发生成id值时不会冲突

### 3.9 _source元数据分析

其实就是我们在添加文档时request body中的内容

指定返回的结果中含有哪些字段：

```
get /index/type/1?_source=name
复制代码
```

### 3.10 改变文档内容原理解析

替换方式：

```
PUT /lib/user/4
{ "first_name" : "Jane",

"last_name" :   "Lucy",

"age" :         24,

"about" :       "I like to collect rock albums",

"interests":  [ "music" ]
}
复制代码
```

修改方式(partial update)：

```
POST /lib/user/2/_update
{
    "doc":{
       "age":26
     }
}
复制代码
```

删除文档：标记为deleted，随着数据量的增加，es会选择合适的时间删除掉

### 3.11 基于groovy脚本执行partial update

es有内置的脚本支持，可以基于groovy脚本实现复杂的操作

1.修改年龄

```
POST /lib/user/4/_update
{
  "script": "ctx._source.age+=1"
}
复制代码
```

2.修改名字

```
POST /lib/user/4/_update
{
  "script": "ctx._source.last_name+='hehe'"
}
复制代码
```

3.添加爱好

```
POST /lib/user/4/_update
{
  "script": {
    "source": "ctx._source.interests.add(params.tag)",
    "params": {
      "tag":"picture"
    }
  }
}
复制代码
```

4.删除爱好

```
POST /lib/user/4/_update
{
  "script": {
    "source": "ctx._source.interests.remove(ctx._source.interests.indexOf(params.tag))",
    "params": {
      "tag":"picture"
    }
  }
}
复制代码
```

5.删除文档

```
POST /lib/user/4/_update
{
  "script": {
    "source": "ctx.op=ctx._source.age==params.count?'delete':'none'",
    "params": {
        "count":29
    }
  }
}

6.upsert

POST /lib/user/4/_update
{
  "script": "ctx._source.age += 1",
  
  "upsert": {
     "first_name" : "Jane",
     "last_name" :   "Lucy",
     "age" :  20,
     "about" :       "I like to collect rock albums",
     "interests":  [ "music" ]
  }
}
复制代码
```

### 3.12 partial update 处理并发冲突

使用的是乐观锁:_version

retry_on_conflict:

POST /lib/user/4/_update?retry_on_conflict=3

重新获取文档数据和版本信息进行更新，不断的操作，最多操作的次数就是retry_on_conflict的值

### 3.13 文档数据路由原理解析

1.文档路由到分片上：

一个索引由多个分片构成，当添加(删除，修改)一个文档时，es就需要决定这个文档存储在哪个分片上，这个过程就称为数据路由(routing)

2.路由算法：

```
 shard=hash(routing) % number_of_pirmary_shards
复制代码
```

示例：一个索引，3个primary shard

(1)每次增删改查时，都有一个routing值，默认是文档的_id的值

(2)对这个routing值使用哈希函数进行计算

(3)计算出的值再和主分片个数取余数

余数肯定在0---（number_of_pirmary_shards-1）之间，文档就在对应的shard上

routing值默认是文档的_id的值，也可以手动指定一个值，手动指定对于负载均衡以及提高批量读取的性能都有帮助

3.primary shard个数一旦确定就不能修改了

### 3.14 文档增删改内部原理

1:发送增删改请求时，可以选择任意一个节点，该节点就成了协调节点(coordinating node)

2.协调节点使用路由算法进行路由，然后将请求转到primary shard所在节点，该节点处理请求，并把数据同步到它的replica shard

3.协调节点对客户端做出响应

### 3.15 写一致性原理和quorum机制

1.任何一个增删改操作都可以跟上一个参数 consistency

可以给该参数指定的值：

one: (primary shard)只要有一个primary shard是活跃的就可以执行

all: (all shard)所有的primary shard和replica shard都是活跃的才能执行

quorum: (default) 默认值，大部分shard是活跃的才能执行 （例如共有6个shard，至少有3个shard是活跃的才能执行写操作）

2.quorum机制：多数shard都是可用的，

int((primary+number_of_replica)/2)+1

例如：3个primary shard，1个replica

int((3+1)/2)+1=3

至少3个shard是活跃的

注意：可能出现shard不能分配齐全的情况

比如：1个primary shard,1个replica int((1+1)/2)+1=2 但是如果只有一个节点，因为primary shard和replica shard不能在同一个节点上，所以仍然不能执行写操作

再举例：1个primary shard,3个replica,2个节点

int((1+3)/2)+1=3

最后:当活跃的shard的个数没有达到要求时， es默认会等待一分钟，如果在等待的期间活跃的shard的个数没有增加，则显示timeout

put /index/type/id?timeout=60s

### 3.16 文档查询内部原理

第一步：查询请求发给任意一个节点，该节点就成了coordinating node，该节点使用路由算法算出文档所在的primary shard

第二步：协调节点把请求转发给primary shard也可以转发给replica shard(使用轮询调度算法(Round-Robin Scheduling，把请求平均分配至primary shard 和replica shard)

第三步：处理请求的节点把结果返回给协调节点，协调节点再返回给应用程序

特殊情况：请求的文档还在建立索引的过程中，primary shard上存在，但replica shar上不存在，但是请求被转发到了replica shard上，这时就会提示找不到文档

### 3.17 bulk批量操作的json格式解析

bulk的格式：

```
{action:{metadata}}\n

{requstbody}\n
复制代码
```

为什么不使用如下格式：

```
[{

"action": {

},

"data": {

}

}]
复制代码
```

这种方式可读性好，但是内部处理就麻烦了：

1.将json数组解析为JSONArray对象，在内存中就需要有一份json文本的拷贝，另外还有一个JSONArray对象。

2.解析json数组里的每个json，对每个请求中的document进行路由

3.为路由到同一个shard上的多个请求，创建一个请求数组

4.将这个请求数组序列化

5.将序列化后的请求数组发送到对应的节点上去

耗费更多内存，增加java虚拟机开销

1.不用将其转换为json对象，直接按照换行符切割json，内存中不需要json文本的拷贝

2.对每两个一组的json，读取meta，进行document路由

3.直接将对应的json发送到node上去

### 3.18 查询结果分析

```
{
  "took": 419,
  "timed_out": false,
  "_shards": {
    "total": 3,
    "successful": 3,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 0.6931472,
    "hits": [
      {
        "_index": "lib3",
        "_type": "user",
        "_id": "3",
        "_score": 0.6931472,
        "_source": {
          "address": "bei jing hai dian qu qing he zhen",
          "name": "lisi"
        }
      },
      {
        "_index": "lib3",
        "_type": "user",
        "_id": "2",
        "_score": 0.47000363,
        "_source": {
          "address": "bei jing hai dian qu qing he zhen",
          "name": "zhaoming"
        }
      }
复制代码
```

took：查询耗费的时间，单位是毫秒

_shards：共请求了多少个shard

total：查询出的文档总个数

max_score： 本次查询中，相关度分数的最大值，文档和此次查询的匹配度越高，_score的值越大，排位越靠前

hits：默认查询前10个文档

timed_out：

```
GET /lib3/user/_search?timeout=10ms
{
    "_source": ["address","name"],
    "query": {
        "match": {
            "interests": "changge"
        }
    }
}
复制代码
```

### 3.19 多index，多type查询模式

```
GET _search

GET /lib/_search

GET /lib,lib3/_search

GET /*3,*4/_search

GET /lib/user/_search

GET /lib,lib4/user,items/_search

GET /_all/_search

GET /_all/user,items/_search
复制代码
```

### 3.20 分页查询中的deep paging问题

```
GET /lib3/user/_search
{
    "from":0,
    "size":2,
    "query":{
        "terms":{
            "interests": ["hejiu","changge"]
        }
    }
}

GET /_search?from=0&size=3
复制代码
```

deep paging:查询的很深，比如一个索引有三个primary shard，分别存储了6000条数据，我们要得到第100页的数据(每页10条)，类似这种情况就叫deep paging

如何得到第100页的10条数据？

在每个shard中搜索990到999这10条数据，然后用这30条数据排序，排序之后取10条数据就是要搜索的数据，这种做法是错的，因为3个shard中的数据的_score分数不一样，可能这某一个shard中第一条数据的_score分数比另一个shard中第1000条都要高，所以在每个shard中搜索990到999这10条数据然后排序的做法是不正确的。

正确的做法是每个shard把0到999条数据全部搜索出来（按排序顺序），然后全部返回给coordinate node，由coordinate node按_score分数排序后，取出第100页的10条数据，然后返回给客户端。

deep paging性能问题

1.耗费网络带宽，因为搜索过深的话，各shard要把数据传送给coordinate node，这个过程是有大量数据传递的，消耗网络，

2.消耗内存，各shard要把数据传送给coordinate node，这个传递回来的数据，是被coordinate node保存在内存中的，这样会大量消耗内存。

3.消耗cpu coordinate node要把传回来的数据进行排序，这个排序过程很消耗cpu.

鉴于deep paging的性能问题，所以应尽量减少使用。

### 3.21 query string查询及copy_to解析

```
GET /lib3/user/_search?q=interests:changge

GET /lib3/user/_search?q=+interests:changge

GET /lib3/user/_search?q=-interests:changge
复制代码
```

copy_to字段是把其它字段中的值，以空格为分隔符组成一个大字符串，然后被分析和索引，但是不存储，也就是说它能被查询，但不能被取回显示。

注意:copy_to指向的字段字段类型要为：text

当没有指定field时，就会从copy_to字段中查询

```
GET /lib3/user/_search?q=changge
复制代码
```

### 3.22 字符串排序问题

对一个字符串类型的字段进行排序通常不准确，因为已经被分词成多个词条了

解决方式：对字段索引两次，一次索引分词（用于搜索），一次索引不分词(用于排序)

```
GET /lib3/_search

GET /lib3/user/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "interests": {
        "order": "desc"
      }
    }
  ]
}

GET /lib3/user/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "interests.raw": {
        "order": "asc"
      }
    }
  ]
}

DELETE lib3

PUT /lib3
{
    "settings":{
        "number_of_shards" : 3,
        "number_of_replicas" : 0
      },
     "mappings":{
      "user":{
        "properties":{
            "name": {"type":"text"},
            "address": {"type":"text"},
            "age": {"type":"integer"},
            "birthday": {"type":"date"},
            "interests": {
                "type":"text",
                "fields": {
                  "raw":{
                     "type": "keyword"
                   }
                },
                "fielddata": true
             }
          }
        }
     }
}
复制代码
```

### 3.23 如何计算相关度分数

使用的是TF/IDF算法(Term Frequency&Inverse Document Frequency)

1.Term Frequency:我们查询的文本中的词条在document本中出现了多少次，出现次数越多，相关度越高

```
搜索内容： hello world

Hello，I love china.

Hello world,how are you!
复制代码
```

2.Inverse Document Frequency：我们查询的文本中的词条在索引的所有文档中出现了多少次，出现的次数越多，相关度越低

```
搜索内容：hello world

hello，what are you doing?

I like the world.

hello 在索引的所有文档中出现了500次，world出现了100次
复制代码
```

3.Field-length(字段长度归约) norm:field越长，相关度越低

搜索内容：hello world

```
{"title":"hello,what's your name?","content":{"owieurowieuolsdjflk"}}

{"title":"hi,good morning","content":{"lkjkljkj.......world"}}
复制代码
```

查看分数是如何计算的：

```
GET /lib3/user/_search?explain=true
{
    "query":{
        "match":{
            "interests": "duanlian,changge"
        }
    }
}
复制代码
```

查看一个文档能否匹配上某个查询：

```
GET /lib3/user/2/_explain
{
    "query":{
        "match":{
            "interests": "duanlian,changge"
        }
    }
}
复制代码
```

### 3.24 Doc Values 解析

DocValues其实是Lucene在构建倒排索引时，会额外建立一个有序的正排索引(基于document => field value的映射列表)

```
{"birthday":"1985-11-11",age:23}

{"birthday":"1989-11-11",age:29}

document     age       birthday

doc1         23         1985-11-11

doc2         29         1989-11-11
复制代码
```

存储在磁盘上，节省内存

对排序，分组和一些聚合操作能够大大提升性能

注意：默认对不分词的字段是开启的，对分词字段无效（需要把fielddata设置为true）

```
PUT /lib3
{
    "settings":{
    "number_of_shards" : 3,
    "number_of_replicas" : 0
    },
     "mappings":{
      "user":{
        "properties":{
            "name": {"type":"text"},
            "address": {"type":"text"},
            "age": {
              "type":"integer",
              "doc_values":false
            },
            "interests": {"type":"text"},
            "birthday": {"type":"date"}
        }
      }
     }
}
复制代码
```

### 3.25 基于scroll技术滚动搜索大量数据

如果一次性要查出来比如10万条数据，那么性能会很差，此时一般会采取用scoll滚动查询，一批一批的查，直到所有数据都查询完为止。

1.scoll搜索会在第一次搜索的时候，保存一个当时的视图快照，之后只会基于该旧的视图快照提供数据搜索，如果这个期间数据变更，是不会让用户看到的

2.采用基于_doc(不使用_score)进行排序的方式，性能较高

3.每次发送scroll请求，我们还需要指定一个scoll参数，指定一个时间窗口，每次搜索请求只要在这个时间窗口内能完成就可以了

```
GET /lib3/user/_search?scroll=1m
{
  "query": {
    "match_all": {}
  },
  "sort":["_doc"],
  "size":3
}

GET /_search/scroll
{
   "scroll": "1m",
   "scroll_id": "DnF1ZXJ5VGhlbkZldGNoAwAAAAAAAAAdFkEwRENOVTdnUUJPWVZUd1p2WE5hV2cAAAAAAAAAHhZBMERDTlU3Z1FCT1lWVHdadlhOYVdnAAAAAAAAAB8WQTBEQ05VN2dRQk9ZVlR3WnZYTmFXZw=="
}
复制代码
```

### 3.26 dynamic mapping策略

**dynamic**:

1.true:遇到陌生字段就 dynamic mapping

2.false:遇到陌生字段就忽略

3.strict:约到陌生字段就报错

```
PUT /lib8
{
    "settings":{
    "number_of_shards" : 3,
    "number_of_replicas" : 0
    },
     "mappings":{
      "user":{
        "dynamic":strict,
        "properties":{
            "name": {"type":"text"},
            "address":{
                "type":"object",
                "dynamic":true
            },
        }
      }
     }
}

#会报错

PUT  /lib8/user/1
{
  "name":"lisi",
  "age":20,
  "address":{
    "province":"beijing",
    "city":"beijing"
  }
}
复制代码
```

**date_detection**:默认会按照一定格式识别date，比如yyyy-MM-dd

可以手动关闭某个type的date_detection

```
PUT /lib8
{
    "settings":{
    "number_of_shards" : 3,
    "number_of_replicas" : 0
    },
     "mappings":{
      "user":{
        "date_detection": false,
        }
    }
}
复制代码
```

**定制 dynamic mapping template(type)**

```
PUT /my_index
{ 
  "mappings": { 
    "my_type": { 
      "dynamic_templates": [ 
        { 
          "en": { 
            "match": "*_en", 
            "match_mapping_type": "string", 
            "mapping": { 
              "type": "text", 
              "analyzer": "english" 
            } 
          } 
        } 
      ] 
     } 
  } 
}
#使用了模板

PUT /my_index/my_type/3
{
  "title_en": "this is my dog"
  
}
#没有使用模板

PUT /my_index/my_type/5
{
  "title": "this is my cat"
}

GET my_index/my_type/_search
{
  "query": {
    "match": {
      "title": "is"
    }
  }
}
复制代码
```

### 3.27 重建索引

一个field的设置是不能修改的，如果要修改一个field，那么应该重新按照新的mapping，建立一个index，然后将数据批量查询出来，重新用bulk api写入到index中。

批量查询的时候，建议采用scroll api，并且采用多线程并发的方式来reindex数据，每次scroll就查询指定日期的一段数据，交给一个线程即可。

```
PUT /index1/type1/4
{
   "content":"1990-12-12"
}

GET /index1/type1/_search

GET /index1/type1/_mapping



#报错
PUT /index1/type1/4
{
   "content":"I am very happy."
}
复制代码
```

\#修改content的类型为string类型,报错，不允许修改

```
PUT /index1/_mapping/type1
{
  "properties": {
    "content":{
      "type": "text"
    }
  }
}
复制代码
```

\#创建一个新的索引，把index1索引中的数据查询出来导入到新的索引中 #但是应用程序使用的是之前的索引，为了不用重启应用程序，给index1这个索引起个#别名

```
PUT /index1/_alias/index2
复制代码
```

\#创建新的索引，把content的类型改为字符串

```
PUT /newindex
{
  "mappings": {
    "type1":{
      "properties": {
        "content":{
          "type": "text"
        }
      }
    }
  }
}
复制代码
```

\#使用scroll批量查询

```
GET /index1/type1/_search?scroll=1m
{
  "query": {
    "match_all": {}
  },
  "sort": ["_doc"],
  "size": 2
}
复制代码
```

\#使用bulk批量写入新的索引

```
POST /_bulk
{"index":{"_index":"newindex","_type":"type1","_id":1}}
{"content":"1982-12-12"}
复制代码
```

\#将别名index2和新的索引关联，应用程序不用重启

```
POST /_aliases
{
  "actions": [
    {"remove": {"index":"index1","alias":"index2"}},
    {"add": {"index": "newindex","alias": "index2"}}
]
}

GET index2/type1/_search
复制代码
```

### 3.28 索引不可变的原因

倒排索引包括：

文档的列表，文档的数量，词条在每个文档中出现的次数，出现的位置，每个文档的长度，所有文档的平均长度

索引不变的原因：

不需要锁，提升了并发性能

可以一直保存在缓存中（filter）

节省cpu和io开销

## 第四节 在Java应用中访问ElasticSearch

### 4.1 在Java应用中实现查询文档

pom中加入ElasticSearch6.2.4的依赖：

```
<dependencies>
    <dependency>
      <groupId>org.elasticsearch.client</groupId>
      <artifactId>transport</artifactId>
      <version>6.2.4</version>
    </dependency>    
    
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.12</version>
      <scope>test</scope>
    </dependency>
  
  </dependencies> 
  
  <build>
      <plugins>
			<!-- java编译插件 -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.2</version>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
					<encoding>UTF-8</encoding>
				</configuration>
			</plugin>
		</plugins>
  </build>
复制代码
```

### 4.2 在Java应用中实现添加文档

```
        "{" +
                "\"id\":\"1\"," +
                "\"title\":\"Java设计模式之装饰模式\"," +
                "\"content\":\"在不必改变原类文件和使用继承的情况下，动态地扩展一个对象的功能。\"," +
                "\"postdate\":\"2018-05-20 14:38:00\"," +
                "\"url\":\"csdn.net/79239072\"" +
        "}"
        
 XContentBuilder doc1 = XContentFactory.jsonBuilder()
                    .startObject()
                    .field("id","3")
                    .field("title","Java设计模式之单例模式")
                    .field("content","枚举单例模式可以防反射攻击。")
                    .field("postdate","2018-02-03")
                    .field("url","csdn.net/79247746")
                    .endObject();
        
        IndexResponse response = client.prepareIndex("index1", "blog", null)
                .setSource(doc1)
                .get();
        
		System.out.println(response.status());
复制代码
```

### 4.3 在Java应用中实现删除文档

```
DeleteResponse response=client.prepareDelete("index1","blog","SzYJjWMBjSAutsuLRP_P").get();

//删除成功返回OK，否则返回NOT_FOUND

System.out.println(response.status());
复制代码
```

### 4.4 在Java应用中实现更新文档

```
UpdateRequest request=new UpdateRequest();
        request.index("index1")
                .type("blog")
                .id("2")
                .doc(
                		XContentFactory.jsonBuilder().startObject()
                        .field("title","单例模式解读")
                        .endObject()
                );
UpdateResponse response=client.update(request).get();

//更新成功返回OK，否则返回NOT_FOUND

System.out.println(response.status());

upsert方式：

IndexRequest request1 =new IndexRequest("index1","blog","3")
                .source(
                		XContentFactory.jsonBuilder().startObject()
                                .field("id","3")
                                .field("title","装饰模式")
                                .field("content","动态地扩展一个对象的功能")
                                .field("postdate","2018-05-23")
                                .field("url","csdn.net/79239072")
                                .endObject()
                );
        UpdateRequest request2=new UpdateRequest("index1","blog","3")
                .doc(
                		XContentFactory.jsonBuilder().startObject()
                        .field("title","装饰模式解读")
                        .endObject()
                ).upsert(request1);
        
UpdateResponse response=client.update(request2).get();
        
//upsert操作成功返回OK，否则返回NOT_FOUND

System.out.println(response.status());
复制代码
```

### 4.5 在Java应用中实现批量操作

```
MultiGetResponse mgResponse = client.prepareMultiGet()
	                .add("index1","blog","3","2")
	                .add("lib3","user","1","2","3")
	                .get();
		    
for(MultiGetItemResponse response:mgResponse){
	            GetResponse rp=response.getResponse();
	            if(rp!=null && rp.isExists()){
	                System.out.println(rp.getSourceAsString());
	            }
	        }
	        
bulk：

BulkRequestBuilder bulkRequest = client.prepareBulk();

bulkRequest.add(client.prepareIndex("lib2", "books", "4")
                .setSource(XContentFactory.jsonBuilder()
                        .startObject()
                        .field("title", "python")
                        .field("price", 68)
                        .endObject()
                )
        );
bulkRequest.add(client.prepareIndex("lib2", "books", "5")
                .setSource(XContentFactory.jsonBuilder()
                        .startObject()
                        .field("title", "VR")
                        .field("price", 38)
                        .endObject()
                )
        );
        //批量执行
BulkResponse bulkResponse = bulkRequest.get();
        
System.out.println(bulkResponse.status());
if (bulkResponse.hasFailures()) { 
            System.out.println("存在失败操作");
        }
```


作者：慢慢成长
链接：https://juejin.cn/post/6844903945261826061
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

# ES入门

### hadoop与mpp

| 比较项目           | MPP                                                       | Hadoop                                                       |
| ------------------ | --------------------------------------------------------- | ------------------------------------------------------------ |
| 平台开放           | 封闭和专有，对于某些技术,甚至不能为非客户下载文档。       | 通过互联网免费提供供应商和社区资源的完全开源                 |
| 可扩展性           | 平均数十个节点，最大100-200个节点                         | 平均100个节点，可扩展至几千个节点                            |
| 可处理数据         | 平均10TB,最大1PB                                          | 平均100TB, 最大PB                                            |
| 延迟               | 10-20毫秒                                                 | 10-20秒                                                      |
| 平均查询时间       | 5-7 秒                                                    | 10-15 分钟                                                   |
| 最大查询时间       | 1-2小时                                                   | 1-2周                                                        |
| 查询优化           | 拥有复杂的企业级优化器                                    | 没有优化器或优化器功能非常有限，有时甚至优化不是基于成本的   |
| 查询调试和分析     | 便于查看的查询执行计划、查询执行统计信息、说明性错误信息  | OOM问题和Java堆转储分析，GC在群集组件上暂停，每个任务的单独日志给你很多有趣的时间 |
| 技术价格           | 每个节点数十万美元                                        | 每个节点免费或高达数千美元                                   |
| 易用性             | 简单友好的SQL界面和简单可编译的数据库内功的数据库内置函数 | SQL不完全符合ANSI标准，用户应该关心执行逻辑，底层数据布局。函数通常需要用Java编写，编译并放在集群上 |
| 目标用户           | 业务分析师                                                | Java开发人员和经验丰富的DBA                                  |
| 单作业冗余         | 低，当MPP节点发生故障时作业失败                           | 高，作业只有当节点管理作业执行失败时才会失败                 |
| 最小数据集         | 任意                                                      | GB                                                           |
| 最大并发性         | 十到数百个查询                                            | 多达10-20个job                                               |
| 技术可扩展性       | 仅使用供应商提供的工具                                    | 混搭                                                         |
| DBA技能等级要求    | 普通DBA                                                   | 需要懂Java编程的高级RDBMSDBA                                 |
| 解决方案实施复杂性 | 一般                                                      | 复杂                                                         |

> 通过这些信息，您可以总结为什么Hadoop不能用作传统企业数据仓库的完全替代品，
>
> 但它可以用作以分布式方式处理大量数据并从数据中获得重要见解的引擎。 Facebook 拥有一个300PB的Hadoop，并且仍然使用一个小型的50TB Vertica集群，LinkedIn 拥有一个巨大的Hadoop集群，并且仍然使用Aster Data集群（由Teradata购买的MPP）， 您可以继续使用此列表。

#### 基本术语

- 在Elasticsearch中存储数据的行为就叫做**索引(indexing)**

- 在Elasticsearch中，文档归属于一种**类型(type)**,而这些类型存在于**索引(index)**中，我们可以画一些简单的对比图来类比传统关系型数据库

  ```
  Relational DB -> Databases -> Tables -> Rows   -> Columns
  Elasticsearch -> Indices   -> Types  -> Documents -> Fields
  ```

- Elasticsearch集群可以包含多个**索引(indices)**（数据库）

- 每一个**索引**可以包含多个**类型(types)**（表）

- 每一个**类型**包含多个**文档(documents)**（行）

- 每个**文档**包含多个**字段(Fields)**（列）

#### 索引的区分

- 索引（名词） 如上文所述，一个**索引(index)\**就像是传统关系数据库中的\**数据库**，它是相关文档存储的地方，index的复数是**indices** 或**indexes**
- 索引（动词） **「索引一个文档」**表示把一个文档存储到**索引（名词）**里，以便它可以被检索或者查询。这很像SQL中的`INSERT`关键字，差别是，如果文档已经存在，新的文档将覆盖旧的文档
- 倒排索引 传统数据库为特定列增加一个索引，例如B-Tree索引来加速检索。Elasticsearch和Lucene使用一种叫做**倒排索引(inverted index)**的数据结构来达到相同目的。

#### 索引(建库)示例

- 所以为了创建员工目录，我们将进行如下操作：
  - 为每个员工的**文档(document)**建立索引，每个文档包含了相应员工的所有信息。
  - 每个文档的类型为`employee`。
  - `employee`类型归属于索引`megacorp`。
  - `megacorp`索引存储在Elasticsearch集群中。

```
PUT /megacorp/employee/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
PUT /megacorp/employee/2
{
    "first_name" :  "Jane",
    "last_name" :   "Smith",
    "age" :         32,
    "about" :       "I like to collect rock albums",
    "interests":  [ "music" ]
}

PUT /megacorp/employee/3
{
    "first_name" :  "Douglas",
    "last_name" :   "Fir",
    "age" :         35,
    "about":        "I like to build cabinets",
    "interests":  [ "forestry" ]
}
```

**path:/megacorp/employee/1分析:**

| 名字     | 说明                   |
| -------- | ---------------------- |
| megacorp | 索引名（数据库）       |
| employee | 类型名（表）           |
| 1        | 员工id（文档，列，id） |

#### 地理位置格式

```
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "location": {
          "type": "geo_point"
        }
      }
    }
  }
}

PUT my_index/_doc/1
{
  "text": "Geo-point as an object",
  "location": { 
    "lat": 41.12,
    "lon": -71.34
  }
}

PUT my_index/_doc/2
{
  "text": "Geo-point as a string",
  "location": "41.12,-71.34" 
}
```

#### 简单搜索

**检索单个员工的信息：**

- 执行HTTP GET请求并指出文档的“地址”——索引、类型和ID既可

  ```
  GET /megacorp/employee/1
  ```

- 响应的内容中包含一些文档的元信息，John Smith的原始JSON文档包含在

  **_source**字段中

  ```
  {
    "_index" :   "megacorp",
    "_type" :    "employee",
    "_id" :      "1",
    "_version" : 1,
    "found" :    true,
    "_source" :  {
        "first_name" :  "John",
        "last_name" :   "Smith",
        "age" :         25,
        "about" :       "I love to go rock climbing",
        "interests":  [ "sports", "music" ]
    }
  }
  ```

  > 我们通过HTTP方法`GET`来检索文档，
  >
  > 我们可以使用`DELETE`方法删除文档，
  >
  > 使用`HEAD`方法检查某文档是否存在。
  >
  > 如果想更新已存在的文档，我们只需再`PUT`一次。

- **搜索全部信息**

 `GET /megacorp/employee/_search`

- **搜索姓氏中包含“Smith”的员工:**

 `GET /megacorp/employee/_search?q=last_name:Smith`

#### 使用DSL语句查询

 Elasticsearch提供丰富且灵活的查询语言叫做**DSL查询(Query DSL)**,它允许你构建更加`复杂`、`强大`的查询。

##### 映射

###### 空值

```
{
  "mappings": {
    "_doc": {
      "properties": {
        "status_code": {
          "type":       "keyword",
          "null_value": "NULL" 
        }
      }
    }
  }
}
```

##### DSL语法

**DSL(Domain Specific Language特定领域语言)**以JSON请求体的形式出现。我们可以这样表示之前关于“Smith”的查询:

```
`GET /megacorp/employee/_search`

{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```

> 这个请求体使用JSON表示，其中使用了`match`语句（查询类型之一）。

- **我们依旧想要找到姓氏为“Smith”的员工，年龄大于30岁的员工**。我们的语句将添加**过滤器(filter)**,它使得我们高效率的执行一个结构化搜索：

  ```
  `GET /megacorp/employee/_search`
  {
      "query" : {
          "filtered" : {
              "filter" : {
                  "range" : {
                      "age" : { "gt" : 30 } <1>
                  }
              },
              "query" : {
                  "match" : {
                      "last_name" : "smith" <2>
                  }
              }
          }
      }
  }
  ```

  > - <1> 这部分查询属于**区间过滤器(range filter)**,它用于查找所有年龄大于30岁的数据——`gt`为"greater than"的缩写。
  > - <2> 这部分查询与之前的`match`**语句(query)**一致。

##### **全文搜索匹配度：**

```
{
     "last_name":   "John Smith",
     "about":       "I love to go rock climbing",
},
{
     "last_name":   "Jane Smith",
     "about":       "I like to collect rock albums",
}
```

默认情况下，Elasticsearch根据结果相关性评分来对结果集进行排序，所谓的「结果相关性评分」就是文档与查询条件的匹配程度。很显然，排名第一的`John Smith`的`about`字段明确的写到**“rock climbing”**。

但是为什么`Jane Smith`也会出现在结果里呢？原因是**“rock”**在她的`abuot`字段中被提及了。因为只有**“rock”**被提及而**“climbing”**没有，所以她的`_score`要低于John。

这个例子很好的解释了Elasticsearch如何在各种文本字段中进行全文搜索，并且返回相关性最大的结果集。**相关性(relevance)**的概念在Elasticsearch中非常重要，而这个概念在传统关系型数据库中是不可想象的，因为传统数据库对记录的查询只有匹配或者不匹配。

------

##### 删除

语法：indexname/type/_delete_by_query

请求方式：post

dsl:

```
{
  "query": {
    "match_all": {}
  }
}
```

示例：`basisdata_usr_login_info-201905/information/_delete_by_query`

------

##### 短语搜索

目前我们可以在字段中搜索单独的一个词，这挺好的，但是有时候你想要确切的匹配若干个单词或者**短语(phrases)**。例如我们想要查询同时包含"rock"和"climbing"（并且是相邻的）的员工记录。

要做到这个，我们只要将`match`查询变更为`match_phrase`查询即可:

------

##### 高亮搜索

从每个搜索结果中**高亮(highlight)**匹配到的关键字，这样用户可以知道为什么这些文档和查询相匹配。在Elasticsearch中高亮片段是非常容易的。

```
`GET /megacorp/employee/_search`
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}
```

当我们运行这个语句时，会命中与之前相同的结果，但是在返回结果中会有一个新的部分叫做`highlight`，这里包含了来自`about`字段中的文本，并且用`<em></em>`来标识匹配到的单词。

```
{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.23013961,
      "hits": [
         {
            ...
            "_score":         0.23013961,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            },
            "highlight": {
               "about": [
                  "I love to go <em>rock</em> <em>climbing</em>" <1>
               ]
            }
         }
      ]
   }
}
```

> - <1> 原有文本中高亮的片段

------

##### 聚合

允许管理者在职员目录中进行一些分析。 Elasticsearch有一个功能叫做**聚合(aggregations)**，它允许你在数据上生成复杂的分析统计。它很像SQL中的`GROUP BY`但是功能更强大。

```
`GET /megacorp/employee/_search`
{
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests" }
    }
  }
}
```

暂时先忽略语法只看查询结果：

```
{
   ...
   "hits": { ... },
   "aggregations": {
      "all_interests": {
         "buckets": [
            {
               "key":       "music",
               "doc_count": 2
            },
            {
               "key":       "forestry",
               "doc_count": 1
            },
            {
               "key":       "sports",
               "doc_count": 1
            }
         ]
      }
   }
}
```

where条件+group by 形式查询

```
`GET /megacorp/employee/_search`
{
  "query": {
    "match": {
      "last_name": "smith"
    }
  },
  "aggs": {
    "all_interests": {
      "terms": {
        "field": "interests"
      }
    }
  }
}
```

结果：

```
"all_interests": {
     "buckets": [
        {
           "key": "music",
           "doc_count": 2
        },
        {
           "key": "sports",
           "doc_count": 1
        }
     ]
  }
```

聚合也允许分级汇总。例如，让我们统计每种兴趣下职员的平均年龄：

```
`GET /megacorp/employee/_search`
{
    "aggs" : {
        "all_interests" : {
            "terms" : { "field" : "interests" },
            "aggs" : {
                "avg_age" : {
                    "avg" : { "field" : "age" }
                }
            }
        }
    }
}
```

虽然这次返回的聚合结果有些复杂，但任然很容易理解：

```
"all_interests": {
     "buckets": [
        {
           "key": "music",
           "doc_count": 2,
           "avg_age": {
              "value": 28.5
           }
        },
        {
           "key": "forestry",
           "doc_count": 1,
           "avg_age": {
              "value": 35
           }
        },
        {
           "key": "sports",
           "doc_count": 1,
           "avg_age": {
              "value": 25
           }
        }
     ]
  }
```

##### 分页

和SQL使用`LIMIT`关键字返回只有一页的结果一样，Elasticsearch接受`from`和`size`参数：

```
size: 结果数，默认`10`
from: 跳过开始的结果数，默认`0`

GET /_search?size=5
GET /_search?size=5&from=5
GET /_search?size=5&from=10
```

#### 最重要的查询过滤语句

Elasticsearch 提供了丰富的查询过滤语句，而有一些是我们较常用到的。 我们将会在后续的《深入搜索》中展开讨论，现在我们快速的介绍一下 这些最常用到的查询过滤语句。

##### 组合过滤条件查询DSL

`bool` 过滤可以用来合并多个过滤条件查询结果的布尔逻辑，它包含一下操作符：

`must` :: 多个查询条件的完全匹配,相当于 `and`。

`must_not` :: 多个查询条件的相反匹配，相当于 `not`。

`should` :: 至少有一个查询条件匹配, 相当于 `or`。

这些参数可以分别继承一个过滤条件或者一个过滤条件的数组：

```
#多条件组合
{
    "bool": {
        "must":     { "term": { "folder": "inbox" }},
        "must_not": { "term": { "tag":    "spam"  }},
        "should": [
                    { "term": { "starred": true   }},
                    { "term": { "unread":  true   }}
        ]
    }
}
#例如es6.5版本
{
  "from": 1, 
  "size": 2, 
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "DevName" : "输入IO2"
          }
        },
        {
          "range": {
            "LogTime": {
              "gte": "2019-06-15",
              "lte": "2019-06-25"
            }
          }
        }
      ]
    }
  }
}
#例如es2.1版本
{
	"query": {
		"bool": {
			"filter": [
				{
					"bool": {
						"must": [
							{
								"term": {
									"PLATENUMBER": "粤S625BG"
								}
							},
							{
								"term": {
									"PLATETYPE": "02"
								}
							},
							{
								"range": {
									"DATEPART": {
										"from": 20190601,
										"to": 20190708
									}
								}
							}
						]
					}
				}
			]
		}
	},
	"aggs": {
		"group": {
			"terms": {
				"field": "PASSPORTID",
				"size": 10   #只查前10个
			}
		}
	},
	"size": 0
}
```

##### match查询

`match`查询是一个标准查询，不管你需要全文本查询还是精确查询基本上都要用到它。

如果你使用 `match` 查询一个全文本字段，它会在真正查询之前用分析器先分析`match`一下查询字符：

```
{
    "match": {
        "tweet": "About Search"
    }
}
```

如果用`match`下指定了一个确切值，在遇到数字，日期，布尔值或者`not_analyzed` 的字符串时，它将为你搜索你给定的值：

```
{ "match": { "age":    26           }}
{ "match": { "date":   "2014-09-01" }}
{ "match": { "public": true         }}
{ "match": { "tag":    "full_text"  }}
```

> **提示**： 做精确匹配搜索时，你最好用过滤语句，因为过滤语句可以缓存数据。

### DSL

##### 过滤日期(搜索)

- 查询时间区间的数据

- ```
  //GET log_alarmevent_topic/information/_search
  {   
      "size": 100,   
      "query": {  
          "range": {
              "LogTime": {
                  "gte": "2018-07-04",
                  "lt": "2019-07-05"
              }
           }
       }
  }
  ```

##### 建设日期Mapping

- ```
  //POST indexname/indextype/_mapping
  {
      "properties": {
          "UserIds": {
  			"type": "keyword"
  		},
          "CreateTime": {
  			"format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis",
  			"type": "date"
  		}
      }
  }
  ```

##### 更换索引reindex

- ```
  #POST _reindex
  {
      "conflicts": "proceed", // 将冲突进行类似于continue的操作
      "source": {
          "remote": {
              "host": "http://otherhost:9200", // 远程es的ip和port列表
              "socket_timeout": "1m",
              "connect_timeout": "10s"  // 超时时间设置
          },
          "index": "my_index_name", // 源索引名称
          "query": {         // 满足条件的数据
              "match": {
                  "test": "data"
              }
          }
      },
    	"dest": {
      	"index": "index_to",
          "op_type": "create" //只会对发生不同的document进行reindex
    	}
  }
  ```

  > 重新建设mapping,数据会自动同步，类型也可以转换
  >
  > 具体使用：https://blog.csdn.net/ctwy291314/article/details/82734667

##### 建设别名

- 为索引secisland添加别名secisland_alias：

  ```
  #POST _aliases
  {
    "actions": [
      {
        "add": {
          "index": "indexName",
          "alias": "indexName_alias"
        }
      }
    ]
  }
  ```

- 删除别名：

  ```
  #POST _aliases
  {
    "actions": [
      {
        "remove": {
          "index": "indexName",
          "alias": "indexName_alias"
        }
      }
    ]
  }
  ```

##### 一个别名关联多个索引：

- ```
  #POST _aliases
  {
    "actions": [
      {
        "remove": {
          "index": "secisland",
          "alias": "secisland_alias"
        }
      },
      {
        "add": {
          "index": "secisland",
          "alias": "secisland_alias"
        }
      },
      {
        "add": {
          "index": "secisland2",
          "alias": "secisland_alias"
        }
      }
    ]
  }
  ```

##### 过滤索引别名：

- 通过过滤索引来指定别名提供了对索引查看的不同视图，该过滤器可以使用查询DSL来定义适用于所有的搜索，计算，查询，删除等，以及更多类似这样的与此别名类似的操作。

- 如以下索引：

  ```
  # 现有如下索引的映射
  {
      "index3":{
          "mappings":{
              "properties":{
                  "city":{
                      "type":"keyword"
                  },
                  "createTime":{
                      "type":"date"
                  },
                  "message":{
                      "type":"text"
                  }
                  
              }
          }
      }
  }
  
  # post _aliases --创建过滤索引
  {
      "actions":[
          {
              "add":{
                  "index":"index3",
                  "alias":"index3_alias_beijing",
                  "filter":{
                      "term":{
                          "city":"beijing"
                      }
                  }
              }
          }
      ]
  }
  ```

- ## 删除别名

  DELETE secisland3/_alias/secisland3_alias_beijing

  或者：

  DELETE secisland3/_alias/*

##### painless脚本

```
{
  "query": {
    "range": {
      "PASSTIME": {
        "gte": "2020-03-14 15:04:08",
        "lte": "2020-03-18 15:04:08"
      }
    }
  },
  "script_fields": {
    "rangepasstime": {
      "script": {
        "lang": "painless",
        "source":"doc['ID'].value * params.factor",
        "params": {
          "factor":1
        }
      }
    }
  }
}
```

### es优化

https://www.cnblogs.com/technologykai/articles/10899582.html

https://www.jianshu.com/p/93f25c1f138c

#### 分片设置多少

forcemerge

**注1**：小的分片会造成小的分段，从而会增加开销。我们的目的是将平均分片大小控制在几 GB 到几十 GB 之间。对于基于时间的数据的使用场景来说，通常将分片大小控制在 20GB 到 40GB 之间。

**注2**：由于每个分片的开销取决于分段的数量和大小，因此通过 **forcemerge** 操作强制将较小的分段合并为较大的分段，这样可以减少开销并提高查询性能。 理想情况下，一旦不再向索引写入数据，就应该这样做。 请注意，这是一项比较耗费性能和开销的操作，因此[应该在非高峰时段](https://github.com/JGMa-java/elasticsearch-notes/blob/master)执行。

**注3**：我们可以在[节点上保留的分片数量](https://github.com/JGMa-java/elasticsearch-notes/blob/master)与可用的[堆内存](https://github.com/JGMa-java/elasticsearch-notes/blob/master)成正比，但 Elasticsearch 没有强制的固定限制。 一个好的经验法则是确保每个节点的分片数量低于每GB堆内存配置20到25个分片。 因此，具有30GB堆内存的节点应该具有最多600-750个分片，但是低于该限制可以使其保持更好。 这通常有助于集群保持健康。

**注4**：如果担心数据的快速增长, 建议根据这条限制: [ElasticSearch推荐的最大JVM堆空间](https://www.elastic.co/guide/en/elasticsearch/guide/current/relevance-intro.html) 是 30~32G, 所以把分片最大容量限制为 30GB, 然后再对分片数量做合理估算。例如, 如果的数据能达到 200GB, 则最多分配7到8个分片。

数据分片也是要有相应资源消耗,并且需要持续投入。当索引拥有较多分片时, 为了组装查询结果, ES 必须单独查询每个分片(当然并行的方式)并对结果进行合并。所以高性能 IO 设备(SSDs)和多核处理器无疑对分片性能会有巨大帮助。尽管如此, 还是要多关心数据本身的大小,更新频率以及未来的状态。在分片分配上并没有绝对的答案。

#### 集群数目(jvm分配)

https://www.jianshu.com/p/93f25c1f138c

1. jvm heap分配 （官方jvm堆：30G）

2. 将机器上少于一半的内存分配给es

3. 不要给jvm分配超过32G内存

   > ****一旦超过了这个神奇的30-32GB的边界，指针会切换会普通对象指针（oop）****所以也正是因为32G的限制，一般来说，都是建议说，如果你的es要处理的数据量上亿的话，几亿，或者十亿以内的规模的话，建议，就是用64G的内存的机器比较合适，有个5台，差不多也够了。给jvm heap分配32G，留下32G给os cache。

4. 在32G以内的话具体应该设置heap为多大？

5. 对于有1TB内存的超大内存机器该如何分配？

   > 如果我们的机器是一台超级服务器，内存资源甚至达到了1TB，或者512G，128G，该怎么办？首先es官方是建议避免用这种超级服务器来部署es集群的，但是如果我们只有这种机器可以用的话，我们要考虑以下几点：
   >
   > 1. 我们是否在做大量的全文检索？考虑一下分配4~32G的内存给es进程，同时给lucene留下其余所有的内存用来做os filesystem cache。所有的剩余的内存都会用来cache segment file，而且可以提供非常高性能的搜索，几乎所有的数据都是可以在内存中缓存的，es集群的性能会非常高
   > 2. 是否在做大量的排序或者聚合操作？聚合操作是不是针对数字、日期或者[未分词](https://github.com/JGMa-java/elasticsearch-notes/blob/master)的string？如果是的化，那么还是给es 4~32G的内存即可，其他的留给es filesystem cache，可以将聚合好用的正排索引，doc values放在os cache中
   > 3. 如果在[针对分词的string](https://github.com/JGMa-java/elasticsearch-notes/blob/master)做大量的排序或聚合操作？如果是的化，那么就需要使用fielddata，这就得给jvm heap分配更大的内存空间。此时[不建议运行一个节点在机器上](https://github.com/JGMa-java/elasticsearch-notes/blob/master)，而是运行多个节点在一台机器上，那么如果我们的服务器有128G的内存，可以运行两个es节点，然后每个节点分配32G的内存，剩下64G留给os cache。

6. swapping

   > swap（交换区）是性能终结者
   >
   > https://blog.csdn.net/u013063153/article/details/60876897
   >
   > 如果两种方法都不可用，你应该在ElasticSearch的配置中启用mlockall.file。这允许JVM锁定其使用的内存，而避免被放入操作系统交换区。
   >
   > 在elasticsearch.yml中，做如下设置：
   >
   > ```
   > bootstrap.mlockall: true 
   > ```

机器性能：内存64G

单个节点：32G -> 3 * 32G = 96G

#### scroll

- **Scroll-Scan**：Scroll 是先做一次初始化搜索把所有符合搜索条件的结果[缓存起来生成一个快照](https://github.com/JGMa-java/elasticsearch-notes/blob/master)，然后持续地、批量地[从快照里拉取数据](https://github.com/JGMa-java/elasticsearch-notes/blob/master)直到没有数据剩下。

  而这时对索引数据的插入、删除、更新都不会影响遍历结果，因此 Scroll 并[不适合用来做实时搜索](https://github.com/JGMa-java/elasticsearch-notes/blob/master)。

- 其思路和使用方式与 Scroll 非常相似，但是 Scroll-Scan 关闭了 Scroll 中最耗时的[文本相似度计算和排序](https://github.com/JGMa-java/elasticsearch-notes/blob/master)，使得性能更加高效

> 注意：Elasticsearch 2.1.0 版本之后移除了 search_type=scan，使用 "sort": [ "_doc"] 进行代替。

**Scroll** 和 **Scroll-Scan** 有一些差别，如下所示：

- Scroll-Scan不进行文本相似度计算，不排序，按照索引中的数据顺序返回。
- Scroll-Scan 不支持聚合操作。
- Scroll-Scan 的参数 Size 代表着每个分片上的请求的结果数量，每次返回 n×size 条数据。而 Scroll 每次返回 size 条数据。

#### **角色隔离和脑裂**

ES 集群中的数据节点负责对数据进行增、删、改、查和聚合等操作，所以对 CPU、内存和 I/O 的消耗很大。

在搭建 ES 集群时，我们应该对 ES 集群中的节点进行角色划分和隔离。

- **候选主节点：**

```
node.master=true
node.data=false
```

- **数据节点：**

```
node.master=false
node.data=true
```

- **避免脑裂**

网络异常可能会导致集群中节点划分出多个区域，区域发现没有 Master 节点的时候，会选举出了自己区域内 Maste 节点 r，导致一个集群被分裂为多个集群，使集群之间的数据无法同步，我们称这种现象为脑裂。

为了防止脑裂，我们需要在 Master 节点的配置文件中添加如下参数：

```
discovery.zen.minimum_master_nodes=（master_eligible_nodes/2）+1 //默认值为1
```

其中 master_eligible_nodes 为 Master 集群中的节点数。这样做可以避免脑裂的现象都出现，最大限度地提升集群的高可用性。

只要不少于 discovery.zen.minimum_master_nodes 个候选节点存活，选举工作就可以顺利进行。

#### **ES 配置说明**

下面我们列出一些比较重要的配置信息：

- **cluster.name：elasticsearch：**配置 ES 的集群名称，默认值是 ES，建议改成与所存数据相关的名称，ES 会自动发现在同一网段下的集群名称相同的节点。

- **node.nam： "node1"：**集群中的节点名，在同一个集群中不能重复。节点的名称一旦设置，就不能再改变了。当然，也可以设置成服务器的主机名称，例如 node.name:${HOSTNAME}。

- **noed.master：true：**指定该节点是否有资格被选举成为 Master 节点，默认是 True，如果被设置为 True，则只是有资格成为 Master 节点，具体能否成为 Master 节点，需要通过选举产生。

- **node.data：true：**指定该节点是否存储索引数据，默认为 True。数据的增、删、改、查都是在 Data 节点完成的。

- **index.number_of_shards：5：**设置都索引分片个数，默认是 5 片。也可以在创建索引时设置该值，具体设置为多大都值要根据数据量的大小来定。如果数据量不大，则设置成 1 时效率最高。

- **index.number_of_replicas：1：**设置默认的索引副本个数，默认为 1 个。副本数越多，集群的可用性越好，但是写索引时需要同步的数据越多。

- **path.conf：/path/to/conf：**设置配置文件的存储路径，默认是 ES 目录下的 Conf 文件夹。建议使用默认值。

- **path.data：/path/to/data1,/path/to/data2：**设置索引数据多存储路径，默认是 ES 根目录下的 Data 文件夹。切记不要使用默认值，因为若 ES 进行了升级，则有可能数据全部丢失。

  可以用半角逗号隔开设置的多个存储路径，在多硬盘的服务器上设置多个存储路径是很有必要的。

- **path.logs：/path/to/logs：**设置日志文件的存储路径，默认是 ES 根目录下的 Logs，建议修改到其他地方。

- **path.plugins：/path/to/plugins：**设置第三方插件的存放路径，默认是 ES 根目录下的 Plugins 文件夹。

- **bootstrap.mlockall：true：**设置为 True 时可锁住内存。因为当 JVM 开始 Swap 时，ES 的效率会降低，所以要保证它不 Swap。

- **network.bind_host：192.168.0.1：**设置本节点绑定的 IP 地址，IP 地址类型是 IPv4 或 IPv6，默认为 0.0.0.0。

- **network.publish_host：192.168.0.1：**设置其他节点和该节点交互的 IP 地址，如果不设置，则会进行自我判断。

- **network.host：192.168.0.1：**用于同时设置 bind_host 和 publish_host 这两个参数。

- **http.port：9200：**设置对外服务的 HTTP 端口，默认为 9200。ES 的节点需要配置两个端口号，一个对外提供服务的端口号，一个是集群内部使用的端口号。

  http.port 设置的是对外提供服务的端口号。注意，如果在一个服务器上配置多个节点，则切记对端口号进行区分。

- **transport.tcp.port：9300：**设置集群内部的节点间交互的 TCP 端口，默认是 9300。注意，如果在一个服务器配置多个节点，则切记对端口号进行区分。

- **transport.tcp.compress：true：**设置在节点间传输数据时是否压缩，默认为 False，不压缩。

- **discovery.zen.minimum_master_nodes：1：**设置在选举 Master 节点时需要参与的最少的候选主节点数，默认为 1。如果使用默认值，则当网络不稳定时有可能会出现脑裂。

  合理的数值为(master_eligible_nodes/2)+1，其中 master_eligible_nodes 表示集群中的候选主节点数。

- **discovery.zen.ping.timeout：3s：**设置在集群中自动发现其他节点时 Ping 连接的超时时间，默认为 3 秒。

  在较差的网络环境下需要设置得大一点，防止因误判该节点的存活状态而导致分片的转移。

- **action.destructive_requires_name: true：**进行全部索引删除是很危险的，我们可以通过在配置文件中添加下面的配置信息，来关闭使用 _all 和使用通配符删除索引的接口，使用删除索引职能通过索引的全称进行。

#### ElasticSearch分片不均匀，集群负载不均衡(并发大)

https://blog.csdn.net/qq_20545159/article/details/80549335

xpack参考

https://www.cnblogs.com/WSPJJ/articles/11121138.html

#### xpack集群加密，客户端链接加密后的集群，kibana操作

[elasticsearch6.5.0(es6.5)加密xpack,java客户端访问xpack集群SSL_JGMa_TiMo的博客-CSDN博客_elasticsearch6 加密](https://blog.csdn.net/JGMa_TiMo/article/details/97396313)

## 介绍

ElasticSearch是基于Lucene的==搜索服务器==（可以启动的web应用），可以提供大量的客户端使用HTTP协议访问

![](https://note.youdao.com/yws/api/personal/file/65DF23EBFF854287818835CD6C171D8C?method=download&shareKey=93bb78924a1988c2497e793692e70486)

## REST风格

> REST是HTTP协议创始人在2000年一篇论文上，提出的一种对HTTP协议使用的风格建议



### url地址表示资源

> 同类资源可以用路径中的不同数据表示

访问一个用户数据：

- 平时：`http://localhost/user/manage/query?username=eeee`，不满足REST风格，没有体现url是资源的概念

- REST风格

  `http://localhost:8080/user/manage/query/eeee`

  `http://localhost:8080/user/manage/query/admin`

  `http://localhost:8080/user/manage/query/haha`

  `http://localhost:8080/user/manage/query/adsfc`



### HTTP请求方式表示操作

> 对同一个资源有多种不同的操作

- 平时

  `/user/manage/saveUsername` 表示新增

  `/user/manage/saveUserEmail`

  `/user/manage/query` 表示查询

  `/user/manage/update` 表示更新

  ==三个url地址，可能操作同一个用户（资源）==

- REST风格

  - put请求：`/user/manage/{userId}` 请求体参数  表示新增
  - get请求：`/user/manage/{userId}` 请求体参数  表示查询
  - delete请求：表示删除
  - post请求：表示更新

## SpringMVC支持REST风格

```java
//支持REST风格

@RequestMapping(value="haha/{id}",method = RequestMethod.DELETE)
public void delete(){ }

@RequestMapping(value="haha/{id}",method = RequestMethod.PUT)
public void save(){ }

@RequestMapping(value="haha/{id}",method = RequestMethod.POST)
public void update(){ }
```

> 1. url定义资源，路径参数必定存在 
> 2. 请求方式定义了操作put新增，post更新，delete删除，get查询 



## 不是所有请求都必须满足REST风格

> 微服务集群中，很多情况下，对用一个资源除了增删查改之外，还有很多细小的功能请求
>
> 比如：查username、userId、userJson等
>
> 一般在这种情况下，还是需要很多自定义的url请求



## ElasticSearch

> 请求url采用REST风格，给客户端使用

**原因**：如果url不采用REST风格，而是随意在后面拼接，各种操作多达20000url时就记不住了

**特点**：

- ES天生就是分布式高可用的集群
- 分布式：索引被切分成5份，每一份都是一个完整的索引文件结构，大量document只要做分片计算，在对应不同的分片文件夹中存储，就能实现分布式
  - N个分片，文件夹名称: 0-N-1
  - hash取余算法，对docId唯一值做取余，找到对应分片进行存储
- 高可用：每一个切分出来的分片，都会计算输出一个从分片



# ElasticSearch基本结构

==存储层==：存储生成的索引、索引中的文档数据、倒排索引表数据

==功能扩展==：基于Lucene实现了索引文件的管理（创建、搜索），还实现了分布式、实现了搭建集群的各种功能

==用户协议层==：HTTP协议的REST风格，给所有的客户端提供统一的访问功能

![](https://note.youdao.com/yws/api/personal/file/8CBE615F28B54F56AE1C4E9F79B95AEF?method=download&shareKey=78b44977082c263b00822bbad995cb85)



# REST测试工具

## postman

> 自己的账号：17603393007@163.com；密码：sxh19970225

![](https://note.youdao.com/yws/api/personal/file/30E70075D1554A52ACD97F70172FE4F5?method=download&shareKey=9d12c5d50c3755458d80ae4f811d4563)

## 火狐插件：RESTCLIENT

![](https://note.youdao.com/yws/api/personal/file/B5A6B6BE43D24106AADD0D6B31EF71E8?method=download&shareKey=57f22bced700c1a077cc8dba6b5c9ad6)



# ElasticSearch安装配置

## 安装

> 安装到3台云主机上，搭建ES的高可用分布式集群

1. 解压安装包到`/home/software/`下

   ```sh
   [root@10-9-104-184 ~]# tar -xf /home/resources/elasticsearch-5.5.2.tar.gz -C /home/software/
   ```

2. 目录结构

   - bin：启动命令所在文件夹
   - config：核心配置文件`elasticsearch`所在目录
   - plugins：ik分词器插件等

3. 更换用户和用户组

   > ES对安全性有要求，不允许root用户直接启动。
   >
   > - 需要注册一个普通用户：es
   > - 并把用户和用户组赋给该普通用户

   ```sh
   [root@10-9-104-184 software]# chown -R es:es /home/software/elasticsearch-5.5.2/
   ```

   > `-R`：递归处理，表示将后面文件夹中的所有文件的用户和用户组都改为es



## 配置和启动

> 即使使用es普通用户，进入bin目录执行elasticsearch命令启动，也会出现各种问题
>
> 问题之一就是ES默认只允许localhost客户端访问，不允许远程访问，需要修改配置文件



### 修改`elasticsearch.yml`配置文件

1. 修改集群名称

   > ==17行，集群名称==。是ES启动整个集群时使用的名称标识

   ![](https://note.youdao.com/yws/api/personal/file/7D038E5491954591A0F9F0C7E2678B9F?method=download&shareKey=77477e4357a27e0714ee83b6c2ef0d40)

2. 修改节点名称

   > ==23行，节点名称。==
   >
   > 每一个集群中的ES节点，都应该有一个唯一的名称。
   >
   > （如果注释掉，每次启动都会分配一个随机的名字）

   ![](https://note.youdao.com/yws/api/personal/file/5543C411C66F452D8524956F8E314DE1?method=download&shareKey=d114519e2ff5305e4142d5b69054517f)

3. 关闭bootstrap

   > ==43行，bootstrap相关==
   >
   > CentOS 6.5没有安装bootstrap相关插件，所以ES启动时如果要求开启bootstrap，会报错，但是不影响使用

   ![](https://note.youdao.com/yws/api/personal/file/4CF97226D0DA434E8D7C437CC1C1CA38?method=download&shareKey=c066ff0962eaa7457b96216da8a7cfc8)

4. 修改host地址

   > ==56行，host地址==
   >
   > 默认使用localhost，只允许本地请求访问到ES节点，需要修改成服务器ip地址，允许远程访问

   ![](https://note.youdao.com/yws/api/personal/file/BCD15C5051BA481E8E68DC1EF1C8469C?method=download&shareKey=3e88a2cb26181aa641104d475a73ef9f)

5. 打开http协议端口

   > ==60行，http协议端口==
   >
   > 还有一个TCP/IP协议的Java语言访问的端口9300

   ![](https://note.youdao.com/yws/api/personal/file/9F19CAC9CA39462BB695954AC654C93C?method=download&shareKey=f6d102dfddf7d4e778d2ca3334e00f68)

6. 开启http访问插件权限

   > 在文件末尾添加一个http访问插件权限 
   >
   > 后续会开启一个`head-master`插件，视图化展示es中的数据。需要es开启这个权限。

   ![](https://note.youdao.com/yws/api/personal/file/C60CAF31F4C846E091E38E51E30B9FE6?method=download&shareKey=3b3aa47e35edba425c9751ff93cfc5ed)

7. 搭建集群时，需要多配置2行内容

   - 一个是：协调器
   - 一个是：最小有效master数量，防止脑裂出现



### 启动

> 进入到bin文件夹，使用普通用户执行启动命令

```sh
[root@10-9-104-184 software]# su es
[es@10-9-104-184 bin]$ elasticsearch
```

> ` elasticsearch -d` ：可以后台运行elasticsearch





# 操作ES

> 可以对启动的ES实现命令操作，本质就是HTTP请求，满足REST风格

## ES中的数据管理结构

> 与数据库结构对比

| mysql数据库  | elasticsearch全文检索 |
| ------------ | --------------------- |
| database库   | index索引             |
| table表格    | ==type类型==          |
| rows行数据   | document文档          |
| column列数据 | field域属性           |



## 添加一个索引文件

> 新增了一个叫做index01的索引文件到ES节点中

```sh
[root@10-9-104-184 software]# curl -XPUT http://10.9.104.184:9200/index01
```

> `curl`：linux提供的一个支持HTTP协议的命令脚本，有很多选项可以携带：HTTP协议的头、参数等
>
> - `-X`：后面跟着这次请求的方式（PUT，DELETE，GET，POST）
> - `-d`：请求体的内容
> - `-h`：请求头的内容



## ES创建的索引结构

在Linux调用curl命令创建一个索引文件后，会返回一个json字符串

```json
{
  "acknowledged": true, 操作确认
  "shards_acknowledged": true 分片成功
}
```

> 索引文件创建后，这个索引文件初始是空的

![](https://note.youdao.com/yws/api/personal/file/F9A6B0671BDE49EE98AA5B6A5FF3FD15?method=download&shareKey=acd5fb66fe754f783bbda417b801309f)

> 开始创建的索引文件叫：index01，但是ES创建成功后就不叫index01了，该文件名称是计算后的ID值，例如：`neShnvvZQyiT2-3BLXmYJA`
>
> 要知道index01对应的是哪个ID值的文件夹，可以通过curl命令获取index01的信息，在信息中可以查看

```sh
curl -XGET http://10.42.0.33:9200/index01
```

> 结果：
>
> ```json
> {
>  "index02": {
>      "aliases": {},//别名，可以通过别名操作索引
>      "mappings": {},//默认空的，一旦数据增加，mapping将会出现动态结构
>      "settings": {//索引的各种配置信息
>          "index": {
>              "creation_date": "1582533919910",
>              "number_of_shards": "5",
>              "number_of_replicas": "1",
>              "uuid": "neShnvvZQyiT2-3BLXmYJA",//就是当前索引id值
>              "version": {
>                  "created": "5050299"
>              },
>              "provided_name": "index02"
>          }
>      }
>  }
> }
> ```
>
> 



## ES的数据分布计算

> document数据一旦新增到某个索引中，会利用document中的唯一值`docId`计算hash取余（对分片个数取余），然后存到对应的分片中
>
> 注意：==一个索引文件被切分成多个分片后，不允许修改分片数量==（防止数据未命中）

![](https://note.youdao.com/yws/api/personal/file/A1AE81E382914EA19B70D50C4D02184D?method=download&shareKey=06bf5b2d452faf77233e7f62307dbfa8)



# ES的视图化插件：head

> 在启动ES之前配置的`elasticsearch.yml`文件最后一行：`http.cors.enabled: true`
>
> 修改`head`插件的配置文件`/home/presoftware/elasticsearch-head-master/Gruntfile.js`

```sh
[root@10-9-104-184 elasticsearch-head-master]# vim Gruntfile.js  
```

> 这是一个js文件，将==93行==的host改成`head`插件所在的**服务器ip地址**



## 启动

在`head`文件夹根目录执行启动命令

- `grunt server`：控制台运行
- `grunt server &`：后台运行

> 启动成功后，会在控制台打印访问head插件的url地址和端口号：`9100`



## 远程访问

> 通过浏览器输入head提示的ip和端口，访问head插件
>
> 可以通过head插件，连接ES，发送各种命令

在连接标签中，输入ES的地址：`http://ES服务器ip:9200`

![](https://note.youdao.com/yws/api/personal/file/E1520BC241B9422FB0DCCD0F6981B8D4?method=download&shareKey=24650bdc2346770901f6097d96f32168)

1. `es01`：节点名称，elasticsearch.yml配置的node-name值
2. ⭐：表示当前ES集群的master节点（与之前主从复制的master意义不同）
3. 0-4方框：每个索引切分的分片文件夹标号（取余的数组）
4. `Unassigned`：集群最小结构是2个，如果启动一个，认为另一个没有启动。从分配虽然计算出来了，但是没有第二个节点时是不会分配写数据的

[TOC]

# 索引管理



## 新增索引

> ES根据请求方式PUT，根据请求url地址中资源定义的indexName为索引名称，在ES中创建indexName的索引
>
> `curl -XPUT http://10.42.0.33:9200/{indexName}`

```sh
curl -XPUT http://10.42.0.33:9200/book
```

成功后返回响应

```json
{
    "acknowledged": true, //表示确认成功
    "shards_acknowledged": true //分片创建成功
}
```



## 删除索引

> 提供索引名称，使用delete请求，将索引数据从ES中删除
>
> `curl -XDELETE http://10.42.0.33:9200/{indexName}`

```sh
curl -XDELETE http://10.42.0.33:9200/book
```

成功后返回响应：

```json
{
  "acknowledged": true
}
```



## 查看索引

> 在ES中保存的索引文件，都会对应有各种状态存储对象信息，可以通过GET查询将这些信息返回。
>
> `aliases`别名、`mapping`映射、`settings`配置
>
> `curl -XGET http://10.9.104.184:9200/{indexName}`

```sh
curl -XGET http://10.9.104.184:9200/book
```

成功后返回响应：

```json
{
    "index01": {
        "aliases": {}, //别名，可以通过别名操作索引
        "mappings": {},//默认是空，一旦数据新增，mapping将会出现动态结构
        "settings": { //索引的各种配置信息
            "index": {
                "creation_date": "1582594169179",
                "number_of_shards": "5",
                "number_of_replicas": "1",
                "uuid": "sDnRylu6RXu2INuyDXctbA",
                "version": {
                    "created": "5050299"
                },
                "provided_name": "index01"
            }
        }
    }
}
```



## 关闭/打开索引

1. 关闭

   `curl -XPOST http://10.42.0.33:9200/{indexName}/_close`

2. 打开

   `curl -XPOST http://10.42.0.33:9200/{indexName}/_open`

> 索引文件在ES中创建后，会存在保存状态的持久化文件中，其中一个状态是：打开/关闭。
>
> 一旦将索引文件进行关闭操作，这个索引文件就不能再使用，直到再被打开

## 读写权限限制

> 可以对索引实现不能读的权限

```sh
curl -XPUT -d '{"blocks.read":true}' http://10.42.0.33:9200/{indexName}/_settings
```

`blocks.read:true`：阻止读

`blocks.read:false`：不阻止读



# 文档管理

> http协议访问ES时，新增文档数据操作时，一定要保证url的JSON字符串结构正确



## 新增(覆盖)文档

> 在已存在的索引中，通过url指定索引、类型和文档id，将文档数据作为请求体参数发送到ES中，进行处理

```sh
curl -XPUT -d '{"id":1,"title":"ES简介","content":"ES很好用"}' http://10.42.0.33:9200/{indexName}/{typeName}/{_docId}
```

成功后返回响应:

```json
//document对象的基础属性
{
    "_index": "book", //所属索引
    "_type": "article", //所属类型
    "_id": "1", //document的id值
    
    //版本号
    //随着对document的操作执行，
    //_version会自增处理。在分布式高可用的集群中，
    //用来判断哪些数据可用，哪些延迟
    "_version": 1, 
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "created": true
}
```



## 获取文档

> 使用GET请求，获取指定索引、类型、docId的文档数据

```sh
curl -XGET http://10.9.104.184:9200/{indexName}/{typeName}/{_docId}
```

成功后返回响应

```json
{
    "_index": "book",
    "_type": "article",
    "_id": "1",
    "_version": 3,
    "found": true,
    "_source": {  //source表示数据内容
        "id": "doc1",
        "title": "java编程思想",
        "content": "这是一本开发的圣经"
    }
}
```





## 删除文档

> 删除指定的索引名称、类型、docId的文档数据
>
> ==即使删除了数据，该数据的`_version`也会自增1

```sh
curl -XDELETE http://10.9.104.184:9200/{indexName}/{typeName}/{_docId}
```

成功后返回响应

```json
{
    "found": true,
    "_index": "book",
    "_type": "article",
    "_id": "1",
    "_version": 4,
    "result": "deleted",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    }
}
```



# 搜索功能

ES是基于Lucene的，本身也支持大量搜索功能：词项、布尔、多域、模糊等等。

使用http协议访问ES进行搜索时，只需要将JSON结构传递到不同的query实现类中即可

## match_all

> 查询所有

```sh
curl -XGET http://10.42.0.33:9200/index01/_search -d '{"query":{"match_all":{}}}'
```

返回查询结果：

![](https://note.youdao.com/yws/api/personal/file/18546201B8B74CE1B590EE12EF89B0B3?method=download&shareKey=988f3ff18913b0b53d744551ce1a7fec)

## term_query

> 词项查询，查询的基础方式
>
> ==通过词项查询可以看到一些现象，比如：能看出生成document时使用的什么分词器==

```sh
curl -XGET http://10.9.104.184:9200/index01/_search -d '{"query":{"term":{"title":"java"}}}'
```

> 👆👆👆👆可以搜索到：java编程思想，这本书

```sh
curl -XGET http://10.9.104.184:9200/index01/_search -d '{"query":{"term":{"title":"编程"}}}'
```

> 👆👆👆👆搜不到：Java编程思想，这本书
>
> 得到：这个document在生成时，分词器没有分出编程、思想



结论：

在Lucene中，封装document时，给每一个属性都赋值了很多参数，使用writer时也绑定了分词器。但是ES中这些内容都看不到

- ES中，document的属性对应类型，对应存储，对应分词计算
- 分词器默认是：`standardAnalyzer`
- 自定义分词器：在创建索引文件时传入mapping参数，指定分词器



## match_query

> 在搜索功能中常用的一个查询对象。

```sh
curl -XGET http://10.9.104.184:9200/index01/_search -d '{"query":{"match":{"title":"java编程思想"}}}'
```

> 可以对文本做分词计算，默认也是使用的`standardAnalyzer`，会将匹配到的所有document都返回
>
> 例如：java编程思想———>java，编，程，思，想
>
> - 查询所有的词项：
>   - title：java
>   - title：编
>   - title：程
>   - title：思
>   - title：想





# ES的mapping和IK分词器

## 安装IK分词器

> 在集群中，所有的ES节点配置都要保持一致，IK分词器要在每一台云主机的ES中安装

1. 将IK插件包解压到ES的`plugins`目录下

   ```sh
   [root@10-9-104-184 resources]# unzip /home/resources/elasticsearch-analysis-ik-5.5.2.zip -d /home/software/elasticsearch-5.5.2/plugins/
   ```

2. 测试IK分词器

   - 重启ES

   - 测试使用IK分词器计算文本

     ```sh
     curl -XPOST http://10.42.0.33:9200/_analyze -d '{"analyzer":"ik_max_word","text":"JAVA编程思想"}' 
     ```

     浏览器测试，通过浏览器url计算分词

     > 参数
     >
     > - analyzer：分词器名称
     > - text：测试的文本

     ```
     http://10.9.104.184:9200/index01/_analyze?analyzer=ik_max_word&text=中华人民共和国
     ```

     



## mapping映射

> mapping可以设置一个索引下的document的各种不同域属性的详细类型、不同属性使用的分词器



### 动态mapping

> 根据document中新增时携带的数据，决定对应的各种field域属性的类型，和分词器的使用

新建的索引文件，mapping默认是空的，直到新增一个document才会出现mapping结构，这就是动态mapping

```sh
[root@10-9-104-184 logs]# curl -XGET http://10.9.104.184:9200/index02/_mappings
########结果#############
{"index02":{"mappings":{}}} 
```



> 新增一条document数据，动态mapping会生成一些数据，实现document在该索引的映射

```json
{
    "index02": {
        "mappings": {
            "article": {//表示mapping中的类型
                "properties": {//类型的属性
                    "content": {//域名称
                        "type": "text",//域的类型相当于TextField
                        "analyzer": "standard",
                        "fields": {//对当前域扩展
                            "keyword": {
                                "type": "keyword",//keyword相当于StringField
                                "ignore_above": 256//虽然默认对字符串做整体存储，但是认为一旦长度超过256字节，就超过整体存储的默认要求
                            }
                        }
                    },
                    "id": {
                        //域属性"type": "long"//域类型
                    },
                    "title": {
                        //域属性"type": "text",
                        //相当于TextField"fields": {
                        "keyword": {
                            "type": "keyword",
                            "ignore_above": 256
                        }
                    }
                }
            }
        }
    }
}
}
```



### 静态mapping

> 手动设置静态mapping
>
> 在手动创建索引的时候，就将自定义生成的静态mapping结构一并的添加进去

```sh
curl -XPUT 
	-d '{"mappings":{"article":{"properties":{"id":{"type":"long"},"title":{"type":"text","analyzer":"ik_max_word"},"content":{"type":"text","analyzer":"ik_max_word"}}}}}' 
	http://10.9.104.184:9200/index03/
```

👇👇👇👇👇👇自定义的mapping结构👇👇👇👇👇

```json
{
  "mappings": {
    "article": {
      "properties": {
        "id": {
          "type": "long"
        },
        "title": {
          "type": "text",
          "analyzer": "ik_max_word"
        },
        "content": {
          "type": "text",
          "analyzer": "ik_max_word"
        }
      }
    }
  }
}
```



> 测试数据

1. 新增一条document数据

   ```sh
   curl -XPUT 
   	-d '{"id":1,"title":"es简介","content":"es好用好用真好用"}' 
   	http://10.9.104.184:9200/index03/article/1
   ```

   

2. 使用term_query查询

   ```sh
   curl -XGET 
   	-d '{"query":{"term":{"title":"编程"}}}'
   	http://10.9.104.184:9200/index03/_search 
   ```

   > 如果返回数据，说明该数据的分词词项有编程2个字，IK分词器生效







# ES使用的命令

```sh
############索引管理###############
#1新建索引
curl -XPUT http://10.42.0.33:9200/index01

#2 读写权限
curl -XPUT -d '{"blocks.read":false}' http://10.42.0.33:9200/index01/_settings

#3 查看索引
#单个
curl -XGET http://10.42.0.33:9200/index01/_settings
#多个
curl -XGET http://10.42.0.33:9200/index01,blog/_settings

#4 删除索引
curl -XDELETE http://10.42.0.33:9200/index02

#5打开关闭索引
#关闭
curl -XPOST http://10.42.0.33:9200/index01/_close
#打开
curl -XPOST http://10.42.0.33:9200/index01/_open

#多个
curl -XPOST http://10.42.0.33:9200/index01,blog,index02/_close
curl -XPOST http://10.42.0.33:9200/index01,blog,index02/_open

####################文档管理#################
#1新建文档
curl -XPUT -d '{"id":1,"title":"es简介","content":"es好用好用真好用"}' http://10.42.0.33:9200/book/article/1

#2 获取文档
curl -XGET http://10.42.0.33:9200/index01/article/1

#3 获取多个文档
curl -XGET  -d '{"docs":[{"_index":"index01","_type":"article","_id":"1"},{"_index":"index01","_type":"article","_id":"2"}]}' http://10.42.0.33:9200/_mget

#4删除文档
curl -XDELETE http://10.42.0.33:9200/index01/article/1

###################搜索###################
#1 查询所有文档
#准备一些文档数据

curl -XPUT -d '{"id":1,"title":"es简介","content":"es好用好用真好用"}' http://10.42.0.33:9200/index01/article/1
curl -XPUT -d '{"id":1,"title":"java编程思想","content":"这就是个工具书"}' http://10.42.0.33:9200/index01/article/2
curl -XPUT -d '{"id":1,"title":"大数据简介","content":"你知道什么是大数据吗，就是大数据"}' http://10.42.0.33:9200/index01/article/3

#2 match_all
curl -XGET http://10.42.0.33:9200/index01/_search -d '{"query": {"match_all": {}}}'

#3 term query
curl -XGET http://10.42.0.33:9200/index01/_search -d '{"query":{"term":{"title":"java"}}}'
curl -XGET http://10.42.0.33:9200/index01/_search -d '{"query":{"term":{"title":"java编程思想"}}}'
curl -XGET http://10.42.0.33:9200/jtdb_item/_search -d '{"query":{"term":{"title":"双卡双"}}}'

#4 match query
curl -XGET http://10.42.0.33:9200/index01/_search -d '{"query":{"match":{"title":"java编程思想"}}}'

#logstash启动
logstash -e 'input{stdin{}}output{stdout{codec=>rubydebug}}'


#IK分词器
curl -XPOST http://10.42.0.33:9200/_analyze -d '{"analyzer":"ik_max_word","text":"JAVA编程思想"}'
http://10.42.0.33:9200/index01/_analyze?analyzer=ik&text=%E4%B8%AD%E5%8D%8E%E4%BA%BA%E6%B0%91%E5%85%B1%E5%92%8C%E5%9B%BD
curl 

#IK分词器
curl -XPUT -d '{"id":1,"kw":"我们都爱中华人民共和国"}' http://10.42.0.33:9200/haha1/haha/1

#查看mapping
curl -XGET http://10.42.0.33:9200/jtdb_item/tb_item/_mapping

curl -XPUT -d '{"mappings":{"article":{"properties":{"id":{"type":"keyword"},"title":{"type":"text"},"content":{"type":"text"}}}}}' http://10.42.0.33:9200/book/

```

# ES集群的结构

## ES集群搭建逻辑

和redis、mycat数据库集群有区别。ES是一个可以启动的web应用，封装了扩展的功能，可以通过发现节点的==协调器==将一个集群所有的启动节点（在同一个网络中）自动添加到一起，实现集群的功能



## 节点角色

1. ==master==：整个集群的大脑，管理集群所有的元数据(meta-data)信息
   - ==元数据==：记录数据信息的数据
   - master管理的元数据：哪些索引、多少索引分片、多少分片副本、分片存放的位置、副本存放的位置等
   - master可以修改元数据，集群中唯一一个可以有权限修改元数据的角色，其他节点角色要想使用元数据，只能从master同步获取
2. ==data==：数据节点，用来读写数据。分配读任务，就读数据；分配写任务，就写数据。存储的就是索引的分片数据
3. ==ingest==：对外访问的连接节点
4. ==协调器==：众多的节点、角色，要想按照运行逻辑完美的整合添加到一个集群中，就需要协调器的加入



## 结构图

![](https://note.youdao.com/yws/api/personal/file/B5590B9829514EA095B10A2E0D26C07D?method=download&shareKey=c5b471848b66327c5d64e0880efbc17c)



# ES集群的搭建

## 准备ES节点

集群中有3个ES节点，分别在不同的服务器

- 节点名：es01,es02,es03
- 集群名称相同：elasticsearch
- 全部安装IK分词器



## 修改`elasticsearch.yml`

### 协调器

> 三个节点，每个节点的角色取决于配置文件中node值
>
> - node.master:true
> - node.data:true
> - node.ingest:true
>
> 默认情况下，每一个ES节点都默认有这三个配置，每个节点都可以是三个角色

协调器：任意一个ES节点都可以配置为当前集群的协调器。保证协调器不会出现宕机无法使用，一般这里都需要配置多个协调器。【从3台ES节点选择2个ES节点作为协调器即可】

> 69行：配置协调器ip地址

![](https://note.youdao.com/yws/api/personal/file/E6777BF04DE042D7ACC0922EA3F45782?method=download&shareKey=61432c4d93711ebf0fc23bf8302ab64d)



### 最小有效master

> 73行，配置最小有效master数，防止==脑裂==的出现
>
> 最小有效master数：==（master总数/2）+1==

![](https://note.youdao.com/yws/api/personal/file/82494910DD404E1D87EF95CF8A5D5DBF?method=download&shareKey=1646f36ab8823db3f98f13f1abfb5832)



> 脑裂：
>
> - 在一个ES集群中，现役master由于网络波动，导致集群判断它宕机，进而会进行一轮选举，出现新的现役master。
> - 当网络波动恢复后，整个集群中存在了2个master，每个master中有一个meta数据，导致集群数据混乱，使整个集群停止工作

![](https://note.youdao.com/yws/api/personal/file/204BF6DA076748418E79ED001C65DBD9?method=download&shareKey=045976f549b5116996d9ed1740d5c4df)

> ES采用配置==过半集群有效master数量==，防止脑裂的出现
>
> ==出现网络波动后，无论如何，集群中保证有一个有效的master==



## 启动

把每个服务器的ES节点启动

- 切换到普通用户：`su es`
- 到`elasticsearch/bin`目录执行`elasticsearch`命令启动

通过插件可以看到一个分布式的高可用的ES集群

![](https://note.youdao.com/yws/api/personal/file/151518D6165B4A3189C87F668EA9BB7E?method=download&shareKey=c1230c6369538a0e050a1aa3a772cf73)

> 启动之后，数据会经过ES计算，实现迁移，将分片分部到多个节点的分布式集群中（可以配置分片和副本数量，适当调整集群的分布式和高可用）
>
> - 分片数量调高：一个集群就扩展更过的节点
> - 副本分片数量调高：支持更高的高可用





# 选举逻辑

> 集群运行过程中，选举逻辑一直存在，只要协调器不全部宕机，集群就可以正常执行选举逻辑

1. 每个节点（具备node.master:true）启动时都会连接ping通协调器（任何一个都可以），获取集群当前的状态。其中有个对象：`activeMaster`保存了现役master的信息。【进入第2步】
2. 判断`activeMaster`是否为空，
   - 空：表明没有选举完成，进入【第3步】
   - 不为空：说明已经选举完成，并选举出一个master，**选举逻辑结束**
3. 当前节点将集群中所有的master节点都保存到一个备选人list中：`candidata{es01,es02,...}`。【进入第4步】
4. 判断备选人list里的master节点数量是否满足最小master数量
   - 满足：从中比较节点Id，较大的为现役master，并将该master信息存放在`activeMaster`中。
   - 不满足：返回第一步，重新执行逻辑



> 节点宕机重新选举：
>
> - ==现役master宕机==（此时`activeMaster`为空）：剩余master重新执行选举逻辑
> - ==备选master宕机==：只要剩余master满足最小master数量，就不影响集群使用



# ES节点内存不够

1. ES进程的jvm默认判断需要2G的内存
2. 修改配置文件`elasticsearch/config/jvm.options`

![](https://note.youdao.com/yws/api/personal/file/B3C41805BCA448EFA493BAA5566A64E4?method=download&shareKey=63b967c7f8550858fda7c8f5548498ff)







# Java客户端代码

> 理论上来讲，通过http协议的访问操作ES，也就可以使用RestTemplate操作

TransportClient的Java客户端：包装了TCP/IP协议，可以通过访问ES节点的9300端口，实现通过API操作ES



## 依赖资源

> 不能和Lucene6.0.0的依赖在同一个系统中
>
> ES的依赖中包含了Lucene 5.5.2，版本不一样会冲突

```xml
<!--elasticsearch 核心包-->
<dependency>
    <groupId>org.elasticsearch</groupId>
    <artifactId>elasticsearch</artifactId>
    <version>5.5.2</version>
</dependency>

<!--transportclient-->
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>transport</artifactId>
    <version>5.5.2</version>
</dependency>
```



## 测试代码

### 索引管理

> 增加索引、删除索引、判断索引是否存在等

```java
@Test
public void indexManage(){

    //通过client拿到索引管理对象
    AdminClient admin = client.admin();

    // ClusterAdminClient cluster = admin.cluster();
    IndicesAdminClient indices = admin.indices();

    //TransportClient中有2套方法，一套直接发送调用
    //一套是预先获取request对象。
    CreateIndexRequestBuilder request1 = indices.prepareCreate("index01");//不存在的索引名称

    IndicesExistsRequestBuilder request2 = indices.prepareExists("index01");
    boolean exists = request2.get().isExists();
    if(exists){
        System.out.println("index01已存在");
        return;
        //indices.prepareDelete("index01").get();
    }

    //发送请求request1到es
    CreateIndexResponse response1 = request1.get();

    //从reponse中解析一些有需要的数据
    System.out.println(response1.isShardsAcked());//json一部分 shards_acknowleged:true
    System.out.println(response1.remoteAddress());
    response1.isAcknowledged();//json acknowledged:true
}
```



### 文档管理

> 向ES集群中新增、查询、删除文档

> 新增

```java
@Test

public void documentManage() throws JsonProcessingException {

    //新增文档，准备文档数据

    //拼接字符串 对象与json的转化工具ObjectMapper
    Product p=new Product();
    p.setProductNum(500);
    p.setProductName("三星手机");
    p.setProductCategory("手机");
    p.setProductDescription("能攻能守，还能爆炸");
    ObjectMapper om=new ObjectMapper();
    String pJson = om.writeValueAsString(p);

    //client 获取request发送获取response
    //curl -XPUT -d {json} http://10.9.104.184:9200/index01/product/1
    IndexRequestBuilder request = client.prepareIndex("index01", "product", "1");

    //source填写成pJson
    request.setSource(pJson);
    request.get();
}
```



> 查询

```java
@Test

public void getDocument(){

    //获取一个document只需要index type id坐标
    GetRequestBuilder request = client.prepareGet("index01", "product", "1");
    GetResponse response = request.get();

    //从response中解析数据
    System.out.println(response.getIndex());
    System.out.println(response.getId());
    System.out.println(response.getVersion());
    System.out.println(response.getSourceAsString());
    client.prepareDelete("index01","product","1");
}
```



### 搜索功能

> 搜索索引文件中的文档信息

```java
@Test
public void search(){
    //其他查询条件也可以创建
    MatchQueryBuilder query = QueryBuilders.matchQuery("productName","三星手机");
    //封装查询条件，不同逻辑查询条件对象不同
    //TermQueryBuilder query = QueryBuilders.termQuery
    // ("productName", "星");
    //底层使用lucene查询，基于浅查询，可以支持分页
    SearchRequestBuilder request = client.prepareSearch("index01");

    //在request中封装查询条件，分页条件等
    request.setQuery(query);
    request.setFrom(0);//起始位置 类似limit start
    request.setSize(5);
    SearchResponse response = request.get();

    //从response对象中解析搜索的结果
    //解析第一层hits相当于topDocs
    SearchHits searchHits = response.getHits();
    System.out.println("总共查到："+searchHits.totalHits);
    System.out.println("最大评分："+searchHits.getMaxScore());

    //循环遍历的结果
    SearchHit[] hits = searchHits.getHits();//第二层hits

    //hits包含想要的查询结果
    for (SearchHit hit:hits){
        System.out.println(hit.getSourceAsString());
    }
}
```

> 双层hits如下图👇👇👇👇👇👇👇👇

![](https://note.youdao.com/yws/api/personal/file/18546201B8B74CE1B590EE12EF89B0B3?method=download&shareKey=988f3ff18913b0b53d744551ce1a7fec)



# ELK家族

## 搜索系统结构

1. 数据整理封装，放到ES的索引文件中保存

2. 客户端系统访问ES封装的请求中，携带了query对象的查询参数

   ![](https://note.youdao.com/yws/api/personal/file/97137132852745FAA4F0122DF0844B79?method=download&shareKey=e8ce2da8d69f4c3975aee2120c1ca999)

> 通过业务逻辑考虑数据一致性，数据库和索引文件数据要一致。
>
> 数据库的数据源可能发生各种数据变动，要考虑到ES中索引文件的一致性



## ELK家族

> 基于ElasticSearch的全文检索功能，形成一整套技术体系。
>
> - 包含了数据导入、数据更新
> - 包含了全文检索数据管理
> - 包含了数据视图化展示和分析

E：ElasticSearch----有大量的数据（index）

L：logstash----日志数据收集器，将各种数据源的数据导入ES中

K：kibana----试图展示，数据分析工具

> 这样一个结构，使得存储在ES中的数据可以实时的更新，并且除了提供搜索以外，还可以实现更高的数据分析的价值

![](https://note.youdao.com/yws/api/personal/file/8FE5B851531649EB802AC9422B367A65?method=download&shareKey=1f4361dd4a303f9fdf387c5f484e7915)





# 整合Springboot

> TransportClient整合到springboot，实现搜索功能

1. 准备配置类。注解：@Configuration

2. bean方法@Bean初始化一个TransportClient对象

3. @ConfigurationProperties("es") 读取配置文件属性

   ```properties
   es.nodes=10.42.0.33,10.42.24.13,10.42.104.102
   ```

   

  //Highlight
    @Test
    public void testHighlight() throws IOException, ParseException {
        //搜索请求对象
        SearchRequest searchRequest = new SearchRequest("xc_course");
        //指定类型
        searchRequest.types("doc");
        //搜索源构建对象
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();

        //boolQuery搜索方式
        //先定义一个MultiMatchQuery
        MultiMatchQueryBuilder multiMatchQueryBuilder = QueryBuilders.multiMatchQuery("开发框架", "name", "description")
                .minimumShouldMatch("50%")
                .field("name", 10);
    
        //定义一个boolQuery
        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        boolQueryBuilder.must(multiMatchQueryBuilder);
        //定义过虑器
        boolQueryBuilder.filter(QueryBuilders.rangeQuery("price").gte(0).lte(100));
    
        searchSourceBuilder.query(boolQueryBuilder);
        //设置源字段过虑,第一个参数结果集包括哪些字段，第二个参数表示结果集不包括哪些字段
        searchSourceBuilder.fetchSource(new String[]{"name","studymodel","price","timestamp"},new String[]{});
    
        //设置高亮
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        highlightBuilder.preTags("<tag>");
        highlightBuilder.postTags("</tag>");
        highlightBuilder.fields().add(new HighlightBuilder.Field("name"));
        highlightBuilder.fields().add(new HighlightBuilder.Field("description"));
        searchSourceBuilder.highlighter(highlightBuilder);
    
        //向搜索请求对象中设置搜索源
        searchRequest.source(searchSourceBuilder);
        //执行搜索,向ES发起http请求
        SearchResponse searchResponse = client.search(searchRequest);
        //搜索结果
        SearchHits hits = searchResponse.getHits();
        //匹配到的总记录数
        long totalHits = hits.getTotalHits();
        //得到匹配度高的文档
        SearchHit[] searchHits = hits.getHits();
        //日期格式化对象
//        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS Z");
        for(SearchHit hit:searchHits){
            //文档的主键
            String id = hit.getId();
            //源文档内容
            Map<String, Object> sourceAsMap = hit.getSourceAsMap();
            //源文档的name字段内容
            String name = (String) sourceAsMap.get("name");
            //取出高亮字段
            Map<String, HighlightField> highlightFields = hit.getHighlightFields();
            if(highlightFields!=null){
                //取出name高亮字段
                HighlightField nameHighlightField = highlightFields.get("name");
                if(nameHighlightField!=null){
                    Text[] fragments = nameHighlightField.getFragments();
                    StringBuffer stringBuffer = new StringBuffer();
                    for(Text text:fragments){
                        stringBuffer.append(text);
                    }
                    name = stringBuffer.toString();
                }
            }

            //由于前边设置了源文档字段过虑，这时description是取不到的
            String description = (String) sourceAsMap.get("description");
            //学习模式
            String studymodel = (String) sourceAsMap.get("studymodel");
            //价格
            Double price = (Double) sourceAsMap.get("price");
            //日期
            Date timestamp = dateFormat.parse((String) sourceAsMap.get("timestamp"));
            System.out.println(name);
            System.out.println(studymodel);
            System.out.println(description);
        }
    
    }

# ElasticSearch介绍

## 介绍

1、elasticsearch是一个基于Lucene的高扩展的分布式搜索服务器，支持开箱即用。    

2、elasticsearch隐藏了Lucene的复杂性，对外提供Restful 接口来操作索引、搜索。

 

**突出优点：** 

1. 扩展性好，可部署上百台服务器集群，处理PB级数据。

2. 近实时的去索引数据、搜索数据。

**es和solr选择哪个？**

1. 如果你公司现在用的solr可以满足需求就不要换了。

2. 如果你公司准备进行全文检索项目的开发，建议优先考虑elasticsearch，因为像Github这样大规模的搜索都在用它。

 

## 倒排索引

下图是ElasticSearch的索引结构，下边黑色部分是物理结构，上边黄色部分是逻辑结构，逻辑结构也是为了更好的  去描述ElasticSearch的工作原理及去使用物理结构中的索引文件。

<img src="https://unpkg.zhimg.com/youthlql@1.0.8/ElasticSearch/Introduction/0001.png" width=80%>

逻辑结构部分是一个倒排索引表：

1、将要搜索的文档内容分词，所有不重复的词组成分词列表。

2、将搜索的文档最终以Document方式存储起来。

3、每个词和docment都有关联。

如下：

<img src="https://unpkg.zhimg.com/youthlql@1.0.8/ElasticSearch/Introduction/0002.png" width=40%>

现在，如果我们想搜到`quick brown`我们只需要查找包含每个词条的文档：

<img src="https://unpkg.zhimg.com/youthlql@1.0.8/ElasticSearch/Introduction/0003.png" width=80%>

两个文档都匹配，但是第一个文档比第二个匹配度更高。如果我们使用仅计算匹配词条数量的简单 相似性算法 ， 那么，我们可以说，对于我们查询的相关性来讲，第一个文档比第二个文档更佳

# 基本概念



1.创建索引库  --------------------->类似于:数据库的建表

2.创建映射  --------------------->类似于:数据库的添加表中字段

3.创建(添加)文档  --------------------->类似于:数据库的往表中添加数据。术语称这个过程为:创建索引

5.搜索文档  --------------------->类似于:从数据库里查数据

6.文档      --------------------->类似于:数据库中的一行记录(数据)

7.Field(域)   --------------------->类似于:数据库中的字段

 





## 创建索引库

### 概念：

ES的索引库是一个逻辑概念，它包括了分词列表及文档列表，同一个索引库中存储了相同类型的文档。它就相当于MySQL中的表，或相当于Mongodb中的集合。

索引(index)

```shell
# 索引是 ES 对逻辑数据的逻辑存储，所以可以被分为更小的部分

# 可以将索引看成 MySQL 的 Table，索引的结构是为快速有效的全文索引准备的，特别是它不存储原始值

# 可以将索引存放在一台机器，或分散在多台机器上

# 每个索引有一或多个分片(shard)，每个分片可以有多个副本(replica)
```

### 操作：

使用postman这样的工具创建： put http://localhost:9200/索引库名称


```shell
# ES 中提供非结构化索引，实际上在底层 ES 会进行结构化操作，对用户透明

PUT http://localhost:9200/索引库名称
{
    "settings":{
        "index":{
            "number_of_shards":"1", # 分片数
            "number_of_replicas":"0" # 副本数
        }
    }
}
```

- number_of_shards：设置分片的数量，在集群中通常设置多个分片，表示一个索引库将拆分成多片分别存储不同  的结点，提高了ES的处理能力和高可用性，入门程序使用单机环境，这里设置为1。

- number_of_replicas：设置副本的数量，设置副本是为了提高ES的高可靠性，单机环境设置为0.



## 创建映射

### 概念

在索引中每个文档都包括了一个或多个ﬁeld，创建映射就是向索引库中创建ﬁeld的过程，下边是document和ﬁeld  与关系数据库的概念的类比：

文档（Document）----- Row记录

字段（Field）----- Columns 列

注意：6.0之前的版本有type（类型）概念，type相当于关系数据库的表，ES官方将在ES9.0版本中彻底删除type。  上边讲的创建索引库相当于关系数据库中的数据库还是表？

1、如果相当于数据库就表示一个索引库可以创建很多不同类型的文档，这在ES中也是允许的。

2、如果相当于表就表示一个索引库只能存储相同类型的文档，ES官方建议在一个索引库中只存储相同类型的文档。

3、所以索引库相当于数句酷的一个表



### 操作

1、我们要把课程信息存储到ES中，这里我们创建课程信息的映射，先来一个简单的映射，如下： 

发送：post http://localhost:9200/索引库名称/类型名称/_mapping

2、创建类型为xc_course的映射，共包括三个字段：name、description、studymondel 由于ES6.0版本还没有将type彻底删除，所以暂时把type起一个没有特殊意义的名字doc。post 请求：http://localhost:9200/xc_course/doc/_mapping

表示：在xc_course索引库下的doc类型下创建映射。doc是类型名，可以自定义，在ES6.0中要弱化类型的概念，  给它起一个没有具体业务意义的名称。

```json
 {
	"properties": {
        "name": {
        "type": "text"
        },

        "description":{ 
        "type": "text"
        },

        "studymodel":{ 
        "type":"keyword"
        }
 	}
}
```


## 创建文档

### 概念

ES中的文档相当于MySQL数据库表中的记录。

```shell
# 存储在 ES 中的主要实体叫文档，可以看成 MySQL 的一条记录

# ES 与 Mongo 的 document 类似，都可以有不同的结构，但 ES 相同字段必须有相同类型

# document 由多个字段组成，每个字段可能多次出现在一个文档里，这样的字段叫多值字段(multivalued)

# 每个字段的类型，可以使文本、数值、日期等。

# 字段类型也可以是复杂类型，一个字段包含其他子文档或者数组

# 在 ES 中，一个索引对象可以存储很多不同用途的 document，例如一个博客App中，可以保存文章和评论

# 每个 document 可以有不同的结构

# 不同的 document 不能为相同的属性设置不同的类型，例 : title 在同一索引中所有 Document 都应该相同数据类型
```



### 操作

发送：put 或Post http://localhost:9200/xc_course/doc/id值

（如果不指定id值ES会自动生成ID）

http://localhost:9200/xc_course/doc/4028e58161bcf7f40161bcf8b77c0000

```json
{
	"name":”Bootstrap开发框架",

    "description" : "Bootstrap是由Twitter推出的一个前台页面开发框架,在行业之中使用较为广泛。此开发框架包含	了大量的CSS、JS程序代码，可以帮助开发者(尤其是不擅长页面开发的程序人员)轻松的实现个不受浏览器限制的精美界面	 效果。”,

	"studymodel": "201001"

}
```



## 搜索文档

1、根据课程id查询文档

发送：get http://localhost:9200/xc_course/doc/4028e58161bcf7f40161bcf8b77c0000

使用postman测试：

<img src="https://unpkg.zhimg.com/youthlql@1.0.8/ElasticSearch/Introduction/0004.png">



2、查询所有记录

发送 get http://localhost:9200/xc_course/doc/_search

 

 

3、查询名称中包括spring 关键字的的记录

发送：get http://localhost:9200/xc_course/doc/_search?q=name:bootstrap

 

 

4、查询学习模式为201001的记录

发送 get http://localhost:9200/xc_course/doc/_search?q=studymodel:201001



**查询结果分析：**

```json
{
	"took": 1,
	"timed_out": false,
	"_shards": {
		"total": 1,
		"successful": 1,
		"skipped": 0,
		"failed": 0
	},
	"hits": {
		"total": 1,
		"max_score": 0.2876821,
		"hits": [
			{
				"_index": "xc_course",
				"_type": "doc",
				"_id": "4028e58161bcf7f40161bcf8b77c0000",
				"_score": 0.2876821,
				"_source": {
					"name": "Bootstrap开发框架",
					"description": "Bootstrap是由Twitter推出的一个前台页面开发框架，在行业之中使用较 为广泛。此开发框架包含了大量的CSS、JS程序代码，可以帮助开发者（尤其是不擅长页面开发的程序人员）轻松的实现 一个不受浏览器限制的精美界面效果。",
					"studymodel": "201001"
				}
			}
		]
	}
}
```

**结果说明：**

took：本次操作花费的时间，单位为毫秒。timed_out：请求是否超时

_shards：说明本次操作共搜索了哪些分片hits：搜索命中的记录

hits.total ： 符合条件的文档总数 hits.hits ：匹配度较高的前N个文档

hits.max_score：文档匹配得分，这里为最高分

_score：每个文档都有一个匹配度得分，按照降序排列。

_source：显示了文档的原始内容。





# 分词

## 内置分词

### 分词API

分词是将一个文本转换成一系列单词的过程，也叫文本分析，在 ES 中称之为 Analysis

例如 : 我是中国人 -> 我 | 是 | 中国人

```json
# 指定分词器进行分词
POST http://['自己的ip 加 port']/_analyze
{
    "analyzer":"standard",
    "text":"hello world"
}

# 结果中不仅可以看出分词的结果，还返回了该词在文本中的位置

# 指定索引分词
POST http://['自己的ip 加 port']/beluga/_analyze
{
    "analyzer":"standard",
    "field":"hobby",
    "text":"听音乐"
}
```



### Standard

```shell
# Standard 标准分词，按单词切分，并且会转换成小写
POST http://['自己的ip 加 port']/_analyze
{
    "analyzer":"standard",
    "text": "A man becomes learned by asking questions."
}
```

### Simple

```shell
# Simple 分词器，按照非单词切分，并且做小写处理
POST http://['自己的ip 加 port']/_analyze
{
    "analyzer":"simple",
    "text":"If the document does't already exist"
}
```

### Whitespace

```shell
# Whitespace 是按照空格切分
POST http://['自己的ip 加 port']/_analyze
{
    "analyzer":"whitespace",
    "text":"If the document does't already exist"
}
```

### Stop

```shell
# Stop 去除 Stop Word 语气助词，如 the、an 等
POST http://['自己的ip 加 port']/_analyze
{
    "analyzer":"stop",
    "text":"If the document does't already exist"
}
```

### Keyword

```shell
# keyword 分词器，意思是传入就是关键词，不做分词处理
POST http://['自己的ip 加 port']/_analyze
{
    "analyzer":"keyword",
    "text":"If the document does't already exist"
}
```

### 中文分词

```shell
# 中文分词的难点在于，汉语中没有明显的词汇分界点

# 常用中文分词器，IK jieba THULAC 等，推荐 IK

# IK Github 站点<自定义词典扩展，禁用词典扩展等>
https://github.com/medcl/elasticsearch-analysis-ik
```



## IK分词器

安装过程这里不介绍，主要是解决常见中文分词的问题

Github地址：https://github.com/medcl/elasticsearch-analysis-ik

### 两种分词模式

ik分词器有两种分词模式：ik_max_word和ik_smart模式。

 1、ik_max_word

会将文本做最细粒度的拆分，比如会将“中华人民共和国人民大会堂”拆分为“中华人民共和国、中华人民、中华、  华人、人民共和国、人民、共和国、大会堂、大会、会堂等词语。

2、ik_smart

会做最粗粒度的拆分，比如会将“中华人民共和国人民大会堂”拆分为中华人民共和国、人民大会堂。  测试两种分词模式：





# 映射

上边章节安装了ik分词器，如果在索引和搜索时去使用ik分词器呢？如何指定其它类型的ﬁeld，比如日期类型、数  值类型等。本章节学习各种映射类型及映射维护方法。

## 映射维护方法

1、查询所有索引的映射：

GET： http://localhost:9200/_mapping

2、创建映射

post 请求：http://localhost:9200/xc_course/doc/_mapping

在上面提到过

```
 {
	"properties": {
        "name": {
        "type": "text"
        },

        "description":{ 
        "type": "text"
        },

        "studymodel":{ 
        "type":"keyword"
        }
 	}
}
```



3、更新映射

映射创建成功可以添加新字段，已有字段不允许更新。

4、删除映射

通过删除索引来删除映射。



## 常用映射类型

### text文本字段

**1）text**

字符串包括text和keyword两种类型： 通过analyzer属性指定分词器。 

下边指定name的字段类型为text，使用ik分词器的ik_max_word分词模式。 

```json
{
	"name": {
		"type": "text",
		"analyzer": "ik_max_word"
	}
}
```

上边指定了analyzer是指在索引和搜索都使用ik_max_word，如果单独想定义搜索时使用的分词器则可以通过search_analyzer属性。

对于ik分词器建议是索引时使用ik_max_word将搜索内容进行细粒度分词，搜索时使用ik_smart提高搜索精确性。

```json
{
	"name": {
		"type": "text",
		"analyzer": "ik_max_word",
		"search_analyzer": "ik_smart"
	}
}
```

**2） index**

通过index属性指定是否索引。

默认为index=true，即要进行索引，只有进行索引才可以从索引库搜索到。

但是也有一些内容不需要索引，比如：商品图片地址只被用来展示图片，不进行搜索图片，此时可以将index设置  为false。

删除索引，重新创建映射，将pic的index设置为false，尝试根据pic去搜索，结果搜索不到数据

```json
{
	"pic": {
		"type": "text",
		"index": false
	}
}
```



**3）store**

是否在source之外存储，每个文档索引后会在 ES中保存一份原始文档，存放在"_source"中，一般情况下不需要设置 store为true，因为在_source中已经有一份原始文档了。



### keyword关键字字段

上边介绍的text文本字段在映射时要设置分词器，keyword字段为关键字字段，通常搜索keyword是按照整体搜     索，所以创建keyword字段的索引时是不进行分词的，比如：邮政编码、手机号码、身份证等。keyword字段通常  用于过虑、排序、聚合等。

**测试：**

更改映射：

```json
{
	"properties": {
		"studymodel": {
			"type": "keyword"
		},
		"name": {
			"type": "keyword"
		}
	}
}
```

添加文档：

```json
{
	"name": "java编程基础",
	"description": "java语言是世界第一编程语言，在软件开发领域使用人数最多。",
	"pic": "group1/M00/00/01/wKhlQFqO4MmAOP53AAAcwDwm6SU490.jpg",
	"studymodel": "201001"
}
```

根据name查询文档。搜索：http://localhost:9200/xc_course/_search?q=name:java name是keyword类型，所以查询方式是精确查询。



### 日期类型

日期类型不用设置分词器。

通常日期类型的字段用于排序。

1)format

通过format设置日期格式例子：

下边的设置允许date字段存储年月日时分秒、年月日及毫秒三种格式

```json
{
	"properties": {
		"timestamp": {
			"type": "date",
			"format": "yyyy‐MM‐dd HH:mm:ss||yyyy‐MM‐dd"
		}
	}
}
```

插入文档： 

Post :http://localhost:9200/xc_course/doc/3 

```json
{
	"name": "spring开发基础",
	"description": "spring 在java领域非常流行，java程序员都在用。",
	"studymodel": "201001",
	"pic": "group1/M00/00/01/wKhlQFqO4MmAOP53AAAcwDwm6SU490.jpg",
	"timestamp": "2018‐07‐04 18:28:58"
}
```



### 综合例子

post：http://localhost:9200/xc_course/doc/_mapping

```json
{
	"properties": {
		"description": {
			"type": "text",
			"analyzer": "ik_max_word",
			"search_analyzer": "ik_smart"
		},
		"name": {
			"type": "text",
			"analyzer": "ik_max_word",
			"search_analyzer": "ik_smart"
		},
		"pic": {
			"type": "text",
			"index": false
		},
		"price": {
			"type": "float"
		},
		"studymodel": {
			"type": "keyword"
		},
		"timestamp": {
			"type": "date",
			"format": "yyyy‐MM‐dd HH:mm:ss||yyyy‐MM‐dd||epoch_millis"
		}
	}
}
```

