---
layout: post
categories: [Mongodb]
description: none
keywords: MongoDB
---
# MongoDB源码BSON
MongoDB 作为一款流行的文档数据库，采用 BSON 格式来支持文档模型。

## BSON 是什么
BSON 全称是 Binary JSON, 和 JSON 很像，但是采用二进制格式进行存储。相比 JSON 有以下优势：
- 访问速度更快。
BSON 会存储 Value 的类型，相比于明文存储，不需要进行字符串类型到其他类型的转换操作。以整型 12345678 为例，JSON 需要将字符串转成整型，而 BSON 中存储了整型类型标志，并用 4 个字节直接存储了整型值。对于 String 类型，会额外存储 String 长度，这样解析操作也会快很多；
- 存储空间更低。
还是以整型 12345678 为例，JSON 采用明文存储的方式需要 8 个字节，但是 BSON 对于 Int32 的值统一采用 4 字节存储，Long 和 Double 采用 8 字节存储。 当然这里说存储空间更低也分具体情况，比如对于小整型，BSON 消耗的空间反而更高；
- 数据类型更丰富。
BSON 相比 JSON，增加了 BinData，TimeStamp，ObjectID，Decimal128 等类型。

BSON是一种类json的一种二进制形式的存储格式，简称Binary JSON，它和JSON一样，支持内嵌的文档对象和数组对象，但是BSON有JSON没有的一些数据类型，如Date和BinData类型。

MongoDB 官方文档 对此有比较权威直观的描述，总结如下：

|      | JSON                           | BSON                                                                                   |
|------|--------------------------------|----------------------------------------------------------------------------------------|
| 编码方式 | UTF-8 String                   | Binary                                                                                 |
| 数据类型 | String, Boolean, Number, Array | String, Boolean, Number (Integer, Float, Long, Decimal128...), Array, Date, Raw Binary |
| 可读性  | Human and Machine              | Machine Only                                                                           |

## BSON在MongoDB中的使用
MongoDB使用了BSON这种结构来存储数据和网络数据交换。把这种格式转化成一文档这个概念(Document)，因为BSON是schema-free的，所以在MongoDB中所对应的文档也有这个特征，这里的一个Document也可以理解成关系数据库中的一条记录(Record)，只是这里的Document的变化更丰富一些，如Document可以嵌套。

MongoDB以BSON做为其存储结构的一种重要原因是其可遍历性。

## BSON 存储格式
一条最简单的 BSON 文档，从前向后可以拆解成以下几个部分：

首先是文档的总长度， 占 4 个字节；
然后是多个BSONElement按照顺序排列。每个BSONElement包含的内容有：
- Value 类型，参考代码定义，占 1 个字节；
- Key 的 C-String 表示形式，只存储 C-String内容，不存储长度，以 '\0' 结尾，占 len(Key)+1 个字节；
- Value 的二进制存储，比如 Int32 占 4 字节，Long 和 Double 占 8 个字节等
- 文档以 '\0' 结尾，也就是在遍历 BSON 到末尾时，常见的 EOO(End Of Object)，占 1 个字节；

下面列举常用的 Int32, Double, String, 内嵌文档，Array 类型，并分析它们的 16 进制表现形式。
```
{“hello":"world"}
```
这是一个BSON的例子，其中"hello"是key name，它一般是cstring类型，字节表示是cstring::= (byte*) "/x00" ,其中*表示零个或多个byte字节，/x00表示结束符;

后面的"world"是value值，它的类型一般是string,double,array,binarydata等类型。


## BSON源码结构
- builder.h 包含bson所需的内存管理类和将bson对象转成内存的工具方法
- bsontypes.h 定义了bson所需的数据类型列表
- oid.h 定义Object ID的数据结构及实现
- bsonelement.h 定义了bson的节点
- bsonobj.h bson对象（主要对象，提供了数据的基本操作）
- bsonmisc.h 定义了与bson相关的助手函数（流输入/输出）
- bsonobjbuilder.h 创建bsonObj的类，用到了builder中定义的内存管理类
- bsonobjiterator.h 提供了一个类似STL的接口是element的iterator封装（只实现了基本接口）

