---
layout: post
categories: [JSON]
description: none
keywords: Jackson
---
# JSON实战Jackson
Jackson是一个`Java`语言编写的，可以进行`JSON`处理的开源工具库，`Jackson`的使用非常广泛，`Spring`框架默认使用`Jackson`进行 JSON 处理。

## 简介
Jackson 是当前用的比较广泛的，用来序列化和反序列化 json 的 Java 的开源框架。Jackson 社区相对比较活跃，更新速度也比较快， 从 Github 中的统计来看，Jackson 是最流行的 json 解析器之一 。

Spring MVC 的默认 json 解析器便是 Jackson。 Jackson 优点很多。 Jackson 所依赖的 jar 包较少 ，简单易用。与其他 Java 的 json 的框架 Gson 等相比， Jackson 解析大的 json 文件速度比较快；Jackson 运行时占用内存比较低，性能比较好；Jackson 有灵活的 API，可以很容易进行扩展和定制。

Jackson 的 1.x 版本的包名是 org.codehaus.jackson ，当升级到 2.x 版本时，包名变为 com.fasterxml.jackson。

## 核心模块
Jackson 的核心模块由三部分组成。

- jackson-core，核心包
提供基于"流模式"解析的相关 API，它包括 JsonPaser 和 JsonGenerator。 Jackson 内部实现正是通过高性能的流模式 API 的 JsonGenerator 和 JsonParser 来生成和解析 json。

- jackson-annotations，注解包
提供标准注解功能；

- jackson-databind ，数据绑定包
提供基于"对象绑定" 解析的相关 API （ ObjectMapper ） 和"树模型" 解析的相关 API （JsonNode）；基于"对象绑定" 解析的 API 和"树模型"解析的 API 依赖基于"流模式"解析的 API。

## JSON实战Jackson

### Maven 依赖
使用Maven构建项目，需要添加依赖：
```
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.13.3</version>
</dependency>
        
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-annotations</artifactId>
    <version>2.13.3</version>
</dependency>
        
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.13.3</version>
</dependency>
```
当然了，jackson-databind 依赖 jackson-core 和 jackson-annotations，所以可以只显示地添加jackson-databind依赖，jackson-core 和 jackson-annotations 也随之添加到 Java 项目工程中。

```xml
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
    <version>2.13.3</version>
</dependency>
```

为了方便后续的代码演示，我们同时引入 Junit 进行单元测试和 Lombok 以减少 Get/Set 的代码编写
```
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.8.2</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.22</version>
</dependency>
```

`Jackson`作为一个 Java 中的 JSON 工具库，处理 JSON 字符串和 Java 对象是它最基本最常用的功能，下面通过一些例子来演示其中的用法

## Jackson用法

### ObjectMapper
Jackson 最常用的 API 就是基于"对象绑定" 的 ObjectMapper：

- ObjectMapper可以从字符串，流或文件中解析JSON，并创建表示已解析的JSON的Java对象。 将JSON解析为Java对象也称为从JSON反序列化Java对象。
- ObjectMapper也可以从Java对象创建JSON。 从Java对象生成JSON也称为将Java对象序列化为JSON。
- Object映射器可以将JSON解析为自定义的类的对象，也可以解析置JSON树模型的对象。

之所以称为ObjectMapper是因为它将JSON映射到Java对象（反序列化），或者将Java对象映射到JSON（序列化）。

## 示例
编写一个 Person 类，定义三个属性，名称、年龄以及技能
```java
@Data
public class Person {
    // 名称
    private String name;
    // 年龄
    private Integer age;
    // 技能
    private List<String> skillList;
}
```

**将 Java 对象转换成 JSON 字符串**

