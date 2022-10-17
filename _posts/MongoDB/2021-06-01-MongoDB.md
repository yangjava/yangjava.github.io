# MongoDB

　**MongoDB概念**

　　MongoDB是一个NoSQL(Not only SQL)数据库，其在概念上与传统的关系型数据库有很大差别。以下是其与传统的关系型数据库概念的比较：

| 关系型数据库 | MongoDB    | 解释                                  |
| ------------ | ---------- | ------------------------------------- |
| database     | database   | 数据库                                |
| table        | collection | 表/集合                               |
| row          | document   | 一条记录/一个文档                     |
| column       | field      | 字段/域                               |
| index        | index      | 索引                                  |
| tablejoins   | 嵌入文档   | 表连接/mongoDB不支持                  |
| primarykey   | primarykey | 主键,mongoDB将自动生成_id的字段为主键 |

　　数据库服务端由mongod开启，客户端由mongo开启。

　　

　　数据库：mongoDB服务端可以创建多个数据库，在shell下可通过**show dbs**来查看所有数据库的列表，**db**命令表示当前的数据库。

　　文档：文档表示一条记录，有键值对构成(BSON)，形式如下：{"title":"myfirstRecorder","by":"insaneXs"}

　　集合：集合是文档组，相当于关系型数据库中的表的概念。但是集合没有固定的结构，意味着你可以对集合插入不同的数据结构和格式。

　　capped collections:是固定大小的集合。有着很高的性能和队列过期的性质。集合可以被默认创建，但是capped collection需要通过以下语法被显示创建：

　　db.createCollection("myCappedCollection",{capped:true, size:10000}); 其中size指定了集合的大小，单位是字节。

 

　　元数据：数据库的信息是存储在集合中的,它们使用了系统的命名空间： dbname.system.*

　　在MongoDB数据库中名字空间 <dbname>.system.* 是包含多种系统信息的特殊集合(Collection)，如下:

| 集合命名空间             | 描述                                      |
| ------------------------ | ----------------------------------------- |
| dbname.system.namespaces | 列出所有名字空间。                        |
| dbname.system.indexes    | 列出所有索引。                            |
| dbname.system.profile    | 包含数据库概要(profile)信息。             |
| dbname.system.users      | 列出所有可访问数据库的用户。              |
| dbname.local.sources     | 包含复制对端（slave）的服务器信息和状态。 |

　　

　　

 

 

 

　　

　MongoDB数据类型：

　

| 数据类型           | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| String             | 字符串。存储数据常用的数据类型。在 MongoDB 中，**UTF-8** 编码的字符串才是合法的。 |
| Integer            | 整型数值。用于存储数值。**根据你所采用的服务器，可分为 32 位或 64 位。** |
| Boolean            | 布尔值。用于存储布尔值（真/假）。                            |
| Double             | 双精度浮点值。用于存储浮点值。                               |
| Min/Max keys       | 将一个值与 BSON（二进制的 JSON）元素的最低值和最高值相对比。 |
| Arrays             | 用于将数组或列表或多个值存储为一个键。                       |
| Timestamp          | 时间戳。记录文档修改或添加的具体时间。                       |
| Object             | 用于内嵌文档。                                               |
| Null               | 用于创建空值。                                               |
| Symbol             | 符号。该数据类型基本上等同于字符串类型，但不同的是，它一般用于采用特殊符号类型的语言。 |
| Date               | 日期时间。用 UNIX 时间格式来存储当前日期或时间。你可以指定自己的日期时间：**创建 Date 对象，传入年月日信息**。 |
| Object ID          | 对象 ID。用于创建文档的 ID。                                 |
| Binary Data        | 二进制数据。用于存储二进制数据。                             |
| Code               | **代码类型**。用于在文档中存储 JavaScript 代码。             |
| Regular expression | 正则表达式类型。用于存储正则表达式。                         |



**常用语句：**

**创建数据库：**use database_name

示例：use myfisrtmongoDB 

shell下提示：switched to db myfisrtmongoDB 表示成功

但此时用show dbs命令任然无法查看到新创建的数据库，需要插入第一条数据后，才可以查看。

 

**删除数据库：**db.dropDatabase() 

该命令是删除当前的数据库

提示：{"dropped":"myfirstmongoDB","ok":1}

 

**插入文档：**db.collections_name.insert(document)

表示在当前数据库的名为collections_name的集合中插入一条文档

示例：db.cols.insert({"content":"myFirstRecorder","by":"insaneXs"})

记录插入完成后，可以通过db.collections_name.find()命令来查看该数据库下名为collections_name的集合中全部的记录(文档)。

 

我们也可以将数据定义成一个变量，然后再通过上述命令插入

示例：document_name = {"content":"insertByDocumentVar"}

db.cols.insert(document_name)

定义了一个名为document_name的document变量，然后将document变量插入到集合中。

 

 

**更新文档：**mongoDB更新文档有两种方式，update()和save()

 

update()的语法格式：

db.collections_name.update(criteria, objNew, upsert, multi, writeConcern)

criteria:标准，即是查询条件，相当于SQL语句中的where语句

objNew: update的对象和一些更新的操作符（如$,$inc...）等，也可以理解为sql update查询内set后面的部分

upsert: boolean类型，可选，表示文档不存在时，是否插入该文档，true表示插入，数据库默认为false

multi:boolean类型，可选，表示多条记录满足时，是否更新全部记录，true表示更新全部记录,数据库默认为false

writerConcert:可选，表示抛出异常的级别

 

示例:

db.cols.update({"content":"insertByDocumentVar"},{$set:{"content":"recorderUpdateByupdate"},true,true})

意义：

将当前数据库下cols集合中content值为insertByDocumentVar的文档全部更新成set后的文档(字段如果存在，则更新值，不存在，则在之前基础上增加字段)，如果不存在，则插入该文档。

 

save():通过传入的文档来替换已有的文档

