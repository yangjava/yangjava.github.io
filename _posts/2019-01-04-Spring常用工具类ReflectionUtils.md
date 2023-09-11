---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring常用工具类ReflectionUtils
ReflectionUtils是Spring框架中的反射工具类，它提供了一系列静态方法，可以方便地进行类、对象、方法、字段等反射操作。

## ReflectionUtils

### shallowCopyFieldState
```
shallowCopyFieldState(final Object src, final Object dest) 将源对象的属性值浅拷贝到目标对象中。
```

该方法会遍历源对象的所有可读属性，并将其值拷贝到目标对象的对应属性中。拷贝的方式是通过直接赋值来实现的，因此是浅拷贝，即拷贝的是属性的引用而不是属性的副本。

````
Person source = new Person();
source.setName("John");  
source.setAge(25);  
PersonVO target = new PersonVO();  
// 浅拷贝  
ReflectionUtils.shallowCopyFieldState(source, target);          
System.out.println("Name: " + target.getName());  
System.out.println("Age: " + target.getAge());  
````

### handleReflectionException
```
handleReflectionException(Exception ex)用于处理反射操作中可能抛出的异常。
```

它接受一个Exception类型的参数ex，表示要处理的异常。在示例中，我们调用了一个不存在的方法，抛异常了。
```
Person example = new Person();
try {  
    Method method = Person.class.getMethod("nonExistingMethod");  
    ReflectionUtils.invokeMethod(method, example);  
} catch (Exception ex) {  
    // 处理反射操作中的异常  
    ReflectionUtils.handleReflectionException(ex);  
}  
```

### handleInvocationTargetException
```
handleInvocationTargetException(InvocationTargetException ex)
```
处理给定的调用目标异常，InvocationTargetException是Java反射中的一个异常类，它表示在调用方法时发生了异常。

通常情况下，我们可以使用getCause()方法获取原始的异常对象。

### rethrowRuntimeException
```
rethrowRuntimeException(Throwable ex)
```
重新抛出一个运行时异常。

### rethrowException
```
rethrowException(Throwable ex)
```
重新抛出一个异常，重新抛出的异常分为三种类型：

运行时异常RuntimeException、Error、UndeclaredThrowableException。

### accessibleConstructor
```
ReflectionUtils.accessibleConstructor(Class clazz, Class<?>... parameterTypes)
```
用于获取指定类的可访问构造函数。

```
 try {  
    Constructor<Person> constructor = ReflectionUtils.accessibleConstructor(Person.class, String.class, int.class);  
    Person example = constructor.newInstance("John Doe", 25);  
    System.out.println("Name: " + example.getName());  
    System.out.println("Age: " + example.getAge());  
} catch (Exception ex) {  
    ex.printStackTrace();  
}  
```

### findMethod
```
Method findMethod(Class<?> clazz, String name)
```
获取指定类的指定方法。
```
Method getNameMethod = ReflectionUtils.findMethod(Person.class, "getName");  
System.out.println("getName method: " + getNameMethod);  
```
注意：该方法只能获取没有参数的方法。


```
findMethod(Class clazz, String name, @Nullable Class... paramTypes)
```
获取指定类的指定方法，该方法还有一个或多个参数类型。

```
Method setNameMethod = ReflectionUtils.findMethod(Person.class, "setName", String.class);  
System.out.println("setName method: " + setNameMethod);  
```

### invokeMethod
```
invokeMethod(Method method, @Nullable Object target)
```
用于调用指定方法，并传入目标对象（可选）。

```
Method setNameMethod = ReflectionUtils.findMethod(Person.class, "setName", String.class);  
System.out.println("setName method: " + setNameMethod);          
  
Person example = new Person();  
// 设置访问权限才可访问  
setNameMethod.setAccessible(true);  
ReflectionUtils.invokeMethod(setNameMethod, example, "eeefdfdfd");            
System.out.println(example.getName());  
```

### declaresException
```
declaresException(Method method, Class<?> exceptionType)
```
用于检查指定方法是否声明了指定的异常类型。
```
Method method = ReflectionUtils.findMethod(Person.class, "getName");  
boolean declaresIOException = ReflectionUtils.declaresException(method, IOException.class);  
System.out.println("Declares IOException: " + declaresIOException);  
  
boolean declaresNullPointerException = ReflectionUtils.declaresException(method, NullPointerException.class);  
System.out.println("Declares NullPointerException: " + declaresNullPointerException);  
```