```java
public class PersonTest {

    ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void pojoToJsonString() throws JsonProcessingException {
        Person person = new Person();
        person.setName("zhangsan");
        person.setAge(27);
        person.setSkillList(Arrays.asList("java", "c++"));

        String json = objectMapper.writeValueAsString(person);
        System.out.println(json);
        String expectedJson = "{\"name\":\"zhangsan\",\"age\":27,\"skillList\":[\"java\",\"c++\"]}";
        Assertions.assertEquals(json, expectedJson);
    }
}
```
输出的 JSON 字符串
```text
{"name":"zhangsan","age":27,"skillList":["java","c++"]}
```

**Jackson 甚至可以直接把序列化后的 JSON 字符串写入文件或者读取成字节数组**
```java
mapper.writeValue(new File("result.json"), myResultObject);
// 或者
byte[] jsonBytes = mapper.writeValueAsBytes(myResultObject);
// 或者
String jsonString = mapper.writeValueAsString(myResultObject);
```

**Jackson JSON 的反序列化**
```java
public class PersonTest {

    ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void jsonStringToPojo() throws JsonProcessingException {
        String expectedJson = "{\"name\":\"zhangsan\",\"age\":27,\"skillList\":[\"java\",\"c++\"]}";
        Person person = objectMapper.readValue(expectedJson, Person.class);
        System.out.println(person);
        Assertions.assertEquals(person.getName(), "zhangsan");
        Assertions.assertEquals(person.getSkillList().toString(), "[java, c++]");
    }
}
```
输出结果
```text
Person(name=zhangsan, age=27, skillList=[java, c++])
```
上面的例子演示了如何使用 Jackson 把一个 JSON 字符串反序列化成 Java 对象，其实 Jackson 对文件中的 JSON 字符串、字节形式的 JSON 字符串反序列化同样简单
比如先准备了一个 JSON 内容文件 Person.json
```json
{
  "name": "zhangsan",
  "age": 27,
  "skillList": [
      "java",
      "c++"
  ]
}
```
**下面进行读取转换**
```java
ObjectMapper objectMapper = new ObjectMapper();

@Test
void testJsonFilePojo() throws IOException {
    File file = new File("src/Person.json");
    Person person = objectMapper.readValue(file, Person.class);
    // 或者
    // person = mapper.readValue(new URL("http://some.com/api/entry.json"), MyValue.class);
    System.out.println(person);
    Assertions.assertEquals(person.getName(), "zhangsan");
    Assertions.assertEquals(person.getSkillList().toString(), "[java, c++]");
}
```
**同样输出了 Person 内容**
```text
Person(name=aLang, age=27, skillList=[java, c++])
```
**JSON 转 List**
上面演示 JSON 字符串都是单个对象的，如果 JSON 是一个对象列表那么使用 Jackson 该怎么处理呢？
已经存在一个文件 PersonList.json
```json
[
  {
  "name": "aLang",
  "age": 27, 
  "skillList": [
    "java",
    "c++"
  ]
  },
  {
  "name": "darcy",
  "age": 26,
  "skillList": [
  "go",
  "rust"
  ]
  }
]
```
**读取它然后转换成 List<Person>**
```java
ObjectMapper objectMapper = new ObjectMapper();

@Test
void fileToPojoList() throws IOException {
    File file = new File("src/EmployeeList.json");
    List<Person> personList = objectMapper.readValue(file, new TypeReference<List<Person>>() {});
    for (Person person : personList) {
        System.out.println(person);
    }
    Assertions.assertEquals(personList.size(), 2);
    Assertions.assertEquals(personList.get(0).getName(), "aLang");
    Assertions.assertEquals(personList.get(1).getName(), "darcy");
}
```
**可以输出对象内容**
```text
Person(name=aLang, age=27, skillList=[java, c++])
Person(name=darcy, age=26, skillList=[go, rust])
```
**JSON 转 Map**
JSON 转 Map 在我们没有一个对象的 Java 对象时十分实用，下面演示如何使用 Jackson 把 JSON 文本转成 Map 对象
```java
ObjectMapper objectMapper = new ObjectMapper();

@Test
void jsonStringToMap() throws IOException {
    String expectedJson = "{\"name\":\"aLang\",\"age\":27,\"skillList\":[\"java\",\"c++\"]}";
    Map<String, Object> employeeMap = objectMapper.readValue(expectedJson, new TypeReference<Map>() {});
    System.out.println(employeeMap.getClass());
    for (Entry<String, Object> entry : employeeMap.entrySet()) {
    System.out.println(entry.getKey() + ":" + entry.getValue());
    }
    Assertions.assertEquals(employeeMap.get("name"), "aLang");
}
```
可以看到 Map 的输出结果
```text
class java.util.LinkedHashMap
name:aLang
age:27
skillList:[java, c++]
```

