---
layout: post
categories: [JSON]
description: none
keywords: Jackson
---
# JSON源码Jackson

## Jackson 介绍
Jackson 有三个核心包，分别是 `Streaming`、`Databind`、`Annotations`，通过这些包可以方便的对 JSON 进行操作
- `Streaming` 在 `jackson-core` 模块。定义了一些流处理相关的 API 以及特定的 JSON 实现
- `Annotations` 在 `jackson-annotations` 模块，包含了 Jackson 中的注解
- `Databind` 在 `jackson-databind` 模块， 在 Streaming 包的基础上实现了数据绑定，依赖于 Streaming 和 Annotations 包
  得益于 Jackson 高扩展性的设计，有很多常见的文本格式以及工具都有对 Jackson 的相应适配，如 CSV、XML、YAML 等

### 源码
Maven依赖如下：
```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.13.3</version>
</dependency>
```
该依赖引入下面2个依赖
```
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-annotations</artifactId>
      <version>${jackson.version.annotations}</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-core</artifactId>
      <version>${jackson.version.core}</version>
    </dependency>
```
```
        ObjectMapper mapper = new ObjectMapper();
        Person person = new Person();
        person.setName("Tom");
        person.setAge(40);
        person.setNow(LocalDateTime.now());
 String jsonString = mapper.writeValueAsString(person);
        System.out.println(jsonString);
```

## ObjectMapper 对象映射器
`ObjectMapper`是 Jackson 库中最常用的一个类，使用它可以进行 Java 对象和 JSON 字符串之间快速转换。

这个类中有一些常用的方法
- readValue() 方法可以进行 JSON 的反序列化操作，比如可以将字符串、文件流、字节流、字节数组等将常见的内容转换成 Java 对象
- writeValue() 方法可以进行 JSON 的序列化操作，可以将 Java 对象转换成 JSON 字符串

大多数情况下，ObjectMapper 的工作原理是通过 Java Bean 对象的 Get/Set 方法进行转换时映射的，所以正确编写 Java 对象的 Get/Set 方法尤为重要，不过 ObjectMapper 也提供了诸多配置，比如可以通过配置或者注解的形式对 Java 对象和 JSON 字符串之间的转换过程进行自定义。

```
 public String writeValueAsString(Object value)
        throws JsonProcessingException
    {        
        // alas, we have to pull the recycler directly here...
//分段string 写入器，就是一个继承了了Writer的对象
        SegmentedStringWriter sw = new SegmentedStringWriter(_jsonFactory._getBufferRecycler());
        try {
// 关键这里，配置和写
            _configAndWriteValue(_jsonFactory.createGenerator(sw), value);
        } catch (JsonProcessingException e) { // to support [JACKSON-758]
            throw e;
        } catch (IOException e) { // shouldn't really happen, but is declared as possibility so:
            throw JsonMappingException.fromUnexpectedIOE(e);
        }
        return sw.getAndClear();
    }
```

```
protected JsonGenerator _createGenerator(Writer out, IOContext ctxt) throws IOException
    {
// JsonGenerator是WriterBasedJsonGenerator，而且传入了SegmentedStringWriter这个writer类
        WriterBasedJsonGenerator gen = new WriterBasedJsonGenerator(ctxt,
                _generatorFeatures, _objectCodec, out);
        if (_characterEscapes != null) {
// 配置转义符
            gen.setCharacterEscapes(_characterEscapes);
        }
        SerializableString rootSep = _rootValueSeparator;
        if (rootSep != DEFAULT_ROOT_VALUE_SEPARATOR) {
            gen.setRootValueSeparator(rootSep);
        }
        return gen;
    }
```
这是ObjectMapper的方法

```
protected final void _configAndWriteValue(JsonGenerator g, Object value)
        throws IOException
    {
        SerializationConfig cfg = getSerializationConfig();
    //  把ObjectMapper的配置，配置到JsonGenerator中
       cfg.initialize(g); // since 2.5
。。。
        boolean closed = false;
        try {
// 创建和配置好serializerProvider,这里生成的是DefaultSerializerProvider的内部类，也是子类Impl，然后调用serializerProvider的serialize方法，传入JsonGenerator和序列化对象value
// 这里会真正序列化对象
            _serializerProvider(cfg).serializeValue(g, value);
...
        } finally {
        ...
    }
```

DefaultSerializerProvider的serializeValue方法
```
  public void serializeValue(JsonGenerator gen, Object value) throws IOException
    {
        if (value == null) {
            _serializeNull(gen);
            return;
        }
        Class<?> cls = value.getClass();
        // true, since we do want to cache root-level typed serializers (ditto for null property)
// 找到匹配这个类型的json序列化器
        final JsonSerializer<Object> ser = findTypedValueSerializer(cls, true, null);
```

找到匹配这个类型的json序列化器
```
public JsonSerializer<Object> findTypedValueSerializer(Class<?> valueType,
            boolean cache, BeanProperty property)
        throws JsonMappingException
    {
        // 两阶段查找，先class 类hash映射，去_knownSerializers预先定义好的序列化集合中找寻是否有这个类匹配的序列化器
        // Two-phase lookups; local non-shared cache, then shared:
        JsonSerializer<Object> ser = _knownSerializers.typedValueSerializer(valueType);
        if (ser != null) {
            return ser;
        }
      // 没有，则去_serializerCache缓存中找
        // If not, maybe shared map already has it?
        ser = _serializerCache.typedValueSerializer(valueType);
        if (ser != null) {
            return ser;
        }
      // 还没有，则取组成一个
        // Well, let's just compose from pieces:
        ser = findValueSerializer(valueType, property);
...
        return ser;
    }
```

