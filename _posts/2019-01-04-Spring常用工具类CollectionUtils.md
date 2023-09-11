---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring常用工具类CollectionUtils


## CollectionUtils
CollectionUtils是Spring框架org.springframework.util 工具包中提供的一个工具类，可以对Java集合进行非空判断（2个方法）、转换（5个方法）、合并（2个方法）、查找（11个方法）等功能。

在使用CollectionUtils工具类的时候，需要引入该工具类。
```
import org.springframework.util.CollectionUtils;  
```

接下来，我们就该工具类的静态方法的使用逐一进行举例说明。

## 非空判断

```
boolean isEmpty(@Nullable Collection<?> collection)：判断集合是否为空。
```

假设我们有一个List对象，名为list，我们可以使用CollectionUtils中的isEmpty方法来判断list是否为空。

示例代码如下：
```
List<String> list = new ArrayList<>();  
boolean empty = CollectionUtils.isEmpty(list);  
System.out.println(empty);  
```
在这个示例中，我们创建了一个空的List对象，并使用CollectionUtils中的isEmpty方法来判断该集合是否为空。由于该集合确实是空的，因此isEmpty方法返回true。

```
boolean isEmpty(@Nullable Map map)方法则可以传入一个为map类型的集合。
```

## 转换
```
List arrayToList(@Nullable Object source) ：将数组转换为List对象。
```

假设我们有一个String类型的数组，名为array。我们可以使用CollectionUtils中的arrayToList方法将该数组转换为List对象。

示例代码如下：
```
String[] array = {"Java", "Python", "C++"};  
List<String> list = CollectionUtils.arrayToList(array);  
System.out.println(list);  
```
在这个示例中，我们创建了一个包含三个元素的String数组，并使用CollectionUtils中的arrayToList方法将该数组转换为List对象。最终，list集合与原始的数组内容相同。

```
toArray(Enumeration enumeration, A[] array)：将一个Enumeration对象转换成数组。
```
假如我们有一个Enumeration对象，名为enumeration，其中包含一些元素。我们可以使用CollectionUtils中的toArray方法将enumeration转换成String数组。

示例代码如下：
```
Enumeration<Object> enumeration = new StringTokenizer("Java,Python,Javascript", ",");  
String[] array = (String[]) CollectionUtils.toArray(enumeration, new String[0]);  
for (String s : array) {  
    System.out.println(s);  
}  
```
在这个示例中，我们创建了一个包含三个元素的Enumeration对象，并使用CollectionUtils中的toArray方法将其转换成String数组。我们传递了两个参数给toArray方法：第一个参数是要转换的Enumeration对象，第二个参数是用于存储数组元素的空数组。在本例中，我们传递了一个长度为0的String数组作为空数组。

toArray方法返回一个与指定类型和大小相同的新数组，该数组包含指定枚举中的所有元素。我们使用for- each循环遍历返回的String数组，并依次输出数组中的每个元素。

```
toIterator(@Nullable Enumeration enumeration)：将一个Enumeration对象转换成迭代器。
```
假如我们有一个Enumeration对象，名为enumeration，其中包含一些元素。我们可以使用CollectionUtils中的toIterator方法将list转换成迭代器。

示例代码如下：
```
Enumeration<Object> enumeration = new StringTokenizer("Java,Python,Javascript", ",");  
Iterator<Object> iterator = CollectionUtils.toIterator(enumeration);  
while (iterator.hasNext()) {  
    System.out.println(iterator.next());  
}  
```
注意，如果传递给toIterator方法的集合为空，则toIterator方法返回一个空的迭代器，而不是null。

```
toMultiValueMap(Map<K, List> map)：将一个键对应多个值的Map对象转换成MultiValueMap对象。
```
假设我们有一个Map<String, List>对象，名为map，其中的每个键都对应着多个值。我们可以使用CollectionUtils中的toMultiValueMap方法将map转换成MultiValueMap对象。