## Jackson 的忽略字段
如果在进行 JSON 转 Java 对象时，JSON 中出现了 Java 类中不存在的属性，那么在转换时会遇到 com.fasterxml.jackson.databind.exc.UnrecognizedPropertyException 异常
使用 objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false) 可以忽略不存在的属性
```java
ObjectMapper objectMapper = new ObjectMapper();

@Test
void jsonStringToPojoIgnoreProperties() throws IOException {
// UnrecognizedPropertyException
String json = "{\"yyy\":\"xxx\",\"name\":\"aLang\",\"age\":27,\"skillList\":[\"java\",\"c++\"]}";
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
Person person = objectMapper.readValue(json, Person.class);
System.out.printf(person.toString());
Assertions.assertEquals(person.getName(), "aLang");
Assertions.assertEquals(person.getSkillList().toString(), "[java, c++]");
}
```
正常输出
```text
Person(name=aLang, age=27, skillList=[java, c++])
```

## Jackson 的日期格式化
在 Java 8 之前我们通常使用 java.util.Date 类来处理时间，但是在 Java 8 发布时引入了新的时间类 java.time.LocalDateTime. 这两者在 Jackson 中的处理略有不同
先创建一个有两种时间类型属性的 Order 类
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Order {

    private Integer id;

    private Date createTime;

    private LocalDateTime updateTime;
}
```
Date 类型 下面我们新建一个测试用例来测试两种时间类型的 JSON 转换
```java
class OrderTest {

    ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void testPojoToJson0() throws JsonProcessingException {
        Order order = new Order(1, new Date(), null);
        String json = objectMapper.writeValueAsString(order);
        System.out.println(json);

        order = objectMapper.readValue(json, Order.class);
        System.out.println(order.toString());

        Assertions.assertEquals(order.getId(), 1);
    }
}
```
在这个测试代码中，我们只初始化了 Date 类型的时间，下面是输出的结果
```text
{"id":1,"createTime":1658320852395,"updateTime":null}
Order(id=1, createTime=Wed Jul 20 20:40:52 CST 2022, updateTime=null)
```
可以看到正常的进行了 JSON 的序列化与反序列化，但是 JSON 中的时间是一个时间戳格式，可能不是我们想要的

## LocalDateTime 类型
为什么没有设置 LocalDateTime 类型的时间呢？因为默认情况下进行 LocalDateTime 类的 JSON 转换会遇到报错
```java
class OrderTest {

    ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void testPojoToJson() throws JsonProcessingException {
        Order order = new Order(1, new Date(), LocalDateTime.now());
        String json = objectMapper.writeValueAsString(order);
        System.out.println(json);

        order = objectMapper.readValue(json, Order.class);
        System.out.println(order.toString());

        Assertions.assertEquals(order.getId(), 1);
    }
}
```
运行后会遇到报错
```text
com.fasterxml.jackson.databind.exc.InvalidDefinitionException:
Java 8 date/time type `java.time.LocalDateTime` not supported by default:
add Module "com.fasterxml.jackson.datatype:jackson-datatype-jsr310"
to enable handling (through reference chain: com.wdbyte.jackson.Order["updateTime"])
```
这里我们需要添加相应的数据绑定支持包
```xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
    <version>2.13.3</version>