### doWithLocalMethods
```
doWithLocalMethods(Class<?> clazz, MethodCallback mc)
```
用于迭代指定类的所有本地方法，并对每个方法执行回调操作。
```
ReflectionUtils.doWithLocalMethods(Person.class, new PersonCallback());  
```

回调类：
```
// 需要实现MethodCallback 方法  
class PersonCallback implements MethodCallback {  
    @Override  
    public void doWith(Method method) throws IllegalArgumentException, IllegalAccessException {  
        System.out.println("Method name: " + method.getName());  
    }  
}  
```

### doWithMethods
```
doWithMethods(Class<?> clazz, MethodCallback mc)
```
用于迭代指定类的所有方法，并对每个方法执行回调操作。

```
ReflectionUtils.doWithMethods(Person.class, new PersonCallback());  
```

```
doWithMethods(Class<?> clazz, MethodCallback mc, @Nullable MethodFilter mf)
```
用于迭代指定类的所有方法，并对每个方法执行回调操作。

```
ReflectionUtils.doWithMethods(Person.class, new PersonCallback(), new PersonMethodFilter());  
  
// 需要实现MethodFilter接口  
class PersonMethodFilter implements MethodFilter {  
  
    @Override  
    public boolean matches(Method method) {  
        return Modifier.isPublic(method.getModifiers());  
    }      
}  
```

### getAllDeclaredMethods
```
getAllDeclaredMethods(Class<?> leafClass)
```
用于获取指定类及其所有父类中声明的所有方法。
```
Class<?> leafClass = PersonVO.class;  
Method[] methods = ReflectionUtils.getAllDeclaredMethods(leafClass);  
for (Method method : methods) {  
   System.out.println("Method name: " + method.getName());  
}  
```

### getUniqueDeclaredMethods
```
getUniqueDeclaredMethods(Class<?> leafClass)
```
用于获取指定类及其所有父类中声明的所有唯一方法。
```
Class<?> leafClass = PersonVO.class;  
Method[] methods = ReflectionUtils.getUniqueDeclaredMethods(leafClass);  
for (Method method : methods) {  
    System.out.println("Method name: " + method.getName());  
}  
```

### isEqualsMethod
```
isEqualsMethod(@Nullable Method method)
```
用于判断给定的方法是否是equals方法。

```
 try {  
    Class<?> clazz = PersonVO.class;  
    Method method = clazz.getMethod("equals", Object.class);  
    boolean isEquals = ReflectionUtils.isEqualsMethod(method);  
    System.out.println("Is equals method: " + isEquals);  
} catch (NoSuchMethodException | SecurityException e) {  
    e.printStackTrace();  
}  
```

### isHashCodeMethod
```
isHashCodeMethod(@Nullable Method method)
```
用于判断给定的方法是否是hashCode方法。

### isToStringMethod
```
isToStringMethod(@Nullable Method method)
```
用于判断给定的方法是否是toString方法。

### isObjectMethod
```
isObjectMethod(@Nullable Method method)
```
用于判断给定的方法是否是Object类中的方法。

```
try {  
    Class<?> clazz = PersonVO.class;  
    Method method = clazz.getMethod("getName");      
    boolean isObjectMethod = ReflectionUtils.isObjectMethod(method);  
    System.out.println("Is Object method: " + isObjectMethod);  
} catch (NoSuchMethodException | SecurityException e) {  
    e.printStackTrace();  
}  
```

### isCglibRenamedMethod
```
isCglibRenamedMethod(Method renamedMethod)
```
用于判断给定的方法是否是Cglib的类中的renamed方法。

### findField
```
findField(Class<?> clazz, String name)
```
用于在给定的类及其父类中查找指定名称的字段。

```
Class<?> clazz = PersonVO.class;  
Field field = ReflectionUtils.findField(clazz, "name");  
System.out.println("Field name: " + field.getName());  
```

```
findField(Class clazz, @Nullable String name, @Nullable Class type)
```
用于在给定的类及其父类中查找指定名称、类型的字段。

### setField
```
setField(Field field, @Nullable Object target, @Nullable Object value)
```
用于设置指定字段的值。