构成序列化器
```
public JsonSerializer<Object> findValueSerializer(Class<?> valueType, BeanProperty property)
        throws JsonMappingException
    {
      。。。这里只是在从缓存等查找了一遍
// 如果没有，则创建一个，而且放到缓存中 
                    // If neither, must create
                    ser = _createAndCacheUntypedSerializer(valueType);
                    // Not found? Must use the unknown type serializer, which will report error later on
                    if (ser == null) {
                        ser = getUnknownTypeSerializer(valueType);
                        // Should this be added to lookups?
                        if (CACHE_UNKNOWN_MAPPINGS) {
                            _serializerCache.addAndResolveNonTypedSerializer(valueType, ser, this);
                        }
                        return ser;
                    }
                }
            }
        }
        // at this point, resolution has occured, but not contextualization
        return (JsonSerializer<Object>) handleSecondaryContextualization(ser, property);
    }
```

```
protected JsonSerializer<Object> _createUntypedSerializer(JavaType type)
        throws JsonMappingException
    {
    
        synchronized (_serializerCache) {
         // 通过序列化工厂类，这个序列化工厂类是BeanSerializerFactory构建序列化器
            return (JsonSerializer<Object>)_serializerFactory.createSerializer(this, type);
        }
    }
```

```
public JsonSerializer<Object> createSerializer(SerializerProvider prov,
            JavaType origType)
        throws JsonMappingException
    {
        // Very first thing, let's check if there is explicit serializer annotation:
        final SerializationConfig config = prov.getConfig();
        BeanDescription beanDesc = config.introspect(origType);
// 先看有没有注解，注明了序列化器，自然是没有
        JsonSerializer<?> ser = findSerializerFromAnnotation(prov, beanDesc.getClassInfo());
        if (ser != null) {
            return (JsonSerializer<Object>) ser;
        }
        boolean staticTyping;
// 是否有注解，著名修改了这个是什么类，自然没有
        // Next: we may have annotations that further define types to use...
        JavaType type = modifyTypeByAnnotation(config, beanDesc.getClassInfo(), origType);
        if (type == origType) { // no changes, won't force static typing
            staticTyping = false;
        } else { // changes; assume static typing; plus, need to re-introspect if class differs
            staticTyping = true;
            if (!type.hasRawClass(origType.getRawClass())) {
                beanDesc = config.introspect(type);
            }
        }
      //找转换器，这里找不到   
     // Slight detour: do we have a Converter to consider?
        Converter<Object,Object> conv = beanDesc.findSerializationConverter();
        if (conv == null) { // no, simple
          // 真正构成序列化器的是这里
            return (JsonSerializer<Object>) _createSerializer2(prov, type, beanDesc, staticTyping);
        }
```

BeanSerializerFactory的_createSerializer2方法，构建序列化器
```
。。。这里一系列的各种找
 // Modules may provide serializers of POJO types:
// BeanSerializerFactory里自定义的序列器中找
            for (Serializers serializers : customSerializers()) {
                ser = serializers.findSerializer(config, type, beanDesc);
                if (ser != null) {
                    break;
                }
            }
。。。
最后找不到，就会构建一个BeanSerializer, 这个BeanSerializer里面包含了我传入的bean的属性,然后针对属性，又会构造BeanPropertyWriter ，这个BeanPropertyWriter 包含了针对属性的序列化器等，是一个只针对传入bena的一个很自定义的东西
                    ser = findBeanSerializer(prov, type, beanDesc);
```

```
// 构建了BeanSerializer之后，就是调用它的serialize序列化方法了
ser.serialize(value, gen, this);
```

BeanSerializer的serialize方法
```
public final void serialize(Object bean, JsonGenerator gen, SerializerProvider provider)
        throws IOException
    {
        if (_objectIdWriter != null) {
            gen.setCurrentValue(bean); // [databind#631]
            _serializeWithObjectId(bean, gen, provider, true);
            return;
        }
// 写入"{"这个json开始字符，里面是char[]数组存储字符的
        gen.writeStartObject();
        // [databind#631]: Assign current value, to be accessible by custom serializers
        gen.setCurrentValue(bean);
        if (_propertyFilterId != null) {
            serializeFieldsFiltered(bean, gen, provider);
        } else {
// 然后序列化各个属性
            serializeFields(bean, gen, provider);
        }
        gen.writeEndObject();
    }
```

BeanSerializer父类BeanSerializerBase的方法
```
protected void serializeFields(Object bean, JsonGenerator gen, SerializerProvider provider)
        throws IOException, JsonGenerationException
    {
       ...
        try {
//遍历各个属性，然后分别序列化属性，如果属性是常用类型，自然就序列化器会直接序列化，如果是pojo，则属性序列化器是BeanSerializer，又会递归序列化
            for (final int len = props.length; i < len; ++i) {
                BeanPropertyWriter prop = props[i];
                if (prop != null) { // can have nulls in filtered list
                    prop.serializeAsField(bean, gen, provider);
                }
            }
...
    }
```
到此：大概的整个序列化流程就是这样，当然，里面有很多细节，可以遇到具体问题具体来细节分析某一个模块。