</dependency>
```
然后在定义 ObjectMapper 时通过 findAndRegisterModules() 方法来注册依赖
```java
class OrderTest {

    ObjectMapper objectMapper = new ObjectMapper().findAndRegisterModules();

    @Test
    void testPojoToJson() throws JsonProcessingException {
        Order order = new Order(1, new Date(), LocalDateTime.now());
        String json = objectMapper.writeValueAsString(order);
        System.out.println(json);

        order = objectMapper.readValue(json, Order.class);
        System.out.println(order.toString());

        Assertions.assertEquals(order.getId(), 1);
    }
}
```
运行可以得到正常序列化与反序列化日志，不过序列化后的时间格式依旧奇怪
```text
{"id":1,"createTime":1658321191562,"updateTime":[2022,7,20,20,46,31,567000000]}
Order(id=1, createTime=Wed Jul 20 20:46:31 CST 2022, updateTime=2022-07-20T20:46:31.567)
```

## 时间格式化
通过在字段上使用注解 @JsonFormat 来自定义时间格式
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Order {

    private Integer id;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Shanghai")
    private Date createTime;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Shanghai")
    private LocalDateTime updateTime;
}
```
再次运行上面的列子可以得到时间格式化后的 JSON 字符串
```text
{"id":1,"createTime":"2022-07-20 20:49:46","updateTime":"2022-07-20 20:49:46"}
Order(id=1, createTime=Wed Jul 20 20:49:46 CST 2022, updateTime=2022-07-20T20:49:46)
```

## Jackson 的常用注解
@JsonIgnore
使用 @JsonIgnore 可以忽略某个 Java 对象中的属性，它将不参与 JSON 的序列化与反序列化
```java
@Data
public class Cat {

    private String name;

    @JsonIgnore
    private Integer age;
}
```

编写单元测试类
```java
class CatTest {

    ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void testPojoToJson() throws JsonProcessingException {
        Cat cat = new Cat();
        cat.setName("Tom");
        cat.setAge(2);
        String json = objectMapper.writeValueAsString(cat);
        System.out.println(json);

        Assertions.assertEquals(json, "{\"name\":\"Tom\"}");

        cat = objectMapper.readValue(json, Cat.class);
        Assertions.assertEquals(cat.getName(), "Tom");
        Assertions.assertEquals(cat.getAge(), null);
    }
}
```
输出结果中 age 属性为 null
```text
{"name":"Tom"}
```

## @JsonGetter
使用 @JsonGetter 可以在对 Java 对象进行 JSON 序列化时自定义属性名称
```java
@Data
public class Cat {

    private String name;

    private Integer age;

    @JsonGetter(value = "catName")
    public String getName() {
        return name;
    }
}
```
编写单元测试类进行测试
```java
class CatTest {

    ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void testPojoToJson2() throws JsonProcessingException {
        Cat cat = new Cat();
        cat.setName("Tom");
        cat.setAge(2);
        String json = objectMapper.writeValueAsString(cat);
        System.out.println(json);
        Assertions.assertEquals(json, "{\"age\":2,\"catName\":\"Tom\"}");
    }
}
```
输出结果，name 已经设置成了 catName
```text
{"age":2,"catName":"Tom"}
```

## @JsonSetter
使用 @JsonSetter 可以在对 JSON 进行反序列化时设置 JSON 中的 key 与 Java 属性的映射关系
```java
@Data
public class Cat {

    @JsonSetter(value = "catName")
    private String name;

    private Integer age;

    @JsonGetter(value = "catName")
    public String getName() {
        return name;
    }
}
```
编写单元测试类进行测试
```java
class CatTest {

    ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void testPojoToJson2() throws JsonProcessingException {
        String json = "{\"age\":2,\"catName\":\"Tom\"}";
        Cat cat = objectMapper.readValue(json, Cat.class);
        System.out.println(cat.toString());
        Assertions.assertEquals(cat.getName(), "Tom");
    }
}
```
输出结果
```text
Cat(name=Tom, age=2)
```