```
try {  
    Person myObject = new Person();  
    Field field = Person.class.getDeclaredField("name");  
    // 设置为可以访问  
    field.setAccessible(true);  
    ReflectionUtils.setField(field, myObject, "Hello, World!");  
    System.out.println("Field value: " + myObject.getName());  
} catch (NoSuchFieldException | SecurityException e) {  
    e.printStackTrace();  
}  
```

### getField
```
getField(Field field, @Nullable Object target)
```
用于获取指定字段的值。
```
try {  
    Person myObject = new Person("aaaa",10);  
    Field field = Person.class.getDeclaredField("name");  
    field.setAccessible(true);  
    Object value = ReflectionUtils.getField(field, myObject);  
    System.out.println("Field value: " + value);  
} catch (NoSuchFieldException | SecurityException e) {  
    e.printStackTrace();  
}  
```

### doWithLocalFields
```
doWithLocalFields(Class<?> clazz, FieldCallback fc)
```
用于对指定类的本地字段（即不包括父类的字段）进行操作。

```
Class<?> clazz = PersonVO.class;  
MyFieldCallback fieldCallback = new MyFieldCallback();  
ReflectionUtils.doWithLocalFields(clazz, fieldCallback);  
//FieldCallback 类  
class MyFieldCallback implements FieldCallback {  
  @Override  
  public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {  
    System.out.println("Field name: " + field.getName());  
    System.out.println("Field type: " + field.getType());  
  }  
}  
```

### doWithFields
```
doWithFields(Class<?> clazz, FieldCallback fc)
```
用于对指定类的字段（包括父类的字段）进行操作。
```
Class<?> clazz = PersonVO.class;  
MyFieldCallback fieldCallback = new MyFieldCallback();  
ReflectionUtils.doWithFields(clazz, fieldCallback);  
```

```
doWithFields(Class<?> clazz, FieldCallback fc, @Nullable FieldFilter ff)
```
用于对指定类的符合条件的字段进行操作。
```
Person myObject = new Person();  
// 使用doWithFields方法对字段进行操作  
ReflectionUtils.doWithFields(Person.class, new FieldCallback() {  
    @Override  
    public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {  
        // 在这里可以对每个字段进行自定义操作  
        field.setAccessible(true); // 设置字段可访问，因为字段是私有的  
        Object value = field.get(myObject); // 获取字段的值  
        System.out.println("Field name: " + field.getName());  
        System.out.println("Field type: " + field.getType());  
        System.out.println("Field value: " + value);  
    }  
}, new FieldFilter() {  
    @Override  
    public boolean matches(Field field) {  
        // 在这里可以定义过滤条件，只处理满足条件的字段  
        return field.getType() == String.class; // 只处理类型为String的字段  
    }  
});  
```
使用匿名内部类实现了FieldCallback接口，并在doWith方法中定义了对每个字段的操作。

在doWith方法中，我们首先将字段设置为可访问状态，然后使用field.get(myObject)方法获取字段的值，并打印字段的名称、类型和值。

同时，我们还使用匿名内部类实现了FieldFilter接口，并在matches方法中定义了过滤条件。

在这个示例中，我们只处理类型为String的字段。

### isPublicStaticFinal
```
isPublicStaticFinal(Field field)
```
用于判断一个字段是否是public、static和final修饰的。

```
Field[] fields = Person.class.getDeclaredFields();  
for (Field field : fields) {  
    if (ReflectionUtils.isPublicStaticFinal(field)) {  
        System.out.println(field.getName() + " is public, static, and final.");  
    }  
}  
```

最后总结一下，ReflectionUtils工具类的作用：

- 获取类的信息：ReflectionUtils可以通过类的全限定名获取对应的Class对象，进而获取类的各种信息，如类名、包名、父类、接口等。
- 创建对象：ReflectionUtils可以通过Class对象创建实例，即通过反射实现动态创建对象的功能。
- 调用方法：ReflectionUtils可以通过Method对象调用类的方法，包括无参方法和有参方法，可以通过方法名和参数类型来定位方法。
- 访问字段：ReflectionUtils可以通过Field对象访问类的字段，包括获取字段值和设置字段值。
- 调用构造方法：ReflectionUtils可以通过Constructor对象调用类的构造方法，包括无参构造方法和有参构造方法。