示例代码如下：
```
Map<String, List<String>> map = new HashMap<>();  
List<String> list1 = new ArrayList<>();  
list1.add("Java");  
list1.add("Python");  
map.put("languages", list1);  
  
List<String> list2 = new ArrayList<>();  
list2.add("John");  
list2.add("Peter");  
list2.add("Alice");  
map.put("names", list2);  
  
MultiValueMap<String, String> multiValueMap = CollectionUtils.toMultiValueMap(map);  
System.out.println(multiValueMap);  

//  {names=[John, Peter, Alice], languages=[Java, Python]}  
```
在这个示例中，我们创建了一个Map<String, List>对象，并使用CollectionUtils中的toMultiValueMap方法将该map转换成MultiValueMap对象。toMultiValueMap方法返回一个MultiValueMap对象，该对象包含指定map中的所有键值对，其中每个键值都对应着一个值列表。我们使用println方法输出返回的MultiValueMap对象。

注意，如果传递给toMultiValueMap方法的map对象为空，则toMultiValueMap方法返回一个空的MultiValueMap对象，而不是null。

```
unmodifiableMultiValueMap(MultiValueMap<? extends K, ? extends V> map)：将一个MultiValueMap对象包装为不可修改的MultiValueMap对象。
```

假设我们有一个MultiValueMap<String, String>对象，名为map，其中包含一些键值对。我们可以使用CollectionUtils中的unmodifiableMultiValueMap方法将该map对象包装为不可修改的MultiValueMap对象。

示例代码如下：
```
Map<String, List<String>> map = new HashMap<>();  
List<String> list1 = new ArrayList<>();  
list1.add("Java");  
list1.add("Python");  
map.put("languages", list1);  
  
List<String> list2 = new ArrayList<>();  
list2.add("John");  
list2.add("Peter");  
list2.add("Alice");  
map.put("names", list2);  
  
MultiValueMap<String, String> multiValueMap = CollectionUtils.toMultiValueMap(map);  
System.out.println(multiValueMap);   
  
  
MultiValueMap<String, String> unmodifiableMap = CollectionUtils.unmodifiableMultiValueMap(multiValueMap);  
System.out.println(unmodifiableMap.get("languages"));  
unmodifiableMap.put("colors", new ArrayList<>());  
```
在这个示例中，我们创建了一个MultiValueMap<String, String>对象，并使用CollectionUtils中的unmodifiableMultiValueMap方法将该map对象包装为不可修改的MultiValueMap对象。unmodifiableMultiValueMap方法返回一个不可修改的MultiValueMap对象，该对象与指定的MultiValueMap对象具有相同的键值对。我们可以使用get方法获取某个键对应的值列表，但是无法使用put或remove等方法修改该不可修改的MultiValueMap对象。如果尝试调用这些方法，则会抛出UnsupportedOperationException异常。

注意，如果传递给unmodifiableMultiValueMap方法的map对象为空，则unmodifiableMultiValueMap方法返回一个空的MultiValueMap对象，而不是null。

## 合并
```
mergeArrayIntoCollection(@Nullable Object array, Collection collection)：将数组合并到Collection对象中。
```

假设我们有一个String类型的数组，名为array。我们还创建了一个List对象，名为collection。我们可以使用CollectionUtils中的mergeArrayIntoCollection方法将array中的元素合并到collection中。

示例代码如下：
```
String[] array = {"Java", "Python", "C++"};  
List<String> collection = new ArrayList<>();  
CollectionUtils.mergeArrayIntoCollection(array, collection);  
System.out.println(collection);  
```
在这个示例中，我们创建了一个包含三个元素的String数组，并创建了一个空的List对象。我们使用CollectionUtils中的mergeArrayIntoCollection方法，将array中的元素合并到collection中。最终，collection集合与原始的数组内容相同。

```
mergePropertiesIntoMap(@Nullable Properties props, Map<K, V> map)：将Properties对象合并为Map对象。
```
假设我们有一个Properties对象，名为props。我们还创建了一个空的Map<String, String>对象，名为map。我们可以使用CollectionUtils中的mergePropertiesIntoMap方法将props合并到map中。

示例代码如下：
```
Properties props = new Properties();  
props.setProperty("key1", "value1");  
props.setProperty("key2", "value2");  
Map<String, String> map = new HashMap<>();  
CollectionUtils.mergePropertiesIntoMap(props, map);  
System.out.println(map);  
```
在这个示例中，我们创建了一个包含两个键值对的Properties对象，并创建了一个空的HashMap<String, String>对象。我们使用CollectionUtils中的mergePropertiesIntoMap方法，将props中的键值对合并到map中。最终，map中包含了与props相同的两个键值对。

