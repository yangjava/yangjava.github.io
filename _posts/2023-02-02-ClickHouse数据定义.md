---
layout: post
categories: [ClickHouse]
description: none
keywords: ClickHouse
---
# ClickHouse数据定义

## ClickHouse的数据类型
作为一款分析型数据库，ClickHouse提供了许多数据类型，它们可以划分为基础类型、复合类型和特殊类型。其中基础类型使ClickHouse具备了描述数据的基本能力，而另外两种类型则使ClickHouse的数据表达能力更加丰富立体。

### 基础类型
基础类型只有数值、字符串和时间三种类型，没有Boolean类型，但可以使用整型的0或1替代。

- 数值类型
数值类型分为整数、浮点数和定点数三类，接下来分别进行说明。

Int
在普遍观念中，常用Tinyint、Smallint、Int和Bigint指代整数的不同取值范围。而ClickHouse则直接使用Int8、Int16、Int32和Int64指代4种大小的Int类型，其末尾的数字正好表明了占用字节的大小（8位=1字节）
ClickHouse支持无符号的整数，使用前缀U表示

Float
与整数类似，ClickHouse直接使用Float32和Float64代表单精度浮点数以及双精度浮点数

在使用浮点数的时候，应当要意识到它是有限精度的。假如，分别对Float32和Float64写入超过有效精度的数值，下面我们看看会发生什么。例如，将拥有20位小数的数值分别写入Float32和Float64，此时结果就会出现数据误差：
```
:) SELECT toFloat32('0.12345678901234567890') as a , toTypeName(a)
┌──────a─┬─toTypeName(toFloat32('0.12345678901234567890'))─┐
│ 0.12345679 │ Float32                                          │
└────────┴───────────────────────────────┘
 
:) SELECT toFloat64('0.12345678901234567890') as a , toTypeName(a)
┌────────────a─┬─toTypeName(toFloat64('0.12345678901234567890'))─┐
│ 0.12345678901234568 │ Float64                                          │
└─────────────┴──────────────────────────────┘
```
可以发现，Float32从小数点后第8位起及Float64从小数点后第17位起，都产生了数据溢出。

ClickHouse的浮点数支持正无穷、负无穷以及非数字的表达方式。

正无穷：
```
:) SELECT 0.8/0
┌─divide(0.8, 0)─┐
│ inf             │
└──────────┘
```
负无穷：
```
:) SELECT -0.8/0
┌─divide(-0.8, 0)─┐
│ -inf             │
└───────────┘
```
非数字：
```
:) SELECT 0/0
┌─divide(0, 0)──┐
│ nan             │
└──────────┘
```
## Decimal
如果要求更高精度的数值运算，则需要使用定点数。ClickHouse提供了Decimal32、Decimal64和Decimal128三种精度的定点数。可以通过两种形式声明定点：简写方式有Decimal32(S)、Decimal64(S)、Decimal128(S)三种，原生方式为Decimal(P,S)，其中：
- P代表精度，决定总位数（整数部分+小数部分），取值范围是1～38；
- S代表规模，决定小数位数，取值范围是0～P。

在使用两个不同精度的定点数进行四则运算的时候，它们的小数点位数S会发生变化。在进行加法运算时，S取最大值。例如下面的查询，toDecimal64(2,4)与toDecimal32(2,2)相加后S=4：
```
:) SELECT toDecimal64(2,4) + toDecimal32(2,2)
 
┌─plus(toDecimal64(2, 4), toDecimal32(2, 2))─┐
│ 4.0000                                       │
└───────────────────────────┘
```
在进行减法运算时，其规则与加法运算相同，S同样会取最大值。例如toDecimal32(4,4)与toDecimal64(2,2)相减后S=4：
```
:) SELECT toDecimal32(4,4) - toDecimal64(2,2)
┌─minus(toDecimal32(4, 4), toDecimal64(2, 2))┐
│ 2.0000                                     │
└────────────────────────────┘
```
在进行乘法运算时，S取两者S之和。例如下面的查询，toDecimal64(2,4)与toDecimal32(2,2)相乘后S=4+2=6：
```
:) SELECT toDecimal64(2,4) * toDecimal32(2,2)
┌─multiply(toDecimal64(2, 4), toDecimal32(2, 2))┐
│ 4.000000                                      │
└─────────────────────────────┘
```
在进行除法运算时，S取被除数的值，此时要求被除数S必须大于除数S，否则会报错。例如toDecimal64(2,4)与toDecimal32(2,2)相除后S=4：
```
:) SELECT toDecimal64(2,4) / toDecimal32(2,2)
┌─divide(toDecimal64(2, 4), toDecimal32(2, 2))┐
│  1.0000                                      │
└───────────────────────────┘
```
在使用定点数时还有一点值得注意：由于现代计算器系统只支持32位和64位CPU，所以Decimal128是在软件层面模拟实现的，它的速度会明显慢于Decimal32与Decimal64。