db.collections.save(x), x即是要更新的文档，mongoDB会根据x的_id的值去数据库中查找要更新的文档，如果不存在，则插入该文档。

示例：

db.cols.save({"_id":ObjectId("56a860650c412f8aa5c4c25a"),"content":"updateRecorderBySave"})

意义：将db数据库下的cols集合中的id为"56a860650c412f8aa5c4c25a"的文档替换（由于是文档被替换，因此如果新文档不含旧文档的其他字段时，其他字段将被删除。），没有则插入该记录（插入时id将不是自动生成，而就是id的值）。

 

mongoDB中的更新操作符

**$inc** 对一个int字段增加一个val值

{$inc:{filed:value}}

示例：

db.cols.update({"count":1},{$inc : {"count":1}})

意义：将当前数据库的cols集合中count等于1的文档中的count字段增加1

 

**$set** 相当于sql的set field = value，全部数据类型都支持

用法：{ $set : { field : value } }
示例：参考文章开头

 

**$unset** 删除某个字段

用法：{$unset:{filed : value}}

数据库先插入一条数据db.cols.insert{"filed1":"1","filed2":"2"};

示例：db.cols.update({"filed1":"1"},{$unset:{"filed2":"2"}})

用db.cols.find().pretty()查询

看到结果为：{ "_id" : ObjectId("56a874310c412f8aa5c4c261"), "field1" : "1" }

 

**$push** 将某个值追加到指定的filed中，该field必须要是数组

用法：{$push:{filed:value}}

示例：

数据库先插入一条数据 db.cols.insert{"myArray":["a","b"], "count":1}

db.cols.update({"content":1},{$push:{"c"}})

在查询集合，可看到结果：

　　"_id" : ObjectId("56a877ec04d77a4fe6e85ffb"),
　　"count" : 1,
　　"myArray" : [
　　"a",
　　"b",
　　"c"
]

 

**$pushAll** 和$push类似，只是$pushAll可以追加多个值到某个数组字段中

用法：{$pushAll:{field:array}}

示例同上，不再赘述，但是要注意字段值必须要是array

 

**$addToSet** 仅当某个值不再数组内时，才往数组中添加这个值

用法：{$addToSet:{field : value}}

示例同上，field必须要是数组。

 

**$pop** 删除数组内的一个值

用法：

删除数组最后一个值：{$pop :{field : 1}}

删除数组头一个值：{$pop : {field : -1}}

 

**$pull** 从field的数组内删除值为value的值

用法：{$pull : {filed:value}}

用法同上，如果有多个值为value，则全部删除

 

**$pullAll** 从field的数组内删除在数组array中的值

用法：{$pull : {filed : array}}

用法同上。

 

**$** 操作符 表示从数组中找出是自己的那项

　因为得到的是该项的filed值，故要加上引号(红字部分)。

```
看一下官方的例子：
> t.find()
{ "_id" : ObjectId("4b97e62bf1d8c7152c9ccb74"), "title" : "ABC",  "comments" : [ { "by" : "joe", "votes" : 3 }, { "by" : "jane", "votes" : 7 } ] }

> t.update( {'comments.by':'joe'}, {$inc:{'comments.$.votes':1}}, false, true )

> t.find()
{ "_id" : ObjectId("4b97e62bf1d8c7152c9ccb74"), "title" : "ABC",  "comments" : [ { "by" : "joe", "votes" : 4 }, { "by" : "jane", "votes" : 7 } ] }

需要注意的是，$只会应用找到的第一条数组项，后面的就不管了。
还需要注意的是，$和$unset联合使用会在数组中留下一条为NULL的记录> t.insert({x: [1,2,3,4,3,2,3,4]}> t.find({ "_id" : ObjectId("4bde2ad3755d00000000710e"), "x" : [ 1, 2, 3, 4, 3, 2, 3, 4 ] }> t.update({x:3}, {$unset:{"x.$":1}})
> t.find()
{ "_id" : ObjectId("4bde2ad3755d00000000710e"), "x" : [ 1, 2, null, 4, 3, 2, 3, 4 ] }


MongoDB删除文档：db.collection_name.remove()
用法：db.collection_name.remove(query, justone)
query:查询条件
justone:表示有多条记录满足时，是否要同时删除，可选，默认为false

删除一个集合中的全部文档：db.collection_name.remove({})
只删除第一个符合条件的元素：db.collection_name.remove({条件},1)


MongoDB查询文档：db.collection_name.find()
将查询结果格式化显示：db_collection_name.find().pretty()

AND条件查询：
用法：db.collection_name.find({filed1:value,field2:value....})

OR条件查询：
用法：db.collection_name.find({$or:[{filed:value},{field:value}]});
OR条件查询像是把多个条件放在一个数组中，然后由该数组构成一个文档，而AND条件查询则是各个条件直接构成一个文档。
```





**MongoDB条件操作符**

- $gt  > 大于
- $lt  < 小于
- $gte >= 大于等于
- $lte  <= 小于等于
- $ne  !=  不等于

条件操作符可用于查询语句中，帮助刷选查询结果。

用法：{field:{条件操作符 :  value}} 表示该字段的值符合条件操作符所表示的关系

示例：db.cols.find("age":{$gt: 20})

表示在当前数据库的cols集合中查找出age>20的文档。

条件操作符不仅能对数字进行比较，对字符串也能进行比较。

 

表示大于且小于

示例:db.cols.find("age":{$lt:25, $gt:22})

注：大于小于在一个{}内。

 

**MongoDB类型操作符**

$type：类型操作符，用于筛选出某个字段的值为某个类型的文档。

用法:{$type : value}

每个类型对应一个value,参见下表：