## 查找
```
contains(@Nullable Iterator<?> iterator, Object element)：判断集合中是否包含指定元素。
```
假设我们有一个List对象，名为list，其中包含一些元素。我们可以使用CollectionUtils中的contains方法来判断list中是否包含"Java"这个元素。

示例代码如下：
```
List<String> list = new ArrayList<>();  
list.add("Java");  
list.add("Python");  
boolean contains = CollectionUtils.contains(list.iterator(), "Java");  
System.out.println(contains);  
```
在这个示例中，我们创建了一个包含两个元素的List对象，并使用CollectionUtils中的contains方法来判断该集合中是否包含"Java"这个元素。由于list中确实包含"Java"这个元素，因此contains方法返回true。

contains(@Nullable Enumeration<?> enumeration, Object element)，这个方法就不举例了。

```
containsAny(Collection source, Collection candidates)：判断集合source中是否包含candidates中的任意一个元素。
```
假设我们有两个List对象，分别为source和candidates，我们可以使用CollectionUtils中的containsAny方法来判断source中是否包含candidates中的任意一个元素。

示例代码如下：
```
List<String> source = new ArrayList<>();  
source.add("Java");  
source.add("Python");  
List<String> candidates = new ArrayList<>();  
candidates.add("C++");  
candidates.add("Java");  
boolean containsAny = CollectionUtils.containsAny(source, candidates);  
System.out.println(containsAny); // 输出true  
```
在这个示例中，我们创建了两个List对象，分别为source和candidates，并使用CollectionUtils中的containsAny方法来判断source中是否包含candidates中的任意一个元素。由于source中包含"Java"这个元素，因此containsAny方法返回true。

```
containsAny(Collection source, Collection candidates)：判断集合中是否包含指定集合中任意一个元素。
```
假设我们有一个List对象，名为list，其中包含一些元素。我们还有一个包含一些字符串的List对象，名为subList。我们可以使用CollectionUtils中的containsAny方法来判断list中是否包含subList中的任意一个元素。

示例代码如下：
```
List<String> list = new ArrayList<>();  
list.add("Java");  
list.add("Python");  
list.add("C++");  
List<String> subList = new ArrayList<>();  
subList.add("Python");  
subList.add("Ruby");  
boolean containsAny = CollectionUtils.containsAny(list, subList);  
System.out.println(containsAny); // 输出true  
```
在这个示例中，我们创建了一个包含三个元素的List对象，并创建了另一个包含两个元素的List对象。我们使用CollectionUtils中的containsAny方法来判断list中是否包含subList中的任意一个元素。由于"Python"这个元素同时出现在list和subList中，因此containsAny方法返回true。

```
containsInstance(@Nullable Collection<?> collection, Object element)：判断集合中是否包含指定对象的引用。
```
假设我们有一个List对象，名为list，其中包含一些元素和一个String类型的引用，我们可以使用CollectionUtils中的containsInstance方法来判断list中是否包含该引用。

示例代码如下：
```
String str = "Java";  
List<Object> list = new ArrayList<>();  
list.add(str);  
list.add("Python");  
boolean containsInstance = CollectionUtils.containsInstance(list, str);  
System.out.println(containsInstance);  
```
在这个示例中，我们先定义了一个String类型的引用str，并将其添加到一个包含两个元素的List对象中。然后，我们使用CollectionUtils中的containsInstance方法来判断list中是否包含引用str。由于list确实包含引用str，因此containsInstance方法返回true。

```
findFirstMatch(Collection<?> source, Collection candidates)：查找集合中第一个匹配的元素。
```
假设我们有一个List对象，名为list，其中包含一些元素。还有一个被查询List对象，名为subList 。我们可以使用CollectionUtils中的findFirstMatch方法来查找list中subList 中的元素。

