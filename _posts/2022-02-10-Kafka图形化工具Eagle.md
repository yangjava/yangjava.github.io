---
layout: post
categories: [Kafka]
description: none
keywords: Kafka
---
# Kafka图形化工具Eagle


## 安装kafka-eagle
下载kafka-eagle的安装包，下载地址：https://github.com/smartloli/kafka-eagle-bin/releases

下载完成后将kafka-eagle解压到指定目录；
```
cd /mydata/kafka/
tar -zxvf kafka-eagle-web-2.0.5-bin.tar.gz
```

在/etc/profile文件中添加环境变量KE_HOME；
```
vi /etc/profile
# 在profile文件中添加
export KE_HOME=/mydata/kafka/kafka-eagle-web-2.0.5
export PATH=$PATH:$KE_HOME/bin
# 使修改后的profile文件生效
. /etc/profile

```
安装MySQL并添加数据库ke，kafka-eagle之后会用到它；

修改配置文件$KE_HOME/conf/system-config.properties，主要是修改Zookeeper的配置和数据库配置，注释掉sqlite配置，改为使用MySQL；
```
######################################
# multi zookeeper & kafka cluster list
######################################
kafka.eagle.zk.cluster.alias=cluster1
cluster1.zk.list=localhost:2181
 
######################################
# kafka eagle webui port
######################################
kafka.eagle.webui.port=8048
 
######################################
# kafka sqlite jdbc driver address
######################################
# kafka.eagle.driver=org.sqlite.JDBC
# kafka.eagle.url=jdbc:sqlite:/hadoop/kafka-eagle/db/ke.db
# kafka.eagle.username=root
# kafka.eagle.password=www.kafka-eagle.org
 
######################################
# kafka mysql jdbc driver address
######################################
kafka.eagle.driver=com.mysql.cj.jdbc.Driver
kafka.eagle.url=jdbc:mysql://localhost:3306/ke?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
kafka.eagle.username=root
kafka.eagle.password=root
```
使用如下命令启动kafka-eagle；
```
$KE_HOME/bin/ke.sh start
```
命令执行完成后会显示如下信息，但并不代表服务已经启动成功，还需要等待一会；

再介绍几个有用的kafka-eagle命令：
```
# 停止服务
$KE_HOME/bin/ke.sh stop
# 重启服务
$KE_HOME/bin/ke.sh restart
# 查看服务运行状态
$KE_HOME/bin/ke.sh status
# 查看服务状态
$KE_HOME/bin/ke.sh stats
# 动态查看服务输出日志
tail -f $KE_HOME/logs/ke_console.out
```
启动成功可以直接访问，输入账号密码admin:123456，访问地址：http://localhost:8048/ 登录成功后可以访问到Dashboard，界面还是很棒的！

## 可视化工具使用
我们使用命令行创建了Topic，这里可以直接通过界面来创建；

我们还可以直接通过kafka-eagle来发送消息；

我们可以通过命令行来消费Topic中的消息；
```
bin/kafka-console-consumer.sh --topic testTopic --from-beginning --bootstrap-server 192.168.5.78:9092
```
还有一个很有意思的功能叫KSQL，可以通过SQL语句来查询Topic中的消息；

可视化工具自然少不了监控，如果你想开启kafka-eagle对Kafka的监控功能的话，需要修改Kafka的启动脚本，暴露JMX的端口；
```
vi kafka-server-start.sh
# 暴露JMX端口
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-server -Xms2G -Xmx2G -XX:PermSize=128m -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:ParallelGCThreads=8 -XX:ConcGCThreads=5 -XX:InitiatingHeapOccupancyPercent=70"
    export JMX_PORT="9999"
f
```