| **类型**                | **value** | **备注**         |
| ----------------------- | --------- | ---------------- |
| Double                  | 1         |                  |
| String                  | 2         |                  |
| Object                  | 3         |                  |
| Array                   | 4         |                  |
| Binary data             | 5         |                  |
| Undefined               | 6         | 已废弃。         |
| Object id               | 7         |                  |
| Boolean                 | 8         |                  |
| Date                    | 9         |                  |
| Null                    | 10        |                  |
| Regular Expression      | 11        |                  |
| JavaScript              | 13        |                  |
| Symbol                  | 14        |                  |
| JavaScript (with scope) | 15        |                  |
| 32-bit integer          | 16        |                  |
| Timestamp               | 17        |                  |
| 64-bit integer          | 18        |                  |
| Min key                 | 255       | Query with `-1`. |
| Max key                 | 127       |                  |

 

 

 

 

 

 

 

 

 

 

 

 

 

 

示例：db.cols.find("filed_name":{$type : 2})

表示在当前数据库的cols集合中查找出符合"filed_name"这个字段值为String的文档。

 

 

**MongoDB的limit()和skip()方法**

limit()方法可以限制查询的文档数，如果结果大于限制的数量，那么只显示出限制容量大小的文档组。

用法：db.cols.find().limit(LIMIT_NUM)

注：不填LIMIT_NUM时默认显示一条

 

skip()方法可以跳过一定数量的文档之后进行查找。

用法：db.cols.find().skip(SKIP_NUM)

注：不填SKIP_NUM时出错。

 

limit()和skip()可以混合使用，但db.cols.find().limt(3).skip(1)和db.cols.find().skip(1).limit(3)意义相同，都表示跳过一个记录，限制查找到的之后三条。

还有一点需要注意的是当重复使用limt()时，结果也会被多次筛选。如：db.cols.find().limit(3).limit(1)最后只显示一条文档。

 

**MongoDB的sort()排序**

用法：db.cols.find().sort({field:1或-1}) 1表示升序，-1表示降序，filed表示所要根据排序的字段。如果某文档没有该字段，那么会被默认成最小值，排在之前。

 

 

**MongoDB索引**

MongoDB中的索引和关系型数据库下的索引概念基本一致，都是为了能够在大量的数据下快速的查找出所需的数据。

创建索引：

用法：db.cols.ensureIndex({field1:1|-1,field2:1|-1...})

表示根据字段field1,(field2等...)创建索引，其中的1和-1表示升序或者降序。当只有一个字段时，所创建的就单个索引(单个索引的升序、降序效果相同)。当有多个字段时，创建的索引即是复合索引。

ensureIndex()还可接受一些可选参数，用于设置索引的属性。

用法：db.cols.ensureIndex({filed1:1|1,field2:1|1},{可选参数1：value, 可选参数2：value...})

可选参数列表如下：

| Parameter          | Type          | Description                                                  |
| ------------------ | ------------- | ------------------------------------------------------------ |
| background         | Boolean       | 建索引过程会**阻塞其它数据库操作**，background可指定以后台方式创建索引，即增加 "background" 可选参数。 "background" 默认值为**false**。 |
| unique             | Boolean       | 建立的**索引是否唯一**。指定为true创建唯一索引。默认值为**false**. |
| name               | string        | 索引的名称。如果未指定，MongoDB的通过连接索引的字段名和排序顺序生成一个索引名称，如:field1_1_field2_1 |
| dropDups           | Boolean       | 在**建立唯一索引时是否删除重复记录**,指定 true 创建唯一索引。默认值为 **false**. |
| sparse             | Boolean       | 对文档中不存在的字段数据不启用索引；这个参数需要特别注意，如果设置为true的话，在索引字段中不会查询出不包含对应字段的文档.。默认值为 **false**. |
| expireAfterSeconds | integer       | 指定一个以秒为单位的数值，完成 TTL设定，设定集合的生存时间。 |
| v                  | index version | 索引的版本号。默认的索引版本取决于mongod创建索引时运行的版本。 |
| weights            | document      | 索引权重值，数值在 1 到 99,999 之间，表示该索引相对于其他索引字段的得分权重。 |
| default_language   | string        | 对于文本索引，该参数决定了停用词及词干和词器的规则的列表。 默认为英语 |
| language_override  | string        | 对于文本索引，该参数指定了包含在文档中的字段名，语言覆盖默认的language，默认值为 language. |

 

索引创建完成后，可以通过db.cols.getIndexes()来查看cols集合下全部的索引。

或是用过db.system.index.find()来查看当前数据库下的全部索引。(自己验证时，发现没有打印出记录)

 

删除索引：可以通过db.cols.dropIndexes()来删除全部的索引，但是不包括默认的_id字段的索引。

重建索引：可以通过db.cols.reIndex()

 

 

 **MongoDB中的聚合**

MongoDB的聚合管道将MongoDB文档在一个管道处理完毕后将结果传递给下一个管道处理。管道操作是可以重复的。

db.cols.aggregate([Array])

Array中可以填入多个表达式。

表达式：处理输入文档并输出。表达式是无状态的，只能用于计算当前聚合管道的文档，不能处理其它的文档。

- $project：修改输入文档的结构。可以用来**重命名、增加或删除域**，也可以用于创建计算结果以及嵌套文档。
- $match：用于**过滤数据**，只输出符合条件的文档。$match使用MongoDB的标准查询操作。
- $limit：用来限制MongoDB聚合管道返回的文档数。
- $skip：在聚合管道中跳过指定数量的文档，并返回余下的文档。
- $unwind：将文档中的某一个数组类型字段拆分成多条，每条包含数组中的一个值。
- $group：**将集合中的文档分组，可用于统计结果**。
- $sort：将输入文档排序后输出。
- $geoNear：输出接近某一地理位置的有序文档。