### 字符串类型
字符串类型可以细分为String、FixedString和UUID三类。从命名来看仿佛不像是由一款数据库提供的类型，反而更像是一门编程语言的设计。

- String
字符串由String定义，长度不限。因此在使用String的时候无须声明大小。它完全代替了传统意义上数据库的Varchar、Text、Clob和Blob等字符类型。String类型不限定字符集，因为它根本就没有这个概念，所以可以将任意编码的字符串存入其中。但是为了程序的规范性和可维护性，在同一套程序中应该遵循使用统一的编码，例如“统一保持UTF-8编码”就是一种很好的约定。
- FixedString
FixedString类型和传统意义上的Char类型有些类似，对于一些字符有明确长度的场合，可以使用固定长度的字符串。定长字符串通过FixedString(N)声明，其中N表示字符串长度。但与Char不同的是，FixedString使用null字节填充末尾字符，而Char通常使用空格填充。

比如在下面的例子中，字符串‘abc’虽然只有3位，但长度却是5，因为末尾有2位空字符填充：
```
:) SELECT toFixedString('abc',5) , LENGTH(toFixedString('abc',5)) AS LENGTH
┌─toFixedString('abc', 5)─┬─LENGTH─┐
│ abc                      │ 5       │
└────────────────┴──────┘
```
- UUID
UUID是一种数据库常见的主键类型，在ClickHouse中直接把它作为一种数据类型。UUID共有32位，它的格式为8-4-4-4-12。如果一个UUID类型的字段在写入数据时没有被赋值，则会依照格式使用0填充，例如：
```
CREATE TABLE UUID_TEST (
    c1 UUID,
    c2 String
) ENGINE = Memory;
--第一行UUID有值
INSERT INTO UUID_TEST SELECT generateUUIDv4(),'t1'
--第二行UUID没有值
INSERT INTO UUID_TEST(c2) VALUES('t2')
 
:) SELECT * FROM UUID_TEST
┌─────────────────────c1─┬─c2─┐
│ f36c709e-1b73-4370-a703-f486bdd22749 │ t1 │
└───────────────────────┴────┘
┌─────────────────────c1─┬─c2─┐
│ 00000000-0000-0000-0000-000000000000 │ t2 │
└───────────────────────┴────┘
```
可以看到，第二行没有被赋值的UUID被0填充了。

### 时间类型
时间类型分为DateTime、DateTime64和Date三类。ClickHouse目前没有时间戳类型。时间类型最高的精度是秒，也就是说，如果需要处理毫秒、微秒等大于秒分辨率的时间，则只能借助UInt类型实现。

- DateTime
DateTime类型包含时、分、秒信息，精确到秒，支持使用字符串形式写入：
```
CREATE TABLE Datetime_TEST (
    c1 Datetime
) ENGINE = Memory
--以字符串形式写入
INSERT INTO Datetime_TEST VALUES('2019-06-22 00:00:00')
 
 SELECT c1, toTypeName(c1) FROM Datetime_TEST
┌──────────c1─┬─toTypeName(c1)─┐
│ 2019-06-22 00:00:00 │  DateTime        │
└─────────────┴───────────┘
```

- DateTime64
DateTime64可以记录亚秒，它在DateTime之上增加了精度的设置，例如：
```
CREATE TABLE Datetime64_TEST (
    c1 Datetime64(2)    
) ENGINE = Memory
--以字符串形式写入
INSERT INTO Datetime64_TEST VALUES('2019-06-22 00:00:00')
 
 SELECT c1, toTypeName(c1) FROM Datetime64_TEST
┌─────────────c1─┬─toTypeName(c1)─┐
│ 2019-06-22 00:00:00.00 │ DateTime       │
└───────────────┴──────────┘
```

- Date
Date类型不包含具体的时间信息，只精确到天，它同样也支持字符串形式写入：
```
CREATE TABLE Date_TEST (
    c1 Date
) ENGINE = Memory
 
--以字符串形式写入
INSERT INTO Date_TEST VALUES('2019-06-22')
SELECT c1, toTypeName(c1) FROM Date_TEST
┌─────────c1─┬─toTypeName(c1)─┐
│ 2019-06-22       │ Date            │
└───────────┴──────────┘
```

## 复合类型
除了基础数据类型之外，ClickHouse还提供了数组、元组、枚举和嵌套四类复合类型。这些类型通常是其他数据库原生不具备的特性。拥有了复合类型之后，ClickHouse的数据模型表达能力更强了。