## BSON c++ 代码分析
MongoDB源代码树中包括了BSON的代码库，你只要包含bson.h这个头文件就行了，其中有四个类是比较重要的：

- mongo::BSONObj，这个是BSON对象的表示
- mongo::BSONElement，这个是BSON对象中元素的表示方法
- mongo::BSONObjBuilder,这是构建BSON对象的类
- mongo::BSONObjIterator，这是用来遍历BSON对象中每一个元素的一个迭代器

下面是创建一个BSON对象
```
BSONObjBuilder b; 
b.append("name","lemo"), 
b.append("age",23); 
BSONObj p = b.obj(); 
```
或者
```
BSONObj p = BSONObjBuilder().append("name","lemo").append("age",23).obj(); 
```

## 源码实现
这里分析一下这四个类的一些代码：

mongo::BSONObj主要是用于存储BSON对象的，具体的存储格式如下
```
 <unsigned totalSize> {<byte BSONType><cstring FieldName><Data>}* EOO 
--------------------              -------------                -----------------               ----           --- 
totalSize: 一个总的字节长度，包含自身 
BSONType: 对象类型,这里有Boolean,String,Date等类型，具体可以参考bsontypes.h这个文件 
FieldName: 这里表示字段名 
Data: 这里是放具体的数据，数据的存储方式根据不同的BSONType来 
* : 表示可以有多个元素组成 
EOO: 这是一个结束符，一般是/x00来表示 
```
一般来说，BSONObj的创建都是通过BSONObjBuilder来做的，除非你已经得到了其字节流，那可以直接生成BSONObj


mongo::BSONElement 它主要是用于存储对象中的单个元素，存储格式如下
```
<type><fieldName><value> 
```
这个对象主要是指向BSONObj对象中具体元素的地址，它不实际存储元素的值。
mongo::BSONObjBuilder 它主要是用于生成BSONObj，这个对象集成了StringBuilder,它主要用于存储实际的字节点，用于替换std::stringstream，而这个StringBuilder集成了BufBuilder,这是一个可以动态增长内存缓冲区，但最大容量不能超过64MB的大小，也就是说一个BSONObj最大不能超过64MB。

mongo::BSONOBjIterator 它主要是用来遍历BSONObj对象中的每一个元素，提供了类似于stl iterator的一些接口，它还提供了一个ForEach宏来提供更方便的操作，如
```
if (foo) { 
           BSONForEach(e, obj) 
               doSomething(e); 
       } 
```


## java json 转 bson
在开发中，经常需要将json格式的数据转换为bson格式，而Java语言中则可以用bson-util库来进行转换。下面介绍如何使用该库实现转换过程。

首先需要在pom.xml中加入以下依赖：
```
<dependency>
<groupId>org.mongodb</groupId>
<artifactId>bson</artifactId>
<version>3.6.3</version>
</dependency>
```

接下来，可以定义一个JsonUtils工具类，其中包含toJson方法和toBson方法：
```
import org.bson.Document;
import org.json.JSONObject;
public class JsonUtils {
public static JSONObject toJson(Document doc) {
JSONObject json = new JSONObject(doc.toJson());
return json;
}
public static Document toBson(JSONObject json) {
Document doc = Document.parse(json.toString());
return doc;
}
}
```
其中，toJson方法接收一个Document类型的参数，将其转换为JSONObject类型，而toBson方法接收一个JSONObject类型的参数，将其转换为Document类型。在代码中使用示例如下：
```
//将json转换为bson
String jsonString = "{\"name\":\"Jack\", \"age\":20}";
JSONObject json = new JSONObject(jsonString);
Document doc = JsonUtils.toBson(json);
System.out.println(doc.toJson());
//将bson转换为json
Document bson = new Document("name", "Jack").append("age", 20);
JSONObject json2 = JsonUtils.toJson(bson);
System.out.println(json2.toString());
```
以上代码将输出以下结果：
```
{"name": "Jack", "age": 20}
{"name":"Jack","age":20}
```