示例：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
> db.testdb.find()
{ "_id" : ObjectId("56aa0374f593495aa9328e3c"), "name" : "A", "age" : 22, "gender" : "f" }
{ "_id" : ObjectId("56aa0380f593495aa9328e3d"), "name" : "B", "age" : 24, "gender" : "f" }
{ "_id" : ObjectId("56aa03a4f593495aa9328e3e"), "name" : "C", "age" : 24, "gender" : "m" }
{ "_id" : ObjectId("56aa03bdf593495aa9328e3f"), "name" : "D", "age" : 25, "gender" : "f" }
{ "_id" : ObjectId("56aa03ccf593495aa9328e40"), "name" : "E", "age" : 21, "gender" : "m" }
{ "_id" : ObjectId("56aa03fcf593495aa9328e41"), "name" : "F", "age" : 18, "gender" : "f" }
{ "_id" : ObjectId("56aa040df593495aa9328e42"), "name" : "G", "age" : 21, "gender" : "m" }
> 
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

对该集合进行聚合，来统计各个性别的人数，

```
> db.testdb.aggregate([{$group:{"_id":"$gender",count : {$sum : 1}}}])
{ "_id" : "m", "count" : 3 }
{ "_id" : "f", "count" : 4 }
> 
```

如果想对文档在进行筛选，可以增加$match,

```
> db.testdb.aggregate([{$match:{"age":24}},{$group:{"_id":"$gender",count : {$sum : 1}}}])

{ "_id" : "m", "count" : 1 }
{ "_id" : "f", "count" : 1 }
```

对年龄为24的文档进行按性别统计。

 

下表展示了一些聚合的表达式:

| 表达式    | 描述                                           | 实例                                                         |
| --------- | ---------------------------------------------- | ------------------------------------------------------------ |
| $sum      | 计算总和。                                     | db.mycol.aggregate([{$group : {_id : "$by_user", num_tutorial : {$sum : "$likes"}}}]) |
| $avg      | 计算平均值                                     | db.mycol.aggregate([{$group : {_id : "$by_user", num_tutorial : {$avg : "$likes"}}}]) |
| $min      | 获取集合中所有文档对应值得最小值。             | db.mycol.aggregate([{$group : {_id : "$by_user", num_tutorial : {$min : "$likes"}}}]) |
| $max      | 获取集合中所有文档对应值得最大值。             | db.mycol.aggregate([{$group : {_id : "$by_user", num_tutorial : {$max : "$likes"}}}]) |
| $push     | 在结果文档中插入值到一个数组中。               | db.mycol.aggregate([{$group : {_id : "$by_user", url : {$push: "$url"}}}]) |
| $addToSet | 在结果文档中插入值到一个数组中，但不创建副本。 | db.mycol.aggregate([{$group : {_id : "$by_user", url : {$addToSet : "$url"}}}]) |
| $first    | 根据资源文档的排序获取第一个文档数据。         | db.mycol.aggregate([{$group : {_id : "$by_user", first_url : {$first : "$url"}}}]) |
| $last     | 根据资源文档的排序获取最后一个文档数据         | db.mycol.aggregate([{$group : {_id : "$by_user", last_url : {$last : "$url"}}}]) |

------

 

 

**环境配置**