- Array
数组有两种定义形式，常规方式array(T)：
```
SELECT array(1, 2) as a , toTypeName(a)
┌─a───┬─toTypeName(array(1, 2))─┐
│ [1,2] │ Array(UInt8)              │
└─────┴────────────────┘
```
或者简写方式[T]：
```
SELECT [1, 2]
```
通过上述的例子可以发现，在查询时并不需要主动声明数组的元素类型。因为ClickHouse的数组拥有类型推断的能力，推断依据：以最小存储代价为原则，即使用最小可表达的数据类型。例如在上面的例子中，array(1,2)会通过自动推断将UInt8作为数组类型。但是数组元素中如果存在Null值，则元素类型将变为Nullable，例如：
```
SELECT [1, 2, null] as a , toTypeName(a)
┌─a──────┬─toTypeName([1, 2, NULL])─┐
│ [1,2,NULL] │ Array(Nullable(UInt8))    │
└────────┴─────────────────┘
```
在同一个数组内可以包含多种数据类型，例如数组[1,2.0]是可行的。但各类型之间必须兼容，例如数组[1,'2']则会报错。

在定义表字段时，数组需要指定明确的元素类型，例如：
```
CREATE TABLE Array_TEST (
    c1 Array(String)
) engine = Memory
```

- Tuple
元组类型由1～n个元素组成，每个元素之间允许设置不同的数据类型，且彼此之间不要求兼容。元组同样支持类型推断，其推断依据仍然以最小存储代价为原则。与数组类似，元组也可以使用两种方式定义，常规方式tuple(T)：
```
SELECT tuple(1,'a',now()) AS x, toTypeName(x)
┌─x─────────────────┬─toTypeName(tuple(1, 'a', now()))─┐
│ (1,'a','2019-08-28 21:36:32') │ Tuple(UInt8, String, DateTime)    │
└───────────────────┴─────────────────────┘
```
或者简写方式（T）：
```
SELECT (1,2.0,null) AS x, toTypeName(x)
┌─x──────┬─toTypeName(tuple(1, 2., NULL))───────┐
│ (1,2,NULL) │ Tuple(UInt8, Float64, Nullable(Nothing)) │
└───────┴──────────────────────────┘
```
在定义表字段时，元组也需要指定明确的元素类型：
```
CREATE TABLE Tuple_TEST (
    c1 Tuple(String,Int8)
) ENGINE = Memory;
```
元素类型和泛型的作用类似，可以进一步保障数据质量。在数据写入的过程中会进行类型检查。例如，写入INSERT INTO Tuple_TEST VALUES(('abc',123))是可行的，而写入INSERT INTO Tuple_TEST VALUES(('abc','efg'))则会报错。

- Enum
ClickHouse支持枚举类型，这是一种在定义常量时经常会使用的数据类型。ClickHouse提供了Enum8和Enum16两种枚举类型，它们除了取值范围不同之外，别无二致。枚举固定使用(String:Int)Key/Value键值对的形式定义数据，所以Enum8和Enum16分别会对应(String:Int8)和(String:Int16)，例如：
```
CREATE TABLE Enum_TEST (
    c1 Enum8('ready' = 1, 'start' = 2, 'success' = 3, 'error' = 4)
) ENGINE = Memory;
```
在定义枚举集合的时候，有几点需要注意。首先，Key和Value是不允许重复的，要保证唯一性。其次，Key和Value的值都不能为Null，但Key允许是空字符串。在写入枚举数据的时候，只会用到Key字符串部分，例如：
```
INSERT INTO Enum_TEST VALUES('ready');
INSERT INTO Enum_TEST VALUES('start');
```
数据在写入的过程中，会对照枚举集合项的内容逐一检查。如果Key字符串不在集合范围内则会抛出异常，比如执行下面的语句就会出错：
```
INSERT INTO Enum_TEST VALUES('stop');
```
可能有人会觉得，完全可以使用String代替枚举，为什么还需要专门的枚举类型呢？这是出于性能的考虑。因为虽然枚举定义中的Key属于String类型，但是在后续对枚举的所有操作中（包括排序、分组、去重、过滤等），会使用Int类型的Value值。

- Nested
嵌套类型，顾名思义是一种嵌套表结构。一张数据表，可以定义任意多个嵌套类型字段，但每个字段的嵌套层级只支持一级，即嵌套表内不能继续使用嵌套类型。对于简单场景的层级关系或关联关系，使用嵌套类型也是一种不错的选择。例如，下面的nested_test是一张模拟的员工表，它的所属部门字段就使用了嵌套类型：
```
CREATE TABLE nested_test (
    name String,
    age  UInt8 ,
    dept Nested(
        id UInt8,
        name String
    )
) ENGINE = Memory;
```
ClickHouse的嵌套类型和传统的嵌套类型不相同，导致在初次接触它的时候会让人十分困惑。以上面这张表为例，如果按照它的字面意思来理解，会很容易理解成nested_test与dept是一对一的包含关系，其实这是错误的。不信可以执行下面的语句，看看会是什么结果：
```
INSERT INTO nested_test VALUES ('nauu',18, 10000, '研发部');
Exception on client:
Code: 53. DB::Exception: Type mismatch in IN or VALUES section. Expected: Array(UInt8). Got: UInt64
```
注意上面的异常信息，它提示期望写入的是一个Array数组类型。