## @JsonAnySetter
使用 @JsonAnySetter 可以在对 JSON 进行反序列化时，对所有在 Java 对象中不存在的属性进行逻辑处理，下面的代码演示把不存在的属性存放到一个 Map 集合中
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Student {

    private String name;
    private Integer age;
    private Map<String, Object> diyMap = new HashMap<>();

    @JsonAnySetter
    public void otherField(String key, String value) {
        this.diyMap.put(key, value);
    }
}
```
编写单元测试用例
```java
class StudentTest {

    private ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void testJsonToPojo() throws JsonProcessingException {
        Map<String, Object> map = new HashMap<>();
        map.put("name", "aLang");
        map.put("age", 18);
        map.put("skill", "java");

        String json = objectMapper.writeValueAsString(map);
        System.out.println(json);

        Student student = objectMapper.readValue(json, Student.class);
        System.out.println(student);

        Assertions.assertEquals(student.getDiyMap().get("skill"), "java");
    }
}
```
输出结果中可以看到 JSON 中的 skill 属性因为不在 Java 类 Student 中，所以被放到了 diyMap 集合
```text
{"skill":"java","name":"aLang","age":18}
Student(name=aLang, age=18, diyMap={skill=java})
```

## @JsonAnyGetter
使用 @JsonAnyGetter 可以在对 Java 对象进行序列化时，使其中的 Map 集合作为 JSON 中属性的来源
```java
@ToString
@AllArgsConstructor
@NoArgsConstructor
public class Student {

    @Getter
    @Setter
    private String name;

    @Getter
    @Setter
    private Integer age;
  
    @JsonAnyGetter
    private Map<String, Object> initMap = new HashMap() {{
        put("a", 111);
        put("b", 222);
        put("c", 333);
    }};
}
```
编写单元测试用例
```java
class StudentTest {

    private ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void testPojoToJsonTest() throws JsonProcessingException {
        Student student = new Student();
        student.setName("aLang");
        student.setAge(20);
        String json = objectMapper.writeValueAsString(student);
        System.out.println(json);
      
       Assertions.assertEquals(json,"{\"name\":\"aLang\",\"age\":20,\"a\":111,\"b\":222,\"c\":333}");
    }
}
```
输出结果
```text
{"name":"aLang","age":20,"a":111,"b":222,"c":333}
```

### 解决循环引用
jackson中的@JsonBackReference和@JsonManagedReference，以及@JsonIgnore均是为了解决对象中存在双向引用导致的无限递归（infinite recursion）问题。这些标注均可用在属性或对应的get、set方法中。

@JsonBackReference和@JsonManagedReference：这两个标注通常配对使用，通常用在父子关系中。@JsonBackReference标注的属性在序列化（serialization，即将对象转换为json数据）时，会被忽略（即结果中的json数据不包含该属性的内容）。@JsonManagedReference标注的属性则会被序列化。在序列化时，@JsonBackReference的作用相当于@JsonIgnore，此时可以没有@JsonManagedReference。但在反序列化（deserialization，即json数据转换为对象）时，如果没有@JsonManagedReference，则不会自动注入@JsonBackReference标注的属性（被忽略的父或子）；如果有@JsonManagedReference，则会自动注入自动注入@JsonBackReference标注的属性。

@JsonIgnore：直接忽略某个属性，以断开无限递归，序列化或反序列化均忽略。当然如果标注在get、set方法中，则可以分开控制，序列化对应的是get方法，反序列化对应的是set方法。在父子关系中，当反序列化时，@JsonIgnore不会自动注入被忽略的属性值（父或子），这是它跟@JsonBackReference和@JsonManagedReference最大的区别。

示例测试代码（注意反序列化后的TreeNode[readValue]的children里的parent）：
TreeNode.java
```java
import java.util.ArrayList;  
import java.util.List;  
  