在Java项目中使用MongoDB，需要在项目中引入mongo.jar这个包。下载地址：[下载](http://mvnrepository.com/artifact/org.mongodb/mongo-java-driver)[
](http://mvnrepository.com/artifact/org.mongodb/mongo-java-driver)

请尽量下载较新的版本，本文用的是2.10.1。

 

**连接MongoDB**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
1 public synchronized static DB getDBConnection() throws UnknownHostException{
2         if(db == null){
3             MongoClient client = new MongoClient(DB_SERVER_IP, DBSERVER_PORT);
4             db = client.getDB(DB_NAME);
5             System.out.println("GET DBCONNECTION SUCCESSFUL");
6         }
7         return db;
8     }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

其中IP和PORT分别是数据库服务端的IP和端口号。

 

**创建集合**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public static DBCollection getCollection(String colName){
        col = db.getCollection(colName);
    if(col == null){
        col = db.createCollection(colName, null);
    }
      return col;
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

**插入文档**

插入数据有4中方式可选：1.利用DBObjcet，2.利用DBObjectBuilder, 3.先创建Map对象，再用Map对象构造DBObject，4.直接通过json对象创建。

这里我们主要介绍第一种方式——利用DBObject插入。DBObject是一个接口，继承自BSONObject，是可被存入数据库的一个键值的Map。这里我们使用它的基本实现：BasicDBObject。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public static void insert(){
        DBCollection col = getCollection("myCollection1");
        if(col != null){
        DBObject o = new BasicDBObject();
        o.put("name", "Z");
        o.put("gender", "f");
        o.put("age", 1);
        col.insert(o);
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

另外，你也可以使用BasicDBObject提供的append()函数，来为对象插入键值对。

当你需要批量插入数据时，可以使用DBCollection.insert(List<DBObject> list);

 

**查询文档**

可以通过DBCollection.find()来查询集合中的文档。该函数返回一个游标DBCursor。通过对其迭代输出，就可以得到文档组。或者是通过DBCursor.toArray()直接转成DBObject的列表List<DBObject>。代码如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public static void search(){
        DBCollection col = getCollection("myCollection1");
        if(col != null){
            DBCursor cursor = col.find();
            while(cursor.hasNext()){
                DBObject o = cursor.next();
                System.out.println(o);
            }
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public static void search(){
        DBCollection col = getCollection("myCollection1");
        if(col != null){
            List<DBObject> list = col.find().toArray();
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

当然，也可以设置查询条件，并对输出结果的字段进行限制：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public static void search2(){
        DBCollection col = getCollection("myCollection1");
        if(col != null){
            //查询条件
            DBObject query = new BasicDBObject();
            query.put("gender", "m");
            
            //输出结果是否有要输出某个字段。0表示不输出，1表示只输出该字段
            DBObject field = new BasicDBObject();
            field.put("name", 1);
            
            DBCursor cursor = col.find(query,field);
            while(cursor.hasNext()){
                System.out.println(cursor.next());
            }
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

上述代码就表示查询性别为m的全部文档，并输出其姓名。

 

较复杂的条件查询：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
    public static void search2(){
        DBCollection col = getCollection("myCollection1");
        if(col != null){
            //查询条件
            DBObject query = new BasicDBObject();
            DBObject o = new BasicDBObject("$lt",24).append("$gt", 21);
            query.put("age", o);
                        
            DBCursor cursor = col.find(query);
            while(cursor.hasNext()){
                System.out.println(cursor.next());
            }
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

这里我们筛选的是年龄小于24且大于21的全部文档。

 

更新文档

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
    public static void update1(){
        DBCollection col = getCollection("myCollection1");
        if(col != null){
            //查询条件
            DBObject query = new BasicDBObject();
            query.put("name", "A");
            //用来替换的文档
            DBObject newObject = new BasicDBObject();
            newObject.put("name", "A1");
            col.update(query, newObject);
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

这里我们将name这个字段等于A的文档替换成了name字段为A1的值，注意的是，新的文档将不包含旧文档的其他字段，是真正意义上的两个文档的替换，而非替换相同字段！另外一点需要注意的是，该方法只替换第一条符合查询条件的文档。因为multi的默认值为false，可以通过设置这个值为true来修改多条。

findAndModity(DBObject query, DBObject fields, DBObject sort, boolean remove, DBObject update,boolean returnNew,boolean upsert)也提供了类似的功能*。*

`query` - query to match

`fields` - fields to be returned

`sort` - sort to apply before picking first document

`remove` - if true, document found will be removed

`update` - update to apply

`returnNew` - if true, the updated document is returned, otherwise the old document is returned (or it would be lost forever)`upsert` - do upsert (insert if document not present)

 

删除文档

删除指定的一个文档：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public static void remove1(){
        DBCollection col = getCollection("myCollection1");
        if(col != null){
            DBObject o = col.findOne();
            col.remove(o);
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

删除符合某条件的文档：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public static void remove2(){
        DBCollection col = getCollection("myCollection1");
        if(col != null){
            //条件
            DBObject query = new BasicDBObject();
            query.put("name", "A1");
            col.remove(query);
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

会删除符合条件的全部文档。

 

如需要删除集合下的全部文档时，可结合DBCursor实现。

 

参考：

API文档：http://api.mongodb.org/java/2.10.1/

其他资料：[菜鸟教程](http://www.runoob.com/mongodb/mongodb-java.html)

　　　　  http://blog.csdn.net/hx_uestc/article/details/7620938

在mongodb中有一个admin数据库，牵涉到服务器配置层面的操作，需要先切换到admin数据库，use admin命令相当于进入超级用户管理模式。

mongo的用户是以数据库为单位来建立的，每个数据库有自己的管理员。

我们在设置用户时，需要先在admin数据库下建立管理员，这个管理员登陆后，相当于超级管理员，然后可以切换到其他库，添加普通用户。

注意: mongodb服务器启动时，默认是不需要认证的，要让用户生效，需要启动服务器时指定–auth选项。添加用户后，我们再次退出并登陆，认证才有效。

# 1. 创建管理员

```bash
# (1)创建用户管理员(在管理身份验证数据库)。
use admin
db.createUser(
  {
    user: "admin",
    pwd: "xxx",
    roles: [{role: "userAdminAnyDatabase", db: "admin"}]
  }
)

# (2)重新启动MongoDB实例与访问控制。
mongod --dbpath /data/mongodb/database --logpath /data/mongodb/log/mongodb.log --port 27017 --fork --auth

# (3)连接和用户管理员进行身份验证。
mongo --port 27017 -u "krislin" -p "xxx" --authenticationDatabase "admin"
```

# 2. 创建自定义用户

```bash
# (1) 给其他数据库test配置一个读写权限的用户
use test
db.createUser(
  {
    user: "myTester",
    pwd: "xyz123",
    roles: [ { role: "readWrite", db: "test" },
             { role: "read", db: "reporting" } ]
  }
)

# (2)mytest连接和验证。
mongo --port 27017 -u "myTester" -p "xyz123" --authenticationDatabase "test"
# 或登录后执行命令验证
use test
db.auth("myTester","xyz123")
```

# 3. 查看已经创建的用户

```bash
# 查看创建的用户需要在管理员数据库上执行
use admin
db.system.users.find()

# 查看当前数据库用户
show users
```

# 4. 删除用户

```bash
# 删除一个用户
use dbname
db.system.users.remove({user:"username"})

# 删除管理员
use admin
db.system.users.remove({user:"admin"})
```

# 1. 库和集合的基础命令

## 1. 库级命令

```bash
# 查看所有数据库
show dbs # 或使用mysql命令show databases;

# 切换到指定数据库
use 库名

# 创建数据库(mongodb不提供命令)
# mongodb的数据库是隐式创建的(创建表时自动创建)，就算没有指定的库，也可以使用use命令。

# 删除数据库
db.dropDatabase()
```

## 2. 表(集合)级命令

```bash
# 显示所有表
show collections; # 或mysql命令show tables;

# 创建表
db.createCollection('user')

# 删除表
db.user.drop()
```

# 2. insert插入数据

## 1. 增加一个文档

```bash
# 不指定id
db.user.insert({name:'张三',age:22})

# 指定id
db.user.insert({_id:10001,name:'李四',age:23})
```



## 2. 一次增加多个文档

```bash
db.user.insert([{name:'张三',age:24},{_id:10002,name:'李四',age:25},{name:'王五',age:26}])
```

# 3. remove删除数据

语法： 

```bash
db.collection.remove(查询表达式,选项)
```

查询表达式: 类似mysql的where条件，不加条件会删除表所有数据。 选项: 是否删除一行，{justOne:true/false}，默认是false

```bash
# 示例
db.user.remove({_id:10001})
db.user.remove({age:24},{ justOne:true})
```

注：实际应用中一般不使用删除，使用一个字段来标记删除。

# 4. update修改数据

语法：

> ```bash
> db.collection.updae(查询表达式,新值,选项)
> ```

错误示例：

> ```bash
> db.user.update({name:‘张三’},{age:33})
> ```

注意：这种方式是新文档直接替换了旧文档，必须指定赋值表达式，常用表达式有：

| 表达式       | 说明                     |
| :----------- | :----------------------- |
| $set         | 设置字段新的值           |
| $unset       | 删除指定的列             |
| $inc         | 增长                     |
| $rename      | 重命名该列               |
| $setOnInsert | 当upsert时，设置字段的值 |

## 1. 赋值表达式

```bash
# $set设置字段新值(如果字段不存在则新增加该列)
db.user.update({name:'张三'},{$set:{age:33}})

# $inc自增字段值
db.user.update({name:'张三'},{$inc:{age:1}})

# $rename修改字段名
db.user.update({name:'张三'},{$rename:{sex:'gender'}})

# $unset删除指定列
db.user.update({name:'张三'},{$unset:{phone:1}})

# setOnInsert当upsert为true时，添加附加字段
db.user.update({name:'赵六'},{$setOnInsert:{age:25,gender:'male'}},{upsert:true})

# 多种赋值表达可以一次执行
db.collection.updae(查询表达式,{
{$set:{...}},
{$unset:{...}},
{$inc:{...}},
{$rename:{...}}
})
```

## 2. 修改选项

选项包括upsert和multi

- upsert：没有匹配的行，是否直接插入该行，默认为false。
- multi：查询表达式匹配中多行，是否修改多行，默认false(只修改一行)

```bash
# 示例
db.user.update({name:'李四'},{$set:{age:44}},{upsert:true})
db.user.update({gender:'male'},{$set:{age:55}},{multi:true})
```

# 5. query查询

```bash
# 语法：db.collection.find(查询表达式,查询的列)
db.user.find() # 查询所有

# 查询匹配条件的所有列
db.user.find({name:'张三'})

# 查询匹配条件的指定列({列名:1,列名:1,...})
db.user.find({name:'张三'},{age:1})

# 不查询id列
db.user.find({name:'张三'},{_id:0,age:1})
```

## 1. 字段值查询

```bash
# 查询主键为32的商品
db.goods.find({goods_id:32})

# 子文档查询(指向子文档的key是字符串)
db.stu.find({'score.yuwen':75})

# 数组元素查询(指向子文档的key是字符串)
db.stu.find({'hobby.2':'football'})
```

## 2. 范围查询

```bash
# $ne 不等于
# 查询不属第3栏目的所有商品
db.goods.find({cat_id:{$ne:3}})

# $gt 大于
# 查询高于3000元的商品
db.goods.find({shop_price:{$gt:3000}})

# $gte 大于等于
# 查询高于或等于3000元的商品
db.goods.find({shop_price:{$gte: 3000}})

# $lt 小于
# 查询低于500元的商品
db.goods.find({shop_price:{$lt:500}})

# $lte 小于等于
# 查询低于500元的商品
db.goods.find({shop_price:{$lte:500}})
```

## 3. 集合查询

```bash
# $in 在集合，(当查询字段是数组时，不需完全匹配也会命中该行)
# 查询栏目为4和11的商品
db.goods.find({cat_id:{$in:[4,11]}})

# $nin 不在集合
# 查询栏目不是3和4的商品
db.goods.find({cat_id:{$nin:[3,4]}})

# $all完全匹配，(当查询字段是数组时，必须完全匹配才命中该行)
db.goods.find({cat_id:{$all:[4,11]}})
```

## 4. 逻辑查询

```bash
# $and 逻辑与，必须都满足条件
# 查询价格为100到500的商品
db.goods.find({$and:[{shop_price:{$gt:100}},{shop_price:{$lt:500}}]})

# $or 逻辑或，至少满足一个条件
# 查询价格小于100或大于3000的商品
db.goods.find({$or:[{shop_price:{$lt:100}},{shop_price:{$gt:3000}}]})

# $nor 与非逻辑
# 查询不在栏目3，并且价格不小1000的商品
db.goods.find({$nor:[{cat_id:3},{shop_price:{$lt:1000}}]})
```

## 5. 元素运算符查询

```bash
# $mod 求余
# 查询年龄对10求余为0的用户
db.goods.find({age:{$mod:[10,0]}})

# $exist 查询列是否存在
# 查询有电话属性的用户
db.user.find({phone:{$exists:1}})

# $type 查询属性为指定类型的文档
# 查询为浮点型的文档
db.user.find({phone:{$type:1}})

# 查询为字符串型的文档
db.user.find({phone:{$type:2}})
```

## 6. 其他常用查询

```bash
# 统计行数
db.goods.count()
db.goods.find({cat_id:3}).count()

# 查询每页数据
    # 按指定属性排序，1表示升序，-1表示降序
    sort({属性:1|-1})
    # 跳过行数
    skip(num)
    # 限制取多少行
    limit(num)

# 页码page，每页行数n，则skip数=(page-1)*n
db.goods.find({cat_id:3}).sort({shop_price:-1}).skip(0).limit(10)
```

索引提高查询速度，降低写入速度，权衡常用的查询字段，不建议在太多列上建索引。在mongodb中，索引可以按字段升序/降序来创建，便于排序。默认是用btree来组织索引文件，也允许建立hash索引。

```bash
# 在test库下stu下创建10000行数据的成绩表
for (var i=1;i<=10000;i++){
	db.stu.insert({sn:i,name:'stu'+i,email:'stu'+i+'@126.com',score:{yuwen:i%80,shuxue:i%90,yingyu:i%100}})
}
```

# 1. 普通索引

## 1. 单列索引

在表stu创建sn列索引

> ```bash
> db.stu.ensureIndex({sn:1})
> ```

1表示升序，-1表示降序

## 2. 多列索引

在表stu创建sn列和name列共同索引

> ```bash
> db.stu.ensureIndex({sn:1,name:1})
> ```

1表示升序，-1表示降序

## 3. 子文档索引

在表stu的score列下的yuwen字段创建索引

> ```bash
> db.stu.ensureIndex({‘score.yuwen’:1})
> ```

1表示升序，-1表示降序

# 2. 唯一索引

创建唯一索引后字段值都是唯一的

在表stu创建email列索引

> ```bash
> db.stu.ensureIndex({email:1},{unique:true})
> ```

# 3. 稀疏索引

稀疏索引的特点：如果针对field做索引，针对不含field列的文档，将不建立索引。与之相对的普通索引会把该文档的field列的值认为NULL，并建索引。

使用场景：小部分文档含有某列时。

在表stu创建phone列稀疏索引

> ```bash
> db.stu.ensureIndex({age:1},{sparse:true})
> ```

# 4. 哈希索引

哈希索引速度比普通索引快，缺点是不能对范围查询进行优化。使用场景：随机性强的散列

在表stu创建email列哈希索引

> ```bash
> db.stu.ensureIndex({email:‘hashed’})
> ```

# 5. 重建索引

一个表经过很多次修改后，导致表的文件产生空洞，索引文件也如此。可以通过索引的重建，减少索引文件碎片，并提高索引的效率，类似mysql中的optimize table

在表stu重建索引

>```bash
>db.stu.reIndex()
>```

# 6. 删除索引

```bash
# 语法
db.collection.dropIndex({filed:1/-1});

# 示例
db.stu.dropIndex({sn:1})
db.stu.dropIndex ({email:'hashed'})
```

# 7. 查看索引和执行计划

```bash
# 查看表索引
db.stu.getIndexes()

# 查看执行计划
db.stu.find({sn:5555}).explain() # 默认只输出queryPlanner

# 其中explain()参数有三个，分别是'queryPlanner'、'executionStats'、'allPlansExecution'
db.stu.find({sn:5555}).explain('executionStats')

# explain分析结果的几个重要字段，通过结果分析可以判断是否需要优化执行语句
```

executionStats属性下的字段：

- executionTimeMillis：查询耗时，单位(ms)
- totalDocsExamined：扫描文档数
- executionStages.stage：”COLLSCAN”表示全表扫描，”FETCH”表示索引扫描
- executionStages. executionTimeMillisEstimate：索引扫描耗时，单位(ms)

winningPlan.inputStage属性下的字段：

- indexName：索引名字

# 1. mongoexport导出json/csv结构化数据

mongoexport命令导出的只有数据，不包括索引，mongoexport参数说明：

- -h：主机ip或域名 (默认localhost)
- –port：mongodb使用端口 (默认27107)
- -u：认证用户名 (当需要认证时用)
- -p：认证密码 (当需要认证时用)
- -d：指定导出的库名
- -c：指定导出的表名
- -f：指定导出的列名
- -q：查询条件，例如：’{sn:{“$lte”:100}}’
- -o：保存导出数据文件位置
- –csv：指定导出csv格式 (便于和传统数据库交换数据)，默认导出的json格式

```bash
# 导出json数据
mongoexport -h 192.168.8.200 --port 27017 -u vison -p 123456 -d test -c stu -f sn,name,email -q '{sn:{"$lte":100}}' -o /home/vison/src/test.stu.json

# 导出csv数据
mongoexport -h 192.168.8.200 --port 27017 -u vison -p 123456 -d test -c stu -f sn,name,email -q '{sn:{"$lte":100}}' --csv -o /home/vison/src/test.stu.csv
```

# 2. mongoimport导入json/csv结构化数据

mongoimport命令导入的只有数据，不包括索引，mongoimport参数说明：

- -h：主机ip或域名 (默认localhost)
- –port：mongodb使用端口 (默认27107)
- -u：认证用户名 (当需要认证时用)
- -p：认证密码 (当需要认证时用)
- -d：指定导入的库名
- -c：指定导入的表名(不存在会自己创建)
- –type：csv/json(默认json)
- –headline：当导入csv文件时，需要跳过第一行列名
- –file：导入数据文件的位置

```bash
# 导入json数据
mongoimport -h 192.168.8.200 --port 27017 -u vison -p 123456 -d test -c stu_json --type json --file /home/vison/src/test.stu.json

# 导入csv数据
mongoimport -h 192.168.8.200 --port 27017 -u vison -p 123456 -d test -c stu_csv --type csv --headerline --file /home/vison/src/test.stu.csv

# 注：老版本需要指定-fields参数
```

# 3. mongodump导出二进制数据

mongodump导出数据是包括索引的，mongodump的参数说明：

- -h：主机ip或域名 (默认localhost)
- –port：mongodb使用端口 (默认27107)
- -u：认证用户名 (当需要认证时用)
- -p：认证密码 (当需要认证时用)
- -d：指定导出的库名
- -c：指定导出的表名 (可选)
- -q：查询条件(可选)，例如：’{sn:{“$lte”:100}}’
- -o：保存导出数据文件位置(默认是导出到mongo下的dump目录)
- –gzip：导出并压缩

示例：

```bash
mongodump -h 192.168.8.200 –port 27017 -u vison -p 123456 -d test –gzip -o /home/vison/src/mongoDump
```

注：可以写脚本每天凌晨访问少的时候备份一次数据

# 4. mongorestore导入二进制数据

mongorestore导入数据是包括索引的，mongorestore的参数说明：

- -h：主机ip或域名 (默认localhost)
- –port：mongodb使用端口 (默认27107)
- -u：认证用户名 (当需要认证时用)
- -p：认证密码 (当需要认证时用)
- -d：指定导出的库名
- -c：指定导出的表名 (可选)
- –dir：保存导入数据文件位置
- –gzip：导出并压缩

示例：

```bash
mongorestore -h 192.168.8.200 –port 27017 -u vison -p 123456 -d test –gzip –dir /home/vison/src/mongoDump/test
```

# 1. 获取mongodb状态信息

首先在admin下创建一个新能够获取mongodb状态权限的用户

```bash
use admin
db.createUser(
  {
    user: "stat",
    pwd: "123456",
    roles: [{role: "mongostatRole", db: "admin"}]
  }
)
```

每隔2秒采集一次信息保存到文件stat.log

```bash
mongostat -u stat -p 123456 –authenticationDatabase admin -n 2 –json >> stat.log
```

每一条统计信息如下：

```json
JSON{
    "localhost:27017":{
        "arw":"1|0",
        "command":"2|0",
        "conn":"4",
        "delete":"*0",
        "dirty":"0.0%",
        "flushes":"0",
        "getmore":"0",
        "insert":"*0",
        "net_in":"18.9k",
        "net_out":"79.0k",
        "qrw":"0|0",
        "query":"94",
        "res":"49.0M",
        "time":"14:41:32",
        "update":"*0",
        "used":"0.1%",
        "vsize":"938M"
    }
}
```

更多mongostat命令说明看官网 https://docs.mongodb.com/manual/reference/program/mongostat/

# 2. 非正常关闭mongodb导致无法启动的解决方法

非正常关闭包括断电或强制关闭，造成文件mongod.lock锁住了，所以无法正常启动，解决方法：

```bash
(1) 删除mongod.lock文件，文件存放一般在数据库文件夹里
rm /data/mongodb/db/mongod.lock

(2) repair的模式启动
mongod -f /usr/local/mongodb/mongodb.conf –repair

(3) 启动mongodb
mongod -f /usr/local/mongodb/mongodb.conf
```

# SQL 到 Mongo 的对应表

这个列表是 PHP 版本的 [» SQL to Mongo](http://www.mongoing.com/docs/reference/sql-comparison.html) 对应表（在 MongoDB 官方手册中有更加通用的版本）。

| SQL 查询语句                                     | Mongo 查询语句                                               |
| ------------------------------------------------ | ------------------------------------------------------------ |
| CREATE TABLE USERS (a Number, b Number)          | 隐式的创建，或 [MongoDB::createCollection()](mongodb.createcollection.php). |
| INSERT INTO USERS VALUES(1,1)                    | $db->users->insert(array("a" => 1, "b" => 1));               |
| SELECT a,b FROM users                            | $db->users->find(array(), array("a" => 1, "b" => 1));        |
| SELECT * FROM users WHERE age=33                 | $db->users->find(array("age" => 33));                        |
| SELECT a,b FROM users WHERE age=33               | $db->users->find(array("age" => 33), array("a" => 1, "b" => 1)); |
| SELECT a,b FROM users WHERE age=33 ORDER BY name | $db->users->find(array("age" => 33), array("a" => 1, "b" => 1))->sort(array("name" => 1)); |
| SELECT * FROM users WHERE age>33                 | $db->users->find(array("age" => array('$gt' => 33)));        |
| SELECT * FROM users WHERE age<33                 | $db->users->find(array("age" => array('$lt' => 33)));        |
| SELECT * FROM users WHERE name LIKE "%Joe%"      | $db->users->find(array("name" => new MongoRegex("/Joe/")));  |
| SELECT * FROM users WHERE name LIKE "Joe%"       | $db->users->find(array("name" => new MongoRegex("/^Joe/"))); |
| SELECT * FROM users WHERE age>33 AND age<=40     | $db->users->find(array("age" => array('$gt' => 33, '$lte' => 40))); |
| SELECT * FROM users ORDER BY name DESC           | $db->users->find()->sort(array("name" => -1));               |
| CREATE INDEX myindexname ON users(name)          | $db->users->ensureIndex(array("name" => 1));                 |
| CREATE INDEX myindexname ON users(name,ts DESC)  | $db->users->ensureIndex(array("name" => 1, "ts" => -1));     |
| SELECT * FROM users WHERE a=1 and b='q'          | $db->users->find(array("a" => 1, "b" => "q"));               |
| SELECT * FROM users LIMIT 20, 10                 | $db->users->find()->limit(10)->skip(20);                     |
| SELECT * FROM users WHERE a=1 or b=2             | $db->users->find(array('$or' => array(array("a" => 1), array("b" => 2)))); |
| SELECT * FROM users LIMIT 1                      | $db->users->find()->limit(1);                                |
| EXPLAIN SELECT * FROM users WHERE z=3            | $db->users->find(array("z" => 3))->explain()                 |
| SELECT DISTINCT last_name FROM users             | $db->command(array("distinct" => "users", "key" => "last_name")); |
| SELECT COUNT(*y) FROM users                      | $db->users->count();                                         |
| SELECT COUNT(*y) FROM users where AGE > 30       | $db->users->find(array("age" => array('$gt' => 30)))->count(); |
| SELECT COUNT(AGE) from users                     | $db->users->find(array("age" => array('$exists' => true)))->count(); |
| UPDATE users SET a=1 WHERE b='q'                 | $db->users->update(array("b" => "q"), array('$set' => array("a" => 1))); |
| UPDATE users SET a=a+2 WHERE b='q'               | $db->users->update(array("b" => "q"), array('$inc' => array("a" => 2))); |
| DELETE FROM users WHERE z="abc"                  | $db->users->remove(array("z" => "abc"));                     |


## 内嵌、引用选择

| 更适合内嵌                       | 更适合引用             |
| -------------------------------- | ---------------------- |
| 子文档较小                       | 子文档较大             |
| 数据不会定期改变                 | 数据经常改变           |
| 最终数据一致即可                 | 中间阶段的数据必须一致 |
| 文档数据小幅增加                 | 文档数据大幅增加       |
| 数据通常需要执行二次查询才能获得 | 数据通常不包含在结果中 |
| 快速读取                         | 快速写入               |