示例代码如下：
```
List<String> list = new ArrayList<>();  
list.add("Java");  
list.add("Python");  
list.add("Javascript");  
List<String> subList = new ArrayList<>();  
subList.add("Python");  
subList.add("Ruby");  
String match = CollectionUtils.findFirstMatch(list, subList);  
System.out.println(match);  
```

在这个示例中，我们创建了一个包含三个元素的List对象，并创建了另一个包含两个元素的List对象。我们使用CollectionUtils中的findFirstMatch方法来判断list中是否包含subList中的任意一个元素。由于"Python"这个元素同时出现在list和subList中，因此findFirstMatch方法返回Python。

```
findValueOfType(Collection<?> collection, @Nullable Class type)：查找集合中指定类型的元素。
```
假设我们有一个List对象，名为list，其中包含一些元素，可能是不同类型的。我们可以使用CollectionUtils中的findValueOfType方法来查找list中第一个String类型的元素。

示例代码如下：
```
List<Object> list = new ArrayList<>();  
list.add("Java");  
list.add(123);  
list.add("Python");  
  
String valueOfType = CollectionUtils.findValueOfType(list, String.class);  
System.out.println(valueOfType);  
```
在这个示例中，我们创建了一个包含三个元素的List对象，并使用CollectionUtils中的findValueOfType方法来查找第一个String类型的元素。由于有多个String类型，因此findValueOfType方法返回null。如果list中没有任何String类型的元素，则返回null。注意，如果集合中存在一个String类型的元素，则只返回第一个匹配的元素。
findValueOfType(Collection collection, Class[] types)：与该方法的唯一区别就是需要查找的元素为数组。

该方法在Spring Framework 5.3已废弃，不建议再使用。

```
hasUniqueObject(Collection<?> collection)：判断集合中是否只包含一个元素。
```
假设我们有一个List对象，名为list，其中包含一些元素。我们可以使用CollectionUtils中的hasUniqueObject方法来判断list中是否只包含一个元素。

示例代码如下：
```
List<String> list = new ArrayList<>();  
list.add("Java");  
boolean hasUnique = CollectionUtils.hasUniqueObject(list);  
System.out.println(hasUnique);  
```
在这个示例中，我们创建了一个包含一个元素的List对象，并使用CollectionUtils中的hasUniqueObject方法来判断该集合中是否只包含一个元素。由于list只包含一个元素，因此hasUniqueObject方法返回true。如果list中包含多个元素，则hasUniqueObject方法返回false。注意，如果list为空，则hasUniqueObject方法也返回false。

```
findCommonElementType(Collection<?> collection) ：查找集合中所有元素的公共类型。
```
假设我们有一个List对象，名为list，其中包含一些元素，可能是不同类型的。我们可以使用CollectionUtils中的findCommonElementType方法来查找所有元素的公共类型。

示例代码如下：
```
List<Object> list = new ArrayList<>();  
list.add("Java");  
list.add(123);  
Class<?> commonElementType = CollectionUtils.findCommonElementType(list);  
System.out.println(commonElementType);  
```
在这个示例中，我们创建了一个包含两个元素的List对象，并使用CollectionUtils中的findCommonElementType方法来查找该集合中所有元素的公共类型。由于list中的元素类型不同，因此findCommonElementType方法返回null。如果list中的所有元素类型相同，则findCommonElementType方法返回该元素类型的Class对象。如果list为空，则findCommonElementType方法返回null。

```
T lastElement(@Nullable Set set)：获取集合中的最后一个元素。
```
假设我们有一个List对象，名为list，其中包含一些元素。我们可以使用CollectionUtils中的lastElement方法来获取list中的最后一个元素。

示例代码如下：
```
List<String> list = new ArrayList<>();  
list.add("Java");  
list.add("Python");  
list.add("Javascript");  
String lastElement = CollectionUtils.lastElement(list);  
System.out.println(lastElement);  
```
在这个示例中，我们创建了一个包含三个元素的List对象，并使用CollectionUtils中的lastElement方法来获取该集合中的最后一个元素。由于"Javascript"是list中的最后一个元素，因此lastElement方法返回该字符串对象的引用。如果list为空，则lastElement方法返回null。

当我把这个工具类里的方法全都过了一遍才明白，其他朋友为什么不用这个工具类了，能够常用的方法真是太少了。



