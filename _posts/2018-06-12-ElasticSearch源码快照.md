---
layout: post
categories: ElasticSearch
description: none
keywords: ElasticSearch
---
# ElasticSearch源码快照
快照模块是ES备份、迁移数据的重要手段。它支持增量备份，支持多种类型的仓库存储。

## 快照模块分析
本章我们先来看看如何使用快照，以及它的一些细节特性，然后分析创建、删除及取消快照的实现原理。

仓库用于存储快照，支持共享文件系统（例如，NFS），以及通过插件支持的HDFS、AmazonS3、Microsoft Azure、Google GCS。

在跨版本支持方面，可以支持不跨大版本的快照和恢复。
- 在6.x版本中创建的快照可以恢复到6.x版本；
- 在2.x版本中创建的快照可以恢复到5.x版本；
- 在1.x版本中创建的快照可以恢复到2.x版本。
相反，1.x版本创建的快照不可以恢复到5.x版本和6.0版本，2.x版本创建的快照不可以恢复到6.x版本。

升级集群前建议先通过快照备份数据。跨越大版本的数据迁移可以考虑使用reindex API。 当需要迁移数据时，可以将快照恢复到另一个集群。快照不仅可以对索引备份，还可以将模板一起保存。恢复到的目标集群不需要相同的节点规模，只要它的存储空间足够容纳这些数据即可。

要使用快照，首先应该注册仓库。快照存储于仓库中。

## 仓库
仓库用于存储创建的快照。建议为每个大版本创建单独的快照存储库。如果使用多个集群注册相同的快照存储库，那么最好只有一个集群对存储库进行写操作。连接到该存储库的其他集群都应该将存储库设置为readonly模式。

使用下面的命令注册一个仓库：
```
curl -X PUT " localhost: 9200/_ snapshot/my_ backup" -H ' Content-Type: application/json' -d'
{
    "type": "fs",
    "settings": {
        "location": "/mnt/my_backup"
    }
}
```
本例中，注册的仓库名称为my_backup, type 为 fs，指定仓库类型为共享文件系统。共享文件系统支持的配置如下表所示。
- location	指定了一个已挂载的目的地址
- compress	是否开启压缩。压缩仅对元数据进行(mapping 及settings)， 不对数据文件进行压缩，默认为true
- chunk_ size	传输文件时数据被分解为块，此处配置块大小，单位为字节，默认为null (无限块大小)
- max_snapshot_bytes_per_sec	快照操作时节点间限速值，默认为40MB
- max_restore_bytes_per_ sec	从快照恢复时节点间限速值，默认为40MB
- readonly	设置仓库属性为只读，默认为false

要获取某个仓库配置信息，可以使用下面的API：
```
curl -X GET “localhost:9200/_snapshot/my_backup”
```
返回信息如下：
```
{
“my_ backup”: {
“type”: “fs”,
“settings”: {
“location”: “/mnt/my_backup”
}
}
}
```
要获取多个存储库的信息，可以指定一个以逗号分隔的存储库列表，还可以在指定存储库名称时使用“*”通配符。例如:
```
curl -X GET "localhost:9200/_snapshot/repo*,*backup*"
```
要获取当前全部仓库的信息，可以省略仓库名称，使用_ all:
```
curl -X GET "localhost:9200/_snapshot"
或
curl -X GET "localhost:9200/_snapshot/_all"
```
可以使用下面的命令从仓库中删除快照:
```
curl -X DELETE "localhost:9200/_snapshot/my_backup/snapshot_1"
```
可以使用下面的命令删除整个仓库:
```
curl -X DELETE "localhost:9200/_snapshot/my_backup"
```
当仓库被删除时，ES只是删除快照的仓库位置引用信息，快照本身没有删除。

## 共享文件系统
当使用共享文件系统时，需要将同一个共享存储挂载到集群每个节点的同一个挂载点(路径)，包括所有数据节点和主节点。然后将这个挂载点配置到elasticsearch.yml的path.repo字段。 例如，挂载点为 /mnt/my_backup， 那么在elasticsearch.yml中应该添加如下配置：
```
path.repo: ["/mnt/my_backups"]
```
path.repo配置以数组的形式支持多个值。如果配置多个值，则不像path.data一样同时使用这些路径，相反，应该为每个挂载点注册不同的仓库。例如，一个挂载点存储空间不足以容纳集群所有数据，可使用多个挂载点，同时注册多个仓库，将数据分开快照到不同的仓库。

path.repo支持微软的UNC路径，配置格式如下:
```
path.repo: ["MY_SERVERSnapshots"]
```
当配置完毕，需要重启所有节点使之生效。然后就可以通过仓库API注册仓库，执行快照了。

使用共享存储的优点是跨版本兼容性好，适合迁移数据。缺点是存储空间较小。如果使用HDFS，则受限于插件使用的HDFS版本。插件版本要匹配ES，而这个匹配的插件使用固定版本的HDFS客户端。一个HDFS客户端只支持写入某些兼容版本的HDFS集群。