import org.codehaus.jackson.annotate.JsonBackReference;  
import org.codehaus.jackson.annotate.JsonManagedReference;  
  
public class TreeNode {  
    String name;  
    @JsonBackReference  
//  @JsonIgnore  
    TreeNode parent;  
    @JsonManagedReference  
    List<TreeNode> children;  
  
    public TreeNode() {  
    }  
  
    public TreeNode(String name) {  
        this.name = name;  
    }  
  
    public String getName() {  
        return name;  
    }  
  
    public void setName(String name) {  
        this.name = name;  
    }  
  
    public TreeNode getParent() {  
        return parent;  
    }  
  
    public void setParent(TreeNode parent) {  
        this.parent = parent;  
    }  
  
    public List<TreeNode> getChildren() {  
        return children;  
    }  
  
    public void setChildren(List<TreeNode> children) {  
        this.children = children;  
    }  
  
    public void addChild(TreeNode child) {  
        if (children == null)  
            children = new ArrayList<TreeNode>();  
        children.add(child);  
    }  
}  

```

测试JsonTest.java
```
import java.io.IOException;  
  
import org.codehaus.jackson.JsonGenerationException;  
import org.codehaus.jackson.map.JsonMappingException;  
import org.codehaus.jackson.map.ObjectMapper;  
import org.junit.AfterClass;  
import org.junit.BeforeClass;  
import org.junit.Test;  
  
public class JsonTest {  
    static TreeNode node;  
  
    @BeforeClass  
    public static void setUp() {  
        TreeNode node1 = new TreeNode("node1");  
        TreeNode node2 = new TreeNode("node2");  
        TreeNode node3 = new TreeNode("node3");  
        TreeNode node4 = new TreeNode("node4");  
        TreeNode node5 = new TreeNode("node5");  
        TreeNode node6 = new TreeNode("node6");  
  
        node1.addChild(node2);  
        node2.setParent(node1);  
        node2.addChild(node3);  
        node3.setParent(node2);  
        node2.addChild(node4);  
        node4.setParent(node2);  
        node3.addChild(node5);  
        node5.setParent(node3);  
        node5.addChild(node6);  
        node6.setParent(node5);  
  
        node = node3;  
    }  
  
    @Test   
    public void test() throws JsonGenerationException, JsonMappingException, IOException {  
        ObjectMapper mapper = new ObjectMapper();  
        String json = mapper.writeValueAsString(node);  
        System.out.println(json);  
        TreeNode readValue = mapper.readValue(json, TreeNode.class);  
        System.out.println(readValue.getName());  
    }  
  
    @AfterClass  
    public static void tearDown() {  
        node = null;  
    }  
}
```


```
Jackson反序列化泛型List

ObjectMapper mapper = new ObjectMapper();
　　// 排除json字符串中实体类没有的字段
　　objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES,false);



String json = "[{\"name\":\"a\",\"password\":\"345\"},{\"name\":\"b\",\"password\":\"123\"}]";
        
//第一种方法
List<User> list = mapper.readValue(json, new TypeReference<List<User>>(){/**/});
        
//第二种方法
JavaType javaType = mapper.getTypeFactory().constructCollectionType(List.class, User.class);
List<User> list2 = mapper.readValue(json, javaType);
```

public final ObjectMapper mapper = new ObjectMapper();

    public static void main(String[] args) throws Exception{  
        JavaType javaType = getCollectionType(ArrayList.class, YourBean.class); 
        List<YourBean> lst =  (List<YourBean>)mapper.readValue(jsonString, javaType); 
    }   
       /**   
        * 获取泛型的Collection Type  
        * @param collectionClass 泛型的Collection   
        * @param elementClasses 元素类   
        * @return JavaType Java类型   
        * @since 1.0   
        */   
    public static JavaType getCollectionType(Class<?> collectionClass, Class<?>... elementClasses) {   
        return mapper.getTypeFactory().constructParametricType(collectionClass, elementClasses);   
    }