现在大家应该明白了，嵌套类型本质是一种多维数组的结构。嵌套表中的每个字段都是一个数组，并且行与行之间数组的长度无须对齐。所以需要把刚才的INSERT语句调整成下面的形式：
```
INSERT INTO nested_test VALUES ('bruce' , 30 , [10000,10001,10002], ['研发部','技术支持中心','测试部']);
--行与行之间,数组长度无须对齐
INSERT INTO nested_test VALUES ('bruce' , 30 , [10000,10001], ['研发部','技术支持中心']);
```
需要注意的是，在同一行数据内每个数组字段的长度必须相等。例如，在下面的示例中，由于行内数组字段的长度没有对齐，所以会抛出异常：
```
INSERT INTO nested_test VALUES ('bruce' , 30 , [10000,10001], ['研发部','技术支持中心','测试部']); 
DB::Exception: Elements 'dept.id' and 'dept.name' of Nested data structure 'dept' (Array columns) have different array sizes..
```
在访问嵌套类型的数据时需要使用点符号，例如：
```
SELECT name, dept.id, dept.name FROM nested_test
┌─name─┬─dept.id──┬─dept.name─────────────┐
│ bruce │ [16,17,18] │ ['研发部','技术支持中心','测试部'] │
└────┴───────┴────────────────────┘
```

### 特殊类型
ClickHouse还有一类不同寻常的数据类型，我将它们定义为特殊类型。

- Nullable
准确来说，Nullable并不能算是一种独立的数据类型，它更像是一种辅助的修饰符，需要与基础数据类型一起搭配使用。Nullable类型与Java8的Optional对象有些相似，它表示某个基础数据类型可以是Null值。其具体用法如下所示：
```
CREATE TABLE Null_TEST (
    c1 String,
    c2 Nullable(UInt8)
) ENGINE = TinyLog;
通过Nullable修饰后c2字段可以被写入Null值：
INSERT INTO Null_TEST VALUES ('nauu',null)
INSERT INTO Null_TEST VALUES ('bruce',20)
SELECT c1 , c2 ,toTypeName(c2) FROM Null_TEST
┌─c1───┬───c2─┬─toTypeName(c2)─┐
│ nauu   │ NULL    │ Nullable(UInt8) │
│ bruce  │ 20      │ Nullable(UInt8) │
└─────┴──────┴───────────┘

```
在使用Nullable类型的时候还有两点值得注意：首先，它只能和基础类型搭配使用，不能用于数组和元组这些复合类型，也不能作为索引字段；其次，应该慎用Nullable类型，包括Nullable的数据表，不然会使查询和写入性能变慢。因为在正常情况下，每个列字段的数据会被存储在对应的[Column].bin文件中。如果一个列字段被Nullable类型修饰后，会额外生成一个[Column].null.bin文件专门保存它的Null值。这意味着在读取和写入数据时，需要一倍的额外文件操作。

- Domain
域名类型分为IPv4和IPv6两类，本质上它们是对整型和字符串的进一步封装。IPv4类型是基于UInt32封装的，它的具体用法如下所示：
```
CREATE TABLE IP4_TEST (
    url String,
    ip IPv4
) ENGINE = Memory;
INSERT INTO IP4_TEST VALUES ('www.nauu.com','192.0.0.0')
SELECT url , ip ,toTypeName(ip) FROM IP4_TEST
┌─url──────┬─────ip─┬─toTypeName(ip)─┐
│ www.nauu.com │ 192.0.0.0 │ IPv4             │
└────────┴───────┴──────────┘
```
细心的读者可能会问，直接使用字符串不就行了吗？为何多此一举呢？我想至少有如下两个原因。

出于便捷性的考量，例如IPv4类型支持格式检查，格式错误的IP数据是无法被写入的，例如：
```
INSERT INTO IP4_TEST VALUES ('www.nauu.com','192.0.0')
Code: 441. DB::Exception: Invalid IPv4 value.
```
出于性能的考量，同样以IPv4为例，IPv4使用UInt32存储，相比String更加紧凑，占用的空间更小，查询性能更快。IPv6类型是基于FixedString(16)封装的，它的使用方法与IPv4别无二致，此处不再赘述。

在使用Domain类型的时候还有一点需要注意，虽然它从表象上看起来与String一样，但Domain类型并不是字符串，所以它不支持隐式的自动类型转换。如果需要返回IP的字符串形式，则需要显式调用IPv4NumToString或IPv6NumToString函数进行转换。













